# PCIドライバ {#pci-drivers}

> PCIデバイスドライバを書く: probe、BARのマッピング、MSI-X割り込み

## PCIバスの概要 {#pci-bus-overview}

PCI（Peripheral Component Interconnect）は、デバイスをCPUに接続するための標準的なバスである。PCIe（PCI Express）は現代的なシリアル版だが、ドライバのAPIはほぼ同じである。`pci_register_driver()` APIと`struct pci_driver`はLinux 2.4で導入され、2.2以前の場当たり的なprobeのスタイルを置き換えた[（Linux Device Drivers, 3rd ed., ch. 12）](https://lwn.net/Kernel/LDD3/)。

```
CPU ← PCIe root complex → PCIe switch → endpoint devices
                                         (NIC, NVMe, GPU, etc.)

各PCIデバイスが持つもの:
  - Vendor ID + Device ID（各16ビット）: デバイスの種類を識別する
  - BAR（Base Address Register）: デバイスが公開するメモリ/IO領域
  - コンフィギュレーション空間（標準256B、拡張4KB）: PCIレジスタ
  - 割り込み: レガシーINTx、MSI、またはMSI-X
```

## PCIドライバの登録 {#registering-a-pci-driver}

```c
#include <linux/pci.h>
#include <linux/module.h>

/* デバイスIDテーブル: このドライバはどのデバイスを扱うか */
static const struct pci_device_id mydev_ids[] = {
    { PCI_DEVICE(0x1234, 0x5678) },    /* vendor=0x1234, device=0x5678 */
    { PCI_DEVICE(0x1234, 0x5679) },    /* 別のモデル */
    { PCI_DEVICE_CLASS(PCI_CLASS_NETWORK_ETHERNET, ~0) }, /* すべてのEthernet */
    { 0 }  /* 番兵 */
};
MODULE_DEVICE_TABLE(pci, mydev_ids);

/* ドライバの登録 */
static struct pci_driver mydev_driver = {
    .name     = "mydev",
    .id_table = mydev_ids,
    .probe    = mydev_probe,    /* デバイスが見つかったときに呼ばれる */
    .remove   = mydev_remove,   /* デバイスの削除時に呼ばれる */
    .driver   = { .pm = &mydev_pm_ops },  /* dev_pm_ops経由のPM（pci_driverには.suspend/.resumeはない） */
    .shutdown = mydev_shutdown,
};

static int __init mydev_init(void)
{
    return pci_register_driver(&mydev_driver);
}

static void __exit mydev_exit(void)
{
    pci_unregister_driver(&mydev_driver);
}

module_init(mydev_init);
module_exit(mydev_exit);
```

カーネルは、マッチするデバイスが見つかったとき（ブート時、ホットプラグ時、モジュールのロード時）に`probe()`を呼ぶ。

## probe関数 {#probe-function}

```c
struct mydev {
    struct pci_dev *pdev;
    void __iomem   *bar0;    /* BAR 0のMMIOベース */
    void __iomem   *bar2;
    int             irq;
    /* デバイスの状態 */
};

static int mydev_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
    struct mydev *dev;
    int ret;

    /* 1. ドライバのプライベートデータを割り当てる */
    dev = devm_kzalloc(&pdev->dev, sizeof(*dev), GFP_KERNEL);
    if (!dev)
        return -ENOMEM;
    dev->pdev = pdev;
    pci_set_drvdata(pdev, dev);

    /* 2. PCIデバイスを有効化する */
    ret = pci_enable_device(pdev);
    if (ret) {
        dev_err(&pdev->dev, "cannot enable PCI device\n");
        return ret;
    }

    /* 3. BARの所有権を要求する */
    ret = pci_request_regions(pdev, "mydev");
    if (ret) {
        dev_err(&pdev->dev, "cannot request regions\n");
        goto err_disable;
    }

    /* 4. BAR 0（MMIOレジスタ）をマップする */
    dev->bar0 = pci_iomap(pdev, 0, 0);  /* BAR 0の全体をマップ */
    if (!dev->bar0) {
        ret = -ENOMEM;
        goto err_release;
    }

    /* 5. DMAマスクを設定する */
    ret = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));
    if (ret) {
        ret = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(32));
        if (ret) {
            dev_err(&pdev->dev, "no usable DMA configuration\n");
            goto err_unmap;
        }
    }

    /* 6. バスマスタリングを有効化する（DMAに必要） */
    pci_set_master(pdev);

    /* 7. MSI-X割り込みを設定する */
    ret = mydev_setup_interrupts(dev);
    if (ret)
        goto err_unmap;

    /* 8. ハードウェアを初期化する */
    mydev_hw_init(dev);

    dev_info(&pdev->dev, "mydev initialized, BAR0=%pK\n", dev->bar0);
    return 0;

err_unmap:
    pci_iounmap(pdev, dev->bar0);
err_release:
    pci_release_regions(pdev);
err_disable:
    pci_disable_device(pdev);
    return ret;
}

static void mydev_remove(struct pci_dev *pdev)
{
    struct mydev *dev = pci_get_drvdata(pdev);

    mydev_hw_stop(dev);
    mydev_teardown_interrupts(dev);
    pci_iounmap(pdev, dev->bar0);
    pci_release_regions(pdev);
    pci_disable_device(pdev);
    /* devm_*による割り当ては自動的に解放される */
}
```

## BARのマッピングとMMIOアクセス {#bar-mapping-and-mmio-access}

BAR（Base Address Register）は、CPUがデバイスのメモリ/レジスタにアクセスするための窓である。

```c
/* オフセット0x10の32ビットレジスタを読む */
u32 val = ioread32(dev->bar0 + 0x10);

/* 64ビットレジスタに書き込む */
iowrite64(value, dev->bar0 + 0x20);

/* 読み出し-変更-書き込み */
u32 ctrl = ioread32(dev->bar0 + REG_CTRL);
ctrl |= CTRL_ENABLE;
iowrite32(ctrl, dev->bar0 + REG_CTRL);

/* メモリバリア: MMIOへの書き込みがデバイスから見えることを保証する */
mmiowb();  /* ほとんどのアーキテクチャではwmb() */
```

MMIO領域には、（普通のポインタの参照外しではなく）必ず`ioread*`/`iowrite*`を使うこと。これらはアーキテクチャ固有のメモリ順序付けとMMIOのセマンティクスを扱う。

```bash
# ユーザー空間からBARを調べる
lspci -vv -s <bus:dev.fn> | grep -A2 "Memory at\|I/O ports"

# すべてのPCIデバイスとそのドライバを確認
lspci -k

# PCIコンフィギュレーション空間
setpci -s <bus:dev.fn> VENDOR_ID.w   # vendor IDを読む
setpci -s <bus:dev.fn> STATUS.w      # ステータスレジスタを読む
```

## MSIとMSI-X割り込み {#msi-and-msi-x-interrupts}

現代のデバイスは、レガシーなエッジ/レベルのINTx信号の代わりにMSI（Message Signaled Interrupts）を使う。MSIはPCI 2.2仕様（1999年）で導入された。最大2048個の独立したベクタを使えるMSI-Xは、PCI 3.0で追加された[（Wikipedia: Message Signaled Interrupts）](https://en.wikipedia.org/wiki/Message_Signaled_Interrupts)。

- **レガシーINTx**: 共有ピン、低速、IOAPICのプログラミングが必要
- **MSI**: 単一の割り込みベクタ、デバイスが特別なメモリアドレスに書き込む
- **MSI-X**: 最大2048個の独立した割り込みベクタ、ベクタごとのアフィニティ

### MSIの設定（単一の割り込み） {#msi-setup-single-interrupt}

```c
/* 単一のMSI割り込みを要求する */
ret = pci_alloc_irq_vectors(pdev, 1, 1, PCI_IRQ_MSI);
if (ret < 0) {
    /* レガシーINTxにフォールバックする */
    ret = pci_alloc_irq_vectors(pdev, 1, 1, PCI_IRQ_LEGACY);
}

int irq = pci_irq_vector(pdev, 0);  /* 割り当てられたIRQ番号を取得する */

ret = request_irq(irq, mydev_isr, 0, "mydev", dev);

/* remove内で: */
free_irq(pci_irq_vector(pdev, 0), dev);
pci_free_irq_vectors(pdev);
```

### MSI-Xの設定（複数のベクタ） {#msi-x-setup-multiple-vectors}

```c
#define MYDEV_NUM_QUEUES 4

static int mydev_setup_interrupts(struct mydev *dev)
{
    struct pci_dev *pdev = dev->pdev;
    int nvecs, i, ret;

    /* 最大MYDEV_NUM_QUEUES個のMSI-Xベクタを要求する */
    nvecs = pci_alloc_irq_vectors(pdev,
                                   1,                  /* 最小ベクタ数 */
                                   MYDEV_NUM_QUEUES,   /* 最大ベクタ数 */
                                   PCI_IRQ_MSIX);
    if (nvecs < 0)
        return nvecs;

    dev->num_queues = nvecs;

    /* ベクタごとにIRQハンドラを要求する */
    for (i = 0; i < nvecs; i++) {
        int irq = pci_irq_vector(pdev, i);
        snprintf(dev->queue[i].irq_name, sizeof(dev->queue[i].irq_name),
                 "mydev-q%d", i);

        ret = request_irq(irq, mydev_queue_isr, 0,
                           dev->queue[i].irq_name, &dev->queue[i]);
        if (ret) {
            while (--i >= 0)
                free_irq(pci_irq_vector(pdev, i), &dev->queue[i]);
            pci_free_irq_vectors(pdev);
            return ret;
        }

        /* 各キューのIRQを特定のCPUに固定する */
        irq_set_affinity_hint(irq, cpumask_of(i % num_online_cpus()));
    }
    return 0;
}
```

### MSI-X割り込みハンドラ {#msi-x-interrupt-handler}

```c
static irqreturn_t mydev_queue_isr(int irq, void *data)
{
    struct mydev_queue *q = data;
    u32 status;

    /* 割り込みステータスレジスタを読む */
    status = ioread32(q->dev->bar0 + QUEUE_STATUS(q->idx));
    if (!(status & QUEUE_IRQ_PENDING))
        return IRQ_NONE;

    /* 割り込みをクリアする */
    iowrite32(QUEUE_IRQ_CLEAR, q->dev->bar0 + QUEUE_STATUS(q->idx));

    /* 完了を処理する */
    mydev_process_completions(q);

    return IRQ_HANDLED;
}
```

## PCIでのDMA {#dma-with-pci}

```c
/* コヒーレントなDMAメモリを割り当てる（CPUとデバイスが同じデータを見る） */
dma_addr_t dma_handle;
void *cpu_addr = dma_alloc_coherent(&pdev->dev, 4096, &dma_handle, GFP_KERNEL);
/* dma_handle: デバイスのDMAレジスタに設定するアドレス */
/* cpu_addr: CPUの仮想アドレス */

/* デバイスにDMAアドレスを設定する */
iowrite64(dma_handle, dev->bar0 + REG_DMA_BASE);

/* 後で: 解放する */
dma_free_coherent(&pdev->dev, 4096, cpu_addr, dma_handle);

/* ストリーミングDMA（既存のバッファ用）: */
dma_addr_t dma = dma_map_single(&pdev->dev, skb->data, skb->len,
                                  DMA_TO_DEVICE);
/* ... デバイスを設定する ... */
/* デバイスの処理完了後: */
dma_unmap_single(&pdev->dev, dma, skb->len, DMA_TO_DEVICE);
/* これでskb->dataに安全にアクセスできる */
```

## ドライバからPCIコンフィギュレーション空間を読む {#reading-pci-configuration-space-from-a-driver}

```c
u16 vendor, device;
u8  revision;
u32 class_code;

pci_read_config_word(pdev, PCI_VENDOR_ID, &vendor);
pci_read_config_word(pdev, PCI_DEVICE_ID, &device);
pci_read_config_byte(pdev, PCI_REVISION_ID, &revision);
pci_read_config_dword(pdev, PCI_CLASS_REVISION, &class_code);

/* リンク速度を確認する（PCIe） */
u16 lnksta;
pcie_capability_read_word(pdev, PCI_EXP_LNKSTA, &lnksta);
int speed = lnksta & PCI_EXP_LNKSTA_CLS;  /* リンク速度 */
int width = (lnksta & PCI_EXP_LNKSTA_NLW) >> 4;  /* リンク幅 */
```

## devm_ヘルパーの利用 {#using-devm_-helpers}

`devm_*`（device-managed）リソースは、デバイスが削除されたときに自動的に解放される。

```c
/* メモリ */
dev = devm_kzalloc(&pdev->dev, sizeof(*dev), GFP_KERNEL);

/* IOMAP */
dev->bar0 = devm_pci_iomap(pdev, 0, 0);

/* IRQ */
ret = devm_request_irq(&pdev->dev, irq, handler, 0, "mydev", dev);

/* devm_リソースについては.removeでの手動クリーンアップは不要 */
```

## 参考文献 {#further-reading}

- [Linuxデバイスモデル](device-model.md) — struct device、ドライバのbind
- [プラットフォームドライバ](platform-driver.md) — 検出できないデバイス
- [DMA API](../mm/dma.md) — dma_alloc_coherentとスキャッタ・ギャザー
- [PCI BARのマッピング](../mm/pci-bar-mapping.md) — BARの物理レイアウト
- [割り込み処理](../interrupts/interrupts.md) — IRQディスクリプタのパス
- `include/linux/pci.h` — PCIドライバAPI
- `drivers/net/ethernet/intel/e1000e/` — 実際のNICドライバの例
