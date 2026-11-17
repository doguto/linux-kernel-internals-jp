# MGLRU: Multi-Generation LRU {#mglru-multi-generation-lru}

> ワーキングセットをより正確に検出するための、世代別ページ回収アルゴリズム

## 従来のLRUの問題点 {#the-problem-with-classic-lru}

従来のLRU（Least Recently Used）によるページ回収は、メモリゾーンごとに **active** と **inactive** の2つのリストを使う。pageはアクセス状況に応じて両リスト間を移動する。根本的な問題は次のとおりである。

1. **スキャンコスト**: 回収可能なpageを見つけるため、回収処理はinactiveリストを走査する。inactiveリストがホットなpage（偽陰性）で埋まっていると、CPU時間を浪費する。
2. **一度きりのアクセスによる汚染**: 大きな `dd` や `tar` の処理はすべてのファイルを一度だけ読むため、activeリストがコールドなpageでかき回される。
3. **時間的局所性がない**: 従来のLRUは、1秒前にアクセスされたpageと1年前にアクセスされたpageを同じように扱う。

## MGLRUの世代モデル {#mglrus-generational-model}

MGLRU（Multi-Generation LRU、6.1でマージ）はpageを **世代** に分割する。各世代は、ほぼ同じ時期にホットになったpageを表す。最も古い世代から先に追い出される。

```
Generation 0 (oldest): pages not accessed since earliest timestamp
Generation 1:          pages accessed in the second-oldest period
...
Generation N (newest): recently accessed pages

Reclaim: evict from generation 0 first → if empty, evict generation 1 → ...
Age: promote pages to the newest generation when accessed
```

典型的な世代数は4である。各世代はおよそ数秒から数分のアクティビティに相当する。

## 主要なデータ構造 {#key-data-structures}

```c
/* mm/vmscan.c (6.1+) */

/* ノードごと・種別（anon/file）ごとのLRU状態 */
struct lru_gen_folio {
    /* 最小/最大の世代カウンタ */
    unsigned long max_seq;          /* 最新の世代 */
    unsigned long min_seq[ANON_AND_FILE]; /* 空でない最古の世代 */

    /* 世代ごとのページ数 */
    long          nr_pages[MAX_NR_GENS][ANON_AND_FILE][MAX_NR_ZONES];

    /* refault検出を高速化するためのBloomフィルタ */
    unsigned long *filters[NR_BLOOM_FILTERS];
    atomic_long_t  nr_evicted[ANON_AND_FILE];
    atomic_long_t  nr_refaulted[ANON_AND_FILE];
    atomic_long_t  protected[NR_HIST_GENS][ANON_AND_FILE][NR_BIRTH_MARKS];
};

/* folioごとの世代情報はfolioフラグに埋め込まれている */
/* gen = folio_lru_gen(folio): このfolioがどの世代に属するか */
```

## エージング: ホットなpageの走査 {#aging-scanning-for-hot-pages}

**エージング**はページテーブルを走査して最近アクセスされたpageを見つけ、それらを最新の世代に昇格させる。

```c
/* mm/vmscan.c */
static bool walk_mm(struct lruvec *lruvec, struct mm_struct *mm,
                     struct lru_gen_walk_control *lwc)
{
    struct lru_gen_mm_walk walk = {
        .lruvec = lruvec,
        .seq    = min_seq(lruvec, ANON_AND_FILE),
        /* ... */
    };

    /* すべてのVMAを走査し、PTEのaccessedビットを確認する */
    walk_page_range(mm, 0, ULONG_MAX, &lru_gen_mm_walk_ops, &walk);
    return walk.force_scan;
}
```

PTEのaccessedビットは、前回のエージングパス以降にどのpageが触られたかをカーネルに伝える。accessedビットが立っているpageは最新の世代に昇格し、立っていないpageは古い世代に残される（そして、より早く追い出される）。

## 追い出し: 古い世代の回収 {#eviction-reclaiming-old-generations}

**追い出し（eviction）**は最も古い世代からpageを回収する。

```c
/* mm/vmscan.c */
static long evict_folios(struct lruvec *lruvec, struct scan_control *sc,
                          swp_entry_t *swpent)
{
    int type = get_type_to_scan(lruvec, sc, &tier);
    long nr_to_scan = get_nr_to_scan(lruvec, sc, can_age, type);
    long nr_reclaimed = 0;

    /* 最古の世代を取得する */
    unsigned long min_seq = READ_ONCE(lrugen->min_seq[type]);

    /* 最古の世代からfolioを集める */
    isolate_folios(lruvec, sc, type, min_seq, &list);

    /* それらの回収を試みる（ライトバック、スワップ、または単に解放） */
    nr_reclaimed = shrink_folio_list(&list, pgdat, sc, &stat, false);

    /* 最小の世代が空になったら min_seq を進める */
    if (!nr_pages_in_gen(lruvec, type, min_seq))
        WRITE_ONCE(lrugen->min_seq[type], min_seq + 1);

    return nr_reclaimed;
}
```

## ワーキングセットの保護: refault検出 {#working-set-protection-refault-detection}

MGLRUは、追い出したpageがすぐにフォールトで再び読み込まれる状況（**refault**）を検出する。これはワーキングセットが侵されたことを示す。検出にはBloomフィルタを使う。

1. pageが追い出されると、そのフィンガープリントがBloomフィルタに追加される
2. pageがフォールトで読み込まれると、MGLRUはそれが最近追い出されたものかを確認する
3. そうであれば、そのpageは **保護された世代** に置かれる。これは最新の世代より古いが、即座の追い出しからは保護される

```c
/* refault検出 */
static bool lru_gen_test_recent(struct lruvec *lruvec, bool file,
                                 struct folio *folio, ...)
{
    struct lru_gen_folio *lrugen = &lruvec->lrugen;
    /* Bloomフィルタを確認: このfolioは最近追い出されたか？ */
    return test_bloom_filter(lrugen, lrugen->min_seq[file], folio);
}
```

## 比較: 従来のLRUとMGLRU {#comparison-classic-lru-vs-mglru}

| 観点 | 従来のLRU | MGLRU |
|--------|-------------|-------|
| データ構造 | active + inactiveリスト | 4つの世代リスト |
| 回収の対象 | inactiveリストを走査 | 最古の世代を追い出す |
| アクセスの追跡 | active/inactive間の昇格 | PTEのaccessedビットの走査 |
| ワーキングセットの検出 | refault距離による近似 | 世代ごとのBloomフィルタ |
| ストリーミング読み込み | activeリストを汚染する | 一度きりのアクセスは古い世代に留まる |
| メモリオーバーヘッド | folioごとに2つのlist_head | 世代カウンタ + Bloomフィルタ |

## 設定 {#configuration}

```bash
# MGLRU は 6.1 以降ではデフォルトで有効
cat /sys/kernel/mm/lru_gen/enabled
# 0x0007  ← ビットマスク: 1=有効, 2=mglru, 4=エージング

# 各コンポーネントの有効化/無効化
echo 0 > /sys/kernel/mm/lru_gen/enabled    # MGLRU を無効化（従来の LRU を使う）
echo 0x0007 > /sys/kernel/mm/lru_gen/enabled  # すべて有効化

# LRU の最小世代数（デフォルト 2）
cat /sys/kernel/mm/lru_gen/min_ttl_ms
# 0  ← 最小生存期間なし

# 世代が追い出されるまでに留まる最小時間を設定する
echo 1000 > /sys/kernel/mm/lru_gen/min_ttl_ms  # page を 1 秒間保護する

# 世代数: MAX_NR_GENS で制御される（コンパイル時、通常 4）
```

## MGLRUの観測 {#observing-mglru}

```bash
# ノード/memcg/ゾーンごとの世代分布
cat /sys/kernel/debug/lru_gen

# 出力形式:
# memcg    N    lruvec    aged anon  0  1  2  3
#                         evicted anon 0 1 2 3
#                         aged file  0  1  2  3
#                         evicted file 0 1 2 3

# メモリプレッシャーの統計
cat /proc/vmstat | grep -E "pgpromote|pgdemote|lru_gen"
# pgpromote_success 12345    ← より新しい世代に昇格した page
# pgdemote_kswapd 67890      ← 降格した page（エージングで古い世代へ）

# refault 率（ワーキングセットの侵害）
cat /proc/vmstat | grep pgrefault
# pgrefault 1234

# perf: LRU イベントを追跡する
perf stat -e vmscan:mm_vmscan_lru_isolate -a sleep 5
```

## 参考文献 {#further-reading}

- [ページ回収](reclaim.md) — 回収フレームワーク全体
- [回収のスロットリング](reclaim-throttling.md) — メモリプレッシャーの通知
- [スワップ](swap.md) — 追い出されたanonymous pageの行き先
- [ページキャッシュ](page-cache.md) — ファイルバックドpageのライフサイクル
- カーネルツリーの `mm/vmscan.c` — MGLRUの実装
- カーネルツリーの `Documentation/admin-guide/mm/multigen_lru.rst`
