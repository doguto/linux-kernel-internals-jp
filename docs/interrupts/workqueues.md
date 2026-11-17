# Workqueue {#workqueues}

> プロセスコンテキストで実行される遅延処理。ほとんどのドライバにとって正しい選択

## workqueueとは何か {#what-are-workqueues}

workqueueは、カーネルの遅延処理機構の中で最も柔軟なものである。softirqやtaskletと異なり、workqueueのハンドラは**プロセスコンテキスト**で動作する。スリープでき、mutexを取得でき、`GFP_KERNEL`でメモリ割り当てを行え、カーネルスレッドで有効なあらゆる操作を実行できる。

カーネルはワーカースレッドのプールを自動的に管理する。各ワークアイテムはキューに積まれ、ワーカーが空いたときに実行される。

## ワークアイテムと初期化 {#work-items-and-initialization}

```c
#include <linux/workqueue.h>

/* ワークアイテムとそのハンドラを定義する */
struct my_device {
    struct work_struct  work;      /* 1回限りのワーク */
    struct delayed_work dwork;     /* 遅延ワーク */
};

/* ワークハンドラ: プロセスコンテキストで動作する */
static void my_work_handler(struct work_struct *work)
{
    struct my_device *dev = container_of(work, struct my_device, work);
    /* スリープ、mutexの取得、GFP_KERNELでのメモリ割り当てが可能 */
    mutex_lock(&dev->lock);
    process_data(dev);
    mutex_unlock(&dev->lock);
}

static void my_delayed_work_handler(struct work_struct *work)
{
    struct delayed_work *dwork = to_delayed_work(work);
    struct my_device *dev = container_of(dwork, struct my_device, dwork);
    /* ... */
}

/* 初期化 */
INIT_WORK(&dev->work, my_work_handler);
INIT_DELAYED_WORK(&dev->dwork, my_delayed_work_handler);
```

## ワークのスケジューリング {#scheduling-work}

```c
/* システムworkqueueにキューイングする（ワーカーが空き次第実行） */
schedule_work(&dev->work);

/* 遅延付きでキューイングする */
schedule_delayed_work(&dev->dwork, msecs_to_jiffies(100));  /* 100msの遅延 */

/* 保留中の遅延ワークをキャンセルする */
cancel_delayed_work_sync(&dev->dwork);  /* 実行中のハンドラを待つ */
cancel_delayed_work(&dev->dwork);       /* 待たない */

/* 遅延なしのワークをキャンセルする */
cancel_work_sync(&dev->work);
```

`schedule_work()`は冪等である。ワークがすでに保留中なら、もう一度呼んでも何も起きない。ワークはちょうど1回だけ実行される。

## システムworkqueue {#system-workqueues}

カーネルは、優先度や挙動の異なる、あらかじめ用意された複数のworkqueueを提供している。

```c
/* include/linux/workqueue.h */
extern struct workqueue_struct *system_wq;           /* 汎用 */
extern struct workqueue_struct *system_highpri_wq;   /* 高優先度 */
extern struct workqueue_struct *system_long_wq;      /* 長時間実行されるワーク用 */
extern struct workqueue_struct *system_unbound_wq;   /* CPUに束縛されない */
extern struct workqueue_struct *system_freezable_wq; /* サスペンド時にフリーズ可能 */
extern struct workqueue_struct *system_power_efficient_wq; /* 省電力向けに最適化 */

/* 特定のworkqueueにキューイングする */
queue_work(system_highpri_wq, &dev->work);
queue_delayed_work(system_wq, &dev->dwork, delay);
```

ほとんどのドライバにとっては、`system_wq`（`schedule_work()`経由で使う）が正しい選択である。

## 専用workqueueの作成 {#creating-a-private-workqueue}

特定の並行性制御や、システムのワークからの分離が必要なワークの場合:

```c
/* alloc_workqueue(name, flags, max_active) */

/* バウンド: CPUごとに1ワーカー、同時実行は最大1アイテム */
wq = alloc_workqueue("my-driver", WQ_MEM_RECLAIM, 1);

/* アンバウンド: 特定のCPUに縛られず、CPU負荷の高いワークに向く */
wq = alloc_workqueue("my-driver-unbound", WQ_UNBOUND, 0);

/* 高優先度、アンバウンド */
wq = alloc_workqueue("my-driver-hp", WQ_UNBOUND | WQ_HIGHPRI, 0);

/* 専用workqueueにキューイングする */
queue_work(wq, &dev->work);

/* フラッシュ: 保留中のすべてのワークの完了を待つ */
flush_workqueue(wq);
drain_workqueue(wq);

/* 破棄 */
destroy_workqueue(wq);
```

## WQフラグ {#wq-flags}

| フラグ | 意味 |
|------|---------|
| `WQ_UNBOUND` | ワーカーは特定のCPUに束縛されず、ワークはどこでも実行される |
| `WQ_MEM_RECLAIM` | メモリ逼迫時のためにレスキューワーカーを予約する |
| `WQ_HIGHPRI` | ワーカーは高い優先度で動作する |
| `WQ_FREEZABLE` | システムのサスペンド中にワークがフリーズする |
| `WQ_SYSFS` | workqueueを`/sys/bus/workqueue/devices/`に公開する |

## 並行性管理workqueue（cmwq） {#concurrency-managed-workqueues-cmwq}

Linux 2.6.36以降、カーネルはTejun Heoが導入した**並行性管理workqueue**（cmwq）を使っている[(LWN)](https://lwn.net/Articles/394084/)。CPUごとに1スレッドを持つ代わりに、カーネルはワーカーのプールを保持し、需要に応じてスレッドを動的に作成・破棄する。

重要な性質は、**カーネルはCPUごとに実行可能なワーカーをちょうど1つに保とうとする**という点である。ワーカーが（I/Oやmutexを待って）スリープすると、カーネルはCPUを忙しく保つために別のワーカーを作成することがある。

```
Worker pool state:
  Worker A: running (doing work)
  Worker B: sleeping on mutex
                ↓
  Kernel creates Worker C to handle next queued work
  (because Worker B is blocked, Worker A is the only runner)

When Worker B wakes:
  Now two runnable workers → Worker B becomes idle
```

これにより、CPUの使用率低下（ワーカーがスリープしたとき）とスレッドの爆発的増加（cmwqはワーカーを賢く再利用する）の両方を避けられる。

## flush_workとcancel_work {#flushwork-vs-cancelwork}

```c
/* 特定のワークアイテムの完了を待つ */
flush_work(&dev->work);          /* ハンドラが完了するまでブロックする */
flush_delayed_work(&dev->dwork); /* 保留中なら遅延を取り消してから待つ */

/* キャンセルして待つ */
cancel_work_sync(&dev->work);         /* キャンセル + 実行中のハンドラを待つ */
cancel_delayed_work_sync(&dev->dwork); /* 遅延ワークをキャンセル + 待つ */
```

use-after-freeを防ぐため、ドライバの終了処理では常に`cancel_*_sync()`を使うこと。

```c
static void my_driver_remove(struct platform_device *pdev)
{
    struct my_device *dev = platform_get_drvdata(pdev);

    /* 重要: devを解放する前にキャンセルする */
    cancel_work_sync(&dev->work);
    cancel_delayed_work_sync(&dev->dwork);

    kfree(dev);
}
```

## workqueueの動作の観察 {#observing-workqueue-activity}

```bash
# workqueueとワーカースレッド数の一覧
cat /sys/bus/workqueue/devices/*/name 2>/dev/null

# ワーカースレッド
ps aux | grep kworker
# root  1234  0.0 [kworker/0:1-events]    ← CPU 0、events wq
# root  1235  0.0 [kworker/1:2-mm_percpu_wq]
# root  1236  0.0 [kworker/u8:0-writeback] ← アンバウンド

# workqueueの統計（CONFIG_WQ_WATCHDOGが有効な場合）
cat /proc/sys/kernel/watchdog_thresh
```

## 参考文献 {#further-reading}

- [Softirq](softirq.md) — より低レベルで、スリープしない遅延処理
- [Tasklet](tasklets.md) — より単純だが非推奨で、スリープしない
- [スレッド化IRQ](threaded-irq.md) — プロセスコンテキストのIRQハンドラ
- `Documentation/core-api/workqueue.rst` — workqueueの完全なドキュメント
