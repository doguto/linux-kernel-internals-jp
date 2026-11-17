# メモリ帯域幅の最適化 {#memory-bandwidth-optimization}

> レイテンシがボトルネックでないとき — NUMAノード全体でスループットを最大化する

## 帯域幅の壁 {#the-bandwidth-wall}

NUMAに関するドキュメントの多くはレイテンシに焦点を当てている。データを、それを使うCPUの近くに置け、というものだ。しかし帯域幅律速のワークロード — 大規模な行列演算、ストリーミング分析、ML学習データのシャッフル、毎秒数百万のキーを提供するインメモリキャッシュ — では別のボトルネックが現れる。これらのワークロードは、個々のアクセスの高いレイテンシに苦しむというより、単一のNUMAノードが毎秒供給できるバイト数のハードな上限にぶつかるのである。

各NUMAノードには、そのDDRコントローラとチャネル構成によって決まる固定のメモリ帯域幅がある。デュアルチャネルのDDR5-4800システムは、チャネルあたりおよそ77 GB/s × 2チャネル ≈ **ノードあたり154 GB/s**を供給する。現代のサーバーCPUは、各コアがピーク時に20 GB/s以上のメモリトラフィックを駆動できる。計算はすぐに行き詰まる。

```
Node 0 peak bandwidth:   ~154 GB/s (2-channel DDR5-4800)
Peak per CPU core:       ~20 GB/s (sustained streaming)
Saturation point:        ~8 cores driving max bandwidth on one node

A 32-core system on one node: 4x over-subscribed
```

すべてのスレッドが同じNUMAノードから割り当てると、レイテンシはローカルになるが、スループットは人為的に1ノード分の帯域幅に制限される。割り当てをノード間に分散すると、多少のリモートアクセスのレイテンシと引き換えに、利用可能な帯域幅が倍増する。帯域幅律速のワークロードにとって、このトレードオフは有利である。

!!! note "これらの数値は例示である"
    実際の帯域幅は、DDRの世代、チャネル数、ハードウェアレベルのメモリインターリーブ、ワークロードのアクセスパターンに依存する。対象のハードウェアで`stream`を使って計測すること。

## 帯域幅かレイテンシか: 戦略の選択 {#bandwidth-vs-latency-choosing-a-strategy}

適切な戦略は、ワークロードがレイテンシ律速か帯域幅律速かによって決まる。

| ワークロードの種類 | 例 | ボトルネック | 戦略 |
|--------------|---------|------------|---------|
| レイテンシ律速 | OLTPデータベース、ミリ秒未満のSLOを持つインメモリキャッシュ | 操作ごとのアクセスレイテンシ | ローカルノードへの`MPOL_BIND`。データをCPUの近くに置く |
| 帯域幅律速 | 大規模行列乗算、MLのデータロード、Kafkaのコンシューマ、Redisクラスタ | 総GB/s | ノード間での`MPOL_INTERLEAVE`。レイテンシと帯域幅を引き換える |
| 混在 | 多様なワークロードを持つマルチテナントサーバー | テナントに依存 | VMAごとのポリシー。まずプロファイルする |

分岐点はワークロード固有である。512MBのワーキングセットに繰り返しアクセスするワークロードは、1ノードの帯域幅予算に余裕で収まる。毎秒200GBのデータをストリーミングするワークロードは収まらない。チューニングの前にプロファイルすること。

## MPOL_INTERLEAVE: カーネルの仕組み {#mpol_interleave-the-kernel-mechanism}

`MPOL_INTERLEAVE`は、メモリ割り当てをNUMAノード間に分散させるためのLinuxの主要な手段である。指定されたノードマスクにわたってpageの割り当てをラウンドロビンする。page 0はノード0へ、page 1はノード1へ、という具合である。各ノードは、その領域の物理pageのほぼ均等な分け前を受け取る。

実装は[`mm/mempolicy.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mempolicy.c)にある。プロセスレベルのインターリーブでは、`interleave_nodes()`が割り当てのたびに`current->il_prev`をポリシーのノードマスク内で進める。

```c
/* mm/mempolicy.c */
static unsigned int interleave_nodes(struct mempolicy *policy)
{
    unsigned int nid;
    unsigned int cpuset_mems_cookie;

    do {
        cpuset_mems_cookie = read_mems_allowed_begin();
        nid = next_node_in(current->il_prev, policy->nodes);
    } while (read_mems_allowed_retry(cpuset_mems_cookie));

    if (nid < MAX_NUMNODES)
        current->il_prev = nid;
    return nid;
}
```

VMAベースのインターリーブ（`mbind()`で使われる）では、`interleave_nid()`がVMA内でのpageのオフセットから対象ノードを導き出す。そのため、同じマッピングレイアウトでプロセスを再起動した後でも、同じpageは常に同じノードに対応する。

`MPOL_BIND`、`MPOL_PREFERRED`、`MPOL_LOCAL`を含むポリシーAPIの全体については、[NUMAメモリ管理](numa.md)を参照。

### MPOL_INTERLEAVEの適用 {#applying-mpol_interleave}

**プロセス単位（set_mempolicy）** — このプロセスによる以降のすべての割り当てが、全ノードにわたってインターリーブされる。

```c
#include <numaif.h>
#include <numa.h>

/* 利用可能な全ノードにわたってインターリーブする */
struct bitmask *all_nodes = numa_all_nodes_ptr;
set_mempolicy(MPOL_INTERLEAVE,
              all_nodes->maskp,
              all_nodes->size + 1);
```

**領域単位（mbind）** — 特定の割り当て済み領域にインターリーブを適用する。

```c
#include <numaif.h>
#include <sys/mman.h>

void *buf = mmap(NULL, size, PROT_READ | PROT_WRITE,
                 MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

unsigned long nodemask = 0x3;  /* ノード0と1 */
mbind(buf, size, MPOL_INTERLEAVE,
      &nodemask, 3,             /* maxnode = 3（ビット0と1が有効） */
      MPOL_MF_MOVE);            /* 既存の page を今すぐマイグレーションする */
```

**コマンドライン（numactl）** — コードを変更せずに既存のバイナリをラップする。

```bash
# 全ノードにわたってインターリーブする
numactl --interleave=all ./myapp

# 特定のノードにわたってインターリーブする
numactl --interleave=0,1 ./myapp
```

### MPOL_INTERLEAVEを使うべきとき {#when-to-use-mpol_interleave}

!!! tip "MPOL_INTERLEAVEに最適なワークロード"
    - **大規模なインメモリデータセット**: 1ノードの帯域幅予算を大きく上回るデータセットを提供するmemcachedやRedisクラスタのインスタンス
    - **科学技術計算の配列**: RAMには収まるが単一ノードの帯域幅を超える配列に対するNumPy/SciPyの演算
    - **ML学習のデータローダ**: 大規模データセットをGPUメモリへストリーミングする前処理パイプライン
    - **分析エンジン**: フルテーブルスキャンを行うインメモリのカラムナストア（DuckDB、DRAM上のSpark）

### MPOL_WEIGHTED_INTERLEAVE {#mpolweightedinterleave}

Linux 6.9で`MPOL_WEIGHTED_INTERLEAVE`が追加された（[commit fa3bea4e1f82](https://git.kernel.org/linus/fa3bea4e1f82)）。これはpageを均等ではなく比例的に分配する。ノード0が256 GB、ノード1が128 GBを持つなら、利用可能な容量に合わせて割り当てを2:1に重み付けできる。重みは`/sys/kernel/mm/mempolicy/weighted_interleave/nodeN`で設定する。

## ファーストタッチポリシーと帯域幅への影響 {#first-touch-policy-and-its-bandwidth-implications}

Linuxは**ファーストタッチ割り当て**を使う。pageは、そのpageへの最初の書き込みの時点でフォールトしたスレッドが動いているNUMAノード上に割り当てられる。これは、意図的なインターリーブが黙って失敗する最もよくある理由である。

問題のパターンは次のとおり。

```c
// メインスレッド（ノード0上）が大きなバッファを初期化する
// すべての page がノード0に配置される — ここでは MPOL_INTERLEAVE は効果がない
// なぜなら最初のタッチの「前に」mbind が呼ばれていないから
char *buf = malloc(10UL << 30);  /* 10 GB */
memset(buf, 0, 10UL << 30);      /* 最初のタッチ: すべての page がノード0へ */

// ここでポリシーを設定してワーカーを生成する — 手遅れ
set_mempolicy(MPOL_INTERLEAVE, ...);
spawn_workers(buf);
```

`set_mempolicy()`による`MPOL_INTERLEAVE`は将来の割り当てに影響する。すでにフォールトインされたpageには影響しない。同様に、`MPOL_MF_MOVE`なしの`mbind()`は将来のフォールトに対するポリシーを変更するが、既存のpageはマイグレーションしない。

!!! tip "ファーストタッチ問題の修正"
    **方法1 — 最初のタッチの前にポリシーを設定する:**
    ```c
    set_mempolicy(MPOL_INTERLEAVE, &all_nodes_mask, max_node);
    char *buf = malloc(size);
    memset(buf, 0, size);  /* 最初のタッチが正しくインターリーブされる */
    ```

    **方法2 — MPOL_MF_MOVEを指定したmbind:**
    ```c
    char *buf = malloc(size);
    memset(buf, 0, size);          /* すべてノード0上 */
    mbind(buf, size, MPOL_INTERLEAVE,
          &nodemask, max_node, MPOL_MF_MOVE);  /* 既存の page をマイグレーションする */
    ```

    **方法3 — 並列初期化:**
    ```c
    /* ワーカーを異なるノードに固定し、それぞれが自分の担当部分をタッチする */
    #pragma omp parallel for schedule(static)
    for (size_t i = 0; i < n; i++)
        buf[i] = 0;
    ```

    方法3は、単一スレッドが`main()`の前にすべてのコンストラクタを実行する、静的またはグローバルな初期化を持つC++プログラムに最も効果的である。

## ハードウェアプリフェッチャとアクセスパターンの効率 {#hardware-prefetchers-and-access-pattern-efficiency}

CPUのハードウェアプリフェッチャは、シーケンシャルなアクセスパターンを検出し、要求される前にメモリからキャッシュラインをプリフェッチする。これは帯域幅の利用率に直接影響する。プリフェッチャに優しいアクセスパターンは、ランダムなものより効果的に利用可能な帯域幅を飽和させる。

**アクセスパターンの効率（おおよその順位）**:

```
Sequential (stride 1):   ~100% of peak bandwidth
Strided (stride 2-8):    50–80% of peak bandwidth
Strided (stride 16+):    25–50% of peak bandwidth
Random (pointer chase):  5–15% of peak bandwidth
```

匿名メモリについては、カーネルはアクセスパターンを把握できない — ハードウェアプリフェッチャは独立に動作する。しかし**ファイルバックのpage**については、カーネルのリードアヘッド機構がソフトウェア上の類似物を提供する。シーケンシャルなファイル読み出しを検出し、pageがフォールトインされる前にページキャッシュに取り込むため、投機的に`page_cache_ra_unbounded()`を呼び出す。これはファイルに対するストリーミング分析には効果的だが、匿名のヒープ割り当てには効かない。

### 行優先と列優先 {#row-major-vs-column-major}

行列のコードは典型的な例である。行優先のC配列`A[rows][cols]`の場合は次のとおり。

```c
/* 帯域幅に優しい: シーケンシャルアクセスで、プリフェッチャがうまく働く */
for (int i = 0; i < rows; i++)
    for (int j = 0; j < cols; j++)
        sum += A[i][j];

/* 帯域幅に優しくない: 列優先のストライド = cols * sizeof(float) */
for (int j = 0; j < cols; j++)
    for (int i = 0; i < rows; i++)
        sum += A[i][j];
```

10,000×10,000のfloat配列に対する列優先のループは、アクセス間のストライドが40,000バイトになる。これはプリフェッチャを無力化し、典型的なサーバーでは実効帯域幅を5〜10分の1に低下させる。

!!! tip "帯域幅律速の行列ワークロードには"
    `MPOL_INTERLEAVE`を、行優先のアクセスパターンおよび2MBのhuge pageと組み合わせる。各要素は異なるボトルネックに対処する。インターリーブは総帯域幅を増やし、行優先アクセスはプリフェッチを可能にし、huge pageは大きな配列でのTLBオーバーヘッドを減らす。

## Transparent Huge Pageと帯域幅 {#transparent-huge-pages-and-bandwidth}

[Transparent Huge Page](thp.md)は、メモリを4KBではなく2MBのブロックでマップすることでTLBの逼迫を減らす。TLBエントリが少なければ、CPUのメモリアクセス能力のより多くがページテーブルウォークではなく実際のデータ転送に回る。

大規模な帯域幅律速のワークロードでは、`THP + MPOL_INTERLEAVE`の組み合わせが2つの別々のボトルネックに対処する。

| 仕組み | 解決するもの |
|-----------|---------------|
| `MPOL_INTERLEAVE` | 単一ノードの帯域幅の上限 |
| THP（2MBのpage） | 大きなワーキングセットでのTLBスラッシング |

該当する領域でTHPを有効にする。

```c
void *buf = mmap(NULL, size, PROT_READ | PROT_WRITE,
                 MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
madvise(buf, size, MADV_HUGEPAGE);     /* THP を要求する */

unsigned long nodemask = (1UL << nr_nodes) - 1;
set_mempolicy(MPOL_INTERLEAVE, &nodemask, nr_nodes + 1);

memset(buf, 0, size);  /* 最初のタッチ: インターリーブされた 2MB の huge page */
```

注: THPが有効な場合、khugepagedは隣接する4KBのpageを非同期に2MBのpageへ統合（collapse）することがある。結果のhuge pageは、その構成要素である4KBのpageの大半を保持していたノードに配置される。最初のタッチの時点でインターリーブが正しく行われていれば、これは問題にならないはずである。NUMAバランシングとTHPの相互作用については[THP](thp.md)を参照。

## NUMAバランシングと意図的なインターリーブ {#numa-balancing-and-deliberate-interleaving}

NUMA自動バランシング（`/proc/sys/kernel/numa_balancing`）は、定期的にpageのマッピングを外してアクセスフォールトを捕捉し、pageを最も多くアクセスするノードへ向けてマイグレーションすることで機能する。ほとんどのワークロードでは有益である。意図的にインターリーブしたワークロードでは、スレッドの大半が動いているノードへすべてのpageをマイグレーションすることで、チューニングを黙って台無しにしうる。

```bash
# 現在の状態を確認する
cat /proc/sys/kernel/numa_balancing

# 意図的にインターリーブしたワークロードでは無効化する
echo 0 > /proc/sys/kernel/numa_balancing
```

!!! tip "NUMAバランシングを無効化すべきとき"
    次の場合はNUMAバランシングを無効化する。

    - `MPOL_INTERLEAVE`を明示的に設定しており、その分布を維持したい
    - ワークロードがすべてのノードから均等にメモリにアクセスする（例: 各ワーカースレッドが自身のシャードを処理する）
    - インターリーブを正しく設定した後で、予想外に高い`numa_miss`の数が見られる

    次の場合はNUMAバランシングを有効のままにする。

    - 明示的なポリシーを設定していない
    - ワークロードのアクセスパターンが不規則、または時間とともに変化する
    - 初期配置の悪さをカーネルに自動的に修正してもらう必要がある

また、**カーネルの割り当てはユーザーのmempolicyの対象外である**点にも注意。`set_mempolicy()`と`mbind()`はユーザー空間の仮想メモリ領域にのみ影響する。カーネルのデータ構造（slabオブジェクト、ページテーブル、ネットワークバッファ）は、カーネル自身のNUMA対応アロケータによって割り当てられ、ユーザーポリシーによってインターリーブされることはない。

## メモリ帯域幅の計測 {#measuring-memory-bandwidth}

### ベンチマークツール {#benchmark-tools}

**`stream`** — 持続可能なメモリ帯域幅を計測するための標準ツール:

```bash
# stream ベンチマークをビルドする
wget https://www.cs.virginia.edu/stream/FTP/Code/stream.c
gcc -O3 -march=native -fopenmp stream.c -o stream
./stream

# 出力例:
# Function    Best Rate MB/s  Avg time     Min time     Max time
# Copy:          145234.4     0.011004     0.010983     0.011031
# Scale:         140123.8     0.011388     0.011419     0.011375
# Add:           148902.1     0.016134     0.016117     0.016152
# Triad:         149015.3     0.016100     0.016105     0.016093

# NUMA ノードごとの帯域幅を計測する
numactl --membind=0 ./stream    # ノード0のみ
numactl --membind=1 ./stream    # ノード1のみ
numactl --interleave=all ./stream  # インターリーブ
```

**`mbw`** — より簡単な帯域幅計測:

```bash
mbw 1024    # 1GB でテストする
```

### トポロジの調査 {#topology-inspection}

```bash
# NUMA ハードウェアのトポロジを調べる
numactl --hardware

# 出力例:
# available: 2 nodes (0-1)
# node 0 cpus: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15
# node 0 size: 129020 MB
# node 0 free: 121432 MB
# node 1 cpus: 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31
# node 1 size: 131072 MB
# node 1 free: 122804 MB
# node distances:
# node   0   1
#   0:  10  21
#   1:  21  10

# プロセスごとの NUMA 分布
numastat -m -p <pid>

# 出力例:
#                           Per-node process memory usage (MB)
#                                            Node 0   Node 1    Total
#                           --------------- -------- -------- --------
# myapp (pid 12345)            Private           987      991     1978
```

### ハードウェアパフォーマンスカウンタ {#hardware-performance-counters}

```bash
# LLC ミスはキャッシュ階層から出ていくメモリトラフィックを示す
perf stat -e LLC-load-misses,LLC-store-misses -p <pid>

# NUMA 固有のイベント（Intel、サポートされている場合）
perf stat -e offcore_response.all_requests.l3_miss.any_snoop \
          -p <pid>
```

### /proc/vmstatのNUMAカウンタ {#procvmstat-numa-counters}

```bash
cat /proc/vmstat | grep numa
```

主要なカウンタ（[`mm/vmstat.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmstat.c)で確認済み）は次のとおり。

| カウンタ | 意味 | 値が高いことの意味 |
|---------|---------|------------------------|
| `numa_hit` | 優先ノードで満たされた割り当て | 良好: メモリが意図した場所に配置されている |
| `numa_miss` | 優先ノード以外にフォールバックした割り当て | ミス率が高い: ノードの逼迫、またはインターリーブが未設定 |
| `numa_foreign` | このノード向けだったが他所に配置されたpage | ノード間の逼迫 |
| `numa_interleave` | `MPOL_INTERLEAVE`によって配置された割り当て | インターリーブポリシーが有効であることを確認できる |
| `numa_local` | 実行中のスレッドと同じノード上の割り当て | 高い: レイテンシ律速のワークロードには良好 |
| `numa_other` | 実行中のスレッドとは異なるノード上の割り当て | 高い: リモート割り当て（インターリーブでは意図的な場合もある） |

!!! tip "カウンタの読み方"
    健全にインターリーブされたワークロードでは、`numa_interleave`が高く、`numastat -m`での`N0`と`N1`の数がほぼ等しくなる。`numa_interleave`が低いのに`numa_miss`が高い場合、ポリシーを設定したにもかかわらず、ファーストタッチ配置がpageを1つのノードに集中させていることを示唆する。

## CXLメモリと帯域幅の階層化 {#cxl-memory-and-bandwidth-tiering}

CXL（Compute Express Link）メモリは、Linuxからは、CPUが接続されていない追加のNUMAノードとして見える。PCIe Gen 5 x16のCXLリンクの帯域幅はローカルDRAMより低い（ローカルDDR5 ≈ 150+ GB/sに対し、PCIe Gen 5 x16上のCXL ≈ 双方向64 GB/s）が、NVMeよりは高い。CXLは、高速なアクセスが必要だがローカルDRAMの完全な帯域幅までは必要としないコールドデータに適している。

帯域幅律速のワークロードでは、CXLノードは一般に`MPOL_INTERLEAVE`の対象として適さない — ホットデータを低帯域幅の階層に分散すると総スループットが低下する。CXLは、カーネルの自動メモリ階層化（DRAMからCXLへのコールドpageの降格）を通じて使う方がよい。

カーネルの階層化フレームワークと、HMAT/CDATの性能データに基づいてCXLノードがどのように階層に割り当てられるかについては、[CXLメモリ階層化](cxl-memory-tiering.md)を参照。

## 判断フローチャート {#decision-flowchart}

```
Is your workload memory bandwidth-bound?
  |
  No → Focus on MPOL_BIND / latency; bandwidth is not the issue
  |
  Yes
  |
  ├── Single NUMA node?
  │     Yes → Bandwidth ceiling is fixed; consider CXL or upgrade
  │     No ↓
  |
  ├── Are pages landing where you expect?
  │     Check: numastat -m, /proc/vmstat numa_interleave
  │     No → First-touch problem → see "First-touch" section above
  │     Yes ↓
  |
  ├── Is NUMA balancing migrating pages away from interleave?
  │     Check: watch -n1 'cat /proc/vmstat | grep numa'
  │     Yes → echo 0 > /proc/sys/kernel/numa_balancing
  │     Already off ↓
  |
  └── Are access patterns sequential?
        No → Optimize data layout (row-major, structure of arrays)
        Yes → Add MADV_HUGEPAGE for TLB headroom
```

## 主要なソースファイル {#key-source-files}

| ファイル | 説明 |
|------|-------------|
| [`mm/mempolicy.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mempolicy.c) | `MPOL_INTERLEAVE`と`MPOL_WEIGHTED_INTERLEAVE`の実装。`interleave_nodes()`、`interleave_nid()` |
| [`include/linux/mempolicy.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mempolicy.h) | `struct mempolicy`、ポリシーモードの定数 |
| [`include/uapi/linux/mempolicy.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/mempolicy.h) | ユーザー空間から見えるポリシー定数（`MPOL_INTERLEAVE`、`MPOL_WEIGHTED_INTERLEAVE`など） |
| [`mm/vmstat.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmstat.c) | `numa_hit`、`numa_miss`、`numa_interleave`のカウンタ名 |
| [`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) | `NUMA_HIT`/`NUMA_MISS`カウンタがインクリメントされる場所 |
| [`arch/x86/mm/numa.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/numa.c) | ACPI SRAT/SLITからのx86 NUMAトポロジの検出 |

## 参考資料 {#references}

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/admin-guide/mm/numa_memory_policy.rst`](https://docs.kernel.org/admin-guide/mm/numa_memory_policy.html)
- [`Documentation/mm/numa.rst`](https://docs.kernel.org/mm/numa.html) — NUMAバランシングの実装の詳細

### メーリングリストでの議論 {#mailing-list-discussions}

- [MPOL_WEIGHTED_INTERLEAVE introduction](https://lore.kernel.org/linux-mm/20231117-weighted-interleave-v6-0-d1af872c78aa@quicinc.com/) — Gregory Price（Qualcomm）、2023年11月
- [NUMA balancing and THP interaction](https://lore.kernel.org/linux-mm/20130926103049.GM19413@suse.de/) — Mel GormanによるTHPのNUMAマイグレーションの設計

### 外部ベンチマーク {#external-benchmarks}

- [STREAM benchmark](https://www.cs.virginia.edu/stream/) — McCalpin, J.D.、メモリ帯域幅計測の標準

### 関連項目 {#related}

- [NUMAメモリ管理](numa.md) — ポリシーAPI、自動バランシング、監視
- [Transparent Huge Pages](thp.md) — THPの設定、khugepaged、TLBの逼迫
- [CXLメモリ階層化](cxl-memory-tiering.md) — 異種メモリの階層
- [データベースのチューニング](tuning-databases.md) — huge page、THP、レイテンシに敏感なワークロード

## 参考文献 {#further-reading}

- [Kernel docs: NUMA memory policy](https://docs.kernel.org/admin-guide/mm/numa_memory_policy.html) — `set_mempolicy()`、`mbind()`、および`MPOL_WEIGHTED_INTERLEAVE`を含む全ポリシーモードの完全なリファレンス
- [`Documentation/admin-guide/mm/numa_memory_policy.rst`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/admin-guide/mm/numa_memory_policy.rst) — NUMAメモリポリシー管理ガイドのカーネルソース
- [`mm/mempolicy.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mempolicy.c) — `MPOL_INTERLEAVE`と`MPOL_WEIGHTED_INTERLEAVE`の実装。`interleave_nodes()`と`interleave_nid()`
- [LWN: NUMA in a hurry](https://lwn.net/Articles/486858/) — NUMAトポロジとファーストタッチ割り当ての落とし穴への実践的な入門（2012年）
- [LWN: Weighted interleave memory policy](https://lwn.net/Articles/948552/) — 異種のメモリ容量に対する`MPOL_WEIGHTED_INTERLEAVE`の根拠と設計（2023年）
- [STREAM benchmark](https://www.cs.virginia.edu/stream/) — McCalpinによる、持続可能なメモリ帯域幅計測の標準。あらゆる帯域幅最適化の較正に不可欠
- [numa.md](numa.md) — NUMAバランシングの実装、`node_distance`、フォールバックのゾーンリスト、mempolicy APIの全体
- [cxl-memory-tiering.md](cxl-memory-tiering.md) — CXLノードがインターリーブの対象に向かない理由と、階層フレームワークが帯域幅に適した配置をどのように割り当てるか
- [thp.md](thp.md) — Transparent Huge Pageと、NUMAバランシングおよび帯域幅律速ワークロードとの相互作用
