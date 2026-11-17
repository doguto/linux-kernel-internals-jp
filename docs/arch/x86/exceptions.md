# x86の例外と割り込みの処理 {#x86-exception-and-interrupt-handling}

> IDT、例外エントリ、ハードウェア例外、そしてページフォールトハンドラのパス

## x86の例外の基礎 {#x86-exception-fundamentals}

x86では、すべての例外と割り込みは**割り込みディスクリプタテーブル（IDT: Interrupt Descriptor Table）**を介して配送される。IDTには256個のエントリがある。

```
IDT entry layout (gate descriptor, 16 bytes on x86-64):
 ┌──────────────────────────────────────────────────┐
 │ Offset[63:32]                                    │
 ├──────────────────────────────────────────────────┤
 │ Reserved                  │ Offset[31:16]        │
 ├───────────┬───────────────┼──────────────────────┤
 │  P DPL S  │   Type        │ Segment Selector     │
 ├───────────┴───────────────┴──────────────────────┤
 │ Offset[15:0]                                     │
 └──────────────────────────────────────────────────┘

P   = present
DPL = descriptor privilege level (0=kernel, 3=user)
Type= 0xE = interrupt gate (clears IF), 0xF = trap gate (keeps IF)
```

エントリ0-31はCPU例外用に予約されている。エントリ32-255は外部割り込みとソフトウェアトラップ用である（システムコールは歴史的にはint 0x80を使い、x86-64ではSYSCALL命令を使う）。

## CPU例外ベクタ {#cpu-exception-vectors}

| ベクタ | ニーモニック | 名称 | エラーコード | フォルト/トラップ/アボート |
|--------|----------|------|------------|-----------------|
| 0 | #DE | Divide Error（除算エラー） | なし | フォルト |
| 1 | #DB | Debug（デバッグ） | なし | フォルト/トラップ |
| 2 | NMI | Non-Maskable Interrupt（マスク不可能割り込み） | なし | 割り込み |
| 3 | #BP | Breakpoint（ブレークポイント） | なし | トラップ |
| 4 | #OF | Overflow（オーバーフロー） | なし | トラップ |
| 5 | #BR | BOUND Range Exceeded（BOUND範囲超過） | なし | フォルト |
| 6 | #UD | Invalid Opcode（無効オペコード） | なし | フォルト |
| 7 | #NM | Device Not Available（デバイス使用不可） | なし | フォルト |
| 8 | #DF | Double Fault（ダブルフォルト） | ゼロ | アボート |
| 10 | #TS | Invalid TSS（無効TSS） | あり | フォルト |
| 11 | #NP | Segment Not Present（セグメント不在） | あり | フォルト |
| 12 | #SS | Stack-Segment Fault（スタックセグメントフォルト） | あり | フォルト |
| 13 | #GP | General Protection（一般保護） | あり | フォルト |
| 14 | #PF | Page Fault（ページフォールト） | あり | フォルト |
| 16 | #MF | x87 FPU Error（x87 FPUエラー） | なし | フォルト |
| 17 | #AC | Alignment Check（アライメントチェック） | ゼロ | フォルト |
| 18 | #MC | Machine Check（マシンチェック） | なし | アボート |
| 19 | #XM | SIMD FP Exception（SIMD浮動小数点例外） | なし | フォルト |
| 20 | #VE | Virtualization Exception（仮想化例外） | なし | フォルト |

**フォルト**: 保存されたCS:RIPはフォルトを起こした命令を指す（ハンドラの後に再実行される）。
**トラップ**: 保存されたCS:RIPはトラップの次の命令を指す。
**アボート**: 回復不能。保存された状態が壊れている可能性がある。

## IDTの初期化 {#idt-initialization}

```c
/* arch/x86/kernel/idt.c */

/* カーネルコード内のIDTゲートディスクリプタ */
struct idt_data {
    unsigned int    vector;
    unsigned int    segment;
    struct idt_bits bits;
    const void      *addr;
};

/* 初期のIDT設定（Cコードより前） */
static const __initconst struct idt_data def_idts[] = {
    INTG(X86_TRAP_DE,       asm_exc_divide_error),
    INTG(X86_TRAP_NMI,      asm_exc_nmi),
    SYSG(X86_TRAP_BP,       asm_exc_int3),     /* DPL=3: ユーザーがINT3を発生させられる */
    SYSG(X86_TRAP_OF,       asm_exc_overflow), /* DPL=3: ユーザーがINTOを発生させられる */
    INTG(X86_TRAP_UD,       asm_exc_invalid_op),
    INTG(X86_TRAP_NM,       asm_exc_device_not_available),
    INTG(X86_TRAP_GP,       asm_exc_general_protection),
    INTG(X86_TRAP_PF,       asm_exc_page_fault),
    /* ... */
};

void __init idt_setup_early_traps(void)
{
    idt_setup_from_table(idt_table, early_idts, ARRAY_SIZE(early_idts), true);
    load_idt(&idt_descr);  /* LIDT命令 */
}
```

`LIDT`命令は、IDTレジスタ（IDTR）にIDTテーブルのベースアドレスとリミットをロードする。

## 例外エントリ: ハードウェアの動作 {#exception-entry-hardware-behavior}

CPUが（ring 3のユーザー空間から）ring 0へ例外を配送するとき:

```
Hardware actions (automatic, before any software):
  1. Look up IDT[vector] → get handler address and DPL
  2. Check DPL ≥ CPL (prevents user invoking kernel-only gates)
  3. Switch to kernel stack (via TSS.RSP0 for ring 0 target)
  4. Push on new stack:
       [SS]        ← old user SS
       [RSP]       ← old user RSP
       [RFLAGS]    ← old RFLAGS
       [CS]        ← old user CS
       [RIP]       ← address of faulting instruction (fault) or next (trap)
       [Error Code] ← for exceptions with error codes (GP, PF, etc.)
  5. Clear IF (interrupt gate): disables maskable interrupts
  6. Jump to handler address from IDT entry
```

ring 0 → ring 0の例外（カーネルがフォルトを受ける場合）では:

- スタックの切り替えは行われない（現在のカーネルスタックに留まる）
- x86-64では、同じ特権レベルの例外であってもSS/RSPが**常に**プッシュされる（32ビットx86とは異なる）
- SS/RSP/RFLAGS/CS/RIPと（該当する場合は）エラーコードは同様にプッシュされる

## 例外エントリのアセンブリ（x86-64） {#exception-entry-assembly-x86-64}

Linuxは`DEFINE_IDTENTRY_*`マクロを使って例外エントリスタブを生成する。

```c
/* arch/x86/include/asm/idtentry.h */

/* asmのエントリスタブ + Cハンドラの宣言を生成する */
#define DEFINE_IDTENTRY_ERRORCODE(func)                         \
    __visible noinstr void func(struct pt_regs *regs, long error_code)

/* arch/x86/kernel/traps.c */
DEFINE_IDTENTRY_ERRORCODE(exc_general_protection)
{
    struct task_struct *tsk = current;
    unsigned long condition = error_code;
    char *desc;

    cond_local_irq_enable(regs);

    if (static_cpu_has(X86_FEATURE_UMIP)) {
        if (user_mode(regs) && fixup_umip_exception(regs))
            return;
    }

    if (v8086_mode(regs)) {
        handle_vm86_fault((struct kernel_vm86_regs *) regs, error_code);
        return;
    }

    tsk->thread.error_code = error_code;
    tsk->thread.trap_nr = X86_TRAP_GP;

    if (show_unhandled_signals && unhandled_signal(tsk, SIGSEGV) &&
        printk_ratelimit()) {
        pr_info("%s[%d] general protection ip:%lx sp:%lx error:%lx",
                tsk->comm, task_pid_nr(tsk),
                regs->ip, regs->sp, error_code);
    }

    force_sig_fault(SIGSEGV, SEGV_MAPERR, (void __user *)0);
}
```

## struct pt_regs {#struct-pt_regs}

すべての例外ハンドラは、レジスタ状態全体を記録した`struct pt_regs`を受け取る。

```c
/* arch/x86/include/asm/ptrace.h */
struct pt_regs {
    /* 呼び出し先保存レジスタ（エントリのasmがプッシュする） */
    unsigned long r15, r14, r13, r12;
    unsigned long bp;
    unsigned long bx;

    /* 引数レジスタ */
    unsigned long r11, r10, r9, r8;
    unsigned long ax, cx, dx, si, di;

    /* 例外エントリ時にCPUがプッシュする */
    unsigned long orig_ax;  /* システムコール番号またはエラーコード */
    unsigned long ip;
    unsigned long cs;
    unsigned long flags;
    unsigned long sp;
    unsigned long ss;
};
```

エントリのアセンブリ（`arch/x86/entry/entry_64.S`）は、Cハンドラを呼ぶ前にすべてのレジスタを保存してこの構造体を構築する。

## タスクステートセグメント（TSS） {#task-state-segment-tss}

TSSは、例外配送時のスタック切り替えにとって極めて重要である。

```c
/* arch/x86/include/asm/processor.h */
struct x86_hw_tss {
    u32         reserved1;
    u64         sp0;    /* RSP0: カーネルスタックポインタ（ring 0が遷移先の場合に使われる） */
    u64         sp1;
    u64         sp2;
    u64         reserved2;
    u64         ist[7]; /* IST1-IST7: Interrupt Stack Tableのエントリ */
    u32         reserved3;
    u32         reserved4;
    u16         reserved5;
    u16         io_bitmap_base;
} __attribute__((packed));
```

**IST（Interrupt Stack Table）**: 一部の重要な例外は、通常のカーネルスタックが壊れていてもフォルトを処理できるよう、IST由来の専用スタックを使う。

| 例外 | IST | 理由 |
|-----------|-----|--------|
| NMI | IST1 | 例外ハンドラを含むあらゆるコードに割り込みうる |
| #DF（ダブルフォルト） | IST2 | スタックポインタが壊れている可能性がある |
| #MC（マシンチェック） | IST3 | ハードウェアが完全に壊れている可能性がある |
| #DB（デバッグ） | IST4 | 奇妙なコンテキストで発生しうる |

## ページフォールトハンドラ（#PF） {#page-fault-handler-pf}

ページフォールトは、稼働中のシステムで最も重要な例外である。

### エラーコードのビット {#error-code-bits}

```
#PF error code:
  bit 0: P   — 0=non-present, 1=protection violation
  bit 1: W/R — 0=read, 1=write
  bit 2: U/S — 0=supervisor (kernel), 1=user
  bit 3: RSVD — reserved bit violation
  bit 4: I/D — 0=data, 1=instruction fetch
  bit 5: PK  — protection key violation
  bit 6: SGX — SGX-related fault
```

### ハンドラの入口 {#handler-entry}

```c
/* arch/x86/mm/fault.c */

/*
 * CR2にはフォルトを起こしたリニアアドレスが入っている。
 * pt_regsを保存した後、asm_exc_page_faultから呼ばれる。
 */
DEFINE_IDTENTRY_RAW_ERRORCODE(exc_page_fault)
{
    unsigned long address = read_cr2();  /* フォルトを起こした仮想アドレス */
    irqentry_state_t state;

    prefetchw(&current->mm->mmap_lock);

    /*
     * KVM: ページフォールトハンドラを実行する前に通知する。
     * ハイパーバイザがEPT関連のフォルトを先に処理できるようにする。
     */
    if (kvm_handle_async_pf(regs, (u32)error_code))
        return;

    state = irqentry_enter(regs);
    instrumentation_begin();
    handle_page_fault(regs, error_code, address);
    instrumentation_end();
    irqentry_exit(regs, state);
}
```

### フォルトの分類 {#fault-classification}

```c
static noinline void
handle_page_fault(struct pt_regs *regs, unsigned long error_code,
                               unsigned long address)
{
    /* カーネルモードからのフォルトか？ */
    if (unlikely(fault_in_kernel_space(address))) {
        do_kern_addr_fault(regs, error_code, address);
        return;
    }

    /* ユーザーモードからのフォルト、またはカーネルがユーザーメモリにアクセスした場合 */
    do_user_addr_fault(regs, error_code, address);
}
```

### ユーザーアドレスのフォルトパス {#user-address-fault-path}

```c
static inline void
do_user_addr_fault(struct pt_regs *regs, unsigned long error_code,
                   unsigned long address)
{
    struct vm_area_struct *vma;
    struct task_struct *tsk = current;
    struct mm_struct *mm = tsk->mm;
    vm_fault_t fault;
    unsigned int flags = FAULT_FLAG_DEFAULT;

    /* 偽のフォルトか？ アドレスが有効なVMA内にあるか確認する */
    if (unlikely(!mm)) {
        bad_area_nosemaphore(regs, error_code, address);
        return;
    }

    /* 書き込みまたは命令フェッチであることを示す */
    if (error_code & X86_PF_WRITE)
        flags |= FAULT_FLAG_WRITE;
    if (error_code & X86_PF_INSTR)
        flags |= FAULT_FLAG_INSTRUCTION;
    if (error_code & X86_PF_USER)
        flags |= FAULT_FLAG_USER;

    /* mmap_lockを取得する（読み取りは共有、必要なら排他） */
    if (unlikely(!mmap_read_trylock(mm))) {
        /* 割り込みコンテキスト内、またはkillされている場合は待てない */
        if (unlikely(faulthandler_disabled() || !user_mode(regs))) {
            bad_area_nosemaphore(regs, error_code, address);
            return;
        }
        mmap_read_lock(mm);
    }

    /* フォルトを起こしたアドレスを含むVMAを探す */
    vma = find_vma(mm, address);
    if (unlikely(!vma)) {
        bad_area(regs, error_code, address, NULL);
        goto done;
    }
    if (likely(vma->vm_start <= address))
        goto good_area;

    /* アドレスがVMA間の隙間にある — スタックの伸長か確認する */
    if (unlikely(!(vma->vm_flags & VM_GROWSDOWN))) {
        bad_area(regs, error_code, address, vma);
        goto done;
    }
    if (unlikely(expand_stack(vma, address))) {
        bad_area(regs, error_code, address, vma);
        goto done;
    }

good_area:
    fault = handle_mm_fault(vma, address, flags, regs);
    /* ... */

done:
    mmap_read_unlock(mm);
}
```

### カーネルアドレスのフォルトパス {#kernel-address-fault-path}

```c
static noinline void
do_kern_addr_fault(struct pt_regs *regs, unsigned long hw_error_code,
                   unsigned long address)
{
    /* fixupテーブルを確認する: カーネルコード内で想定されたフォルト */
    if (fixup_exception(regs, X86_TRAP_PF, hw_error_code, address))
        return;

    /* KASAN/KFENCEのシャドウメモリか？ */
    if (kfence_handle_page_fault(address, hw_error_code & X86_PF_WRITE, regs))
        return;

    /* vmallocのフォルトか？（遅延マッピング、VMALLOC_STARTからVMALLOC_END） */
    if (!(hw_error_code & (X86_PF_RSVD | X86_PF_USER | X86_PF_PK))) {
        if (vmalloc_fault(address) >= 0)
            return;
    }

    /* Oops: 想定外のカーネルフォルト */
    bad_area_nosemaphore(regs, hw_error_code, address);
}
```

## 例外fixupテーブル {#exception-fixup-table}

カーネルの例外fixupテーブルは、フォルトを起こした命令のアドレスを回復用コードに対応付ける。`copy_from_user`/`copy_to_user`で使われる。

```c
/* arch/x86/lib/usercopy_64.S */
/* このコピー命令がフォルトを起こした場合: */
1:  rep movsb
    /* ... */
2:  ret
    /* fixupエントリ: [1]がフォルトを起こしたら[3]へジャンプする */
    _ASM_EXTABLE_UA(1b, 3f)
3:  mov %ecx, %eax   /* コピーされなかったバイト数を返す */
    ret

/* Cでは: */
unsigned long copy_from_user(void *to, const void __user *from, unsigned long n)
{
    /*
     * コピーがフォルトを起こした場合（不正なユーザーポインタ）、CPUの例外
     * ハンドラはfixupテーブルを確認し、oopsする代わりに
     * エラーパスへジャンプする。
     */
    return raw_copy_from_user(to, from, n);
}
```

## NMIの処理 {#nmi-handling}

マスク不可能割り込み（NMI）は`cli`でマスクできない。割り込みハンドラを含む*あらゆる*コードに割り込みうるため、慎重な処理が必要になる。

```c
/* arch/x86/kernel/nmi.c */
DEFINE_IDTENTRY_NMI(exc_nmi)
{
    irqentry_state_t irq_state;

    /*
     * NMIは専用のISTスタックを使う。NMIはネストしうる（NMI処理中のNMI）
     * ため、カーネルは慎重な「NMIネスト」プロトコルを使う:
     */

    /* NMIハンドラ自体に割り込んだかを確認する */
    if (this_cpu_read(nmi_state) != NMI_NOT_RUNNING) {
        this_cpu_write(nmi_cr2, read_cr2());
        this_cpu_write(nmi_state, NMI_LATCHED);
        return;
    }

    this_cpu_write(nmi_state, NMI_EXECUTING);
    this_cpu_write(nmi_cr2, read_cr2());

    nmi_restart:
    irq_state = irqentry_nmi_enter(regs);
    do_nmi(regs, 0);
    irqentry_nmi_exit(regs, irq_state);

    /* 処理中にNMIがラッチされたかを確認する */
    if (this_cpu_cmpxchg(nmi_state, NMI_EXECUTING, NMI_NOT_RUNNING) != NMI_EXECUTING)
        goto nmi_restart;  /* ネストしたNMIによって状態がNMI_LATCHEDに変わった */
}
```

NMIは次の用途に使われる。

- ハードウェアエラー（MCE、ウォッチドッグ）
- perfのPMUオーバーフロー割り込み（PEBS/LBR）
- kdumpのNMIシュートダウン（あるCPUでクラッシュしたら、他の全CPUにNMIを送る）
- カーネルのウォッチドッグ（`CONFIG_HARDLOCKUP_DETECTOR`）

## SYSCALL/SYSRET（高速システムコール） {#syscallsysret-fast-system-call}

現代のx86-64は`int 0x80`の代わりに`SYSCALL`命令を使う。

```
SYSCALL (user → kernel):
  1. Load RIP from MSR_LSTAR (syscall entry point)
  2. Save user RIP in RCX, RFLAGS in R11
  3. Load CS/SS from MSR_STAR segments
  4. Clear RFLAGS bits in MSR_SFMASK
  * Does NOT switch stack — kernel must do this in entry asm

SYSRET (kernel → user):
  1. Restore RIP from RCX, RFLAGS from R11
  2. Restore CS/SS
  * Does NOT save/restore any other registers
```

```c
/* arch/x86/entry/entry_64.S */
SYM_CODE_START(entry_SYSCALL_64)
    /* ユーザースタックからカーネルスタックに切り替える: */
    swapgs                          /* GSとMSR_KERNEL_GS_BASEを交換する */
    movq    %rsp, PER_CPU_VAR(cpu_tss_rw + TSS_sp2)
    SWITCH_TO_KERNEL_CR3 scratch_reg=%rsp
    movq    PER_CPU_VAR(cpu_current_top_of_stack), %rsp

    /* ユーザーの状態を保存する: */
    pushq   $__USER_DS              /* ss */
    pushq   PER_CPU_VAR(cpu_tss_rw + TSS_sp2)  /* rsp */
    pushq   %r11                   /* rflags */
    pushq   $__USER_CS             /* cs */
    pushq   %rcx                   /* rip（SYSCALLが保存したもの） */

    /* ... 残りのレジスタを保存し、do_syscall_64()を呼ぶ ... */
```

## 可観測性 {#observability}

```bash
# 例外を種類ごとに数える（perf）
perf stat -e exceptions:page_fault_user,exceptions:page_fault_kernel \
          -p <pid> sleep 5

# ページフォールトの発生率を監視する（vmstat）
vmstat 1
# pgfault: マイナーフォールト（キャッシュまたは匿名メモリからのページイン）
# pgmajfault: メジャーフォールト（ディスク読み込み）

# すべてのページフォールトをトレースする（重い！）
bpftrace -e '
tracepoint:exceptions:page_fault_user {
    @[ustack] = count();
}'

# IDTをデコードする
crash> idt

# ベクタ14（#PF）の例外ハンドラのアドレスを表示する
python3 -c "
import struct, sys
# /sys/kernel/debug/x86/idt_table is a binary dump of the IDT
with open('/sys/kernel/debug/x86/idt_table','rb') as f:
    data = f.read()
v = 14  # page fault
entry = data[v*16:(v+1)*16]
lo16, seg, bits16, hi16, hi32, _ = struct.unpack('<HHHHII', entry)
addr = lo16 | (hi16 << 16) | (hi32 << 32)
print(f'#PF handler: {addr:#x}')
"

# /proc/interruptsはCPUごとの割り込み回数を表示する
cat /proc/interrupts | head -5
```

## 参考文献 {#further-reading}

- [ARM64の例外モデル](../arm64/exception-model.md) — AArch64における同等の機構
- [ページフォールトハンドラ](../../mm/page-fault.md) — handle_mm_faultとVMAの解決
- [IRQ処理](../../interrupts/interrupts.md) — 外部割り込みの配送
- [システムコールエントリ](../../syscalls/syscall-entry.md) — SYSCALL/SYSRETパスの詳細
- [カーネルのハードニング](../../security/kernel-hardening.md) — SMEP/SMAP、KPTI
- [arch/x86/kernel/traps.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/traps.c) — 例外ハンドラの実装
- [arch/x86/mm/fault.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/fault.c) — ページフォールトハンドラ
- [arch/x86/entry/entry_64.S](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/entry/entry_64.S) — 例外/システムコールエントリのアセンブリ
- [Intel SDM](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) Vol 3A、第6章 — Interrupt and Exception Handling
