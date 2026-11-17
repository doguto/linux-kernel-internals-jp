# ページアロケータ（バディシステム） {#page-allocator-buddy-system}

> Linuxのメモリ管理の土台 — 物理ページフレームの割り当て

## ページアロケータとは何か {#what-is-the-page-allocator}

ページアロケータは、物理メモリをページ単位（通常は`4KB`）で管理する。カーネルで最も低レベルのアロケータであり、他のすべてのアロケータ（SLUB、vmalloc）も最終的にはここからページを得る。

```c
struct page *alloc_pages(gfp_t gfp, unsigned int order);
void free_pages(unsigned long addr, unsigned int order);
```

## バディシステム {#the-buddy-system}

Linuxは**バディアロケータ**を使う。これは1965年にまで遡るアルゴリズムである（[Knowlton, CACM](https://dl.acm.org/doi/10.1145/365628.365655)）。核となる考え方は、メモリを2のべき乗サイズのブロックで管理し、メモリを解放するときに、その「バディ（相方）」も空いていれば結合するというものである。

### 仕組み {#how-it-works}

```
Memory divided into blocks of order 0, 1, 2, ... 10
Order 0 = 1 page (4KB)
Order 1 = 2 pages (8KB)
Order 2 = 4 pages (16KB)
...
Order 10 = 1024 pages (4MB) - max order (MAX_ORDER=11, so 0-10 valid)

Free lists per order:
+-------------------------------------------+
| Order 0: [4KB] [4KB] [4KB] ...            |
| Order 1: [8KB] [8KB] ...                  |
| Order 2: [16KB] ...                       |
| ...                                       |
| Order 10: [4MB] ...                       |
+-------------------------------------------+
```

### 割り当ての例 {#allocation-example}

3ページの要求（order 2 = 4ページが必要）:

```
1. Check order 2 free list
   - If available: remove block, return it
   - If empty: go to step 2

2. Check order 3 free list (8 pages)
   - Split: one 4-page block to satisfy request
   - Other 4-page block goes to order 2 free list

3. Continue splitting from higher orders if needed
```

### 解放の例（バディの結合） {#free-example-buddy-merging}

アドレス0x4000にある4ページのブロックを解放する場合（order-2のブロックは4ページ境界にアラインされていなければならない）:

```
1. Find buddy address: 0x4000 XOR (4 * PAGE_SIZE) = 0x0000

2. Is buddy free and same order?
   - Yes: merge into 8-page block, try to merge again
   - No: add to order 2 free list

Merging continues up to MAX_ORDER or until buddy isn't free
```

### なぜバディなのか {#why-buddies}

バディのアドレスは1回のXOR演算で計算でき、探索は不要である。同じサイズの隣接する2つのブロックがバディになるのは、同じ大きなブロックを分割して生まれた場合に限られる。

## メモリゾーン {#memory-zones}

物理メモリは、ハードウェアの制約に基づいてゾーンに分割される。

| ゾーン | 目的 | 典型的な範囲（x86-64） |
|------|---------|------------------------|
| `ZONE_DMA` | レガシーな16ビットDMAデバイス | 0 - 16MB |
| `ZONE_DMA32` | 32ビットDMAデバイス | 16MB - 4GB |
| `ZONE_NORMAL` | 通常のカーネル割り当て | 4GB以上 |
| `ZONE_MOVABLE` | 移動可能なページ（メモリホットプラグ、分離） | 設定可能 |

各ゾーンは独自のバディのフリーリスト群を持つ。

## GFPフラグ {#gfp-flags}

GFP（Get Free Pages）フラグは割り当ての挙動を制御する。

```c
/* よく使われる組み合わせ */
GFP_KERNEL    /* 通常の割り当て。スリープ可、回収可 */
GFP_ATOMIC    /* スリープ不可 (割り込みコンテキスト) */
GFP_USER      /* ユーザー空間向けの割り当て */
GFP_DMA       /* ZONE_DMAから割り当てる必要がある */
GFP_DMA32     /* ZONE_DMA32から割り当てる必要がある */

/* 修飾子 */
__GFP_ZERO    /* メモリをゼロクリアする */
__GFP_NOWARN  /* 割り当て失敗の警告を抑制する */
__GFP_RETRY_MAYFAIL  /* 粘り強く試すが失敗することもある */
__GFP_NOFAIL  /* 決して失敗しない - 控えめに使うこと。デッドロックしうる */
```

全一覧は[`include/linux/gfp.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/gfp.h)を参照。

## 歴史と変遷 {#history-evolution}

### 起源（1991年） {#origins-1991}

バディアロケータはLinuxの最初期から存在する。基本アルゴリズムの出典は次のとおり。

> Knowlton, K. C. "A Fast Storage Allocator" Communications of the ACM, 1965

*注: LKML以前の時代のものである。初期の実装の詳細は、v2.6.12以降のgit履歴にある。*

### per-CPUページリスト（v2.5/2.6） {#per-cpu-page-lists-v2526}

**問題**: SMPシステムでは、すべての割り当て/解放でゾーンロックが必要になり、これがボトルネックになった。

**解決策**: per-CPUページリスト（`per_cpu_pages`）が単一ページをキャッシュし、ロック競合を減らす。

*注: 2.5の開発サイクル中に段階的に開発された。検索可能なgit履歴（v2.6.12+）より前のものである。*

```
CPU 0: [page] [page] [page] ...  (hot cache)
CPU 1: [page] [page] ...
...
        |
        v
   Zone free lists (locked)
```

### NUMA対応（v2.6） {#numa-awareness-v26}

**問題**: NUMAシステムでは、リモートメモリへのアクセスはローカルメモリより遅い。

**解決策**: ノードごとのゾーン。割り当てはローカルノードのメモリを優先する。

```c
/* 特定のノードから割り当てる */
alloc_pages_node(node_id, gfp, order);
```

*注: NUMAサポートは多くのリリースを経て発展してきた。割り当てポリシーについては`mm/mempolicy.c`を参照。*

### コンパクション（v2.6.35、2010年） {#compaction-v2635-2010}

**コミット**: [748446bb6b5a](https://git.kernel.org/linus/748446bb6b5a) ("mm: compaction: memory compaction core") | [LKML](https://lore.kernel.org/lkml/1269347146-7461-1-git-send-email-mel@csn.ul.ie/)

**作者**: Mel Gorman

**問題**: 外部断片化 — 空きページが散らばり、高次の割り当てを満たせない。

**解決策**: メモリコンパクション — ページを移動して連続した空きブロックを作る。

### CMA（Contiguous Memory Allocator、v3.5、2012年） {#cma-contiguous-memory-allocator-v35-2012}

**コミット**: [c64be2bb1c6e](https://git.kernel.org/linus/c64be2bb1c6e) ("drivers: add Contiguous Memory Allocator")

**作者**: Marek Szyprowski (Samsung)

*注: lore.kernel.orgにある2013年以前のLKMLアーカイブはまばらである。設計に関する議論は[LWNの記事](https://lwn.net/Articles/486301/)を参照。*

**問題**: デバイスは大きな連続バッファを必要とするが、メモリは時間とともに断片化する。

**解決策**: 通常は移動可能なページが使えるが、必要になったら連続割り当てのために取り戻せる領域を予約しておく。

## 試してみる {#try-it-yourself}

### バディアロケータの状態を見る {#view-buddy-allocator-state}

```bash
# ゾーンごと、orderごとの空きブロック
cat /proc/buddyinfo

# 出力例:
# Node 0, zone   Normal  1024  512  256  128   64   32   16    8    4    2    1
#                        ^order 0              ^order 5                    ^order 10

# 各数値 = そのorderの空きブロック数
```

### ゾーン情報を見る {#view-zone-information}

```bash
# ゾーンの詳細な統計
cat /proc/zoneinfo

# 主なフィールド:
# - min/low/high: 回収のためのウォーターマーク
# - free: 現在の空きページ数
# - managed: このゾーンが管理するページ数
```

### メモリ逼迫テスト {#memory-pressure-test}

```bash
# メモリを割り当てながらバディの状態を観察
watch -n 1 cat /proc/buddyinfo

# 別の端末でメモリを消費する
stress --vm 1 --vm-bytes 1G
```

### 割り当てのトレース {#trace-allocations}

```bash
# ページ割り当てのトレーシングを有効化 (debugfsが必要)
echo 1 > /sys/kernel/debug/tracing/events/kmem/mm_page_alloc/enable
cat /sys/kernel/debug/tracing/trace_pipe

# 出力にはページアドレス、order、gfpフラグ、migratetypeが表示される
```

## 主要なデータ構造 {#key-data-structures}

### struct zone {#struct-zone}

```c
struct zone {
    unsigned long watermark[NR_WMARK];     /* min/low/highのしきい値 */
    unsigned long nr_reserved_highatomic;
    struct per_cpu_pages __percpu *per_cpu_pageset;
    struct free_area free_area[NR_PAGE_ORDERS];  /* バディのフリーリスト */
    unsigned long zone_start_pfn;
    unsigned long managed_pages;
    unsigned long spanned_pages;
    unsigned long present_pages;
    const char *name;
    /* ... */
};
```

*注: v6.8で`NR_PAGE_ORDERS`が[`MAX_ORDER`を置き換えた](https://git.kernel.org/linus/fd37721803c6)。古いコードを読むときは、`MAX_ORDER`が同じ概念である。*

### ウォーターマーク {#watermarks}

各ゾーンには、回収の挙動を制御するウォーターマークがある。

| ウォーターマーク | 到達したとき |
|-----------|--------------|
| `high` | ゾーンは健全で、何もする必要はない |
| `low` | `kswapd`を起こしてバックグラウンドでページを回収する |
| `min` | 直接回収 — 割り当てを行うプロセス自身がページの解放を手伝う必要がある |
| `promo` | 階層型メモリ（CXL/NUMA）向けで、ページの昇格（promotion）を制御する |

ウォーターマークは`vm.min_free_kbytes`から導出される。

```bash
# ウォーターマークを見る
cat /proc/zoneinfo | grep -E "pages free|min|low|high"

# 最小空きメモリを調整する (すべてのウォーターマークに影響)
sysctl vm.min_free_kbytes=65536
```

**ゾーンのフォールバック**: あるゾーンが枯渇すると、割り当ては NORMAL → DMA32 → DMA の順に他のゾーンへフォールバックする（上位のゾーンが優先される）。

### struct free_area {#struct-free_area}

```c
struct free_area {
    struct list_head free_list[MIGRATE_TYPES];  /* migratetypeごとのリスト */
    unsigned long nr_free;
};
```

### マイグレートタイプ {#migrate-types}

断片化を減らすため、ページは移動可能性によってグループ化される。

| タイプ | 説明 |
|------|-------------|
| `MIGRATE_UNMOVABLE` | 移動できないカーネルの割り当て |
| `MIGRATE_MOVABLE` | ユーザーページ。マイグレーション/コンパクションが可能 |
| `MIGRATE_RECLAIMABLE` | 逼迫時に解放できるキャッシュ |
| `MIGRATE_HIGHATOMIC` | 優先度の高いアトミック割り当てのために予約 |
| `MIGRATE_CMA` | Contiguous Memory Allocatorの領域 |
| `MIGRATE_ISOLATE` | マイグレーション/オフライン化のために分離中のページ |

```bash
# マイグレートタイプの分布を見る
cat /proc/pagetypeinfo
```

## よくある問題 {#common-issues}

### 高次割り当ての失敗 {#high-order-allocation-failures}

大きな連続割り当て（order > 3）は、空きメモリがあっても断片化のために失敗することがある。

**対策**:

- `__GFP_RETRY_MAYFAIL`を使ってより粘り強く試す
- 代わりにvmallocを使う（仮想的な連続性）
- コンパクションを有効にする（`/proc/sys/vm/compact_memory`）

### ゾーンの不均衡 {#zone-imbalance}

他のゾーンに空きメモリがあるのに、あるゾーンだけメモリが逼迫している。

**確認**: `cat /proc/zoneinfo | grep -E "zone|free|min"`

### 空きページがあるのにOOM {#oom-despite-free-pages}

空きページは存在するが、ゾーンやmigratetypeが合っていない。

**デバッグ**: `/proc/buddyinfo`と`/proc/pagetypeinfo`を確認する

## リファレンス {#references}

### 主要なコード {#key-code}

| ファイル | 説明 |
|------|-------------|
| [`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) | アロケータのコア |
| [`include/linux/gfp.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/gfp.h) | GFPフラグ |
| [`include/linux/mmzone.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mmzone.h) | ゾーンの構造体 |

### 主要なコミット {#key-commits}

| コミット | カーネル | 説明 |
|--------|--------|-------------|
| [748446bb6b5a](https://git.kernel.org/linus/748446bb6b5a) | v2.6.35 | メモリコンパクション |
| [c64be2bb1c6e](https://git.kernel.org/linus/c64be2bb1c6e) | v3.5 | CMAの導入 |

### 関連 {#related}

- [overview](overview.md) - ページアロケータが他のアロケータとどう組み合わさるか
- [slab](slab.md) - SLUBはバディアロケータからページを得る
- [vmalloc](vmalloc.md) - バディから個々のページを得る

## 参考文献 {#further-reading}

- [slab.md](slab.md) — SLUBはページ未満の割り当てのために、バディアロケータのすぐ上に位置する
- [compaction.md](compaction.md) — 高次の要求を満たすためにカーネルが物理メモリをデフラグする方法
- [memblock.md](memblock.md) — 起動時にメモリをバディシステムへ引き渡すブート時アロケータ
- [thp.md](thp.md) — Transparent huge pages：高次（order-9）のバディ割り当ての主な利用者
- [numa.md](numa.md) — ノードごとのゾーンの上に構築されたNUMA対応の割り当てポリシー
- [mm/page_alloc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) — バディアロケータのコア。`__alloc_pages()`が主な入り口
- [include/linux/gfp.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/gfp.h) — GFPフラグの全定義とゾーン選択のロジック
- [include/linux/mmzone.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mmzone.h) — `struct zone`、`struct free_area`、ウォーターマークの定義
- [LWN: Memory compaction](https://lwn.net/Articles/368869/) — Mel Gormanによる断片化とコンパクションという解決策の解説
- [LWN: Grouping pages by mobility](https://lwn.net/Articles/224254/) — 恒久的な断片化を減らすためにマイグレートタイプがどのように導入されたか
- `Documentation/admin-guide/mm/concepts.rst` — ゾーン、ウォーターマーク、バディシステムを扱うカーネルドキュメント
