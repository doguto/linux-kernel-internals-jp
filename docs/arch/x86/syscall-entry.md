# x86-64のシステムコールエントリ {#x86-64-syscall-entry}

> SYSCALL命令、entry_SYSCALL_64、vDSO、そしてシグナル配送

ユーザープロセスが`read()`、`write()`、`mmap()`、あるいはその他のシステムコールを呼ぶたびに、制御はユーザー空間からカーネルへ移り、また戻ってくる。x86-64では、この遷移は`SYSCALL`/`SYSRET`命令のペアによって処理される — これは古い`int 0x80`割り込み機構のオーバーヘッドを避ける高速パスである。

---

## SYSCALL命令 {#the-syscall-instruction}

`SYSCALL`はx86-64の命令（64ビットモードで利用可能）で、ring 3（ユーザー）からring 0（カーネル）への特権遷移を行う。割り込みベースのエントリとは異なり、`SYSCALL`はIDTを参照せず、完全な例外フレームもプッシュしない — システムコールエントリ専用に設計された、無駄を削ぎ落とした機構である。

### SYSCALLが行うこと（ハードウェアの動作） {#what-syscall-does-hardware-behavior}

```
When the CPU executes SYSCALL:

1. Save user RIP → RCX      (return address for SYSRET)
2. Save RFLAGS  → R11       (to restore on return)
3. Clear RFLAGS per SFMASK MSR (e.g., clears IF: no interrupts during entry trampoline)
4. Load CS from STAR MSR[47:32]  (kernel code segment selector)
5. Load SS from STAR MSR[47:32]+8 (kernel data segment selector)
6. Load RIP from LSTAR MSR   (→ entry_SYSCALL_64)
7. Does NOT switch the stack — RSP still points at user stack

Registers at the point entry_SYSCALL_64 begins:
  RAX = syscall number
  RDI = arg1, RSI = arg2, RDX = arg3, R10 = arg4, R8 = arg5, R9 = arg6
  RCX = saved user RIP (destroyed by SYSCALL)
  R11 = saved user RFLAGS (destroyed by SYSCALL)
  RSP = still pointing at user stack (not yet switched)
```

SYSCALLのABIはC ABIと1点だけ異なることに注意してほしい: 第4引数は`RCX`ではなく`R10`で渡される（`RCX`はSYSCALL命令自体によって上書きされるため）。

### MSRの設定 {#msr-setup}

3つのMSRがSYSCALLの動作を設定する。カーネルは`arch/x86/kernel/cpu/common.c`の`syscall_init()`の中でこれらを設定する。

| MSR | アドレス | 内容 |
|-----|---------|---------|
| `STAR` | `0xC0000081` | ビット47:32 = カーネルのCSセレクタ。ビット63:48 = ユーザーのCSセレクタ |
| `LSTAR` | `0xC0000082` | 64ビットSYSCALLのジャンプ先RIP（`entry_SYSCALL_64`のアドレス） |
| `SFMASK` | `0xC0000084` | SYSCALL時にクリアするRFLAGSのビット（`IF`、`TF`、`DF`、`AC`をクリアする） |

```c
/* arch/x86/kernel/cpu/common.c */
void syscall_init(void)
{
    wrmsr(MSR_STAR, 0, (__USER32_CS << 16) | __KERNEL_CS);
    wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);

    /*
     * システムコールエントリ時にクリアするフラグ: 方向フラグ、割り込みフラグ、
     * トラップフラグ、アライメントチェックフラグ。
     */
    wrmsrl(MSR_SYSCALL_MASK,
           X86_EFLAGS_TF|X86_EFLAGS_DF|X86_EFLAGS_IF|
           X86_EFLAGS_IOPL|X86_EFLAGS_AC|X86_EFLAGS_NT);
}
```

---

## entry_SYSCALL_64（`arch/x86/entry/entry_64.S`） {#entrysyscall64-archx86entryentry_64s}

`entry_SYSCALL_64`は、すべての64ビットシステムコールのアセンブリによるエントリポイントである。これは次のことを行わなければならない。
1. ユーザーの状態を保存する
2. カーネルスタックに切り替える
3. 割り込みを有効にする（SFMASKがIFをクリアしている。カーネルは状態を保存した後に有効化する）
4. `do_syscall_64()`を呼ぶ
5. ユーザーの状態を復元して戻る

```asm
/* arch/x86/entry/entry_64.S（簡略化した説明用のもの） */

SYM_CODE_START(entry_SYSCALL_64)
    /* ユーザーのRSPのままここに到着する。保存するまでRSPに触れてはならない */

    /* swapgs: GS.baseとKernelGSbase MSRを交換する。
     * これ以降、GSはper-CPUデータ領域を指す。
     * これによりper-CPU変数にアクセスしてカーネルスタックを見つけられる。 */
    swapgs

    /* ユーザーのRSPをper-CPU領域に保存し、カーネルスタックに切り替える。
     * cpu_current_top_of_stackは、このCPUのカーネルスタックの
     * トップを保持するper-CPU変数である。 */
    movq    %rsp, PER_CPU_VAR(cpu_tss_rw + TSS_sp2)
    SWITCH_TO_KERNEL_CR3 scratch_reg=%rsp   /* KPTI: カーネルのPGDに切り替える */
    movq    PER_CPU_VAR(cpu_current_top_of_stack), %rsp

    /* カーネルスタック上に合成した例外フレーム（struct pt_regs）をプッシュする */
    pushq   $__USER_DS                  /* SS */
    pushq   PER_CPU_VAR(cpu_tss_rw + TSS_sp2)  /* RSP（保存したユーザースタック） */
    pushq   %r11                        /* RFLAGS */
    pushq   $__USER_CS                  /* CS */
    pushq   %rcx                        /* RIP（ユーザーの戻りアドレス） */

    /* 残りのレジスタをすべて保存する（struct pt_regsの残りを構築する） */
    pushq   %rax        /* orig_ax = システムコール番号 */
    PUSH_AND_CLEAR_REGS rax=$-ENOSYS   /* rdi..r15をプッシュし、機微なレジスタをクリアする */

    /* これでカーネルスタック上に完全なstruct pt_regsができた */
    movq    %rsp, %rdi  /* 第1引数としてpt_regs *を渡す */

    /* 割り込みを有効にする — カーネルスタック上にいるので今は安全 */
    TRACE_IRQS_OFF
    call    do_syscall_64

    /* ... 復帰パス: レジスタを復元、SWAPGS、SYSRET/IRET ... */
SYM_CODE_END(entry_SYSCALL_64)
```

`SWITCH_TO_KERNEL_CR3`マクロは、KPTIが組み込まれていて、かつ動作している場合にのみ有効になる。これはカーネルスタックにアクセスする前にカーネルのCR3をロードする。

---

## struct pt_regs {#struct-pt_regs}

アセンブリコードはカーネルスタック上に`struct pt_regs`を構築する。この構造体はユーザープロセスのレジスタ状態を完全に記録する。

```c
/* arch/x86/include/asm/ptrace.h */
struct pt_regs {
    /*
     * C ABIでは、これらのレジスタは呼び出し先保存である。システムコールが
     * レジスタの完全なダンプを必要としない限り（例: ptrace用）、カーネルエントリ時には保存されない。
     */
    unsigned long r15;
    unsigned long r14;
    unsigned long r13;
    unsigned long r12;
    unsigned long rbp;
    unsigned long rbx;

    /* これらのレジスタは呼び出し先で破壊される。カーネルエントリ時に常に保存される。 */
    unsigned long r11;   /* 保存されたRFLAGS（SYSCALLパスの場合） */
    unsigned long r10;
    unsigned long r9;
    unsigned long r8;
    unsigned long rax;   /* システムコールの戻り値 */
    unsigned long rcx;   /* 保存されたRIP（SYSCALLパスの場合） */
    unsigned long rdx;
    unsigned long rsi;
    unsigned long rdi;

    /* SYSCALLエントリ時は、orig_rax = システムコール番号。
     * 例外エントリ時は、orig_rax = -1 またはエラーコード。 */
    unsigned long orig_ax;

    /* これらは例外時にハードウェアがプッシュするか、SYSCALL用に合成される */
    unsigned long ip;    /* ユーザーのRIP */
    unsigned long cs;    /* ユーザーのCS */
    unsigned long flags; /* ユーザーのRFLAGS */
    unsigned long sp;    /* ユーザーのRSP */
    unsigned long ss;    /* ユーザーのSS */
};
```

このレイアウトは割り込み/例外時にハードウェアがプッシュするものと一致しているため、同じ`pt_regs`構造体が両方のパスで使われる。

---

## do_syscall_64()とシステムコールテーブル {#dosyscall64-and-the-syscall-table}

`do_syscall_64()`は実際のシステムコールハンドラへディスパッチする。

```c
/* arch/x86/entry/common.c */
__visible noinstr void do_syscall_64(struct pt_regs *regs, int nr)
{
    add_random_kstack_offset();
    nr = syscall_enter_from_user_mode(regs, nr);

    instrumentation_begin();

    if (!do_syscall_x64(regs, nr) && !do_syscall_x32(regs, nr) && nr != -1) {
        /* システムコールが見つからない */
        regs->ax = __x64_sys_ni_syscall(regs);
    }

    instrumentation_end();
    syscall_exit_to_user_mode(regs);
}

static __always_inline bool do_syscall_x64(struct pt_regs *regs, int nr)
{
    unsigned int unr = nr;

    if (likely(unr < NR_syscalls)) {
        unr = array_index_nospec(unr, NR_syscalls);
        regs->ax = sys_call_table[unr](regs);
        return true;
    }
    return false;
}
```

`array_index_nospec()`はSpectre v1の緩和策である: インデックスをマスクして、`sys_call_table`への投機的な範囲外アクセスを防ぐ。

### システムコールテーブル {#the-syscall-table}

`sys_call_table[]`は関数ポインタの配列で、システムコール番号でインデックスされる。

```c
/* arch/x86/entry/syscall_64.c */
asmlinkage const sys_call_ptr_t sys_call_table[] = {
    [0 ... __NR_syscall_max] = &__x64_sys_ni_syscall, /* デフォルト: ENOSYS */
#include <asm/syscalls_64.h>   /* syscall_64.tblから生成される */
};
```

このテーブルは`arch/x86/entry/syscalls/syscall_64.tbl`から生成される。

```
# arch/x86/entry/syscalls/syscall_64.tbl (excerpt)
0    common  read            sys_read
1    common  write           sys_write
2    common  open            sys_open
...
60   common  exit            sys_exit
...
```

各システムコールハンドラは`SYSCALL_DEFINE`マクロで定義される。

```c
/* fs/read_write.c */
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
{
    return ksys_read(fd, buf, count);
}
```

このマクロは`__x64_sys_read(struct pt_regs *regs)`に展開され、`regs->di`、`regs->si`、`regs->dx`から引数を取り出す。

---

## 復帰パス: SYSRETとIRET {#the-return-path-sysret-and-iret}

`do_syscall_64()`が戻った後、エントリコードはレジスタを復元してユーザー空間に戻る。

**SYSRETによる通常の復帰:**

```asm
/* SYSRETが復元するもの: */
RIP    ← RCX   (saved user RIP)
RFLAGS ← R11   (saved user RFLAGS)
CS     ← from STAR MSR (user CS selector)
SS     ← from STAR MSR (user SS selector)
/* RSPはSYSRETの前にpt_regs->spから明示的に復元される */
```

**IRETへのフォールバック:** 戻りアドレスが非正規（non-canonical）である場合、タスクがシステムコールの途中でシグナルによって中断された場合、その他さまざまなエッジケースでは、SYSRETは使えない。そうした場合、カーネルは`IRET`（割り込みからの復帰）を使う。これはスタックから完全な例外フレームをポップする。`IRET`は低速だが、すべてのケースを正しく処理する。

カーネルは、どちらのパスが安全かを確認してから決定する。

```c
/* arch/x86/entry/entry_64.S */
/* pt_regs->csが__USER_CSでないか、RIPが非正規なら: IRETを使う */
/* それ以外: SYSRETを使う */
```

---

## vDSO: カーネルに入らない高速なシステムコール {#vdso-fast-syscalls-without-entering-the-kernel}

**vDSO**（virtual Dynamic Shared Object）は、カーネルがすべてのプロセスのアドレス空間にマップする小さな共有ライブラリである。実際にはカーネルのring 0に入る必要のない特定のシステムコールについて、高速な実装を提供する。

### vDSOの仕組み {#how-vdso-works}

カーネルは、すべてのプロセスの仮想アドレス空間の上端に2つの特別なページをマップする。

1. **vDSOページ**: 高速なシステムコールラッパー（`clock_gettime`、`gettimeofday`、`getcpu`、`clock_getres`）を実装した実行可能コード
2. **vvarページ**: vDSOのコードが読み取るカーネルデータ（現在時刻、クロック分解能など）を含む、読み取り専用の共有メモリページ

```
Process virtual address space (top):
  vvar page (r--): kernel writes time data here
  vDSO page (r-x): user code calls into this; reads from vvar
```

vvarページはカーネルの時刻管理サブシステムによって更新される。vDSOのコードはそこを直接読み取る — システムコールも特権遷移もない。`clock_gettime(CLOCK_REALTIME)`の場合、これによりSYSCALL/SYSRETのオーバーヘッドが完全になくなる。

```bash
# プロセスにマップされたvDSOを確認する
cat /proc/$$/maps | grep vdso
# 7ffd2a3d1000-7ffd2a3d3000 r--p 00000000 00:00 0      [vvar]
# 7ffd2a3d3000-7ffd2a3d5000 r-xp 00000000 00:00 0      [vdso]

# 解析用にvDSOを取り出す
cat /proc/self/maps | grep vdso
# カーネルツリーのtools/testing/selftests/vDSO/にあるvdso_testを使う
```

### vDSOのソース {#vdso-source}

vDSOは特別な共有ライブラリとしてコンパイルされる。

- `arch/x86/vdso/` — x86-64のvDSOソース
- `arch/x86/vdso/vdso64.lds.S` — リンカスクリプト
- `arch/x86/vdso/vclock_gettime.c` — `clock_gettime`の実装

vvarページは`struct vdso_data`（汎用のアーキテクチャ非依存ヘッダである`include/vdso/datapage.h`で定義）によって記述される。この構造体は時刻フィールドを含み、カーネルはvDSOの読み手が一貫したスナップショットを見られるよう、seqlockのセマンティクスを用いてそれらを更新する。`arch/x86/include/asm/vvar.h`は、vvarページを参照するために使われる`VVAR()`アクセスマクロを定義しており、構造体そのものを定義しているわけではない。

---

## int 0x80: レガシーな32ビットインターフェース {#int-0x80-the-legacy-32-bit-interface}

`SYSCALL`以前（さらには`SYSENTER`以前）は、x86のシステムコールはソフトウェア割り込みである`INT 0x80`を介して行われていた。この機構は、32ビットのアプリケーションやコードとの互換性のために、64ビットカーネルでも今なお動作する。

- `INT 0x80`はIDTベクタ0x80を通じてソフトウェア割り込みを配送する
- ベクタ0x80のIDTエントリは`entry_INT80_compat`（`arch/x86/entry/entry_64_compat.S`内）を指す
- このハンドラは**32ビットのシステムコールテーブル**（`ia32_sys_call_table[]`）を使う
- 64ビットプロセス内の`INT 0x80`は32ビットのレジスタ（EAX、EBXなど）と32ビットのABIのみを使う

`INT 0x80`は、完全なIDT例外配送機構を経由する（より多くの状態を保存する、DPLをチェックするなど）ため、`SYSCALL`よりかなり遅い。

---

## ユーザー空間に戻る前のシグナル配送 {#signal-delivery-before-returning-to-userspace}

`entry_SYSCALL_64`がユーザー空間に戻る前に、復帰パスは`syscall_exit_to_user_mode()`を呼び、保留中のシグナルを確認する。

```c
/* kernel/entry/common.c */
static void exit_to_user_mode_prepare(struct pt_regs *regs)
{
    u32 cached_flags = READ_ONCE(current_thread_info()->flags);

    if (unlikely(cached_flags & EXIT_TO_USER_MODE_WORK))
        exit_to_user_mode_loop(regs, cached_flags);
}

static void exit_to_user_mode_loop(struct pt_regs *regs, u32 cached_flags)
{
    while (true) {
        /* ... */
        if (cached_flags & _TIF_SIGPENDING)
            handle_signal_work(regs, cached_flags);
        /* ... */
    }
}
```

### シグナルフレームの構築 {#signal-frame-setup}

`do_signal()`がシグナルを配送すべきだと判断すると、復帰する前にユーザースタック上に**シグナルフレーム**を構築する。

```
User stack after signal frame setup:
  ┌─────────────────────────────────┐  ← rsp (new, after frame)
  │  struct rt_sigframe             │
  │    • uc (ucontext): saved regs  │
  │    • info (siginfo_t)           │
  │    • pretcode: → sa_restorer or │
  │      vDSO signal trampoline     │
  └─────────────────────────────────┘  ← old rsp (before signal)
```

`struct rt_sigframe`（`arch/x86/include/asm/sigframe.h`で定義）は`ucontext_t`を含み、その中に`pt_regs`のコピー — シグナルが配送された時点で保存されたユーザーのレジスタ状態 — が入っている。

次にカーネルは`regs->ip`をシグナルハンドラのアドレスに、`regs->sp`を新しいスタックトップに設定する。`entry_SYSCALL_64`が復帰処理を終えると、CPUは元のユーザーコードではなくシグナルハンドラへジャンプする。

シグナルハンドラが戻ると、`sigreturn()`を実行する（またはvDSOトランポリンを経由して`rt_sigreturn`を呼ぶ）。これは`rt_sigreturn`システムコールを呼び出す。カーネルのハンドラ（`sys_rt_sigreturn`）はシグナルフレームから保存された`ucontext_t`を読み取り、`pt_regs`を復元して、元のユーザー命令に戻る。

---

## トレーシングと可観測性 {#tracing-and-observability}

### strace: すべてのシステムコールを横取りする {#strace-intercepting-every-syscall}

`strace`は`ptrace(PTRACE_SYSCALL)`を使い、トレース対象をすべてのシステムコールの入口と出口で一時停止させる。

```bash
# コマンドのすべてのシステムコールをトレースする
strace ls

# 特定のシステムコールをトレースする
strace -e trace=read,write cat /etc/hostname

# 所要時間を表示する
strace -T ls

# 実行中のプロセスにアタッチする
strace -p <pid>
```

### perf trace: 低オーバーヘッドなシステムコールトレーシング {#perf-trace-low-overhead-syscall-tracing}

```bash
# システム全体のシステムコールの要約
perf trace --summary sleep 5

# プロセス単位のトレース
perf trace -p <pid>

# システムコールを名前ごとに数える
perf stat -e 'syscalls:sys_enter_*' -p <pid> sleep 5
```

### tracepoint {#tracepoints}

カーネルはすべてのシステムコールについて、入口と出口のtracepointを提供している。

```bash
# システムコールのtracepointを一覧表示する
ls /sys/kernel/debug/tracing/events/syscalls/ | head -20

# 特定のシステムコールのトレーシングを有効にする
echo 1 > /sys/kernel/debug/tracing/events/syscalls/sys_enter_read/enable
cat /sys/kernel/debug/tracing/trace
```

### Seccomp BPF {#seccomp-bpf}

Seccomp BPFフィルタは、システムコールハンドラが実行される前の`syscall_enter_from_user_mode()`の中で適用される。各フィルタルールはBPFプログラムで、システムコール番号と引数を受け取り、アクション（許可、拒否、kill、trapなど）を返す。`kernel/seccomp.c`を参照。

```bash
# プロセスでseccompが有効か確認する
cat /proc/<pid>/status | grep Seccomp
# Seccomp: 2  (2 = フィルタモード)
```

### 実行中のカーネルでentry_SYSCALL_64を見つける {#finding-entrysyscall64-in-the-running-kernel}

```bash
# システムコールのエントリポイントのアドレス
grep entry_SYSCALL_64 /proc/kallsyms

# LSTAR MSRと一致するか確認する
modprobe msr
rdmsr 0xC0000082   # LSTAR — 上のアドレスと一致するはず
```

---

## 参考文献 {#further-reading}

### カーネルソースとドキュメント {#kernel-source-documentation}

- [arch/x86/entry/entry_64.S](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/entry/entry_64.S) — `entry_SYSCALL_64`そのもの。珍しいほど丁寧にコメントされている
- [arch/x86/entry/syscall_64.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/entry/syscall_64.c) — システムコールテーブルと`do_syscall_64()`

### manページ {#man-pages}

- [`syscall(2)`](https://man7.org/linux/man-pages/man2/syscall.2.html) — 全アーキテクチャの呼び出し規約を1つの表にまとめたもの
- [`vdso(7)`](https://man7.org/linux/man-pages/man7/vdso.7.html) — どのシステムコールがカーネルに一切入らないか

### 関連ページ {#related-pages}

- [システムコールのエントリパス](../../syscalls/syscall-entry.md) — アーキテクチャ中立の視点
- [ARM64のシステムコールエントリ](../arm64/syscall-entry.md) — SYSCALLの代わりにSVC
- [vDSO](../../mm/vdso.md) — 高速パスの詳細

### LWNの記事 {#lwn-articles}

- [Anatomy of a system call, part 1](https://lwn.net/Articles/604287/) — まさにこのパスを順にたどる定番の解説
