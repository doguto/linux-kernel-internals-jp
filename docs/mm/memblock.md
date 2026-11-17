# memblock: ブート時のメモリアロケータ {#memblock-the-boot-time-memory-allocator}

> バディシステムがメモリを管理できるようになる前に、まず何かがメモリを管理しなければならない

## memblockとは何か {#what-is-memblock}

memblockは、カーネルのブート初期のメモリアロケータである。どの物理メモリ領域が存在し、どれがすでに予約されているかを追跡し、バディアロケータ（ページアロケータ）が初期化されるよりずっと前の、ブートの最初期段階でカーネルがメモリを割り当てられるようにする。

```c
/* 物理メモリの領域を追加する */
int memblock_add(phys_addr_t base, phys_addr_t size);

/* 領域を予約済み（使用中）としてマークする */
int memblock_reserve(phys_addr_t base, phys_addr_t size);

/* ブート初期にメモリを割り当てる */
void *memblock_alloc(phys_addr_t size, phys_addr_t align);
```

## 鶏が先か卵が先かの問題 {#the-chicken-and-egg-problem}

バディアロケータは、実行時におけるカーネルの主要な物理メモリ管理機構である。しかし、それをセットアップするにはメモリが必要になる:

- **ゾーン構造体**（`struct zone`）を、各メモリゾーンについて割り当てる必要がある
- **ページフレームのメタデータ**（`struct page`）は物理ページごとに1エントリ必要であり、16GBのシステムでは数百万エントリになる
- 割り当てキャッシュのための**CPUごとのデータ構造**
- バディシステム自体の**フリーリスト**

しかし、メモリアロケータなしにメモリを割り当てることはできない。もっと単純な何かが先に来なければならない。

memblockは、意図的に最小限にとどめた設計でこのブートストラップ問題を解決する。メモリ領域を追跡する2つの平坦な配列である。ページテーブルも、ロックも、CPUごとのキャッシュもない。あるのは「何が存在するか」と「何が使われているか」だけである。

```
Boot sequence:

  1. Firmware / bootloader loads kernel into RAM

  2. Architecture setup discovers physical memory layout
     (via device tree, ACPI, or e820 on x86)

  3. memblock tracks available and reserved regions
     +----------------------------------------------+
     | memblock.memory:   [0 - 4GB], [8GB - 16GB]  |  <- what exists
     | memblock.reserved: [0 - 16MB], [64MB - 80MB] |  <- what's taken
     +----------------------------------------------+

  4. Kernel uses memblock_alloc() to get memory for
     page tables, struct page array, zone structures...

  5. Buddy allocator initialized using memblock info

  6. memblock_free_all() hands remaining free memory
     to the buddy system

  7. memblock is done (memory can be freed if not needed)
```

## memblockの仕組み {#how-memblock-works}

### 2つの配列: memoryとreserved {#two-arrays-memory-and-reserved}

中核となるデータ構造は単純である。memblockは2つの領域の集合を保持する:

```c
struct memblock {
    bool bottom_up;                          /* 割り当ての方向 */
    phys_addr_t current_limit;               /* 割り当ての上限 */
    struct memblock_type memory;             /* 物理メモリ領域 */
    struct memblock_type reserved;           /* 予約済み（使用中）の領域 */
};
```

各`memblock_type`は、動的に拡張される領域の配列を保持する:

```c
struct memblock_type {
    unsigned long cnt;                       /* 領域の数 */
    unsigned long max;                       /* 最大領域数（必要に応じて拡張） */
    phys_addr_t total_size;                  /* 全領域の合計サイズ */
    struct memblock_region *regions;         /* 領域の配列 */
    char *name;                              /* "memory"または"reserved" */
};

struct memblock_region {
    phys_addr_t base;                        /* 開始アドレス */
    phys_addr_t size;                        /* 領域のサイズ */
    enum memblock_flags flags;               /* HOTPLUG、MIRROR、NOMAP */
#ifdef CONFIG_NUMA
    int nid;                                 /* NUMAノードID */
#endif
};
```

`memory`配列は、（ファームウェアが報告する）どの物理メモリが存在するかを記述する。`reserved`配列は、すでに確保されているもの、すなわちカーネルイメージ自体、device tree blob、initrd、初期の割り当てなどを記述する。

**空きメモリ**は暗黙的に、`memory`内の領域のうち`reserved`に覆われていない部分として定義される。

```
Physical address space:
0                                                              16GB
|--------------------------------------------------------------|

memblock.memory (what exists):
|==========|                  |================================|
0        4GB                 8GB                             16GB

memblock.reserved (what's taken):
|==|    |=|                  |=|
kernel  DTB                  initrd

Free = memory minus reserved
```

### 領域の結合 {#region-merging}

隣接する領域や重なり合う領域が追加されると、memblockはそれらを自動的に結合する。これによって配列がコンパクトに保たれる:

```
memblock_add(0, 1GB)    -> memory: [0, 1GB]
memblock_add(1GB, 1GB)  -> memory: [0, 2GB]     (merged)
memblock_add(3GB, 1GB)  -> memory: [0, 2GB], [3GB, 4GB]  (gap, no merge)
```

### 静的なブートストラップ {#static-bootstrap}

最初の領域配列は静的に割り当てられている:

```c
static struct memblock_region memblock_memory_init_regions[INIT_MEMBLOCK_REGIONS];
static struct memblock_region memblock_reserved_init_regions[INIT_MEMBLOCK_RESERVED_REGIONS];
```

`INIT_MEMBLOCK_REGIONS`のデフォルトは128である。より多くの領域が必要になると、memblockは`memblock_double_array()`を使って配列を2倍にする。これはmemblock自身から新しい領域を割り当てるもので、古い配列はコピーが完了するまで有効なままなので成り立つ、うまい自己参照的なトリックである。

## 主要なAPI {#key-apis}

### メモリの追加と削除 {#adding-and-removing-memory}

```c
/* memblockに物理メモリ領域を伝える（アーキテクチャのコードから呼ばれる） */
int memblock_add(phys_addr_t base, phys_addr_t size);

/* 領域を削除する（例: ファームウェアが予約した領域） */
int memblock_remove(phys_addr_t base, phys_addr_t size);

/* 領域を予約済みとしてマークする */
int memblock_reserve(phys_addr_t base, phys_addr_t size);

/* 以前に予約した領域を解放する */
int memblock_phys_free(phys_addr_t base, phys_addr_t size);

/* メモリを直接使用不可としてマークする（マップされない） */
int memblock_mark_nomap(phys_addr_t base, phys_addr_t size);
```

### メモリの割り当て {#allocating-memory}

```c
/* アラインされたメモリを割り当てる（最も一般的） */
void *memblock_alloc(phys_addr_t size, phys_addr_t align);

/* 特定のアドレス範囲から割り当てる */
void *memblock_alloc_range_nid(phys_addr_t size, phys_addr_t align,
                               phys_addr_t start, phys_addr_t end,
                               int nid, bool exact_nid);

/* 生の物理アドレスを割り当てる（仮想マッピングなし） */
phys_addr_t memblock_phys_alloc(phys_addr_t size, phys_addr_t align);
```

`memblock_alloc()`は2つのことを行う。空いている物理領域を見つけ、それをカーネルの仮想アドレス空間にマップする（`memblock_alloc_internal()`を介して行われ、これは`memblock_alloc_range_nid()`を呼んだ後、その結果に対して`memblock_reserve()`を呼ぶ）。返されるポインタは、すぐに使えるカーネル仮想アドレスである。

### メモリの照会 {#querying-memory}

```c
/* このアドレスは予約済み領域内にあるか? */
bool memblock_is_reserved(phys_addr_t addr);

/* このアドレスはメモリ領域内にあるか? */
bool memblock_is_memory(phys_addr_t addr);

/* 使用可能なRAMの総量 */
phys_addr_t memblock_phys_mem_size(void);

/* 空いている（予約されていない）メモリ領域を走査する */
for_each_free_mem_range(i, nid, flags, &start, &end, NULL)
```

## メモリの検出: Device TreeとACPI {#memory-discovery-device-tree-and-acpi}

memblockは自分でメモリを検出するわけではない。アーキテクチャ固有のコードが、ファームウェアの情報を使ってmemblockにデータを投入する。

### Device Tree（ARM、RISC-V、PowerPCなど） {#device-tree-arm-risc-v-powerpc-etc}

device treeを使うプラットフォームでは、`/memory`ノードが物理RAMを記述する:

```dts
/ {
    memory@80000000 {
        device_type = "memory";
        reg = <0x00 0x80000000 0x00 0x80000000>;  /* 0x80000000から2GB */
    };
};
```

カーネルの初期のDTスキャンコード（[`drivers/of/fdt.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/of/fdt.c)の`early_init_dt_scan_memory()`）がこれらのノードをパースし、各領域について`memblock_add()`を呼び出す。予約メモリのノード（`/reserved-memory`）は`memblock_reserve()`呼び出しになる。

### ACPI（x86、ARMサーバー） {#acpi-x86-arm-servers}

ACPIシステムでは、ファームウェアはさまざまな仕組みを通じてメモリマップを提供する:

- **x86**: BIOS/UEFIが**e820メモリマップ**を提供する。これは種別（usable、reserved、ACPI reclaimableなど）付きのアドレス範囲の表である。[`arch/x86/kernel/e820.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/e820.c)の関数`e820__memblock_setup()`が、e820のエントリを`memblock_add()`と`memblock_reserve()`の呼び出しに変換する。
- **ACPIを使うARM64**: EFIメモリマップのエントリを使い、`efi_init()`で処理される。

### パターン {#the-pattern}

プラットフォームにかかわらず、流れは同じである:

```
Firmware provides memory layout
        |
        v
Architecture code parses it
        |
        v
memblock_add() for usable regions
memblock_reserve() for reserved regions
        |
        v
memblock has complete picture of physical memory
```

## バディアロケータへの引き渡し {#the-handoff-to-the-buddy-allocator}

カーネルがmemblockを使って、バディアロケータが必要とするすべてのデータ構造をセットアップし終えると、memblockの仕事はほぼ終わりである。引き渡しは`memblock_free_all()`を通じて行われる:

```c
void __init memblock_free_all(void)
{
    unsigned long pages;

    pages = free_low_memory_core_early();  /* memblockのページをバディに解放する */
    totalram_pages_add(pages);
}
```

この関数はmemblockのすべてのメモリ領域をたどり、予約されて**いない**各ページについて`__free_pages_core()`を呼び出してバディアロケータに引き渡す。この時点以降、バディシステムがすべての空き物理メモリを所有し、割り当てにmemblockはもはや必要ない。

memblockのデータ構造自体には`__initdata`が付けられている。つまりカーネルのinitセクションに置かれる。ブートが完了すると（`free_initmem()`）、このメモリは回収される。ただし`CONFIG_ARCH_KEEP_MEMBLOCK`が有効な場合は別で、その場合はmemblockのデータが後の照会のために保持される（kexecやメモリホットプラグに有用）。

```
memblock_free_all() walkthrough:

  For each page in memblock.memory:
      if not in memblock.reserved:
          __free_pages_core(page)   ->  buddy allocator now owns it
          totalram_pages++

  Result: buddy free lists populated, system ready for normal allocation
```

## 歴史と進化 {#history-and-evolution}

### bootmem: 元々の手法（Linux 2.4から初期の3.xまで） {#bootmem-the-original-approach-linux-24-through-early-3x}

memblock以前、カーネルは`bootmem`と呼ばれる**ビットマップベース**のブートアロケータを使っていた。各ビットが1つの物理ページを表し、1 = 割り当て済み、0 = 空きである。

bootmemの問題点:

- **ビットマップのオーバーヘッド**: 64GBのシステムでは、集計のためだけに2MBのビットマップが必要になる
- **線形探索**: 空きメモリを見つけるにはビットマップを走査する必要があり、ページ数に対してO(n)である
- **NUMAを意識しない**: 元々のbootmemにはメモリノードという概念がなかった
- **脆弱**: ビットマップ自体もメモリのどこかに置く必要があり、別のブートストラップ問題を生んでいた

### memblock（Logical Memory Blocks）の台頭 {#the-rise-of-memblock-logical-memory-blocks}

memblockの前身は**lmb**（Logical Memory Blocks）と呼ばれ、PowerPCで使われていた。これはメモリをページごとのビットマップではなく領域のリストとして追跡するもので、メモリがほぼ連続している場合にははるかに効率的である。

**コミット**: [95f72d1ed41a](https://git.kernel.org/linus/95f72d1ed41a) ("lmb: rename to memblock") | [LWNの記事](https://lwn.net/Articles/382559/)
**カーネル**: v2.6.35（2010年）
**作者**: Yinghai Lu

Yinghai Luはlmbをmemblockに改名し、x86で使えるよう拡張し始めた。この改名は、それをアーキテクチャ非依存にするための第一歩であった。

### memblockがbootmemを置き換える {#memblock-replaces-bootmem}

いくつものカーネルリリースにわたって、各アーキテクチャはbootmemからmemblockへ移行した:

- **ARM**: PowerPCと並ぶ初期の採用者の1つ
- **x86**: v2.6.35からv3.xの時期に移行した

**コミット**: [9a8dd708d547](https://git.kernel.org/linus/9a8dd708d547) ("memblock: remove bootmem dependency on memblock")
**カーネル**: v3.12（2013年）

すべてのアーキテクチャの変換が終わったとき、bootmemは最終的に削除された:

**コミット**: [355c45affca7](https://git.kernel.org/linus/355c45affca7114ab510e296a5b7012943aeea17) ("mm: remove bootmem allocator implementation")
**カーネル**: v4.20（2018年）
**作者**: Mike Rapoport（IBM）

これは大きな整理であった。bootmemのコードには、数十年にわたって回避策や特殊ケースが積み重なっていた。

### 最近の改良 {#recent-improvements}

**コミット**: [35fd0808de0e](https://git.kernel.org/linus/35fd0808de0e) ("memblock: introduce memblock_phys_free()")
**カーネル**: v6.1（2022年）
**作者**: Mike Rapoport

初期のメモリ管理を整理し、紛らわしいセマンティクスを持っていた（memoryリストではなくreservedリストから解放する）`memblock_free()`の使用を減らすという、より広範な取り組みの一環である。

## memblockを観測する {#observing-memblock}

### debugfsインターフェース {#debugfs-interface}

`CONFIG_ARCH_KEEP_MEMBLOCK`が有効であれば、ブート後にmemblockの状態を調べられる:

```bash
# メモリ領域を表示する
cat /sys/kernel/debug/memblock/memory

# 予約済み領域を表示する
cat /sys/kernel/debug/memblock/reserved

# 出力例（memory）:
#    0: 0x0000000000000000..0x000000009fffffff (2560 MB)
#    1: 0x0000000100000000..0x00000004ffffffff (16384 MB)
```

### ブート初期のメッセージ {#early-boot-messages}

memblockはブート初期に自身の動作をdmesgに記録する。`memblock`という接頭辞の付いた行を探すとよい:

```bash
dmesg | grep -i memblock

# 出力例:
# [    0.000000] memblock_add: [mem 0x00000000-0x9fffffff]
# [    0.000000] memblock_reserve: [mem 0x01000000-0x01ffffff]
# [    0.000000] memblock: memory size = 16384 MB
```

より詳細な出力を得るには、カーネルコマンドラインに`memblock=debug`を渡す。これにより、すべてのmemblock操作の詳細なログ出力が有効になる:

```bash
# ブートローダーの設定内（例: GRUB）:
linux /vmlinuz memblock=debug ...
```

### e820テーブル（x86） {#e820-table-x86}

x86システムでは、memblockに供給されるファームウェアのメモリマップを見ることができる:

```bash
dmesg | grep e820

# 出力例:
# BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
# BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
# BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
# BIOS-e820: [mem 0x0000000000100000-0x000000003fffffff] usable
```

## 試してみる {#try-it-yourself}

### memblockの領域を調べる {#inspect-memblock-regions}

```bash
# カーネルがmemblockのデータを保持しているか確認する
ls /sys/kernel/debug/memblock/ 2>/dev/null && echo "memblock debugfs available"

# 利用可能なら、メモリレイアウトを調べる
cat /sys/kernel/debug/memblock/memory
cat /sys/kernel/debug/memblock/reserved

# メモリ領域の数を数える
wc -l < /sys/kernel/debug/memblock/memory
```

### ブート時のメモリセットアップを追跡する {#trace-the-boot-memory-setup}

```bash
# ファームウェアがカーネルにメモリをどう報告したかを見る
dmesg | grep -E "(e820|memblock|Memory:)"

# "Memory:"の行が最終的な集計を示す
# Memory: 16123456K/16777216K available
#          (X kernel code, Y rwdata, Z rodata, W init, V bss, ...)
```

### memblockの詳細なログ出力を有効にする {#enable-verbose-memblock-logging}

```bash
# カーネルコマンドラインに追加する（再起動が必要）
# GRUBの場合: /etc/default/grubを編集し、GRUB_CMDLINE_LINUXにmemblock=debugを追加する
sudo sed -i 's/GRUB_CMDLINE_LINUX="\(.*\)"/GRUB_CMDLINE_LINUX="\1 memblock=debug"/' /etc/default/grub
sudo update-grub

# 再起動後、詳細なログを確認する
dmesg | grep memblock | head -50
```

### memblockと実行時のビューを比較する {#compare-memblock-with-runtime-view}

```bash
# memblockのビュー（ブート時、保持されていれば）
cat /sys/kernel/debug/memblock/memory 2>/dev/null

# バディアロケータのビュー（実行時）
cat /proc/buddyinfo

# 管理されているメモリの総量
grep MemTotal /proc/meminfo
```

## 主要なデータ構造 {#key-data-structures}

### struct memblock（グローバルな状態） {#struct-memblock-global-state}

memblockの状態全体は、単一のグローバル変数にある:

```c
struct memblock memblock __initdata_memblock = {
    .memory.regions     = memblock_memory_init_regions,
    .memory.cnt         = 1,    /* 空のダミー領域 */
    .memory.max         = INIT_MEMBLOCK_REGIONS,
    .memory.name        = "memory",

    .reserved.regions   = memblock_reserved_init_regions,
    .reserved.cnt       = 1,
    .reserved.max       = INIT_MEMBLOCK_RESERVED_REGIONS,
    .reserved.name      = "reserved",

    .bottom_up          = false,
    .current_limit      = MEMBLOCK_ALLOC_ANYWHERE,
};
```

`__initdata_memblock`アノテーションは、`CONFIG_ARCH_KEEP_MEMBLOCK`に応じて、このデータが`__initdata`（ブート後に解放される）か`__meminitdata`（メモリホットプラグのために保持される）のいずれかになることを意味する。

### memblock_flags {#memblock_flags}

```c
enum memblock_flags {
    MEMBLOCK_NONE       = 0x0,   /* 特別なフラグなし */
    MEMBLOCK_HOTPLUG    = 0x1,   /* ホットプラグ可能なメモリ */
    MEMBLOCK_MIRROR     = 0x2,   /* ミラーされた（EFI）メモリ */
    MEMBLOCK_NOMAP      = 0x4,   /* ダイレクトマッピングに追加しない */
    MEMBLOCK_DRIVER_MANAGED = 0x8, /* ドライバが管理する */
};
```

## 参考資料 {#references}

### 主要なコード {#key-code}

| ファイル | 説明 |
|------|-------------|
| [`mm/memblock.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memblock.c) | memblockの中核実装 |
| [`include/linux/memblock.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/memblock.h) | 公開APIとデータ構造 |
| [`drivers/of/fdt.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/of/fdt.c) | device treeのメモリスキャン |
| [`arch/x86/kernel/e820.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/e820.c) | x86のe820からmemblockへの変換 |

### 主要なコミット {#key-commits}

| コミット | カーネル | 説明 |
|--------|--------|-------------|
| [95f72d1ed41a](https://git.kernel.org/linus/95f72d1ed41a) | v2.6.35 | lmbをmemblockに改名 |
| [9a8dd708d547](https://git.kernel.org/linus/9a8dd708d547) | v3.12 | memblockに対するbootmemの依存を削除 |
| [355c45affca7](https://git.kernel.org/linus/355c45affca7114ab510e296a5b7012943aeea17) | v4.20 | bootmemアロケータの実装を削除 |
| [35fd0808de0e](https://git.kernel.org/linus/35fd0808de0e) | v6.1 | memblock_phys_free()を導入 |

### LWNの記事 {#lwn-articles}

- [A quick history of early-boot memory allocators](https://lwn.net/Articles/382559/) -- lmbからmemblockへの改名とその理由についての記事
- [Improvements to memblock](https://lwn.net/Articles/761215/) -- 初期メモリアロケータを整理したMike Rapoportの取り組み

### 関連 {#related}

- [page-allocator](page-allocator.md) - memblockが引き渡す先のバディアロケータ
- [overview](overview.md) - アロケータの階層におけるmemblockの位置付け
- [numa](numa.md) - NUMAを意識したmemblockの割り当て

## 参考文献 {#further-reading}

- [page-allocator.md](page-allocator.md) — `memblock_free_all()`の後にmemblockから処理を引き継ぐバディアロケータ
- [boot-page-tables.md](boot-page-tables.md) — メモリ割り当てをmemblockに依存する、初期のページテーブルのセットアップ
- [numa.md](numa.md) — ブート初期に`memblock_region.nid`を通じて追跡されるNUMAノードの割り当て
- [memory-hotplug.md](memory-hotplug.md) — `CONFIG_ARCH_KEEP_MEMBLOCK`は、ブート後のホットプラグ操作のためにmemblockのデータを保持する
- [mm/memblock.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memblock.c) — memblockの中核実装。`memblock_alloc_range_nid()`が中心となるアロケータである
- [include/linux/memblock.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/memblock.h) — 公開API、データ構造、`for_each_free_mem_range()`のような走査マクロ
- [arch/x86/kernel/e820.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/e820.c) — x86のファームウェアメモリマップをmemblockの領域に変換する
- [drivers/of/fdt.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/of/fdt.c) — ARMとRISC-Vで`memblock_add()`にデータを供給するdevice treeのメモリスキャン
- [LWN: A quick history of early-boot memory allocators](https://lwn.net/Articles/382559/) — bootmemのビットマップから領域リスト方式への移行の背景
- [LWN: Improvements to memblock](https://lwn.net/Articles/761215/) — Mike Rapoportによる整理作業と、bootmemの最終的な削除
- `Documentation/core-api/boot-time-mm.rst` — ブート時のメモリ管理APIに関するカーネルドキュメント
