# /proc/vmstatを理解する {#understanding-procvmstat}

> カーネルのメモリ管理カウンタを使うための実践ガイド

*これはシナリオに基づく診断ガイドである。フィールドごとのリファレンスについては[/proc/vmstatリファレンス](proc-vmstat.md)を参照してほしい。*

## /proc/vmstatとは何か? {#what-is-procvmstat}

`/proc/vmstat`は、カーネル内部のメモリ管理カウンタを公開する疑似ファイルである。カーネルはpageを割り当てる、メモリを回収する、スワップする、コンパクションする、フォールトを処理するたびに、カウンタを1つ増やす。`/proc/vmstat`を使えば、それらすべてを読むことができる。

```
$ cat /proc/vmstat | head -20
nr_free_pages 234567
nr_zone_inactive_anon 45678
nr_zone_active_anon 123456
nr_zone_inactive_file 89012
nr_zone_active_file 345678
nr_zone_unevictable 1234
nr_mlock 1234
nr_slab_reclaimable 56789
nr_slab_unreclaimable 12345
...
pgfault 987654321
pgmajfault 12345
pgfree 876543210
pgactivate 23456789
pgdeactivate 12345678
pgscan_kswapd 34567890
pgscan_direct 1234567
pgsteal_kswapd 33456780
pgsteal_direct 1134567
```

このファイルの実装は[`mm/vmstat.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmstat.c)にあり、すべてのノードとゾーンにわたってCPUごとのカウンタを集計する。

## 決定的な区別: ゲージとカウンタ {#the-critical-distinction-gauges-vs-counters}

これは`/proc/vmstat`について理解すべき最も重要な点である。根本的に異なる2種類の値が存在する。

### ゲージ（`nr_*`）: 現在の状態 {#gauges-nr_-current-state}

ゲージは**現在の**値を報告する。値は増えたり減ったりする。1回読むだけで有用な情報が得られる。

```
nr_free_pages 234567       ← 今この瞬間、234567 pageが空いている
nr_active_anon 123456      ← 今この瞬間、123456の無名pageがアクティブである
nr_slab_reclaimable 56789  ← 今この瞬間、56789のslab pageが回収可能である
```

ゲージは温度計のようなものだと考えればよい。読めば*今の*温度がわかる。

### カウンタ（`pg*`、`pswp*`、`thp_*`など）: ブート以降単調増加 {#counters-pg-pswp-thp_-etc-monotonic-since-boot}

カウンタはシステムのブート以降の**累積合計**を報告する。値は増える一方である。1回読んだだけではほとんど何もわからない。レートを計算するには**2回の読み取りと時間差**が必要である。

```
pgfault 987654321          ← ブート以降9億8700万回のページフォールト
pgmajfault 12345           ← ブート以降12,345回のメジャーフォールト
pswpout 567890             ← ブート以降567,890 pageがスワップアウトされた
```

カウンタは走行距離計のようなものだと考えればよい。合計で5万マイル走ったと知るより、直近1時間で100マイル走ったと知るほうが役に立つ。

```
Gauges (nr_*)              Counters (pg*, pswp*, thp_*, etc.)
┌──────────────┐           ┌──────────────┐
│ Current      │           │ Cumulative   │
│ snapshot     │           │ since boot   │
│              │           │              │
│ One read     │           │ Need delta:  │
│ = useful     │           │ (v2-v1)/dt   │
│              │           │ = rate       │
│ Goes up      │           │              │
│ AND down     │           │ Only goes up │
└──────────────┘           └──────────────┘
```

!!! warning "最もよくある間違い"
    `pgmajfault 50000`を見てパニックになること。これは**ブート以降**（おそらく何週間も前から）の50,000回のメジャーフォールトである。重要なのは*レート*、つまり今この瞬間に毎秒何回起きているかである。

## シナリオ別の主要カウンタ {#key-counters-by-scenario}

### メモリ逼迫 {#memory-pressure}

システムがメモリ逼迫下にあると疑われるときに注目すべきカウンタは次のとおりである。

| カウンタ | 種類 | 意味 |
|---------|------|---------------|
| `allocstall_normal` | カウンタ | プロセスがNormalゾーンのために直接回収でブロックした |
| `allocstall_movable` | カウンタ | プロセスがMovableゾーンのために直接回収でブロックした |
| `allocstall_dma32` | カウンタ | プロセスがDMA32ゾーンのために直接回収でブロックした |
| `pgscan_direct` | カウンタ | 直接回収でスキャンされたpage（プロセスは停止している） |
| `pgscan_kswapd` | カウンタ | kswapdでスキャンされたpage（バックグラウンド、健全） |
| `pgmajfault` | カウンタ | メジャーページフォールト（ディスクI/Oが必要だった） |
| `nr_free_pages` | ゲージ | 現在の空きpage |

**注目すべき点**:

- `allocstall_*`の増加は、プロセスがメモリ割り当てで**ブロックしている**ことを意味する。これはメモリ逼迫の最も明確なシグナルである。
- `pgscan_direct`のレートの上昇は、[直接回収](reclaim.md)が動いていること、つまりkswapdが追いつけていないことを意味する。
- `pgscan_kswapd`と`pgscan_direct`を比較する。健全なシステムでは、スキャンのほとんどすべてはkswapdが行う。直接回収が支配的なら問題がある。

### スワップの活動 {#swap-activity}

| カウンタ | 種類 | 意味 |
|---------|------|---------------|
| `pswpin` | カウンタ | ディスクからスワップインされたpage |
| `pswpout` | カウンタ | ディスクへスワップアウトされたpage |
| `nr_zone_inactive_anon` | ゲージ | 非アクティブLRU上の無名page（スワップの候補） |

**注目すべき点**:

- `pswpout`のレートが持続しているのは、システムがpageを活発に[スワップアウト](swapping.md)していることを意味する。逼迫下ではある程度のスワップアウトは正常である。
- `pswpin`のレートが持続しているのは、システムがpageを**スワップインして戻している**こと、つまり追い出したpageが再び使われていることを意味する。これは痛いケースである。ワーキングセットがRAMに収まっていないことを意味するからだ。
- 高い`pswpin`と高い`pswpout`が同時に起きている = **スワップスラッシング**。システムは同じpageを行ったり来たりさせている。

### 回収の効率 {#reclaim-efficiency}

| カウンタ | 種類 | 意味 |
|---------|------|---------------|
| `pgsteal_kswapd` | カウンタ | kswapdが回収に成功したpage |
| `pgsteal_direct` | カウンタ | 直接回収が回収に成功したpage |
| `pgscan_kswapd` | カウンタ | kswapdがスキャンしたpage |
| `pgscan_direct` | カウンタ | 直接回収がスキャンしたpage |

**回収効率** = `pgsteal / pgscan`

この比率は回収がどれだけ生産的かを示す。カーネルが100 pageを回収するために1000 pageをスキャンしているなら、効率は10%である。ほとんどのpageは使用中で解放できない。これは悪い状態である。

```
Efficiency = pgsteal_kswapd / pgscan_kswapd

  > 50%   Good — most scanned pages are reclaimable
  10-50%  Warning — many pages are in active use
  < 10%   Bad — the system is desperately scanning
```

これらのしきい値は運用上の経験則であり、カーネルが定めた値ではない。大まかな目安として扱うこと。最も重要なのは個々の値ではなく、*傾向*（時間とともに効率が下がっているか）である。

!!! note "なぜ低い効率が問題なのか"
    回収効率が低いということは、カーネルがわずかな成果のために多くの作業（LRUリストのスキャン、pageフラグの確認、ロックの取得）をしていることを意味する。これはCPUを浪費し、割り当てのレイテンシを増やす。

### THP（Transparent Huge Pages）の健全性 {#thp-transparent-huge-pages-health}

| カウンタ | 種類 | 意味 |
|---------|------|---------------|
| `thp_fault_alloc` | カウンタ | ページフォールト時のTHP割り当ての成功 |
| `thp_fault_fallback` | カウンタ | THP割り当てが失敗し、4KBのpageにフォールバックした |
| `thp_collapse_alloc` | カウンタ | THPへの集約（collapse）の成功（khugepaged） |
| `thp_collapse_alloc_failed` | カウンタ | THPへの集約の失敗 |
| `thp_split_page` | カウンタ | 4KBに分割し直さなければならなかったTHP |

**注目すべき点**:

- 比率`thp_fault_fallback / (thp_fault_alloc + thp_fault_fallback)`が[THP](thp.md)の失敗率である。失敗率が高いということは、システムが連続した2MBの塊を見つけられないことを意味する。
- `thp_split_page`の上昇は、THPが分割されていることを意味する。部分的なmunmapやメモリ逼迫が原因の可能性がある。
- `thp_collapse_alloc_failed`が上昇しているなら、khugepagedが4KBのpageをTHPに昇格させようとして失敗している。

### コンパクションの健全性 {#compaction-health}

| カウンタ | 種類 | 意味 |
|---------|------|---------------|
| `compact_stall` | カウンタ | プロセスがコンパクションを待ってブロックした |
| `compact_fail` | カウンタ | コンパクションを試みたが失敗した |
| `compact_success` | カウンタ | コンパクションが成功した |
| `compact_migrate_scanned` | カウンタ | コンパクション中にマイグレーションのためにスキャンされたpage |
| `compact_free_scanned` | カウンタ | コンパクション中にスキャンされた空きpage |

**注目すべき点**:

- `compact_stall`の増加は、プロセスが[コンパクション](compaction.md)で**ブロックしている**ことを意味する。これは割り当てにレイテンシを加える。
- 失敗率`compact_fail / (compact_fail + compact_success)`は、コンパクションがどれくらいの頻度でうまくいくかを示す。失敗率が高いということは、メモリの断片化がひどすぎることを意味する。
- `compact_migrate_scanned`のレートが高く、かつ`compact_fail`も高い場合、カーネルは連続ブロックを生み出せないまま高くつく作業（スキャン、pageのマイグレーション）をしている。

## カウンタの差分からレートを計算する {#computing-rates-from-counter-deltas}

カウンタは単調増加なので、レートを計算するには2つのサンプルと時間間隔が必要である。

```
rate = (value_t2 - value_t1) / (t2 - t1)
```

### 手動で2回サンプリングする方法 {#manual-two-sample-method}

```bash
# サンプル1
T1=$(date +%s)
V1=$(awk '/^pgmajfault / {print $2}' /proc/vmstat)

sleep 10

# サンプル2
T2=$(date +%s)
V2=$(awk '/^pgmajfault / {print $2}' /proc/vmstat)

# レート
echo "Major faults/sec: $(( (V2 - V1) / (T2 - T1) ))"
```

### 複数のカウンタを経時的に監視する {#watch-multiple-counters-over-time}

```bash
# 主要カウンタを5秒ごとに差分付きで表示する
while true; do
    awk '
    /^(pgscan_direct|pgsteal_direct|allocstall|pgmajfault|pswpin|pswpout) / {
        print $1, $2
    }' /proc/vmstat
    echo "---"
    sleep 5
done
```

実際の差分を計算する、よりきれいな方法は次のとおりである。

```bash
#!/bin/bash
# vmstat-delta.sh — 主要カウンタの毎秒レートを表示する
INTERVAL=${1:-5}
COUNTERS="pgscan_direct pgsteal_direct pgscan_kswapd pgsteal_kswapd allocstall_normal pgmajfault pswpin pswpout"

declare -A prev
first=1

while true; do
    while read name value; do
        for c in $COUNTERS; do
            if [[ "$name" == "$c" ]]; then
                if [[ $first -eq 0 ]]; then
                    delta=$(( value - prev[$name] ))
                    rate=$(( delta / INTERVAL ))
                    printf "%-25s %8d/s\n" "$name" "$rate"
                fi
                prev[$name]=$value
            fi
        done
    done < /proc/vmstat

    [[ $first -eq 0 ]] && echo "---"
    first=0
    sleep "$INTERVAL"
done
```

## 一般的なツールとの連携 {#integration-with-common-tools}

### vmstatコマンド {#vmstat-command}

（procps-ngの）`vmstat`コマンドは内部ですでに`/proc/vmstat`を読んでいるが、簡略化・集約されたビューを表示する。

```bash
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0  12340 234567  89012 456789    0    0    12    34  567  890  5  2 92  1  0
```

| vmstatの列 | /proc/vmstatのソース |
|---------------|---------------------|
| `si`（スワップイン） | `pswpin`の差分 |
| `so`（スワップアウト） | `pswpout`の差分 |
| `free` | `nr_free_pages` * pageサイズ |

`vmstat`は手早く概要をつかむのに適している。`/proc/vmstat`は、`allocstall_*`、`pgscan_direct`、THPの統計など`vmstat`が公開しないカウンタも含めた全体像を与えてくれる。

### sar（sysstat） {#sar-sysstat}

`sar`は`/proc/vmstat`のデータを経時的に収集して報告する。

```bash
# 毎秒のページフォールトとメジャーフォールト
sar -B 1

# 毎秒のスワップ活動
sar -W 1

# 過去のデータ（収集されたログから）
sar -B -f /var/log/sa/sa28
```

| sar -Bのフィールド | /proc/vmstatのソース |
|--------------|---------------------|
| `pgscank/s` | `pgscan_kswapd`の差分 |
| `pgscand/s` | `pgscan_direct`の差分 |
| `pgsteal/s` | `pgsteal_kswapd` + `pgsteal_direct`の差分 |
| `fault/s` | `pgfault`の差分 |
| `majflt/s` | `pgmajfault`の差分 |

!!! tip "過去の分析にはsar"
    `sysstat`がインストールされ、`sa1`/`sa2`のコレクタが動いていれば、`sar`を使って過去のメモリ逼迫イベントを振り返ることができる。これはインシデント後の分析で非常に役立つ。

### 監視システム（Prometheus、Grafanaなど） {#monitoring-systems-prometheus-grafana-etc}

Node Exporter（Prometheus）は、`/proc/vmstat`のカウンタを`node_vmstat_*`というメトリクスファミリーの下で直接公開する。

```
node_vmstat_pgscan_direct
node_vmstat_pgsteal_direct
node_vmstat_pgmajfault
node_vmstat_pswpin
node_vmstat_pswpout
node_vmstat_allocstall_normal
node_vmstat_thp_fault_alloc
node_vmstat_thp_fault_fallback
```

これらはPrometheusのカウンタとして公開されるので、毎秒のレートを計算するには`rate()`や`irate()`を使う。

```promql
# 直接回収のスキャンレート（page/秒）
rate(node_vmstat_pgscan_direct[5m])

# 回収効率
rate(node_vmstat_pgsteal_kswapd[5m]) / rate(node_vmstat_pgscan_kswapd[5m])

# THPの失敗率
rate(node_vmstat_thp_fault_fallback[5m])
/ (rate(node_vmstat_thp_fault_alloc[5m]) + rate(node_vmstat_thp_fault_fallback[5m]))

# アラート: 持続的な直接回収
rate(node_vmstat_pgscan_direct[5m]) > 1000
```

## 実用的なワンライナー {#practical-one-liners}

### システムは今メモリ逼迫下にあるか? {#is-the-system-under-memory-pressure-right-now}

```bash
# 直接回収や割り当てのストールが起きているか確認する
# 間隔を空けて2回実行し、値が増えているかを見る
grep -E '^(allocstall|pgscan_direct)' /proc/vmstat
```

読み取りの間で`allocstall_*`の値が増えていれば、プロセスはメモリでストールしている。

### システムはスワップスラッシングしているか? {#is-the-system-swap-thrashing}

```bash
# スワップイン/アウトのレートを監視する（page/秒）
watch -n1 "grep -E '^(pswpin|pswpout)' /proc/vmstat"
```

両方がそろって増えていれば、pageが追い出されてはフォールトで戻されることを繰り返している。

### pageの回収はどれくらい効率的か? {#how-efficient-is-page-reclaim}

```bash
# 回収効率のスナップショット
awk '
/^pgsteal_kswapd/  { steal += $2 }
/^pgsteal_direct/  { steal += $2 }
/^pgscan_kswapd/   { scan += $2 }
/^pgscan_direct/   { scan += $2 }
END {
    if (scan > 0) printf "Reclaim efficiency: %.1f%% (%d stolen / %d scanned)\n",
        100*steal/scan, steal, scan
    else print "No reclaim activity recorded"
}' /proc/vmstat
```

### THPは機能しているか、それともフォールバックしているか? {#are-thps-working-or-falling-back}

```bash
awk '
/^thp_fault_alloc /    { alloc = $2 }
/^thp_fault_fallback / { fallback = $2 }
END {
    total = alloc + fallback
    if (total > 0) printf "THP success rate: %.1f%% (%d alloc, %d fallback)\n",
        100*alloc/total, alloc, fallback
    else print "No THP fault activity recorded"
}' /proc/vmstat
```

### 簡易メモリ逼迫ダッシュボード {#quick-memory-pressure-dashboard}

```bash
# 主要指標の1回限りの要約
awk '
/^nr_free_pages /          { printf "Free pages:          %d (%.0f MB)\n", $2, $2*4/1024 }
/^nr_zone_inactive_anon /  { printf "Inactive anon:       %d (%.0f MB)\n", $2, $2*4/1024 }
/^pgscan_direct /          { printf "Direct reclaim scans: %d (since boot)\n", $2 }
/^pgscan_kswapd /          { printf "kswapd scans:        %d (since boot)\n", $2 }
/^allocstall_normal /      { printf "Alloc stalls:        %d (since boot)\n", $2 }
/^pgmajfault /             { printf "Major faults:        %d (since boot)\n", $2 }
/^pswpin /                 { printf "Swap in:             %d pages (since boot)\n", $2 }
/^pswpout /                { printf "Swap out:            %d pages (since boot)\n", $2 }
/^compact_stall /          { printf "Compaction stalls:   %d (since boot)\n", $2 }
' /proc/vmstat
```

## 参考文献 {#further-reading}

- [pageの回収](reclaim.md): カーネルがどのpageを解放するかをどう決めるか
- [スワッピング中に何が起きるか](swapping.md): スワップアウトとスワップインのパス
- [Transparent Huge Pages](thp.md): 2MBのpageの割り当てと管理
- [コンパクション](compaction.md): カーネルがメモリの断片化をどう解消するか
- [メモリ不足](oom.md): 回収が失敗したときに何が起きるか
- [`mm/vmstat.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmstat.c): /proc/vmstatのカーネルソース
