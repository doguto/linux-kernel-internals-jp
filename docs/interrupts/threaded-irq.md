# スレッド化IRQ {#threaded-irqs}

> 時間のかかる処理のために、割り込みの処理をプロセスコンテキストへ移す

## hardirqハンドラの問題 {#the-problem-with-hardirq-handlers}

ハードウェア割り込みハンドラ（hardirq）は、現在のCPUで割り込みが無効化された状態で動作する。これは次のことを意味する。

- スリープできない
- スリープするロック（mutex、セマフォ）を取得できない
- 同じCPU上の他の割り込みをブロックする
- 他の高優先度割り込みのレイテンシが増える

複雑なデバイス（I2Cバス、SPIコントローラ、タッチスクリーン）では、割り込みハンドラがミリ秒単位の時間がかかりうるI2C/SPIのレジスタ読み込みを行う必要がある。これはhardirqハンドラにとってはあまりに長すぎる。

**スレッド化IRQ**はThomas GleixnerによってLinux 2.6.30で導入された[(commit)](https://git.kernel.org/linus/3aa551c9b4c40018f0e261a178e3d25478dc04a9) [(LWN)](https://lwn.net/Articles/302043/)。これは重い処理を、プロセスコンテキストで動作する専用のカーネルスレッドに移すことでこの問題を解決する。

## これは実際どこから来たのか {#where-this-actually-came-from}

メインラインは2009年にスレッド化割り込み処理をゼロから発明したのではなく、ツリー外の[PREEMPT_RT](../locking/preempt-rt.md)パッチセットが何年も前から依存していたアイデアを取り込んだのである。元の提案を扱ったLWNの記事はこの点を明言している。*"In the realtime tree, nearly all drivers were mass converted to use threads"*（訳: リアルタイムツリーでは、ほぼすべてのドライバがスレッドを使うように一括変換された）。PREEMPT_RTの前提そのものが、割り込みを完全に無効化したまま長時間動作するものが何もないことに依存しているため、RTビルドではhardirqの処理をスケジュール可能でプリエンプト可能なスレッドに移す以外に選択肢がなかったのである（[LWN](https://lwn.net/Articles/302043/)、Jake Edge、2008年10月8日）。

Gleixnerがメインライン向けに提案したものは、RTツリーのやり方よりも範囲が狭く保守的だった。すべてのドライバを強制的に変換するのではなく、`request_threaded_irq()`によってスレッド化を*オプトイン*にし、ドライバを1つずつ移行させるようにした。この設計判断は特定の制約に由来する。LWNの報告によれば、*"as requested by Linus Torvalds at this year's Kernel Summit, a new function was introduced rather than changing countless drivers to use a new `request_irq()`"*（訳: その年のKernel SummitでLinus Torvaldsが求めたとおり、無数のドライバを新しい`request_irq()`を使うように変更するのではなく、新しい関数が導入された）。Gleixnerはまた、自動的・機械的な変換が目的ではないことを率直に述べている。*"Converting an interrupt to threaded makes only sense when the handler code takes advantage of it by integrating tasklet/softirq functionality and simplifying the locking."*（訳: 割り込みをスレッド化することに意味があるのは、ハンドラのコードがtasklet/softirqの機能を統合し、ロックを単純化することでそれを活用する場合だけである）。スレッド化それ自体のためにスレッド化しても、ドライバが得るものは何もない。その利点が現れるのは、スレッド化によって、本来ならhardirqハンドラと並べて必要になるtaskletやsoftirqをドライバが削除できる場合だけである。

同じ記事は、期待される見返りがレイテンシにとどまらないことにも触れている。*"Threaded handlers will also help the debuggability of the kernel and may eventually lead to the removal of tasklets from Linux."*（訳: スレッド化ハンドラはカーネルのデバッグ容易性も高め、最終的にはLinuxからtaskletを削除することにつながるかもしれない）。このつながりこそが、[taskletがスレッド化IRQを優先して非推奨になった](tasklets.md#the-deprecation-nobody-planned-to-start-then-couldnt-finish)理由である。2008年の提案は、実際の削除の動きが始まる10年以上前に、いずれtaskletを削除可能にする仕組みをすでに名指ししていたのである。

## request_threaded_irq {#requestthreadedirq}

```c
int request_threaded_irq(unsigned int irq,
                         irq_handler_t handler,     /* hardirqハンドラ（プライマリ） */
                         irq_handler_t thread_fn,   /* スレッド化ハンドラ（セカンダリ） */
                         unsigned long irqflags,
                         const char *devname,
                         void *dev_id);
```

スレッド化IRQが発火すると:
1. `handler`（プライマリ）がhardirqコンテキストで動作する
   - スレッドを起床させるには`IRQ_WAKE_THREAD`を返し、そうでなければ`IRQ_HANDLED` / `IRQ_NONE`を返さなければならない
2. IRQスレッド（`irq/N-name`という名前）がプロセスコンテキストで`thread_fn`を実行する
   - スリープ、mutexの取得、I2Cの読み込みなどが可能
   - `IRQF_ONESHOT`は、`thread_fn`が戻るまでIRQラインをマスクしたままにする

## IRQF_ONESHOTの要件 {#the-irqf_oneshot-requirement}

レベルトリガ割り込みでスレッド化IRQを使う場合は、`IRQF_ONESHOT`を**必ず**使わなければならない。理由は次のとおりである。

`IRQF_ONESHOT`がない場合:
1. 割り込みが発火 → hardirqが動作 → IRQラインがアンマスクされる → 割り込みが再び発火
2. スレッドは起床するが、スレッドがレジスタを読む前に割り込みが再び発火する
3. IRQラインが一向に静まらない → 割り込みストーム

`IRQF_ONESHOT`がある場合:
1. 割り込みが発火 → hardirqが動作 → IRQラインは**マスクされたまま**
2. スレッドが動作 → レジスタを読み、割り込み要因をクリアする
3. スレッドが完了 → カーネルがIRQラインを**アンマスク**する
4. これでラインは静かになる

```c
/* レベルトリガのスレッド化IRQでは常にIRQF_ONESHOTを使う */
request_threaded_irq(irq, primary_handler, thread_fn,
                     IRQF_ONESHOT | IRQF_TRIGGER_LOW, "my-device", dev);
```

## スレッド化IRQの完全な例 {#a-complete-threaded-irq-example}

```c
/* プライマリハンドラ: hardirqコンテキストで動作する */
static irqreturn_t my_primary_handler(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;

    /* 最小限の処理: 自分の割り込みかどうかを確認する */
    if (!(readl(dev->regs + STATUS) & MY_BIT))
        return IRQ_NONE;

    /* 再発火を防ぐため、（デバイス側で）割り込み要因を無効化する */
    writel(0, dev->regs + IRQ_ENABLE);

    /* 実際の処理を行うためにスレッドを起床させる */
    return IRQ_WAKE_THREAD;
}

/* スレッドハンドラ: プロセスコンテキストで動作する */
static irqreturn_t my_thread_handler(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;

    /* スリープ、mutexの取得、I2Cの読み込みなどが可能 */
    i2c_smbus_read_byte_data(dev->client, REG_DATA);

    /* データを処理する */
    input_report_key(dev->input, KEY_ENTER, 1);
    input_sync(dev->input);

    /* デバイス側の割り込みを再度有効化する */
    writel(MY_BIT, dev->regs + IRQ_ENABLE);

    return IRQ_HANDLED;
}

static int my_probe(struct i2c_client *client)
{
    /* ... */
    return devm_request_threaded_irq(&client->dev, client->irq,
                                     my_primary_handler,
                                     my_thread_handler,
                                     IRQF_ONESHOT | IRQF_TRIGGER_LOW,
                                     "my-touchscreen", dev);
}
```

## NULLのプライマリハンドラ {#null-primary-handler}

すべての処理をスレッドに任せることができ、hardirqコンテキストで「これは自分の割り込みか？」を判断する必要がなければ、プライマリハンドラとして`NULL`を渡せる。

```c
/* NULLのプライマリ: カーネルが常にIRQ_WAKE_THREADを返すデフォルトを提供する */
request_threaded_irq(irq, NULL, my_thread_fn,
                     IRQF_ONESHOT | IRQF_TRIGGER_RISING, "my-sensor", dev);
```

カーネルのデフォルトのプライマリハンドラは、割り込みを受領通知してスレッドを起床させる。

## IRQスレッド {#the-irq-thread}

カーネルはスレッド化IRQごとに1つのカーネルスレッドを作成し、`irq/N-name`（NはIRQ番号）という名前を付ける。

```bash
# IRQスレッドを一覧表示する
ps aux | grep "irq/"
# root     1234  0.0  0.0      0     0 ?   S    10:00   0:00 [irq/45-eth0]
# root     1235  0.0  0.0      0     0 ?   S    10:00   0:00 [irq/46-spi0]

# IRQスレッドはデフォルトでSCHED_FIFOの優先度50で動作する
chrt -p 1234  # スケジューリングポリシーと優先度を表示する
```

IRQスレッドの優先度は調整できる。

```bash
# IRQ 45のスレッドをSCHED_FIFOの優先度90に設定する（RTワークロード向け）
chrt -f -p 90 $(pgrep -f "irq/45")
```

## スレッド化IRQを使うべきとき {#when-to-use-threaded-irqs}

```
Use threaded IRQs when:
  ✓ Handler needs to do I2C/SPI communication (sleeps)
  ✓ Handler needs to acquire a mutex
  ✓ Handler does complex processing that takes > a few µs
  ✓ Level-triggered interrupt (IRQF_ONESHOT prevents storm)

Keep in hardirq when:
  ✓ Very simple: read register, copy data, ack
  ✓ Network packet receive (NAPI handles deferral differently)
  ✓ Hard real-time requirements (avoid thread scheduling latency)
```

## 参考文献 {#further-reading}

- [request_irqとfree_irq](request-irq.md) — スレッド化しない版
- [Workqueue](workqueues.md) — 処理をプロセスコンテキストへ遅延させる別の方法
- [Softirq](softirq.md) — softirqコンテキストに留まる必要がある遅延処理向け
