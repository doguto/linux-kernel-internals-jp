# eventfdとsignalfd {#eventfd-and-signalfd}

> ファイルディスクリプタを通じたイベント通知とシグナル配送

## eventfd: pollできるカウンタ {#eventfd-a-counter-you-can-poll}

`eventfd`はカーネル内のカウンタを裏付けとするファイルディスクリプタを作成する。用途は次のとおり:

- スレッド間またはプロセス間のイベント通知
- 別のコンテキストから`epoll`/`select`/`poll`を起こす
- 発生回数のカウント（セマフォに似ているが、pollできる）

```c
#include <sys/eventfd.h>

int efd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
/* initval=0: カウンタの初期値 */
/* EFD_SEMAPHORE: セマフォモード（後述） */
```

### 書き込み（通知） {#writing-signaling}

8バイトの`uint64_t`を書き込むと、カウンタに加算される:

```c
uint64_t value = 1;
write(efd, &value, sizeof(value));  /* counter += 1 */

/* 複数回分の通知: */
value = 5;
write(efd, &value, sizeof(value));  /* counter += 5 */

/* カウンタの上限は UINT64_MAX - 1 */
/* 上限に達すると write はブロックする（EFD_NONBLOCK なら EAGAIN を返す）。飽和はしない */
```

### 読み出し（消費） {#reading-consuming}

8バイトの`uint64_t`を読み出すと、現在のカウンタ値が返され、カウンタは0にリセットされる:

```c
uint64_t count;
ssize_t n = read(efd, &count, sizeof(count));
/* n == 8、count == 蓄積された値、カウンタは0にリセット */

/* EFD_NONBLOCK: counter == 0 なら EAGAIN を返す */
```

### EFD_SEMAPHOREモード {#efd_semaphore-mode}

`EFD_SEMAPHORE`を指定すると、各`read()`はカウンタを1減らし、（総数ではなく）1を返す:

```c
int efd = eventfd(0, EFD_SEMAPHORE | EFD_NONBLOCK);

write(efd, &(uint64_t){3}, 8);  /* counter = 3 */

uint64_t val;
read(efd, &val, 8);  /* val=1, counter=2 */
read(efd, &val, 8);  /* val=1, counter=1 */
read(efd, &val, 8);  /* val=1, counter=0 */
read(efd, &val, 8);  /* EAGAIN: counter==0 */
```

### epoll統合 {#epoll-integration}

重要な特徴は、`eventfd`が`epoll`と統合されるため、眠っているイベントループを起こせることである:

```c
int epfd = epoll_create1(0);
int efd  = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);

struct epoll_event ev = { .events = EPOLLIN, .data.fd = efd };
epoll_ctl(epfd, EPOLL_CTL_ADD, efd, &ev);

/* 別のスレッドまたはシグナルハンドラ内で: */
write(efd, &(uint64_t){1}, 8);  /* epoll_wait を起こす */

/* メインループ: */
struct epoll_event events[16];
int n = epoll_wait(epfd, events, 16, -1);
for (int i = 0; i < n; i++) {
    if (events[i].data.fd == efd) {
        uint64_t count;
        read(efd, &count, 8);   /* 消費 + リセット */
        /* イベントを処理する */
    }
}
```

### 典型的な用途 {#typical-uses}

**スレッドプールの起床**: mutex+条件変数の代わりに、eventfdを作業キューと組み合わせて使う:

```c
struct thread_pool {
    int        wakeup_efd;  /* eventfd */
    /* ... 作業キュー ... */
};

/* 作業をキューに入れ、スレッドを起こす */
void enqueue(struct thread_pool *pool, struct work *w) {
    work_queue_push(pool, w);
    write(pool->wakeup_efd, &(uint64_t){1}, 8);
}

/* ワーカースレッド */
void worker(struct thread_pool *pool) {
    while (1) {
        uint64_t count;
        read(pool->wakeup_efd, &count, 8);  /* 作業が来るまでブロック */
        /* 'count' 個の項目を処理 */
    }
}
```

**io_uringの完了通知**: io_uringは完了が到着したときにeventfdへ通知を送ることができる:

```c
io_uring_register_eventfd(ring, efd);
/* これで CQE が利用可能になると efd に対する epoll_wait が起きる */
```

### カーネルでの実装 {#kernel-implementation}

```c
/* fs/eventfd.c */
struct eventfd_ctx {
    struct kref     kref;
    wait_queue_head_t wqh;
    __u64           count;
    unsigned int    flags;
    int             id;   /* デバッグ用 */
};

static ssize_t eventfd_write(struct file *file, const char __user *buf,
                             size_t count, loff_t *ppos)
{
    struct eventfd_ctx *ctx = file->private_data;
    ssize_t res;
    __u64 ucnt;

    if (count != sizeof(ucnt))
        return -EINVAL;
    if (copy_from_user(&ucnt, buf, sizeof(ucnt)))
        return -EFAULT;
    /* ちょうど最大値を書き込むと、飽和させずにそのまま拒否する */
    if (ucnt == ULLONG_MAX)
        return -EINVAL;

    spin_lock_irq(&ctx->wqh.lock);
    res = -EAGAIN;
    if (ULLONG_MAX - ctx->count > ucnt)
        res = sizeof(ucnt);
    else if (!(file->f_flags & O_NONBLOCK)) {
        /* 黙って飽和させるのではなく、空きができるまでブロックする */
        res = wait_event_interruptible_locked_irq(ctx->wqh,
                ULLONG_MAX - ctx->count > ucnt);
        if (!res)
            res = sizeof(ucnt);
    }
    if (likely(res > 0)) {
        ctx->count += ucnt;
        if (waitqueue_active(&ctx->wqh))
            wake_up_locked_poll(&ctx->wqh, EPOLLIN);
    }
    spin_unlock_irq(&ctx->wqh.lock);

    return res;
}

/* eventfd は古典的な .read(file, buf, count, ppos) シグネチャではなく、
 * iov_iter の読み出しインターフェース（.read_iter）を使う */
static ssize_t eventfd_read(struct kiocb *iocb, struct iov_iter *to)
{
    struct file *file = iocb->ki_filp;
    struct eventfd_ctx *ctx = file->private_data;
    __u64 ucnt = 0;

    if (iov_iter_count(to) < sizeof(ucnt))
        return -EINVAL;

    spin_lock_irq(&ctx->wqh.lock);
    if (!ctx->count) {
        if ((file->f_flags & O_NONBLOCK) || (iocb->ki_flags & IOCB_NOWAIT)) {
            spin_unlock_irq(&ctx->wqh.lock);
            return -EAGAIN;
        }
        /* count > 0 になるまでスリープ */
        if (wait_event_interruptible_locked_irq(ctx->wqh, ctx->count)) {
            spin_unlock_irq(&ctx->wqh.lock);
            return -ERESTARTSYS;
        }
    }

    if (ctx->flags & EFD_SEMAPHORE) {
        ucnt = 1;
        ctx->count--;
    } else {
        ucnt = ctx->count;
        ctx->count = 0;
    }
    /* count が上限だった場合に備え、ブロック中の書き手を起こす */
    if (waitqueue_active(&ctx->wqh))
        wake_up_locked_poll(&ctx->wqh, EPOLLOUT);
    spin_unlock_irq(&ctx->wqh.lock);

    if (unlikely(copy_to_iter(&ucnt, sizeof(ucnt), to) != sizeof(ucnt)))
        return -EFAULT;
    return sizeof(ucnt);
}
```

## signalfd: ファイルディスクリプタ経由でシグナルを受け取る {#signalfd-receive-signals-via-file-descriptor}

通常、シグナルは予測できない時点で現在の実行に割り込む。`signalfd`はシグナルをファイルディスクリプタ上の読み出し可能なデータとして配送し、シグナル処理を`epoll`駆動のイベントループと両立させる。

```c
#include <sys/signalfd.h>
#include <signal.h>

/* まず通常の配送からシグナルをブロックする */
sigset_t mask;
sigemptyset(&mask);
sigaddset(&mask, SIGINT);
sigaddset(&mask, SIGTERM);
sigaddset(&mask, SIGUSR1);
sigprocmask(SIG_BLOCK, &mask, NULL);

/* これらのシグナル用の signalfd を作成 */
int sfd = signalfd(-1, &mask, SFD_NONBLOCK | SFD_CLOEXEC);

/* シグナル情報を読む（ブロックする。SFD_NONBLOCK なら EAGAIN） */
struct signalfd_siginfo ssi;
ssize_t n = read(sfd, &ssi, sizeof(ssi));
if (n == sizeof(ssi)) {
    printf("signal %u from pid %u\n", ssi.ssi_signo, ssi.ssi_pid);
    printf("uid=%u code=%d status=%d\n",
           ssi.ssi_uid, ssi.ssi_code, ssi.ssi_status);
}
```

### signalfd_siginfoのフィールド {#signalfd_siginfo-fields}

```c
struct signalfd_siginfo {
    uint32_t ssi_signo;    /* シグナル番号 */
    int32_t  ssi_errno;    /* エラー番号（通常は0） */
    int32_t  ssi_code;     /* シグナルコード（SI_USER、SI_KERNEL など） */
    uint32_t ssi_pid;      /* 送信者のPID */
    uint32_t ssi_uid;      /* 送信者のUID */
    int32_t  ssi_fd;       /* ファイルディスクリプタ（SIGIO） */
    uint32_t ssi_tid;      /* タイマーID（SIGALRM/SIGVTALRM） */
    uint32_t ssi_band;     /* バンドイベント（SIGIO） */
    uint32_t ssi_overrun;  /* タイマーのオーバーラン回数 */
    uint32_t ssi_trapno;   /* トラップ番号（ハードウェアフォールト） */
    int32_t  ssi_status;   /* 終了ステータス/シグナル（SIGCHLD） */
    int32_t  ssi_int;      /* 整数ペイロード（sigqueue） */
    uint64_t ssi_ptr;      /* ポインタペイロード（sigqueue） */
    uint64_t ssi_utime;    /* 消費したユーザーCPU時間（SIGCHLD） */
    uint64_t ssi_stime;    /* 消費したシステムCPU時間（SIGCHLD） */
    uint64_t ssi_addr;     /* フォールトしたアドレス（SIGSEGV/SIGBUS） */
    /* ... パディング ... */
};
```

### signalfdを使ったepoll {#epoll-with-signalfd}

I/Oとシグナルの両方を扱うシングルスレッドサーバーの定番パターン:

```c
/* スレッド/プロセスレベルでシグナルをブロック */
sigset_t mask;
sigemptyset(&mask);
sigaddset(&mask, SIGINT);
sigaddset(&mask, SIGTERM);
sigaddset(&mask, SIGCHLD);
pthread_sigmask(SIG_BLOCK, &mask, NULL);

int sfd = signalfd(-1, &mask, SFD_NONBLOCK | SFD_CLOEXEC);
int epfd = epoll_create1(EPOLL_CLOEXEC);

/* signalfd と他の fd を epoll に追加 */
struct epoll_event ev = { .events = EPOLLIN, .data.fd = sfd };
epoll_ctl(epfd, EPOLL_CTL_ADD, sfd, &ev);

/* ... ソケット fd、タイマー fd などを追加 ... */

/* すべてを1つのイベントループで扱う */
while (running) {
    struct epoll_event events[32];
    int n = epoll_wait(epfd, events, 32, -1);

    for (int i = 0; i < n; i++) {
        int fd = events[i].data.fd;

        if (fd == sfd) {
            struct signalfd_siginfo ssi;
            read(sfd, &ssi, sizeof(ssi));
            if (ssi.ssi_signo == SIGTERM || ssi.ssi_signo == SIGINT)
                running = 0;
            else if (ssi.ssi_signo == SIGCHLD)
                reap_children();
        } else {
            handle_io(fd);
        }
    }
}
```

### signalfdと従来のシグナル処理の比較 {#signalfd-vs-traditional-signal-handling}

| 手法 | スレッドセーフ | epoll対応 | シグナル情報 |
|----------|------------|-----------------|-------------|
| `signal()`/`sigaction()` | SA_RESTARTに注意が必要 | 不可 | 限定的 |
| `sigwaitinfo()` | 可（ブロッキング） | 不可 | 完全なsiginfo |
| `signalfd` | 可 | **可** | 完全なsiginfo_t |
| `self-pipe trick` | 可 | 可 | シグナル番号のみ |

**self-pipe trick**（シグナルハンドラ内でパイプに1バイト書き込み、イベントループ内でもう一方の端から読む）が従来の解決策だった。`signalfd`はこれをすっきりと置き換える。

### signalfdの更新 {#signalfd-update}

既存のsfdを渡すと、そのマスクを更新できる:

```c
/* 既存の signalfd に SIGUSR2 を追加 */
sigaddset(&mask, SIGUSR2);
signalfd(sfd, &mask, 0);  /* 第1引数は既存の fd */
```

### カーネルでの実装 {#kernel-implementation_1}

```c
/* fs/signalfd.c — 実際の関数は古典的な .read(file, buf, count, ppos)
 * シグネチャではなく .read_iter を使い、dequeue_signal() をインラインで呼ぶ
 * のではなく、取り出しをヘルパー（signalfd_dequeue()）に委ねる */
static ssize_t signalfd_read_iter(struct kiocb *iocb, struct iov_iter *to)
{
    struct file *file = iocb->ki_filp;
    struct signalfd_ctx *ctx = file->private_data;
    size_t count = iov_iter_count(to);
    ssize_t ret, total = 0;
    kernel_siginfo_t info;
    bool nonblock;

    count /= sizeof(struct signalfd_siginfo);
    if (!count)
        return -EINVAL;

    nonblock = file->f_flags & O_NONBLOCK || iocb->ki_flags & IOCB_NOWAIT;
    do {
        /* マスクに一致する保留中のシグナルを取り出す（nonblock でなければ
         * 内部でブロックし、current->sighand->signalfd_wqh 上でスリープする） */
        ret = signalfd_dequeue(ctx, &info, nonblock);
        if (unlikely(ret <= 0))
            break;
        /* siginfo をユーザー空間の signalfd_siginfo 形式で iov_iter に直接
         * コピーする -- 中間の siginfo_t バッファは使わない */
        ret = signalfd_copyinfo(to, &info);
        if (ret < 0)
            break;
        total += ret;
        nonblock = 1;   /* ブロックしてよいのは最初の反復だけ */
    } while (--count);

    return total ? total : ret;
}
```

`dequeue_signal()`自体はちょうど3つの引数 `(sigset_t *mask, kernel_siginfo_t *info, enum pid_type *type)` を取り、明示的な`current`引数はない（呼び出し元自身の保留中シグナルを操作する）。カーネル内のどこにも`copy_siginfo_to_user_sighand()`は存在しない。ユーザー空間の`struct signalfd_siginfo`レイアウトへの実際の変換は`signalfd_copyinfo()`が行い、`siginfo_layout()`のswitchを介してフィールドを対応付ける。

## timerfd: pollできるタイマー {#timerfd-pollable-timers}

`timerfd`（[POSIXタイマー](../time/posix-timers.md)で解説）も同じパターンに従う。タイマーが発火すると読み出し可能になるファイルディスクリプタである。eventfdやsignalfdと組み合わせれば、スレッドなしで完全なイベントループを実現できる:

```c
/* 4つすべてが epoll で使える: */
epoll_ctl(epfd, EPOLL_CTL_ADD, timerfd, &ev_timer);
epoll_ctl(epfd, EPOLL_CTL_ADD, signalfd, &ev_signal);
epoll_ctl(epfd, EPOLL_CTL_ADD, eventfd,  &ev_event);
epoll_ctl(epfd, EPOLL_CTL_ADD, sock_fd,  &ev_socket);
/* 1つの epoll_wait でタイマー + シグナル + イベント + I/O を扱う */
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [fs/eventfd.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/eventfd.c) — `do_eventfd()`、`eventfd_read()`/`eventfd_write()`、およびカーネル内からの通知のエントリポイントである`eventfd_signal_mask()`
- [fs/signalfd.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/signalfd.c) — `signalfd_read_iter()`、`signalfd_copyinfo()`、`do_signalfd4()`
- [include/uapi/linux/eventfd.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/eventfd.h) — `EFD_SEMAPHORE`/`EFD_CLOEXEC`/`EFD_NONBLOCK`フラグの定義
- [include/uapi/linux/signalfd.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/signalfd.h) — `signalfd_siginfo`構造体のレイアウト

### manページ {#man-pages}

- [`eventfd(2)`](https://man7.org/linux/man-pages/man2/eventfd.2.html) — カウンタの意味論、`EFD_SEMAPHORE`、ブロック/EAGAINとなる`ULLONG_MAX - 1`の上限
- [`signalfd(2)`](https://man7.org/linux/man-pages/man2/signalfd.2.html) — `signalfd_siginfo`のフィールドと、epoll+`fork()`に関する注意点

### 関連ページ {#related-pages}

- [シグナル](signals.md) — `signalfd`が置き換える従来のシグナル配送パス
- [epollの内部構造](../net/epoll.md) — eventfd/signalfdが統合されるよう設計されたイベントループ
- [POSIXタイマーとtimerfd](../time/posix-timers.md) — すべてをfdで扱うイベントループを完成させる3つ目のpoll可能なfdプリミティブ
- [完了通知と待ちキュー](../locking/completions.md) — `eventfd_ctx`とsignalfdのsighandごとのキューの土台となる`wait_queue_head_t`プリミティブ
- [IPC事件簿](war-stories.md) — `eventfd_write()`に照らして解説する、実際のeventfdの`EAGAIN`/バックプレッシャー事例
