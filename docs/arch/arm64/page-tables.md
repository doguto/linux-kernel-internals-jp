# ARM64のページテーブル {#arm64-page-tables}

> TTBR0/TTBR1、変換グラニュール、PTE形式、ASID

ARM64のMMUは、ユーザーとカーネルのアドレス空間を独立に扱うために2レジスタ構成を採用している。このページでは、ハードウェアによるページテーブルウォーク、Linuxがページテーブルの型をどう構成しているか、PTEのビット配置、ASIDベースのTLB管理、そしてKVM向けのStage 2変換を扱う。

---

## 2つのTTBR: ユーザーとカーネルのアドレス空間 {#two-ttbrs-user-and-kernel-address-spaces}

ARM64は**仮想アドレスの最上位ビット**を使って仮想アドレス空間を中央で分割する。2つの変換ベースレジスタが、MMUにウォークの開始位置を伝える。

| レジスタ | アドレス範囲 | 用途 |
|---|---|---|
| `TTBR0_EL1` | `[0, 2^VA_BITS)` — 低位アドレス | ユーザー空間（プロセスごと） |
| `TTBR1_EL1` | `[2^64 - 2^VA_BITS, 2^64)` — 高位アドレス | カーネル（グローバル） |

デフォルトの`VA_BITS=48`では、ユーザー空間は`[0, 0x0000_FFFF_FFFF_FFFF]`を、カーネルは`[0xFFFF_0000_0000_0000, 0xFFFF_FFFF_FFFF_FFFF]`を占める。仮想アドレスの最上位ビット（ビット63）がレジスタを選択する。ビット63 = 0ならTTBR0、ビット63 = 1ならTTBR1を使う。VA_BITS-1から63までのビットがすべて一致しない（つまり正しく符号拡張されていない）アドレスは、変換フォールト（Translation Fault）を引き起こす。

**52ビットVA（オプション）:** ARMv8.7以降は`CONFIG_ARM64_VA_BITS_52`により52ビットの仮想アドレス（`VA_BITS=52`）をサポートする。この場合も変換レベルは4段（5段ではない）であり、追加のビットはLevel 0のインデックスフィールドを広げることで賄われる。これにより、ユーザーとカーネルの範囲はそれぞれ4PBに拡張される。ほとんどの本番構成はいまだに48ビットを使っている。

コンテキストスイッチ時、カーネルは新しいプロセスのPGDの物理アドレスを`TTBR0_EL1`に書き込む。同時にASID（後述）が`TTBR0_EL1`のビット`[63:48]`に詰め込まれる。`TTBR1_EL1`はブート時に一度設定され、以後変わらない。

```c
/* arch/arm64/include/asm/mmu_context.h */
static inline void cpu_switch_mm(pgd_t *pgd, struct mm_struct *mm)
{
    BUG_ON(pgd == swapper_pg_dir);
    cpu_set_reserved_ttbr0();
    /* ASIDはmm->context.idに詰め込まれている。cpu_do_switch_mmは
     * 新しいPGDの物理アドレスとASIDを一緒にTTBR0_EL1に書き込む — 異なるASIDへ
     * 切り替える際にTLBフラッシュは不要である（それこそがASIDの存在意義である）。 */
    cpu_do_switch_mm(virt_to_phys(pgd), mm);
}
```

---

## 変換グラニュール {#translation-granules}

**変換グラニュール**とはベースとなるページサイズである。ARM64は3種類のグラニュールサイズをサポートし、それぞれで必要なレベル数と各レベルのテーブルサイズが変わる。

| グラニュール | ページサイズ | レベル数（48ビットVA） | 備考 |
|---|---|---|---|
| 4KB | 4KB | 4（L0→L1→L2→L3） | 最も一般的。Linuxのデフォルト |
| 16KB | 16KB | 4（L0→L1→L2→L3） | Apple Siliconが使用 |
| 64KB | 64KB | 3（L1→L2→L3） | レベル数が少ない。大きな連続TLBエントリ |

グラニュールは`TCR_EL1.TG0`（TTBR0用）と`TCR_EL1.TG1`（TTBR1用）で選択される。

### 4KBグラニュール: 各レベルのカバー範囲（48ビットVA） {#4kb-granule-level-coverage-48-bit-va}

4KBグラニュールでは、各ページテーブルはちょうど4KBの1ページであり、8バイトのエントリを512個含む（レベルごとに9ビットのインデックス）。

```
Level   Index bits   Entries   Coverage per entry
─────   ──────────   ───────   ──────────────────
L0      [47:39]        512      512 GB
L1      [38:30]        512        1 GB
L2      [29:21]        512        2 MB  (block descriptor = huge page)
L3      [20:12]        512        4 KB  (page descriptor = normal page)
                                ────
Page offset [11:0]     —        4 KB
```

---

## 4段のページテーブルウォーク（48ビットVA、4KBグラニュール） {#the-4-level-page-table-walk-48-bit-va-4kb-granule}

4KBグラニュールの48ビット仮想アドレスに対して、ハードウェアは4段のウォークを行う。

```
48-bit Virtual Address:
 47      39 38      30 29      21 20      12 11       0
 ┌─────────┬──────────┬──────────┬──────────┬──────────┐
 │ L0 index│ L1 index │ L2 index │ L3 index │  offset  │
 │ (9 bits)│ (9 bits) │ (9 bits) │ (9 bits) │ (12 bits)│
 └─────────┴──────────┴──────────┴──────────┴──────────┘

Walk (TTBR0 for user, TTBR1 for kernel):

TTBR0_EL1 or TTBR1_EL1 → physical address of L0 table
  L0_table[VA[47:39]]   → physical address of L1 table
    L1_table[VA[38:30]] → physical address of L2 table  (or 1GB block)
      L2_table[VA[29:21]] → physical address of L3 table (or 2MB block)
        L3_table[VA[20:12]] → physical address of 4KB page frame
          + VA[11:0]         = final physical address
```

MMUは変換結果をVA + ASIDをキーとしてTLBにキャッシュする。TLBミス時には、ハードウェアのページテーブルウォーカが自動的にウォークを行う（ARM64のTLBはハードウェア管理であり、通常のケースではソフトウェアによるTLBの補充は不要である）。

---

## PTE形式 {#pte-format}

ARM64のディスクリプタは64ビットの値である。ビットの意味は、レベルと、エントリが**テーブル**、**ブロック**、**ページ**のどのディスクリプタであるかによって決まる。

```
ARM64 Page/Block Descriptor (64-bit):

 63   59 58   55 54 53 52 51      12 11 10  9  8  7  6  5  4  2  1  0
 ┌──────┬───────┬──┬──┬──┬──────────┬──┬──┬──┬──┬──┬──┬─────┬──┬──┐
 │ PBHA │ SW   │UXN│PXN│Cont│  OA   │nG│AF│SH│AP│NS│  │AtIdx│type│V│
 └──────┴───────┴──┴──┴──┴──────────┴──┴──┴──┴──┴──┴──┴─────┴──┴──┘

 bit 0:     Valid (V)        — 1 = エントリは有効
 bit 1:     Type             — L0-L2では: 1 = テーブルディスクリプタ、0 = ブロックディスクリプタ
                               L3では: 1 = ページディスクリプタ（有効なページでは1でなければならない）
 bits [4:2]: AttrIndx[2:0]  — MAIR_EL1へのインデックス（メモリタイプの選択）
 bit 5:     NS               — Non-Secure（Secure状態で適用）
 bits [7:6]: AP[2:1]         — アクセス許可（下表参照）
 bits [9:8]: SH[1:0]         — 共有可能性: 00=non-shareable、10=outer、11=inner
 bit 10:    AF               — Access Flag: 0なら初回アクセスでフォールト（SWが管理）
 bit 11:    nG               — not-Global: 1 = ASIDタグ付き（ユーザー）、0 = グローバル（カーネル）
 bits [47:12]: Output Address (OA) — 物理ページフレーム番号（PAのビット[47:12]）
 bit 52:    Contiguous hint  — TLBが連続したエントリをまとめられる
 bit 53:    PXN              — Privileged Execute Never（EL1は実行できない）
 bit 54:    UXN              — Unprivileged Execute Never（EL0は実行できない）
 bits [58:55]: SW            — ソフトウェア定義（カーネルが_PAGE_*フラグに使う）
 bits [63:59]: PBHA          — Page-Based Hardware Attributes（ARMv8.2以降）
```

### アクセス許可（AP[2:1]）のエンコーディング {#access-permission-ap21-encoding}

| AP[2:1] | EL1（カーネル） | EL0（ユーザー） |
|---|---|---|
| `00` | 読み書き | アクセス不可 |
| `01` | 読み書き | 読み書き |
| `10` | 読み取り専用 | アクセス不可 |
| `11` | 読み取り専用 | 読み取り専用 |

Linuxはすべてのカーネルマッピングに`UXN`を、すべてのユーザーマッピングに`PXN`をセットする（ユーザーのコードがEL1で実行されること、およびその逆を防ぐ）。`AF`（Access Flag）ビットについては、ARMv8.1より前のハードウェアではソフトウェアが管理する。AF=0のPTEは初回アクセス時にAccess Flagフォールトを発生させ、カーネルはAF=1をセットしてそれを処理する。`FEAT_HAFDBS`を持つARMv8.1以降のハードウェアでは、Linuxはハードウェア管理のAF（`TCR_EL1.HA=1`）を有効にするので、CPUはフォールトなしに自動的にAFをセットする。Access Flagは、ページ回収のパスで最近アクセスされたページを見分けるために使われる。

### ARM64とx86-64のPTEの比較 {#arm64-vs-x86-64-pte-comparison}

| 概念 | ARM64 | x86-64 |
|---|---|---|
| 有効ビット | ビット0（`V`） | ビット0（`P` — Present） |
| 読み書き | `AP[2:1]`フィールド | ビット1（`R/W`） |
| ユーザーアクセス | `AP[2:1]`フィールド | ビット2（`U/S`） |
| 実行禁止 | `UXN`（ビット54）、`PXN`（ビット53） | ビット63（`NX`） |
| キャッシュタイプ | `AttrIndx` → `MAIR_EL1` | `PCD`/`PWT`/`PAT`ビット |
| huge pageの目印 | L1/L2でビット1 = 0（ブロック） | PMD/PUDの`PS`ビット |
| グローバル（ASIDフラッシュなし） | `nG` = 0 | ビット8（`G`） |
| Dirty / Accessed | AFフォールトによるソフトウェア管理 | ハードウェアがセットするビット5（`A`）と6（`D`） |

---

## huge page: ブロックディスクリプタ {#huge-pages-block-descriptors}

ARM64は、4レベルすべてをたどらずに大きな連続物理領域をマップするために**ブロックディスクリプタ**を使う。L1またはL2のブロックディスクリプタはビット1（`type`）= 0である。

| レベル | ブロックサイズ | Linuxでの用語 |
|---|---|---|
| L1 | 1 GB | `pud_huge()` — 1GBのhuge page |
| L2 | 2 MB | `pmd_huge()` — 2MBのhuge page（THP、hugetlbfs） |

Linuxは`arch/arm64/include/asm/pgtable.h`で定義された`pmd_huge()`でブロックディスクリプタを検出する。

```c
/* arch/arm64/include/asm/pgtable.h */
static inline int pmd_huge(pmd_t pmd)
{
    return pmd_val(pmd) && !(pmd_val(pmd) & PMD_TABLE_BIT);
}
```

`PMD_TABLE_BIT`はビット1である。ビット1がクリアされた有効なPMDエントリは2MBのブロックディスクリプタである。

Transparent Huge Pages（THP）と`hugetlbfs`はどちらも2MBのブロックディスクリプタを使う。カーネルはまた、アラインメントが許す場合には自身のtextセクションとdataセクションを2MBブロックでマップする（`arch/arm64/mm/mmu.c`の`create_mapping_noalloc()`で確認できる）。

---

## Linuxカーネルのページテーブルの型 {#linux-kernel-page-table-types}

Linuxはハードウェアのレベルの上に汎用的な5段の抽象化を用いる。48ビットVAで4KBグラニュールのARM64では、`p4d_t`は**折りたたまれる**（`pgd_t`に対する何もしない別名となる）。

| Linuxの型 | ハードウェアのレベル | ソース |
|---|---|---|
| `pgd_t` | L0（TTBR0/TTBR1が指す） | `arch/arm64/include/asm/pgtable-types.h` |
| `p4d_t` | PGDに折りたたまれる（48ビットVA） | `include/asm-generic/pgtable-nop4d.h` |
| `pud_t` | L1 | `arch/arm64/include/asm/pgtable-types.h` |
| `pmd_t` | L2 | `arch/arm64/include/asm/pgtable-types.h` |
| `pte_t` | L3 | `arch/arm64/include/asm/pgtable-types.h` |

すべての型は`u64`のラッパーである。基になるハードウェアのディスクリプタ値には、`pgd_val()`、`pud_val()`、`pmd_val()`、`pte_val()`でアクセスする。

標準的なページテーブルのナビゲーションマクロ:

```c
/* mm_structと仮想アドレスが与えられたとき、PTEまでたどる: */
pgd_t *pgd = pgd_offset(mm, addr);       /* mm->pgdへのインデックス */
p4d_t *p4d = p4d_offset(pgd, addr);      /* 48ビットARM64では折りたたまれる */
pud_t *pud = pud_offset(p4d, addr);
pmd_t *pmd = pmd_offset(pud, addr);
pte_t *pte = pte_offset_map(pmd, addr);  /* PTEページをマップし、カーネルVAを返す */
```

`pte_offset_map()`は`arch/arm64/include/asm/pgtable.h`で定義されており、PTEページのポインタについて物理アドレスから仮想アドレスへの変換を処理する。

---

## ASID: アドレス空間識別子 {#asid-address-space-identifiers}

PTE中のすべてのユーザー空間マッピングは`nG`（not-Global）= 1である。これは、TLBがそのエントリに現在の**ASID**（Address Space Identifier）をタグ付けすることを意味する。異なるASIDを持つ異なるプロセスは互いに干渉せずにTLB内で共存でき、コンテキストスイッチのたびにTLB全体をフラッシュすることを避けられる。

### ASIDの格納場所 {#asid-storage}

ASIDは`TTBR0_EL1[63:48]`に格納される（`TCR_EL1.AS=1`のとき16ビットのASIDフィールド）。プロセスがスケジュールされるたびに、カーネルはASIDとPGDアドレスの両方を`TTBR0_EL1`にアトミックに書き込む。カーネルマッピングは`nG=0`（グローバル）を使うので、ASIDでタグ付けされず、ASIDベースのTLB操作で追い出されることはない。

プロセスごとのASIDは`mm->context.id`に格納され、`arch/arm64/mm/context.c`の`check_and_switch_context()`によって管理される。

```c
/* arch/arm64/mm/context.c */
void check_and_switch_context(struct mm_struct *mm)
{
    unsigned long flags;
    unsigned int cpu;
    u64 asid, old_active_asid;

    asid = atomic64_read(&mm->context.id);

    /*
     * 現在のASIDがこのCPUでまだ有効なら、ロックを取らずにそれを使う。
     * そうでなければ低速パスへ進む。
     */
    old_active_asid = atomic64_read(this_cpu_ptr(&active_asids));
    if (asid_gen_match(asid) &&
        atomic64_cmpxchg_relaxed(this_cpu_ptr(&active_asids),
                                 old_active_asid, asid))
        goto switch_mm_fastpath;

    raw_spin_lock_irqsave(&cpu_asid_lock, flags);
    /* ... 低速パス: 新しいASIDを割り当てる、または世代のラップ時にフラッシュする ... */
    raw_spin_unlock_irqrestore(&cpu_asid_lock, flags);

switch_mm_fastpath:
    cpu_switch_mm(mm->pgd, mm);
}
```

### ASID世代のラップ {#asid-generation-wrap}

ARM64は8ビットまたは16ビットのASIDをサポートする（`TCR_EL1.AS`）。Linuxは実行時に`ID_AA64MMFR0_EL1.ASIDBits`からASIDの幅を検出し、利用可能なら16ビットASIDを使う。16ビットASIDでは65536個の値が使える。それらを使い果たすと、カーネルは内部の**世代カウンタ**をインクリメントしてTLB全体をフラッシュし（`TLBI VMALLE1IS`）、ASIDを最初から割り当て直す。

---

## MAIR_EL1: Memory Attribute Indirection Register {#mair_el1-memory-attribute-indirection-register}

各PTEの`AttrIndx[2:0]`フィールドは`MAIR_EL1`へのインデックスであり、`MAIR_EL1`は8つの属性スロットを実際のメモリタイプに対応付ける。この間接参照により、PTE形式をコンパクトに保ちつつ多くのメモリタイプをサポートできる。

Linuxはブート時に一度、`arch/arm64/mm/proc.S`で`MAIR_EL1`を設定する。

```asm
/* arch/arm64/mm/proc.S — __cpu_setup中のMAIR_EL1設定 */
/*
 * MAIR_EL1の属性エンコーディング（スロットごとに1バイト、8スロット = 64ビット）:
 *
 * Slot  AttrIndx  Attr byte  Memory type
 * ────  ────────  ─────────  ───────────────────────────────────────
 *   0     0b000   0x00       Device-nGnRnE  （強く順序付けられたデバイス）
 *   1     0b001   0x04       Device-nGnRE   （デバイス、gathering+reorderingは可）
 *   2     0b010   0x0C       Device-GRE     （デバイス、gathering+reordering+early-writeは可）
 *   3     0b011   0x44       Normal Non-Cacheable (NC)
 *   4     0b100   0xFF       Normal Write-Back, Read-Allocate, Write-Allocate (WB)
 *   5     0b101   0xBB       Normal Write-Through, Read-Allocate (WT)
 *   6     0b110   0x40       Normal Non-Cacheable Outer, NC Inner
 *   7     0b111   0xFF       Normal WB （重複。Tagged Normalメモリで使用）
 *
 * Linuxの主要スロット:
 *   MT_DEVICE_nGnRnE  = 0  （強く順序付けられたioremap）
 *   MT_DEVICE_nGnRE   = 1  （デバイスのioremap）
 *   MT_DEVICE_GRE     = 2  （ライトコンバイニングのioremap）
 *   MT_NORMAL_NC      = 3  （キャッシュ不可のDMA）
 *   MT_NORMAL         = 4  （通常のRAM — カーネル/ユーザーのすべてのメモリ）
 *   MT_NORMAL_WT      = 5  （ライトスルー）
 */
ldr x5, =MAIR_EL1_SET
msr mair_el1, x5
isb
```

定数`MAIR_EL1_SET`は、`arch/arm64/include/asm/pgtable-hwdef.h`の個々の`MAIR_ATTRIDX()`マクロから組み立てられる。ユーザー空間とカーネルのtext/dataはスロット4（`MT_NORMAL`、`AttrIndx=0b100`）を使う。`ioremap()`でマップされたデバイスメモリはスロット0（`MT_DEVICE_nGnRnE`）を使う。

---

## TLB操作 {#tlb-operations}

ARM64はIPIではなく**ブロードキャスト型のTLB無効化命令**（`TLBI`）を使う。これらの命令は、`IS`（Inner Shareable）サフィックスを使うと、共有可能ドメイン内のCPU間に自動的に伝播する。

### 主なTLBI命令 {#key-tlbi-instructions}

| 命令 | 無効化する対象 | 使う場面 |
|---|---|---|
| `TLBI VAE1IS, Xt` | VA + 現在のASIDによるエントリ、inner shareable | 単一ページのアンマップ（ユーザー） |
| `TLBI VALE1IS, Xt` | VA + ASIDによるエントリ、最終レベルのみ | リーフページではこちらがより一般的 |
| `TLBI ASIDE1IS, Xt` | ASIDに一致する全エントリ、inner shareable | プロセス終了（mmの破棄） |
| `TLBI VMALLE1IS` | EL1の全エントリ（全ASID）、inner shareable | ASID世代のラップ |
| `TLBI VAAE1IS, Xt` | VAによるエントリ、全ASID | カーネルマッピングの変更 |

オペランド`Xt`は、12ビット右シフトした（ページアラインされた）VAと、ビット`[63:48]`のASIDのORをエンコードする。

### 必須のTLB無効化シーケンス {#required-tlb-invalidation-sequence}

新しいPTEを書き込んでからTLBを無効化する際は、MMUが古い変換をキャッシュしないよう、厳密な順序に従わなければならない。

```
1. Write new PTE to memory
2. DSB ISHST       — ensure the PTE store is visible to all CPUs before TLBI
3. TLBI VAE1IS     — broadcast TLB invalidation
4. DSB ISH         — wait for TLB invalidation to complete on all CPUs
5. ISB             — flush instruction pipeline (see new mappings for instruction fetch)
```

LinuxのCコードでは、`arch/arm64/include/asm/tlbflush.h`の`flush_tlb_page()`がこれを実装している。

```c
/* arch/arm64/include/asm/tlbflush.h */
static inline void flush_tlb_page(struct vm_area_struct *vma,
                                   unsigned long uaddr)
{
    unsigned long addr = __TLBI_VADDR(uaddr, ASID(vma->vm_mm));

    dsb(ishst);              /* DSB ISH ST: PTEのストアの完了を待つ */
    __tlbi(vale1is, addr);   /* TLBI VALE1IS: 最終レベルのTLBエントリを無効化 */
    dsb(ish);                /* DSB ISH: TLB無効化の伝播を待つ */
}
```

`flush_tlb_page()`自体には`ISB`がないことに注意してほしい。これはデータマッピングにのみ使われるからであり、ISBは命令マッピングが変わる場合（例: `flush_icache_range()`）にのみ必要である。

`__TLBI_VADDR()`は、ページアラインされたアドレスとASIDを`TLBI`オペランドが期待する形式にエンコードするマクロである。`__tlbi()`は、`SYS`命令エンコーディングを介したインラインアセンブリの`TLBI`命令に展開される。

---

## Stage 2変換（KVM） {#stage-2-translation-kvm}

KVMが有効なとき、CPUはEL2で動作し、ゲストはEL1/EL0で動作する。ゲストの仮想アドレスはまず**Stage 1**のページテーブル（ゲストOS自身の`TTBR0_EL1`/`TTBR1_EL1`）によって**中間物理アドレス（IPA: Intermediate Physical Address）**に変換され、次に**Stage 2**のページテーブルによって実際の**物理アドレス（PA）**に変換される。

Stage 2は`VTTBR_EL2`によって制御される。これはStage 2のPGD（**IPA→PA**の変換テーブル）の物理アドレスを保持する。Stage 2のテーブルはStage 1と似たディスクリプタ形式を使うが、属性フィールドが異なる（`S2AP`、`S2SH`、`MemAttr`）。

```
Guest VA ──[Stage 1: TTBR0/TTBR1]──► IPA ──[Stage 2: VTTBR_EL2]──► PA
                 (Guest OS controls)              (KVM controls)
```

ゲストに対するTLB無効化には、現在のVMのVMIDに対応するエントリだけを無効化するために、`VMID`を伴う`TLBI`命令（例: `TLBI IPAS2E1IS`）が必要である。VMIDは`VTTBR_EL2[55:48]`に格納される。

LinuxにおけるStage 2の管理は`arch/arm64/kvm/mmu.c`にある。詳細は[KVMアーキテクチャ](../../virtualization/kvm-arch.md)と[メモリ仮想化](../../virtualization/kvm-memory.md)のドキュメントを参照のこと。

---

## ARM64のページテーブルを観察する {#observing-arm64-page-tables}

```bash
# ARM64の仮想アドレスレイアウト
dmesg | grep -E "Virtual kernel memory layout" -A 20

# プロセスごとのページテーブルのメモリ
cat /proc/$$/status | grep VmPTE

# 現在のプロセスのメモリマッピング
cat /proc/$$/maps
cat /proc/$$/smaps   # ページレベルの内訳を含む

# VA_BITSを確認（カーネルconfig）
zcat /proc/config.gz | grep ARM64_VA_BITS

# 変換グラニュールを確認
zcat /proc/config.gz | grep ARM64_4K_PAGES

# TLBミス率（PMUへのアクセスが必要）
perf stat -e dTLB-load-misses,iTLB-load-misses <command>

# ASIDの割り当て（DEBUG_VMビルドではdmesg経由でASID情報が見える）
dmesg | grep -i asid

# ページテーブルのダンプ（CONFIG_PTDUMP_DEBUGFSが必要）
ls /sys/kernel/debug/page_tables/
cat /sys/kernel/debug/page_tables/kernel
```

---

## 主要なカーネル関数とファイル {#key-kernel-functions-and-files}

| シンボル | ファイル | 目的 |
|---|---|---|
| `pgd_t`、`pud_t`、`pmd_t`、`pte_t` | `arch/arm64/include/asm/pgtable-types.h` | ページテーブルエントリの型 |
| `pgd_offset()`、`pud_offset()`、`pmd_offset()` | `arch/arm64/include/asm/pgtable.h` | VA → ページテーブルの各レベルへのポインタ |
| `pte_offset_map()` | `arch/arm64/include/asm/pgtable.h` | PTEページをマップしてポインタを返す |
| `pmd_huge()` | `arch/arm64/include/asm/pgtable.h` | 2MBブロックディスクリプタを検出 |
| `pud_huge()` | `arch/arm64/include/asm/pgtable.h` | 1GBブロックディスクリプタを検出 |
| `set_pte_at()` | `arch/arm64/mm/pgtable.c` | PTEを書き込む（バリア付き） |
| `flush_tlb_page()` | `arch/arm64/include/asm/tlbflush.h` | 単一ページのTLB無効化 |
| `flush_tlb_mm()` | `arch/arm64/include/asm/tlbflush.h` | mm全体のTLBフラッシュ（ASIDE1IS） |
| `check_and_switch_context()` | `arch/arm64/mm/context.c` | ASIDの割り当てとmmの切り替え |
| `cpu_do_switch_mm()` | `arch/arm64/mm/proc.S` | 新しいASID+PGDでTTBR0_EL1を書き込む |
| `__cpu_setup()` | `arch/arm64/mm/proc.S` | TCR_EL1、MAIR_EL1、SCTLR_EL1を設定 |
| `create_mapping_noalloc()` | `arch/arm64/mm/mmu.c` | カーネルのページテーブルエントリを構築 |
| `MAIR_EL1_SET` | `arch/arm64/include/asm/pgtable-hwdef.h` | スロットマクロから組み立てたMAIR値 |

---

## 参考文献 {#further-reading}

- [メモリモデル](memory-model.md) — ARM64の弱いメモリ順序付け、DSB/ISB、ページテーブルのバリア
- [例外モデル](exception-model.md) — EL0–EL3、EL2とStage 2変換の関係
- [ページテーブル（汎用）](../../mm/page-tables.md) — Linuxのpgd/pud/pmd/pte抽象化層
- [ページフォールトハンドラ](../../mm/page-fault.md) — カーネルが変換フォールトとAccess Flagフォールトをどう処理するか
- [Transparent Huge Pages](../../mm/thp.md) — Linuxが4KBページを2MBブロックディスクリプタに昇格させる方法
- [KVMのメモリ仮想化](../../virtualization/kvm-memory.md) — Stage 2ページテーブル、ARM64におけるEPT/NPT相当
- [TLB最適化](../../mm/tlb-optimization.md) — バッチ化されたTLB無効化とmmu_gather
- ARM Architecture Reference Manual（DDI 0487） — D5: AArch64 Virtual Memory System Architecture
- [Documentation/arch/arm64/memory.rst](https://docs.kernel.org/arch/arm64/memory.html) — ARM64の仮想アドレスレイアウト
