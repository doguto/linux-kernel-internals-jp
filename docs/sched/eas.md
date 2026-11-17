# Energy-Aware Scheduling（EAS） {#energy-aware-scheduling-eas}

> 異種CPUトポロジにおけるエネルギー消費の最小化

## 動機 {#motivation}

ARMのbig.LITTLEやIntelのハイブリッド（P+Eコア）CPUでは、コアごとに性能*と*エネルギー特性が異なる。EASはもともとARM big.LITTLE向けに、Quentin Perret（ARM）によってLinux 5.0でマージされた — [`732cd75b8c92`](https://git.kernel.org/linus/732cd75b8c920d3727e69957b14faa7c2d7c3b75)（[パッチシリーズ](https://lore.kernel.org/lkml/20181203095628.11858-1-quentin.perret@arm.com/)、[LWN](https://lwn.net/Articles/772475/)）。Intelのハイブリッド（P+Eコア）のサポートは、Rafael J. WysockiによってLinux 6.16で追加された — [`7b010f9b9061`](https://git.kernel.org/linus/7b010f9b906107ae4e5ac626329ab818b3f0a6b6)。

```
ARM big.LITTLE:
  "big" cores (A76):   high performance, high power
  "LITTLE" cores (A55): low performance, low power

Intel Alder Lake:
  P-cores (Golden Cove): high performance, high power
  E-cores (Gracemont):   lower performance, lower power
```

計算集約的なタスクはbig/Pコアで実行すべきである。アイドル気味のバックグラウンドタスクはLITTLE/Eコアに置いておくべきである。EASは、ハードウェアのエネルギーモデルに基づいてこの判断を行う。

## エネルギーモデル {#energy-model}

エネルギーモデルは、異なる使用率のレベルにおける各CPUの消費電力を記述する:

```c
/* include/linux/energy_model.h */
struct em_perf_state {
    unsigned long frequency;  /* kHz */
    unsigned long power;      /* mW */
    unsigned long cost;       /* power * max_freq / frequency（正規化済み） */
    unsigned long flags;
};

struct em_perf_domain {
    struct em_perf_state   *table;      /* 性能状態（P-state） */
    int                     nr_perf_states;
    unsigned long           flags;
    struct cpumask          cpus;       /* このドメイン内のCPU */
};
```

エネルギーモデルは、cpufreqドライバまたはデバイスツリーによって登録される:

```c
/* drivers/cpufreq/cpufreq-dt.c（またはARMのファームウェア） */
em_dev_register_perf_domain(cpu_dev, nr_states, &em_cb, cpumask, milliwatts);

/* em_cb.active_power(dev, freq, power) — 指定周波数での電力 */
/* em_cb.get_cost(dev, freq, cost) — 任意: コストを事前計算する */
```

## CPU能力 {#cpu-capacity}

EASは、各CPUの最大計算能力（1024に正規化）をモデル化する:

```c
/* 最大能力: 最速のCPU = 1024 */
/* 各CPUは最大周波数に基づくcapacity_origを持つ */

/* arch/arm64/kernel/topology.c */
static void update_cpu_capacity(unsigned int cpu)
{
    unsigned long capacity = SCHED_CAPACITY_SCALE;  /* 1024 */

    /* システム内の最大CPUに対する周波数の比でスケールする */
    capacity *= per_cpu(cpu_scale, cpu);
    capacity >>= SCHED_CAPACITY_SHIFT;

    topology_set_cpu_scale(cpu, capacity);
}
```

1.0 GHzで動作するLITTLEコアと2.5 GHzのbigコアなら、能力はそれぞれ400と1024といったところになる。

## find_energy_efficient_cpu {#find_energy_efficient_cpu}

EASの中核となる関数は、システム全体のエネルギーを最小化するCPUを選ぶ:

```c
/* kernel/sched/fair.c */
static int find_energy_efficient_cpu(struct task_struct *p, int prev_cpu)
{
    struct cpumask *pd_cpus;
    struct em_perf_domain *pd;
    long prev_delta = LONG_MAX, best_delta = LONG_MAX;
    unsigned long p_util_cfs = task_util_est(p);
    int best_energy_cpu = prev_cpu;

    /* 各性能ドメイン（周波数を共有するCPUのグループ）について: */
    for_each_perf_domain(pd) {
        pd_cpus = em_span_cpus(pd);

        if (!cpumask_intersects(sched_domain_span(sd), pd_cpus))
            continue;

        /* このドメイン内で最もエネルギー効率の良いCPUを探す */
        for_each_cpu_and(cpu, pd_cpus, sched_domain_span(sd)) {
            long energy_delta = compute_energy(pd, cpu, p, prev_cpu);

            /* pをcpuに配置すると総エネルギーが減るか？ */
            if (cpu == prev_cpu)
                prev_delta = energy_delta;
            else if (energy_delta < best_delta) {
                best_delta    = energy_delta;
                best_energy_cpu = cpu;
            }
        }
    }

    /* エネルギーの節約がマイグレーションコストのしきい値（約6%）を超える場合のみ切り替える */
    if ((prev_delta - best_delta) > (prev_delta >> 4))
        return best_energy_cpu;

    return prev_cpu;  /* 前のCPUに留まる */
}
```

## compute_energy {#compute_energy}

タスクを特定のCPUに配置した場合のエネルギーの推定:

```c
/* kernel/sched/fair.c */
static long compute_energy(struct em_perf_domain *pd, int cpu,
                             struct task_struct *p, int prev_cpu)
{
    struct em_perf_state *ps;
    unsigned long util_sum = 0;

    /* このドメインのCPUに載ることになるすべてのタスクの使用率を合計する */
    for_each_cpu(cpu_iter, em_span_cpus(pd)) {
        unsigned long util = cpu_util_next(cpu_iter, p, cpu);
        util_sum += util;
    }

    /* util_sumを維持するのに必要な性能状態（P-state）を探す */
    ps = em_pd_get_efficient_state(pd, util_sum, ..., pd->nr_perf_states);

    /* エネルギー = 電力 * 時間
       正規化: cost = power * max_freq / freq
       energy ≈ cost * util_sum */
    return em_cpu_energy(pd, ps, util_sum, ...);
}
```

## 能力を意識した起床 {#capacity-aware-wakeup}

EASが完全なエネルギー計算を実行する前に、より単純な能力チェックによって、すでに飽和しているCPUにタスクを配置することを防ぐ:

```c
/* システムが過剰使用状態でない場合のみEASを実行する */
static bool sched_energy_enabled(void)
{
    static u64 last_check;
    u64 now;

    /* 過剰使用状態のCPUがあるかチェックする */
    if (READ_ONCE(overutilized))
        return false;

    return true;
}

/* CPUごと: このCPUの能力は十分か？ */
static inline bool cpu_overutilized(int cpu)
{
    return !fits_capacity(cpu_util(cpu), capacity_of(cpu));
}
```

システムが過剰使用状態（いずれかのCPUが100%）のとき、EASは通常のロードバランサにフォールバックする。すでに限界に達しているときには、エネルギーの最適化は役に立たないからである。

## schedutilとEASの相互作用 {#schedutil-eas-interaction}

EASは*どの*CPUかを決め、schedutilは*どの周波数*かを決める:

```
1. EAS: place task on CPU that minimizes total system energy
2. schedutil: set frequency of that CPU based on utilization
   (freq = max_freq * util / capacity)
3. cpufreq driver: select actual P-state or HWP hint
```

```bash
# EASはschedutilガバナが有効な場合にのみ有効になる
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# schedutil  ← EASが有効
# ondemand   ← EASは無効（負荷分散にフォールバックする）
```

## EASを観察する {#observing-eas}

```bash
# EASが有効か確認する
dmesg | grep -i "energy"
# sched: EAS: Energy Model successfully initialized

# CPU能力
cat /sys/devices/system/cpu/cpu*/cpu_capacity
# 512 512 512 512 1024 1024 1024 1024
# （LITTLEコアは512、bigコアは1024）

# ドメインごとのエネルギーモデル
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/time_in_state

# タスクの配置: 各タスクはどの種類のコアを使っているか？
ps aux --sort=-%cpu | head -10
# 次にCPUアフィニティを確認する:
taskset -cp <pid>

# perf: エネルギーのイベント
perf stat -e power/energy-pkg/,power/energy-cores/ sleep 10

# EASの判断をトレースする
echo 1 > /sys/kernel/tracing/events/sched/sched_find_energy_efficient_cpu/enable
cat /sys/kernel/tracing/trace_pipe

# タスクをLITTLEコアのみに強制する（テスト用）
taskset -cp 0-3 <pid>  # CPU 0-3（LITTLEコア）にピン留めする
```

## Linuxにおけるbig.LITTLEのサポート {#biglittle-linux-support}

```bash
# 非対称なCPUトポロジを確認する
cat /sys/devices/system/cpu/cpu*/topology/physical_package_id
cat /sys/devices/system/cpu/cpu*/cpu_capacity

# クラスタ情報のためのARMのsysfs
cat /sys/devices/system/cpu/cpufreq/policy*/cpuinfo_max_freq

# 現在のエネルギーモデル
for cpu in /sys/devices/system/cpu/cpu*/; do
    echo -n "$cpu: capacity "
    cat "$cpu/cpu_capacity" 2>/dev/null || echo "N/A"
done
```

## 熱圧力 {#thermal-pressure}

CPUが熱によってスロットリングされると、その能力は一時的に低下する。EASはこれを考慮に入れる:

```c
/* drivers/base/arch_topology.c */
void arch_update_thermal_pressure(const struct cpumask *cpus,
                                   unsigned long capped_frequency)
{
    unsigned long max_freq, thermal_pressure;

    max_freq = per_cpu(freq_factor, cpu) * arch_max_freq_scale;

    /* thermal_pressure = スロットリングによる能力の損失 */
    thermal_pressure = max_freq - capped_frequency;
    thermal_pressure = div_u64(thermal_pressure << SCHED_CAPACITY_SHIFT,
                                max_freq);

    /* 実効能力を下げる */
    WRITE_ONCE(per_cpu(thermal_pressure, cpu), thermal_pressure);
}
```

```bash
# CPUごとの現在の熱圧力
cat /sys/devices/system/cpu/cpu*/thermal_throttle/throttle_count
```

## 参考文献 {#further-reading}

- [負荷分散](load-balancing.md) — CPUをまたぐタスクのマイグレーション
- [cpufreq: P-state](../power/cpufreq.md) — 周波数の選択、schedutil
- [スケジューラ: EEVDF](eevdf.md) — CPUごとのスケジューリングポリシー
- [スケジューラドメイン](sched-domains.md) — EASのためのトポロジの構築
- `kernel/sched/fair.c` — `find_energy_efficient_cpu`
- カーネルツリー内の`Documentation/scheduler/sched-energy.rst`
