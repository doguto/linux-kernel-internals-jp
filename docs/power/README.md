# 電源管理 {#power-management}

> LinuxカーネルにおけるCPU周波数スケーリング、デバイスのランタイム電源管理、システムサスペンド

## 電源管理スタック {#the-power-management-stack}

```
User space (udev, systemd, powertop)
    │
    ▼
sysfs / /sys/devices/.../power/  /sys/devices/system/cpu/cpu*/cpufreq/
    │
    ▼
PM core (drivers/base/power/)
    ├── Runtime PM:  device active/suspended lifecycle
    ├── System PM:   suspend/hibernate state machine
    └── Wakeup:      wakeup_source tracking
    │
    ▼
cpufreq subsystem (drivers/cpufreq/)
    ├── Governor:    schedutil / ondemand / powersave / performance
    └── Driver:      intel_pstate / acpi-cpufreq / cppc_cpufreq
    │
    ▼
Hardware: ACPI P-states, HWP (Hardware P-state), C-states, ARM DVFS
```

## このセクションのページ {#pages-in-this-section}

| ページ | 扱う内容 |
|------|----------------|
| [cpufreq](cpufreq.md) | P-state、ガバナ、schedutil、intel_pstate、HWP |
| [cpuidle](cpuidle.md) | C-state、レイテンシと電力のトレードオフ、menuガバナとTEOガバナ |
| [Runtime PM](runtime-pm.md) | dev_pm_ops、pm_runtime_get/put、使用カウント、autosuspend |
| [システムサスペンド](suspend.md) | suspend-to-RAM、ハイバネート、ウェイクアップソース、PM notifier |
| [熱管理](thermal.md) | サーマルゾーン、トリップポイント、冷却デバイス、step_wise、power_allocator |
| [電源ドメインとgenpd](power-domains.md) | generic_pm_domain、電源レールの依存関係、DTバインディング、debugfs |
| [パワーキャッピングとRAPL](power-capping.md) | Intel RAPL、powercapフレームワーク、MSR、ARM SCMI |
| [事件簿](war-stories.md) | サスペンドのリグレッション、ランタイムPMのデッドロック、サーマルストーム、RAPLの意外な挙動 |

## クイックリファレンス {#quick-reference}

```bash
# 現在のCPU周波数スケーリング
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq

# ガバナを変更する（全CPU）
echo schedutil | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# デバイスのランタイムPM状態
cat /sys/bus/pci/devices/*/power/runtime_status
# active / suspended / error

# 利用可能なシステムサスペンド状態
cat /sys/power/state
# freeze mem disk

# RAMへのサスペンド
echo mem | sudo tee /sys/power/state

# 誰がサスペンドを妨げているか？
cat /sys/kernel/debug/wakeup_sources
```
