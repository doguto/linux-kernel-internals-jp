# スケジューリングティック {#the-scheduling-tick}

> カーネルが時間ベースのスケジューリングをどう駆動するか: jiffies、HZ、NOHZ

## 周期的なティック {#the-periodic-tick}

カーネルは、毎秒`HZ`回の頻度で周期的なタイマー割り込み（「ティック」）を実行する。ティックが駆動するものは次のとおりである:

- `jiffies`（粗い粒度の時間）の更新
- プロセスのCPU時間の会計の更新
- 現在のタスクをプリエンプトすべきかどうかのチェック（`scheduler_tick`）
- 満了したタイマー（ホイールタイマー）の実行

```c
/* kernel/time/tick-common.c */
static void tick_handle_periodic(struct clock_event_device *dev)
{
    tick_periodic(cpu);
}

static void tick_periodic(int cpu)
{
    if (tick_do_timer_cpu == cpu) {
        /* 指定された1つのCPU上でjiffiesを更新する */
        do_timer(1);           /* jiffies++ */
        update_wall_time();    /* CLOCK_REALTIMEを更新する */
    }

    /* CPUごと: 時間を会計し、プリエンプションをチェックする */
    update_process_times(user_mode(get_irq_regs()));
}

void update_process_times(int user_tick)
{
    struct task_struct *p = current;

    /* 現在のタスクにCPU時間を計上する */
    account_process_tick(p, user_tick);

    /* 満了したタイマーホイールのエントリを実行する */
    run_local_timers();

    /* タスクをプリエンプトすべきかチェックする */
    scheduler_tick();
}
```

## HZ: ティックの頻度 {#hz-the-tick-rate}

`HZ`はコンパイル時の定数である:

```c
/* ほとんどのx86カーネルでは250がデフォルト */
/* 100: サーバー（オーバーヘッドが小さい）   */
/* 250: デスクトップ（よい妥協点） */
/* 1000: 最低のレイテンシ（1msのタイマー分解能） */

CONFIG_HZ_100  → HZ = 100  (10ms tick)
CONFIG_HZ_250  → HZ = 250  (4ms tick)
CONFIG_HZ_300  → HZ = 300
CONFIG_HZ_1000 → HZ = 1000 (1ms tick)

/* HZ=1000での実用的なタイマー分解能: 約1ms */
/* NOHZ+hrtimerを使えば: HZにかかわらずミリ秒未満 */
```

```bash
# 設定されたHZを確認する
zcat /proc/config.gz | grep "^CONFIG_HZ="
# CONFIG_HZ=250

# 現在のjiffies
cat /proc/uptime   # ブートからの秒数 = jiffies/HZ
```

## jiffies: 粗い粒度の時間 {#jiffies-coarse-grained-time}

`jiffies`は、ティックごとにインクリメントされる64ビットのカウンタである:

```c
/* include/linux/jiffies.h */
extern unsigned long volatile jiffies;

/* jiffiesと時間の変換: */
unsigned long timeout = jiffies + msecs_to_jiffies(100);  /* 100ms */
unsigned long timeout = jiffies + HZ / 10;                 /* 100ms（HZ=1000） */
unsigned long timeout = jiffies + HZ * 2;                  /* 2秒 */

/* 比較（ラップアラウンドを扱う）: */
if (time_after(jiffies, timeout))
    /* タイムアウトが満了した */
if (time_before(jiffies, deadline))
    /* デッドラインにまだ達していない */

/* jiffies_to_msecs: */
unsigned long elapsed_ms = jiffies_to_msecs(jiffies - start);
```

`jiffies`は`ULONG_MAX / HZ`秒（32ビットでは約497日、64ビットでは無関係）でラップアラウンドする。`time_after`/`time_before`マクロは引き算を使ってラップアラウンドを正しく扱う。

## scheduler_tick: プリエンプションのチェック {#scheduler_tick-preemption-check}

```c
/* kernel/sched/core.c */
void scheduler_tick(void)
{
    int cpu = smp_processor_id();
    struct rq *rq = cpu_rq(cpu);
    struct task_struct *curr = rq->curr;

    /* スケジューリングクラスに状態を更新させる */
    curr->sched_class->task_tick(rq, curr, 0);
    /* CFSの場合: vruntimeを更新し、currentが譲るべきかチェックする */
    /* RTの場合: RRタイムスライスの満了をチェックする */

    /* ティックごとに負荷分散を起動する */
    trigger_load_balance(rq);

    /* スケジューラの統計を更新する */
    rq->idle_stamp = 0;
}

/* CFSのティック: */
static void task_tick_fair(struct rq *rq, struct task_struct *curr, int queued)
{
    struct cfs_rq *cfs_rq;
    struct sched_entity *se = &curr->se;

    for_each_sched_entity(se) {
        cfs_rq = cfs_rq_of(se);
        entity_tick(cfs_rq, se, queued);
    }
}

static void entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr, int queued)
{
    /* vruntimeを更新する（CPU時間の会計） */
    update_curr(cfs_rq);

    /* 負荷平均を更新する（PELT） */
    update_load_avg(cfs_rq, curr, UPDATE_TG);

    /* 現在のタスクをプリエンプトすべきかチェックする */
    if (cfs_rq->nr_running > 1)
        check_preempt_tick(cfs_rq, curr);
}

static void check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
    unsigned long ideal_runtime;
    struct sched_entity *se;

    /* このタスクはどれだけ実行すべきか？（CPUの公平な取り分） */
    ideal_runtime = sched_slice(cfs_rq, curr);

    /* 実際にどれだけ実行したか？ */
    delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;

    if (delta_exec > ideal_runtime) {
        /* タスクがタイムスライスを超えた: TIF_NEED_RESCHEDをセットする */
        resched_curr(rq_of(cfs_rq));
    }
}
```

## NOHZ idle: アイドル時にティックを止める {#nohz-idle-stopping-the-tick-when-idle}

CPUがアイドルのときは、ティック割り込みを発生させる必要はない。**NOHZ idle**（CONFIG_NO_HZ_IDLE、デフォルトで有効）は、CPUがアイドルのときにティックを抑止する:

```
Normal (HZ=250):             NOHZ idle:
CPU0 idle: tick every 4ms    CPU0 idle: tick suppressed
                              Next tick: when timer expires or task wakes

Power savings: significant on idle systems
```

```c
/* kernel/time/tick-sched.c */
void tick_nohz_idle_enter(void)
{
    /* CPUがアイドルに入るときに呼ばれる */
    struct tick_sched *ts = this_cpu_ptr(&tick_cpu_sched);

    ts->inidle = 1;
    /* タイマーホイールから次の起床時刻を計算する */
    /* その時刻に合わせてクロックイベントデバイスをプログラムする */
    /* 起床時刻まで周期的なティックはもう発生しない */
}

void tick_nohz_idle_exit(void)
{
    /* CPUがアイドルから抜けるときに呼ばれる */
    /* スリープしていた時間の分だけjiffiesを追いつかせる */
    /* 周期的なティックを復元する */
}
```

## NOHZ full: 実行中のタスクのためのティックレス {#nohz-full-tickless-for-running-tasks}

**NOHZ full**（CONFIG_NO_HZ_FULL）はさらに踏み込み、CPU上で単一のタスクが実行されているときでもティックを抑止する:

```bash
# ブートパラメータ: CPU 1,2,3を完全にティックレスにする
GRUB_CMDLINE_LINUX="nohz_full=1-3 rcu_nocbs=1-3"

# CPU 1-3への影響:
# - ちょうど1つのタスクが実行されているときは周期的なティックがない
# - jiffiesの更新はCPU 0に委譲される
# - RCUコールバックはオフロードされる
# - 用途: HPC、リアルタイム、レイテンシがクリティカルなワークロード
```

```c
/* tick-sched.c: NOHZ fullのロジック */
static bool can_stop_full_tick(int cpu, struct tick_sched *ts)
{
    /* ちょうど1つのタスクが実行されているときだけティックレスにする */
    if (rq->nr_running > 1)
        return false;

    /* RCUがコールバックのためにティックを必要としていてはならない */
    if (rcu_needs_cpu())
        return false;

    /* ティックを必要とするperfイベントがないこと */
    if (perf_event_task_tick())
        return false;

    return true;
}
```

NOHZ fullのCPUは、次のような場合にCPU 0からティックの「キック」を受け取る:

- 別のタスクがランキューに追加されたとき
- RCUが静止状態（quiescent state）を必要とするとき
- タイマーが満了したとき

## ティックデバイスとclockevents {#tick-device-and-clockevents}

ティックは**クロックイベントデバイス**によって駆動される。これは特定の時刻に発火するようプログラムできるハードウェアタイマーである:

```c
/* kernel/time/clockevents.c */
struct clock_event_device {
    void   (*event_handler)(struct clock_event_device *);
    int    (*set_next_event)(unsigned long evt, struct clock_event_device *);
    int    (*set_next_ktime)(ktime_t expires, struct clock_event_device *);
    ktime_t next_event;
    u64     max_delta_ns;
    u64     min_delta_ns;
    u32     mult;     /* ns変換 */
    u32     shift;
    enum clock_event_state state_use_accessors;
    unsigned int features;
    /* ... */
};

/* CPUごと: 各CPUは自分のティックデバイスを持つ */
DEFINE_PER_CPU(struct tick_device, tick_cpu_device);
```

x86では:

- BSP（ブートCPU）: ティックにTSCまたはHPETを使う
- その他のCPU: ローカルAPICタイマーを使う
- すべてのCPU: ワンショットモードに切り替えられる（NOHZ用）

```bash
# クロックイベントデバイスを確認する
cat /sys/bus/clockevents/devices/*/current_mode
# periodic（HZモード）
# oneshot   （NOHZモード）

cat /sys/bus/clockevents/devices/*/name
# lapic（ローカルAPICタイマー）
# tsc-deadline（デッドラインモードのTSC — 最も効率的）
```

## 会計のオーバーヘッド {#accounting-overhead}

```bash
# スケジューラティックの統計を確認する
cat /proc/sched_debug | grep -E "\.nr_switches|\.clock|avg_idle"

# プロファイリング: scheduler_tickにどれだけ時間が費やされているか？
perf stat -e cycles:k -C 0 sleep 1 | grep cycles
# ほとんどはアイドルのはず（NOHZ_IDLEがティックを抑止する）

# タイマー割り込みを数える:
grep "^LOC:" /proc/interrupts  # "Local timer interrupts"
```

## 参考文献 {#further-reading}

- [プリエンプションモデル](preemption.md) — TIF_NEED_RESCHEDとプリエンプションのコンテキスト
- [CFS](cfs.md) — vruntime、sched_slice、ideal_runtime
- [hrtimers](../time/hrtimers.md) — ティックを迂回する高分解能タイマー
- [タイマーとhrtimers](../interrupts/timers.md) — タイマーホイールとhrtimerの統合
- [Energy-Aware Scheduling](eas.md) — ティックと負荷追跡の相互作用
- `kernel/time/tick-sched.c` — NOHZの実装
- `kernel/sched/core.c` — scheduler_tick
- `kernel/sched/fair.c` — check_preempt_tick
