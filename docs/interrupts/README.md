# 割り込みと非同期処理 {#interrupts-and-async-processing}

> ハードウェアイベントがどのようにカーネルコードへ届くか、そして遅延された処理がどのように実行されるか

## 割り込みの問題 {#the-interrupt-problem}

ハードウェアデバイスは、何かが起きたこと（ネットワークパケットの到着、ディスク読み込みの完了、タイマーの満了など）をCPUに伝える必要がある。CPUがあらゆるものを常時ポーリングすることはできないため、ハードウェアは**割り込み**を使う。割り込みとは、CPUが実行中の処理を止めさせ、割り込みハンドラを実行させるシグナルである。

しかし割り込みハンドラは高速でなければならない。割り込みハンドラは割り込みが（部分的に）無効化された状態で動作し、他のすべての割り込みをブロックするからである。そこでカーネルの設計では、処理を2つの半分に分割する。

```
Hardware interrupt
    │
    ▼
Top half (hardirq)     ← runs immediately, must be fast
  - Acknowledge interrupt
  - Copy data from device
  - Schedule bottom half

Bottom half            ← deferred, runs with interrupts enabled
  - Process the data
  - Complete the work
  - May sleep (workqueues only)
```

## なぜ遅延処理の仕組みが1つではなく5つあるのか {#why-five-deferred-work-mechanisms-not-one}

この2分割自体は簡単な部分である。より難しい問いは、カーネルのこの部分に初めて触れる人がもっともな疑問として抱くもので、上図の「ボトムハーフ」の箱を実装する方法が、明快な1つの答えではなく、なぜ*5つ*（softirq、tasklet、workqueue、スレッド化IRQ、タイマー/hrtimer）もあるのか、というものである。短く言えば、それぞれの仕組みは、以前の仕組みが何らかの実際のワークロードに適さないと判明したために存在し、カーネルは古い仕組みを丸ごと置き換えるのではなく、たいていそれを残したまま新しい選択肢を横に追加してきた。その系譜は[Softirq](softirq.md#from-bottom-halves-to-softirqs-to-threaded-irqs)のページで詳しく述べている。Linux 2.0〜2.3の最初の単一ロック方式の「ボトムハーフ」（BH）機構は、*すべて*の遅延処理を1つのグローバルspinlockの背後で直列化しており、これがSMPスケーラビリティの壁となった。softirqは、異なる種類の遅延処理を異なるCPU上で並列に実行できるようにすることでこれを置き換えた。そしてtaskletは、すべてのドライバ作者にリエントラントでper-CPU安全なコードを書くことを求めずに、softirqに似た仕組みをドライバに提供した。

しかし、これら3つはいずれもスリープできない。そして2000年代半ばには、本当にスリープを必要とするドライバが増えていた。I2CやSPIの読み込み、mutexを取得しなければならないあらゆる処理である。その隙間を埋めるのが[スレッド化IRQ](threaded-irq.md#where-this-actually-came-from)と[workqueue](workqueues.md)であり、特にスレッド化IRQには、最初に知っておく価値のある具体的で検証可能な起源がある。メインラインは2009年にこのアイデアを発明したのではなく、ツリー外の[PREEMPT_RT](../locking/preempt-rt.md)パッチセットが何年も前から依存していたものを取り込んだのである。RTの前提そのもの、すなわち最悪ケースのレイテンシに上限を設けることは、割り込みを完全に無効化したまま長時間動作するものとは相容れないからである。同じパターンはこのサブシステムでさらに2回現れる。[hrtimer](timers.md#hrtimer-high-resolution-timers)が存在するのは、高分解能タイマーのパッチをRTツリーに統合する際に「Linuxはアーキテクチャをまたいで時点をどう精密に表現するのか」という問いに本当の答えを出さざるを得なくなったためであり、それ以前には単一目的・単一アーキテクチャ向けのパッチが散在しているだけだった。また、スレッド化IRQを優先して[taskletを非推奨化する](tasklets.md#the-deprecation-nobody-planned-to-start-then-couldnt-finish)動きの圧力の多くは、RTツリーを構築・保守してきたのと同じLinutronix系のエンジニア（Thomas Gleixner、Sebastian Andrzej Siewior）から来ていた。taskletはアトミックコンテキスト専用で、スリープもプリエンプトもできず、まさにRTのモデルが受け入れる余地のない種類のものだからである。

これは古い仕組みが消えたことを意味しない。softirqが今も存在するのは、ごく一部の非常にホットなパス（特にネットワークの`NET_RX_SOFTIRQ`）が、スレッドスケジューリングの往復を本当に許容できないからである。taskletも非推奨ではあるが削除されずにツリーに残っている。一部のドライバ（AMDの`ccp`暗号エンジン、Intelの`i915` GPUドライバ）が、1行の変換では済まない形でtaskletを他の仕組みと組み合わせているからである。これらのページ全体で目にするのは、単一の一貫した設計というよりも、およそ30年にわたって積み重なった5つの答えであり、それぞれがその時点で作者の目の前にあった具体的な問題を解決したものである。そしてツリー外のリアルタイム化の取り組みが、場当たり的な修正をいずれすべてのドライバが使えるものへと一般化させる、並外れて粘り強い圧力源として働いてきた。

## 目次 {#contents}

### 基礎 {#fundamentals}
- [割り込み処理の概要](interrupts.md) — ハードウェアからハンドラまで
- [IRQディスクリプタとirq_chip](irq-desc.md) — カーネルの割り込み抽象化
- [request_irqとfree_irq](request-irq.md) — 割り込みハンドラの登録
- [スレッド化IRQ](threaded-irq.md) — 処理をhardirqコンテキストの外へ移す

### 遅延処理（ボトムハーフ） {#deferred-work-bottom-halves}
- [Softirq](softirq.md) — 最も低レベルな遅延機構
- [Tasklet](tasklets.md) — per-CPUで直列化されたsoftirqの利用者
- [Workqueue](workqueues.md) — プロセスコンテキストでの遅延処理
- [タイマーとhrtimer](timers.md) — 特定の時刻に実行する遅延処理

### 事件簿 {#war-stories}
- [事件簿](war-stories.md) — IRQアフィニティとベクタ割り当ての仕組みにおける2件のCVEと2件の長年のバグ

## 実行コンテキスト {#execution-contexts}

コードがどのコンテキストで動作するかを理解することは、適切なロックを選ぶうえで極めて重要である。

| コンテキスト | `in_interrupt()` | スリープ可能？ | プリエンプト可能？ |
|---------|-----------------|------------|--------------|
| プロセスコンテキスト | false | 可 | 可 |
| Softirq | true | 不可 | 不可 |
| Hardirq | true | 不可 | 不可 |
| スレッド化IRQ | false | 可 | 可 |
| Workqueue | false | 可 | 可 |

`in_interrupt()`はsoftirqコンテキストとhardirqコンテキストの両方でtrueを返す。hardirqのみを判定するには`in_hardirq()`を、softirqのみを判定するには`in_softirq()`を使う。
