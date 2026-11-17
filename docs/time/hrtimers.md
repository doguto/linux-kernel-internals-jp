# hrtimers {#hrtimers}

> 高分解能タイマー: ナノ秒精度のカーネルタイマー

## 2つのタイマーシステム {#the-two-timer-systems}

Linuxには2つのタイマー実装がある。

| システム | 精度 | データ構造 | 典型的な用途 |
|--------|-----------|---------------|-------------|
| **timer_list**（従来型） | 約1/HZ（1〜10ms） | CPUごとのハッシュ化されたホイール | タイムアウト検出、ネットワーク再送 |
| **hrtimer** | 約ナノ秒 | CPUごとの赤黒木 | sleep()、オーディオ、ネットワークペーシング、リアルタイム |

従来型のホイールはO(1)だが粒度が粗い。hrtimerはソート済みの赤黒木を使い、最も早く満了するタイマーが木の最小要素にあり、それがハードウェアのclockeventを直接プログラムする。

## struct hrtimer {#struct-hrtimer}

```c
/* include/linux/hrtimer_types.h */
struct hrtimer {
    struct timerqueue_linked_node node;
    struct hrtimer_clock_base   *base;
    bool                        is_queued;
    bool                        is_rel;
    bool                        is_soft;
    bool                        is_hard;
    bool                        is_lazy;
    ktime_t                     _softexpires; /* 最も早い満了時刻 */
    enum hrtimer_restart      (*__private function)(struct hrtimer *); /* コールバック */
};

```

## クロックベース {#clock-bases}

各CPUはクロックIDごとに別々のタイマーキューを持つ。

```c
/* hrtimer_cpu_base.clock_base[] へのインデックス HRTIMER_BASE_* */
HRTIMER_BASE_MONOTONIC        /* CLOCK_MONOTONIC */
HRTIMER_BASE_REALTIME         /* CLOCK_REALTIME */
HRTIMER_BASE_BOOTTIME         /* CLOCK_BOOTTIME（サスペンド時間を含む） */
HRTIMER_BASE_TAI              /* CLOCK_TAI */
HRTIMER_BASE_MONOTONIC_SOFT   /* CLOCK_MONOTONIC、softirqで配送 */
HRTIMER_BASE_REALTIME_SOFT    /* CLOCK_REALTIME、softirqで配送 */
HRTIMER_BASE_BOOTTIME_SOFT    /* CLOCK_BOOTTIME、softirqで配送 */
HRTIMER_BASE_TAI_SOFT         /* CLOCK_TAI、softirqで配送 */
```

hard配送のタイマーはclockevent割り込みのhardirqコンテキストで直接発火する（最低レイテンシ）。soft配送のタイマーはsoftirqコンテキストで発火する（レイテンシはわずかに高いが、hard-IRQレベルでの実行を避けられる）。

## hrtimerの使い方 {#using-hrtimers}

### カーネルドライバの例 {#kernel-driver-example}

```c
#include <linux/hrtimer.h>
#include <linux/ktime.h>

struct mydev {
    struct hrtimer poll_timer;
    /* ... */
};

/* コールバック: タイマー発火時に呼ばれる */
static enum hrtimer_restart mydev_timer_cb(struct hrtimer *timer)
{
    struct mydev *dev = container_of(timer, struct mydev, poll_timer);

    /* 処理を行う */
    mydev_poll(dev);

    /* 今から10ms後に再設定 */
    hrtimer_forward_now(timer, ms_to_ktime(10));
    return HRTIMER_RESTART;  /* 再スケジュール */

    /* あるいは: 停止するなら HRTIMER_NORESTART を返す */
}

static int mydev_probe(struct platform_device *pdev)
{
    struct mydev *dev = devm_kzalloc(&pdev->dev, sizeof(*dev), GFP_KERNEL);

    /* タイマーを初期化し、コールバック関数を設定 */
    hrtimer_setup(&dev->poll_timer, mydev_timer_cb, CLOCK_MONOTONIC, HRTIMER_MODE_REL);

    /* 開始: 今から10ms後に発火 */
    hrtimer_start(&dev->poll_timer, ms_to_ktime(10), HRTIMER_MODE_REL);

    return 0;
}

static void mydev_remove(struct platform_device *pdev)
{
    struct mydev *dev = platform_get_drvdata(pdev);
    hrtimer_cancel(&dev->poll_timer);  /* キャンセルし、コールバックの完了を待つ */
}
```

### タイマーモード {#timer-modes}

```c
/* 絶対: 特定の時刻に発火 */
hrtimer_start(&timer, ktime_set(1735689600, 0), HRTIMER_MODE_ABS);

/* 相対: 今からNナノ秒後に発火 */
hrtimer_start(&timer, ns_to_ktime(5000000), HRTIMER_MODE_REL);

/* 現在のCPUに固定（CPUホットプラグ時に移動しない） */
hrtimer_start(&timer, ns_to_ktime(5000000), HRTIMER_MODE_REL_PINNED);

/* Soft（softirqコンテキスト）: */
hrtimer_start(&timer, ns_to_ktime(5000000), HRTIMER_MODE_REL_SOFT);
```

## 高分解能モード {#high-resolution-mode}

ブート時、カーネルは**低分解能モード**で動作し、タイマー割り込みはHZのレート（毎秒100〜1000回）で発火する。最初のhrtimerデバイスが検出されると、システムは**高分解能モード**に切り替わる。

```c
/* kernel/time/hrtimer.c */
static void hrtimer_switch_to_hres(void)
{
    struct hrtimer_cpu_base *base = this_cpu_ptr(&hrtimer_bases);

    if (tick_init_highres()) {
        pr_warn("Could not switch to high resolution mode on CPU %u\n", base->cpu);
        return;
    }
    base->hres_active = true;
    hrtimer_resolution = HIGH_RES_NSEC;

    tick_setup_sched_timer(true);
    /* 割り込みを「再トリガー」して動作を開始させる */
    retrigger_next_event(NULL);
    hrtimer_schedule_hres_work();
}
```

切り替え後は次のようになる。

- スケジューラティックはhrtimerで実装される（もはや固定レートの割り込みではない）
- タイマー分解能はハードウェアのレイテンシ（典型的には約100ns）によってのみ制限される
- `NOHZ`（tickless idle）により、CPUがアイドルのときはティックを完全にスキップできる

```bash
dmesg | grep "high resolution"
# Switched to high resolution mode on CPU 0
```

## NOHZ: ティックレス動作 {#nohz-tickless-operation}

`CONFIG_NO_HZ_IDLE`（デフォルト）では、ランキュー上のすべてのタスクがスリープしており、近いうちに満了するhrtimerもない場合、ティックは完全に停止する。

```
CPU goes idle:
  1. Calculate next hrtimer expiry
  2. Program clockevent for that time
  3. Enter C-state
  4. Wake on clockevent (or external IRQ)
  5. Process any expired hrtimers
  6. Resume scheduler
```

`CONFIG_NO_HZ_FULL`（リアルタイム/HPC向け）では、実行中のタスクが1つだけのCPUでもティックが停止する。これにより毎秒約1000回の割り込みというOSノイズが除去される。

```bash
# NOHZの状態を確認
cat /sys/devices/system/cpu/cpu0/cpuidle/state*/name
dmesg | grep nohz

# CPUごとのアイドル時間を見る
cat /proc/stat | awk 'NR>1 && /cpu[0-9]/ {print $1, "idle:", $5}'
```

## タイマースラック {#timer-slack}

`nanosleep`などの呼び出しは、消費電力を削減するために近くのウェイクアップとまとめる（coalesce）ことができる。カーネルは**スラック**、すなわち要求された時刻を過ぎてもよい許容ずれ幅を適用する。

```c
/* ユーザー空間: 現在のスレッドのタイマースラックを設定 */
prctl(PR_SET_TIMERSLACK, 50000 /* ns */);

/* カーネル内: clock_nanosleep はスラックを尊重する */
hrtimer_sleeper_start_expires(&t, HRTIMER_MODE_ABS | HRTIMER_MODE_SOFT);
```

デフォルトのスラックは固定の定数（`init/init_task.c`の`.timer_slack_ns = 50000`）で、50µsに相当する。slack=0に設定するとまとめが無効になる。これは正確なウェイクアップを必要とするリアルタイムタスクに有用である。

```bash
# プロセスのタイマースラックを見る
cat /proc/<pid>/timerslack_ns
```

## hrtimerとtimer_list {#hrtimer-vs-timer_list}

それぞれを使うべき場面:

| ユースケース | 使うもの |
|----------|-----|
| タイムアウト（ネットワーク、ロック、wait_for_completion） | `timer_list` — より安価で、高い精度は不要 |
| 正確な間隔での周期的処理 | `hrtimer` |
| `schedule_timeout` / `msleep` | `timer_list` — `msleep`はタイマーホイールを使う`schedule_timeout`を呼ぶ |
| リアルタイムのオーディオ/ビデオのペーシング | `HRTIMER_MODE_ABS`を指定した`hrtimer` |
| ウォッチドッグタイマー | `timer_list` |

## hrtimerの観察 {#observing-hrtimers}

```bash
# システム内のすべてのアクティブなhrtimer
cat /proc/timer_list

# hrtimer割り込みレイテンシ（タイマーがどれだけ遅れて発火するか）
# リアルタイムレイテンシの測定には cyclictest を使う
cyclictest -p 99 -t 1 -m -n

# perf: タイマーイベント
perf stat -e hrtimer:hrtimer_start,hrtimer:hrtimer_expire_entry sleep 5

# hrtimerの動作をトレースする
echo 1 > /sys/kernel/tracing/events/timer/hrtimer_start/enable
echo 1 > /sys/kernel/tracing/events/timer/hrtimer_expire_entry/enable
cat /sys/kernel/tracing/trace_pipe
# kworker/0:1 [000] hrtimer_start: hrtimer=0xffff... function=tick_sched_timer expires=...
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/hrtimer.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/hrtimer.h) — hrtimerの公開API: `hrtimer_setup()`、`hrtimer_start()`、`hrtimer_cancel()`、`hrtimer_forward_now()`
- [include/linux/hrtimer_defs.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/hrtimer_defs.h) — `struct hrtimer_cpu_base`（`struct hrtimer`については[include/linux/hrtimer_types.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/hrtimer_types.h)を、`HRTIMER_MODE_*`フラグについては[include/linux/hrtimer.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/hrtimer.h)を参照）
- [kernel/time/hrtimer.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/hrtimer.c) — 高分解能タイマーキューの管理、`hrtimer_switch_to_hres()`、赤黒木の満了処理
- [kernel/time/tick-sched.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/tick-sched.c) — `sched_timer` hrtimerによるスケジューラティックのエミュレーションと、NOHZ idle/fullによるティック抑制
- [lib/timerqueue.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/timerqueue.c) — 最も早いタイマーをキャッシュし続ける拡張赤黒木の操作（`timerqueue_add()`、`timerqueue_del()`）

### manページ {#man-pages}

- [`nanosleep(2)`](https://man7.org/linux/man-pages/man2/nanosleep.2.html) — カーネルのhrtimer上に実装された高分解能スリープのシステムコール
- [`clock_nanosleep(2)`](https://man7.org/linux/man-pages/man2/clock_nanosleep.2.html) — `TIMER_ABSTIME`とタイマースラックによるまとめをサポートする、クロック選択可能なスリープ
- [`prctl(2)`](https://man7.org/linux/man-pages/man2/prctl.2.html) — スレッドごとのタイマーまとめのスラックを設定する`PR_SET_TIMERSLACK`と`PR_GET_TIMERSLACK`

### 関連ページ {#related-pages}

- [時刻管理とクロックソース](timekeeping.md) — 時刻管理のアーキテクチャ、クロックID、時刻オフセットの変換
- [POSIXタイマーとtimerfd](posix-timers.md) — カーネルのhrtimerに支えられたユーザー空間のタイマーAPI（`timer_create()`、`timerfd`）
- [タイマーホイール](timer-wheel.md) — 低分解能の`timer_list`ホイールとの比較と選択ガイド
- [EEVDFスケジューラ](../sched/eevdf.md) — 高分解能モードではhrtimerによって駆動されるCPUスケジューラのティック

### LWN記事 {#lwn-articles}

- [The high-resolution timer API](https://lwn.net/Articles/167897/) — Jonathan Corbet、2006年1月: Thomas Gleixnerのhrtimerサブシステムの設計と統合
- [hrtimer: Provide softirq context hrtimers](https://lwn.net/Articles/732536/) — Anna-Maria Gleixner、2017年8月31日: 低レイテンシのRT安全性のため、hardirqとsoftirqのタイマー満了処理を分離する

### 外部資料 {#external}

- [Timekeeping and Timers in Linux](https://docs.kernel.org/core-api/timekeeping.html) — カーネルの時刻管理とタイマー基盤のコアAPIドキュメント
- [High-resolution timers subsystem](https://docs.kernel.org/timers/hrtimers.html) — 高分解能タイマーキューの設計ドキュメントとアーキテクチャ
