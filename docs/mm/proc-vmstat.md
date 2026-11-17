# /proc/vmstatを理解する {#understanding-procvmstat}

> メモリサブシステムを診断するためのカウンタとゲージ

## /proc/vmstatとは何か {#what-is-procvmstat}

`/proc/vmstat`は、カーネル内部のメモリ管理カウンタを公開する。現在の状態を示す`/proc/meminfo`とは異なり、`/proc/vmstat`には**ゲージ**（現在値）と**カウンタ**（ブート以降単調増加する値）の両方が含まれる。

出力は`mm/vmstat.c`の[`vmstat_show()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmstat.c)によって生成され、ゾーンごと、ノードごと、CPUごとの統計が集約される。

```bash
$ cat /proc/vmstat | head -20
nr_free_pages 131072
nr_zone_inactive_anon 262144
nr_zone_active_anon 524288
nr_zone_inactive_file 196608
nr_zone_active_file 131072
...
pgfault 48392017
pgmajfault 12847
pgscan_kswapd 0
pgscan_direct 0
```

## カウンタとゲージ {#counters-vs-gauges}

この区別は監視において決定的に重要である。`/proc/vmstat`には両方の種類が混在している。

### ゲージ（現在の状態 — 増えも減りもする） {#gauges-current-state-can-go-up-or-down}

`nr_`という接頭辞が付く。累積値ではなく**現在**の値を反映する。現在の状態を知るにはそのまま読めばよい。

| フィールド | 測るもの |
|-------|-----------------|
| `nr_free_pages` | バディアロケータ内の空きページ |
| `nr_inactive_anon` / `nr_active_anon` | LRUリスト上の匿名ページ |
| `nr_inactive_file` / `nr_active_file` | LRUリスト上のファイルページ |
| `nr_dirty` | 変更されたがまだディスクに書き出されていないページ |
| `nr_writeback` | 現在ディスクへ書き出し中のページ |
| `nr_slab_reclaimable` | カーネルが解放できるslabページ |
| `nr_slab_unreclaimable` | 解放できないslabページ |
| `nr_page_table_pages` | ページテーブルに使われているページ |
| `nr_shmem` | 共有メモリ / tmpfsのページ |
| `nr_file_pages` | ページキャッシュ内のページの総数 |
| `nr_anon_pages` | マップされている匿名ページ |
| `nr_unevictable` | ロックされた/追い出し不能なページ |
| `nr_swapcached` | スワップキャッシュ内のページ |
| `nr_kernel_stack` | カーネルスタックの使用量（KiB単位） |

### カウンタ（ブート以降単調増加） {#counters-monotonically-increasing-since-boot}

[`enum vm_event_item`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/vm_event_item.h)で定義されている。レートを得るには、2回の読み取りの間の**差分**を計算する必要がある。

**落とし穴**: `nr_dirtied`と`nr_written`は`nr_`接頭辞を持つが、実際にはゲージではなくカウンタ（ブート以降にdirtyになった/書き出されたページの総数）である。

## ページフォールトのカウンタ {#page-fault-counters}

| フィールド | 数えるもの | ソース |
|-------|---------------|--------|
| `pgfault` | ページフォールトの総数（マイナー + メジャー） | [`handle_mm_fault()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memory.c) |
| `pgmajfault` | メジャーフォールトのみ（ディスクI/Oを必要としたもの） | `mm/memory.c`、`mm/filemap.c` |

マイナーフォールト = `pgfault - pgmajfault`。マイナーフォールトは、ページがすでにメモリ上（ページキャッシュやCOW）にあり、ページテーブルエントリだけが必要だったことを意味する。メジャーフォールトは、カーネルがディスクから読み込まなければならなかったことを意味する。

**診断**: `pgmajfault / pgfault`の比率が約1%を超える場合、I/Oのレイテンシがアプリケーションの性能に影響している可能性が高い。

## 回収のカウンタ {#reclaim-counters}

メモリ逼迫を診断するうえで最も重要なカウンタである。

### スキャンとスティール {#scanning-vs-stealing}

| フィールド | 数えるもの |
|-------|---------------|
| `pgscan_kswapd` | kswapd（バックグラウンド回収）が調べたページ |
| `pgscan_direct` | 直接回収（割り当てを行うプロセス、同期的）が調べたページ |
| `pgsteal_kswapd` | kswapdが回収に成功したページ |
| `pgsteal_direct` | 直接回収が回収に成功したページ |

いずれも[`mm/vmscan.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c)の`shrink_inactive_list()`と関連する関数でインクリメントされる。

**ページの種類別**:

| フィールド | 数えるもの |
|-------|---------------|
| `pgscan_anon` | スキャンされた匿名ページ |
| `pgscan_file` | スキャンされたファイルバックページ |
| `pgsteal_anon` | 回収された匿名ページ |
| `pgsteal_file` | 回収されたファイルバックページ |

### 回収効率 {#reclaim-efficiency}

```
efficiency = pgsteal / pgscan
```

| 比率 | 意味 |
|-------|---------|
| > 0.8 | 効率的 — 回収できるcold/cleanなページが十分にある |
| 0.3 - 0.8 | 中程度の逼迫 — 一部のページがdirtyまたはピン留めされている |
| < 0.3 | 苦戦している — スキャンしたページの大半が解放できず、OOMが続く可能性がある |

ボトルネックを突き止めるには、kswapdと直接回収、匿名とファイルを別々に計算する。

### 割り当てストール {#allocation-stalls}

| フィールド | 数えるもの |
|-------|---------------|
| `allocstall_normal` | Normalゾーンで直接回収に入ったプロセス |
| `allocstall_dma32` | DMA32ゾーンについて同上 |
| `allocstall_dma` | DMAゾーンについて同上 |
| `allocstall_movable` | Movableゾーンについて同上 |

`__alloc_pages_direct_reclaim()`に入ったときに[`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c)でインクリメントされる。

**インクリメントのたびに、あるプロセスがメモリを待ってブロックされたことを意味する。** これはレイテンシのスパイクを直接引き起こす。`allocstall_normal`が毎秒数回を超えるペースで持続的に増えているなら、システムには空きメモリの余裕がもっと必要である。

## スワップのカウンタ {#swap-counters}

| フィールド | 数えるもの |
|-------|---------------|
| `pswpin` | スワップから読み込んだページ（スワップイン） |
| `pswpout` | スワップへ書き出したページ（スワップアウト） |
| `pgpgin` | ブロックデバイスから読み込んだ全ページ（KiB） — ファイルシステムのI/Oを含む |
| `pgpgout` | ブロックデバイスへ書き出した全ページ（KiB） |

`pswpin`/`pswpout`はスワップ操作に特化したものである。`pgpgin`/`pgpgout`はより広い（すべてのブロックI/O）。

## LRU間の移動 {#lru-movement}

| フィールド | 数えるもの |
|-------|---------------|
| `pgactivate` | 昇格したページ: inactive -> active（inactive中にアクセスされた） |
| `pgdeactivate` | 降格したページ: active -> inactive（回収の候補） |
| `pglazyfree` | `MADV_FREE`で遅延解放（lazy free）とマークされたページ |
| `pglazyfreed` | 実際に回収されたlazyfreeページ |

## コンパクションのカウンタ {#compaction-counters}

| フィールド | 数えるもの |
|-------|---------------|
| `compact_stall` | コンパクションを待ってブロックされたプロセス |
| `compact_success` | 連続ブロックを作り出せたコンパクションの実行 |
| `compact_fail` | 失敗したコンパクションの実行 |
| `compact_migrate_scanned` | マイグレーションスキャナがスキャンしたページ |
| `compact_free_scanned` | フリースキャナがスキャンしたページ |
| `compact_isolated` | マイグレーションのために分離されたページ |
| `compact_daemon_wake` | kcompactdが起こされた回数 |

ソース: [`mm/compaction.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/compaction.c)。

**失敗率**: `compact_fail / (compact_fail + compact_success)`が50%を超える場合、移動不能なページがコンパクションを妨げる深刻な断片化が起きていることを意味する。

## THPのカウンタ {#thp-counters}

| フィールド | 数えるもの |
|-------|---------------|
| `thp_fault_alloc` | ページフォールト時に割り当てられたTHP（同期的） |
| `thp_fault_fallback` | フォールト時のTHP割り当てに失敗し、4KBにフォールバックした |
| `thp_collapse_alloc` | khugepagedが割り当てたTHP（非同期のcollapse） |
| `thp_collapse_alloc_failed` | khugepagedがTHPの割り当てに失敗した |
| `thp_split_page` | ベースページに分割し直されたTHP |
| `thp_split_pmd` | PMDレベルの分割（PMDエントリを置き換えるPTEが作られた） |
| `thp_swpout` | 丸ごとスワップアウトされたTHP（[v4.13](https://git.kernel.org/linus/38d8b4e6bdc8)以降） |
| `thp_swpout_fallback` | THPのスワップアウトが分割にフォールバックした |

ソース: [`mm/huge_memory.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/huge_memory.c)、[`mm/khugepaged.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/khugepaged.c)。

## OOMのカウンタ {#oom-counter}

| フィールド | 数えるもの |
|-------|---------------|
| `oom_kill` | OOM killerが起動された回数 |

[`mm/oom_kill.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/oom_kill.c)でインクリメントされる。

## 診断のパターン {#diagnostic-patterns}

### メモリ逼迫の検出 {#detecting-memory-pressure}

```bash
# 回収の活動を観察 (1秒ごとの差分)
watch -n 1 'grep -E "pgscan_|pgsteal_|allocstall" /proc/vmstat'
```

| シグナル | 意味 |
|--------|---------|
| `pgscan_kswapd`が増加 | バックグラウンド回収が動作中 — 中程度の逼迫 |
| `pgscan_direct`が増加 | 直接回収が動作中 — 深刻な逼迫、プロセスがブロックされている |
| `allocstall_normal`が増加 | 割り当てストール — アプリケーションのレイテンシに影響 |
| `pgsteal / pgscan`の比率が低下 | 回収効率が低下 — 回収しやすいページが尽きつつある |

### スワップのスラッシングの検出 {#detecting-swap-thrashing}

```bash
grep -E "pswpin|pswpout|pgmajfault" /proc/vmstat
```

`pswpin` + `pswpout`が多く、`pgmajfault`も多い = システムが活発にスワップしており、アプリケーションがページをフォールトで読み戻している。ワーキングセットがRAMに収まっていない。

### THPの問題の検出 {#detecting-thp-problems}

```bash
grep "thp_" /proc/vmstat
```

| パターン | 問題 |
|---------|---------|
| `thp_fault_fallback`が急増 | メモリの断片化が進み、2MBの割り当てができない |
| `thp_fault_alloc`と`thp_split_page`が両方増加 | THPのチャーン — 割り当てては分割し、CPUを浪費している |
| `compact_stall`が多く`thp_fault_fallback`も多い | コンパクションでストールしているのに失敗し続けている |

**対処**: `madvise`モードに切り替えるか、`defrag=defer`を設定する。
```bash
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
```

### コンパクションの問題の検出 {#detecting-compaction-problems}

| シグナル | 問題 |
|--------|---------|
| `compact_stall`が増加 | プロセスがコンパクションでブロックされている |
| `compact_fail >> compact_success` | 移動不能なページがデフラグを妨げている |
| `compact_migrate_scanned >> compact_isolated` | マイグレーションスキャナはページを見つけるが移動できない |

## ツールで/proc/vmstatを使う {#using-procvmstat-with-tools}

### 直接読む（最も網羅的） {#direct-reading-most-complete}

```bash
# 差分 (毎秒のレート) を計算
paste <(cat /proc/vmstat) <(sleep 1; cat /proc/vmstat) | \
  awk '$1 == $3 && $2 != $4 {print $1, $4-$2, "delta/s"}'
```

### vmstatコマンド（procps） {#vmstat-command-procps}

```bash
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
```

| 列 | /proc/vmstatでの元データ |
|--------|-------------------|
| `si`（swap in） | `pswpin`の差分をKB/sに換算 |
| `so`（swap out） | `pswpout`の差分をKB/sに換算 |
| `bi`（block in） | `pgpgin`の差分 |
| `bo`（block out） | `pgpgout`の差分 |

### sarコマンド（sysstat） {#sar-command-sysstat}

```bash
# ページフォールトと回収のレート
sar -B 1
# fault/s = pgfaultの差分, majflt/s = pgmajfaultの差分
# pgscank/s = pgscan_kswapdの差分, pgscand/s = pgscan_directの差分

# スワップのレート
sar -W 1
# pswpin/s, pswpout/s はそれぞれ対応するカウンタから
```

## 試してみる {#try-it-yourself}

```bash
# どのカウンタが活発に変化しているかを見る (メモリ逼迫テスト)
# 端末1: 逼迫を発生させる
stress --vm 2 --vm-bytes 80% --timeout 30s

# 端末2: カウンタの変化を観察
watch -d -n 1 'grep -E "pgscan|pgsteal|allocstall|pgmajfault|pswp|oom_kill|compact_stall" /proc/vmstat'

# 回収効率を確認
awk '/pgscan_kswapd/{scan=$2} /pgsteal_kswapd/{steal=$2} END{if(scan>0) printf "kswapd efficiency: %.1f%%\n", steal/scan*100; else print "No kswapd activity"}' /proc/vmstat

# THPの成功率を確認
awk '/thp_fault_alloc /{ok=$2} /thp_fault_fallback /{fail=$2} END{total=ok+fail; if(total>0) printf "THP success: %.1f%% (%d/%d)\n", ok/total*100, ok, total; else print "No THP faults"}' /proc/vmstat
```

## 主要なソースファイル {#key-source-files}

| ファイル | 内容 |
|------|-----------------|
| [`mm/vmstat.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmstat.c) | 出力の生成、per-CPUの集約 |
| [`include/linux/vm_event_item.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/vm_event_item.h) | カウンタの定義 |
| [`include/linux/mmzone.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mmzone.h) | ゲージの定義（ゾーン/ノードの統計項目） |
| [`mm/vmscan.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) | 回収のカウンタ |
| [`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) | 割り当てとストールのカウンタ |
| [`mm/compaction.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/compaction.c) | コンパクションのカウンタ |

## 参考文献 {#further-reading}

- [/proc/meminfo](proc-meminfo.md) — 現在の状態を補完的に示すビュー
- [OOMログの読み方](reading-oom-log.md) — これらのカウンタが危機的になったときに何が起きるか
- [ページ回収](reclaim.md) — kswapdと直接回収の仕組み
- [vm sysctlのドキュメント](https://docs.kernel.org/admin-guide/sysctl/vm.html) — チューニング可能なパラメータ
