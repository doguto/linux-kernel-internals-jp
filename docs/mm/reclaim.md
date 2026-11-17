# pageの回収 {#page-reclaim}

> メモリが不足したときに何が起きるか

## pageの回収とは何か? {#what-is-page-reclaim}

システムの空きメモリが少なくなると、カーネルは既存の利用者からpageを回収しなければならない。pageの回収は、解放できるpageやスワップへ書き出せるpageを見つけ、新しい割り当てのための空きを作る。

```
Memory Pressure
      │
      v
┌─────────────────┐
│  Page Reclaim   │
├─────────────────┤
│ - File pages    │──> Write dirty pages, drop clean
│ - Anonymous     │──> Swap out
│ - Slab caches   │──> Shrink caches
└─────────────────┘
      │
      v
Free Pages Available
```

## 回収への2つのパス {#two-paths-to-reclaim}

### 1. バックグラウンド回収（kswapd） {#1-background-reclaim-kswapd}

[`kswapd`](https://docs.kernel.org/mm/physical_memory.html)はノードごとのカーネルスレッドで、メモリが`low`ウォーターマークを下回るとバックグラウンドでpageを回収する。

```
Free pages fall below low watermark
           │
           v
    Wake kswapd
           │
           v
    Reclaim pages until high watermark reached
           │
           v
    kswapd sleeps
```

**主な特徴**: プロセスはブロックしない。kswapdは非同期に動作する。

### 2. 直接回収 {#2-direct-reclaim}

kswapdが追いつけず、空きpageが`min`ウォーターマークを下回ると、割り当てを行うプロセスが自らpageを回収しなければならない。

```
Allocation request
        │
        v
    Free pages < min?
        │ yes
        v
    Direct reclaim (process blocks)
        │
        v
    Retry allocation
```

**主な特徴**: 割り当てを行うプロセスは、pageが解放されるまでブロックする。

## ウォーターマーク {#watermarks}

各ゾーンには回収の挙動を制御する3つのウォーターマークがある。

```
             ┌──────────────────┐
  high ─────>│                  │  Zone healthy, no action
             ├──────────────────┤
  low  ─────>│  Wake kswapd     │  Background reclaim starts
             ├──────────────────┤
  min  ─────>│  Direct reclaim  │  Allocating process helps
             ├──────────────────┤
             │  OOM territory   │  Kill processes
             └──────────────────┘
```

ウォーターマークは[`vm.min_free_kbytes`から導出される](https://docs.kernel.org/admin-guide/sysctl/vm.html)。

```bash
# ウォーターマークを見る
cat /proc/zoneinfo | grep -E "min|low|high"

# 基準値を調整する（すべてのウォーターマークに影響する）
sysctl vm.min_free_kbytes=65536
```

## LRUの進化 {#lru-evolution}

### 単一リストのLRUとストリーミングの問題 {#single-list-lru-and-the-streaming-problem}

元々のページキャッシュのLRUは単一のリストだった。最近アクセスされたpageが先頭に、最も古いpageが末尾に来る。追い出しは末尾から行われた。安定したワーキングセットを持つワークロードに対しては、単純かつ正しい方式である。

問題は、大きなファイルを順次読み込むプロセス（「ストリーミング」）が、何千ものpageにちょうど1回ずつアクセスすることだった。各pageがLRUの先頭に置かれるたびに、ホットなワーキングセットのpageがすべて末尾へ押しやられる。ストリーミング読み込みが終わる頃には、カーネルは二度と読まれないファイルデータのための場所を作るために、データベースのバッファプールやアプリケーションのコードpageを追い出してしまっている。

これはLinuxの歴史の早い段階で観測された。修正である2つのリストは、Linux 2.6までには導入されていた。

### 2リストのLRU（active + inactive） {#two-list-lru-active-inactive}

2リストのモデルは、LRUを**active**リストと**inactive**リストに分ける。

- 新しいpageは（最初のフォールト/読み込み時に）**inactive**リストに入る
- 2回目にアクセスされたpageは**active**に昇格する
- 追い出しは**inactive**からのみ行われ、activeのpageは保護される

ストリーミング読み込みはinactiveリストを満たすが、それらのpageが2回目のアクセスを受けることはないので、activeリストに一度も触れることなく追い出される。ホットなワーキングセットは`active`に留まり、ストリームを生き延びる。

これは何十年もうまく機能したが、根本的な制約があった。1ビットの情報（「このpageはinactiveリストに置かれてからアクセスされたか?」）は、「このpageはどれくらいホットか?」の粗い近似にすぎない。過去1時間ずっとアクセスされ続けたpageも、1回だけアクセスされたpageも同じに見え、どちらも`active`に入ってしまう。

### Multi-Gen LRU（MGLRU、Linux 6.1、2022年） {#multi-gen-lru-mglru-linux-61-2022}

Yu Zhao（Google）は、（RAMが限られた）Androidデバイスが2リストのモデルの下でホットなpageとウォームなpageの間でスラッシングするのを観察した後、MGLRUを設計した。中核となる洞察は、2つのカテゴリ（active/inactive）の代わりに、時間帯を表す複数の**世代**（generation）を使うというものである。

pageはフォールト時に世代番号を割り当てられる。カーネルは定期的に世代を「エージング」する。最近アクセスされたpageはより新しい世代番号を得て、アクセスされていないpageは古い世代に留まる。追い出しは常に最も古い世代から行われる。

これにより、カーネルはpageの年齢についてはるかに細かい情報を得られる。世代8のpageは世代12より古く、追い出しのアルゴリズムはより良い選択ができる。MGLRUはまた、ハードウェアのダーティビットやアクセスビットをより積極的に使い、アクセスのたびにソフトウェアで明示的に追跡することなくpageの活動を追跡する。

## LRUリスト {#lru-lists}

カーネルはLRU（Least Recently Used）リストでpageの使用状況を追跡する。pageはアクセスパターンに基づいてリスト間を移動する。

### 古典的なLRU（2リスト） {#classic-lru-two-list}

```
              Active                    Inactive
         ┌─────────────┐           ┌─────────────┐
  Hot    │   Recent    │           │   Recent    │
 pages   │   access    │ ────────> │   access    │ ──> Reclaim
         │             │  demote   │             │
         └─────────────┘           └─────────────┘
               ^                          │
               │        promote           │
               └──────────────────────────┘
```

ノードごとに4つのリストがある（2種類 x 2状態）。

- `LRU_INACTIVE_ANON`: 非アクティブな無名page
- `LRU_ACTIVE_ANON`: アクティブな無名page
- `LRU_INACTIVE_FILE`: 非アクティブなファイルバックのpage
- `LRU_ACTIVE_FILE`: アクティブなファイルバックのpage

### Multi-Gen LRU（MGLRU） {#multi-gen-lru-mglru}

[v6.1で導入された](https://lwn.net/Articles/910608/)MGLRUは、エージングの精度を高めるために、2リストのモデルを[複数の世代](https://docs.kernel.org/admin-guide/mm/multigen_lru.html)で置き換える（[commit ec1c86b25f4b](https://git.kernel.org/linus/ec1c86b25f4b)）。

```
Generation 0 (oldest) ──> Generation 1 ──> ... ──> Generation N (youngest)
      │
      v
   Reclaim
```

**利点**:

- ホットなpageとコールドなpageをよりよく見分けられる
- ページテーブルのスキャンによるCPUオーバーヘッドが減る
- メモリ逼迫下での性能が向上する

```bash
# MGLRUが有効か確認する（16進のビットマスク）
cat /sys/kernel/mm/lru_gen/enabled
# 0x0007 = 完全に有効（0x0001 | 0x0002 | 0x0004）

# MGLRUのすべての機能を有効にする（CONFIG_LRU_GENでビルドされている場合）
echo 0x0007 > /sys/kernel/mm/lru_gen/enabled
# ビット: 0x0001=基本、0x0002=リーフPTEのアクセスビットをクリア、0x0004=非リーフPTE
```

## 何が回収されるか {#what-gets-reclaimed}

### ファイルpage（ページキャッシュ） {#file-pages-page-cache}

ディスク上のファイルに裏付けられたpage:

| 状態 | 動作 |
|-------|--------|
| クリーン | ただちに破棄する（ディスクから読み直せる） |
| ダーティ | まずディスクに書き込み、それから破棄する |

### 無名page {#anonymous-pages}

ファイルの裏付けがないpage（ヒープ、スタック）:

| スワップの有無 | 動作 |
|----------------|--------|
| あり | スワップに書き込んでから解放する |
| なし | 所有者をkillしない限り回収できない（OOM killer） |

### slabキャッシュ {#slab-caches}

カーネルオブジェクトのキャッシュは**shrinker**を通じて縮小できる。

```c
/* サブシステムが登録するshrinker */
register_shrinker(&my_shrinker);

/* 回収中に呼び出される */
struct shrinker {
    unsigned long (*count_objects)(struct shrinker *, ...);
    unsigned long (*scan_objects)(struct shrinker *, ...);
};
```

例: dentryキャッシュ、inodeキャッシュ、VFSのキャッシュ。

```bash
# shrinkerの統計を見る
cat /sys/kernel/debug/shrinker/*
```

## swappiness {#swappiness}

[`vm.swappiness`](https://docs.kernel.org/admin-guide/sysctl/vm.html)は、ファイルpageの回収と無名pageの回収のバランスを制御する。

| 値 | 挙動 |
|-------|----------|
| 0 | スワップを避け、ファイルキャッシュの回収を優先する |
| 60 | デフォルトのバランス |
| 100 | 無名pageを積極的にスワップする |

```bash
# 現在の値を見る
cat /proc/sys/vm/swappiness

# 調整する（/etc/sysctl.confで永続化）
sysctl vm.swappiness=10
```

## OOM killer {#oom-killer}

回収が失敗してメモリが枯渇すると、OOM（Out of Memory）killerがプロセスを終了させてメモリを解放する。

### OOMスコア {#oom-score}

各プロセスはOOMスコア（0〜1000）を持つ。高いほどkillされやすい。

```bash
# OOMスコアを見る
cat /proc/<pid>/oom_score

# OOMの優先度を調整する（-1000〜1000）
echo -500 > /proc/<pid>/oom_score_adj  # killされにくくする
echo 1000 > /proc/<pid>/oom_score_adj  # これを最初にkillする
```

### OOMの選択基準 {#oom-selection-criteria}

カーネルは次の点を考慮する。
1. メモリ使用量（主要な要素）
2. `oom_score_adj`による調整
3. killすることでメモリが解放されるか（子プロセス、共有page）
4. rootのプロセスは[わずかに保護されていた](https://www.kernel.org/doc/gorman/html/understand/understand016.html)（3%のボーナス、[v4.17で削除](https://git.kernel.org/linus/d46078b28889)）

```bash
# OOMイベントを監視する
dmesg | grep -i "out of memory"

# あるいはトレーシングで
echo 1 > /sys/kernel/debug/tracing/events/oom/oom_score_adj_update/enable
```

## 回収の挙動に関するチューニング可能なパラメータ {#reclaim-behavior-tunables}

| sysctl | 説明 |
|--------|-------------|
| `vm.min_free_kbytes` | ウォーターマーク計算の基準 |
| `vm.swappiness` | 無名とファイルの回収のバランス |
| `vm.vfs_cache_pressure` | dentry/inodeキャッシュにかける圧力 |
| `vm.dirty_ratio` | 書き込みをブロックする前のダーティpageの上限 |
| `vm.dirty_background_ratio` | バックグラウンドのライトバックを開始する時点 |

## 歴史 {#history}

### 起源 {#origins}

pageの回収は初期のLinuxから存在している。基本的な[kswapdの仕組み](https://www.kernel.org/doc/gorman/html/understand/understand013.html)は1996年1月にStephen Tweedieによって追加された（[vmscan.cのヘッダ](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c)に記されている）もので、gitの履歴より前のものである。

### rmap（逆マッピング、v2.5） {#rmap-reverse-mapping-v25}

**問題**: pageを回収するには、カーネルはそれを指すすべてのPTEを見つける必要があった。そのためにはすべてのページテーブルをスキャンする必要があった。

**解決策**: 逆マッピング。各pageが、どのPTEが自分をマップしているかを追跡する。

### 分割LRU（v2.6.28、2008年） {#split-lru-v2628-2008}

**コミット**: [4f98a2fee8ac](https://git.kernel.org/linus/4f98a2fee8ac)（"vmscan: split LRU lists into anon & file sets"）

**作者**: Rik van Riel

*注: lore.kernel.org上の2009年以前のLKMLアーカイブはまばらである。根拠はコミットメッセージに記されている。*

より良い回収の判断のために、単一のLRUを無名用とファイル用の別々のリストに分割した。

### MGLRU（v6.1、2022年） {#mglru-v61-2022}

**コミット**: [ac35a4902374](https://git.kernel.org/linus/ac35a4902374)（"mm: multi-gen LRU: minimal implementation"） | [LKML](https://lore.kernel.org/linux-mm/20220815071332.627393-1-yuzhao@google.com/)

**作者**: Yu Zhao

pageのエージングを改善し、オーバーヘッドを減らすための多世代LRU。

## 自分で試す {#try-it-yourself}

### 回収の活動を監視する {#monitor-reclaim-activity}

```bash
# リアルタイムの回収統計
watch -n 1 'cat /proc/vmstat | grep -E "pgscan|pgsteal|kswapd"'

# 主要な指標:
# pgscan_kswapd  - kswapdがスキャンしたpage
# pgscan_direct  - 直接回収がスキャンしたpage
# pgsteal_*      - 回収に成功したpage
```

### メモリ逼迫を起こす {#trigger-memory-pressure}

```bash
# メモリを消費して回収を引き起こす
stress --vm 1 --vm-bytes 2G --vm-keep

# kswapdが起きるのを監視する
watch -n 1 'ps aux | grep kswapd'
```

### 回収イベントをトレースする {#trace-reclaim-events}

```bash
# 回収のトレーシングを有効にする
echo 1 > /sys/kernel/debug/tracing/events/vmscan/mm_vmscan_direct_reclaim_begin/enable
echo 1 > /sys/kernel/debug/tracing/events/vmscan/mm_vmscan_direct_reclaim_end/enable

# イベントを監視する
cat /sys/kernel/debug/tracing/trace_pipe
```

### ゾーンの逼迫を確認する {#check-zone-pressure}

```bash
# ウォーターマークを含む詳細なゾーン情報
cat /proc/zoneinfo

# 手早い確認
cat /proc/buddyinfo
```

## よくある問題 {#common-issues}

### 直接回収によるストール {#direct-reclaim-stalls}

プロセスが直接回収でブロックされ、レイテンシのスパイクを引き起こす。

**デバッグ**: `/proc/vmstat`の`allocstall_*`を確認する

**解決策**:

- `vm.min_free_kbytes`を増やす
- スワップがなければ追加する
- メモリ逼迫を減らす

### kswapdのCPU使用率 {#kswapd-cpu-usage}

kswapdが過剰にCPUを消費する。

**デバッグ**: `top`または`perf top`

**原因**:

- 空きメモリが少なすぎる
- ワークロードが絶えずpageをダーティにしている
- スワップスラッシング

### 空きメモリがあるのにOOM killが起きる {#oom-kills-despite-free-memory}

空きメモリは存在するが、間違ったゾーンにあるか断片化している。

**デバッグ**: `/proc/buddyinfo`と`/proc/zoneinfo`を確認する

## 参照 {#references}

### 主要なコード {#key-code}

| ファイル | 説明 |
|------|-------------|
| [`mm/vmscan.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) | 回収の中核ロジック |
| [`mm/oom_kill.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/oom_kill.c) | OOM killer |
| [`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) | ウォーターマークのチェック |

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/admin-guide/mm/multigen_lru.rst`](https://docs.kernel.org/admin-guide/mm/multigen_lru.html)
- [`Documentation/mm/balance.rst`](https://docs.kernel.org/mm/balance.html)

### メーリングリストでの議論 {#mailing-list-discussions}

- [Multi-Gen LRU Framework v14](https://lore.kernel.org/linux-mm/20220815071332.627393-1-yuzhao@google.com/): Yu Zhaoによる2022年8月のパッチシリーズ
- [MGLRU design doc](https://lore.kernel.org/all/20220815071332.627393-15-yuzhao@google.com/): 世代、ページテーブルのウォーク、PIDコントローラを説明する詳細な設計文書

### 関連 {#related}

- [page-allocator](page-allocator.md): ウォーターマーク、ゾーン
- [glossary](glossary.md): LRU、OOM、スワップの定義
- [memcg](memcg.md): cgroupごとのメモリ上限と回収

## 参考文献 {#further-reading}

- [mm/vmscan.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c): 回収の中核エンジン。`shrink_node()`、`shrink_lruvec()`、`shrink_folio_list()`、`balance_pgdat()`（kswapdのメインループ）、`try_to_free_pages()`（直接回収のエントリポイント）
- `Documentation/admin-guide/mm/multigen_lru.rst`: MGLRUのアーキテクチャ、世代モデル、ページテーブルウォークの最適化、スキャンのオーバーヘッドを調整する組み込みのPIDコントローラ
- `Documentation/mm/balance.rst`: カーネルが回収の主体、ゾーン、NUMAノードの間でメモリのバランスをどう取るかについての高レベルの設計
- [LWN: Multi-generational LRU](https://lwn.net/Articles/910608/): MGLRUの設計目標、古典的な2リストモデルをどう改善するか、そして混在ワークロードでなぜCPUオーバーヘッドを減らすのかの紹介
- [LWN: Better active/inactive list balancing](https://lwn.net/Articles/495543/): 分割LRUの作業、ひいてはMGLRUの動機となった、古典的な2リストLRUの問題についての背景
- [reclaim-throttling](reclaim-throttling.md): 回収が割り当てのペースに追いつけないときに何が起きるか。`reclaim_throttle()`、`memory.high`による比例的な遅延、kswapdと直接回収者がどう協調するか
- [shrinker](shrinker.md): カーネルのサブシステム（dentry/inodeキャッシュ、ドライバのオブジェクトプール）がコールバックを登録し、回収がページキャッシュや無名pageと並んでそれらを縮小できるようにする仕組み
- [memcg](memcg.md): `memory.high`と`memory.max`によって引き起こされるcgroupごとの回収と、`memory.reclaim`によるプロアクティブ回収のインターフェース
