# 共有仮想アドレッシング {#shared-virtual-addressing}

> アクセラレータやスマートNIC向けのPASIDベースのSVA

## SVAが解決する問題 {#the-problem-sva-solves}

従来のIOMMU DMAでは、デバイスは独自のI/O仮想アドレス空間（IOVA空間）で動作する。ドライバがデバイスにユーザー空間のバッファへアクセスさせたい場合、カーネルは次のことをしなければならない。

1. ユーザーページをピン留めする。
2. IOVA範囲を割り当てる。
3. それらのページのIOMMUマッピングを作成する。
4. IOVAをデバイスに渡す。
5. 操作の後: アンマップし、ピン留めを解除し、IOVAを解放する。

ユーザー空間の小さな操作を大量に処理するアクセラレータ（Intel DSAのスキャッタギャザー、GPUの計算カーネル、SmartNICのオフロード）にとって、操作ごとのこのピン留めとマッピングは高コストである。さらに、すべてのアクセスをカーネルまたは特権ドライバが仲介する必要がある。

**共有仮想アドレッシング（SVA: Shared Virtual Addressing）**は、デバイスがユーザープロセスと同じ仮想アドレス空間を共有できるようにする。デバイスはプロセスの仮想アドレスを直接使える。IOVA変換層もなく、投入時のピン留めもない。IOMMUは、どのプロセスのページテーブルを使うかを識別する**PASID**をデバイスのすべてのトランザクションに付けることで、分離を強制する。

## PASID: Process Address Space ID {#pasid-process-address-space-id}

PASIDは、PCIe仕様で定義された20ビットのPCIe TLPプレフィックスフィールドである。デバイスはDMAの読み出しや書き込みを発行する際、トランザクションにPASID値のタグを付けられる。IOMMUはこのタグを使ってアドレス変換に用いる正しいページテーブルを選ぶ。

```
Device DMA request:
  [PASID=42] [virtual address 0x7fff1000] [read 4096 bytes]
                        │
                        ▼
              IOMMU PASID table (indexed by PASID)
                        │
                        ▼
              Process 42's page tables (cr3 equivalent)
                        │
                        ▼
              Physical address → memory access
```

SVAがなければ、あるIOMMUドメインに対するデバイスのトランザクションはすべて単一のページテーブル（そのドメインのIOVA空間）を使う。SVAでは、各PASIDが1つのプロセスのページテーブルに対応する。IOMMUハードウェアは、デバイスのコンテキストエントリとともにPASIDテーブルを保持する。

PASID値は0から2^20−1（100万強）まで使える。実際には、カーネルは`iommu_alloc_global_pasid()`（drivers/iommu/iommu.c、カーネル6.1以降）を使ってグローバルなID空間からPASIDを割り当てる。

## PCIeケーパビリティの要件 {#pcie-capability-requirements}

SVAは、連携して動作する3つのPCIeケーパビリティに依存する。

### ATS（Address Translation Services） {#ats-address-translation-services}

ATSは、デバイスがIOVAから物理アドレスへの変換をデバイス側のTLB（ATC — Address Translation Cache）にキャッシュできるようにする。SVAの下では、デバイスは変換要求を上流に送り、IOMMUは物理アドレスで応答する。デバイスはそれをキャッシュし、同じ仮想ページへの以後のアクセスに使う。

ATSは任意である。ATSがなければ、デバイスのメモリアクセスのたびにIOMMUのページウォークが完全に行われる。ATSがあれば、デバイスのATCがほとんどのルックアップを吸収する。

```bash
# デバイスが ATS をサポートしているか確認する
lspci -vvv -s 0000:01:00.0 | grep "Address Translation"
# Capabilities: [100] Address Translation Service (ATS)
#   ATSCap: Invalidate Queue Depth: 32
#   ATSCtl: Enable+, Smallest Translation Unit: 0
```

### PRI（Page Request Interface） {#pri-page-request-interface}

SVAなしでは、カーネルはデバイスに作業を投入する前にすべてのページを事前にフォールトさせておく。SVAでは、デバイスはトランザクションの途中でページフォールト（ページが存在しない、スワップアウトされている、まだフォールトインされていない）に遭遇しうる。PRIにより、デバイスは**Page Request**を上流に送り、IOMMU/CPUにフォールトを解決してページの準備ができたら通知するよう要求できる。

IOMMUはページ要求をカーネルのIOMMUページフォールトハンドラに配送する。カーネルはこれをCPUのページフォールトと同様に処理する。ページを割り当てるか読み込み、ページテーブルを更新し、**Page Request Response**をデバイスに返して処理を再開させる。

PRIにはATSが必要である。すべてのハードウェアがPRIをサポートしているわけではない。PRIのないハードウェアでも、ドライバが投入前にすべてのページを事前フォールトさせればSVAは動作しうる（ただしオンデマンドページングの利点は失われる）。

### ACS（Access Control Services） {#acs-access-control-services}

ACSは、PCIeのピアツーピアトランザクションがIOMMUを迂回するのを防ぐ。ACSがなければ、デバイスはIOMMUを通さずに別のデバイスのメモリへDMAでき、分離が無効になる。ACSはIOMMUグループの分離の前提条件であり、IOMMUグループの分離はSVAの安全性の前提条件である。

```bash
# ACS ケーパビリティを確認する
lspci -vvv -s 0000:00:1c.0 | grep "Access Control"
# Capabilities: [148] Access Control Services
#   ACSCap: SrcValid+ TransBlk+ ReqRedir+ CmpltRedir+ UpstreamFwd- EgressCtrl- DirectTrans-
#   ACSCtl: SrcValid+ TransBlk+ ReqRedir+ CmpltRedir+ UpstreamFwd- EgressCtrl- DirectTrans-
```

## io_pgtable_ops: ページテーブルの抽象化 {#iopgtableops-the-page-table-abstraction}

IOMMUハードウェアによってページテーブルの形式は異なる。Intel VT-dは独自の多段ページテーブルを使い、AMD-Viは別の形式を使う。ARM SMMU v3はネイティブ形式とARM64 CPUのページテーブル形式の両方をサポートする（これがARMでSVAを効率的にしている。デバイスは文字どおりCPUと同じテーブルをウォークする）。

カーネルはこれを`struct io_pgtable_ops`の背後に抽象化している。

```c
/* include/linux/io-pgtable.h */
struct io_pgtable_ops {
    int (*map_pages)(struct io_pgtable_ops *ops,
                     unsigned long iova, phys_addr_t paddr,
                     size_t pgsize, size_t pgcount,
                     int prot, gfp_t gfp, size_t *mapped);

    size_t (*unmap_pages)(struct io_pgtable_ops *ops,
                          unsigned long iova,
                          size_t pgsize, size_t pgcount,
                          struct iommu_iotlb_gather *gather);

    phys_addr_t (*iova_to_phys)(struct io_pgtable_ops *ops,
                                 unsigned long iova);
};
```

`io_pgtable_ops`は、従来のカーネル管理のドメインが使うものである。IOMMUドライバが自前のページテーブルをウォークして埋める場合に`alloc_io_pgtable_ops()`が返す抽象化である。**SVAドメインは`io_pgtable_ops`のインスタンスをまったく割り当てない**。IOMMUが別のテーブルを保持する代わりに、ハードウェアはプロセスの*既存の*CPUページテーブルを直接指す。ARM SMMU v3の実装（`drivers/iommu/arm/arm-smmu-v3/`）はこれをContext Descriptor（CD）テーブルによって行う。ストリームテーブルエントリはCDテーブルを指し、各CDエントリ（PASIDでインデックス付け）はプロセスの`pgd`を指すTTBR相当の値を保持する。Intelの実装（`drivers/iommu/intel/svm.c`）は、DMARのPASIDテーブルに第1レベルページテーブル（スケーラブルモードのFLPTR）を指すPASIDエントリを作る。`intel_pasid_setup_first_level()`はそこに`__pa(mm->pgd)`を設定する。これはそのプロセスについてCPU自身のCR3レジスタが保持するのと同じ物理アドレスであり、IOMMUがCR3そのものを読むわけではない。

## LinuxのSVA API（カーネル5.2以降） {#the-linux-sva-api-kernel-52}

Linux 5.2で導入された`iommu_sva` API（`iommu_sva_bind_device()`/`iommu_sva_unbind_device()`/`iommu_sva_get_pasid()`）は、3つの操作を提供する。

```c
/* include/linux/iommu.h */

/*
 * デバイスを現在のプロセスのアドレス空間にバインドする。
 * バインディングを表すハンドルを返す。呼び出し元は
 * iommu_sva_get_pasid() で PASID を得る。
 *
 * デバイスは SVA をサポートしていなければならない（IOMMU ドライバがケーパビリティを確認する）。
 * 呼び出しプロセスの mm_struct はハンドルの存続期間中ピン留めされる。
 */
struct iommu_sva *iommu_sva_bind_device(struct device *dev,
                                         struct mm_struct *mm);

/*
 * バインディングを解放する。この呼び出しの後、デバイスは対応する
 * PASID で DMA トランザクションを発行してはならない。
 */
void iommu_sva_unbind_device(struct iommu_sva *handle);

/*
 * このバインディングに割り当てられた PASID を取得する。
 * ドライバはこの値をデバイスに設定し、デバイスがトランザクションにタグを付けられるようにする。
 */
u32 iommu_sva_get_pasid(struct iommu_sva *handle);
```

### 典型的なドライバの流れ {#typical-driver-flow}

```c
/* キャラクタデバイスの open または ioctl ハンドラ内 */
struct iommu_sva *sva_handle;
u32 pasid;

/* バインド: current->mm をデバイスに関連付ける */
sva_handle = iommu_sva_bind_device(dev->device, current->mm);
if (IS_ERR(sva_handle)) {
    ret = PTR_ERR(sva_handle);
    goto err;
}

/* ハードウェアに設定する PASID を取得する */
pasid = iommu_sva_get_pasid(sva_handle);

/* デバイスが DMA にこの値のタグを付けるよう、PASID をデバイスレジスタに書き込む */
writel(pasid, dev->bar + DEV_PASID_REG);

/* これでユーザー空間は仮想アドレスを直接使って作業を投入できる */
/* デバイスは IOMMU 経由でプロセスのページテーブルをウォークする */

/* close/クリーンアップ時 */
iommu_sva_unbind_device(sva_handle);
```

## mm_structからPASIDテーブルへ: カーネルの流れ {#mm_struct-to-pasid-table-the-kernel-flow}

`iommu_sva_bind_device()`が呼ばれると、次のことが起きる。

1. **PASIDの割り当て**: `iommu_alloc_global_pasid()`がグローバルなID空間からPASIDを確保する。
2. **PASIDテーブルエントリ**: IOMMUドライバが、プロセスのページテーブルのルートをPASIDテーブルのインデックスPASIDの位置に書き込む。Intelでは、`intel_svm_set_dev_pasid()`が`intel_pasid_setup_first_level()`を呼び、DMARのPASIDテーブルエントリにプロセスのCR3を設定する（IntelのSVAは常に第1レベルページテーブルを使う）。ARM SMMU v3では、`arm_smmu_sva_set_dev_pasid()`がCD（Context Descriptor）エントリを設定する。
3. **mm notifier**: カーネルはプロセスの`mm_struct`に`mmu_notifier`コールバックを登録する。プロセスのページテーブルが変わると（ページのアンマップ、THPの分割、プロセスの終了）、notifierが発火し、このPASIDを共有するすべてのデバイスに対してIOTLB無効化を引き起こす。これによってデバイスのATCがCPUのページテーブルとコヒーレントに保たれる。
4. **ハンドルを返す**: `struct iommu_sva`がPASIDとmmへの参照をラップする。

```
mm_struct
  ├── pgd (page table root)
  └── mmu_notifier_mm
        └── iommu_sva_notifier ──► triggers ATC invalidation on page table changes
                                     │
                                     ▼
                               iommu_sva_handle
                                 ├── pasid = 42
                                 └── iommu_domain (PASID entry points here)
```

## ユースケース {#use-cases}

### Intel DSA（Data Streaming Accelerator） {#intel-dsa-data-streaming-accelerator}

DSA（Intel Sapphire Rapids以降で利用可能）は、メモリのコピー、フィル、CRC、比較の操作を行う。SVAを使えば、ユーザー空間のライブラリ（DPDKやSPDKが使うIntelの`idxd`ユーザー空間ドライバ）が仮想アドレスを含むディスクリプタを投入できる。DSAは、カーネルが何もピン留めやマップをすることなく、ユーザーの仮想アドレスを読み書きする。

```c
/* SVA を使った DSA ディスクリプタ（ユーザー空間の PASID フロー） */
struct dsa_hw_desc desc = {
    .opcode     = DSA_OPCODE_MEMMOVE,
    .src_addr   = (uint64_t)src_vaddr,  /* プロセスの仮想アドレス */
    .dst_addr   = (uint64_t)dst_vaddr,  /* プロセスの仮想アドレス */
    .xfer_size  = len,
    .flags      = IDXD_OP_FLAG_CRAV | IDXD_OP_FLAG_RCR,
};
/* DSA のワークキューに投入される。デバイスは PASID を使ってアドレスを解決する */
```

### SVAを使うARM SMMU v3 {#arm-smmu-v3-with-sva}

SVAをサポートするARM SMMUv3（`CONFIG_ARM_SMMU_V3_SVA`）は、ARM64のページテーブルをSMMUと直接共有できる。SMMUはARM64のページテーブル形式をネイティブに理解するので、SVAの設定はStream Table内のポインタを設定するだけで済み、形式の変換は不要である。

### GPU計算（ROCm/CUDA） {#gpu-compute-rocmcuda}

GPU計算のワークロード（AMD GPU上のROCm、一部のCUDA構成）は、ユニファイドメモリのためにSVAの恩恵を受ける。GPUとCPUは仮想アドレス空間を共有し、IOMMUがプロセスごとのアクセス制御を強制する。GPU側のページフォールトはカーネルのIOMMUフォールトハンドラが処理し、ページを読み込んでからGPUに再試行を通知する。

## 制限事項 {#limitations}

- **ハードウェアのサポートが必要**: SVAには、PASIDテーブルをサポートするIOMMUハードウェア（PASIDケーパビリティを持つIntel VT-d、PASIDをサポートするAMD-Vi、CDテーブルを持つARM SMMUv3）が必要である。
- **PRIは任意だが重要**: PRIがなければ、デバイスへの投入前にすべてのページを事前フォールトさせなければならない。オンデマンドページングはできなくなるが、仮想アドレスの再利用は依然として可能である。
- **分離にはACSが必要**: PCIeトポロジーは、デバイスとルートコンプレックスの間のすべてのブリッジでACSをサポートしていなければならない。そうでなければIOMMUグループの分離は健全でない。
- **PASIDテーブルのサイズ制限**: PASIDが20ビットであるため、IOMMUあたりの同時バインディング数は最大1M。実際には、IOMMUハードウェアがサポートする数はそれより少ないことがある（Intelでは`ecap`レジスタのビット、ARMではSMMU IDR1を確認する）。
- **プロセスの存続期間**: デバイスのバインディングが有効な間にプロセスが終了すると、`mmu_notifier`のexitコールバックが発火する。カーネルはPASIDテーブルエントリを破棄し、そのPASIDのトランザクションを受け付けなくなる。デバイスドライバはこれを適切に処理しなければならない（保留中のトランザクションではDMAエラーになる）。

```bash
# Intel IOMMU の PASID サポートを確認する
dmesg | grep -i pasid
# DMAR: PASID supported

# ARM SMMU の SVA サポート
dmesg | grep -i sva
# arm-smmu-v3: SVA support enabled

# カーネルの SVA 設定を確認する
grep CONFIG_IOMMU_SVA /boot/config-$(uname -r)
# CONFIG_IOMMU_SVA=y
```

## ソースファイル {#source-files}

| ファイル | 役割 |
|------|------|
| `drivers/iommu/iommu-sva.c` | `iommu_sva_bind_device()`、mm notifier、PASIDのライフサイクル |
| `drivers/iommu/intel/svm.c` | Intel VT-dのSVA: PASIDテーブルの設定、PRIの処理 |
| `drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3-sva.c` | ARM SMMUv3のSVA: CDテーブル、s1ページテーブルの共有 |
| `drivers/iommu/ioasid.c` | PASIDの割り当て（`ioasid_alloc()`、`ioasid_free()`）— 5.11〜6.0に存在した。PASIDの割り当てがiommuコア（`drivers/iommu/iommu.c`）に統合された6.1で削除された |
| `include/linux/iommu.h` | `iommu_sva`、`iommu_sva_bind_device()`の宣言 |
| `include/linux/io-pgtable.h` | `io_pgtable_ops`の抽象化 |

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [drivers/iommu/iommu-sva.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu/iommu-sva.c) — `iommu_sva_bind_device()`、`iommu_sva_unbind_device()`、`iommu_sva_get_pasid()`、およびSVAのI/Oページフォールトハンドラ
- [drivers/iommu/iommu.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu/iommu.c) — `iommu_alloc_global_pasid()` / `iommu_free_global_pasid()`: グローバルなPASID割り当て。現在は削除された`ioasid.c`からここに統合された
- [drivers/iommu/intel/svm.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu/intel/svm.c) — Intel VT-dのSVA: `intel_svm_set_dev_pasid()`と第1レベルPASIDテーブルのセットアップ
- [drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3-sva.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3-sva.c) — ARM SMMUv3のSVA: `arm_smmu_sva_set_dev_pasid()`と、`mm->pgd`からのContext Descriptorの設定
- [include/linux/iommu.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/iommu.h) — `struct iommu_sva`と`iommu_sva_bind_device()`系の宣言

### 関連ページ {#related-pages}

- [IOMMUアーキテクチャ](iommu-arch.md) — SVAが土台とするIOMMUドメイン、グループ、VT-d/AMD-Viハードウェア
- [DMA API](dma-api.md) — SVAが回避するために設計された、ピン留めとマップの経路
- [VFIOの内部](vfio-internals.md) — デバイスパススルー: コンテナへのピン留め+マップ。SVAのPASIDタグ付き共有とは異なる分離モデル
- [IOMMU事件簿](war-stories.md) — ケース4「アクセラレータによるSVAページフォールトストーム」

### LWNの記事 {#lwn-articles}

- [5.7 Merge window part 1](https://lwn.net/Articles/816313/) — Jonathan Corbet、2020年4月3日。「unified user-space access-intended accelerator framework」（uacce）のマージを取り上げている。これはSVAを使うデバイスをユーザー空間に公開するための汎用ドライバフレームワーク（HiSiliconのアクセラレータが使用）である

### 外部資料 {#external}

- [Documentation/arch/x86/sva.rst](https://docs.kernel.org/arch/x86/sva.html) — SVA/PASID/ENQCMDに関するカーネル自身のドキュメント。PASIDのライフサイクル、`IA32_PASID` MSR、PCIe ATS/PRIの背景
