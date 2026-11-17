# x86-64のページテーブル {#x86-64-page-tables}

> 4レベルおよび5レベルのページング、CR3、PCID、そしてKPTI

x86-64の仮想メモリは、ページテーブルによる多段の基数木（radix tree）を使って仮想アドレスを物理アドレスに変換する。この変換を理解することは、メモリ割り当て、保護、隔離、そしてMeltdownの緩和策を理解するための基礎である。

---

## 4レベルページング: 標準的なケース {#4-level-paging-the-standard-case}

x86-64では、48ビットの仮想アドレスを使う**4レベルページング**がデフォルトである。各仮想アドレスは5つのフィールドとして解釈される。

```
48-bit virtual address (canonical form):
 63        48 47    39 38    30 29    21 20    12 11      0
 ┌───────────┬────────┬────────┬────────┬────────┬────────┐
 │ sign ext  │  PGD   │  PUD   │  PMD   │  PTE   │ offset │
 │  (16 bit) │ (9 bit)│ (9 bit)│ (9 bit)│ (9 bit)│(12 bit)│
 └───────────┴────────┴────────┴────────┴────────┴────────┘
               L4 idx   L3 idx   L2 idx   L1 idx   byte off

Terminology used in the kernel source:
  PGD = Page Global Directory  (level 4, pointed to by CR3)
  PUD = Page Upper Directory   (level 3)
  PMD = Page Middle Directory  (level 2)
  PTE = Page Table Entry       (level 1, maps a 4KB page)
```

各レベルは512（2^9）個のエントリからなる配列で、各エントリは8バイト幅である — つまり各レベルがちょうど4KBのページ1つに収まる。

ハードウェアは、仮想アドレスを変換するためにこれら4つのレベルをたどる。

```
CR3 (physical addr of PGD)
  → PGD[VA[47:39]]  (physical addr of PUD page)
    → PUD[VA[38:30]] (physical addr of PMD page)
      → PMD[VA[29:21]] (physical addr of PTE page)
        → PTE[VA[20:12]] (physical addr of 4KB page frame)
          + VA[11:0]    (byte offset within the page)
          = Physical Address
```

MMUは、メモリアクセスのたびにページテーブルをたどらずに済むよう、最近の変換結果を**TLB**（Translation Lookaside Buffer）にキャッシュする。

---

## ページテーブルエントリ（PTE）のビット配置 {#page-table-entry-pte-bit-layout}

ページテーブルの8バイトの各エントリは、定められたビットの集合を使う。

```
PTE (64-bit) format — applies to all levels (PGD, PUD, PMD, PTE):

 63  52 51      12 11 10  9  8  7  6  5  4  3  2  1  0
 ┌─────┬──────────┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
 │ NX  │  PFN     │  │G │PS│D │A │PCD│PWT│U/S│R/W│ P │
 └─────┴──────────┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘

 bit 0:  P   — Present: 1 = entry is valid; 0 = page not present (triggers #PF)
 bit 1:  R/W — Read/Write: 0 = read-only; 1 = writable
 bit 2:  U/S — User/Supervisor: 0 = kernel only; 1 = user accessible
 bit 3:  PWT — Page Write-Through (cache behavior)
 bit 4:  PCD — Page Cache Disable
 bit 5:  A   — Accessed: set by hardware on any access; used by LRU reclaim
 bit 6:  D   — Dirty: set by hardware on write; indicates page was modified
 bit 7:  PS  — Page Size: at PMD level → 2MB huge page; at PUD level → 1GB huge page
               At PTE level this bit is PAT (Page Attribute Table index bit 2)
 bit 8:  G   — Global: TLB entry not flushed on CR3 reload; used for kernel mappings
 bits 9-11:   Software-defined (kernel uses for various flags)
 bits 12-51:  PFN (Page Frame Number) — physical address >> 12
 bits 52-62:  Reserved (must be zero)
 bit 63:  NX  — No-Execute: 1 = code cannot be fetched from this page
                Requires EFER.NXE = 1 (set by kernel during boot)
```

### カーネル側でのPTE操作 {#kernel-side-pte-manipulation}

カーネルは、PTEを安全に読み取り・変更するための一連の関数を提供している。

```c
/* arch/x86/include/asm/pgtable.h */

/* フラグを確認する */
static inline int pte_present(pte_t pte)  { return pte_flags(pte) & _PAGE_PRESENT; }
static inline int pte_write(pte_t pte)    { return pte_flags(pte) & _PAGE_RW; }
static inline int pte_dirty(pte_t pte)    { return pte_flags(pte) & _PAGE_DIRTY; }
static inline int pte_young(pte_t pte)    { return pte_flags(pte) & _PAGE_ACCESSED; }
static inline int pte_exec(pte_t pte)     { return !(pte_flags(pte) & _PAGE_NX); }

/* フラグを変更する */
static inline pte_t pte_mkwrite(pte_t pte, struct vm_area_struct *vma)
{
    return pte_set_flags(pte, _PAGE_RW);
}
static inline pte_t pte_mkdirty(pte_t pte)  { return pte_set_flags(pte, _PAGE_DIRTY); }
static inline pte_t pte_mkyoung(pte_t pte)  { return pte_set_flags(pte, _PAGE_ACCESSED); }
static inline pte_t pte_wrprotect(pte_t pte) { return pte_clear_flags(pte, _PAGE_RW); }

/* アドレス変換のヘルパー */
#define __pa(x)  ((unsigned long)(x) - PAGE_OFFSET)  /* 仮想 → 物理 */
#define __va(x)  ((void *)((unsigned long)(x) + PAGE_OFFSET))  /* 物理 → 仮想 */
```

`set_pte_at(mm, addr, ptep, pte)`はPTEを不可分に書き込み、必要に応じてTLBフラッシュを処理する。

---

## 5レベルページング（LA57） {#5-level-paging-la57}

5レベルページングは、PGDの上に**P4D**（Page 4 Directory）と呼ばれる5番目のレベルを追加し、仮想アドレス空間を48ビットから**57ビット**に拡張する。

```
57-bit virtual address (5-level paging):
 63        57 56    48 47    39 38    30 29    21 20    12 11      0
 ┌───────────┬────────┬────────┬────────┬────────┬────────┬────────┐
 │ sign ext  │  PGD   │  P4D   │  PUD   │  PMD   │  PTE   │ offset │
 │  (7 bit)  │ (9 bit)│ (9 bit)│ (9 bit)│ (9 bit)│ (9 bit)│(12 bit)│
 └───────────┴────────┴────────┴────────┴────────┴────────┴────────┘
```

Linuxでは、`pgd_t`が常に最上位レベル（5レベルページングではビット56:48）であり、`p4d_t`がその次のレベル（ビット47:39）である。たどる順序は、PGD（ビット56:48） → P4D（ビット47:39） → PUD → PMD → PTEとなる。

これによりプロセスごとに128PBの仮想アドレス空間が得られる（4レベルページングでは128TB）。

5レベルページングは、カーネルのブート中に`CR4.LA57 = 1`を設定することで有効になる。カーネルはCPUIDのリーフ7、サブリーフ0、ECXのビット16でサポートを検出する。Linuxでのサポートは**Linux 4.14**で導入された。

```bash
# 5レベルページングが使われているか確認する
grep -i la57 /proc/cpuinfo     # CPUがサポートしている
dmesg | grep -i "5-level\|la57"  # カーネルが有効にした
```

カーネルソースでは、`CONFIG_X86_5LEVEL=y`でコンパイル時のサポートが有効になる。実行時には、カーネルはCPUとBIOSがそれをサポートしているかを検出し、最終的なページテーブルを用意する前の`startup_64`の中で有効にする。

---

## CR3: ページテーブルレジスタ {#cr3-the-page-table-register}

`CR3`は、現在のプロセスの最上位ページテーブルがどこにあるかをMMUに伝えるレジスタである。

```
CR3 register (64-bit):
 63      12 11      0
 ┌─────────┬─────────┐
 │  PGD PA │  PCID   │
 │ (52 bit)│ (12 bit)│
 └─────────┴─────────┘

 bits 12-63: Physical address of the PGD (page-aligned, so bits 11:0 are zero)
 bits 11:0:  PCID (Process-Context Identifier) — see below
 bit 63:     (when writing CR3) if set AND PCID is in use, skip TLB flush
```

CR3に新しい値をロードすると、現在のページテーブルが切り替わる。プロセスのコンテキストスイッチのたびに`switch_mm_irqs_off()`が呼ばれ、新しいプロセスのPGDの物理アドレスがCR3に書き込まれる。

```c
/* arch/x86/mm/tlb.c */
void switch_mm_irqs_off(struct mm_struct *prev, struct mm_struct *next,
                         struct task_struct *tsk)
{
    /* ... 正しいCR3の値を選ぶ（KPTI、PCIDを考慮する） ... */
    load_new_mm_cr3(next_pgd, new_asid, new_lam, true);
}
```

---

## ヒュージページ: 2MBと1GB {#huge-pages-2mb-and-1gb}

リーフでないページテーブルエントリの`PS`ビットがセットされていると、ハードウェアはそのエントリをリーフとして扱う — 次のテーブルレベルを指す代わりに、大きなページを直接マップする。

| PSビットの位置 | ページサイズ | 必要なアライメント |
|----------------|-----------|-------------------|
| PMD（レベル2） | 2MB | 2MBアライン |
| PUD（レベル3） | 1GB | 1GBアライン |

2MBのPMDエントリでは、仮想アドレスのビット20:0が2MBページ内のオフセットとして使われる（PTEレベルはたどらない）。PMDエントリのPFNフィールドは、2MBにアラインされた物理フレームを直接指す。

ヒュージページはTLBへの負荷を減らす: 1つのTLBエントリが4KBではなく2MBをカバーするため、同じワーキングセットに必要なエントリが少なくて済む。

カーネルは、すべての物理メモリの**ダイレクトマッピング**（physmap。`__va()`/`__pa()`で別名として参照される）に2MBのヒュージページを使う。ユーザーから見えるヒュージページは、`hugetlbfs`またはTransparent Huge Pages（THP）によって管理される。

---

## PCID: Process-Context Identifier {#pcid-process-context-identifiers}

PCIDがない場合、CR3をロードするたびに（コンテキストスイッチのたびに）TLB全体をフラッシュしなければならない — そうしないと、新しいプロセスの仮想→物理マッピングが、古いプロセスのキャッシュされた変換と衝突してしまうからである。

**PCID**（Process-Context Identifier）は、CR3のビット11:0に格納される12ビットのタグである。TLBはキャッシュした各変換とともにPCIDを保持する。新しいプロセスに切り替えるとき、そのPCIDが現在のものと異なれば、CPUは古いエントリをフラッシュせずに、新しいプロセスのキャッシュされた変換（そのPCIDでタグ付けされたもの）を使う。

```
TLB entry with PCID:
  [ PCID (12 bits) | Virtual Page Number (52 bits) ] → Physical Frame Number
```

PCIDは`CR4.PCIDE = 1`を設定することで有効になる。カーネルは、コンテキストスイッチ時のTLB全体のフラッシュを避けるためにPCIDを使う。

- 各`mm_struct`には、`pcid_asid_map[]`配列からASID（アドレス空間ID）が割り当てられる
- コンテキストスイッチ時、プロセスに有効なPCIDが割り当てられていて、その変換がキャッシュされていれば、TLBエントリを保持するために`bit 63 = 1`（フラッシュなし）でCR3がロードされる

```c
/* arch/x86/mm/tlb.c */
/*
 * ASIDが与えられたら、PCIDを選ぶ。カーネルはper-CPUの
 * 配列を使って、現在どのPCIDが使用中かを追跡する。
 */
static u16 user_pcid(u16 asid)
{
    u16 ret = asid;
    /* カーネルのPCIDはasid + TLB_NR_DYN_ASIDS */
    return ret;
}
```

PCIDで個々のPCIDタグ付きTLBエントリを無効化するには、`INVPCID`命令が必要である。初期のHaswell CPUの一部はPCIDをサポートするがINVPCIDはサポートしない — カーネルはこれを検出してフォールバックする。

---

## KPTI: Kernel Page Table Isolation {#kpti-kernel-page-table-isolation}

**KPTI**（Kernel Page Table Isolation）は、**Meltdown**（CVE-2017-5754）を緩和するために**Linux 4.15**（2018年1月）で導入された。

### KPTIが解決する問題 {#the-problem-kpti-solves}

KPTI以前は、カーネルのページテーブルは常にマップされていた — ユーザーコードを実行している間も含めて。つまり、ユーザープロセスのページテーブルには常にカーネルの仮想アドレスが含まれていた（U/S=0によってアクセス不可とマークされている）。Meltdownは投機的実行を悪用し、CPUの権限チェックが#PF（ページフォールト、ベクタ14）を発生させる前に、これらのマッピングを通じてカーネルメモリを読み取った。

### 2つのPGDによる解決策 {#the-two-pgd-solution}

KPTIは**プロセスごとに2つのページテーブル**を保持する。

```
User PGD (loaded when running in userspace):
  - All user VMA mappings
  - Minimal kernel mappings: only what is needed to enter the kernel
    • The syscall/interrupt entry trampoline page (entry_SYSCALL_64)
    • The CPU's own per-CPU data needed for the initial stack switch
  - No kernel code, no kernel data, no physmap

Kernel PGD (loaded when running in the kernel):
  - All user VMA mappings
  - Full kernel mappings: all kernel code, data, physmap, modules, vmalloc
```

2つのPGDはペアとしてまとめて割り当てられる。`pgd_alloc()`の中で、カーネルは隣接する2つのPGDページを割り当てて、それらを結び付ける。

```c
/* arch/x86/mm/pgtable.c */
pgd_t *pgd_alloc(struct mm_struct *mm)
{
    pgd_t *pgd;
    /* 2ページを割り当てる: 1つはユーザーPGD用、1つはカーネルPGD用 */
    pgd = _pgd_alloc();
    /* ... */
    pgd_prepopulate_pmd(mm, pgd, pmds);
    /* 「カーネル」PGDはpgd + PTRS_PER_PGD（1ページ後ろ）にある */
    return pgd;
}
```

### カーネルへの出入りのたびにCR3を切り替える {#cr3-switches-on-every-kernel-entryexit}

すべてのシステムコール、割り込み、例外で:

```
User → Kernel:
  1. CPU delivers exception/syscall
  2. Entry trampoline (mapped in user PGD) executes swapgs
  3. Load kernel CR3 (switch to full kernel page table)
  4. Switch to kernel stack
  5. Continue in kernel

Kernel → User:
  1. Restore user registers
  2. Load user CR3 (switch to user-only page table)
  3. swapgs
  4. SYSRET or IRET
```

エントリトランポリンは`arch/x86/entry/entry_64.S`にある。KPTIは`PAGE_TABLE_ISOLATION`設定オプションで制御される。

### KPTI + PCID: オーバーヘッドの削減 {#kpti-pcid-reducing-the-overhead}

素朴なKPTIの実装では、ユーザー↔カーネルの遷移のたびに（システムコールごとに2回のCR3ロードで）TLB全体をフラッシュすることになる。PCIDを使えば、カーネルは各プロセスのユーザーPGDとカーネルPGDに別々のPCIDを割り当てる。

```
Process A, user PGD:   PCID = asid (e.g., 5)
Process A, kernel PGD: PCID = asid | 0x80 (e.g., 5 | 0x80 = 133)
```

PCIDを使うと、ユーザーPGDとカーネルPGDの間の切り替えで、両方のTLBエントリが保持される。CR3は依然として再ロードされるが、`bit 63 = 1`（フラッシュなし）で行われる。これにより、システムコールの多いワークロードでのKPTIのオーバーヘッドが約30%からおよそ5%に減少する。

```bash
# KPTIが有効か確認する
cat /sys/devices/system/cpu/vulnerabilities/meltdown
# "Mitigation: PTI" = KPTIが有効
# "Not affected"   = CPUが脆弱でない（またはハードウェアで修正されている）

# perfでPTIのCR3切り替えコストを見る
perf stat -e tlb:tlb_flush ls
```

---

## カーネルの仮想アドレスレイアウト {#kernel-virtual-address-layout}

> 詳細: [カーネルのアドレス空間](../../mm/kernel-address-space.md)では、これらの各領域 — ダイレクトマップ、vmemmap、vmalloc、KASLR — とレイアウトの根拠を扱っている。

64ビットカーネルの仮想アドレス空間は、いくつかの領域に分割されている（値は、`CONFIG_X86_64`を使う典型的な4レベルページングの、5レベルでないシステムのもの）。

```
Virtual address layout (x86-64, 4-level paging, kernel 6.x):

ffff880000000000 - ffff887fffffffff  LDT remap area (for KPTI)
ffff888000000000 - ffffc87fffffffff  Direct mapping of all physical memory (physmap)
                                     Accessed via __va() / __pa()
ffffc90000000000 - ffffe8ffffffffff  vmalloc / ioremap area
ffffe90000000000 - ffffe9ffffffffff  Hole
ffffea0000000000 - ffffeaffffffffff  Virtual memory map (struct page array)
ffffec0000000000 - fffffbffffffffff  KASAN shadow memory (if CONFIG_KASAN)
fffffe0000000000 - fffffe7fffffffff  cpu_entry_area (per-CPU entry trampoline, fixmap)
fffffe8000000000 - fffffeffffffffff  Hole (LDT remap lived here before 4.20)
ffffff0000000000 - ffffff7fffffffff  %esp fixup stacks
ffffffef00000000 - fffffffeffffffff  EFI runtime services mapping
ffffffff80000000 - ffffffff9fffffff  Kernel text (.text, .rodata)
ffffffffa0000000 - fffffffffeffffff  Modules
ffffffffff000000 - ffffffffffffffff  Fixmap
```

physmap（ダイレクトマッピング）により、カーネルは単純な算術オフセットで任意の物理ページにアクセスできる: `phys_addr + PAGE_OFFSET = virt_addr`。標準的なx86-64カーネルでは、`PAGE_OFFSET`は`0xffff888000000000`である。

---

## ページテーブルを観察する {#observing-page-tables}

```bash
# プロセスの仮想メモリマップ
cat /proc/$$/maps
cat /proc/$$/smaps       # RSS、匿名/ファイルバックの内訳を含む

# ページテーブルの使用統計
cat /proc/$$/status | grep VmPTE   # ページテーブルが使っているメモリ

# カーネルの仮想アドレスレイアウト
dmesg | grep -E "Virtual kernel memory layout" -A 20

# ページテーブルのデバッグ（CONFIG_X86_PTDUMPまたはCONFIG_PTDUMP_DEBUGFSが必要）
ls /sys/kernel/debug/page_tables/
cat /sys/kernel/debug/page_tables/kernel

# PCID / ASIDの使用状況 — 直接のインターフェースはないが、perfで見える
perf stat -e dTLB-load-misses,iTLB-load-misses <command>

# vmcoreに対してcrash(8)を使い、仮想アドレスを変換する
# crash> vtop <address>
# crash> ptov <physical_address>

# カーネルテキストにヒュージページが使われているか確認する
dmesg | grep -i "huge\|2M\|PMD"
```

---

## 主要なカーネル関数 {#key-kernel-functions}

| 関数 | ファイル | 用途 |
|----------|------|---------|
| `pgd_alloc()` | `arch/x86/mm/pgtable.c` | PGD（およびKPTI用のシャドウPGD）を割り当てる |
| `pgd_free()` | `arch/x86/mm/pgtable.c` | PGDのペアを解放する |
| `__pa()` / `__va()` | `arch/x86/include/asm/page.h` | 物理↔仮想アドレスの変換 |
| `pte_mkwrite()` | `arch/x86/include/asm/pgtable.h` | PTEのR/Wビットをセットする |
| `set_pte_at()` | `arch/x86/mm/pgtable.c` | PTEを不可分に書き込む |
| `flush_tlb_mm()` | `arch/x86/mm/tlb.c` | mm全体のTLBフラッシュ |
| `flush_tlb_page()` | `arch/x86/mm/tlb.c` | 単一ページのTLBフラッシュ |
| `switch_mm_irqs_off()` | `arch/x86/mm/tlb.c` | コンテキストスイッチ — 新しいCR3をロードする |
| `load_new_mm_cr3()` | `arch/x86/mm/tlb.c` | PCIDとKPTIを考慮してCR3に書き込む |

---

## バージョンに関する注記 {#version-notes}

| 機能 | Linuxバージョン | 備考 |
|---------|--------------|-------|
| 4レベルページング | 当初から | x86-64でのデフォルト |
| NXビット（EFER.NXE） | 2.6.8 | 実行禁止のサポート |
| PCIDのサポート | 3.2 | CR4.PCIDE。TLB全体のフラッシュを避ける |
| 5レベルページング（LA57） | 4.14 | `CONFIG_X86_5LEVEL`、CR4.LA57 |
| KPTI（Meltdownの修正） | 4.15 | 2つのPGD、エントリ/イグジット時のCR3切り替え |
| KPTI + PCIDの最適化 | 4.15 | デュアルASID方式、`bit 63`によるフラッシュなし |

---

## 参考文献 {#further-reading}

### カーネルソースとドキュメント {#kernel-source-documentation}

- [Documentation/arch/x86/x86_64/mm.rst](https://docs.kernel.org/arch/x86/x86_64/mm.html) — 正式な仮想メモリレイアウトの表
- [arch/x86/include/asm/pgtable_types.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/pgtable_types.h) — すべての`_PAGE_*`ビットの定義
- [arch/x86/mm/tlb.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/tlb.c) — CR3の切り替え、PCID/ASIDの管理、KPTIのCR3の組み立て

### 関連ページ {#related-pages}

- [ページテーブル（汎用）](../../mm/page-tables.md) — アーキテクチャ非依存のpgd/pud/pmd/pteの視点
- [x86_64のブート時ページテーブル設定](../../mm/boot-page-tables.md) — これらの構造が最初にどのように構築されるか
- [TLBの最適化](../../mm/tlb-optimization.md) — アーキテクチャ層より上でのフラッシュのバッチ処理
- [ARM64のページテーブル](../arm64/page-tables.md) — 1つのCR3の代わりにTTBR0/TTBR1

### LWNの記事 {#lwn-articles}

- [KAISER: hiding the kernel from user space](https://lwn.net/Articles/738975/) — KPTIとなった設計。Meltdown以前に書かれたもの
- [Notes from the Intelpocalypse](https://lwn.net/Articles/742702/) — Meltdown/Spectreと、それがカーネルにとって何を意味したか
- [Kernel address space layout randomization](https://lwn.net/Articles/569635/) — KASLRの設計と限界
