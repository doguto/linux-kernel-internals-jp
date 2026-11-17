# カーネルのトレーシングと可観測性 {#kernel-tracing-and-observability}

> カーネルが実際に何をしているかを見る方法 — 本番環境でライブに、デバッガも再起動もなしに

## はじめに {#getting-started}

見えないものは直せない。プロセスが停止する、テールレイテンシが跳ね上がる、システムコールがときどきハングする — そしてそのどれもユーザー空間からは見えない。カーネルのトレーシングとは、カーネル内部の挙動を観測可能にするツール群である。どの関数が実行されるか、どのイベントが発火するか、それぞれにどれだけ時間がかかるか、そしてその下でハードウェアが何をしているかを、稼働中のシステム上で、たいていは本番環境で動かしっぱなしにできるほど低いオーバーヘッドで観測できる。

このセクションのすべては、3つのイベント源と、それらを収集するいくつかのフロントエンドの上に成り立っている。

```
                    観測したいもの
                           │
          ┌────────────────┼────────────────────┐
          │                │                    │
    関数呼び出し     カーネルイベント       ハードウェア
      kprobe/BPF      tracepoints         perf PMU
      ftrace           TRACE_EVENT         cycles/cache
          │                │                    │
          └────────────────┼────────────────────┘
                           │
                  収集フロントエンド
                   ┌───────┴──────────┐
                   │                  │
                ftrace              perf
                trace-cmd         bpftrace
                perf-tools           BCC
```

### イベントはどこから来るか {#where-the-events-come-from}

- **静的tracepoint**（`TRACE_EVENT`）は、カーネル開発者が意味のある箇所 — リクエストのキューイング、タスクのスケジューリング、ページフォールトの発生など — に配置した計装ポイントである。安定しており、自己記述的で、無効時のコストはほぼゼロであるため、本番環境のトレーシングで推奨されるイベント源である。
- **動的プローブ** — **kprobe**（およびユーザー空間ではuprobe） — は*ほぼ任意の*命令アドレスにアタッチでき、カーネルの作者が想定していなかった箇所まで見えるようにする。その代わり、特定のカーネルバージョンに依存する。
- **ハードウェアカウンタ** — CPUの性能監視ユニット（PMU）を**perf**経由で読み出すもので、サイクル、キャッシュミス、分岐予測ミスを数え、サンプリングプロファイラを駆動する。

### フロントエンド {#the-frontends}

- **ftrace** — カーネル組み込みのトレーサー。`/sys/kernel/tracing`にある`tracefs`ファイルシステムを通じて操作する。関数トレース、function-graph、tracepointの捕捉を、追加のツールなしで行える。`trace-cmd`はその使いやすいフロントエンドである。
- **perf** — サンプリング、PMUカウンタ、プロファイリング。ほとんどのフレームグラフの出どころである。
- **BPF**（`bpftrace`、BCC） — 現代的なプログラマブルなフロントエンド。生のイベントをユーザー空間へ送る代わりに、BPFプログラムが各イベントで*カーネル内で*実行され、その場で集計する — ヒストグラム、プロセスごとの合計、レイテンシ分布など。これにより毎秒数百万イベントのトレースが現実的なコストで可能になる。[BPF](../bpf/README.md)を参照。

### ツールの選び方 {#choosing-a-tool}

| やりたいこと | 使うもの |
|---|---|
| どの関数が実行され、どう入れ子になっているかを見る | [ftrace](ftrace.md)のfunction-graph |
| イベントを安価に集計する（ヒストグラム、カウント） | [bpftrace / BPF](../bpf/README.md) |
| tracepointのない箇所をプローブする | [kprobe](kprobes-tracepoints.md) |
| CPU時間をプロファイルする／ホットパスを見つける | [perf](perf-profiling.md) |
| ユーザー空間の関数をトレースする | [uprobe / USDT](uprobes-usdt.md) |

### 前提知識 {#prerequisites}

シェルの扱いと、カーネルの基本概念（プロセス、システムコール、割り込み）に慣れていること。ほとんどのツールはroot（または`CAP_BPF` / `CAP_PERFMON`）を必要とする。tracepointやBPFのツールは、`CONFIG_FTRACE`、`CONFIG_BPF_SYSCALL`などを有効にしてビルドされたカーネルを前提とする — 主要なディストリビューションではすべて標準である。

### 推奨する読む順番 {#suggested-reading-order}

1. **[ftrace](ftrace.md)** — 基礎: tracefs、リングバッファ、関数トレーサー
2. **[ftrace応用](ftrace-advanced.md)** — function-graph、トリガー、カーネル内集計
3. **[kprobeとtracepoint](kprobes-tracepoints.md)** — 動的計装と静的計装、そして`TRACE_EVENT`
4. **[uprobeとUSDT](uprobes-usdt.md)** — ユーザー空間への到達
5. **[perfイベント](perf-events.md)** — PMU、サンプリング、`perf_event_open()`
6. **[perfプロファイリング](perf-profiling.md)** — フレームグラフとプロファイリングのワークフロー

### 学べること {#what-youll-learn}

| 問い | どこで答えられているか |
|---|---|
| 「この関数はなぜ遅いのか？」 | [ftraceのfunction-graph](ftrace-advanced.md)が、呼び出しごとの時間付きでコールツリーを示す |
| 「出力に溺れずにイベントを数えるには？」 | [BPF](../bpf/README.md)がカーネル内で集計する — 生のストリームではなくヒストグラムが得られる |
| 「必要な場所にtracepointがない」 | [kprobe](kprobes-tracepoints.md)はほぼどこにでもアタッチできる |
| 「CPUは実際にどこで時間を使っているのか？」 | [perf](perf-profiling.md)のサンプリング＋フレームグラフ |

## ドキュメント {#documentation}

| ページ | 内容 |
|------|----------------|
| [ftrace](ftrace.md) | 関数トレース、tracefs、リングバッファ、trace-cmd |
| [ftrace応用](ftrace-advanced.md) | function graph、トリガー、ftraceが関数をパッチする仕組み |
| [リングバッファとtracefs](ring-buffer.md) | ロックレスなper-CPUバッファとtracefsの制御インターフェース |
| [トレースイベントと動的イベント](trace-events.md) | TRACE_EVENT、フィルタ、トリガー、実行時のkprobe/uprobeイベント |
| [kprobeとtracepoint](kprobes-tracepoints.md) | kprobe/kretprobe、静的tracepoint、TRACE_EVENT |
| [uprobeとUSDT](uprobes-usdt.md) | ユーザー空間プローブと静的マーカー |
| [トレーシングのためのBPF](bpf-tracing.md) | bpftraceとBCC — イベントのカーネル内集計 |
| [perfイベント](perf-events.md) | perf_event_open、PMUカウンタ、サンプリング、フレームグラフ |
| [perfプロファイリング](perf-profiling.md) | CPUプロファイリングのワークフロー、フレームグラフのレシピ |
| [事件簿](war-stories.md) | トレーシング／可観測性に関する実際のインシデントとその教訓 |

## クイックリファレンス {#quick-reference}

```bash
# ftrace: schedule() へのすべての呼び出しをトレースする
echo function > /sys/kernel/tracing/current_tracer
echo schedule > /sys/kernel/tracing/set_ftrace_filter
echo 1 > /sys/kernel/tracing/tracing_on
sleep 1
echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace | head -20

# perf: CPU サイクルのプロファイル
perf record -g -F 99 -- sleep 10
perf report

# bpftrace: write() 呼び出しをトレースする
bpftrace -e 'tracepoint:syscalls:sys_enter_write { printf("%s %d\n", comm, args->count); }'

# trace-cmd: ftrace のフロントエンド
trace-cmd record -e syscalls:sys_enter_read sleep 1
trace-cmd report
```

## 参考文献 {#further-reading}

- [Kernel docs: tracing](https://docs.kernel.org/trace/index.html) — ftrace、kprobe、tracepointなど
- [BPF](../bpf/README.md) — プログラマブルなトレーシングフロントエンド
