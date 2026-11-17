# トレースイベントと動的イベント {#trace-events-and-dynamic-events}

> カーネルに組み込まれた数千の計装ポイント — それらがどのように定義され、フィルタされ、トリガーされるか — と、実行時に独自のものを追加する方法

[リングバッファ](ring-buffer.md)はイベントを記録する。そこを埋めるのが**トレースイベント**である。カーネルには数千のトレースイベントが同梱されており、独自のものをアタッチすることもできる。この層を理解することで、「既成のトレースを実行できる」段階から「カーネルに的確な問いを投げかけられる」段階へと進める。

## 静的トレースイベント: `TRACE_EVENT` {#static-trace-events-trace_event}

静的トレースイベントは、カーネル開発者が`TRACE_EVENT`マクロで配置した計装ポイントである。1つのマクロから、カーネルは3つのものを生成する。tracepointの呼び出し箇所、捕捉するフィールドの*型付きレコード*、そしてツールがデコード方法を知るための自己記述的な**フォーマット**である。意図的に宣言されており、無効時のコストは実質ゼロであるため、静的イベントは本番環境向けの安定した推奨される計装である。

これらはtracefs配下に、サブシステムごとにまとめて現れる。

```
/sys/kernel/tracing/events/
├── sched/
│   ├── sched_switch/
│   │   ├── enable      # 1 を書き込むとこのイベントが有効になる
│   │   ├── filter      # イベントごとの述語（カーネル内で評価される）
│   │   ├── trigger     # イベント発火時に実行するアクション
│   │   └── format      # デコーダ向けのフィールドレイアウト
│   └── sched_wakeup/
├── block/
├── syscalls/
└── ...
```

有効にするのは1回の書き込みで済む。

```bash
echo 1 > /sys/kernel/tracing/events/sched/sched_switch/enable
```

## フィルタ: 記録する価値があるものをカーネル内で判断する {#filters-decide-in-kernel-whats-worth-recording}

`filter`ファイルは、イベントがバッファに書き込まれる**前に**実行される述語をアタッチする。記録コストがかかるのは条件に一致したイベントだけで、残りは発生源で破棄される。これにより、頻繁に発生するイベントでも溺れることなくトレースできる。

```bash
# 4 KiB より大きい read のみを記録する
echo 'count > 4096' > events/syscalls/sys_enter_read/filter
```

フィルタはイベント自身のフィールド（`format`に列挙されているもの）を参照し、通常の比較演算と`&&` / `||`をサポートする。

## トリガー: イベント発火時に動作する {#triggers-act-when-an-event-fires}

`trigger`ファイルは、イベントに*アクション*をアタッチする。単に記録する代わりに、カーネルが反応できるようになる。

- `traceon` / `traceoff` — トレーシングを開始または停止する（例: まれなエラーイベントが発火した瞬間に停止し、そこに至るまでのフライトレコーダーの履歴を凍結する）
- `stacktrace` — イベント時点のカーネルスタックを捕捉する
- `snapshot` — 後で調べるためにバッファを退避させる
- `hist` — イベントのフィールドをキーとする**カーネル内ヒストグラム**を構築する

ヒストグラムトリガーは目立たないが強力である。ユーザー空間へのストリームもBPFも一切使わずに、カーネル内で集計する。

```bash
# プロセスごとの read サイズのヒストグラムを、完全にカーネル内で
echo 'hist:key=comm:val=count' > events/syscalls/sys_enter_read/trigger
cat events/syscalls/sys_enter_read/hist
```

## 動的イベント: 実行時に計装を追加する {#dynamic-events-add-instrumentation-at-runtime}

必要な場所に静的イベントが存在しない場合は、その場で作成できる。作成したイベントは`events/`配下に現れ、組み込みイベントとまったく同じように振る舞う（enable/filter/trigger/formatも同じ）。

- **kprobeイベント**（`kprobe_events`） — 任意のカーネル関数や命令をプローブし、選んだ引数を捕捉する
- **uprobeイベント**（`uprobe_events`） — 同じことをユーザー空間のバイナリに対して行う
- **合成イベント**（`synthetic_events`） — *2つの*イベントのフィールドを組み合わせる（例: 終了タイムスタンプから開始タイムスタンプを引いてレイテンシを測る）

```bash
# do_sys_openat2() の filename 引数を捕捉する kprobe イベントを作成する
echo 'p:myopen do_sys_openat2 file=+0(%si):string' > kprobe_events
echo 1 > events/kprobes/myopen/enable
cat trace_pipe
```

結果として統一されたモデルが得られる。イベントが10年前からある静的tracepointであっても、1秒前にkprobeで定義したものであっても、同じtracefsインターフェースを通して有効化、フィルタ、トリガー、デコードが行われる。基盤となるプローブの仕組みについては[kprobeとtracepoint](kprobes-tracepoints.md)を参照。

## 参考文献 {#further-reading}

- [Kernel docs: event tracing](https://docs.kernel.org/trace/events.html) — イベント、フィルタ、トリガー
- [Kernel docs: histogram triggers](https://docs.kernel.org/trace/histogram.html) — カーネル内の`hist`集計
- [kprobeとtracepoint](kprobes-tracepoints.md) — 動的イベントの背後にあるプローブ機構
