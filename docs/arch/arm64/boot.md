# ARM64のブートシーケンス {#arm64-boot-sequence}

> リセットベクタからstart_kernelまで: head.S、ページテーブルの構築、MMUの有効化

## 概要 {#overview}

ARM64システムに電源が投入されると、CPUはMMUが無効の状態で、ハードウェアがサポートする最上位の例外レベルにおいてリセットベクタから実行を開始する。

```
Power on / Reset
     │
     ▼
EL3 (Secure Monitor, if present) → firmware (ATF / TF-A)
     │  Sets up security configuration, drops to EL2
     ▼
EL2 (Hypervisor, if present) → firmware (QEMU, KVM host)
     │  Sets up virtualization, drops to EL1
     ▼
EL1 (Kernel) → arch/arm64/kernel/head.S
     │
     └─► primary_entry
          │
          ├─ el2_setup: configure EL2 system registers
          ├─ set_cpu_boot_mode_flag
          ├─ __create_page_tables: build early page tables
          ├─ __cpu_setup: enable caches, initialize SCTLR_EL1
          ├─ __primary_switch:
          │    ├─ adrp x0, init_pg_dir  (set TTBR1_EL1)
          │    ├─ enable_mmu (set SCTLR_EL1.M=1)
          │    └─ b __primary_switched
          └─ __primary_switched:
               ├─ set up kernel stack
               ├─ zero BSS
               └─ b start_kernel  (C code)
```

## head.S: エントリポイント {#heads-the-entry-point}

```asm
/* arch/arm64/kernel/head.S */

    /*
     * Linux/arm64 Imageヘッダ: 先頭64バイトはブートローダが使用する。
     * ブートローダは次の状態でここへジャンプする:
     *   x0 = デバイスツリーブロブ（DTB）の物理アドレス
     *   x1 = 0（予約）
     *   x2 = 0（予約）
     *   x3 = 0（予約）
     */
    .section ".head.text","ax"

SYM_CODE_START(primary_entry)
    /* すべての割り込みを無効化 */
    msr     daifset, #0xf

    /* x0（DTBアドレス）を後で使うために保存 */
    mov     x21, x0

    /* EL2で動作しているか? ハイパーバイザ設定を行う */
    bl      record_mmu_state
    bl      preserve_boot_args

    /*
     * EL1またはEL2のシステムレジスタを設定する。
     * CPUはMMUオフで起動する。エンディアンとキャッシュ状態は
     * MMU有効化の前に__cpu_setupで設定される。
     */
    bl      __create_page_tables
    /* x23 = TTBR1（カーネルVAページテーブルの物理アドレス） */
    /* x25 = VA_BITS */

    bl      __cpu_setup
    adr_l   x8, __primary_switch
    br      x8
SYM_CODE_END(primary_entry)
```

## 例外レベルの設定 {#exception-level-setup}

```c
/* arch/arm64/kernel/head.S */

/*
 * EL2でブートした場合（KVM/Xenでよくある）、EL2レジスタを設定する:
 * - HCR_EL2を設定: 仮想化の構成
 * - EL1のトラップ設定を行う
 * - eretでEL1へ降りる
 */
SYM_FUNC_START_LOCAL(el2_setup)
    msr     SPsel, #1           /* EL1ではSP_EL1を使う */

    /* 現在の例外レベルを確認: */
    mrs     x0, CurrentEL
    cmp     x0, #CurrentEL_EL2
    b.eq    1f

    /* EL1で動作中: 何もしない */
    mov     w0, #BOOT_CPU_MODE_EL1
    ret

1:  /* EL2で動作中: EL1で動くカーネル向けに設定 */
    mov_q   x0, HCR_HOST_NVHE_FLAGS  /* VHEなしのホスト構成 */
    msr     HCR_EL2, x0

    /* カーネルが物理タイマーとPMUにアクセスできるようにする: */
    mov     x0, #(CNTHCTL_EL2_EL1PCTEN | CNTHCTL_EL2_EL1PTEN)
    msr     CNTHCTL_EL2, x0

    /* ERETでEL1へ降りる: */
    mov     x0, #(PSR_F_BIT | PSR_I_BIT | PSR_A_BIT | PSR_D_BIT | \
                  PSR_MODE_EL1h)
    msr     SPSR_EL2, x0
    msr     ELR_EL2, lr
    mov     w0, #BOOT_CPU_MODE_EL2
    eret
SYM_FUNC_END(el2_setup)
```

## 初期ページテーブル {#early-page-tables}

MMUを有効化する前に、カーネルは次をマップする最小限のページテーブルを作成する。
1. カーネル自身（ブートパス用のアイデンティティマップ）
2. カーネル仮想アドレス空間（高位アドレス、TTBR1）

```c
/* arch/arm64/kernel/head.S */
SYM_FUNC_START_LOCAL(__create_page_tables)
    /*
     * ARM64は2組のページテーブルを使う:
     *   TTBR0_EL1: ユーザー空間（VA [0, VA_BITS-1]）
     *   TTBR1_EL1: カーネル空間（VA [-(1<<VA_BITS), -1]）
     *
     * ブート時: TTBR0に[_text, _end)のアイデンティティマップを、
     *          TTBR1にカーネルマップを作成する
     */

    /* init_pg_dir（カーネルページテーブル）をクリア */
    adrp    x0, init_pg_dir
    adrp    x1, init_pg_end
    sub     x1, x1, x0
    bl      __pi_memset             /* init_pg_dirをゼロ化（__memzeroはARM32専用） */

    /* 初期カーネルページテーブルを作成: */
    /* カーネルのtext/dataをKIMAGE_VADDRにマップ */
    adrp    x0, init_pg_dir
    adrp    x6, _text
    bl      map_kernel_segment      /* _text.._endをカーネルVAにマップ */
    ret
SYM_FUNC_END(__create_page_tables)
```

## CPUの設定: MMUの有効化 {#cpu-setup-enabling-the-mmu}

```asm
/* arch/arm64/kernel/head.S */
SYM_FUNC_START(__primary_switch)
    /*
     * TTBR0（アイデンティティマップ、MMU有効化中に使用）と
     * TTBR1（カーネルVAマップ）を設定
     */
    adrp    x1, idmap_pg_dir    /* TTBR0: ブートパス用アイデンティティマップ */
    adrp    x2, init_pg_dir     /* TTBR1: カーネルVAマップ */
    bl      __enable_mmu

    /* MMU有効化後: ここからはカーネルVAで実行している */
    ldr     x8, =__primary_switched
    adrp    x0, KERNEL_START        /* _textの物理アドレス */
    br      x8
SYM_FUNC_END(__primary_switch)

SYM_FUNC_START_LOCAL(__enable_mmu)
    /*
     * MAIR_EL1を設定: memory attribute indirection register
     * 8種類のメモリタイプ（Normal WB、Deviceなど）を定義する
     */
    mov_q   x5, MAIR_EL1_SET
    msr     mair_el1, x5

    /*
     * TCR_EL1を設定: Translation Control Register
     * - IPS: 物理アドレスサイズ
     * - T0SZ: TTBR0のVA範囲（ユーザー、48ビット）
     * - T1SZ: TTBR1のVA範囲（カーネル、48ビット）
     * - TG0/TG1: 4KBグラニュール
     * - SH0/SH1: inner shareable
     * - ORGN/IRGN: outer/innerライトバックキャッシュ可能
     */
    mrs     x10, ID_AA64MMFR0_EL1
    ...
    msr     tcr_el1, x10

    /* TTBR0とTTBR1を設定: */
    msr     ttbr0_el1, x1
    msr     ttbr1_el1, x2
    isb

    /* MMUを有効化: SCTLR_EL1.M = 1を設定 */
    mrs     x0, sctlr_el1
    orr     x0, x0, #SCTLR_EL1_M     /* MMU有効化 */
    orr     x0, x0, #SCTLR_EL1_C     /* キャッシュ有効化 */
    orr     x0, x0, #SCTLR_EL1_I     /* 命令キャッシュ有効化 */
    msr     sctlr_el1, x0
    isb     /* SCTLR_EL1への書き込み後はISBが必要 */
    ret
SYM_FUNC_END(__enable_mmu)
```

## MMU有効化の後: __primary_switched {#after-mmu-enable-_primaryswitched}

```asm
SYM_FUNC_START_LOCAL(__primary_switched)
    /* 例外ベクタテーブル（VBAR_EL1）を設定 */
    adr_l   x4, vectors             /* arch/arm64/kernel/entry.S */
    msr     vbar_el1, x4
    isb

    /* 初期カーネルスタックを設定 */
    adr_l   x5, init_task           /* init用のstruct task_struct */
    msr     sp_el0, x5              /* currentマクロで使用 */
    adr_l   x6, init_thread_union
    add     sp, x6, #THREAD_SIZE    /* カーネルスタックの先頭（init_thread_union + THREAD_SIZE） */

    /* どの例外レベルでブートしたかを記録 */
    str_l   x5, __boot_cpu_mode, x6

    /* BSSをゼロ化: */
    adrp    x0, __bss_start
    adrp    x1, __bss_stop
    sub     x1, x1, x0
    bl      __pi_memset

    /* setup_arch用にDTBアドレスを保存: */
    str_l   x21, __fdt_pointer, x5

    /* Cコードへジャンプ! */
    b       start_kernel
SYM_FUNC_END(__primary_switched)
```

## SMPセカンダリCPUのブート {#smp-secondary-cpu-boot}

セカンダリCPUは、プライマリによって解放された後に`secondary_entry`から開始する。

```c
/* arch/arm64/kernel/head.S */
SYM_CODE_START(secondary_entry)
    /* プライマリと同じ設定: EL2、CPU機能、ページテーブル */
    bl      el2_setup           /* EL2を設定（プライマリと共通） */
    bl      __cpu_setup

    adr_l   x0, KERNEL_START
    bl      __secondary_switch
    /* → secondary_start_kernel() */
SYM_CODE_END(secondary_entry)

/* arch/arm64/kernel/smp.c */
void secondary_start_kernel(void)
{
    struct mm_struct *mm = &init_mm;
    unsigned int cpu = smp_processor_id();

    mmgrab(mm);
    current->active_mm = mm;

    /* per-CPUデータ構造を初期化 */
    notify_cpu_starting(cpu);

    /* このCPUの割り込みとタイマーを設定 */
    local_irq_enable();
    cpu_startup_entry(CPUHP_AP_ONLINE_IDLE);
    /* → アイドルループ */
}
```

## ブートを観察する {#observing-the-boot}

```bash
# カーネルのブートメッセージを見る:
dmesg | head -50
# [    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd034]
# [    0.000000] Linux version 6.8.0 (gcc 13.2.0)
# [    0.000000] Machine model: Raspberry Pi 4 Model B
# [    0.000000] efi: UEFI not found.
# [    0.000000] OF: fdt: Ignoring memory range 0x0 - 0x80000
# [    0.000000] Zone ranges:
# [    0.000000]   DMA      [mem 0x0000000000000000-0x000000003fffffff]
# [    0.000000]   DMA32    empty
# [    0.000000]   Normal   [mem 0x0000000040000000-0x000000013fffffff]

# ブートした例外レベルを確認:
dmesg | grep "booted at EL"
# Kernel booted at EL2

# KASLRオフセット（有効な場合）:
dmesg | grep "kaslr"
# KASLR enabled: offset 0x5500000

# 検出されたarm64のCPU機能を確認:
dmesg | grep "CPU features"
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq
```

## 参考文献 {#further-reading}

- [ARM64の例外モデル](exception-model.md) — EL0-EL3、例外の入口と出口
- [ARM64のメモリモデル](memory-model.md) — TTBR0/TTBR1、ページテーブル形式
- [ページテーブル](../../mm/page-tables.md) — ホスト側のページテーブル内部構造
- [SMP](../../sched/context-switch.md) — セカンダリCPUの初期化
- [arch/arm64/kernel/head.S](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/kernel/head.S) — ブート用アセンブリの全体
- [arch/arm64/mm/mmu.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/mm/mmu.c) — ページテーブルの作成
- [Booting AArch64 Linux](https://docs.kernel.org/arch/arm64/booting.html) — ファームウェア/ブートローダとカーネルの間のブート規約
- ARM Architecture Reference Manual（ARM DDI 0487） — 権威ある参照資料
