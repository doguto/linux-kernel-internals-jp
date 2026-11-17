# メモリコンパクション {#memory-compaction}

> 大きな割り当てのために物理メモリをデフラグする

## コンパクションとは何か {#what-is-compaction}

メモリコンパクションは、ページを移動して連続した空き領域を作り出す。時間が経つにつれ、空きメモリは断片化し、物理メモリ全体に小さな断片として散らばる。コンパクションはこれらの断片を集約する。

```
Before Compaction:
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ U │ F │ U │ F │ U │ U │ F │ U │ F │ F │ U │ F │ U │ F │ U │ F │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  U = Used, F = Free (scattered)

After Compaction:
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ U │ U │ U │ U │ U │ U │ U │ U │ U │ F │ F │ F │ F │ F │ F │ F │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  Contiguous free region created
```

## なぜコンパクションか {#why-compaction}

### 断片化の問題 {#the-fragmentation-problem}

バディアロケータは、高次（high-order）の割り当てに物理的に連続したページを必要とする。

| order | サイズ | 用途 |
|-------|------|-----------|
| 0 | 4KB | 通常のページ |
| 1 | 8KB | 一部のカーネル割り当て |
| 2 | 16KB | ジャンボフレーム |
| 3+ | 32KB+ | huge page、DMAバッファ |

数日〜数週間稼働した後のシステムでは、空きメモリが十分にあっても高次割り当てに使える連続領域がないことがある。

### コンパクション以前 {#before-compaction}

コンパクションが導入される前は、高次割り当ての失敗はよくあることだった。

- THPが2MBのページを割り当てられない
- DMAバッファの割り当てが失敗する
- ネットワークのジャンボフレームが使えない

解決策は再起動するか、ブート時にメモリを予約しておくことしかなかった。

## コンパクションの仕組み {#how-compaction-works}

### アルゴリズム {#the-algorithm}

コンパクションは、互いに向かって進む2つのスキャナを使う。

```
Zone start                                              Zone end
    │                                                       │
    ▼                                                       ▼
┌───────────────────────────────────────────────────────────────┐
│                           Zone                                │
└───────────────────────────────────────────────────────────────┘
    │                                                       │
    ▼ Migration scanner                   Free scanner ▼    │
    │ (finds movable pages)         (finds free pages)      │
    │                                                       │
    └──────────────► ◄──────────────────────────────────────┘
                     Meet in middle
```

1. **フリースキャナ**: ゾーンの末尾から開始して後方へ進み、空きページを探す
2. **マイグレーションスキャナ**: ゾーンの先頭から開始して前方へ進み、移動可能なページを探す
3. **マイグレーション**: マイグレーションスキャナが見つけたページを、フリースキャナが見つけた位置へ移動する
4. **結果**: 空き領域が一方の端に集約される

### ページの移動可能性 {#page-mobility}

すべてのページが移動できるわけではない。

| マイグレートタイプ | 移動可能か | 例 |
|--------------|----------|----------|
| `MIGRATE_MOVABLE` | はい | ユーザーページ、ページキャッシュ |
| `MIGRATE_RECLAIMABLE` | 場合による | キャッシュ（代わりに解放できる） |
| `MIGRATE_UNMOVABLE` | いいえ | カーネルの割り当て、DMA |

コンパクションは移動可能なページにしか作用しない。移動不能なページは恒久的な断片化を生む。

## コンパクションのトリガ {#compaction-triggers}

### 1. 直接コンパクション（同期） {#1-direct-compaction-synchronous}

高次割り当てが失敗すると、割り当てを行っているプロセス自身がコンパクションを実行する。

```
alloc_pages(order=9)  /* 2MB for THP */
        │
        v
    Allocation fails
        │
        v
    Direct compaction (process blocks)
        │
        v
    Retry allocation
```

### 2. kcompactd（非同期） {#2-kcompactd-asynchronous}

先回りしてコンパクションを行うバックグラウンドのカーネルスレッドである。

```bash
# NUMAノードごとに1つのkcompactd
ps aux | grep kcompactd
# kcompactd0, kcompactd1, ...
```

起床するのは次のときである。

- ウォーターマークが断片化を示しているとき
- 高次割り当てが失敗しているとき
- 明示的にトリガされたとき

### 3. 手動トリガ {#3-manual-trigger}

```bash
# 全ノードでコンパクションをトリガ
echo 1 > /proc/sys/vm/compact_memory

# ノード単位のトリガ
echo 1 > /sys/devices/system/node/node0/compact
```

## 設定 {#configuration}

### プロアクティブコンパクション（v5.9+） {#proactive-compaction-v59}

割り当てで必要になる前に、バックグラウンドでコンパクションを行う。

```bash
# プロアクティブコンパクションを有効化 (0-100、0=無効)
cat /proc/sys/vm/compaction_proactiveness
echo 20 > /proc/sys/vm/compaction_proactiveness

# 高い = より積極的なバックグラウンドコンパクション
# 低い = CPU使用量は少ないが、直接コンパクションが増える
```

### コンパクションの挙動 {#compaction-behavior}

```bash
# コンパクションの断片化インデックスしきい値
# fragmentation_index > extfrag_threshold のときコンパクションが起動する
# インデックス0 = メモリ不足による失敗、1000 = 断片化による失敗
cat /proc/sys/vm/extfrag_threshold
# 500 (デフォルト) - 低いほど積極的 (より低い断片化度でコンパクションする)

# orderごとの断片化インデックスを見る
cat /sys/kernel/debug/extfrag/extfrag_index
```

## 監視 {#monitoring}

### コンパクションの統計 {#compaction-statistics}

```bash
cat /proc/vmstat | grep compact
# compact_migrate_scanned  - マイグレーションのためにスキャンしたページ数
# compact_free_scanned     - 空き領域を探してスキャンしたページ数
# compact_isolated         - マイグレーションのために分離したページ数
# compact_stall            - 直接コンパクションによるストール
# compact_fail             - コンパクションの失敗
# compact_success          - コンパクションの成功
```

### 断片化インデックス {#fragmentation-index}

```bash
# orderごとの断片化 (0.0 = 断片化なし、1.0 = 深刻)
cat /sys/kernel/debug/extfrag/extfrag_index

# 例:
# Node 0, zone   Normal
#     order   0   1   2   3   4   5   6   7   8   9  10
#     index 0.0 0.0 0.0 0.0 0.1 0.2 0.4 0.6 0.8 0.9 1.0
```

### トレーシング {#tracing}

```bash
# コンパクションイベントをトレース
echo 1 > /sys/kernel/debug/tracing/events/compaction/mm_compaction_begin/enable
echo 1 > /sys/kernel/debug/tracing/events/compaction/mm_compaction_end/enable
cat /sys/kernel/debug/tracing/trace_pipe
```

## 変遷 {#evolution}

### 導入（v2.6.35、2010年） {#introduction-v2635-2010}

**コミット**: [748446bb6b5a](https://git.kernel.org/linus/748446bb6b5a) ("mm: compaction: memory compaction core") | [LKML](https://lore.kernel.org/lkml/1269347146-7461-1-git-send-email-mel@csn.ul.ie/)

**作者**: Mel Gorman

THPをサポートし、高次割り当ての失敗を減らすための最初のコンパクション実装。

### kcompactd（v4.6、2016年） {#kcompactd-v46-2016}

**コミット**: [698b1b30642f](https://git.kernel.org/linus/698b1b30642f) ("mm, compaction: introduce kcompactd") | [LKML](https://lore.kernel.org/lkml/1454938691-2197-1-git-send-email-vbabka@suse.cz/)

**作者**: Vlastimil Babka

回収におけるkswapdに相当する、バックグラウンドのコンパクションデーモン。

### プロアクティブコンパクション（v5.9、2020年） {#proactive-compaction-v59-2020}

**コミット**: [facdaa917c4d](https://git.kernel.org/linus/facdaa917c4d) ("mm: proactive compaction") | [LKML](https://lore.kernel.org/linux-mm/20200428221055.598-1-nigupta@nvidia.com/)

**作者**: Nitin Gupta

断片化の度合いに基づいて先回りしてコンパクションを行い、直接コンパクションによるストールを減らす。

## コンパクションと回収の比較 {#compaction-vs-reclaim}

どちらもメモリ逼迫時に動作するが、目的が異なる。

| 観点 | 回収 | コンパクション |
|--------|---------|------------|
| 目的 | メモリを解放する | 連続領域を作る |
| タイミング | 空きページが少ないとき | 高次割り当てが失敗したとき |
| 動作 | ページを追い出す/スワップする | ページを移動する |
| デーモン | kswapd | kcompactd |

両者はしばしば協調して働く。回収がページを解放し、コンパクションがそれを並べ直す。

## よくある問題 {#common-issues}

### 直接コンパクションによるストール {#direct-compaction-stalls}

プロセスがコンパクションを待ってブロックされる。

**症状**: レイテンシのスパイク、`compact_stall`のカウントが多い

**対策**:

- プロアクティブコンパクションを有効にする
- `compaction_proactiveness`を調整する
- 高次割り当てを減らす

### コンパクションの失敗 {#compaction-failures}

連続領域を作り出せない。

**症状**: `compact_fail`が多い、THPの割り当て失敗

**原因**:

- 移動不能なページが多すぎる
- 断片化が深刻

**対策**:

- カーネルのメモリ使用量を減らす
- movableゾーンのサイズを増やす
- 分離のためにメモリホットプラグを検討する

### kcompactdによる高いCPU使用率 {#high-cpu-from-kcompactd}

バックグラウンドのコンパクションがCPUを消費している。

**デバッグ**: `top`、`perf top`

**対策**:

- `compaction_proactiveness`を下げる
- 代わりに直接コンパクションが増えることを受け入れる

## リファレンス {#references}

### 主要なコード {#key-code}

| ファイル | 説明 |
|------|-------------|
| [`mm/compaction.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/compaction.c) | コンパクションの実装 |
| [`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) | コンパクションのトリガ |

### 関連 {#related}

- [page-allocator](page-allocator.md) - バディシステム、マイグレートタイプ
- [thp](thp.md) - コンパクションの主な利用者
- [reclaim](reclaim.md) - コンパクションと並行して動作する
- [contiguous-memory](contiguous-memory.md) - 連続割り当てが失敗する理由と解決策

## 参考文献 {#further-reading}

- [page-allocator.md](page-allocator.md) — コンパクションが何を移動できるかを決める、バディアロケータのマイグレートタイプ（`MIGRATE_MOVABLE`、`MIGRATE_UNMOVABLE`）
- [thp.md](thp.md) — Transparent huge pages：コンパクションの主な動機。order-9の連続した物理ページを必要とする
- [reclaim.md](reclaim.md) — kswapdと直接回収。コンパクションと回収は同じウォーターマークのロジックのもとで連携する
- [cma.md](cma.md) — Contiguous Memory Allocator。予約されたCMA領域を取り戻すためにマイグレーションベースのコンパクションを使う
- [mm/compaction.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/compaction.c) — コンパクションの実装全体。`compact_zone()`が2スキャナのアルゴリズムを実行する
- [mm/page_alloc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) — 割り当て失敗時に直接コンパクションがトリガされる場所は`try_to_compact_pages()`
- [LWN: Memory compaction](https://lwn.net/Articles/368869/) — Mel Gormanによる2スキャナのアルゴリズムと断片化インデックスの紹介
- [LWN: Proactive compaction](https://lwn.net/Articles/817905/) — Nitin Guptaによるv5.9のプロアクティブコンパクションと`compaction_proactiveness`パラメータ
- [748446bb6b5a](https://git.kernel.org/linus/748446bb6b5a) — v2.6.35でのMel Gormanによる最初のコンパクションコアのコミット
- [facdaa917c4d](https://git.kernel.org/linus/facdaa917c4d) — 直接コンパクションによるストールを減らすためにv5.9で追加されたプロアクティブコンパクション
- `Documentation/admin-guide/mm/transhuge.rst` — THPの割り当て、コンパクションの調整、`compact_memory` sysctlを扱う
