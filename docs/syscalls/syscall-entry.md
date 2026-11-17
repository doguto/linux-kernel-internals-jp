# syscallのエントリパス {#syscall-entry-path}

> `syscall` 命令からカーネルコードへ — そして戻るまで

## x86-64のハードウェア機構 {#x86-64-hardware-mechanism}

ユーザー空間が `syscall` を実行すると、次のことが起こる。

1. **CPUが保存する**: `rip` → `rcx`、`rflags` → `r11`。その後IFをクリアする
2. **CPUがロードする**: `IA32_LSTAR` MSRから `rip` を（ブート時に `entry_SYSCALL_64` に設定される）
3. **CPUが切り替える**: `IA32_STAR` MSRからCS/SSを（カーネルセグメント）

`rax` レジスタはsyscall番号を保持する。引数は `rdi`、`rsi`、`rdx`、`r10`、`r8`、`r9` に入る（注意: `rcx` ではなく `r10` である — `rcx` は `syscall` によって上書きされる）。

戻り値は `rax` で返ってくる。エラー時には、カーネルはerrnoを符号反転した値を返し、ユーザー空間のglibcがそれを変換する。

## entry_SYSCALL_64: アセンブリのエントリポイント {#entry_syscall_64-the-assembly-entry-point}

```asm
/* arch/x86/entry/entry_64.S */
SYM_CODE_START(entry_SYSCALL_64)
    UNWIND_HINT_ENTRY
    ENDBR

    swapgs                  /* カーネルのGSをロードする（per-CPUデータを指す） */
    /* per-CPUの cpu_current_top_of_stack を使ってカーネルスタックを設定する */
    movq    %rsp, PER_CPU_VAR(cpu_tss_rw + TSS_sp2)
    SWITCH_TO_KERNEL_CR3 scratch_reg=%rsp
    movq    PER_CPU_VAR(cpu_current_top_of_stack), %rsp

    /* pt_regs をプッシュする — ユーザーのレジスタをすべて保存 */
    pushq   $__USER_DS                  /* ss */
    pushq   PER_CPU_VAR(cpu_tss_rw + TSS_sp2)  /* rsp */
    pushq   %r11                        /* rflags */
    pushq   $__USER_CS                  /* cs */
    pushq   %rcx                        /* rip */
    pushq   %rax                        /* syscall番号（orig_rax） */
    PUSH_AND_CLEAR_REGS rax=$-ENOSYS    /* デフォルトの戻り値として rax=-ENOSYS */

    /* ...（IBRS緩和策、コンテキストトラッキング）... */

    movq    %rsp, %rdi                  /* 第1引数として pt_regs へのポインタ */
    movslq  %eax, %rsi                  /* syscall番号を符号拡張して第2引数へ */
    call    do_syscall_64
    /* ... 復帰パス ... */
SYM_CODE_END(entry_SYSCALL_64)
```

## struct pt_regs: 保存されたレジスタ状態 {#struct-pt_regs-saved-register-state}

```c
/* arch/x86/include/asm/ptrace.h — フィールド名には接頭辞が付かない
 * （rax, rcx, ... ではなく ax, cx, ...）。レジスタ幅は64ビットのままであり、
 * 説明用コードで "r" を付けるのはよくあるが誤った習慣である。 */
struct pt_regs {
    unsigned long r15;
    unsigned long r14;
    unsigned long r13;
    unsigned long r12;
    unsigned long bp;
    unsigned long bx;
    unsigned long r11;
    unsigned long r10;
    unsigned long r9;
    unsigned long r8;
    unsigned long ax;       /* syscall番号 / 戻り値 */
    unsigned long cx;       /* 保存された rip（syscall 命令による） */
    unsigned long dx;
    unsigned long si;
    unsigned long di;
    unsigned long orig_ax;  /* 元のsyscall番号（呼び出し前の ax） */
    unsigned long ip;       /* ユーザー空間の命令ポインタ */
    u16           cs;       /* 実際にはFRED CS拡張との共用体 */
    unsigned long flags;
    unsigned long sp;       /* ユーザー空間のスタックポインタ */
    u16           ss;       /* 実際にはFRED SS拡張との共用体 */
};
```

## do_syscall_64: ディスパッチ {#do_syscall_64-the-dispatch}

```c
/* arch/x86/entry/syscall_64.c */
__visible noinstr bool do_syscall_64(struct pt_regs *regs, int nr)
{
    nr = syscall_enter_from_user_mode(regs, nr);

    instrumentation_begin();
    add_random_kstack_offset();

    if (!do_syscall_x64(regs, nr) && !do_syscall_x32(regs, nr) && nr != -1) {
        /* syscall番号が範囲外 */
        regs->ax = __x64_sys_ni_syscall(regs);  /* -ENOSYS */
    }

    instrumentation_end();
    syscall_exit_to_user_mode(regs);
    /* ... SYSRETかIRETかの判断、後述 ... */
}

static __always_inline bool do_syscall_x64(struct pt_regs *regs, int nr)
{
    unsigned int unr = nr;

    if (likely(unr < NR_syscalls)) {
        unr = array_index_nospec(unr, NR_syscalls);  /* Spectre緩和策 */
        regs->ax = x64_sys_call(regs, unr);           /* ディスパッチ！ */
        return true;
    }
    return false;
}
```

`do_syscall_64()` は `bool` を返す — その最後の数行（上では省略）は、ユーザー空間への復帰に高速な `SYSRET`
命令を安全に使えるか、それとも低速な `IRET` パスが必要かを判断する。

## x64_sys_call: ディスパッチのswitch {#x64_sys_call-the-dispatch-switch}

```c
/* arch/x86/entry/syscall_64.c */
#define __SYSCALL(nr, sym) case nr: return __x64_##sym(regs);
long x64_sys_call(const struct pt_regs *regs, unsigned int nr)
{
    switch (nr) {
    #include <asm/syscalls_64.h>
    default: return __x64_sys_ni_syscall(regs);
    }
}
```

同じファイルでは、同じ `syscalls_64.h` から `sys_call_table[]` 配列も構築されるが、その定義の上にあるカーネル自身のコメントによれば、
"is no longer used for system calls;
`kernel/trace/trace_syscalls.c` still wants to know the system call address."（訳: もはやシステムコールには使われていないが、`kernel/trace/trace_syscalls.c` がまだシステムコールのアドレスを知りたがっている）という。実際のディスパッチは
上のswitch文で行われる。

`syscalls_64.h` は `syscall_64.tbl` から生成される。

```
# syscall_64.tbl (arch/x86/entry/syscalls/syscall_64.tbl)
# <number> <abi>  <name>           <entry point>
0          common  read              sys_read
1          common  write             sys_write
2          common  open              sys_open
3          common  close             sys_close
...
62         common  kill              sys_kill
...
435        common  clone3            sys_clone3
```

## 引数の受け渡しとコピー {#argument-passing-and-copying}

syscallの引数はレジスタで渡される。ユーザー空間を指すポインタについては、カーネルはそれを**コピー**して取り込まなければならない — ユーザー空間のポインタを直接参照外しすることは決してできない。

```c
/* ユーザー空間からのコピー */
copy_from_user(kernel_buf, user_ptr, size)
    → access_ok(user_ptr, size)        /* アドレスはユーザー範囲内か？ */
    → __copy_from_user()               /* アーキテクチャ固有のコピー */
    → コピーされなかったバイト数を返す（0 = 成功）

/* ユーザー空間へのコピー */
copy_to_user(user_ptr, kernel_buf, size)

/* ユーザー空間から文字列をコピーする（NUL または len で停止） */
strncpy_from_user(kernel_buf, user_ptr, max_len)

/* 単一値用のヘルパー（インライン化、最適化済み） */
get_user(x, user_ptr)   /* x = *user_ptr */
put_user(x, user_ptr)   /* *user_ptr = x */
```

なぜコピーが必須なのか？

- ユーザーのポインタはNULLや無効な値かもしれない
- カーネルとユーザーのアドレス空間は分離されている（KPTI）
- ユーザー空間は、チェックの後でポインタの指すメモリを変更できる（TOCTOU）

## KPTIとCR3の切り替え {#kpti-and-cr3-switching}

Kernel Page Table Isolation（KPTI）は、ユーザーモードとカーネルモードで異なるページテーブルを使うことでMeltdownを防ぐ。

```
User mode:  CR3 → user page tables (kernel not mapped, only entry stubs)
                  ↓ syscall instruction
Kernel mode: SWITCH_TO_KERNEL_CR3 (expensive TLB flush!)
             CR3 → kernel page tables (full mapping)
                  ↓ sysretq / swapgs
User mode:  SWITCH_TO_USER_CR3
```

PCIDをサポートする最新のCPUでは、KPTIはProcess Context IDを使って各アドレス空間のTLBエントリにタグを付けたまま保持し、TLBの全フラッシュを回避する。

## vDSO: 高速な操作のためにsyscallを回避する {#the-vdso-avoiding-syscalls-for-fast-operations}

一部のsyscallは毎秒何百万回も呼び出される（例: `gettimeofday`、`clock_gettime`）。カーネルは仮想動的共有オブジェクト（vDSO）をエクスポートする — これはすべてのプロセスにマップされる小さなコード片であり、カーネルモードに入らずにこれらを実装する。

```bash
# プロセスにマップされたvDSOを見る
cat /proc/self/maps | grep vdso
# 7fff12345000-7fff12346000 r-xp 00000000 00:00 0  [vdso]

# vDSO内の関数
nm /proc/self/maps  # これは動かないが、代わりに:
objdump -T /lib/x86_64-linux-gnu/vdso.so.1 2>/dev/null | grep -i clock
```

vDSOは、カーネルがアトミックに更新する共有メモリページ（`vvar`）からカーネルの時刻管理データを読む。vDSO関数は特権レベルの切り替えなしにデータを直接読む。

```c
/* lib/vdso/gettimeofday.c — アーキテクチャ共通であり、x86固有ではない。
 * 古いカーネルの単一の struct vdso_data は、現在は2つに分割されている:
 * vdso_time_data（最上位の vvar ページ、アーキテクチャのクロックソース状態）と
 * vdso_clock（クロックごとの basetime/mult/shift、クロックごとに1インスタンス。
 * 時間namespaceが各namespaceに独自のクロックデータを持たせたため必要になった）。 */
bool do_hres(const struct vdso_time_data *vd, const struct vdso_clock *vc,
             clockid_t clk, struct __kernel_timespec *ts)
{
    u64 sec, ns;
    u32 seq;

    do {
        /* 時間namespaceに属するタスクは独自の vdso_clock を持ち、
         * ここで別の（やはりseqlockで保護された）読み取りパスを通る。 */
        if (vdso_read_begin_timens(vc, &seq))
            return do_hres_timens(vd, vc, clk, ts);

        /* TSCの読み取りとサイクル差分からnsへの計算は、現在この共有
         * ヘルパーにあり、すべてのクロック読み取りパス（hres、coarse、timens）で使われる: */
        if (!vdso_get_timestamp(vd, vc, clk, &sec, &ns))
            return false;
    } while (vdso_read_retry(vc, seq));  /* seqlockのリトライ */

    vdso_set_timespec(ts, sec, ns);  /* sec/ns -> tv_sec/tv_nsec、nsの桁あふれを sec に繰り上げる */
    return true;
}
```

（簡略化している: 実際の関数には早期の `__arch_vdso_hres_capable()` による脱出もあり、`vdso_get_timestamp()` 自体が、古いバージョンでは `do_hres()` に直接インライン化されていたTSCの読み取りとクロックソースの有効性チェックを行う。）

vDSOを通じて高速パス化されているsyscall（リングの切り替えなし）:

- `clock_gettime(CLOCK_REALTIME/MONOTONIC/...)`
- `gettimeofday()`
- `clock_getres()`
- `getcpu()`

## Seccomp: syscallの制限 {#seccomp-restricting-syscalls}

seccompフィルタは、すべてのsyscallでディスパッチテーブルの前に実行される。

```c
/* 簡略化: 実際の連鎖はいくつかの関数を経由する —
 * syscall_enter_from_user_mode_work()（include/linux/entry-common.h）が
 * SYSCALL_WORK_SECCOMP ビットをチェックして syscall_trace_enter() を呼び、
 * それが kernel/seccomp.c の __seccomp_filter() に到達する: */
unsigned long work = READ_ONCE(current_thread_info()->syscall_work);

if (work & SYSCALL_WORK_SECCOMP) {
    struct seccomp_data sd;
    struct seccomp_filter *match = NULL;

    populate_seccomp_data(&sd);
    u32 filter_ret = seccomp_run_filters(&sd, &match);
    u32 action = filter_ret & SECCOMP_RET_ACTION_FULL;
    /* action は次のいずれか: SECCOMP_RET_ALLOW, SECCOMP_RET_KILL_(PROCESS|THREAD),
       SECCOMP_RET_ERRNO, SECCOMP_RET_TRACE, SECCOMP_RET_LOG, SECCOMP_RET_TRAP */
}
```

## シグナルと再開 {#signals-and-restarts}

一部のsyscallはシグナルによって中断されうる（`EINTR`）。カーネルはこれらの自動再開をサポートする。

```c
/* ERESTARTSYS: シグナルハンドラがなければ再開し、あれば EINTR を返す */
return -ERESTARTSYS;

/* ERESTARTNOINTR: 常に再開し、ユーザー空間からは透過的 */
return -ERESTARTNOINTR;

/* ERESTARTNOHAND: ハンドラがインストールされていなければ再開する */
return -ERESTARTNOHAND;
```

これらの特殊なコード（`< -MAX_ERRNO`）がユーザー空間に届くことはない — シグナルパスがこれらを `EINTR` に変換するか、syscallを再度キューに入れる。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [arch/x86/entry/entry_64.S](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/entry/entry_64.S) — `entry_SYSCALL_64`: アセンブリのsyscallエントリポイント（swapgs、`pt_regs` の構築、`do_syscall_64` の呼び出し）
- [arch/x86/entry/syscall_64.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/entry/syscall_64.c) — `do_syscall_64()` と `do_syscall_x64()`: 現在のディスパッチパス。注意: 現在のメインラインでは、これらの関数があるのはこのファイル（`common.c` ではない）であり、ディスパッチ自体は `switch` 文である `x64_sys_call()` を通る — `sys_call_table[]` は `kernel/trace/trace_syscalls.c` のためだけに残されており、ディスパッチには使われない
- [arch/x86/entry/syscalls/syscall_64.tbl](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/entry/syscalls/syscall_64.tbl) — `syscalls_64.h` の生成元となるsyscall番号テーブル
- [arch/x86/include/asm/ptrace.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/ptrace.h) — `struct pt_regs`: 保存されたレジスタフレーム
- [include/linux/uaccess.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/uaccess.h) — `copy_from_user()`、`copy_to_user()`、`access_ok()`
- [kernel/seccomp.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/seccomp.c) — `seccomp_run_filters()`: syscallエントリ時のBPFフィルタ評価
- [lib/vdso/gettimeofday.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/vdso/gettimeofday.c) — `do_hres()`: アーキテクチャ共通のvDSO高分解能クロック読み取り
- [include/linux/errno.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/errno.h) — `ERESTARTSYS`、`ERESTARTNOINTR`、`ERESTARTNOHAND`: 内部的なsyscall再開コード

### manページ {#man-pages}

- [`syscall(2)`](https://man7.org/linux/man-pages/man2/syscall.2.html) — 生のsyscall呼び出しと、x86-64のレジスタ呼び出し規約
- [`vdso(7)`](https://man7.org/linux/man-pages/man7/vdso.7.html) — 仮想動的共有オブジェクト

### 関連ページ {#related-pages}

- [SYSCALL_DEFINEとディスパッチ](syscall-define.md) — Cでのsyscallの定義
- [新しいsyscallの追加](adding-syscall.md) — カーネルコントリビュータ向けの実践ガイド
- [vDSOと仮想システムコール](vdso.md) — 共有の `vvar` 時刻管理ページと高速パスでのクロック読み取り
- [syscallの再開メカニズム](restart-block.md) — `ERESTARTSYS`、`ERESTART_RESTARTBLOCK`、`restart_syscall()`
- [32ビット互換syscall](compat.md) — 真の32ビットユーザー空間を扱うia32 compat層。`do_syscall_x32()` のx32パス（64ビットモード、ILP32ポインタ）と概念的には兄弟関係にあるが、`do_syscall_64()`/`do_syscall_x32()` を経由せず、独自の別のエントリポイント（`ia32_sys_call()`）を通じてディスパッチされる

### LWN記事 {#lwn-articles}

- [LWN: The current state of kernel page-table isolation](https://lwn.net/Articles/741878/) — Jonathan CorbetによるKPTIの解説。このページで説明したCR3切り替えの背景にあるMeltdown緩和策

### 外部資料 {#external}

- [Kernel Entries](https://docs.kernel.org/arch/x86/entry_64.html) — `arch/x86/entry/entry_64.S` についての正式なカーネルドキュメント。swapgsとx86-64のさまざまなエントリパスを扱う
- [Page Table Isolation (PTI)](https://docs.kernel.org/arch/x86/pti.html) — このページで説明したCR3切り替えの背景にあるMeltdown緩和策
- [Adding a New System Call](https://docs.kernel.org/process/adding-syscalls.html) — コントリビュータ向けのカーネルドキュメント
