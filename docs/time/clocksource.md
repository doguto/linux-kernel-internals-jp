# clocksourceとclockeventのドライバ {#clocksource-and-clockevent-drivers}

> カーネルにおけるハードウェアタイマー: TSC、HPET、clockevent、tick_device

## 2つのクロック抽象化 {#two-clock-abstractions}

カーネルは時間を2つの概念に分けている。

```
clocksource:   reads current time (monotonic counter)
               "What time is it right now?"
               Example: TSC counter, HPET counter

clockevent:    programs a future interrupt (one-shot or periodic)
               "Wake me up in N nanoseconds"
               Example: Local APIC timer, HPET, ARM generic timer
```

これらが組み合わさってカーネルの時刻管理を実現する。

```
clocksource  →  timekeeping (clock_gettime, ktime_get)
clockevent   →  tick device  →  scheduler tick, hrtimers, timers
```

## struct clocksource {#struct-clocksource}

```c
/* include/linux/clocksource.h */
struct clocksource {
    u64             (*read)(struct clocksource *cs);
    u64             mask;          /* カウンタのラップを扱うためのビットマスク */
    u32             mult;          /* サイクル→ns: ns = cycles * mult >> shift */
    u32             shift;
    u64             max_idle_ns;   /* 精度を失わずに読み取りの間隔を空けられる最大時間 */
    u32             maxadj;
    u64             max_cycles;    /* ns計算がオーバーフローする前の最大サイクル数 */
    u64             max_raw_delta;
    const char      *name;
    struct list_head list;
    u32             freq_khz;
    int             rating;        /* 品質: 400-499 完璧、300-399 望ましい、200-299 良好、100-199 基本、1-99 不適 */
    enum clocksource_ids id;
    enum vdso_clock_mode vdso_clock_mode;
    unsigned long   flags;
    struct clocksource_base *base;

    u64             (*read_snapshot)(struct clocksource *cs, struct clocksource_hw_snapshot *chs);
    int             (*enable)(struct clocksource *cs);
    void            (*disable)(struct clocksource *cs);
    void            (*suspend)(struct clocksource *cs);
    void            (*resume)(struct clocksource *cs);
    void            (*mark_unstable)(struct clocksource *cs);
    void            (*tick_stable)(struct clocksource *cs);
};
```

## TSC（タイムスタンプカウンタ） {#tsc-time-stamp-counter}

TSCは現代のx86における主要なclocksourceである。

```c
/* arch/x86/kernel/tsc.c */
static struct clocksource clocksource_tsc = {
    .name                   = "tsc",
    .rating                 = 300,
    .read                   = read_tsc,
    .mask                   = CLOCKSOURCE_MASK(64),
    .flags                  = CLOCK_SOURCE_IS_CONTINUOUS |
                              CLOCK_SOURCE_CAN_INLINE_READ |
                              CLOCK_SOURCE_MUST_VERIFY |
                              CLOCK_SOURCE_HAS_COUPLED_CLOCK_EVENT,
    .id                     = CSID_X86_TSC,
    .vdso_clock_mode        = VDSO_CLOCKMODE_TSC,
    .enable                 = tsc_cs_enable,
    .resume                 = tsc_resume,
    .mark_unstable          = tsc_cs_mark_unstable,
    .tick_stable            = tsc_cs_tick_stable,
};

static u64 read_tsc(struct clocksource *cs)
{
    return (u64)rdtsc_ordered();  /* RDTSC命令 */
}

/* TSCサイクルをナノ秒に変換する: */
/* ns = (cycles * mult) >> shift */
/* mult と shift はブート時にHPETまたはPITと比べて較正される */
```

```bash
# 現在のクロックソースを確認する:
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
# tsc

# 利用可能なクロックソース（優先度順）:
cat /sys/devices/system/clocksource/clocksource0/available_clocksource
# tsc hpet acpi_pm

# TSC周波数:
dmesg | grep "tsc: Detected"
# tsc: Detected 3600.000 MHz processor

# TSCの安定性:
dmesg | grep tsc | grep -i "unstable\|skew\|reliable"
```

### TSCの信頼性の問題 {#tsc-reliability-issues}

```bash
# TSCが信頼できない場合がある:
# 1. 古いマルチソケットシステム: ソケット間でTSCが同期していない
# 2. CPU周波数スケーリング: TSCのレートが周波数とともに変わる（古いCPU）
# 3. 仮想化: ハイパーバイザが安定したTSCを提供しないことがある

# 現代のCPUは invariant TSC を持つ（Pステートにかかわらず一定レート）:
grep "constant_tsc\|nonstop_tsc" /proc/cpuinfo | head -2
# flags: ... constant_tsc nonstop_tsc ...

# 別のクロックソースを強制する:
echo hpet > /sys/devices/system/clocksource/clocksource0/current_clocksource
```

## clocksourceの登録 {#registering-a-clocksource}

```c
/* 独自のハードウェアカウンタの場合（例: SoCドライバ内）: */
#include <linux/clocksource.h>

static u64 my_timer_read(struct clocksource *cs)
{
    return readl(my_timer_base + TIMER_COUNT_REG);
}

static struct clocksource my_clocksource = {
    .name    = "my-hw-timer",
    .rating  = 200,
    .read    = my_timer_read,
    .mask    = CLOCKSOURCE_MASK(32),
    .flags   = CLOCK_SOURCE_IS_CONTINUOUS,
};

/* probe() 内で: */
clocksource_register_hz(&my_clocksource, 24000000); /* 24 MHz */
/* 周波数から mult/shift を計算する */
```

## struct clock_event_device {#struct-clock_event_device}

```c
/* include/linux/clockchips.h */
struct clock_event_device {
    void (*event_handler)(struct clock_event_device *);
    /* タイマー割り込みごとに呼ばれる（tickレイヤーが設定する） */

    int  (*set_next_event)(unsigned long evt,
                           struct clock_event_device *);
    /* `evt` サイクル後に発火するようハードウェアをプログラムする */

    int  (*set_next_ktime)(ktime_t expires,
                           struct clock_event_device *);

    ktime_t             next_event;
    u64                 max_delta_ns;
    u64                 min_delta_ns;
    u32                 mult;
    u32                 shift;

    enum clock_event_state  state_use_accessors;
    unsigned int        features;
    /* CLOCK_EVT_FEAT_PERIODIC: 周期モードをサポート */
    /* CLOCK_EVT_FEAT_ONESHOT:  ワンショットモードをサポート */
    /* CLOCK_EVT_FEAT_C3STOP:   深いアイドルで停止する */

    const char          *name;
    int                 rating;
    int                 irq;
    int                 bound_on;
    const struct cpumask *cpumask;
    struct list_head    list;
};
```

## Local APICタイマー（CPUごとのclockevent） {#local-apic-timer-per-cpu-clockevent}

```c
/* arch/x86/kernel/apic/apic.c */
static struct clock_event_device lapic_clockevent = {
    .name                   = "lapic",
    .features               = CLOCK_EVT_FEAT_PERIODIC |
                              CLOCK_EVT_FEAT_ONESHOT  |
                              CLOCK_EVT_FEAT_C3STOP   |
                              CLOCK_EVT_FEAT_DUMMY,
    .shift                  = 32,
    .set_state_shutdown     = lapic_timer_shutdown,
    .set_state_periodic     = lapic_timer_set_periodic,
    .set_state_oneshot      = lapic_timer_set_oneshot,
    .set_state_oneshot_stopped = lapic_timer_shutdown,
    .set_next_event         = lapic_next_event,
    .broadcast              = lapic_timer_broadcast,
    .rating                 = 100,
    .irq                    = -1,
};

static int lapic_next_event(unsigned long delta,
                             struct clock_event_device *evt)
{
    /* `delta` サイクル後に発火するようAPICタイマーをプログラムする */
    apic_write(APIC_TMICT, delta);
    return 0;
}
```

## tickデバイスとNOHZ {#tick-device-and-nohz}

各CPUは**tickデバイス**を持つ。これはスケジューリングティックに使われるclockeventである。

```c
/* kernel/time/tick-sched.h */
struct tick_device {
    struct clock_event_device *evtdev;
    enum tick_device_mode      mode;  /* TICKDEV_MODE_PERIODIC または ONESHOT */
};

/* kernel/time/tick-common.c（tick-internal.h で extern 宣言） */
DEFINE_PER_CPU(struct tick_device, tick_cpu_device);
```

```bash
# 周期モードでは: ティックは1/HZ秒ごとに発火する（HZ=250 → 4ms）
grep "^CONFIG_HZ=" /boot/config-$(uname -r)
# CONFIG_HZ=250

# NOHZ（ティックレス）モードでは: CPUがアイドルのときティックが抑制される
# 消費電力を削減し、リアルタイムレイテンシを改善する

# NOHZモードを確認する:
cat /sys/kernel/debug/sched/domains/cpu0/domain0/stats
# あるいは:
cat /proc/timer_list | grep "Tick Device" | head -5

# NOHZ_FULL（適応ティック）: 実行中でもティックを抑制する
# 必要条件: isolcpus + nohz_full= ブートパラメータ（RT/HPC向け）
```

## 時刻管理: clocksource → 壁時計 {#timekeeping-clocksource-wall-clock}

```c
/* include/linux/timekeeper_internal.h */
struct timekeeper {
    struct tk_read_base     tkr_mono;   /* モノトニッククロック（キャッシュライン0） */

    u64                     xtime_sec;  /* 実際の壁時計の秒 */
    unsigned long           ktime_sec;  /* モノトニックの秒 */
    struct timespec64       wall_to_monotonic; /* オフセット */
    ktime_t                 offs_real;  /* モノトニック → リアルタイムのオフセット */
    ktime_t                 offs_boot;  /* ブート時間のオフセット */
    ktime_t                 offs_tai;   /* TAIオフセット */

    struct tk_read_base     tkr_raw;    /* 生のハードウェアクロック（キャッシュライン2、tkr_mono とは隣接しない） */

    unsigned int            clock_was_set_seq;
    u8                      cs_was_changed_seq;
    ktime_t                 next_leap_ktime;
    s32                     tai_offset; /* 秒単位の TAI - UTC。実際の構造体では最後のフィールド */
};

/* 現在時刻の読み取り（seqcountが並行更新から保護する）: */
/* kernel/time/timekeeping.c */
ktime_t ktime_get(void)
{
    struct timekeeper *tk = &tk_core.timekeeper;
    unsigned int seq;
    ktime_t base;
    u64 nsecs;

    do {
        seq = read_seqcount_begin(&tk_core.seq);
        base  = tk->tkr_mono.base;
        nsecs = timekeeping_get_ns(&tk->tkr_mono);
    } while (read_seqcount_retry(&tk_core.seq, seq));

    return ktime_add_ns(base, nsecs);
}
```

## 時刻管理の観察 {#observing-timekeeping}

```bash
# 現在の時刻ソースとオフセット:
cat /proc/timer_list

# NTP同期:
timedatectl status
# System clock synchronized: yes
# NTP service: active
# RTC in local TZ: no

# クロック誤差の推定（NTP）:
adjtimex -p | grep -E "offset|freq|error"

# 高分解能タイマーのベンチマーク:
cyclictest -n -m -p99 -i200 -l1000000
# 測定内容: スリープ要求から実際のウェイクアップまでの時間（hrtimerを使用）

# TSCとHPETの精度比較:
bpftrace -e '
BEGIN { @start = nsecs; }
interval:s:1 {
    printf("elapsed: %lld ns\n", nsecs - @start);
    @start = nsecs;
}'
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/clocksource.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/clocksource.h) — `struct clocksource`、`CLOCK_SOURCE_*`フラグ、周波数からmult/shiftへの変換ヘルパーの定義
- [include/linux/clockchips.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/clockchips.h) — `struct clock_event_device`、`CLOCK_EVT_FEAT_*`機能フラグ、clockeventの状態アクセサ
- [kernel/time/clocksource.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/clocksource.c) — clocksourceの登録、レーティングに基づく選択、`clocksource_watchdog()`による安定性検証の枠組み
- [kernel/time/clockevents.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/clockevents.c) — clockeventのプログラミング、イベントハンドラのディスパッチ、clockeventデバイスの登録
- [kernel/time/tick-sched.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/tick-sched.h) — CPUごとの`struct tick_device`、周期ティックの設定、clockeventとスケジューラティックの間のインターフェース
- [arch/x86/kernel/tsc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/tsc.c) — `clocksource_tsc`、PIT/HPETに対するブート初期の較正、実行時の不安定性検出
- [arch/x86/kernel/apic/apic.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/apic/apic.c) — `lapic_clockevent`、`APIC_TMICT`によるlocal APICタイマーのプログラミング、ブロードキャストの処理

### manページ {#man-pages}

- [`clock_gettime(2)`](https://man7.org/linux/man-pages/man2/clock_gettime.2.html) — ハードウェアclocksourceを読むためのPOSIXシステムコールとvDSO高速パス
- [`adjtimex(2)`](https://man7.org/linux/man-pages/man2/adjtimex.2.html) — 周波数と位相オフセットを調整するためのカーネルのクロック規律インターフェース

### 関連ページ {#related-pages}

- [高分解能タイマー（hrtimers）](hrtimers.md) — clockevent上に構築された高分解能タイマーとスケジューラティックのエミュレーション
- [時刻管理とクロックソース](timekeeping.md) — timekeeperのアーキテクチャ、`ktime_get()`、壁時計のオフセット
- [POSIXタイマーとtimerfd](posix-timers.md) — clockeventとhrtimerに支えられたユーザー向けタイマーインターフェース
- [CPUアイドルとCステート](../power/cpuidle.md) — 動的なティック抑制と、`CLOCK_EVT_FEAT_C3STOP`によるブロードキャストタイマーの管理
- [リアルタイムチューニング](../sched/rt-tuning.md) — 低レイテンシのワークロードのための分離CPUと完全ティックレス（`nohz_full`）動作

### LWN記事 {#lwn-articles}

- [Counting on the time stamp counter](https://lwn.net/Articles/209101/) — Jonathan Corbet、2006年11月: clocksourceとclockeventの抽象化のアーキテクチャ上の導入

### 外部資料 {#external}

- [Timekeeping and Timers in Linux](https://docs.kernel.org/core-api/timekeeping.html) — clocksource、clockevent、timekeeperの各サブシステムを扱うアップストリームのカーネルドキュメント
- [The kernel's command-line parameters](https://docs.kernel.org/admin-guide/kernel-parameters.html) — ブート時のクロックオプション: `clocksource=`、`tsc=reliable`、`nohz=`、`nohz_full=`
