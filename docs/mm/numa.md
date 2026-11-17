# NUMAのメモリ管理 {#numa-memory-management}

> 複数のノードにまたがるメモリを管理する

## NUMAとは何か {#what-is-numa}

NUMA（Non-Uniform Memory Access）は、どのCPUがどのメモリにアクセスするかによってメモリアクセス時間が変わるシステムを指す。各CPUには「ローカル」メモリ（高速）と「リモート」メモリ（低速）がある。

```
┌─────────────────────────────────────────────────────────┐
│                      NUMA System                        │
│                                                         │
│   Node 0                          Node 1                │
│  ┌──────────────┐                ┌──────────────┐       │
│  │    CPU 0-7   │                │   CPU 8-15   │       │
│  └──────┬───────┘                └──────┬───────┘       │
│         │ fast                          │ fast          │
│  ┌──────▼───────┐                ┌──────▼───────┐       │
│  │  Memory 64GB │◄───────────────►  Memory 64GB │       │
│  └──────────────┘     slow       └──────────────┘       │
│                    interconnect                         │
└─────────────────────────────────────────────────────────┘

CPU 0 accessing Node 0 memory: ~80-100ns (local)
CPU 0 accessing Node 1 memory: ~130-150ns (1.5-1.7x slower, varies by platform)
(measured with Intel MLC; see Lameter, "NUMA: An Overview", ACM Queue 2013)
```

## なぜNUMAが重要か {#why-numa-matters}

### スケーラビリティの問題 {#the-scalability-problem}

システムが4〜8 CPUを超えて大きくなるにつれ、単一のメモリバスがボトルネックになった。すべてのCPUが同じメモリ帯域を奪い合っていた。

### 解決策としてのNUMA {#numa-as-solution}

NUMAはメモリをノードに分割し、それぞれに専用の帯域を持たせる。ローカルアクセスは高速で、リモートアクセスはインターコネクトを経由するが、システムはスケールする。

| システムの種類 | CPU数 | メモリ帯域 |
|-------------|------|------------------|
| UMA（単一バス） | 1-8 | 共有、競合あり |
| NUMA（2ノード） | 8-32 | 2倍の帯域 |
| NUMA（8ノード） | 32-128+ | 8倍の帯域 |

## LinuxのNUMAサポート {#linux-numa-support}

### トポロジの検出 {#topology-discovery}

LinuxはファームウェアからNUMAトポロジを検出する（ACPIのSRAT/SLITテーブル）。

```bash
# NUMA トポロジを表示する
numactl --hardware

# 出力例:
# available: 2 nodes (0-1)
# node 0 cpus: 0 1 2 3 4 5 6 7
# node 0 size: 65536 MB
# node distances:
# node   0   1
#   0:  10  21
#   1:  21  10
```

距離行列は相対的なアクセスレイテンシを示す（10 = ローカル、それより大きい = リモート）。

### デフォルトの割り当てポリシー {#default-allocation-policy}

デフォルトでは、Linuxはタスクが実行されているノードからメモリを割り当てる（「ローカル割り当て」）。

```c
/* デフォルト: 現在のノードから割り当てる */
void *ptr = malloc(size);  /* ローカルノードに行く */
```

## メモリポリシー {#memory-policies}

Linuxはメモリポリシーを通じて、NUMA配置をきめ細かく制御する手段を提供する。

### ポリシーの種類 {#policy-types}

| ポリシー | 挙動 |
|--------|----------|
| `MPOL_DEFAULT` | ローカル割り当て（システムのデフォルト） |
| `MPOL_BIND` | 指定されたノードからのみ割り当てる |
| `MPOL_PREFERRED` | 指定されたノードを優先し、他にフォールバックする |
| `MPOL_INTERLEAVE` | 指定されたノード間でラウンドロビンする |
| `MPOL_LOCAL` | 常にローカルノードから割り当てる |

### ポリシーの設定 {#setting-policies}

**プロセス全体（set_mempolicy）**:
```c
#include <numaif.h>

unsigned long nodemask = 0x1;  /* ノード 0 */
set_mempolicy(MPOL_BIND, &nodemask, sizeof(nodemask) * 8);
/* 以降の割り当てはすべてノード 0 のみから */
```

**領域ごと（mbind）**:
```c
#include <numaif.h>

void *ptr = mmap(...);
unsigned long nodemask = 0x3;  /* ノード 0 と 1 */
mbind(ptr, size, MPOL_INTERLEAVE, &nodemask,
      sizeof(nodemask) * 8, 0);
/* この領域はノード 0-1 にインターリーブされる */
```

**コマンドライン（numactl）**:
```bash
# 特定の CPU とメモリノードで実行する
numactl --cpunodebind=0 --membind=0 ./myapp

# すべてのノードにメモリをインターリーブする
numactl --interleave=all ./myapp
```

### 各ポリシーをいつ使うか {#when-to-use-each-policy}

| ユースケース | 推奨ポリシー |
|----------|-------------------|
| 一般的なアプリケーション | `MPOL_DEFAULT`（カーネルに任せる） |
| メモリ帯域律速 | `MPOL_INTERLEAVE` |
| レイテンシ重視 | ローカルノードへの `MPOL_BIND` |
| 大きな共有バッファ | `MPOL_INTERLEAVE` |
| データベースのバッファプール | `MPOL_BIND` または `MPOL_INTERLEAVE` |

## 自動NUMAバランシング {#automatic-numa-balancing}

v3.8以降、Linuxはpageを、それにアクセスするCPUの近くへ自動的にマイグレーションできる。

### 仕組み {#how-it-works}

1. カーネルは定期的にpageをアンマップする（PTEを「NUMAヒントフォールト」用にマークする）
2. アクセスされると、NUMAヒントフォールトが発生する
3. カーネルはどのノードがそのpageにアクセスしたかを記録する
4. pageがリモートノードから頻繁にアクセスされていれば、マイグレーションする

```
Page on Node 0, accessed by CPU on Node 1
              │
              v
      NUMA hint fault
              │
              v
    Record access from Node 1
              │
              v
    Migrate page to Node 1
              │
              v
    Future accesses are local (fast)
```

### 設定 {#configuration}

```bash
# NUMA バランシングの有効化/無効化
cat /proc/sys/kernel/numa_balancing
echo 1 > /proc/sys/kernel/numa_balancing

# スキャン周期の上下限（スキャン間隔のミリ秒）
cat /proc/sys/kernel/numa_balancing_scan_period_min_ms  # スキャン間の最小時間
cat /proc/sys/kernel/numa_balancing_scan_period_max_ms  # スキャン間の最大時間
# カーネルはマイグレーションの活動に基づき、この範囲内でスキャン頻度を調整する
```

### トレードオフ {#trade-offs}

**利点**:

- アプリケーションの変更が不要
- 変化するアクセスパターンに適応する
- 配置の悪いワークロードを助ける

**コスト**:

- フォールト処理によるCPUオーバーヘッド
- マイグレーションのオーバーヘッド
- 明示的なポリシーと衝突しうる

## メモリ階層化（CXL時代） {#memory-tiering-cxl-era}

現代のシステムはCXL（Compute Express Link）によって新しいメモリ階層を追加する。

```
┌─────────────────────────────────────────────┐
│              Tiered Memory                  │
│                                             │
│  Tier 0: Local DRAM     (~80ns, expensive)  │
│  Tier 1: CXL Memory     (~200ns, cheaper)   │
│  Tier 2: Persistent Mem (~300ns, persistent)│
└─────────────────────────────────────────────┘
```

LinuxはCXLメモリを、距離の大きい追加のNUMAノードとして扱う。カーネルはコールドなpageを低速な階層へ降格（demote）し、ホットなpageを高速な階層へ昇格（promote）できる。

```bash
# メモリ階層を表示する（v6.1 以降）
cat /sys/devices/virtual/memory_tiering/memory_tier*/nodelist
```

## 監視 {#monitoring}

### ノードごとの統計 {#per-node-statistics}

```bash
# ノードごとのメモリ
cat /sys/devices/system/node/node*/meminfo

# NUMA のヒット/ミス
cat /proc/vmstat | grep numa
# numa_hit        - 優先ノードで満たされた割り当て
# numa_miss       - 優先ノード以外からの割り当て
# numa_foreign    - このノード向けだったが他に配置された page
# numa_interleave - インターリーブされた割り当て
# numa_local      - ローカルな割り当て
# numa_other      - ローカルでない割り当て
```

### プロセスごとのNUMAマップ {#per-process-numa-map}

```bash
# プロセスの page がどのノードにあるかを表示する
cat /proc/<pid>/numa_maps

# 例:
# 7f8a1000 default file=/lib/libc.so anon=1 dirty=1 N0=1
# 7f8a2000 interleave:0-1 anon=512 N0=256 N1=256
```

## 変遷 {#evolution}

### 初期のNUMA（v2.5-2.6、2003-2004年） {#early-numa-v25-26-2003-2004}

SGI、IBMなどが自社の大規模システムのために基本的なNUMAサポートを追加した。メモリポリシー（[`set_mempolicy`](https://man7.org/linux/man-pages/man2/set_mempolicy.2.html)、`mbind`）は[Andi Kleenによって導入された](https://lwn.net/Articles/79100/)（[KleenのNUMA API論文](https://halobates.de/numaapi3.pdf)も参照）。mempolicyのコードはgitの歴史より古い（v2.6.12の最初のインポートに含まれている）。

### cpusetsとの統合（v2.6.16、2005年） {#cpusets-integration-v2616-2005}

**コミット**: [68860ec10bcc](https://git.kernel.org/linus/68860ec10bcc) ("cpusets: automatic numa mempolicy rebinding")

**作者**: Paul Jackson (SGI)

*注: lore.kernel.org上の2008年以前のLKMLアーカイブはまばらである。コミットメッセージに設計が記されている。*

cpusetsによってプロセスを特定のノードに制約できるようになり、NUMAを意識したコンテナの分離が可能になった。

### 自動NUMAバランシング（v3.8、2013年） {#automatic-numa-balancing-v38-2013}

**マージ**: [3d59eebc5e13](https://git.kernel.org/linus/3d59eebc5e13) ("Merge tag 'balancenuma-v11'")

**作者**: Mel Gorman, Rik van Riel, Ingo Molnar, Peter Zijlstra

*注: lore.kernel.org上の2013年以前のLKMLアーカイブはインデックスが限られている。設計の議論については[LWNの記事](https://lwn.net/Articles/568870/)を参照。*

アクセスパターンに基づくカーネルレベルの自動pageマイグレーション。

### メモリ階層化（v5.18以降、2022年） {#memory-tiering-v518-2022}

**コミット**: [c574bbe91703](https://git.kernel.org/linus/c574bbe91703) ("NUMA balancing: optimize page placement for memory tiering system") | [LKML](https://lore.kernel.org/linux-mm/20220128082751.593478-1-ying.huang@intel.com/)

**作者**: Huang Ying (Intel)

異種メモリ（CXL、永続メモリ）を、自動的な昇格/降格を伴う追加のNUMA階層としてサポートする。v6.1で明示的なメモリ階層フレームワークによってさらに改良された（[992bf77591cb](https://git.kernel.org/linus/992bf77591cb)）。

## よくある問題 {#common-issues}

### リモートメモリへの圧力 {#remote-memory-pressure}

ローカルの枯渇により、割り当てがリモートノードに強制される。

**症状**: `/proc/vmstat` の `numa_miss` が高い

**解決策**:

- ワークロードのメモリをノード間でバランスさせる
- 大きな割り当てに `MPOL_INTERLEAVE` を使う
- 各ノードでメモリを予約する

### NUMAバランシングのオーバーヘッド {#numa-balancing-overhead}

過剰なマイグレーションとフォールト処理。

**症状**: `task_numa_fault` でのCPU使用率が高く、pageのマイグレーションが多い

**解決策**:

- ワークロードのアクセスパターンが安定しているなら無効にする: `echo 0 > /proc/sys/kernel/numa_balancing`
- スキャン頻度をチューニングする
- 明示的なメモリポリシーを使う

### ノード使用量の偏り {#unbalanced-node-usage}

他のノードに空きメモリがあるのに、1つのノードが枯渇する。

**デバッグ**: `/sys/devices/system/node/node*/meminfo` を確認する

**解決策**:

- 大きな割り当てに `MPOL_INTERLEAVE` を使う
- ワークロードの分散を改善する

## リファレンス {#references}

### 主要なコード {#key-code}

| ファイル | 説明 |
|------|-------------|
| [`mm/mempolicy.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mempolicy.c) | メモリポリシーの実装 |
| [`kernel/sched/fair.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/fair.c) | スケジューラ内のNUMAバランシング |
| [`mm/migrate.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/migrate.c) | pageのマイグレーション |

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/admin-guide/mm/numa_memory_policy.rst`](https://docs.kernel.org/admin-guide/mm/numa_memory_policy.html)

### メーリングリストでの議論 {#mailing-list-discussions}

- [Automatic NUMA Balancing V3 (RFC)](https://lore.kernel.org/linux-mm/20121116160852.GA4302@gmail.com/t/) - 4段階の設計を示したMel Gormanの2012年のRFC
- [NUMA balancing sysctls documentation](https://lore.kernel.org/lkml/1373065742-9753-2-git-send-email-mgorman@suse.de/) - 2013年7月のパッチシリーズ

### 関連 {#related}

- [page-allocator](page-allocator.md) - ノードごとのゾーン
- [reclaim](reclaim.md) - ノードごとの回収
- [glossary](glossary.md) - NUMAの用語

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [mm/mempolicy.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mempolicy.c) — `set_mempolicy()`、`mbind()`、`MPOL_BIND`、`MPOL_INTERLEAVE`、およびメモリポリシーの実装全体
- [kernel/sched/fair.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/fair.c) — 自動NUMAバランシング: `task_numa_work()`、`task_numa_fault()`、`task_numa_migrate()`
- [mm/migrate.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/migrate.c) — `migrate_misplaced_folio()` とpageマイグレーションのエンジン

### カーネルドキュメント {#kernel-documentation_1}

- `Documentation/admin-guide/mm/numa_memory_policy.rst` — メモリポリシーのモード、`set_mempolicy(2)`、`mbind(2)`、numactlとの連携についての公式リファレンス（[レンダリング版](https://docs.kernel.org/admin-guide/mm/numa_memory_policy.html)）

### manページ {#man-pages}

- [`numactl(8)`](https://man7.org/linux/man-pages/man8/numactl.8.html) — CPUとメモリノードのバインドのためのコマンドラインインターフェース
- [`set_mempolicy(2)`](https://man7.org/linux/man-pages/man2/set_mempolicy.2.html) — プロセス全体のNUMAメモリポリシー
- [`mbind(2)`](https://man7.org/linux/man-pages/man2/mbind.2.html) — 領域ごとのNUMAメモリポリシー

### LWNの記事 {#lwn-articles}

- [NUMA in a hurry](https://lwn.net/Articles/486858/) — NUMAの概念とLinuxのメモリポリシーAPIの概要
- [Automatic NUMA balancing](https://lwn.net/Articles/568870/) — カーネルの自動pageマイグレーションの設計と実装（v3.8）
- [Memory tiering and CXL](https://lwn.net/Articles/898766/) — CXLが明示的なメモリ階層によってNUMAノードモデルをどう拡張するか

### 関連ドキュメント {#related-docs}

- [NUMAトポロジの検出: ACPI SRATとSLIT](numa-acpi-srat.md) — ファームウェアのテーブルがどう解析されてノードトポロジが構築されるか
- [NUMA距離とソケット間レイテンシ](numa-distance.md) — カーネルがノード間距離をどう定量化し、利用するか
- [NUMAのzonelist構築とフォールバック順序](numa-zonelist.md) — ページアロケータのフォールバックリストがNUMA距離からどう構築されるか
- [NUMAがメモリ回収に与える影響](numa-reclaim.md) — ノードごとのkswapd、ウォーターマーク、NUMAバランシングとの相互作用
- [ゾーン回収ポリシー](zone-reclaim.md) — `vm.zone_reclaim_mode` と、ローカルでの回収がリモート割り当てに勝るのはいつか
