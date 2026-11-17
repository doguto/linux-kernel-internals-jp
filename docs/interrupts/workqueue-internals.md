# Workqueueの内部 {#workqueue-internals}

> cmwqのアーキテクチャ、ワーカープール、ワークアイテムのライフサイクル、NUMAアフィニティ

## なぜworkqueueが存在するのか {#why-workqueues-exist}

割り込みハンドラ（hardirqとsoftirq）はアトミックに動作し、スリープもブロックもできない。スリープしたり、mutexを取得したり、I/Oを行ったりする必要がある処理は遅延させなければならない。workqueueは、こうした遅延処理をプロセスコンテキストで実行できるカーネルスレッドのプールを提供する。

```
Interrupt fires:
  hardirq handler → schedule_work(&my_work)   ← non-sleeping, fast
                         │
                         ▼
                    [kworker thread]            ← sleeps OK
                    my_work_func()
                    → can mutex_lock(), kmalloc(GFP_KERNEL), etc.
```

## 並行性管理workqueue（cmwq） {#concurrency-managed-workqueues-cmwq}

Linux 2.6.36以降、workqueueは**cmwq**を使っている。これはTejun Heoが導入したもので[(LWN)](https://lwn.net/Articles/394084/)、並行性を自動的に管理する統一されたワーカープールである。

```
Before cmwq (per-CPU dedicated threads):
  cpu0: kblockd/0, kworker/0:0, kworker/0:1, ...
  cpu1: kblockd/1, kworker/1:0, kworker/1:1, ...
  Problem: each workqueue had its own per-CPU thread → thread explosion

After cmwq (shared worker pool):
  cpu0: [shared pool] kworker/0:0, kworker/0:1, kworker/0:2H (high-prio)
  cpu1: [shared pool] kworker/1:0, kworker/1:1, kworker/1:2H
  All workqueues share the pool → far fewer threads
```

### ワーカープールの種類 {#worker-pool-types}

```c
/* kernel/workqueue.c */

/* per-CPUのワーカープール（CPUごとに2つ: 通常と高優先度）: */
static DEFINE_PER_CPU_SHARED_ALIGNED(struct worker_pool [NR_STD_WORKER_POOLS],
                                     cpu_worker_pools);
/*
 * cpu_worker_pools[0]: 通常優先度（SCHED_NORMAL, nice=0）
 * cpu_worker_pools[1]: 高優先度  （SCHED_NORMAL, nice=-20）
 */

/* アンバウンドなワーカープール: どのCPUにも縛られず、WQ_UNBOUNDのworkqueue用 */
static DEFINE_HASHTABLE(unbound_pool_hash, UNBOUND_POOL_HASH_ORDER);
```

## workqueueの作成 {#creating-workqueues}

```c
#include <linux/workqueue.h>

/* システムworkqueue（こちらを優先する）: */
system_wq             /* 汎用、スリープ可能 */
system_highpri_wq     /* 高優先度、時間的にクリティカルなワーク */
system_long_wq        /* 長時間実行されうるワーク用 */
system_unbound_wq     /* 特定のCPUに束縛されない */
system_freezable_wq   /* システムのフリーズ中は停止する */
system_power_efficient_wq  /* 低電力のCPUを好む */

/* ほとんどの場合はsystem_wqを使う: */
schedule_work(&my_work);  /* system_wqに投入する */

/* 独自のworkqueue: */
struct workqueue_struct *wq;

/* CPUにバウンド（ワークは投入元のCPUで実行される）: */
wq = alloc_workqueue("my-wq", WQ_MEM_RECLAIM, 1);

/* アンバウンド（任意のCPUで実行でき、NUMAに適する）: */
wq = alloc_workqueue("my-unbound-wq", WQ_UNBOUND, 0);

/* 高優先度 + アンバウンド: */
wq = alloc_workqueue("my-hipri-wq", WQ_UNBOUND | WQ_HIGHPRI, 0);

/* フラグ: */
WQ_UNBOUND          /* 特定のCPUに束縛されない */
WQ_FREEZABLE        /* システムのサスペンド中にフリーズする */
WQ_MEM_RECLAIM      /* メモリ回収中に必要になりうる（レスキュースレッドを予約する） */
WQ_HIGHPRI          /* 高優先度のワーカープール */
WQ_CPU_INTENSIVE    /* CPU負荷の高いワーク: プールの並行数カウントから除外され、
                       他の保留中のワークがそれと並んで開始できる */
WQ_SYSFS            /* workqueueの属性を/sys/bus/workqueueに公開する */

/* 最後の引数: max_active = CPUごとの最大同時実行ワークアイテム数（0 = デフォルト=256） */
```

## ワークアイテム {#work-items}

```c
/* ワークを宣言して初期化する: */
DECLARE_WORK(my_work, my_work_func);
/* または: */
struct work_struct my_work;
INIT_WORK(&my_work, my_work_func);

/* ワーク関数: */
static void my_work_func(struct work_struct *work)
{
    struct my_device *dev = container_of(work, struct my_device, work);
    /* dev->dataを処理する。ここではスリープできる */
    mutex_lock(&dev->lock);
    process_data(dev);
    mutex_unlock(&dev->lock);
}

/* ワークを投入する: */
schedule_work(&my_work);          /* system_wqの現在のCPUへ */
queue_work(wq, &my_work);         /* 独自のwqへ */
queue_work_on(cpu, wq, &my_work); /* 特定のCPUのプールへ */

/* ワークの完了を待つ: */
flush_work(&my_work);             /* このワークアイテムが終わるまでブロックする */
flush_workqueue(wq);              /* wq内のすべてのアイテムが終わるまでブロックする */
```

## 遅延ワーク {#delayed-work}

```c
struct delayed_work my_dwork;
INIT_DELAYED_WORK(&my_dwork, my_dwork_func);

/* 100ms後に実行するようスケジュールする: */
schedule_delayed_work(&my_dwork, msecs_to_jiffies(100));
queue_delayed_work(wq, &my_dwork, msecs_to_jiffies(100));

/* キャンセルする（すでに実行中ならキャンセルできるとは限らない）: */
cancel_delayed_work(&my_dwork);

/* キャンセルして完了を待つ: */
cancel_delayed_work_sync(&my_dwork);
```

## struct work_structの内部 {#struct-work_struct-internals}

```c
struct work_struct {
    atomic_long_t   data;       /* プールへのポインタ + フラグをエンコードする */
    struct list_head entry;     /* pool->worklist内 */
    work_func_t     func;
#ifdef CONFIG_LOCKDEP
    struct lockdep_map lockdep_map;
#endif
};

/* dataフィールドのエンコード: */
/* ビット[0]: WORK_STRUCT_PENDING — ワークがキューに積まれている */
/* ビット[1]: WORK_STRUCT_INACTIVE — ワークが非アクティブリストにある */
/* ビット[2]: WORK_STRUCT_PWQ — 下位ビットがpool_workqueueを指す */
/* ビット[WORK_STRUCT_FLAG_BITS..]: pool_workqueueまたは最後のプールへのポインタ */
```

## ワーカープールと並行性管理 {#worker-pool-and-concurrency-management}

```c
struct worker_pool {
    spinlock_t          lock;
    int                 cpu;        /* CPU、アンバウンドなら-1 */
    int                 node;       /* NUMAノード */
    int                 id;
    unsigned int        flags;

    unsigned long       watchdog_ts;
    bool                cpu_stall;

    struct list_head    worklist;   /* 保留中のワークアイテム */
    int                 nr_workers;
    int                 nr_idle;

    struct list_head    idle_list;  /* アイドルなワーカー */
    struct timer_list   idle_timer;
    struct work_struct  idle_cull_work;

    struct timer_list   mayday_timer; /* 安全弁 */
    DECLARE_HASHTABLE(busy_hash, BUSY_WORKER_HASH_ORDER);

    struct worker       *manager;   /* プールを管理するスレッド */
    struct list_head    workers;    /* プール内のすべてのワーカー */
    struct completion   *detach_completion;

    struct ida          worker_ida;
    struct workqueue_attrs *attrs;
    struct hlist_node   hash_node;
    int                 refcnt;
    struct rcu_head     rcu;
};
```

### 並行性管理のアルゴリズム {#concurrency-management-algorithm}

cmwqは、CPUを忙しく保つのにちょうど足りるだけのワーカーを維持する。

```
Rule: pool tries to keep at least one runnable worker
  (one that is executing work AND not sleeping)

When a worker goes to sleep (blocks on mutex, I/O, etc.):
  → pool checks: is there another runnable worker?
  → if not: wake an idle worker (or create a new one)
  → prevents CPU stall while work is blocked

When a worker wakes up:
  → pool checks: too many runnable workers?
  → if yes: go back to sleep (idle)

Result: CPU always busy, minimal thread creation
```

```c
/* kernel/workqueue.c */
static void worker_enter_idle(struct worker *worker)
{
    struct worker_pool *pool = worker->pool;

    /* ... */
    pool->nr_idle++;
    worker->last_active = jiffies;
    worker->flags |= WORKER_IDLE;
    list_add(&worker->entry, &pool->idle_list);

    /* 5分間アイドルが続いたワーカーを破棄する */
    if (too_many_workers(pool))
        mod_timer(&pool->idle_timer, jiffies + IDLE_WORKER_TIMEOUT);
}

static bool keep_working(struct worker_pool *pool)
{
    return !list_empty(&pool->worklist) &&
           atomic_read(&pool->nr_running) <= 1;
}
```

## workqueueの属性とNUMAアフィニティ {#workqueue-attributes-and-numa-affinity}

```c
/* workqueueの属性を取得/設定する（WQ_UNBOUNDのみ）: */
struct workqueue_attrs *attrs = alloc_workqueue_attrs();
attrs->nice = -5;           /* ワーカースレッドのnice値 */
attrs->cpumask = cpu_mask;  /* ワーカーが動作できるCPU */
attrs->no_numa = false;     /* NUMAの局所性を尊重する */

apply_workqueue_attrs(wq, attrs);
free_workqueue_attrs(attrs);

/* NUMAを考慮したアンバウンドプール:
   デフォルトでは、アンバウンドなworkqueueはNUMAノードごとのプールを持つ。
   ノード0から投入されたワークはノード0のワーカーで実行される。
   これによりキャッシュがホットなデータが同じノードに留まる。 */
```

## workqueueのデバッグ {#workqueue-debugging}

```bash
# すべてのworkqueueとその統計を表示する:
cat /sys/kernel/debug/workqueue/stats

# ワーカースレッドを表示する:
ps aux | grep kworker
# kworker/0:1   — CPU 0にバウンド、プール1（通常）
# kworker/0:1H  — CPU 0にバウンド、プールH（高優先度）
# kworker/u8:2  — アンバウンドプール、プールID=8、ワーカーID=2

# workqueueごとの統計を表示する（WQ_SYSFSの場合）:
ls /sys/bus/workqueue/devices/
cat /sys/bus/workqueue/devices/system_wq/per_cpu

# SysRqですべてのタスクのスタック（kworkerスレッドを含む）をダンプする:
echo t > /proc/sysrq-trigger
# dmesgにすべてのスレッドが表示される。workqueue固有の状態（保留数、キューの深さ）には
# /sys/kernel/debug/workqueueを使う

# ワークの投入と実行をトレースする:
bpftrace -e '
kprobe:queue_work_on {
    /* queue_work_on(int cpu, struct workqueue_struct *wq, struct work_struct *work) */
    printf("queue_work: cpu=%d func=%s\n", (int)arg0,
           ksym(((struct work_struct *)arg2)->func));
}
kprobe:process_one_work {
    printf("exec_work: cpu=%d pid=%d func=%s\n",
           cpu, pid, ksym(((struct work_struct *)arg1)->func));
}'

# ウォッチドッグ: ハングしたワーカーを検出する（CONFIG_WQ_WATCHDOG=y）
# ワーカープールが30秒を超えてストールすると、カーネルが警告を出す:
# "BUG: workqueue lockup"
# 調整:
echo 60 > /sys/module/workqueue/parameters/watchdog_thresh
```

## WQ_MEM_RECLAIMとレスキューワーカー {#wqmemreclaim-and-rescue-workers}

```c
/*
 * メモリ回収中に使われるワーク（例: ライトバック）は必ずWQ_MEM_RECLAIMを使わなければならない。
 * これにより、メモリを待ってブロックされることのない専用のレスキューワーカーが
 * 存在することが保証される（デッドロックを回避する）:
 *
 * メモリ回収のパス:
 *   1. kswapdがダーティページを書き出す必要がある → ワークをキューに積む
 *   2. 通常のワーカーはメモリを待ってスリープしているかもしれない
 *   3. レスキューワーカーがそのワークを拾う — 常に1スレッドが予約されている
 */
wq = alloc_workqueue("writeback", WQ_MEM_RECLAIM | WQ_UNBOUND, 1);
```

## 参考文献 {#further-reading}

- [Workqueue](workqueues.md) — APIリファレンスと基本的な使い方
- [Softirqとtasklet](softirq.md) — より軽量な遅延（スリープなし）
- [スレッド化IRQ](threaded-irq.md) — スレッドコンテキストのIRQハンドラ
- [RCU](../locking/rcu.md) — ロックフリーな遅延処理
- `kernel/workqueue.c` — 完全な実装（約6000行）
- `Documentation/core-api/workqueue.rst` — カーネルのドキュメント
