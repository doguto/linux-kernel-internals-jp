# syscallの再開メカニズム {#syscall-restart-mechanisms}

> 中断されたsyscallが透過的に再開される仕組み — ERESTARTSYS、ERESTART_RESTARTBLOCK、restart_syscall()

## 問題 {#the-problem}

あるプロセスが低速なデバイスに対して `read()` を呼ぶ。カーネルはタスクをブロックし、
データを待つ。ブロック中にシグナルが届く。子プロセスからの `SIGCHLD` や、
タイマーからの `SIGALRM` である。カーネルはシグナルを配送するためにタスクを起こす。
では、`read()` はどうなるべきか？

考えられる結果は3つある。

1. ユーザー空間に `-EINTR` を返す。呼び出し元は、必要なら手動でリトライしなければならない。
2. シグナルによる中断がなかったかのように、`read()` を透過的に再開する。
3. 引数を調整したうえで `read()` を再開する — たとえば `nanosleep()` は、
   元の期間ではなく*残りの*時間だけスリープすべきである。

どの結果になるかは、syscallがどの再開コードを返すか、そしてシグナルハンドラが
`SA_RESTART` 付きでインストールされたかどうかで決まる。

## 4つの再開コード {#the-four-restart-codes}

これらは特別な負のエラー値であり、すべて `-MAX_ERRNO`（4096と定義）より小さい。
通常のエラー範囲の外にあるため、ユーザー空間に届くことはない
— シグナル配送パスが `sysretq` の前にこれらを横取りする。

```c
/* include/linux/errno.h */
#define ERESTARTSYS          512
#define ERESTARTNOINTR       513
#define ERESTARTNOHAND       514
#define ERESTART_RESTARTBLOCK 516
```

syscallから戻った後の `regs->ax` に入る負の値としては、次のようになる。

| コード | 値 | 意味 |
|------|-------|---------|
| `ERESTARTSYS` | -512 | シグナルハンドラに `SA_RESTART` が設定されていれば再開し、そうでなければ `-EINTR` を返す |
| `ERESTARTNOINTR` | -513 | `SA_RESTART` に関係なく常に再開する。ユーザー空間からは透過的 |
| `ERESTARTNOHAND` | -514 | ハンドラが実行されたら `-EINTR` を返す。ハンドラがインストールされていない場合のみ再開する |
| `ERESTART_RESTARTBLOCK` | -516 | `restart_syscall()` を介して再開する — 引数を調整しなければならないsyscall用 |

`EINTR`（-4）は再開コードではない。カーネルがsyscallを再開すべきでないと
判断したときにユーザー空間へ返される、通常のエラーである。

## シグナルパスが再開コードをどう使うか {#how-the-signal-path-uses-restart-codes}

シグナルハンドラが戻った後、アーキテクチャのシグナルコードは
`arch_do_signal_or_restart()`（x86: `arch/x86/kernel/signal.c`）を呼ぶ。これは
`regs->ax` を調べて、何をすべきかを決める。

```c
/* arch/x86/kernel/signal.c — 簡略化 */
static void handle_signal(struct ksignal *ksig, struct pt_regs *regs)
{
    /* ... ユーザースタック上にシグナルフレームを設定する ... */

    /*
     * syscallが中断された場合、再開するかどうかを決める:
     * regs->orig_ax はsyscall番号を保持し、regs->ax は
     * 中断されたsyscallの戻り値（ERESTART* コードのいずれか）を保持する。
     *
     * handle_signal() が実行されるとき、シグナルハンドラは実際に配送されている。
     * ここでは ERESTART_RESTARTBLOCK は ERESTARTNOHAND と同様に
     * -EINTR に変換される — syscallは再開されない。
     */
    if (regs->orig_ax != -1) {
        switch ((int)regs->ax) {
        case -ERESTART_RESTARTBLOCK:
        case -ERESTARTNOHAND:
            regs->ax = -EINTR;   /* ユーザー空間に EINTR を返す */
            break;
        case -ERESTARTSYS:
            if (!(ksig->ka.sa.sa_flags & SA_RESTART)) {
                regs->ax = -EINTR;
                break;
            }
            fallthrough;
        case -ERESTARTNOINTR:
            /* 再開: rip を巻き戻して syscall 命令を再実行する */
            regs->ax  = regs->orig_ax;
            regs->ip -= 2;   /* 2バイトの syscall 命令の分だけ戻す */
            break;
        }
    }
}
```

`ERESTART_RESTARTBLOCK` によって `restart_syscall()` が実行されるのは、**シグナルハンドラが
実行されない場合のみ**である（`arch_do_signal_or_restart()` 内のハンドラなしのパス）。

```c
/* ハンドラなしのパス（arch_do_signal_or_restart）でのみ: */
case -ERESTART_RESTARTBLOCK:
    regs->ax  = get_nr_restart_syscall(regs);
    regs->ip -= 2;
    /* rip はすでに syscall 命令を指している */
    break;
```

`get_nr_restart_syscall()` は常に単純な `__NR_restart_syscall` を返すわけではない。32ビット
compatタスク（`restart_block.arch_data & TS_COMPAT` — 上の構造体と同じ `arch_data` フィールド）
の場合は代わりに `__NR_ia32_restart_syscall` を返し、
x32 ABIの場合は `__X32_SYSCALL_BIT` をORで加える。`arch_data` を再開の設定時点で
取得しておかなければならないのはこのためである。CPUは、ネイティブかcompatかに応じて、
再ディスパッチに使う*正しい*syscall番号を必要とする。

シグナルハンドラが実際に配送される場合は、`handle_signal()` が代わりに
`ERESTART_RESTARTBLOCK` を `-EINTR` に変換する（上のswitchを参照）。

## struct restart_block {#struct-restart_block}

`struct restart_block` は `struct task_struct`（`include/linux/sched.h`）に直接
埋め込まれている。これは再開関数と、syscallの再開に必要なすべての引数を
格納する。

```c
/* include/linux/restart_block.h */
struct restart_block {
    unsigned long arch_data;   /* アーキテクチャ固有のフラグ。例: x86 の TS_COMPAT */
    long (*fn)(struct restart_block *);

    union {
        /* futex_wait / futex_wait_bitset */
        struct {
            u32 __user    *uaddr;
            u32            val;
            u32            flags;
            u32            bitset;
            ktime_t        time;
            u32 __user    *uaddr2;
        } futex;

        /* nanosleep / clock_nanosleep */
        struct {
            clockid_t           clockid;
            enum timespec_type  type;
            union {
                struct __kernel_timespec __user *rmtp;      /* ネイティブ（64ビット） */
                struct old_timespec32 __user    *compat_rmtp; /* 32ビット互換 */
            };
            ktime_t             expires;
        } nanosleep;

        /* poll / ppoll */
        struct {
            struct pollfd __user *ufds;
            int                   nfds;
            int                   has_timeout;
            struct timespec64     end_time;
        } poll;
    };
};
```

`restart_block` はタスクごとに1つあり、中断されたsyscallの間で使い回される。
`ERESTART_RESTARTBLOCK` を返す前に、syscallは `fn` と、適切な共用体メンバーに
調整済みの引数を書き込む。

## restart_syscall(): 再入ポイント {#restart_syscall-the-re-entry-point}

`restart_syscall()` は本物のsyscallである — syscall番号は `__NR_restart_syscall`
（x86-64では219）。引数はなく、必要なものはすべて
`current->restart_block` から読み取る。

```c
/* kernel/signal.c */
SYSCALL_DEFINE0(restart_syscall)
{
    struct restart_block *restart = &current->restart_block;
    return restart->fn(restart);
}
```

`fn` ポインタは、中断されたsyscallが `ERESTART_RESTARTBLOCK` を返す前に
設定する。`restart_syscall()` が発火すると、事前に書き込まれた `restart_block` を
引数にその関数を呼ぶ。`restart_block` にはすでに調整済みの引数
（残りのスリープ時間など）が入っている。

通常の動作でユーザー空間が `__NR_restart_syscall` を目にすることはない。これは
`nanosleep`、`poll`、`select`、`futex_wait` がシグナル配送後に再開されたとき、
`strace` の出力に `restart_syscall()` として現れる。

## nanosleep() の再開: 完全な解説 {#nanosleep-restart-a-complete-walkthrough}

```
1.  Userspace calls nanosleep({.tv_sec=5, .tv_nsec=0}, &rem)
    → kernel: sys_nanosleep()
    → hrtimer started, task sleeps

2.  3 seconds later: signal arrives, task woken up

3.  sys_nanosleep() calculates remaining time: 2 seconds

4.  Fills in restart_block:
        current->restart_block.fn              = hrtimer_nanosleep_restart;
        current->restart_block.nanosleep.clockid = CLOCK_MONOTONIC;  /* plain nanosleep() always sleeps against CLOCK_MONOTONIC; clock_nanosleep() could pass a different clock */
        current->restart_block.nanosleep.rmtp    = &rem (userspace pointer);
        current->restart_block.nanosleep.expires = now + 2_seconds_in_ns;

5.  Returns -ERESTART_RESTARTBLOCK

6.  Signal path (no handler installed / no-handler restart path):
        - Sees -ERESTART_RESTARTBLOCK in regs->ax
        - Rewrites regs->ax = get_nr_restart_syscall(regs)
          = __NR_restart_syscall (219) — this is a native 64-bit task,
            so no compat/x32 translation applies
        - Rewinds rip by 2 bytes
        (If a handler IS installed and runs, handle_signal() converts
         -ERESTART_RESTARTBLOCK to -EINTR instead — no restart occurs.)

7.  CPU re-executes syscall instruction with rax=219

8.  Kernel dispatches restart_syscall()
    → current->restart_block.fn(&current->restart_block)
    → hrtimer_nanosleep_restart()
    → starts a new hrtimer for the remaining 2 seconds
    → task sleeps again

9.  If no further signals: hrtimer_nanosleep_restart() returns 0 to userspace.
    If another signal arrives: the whole cycle repeats with the new
    remaining time.
```

ユーザー空間のプロセスから見ると、元の `nanosleep()` 呼び出しが最終的に
0を返す — 再開には気づかない。`rem` ポインタは、プログラムが調べる場合に備えて、
中断のたびにその時点の残り時間で更新される。

## どのsyscallがどのメカニズムを使うか {#which-syscalls-use-which-mechanism}

| 再開コード | 例 | ハンドラが実行された場合 | ハンドラが実行されない場合 |
|---|---|---|---|
| `ERESTARTSYS` | `read`、`write`、`accept`、`connect`、`waitpid`、`select`（短い待ち） | EINTR（SA_RESTART があれば再開） | 常に再開 |
| `ERESTARTNOINTR` | ページフォールト中に入る待ち状態、一部の内部的な待ち | 常に再開 | 常に再開 |
| `ERESTARTNOHAND` | `pause`、`sigsuspend`、`epoll_wait`（シグナルマスクの変更を伴う） | 常にEINTR | 常に再開 |
| `ERESTART_RESTARTBLOCK` | `nanosleep`、`clock_nanosleep`、`poll`、`ppoll`、`select`（タイムアウト付き）、`futex`（FUTEX_WAIT） | 常にEINTR | `restart_syscall()` を介して再開 |

`SA_RESTART` が制御するのは `ERESTARTSYS` の挙動だけである。`ERESTART_RESTARTBLOCK`
が `restart_syscall()` を介して再開するのは、**シグナルハンドラが実行されない場合のみ**である。
シグナルハンドラが実際に配送される場合、`ERESTART_RESTARTBLOCK` は `ERESTARTNOHAND`
と同様に `-EINTR` に変換される。違いを決めるのは、ハンドラが実際に実行されたかどうかであり、
`SA_RESTART` が設定されているかどうかではない。よくある間違いは、`SA_RESTART` 付きで
シグナルハンドラをインストールし、`nanosleep()` が再開されると期待することである — 再開はされず、
`-EINTR` が返る。ハンドラがなければ、元の期間ではなく残り時間で
`restart_syscall()` を介して再開される。

## 再開のデバッグ {#debugging-restarts}

### strace {#strace}

`strace` は、再開されたsyscallが呼び出されるたびに `restart_syscall()` を
明示的に表示する。

```
nanosleep({tv_sec=5, tv_nsec=0}, 0x7ffd...)  = ? ERESTART_RESTARTBLOCK (Interrupted by signal)
--- SIGALRM {si_signo=SIGALRM, si_code=SI_KERNEL} ---
rt_sigreturn()                               = 0
restart_syscall(<... resuming interrupted nanosleep ...>) = 0
```

### perf trace {#perf-trace}

`perf trace` は、再開を含むsyscallのエントリとイグジットのイベントを記録する。

```bash
# プロセスの全syscallをトレースし、restart_syscall のエントリを表示する
perf trace -p <pid> 2>&1 | grep -E 'restart|nanosleep|poll'
```

### /proc/pid/syscall {#procpidsyscall}

`/proc/<pid>/syscall` は、タスクが現在ブロックしているsyscallを表示する。
`219`（x86-64の場合）と表示されたら、そのタスクは `restart_syscall()` の中にいる。

```bash
cat /proc/$(pgrep myapp)/syscall
# 219 0x0 0x0 0x0 0x0 0x0 0x0 0x7ffd12345678 0xffffffffffffffff
# ^219 = __NR_restart_syscall
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/errno.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/errno.h) — `ERESTARTSYS`、`ERESTARTNOINTR`、`ERESTARTNOHAND`、`ERESTART_RESTARTBLOCK` の定義
- [include/linux/restart_block.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/restart_block.h) — `struct restart_block` の定義。`futex`、`nanosleep`、`poll` の共用体メンバーを含む
- [include/linux/sched.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/sched.h) — `struct task_struct` に直接埋め込まれた `restart_block`
- [include/linux/thread_info.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/thread_info.h) — `set_restart_fn()`: `restart->fn` を設定して `-ERESTART_RESTARTBLOCK` を返す
- [kernel/signal.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/signal.c) — `SYSCALL_DEFINE0(restart_syscall)` と `do_no_restart_syscall()`
- [arch/x86/kernel/signal.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/signal.c) — `handle_signal()` と `arch_do_signal_or_restart()`: `regs->ax` 内の `ERESTART*` コードが調べられ、振り分けられる場所。`get_nr_restart_syscall()`: `restart_block.arch_data` に基づいて正しい再開syscall番号（ネイティブ、32ビット互換用の `__NR_ia32_restart_syscall`、またはx32）を選ぶ
- [kernel/time/hrtimer.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/hrtimer.c) — `hrtimer_nanosleep_restart()`: `nanosleep()`/`clock_nanosleep()` が `restart_block.fn` に登録する再開ハンドラ

### manページ {#man-pages}

- [`signal(7)`](https://man7.org/linux/man-pages/man7/signal.7.html) — `SA_RESTART` のユーザー空間から見える効果については "Interruption of system calls and library functions by signal handlers" を参照
- [`restart_syscall(2)`](https://man7.org/linux/man-pages/man2/restart_syscall.2.html) — このページの解説でディスパッチ先となるsyscall。「手動で呼び出さない」という挙動に注意

### 関連ページ {#related-pages}

- [syscallのエントリパス](syscall-entry.md) — `syscall_exit_to_user_mode()`、シグナルパス、`pt_regs`
- [シグナル](../ipc/signals.md) — シグナル配送、`sigaction`、`SA_RESTART`
- [futexの内部](../locking/futex.md) — `FUTEX_WAIT`/`FUTEX_WAKE` の仕組みとfutexキーのハッシュ。このページの `restart_block.futex` メンバーは、シグナルで中断された `FUTEX_WAIT` の再開に必要な引数を格納する

### LWN記事 {#lwn-articles}

- [A new system call restart mechanism](https://lwn.net/Articles/17744/)（2002）— restart-blockメカニズムが追加された理由についてのJonathan Corbetの記事。`nanosleep()` は元の期間ではなく*調整済みの*（残りの）期間で再開する必要があり、単純な `-ERESTARTSYS` による最初からのリトライではそれを表現できない
- [\[braindump\]\[RFC\] signals and syscall restarts](https://lwn.net/Articles/528935/)（2012）— `ERESTARTSYS`/`ERESTARTNOINTR`/`SA_RESTART` のセマンティクスと、アーキテクチャのシグナル配送コードがそれらを誤って実装してきた事例についてのAl Viroの詳細な技術メモ
