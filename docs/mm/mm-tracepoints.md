# メモリサブシステムのtracepoint {#memory-subsystem-tracepoints}

> perf、ftrace、BPFによる、カーネルのメモリイベントの実行時トレーシング

カーネルのメモリ管理サブシステムは数十個のtracepointを公開しており、ページ割り当て、回収、OOMの判断、ページフォールト、コンパクション、huge page、slabの活動を実行時に観測できる。再コンパイルは不要で、無効時のオーバーヘッドも非常に小さい。

このリファレンスでは、tracepointをカテゴリごとにまとめ、利用可能なフィールドと、ftrace、`perf`、BPF（`bpftrace`経由）を使った実践的な診断例を扱う。

## 前提条件と一般的な使い方 {#prerequisites-and-general-usage}

### 利用可能なtracepointを探す {#finding-available-tracepoints}

```bash
# mm関連のtracepointをすべて列挙する
ls /sys/kernel/debug/tracing/events/kmem/
ls /sys/kernel/debug/tracing/events/vmscan/
ls /sys/kernel/debug/tracing/events/compaction/
ls /sys/kernel/debug/tracing/events/huge_memory/

# あるいはperfで
perf list 'mm:*' 'kmem:*' 'vmscan:*' 'compaction:*' 2>/dev/null
```

### ftraceで有効にする {#enabling-with-ftrace}

```bash
# 単一のtracepointを有効にする
echo 1 > /sys/kernel/debug/tracing/events/kmem/mm_page_alloc/enable

# サブシステム全体を有効にする
echo 1 > /sys/kernel/debug/tracing/events/vmscan/enable

# トレースバッファを読む
cat /sys/kernel/debug/tracing/trace

# イベントをライブでストリーミングする
cat /sys/kernel/debug/tracing/trace_pipe

# 後片付け
echo 0 > /sys/kernel/debug/tracing/events/kmem/mm_page_alloc/enable
echo > /sys/kernel/debug/tracing/trace   # バッファをクリアする
```

### perfで有効にする {#enabling-with-perf}

```bash
# mmのtracepointを10秒間記録する
perf record -e 'kmem:mm_page_alloc,kmem:mm_page_free' -a -- sleep 10
perf script

# 毎秒のイベント数を数える
perf stat -e 'kmem:mm_page_alloc,kmem:mm_page_free' -a -- sleep 5
```

### bpftraceで有効にする {#enabling-with-bpftrace}

```bash
# bpftraceは同じtracepoint名を使う
bpftrace -e 'tracepoint:kmem:mm_page_alloc { @[comm] = count(); }'
```

---

## ページ割り当てのtracepoint {#page-allocation-tracepoints}

**ソース**: [`include/trace/events/kmem.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/kmem.h)

### mm_page_alloc {#mmpagealloc}

ページアロケータ（[`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c)の`__alloc_pages()`）がpageまたはcompound pageを正常に返すたびに発火する。

**フィールド**:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `pfn` | `unsigned long` | 先頭pageのPFN（`struct page *`を得るには`pfn_to_page()`を使う） |
| `order` | `unsigned int` | 割り当てのorder（0 = 単一page、1 = 2 page、...） |
| `gfp_flags` | `gfp_t` | 割り当てに使われたGFPフラグ |
| `migratetype` | `int` | 割り当てのmigrate type（UNMOVABLE、MOVABLE、RECLAIMABLE） |

**フォーマット文字列**（`/sys/kernel/debug/tracing/events/kmem/mm_page_alloc/format`より）:

```
page=%p pfn=%lu order=%d migratetype=%d gfp_flags=%s
```

**有効化**:

```bash
echo 1 > /sys/kernel/debug/tracing/events/kmem/mm_page_alloc/enable
```

**診断での使い方**: ページ割り当ての頻度とorderごとの分布を測定する。order > 0の割り当てが高い頻度で失敗する（`mm_page_alloc_extfrag`にフォールバックする）場合、断片化の圧力を示している。

```bash
# bpftrace: 割り当てorderのヒストグラム
bpftrace -e '
tracepoint:kmem:mm_page_alloc {
    @order_hist = hist(args->order);
}
interval:s:5 {
    print(@order_hist);
    clear(@order_hist);
}'
```

### mm_page_free {#mmpagefree}

page（またはcompound page）が`free_pages()` / `__free_pages()`を介してページアロケータに返却されたときに発火する。

**フィールド**:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `pfn` | `unsigned long` | 解放されるpageのPFN |
| `order` | `unsigned int` | 解放されるcompound pageのorder |

**診断での使い方**: `mm_page_alloc`と組み合わせると、正味の割り当て率（割り当て数 − 解放数）を追跡できる。両者の数が乖離していく場合、メモリリークまたは蓄積を示している。

```bash
# perf: 30秒間の割り当て数と解放数を数える
perf stat -e kmem:mm_page_alloc,kmem:mm_page_free -a -- sleep 30
```

### mm_page_alloc_zone_locked {#mmpagealloczonelocked}

ページアロケータがzoneロックのパスにフォールバックしたときに発火する。典型的には、per-CPUページセット（PCP）が空で、zoneのフリーリストから補充しなければならない場合である。

**フィールド**:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `pfn` | `unsigned long` | 割り当てられたpageのPFN |
| `order` | `unsigned int` | 割り当てのorder |
| `migratetype` | `int` | migration type |

**診断での使い方**: `mm_page_alloc`に対して`mm_page_alloc_zone_locked`イベントが頻繁な場合、PCPリストが頻繁に枯渇していることを意味する。これは割り当てのバースト時や、`vm.percpu_pagelist_high_fraction`が控えめに設定されすぎている場合に起こりうる。

### mm_page_alloc_extfrag {#mmpagealloc_extfrag}

アロケータが、別のmigratetypeのフリーリストからpageを**盗む**ことで要求を満たしたときに発火する。これは断片化の主要な指標である。

**フィールド**:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `page` | `struct page *` | 盗まれたpage |
| `alloc_order` | `int` | 本来要求されたorder |
| `fallback_order` | `int` | フォールバックリストから実際に使われたorder |
| `alloc_migratetype` | `int` | 要求されたmigration type |
| `fallback_migratetype` | `int` | 盗まれた側のmigration type |
| `change_ownership` | `int` | pageblockのmigratetypeが変更されたかどうか |

**診断での使い方**: `change_ownership=1`の`mm_page_alloc_extfrag`イベントが持続する場合、移動不可能な割り当てが移動可能なpageblockを恒久的に侵食していることを意味する。これはコンパクションの失敗につながる。[コンパクション](compaction.md)を参照。

---

## ページ回収のtracepoint {#page-reclaim-tracepoints}

**ソース**: [`include/trace/events/vmscan.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/vmscan.h)

これらのtracepointは、バックグラウンドの回収（kswapd）と、割り当てを行うプロセスをブロックする同期的な直接回収の両方をカバーする。

### mm_vmscan_kswapd_wake {#mmvmscankswapd_wake}

バックグラウンドのページ回収を開始するために`kswapd`が起床したときに発火する。

**フィールド**:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `nid` | `int` | NUMAノードID |
| `zid` | `int` | zoneインデックス |
| `order` | `int` | 起床のきっかけとなった割り当てのorder |

**診断での使い方**: kswapdが毎秒何回起床するかを数える。起床頻度が高いのは、システムがメモリ圧迫下にあるものの、まだバックグラウンド回収の段階（直接回収が始まる前）にあることを意味する。これが頻繁であれば`vm.min_free_kbytes`を確認する。この値を上げるとkswapdにより多くの余裕を与えられる。

```bash
# NUMAノードごとのkswapd起床頻度
bpftrace -e '
tracepoint:vmscan:mm_vmscan_kswapd_wake {
    @[args->nid] = count();
}
interval:s:1 { print(@); clear(@); }'
```

### mm_vmscan_kswapd_sleep {#mmvmscankswapd_sleep}

十分なメモリを回収した後、kswapdが再びスリープするときに発火する。

**フィールド**: `nid`（NUMAノードID）

**診断での使い方**: `mm_vmscan_kswapd_wake`から`mm_vmscan_kswapd_sleep`までの時間が、各回収サイクルの所要時間となる。

### mm_vmscan_direct_reclaim_begin / mm_vmscan_direct_reclaim_end {#mmvmscandirectreclaimbegin-mmvmscandirectreclaimend}

割り当てが**直接回収**を引き起こしたときに発火する。直接回収では、割り当てを行うタスク自身が、処理を進める前にpageを回収しなければならない。直接回収は、アプリケーションから見える割り当てレイテンシを引き起こす。

**フィールド**（begin）:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `order` | `int` | 割り当て中のorder |
| `gfp_flags` | `gfp_t` | 割り当てのGFPフラグ |

**フィールド**（end）:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `nr_reclaimed` | `unsigned long` | 回収されたpage数 |

**診断での使い方**: 直接回収の頻度と所要時間を測定する。頻繁な直接回収はメモリ圧迫の強い兆候であり、カーネルとアプリケーションのコードパスに直接レイテンシを加える。`/proc/vmstat`の`allocstall_normal`（[proc vmstat](understanding-proc-vmstat.md)で説明）と相関を取ること。

```bash
# タスクごとの直接回収レイテンシ（ナノ秒）を測定する
bpftrace -e '
tracepoint:vmscan:mm_vmscan_direct_reclaim_begin {
    @start[tid] = nsecs;
}
tracepoint:vmscan:mm_vmscan_direct_reclaim_end /@start[tid]/ {
    @latency_ns = hist(nsecs - @start[tid]);
    delete(@start[tid]);
}
interval:s:10 { print(@latency_ns); }'
```

### mm_vmscan_lru_isolate {#mmvmscanlru_isolate}

回収の検討対象としてpageがLRUリストから隔離されたときに発火する。

**フィールド**:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `highest_zoneidx` | `int` | 回収対象の最上位zone |
| `order` | `int` | 要求された割り当てのorder |
| `nr_requested` | `unsigned long` | 隔離を要求されたpage数 |
| `nr_scanned` | `unsigned long` | スキャンされたpage数 |
| `nr_skipped` | `unsigned long` | スキップされたpage数（例: ビジーなpage） |
| `nr_taken` | `unsigned long` | 実際に隔離されたpage数 |
| `lru` | `unsigned int` | どのLRUリストか（active/inactive、anon/file） |

**診断での使い方**: `nr_skipped` / `nr_taken`の比率が高い場合、多くのpageが一時的にビジー（ライトバック中、ロック中など）で回収できないことを意味する。これは回収のストールを引き起こしうる。

### mm_vmscan_lru_shrink_inactive {#mmvmscanlrushrinkinactive}

inactive LRUリストが縮小されたとき（pageが回収されるか、参照チェックの結果activeリストへ移される）に発火する。

**フィールド**:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `nid` | `int` | NUMAノード |
| `nr_scanned` | `unsigned long` | inactiveリスト上でスキャンされたpage数 |
| `nr_reclaimed` | `unsigned long` | 実際に解放されたpage数 |
| `nr_dirty` | `unsigned long` | 遭遇したダーティページ数 |
| `nr_writeback` | `unsigned long` | 現在ライトバック中のpage数 |
| `nr_congested` | `unsigned long` | 輻輳した裏側デバイスを待っているpage数 |
| `nr_immediate` | `unsigned long` | 即時回収の対象となるpage数 |
| `nr_activate0` | `unsigned long` | activeリストへ昇格し直した無名page数 |
| `nr_activate1` | `unsigned long` | activeリストへ昇格し直したファイルバックのpage数 |
| `nr_ref_keep` | `unsigned long` | 参照されていたために保持されたpage数 |
| `nr_unmap_fail` | `unsigned long` | アンマップに失敗したpage数 |
| `priority` | `int` | 回収の緊急度（低いほど緊急） |

**診断での使い方**: `nr_dirty`が大きく`nr_reclaimed`が小さい場合、回収がダーティページに突き当たってライトバックを待たなければならないことを意味する。これは回収レイテンシのよくある原因である。`priority`フィールド（開始時の12から、追い詰められると0まで下がる）は、カーネルがどれだけ積極的に回収しようとしているかを示す。

### mm_vmscan_write_folio {#mmvmscanwrite_folio}

!!! note "最近のカーネルでは名前が変わっている"
    このtracepointは元々`mm_vmscan_writepage`という名前だったが、folio化の際に`mm_vmscan_write_folio`に改名された。正確な名前は、実行中のカーネルの`/sys/kernel/debug/tracing/events/vmscan/`で確認すること。

回収パスが、ダーティなpage/folioをスワップまたは裏側のストレージに書き出すと判断したときに発火する。

**フィールド**:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `pfn` | `unsigned long` | 書き出されるfolioのPFN |
| `reclaim_flags` | `int` | ライトバックの文脈を表すフラグ |

**診断での使い方**: 頻度が高い場合、システムがメモリ圧迫のためにスワップへ書き出したり、ダーティなファイルpageを同期したりしていることを意味する。これはI/O駆動の回収であり、アプリケーションのレイテンシに大きく影響する。

---

## OOMのtracepoint {#oom-tracepoints}

**ソース**: [`include/trace/events/oom.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/oom.h)

### oom_score_adj_update {#oomscoreadj_update}

プロセスの`oom_score_adj`の値が（`/proc/PID/oom_score_adj`を介して）変更されたときに発火する。

**フィールド**:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `pid` | `int` | 調整されるプロセスのID |
| `comm` | `char[]` | プロセス名 |
| `oom_score_adj` | `short` | 新しいoom_score_adjの値 |

**診断での使い方**: どのプロセスがいつOOMスコアを調整しているかを監査する。起動時に自分自身を繰り返し-1000（完全に保護）に設定するプロセスがあると、システムにOOMの犠牲者として妥当な候補がいなくなることがある。

### mark_victim {#mark_victim}

OOM killerがkillするプロセスを選択したときに発火する。

**フィールド**:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `pid` | `int` | 犠牲となるプロセスのPID |

**診断での使い方**: これは本番環境の監視において最も有用なOOMのtracepointである。`mark_victim`を購読すれば、OOM killerが発動するたびに、どのプロセスがなぜ選ばれたかを含む構造化されたイベントが得られる。

```bash
# プロセスの詳細付きでOOM killを通知する（RSSの値はkB単位）
bpftrace -e '
tracepoint:oom:mark_victim {
    $rss_kb = args->anon_rss + args->file_rss + args->shmem_rss;
    printf("OOM kill: pid=%d comm=%s rss=%lu kB adj=%d\n",
           args->pid, args->comm, $rss_kb, args->oom_score_adj);
}'
```

OOMデバッグの完全なワークフローについては、[OOMのデバッグ](oom-debugging.md)を参照。

---

## ページフォールトのtracepoint {#page-fault-tracepoints}

カーネルは、mmレベルの汎用的なページフォールトのtracepointを公開していない。ページフォールトの観測はアーキテクチャ固有である。

- **x86/x86_64**: `exceptions:page_fault_user`と`exceptions:page_fault_kernel`
- **ARM64**: `exceptions:page_fault_user`と`exceptions:page_fault_kernel`

```bash
# ユーザー空間のページフォールトを数える（これをサポートするすべてのアーキテクチャ）
perf stat -e exceptions:page_fault_user -a -- sleep 10

# あるいは/proc/vmstatのカウンタを使う（アーキテクチャ非依存）
grep pgfault /proc/vmstat     # マイナーフォールト
grep pgmajfault /proc/vmstat  # メジャーフォールト（I/Oが必要だったもの）
```

ページフォールトの分析には、一般にtracepointよりも`/proc/vmstat`のカウンタ（`pgfault`、`pgmajfault`）やプロセスごとの`/proc/<pid>/stat`のフィールド（minflt、majflt）の方が移植性が高い。

---

## コンパクションのtracepoint {#compaction-tracepoints}

**ソース**: [`include/trace/events/compaction.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/compaction.h)

### mm_compaction_begin / mm_compaction_end {#mmcompactionbegin-mmcompactionend}

コンパクションのパスが開始・終了したときに発火する。

**フィールド**（begin）:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `zone_start` | `unsigned long` | コンパクション対象zoneの開始PFN |
| `migrate_pfn` | `unsigned long` | 現在の移行スキャナの位置 |
| `free_pfn` | `unsigned long` | 現在の空きpageスキャナの位置 |
| `zone_end` | `unsigned long` | 終了PFN |
| `sync` | `bool` | 同期的（ブロッキング）なコンパクションかどうか |

**フィールド**（end）:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `zone_start` | `unsigned long` | zoneの開始PFN |
| `migrate_pfn` | `unsigned long` | 最終的な移行スキャナの位置 |
| `free_pfn` | `unsigned long` | 最終的な空きpageスキャナの位置 |
| `zone_end` | `unsigned long` | zoneの終了PFN |
| `sync` | `bool` | 同期的なコンパクション |
| `status` | `int` | 結果: `COMPACT_SUCCESS`、`COMPACT_PARTIAL_SKIPPED`、`COMPACT_CONTINUE`、`COMPACT_SKIPPED`、`COMPACT_DEFERRED`、`COMPACT_NOT_SUITABLE_ZONE`、`COMPACT_CONTENDED` |

**診断での使い方**: コンパクションの所要時間と成功率を追跡する。`COMPACT_DEFERRED`のステータスが頻繁な場合、カーネルがそのzoneのコンパクションを諦めた（失敗した試行が多すぎる）ことを意味する。`COMPACT_SUCCESS`の後に高orderの割り当てが成功していれば、断片化が根本原因だったことが確認できる。

```bash
# コンパクションはどれくらいの頻度で成功・失敗しているか?
bpftrace -e '
tracepoint:compaction:mm_compaction_end {
    @status[args->status] = count();
}
interval:s:10 { print(@status); }'
```

### mm_compaction_isolate_migratepages / mm_compaction_isolate_freepages {#mmcompactionisolatemigratepages-mmcompactionisolatefreepages}

コンパクション中に、移行対象として、あるいは空きの移行先としてpageが隔離されたときに発火する。

**フィールド**:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `start_pfn` | `unsigned long` | スキャンした範囲の開始 |
| `end_pfn` | `unsigned long` | スキャンした範囲の終了 |
| `nr_scanned` | `unsigned long` | スキャンされたpage数 |
| `nr_taken` | `unsigned long` | 隔離されたpage数 |

**診断での使い方**: `nr_taken / nr_scanned`の比率が低い場合、zone内のほとんどのpageがピン留めされて（移動不可能で）おり、コンパクションできないことを意味する。これはコンパクションでは解決できない根本的な断片化を示しているので、[CMA](cma.md)やhuge pageの予約の変更を検討すること。

### mm_compaction_migratepages {#mmcompactionmigratepages}

隔離されたpageが実際に新しい位置へ移行されたときに発火する。

**フィールド**:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `nr_migrated` | `unsigned long` | 正常に移動されたpage数 |
| `nr_failed` | `unsigned long` | 移行に失敗したpage数 |

---

## huge pageのtracepoint {#huge-page-tracepoints}

**ソース**: [`include/trace/events/huge_memory.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/huge_memory.h)

利用可能なtracepointは、khugepagedのcollapse（まとめ上げ）の活動をカバーする。名前はfolioやファイルTHPの作業に伴って変化するので、完全な一覧は実行中のカーネルの`/sys/kernel/debug/tracing/events/huge_memory/`で確認すること。

### mm_collapse_huge_page {#mmcollapsehuge_page}

khugepagedデーモンがcollapseの試行を終えたとき（成功・失敗を問わず）に発火する。

**フィールド**:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `mm` | `struct mm_struct *` | プロセスのアドレス空間 |
| `isolated` | `int` | collapseのために隔離されたpage数 |
| `status` | `int` | 結果: 0 = 成功、0以外 = 失敗 |

**診断での使い方**: THPのcollapse活動を追跡する。頻繁な失敗（`status != 0`）は、プロセスのメモリが断片化しすぎていてkhugepagedが処理を進められないことを示している可能性がある。collapseされたTHPは`/proc/meminfo`の`AnonHugePages`に現れる。

```bash
# 毎分のTHP collapse数を、結果別に数える
bpftrace -e '
tracepoint:huge_memory:mm_collapse_huge_page {
    @[args->status == 0 ? "success" : "fail"] = count();
}
interval:s:60 { print(@); clear(@); }'
```

### mm_collapse_huge_page_isolate {#mmcollapsehugepageisolate}

collapseの試行の一環としてpageがLRUから隔離されたときに発火する。

**フィールド**: `pfn`、`none_or_zero`、`referenced`、`status`

**診断での使い方**: ここで`status`が0以外の場合、隔離のステップ自体が失敗したことを意味する。pageがビジー（I/O中、ロック中など）で移動できなかったのである。

### mm_khugepaged_scan_pmd {#mmkhugepagedscan_pmd}

khugepagedがcollapseの機会を探してPMDエントリをスキャンしたときに発火する。

**フィールド**: `mm`、`pfn`、`referenced`、`none_or_zero`、`status`、`unmapped`

---

## slabのtracepoint {#slab-tracepoints}

**ソース**: [`include/trace/events/kmem.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/kmem.h)

### kmem_cache_alloc {#kmemcachealloc}

名前付きslabキャッシュからの割り当て（`kmem_cache_alloc()`、`kmem_cache_alloc_node()`）のたびに発火する。

**フィールド**:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `call_site` | `unsigned long` | 呼び出し元の戻りアドレス（命令ポインタ） |
| `ptr` | `const void *` | 割り当てられたオブジェクトへのポインタ |
| `bytes_req` | `size_t` | 要求されたバイト数 |
| `bytes_alloc` | `size_t` | 実際に割り当てられたバイト数（アライメントのため大きくなることがある） |
| `gfp_flags` | `gfp_t` | 割り当てフラグ |
| `node` | `int` | 割り当て元のNUMAノード（-1 = 任意） |

**診断での使い方**: どの割り当て箇所が最も活発か、どのslabキャッシュが最も速く成長しているかを特定する。`call_site`フィールドは生の命令ポインタなので、`addr2line`や`perf script`でシンボル化して関数名を得る。

```bash
# slab割り当ての呼び出し箇所トップ10
bpftrace -e '
tracepoint:kmem:kmem_cache_alloc {
    @[ksym(args->call_site)] = count();
}
interval:s:10 {
    print(@, 10);
    clear(@);
}'
```

### kmem_cache_free {#kmemcachefree}

slabオブジェクトの解放（`kmem_cache_free()`）のたびに発火する。

**フィールド**:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `call_site` | `unsigned long` | 呼び出し元の命令ポインタ |
| `ptr` | `const void *` | 解放されるポインタ |

**診断での使い方**: `kmem_cache_alloc`と組み合わせて、割り当てるだけで決して解放しない割り当て箇所、つまりslabレベルのリークを見つける。

```bash
# slabの正味の収支: 呼び出し箇所ごとの割り当て数 − 解放数
bpftrace -e '
tracepoint:kmem:kmem_cache_alloc { @net[ksym(args->call_site)]++; }
tracepoint:kmem:kmem_cache_free  { @net[ksym(args->call_site)]--; }
interval:s:30 {
    print(@net);  # 正 = 解放より割り当てが多い
    clear(@net);
}'
```

### kmalloc / kfree {#kmalloc-kfree}

**`kmalloc`**は`kmalloc()` / `kzalloc()`による割り当て（汎用slab）で発火する。

**`kfree`**は`kfree()`で発火する。

**フィールド**（kmalloc）:

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `call_site` | `unsigned long` | 呼び出し元のIP |
| `ptr` | `const void *` | 割り当てられたポインタ |
| `bytes_req` | `size_t` | 要求されたバイト数 |
| `bytes_alloc` | `size_t` | 割り当てられたバイト数（次の2のべき乗） |
| `gfp_flags` | `gfp_t` | GFPフラグ |
| `node` | `int` | NUMAノード |

**診断での使い方**: `bytes_req`と`bytes_alloc`の差から内部断片化が分かる。常に33バイトを要求する呼び出し箇所は64バイトを得るので、ほぼ2倍の無駄になる。これは割り当てサイズを調整するか、専用の`kmem_cache`を使うべきだという兆候である。

```bash
# 呼び出し箇所ごとの平均的な無駄の比率を表示する
bpftrace -e '
tracepoint:kmem:kmalloc {
    @req[ksym(args->call_site)]  = sum(args->bytes_req);
    @alloc[ksym(args->call_site)] = sum(args->bytes_alloc);
}
interval:s:15 {
    print(@req);
    print(@alloc);
}'
```

---

## 実践的な診断レシピ {#practical-diagnostic-recipes}

### レシピ1: メモリ圧迫がアプリケーションのレイテンシを引き起こしているか? {#recipe-1-is-memory-pressure-causing-application-latency}

直接回収がプロセスにレイテンシを加えているかどうかを判定する。

```bash
# ステップ1: そもそも直接回収が起きているかを確認する
grep allocstall /proc/vmstat

# ステップ2: 起きていれば、tracepointでその持続時間を測定する
bpftrace -e '
tracepoint:vmscan:mm_vmscan_direct_reclaim_begin {
    @start[tid] = nsecs;
    @task = comm;
}
tracepoint:vmscan:mm_vmscan_direct_reclaim_end /@start[tid]/ {
    $lat = nsecs - @start[tid];
    printf("direct_reclaim: comm=%s latency=%.3fms reclaimed=%lu pages\n",
           @task, $lat / 1e6, args->nr_reclaimed);
    delete(@start[tid]);
}'
```

出力にアプリケーションのタスク名が高いレイテンシ値とともに現れるなら、メモリ圧迫がワークロードに直接レイテンシを加えている。

### レシピ2: メモリ増加の原因を見つける {#recipe-2-finding-the-source-of-memory-growth}

どのコードパスが最も多くのメモリを割り当てているかを特定する。

```bash
# 60秒間に割り当てた総バイト数によるkmalloc呼び出し箇所のランキング
bpftrace -e '
tracepoint:kmem:kmalloc {
    @bytes[ksym(args->call_site)] = sum(args->bytes_alloc);
}
interval:s:60 {
    print(@bytes, 20);  # 上位20件
    exit();
}'
```

増加が予期しないものであれば、[KASAN](kasan.md)と突き合わせること。正当な機能がメモリを割り当てている場合もあれば、リークである場合もある。

### レシピ3: THPのcollapse失敗を診断する {#recipe-3-diagnosing-thp-collapse-failures}

THPが役立っているはずなのに、`/proc/meminfo`の`AnonHugePages`が増えていない場合:

```bash
# collapseの結果を監視する
bpftrace -e '
tracepoint:huge_memory:mm_collapse_huge_page {
    @result[args->status] = count();
}
interval:s:30 {
    printf("\nTHP collapse outcomes (status 0 = success):\n");
    print(@result);
    clear(@result);
}'
```

`status`が0なら成功である。0以外の値は`mm/khugepaged.c`の`SCAN_*` enumの値に対応する（例: `SCAN_ALLOC_HUGE_PAGE_FAIL`、`SCAN_CGROUP_CHARGE_FAIL`）。現在の対応関係はソースで確認すること。collapseが一貫して失敗している場合は、より長期的な視点として`/sys/kernel/mm/transparent_hugepage/khugepaged/pages_collapsed`と`full_scans`も確認する。

### レシピ4: OOM killerを監視する {#recipe-4-watching-the-oom-killer}

すべてのOOM killをプロセスの詳細付きで記録する常駐モニタを設定する。

```bash
bpftrace -e '
tracepoint:oom:mark_victim {
    $rss_kb = args->anon_rss + args->file_rss + args->shmem_rss;
    time("%H:%M:%S ");
    printf("OOM KILL pid=%d comm=%s rss=%lu kB (%lu MB) adj=%d\n",
           args->pid,
           args->comm,
           $rss_kb,
           $rss_kb / 1024,
           args->oom_score_adj);
}'
```

OOMイベント後の調査ワークフロー全体については、[OOMのデバッグ](oom-debugging.md)を参照。

### レシピ5: コンパクションのヘルスチェック {#recipe-5-compaction-health-check}

コンパクションが高orderの割り当て需要に追いついているかどうかを判定する。

```bash
# 60秒間実行してから報告する
bpftrace -e '
tracepoint:compaction:mm_compaction_end {
    @[args->status] = count();
}
tracepoint:kmem:mm_page_alloc_extfrag {
    @extfrag_total = count();
    if (args->change_ownership) {
        @ownership_stolen = count();
    }
}
interval:s:60 {
    printf("\nCompaction outcomes:\n");
    print(@);
    printf("\nFragmentation events: %d total, %d ownership changes\n",
           @extfrag_total, @ownership_stolen);
    exit();
}'
```

`@ownership_stolen`が高く、かつ`COMPACT_DEFERRED`の結果が頻繁な場合は、`vm.min_free_kbytes`のチューニングや、移動不可能な割り当てを減らすワークロードの変更を検討すべき強い兆候である。

---

## tracepointクイックリファレンス {#tracepoint-quick-reference}

| カテゴリ | tracepoint | 主な用途 |
|----------|-----------|---------|
| 割り当て | `kmem:mm_page_alloc` | ページ割り当ての頻度とorderの分布 |
| 割り当て | `kmem:mm_page_free` | ページ解放の頻度（allocと組み合わせて正味の増加を見る） |
| 割り当て | `kmem:mm_page_alloc_extfrag` | 断片化: migratetypeの盗用 |
| 割り当て | `kmem:mm_page_alloc_zone_locked` | PCPリストの枯渇 |
| 回収 | `vmscan:mm_vmscan_kswapd_wake` | バックグラウンド回収の圧力 |
| 回収 | `vmscan:mm_vmscan_direct_reclaim_begin/end` | 回収による割り当てレイテンシ |
| 回収 | `vmscan:mm_vmscan_lru_shrink_inactive` | 回収の効率（ダーティページによるボトルネック） |
| 回収 | `vmscan:mm_vmscan_write_folio` | 回収によるスワップ/ライトバックの頻度（folio以前は`mm_vmscan_writepage`） |
| OOM | `oom:mark_victim` | OOM kill: 犠牲者の選択 |
| OOM | `oom:oom_score_adj_update` | OOMスコア操作の監査 |
| コンパクション | `compaction:mm_compaction_begin/end` | コンパクションの所要時間と成功 |
| コンパクション | `compaction:mm_compaction_migratepages` | コンパクションのパスごとに移動されたpage数 |
| huge page | `huge_memory:mm_collapse_huge_page` | khugepagedのcollapseの結果 |
| huge page | `huge_memory:mm_collapse_huge_page_isolate` | collapseにおけるpage隔離のステップ |
| huge page | `huge_memory:mm_khugepaged_scan_pmd` | khugepagedのPMDスキャン活動 |
| slab | `kmem:kmem_cache_alloc` | 呼び出し箇所ごとの名前付きキャッシュの割り当て |
| slab | `kmem:kmem_cache_free` | 名前付きキャッシュの解放 |
| slab | `kmem:kmalloc` | 呼び出し箇所とサイズごとの汎用kmalloc |
| slab | `kmem:kfree` | kfreeの呼び出し箇所 |

---

## 主要なソースファイル {#key-source-files}

| ファイル | 説明 |
|------|-------------|
| [`include/trace/events/kmem.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/kmem.h) | ページアロケータとslabのtracepoint定義 |
| [`include/trace/events/vmscan.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/vmscan.h) | 回収のtracepoint定義 |
| [`include/trace/events/compaction.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/compaction.h) | コンパクションのtracepoint定義 |
| [`include/trace/events/huge_memory.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/huge_memory.h) | THP/huge pageのtracepoint定義 |
| [`include/trace/events/oom.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/oom.h) | OOMのtracepoint定義 |
| [`include/trace/events/mmflags.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/mmflags.h) | tracepointの出力で使われるGFPフラグ名の文字列 |
| [`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) | ページアロケータ — `trace_mm_page_alloc()`などを呼び出す |
| [`mm/vmscan.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) | 回収エンジン — vmscanのトレースイベントを呼び出す |
| [`mm/compaction.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/compaction.c) | コンパクション — compactionのトレースイベントを呼び出す |
| [`mm/khugepaged.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/khugepaged.c) | khugepagedデーモン — huge_memoryのトレースイベントを呼び出す |

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/trace/events/mmflags.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/mmflags.h) — tracepointの出力の`gfp_flags`フィールドをデコードするのに使われるGFPフラグ名の文字列（`gfp_flag_names[]`）
- [include/trace/events/kmem.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/kmem.h) — ページアロケータとslabのtracepoint定義
- [include/trace/events/vmscan.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/vmscan.h) — 回収のtracepoint定義
- [include/trace/events/compaction.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/compaction.h) — コンパクションのtracepoint定義
- [include/trace/events/huge_memory.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/huge_memory.h) — THPとkhugepagedのtracepoint定義

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/trace/ftrace.rst`](https://docs.kernel.org/trace/ftrace.html) — ftrace基盤の完全なリファレンス: リングバッファ、フィルタ構文、関数トレーシング
- [`Documentation/trace/tracepoints.rst`](https://docs.kernel.org/trace/tracepoints.html) — `TRACE_EVENT()`によるtracepointの定義方法と、新しいtracepointの追加方法
- [`Documentation/trace/events.rst`](https://docs.kernel.org/trace/events.html) — `/sys/kernel/debug/tracing/events/`を介したイベントの有効化とフィルタリング
- [`Documentation/bpf/index.rst`](https://docs.kernel.org/bpf/index.html) — `tracepoint`プログラムタイプを含むBPFサブシステムのドキュメント

### 関連ページ {#related-pages}

- [understanding-proc-vmstat.md](understanding-proc-vmstat.md) — 個々のtracepointが観測する内容を要約する集計カウンタ
- [oom-debugging.md](oom-debugging.md) — OOM後の調査で`oom:mark_victim`とPSIを組み合わせて使う
- [why-is-my-process-slow.md](why-is-my-process-slow.md) — `mm_vmscan_direct_reclaim_begin/end`を使ってレイテンシをメモリ圧迫に帰属させる
- [kasan.md](kasan.md) — use-after-freeを探す際に割り当てのトレーシングと組み合わせるメモリバグ検出

### LWNの記事 {#lwn-articles}

- [LWN: Tracepoints in the Linux kernel](https://lwn.net/Articles/379903/) — `TRACE_EVENT()`マクロと、静的tracepointの背後にある設計思想の最初の紹介
- [LWN: Dynamic tracing with BPF](https://lwn.net/Articles/740157/) — `bpftrace`とBCCを使い、低オーバーヘッドでカーネルのtracepointにアタッチする
