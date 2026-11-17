# シグナル {#signals}

> プロセスに配送される非同期通知

## シグナルの基本 {#signal-basics}

シグナルは、プロセスまたはスレッドに送られる整数（x86-64では1〜64）である。カーネルは、次の機会（システムコールからの復帰、割り込みからの復帰）に対象プロセスに割り込むことでシグナルを配送する。

```
Sender                Kernel                Receiver
──────                ──────                ────────
kill(pid, SIGTERM) →  pending bit set   →   process checks TIF_SIGPENDING
                      on return to user     arch_do_signal_or_restart()
                      mode:                 → runs signal handler
                                           → or default action (terminate)
```

標準シグナル（1〜31）はキューイングされない。配送前に複数回送られても1つにまとめられる。リアルタイムシグナル（32〜64、`SIGRTMIN`..`SIGRTMAX`）はキューイングされる。

## シグナル番号とデフォルト動作 {#signal-numbers-and-default-actions}

```
SIGHUP  (1)  — hangup (terminal closed) → terminate
SIGINT  (2)  — interrupt (Ctrl+C) → terminate
SIGQUIT (3)  — quit (Ctrl+\) → core dump
SIGILL  (4)  — illegal instruction → core dump
SIGTRAP (5)  — debug trap → core dump
SIGABRT (6)  — abort() → core dump
SIGBUS  (7)  — bad memory access (e.g. misaligned access, truncated mmap) → core dump
SIGFPE  (8)  — arithmetic exception → core dump
SIGKILL (9)  — kill (cannot be caught/ignored) → terminate
SIGSEGV (11) — invalid memory access → core dump
SIGPIPE (13) — write to closed pipe → terminate
SIGALRM (14) — timer (alarm()) → terminate
SIGTERM (15) — termination request → terminate
SIGCHLD (17) — child status change → ignore
SIGCONT (18) — continue if stopped → continue
SIGSTOP (19) — stop (cannot be caught/ignored) → stop
SIGTSTP (20) — terminal stop (Ctrl+Z) → stop
SIGUSR1 (10) — user defined → terminate
SIGUSR2 (12) — user defined → terminate
SIGRTMIN..SIGRTMAX (32..64) — real-time → terminate
```

## シグナルの送信 {#sending-signals}

```c
/* kill: プロセスまたはプロセスグループに送る */
kill(pid, SIGTERM);   /* pid>0: 特定のプロセス */
kill(-pgrp, SIGTERM); /* pid<0: プロセスグループ */
kill(0, SIGTERM);     /* 現在のプロセスグループ */
kill(-1, SIGTERM);    /* すべてのプロセス（init を除く）— 特権が必要 */

/* tkill: 特定のスレッドに送る（非推奨。tgkill を使う） */
tkill(tid, SIGTERM);

/* tgkill: 特定プロセス内の特定スレッドに送る（Linux 固有） */
tgkill(tgid, tid, SIGTERM);

/* sigqueue: データ付きで送る（リアルタイムシグナル用） */
union sigval value;
value.sival_int = 42;
sigqueue(pid, SIGRTMIN, value);
/* 受信者は siginfo_t 内で si_value.sival_int == 42 を受け取る */

/* raise: 現在のスレッドに送る */
raise(SIGINT);

/* カーネルから: */
send_sig(SIGTERM, task, 0);         /* プロセスに送る */
send_sig_info(SIGTERM, &info, task); /* siginfo_t 付き */
kill_pgrp(pgrp, SIGTERM, 0);
```

## カーネルのシグナル配送パス {#kernel-signal-delivery-path}

```c
/* 3つのファイルにまたがる: do_send_sig_info()/get_signal() は kernel/signal.c
 * にある（signal_wake_up() は include/linux/sched/signal.h にある、
 * kernel/signal.c の signal_wake_up_state() の薄いインラインラッパー）。
 * exit_to_user_mode_loop() は kernel/entry/common.c にある。
 * arch_do_signal_or_restart()/handle_signal()/setup_rt_frame() は x86-64 固有で、
 * arch/x86/kernel/signal.c にある */

/* 1. do_send_sig_info(): 対象タスクにシグナルをキューイングする。これは
 *    send_signal_locked() -> __send_signal_locked() の薄いラッパーであり、
 *    以下に示す実際のキューイングとディスパッチはそちらが行う（do_send_sig_info() 自身ではない） */
do_send_sig_info(sig, info, task, type)
  → send_signal_locked(sig, info, task, type)
      → __send_signal_locked(sig, info, task, type, force)
          /* プロセス宛て（kill）: task->signal->shared_pending に入る */
          /* スレッド宛て（tgkill）: task->pending に入る */
          → sigaddset(&pending->signal, sig)
          → complete_signal(sig, task, type)
              → signal_wake_up(task, sig == SIGKILL)  /* スリープ中なら起こし、TIF_SIGPENDING を立てる */

/* 2. ユーザー空間への復帰時（システムコール復帰、割り込み復帰）: */
exit_to_user_mode_loop()
  → if TIF_SIGPENDING: arch_do_signal_or_restart(regs)
      → get_signal(&ksig)               /* 次のシグナルを取り出す（無視/デフォルト
                                          * 動作のケースも内部で処理する --
                                          * 下の呼び出し元に戻るのは
                                          * "handled" の場合だけ） */
      → if handled: handle_signal(&ksig) /* ユーザー空間のスタックフレームを構築 */

/* 3. handle_signal: ユーザー空間スタック上にシグナルフレームを構築 */
handle_signal(ksig, regs)
  → setup_rt_frame(ksig, regs)
      /* ユーザースタックに siginfo_t、ucontext_t を積む */
      /* rip = シグナルハンドラ、rsp = 新しいユーザースタック に設定 */
      /* frame->pretcode = sa_restorer を設定（積まれるのはコードではなくポインタ） */
```

`setup_rt_frame()`（`arch/x86/kernel/signal.c`）自体は、適切なABIの変種を選ぶディスパッチャにすぎない。64ビットプロセスでは`arch/x86/kernel/signal_64.c`の`x64_setup_rt_frame()`を呼び、それが以下の実際の処理を行う。トランポリンの機械語をスタックに積むことはしない。`SA_RESTORER`を必須とし、ユーザー空間のrestorerスタブ（通常はglibc/vDSO内の`__restore_rt`）へのポインタだけを書き込む（`unsafe_put_user(ksig->ka.sa.sa_restorer, &frame->pretcode, ...)`）。シグナルハンドラが戻ると、そのrestorerが`rt_sigreturn`を呼び、保存された`ucontext_t`（シグナル配送前のCPUレジスタ）を復元して通常の実行を再開する。

## struct sigaction: ハンドラのインストール {#struct-sigaction-installing-handlers}

```c
#include <signal.h>

/* sa_handler か sa_sigaction のどちらか（両方は不可） */
struct sigaction {
    union {
        void (*sa_handler)(int signum);
        void (*sa_sigaction)(int signum, siginfo_t *info, void *ucontext);
    };
    sigset_t sa_mask;     /* ハンドラ実行中にブロックするシグナル */
    int      sa_flags;    /* SA_SIGINFO, SA_RESTART, SA_NODEFER, ... */
    void    (*sa_restorer)(void);  /* 内部用 */
};
```

```c
/* シグナルハンドラをインストールする */
struct sigaction sa = {
    .sa_sigaction = my_handler,
    .sa_flags = SA_SIGINFO | SA_RESTART,
};
sigemptyset(&sa.sa_mask);
sigaddset(&sa.sa_mask, SIGTERM);  /* ハンドラ実行中は SIGTERM をブロック */
sigaction(SIGUSR1, &sa, NULL);

/* ハンドラ: */
static void my_handler(int sig, siginfo_t *info, void *ctx)
{
    /* info->si_pid: 送信者の PID */
    /* info->si_code: SI_USER, SI_QUEUE, SI_TIMER, ... */
    /* info->si_value: sigqueue() からのデータ */
    printf("signal %d from pid %d\n", sig, info->si_pid);
}
```

**ここで`printf()`を呼ぶのは実際には安全ではない。** 上の例（および後述の`sigchld_handler`の例）では簡潔さのために示しているが、`printf()`は[signal-safety(7)](https://man7.org/linux/man-pages/man7/signal-safety.7.html)の非同期シグナル安全な関数の一覧に含まれていない。内部のstdioバッファリングはリエントラントではないため、すでに実行中の別の`printf()`呼び出しに割り込んだハンドラは、そのバッファを破壊しうる。実際的な対処は2つある。`write(2)`を直接呼んで（これは非同期シグナル安全*である*）事前に整形したテキストを出力するか、より一般的なパターンとして、ハンドラでは`volatile sig_atomic_t`フラグを立てるだけにし、実際の処理（`printf()`を含む）はシグナルコンテキストの外、メインループに戻ってから行う。

重要なフラグ:

- `SA_SIGINFO` — `sa_handler`の代わりに`sa_sigaction`（3引数のハンドラ）を使う
- `SA_RESTART` — 中断されたシステムコールを自動的に再開する（アプリから`EINTR`が見えなくなる）
- `SA_NODEFER` — 自身のハンドラ実行中にそのシグナルをブロックしない（リエントラント）
- `SA_RESETHAND` — 1回配送した後にデフォルトに戻す

## シグナルマスク {#signal-masks}

各スレッドはシグナルマスクを持つ。ブロックされたシグナルは、ブロックが解除されるまで配送されない:

```c
/* SIGINT と SIGTERM を一時的にブロック */
sigset_t block_set, old_set;
sigemptyset(&block_set);
sigaddset(&block_set, SIGINT);
sigaddset(&block_set, SIGTERM);
sigprocmask(SIG_BLOCK, &block_set, &old_set);

/* ... クリティカルセクション ... */

/* 元のマスクを復元 */
sigprocmask(SIG_SETMASK, &old_set, NULL);

/* アトミックに: ブロックを解除しつつ特定のシグナルを待つ */
sigsuspend(&old_set);  /* マスクを置き換え、何らかのシグナルが来るまでスリープ */
```

### カーネルのデータ構造 {#kernel-data-structures}

```c
/* プロセスごとのシグナル状態 */
struct signal_struct {
    /* 全スレッドで共有される保留中シグナル: */
    struct sigpending   shared_pending;
    /* ... */
};

/* スレッドごとのシグナル状態 */
struct task_struct {
    /* スレッド固有の保留中シグナル: */
    struct sigpending   pending;
    sigset_t            blocked;        /* 現在のシグナルマスク */
    sigset_t            real_blocked;   /* 保存されたマスク（sigwaitinfo 用） */
    sigset_t            saved_sigmask;  /* sigreturn 時に復元される */
    /* ... */
};

struct sigpending {
    struct list_head    list;           /* RT シグナル用にキューイングされた siginfo_t */
    sigset_t            signal;         /* 保留中の標準シグナルのビットマスク */
};
```

## リアルタイムシグナル {#real-time-signals}

リアルタイムシグナル（SIGRTMIN..SIGRTMAX）は標準シグナルと次の点で異なる:

- **キューイングされる**: 配送前に複数回送られても、すべて配送される（FIFO順）
- **値**: 整数またはポインタを運べる（sigqueue + `si_value`経由）
- **優先度**: リアルタイムの範囲内では、シグナル番号が小さいほど優先度が高い

```c
/* データ付きで送信 */
union sigval val = { .sival_int = 100 };
sigqueue(pid, SIGRTMIN+1, val);

/* 複数回分を受信 */
struct sigaction sa = {
    .sa_sigaction = rt_handler,
    .sa_flags = SA_SIGINFO,
};
sigaction(SIGRTMIN+1, &sa, NULL);

/* 同期的にシグナルを待つ */
sigset_t wait_set;
sigemptyset(&wait_set);
sigaddset(&wait_set, SIGRTMIN+1);
sigprocmask(SIG_BLOCK, &wait_set, NULL);  /* sigwaitinfo の前にブロックしておく必要がある */

siginfo_t info;
sigwaitinfo(&wait_set, &info);
/* info.si_value.sival_int == 100 */
```

## signalfd: ファイルディスクリプタとしてのシグナル {#signalfd-signals-as-file-descriptors}

`signalfd`はシグナル配送を`read()`呼び出しに変換し、`epoll`と組み合わせて使えるようにする:

```c
#include <sys/signalfd.h>
#include <sys/epoll.h>

/* signalfd 経由で受け取りたいシグナルをブロック */
sigset_t mask;
sigemptyset(&mask);
sigaddset(&mask, SIGINT);
sigaddset(&mask, SIGTERM);
sigprocmask(SIG_BLOCK, &mask, NULL);

/* signalfd を作成 */
int sfd = signalfd(-1, &mask, SFD_NONBLOCK | SFD_CLOEXEC);

/* epoll に追加 */
struct epoll_event ev = { .events = EPOLLIN, .data.fd = sfd };
epoll_ctl(epfd, EPOLL_CTL_ADD, sfd, &ev);

/* シグナル情報を読む */
struct signalfd_siginfo si;
read(sfd, &si, sizeof(si));
if (si.ssi_signo == SIGTERM) {
    /* クリーンにシャットダウン */
}
```

## SIGCHLDとwaitpid {#sigchld-and-waitpid}

`SIGCHLD`は、子の状態が変化した（終了、停止、再開）ときに親に送られる。親は`waitpid`を呼んでゾンビを回収しなければならない:

```c
static void sigchld_handler(int sig, siginfo_t *info, void *ctx)
{
    int status;
    pid_t pid;

    /* 終了したすべての子を回収する（複数の子のためにループ） */
    while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
        if (WIFEXITED(status))
            printf("child %d exited with %d\n", pid, WEXITSTATUS(status));
        else if (WIFSIGNALED(status))
            printf("child %d killed by signal %d\n", pid, WTERMSIG(status));
    }
}
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/signal.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/signal.c) — `do_send_sig_info()`、`send_signal_locked()`/`__send_signal_locked()`、`complete_signal()`、`signal_wake_up_state()`、`get_signal()`: シグナルのキューイングと取り出しの中核実装
- [kernel/entry/common.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/entry/common.c) — `exit_to_user_mode_loop()`: ユーザー空間へ戻る途中で`TIF_SIGPENDING`がチェックされる場所
- [arch/x86/kernel/signal.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/signal.c) — `arch_do_signal_or_restart()`、`handle_signal()`、`setup_rt_frame()`（ABIディスパッチャ）: x86-64のシグナルフレーム構築と、`sa_restorer`/`rt_sigreturn`による復帰パス
- [arch/x86/kernel/signal_64.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/signal_64.c) — `x64_setup_rt_frame()`: `setup_rt_frame()`のディスパッチ先となる、実際の64ビットフレーム構築コード
- [include/linux/sched.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/sched.h) — `task_struct`: スレッドごとの`pending`、`blocked`、`real_blocked`、`saved_sigmask`
- [include/linux/sched/signal.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/sched/signal.h) — `signal_struct`: プロセスごとの`shared_pending`
- [include/linux/signal_types.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/signal_types.h) — `struct sigpending`の定義

### manページ {#man-pages}

- [signal(7)](https://man7.org/linux/man-pages/man7/signal.7.html) — シグナルの一覧、デフォルトの処理方法、標準シグナルとリアルタイムシグナルのキューイングの違い
- [sigaction(2)](https://man7.org/linux/man-pages/man2/sigaction.2.html) — `struct sigaction`と`SA_*`フラグ
- [kill(2)](https://man7.org/linux/man-pages/man2/kill.2.html) — `pid`による対象指定の規則（プロセス、プロセスグループ、ブロードキャスト）
- [rt_sigqueueinfo(2)](https://man7.org/linux/man-pages/man2/rt_sigqueueinfo.2.html) — `sigqueue(3)`の下にあるシステムコールで、データ付きのリアルタイムシグナルをキューイングするのに使う
- [signalfd(2)](https://man7.org/linux/man-pages/man2/signalfd.2.html) — `read()`可能なファイルディスクリプタのイベントとして配送されるシグナル
- [signal-safety(7)](https://man7.org/linux/man-pages/man7/signal-safety.7.html) — シグナルハンドラから安全に呼び出せる関数

### 関連ページ {#related-pages}

- [eventfdとsignalfd](eventfd-signalfd.md) — `struct signalfd_siginfo`や`epoll`統合パターンを含む、`signalfd`のより詳しい解説
- [パイプとFIFO](pipes.md) — 壊れたパイプに書き込んだときの`SIGPIPE`

### LWN記事 {#lwn-articles}

- [Ghosts of Unix past, part 3: Unfixable designs](https://lwn.net/Articles/414618/) — 元のシグナル設計がなぜ修正不能なほど複雑になったのか、そして`signalfd()`が配送を同期的にすることでその問題をどう回避しているのかについてのNeil Brownの記事
