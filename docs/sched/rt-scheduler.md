# RTスケジューラ: SCHED_FIFOとSCHED_RR {#rt-scheduler-sched_fifo-and-sched_rr}

> 優先度ベースのプリエンプションによるハードリアルタイムスケジューリング

## RTスケジューラが提供するもの {#what-the-rt-scheduler-provides}

RTスケジューラ（`rt_sched_class`）は、タスクに厳格な優先度の保証を与える。優先度NのRTタスクは、どのfairクラスのタスクよりも、また優先度がNより低いどのRTタスクよりも、常に優先してプリエンプトする。異なる優先度のタスク間には、公平性も、vruntimeも、時分割もない。実行可能なタスクのうち最も優先度の高いものが実行される、ただそれだけである。

2つのポリシーがRTクラスを共有している:

| ポリシー | タイムスライス | プリエンプション |
|--------|-----------|------------|
| `SCHED_FIFO` | なし — ブロックするかyieldするまで実行される | より高い優先度のタスクによってのみ |
| `SCHED_RR` | 100ms（デフォルト） | より高い優先度のタスクによって、またはスライスが満了したときに同じ優先度のタスクによって |

RTの優先度は1（最低）から99（最高）の範囲である。これはnice値の逆で、数値が大きいほど優先度が高い。

## RTランキュー {#the-rt-runqueue}

```c
// kernel/sched/sched.h
struct rt_prio_array {
    DECLARE_BITMAP(bitmap, MAX_RT_PRIO + 1); // 101ビットのビットマップ（+1は区切り用）
    struct list_head queue[MAX_RT_PRIO];     // 優先度ごとの100個のFIFOキュー
};

struct rt_rq {
    struct rt_prio_array active;    // 優先度ごとの実行可能なタスク
    unsigned int         rt_nr_running;  // 実行可能なRTタスクの総数
    int                  rt_queued;      // このrqはオーバーロードリストに載っているか？
    // （CONFIG_RT_GROUP_SCHEDのときはスロットリング用のフィールド）
};
```

`MAX_RT_PRIO = 100`（`include/linux/sched/prio.h`で定義）。ビットマップは優先度レベルごとに1ビットを持つ。優先度Pのタスクが実行可能になると、ビットPがセットされる。タスクの選択は次のとおりである:

```c
int idx = sched_find_first_bit(rt_rq->active.bitmap);
// idxは空でないキューのうち最も優先度が高いもの
next = list_first_entry(&rt_rq->active.queue[idx], ...);
```

これはRTタスクがいくつあってもO(1)である。ビットマップの走査は1つのハードウェア命令（x86では`bsf`/`bsr`）で済む。

## struct sched_rt_entity {#struct-sched_rt_entity}

すべてのRTタスクは、`task_struct`に埋め込まれた`sched_rt_entity`を持つ:

```c
// include/linux/sched.h
struct sched_rt_entity {
    struct list_head    run_list;       // 優先度キュー内のノード
    unsigned long       timeout;        // RLIMIT_RTTIMEのウォッチドッグ
    unsigned long       watchdog_stamp; // 最後にチェックしたときのjiffies
    unsigned int        time_slice;     // RRタイムスライスの残り
    unsigned short      on_rq;          // 現在キューに入っているか？
    unsigned short      on_list;        // アクティブな優先度リストに載っているか？
    // （CONFIG_RT_GROUP_SCHEDのときはグループスケジューリング用のフィールド）
};
```

## タスクの選択: pick_next_task_rt() {#task-selection-pick_next_task_rt}

```c
// kernel/sched/rt.c
static struct task_struct *pick_task_rt(struct rq *rq)
{
    struct sched_rt_entity *rt_se = pick_next_rt_entity(&rq->rt);
    if (!rt_se)
        return NULL;
    return rt_task_of(rt_se);
}
```

`pick_next_rt_entity()`は優先度ビットマップで最初にセットされたビットを見つけ、そのキューの先頭を返す。同じ優先度のSCHED_RRタスクについては、キューはFIFOである。先頭のタスクは最も長い間実行されていない（最も長く待っている）タスクであり、スライスを使い終えると末尾に回る。

## SCHED_RRのタイムスライスのローテーション {#sched_rr-timeslice-rotation}

タイマーティックごとに`task_tick_rt()`が実行される:

```c
// kernel/sched/rt.c
static void task_tick_rt(struct rq *rq, struct task_struct *p, int queued)
{
    struct sched_rt_entity *rt_se = &p->rt;

    // ウォッチドッグ: RLIMIT_RTTIMEを超えたらタスクをkillする
    watchdog(rq, p);

    // タイムスライスを使うのはSCHED_RRだけ
    if (p->policy != SCHED_RR)
        return;

    if (--rt_se->time_slice)
        return;   // まだ時間が残っている

    // タイムスライスが満了: リセットして末尾に再キューする
    rt_se->time_slice = sched_rr_timeslice;

    if (rt_se->run_list.prev != rt_se->run_list.next) {
        // 同じ優先度の他のタスクがいる: キューの末尾に移動する
        requeue_task_rt(rq, p, 0);
        resched_curr(rq);
    }
}
```

SCHED_RRのデフォルトのタイムスライスは100msである:
```c
// include/linux/sched/rt.h
#define RR_TIMESLICE    (100 * HZ / 1000)

// sysctlで設定可能:
// /proc/sys/kernel/sched_rr_timeslice_ms（デフォルト: 100）
int sched_rr_timeslice = RR_TIMESLICE;
```

## RTスロットリング {#rt-throttling}

決してスリープしないRTタスクは、すべての通常タスクを飢餓状態に陥らせる。カーネルは**RTスロットリング**によってこれを防ぐ:

```c
// kernel/sched/rt.c
int sysctl_sched_rt_period  = 1000000;  // 1秒（µs）
int sysctl_sched_rt_runtime =  950000;  // 950ms（µs）
```

RTタスクは合計で最大`rt_runtime / rt_period` = CPU時間の95%までしか使えない。残りの5%はSCHED_NORMALタスク（シェルを含むので、暴走したRTタスクを`kill -9`できる）のために確保されている。

```bash
# RTスロットリングの設定を表示/変更する
cat /proc/sys/kernel/sched_rt_period_us    # デフォルト: 1000000（1秒）
cat /proc/sys/kernel/sched_rt_runtime_us   # デフォルト: 950000（950ms）

# スロットリングを無効にする（危険 — システムがフリーズしうる）
echo -1 > /proc/sys/kernel/sched_rt_runtime_us
```

`sched_rt_runtime_us`を-1に設定すると、スロットリングが完全に無効になる。RTタスクを完全に信頼できる場合にのみ行うこと。

## RTポリシーの設定 {#setting-rt-policy}

```c
// 優先度50でSCHED_FIFOを設定する
struct sched_param param = { .sched_priority = 50 };
sched_setscheduler(pid, SCHED_FIFO, &param);

// あるいはsched_setattrを使う（より柔軟）
struct sched_attr attr = {
    .sched_policy   = SCHED_FIFO,
    .sched_priority = 50,
};
sched_setattr(0, &attr, 0);
```

`CAP_SYS_NICE`（またはrootとしての実行）が必要である。非特権ユーザーはRTポリシーを設定できない。

```bash
# プロセスをSCHED_FIFO優先度80に設定する
chrt -f 80 ./my_realtime_app

# 既存のプロセスに設定する
chrt -f -p 80 $PID

# スケジューリングポリシーを確認する
chrt -p $PID
```

## SMPでの考慮事項: pushableタスク {#smp-considerations-pushable-tasks}

マルチCPUシステムでは、RTスケジューラは**pushableタスク**のリストを管理する。これは、現在のCPUがより高い優先度のRTタスクに占有されている場合に、別のCPUへマイグレーションできるRTタスクである。

すでにRTタスクを実行しているCPUに、より優先度の高いRTタスクが到着すると、優先度の低いタスクは別のCPUへ「プッシュ」される:

```
Before:                          After:
CPU 0: RT-90 (running)          CPU 0: RT-90 (running)
CPU 1: RT-70 (running)          CPU 1: RT-80 (running, migrated here)
                                CPU 1: RT-70 → pushed to CPU 2
CPU 2: normal task              CPU 2: RT-70 (migrated)
```

`push_rt_task()`関数と`pull_rt_task()`関数がこのマイグレーションを処理する。

## 実践的なパターン {#practical-patterns}

### オーディオ/ビデオ処理 {#audiovideo-processing}

```c
// リアルタイムのオーディオスレッド: SCHED_FIFO、優先度は約70
// すべての通常タスクをプリエンプトするには十分に高く、
// 必要ならSCHED_DEADLINEタスクがプリエンプトできる程度に低い
struct sched_param param = { .sched_priority = 70 };
sched_setscheduler(0, SCHED_FIFO, &param);

// タイトなループでオーディオを処理し、条件変数でスリープする
while (running) {
    pthread_mutex_lock(&buf_lock);
    while (buf_empty)
        pthread_cond_wait(&buf_cond, &buf_lock);
    process_audio();
    pthread_mutex_unlock(&buf_lock);
}
```

### PIの問題を避ける {#avoiding-pi-issues}

RTタスクでmutexを使うときは、`pthread_mutexattr_setprotocol(PTHREAD_PRIO_INHERIT)`を使って優先度継承を有効にする:

```c
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
pthread_mutex_init(&mutex, &attr);
```

PIがないと、mutexを保持している低優先度のタスクが、高優先度のRTタスクを無期限にブロックしうる。これが古典的な優先度逆転の問題である。[優先度逆転とPI mutex](pi-mutexes.md)を参照のこと。

## 参考文献 {#further-reading}

- [SCHED_DEADLINE](deadline.md) — 帯域の強制によるさらに厳格な保証
- [優先度逆転とPI mutex](pi-mutexes.md) — RTにおけるロックの問題とその解決策
- [スケジューラクラス](scheduler-classes.md) — クラス階層におけるRTクラス
