# Softirq {#softirqs}

> カーネルの高優先度な遅延処理機構

## ボトムハーフからsoftirq、そしてスレッド化IRQへ {#from-bottom-halves-to-softirqs-to-threaded-irqs}

Linuxにおける割り込み処理の遅延は3つの世代を経てきており、それぞれが前の世代の制約に対処してきた。

### ボトムハーフ（Linux 2.0〜2.3） {#bottom-halves-linux-2023}

最初の遅延処理機構は「ボトムハーフ」（BH）と呼ばれていた。そのモデルは次のとおりである。割り込みハンドラ（「トップハーフ」）は最小限の処理だけを行い、後で実行するBHに印を付ける。次に安全なタイミングで、カーネルが印の付いたBHを実行する。

致命的な欠陥は、**システム全体で同時に実行できるBHは1つだけ**であり、それがグローバルなspinlockで保護されていたことである。SMPマシンでは、BHハンドラを実行するためにすべてのCPUが1つのロックを通じて直列化されることを意味した。CPU 0でのネットワーク受信が、CPU 1でのタイマー処理をブロックした。1990年代後半にSMPシステムが一般的になるにつれ、BHモデルはスケーラビリティのボトルネックになった。

また、BHのスロットは静的に定義された32個しかなく、ドライバが独自のものを追加することはできなかった。

### Softirq（Linux 2.3、SMP向けの書き直し） {#softirqs-linux-23-smp-rewrite}

softirqは、SMPスケーラビリティ向上の取り組みの中でLinux 2.3においてBHを置き換えた。重要な変更点は、**複数のsoftirqが異なるCPU上で並列に実行できる**ことである。CPU 0の`NET_RX_SOFTIRQ`が、CPU 1の`TIMER_SOFTIRQ`をブロックすることはもはやない。

そのトレードオフとして、softirqは並列に実行されうるため、softirqハンドラ自身はリエントラントでなければならず、共有状態にはper-CPUデータかspinlockを使わなければならない。これはすべてのドライバ作者に求めるには難しすぎたため、softirqは静的に定義され、厳しく監査されたサブシステムのコードに限定されたままとなった。

tasklet（`TASKLET_SOFTIRQ`の上に構築されている）は、リエントラントである必要なしに処理をsoftirqコンテキストへ遅延させる手段をドライバに与えるために追加された。taskletは同時に1つのCPU上でしか実行されないことが保証されるが、異なるtaskletどうしは並列に実行されうる。

### スレッド化IRQ（Linux 2.6.30、2009年） {#threaded-irqs-linux-2630-2009}

taskletはリエントランシーの問題を解決したが、新たな問題を生んだ。taskletはsoftirqコンテキストで動作するため、スリープできず、素早く完了しなければならず、さらに`bh_disable()`のセマンティクスで動作するため高優先度プロセスのスケジューリングレイテンシを引き起こす。

Thomas Gleixnerは、ほとんどのドライバにとってより良いモデルとしてスレッド化IRQハンドラを導入した[(commit)](https://git.kernel.org/linus/3aa551c9b4c40018f0e261a178e3d25478dc04a9) [(LWN)](https://lwn.net/Articles/302043/)。`IRQF_THREAD`を使うと、割り込みハンドラはhardirq/softirqコンテキストではなくカーネルスレッドとして動作する。

- ハンドラは**スリープできる**（mutex、GFP_KERNELでの割り当てなど）
- **スケジュール可能な優先度**を持つ。RTシステムでは適切な優先度を与えられる
- `CONFIG_PREEMPT_RT`では、すべてのsoftirq処理（タイマーを含む）がスレッドで動作し、システムが完全にプリエンプト可能になる

taskletは、Linux 5.9（2020年）で新規の使用について明示的に非推奨とされた[(LWN)](https://lwn.net/Articles/830964/)。非推奨化をめぐる議論そのものについては[Tasklet](tasklets.md#the-deprecation-nobody-planned-to-start-then-couldnt-finish)を参照。新しいドライバコードに対する推奨は次のとおりである。ハンドラがスリープでき、プロセスコンテキストを必要とするならスレッド化IRQか通常のworkqueueを使い、taskletと同じようにアトミックな（softirqに似た）コンテキストに留まらなければならないなら`WQ_BH` workqueueを使う。`WQ_BH`は、その高速パスのケースに対するより直接的で機械的な置き換えであり、プロセスコンテキストの代替手段ではない。taskletは既存のドライバには残っているが、新しいコードに現れるべきではない。

## softirqとは何か {#what-are-softirqs}

softirq（ソフトウェア割り込み）は、カーネルで最も低レベルな遅延実行機構である。割り込みを無効にして動作するhardirqハンドラと異なり、softirqは割り込みが**有効**な状態で動作する。hardirqによってプリエンプトされうるが、同じCPU上の他のsoftirqによってプリエンプトされることはない。

softirqは静的に定義されている。softirqの種類はちょうど10個で、ブート時に登録される。新しい種類を動的に追加することはできない。ドライバはsoftirqを直接使わず、代わりに[tasklet](tasklets.md)や[workqueue](workqueues.md)を使う。

## 10種類のsoftirq {#the-10-softirq-types}

```c
/* include/linux/interrupt.h */
enum {
    HI_SOFTIRQ      = 0,  /* 高優先度のtasklet */
    TIMER_SOFTIRQ   = 1,  /* タイマーホイールの満了 */
    NET_TX_SOFTIRQ  = 2,  /* ネットワーク送信 */
    NET_RX_SOFTIRQ  = 3,  /* ネットワーク受信（NAPI） */
    BLOCK_SOFTIRQ   = 4,  /* ブロック層の完了処理 */
    IRQ_POLL_SOFTIRQ= 5,  /* IRQポーリング（blk-mq） */
    TASKLET_SOFTIRQ = 6,  /* 通常優先度のtasklet */
    SCHED_SOFTIRQ   = 7,  /* スケジューラ（負荷分散） */
    HRTIMER_SOFTIRQ = 8,  /* 高分解能タイマー */
    RCU_SOFTIRQ     = 9,  /* RCUコールバック */
    NR_SOFTIRQS
};
```

番号は**優先度**であり、番号が小さいほど先に実行される。高優先度tasklet用の`HI_SOFTIRQ`（優先度0）は、通常のtasklet用の`TASKLET_SOFTIRQ`（優先度6）より常に先に実行される。

## softirqはどのように起動されるか {#how-softirqs-are-triggered}

softirqは`raise_softirq()`を呼ぶことで発生させる。これはper-CPUの`__softirq_pending`ビットマスクのビットを立てる。

```c
/* softirqを発生させる（hardirqコンテキストから呼べる） */
raise_softirq(NET_RX_SOFTIRQ);

/* IRQがすでに無効化されているときに使う版 */
raise_softirq_irqoff(NET_TX_SOFTIRQ);
```

## softirqはいつ実行されるか {#when-softirqs-run}

softirqは3つの時点で処理される。

1. **hardirqハンドラが戻った後**（`irq_exit_rcu()`）
2. softirqの頻度が高すぎる場合は**ksoftirqdスレッド内**で
3. ボトムハーフが再度有効化されたときに`local_bh_enable()`内で**明示的に**

`__do_softirq()` → `handle_softirqs()`のループ:

```c
/* kernel/softirq.c（簡略化） */
static void handle_softirqs(bool ksirqd)
{
    __u32 pending = local_softirq_pending();

    /* IRQを有効にする（softirqは割り込みが有効な状態で動作する） */
    local_irq_enable();

    /* 保留中の各softirqを優先度順に処理する */
    while ((softirq_bit = ffs(pending))) {
        h = &softirq_vec[softirq_bit - 1];
        h->action();    /* 登録されたハンドラを呼ぶ */
        pending >>= softirq_bit;
    }

    local_irq_disable();

    /* さらにsoftirqが保留中で時間に余裕があれば、やり直す */
    if (pending) {
        if (time_before(jiffies, end) && !need_resched() && --max_restart)
            goto restart;
        /* そうでなければksoftirqdを起床させる */
        wakeup_softirqd();
    }
}
```

このループは`ksoftirqd`を起床させる前に`MAX_SOFTIRQ_RESTART`（10）回の反復に制限されており、softirqがプロセスを無期限に飢餓状態にすることを防いでいる。

## ksoftirqd {#ksoftirqd}

softirqの頻度が高すぎる場合（高いネットワーク負荷、多数のタイマー）、ループの実行時間が長くなりすぎる。代わりにカーネルは`ksoftirqd/N`（CPUごとに1つ）を起床させる。これは`SCHED_NORMAL`優先度のカーネルスレッドで、保留中のsoftirqを処理しきる。

```bash
# ksoftirqdスレッド: CPUごとに1つ
ps aux | grep ksoftirqd
# root         9  0.0  0.0      0     0 ?   S    10:00   0:01 [ksoftirqd/0]
# root        18  0.0  0.0      0     0 ?   S    10:00   0:00 [ksoftirqd/1]
```

`ksoftirqd`は通常のプロセスなので、ユーザータスクによってプリエンプトされうる。これによりライブロックが防がれる。`ksoftirqd`のCPU使用率が高いことは、システムが多くの割り込みを処理していることを示す。

## softirqの動作の確認 {#checking-softirq-activity}

```bash
# CPUごとのsoftirq回数
cat /proc/softirqs
#                     CPU0       CPU1       CPU2       CPU3
#           HI:          1          1          1          1
#        TIMER:    1234567    1234456    1234123    1234234
#       NET_TX:       1234       1567       1123       1345
#       NET_RX:    2345678    2345234    2345123    2345456
#        BLOCK:      23456      23234      23123      23345
#     IRQ_POLL:          0          0          0          0
#      TASKLET:       1234       1234       1234       1234
#        SCHED:    3456789    3456234    3456123    3456456
#      HRTIMER:      12345      12234      12123      12234
#          RCU:    4567890    4567234    4567123    4567456

# NET_RXが高い = ネットワーク受信が多い
# TASKLETが高い = ドライバのtaskletが多い
# RCUが高い = 多数のRCUコールバックが処理されている
```

## softirqハンドラの登録（カーネルサブシステムのみ） {#registering-a-softirq-handler-kernel-subsystems-only}

```c
/* ブート時のみ（ドライバからではない） */
open_softirq(MY_SOFTIRQ_NR, my_softirq_action);

/* ハンドラのシグネチャ */
static void my_softirq_action(void)
{
    /* このCPUの処理を行う */
    /* IRQが有効な状態で動作し、hardirqによってプリエンプトされうる */
    /* スリープできない */
}
```

ドライバは代わりにtaskletかworkqueueを使うべきであり、新しいsoftirqの種類を追加してはならない。

## local_bh_disable / local_bh_enable {#localbhdisable-localbhenable}

softirqハンドラとデータを共有するコードは、そのデータにアクセスする間、ローカルCPUでsoftirqを無効にしなければならない。

```c
/* このCPUでsoftirq（とtasklet）を無効にする */
local_bh_disable();
/* softirqハンドラと共有するデータにアクセスする */
local_bh_enable();  /* 再度有効にし、保留中のsoftirqを実行する */

/* spin_lock_bh: spinlock + local_bh_disableを1回の呼び出しで */
spin_lock_bh(&my_lock);
/* softirqも触るデータに安全にアクセスできる */
spin_unlock_bh(&my_lock);
```

## 参考文献 {#further-reading}

- [Tasklet](tasklets.md) — ドライバから利用できるsoftirqのラッパー
- [Workqueue](workqueues.md) — 遅延処理の中でスリープする必要がある場合
- [ネットワークデバイスとNAPI](../net/napi.md) — 実際のNET_RX_SOFTIRQ
