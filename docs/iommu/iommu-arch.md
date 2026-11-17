# IOMMUアーキテクチャ {#iommu-architecture}

> Intel VT-d、AMD-Vi、iommu_domain、デバイスパススルー

## ハードウェア実装 {#hardware-implementations}

### Intel VT-d（Virtualization Technology for Directed I/O） {#intel-vt-d-virtualization-technology-for-directed-io}

Intel VT-dは、ルートコンプレックス上にリマッピング用のハードウェアユニット（DMARユニット）を配置する。各DMARユニットはPCIeバスの1セグメントを管理する。

```
CPU ──► System Agent ──► DMAR Unit ──► PCIe Root Port ──► Device
                              │
                         Page Tables
                         (IOVA → HPA)
                              │
                         Access Control
                         (read/write/execute bits)
```

DMARユニットは2レベルの構造を使う。
1. **ルートテーブル**（バス番号でインデックス付け）
2. **コンテキストテーブル**（デバイス/ファンクションでインデックス付け）
3. 各コンテキストエントリはそのデバイスの**IOMMUページテーブル**を指す

```c
/* drivers/iommu/intel/iommu.h — 構造体の定義はここにあり、iommu.c ではない */
struct intel_iommu {
    void __iomem    *reg;        /* MMIO レジスタ */
    u64              cap;        /* ケーパビリティレジスタ */
    u64              ecap;       /* 拡張ケーパビリティ */
    /* ... */
    struct root_entry *root_entry; /* ルートテーブル（バスでインデックス付け） */
    int              seq_id;
    struct iommu_device iommu;   /* "iommu_dev" ではない */
};
```

### AMD-Vi（AMD I/O Virtualization） {#amd-vi-amd-io-virtualization}

AMDは、システム全体で単一のDevice Table（16ビットのデバイスID = bus+dev+fnでインデックス付け）を使う。各エントリはそのデバイスのページテーブルと制御フラグを指す。

```c
/* drivers/iommu/amd/amd_iommu_types.h — 実際の構造体はソース内にビットフィールドの
 * コメントがない素の 256 ビット union である。以下のビットの意味は説明用 */
struct dev_table_entry {
    union {
        u64  data[4];
        u128 data128[2];
    };
};
/* AMD-Vi ハードウェアのビットレイアウト（AMD I/O Virtualization 仕様による。
 * このヘッダ内には記述されていない）:
 *   [0]    V:  有効
 *   [1]    TV: 変換有効
 *   [11:9] Mode: ページテーブルのレベル数
 *   [51:12] PTP: ページテーブルポインタ
 *   （割り込みリマッピングのフィールドは 256 ビットエントリの data[2] にある） */
```

## IOMMUサブシステム {#the-iommu-subsystem}

カーネルは両方の実装を、`drivers/iommu/`にある共通APIの背後に抽象化している。

### struct iommu_domain {#struct-iommu_domain}

`iommu_domain`は分離されたアドレス空間を表す。1つ以上のデバイスがドメインにアタッチされ、それらは同じIOVA→PA変換を共有する。

```c
/* include/linux/iommu.h */
struct iommu_domain {
    unsigned            type;      /* IOMMU_DOMAIN_UNMANAGED / BLOCKED / IDENTITY / DMA */
    const struct iommu_domain_ops *ops;
    unsigned long       pgsize_bitmap; /* サポートするページサイズ */
    struct iommu_domain_geometry geometry;  /* IOVA 範囲の制約 */
    struct iommu_dma_cookie *iova_cookie;   /* IOVA アロケータの状態 */
    /* ... */
};

struct iommu_domain_ops {
    /* `old` は置き換えられるドメイン。独立した detach_dev コールバックは
     * 存在しない -- デタッチは専用のコールバックではなく、別のドメイン
     * （例: ブロッキングドメイン）をアタッチすることで行われる。 */
    int  (*attach_dev)(struct iommu_domain *domain, struct device *dev,
                       struct iommu_domain *old);

    int  (*map_pages)(struct iommu_domain *domain, unsigned long iova,
                      phys_addr_t paddr, size_t pgsize, size_t pgcount,
                      int prot, gfp_t gfp, size_t *mapped);
    size_t (*unmap_pages)(struct iommu_domain *domain, unsigned long iova,
                           size_t pgsize, size_t pgcount,
                           struct iommu_iotlb_gather *iotlb_gather);

    phys_addr_t (*iova_to_phys)(struct iommu_domain *domain, dma_addr_t iova);
    /* ... */
};
```

### ドメインの種類 {#domain-types}

```c
/* IOMMU_DOMAIN_BLOCKED: すべてのアクセスを拒否（デフォルト） */
/* IOMMU_DOMAIN_IDENTITY: IOVA == PA（パススルー） */
/* IOMMU_DOMAIN_UNMANAGED: ドライバがマッピングを管理 */
/* IOMMU_DOMAIN_DMA: カーネルの DMA 層がマッピングを管理 */
/* IOMMU_DOMAIN_DMA_FQ: TLB 無効化を遅延させるフラッシュキュー付きの DMA */
```

通常のDMAでは`IOMMU_DOMAIN_DMA`を使う。カーネルのDMA APIがIOVAの割り当てとマッピングを自動的に管理する。

## IOMMUグループ {#iommu-groups}

IOMMUグループとは、IOMMUが互いに分離できるデバイスの最小集合である。同じグループのデバイスはドメインを共有しなければならず、それぞれに独立したアドレス空間を与えることはできない。

なぜグループが必要なのか。PCIeのピアツーピアトランザクションがあるからである。PCIeブリッジは、その配下のデバイス同士がIOMMUを迂回して互いにDMAすることを許す場合がある。カーネルはこうしたデバイスをグループにまとめ、見かけだけの分離を防ぐ。

```bash
# 典型的なシステム: ほとんどのデバイスは専用のグループを持つ
# PCIe ルートポートとその子はグループ化されることがある
ls /sys/kernel/iommu_groups/
# 0  1  2  3  ... 25  ...

# グループ 0: Intel のオーディオデバイス
ls /sys/kernel/iommu_groups/0/devices/
# 0000:00:1f.3

# グループ 12: ディスクリート GPU（オーディオファンクションを含むことがある）
ls /sys/kernel/iommu_groups/12/devices/
# 0000:01:00.0   (GPU)
# 0000:01:00.1   (GPU HDMI audio)
```

## VFIO: VMへのデバイスパススルー {#vfio-device-passthrough-to-vms}

VFIO（Virtual Function I/O）はIOMMUを使って、物理デバイスをVM（KVM）やユーザー空間ドライバに安全に渡す。

```
VM guest
  │  writes to virtual device MMIO
  ▼
KVM VFIO handler
  │
  ▼
VFIO container (an IOMMU domain)
  │  guest physical address → IOVA mapping
  ▼
IOMMU page tables
  │  IOVA → host physical address
  ▼
physical device
```

### VFIOの使い方 {#vfio-usage}

```c
/* ユーザー空間の VFIO API */
#include <linux/vfio.h>

/* 1. VFIO コンテナを開く */
int container = open("/dev/vfio/vfio", O_RDWR);

/* 2. IOMMU グループを開く */
int group = open("/dev/vfio/12", O_RDWR);  /* グループ 12 */
ioctl(group, VFIO_GROUP_SET_CONTAINER, &container);

/* 3. コンテナで IOMMU を有効にする */
ioctl(container, VFIO_SET_IOMMU, VFIO_TYPE1_IOMMU);

/* 4. デバイスを開く */
int device = ioctl(group, VFIO_GROUP_GET_DEVICE_FD, "0000:01:00.0");

/* 5. ゲストメモリを IOMMU ドメインにマップする */
struct vfio_iommu_type1_dma_map dma_map = {
    .argsz = sizeof(dma_map),
    .flags = VFIO_DMA_MAP_FLAG_READ | VFIO_DMA_MAP_FLAG_WRITE,
    .vaddr = (uint64_t)guest_ram,     /* HVA */
    .iova  = 0x0,                     /* ゲスト物理アドレスのベース */
    .size  = guest_ram_size,
};
ioctl(container, VFIO_IOMMU_MAP_DMA, &dma_map);

/* これでデバイスは IOVA 0..guest_ram_size でゲストメモリに DMA できる */
/* IOMMU はデバイスが他のホストメモリにアクセスするのを防ぐ */
```

## SR-IOV: 仮想ファンクション {#sr-iov-virtual-functions}

SR-IOV（Single Root I/O Virtualization）は、1つの物理ファンクション（PF）から複数のPCIe**仮想ファンクション**（VF）を作り出す。各VFは独自のPCIeコンフィギュレーション空間を持ち、独立してVMに割り当てられる。

```bash
# NIC で VF を有効にする（例: 4 つの VF を作成）
echo 4 | sudo tee /sys/bus/pci/devices/0000:01:00.0/sriov_numvfs

# 各 VF は独自の PCI アドレスを持つ
lspci | grep "Virtual Function"
# 0000:01:10.0 Ethernet controller: Intel ... Virtual Function
# 0000:01:10.2 ...

# VFIO 経由で VF を VM に割り当てる
echo "0000:01:10.0" > /sys/bus/pci/devices/0000:01:10.0/driver/unbind
echo "8086 154c" > /sys/bus/pci/drivers/vfio-pci/new_id
```

## IOMMU TLBの管理 {#iommu-tlb-management}

IOMMUは、IOVA→PA変換をキャッシュする独自のTLB（IOTLB）を持つ。アンマップの後、カーネルはIOTLBをフラッシュしなければならない。

```c
/* include/linux/iommu.h — これらはヘッダ内の static inline ヘルパーであり、
 * drivers/iommu/iommu.c で定義された関数ではない */
static inline void iommu_iotlb_gather_add_page(struct iommu_domain *domain,
                                                struct iommu_iotlb_gather *gather,
                                                unsigned long iova, size_t size)
{
    /* 新しい範囲が集めた範囲と離れている、またはページサイズが異なる場合は、
     * 連続していない範囲を黙って 1 つの無効化にまとめるのではなく、
     * 先に集めた分をフラッシュする */
    if ((gather->pgsize && gather->pgsize != size) ||
        iommu_iotlb_gather_is_disjoint(gather, iova, size))
        iommu_iotlb_sync(domain, gather);

    gather->pgsize = size;
    /* （ここで互換になった）範囲を蓄積する */
    if (gather->start > iova)
        gather->start = iova;
    if (gather->end < iova + size - 1)
        gather->end = iova + size - 1;
}

/* 蓄積した範囲をフラッシュする（効率のためにバッチ化） */
static inline void iommu_iotlb_sync(struct iommu_domain *domain,
                                     struct iommu_iotlb_gather *iotlb_gather)
{
    /* 実際に何かが集められた場合にのみドライバを呼び出す --
     * 呼び出しのたびに無駄な sync が起きるのを避ける */
    if (domain->ops->iotlb_sync && iotlb_gather->start < iotlb_gather->end)
        domain->ops->iotlb_sync(domain, iotlb_gather);
    iommu_iotlb_gather_init(iotlb_gather);
}
```

遅延フラッシュ（フラッシュキュー）は、アンマップ操作をバッチ化することでIOTLB無効化のコストを償却する。

## IOMMUの観測 {#observing-the-iommu}

```bash
# DMAR フォールト（デバイスが IOMMU のポリシーに違反した）
dmesg | grep DMAR
# DMAR: [DMA Write] Request device [01:00.0] fault addr 7f000000
# DMAR: [fault reason 02] Present bit in context entry is clear

# IOMMU の統計情報
cat /sys/kernel/debug/iommu/intel/iommu_perf_stats

# AMD-Vi の統計情報
cat /sys/kernel/debug/iommu/amd/amd_iommu_stats

# IOVA アロケータの状態
cat /sys/kernel/debug/iommu/iova

# IOMMU の分離をテストする（VFIO テスト）
modprobe vfio-pci
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/iommu.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/iommu.h) — `struct iommu_domain`、`struct iommu_domain_ops`、`IOMMU_DOMAIN_*`型定数、`iommu_iotlb_gather_add_page()`/`iommu_iotlb_sync()`インラインヘルパー
- [drivers/iommu/iommu.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu/iommu.c) — `iommu_group_alloc()`をはじめとするIOMMUグループ/ドメイン管理
- [drivers/iommu/intel/iommu.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu/intel/iommu.h) — `struct intel_iommu`: リマッピングユニットごとのVT-dの状態（ルートテーブル、ケーパビリティレジスタ、キュー型無効化）
- [drivers/iommu/amd/amd_iommu_types.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu/amd/amd_iommu_types.h) — `struct dev_table_entry`: デバイスIDごとのAMD-Viテーブルエントリ
- [Documentation/ABI/testing/sysfs-kernel-iommu_groups](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/ABI/testing/sysfs-kernel-iommu_groups) — このページのグループの例で参照している`/sys/kernel/iommu_groups/`のABI

### 関連ページ {#related-pages}

- [DMA API](dma-api.md) — ここで説明したドメイン/グループモデルの上に構築される、カーネルドライバのDMAプログラミング
- [IOVAアロケータ](iova-allocator.md) — `alloc_iova()`/`free_iova()`が`IOMMU_DOMAIN_DMA`ドメイン内のアドレス空間をどう管理するか
- [共有仮想アドレッシング](sva.md) — プロセスのページテーブルをデバイスと共有するPASIDベースの変換。同じ`iommu_domain`基盤の上に構築されている
- [VFIOの内部](vfio-internals.md) — ここで示したVFIO APIの背後にある、コンテナ/グループ/デバイス階層とtype1 IOMMUバックエンド
- [IOMMU事件簿](war-stories.md) — このページのツールで診断した、実際のDMARフォールトストームやグループ分離のインシデント
- [VFIO: Virtual Function I/Oとデバイスパススルー](../virtualization/vfio.md) — このページのVFIOセクションが要約しているVMパススルーの解説
- [メモリ仮想化](../virtualization/kvm-memory.md) — EPT/NPT。IOMMUによるデバイス側の変換に対応するCPU側の仕組み
- [メモリ管理: DMA](../mm/dma.md) — DMAゾーン、SWIOTLB、デバイスのコヒーレンシ
- [メモリ管理: デバイスのコヒーレンシ](../mm/device-coherency.md) — DMA可能なデバイスのキャッシュコヒーレンシのルール

### LWNの記事 {#lwn-articles}

- [LWN: Safe device assignment with VFIO](https://lwn.net/Articles/474088/) — Alex WilliamsonのVFIOフレームワークに関するJonathan Corbetの記事。"any device can access any memory regions made available to any other devices in the same group"（同じグループ内の任意のデバイスは、他のデバイスに提供された任意のメモリ領域にアクセスできる）理由と、グループがパススルーにおける所有の単位となった経緯を説明している（2012年1月3日）

### 外部資料 {#external}

- [x86 IOMMU Support](https://docs.kernel.org/arch/x86/iommu.html) — DMAR/IVRS ACPIテーブル、`intel_iommu=igfx_off`/`iommu=pt`ブートオプション、両ベンダーのフォールト報告形式に関する公式カーネルドキュメント
