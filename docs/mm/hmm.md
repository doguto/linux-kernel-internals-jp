# HMM: Heterogeneous Memory Management {#hmm-heterogeneous-memory-management}

> CPUとGPUの間で仮想アドレス空間を共有する — デバイスメモリのためのstruct page

## 問題 {#the-problem}

現代のシステムには、独自のメモリ（VRAM）を持つGPUやその他のアクセラレータがある。アプリケーションは、CPUとGPUにまたがって単一の仮想アドレス空間を使いたい。

```
Traditional (two address spaces):
  CPU:  malloc → virtual addr 0x7f000000
  GPU:  cudaMalloc → device addr 0xc000000000
  Programmer: explicit copies between the two

With HMM (unified virtual address space):
  CPU:  mmap → virtual addr 0x7f000000
  GPU:  same virtual addr 0x7f000000 (GPU MMU maps same pages)
  Programmer: writes to address, GPU reads from same address
  Kernel: tracks ownership and handles faults on both sides
```

## HMMの構成要素 {#hmm-components}

```
Application virtual address space
         │
    ┌────┴────────────────────────────────┐
    │         CPU page tables (MMU)       │
    │         GPU page tables (IOMMU)     │
    └────┬────────────────────────────────┘
         │
    ┌────┴───────────────────────┐
    │        HMM layer           │
    │  - mirror (CPU↔GPU sync)   │
    │  - range (snapshot VA)     │
    │  - device memory (VRAM)    │
    └────┬───────────────────────┘
         │
    ┌────┴────────────────────────────────────────────┐
    │  Physical memory:                               │
    │  DRAM (system RAM)    VRAM (GPU memory)         │
    │  struct page          struct page (ZONE_DEVICE) │
    └─────────────────────────────────────────────────┘
```

## デバイスメモリのためのstruct page {#struct-page-for-device-memory}

HMMは `ZONE_DEVICE` を通じて `struct page` を拡張し、GPU/デバイスのメモリを表現できるようにする。

```c
/* mm/memremap.c */

/* デバイスメモリの種類: */
enum memory_type {
    MEMORY_DEVICE_PRIVATE,     /* GPUのプライベートメモリ。CPUからアクセスできない */
    MEMORY_DEVICE_COHERENT,    /* CPUからアクセス可能なデバイスメモリ（CXLなど、5.19+） */
    MEMORY_DEVICE_FS_DAX,      /* DAXファイルシステム */
    MEMORY_DEVICE_PCI_P2PDMA,  /* PCIeデバイス間のP2P DMA */
};

/* デバイスメモリをZONE_DEVICEのページとして登録する: */
struct dev_pagemap pgmap = {
    .type  = MEMORY_DEVICE_PRIVATE,
    .range = {
        .start = phys_base,
        .end   = phys_base + size - 1,
    },
    .ops = &my_device_pagemap_ops,
};
devm_memremap_pages(dev, &pgmap);
/* これで: デバイスメモリのページに対してpfn_to_page(pfn)が機能する */
```

## mmu_interval_notifier: CPU→GPUのページテーブル同期 {#mmu_interval_notifier-cpugpu-page-table-synchronization}

ドライバは `mmu_interval_notifier`（Linux 5.5で導入され、以前の `hmm_mirror` が5.10で削除されて唯一のAPIとなった）を使い、仮想アドレス範囲のCPUページテーブルが変化したときにコールバックを受け取る。

### mmu_interval_notifier（Linux 5.5+） {#mmu_interval_notifier-linux-55}

```c
/* 推奨される新しいAPI: 完全なミラーではなく、範囲の通知だけ */
struct mmu_interval_notifier notifier;

static const struct mmu_interval_notifier_ops gpu_mni_ops = {
    .invalidate = gpu_invalidate_range,
};

/* 登録: [start, end) のVA範囲が変化したときに通知される */
mmu_interval_notifier_insert(&notifier, mm, start, length, &gpu_mni_ops);

static bool gpu_invalidate_range(struct mmu_interval_notifier *mni,
                                  const struct mmu_notifier_range *range,
                                  unsigned long cur_seq)
{
    /* GPUはこの範囲へのアクセスを止めなければならない */
    gpu_unmap_range(mni->start, mni->length);
    mmu_interval_set_seq(mni, cur_seq);
    return true;
}
```

## HMMの範囲スナップショット {#hmm-range-snapshot}

ドライバは `hmm_range_fault()` を使って、ある範囲の現在のCPUページテーブルの状態のスナップショットを取り、GPUがマップすべき物理アドレス（PFN）を得る。

```c
/* drivers/gpu/my_driver.c（簡略化） */

static int gpu_fault_and_map(struct mm_struct *mm, unsigned long addr,
                              unsigned long len)
{
    unsigned long npages = len / PAGE_SIZE;
    unsigned long hmm_pfns[npages];
    struct hmm_range range = {
        .notifier     = &notifier,
        .notifier_seq = mmu_interval_read_begin(&notifier),
        .start        = addr,
        .end          = addr + len,
        .hmm_pfns     = hmm_pfns,   /* フィールド名はpfnsではなくhmm_pfns */
        .pfn_flags_mask = HMM_PFN_REQ_FAULT | HMM_PFN_REQ_WRITE,
    };
    /* 要求フラグを設定: すべてのページをフォールトで読み込み、書き込み可能にするよう求める */
    for (int i = 0; i < npages; i++)
        hmm_pfns[i] = HMM_PFN_REQ_FAULT | HMM_PFN_REQ_WRITE;

    int ret;
retry:
    ret = hmm_range_fault(&range);
    if (ret == -EBUSY) {
        /* スナップショット中にページテーブルが変化した — リトライする */
        range.notifier_seq = mmu_interval_read_begin(&notifier);
        goto retry;
    }
    if (ret)
        return ret;

    /* hmm_pfns[i]には各ページのPFNとフラグが入っている */
    /* ページが存在すればHMM_PFN_VALIDが立つ */
    /* 書き込み可能ならHMM_PFN_WRITEが立つ */
    for (int i = 0; i < npages; i++) {
        if (!(hmm_pfns[i] & HMM_PFN_VALID))
            continue;
        /* PFNを取り出す: フラグビットをシフトして取り除く */
        unsigned long pfn = hmm_pfns[i] >> HMM_PFN_SHIFT;
        phys_addr_t phys = PFN_PHYS(pfn);
        gpu_map_page(addr + i * PAGE_SIZE, phys,
                     !!(hmm_pfns[i] & HMM_PFN_WRITE));
    }
    return 0;
}
```

## デバイスプライベートメモリのマイグレーション {#device-private-memory-migration}

（CPUからアドレス指定できない）プライベートなVRAMを持つGPUでは、pageは必要に応じてシステムRAMとVRAMの間でマイグレーションされる。

```
GPU accesses CPU page:
  GPU page fault → driver's fault handler
  → migrate_vma_setup() + migrate_vma_pages()
  → pages move from DRAM to VRAM
  → GPU page table updated with VRAM physical address

CPU accesses GPU page:
  CPU page fault (page has ZONE_DEVICE private struct page)
  → driver's migrate_to_ram callback
  → pages move from VRAM to DRAM
  → CPU page table updated with DRAM physical address
```

```c
/* ドライバが実装するもの: */
static const struct dev_pagemap_ops my_pgmap_ops = {
    /* CPUがデバイスプライベートなページにアクセスしようとしたときに呼ばれる */
    .migrate_to_ram = my_gpu_migrate_to_ram,

    /* デバイスプライベートなページの参照カウントが0になったときに呼ばれる */
    .page_free = my_gpu_page_free,
};

static vm_fault_t my_gpu_migrate_to_ram(struct vm_fault *vmf)
{
    /* GPUのページをシステムRAMへマイグレーションして戻す */
    struct migrate_vma args = {
        .vma        = vmf->vma,
        .start      = vmf->address & PAGE_MASK,
        .end        = (vmf->address & PAGE_MASK) + PAGE_SIZE,
        .src        = src_pfns,
        .dst        = dst_pfns,
        .pgmap_owner = my_gpu_device,
        .flags      = MIGRATE_VMA_SELECT_DEVICE_PRIVATE,
    };

    migrate_vma_setup(&args);
    /* args.srcにはマイグレーションするGPUページのPFNが入っている */

    /* システムRAMのページを割り当て、GPUメモリの内容をそこへコピーする */
    for (int i = 0; i < npages; i++) {
        if (!(args.src[i] & MIGRATE_PFN_MIGRATE))
            continue;
        struct page *dpage = alloc_page(GFP_HIGHUSER_MOVABLE);
        gpu_copy_to_ram(args.src[i], dpage);
        args.dst[i] = migrate_pfn(page_to_pfn(dpage));
    }

    migrate_vma_pages(&args);    /* RAMのページをCPUのPTEに設定する */
    migrate_vma_finalize(&args); /* GPUのページを解放する */
    return 0;
}
```

## 実際のSVM（Shared Virtual Memory） {#svm-shared-virtual-memory-in-practice}

CUDAとROCmはHMMを使ってSVMを実装している。

```c
/* CUDAのマネージドメモリ: */
cudaMallocManaged(&ptr, size, cudaMemAttachGlobal);
/* ptrはHMMに裏付けられたCPUのVA */
/* CPUからのアクセス → 通常のページフォールト。カーネルがDRAMを割り当てる */
/* GPUからのアクセス → GPUフォールト。ドライバがhmm_range_fault()を呼ぶ */
/*                   か、VRAMへマイグレーションする */

/* cudaMemPrefetchAsync: 必要になる前にページを移動するためのヒント */
cudaMemPrefetchAsync(ptr, size, cudaCpuDeviceId);  /* CPUへプリフェッチ */
cudaMemPrefetchAsync(ptr, size, 0);                /* GPU 0へプリフェッチ */
```

## HMMの観測 {#observing-hmm}

```bash
# カーネルが HMM をサポートしているか確認する:
zcat /proc/config.gz | grep -E "ZONE_DEVICE|DEVICE_PRIVATE"
# CONFIG_ZONE_DEVICE=y
# CONFIG_DEVICE_PRIVATE=y
# （CONFIG_HMM_MIRROR は 5.10 で削除された。HMM のサポートは DEVICE_PRIVATE によって暗黙的に有効になる）

# ZONE_DEVICE メモリの統計:
cat /proc/zoneinfo | grep -A20 "Device"

# nvidia-smi はユニファイドメモリの統計を表示する:
nvidia-smi --query-gpu=memory.used,memory.free --format=csv

# HMM の範囲フォールトをトレースする:
bpftrace -e '
kprobe:hmm_range_fault {
    printf("hmm_range_fault: pid=%d addr=%lx\n", pid, ((struct hmm_range *)arg0)->start);
}'

# NUMA を考慮: HMM は page を GPU メモリ（別の NUMA ノード）へマイグレーションする
numastat -p <cuda_process>
```

## 参考文献 {#further-reading}

- [メモリ階層化](memory-tiering.md) — CXLとNUMAのメモリ階層
- [IOMMU](../iommu/iommu-arch.md) — GPUのDMAのためのIOMMU
- [VFIO](../virtualization/vfio.md) — IOMMUを使ったデバイスパススルー
- [DMA API](../iommu/dma-api.md) — DMAマッピングの基礎
- [ページアロケータ](page-allocator.md) — ZONE_DEVICEの統合
- `mm/hmm.c` — HMMのコア実装
- `mm/migrate_device.c` — デバイスメモリのマイグレーション
- `Documentation/mm/hmm.rst` — カーネルのHMMドキュメント
