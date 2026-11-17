# KVMのexit処理 {#kvm-exit-handling}

> ゲストがフォールトしたときに何が起こるか: exit理由、I/Oエミュレーション、MMIO

## vcpu_runループ {#the-vcpu_run-loop}

KVMのすべてのvCPUは、`arch/x86/kvm/x86.c`の`kvm_arch_vcpu_ioctl_run()`内のタイトなループで動作する。ループはゲストに入り、ハイパーバイザの介入を必要とする何か（VM exit）が起こるまでハードウェアがゲストを実行し、その後KVMがそのexitを適切なハンドラにディスパッチする。

```
kvm_arch_vcpu_ioctl_run()          arch/x86/kvm/x86.c
  └── vcpu_run()
        └── vcpu_enter_guest()
              ├── kvm_x86_call(prepare_switch_to_guest)(vcpu)
              ├── VMLAUNCH / VMRESUME  ← hardware runs guest here
              │       (guest executes at near-native speed)
              │
              │   [VM exit fires — hardware saves guest state to VMCS]
              │
              ├── kvm_x86_call(handle_exit_irqoff)(vcpu)   (interrupts still off)
              │
              └── kvm_x86_call(handle_exit)(vcpu, exit_fastpath)
                    └── kvm_vmx_exit_handlers[exit_reason](vcpu)
                          returns 1 → resume guest
                          returns 0 → exit to userspace VMM
```

重要な約束事: ハンドラが`1`を返すと、KVMはすぐにゲストに再び入れることを意味する。`0`または負のエラーを返すと、`KVM_RUN` ioctlがユーザー空間に戻ることを意味し、ユーザー空間は`struct kvm_run`を調べて何が起きたかを知る。

### vcpu_enter_guestの内部 {#vcpu_enter_guest-internals}

各反復でゲストに入る前に、`vcpu_enter_guest()`は保留中の作業を処理する。

```c
/* arch/x86/kvm/x86.c（簡略化したもの） */
static int vcpu_enter_guest(struct kvm_vcpu *vcpu)
{
    fastpath_t exit_fastpath;
    u64 run_flags = 0;   /* KVM_RUN_FORCE_IMMEDIATE_EXIT、KVM_RUN_LOAD_GUEST_DR6、
                           * KVM_RUN_LOAD_DEBUGCTL — 保留中のexit、デバッグレジスタ、
                           * debugctlの状態から組み立てられる。ここでは省略 */

    /* 保留中のリクエストを処理する: TLBフラッシュ、MMUのリロードなど */
    if (kvm_check_request(KVM_REQ_MMU_SYNC, vcpu))
        kvm_mmu_sync_roots(vcpu);
    if (kvm_check_request(KVM_REQ_TLB_FLUSH, vcpu))
        kvm_vcpu_flush_tlb_all(vcpu);
    /* ... 他のKVM_REQ_*フラグ ... */

    /* 入る前に保留中の割り込みがあれば注入する */
    if (kvm_cpu_has_injectable_intr(vcpu))
        kvm_x86_call(inject_irq)(vcpu, false /* reinjected */);

    /* プリエンプションを無効にし、ゲストのCR3に切り替え、VMX non-rootに入る */
    exit_fastpath = kvm_x86_call(vcpu_run)(vcpu, run_flags);  /* VMLAUNCHまたはVMRESUME */

    /*
     * 戻ってきた — VM exitが発生した。handle_exit_irqoff()は割り込みが
     * 無効なまま最小限の作業（例: #NMの処理）を行う。何も返さない —
     * exit_fastpathは上のvcpu_run()呼び出しから得たもの。
     */
    kvm_x86_call(handle_exit_irqoff)(vcpu);

    /* ... 割り込み/プリエンプションを再び有効にし、ゲスト時間を計上するなど ... */

    return kvm_x86_call(handle_exit)(vcpu, exit_fastpath);
}
```

## exit理由のディスパッチ {#exit-reason-dispatch}

Intelでは、exit理由はVMCSの`VM_EXIT_REASON`フィールドに格納される。`vmx_get_exit_reason(vcpu)`は（`vcpu->arch`ではなくvCPUごとのVMXのフィールドから）それを`union vmx_exit_reason`に読み込み、下位ビット（`.basic`）が`kvm_vmx_exit_handlers[]`のインデックスとなる。

```c
/* arch/x86/kvm/vmx/vmx.c */
static int (*kvm_vmx_exit_handlers[])(struct kvm_vcpu *vcpu) = {
    [EXIT_REASON_EXCEPTION_NMI]     = handle_exception_nmi,
    [EXIT_REASON_EXTERNAL_INTERRUPT]= handle_external_interrupt,
    [EXIT_REASON_IO_INSTRUCTION]    = handle_io,
    [EXIT_REASON_CR_ACCESS]         = handle_cr,
    [EXIT_REASON_DR_ACCESS]         = handle_dr,
    [EXIT_REASON_CPUID]             = kvm_emulate_cpuid,
    [EXIT_REASON_MSR_READ]          = kvm_emulate_rdmsr,
    [EXIT_REASON_MSR_WRITE]         = kvm_emulate_wrmsr,
    [EXIT_REASON_HLT]               = kvm_emulate_halt,
    [EXIT_REASON_EPT_VIOLATION]     = handle_ept_violation,
    [EXIT_REASON_EPT_MISCONFIG]     = handle_ept_misconfig,
    [EXIT_REASON_VMCALL]            = kvm_emulate_hypercall,
    [EXIT_REASON_APIC_ACCESS]       = handle_apic_access,
    /* ... 合計で約50個のハンドラ ... */
};

static int __vmx_handle_exit(struct kvm_vcpu *vcpu,
                              fastpath_t exit_fastpath)
{
    struct vcpu_vmx *vmx = to_vmx(vcpu);
    union vmx_exit_reason exit_reason = vmx_get_exit_reason(vcpu);

    /* ゲストの状態がVMX non-root modeでネイティブに実行できない場合
     * （よくあるケース: "unrestricted guest"をサポートしないハードウェア上の
     * リアルモード/16ビットのコード、例えばBIOS）、下のテーブルによる
     * ディスパッチではなく完全にエミュレートする: */
    if (vmx->vt.emulation_required)
        return handle_invalid_guest_state(vcpu);

    /* ... entry失敗とイベントのベクタリングのチェックは省略 ... */

    if (exit_reason.basic >= kvm_vmx_max_exit_handlers ||
        !kvm_vmx_exit_handlers[exit_reason.basic]) {
        /* 認識できないexit理由: 状態をダンプし、ユーザー空間にexitする */
        dump_vmcs(vcpu);
        kvm_prepare_unexpected_reason_exit(vcpu, exit_reason.full);
        return 0;
    }

    return kvm_vmx_exit_handlers[exit_reason.basic](vcpu);
}
```

実際のディスパッチは`__vmx_handle_exit()`が行う。`kvm_x86_call(handle_exit)`で到達するエントリポイントは、それを包む薄い`vmx_handle_exit()`ラッパーであり、検出されたバスロック（`KVM_EXIT_X86_BUS_LOCK`）のチェックも行う。

AMD SVMも同じパターンに従い、`arch/x86/kvm/svm/svm.c`の`svm_exit_handlers[]`を、VMCBの`EXITCODE`フィールドでインデックスする。

## 代表的なexit理由 {#common-exit-reasons}

### EXIT_REASON_IO_INSTRUCTION — ポートI/O {#exit_reason_io_instruction-port-io}

ポートI/O命令（`in`/`out`）はデフォルトでexitする。VMCSごとのI/Oビットマップを設定すれば、特定のポートについてはまったくexitせずに実行させることができる。exitするポートのうち、単純なケースは高速パスがカーネル内で処理し、低速パス（ストリングI/O）は完全な命令エミュレーションにフォールバックする。

```c
/* arch/x86/kvm/vmx/vmx.c */
static int handle_io(struct kvm_vcpu *vcpu)
{
    unsigned long exit_qualification = vmx_get_exit_qual(vcpu);
    int size   = (exit_qualification & 7) + 1; /* 1、2、または4バイト */
    int in     = (exit_qualification >> 3) & 1;
    int string = (exit_qualification >> 4) & 1;
    int port   = exit_qualification >> 16;

    if (string)
        return kvm_emulate_instruction(vcpu, 0);

    return kvm_fast_pio(vcpu, size, port, in);
}
```

`kvm_fast_pio()`（`arch/x86/kvm/x86.c`内）は`kvm_fast_pio_in()`/`kvm_fast_pio_out()`に処理を渡し、これらはエミュレータの`emulator_pio_in()`/`emulator_pio_out()`の経路を通る。この経路はまず`kvm_io_bus`のディスパッチテーブルを介してカーネル内のデバイスエミュレーションを試み、そのポートを所有するカーネル内デバイスがなければ`kvm_run`を埋めて`0`を返す。

```c
/* struct kvm_run — KVM_EXIT_IOのときにユーザー空間が見るもの */
struct kvm_run {
    __u32 exit_reason;    /* KVM_EXIT_IO */
    /* ... */
    struct {
        __u8  direction;  /* KVM_EXIT_IO_INまたはKVM_EXIT_IO_OUT */
        __u8  size;       /* 1、2、または4 */
        __u16 port;       /* I/Oポート番号 */
        __u32 count;      /* ストリングI/Oの場合: 繰り返し回数 */
        __u64 data_offset;/* データが置かれているkvm_run内のオフセット */
    } io;
};
```

### EXIT_REASON_EPT_VIOLATION — EPT/NPTのページフォールト {#exit_reason_ept_violation-eptnpt-page-fault}

ゲストが、有効なExtended Page Table（EPT）エントリを持たないゲスト物理アドレス（GPA）にアクセスするか、アクセス権に違反すると、ハードウェアはEPT violationのexitを発生させる。

```c
/* arch/x86/kvm/vmx/vmx.c — handle_ept_violation()に、その
 * __vmx_handle_ept_violation()ヘルパーを畳み込んで簡略化したもの */
static int handle_ept_violation(struct kvm_vcpu *vcpu)
{
    unsigned long exit_qualification = vmx_get_exit_qual(vcpu);
    gpa_t gpa = vmcs_read64(GUEST_PHYSICAL_ADDRESS);
    u64 error_code = 0;

    /* 読み込み、書き込み、命令フェッチのどれだったか */
    /* 読み込み用のPFERRビットはない（WRITEとFETCHがなければ読み込みを意味する） */
    if (exit_qualification & EPT_VIOLATION_ACC_WRITE)
        error_code |= PFERR_WRITE_MASK;
    if (exit_qualification & EPT_VIOLATION_ACC_INSTR)
        error_code |= PFERR_FETCH_MASK;

    return kvm_mmu_page_fault(vcpu, gpa, error_code, NULL, 0);
}
```

`kvm_mmu_page_fault()`（`arch/x86/kvm/mmu/mmu.c`内）は原因を判定する。

- **マッピングの欠如**: ホストの物理ページを指す新しいEPTリーフエントリをインストールする。
- **MMIO範囲**: GPAがRAMではなくデバイスにマップされている — MMIOエミュレーションを発生させる（下記参照）。
- **権限フォールト**: ゲストが許可されていないアクセスを試みた（例: CoWのための読み取り専用ページへの書き込み）。

### EXIT_REASON_CPUID {#exit_reason_cpuid}

`CPUID`は常にインターセプトされ、KVMが機能ビットをフィルタリングまたは合成できるようになっている。ハンドラは`arch/x86/kvm/cpuid.c`の`kvm_emulate_cpuid()`を呼び、これは`vcpu->arch.cpuid_entries[]`（`KVM_SET_CPUID2` ioctlから設定される）でリーフを検索する。機微なリーフはマスクされる。たとえば、hypervisorビットがセットされ、特定のVMX機能ビットはゲストから隠される。

### EXIT_REASON_HLT {#exit_reason_hlt}

ゲストが`HLT`を実行し、割り込みが無効になっている場合、vCPUがすることは何もない。KVMは`kvm_vcpu_halt()`を呼び、これが`kvm_vcpu_block()`を呼ぶ。

```c
/* virt/kvm/kvm_main.c、簡略化したもの */
bool kvm_vcpu_block(struct kvm_vcpu *vcpu)
{
    struct rcuwait *wait = kvm_arch_vcpu_get_wait(vcpu);  /* 汎用的には&vcpu->wait */
    bool waited = false;

    kvm_arch_vcpu_blocking(vcpu);
    prepare_to_rcuwait(wait);

    for (;;) {
        set_current_state(TASK_INTERRUPTIBLE);
        if (kvm_vcpu_check_block(vcpu) < 0)
            break;          /* 再び実行可能になった、または他の保留中の作業がある */
        waited = true;
        schedule();
    }

    finish_rcuwait(wait);
    kvm_arch_vcpu_unblocking(vcpu);
    return waited;          /* 実際にスリープしたか、それとも既に作業があったか */
}
```

vCPUスレッドは、`kvm_vcpu_kick()`が起こすまでスリープする。これは典型的には仮想割り込みが到着したためである。

### EXIT_REASON_MSR_READ / MSR_WRITE {#exit_reason_msr_read-msr_write}

MSRへのアクセスは、**MSRビットマップ**によって安価にできる。これは4KBのページ（VMCS自体に埋め込まれているのではなく、VMCSの`MSR_BITMAP`フィールドが指す）で、各ビットが特定のMSRがVM exitを引き起こすかどうかを制御する。`IA32_TSC`や`IA32_SYSENTER_EIP`のような頻繁に読まれるMSRはパススルー（exitなし）にできる。インターセプトされるMSRについては、KVMは`kvm_emulate_rdmsr()`または`kvm_emulate_wrmsr()`（`arch/x86/kvm/x86.c`内）にディスパッチし、これらは`__kvm_get_msr()` / `__kvm_set_msr()`の中でswitchによるMSRごとのディスパッチを行う。

### EXIT_REASON_EXCEPTION_NMI {#exit_reason_exception_nmi}

ゲストの例外（ページフォールト、一般保護例外、ブレークポイント）とNMIはここでexitする。`arch/x86/kvm/vmx/vmx.c`の`handle_exception_nmi()`は、VMCSの`VM_EXIT_INTR_INFO`フィールドを調べてベクタを判定する。状況に応じて次のようになる。

- **ゲストのページフォールト**（`#PF`）: `kvm_handle_page_fault()`が処理する — ホストのページをマップするか、フォールトをゲストに再注入する。
- **ブレークポイント**（`#BP`）: ゲストに配送されるか、GDBスタブのためにインターセプトされる。
- **NMI**: VMCSのイベント注入の仕組みによってゲストに注入される。

## MMIOエミュレーション {#mmio-emulation}

`handle_ept_violation()`が、フォールトしたGPAがMMIO領域にある（ホストの物理マッピングが存在しない）と判定すると、KVMはページをマップする代わりに、フォールトを引き起こした命令をエミュレートする。

```
Guest executes:   mov [0xfee00000], eax   (write to local APIC MMIO)

  1. EPT violation exits — GPA 0xfee00000 has no EPT mapping
  2. handle_ept_violation() → kvm_mmu_page_fault()
  3. kvm_mmu_page_fault() detects MMIO: no struct page for this GPA
  4. kvm_mmu_page_fault() calls x86_emulate_instruction() to decode and
     execute the faulting instruction (uses struct x86_emulate_ops to
     fetch instruction bytes from guest)
  5. The emulator's write-emulated callback reaches vcpu_mmio_write(),
     which dispatches the actual write:
     ├── in-kernel local APIC — checked directly (vcpu->arch.apic->dev),
     │   not via the generic kvm_io_bus
     │     hit: handled in-kernel, resume guest (return 1)
     └── miss → kvm_io_bus_write(KVM_MMIO_BUS, ...) — other in-kernel
           devices (e.g. IOAPIC), matched by registered GPA range
           ├── hit: handled in-kernel, resume guest (return 1)
           └── miss: kvm_run.mmio filled; ioctl returns to userspace
```

`struct kvm_run`のMMIOフィールド（`include/uapi/linux/kvm.h`より）:

```c
/* exit_reason == KVM_EXIT_MMIOのとき */
struct kvm_run {
    __u32 exit_reason;   /* KVM_EXIT_MMIO */
    /* ... */
    struct {
        __u64 phys_addr; /* MMIOアクセスのゲスト物理アドレス */
        __u8  data[8];   /* 書き込むデータ（書き込みの場合）。読み込みの場合はユーザー空間が埋める */
        __u32 len;       /* アクセスサイズ: 1、2、4、または8バイト */
        __u8  is_write;  /* 1 = ゲストの書き込み、0 = ゲストの読み込み */
    } mmio;
};
```

ゲストの**読み込み**の場合: ユーザー空間（例: QEMU）は`kvm_run.mmio.data`にデバイスレジスタの値を埋め、`KVM_RUN`でKVMに再び入る。KVMはそのデータをゲストのレジスタに書き込んで、エミュレートした命令を完了させる。

ゲストの**書き込み**の場合: ユーザー空間は`kvm_run.mmio.data`を読み、それを自身のデバイスモデルに転送する。

### KVMのカーネル内デバイス {#kvm-in-kernel-devices}

いくつかのデバイスは、ユーザー空間への往復を避けるために完全にカーネル内でエミュレートされる。

| デバイス | カーネル内エミュレーション | 登録 |
|--------|---------------------|---------------|
| ローカルAPIC | `arch/x86/kvm/lapic.c` | 汎用バスより先に直接チェックされる（`vcpu->arch.apic->dev`） — `kvm_io_bus_register_dev()`では登録されない |
| IOAPIC | `arch/x86/kvm/ioapic.c` | `kvm_io_bus_register_dev(kvm, KVM_MMIO_BUS, ...)`、GPA範囲で照合される |
| PIT（8254タイマー） | `arch/x86/kvm/i8254.c` | `kvm_io_bus_register_dev(kvm, KVM_PIO_BUS, ...)` — MMIOではなく、レガシーなポートI/O（ポート0x40-0x43、0x61） |

汎用の`kvm_io_bus`のGPA範囲による検索を通るのはIOAPIC（およびその他の本物のMMIOデバイス）だけである。ローカルAPICはその前に特別扱いされ、PITはMMIOではまったくなく、別のポートI/Oバスを介して到達する。

## 割り込みの注入 {#interrupt-injection}

ゲストのvCPUへの割り込みは、通常のハードウェア割り込み線を通じて届くわけではない。KVMは仮想割り込みの配送を管理する。

```c
/* virt/kvm/kvm_main.c — __kvm_vcpu_kick()、簡略化したもの。
 * kvm_vcpu_kick(vcpu)は薄いインラインラッパー: __kvm_vcpu_kick(vcpu, false)。 */
void __kvm_vcpu_kick(struct kvm_vcpu *vcpu, bool wait)
{
    int me, cpu;

    /* まず起こす: ブロックしていた（例: HLT中の）場合は、起こすだけで十分 —
     * pCPU上で実行中ではなかったのでIPIは不要。 */
    if (kvm_vcpu_wake_up(vcpu))
        return;

    me = get_cpu();

    /* 自分自身のスレッドから自分のvCPUをキックする場合（例: 自己IPIのような
     * イベント）: ポーリングしているモードフラグを切り替えるだけで、実際のIPIは不要。 */
    if (vcpu == __this_cpu_read(kvm_running_vcpu)) {
        if (vcpu->mode == IN_GUEST_MODE)
            WRITE_ONCE(vcpu->mode, EXITING_GUEST_MODE);
        goto out;
    }

    /* そうでなければ他のpCPU上で実行中: IPIを送ってVM exitを強制し、
     * 保留中の作業に気付かせる。 */
    if (kvm_arch_vcpu_should_kick(vcpu)) {
        cpu = READ_ONCE(vcpu->cpu);   /* 現在ロードされていなければ-1 */
        if (cpu != me && (unsigned int)cpu < nr_cpu_ids && cpu_online(cpu))
            smp_send_reschedule(cpu);
    }
out:
    put_cpu();
}
```

デバイス割り込みのルーティング経路:

```
Device (e.g., virtio-net) signals IRQ
  → kvm_set_irq(kvm, irq_source_id, irq, level, ...)   virt/kvm/irqchip.c
  → kvm_irq_routing_table lookup: GSI → IOAPIC pin or MSI
  → kvm_ioapic_set_irq() / kvm_apic_set_irq()
  → kvm_vcpu_kick(target_vcpu)
  → on next vcpu_enter_guest(): inject via VMCS VM-entry interrupt info field
```

ルーティングテーブル（`struct kvm_irq_routing_table`、`virt/kvm/irqchip.c`で割り当てられる）はGSI番号をIOAPIC/MSIのエントリにマップし、`KVM_SET_GSI_ROUTING` ioctlで更新される。

exitを減らす高度な割り込み機能:

- **APICv / AVIC**: ローカルAPICをハードウェアで仮想化し（Intel APICv / AMD AVIC）、APICのMMIOによるexitのほとんどをなくし、VM exitなしでのposted interruptの配送を可能にする。
- **Posted interrupt**: CPUはVM exitなしで仮想割り込みをゲストに直接配送する。これはvCPUごとのposted interruptディスクリプタ（`struct pi_desc`）で追跡される。

## 性能: exitの測定 {#performance-measuring-exits}

VM exitは、KVMのオーバーヘッドの主な発生源である。exitの頻度とexit処理のレイテンシを最小化することが、性能上の主な手段となる。

```bash
# debugfs内のVMごとのディレクトリ。VMごとに1つ: <creator-pid>-<vm-fd-name>
ls /sys/kernel/debug/kvm/
# 1234-3/   ...   (VMレベルの統計ファイルはここに直接置かれる)

# vCPUごとにサブディレクトリがあり、統計ごとに生カウンタのファイルが1つある
# （arch/x86/kvm/x86.cのkvm_vcpu_stats_desc[]を参照）— 単一の集約ファイルはない:
ls /sys/kernel/debug/kvm/1234-3/vcpu0/
# exits  io_exits  mmio_exits  irq_window_exits  halt_exits  halt_wakeup  ...

cat /sys/kernel/debug/kvm/1234-3/vcpu0/exits
# 8473921
cat /sys/kernel/debug/kvm/1234-3/vcpu0/mmio_exits
# 340291

# perf KVM: すべてのVMにわたるexit統計を集計する
perf kvm stat record -a sleep 10
perf kvm stat report
# Analyze VM exits:
# VM-EXIT  Samples  Samples%  Time%    Min Time  Max Time  Avg time
# HLT          ...
# EPT_VIOLATION ...

# kvm_exit tracepoint: exit理由ごとのトレース
echo 1 > /sys/kernel/tracing/events/kvm/kvm_exit/enable
cat /sys/kernel/tracing/trace_pipe
# qemu-1234 [002] kvm_exit: reason EPT_VIOLATION rip 0xffffffff81234567 ...

# kvm_mmio tracepoint: MMIOエミュレーションのイベント
echo 1 > /sys/kernel/tracing/events/kvm/kvm_mmio/enable

# kvm_pio tracepoint: ポートI/Oのイベント
echo 1 > /sys/kernel/tracing/events/kvm/kvm_pio/enable
```

性能に関する主な指針:

- **MMIOのexit**: カーネル内エミュレーション（APIC、IOAPIC）やvirtio（キックをバッチ化する）を使って頻度を減らす。
- **MSRのexit**: MSRビットマップを調整し、頻繁に読まれるMSRをパススルーにする。
- **PIOのexit**: ブート時のCOM1シリアルがよくある発生源である。本番のVMではシリアルコンソールを無効にする。
- **起動時のEPT violation**: 正常である — ワーキングセットのページが最初の数秒間で読み込まれる。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [arch/x86/kvm/x86.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kvm/x86.c) — `vcpu_enter_guest()`: exitごとのディスパッチループ。`kvm_fast_pio()`: カーネル内のポートI/Oの高速パス。`kvm_emulate_rdmsr()`/`kvm_emulate_wrmsr()`: MSRの読み書きによるVM exitのディスパッチ
- [arch/x86/kvm/vmx/vmx.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kvm/vmx/vmx.c) — `kvm_vmx_exit_handlers[]`: VMXのexit理由のディスパッチテーブル。`handle_io()`、`handle_ept_violation()`、`handle_exception_nmi()`
- [arch/x86/kvm/svm/svm.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kvm/svm/svm.c) — `svm_exit_handlers[]`: AMD SVMにおける同等のexit理由テーブル。VMCBの`EXITCODE`フィールドでインデックスされる
- [arch/x86/kvm/mmu/mmu.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kvm/mmu/mmu.c) — `kvm_mmu_page_fault()`: EPTマッピングの欠如、MMIO領域、権限フォールトを区別する
- [arch/x86/kvm/cpuid.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kvm/cpuid.c) — `kvm_emulate_cpuid()`: CPUIDリーフの検索と機能ビットのフィルタリング
- [arch/x86/kvm/emulate.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kvm/emulate.c) — `x86_emulate_insn()`: MMIOとストリングI/Oのエミュレーションの最上位のエントリポイントである`x86_emulate_instruction()`（`x86.c`内）が内部で呼び出す、低レベルのデコードと実行のステップ。`vcpu_mmio_write()`/`vcpu_mmio_read()`（`x86.c`内）は、ローカルAPICに直接ディスパッチするか`kvm_io_bus_write()`/`kvm_io_bus_read()`にフォールバックするエミュレータのコールバックである
- [virt/kvm/kvm_main.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/virt/kvm/kvm_main.c) — `kvm_vcpu_block()`/`kvm_vcpu_kick()`: HLTによるvCPUのスリープと起床
- [virt/kvm/irqchip.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/virt/kvm/irqchip.c) — `kvm_set_irq()`と、デバイス割り込みの配送に使われるGSIのルーティングテーブル
- [arch/x86/kvm/lapic.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kvm/lapic.c) — カーネル内のローカルAPICのエミュレーション
- [arch/x86/kvm/ioapic.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kvm/ioapic.c) — カーネル内のIOAPICのエミュレーション
- [arch/x86/kvm/i8254.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kvm/i8254.c) — カーネル内のPIT（8254タイマー）のエミュレーション
- [include/uapi/linux/kvm.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/kvm.h) — `struct kvm_run`、`KVM_EXIT_IO`、`KVM_EXIT_MMIO`: ユーザー空間から見えるexitのABI

### 関連ページ {#related-pages}

- [KVMアーキテクチャ](kvm-arch.md) — vCPUのライフサイクル、VMCS、KVMのioctl
- [メモリ仮想化](kvm-memory.md) — EPT、シャドウページング、MMU notifier
- [ネストされた仮想化](nested-virt.md) — L2ゲストのexit処理

### 外部資料 {#external}

- [The Definitive KVM API Documentation](https://docs.kernel.org/virt/kvm/api.html) — `KVM_RUN`、`struct kvm_run`、`KVM_EXIT_IO`/`KVM_EXIT_MMIO`のフィールドレイアウト
- [KVM VCPU Requests](https://docs.kernel.org/virt/kvm/vcpu-requests.html) — `vcpu_enter_guest()`の冒頭で処理される`KVM_REQ_*`フラグの仕組み
