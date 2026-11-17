# POSIXタイマーとtimerfd {#posix-timers-and-timerfd}

> ユーザー向けのタイマーAPI: timer_create、clock_nanosleep、timerfd

## POSIXタイマーAPI {#posix-timer-apis}

POSIXは、任意のクロックIDの上に構築された移植性のあるタイマーインターフェースを定義している。

```c
#include <signal.h>
#include <time.h>

/* タイマーを作成する */
timer_t timerid;
struct sigevent sev = {
    .sigev_notify = SIGEV_SIGNAL,   /* シグナルで通知 */
    .sigev_signo  = SIGRTMIN,       /* どのシグナルか */
};
timer_create(CLOCK_MONOTONIC, &sev, &timerid);

/* 起動する: 最初の満了は100ms後、その後50msごとに繰り返す */
struct itimerspec its = {
    .it_value    = { .tv_sec = 0, .tv_nsec = 100000000 }, /* 100ms */
    .it_interval = { .tv_sec = 0, .tv_nsec =  50000000 }, /* 50msで繰り返し */
};
timer_settime(timerid, 0 /* 相対 */, &its, NULL);

/* 現在の状態を問い合わせる */
struct itimerspec cur;
timer_gettime(timerid, &cur);

/* 削除する */
timer_delete(timerid);
```

### 通知モード {#notification-modes}

```c
/* sigev_notify の選択肢: */

/* シグナルを配送する（デフォルト） */
sev.sigev_notify = SIGEV_SIGNAL;
sev.sigev_signo  = SIGRTMIN + 1;

/* 満了ごとに新しいスレッドを作成する */
sev.sigev_notify            = SIGEV_THREAD;
sev.sigev_notify_function   = my_callback;
sev.sigev_notify_attributes = NULL;

/* 特定のスレッドにシグナルを送る */
sev.sigev_notify            = SIGEV_THREAD_ID;
sev.sigev_signo             = SIGRTMIN;
sev._sigev_un._tid          = gettid();  /* 特定のスレッド */

/* 通知なし（timer_gettime で手動で問い合わせる） */
sev.sigev_notify = SIGEV_NONE;
```

## clock_nanosleep {#clock_nanosleep}

`clock_nanosleep`は、呼び出したスレッドを絶対時刻または相対時間まで停止させる。

```c
struct timespec req, rem;

/* 相対スリープ: 10ms */
req.tv_sec  = 0;
req.tv_nsec = 10000000;
clock_nanosleep(CLOCK_MONOTONIC, 0, &req, &rem);

/* 絶対スリープ: 特定のモノトニック時刻に起床する */
struct timespec now, abs;
clock_gettime(CLOCK_MONOTONIC, &now);
abs.tv_sec  = now.tv_sec;
abs.tv_nsec = now.tv_nsec + 5000000;  /* 今から5ms後 */
if (abs.tv_nsec >= 1000000000L) {
    abs.tv_sec++;
    abs.tv_nsec -= 1000000000L;
}
/* TIMER_ABSTIME: 絶対時刻を過ぎていなければシグナルによって早く起床しない */
clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, &abs, NULL);
```

絶対形式は、正確な周期ループにとって重要である。相対スリープの累積によるドリフトを避けられる。

### 正確な周期ループ {#precise-periodic-loop}

```c
/* よくあるパターン: 固定周期でのオーディオ/ビデオコールバック */
static void run_periodic(long period_ns)
{
    struct timespec next;
    clock_gettime(CLOCK_MONOTONIC, &next);

    while (running) {
        /* 処理を行う */
        do_work();

        /* 次の周期へ進める（絶対時刻なのでドリフトなし） */
        next.tv_nsec += period_ns;
        if (next.tv_nsec >= 1000000000L) {
            next.tv_sec++;
            next.tv_nsec -= 1000000000L;
        }
        clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, &next, NULL);
    }
}
```

## timerfd {#timerfd}

`timerfd_create`は、タイマー満了時に読み取り可能になるファイルディスクリプタを返す。これにより、タイマーを`epoll`や`select`と統合できる。シグナルは不要である。

```c
#include <sys/timerfd.h>
#include <sys/epoll.h>

/* timerfdを作成する */
int tfd = timerfd_create(CLOCK_MONOTONIC, TFD_NONBLOCK | TFD_CLOEXEC);

/* 起動する: 100msごとに発火 */
struct itimerspec its = {
    .it_value    = { .tv_nsec = 100000000 },
    .it_interval = { .tv_nsec = 100000000 },
};
timerfd_settime(tfd, 0, &its, NULL);

/* epollと統合する */
int epfd = epoll_create1(EPOLL_CLOEXEC);
struct epoll_event ev = { .events = EPOLLIN, .data.fd = tfd };
epoll_ctl(epfd, EPOLL_CTL_ADD, tfd, &ev);

/* イベントループ */
while (1) {
    struct epoll_event events[8];
    int n = epoll_wait(epfd, events, 8, -1);

    for (int i = 0; i < n; i++) {
        if (events[i].data.fd == tfd) {
            uint64_t expirations;
            read(tfd, &expirations, sizeof(expirations));
            /* expirations = 前回のreadからタイマーが発火した回数
               （処理が遅れていれば1より大きくなりうる） */
            handle_timer(expirations);
        }
    }
}
```

### timerfd_settimeのフラグ {#timerfd_settime-flags}

```c
/* 絶対時刻 */
timerfd_settime(tfd, TFD_TIMER_ABSTIME, &its, NULL);

/* クロック変更時にキャンセル（CLOCK_REALTIME。TFD_TIMER_ABSTIME と組み合わせる必要がある） */
timerfd_settime(tfd, TFD_TIMER_ABSTIME | TFD_TIMER_CANCEL_ON_SET, &its, NULL);
/* クロックが設定された後の read で ECANCELED を返す */
```

## カーネルの実装 {#kernel-implementation}

### カーネル内のPOSIXタイマー {#posix-timers-in-the-kernel}

```c
/* include/linux/posix-timers.h */
struct k_itimer {
    /* 1つ目のキャッシュラインには主に読み取られるフィールドを置く */
    struct hlist_node   t_hash;
    struct hlist_node   list;
    timer_t             it_id;
    clockid_t           it_clock;
    int                 it_sigev_notify;
    enum pid_type       it_pid_type;
    struct signal_struct *it_signal;
    const struct k_clock *kclock;

    /* 2つ目以降のキャッシュラインには頻繁に変更されるフィールドを置く */
    spinlock_t          it_lock;
    int                 it_status;
    bool                it_sig_periodic;
    s64                 it_overrun;     /* 取りこぼした満了の数 */
    s64                 it_overrun_last;
    unsigned int        it_signal_seq;
    unsigned int        it_sigqueue_seq;
    ktime_t             it_interval;    /* 再設定の間隔 */
    struct hlist_node   ignored_list;
    union {
        struct pid         *it_pid;
        struct task_struct *it_process;
    };
    struct sigqueue     sigq;           /* 事前に割り当てられたシグナル */
    rcuref_t            rcuref;

    union {
        struct {
            struct hrtimer  timer;      /* このPOSIXタイマーを支えるhrtimer */
        } real;
        struct cpu_timer    cpu;        /* CPU時間タイマー */
        struct {
            struct alarm    alarmtimer;
        } alarm;
    } it;
    struct rcu_head     rcu;
};
```

`timer_create()`はそれぞれ、`hrtimer`に支えられた`k_itimer`を割り当てる。hrtimerが発火すると、シグナルをキューに入れる。

### カーネル内のtimerfd {#timerfd-in-the-kernel}

```c
/* fs/timerfd.c */
struct timerfd_ctx {
    union {
        struct hrtimer tmr;
        struct alarm alarm;
    } t;
    ktime_t tintv;
    ktime_t moffs;
    wait_queue_head_t wqh;
    u64 ticks;
    int clockid;
    short unsigned expired;
    short unsigned settime_flags;	/* fdinfo に表示するため */
    struct rcu_head rcu;
    struct list_head clist;
    spinlock_t cancel_lock;
    bool might_cancel;
};

/* hrtimerコールバック: epollの待機者を起床させる */
static enum hrtimer_restart timerfd_tmrproc(struct hrtimer *htmr)
{
    struct timerfd_ctx *ctx = container_of(htmr, struct timerfd_ctx,
                                           t.tmr);
    timerfd_triggered(ctx);
    return HRTIMER_NORESTART;
}
```

## オーバーラン {#overruns}

周期タイマーが、消費するよりも速く発火すると、満了が蓄積する。

```c
/* timer_create: オーバーランを確認する */
struct itimerspec its;
timer_gettime(timerid, &its);
int overruns = timer_getoverrun(timerid);
/* overruns > 0: この回数の満了を取りこぼした */

/* timerfd: read は蓄積された回数を返す */
uint64_t count;
ssize_t sz = read(tfd, &count, sizeof(count));
if (count > 1)
    fprintf(stderr, "Behind by %llu expirations\n", count - 1);
```

オーバーランが起きるのは次のような場合である。

- ハンドラが周期より長くかかる
- システムに負荷がかかっており、スケジューリングレイテンシが周期を超える
- スレッドがブロックされていた（スリープ中、I/O待ち）

## 適切なAPIの選び方 {#choosing-the-right-api}

| シナリオ | 使うもの |
|----------|-----|
| 単純なスリープ | `clock_nanosleep` |
| シグナルベースの周期的処理 | `timer_create(SIGEV_SIGNAL)` |
| イベントループ（epoll/select） | `timerfd_create` |
| リアルタイムの周期スレッド | `clock_nanosleep(TIMER_ABSTIME)` |
| I/Oのタイムアウト | `timerfd` + `epoll` |
| カーネルドライバのタイマー | `hrtimer` |

## POSIXタイマーの観察 {#observing-posix-timers}

```bash
# プロセスのアクティブなタイマー
cat /proc/<pid>/timers
# ID: 0
# signal: 34 /...
# notify: signal/pid.12345
# ClockID: 1 (CLOCK_MONOTONIC)

# timerfd は lsof では匿名inodeとして現れる
lsof -p <pid> | grep timerfd

# perf: タイマー関連のシステムコール
perf trace -e timer_create,clock_nanosleep -p <pid>

# tracepoint: POSIXタイマーのイベント
echo 1 > /sys/kernel/tracing/events/timer/hrtimer_start/enable
echo 1 > /sys/kernel/tracing/events/syscalls/sys_enter_timer_settime/enable
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/posix-timers.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/posix-timers.h) — `struct k_itimer`、`struct k_clock`操作テーブル、参照カウントのヘルパーの定義
- [kernel/time/posix-timers.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/posix-timers.c) — `timer_create()`、`timer_settime()`、`timer_delete()`、オーバーランの計算
- [kernel/time/posix-cpu-timers.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/posix-cpu-timers.c) — プロセスごと・スレッドごとのCPU時間クロックのハンドラ（`CLOCK_PROCESS_CPUTIME_ID`、`CLOCK_THREAD_CPUTIME_ID`）
- [fs/timerfd.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/timerfd.c) — ファイルディスクリプタタイマーの実装（`timerfd_create()`、`timerfd_settime()`、`timerfd_read_iter()`）、epollのポーリングとの統合、cancel-on-setのロジック

### manページ {#man-pages}

- [`timer_create(2)`](https://man7.org/linux/man-pages/man2/timer_create.2.html) — シグナルまたはスレッドで通知するPOSIXのプロセスごとのタイマーを作成する
- [`timer_settime(2)`](https://man7.org/linux/man-pages/man2/timer_settime.2.html) — POSIXのプロセスごとのタイマーの間隔を設定・解除・確認する
- [`timer_getoverrun(2)`](https://man7.org/linux/man-pages/man2/timer_getoverrun.2.html) — 周期POSIXタイマーの取りこぼした満了回数を取得する
- [`timer_delete(2)`](https://man7.org/linux/man-pages/man2/timer_delete.2.html) — POSIXのプロセスごとのタイマーを削除し、そのIDを回収する
- [`timerfd_create(2)`](https://man7.org/linux/man-pages/man2/timerfd_create.2.html) — epoll/poll/selectで待てるファイルディスクリプタのイベントで通知するタイマー
- [`clock_nanosleep(2)`](https://man7.org/linux/man-pages/man2/clock_nanosleep.2.html) — クロック指定と絶対的な起床期限を持つ高分解能スリープ

### 関連ページ {#related-pages}

- [高分解能タイマー（hrtimers）](hrtimers.md) — POSIXタイマーとtimerfdの基盤となる、ナノ秒分解能のタイマーサブシステム
- [時刻管理とクロックソース](timekeeping.md) — POSIXクロックID（`CLOCK_REALTIME`、`CLOCK_MONOTONIC`、`CLOCK_BOOTTIME`、`CLOCK_TAI`）
- [シグナルとシグナル処理](../ipc/signals.md) — `struct sigqueue`によるリアルタイムシグナルのキューイングと配送
- [io_uringの操作](../io-uring/io-uring-ops.md) — `IORING_OP_TIMEOUT`によるリングベースの非同期タイムアウト

### LWN記事 {#lwn-articles}

- [The new timerfd() API](https://lwn.net/Articles/251413/) — Jonathan Corbet、2007年9月25日: `timerfd_create()`の設計とAPI
- [posix-timers: Cure inconsistencies and the SIG_IGN mess](https://lwn.net/Articles/977822/) — Thomas Gleixner、2024年6月: SIG_IGNの自己再設定/スロットリングという回避策を取り除く正しさの修正。`posix-timers.c`、`posix-cpu-timers.c`、`kernel/signal.c`にまたがる

### 外部資料 {#external}

- [Timekeeping and Timers in Linux](https://docs.kernel.org/core-api/timekeeping.html) — POSIXクロックとタイマーを扱うカーネルドキュメント
