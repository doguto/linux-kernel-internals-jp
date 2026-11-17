# I/Oの可観測性 {#io-observability}

> /proc、iostat、blktrace、ftrace、perfを使ってI/Oの挙動を観測・デバッグする

LinuxカーネルはI/Oの活動を、互いに重なり合ういくつかの層を通じて公開している。`/proc` と `/sys` の疑似ファイルシステムのカウンタ、`blktrace` によるブロック層のトレーシング、ftrace経由のカーネルtracepoint、そして `/proc/PID/io` のプロセスごとの集計である。このリファレンスはそのすべてを扱い、各メトリクスの意味を説明し、実際の問題を診断する際にそれらをどう組み合わせるかを示す。

---

## /proc/diskstats {#procdiskstats}

`/proc/diskstats` は、デバイスごとのI/O統計の正規の情報源である。ディスクのスループットやレイテンシを報告するツール（`iostat`、`sar`、`dstat`、Prometheusの `node_exporter`）はすべて、このファイルを読んで差分を計算している。

**ソース**: [`block/genhd.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/genhd.c)。ディスクごとのカウンタは `struct gendisk` 内の `struct disk_stats` にある。

```bash
$ cat /proc/diskstats
  8   0 sda 487291 21643 42316994 316924 1082744 508437 60720520 5468340 0 1535480 5787056 0 0 0 0 28491 42832
  8   1 sda1 433 0 17624 716 0 0 0 0 0 716 716 0 0 0 0 0 0
259   0 nvme0n1 1284731 84203 98471824 712341 3827164 921842 241837952 14382910 0 3071428 15096084 84291 0 7831204 62831 0 0
```

形式は major minor name [field1 ... fieldN] であり、Nはカーネルバージョンに依存する。4.18より前は11フィールド、4.18からは15フィールド（discard統計が追加）、5.5からは17フィールド（flush統計が追加）である。

### フィールドごとのリファレンス {#field-by-field-reference}

| # | フィールド | 種類 | 説明 |
|---|-------|------|-------------|
| 1 | reads completed | カウンタ | 正常に完了したリードの総数。「完了した」リードとは、要求したプロセスにデータが返されたことを意味する。 |
| 2 | reads merged | カウンタ | ディスパッチ前にI/Oスケジューラでマージされた隣接するリードリクエストの数。マージされたリードはキュー深度を減らし、スループットを向上させる。 |
| 3 | sectors read | カウンタ | 読み込んだ512バイトセクタの総数。2で割るとKiBに換算できる。 |
| 4 | time reading (ms) | カウンタ | リードI/Oに費やしたミリ秒数。これは完了したすべてのリードのレイテンシの合計であり、reads completedで割ると平均リードレイテンシが得られる。 |
| 5 | writes completed | カウンタ | 正常に完了したライトの総数。 |
| 6 | writes merged | カウンタ | I/Oスケジューラによってマージされた隣接するライトリクエストの数。 |
| 7 | sectors written | カウンタ | 書き込んだ512バイトセクタの総数。 |
| 8 | time writing (ms) | カウンタ | ライトI/Oに費やしたミリ秒数。 |
| 9 | I/Os in progress | ゲージ | デバイスに発行されたがまだ完了していないI/Oの**現在の**数。diskstatsで唯一のゲージであり、他はすべてカウンタである。解釈については後述。 |
| 10 | time doing I/Os (ms) | カウンタ | 少なくとも1つのI/Oが進行中だった（つまりデバイスが「ビジー」だった）ミリ秒数。`iostat` の `%util` はここから導出される。 |
| 11 | weighted time doing I/Os (ms) | カウンタ | `(I/Os in progress) × (経過ms)` の時間にわたる合計。`iostat` が `avgqu-sz` と `await` を計算するのに使う。 |
| 12 | discards completed | カウンタ | 完了したdiscard（TRIM）リクエストの総数。discardに対応していないデバイスでは0。カーネル4.18で追加。 |
| 13 | discards merged | カウンタ | マージされたdiscardリクエストの数。 |
| 14 | sectors discarded | カウンタ | discardされた512バイトセクタの総数。 |
| 15 | time discarding (ms) | カウンタ | discard I/Oに費やしたミリ秒数。 |
| 16 | flush requests completed | カウンタ | 完了したflush（キャッシュフラッシュ/FUAバリア）リクエストの総数。カーネル5.5で追加。 |
| 17 | time flushing (ms) | カウンタ | flushリクエストに費やしたミリ秒数。 |

!!! note "カーネルバージョンごとの利用可否"
    フィールド数はカーネルバージョンに依存する。Linux 5.5以降は17フィールド、4.18–5.4は15フィールド（フィールド16–17のflush統計はまだない）、4.18より前は11フィールド（フィールド12–17のdiscardとflushの統計はまだない）である。diskstatsを解析するツールは、可変の列数に対応しなければならない。

### 「I/Os in progress」（フィールド9）の解釈 {#interpreting-ios-in-progress-field-9}

フィールド9は、ファイルを読んだ瞬間のデバイスのキュー深度である。ドライバに発行されたが、完了割り込みがまだ処理されていないI/Oを反映する。

- **回転ディスク**: 1–2の値が持続するのは正常である。デバイスの定格キュー深度（最近のHDDでは通常32–64）を一貫して上回る値は、デバイスが処理できる以上のI/Oをブロック層がソフトウェアでキューに溜めていることを意味する。
- **NVMe**: NVMeデバイスは複数のハードウェアキュー（多くの場合CPUごとに1つ）を公開する。`iostat` はすべてのキューを合算した値を報告するため、16キューのNVMeデバイスで `avgqu-sz` が16でも飽和ではない。単にキューごとに1つのI/Oが保留中であるだけかもしれない。
- スループットが高いのに**値が0**の場合、I/Oがすぐに完了しており、たまたま静かな瞬間にサンプリングしただけである。時間平均で見るには `iostat` の `avgqu-sz` を使う。

### iostatがdiskstatsの上にどう構築されているか {#how-iostat-is-built-on-diskstats}

`iostat` は `/proc/diskstats` を2回（時刻T1とT2に）読み、次を計算する。

```
interval = T2 - T1 (in milliseconds)
r/s      = Δreads_completed / interval_seconds
rkB/s    = Δsectors_read × 512 / 1024 / interval_seconds
await    = Δtime_reading / Δreads_completed  (average latency per read)
%util    = Δtime_doing_IOs / interval × 100
avgqu-sz = Δweighted_time_IOs / interval
```

ソースは `sysstat` パッケージの [`iostat.c`](https://github.com/sysstat/sysstat/blob/master/iostat.c) にある。

---

## iostat -xの出力 {#iostat-x-output}

`iostat -x`（拡張統計）は、I/O診断で最初に使われる最も一般的なツールである。起動以降の平均ではなく差分を得るために、間隔を指定して実行する。

```bash
iostat -x 1          # 1秒間隔、全デバイス
iostat -x -d sda 5   # 5秒間隔、sdaのみ
iostat -xz 1         # 活動のないデバイスを表示しない
```

### 現実的な出力例 {#realistic-example-output}

```
Device            r/s     rkB/s   rrqm/s  %rrqm r_await rareq-sz     w/s     wkB/s   wrqm/s  %wrqm w_await wareq-sz     d/s     dkB/s   drqm/s  %drqm d_await dareq-sz     f/s f_await  aqu-sz  %util
nvme0n1        182.00  46081.00     0.00   0.00    0.22   253.2    94.00  24318.00     8.00   7.84    1.14   258.7    0.00      0.00     0.00   0.00    0.00      0.0    4.00    0.18    0.31   5.20
sda              2.00    128.00    14.00  87.50    4.31    64.0   118.00   7432.00   243.00  67.32   42.18    63.0    0.00      0.00     0.00   0.00    0.00      0.0    0.00    0.00    4.98  87.40
```

### フィールドの説明 {#field-explanations}

| フィールド | 意味 |
|-------|---------|
| `r/s`, `w/s` | 1秒あたりに完了したリードとライトの数（マージ後）。これがデバイスのI/Oレートである。 |
| `rkB/s`, `wkB/s` | 1秒あたりに読み書きしたキロバイト数。1024で割るとMB/sになる。デバイスの定格シーケンシャルスループットと比較する。 |
| `rrqm/s`, `wrqm/s` | ディスパッチ前にI/Oスケジューラで1秒あたりにマージされたリードとライトのリクエスト数。マージ率が高いことは、スケジューラがリクエストをまとめられるシーケンシャルまたは空間的に局所的なワークロードであることを示す。 |
| `%rrqm`, `%wrqm` | マージされたリード／ライトリクエストの割合。例では、`sda` はリードの87.5%、ライトの67.3%をマージしており、非常にシーケンシャルなワークロードである。`nvme0n1` はほとんどマージしておらず、ランダムI/Oのワークロードに典型的である。 |
| `r_await`, `w_await` | リードとライトのリクエストの発行から完了までの平均レイテンシ（ms）。カーネルのキューで費やした時間とデバイスのサービス時間を含む。ユーザーがI/Oレイテンシとして体感するのはこれである。 |
| `aqu-sz`（古いsysstatでは `avgqu-sz`） | 平均キューサイズ。処理中のリクエスト数の時間平均。フィールド11（重み付き時間）から導出される。回転ディスクでは `aqu-sz > 1` が持続するとキューイングが起きていることを示す。NVMeでは全キューにわたってより高い値になるのが普通である。 |
| `rareq-sz`, `wareq-sz` | リードとライトの平均リクエストサイズ（KiB）。値が小さく（< 16 KiB）I/Oレートが高い場合はランダムI/Oを示す。値が大きい（> 128 KiB）場合はシーケンシャルまたは大ブロックのI/Oを示す。 |
| `%util` | インターバルのうち、デバイスに少なくとも1つのI/Oが処理中だった時間の割合（diskstatsのフィールド10から）。**これはCPU使用率ではない**。デバイスのビジー時間である。 |
| `svctm` | **非推奨であり、最近のsysstatでは削除された。** 推定サービス時間 = `%util / (r/s + w/s)` だった。この推定はSSDや内部並列性を持つデバイスでは信頼できないため、使ってはならない。 |

### 飽和の特定 {#identifying-saturation}

**回転ディスク**で `%util` が100%に近づくことは、デバイスが飽和していることを意味する。実質的に常にビジーであり、レイテンシを増やさずにこれ以上のI/Oを吸収できない。あわせて現れるシグナル:

- `await` がデバイスの典型的なサービス時間（HDDで1–5 ms、NVMeで0.05–0.3 ms）を大きく上回って上昇する
- `aqu-sz` が増加する。リクエストが完了するよりも速くキューに溜まっていく

**NVMeとSSD**では、デバイスが内部でI/Oを並列処理するため、`%util` が100%でも必ずしも飽和を意味しない。NVMeドライブは多数のハードウェアキューを持つため、`await` 0.1 msで `%util` 100%を持続できる。代わりに `await`（または `r_await`/`w_await`）のレイテンシ増加を監視する。

**飽和のチェックリスト**:

1. `%util` → 100% **かつ** `r_await` または `w_await` が上昇: デバイスがボトルネックである
2. HDDで `aqu-sz` > 1: デバイスの速度に対してキュー深度が高すぎる
3. 通常はシーケンシャルなワークロードで `rrqm/s` が0に落ちる: ワークロードがランダム化し、レイテンシが増えている
4. `wkB/s` がデバイスの書き込み帯域で頭打ちになる: 書き込みがスロットリングされている

---

## /proc/sys/vm/ のライトバック用パラメータ {#procsysvm-writeback-parameters}

VMのdirtyライトバックの仕組みは、変更されたページをページキャッシュからストレージへいつフラッシュするかを制御する。これらのパラメータのチューニングは、書き込みレイテンシの一貫性とスループットに影響する。すべての値は `/proc/sys/vm/` にあり、`sysctl` で設定できる。

**ソース**: [`mm/page-writeback.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page-writeback.c)。

### dirty_ratioとdirty_bytes {#dirty_ratio-and-dirty_bytes}

```bash
$ cat /proc/sys/vm/dirty_ratio
20
$ cat /proc/sys/vm/dirty_bytes
0
```

`dirty_ratio` は、システムの総メモリのうちdirty（変更済みでまだ書き込まれていない）ページで埋められる最大の割合を設定する。このしきい値を超えると、**`write()` を呼ぶプロセスがスロットリングされる**。ライトバックが進むまでカーネル内でブロックする。これがハードリミットであり、到達するとアプリケーションに直接レイテンシが生じる。

`dirty_bytes` は同じ上限を絶対的なバイト数で表したものである。`dirty_bytes` が0でなければ、`dirty_ratio` より優先される。同時に有効になるのは2つのうち一方だけである。

**調整すべきとき**: 何GiBものRAMを持つシステムでは、`dirty_ratio=20%` によって数十GiBのdirtyデータが溜まりうる。それだけの量のデータが一気にライトバックされると、レイテンシのスパイクが生じる。レイテンシに敏感なワークロードでは、割合を下げる（5–10%）か、絶対値の `dirty_bytes` の上限（例: 1 GiB）を設定する。

```bash
# dirtyデータを絶対値で1 GiBに制限する
sysctl vm.dirty_bytes=1073741824
```

### dirty_background_ratioとdirty_background_bytes {#dirty_background_ratio-and-dirty_background_bytes}

```bash
$ cat /proc/sys/vm/dirty_background_ratio
10
```

`dirty_background_ratio` は、カーネルが `flusher` スレッド（以前の `pdflush`）による**バックグラウンドのライトバック**を開始するしきい値を設定する。これはソフトリミットであり、ここでライトバックは始まるが、プロセスはまだブロックされない。

`dirty_background_bytes` はその絶対値版である。優先順位の規則は上と同じである。

2つのしきい値の関係:

```
dirty_background_ratio < dirty_ratio
(background writeback starts)    (process throttling starts)
```

2つの値の間隔が広いと、アプリケーションがスロットリングされずにdirtyデータが溜まる大きな領域ができる。間隔が狭いと、ライトバックがより早く始まり、システムはdirtyページが0に近い状態に保たれる。

### dirty_writeback_centisecs {#dirty_writeback_centisecs}

```bash
$ cat /proc/sys/vm/dirty_writeback_centisecs
500
```

`wb_workfn` のflusherスレッドが起床してdirtyページを書き込む間隔（センチ秒、すなわち100分の1秒単位）。デフォルトの500 = 5秒。

これを0にすると定期的なライトバックが完全に無効になる（ほとんどのワークロードでは推奨されない）。減らす（例: 100 = 1秒）と、より小さなライトバックのバーストがより頻繁に起き、バックグラウンドI/Oがわずかに増える代わりにレイテンシの一貫性が向上しうる。

### dirty_expire_centisecs {#dirty_expire_centisecs}

```bash
$ cat /proc/sys/vm/dirty_expire_centisecs
3000
```

dirtyページがページキャッシュに留まれる時間（センチ秒）で、これを過ぎるとライトバックが「期限切れ」とみなされ、そのページは書き込まれなければならない。デフォルトの3000 = 30秒。

この値より長くdirtyのままのページは、次の `dirty_writeback_centisecs` の起床時にライトバックの対象になる。この値を減らすと、カーネルはページが古くなるにつれてより積極的にフラッシュし、dirtyの窓を小さく保つ。

### ライトバックが追いついているかの確認 {#checking-if-writeback-is-keeping-up}

```bash
# 現在のdirtyページとライトバック中のページの数（ページ数 × 4 KiB = バイト数）
grep -E "^Dirty:|^Writeback:" /proc/meminfo

# ライトバックのパイプラインをリアルタイムで監視する
watch -n 1 'grep -E "^Dirty:|^Writeback:|^NFS_Unstable:" /proc/meminfo'

# /proc/vmstat経由で、ページがdirtyになるレートと書き込まれるレートを見る
grep -E "nr_dirty|nr_writeback|nr_dirtied|nr_written" /proc/vmstat
```

**ライトバックが追いついていないことを示すシグナル**:

- `/proc/meminfo` の `Dirty:` が、一貫して `dirty_bytes` /（`dirty_ratio` × 総メモリ）付近かそれ以上にある
- `/proc/vmstat` の `nr_dirtied` のレートが `nr_written` のレートを大幅に上回る
- アプリケーションの `write()` 呼び出しのレイテンシが高くなっている（strace、perf、eBPFで確認）。これはdirtyの上限でのプロセスのスロットリングの特徴である

```bash
# ライトバック活動の差分（1秒ごとのサンプル）
while true; do
    paste <(grep -E "nr_dirtied|nr_written" /proc/vmstat) \
          <(sleep 1; grep -E "nr_dirtied|nr_written" /proc/vmstat) | \
    awk '$1==$3 {print $1, $4-$2, "pages/s"}'
done
```

---

## /sys/class/bdi/ {#sysclassbdi}

BDI（Backing Device Info）サブシステムは、デバイスごとのライトバックの状態を追跡する。各ブロックデバイス（およびネットワークファイルシステムのマウント）は `/sys/class/bdi/` の下にエントリを持つ。

**ソース**: [`include/linux/backing-dev.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/backing-dev.h) と [`mm/backing-dev.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/backing-dev.c)。

```bash
$ ls /sys/class/bdi/
259:0   8:0   default   loop0

$ ls /sys/class/bdi/259:0/   # nvme0n1
avg_read_bandwidth  dirty    max_bytes       max_ratio     read_ahead_kb
avg_write_bandwidth  error   max_prop_frac   min_bytes     stable_pages_required
```

### 主なファイル {#key-files}

**`read_ahead_kb`** — デバイスごとのreadaheadウィンドウ（KiB）。カーネルのreadaheadアルゴリズムは、シーケンシャルアクセスを検出すると、現在のI/O位置からこのバイト数だけ先まで読む。デフォルトは通常128 KiBだが、カーネルはアクセスパターンに基づいてこれを動的に調整する。大きなシーケンシャルリード（データベースのバックアップ、ストリーミング）では、この値を増やすと、必要ないかもしれないデータを読むことと引き換えにI/Oリクエストの数が減る。

```bash
# データベースのバックアップ用デバイスでreadaheadを増やす
echo 4096 > /sys/class/bdi/8:0/read_ahead_kb
# あるいはblockdev経由で:
blockdev --setra 8192 /dev/sda   # 8192 × 512バイト = 4 MiB
```

**`max_ratio`** — 総dirtyメモリのうち、このデバイスのdirtyページが占められる最大の割合。デフォルトは100（デバイスごとの制限なし）。同じシステムに高速なNVMeと低速なHDDがあり、HDDのライトバックにdirtyキャッシュを独占させたくない場合に有用である。

```bash
# sdaをdirtyプールの30%に制限する
echo 30 > /sys/class/bdi/8:0/max_ratio
```

**`stable_pages_required`** — 1の場合、そのブロックデバイスはstable pageを必要とする。つまり、ページがライトバック中の間、カーネルはそのページを変更してはならない。一部のドライバ（特に特定のネットワークファイルシステムやRDMA NIC）がこれを設定する。これはdirtyページに対するコピーオンライトの最適化を無効にし、CPUのオーバーヘッドを加える。

!!! warning "Linux 5.18で削除"
    `stable_pages_required` は、stable pageの要件がブロック層から取り除かれたLinux 5.18で削除された。このファイルは5.18以降のカーネルには存在しない。

```bash
# Linux 5.17以前のみ:
cat /sys/class/bdi/*/stable_pages_required
```

### bdi_statカウンタ {#bdi_stat-counters}

BDIごとのカウンタは、`nr_` で始まる `/proc/vmstat` のフィールド（例: `nr_dirty`、`nr_writeback`）を通じて間接的に公開される。BDIレベルのカウンタは次を追跡する。

- **BDI_WRITEBACK**: 現在このデバイスへライトバック中のページ
- **BDI_RECLAIMABLE**: このデバイスに関連付けられたdirtyページのうち、回収の候補となるもの
- **BDI_WRITTEN**: 書き込まれたページの総数（単調増加カウンタ）

これらは、`writeback` tracepointグループに対して `bpftrace` を使うことで観測できる。このグループのほとんどのイベントには `bdi_name` が含まれる。

---

## blktraceとblkparse {#blktrace-and-blkparse}

`blktrace` はデバイスのブロック層におけるすべてのイベントを捕捉し、バイナリのトレースデータとして書き出す。`blkparse` はそれをデコードする。これらを組み合わせると、個々のI/Oリクエストのライフサイクルを、キューに入れられた瞬間から完了するまでトレースできる。

**インストール**: 通常は `blktrace` パッケージに含まれる。

**ソース**: [`kernel/trace/blktrace.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/trace/blktrace.c)（カーネル内のトレーシング基盤）、[`include/uapi/linux/blktrace_api.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/blktrace_api.h)（`BLK_TA_*` アクションコード）、[`blktrace` documentation](https://git.kernel.org/pub/scm/linux/kernel/git/axboe/blktrace.git/tree/doc)（使い方ガイド、manページ）。

### トレースの記録 {#recording-a-trace}

```bash
# /dev/sda 上のすべてのブロックイベントを10秒間記録する
blktrace -d /dev/sda -o /tmp/sda_trace -- sleep 10

# これによりCPUごとのファイルが作られる: /tmp/sda_trace.blktrace.0, .1, .2 ...

# すぐに解析する（CPUごとのファイルを結合する）
blkparse -i /tmp/sda_trace -o /tmp/sda_trace.txt

# あるいはファイルに書かずに直接パイプする
blktrace -d /dev/sda -o - | blkparse -i -
```

### ブロック層のアクション（アクションコード） {#block-layer-actions-the-action-codes}

各トレースイベントには、ブロックスタックのどこでイベントが発生したかを示すアクションコードがある。主なコード:

| コード | 名前 | 説明 |
|------|------|-------------|
| `Q` | Queue | ファイルシステムまたはアプリケーションによってI/Oリクエストがブロック層に発行された。 |
| `G` | Get request | リクエストプールから `struct request` が割り当てられた。 |
| `M` | Back merge | リクエストがI/Oスケジューラのキュー内の既存リクエストに後方マージされた（キュー内の既存リクエストの末尾にマージされた）。 |
| `I` | Insert | リクエストがI/Oスケジューラのキューに挿入された。 |
| `S` | Sleep | 利用可能なリクエスト構造体がなく、発行元がそれを待ってスリープしなければならなかった。 |
| `D` | Dispatch | リクエストがデバイスドライバへディスパッチ（送出）された。 |
| `C` | Complete | デバイスドライバが完了を報告した。I/Oは終わった。 |
| `P` | Plug | リクエストキューがplugされた（まとめるための窓が始まった）。 |
| `U` | Unplug | リクエストキューがunplugされた（まとめるための窓が終わり、I/Oがドライバへフラッシュされた）。 |
| `F` | Front merge | リクエストが既存リクエストに前方マージされた（キュー内の既存リクエストの先頭にマージされた）。 |
| `X` | Split | 大きなbioがより小さなリクエストに分割された。 |
| `A` | Remap | リクエストが再マップされた（例: device mapperのデバイスから下位のデバイスへ）。 |
| `R` | Requeue | リクエストが再キューされた（再試行のためにドライバからブロック層へ戻された）。 |

### blkparseの出力例 {#sample-blkparse-output}

```
  8,0    3        1     0.000000000   693  Q   W 4456448 + 256 [kworker/u8:2]
  8,0    3        2     0.000004123   693  G   W 4456448 + 256 [kworker/u8:2]
  8,0    3        3     0.000005891   693  I   W 4456448 + 256 [kworker/u8:2]
  8,0    3        4     0.000008341     0  D   W 4456448 + 256 [swapper/3]
  8,0    3        5     0.004182934     0  C   W 4456448 + 256 [swapper/3]
```

列: `device  cpu  sequence  timestamp(s)  pid  action  rwbs  sector + nr_sectors  [comm]`

`rwbs` フィールドはI/Oの種類を表す: `R`=リード、`W`=ライト、`D`=discard、`F`=flush、`S`=sync、`N`=なし、`A`=readahead。

### blkparseによるI/Oレイテンシの測定 {#measuring-io-latency-with-blkparse}

1つのI/Oリクエストのレイテンシは、`D`（ドライバへのディスパッチ）から `C`（完了）までの時間である。`Q`（キュー）から `C` までの時間には、スケジューラでのキューイング時間が含まれる。

```bash
# blkparse組み込みのレイテンシサマリーを使う
blkparse -i /tmp/sda_trace -d /tmp/sda_trace.bin
btt -i /tmp/sda_trace.bin

# あるいはDからCまでのレイテンシを手動で抽出する
blkparse -i /tmp/sda_trace -f "%T %a %S\n" | \
  awk '
    /D/ { dispatch[$3] = $1 }
    /C/ { if ($3 in dispatch) { lat = $1 - dispatch[$3]; print lat * 1000 " ms"; delete dispatch[$3] } }
  ' | sort -n | awk 'BEGIN{n=0; sum=0} {n++; sum+=$1; a[n]=$1} END{
    print "count=" n " avg=" sum/n " ms"
    print "p50=" a[int(n*0.50)] " ms"
    print "p99=" a[int(n*0.99)] " ms"
  }'
```

### btt — block trace toolkit {#btt-block-trace-toolkit}

`btt` は `blktrace` のバイナリデータを解析し、構造化されたレイテンシの内訳を出力する付属ツールである。

```bash
blktrace -d /dev/sda -o /tmp/trace -- sleep 30
blkparse -i /tmp/trace -d /tmp/trace.bin -O
btt -i /tmp/trace.bin
```

`btt` はI/Oレイテンシを次の要素に分解する。

- **Q2G**: QueueからGet requestまで（リクエスト構造体の割り当て）
- **G2I**: GetからInsertまで（I/Oスケジューラの処理）
- **I2D**: InsertからDispatchまで（スケジューラのキューで費やした時間。これが「キューイング」の部分）
- **D2C**: DispatchからCompleteまで（デバイスのサービス時間）
- **Q2C**: 全体のレイテンシ（QueueからCompleteまで）

健全なシステムでは、`I2D`（スケジューラでのキューイング）は小さく、`D2C` はデバイスの公称レイテンシと一致するはずである。`I2D` が大きい場合は、I/Oスケジューラがリクエストをバッチ化するためにディスパッチを遅らせている。これは意図的なこと（HDDの場合）もあれば、問題となること（レイテンシに敏感なNVMeワークロードの場合）もある。

### blkiomon {#blkiomon}

`blkiomon` は、ディスクに書き込むことなく、blktraceのストリームからオンラインでI/Oを監視する。

```bash
blktrace -d /dev/nvme0n1 -o - | blkiomon -I 5 -
# 5秒ごとにD2Cレイテンシの統計を報告する
```

---

## ftraceのI/O tracepoint {#ftrace-io-tracepoints}

`block` と `writeback` のtracepointグループは、`blktrace` と同じイベントを標準のftrace基盤を通じて公開する。これにより、他のカーネルtracepointと組み合わせたり、PIDやCPUでフィルタしたり、`perf` や `bpftrace` で利用したりできる。

### 利用可能なtracepointを探す {#finding-available-tracepoints}

```bash
ls /sys/kernel/debug/tracing/events/block/
ls /sys/kernel/debug/tracing/events/writeback/
ls /sys/kernel/debug/tracing/events/filemap/

# あるいはperf経由で
perf list 'block:*' 'writeback:*' 'filemap:*'
```

### ftraceで有効にする {#enabling-with-ftrace}

```bash
# writebackグループ全体を有効にする
echo 1 > /sys/kernel/debug/tracing/events/writeback/enable

# イベントをストリーミングする
cat /sys/kernel/debug/tracing/trace_pipe

# 後片付け
echo 0 > /sys/kernel/debug/tracing/events/writeback/enable
echo > /sys/kernel/debug/tracing/trace
```

### block:block_rq_issueとblock:block_rq_complete {#blockblock_rq_issue-and-blockblock_rq_complete}

`block_rq_issue` は、リクエストがデバイスドライバへディスパッチされたとき（blktraceの `D` イベント）に発火する。`block_rq_complete` は完了時（`C` イベント）に発火する。

この2つのtracepointは、`blktrace` を使わずにD2Cレイテンシを測定するためのftraceにおける等価物である。

**フィールド（block_rq_complete）**:

| フィールド | 説明 |
|-------|-------------|
| `dev` | デバイスのmajor:minor |
| `sector` | 開始セクタ |
| `nr_sector` | セクタ数 |
| `errors` | エラーコード（0 = 成功） |
| `rwbs` | I/O種別のフラグ（R/W/D/F） |
| `cmd` | SCSIコマンド文字列（SCSIデバイスの場合） |

```bash
# bpftraceでブロックI/Oのレイテンシを測定する
bpftrace -e '
tracepoint:block:block_rq_issue {
    @start[args->dev, args->sector] = nsecs;
}
tracepoint:block:block_rq_complete
/@start[args->dev, args->sector]/ {
    $lat = nsecs - @start[args->dev, args->sector];
    delete(@start[args->dev, args->sector]);
    @latency_us = hist($lat / 1000);
}
interval:s:10 { print(@latency_us); clear(@latency_us); }'
```

### writeback:writeback_single_inode {#writebackwriteback_single_inode}

ライトバックのコードが1つのinodeのdirtyページを処理するときに発火する。これはライトバック活動をinodeごとに見たものである。

**フィールド**:

| フィールド | 説明 |
|-------|-------------|
| `name` | バッキングデバイス名（例: `sda`、`nvme0n1`） |
| `ino` | inode番号 |
| `state` | inodeのライトバック状態フラグ |
| `dirtied_when` | inodeが最初にdirtyになったときのjiffies |
| `writeback_index` | ライトバックが開始されたページオフセット |
| `nr_to_write` | 書き込みを要求されたページ数 |
| `wrote` | 実際に書き込まれたページ数 |

```bash
# どのinodeが最も多くのライトバックを生んでいるか？
bpftrace -e '
tracepoint:writeback:writeback_single_inode {
    @pages[args->ino, str(args->name)] = sum(args->wrote);
}
interval:s:10 {
    print(@pages, 10);
    clear(@pages);
}'
```

特定のinode番号が繰り返し現れる場合は、それをパスに変換する。

```bash
find / -inum <inode_number> 2>/dev/null
```

### writeback:global_dirty_state {#writebackglobal_dirty_state}

ライトバックのスロットリングのコードにある `balance_dirty_pages()` から定期的に発火する。現在の上限の計算結果を含め、グローバルなdirtyページの状態のスナップショットを提供する。

**フィールド**:

| フィールド | 説明 |
|-------|-------------|
| `nr_dirty` | 現在のdirtyページ数 |
| `nr_writeback` | 現在ライトバック中のページ数 |
| `background_thresh` | バックグラウンドのライトバックのしきい値（ページ数） |
| `dirty_thresh` | dirtyのハードリミット（ページ数） |
| `dirty_limit` | 平滑化後の実効的なdirtyの上限 |
| `nr_dirtied` | 単調増加するdirtyページのカウンタ |
| `nr_written` | 単調増加する書き込み済みページのカウンタ |

```bash
echo 1 > /sys/kernel/debug/tracing/events/writeback/global_dirty_state/enable
cat /sys/kernel/debug/tracing/trace_pipe
# 出力: writeback-global_dirty_state: dirty=12345 writeback=234 bg_thresh=51200 thresh=102400
```

このtracepointは、`dirty_ratio`/`dirty_background_ratio` のチューニングが意図した効果を上げているかを理解するうえで非常に有用である。`dirty_thresh` に対する `nr_dirty` を監視する。両者が一貫して近い場合、プロセスはスロットリングされている。

### writeback:writeback_writepage {#writebackwriteback_writepage}

`write_cache_pages()` で書き込まれる各ページについて発火する。頻度が非常に高いため、フィルタを使って対象のinodeに絞り、短時間だけ有効にする。

```bash
# 特定のデバイスに絞り込む
echo 'name == "sda"' > /sys/kernel/debug/tracing/events/writeback/writeback_writepage/filter
echo 1 > /sys/kernel/debug/tracing/events/writeback/writeback_writepage/enable
```

### filemap:mm_filemap_add_to_page_cache {#filemapmm_filemap_add_to_page_cache}

ページがページキャッシュに追加されたときに発火する。ストレージからページを読み込むリードフォールトの際や、ファイルが初めてmmapまたはreadされたときである。

**フィールド**: `i_ino`、`s_dev`、`index`（ファイル内のページインデックス）、`pfn`

```bash
# デバイスごとのキャッシュ投入レート
bpftrace -e '
tracepoint:filemap:mm_filemap_add_to_page_cache {
    @add[args->s_dev] = count();
}
interval:s:1 { print(@add); clear(@add); }'
```

### filemap:mm_filemap_delete_from_page_cache {#filemapmm_filemap_delete_from_page_cache}

ページがページキャッシュから追い出されたときに発火する。メモリ回収、明示的な `fadvise(POSIX_FADV_DONTNEED)`、ファイルの切り詰めのいずれかによる。

**フィールド**: addと同じ。`i_ino`、`s_dev`、`index`、`pfn`

```bash
# キャッシュの入れ替わり: 1秒あたりの追加と追い出し
bpftrace -e '
tracepoint:filemap:mm_filemap_add_to_page_cache    { @add = count(); }
tracepoint:filemap:mm_filemap_delete_from_page_cache { @del = count(); }
interval:s:5 {
    printf("cache: +%d -%d pages/5s (net %d)\n", @add, @del, @add - @del);
    clear(@add); clear(@del);
}'
```

追い出しのレート（`mm_filemap_delete_from_page_cache`）が高く、同時にメジャーフォールトのレート（`/proc/vmstat` の `pgmajfault`）も高い場合、ページキャッシュがワーキングセットに対して小さすぎることを示す。ページが追い出された直後に再び必要になり、ディスクからの読み込みが繰り返されている。

---

## I/Oのためのperf {#perf-for-io}

`perf` は、低いオーバーヘッドでブロック層のtracepointを記録、カウント、サンプリングできる。特に、I/Oを特定のプロセスやコールスタックに帰属させるのに有用である。

### ブロックイベントのカウント {#counting-block-events}

```bash
# システム全体のすべてのブロックイベントを10秒間カウントする
perf stat -e 'block:block_rq_issue,block:block_rq_complete,block:block_bio_queue' \
           -a -- sleep 10

# 出力例:
#  Performance counter stats for 'system wide':
#     182,413      block:block_rq_issue
#     182,389      block:block_rq_complete
#     201,847      block:block_bio_queue
```

`block_bio_queue` と `block_rq_issue` の差は、マージされたbioの数である。これらはブロック層に入ったが、既存のリクエストに吸収された。

### コールスタック付きの記録 {#recording-with-call-stacks}

```bash
# ブロックの完了をカーネル+ユーザーのスタック付きで30秒間記録する
perf record -e block:block_rq_complete -ag -- sleep 30
perf report
```

`-a` フラグはシステム全体（全CPU）を記録する。`-g` はコールスタックを捕捉する。これにより、どのコードパスが最も多くのI/O完了を生み出しているかがわかる。

```bash
# 後処理用のperf script
perf script | head -50
# 出力形式:
# kworker/u8:2   693 [003] 12345.678901: block:block_rq_complete: 8,0 W () 4456448 + 256 [0]
```

### perf topでライトバックのホットスポットを見つける {#finding-writeback-hotspots-with-perf-top}

```bash
# ライトバックで時間を費やしている関数のライブビュー
perf top -e block:block_rq_issue --sort comm,dso,symbol

# ライトバック固有: flusherスレッドのCPU使用率を監視する
perf top -p $(pgrep -d, kworker) -e cycles
```

### ライトバックのtracepointのカウント {#counting-writeback-tracepoints}

```bash
# 60秒間のライトバック活動のサマリー
perf stat -e 'writeback:writeback_single_inode,writeback:global_dirty_state,writeback:writeback_written' \
           -a -- sleep 60
```

### I/Oの帰属のためのperfとbpftrace {#perf-with-bpftrace-for-io-attribution}

```bash
# 書き込みI/Oのバイト数の上位プロセス（ブロックのtracepointを使用）
bpftrace -e '
tracepoint:block:block_rq_issue /args->rwbs[0] == "W"/ {
    @bytes[comm] = sum(args->nr_sector * 512);
}
interval:s:10 { print(@bytes, 10); clear(@bytes); }'
```

---

## /proc/PID/io {#procpidio}

すべてのプロセスは、プロセスの開始以降のI/O活動を集計する `/proc/PID/io` ファイルを持つ。これは `/proc/diskstats` とは異なり、I/Oをデバイスではなく個々のプロセスに帰属させる。

**ソース**: [`fs/proc/base.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/proc/base.c)。集計は `include/linux/task_io_accounting.h` の `task_io_accounting` で行われる。

```bash
$ cat /proc/$(pgrep postgres | head -1)/io
rchar: 18492831744
wchar: 4831291392
syscr: 2847291
syscw: 1038472
read_bytes: 12849381376
write_bytes: 3748103168
cancelled_write_bytes: 104857600
```

### フィールドごとのリファレンス {#field-by-field-reference_1}

| フィールド | 説明 |
|-------|-------------|
| `rchar` | `read()`、`pread()`、`readv()` などのシステムコールに渡されたバイト数。ページキャッシュとディスクのどちらから来たかにかかわらず、読み込んだ**すべての**バイトを数える。完全にキャッシュされたファイルを読んでも `rchar` は増加する。 |
| `wchar` | `write()`、`pwrite()`、`writev()` などに渡されたバイト数。これも、ストレージに到達したバイトではなく、システムコールに渡されたバイトを数える。 |
| `syscr` | 読み込みシステムコール（read、pread64、readvなど）の回数。 |
| `syscw` | 書き込みシステムコールの回数。 |
| `read_bytes` | 実際にブロック層の読み込みを引き起こしたバイト数。ストレージから読み込まれたバイト（ページキャッシュミス）である。このカウンタは、カーネルがディスクからページを取得しなければならなかったときにだけ増える。 |
| `write_bytes` | ライトバックによってストレージに書き込まれたバイト数。このプロセスがdirtyにしたページのうち、ディスクへフラッシュされたもの。 |
| `cancelled_write_bytes` | 「キャンセル」されたライトバックのバイト数。書き込まれずに解放されたdirtyページであり、典型的には、dirtyページがフラッシュされる前にプロセスがファイルを切り詰めたり削除したりした場合である。例では、100 MiBの書き込みが回避された。 |

### 重要な区別 {#key-distinctions}

**`wchar` と `write_bytes`**: `wchar` は `write()` に渡されたバイト数を数える。`write_bytes` は実際にストレージに到達したバイト数を数える。両者の差は、ページキャッシュに吸収された書き込みI/Oの量である（いずれライトバックでフラッシュされる書き込み、またはキャンセルされた書き込み）。`wchar = 10 GiB` で `write_bytes = 500 MiB` のプロセスは強くバッファリングされており、書き込みの95%がキャッシュに留まった。

**`read_bytes` は累積値であり、読み込みごとの値ではない**: キャッシュされた同じファイルを1000回読むプロセスでは、`rchar` は毎回ファイルサイズずつ増えるが、`read_bytes` は1回（ディスクからページを読み込んだ最初の読み込み）しか増えない。以降の読み込みはページキャッシュにヒットし、`read_bytes` を増やさない。

**`cancelled_write_bytes` はストレージへの負荷を下げうる**: 一時ファイルに書き込み、フラッシュ前にそれをunlinkするデータベースは、`wchar` が大きくても `cancelled_write_bytes` も大きいことがあり、その場合、実際のストレージのトラフィックはずっと少なかったことを意味する。

### /proc/PID/ioを使った診断 {#using-procpidio-for-diagnosis}

```bash
# プロセスのI/Oのスナップショット
pid=$(pgrep myapp)
cat /proc/$pid/io

# 5秒間の差分
a=$(cat /proc/$pid/io)
sleep 5
b=$(cat /proc/$pid/io)

paste <(echo "$a") <(echo "$b") | \
  awk '{split($1,k,":");split($2,v,":");printf "%s: %d/s\n", k[1], (v[2]-$2)/5}'
```

### iotop {#iotop}

`iotop` はすべてのプロセスの `/proc/PID/io` を読み、ソートされたライブビューを表示する。

```bash
iotop -o        # I/Oを行っているプロセスだけを表示する
iotop -a        # iotop開始以降の累積I/O
iotop -p <pid>  # 特定のプロセスを監視する
```

`iotop` の `DISK READ` と `DISK WRITE` の列は、`read_bytes` と `write_bytes` の差分、つまり実際のストレージのトラフィックから来ている。`SWAPIN` と `IO%` の列は、プロセスがI/Oでブロックされていた時間を示す。

---

## Pressure Stall Information（PSI） {#pressure-stall-information-psi}

PSIは、タスクがI/Oを待って停止している時間の割合を測定する。（デバイスのビジー度を測る）使用率のメトリクスとは異なり、PSIは**アプリケーションへの影響**、つまりワークロードが実際に待つのに費やした時間を測定する。

**ソース**: [`kernel/sched/psi.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/psi.c)。`CONFIG_PSI=y` が必要（5.2以降はデフォルトで有効）。

### /proc/pressure/io {#procpressureio}

```bash
$ cat /proc/pressure/io
some avg10=0.12 avg60=0.08 avg300=0.03 total=47382910
full avg10=0.00 avg60=0.00 avg300=0.00 total=8241033
```

**`some`**: **少なくとも1つのタスク**がI/Oを待って停止していた時間の割合。例では、直近10秒間の0.12%の時間で、少なくとも1つのタスクがI/Oでブロックされていた。これはいずれかのタスクに影響するあらゆるI/O停止を表す。

**`full`**: **すべての実行可能タスク**が同時にI/Oで停止していた時間の割合。システム全体がI/Oでブロックされ、どの実行可能タスクも進めなかった状態である。これはI/O飽和のより強いシグナルである。

**`avg10`、`avg60`、`avg300`**: それぞれ10秒、60秒、300秒にわたる指数加重移動平均。ロードアベレージに似ているが、I/O停止の割合についてのものである。

**`total`**: 起動以降に停止状態で費やした累積マイクロ秒数。差分の計算に有用である。

### PSIの値の解釈 {#interpreting-psi-values}

| 値 | 意味 |
|-------|---------|
| `some` < 1% | I/O停止はまれであり、目立った負荷はない |
| `some` 1–10% | 中程度の停止。一部のタスクがときどきI/Oを待つ |
| `some` > 10% | 大きなI/O負荷。iostatとblktraceで調査する |
| `full` > 0%（持続） | すべての実行可能タスクがブロックされている。I/Oが明確なボトルネックである |
| `full` > 1% | 深刻な飽和。システム全体がI/Oを待っている |

`some` と `full` の差からは情報が得られる。`some` が大きく `full` がほぼ0の場合、I/Oは一部のタスクに影響しているが、他のタスクはまだ進める。`full` が `some` に近づく場合、I/O停止は実質的にすべての活動に影響している。

### cgroupのio.pressure {#cgroup-iopressure}

cgroup v2の階層では、各cgroupが独自のI/O負荷を公開する。

```bash
cat /sys/fs/cgroup/system.slice/io.pressure
some avg10=0.43 avg60=0.21 avg300=0.09 total=1283910
full avg10=0.00 avg60=0.00 avg300=0.00 total=29341
```

これにより、システム全体の計装なしに、I/O負荷を特定のサービスに帰属させられる。システムレベルの `some` が低いのに特定のcgroupの `some` が高い場合、そのサービスが他と比べてI/Oバウンドであることを示す。

### systemd-oomdとの統合 {#integrating-with-systemd-oomd}

`systemd-oomd` はPSIの値を監視し、設定可能な負荷のしきい値を超えたcgroupをkillできる。その設定（`/etc/systemd/oomd.conf`）は `DefaultMemoryPressureLimit=` と `DefaultMemoryPressureDurationSec=` を使う。これらはメモリのPSIの設定だが、同じ仕組みがcgroupのリソース制御を通じてI/Oにも適用される点に注意する。

### PSIアラートの設定 {#setting-up-psi-alerts}

```bash
# I/O停止が10秒間で5%を超えたらアラートする（カーネルのPSI通知を使う）
# ファイルをO_RDWRで開き、トリガーを書き込んでから、イベントをpollする
# シェルからなら、スクリプトを使う方が簡単:

while true; do
    some=$(awk '/^some/ {print $2}' /proc/pressure/io | cut -d= -f2)
    if awk "BEGIN {exit !($some > 5.0)}"; then
        echo "$(date): I/O PSI some=$some% exceeds threshold"
    fi
    sleep 10
done
```

本番環境では、Prometheusの `node_exporter`（バージョン1.1以降）のようなツールが、PSIのメトリクスを `node_pressure_io_stalled_seconds_total` として公開している。

---

## 実践的なデバッグシナリオ {#practical-debugging-scenarios}

### シナリオ1: 「ディスク使用率は100%なのにレイテンシは低い」 {#scenario-1-my-disk-is-100-utilised-but-latency-is-low}

**症状**: `iostat -x` がNVMeデバイスで `%util=100` を示すが、`r_await` と `w_await` は1 msを大きく下回っている。アプリケーションの性能は正常である。

**説明**: `%util` が測るのは、インターバルの各ミリ秒の間にI/Oが処理中だったかどうかであり、デバイスにどれだけ余裕があるかではない。高性能なNVMeデバイスは、多数の並列ハードウェアキューにわたって数万IOPSを持続できる。各キューは1つのI/Oを0.1 msで処理するかもしれないが、16個のキューが並列に動いていれば、デバイスが飽和からほど遠くても `%util` は100%を示す。

**診断手順**:

```bash
# 実際のキュー深度とレイテンシを確認する
iostat -x 1

# aqu-szを見る: 1に近ければ、デバイスはほとんどキューイングされていない
# r_awaitがデバイスの仕様を下回っていれば、余裕がある

# NVMeのキュー数を確認する
ls /sys/class/nvme/nvme0/nvme0n1/
cat /sys/class/nvme/nvme0/num_queues

# キューごとの深度を調べる
ls /sys/block/nvme0n1/mq/
```

**解決**: `%util` はNVMeにとって正しい飽和のメトリクスではない。レイテンシが悪化しているかを評価するには `await`（理想的には `r_await` と `w_await`）を使う。容量を追加したりI/Oを減らしたりするのは、`await` がSLAのしきい値を超えて上昇している場合だけにする。

### シナリオ2: 「書き込みレイテンシが30秒ごとにスパイクする」 {#scenario-2-write-latency-spikes-every-30-seconds}

**症状**: アプリケーションの書き込みレイテンシは通常1–2 msだが、約30秒の規則的な間隔で50–200 msにスパイクし、その後回復する。

**説明**: カーネルのdirtyページのflusher（`wb_workfn`）は `dirty_writeback_centisecs`（デフォルト500 = 5秒）ごとに起床し、`dirty_expire_centisecs`（デフォルト3000 = 30秒）の間dirtyだったページを書き込む。ライトバックが大きなI/Oのバーストを生むほどdirtyページが溜まると、アプリケーションからのリクエストは、ブロックキュー内でライトバックのフラッシュの後ろで待たなければならない。

**診断手順**:

```bash
# ライトバック間隔の設定を確認する
sysctl vm.dirty_writeback_centisecs vm.dirty_expire_centisecs

# dirtyページの蓄積をリアルタイムで監視する
watch -n 1 'grep -E "^Dirty:|^Writeback:|^nr_dirtied|^nr_written" /proc/meminfo /proc/vmstat 2>/dev/null'

# 書き込みレイテンシのスパイクとライトバックのイベントを関連付ける
echo 1 > /sys/kernel/debug/tracing/events/writeback/writeback_single_inode/enable
cat /sys/kernel/debug/tracing/trace_pipe &
# ... レイテンシのスパイクを再現する ...
echo 0 > /sys/kernel/debug/tracing/events/writeback/writeback_single_inode/enable

# ライトバック中にディスクが飽和しているかを確認する
iostat -x 1 | grep -E "Device|nvme|sda"
```

**解決の選択肢**:

1. `dirty_expire_centisecs` を減らし、より小さなバッチでより頻繁にフラッシュする:
   ```bash
   sysctl vm.dirty_expire_centisecs=500   # 5秒後にページをフラッシュする
   sysctl vm.dirty_writeback_centisecs=100  # 1秒ごとに確認する
   ```
2. `dirty_bytes` または `dirty_ratio` を減らし、スロットリングが始まる前に溜まりうるdirtyデータの総量に上限を設ける。
3. アプリケーションで適切な粒度の `sync` または `fsync` を使い、フラッシュのタイミングを明示的に制御する。
4. I/Oスケジューリングを使う: `blk-mq` で `kyber` スケジューラを使うと、フォアグラウンドの読み込みをライトバックより優先できる。

### シナリオ3: 「読み込みが遅いのにディスクには読み込みが見えない」 {#scenario-3-reads-are-slow-but-disk-shows-no-reads}

**症状**: アプリケーションの読み込みレイテンシが高い。`iostat` はほぼ0の読み込みスループットを示す。`/proc/vmstat` では `pgmajfault` が高くなっている。マシンの空きメモリが少ない。

**説明**: アプリケーションのワーキングセットはページキャッシュにあったが、メモリ逼迫によってカーネルがそれらのページを追い出した（`mm_filemap_delete_from_page_cache` 経由）。今では、追い出されたページへの各アクセスがメジャーページフォールトを生み、カーネルがディスクからページを読み戻す間プロセスをブロックする。ディスクの読み込み単体はすぐに完了するかもしれないが、メジャーフォールトのレートが、アプリケーションから見て持続的なレイテンシを生み出す。

**診断手順**:

```bash
# メジャーフォールトのレートを確認する
grep pgmajfault /proc/vmstat
# 増えていく様子を監視する
watch -n 1 'grep pgmajfault /proc/vmstat'

# メモリ逼迫を確認する
grep -E "^(MemAvailable|Active|Inactive|Cached|Dirty):" /proc/meminfo

# 回収の活動を確認する
grep -E "pgscan_kswapd|pgscan_direct|pgsteal" /proc/vmstat

# filemapのtracepointで、どのファイルが追い出されているかを特定する
echo 1 > /sys/kernel/debug/tracing/events/filemap/mm_filemap_delete_from_page_cache/enable
cat /sys/kernel/debug/tracing/trace_pipe | head -100
echo 0 > /sys/kernel/debug/tracing/events/filemap/mm_filemap_delete_from_page_cache/enable
```

**解決**:

- キャッシュを押し出しているメモリを消費しているプロセスまたはcgroupを特定する。`smem` やcgroupの `memory.stat` で、どのプロセスが原因かがわかる。
- 重要なファイルをページキャッシュに固定する: `vmtouch -l /path/to/file`（ユーザー空間）または `mlock()`（アプリケーション）。
- cgroup v2のメモリ制限を使い、他のワークロードが共有キャッシュを追い出さないようにする。
- メモリを追加するか、競合するワークロードを別のマシンに移す。

### シナリオ4: 「プロセスが想定より多くのI/Oを行っている」 {#scenario-4-a-process-is-doing-more-io-than-expected}

**症状**: `iotop` で特定のプロセスが毎秒何MBも書き込んでいる。アプリケーションのログでは説明がつかない。I/Oの量が、想定される論理的な書き込みレートよりはるかに多い。

**診断手順**:

```bash
# プロセスのI/Oの内訳を取得する
pid=$(pgrep suspect_app)
cat /proc/$pid/io

# wchar（論理的な書き込み）とwrite_bytes（実際のストレージへの書き込み）を比較する
# write_bytes >> wchar / (page_size) の場合: アプリケーションがバッファリングせずに
# 小さなI/Oを大量に書いているか、過剰なメタデータ更新（例: 書き込みごとのfsync）を行っている

# fsync/fdatasync呼び出しを確認する
strace -p $pid -e trace=fsync,fdatasync,sync_file_range -T 2>&1 | head -50

# ブロック書き込みのコールスタックを記録する
perf record -e block:block_rq_issue -p $pid -ag -- sleep 10
perf report

# 同じファイルに対して別のプロセスがライトバックを行っていないかを確認する
# （例: バックアップツールが読み込みを行い、dirtyページのフラッシュを引き起こしている）
lsof -p $pid | grep -v mem   # 開いているファイルディスクリプタ
```

**よくある原因**:

- アプリケーションが書き込みのたびに `fsync()` を呼んでいる: `strace -e fsync` で確認する。各fsyncは、そのファイルディスクリプタのdirtyページ範囲全体をフラッシュする。
- ログのローテーションや監視エージェントが大きなファイルを読み、ページキャッシュを押し出して、別の場所での再読み込みを引き起こしている。
- `O_DIRECT` のI/Oがページキャッシュをバイパスしている: `wchar` ≈ `write_bytes` がその特徴である（バッファリングによる差がない）。
- SQLiteなどの組み込みデータベースが積極的にWALをフラッシュしている。

```bash
# O_DIRECTのファイルディスクリプタを確認する
ls -la /proc/$pid/fd | xargs -I{} sh -c 'fdinfo=$(cat /proc/$pid/fdinfo/{} 2>/dev/null); echo "$fdinfo" | grep -i "flags.*0100000"'

# あるいはもっと手軽に:
for fd in /proc/$pid/fd/*; do
    flags=$(awk '/^flags:/{print $2}' /proc/$pid/fdinfo/$(basename $fd) 2>/dev/null)
    # O_DIRECTは0100000（8進数）= 0x8000
    [ -n "$flags" ] && printf '%d\n' "0x$flags" | grep -q '^[0-9]' && \
      echo "fd $(basename $fd): flags=$flags"
done
```

---

## クイックリファレンス: ツールの選び方 {#quick-reference-tool-selection}

| 問い | ツール |
|----------|------|
| デバイスは飽和しているか？ | `iostat -x 1` — `%util`、`r_await`、`w_await` を監視する |
| どのプロセスがI/Oを行っているか？ | `iotop -o` または `/proc/PID/io` の差分 |
| ブロックスタックのどこでレイテンシが加わっているか？ | Q2G/G2I/I2D/D2Cの内訳を得るための `blktrace` + `btt` |
| どのinodeがライトバックを生んでいるか？ | `writeback:writeback_single_inode` tracepoint |
| dirtyページのライトバックは追いついているか？ | `/proc/meminfo` の `Dirty:`、`/proc/vmstat` の `nr_dirtied` と `nr_written` |
| I/Oがワークロードを停止させているか？ | `/proc/pressure/io` のPSIの値 |
| ページキャッシュがスラッシングしているか？ | `filemap:mm_filemap_delete_from_page_cache` のレート、`/proc/vmstat` の `pgmajfault` |
| どのコールスタックがI/Oを引き起こしているか？ | `perf record -e block:block_rq_issue -ag` |
| fsyncによって書き込みが増幅されているか？ | `strace -e fsync,fdatasync`、`block_rq_issue` の回数と `write()` の回数 |
| デバイスごとのライトバックのチューニング | `/sys/class/bdi/<dev>/max_ratio`、`read_ahead_kb` |

---

## 主要なソースファイル {#key-source-files}

| ファイル | 説明 |
|------|-------------|
| [`block/genhd.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/genhd.c) | `/proc/diskstats` の出力生成 |
| [`kernel/trace/blktrace.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/trace/blktrace.c) | blktraceのカーネル内トレーシング基盤 |
| [`include/trace/events/block.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/block.h) | ブロック層のtracepoint定義 |
| [`include/trace/events/writeback.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/writeback.h) | ライトバックのtracepoint定義 |
| [`include/trace/events/filemap.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/filemap.h) | ページキャッシュのtracepoint定義 |
| [`mm/page-writeback.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page-writeback.c) | dirtyページのスロットリングとライトバック制御 |
| [`mm/backing-dev.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/backing-dev.c) | BDI（backing device info）サブシステム |
| [`kernel/sched/psi.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/psi.c) | Pressure Stall Informationの実装 |
| [`fs/proc/base.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/proc/base.c) | `/proc/PID/io` の集計 |
| [`include/linux/task_io_accounting.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/task_io_accounting.h) | タスクごとのI/O集計の構造体 |

---

## 参考文献 {#further-reading}

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/block/stat.rst`](https://docs.kernel.org/block/stat.html) — `/proc/diskstats` と `/sys/block/<dev>/stat` のフィールドに関する公式ドキュメント
- [`Documentation/admin-guide/sysctl/vm.rst`](https://docs.kernel.org/admin-guide/sysctl/vm.html) — dirtyページのライトバックのチューニングを含む、すべての `vm.*` sysctlパラメータ
- [`Documentation/accounting/psi.rst`](https://docs.kernel.org/accounting/psi.html) — PSIの設計、解釈、通知インターフェース

### 関連ページ {#related-pages}

- [page-cache-writeback.md](page-cache-writeback.md) — dirtyページがどのように管理・フラッシュされるか
- [buffered-io.md](buffered-io.md) — `write()` からストレージまでのバッファード書き込みの全経路
- [readahead.md](readahead.md) — カーネルがページをどのように先読みするか、readaheadをどうチューニングするか
- [direct-io.md](direct-io.md) — O_DIRECT I/O: ページキャッシュのバイパスとそのトレードオフ

### LWNの記事 {#lwn-articles}

- [Tracking pressure-stall information](https://lwn.net/Articles/759781/) — PSIの設計と動機
- [LWN: The io.weight I/O-bandwidth controller](https://lwn.net/Articles/792256/) — cgroup v2のI/O帯域コントローラ
- [LWN: The multiqueue block layer](https://lwn.net/Articles/552904/) — blk-mqの設計
