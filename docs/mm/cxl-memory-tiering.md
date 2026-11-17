# CXLメモリとカーネルのメモリ階層化 {#cxl-memory-and-kernel-memory-tiering}

> CXLとカーネルのメモリティアフレームワークでDRAMの限界を超えて拡張する

## NUMAノードとしてのCXLメモリ {#cxl-memory-as-a-numa-node}

Compute Express Link（CXL）はPCIeの上に構築された高速なキャッシュコヒーレント相互接続であり、メモリ拡張カードをCPUに接続できるようにする。オペレーティングシステムの視点からは、CXLメモリは**CPUを持たないNUMAノード**、すなわちローカルDRAMよりアクセスレイテンシの高いメモリ専用ノードとして見える。

CXL 1.1と2.0のデバイスは単一のPCIeリンクで接続される。要求がPCIeファブリックを通過しなければならないため、CXLメモリへのレイテンシはDDRより高いが、その差はハードウェアやリンクの世代によって異なる。CXL 3.0はファブリックレベルのスイッチングを追加し、複数ホストでの共有や追加のトポロジーを可能にする。

!!! note "実際のレイテンシと帯域幅はハードウェア依存である"
    特定のレイテンシや帯域幅の数値を保証として扱ってはならない。実際の数値はCXLデバイス、ホストのメモリコントローラ、プラットフォームに依存する。カーネルは、ハードコードされた値ではなく、HMAT（Heterogeneous Memory Attribute Table）やCXLのCDATテーブルから得た性能測定値を使って、デバイスを正しいティアに配置する。

LinuxのCXLドライバは、リージョンの有効化時に各CXLメモリリージョンをNUMAノードとして登録する。リージョンの性能特性は（CDATから設定される）`access_coordinate`データから読み取られ、notifierを通じてメモリティアフレームワークに渡される。`drivers/cxl/core/region.c`内の関連する呼び出しチェーンは次のとおりである。

```c
/* drivers/cxl/core/region.c */
static int cxl_region_calculate_adistance(struct notifier_block *nb,
                                          unsigned long nid, void *data)
{
    struct cxl_region *cxlr = container_of(nb, struct cxl_region,
                                           adist_notifier);
    struct access_coordinate *perf;
    int *adist = data;

    perf = &cxlr->coord[ACCESS_COORDINATE_CPU];
    if (mt_perf_to_adistance(perf, adist))
        return NOTIFY_OK;
    return NOTIFY_STOP;
}
```

`mt_perf_to_adistance()`は、ハードウェアの性能座標を、メモリティアフレームワークがノードをティアに割り当てるために使う抽象距離の値に変換する。

## メモリティアフレームワーク {#the-memory-tier-framework}

### 抽象距離 {#abstract-distance}

カーネルはメモリの速度を**抽象距離**（abstract distance）として表す。これは無次元の整数で、小さいほど速い。通常のDRAMは`MEMTIER_ADISTANCE_DRAM`で定義される基準の抽象距離を持つ。

```c
/* include/linux/memory-tiers.h */
#define MEMTIER_CHUNK_BITS   7
#define MEMTIER_CHUNK_SIZE   (1 << MEMTIER_CHUNK_BITS)   /* 128 */

/* DRAMよりわずかに速いデバイスが同じティアを共有するよう少しずらす */
#define MEMTIER_ADISTANCE_DRAM  ((4L * MEMTIER_CHUNK_SIZE) + (MEMTIER_CHUNK_SIZE >> 1))
```

各`memory_tier`は幅`MEMTIER_CHUNK_SIZE`（128）の抽象距離の範囲をカバーする。抽象距離が同じ128単位の窓に入るデバイスは同じティアにまとめられる。`adistance_start`が小さいほど上位の（速い）ティアである。

### ティアの割り当て {#tier-assignment}

NUMAノードは`struct memory_dev_type`を通じてティアに割り当てられる。

```c
/* include/linux/memory-tiers.h */
struct memory_dev_type {
    struct list_head tier_sibling;  /* 同じティア内の他のタイプ */
    struct list_head list;          /* 1つのドライバが管理するタイプ */
    int adistance;                  /* このタイプの抽象距離 */
    nodemask_t nodes;               /* この抽象距離を持つノード */
    struct kref kref;
};
```

ブート時、`memory_tier_init()`は`default_dram_type`を使って、オンラインのすべてのノードをデフォルトのティアに配置する。CXLリージョンが有効化されると、そのドライバはadistanceのnotifierを登録する（`register_mt_adistance_algorithm()`）。するとフレームワークがnotifierチェーンを呼び出し、CXLドライバがCXLリージョンのadistanceを報告し、`find_create_memory_tier()`がノードを既存のティアに配置するか、新しいティアを作成する。

### sysfsでの表現 {#sysfs-representation}

メモリティアは次の場所で見ることができる。

```
/sys/devices/virtual/memory_tiering/memory_tierN/
    nodelist   # このティア内のNUMAノード
```

`memory_tierN`の`N`は、ティアの`adistance_start`を`MEMTIER_CHUNK_BITS`だけ右シフトしたものから導かれる。`N`が小さいほど上位の（速い）ティアである。

ローカルDRAM（ノード0）とCXL拡張ノード（ノード1）を持つシステムでは、次のように見えるかもしれない。

```
/sys/devices/virtual/memory_tiering/memory_tier4/nodelist  → 0
/sys/devices/virtual/memory_tiering/memory_tier8/nodelist  → 1
```

!!! note "ティアの番号は挿入順ではなく抽象距離から導かれる"
    同じハードウェアトポロジーを持つ2つのシステムは同じティア番号を持つ。番号は順番に割り振られるのではなく、性能座標から計算されるからである。

### 降格先のトポロジー {#demotion-target-topology}

フレームワークは、各NUMAノードをその優先降格先に対応付ける`node_demotion[]`配列を構築する。3つのノードがあるとする。

```
memory_tier0 (fastest) → node 1  (HBM)
memory_tier1           → node 0  (DRAM)
memory_tier2 (slowest) → node 2  (CXL/PMEM)
```

降格パスは次のようになる。

- ノード1（HBM）→ ノード0（DRAM）
- ノード0（DRAM）→ ノード2（CXL）
- ノード2（CXL）→ どこへもなし（終端）

`mm/memory-tiers.c`の`next_demotion_node()`は優先降格先を返し、複数の候補がある場合は優先マスク内のノードからランダムに選ぶ。

`node_is_toptier()`は、`top_tier_adistance`以上のティアにあるノードに対してtrueを返す。最上位ティアのノードが降格先になることはない。

## pageの降格: 低速パスを駆動するkswapd {#page-demotion-kswapd-driving-the-slow-path}

メモリ逼迫下でカーネルがメモリを回収するとき、カーネルはコールドなpageをディスクへスワップするより、高速メモリから低速メモリへ**降格**（demote）することを好む。これは`mm/vmscan.c`で実装されている。

### 降格が発動するとき {#when-demotion-fires}

鍵となる述語は`mm/vmscan.c`の`can_demote()`である。

```c
static bool can_demote(int nid, struct scan_control *sc, struct mem_cgroup *memcg)
```

これは、ノードが有効な降格先（空き容量のある下位ティア）を持ち、かつ`numa_demotion_enabled`が設定されていればtrueを返す。降格が利用可能な場合、回収パスは通常の`folio_list`と並んで`demote_folios`リストを構築する。

```c
/* mm/vmscan.c — shrink_folio_list() */
if (do_demote_pass &&
    can_demote_folio(folio, ...))
    list_add(&folio->lru, &demote_folios);
```

続いて`demote_folio_list()`が`migrate_pages()`を呼び出し、それらのfolioを降格先ノードへ移動する。降格できなかったfolio（降格先に空きがない、マイグレーションの失敗）は回収リストに戻され、スワップされることがある。

カーネルがスワップより降格を好むのは、降格されたメモリへのアクセスが通常スワップI/Oよりはるかに速く、またデータを破棄せずにメモリ上に保てるからである。

!!! tip "numa_demotion_enabledで降格を有効にする"
    降格は`numa_demotion_enabled`変数で制御され、次の場所で公開されている。
    ```
    /proc/sys/kernel/numa/demotion_enabled
    ```
    デフォルトでは無効である。有効にするには`1`を書き込む。これがないと、kswapdはCXLへ降格するのではなく、ディスク（またはzswap）へスワップする。

### 降格のvmstatカウンタ {#vmstat-counters-for-demotion}

`/proc/vmstat`は降格イベントを起点別に追跡する。

| カウンタ | 意味 |
|---------|---------|
| `pgdemote_kswapd` | kswapdのバックグラウンド回収によって降格されたpage |
| `pgdemote_direct` | 直接回収（割り当てを行うプロセス）によって降格されたpage |
| `pgdemote_khugepaged` | khugepagedの分割中に降格されたpage |
| `pgdemote_proactive` | プロアクティブ回収によって降格されたpage |

## pageの昇格: 高速パスを駆動するNUMAバランシング {#page-promotion-numa-balancing-driving-the-fast-path}

**昇格**（promotion）はその逆で、ホットなpageを低速メモリ（CXL）から高速メモリ（DRAM）へ移動することである。その仕組みはNUMAバランシングである。

NUMAバランシングは、pageを定期的にページテーブルからアンマップすることで機能する（NUMAヒンティングフォールト）。pageがアクセスされるとマイナーフォールトが発生し、カーネルはどのCPUがそのpageにアクセスしたかを観測する。CXLメモリ内のpageが、ローカルDRAMがより上位のティアにあるCPUから頻繁にアクセスされていれば、カーネルはそのpageをより速いティアへマイグレーションする。

NUMAバランシングのメモリ階層化モードは、`numa_balancing_mode`のビット1を設定することで有効になる。

```c
/* include/linux/sched/sysctl.h */
#define NUMA_BALANCING_MEMORY_TIERING  0x2
extern int sysctl_numa_balancing_mode;
```

`NUMA_BALANCING_MEMORY_TIERING`が有効なとき、`folio_use_access_time()`関数はfolioの`_last_cpupid`フィールドを転用し、最後のCPU/PIDの代わりにpageのアクセス時刻を記録する。これにより、バランサはCPUアフィニティだけでなくアクセスの新しさに基づいて、低速メモリ内のホットなpageを特定できる。

### 昇格のvmstatカウンタ {#vmstat-counters-for-promotion}

| カウンタ | 意味 |
|---------|---------|
| `pgpromote_success` | より速いティアへの昇格に成功したpage |
| `pgpromote_candidate` | 昇格の候補として特定されたpage |
| `pgpromote_candidate_nrl` | スキップされた候補（最近ローカルでなかったもの） |

## コールドpageの降格のためのDAMON連携 {#damon-integration-for-cold-page-demotion}

DAMON（Data Access MONitor）を使うと、高速メモリ内のコールドなpageを特定し、メモリ逼迫によってkswapdが動かざるを得なくなる前に、それらをプロアクティブに降格できる。

関連するDAMOS（DAMON-based Operation Scheme）のアクションは`DAMOS_MIGRATE_COLD`である。

```c
/* include/linux/damon.h */
enum damos_action {
    DAMOS_WILLNEED,
    DAMOS_COLD,
    DAMOS_PAGEOUT,       /* スワップへ回収する — 降格はしない */
    DAMOS_HUGEPAGE,
    DAMOS_NOHUGEPAGE,
    DAMOS_MIGRATE_HOT,
    DAMOS_MIGRATE_COLD,  /* コールドな領域をより遅いノードへマイグレーションする */
    ...
};
```

!!! warning "DAMOS_PAGEOUTは降格しない"
    `DAMOS_PAGEOUT`はpageをスワップまたはzswapへ回収する。pageをCXLノードへマイグレーション**しない**。階層化メモリの降格が目的なら`DAMOS_MIGRATE_COLD`を使うこと。

`DAMOS_MIGRATE_COLD`と、CXLノードを含むターゲットで構成されたDAMONスキームは、コールドなpageをDRAMからCXLへプロアクティブに移動し、メモリ逼迫を待たずにウォームなpageを高速メモリに保つ。

`/sys/kernel/mm/damon/`経由でDAMONスキームを構成する方法の詳細は、[DAMONのドキュメント](damon.md)を参照してほしい。

## 設定 {#configuration}

### CXLハードウェアでは自動 {#automatic-with-cxl-hardware}

BIOSが有効なHMATまたはCDATテーブルを提供し、CXLドライバがロードされているシステムでは、メモリティアフレームワークはブート時に自動的に自身を構成する。

1. `memory_tier_init()`が既存のNUMAノードをデフォルトのDRAMティアに配置する。
2. CXLリージョンの有効化がadistanceのnotifierを登録する。
3. フレームワークが、測定された抽象距離に基づいてCXLノードをティアに割り当てる。
4. `establish_demotion_targets()`が`node_demotion[]`テーブルを構築する。

降格と、望むNUMAバランシングモードを有効にする以外に、手動での設定は必要ない。

### sysctlのつまみ {#sysctl-knobs}

```
/proc/sys/kernel/numa/demotion_enabled
```
降格を有効または無効にする（0/1）。デフォルトは0（無効）である。

```
/proc/sys/kernel/numa_balancing
```
NUMAバランシングの挙動を制御するビットマップ。通常のNUMAバランシングと併せてメモリ階層化モードを有効にするには、ビット1を設定する（`value |= 2`）。

### 手動配置のためのnumactl {#numactl-for-manual-placement}

CXLのティアが自動設定されないシステムやテスト用には、`numactl`で割り当てを特定のノードに束縛できる。

```bash
# ノード1（CXL）から割り当て、あふれたときはノード0（DRAM）を優先する
numactl --preferred=0 --membind=0,1 ./my_process

# CXLノードのみからの割り当てを強制する
numactl --membind=1 ./my_process
```

## 監視 {#monitoring}

### /proc/vmstat {#procvmstat}

上で述べた降格と昇格のカウンタは、`/proc/vmstat`で、また`sar`、`vmstat`、`perf stat`経由で利用できる。

```bash
watch -n 1 'grep -E "pgdemote|pgpromote" /proc/vmstat'
```

軽い逼迫下にある健全な階層化メモリシステムでは、コールドなpageをCXLへ移す`pgdemote_kswapd`の活動が安定して見られ、NUMAバランシングがホットなpageをDRAMへ戻すときに`pgpromote_success`のイベントが時折見られる。

### ティアへの所属 {#tier-membership}

```bash
cat /sys/devices/virtual/memory_tiering/memory_tier*/nodelist
```

どのNUMAノードがどのティアに属しているかを示す。

### tracepoint {#tracepoints}

`mm/vmscan.c`には現在、階層化専用のtracepointはないが、標準の`mm_migrate_pages` tracepoint（`include/trace/events/migrate.h`）が降格のマイグレーションごとに発火し、マイグレーションのモードと理由を報告する。

## 現在の制約 {#current-limitations}

**CXLノードの自動検出にはファームウェアの協力が必要である。** HMATやCDATのデータが欠けているか誤っていると、CXLノードがDRAMと同じティアに入ってしまい、降格が起こらない。ブート時のCXLドライバとメモリティアフレームワークのメッセージを`dmesg`で確認すること。

**v6.15およびそれ以降のパッチで修正が入る前のカーネルでは、降格は`cpusets.mems_allowed`を尊重しない。** cgroupに属するpageが、その`mems_allowed`集合の外にあるノードへ降格されることがある。これは`Documentation/driver-api/cxl/allocation/reclaim.rst`に記載された既知の制約である。

**khugepagedが階層化と衝突することがある。** khugepagedがCXLノード上のTHPを分割すると、`pgdemote_khugepaged`カウンタが増加する。構成によっては、khugepagedの活動が、降格で移動したばかりのpageを再び昇格させてしまうことがある。

**NUMAバランシングによる昇格はオーバーヘッドを増やす。** バランサはアクセスパターンを観測するためにpageをアンマップする（次のアクセス時にフォールトを起こす）ことで機能する。コールドなpageを持つCXLノードでは通常問題ないが、昇格が積極的すぎる誤った構成のスキームはフォールト率を上げうる。

!!! note "この分野は活発に開発されている"
    メモリ階層化、CXLのサポート、そしてDAMON、kswapd、NUMAバランシングの間の相互作用は、カーネルのバージョンごとに活発に変化している。ここで説明した特定の挙動に依存する場合は、必ずそのカーネルバージョンの変更履歴を確認すること。

## 参考文献 {#further-reading}

- [Kernel docs: CXL](https://docs.kernel.org/driver-api/cxl/index.html): メモリリージョンの有効化やCDATの解析を含む、CXLサブシステムの公式カーネルドキュメント
- [`Documentation/driver-api/cxl/`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/driver-api/cxl): CXLメモリリージョン、割り当て、回収の挙動に関するカーネルソース内のドキュメント
- [`drivers/cxl/`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/cxl): CXLドライバのソース。リージョンの有効化、CDATの解析、adistanceのnotifier
- [`mm/memory-tiers.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memory-tiers.c): メモリティアフレームワーク。`find_create_memory_tier()`、`establish_demotion_targets()`、抽象距離のロジック
- [LWN: Memory tiering in the kernel](https://lwn.net/Articles/858215/): 元々のメモリ階層化フレームワークの設計と降格の基盤（2021年）
- [LWN: CXL memory management](https://lwn.net/Articles/894816/): CXLノードがLinuxのNUMAおよび階層化サブシステムとどう統合されるか（2022年）
- [LWN: DAMOS_MIGRATE_COLD and proactive reclaim](https://lwn.net/Articles/905967/): DAMONスキームを使い、メモリ逼迫が高まる前にコールドなpageをより遅いティアへ降格する（2022年）
- [numa.md](numa.md): NUMAノードのトポロジー、HMATの距離テーブル、昇格を駆動する自動バランシングの基盤
- [memory-bandwidth.md](memory-bandwidth.md): なぜCXLノードが`MPOL_INTERLEAVE`の対象に適さないのか、そして帯域幅のティアがポリシー判断にどう影響するか
- [damon.md](damon.md): CXLへのプロアクティブな降格のための`DAMOS_MIGRATE_COLD`スキームの構成
