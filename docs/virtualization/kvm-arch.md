# KVMアーキテクチャ {#kvm-architecture}

> Linuxカーネルがタイプ2ハイパーバイザになる仕組み

## Intel VT-x: ハードウェア仮想化の基礎 {#intel-vt-x-hardware-virtualization-basics}

Intel VT-xは、2つの新しいCPU動作モードを追加する。

- **VMX root mode**: ハイパーバイザ（KVM）が動作するモード。完全な特権を持つ
- **VMX non-root mode**: ゲストOSが動作するモード。ハードウェアによって制限される

中核となるデータ構造は**VMCS**（Virtual Machine Control Structure）である。これはvCPUごとのハードウェア構造体で、ゲストとホストの状態を格納し、どのような場合にVM exitが発生するかを制御する。

```
Host CPU (ring 0)                    VMCS
    │                          ┌────────────────────┐
    │                          │ Guest state area:  │
    │  VMLAUNCH/VMRESUME ──────►  rip, rsp, cr3,    │
    │                          │  cs, es, tr, ...   │
    │  ◄─── VM Exit ──────────  ├────────────────────┤
    │                          │ Host state area:   │
    │                          │  (restored on exit)│
    │                          ├────────────────────┤
    │                          │ Control fields:    │
    │                          │  - exit reasons    │
    │                          │  - VMCS pointers   │
    │                          │  - MSR bitmaps     │
    │                          └────────────────────┘
```

## /dev/kvm API {#the-devkvm-api}

KVMは、`/dev/kvm`および`KVM_CREATE_VM`と`KVM_CREATE_VCPU`が返す無名ファイルディスクリプタに対するioctlを通じて機能を公開する。

```c
#include <linux/kvm.h>
#include <sys/ioctl.h>

/* ステップ1: KVM APIを開く */
int kvm_fd = open("/dev/kvm", O_RDWR);

/* ステップ2: VMを作成する */
int vm_fd = ioctl(kvm_fd, KVM_CREATE_VM, 0);

/* ステップ3: メモリをセットアップする */
struct kvm_userspace_memory_region region = {
    .slot            = 0,
    .guest_phys_addr = 0x0,      /* ゲスト物理アドレス */
    .memory_size     = 0x100000, /* 1MB */
    .userspace_addr  = (uint64_t)mmap(NULL, 0x100000,
                                       PROT_READ|PROT_WRITE,
                                       MAP_PRIVATE|MAP_ANONYMOUS, -1, 0),
};
ioctl(vm_fd, KVM_SET_USER_MEMORY_REGION, &region);

/* ステップ4: vCPUを作成する */
int vcpu_fd = ioctl(vm_fd, KVM_CREATE_VCPU, 0);

/* ステップ5: vCPUのrun構造体をマップする */
int vcpu_mmap_size = ioctl(kvm_fd, KVM_GET_VCPU_MMAP_SIZE, 0);
struct kvm_run *run = mmap(NULL, vcpu_mmap_size,
                           PROT_READ|PROT_WRITE, MAP_SHARED, vcpu_fd, 0);

/* ステップ6: vCPUの初期状態（レジスタ）を設定する */
struct kvm_regs regs = { .rip = 0x0, .rsp = 0x10000, .rflags = 0x2 };
ioctl(vcpu_fd, KVM_SET_REGS, &regs);

/* ステップ7: vCPUを実行する */
while (1) {
    ioctl(vcpu_fd, KVM_RUN, 0);  /* VMX non-root modeに入る */

    switch (run->exit_reason) {
    case KVM_EXIT_HLT:
        printf("Guest halted\n");
        return 0;
    case KVM_EXIT_IO:
        /* ゲストがI/O（in/out命令）を行った */
        handle_io(run);
        break;
    case KVM_EXIT_MMIO:
        /* ゲストがマップされていないMMIO領域にアクセスした */
        handle_mmio(run);
        break;
    case KVM_EXIT_INTERNAL_ERROR:
        fprintf(stderr, "KVM internal error\n");
        return 1;
    }
}
```

## VM exitとその原因 {#vm-exits-and-their-causes}

VM exitは、制御をゲストからホストへ戻す。代表的なexit理由は次のとおりである。

```c
/* include/uapi/linux/kvm.h — exit理由 */
KVM_EXIT_IO              /* PMIO（in/out）命令 */
KVM_EXIT_MMIO            /* メモリマップドI/O */
KVM_EXIT_HYPERCALL       /* ゲストがハイパーコールを呼んだ（vmcall/vmmcall） */
KVM_EXIT_DEBUG           /* デバッグイベント */
KVM_EXIT_HLT             /* hlt命令 */
KVM_EXIT_FAIL_ENTRY      /* VM entryが失敗した */
KVM_EXIT_INTR            /* KVM_RUN中にシグナルを受信した */
KVM_EXIT_SHUTDOWN        /* トリプルフォルトまたはマシンチェック */
KVM_EXIT_TPR_ACCESS      /* TPR（タスクプライオリティレジスタ）へのアクセス */
KVM_EXIT_X86_RDMSR       /* RDMSR命令 */
KVM_EXIT_X86_WRMSR       /* WRMSR命令 */
KVM_EXIT_DIRTY_RING_FULL /* ダーティページ追跡用のリングが満杯 */
KVM_EXIT_AP_RESET_HOLD   /* マルチプロセッサのブート */
```

頻度が重要である。頻繁なexit（タイマーティックやMMIOなど）は、CPU状態の切り替えを必要とするためコストが高い。APICv、posted interrupt、EPTといったハードウェア機能はexitを減らす。

## struct kvmとstruct kvm_vcpu {#struct-kvm-and-struct-kvm_vcpu}

```c
/* 簡略化したもの — 実際のstruct kvmはinclude/linux/kvm_host.hで定義される。
 * アーキテクチャ固有の状態（page-track notifierのヘッドを含む）は、
 * 埋め込まれたstruct kvm_archにあり、これはアーキテクチャごとに定義される
 * （例: arch/x86/include/asm/kvm_host.h）。ここではない。 */
struct kvm {
    spinlock_t           mmu_lock;      /* KVM_HAVE_MMU_RWLOCKを持つ
                                            アーキテクチャではrwlock_t */
    struct mutex         slots_lock;
    struct mutex         slots_arch_lock;

    struct mm_struct     *mm;            /* このVMに結び付いたユーザー空間 */
    unsigned long        nr_memslot_pages;
    struct kvm_memslots  *memslots[KVM_MAX_NR_ADDRESS_SPACES];

    struct xarray        vcpu_array;     /* vcpu_idxでインデックスされる。単純な
                                             配列ではない — 下のvcpu_idxを参照 */
    atomic_t             online_vcpus;
    int                  max_vcpus;
    int                  created_vcpus;

    struct list_head     vm_list;
    struct mutex         lock;
    struct kvm_io_bus    *buses[KVM_NR_BUSES];
    struct kvm_irq_routing_table *irq_routing;
    struct hlist_head    irq_ack_notifier_list;

    struct kvm_arch      arch;           /* アーキテクチャ固有の状態 */
};

struct kvm_vcpu {
    struct kvm          *kvm;
    int                  cpu;         /* 現在ロードされている物理CPU。
                                          ロードされていなければ-1（vcpu_load()/vcpu_put()を参照） */
    int                  vcpu_id;     /* 作成時にユーザー空間が与えたID */
    int                  vcpu_idx;    /* kvm->vcpu_arrayへのインデックス */

    int                  ____srcu_idx; /* 直接使わないこと — 先頭の
                                           アンダースコアは意図的なもの */
    int                  mode;        /* IN_GUEST_MODEなど */
    u64                  requests;    /* 保留中のリクエストのビットマスク */
    unsigned long        guest_debug; /* デバッグフラグ */

    struct mutex         mutex;
    struct kvm_run       *run;        /* mmapされたrun構造体 */

    struct rcuwait        wait;       /* halt時のブロック用 */

    struct kvm_vcpu_arch  arch;       /* x86/arm64固有の状態 */
    struct kvm_vcpu_stat  stat;

    /* 変更されたゲストページを追跡するためのダーティリング */
    struct kvm_dirty_ring dirty_ring;
};
```

## vCPUの実行ループ {#the-vcpu-run-loop}

`kvm_arch_vcpu_ioctl_run()`は`KVM_RUN` ioctlの直接のハンドラであり、主に状態を検証して`vcpu_run()`に処理を渡すだけである。`vcpu_run()`は、ゲストへのentry/exitごとに1回`vcpu_enter_guest()`を呼び出すループを回す。

```c
/* arch/x86/kvm/x86.c: vcpu_enter_guest()、簡略化したもの */
static int vcpu_enter_guest(struct kvm_vcpu *vcpu)
{
    fastpath_t exit_fastpath;
    u64 run_flags = 0;   /* 保留中のexit、デバッグレジスタ、debugctlの状態から
                          * 事前に組み立てられる。ここでは省略 */
    int r;

    /* ゲストに入る前に保留中の作業を処理する */
    if (kvm_check_request(KVM_REQ_TLB_FLUSH, vcpu))
        kvm_vcpu_flush_tlb_all(vcpu);
    /* ... 他のリクエストを処理する ... */

    /* 保留中の割り込みがあれば注入する */
    if (kvm_cpu_has_injectable_intr(vcpu))
        kvm_x86_call(inject_irq)(vcpu, false /* reinjected */);

    /* VM Entry */
    exit_fastpath = kvm_x86_call(vcpu_run)(vcpu, run_flags);  /* VMLAUNCH/VMRESUME */

    /* VM Exit: 理由に応じて処理する */
    r = kvm_x86_call(handle_exit)(vcpu, exit_fastpath);

    return r;
}
```

`kvm_x86_call(op)`は、`struct kvm_x86_ops`への`static_call`をラップするマクロである。ベンダーのバックエンド（VMXまたはSVM）は、`.vcpu_run`、`.handle_exit`、`.inject_irq`といったフィールドを通じて組み込まれる。`vcpu_run()`は、exit理由がユーザー空間を必要とするまで、このentryごとの処理を`for (;;)`ループで繰り返し、その時点で`kvm_arch_vcpu_ioctl_run()`へ戻る。

## ハイパーコール: ゲストからホストへの通信 {#hypercalls-guest-to-host-communication}

```c
/* ゲスト: ハイパーコールを発行する */
/* x86では: vmcall（Intel）またはvmmcall（AMD） */
/* arch/x86/include/asm/kvm_para.hは引数の数ごとに1つのバリアント、
 * kvm_hypercall0()からkvm_hypercall4()までを定義している — 単一の可変長引数の
 * kvm_hypercall()は存在しない。引数ゼロの例: */
static inline long kvm_hypercall0(unsigned int nr)
{
    long ret;
    asm volatile("vmcall"
                 : "=a"(ret)
                 : "a"(nr)
                 : "memory");
    return ret;
}

/* KVMのハイパーコール番号（include/uapi/linux/kvm_para.h） */
KVM_HC_VAPIC_POLL_IRQ     /* 仮想APICの割り込みをポーリングする */
KVM_HC_MMU_OP             /* MMU操作 */
KVM_HC_FEATURES           /* KVMの機能フラグを取得する */
KVM_HC_PPC_MAP_MAGIC_PAGE /* PowerPC */
KVM_HC_KICK_CPU           /* 別のvCPUをキックする（IPI） */
KVM_HC_CLOCK_PAIRING      /* 時刻同期 */
KVM_HC_SEND_IPI           /* vCPUのビットマップにIPIを送る */
KVM_HC_SCHED_YIELD        /* 別のvCPUに譲る */
KVM_HC_MAP_GPA_RANGE      /* メモリ属性の通知 */
```

## KVMの観測 {#observing-kvm}

```bash
# VMごとに1つのディレクトリ。名前は<creator-pid>-<vm-fd-name>（debugfsが必要）:
ls /sys/kernel/debug/kvm/
# 1234-3/   ...

# vCPUごとにサブディレクトリがあり、その中に統計ごとのファイルがある
# （arch/x86/kvm/x86.cのkvm_vcpu_stats_desc[]を参照）— 集約された"exits"ファイルはない:
ls /sys/kernel/debug/kvm/1234-3/vcpu0/
# exits  io_exits  mmio_exits  halt_exits  irq_window_exits  nmi_window_exits  ...

cat /sys/kernel/debug/kvm/1234-3/vcpu0/exits
# 12345678

# perfによるKVMの統計
perf kvm stat record -a sleep 10
perf kvm stat report

# KVMのtracepoint
ls /sys/kernel/tracing/events/kvm/
echo 1 > /sys/kernel/tracing/events/kvm/kvm_exit/enable
cat /sys/kernel/tracing/trace_pipe
# bash-1234 [003] kvm_exit: reason EXTERNAL_INTERRUPT rip 0xffff...
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/kvm_host.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/kvm_host.h) — `struct kvm`と`struct kvm_vcpu`の定義
- [arch/x86/kvm/x86.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kvm/x86.c) — `kvm_arch_vcpu_ioctl_run()`、`vcpu_run()`、`vcpu_enter_guest()`: vCPUの実行ループ（リクエスト処理、VM entry、exitのディスパッチ）
- [arch/x86/kvm/vmx/vmx.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kvm/vmx/vmx.c) — Intel VT-xのバックエンド: VMLAUNCH/VMRESUME、`vmx_inject_irq(vcpu, bool reinjected)`、VMCSベースのVM entry/exit
- [virt/kvm/kvm_main.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/virt/kvm/kvm_main.c) — `/dev/kvm`のコア: `KVM_CREATE_VM`を処理する`kvm_dev_ioctl()`、得られたVMのfdに対する`KVM_CREATE_VCPU`を処理する`kvm_vm_ioctl()`
- [include/uapi/linux/kvm.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/kvm.h) — `KVM_EXIT_*`のexit理由定数と`kvm_run`のABI
- [include/uapi/linux/kvm_para.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/kvm_para.h) — `KVM_HC_*`ハイパーコール番号

### 関連ページ {#related-pages}

- [メモリ仮想化](kvm-memory.md) — EPT、シャドウページング、バルーン
- [virtio](virtio.md) — I/Oの準仮想化
- [メモリ管理: ページテーブル](../mm/page-tables.md) — EPTはx86のページングの上に構築されている

### 外部資料 {#external}

- [The KVM API](https://docs.kernel.org/virt/kvm/api.html) — `/dev/kvm`の公式ioctlリファレンス: `KVM_CREATE_VM`、`KVM_CREATE_VCPU`、`KVM_SET_USER_MEMORY_REGION`、`KVM_RUN`、および`struct kvm_run`のexit理由の共用体
- [KVM VCPU Requests](https://docs.kernel.org/virt/kvm/vcpu-requests.html) — vCPUの実行ループで使われる`vcpu->requests`ビットマスクの仕組み（`kvm_check_request()`、`KVM_REQ_TLB_FLUSH`）
