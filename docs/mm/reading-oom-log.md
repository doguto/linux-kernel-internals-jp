# OOMログの読み方 {#reading-an-oom-log}

> OOM（Out-of-Memory）キラーのメッセージを読み、診断する方法

## OOMログはどのようなものか {#what-does-an-oom-log-look-like}

カーネルがメモリを使い果たし、十分なpageを回収できないとき、OOMキラーが起動される。ログは [`mm/oom_kill.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/oom_kill.c) の関数、主に `dump_header()`、`dump_tasks()`、`__oom_kill_process()` によって生成される。

ログは4つのセクションからなる。

1. **トリガー行** — 誰がなぜOOMを起動したか
2. **メモリ状態のダンプ** — システムまたはcgroupのメモリ統計
3. **タスク表** — 全プロセスとそのメモリ使用量
4. **キル行** — どのプロセスが選ばれ、キルされたか

## 注釈付きの例: システム全体のOOM {#annotated-example-system-wide-oom}

```
[  621.423101] python3 invoked oom-killer: gfp_mask=0x140cca(GFP_HIGHUSER_MOVABLE|__GFP_COMP), order=0, oom_score_adj=0
```

| フィールド | 意味 |
|-------|---------|
| `python3` | 割り当てに失敗したプロセス（犠牲者とは限らない） |
| `gfp_mask=0x140cca` | 割り当てフラグ — `GFP_HIGHUSER_MOVABLE` はユーザー空間のpage割り当てを意味する |
| `order=0` | 単一のpageを要求した（2^0 = 1 page）。それより高いorderは連続割り当てを意味する。 |
| `oom_score_adj=0` | *トリガーした*プロセス（犠牲者ではない）のOOM調整値 |

### スタックトレース {#stack-trace}

```
[  621.423118]  <TASK>
[  621.423137]  oom_kill_process+0x101/0x170
[  621.423143]  out_of_memory+0x116/0x570
[  621.423149]  __alloc_pages_slowpath+0xc9a/0xe00
[  621.423168]  handle_pte_fault+0xc6/0x260
[  621.423182]  do_user_addr_fault+0x1c5/0x660
[  621.423187]  exc_page_fault+0x6e/0x160
```

これは割り当ての経路を示している。ここでは、ページフォールト（`exc_page_fault`）が匿名pageの割り当て（`handle_pte_fault`）を引き起こし、それが低速パスに入ってOOMに至った。最もよくある経路は次のとおりである。

| スタックの特徴 | 原因 |
|----------------|-------|
| `do_anonymous_page` / `handle_pte_fault` | プロセスが新しいヒープ/スタックのメモリに触れた |
| `do_read_fault` / `filemap_fault` | 新しいページキャッシュのpageを必要とするファイルを読んだ |
| フォールト経路のない `__alloc_pages` | カーネルサブシステムによる割り当て（slab、ネットワークなど） |

### メモリ状態 {#memory-state}

```
[  621.423200] active_anon:1932541 inactive_anon:12038 isolated_anon:0
[  621.423200]  active_file:524 inactive_file:618 isolated_file:0
[  621.423200]  unevictable:0 dirty:0 writeback:0
[  621.423200]  slab_reclaimable:6592 slab_unreclaimable:11840
[  621.423200]  mapped:1296 shmem:396 pagetables:7912
[  621.423200]  free:3248 free_pcp:112 free_cma:0
```

値はすべて**page**単位である（通常は1つ4KB）。確認すべき主な点は次のとおり。

| フィールド | この例 | わかること |
|-------|-------------|-------------------|
| `active_anon` | 1,932,541（約7.4 GB） | 匿名メモリが支配的 — あるアプリケーションがRAMの大半を消費している |
| `active_file + inactive_file` | 1,142（約4.5 MB） | ページキャッシュがほぼ完全に追い出されている — ファイルデータの余地がない |
| `free` | 3,248（約12.7 MB） | 危機的に少ない |
| `slab_unreclaimable` | 11,840（約46 MB） | 解放できないカーネルメモリ |

### ゾーンごとのウォーターマーク {#per-zone-watermarks}

```
[  621.423225] Node 0 Normal free:11984kB min:11408kB low:14260kB high:17112kB
```

空き（11,984kB）はminウォーターマーク（11,408kB）をかろうじて上回っている。これはシステムが回収を尽くしたことを示している。OOMが発動するのは、カーネルがすでに直接回収を試み、要求を満たすのに十分なpageを解放できず、割り当てが失敗したときである。

```
[  621.423225] ... all_unreclaimable? yes
```

`all_unreclaimable? yes` は、カーネルがこのノードからの回収を諦めたことを意味する。

### スワップの状態 {#swap-state}

```
[  621.423252] Free swap  = 0kB
[  621.423253] Total swap = 0kB
```

| 状況 | 意味 |
|-----------|---------|
| `Total swap = 0kB` | スワップが設定されていない — 匿名pageはまったく回収できない |
| Totalが非ゼロで `Free swap = 0kB` | スワップはあるが満杯 — すべての選択肢を使い果たした |
| `Free swap` に空きがある | スワップが使えるのにOOM — おそらくcgroupの制限かGFPの制約 |

### タスク表 {#task-table}

```
[  621.423260] Tasks state (memory values in pages):
[  621.423262] [  pid  ]   uid  tgid total_vm      rss rss_anon rss_file rss_shmem pgtables_bytes swapents oom_score_adj name
[  621.423268] [    567]     0   567    13876     1120      460      660         0       81920        0             0 systemd-journal
[  621.423275] [    601]     0   601     5765      948      512      436         0       69632        0         -1000 systemd-udevd
[  621.423296] [   8340]  1000  8340  1956284  1920568  1920012      460        96    15466496        0             0 python3
```

| 列 | 意味 |
|--------|---------|
| `total_vm` | 仮想アドレス空間（page数） |
| `rss` | 常駐page（実際にRAM上にある） |
| `rss_anon` | 匿名の常駐page（ヒープ、スタック） |
| `rss_file` | ファイルに裏付けられた常駐page（ライブラリ、mmapされたファイル） |
| `rss_shmem` | 共有メモリの常駐page |
| `pgtables_bytes` | このプロセスのページテーブルのメモリ |
| `swapents` | スワップ上のpage |
| `oom_score_adj` | 手動のOOM優先度（-1000 = 免除、+1000 = 最初にキル） |

この例では、PID 8340（`python3`）が1,920,568の常駐page（約7.3 GB）を持ち、そのほぼすべてが匿名である。明らかな外れ値である。

### キル行 {#kill-line}

```
[  621.423315] oom-kill:constraint=CONSTRAINT_NONE,nodemask=(null),global_oom,task_memcg=/user.slice/...,task=python3,pid=8340,uid=1000
[  621.423320] Out of memory: Killed process 8340 (python3) total-vm:7825136kB, anon-rss:7680048kB, file-rss:1840kB, shmem-rss:384kB, UID:1000 pgtables:15104kB oom_score_adj:0
```

| フィールド | 意味 |
|-------|---------|
| `constraint=CONSTRAINT_NONE` | システム全体のOOM（cpuset、NUMA、cgroupによる制約ではない） |
| `global_oom` | memcgスコープのOOMではない |
| `Out of memory:` プレフィックス | システム全体（cgroupのOOMでは `Memory cgroup out of memory:`） |
| `anon-rss:7680048kB` | プロセスの匿名メモリ（実際の消費量） |

## スコアリングのアルゴリズム {#the-scoring-algorithm}

犠牲者は [`oom_badness()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/oom_kill.c) によって選ばれる。

```
score = RSS + swap entries + page table pages
      + (oom_score_adj * totalpages / 1000)
```

| 要素 | 数えるもの |
|-----------|---------------|
| RSS | `rss_anon + rss_file + rss_shmem` — 常駐page |
| スワップエントリ | このプロセスがスワップ上に持つpage |
| ページテーブル | `mm_pgtables_bytes / PAGE_SIZE` |
| 調整値 | `oom_score_adj * totalpages / 1000` |

特殊なケース:

- `oom_score_adj = -1000`: プロセスは免除される（決して選ばれない）
- `oom_score_adj = 1000`: スコアに `totalpages` が加算される（確実に犠牲者になる）
- カーネルスレッドとinit（PID 1）は決してキルされない

`/proc/<pid>/oom_score` で見えるスコアは、この計算を正規化したもので、読み出し時に計算される。

## cgroupのOOMとシステム全体のOOM {#cgroup-oom-vs-system-wide-oom}

### 見分け方 {#how-to-tell-the-difference}

| | システム全体 | cgroup |
|---|---|---|
| キルのプレフィックス | `Out of memory:` | `Memory cgroup out of memory:` |
| 制約 | `CONSTRAINT_NONE` | `CONSTRAINT_MEMCG` |
| メモリダンプ | ゾーン/ノードの統計を含む完全な `Mem-Info:` | cgroupのメモリ統計（形式はcgroup v1とv2で異なる） |
| タスク表 | 全プロセス | cgroup内のプロセスのみ |
| スコアリングの `totalpages` | システムのRAM + スワップ | cgroupの `memory.max` |

### cgroupのOOMの例（主な違い） {#cgroup-oom-example-key-differences}

**cgroup v2**（現代のカーネルのデフォルト。[`mm/memcontrol.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c) の `mem_cgroup_print_oom_meminfo()` による）:

```
# cgroup のメモリ統計
[  891.112245] Memory cgroup stats for /myapp.slice/myapp.service:
[  891.112248] anon 536346624
[  891.112248] file 262144
[  891.112248] kernel 2158592
...

# 制約は MEMCG と cgroup のパスを示す
[  891.112285] oom-kill:constraint=CONSTRAINT_MEMCG,...,oom_memcg=/myapp.slice/myapp.service,...

# キルのプレフィックスが異なる
[  891.112290] Memory cgroup out of memory: Killed process 12001 (python3) ...
```

**cgroup v1**は、`usage`、`limit`、`failcnt` フィールドを持つ異なる形式で表示される。cgroup v2では、メモリ統計はカウンタ名とバイト値として出力される。

### memory.oom.group {#memoryoomgroup}

cgroupに `memory.oom.group = 1` が設定されていると、OOMキラーは1つだけでなく、そのcgroup内の**すべての**プロセスをキルする。追加の行が現れる。

```
[  891.112295] Tasks in /myapp.slice/myapp.service are going to be killed due to memory.oom.group set
```

これはcgroup階層をさかのぼって `oom_group` が設定された最上位の祖先を見つけ、そのサブツリー内のすべてのプロセス（`oom_score_adj = -1000` のものを除く）をキルする。

## よくあるOOMのパターン {#common-oom-patterns}

### パターン1: メモリリーク {#pattern-1-memory-leak}

**特徴**: 1つのプロセスのRSSが他のすべてより劇的に大きい。`rss_anon` が支配的。

```
[  12340]  1000 12340  4521984  3906712  3906200      440        72    31457280        0             0 leaky-app
[    567]     0   567    13876     1120      460      660         0       81920        0             0 systemd-journal
```

**診断**: `leaky-app` のRSSが約14.9 GBなのに対し、他はすべて5 MB未満。ヒープが際限なく成長している。

**対処**: リークを修正する。一時的な措置として、cgroupの `memory.max` を設定して封じ込める。

### パターン2: フォーク爆弾 {#pattern-2-fork-bomb}

**特徴**: 何百もの同一の小さなプロセス。`pagetables` と `slab_unreclaimable` が高い。

```
[  15001]  1000 15001     2710      892      504      388         0       61440        0             0 bomb.sh
[  15002]  1000 15002     2710      892      504      388         0       61440        0             0 bomb.sh
... (hundreds more)
```

**診断**: プロセスごとのカーネルのオーバーヘッド（スタック、ページテーブル、task_struct）が積み重なる。OOMは1つだけをキルするかもしれず、フォーク爆弾はそれを再生成する。

**対処**: `ulimit -u`（最大プロセス数）、cgroupの `pids.max`、または `systemd` の `TasksMax=` を使う。

### パターン3: 正当なワークロードで、RAMが足りない {#pattern-3-legitimate-workload-not-enough-ram}

**特徴**: 複数のプロセスが妥当なRSSを持ち、突出した外れ値がない。`active_file + inactive_file` がほぼゼロ。スワップがないか、満杯。

```
[   1200]     0  1200   524288   262100   261800      220        80     2097152        0             0 postgres
[   1201]     0  1201   524288   261984   261700      200        84     2097152        0             0 postgres
[   2001]  1000  2001   131072    65500    65200      240        60      524288        0             0 redis-server
```

**診断**: どのプロセスも実際の仕事をしている。システムは本当にもっと多くのメモリを必要としている。

**対処**: RAMを増やすか、スワップを追加するか、ワークロードを複数のホストに分散する。

### パターン4: カーネルのslabリーク {#pattern-4-kernel-slab-leak}

**特徴**: `slab_unreclaimable` が非常に大きい。タスク表のRSSでは使用メモリの大半を説明できない。

```
slab_reclaimable:8192 slab_unreclaimable:524288   <- 2 GB unreclaimable slab
```

**診断**: カーネルのデータ構造がリークしている。RAMの総量と全プロセスのRSSの合計との差は、カーネルのslabにある。

**対処**: `slabtop` または `/proc/slabinfo` を使って、どのキャッシュが増加しているかを特定する。よくある原因: `dentry`、`inode_cache`、NFSのinodeキャッシュ。

## 答えるべき5つの問い {#five-questions-to-answer}

どのOOMログを読むときも、次の問いに答えること。

| 問い | 見る場所 |
|----------|--------------|
| システム全体のOOMか、cgroupのOOMか？ | キルのプレフィックスとconstraintフィールド |
| 何が最もメモリを消費したか？ | タスク表の最大の `rss` |
| スワップは利用可能だったか？ | `Free swap` / `Total swap` の行 |
| OOMの前にシステムはスラッシングしていたか？ | `active_file + inactive_file` がほぼゼロ、`all_unreclaimable? yes` |
| 防げたか？ | 下の表を参照 |

| 発見 | 予防策 |
|---------|-----------|
| スワップが設定されていない | 緊急時の余裕としてスワップを追加する |
| 1つのプロセスのRSSが巨大 | cgroupの `memory.max`、またはリークを修正する |
| 重要でないプロセスに `oom_score_adj=-1000` | OOM保護を監査する — すべてを保護してはならない |
| `slab_unreclaimable` が支配的 | カーネルのslabリークを調査する |
| ページキャッシュがほぼゼロ、I/O中心のワークロード | RAMを増やす — システムは本当にもっと必要としている |
| `min_free_kbytes` が低い | [`vm.min_free_kbytes`](https://docs.kernel.org/admin-guide/sysctl/vm.html) を増やして、kswapdを早めに起動させる |

## 試してみる {#try-it-yourself}

```bash
# 過去の OOM イベントを見る
dmesg | grep -i "out of memory"
dmesg | grep "oom-kill"
dmesg | grep "oom_kill"

# 全プロセスの現在の OOM スコアを確認する
for pid in /proc/[0-9]*/; do
    score=$(cat ${pid}oom_score 2>/dev/null)
    adj=$(cat ${pid}oom_score_adj 2>/dev/null)
    name=$(cat ${pid}comm 2>/dev/null)
    [ -n "$score" ] && echo "$score $adj $name"
done | sort -rn | head -20

# トレーシングで OOM キルのイベントを監視する（oom:mark_victim は実際のキルをトレースする）
echo 1 > /sys/kernel/debug/tracing/events/oom/mark_victim/enable
cat /sys/kernel/debug/tracing/trace_pipe

# cgroup 内で制御された OOM を起こす（テスト用）
mkdir -p /sys/fs/cgroup/test-oom
echo 50M > /sys/fs/cgroup/test-oom/memory.max
echo $$ > /sys/fs/cgroup/test-oom/cgroup.procs
stress --vm 1 --vm-bytes 100M --timeout 10s
```

## 主要なソースファイル {#key-source-files}

| ファイル | 含まれるもの |
|------|-----------------|
| [`mm/oom_kill.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/oom_kill.c) | スコアリング、選択、キル、ログ出力 |
| [`mm/memcontrol.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c) | cgroupのOOM: `mem_cgroup_print_oom_meminfo()`、`mem_cgroup_get_oom_group()` |
| [`mm/show_mem.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/show_mem.c) | Mem-Infoダンプを生成する `__show_mem()` |
| [`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) | `__alloc_pages_may_oom()` のエントリポイント、ゾーンの統計 |

## 参考文献 {#further-reading}

- [メモリ不足](oom.md) — OOMのライフサイクル全体（ウォーターマークから回収を経てOOMまで）
- [/proc/meminfo](proc-meminfo.md) — メモリ状態のフィールドを理解する
- [/proc/vmstat](proc-vmstat.md) — 回収と逼迫のカウンタ
- [ページ回収](reclaim.md) — OOMが発動する前に何が起きるか
- [メモリcgroup](memcg.md) — cgroupのメモリ制限とOOMの挙動
- [vm sysctlのドキュメント](https://docs.kernel.org/admin-guide/sysctl/vm.html) — チューニング可能なパラメータ
