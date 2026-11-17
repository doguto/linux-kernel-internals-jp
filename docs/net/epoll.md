# epollの内部 {#epoll-internals}

> epollが何千ものファイルディスクリプタをいかに効率よく監視するか

## poll/selectの問題点 {#the-problem-with-pollselect}

`select()`と`poll()`は、呼び出しのたびに監視対象のファイルディスクリプタの集合全体を渡す必要がある。N個のfdに対して:

- ユーザー空間 → カーネルへのコピー: O(N)
- カーネルがすべてのfdの準備状態を走査: O(N)
- 準備完了したfdのカーネル → ユーザー空間へのコピー: O(N)
- 1回の呼び出しの合計: 準備完了したfdが1つだけでもO(N)

コネクションが10,000あれば、これは許容できない。

## epoll: イベントあたりO(1) {#epoll-o1-per-event}

epollは**カーネル内に常駐する関心リスト**を使う。一度登録すれば、再コピーは不要である:

```
epoll workflow:

  epoll_create()  →  create struct eventpoll (rb-tree + ready list)
  epoll_ctl(ADD)  →  insert fd into rb-tree, add wait queue callback
  epoll_wait()    →  sleep; wake when ready list is non-empty
  [fd becomes ready] → ep_poll_callback adds to ready list, wakes waiter
  epoll_wait returns → copy only ready events (O(ready), not O(N))
```

## epollインスタンスの作成 {#creating-an-epoll-instance}

```c
#include <sys/epoll.h>

/* epoll インスタンスを作成 */
int epfd = epoll_create1(EPOLL_CLOEXEC);

/* ファイルディスクリプタを追加 */
struct epoll_event ev = {
    .events  = EPOLLIN | EPOLLERR | EPOLLHUP,
    .data.fd = sock_fd,   /* または .data.ptr = &mydata */
};
epoll_ctl(epfd, EPOLL_CTL_ADD, sock_fd, &ev);

/* 変更 */
ev.events = EPOLLIN | EPOLLOUT;
epoll_ctl(epfd, EPOLL_CTL_MOD, sock_fd, &ev);

/* 削除 */
epoll_ctl(epfd, EPOLL_CTL_DEL, sock_fd, NULL);

/* 待機: 最大 maxevents 個の準備完了 fd を返す */
struct epoll_event events[64];
int n = epoll_wait(epfd, events, 64, -1);   /* -1 = タイムアウトなし */
for (int i = 0; i < n; i++) {
    int fd = events[i].data.fd;
    if (events[i].events & EPOLLIN)
        handle_read(fd);
    if (events[i].events & EPOLLOUT)
        handle_write(fd);
}
```

## レベルトリガーとエッジトリガー {#level-triggered-vs-edge-triggered}

### レベルトリガー（LT）— デフォルト {#level-triggered-lt-default}

`epoll_wait`は、fdが準備完了である限り返る:

```c
/* LT: フラグは不要、これがデフォルト */
ev.events = EPOLLIN;

/* 挙動: */
/* 受信バッファに 100 バイトある */
/* epoll_wait は fd=5 (EPOLLIN) を返す */
/* 50 バイトだけ読む */
/* epoll_wait は再び fd=5 (EPOLLIN) を返す */
/* 残りの 50 バイトを読む */
/* epoll_wait はブロックする（バッファが空） */
```

安全である: 1回の呼び出しでバッファを空にしなくても、再び通知される。

### エッジトリガー（ET） {#edge-triggered-et}

`epoll_wait`は、fdが準備未完了から準備完了へ**遷移**したときにだけ返る:

```c
ev.events = EPOLLIN | EPOLLET;

/* 挙動: */
/* 受信バッファに 100 バイトある */
/* epoll_wait は fd=5 (EPOLLIN) を返す — 遷移: 空 → データあり */
/* 50 バイトだけ読む */
/* epoll_wait は fd=5 を再び返さない（まだ準備完了だが、遷移がない） */
/* さらにデータが届く: 新たな遷移 → epoll_wait が返る */
```

EPOLLETを使う場合は、ノンブロッキングのループで`EAGAIN`になるまで読み続け**なければならない**:

```c
void handle_read_et(int fd) {
    char buf[4096];
    while (1) {
        ssize_t n = recv(fd, buf, sizeof(buf), 0);
        if (n > 0) {
            process(buf, n);
        } else if (n == 0) {
            close_connection(fd);
            break;
        } else {
            if (errno == EAGAIN || errno == EWOULDBLOCK)
                break;  /* すべてのデータを読んだ */
            if (errno == EINTR)
                continue;
            handle_error(fd);
            break;
        }
    }
}
```

### EPOLLONESHOT: 1回だけの通知 {#epolloneshot-single-notification}

通知を1回受け取ると、そのfdは自動的に無効化される:

```c
ev.events = EPOLLIN | EPOLLONESHOT;
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);

/* 1回のイベントの後、さらに受け取るには再設定が必要: */
epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev);  /* 再有効化 */
```

各イベントを1つのスレッドだけが処理すべきマルチスレッドのepollで有用である。

## カーネルのデータ構造 {#kernel-data-structures}

### struct eventpoll {#struct-eventpoll}

```c
/* fs/eventpoll.c */
struct eventpoll {
    /* この構造体へのアクセスを保護する */
    spinlock_t        lock;
    struct mutex      mtx;

    /* sys_epoll_wait() が使う待ちキュー */
    wait_queue_head_t   wq;

    /* file->poll() が使う待ちキュー */
    wait_queue_head_t   poll_wait;

    /* 準備完了したファイルディスクリプタのリスト */
    struct list_head    rdllist;

    /* 監視対象の fd を格納する RB ツリーのルート */
    struct rb_root_cached rbr;

    /* 準備完了イベントをユーザー空間へ転送している間に発生した
       すべての "struct epitem" をつなぐ単方向リンクリスト */
    struct epitem      *ovflist;

    /* ep_scan_ready_list の実行中に使われる wakeup_source */
    struct wakeup_source *ws;

    /* eventpoll ディスクリプタを作成したユーザー */
    struct user_struct *user;

    /* epoll ファイル自体 */
    struct file        *file;

    /* 準備状態チェックの最適化に使われる */
    int                napi_id;
    u64                gen;
    struct hlist_head  refs;
};
```

### struct epitem {#struct-epitem}

監視対象のfdごとに1つの`epitem`がある:

```c
struct epitem {
    /* この構造体を eventpoll の RB ツリーにつなぐためのノード */
    union {
        struct rb_node  rbn;
        struct rcu_head rcu;
    };

    /* この項目を eventpoll の準備完了リストにつなぐためのリストヘッダ */
    struct list_head    rdllink;

    /* ovflist が使われるときに、ポインタの一時的な
       格納場所として働く */
    struct epitem      *next;

    /* この項目が参照するファイルディスクリプタの情報 */
    struct epoll_filefd  ffd;   /* {file*, fd} の組 */

    /* poll 操作にアタッチされたアクティブな待ちキューの数 */
    int                 nwait;

    /* poll の待ちキューを含むリスト */
    struct list_head    pwqlist;

    /* この項目の「コンテナ」 */
    struct eventpoll   *ep;

    /* この項目を ep->items リストにつなぐためのリストヘッダ */
    struct list_head    fllink;

    /* epoll_ctl(EPOLL_CTL_ADD/MOD) からの epoll_event 構造体 */
    __poll_t            revents;
    struct epoll_event  event;
};
```

### ep_poll_callback: 重要な起床パス {#ep_poll_callback-the-critical-wakeup-path}

監視対象のfdが準備完了になると、このコールバックがfdの待ちキューから実行される:

```c
/* fs/eventpoll.c */
static int ep_poll_callback(wait_queue_entry_t *wait, unsigned mode,
                              int sync, void *key)
{
    struct epitem *epi = ep_item_from_wait(wait);
    struct eventpoll *ep = epi->ep;
    __poll_t pollflags = key_to_poll(key);
    unsigned long flags;
    int ewake = 0;

    spin_lock_irqsave(&ep->wq.lock, flags);

    /* 待っているイベントがここになければ、チェイン内の次を試す */
    if (pollflags && !(pollflags & epi->event.events))
        goto out_unlock;

    /* このファイルがすでに準備完了リストにあれば、すぐに抜ける */
    if (!ep_is_linked(epi)) {
        list_add_tail(&epi->rdllink, &ep->rdllist);
        ep_pm_stay_awake_rcu(epi);
    }

    /* eventpoll の待ちリストと ->poll() の待ちリストの両方を
       （アクティブであれば）起床させる。 */
    if (waitqueue_active(&ep->wq)) {
        if ((epi->event.events & EPOLLEXCLUSIVE) &&
            !(pollflags & POLLFREE)) {
            switch (pollflags & EPOLLINOUT_BITS) {
            case EPOLLIN:
                if (epi->event.events & EPOLLIN)
                    ewake = 1;
                break;
            /* ... */
            }
        } else
            ewake = 1;
        wake_up_locked(&ep->wq);
    }

out_unlock:
    spin_unlock_irqrestore(&ep->wq.lock, flags);
    return ewake;
}
```

### epoll_ctl(ADD)のカーネルパス {#epoll_ctladd-kernel-path}

```c
/* fs/eventpoll.c */
static int ep_insert(struct eventpoll *ep, const struct epoll_event *event,
                      struct file *tfile, int fd, int full_check)
{
    struct epitem *epi;

    /* epitem 構造体を割り当てる */
    epi = kmem_cache_zalloc(epi_cache, GFP_KERNEL);

    /* キューコールバックを使って poll テーブルを初期化する */
    epq.epi = epi;
    init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);

    /* ->poll() 経由でファイルの待ちキューにアタッチする */
    revents = ep_item_poll(epi, &epq.pt, 1);
    /* これは file->f_op->poll(file, &epq.pt) を呼ぶ */
    /* ドライバの poll 内の poll_wait() が ep_ptable_queue_proc を
       待ちキューのコールバックとして追加する */

    /* eventpoll の RB ツリーに挿入する */
    ep_rbtree_insert(ep, epi);

    /* すでに準備完了なら、すぐに準備完了リストに追加する */
    if (revents & event->events) {
        list_add_tail(&epi->rdllink, &ep->rdllist);
        /* スリープ中の epoll_wait を起床させる */
        wake_up(&ep->wq);
    }
}
```

## thundering herdとEPOLLEXCLUSIVE {#thundering-herd-and-epollexclusive}

多数のスレッドが同じepfdで`epoll_wait`しているときに1つのイベントが届くと、すべてのスレッドが起床するが、それを処理するのは1つだけである。これが**thundering herd**（群発的起床）である。

`EPOLLEXCLUSIVE`（4.5以降）はこれを防ぐ。イベントごとに1つの待機者だけが起床する:

```c
/* サーバー: accept ソケットごとに1つの epfd、複数のワーカースレッド */
ev.events = EPOLLIN | EPOLLEXCLUSIVE;
epoll_ctl(shared_epfd, EPOLL_CTL_ADD, listen_fd, &ev);

/* コネクションごとに、N 個の epoll_wait スレッドのうち1つだけが起床する */
```

代替策: スレッドごとに別々のepfdを使い、それぞれが`SO_REUSEPORT`ソケット上で自身の`accept()`スレッドを持つようにする:

```c
/* スレッドごとの accept: 各スレッドが自身の listen ソケットを持つ */
setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, &(int){1}, sizeof(int));
/* カーネルは新しいコネクションをすべての SO_REUSEPORT ソケットに負荷分散する */
```

## eventfd、signalfd、timerfdとepoll {#epoll-with-eventfd-signalfd-timerfd}

epollは`->poll`を実装する任意のfdで動作する:

```c
/* 統一されたイベントループ: I/O + シグナル + タイマー + イベント */
epoll_ctl(epfd, EPOLL_CTL_ADD, sock_fd,   &ev_socket);
epoll_ctl(epfd, EPOLL_CTL_ADD, signalfd,  &ev_signal);
epoll_ctl(epfd, EPOLL_CTL_ADD, timerfd,   &ev_timer);
epoll_ctl(epfd, EPOLL_CTL_ADD, eventfd,   &ev_event);
epoll_ctl(epfd, EPOLL_CTL_ADD, pipe_rfd,  &ev_pipe);
```

すべてが単一の`epoll_wait`でブロックする。ポーリングループもビジーウェイトも不要である。

## epollの制限 {#epoll-limitations}

- **通常ファイル**: 常に「準備完了」である。ディスクファイルのI/Oにはepollを使えない（io_uringを使う）
- **ディレクトリ**: poll可能ではない
- **fdごとのタイムアウトがない**: epoll_waitのタイムアウトは全体で1つ
- **通知のみ**: epollはfdが準備完了であることを伝えるだけで、I/Oそのものは行わない（io_uringとは異なる）

## epollの観察 {#observing-epoll}

```bash
# プロセスごとの epoll インスタンス数を数える
ls -l /proc/<pid>/fd | grep eventpoll

# /proc で epoll の fd を見る
cat /proc/<pid>/fdinfo/<epfd>
# pos: 0
# flags: 02
# mnt_id: 9
# tfd:   3 events: 19 data: 3000000003  pos:0 ino:7 stype:0x8001

# epoll_wait の呼び出しをトレース
strace -e epoll_wait,epoll_ctl nginx &

# perf: epoll_wait に費やした時間
perf stat -e syscalls:sys_enter_epoll_wait nginx
```

## 参考文献 {#further-reading}

- [ソケット層の概要](socket-layer.md) — ソケットのpoll()の実装
- [eventfdとsignalfd](../ipc/eventfd-signalfd.md) — epollと互換性のあるイベントfd
- [POSIXタイマーとtimerfd](../time/posix-timers.md) — epollとtimerfdの併用
- [io_uringのアーキテクチャ](../io-uring/io-uring-arch.md) — io_uringはepollのファイルI/Oに関する制限を解決する
- [completionと待ちキュー](../locking/completions.md) — epollが使う待ちキューの仕組み
- `fs/eventpoll.c` — epollの完全な実装
