# ARM64の例外モデル {#arm64-exception-model}

> 例外レベル、割り込み処理、そしてGIC

## 例外レベル {#exception-levels}

ARM64は4つの例外レベル（EL0-EL3）を定義している。これはx86の保護リングにおおよそ相当するが、より階層的である。

```
EL3: Secure Monitor (TrustZone firmware, e.g., ARM Trusted Firmware)
  │
EL2: Hypervisor (KVM runs here)
  │
EL1: OS Kernel (Linux kernel runs here)
  │
EL0: User space (applications)
```

主な特性:

- **EL0**: 非特権。ユーザーがアクセス可能なシステムレジスタにしかアクセスできない
- **EL1**: カーネルモード。カーネルアドレス空間とEL1システムレジスタに完全にアクセスできる
- **EL2**: ハイパーバイザモード。EL1の操作をトラップしてエミュレートできる
- **EL3**: 最も特権が高い。セキュアワールドと非セキュアワールドの間の遷移（TrustZone）を管理する

Linuxカーネルは通常EL1で動作するが、VHE（Virtualization Host Extension、ARMv8.1以降）があるとKVMはホストカーネルを直接EL2で動かす（VHEによってEL2レジスタがEL1レジスタのように見えるようになる）。VHEがない場合は、小さなEL2スタブがワールドの切り替えを処理し、ホストカーネルはEL1にとどまる。ゲストは常にEL1/EL0で動作する。

## 例外の種類 {#exception-types}

ARM64には4つのカテゴリの例外がある。

| カテゴリ | 説明 | 例 |
|----------|-------------|---------|
| **Synchronous** | 現在の命令によって引き起こされる | SVC、データアボート、命令アボート、未定義命令 |
| **IRQ** | 外部割り込み（通常優先度） | デバイス割り込み |
| **FIQ** | 高速割り込み（高優先度、通常はEL3へルーティングされる） | セキュアワールドの割り込み |
| **SError** | 非同期システムエラー | バスエラー、DRAM ECC |

## 例外ベクタ: VBAR_EL1 {#exception-vectors-vbar_el1}

例外が発生すると、CPUは`VBAR_EL1`（Vector Base Address Register for EL1）が指す**ベクタテーブル**内のエントリへジャンプする。

```
VBAR_EL1 base
  + 0x000: Synchronous, current EL with SP_EL0
  + 0x080: IRQ, current EL with SP_EL0
  + 0x100: FIQ, current EL with SP_EL0
  + 0x180: SError, current EL with SP_EL0
  + 0x200: Synchronous, current EL with SP_ELx
  + 0x280: IRQ, current EL with SP_ELx        ← kernel IRQ from EL1
  + 0x300: FIQ, current EL with SP_ELx
  + 0x380: SError, current EL with SP_ELx
  + 0x400: Synchronous, lower EL (AArch64)    ← syscall / fault from EL0
  + 0x480: IRQ, lower EL (AArch64)            ← IRQ while in userspace
  + 0x500: FIQ, lower EL (AArch64)
  + 0x580: SError, lower EL (AArch64)
  + 0x600: Synchronous, lower EL (AArch32)    ← 32-bit compat
  ...
```

### Linuxのベクタテーブル {#linux-vector-table}

```asm
/* arch/arm64/kernel/entry.S */
    .align  11
SYM_CODE_START(vectors)
    kernel_ventry   1, t, 64, sync    /* 同期 EL1t（SP_EL0） */
    kernel_ventry   1, t, 64, irq     /* IRQ EL1t */
    kernel_ventry   1, t, 64, fiq     /* FIQ EL1t */
    kernel_ventry   1, t, 64, error   /* エラー EL1t */

    kernel_ventry   1, h, 64, sync    /* 同期 EL1h（SP_ELx） */
    kernel_ventry   1, h, 64, irq     /* IRQ EL1h */
    kernel_ventry   1, h, 64, fiq     /* FIQ EL1h */
    kernel_ventry   1, h, 64, error   /* エラー EL1h */

    kernel_ventry   0, t, 64, sync    /* 同期 EL0 64ビット */
    kernel_ventry   0, t, 64, irq     /* IRQ EL0 64ビット */
    kernel_ventry   0, t, 64, fiq     /* FIQ EL0 64ビット */
    kernel_ventry   0, t, 64, error   /* エラー EL0 64ビット */

    kernel_ventry   0, t, 32, sync    /* 同期 EL0 32ビット */
    kernel_ventry   0, t, 32, irq     /* IRQ EL0 32ビット */
    kernel_ventry   0, t, 32, fiq     /* FIQ EL0 32ビット */
    kernel_ventry   0, t, 32, error   /* エラー EL0 32ビット */
SYM_CODE_END(vectors)
```

## シンドロームレジスタ: ESR_EL1 {#syndrome-registers-esr_el1}

同期例外が発生すると、`ESR_EL1`（Exception Syndrome Register）がその原因を示す。

```c
/* arch/arm64/include/asm/esr.h */

/* ESR_EL1のビット: */
/* [31:26] EC: Exception Class（例外クラス） */
#define ESR_ELx_EC_SHIFT    26
#define ESR_ELx_EC_MASK     (0x3FUL << ESR_ELx_EC_SHIFT)

/* 例外クラス: */
#define ESR_ELx_EC_UNKNOWN  0x00  /* 原因不明 */
#define ESR_ELx_EC_SVC64   0x15  /* SVC命令（EL0 → EL1、64ビット） */
#define ESR_ELx_EC_IABT_LOW 0x20  /* 下位ELからの命令アボート */
#define ESR_ELx_EC_IABT_CUR 0x21  /* 現在のELからの命令アボート */
#define ESR_ELx_EC_DABT_LOW 0x24  /* 下位ELからのデータアボート */
#define ESR_ELx_EC_DABT_CUR 0x25  /* 現在のELからのデータアボート */
#define ESR_ELx_EC_SOFTSTP_LOW 0x32 /* 下位ELからのソフトウェアステップ（デバッグ） */

/* [24] IL: 命令長（0=16ビット、1=32ビット） */
/* [23:0] ISS: Instruction-Specific Syndrome（命令固有シンドローム） */
/* データアボートの場合、ISSには次が含まれる: DFSC（フォールトステータスコード）、WnR（書き込み）、SRT（レジスタ） */
```

### フォールトステータスコード（DFSC） {#fault-status-codes-dfsc}

```c
/* ESR_EL1.ISS.DFSC内のデータフォールトステータスコード */
#define ESR_ELx_FSC_TYPE    0x3C
#define ESR_ELx_FSC_EXTABT  0x10  /* 外部アボート */
#define ESR_ELx_FSC_SERROR  0x11  /* 非同期SErrorアボート */
#define ESR_ELx_FSC_ACCESS  0x08  /* アクセスフラグフォールト */
#define ESR_ELx_FSC_FAULT   0x04  /* 変換フォールト（ページが存在しない） */
#define ESR_ELx_FSC_PERM    0x0C  /* パーミッションフォールト */

/* レベルビット[1:0]: */
/* 0b00 = レベル0（PGD）、0b01 = レベル1、0b10 = レベル2、0b11 = レベル3（PTE） */
```

### Linuxのフォールト処理 {#linux-fault-handling}

```c
/* arch/arm64/mm/fault.c */
static int __kprobes do_mem_abort(unsigned long far, unsigned long esr,
                                   struct pt_regs *regs)
{
    const struct fault_info *inf = esr_to_fault_info(esr);

    if (!inf->fn(far, esr, regs))
        return 0;

    /* 処理されなかったフォールト */
    arm64_notify_die(inf->name, regs, esr, far);
    return 0;
}

/* ESR EC → ハンドラへのディスパッチ */
static const struct fault_info fault_info[] = {
    { do_bad,           SIGKILL, 0,                   "ttbr address size fault" },
    /* ... */
    { do_translation_fault, SIGSEGV, SEGV_MAPERR,     "level 3 translation fault" },
    { do_page_fault,    SIGSEGV, SEGV_MAPERR,          "level 3 translation fault" },
    { do_page_fault,    SIGSEGV, SEGV_ACCERR,          "level 3 permission fault"  },
};
```

## システムコールのエントリ（SVC） {#syscall-entry-svc}

ARM64では、システムコールは`SVC #0`命令（Supervisor Call）を使う。

```asm
/* ユーザー空間のシステムコール: */
mov x8, #__NR_write    /* x8にシステムコール番号 */
mov x0, #1             /* fd = 1（stdout） */
mov x1, x_buf          /* バッファアドレス */
mov x2, x_len          /* 長さ */
svc #0                 /* EL1へトラップ */
/* x0に戻り値が返る */
```

カーネルのSVCハンドラ:

```asm
/* arch/arm64/kernel/entry.S */
SYM_CODE_START(el0t_64_sync_handler)
    /* すべてのレジスタを保存 */
    kernel_entry 0, 64

    mrs     x22, esr_el1        /* 例外シンドロームを読む */
    lsr     x24, x22, #ESR_ELx_EC_SHIFT  /* ECを取り出す */

    /* EC = 0x15は64ビットEL0からのSVCを意味する */
    cmp     x24, #ESR_ELx_EC_SVC64
    b.eq    el0_svc

    /* その他の例外の種類 */
    b       el0_sync_handler
SYM_CODE_END(el0t_64_sync_handler)
```

```c
/* arch/arm64/kernel/syscall.c */
static void el0_svc_common(struct pt_regs *regs, int scno,
                             int sc_nr, const syscall_fn_t syscall_table[])
{
    /* システムコールのトレーシング（seccomp、ptrace）を適用 */
    invoke_syscall(regs, scno, sc_nr, syscall_table);
}
```

システムコール番号は`x8`に、引数は`x0-x5`に入り、戻り値は`x0`に入る。

## GIC: Generic Interrupt Controller {#gic-generic-interrupt-controller}

ARMシステムは割り込み管理にx86のAPICではなくGIC（Generic Interrupt Controller）を使う。

```
Peripheral ──► GIC Distributor (shared)
                    │
                    ├──► CPU Interface 0 (CPU 0)
                    ├──► CPU Interface 1 (CPU 1)
                    └──► CPU Interface N (CPU N)
                              │
                          IRQ line to CPU core
```

### GICの割り込みの種類 {#gic-interrupt-types}

```c
/* GICの割り込みID */
/* 0-15:   SGI（Software Generated Interrupts） — IPI */
/* 16-31:  PPI（Private Peripheral Interrupts） — CPUごと（タイマー、PMU） */
/* 32+:    SPI（Shared Peripheral Interrupts） — CPU間で共有 */
/* 8192+:  LPI（Locality-specific Peripheral Interrupts） — PCIe向けMSI */
```

### LinuxのGICドライバ {#linux-gic-driver}

```c
/* drivers/irqchip/irq-gic-v3.c */
static struct irq_chip gic_chip = {
    .name                   = "GICv3",
    .irq_mask               = gic_mask_irq,
    .irq_unmask             = gic_unmask_irq,
    .irq_eoi                = gic_eoi_irq,
    .irq_set_type           = gic_set_type,
    .irq_set_affinity       = gic_set_affinity,   /* CPUアフィニティ */
    .irq_retrigger          = gic_retrigger,
    .irq_get_irqchip_state  = gic_irq_get_irqchip_state,
    .irq_set_irqchip_state  = gic_irq_set_irqchip_state,
};

/* EOI（End of Interrupt）: 完了時にGICへ通知する */
static void gic_eoi_irq(struct irq_data *d)
{
    gic_write_eoir(gic_irq(d));  /* EOIR1_EL1: IRQ番号をEOIに書き込む */
}
```

### ARM64のIPI {#ipis-on-arm64}

ARM64のIPI（TLBシュートダウン、関数呼び出し、再スケジューリング用）はGICのSGIを使う。

```c
/* arch/arm64/kernel/smp.c */
static void smp_send_reschedule(int cpu)
{
    smp_cross_call(cpumask_of(cpu), IPI_RESCHEDULE);
}

void arch_send_call_function_ipi_mask(const struct cpumask *mask)
{
    smp_cross_call(mask, IPI_CALL_FUNC);
}

/* smp_cross_call → GIC SGI → IPI_RESCHEDULE/IPI_CALL_FUNCハンドラ */
```

## システムレジスタ {#system-registers}

ARM64のシステムレジスタには`MRS`/`MSR`命令でアクセスする。

```asm
/* 現在のELを読む */
mrs x0, CurrentEL          /* ビット[3:2] = EL */
lsr x0, x0, #2             /* シフトして0/1/2/3を得る */

/* 仮想カウンタを読む（vDSOはシステムコールの代わりにこれを使う） */
mrs x0, CNTVCT_EL0

/* Thread IDレジスタ（TLSベース）を読む */
mrs x0, TPIDR_EL0

/* 割り込みを有効化/無効化 */
msr daifclr, #2            /* IRQマスクビットをクリア = IRQを有効化 */
msr daifset, #2            /* IRQマスクビットをセット = IRQを無効化 */
```

カーネル内（Cを使う場合）:

```c
/* arch/arm64/include/asm/sysreg.h */
u64 cntvct = read_sysreg(cntvct_el0);   /* 仮想カウンタ */
write_sysreg(ttbr0_val, ttbr0_el1);     /* ページテーブルのベース */
```

## 参考文献 {#further-reading}

- [メモリモデル](memory-model.md) — ARM64の弱い順序付けとバリア
- [メモリ管理: ページテーブル](../../mm/page-tables.md) — ARM64のページテーブル形式
- [割り込み: 割り込み処理の概要](../../interrupts/interrupts.md) — 汎用割り込みフレームワーク
- [仮想化: KVMアーキテクチャ](../../virtualization/kvm-arch.md) — ARM64上のKVM（EL2）
- カーネルツリーの`arch/arm64/kernel/` — ARM64の例外ハンドラ
- ARM Architecture Reference Manual（ARM DDI 0487）
