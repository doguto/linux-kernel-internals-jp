# VFIO: Virtual Function I/Oとデバイスパススルー {#vfio-virtual-function-io-and-device-passthrough}

> IOMMUグループ、DMAリマッピング、そしてユーザー空間/VMからのPCIデバイスへの直接アクセス

## VFIOとは {#what-is-vfio}

VFIO（Virtual Function I/O）は、カーネルドライバを迂回して、PCIデバイスをユーザー空間のプロセスやVMに直接公開する安全な仕組みを提供する。VFIOパススルーを使うゲストVMは、ネイティブに近い速度でデバイスにアクセスできる。

```
Without passthrough:
  Guest driver → virtio protocol → host vhost/QEMU → host kernel driver → hardware
  Every I/O: multiple context switches, data copies

With VFIO passthrough:
  Guest driver → VFIO → IOMMU DMA remapping → hardware
  Direct DMA: guest physical addresses remapped by IOMMU to real hardware addresses
```

**要件**:

- IOMMUハードウェア（Intel VT-dまたはAMD-Vi）
- デバイスが独自のIOMMUグループに属していること（またはグループの分離が満たされていること）
- ブートパラメータ`intel_iommu=on`または`amd_iommu=on`

**2世代のAPI**: 以下の「ユーザー空間のVFIO API」セクションまでの内容はすべて、当初のグループ/コンテナモデル（`/dev/vfio/vfio` + `/dev/vfio/<group_id>`）を説明している。これは現在もデフォルトで有効であり、大多数が使っている経路である。`CONFIG_VFIO_GROUP`自身のヘルプテキストも、これを「the traditional model ... used by the majority of userspace applications and drivers」（従来のモデルであり、大多数のユーザー空間アプリケーションとドライバが使用している）と呼んでいる。しかしカーネル自身のドキュメントは、これが置き換えられつつあるレガシーなインターフェイスであることを明言している。「Eventually the vfio_iommu_type1 driver, as well as the legacy vfio container and group model is intended to be deprecated」（最終的にはvfio_iommu_type1ドライバ、およびレガシーなvfioのコンテナとグループのモデルは非推奨とされる予定である）とあり、より新しいdevice cdev + IOMMUFDモデルへの移行が想定されている。これについては、以下の従来型の手順説明の後で扱う。

## IOMMUグループ {#iommu-groups}

IOMMUグループとは、DMAの安全性のために一緒に分離しなければならないデバイスの最小の集合である。同じPCIe階層にあり、互いにピアツーピアのDMAを行えるデバイスがグループを形成する。

```bash
# すべてのIOMMUグループを一覧表示:
find /sys/kernel/iommu_groups -type l | sort -V | head -20
# /sys/kernel/iommu_groups/0/devices/0000:00:00.0
# /sys/kernel/iommu_groups/1/devices/0000:00:01.0
# /sys/kernel/iommu_groups/2/devices/0000:00:02.0
# ...

# デバイスがどのグループに属するかを確認:
ls /sys/bus/pci/devices/0000:03:00.0/iommu_group
# → ../../../kernel/iommu_groups/15

# グループ内のすべてのデバイスを一覧表示:
ls /sys/kernel/iommu_groups/15/devices/
# 0000:03:00.0  0000:03:00.1  ← 例: 2つのファンクションを持つNIC

# パススルーする場合、グループ内のすべてのデバイスをパススルーする必要がある
# （またはvfio-pciのスタブドライバに任せる）
```

## VFIOのアーキテクチャ {#vfio-architecture}

```
Userspace (QEMU/application)
    ↓ open/ioctl
/dev/vfio/vfio           ← container (IOMMU domain)
/dev/vfio/<group_id>     ← IOMMU group
    ↓
VFIO kernel driver
    ↓
IOMMU driver (Intel VT-d / AMD-Vi)
    ↓
PCIe device BAR access (MMIO mapped to guest/userspace)
DMA: guest GPA → IOMMU → host HPA (remapped per container)
```

### 主要なオブジェクト {#key-objects}

```c
/* 1. コンテナ: IOMMUのアドレス空間（複数のグループで共有される） */
/*    fd = open("/dev/vfio/vfio", O_RDWR) */

/* 2. グループ: DMA分離の要件を共有するデバイスの集合 */
/*    fd = open("/dev/vfio/15", O_RDWR)  (グループ15) */

/* 3. デバイス: グループ内の1つのPCIファンクション */
/*    fd = ioctl(group_fd, VFIO_GROUP_GET_DEVICE_FD, "0000:03:00.0") */
```

## VFIOパススルーのセットアップ {#setting-up-vfio-passthrough}

```bash
# ステップ1: ブートパラメータでIOMMUを有効にする（/etc/default/grub）:
GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt"
# iommu=pt: パススルーモード（vfio以外のデバイスの性能が向上する）
update-grub && reboot

# ステップ2: vfio-pciモジュールをロードする:
modprobe vfio-pci

# ステップ3: デバイスを現在のドライバからアンバインドする:
echo 0000:03:00.0 > /sys/bus/pci/devices/0000:03:00.0/driver/unbind

# ステップ4: vfio-pciにバインドする:
echo "10de 1234" > /sys/bus/pci/drivers/vfio-pci/new_id  # vendor:device
# または:
echo 0000:03:00.0 > /sys/bus/pci/drivers/vfio-pci/bind

# ステップ5: 確認する:
ls /sys/bus/pci/devices/0000:03:00.0/driver
# → /sys/bus/pci/drivers/vfio-pci

# ステップ6: グループを確認する:
ls /sys/kernel/iommu_groups/15/devices/
# グループ内のすべてのデバイスがvfio-pciにバインドされている（またはドライバなし）必要がある
```

## QEMUでのデバイスパススルー {#qemu-device-passthrough}

```bash
# NVIDIA GPUをVMに渡す:
qemu-system-x86_64 \
    -enable-kvm \
    -m 8G \
    -cpu host \
    -device vfio-pci,host=03:00.0,multifunction=on \
    -device vfio-pci,host=03:00.1 \   # オーディオファンクション
    -drive if=virtio,file=vm.img \
    -net user,hostfwd=tcp::2222-:22

# NVMe SSDを渡す:
qemu-system-x86_64 \
    -enable-kvm \
    -device vfio-pci,host=02:00.0 \   # NVMeコントローラ
    ...
```

## ユーザー空間のVFIO API {#userspace-vfio-api}

（VMだけでなく）アプリケーションも、デバイスアクセスのためにVFIOを直接使える。

```c
#include <linux/vfio.h>

/* 1. コンテナを開く */
int container = open("/dev/vfio/vfio", O_RDWR);
ioctl(container, VFIO_GET_API_VERSION);  /* VFIO_API_VERSIONでなければならない */

/* 2. IOMMUグループを開く */
int group = open("/dev/vfio/15", O_RDWR);
struct vfio_group_status status;
ioctl(group, VFIO_GROUP_GET_STATUS, &status);
assert(status.flags & VFIO_GROUP_FLAGS_VIABLE);

/* 3. グループをコンテナに追加する */
ioctl(group, VFIO_GROUP_SET_CONTAINER, &container);

/* 4. IOMMUタイプを設定する（DMAマッピングのドメインを作成する） */
ioctl(container, VFIO_SET_IOMMU, VFIO_TYPE1_IOMMU);

/* 5. DMAバッファをIOMMUにマップする */
struct vfio_iommu_type1_dma_map dma_map = {
    .argsz = sizeof(dma_map),
    .flags = VFIO_DMA_MAP_FLAG_READ | VFIO_DMA_MAP_FLAG_WRITE,
    .vaddr = (uintptr_t)buf,    /* ホストの仮想アドレス */
    .iova  = 0x10000000,        /* IOVA（デバイスから見えるアドレス） */
    .size  = BUF_SIZE,
};
ioctl(container, VFIO_IOMMU_MAP_DMA, &dma_map);
/* これでデバイスはIOVA 0x10000000との間でDMAできる */

/* 6. デバイスのfdを取得する */
int device = ioctl(group, VFIO_GROUP_GET_DEVICE_FD, "0000:03:00.0");

/* 7. デバイス情報（BAR、割り込み）を取得する */
struct vfio_device_info device_info = { .argsz = sizeof(device_info) };
ioctl(device, VFIO_DEVICE_GET_INFO, &device_info);
/* device_info.num_regions: BAR/ケーパビリティの数 */
/* device_info.num_irqs:    割り込みタイプの数 */

/* 8. BAR 0（MMIOレジスタ）をプロセスのアドレス空間にマップする */
struct vfio_region_info reg = {
    .argsz = sizeof(reg),
    .index = VFIO_PCI_BAR0_REGION_INDEX,
};
ioctl(device, VFIO_DEVICE_GET_REGION_INFO, &reg);
void *mmio = mmap(NULL, reg.size, PROT_READ|PROT_WRITE,
                   MAP_SHARED, device, reg.offset);
/* これで: カーネルドライバと同じようにmmioを直接読み書きできる */
```

## 新しいモデル: device cdev + IOMMUFD {#the-newer-model-device-cdev-iommufd}

上記の従来型モデルは、DMAの分離をIOMMU*グループ*に結び付けている。デバイスのfdを得るには、まずそのグループを開かなければならない。新しいモデルはこの要件を取り除く。`CONFIG_VFIO_DEVICE_CDEV=y`であれば、デバイスのfdを直接取得できる。

```c
/* 従来型: グループを経由する */
device = ioctl(group_fd, VFIO_GROUP_GET_DEVICE_FD, "0000:03:00.0");

/* 新しい方式: デバイス自身のキャラクタデバイスを直接開く */
device = open("/dev/vfio/devices/vfio0", O_RDWR);
```

device cdevはIOMMUFDとの組み合わせでのみ動作し、レガシーなコンテナとは組み合わせられない。ユーザー空間は`VFIO_GROUP_SET_CONTAINER`/`VFIO_SET_IOMMU`の代わりに`/dev/iommu`を開き、デバイスを使う前に`VFIO_DEVICE_BIND_IOMMUFD`でデバイスのDMAの所有権を取得する。IOMMUFDの互換モードは、*既存の*コンテナのioctlを受け付けることもできる（`CONFIG_IOMMUFD_VFIO_CONTAINER`を設定するか、`/dev/vfio/vfio`を`/dev/iommu`へのシンボリックリンクにする）。ただし本稿執筆時点では、この互換経路は`VFIO_TYPE1v2_IOMMU`に比べて機能が揃っておらず、カーネルのドキュメントは特にMMIOのDMAマッピングを欠けている機能として挙げている。cdevによるアクセスの下でも、グループのセマンティクスは依然として適用される。同じIOMMUグループのデバイスを異なる`iommufd_ctx`インスタンスにバインドしたり、カーネルとVFIOの所有に分けたりすることはできない。

`iommufd`は、レガシーな`vfio_iommu_type1`ドライバが一度もサポートしなかった機能への道も開く。たとえば、ネストされたIOMMUページテーブル（ネストされたEPTのIOMMUレベルにおける類似物）や、PASIDベースのサブデバイス分離である。

## SR-IOV: 仮想ファンクション {#sr-iov-virtual-functions}

SR-IOV（Single Root I/O Virtualization）は、1つの物理デバイスを複数の仮想ファンクション（VF）として見せられるようにする。

```bash
# SR-IOVを有効にする（1つのPFから4つのVFを作成する）:
echo 4 > /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs

# VFは新しいPCIデバイスとして現れる:
lspci | grep "Virtual Function"
# 03:00.1  VF 1
# 03:00.2  VF 2
# 03:00.3  VF 3
# 03:00.4  VF 4

# 各VFをvfio-pciにバインドし、別々のVMに渡す:
echo 0000:03:00.1 > /sys/bus/pci/drivers/vfio-pci/bind

# ネットワークのSR-IOV: VFは独自のMAC/VLANと専用のキューリングを持つ
ip link set eth0 vf 0 mac 52:54:00:01:02:03 vlan 100 rate 1000
```

## Mediated device（mdev） {#mediated-devices-mdev}

一部のデバイスは、mediated passthroughをサポートする。ホストのドライバがデバイスを分割し、仮想的なサブデバイスを提示する。

```bash
# Intel GVT-g: 1つのGPUを複数のVMで共有する
ls /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/
# i915-GVTg_V5_4  ← 1つのGPUを4つのGPU VMで共有

# mediated deviceのインスタンスを作成する:
uuidgen  # 例: 83b8f4f2-509f-382f-3c1e-e6bfe0fa1001
echo "83b8f4f2-509f-382f-3c1e-e6bfe0fa1001" > \
    /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/i915-GVTg_V5_4/create

# QEMUに割り当てる:
qemu-system-x86_64 \
    -device vfio-pci,sysfsdev=/sys/bus/mdev/devices/83b8f4f2-...
```

## VFIOのカーネル内部 {#vfio-kernel-internals}

```c
/* 簡略化したもの — 実際の定義はdrivers/vfio/vfio.h、
 * container.c、group.c、pci/vfio_pci_core.cに分かれている */

/* コンテナはIOMMUドメインを管理する: */
struct vfio_container {
    struct kref         kref;
    struct list_head    group_list;
    struct rw_semaphore group_lock;
    struct vfio_iommu_driver *iommu_driver;
    void               *iommu_data;   /* iommu_domain */
    bool                noiommu;
};

/* 各グループがデバイスをコンテナに接続する: */
struct vfio_group {
    struct iommu_group  *iommu_group;
    struct vfio_container *container;
    struct list_head    device_list;
    struct cdev         cdev;         /* /dev/vfio/<id> */
};

/* VFIOのPCIデバイス - ユーザー空間からのアクセスのためにPCIデバイスをラップする: */
struct vfio_pci_core_device {
    struct vfio_device  vdev;
    struct pci_dev     *pdev;
    void __iomem       *barmap[PCI_STD_NUM_BARS]; /* BARのマッピング */
    struct xarray       ctx;          /* IRQのeventfd。IRQベクタでインデックスされる */
    /* ... */
};
```

### DMAマッピングの経路 {#dma-mapping-path}

```c
/* ユーザー空間がVFIO_IOMMU_MAP_DMAを呼ぶと、vfio_iommu_type1.cの
 * vfio_dma_do_map()がそれを処理する（簡略化したもの）: */
static int vfio_dma_do_map(struct vfio_iommu *iommu,
                            struct vfio_iommu_type1_dma_map *map)
{
    /* 1. ユーザー空間のページをピン留めする */
    vfio_pin_pages_remote(dma, vaddr, npage, &pfn, limit, batch);

    /* 2. IOMMUマッピングを作成する: IOVA → 物理ページ。このコンテナを
     *    支えるすべてのIOMMUドメインに対して行う */
    iommu_map(domain->domain, iova, phys, size,
              prot | IOMMU_CACHE, GFP_KERNEL_ACCOUNT);

    /* これでIOVAへのデバイスのDMAはIOMMUを経由して正しい物理ページに届く */
    /* IOMMUがなければ: デバイスはどこにでもDMAできてしまう → セキュリティホール */
}
```

## 可観測性 {#observability}

```bash
# IOMMUが有効であることを確認:
dmesg | grep -i iommu | head -5
# [    0.234] DMAR: IOMMU enabled

# VFIOグループを確認:
ls /dev/vfio/
# vfio  15  23  ...

# IOMMUのドメイン/デバイスのマッピング（Intel VT-d）:
cat /sys/kernel/debug/iommu/intel/dmar_translation_struct 2>/dev/null | head -20

# 性能: Intel IOMMUはDMAリマッピングユニットごとに実際のPMUを公開する
# （drivers/iommu/intel/perfmon.c）。名前はdmar0、dmar1、... — 直接の
# "miss"カウンタはないが、ミス率 = iotlb_lookup - iotlb_hit:
perf stat -e dmar0/iotlb_lookup/,dmar0/iotlb_hit/ -a sleep 5
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [drivers/vfio/vfio_main.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/vfio/vfio_main.c) — VFIOのコア: デバイスの登録（`vfio_register_group_dev()`）、ioctl/mmap/read/writeのファイル操作、サブシステムのモジュール初期化（グループとコンテナのioctlのディスパッチは`group.c`/`container.c`にある）
- [drivers/vfio/vfio.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/vfio/vfio.h) — `struct vfio_group`の定義（`struct vfio_container`はここでは前方宣言のみ）
- [drivers/vfio/container.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/vfio/container.c) — `struct vfio_container`の定義
- [drivers/vfio/vfio_iommu_type1.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/vfio/vfio_iommu_type1.c) — Type1 IOMMUバックエンド: `vfio_dma_do_map()`（`VFIO_IOMMU_MAP_DMA`を処理する）、`vfio_pin_pages_remote()`、`iommu_map()`
- [drivers/vfio/pci/vfio_pci_core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/vfio/pci/vfio_pci_core.c) — vfio-pciのコア: BAR/MMIOのマッピング、コンフィギュレーション空間、IRQのeventfd（[include/linux/vfio_pci_core.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/vfio_pci_core.h)の`struct vfio_pci_core_device`を支える）
- [drivers/vfio/mdev/mdev_core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/vfio/mdev/mdev_core.c) — Intel GVT-gのようなGPU共有ドライバが使うmediated device（mdev）フレームワーク
- [drivers/iommu/iommu.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu/iommu.c) — `iommu_group_alloc()`とIOMMUグループの管理
- [drivers/iommu/intel/perfmon.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu/intel/perfmon.c) — Intel IOMMU PerfMon: DMARユニットごと（`dmar0`、`dmar1`、...）のperf PMUと、その`iotlb_lookup`/`iotlb_hit`イベント
- [drivers/vfio/vfio_main.c: `VFIO_DEVICE_BIND_IOMMUFD`の処理](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/vfio/vfio_main.c) — グループ/コンテナによる所有に代わるdevice cdevのセキュリティモデル

### 関連ページ {#related-pages}

- [KVMアーキテクチャ](kvm-arch.md) — VM exitとVMCS
- [メモリ仮想化](kvm-memory.md) — EPTとシャドウページング
- [IOMMUアーキテクチャ](../iommu/iommu-arch.md) — IOMMUハードウェアとDMA API
- [PCIドライバ](../drivers/pci-driver.md) — VFIO以前にPCIドライバがどう動作するか

### LWNの記事 {#lwn-articles}

- [LWN: Safe device assignment with VFIO](https://lwn.net/Articles/474088/) — Alex WilliamsonのVFIOフレームワークと、IOMMUグループに基づくデバイス割り当てについてのJonathan Corbetによる記事（2012年1月3日）

### 外部資料 {#external}

- [VFIO — "Virtual Function I/O"](https://docs.kernel.org/driver-api/vfio.html) — VFIOフレームワークの公式カーネルドキュメント: 従来のコンテナ/グループ/デバイスモデル、新しいdevice cdevモデル、IOMMUFDとの関係。上で引用した非推奨の記述も含む
- [VFIO Mediated devices](https://docs.kernel.org/driver-api/vfio-mediated-device.html) — 本ページのmediated deviceのセクションで扱ったmdevフレームワークの公式ドキュメント
