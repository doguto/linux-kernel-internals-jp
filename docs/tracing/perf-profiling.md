# perfプロファイリング応用 {#perf-profiling-advanced}

> フレームグラフ、オフCPU分析、メモリプロファイリング、スケジューラのトレース

## perf recordの基本（おさらい） {#perf-record-basics-recap}

```bash
# CPU プロファイリング: 99Hz で 30 秒間サンプリングする
perf record -F 99 -a -g -- sleep 30

# 特定のコマンドをプロファイルする
perf record -F 999 -g ./my_program

# レポートを読む
perf report --stdio | head -50
```

基礎については[perfイベント](perf-events.md)を参照。このページでは応用的な分析を扱う。

## フレームグラフ: CPU時間の可視化 {#flame-graphs-visualizing-cpu-time}

フレームグラフは、CPU時間がどこで費やされているかをコールスタックの階層として示す。2011年頃にOracleでBrendan Greggが考案したもので[（brendangregg.com）](http://www.brendangregg.com/flamegraphs.html)、x軸は時間（時系列順ではなくアルファベット順にソートされる）であり、幅が費やされた時間を表す。

### フレームグラフの生成 {#generating-flame-graphs}

```bash
# 1. コールグラフ付きで記録する
perf record -F 99 -a -g -- sleep 30
# あるいは特定のプロセスについて:
perf record -F 99 -g -p <pid> -- sleep 30

# 2. フレームグラフ形式に変換する（Brendan Gregg のツール）
# インストール: git clone https://github.com/brendangregg/FlameGraph
perf script | \
    /path/to/FlameGraph/stackcollapse-perf.pl | \
    /path/to/FlameGraph/flamegraph.pl > cpu-flamegraph.svg

# ブラウザで開く
open cpu-flamegraph.svg  # または firefox、chromium
```

### 差分フレームグラフ {#differential-flame-graphs}

変更の前後を比較する。

```bash
# 変更前
perf record -F 99 -g -o before.data -- ./workload
perf script -i before.data | stackcollapse-perf.pl > before.folded

# 変更後
perf record -F 99 -g -o after.data -- ./workload
perf script -i after.data | stackcollapse-perf.pl > after.folded

# 差分フレームグラフを生成する
difffolded.pl before.folded after.folded | flamegraph.pl > diff.svg
# 赤 = リグレッション（時間が増えた）、青 = 改善（時間が減った）
```

### フレームグラフ内のカーネルシンボル {#kernel-symbols-in-flame-graphs}

```bash
# カーネルシンボルが読めるようにする:
echo 0 > /proc/sys/kernel/kptr_restrict
echo -1 > /proc/sys/kernel/perf_event_paranoid

# カーネルモジュールの場合: デバッグシンボルをロードする
modprobe --force-modversion mymodule
```

## オフCPU分析: スレッドがどこでブロックしているか {#off-cpu-analysis-where-threads-are-blocked}

**オンCPU**プロファイリングはCPU時間がどこで費やされているかを示す。**オフCPU**分析は、スレッドがどこで*スリープ*しているか — I/O、ロック、ページフォールトなどを待っているか — を示す。

### perf schedによるオフCPU分析 {#off-cpu-with-perf-sched}

```bash
# スケジューライベントを記録する
perf sched record -- sleep 10

# オフ CPU 時間（スケジュールされるのを待っている時間）を表示する
perf sched latency | head -30
# -------------------------------------------------
# Task                  |   Runtime ms  | Switches | Average delay ms | Maximum delay ms |
# -------------------------------------------------
# myapp:1234            |     1234.567  |     8901 |          0.138  |          12.345  |

# スケジューラレイテンシのフレームグラフ
perf sched script | \
    stackcollapse-perf-sched.pl | \
    flamegraph.pl --color=io > offcpu-flamegraph.svg
```

### BPF（bpftrace）によるオフCPU分析 {#off-cpu-with-bpf-bpftrace}

```bash
# do_nanosleep でスリープしている時間をトレースする:
bpftrace -e '
kprobe:do_nanosleep { @start[tid] = nsecs; }
kretprobe:do_nanosleep /@start[tid]/
{
    @sleep_ms = hist((nsecs - @start[tid]) / 1000000);
    delete(@start[tid]);
}'

# オフ CPU プロファイリング: ブロックした地点でスタックをサンプリングする
bpftrace -e '
tracepoint:sched:sched_switch
/args->prev_state != 0 && args->prev_pid > 0/
{
    @stacks[ustack, kstack, args->prev_comm] = sum(1);
}'
```

### BCCのoffcputimeによるオフCPU分析 {#off-cpu-with-bcc-offcputime}

```bash
# BCC ツールをインストールする
apt install bpfcc-tools python3-bpfcc

# 30 秒間オフ CPU 時間をプロファイルする
offcputime-bpfcc -f 30 | flamegraph.pl --color=io --title="Off-CPU Time" > offcpu.svg
offcputime-bpfcc -f -p <pid> 30 > off-cpu.folded
```

## メモリプロファイリング {#memory-profiling}

### perf memでメモリ割り当てを見つける {#finding-memory-allocations-with-perf-mem}

```bash
# メモリアクセスイベントを記録する（ハードウェア PMU のサポートが必要）
perf mem record -a -- sleep 5
perf mem report | head -30

# レイテンシ付きのロードとストア:
perf c2c record -g -a -- sleep 5
perf c2c report   # キャッシュ間転送の分析
```

### perfによるmallocのサンプリング {#perf-malloc-sampling}

```bash
# uprobe を使って malloc 呼び出しをプロファイルする
perf probe -x /lib/x86_64-linux-gnu/libc.so.6 malloc
perf record -e probe_libc:malloc -a -g -- sleep 5
perf report

# あるいは BCC で:
memleak-bpfcc -p <pid> 10
# TOP 10 stacks with outstanding allocations:
#    bytes      count     stack
#  1048576        100     malloc -> myapp_allocate_buffer -> main
```

### ページフォールトの追跡 {#tracking-page-faults}

```bash
# プロセスごとのページフォールトを数える
perf stat -e page-faults,major-faults -p <pid> sleep 5

# ページフォールトがどこで発生しているかを見つける
perf record -e page-faults -g -p <pid> -- sleep 5
perf report | head -20

# bpftrace でマイナーフォールトとメジャーフォールト:
bpftrace -e '
tracepoint:exceptions:page_fault_user
{
    /* error_code bit 0: protection violation (1) vs not-present (0) — not minor/major */
    @faults[comm, args->error_code & 4 ? "user" : "kernel"] = count();
}'
```

## スケジューラの分析 {#scheduler-analysis}

### コンテキストスイッチの可視化 {#visualizing-context-switches}

```bash
# あるプロセスのスケジューライベントを記録する
perf sched record -p <pid> -- sleep 5

# タイムラインを表示: タスクはいつ実行中で、いつスリープしていたか？
perf sched timehist | head -40
# time   cpu   task name    wait time  sch delay  run time
# ...    ...   myapp:1234   0.045ms    0.002ms    1.234ms

# スローモーションで再生する（シミュレーションモード）:
perf sched replay
```

### スケジューラのwakeupレイテンシを見つける {#finding-scheduler-wakeup-latency}

```bash
# wakeup イベントを記録する
perf record -e sched:sched_wakeup,sched:sched_switch -a -g -- sleep 10

# スクリプトで分析する
perf script | awk '
/sched_wakeup/ { wake[$3] = $1 }
/sched_switch/ { if ($6 in wake) { print $6, $1 - wake[$6], "ms wakeup latency"; delete wake[$6] } }
'
```

### runqlat: ランキューレイテンシのヒストグラム {#runqlat-run-queue-latency-histogram}

```bash
# BCC ツール: タスクがランキューで待つ時間のヒストグラム
runqlat-bpfcc 5
# Tracing run queue latency... Hit Ctrl-C to end.
# usecs               : count     distribution
#     0 -> 1          : 12345    |******************************|
#     2 -> 3          : 4567     |***********                   |
#     4 -> 7          : 1234     |***                           |
#     8 -> 15         : 89       |                              |
#    16 -> 31         : 12       |                              |
#   64 -> 127         : 2        |                              |
# 2048 -> 4095        : 1        |                              |  ← 問題あり！

# bpftrace での同等のもの:
bpftrace -e '
tracepoint:sched:sched_wakeup,
tracepoint:sched:sched_wakeup_new
{ @qtime[args->pid] = nsecs; }

tracepoint:sched:sched_switch
{
    if (@qtime[args->next_pid]) {
        @usecs = hist((nsecs - @qtime[args->next_pid]) / 1000);
        delete(@qtime[args->next_pid]);
    }
}'
```

## ハードウェアカウンタを使ったプロファイリング {#profiling-with-hardware-counters}

```bash
# 分岐予測ミス（CPU パイプラインのストール）
perf stat -e branch-misses,branches ./my_program
# branch-misses: 1,234,567  #  12.34% of all branches

# キャッシュミス
perf stat -e L1-dcache-load-misses,L1-dcache-loads,
              LLC-load-misses,LLC-loads ./my_program

# TLB ミス（高コスト: ページテーブルウォークを引き起こす）
perf stat -e dTLB-load-misses,dTLB-loads ./my_program

# 命令レベルのプロファイリング:
perf record -e cycles:pp -g ./my_program   # 精密なサンプリング
perf report --sort=sym,dso --stdio
```

### perf annotate: ホットな命令の特定 {#perf-annotate-hot-instruction-identification}

```bash
# どの命令が最もホットかを見る
perf record -F 999 -g ./my_program
perf annotate --stdio | head -60

# 出力:
#  Percent |     Source code & Disassembly of my_program for cycles
# -------------------------
#          :    for (i = 0; i < N; i++) {
#   42.34  :      400b23:  mov    (%rax,%rdx,8),%rcx   ← ホットなロード
#    0.01  :      400b27:  add    %rcx,%rbx
#    0.12  :      400b2a:  inc    %rdx
```

## Java/JVM向けのLinux perf {#linux-perf-for-javajvm}

```bash
# JIT シンボルのマッピングを有効にする（perf-map-agent または async-profiler が必要）:
java -XX:+PreserveFramePointer \
     -agentpath:/path/to/libperfmap.so \
     MyApp &

# 記録してフレームグラフを生成する
perf record -F 99 -g -p $(pgrep java) -- sleep 30
perf script | stackcollapse-perf.pl | flamegraph.pl > java-flamegraph.svg

# async-profiler（JVM にはこちらが適している）:
./profiler.sh -d 30 -f flamegraph.html $(pgrep java)
```

## perf top: ライブプロファイリング {#perf-top-live-profiling}

```bash
# 関数ごとのライブな CPU 使用状況（関数版の 'top'）
perf top

# スレッドごとの内訳
perf top -t <tid>

# カーネルシンボルを隠す（ユーザー空間のみ）
perf top -K

# アセンブリを表示する
perf top --disassembler-style=att
```

## 参考文献 {#further-reading}

- [perfイベント](perf-events.md) — perfの基礎、ハードウェアカウンタ
- [ftrace応用](ftrace-advanced.md) — カーネル内ヒストグラム、function_graph
- [kprobeとtracepoint](kprobes-tracepoints.md) — 独自のtracepointの追加
- [MMのtracepoint](../mm/mm-tracepoints.md) — メモリ固有のトレーシング
- [スケジューラのトレーシング](../sched/sched-tracing.md) — スケジューラ固有のperf
- FlameGraph: `https://github.com/brendangregg/FlameGraph`
- BCCツール: `https://github.com/iovisor/bcc`
