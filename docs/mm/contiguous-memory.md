# なぜ連続メモリを割り当てられないのか {#why-cant-i-allocate-contiguous-memory}

> 長時間稼働するすべてのシステムを悩ませる断片化の問題

## 問い {#the-question}

空きメモリは4GBある。DMA用に4MBの連続バッファが必要だ。しかし割り当ては失敗する。なぜか？

```c
// 何ギガバイトも空いていても失敗することがある
struct page *pages = alloc_pages(GFP_KERNEL, 10);  // order 10 = 4MB (1024 pages)

// 大きなサイズの kmalloc は内部でページアロケータにフォールバックするため、
// 同じ断片化の制約を受ける
void *buf = kmalloc(4 * 1024 * 1024, GFP_KERNEL);
```

*注: [`MAX_PAGE_ORDER`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mmzone.h) はほとんどの構成で通常10（4KBページで4MB）である。`kmalloc()` には個別のサイズ上限はなく、大きな要求は内部で高次（high-order）ページ割り当てになる（mm/slub.c の `__kmalloc_large_noprof()` を参照）。*

答えは**外部断片化**である。空きメモリは存在するが、連続した塊になっていないのだ。

## なぜ連続メモリが重要か {#why-contiguous-memory-matters}

カーネルの割り当ての大半は物理的な連続性を必要としない。仮想メモリを使えば、散らばった物理pageを連続した仮想アドレス範囲にマップできる（それが `vmalloc()` の仕事である）。

しかし、本当に物理的に連続したメモリを必要とするものもある。

| ユースケース | なぜ連続が必要か |
|----------|-----------------|
| DMAバッファ | 多くのデバイスはscatter-gatherリストを扱えない |
| ヒュージページ | 2MB/1GBのpageはアラインされた連続領域を必要とする |
| ネットワークバッファ | 一部のデバイスや経路では連続性が有利になる（ただし多くはscatter-gatherを使う） |
| GPUメモリ | グラフィックスドライバは大きな連続割り当てを必要とすることが多い |

## 断片化の問題 {#the-fragmentation-problem}

バディアロケータはメモリを2のべき乗のブロックに分割する。数日から数週間稼働した後、システムのメモリは次のようになる。

```
Physical memory after extended use:

┌─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┐
│U│F│U│U│F│U│F│F│U│U│U│F│U│F│U│F│F│F│U│U│F│U│F│U│U│F│F│U│F│U│F│U│
└─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┘
 U = Used page, F = Free page

Free pages: 14 (56KB)
Largest contiguous block: 3 pages (12KB)
```

56KBの空きがあるのに16KBのブロックを割り当てられない。これが外部断片化である。

### どのように起きるか {#how-it-happens}

1. システムは大きな連続した空き領域を持った状態で起動する
2. さまざまなサイズの割り当てが行われては解放される
3. 一部の割り当ては残り続ける（カーネル構造体、ピン留めされたpage）
4. 空きメモリは残り続ける割り当ての周りに散らばっていく

残り続ける割り当ては川の中の岩のように振る舞う。空きメモリはその周りを流れるが、合体することはできない。

## マイグレートタイプ: カーネルの最初の防御策 {#migrate-types-the-kernels-first-defense}

カーネルは断片化を減らすため、pageを**移動可能性**（mobility）によってグループ化する。

```
MIGRATE_UNMOVABLE   - Kernel structures that can't move
MIGRATE_MOVABLE     - User pages that can be migrated
MIGRATE_RECLAIMABLE - Caches that can be freed
```

バディアロケータはマイグレートタイプごとに別々のフリーリストを保持する（[mm/page_alloc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c)）。

```
┌─────────────────────────────────────────────────────────────┐
│                     Physical memory                         │
├───────────────────┬────────────────────┬────────────────────┤
│    UNMOVABLE      │      MOVABLE       │    RECLAIMABLE     │
│  (kernel stuff)   │   (user pages)     │    (caches)        │
└───────────────────┴────────────────────┴────────────────────┘
```

これが役立つ理由は次のとおりである。

- 移動可能なpageはマイグレーションして連続した空き領域を作れる
- 回収可能なpageはメモリ逼迫時に解放できる
- 移動不可能なpageはまとめてグループ化され、断片化の広がりを抑える

しかし完璧ではない。メモリ逼迫時には、アロケータは他のマイグレートタイプからpageを「盗み」、移動可能な領域を移動不可能な割り当てで汚染してしまう。

## メモリコンパクション: Linuxのデフラグ {#memory-compaction-defragmentation-for-linux}

高次の割り当てが失敗すると、カーネルは**コンパクション**、すなわちpageを移動して連続した空き領域を作ることを試みられる。

```
Before compaction:
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ M │ F │ M │ F │ M │ F │ M │ F │  M = Movable, F = Free
└───┴───┴───┴───┴───┴───┴───┴───┘

After compaction:
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ M │ M │ M │ M │ F │ F │ F │ F │  Contiguous free region!
└───┴───┴───┴───┴───┴───┴───┴───┘
```

コンパクションはMel Gormanによってカーネル2.6.35で追加された（[commit 748446bb6b5a](https://git.kernel.org/linus/748446bb6b5a) | [LKML](https://lore.kernel.org/lkml/1269347146-7461-1-git-send-email-mel@csn.ul.ie/)、2010年）。それ以前は、長時間稼働するシステムで高次割り当ての失敗はよくあることで、唯一の解決策は再起動だった。

アルゴリズムの詳細は[コンパクション](compaction.md)を参照。

### コンパクションの限界 {#limitations-of-compaction}

コンパクションは**移動可能な**pageしか移動できない。移動不可能なpage（カーネルの割り当て、ピン留めされたメモリ）は恒久的な障害物となる。

```
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ U │ M │ F │ U │ M │ F │ U │ F │  U = Unmovable (can't move!)
└───┴───┴───┴───┴───┴───┴───┴───┘
                │
                ▼ After compaction
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ U │ M │ M │ U │ F │ F │ U │ F │  Still fragmented around U
└───┴───┴───┴───┴───┴───┴───┴───┘
```

## CMA: デバイスドライバのための解決策 {#cma-the-solution-for-device-drivers}

**Contiguous Memory Allocator**（CMA）は、二重の役割を果たす領域を予約することでデバイスドライバの問題を解決する。

1. 通常時は移動可能なpage（ページキャッシュ、ユーザーメモリ）に使われる
2. デバイスが必要とするときに連続割り当てのために回収される

```
CMA region:
┌─────────────────────────────────────────────────────────────┐
│  Normal operation: filled with movable pages (page cache)   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ Device requests contiguous buffer
┌─────────────────────────────────────────────────────────────┐
│  Pages migrated out, region available for DMA buffer        │
└─────────────────────────────────────────────────────────────┘
```

CMAはSamsungのMarek Szyprowskiが開発し、カーネル3.5でマージされた（[commit c64be2bb1c6e](https://git.kernel.org/linus/c64be2bb1c6e)、2012年）。設計の議論については[LWNの記事](https://lwn.net/Articles/486301/)を参照。

### CMAを使う {#using-cma}

ドライバからは次のように使う。

```c
#include <linux/dma-mapping.h>

// CMA から割り当てる（DMA API が自動的に処理する）
void *buf = dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL);

// あるいは直接（必要になることはまれ）:
#include <linux/cma.h>
struct page *pages = cma_alloc(cma, count, align, false /* no_warn */);
```

CMA領域はブート時に設定する。

```bash
# CMA 用に 256MB を予約する
cma=256M

# あるいはデバイスツリーで (ARM):
reserved-memory {
    linux,cma {
        compatible = "shared-dma-pool";
        size = <0x10000000>;  /* 256MB */
        reusable;
    };
};
```

## ドライバ開発者のためのベストプラクティス {#best-practices-for-driver-developers}

### 1. DMA APIを使う {#1-use-the-dma-api}

連続メモリを直接割り当てないこと。`dma_alloc_coherent()` または `dma_alloc_attrs()` を使う。

```c
// 良い例: DMA API に任せる
void *buf = dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL);

// 避けるべき例: 連続メモリの直接割り当て
void *buf = kmalloc(size, GFP_KERNEL);  // 大きなサイズでは失敗しうる
```

DMA APIは状況に応じてCMA、IOMMUによるリマッピング、バウンスバッファを使い分けられる。

### 2. 可能ならscatter-gatherを使う {#2-use-scatter-gather-when-possible}

最近の多くのデバイスはscatter-gather DMAをサポートしている。

```c
// 1つの大きな連続バッファの代わりに
struct scatterlist sg[MAX_SEGS];
sg_init_table(sg, nents);

// 散らばった page をマップする
dma_map_sg(dev, sg, nents, direction);
```

これにより連続メモリの要件そのものがなくなる。

### 3. 早期に割り当てるかCMAを使う {#3-allocate-early-or-use-cma}

大きな連続バッファが必要な場合は次のようにする。

- ブート時にメモリを予約する（`memblock` またはCMAで）
- 必要に応じたサイズのCMA領域を使う
- オンデマンドではなく、ドライバのprobe時に割り当てる

### 4. 割り当て失敗を適切に処理する {#4-handle-allocation-failures-gracefully}

高次の割り当ては失敗しうる。大きな割り当てに `__GFP_NOFAIL` を使ってはならない。

```c
// 悪い例: システムをハングさせうる
void *buf = kmalloc(large_size, GFP_KERNEL | __GFP_NOFAIL);

// 良い例: 失敗を処理する
void *buf = kmalloc(large_size, GFP_KERNEL | __GFP_RETRY_MAYFAIL);
if (!buf)
    return -ENOMEM;
```

## 試してみる {#try-it-yourself}

```bash
# order ごとの断片化を見る
cat /proc/buddyinfo
# 例: Node 0, zone Normal  1024  512  256  128  64  32  16  8  4  2  1
#          (order 0-10 それぞれの個数)

# 断片化指数を見る（debugfs が必要。構成によっては存在しない）
cat /sys/kernel/debug/extfrag/extfrag_index
# 0.0 = 断片化なし、1.0 = 深刻な断片化

# CMA 領域を見る
cat /proc/meminfo | grep Cma
# CmaTotal:        262144 kB
# CmaFree:         131072 kB

# 手動でコンパクションを起動する
echo 1 > /proc/sys/vm/compact_memory

# コンパクションの統計を監視する
watch -n 1 'grep -E "compact_" /proc/vmstat'
```

## 変遷 {#evolution}

連続割り当てに対するカーネルのアプローチは大きく進化してきた。

### ZONE_MOVABLE（v2.6.23、2007年） {#zone_movable-v2623-2007}

**コミット**: [2a1e274acf0b](https://git.kernel.org/linus/2a1e274acf0b) ("Create the ZONE_MOVABLE zone")

**作者**: Mel Gorman

*注: このコミットはlore.kernel.orgでの現代的なLKMLアーカイブ以前のものである。*

最初の主要な断片化対策機能である。移動可能な割り当てのみを受け付けるゾーンを作り、このゾーンのメモリが常にマイグレーションまたは回収できることを保証した。`kernelcore=` または `movablecore=` ブートパラメータで設定する。

```bash
# 移動不可能なカーネル割り当て用に 4GB を予約し、残りを ZONE_MOVABLE にする
kernelcore=4G

# あるいは movable ゾーンのサイズを直接指定する
movablecore=8G
```

### マイグレートタイプ（v2.6.24、2007年） {#migrate-types-v2624-2007}

**コミット**: [b2a0ac8875a0](https://git.kernel.org/linus/b2a0ac8875a0) ("Split the free lists for movable and unmovable allocations")

**作者**: Mel Gorman

*注: このコミットはlore.kernel.orgでの現代的なLKMLアーカイブ以前のものである。*

バディアロケータのフリーリストをpageの移動可能性によって分割した。これはZONE_MOVABLEより侵襲性の低い代替策で、ブートパラメータなしで自動的に機能する。2つのアプローチは互いに補完し合う。

### メモリコンパクション（v2.6.35、2010年） {#memory-compaction-v2635-2010}

**コミット**: [748446bb6b5a](https://git.kernel.org/linus/748446bb6b5a) ("mm: compaction: memory compaction core") | [LKML](https://lore.kernel.org/lkml/1269347146-7461-1-git-send-email-mel@csn.ul.ie/)

**作者**: Mel Gorman

pageのマイグレーションによる能動的なデフラグである。それ以前のカーネルは（マイグレートタイプによって）断片化を*防ぐ*ことしかできず、*修復する*ことはできなかった。コンパクションにより、長時間稼働するシステムでTHPが実用的になった。

### CMA（v3.5、2012年） {#cma-v35-2012}

**コミット**: [c64be2bb1c6e](https://git.kernel.org/linus/c64be2bb1c6e) ("drivers: add Contiguous Memory Allocator")

**作者**: Marek Szyprowski (Samsung)

*注: 元のパッチシリーズはlore.kernel.orgにアーカイブされていない。議論については[LWNの記事](https://lwn.net/Articles/486301/)を参照。*

二重の役割を果たす領域を作ることでデバイスドライバの問題を解決した。通常は移動可能なpageに使い、必要に応じて連続DMAバッファのために回収する。[LWNの記事](https://lwn.net/Articles/486301/)を参照。

### プロアクティブコンパクション（v5.9、2020年） {#proactive-compaction-v59-2020}

**コミット**: [facdaa917c4d](https://git.kernel.org/linus/facdaa917c4d) ("mm: proactive compaction") | [LKML](https://lore.kernel.org/linux-mm/20200428221055.598-1-nigupta@nvidia.com/)

**作者**: Nitin Gupta (NVIDIA)

断片化レベルに基づくバックグラウンドのコンパクションで、直接コンパクションによるストールを減らす。`vm.compaction_proactiveness` で制御する。

## 参考文献 {#further-reading}

- [メモリコンパクション](compaction.md) - コンパクションの仕組み
- [ページアロケータ](page-allocator.md) - バディシステムとマイグレートタイプ
- [LWN: CMA](https://lwn.net/Articles/486301/) - CMAの設計と根拠
- [LWN: Memory compaction](https://lwn.net/Articles/368869/) - 最初のコンパクション提案
- [Kernel docs: DMA API](https://docs.kernel.org/core-api/dma-api.html) - DMA APIの正しい使い方
