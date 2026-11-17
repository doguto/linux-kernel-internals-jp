# ARM64のシステムコールのエントリ {#arm64-syscall-entry}

> SVC命令からカーネルのCハンドラへ、そして戻るまで

64ビットARMプロセスが行うすべてのシステムコール（`read()`、`write()`、`mmap()`、その他何であれ）は、ユーザー空間の`SVC #0`から、カーネルのアセンブリのエントリポイント、レジスタ保存の仕組み、Cのディスパッチ層を通り、`ERET`でユーザー空間へ戻るという、ハードウェアとソフトウェアの決まった経路をたどる。このページではその経路を追う。

---

## SVC命令 {#the-svc-instruction}

ARM64では、システムコールは`SVC #0`（Supervisor Call）で行う。呼び出し規約は次のとおりである。

| レジスタ | 役割 |
|----------|------|
| `x8` | システムコール番号 |
| `x0`–`x5` | 引数（最大6個） |
| `x0` | 戻り値（復帰後） |

```asm
/* ユーザー空間: ARM64でwrite(1, buf, len)を呼ぶ */
mov  x8, #64        /* __NR_write = 64 */
mov  x0, #1         /* fd = 1（stdout） */
ldr  x1, =buf       /* バッファアドレス */
mov  x2, #len       /* 長さ */
svc  #0             /* EL1へトラップ。戻り値はx0 */
```

`SVC #0`の即値は慣例的にゼロであり、Linuxはこれを無視する。システムコール番号は`x8`から取られる。

### x86-64との対比 {#contrast-with-x86-64}

| 観点 | ARM64（`SVC #0`） | x86-64（`SYSCALL`） |
|--------|------------------|-------------------|
| システムコール番号 | `x8` | `rax` |
| 引数 | `x0`–`x5` | `rdi`、`rsi`、`rdx`、`r10`、`r8`、`r9` |
| 戻り値 | `x0` | `rax` |
| 保存される戻りアドレス | CPUが`ELR_EL1`に書き込む | CPUが`rcx`に書き込む |
| 保存されるフラグ | CPUが`SPSR_EL1`に書き込む | CPUが`r11`に書き込む |
| エントリ機構 | 例外ベクタ（VBAR_EL1） | 固定のMSR（LSTAR） |

ARM64では、`SVC #0`はベクタテーブルを通じてディスパッチされる同期例外を引き起こす。x86-64では、`SYSCALL`はIDTを経由せずに`LSTAR` MSRのアドレスへ直接ジャンプする。

---

## 例外ベクタによるディスパッチ {#exception-vector-dispatch}

`SVC #0`が実行されると、CPUは**同期例外**を受け、`VBAR_EL1 + 0x400`、すなわち64ビットEL0からの同期例外用のスロットへジャンプする。

```
VBAR_EL1 + 0x200: Synchronous, EL1 with SP_ELx   (kernel exceptions)
VBAR_EL1 + 0x400: Synchronous, lower EL (AArch64) ← SVC from 64-bit EL0
VBAR_EL1 + 0x600: Synchronous, lower EL (AArch32) ← SVC from 32-bit EL0
```

`arch/arm64/kernel/entry.S`にあるLinuxのベクタテーブル:

```asm
/* arch/arm64/kernel/entry.S */
    .align  11
SYM_CODE_START(vectors)
    kernel_ventry   1, h, 64, sync    /* 同期 EL1h（カーネルの例外） */
    kernel_ventry   1, h, 64, irq     /* IRQ EL1h */
    /* ... */
    kernel_ventry   0, t, 64, sync    /* 同期 EL0、64ビット ← SVCはここ */
    kernel_ventry   0, t, 64, irq     /* IRQ EL0、64ビット */
    /* ... */
    kernel_ventry   0, t, 32, sync    /* 同期 EL0、32ビット（AArch32 compat） */
    /* ... */
SYM_CODE_END(vectors)
```

### ESR_EL1とEC == 0x15 {#esr_el1-and-ec-0x15}

CPUは`ESR_EL1`に例外の原因を格納する。ビット31:26が**Exception Class（EC）**であり、64ビットのSVCではECは`0x15`である。

```c
/* arch/arm64/include/asm/esr.h */
#define ESR_ELx_EC_SHIFT    26
#define ESR_ELx_EC_SVC64    0x15   /* 64ビットEL0からのSVC */
#define ESR_ELx_EC_SVC32   0x11   /* 32ビットEL0からのSVC */
```

ハンドラは`ESR_EL1`を読み、ECを取り出して`el0_svc`へ分岐する。

```c
/* arch/arm64/kernel/entry-common.c */
asmlinkage void noinstr el0t_64_sync_handler(struct pt_regs *regs)
{
    unsigned long esr = read_sysreg(esr_el1);

    switch (ESR_ELx_EC(esr)) {
    case ESR_ELx_EC_SVC64:
        el0_svc(regs);
        break;
    case ESR_ELx_EC_DABT_LOW:
        el0_da(regs, esr);
        break;
    /* ... その他の例外クラス ... */
    default:
        el0_inv(regs, esr);
    }
}
```

注: `el0t_64_sync_handler`は`arch/arm64/kernel/entry-common.c`にあるC関数である（アセンブリではない）。`entry.S`の低レベルなベクタスタブが`kernel_entry 0, 64`でレジスタを保存し、その後このCハンドラを呼ぶ。

---

## レジスタの保存と復元 {#register-save-and-restore}

`kernel_entry 0, 64`は、ユーザー空間のすべてのレジスタを`struct pt_regs`としてカーネルスタックに保存する。

```c
/* arch/arm64/include/asm/ptrace.h */
struct pt_regs {
    union {
        struct user_pt_regs user_regs;
        struct {
            u64 regs[31];   /* x0–x30 */
            u64 sp;         /* ユーザースタックポインタ（SP_EL0） */
            u64 pc;         /* ELR_EL1: ユーザーへの戻りアドレス */
            u64 pstate;     /* SPSR_EL1: 保存されたプロセッサ状態 */
        };
    };
    u64 orig_x0;    /* 元のx0（システムコール再開用） */
    s32 syscallno;  /* システムコール番号 */
    /* ... MTE、SVEの状態用の追加フィールド ... */
};
```

このマクロはタスクごとのカーネルスタック（`SP_EL1`から）に切り替え、`x0`–`x30`、`sp`、`ELR_EL1`（戻りPC）、`SPSR_EL1`（保存されたPSTATE）を保存する。得られた`pt_regs *`がすべてのCハンドラに渡される。

復帰パスでは、`kernel_exit 0`がすべてのフィールドを復元して`ERET`を実行する。`ERET`は`ELR_EL1`からプログラムカウンタを、`SPSR_EL1`からプロセッサ状態をアトミックに復元し、EL0へ戻る。

---

## Cのディスパッチパス {#the-c-dispatch-path}

レジスタ保存の後、制御はアセンブリからCへ移る。

```
el0_svc (entry-common.c)
  └─► do_el0_svc (syscall.c)
        └─► el0_svc_common (syscall.c)
              ├── syscall_enter_from_user_mode_work()  [tracing, seccomp]
              └── invoke_syscall()
                    └─► sys_call_table[scno](regs)
```

`arch/arm64/kernel/syscall.c`の`el0_svc_common`は、`orig_x0`と`syscallno`を`pt_regs`に保存し、システムコール前の処理を実行してから`invoke_syscall`を呼ぶ。

```c
/* arch/arm64/kernel/syscall.c */
static void invoke_syscall(struct pt_regs *regs, unsigned int scno,
                            unsigned int sc_nr,
                            const syscall_fn_t syscall_table[])
{
    long ret;

    if (scno < sc_nr) {
        syscall_fn_t syscall_fn;
        syscall_fn = syscall_table[array_index_nospec(scno, sc_nr)];  /* Spectre v1 */
        ret = __invoke_syscall(regs, syscall_fn);
    } else {
        ret = do_ni_syscall(regs, scno);   /* -ENOSYS */
    }

    syscall_set_return_value(current, regs, 0, ret);
}
```

### システムコールテーブル {#the-syscall-table}

ARM64ネイティブのシステムコールテーブルは、`arch/arm64/kernel/sys.c`で`__SYSCALL()`マクロによって構築される。

```c
/* arch/arm64/kernel/sys.c */
#define __SYSCALL(nr, sym)  [nr] = __arm64_##sym,

const syscall_fn_t sys_call_table[__NR_syscalls] = {
    [0 ... __NR_syscalls - 1] = __arm64_sys_ni_syscall,
#include <asm/unistd.h>
};
```

ARM64は`include/uapi/asm-generic/unistd.h`の汎用システムコール番号体系を使う。これはRISC-VやLoongArchと共有されている同じテーブルである。`__NR_write`は64、`__NR_read`は63、`__NR_openat`は56である。

---

## システムコールのトレーシングとseccomp {#syscall-tracing-and-seccomp}

`kernel/entry/common.c`の`syscall_enter_from_user_mode_work()`が、システムコール前のすべてのフックを処理する。これは`thread_info`の`syscall_work`フラグをチェックする。

```c
/* kernel/entry/common.c */
static long syscall_enter_from_user_mode_work(struct pt_regs *regs, long syscall)
{
    unsigned long work = READ_ONCE(current_thread_info()->syscall_work);

    if (work & SYSCALL_WORK_SECCOMP) {
        int ret = __secure_computing(NULL);   /* BPFフィルタを実行 */
        if (ret == -1)
            return ret;  /* seccompによりプロセスがkillされた */
    }

    if (work & SYSCALL_WORK_SYSCALL_TRACEPOINT)
        trace_sys_enter(regs, syscall);       /* sys_enter raw tracepoint */

    syscall = ptrace_report_syscall_entry(regs);  /* トレース中ならptraceで停止 */

    return syscall;
}
```

タスクが`PTRACE_SYSCALL`でトレースされている場合（`strace`が使う方法）、`SYSCALL_WORK_SYSCALL_TRACE`がセットされ（Linux 5.12以降。古い`TIF_SYSCALL_TRACE`を置き換えた）、`ptrace_report_syscall_entry()`がタスクを一時停止する。トレーサは`NT_PRSTATUS`を指定した`PTRACE_GETREGSET`でレジスタを読む。これは`struct user_pt_regs`を公開する。

復帰パスはエントリと対称である。`syscall_exit_to_user_mode()`が`sys_exit` tracepointを発火し、ptraceのexit-stopを処理し、`kernel_exit`がレジスタを復元する前に保留中のシグナルを配送する。

---

## AArch32 compat {#aarch32-compat}

64ビットARM64カーネルは、32ビットARM（AArch32）のELFバイナリを実行できる（`CONFIG_COMPAT`）。これらのプロセスも`SVC #0`を使うが、レジスタ規約が異なる。

| レジスタ | 役割 |
|----------|------|
| `r7` | システムコール番号（EABI） |
| `r0`–`r5` | 引数 |
| `r0` | 戻り値 |

ハードウェアはAArch32の同期例外を`+0x400`ではなく**VBAR_EL1 + 0x600**へルーティングする。エントリハンドラ`el0t_32_sync_handler`はEC `0x11`（`ESR_ELx_EC_SVC32`）を認識して`el0_svc_compat`へ分岐し、これが別のテーブルを介してディスパッチする。

```c
/* arch/arm64/kernel/sys_compat.c */
#define __SYSCALL(nr, sym)  [nr] = __arm64_compat_##sym,

const syscall_fn_t compat_sys_call_table[__NR_compat_syscalls] = {
    [0 ... __NR_compat_syscalls - 1] = __arm64_compat_sys_ni_syscall,
#include <asm/unistd32.h>
};
```

AArch32のシステムコール番号は歴史的な32ビットARM ABIに一致し、ネイティブarm64の番号とは異なる。compatハンドラは32ビットの構造体レイアウトを変換するために`compat_*`型（`compat_size_t`、`compat_timespec64`、`compat_uptr_t`）を使う。`TIF_32BIT`がcompatスレッドを示し、`is_compat_task()`がそれをチェックする。

---

## vDSO: ホットパスでSVCを避ける {#vdso-avoiding-svc-for-hot-paths}

`clock_gettime`、`gettimeofday`、`clock_getres`は、ワークロードによっては毎秒数百万回呼ばれる。**vDSO**（virtual Dynamic Shared Object）は、カーネルが管理する共有ページから時刻管理データを読むことで、これらを`SVC #0`なしで実装する。

```bash
cat /proc/self/maps | grep -E 'vdso|vvar'
# ffff800000000000-ffff800000001000 r--p 00000000 00:00 0  [vvar]
# ffff800000001000-ffff800000002000 r-xp 00000000 00:00 0  [vdso]
```

ARM64のvDSOは`arch/arm64/kernel/vdso/`にある。x86のTSCの代わりに、`CNTVCT_EL0`を読む。これはARMの汎用仮想タイマーのカウンタであり、カーネルがEL0からアクセス可能に設定している。

```asm
mrs x0, CNTVCT_EL0   /* 仮想カウンタを読む — 特権遷移なし */
```

vDSOは**vvarページ**内の`struct vdso_data`（`include/vdso/datapage.h`由来）を読み、seqlockを使ってカーネルによる並行更新を検出する。`lib/vdso/gettimeofday.c`にある共通実装はarm64とx86の両方向けにコンパイルされ、アーキテクチャ固有なのは`__arch_get_hw_counter()`だけである。クロックモードが`VDSO_CLOCKMODE_NONE`の場合（例: VMマイグレーション後）、vDSOは透過的に`svc #0`へフォールバックする。

ARM64にはvsyscallの固定アドレスページはない。それはx86-64だけのレガシーである。

---

## 可観測性 {#observability}

### strace {#strace}

```bash
strace ls                       # すべてのシステムコールをトレース
strace -e trace=read,write ls   # システムコール名でフィルタ
strace -T ls                    # システムコールごとの経過時間を表示
strace -p <pid>                 # 実行中のプロセスにアタッチ
```

`strace`は`ptrace(PTRACE_SYSCALL)`を使うので、vDSOで高速化された呼び出しは決して見えない。それらは`svc #0`を発行しないからである。

### perf traceとbpftrace {#perf-trace-and-bpftrace}

```bash
perf trace ls                   # tracepointによる低オーバーヘッドなシステムコールトレース
perf trace --summary sleep 5    # システムコールごとの回数とレイテンシの要約

# システムコールを名前ごとに数える
bpftrace -e 'tracepoint:syscalls:sys_enter_* { @[probe] = count(); }'

# 特定のプロセスをトレース
bpftrace -e 'tracepoint:syscalls:sys_enter_read /pid == 1234/ { printf("fd=%d\n", args->fd); }'
```

### /proc/PID/syscall {#procpidsyscall}

```bash
cat /proc/$(pidof sshd)/syscall
# 281 0x4 0x7ffd3a2b1000 0x400 0x0 0x0 0x0 0x7ffd3a2b1080 0xffffb3a82000
# ^nr  ^args...                                              ^sp    ^pc
```

システムコール内でブロックしているプロセスについて、システムコール番号、引数、ユーザーSP、ユーザーPCを表示する。

### raw tracepoint {#raw-tracepoints}

```bash
# ftraceでsys_enter_readのトレーシングを有効化
echo 1 > /sys/kernel/debug/tracing/events/syscalls/sys_enter_read/enable
cat /sys/kernel/debug/tracing/trace
```

---

## 参考文献 {#further-reading}

**このサイト内:**

- [ARM64の例外モデル](exception-model.md) — VBAR_EL1、ESR_EL1、例外レベル、GIC
- [ARM64のメモリモデル](memory-model.md) — 弱い順序付け、バリア、LDAR/STLR
- [x86-64のシステムコールのエントリ](../x86/syscall-entry.md) — SYSCALL/SYSRET、entry_SYSCALL_64、KPTI
- [システムコールのエントリパス](../../syscalls/syscall-entry.md) — アーキテクチャ中立な概要
- [vDSOと仮想システムコール](../../syscalls/vdso.md) — vvarページ、seqlock、クロックモード
- [32ビットcompatシステムコール](../../syscalls/compat.md) — compat型、compat_sys_call_table
- [ptraceとシステムコールの横取り](../../syscalls/ptrace-interception.md) — straceの仕組み

**カーネルのソースファイル:**

- `arch/arm64/kernel/entry.S` — ベクタテーブル、`kernel_entry`/`kernel_exit`マクロ、低レベルスタブ
- `arch/arm64/kernel/entry-common.c` — `el0t_64_sync_handler`を含むCの例外ハンドラ
- `arch/arm64/kernel/syscall.c` — `do_el0_svc`、`el0_svc_common`、`invoke_syscall`
- `arch/arm64/kernel/sys.c` — `sys_call_table`の定義
- `arch/arm64/kernel/sys_compat.c` — AArch32プロセス用の`compat_sys_call_table`
- `arch/arm64/kernel/vdso/` — ARM64のvDSOソース
- `arch/arm64/include/asm/ptrace.h` — `struct pt_regs`のレイアウト
- `arch/arm64/include/asm/esr.h` — ESR_EL1の例外クラス定数
- `include/vdso/datapage.h` — `struct vdso_data`（共有vvarのレイアウト）
- `lib/vdso/gettimeofday.c` — 共通のvDSOクロック実装（arm64とx86）
- `kernel/entry/common.c` — `syscall_enter_from_user_mode`、`syscall_exit_to_user_mode`

**外部:**

- ARM Architecture Reference Manual（ARM DDI 0487） — AArch64例外モデルの決定版リファレンス
- カーネルツリーの`Documentation/arm64/`
