# completionと待ちキュー {#completions-and-wait-queues}

> ブロックする同期: イベントが起きるまでスリープする

## struct completion {#struct-completion}

`completion`は単純な一回限りの同期プリミティブである。一方のスレッドが待ち、もう一方が完了時に通知する。このよくあるパターンについては、`wait_queue_head_t`を直接使うよりも単純で安全である。

```c
#include <linux/completion.h>

/* 宣言と初期化 */
DECLARE_COMPLETION(my_completion);
/* または動的に: */
struct completion c;
init_completion(&c);

/* 待機側スレッド: 通知されるまでブロックする */
wait_for_completion(&c);
/* 通知後、実行はここから続く */

/* 通知側スレッド: 待機者を起床させる */
complete(&c);
/* complete()の後、待機者が1つ起床する */
```

### completionのバリエーション {#completion-variants}

```c
/* 時間制限付きの待機: タイムアウトで0、完了で1を返す */
unsigned long remaining = wait_for_completion_timeout(&c, HZ);
if (!remaining)
    pr_err("Timed out waiting!\n");

/* 割り込み可能な待機: シグナルが届くと-ERESTARTSYSを返す */
int ret = wait_for_completion_interruptible(&c);
if (ret)
    return ret;  /* 割り込まれた */

/* 割り込み可能かつタイムアウト付き */
long ret = wait_for_completion_interruptible_timeout(&c, timeout);
/* ret > 0: 完了した */
/* ret == 0: タイムアウトした */
/* ret < 0: 割り込まれた */

/* （1つだけでなく）すべての待機者を起床させる */
complete_all(&c);

/* すべての待機者が起床した後に再初期化する */
reinit_completion(&c);

/* ブロックせずにチェックする */
if (completion_done(&c)) {
    /* すでに通知済み */
}
```

### try_waitを使うcompletion {#completion-with-try_wait}

```c
/* ブロックしないチェック: すでに通知済みなら取得する */
if (try_wait_for_completion(&c)) {
    /* すでに完了していた — ブロックなし */
} else {
    /* まだ完了していない — 本来ならブロックしていた */
}
```

## struct wait_queue_head_t {#struct-waitqueuehead_t}

`wait_queue_head_t`は、`completion`の土台となる低レベルのプリミティブである。次をサポートする。

- 同じイベントに対する複数の待機者
- 独自の起床条件
- 排他的な起床と非排他的な起床
- `poll()/select()/epoll`との統合

```c
#include <linux/wait.h>

/* 宣言 */
DECLARE_WAIT_QUEUE_HEAD(my_wq);
/* または: */
wait_queue_head_t wq;
init_waitqueue_head(&wq);
```

## wait_event: 標準的なパターン {#wait_event-the-standard-pattern}

`wait_event`は、待機と条件チェックを組み合わせる。

```c
/* 待機者: 条件が真になるまでスリープする */
wait_event(wq, condition);
/* 条件は起床のたびに再評価される */

/* 割り込み可能版 */
int ret = wait_event_interruptible(wq, condition);
if (ret)
    return ret;  /* -ERESTARTSYS: シグナルを受信した */

/* タイムアウト付き */
long ret = wait_event_timeout(wq, condition, timeout_jiffies);
/* ret > 0: 条件が満たされた */
/* ret == 0: タイムアウトした */

/* 割り込み可能 + タイムアウト付き */
long ret = wait_event_interruptible_timeout(wq, condition, timeout);
```

**wake_up / wait_eventの約束事**:

```c
/* 通知側: 条件変数を変更してから起床させる */
WRITE_ONCE(data_ready, true);
wake_up(&wq);          /* 非排他的な待機者を1つ起床させる */
wake_up_all(&wq);      /* すべての待機者を起床させる */
wake_up_interruptible(&wq);  /* 割り込み可能な待機者のみ起床させる */
```

`wait_event`はループを使う。
```c
/* wait_eventの展開形（簡略化）: */
for (;;) {
    prepare_to_wait(&wq, &__wait, TASK_UNINTERRUPTIBLE);
    if (condition)
        break;
    schedule();   /* スリープする */
}
finish_wait(&wq, &__wait);
```

これは競合を正しく扱う。チェックと`schedule()`の間に条件が真になった場合、wake_upがタスクの状態をTASK_RUNNINGに戻しているはずである。

## 独自の待機: prepare_to_wait / finish_wait {#custom-wait-preparetowait-finish_wait}

より複雑なパターンの場合:

```c
DEFINE_WAIT(wait);

prepare_to_wait(&wq, &wait, TASK_INTERRUPTIBLE);
while (!condition) {
    if (signal_pending(current)) {
        /* シグナルを受信した */
        break;
    }
    /* スリープする前に、ここで保持しているロックを解放する */
    spin_unlock_irq(&lock);
    schedule();
    spin_lock_irq(&lock);
    /* ここで条件を再チェックする */
}
finish_wait(&wq, &wait);
```

## 排他的な待機者 {#exclusive-waiters}

排他的な待機者は、1つずつ起床させられる（thundering herdの防止）。

```c
/* 排他的な待機者として追加する（wake_upでのみ起床し、wake_up_allでは起床しない） */
prepare_to_wait_exclusive(&wq, &wait, TASK_INTERRUPTIBLE);

/* wake_up: 排他的な待機者をちょうど1つ + 非排他的な待機者をすべて起床させる */
wake_up(&wq);
```

TCPのlistenソケットに対するaccept()で使われている。接続が到着したとき、待機している多数のaccept()呼び出しのうち1つだけを起床させればよい。

## poll/select/epollとの統合 {#pollselectepoll-integration}

ドライバは、待ちキューを使って`poll`（file_operationsの`->poll`）を実装する。

```c
/* file_operations.pollまたは.f_poll内: */
static __poll_t mydev_poll(struct file *file, poll_table *wait)
{
    struct mydev *dev = file->private_data;
    __poll_t mask = 0;

    /* このファイルの待ちキューをpollテーブルに登録する */
    poll_wait(file, &dev->read_wq, wait);
    poll_wait(file, &dev->write_wq, wait);

    /* 現在の状態をチェックする */
    if (!kfifo_is_empty(&dev->recv_fifo))
        mask |= EPOLLIN | EPOLLRDNORM;   /* 読めるデータがある */
    if (!kfifo_is_full(&dev->send_fifo))
        mask |= EPOLLOUT | EPOLLWRNORM;  /* 書き込む空きがある */

    return mask;
}

/* データが到着したとき: 待ちキューを起床させる */
void mydev_data_arrived(struct mydev *dev)
{
    kfifo_put(&dev->recv_fifo, data);
    wake_up_interruptible(&dev->read_wq);
}
```

`poll_wait`は待ちキューをpollテーブルに追加し、epollがそれを監視できるようにする。`wake_up`が発火すると、epollは呼び出し元のプロセスを起床させる。

## wait_queue_entry: 独自のコールバック {#waitqueueentry-custom-callbacks}

高度な使い方として、起床時に呼ぶ独自の関数を追加できる。

```c
/* コールバック付きの独自の待ちエントリ */
static int my_wakeup_func(struct wait_queue_entry *curr,
                           unsigned mode, int wake_flags, void *key)
{
    struct mydata *data = container_of(curr, struct mydata, wait_entry);

    /* 起床させる側のコンテキスト（IRQまたはプロセス）で呼ばれる */
    /* 0を返すと他の待機者の起床を続け、1を返すと止める */

    /* 後で処理するためにタスクをスケジュールする */
    schedule_work(&data->work);
    return 1;  /* タスクを直接は起床させない */
}

struct wait_queue_entry wqe = {
    .private = &mydata,
    .func    = my_wakeup_func,
    .flags   = 0,  /* 排他的にするならWQ_FLAG_EXCLUSIVE */
};

add_wait_queue(&wq, &wqe);
/* ... */
remove_wait_queue(&wq, &wqe);
```

## よくあるパターン {#common-patterns}

### 待ちキューを使った生産者/消費者 {#producerconsumer-with-wait-queue}

```c
struct ring_buffer {
    u8               data[1024];
    unsigned int     head, tail;
    wait_queue_head_t not_empty;   /* 消費者はここで待つ */
    wait_queue_head_t not_full;    /* 生産者はここで待つ */
    spinlock_t        lock;
};

/* 消費者 */
int rb_read(struct ring_buffer *rb, u8 *buf, size_t len)
{
    int ret;

    spin_lock(&rb->lock);
    ret = wait_event_interruptible_lock_irq(rb->not_empty,
                                             !rb_is_empty(rb),
                                             rb->lock);
    if (ret) {
        spin_unlock(&rb->lock);
        return ret;  /* 割り込まれた */
    }

    memcpy(buf, &rb->data[rb->tail], len);
    rb->tail = (rb->tail + len) % sizeof(rb->data);
    spin_unlock(&rb->lock);

    wake_up(&rb->not_full);
    return len;
}

/* 生産者 */
int rb_write(struct ring_buffer *rb, const u8 *buf, size_t len)
{
    spin_lock(&rb->lock);
    wait_event_interruptible_lock_irq(rb->not_full, !rb_is_full(rb), rb->lock);
    memcpy(&rb->data[rb->head], buf, len);
    rb->head = (rb->head + len) % sizeof(rb->data);
    spin_unlock(&rb->lock);

    wake_up(&rb->not_empty);
    return len;
}
```

## 参考文献 {#further-reading}

- [SRCU](srcu.md) — 読み側でブロックできるスリープ可能なRCU
- [mutex](mutex.md) — スリープする相互排他
- [IPC: シグナル](../ipc/signals.md) — 待機中のタスクへのシグナル配送
- [io_uring](../io-uring/io-uring-arch.md) — io_uringは待ちキューを完全に回避する
- `include/linux/wait.h` — 待ちキューAPIの全体
