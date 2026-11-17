# MSIとMSI-X: PCIeのメッセージシグナル割り込み {#msi-and-msi-x-pcie-message-signaled-interrupts}

> NVMe、NIC、GPU向けに、レガシーなINTxをキューごとの割り込みベクタで置き換える

## なぜMSI-Xか {#why-msi-x}

レガシーなPCI割り込み（INTx、INTA〜INTD）には根本的な制約がある。

- **共有**: 複数のデバイスが1本のIRQラインを共有する → スプリアス割り込みが起き、分離されない
- **レベルトリガ**: 割り込みラインは受領通知されるまで保持される → 取りこぼしはない（クリアされるまでラインはアサートされたまま）。一方、エッジトリガの割り込みは、マスク中に遷移が起きると取りこぼされることがある
- **数が限られる**: PCIバスあたり4本のラインしかない

MSI（Message-Signaled Interrupts）のサポートはTom Long Nguyen（Intel）によってLinux 2.6.8（2004年）で追加された。MSIは専用の配線の代わりにメモリ書き込みを使うことで、これらの問題を解決する。

```
Legacy INTx:
  Device → asserts pin → CPU interrupt controller → CPU

MSI/MSI-X:
  Device → writes message to memory address (MSI address reg) → CPU
           This write IS the interrupt — the memory write triggers the CPU
```

**MSI-X**はMSIを拡張し、デバイスあたり2048個のベクタ（MSIでは32個）を扱えるようにしたもので、各ベクタは独立したアフィニティを持てる。

## MSI capability構造 {#msi-capability-structure}

```
PCI config space:
  [PCI header]
  [Capabilities list]
    └── MSI capability (cap ID = 0x05):
         offset  size  field
         00      1     Capability ID (0x05)
         01      1     Next Pointer
         02-03   2     Message Control
                         [0]: MSI Enable
                         [3:1]: Multiple Message Capable (log2 of requested vectors)
                         [6:4]: Multiple Message Enable
                         [7]: 64-bit capable
         04-07   4     Message Address (low 32 bits, 4-byte aligned)
         08-0B   4     Message Address High (if 64-bit)
         0C-0D   2     Message Data
                         CPU writes: APIC ID + vector number
```

```c
/* カーネルからMSI capabilityを読む: */
u16 ctrl;
pci_read_config_word(dev, pos + PCI_MSI_FLAGS, &ctrl);
int nvec = 1 << ((ctrl & PCI_MSI_FLAGS_QMASK) >> 1);  /* 最大ベクタ数 */
```

## MSI-Xテーブル {#msi-x-table}

MSI-Xは、デバイスのBARの1つに置かれたテーブルを使う。

```
MSI-X capability (cap ID = 0x11):
  Message Control:
    [10:0]: Table Size - 1  (up to 2048 entries)
    [13]: Function Mask      (mask all vectors)
    [15]: MSI-X Enable

  Table Offset/BIR:
    [2:0]: BAR Index (which BAR contains the table)
    [31:3]: Table Offset (byte offset into that BAR)

  Pending Bit Array (PBA) Offset/BIR:
    [2:0]: BAR Index
    [31:3]: PBA Offset

MSI-X Table entry (16 bytes each):
  [31:0]:  Message Address Low
  [63:32]: Message Address High
  [95:64]: Message Data          (APIC ID + vector)
  [127:96]: Vector Control
               [0]: Masked (1 = masked)
```

## Linuxカーネル: MSI-Xベクタの要求 {#linux-kernel-requesting-msi-x-vectors}

```c
/* drivers/net/ethernet/intel/igb/igb_main.c（簡略化） */

static int igb_set_interrupt_capability(struct igb_adapter *adapter)
{
    int numvecs, err;

    /* TXキューごとに1つ + RXキューごとに1つ + その他用に1つのベクタ取得を試みる */
    numvecs = adapter->num_tx_queues + adapter->num_rx_queues + 1;

    /* MSI-Xベクタを要求する: */
    /* pci_alloc_irq_vectors: まずMSI-Xを試し、だめならMSI、さらにINTxにフォールバックする */
    err = pci_alloc_irq_vectors(adapter->pdev,
                                 1,         /* 最小ベクタ数 */
                                 numvecs,   /* 最大ベクタ数 */
                                 PCI_IRQ_MSIX | PCI_IRQ_MSI);
    if (err < 0) {
        /* 単一の割り込みにフォールバックする */
        pci_alloc_irq_vectors(adapter->pdev, 1, 1, PCI_IRQ_ALL_TYPES);
        return 0;
    }
    adapter->num_q_vectors = err;  /* 実際に割り当てられたベクタ数 */
    return 0;
}

/* 各ベクタにハンドラを登録する: */
for (i = 0; i < adapter->num_q_vectors; i++) {
    /* pci_irq_vector: ベクタiのIRQ番号を取得する */
    int irq = pci_irq_vector(adapter->pdev, i);

    snprintf(name, sizeof(name), "%s-q%d", netdev->name, i);
    err = request_irq(irq, igb_msix_ring, 0, name,
                       adapter->q_vector[i]);
}

/* 後始末時に解放する: */
pci_free_irq_vectors(adapter->pdev);
```

## ベクタごとのIRQアフィニティ {#per-vector-irq-affinity}

MSI-Xでは、各ベクタを別々のCPUに固定できる。

```bash
# NVMeデバイスのMSI-X IRQを一覧表示する:
ls /proc/irq/ | xargs -I{} cat /proc/irq/{}/smp_affinity_list 2>/dev/null
# ほとんどのIRQは多数のCPUに対応付けられている

# 8キューのMSI-X NVMeの場合:
# irq 32 → cpu 0  (queue 0)
# irq 33 → cpu 1  (queue 1)
# ...
# irq 39 → cpu 7  (queue 7)

# ベクタ0のアフィニティをCPU 0に設定する:
echo 0 > /proc/irq/32/smp_affinity_list  # CPU 0のみ（smp_affinity_listはビットマスクではなくCPU番号を使う）

# あるいはirqbalanceで自動的に分散させる:
systemctl start irqbalance
```

```c
/* カーネル: アフィニティヒントを設定する（ドライバが呼ぶ）: */
irq_set_affinity_hint(irq, cpumask);
/* irqbalanceはIRQを分散する際にヒントを尊重する */

/* あるいはアフィニティを強制する: */
irq_set_affinity(irq, cpumask);
```

## 割り込みコアレッシング {#interrupt-coalescing}

割り込みレート（とCPUのオーバーヘッド）を減らすため、ドライバは複数の完了をまとめる。

```bash
# 現在のコアレッシング設定を確認する:
ethtool -c eth0
# Coalesce parameters for eth0:
# Adaptive RX: off  TX: off
# rx-usecs: 50       ← 50µsアイドルの後に割り込みを生成
# rx-frames: 0       ← または0パケット後（無効）
# tx-usecs: 50
# tx-frames: 0

# 割り込みレートを下げる（コアレッシングを増やす = レイテンシ増、CPU負荷減）:
ethtool -C eth0 rx-usecs=100 tx-usecs=100

# 適応型コアレッシング（負荷に応じて自動調整）:
ethtool -C eth0 adaptive-rx on adaptive-tx on
```

### カーネルのコアレッシングサポート {#kernel-coalescing-support}

```c
/* NICドライバが実装する: */
static int igb_set_coalesce(struct net_device *netdev,
                             struct ethtool_coalesce *ec,
                             struct kernel_ethtool_coalesce *kernel_coal,
                             struct netlink_ext_ack *extack)
{
    struct igb_adapter *adapter = netdev_priv(netdev);

    /* rx-usecs → RDTR（Receive Delay Timer Register） */
    adapter->rx_itr_setting = ec->rx_coalesce_usecs;

    /* ハードウェアレジスタを設定する: */
    wr32(E1000_ITR, adapter->rx_itr_setting * 256000000 / 1000000);
    /* ハードウェア: キューがITRマイクロ秒アイドルになると割り込みが発火する */
    return 0;
}
```

## MSI-Xの状態確認 {#checking-msi-x-status}

```bash
# MSI-Xが有効か確認する:
lspci -vvv -s 03:00.0 | grep -A 10 "MSI-X"
# Capabilities: [b0] MSI-X: Enable+ Count=33 Masked-
# Vector table: BAR=4 offset=00002000
# PBA: BAR=4 offset=00003000

# 使用中のMSI-Xベクタ数:
cat /proc/interrupts | grep nvme
# 32:          0          0    PCI-MSI 2621440-edge   nvme0q0
# 33:      12345         67    PCI-MSI 2621441-edge   nvme0q1
# ...

# デバイスのCPUごとの割り込み回数:
cat /proc/irq/32/affinity_hint     # CPUアフィニティのヒント
cat /proc/irq/32/effective_affinity_list  # 実際のCPU

# ベクタごとの統計:
cat /sys/bus/pci/devices/0000:03:00.0/msi_irqs/
# 32  33  34  ...  （MSI-Xベクタごとに1つ）
```

## MSIとMSI-Xの比較 {#msi-vs-msi-x-comparison}

| | INTx | MSI | MSI-X |
|---|------|-----|-------|
| 最大ベクタ数 | 4 | 32 | 2048 |
| 共有 | あり（共有配線） | なし | なし |
| ベクタごとのアフィニティ | なし | なし | あり |
| マスク | INTx無効化 | 全ベクタ一括 | ベクタごと |
| テーブルの場所 | PCIピン | コンフィグ空間 | BAR |
| 用途 | レガシー | 単純なデバイス | NVMe、10GbE、GPU |

## 参考文献 {#further-reading}

- [PCIドライバ](../drivers/pci-driver.md) — MSIの設定を含むPCIeドライバの基本
- [IRQアフィニティ](../interrupts/irq-affinity.md) — IRQのCPUアフィニティの設定
- [NVMeドライバ](../block/nvme.md) — キューごとに1つのMSI-Xベクタを使う
- [IRQディスクリプタ](../interrupts/irq-desc.md) — struct irq_descとirq_chip
- `drivers/pci/msi/` — MSI/MSI-Xの実装
- `Documentation/PCI/msi-howto.rst` — ドライバ向けMSI利用ガイド
