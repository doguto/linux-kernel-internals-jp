# タイマーとhrtimer {#timers-and-hrtimers}

> カーネルの処理を特定の時刻にスケジュールする

## 2つのタイマーサブシステム {#two-timer-subsystems}

カーネルには2つの独立したタイマー機構がある。

| | timer_list | hrtimer |
|---|-----------|---------|
| 分解能 | jiffies（1ms〜10ms） | ナノ秒 |
| データ構造 | 階層的な配列（タイマーホイール） | 赤黒木 |
| コールバックのコンテキスト | softirq（TIMER_SOFTIRQ） | softirqまたはhardirq |
| 用途 | 粗いタイムアウト、ウォッチドッグ | 高精度、音声/映像、ユーザー空間のスリープ |

## timer_list: jiffiesベースのタイマー {#timer_list-jiffies-based-timers}

`timer_list`は伝統的なタイマーAPIである。タイマーはjiffiesの境界（`HZ`に応じて通常1ms〜10ms）で満了する。

```c
/* include/linux/timer.h */
struct timer_list {
    struct hlist_node entry;     /* タイマーホイール内の位置 */
    unsigned long expires;       /* jiffies単位の満了時刻 */
    void (*function)(struct timer_list *);
    u32 flags;
};
```

### API {#api}

```c
#include <linux/timer.h>

struct my_device {
    struct timer_list watchdog;
};

static void my_watchdog_handler(struct timer_list *t)
{
    struct my_device *dev = timer_container_of(dev, t, watchdog);
    /* TIMER_SOFTIRQコンテキストで動作する: スリープ不可 */
    if (device_stuck(dev))
        recover_device(dev);
    /* タイマーを再始動する */
    mod_timer(&dev->watchdog, jiffies + HZ * 5);  /* 5秒 */
}

/* 初期化 */
timer_setup(&dev->watchdog, my_watchdog_handler, 0);

/* タイマーを開始する: 5秒後に満了 */
mod_timer(&dev->watchdog, jiffies + HZ * 5);

/* 保留中かどうかを確認する */
if (timer_pending(&dev->watchdog)) { ... }

/* キャンセルする（非同期: まだ発火する可能性がある） */
timer_delete(&dev->watchdog);

/* キャンセルし、実行中のハンドラの完了を待つ */
timer_delete_sync(&dev->watchdog);

/* タイマーを新しい満了時刻で再始動する（保留中でなければ設定する） */
mod_timer(&dev->watchdog, jiffies + HZ);
```

### 時間変換のヘルパー {#time-conversion-helpers}

```c
jiffies + HZ           /* 1秒 */
jiffies + HZ / 2       /* 500ms */
jiffies + msecs_to_jiffies(250)  /* 250ms */
jiffies + usecs_to_jiffies(500)  /* 500µs */

/* 残り時間 */
long remaining = timer->expires - jiffies;
```

### タイマーホイール、そしてなぜ2016年に作り直されたのか {#the-timer-wheel-and-why-it-was-rebuilt-in-2016}

タイマーは**タイマーホイール**と呼ばれる階層的な配列構造に格納される。しかし現在のカーネルのホイールは、Linuxの歴史の大半で使われていたものとは違い、その違い自体が知っておく価値のある「なぜ」の話である。

元のホイールは、最も細かいレベルでは`jiffies`の下位8ビット（256スロット）でタイマーを索引し、その上により粗いレベルがあった。256 jiffiesごとに、カーネルは上のレベルからタイマーをまとめて個々のより細かいスロットへ「カスケード」しなければならなかった。この操作のコストは、Jonathan Corbetの言葉を借りれば「第一近似として、予測不能」であり、キャッシュフレンドリーでもなく、さらにこの設計には、複数のレベルを走査せずに「次に発火しなければならないのはいつか」に答える安価な方法がなかった（[LWN: Reinventing the timer wheel](https://lwn.net/Articles/646950/)、2015年6月）。タイマーがほとんど予定どおりに発火するだけだった頃は、これはあまり問題にならなかった。しかし`NO_HZ`（tickレスアイドル）によって「このCPUはどれだけ長く安全にスリープできるか」が、スケジューラが安価かつ頻繁に答えを必要とする問いになると、大きな問題になった。

Thomas Gleixnerによる書き直しは**Linux 4.8**（2016年）でマージされた（`kernel/time/timer.c`でv4.7には存在せず、v4.8には存在することを確認済み）。これは単にレベルのサイズを変えただけでなく、カスケードを完全に取り除いた。コード自身のコメントはそれを率直に述べている。*"We don't have cascading anymore. timers with a expiry time above the capacity of the last wheel level are force expired at the maximum timeout value of the last wheel level."*（訳: もはやカスケードは存在しない。満了時刻が最後のホイールレベルの容量を超えるタイマーは、最後のホイールレベルの最大タイムアウト値で強制的に満了させられる）。`HZ=1000`ではその容量はおよそ12日になる（`HZ`に比例し、tickレートが速いほど短く、遅いほど長い）。それ以上を要求するタイマーは、もはやカスケードによっていずれ正しく配置されるのを待つことはなく、単に最大値で発火するように切り詰められる。これは、数日単位の時間スケールでそれ以上の精度を必要とする正当なものはないという（計測に基づく）前提によるものである。レベルの*サイズ*も変わった。従来のホイールでは最初のレベルだけで256スロット（`TVR_SIZE`、それぞれ1 jiffy）あり、その上のすべてのレベルは64（`TVN_SIZE`）だった。書き直しではすべてのレベルを64スロット（`LVL_SIZE`）に統一した。`HZ=1000`での現在の実際のレイアウト（`kernel/time/timer.c`）は次のとおりである。

```
Level 0:  64 slots × 1 ms granularity   → covers    0 ms –     63 ms
Level 1:  64 slots × 8 ms granularity   → covers   64 ms –    511 ms
Level 2:  64 slots × 64 ms granularity  → covers  512 ms –   4095 ms  (~4s)
Level 3:  64 slots × 512 ms granularity → covers 4096 ms –  32767 ms (~32s)
...continuing through Level 8 (HZ > 100 builds add a 9th level),
each level 8× coarser than the one below, up to roughly 12 days.
```

各tickでホイールは前進し、クロックが進んだ*すべて*のレベルから期限の来たものを回収する。レベル0のタイマーはレベル0から1ms粒度で発火するが、レベル5にあるタイマーは、そのレベルのクロックがそこに到達したときに、レベル5自身のバケットから、レベル5のより粗い粒度で直接発火する。そこに至るまでにレベル間を移動するものは何もない。それこそがこの書き直しの真の要点である。粗いタイマーをまず細かいレベルへ渡すカスケードのステップに頼るのではなく、各レベルが自分のタイマーを自分の分解能で満了させるのである。`timer_list`は引き続き**タイムアウト**のための仕組みである。つまり、カーネルが発火する前にキャンセルすることを想定しているもの（来ないI/O完了、来ないネットワークACK）であり、数ミリ秒遅れても問題にならず、精度よりも安価な挿入・削除の方が重要なものである。これは以下で述べるhrtimerとは異なる役割である。hrtimerは、実際に時間どおりに実行されることが想定されているもののためのものである。

## hrtimer: 高分解能タイマー {#hrtimer-high-resolution-timers}

hrtimerはThomas GleixnerによってLinux 2.6.16で導入された[(LWN)](https://lwn.net/Articles/167897/)。`ktime_t`（ナノ秒の分解能）と、満了時刻順に並んだ赤黒木を使う。最も近い満了時刻が、ハードウェアのタイマー割り込みを設定する。

hrtimerが統一されたサブシステムとして存在する以前、Linuxでの高精度なタイミングはつぎはぎだらけだった。GleixnerとIngo Molnarによる元の設計の発表（「hrtimer」に改名される前の「ktimers」という名前で2005年9月にLKMLに投稿された）は、単一の答えの代わりに積み重なっていたものを列挙することから始まっている。UTIME（Linux 2.0にまでさかのぼるマイクロ秒タイマーで、カンザス大学のリアルタイム研究プロジェクトが別途保守しており、少数のアーキテクチャに限定されていた）に加え、HRT、VST、DTCK、NEWTODがあり、それぞれが1つか2つのアーキテクチャ向けに狭い一部分（こちらではnanosleep、あちらではPOSIXインターバルタイマー）だけを解決しており、どれも汎用的ではなかった。発表自体の言い方はこうである。*"All of those patches have one thing in common. They are restricted to a few architectures and address only single problems of timers and timekeeping."*（訳: これらのパッチにはすべて共通点が1つある。少数のアーキテクチャに限定されており、タイマーと時刻管理の単一の問題にしか対処していない）（[LWNによる元のLKML投稿の写し](https://lwn.net/Articles/152363/)）

この提案はまた、統一への圧力が実際にどこから来たかも明示している。*"The efforts to integrate the High Resolution Timer patches into the -rt tree gave a deep insight into the big picture and initiated the ktimers implementation."*（訳: High Resolution Timerのパッチを-rtツリーに統合する取り組みによって全体像への深い洞察が得られ、それがktimersの実装の発端となった）。精密なタイミングは、汎用カーネルよりも[PREEMPT_RT](../locking/preempt-rt.md)にとってより厳しい要件である。「次の10ms以内のどこか」としか約束できないリアルタイムシステムはリアルタイムではない。そのため、スレッド化IRQの場合と同様に、RTツリーの必要性によって、根底にある問い（「Linuxはアーキテクチャをまたいで時点を精密かつ一様にどう表現するのか？」）は、もう一度その場しのぎで回避されるのではなく、実際に答えを出されることになった。`ktime_t`と以下に示すAPIがその答えであり、設計が公の場で約6か月かけて練り直された後、hrtimerとして2.6.16でメインラインにマージされた。

```c
/* include/linux/hrtimer.h */
struct hrtimer {
    struct timerqueue_node node;   /* 赤黒木のノード、満了時刻を格納する */
    ktime_t _softexpires;          /* 最も早く満了しうる時刻 */
    enum hrtimer_restart (*function)(struct hrtimer *);
    struct hrtimer_clock_base *base;
    u8 state;
    u8 is_rel;    /* 相対時間か？ */
    u8 is_soft;   /* softirqで配送するか？ */
    u8 is_hard;   /* hardirqで配送するか？ */
};
```

### API {#api_1}

```c
#include <linux/hrtimer.h>

struct my_device {
    struct hrtimer timer;
};

static enum hrtimer_restart my_hrtimer_handler(struct hrtimer *timer)
{
    struct my_device *dev = container_of(timer, struct my_device, timer);

    /* 処理を行う */
    do_periodic_work(dev);

    /* 再始動: 10ms進める */
    hrtimer_forward_now(timer, ms_to_ktime(10));
    return HRTIMER_RESTART;

    /* あるいは: 再始動しない */
    /* return HRTIMER_NORESTART; */
}

/* 初期化 */
hrtimer_setup(&dev->timer, my_hrtimer_handler,
              CLOCK_MONOTONIC, HRTIMER_MODE_REL);

/* 開始: 10ms後に発火 */
hrtimer_start(&dev->timer, ms_to_ktime(10), HRTIMER_MODE_REL);

/* 絶対時刻で開始する */
hrtimer_start(&dev->timer, ktime_get() + ms_to_ktime(100),
              HRTIMER_MODE_ABS);

/* キャンセル */
hrtimer_cancel(&dev->timer);         /* キャンセルし、発火中なら待つ */
hrtimer_try_to_cancel(&dev->timer);  /* 発火中でない場合のみキャンセルする */

/* アクティブかどうかを確認する */
if (hrtimer_active(&dev->timer)) { ... }
```

### クロックソース {#clock-sources}

```c
CLOCK_MONOTONIC     /* 常に増加し、settimeofdayの影響を受けない */
CLOCK_REALTIME      /* 壁時計の時刻、飛ぶことがある */
CLOCK_BOOTTIME      /* MONOTONICと同様だがサスペンド時間を含む */
CLOCK_TAI           /* 国際原子時 */
```

ほとんどのカーネルタイマーには`CLOCK_MONOTONIC`を使う。`CLOCK_REALTIME`は、タイマーが壁時計の時刻に追従しなければならない場合にのみ使う。

### 配送モード {#delivery-modes}

```c
/* ソフトモード: HRTIMER_SOFTIRQでコールバック（非RTでのデフォルト） */
HRTIMER_MODE_REL_SOFT

/* ハードモード: hardirqコンテキストでコールバック（低レイテンシだが制約が多い） */
HRTIMER_MODE_REL_HARD

/* ピン留め: タイマーは現在のCPUに留まる */
HRTIMER_MODE_REL_PINNED
```

## ユーザー空間のスリープ: 内部ではhrtimer {#userspace-sleep-hrtimer-under-the-hood}

カーネルの`nanosleep()`システムコールとglibcの`usleep()`はhrtimerで実装されている。プロセスが`nanosleep(10ms)`を呼ぶと:

1. `hrtimer_start()`が10ms後のタイマーを設定する
2. プロセスは`TASK_INTERRUPTIBLE`でスリープする
3. タイマーが発火 → コールバックがプロセスを起床させる
4. プロセスが`nanosleep()`から戻る

`usleep(1)`（1µsのスリープ）の精度は、`CONFIG_HIGH_RES_TIMERS`が設定されているかどうかと、ハードウェアが高分解能タイマーモードをサポートしているかどうかに依存する。

```bash
# 高分解能タイマーが有効かどうかを確認する
cat /sys/devices/system/clocksource/clocksource0/available_clocksource
# tsc hpet acpi_pm

# 現在のクロックソース
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
# tsc
```

## 参考文献 {#further-reading}

- [Softirq](softirq.md) — TIMER_SOFTIRQとHRTIMER_SOFTIRQが実行される場所
- [Workqueue](workqueues.md) — タイマーによって遅延され、スリープが必要な処理向け
- `Documentation/timers/timers-howto.rst` — どのタイマーをいつ使うか
