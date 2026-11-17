# ストレージ負荷の高いワークロード向けI/O性能チューニング {#tuning-io-performance-for-storage-heavy-workloads}

> カーネルがI/Oに対して何を行うのか、そしてデフォルト設定が邪魔にならなくなるのはどこかを理解する

## なぜI/Oチューニングにはワークロードモデルが必要か {#why-io-tuning-requires-a-workload-model}

Webサーバー、OLTPデータベース、時系列データの取り込みパイプライン、バックアップジョブはいずれも同じブロック層を使うが、必要な設定はまったく異なる。sysctlを1つでも触る前に、次の点を押さえておく必要がある。

- ランダム4KリードでNVMeのレイテンシを半分にする設定が、シーケンシャル1Mリードのスループットを下げることがある。
- readaheadの無効化は、PostgreSQLのインデックススキャンには不可欠だが、ログのストリーミングには明確に有害である。
- 「正しい」I/Oスケジューラは、ハードウェアキューを64本持つNVMeが1台なのか、32エントリのNCQキューを持つ回転ディスクなのかによって変わる。

ワークロードモデルなしのチューニングは当て推量にすぎない。以下の節では、まずワークロードの特性を把握する方法を説明し、次にワークロードの形ごとに具体的な調整項目を示す。

## まずワークロードを理解する {#understanding-your-workload-first}

### 4つの軸 {#the-four-axes}

あらゆるI/Oワークロードは、次の4次元空間のどこかに位置する。

```
1. Access pattern     random        ←————————→   sequential
2. Direction          write-heavy   ←————————→   read-heavy
3. Optimization goal  latency       ←————————→   throughput
4. Queue depth        shallow (1)   ←————————→   deep (128+)
```

これらは独立していない。シーケンシャルI/Oは深いキューと大きなreadaheadの恩恵を受け、低いキュー深度のランダムI/Oは操作あたりのレイテンシが低いことを必要とする。ある軸を最適化するチューニング可能なパラメータを選ぶと、別の軸が悪化することがある。

代表的なワークロードの典型例:

| ワークロード | パターン | 目標 | 典型的なキュー深度 |
|----------|---------|------|---------------------|
| OLTPデータベース（pg, mysql） | ランダム 4–16K | レイテンシ | 4–32 |
| データウェアハウス / OLAP | シーケンシャルスキャン、大きなブロック | スループット | 32–256 |
| 時系列データの取り込み | シーケンシャルライト、小さな書き込みが多数 | スループット + レイテンシ | 8–64 |
| バックアップ / リストア | シーケンシャル | スループット | 32–128 |
| オブジェクトストレージ | ランダムとシーケンシャルの混在 | スループット | 32–256 |
| WAL / redoログ | シーケンシャルライト | レイテンシ（fsync） | 1–4 |

### iostatによる手早いプロファイリング {#quick-profiling-with-iostat}

`sysstat` パッケージの `iostat` は、最初に手に取るべきツールである。これは `/proc/diskstats` を読み取る。このファイルはブロック層がI/Oの完了ごとに更新する。

```bash
# 1秒間隔、拡張統計、人間が読みやすい形式
iostat -x -h 1

# 主な列:
#  r/s    w/s    — IOPS（1秒あたりに完了したリード/ライト数）
#  rkB/s  wkB/s  — スループット（kB/s）
#  r_await w_await — 平均完了レイテンシ（ms）
#  aqu-sz          — 平均キュー深度（処理中 + 待機中）
#  %util           — デバイスに未完了のI/Oがあった時間の割合
```

注目すべき点:

- 回転ディスクで `%util` が100%近く、`aqu-sz` が1よりはるかに大きい場合、デバイスは飽和している。
- `r_await` が高い（SSDで5ms超、HDDで20ms超）のに `%util` が低い場合、I/Oスケジューラがキューイングのオーバーヘッドを加えていることが多い。
- NVMeで `aqu-sz` が高く `%util` が100%を大きく下回るのは想定どおりである。NVMeデバイスは深い内部キューを持ち、自由に並列処理する。

### fioによるプロファイリング {#profiling-with-fio}

`fio` を使うと、制御された再現可能なワークロードを発行し、アプリケーションの負荷をかける前にデバイス本来の能力を測定できる。

```bash
# ランダム4Kリード、キュー深度32、30秒 — OLTPのベースライン
fio --name=randread-baseline \
    --filename=/dev/nvme0n1 \
    --rw=randread \
    --bs=4k \
    --iodepth=32 \
    --numjobs=1 \
    --runtime=30 \
    --time_based \
    --direct=1 \
    --lat_percentiles=1 \
    --output-format=normal

# シーケンシャル1Mリード — ストリーミングワークロードのベースライン
fio --name=seqread-baseline \
    --filename=/dev/nvme0n1 \
    --rw=read \
    --bs=1m \
    --iodepth=4 \
    --numjobs=4 \
    --runtime=30 \
    --time_based \
    --direct=1 \
    --output-format=normal
```

!!! warning "デバイスの特性評価にはファイルではなくブロックデバイスに対して実行する"
    ファイルに対して `fio` を実行すると、ファイルシステムとページキャッシュを経由するため、ばらつきが加わる。デバイスのベースライン測定には、生のブロックデバイス（`/dev/nvme0n1`）を使う。ファイルに切り替えるのは、ファイルシステムのオーバーヘッドを評価するときだけにする。

### blktraceによる詳細なトレーシング {#deep-tracing-with-blktrace}

`iostat` で問題は見えるが原因がわからない場合、`blktrace` を使うと、ブロック層がリクエストごとに観測するすべてのイベント（キューへの挿入、マージ、ドライバへのディスパッチ、完了）を記録できる。

```bash
# /dev/sda を10秒間トレースする
blktrace -d /dev/sda -o /tmp/sda-trace -- sleep 10

# 人間が読める形式に後処理する
blkparse -i /tmp/sda-trace.blktrace.* -o /tmp/sda-trace.txt

# 集計統計を生成する
btt -i /tmp/sda-trace.blktrace.*

# btt出力の主なセクション:
#  D2C  — ドライバへのディスパッチから完了までの時間（デバイスのレイテンシ）
#  Q2C  — キューへの挿入から完了までの時間（全体のレイテンシ）
#  Q2D  — ソフトウェアキューでの待ち時間（Q2C - D2C）
```

`Q2C` に比べて `Q2D` が大きい場合、ソフトウェアキューがレイテンシを加えており、スケジューラのチューニングが有効である。`D2C` が支配的な場合は、デバイス自体がボトルネックである。

### チューニング前にベンチマークを取るという原則 {#the-benchmark-before-you-tune-principle}

何かを変更する前にベースラインの数値を記録しておく。変更のたびに、同じパラメータで同じベンチマークを再実行する。99パーセンタイルのレイテンシを改善する一方でスループットを下げる変更が改善かどうかは、ワークロード次第である。判断には両方の数値が必要になる。

```bash
# 最小限だが再現可能なベースラインスクリプト
#!/bin/bash
DEVICE=${1:-/dev/nvme0n1}
echo "=== Baseline: $(date) === Device: $DEVICE ==="

# ランダム4K、単一キュー
fio --name=rand4k-qd1 --filename=$DEVICE --rw=randread \
    --bs=4k --iodepth=1 --numjobs=1 --runtime=30 \
    --time_based --direct=1 --lat_percentiles=1 \
    --output-format=terse --terse-version=5 2>/dev/null \
    | awk -F';' '{printf "rand4k-qd1: IOPS=%s lat_p99=%sµs lat_p999=%sµs\n",$49,$83,$84}'

# ランダム4K、深いキュー
fio --name=rand4k-qd32 --filename=$DEVICE --rw=randread \
    --bs=4k --iodepth=32 --numjobs=4 --runtime=30 \
    --time_based --direct=1 --lat_percentiles=1 \
    --output-format=terse --terse-version=5 2>/dev/null \
    | awk -F';' '{printf "rand4k-qd32: IOPS=%s lat_p99=%sµs lat_p999=%sµs\n",$49,$83,$84}'
```

## I/Oスケジューラの選択 {#io-scheduler-selection}

I/Oスケジューラは、ブロック層のリクエストキューとデバイスドライバの間に位置する。最近のカーネルでは、すべてのスケジューラがマルチキューブロック層（`blk-mq`）の下で動作する。`blk-mq` はソフトウェアキューをハードウェアのディスパッチキューに対応付ける。

### スケジューラの確認と設定 {#checking-and-setting-the-scheduler}

```bash
# デバイスの有効なスケジューラと選択可能なスケジューラを表示する
cat /sys/block/nvme0n1/queue/scheduler
# [none] mq-deadline kyber bfq

# 角括弧内の値が有効なスケジューラ

# 実行時に設定する
echo mq-deadline > /sys/block/nvme0n1/queue/scheduler

# udevルールで永続化する
cat > /etc/udev/rules.d/60-ioscheduler.rules << 'EOF'
# NVMe: スケジューラなし（パススルー）
ACTION=="add|change", KERNEL=="nvme[0-9]*", ATTR{queue/scheduler}="none"

# SATA SSD: 予測可能なレイテンシのためにmq-deadline
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="mq-deadline"

# HDD: 公平性のためにbfq
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="bfq"
EOF

udevadm control --reload-rules
udevadm trigger --type=devices --action=change
```

### none（パススルー） {#none-passthrough}

`none` は並べ替えを完全に無効にする。リクエストは発行された順にデバイスへディスパッチされる。NVMeにはこれが正しい選択である。理由は次のとおり。

- NVMeデバイスは内部にコマンドキュー（キューあたり最大64Kエントリ、最大64Kキュー）を持ち、ファームウェア内で独自に並べ替えを行う。
- ソフトウェアによる並べ替えは、利点なしにレイテンシを加えるだけである。ドライブは自身のアクセスパターンについてスケジューラよりも賢い。
- NVMeのレイテンシはすでに1桁マイクロ秒の範囲にあり、スケジューラのオーバーヘッドを加えると、何の得もなくそれが倍になる。

```bash
echo none > /sys/block/nvme0n1/queue/scheduler
```

### mq-deadline {#mq-deadline}

`mq-deadline` は各リクエストに期限（デッドライン）を割り当て、その期限までにディスパッチされることを保証する。また、リードとライトを別々にバッチ処理し、ライトがリードを枯渇させるのを防ぐ。

これは、SATA SSD上のレイテンシに敏感なワークロードにとって最良の汎用スケジューラである。

```bash
echo mq-deadline > /sys/block/sda/queue/scheduler

# デッドラインとバッチのパラメータを調整する
# リードのデッドライン（ms、デフォルトは500ms — データベースでは減らす）
echo 100 > /sys/block/sda/queue/iosched/read_expire

# ライトのデッドライン（ms、デフォルトは5000ms）
echo 1000 > /sys/block/sda/queue/iosched/write_expire

# 1バッチでディスパッチするシーケンシャルなリクエストの数
echo 16 > /sys/block/sda/queue/iosched/fifo_batch

# リードあたりにディスパッチするライト（1 = リードバッチ1回につきライトバッチ1回）
echo 1 > /sys/block/sda/queue/iosched/writes_starved
```

データベースのWALディスク（SATA SSD）では、`read_expire` と `write_expire` を下げると、並べ替えの効率を多少犠牲にしてテールレイテンシを減らせる。

### kyber {#kyber}

`kyber` は高速なSSD向けに設計された、レイテンシ目標型のスケジューラである。リードと同期ライトに別々のキューを持ち、設定可能なレイテンシ目標を達成するようにディスパッチレートを動的に調整する。

```bash
echo kyber > /sys/block/sda/queue/scheduler

# リードの目標レイテンシ（ナノ秒、デフォルトは2ms）
echo 500000 > /sys/block/sda/queue/iosched/read_lat_nsec

# 同期ライトの目標レイテンシ（デフォルトは10ms）
echo 2000000 > /sys/block/sda/queue/iosched/write_lat_nsec
```

kyberは、SSDのデバイスレイテンシが比較的安定している場合にうまく機能する。デバイス自体のレイテンシのばらつきが大きい場合（安価なコンシューマ向けSSDで書き込み負荷がかかったときによくある）、kyberのフィードバックループが振動することがある。その場合は、明示的な期限を設定した `mq-deadline` の方が予測しやすい。

### bfq（Budget Fair Queuing） {#bfq-budget-fair-queuing}

`bfq` はプロセスごとのI/Oバジェットを追跡する比例配分型のスケジューラである。複数のプロセス間の公平性が生のスループットよりも重要な、デスクトップ／対話的用途向けに設計されている。

専用のデータベースサーバーでは、`bfq` は利点なしにオーバーヘッドを加えるだけである。公平に扱うべきワークロードが1つしかないからだ。次の場合に使う。

- システムが混在ワークロード（データベース + バックアップ + 分析）を実行しており、対話的なI/Oをバックグラウンドジョブから守る必要がある。
- デバイスが回転型HDDで、並べ替えの判断がスループットに本当に影響する。

```bash
echo bfq > /sys/block/sda/queue/scheduler

# bfqとともにcgroups v2のio.weightを使い、cgroupごとのI/O優先度を制御する
echo "100" > /sys/fs/cgroup/databases/io.bfq.weight
echo "10"  > /sys/fs/cgroup/backups/io.bfq.weight
```

### スケジューラがレイテンシ分布に与える影響 {#scheduler-effect-on-latency-distribution}

スケジューラの選択が主に影響するのは、レイテンシの中央値ではなく分布（p99、p999）である。NVMe上でよくチューニングされた `none` は、たとえば次のようになる。

```
Median (p50):  50µs
p99:           120µs
p999:          200µs
```

一方、同じNVMe上の `mq-deadline` は次のようになることがある。

```
Median (p50):  55µs    (slightly worse — scheduling overhead)
p99:           85µs    (better — deadline enforcement prevents outliers)
p999:          110µs   (better — expiry prevents starvation)
```

中央値がわずかに高くなる代わりにテールレイテンシが改善するというこのトレードオフは、p99がユーザーから見えるレイテンシに直結するデータベースでは、通常は正しい選択である。

## キュー深度のチューニング {#queue-depth-tuning}

キュー深度は、同時に処理中にできるI/Oリクエストの数を制御する。浅すぎるとデバイスを使い切れず、深すぎるとリクエストあたりのキューイングレイテンシが増える。

### ソフトウェアのキュー深度: nr_requests {#software-queue-depth-nr_requests}

`nr_requests` は、発行元にバックプレッシャーがかかる前の、ソフトウェアキューあたりの最大リクエスト数を設定する。

```bash
# 現在の値を確認する
cat /sys/block/nvme0n1/queue/nr_requests
# 64（NVMeのデフォルト）

# 高スループットのワークロードでは増やす
echo 256 > /sys/block/nvme0n1/queue/nr_requests

# レイテンシに敏感なワークロードでは減らす（バックプレッシャーを早めにかける）
echo 32 > /sys/block/nvme0n1/queue/nr_requests
```

数十万IOPSを持続できるNVMeデバイスでは、`nr_requests` を増やすことで、カーネルはデバイス内部のスケジューラ向けにより多くのリクエストをバッファできる。IOPSの上限がはるかに低いSATA SSDでは、通常はデフォルトで十分である。

### ハードウェアキューの数: nr_hw_queues {#hardware-queue-count-nrhwqueues}

NVMeデバイスは複数のハードウェア発行キューを公開する。カーネルはソフトウェアキューをハードウェアキューに対応付け、理想的にはCPUコアまたはNUMAノードと1対1にする。

```bash
# ハードウェアキューの数を確認する
cat /sys/block/nvme0n1/queue/nr_hw_queues
# 16（例: 16キューのNVMe）

# blk-mqがCPUをキューにどう対応付けているかを確認する
cat /sys/block/nvme0n1/mq/*/cpu_list
```

`nr_hw_queues` がCPUコア数より少ない場合、複数のCPUが1つのハードウェアキューを共有し、そのロックで競合する。コア数が多くNVMeキューが少ないシステムでは、これが `perf top` に `nvme_irq` として現れる。

```bash
# CPU間のIRQ分散を確認する（各NVMeキューに1つのIRQがある）
grep nvme /proc/interrupts | head -20
```

### fioで飽和点を見つける {#finding-the-saturation-point-with-fio}

最適なキュー深度は、リクエストあたりのレイテンシを大きく増やさずにIOPSがピークに達する点である。`iodepth` を掃引して見つける。

```bash
#!/bin/bash
DEVICE=${1:-/dev/nvme0n1}
for QD in 1 2 4 8 16 32 64 128 256; do
    RESULT=$(fio --name=qd-sweep \
        --filename=$DEVICE \
        --rw=randread \
        --bs=4k \
        --iodepth=$QD \
        --numjobs=1 \
        --runtime=10 \
        --time_based \
        --direct=1 \
        --output-format=terse \
        --terse-version=5 2>/dev/null)
    IOPS=$(echo "$RESULT" | awk -F';' '{print $49}')
    LAT_P99=$(echo "$RESULT" | awk -F';' '{print $83}')
    printf "qd=%-4d  iops=%-8s  lat_p99=%s µs\n" $QD "$IOPS" "$LAT_P99"
done
```

NVMeでの典型的な結果は次のようになる。

```
qd=1     iops=50000    lat_p99=35 µs
qd=4     iops=185000   lat_p99=38 µs
qd=16    iops=450000   lat_p99=52 µs
qd=32    iops=520000   lat_p99=89 µs     ← diminishing returns start
qd=64    iops=540000   lat_p99=155 µs
qd=128   iops=542000   lat_p99=290 µs    ← IOPS plateau, latency rising fast
```

OLTPデータベースでは、IOPSがまだ伸びており、かつレイテンシがSLAに対してまだ許容範囲にある（p99 < 1msがよくある目標）キュー深度を選ぶ。

### 深いキューでのio_uring {#io_uring-with-deep-queues}

`io_uring` は、システムコールごとのコンテキストスイッチのオーバーヘッドを回避する最新の非同期I/Oインターフェースである。1回の `io_uring_enter()` 呼び出しで複数のI/Oリクエストを発行できるため、高いキュー深度で真価を発揮する。

```bash
# さまざまなキュー深度でio_uringとlibaioをベンチマークする
fio --name=io_uring_deep \
    --ioengine=io_uring \
    --filename=/dev/nvme0n1 \
    --rw=randread \
    --bs=4k \
    --iodepth=128 \
    --numjobs=4 \
    --runtime=30 \
    --time_based \
    --direct=1 \
    --lat_percentiles=1

# libaioと比較する
fio --name=libaio_deep \
    --ioengine=libaio \
    --filename=/dev/nvme0n1 \
    --rw=randread \
    --bs=4k \
    --iodepth=128 \
    --numjobs=4 \
    --runtime=30 \
    --time_based \
    --direct=1 \
    --lat_percentiles=1
```

キュー深度が32を超えると、`io_uring` は通常 `libaio` より10–20%高いIOPSを示す。システムコールのオーバーヘッドが減ることと、完了をまとめて処理することでCPUキャッシュの挙動が良くなることによる。

## readaheadのチューニング {#readahead-tuning}

カーネルのreadahead機構（詳細は[readahead](readahead.md)で説明）は、明示的に要求される前にページをページキャッシュへ先読みする。デフォルトの `read_ahead_kb` はデバイスごとに設定され、1つのウィンドウで先読みする最大バイト数を表す。

### read_ahead_kbの確認と設定 {#checking-and-setting-readaheadkb}

```bash
# 現在の値を確認する
cat /sys/block/nvme0n1/queue/read_ahead_kb
# 128（典型的なデフォルト）

# SATA HDDの場合を確認する
cat /sys/block/sda/queue/read_ahead_kb
# 128

# デバイスごとに設定する
echo 4096 > /sys/block/sda/queue/read_ahead_kb   # HDD上のストリーミングワークロード
echo 16   > /sys/block/nvme0n1/queue/read_ahead_kb # NVMe上のデータベースのランダムI/O
```

### readaheadを減らすべきとき {#when-to-reduce-readahead}

**ランダムI/Oのワークロード** — データベースのインデックススキャンは、8–16KBのページにランダムにアクセスする。readaheadは決して読まれない隣接ページを先読みし、ページキャッシュの領域を浪費し、不要なI/Oを発生させる。PostgreSQLサーバーでは次のように設定する。

```bash
# 最小限のreadahead: カーネルのシーケンシャル検出器が要求する分だけ先読みする
echo 16 > /sys/block/nvme0n1/queue/read_ahead_kb
```

**独自のキャッシュを持つデバイス** — 大きなDRAMキャッシュを持つNVMeドライブは、すでに内部で先読みを行っている。ドライブの先読みの上にカーネルのreadaheadを重ねると、作業が二重になる。

### readaheadを増やすべきとき {#when-to-increase-readahead}

**シーケンシャルなストリーミング** — ログの取り込みパイプライン、動画のトランスコード、バックアップジョブは、大きなファイルをシーケンシャルに読む。readaheadのウィンドウを大きくすると、ページキャッシュをより先まで埋めることで実際のI/Oの頻度が減る。

```bash
# HDD上のシーケンシャルなストリーミング向けの積極的なreadahead
echo 8192 > /sys/block/sda/queue/read_ahead_kb   # 8MBのウィンドウ
```

**ネットワーク接続ストレージ** — レイテンシの高いブロックデバイス（iSCSI、NBD）では、往復のレイテンシが支配的なため、小さなシーケンシャルアクセスパターンであっても積極的なreadaheadの恩恵を受ける。

### fadviseによるファイル単位のreadahead {#per-file-readahead-with-fadvise}

アプリケーションは、システム全体の設定を変えずに、自身のアクセスパターンをカーネルに助言できる。カーネルはこれらのヒントを使い、特定のファイル記述（file description）に対するreadaheadのウィンドウを調整する。

```c
#include <fcntl.h>

int fd = open("data.db", O_RDONLY);

// カーネルに伝える: ランダムアクセスなので、このfdのreadaheadを無効にする
posix_fadvise(fd, 0, 0, POSIX_FADV_RANDOM);

// カーネルに伝える: シーケンシャルアクセスなので、このfdでは積極的に先読みする
posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL);

// この範囲がまもなくアクセスされることを示すヒント
posix_fadvise(fd, offset, length, POSIX_FADV_WILLNEED);

// 使い終わったらキャッシュされたページを破棄する（1回きりの読み込みでキャッシュを汚さないため）
posix_fadvise(fd, offset, length, POSIX_FADV_DONTNEED);
```

PostgreSQLはシーケンシャルスキャンの後に `POSIX_FADV_DONTNEED` を使い、再利用されないスキャンデータでページキャッシュがあふれるのを防いで、インデックス検索で使われるバッファプールのページを守っている。

## dirtyページのライトバックのチューニング {#dirty-writeback-tuning}

dirtyページのライトバックは、変更されたページキャッシュのページをカーネルがどれだけ早くディスクへフラッシュするかを制御する。同じチューニング可能なパラメータは、メモリの観点から[データベースのチューニング](../mm/tuning-databases.md)で扱った。この節ではI/Oへの影響を見る。

### ライトバックのチューニング可能なパラメータ {#the-writeback-tunables}

```bash
# 現在の値
sysctl vm.dirty_ratio vm.dirty_background_ratio \
       vm.dirty_expire_centisecs vm.dirty_writeback_centisecs

# 典型的なデフォルト:
# vm.dirty_ratio = 20
# vm.dirty_background_ratio = 10
# vm.dirty_expire_centisecs = 3000   （30秒）
# vm.dirty_writeback_centisecs = 500 （5秒）
```

- **`dirty_background_ratio`**: dirtyページが利用可能なメモリのこの割合を超えると、バックグラウンドのライトバックが始まる。ライトバックを均等に分散させるため、低め（3–5%）に保つ。
- **`dirty_ratio`**: dirtyページがこの水準に達すると、書き込み側がスロットリングされる。これは強制的な停止であり、書き込み中のプロセスはライトバックが追いつくまでスリープする。
- **`dirty_expire_centisecs`**: これより古いページは、割合のしきい値に達していなくてもライトバックの対象になる。多くのdirtyページを短時間で生み出すバースト的な書き込みワークロードで有用である。
- **`dirty_writeback_centisecs`**: デバイスごとのライトバックスレッドが、フラッシュすべきページを確認するために起床する間隔。

### レイテンシに敏感なワークロード向けのチューニング {#tuning-for-latency-sensitive-workloads}

書き込み停止の問題: `dirty_ratio=20` の256GBサーバーでは、書き込み側が停止するまでに最大51GBのdirtyデータが溜まりうる。チェックポイントやsyncイベントが大規模なフラッシュを引き起こすと、その結果生じるI/Oの嵐がデバイスを飽和させ、WALの書き込みを停止させる。

```bash
# データベース向けの低いdirty比率: 早めに頻繁にフラッシュする
cat >> /etc/sysctl.d/io-database.conf << 'EOF'
vm.dirty_background_ratio = 3
vm.dirty_ratio = 8
vm.dirty_expire_centisecs = 1000
vm.dirty_writeback_centisecs = 200
EOF
sysctl --system
```

あるいは、大容量メモリのサーバーでより予測しやすい、バイト単位の絶対的な上限を使う。

```bash
cat >> /etc/sysctl.d/io-database.conf << 'EOF'
vm.dirty_background_bytes = 134217728    # 128MB: バックグラウンドのライトバックを開始
vm.dirty_bytes = 536870912               # 512MB: 強制停止のしきい値
EOF
```

!!! tip "バイト単位の上限は割合による上限に優先する"
    `dirty_background_bytes` と `dirty_background_ratio` の両方が設定されている場合、バイト単位の上限が有効になり、割合は無視される。割合ベースのしきい値が大きくなりすぎて意味をなさなくなる、64GBを超えるRAMを持つサーバーではバイト単位の上限を使う。

### 書き込み中心のスループット向けのチューニング {#tuning-for-write-heavy-throughput}

レイテンシよりスループットが重要な一括取り込みパイプラインの場合:

```bash
cat >> /etc/sysctl.d/io-bulk.conf << 'EOF'
vm.dirty_background_ratio = 15
vm.dirty_ratio = 40
vm.dirty_expire_centisecs = 6000
vm.dirty_writeback_centisecs = 1000
EOF
```

しきい値を大きくすると、書き込みがページキャッシュ内でまとめられ、より大きなシーケンシャルなバッチでフラッシュされる。これはHDDにとって効率的であり、SSDでもライトバックのマージの恩恵を受ける。

### cgroup v2によるデバイスごとのライトバックのスロットリング {#per-device-writeback-throttling-with-cgroup-v2}

混在ワークロードのサーバーでは、cgroup v2の `io.max` を使うと、システム全体のdirty比率とは独立にデバイスごとのI/Oレート制限をかけられる。

```bash
# /dev/sda（major:minor 8:0）上でバックアップジョブの書き込みを100MB/sに制限する
echo "8:0 wbps=104857600" > /sys/fs/cgroup/backups/io.max

# バックアップジョブを1000 IOPSに制限する
echo "8:0 wiops=1000" > /sys/fs/cgroup/backups/io.max

# 現在の制限を確認する
cat /sys/fs/cgroup/backups/io.max
```

これにより、グローバルなdirty比率の設定に触れることなく、バックアップジョブがデバイスを飽和させてデータベースの書き込みレイテンシに影響するのを防げる。

## O_DIRECTとバッファードI/O {#o_direct-vs-buffered-io}

`O_DIRECT` とバッファードI/Oのどちらを選ぶかは、データベースのI/O性能にとって最も影響の大きい判断の1つである。

### 二重バッファリングの問題 {#the-double-buffering-problem}

32GBのバッファプールを持ち、バッファードI/Oを使うデータベースは、メモリを二重に使う。データベースは自身のプールにデータをバッファし、OSのページキャッシュは同じページの2つ目のコピーを保持する。128GBのサーバーでは、同じデータの2つのコピーで64GBが消費されることもある。

```
Buffered I/O with database:
┌─────────────────────────────────────────┐
│ OS Page Cache (32GB of duplicated data) │
├─────────────────────────────────────────┤
│ Database Buffer Pool (32GB)             │
├─────────────────────────────────────────┤
│ OS + Applications (~10GB)               │
└─────────────────────────────────────────┘
Total: 74GB used, 54GB wasted on duplication

O_DIRECT with database:
┌─────────────────────────────────────────┐
│ Database Buffer Pool (32GB)             │
├─────────────────────────────────────────┤
│ OS + Applications (~10GB)               │
│ Page cache (WAL, temp files, etc.)      │
└─────────────────────────────────────────┘
Total: ~45GB used, 83GB available
```

### データベースがO_DIRECTを使うべきとき {#when-databases-should-use-o_direct}

**PostgreSQL** はデータファイルに対してデフォルトでバッファードI/Oを使うが、`O_DIRECT` 相当の挙動を推奨している。`effective_io_concurrency` を設定し、ダイレクトI/Oとともに `pg_prewarm` を使うのが標準的な方法である。WALについては、PostgreSQLの `wal_sync_method = open_datasync` が `O_DIRECT | O_SYNC` を使う。

**MySQL InnoDB** は `innodb_flush_method = O_DIRECT` のときに `O_DIRECT` を使う。

```ini
# /etc/my.cnf.d/innodb.cnf
[mysqld]
innodb_flush_method = O_DIRECT        # データファイルにO_DIRECT
# innodb_flush_method = O_DIRECT_NO_FSYNC  # バッテリーバックアップ付きキャッシュを使う場合はfsyncを省略
```

**Oracle** は、ファイルシステムが対応していれば、データファイルに対してデフォルトで `O_DIRECT` を有効にする。

### アライメントの要件 {#alignment-requirements}

`O_DIRECT` は、バッファのアドレス、ファイルオフセット、転送サイズにアライメントの要件を課す。3つすべてがデバイスの論理ブロックサイズ（通常、SATAでは512バイト、4Knフォーマットの NVMeでは4096バイト）にアラインされていなければならない。アライメントに違反すると `EINVAL` が返る。

```bash
# デバイスの論理ブロックサイズを確認する
cat /sys/block/nvme0n1/queue/logical_block_size
# 512（4Knデバイスでは4096）

# 物理ブロックサイズ（最適なアライメント）を確認する
cat /sys/block/nvme0n1/queue/physical_block_size
# 4096
```

アプリケーションのコードでは、デバイスのファームウェア内で部分ブロックの読み込みが発生するのを避けるため、O_DIRECTのバッファを論理ブロックサイズだけでなく常に物理ブロックサイズにアラインする。

```c
// O_DIRECT用にアラインされたバッファを割り当てる
void *buf;
posix_memalign(&buf, 4096, IO_SIZE);   // 4096バイト境界にアライン、IO_SIZEは4096の倍数
```

### fioによるO_DIRECTのベンチマーク {#benchmarking-o_direct-with-fio}

```bash
# ランダム4KリードについてバッファードとO_DIRECTを比較する
# バッファード（1回目の実行でキャッシュが埋まり、2回目はキャッシュから読む）:
fio --name=buffered-randread \
    --filename=/var/lib/postgresql/data/base/16384/1259 \
    --rw=randread --bs=4k --iodepth=8 --numjobs=4 \
    --runtime=30 --time_based --direct=0 \
    --lat_percentiles=1

# O_DIRECT（常にデバイスへアクセスする）:
fio --name=odirect-randread \
    --filename=/var/lib/postgresql/data/base/16384/1259 \
    --rw=randread --bs=4k --iodepth=8 --numjobs=4 \
    --runtime=30 --time_based --direct=1 \
    --lat_percentiles=1
```

### O_DIRECTの欠点 {#downsides-of-o_direct}

- **readaheadがない**: `O_DIRECT` はページキャッシュを完全にバイパスするため、カーネルは隣接ページを先読みできない。同じプロセス内でシーケンシャルアクセスとランダムアクセスを混在させるアプリケーションは、シーケンシャルな部分でreadaheadの恩恵を失う可能性がある。
- **ページキャッシュを共有できない**: 同じファイルを複数のプロセスが読む場合（例: 複数のPostgreSQLバックエンドが同じヒープページを読む）、バッファードI/Oならキャッシュされたページを共有できる。`O_DIRECT` では、データベースが独自の共有バッファを管理しない限り、各プロセスが個別にデバイスへアクセスすることになる。
- **アライメントの複雑さ**: アプリケーションのコードがバッファのアライメントを明示的に管理しなければならない。

独自のバッファ管理を持つ1つのプロセスだけがアクセスするファイル（データベースのデータファイル）では、利点がコストを上回る。プロセス間で共有されるファイルや、キャッシュなしでシーケンシャルにアクセスされるファイル（ログファイル、一時ファイル）では、通常はバッファードI/Oの方が良い。

## NVMe固有のチューニング {#nvme-specific-tuning}

NVMeデバイスはSATAとは質的に異なり、その違いはカーネルがデバイスとどうやり取りすべきかに影響する。

### NVMe完了キューのIRQアフィニティ {#irq-affinity-for-nvme-completion-queues}

各NVMeハードウェアキューには専用のMSI-X割り込みがある。リクエストが完了すると、NVMeコントローラがこの割り込みを発生させ、カーネルのIRQハンドラが完了を処理する。このIRQハンドラがリクエストを発行したCPUとは別のCPUで実行されると、完了データ（結果バッファを含む）がCPUキャッシュ間を移動しなければならない。

```bash
# NVMeのIRQ割り当てを一覧表示する
grep nvme /proc/interrupts

# 各NVMeキューのIRQを特定のCPUに固定する
# IRQ 35 → CPU 0、IRQ 36 → CPU 1、など
echo 1 > /proc/irq/35/smp_affinity_list
echo 2 > /proc/irq/36/smp_affinity_list

# あるいはポリシーのヒント付きでirqbalanceを使う
# /etc/sysconfig/irqbalance:
# IRQBALANCE_ARGS="--policy=exact"
```

大規模なNVMe環境では、CPUへの固定によって完了時のNUMAをまたぐキャッシュミスが減り、p99レイテンシが測定可能なほど改善することがある。

### io_poll: IRQなしのビジーポーリング {#io_poll-busy-polling-without-irqs}

超低レイテンシのアプリケーションでは、`io_uring` を `IORING_SETUP_IOPOLL`（io_pollモード）とともに使うことで、IRQの経路を完全に回避できる。NVMeコントローラが割り込みを発生させるのを待つ代わりに、カーネルがNVMe完了キューのレジスタをビジーポーリングする。

これはCPUサイクルとレイテンシを引き換えにする。

```bash
# io_uringのHIPRI（ポーリング）とデフォルトのIRQ経路をベンチマークする
fio --name=uring-poll \
    --ioengine=io_uring \
    --hipri=1 \
    --filename=/dev/nvme0n1 \
    --rw=randread \
    --bs=4k \
    --iodepth=1 \
    --numjobs=1 \
    --runtime=30 \
    --time_based \
    --direct=1 \
    --lat_percentiles=1

# ポーリングなしと比較する
fio --name=uring-irq \
    --ioengine=io_uring \
    --hipri=0 \
    --filename=/dev/nvme0n1 \
    --rw=randread \
    --bs=4k \
    --iodepth=1 \
    --numjobs=1 \
    --runtime=30 \
    --time_based \
    --direct=1 \
    --lat_percentiles=1
```

io_pollが割に合うのは次の場合だけである。

- 専用のCPUコアを使える（ポーリング中はコアを100%占有する）。
- レイテンシのSLAが100µs未満である。
- キュー深度が低い（1–4）。キュー深度が高い場合は、IRQ経路がより多くのリクエストでオーバーヘッドを償却する。

### NVMeでのnomerges {#nomerges-for-nvme}

ブロック層のマージロジックは、隣接するリクエストを1つの大きなI/Oにまとめ、IOPSを減らす。これは低速なHDDでは有益だが、数十万の個別IOPSをネイティブに処理できるNVMeでは逆効果である。

```bash
# 現在のマージの挙動を確認する
cat /sys/block/nvme0n1/queue/nomerges
# 0 = マージ有効（デフォルト）
# 1 = 連続したマージのみ
# 2 = マージを一切行わない

# NVMeのOLTPワークロードではマージを無効にする
echo 2 > /sys/block/nvme0n1/queue/nomerges
```

マージを無効にすると、ブロック層がマージツリーの探索に費やす時間がなくなり、レイテンシが減る。この効果は、マージツリーの検索がボトルネックになりうる高IOPS（100K IOPS超）で測定可能になる。

### nvme-tcpのレイテンシ {#nvme-tcp-latency}

TCPトランスポートを使うNVMe-oF（NVMe over Fabrics）では、ネットワーク層で追加のチューニングが必要である。

```bash
# NVMe-TCP接続のソケットでNagleアルゴリズムを無効にする
# （直接ではなくnvme-cli経由で行う）
nvme connect -t tcp -a 192.168.1.100 -s 4420 -n nqn.example:storage1 \
    --reconnect-delay=1 --ctrl-loss-tmo=600

# NVMe-TCP向けのTCP最適化
sysctl -w net.ipv4.tcp_nodelay=1          # Nagleを無効化（レイテンシを減らす）
sysctl -w net.core.rmem_max=134217728     # 128MBの受信バッファ
sysctl -w net.core.wmem_max=134217728     # 128MBの送信バッファ
```

## I/Oのためのファイルシステム選択 {#filesystem-choices-for-io}

ファイルシステム層はブロックデバイスの上に位置し、あらゆるI/O経路にメタデータ操作、ジャーナリング、エクステント管理のオーバーヘッドを加える。

### XFS {#xfs}

XFSは、大きなファイルを扱うストレージ負荷の高いワークロードに最適な選択である。

- エクステントベースの割り当てにより、大きなシーケンシャルファイルの断片化が減る。
- inodeごとのアロケーショングループにより、異なるファイルへの並列書き込みが競合なしに行える。
- 遅延割り当てにより、小さな書き込みがより大きなエクステントの割り当てにまとめられる。
- スケーラブルなBツリーのメタデータ構造により、ファイルシステムが埋まっていっても性能が維持される。

```bash
# データベース向けの最適な設定でフォーマットする（agcountはCPU数に合わせて調整する）
mkfs.xfs -f \
    -d agcount=16 \
    -l size=256m,lazy-count=1 \
    /dev/nvme0n1p1

# 性能向けのオプションでマウントする
mount -o noatime,nodiratime,logbufs=8,logbsize=256k /dev/nvme0n1p1 /var/lib/postgresql
```

### ext4 {#ext4}

ext4は、ツールのサポートが幅広い堅実な汎用の選択肢である。

```bash
# データベースワークロード向けにフォーマットする（大きなジャーナル、inodeの遅延初期化による遅れなし）
mkfs.ext4 -E lazy_itable_init=0,lazy_journal_init=0 \
           -J size=1024 \
           /dev/sda1

# データベースワークロード向けのマウントオプション
mount -o noatime,nodiratime,data=ordered,barrier=1 /dev/sda1 /var/lib/mysql
```

### btrfs {#btrfs}

btrfsは、ランダム書き込みのデータベースワークロードには一般に推奨されない。コピーオンライト（CoW）の設計により、ページの上書きは常に新しい場所への書き込みになり、インプレース書き込みのファイルシステムよりもはるかに多くのI/Oと断片化を生む。

```
In-place write (ext4/XFS):    single 4KB write → one 4KB I/O
CoW write (btrfs):            one 4KB write → read old block, write new block,
                               update parent tree node, possibly cascade
```

btrfsにも正当な用途はある。バックアップパイプライン向けの増分スナップショット、RAIDの統合、整合性が重要なワークロード向けのチェックサムなどである。しかし、ランダム書き込みでの生のI/O性能を求めるなら、XFSかext4を使う。

### 重要なマウントオプション {#mount-options-that-matter}

**`noatime` / `nodiratime`**: デフォルトでは、カーネルは読み込みのたびにすべてのファイルとディレクトリの `atime`（アクセス時刻）を更新する。これにより、すべての読み込みが潜在的な書き込み（inodeの更新）になる。`noatime` はこれを完全に無効にし、`nodiratime` はディレクトリについてのみ無効にする。

```bash
# /etc/fstab に追加する
/dev/nvme0n1p1 /var/lib/postgresql xfs noatime,nodiratime 0 2
```

**`nobarrier`**: 書き込みバリアは、ジャーナルトランザクションを完了とマークする前にデバイスへキャッシュのフラッシュを強制することで、ジャーナルのコミットが永続化されることを保証する。バリアを無効にする（`nobarrier`）と書き込みスループットは向上するが、書き込み途中でシステムの電源が落ちるとファイルシステムが破損するリスクがある。

```
With barrier:     write data → flush → write journal commit → flush
Without barrier:  write data + journal commit (no ordering guarantee)
```

バリアを無効にするのは、順序の永続性を保証するバッテリーバックアップ付き書き込みキャッシュ（BBU付きRAIDコントローラ、電源喪失保護付きのエンタープライズ向けNVMe）をストレージが備えている場合だけにする。コンシューマ向けSSDでは決して無効にしない。

!!! warning "データ損失のリスク"
    書き込みバリアの無効化が安全なのは、バッテリーバックアップ付き書き込みキャッシュ（BBWC）またはUPSがある場合だけである。ハードウェアレベルで書き込み順序の保証がないと、ジャーナルのコミット途中での電源障害によって、ファイルシステムが気づかないうちに破損しうる。

!!! note "ext4の構文"
    `barrier=0` はext3の構文であり、ext4では `nobarrier` を使う。

```bash
# バッテリーバックアップ付き書き込みキャッシュがある場合のみ:
mount -o noatime,nobarrier /dev/nvme0n1p1 /var/lib/postgresql
```

**WALパーティション向けの `data=writeback`**:

!!! warning "data=writebackはデータベースのデータディレクトリには安全ではない"
    `data=writeback` はメタデータのジャーナリングをデータのライトバックから切り離すため、
    クラッシュ後に古いデータが露出することがある。データベースが独自のWALを持っていても同様である。
    データベースのWALは*論理的な*順序を提供するが、新たに割り当てられたブロックにおいて、
    WALレコードより前のゴミデータをファイルシステムが見せてしまうことは防げない。

    PostgreSQLのドキュメントは、データディレクトリに対して `data=ordered`（ext4のデフォルト）を
    明示的に推奨している。`data=writeback` モードが適切なのは、一般にクラッシュ時の整合性が
    不要なスクラッチ用または一時的なパーティションだけである。

```bash
mount -o noatime,data=writeback /dev/nvme1n1p1 /var/lib/postgresql/pg_wal
```

## HDD固有のチューニング {#hdd-specific-tuning}

回転ディスクは性能特性が根本的に異なり、異なるチューニング戦略が必要になる。

### 回転メディアの検出 {#detecting-rotational-media}

```bash
# 1 = 回転型（HDD）、0 = 非回転型（SSD/NVMe）
cat /sys/block/sda/queue/rotational
# 1

# すべてのブロックデバイスを回転型かどうかとともに一覧表示する
lsblk -d -o NAME,ROTA,SCHED,RA,NR_HW_QUEUES
```

### HDD向けのスケジューラ選択 {#scheduler-selection-for-hdds}

HDDでは、ドライブがトラック間を物理的にシークしなければならないため、スケジューラの並べ替えの判断がスループットに直接影響する。混在ワークロードには `bfq` を、レイテンシに敏感な単一ワークロードのデータベースには `mq-deadline` を使う。

```bash
echo bfq > /sys/block/sda/queue/scheduler

# BFQのチューニング: シーケンシャルワークロード向けに最大バジェットを増やす
echo 8192 > /sys/block/sda/queue/iosched/max_budget

# 非シーケンシャルなリクエストが優先されるまでのタイムアウトを引き上げる
echo 100 > /sys/block/sda/queue/iosched/slice_idle_us
```

### NCQ（Native Command Queuing）の深度 {#ncq-native-command-queuing-depth}

最近のHDDはNCQに対応しており、ドライブが最大32個のコマンドを内部で並べ替えて、シークのスケジューリングを最適化できる。NCQが有効になっていることを確認し、それを活用するようにキュー深度を設定する。

```bash
# NCQ深度の対応状況を確認する
cat /sys/block/sda/device/queue_depth
# 32

# キュー深度を設定する（テスト目的でNCQを実質的に無効にするには1に減らす）
echo 32 > /sys/block/sda/device/queue_depth

# dmesgで確認する
dmesg | grep -i "ncq\|queue depth"
```

### HDD向けの大きなreadahead {#large-readahead-for-hdds}

HDDでは、シークを引き起こすシーケンシャルリードは高コスト（約5ms）である。大きなreadaheadによって、シークのコストを多数のページで償却する。

```bash
# HDD上のシーケンシャルワークロード向けの16MBのreadaheadウィンドウ
echo 16384 > /sys/block/sda/queue/read_ahead_kb
```

### HDDでランダムI/Oの混在を避ける {#avoiding-mixed-random-io-on-hdds}

ランダムなOLTPリードとシーケンシャルなバックアップライトの両方を1台のHDDで処理すると、どちらも性能が悪くなる。シークヘッドは両方のパターンを同時に最適化できないからだ。別々の物理デバイスか、少なくとも別々のディスクアームを使う。

1台のHDDでワークロードを混在させざるをえない場合は、cgroup v2の `io.weight` を `bfq` とともに使い、レイテンシに敏感なワークロードを優先する。

```bash
# データベースに高い優先度、バックアップに低い優先度
echo "8:0 100" > /sys/fs/cgroup/databases/io.bfq.weight
echo "8:0 10"  > /sys/fs/cgroup/backups/io.bfq.weight
```

### HDDのチェックリスト {#hdd-checklist}

```bash
#!/bin/bash
# hdd-check.sh: HDDが最高の性能を出せるよう設定されているかを確認する
DEVICE=${1:-sda}

echo "=== $DEVICE HDD Configuration ==="
echo "Scheduler:   $(cat /sys/block/$DEVICE/queue/scheduler)"
echo "Rotational:  $(cat /sys/block/$DEVICE/queue/rotational)"
echo "Queue depth: $(cat /sys/block/$DEVICE/device/queue_depth 2>/dev/null || echo 'N/A')"
echo "Read-ahead:  $(cat /sys/block/$DEVICE/queue/read_ahead_kb) KB"
echo "nr_requests: $(cat /sys/block/$DEVICE/queue/nr_requests)"

# 書き込みキャッシュを確認する
hdparm -W /dev/$DEVICE 2>/dev/null | grep "write-caching"
```

## ベンチマークの方法論 {#benchmarking-methodology}

誤解を招く結果を出すベンチマークは、ベンチマークがないよりも悪い。本番環境で性能を損なうチューニング変更につながるからだ。

### よくある間違い {#common-mistakes}

1. **実行時間が足りない** — ストレージデバイスには書き込みキャッシュがある。5秒のベンチマークは全体がドライブのDRAMキャッシュ内で実行され、非現実的に高いスループットを示す。SSDでは少なくとも60秒、HDDでは120秒実行する。

2. **ウォームアップをしない** — ベンチマークの最初の数秒は、デバイスがキャッシュを埋めたりカーネルがreadaheadの状態を構築したりするため、異なる挙動を示すことが多い。fioでは `--ramp_time=10` を使う。

3. **tmpfs上でテストする** — `tmpfs` はDRAMである。tmpfs上のベンチマークはストレージではなくメモリ帯域を測定する。常に、物理ストレージを背後に持つ実際のブロックデバイスかファイルシステムを使う。

4. **ブロックサイズが間違っている** — 実際のI/Oが8Kのデータベースを128Kのブロックでベンチマークしても、役に立たない結果しか得られない。ベンチマークのブロックサイズをアプリケーションの実際のI/Oサイズに合わせる。

5. **デバイスを分離していない** — ベンチマーク対象と同じデバイスにOSがインストールされていると、OSからのバックグラウンドI/Oが結果を汚す。専用のデバイスを使う。

6. **アクセスパターンを合わせていない** — ランダムリードのIOPSとシーケンシャルリードのスループットは別の数値である。ワークロードに合ったパターンでベンチマークする。

### fioのジョブファイル {#fio-job-files}

コマンドラインフラグの代わりにfioのジョブファイルを使うと、再現性が確保され、ベンチマーク設定の共有も容易になる。

**OLTPのランダム4K（データベースのランダムなページ読み込みを模擬する）**

```ini
# oltp-randread.fio
[global]
ioengine=libaio
direct=1
runtime=120
time_based=1
ramp_time=10
numjobs=4
group_reporting=1
lat_percentiles=1
clat_percentiles=1

[randread-4k]
name=oltp-randread-4k
filename=/dev/nvme0n1
rw=randread
bs=4k
iodepth=32
```

```bash
fio oltp-randread.fio
```

**シーケンシャルなストリーミング1M（バックアップ、分析スキャン、ログの取り込みを模擬する）**

```ini
# sequential-1m.fio
[global]
ioengine=libaio
direct=1
runtime=120
time_based=1
ramp_time=10
group_reporting=1

[seqread-1m]
name=sequential-read-1m
filename=/dev/nvme0n1
rw=read
bs=1m
iodepth=4
numjobs=4

[seqwrite-1m]
name=sequential-write-1m
filename=/dev/nvme0n1
rw=write
bs=1m
iodepth=4
numjobs=4
```

**リード／ライトの混在（継続的な書き込みのあるOLTPを模擬する）**

```ini
# mixed-rw.fio
[global]
ioengine=libaio
direct=1
runtime=120
time_based=1
ramp_time=10
group_reporting=1
lat_percentiles=1
clat_percentiles=1

[mixed-oltp]
name=mixed-oltp-70r-30w
filename=/dev/nvme0n1
rw=randrw
rwmixread=70
bs=4k
iodepth=16
numjobs=4
```

### fioの出力の読み方 {#interpreting-fio-output}

```
Jobs: 4 (f=4): [r(4)][100.0%][r=1824MiB/s,w=0KiB/s][r=466k,w=0 IOPS][eta 00m:00s]
mixed-oltp-70r-30w: (groupid=0, jobs=4): err= 0: pid=12345: ...
  read: IOPS=326k, BW=1274MiB/s (1336MB/s)(149GiB/120001msec)
    clat (usec): min=48, max=12806, avg=94.51, stdev=71.23
     lat (usec): min=48, max=12807, avg=94.63, stdev=71.24
    clat percentiles (usec):
     |  1.00th=[   59],  5.00th=[   64], 10.00th=[   68], 20.00th=[   73],
     | 50.00th=[   81], 75.00th=[   94], 90.00th=[  112], 95.00th=[  135],
     | 99.00th=[  247], 99.50th=[  363], 99.90th=[  742], 99.95th=[ 1045],
     | 99.99th=[ 3195]
```

記録すべき主な指標:

- **IOPS**: 1秒あたりの操作数で表した総スループット。
- **BW**: MB/s単位の帯域（シーケンシャルワークロードで重要）。
- **clat p50**: 完了レイテンシの中央値。大半のリクエストが経験する値。
- **clat p99**: 99パーセンタイル。対話的なクエリに必要なレイテンシの予算。
- **clat p999**: 99.9パーセンタイル。接続タイムアウトを引き起こすテール。

`group_reporting=1` はすべてのジョブの結果を1つのサマリーに集約するため、実行同士の比較が容易になる。これがないと、ジョブごとに1ブロックの出力が得られる。

### キャッシュのウォームアップ {#warming-the-cache}

キャッシュをウォームアップするフェーズを含むベンチマーク（数時間稼働しているサーバーを模擬する）の場合:

```bash
# 測定前にページキャッシュまたはデバイスのキャッシュをウォームアップする
fio --name=warm \
    --filename=/dev/nvme0n1 \
    --rw=read \
    --bs=1m \
    --iodepth=8 \
    --numjobs=4 \
    --size=100% \
    --direct=0 \
    --loops=1

# その後、実際のベンチマークを実行する
fio actual-workload.fio
```

`O_DIRECT` のベンチマークではページキャッシュはバイパスされるが、デバイス内部のDRAMキャッシュはやはりウォームアップが必要である。測定前にデバイスのキャッシュを埋めるため、`--ramp_time=30` を使う。

## 実践的な設定例 {#practical-configuration-examples}

### NVMe専用のデータベースサーバー {#dedicated-nvme-database-server}

NVMe上でPostgreSQLを動かすサーバー向けの、完全なI/O設定:

```bash
#!/bin/bash
# nvme-database-tune.sh

NVME_DEV=nvme0n1

# スケジューラ: none（キューイングはNVMeに任せる）
echo none > /sys/block/$NVME_DEV/queue/scheduler

# マージなし（NVMeは細粒度のI/Oをネイティブに処理できる）
echo 2 > /sys/block/$NVME_DEV/queue/nomerges

# 最小限のreadahead（データベースが独自に先読みを管理する）
echo 16 > /sys/block/$NVME_DEV/queue/read_ahead_kb

# 控えめなdirty比率（書き込み停止を避ける）
sysctl -w vm.dirty_background_bytes=134217728  # 128MB
sysctl -w vm.dirty_bytes=536870912             # 512MB
sysctl -w vm.dirty_expire_centisecs=1000
sysctl -w vm.dirty_writeback_centisecs=200

echo "NVMe database tuning applied to $NVME_DEV"
```

```bash
# udev + sysctl.d で永続化する
cat > /etc/udev/rules.d/60-nvme-database.rules << 'EOF'
ACTION=="add|change", KERNEL=="nvme[0-9]*n[0-9]*", \
  ATTR{queue/scheduler}="none", \
  ATTR{queue/nomerges}="2", \
  ATTR{queue/read_ahead_kb}="16"
EOF

cat > /etc/sysctl.d/30-io-database.conf << 'EOF'
vm.dirty_background_bytes = 134217728
vm.dirty_bytes = 536870912
vm.dirty_expire_centisecs = 1000
vm.dirty_writeback_centisecs = 200
EOF
```

### HDDのデータウェアハウスサーバー {#hdd-data-warehouse-server}

HDD上のシーケンシャル中心の分析向け:

```bash
#!/bin/bash
# hdd-dw-tune.sh

HDD_DEV=sda

# 回転メディア向けのBFQスケジューラ
echo bfq > /sys/block/$HDD_DEV/queue/scheduler

# シーケンシャルスキャン向けの大きなreadahead
echo 8192 > /sys/block/$HDD_DEV/queue/read_ahead_kb

# NCQの深度を最大にする
echo 32 > /sys/block/$HDD_DEV/device/queue_depth

# 高めのdirtyしきい値: フラッシュ前に書き込みをまとめさせる
sysctl -w vm.dirty_background_ratio=15
sysctl -w vm.dirty_ratio=40

echo "HDD data warehouse tuning applied to $HDD_DEV"
```

## I/Oの健全性の監視 {#monitoring-io-health}

```bash
#!/bin/bash
# io-health.sh: I/Oサブシステムの簡易ヘルスチェック

echo "=== I/O Scheduler Configuration ==="
for dev in $(lsblk -d -n -o NAME); do
    sched=$(cat /sys/block/$dev/queue/scheduler 2>/dev/null | tr -d '[]' | awk '{print $1}')
    rota=$(cat /sys/block/$dev/queue/rotational 2>/dev/null)
    ra=$(cat /sys/block/$dev/queue/read_ahead_kb 2>/dev/null)
    printf "  %-12s  scheduler=%-12s  rotational=%s  read_ahead=%s KB\n" \
           "$dev" "$sched" "$rota" "$ra"
done

echo ""
echo "=== Dirty Page Status ==="
grep -E "^Dirty:|^Writeback:" /proc/meminfo

echo ""
echo "=== I/O Wait ==="
iostat -x 1 3 | awk '/^Device/{header=1} header && /[0-9]/{print $1, "await="$10, "util="$NF}'

echo ""
echo "=== Block Layer Errors ==="
dmesg --since "1 hour ago" 2>/dev/null | grep -iE "blk_|io error|medium error|ata.*error" | tail -10
```

## 参考文献 {#further-reading}

- [docs.kernel.org: block layer documentation](https://docs.kernel.org/block/index.html) — ブロックサブシステム、スケジューラ、blk-mqに関するカーネルドキュメント
- [docs.kernel.org: admin-guide sysctl vm](https://docs.kernel.org/admin-guide/sysctl/vm.html) — dirtyページのライトバックを含む `vm.*` のチューニング可能なパラメータの完全なリファレンス
- [LWN: The multiqueue block layer](https://lwn.net/Articles/552904/) — 最新のすべてのI/Oスケジューラの基盤であるblk-mqの設計
- [LWN: io_uring](https://lwn.net/Articles/776703/) — io_uringの設計とlibaioとの比較
- [LWN: BFQ, the budget fair queueing scheduler](https://lwn.net/Articles/601799/) — BFQの仕組みと使いどころ
- [fio documentation](https://fio.readthedocs.io/) — すべてのエンジンオプションと出力フィールドを含むfioの完全なリファレンス
- [block/blk-mq.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/blk-mq.c) — マルチキューブロック層の実装
- [block/mq-deadline.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/mq-deadline.c) — mq-deadlineスケジューラのソース
- [block/kyber-iosched.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/kyber-iosched.c) — Kyberスケジューラのソース
- [mm/readahead.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/readahead.c) — readaheadの実装
- [readahead](readahead.md) — カーネルのreadaheadウィンドウがどのように拡大・縮小するか
- [ダイレクトI/O](direct-io.md) — O_DIRECTのアライメント規則とカーネル内の経路
- [データベース向けメモリチューニング](../mm/tuning-databases.md) — 同じワークロードについてhuge page、swappiness、NUMAを扱う姉妹ガイド
