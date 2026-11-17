# IOMMUとDMA {#iommu-and-dma}

> デバイスのためのハードウェアアドレス変換とメモリ保護

## なぜIOMMUか {#why-iommu}

IOMMUがなければ、DMA可能なデバイスは任意の物理メモリにアクセスできる。乗っ取られたネットワークカードがカーネルメモリを読み取ったり上書きしたりできてしまう。IOMMUはPCIeバスと物理メモリの間に位置し、デバイスのI/O仮想アドレス（IOVA）を物理アドレスに変換するとともに、アクセス権限を強制する。

```
Without IOMMU:
  Device → physical memory (unrestricted)

With IOMMU:
  Device → IOVA → IOMMU translation → physical memory
                  (checked against per-domain page tables)
```

セキュリティ以外にも、IOMMUは次のことを可能にする。

- **IOVAリマッピング**: アドレスバスが64ビット未満のデバイスでも高位メモリにDMAできる
- **スキャッタギャザーの平坦化**: 不連続な物理ページがデバイスからは連続して見える
- **デバイスパススルー（VFIO）**: 物理デバイスを安全にVMへ割り当てる

## このセクションのページ {#pages-in-this-section}

| ページ | 扱う内容 |
|------|----------------|
| [IOMMUアーキテクチャ](iommu-arch.md) | Intel VT-d、AMD-Vi、iommu_domain、IOMMUグループ、VFIO |
| [DMA API](dma-api.md) | dma_alloc_coherent、dma_map_single/sg、swiotlb、IOVA割り当て |
| [IOVAアロケータ](iova-allocator.md) | iova_domain、rbtree、per-CPU rcacheマガジン、フラッシュキュー |
| [共有仮想アドレッシング](sva.md) | PASID、ATS、PRI、iommu_sva API、Intel DSA、ARM SMMU v3 |
| [VFIOの内部](vfio-internals.md) | コンテナ/グループ/デバイスの階層、mdev、MSI irqfd、KVMパススルー |
| [IOMMU事件簿](war-stories.md) | DMARフォールト、IOVAの性能ボトルネック、グループ分離、swiotlb枯渇 |

## 読む順番 {#reading-order}

IOMMUの内部に初めて触れるなら、次の順にページを読み進めるとよい。

1. **[README](README.md)** — このページ。IOMMUが存在する理由とクイックリファレンス
2. **[IOMMUアーキテクチャ](iommu-arch.md)** — ハードウェア（VT-d、AMD-Vi、ARM SMMU）、カーネルの抽象化（`iommu_domain`、グループ）、IOTLB
3. **[DMA API](dma-api.md)** — ドライバが実際にDMAをどう使うか。コヒーレント、ストリーミング、スキャッタギャザー、swiotlb
4. **[IOVAアロケータ](iova-allocator.md)** — DMAマッピングの下にあるアドレスアロケータ。rcacheとフラッシュキューの内部、性能への影響
5. **[共有仮想アドレッシング](sva.md)** — 発展的内容。アクセラレータ向けのPASIDベースのVA共有。IOMMUドメインの理解が前提
6. **[VFIOの内部](vfio-internals.md)** — デバイスパススルーのアーキテクチャ。ページのピン留め、割り込み配送、mdev
7. **[IOMMU事件簿](war-stories.md)** — ここまでの内容すべてを結びつける実際のインシデント。理解の定着に役立つ

## クイックリファレンス {#quick-reference}

```bash
# IOMMUが有効か確認する
dmesg | grep -i iommu | head -20
# DMAR: IOMMU enabled
# AMD-Vi: device isolation enabled

# IOMMUグループ（アドレス空間を共有するデバイス群）を一覧表示する
ls /sys/kernel/iommu_groups/
for g in /sys/kernel/iommu_groups/*/devices/*; do echo "$g"; done

# ブート時にIOMMUを有効にする（カーネルパラメータ）
# Intel: intel_iommu=on
# AMD:   amd_iommu=on
# 両方:  iommu=pt (パススルー — 変換なし、グループのみ)

# VFIO: VMパススルー用にデバイスをバインドする
echo "8086 1521" | sudo tee /sys/bus/pci/drivers/vfio-pci/new_id
echo "0000:01:00.0" | sudo tee /sys/bus/pci/devices/0000:01:00.0/driver/unbind
echo "0000:01:00.0" | sudo tee /sys/bus/pci/drivers/vfio-pci/bind
```
