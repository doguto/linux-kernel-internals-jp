# cpuidle: CPUのC-stateとアイドル電源管理 {#cpuidle-cpu-c-states-and-idle-power-management}

> ハードウェアのスリープ状態、レイテンシと電力のトレードオフ、そしてcpuidleガバナフレームワーク

## CPUのC-state {#cpu-c-states}

CPUに実行すべき仕事がないとき、CPUは**C-state**（アイドル状態）に入る。各状態は、より深い省電力と引き換えに、より大きなウェイクアップレイテンシを伴う。

```
C-state hierarchy (x86, representative intel_idle/skl_cstates[] values --
exact numbers vary by CPU generation, see drivers/idle/intel_idle.c):
  C0  - Active (running instructions)
  C1  - HALT (stops pipeline, keeps power on)      ~2µs wake
  C1E - C1 + Enhanced (lower voltage)               ~10µs wake
  C3  - SLEEP (LLC may flush, snoop disabled)       ~70µs wake, large savings
  C6  - DEEP POWER DOWN (core voltage off)          ~85µs wake, max savings
  C7s - ENHANCED C6 (LLC flushed)                   ~124µs wake
  C8  - Platform C-state (package + RAM)            ~200µs wake
  C9  - Platform C-state                            ~480µs wake
  C10 - Deepest per-core idle state                 ~890µs (~0.9ms) wake

C10 here is still a per-core ACPI C-state -- not to be confused with
"Modern Standby" (S0ix), a separate platform-wide low-power mode entered
instead of S3 suspend, with its own coarser latency budget.
```

```bash
# 利用可能なC-stateを確認する:
cat /sys/devices/system/cpu/cpu0/cpuidle/state*/name
# POLL
# C1
# C1E
# C3
# C6
# C7s
# C8
# C9
# C10

# C-stateのレイテンシと電力を確認する:
for state in /sys/devices/system/cpu/cpu0/cpuidle/state*; do
    echo -n "$(cat $state/name): "
    echo -n "latency=$(cat $state/latency)µs "
    echo "power=$(cat $state/power)mW"
done
# C1: latency=2µs power=0mW
# C3: latency=70µs power=0mW
# C6: latency=85µs power=0mW
```

## cpuidleフレームワーク {#cpuidle-framework}

Linuxのcpuidleフレームワークは、CPUがアイドルのときにどのC-stateに入るかを選択する。

```c
/* include/linux/cpuidle.h */

/* 1つのC-stateを記述する: */
struct cpuidle_state {
    char        name[CPUIDLE_NAME_LEN];
    char        desc[CPUIDLE_DESC_LEN];

    s64         exit_latency_ns;      /* 最悪ケースのウェイクアップレイテンシ */
    s64         target_residency_ns;  /* 入る価値があるための最小滞在時間 */
    unsigned int flags;               /* CPUIDLE_FLAG_TIMER_STOP など */

    /* 旧来のµs粒度のフィールド -- ほとんどのドライバ（例: intel_idle）は今も
     * これらを設定する。コアはこれらから上記の_nsフィールドを導出する */
    unsigned int exit_latency;        /* 単位はµs */
    int          power_usage;         /* 単位はmW */
    unsigned int target_residency;    /* 単位はµs */

    int (*enter)(struct cpuidle_device *dev,
                 struct cpuidle_driver *drv, int index);
    /* ... enter_dead()、enter_s2idle() もここでは省略 ... */
};

/* CPUごとのアイドルデバイス: */
struct cpuidle_device {
    unsigned int    registered:1;
    unsigned int    enabled:1;
    unsigned int    poll_time_limit:1;
    unsigned int    cpu;
    ktime_t         next_hrtimer;          /* 次のタイマー満了時刻 */
    struct cpuidle_state_usage states_usage[CPUIDLE_STATE_MAX];
    /* states_usage[i].time_ns: 状態iで過ごした合計ns */
    /* states_usage[i].usage:   状態iに入った回数 */
};

/* ドライバ: プラットフォームのすべてのC-stateを記述する: */
struct cpuidle_driver {
    const char          *name;
    struct module       *owner;
    struct cpuidle_state states[CPUIDLE_STATE_MAX];
    int                  state_count;
    int                  safe_state_index; /* 同期（COUPLED）アイドル進入の際に、
                                             * CPUが連動するCPUを待つ間に使う
                                             * 浅い状態。drivers/cpuidle/coupled.c
                                             * を参照 */
};
```

### intel_idleドライバ {#intel_idle-driver}

`intel_idle`ドライバは、レイテンシが既知のIntel固有のC-stateを提供する。

```c
/* drivers/idle/intel_idle.c */
static struct cpuidle_state skl_cstates[] = {
    {
        .name = "C1",
        .desc = "MWAIT 0x00",
        .flags = MWAIT2flg(0x00),
        .exit_latency = 2,
        .target_residency = 2,
        .enter = intel_idle,
    },
    {
        .name = "C1E",
        .desc = "MWAIT 0x01",
        .flags = MWAIT2flg(0x01) | CPUIDLE_FLAG_ALWAYS_ENABLE,
        .exit_latency = 10,
        .target_residency = 20,
        .enter = intel_idle,
    },
    {
        .name = "C3",
        .desc = "MWAIT 0x10",
        .flags = MWAIT2flg(0x10) | CPUIDLE_FLAG_TLB_FLUSHED,
        .exit_latency = 70,
        .target_residency = 100,
        .enter = intel_idle,
    },
    {
        .name = "C6",
        .desc = "MWAIT 0x20",
        .exit_latency = 85,
        .target_residency = 200,
        .enter = intel_idle,
        .flags = MWAIT2flg(0x20) | CPUIDLE_FLAG_TLB_FLUSHED,
    },
    /* ... さらに状態が続く ... */
};

/* MWAIT命令によってC-stateに入る: */
#define flg2MWAIT(flags) (((flags) >> 24) & 0xFF)  /* ヒントは.flagsの最上位バイトに格納される */

static __always_inline int __intel_idle(struct cpuidle_device *dev,
                                        struct cpuidle_driver *drv,
                                        int index, bool irqoff)
{
    struct cpuidle_state *state = &drv->states[index];
    unsigned int eax = flg2MWAIT(state->flags);
    unsigned int ecx = 1 * irqoff; /* 割り込みで抜ける */

    mwait_idle_with_hints(eax, ecx);  /* MONITOR/MWAITを発行する */
    return index;
}

static __cpuidle int intel_idle(struct cpuidle_device *dev,
                                struct cpuidle_driver *drv, int index)
{
    return __intel_idle(dev, drv, index, true);
}
```

## cpuidleガバナ {#cpuidle-governors}

ガバナは、予想されるアイドル時間に基づいてどのC-stateに入るかを決定する。

### ladderガバナ {#ladder-governor}

単純である。現在の状態またはその近くに留まろうとし、最後に入った状態での*実際の*滞在時間を隣接する状態の脱出レイテンシと比較して、上下に移動する（`drivers/cpuidle/governors/ladder.c`）。

```
last_residency = actual_time_in_last_state - exit_latency[last_state]

# promote (go deeper), after PROMOTION_COUNT consecutive qualifying idles:
if last_residency > exit_latency[last_state + 1]:
    → promote to last_state + 1

# demote (go shallower):
if exit_latency[last_state] > latency_req:
    → demote immediately to the deepest state still within latency_req
elif last_residency < exit_latency[last_state]:
    → demote to last_state - 1  (DEMOTION_COUNT == 1, so this is immediate)
```

### menuガバナ（ticklessシステムでのデフォルト） {#menu-governor-default-for-tickless-systems}

次のタイマーイベントと過去のデータを用いてアイドル時間を予測する。現在のメインラインでは、`menu_select()`はもはやiowaitに基づく補正入力を持たない（iowaiterを考慮する古い`which_bucket()`の変種は削除された）。予測は今では純粋にタイマーと履歴によって駆動される。

```c
/* drivers/cpuidle/governors/menu.c */

/* 予測への入力: */
/* 1. 次のタイマーイベントまでの時間（tick_nohz_get_sleep_length()） */
/* 2. 桁ごとの補正係数: EWMAであり、予測時間のみに基づいてwhich_bucket()で
 *    バケット分けされる（桁ごとに6バケット）-- 次のタイマーより先に
 *    起こるウェイクアップ（例: 割り込み）を補正する */
/* 3. 繰り返し間隔検出器: get_typical_interval()は直近8回のアイドル間隔を
 *    調べ、標準偏差が小さければ、次タイマーによる見積もりの代わりに
 *    その平均値を予測として使う */
/* 4. pm_qosのレイテンシ制約（cpuidle_governor_latency_req()）: 時間の予測に
 *    かかわらず、選択できる状態の深さに上限を設ける */

/* 出力: 選択されたC-stateのインデックス */
```

### TEOガバナ（Timer Events Oriented、Linux 5.1以降） {#teo-governor-timer-events-oriented-linux-51}

最近のticklessシステムでは、より正確である。

```bash
# 現在のガバナを確認/設定する:
cat /sys/devices/system/cpu/cpuidle/current_governor
# menu

echo teo > /sys/devices/system/cpu/cpuidle/current_governor
```

## C-stateの無効化 {#disabling-c-states}

レイテンシに敏感なワークロードでは、深いC-stateを無効にする。

```bash
# C6以降を無効にする。上で使った状態の並び
# （state0=POLL, state1=C1, state2=C1E, state3=C3, state4=C6, ...）では、
# C6はstate4である:
for cpu in /sys/devices/system/cpu/cpu*/cpuidle/state[4-9]*/; do
    echo 1 > $cpu/disable 2>/dev/null
done

# 恒久的に: カーネルのブートパラメータに"processor.max_cstate=1"を追加する
# または: "intel_idle.max_cstate=1"

# RTシステム向け: C0/POLL以外のすべてのC-stateを無効にする:
for cpu in /sys/devices/system/cpu/cpu*/cpuidle/state[1-9]*/; do
    echo 1 > $cpu/disable 2>/dev/null
done

# pm_qosのレイテンシ制約を確認する（アプリケーション/ドライバが設定する）:
cat /sys/devices/system/cpu/cpu0/power/pm_qos_resume_latency_us
# 0 = 制約なし、それ以外の値 = 許容できる最大ウェイクアップレイテンシ

# アプリケーション: 深いC-stateを防ぐ（例: オーディオデーモン）。
#
# /dev/cpu_dma_latencyには10進数の文字列ではなく、*バイナリの*4バイトs32が必要:
# cpu_latency_qos_write()（kernel/power/qos.c）は、書き込みの長さが
# ちょうどsizeof(s32) == 4バイトで「ない」ときにのみ文字列として解析する。
# `echo 100`による"100\n"はちょうど4バイトなので、文字列"100"ではなく
# 生のバイナリ値として再解釈され、100usではなくでたらめなレイテンシになる。
# シェルのリダイレクト（や`tee`）は書き込み直後にfdを閉じるため、
# 制約もすぐに破棄される -- 制約はfdが開いている間しか持続しないので、
# 実際のデーモンは自身の生存期間中ずっとfdを開いたままにする:
python3 -c "
import os, struct, time
fd = os.open('/dev/cpu_dma_latency', os.O_WRONLY)
os.write(fd, struct.pack('i', 100))  # binary s32: max 100us wakeup latency
time.sleep(3600)                     # keep the fd open -- closing it reverts the constraint
os.close(fd)
"
```

## 統計と可観測性 {#statistics-and-observability}

```bash
# CPUごと・状態ごとの時間と使用回数:
cat /sys/devices/system/cpu/cpu0/cpuidle/state*/usage
cat /sys/devices/system/cpu/cpu0/cpuidle/state*/time   # µs
cat /sys/devices/system/cpu/cpu0/cpuidle/state*/name

# cpupowerツール:
cpupower idle-info
# CPUidle driver: intel_idle
# CPUidle governor: menu
#
# analyzing CPU 0:
#
# Number of idle states: 9
# Available idle states: POLL C1 C1E C3 C6 C7s C8 C9 C10
# C1:
# Flags/Description: MWAIT 0x00
# Latency: 2
# Residency: 2
# Usage: 8214532
# Duration: 431029841203
# C6:
# Flags/Description: MWAIT 0x20
# Latency: 85
# Residency: 200
# Usage: 5127004
# Duration: 981234501122

cpupower monitor -i 5  # 5秒間のサンプル
# Mperf | C0   Cx   Freq | POLL C1   C1E  C3   C6   C7s  C8   C9   C10
#      0| 25%  75% 3.2G  |   0%   0%   0%   0%   0%  10%  20%  45%   0%

# turbostat: CPUごとのC-state滞在率:
turbostat --interval 1 --show CPU,Busy%,Bzy_MHz,PkgWatt,C1%,C6%,C8%
# CPU   Busy%  Bzy_MHz  PkgWatt  C1%    C6%    C8%
#   0    5.2   3400.0   18.4    2.3    3.1   89.4
#   1    2.1   3400.0            1.2    1.8   94.9

# BPFでcpuidleイベントをトレースする:
bpftrace -e '
tracepoint:power:cpu_idle
{ @[args->state, cpu] = count(); }'
# [state, cpu] → 進入回数

# CPUウェイクアップレイテンシのヒストグラム（アイドル進入から次のイベントまでの時間）:
bpftrace -e '
tracepoint:power:cpu_idle /args->state != 4294967295/
{ @start[cpu] = nsecs; }
tracepoint:power:cpu_idle /args->state == 4294967295 && @start[cpu]/
{
    @lat_us[cpu] = hist((nsecs - @start[cpu]) / 1000);
    delete(@start[cpu]);
}'
```

## 消費電力への影響 {#power-consumption-impact}

```bash
# powertopで電力を計測する:
powertop --auto-tune   # 推奨される電源設定を適用する
powertop --csv=power.csv --time=60  # 60秒間の計測

# RAPL（Running Average Power Limit）のエネルギーカウンタ:
cat /sys/class/powercap/intel-rapl:0/energy_uj       # パッケージのエネルギー
cat /sys/class/powercap/intel-rapl:0:0/energy_uj     # コアのエネルギー
cat /sys/class/powercap/intel-rapl:0:1/energy_uj     # uncoreのエネルギー

# turbostatはパッケージごとの瞬時ワット数を表示する
turbostat --show PkgWatt,PkgTmp --interval 1
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/cpuidle.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/cpuidle.h) — `struct cpuidle_state`、`struct cpuidle_device`、`struct cpuidle_driver`
- [drivers/idle/intel_idle.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/idle/intel_idle.c) — IntelのC-stateドライバ: `skl_cstates[]`テーブルと`intel_idle()`/MWAITによる進入経路
- [drivers/cpuidle/governors/menu.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/cpuidle/governors/menu.c) — `menu`ガバナによるアイドル時間の予測
- [drivers/cpuidle/governors/teo.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/cpuidle/governors/teo.c) — Timer Events Orientedガバナ
- [kernel/sched/idle.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/idle.c) — `cpuidle_idle_call()`。ガバナとドライバを呼び出すアイドルループ

### 関連ページ {#related-pages}

- [cpufreqとP-state](cpufreq.md) — 周波数スケーリング（C-stateとは直交する）
- [リアルタイムチューニング](../sched/rt-tuning.md) — RTタスクのためのCPU分離と周波数スケーリングのチューニング
- [スケジューリングティック](../sched/sched-tick.md) — NOHZとcpuidleの相互作用
- [hrtimer](../time/hrtimers.md) — アイドルを中断させるタイマー

### LWNの記事 {#lwn-articles}

- [LWN: cpuidle: New timer events oriented governor for tickless systems](https://lwn.net/Articles/769571/) — TEOガバナに関するRafael Wysockiの最初のパッチとその根拠。Linux 5.1でマージされた

### 外部資料 {#external}

- [Documentation/admin-guide/pm/cpuidle.rst](https://docs.kernel.org/admin-guide/pm/cpuidle.html) — アップストリームのCPUIdleサブシステム管理者ガイド: ガバナ、ドライバ、`disable`および`pm_qos_resume_latency_us`のsysfsノブ
