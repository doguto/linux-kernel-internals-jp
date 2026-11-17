# IOVAアロケータ {#iova-allocator}

> カーネルはI/O仮想アドレス空間をどう管理するか

## IOVA空間とは {#what-is-iova-space}

IOMMUドメインにアタッチされた各デバイスは、独自のI/O仮想アドレス空間を持つ。ドライバが`dma_map_single()`や`dma_map_sg()`を呼ぶと、DMA-IOMMU層はI/O仮想アドレス（IOVA）の範囲を切り出し、その範囲についてIOMMUページテーブルにマッピングを作成し、IOVAをドライバに返さなければならない。転送が完了してドライバがアンマップすると、それらのアドレスはプールに返されなければならない。

パケットレートやI/Oレートが高い場合、この割り当てと解放のサイクルは高速パス上にある。小さなパケットを受信する25 GbpsのNICは、毎秒数百万回のDMAマップを発行しうる。そのためIOVAアロケータはCPUオーバーヘッドの大きな発生源となり、ネットワークやNVMeのワークロードの`perf top`プロファイルに現れる。

アロケータは`drivers/iommu/iova.c`にあり、型は`include/linux/iova.h`で宣言されている。

## 中核となるデータ構造 {#core-data-structures}

### struct iova_domain {#struct-iova_domain}

```c
/* include/linux/iova.h */
struct iova_domain {
    spinlock_t      iova_rbtree_lock; /* rbtree へのアクセスを直列化する */
    struct rb_root  rbroot;           /* 割り当て済み IOVA の rbtree */
    struct rb_node *cached_node;      /* 直近の free() 位置のヒント */
    struct rb_node *cached32_node;    /* 4GB 未満の範囲用のキャッシュノード */
    unsigned long   granule;          /* 割り当て粒度（ページサイズ） */
    unsigned long   start_pfn;        /* 割り当て可能な最小の IOVA PFN */
    unsigned long   dma_32bit_pfn;    /* 4GB 未満で最大の IOVA PFN */
    unsigned long   max32_alloc_size; /* 4GB 未満での 1 回の割り当ての最大値 */
    struct iova      anchor;          /* rbtree の番兵ノード */
    struct iova_rcache *rcaches; /* iova_domain_init_rcaches() で動的に割り当てられる */
    struct hlist_node cpuhp_dead;     /* CPU ホットプラグ時の rcache 状態の破棄 */
};
```

ドメインは、デバイスのグループが利用できるIOVAアドレス空間全体をカバーする。`granule`は通常4 KB（1ページ）であり、IOVAはページサイズ単位で割り当てられる。rbtreeは割り当て済みの範囲を追跡し、アロケータが空き領域を見つけられるようにする。

### struct iova {#struct-iova}

```c
/* include/linux/iova.h */
struct iova {
    struct rb_node  node;
    unsigned long   pfn_hi;   /* この割り当ての最後の PFN（この値を含む） */
    unsigned long   pfn_lo;   /* この割り当ての最初の PFN */
};
```

割り当て済みの各範囲は、rbtreeに埋め込まれた`struct iova`として表される。`pfn_lo`と`pfn_hi`はIOVAのPFN（IOVA >> PAGE_SHIFT）である。

## 割り当てと解放の呼び出し経路 {#allocation-and-free-call-paths}

### alloc_iova() {#alloc_iova}

```c
/* drivers/iommu/iova.c */
struct iova *alloc_iova(struct iova_domain *iovad,
                        unsigned long size,      /* ページフレーム単位 */
                        unsigned long limit_pfn, /* 上限 */
                        bool size_aligned);
```

この関数はrbtreeを`limit_pfn`から下向きに探索する（32ビットデバイスのために低位アドレスを空けておくよう、上から下へ割り当てる）。まずキャッシュされたヒントをたどり、それが外れた場合は空きの隙間を見つけるためにrbtree全体の走査にフォールバックする。

上から下へ割り当てるのには重要な理由がある。後から来る32ビットのレガシーデバイスのために、低位のIOVA空間（4 GB未満）を空けておくためである。64ビット対応デバイスは上から埋めていき、32ビットに制限されたデバイスは`dma_32bit_pfn`から下向きに埋めていく。

ただし、これは単純化した説明である。64ビットのDMAマスクを持つデバイスであっても、最初の割り当てで高位のIOVAを得るわけではない。PCIデバイスの場合、`iommu_dma_alloc_iova()`は*まず*32ビットの範囲を試す。これはSAC-before-DACの互換性回避策であり、デバイスごとの`pci_32bit_workaround`フラグ（`struct dev_iommu`）で制御される。この32ビットでの試行が失敗して初めて、アドレス範囲全体にフォールバックする。実際の割り当てロジックは次の節を参照。

### free_iova() {#free_iova}

```c
/* drivers/iommu/iova.c */
void free_iova(struct iova_domain *iovad, unsigned long pfn);
```

rbtreeからノードを削除し、キャッシュされたヒントを更新する。spinlockによってこれは直列化されるため、高頻度ではボトルネックになる。per-CPUのrcacheは、よく使われるサイズについてこの経路を完全に回避するために存在する。

### DMA-IOMMUラッパー {#the-dma-iommu-wrapper}

実際には、ドライバが`alloc_iova()`を直接呼ぶことはない。DMA-IOMMU層は`iommu_dma_alloc_iova()`を呼ぶ。この関数は、アドレス範囲全体が必要だとまだ判明していないPCIデバイスについては、まず32ビットの領域を試し、それが尽きて初めて汎用の経路（これもper-CPU rcacheを経由し、ミスすると`alloc_iova()`にフォールバックする）を呼び出す。

```c
/* drivers/iommu/dma-iommu.c（簡略版 — この前に行われる MSI クッキーの
 * 短絡処理と、デバイス/ドメインのアパーチャによる dma_limit の
 * クランプは省略している） */
static dma_addr_t iommu_dma_alloc_iova(struct iommu_domain *domain,
                                        size_t size, u64 dma_limit,
                                        struct device *dev)
{
    struct iommu_dma_cookie *cookie = domain->iova_cookie;
    struct iova_domain *iovad = &cookie->iovad;
    unsigned long shift = iova_shift(iovad);
    unsigned long iova_len = size >> shift;
    unsigned long iova;

    /* SAC-before-DAC: PCI デバイスについては、まず一度だけ 32 ビットの範囲を試す */
    if (dma_limit > DMA_BIT_MASK(32) && dev->iommu->pci_32bit_workaround) {
        iova = alloc_iova_fast(iovad, iova_len, DMA_BIT_MASK(32) >> shift, false);
        if (iova)
            goto done;
        /* 失敗: このデバイスは本当に高位の範囲を必要としている */
        dev->iommu->pci_32bit_workaround = false;
        dev_notice(dev, "Using %d-bit DMA addresses\n", bits_per(dma_limit));
    }

    /* 高速パス: per-CPU rcache（ミスすると alloc_iova() にフォールバック） */
    iova = alloc_iova_fast(iovad, iova_len, dma_limit >> shift, true);
done:
    return (dma_addr_t)iova << shift;
}
```

`pci_32bit_workaround`はすべてのPCIデバイスで`true`から始まり（`iommu.forcedac`ブートパラメータで無効にされない限り）、32ビットの範囲で要求を満たせなかった最初の時点で`false`にラッチされる（そのデバイスについては恒久的に）。実際のソースのコメントは、これが現代のハードウェアでもなお重要な理由をはっきり述べている。"the original SAC vs. DAC reasoning loses relevance with PCIe, but enough hardware and firmware bugs are still lurking out there that it's safest not to venture into the 64-bit space until necessary."（訳: 元々のSAC対DACの理由づけはPCIeでは意味を失っているが、ハードウェアやファームウェアのバグはまだ十分に潜んでいるので、必要になるまで64ビット空間に踏み込まないのが最も安全である。）

## per-CPU IOVAキャッシュ（rcache） {#per-cpu-iova-caches-rcache}

マガジン方式のper-CPUキャッシュこそが、高速パスでのIOVA割り当てをrbtreeのロックから切り離す鍵である。

### struct iova_rcacheとstruct iova_cpu_rcache {#struct-iovarcache-and-struct-iovacpu_rcache}

```c
/* drivers/iommu/iova.c — これらはすべて iova.c の内部にあり、
 * include/linux/iova.h には公開されていない（ヘッダは struct iova_rcache を前方宣言するだけ） */

/* 127 は sizeof(struct iova_magazine) がちょうど 1024 バイトになるよう選ばれている
 * (127 × 8 + 8 = 1024)。kmalloc の内部断片化を避けるため。 */
#define IOVA_MAG_SIZE 127

struct iova_magazine {
    union {
        unsigned long size;         /* 使用中は充填数 */
        struct iova_magazine *next; /* デポ内ではフリーリストのリンク */
    };
    unsigned long  pfns[IOVA_MAG_SIZE];
};

struct iova_cpu_rcache {
    spinlock_t         lock;
    struct iova_magazine *loaded;   /* 現在のマガジン（ここから取り出す） */
    struct iova_magazine *prev;     /* 予備のマガジン */
};

struct iova_rcache {
    spinlock_t          lock;       /* デポを保護する */
    unsigned int         depot_size;
    struct iova_magazine *depot;    /* デポのマガジンの単方向リスト */
    struct iova_cpu_rcache __percpu *cpu_rcaches;
    struct iova_domain  *iovad;     /* 所有するドメイン（下記のデポ縮小ワーカー用） */
    struct delayed_work  work;      /* 定期的にデポを縮小する */
};
```

遅延ワークのコールバック（`iova_depot_work_func()`、周期は`IOVA_DEPOT_DELAY` = 100ms）が定期的に
デポのマガジンを`num_online_cpus()`個まで解放する。デポは無制限の貯蔵庫ではない。

最近のカーネルでは`IOVA_RANGE_CACHE_MAX_SIZE`は6と定義されており、6つのサイズクラスがある。各サイズクラスは特定の2のべき乗のページ数（1、2、4、8、16、32ページ）のIOVAを管理し、ネットワークやNVMeのワークロードで最もよく使われるDMAバッファサイズをカバーする。

### マガジンキャッシュの仕組み {#how-the-magazine-cache-works}

この設計はslabのマガジンアロケータ（Bonwick & Adams, 2001）に着想を得ている。

```
Per-CPU rcache:
  loaded magazine  [pfn, pfn, pfn, ..., pfn]  <- alloc/free here
  prev magazine    [pfn, pfn, pfn, ..., pfn]  <- spare

Global depot:
  [mag0] [mag1] [mag2] ... [magN]             <- overflow magazines
```

**割り当て（高速パス、rbtreeロックなし）:**
1. 該当するサイズクラスの`loaded`マガジンからPFNを1つ取り出す。
2. `loaded`が空なら、`loaded`と`prev`を入れ替える。`prev`も空なら、グローバルなデポから満杯のマガジンを取ってくる。デポも空なら、`alloc_iova()`（低速パス、rbtreeロックを取得する）にフォールバックする。

**解放（高速パス、rbtreeロックなし）:**
1. PFNを`loaded`マガジンに積む。
2. `loaded`が満杯なら、`loaded`と`prev`を入れ替える。`prev`も満杯なら、`prev`をデポに送り、`prev`用に新しい空のマガジンを割り当てる。

重要な性質は、通常の割り当て/解放サイクルでは他のCPUとのspinlock競合が起きないことである。各CPUは自分のloadedマガジンを独立して管理する。

**性能に関する注記:** rcacheこそが、IOMMUを有効にしたシステムで10 Gbps以上のNICを実用的にしているものである。rcacheがなければ、パケットごとのDMAマップがグローバルなrbtreeのspinlockを取得することになる。シングルスレッドの25 Gbps小パケットワークロードは毎秒300万回以上の`dma_map_single()`呼び出しを発行しうるが、rcacheは一般的なケースではCPU間の同期なしにこれを吸収する。

## IOVAフラッシュキュー（struct iova_fq） {#iova-flush-queues-struct-iova_fq}

フラッシュキューは、IOVAの解放とIOTLBの無効化を切り離す。DMA範囲のアンマップには、IOVAの解放とIOMMU TLBのフラッシュの両方が必要である。IOTLBのフラッシュは高コストであり、Intel VT-dではDMARレジスタへの書き込みを必要とし、バスをストールさせることがある。

この概念はLinux 5.15にさかのぼるが、下に示すper-CPUの形ではなかった。5.15のフラッシュキューは、`iova.c`にある単一の、より単純なドメインごとの構造体だった。`dma-iommu.c`にあるper-CPUの`struct iova_fq`の設計（現在サポートされているすべてのカーネルが実際に動かしているもの）は、その後の作り直しで導入され、Linux **5.17**で初めて登場した。

```c
/* drivers/iommu/dma-iommu.c — include/linux/iova.h ではない。そちらには "fq" のシンボルは一切ない */
#define IOVA_DEFAULT_FQ_SIZE   256

struct iova_fq_entry {
    unsigned long iova_pfn;
    unsigned long pages;
    struct iommu_pages_list freelist; /* このエントリとともに解放される IOMMU ページテーブルのページ。
                                        * バッチ化されたフラッシュが実際に走るまで遅延される */
    u64 counter;               /* このエントリが追加されたときのフラッシュカウンタ */
};

struct iova_fq {
    spinlock_t lock;
    unsigned int head, tail;
    unsigned int mod_mask;     /* リングサイズ - 1。entries[] のサイズは 2 のべき乗 */
    struct iova_fq_entry entries[]; /* フレキシブル配列メンバ */
};
```

アンマップのたびにすぐ`iommu_iotlb_sync()`を呼ぶ代わりに、フラッシュキューは解放されたIOVAを蓄積する。バックグラウンドのワーカー（または次のドメイン全体のフラッシュ）がキューを排出し、バッチ化された1回のIOTLB無効化を行ってから、IOVAをrcacheまたはrbtreeに返す。これにより、ハードウェアの無効化コストが多数のアンマップにわたって償却される。

ドメインタイプ`IOMMU_DOMAIN_DMA_FQ`がフラッシュキューモードを有効にする。

```c
/* drivers/iommu/dma-iommu.c — 実際のシグネチャはクッキーを直接ではなく
 * ドメインを受け取り、MSI クッキーの破棄という別のケースも扱う */
static void iommu_dma_free_iova(struct iommu_domain *domain, dma_addr_t iova,
                                size_t size, struct iommu_iotlb_gather *gather)
{
    struct iova_domain *iovad = &domain->iova_cookie->iovad;

    if (domain->cookie_type == IOMMU_COOKIE_DMA_MSI) {
        /* MSI の場合: 常に直近の割り当てだけを片付ける */
        domain->msi_cookie->msi_iova -= size;
    } else if (gather && gather->queued) {
        queue_iova(domain->iova_cookie, iova_pfn(iovad, iova),
                   size >> iova_shift(iovad), &gather->freelist);
    } else {
        free_iova_fast(iovad, iova_pfn(iovad, iova),
                       size >> iova_shift(iovad));
    }
}
```

## 32ビットDMAの制限 {#the-32-bit-dma-limit}

多くのレガシーデバイスや組み込みデバイスは、32ビットのDMAアドレスレジスタを持つ。RAMが4 GBを超えるシステムでは、IOMMUはそれらのデバイスが4 GB未満のIOVAを受け取るよう保証しなければならない。4 GBより上の物理メモリは問題ないが、デバイスから見えるIOVAは32ビットに収まらなければならない。

`struct iova_domain`の`dma_32bit_pfn`フィールドは、32ビットIOVA領域の上限を示す。

```c
/* init_iova_domain() は、ドメインの granule だけから導出した固定の 4 GB 境界を
 * 設定する -- struct device は受け取らず、どのデバイスの DMA マスクによっても
 * クランプされない:
 *   iovad->dma_32bit_pfn = 1UL << (32 - iova_shift(iovad));
 * デバイスごとの DMA マスクによるクランプは、このフィールドを書き換えるのではなく、
 * iommu_dma_alloc_iova() 内のローカルな dma_limit パラメータとして別途行われる。 */
```

ドライバが`dma_set_mask(dev, DMA_BIT_MASK(32))`を呼ぶと、以後の割り当ては`limit_pfn = iovad->dma_32bit_pfn`を使う。アロケータは4 GB未満の領域だけを探索することで、この制約を満たす。

その結果、32ビットデバイスが多いシステムでは、低位のIOVA範囲が奪い合いになる。大規模な64ビット対応デバイスは明示的に`dma_set_mask(dev, DMA_BIT_MASK(64))`を呼んで制約のない高位範囲から割り当てられるようにし、低位のIOVAを本当に必要とするデバイスのために残しておくべきである。

## DMA性能への影響 {#impact-on-dma-performance}

IOVAの割り当ては、プロファイルに2つの形で現れる。

1. **`iova_rbtree_lock`のロック競合** — rcacheが冷えているとき（ドライバの起動時、または6つのサイズクラスのどれにも当たらないバッファサイズ）に起きる。`perf`では、spinlockを保持したまま`alloc_iova()`で費やされる時間として見える。

2. **IOTLBフラッシュのコスト** — フラッシュキューがなければ、`dma_unmap_single()`のたびにIOTLB無効化が発行される。Intel VT-dでは呼び出しごとに数百ナノ秒かかる。フラッシュキューはこのコストをバッチ化する。

```bash
# ドメインごとの IOVA アロケータの状態を見る
# 注: 汎用の /sys/kernel/debug/iommu/iova というファイルは 1 つも存在しない。
# IOVA 関連の統計はベンダー固有のパスの下に現れる。例:
#   /sys/kernel/debug/iommu/intel/  (Intel VT-d)

# perf: NIC ドライバのワークロードで IOVA 割り当てのコストを見つける
perf record -g -- netperf -t UDP_STREAM -H <host> -l 30
perf report --sort=symbol | grep -i iova

# DMA の map/unmap のタイミング用 tracepoint
echo 1 > /sys/kernel/tracing/events/dma/enable
cat /sys/kernel/tracing/trace
```

高スループットのドライバでよくあるチューニングの誤りは、バッチ化したスキャッタギャザーエントリで`dma_map_sg()`を使う代わりに、パケットごとに`dma_map_single()`を使うことである。`dma_map_sg()`は連続したページをまとめてIOVAの割り当て回数を減らすので、アロケータへの負荷と、フラッシュしなければならないIOMMUページテーブルエントリの数の両方が減る。

## ソースファイル {#source-files}

| ファイル | 役割 |
|------|------|
| `drivers/iommu/iova.c` | IOVAのrbtreeアロケータ、rcache、フラッシュキュー |
| `include/linux/iova.h` | `iova_domain`、`iova`、`iova_rcache`、`iova_fq`構造体の定義 |
| `drivers/iommu/dma-iommu.c` | DMA-IOMMUのグルー: DMA APIからiova.cを呼び出す |
| `drivers/iommu/iommu.c` | `iommu_iotlb_gather`の蓄積とsync |

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [drivers/iommu/iova.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu/iova.c) — `alloc_iova()`、`free_iova()`、rbtreeアロケータ、per-CPU rcache/マガジンの実装
- [include/linux/iova.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/iova.h) — `struct iova_domain`と`struct iova`の定義
- [drivers/iommu/dma-iommu.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu/dma-iommu.c) — `iommu_dma_alloc_iova()` / `iommu_dma_free_iova()`、およびフラッシュキュー（`struct iova_fq`）の実装

### 関連ページ {#related-pages}

- [DMA API](dma-api.md) — `dma_map_single`、`dma_map_sg`、swiotlb
- [IOMMUアーキテクチャ](iommu-arch.md) — IOMMUドメイン、IOTLBフラッシュの仕組み
- [IOMMU事件簿](war-stories.md) — 性能ボトルネックとなったIOVAアロケータ（実際のインシデント）
