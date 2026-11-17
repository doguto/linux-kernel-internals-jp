# cpufreq: CPU周波数スケーリング {#cpufreq-cpu-frequency-scaling}

> P-state、ガバナ、そしてハードウェア協調型の周波数管理

## なぜCPU周波数スケーリングか {#why-cpu-frequency-scaling}

現代のCPUは、**P-state**（パフォーマンス状態）と呼ばれる、電圧と周波数の異なる動作点で動作できる。周波数が低い = 電圧が低い = 消費電力が少ない。カーネルのcpufreqサブシステムは、現在の負荷に基づいて動作点を選択する。

```
P0 (max turbo): 4.5 GHz, 1.4V  — high throughput, high power
P1 (base):      3.5 GHz, 1.2V  — sustained all-core
P2:             2.5 GHz, 1.0V
...
Pn (minimum):   0.8 GHz, 0.7V  — idle, minimal power
```

## アーキテクチャ {#architecture}

```
cpufreq core (drivers/cpufreq/cpufreq.c)
    │
    ├── Governor (policy): schedutil / ondemand / performance / powersave
    │       decides target frequency based on utilization
    │
    └── Driver (hardware): intel_pstate / acpi-cpufreq / cppc_cpufreq
            programs the actual hardware MSRs / ACPI commands
```

## struct cpufreq_policy {#struct-cpufreq_policy}

```c
/* include/linux/cpufreq.h */
struct cpufreq_policy {
    /* このポリシーを共有するCPU（例: SMTの兄弟スレッドやダイ上の全コア） */
    cpumask_var_t   cpus;
    cpumask_var_t   related_cpus;

    unsigned int    shared_type;   /* CPUFREQ_SHARED_TYPE_ALL/ANY/HW */

    unsigned int    cpu;           /* 管理を担うCPU */
    struct clk     *clk;

    struct cpufreq_cpuinfo cpuinfo; /* 最小/最大周波数、遷移レイテンシ */

    unsigned int    min;     /* 現在のポリシーの最小値（kHz） */
    unsigned int    max;     /* 現在のポリシーの最大値（kHz） */
    unsigned int    cur;     /* 現在の周波数 */
    unsigned int    suspend_freq;

    unsigned int    policy;  /* CPUFREQ_POLICY_PERFORMANCE または _POWERSAVE */
    unsigned int    last_policy;

    struct cpufreq_governor *governor;
    void            *governor_data;

    /* ポインタではなく埋め込み -- QoS制約はインラインで追跡される */
    struct freq_qos_request min_freq_req;
    struct freq_qos_request max_freq_req;
    /* ... 追加のフィールド（統計、kobject、ロック）は省略 ... */
};
```

## ガバナ {#governors}

ガバナはCPU使用率を観測し、目標周波数を選択する。

### schedutil（推奨、4.7以降） {#schedutil-recommended-47}

schedutilは、スケジューラのCPUごとの使用率追跡と統合されている。使用率が変化したときにスケジューラのパスから直接呼び出されるため、ポーリングは不要である。

```c
/* kernel/sched/cpufreq_schedutil.c */
static void sugov_update_single_freq(struct update_util_data *hook, u64 time,
                                     unsigned int flags)
{
    struct sugov_cpu *sg_cpu = container_of(hook, struct sugov_cpu, update_util);
    struct sugov_policy *sg_policy = sg_cpu->sg_policy;
    unsigned int cached_freq = sg_policy->cached_raw_freq;
    unsigned long max_cap;
    unsigned int next_f;

    max_cap = arch_scale_cpu_capacity(sg_cpu->cpu);

    /* 使用率を更新し、iowaitブーストを適用し、レート制限に従う */
    if (!sugov_update_single_common(sg_cpu, time, max_cap, flags))
        return;

    /* 使用率 → 目標周波数への対応付け */
    next_f = get_next_freq(sg_policy, sg_cpu->util, max_cap);

    if (sugov_hold_freq(sg_cpu) && next_f < sg_policy->next_freq &&
        !sg_policy->need_freq_update) {
        next_f = sg_policy->next_freq;
        sg_policy->cached_raw_freq = cached_freq;
    }

    if (!sugov_update_next_freq(sg_policy, time, next_f))
        return;

    if (sg_policy->policy->fast_switch_enabled)
        cpufreq_driver_fast_switch(sg_policy->policy, next_f);
    else
        sugov_deferred_update(sg_policy);  /* sugov kthreadを起こす */
}

/*
 * get_next_freq() - 使用率に比例した目標周波数:
 *   next_freq = C * ref_freq * util / max   （C = 1.25 のマージン）
 */
static unsigned int get_next_freq(struct sugov_policy *sg_policy,
                                  unsigned long util, unsigned long max)
{
    struct cpufreq_policy *policy = sg_policy->policy;
    unsigned int freq = get_capacity_ref_freq(policy);

    freq = map_util_freq(util, freq, max);

    if (freq == sg_policy->cached_raw_freq && !sg_policy->need_freq_update)
        return sg_policy->next_freq;

    sg_policy->cached_raw_freq = freq;
    return cpufreq_driver_resolve_freq(policy, freq);
}
```

### ondemand {#ondemand}

CPUのアイドル時間を定期的にポーリングし、非アイドルの割合に比例して周波数をスケーリングする。サンプリング間隔は固定の10msではない。デフォルトは`max(2 * tick_period, 1.5 * transition_latency)`である（ドライバが遷移レイテンシを報告しない場合は1msにフォールバックする）。`drivers/cpufreq/cpufreq.c`の`cpufreq_policy_transition_delay_us()`と、`drivers/cpufreq/cpufreq_governor.c`で計算されるデフォルト値を参照のこと。実行時にはガバナの`sampling_rate` sysfsファイルで調整できる。

```
target_freq = max_freq * (non_idle_time / sample_time)
```

schedutilより反応が遅い（イベント駆動ではなくポーリングであるため）が、より単純で、今でも広く使われている。

### performance / powersave {#performance-powersave}

静的なガバナである。ポリシー範囲内の最大周波数（performance）または最小周波数（powersave）を常に選択する。使用率のサンプリングは一切行わない。

```bash
# レイテンシが重要なワークロードにはperformanceガバナを使う
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# バッテリー/熱の節約にはpowersaveを使う
echo powersave | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

## cpufreqドライバ {#cpufreq-drivers}

### acpi-cpufreq {#acpi-cpufreq}

ACPIの_PCT（Performance Control）と_PSS（Performance Supported States）オブジェクトを使ってP-stateを列挙し、切り替える。OSはMSR_IA32_PERF_CTLに書き込む。

```c
/* drivers/cpufreq/acpi-cpufreq.c */
static void drv_write(struct acpi_cpufreq_data *data,
                      const struct cpumask *mask, u32 val)
{
    struct acpi_processor_performance *perf = to_perf_data(data);
    struct drv_cmd cmd = {
        .reg = &perf->control_register,
        .val = val,
        .func.write = data->cpu_freq_write,  /* 通常はMSR_IA32_PERF_CTLに書き込む */
    };

    /* @mask内のすべてのCPUでdo_drv_write() -> cmd.func.write()を実行する */
    on_each_cpu_mask(mask, do_drv_write, &cmd, true);
}
```

### intel_pstate {#intel_pstate}

Sandy Bridge以降向けのIntelのネイティブドライバである。HWP（Hardware P-state control、Broadwell以降）を持つCPUでは、従来のガバナ階層をバイパスする。

```
Without HWP: intel_pstate governor → write PERF_CTL MSR every ~10ms
With HWP:    intel_pstate sets HWP_{MIN,MAX,EPP} once
             hardware manages frequency autonomously between min/max
```

```c
/* drivers/cpufreq/intel_pstate.c */
static void intel_pstate_hwp_set(unsigned int cpu)
{
    struct cpudata *cpu_data = all_cpu_data[cpu];
    int max, min;
    u64 value;
    s16 epp;

    max = cpu_data->max_perf_ratio;
    min = cpu_data->min_perf_ratio;
    if (cpu_data->policy == CPUFREQ_POLICY_PERFORMANCE)
        min = max;  /* "performance"ポリシーでは最大値に固定する */

    rdmsrq_on_cpu(cpu, MSR_HWP_REQUEST, &value);
    value &= ~HWP_MIN_PERF(~0L);
    value |= HWP_MIN_PERF(min);
    value &= ~HWP_MAX_PERF(~0L);
    value |= HWP_MAX_PERF(max);

    /* EPP自体はintel_pstate_get_epp() / epp_powersaveから得られ、
     * 上で計算したmin/maxのperf比からではない */
    epp = intel_pstate_get_epp(cpu_data, value);
    /* ... epp を value のビット[31:24]にエンコードしてから: */
    wrmsrq_on_cpu(cpu, MSR_HWP_REQUEST, value);
}
```

ここでは`HWP_DESIRED_PERF`フィールドは設定されない。MSRのdesired-performanceビットを0のままにしておくと、ハードウェアは`HWP_MIN_PERF`と`HWP_MAX_PERF`の間で自律的に選択する。

#### Energy Performance Preference（EPP） {#energy-performance-preference-epp}

EPPは、性能と電力のトレードオフについてハードウェアに与えるヒントである。

| EPP値 | ヒント | 典型的な用途 |
|-----------|------|-------------|
| 0 | performance | レイテンシが重要な場合 |
| 128 | balance_performance | デフォルト |
| 192 | balance_power | |
| 255 | power | バッテリーの節約 |

```bash
# CPU 0のEPPを読み書きする
cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference
# balance_performance

echo performance | sudo tee /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference
```

### ARM / cppc_cpufreq {#arm-cppc_cpufreq}

ARMサーバーは、ファームウェア支援による周波数スケーリングのためにACPI CPPC（Collaborative Processor Performance Control）を使う。HWPと同様に、OSは希望する性能を書き込み、ファームウェアがクロックを調整する。

## 周波数QoS（Quality of Service） {#frequency-qos-quality-of-service}

デバイスやドライバは、周波数QoS APIを通じてCPU周波数に制約を課すことができる。

```c
#include <linux/pm_qos.h>

/* レイテンシに敏感なデバイスのために最小周波数1 GHzを要求する */
struct freq_qos_request req;
freq_qos_add_request(&policy->constraints, &req,
                     FREQ_QOS_MIN, 1000000 /* kHz */);

/* 後で: 制約を取り除く */
freq_qos_remove_request(&req);
```

複数の要求元からの制約は統合される（最も高い最小値 / 最も低い最大値が優先される）。

## C-state: アイドル時の電力 {#c-states-idle-power}

P-stateとは別に、C-stateはCPUのアイドル状態（仕事がないとき）である。

| C-state | 名前 | ウェイクレイテンシ | 節約される電力 |
|---------|------|-------------|-------------|
| C0 | Active | 0 | なし |
| C1 | Halt | ~2µs | 低 |
| C1E | Enhanced halt | ~10µs | 中 |
| C3 | Sleep | ~70µs | 高 |
| C6 | Deep power down | ~85µs | 最高 |
| C10 | Deepest core idle | ~0.9ms | 最大 |

（数値はSkylakeクラスのコアにおける代表的な`intel_idle`の値で、`drivers/idle/intel_idle.c`の`skl_cstates[]`から取ったものである。正確な数値はCPUの世代によって異なる。）

ここでのC10はコアごとのACPI C-stateであり、**Modern Standby**（S0ix / Low Power Idleとも呼ばれる）とは別物であることに注意されたい。Modern Standbyは、すべてのコアとチップセットが条件を満たしていると合意したときに、S3サスペンドの代わりに入る*プラットフォーム全体の*低電力モードである。Modern StandbyはコアごとのC-stateの階段の一部ではなく、独自の、はるかに粒度の粗いレイテンシ予算を持つ。この2つを混同してはならない。

C-stateを選択するのはcpuidleサブシステムである。ガバナ（`menu`または`teo`）は次のウェイクアップ時刻を予測し、許容可能なレイテンシを持つ最も深いC-stateを選ぶ。

```bash
# CPUごとのC-state統計
grep . /sys/devices/system/cpu/cpu0/cpuidle/state*/name
grep . /sys/devices/system/cpu/cpu0/cpuidle/state*/usage
grep . /sys/devices/system/cpu/cpu0/cpuidle/state*/time  # マイクロ秒

# 深いC-stateを防ぐ（レイテンシに敏感なワークロード）
# レイテンシQoSを設定する: 許容される最大の脱出レイテンシ（µs）
#
# /dev/cpu_dma_latencyには10進数の文字列ではなく、*バイナリの*4バイトs32が必要:
# cpu_latency_qos_write()（kernel/power/qos.c）は、書き込みの長さが
# ちょうどsizeof(s32) == 4バイトで「ない」ときにのみ文字列として解析する。
# `echo 100`による"100\n"はちょうど4バイトなので、文字列"100"ではなく
# 生のバイナリ値として再解釈され、100usではなくでたらめなレイテンシになる。
# `sudo tee`も書き込み直後にfdを閉じるため、制約はすぐに破棄される
# （制約はfdが開いている間しか持続しない）。
sudo python3 -c "
import os, struct, time
fd = os.open('/dev/cpu_dma_latency', os.O_WRONLY)
os.write(fd, struct.pack('i', 100))  # binary s32: max 100us wakeup latency
time.sleep(3600)                     # keep the fd open -- closing it reverts the constraint
os.close(fd)
"
```

## cpufreqの観測 {#observing-cpufreq}

```bash
# 現在の周波数（全CPU）
grep MHz /proc/cpuinfo

# 周波数遷移の統計
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/time_in_state
# 800000   12345678   (freq_kHz  time_in_10ms_units)
# 1000000  8765432
# 2400000  1234567

cat /sys/devices/system/cpu/cpu0/cpufreq/stats/total_trans

# turbostat: CPUごとのP-stateとC-stateの分布
sudo turbostat --interval 5

# perf: CPU周波数関連のイベント
perf stat -e power/energy-pkg/,power/energy-cores/ sleep 10

# 周波数遷移をトレースする
echo 1 > /sys/kernel/tracing/events/power/cpu_frequency/enable
cat /sys/kernel/tracing/trace_pipe
# kworker/0:1-42 [000] cpu_frequency: state=2400000 cpu_id=0
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/cpufreq.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/cpufreq.h) — `struct cpufreq_policy`、`struct cpufreq_driver`、およびガバナ登録API
- [drivers/cpufreq/cpufreq.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/cpufreq/cpufreq.c) — cpufreqのコア: ポリシーのライフサイクル、sysfs、ドライバ/ガバナの登録
- [kernel/sched/cpufreq_schedutil.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/cpufreq_schedutil.c) — schedutilガバナ: `sugov_update_single_freq()`、`get_next_freq()`
- [drivers/cpufreq/intel_pstate.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/cpufreq/intel_pstate.c) — Intelのネイティブドライバ。`intel_pstate_hwp_set()`とEPPの処理を含む
- [drivers/cpufreq/acpi-cpufreq.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/cpufreq/acpi-cpufreq.c) — ACPIの`_PSS`/`_PCT`に基づく、`drv_write()`によるP-state切り替え
- [drivers/cpufreq/cppc_cpufreq.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/cpufreq/cppc_cpufreq.c) — ARMサーバーで使われるACPI CPPCドライバ
- [include/linux/pm_qos.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/pm_qos.h) — `freq_qos_add_request()` / `freq_qos_remove_request()`と`struct freq_qos_request`

### 関連ページ {#related-pages}

- [cpuidle: CPUのC-state](cpuidle.md) — cpufreqの稼働時スケーリングに対応する、アイドル時側の仕組み
- [Runtime PM](runtime-pm.md) — デバイスレベルの電源管理
- [システムサスペンド](suspend.md) — システム全体の電源状態
- [スケジューラ: EEVDF](../sched/eevdf.md) — schedutilはスケジューラのCPUごとの追跡から直接使用率を読み取る

### LWNの記事 {#lwn-articles}

- [LWN: Improvements in CPU frequency management](https://lwn.net/Articles/682391/) — schedutilガバナとそのスケジューラ統合を導入した4.7サイクルの作業を扱う

### 外部資料 {#external}

- [Documentation/admin-guide/pm/cpufreq.rst](https://docs.kernel.org/admin-guide/pm/cpufreq.html) — アップストリームのCPUFreqサブシステム管理者ガイド: ガバナ、ドライバ、sysfsインターフェース
- [Documentation/admin-guide/pm/intel_pstate.rst](https://docs.kernel.org/admin-guide/pm/intel_pstate.html) — `intel_pstate`ドライバの詳細: active/passiveモード、HWP、EPP
