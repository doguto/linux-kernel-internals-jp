# メモリ管理（mm/） {#memory-management-mm}

> Linuxは最も基本的なリソースであるメモリをどう管理しているか

## はじめに {#getting-started}

このドキュメントは、Linuxがメモリをどう管理しているかを説明する。理論だけでなく、30年以上の開発を通じて積み重ねられた実際の実装上の判断、トレードオフ、そして得られた教訓を扱う。

### 前提知識 {#prerequisites}

このドキュメントは次の事項に通じていることを前提とする。

- **Cプログラミング** - カーネルはCで書かれている
- **ポインタとメモリアドレス** - 仮想アドレスと物理アドレス
- **OSの基本概念** - プロセス、カーネルとユーザー空間

仮想メモリとページングを学んだことがあれば、それが良い基礎になる。推奨する教科書は[参考文献](#further-reading)を参照。

### カーネルソースの入手 {#getting-the-kernel-source}

```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
```

ビルドと設定については[Documentation/admin-guide/README.rst](https://docs.kernel.org/admin-guide/README.html)を参照。

### テストの実行 {#running-tests}

マシンを再起動せずにカーネルの変更をテストする最速の方法:

```bash
# virtme-ng をインストールする（手元のファイルシステムで QEMU 上のカーネルを実行する）
pip install virtme-ng

# テストモジュール付きでビルドして起動する
virtme-ng --kdir . --append 'test_vmalloc.run_test_mask=0xFFFF'

# 結果を確認する
# （dmesg の出力は仮想コンソールに表示される）
```

テストに関する包括的なドキュメントは[Documentation/dev-tools/testing-overview.rst](https://docs.kernel.org/dev-tools/testing-overview.html)を参照。

### 推奨する読む順番 {#suggested-reading-order}

**基礎:**

1. **[概要](overview.md)** - mm/がどう進化してきたかの物語
2. **[ページアロケータ](page-allocator.md)** - バディシステムの基礎
3. **[slab](slab.md)** - 小さなオブジェクトのためのSLUBアロケータ
4. **[vmalloc](vmalloc.md)** - 仮想メモリの割り当て

**アドレス変換とプロセスのメモリ:**

5. **[ページテーブル](page-tables.md)** - 仮想アドレスの変換
6. **[カーネルのアドレス空間](kernel-address-space.md)** - ダイレクトマップ、vmemmap、vmalloc空間、KASLR
7. **[プロセスのアドレス空間](mmap.md)** - VMA、mmap、デマンドページング

**メモリプレッシャーと回収:**

8. **[ページ回収](reclaim.md)** - LRU、kswapd、MGLRU
9. **[スワップ](swap.md)** - メモリをディスクに拡張する
10. **[ページキャッシュ](page-cache.md)** - ファイルデータのキャッシュ

**発展的なトピック:**

11. **[メモリcgroup](memcg.md)** - コンテナのメモリ制限
12. **[NUMA](numa.md)** - 複数ノードのメモリ管理
13. **[Transparent Huge Pages](thp.md)** - 自動的なヒュージページ
14. **[コンパクション](compaction.md)** - メモリのデフラグ
15. **[KSM](ksm.md)** - VMのためのページ重複排除

**エンドツーエンドのウォークスルー:**

基礎を学んだ後は、これらで操作をスタック全体にわたって追跡する。

16. **[mallocの一生](life-of-malloc.md)** - ユーザー空間から物理pageまで
17. **[pageの一生](life-of-page.md)** - 割り当てから回収まで
18. **[fork()したときに何が起きるか](fork.md)** - COWの仕組み
19. **[メモリ不足](oom.md)** - OOM killに至る経路
20. **[ファイル読み込みの一生](life-of-read.md)** - ページキャッシュの実際の動作
21. **[スワップ処理の間に何が起きるか](swapping.md)** - スワップアウトとスワップイン

### 学べること {#what-youll-learn}

| 教科書の概念 | Linuxの実際 |
|------------------|---------------|
| 「mallocはメモリを割り当てる」 | [違う。アドレス空間を予約するだけである](life-of-malloc.md)。物理pageは後からデマンドページングによって与えられる |
| 「forkはプロセスのメモリをコピーする」 | [Copy-on-write](fork.md)により、pageは書き込まれるまで共有される |
| 「ページテーブルは仮想アドレスを物理アドレスにマップする」 | 多段のページテーブル、TLBフラッシュのコスト、[遅延作成](life-of-malloc.md) |
| 「メモリアロケータ」 | 用途ごとに複数のアロケータがある（slab、vmalloc、buddy） |
| 「断片化」 | なぜ物理的に連続したメモリの確保は難しいのか、その解決策としてのvmalloc |
| 「メモリは有限である」 | [ウォーターマーク、kswapd、直接回収、OOM killer](oom.md) - 崖ではなく段階的な進行 |
| 「ファイルの読み込みはディスクにアクセスする」 | [ページキャッシュ](life-of-read.md)により、ほとんどの読み込みはメモリから行われる |
| 「スワップは遅い」 | [スワップの仕組み](swapping.md) - いつ、なぜ性能を損なうのかを理解する |

### このドキュメントの特徴 {#what-makes-this-different}

- **実際のコミット**: すべての主張は実際のカーネルコミットにリンクしている
- **実際のバグ**: 何が壊れたか、なぜ壊れたかから学ぶ
- **実際の議論**: 判断が議論されたLKMLへのリンク
- **実際のトレードオフ**: 「何を」だけでなく「なぜこれであれではないのか」
- **ハンズオン**: 実行するコマンドや読むべきコードを含む「Try It Yourself」セクション

---

## ドキュメント {#documentation}

### 中核となる読み物 {#core-reading}

| ドキュメント | 学べること |
|----------|-------------------|
| [overview](overview.md) | Linuxのメモリ管理の30年の物語 |
| [page-allocator](page-allocator.md) | バディシステム - 物理pageの管理 |
| [slab](slab.md) | SLUB - 小さなオブジェクトの割り当て |
| [vmalloc](vmalloc.md) | 仮想的に連続した割り当て |
| [vrealloc](vrealloc.md) | 割り当てのサイズ変更 |
| [page-tables](page-tables.md) | 仮想アドレスの変換 |
| [kernel-address-space](kernel-address-space.md) | カーネル側の半分: ダイレクトマップ、vmemmap、KASLR |
| [reclaim](reclaim.md) | メモリプレッシャー下で何が起きるか |
| [memcg](memcg.md) | コンテナのメモリ制限 |
| [thp](thp.md) | Transparent Huge Pages - 自動的な2MBのpage |
| [numa](numa.md) | NUMAのメモリポリシーとバランシング |
| [compaction](compaction.md) | 大きな割り当てのためのメモリのデフラグ |
| [ksm](ksm.md) | Kernel Same-page Merging - メモリの重複排除 |
| [mmap](mmap.md) | プロセスのアドレス空間とVMA |
| [page-cache](page-cache.md) | メモリ上でのファイルデータのキャッシュ |
| [swap](swap.md) | メモリをディスクに拡張する |

### ライフサイクル {#lifecycle}

メモリ操作をエンドツーエンドで追う物語形式のウォークスルー:

| ドキュメント | 学べること |
|----------|-------------------|
| [life-of-malloc](life-of-malloc.md) | ユーザー空間から物理pageまでmalloc()を追跡する |
| [life-of-page](life-of-page.md) | 物理pageが割り当て、使用、回収を経る旅 |
| [fork](fork.md) | COWの設定、ページテーブルのコピー、COWフォールトの処理 |
| [oom](oom.md) | ウォーターマーク、kswapd、直接回収、OOM killer |
| [life-of-read](life-of-read.md) | read()がページキャッシュをどう流れるか |
| [swapping](swapping.md) | スワップアウトとスワップインの仕組みの詳細 |

### 解説 {#explainers}

メモリ管理に関するよくある疑問に答える概念的なガイド:

| ドキュメント | 学べること |
|----------|-------------------|
| [virtual-physical-resident](virtual-physical-resident.md) | VSZ、RSS、PSS、USSの違い - なぜメモリの数値は合計が合わないのか |
| [overcommit](overcommit.md) | なぜオーバーコミットが唯一の妥当なデフォルトなのか |
| [brk-vs-mmap](brk-vs-mmap.md) | カーネルからメモリを得る2つの方法 |
| [contiguous-memory](contiguous-memory.md) | 空きメモリがあるのに大きな割り当てが失敗する理由 |
| [cow](cow.md) | Copy-on-writeのエッジケースと、それが破綻するとき |

### カーネルのバグとCVE {#kernel-bugs-cves}

Linuxカーネルの注目すべき脆弱性、データ破損バグ、エッジケースの分析:

| ドキュメント | 学べること |
|----------|-------------------|
| [バグ索引](bugs/README.md) | 深刻度・年・サブシステム別のmmバグ一覧 |
| [SLUBのバグ](slab.md#notorious-bugs-and-edge-cases) | ヒープの悪用（CVE-2021-22555、CVE-2022-29582） |
| [THPのバグ](thp.md#notorious-bugs-and-edge-cases) | khugepagedの競合、COWの問題（CVE-2020-29368） |
| [ページテーブルのバグ](page-tables.md#notorious-bugs-and-edge-cases) | Meltdown、Spectre、TLBフラッシュの競合 |

---

## 主要なソースファイル {#key-source-files}

実際のコードを読みたい場合:

| ファイル | 役割 |
|------|--------------|
| [`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) | バディアロケータ - 物理pageの管理 |
| [`mm/slub.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/slub.c) | SLUBアロケータ - 小さなオブジェクトのキャッシュ |
| [`mm/vmalloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmalloc.c) | vmalloc - 仮想的に連続した割り当て |
| [`mm/vmscan.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) | プレッシャー下でのメモリ回収 |
| [`mm/memcontrol.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c) | メモリcgroup |
| [`mm/huge_memory.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/huge_memory.c) | Transparent Huge Pages |
| [`mm/mempolicy.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mempolicy.c) | NUMAのメモリポリシー |
| [`mm/compaction.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/compaction.c) | メモリコンパクション |
| [`mm/ksm.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/ksm.c) | Kernel Same-page Merging |
| [`mm/mmap.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mmap.c) | プロセスのアドレス空間、VMA |
| [`mm/filemap.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/filemap.c) | ページキャッシュ |
| [`mm/swapfile.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/swapfile.c) | スワップ領域の管理 |
| [`mm/oom_kill.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/oom_kill.c) | OOM killer、犠牲プロセスの選択 |
| [`kernel/fork.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/fork.c) | プロセスの生成、dup_mm() |
| [`include/linux/gfp.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/gfp.h) | GFPフラグの定義 |

---

## 参考文献 {#further-reading}

### 無料のリソース {#free-resources}
- Bonwick, [*The Slab Allocator*](https://people.eecs.berkeley.edu/~kubitron/courses/cs194-24-S14/hand-outs/bonwick_slab.pdf) (1994) - USENIXで発表されたslabの原論文
- Gorman, [*Understanding the Linux Virtual Memory Manager*](https://www.kernel.org/doc/gorman/) - 無料のオンライン書籍

### カーネルのドキュメント {#kernel-documentation}
- カーネルツリーの[Documentation/mm/](https://docs.kernel.org/mm/)
- システム管理者向けの観点からは[Documentation/admin-guide/mm/](https://docs.kernel.org/admin-guide/mm/)

### 教科書 {#textbooks}
- Silberschatz et al., [*Operating System Concepts*](https://www.os-book.com/) - 第9〜10章（メモリ）
- Tanenbaum & Bos, [*Modern Operating Systems*](https://www.pearson.com/en-us/subject-catalog/p/modern-operating-systems/P200000003295) - 第3章（メモリ管理）
- Love, [*Linux Kernel Development*](https://www.oreilly.com/library/view/linux-kernel-development/9780768696974/) - 第12章（メモリ管理）

---

## 付録 {#appendix}

- **[用語集](glossary.md)** - 用語のリファレンス
