# ptraceとsyscallのインターセプト {#ptrace-and-syscall-interception}

> strace、デバッガ、seccomp-notifyがsyscallをインターセプトする仕組み

## ptraceの概要 {#ptrace-overview}

`ptrace(2)` は、デバッガ（`gdb`）、トレーサ（`strace`）、システムコールスーパーバイザを支える
カーネルの仕組みである。あるプロセス（トレーサ）が別のプロセス（トレーシー）のレジスタ状態とメモリを調べたり変更したりでき、
特定のイベントでトレーシーを一時停止させることができる。

アタッチ方法は2つある。

```c
/* トレーシー側がオプトインする — デバッグ対象の子をforkするプロセスが使う */
ptrace(PTRACE_TRACEME, 0, NULL, NULL);

/* トレーサが既存のプロセスにアタッチする（SIGSTOP を送る） */
ptrace(PTRACE_ATTACH, target_pid, NULL, NULL);
/* 現代的な同等手段（安定化のための SIGSTOP を必要としない） */
ptrace(PTRACE_SEIZE, target_pid, NULL, 0);
```

アタッチ後、トレーシーは監視対象のイベント（syscall、シグナル、exec、
fork、clone）に到達すると**ptrace-stop**に入る。ptrace-stop中のトレーシーは一時停止しており、トレーサは `waitpid()` を呼んで停止理由を
取得し、再開する前に状態を調べたり変更したりできる。

## PTRACE_SYSCALL: straceの仕組み {#ptrace_syscall-how-strace-works}

```c
/* トレーシーを再開するが、次のsyscallのエントリまたはイグジットで再び停止させる */
ptrace(PTRACE_SYSCALL, pid, NULL, NULL);
```

これにより、カーネルはトレーシーを**syscallごとに2回**停止させる。1回はエントリ時（syscallの実行前）、
もう1回はイグジット時（カーネルが戻り値を設定した後）である。`strace` はこれを使ってすべてのsyscallを記録する。

```
waitpid(pid, &status)     /* wait for entry-stop */
ptrace(PTRACE_GETREGS, pid, NULL, &regs)
/* regs.orig_rax = syscall number, regs.rdi/rsi/rdx = args */
ptrace(PTRACE_SYSCALL, pid, NULL, NULL)  /* resume to exit-stop */

waitpid(pid, &status)     /* wait for exit-stop */
ptrace(PTRACE_GETREGS, pid, NULL, &regs)
/* regs.rax = return value */
ptrace(PTRACE_SYSCALL, pid, NULL, NULL)  /* resume to next entry-stop */
```

`strace` はsyscallごとに `waitpid()` を2回呼ぶので、トレーシングによってコンテキストスイッチの回数は倍になり、
コストが高い。syscallを多用するワークロードでは、トレースされたプロセスは100倍遅くなることもある。

## syscall-stop: エントリとイグジット {#syscall-stops-entry-and-exit}

カーネルは、`include/linux/ptrace.h` で `static inline` として宣言された2つの関数を介して、syscallの境界を
ptraceに通知する（どちらも共通の `ptrace_report_syscall()` ヘルパーを呼ぶ）。

- `ptrace_report_syscall_permit_entry(regs)` — syscallハンドラの実行前に `syscall_enter_from_user_mode_work()` から呼ばれる。`bool` を返し、falseが返された場合、呼び出し元はsyscallをディスパッチせずに完全に中止しなければならない。
- `ptrace_report_syscall_exit(regs, step)` — 戻り値が設定された後に `syscall_exit_to_user_mode_work()` から呼ばれる。`step` が非ゼロ（シングルステップ/ブロックステップが有効）の場合、通常のsyscall-stopのパスではなく `user_single_step_report()` を介して報告する。

`ptrace_report_syscall()` は `ptrace_notify()` を呼ぶ。これはトレーシーの停止コードを `(SIGTRAP | 0x80)` に設定し（最上位ビットは
syscall-stopを通常のシグナル停止と区別する）、トレーサに `SIGCHLD` を送る。

## syscall引数の読み取りと変更 {#reading-and-modifying-syscall-args}

エントリ停止時点では、syscallの引数は `pt_regs` にある。

```c
struct user_regs_struct regs;
ptrace(PTRACE_GETREGS, pid, NULL, &regs);

/* syscall番号 */
long syscall_nr = regs.orig_rax;

/* 引数（syscallにおけるx86-64の呼び出し規約） */
unsigned long arg1 = regs.rdi;
unsigned long arg2 = regs.rsi;
unsigned long arg3 = regs.rdx;
unsigned long arg4 = regs.r10;
unsigned long arg5 = regs.r8;
unsigned long arg6 = regs.r9;
```

syscall番号を変更するには（例: `unlink` を何もしない処理に置き換える）:

```c
regs.orig_rax = __NR_getpid;   /* 無害なsyscallに差し替える */
ptrace(PTRACE_SETREGS, pid, NULL, &regs);
```

syscallを完全に取り消すには、`orig_rax` を `-1` に設定する。`do_syscall_64()` が
`__x64_sys_ni_syscall()`（`-ENOSYS`）にフォールバックするのは、`-1` *以外の*範囲外のsyscall番号の場合だけである — `-1`
の番号はディスパッチを完全にスキップし、`regs->ax` はトレーサが設定したとおりのまま残る。これが、
ptraceを介して戻り値を注入したりsyscallを無効化したりするための実際の仕組みである。`orig_rax = -1` を設定し、
トレース対象のプロセスに見せたい戻り値を `rax` に設定すればよい。

## SYSCALL_WORK_SYSCALL_TRACE {#syscall_work_syscall_trace}

ptraceのsyscall-stopを引き起こすフラグは、タスクの `thread_info.syscall_work` にある
`SYSCALL_WORK_SYSCALL_TRACE` である。これは `PTRACE_SYSCALL` リクエストが発行されたときに `ptrace_resume()` が設定し
（アタッチ時ではない — 単なる `PTRACE_CONT` では設定されない）、トレーサがデタッチするときに `__ptrace_unlink()` でクリアされる。
カーネルはこのフラグを `syscall_enter_from_user_mode()` と
`syscall_exit_to_user_mode()` でチェックする。

```c
/* include/linux/entry-common.h（簡略化 — 実際の関数はsyscall user dispatchと
 * rseqのタイムスライス拡張も扱うが、ここでは省略） */
static __always_inline long syscall_trace_enter(struct pt_regs *regs, unsigned long work,
                                                 long syscall)
{
    if (work & (SYSCALL_WORK_SYSCALL_TRACE | SYSCALL_WORK_SYSCALL_EMU)) {
        if (!arch_ptrace_report_syscall_permit_entry(regs) ||
            (work & SYSCALL_WORK_SYSCALL_EMU))
            return false;   /* 呼び出し元はsyscallを中止しなければならない */
        work = READ_ONCE(current_thread_info()->syscall_work); /* ptrace が変更したかもしれない */
    }
    if (work & SYSCALL_WORK_SECCOMP && !__seccomp_permit_syscall())
        return false;
    if (work & SYSCALL_WORK_SYSCALL_TRACEPOINT)
        trace_syscall_enter(regs);
    if (audit_context())
        syscall_enter_audit(regs);
    return true;   /* syscallを続行する */
}
```

`arch_ptrace_report_syscall_permit_entry()` はアーキテクチャによるオーバーライドポイントであり、アーキテクチャが独自に定義しない限り、前節の `ptrace_report_syscall_permit_entry()` にそのままフォールバックする。

同じワークフラグの仕組みは、`SYSCALL_WORK_SECCOMP`（seccompフィルタリング用）と
`SYSCALL_WORK_SYSCALL_AUDIT`（auditサブシステム用）も運ぶので、3つすべてが共存できる。

## PTRACE_GET_SYSCALL_INFO（Linux 5.3） {#ptrace_get_syscall_info-linux-53}

Linux 5.3より前は、トレーサは `PTRACE_GETREGS` を呼び、waitpidのステータスから停止の種類を推測しなければならなかった。
Linux 5.3は `PTRACE_GET_SYSCALL_INFO`（コミット `201766a20e30f982ccfe36bebfad9602c3ff574a`）を追加した。これは構造化された記述を返す。

```c
/* include/uapi/linux/ptrace.h */
struct ptrace_syscall_info {
    __u8  op;          /* PTRACE_SYSCALL_INFO_NONE/ENTRY/EXIT/SECCOMP */
    __u8  reserved;
    __u16 flags;
    __u32 arch;
    __u64 instruction_pointer;
    __u64 stack_pointer;
    union {
        struct { /* PTRACE_SYSCALL_INFO_ENTRY または SECCOMP */
            __u64 nr;
            __u64 args[6];
        } entry;
        struct { /* PTRACE_SYSCALL_INFO_EXIT */
            __s64 rval;
            __u8  is_error;
        } exit;
        struct { /* PTRACE_SYSCALL_INFO_SECCOMP */
            __u64 nr;
            __u64 args[6];
            __u32 ret_data;
            __u32 reserved2;
        } seccomp;
    };
};

/* 使用例 */
struct ptrace_syscall_info info;
ptrace(PTRACE_GET_SYSCALL_INFO, pid, sizeof(info), &info);
if (info.op == PTRACE_SYSCALL_INFO_ENTRY) {
    printf("syscall %llu, arg0=%llu\n", info.entry.nr, info.entry.args[0]);
}
```

これは、現代の `strace`（5.1以降）が `PTRACE_GETREGS` の代わりに使っているインターフェースである。

## ptraceイベント: PTRACE_O_TRACEFORKとその仲間 {#ptrace-events-ptrace_o_tracefork-and-friends}

トレーサは、ptraceオプションを設定することで、syscall-stop以外のイベントも要求できる。

```c
ptrace(PTRACE_SETOPTIONS, pid, 0,
       PTRACE_O_TRACEFORK   |  /* fork() で停止 */
       PTRACE_O_TRACEVFORK  |  /* vfork() で停止 */
       PTRACE_O_TRACECLONE  |  /* clone() で停止 */
       PTRACE_O_TRACEEXEC   |  /* execve() の後に停止 */
       PTRACE_O_TRACEEXIT   |  /* プロセス終了の前に停止 */
       PTRACE_O_TRACESECCOMP); /* SECCOMP_RET_TRACE で停止 */
```

これらのイベントのいずれかが発生すると、`waitpid()` は
`(status >> 8) == (SIGTRAP | (PTRACE_EVENT_xxx << 8))` となるステータスを返す。たとえば、forkイベントでは
`PTRACE_EVENT_FORK` が返る。その後、トレーサは `ptrace(PTRACE_GETEVENTMSG, pid, NULL, &msg)` を呼んで
新しい子のPIDを取得できる。

## seccomp-notify: ptraceの現代的な代替 {#seccomp-notify-a-modern-alternative-to-ptrace}

`SECCOMP_RET_USER_NOTIF`（Linux 5.0で追加、`include/uapi/linux/seccomp.h`）により、特権を持つスーパーバイザは
ptraceのオーバーヘッドなしにsyscallをインターセプトできる。トレーシーを停止してトレーサを起こす代わりに、
カーネルはファイルディスクリプタに通知を送り、トレーシーはスーパーバイザが応答するまでスリープする。

### フィルタのインストール {#installing-the-filter}

```c
/* 監視対象のプロセス内で（または exec の前に） */
int notif_fd = seccomp(SECCOMP_SET_MODE_FILTER, SECCOMP_FILTER_FLAG_NEW_LISTENER,
                       &prog);
/* notif_fd は新しいファイルディスクリプタ — SCM_RIGHTS でスーパーバイザに渡す */
```

BPFプログラムは、インターセプトするsyscallに対しては `SECCOMP_RET_USER_NOTIF` を、
それ以外のすべてに対しては `SECCOMP_RET_ALLOW` を返す。

### 通知の受信と応答 {#receiving-and-responding-to-notifications}

```c
/* include/uapi/linux/seccomp.h */
struct seccomp_notif {
    __u64   id;         /* 一意な通知ID */
    __u32   pid;        /* 通知を引き起こしたタスクのPID */
    __u32   flags;
    struct seccomp_data data;  /* syscall番号 + 引数 + アーキテクチャ */
};

struct seccomp_notif_resp {
    __u64   id;         /* リクエストと一致しなければならない */
    __s64   val;        /* 注入する戻り値 */
    __s32   error;      /* 注入するerrno（0 = val を使う） */
    __u32   flags;      /* SECCOMP_USER_NOTIF_FLAG_CONTINUE */
};

/* スーパーバイザのループ */
struct seccomp_notif req;
struct seccomp_notif_resp resp = {};

ioctl(notif_fd, SECCOMP_IOCTL_NOTIF_RECV, &req);   /* ブロックする */
/* req.data.nr（syscall番号）と req.data.args[] を調べる */

resp.id    = req.id;
resp.error = 0;
resp.val   = 42;   /* 戻り値を注入する */
ioctl(notif_fd, SECCOMP_IOCTL_NOTIF_SEND, &resp);
```

監視対象プロセス内の一時停止していたタスクが起き、syscallの戻り値として `42` を受け取る。これにより、
ptraceが必要とするsyscallごとの2回のコンテキストスイッチが回避される。

`bubblewrap` や `systemd` の `RootDirectory=` サンドボックスなどのコンテナマネージャは、seccomp-notifyを使って
syscallのエミュレーション（例: 非特権コンテナ向けに `mount()` を偽装する）を実装している。

### SECCOMP_USER_NOTIF_FLAG_CONTINUE {#seccomp_user_notif_flag_continue}

スーパーバイザが `resp.flags` に `SECCOMP_USER_NOTIF_FLAG_CONTINUE` を設定すると、カーネルは戻り値を注入する代わりに
syscallを通常どおり実行する。これは、スーパーバイザが呼び出しをブロックせずに記録だけしたい場合に
使われる。

## perfによるsyscallトレーシング {#perf-syscall-tracing}

```bash
# 生のsyscall tracepoint（全syscall、エントリのみ）
perf record -e raw_syscalls:sys_enter -- my_program

# きめ細かく: openat() のエントリのみ
perf record -e syscalls:sys_enter_openat -- my_program

# ls 実行中のsyscallを数える
perf stat -e 'syscalls:sys_enter_*' -- ls 2>&1 | head -20
```

`raw_syscalls:sys_enter` はすべてのsyscallで発火し、番号と第1引数を提供する。
`syscalls:sys_enter_<name>` tracepointは指定したsyscallでのみ発火し、すべての引数を
シンボリックにデコードする。どちらも標準のftrace tracepointであり、`SYSCALL_DEFINE` が生成する
`SYSCALL_METADATA` マクロによって定義される。

## syscall用のeBPF tracepoint {#ebpf-tracepoints-for-syscalls}

BPFプログラムは、libbpfを使ってsyscallのtracepointにアタッチできる。

```c
/* BPFプログラム — openat() のエントリにアタッチする */
SEC("tracepoint/syscalls/sys_enter_openat")
int trace_openat_entry(struct trace_event_raw_sys_enter *ctx)
{
    /* ctx->args[0] = dirfd, ctx->args[1] = ファイル名へのポインタ,
       ctx->args[2] = flags, ctx->args[3] = mode */
    char fname[256];
    bpf_probe_read_user_str(fname, sizeof(fname), (void *)ctx->args[1]);
    bpf_printk("openat: %s\n", fname);
    return 0;
}
```

構造体 `trace_event_raw_sys_enter` は、
`/sys/kernel/tracing/events/syscalls/sys_enter_openat/format` にあるtracepointのフォーマットファイルからBTFによって生成される。`args[]` 配列には、
引数の順に生のレジスタ値が入っている。

syscallのイグジットの場合:

```c
SEC("tracepoint/syscalls/sys_exit_openat")
int trace_openat_exit(struct trace_event_raw_sys_exit *ctx)
{
    long ret = ctx->ret;   /* 戻り値（負 = -errno） */
    if (ret < 0)
        bpf_printk("openat failed: %ld\n", ret);
    return 0;
}
```

## ftrace: syscallハンドラ関数のトレーシング {#ftrace-tracing-syscall-handler-functions}

```bash
# function トレーサで sys_openat 関数をトレースする
echo sys_openat > /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace_pipe
```

functionトレーサがトレースするのは `__x64_sys_openat`（pt_regsラッパー）であり、`do_sys_openat2()`（
実際の実装）ではないことに注意すること。実装をトレースするには、明示的に追加する。

```bash
echo '__x64_sys_openat do_sys_openat2' > /sys/kernel/tracing/set_ftrace_filter
```

syscallの関数グラフ（syscall配下の呼び出しツリー全体を表示する）の場合:

```bash
echo function_graph > /sys/kernel/tracing/current_tracer
echo __x64_sys_openat > /sys/kernel/tracing/set_graph_function
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/ptrace.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/ptrace.h) — `ptrace_report_syscall_permit_entry()`、`ptrace_report_syscall_exit()`、および `ptrace_notify()` を呼ぶ共通の `ptrace_report_syscall()` ヘルパー
- [include/linux/entry-common.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/entry-common.h) — `syscall_trace_enter()`、`syscall_enter_from_user_mode_work()`、`syscall_exit_to_user_mode_work()`。`SYSCALL_WORK_SYSCALL_TRACE` をチェックし、ptrace、seccomp、auditを順に駆動する
- [include/uapi/linux/ptrace.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/ptrace.h) — `struct ptrace_syscall_info` と `PTRACE_GET_SYSCALL_INFO` リクエスト
- [include/uapi/linux/seccomp.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/seccomp.h) — `struct seccomp_notif` / `struct seccomp_notif_resp`、`SECCOMP_RET_USER_NOTIF`、`SECCOMP_USER_NOTIF_FLAG_CONTINUE`
- [kernel/seccomp.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/seccomp.c) — `seccomp_do_user_notification()` と、`SECCOMP_IOCTL_NOTIF_RECV`/`SECCOMP_IOCTL_NOTIF_SEND` のioctl処理
- [arch/x86/entry/syscall_64.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/entry/syscall_64.c) — `do_syscall_64()`/`do_syscall_x64()`: トレーサが変更した `orig_rax` がどのようにディスパッチされる（またはスキップされる）か
- コミット [`201766a20e30f982ccfe36bebfad9602c3ff574a`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=201766a20e30f982ccfe36bebfad9602c3ff574a)（"ptrace: add PTRACE_GET_SYSCALL_INFO request"）— Linux 5.3のマージウィンドウで取り込まれた

### manページ {#man-pages}

- [`ptrace(2)`](https://man7.org/linux/man-pages/man2/ptrace.2.html) — インターフェースの完全なリファレンス: syscall-stop、`PTRACE_SETOPTIONS`、`PTRACE_O_TRACESYSGOOD`、`PTRACE_GET_SYSCALL_INFO`
- [`seccomp(2)`](https://man7.org/linux/man-pages/man2/seccomp.2.html) — `SECCOMP_SET_MODE_FILTER`、`SECCOMP_FILTER_FLAG_NEW_LISTENER`、`SECCOMP_RET_USER_NOTIF`
- [`seccomp_unotify(2)`](https://man7.org/linux/man-pages/man2/seccomp_unotify.2.html) — ユーザー空間通知のプロトコル: `SECCOMP_IOCTL_NOTIF_RECV`/`SEND`、`struct seccomp_notif`/`seccomp_notif_resp`、`SECCOMP_USER_NOTIF_FLAG_CONTINUE`

### 関連ページ {#related-pages}

- [syscallのエントリパス](syscall-entry.md) — `entry_SYSCALL_64`、`pt_regs`、そしてこのページのトレーサフックが実行される場である `do_syscall_64()` のディスパッチ
- [SYSCALL_DEFINEとディスパッチ](syscall-define.md) — `SYSCALL_DEFINE` がsyscallをディスパッチテーブルにどう組み込むか
- [syscallの監査](audit.md) — `SYSCALL_WORK_SYSCALL_AUDIT` と、auditサブシステム独自のエントリ/イグジットフック

### 外部資料 {#external}

- [seccomp BPF (SECCOMP_MODE_FILTER)](https://docs.kernel.org/userspace-api/seccomp_filter.html) — `SECCOMP_RET_USER_NOTIF`、通知用の構造体、ioctlプロトコルについてのカーネルドキュメント
