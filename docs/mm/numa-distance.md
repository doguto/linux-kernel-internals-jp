# NUMA距離とソケット間レイテンシ {#numa-distance-and-inter-socket-latency}

> カーネルはノードをまたぐメモリアクセスのコストをどう数値化しているのか — そしてその数値がナノ秒ではない理由

## 主要なソースファイル {#key-source-files}

| ファイル | 説明 |
|------|-------------|
| [`include/linux/topology.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/topology.h) | `LOCAL_DISTANCE`、`REMOTE_DISTANCE`、`RECLAIM_DISTANCE`、デフォルトの `node_distance()` マクロ |
| [`arch/x86/include/asm/topology.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/topology.h) | x86でのオーバーライド: `node_distance(a, b)` → `__node_distance(a, b)` |
| [`mm/numa_memblks.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/numa_memblks.c) | フラットな行列 `numa_distance[]`、`numa_set_distance()`、`__node_distance()` |
| [`drivers/base/node.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/base/node.c) | `node_read_distance()` — 各ノード配下のsysfs `distance` 属性 |
| [`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) | `build_zonelists()`、`find_next_best_node()`、`node_reclaim_distance` |
| [`kernel/sched/fair.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/fair.c) | `task_numa_migrate()`、`should_numa_migrate_memory()` |
| [`arch/x86/mm/numa.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/numa.c) | x86のNUMA初期化、ACPI/AMD/OFの初期化パス |

---

## NUMA距離の定義 {#numa-distance-defined}

カーネルはNUMAインターコネクトをまたぐコストを、ナノ秒単位のレイテンシではなく **無次元の整数** で表現する。スケールを決める2つの定数が `include/linux/topology.h` で定義されている:

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

x86では、この汎用マクロは実際のテーブル参照でオーバーライドされる:

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

`LOCAL_DISTANCE = 10` は10ナノ秒を意味する **わけではない**。値は比例的なもので、距離20は「ローカルアクセスのおよそ2倍のコスト」を意味する。ACPI SLIT仕様はこれらを相対的な性能比として定義している。実際の2ソケットIntelシステムでは、ローカルDRAMが約80 ns、リモートDRAMが約130 nsということもあるが、それでも距離値はそれぞれ10と20になる。カーネルにとって重要なのは絶対的な測定値ではなく、1:2という比率である。

メモリティアフレームワークが使う抽象距離モデル（ハードウェアの性能座標をティア配置に変換するもの）については、[CXL Memory and Kernel Memory Tiering](cxl-memory-tiering.md) を参照。

---

## 距離はどこから来るのか {#where-distances-come-from}

### ACPI SLITテーブル {#acpi-slit-table}

ACPIを完全にサポートするシステムでは、距離は **System Locality Information Table（SLIT）** によって提供される。各エントリ `SLIT[i][j]` は、ノード `i` からノード `j` への相対レイテンシを表す。SLITは、マルチソケットのx86サーバー、ARMサーバー、そしてソケット間トポロジが単純でないほとんどのハードウェアにおける主要な情報源である。

SLITは、CPUとメモリ範囲をノードに対応付けるSRAT（System Resource Affinity Table）と同じACPIファームウェア基盤の一部である。カーネルがSRATとSLITをどう解析するかの詳しい解説は、今後公開予定の `numa-acpi-srat.md` を参照。

`arch/x86/mm/numa.c` の `x86_numa_init()` 関数は、次の初期化パスを順に試し、最初に成功したところで止まる:

```
x86_acpi_numa_init()    ← SRAT + SLIT
amd_numa_init()         ← AMD 固有の NUMA（古いプラットフォーム）
of_numa_init()          ← Device Tree（DT 経由の非 ACPI な ARM/RISC-V）
dummy_numa_init()       ← フォールバック: 単一ノード、全メモリをノード0に
```

### SLITのないシステム {#systems-without-slit}

SLITが存在しない場合、あるいはACPIが無効化されている場合、`numa_alloc_distance()` が行列をデフォルト値で埋める:

```c
/* mm/numa_memblks.c */
for (i = 0; i < cnt; i++)
    for (j = 0; j < cnt; j++)
        numa_distance[i * cnt + j] = i == j ?
            LOCAL_DISTANCE : REMOTE_DISTANCE;
```

すべてのノードペアに距離10（自身）または20（他ノード）が割り当てられる。これは真に2ノードのUMA相当システムでは正しいが、マルチホップのトポロジは反映されない。

### エミュレートされたNUMAとVMのNUMA {#emulated-and-vm-numa}

仮想化ハイパーバイザは、すべての距離が10のフラットなNUMAトポロジを公開することが多い。ゲストOSは実際のハードウェアインターコネクトをまたがないからである。このため、距離に基づくカーネルの判断（zonelistの順序付け、NUMAバランシングのしきい値）は、すべてのメモリが等距離であるかのように振る舞う — そしてVMでは通常それが事実である。[よくある落とし穴](#common-pitfalls)のセクションを参照。

---

## 距離行列 {#the-distance-matrix}

### /sys/devices/system/node/nodeN/distanceの読み方 {#reading-sysdevicessystemnodenodendistance}

カーネルは各ノードについて、距離行列の行全体をsysfs経由で公開している。`drivers/base/node.c` の `node_read_distance()` 関数は、オンラインのすべてのノードを走査して、それぞれについて `node_distance(nid, i)` を出力する:

```bash
# 2ソケットシステム — 2ノード
$ cat /sys/devices/system/node/node0/distance
10 21
$ cat /sys/devices/system/node/node1/distance
21 10
```

ノード0の行: 自身への距離は10、ノード1への距離は21。ノード1の行はその鏡像になっている。（ちょうど20ではなく）21という値はよく見られる。ファームウェアは「1ホップで、REMOTE_DISTANCEの基準値をわずかに上回る」ことを示すために21を報告することが多い。

`numactl --hardware` は同じデータをより読みやすい行列形式で表示する:

```
node distances:
node   0   1
  0:  10  21
  1:  21  10
```

### 4ソケットシステムの例 {#4-socket-system-example}

ノード0/1が1つのNUMAドメインを共有し、ノード2/3が別のドメインを共有し、ドメイン間のトラフィックが2つのインターコネクトホップを経由する4ソケットサーバーでは、次のように表示されることがある:

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

行列の読み方: 行が *アクセス元* ノード、列が *アクセス先* ノードである。エントリ `[0][2] = 21` は、ノード0上のCPUがノード2上のメモリにアクセスする相対コストが21であることを意味する。

### 非対称な距離 {#asymmetric-distances}

SLIT仕様は非対称なエントリを許容している。`SLIT[i][j]` が `SLIT[j][i]` と等しい必要はない。これは、ある方向の読み出しコストが逆方向と異なる、特定のNUMA-over-fabric構成（例: コヒーレントなアクセラレータインターコネクト）で起こりうる。Linuxは対称性を強制せずに行列全体を保存して使用するため、非対称なファームウェアテーブルはsysfsにも `node_distance()` の呼び出しにも忠実に反映される。

### 内部での格納方法 {#internal-storage}

行列は `numa_distance[from * numa_distance_cnt + to]` としてインデックス付けされるフラットな `u8` 配列として格納され、ブートの初期段階で `memblock` を通じて割り当てられる。配列の値は `numa_set_distance()` によって設定される。この関数は有効な `u8` 範囲外の値を拒否し、対角要素が常に `LOCAL_DISTANCE` となるよう強制する。

---

## カーネルは距離をどう使うか {#how-the-kernel-uses-distances}

### zonelistの順序付け（メモリアロケータのフォールバック） {#zonelist-ordering-memory-allocator-fallback}

`mm/page_alloc.c` の `build_zonelists()` は **フォールバックzonelist** を構築する。これは、優先ノードが要求を満たせないときにアロケータが試すメモリゾーンの順序付きリストである。ノードはローカルノードからの距離でソートされる:

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

AMD EPYCシステムは `node_reclaim_distance` を上方向にオーバーライドする。2ホップの距離（32）でも十分に高速なので、ノードをまたぐ回収が性能を損なうのではなく向上させるためである。これは `topology.h` 内のカーネルコメントに記載されている。

### スケジューラ: NUMAドメインの構築 {#scheduler-numa-domain-construction}

スケジューラはNUMA距離を使って `sched_domain` 階層を構築する。互いに同じ距離にあるノードがNUMAスケジューリングドメイン（`SD_NUMA`）を形成する。`kernel/sched/fair.c` の `task_numa_migrate()` は、マイグレーションするかどうかを判断する際に、タスクの現在のノードと優先ノードの間の距離を問い合わせる:

```c
/* kernel/sched/fair.c — task_numa_migrate() */
env.dst_nid = p->numa_preferred_nid;
dist = env.dist = node_distance(env.src_nid, env.dst_nid);
taskweight = task_weight(p, env.src_nid, dist);
...
taskimp = task_weight(p, env.dst_nid, dist) - taskweight;
```

`dist` は重みの計算に渡され、マイグレーションの利益がタスクの移動距離に応じてスケーリングされる。距離の大きいリンクをまたぐ移動を正当化するには、より大きな不均衡が必要になる。

### NUMAバランシング: マイグレーションの判断 {#numa-balancing-migration-decisions}

`kernel/sched/fair.c` の `should_numa_migrate_memory()` は、リモートページでのページフォールトがマイグレーションを引き起こすべきかどうかを判断する。低速メモリティアにあるページについては、距離だけでなくアクセス頻度（そのページが最近いつフォールトしたか）に基づいて判断する。従来のNUMAバランシングでは、mempolicyレイヤの `mm/mempolicy.c` にある `numa_nearest_node()` 関数が `node_distance()` を使ってフォールトしたCPUにより近いノードを探し、`mm/memory.c` の `numa_migrate_check()` がマイグレーションを準備する:

```c
/* mm/mempolicy.c — 最も近いノードの探索 */
dist = node_distance(node, n);
```

ページは、メモリの空き状況とcpusetの制約を満たす範囲で、フォールトしたCPUからの距離が最小のノードへと向けられる。

---

## 実際のレイテンシの測定 {#measuring-actual-latency}

!!! warning "距離値は実際のハードウェアを反映していないことがある"
    ファームウェアの SLIT エントリは BIOS/ファームウェアの作成者が設定する。プラットフォームによっては、それが近似値、プレースホルダー、あるいは単純に誤った値であることもある。性能に関する結論を出す前に、必ず直接のレイテンシ測定で距離値を検証すること。

### Intel MLC（Memory Latency Checker） {#intel-mlc-memory-latency-checker}

Intel MLCは、負荷時および無負荷時のNUMAメモリレイテンシを測定するための業界標準ツールである。設定可能なアクセスパターンで各ノードペアを計測する:

```bash
# 全ノードペアにわたる無負荷時レイテンシの行列を測定
./mlc --latency_matrix

# 負荷時レイテンシを測定（トラフィックインジェクタあり）
./mlc --loaded_latency

# 帯域幅の行列
./mlc --bandwidth_matrix
```

MLCはレイテンシをナノ秒で報告する。リモート対ローカルのレイテンシ比をカーネルの距離比と比較して、SLITが正しく較正されているかを評価する。

### numactl + stream {#numactl-stream}

MLCを使わずに手早く帯域幅を測定するには:

```bash
# ノード0の CPU がノード1のメモリから読む
numactl --cpunodebind=0 --membind=1 ./stream

# ノード0の CPU がローカルメモリから読む
numactl --cpunodebind=0 --membind=0 ./stream
```

帯域幅の比は、リモートアクセスがスループットをどれだけ低下させるかを示す。SLITが報告する距離比が2:1の2ソケットシステムでは、リモートアクセスの帯域幅低下は通常50%ではなく30–50%になる。インターコネクトの帯域幅は非対称であることが多く、ボトルネックがDRAM帯域幅ではなくQPI/UPIリンクに移るからである。

### perf mem {#perf-mem}

`perf mem` は、ハードウェアの性能監視ユニットを使って、キャッシュミスやDRAMミスを特定のNUMAノードに帰属させることができる:

```bash
# /proc/sys/kernel/perf_event_paranoid <= 1（または CAP_PERFMON）が必要
perf mem record -a -- sleep 5
perf mem report --sort=mem,sym
```

レポートには各メモリアクセスサンプルのソースノード（local/remote）が表示されるため、ワークロードを変更することなく、局所性の悪いホットスポットを特定できる。

### 負荷時レイテンシと無負荷時レイテンシ {#loaded-vs-idle-latency}

シナリオによって重要となる測定値は2つある:

- **無負荷時レイテンシ**（競合するトラフィックなし）: ハードウェアの生のラウンドトリップ時間を表す。SLITの比率を検証するのに役立つ。
- **負荷時レイテンシ**（帯域幅インジェクタあり）: 実際のワークロード条件下でのレイテンシを表す。負荷時にはインターコネクトの競合により、リモートレイテンシが無負荷時の測定値から想定されるよりも大幅に高くなることがある。

帯域幅律速のワークロード（データベース、HPC）では、負荷時レイテンシと帯域幅の飽和が主な制約となる。レイテンシに敏感なワークロード（インメモリキャッシュ、トレーディングシステム）では、無負荷時レイテンシと軽負荷時のテールレイテンシのほうが重要である。

---

## マルチホップNUMA {#multi-hop-numa}

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

### AMD EPYCのマルチダイトポロジ {#amd-epyc-multi-die-topology}

AMD EPYCプロセッサは、1ソケットあたり複数のNUMAノード（ダイごと、またはクアドラントごとに1つ）を公開する。2ソケットのEPYCシステムは4個または8個のNUMAノードを提示することがあり、ソケット内の距離は12、ソケット間の距離は32となる。EPYCシステムのBIOSが提供するSLITは概ね正確である — AMDは検証済みのACPIテーブルを出荷している。EPYC向けの `node_reclaim_distance` のオーバーライド（前述）は、32でも回収の恩恵を受けられるほど十分に高速であることを考慮したものである。

---

## よくある落とし穴 {#common-pitfalls}

!!! warning "すべての距離が10: 距離はおそらく誤っている"
    `numactl --hardware` がすべてのノード距離を10と表示する場合、ファームウェアが実際の SLIT を提供していないか、SLIT がすべてローカルの値で合成されている。これは次の2つのよくあるシナリオで起こる:

    1. **仮想化された NUMA** — ハイパーバイザは（例えば vCPU を分散させるために）NUMA ノードを公開するが、ゲストのメモリはすべてホストの同じ NUMA ドメイン上にある。実際のリモートコストが存在しないため、ゲストの距離はすべて10になる。

    2. **壊れた BIOS SLIT** — 一部のファームウェアは、実際のマルチソケットハードウェア上でも誤ってすべての距離を10と報告する。カーネルはすべてのノードを等距離として扱い、ローカルメモリからの割り当てを優先しなくなる。NUMA ワークロードの性能が、明らかな理由なく悪化することがある。

    壊れた BIOS のケースを検出するには: `lscpu` が複数のソケットを示しているのに `numactl -H` がすべての距離を10と示している場合、SLIT はほぼ確実に誤っている。BIOS のアップデートを確認するか、`acpidump | acpixtract -sLIT` を使って生のテーブルを調べる。

!!! warning "実際のNUMA効果のないVMのNUMAトポロジ"
    `numaNodes=2` で構成された VM は距離行列を持つが、ハイパーバイザがゲストメモリを単一のホスト NUMA ノードから割り当てている場合、どちらのゲストノードへのアクセスも同じ物理メモリに行く。カーネルの NUMA バランシングと zonelist の判断はゲストの距離テーブルに基づいて行われるため、性能上の利益のないマイグレーションを試み、オーバーヘッドを増やすこともある。実際のハードウェアトポロジに対応していない VM の中では、NUMA バランシングの無効化を検討すること（`echo 0 > /proc/sys/kernel/numa_balancing`）。

!!! warning "距離値はu8 — 最大255"
    `DISTANCE_BITS 8` は、距離が符号なし8ビット値として格納されることを意味する。`numa_set_distance()` は `u8` に収まらない値を拒否する。255を超える距離を提供するファームウェア（珍しいが、深く階層化されたファブリックトポロジでは起こりうる）は、クランプされるか拒否され、`__node_distance()` は `REMOTE_DISTANCE` にフォールバックする。

!!! warning "numactlはメモリのみのノードを表示しないことがある"
    CPU を持たないノード（例: CXL メモリノード、HMAT の generic initiator ノード）は、numactl のバージョンやカーネルの設定によっては `numactl --hardware` の出力に現れないことがある。`ls /sys/devices/system/node/` を使ってメモリのみのノードも含めたすべてのノードを列挙し、各ノードの `distance` ファイルを直接読むこと。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [arch/x86/mm/numa.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/numa.c) — `x86_numa_init()` と、距離行列に値を設定するACPI/AMD/OF/dummyの初期化チェーン
- [mm/numa_memblks.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/numa_memblks.c) — `__node_distance()`、`numa_set_distance()`、およびフラット配列 `numa_distance[]` による格納
- [drivers/base/node.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/base/node.c) — `node_read_distance()`: `/sys/devices/system/node/nodeN/distance` を公開するsysfsハンドラ
- [mm/page_alloc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) — `build_zonelists()` と `find_next_best_node()`: 距離がフォールバックzonelistの順序をどう決めるか
- [kernel/sched/fair.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/fair.c) — `task_numa_migrate()` と `should_numa_migrate_memory()`: スケジューラによる `node_distance()` の利用

### LWNの記事 {#lwn-articles}

- [NUMA distance and scheduler topology](https://lwn.net/Articles/392116/) — スケジューラが距離に基づいてノードをスケジューリングドメインにグループ化する方法
- [Heterogeneous memory management and NUMA](https://lwn.net/Articles/756022/) — 距離特性の異なるメモリティアが最近のカーネルでどう扱われるか

### 関連ドキュメント {#related-docs}

- [NUMA Topology Discovery: ACPI SRAT and SLIT](numa-acpi-srat.md) — ファームウェアのSRAT/SLITテーブルがどのように解析され、`numa_distance[]` に値が設定されるか
- [NUMA Zonelist Construction and Fallback Ordering](numa-zonelist.md) — `node_distance()` がページアロケータのフォールバックノード順序をどう決めるか
- [NUMA Effects on Memory Reclaim](numa-reclaim.md) — `RECLAIM_DISTANCE` が距離値を使ってローカル回収をどう制御するか
- [CXL Memory and Kernel Memory Tiering](cxl-memory-tiering.md) — DRAM以外のメモリティアが距離モデルをどう拡張するか
