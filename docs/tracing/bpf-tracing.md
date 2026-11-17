# トレーシングのためのBPF（bpftraceとBCC） {#bpf-for-tracing-bpftrace-and-bcc}

> Linuxを観測する現代的な方法: 検証済みの小さなプログラムをほぼあらゆるイベントにアタッチし、結果を*カーネル内で*集計する

静的tracepoint、ftrace、perfはいずれもイベントを捕捉し、それをユーザー空間に送ってから集計する。このモデルには上限がある。毎秒数百万イベントともなると、バッファへの書き込み、コピー、ユーザー空間でのパースがボトルネックになり、得られる知見よりも計測そのものにコストを費やすことになる。BPFはこの上限を取り払う。それがBPFが主流のトレーシングフロントエンドになった理由である。

## なぜBPFがトレーシングを変えたか {#why-bpf-changed-tracing}

生のイベントを外へ流し出す代わりに、BPFでは小さなプログラムをアタッチし、それが**イベントの発生地点で、カーネル内で**実行されてその場で集計を行う。per-CPUマップのインクリメント1回が、バッファ書き込み＋コピー＋ユーザー空間でのパースを置き換える。つまり読み出すのは*ヒストグラム*であって、それを生み出した大量のイベントの奔流ではない。

本番環境で使えるほど安全にしているのは、2つの性質である。

- **検証器（verifier）**が、プログラムがアタッチされる前に、カーネルをクラッシュさせたり、無限ループしたり、触れてはならないメモリに触れたりしないことを静的に証明する（検証器とJITの仕組みについては[BPF](../bpf/README.md)を参照）。
- **動的である**。ロード、アタッチ、デタッチはすべて実行時に行え、再起動もモジュールも不要である。

## どこにアタッチできるか {#where-you-can-attach}

トレーシング用BPFプログラムは、イベントが発生するほぼあらゆる場所にフックできる。

- **kprobe / kretprobe** — ほぼ任意のカーネル関数の入口または戻り
- **tracepoint**と**raw tracepoint** — カーネルの静的計装ポイント
- **fentry / fexit** — 関数の入口／出口をよりオーバーヘッド少なくトレースする方法。[ftrace](ftrace.md)が使うのと同じ、コンパイラが挿入する`__fentry__`フックの上に構築されている。kprobeより安価で、引数に型付きでアクセスできる
- **perfイベント** — PMUカウンタと時間ベースのサンプリング（プロファイリング用）
- **uprobe / USDT** — ユーザー空間の関数と静的マーカー

## マップ: カーネル内にとどまる集計 {#maps-aggregation-that-stays-in-the-kernel}

**マップ**は、ホットパスでユーザー空間を介さずに、プログラムがイベントをまたいで状態を蓄積する手段である。PIDをキーとするハッシュ、per-CPUカウンタ、レイテンシのlog2**ヒストグラム**などがある。ユーザー空間はイベントごとではなく、答えが欲しいときに一度だけマップを読み出す。

## 2つのツールキット {#the-two-toolkits}

- **bpftrace** — ワンライナーや短いスクリプト向けの、awkに似た高水準言語。対話的な調査のための近道である。
- **BCC** — 完成したパッケージ済みツールを構築するためのPython/C++ライブラリ。`bcc-tools`スイート（`execsnoop`、`biolatency`、`tcplife`、`runqlat`、…）はこれで書かれている。

## 実践 {#in-practice}

```bash
# プロセス名ごとにシステムコールを数える
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'

# vfs_read() のレイテンシヒストグラムをカーネル内で — fentry/fexit で入口と出口を捕捉
bpftrace -e '
  fentry:vfs_read { @start[tid] = nsecs; }
  fexit:vfs_read  /@start[tid]/ {
      @ns = hist(nsecs - @start[tid]); delete(@start[tid]);
  }'

# 誰がどのファイルを開いているか
bpftrace -e 'tracepoint:syscalls:sys_enter_openat {
    printf("%s -> %s\n", comm, str(args->filename)); }'
```

いずれも集計やバケット分けをカーネル内で行い、終了時にコンパクトな要約を表示する。[ftraceのヒストグラムトリガー](ftrace.md)が答えるのと同じ問いだが、任意に拡張できる汎用言語で扱える。

## 位置づけ {#where-this-fits}

トレーシングのためのBPFは、汎用BPF機構の可観測性向けの用途である。このページは問いに答えるためにBPFを*使う*ことを扱う。検証器、JIT、マップ、プログラムのライフサイクルが実際にどう動くかについては[BPF](../bpf/README.md)セクションを参照。

## 参考文献 {#further-reading}

- [Kernel docs: BPF](https://docs.kernel.org/bpf/index.html) — サブシステムのリファレンス
- [bpftrace reference guide](https://github.com/bpftrace/bpftrace/blob/master/man/adoc/bpftrace.adoc) — 言語とそのプローブ種別
- [BPF](../bpf/README.md) — これらのツールの背後にある検証器、JIT、マップ、プログラムモデル
- [kprobeとtracepoint](kprobes-tracepoints.md) — BPFがアタッチするプローブ機構
