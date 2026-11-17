# VFIOの内部 {#vfio-internals}

> デバイスパススルー: コンテナ、グループ、mdev、KVMとの統合

## 概要 {#overview}

VFIO（Virtual Function I/O）は、物理デバイスをユーザー空間のプロセスや仮想マシンに安全に公開するためのカーネルフレームワークである。IOMMUを使ってメモリアクセスの分離を強制しつつ、ユーザー空間ドライバ（またはハイパーバイザ）にデバイスのMMIO、割り込み、DMAへの直接アクセスを与える。

主なユースケースは次の3つである。

- **VMへのデバイスパススルー**: QEMU/KVMが物理NIC、GPU、NVMeをゲストに渡す。
- **ユーザー空間ドライバ**: DPDKがカーネルバイパスのパケット処理のためにNICをVFIOにバインドする。
- **Mediated device**: 物理GPUやNICを仮想インスタンスに分割し、別々のVMに割り当てる。

VFIOのソースは`drivers/vfio/`にあり、ヘッダは`include/linux/vfio.h`にある。

## コンテナ → グループ → デバイスの階層 {#container-group-device-hierarchy}

```
/dev/vfio/vfio          (VFIO container)
    │
    ├── /dev/vfio/12    (IOMMU group 12)
    │       ├── 0000:01:00.0   (GPU)
    │       └── 0000:01:00.1   (GPU HDMI audio)
    │
    └── /dev/vfio/7     (IOMMU group 7)
            └── 0000:03:00.0   (NVMe)
```

**コンテナ**: IOMMUドメインである。コンテナに追加されたすべてのグループは1つのIOMMUアドレス空間を共有する。コンテナは`VFIO_IOMMU_MAP_DMA`呼び出し（ゲストの物理メモリをデバイスのIOVA空間にマッピングする）を受け取る単位である。

**グループ**: IOMMUグループ（分離可能なデバイスの最小集合）に対応する。グループのデバイスを開く前に、グループをコンテナに追加しなければならない。グループ内のすべてのデバイスはコンテナのIOMMUドメインを共有する。同じグループのデバイスを別々のVMに割り当てることはできない。

**デバイス**: 個々のPCIファンクションである。グループがコンテナに入ると、デバイスのfdを通じてMMIO領域のマッピング、割り込みの設定、デバイスのリセットが可能になる。

## 中核となるデータ構造 {#core-data-structures}

```c
/* include/linux/vfio.h */
struct vfio_device {
    struct device            *dev;
    const struct vfio_device_ops *ops;
    struct vfio_group        *group;
    /* 6.x 以降、IOMMU との結び付けは iommufd_device 経由 */
    /* ... */
};

/* drivers/vfio/vfio.h -- VFIO コア専用。上の公開ヘッダではない */
struct vfio_group {
    struct device             dev;
    struct cdev                cdev;
    refcount_t                drivers;
    unsigned int               container_users;
    struct iommu_group       *iommu_group;
    struct vfio_container    *container;
    struct list_head          device_list;
    /* ... */
};

/* drivers/vfio/container.c -- ドメインへのポインタはない。コンテナは
 * 設定されている IOMMU バックエンド（iommu_driver）に処理を振り分けるだけ */
struct vfio_container {
    struct kref                kref;
    struct list_head          group_list;
    struct rw_semaphore       group_lock;
    struct vfio_iommu_driver *iommu_driver;
    void                     *iommu_data; /* ドライバ固有（例: vfio_iommu） */
    bool                        noiommu;
};
```

`vfio_device_ops`構造体は、デバイスごとの操作（`open_device`/`close_device`、read、write、MMIO/割り込み/リセット用のioctl）を定義する。

## IOMMU type1バックエンド {#iommu-type1-backend}

最も一般的なIOMMUバックエンドは`vfio_iommu_type1`（`drivers/vfio/vfio_iommu_type1.c`）である。Intel VT-dまたはAMD-Viを持つx86システムと、ARM SMMUを持つarm64システムを扱う。

### DMAマッピング: ピン留めとマッピング {#dma-mapping-pinning-and-mapping}

QEMUが`VFIO_IOMMU_MAP_DMA`を呼ぶと、カーネルは次のことをしなければならない。

1. **ホストのページをピン留めする**（ユーザー仮想アドレス → 物理ページ。スワップを防ぐ）。
2. **IOMMUドメインにマップする**（IOVA → 物理アドレス。デバイスがゲストメモリにDMAできるようにする）。

```c
/* drivers/vfio/vfio_iommu_type1.c（簡略版） */
static int vfio_dma_do_map(struct vfio_iommu *iommu,
                            struct vfio_iommu_type1_dma_map *map)
{
    unsigned long vaddr = map->vaddr;
    dma_addr_t iova = map->iova;
    size_t size = map->size;

    /* ピン留めの前に、まずマッピングを rb-tree に記録する */
    dma = vfio_find_dma(iommu, iova, size);  /* 重複がないことを確認 */
    vfio_link_dma(iommu, dma);               /* ツリーに挿入 */

    /* ページのピン留めと IOMMU ページテーブルエントリの作成 -- どちらも
     * 別々のステップではなく、この 1 回の呼び出しの中で行われる:
     * vfio_pin_map_dma() は、iommu->domain_list 内の各ドメインについて、
     * ピン留め（HVA → HPA、呼び出しチェーンの先の pin_user_pages_remote() 経由）と
     * マップ（IOVA → HPA）をドメインごとにまとめて行う（コンテナは
     * 複数の IOMMU ドメインにまたがりうる -- 単一の
     * iommu->domain ポインタではない） */
    vfio_pin_map_dma(iommu, dma, size);
}
```

`pin_user_pages_remote()`呼び出し（DMA用のピン留めで`get_user_pages_remote()`を置き換えるためにカーネル5.6で導入された）は、ページの参照カウントを増やしてページをピン留め済みとしてマークし、デバイスがアクセスしている可能性のある間、ページがスワップされたり移動されたりするのを防ぐ。

### DMAマップのrb-tree {#the-dma-map-rb-tree}

すべてのアクティブなDMAマッピングは赤黒木で追跡され、IOVAをキーとして重複が探索される（`vfio_find_dma()`）。radix treeやカーネルのinterval tree APIではない。これによりカーネルは次のことができる。

- 重複する`MAP_DMA`要求を検出して拒否する。
- グループが削除されたときに、クリーンアップのためすべてのマッピングを列挙する。
- エントリを検索して削除することで`VFIO_IOMMU_UNMAP_DMA`を実装する。

```c
/* struct vfio_iommu がマッピングを追跡する */
struct vfio_iommu {
    struct list_head       domain_list; /* 1 つのコンテナは複数の IOMMU ドメインにまたがりうる */
    struct rb_root          dma_list;   /* vfio_dma の rb-tree。IOVA で重複を探索 */
    unsigned int            dma_avail;   /* 残りの DMA マップ数の上限 */
    /* ... */
};

struct vfio_domain {
    struct iommu_domain    *domain;
    struct list_head       next;
    struct list_head       group_list;
    /* ... */
};

struct vfio_dma {
    struct rb_node        node;
    dma_addr_t            iova;
    unsigned long         vaddr;
    size_t                size;
    int                   prot;        /* IOMMU_READ | IOMMU_WRITE */
    struct rb_root        pfn_list;    /* ピン留めされたページ */
};
```

### IOMMUバックエンドの種類 {#iommu-backend-variants}

| バックエンド | 使われる場所 | 備考 |
|---------|-----------|-------|
| `vfio_iommu_type1` | x86（VT-d、AMD-Vi）、arm64（SMMU） | 標準。pin_user_pages + iommu_map |
| `vfio_iommu_spapr_tce` | PowerPC（POWER8/9 TCE IOMMU） | TCE（Translation Control Entry）テーブル |
| `vfio-noiommu` | IOMMUが存在しない場合 | 安全でない。分離なし。`enable_unsafe_noiommu_mode=1`が必要 |

`vfio-noiommu`は、IOMMUを持たない開発ボードや組み込みシステムのために存在する。VFIOのユーザー空間APIを提供するが、分離はまったく提供しない。デバイスはどこにでもDMAできる。これはコンパイル時のオプションであり、dmesgに警告を出力する。

## MSI/MSI-X割り込みの配送 {#msimsi-x-interrupt-delivery}

VFIOで最も複雑な部分の1つが、ユーザー空間との往復なしにデバイスの割り込みをVMに配送することである。MSI配送の全経路は次のとおり。

```
Device raises MSI
    │
    ▼
IOMMU interrupt remapping table (translates MSI address/data → vector)
    │
    ▼
Host CPU vector delivery
    │
    ▼
KVM irqfd: bypasses userspace entirely
    │
    ▼
KVM in-kernel irqchip (APIC emulation)
    │
    ▼
vCPU interrupt injection → guest ISR
```

### vfio_irq_ctxとeventfd {#vfio_irq_ctx-and-eventfd}

```c
/* drivers/vfio/pci/vfio_pci_intrs.c */
struct vfio_pci_irq_ctx {
    struct vfio_pci_core_device *vdev;    /* この IRQ コンテキストが属するデバイス */
    struct eventfd_ctx        *trigger;  /* 割り込み通知用の eventfd */
    struct virqfd             *unmask;   /* レベルトリガ IRQ 用 */
    struct virqfd             *mask;
    char                      *name;
    bool                       masked;
    struct irq_bypass_producer producer;
};
```

QEMUが`VFIO_DEVICE_SET_IRQS`で割り込みを設定すると、VFIOはMSIベクタごとに`eventfd`を作成する。デバイスのMSIハンドラは`eventfd_signal(ctx->trigger)`を呼び、eventfdで待っている者を起こす。

### irqfd: 高速パス {#irqfd-the-fast-path}

鍵となる最適化が`irqfd`である。これは、ユーザー空間に戻ることなくeventfdから割り込みを注入するKVMの仕組みである。

```c
/* KVM irqfd (virt/kvm/eventfd.c) */
/*
 * QEMU は KVM_IRQFD ioctl を呼んで、eventfd をゲストの IRQ ラインに関連付ける。
 * eventfd が（VFIO の MSI ハンドラによって）シグナルされると、irqfd の workqueue が
 * ホストカーネル内で直接 kvm_set_irq() を実行する。
 */
```

irqfdを使った流れ:
1. QEMUが`KVM_IRQFD`を設定し、eventfd → ゲストIRQ番号を結び付ける。
2. デバイスのMSIが発火する → ホストの割り込みハンドラ → `eventfd_signal()`。
3. irqfdの待ちキューが起こされる → `kvm_set_irq()`が実行される → ゲストのvCPUに割り込みを注入する。

ステップ2〜3にユーザー空間は一切関与しない。ゲストは、デバイスがMSIを上げてから数マイクロ秒以内に割り込みを受け取る。

レガシーなINTx割り込みでは経路はより遅い。VFIOはレベルトリガのセマンティクスを扱うために、`irqfd`に加えてリサンプル機構を使う。

## mdev: Mediated Device {#mdev-mediated-devices}

mdevは、内部リソース（GPUの計算エンジン、NICのキュー、暗号エンジン）を持つ物理デバイスを、複数の独立した仮想デバイスに分割できるようにする。それぞれを別のVMに割り当てられる。

### アーキテクチャ（カーネル5.x） {#architecture-kernel-5x}

```c
/* include/linux/mdev.h — 現在の構造体の形。この節がそれ以外の部分で説明している
 * mdev_parent_ops ベースの設計はこれより前のもので、置き換えられた
 * （下記の「struct vfio_device_ops」を参照） */
struct mdev_parent {
    struct device            *dev;         /* 物理デバイス（PF） */
    struct mdev_driver       *mdev_driver;
    struct kset              *mdev_types_kset;  /* サポートする type ごとに 1 エントリ */
    struct rw_semaphore       unreg_sem;
    struct mdev_type        **types;       /* 各 type はリソースの一切れ */
    unsigned int              nr_types;
    atomic_t                  available_instances;
};

struct mdev_device {
    struct device             dev;
    guid_t                    uuid;        /* mdev インスタンスを識別する */
    struct list_head          next;
    struct mdev_type         *type;        /* このインスタンスの作成元のリソーススライス */
    bool                      active;
};
```

物理デバイスのドライバは、サポートするmdev typeとともに`mdev_parent`を登録する。各typeはリソースの一切れ（例: 「GPUの1/4」）を表す。ユーザー空間はsysfs経由でインスタンスを作成する。

```bash
# GPU 0000:01:00.0 上に nvidia vGPU の mdev インスタンスを作成する
UUID=$(uuidgen)
echo $UUID > /sys/bus/pci/devices/0000:01:00.0/mdev_supported_types/nvidia-35/create
# nvidia-35 = GRID RTX 6000-1Q（1GB の vGPU type）

# mdev は新しいデバイスとして現れる:
ls /sys/bus/mdev/devices/
# $UUID
```

### struct vfio_device_ops（カーネル5.19以降） {#struct-vfio_device_ops-kernel-519}

Linux 5.19で、古い`mdev_parent_ops`インターフェースは`struct vfio_device_ops`に統合された（コミット`6b42f491e17c`、"vfio/mdev: Remove mdev_parent_ops"）。mdevをサポートする物理デバイスのドライバは、現在では`vfio_device_ops`を直接実装する。

```c
/* include/linux/vfio.h */
struct vfio_device_ops {
    char *name;

    int  (*init)(struct vfio_device *vdev);
    void (*release)(struct vfio_device *vdev);

    /* IOMMUFD のライフサイクル — デバイスを iommufd コンテキストにバインドし、
     * IOAS/hwpt にアタッチする。このスケッチでは扱わない。下記を参照 */
    int  (*bind_iommufd)(struct vfio_device *vdev, struct iommufd_ctx *ictx, u32 *out_device_id);
    void (*unbind_iommufd)(struct vfio_device *vdev);
    int  (*attach_ioas)(struct vfio_device *vdev, u32 *pt_id);
    void (*detach_ioas)(struct vfio_device *vdev);

    int  (*open_device)(struct vfio_device *vdev);
    void (*close_device)(struct vfio_device *vdev);

    ssize_t (*read)(struct vfio_device *vdev, char __user *buf,
                    size_t count, loff_t *ppos);
    ssize_t (*write)(struct vfio_device *vdev, const char __user *buf,
                     size_t count, loff_t *ppos);
    long    (*ioctl)(struct vfio_device *vdev, unsigned int cmd,
                     unsigned long arg);

    int  (*mmap)(struct vfio_device *vdev, struct vm_area_struct *vma);
    void (*request)(struct vfio_device *vdev, unsigned int count);

    int  (*match)(struct vfio_device *vdev, char *buf);
    void (*dma_unmap)(struct vfio_device *vdev, u64 iova, u64 length);
    /* ... さらに pasid_attach_ioas/pasid_detach_ioas、match_token_uuid、
     * get_region_info_caps、device_feature がある */
};
```

`bind_iommufd`/`attach_ioas`の組（およびそのPASIDスコープ版）は、[仮想化側のVFIOページ](../virtualization/vfio.md#the-newer-model-device-cdev-iommufd)で扱うdevice cdev + IOMMUFDの所有権フックである。これらは、このページがそれ以外の部分で説明している従来のグループ/コンテナ経路ではなく、`/dev/vfio/devices/vfioN`を通じてデバイスが開かれた場合にのみ実行される。

**Intel GVT-g**（Intel内蔵GPUの仮想化、`drivers/gpu/drm/i915/gvt/`）と**nvidia vGPU**は、独自の`vfio_device_ops`を実装している。mdevデバイスはQEMUからは標準のVFIOデバイスに見えるので、QEMUの変更は不要である。QEMUは単に`/dev/vfio/<group>`を開いて通常どおり動作する。

### mdevの分離モデル {#mdev-isolation-model}

同じ物理ファンクションのmdevデバイスは、1つのIOMMUグループ（PFのグループ）を共有する。したがって1つのIOMMUドメインを共有する。vGPUインスタンス間の分離は、別々のIOMMUドメインによってではなく、物理デバイス自身のハードウェア（GPUのMMU、NICのキュー分離）によって強制される。これは完全なデバイスパススルーより弱く、セキュリティモデルは完全にPFドライバの仮想デバイス実装の正しさに依存する。

## VMパススルーの全体像: QEMU → KVMの流れ {#complete-vm-passthrough-qemu-kvm-flow}

```
QEMU process
  ├── opens /dev/vfio/vfio            (container)
  ├── opens /dev/vfio/12              (group)
  ├── VFIO_GROUP_SET_CONTAINER        (attach group to container)
  ├── VFIO_SET_IOMMU (TYPE1_IOMMU)   (activate IOMMU on container)
  ├── opens device fd (VFIO_GROUP_GET_DEVICE_FD)
  ├── VFIO_IOMMU_MAP_DMA              (pin guest RAM, map into IOMMU)
  │       │
  │       ▼ kernel: pin_user_pages() + iommu_map()
  │
  ├── VFIO_DEVICE_GET_REGION_INFO     (discover MMIO/config BAR regions)
  ├── mmap(device fd, BAR0_offset)    (map MMIO BAR directly into QEMU VA)
  │       │
  │       ▼ guest MMIO write → VMExit → KVM → QEMU → write to mmap'd BAR
  │         (or with assigned device: VMExit suppressed, write goes directly)
  │
  ├── VFIO_DEVICE_SET_IRQS            (set up MSI eventfds)
  └── KVM_IRQFD                       (bind eventfds to guest IRQ lines)

Runtime:
  Device DMA:    Device → IOMMU (IOVA → HPA) → host memory → guest sees it
  Device MSI:    Device → IOMMU interrupt remap → host IRQ → irqfd → vCPU
```

### VFIO_IOMMU_MAP_DMAの詳細 {#vfio_iommu_map_dma-in-detail}

QEMUはゲストの起動時に、ゲストのRAM全体をIOMMUドメインにマップする。4 GBのゲストの場合は次のようになる。

```c
struct vfio_iommu_type1_dma_map dma_map = {
    .argsz = sizeof(dma_map),
    .flags = VFIO_DMA_MAP_FLAG_READ | VFIO_DMA_MAP_FLAG_WRITE,
    .vaddr = (uint64_t)guest_ram_hva,  /* ゲスト RAM に対する QEMU の VA */
    .iova  = 0x0,                      /* ゲスト物理アドレスのベース */
    .size  = 0x100000000ULL,           /* 4 GB */
};
ioctl(container, VFIO_IOMMU_MAP_DMA, &dma_map);
```

カーネルはQEMUのVA範囲4 GB全体をたどってページをピン留めし、IOVA 0..4GB → ホスト物理ページをマッピングするIOMMUページテーブルエントリを作成する。これ以降、割り当てられたデバイスはゲスト物理アドレスに直接DMAでき、IOMMUがそれを正しいホストページに変換する。

## セキュリティモデル {#security-model}

IOMMUグループがセキュリティ境界である。同じグループのデバイスはIOMMUドメインを共有し、IOMMUによる強制なしに互いにDMAできる。VFIOは次のことを強制する。

1. **グループ全体の割り当て**: あるグループのデバイスAをVM1に、同じグループのデバイスBをVM2に割り当てることはできない。グループ全体を1つのVMに渡す（またはホストに残す）必要がある。
2. **グループ間のDMAの禁止**: IOMMUページテーブルにより、デバイスは`VFIO_IOMMU_MAP_DMA`で明示的にマップされたメモリにしか到達できない。乗っ取られたデバイスがホストカーネルのメモリや他のVMのメモリに到達することはできない。
3. **割り込みリマッピングが必須**: Intelでは、VT-dの割り込みリマッピングが有効になっていなければならない（`dmesg | grep "Enabled IRQ remapping"`で確認）。割り込みリマッピングがなければ、デバイスは任意の割り込みを注入でき、セキュリティモデルを迂回できてしまう。

```bash
# IOMMU が分離を強制していること（パススルーモードでないこと）を確認する
dmesg | grep -i "iommu"
# DMAR: IOMMU enabled  ← 変換が有効
# (iommu=pt ではないこと ← パススルーモード、分離なし)

# 割り込みリマッピングを確認する
dmesg | grep -i "remapping"
# DMAR-IR: Enabled IRQ remapping in x2apic mode

# パススルーの前にグループ内のデバイスを一覧表示する
ls /sys/kernel/iommu_groups/12/devices/
# 複数のデバイスがあれば、両方をまとめてパススルーしなければならない
```

## VFIOの観測 {#observing-vfio}

```bash
# VFIO のデバイスとグループ
ls /sys/bus/pci/drivers/vfio-pci/

# アクティブな VFIO のコンテナ/グループ（root 権限が必要）
ls /dev/vfio/

# パススルー中の DMAR フォールト（デバイスがマップされていないアドレスに DMA した）
dmesg | grep DMAR

# VFIO のトレーシング（汎用の x86/arm64）
echo 1 > /sys/kernel/tracing/events/vfio/enable
echo 1 > /sys/kernel/tracing/events/iommu/enable
# 注: events/vfio_ap/ の tracepoint は s390（IBM Z）専用 — AP
# （Adjunct Processor）暗号デバイス向け。x86 や arm64 では使わないこと。

# MSI eventfd: 割り込みの発生頻度を見る
# （/proc/interrupts または perf stat で）
grep vfio /proc/interrupts
```

## ソースファイル {#source-files}

| ファイル | 役割 |
|------|------|
| `drivers/vfio/vfio_main.c` | コンテナ/グループ/デバイスのライフサイクル、ioctl |
| `drivers/vfio/vfio_iommu_type1.c` | IOMMUバックエンド: pin_user_pages、iommu_map、DMAマップのツリー |
| `drivers/vfio/pci/vfio_pci_core.c` | PCIデバイスの操作: MMIOのmmap、コンフィギュレーション空間、割り込み |
| `drivers/vfio/pci/vfio_pci_intrs.c` | MSI/MSI-X/INTxのeventfd設定、irqfdとの統合 |
| `drivers/vfio/mdev/` | Mediated deviceのバスとsysfsインターフェース |
| `virt/kvm/eventfd.c` | `irqfd` — eventfd → vCPUへの割り込み注入 |
| `include/linux/vfio.h` | VFIOのすべての構造体とioctlの定義 |
| `include/uapi/linux/vfio.h` | ユーザー空間から見えるioctl API |

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [drivers/vfio/vfio_main.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/vfio/vfio_main.c) — `vfio_register_group_dev()`と、デバイス/グループ/コンテナのライフサイクルの中核
- [drivers/vfio/vfio.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/vfio/vfio.h) — `struct vfio_group`（VFIOコア専用であり、`include/linux/vfio.h`にはない）
- [drivers/vfio/container.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/vfio/container.c) — `struct vfio_container`と`VFIO_SET_IOMMU`/グループのアタッチ経路
- [drivers/vfio/vfio_iommu_type1.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/vfio/vfio_iommu_type1.c) — `vfio_dma_do_map()`、`struct vfio_iommu`、`struct vfio_dma`: type1 IOMMUバックエンド
- [include/linux/vfio.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/vfio.h) — `struct vfio_device`と`struct vfio_device_ops`。公開されたバスドライバAPI
- [drivers/vfio/pci/vfio_pci_intrs.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/vfio/pci/vfio_pci_intrs.c) — `struct vfio_pci_irq_ctx`: MSI/MSI-X/INTxのeventfd設定
- [drivers/vfio/mdev/mdev_core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/vfio/mdev/mdev_core.c) — `mdev_register_parent()`とmediated deviceバスのコア
- [virt/kvm/eventfd.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/virt/kvm/eventfd.c) — `irqfd`: バインドされたeventfdが発火するとworkqueueが`kvm_set_irq()`を呼ぶ

### manページ {#man-pages}

- [`ioctl(2)`](https://man7.org/linux/man-pages/man2/ioctl.2.html) — VFIOのコンテナ/グループ/デバイスの操作（`VFIO_SET_IOMMU`、`VFIO_IOMMU_MAP_DMA`、`VFIO_DEVICE_SET_IRQS`など）はすべて`/dev/vfio`のファイルディスクリプタに対するioctlである

### 関連ページ {#related-pages}

- [IOMMUアーキテクチャ](iommu-arch.md) — IOMMUドメイン、グループ、SR-IOV
- [IOVAアロケータ](iova-allocator.md) — `dma_map_*()`が使うカーネル内のIOVAアロケータ。ユーザー空間が`VFIO_IOMMU_MAP_DMA`でIOVAを直接指定するため、VFIOのtype1バックエンドはこれを迂回する
- [IOMMU事件簿](war-stories.md) — グループ分離がパススルーを阻んだ事例（実際のインシデント）
- [VFIO: Virtual Function I/Oとデバイスパススルー](../virtualization/vfio.md) — 同じサブシステムをセットアップ/運用の観点から扱う。QEMUパススルー、SR-IOV、mdevの手順解説
- [ユーザーページの取得（GUP）](../mm/gup.md) — `pin_user_pages_remote()`。VFIOのDMAマッピング経路の背後にあるページピン留めのプリミティブ

### LWNの記事 {#lwn-articles}

- [Safe device assignment with VFIO](https://lwn.net/Articles/474088/) — VFIOフレームワークとIOMMUグループに基づくデバイス割り当てに関するJonathan Corbetの最初の記事（2012年1月3日）

### 外部資料 {#external}

- [VFIO — "Virtual Function I/O"](https://docs.kernel.org/driver-api/vfio.html) — コンテナ/グループ/デバイスモデルとユーザー空間APIに関する公式カーネルドキュメント
- [VFIO Mediated devices](https://docs.kernel.org/driver-api/vfio-mediated-device.html) — mdevフレームワークに関する公式ドキュメント
