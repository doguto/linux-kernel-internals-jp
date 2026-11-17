# DMA API {#dma-api}

> メモリアクセスのためのデバイスのプログラミング: コヒーレントDMAとストリーミングDMA

## なぜDMA APIか {#why-a-dma-api}

DMAのプログラミングはアーキテクチャに依存する。

- IOMMUのあるx86: IOVAを割り当て、デバイスにIOVAを設定する
- IOMMUのないx86: デバイスは物理アドレスを直接使う
- 非コヒーレントキャッシュのARM: DMAの前後でキャッシュをフラッシュ/無効化しなければならない
- 64ビットシステム上の32ビットDMA対応デバイス: アドレスは4GB未満でなければならない

DMA APIはこれらすべてを抽象化する。デバイスドライバは、アーキテクチャ、IOMMUの有無、デバイスのアドレッシング制約にかかわらず、1つのAPIを使う。

## デバイスのDMA設定 {#setting-up-a-device-for-dma}

DMAを使う前に、ドライバはデバイスのアドレッシング能力を宣言する。

```c
/* DMA マスクを設定する: デバイスは 64 ビットアドレスを扱える */
if (dma_set_mask_and_coherent(dev, DMA_BIT_MASK(64))) {
    /* 64 ビットがサポートされなければ 32 ビットにフォールバックする */
    if (dma_set_mask_and_coherent(dev, DMA_BIT_MASK(32))) {
        dev_err(dev, "No suitable DMA mask\n");
        return -ENODEV;
    }
}
```

`DMA_BIT_MASK(n)`は`n`ビットのマスクを作る。32ビットのマスクを設定すると、デバイスは4GBより上にDMAできないことになり、カーネルはDMAバッファを4GB未満に割り当てるよう保証しなければならない。

## コヒーレントDMA {#coherent-dma}

コヒーレント（「consistent」とも呼ぶ）DMAは、明示的なキャッシュ管理なしにCPUからもデバイスからもアクセスできるメモリを割り当てる。CPUとデバイスは常に同じデータを見る。

```c
/* コヒーレント DMA バッファを割り当てる */
dma_addr_t dma_handle;
void *cpu_addr = dma_alloc_coherent(dev,
                                     size,         /* バイト数 */
                                     &dma_handle,  /* デバイスから見えるアドレス */
                                     GFP_KERNEL);
if (!cpu_addr)
    return -ENOMEM;

/* cpu_addr: CPU 側の仮想アドレス */
/* dma_handle: デバイスのレジスタに設定するアドレス */
device_set_dma_addr(dev, dma_handle);

/* バッファを使う（キャッシュ管理は不要） */
memset(cpu_addr, 0, size);
/* デバイスはすぐにこれを読める */

/* 使い終わったら解放する */
dma_free_coherent(dev, size, cpu_addr, dma_handle);
```

**内部での動作:**

- キャッシュコヒーレントなバスを持つx86: `dma_alloc_coherent` → `alloc_pages` + IOMMUマッピング
- 非コヒーレントなARM: `dma_alloc_coherent` → `alloc_pages` + キャッシュ無効としてマーク（`pgprot_noncached`）
- IOMMUなし: `dma_alloc_coherent` → DMAゾーンに制限された`alloc_pages`（32ビットマスクなら4GB未満）

## ストリーミングDMA（map/unmap） {#streaming-dma-mapunmap}

ストリーミングDMAは、1回だけ転送される既存のバッファ向けである。CPUがデータを書き込み、デバイスに渡し、その後取り戻す。

### 単一バッファ {#single-buffer}

```c
/* DMA の方向: */
/* DMA_TO_DEVICE:     CPU → デバイス（書き込み DMA） */
/* DMA_FROM_DEVICE:   デバイス → CPU（読み出し DMA）  */
/* DMA_BIDIRECTIONAL: 双方向 */

/* 転送前にマップする */
dma_addr_t dma_handle = dma_map_single(dev,
                                        cpu_addr,      /* カーネル仮想アドレス */
                                        size,
                                        DMA_TO_DEVICE);
if (dma_mapping_error(dev, dma_handle)) {
    dev_err(dev, "DMA mapping failed\n");
    return -ENOMEM;
}

/* dma_handle をデバイスに設定する */
device_write(dev, dma_handle, size);

/* DMA の完了を待つ（デバイスは割り込みで通知する） */
wait_for_completion(&transfer_done);

/* 転送後にアンマップする */
dma_unmap_single(dev, dma_handle, size, DMA_TO_DEVICE);

/* これで CPU は再びバッファを読み書きできる */
```

**map/unmap時に起きること:**

- IOMMUあり: IOVAを割り当て、IOMMUマッピングを作成する
- IOMMUなし、コヒーレントキャッシュ: マッピングは何もしない（物理アドレス == デバイスアドレス）
- IOMMUなし、非コヒーレントキャッシュ: `map`はCPUキャッシュをフラッシュし、`unmap`はCPUキャッシュを無効化する

### スキャッタギャザー {#scatter-gather}

実際のI/Oでは、不連続なメモリ（例: ページ境界に揃ったskbフラグメント、bioのページ）を扱うことが多い。

```c
#include <linux/dma-mapping.h>
#include <linux/scatterlist.h>

struct scatterlist sg[4];
sg_init_table(sg, 4);
sg_set_page(&sg[0], page0, PAGE_SIZE, 0);
sg_set_page(&sg[1], page1, PAGE_SIZE, 0);
sg_set_page(&sg[2], page2, 512,       0);

/* 全エントリを一度にマップする */
int nents = dma_map_sg(dev, sg, 3, DMA_TO_DEVICE);
if (!nents) {
    /* マッピング失敗 */
    return -ENOMEM;
}

/* デバイスを設定する: マップ済みエントリを走査する */
for_each_sg(sg, s, nents, i) {
    dma_addr_t addr = sg_dma_address(s);
    unsigned int len = sg_dma_len(s);
    device_add_descriptor(dev, addr, len);
}

/* 完了を待ってからアンマップする */
dma_unmap_sg(dev, sg, 3, DMA_TO_DEVICE);
```

IOMMUがあれば、`dma_map_sg`は隣接するページを1つのIOVA範囲にまとめられる。物理メモリが断片化していても、デバイスからは1つの連続したバッファに見える。

## IOVAの割り当て {#iova-allocation}

IOMMUがある場合、ストリーミングDMAのマップのたびにIOVAの割り当てが必要になる。

```c
/* drivers/iommu/dma-iommu.c — 現在のシグネチャは struct page + offset の組ではなく
 * phys_addr_t を直接受け取る（古いカーネルは page+offset 形式だった） */
dma_addr_t iommu_dma_map_phys(struct device *dev, phys_addr_t phys, size_t size,
                               enum dma_data_direction dir, unsigned long attrs)
{
    bool coherent = dev_is_dma_coherent(dev);
    int prot = dma_info_to_prot(dir, coherent, attrs);
    struct iommu_domain *domain = iommu_get_dma_domain(dev);
    struct iommu_dma_cookie *cookie = domain->iova_cookie;
    struct iova_domain *iovad = &cookie->iovad;
    dma_addr_t iova, dma_mask = dma_get_mask(dev);

    /* バッファにバウンスが必要な場合（デバイスにとってアラインされていない）、
     * IOVA 割り当ての前に swiotlb がここで処理する */
    if (dev_use_swiotlb(dev, size, dir) && iova_unaligned(iovad, phys, size)) {
        phys = iommu_dma_map_swiotlb(dev, phys, size, dir, attrs);
        if (phys == (phys_addr_t)DMA_MAPPING_ERROR)
            return DMA_MAPPING_ERROR;
    }

    /* IOVA（I/O 仮想アドレス範囲）を割り当て、phys にマップする */
    iova = iommu_dma_alloc_iova(domain, size, dma_mask, dev);
    if (!iova)
        return DMA_MAPPING_ERROR;
    if (iommu_map(domain, iova, phys, size, prot, GFP_ATOMIC)) {
        iommu_dma_free_iova(domain, iova, size, NULL);
        return DMA_MAPPING_ERROR;
    }

    return iova;
}
```

IOVAアロケータは、空き範囲を管理する赤黒木と、同じサイズの割り当て向けのキャッシュ付きアロケータを使う。

## swiotlb: バウンスバッファ {#swiotlb-bounce-buffers}

デバイスが特定のメモリ（例: 32ビットデバイスにとっての4GBより上のメモリ）をアドレッシングできない場合、カーネルはアドレッシング可能なメモリ上のバッファを経由してDMAを**バウンス**させなければならない。

```
Bounce buffer mechanism:
  1. Device needs to DMA to high memory (> 4GB, but device is 32-bit)
  2. swiotlb allocates a buffer in low memory (< 4GB)
  3. For writes (DMA_TO_DEVICE):  CPU copies data to bounce buffer first
  4. Device DMAs from bounce buffer
  5. For reads (DMA_FROM_DEVICE): Device DMAs to bounce buffer, then CPU copies out

  High memory buffer ──[CPU copy]──► bounce buffer ──[DMA]──► device
```

```c
/* kernel/dma/swiotlb.c — 簡略版。実際の関数には独立した alloc_size 引数は
 * なく（mapping_size とアラインメントのパディングから導出される）、スロットは
 * struct io_tlb_pool ごとに追跡される。システムは複数の swiotlb プールを
 * 持ちうるためである（CONFIG_SWIOTLB_DYNAMIC） */
phys_addr_t swiotlb_tbl_map_single(struct device *dev,
                                     phys_addr_t orig_addr,
                                     size_t mapping_size,
                                     unsigned int alloc_align_mask,
                                     enum dma_data_direction dir,
                                     unsigned long attrs)
{
    unsigned int offset = swiotlb_align_offset(dev, alloc_align_mask, orig_addr);
    size_t size = ALIGN(mapping_size + offset, alloc_align_mask + 1);
    struct io_tlb_pool *pool;
    int index;
    phys_addr_t tlb_addr;

    /* 空きのあるいずれかのプールで、空きスロットを探す */
    index = swiotlb_find_slots(dev, orig_addr, size, alloc_align_mask, &pool);
    tlb_addr = slot_addr(pool->start, index) + offset;

    /* DMA_TO_DEVICE の場合はデータをバウンスバッファにコピーする */
    if (!(attrs & DMA_ATTR_SKIP_CPU_SYNC) &&
        (dir == DMA_TO_DEVICE || dir == DMA_BIDIRECTIONAL))
        swiotlb_bounce(dev, tlb_addr, mapping_size, DMA_TO_DEVICE);

    return tlb_addr;
}
```

```bash
# swiotlb プールのサイズ（ブート時に swiotlb= パラメータで設定）
dmesg | grep "software IO TLB"
# software IO TLB: mapped [mem 0x...] (64MB)

# swiotlb の使用状況
cat /sys/kernel/debug/swiotlb/io_tlb_nslabs    # 総スロット数
cat /sys/kernel/debug/swiotlb/io_tlb_used      # 現在使用中
```

## DMAプール {#dma-pools}

小さく頻繁に割り当てられるDMAバッファ（例: ディスクリプタリング）には、断片化を避けるためにDMAプールを使う。

```c
#include <linux/dmapool.h>

/* 64 バイトにアラインされた 64 バイトの DMA コヒーレントバッファのプールを作成する */
struct dma_pool *pool = dma_pool_create("my_descriptors", dev,
                                         64,   /* サイズ */
                                         64,   /* アラインメント */
                                         0);   /* 境界（0 = 境界なし） */

/* プールから割り当てる */
dma_addr_t dma_handle;
void *vaddr = dma_pool_alloc(pool, GFP_KERNEL, &dma_handle);

/* ディスクリプタを使う */
setup_descriptor(vaddr, dma_handle);

/* プールに返す */
dma_pool_free(pool, vaddr, dma_handle);

/* プールを破棄する（下層のコヒーレントメモリをすべて解放する） */
dma_pool_destroy(pool);
```

## DMAの観測 {#observing-dma}

```bash
# DMA メモリゾーン（IOMMU のないシステム向け）
cat /proc/zoneinfo | grep -A5 "Node 0, zone  DMA"

# IOVA アロケータの統計情報
cat /sys/kernel/debug/iommu/iova

# swiotlb が使われているか確認する（アクティブなリマッピング）
cat /sys/kernel/debug/swiotlb/io_tlb_used

# DMA API のデバッグ（アンマップ済み領域へのアクセスや誤用を検出する）
# カーネル設定で CONFIG_DMA_API_DEBUG=y が必要
echo 0 > /sys/kernel/debug/dma-api/disabled  # 再有効化（1=無効、0=有効）
cat /sys/kernel/debug/dma-api/error_count
cat /sys/kernel/debug/dma-api/dump

# tracepoint: DMA の map/unmap
echo 1 > /sys/kernel/tracing/events/dma/enable
```

## デバイスツリーのDMAバインディング（ARM/組み込み） {#device-tree-dma-binding-armembedded}

ARM SoCでは、デバイスツリーがDMAの能力を指定する。

```dts
/* デバイスツリーソース */
dma-controller@40400000 {
    compatible = "arm,pl330";
    reg = <0x40400000 0x1000>;
    #dma-cells = <1>;
};

ethernet@e0100000 {
    compatible = "cdns,macb";
    reg = <0xe0100000 0x1000>;
    dmas = <&dma_controller 0>,   /* TX DMA チャネル */
           <&dma_controller 1>;   /* RX DMA チャネル */
    dma-names = "tx", "rx";
};
```

```c
/* ドライバ: デバイスツリーから DMA チャネルを要求する */
priv->dma_tx = dma_request_chan(dev, "tx");
priv->dma_rx = dma_request_chan(dev, "rx");
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/dma-mapping.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/dma-mapping.h) — `dma_map_single()`、`dma_alloc_coherent()`、`dma_mapping_error()`、`dma_set_mask_and_coherent()`: このページ全体で使っているAPIの宣言
- [kernel/dma/mapping.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma/mapping.c) — `dma_map_sg_attrs()`、`dma_alloc_attrs()`: プラットフォームの`dma_map_ops`に振り分けるコア実装
- [kernel/dma/swiotlb.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma/swiotlb.c) — `swiotlb_tbl_map_single()`、`swiotlb_find_slots()`: 実際のバウンスバッファ実装
- [drivers/iommu/dma-iommu.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu/dma-iommu.c) — `iommu_dma_map_phys()`: IOMMUを使った現在のストリーミングDMAマッピング関数
- [mm/dmapool.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/dmapool.c) — `dma_pool_create_node()`、`dma_pool_alloc()`、`dma_pool_free()`: `dma_pool_create()`の背後にあるDMAプールアロケータ
- [Documentation/core-api/dma-api-howto.rst](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/core-api/dma-api-howto.rst) — コヒーレントマッピングとストリーミングマッピングの使い分けに関するカーネル自身の実践ガイド

### 関連ページ {#related-pages}

- [IOMMUアーキテクチャ](iommu-arch.md) — `iommu_domain`、IOTLB、そしてIOMMUがある場合にこのAPIが依存するハードウェアDMAリマッピング
- [IOVAアロケータ](iova-allocator.md) — IOMMUを使うすべての`dma_map_single()`/`dma_map_sg()`呼び出しの背後にあるrbtree/rcacheアロケータ
- [IOMMU事件簿](war-stories.md) — パケットごとの`dma_map_single()`呼び出しがCPUのボトルネックとなり、`dma_map_sg()`によるバッチ化で修正された実際のインシデント
- [メモリ管理: DMA](../mm/dma.md) — 同じAPIを掘り下げる姉妹ページ。`dma_sync_*`、DMAゾーン、swiotlbの内部をより詳しく扱う
- [メモリ管理: デバイスのコヒーレンシ](../mm/device-coherency.md) — `dma_map_single()`/`dma_sync_*`が実際に何かをする必要があるのはいつかを決める、キャッシュコヒーレンシのルール
- [デバイスドライバ: PCIドライバ](../drivers/pci-driver.md) — 実際のPCIネットワークドライバで使われる`dma_alloc_coherent()`/`dma_map_single()`

### LWNの記事 {#lwn-articles}

- [The trouble with 64-bit DMA](https://lwn.net/Articles/904210/) — Jonathan Corbet、2022年8月11日。IOMMU層が64ビット対応デバイスに対しても保守的に4GB未満のIOVAを選ぶ理由と、Robin Murphyがそれを緩和しようとしたときに何が問題になったか。このページのセットアップの節で使う`DMA_BIT_MASK()`/`dma_set_mask_and_coherent()`マスクの背景
- [Noncoherent DMA mappings](https://lwn.net/Articles/855328/) — Jonathan Corbet、2021年5月7日。`dma_alloc_noncontiguous()` APIと、キャッシュ非コヒーレントなアーキテクチャ向けの手動キャッシュ同期（`dma_sync_sgtable_for_device()`/`_for_cpu()`）。このページのコヒーレントとストリーミングの区別の背景
- [Restricted DMA](https://lwn.net/Articles/841916/) — Jonathan Corbet、2021年1月7日。完全なIOMMUなしで信頼できないデバイスを分離するための、Claire Changによるデバイスごとの制限付きSWIOTLBプール。このページで説明したswiotlbバウンスバッファ機構の背景
