# ランキューとタスク選択 {#runqueues-and-task-selection}

> CPUごとのランキュー、struct rq、そしてスケジューラが次のタスクを選ぶ仕組み

## CPUごとの設計 {#the-per-cpu-design}

各CPUはそれぞれ独自のランキューを持つ。実行可能タスクのグローバルなキューは存在しない — 各CPUは自分のタスク集合を独立して管理し、そこから選択する。この設計はスケールする: 次のタスクを選ぶためだけにCPUが共有ロックを奪い合うことは決してない。

```
CPU 0             CPU 1             CPU 2             CPU 3
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│ struct rq│     │ struct rq│     │ struct rq│     │ struct rq│
│  cfs_rq  │     │  cfs_rq  │     │  cfs_rq  │     │  cfs_rq  │
│  rt_rq   │     │  rt_rq   │     │  rt_rq   │     │  rt_rq   │
│  dl_rq   │     │  dl_rq   │     │  dl_rq   │     │  dl_rq   │
│  curr ───┼─>T1 │  curr ───┼─>T4 │  curr ───┼─>T7 │  curr ───┼─>T9
└──────────┘     └──────────┘     └──────────┘     └──────────┘
```

負荷分散は、CPUを遊ばせないよう定期的にCPU間でタスクをマイグレーションするが、これはCPUごとの選択パスとは別のものである。

## struct rq {#struct-rq}

`struct rq`はCPUごとのスケジューラ状態の中心であり、`kernel/sched/sched.h`で定義されている:

```c
// kernel/sched/sched.h
struct rq {
    /* ランキューのロック */
    raw_spinlock_t      __lock;

    /* 実行可能タスク数（全クラス） */
    unsigned int        nr_running;

    /* 現在実行中のタスク */
    struct task_struct __rcu *curr;

    /* このCPUのアイドルスレッド */
    struct task_struct  *idle;

    /* stopタスク（マイグレーション/ホットプラグ） */
    struct task_struct  *stop;

    /* ランキューのクロック（ns） */
    u64                 clock;
    u64                 clock_task;

    /* クラスごとのサブキュー */
    struct cfs_rq       cfs;   // fair_sched_class
    struct rt_rq        rt;    // rt_sched_class
    struct dl_rq        dl;    // dl_sched_class

    /* このランキューが属するCPU */
    int                 cpu;

    /* 負荷の追跡 */
    struct sched_avg    avg_rt;
    struct sched_avg    avg_dl;

    /* ...（負荷分散、エネルギー、統計のためのフィールドが他に多数） */
};
```

### 主要なフィールド {#key-fields}

| フィールド | 型 | 目的 |
|-------|------|---------|
| `__lock` | `raw_spinlock_t` | ランキューの変更を保護する |
| `nr_running` | `unsigned int` | 全クラスを通じた実行可能タスクの総数 |
| `curr` | `struct task_struct *` | 現在実行中のタスク |
| `idle` | `struct task_struct *` | アイドルスレッド（swapper/N） |
| `clock` | `u64` | このCPUの単調増加ナノ秒クロック |
| `cfs` | `struct cfs_rq` | CFS/EEVDFランキュー（fairタスク） |
| `rt` | `struct rt_rq` | RTランキュー（FIFO/RRタスク） |
| `dl` | `struct dl_rq` | デッドラインランキュー（SCHED_DEADLINEタスク） |

## ランキューへのアクセス {#accessing-runqueues}

ランキューはCPUごとの変数として宣言されている:

```c
// kernel/sched/sched.h
DECLARE_PER_CPU_SHARED_ALIGNED(struct rq, runqueues);
```

アクセス用マクロ:

```c
// kernel/sched/sched.h
#define cpu_rq(cpu)    (&per_cpu(runqueues, (cpu)))
#define this_rq()      this_cpu_ptr(&runqueues)
#define task_rq(p)     cpu_rq(task_cpu(p))
```

スケジューラのコードの大半は`this_rq()` — 現在実行中のCPUのランキュー — を使う。別のCPUのランキューにアクセスするには、そのCPUの`rq->__lock`を保持する必要がある。

## 3つのサブキュー {#the-three-sub-queues}

### struct cfs_rq（fairタスク） {#struct-cfs_rq-fair-tasks}

```c
// kernel/sched/sched.h
struct cfs_rq {
    struct rb_root_cached tasks_timeline; // vruntime/deadlineをキーとするRBツリー
    struct sched_entity  *curr;           // 実行中のエンティティ
    struct sched_entity  *next;           // 次の候補（起床時のヒント）
    unsigned int          nr_queued;      // 実行可能タスク数
    s64                   avg_vruntime;   // 重み付きvruntimeの合計（適格性判定用）
    long                  avg_load;       // キュー内タスクの負荷重みの合計
    // ...
};
```

`tasks_timeline` RBツリーは、実行可能なすべてのfairタスクを`vruntime`順に保持する。`rb_root_cached`版はO(1)でアクセスできるよう最左ノードをキャッシュする。EEVDF以降は最左ノードが常に選ばれるわけではなくなったが、それでもキャッシュはツリー探索の出発点として有用である。

### struct rt_rq（RTタスク） {#struct-rt_rq-rt-tasks}

```c
// kernel/sched/sched.h
struct rt_rq {
    struct rt_prio_array active;    // 100段階の優先度ビットマップ + キュー
    unsigned int         rt_nr_running;
    int                  rt_queued; // このrqはオーバーロードリストに載っているか？
    // スロットリング関連のフィールド...
};

struct rt_prio_array {
    DECLARE_BITMAP(bitmap, MAX_RT_PRIO + 1); // 101ビット
    struct list_head queue[MAX_RT_PRIO];     // 優先度ごとのFIFOキュー
};
```

RTタスクの選択はO(1)である: `sched_find_first_bit(bitmap)`が空でない最高優先度のキューを見つけ、そのキューの先頭を取り出す。

### struct dl_rq（デッドラインタスク） {#struct-dl_rq-deadline-tasks}

```c
// kernel/sched/sched.h
struct dl_rq {
    struct rb_root_cached root;      // デッドライン順のRBツリー
    unsigned int          dl_nr_running;
    // CBSの帯域追跡フィールド...
};
```

デッドラインタスクの選択では、RBツリーから最も早い`dl_deadline`を持つタスクを選ぶ — EEVDFに似ているが、仮想デッドラインではなく絶対デッドラインを用いる。

## スケジューリングの決定 {#the-scheduling-decision}

スケジューラの中核は`kernel/sched/core.c`の`__schedule()`にある:

```c
// kernel/sched/core.c（簡略化）
static void __sched notrace __schedule(int sched_mode)
{
    struct task_struct *prev, *next;
    struct rq *rq;

    rq = this_rq();
    prev = rq->curr;

    // ランキューのクロックを更新する
    update_rq_clock(rq);

    // prevがスリープしようとしているなら、デキューする
    if (!(sched_mode & SM_MASK_PREEMPT) && prev->__state) {
        // タスクはブロックしようとしている
        deactivate_task(rq, prev, DEQUEUE_SLEEP | DEQUEUE_NOCLOCK);
    }

    // 次のタスクを選ぶ（sched_classの階層をたどる）
    next = pick_next_task(rq, prev, &rf);

    // 同じタスクならスイッチは不要
    if (likely(prev != next)) {
        rq->nr_switches++;
        rq->curr = next;
        context_switch(rq, prev, next, &rf);
    }
}
```

### pick_next_task() {#pick_next_task}

いずれかのクラスがタスクを持つまで、スケジューラクラスの階層をたどる:

```c
// kernel/sched/core.c（簡略化）
static struct task_struct *
pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
    const struct sched_class *class;
    struct task_struct *p;

    // 高速パス: すべてのタスクがfairクラス
    if (likely(!sched_class_above(prev->sched_class, &fair_sched_class) &&
               rq->nr_running == rq->cfs.h_nr_running)) {
        p = pick_next_task_fair(rq, prev, rf);
        if (unlikely(p == RETRY_TASK))
            goto restart;
        return p;
    }

restart:
    // 一般パス: stop → deadline → rt → fair → idle
    for_each_class(class) {
        p = class->pick_next_task(rq, prev, rf);
        if (p)
            return p;
    }
    BUG(); /* idle_sched_classは常にアイドルスレッドを返す */
}
```

高速パスは、すべてのタスクがSCHED_NORMALである場合（ほとんどのシステムで一般的なケース）にクラスのたどりを完全に省略する。これにより、ほぼタスクを持つことのない3つの上位クラスの確認を省ける。

## context_switch() {#context_switch}

`next`が選ばれると、`context_switch()`が実行を切り替える:

```c
// kernel/sched/core.c
static __always_inline struct rq *
context_switch(struct rq *rq, struct task_struct *prev,
               struct task_struct *next, struct rq_flags *rf)
{
    prepare_task_switch(rq, prev, next);

    // 別プロセスならメモリマッピングを切り替える
    if (!next->mm) {
        // カーネルスレッド: prevのページテーブルを借りる
        next->active_mm = prev->active_mm;
        enter_lazy_tlb(prev->active_mm, next);
    } else {
        // ユーザープロセス: ページテーブルを切り替える
        switch_mm_irqs_off(prev->active_mm, next->mm, next);
    }

    // CPUレジスタとスタックを切り替える
    switch_to(prev, next, prev);
    barrier();

    return finish_task_switch(prev);
}
```

`switch_to()`はアーキテクチャ固有である。x86-64ではレジスタを保存/復元し、スタックポインタを切り替える。`switch_to()`の後、実行は`next`のコンテキストで継続する — `prev`は、`next`が最後にCPUを明け渡す前に実行されていたタスクとなる。

## ランキューのクロック {#runqueue-clock}

`rq->clock`はCPUごとのナノ秒精度の単調増加クロックであり、スケジューリングイベントのたびに更新される:

```c
// クロックを更新する（__scheduleの冒頭で呼ばれる）
update_rq_clock(rq);

// タスクはゲスト（VM）に奪われた時間を除外するrq_clock_task()を使う
u64 now = rq_clock_task(rq);
```

CFSはvruntimeの更新に（実時間ではなく）`rq_clock_task()`を使う。つまり、VMゲスト内で費やされた時間は、ホストのタスクのCPU消費としてカウントされない。

## ランキュー状態の観測 {#observing-runqueue-state}

```bash
# CPUごとの現在実行中または実行可能なタスク
cat /proc/schedstat | head -20

# 現時点でのCPUごとの実行可能タスク数
for cpu in /sys/devices/system/cpu/cpu*/; do
  echo -n "$(basename $cpu): "
  cat "$cpu/cpufreq/stats/time_in_state" 2>/dev/null | head -1
done

# リアルタイムのランキュー可視化
watch -n 0.5 'cat /proc/$(pgrep stress | head -1)/sched 2>/dev/null'

# schedstatからのCPUごとのタスク数
awk '/^cpu[0-9]/ {print $1, "running:", $2, "switches:", $8}' /proc/schedstat
```

## 参考文献 {#further-reading}

- [スケジューラクラス](scheduler-classes.md) — これらのキューにタスクを入れる5つのクラス
- [CFS](cfs.md) — `struct cfs_rq`とfairクラスの詳細
- [EEVDF](eevdf.md) — fairクラス内部で`pick_next_entity()`がどう動くか
- [コンテキストスイッチの一生](context-switch.md) — `__schedule()`を通る全経路
