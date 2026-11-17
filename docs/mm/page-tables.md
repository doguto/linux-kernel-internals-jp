# ページテーブル {#page-tables}

> 仮想アドレスを物理メモリに対応付ける

## ページテーブルとは何か? {#what-are-page-tables}

ページテーブルは、仮想アドレス（プログラムから見えるもの）を物理アドレス（実際のRAM上の位置）に対応付ける階層的なデータ構造である。MMU（Memory Management Unit）ハードウェアは、これを使ってすべてのメモリアクセスを変換する。

```
Virtual Address                Physical Address
0x7fff12340000   ──────────>   0x1a2b3c000
       │                              │
       │    Page Table Walk           │
       └──────────────────────────────┘
```

## 5段階の階層 {#the-five-level-hierarchy}

Linuxは5段階のページテーブル階層を使う（x86-64では[v4.14](https://git.kernel.org/linus/77ef56e4f0fb)以降。[x86_64 memory map](https://docs.kernel.org/arch/x86/x86_64/mm.html)を参照）。

```
+-----+
| PGD |  Page Global Directory
+-----+
   │
   v
+-----+
| P4D |  Page Level 4 Directory (5-level paging only)
+-----+
   │
   v
+-----+
| PUD |  Page Upper Directory
+-----+
   │
   v
+-----+
| PMD |  Page Middle Directory
+-----+
   │
   v
+-----+
| PTE |  Page Table Entry
+-----+
   │
   v
 PAGE    Physical memory page
```

### 各レベルの詳細 {#level-details}

| レベル | 型 | ビット（x86-64） | エントリ数 | マップする範囲 |
|-------|------|---------------|---------|------|
| PGD | `pgd_t` | 47-39（4段階）または56-48（5段階） | 512 | 512GBまたは128PB |
| P4D | `p4d_t` | 47-39 | 512 | 512GB（4段階では畳み込まれる） |
| PUD | `pud_t` | 38-30 | 512 | 1GB |
| PMD | `pmd_t` | 29-21 | 512 | 2MB |
| PTE | `pte_t` | 20-12 | 512 | 4KB |

### アドレス変換（x86-64、4段階） {#address-translation-x86-64-4-level}

48ビットの仮想アドレスは次のように分解される。

```
 47      39 38      30 29      21 20      12 11       0
+----------+----------+----------+----------+----------+
| PGD idx  | PUD idx  | PMD idx  | PTE idx  |  Offset  |
+----------+----------+----------+----------+----------+
   9 bits     9 bits     9 bits     9 bits    12 bits
```

## ページテーブルの畳み込み {#page-table-folding}

すべてのアーキテクチャが5段階すべてを必要とするわけではない。Linuxはこれを**畳み込み**（folding）で扱う。使われないレベルはコンパイル時に最適化で取り除かれる。

| アーキテクチャ | レベル数 | 備考 |
|--------------|--------|-------|
| x86-64（LA57） | 5 | 57ビットの仮想アドレス |
| x86-64（標準） | 4 | 48ビットの仮想アドレス |
| x86-32（PAE） | 3 | 36ビットの物理アドレス |
| x86-32 | 2 | 元々の32ビット |
| ARM64 | 3-4 | 設定可能 |

あるレベルが畳み込まれると、`p4d_offset()`のような関数は入力をそのまま返す何もしない処理になる。

## 主要なデータ構造 {#key-data-structures}

### プロセスごとのページテーブル {#per-process-page-tables}

各プロセスは`mm_struct`を介して自身のページテーブルを持つ。

```c
struct mm_struct {
    pgd_t *pgd;              /* 最上位のページテーブル */
    atomic_t mm_users;       /* このmmの利用者 */
    atomic_t mm_count;       /* 構造体への参照 */
    /* ... */
};
```

カーネルはカーネルのマッピング用に単一の`swapper_pg_dir`を持ち、それはすべてのプロセスで共有される。

### ページテーブルエントリの形式（x86-64固有） {#page-table-entry-format-x86-64-specific}

```
 63    62    52 51                           12 11  9 8 7 6 5 4 3 2 1 0
+----+--------+-------------------------------+-----+-+-+-+-+-+-+-+-+-+
| XD | (avail)|        Physical Address       |avail|G|S|D|A|C|W|U|W|P|
+----+--------+-------------------------------+-----+-+-+-+-+-+-+-+-+-+
  │                        │                          │ │ │ │ │ │ │ │ │
  │                        │                          │ │ │ │ │ │ │ │ └─ Present
  │                        │                          │ │ │ │ │ │ │ └─── Writable
  │                        │                          │ │ │ │ │ │ └───── User accessible
  │                        │                          │ │ │ │ │ └─────── Write-through
  │                        │                          │ │ │ │ └───────── Cache disable
  │                        │                          │ │ │ └─────────── Accessed
  │                        │                          │ │ └───────────── Dirty
  │                        │                          │ └─────────────── Page size (huge)
  │                        │                          └───────────────── Global
  │                        └──────────────────────────────────────────── PFN
  └───────────────────────────────────────────────────────────────────── Execute disable
```

## ページフォールト {#page-faults}

MMUがアドレスを変換できないと、**ページフォールト**が発生する。カーネルはこれを`handle_mm_fault()`で処理する。

```
Page Fault
    │
    v
handle_mm_fault()
    │
    v
__handle_mm_fault()
    │
    ├── pgd_offset() ─> p4d_alloc()
    ├── p4d_offset() ─> pud_alloc()
    ├── pud_offset() ─> pmd_alloc()
    ├── pmd_offset() ─> pte_alloc()
    │
    v
handle_pte_fault()
    │
    ├── do_read_fault()    (file read)
    ├── do_cow_fault()     (copy-on-write)
    └── do_shared_fault()  (shared mapping)
```

### フォールトの種類 {#fault-types}

| 種類 | 原因 | 解決方法 |
|------|-------|------------|
| マイナー | pageはメモリ上にあるが、PTEが設定されていない | PTEを更新する |
| メジャー | pageがメモリ上にない | ディスク/スワップから読み込む |
| 不正 | 不正なアドレスまたは権限 | SIGSEGV |

## TLB（Translation Lookaside Buffer） {#tlb-translation-lookaside-buffer}

TLBは最近の変換結果をキャッシュする。これがなければ、すべてのメモリアクセスで複数回のページテーブル参照が必要になる。

### TLBフラッシュ操作 {#tlb-flush-operations}

```c
/* 単一のpageをフラッシュする */
flush_tlb_page(vma, addr);

/* 範囲をフラッシュする */
flush_tlb_range(vma, start, end);

/* mm全体をフラッシュする */
flush_tlb_mm(mm);
```

SMPではTLBフラッシュは高くつく。影響を受けるプロセスを実行しているすべてのCPUへのIPI（Inter-Processor Interrupt、プロセッサ間割り込み）が必要になるからである。

## ヒュージページ {#huge-pages}

上位レベルのエントリは、下位レベルを飛ばして大きなpageを直接マップできる。

| レベル | pageサイズ | 用途 |
|-------|-----------|----------|
| PUD | 1GB | 大規模データベース、VM |
| PMD | 2MB | 一般的な大きな割り当て |
| PTE | 4KB | デフォルト |

```c
/* PMDがヒュージページをマップしているか確認する */
if (pmd_large(*pmd)) {
    /* 2MBのpage。PTEのウォークは不要 */
}
```

利点:

- 必要なTLBエントリが少ない
- ページテーブルのメモリが減る
- ページフォールトが少ない

トレードオフ:

- 内部断片化
- 割り当ての難しさ

## 歴史 {#history}

### 起源（1991〜1994年） {#origins-1991-1994}

i386上の初期のLinuxは2段階のページテーブル（PGD + PTE）を使っていた。`swapper_pg_dir`は最上位のPage Global Directoryであり、単一のフラットなテーブルではなかった。この構造はi386のハードウェアによって決められていた。

### 3段階（v2.3.23、1999年） {#three-level-v2323-1999}

x86のPAE（Physical Address Extension）のためにPMDが追加され、32ビットで4GBを超える物理メモリが使えるようになった。

### 4段階（v2.6.11、2005年） {#four-level-v2611-2005}

x86-64の48ビット仮想アドレス空間のためにPUDが追加された。これはgitの履歴より前のことである（カーネルはv2.6.12でgitに移行した）。

### 5段階（v4.14、2017年） {#five-level-v414-2017}

**コミット**: [77ef56e4f0fb](https://git.kernel.org/linus/77ef56e4f0fb)（"x86: Enable 5-level paging support via CONFIG_X86_5LEVEL=y"） | [LKML](https://lore.kernel.org/all/20170313055020.69655-1-kirill.shutemov@linux.intel.com/)

57ビットの仮想アドレス（128PB）のためにP4Dが追加された。これは64TBを超える物理メモリを持つマシンに必要である。

## 自分で試す {#try-it-yourself}

### プロセスのページテーブルを見る {#view-process-page-tables}

```bash
# プロセスのページテーブル統計
cat /proc/<pid>/smaps | grep -E "Size|Rss|Pss"

# 詳細なpageマッピング
cat /proc/<pid>/pagemap  # バイナリ形式

# ツールで解析する
pagemap <pid> <address>
```

### TLBフラッシュを確認する {#check-tlb-flushes}

```bash
# TLBフラッシュの統計
cat /proc/vmstat | grep tlb

# TLBフラッシュをトレースする
echo 1 > /sys/kernel/debug/tracing/events/tlb/tlb_flush/enable
cat /sys/kernel/debug/tracing/trace_pipe
```

### ヒュージページの使用状況を確認する {#check-huge-page-usage}

```bash
# システム全体のヒュージページ統計
cat /proc/meminfo | grep -i huge

# プロセスごとのヒュージページ
cat /proc/<pid>/smaps | grep -i huge
```

## よくある問題 {#common-issues}

### TLBシュートダウンの嵐 {#tlb-shootdown-storms}

`mprotect()`や`munmap()`を多用すると、過剰なIPIが発生する。

**デバッグ**: `/proc/interrupts`でTLBのIPI数を確認する。

### ページテーブルのメモリオーバーヘッド {#page-table-memory-overhead}

疎なアドレス空間はページテーブルのメモリを浪費する。

**デバッグ**: `/proc/meminfo`の`PageTables`を確認する。

### ヒュージページの割り当て失敗 {#huge-page-allocation-failures}

断片化のためにヒュージページを割り当てられない。

**解決策**:

- ブート時に予約する: `hugepages=N`
- THPを有効にする: `/sys/kernel/mm/transparent_hugepage/enabled`
- メモリをコンパクションする: `echo 1 > /proc/sys/vm/compact_memory`

## 悪名高いバグとエッジケース {#notorious-bugs-and-edge-cases}

ページテーブルはすべてのメモリアクセスを仲介する。ここでのバグは、権限昇格から情報漏洩、さらにはCPUのセキュリティモデルの完全な崩壊にまで及ぶ。

### ケース1: Meltdown（CVE-2017-5754） {#case-1-meltdown-cve-2017-5754}

#### 何が起きたか {#what-happened}

2018年1月、研究者らが[Meltdown](https://meltdownattack.com/)を公表した。これはIntel CPUに影響するハードウェアの脆弱性で、特権のないプロセスがカーネルメモリを読むことを可能にした。

#### バグ {#the-bug}

Meltdownは**投機的実行**と**キャッシュタイミング**を組み合わせて悪用する。CPUは不正なカーネルの読み取りを投機的に実行し、フォールトが発生してもキャッシュへの副作用は残るため、データが漏洩する。

#### 修正: KPTI {#the-fix-kpti}

**コミット**: [aa8c6248f8c7](https://git.kernel.org/linus/aa8c6248f8c7)（"x86/mm/pti: Add infrastructure for page table isolation"）

**作者**: Thomas Gleixner

KPTIはカーネルモードとユーザーモードで別々のページテーブルを作成する。カーネルに入るとき、ページテーブルはカーネルのマッピングを含むものに切り替えられる。

```bash
# KPTIの状態を確認する
cat /sys/devices/system/cpu/vulnerabilities/meltdown
```

---

### ケース2: Spectre（CVE-2017-5753、CVE-2017-5715） {#case-2-spectre-cve-2017-5753-cve-2017-5715}

#### 何が起きたか {#what-happened_1}

Meltdownと同時に公表された[Spectre](https://spectreattack.com/)は、投機的実行を悪用して他のプロセスやカーネルのデータを漏洩させる。

#### バリアント1（Bounds Check Bypass） {#variant-1-bounds-check-bypass}

CPUは境界チェックを越えて投機的に実行し、キャッシュタイミングを通じて範囲外の読み取りを可能にしてしまう。

**修正**: `array_index_nospec()`による配列インデックスのマスキング。

#### バリアント2（Branch Target Injection） {#variant-2-branch-target-injection}

攻撃者は分岐予測器を汚染し、投機的実行を選んだガジェットへ誘導する。

**修正**: Retpoline。間接呼び出しを、投機的実行を閉じ込めるシーケンスに置き換える。

**コミット**: [76b043848fd2](https://git.kernel.org/linus/76b043848fd2)（"x86/retpoline: Add initial retpoline support"）

---

### ケース3: TLBフラッシュの競合 {#case-3-tlb-flush-races}

#### 何が起きたか {#what-happened_2}

TLBはページテーブルエントリをキャッシュする。ページテーブルが変わると、TLBをフラッシュしなければならない。変更とフラッシュの間の競合がセキュリティ脆弱性を引き起こす。

#### バグのパターン {#the-bug-pattern}

CPUは古いTLBエントリを使って、アンマップされているはずのpageにアクセスできてしまう。
1. ページテーブルエントリがクリアされる
2. TLBフラッシュが保留中（まだ完了していない）
3. 別のCPUが古いTLBエントリ経由でアクセスする
4. 解放済み/再割り当て済みのpageへのアクセスが成功してしまう

#### 例: CVE-2018-18281 {#example-cve-2018-18281}

`mremap()`システムコールはページテーブルエントリを移動したが、TLBのフラッシュが遅すぎた。

**コミット**: [eb66ae030829](https://git.kernel.org/linus/eb66ae030829)（"mremap: properly flush TLB before releasing the page"）

---

### ケース4: KASLRの回避 {#case-4-kaslr-bypasses}

#### KASLRとは何か? {#what-is-kaslr}

Kernel Address Space Layout Randomization（カーネルアドレス空間配置のランダム化）は、カーネルがロードされる場所をランダム化し、エクスプロイトを難しくする。

#### 回避手法 {#bypass-techniques}

| 手法 | 方法 |
|-----------|--------|
| 情報漏洩 | カーネルのポインタがユーザー空間に露出する |
| タイミングのサイドチャネル | アクセス時間を測定する |
| dmesgからの漏洩 | ログ内のカーネルアドレス |

#### ハードニング {#hardening}

```bash
# カーネルポインタを制限する
echo 2 > /proc/sys/kernel/kptr_restrict

# dmesgを制限する
echo 1 > /proc/sys/kernel/dmesg_restrict
```

---

### まとめ: 得られた教訓 {#summary-lessons-learned}

| バグ | 年 | 種類 | 緩和策 |
|-----|------|------|------------|
| Meltdown | 2017 | ハードウェア | KPTI |
| Spectre v1 | 2017 | ハードウェア | nospecバリア |
| Spectre v2 | 2017 | ハードウェア | Retpoline |
| TLBの競合 | 継続中 | ソフトウェア | 適切なフラッシュの順序 |
| KASLRの回避 | 継続中 | 情報漏洩 | ポインタの制限 |

**パターン**: ページテーブルのバグには、タイミング（TLBの競合、投機）や、誤りだと判明するハードウェアの挙動についての前提が絡むことが多い。

## 参照 {#references}

### 主要なコード {#key-code}

| ファイル | 説明 |
|------|-------------|
| [`include/linux/pgtable.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/pgtable.h) | 汎用のページテーブルAPI |
| [`arch/x86/include/asm/pgtable.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/pgtable.h) | x86のページテーブル定義 |
| [`mm/memory.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memory.c) | ページフォールト処理 |

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/mm/page_tables.rst`](https://docs.kernel.org/mm/page_tables.html)

### 関連 {#related}

- [overview](overview.md): メモリ管理の概要
- [glossary](glossary.md): PFN、TLB、MMUの定義
- [バグ索引](bugs/README.md): mmのカーネルバグすべての索引

## 参考文献 {#further-reading}

- [arch/x86/mm/pgtable.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/pgtable.c): x86固有のページテーブル割り当て、PTE操作のヘルパー、KPTIのページテーブル切り替え
- [include/linux/pgtable.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/pgtable.h): アーキテクチャ非依存のページテーブルAPI。`pgd_offset()`、`pte_present()`、`pte_mkwrite()`、およびPTEフラグのアクセサ一式
- [mm/memory.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memory.c): ページフォールトのエントリポイント`handle_mm_fault()`、COWの処理、`munmap()`時のページテーブルの解体
- `Documentation/admin-guide/mm/concepts.rst`: 仮想メモリ、ページテーブル、TLBについてのカーネルドキュメントの概説
- [Five-level page tables](https://lwn.net/Articles/717293/)（LWN、2017年）: 57ビットの仮想アドレス空間をサポートするためにP4Dレベルを追加した設計と根拠
- [KPTI: kernel page-table isolation](https://lwn.net/Articles/741878/)（LWN、2017年）: カーネルモードとユーザーモードで別々のページテーブルを維持することでMeltdownの緩和策がどう機能するか
- [page-fault](page-fault.md): ハードウェアのトラップから`handle_pte_fault()`を経てメモリの解決に至る、ページフォールト処理の完全なパス
- [tlb-optimization](tlb-optimization.md): TLBのバッチ処理、遅延フラッシュ、そしてカーネルがIPIの多いTLBシュートダウンをどう最小化するか
