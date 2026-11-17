# NUMA距離とソケット間レイテンシ {#numa-distance-and-inter-socket-latency}

> カーネルはノードをまたぐメモリアクセスのコストをどう数値化しているのか — そしてその数値がナノ秒ではない理由

## 主要なソースファイル {#key-source-files}

| ファイル | 説明 |
|------|-------------|
| [`include/linux/topology.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/topology.h) | `LOCAL_DISTANCE`、`REMOTE_DISTANCE`、`RECLAIM_DISTANCE`、デフォルトの `node_distance()` マクロ |
| [`arch/x86/include/asm/topology.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/topology.h) | x86 でのオーバーライド: `node_distance(a, b)` → `__node_distance(a, b)` |
| [`mm/numa_memblks.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/numa_memblks.c) | フラットな行列 `numa_distance[]`、`numa_set_distance()`、`__node_distance()` |
| [`drivers/base/node.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/base/node.c) | `node_read_distance()` — 各ノード配下の sysfs `distance` 属性 |
| [`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) | `build_zonelists()`、`find_next_best_node()`、`node_reclaim_distance` |
| [`kernel/sched/fair.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/fair.c) | `task_numa_migrate()`、`should_numa_migrate_memory()` |
| [`arch/x86/mm/numa.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/numa.c) | x86 の NUMA 初期化、ACPI/AMD/OF の初期化パス |

---

## NUMA距離の定義 {#numa-distance-defined}

カーネルは NUMA インターコネクトをまたぐコストを、ナノ秒単位のレイテンシではなく **無次元の整数** で表現する。スケールを決める2つの定数が `include/linux/topology.h` で定義されている:

```c
/* include/linux/topology.h — ACPI 2.0 SLIT の距離定義に準拠 */
#define LOCAL_DISTANCE    10
#define REMOTE_DISTANCE   20
#define DISTANCE_BITS      8   /* 距離は u8、範囲は 0–255 */
```

ハードウェア由来のテーブルを提供しないプラットフォームで使われる汎用のフォールバックマクロは次のとおり:

```c
/* include/linux/topology.h */
#ifndef node_distance
#define node_distance(from, to) \
    ((from) == (to) ? LOCAL_DISTANCE : REMOTE_DISTANCE)
#endif
```

x86 では、この汎用マクロは実際のテーブル参照でオーバーライドされる:

```c
/* arch/x86/include/asm/topology.h */
extern int __node_distance(int, int);
#define node_distance(a, b) __node_distance(a, b)
```

`__node_distance()` は `numa_distance` というフラットな `u8` 配列を参照する。この配列はブート時にファームウェアのデータから割り当てられ、値が設定される:

```c
/* mm/numa_memblks.c */
int __node_distance(int from, int to)
{
    if (from >= numa_distance_cnt || to >= numa_distance_cnt)
        return from == to ? LOCAL_DISTANCE : REMOTE_DISTANCE;
    return numa_distance[from * numa_distance_cnt + to];
}
EXPORT_SYMBOL(__node_distance);
```

### 距離は比率であってナノ秒ではない {#distances-are-ratios-not-nanoseconds}

`LOCAL_DISTANCE = 10` は10ナノ秒を意味する **わけではない**。値は比例的なもので、距離20は「ローカルアクセスのおよそ2倍のコスト」を意味する。ACPI SLIT 仕様はこれらを相対的な性能比として定義している。実際の2ソケット Intel システムでは、ローカル DRAM が約80 ns、リモート DRAM が約130 ns ということもあるが、それでも距離値はそれぞれ10と20になる。カーネルにとって重要なのは絶対的な測定値ではなく、1:2という比率である。

メモリティアフレームワークが使う抽象距離モデル（ハードウェアの性能座標をティア配置に変換するもの）については、[CXL Memory and Kernel Memory Tiering](cxl-memory-tiering.md) を参照。

---

## 距離はどこから来るのか {#where-distances-come-from}

### ACPI SLIT テーブル {#acpi-slit-table}

ACPI を完全にサポートするシステムでは、距離は **System Locality Information Table（SLIT）** によって提供される。各エントリ `SLIT[i][j]` は、ノード `i` からノード `j` への相対レイテンシを表す。SLIT は、マルチソケットの x86 サーバー、ARM サーバー、そしてソケット間トポロジが単純でないほとんどのハードウェアにおける主要な情報源である。

SLIT は、CPU とメモリ範囲をノードに対応付ける SRAT（System Resource Affinity Table）と同じ ACPI ファームウェア基盤の一部である。カーネルが SRAT と SLIT をどう解析するかの詳しい解説は、今後公開予定の `numa-acpi-srat.md` を参照。

`arch/x86/mm/numa.c` の `x86_numa_init()` 関数は、次の初期化パスを順に試し、最初に成功したところで止まる:

```
x86_acpi_numa_init()    ← SRAT + SLIT
amd_numa_init()         ← AMD 固有の NUMA（古いプラットフォーム）
of_numa_init()          ← Device Tree（DT 経由の非 ACPI な ARM/RISC-V）
dummy_numa_init()       ← フォールバック: 単一ノード、全メモリをノード0に
```

### SLIT のないシステム {#systems-without-slit}

SLIT が存在しない場合、あるいは ACPI が無効化されている場合、`numa_alloc_distance()` が行列をデフォルト値で埋める:

```c
/* mm/numa_memblks.c */
for (i = 0; i < cnt; i++)
    for (j = 0; j < cnt; j++)
        numa_distance[i * cnt + j] = i == j ?
            LOCAL_DISTANCE : REMOTE_DISTANCE;
```

すべてのノードペアに距離10（自身）または20（他ノード）が割り当てられる。これは真に2ノードの UMA 相当システムでは正しいが、マルチホップのトポロジは反映されない。

### エミュレートされた NUMA と VM の NUMA {#emulated-and-vm-numa}

仮想化ハイパーバイザは、すべての距離が10のフラットな NUMA トポロジを公開することが多い。ゲスト OS は実際のハードウェアインターコネクトをまたがないからである。このため、距離に基づくカーネルの判断（zonelist の順序付け、NUMA バランシングのしきい値）は、すべてのメモリが等距離であるかのように振る舞う — そして VM では通常それが事実である。[よくある落とし穴](#common-pitfalls)のセクションを参照。

---

## 距離行列 {#the-distance-matrix}

### /sys/devices/system/node/nodeN/distance の読み方 {#reading-sysdevicessystemnodenodendistance}

カーネルは各ノードについて、距離行列の行全体を sysfs 経由で公開している。`drivers/base/node.c` の `node_read_distance()` 関数は、オンラインのすべてのノードを走査して、それぞれについて `node_distance(nid, i)` を出力する:

```bash
# 2ソケットシステム — 2ノード
$ cat /sys/devices/system/node/node0/distance
10 21
$ cat /sys/devices/system/node/node1/distance
21 10
```

ノード0の行: 自身への距離は10、ノード1への距離は21。ノード1の行はその鏡像になっている。（ちょうど20ではなく）21という値はよく見られる。ファームウェアは「1ホップで、REMOTE_DISTANCE の基準値をわずかに上回る」ことを示すために21を報告することが多い。

`numactl --hardware` は同じデータをより読みやすい行列形式で表示する:

```
node distances:
node   0   1
  0:  10  21
  1:  21  10
```

### 4ソケットシステムの例 {#4-socket-system-example}

ノード0/1が1つの NUMA ドメインを共有し、ノード2/3が別のドメインを共有し、ドメイン間のトラフィックが2つのインターコネクトホップを経由する4ソケットサーバーでは、次のように表示されることがある:

```
node distances:
node   0   1   2   3
  0:  10  11  21  22
  1:  11  10  22  21
  2:  21  22  10  11
  3:  22  21  11  10
```

- 距離10–11: ローカルソケット（同一ダイ内、または隣接するダイ間の1ホップ）
- 距離21–22: リモートソケット（ソケット間ファブリックをまたぐ2ホップ）

行列の読み方: 行が *アクセス元* ノード、列が *アクセス先* ノードである。エントリ `[0][2] = 21` は、ノード0上の CPU がノード2上のメモリにアクセスする相対コストが21であることを意味する。

### 非対称な距離 {#asymmetric-distances}

SLIT 仕様は非対称なエントリを許容している。`SLIT[i][j]` が `SLIT[j][i]` と等しい必要はない。これは、ある方向の読み出しコストが逆方向と異なる、特定の NUMA-over-fabric 構成（例: コヒーレントなアクセラレータインターコネクト）で起こりうる。Linux は対称性を強制せずに行列全体を保存して使用するため、非対称なファームウェアテーブルは sysfs にも `node_distance()` の呼び出しにも忠実に反映される。

### 内部での格納方法 {#internal-storage}

行列は `numa_distance[from * numa_distance_cnt + to]` としてインデックス付けされるフラットな `u8` 配列として格納され、ブートの初期段階で `memblock` を通じて割り当てられる。配列の値は `numa_set_distance()` によって設定される。この関数は有効な `u8` 範囲外の値を拒否し、対角要素が常に `LOCAL_DISTANCE` となるよう強制する。

---

## カーネルは距離をどう使うか {#how-the-kernel-uses-distances}

### zonelist の順序付け（メモリアロケータのフォールバック） {#zonelist-ordering-memory-allocator-fallback}

`mm/page_alloc.c` の `build_zonelists()` は **フォールバック zonelist** を構築する。これは、優先ノードが要求を満たせないときにアロケータが試すメモリゾーンの順序付きリストである。ノードはローカルノードからの距離でソートされる:

```c
/* mm/page_alloc.c — build_zonelists() */
while ((node = find_next_best_node(local_node, &used_mask)) >= 0) {
    if (node_distance(local_node, node) !=
        node_distance(local_node, prev_node))
        node_load[node] += 1;    /* 新しい距離ティア: ラウンドロビンをリセット */
    node_order[nr_nodes++] = node;
    prev_node = node;
}
build_zonelists_in_node_order(pgdat, node_order, nr_nodes);
```

`find_next_best_node()` は `node_distance()` を第一ソートキーとして使うため、距離10のノードが最初に来て、次に距離21、次に距離30、という順になる。アロケータは、より近いノードのメモリをすべて使い切ってから、より遠いノードにフォールバックする。

### ノード回収のしきい値 {#node-reclaim-threshold}

`RECLAIM_DISTANCE`（デフォルト30）は、`node_reclaim()` を近くのノードに制限するタイミングを制御する:

```c
/* include/linux/topology.h */
#ifndef RECLAIM_DISTANCE
#define RECLAIM_DISTANCE 30
#endif

extern int __read_mostly node_reclaim_distance;
```

`node_reclaim_mode` が有効で、候補ノードへの距離が `node_reclaim_distance` を超える場合、カーネルはそのノードでメモリ回収を試みない。高レイテンシのリンクをまたいで回収と移動を行うより、リモートで割り当てるほうが安上がりだからである。

AMD EPYC システムは `node_reclaim_distance` を上方向にオーバーライドする。2ホップの距離（32）でも十分に高速なので、ノードをまたぐ回収が性能を損なうのではなく向上させるためである。これは `topology.h` 内のカーネルコメントに記載されている。

### スケジューラ: NUMA ドメインの構築 {#scheduler-numa-domain-construction}

スケジューラは NUMA 距離を使って `sched_domain` 階層を構築する。互いに同じ距離にあるノードが NUMA スケジューリングドメイン（`SD_NUMA`）を形成する。`kernel/sched/fair.c` の `task_numa_migrate()` は、マイグレーションするかどうかを判断する際に、タスクの現在のノードと優先ノードの間の距離を問い合わせる:

```c
/* kernel/sched/fair.c — task_numa_migrate() */
env.dst_nid = p->numa_preferred_nid;
dist = env.dist = node_distance(env.src_nid, env.dst_nid);
taskweight = task_weight(p, env.src_nid, dist);
...
taskimp = task_weight(p, env.dst_nid, dist) - taskweight;
```

`dist` は重みの計算に渡され、マイグレーションの利益がタスクの移動距離に応じてスケーリングされる。距離の大きいリンクをまたぐ移動を正当化するには、より大きな不均衡が必要になる。

### NUMA バランシング: マイグレーションの判断 {#numa-balancing-migration-decisions}

`kernel/sched/fair.c` の `should_numa_migrate_memory()` は、リモートページでのページフォールトがマイグレーションを引き起こすべきかどうかを判断する。低速メモリティアにあるページについては、距離だけでなくアクセス頻度（そのページが最近いつフォールトしたか）に基づいて判断する。従来の NUMA バランシングでは、mempolicy レイヤの `mm/mempolicy.c` にある `numa_nearest_node()` 関数が `node_distance()` を使ってフォールトした CPU により近いノードを探し、`mm/memory.c` の `numa_migrate_check()` がマイグレーションを準備する:

```c
/* mm/mempolicy.c — 最も近いノードの探索 */
dist = node_distance(node, n);
```

ページは、メモリの空き状況と cpuset の制約を満たす範囲で、フォールトした CPU からの距離が最小のノードへと向けられる。

---

## 実際のレイテンシの測定 {#measuring-actual-latency}

!!! warning "距離値は実際のハードウェアを反映していないことがある"
    ファームウェアの SLIT エントリは BIOS/ファームウェアの作成者が設定する。プラットフォームによっては、それが近似値、プレースホルダー、あるいは単純に誤った値であることもある。性能に関する結論を出す前に、必ず直接のレイテンシ測定で距離値を検証すること。

### Intel MLC（Memory Latency Checker） {#intel-mlc-memory-latency-checker}

Intel MLC は、負荷時および無負荷時の NUMA メモリレイテンシを測定するための業界標準ツールである。設定可能なアクセスパターンで各ノードペアを計測する:

```bash
# 全ノードペアにわたる無負荷時レイテンシの行列を測定
./mlc --latency_matrix

# 負荷時レイテンシを測定（トラフィックインジェクタあり）
./mlc --loaded_latency

# 帯域幅の行列
./mlc --bandwidth_matrix
```

MLC はレイテンシをナノ秒で報告する。リモート対ローカルのレイテンシ比をカーネルの距離比と比較して、SLIT が正しく較正されているかを評価する。

### numactl + stream {#numactl-stream}

MLC を使わずに手早く帯域幅を測定するには:

```bash
# ノード0の CPU がノード1のメモリから読む
numactl --cpunodebind=0 --membind=1 ./stream

# ノード0の CPU がローカルメモリから読む
numactl --cpunodebind=0 --membind=0 ./stream
```

帯域幅の比は、リモートアクセスがスループットをどれだけ低下させるかを示す。SLIT が報告する距離比が2:1の2ソケットシステムでは、リモートアクセスの帯域幅低下は通常50%ではなく30–50%になる。インターコネクトの帯域幅は非対称であることが多く、ボトルネックが DRAM 帯域幅ではなく QPI/UPI リンクに移るからである。

### perf mem {#perf-mem}

`perf mem` は、ハードウェアの性能監視ユニットを使って、キャッシュミスや DRAM ミスを特定の NUMA ノードに帰属させることができる:

```bash
# /proc/sys/kernel/perf_event_paranoid <= 1（または CAP_PERFMON）が必要
perf mem record -a -- sleep 5
perf mem report --sort=mem,sym
```

レポートには各メモリアクセスサンプルのソースノード（local/remote）が表示されるため、ワークロードを変更することなく、局所性の悪いホットスポットを特定できる。

### 負荷時レイテンシと無負荷時レイテンシ {#loaded-vs-idle-latency}

シナリオによって重要となる測定値は2つある:

- **無負荷時レイテンシ**（競合するトラフィックなし）: ハードウェアの生のラウンドトリップ時間を表す。SLIT の比率を検証するのに役立つ。
- **負荷時レイテンシ**（帯域幅インジェクタあり）: 実際のワークロード条件下でのレイテンシを表す。負荷時にはインターコネクトの競合により、リモートレイテンシが無負荷時の測定値から想定されるよりも大幅に高くなることがある。

帯域幅律速のワークロード（データベース、HPC）では、負荷時レイテンシと帯域幅の飽和が主な制約となる。レイテンシに敏感なワークロード（インメモリキャッシュ、トレーディングシステム）では、無負荷時レイテンシと軽負荷時のテールレイテンシのほうが重要である。

---

## マルチホップ NUMA {#multi-hop-numa}

3ソケット以上のシステムでは、距離行列はソケット間ファブリックをまたぐ **ホップ数** をエンコードしている。リング状または完全結合のメッシュとして構成された4ソケットシステムでは、リモート距離の種類は最大で2つになる:

```
distance 20-22: one hop  (directly connected socket pair)
distance 30-32: two hops (socket pair connected only through a third)
```

### マルチホップトポロジの識別 {#identifying-multi-hop-topology}

距離行列を読み、非対角要素に3種類以上の異なる値があるかを確認する:

```bash
numactl --hardware | grep -A 10 "node distances"
```

3種類以上の異なる距離値がある場合、そのシステムにはマルチホップの経路がある。最も長い経路が、最悪ケースのリモートアクセスコストを示す。

### スケジューラとアロケータへの影響 {#scheduler-and-allocator-impact}

`build_zonelists()` はノードを **距離ティア** にグループ化する。ローカルノードから同じ距離にあるすべてのノードが1つのティアを形成し、アロケータは1つのティアを使い切ってから次のティアに進む。距離が {10, 21, 32} の4ソケットシステムでは、ノード0のフォールバック順序は次のようになる:

1. ノード0（距離10 — ローカル）
2. 直接接続されたノード（距離21） — ティア内でラウンドロビン
3. 2ホップ先のノード（距離32） — ティア1と2を使い切った場合のみ

スケジューラの `SD_NUMA` ドメインも同様に、1ホップ目と2ホップ目を区別して扱う。近いティア内でのマイグレーションは、2ホップの境界をまたぐマイグレーションよりもペナルティが小さい。

### AMD EPYC のマルチダイトポロジ {#amd-epyc-multi-die-topology}

AMD EPYC プロセッサは、1ソケットあたり複数の NUMA ノード（ダイごと、またはクアドラントごとに1つ）を公開する。2ソケットの EPYC システムは4個または8個の NUMA ノードを提示することがあり、ソケット内の距離は12、ソケット間の距離は32となる。EPYC システムの BIOS が提供する SLIT は概ね正確である — AMD は検証済みの ACPI テーブルを出荷している。EPYC 向けの `node_reclaim_distance` のオーバーライド（前述）は、32でも回収の恩恵を受けられるほど十分に高速であることを考慮したものである。

---

## よくある落とし穴 {#common-pitfalls}

!!! warning "すべての距離が10: 距離はおそらく誤っている"
    `numactl --hardware` がすべてのノード距離を10と表示する場合、ファームウェアが実際の SLIT を提供していないか、SLIT がすべてローカルの値で合成されている。これは次の2つのよくあるシナリオで起こる:

    1. **仮想化された NUMA** — ハイパーバイザは（例えば vCPU を分散させるために）NUMA ノードを公開するが、ゲストのメモリはすべてホストの同じ NUMA ドメイン上にある。実際のリモートコストが存在しないため、ゲストの距離はすべて10になる。

    2. **壊れた BIOS SLIT** — 一部のファームウェアは、実際のマルチソケットハードウェア上でも誤ってすべての距離を10と報告する。カーネルはすべてのノードを等距離として扱い、ローカルメモリからの割り当てを優先しなくなる。NUMA ワークロードの性能が、明らかな理由なく悪化することがある。

    壊れた BIOS のケースを検出するには: `lscpu` が複数のソケットを示しているのに `numactl -H` がすべての距離を10と示している場合、SLIT はほぼ確実に誤っている。BIOS のアップデートを確認するか、`acpidump | acpixtract -sLIT` を使って生のテーブルを調べる。

!!! warning "実際の NUMA 効果のない VM の NUMA トポロジ"
    `numaNodes=2` で構成された VM は距離行列を持つが、ハイパーバイザがゲストメモリを単一のホスト NUMA ノードから割り当てている場合、どちらのゲストノードへのアクセスも同じ物理メモリに行く。カーネルの NUMA バランシングと zonelist の判断はゲストの距離テーブルに基づいて行われるため、性能上の利益のないマイグレーションを試み、オーバーヘッドを増やすこともある。実際のハードウェアトポロジに対応していない VM の中では、NUMA バランシングの無効化を検討すること（`echo 0 > /proc/sys/kernel/numa_balancing`）。

!!! warning "距離値は u8 — 最大255"
    `DISTANCE_BITS 8` は、距離が符号なし8ビット値として格納されることを意味する。`numa_set_distance()` は `u8` に収まらない値を拒否する。255を超える距離を提供するファームウェア（珍しいが、深く階層化されたファブリックトポロジでは起こりうる）は、クランプされるか拒否され、`__node_distance()` は `REMOTE_DISTANCE` にフォールバックする。

!!! warning "numactl はメモリのみのノードを表示しないことがある"
    CPU を持たないノード（例: CXL メモリノード、HMAT の generic initiator ノード）は、numactl のバージョンやカーネルの設定によっては `numactl --hardware` の出力に現れないことがある。`ls /sys/devices/system/node/` を使ってメモリのみのノードも含めたすべてのノードを列挙し、各ノードの `distance` ファイルを直接読むこと。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [arch/x86/mm/numa.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/numa.c) — `x86_numa_init()` と、距離行列に値を設定する ACPI/AMD/OF/dummy の初期化チェーン
- [mm/numa_memblks.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/numa_memblks.c) — `__node_distance()`、`numa_set_distance()`、およびフラット配列 `numa_distance[]` による格納
- [drivers/base/node.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/base/node.c) — `node_read_distance()`: `/sys/devices/system/node/nodeN/distance` を公開する sysfs ハンドラ
- [mm/page_alloc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) — `build_zonelists()` と `find_next_best_node()`: 距離がフォールバック zonelist の順序をどう決めるか
- [kernel/sched/fair.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/fair.c) — `task_numa_migrate()` と `should_numa_migrate_memory()`: スケジューラによる `node_distance()` の利用

### LWN の記事 {#lwn-articles}

- [NUMA distance and scheduler topology](https://lwn.net/Articles/392116/) — スケジューラが距離に基づいてノードをスケジューリングドメインにグループ化する方法
- [Heterogeneous memory management and NUMA](https://lwn.net/Articles/756022/) — 距離特性の異なるメモリティアが最近のカーネルでどう扱われるか

### 関連ドキュメント {#related-docs}

- [NUMA Topology Discovery: ACPI SRAT and SLIT](numa-acpi-srat.md) — ファームウェアの SRAT/SLIT テーブルがどのように解析され、`numa_distance[]` に値が設定されるか
- [NUMA Zonelist Construction and Fallback Ordering](numa-zonelist.md) — `node_distance()` がページアロケータのフォールバックノード順序をどう決めるか
- [NUMA Effects on Memory Reclaim](numa-reclaim.md) — `RECLAIM_DISTANCE` が距離値を使ってローカル回収をどう制御するか
- [CXL Memory and Kernel Memory Tiering](cxl-memory-tiering.md) — DRAM 以外のメモリティアが距離モデルをどう拡張するか
