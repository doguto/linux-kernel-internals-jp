# カーネルのメモリ予約メカニズム {#kernel-memory-reservation-mechanisms}

> 物理RAMのすべてが自由に使えるわけではない。ユーザー空間が動き出すずっと前に、カーネル、ファームウェア、ハードウェアが領域を切り出している

## なぜ予約が存在するか {#why-reservations-exist}

物理メモリは、平坦で一様に利用できる資源ではない。カーネルが`start_kernel()`に到達する時点で、すでにいくつもの当事者が物理アドレス空間の一部を占有している。

- **ファームウェアテーブル** — BIOS、ACPIテーブル（RSDT、XSDT、DSDT）、SMBIOSデータはRAM上に置かれており、上書きしてはならない
- **カーネルのテキストとデータ** — `.text`、`.rodata`、`.data`、`.bss`および関連セクションは、ブートローダが設定した固定の物理アドレスを占める
- **EFIランタイムサービス** — UEFIシステムでは、ブートサービスのメモリとEFIメモリマップ自体を保護する必要がある
- **EBDAとレガシーBIOS領域** — Extended BIOS Data Areaと先頭1メガバイトの上端は、歴史的に破壊されやすい
- **デバイスメモリとメモリマップドI/O** — ファームウェアは範囲をRAMではなく`E820_TYPE_RESERVED`として記述することがある
- **crashkernel領域** — kdumpのキャプチャカーネル用に確保しておく専用領域
- **CMAプール** — DMA用の連続メモリで、必要になるまでは通常のmovableメモリとしても使われる
- **ハードウェアの不具合（quirk）** — 一部のチップセットは特定の物理範囲を破壊する（Sandy Bridge iGPUはx86における代表例）

汎用のpage割り当てをすべて管理するバディアロケータは、空いているとわかっているpageしか払い出せない。その集合の外にあるものは、`memblock_free_all()`がメモリを引き渡す前に予約されていなければならない。

!!! note "主要なソースファイル"
    | ファイル | 役割 |
    |---|---|
    | `mm/memblock.c` | memblockアロケータと予約エンジンの中核 |
    | `include/linux/memblock.h` | 公開API、フラグ定義、インラインラッパー |
    | `arch/x86/kernel/setup.c` | `setup_arch()` — x86のブート時予約をすべて統括する |
    | `arch/x86/kernel/ebda.c` | `reserve_bios_regions()` — EBDAとレガシーBIOS領域 |
    | `arch/x86/realmode/init.c` | `reserve_real_mode()` — 先頭1 MBとトランポリン |
    | `arch/x86/platform/efi/efi.c` | `efi_memblock_x86_reserve_range()` — EFIメモリマップ |
    | `arch/x86/kernel/e820.c` | `e820__memblock_setup()` — e820マップをmemblockに変換する |
    | `kernel/crash_reserve.c` | `reserve_crashkernel_generic()` — kdump領域 |
    | `mm/memory_hotplug.c` | `movable_node`パラメータの処理 |

---

## memblockによる予約 {#memblock-reservations}

memblockはカーネルのブート時メモリマネージャである。`struct memblock_region`の平坦な配列を2つ保持する。1つは利用可能なすべての物理メモリ用（`memblock.memory`）、もう1つは予約領域用（`memblock.reserved`）である。`mm_init()`の終盤で`memblock_free_all()`が呼ばれると、`memblock.memory`に含まれ、かつ`memblock.reserved`には*含まれない*pageだけがバディアロケータに解放される。それ以外はすべて立ち入り禁止のままとなる。

### memblock_reserve() {#memblock_reserve}

主要な予約関数は、`include/linux/memblock.h`にある薄いインラインラッパーである。

```c
static __always_inline int memblock_reserve(phys_addr_t base, phys_addr_t size)
{
    return __memblock_reserve(base, size, NUMA_NO_NODE, 0);
}
```

`__memblock_reserve()`は`memblock.reserved`に対して`memblock_add_range()`を呼び、重なり合う領域や隣接する領域をその都度マージする。この関数は、ページングが完全に初期化される前の`setup_arch()`の最初期段階から安全に呼び出せる。

派生版の`memblock_reserve_kern()`は、追加で`MEMBLOCK_RSRV_KERN`フラグを設定し、その領域が（ファームウェアやデバイスメモリなどではなく）カーネル自身によって消費されることを示す。

```c
static __always_inline int memblock_reserve_kern(phys_addr_t base, phys_addr_t size)
{
    return __memblock_reserve(base, size, NUMA_NO_NODE, MEMBLOCK_RSRV_KERN);
}
```

memblock内部の割り当てはすべて、自動的に`MEMBLOCK_RSRV_KERN`を設定する。

### memblock_phys_alloc() {#memblockphysalloc}

バディアロケータが使えるようになる前に、カーネルは自身のデータ構造（ページテーブル、`struct page`配列、zone構造体）のためのメモリを割り当てなければならない。`memblock_phys_alloc()`はこうした要求を満たし、物理アドレスを返す。

```c
static __always_inline phys_addr_t memblock_phys_alloc(phys_addr_t size,
                                                        phys_addr_t align)
{
    return memblock_phys_alloc_range(size, align, 0, MEMBLOCK_ALLOC_ACCESSIBLE);
}
```

この呼び出しは割り当てを行うと同時に、`MEMBLOCK_RSRV_KERN`付きの`__memblock_reserve()`を介して暗黙的にその領域を予約済みとしてマークする。マップ済み仮想アドレスを返す派生版の`memblock_alloc()`も、カーネルのデータ構造の初期化で広く使われている。

### memblock_mark_nomap() {#memblockmarknomap}

予約された領域にも、デフォルトではstruct pageが割り当てられ、カーネルのダイレクトマップでカバーされる。`memblock_mark_nomap()`はさらに踏み込んで`MEMBLOCK_NOMAP`フラグを設定し、その領域がカーネルのダイレクト物理マッピングにまったく含まれないようにする。

```c
int __init_memblock memblock_mark_nomap(phys_addr_t base, phys_addr_t size)
{
    return memblock_setclr_flag(&memblock.memory, base, size, 1, MEMBLOCK_NOMAP);
}
```

`MEMBLOCK_NOMAP`とマークされた領域もメモリマップにはカバーされる（struct pageは存在し、`PageReserved()`となる）が、カーネルはその領域に対するリニアマップ上の仮想アドレスを持たない。これは、特別なマッピング処理を必要とするデバイスメモリの範囲や永続メモリに使われる。`include/linux/memblock.h`のコメントにあるとおり、*"don't add to kernel direct mapping and treat as reserved in the memory map."*（カーネルのダイレクトマッピングに追加せず、メモリマップ上では予約済みとして扱う）である。

### memblockフラグのまとめ {#memblock-flags-summary}

| フラグ | 意味 |
|---|---|
| `MEMBLOCK_NONE` | 通常の予約領域 |
| `MEMBLOCK_HOTPLUG` | ファームウェアが記述したホット（アン）プラグ可能なメモリ |
| `MEMBLOCK_MIRROR` | ミラー化（冗長化）されたメモリ領域 |
| `MEMBLOCK_NOMAP` | カーネルのダイレクトマップから除外する |
| `MEMBLOCK_DRIVER_MANAGED` | ファームウェアのRAMではなく、常にドライバ経由で追加される |
| `MEMBLOCK_RSRV_NOINIT` | 予約済み。struct pageは完全には初期化されない |
| `MEMBLOCK_RSRV_KERN` | カーネル用に予約済み |

---

## x86でブート時に予約されるもの {#what-gets-reserved-at-boot-on-x86}

`arch/x86/kernel/setup.c`の`setup_arch()`が、x86における物理メモリの予約をすべて統括する。順序が重要であり、後の段階は先の予約がすでに済んでいることを前提とする。

### フェーズ1: early_reserve_memory() {#phase-1-earlyreservememory}

`e820__memory_setup()`がRAMを`memblock.memory`に追加する前に呼ばれるため、これらの領域の上に何かが割り当てられることはない。

```c
static void __init early_reserve_memory(void)
{
    /*
     * _text から __end_of_kernel_reserve シンボルまでの間で
     * カーネルが占有するメモリを予約する。
     */
    memblock_reserve_kern(__pa_symbol(_text),
                  (unsigned long)__end_of_kernel_reserve - (unsigned long)_text);

    /* 先頭 64K を予約する。BIOS は低位メモリを破壊することが知られている */
    memblock_reserve(0, SZ_64K);

    early_reserve_initrd();

    memblock_x86_reserve_range_setup_data();

    reserve_bios_regions();
    trim_snb_memory();
}
```

**カーネルイメージ:** `_text`から`__end_of_kernel_reserve`（`arch/x86/kernel/vmlinux.lds.S`でリンカが定義するシンボル）までの範囲は、`.text`、`.rodata`、`.data`、`.bss`および関連セクションをカバーする。`__end_of_kernel_reserve`の*後*に置かれたセクション（`.brk`など）は、別途予約しなければならない。

**低位メモリ:** 先頭64 KB（後に`reserve_real_mode()`によって1 MBに拡大される）が予約されるのは、レガシーBIOSが低位RAMを破壊するためであり、また先頭pageはL1TFのサイドチャネル漏洩を防ぐために予約しなければならないからである。

**initrd:** `early_reserve_initrd()`は、`boot_params`から得たramdiskイメージのアドレスに対して即座に`memblock_reserve_kern()`を呼び、セットアップが完了する前にmemblockがその上に割り当てを行えないようにする。

**BIOS/EBDA:** `arch/x86/kernel/ebda.c`の`reserve_bios_regions()`はEBDAポインタを読み取り、EBDAの先頭から先頭1メガバイトの上端までをすべて予約する。

```c
memblock_reserve(bios_start, 0x100000 - bios_start);
```

### フェーズ2: e820とEFIメモリマップの処理 {#phase-2-e820-and-efi-memory-map-processing}

`e820__memory_setup()`はファームウェアの物理メモリマップ（BIOSのe820またはEFIメモリマップ）を読み取り、何がRAMで、何が予約済みで、何が存在しないかについての信頼できる全体像を構築する。`E820_TYPE_RAM`型の領域は、その後`e820__memblock_setup()`によって`memblock.memory`に追加される。

EFIシステムでは、`efi_memblock_x86_reserve_range()`がEFIメモリマップのテーブル自体を予約する。

```c
memblock_reserve(pmap, efi.memmap.nr_map * efi.memmap.desc_size);
```

### フェーズ3: e820処理後の予約 {#phase-3-post-e820-reservations}

`e820__memblock_setup()`の後にはメモリトポロジ全体が判明しており、さらに予約が行われる。

- **EFIブートサービス:** `efi_reserve_boot_services()`はEFIブートサービスのメモリを固定し、ブートCPUが立ち上がった後に`efi_free_boot_services()`が呼ばれるまで、ランタイムサービスがアクセス可能な状態を保つ。
- **リアルモードトランポリン:** `x86_platform.realmode_reserve()`は`reserve_real_mode()`を呼ぶ。これは1 MB未満のトランポリン領域（AP起動用）を割り当て、その後`memblock_reserve(0, SZ_1M)`で先頭1メガバイト全体を無条件に予約する。
- **initrd（最終）:** `reserve_initrd()`は後でinitrdのマッピングを検証し、コピーが済んだ時点で物理pageを解放する。
- **ACPIテーブル:** `acpi_boot_table_init()`は最終的に、ファームウェアのマップで見つかったACPIテーブルの範囲に対して`memblock_reserve()`を呼ぶ。
- **CMA:** `dma_contiguous_reserve()`がCMA領域を切り出す（後述の[CMAによる予約](#cma-reservations)を参照）。
- **crashkernel:** `arch_reserve_crashkernel()`がkdump領域を予約する（後述の[crashkernelの予約](#crashkernel-reservation)を参照）。

### 予約ログの読み方 {#reading-the-reservation-log}

カーネルはブート初期にmemblockの予約をすべてログに出力する。典型的なx86のdmesgには次のような内容が含まれる。

```
[    0.000000] BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x00000000bffdffff] usable
[    0.000000] BIOS-e820: [mem 0x00000000bffe0000-0x00000000bfffffff] reserved
...
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000000001000-0x000000000009efff]
[    0.000000]   node   0: [mem 0x0000000000100000-0x00000000bffdffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000000000000-0x00000000bfffffff]
[    0.000000] crashkernel reserved: 0x0000000009000000 - 0x0000000049000000 (1024 MB)
```

---

## 予約済みメモリのマップ {#the-reserved-memory-map}

### /proc/iomem {#prociomem}

`/proc/iomem`はカーネルのリソースツリーを示す正規のビューであり、占有されたすべての物理アドレス範囲の階層的な記録である。`insert_resource()`や`request_resource()`を呼ぶ予約はすべてここに現れる。

```
# cat /proc/iomem
00000000-00000fff : Reserved
00001000-0009fbff : System RAM
0009fc00-0009ffff : Reserved
000a0000-000bffff : PCI Bus 0000:00
000c0000-000c99ff : Video ROM
000f0000-000fffff : Reserved
  000f0000-000fffff : System ROM
00100000-bffdffff : System RAM
  01000000-01c0ae8b : Kernel code
  01c0ae8c-0228317f : Kernel rodata
  02284000-02430abf : Kernel data
  02623000-02ffffff : Kernel bss
09000000-48ffffff : Crash kernel
bffe0000-bfffffff : Reserved
fd000000-fdffffff : PCI Bus 0000:00
...
```

注目すべきフィールド:

- `System RAM` — カーネルが利用できるメモリ
- `Kernel code` / `Kernel rodata` / `Kernel data` / `Kernel bss` — カーネルイメージが占める正確な領域
- `Crash kernel` — crashkernel用に予約された領域
- `Reserved` — カーネルが使用しない、ファームウェアが予約した領域
- `PCI Bus` — PCIが占有するメモリマップドI/Oの範囲

### /sys/kernel/debug/memblock/reserved {#syskerneldebugmemblockreserved}

`CONFIG_DEBUG_FS`が有効な場合、`mm/memblock.c`の`memblock_init_debugfs()`がdebugfsのエントリを登録する。

```c
static int __init memblock_init_debugfs(void)
{
    struct dentry *root = debugfs_create_dir("memblock", NULL);

    debugfs_create_file("memory", 0444, root,
                &memblock.memory, &memblock_debug_fops);
    debugfs_create_file("reserved", 0444, root,
                &memblock.reserved, &memblock_debug_fops);
    ...
}
```

```
# cat /sys/kernel/debug/memblock/reserved
   0: 0x0000000000000000..0x00000000000fffff
   1: 0x0000000001000000..0x0000000002ffffff
   2: 0x0000000009000000..0x0000000048ffffff
   3: 0x00000000bffe0000..0x00000000bfffffff
```

!!! note
    このファイルは読み出した時点でのmemblockの状態を反映する。（`mm_init()`中に）`memblock_free_all()`が呼ばれた後、`CONFIG_ARCH_KEEP_MEMBLOCK`が設定されていなければ、memblockの内部配列は解放されることがある。そのようなカーネルでは、ブート後にこのdebugfsファイルは存在しないか空になる場合がある。x86では通常`CONFIG_ARCH_KEEP_MEMBLOCK`が有効になっており、データは保持される。

---

## crashkernelの予約 {#crashkernel-reservation}

カーネルコマンドラインパラメータ`crashkernel=`は、ブート時に物理メモリの連続領域を予約する。この領域は、実行中のカーネルがパニックしたときに2つ目の「キャプチャカーネル」に引き渡される。これがkdumpの基盤である。

### 仕組み {#how-it-works}

`arch/x86/kernel/setup.c`の`arch_reserve_crashkernel()`はコマンドラインを解析し、`kernel/crash_reserve.c`の`reserve_crashkernel_generic()`に処理を委ねる。

```c
static void __init arch_reserve_crashkernel(void)
{
    unsigned long long crash_base, crash_size, low_size = 0, cma_size = 0;
    bool high = false;

    ret = parse_crashkernel(boot_command_line, memblock_phys_mem_size(),
                &crash_size, &crash_base, &low_size, &cma_size, &high);
    ...
    reserve_crashkernel_generic(crash_size, crash_base, low_size, high);
    reserve_crashkernel_cma(cma_size);
}
```

`reserve_crashkernel_generic()`は`memblock_phys_alloc_range()`を使って適切な領域を探し、予約する。

```c
crash_base = memblock_phys_alloc_range(crash_size, CRASH_ALIGN,
                                       search_base, search_end);
```

割り当てが済むと、その領域は`crashk_res`に記録され、iomemのリソースツリーに登録される。

```c
crashk_res.start = crash_base;
crashk_res.end   = crash_base + crash_size - 1;
insert_resource(&iomem_resource, &crashk_res);
```

予約されたメモリは、誤って書き込まれないよう実行中のカーネルのリニアマップから取り除かれる。`kexec_load()`はキャプチャカーネルのイメージとinitrdをこの領域内に配置し、パニック時には`crash_kexec()`がそこへジャンプする。

### 構文のオプション {#syntax-options}

| 構文 | 意味 |
|---|---|
| `crashkernel=256M` | 256 MBを予約し、アドレスはカーネルが選ぶ |
| `crashkernel=256M@512M` | 物理オフセット512 MBの位置に256 MBを予約する |
| `crashkernel=256M,high` | 4 GBより上のメモリを優先する |
| `crashkernel=256M,high crashkernel=64M,low` | high側256 MB + DMAでアクセス可能な64 MBの領域 |
| `crashkernel=512M-2G:64M,2G-:128M` | サイズ範囲指定: RAMが512 M〜2 Gなら64 MB、2 Gを超えるなら128 MB |

!!! tip
    x86-64では、最初の低位メモリでの探索が失敗すると、カーネルは自動的に高位メモリで再試行し、キャプチャカーネル内のDMA対応ハードウェアの要件を満たすため、4 GB未満に小さな`crashk_low_res`領域を割り当てる。

### タイミング {#timing}

`setup_arch()`では、`arch_reserve_crashkernel()`が（ACPI SRATを解析する）`initmem_init()`の*後*に呼ばれる点に注意してほしい。これにより、予約がホットプラグ可能なメモリ領域を避けるようになっている。

```c
initmem_init();
dma_contiguous_reserve(max_pfn_mapped << PAGE_SHIFT);

/*
 * SRAT を解析した後で crash kernel 用のメモリを予約し、
 * ホットプラグ可能なメモリを消費しないようにする。
 */
arch_reserve_crashkernel();
```

---

## CMAによる予約 {#cma-reservations}

CMA（Contiguous Memory Allocator）はブート時に物理的に連続した領域を切り出すが、ハードな予約とは異なり、その領域は無駄にならない。通常運用中、カーネルはCMA領域にmovableなpage（ページキャッシュ、匿名メモリ）を配置する。デバイスドライバが大きな連続DMAバッファを要求すると、カーネルはそれらのmovableなpageを外へ移動（マイグレーション）し、空になった連続範囲をドライバに引き渡す。

x86では、`setup_arch()`が`dma_contiguous_reserve(max_pfn_mapped << PAGE_SHIFT)`を呼び、デフォルトのCMA領域のサイズを決めて予約する。

CMAのライフサイクル、`dma_contiguous_reserve()`、`cma_declare_contiguous()`、`alloc_contig_range()`についての詳しい解説は[CMA](cma.md)を参照のこと。

---

## 実行時の予約 {#runtime-reservations}

`memblock_free_all()`が予約されていないpageをバディアロケータに引き渡すと、予約をめぐる状況は根本的に変わる。任意の物理範囲に対する**`memblock_reserve()`の実行時版は存在しない**。memblockの配列はすでに解放されているかもしれず、バディアロケータには初期化後に任意の範囲を立ち入り禁止としてマークするという概念がない。

実行時にメモリを確保しておくには、カーネルはそれを割り当て、割り当てたまま保持しなければならない。

```c
/* page を割り当てて固定する — 決して解放しない */
struct page *page = alloc_pages(GFP_KERNEL, order);
```

`alloc_pages()`を呼ぶと、そのpageは空きプールから取り除かれる。カーネルが参照を保持し、`__free_pages()`を呼ばない限り、それらのpageは他の用途に使えないままとなる。

!!! warning
    実行時に特定の物理アドレスを「予約」する方法はない。ドライバやサブシステムが特定の物理範囲を必要とする場合は、`early_param`ハンドラやアーキテクチャのセットアップコードから`memblock_reserve()`を使ってブート時に予約するか、`MEMBLOCK_NOMAP`を使う永続メモリ（pmem）のような仕組みを通してマップしなければならない。

### メモリホットプラグ {#memory-hotplug}

実行時にバディアロケータが利用できる物理メモリを変える唯一の仕組みが、メモリホットプラグである。対応ハードウェア上で新しいDIMMスロットを追加したり、既存のものを取り外したりする。これにより`memblock.memory`とバディアロケータの管理範囲が拡大（または縮小）する。`add_memory()`、`remove_memory()`、`MEMBLOCK_HOTPLUG`フラグの詳細については、メモリホットプラグのドキュメントを参照のこと。

---

## ZONE_MOVABLEとmovable_node {#zonemovable-and-movablenode}

ハードな予約以外にも、カーネルはよりソフトな形のメモリ隔離をサポートしている。それは、あるzoneやNUMAノード全体のメモリを`ZONE_MOVABLE`に配置することを保証するものである。`ZONE_MOVABLE`内のpageはmovableな割り当てにしか使われないため、それらを外へマイグレーションできることが保証され、このzoneはメモリのオフライン化やホットプラグに適したものとなる。

カーネルコマンドラインパラメータ`movable_node`がこの動作を有効にする。これは`mm/memory_hotplug.c`の`cmdline_parse_movable_node()`で処理される。

```c
static int __init cmdline_parse_movable_node(char *p)
{
    movable_node_enabled = true;
    return 0;
}
early_param("movable_node", cmdline_parse_movable_node);
```

`movable_node_is_enabled()`がtrueを返すと、`mm_init.c`は`MEMBLOCK_HOTPLUG`フラグの付いたmemblock領域を走査し、そのPFN範囲を`ZONE_NORMAL`ではなく`ZONE_MOVABLE`に配置する。これにより、カーネルがそのノードにマイグレーション不可能な割り当て（カーネルコード、固定されたメモリ）を置くことが防がれ、ノード全体をオフライン化して取り外せるようになる。

```
zone_movable_pfn[nid] = min(usable_startpfn, zone_movable_pfn[nid]);
```

`ZONE_MOVABLE`は、`movable_node`なしでも`kernelcore=`と`movablecore=`パラメータを通じて使われる。これらは各ノードのメモリの一部をmovable zoneに分割する。

!!! note "メモリ階層化"
    `movable_node`はCXLのメモリ階層化構成でも使われ、低速メモリのノードを完全にmovableとしてマークする。これにより、カーネルの階層化メモリにおける昇格／降格の仕組みが、高速（ローカルDRAM）層と低速（CXL接続）層の間でpageを移動できるようになる。詳細はCXLメモリ階層化のドキュメントを参照のこと。

---

## まとめ: 予約メカニズムの一覧 {#summary-reservation-mechanisms-at-a-glance}

```
Boot time                          Runtime
─────────────────────────────────────────────────────────────
memblock_reserve()                 alloc_pages() + hold
  └─ reserves arbitrary range      └─ allocate and never free
  └─ buddy allocator never sees it

memblock_reserve_kern()            memory hotplug
  └─ like above + RSRV_KERN flag   └─ adds/removes whole regions

memblock_phys_alloc()
  └─ allocates AND reserves

memblock_mark_nomap()
  └─ reserves + excludes from
     kernel direct map

ZONE_MOVABLE / movable_node
  └─ soft reservation: isolates
     zone for movable allocs only
─────────────────────────────────────────────────────────────
Visible in:
  /proc/iomem                      (resource tree, always)
  /sys/kernel/debug/memblock/reserved  (if CONFIG_DEBUG_FS)
  dmesg | grep -i reserved
```

## 参考文献 {#further-reading}

- [カーネルドキュメント: Memory hotplug](https://docs.kernel.org/admin-guide/mm/memory-hotplug.html) — `MEMBLOCK_HOTPLUG`領域、`movable_node`、および予約が実行時のメモリ追加／削除とどう相互作用するかを解説している
- [`Documentation/admin-guide/mm/memory-hotplug.rst`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/admin-guide/mm/memory-hotplug.rst) — ブート時のZONE_MOVABLEの分割を決める`kernelcore=`と`movablecore=`パラメータ
- [`mm/memblock.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memblock.c) — memblockの中核実装: `memblock_reserve()`、`memblock_mark_nomap()`、`memblock_free_all()`
- [`arch/x86/kernel/setup.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/setup.c) — x86のブート時予約をすべて順に統括する`setup_arch()`
- [`kernel/crash_reserve.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/crash_reserve.c) — `reserve_crashkernel_generic()`の実装
- [LWN: memblock and early memory management](https://lwn.net/Articles/438225/) — memblockが以前のbootmemアロケータをどのように置き換えたか、そしてブート時の割り当てに平坦な配列がなぜうまく機能するか（2011年）
- [memblock.md](memblock.md) — memblockアロケータのライフサイクル、領域のマージ、バディアロケータへの移行についての詳しい解説
- [cma.md](cma.md) — memblockの予約フェーズ中に`dma_contiguous_reserve()`がCMA領域をどのように切り出すか
- [memory-hotplug.md](memory-hotplug.md) — ブート時予約の実行時版: `memblock_free_all()`の後に物理メモリを追加・削除する
