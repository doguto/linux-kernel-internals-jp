# ftraceリングバッファとtracefs {#the-ftrace-ring-buffer-and-tracefs}

> カーネルが毎秒数百万のトレースイベントを、あらゆるコンテキストから、ほぼオーバーヘッドなしで記録する仕組み — そしてそれを操作するためのファイルシステム

トレーシングには過酷な制約がある。計装は計測対象そのものより*安価で*なければならず、しかも**あらゆる**コンテキスト — プロセス、softirq、ハードIRQ、さらにはNMI — で、デッドロックしうるロックを一切取らずに動作しなければならない。素朴な、mutexで保護されたグローバルバッファは、この両方の点で失格である。これを解決するデータ構造が**ftraceリングバッファ**（[`kernel/trace/ring_buffer.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/trace/ring_buffer.c)）であり、それを操作するインターフェースが**tracefs**である。

## per-CPUかつロックレス {#per-cpu-and-lockless}

2つの設計判断によって、このバッファは本番環境で有効にしたままにしておけるほど高速になっている。

- **per-CPU。** 各CPUは*自分専用の*リングバッファに書き込むため、2つのCPUが同時にトレースしても競合しない。共有ロックもなく、コア間でキャッシュラインが行き来することもない。その代償として、グローバルで時刻順のビューの再構成は、後で読み出し時にper-CPUのストリームをマージすることで行われる。
- **ロックレスな書き込み。** CPU内では、イベントの記録はロックではなく、ローカルなアトミック操作の上に構築された**reserve/commit**プロトコルで行われる。

  1. `ring_buffer_lock_reserve()`がイベント用の領域を予約し、
  2. 呼び出し元がその領域にイベントのペイロードを書き込み、
  3. `ring_buffer_unlock_commit()`がそれを確定する。

プロトコルがper-CPUかつロックレスであるため、ハードIRQ — あるいはNMIでさえ — が*同じCPU上の別の書き込みに割り込んでいる最中に*イベントを記録しても安全である。バッファはこうした入れ子の書き込みを扱えるよう明示的に設計されており、これはまさにグローバルロックでは決して実現できない性質である。

## サブバッファ {#sub-buffers}

per-CPUバッファは、ページサイズの**サブバッファ**のリングである。イベントは現在のページに詰めて格納され、ページが満杯になると書き込み側はリング内の次のページへ進む。読み出しと書き込みは*別々の*ページに対して行われるため、生産者が新しいページを埋め続けている間に、消費者は完成したページを取り出せる。ホットパスでの調整は不要である。

## 2つのモード: フライトレコーダーとロスレス {#two-modes-flight-recorder-vs-lossless}

バッファが満杯になったときに何が起きるかはモードによる。

- **上書き（デフォルト）** — バッファは循環する「フライトレコーダー」である。満杯になると、最も古いイベントが上書きされて空きが作られる。常に最も*新しい*履歴が手元にあり、トレースを仕掛けてまれなイベントの再現を待つときに望ましい動作である。
- **生産者/消費者** — 上書きする代わりに書き込みを停止する（イベントは破棄され、その数が記録される）ため、すでに記録されたものは失われない。イベントを取りこぼしてはならず、バッファを十分速く吸い出す消費者がいる場合にこちらを選ぶ。

バッファサイズはper-CPUで、`buffer_size_kb`で設定する。メモリと履歴の長さのトレードオフである。

## 読み出し: `trace`と`trace_pipe` {#reading-trace-vs-trace_pipe}

tracefsはバッファを2通りの方法で公開しており、その違いが重要である。

- **`trace`** — *消費しないスナップショット*。読み出してもイベントは削除されないため、何度でも`cat`して出力の差分を取れる。「仕掛ける、再現する、調べる」の流れに向いている。
- **`trace_pipe`** — *消費する、ブロッキングなストリーム*。イベントは到着するたびに配信され、一度読まれると消える。これはライブでtailするためのインターフェースであり、ツールにパイプで渡すのに使う。
- **`per_cpu/cpuN/trace`** — グローバルなマージを行う前の、単一CPUの生のストリーム。

## tracefs: 制御インターフェース {#tracefs-the-control-surface}

すべては`/sys/kernel/tracing`を通じて操作する。

| ファイル | 用途 |
|---|---|
| `current_tracer` | 有効なトレーサーを選択する（`function`、`function_graph`、`nop`、…） |
| `tracing_on` | マスターのオン/オフスイッチ（`0`/`1`を書き込む） |
| `buffer_size_kb` | per-CPUバッファのサイズ |
| `options/` | 上書きモードを含む各種トグル |
| `trace_marker` | 書き込むと*独自の*イベントをタイムラインに挿入できる |
| `instances/` | 独立したバッファを作成し、2つのツールが1つのバッファを奪い合わないようにする |

`instances/`は目立たないが重要である。各インスタンスは独自のトレーサーとイベントを持つ完全に別個のリングバッファであるため、バックグラウンドのトレーサーと場当たり的な調査を同時に実行しても、互いの捕捉内容を壊さない。

## 試してみる {#try-it-yourself}

```bash
cd /sys/kernel/tracing

# 履歴を長くするため per-CPU バッファを 8 MB に拡大する
echo 8192 > buffer_size_kb

# スケジューラの切り替えを発生と同時にライブで tail する（消費する読み出し）
echo 1 > events/sched/sched_switch/enable
cat trace_pipe        # Ctrl-C で停止

# メインバッファを乱さない、独立したインスタンス
mkdir instances/myprobe
echo 1 > instances/myprobe/events/block/block_rq_issue/enable
cat instances/myprobe/trace
```

## 参考文献 {#further-reading}

- [Kernel docs: ring buffer design](https://docs.kernel.org/trace/ring-buffer-design.html) — ロックレスアルゴリズムの全容
- [`kernel/trace/ring_buffer.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/trace/ring_buffer.c) — 実装
- [ftrace](ftrace.md) — このバッファを埋めるトレーサー
