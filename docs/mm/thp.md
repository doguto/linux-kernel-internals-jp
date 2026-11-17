# Transparent Huge Pages（THP） {#transparent-huge-pages-thp}

> アプリケーションを変更せずにhuge pageを自動的に割り当てる

## THPとは何か {#what-is-thp}

Transparent Huge Pagesは、可能な場合に通常の4KBページを2MBのhuge pageへ自動的に昇格させる。hugetlbfsと異なり、THPはアプリケーションの変更を必要としない。カーネルがすべてを透過的に処理する。

```
Without THP:                    With THP:
┌─────┬─────┬─────┬─────┐      ┌─────────────────────┐
│ 4KB │ 4KB │ 4KB │ ... │      │       2MB           │
│     │     │     │     │      │    (512 x 4KB)      │
└─────┴─────┴─────┴─────┘      └─────────────────────┘
   512 TLB entries needed         1 TLB entry needed
```

## hugetlbfsからTHPへ {#from-hugetlbfs-to-thp}

### 最初のhuge pageインターフェース（hugetlbfs、Linux 2.6） {#the-original-huge-page-interface-hugetlbfs-linux-26}

ハードウェアのhuge page（x86-64では2MB）はPentium Proの時代から利用可能だった。Linuxはこれを`hugetlbfs`を通じて公開していた。これは特殊なファイルシステムで、huge pageを裏付けとするファイルを`MAP_HUGETLB`で`mmap()`できる。

問題は運用面にあった。huge pageは、専用の事前割り当てプールからブート時に（または`sysctl vm.nr_hugepages`で）予約しておく必要があった。アプリケーションは別の`mmap()`フラグ（`MAP_HUGETLB`）を使うか、`hugetlbfs`上のファイルを開く必要があった。OracleやPostgreSQLのようなデータベースはこれに対応していたが、プールのサイズを決めるための管理者の介入と、APIを使うための開発者の労力が必要だった。

プールが小さすぎると、アプリケーションは4KBページにフォールバックした。大きすぎると、事前予約したメモリが遊んだままになった。しかも完全にオプトイン方式だったため、明示的にhuge pageを要求しない何千ものアプリケーションは、ワーキングセットが大きく劇的な恩恵を受けられる場合でも、何の恩恵も受けられなかった。

### Transparent Huge Pages（Linux 2.6.38、2011年） {#transparent-huge-pages-linux-2638-2011}

Andrea Arcangeliは、アプリケーションに一切の変更を加えずにhuge pageの性能を届けるためにTHPを設計した。アラインメント、空き状況、`transparent_hugepage`ポリシーに基づき、512個の4KBページの代わりに2MBのhuge pageをいつ割り当てるかをカーネル自身が決める。

鍵となる着想は次のとおりである。匿名メモリ（ヒープ、スタック）については、2MBにアラインされた物理的に連続な範囲が利用可能であれば、カーネルはフォールト時に2MBアラインのhuge pageを割り当てられる。アプリケーションからは通常の仮想メモリに見え、ハードウェアはページテーブルを1段少なくたどるだけで済む。

コンパクションの課題により、THPのチューニングは複雑になった。物理メモリは時間とともに断片化し、ページの割り当てと解放が繰り返されるにつれて2MBの連続範囲は希少になる。カーネルの`khugepaged`デーモンは、huge pageにまとめ上げ（collapse）られる小さなページの領域を探してスキャンし、メモリコンパクションの仕組み（`MADV_HUGEPAGE`、バックグラウンドコンパクション、デフラグモード）は連続範囲を作り出すために存在する。コンパクションのストールによるレイテンシスパイクを起こさずにこれを正しく機能させるには、何年もの改良が必要だった。

`transparent_hugepage=defer`モード（Linux 4.14以降）は、フォールト時にhuge pageの割り当てを試みるが、コンパクションは`khugepaged`に先送りし、アプリケーションのホットパスで同期的なコンパクションのレイテンシが発生するのを避ける。`MADV_HUGEPAGE` / `MADV_NOHUGEPAGE`によるプロセスごとの制御により、性能が重要なアプリケーションはオプトインし、それ以外は4KBページを使うといった使い分けができる。

## なぜTHPか {#why-thp}

### TLBの問題 {#the-tlb-problem}

TLB（Translation Lookaside Buffer）は、仮想アドレスから物理アドレスへの変換を[多段の階層](https://lwn.net/Articles/379748/)でキャッシュする。

現代のx86-64では、マイクロアーキテクチャによってL1 dTLBは64〜96エントリ、L2（STLB）は1024〜3072エントリを保持する（出典: [7-cpu.com](https://www.7-cpu.com/cpu/Skylake.html)、[WikiChip](https://en.wikichip.org/wiki/intel/microarchitectures/skylake_(server)#Memory_Hierarchy)）。4KBページでは、数GB程度のワーキングセットでもTLBを使い果たしてしまう。

| ワークロードのサイズ | 必要なページ数 | TLBへの圧力 |
|---------------|--------------|--------------|
| 1MB | 256 | 低 |
| 1GB | 262,144 | 高 |
| 100GB | 26,214,400 | 深刻 |

大きなワークロードは絶えずTLBエントリを追い出し、高コストなページテーブルウォークを引き起こす。

### 解決策としてのhuge page {#huge-pages-as-solution}

2MBページはTLBへの圧力を512分の1に減らす。

| ページサイズ | 1GBに必要なページ数 | TLBエントリ数 |
|-----------|---------------|-------------|
| 4KB | 262,144 | 262,144 |
| 2MB | 512 | 512 |
| 1GB | 1 | 1 |

## THPの仕組み {#how-thp-works}

### 2つの割り当てパス {#two-allocation-paths}

**1. フォールト時（同期）**

プロセスがメモリでフォールトすると、カーネルはhuge pageの割り当てを試みる。

```
Page fault
    │
    v
Can we allocate 2MB?
    │
    ├── Yes: Map huge page
    │
    └── No: Fall back to 4KB pages
```

**2. khugepaged（非同期）**

カーネルスレッドが、huge pageにまとめ上げられる領域を探してスキャンする。

```
khugepaged thread
    │
    v
Scan process VMAs
    │
    v
Find 512 contiguous 4KB pages?
    │
    ├── Yes: Collapse into 2MB huge page
    │
    └── No: Skip, try again later
```

## 設定 {#configuration}

### グローバル設定 {#global-settings}

```bash
# 現在のモードを表示する
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never

# 選択肢:
# always  - すべての匿名メモリでTHPを使う
# madvise - MADV_HUGEPAGE領域でのみTHPを使う
# never   - THPを無効化する
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
```

### デフラグ設定 {#defrag-settings}

```bash
# huge pageのためにいつメモリをデフラグするか
cat /sys/kernel/mm/transparent_hugepage/defrag
# [always] defer defer+madvise madvise never

# always        - フォールト時に同期デフラグ（ストールし得る）
# defer         - khugepagedによる非同期デフラグ
# defer+madvise - 大半はdefer、MADV_HUGEPAGEでは同期
# madvise       - MADV_HUGEPAGEの場合のみデフラグ
# never         - THPのためにデフラグしない
```

### プロセスごとの制御 {#per-process-control}

```c
#include <sys/mman.h>

void *ptr = mmap(NULL, size, PROT_READ|PROT_WRITE,
                 MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);

/* この領域でTHPを有効化する */
madvise(ptr, size, MADV_HUGEPAGE);

/* この領域でTHPを無効化する */
madvise(ptr, size, MADV_NOHUGEPAGE);
```

### khugepagedのチューニング {#khugepaged-tuning}

```bash
# スキャン間隔（ミリ秒）
cat /sys/kernel/mm/transparent_hugepage/khugepaged/scan_sleep_millisecs
# 10000（デフォルト）

# 1回の間隔でスキャンするページ数
cat /sys/kernel/mm/transparent_hugepage/khugepaged/pages_to_scan
# 4096（デフォルト）

# まとめ上げ時に許容するマップされていないPTEの最大数
cat /sys/kernel/mm/transparent_hugepage/khugepaged/max_ptes_none
# 511（デフォルト - 512個中511個のPTEがnone/ゼロでもまとめ上げる）
```

（すべてのチューニング可能なパラメータについては[THPのカーネルドキュメント](https://docs.kernel.org/admin-guide/mm/transhuge.html)を参照。）

## THPと特定のワークロード {#thp-and-specific-workloads}

### データベース {#databases}

データベースはTHPの恩恵を受けることが多いが、次の問題に悩まされることもある。

- コンパクション中のレイテンシスパイク
- メモリの肥大化（2MB単位）

多くのデータベースは、明示的にhuge pageを要求する`madvise`モードを推奨している。

### 仮想マシン（KVM） {#virtual-machines-kvm}

THPは元々KVMのために設計された。

コミット[71e3aac0724f](https://git.kernel.org/linus/71e3aac0724f)より:
> "Lately I've been working to make KVM use hugepages transparently without the usual restrictions of hugetlbfs."

（訳: 最近、hugetlbfsの通常の制約なしにKVMがhuge pageを透過的に使えるようにする作業をしている。）

VMは、ネストされたページテーブルにおけるTLBへの圧力が減ることで大きな恩恵を受ける。

### Redis/Memcached {#redismemcached}

インメモリストアでは、THPのデフラグによるレイテンシスパイクが見られることがある。多くの場合、次の設定が推奨される。
```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

あるいは、`madvise`モードを使ってアプリケーションレベルで制御する。

## THPとhugetlbfsの比較 {#thp-vs-hugetlbfs}

| 特徴 | THP | hugetlbfs |
|---------|-----|-----------|
| アプリケーションの変更 | 不要（ヒントとしてMADV_HUGEPAGEを使う場合あり） | 必要 |
| ページサイズ | 2MB（PMD）、mTHPでは16KB〜1MB（v6.10以降） | 2MB、1GB |
| スワップ可能 | 可（v4.13以降） | 不可 |
| 予約 | オンデマンド | 事前割り当て |
| 断片化のリスク | 高い | 低い |
| オーバーコミット | 可 | 不可 |

## 監視 {#monitoring}

### システム全体の統計 {#system-wide-statistics}

```bash
# THPの統計
cat /proc/meminfo | grep -i huge
# AnonHugePages:    2097152 kB  <- THPの使用量
# HugePages_Total:       0     <- hugetlbfs
# HugePages_Free:        0

# THPの詳細な統計
cat /proc/vmstat | grep thp
# thp_fault_alloc        - フォールト時に割り当てられたhuge page
# thp_fault_fallback     - 4KBにフォールバックした
# thp_collapse_alloc     - khugepagedによるまとめ上げ
# thp_split_page         - 4KBに分割し直されたhuge page
```

### プロセスごと {#per-process}

```bash
# プロセスのhuge page使用量
cat /proc/<pid>/smaps | grep -i huge
# AnonHugePages:     2048 kB

# または集計値
cat /proc/<pid>/smaps_rollup | grep AnonHugePages
```

### トレーシング {#tracing}

```bash
# THPイベントをトレースする
echo 1 > /sys/kernel/debug/tracing/events/huge_memory/mm_khugepaged_scan_pmd/enable
echo 1 > /sys/kernel/debug/tracing/events/huge_memory/mm_collapse_huge_page/enable
cat /sys/kernel/debug/tracing/trace_pipe
```

## 歴史 {#history}

### THPの導入（v2.6.38、2011年） {#thp-introduction-v2638-2011}

**コミット**: [71e3aac0724f](https://git.kernel.org/linus/71e3aac0724f)（"thp: transparent hugepage core"）

**作者**: Andrea Arcangeli（Red Hat）

*注: lore.kernel.org上の2012年以前のLKMLアーカイブは不完全である。設計の根拠はコミットメッセージに記されている。*

匿名メモリ向けのTHPの初期実装。

### THPのスワップ対応（v4.13、2017年） {#thp-swap-support-v413-2017}

**コミット**: [38d8b4e6bdc8](https://git.kernel.org/linus/38d8b4e6bdc8)（"mm, THP, swap: delay splitting THP during swap out"） | [LKML](https://lore.kernel.org/lkml/20170724051840.2309-1-ying.huang@intel.com/)

**作者**: Huang Ying

THPのページを、先に分割することなく1つの単位としてスワップできるようになり、スワップ性能が向上した。

### Multi-Size THP（v6.10、2024年） {#multi-size-thp-v610-2024}

**コミット**: [19eaf44954df](https://git.kernel.org/linus/19eaf44954df)（"mm: thp: support allocation of anonymous multi-size THP"） | [LKML](https://lore.kernel.org/linux-mm/20231204102027.57185-1-ryan.roberts@arm.com/)

**作者**: Ryan Roberts

4KBと2MBの間のサイズ（例: 16KB、64KB）でTHPを使えるようにし、2MBでは大きすぎるワークロードに対してより柔軟性を提供する。

## よくある問題 {#common-issues}

### レイテンシスパイク {#latency-spikes}

THPのデフラグは割り当てのストールを引き起こし得る。

**症状**: レイテンシに敏感なアプリケーションで不定期にレイテンシスパイクが起きる

**解決策**:

- `defrag=defer`または`defrag=madvise`を使う
- `enabled=madvise`に設定し、領域ごとに制御する
- レイテンシが重要なアプリではTHPを無効化する

### メモリの肥大化 {#memory-bloat}

小さな割り当てが2MBに切り上げられる。

**症状**: 予想よりもメモリ使用量が多い

**デバッグ**: `/proc/meminfo`の`AnonHugePages`と`Anonymous`を比較する

**解決策**:

- `madvise`モードを使う
- アプリケーションの割り当てパターンを調整する

### khugepagedのCPU使用率 {#khugepaged-cpu-usage}

khugepagedがCPUを過剰に消費する。

**デバッグ**: `top`または`perf top`

**解決策**:

- `scan_sleep_millisecs`を増やす
- `pages_to_scan`を減らす

## 悪名高いバグとエッジケース {#notorious-bugs-and-edge-cases}

THPは、ページのまとめ上げ、huge pageの分割、フォールトの処理といった複雑な操作を、プロセスが実行を続けている間に行う。この複雑さにより、数多くの競合状態やデータ破壊のバグが生じてきた。

### 事例1: THPのCOWの競合（CVE-2020-29368） {#case-1-thp-cow-race-cve-2020-29368}

#### 何が起きたか {#what-happened}

Jann Horn（Google Project Zero）は、THPのcopy-on-write実装が、mapcountのチェックにおける競合状態のために、意図しない書き込みアクセスを許してしまう可能性があることを発見した。

#### バグ {#the-bug}

COWフォールト中にhuge PMDを分割する際、カーネルは`mapcount`を調べてページが共有されているかをチェックする。しかしこのチェックは他の操作と競合する。チェックからページテーブルの変更までの間に、別のスレッドが同じページをマップできてしまう。

#### 修正 {#the-fix}

**コミット**: [c444eb564fb1](https://git.kernel.org/linus/c444eb564fb1)（"mm: thp: make the THP mapcount atomic against __split_huge_pmd_locked()"）

**作者**: Andrea Arcangeli

---

### 事例2: khugepagedのまとめ上げにおける競合 {#case-2-khugepaged-collapse-races}

#### 何が起きたか {#what-happened_1}

khugepagedデーモンは、2MBのhuge pageにまとめ上げられる512個の連続した4KBページを探してメモリをスキャンする。このバックグラウンド操作はアプリケーションの動作と競合する。

#### バグの類型 {#the-bug-class}

khugepagedのまとめ上げでは複数のバグが見つかっている。

- 並行するページフォールトとの競合
- munmapとの競合
- madvise(MADV_DONTNEED)との競合

#### 現代のカーネルにおける緩和策 {#mitigations-in-modern-kernels}

- khugepagedとフォールトパスの間のより良いロックプロトコル
- クリティカルセクション中にページロックを保持する
- ロック取得後の検証

---

### 事例3: khugepagedのCPUストーム {#case-3-khugepaged-cpu-storms}

#### 何が起きたか {#what-happened_2}

特定の条件下では、khugepagedが過剰なCPU時間を消費し、アプリケーションの性能に影響を与えることがある。

#### バグ {#the-bug_1}

khugepagedはまとめ上げ可能なページを継続的にスキャンする。まとめ上げを妨げる条件がありながら見込みがありそうに見える場合（例: 512ページ中500ページが存在する）、無駄に再試行を続けてしまう。

#### チューニング {#tuning}

```bash
# スキャン頻度を下げる
echo 60000 > /sys/kernel/mm/transparent_hugepage/khugepaged/scan_sleep_millisecs

# または無効化する
echo 0 > /sys/kernel/mm/transparent_hugepage/khugepaged/defrag
```

---

### 事例4: THPの分割失敗 {#case-4-thp-split-failures}

#### 何が起きたか {#what-happened_3}

huge pageを分割する必要がある場合（例: 部分的なmunmap）、分割が失敗してシステムが予期しない状態に置かれることがある。

#### バグの類型 {#the-bug-class_1}

分割には、511個の追加のページディスクリプタを割り当て、すべてのPTEを更新する必要がある。失敗のシナリオには、メモリの枯渇やページをフリーズできないことが含まれる。

#### 緩和策 {#mitigations}

現代のカーネルは、再試行ロジックとフォールバックパスによって分割失敗をより穏便に処理する。

---

### 事例5: THPとNUMAバランシングの衝突 {#case-5-thp-and-numa-balancing-conflicts}

#### 何が起きたか {#what-happened_4}

THPは、アクセスしているCPUの近くへページを移動させようとするNUMAバランシングとの相性が悪い。

#### バグ {#the-bug_2}

NUMAバランシングは、フォールトを捕捉してマイグレーションするためにページをアクセス不可にマークする。2MBのTHPでは、内部のどの4KBへのアクセスでも2MB全体のマイグレーションが引き起こされ、ノード間でスラッシングが起きる。

#### チューニング {#tuning_1}

```bash
# NUMAバランシングを無効化する（THPのほうが価値が高い場合）
echo 0 > /proc/sys/kernel/numa_balancing

# またはTHPを無効化する
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

---

### まとめ: 得られた教訓 {#summary-lessons-learned}

| バグ | 年 | 根本原因 | 影響 |
|-----|------|------------|--------|
| CVE-2020-29368 | 2020 | mapcountの競合 | COWの回避 |
| khugepagedの競合 | 継続中 | ロック順序 | データ破壊 |
| CPUストーム | 継続中 | 積極的すぎるスキャン | 性能 |
| 分割失敗 | 継続中 | リソース枯渇 | ENOMEM |
| NUMAスラッシング | 継続中 | 設計上の衝突 | 性能 |

**パターン**: THPのバグは、同じメモリに対する並行操作に起因する。この最適化には、正しく実装するのが難しい複雑な協調が必要になる。

## 参考資料 {#references}

### 主要なコード {#key-code}

| ファイル | 説明 |
|------|-------------|
| [`mm/huge_memory.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/huge_memory.c) | THPのコア実装 |
| [`mm/khugepaged.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/khugepaged.c) | まとめ上げデーモン |

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/admin-guide/mm/transhuge.rst`](https://docs.kernel.org/admin-guide/mm/transhuge.html)

### メーリングリストでの議論 {#mailing-list-discussions}

- [THP defrag behavior](https://lore.kernel.org/patchwork/patch/745411/) - MADV_HUGEPAGEのストールのセマンティクスと、"defer"モードに関するMel Gormanの意図についての議論
- [Node-local hugepage allocations](https://lore.kernel.org/lkml/20181206045917.GG23332@shao2-debian/) - THPのgfp処理とリモート割り当てに関する2018年の議論

### 関連項目 {#related}

- [page-tables](page-tables.md) - PMDレベルのhuge pageマッピング
- [compaction](compaction.md) - THPのためのメモリデフラグ
- [reclaim](reclaim.md) - THPとメモリ圧迫
- [バグ索引](bugs/README.md) - mmのカーネルバグすべての索引

## 参考文献 {#further-reading}

- `Documentation/admin-guide/mm/transhuge.rst` — すべてのsysfsのチューニング可能なパラメータ、デフラグモード、監視カウンタを網羅した、権威あるカーネル管理ガイド。[docs.kernel.org](https://docs.kernel.org/admin-guide/mm/transhuge.html)で閲覧できる
- [mm/huge_memory.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/huge_memory.c) — THPのコア: フォールト時の割り当て、PMDレベルのマッピング、COWでの分割、`MADV_HUGEPAGE` / `MADV_COLLAPSE`の処理
- [mm/khugepaged.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/khugepaged.c) — `khugepaged`デーモン: スキャンロジック、まとめ上げのパス、そしていくつもの競合状態バグの原因となってきたロックプロトコル
- [LWN: Transparent huge pages](https://lwn.net/Articles/423584/) — Andrea Arcangeliによる最初のTHP設計と、hugetlbfsの制約を取り除く動機についての2011年の記事
- [LWN: Improving huge page handling](https://lwn.net/Articles/619738/) — 遅延コンパクション（`defer`デフラグモード）の作業と、それがレイテンシスパイクの問題にどう対処したかの分析
- コミット[71e3aac0724f](https://git.kernel.org/linus/71e3aac0724f) — v2.6.38でのTHPの最初のマージ。コミットメッセージには元々のKVMの動機が詳しく記されている
- コミット[38d8b4e6bdc8](https://git.kernel.org/linus/38d8b4e6bdc8) — v4.13のTHPスワップ対応。huge pageを分割せずに1つの単位としてスワップアウトできるようになった
- [mthp.md](mthp.md) — multi-size THP: 2MBでは粗すぎるワークロード向けにLinux 6.10で導入された、PMD未満のhuge page（16KB〜512KB）
- [hugetlbfs-vs-thp.md](hugetlbfs-vs-thp.md) — THPと明示的なhugetlbfsの並列比較。それぞれをどのような場合に選ぶべきかも含む
