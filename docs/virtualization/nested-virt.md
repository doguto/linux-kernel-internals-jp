# ネストされた仮想化 {#nested-virtualization}

> VMの中でハイパーバイザを動かす: L0、L1、L2、そしてvmcs12

## 用語 {#terminology}

ネストされた仮想化は、通常のホスト/ゲストの区分にもう1つの層を加える。

```
L0: The real hardware host, running Linux + KVM
    (the "true" hypervisor — sees real VMCS, real EPT)

L1: A virtual machine running its own hypervisor (e.g., another KVM, Hyper-V, VMware ESXi)
    (thinks it owns the hardware; executes VMLAUNCH/VMRESUME)

L2: A virtual machine created by L1
    (nested guest — two levels of virtualization deep)
```

ネストされた仮想化のサポートがなければ、L1が`VMLAUNCH`を実行しようとすると、ゲストはVMX root modeにいないため、`#UD`（未定義命令）または`#GP`（一般保護例外）が発生する。KVMのネストされた仮想化は、`VMLAUNCH`や関連する命令をL1の内部から動作するようにする。

ネストされた仮想化を有効にする:

```bash
# Intel
modprobe kvm-intel nested=1
# または恒久的に:
echo "options kvm-intel nested=1" > /etc/modprobe.d/kvm.conf

# AMD
modprobe kvm-amd nested=1

# 機能がL1に公開されていることを確認する
cat /sys/module/kvm_intel/parameters/nested   # Y
grep -m1 vmx /proc/cpuinfo                    # L1ゲスト内でvmxフラグが見える
```

## 問題: L1がVMLAUNCHを実行する {#the-problem-l1-executes-vmlaunch}

KVMはL1を作成するとき、`vmx` CPUIDフラグを公開するので、L1は自分がVT-xを持っていると信じる。L1は`VMPTRLD`、`VMWRITE`、`VMLAUNCH`、`VMRESUME`を実行できる。実際のハードウェアでは、これらはVMX root modeでのみ使える特権命令である。L1はVMX non-root modeで動作しているので、これらはすべてL0へのVM exitを引き起こす。

L0は、これらのexitのそれぞれに対して、L1が本物のハイパーバイザであった場合にハードウェアが行うことをエミュレートして応答しなければならない。

| L1の命令 | L0へのexit | L0の動作 |
|---------------|-----------|-----------|
| `VMXON` | `EXIT_REASON_VMXON` | `handle_vmxon()` — このvCPUのネストされたVMXの状態を有効にする |
| `VMPTRLD`（カレントVMCSの設定） | `EXIT_REASON_VMPTRLD` | `handle_vmptrld()` — L1がどのvmcs12を扱っているかを記録する |
| `VMWRITE` | `EXIT_REASON_VMWRITE` | `handle_vmwrite()` — メモリ上のvmcs12にフィールドを書き込む |
| `VMREAD` | `EXIT_REASON_VMREAD` | `handle_vmread()` — メモリ上のvmcs12からフィールドを読み出す |
| `VMLAUNCH` | `EXIT_REASON_VMLAUNCH` | `handle_vmlaunch()` → `nested_vmx_run()` — L2へのentryを合成する |
| `VMRESUME` | `EXIT_REASON_VMRESUME` | `handle_vmresume()` → `nested_vmx_run()` — L2に再び入る |
| `VMXOFF` | `EXIT_REASON_VMXOFF` | `handle_vmxoff()` — ネストされたVMXの状態を無効にする |

上記のハンドラはすべて`arch/x86/kvm/vmx/nested.c`にある。

## struct vmcs12: L1から見たVMCS {#struct-vmcs12-l1s-view-of-the-vmcs}

L0は、L1がVMCSに入っていると信じている内容のメモリ上のコピーを保持する。これが`arch/x86/kvm/vmx/vmcs12.h`で定義される`struct vmcs12`である。

```c
/* arch/x86/kvm/vmx/vmcs12.h（一部のフィールド） */
typedef u64 natural_width;  /* vmcs12が32/64ビットのL1間で固定レイアウトを持つように別名にしている */

struct __packed vmcs12 {
    /* ヘッダ: ハードウェアのVMCSリビジョン識別子と一致しなければならない */
    struct vmcs_hdr hdr;   /* revision_id:31, shadow_vmcs:1 */
    u32 abort;

    /* ゲスト状態領域: L1がここに置くものはL2の状態である */
    natural_width guest_cr0, guest_cr3, guest_cr4;
    natural_width guest_rsp, guest_rip, guest_rflags;
    u64 guest_ia32_efer;
    u16 guest_cs_selector, guest_ss_selector;
    /* ... すべてのセグメントレジスタ、ディスクリプタテーブルレジスタ ... */

    /* ホスト状態領域: L2がexitした後にL1が戻りたい場所 */
    natural_width host_cr0, host_cr3, host_cr4;
    natural_width host_rsp, host_rip;
    u64 host_ia32_efer;

    /* 制御フィールド: L1がL2に対してインターセプトしたいもの */
    u32 pin_based_vm_exec_control;
    u32 cpu_based_vm_exec_control;
    u32 secondary_vm_exec_control;
    u64 ept_pointer;          /* L2のためのL1のEPT（GPA → L1 HPAのマッピング） */
    u64 msr_bitmap;           /* L2のためのL1のMSRビットマップ */

    /* exit/entryの情報: L1へのexitを合成するときにL0が埋める */
    u32 vm_exit_reason;
    natural_width exit_qualification;
    u64 guest_physical_address;  /* EPT violationを引き起こしたGPA */
    u32 vm_instruction_error;
    /* ... */
};
```

`natural_width`は`u64`のtypedefであり、独立したマシン型ではない。`vmcs12.h`のコメントがその理由を説明している。natural width（ネイティブのワード幅）が異なるホスト間（32ビットと64ビット）でL1（とそのL2ゲスト）をマイグレーションするには、これらのフィールドを素の`unsigned long`にはできない。そのためKVMはこれらを64ビットに固定し、x86がリトルエンディアンであることに依存している。

L0はL1のvCPUごとに1つの`struct vmcs12`を割り当て、そのポインタを`struct vcpu_vmx.nested.cached_vmcs12`（`arch/x86/kvm/vmx/vmx.h`内）に格納する。L1が`VMREAD`/`VMWRITE`を実行すると、L0は実際のVMCSに触れるのではなく、このメモリ上の構造体のフィールドを読み書きする。

## マージ: vmcs01 + vmcs12 = vmcs02 {#the-merge-vmcs01-vmcs12-vmcs02}

L0は、L1を動かすための自身のVMCSを持っている。これを**vmcs01**と呼ぶ。L1がL2を起動するとき、L0はL1のvmcs12をそのまま使うことはできない。理由は次のとおりである。

- vmcs12は、L2の状態をL1の観点で記述している（L1のHPAはL0のGPAであり得る）。
- L0は自身の制御（例: L0自身のEPTポインタ、L0のMSRビットマップ）を加える必要がある。

代わりに、L0は**vmcs02**を作成する。これは、L1の意図とL0の要件をマージした本物のVMCSである。

```
vmcs01 (L0 → L1):        vmcs12 (L1's intent for L2):     vmcs02 (L0 → L2 on real hardware):
  L0's EPT for L1           L1's EPT for L2                  A shadow EPT flattening both (see below)
  L0's MSR bitmap           L1's MSR bitmap                  Union of both bitmaps
  L0 host state             L1 host state (= L2 exit target) L0 host state
  L1 guest state            L2 guest state                   L2 guest state
```

マージは`nested_vmx_run()` → `nested_vmx_enter_non_root_mode()` → `prepare_vmcs02_early()`および`prepare_vmcs02()`を通じて行われ、いずれも`arch/x86/kvm/vmx/nested.c`にある。制御フィールドのマージが先に、`prepare_vmcs02_early()`で行われる。

```c
/* arch/x86/kvm/vmx/nested.c（簡略化したもの） */
static void prepare_vmcs02_early(struct vcpu_vmx *vmx, struct loaded_vmcs *vmcs01,
                                  struct vmcs12 *vmcs12)
{
    u32 exec_control;

    /* PIN CONTROLS: L0がL1に対して望むものと、L1がL2に対して望むものの和集合 */
    exec_control = __pin_controls_get(vmcs01);
    exec_control |= (vmcs12->pin_based_vm_exec_control &
                      ~PIN_BASED_VMX_PREEMPTION_TIMER);
    pin_controls_set(vmx, exec_control);
    /* EXEC CONTROLSとSECONDARY EXEC CONTROLSも同じ方法でマージされる */
}
```

ゲスト状態とEPTの切り替えは、その後の`prepare_vmcs02()`で行われる。

```c
/* arch/x86/kvm/vmx/nested.c（簡略化したもの） */
static int prepare_vmcs02(struct kvm_vcpu *vcpu, struct vmcs12 *vmcs12,
                           bool from_vmentry, enum vm_entry_failure_code *entry_failure_code)
{
    struct vcpu_vmx *vmx = to_vmx(vcpu);

    /* vmcs12からvmcs02へL2のゲスト状態をロードする */
    vmx_set_rflags(vcpu, vmcs12->guest_rflags);
    /* ... 他のすべてのゲスト状態フィールド ... */

    if (nested_cpu_has_ept(vmcs12))
        nested_ept_init_mmu_context(vcpu);

    return 0;
}
```

EPTのセットアップは`EPT_POINTER`への直接の書き込みではない。`nested_ept_init_mmu_context()`はvCPUを*シャドウ*EPTのMMU（`kvm_init_shadow_ept_mmu()`）に切り替え、これがハードウェアがたどるための独自の単一のフラット化されたEPTテーブルを構築する。これが性能上実際に何を意味するかについては、下のウォークのコストのセクションを参照。

## L2のexit処理: 誰が何を処理するか {#l2-exit-handling-who-handles-what}

L2がVM exitを引き起こすと、exitはまずL0に届く（ハードウェアは常に本当のホストへexitする）。L0は次のどちらかを判断しなければならない。

1. **L0が処理する**: exitはL0自身のインターセプトによって引き起こされた（例: L0のEPTにまだマップされていないメモリに対するL0のEPT violation）。L0はそれを処理し、L1に一度も見せることなくL2に再び入る。
2. **L1が処理すべき**: exitはL1が設定したvmcs12の制御フィールドの条件に一致する。L0はL1へのvmexitを合成する。

```c
/* arch/x86/kvm/vmx/nested.c */
static int vmx_check_nested_events(struct kvm_vcpu *vcpu)
{
    /* 保留中のイベントをL2に注入すべきか、
       L1への合成exitを引き起こすべきかを判断する */
    ...
}

bool nested_vmx_reflect_vmexit(struct kvm_vcpu *vcpu)
{
    /*
     * このexitをL1に"reflect"（転送）すべきならtrueを返す。
     * 判断はvmcs12の実行制御に基づく。
     *
     * 例:
     *   EXIT_REASON_CPUID  → L1がCPU_BASED_CPUID_EXITINGを設定していれば常にreflectされる
     *   EXIT_REASON_IO     → I/OポートがL1のI/Oビットマップにあればreflectされる
     *   EXIT_REASON_EPT_VIOLATION → L1がL2に対してEPTを有効にしていればreflectされる
     */
    ...
}
```

L0がexitをL1にreflectすると判断した場合:

```c
/* nested_vmx_vmexit()自体は、exit_insn_lenを計算して__nested_vmx_vmexit()に
 * 転送するだけのインラインのラッパーである — 実際の処理
 * （ここでは簡略化している）は後者で行われる: */
void __nested_vmx_vmexit(struct kvm_vcpu *vcpu, u32 vm_exit_reason,
                          u32 exit_intr_info, unsigned long exit_qualification,
                          u32 exit_insn_len)
{
    struct vcpu_vmx *vmx = to_vmx(vcpu);
    struct vmcs12 *vmcs12 = get_vmcs12(vcpu);

    leave_guest_mode(vcpu);

    /* prepare_vmcs12()はvm_exit_reason/exit_qualification/vm_exit_intr_infoを
     * vmcs12に書き込み、L1がVMREADで読み戻せるようにする。guest_physical_addressは
     * EPT violationのexitの場合にのみ、この関数の実行前に別途設定される。 */
    prepare_vmcs12(vcpu, vmcs12, vm_exit_reason, exit_intr_info, exit_qualification, 0);

    /* vmcs02からvmcs01に戻す（L1をアクティブなゲストとして復元する） */
    vmx->loaded_vmcs = &vmx->vmcs01;
    vmcs_load(vmx->loaded_vmcs->vmcs);

    /* vmcs12->host_*フィールドからL1のホスト（= L0のゲスト）状態を復元する */
    /* L1はvmcs12->host_rip（自身のvmexitハンドラ）から再開する */
}
```

## AMD-Vのネストされた仮想化 {#amd-v-nested-virtualization}

AMDはVMCSの代わりにVMCB（Virtual Machine Control Block）を使う。ネストにおける対応物は次のとおりである。

| Intel（VMX） | AMD（SVM） |
|------------|-----------|
| `struct vmcs12`（L0が持つL1のVMCSのコピー） | 構造体のコピーではない — L0はL1の制御/セーブ領域のフィールドを`nested.ctl`/`nested.save`にキャッシュし、ゲスト物理アドレス`nested.vmcb12_gpa`をキーとする |
| `prepare_vmcs02_early()`/`prepare_vmcs02()` | `nested_vmcb02_prepare_control()`/`nested_vmcb02_prepare_save()`。`arch/x86/kvm/svm/nested.c`の`nested_svm_vmrun()`から呼ばれる |
| vmcs02 | vmcb02 — L2を動かすために使われるマージ済みのVMCB |
| 2段階（シャドウ）EPT | 2段階（シャドウ）NPT — 同じ多次元ページングの手法 |
| `VMLAUNCH`/`VMRESUME` | `VMRUN` |
| `VMPTRLD`/`VMREAD`/`VMWRITE` | *（相当するものは不要 — 下記参照）* |

AMDのアプローチは概念的には似ている。L1が`VMRUN`を実行するとL0にexitし、L0はL1のVMCB（vmcb12）を自身のもの（vmcb01）とマージしてvmcb02を作り、それをロードして実際のハードウェア上でL2を動かす。しかし命令レベルの仕組みは、ある重要な点でIntelと異なる。**VMCBは、Intelの不透明なVMCSと異なり、固定された公開文書化済みのメモリ上のレイアウトを持つ**。そのため、`VMPTRLD`/`VMREAD`/`VMWRITE`に相当するものはAMDにはない。L1（およびそれをエミュレートするL0）は、通常のメモリのロードとストアでVMCBのフィールドを読み書きできる。`VMLOAD`/`VMSAVE`は無関係の、実在するSVM命令であり、VMCBのセーブ領域で扱われ*ない*少数の状態（FS/GS/LDTR/TRのセグメントベース、`KernelGSBase`、`STAR`/`LSTAR`/`CSTAR`/`SFMASK`、SYSENTERのMSR）をメモリとプロセッサの間でコピーする。ネストしているかどうかにかかわらず、すべてのSVMハイパーバイザがこれらを使う。

## 2段階EPT: ハードウェアが行うことではなく、KVMが構築するもの {#two-level-ept-not-what-hardware-does-what-kvm-builds}

VMXもSVMも、1回のメモリアクセスのために2つのEPT/NPTテーブルを続けてたどる手段をハードウェアに与えていない。「EPT-on-EPT」のような命令レベルの機能は存在しない。もしL0が素朴にL1自身のエミュレートされたEPTをL2に使わせたなら、L2のすべてのメモリアクセスについて、ソフトウェアがL2 GPA → L1 HPA（`vmcs12->ept_pointer`経由）を解決し、続いてL1 HPA（実際にはL0 GPAである）→ L0 HPA（L0自身のEPT経由）を解決する必要がある。これが「shadow-on-EPT」のアプローチであり、正しいが遅い。L2のすべてのページフォールトとページテーブルへの書き込みを、処理のためにL1までトラップしなければならないからである。

KVMは代わりに**多次元ページング**を使う。`nested_ept_init_mmu_context()`（`arch/x86/kvm/vmx/nested.c`）は、vCPUを*シャドウ*EPTのMMU（`kvm_init_shadow_ept_mmu()`、`arch/x86/kvm/mmu/mmu.c`）に切り替える。これは2つの段階を1つのEPTテーブル（EPT02と呼ぶ）にフラット化し、ハードウェアはそれを通常のネストしていないEPTのウォークと同じコストで直接たどる。L0はEPT02を遅延的に構築する。EPT violationが起きるたびに、L1のEPT構造（`vmcs12->ept_pointer`）をソフトウェアでたどってエントリを組み立てる。これは古典的なシャドウページングとまったく同じである。AMDのネストされたSVMのコードも、NPTに対して同じ手法を使う。

したがって、オーバーヘッドはメモリアクセスのたびに支払われるのではなく、シャドウEPT02のエントリを構築または無効化しなければならないとき（ページへの最初のアクセス時、またはL1が自身のEPTを変更した後や`INVEPT`を実行した後）に支払われる。これは、具体的な命令の組み合わせが重要である理由でもある。多くのページフォールトを引き起こすワークロードは、構築済みのEPT02エントリにほぼヒットするワークロードよりも、はるかに大きなネストのオーバーヘッドを被る。

測定値について: IBMのTurtlesプロジェクト — [ネストされたVMXのカーネルドキュメント](https://docs.kernel.org/virt/kvm/x86/nested-vmx.html)自身が「the theory behind the nested VMX feature, its implementation and its performance characteristics」（ネストされたVMX機能の背後にある理論、その実装、性能特性）について引用している論文 — は、この多次元ページングの手法を用いて、ネストされたKVMが**一般的なワークロードにおいて単一レベルの仮想化の6～8%以内**に収まると報告しており、CPUバウンドのマクロベンチマーク（`kernbench`、`SPECjbb`）では最適化レベルに応じて6～15%のオーバーヘッドを測定している。同じ論文は、ページフォールトの多いワークロードにおいて多次元ページングが素朴なshadow-on-EPTのアプローチに比べて*約3倍*高速になることを測定し、またI/Oの多いネストされたワークロード（特に割り込みの多いもの）がCPUバウンドのものよりかなり高コストであることを見出している。完全な測定結果は[The Turtles Project](https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Ben-Yehuda.pdf)（USENIX OSDI 2010）を参照。

## KVM_CAP_NESTED_STATE: ネストされたゲストのマイグレーション {#kvm_cap_nested_state-migrating-nested-guests}

自身がハイパーバイザ（L1）を動かしているVMをライブマイグレーションするには、ネストされたVMX/SVMの状態を保存する必要がある。この状態には、vmcs12、vmcb12、および関連する制御構造が含まれる。

```c
/* ネストされた状態のサイズを確認する */
int size = ioctl(vcpu_fd, KVM_CHECK_EXTENSION, KVM_CAP_NESTED_STATE);

/* ネストされた状態を取得する */
struct kvm_nested_state *state = malloc(size);
state->size = size;
ioctl(vcpu_fd, KVM_GET_NESTED_STATE, state);
/* state->formatがVMXとSVMを区別する。その後にvmcs12のデータなどが続く */

/* 移行先で復元する */
ioctl(vcpu_fd, KVM_SET_NESTED_STATE, state);
```

`struct kvm_nested_state`は`arch/x86/include/uapi/asm/kvm.h`で定義される。`format`フィールドは、VMX（`KVM_STATE_NESTED_FORMAT_VMX`）とSVM（`KVM_STATE_NESTED_FORMAT_SVM`）を区別する。ヘッダに続く可変長のデータには、vmcs12/vmcb12の内容と、あればシャドウVMCSが含まれる。

## ネストされた仮想化の観測 {#observing-nested-virtualization}

```bash
# L2が動作していることを確認する（L1ゲスト内で）
cat /proc/cpuinfo | grep hypervisor   # L2ではhypervisorフラグが存在する

# L0: ネストされたexitが発生していることを確認する
echo 1 > /sys/kernel/tracing/events/kvm/kvm_nested_vmexit/enable
cat /sys/kernel/tracing/trace_pipe
# qemu-1234 [000] kvm_nested_vmexit: rip=... reason=EPT_VIOLATION ...

# perf: ネストあり/なしでexitの頻度を比較する
perf kvm stat report   # EPT_VIOLATION、VMLAUNCH/VMRESUMEのカウントの増加を探す

# L0のdebugfs: vCPUごとのnested-runカウンタ（KVMアーキテクチャのvcpuN/階層を参照）
cat /sys/kernel/debug/kvm/<pid>-<vm-fd-name>/vcpuN/nested_run

# モジュールパラメータ（ブート後は読み取り専用。ロード時に設定する）
cat /sys/module/kvm_intel/parameters/nested   # YまたはN
cat /sys/module/kvm_amd/parameters/nested     # 1または0
```

## セキュリティ上の考慮事項 {#security-considerations}

ネストされた仮想化は大きな攻撃対象領域を露出させる。L1は任意のVMCS/VMCBの値を細工してL0を混乱させようとすることができる。

- **vmcs12のフィールドの検証**: `nested_vmx_check_controls()`、`nested_vmx_check_host_state()`、`nested_vmx_check_guest_state()`は、L2に入る前にvmcs12のすべての制御フィールドと状態フィールドを検証する。不正な組み合わせは、ハードウェアの挙動に合わせて、`VM_INSTRUCTION_ERROR`を設定した`VM_FAIL`を引き起こす。
- **EPTポインタの健全性**: L0は`vmcs12->ept_pointer`のアラインメントと予約ビットを検証する。
- **Spectre/Meltdownの緩和策**: L2がL1を通じてL0の分岐予測に影響を与えるのを防ぐため、L0はL2 → L1 → L0の遷移時に分岐予測器の状態をフラッシュしなければならない。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [arch/x86/kvm/vmx/nested.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kvm/vmx/nested.c) — IntelのネストされたVMX: `handle_vmxon()`、`handle_vmptrld()`、`handle_vmlaunch()`/`handle_vmresume()` → `nested_vmx_run()`、`prepare_vmcs02_early()`/`prepare_vmcs02()`、`vmx_check_nested_events()`、`nested_vmx_reflect_vmexit()`、`nested_vmx_vmexit()`/`__nested_vmx_vmexit()`
- [arch/x86/kvm/vmx/vmcs12.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kvm/vmx/vmcs12.h) — `struct vmcs12`の定義。L2のためのVMCSをL1から見たもの
- [arch/x86/kvm/svm/nested.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kvm/svm/nested.c) — AMDのネストされたSVM: `nested_svm_vmrun()`、`nested_vmcb02_prepare_control()`/`nested_vmcb02_prepare_save()`（SVMにおける`prepare_vmcs02_early()`/`prepare_vmcs02()`の対応物）
- [include/uapi/linux/kvm.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/kvm.h) — `KVM_GET_NESTED_STATE`/`KVM_SET_NESTED_STATE` ioctlの定義と`KVM_CAP_NESTED_STATE`
- [arch/x86/include/uapi/asm/kvm.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/uapi/asm/kvm.h) — `struct kvm_nested_state`と、`KVM_STATE_NESTED_FORMAT_VMX`/`KVM_STATE_NESTED_FORMAT_SVM`のフォーマット定数

### 関連ページ {#related-pages}

- [KVMアーキテクチャ](kvm-arch.md) — VMCSの構造、vCPUの実行ループ
- [KVMのexit処理](kvm-exits.md) — exitのディスパッチテーブル、exit理由の処理
- [ライブマイグレーション](live-migration.md) — L1ハイパーバイザをマイグレーションするための`KVM_GET_NESTED_STATE`
- [メモリ仮想化](kvm-memory.md) — EPT/NPT、2段階のページテーブルウォーク

### 外部資料 {#external}

- [Nested VMX](https://docs.kernel.org/virt/kvm/x86/nested-vmx.html) — IntelのネストされたVMXのカーネルドキュメント: L0/L1/L2モデル、vmcs01/vmcs12/vmcs02、`struct vmcs12`の全フィールドのレイアウト
- [KVM API docs: `KVM_GET_NESTED_STATE`/`KVM_SET_NESTED_STATE`](https://docs.kernel.org/virt/kvm/api.html#kvm-get-nested-state) — ネストされたゲストのライブマイグレーションに使われる`struct kvm_nested_state`のレイアウトと`KVM_CAP_NESTED_STATE`
- [The Turtles Project: Design and Implementation of Nested Virtualization](https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Ben-Yehuda.pdf)（USENIX OSDI 2010） — 多次元ページング、シャドウEPT/NPTの仕組み、ネストされた仮想化の性能オーバーヘッドの測定値
