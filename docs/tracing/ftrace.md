# ftrace: 関数トレーサー {#ftrace-function-tracer}

> カーネル組み込みの関数トレーシング基盤

## ftraceとは {#what-ftrace-is}

ftraceはカーネルの主要なトレーシングフレームワークであり、`/sys/kernel/tracing/`のtracefsを通じてアクセスする。Steven Rostedtが作成し、Linux 2.6.27で導入された[（コミット）](https://git.kernel.org/linus/16444a8a40d4c7b4f6de34af0cae1f76a4f6c901) [（LWN）](https://lwn.net/Articles/290277/)。ftraceは次のことを行う。

- カーネルのすべての関数呼び出しを記録する（functionトレーサー）
- 実行パスとコールグラフをトレースする（function_graphトレーサー）
- 低オーバーヘッドなイベント収集のためのリングバッファを提供する
- BPFの`fentry`/`fexit`（Linux 5.5）[（コミット）](https://git.kernel.org/linus/fec56f5890d93fc2ed74166c397dc186b1c25951)、kprobe、tracepointのバックエンドとなる

## tracefsインターフェース {#tracefs-interface}

tracefsはLinux 4.1で独立したファイルシステムとして導入された[（コミット）](https://git.kernel.org/linus/4282d60689d4f21b40692029080440cc58e8a17d)。

```bash
# tracefs をマウントする（まだマウントされていない場合）
mount -t tracefs tracefs /sys/kernel/tracing

# 主なファイル:
/sys/kernel/tracing/
├── current_tracer           # 有効なトレーサー（nop/function/function_graph など）
├── tracing_on               # 1=有効, 0=無効
├── trace                    # トレースバッファを読む（ブロッキング）
├── trace_pipe               # トレース出力をストリームする（消費する読み出し）
├── trace_options            # カンマ区切りのオプション
├── available_tracers        # 組み込まれているトレーサーの一覧
├── available_filter_funcs   # フィルタ可能な関数
├── set_ftrace_filter        # 関数トレースをこれらの関数に限定する
├── set_ftrace_notrace       # これらの関数を除外する
├── set_graph_function       # function_graph: これらの関数からグラフに入る
├── set_graph_notrace
├── tracing_cpumask          # トレースする CPU
├── per_cpu/                 # per-CPU リングバッファ
│   └── cpu0/
│       ├── trace
│       └── stats
├── events/                  # tracepoint イベント
│   ├── enable               # すべてのイベントを有効にする
│   └── syscalls/
│       ├── sys_enter_read/
│       │   ├── enable
│       │   ├── filter
│       │   └── format
│       └── ...
├── instances/               # 独立したトレースインスタンス（互いに干渉しない）
└── snapshot                 # リングバッファのスナップショットを凍結する
```

## functionトレーサー {#function-tracer}

```bash
# すべての関数呼び出しをトレースする
echo function > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
sleep 0.001
echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace | head -30
# bash-1234  [000] ..... 12345.678901: do_sys_open <-do_open_execat
# bash-1234  [000] ..... 12345.678902: _raw_spin_lock <-jiffies_to_timespec64

# 特定の関数に限定する
echo "tcp_*" > /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on

# モジュールでフィルタする
echo ":mod:ext4" > /sys/kernel/tracing/set_ftrace_filter

# 複数のパターン（>> で追記する）
echo "schedule" >> /sys/kernel/tracing/set_ftrace_filter

# フィルタをクリアする（すべてをトレース）
echo "" > /sys/kernel/tracing/set_ftrace_filter
```

## function graphトレーサー {#function-graph-tracer}

呼び出し階層と実行時間を表示する。Frédéric WeisbeckerによってLinux 2.6.29で追加された[（コミット）](https://git.kernel.org/linus/15e6cb3673ea6277999642802406a764b49391b0)。

```bash
echo function_graph > /sys/kernel/tracing/current_tracer
echo vfs_read > /sys/kernel/tracing/set_graph_function  # エントリポイント
echo 1 > /sys/kernel/tracing/tracing_on
cat /some/file  # トレースを発生させる
echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace
# CPU DURATION              FUNCTION CALLS
#  0  |  8.234 us  |        } /* vfs_read */
#  0  |            |        vfs_read() {
#  0  |            |          rw_verify_area() {
#  0  |  0.512 us  |            security_file_permission();
#  0  |  1.234 us  |          }
#  0  |            |          generic_file_read_iter() {
#  0  |            |            filemap_read() {
```

## ftraceが関数を計装する仕組み {#how-ftrace-instruments-functions}

`CONFIG_DYNAMIC_FTRACE=y`かつ`CONFIG_HAVE_FENTRY=y`のとき、コンパイラはトレース可能なすべての関数の先頭に`__fentry__`（または`mcount`）の呼び出しを挿入する。

```asm
/* すべてのカーネル関数は次のように始まる: */
func:
    callq   __fentry__      ; 5 バイト（E8 + 4 バイトのオフセット）
    ; ... 実際の関数コード ...
```

トレーシングが無効なとき、これらの呼び出しはブート時に`nop`命令へパッチされる — **トレースしていないときのオーバーヘッドはゼロ**である。

有効にすると:
```asm
func:
    callq   ftrace_caller   ; 呼び出しがパッチで戻される
```

`ftrace_caller`は、この関数にどのトレーサーが登録されているかを調べ、それらを呼び出す。パッチ適用は`text_poke_bp()`で行われる。これはブレークポイントを用いたパッチ手法で、全CPUを停止させずに済む。

`CONFIG_HAVE_FENTRY`のあるLinux 5.x以降、コンパイラは`mcount`ではなく`__fentry__`（関数プロローグがレジスタを退避する前に呼ばれる）を使う。これによりフックが高速になり、信頼性の高いスタック巻き戻しが可能になる。

## リングバッファ {#ring-buffer}

ftraceはロックレスなper-CPUリングバッファ（`kernel/trace/ring_buffer.c`）を使う。

```c
/* アーキテクチャ: */
struct ring_buffer {
    struct ring_buffer_per_cpu *buffers[NR_CPUS];
};

struct ring_buffer_per_cpu {
    /* 書き込み側: */
    unsigned long   tail_page;   /* 現在の書き込みページ */
    unsigned long   commit_page; /* 最後にコミットされたページ */
    /* 読み出し側: */
    unsigned long   reader_page;
    unsigned long   head_page;
    /* ... */
};
```

- 各CPUは自分専用のバッファに書き込む（ロック競合なし）
- リングとしての動作: 満杯になると古いイベントが上書きされる（`trace_options`で`overwrite`が無効にされていない限り）
- イベントにはper-CPUクロックでタイムスタンプが付けられる

```bash
# バッファサイズを確認する
cat /sys/kernel/tracing/buffer_size_kb
# 1408 (per CPU)

# 長時間の捕捉のために増やす
echo 65536 > /sys/kernel/tracing/buffer_size_kb
```

## 動的イベント: kprobe_eventsとuprobe_events {#dynamic-events-kprobe_events-and-uprobe_events}

```bash
# tcp_sendmsg に kprobe を追加: 第 1 引数（sk）を表示する
echo 'p:my_probe tcp_sendmsg sk=%di' > /sys/kernel/tracing/kprobe_events
echo 1 > /sys/kernel/tracing/events/kprobes/my_probe/enable

cat /sys/kernel/tracing/trace_pipe
# bash-1234 [000] ..... 12345.0: my_probe: (tcp_sendmsg+0x0) sk=0xffff888...

# プローブを削除する
echo '-:my_probe' >> /sys/kernel/tracing/kprobe_events

# kretprobe を追加: 戻り値を表示する
echo 'r:my_ret tcp_sendmsg retval=$retval' > /sys/kernel/tracing/kprobe_events

# uprobe を追加する
echo 'p:myapp /usr/bin/myapp:0x1234 arg1=%di' > /sys/kernel/tracing/uprobe_events
```

## イベントトレーシング: tracepoint {#event-tracing-tracepoints}

```bash
# 利用可能な tracepoint イベントを一覧表示する
ls /sys/kernel/tracing/events/

# あるサブシステムのすべてのイベントを有効にする
echo 1 > /sys/kernel/tracing/events/block/enable

# 特定のイベントを有効にする
echo 1 > /sys/kernel/tracing/events/syscalls/sys_enter_read/enable

# フィルタ: PID 1234 からのイベントのみ
echo "common_pid == 1234" > /sys/kernel/tracing/events/syscalls/sys_enter_read/filter

# イベントのフォーマットを表示する
cat /sys/kernel/tracing/events/syscalls/sys_enter_read/format
# name: sys_enter_read
# ID: 682
# format:
#     field:unsigned short common_type;    offset:0; size:2; signed:0;
#     field:unsigned char common_flags;    offset:2; size:1; signed:0;
#     field:unsigned char common_preempt_count;  offset:3; size:1; signed:0;
#     field:int common_pid;                offset:4; size:4; signed:1;
#     field:int __syscall_nr;              offset:8; size:4; signed:1;
#     field:unsigned int fd;               offset:16; size:8; signed:0;
#     field:char * buf;                    offset:24; size:8; signed:0;
#     field:size_t count;                  offset:32; size:8; signed:0;
```

## trace-cmd: ftraceのフロントエンド {#trace-cmd-the-ftrace-frontend}

```bash
# システムコールを 5 秒間記録する
trace-cmd record -e 'syscalls:sys_enter_*' sleep 5

# ext4 の function graph を記録する
trace-cmd record -p function_graph -l "ext4_*" sleep 1

# レポート
trace-cmd report | head -50

# CPU でフィルタする
trace-cmd report --cpu 0 | head -20

# ライブトレーシング
trace-cmd stream -e sched:sched_switch

# kernel-shark: trace-cmd の出力用 GUI フロントエンド
kernelshark trace.dat
```

## トレースインスタンス: 互いに干渉しないセッション {#trace-instances-non-interfering-sessions}

```bash
# プライベートなインスタンスを作成する（グローバルなトレースに影響しない）
mkdir /sys/kernel/tracing/instances/myinstance

# 各インスタンスは独自のリングバッファとトレーサーを持つ
echo function > /sys/kernel/tracing/instances/myinstance/current_tracer
echo "net_*" > /sys/kernel/tracing/instances/myinstance/set_ftrace_filter
echo 1 > /sys/kernel/tracing/instances/myinstance/tracing_on

# 複数のツールが互いに干渉せず同時にトレースできる
cat /sys/kernel/tracing/instances/myinstance/trace_pipe &
# ... その間もメインのトレースは通常どおり継続する ...

rmdir /sys/kernel/tracing/instances/myinstance  # 後片付け
```

## 参考文献 {#further-reading}

- [kprobeとtracepoint](kprobes-tracepoints.md) — 動的・静的なイベントフック
- [perfイベント](perf-events.md) — ハードウェアPMUとサンプリング
- [BPF: アーキテクチャ](../bpf/bpf-overview.md) — fentry/fexitにアタッチされるBPFプログラム
- `Documentation/trace/ftrace.rst` — カーネルドキュメント
