# 時刻管理とクロックソース {#timekeeping-and-clocksources}

> カーネルはどのように時間を追跡するか: TSC、HPET、クロックソース、NTP

## 2つの抽象化: clocksourceとclockevent {#two-abstractions-clocksource-and-clockevent}

カーネルは時間関連のハードウェアを2つの役割に分ける。

| 抽象化 | 役割 | 例 |
|-------------|------|---------|
| **clocksource** | 経過時間を*読み取る*ための自走カウンタ | TSC、HPET、ACPI PMタイマー |
| **clockevent** | 将来の時刻に割り込みを*発生させる*プログラム可能なデバイス | LAPICタイマー、HPET、ARM generic timer |

clocksourceはポーリングされ、clockeventは割り込みを発生させる。両方が必要である。clocksourceはナノ秒分解能の読み取りを提供し、clockeventはスケジューラティックとhrtimerを駆動する。

## struct clocksource {#struct-clocksource}

```c
/* include/linux/clocksource.h */
struct clocksource {
    u64             (*read)(struct clocksource *cs);  /* ハードウェアカウンタを読む */
    u64             mask;          /* カウンタ幅のビットマスク */
    u32             mult;          /* カウンタ→ナノ秒の乗数 */
    u32             shift;         /* mult の小数部シフト */
    u64             max_idle_ns;   /* 読み取りなしでよい最大時間（ラップ検出用） */
    u32             maxadj;        /* mult の最大調整量 */
    u64             max_cycles;    /* オーバーフロー前のカウンタ差分の最大値 */
    const char     *name;
    int             rating;        /* 品質: 400-499 完璧、300-399 望ましい、200-299 良好、100-199 基本、1-99 不適 */
    /* ... */
};

/* TSCのclocksource（arch/x86/kernel/tsc.c） */
static struct clocksource clocksource_tsc = {
    .name           = "tsc",
    .rating         = 300,
    .read           = read_tsc,   /* rdtsc */
    .mask           = CLOCKSOURCE_MASK(64),
    .flags          = CLOCK_SOURCE_IS_CONTINUOUS | CLOCK_SOURCE_MUST_VERIFY,
    /* mult/shift はブート時に較正される */
};
```

### カウンタ→ナノ秒の変換 {#counter-nanoseconds-conversion}

`mult`と`shift`フィールドにより、ホットパスでの除算を避ける。

```c
/*
 * ns = (cycles * mult) >> shift
 *
 * mult は次を満たすように計算される:
 *   (10^9 * 2^shift) / frequency = mult
 */
static inline s64 clocksource_cyc2ns(u64 cycles, u32 mult, u32 shift)
{
    return ((u64) cycles * mult) >> shift;
}
```

3.6GHzのTSCの場合: `mult ≈ 1165084`、`shift = 22`。`clock_gettime`の各呼び出しは、`rdtsc`1回+乗算1回+シフト1回を行うだけであり、除算もロックもない。

## struct timekeeper {#struct-timekeeper}

timekeeperは現在の時刻状態を保持し、ティックごととNTP調整ごとに更新される。

```c
/* include/linux/timekeeper_internal.h */
struct timekeeper {
    /* キャッシュライン0（timekeeperコアの前置seqcountとともに）: */
    struct tk_read_base    tkr_mono;

    /* キャッシュライン1: */
    u64                    xtime_sec;
    unsigned long          ktime_sec;
    struct timespec64      wall_to_monotonic;
    ktime_t                offs_real;
    ktime_t                offs_boot;
    union {
        ktime_t            offs_tai;
        ktime_t            offs_aux;
    };
    u32                    coarse_nsec;
    enum timekeeper_ids    id;

    /* キャッシュライン2: */
    struct tk_read_base    tkr_raw;
    u64                    raw_sec;

    /* キャッシュライン3と4（時刻管理の内部変数）: */
    enum clocksource_ids   cs_id;
    u32                    cs_ns_to_cyc_mult;
    u32                    cs_ns_to_cyc_shift;
    u64                    cs_ns_to_cyc_maxns;
    unsigned int           clock_was_set_seq;
    u8                     cs_was_changed_seq;
    u8                     clock_valid;

    union {
        struct timespec64  monotonic_to_boot;
        struct timespec64  monotonic_to_aux;
    };

    u64                    cycle_interval;
    u64                    xtime_interval;
    u64                    raw_interval;

    ktime_t                next_leap_ktime;
    u64                    ntp_tick;
    s64                    ntp_error;
    u32                    ntp_error_shift;
    u32                    ntp_err_mult;
    s64                    cs_tick_adj;
    u32                    skip_second_overflow;
    s64                    skew_delta;
    s32                    tai_offset;
};

struct tk_read_base {
    struct clocksource  *clock;
    u64                  mask;
    u64                  cycle_last;   /* 最後に読んだカウンタ値 */
    u32                  mult;         /* 調整済みの mult（NTPがこれを変更する） */
    u32                  shift;
    u64                  xtime_nsec;   /* 累積ナノ秒（小数部） */
    ktime_t              base;         /* ナノ秒のベース */
    u64                  base_real;
};
```

### クロックID {#clock-ids}

| クロックID | 説明 | 影響を受けるもの |
|----------|-------------|------------|
| `CLOCK_REALTIME` | 壁時計（UTC） | settimeofday、NTPによるジャンプ |
| `CLOCK_MONOTONIC` | ブートから単調増加 | NTPによるレート調整のみ。ジャンプしない |
| `CLOCK_MONOTONIC_RAW` | MONOTONICと同様だがNTP調整なし | なし — 純粋なハードウェア |
| `CLOCK_BOOTTIME` | MONOTONICと同様だがサスペンド時間を含む | なし |
| `CLOCK_TAI` | 国際原子時（うるう秒なし） | うるう秒オフセットのみ |
| `CLOCK_PROCESS_CPUTIME_ID` | プロセスのCPU時間 | |
| `CLOCK_THREAD_CPUTIME_ID` | スレッドのCPU時間 | |

```c
struct timespec64 ts;
clock_gettime(CLOCK_MONOTONIC, &ts);  /* ナノ秒精度 */
clock_gettime(CLOCK_REALTIME, &ts);   /* 壁時計 */
```

## TSC: タイムスタンプカウンタ {#tsc-time-stamp-counter}

TSCはx86における主要なclocksourceである。CPUサイクルごとにインクリメントされる64ビットのカウンタである。

### TSCの較正 {#tsc-calibration}

ブート時、カーネルはTSCの周波数を、信頼できる既知の基準（HPET、PIT、ACPI PMタイマー）と比べて較正する。

```c
/* arch/x86/kernel/tsc.c */
static unsigned long pit_calibrate_tsc(u32 latch, unsigned long ms, int loopmin)
{
    u64 tsc, t1, t2, delta;
    unsigned long tscmin, tscmax;

    /* PITチャネル2を既知の間隔にプログラムする */
    outb((inb(0x61) & ~0x02) | 0x01, 0x61);

    /* その間隔におけるTSCのティック数を測る */
    t1 = get_cycles();
    /* ... PITを待つ ... */
    t2 = get_cycles();

    delta = t2 - t1;
    /* delta / ms = kHz単位のTSC周波数 */
    return delta / ms;
}
```

### TSCの不変性 {#tsc-invariance}

現代のCPUは**invariant TSC**（`CPUID[0x80000007] bit 8`）を持ち、PステートやCステートにかかわらず固定レートで動作する。これにより、周波数スケーリングを行うラップトップでもTSCは信頼できるclocksourceとなる。

```bash
# invariant TSC のサポートを確認する
grep "constant_tsc nonstop_tsc" /proc/cpuinfo
```

invariant TSCがなければ、CPU周波数が変化するとTSCがドリフトするため、時刻管理には信頼できない。

## NTP: クロックの調整 {#ntp-adjusting-the-clock}

NTPはユーザー空間（ntpd/chronyd）で動作するが、`adjtimex(2)`を介してカーネルのクロックに補正を適用する。

```c
/* ユーザー空間のNTPデーモンが呼ぶ: */
struct timex tx = {
    .modes  = ADJ_FREQUENCY | ADJ_OFFSET,
    .freq   = ppm_scaled,   /* スケーリングされたPPM単位の周波数誤差 */
    .offset = offset_ns,    /* ナノ秒単位の時刻オフセット */
};
adjtimex(&tx);
```

カーネルは、clocksourceの`mult`値を調整することで周波数補正を適用する。

```
new_mult = orig_mult + (freq_error * orig_mult) / NSEC_PER_SEC
```

これにより、カウンタからナノ秒への変換が、目に見えるジャンプなしにわずかに速く、あるいは遅く進むようになる。

### NTPの位相同期ループ（PLL） {#ntp-phase-locked-loop-pll}

```c
/* kernel/time/ntp.c */
static void ntp_update_frequency(struct ntp_data *ntpdata)
{
    u64 second_length, new_base, tick_usec = (u64)ntpdata->tick_usec;

    second_length        = (u64)(tick_usec * NSEC_PER_USEC * USER_HZ) << NTP_SCALE_SHIFT;

    second_length       += ntpdata->ntp_tick_adj;
    second_length       += ntpdata->cs_tick_adj;
    second_length       += ntpdata->time_freq;

    new_base             = div_u64(second_length, NTP_INTERVAL_FREQ);

    /*
     * 次の second_overflow を待たず、ティック長に変更を
     * 即座に適用する:
     */
    ntpdata->tick_length = new_base;
}
```

## vDSO: システムコールなしの時刻取得 {#vdso-time-without-a-syscall}

`clock_gettime(CLOCK_MONOTONIC)`は最も頻繁に呼ばれるlibc関数である。カーネルはこれを**vDSO**（virtual dynamic shared object）によって最適化している。vDSOはすべてのプロセスにマップされる読み取り専用ページで、時刻管理の高速パスを含む。

```c
/* lib/vdso/gettimeofday.c — システムコールなしでユーザー空間で動作する */
static __always_inline int
do_hres(const struct vdso_time_data *vd, const struct vdso_clock *vc, clockid_t clk, struct __kernel_timespec *ts)
{
    u64 sec, ns;
    u32 seq;

    /* 高分解能部分をコンパイルから除外できるようにする */
    if (!__arch_vdso_hres_capable())
        return false;

    do {
        if (vdso_read_begin_timens(vc, &seq))
            return do_hres_timens(vd, vc, clk, ts);

        if (!vdso_get_timestamp(vd, vc, clk, &sec, &ns))
            return false;
    } while (vdso_read_retry(vc, seq));

    vdso_set_timespec(ts, sec, ns);

    return true;
}
```

vDSOページはティックごとにカーネルによって更新される。ユーザーの読み取りは完全にユーザー空間で行われ、リング遷移もカーネルページへのTLBミスもない。

## 時刻管理の観察 {#observing-timekeeping}

```bash
# 現在のクロックソースと利用可能な代替
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
cat /sys/devices/system/clocksource/clocksource0/available_clocksource

# 別のクロックソースを強制する（テスト/デバッグ用）
echo hpet | sudo tee /sys/devices/system/clocksource/clocksource0/current_clocksource

# TSCの周波数と安定性
dmesg | grep -E "TSC|tsc"
# tsc: Detected 3600.000 MHz processor
# tsc: Detected 3600.000 MHz TSC
# clocksource: tsc: mask: 0xffffffffffffffff max_cycles: 0x...

# NTPの同期状態
timedatectl show | grep NTP
chronyc tracking

# 時間namespace（コンテナは独自のオフセットを持ちうる）
# /proc/<pid>/timens_offsets は MONOTONIC/BOOTTIME のオフセットを示す

# カーネルの時刻のデバッグ
cat /proc/timer_list | head -50
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/timekeeper_internal.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/timekeeper_internal.h) — キャッシュライン配置のコメント付きの`struct timekeeper`と`struct tk_read_base`の定義
- [kernel/time/timekeeping.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/timekeeping.c) — 時刻管理のコア: `ktime_get()`、`timekeeping_advance()`、`timekeeping_update_from_shadow()`、うるう秒の処理
- [kernel/time/timekeeping_internal.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/timekeeping_internal.h) — 時刻管理の内部定義、高速な時刻取得関数、vDSO更新関数のプロトタイプ
- [lib/vdso/gettimeofday.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/vdso/gettimeofday.c) — ロックレスなseqlock読み取りを使う、アーキテクチャ非依存のvDSOクロック読み取りループ（`do_hres()`、`do_coarse()`）
- [arch/x86/kernel/tsc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/tsc.c) — x86のTSC較正（`pit_calibrate_tsc()`、`native_calibrate_tsc()`）と安定性テスト

### manページ {#man-pages}

- [`clock_gettime(2)`](https://man7.org/linux/man-pages/man2/clock_gettime.2.html) — サポートされる各クロックドメインのタイムスタンプを取得する
- [`clock_settime(2)`](https://man7.org/linux/man-pages/man2/clock_settime.2.html) — `CLOCK_REALTIME`を設定し、`clock_was_set()`通知をトリガーする
- [`adjtimex(2)`](https://man7.org/linux/man-pages/man2/adjtimex.2.html) — カーネルのクロックパラメータを調整し、同期状態を取得する
- [`time(7)`](https://man7.org/linux/man-pages/man7/time.7.html) — Linuxにおける時間とタイマーのAPIの概要

### 関連ページ {#related-pages}

- [clocksourceとclockeventのドライバ](clocksource.md) — ハードウェアclocksourceの登録、レーティング、サイクルからnsへの変換
- [NTPとクロック規律](ntp.md) — PLL/FLLによる周波数補正、うるう秒の挿入、11分ごとのRTC同期
- [高分解能タイマー（hrtimers）](hrtimers.md) — timekeeperのベースによって駆動されるナノ秒精度のタイマーキュー
- [時間namespace](time-namespaces.md) — コンテナごとに分離された`CLOCK_MONOTONIC`と`CLOCK_BOOTTIME`のオフセット
- [vDSO高速システムコール](../mm/vdso.md) — 共有データページと、ユーザー空間vDSO高速化のアーキテクチャ

### LWN記事 {#lwn-articles}

- [Introduce CLOCK_REALTIME_COARSE](https://lwn.net/Articles/347811/) — John Stultz、2009年8月: vDSOクロックであるCLOCK_REALTIME_COARSEとCLOCK_MONOTONIC_COARSEの導入
- [The trouble with the TSC](https://lwn.net/Articles/388188/) — Jake Edge、2010年5月: invariant TSCの保証、周波数スケーリングによるドリフト、ハードウェアクロックの較正

### 外部資料 {#external}

- [Timekeeping and Timers in Linux](https://docs.kernel.org/core-api/timekeeping.html) — 時刻管理サブシステムとクロックインターフェースに関する公式カーネルガイド
