# 時間サブシステム {#time-subsystem}

> Linuxカーネルにおける時刻管理、クロックソース、タイマー

## 時間スタック {#the-time-stack}

```
User space
  clock_gettime(CLOCK_MONOTONIC)     timer_create()     timerfd_create()
        │                                  │                   │
        ▼                                  ▼                   ▼
  vDSO (userspace page, no syscall)   POSIX timers        timerfd
        │                                  │                   │
        └──────────────┬───────────────────┘                   │
                       ▼                                       │
             Timekeeping core                                   │
             (kernel/time/timekeeping.c)                        │
             struct timekeeper                                  │
                       │                                        │
             ┌─────────┴──────────┐                            │
             ▼                    ▼                             │
       Clocksource           Clockevent                        │
       (read counter)        (program interrupt)               │
       TSC / HPET            HPET / LAPIC / ARM              hrtimer
                                    │                           │
                              tick_device                  hrtimer_cpu_base
                              tick_sched                        │
                                    │                           │
                             Scheduler tick              High-res callback
                             NOHZ idle                   (on clockevent IRQ)
```

## このセクションのページ {#pages-in-this-section}

| ページ | 扱う内容 |
|------|----------------|
| [時刻管理](timekeeping.md) | クロックソース、TSC、NTP、struct timekeeper、vDSO |
| [hrtimers](hrtimers.md) | 高分解能タイマー、CLOCK_MONOTONIC/REALTIME、タイマースラック |
| [POSIXタイマー](posix-timers.md) | timer_create、clock_nanosleep、timerfd、epollとの統合 |
| [タイマーホイール](timer-wheel.md) | jiffies、struct timer_list、階層型ホイール、遅延可能タイマー |
| [NTPとクロック規律](ntp.md) | adjtimex、PLL/FLL、うるう秒、TAIオフセット、chrony/ntpd |
| [時間namespace](time-namespaces.md) | CLONE_NEWTIME、CLOCK_MONOTONICの分離、CRIUのユースケース |
| [事件簿](war-stories.md) | TSCドリフト、うるう秒のthundering herd、timer_deleteとtimer_delete_syncの競合、jiffiesのラップアラウンド |

## クイックリファレンス {#quick-reference}

```bash
# 利用可能なクロックソース（現在のものは括弧内）
cat /sys/devices/system/clocksource/clocksource0/available_clocksource
# tsc-early tsc hpet acpi_pm

cat /sys/devices/system/clocksource/clocksource0/current_clocksource
# tsc

# TSC周波数
dmesg | grep "TSC.*MHz"
# tsc: Detected 3600.000 MHz processor

# タイマー分解能
clock_getres(CLOCK_MONOTONIC) → 1 ns on most x86 systems

# hrtimerモード
dmesg | grep "hrtimer"
# Switched to high resolution mode on CPU 0

# システム時刻のオフセット（NTP）
timedatectl
# NTP synchronized: yes
```
