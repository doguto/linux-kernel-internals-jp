# zone回収ポリシー {#zone-reclaim-policy}

> カーネルがzone単位でメモリを回収する仕組み、`vm.zone_reclaim_mode`の役割、そしてそれがデフォルトで無効になっている理由

## 主要なソースファイル {#key-source-files}

| ファイル | 関連する内容 |
|------|-----------|
| `mm/vmscan.c` | `node_reclaim()`、`balance_pgdat()`、`node_reclaim_mode` |
| `mm/page_alloc.c` | `get_page_from_freelist()`、`zone_watermark_fast()`、`zone_allows_reclaim()`、`boost_watermark()` |
| `include/linux/mmzone.h` | `struct zone`、`enum zone_watermarks`、`enum zone_type`、ウォーターマークのアクセサ |
| `include/uapi/linux/mempolicy.h` | `RECLAIM_ZONE`、`RECLAIM_WRITE`、`RECLAIM_UNMAP`のビット定義 |
| `include/linux/topology.h` | `RECLAIM_DISTANCE`のデフォルト値 |

---

## メモリzone {#memory-zones}

Linuxは物理メモリをzoneに分割し、アドレッシング上の制約が異なる割り当てを無駄なく満たせるようにしている。zoneの種類は`include/linux/mmzone.h`の`enum zone_type`で定義されている。

```c
enum zone_type {
#ifdef CONFIG_ZONE_DMA
    ZONE_DMA,       /* 先頭 約16MB: ISA/レガシー DMA デバイス */
#endif
#ifdef CONFIG_ZONE_DMA32
    ZONE_DMA32,     /* 先頭 4GB: 32 ビット DMA デバイス */
#endif
    ZONE_NORMAL,    /* 残りのダイレクトマップされた物理メモリ */
#ifdef CONFIG_HIGHMEM
    ZONE_HIGHMEM,   /* 32 ビットアーキテクチャで ZONE_NORMAL より上 */
#endif
    ZONE_MOVABLE,   /* movable な page のみ。ホットプラグ/THP を助ける */
#ifdef CONFIG_ZONE_DEVICE
    ZONE_DEVICE,    /* 永続メモリ/デバイスメモリ */
#endif
    __MAX_NR_ZONES
};
```

zoneは階層を成す。`ZONE_DMA`のメモリを必要とするアロケータは`ZONE_DMA`しか使えない。`ZONE_NORMAL`のメモリを要求するアロケータは、必要に応じて`ZONE_DMA32`や`ZONE_DMA`にフォールバックできるが、その逆はできない。各zoneは独立しており、固有のフリーリスト、固有のウォーターマーク、固有の回収圧力を持つ。

```
Physical address space (typical 64-bit x86):

0         16MB          4GB                          RAM top
|─ZONE_DMA─|───ZONE_DMA32───|────────ZONE_NORMAL────────────|
  GFP_DMA     GFP_DMA32          GFP_KERNEL (default)
```

---

## zoneのウォーターマーク {#zone-watermarks}

すべてのzoneは、回収がいつ始まるかを制御する3つのウォーターマークを管理している。これらは`struct zone._watermark[]`に格納され、インラインヘルパーを通してアクセスされる。配列を直接読んではならない。

### `enum zone_watermarks`列挙型 {#the-enum-zone_watermarks-enum}

```c
/* include/linux/mmzone.h */
enum zone_watermarks {
    WMARK_MIN,
    WMARK_LOW,
    WMARK_HIGH,
    WMARK_PROMO,   /* メモリ階層化 / NUMA バランシングで使われる */
    NR_WMARK
};
```

主要な3つのレベル:

| ウォーターマーク | 意味 |
|-----------|---------|
| `WMARK_MIN` | 最後の砦。空きpageがこのレベルを上回るまで、直接回収が割り当て中のプロセスをストールさせる。 |
| `WMARK_LOW` | バックグラウンドの閾値。これを下回ると`kswapd`が起床する。 |
| `WMARK_HIGH` | 目標値。`kswapd`は空きpageがこのレベルに達するまで回収し、その後スリープする。 |

### ウォーターマークのアクセサマクロ {#watermark-accessor-macros}

`struct zone`は生のウォーターマーク値を`_watermark[]`に格納する。一時的なブースト（[ウォーターマークブースト](#watermark-boosting)を参照）を実現するため、読み出し時に`watermark_boost`フィールドの値が上乗せされる。必ず次のアクセサを使うこと。

```c
/* include/linux/mmzone.h */

struct zone {
    unsigned long _watermark[NR_WMARK];
    unsigned long watermark_boost;
    /* ... */
};

static inline unsigned long wmark_pages(const struct zone *z,
                                        enum zone_watermarks w)
{
    return z->_watermark[w] + z->watermark_boost;
}

static inline unsigned long min_wmark_pages(const struct zone *z)  { return wmark_pages(z, WMARK_MIN); }
static inline unsigned long low_wmark_pages(const struct zone *z)  { return wmark_pages(z, WMARK_LOW); }
static inline unsigned long high_wmark_pages(const struct zone *z) { return wmark_pages(z, WMARK_HIGH); }
```

ウォーターマークの値は`vm.min_free_kbytes`から導出される。このsysctlを引き上げると、すべてのzoneの3つのレベルすべてが比例して上がる。

### ウォーターマークの確認: `zone_watermark_fast()`と`zone_watermark_ok()` {#checking-watermarks-zone_watermark_fast-and-zone_watermark_ok}

アロケータはホットパスで`zone_watermark_fast()`を呼び、より低速な完全チェックには`zone_watermark_ok()`にフォールバックする。どちらも`mm/page_alloc.c`にある。

```c
bool zone_watermark_ok(struct zone *z, unsigned int order,
                       unsigned long mark,
                       int highest_zoneidx, unsigned int alloc_flags);

static inline bool zone_watermark_fast(struct zone *z, unsigned int order,
                                       unsigned long mark, int highest_zoneidx,
                                       unsigned int alloc_flags, gfp_t gfp_mask);
```

`zone_watermark_fast()`はまず高速パスを試す。order-Nを考慮せずに空きpageが`mark + lowmem_reserve[highest_zoneidx]`を超えていれば、ただちに`true`を返す。そうでなければ`__zone_watermark_ok()`を呼び、フリーリストの全orderにわたる完全な集計を行う。

---

## zone単位のkswapd: `balance_pgdat()` {#per-zone-kswapd-balance_pgdat}

`kswapd`はノード単位（`pgdat`単位）のカーネルスレッドだが、回収はzoneごとに行う。中核となるループは`mm/vmscan.c`の`balance_pgdat()`である。

```c
/*
 * kswapd は highmem->normal->dma の方向に zone を走査する。
 * free_pages > high_wmark_pages(zone) である zone はスキップするが、
 * free_pages <= high_wmark_pages(zone) である zone が見つかると、
 * 少なくとも 1 つの使用可能な zone がバランスするまで、その zone
 * またはそれより下位の zone にあるあらゆる page が回収対象となる。
 */
static int balance_pgdat(pg_data_t *pgdat, int order, int highest_zoneidx)
```

方向が重要である。kswapdは**最も高いzoneインデックスから下に向かって**走査する（例: `ZONE_MOVABLE` → `ZONE_NORMAL` → `ZONE_DMA32` → `ZONE_DMA`）。`WMARK_HIGH`を下回るzoneを見つけると、そのzoneとそれより下位のすべてのzoneが、同じパスで回収の候補となる。きっかけとなった割り当てに使える少なくとも1つのzoneが`WMARK_HIGH`を上回ってバランスを取り戻すまで、処理は続く。

```
kswapd zone scan order (highest → lowest):

  ZONE_MOVABLE  ← scanned first
  ZONE_NORMAL
  ZONE_DMA32
  ZONE_DMA      ← scanned last

If ZONE_NORMAL falls below WMARK_HIGH, pages are reclaimed from
ZONE_NORMAL and ZONE_DMA32 and ZONE_DMA in the same pass.
```

`kswapd`はウォーターマークブーストも考慮する。`balance_pgdat()`の呼び出しの冒頭で、すべてのzoneについて`zone->watermark_boost`のスナップショットをローカルの`zone_boosts[]`配列に取る。shrink_nodeの各イテレーションの後、スナップショットした量だけ`zone->watermark_boost`を減らし、回収が進むにつれて徐々にブーストを解消していく。

---

## `node_reclaim()`と`vm.zone_reclaim_mode` {#node_reclaim-and-vmzone_reclaim_mode}

### sysctl {#the-sysctl}

`vm.zone_reclaim_mode`は、カーネル変数`node_reclaim_mode`（`mm/vmscan.c`で宣言）のユーザーから見える名前である。

```c
/* mm/vmscan.c */
int node_reclaim_mode __read_mostly;  /* デフォルト: 0（無効） */
```

そのビットは`include/uapi/linux/mempolicy.h`で定義されている。

```c
#define RECLAIM_ZONE  (1<<0)  /* zone 回収を有効にする */
#define RECLAIM_WRITE (1<<1)  /* 回収中に page を書き出す */
#define RECLAIM_UNMAP (1<<2)  /* 回収中に page をアンマップする */
```

ビットは加算的である。

| 値 | 効果 |
|-------|--------|
| `0` | zone回収は無効（デフォルト） |
| `1` | ローカルノードからクリーンなファイルpageを回収する |
| `2` | + 回収中にダーティpageのライトバックを許可する |
| `4` | + マップされたファイルpageのアンマップを許可する |
| `7` | すべての種類の回収が有効 |

いずれかのビットが設定されていれば、`node_reclaim_enabled()`はtrueを返す。

```c
/* mm/internal.h */
static inline bool node_reclaim_enabled(void)
{
    return node_reclaim_mode & (RECLAIM_ZONE|RECLAIM_WRITE|RECLAIM_UNMAP);
}
```

### `get_page_from_freelist()`内の呼び出し箇所 {#the-call-site-in-get_page_from_freelist}

フックは`mm/page_alloc.c`の`get_page_from_freelist()`内にあり、これが主要な割り当てパスである。あるzoneがウォーターマークのチェックに失敗すると、アロケータはノード回収を試す価値があるかどうかを確認する。

```c
/* mm/page_alloc.c — get_page_from_freelist() の内部 */
if (!node_reclaim_enabled() ||
    !zone_allows_reclaim(zonelist_zone(ac->preferred_zoneref), zone))
    continue;

ret = node_reclaim(zone->zone_pgdat, gfp_mask, order);
switch (ret) {
case NODE_RECLAIM_NOSCAN:
    /* 走査しなかった */
    continue;
case NODE_RECLAIM_FULL:
    /* 走査したが回収できなかった */
    continue;
default:
    /* 十分に回収できたか? */
    if (zone_watermark_ok(zone, order, mark,
            ac->highest_zoneidx, alloc_flags))
        goto try_this_zone;
    continue;
}
```

回収が成功し、zoneがウォーターマークを上回るようになれば、割り当てはそのzoneから行われる。そうでなければ、アロケータはzonelist内の次のzoneに進む。

### `node_reclaim()`の内部 {#inside-node_reclaim}

`mm/vmscan.c`の`node_reclaim()`は、同期的なノード単位の回収パスである。アロケータから直接呼ばれ、割り当て中のプロセスをブロックする。

```c
int node_reclaim(struct pglist_data *pgdat, gfp_t gfp_mask, unsigned int order)
{
    const unsigned long nr_pages = 1 << order;
    struct scan_control sc = {
        .nr_to_reclaim = max(nr_pages, SWAP_CLUSTER_MAX),
        .gfp_mask      = current_gfp_context(gfp_mask),
        .order         = order,
        .priority      = NODE_RECLAIM_PRIORITY,  /* = 4: zone の 1/16 を走査 */
        .may_writepage = !!(node_reclaim_mode & RECLAIM_WRITE),
        .may_unmap     = !!(node_reclaim_mode & RECLAIM_UNMAP),
        .may_swap      = 1,
        .reclaim_idx   = gfp_zone(gfp_mask),
    };
    /* ... */
}
```

走査を始める前に、呼び出しを打ち切る2つの条件を確認する。

1. **回収の閾値**: `node_pagecache_reclaimable(pgdat) <= pgdat->min_unmapped_pages`*かつ*回収可能なslabが`pgdat->min_slab_pages`未満であれば、ただちに`NODE_RECLAIM_FULL`を返す（回収可能な量が少なすぎて試す価値がない）。
2. **CPUの局所性**: ノードにCPUが接続されていて、それが呼び出し元CPUのノードでなければ、`NODE_RECLAIM_NOSCAN`を返す。これにより、node_reclaimはローカルノードか、CPUを持たない（メモリのみの）ノードでしか実行されない。

成功すると`PGSCAN_ZONE_RECLAIM_SUCCESS`を、失敗すると`PGSCAN_ZONE_RECLAIM_FAILED`をインクリメントする。どちらのカウンタも`/proc/vmstat`で`zone_reclaim_success`および`zone_reclaim_failed`として見ることができる。

---

## なぜ`zone_reclaim_mode`はデフォルトで無効なのか {#why-zone_reclaim_mode-is-disabled-by-default}

`zone_reclaim_mode=1`のとき、カーネルはリモートのNUMAノードからの割り当てにフォールバックする前に、ローカルメモリからpageを回収することを優先する。その意図はNUMAの局所性を保つことだったが、実際には重大な問題を引き起こす。

**`node_reclaim()`は割り当て中のプロセスをストールさせる。** 割り当てのホットパスで同期的に実行されるため、プロセスは走査と回収のサイクルが続く間ブロックする。それなりのメモリ圧力がかかったシステムでは、これがアプリケーションにレイテンシスパイクをもたらす。zonelistのフォールバックを通じてリモートノードから割り当てるほうが、ローカルの回収を待つよりも通常はるかに速い。

トレードオフは次のように整理できる。

| シナリオ | 望ましい動作 |
|----------|--------------------|
| リモートノードへのアクセスによるレイテンシのオーバーヘッドが2倍未満 | `zone_reclaim_mode=0` — リモートから割り当てる |
| アプリケーションがNUMA局所性を厳格に求め、大きなワーキングセットがノードごとに収まる | `zone_reclaim_mode=1`が役立つ可能性がある |
| ストリーミングやI/O中心のワークロード | `zone_reclaim_mode=0` — ページキャッシュの入れ替わりにより、ローカルの回収は逆効果になる |
| ノードごとに固定された大きなバッファプールを持つデータベース | `zone_reclaim_mode=1`によりノード間のトラフィックを減らせる |

`mm/vmscan.c`のカーネルコメントははっきりとこう述べている。

> If non-zero call node_reclaim when the number of free pages falls below the watermarks.

（訳: 非ゼロの場合、空きpageの数がウォーターマークを下回ったときにnode_reclaimを呼ぶ。）

この「非ゼロの場合」という表現は、「ゼロ」が安全なデフォルトであることを示唆している。現代のサーバーワークロードでは、ほぼ例外なくこれを`0`のままにしている。

---

## `RECLAIM_DISTANCE`と`zone_allows_reclaim()` {#reclaim_distance-and-zone_allows_reclaim}

`zone_reclaim_mode`が非ゼロであっても、カーネルは要求元ノードとトポロジ的に近いzoneに対してしか`node_reclaim()`を呼ばない。その閾値が、`include/linux/topology.h`で定義された`RECLAIM_DISTANCE`である。

```c
#ifndef RECLAIM_DISTANCE
/*
 * システム内のノード間の距離が RECLAIM_DISTANCE（node_distance() が返す
 * アーキテクチャ固有の単位）より大きく、かつ node_reclaim_mode が有効な場合、
 * VM はこの距離内にあるノードに対してのみ node_reclaim() を呼ぶ。
 */
#define RECLAIM_DISTANCE 30
#endif

extern int __read_mostly node_reclaim_distance;
```

実行時の変数`node_reclaim_distance`は`RECLAIM_DISTANCE`（30）で始まり、プラットフォームのコードで上書きできる。たとえばAMD EPYCマシンではこの値を引き上げている。2ホップ距離（ACPI SLIT距離32）でのノード間メモリアクセスでも、回収のコストと比較すれば性能が向上するためである。

チェックは`mm/page_alloc.c`の`zone_allows_reclaim()`で行われる。

```c
#ifdef CONFIG_NUMA
int __read_mostly node_reclaim_distance = RECLAIM_DISTANCE;

static bool zone_allows_reclaim(struct zone *local_zone, struct zone *zone)
{
    return node_distance(zone_to_nid(local_zone), zone_to_nid(zone)) <=
                node_reclaim_distance;
}
```

ACPI SLITの距離値は、`LOCAL_DISTANCE=10`（同一ノード）、`REMOTE_DISTANCE=20`（1ホップ）である。`30`という値は次のことを意味する。ローカル（距離10）では回収するが、候補zoneが1ホップ先なら回収しない（距離20 ≤ 30 — したがって、これは1ホップ先のノードでの回収を許可する）。2ホップを超えるzone（距離 > 30）は回収をスキップし、リモートから直接割り当てる。

```
node_distance(local, candidate) <= node_reclaim_distance?
  Yes → node_reclaim() may run
  No  → skip reclaim, allocate from remote zone directly
```

---

## DMA zoneへの圧力 {#dma-zone-pressure}

`ZONE_DMA`（先頭約16MB）と`ZONE_DMA32`（先頭4GB）は、システムのRAM全体に比べて小さい。これらはデバイス固有の割り当て（`GFP_DMA`、`GFP_DMA32`）で埋まり、簡単には回収できない。

### なぜDMA zoneが枯渇するか {#why-dma-zones-deplete}

- デバイスドライバは、バウンスバッファ、ディスクリプタリング、ファームウェア領域を`GFP_DMA`や`GFP_DMA32`で割り当てる。
- これらの割り当ては多くの場合長寿命である（デバイスが存在する間ずっと保持される）。
- カーネルはDMA割り当てを他のzoneへ自動的にマイグレーションしない。

### フォールバックとOOMの挙動 {#fallback-and-oom-behavior}

- `GFP_KERNEL`の割り当て（通常メモリ）は、`ZONE_NORMAL`が枯渇すれば`ZONE_DMA32`や`ZONE_DMA`にフォールバックできるが、その逆は成り立たない。
- `ZONE_DMA`に空きpageがない状態での`GFP_DMA`割り当ては、たとえ`ZONE_NORMAL`にギガバイト単位の空きがあってもOOM killerを起動する。`ZONE_NORMAL`のメモリにはデバイスが到達できないからである。

```
GFP_KERNEL allocation path (zonelist fallback order):
  ZONE_NORMAL → ZONE_DMA32 → ZONE_DMA
  (can fall down, never up)

GFP_DMA allocation path:
  ZONE_DMA only — OOM if exhausted
```

### DMA zoneへの圧力の検出 {#detecting-dma-zone-pressure}

```bash
# zone ごとの空き page とウォーターマークを比較する
grep -A 10 "zone" /proc/zoneinfo | grep -E "Node|zone|free|min|low|high"

# 健全な DMA zone の例:
# Node 0, zone      DMA
#   pages free     2652
#         min      33
#         low      41
#         high     49
```

問題の兆候: `DMA`または`DMA32`の`free` pageが`min`に近づき、それと同時に`dmesg`に`DMA-zone allocation failure`として割り当て失敗が記録される。

---

## ウォーターマークブースト {#watermark-boosting}

### 解決する問題 {#the-problem-it-solves}

高orderの割り当て（例: THP）が下位のmigratetypeにフォールバックすると（例: `MIGRATE_UNMOVABLE`が`MIGRATE_MOVABLE`から奪う）、低orderのフリーリストが汚染される。何も手を打たなければ、以後の高order割り当ては失敗し続け、kswapdも決して起床しない。zoneは内部的に断片化していても、通常のウォーターマークは上回っている可能性があるからである。

### 仕組み {#how-it-works}

`mm/page_alloc.c`の`boost_watermark()`は、`try_to_claim_block()`が`pageblock_order`未満のorderでmigratetypeのフォールバックを検出したときに呼ばれる。これは一時的に`zone->watermark_boost`を引き上げる。

```c
static inline bool boost_watermark(struct zone *zone)
{
    unsigned long max_boost;

    if (!watermark_boost_factor)
        return false;
    /* ブーストすると OOM を引き起こすような小さな zone はスキップする */
    if ((pageblock_nr_pages * 4) > zone_managed_pages(zone))
        return false;

    max_boost = mult_frac(zone->_watermark[WMARK_HIGH],
                          watermark_boost_factor, 10000);
    if (!max_boost)
        return false;

    max_boost = max(pageblock_nr_pages, max_boost);

    zone->watermark_boost = min(zone->watermark_boost + pageblock_nr_pages,
                                max_boost);
    return true;
}
```

`watermark_boost_factor`のデフォルトは`15000`（つまり`WMARK_HIGH`の150%）である。ブーストはすべての`wmark_pages()`呼び出しに上乗せされ、zoneを実際よりも空きが少ないように見せる。これにより、zoneが名目上`WMARK_HIGH`を上回っていてもkswapdが起動される。

ブースト後、zoneのフラグに`ZONE_BOOSTED_WATERMARK`が設定される。次の`rmqueue()`の戻りパスでこのフラグがチェックされ、kswapdが起床させられる。

```c
/* mm/page_alloc.c */
if ((alloc_flags & ALLOC_KSWAPD) &&
    unlikely(test_bit(ZONE_BOOSTED_WATERMARK, &zone->flags))) {
    clear_bit(ZONE_BOOSTED_WATERMARK, &zone->flags);
    wakeup_kswapd(zone, 0, 0, zone_idx(zone));
}
```

その後、`balance_pgdat()`はブーストを段階的に解消する。各イテレーションで、ブーストがなくなりzoneが本当にバランスするまで、最初の`zone_boosts[i]`スナップショットを`zone->watermark_boost`から差し引く。

### ブーストの挙動の制御 {#controlling-boost-behavior}

```bash
# デフォルト: 15000（WMARK_HIGH の 150%）
cat /proc/sys/vm/watermark_boost_factor

# ブーストを完全に無効にする
echo 0 > /proc/sys/vm/watermark_boost_factor
```

THPのフォールバックが頻繁に起き、その結果のkswapdの起床が望ましくない回収圧力を生むシステムでは、`0`に設定する。

---

## zoneの状態の観測 {#observing-zone-state}

### `/proc/zoneinfo` {#proczoneinfo}

zone単位の標準的な診断手段である。主要なフィールド:

```
Node 0, zone   Normal
  pages free     524288
        boost    0         ← current watermark_boost
        min      8192
        low      10240
        high     12288
        spanned  2097152
        present  2097152
        managed  1998765
```

`free`が`low`を下回っているか、`min`に近づいているzoneがないか確認する。

### `/proc/vmstat` — zone回収のカウンタ {#procvmstat-zone-reclaim-counters}

これらのカウンタは`CONFIG_NUMA=y`の場合にのみ存在する。

```bash
grep zone_reclaim /proc/vmstat
# zone_reclaim_success  42      ← node_reclaim() が十分な page を解放した
# zone_reclaim_failed   1831    ← node_reclaim() は実行されたが要求を満たせなかった
```

`zone_reclaim_mode`が有効な状態で`zone_reclaim_failed`のカウントが多い場合、ワークロードのホットなpageをローカルで回収できず、`zone_reclaim_mode`を無効にすべきだという強いシグナルである。

### 簡易診断の手順 {#quick-diagnostic-workflow}

```bash
# 1. zone のウォーターマークと空き page を確認する
grep -A 15 "^Node" /proc/zoneinfo | grep -E "Node|zone|free|boost|min|low|high"

# 2. zone 回収が有効かどうかを確認する
cat /proc/sys/vm/zone_reclaim_mode

# 3. 回収の成功/失敗の比率を確認する（NUMA のみ）
grep -E "zone_reclaim|pgscan" /proc/vmstat

# 4. DMA zone の枯渇を確認する
grep -A 10 "zone.*DMA" /proc/zoneinfo | grep free

# 5. ウォーターマークブーストによる kswapd の活動を監視する
grep -E "pageoutrun|kswapd" /proc/vmstat
```

### チューニングの指針 {#tuning-guidance}

| 症状 | 考えられる原因 | 対処 |
|---------|-------------|--------|
| `zone_reclaim_failed` >> `zone_reclaim_success` | ワーキングセットがローカルノードに収まらない | `echo 0 > /proc/sys/vm/zone_reclaim_mode` |
| NUMAシステムでのレイテンシスパイク | `zone_reclaim_mode`がアロケータをストールさせている | zone回収モードを無効にする |
| 他のzoneに空きメモリがあるのにOOM | DMA zoneの枯渇 | `GFP_DMA`/`GFP_DMA32`の消費者を減らす。ドライバのメモリ使用量を確認する |
| メモリ圧力がないのにkswapdが動いている | ウォーターマークブーストが有効 | `/proc/zoneinfo`の`boost`フィールドを確認する。`watermark_boost_factor`を下げることを検討する |
| 高orderの割り当て失敗が続く | zoneの断片化 | `/proc/buddyinfo`を調べる。`vm.compaction_proactiveness`を検討する |

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [mm/vmscan.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) — `node_reclaim()`、`balance_pgdat()`、`kswapd()`、および`node_reclaim_mode`変数
- [mm/page_alloc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) — `get_page_from_freelist()`、`zone_watermark_fast()`、`zone_watermark_ok()`、`zone_allows_reclaim()`、`boost_watermark()`
- [include/linux/mmzone.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mmzone.h) — `struct zone`のウォーターマーク関連フィールド、`enum zone_watermarks`、`enum zone_type`、ウォーターマークのアクセサインライン関数
- [include/uapi/linux/mempolicy.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/mempolicy.h) — `RECLAIM_ZONE`、`RECLAIM_WRITE`、`RECLAIM_UNMAP`のビット定義

### カーネルドキュメント {#kernel-documentation}

- `Documentation/admin-guide/sysctl/vm.rst` — `zone_reclaim_mode`、`min_free_kbytes`、`watermark_boost_factor`、`watermark_scale_factor`（[レンダリング版](https://docs.kernel.org/admin-guide/sysctl/vm.html)）
- `/proc/sys/vm/zone_reclaim_mode` — 実行時の切り替え。`0`で無効、`1`/`2`/`4`で段階的により積極的な回収が有効になる

### LWNの記事 {#lwn-articles}

- [Zone reclaim mode considered harmful](https://lwn.net/Articles/432224/) — `zone_reclaim_mode=1`がほとんどのワークロードでスループットを損なう理由と、デフォルトが無効である理由の分析
- [Watermark boosting](https://lwn.net/Articles/750500/) — `watermark_boost_factor`の動機と、断片化に起因するkswapdの活動不足をそれがどう緩和するか
- [DMA zone management](https://lwn.net/Articles/258803/) — DMA zoneが小さい理由、それがどう枯渇するか、zone境界をまたぐフォールバックを妨げる制約

### 関連ドキュメント {#related-docs}

- [メモリ回収に対するNUMAの影響](numa-reclaim.md) — ノード単位のkswapdスレッド、NUMAバランシングと回収の相互作用、NUMAの文脈における`zone_reclaim_mode`
- [NUMAのzonelist構築とフォールバック順序](numa-zonelist.md) — `node_reclaim()`が呼ばれる前にどのzoneが試されるかを、アロケータのフォールバックリストがどう決めるか
- [回収](reclaim.md) — 回収の仕組みの全体: LRUリスト、`shrink_node()`、直接回収、`try_to_free_pages()`
- [コンパクション](compaction.md) — ウォーターマークブーストが表面化させようとする高order割り当ての失敗に、`kcompactd`とプロアクティブなコンパクションがどう対処するか
