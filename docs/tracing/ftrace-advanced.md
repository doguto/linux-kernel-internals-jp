# ftrace応用: ヒストグラム、トリガー、関数トレース {#ftrace-advanced-histograms-triggers-and-function-tracing}

> 基本的なトレーシングの先へ: オーバーヘッドなしにカーネル内でデータを集計する

## ftraceのおさらい {#ftrace-recap}

ftraceの基本インターフェースは`/sys/kernel/tracing/`にある（古い`/sys/kernel/debug/tracing/`からもアクセスできる）。基礎的なセットアップは[ftraceの基本ページ](ftrace.md)で扱っている。このページでは応用的な機能 — カーネル内ヒストグラム、トリガー、合成イベント、`function_graph`トレーサー — を扱う。

## イベントトリガー {#event-triggers}

トリガーは、トレースバッファにデータをコピーすることなく、tracepointの発火時にアクションを実行する。

```bash
# トリガーの書式: echo "action[:count]" > events/.../trigger
# アクション: traceon, traceoff, snapshot, stacktrace, hist:...

# 特定のプロセスが write() を行ったときにトレーシングを有効にする:
echo 'traceon if comm == "myapp"' > \
    events/syscalls/sys_enter_write/trigger

# 1MB を超える kmalloc のたびにスタックトレースを取る:
echo 'stacktrace if bytes_req > 1048576' > \
    events/kmem/kmalloc/trigger

# トリガーを無効にする:
echo '!stacktrace if bytes_req > 1048576' > \
    events/kmem/kmalloc/trigger
```

## ヒストグラムトリガー（hist:） {#histogram-triggers-hist}

ヒストグラムはイベントデータを**カーネル内で**集計し、個々のイベントを読み出す場合に比べてオーバーヘッドを削減する。

### キーごとのカウント {#count-by-key}

```bash
# PID ごとにシステムコールを数える
echo 'hist:keys=pid' > events/raw_syscalls/sys_enter/trigger
sleep 5
cat events/raw_syscalls/sys_enter/hist
# { pid:   1234 } hitcount:   4521
# { pid:   5678 } hitcount:    891
# Totals: Hits: 5412, Entries: 2

# システムコール番号ごとに数える
echo 'hist:keys=id' > events/raw_syscalls/sys_enter/trigger
cat events/raw_syscalls/sys_enter/hist
# { id:        0 } hitcount:  12345  # read
# { id:        1 } hitcount:   8901  # write
```

### 値の集計 {#aggregate-values}

```bash
# PID ごとに書き込まれたバイト数を合計する
echo 'hist:keys=common_pid:vals=count:sort=count.descending' > \
    events/syscalls/sys_enter_write/trigger

# レイテンシのヒストグラム: write() にどれだけ時間がかかるか？
echo 'hist:keys=common_pid:vals=lat:sort=lat' > \
    events/syscalls/sys_exit_write/trigger

# kmalloc のサイズを追跡する: call_site ごとにバイト数を合計する
echo 'hist:keys=call_site.sym:vals=bytes_req:sort=bytes_req.descending' > \
    events/kmem/kmalloc/trigger
cat events/kmem/kmalloc/hist
# { call_site: ext4_write_begin [ext4] } bytes_req:  1048576
# { call_site: tcp_sendmsg              } bytes_req:   524288
```

### 2つのイベント間のレイテンシ計測 {#two-event-latency-measurement}

`onmatch`を使って、イベントのペア間の時間を計測する。

```bash
# read() のレイテンシを計測する: sys_enter から sys_exit までの時間
echo 'hist:keys=common_pid:ts0=common_timestamp.usecs' > \
    events/syscalls/sys_enter_read/trigger

echo 'hist:keys=common_pid:
    vals=lat:
    lat=common_timestamp.usecs-$ts0:
    onmatch(syscalls.sys_enter_read).trace(read_lat,$lat)' > \
    events/syscalls/sys_exit_read/trigger

# あるいはレイテンシのヒストグラムを直接表示する:
echo 'hist:keys=lat:
    lat=common_timestamp.usecs-$ts0.usecs:
    onmatch(syscalls.sys_enter_read).save(comm)' > \
    events/syscalls/sys_exit_read/trigger
```

### 合成イベント: 2つのソースから派生するイベント {#synthetic-events-derived-events-from-two-sources}

```bash
# レイテンシのデータとともに発火する合成イベントを作成する
echo 'wakeup_latency u64 lat; pid_t pid; char comm[16]' > \
    synthetic_events

# wakeup 時に開始時刻を記録する:
echo 'hist:keys=pid:wakeup_ts=common_timestamp.usecs' > \
    events/sched/sched_waking/trigger

# スケジュールインされたとき: レイテンシを計算し、合成イベントを発火させる
echo 'hist:keys=next_pid:
    wakeup_lat=common_timestamp.usecs-$wakeup_ts:
    onmatch(sched.sched_waking).wakeup_latency($wakeup_lat,next_pid,next_comm)' > \
    events/sched/sched_switch/trigger

# 合成イベントを有効にする
echo 1 > events/synthetic/wakeup_latency/enable
cat trace
# wakeup_latency: lat=45 pid=1234 comm=myapp
```

## function_graphトレーサー {#function_graph-tracer}

`function_graph`は関数の入口と出口の両方をトレースし、所要時間付きのコールツリーを表示する。

```bash
# function_graph トレーサーを有効にする
echo function_graph > current_tracer

# コマンドを実行する
bash -c 'ls /tmp > /dev/null'

cat trace | head -30
# # tracer: function_graph
# #
# CPU  DURATION                  FUNCTION CALLS
# |    |   |                     |   |   |   |
# 0)   1.827 us   |            _raw_spin_lock_irqsave();
# 0)   5.380 us   |          } /* _raw_spin_unlock_irqrestore */
# 0) + 12.345 us  |        } /* mutex_lock */
# 0)              |        vfs_read() {
# 0)              |          rw_verify_area() {
# 0)   0.384 us   |            security_file_permission();
# 0)   2.183 us   |          } /* rw_verify_area */
# 0)              |          new_sync_read() {
# 0)   0.234 us   |            __vfs_read();
# 0) + 45.123 us  |          } /* new_sync_read */
# 0) + 52.456 us  |        } /* vfs_read */
```

`+` = 10µs超、`!` = 100µs超、`#` = 1ms超、`$` = 10ms超

### 特定の関数に絞り込む {#filter-to-specific-functions}

```bash
# VFS と ext4 の関数のみをトレースする
echo 'vfs_* ext4_*' > set_ftrace_filter

# 特定の関数とその呼び出し先をトレースする
echo vfs_read > set_graph_function

# 内部関数を除外する（ノイズが多すぎる）:
echo '__irq_*' > set_graph_notrace
```

## コールグラフ付きのfunctionトレーサー {#function-tracer-with-call-graph}

`function`トレーサーはすべての関数呼び出しを記録する（オーバーヘッドは非常に大きい）。

```bash
echo function > current_tracer

# オーバーヘッドを減らすため特定の関数に限定する:
echo vfs_read > set_ftrace_filter
echo 1 > function_profile_enabled

# しばらく待ってからプロファイルを表示する:
cat trace_stat/function0 | sort -k2 -n -r | head -20
# Function                 Hit    Time      Avg
# vfs_read              12345  123456us   10.0us
```

### 動的関数トレース {#dynamic-function-tracing}

```bash
# 特定のモジュールの関数へのすべての呼び出しをトレースする:
echo ':mod:ext4' > set_ftrace_filter

# パターンに一致する関数をトレースする:
echo '*alloc*page*' > set_ftrace_filter

# 利用可能な関数を表示する:
cat available_filter_functions | grep 'ext4_'
```

## 内部の仕組み: ftraceが関数をパッチする方法 {#under-the-hood-how-ftrace-patches-functions}

関数トレースを実行時にオン・オフでき、オフのときのオーバーヘッドが*ゼロ*である理由は、ftraceがカーネル自身の機械語を書き換えるからである。

カーネルが関数トレース（`CONFIG_FUNCTION_TRACER`）付きでビルドされると、コンパイラはほぼすべての関数の入口に特別なスタブ — `__fentry__`（または古い`mcount`） — の呼び出しを挿入する。これらの呼び出しを残したままでは壊滅的に高コストになるため、ブート時に**動的ftrace**がそのすべての位置を記録し、すべてを`NOP`にパッチする。したがって関数トレース可能なカーネルは、実際にトレーシングを要求するまでフルスピードで動作する。

フィルタを有効にすると、ftraceは選択された各関数の`NOP`を呼び出しへとパッチし直す — 呼び出し先は、レジスタを退避して登録済みのコールバックを呼び出す小さな**トランポリン**である。トレーシングをオフにすると、再び`NOP`にパッチし戻す。実行中のカーネルテキストに対するこのライブな書き換えはSMPセーフに行われ（歴史的にはブレークポイントを用いた3段階の更新による）、他のCPUが書きかけの命令を実行することは決してない。

登録側は`struct ftrace_ops`（`include/linux/ftrace.h`）である。これはコールバックと、対象とする関数を指定するフィルタハッシュから成る。重要なのは、多くのサブシステムが*同じ*基盤の上に独自の`ftrace_ops`を登録していることである — functionトレーサーと`function_graph`トレーサー、関数の入口に置かれたkprobe、[ライブパッチ](../livepatch/klp.md)、そしてBPFの`fentry`/`fexit`プログラムはすべて、単一の`__fentry__`フックに相乗りしている。この共有機構があるからこそ、「カーネル関数の入口に何かをアタッチする」ことは、十数個のばらばらなハックではなく、安価で十分に踏み固められた1本の経路になっている。

## スタックトレーサー: スタックを深く使う原因を見つける {#stack-tracer-find-stack-depth-offenders}

```bash
# スタックトレーサーを有効にする（モジュールとしてビルド、または CONFIG_STACK_TRACER）
echo 1 > /proc/sys/kernel/stack_tracer_enabled

# 観測された最大のスタック深さを見る
cat stack_trace
# Depth    Size   Location    (42 entries)
# -----    ----   --------
#     0    6560   interrupt_entry+0x1a/0x20
#     1    6536   do_IRQ+0x41/0xd0
#     ...

# スタック深さでソートする
cat stack_max_size
# 6560
```

## trace-cmd: スクリプト化されたftrace {#trace-cmd-scripted-ftrace}

`trace-cmd`は、ftraceの使用を簡単にするユーザー空間のラッパーである。

```bash
# write() のレイテンシを 5 秒間記録する
trace-cmd record -e syscalls:sys_enter_write \
                 -e syscalls:sys_exit_write \
                 -p function_graph \
                 -g vfs_write \
                 sleep 5

# レポートを表示する
trace-cmd report | head -50

# KernelShark（GUI）で可視化する
kernelshark trace.dat
```

## ftraceと組み合わせたperf {#perf-with-ftrace}

```bash
# perf ftrace: 内部で ftrace を使う
perf ftrace -G vfs_read,vfs_write ls /tmp

# 関数レイテンシのプロファイリング
perf ftrace latency -T vfs_read -- dd if=/dev/zero of=/dev/null bs=4k count=10000
```

## 参考文献 {#further-reading}

- [ftrace](ftrace.md) — ftraceの基本とセットアップ
- [kprobeとtracepoint](kprobes-tracepoints.md) — カーネルコードへのtracepointの追加
- [perfイベント](perf-events.md) — perfベースのプロファイリング
- [perfプロファイリング応用](perf-profiling.md) — フレームグラフとオフCPU分析
- `Documentation/trace/histogram.rst` — ヒストグラムトリガーのドキュメント
- `Documentation/trace/ftrace.rst` — ftraceの完全なドキュメント
