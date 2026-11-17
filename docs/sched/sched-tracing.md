# スケジューラのトレーシング {#scheduler-tracing}

> スケジューラの挙動をリアルタイムに観察するためのftrace、perf、BPFツール

## スケジューラのトレースイベント {#scheduler-trace-events}

カーネルはtracepointの仕組み（ftrace）を通じてスケジューライベントを出力する。これらは`include/trace/events/sched.h`で定義されており、重要なスケジューリング判断のすべてを網羅している。

主なイベント:

| イベント | 発火するタイミング |
|-------|---------------|
| `sched:sched_switch` | すべてのコンテキストスイッチ |
| `sched:sched_wakeup` | タスクがスリープから実行可能に遷移したとき |
| `sched:sched_wakeup_new` | 新しく作成されたタスクが実行可能になったとき |
| `sched:sched_waking` | タスクの起床が始まったとき（ランキューに載る前） |
| `sched:sched_migrate_task` | タスクがCPU間を移動したとき |
| `sched:sched_process_fork` | fork() |
| `sched:sched_process_exec` | exec() |
| `sched:sched_process_exit` | タスクが終了したとき |
| `sched:sched_stat_sleep` | タスクが自発的なスリープから起床したとき（CONFIG_SCHEDSTATS） |
| `sched:sched_stat_wait` | ランキューで待っていた時間（CONFIG_SCHEDSTATS） |
| `sched:sched_stat_iowait` | I/O待ちに費やした時間（CONFIG_SCHEDSTATS） |

## sched_switch: 最も重要なイベント {#sched_switch-the-most-important-event}

```c
// include/trace/events/sched.h
TRACE_EVENT(sched_switch,
    TP_PROTO(bool preempt,
             struct task_struct *prev,
             struct task_struct *next,
             unsigned int prev_state),
    // フィールド:
    // prev_comm, prev_pid, prev_prio, prev_state
    // next_comm, next_pid, next_prio
    //
    // prev_stateはTASK_REPORTフラグを使う:
    //   R = 実行中/実行可能（プリエンプトされた）
    //   S = 割り込み可能なスリープ
    //   D = 割り込み不可能なスリープ（I/O）
    //   T = 停止
    //   Z = ゾンビ
);
```

`prev_state=R`は、タスクがプリエンプトされた（非自発的なスイッチ）ことを意味する。`prev_state=S`は、タスクが自発的にCPUを手放したことを意味する。

## ftraceで記録する {#recording-with-ftrace}

```bash
# 特定のイベントを有効にして5秒間記録する
trace-cmd record -e sched:sched_switch \
                 -e sched:sched_wakeup \
                 -e sched:sched_migrate_task \
                 sleep 5

# 記録を表示する
trace-cmd report | head -50

# PIDで絞り込む
trace-cmd record -e sched:sched_switch \
                 --filter 'next_pid == 1234 || prev_pid == 1234' \
                 sleep 5
```

### ftraceのインターフェースを直接使う {#direct-ftrace-interface}

```bash
# debugfs経由でイベントを有効にする
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_wakeup/enable

# リングバッファを読む
cat /sys/kernel/debug/tracing/trace

# あるいはストリームとして読む
cat /sys/kernel/debug/tracing/trace_pipe &

# comm（タスク名）で絞り込む
echo 'next_comm == "myapp"' > \
    /sys/kernel/debug/tracing/events/sched/sched_switch/filter

# 後片付け
echo 0 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
echo > /sys/kernel/debug/tracing/trace
```

## perf sched: 高レベルのスケジューリング解析 {#perf-sched-high-level-scheduling-analysis}

`perf sched`はスケジューリングの活動を記録し、レイテンシのレポートを生成する:

```bash
# スケジューリングイベントを10秒間記録する
perf sched record -- sleep 10

# レイテンシのレポートを表示する: タスクごとのスケジューリング遅延
perf sched latency
# Task                  |   Runtime ms  | Switches | Average delay ms | Maximum delay ms
# myapp:(2)             |     4500.123  |     1042  |          0.234   |         12.456
# kworker/0:1           |      123.456  |      567  |          0.012   |          0.234

# 記録を再生して正確なスケジューリング順序を確認する
perf sched replay

# すべてのスケジューリングイベントを順に表示する
perf sched script | head -20
```

### perf sched latency {#perf-sched-latency}

あるタスクの「Maximum delay」が大きいことは、そのタスクがときどき長時間CPUを待って足止めされていたことを意味する。よくある原因:

- CPUが一時的に飽和している
- スケジューラ内のロック競合（まれ）
- NUMAマイグレーションのオーバーヘッド
- RTタスクがCPUを独占している

```bash
# 最大レイテンシが最も大きいタスクを表示する
perf sched latency --sort max
```

## スケジューリングカウンタのためのperf stat {#perf-stat-for-scheduling-counters}

```bash
# コンテキストスイッチとマイグレーションを数える
perf stat -e context-switches,cpu-migrations ./workload

# システム全体のスケジューライベントを数える
perf stat -e sched:sched_switch,sched:sched_wakeup,sched:sched_migrate_task \
          -a sleep 10

# 自発的と非自発的なスイッチを比較する（csの内訳が必要）
perf stat -e sched:sched_switch -a sleep 5
```

## bpftraceによるBPFトレーシング {#bpf-tracing-with-bpftrace}

```bash
# コンテキストスイッチをトレースする: 誰から誰へ切り替わったかを表示する
bpftrace -e '
tracepoint:sched:sched_switch {
    printf("%s -> %s\n", args->prev_comm, args->next_comm);
}'

# スケジューラの待ち時間（ランキューにいた時間）が最も長いタスクを表示する
bpftrace -e '
tracepoint:sched:sched_wakeup { @start[args->pid] = nsecs; }
tracepoint:sched:sched_switch {
    if (@start[args->next_pid]) {
        @wait_ns[args->next_comm] = hist(nsecs - @start[args->next_pid]);
        delete(@start[args->next_pid]);
    }
}'

# タスクごとのマイグレーション回数を数える
bpftrace -e '
tracepoint:sched:sched_migrate_task {
    @[args->comm] = count();
}'
```

## 起床レイテンシ: runqlat（BCC） {#wakeup-latency-runqlat-bcc}

`runqlat`（BCCツールキットに含まれる）は、タスクが実行されるまでにランキューでどれだけ待つかを測定する:

```bash
# システム全体のランキューレイテンシのヒストグラムを表示する
runqlat

# PIDごとのヒストグラム
runqlat -p $PID

# 出力例:
# usecs               : count     distribution
# 0 -> 1              : 12345     |********************|
# 2 -> 3              : 4567      |*******             |
# 4 -> 7              : 234       |                    |
# 8 -> 15             : 89        |                    |
# 16 -> 31            : 45        |                    |
# 32 -> 63            : 12        |                    |  ← ここでレイテンシのスパイク
```

`runqlen`はランキュー長の分布を表示する:

```bash
runqlen  # ランキューに通常何個のタスクがいるかを表示する
```

## ftraceのfunction graphによる起床のトレース {#wakeup-tracing-with-ftrace-function-graph}

カーネル関数を含む起床パス全体をトレースするには:

```bash
# 特定のPIDの起床パスをトレースする
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo $PID > /sys/kernel/debug/tracing/set_ftrace_pid
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... ワークロードを実行する ...
cat /sys/kernel/debug/tracing/trace > /tmp/wakeup_trace.txt
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo nop > /sys/kernel/debug/tracing/current_tracer
```

あるいは組み込みのwakeupトレーサを使う:

```bash
# 起床から最初の実行までのレイテンシをトレースする（最悪ケース）
echo wakeup > /sys/kernel/debug/tracing/current_tracer
# 最大の起床レイテンシをマイクロ秒単位で求める
cat /sys/kernel/debug/tracing/tracing_max_latency
```

## sched_switchの出力を読む {#reading-sched_switch-output}

典型的な`trace-cmd report`の行:

```
myapp-1234 [002] 12345.678901: sched_switch: prev_comm=myapp prev_pid=1234
    prev_prio=120 prev_state=S ==> next_comm=kworker/2:1 next_pid=5678 next_prio=120
```

解釈:

- CPU 2上の`myapp`（pid 1234）がスリープに切り替わった（`prev_state=S`）
- `kworker/2:1`（pid 5678）が今CPU 2上で実行されている
- どちらも優先度120（nice 0）である

`prev_state=R`であれば、myappはプリエンプトされたことになる。実行を続けたかったが、より優先度の高い何かが取って代わったのである。

## 参考文献 {#further-reading}

- [スケジューラ統計](schedstat.md) — /procにあるタスクごとの累積統計
- [スケジューラのチューニング](sched-tuning.md) — 観察している挙動に影響するつまみ
- [コンテキストスイッチ](context-switch.md) — sched_switchイベントごとにカーネルが何をするか
- [起床](wakeup.md) — ハードウェア割り込みからタスクの実行までの起床パス全体
