# 負荷分散 {#load-balancing}

> スケジューラがタスクをCPU間に分散させる仕組み

## なぜ負荷分散は難しいのか {#why-load-balancing-is-hard}

負荷分散は次の要素のトレードオフを取らなければならない:

- **スループット**: すべてのCPUを使うようタスクを均等に分散させる
- **キャッシュアフィニティ**: 最近CPU Aで実行されたタスクはキャッシュが温まっている — それをBへマイグレーションするのは高コストである
- **NUMAローカリティ**: NUMAノードをまたぐマイグレーションは、コア間のマイグレーションよりも高コストである
- **トポロジの考慮**: ハイブリッドCPUでは、Pコア/Eコアをまたいでマイグレーションしない

Linuxスケジューラは、これらのトレードオフに対処するために**階層的なスケジューリングドメイン**のトポロジを使う。

## スケジューリングドメインとグループ {#scheduling-domains-and-groups}

```
System topology (example: 2-socket NUMA, 4 cores each, 2-way SMT):

NUMA domain
├── MC (multicore) domain — socket 0
│   ├── SMT group: {cpu0, cpu4}   (physical core 0)
│   ├── SMT group: {cpu1, cpu5}   (physical core 1)
│   ├── SMT group: {cpu2, cpu6}   (physical core 2)
│   └── SMT group: {cpu3, cpu7}   (physical core 3)
└── MC domain — socket 1
    ├── SMT group: {cpu8, cpu12}
    ...
```

各スケジューリングドメインは、いつ分散処理を行うかを制御するパラメータを持つ:

```c
/* include/linux/sched/topology.h */
struct sched_domain {
    struct sched_domain __rcu *parent; /* 上位はNUMAドメインまで */
    struct sched_domain __rcu *child;  /* 下位はSMTドメインまで */
    struct sched_group  *groups;       /* グループの循環リスト */

    unsigned long       min_interval;  /* 負荷最小時の分散間隔 */
    unsigned long       max_interval;  /* 負荷最大時の分散間隔 */
    unsigned int        busy_factor;   /* ビジー時の追加の分散係数 */

    int                 balance_interval; /* 現在の間隔 */
    unsigned int        nr_balance_failed; /* 連続して失敗した分散の回数 */

    unsigned long       last_balance;  /* 最後に分散したときのjiffies */

    int                 span_weight;   /* このドメインがカバーするCPU数 */
    cpumask_var_t       span;          /* このドメイン内のCPU */

    /* 動作を制御するフラグ */
    unsigned int        flags;
    /* SD_LOAD_BALANCE: 分散処理に参加する */
    /* SD_BALANCE_NEWIDLE: CPUがアイドルになったときに分散する */
    /* SD_BALANCE_EXEC: exec()時に分散する */
    /* SD_BALANCE_FORK: fork()時に分散する */
    /* SD_WAKE_AFFINE: 現在のCPUで起床させようとする */
    /* SD_ASYM_PACKING: 非対称パッキング（Eコア/Pコア）にSDを使う */
    /* SD_NUMA: NUMAドメイン */
};
```

## 定期的な負荷分散 {#periodic-load-balancing}

各CPUは（スケジューラティックを介して）定期的に`load_balance()`を実行する:

```c
/* kernel/sched/fair.c */
static int load_balance(int this_cpu, struct rq *this_rq,
                         struct sched_domain *sd, enum cpu_idle_type idle,
                         int *continue_balancing)
{
    struct lb_env env = {
        .sd             = sd,
        .dst_cpu        = this_cpu,
        .dst_rq         = this_rq,
        .idle           = idle,
        .loop_break     = SCHED_NR_MIGRATE_BREAK,
        .cpus           = cpus,
        .fbq_type       = all,
        .tasks          = LIST_HEAD_INIT(env.tasks),
    };

    /* このスケジューリングドメイン内で最もビジーなグループを探す */
    struct sched_group *group = find_busiest_group(&env);
    if (!group)
        goto out_balanced;

    /* 最もビジーなグループ内で最もビジーなランキューを探す */
    struct rq *busiest = find_busiest_queue(&env, group);
    if (!busiest)
        goto out_balanced;

    env.src_cpu = busiest->cpu;
    env.src_rq  = busiest;

    /* 最もビジーなランキューからthis_rqへタスクを移動する */
    ld_moved = move_tasks(&env);

    return ld_moved;
}
```

## 負荷の指標: PELT {#load-metric-pelt}

スケジューラは、各タスクの負荷への寄与を測定するために**PELT**（Per-Entity Load Tracking）を使う。Linux 3.8でPaul Turner（Google）によって導入された — [`9d85f21c94f7`](https://git.kernel.org/linus/9d85f21c94f7f7a84d0ba686c58aa6d9da58fdbb)。PELTはCPU使用率の指数加重移動平均を追跡する:

```
util_avg = util_avg * decay + new_util * (1 - decay)
```

減衰係数は、32msより古い履歴の重みが1%未満になるように設定されている。

```c
/* kernel/sched/pelt.c */
static u32 accumulate_sum(u64 delta, struct sched_avg *sa,
                           unsigned long load, unsigned long runnable,
                           int running)
{
    u32 contrib = (u32)delta;  /* 前回の更新からの差分 */

    /* 累積使用率のための等比級数の和 */
    if (delta < 1024) {
        contrib = div_u64(delta * sa->period_contrib + (contrib << 10),
                          1024);
    } else {
        sa->util_sum = (sa->util_sum >> 1) + (running << (SCHED_CAPACITY_SHIFT - 1));
    }
    return contrib;
}
```

## struct lb_env: 分散処理のコンテキスト {#struct-lb_env-the-balancing-context}

```c
/* kernel/sched/sched.h */
struct lb_env {
    struct sched_domain *sd;         /* 分散対象のドメイン */

    struct rq           *src_rq;     /* 移動元ランキュー（最もビジー） */
    int                  src_cpu;

    int                  dst_cpu;    /* 移動先CPU */
    struct rq           *dst_rq;

    struct cpumask      *dst_grpmask; /* 移動先グループ内のCPU */
    int                  new_dst_cpu;

    enum cpu_idle_type   idle;       /* CPU_IDLE / CPU_NOT_IDLE / CPU_NEWLY_IDLE */

    long                 imbalance;  /* 解消すべき負荷の不均衡 */
    struct cpumask      *cpus;       /* 対象となるCPU */

    unsigned int         flags;
    unsigned int         loop;
    unsigned int         loop_break;
    unsigned int         loop_max;

    enum fbq_type        fbq_type;   /* フィルタ種別: all/regular/remote/base */
    enum migration_type  migration_type;

    struct list_head     tasks;      /* マイグレーションするタスク */
};
```

## newidle_balance: CPUがアイドルになったとき {#newidle_balance-when-cpu-goes-idle}

CPUにタスクがなくなると、すぐによりビジーなCPUから仕事を引き取ろうとする:

```c
/* kernel/sched/fair.c */
static int newidle_balance(struct rq *this_rq, struct rq_flags *rf)
{
    unsigned long next_balance = jiffies + HZ;
    int this_cpu = this_rq->cpu;
    u64 t0, t1, curr_cost = 0;
    struct sched_domain *sd;
    int pulled_task = 0;

    update_blocked_averages(this_cpu);

    rcu_read_lock();
    for_each_domain(this_cpu, sd) {
        int continue_balancing = 1;

        if (this_rq->avg_idle < curr_cost + sd->avg_scan_cost ||
            !READ_ONCE(sd->nohz_idle))
            break;

        if (sd->flags & SD_BALANCE_NEWIDLE) {
            pulled_task = load_balance(this_cpu, this_rq, sd,
                                        CPU_NEWLY_IDLE, &continue_balancing);
        }

        if (pulled_task || this_rq->nr_running > 0)
            break;
    }
    rcu_read_unlock();

    return pulled_task;
}
```

`avg_idle`は平均アイドル時間を追跡する。CPUのアイドル予想時間が走査コストより短ければ、分散処理をスキップする — キャッシュを汚すだけの価値がないからである。

## 起床時の分散: select_task_rq_fair {#wakeup-balancing-select_task_rq_fair}

タスクが起床すると、スケジューラはそれをどのCPUに配置するかを選ぶ:

```c
/* kernel/sched/fair.c */
static int select_task_rq_fair(struct task_struct *p, int prev_cpu,
                                int wake_flags)
{
    int sync = (wake_flags & WF_SYNC) && !(current->flags & PF_EXITING);
    struct sched_domain *tmp, *sd = NULL;
    int cpu = smp_processor_id();
    int new_cpu = prev_cpu;
    int want_affine = 0;

    /* SD_WAKE_AFFINE: 現在のCPUで起床させようとする（プロデューサ/コンシューマに有効） */
    rcu_read_lock();
    for_each_domain(cpu, tmp) {
        if (want_affine && (tmp->flags & SD_WAKE_AFFINE) &&
            cpumask_test_cpu(prev_cpu, sched_domain_span(tmp))) {
            /* wake affine: 現在のCPUが良い選択かを確認する */
            if (wake_affine(tmp, p, cpu, prev_cpu, sync))
                new_cpu = cpu;
            break;
        }
        if (tmp->flags & sd_flag)
            sd = tmp;
    }

    /* 選んだドメイン内で最もアイドルなCPUを探す */
    if (sd)
        new_cpu = find_idlest_cpu(sd, p, cpu, prev_cpu, sd_flag);

    rcu_read_unlock();
    return new_cpu;
}
```

**wake affine**: タスクBが常にタスクAの後に起床する場合（プロデューサ/コンシューマ）、キャッシュラインを共有できるよう、Bを同じCPUか近くのCPUにスケジューリングする。

## 非対称CPUにおけるmisfitタスク {#misfit-tasks-on-asymmetric-cpus}

big.LITTLE/ハイブリッドCPU（ARM big.LITTLE、IntelのハイブリッドP+Eコア）では、現在のCPUが提供する以上の処理能力を要求するタスクは「misfit」となる:

```c
/* kernel/sched/fair.c */
static bool task_fits_capacity(struct task_struct *p, long capacity)
{
    return fits_capacity(task_util(p), capacity);
}

static inline bool fits_capacity(unsigned long util, unsigned long capacity)
{
    /* 80%の閾値: utilがCPU処理能力の80%未満ならタスクは収まる */
    return util < capacity * 1024 / 1280;  /* 80%の閾値 */
}

/* misfitマイグレーション: nohz_balance_kickによって起動される */
static void check_misfit_status(struct rq *rq, struct sched_domain *sd)
{
    if (rq->misfit_task_load &&
        (rq->curr->nr_cpus_allowed == 1 ||
         !task_fits_capacity(rq->curr, rq->cpu_capacity))) {
        sd->balance_interval = 1;  /* 直ちに分散する */
    }
}
```

## 負荷分散の観測 {#observing-load-balancing}

```bash
# CPUごとのランキュー長
watch -n 1 'cat /proc/schedstat | awk "NR%2==0{cpu++; print \"cpu\"cpu, \$9}"'

# 負荷分散の統計
cat /proc/schedstat | head -5
# version 15
# timestamp 123456789
# cpu0 0 0 0 0 0 0 0 0 0   ← 各種フィールド
# domain0 00000003 lb_count lb_balanced lb_failed ...

# perf: スケジューリングイベント
perf stat -e sched:sched_migrate_task,sched:sched_wakeup -a sleep 5

# タスクのマイグレーションをトレースする
echo 1 > /sys/kernel/tracing/events/sched/sched_migrate_task/enable
cat /sys/kernel/tracing/trace_pipe
# myapp-1234 [002] sched_migrate_task: comm=myapp pid=1234 prio=120 orig_cpu=2 dest_cpu=5

# スケジューリングドメインのトポロジ
cat /proc/sys/kernel/sched_domain/cpu0/domain0/name
# MC
cat /proc/sys/kernel/sched_domain/cpu0/domain1/name
# NUMA

# 各タスクのPELT使用率シグナル
cat /proc/<pid>/sched | grep -i util
```

## 参考文献 {#further-reading}

- [EAS: エネルギーを考慮したスケジューリング](eas.md) — ヘテロジニアスなハードウェアにおけるエネルギー最適なCPU選択
- [スケジューラ: EEVDF](eevdf.md) — CPUごとのタスク選択
- [スケジューラ: スケジューリングドメイン](sched-domains.md) — ドメインの構築
- [メモリ管理: NUMA](../mm/numa.md) — NUMAトポロジと負荷分散
- `kernel/sched/fair.c` — CFS/EEVDFと負荷分散
- `kernel/sched/topology.c` — ドメインの構築
