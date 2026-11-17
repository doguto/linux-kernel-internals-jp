# x86_64のブート時ページテーブル設定 {#x86_64-boot-page-table-setup}

> x86_64上でLinuxカーネルが自身のアドレス空間をゼロから構築する方法 — 最初のアセンブリ命令から、ダイレクト物理マップ、KASLR、5レベルページングまで

!!! note "対象アーキテクチャ"
    このページはx86_64のみを扱う。ARM64、RISC-V、その他のアーキテクチャは異なるブート時ページングの手順を持つ。Linuxがアーキテクチャ横断で使う汎用のページテーブル抽象化については[ページテーブル](page-tables.md)を参照。

## 主要なソースファイル {#key-source-files}

| ファイル | 役割 |
|------|------|
| `arch/x86/kernel/head_64.S` | エントリポイント、初期のページテーブルデータ（`early_top_pgt`、`init_top_pgt`、`level3_kernel_pgt`） |
| `arch/x86/boot/startup/map_kernel.c` | `__startup_64()` — ページテーブルの物理アドレスを修正する最初のCコード |
| `arch/x86/mm/init.c` | `init_mem_mapping()`、`memory_map_top_down()`、`memory_map_bottom_up()` |
| `arch/x86/mm/init_64.c` | `paging_init()`、`mem_init()` |
| `arch/x86/mm/kaslr.c` | `kernel_randomize_memory()` — 仮想レイアウトのランダム化 |
| `arch/x86/lib/kaslr.c` | `kaslr_get_random_long()` — ハードウェアエントロピー（RDRAND、TSC、i8254） |
| `arch/x86/boot/compressed/kaslr.c` | 展開時の物理KASLR |
| `arch/x86/include/asm/page_64_types.h` | `__PAGE_OFFSET_BASE_L4`、`__START_KERNEL_map`、`KERNEL_IMAGE_SIZE` |
| `arch/x86/include/asm/pgtable_64_types.h` | `__VMALLOC_BASE_L4`、`VMALLOC_SIZE_TB_L4`、ページテーブルのレベル定数 |
| `arch/x86/include/asm/pgtable_64.h` | `init_top_pgt[]`の宣言、`swapper_pg_dir`エイリアス |
| `Documentation/arch/x86/x86_64/mm.rst` | 4レベルおよび5レベルページングの公式な仮想メモリマップ |

---

## ブート時ページングの問題 {#the-boot-paging-problem}

カーネルが`arch/x86/kernel/head_64.S`の`startup_64`で最初に制御を受け取るとき、CPUはすでに64ビットロングモードにある。しかしページテーブルはカーネル自身のものではない。ブートローダまたはUEFIファームウェアが一時的なアイデンティティマッピング（仮想アドレス == 物理アドレス）を作成しており、それはカーネルイメージをロードして展開するのに十分なメモリをカバーするが、それ以上はカバーしない。

カーネルが通常どおり動作できるようになる前に、次のことをしなければならない:

1. コンパイル時に組み込まれた初期のページテーブルを、物理ロードアドレスに合わせて修正する
2. 自身のコードをカバーするアイデンティティマッピングを構築し、物理アドレスから仮想アドレスへの切り替えで命令の途中でCPUがクラッシュしないようにする
3. カーネルのテキストとデータを高位の仮想アドレス（`0xffffffff80000000`以上）にマップする
4. ダイレクト物理メモリマップ、すなわちすべてのRAMをカーネル仮想空間にマップした完全なマッピングを構築する
5. vmalloc空間、vmemmap、fixmapの各領域を設定する
6. 必要に応じて、これらすべての領域をKASLRでランダム化する

難しいのは、これらの多くを、アロケータが存在する前、`printk`が動作する前、例外ハンドラが完全に整う前に、アセンブリで宣言された静的に割り当てられたページテーブル配列だけを使って行わなければならないことである。

---

## head_64.Sにおける初期ブートのページテーブル {#early-boot-page-tables-in-head_64s}

### 静的配列 {#the-static-arrays}

`arch/x86/kernel/head_64.S`は、`.data`セクションと`__INITDATA`セクションにいくつかのページテーブル配列を宣言している。これらは最初の命令の時点から利用できる唯一のページテーブルである:

```asm
/* arch/x86/kernel/head_64.S */

SYM_DATA_START_PTI_ALIGNED(early_top_pgt)
    .fill   511,8,0
    .quad   level3_kernel_pgt - __START_KERNEL_map + _PAGE_TABLE_NOENC
    /* [CONFIG_MITIGATION_PAGE_TABLE_ISOLATION の場合は + PTI_USER_PGD_FILL のパディング] */
SYM_DATA_END(early_top_pgt)

SYM_DATA_START_PTI_ALIGNED(init_top_pgt)
    .fill   512,8,0
    /* [CONFIG_MITIGATION_PAGE_TABLE_ISOLATION の場合は + PTI_USER_PGD_FILL のパディング] */
SYM_DATA_END(init_top_pgt)

SYM_DATA_START_PAGE_ALIGNED(level4_kernel_pgt)  /* 5レベルモードでのみ使用 */
    .fill   511,8,0
    .quad   level3_kernel_pgt - __START_KERNEL_map + _PAGE_TABLE_NOENC
SYM_DATA_END(level4_kernel_pgt)

SYM_DATA_START_PAGE_ALIGNED(level3_kernel_pgt)
    .fill   L3_START_KERNEL,8,0   /* カーネルの PUD スロットまでの空エントリ */
    .quad   level2_kernel_pgt - __START_KERNEL_map + _KERNPG_TABLE_NOENC
    .quad   level2_fixmap_pgt - __START_KERNEL_map + _PAGE_TABLE_NOENC
SYM_DATA_END(level3_kernel_pgt)

SYM_DATA_START_PAGE_ALIGNED(level2_kernel_pgt)
    /* カーネルイメージ領域のための 2MB huge page の PMD エントリ */
    PMDS(0, __PAGE_KERNEL_LARGE_EXEC, KERNEL_IMAGE_SIZE/PMD_SIZE)
SYM_DATA_END(level2_kernel_pgt)
```

`L3_START_KERNEL`は`pud_index(__START_KERNEL_map)`であり、カーネルを`level3_kernel_pgt`のスロット510に配置する。スロット511はfixmap領域をマップする。

`init_top_pgt`は最初は完全に空である。`__startup_64`の実行後に恒久的なカーネルPGDになる。x86_64では、`swapper_pg_dir`は単に`init_top_pgt`のエイリアスである（`arch/x86/include/asm/pgtable_64.h`で定義）:

```c
extern pgd_t init_top_pgt[];
#define swapper_pg_dir init_top_pgt
```

これらと並んで、64個の作業用PMDテーブルのプール（`early_dynamic_pgts[EARLY_DYNAMIC_PAGE_TABLES]`）が宣言されている。`__startup_64()`はこのプールから切り出して一時的なアイデンティティマッピングを構築する。

### `__startup_64()`が行うこと {#what-_startup64-does}

最初に実行されるC関数は、`arch/x86/boot/startup/map_kernel.c`の`__startup_64()`である。これは`CR3`にカーネル自身のテーブルがロードされる前に、`startup_64`から呼ばれる。

```c
/* arch/x86/boot/startup/map_kernel.c */
unsigned long __init __startup_64(unsigned long p2v_offset,
                                  struct boot_params *bp)
```

カーネルは（KASLRやブートローダの配置により）コンパイル時の想定とは異なる物理アドレスにロードされることがあるため、事前に構築されたページテーブル配列内のアドレスはすべて誤っている。`__startup_64()`はそれらを修正する:

1. **`load_delta`を計算する** — コンパイル時に組み込まれた仮想アドレス`__START_KERNEL_map`と実際の物理アドレスとの差を、加算値として表したもの。

2. **`early_top_pgt`を修正する** — `level3_kernel_pgt`を指すPGDエントリ（スロット511、`__START_KERNEL_map`領域）に`load_delta`を加える。

3. **`level3_kernel_pgt`を修正する** — 値の入っている2つのPUDエントリ（カーネルイメージとfixmap）に`load_delta`を加える。

4. **`level2_fixmap_pgt`を修正する** — fixmapのPMDエントリを再配置する。

5. **`early_dynamic_pgts`内にアイデンティティマップを構築する** — プールからPUDテーブルとPMDテーブルを割り当て、`early_top_pgt`に接続する。アイデンティティマッピングはカーネルイメージ自体を含む物理pageをカバーするので、`CR3`の再ロード中もCPUは命令をフェッチし続けられる。これらのエントリには意図的にグローバル（`_PAGE_GLOBAL`）ビットが**セットされていない**ので、後でカーネルが`init_top_pgt`に切り替えるときにフラッシュされる。

6. **`level2_kernel_pgt`を検証する** — カーネルテキストより前と`_end`より後のpageは`_PAGE_PRESENT`がクリアされ、投機的アクセスが予約済みメモリに到達できないようにする。

### 2段階のCR3切り替え {#the-two-stage-cr3-switch}

```
startup_64 entry
    │
    ├─ call __startup_64()           ← fix physical addresses, build identity map
    │
    ├─ mov early_top_pgt → CR3       ← activate kernel's tables (identity map live)
    │
    └─ jmp common_startup_64         ← jump to virtual address (identity map used here)
           │
           ├─ secondary_startup_64:
           │    mov init_top_pgt → CR3   ← switch to permanent PGD
           │    (identity map gone)
           │
           └─ start_kernel()
```

`early_top_pgt`はこの短い期間にだけ使われる。`secondary_startup_64`が`init_top_pgt`を`CR3`にロードすると、アイデンティティマッピングは消える。この時点以降、すべてのコードはコンパイル時に組み込まれたカーネル仮想アドレスで実行される。

---

## x86_64の仮想メモリレイアウト {#the-x86_64-virtual-memory-layout}

x86_64は64ビットのアドレス空間をカノニカル境界で分割する。4レベル（48ビット）のレイアウトが基本であり、5レベル（57ビット）については次の節で説明する。

### 4レベルページング（48ビット仮想アドレス） {#4-level-paging-48-bit-virtual-addresses}

以下の定数は`arch/x86/include/asm/page_64_types.h`と`arch/x86/include/asm/pgtable_64_types.h`からのものである。KASLRは`page_offset_base`、`vmalloc_base`、`vmemmap_base`をそれぞれの範囲内でずらすことができる（後述の[KASLR](#kaslr-kernel-address-space-layout-randomization)を参照）。

```
Virtual address range                    Size   Region
─────────────────────────────────────────────────────────────────────────
0x0000000000000000 – 0x00007ffffffff000  ~128TB  User space
                   [ non-canonical hole ]
0xffff800000000000 – 0xffff87ffffffffff    8TB  Guard hole (hypervisor)
0xffff880000000000 – 0xffff887fffffffff  0.5TB  LDT remap (PTI)
0xffff888000000000 – 0xffffc87fffffffff   64TB  Direct physical map
                                                (page_offset_base,
                                                 __PAGE_OFFSET_BASE_L4 =
                                                 0xffff888000000000)
0xffffc88000000000 – 0xffffc8ffffffffff  0.5TB  Unused hole
0xffffc90000000000 – 0xffffe8ffffffffff   32TB  vmalloc / ioremap
                                                (vmalloc_base,
                                                 __VMALLOC_BASE_L4 =
                                                 0xffffc90000000000)
0xffffe90000000000 – 0xffffe9ffffffffff    1TB  Unused hole
0xffffea0000000000 – 0xffffeaffffffffff    1TB  vmemmap (struct page array)
0xffffec0000000000 – 0xfffffbffffffffff   16TB  KASAN shadow
0xfffffe0000000000 – 0xfffffe7fffffffff  0.5TB  cpu_entry_area
0xffffff0000000000 – 0xffffff7fffffffff  0.5TB  %esp fixup stacks
0xffffffef00000000 – 0xfffffffeffffffff   64GB  EFI runtime services
0xffffffff00000000 – 0xffffffff7fffffff    2GB  Unused hole
0xffffffff80000000 – 0xffffffff9fffffff  512MB  Kernel text+data
                                                (__START_KERNEL_map =
                                                 0xffffffff80000000)
0xffffffffa0000000 – 0xfffffffffeffffff 1520MB  Module mapping space
   FIXADDR_START   – 0xffffffffff5fffff ~0.5MB  Fixmap
0xffffffffff600000 – 0xffffffffff600fff    4KB  vsyscall ABI
─────────────────────────────────────────────────────────────────────────
```

ソースからの主要な定数:

| シンボル | 値 | ファイル |
|--------|-------|------|
| `__PAGE_OFFSET_BASE_L4` | `0xffff888000000000` | `page_64_types.h` |
| `__PAGE_OFFSET_BASE_L5` | `0xff11000000000000` | `page_64_types.h` |
| `__START_KERNEL_map` | `0xffffffff80000000` | `page_64_types.h` |
| `__VMALLOC_BASE_L4` | `0xffffc90000000000` | `pgtable_64_types.h` |
| `VMALLOC_SIZE_TB_L4` | `32` | `pgtable_64_types.h` |
| `__VMEMMAP_BASE_L4` | `0xffffea0000000000` | `pgtable_64_types.h` |
| `KERNEL_IMAGE_SIZE` | 1 GB（KASLRあり） / 512 MB（KASLRなし） | `page_64_types.h` |

!!! note "KASLRはベースアドレスをずらす"
    `__PAGE_OFFSET`は`__PAGE_OFFSET_BASE_L4`ではなく、実行時の変数`page_offset_base`である。同様に`VMALLOC_START`は`vmalloc_base`、`VMEMMAP_START`は`vmemmap_base`である。`_BASE_`付きの定数はKASLRなしの場合のデフォルト値である。`CONFIG_RANDOMIZE_MEMORY=y`の場合、実際の実行時アドレスは異なる。

---

## `init_mem_mapping()` — ダイレクト物理マップの構築 {#initmemmapping-building-the-direct-physical-map}

ダイレクト物理マップは、カーネルが`page_offset_base`からの単純なポインタ演算のオフセットでRAMの任意のバイトをアドレス指定できるようにするものである。これがなければ、カーネルは静的な`level2_kernel_pgt`のPMDがカバーするメモリ、つまりカーネルイメージ周辺のおよそ1 GBにしかアクセスできない。

`arch/x86/mm/init.c`の`init_mem_mapping()`は、`arch/x86/kernel/setup.c`の1126行目で`setup_arch()`から呼ばれる。これは`kernel_randomize_memory()`がすでに各領域のベースアドレスを確定させた後である。

```c
/* arch/x86/mm/init.c */
void __init init_mem_mapping(void)
{
    unsigned long end = max_pfn << PAGE_SHIFT;   /* 最も高い物理アドレス */

    pti_check_boottime_disable();
    probe_page_size_mask();   /* 使用するページサイズ 4KB / 2MB / 1GB を決める */
    setup_pcid();

    /* ISA 範囲 [0, 1MB) はメモリホールに関係なく常にマップされる */
    init_memory_mapping(0, ISA_END_ADDRESS, PAGE_KERNEL);

    init_trampoline();        /* SMP の AP トランポリンには専用のマッピングが必要 */

    if (memblock_bottom_up()) {
        /* ボトムアップ: ページテーブルがカーネルより上に置かれるよう、先にカーネルより上をマップ */
        memory_map_bottom_up(kernel_end, end);
        memory_map_bottom_up(ISA_END_ADDRESS, kernel_end);
    } else {
        /* トップダウン: RAM の最上部から下に向かってマップ */
        memory_map_top_down(ISA_END_ADDRESS, end);
    }

    load_cr3(swapper_pg_dir);
    __flush_tlb_all();
}
```

トップダウンとボトムアップのどちらを選ぶかは`memblock_bottom_up()`によって決まり、これはページテーブル用のメモリ自体をmemblockがどう割り当てるよう設定されているかを反映する。KASLRはボトムアップモードを強制し、ページテーブルの割り当てが低い物理アドレスから行われるようにして、ランダムに配置されたカーネルイメージとの衝突を避ける。

内部で実際の仕事をするのは`init_range_memory_mapping()`であり、これは`init_memory_mapping()`をチャンク単位で呼ぶ。`init_memory_mapping()`はページテーブル階層を下へたどり、必要に応じてmemblockからPUD、PMD、PTEテーブルを割り当て、物理範囲とアラインメントが許す限り大きなpage（2 MBのPMDエントリまたは1 GBのPUDエントリ）を使う。

`init_mem_mapping()`が戻った後、`swapper_pg_dir`（すなわち`init_top_pgt`）は、`page_offset_base`を通して変換されたすべての物理RAMをカバーするエントリを持つ。カーネルはこれで、`phys_addr + page_offset_base`を計算することで任意の物理メモリを読み書きできる。

---

## KASLR: カーネルアドレス空間配置のランダム化 {#kaslr-kernel-address-space-layout-randomization}

x86_64のKASLRには2つの独立した構成要素がある:

### 1. 物理KASLR — イメージの配置 {#1-physical-kaslr-image-placement}

これは展開中、つまりメインのカーネルが動き出すずっと前に行われる。`arch/x86/boot/compressed/kaslr.c`はファームウェアのメモリマップ（e820またはEFI memmap）をスキャンし、カーネルイメージが収まる2 MBアラインの候補スロットを集め、`kaslr_get_random_long("Physical")`を使ってその1つを選ぶ。選ばれたオフセットは`boot_params`に格納され、`__startup_64()`が`p2v_offset`引数として利用する。

物理KASLRは`CONFIG_RANDOMIZE_BASE`で制御され、カーネルコマンドラインに`nokaslr`を渡すと実行時に無効化される。

### 2. メモリレイアウトKASLR — 仮想領域のランダム化 {#2-memory-layout-kaslr-virtual-region-randomization}

カーネルイメージが動き出した後、`arch/x86/mm/kaslr.c`の`kernel_randomize_memory()`が、3つの主要なカーネルメモリ領域の仮想ベースアドレスをランダム化する:

```c
/* arch/x86/mm/kaslr.c */
static __initdata struct kaslr_memory_region {
    unsigned long *base;   /* 実行時のベース変数へのポインタ */
    unsigned long *end;    /* 省略可能な終端ポインタ */
    unsigned long size_tb; /* TB 単位の領域サイズ */
} kaslr_regions[] = {
    { .base = &page_offset_base, .end = &direct_map_physmem_end },  /* ダイレクトマップ */
    { .base = &vmalloc_base,     },                                  /* vmalloc */
    { .base = &vmemmap_base,     },                                  /* vmemmap */
};
```

利用可能なエントロピーは、これらの領域とその隣接領域（`CPU_ENTRY_AREA_BASE`まで）との間の余裕分である。ランダム化はPGD/P4D/PUDの粒度で行われ（各領域のベースはPUD境界に丸められる）、典型的な構成では領域ごとにおよそ30,000通りの配置が可能になる。

`kernel_randomize_memory()`は`kaslr_get_random_long("Memory")`でPRNGにシードを与える。エントロピー関数自体は`arch/x86/lib/kaslr.c`にあり、ハードウェアのソースを混ぜ合わせる:

```c
/* arch/x86/lib/kaslr.c */
unsigned long kaslr_get_random_long(const char *purpose)
{
    unsigned long random = get_boot_seed();   /* イメージの kaslr_offset() */

    if (has_cpuflag(X86_FEATURE_RDRAND)) {
        rdrand_long(&raw);
        random ^= raw;           /* 利用可能なら RDRAND */
    }

    if (has_cpuflag(X86_FEATURE_TSC)) {
        raw = rdtsc();
        random ^= raw;           /* タイミングのゆらぎとして TSC */
    } else {
        random ^= i8254();       /* フォールバックとしてレガシーな i8254 タイマー */
    }

    /* ビット拡散のための循環乗算 */
    asm(_ASM_MUL "%3" ...);
    return random;
}
```

メモリレイアウトKASLRは`CONFIG_RANDOMIZE_MEMORY`で制御され、`CONFIG_KASAN`が有効な場合は無効になる（`kaslr_memory_enabled()`で確認される）。

`kernel_randomize_memory()`は`setup_arch()`の1047行目で呼ばれ、これは1126行目の`init_mem_mapping()`より**前**である。ページテーブルが構築される時点では、最終的なベースアドレスはすでに確定している。

!!! warning "KASANはメモリKASLRを無効にする"
    KASANは固定されたシャドウメモリのレイアウトを必要とするため、`CONFIG_KASAN=y`の場合、`kaslr_memory_enabled()`は`false`を返す。物理KASLR（`CONFIG_RANDOMIZE_BASE`）はこれとは独立しており、KASANがあっても有効なままである。

---

## 5レベルページング（LA57） {#5-level-paging-la57}

LA57機能ビットをサポートするx86_64 CPUは、5段目のページテーブルを使って57ビットの仮想アドレスを提供する。カーネルオプションは`CONFIG_X86_5LEVEL`である。

### 検出と有効化 {#detection-and-activation}

5レベルページングは展開中に検出され、有効化される。CPUがサポートしており、カーネルが`CONFIG_X86_5LEVEL`付きでビルドされていれば、展開処理が`CR4.LA57`をセットする。`startup_64`が実行される時点では、LA57はすでに有効になっているか、なっていないかのどちらかである。

`__startup_64()`は実行時に確認する:

```c
/* arch/x86/boot/startup/map_kernel.c */
static inline bool check_la57_support(void)
{
    if (!(native_read_cr4() & X86_CR4_LA57))
        return false;

    __pgtable_l5_enabled = 1;
    pgdir_shift           = 48;   /* 4レベルモードでは 39 だった */
    ptrs_per_p4d          = 512;  /* P4D レベルはもう折り畳まれない */

    return true;
}
```

グローバル変数`__pgtable_l5_enabled`（`arch/x86/include/asm/pgtable_64_types.h`で宣言）は、その後ブートの残りの部分を通じて`pgtable_l5_enabled()`インライン関数によって使われる。初期ブートの後、通常のカーネルは代わりに`cpu_feature_enabled(X86_FEATURE_LA57)`を使う。

### 構造上の違い {#structural-difference}

4レベルモードではP4Dレベルは**折り畳まれて**おり、PGDエントリが直接PUDを指す。5レベルモードではP4Dが実在する:

```
4-level:  CR3 → PGD[9] → PUD[9] → PMD[9] → PTE[9] → page offset[12]  = 48 bits
5-level:  CR3 → PGD[9] → P4D[9] → PUD[9] → PMD[9] → PTE[9] → offset[12] = 57 bits
```

5レベルモードが有効な場合、`__startup_64()`は`early_top_pgt`（PGD）と`level3_kernel_pgt`（PUD）の間に`level4_kernel_pgt`（実在するP4Dページ）を挿入する:

```c
if (la57) {
    p4d = (p4dval_t *)rip_rel_ptr(level4_kernel_pgt);
    p4d[MAX_PTRS_PER_P4D - 1] += load_delta;   /* P4D エントリを修正 */

    pgd[pgd_index(__START_KERNEL_map)] = (pgdval_t)p4d | _PAGE_TABLE;
}
```

### 仮想レイアウトの変化 {#virtual-layout-changes}

57ビットアドレスでは、レイアウトは劇的に変わる。`Documentation/arch/x86/x86_64/mm.rst`より:

| 領域 | 4レベルのベース | 5レベルのベース | サイズの変化 |
|--------|-------------|-------------|-------------|
| ダイレクト物理マップ | `0xffff888000000000`（-119.5 TB） | `0xff11000000000000`（-59.75 PB） | 64 TB → 32 PB |
| vmalloc/ioremap | `0xffffc90000000000`（-55 TB） | `0xffa0000000000000`（-24 PB） | 32 TB → 12.5 PB |
| vmemmap | `0xffffea0000000000`（-22 TB） | `0xffd4000000000000`（-11 PB） | 1 TB → 0.5 PB |
| ユーザー空間 | 約128 TB | 約64 PB | 512倍 |

`VMALLOC_SIZE_TB_L5`定数は12,800（TB）であり、`VMALLOC_SIZE_TB_L4`の32（TB）と比べて大きい。

---

## ページテーブルの最終化: `paging_init()`とその後 {#finalizing-page-tables-paging_init-and-beyond}

### `setup_arch()`における呼び出し順序 {#call-sequence-in-setup_arch}

`setup_arch()`（`arch/x86/kernel/setup.c`）におけるページテーブルのライフサイクルは次のように進む:

```
setup_arch()
  │
  ├─ kernel_randomize_memory()     # 仮想領域のベースアドレスを確定
  │
  ├─ init_mem_mapping()            # init_top_pgt にダイレクト物理マップを構築
  │    ├─ memory_map_top_down()    # または
  │    └─ memory_map_bottom_up()
  │    └─ load_cr3(swapper_pg_dir) # 完成したマッピングで CR3 を再ロード
  │
  └─ x86_init.paging.pagetable_init()
       └─ native_pagetable_init()
            └─ paging_init()       # arch/x86/mm/init_64.c
```

x86_64の`paging_init()`（`arch/x86/mm/init_64.c:834`）は意図的に最小限になっている。重い仕事はすでに`init_mem_mapping()`が済ませているので、ゾーンの初期化が正しく設定できるようNUMAノードの状態をクリアするだけである。

`x86_init.paging.pagetable_init()`が戻った後、`setup_arch()`はKASANの初期化（`kasan_init()`）に進み、その後で恒久的なページテーブルが整う。

### `mem_init()` — buddyアロケータへの引き渡し {#mem_init-handing-off-to-the-buddy-allocator}

`mem_init()`（`arch/x86/mm/init_64.c:1368`）は`init/main.c`の`mm_core_init()`から呼ばれる。この時点で、`memblock_free_all()`はすでに予約されていないすべてのメモリをbuddyアロケータに移している。`mem_init()`自体は次を行う:

- `after_bootmem = 1`をセットし、初期ブート時の割り当てにとっての後戻りできない地点を示す
- `register_page_bootmem_info()`を呼び、メモリホットプラグの追跡のためにmemmapのpageを登録する
- `after_bootmem`をマークして初期ブート時の割り当てを終了する

### 恒久的なテーブルと初期のテーブル {#permanent-vs-early-tables}

| テーブル | 寿命 | 目的 |
|-------|----------|---------|
| `early_top_pgt` | `secondary_startup_64`が`init_top_pgt`をロードするまでのみ | アイデンティティマップ + カーネルを持つ |
| `early_dynamic_pgts` | 同上 — `__startup_64`の作業領域 | アイデンティティマップのPUD/PMDテーブル |
| `init_top_pgt`（`swapper_pg_dir`） | 恒久的 — `init_mm.pgd` | `init_mem_mapping`後の完全なカーネルPGD |
| `level3_kernel_pgt` / `level2_kernel_pgt` | 恒久的 | カーネルのテキスト/データのPUDおよびPMDテーブル |

---

## ブート時ページテーブルのデバッグ {#debugging-boot-page-tables}

### `CONFIG_PTDUMP_DEBUGFS` {#configptdumpdebugfs}

`CONFIG_PTDUMP_DEBUGFS=y`の場合、カーネルは`/sys/kernel/debug/page_tables/`の下にdebugfsファイルを登録する（`arch/x86/mm/debug_pagetables.c`で実装）:

```
/sys/kernel/debug/page_tables/kernel         # init_mm のページテーブル
/sys/kernel/debug/page_tables/current_kernel # 現在のプロセスのカーネルマッピング
/sys/kernel/debug/page_tables/current_user   # 現在のプロセスのユーザーマッピング (PTI)
/sys/kernel/debug/page_tables/efi            # EFI ランタイムのページテーブル
```

`kernel`を読むと`ptdump_walk_pgd_level_debugfs()`が呼ばれて`init_top_pgt`をたどり、マップされたすべての範囲を、パーミッションビット（RW、NX、Globalなど）と使用されているページサイズ（4K、2M、1G）とともに出力する。

### W+Xチェック {#wx-check}

`mark_rodata_ro()`がパーミッションビットを確定させた後、カーネルは書き込み可能かつ実行可能なpageが存在しないことを確認する。結果はdmesgに記録される:

```
x86/mm: Checked W+X mappings: passed, no W+X pages found.
```

失敗した場合は:

```
x86/mm: Found insecure W+X mapping at address <symbol>
x86/mm: Checked W+X mappings: FAILED, N W+X pages found.
```

これは`arch/x86/mm/dump_pagetables.c`で実装されている。

### その他の有用なインターフェース {#other-useful-interfaces}

```bash
# ファームウェアからの物理メモリ領域 (e820 マップ)
cat /proc/iomem

# ページフォールトの統計を表示
cat /proc/vmstat | grep pgfault

# 初期マッピングのメッセージを探してカーネルログをダンプ
dmesg | grep -E "x86/mm|PAGE|mapping"

# 5レベルページングが有効か確認
grep . /sys/devices/system/cpu/cpu0/cpufreq/../topology/physical_package_id
# または /proc/cpuinfo の flags に 'la57' があるか調べる
grep la57 /proc/cpuinfo
```

### ブートパラメータ {#boot-parameters}

| パラメータ | 効果 |
|-----------|--------|
| `nokaslr` | 物理KASLRとメモリKASLRの両方を無効にする |
| `no5lvl` | CPUとカーネルがサポートしていても5レベルページングを無効にする |
| `memmap=exactmap` | e820メモリマップを上書きする（テストに有用） |

---

## まとめ {#summary}

x86_64のブート時ページングの手順は明確な段階を経て進み、各段階でシステムのより多くの部分が使えるようになる:

```
Bootloader / UEFI
  └─ Identity-mapped page tables loaded, CPU in 64-bit long mode
        │
        ▼
startup_64  (arch/x86/kernel/head_64.S)
  └─ __startup_64() fixes early_top_pgt physical addresses,
     builds identity map in early_dynamic_pgts
        │
        ▼
CR3 = early_top_pgt
  └─ Identity map + high kernel map both active
        │
        ▼
secondary_startup_64 → CR3 = init_top_pgt
  └─ Identity map gone; kernel runs at virtual addresses only
        │
        ▼
setup_arch()
  ├─ kernel_randomize_memory()   ← KASLR: randomize region bases
  └─ init_mem_mapping()          ← direct physical map built
        │
        ▼
x86_init.paging.pagetable_init() → paging_init()
  └─ NUMA zone state cleared; buddy allocator ready to take over
        │
        ▼
mem_init() → memblock_free_all()
  └─ All free memory handed to the buddy allocator
     Boot page table setup complete
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [arch/x86/kernel/head_64.S](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/head_64.S) — `startup_64`エントリポイント。静的な`early_top_pgt`、`init_top_pgt`、`level3_kernel_pgt`の宣言
- [arch/x86/boot/startup/map_kernel.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/boot/startup/map_kernel.c) — `__startup_64()`: 最初のCコード。物理アドレスを修正し、アイデンティティマップを構築する
- [arch/x86/mm/init.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/init.c) — `init_mem_mapping()`、`memory_map_top_down()`、`memory_map_bottom_up()`
- [arch/x86/mm/kaslr.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/kaslr.c) — `kernel_randomize_memory()`: ダイレクトマップ、vmalloc、vmemmapのベースアドレスをランダム化する

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/arch/x86/boot.rst`](https://docs.kernel.org/arch/x86/boot.html) — x86ブートプロトコル: カーネルにジャンプする前にブートローダが整えなければならない状態
- [`Documentation/arch/x86/x86_64/mm.rst`](https://docs.kernel.org/arch/x86/x86_64/mm.html) — 4レベルおよび5レベルページングの、正確なアドレス範囲付きの公式な仮想メモリマップ

### 関連ページ {#related-pages}

- [page-tables.md](page-tables.md) — ブート時のテーブルが組み込まれる、Linuxの汎用ページテーブル抽象化（PGD/P4D/PUD/PMD/PTE）
- [memblock.md](memblock.md) — buddyアロケータの準備が整う前にページテーブル用pageのメモリを提供する初期アロケータ

### LWNの記事 {#lwn-articles}

- [LWN: Five-level page tables](https://lwn.net/Articles/717293/) — LA57と57ビットアドレス空間拡張の紹介
- [LWN: KASLR for the kernel's virtual address space](https://lwn.net/Articles/569635/) — メモリレイアウトKASLR: 動機、エントロピー源、トレードオフ
