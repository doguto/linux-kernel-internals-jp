# スケジューリングドメイン {#scheduling-domains}

> カーネルが負荷分散のためにCPUをどう階層化するか

## スケジューリングドメインとは何か {#what-scheduling-domains-are}

Linuxスケジューラは、単に個々のCPU間で負荷を分散するだけではない。SMTの兄弟スレッド、CPUコア、NUMAソケットといったハードウェアを反映した階層的なトポロジを使う。この階層は**スケジューリングドメインツリー**と呼ばれる。

ツリーの各レベルは、CPUの集合にまたがる`sched_domain`である。負荷分散が実行されるときは、最も狭いもの（SMT）から最も広いもの（NUMA）へとドメインごとに処理し、トポロジの境界を尊重する。

これが重要なのは次の理由による:

- SMTの兄弟スレッド間でタスクを移動するのは安い（L1/L2キャッシュを共有している）
- NUMAソケット間の移動は高い（リモートメモリアクセス）
- スケジューラは、安いレベルでは積極的に、高いレベルでは控えめに負荷分散すべきである

## struct sched_domain {#struct-sched_domain}

```c
// include/linux/sched/topology.h
struct sched_domain {
    struct sched_domain __rcu *parent;   // より広いドメイン（例: MCの上のNUMA）
    struct sched_domain __rcu *child;    // より狭いドメイン（例: MCの下のSMT）
    struct sched_group *groups;          // このドメイン内のsched_groupの循環リスト

    unsigned long min_interval;          // 負荷分散の最小間隔（ms）
    unsigned long max_interval;          // 負荷分散の最大間隔（ms）
    unsigned int  imbalance_pct;         // 負荷分散を行う前にX%までの不均衡を許容する
    unsigned int  cache_nice_tries;      // この試行回数の間はホットなタスクを放置する

    int flags;                           // 負荷分散の挙動を制御するSD_*フラグ
    int level;                           // 0=SMT, 1=MC, 2=PKG, 3=NUMA, ...

    unsigned long last_balance;          // 最後に負荷分散が実行された時刻（jiffies）
    unsigned int  balance_interval;      // 現在の適応的な間隔

    // CPUごとのポインタ（rcu経由でアクセス）
    // ...
};
```

### SD_*フラグ {#sd_-flags}

フラグは、各ドメインレベルでどの負荷分散が有効になるかを制御する:

```c
// include/linux/sched/sd_flags.h
SD_LOAD_BALANCE     // このレベルで負荷分散を有効にする
SD_BALANCE_NEWIDLE  // CPUがアイドルになったときに負荷分散する
SD_BALANCE_EXEC     // exec()が呼ばれたとき（新しいタスク）に負荷分散する
SD_BALANCE_FORK     // fork()がタスクを作成したときに負荷分散する
SD_WAKE_AFFINE      // 最近起床したタスクを起床元のCPUに留める
SD_ASYM_CPUCAPACITY // ドメインが異なる能力のCPUにまたがる（big.LITTLE）
SD_SHARE_PKG_RESOURCES  // CPUがラストレベルキャッシュを共有している
SD_NUMA             // ドメインがNUMAノードの境界をまたぐ
SD_OVERLAP          // ドメインが兄弟と重なりうる（NUMA）
```

## トポロジの階層 {#the-topology-hierarchy}

デフォルトのトポロジレベル（最も狭いものから最も広いものへ）:

```
NUMA (node 0)  ←────────────────────────────┐
│                                           │
└── PKG (socket 0)       PKG (socket 1) ───┘
    │                    │
    └── MC (core 0-3)    └── MC (core 4-7)
        │
        └── SMT (core 0: cpu0, cpu1)
```

CPUごとのポインタにより、特定のレベルにすばやくアクセスできる:

```c
// kernel/sched/topology.c
DEFINE_PER_CPU(struct sched_domain __rcu *, sd_llc);      // ラストレベルキャッシュのドメイン
DEFINE_PER_CPU(struct sched_domain_shared __rcu *, sd_llc_shared);
DEFINE_PER_CPU(struct sched_domain __rcu *, sd_numa);     // NUMAドメイン
DEFINE_PER_CPU(struct sched_domain __rcu *, sd_asym_packing);
DEFINE_PER_CPU(struct sched_domain __rcu *, sd_asym_cpucapacity);
```

### struct sched_domain_topology_level {#struct-sched_domain_topology_level}

アーキテクチャのコードは、レベルのリストを提供することでトポロジを定義する:

```c
// kernel/sched/topology.c
static struct sched_domain_topology_level default_topology[] = {
#ifdef CONFIG_SCHED_SMT
    { cpu_smt_mask, cpu_smt_flags, SD_INIT_NAME(SMT) },   // ハイパースレッディングの兄弟
#endif
#ifdef CONFIG_SCHED_CLUSTER
    { cpu_clustergroup_mask, cpu_cluster_flags, SD_INIT_NAME(CLS) }, // CPUクラスタ
#endif
#ifdef CONFIG_SCHED_MC
    { cpu_coregroup_mask, cpu_core_flags, SD_INIT_NAME(MC) },  // L2/L3を共有するコア
#endif
    { cpu_cpu_mask, NULL, SD_INIT_NAME(PKG) },             // 物理ソケット/パッケージ
    { NULL, },
};
```

NUMAレベルは、ACPI SRATのデータに基づいて`sched_init_numa()`によってその上に追加される。

## ドメインの構築: build_sched_domains() {#building-domains-build_sched_domains}

トポロジが変化したとき（ブート、CPUホットプラグ、cpusetのパーティション）、カーネルはドメインツリーを再構築する:

```c
// kernel/sched/topology.c
static int build_sched_domains(const struct cpumask *cpu_map,
                                struct sched_domain_attr *attr)
{
    // cpu_map内の各CPUについて:
    //   各トポロジレベルについて:
    //     sd_init() — sched_domainを割り当てて初期化する
    //     親子のポインタを接続する
    //     CPUごとのsdポインタに割り当てる
    // その後:
    //   build_sched_groups() — ドメインごとにCPUをsched_groupにつなぐ
    //   cpu_attach_domain() — 新しいドメインを設置する（古いものとRCUで入れ替える）
}
```

これは`rebuild_sched_domains()`経由で呼び出され、それは次のものによって引き起こされる:

- `sched_domain_sysctl`の変更
- CPUホットプラグのイベント
- cpusetのパーティションルートの変更

## 負荷分散: sched_balance_rq() {#load-balancing-sched_balance_rq}

負荷分散は`sched_balance_rq()`（以前の`load_balance()`）で行われる。これはスケジューラティックによって周期的に、またCPUがアイドルになったときに呼び出される:

```
Idle CPU → sched_balance_rq()
    Walk from narrowest domain to broadest:
    ├── SMT domain: too few tasks? steal from sibling HT
    ├── MC domain: unbalanced across cores? pull tasks
    ├── PKG domain: cross-socket imbalance? migrate tasks
    └── NUMA domain: large imbalance across nodes? migrate
```

重要な原則: **最も狭いドメインから始める**。MCドメイン内に十分な仕事があれば、PKGの境界は越えない。NUMAマイグレーションは最後の手段である。

### アイドル時の負荷分散 {#idle-load-balancing}

CPUが仕事を使い果たすと、`newidle_balance()`に入る:

```c
// kernel/sched/fair.c
static int newidle_balance(struct rq *this_rq, struct rq_flags *rf)
{
    // 最も狭いドメインから外側へたどる
    for_each_domain(this_cpu, sd) {
        if (!(sd->flags & SD_BALANCE_NEWIDLE))
            continue;

        pulled_tasks = sched_balance_rq(this_rq, rf, sd,
                                         CPU_NEWLY_IDLE, &continue_balancing);
        if (pulled_tasks)
            break;  // このレベルで仕事が見つかったので終了する
    }
}
```

### 周期的な負荷分散 {#periodic-balancing}

スケジューラティックは、`balance_interval` jiffiesごとに`sched_balance_softirq()`を起動する。間隔は適応的に変化する。直近の負荷分散で不均衡が見つからなければ、オーバーヘッドを減らすために`balance_interval`は倍になる（`max_interval`まで）。不均衡が見つかると、`min_interval`にリセットされる。

## ドメインのトポロジを見る {#viewing-domain-topology}

```bash
# /proc/schedstatはドメインごとの統計を表示する（CONFIG_SCHEDSTATSが必要）
cat /proc/schedstat

# sched_debugはドメインツリー全体を表示する
cat /proc/sched_debug | grep -A 20 "domain"

# CPUごとのドメイン情報
cat /sys/kernel/debug/sched/domains/cpu0/domain0/name   # "MC"
cat /sys/kernel/debug/sched/domains/cpu0/domain0/flags
cat /sys/kernel/debug/sched/domains/cpu0/domain1/name   # "PKG"
```

## cpusetによるドメインの隔離 {#domain-isolation-with-cpuset}

cpusetのパーティションルートは、スケジューリングドメインツリーを**分断する**。CPU 4-7でパーティションルートを作成すると:

1. それらのCPUは親のドメインから取り除かれる
2. `rebuild_sched_domains()`が呼び出される
3. CPU 4-7のための新しい隔離されたドメインツリーが構築される
4. 負荷分散はもはやパーティションの境界を越えない

これは、コンテナランタイムがVMやコンテナを、ホストのスケジューラからの干渉なしに排他的なCPU集合にピン留めする方法である。

## 非対称なCPU能力（big.LITTLE） {#asymmetric-cpu-capacity-biglittle}

異種CPU（Cortex-A55 + Cortex-A78）を持つARMシステムでは、`SD_ASYM_CPUCAPACITY`フラグが、異なる計算能力のCPUにまたがるドメインを示す。スケジューラは`cpu_capacity`を通じてCPU能力を追跡し、Energy Aware Scheduling（EAS）を使って、適切な大きさのCPUにタスクを配置することを優先する。

```bash
# CPUごとの能力を表示する（正規化済み、1024 = 最大）
cat /sys/devices/system/cpu/cpu*/cpu_capacity
```

## 参考文献 {#further-reading}

- [cpuset](cpuset.md) — cpusetのパーティションルートがスケジューリングドメインとどう相互作用するか
- [CPUアフィニティ](cpu-affinity.md) — タスクごとのアフィニティと、それとドメインとの関係
- [CFS](cfs.md) — 負荷分散が奉仕するfairスケジューラ
- [NUMA](../mm/numa.md) — ノードをまたぐマイグレーションが高コストな理由
