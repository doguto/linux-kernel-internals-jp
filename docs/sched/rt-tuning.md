# リアルタイムLinuxチューニングガイド {#real-time-linux-tuning-guide}

> PREEMPT_RTで決定論的な低レイテンシを実現する: CPU分離、メモリロック、IRQ管理

このカーネルオプションが、20年にわたるツリー外のパッチセットからLinux 6.12で完全にメインライン入りするまでの経緯については、[PREEMPT_RT: メインラインまでの20年](../locking/preempt-rt.md)を参照。このページは歴史ではなく、その使い方を扱う。

## Linuxにおける「リアルタイム」の意味 {#what-real-time-means-on-linux}

リアルタイムシステムは、平均レイテンシだけでなく**最悪ケースの応答時間**を保証する。PREEMPT_RTを使ったLinuxでは次のことを達成できる:

- `cyclictest`の最悪ケースレイテンシ: **10-100µs**（RTなしでは1-100ms） — [PREEMPT_RTのレイテンシベンチマーク](https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/cyclictest/start)を参照
- 適した用途: オーディオ処理、産業制御、モータードライブ、トレーディングシステム

主要な構成要素:
1. `PREEMPT_RT`カーネル（スリープ可能なspinlock、スレッド化されたIRQ）
2. CPU分離（カーネルスレッドなし、割り込みなし）
3. メモリロック（ページフォールトなし）
4. 高優先度のSCHED_FIFOタスク
5. CPU周波数スケーリングの無効化

## ステップ1: PREEMPT_RTカーネルをインストールする {#step-1-install-preempt_rt-kernel}

```bash
# 現在のカーネルがRTかを確認する:
uname -v | grep -i preempt
# #1 SMP PREEMPT_RT ... → RTカーネル
# #1 SMP PREEMPT ...    → 非RTの完全プリエンプション

# Debian/Ubuntu: RTカーネルをインストールする
apt-get install linux-image-rt-amd64
# あるいはCONFIG_PREEMPT_RT=yでソースからビルドする

# RT機能を確認する:
zcat /proc/config.gz | grep -E "PREEMPT_RT|PREEMPT_LAZY"
# CONFIG_PREEMPT_RT=y

# メインラインに入っていないRTパッチを確認する:
# 古いカーネル向け: https://wiki.linuxfoundation.org/realtime/start
```

## ステップ2: ブートパラメータによるCPU分離 {#step-2-cpu-isolation-with-boot-parameters}

```bash
# /etc/default/grub:
GRUB_CMDLINE_LINUX="isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3 irqaffinity=0,1"
# grubを更新する:
update-grub && reboot

# 再起動後:
# - CPU 2,3: ロードバランサから分離され、tickless
# - CPU 0,1: すべての割り込みとカーネルスレッドを処理する
# - irqaffinity=0,1: IRQはCPU 0,1のみ

# 分離を確認する:
cat /sys/devices/system/cpu/isolated    # 2,3
cat /sys/devices/system/cpu/nohz_full   # 2,3
```

## ステップ3: カーネルスレッドをRT用CPUから移動する {#step-3-move-kernel-threads-off-rt-cpus}

```bash
# 移動可能なすべてのカーネルスレッドをCPU 0,1へ移動する:
for pid in $(ps -eo pid,comm | grep -E "^\s*[0-9]+ \[" | awk '{print $1}'); do
    taskset -p 3 $pid 2>/dev/null  # mask=0x3 = CPU 0,1
done

# あるいはtunaを使う（推奨）:
tuna --cpus=2,3 --isolate   # 分離されていないタスクをすべて移動する

# 確認: CPU 2,3上にカーネルスレッドがないこと
ps -eo pid,psr,comm | awk '$2==2 || $2==3' | grep "\["
# （空であるか、次に移動する[irq/...]のみであるべき）
```

## ステップ4: IRQアフィニティ {#step-4-irq-affinity}

```bash
# すべての割り込みをCPU 0,1へ移動する:
for irq_dir in /proc/irq/*/; do
    irq=$(basename $irq_dir)
    [ "$irq" = "0" ] && continue  # タイマーはCPU 0に残す
    cat $irq_dir/smp_affinity_list 2>/dev/null | grep -q "[23]" && \
        echo "0,1" > $irq_dir/smp_affinity_list 2>/dev/null
done

# irqbalanceがこれを元に戻さないよう停止する:
systemctl stop irqbalance
systemctl disable irqbalance

# 確認:
cat /proc/interrupts | awk 'NR>1{print $1, $NF}' | head -20
```

## ステップ5: メモリロック（ページフォールトを防ぐ） {#step-5-memory-locking-prevent-page-faults}

ページフォールトは予測できない。リアルタイムアプリケーションはすべてのメモリをロックしなければならない:

```c
#include <sys/mman.h>
#include <sched.h>

/* 現在および将来のすべてのページをロックする */
mlockall(MCL_CURRENT | MCL_FUTURE);
/* MCL_CURRENT: 既存のマッピングをロックする */
/* MCL_FUTURE: 将来のすべてのmmap/mallocをロックする */

/* スタックを事前にフォールトさせる */
#define STACK_SIZE (8 * 1024 * 1024)  /* 8MB */
char stack_prefault[STACK_SIZE];
memset(stack_prefault, 0, sizeof(stack_prefault));

/* ヒープを事前にフォールトさせる（mallocはすでに触れたページを使う） */
void *heap = malloc(HEAP_SIZE);
memset(heap, 0, HEAP_SIZE);

/* これで: RT動作中にページフォールトは起きない */
```

```bash
# システム全体: RTタスクに無制限のロックメモリを許可する
# /etc/security/limits.conf:
echo "@realtime - memlock unlimited" >> /etc/security/limits.conf
# あるいはrootのみ（通常はすでに無制限）
```

## ステップ6: SCHED_FIFOタスクの優先度 {#step-6-sched_fifo-task-priority}

```c
#include <sched.h>

/* SCHED_FIFOの優先度80（1-99の範囲）を設定する */
struct sched_param param = { .sched_priority = 80 };
sched_setscheduler(0, SCHED_FIFO, &param);
/* SCHED_FIFO: ブロックするかyieldするまで実行される。SCHED_OTHERをプリエンプトする */
/* SCHED_RR: FIFOに似ているがタイムクォンタムがある（同一優先度内でラウンドロビン） */

/* 分離したCPU 2にピン留めする */
cpu_set_t cpuset;
CPU_ZERO(&cpuset);
CPU_SET(2, &cpuset);
sched_setaffinity(0, sizeof(cpuset), &cpuset);

/* RTのメインループ */
while (1) {
    clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, &next_activation, NULL);

    /* ここでリアルタイム処理を行う */
    do_work();

    /* 次の起動時刻を計算する */
    next_activation.tv_nsec += PERIOD_NS;
    if (next_activation.tv_nsec >= 1000000000LL) {
        next_activation.tv_nsec -= 1000000000LL;
        next_activation.tv_sec++;
    }
}
```

```bash
# シェルからプロセスの優先度を設定する:
chrt -f 80 ./my_rt_program    # SCHED_FIFO 優先度80
chrt -r 80 ./my_rt_program    # SCHED_RR 優先度80
chrt -d --sched-deadline 1000000 --sched-period 10000000 --sched-runtime 500000 ./my_rt_prog

# 確認:
chrt -p <pid>
# scheduling policy: SCHED_FIFO
# scheduling priority: 80
```

## ステップ7: CPU周波数スケーリングを無効化する {#step-7-disable-cpu-frequency-scaling}

CPU周波数の変更はレイテンシを増やす:

```bash
# すべてのCPUでperformanceガバナを設定する:
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo performance > $cpu 2>/dev/null
done

# ターボブーストを無効化する（周波数の急変を起こしうる）:
echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo
# または:
echo 0 > /sys/devices/system/cpu/cpufreq/boost

# 確認:
cat /sys/devices/system/cpu/cpu2/cpufreq/scaling_governor
# performance
cat /proc/cpuinfo | grep MHz | head -4
# すべてのCPUが最大の固定周波数になっている
```

## cyclictestによるレイテンシの測定 {#measuring-latency-with-cyclictest}

`cyclictest`は、スケジューリングレイテンシを測定する標準的なツールである:

```bash
# インストール:
apt install rt-tests

# 基本テスト: 分離したCPU 2で起床レイテンシを測定する
cyclictest --cpu=2 --priority=80 --policy=fifo \
           --interval=1000 --duration=60 \
           --mlockall --threads=1 \
           --histogram=200 --histfile=/tmp/hist.dat

# 出力:
# T: 0 ( 1234) P:80 I:1000 C:  60000 Min:      8 Act:     12 Avg:     11 Max:      87
#                                                      ↑           ↑           ↑
#                                                   min µs      avg µs      max µs

# RTカーネル + 分離: Maxは100µs未満であるべき
# なしの場合: Maxは1msを超えることがある

# ストレステスト: 負荷をかけて最悪ケースを探す
cyclictest --cpu=2 -p80 -i1000 -d60 -m -q &
# 同時に負荷を生成する:
hackbench -l 100000 &
stress-ng --cpu 6 --io 4 --vm 2 --vm-bytes 256M &
```

### レイテンシのヒストグラムを生成する {#generating-latency-histogram}

```bash
# ヒストグラム出力付きで実行する
cyclictest -p80 -m -n -i200 -l1000000 --histogram=200 > /tmp/histogram.dat

# gnuplotでプロットする:
cat << 'EOF' > plot.gp
set terminal png
set output "latency_histogram.png"
set xlabel "Latency (µs)"
set ylabel "Frequency"
plot '/tmp/histogram.dat' using 1:2 with histeps
EOF
gnuplot plot.gp
```

## レイテンシスパイクの診断 {#diagnosing-latency-spikes}

```bash
# プリエンプション無効期間が長いもの（閾値µs超）をトレースする:
echo preemptirqsoff > /sys/kernel/debug/tracing/current_tracer
echo 50 > /sys/kernel/debug/tracing/tracing_thresh  # 閾値50µs
cat /sys/kernel/debug/tracing/trace | head -30

# bpftraceでスケジューリングレイテンシをトレースする:
bpftrace -e '
tracepoint:sched:sched_wakeup
/args->pid == target_pid/
{ @wakeup_ts = nsecs; }

tracepoint:sched:sched_switch
/args->next_pid == target_pid && @wakeup_ts/
{
    @lat_us = hist((nsecs - @wakeup_ts) / 1000);
    delete(@wakeup_ts);
}'

# hwlatdetect: ハードウェア起因のレイテンシ（SMI、BIOS）を検出する
hwlatdetect --duration=60 --threshold=50
# CPUを停止させるSystem Management Interrupt（SMI）を検出する
```

## SMI: リアルタイムの最大の敵 {#smi-the-worst-enemy-of-real-time}

**System Management Interrupt（SMI）**はBIOS/UEFIによって生成され、System Management Mode（SMM）で実行される — OSからは完全に見えず、カーネルが無効化することもできない（[Intel SDM Vol. 3, Chapter 34](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)）:

```bash
# SMIの活動を検出する:
hwlatdetect --duration=60 --threshold=100
# DETECTED: 3 latency spikes above threshold
# Spike: duration=150µs
# This is likely an SMI!

# 回避策: BIOS/UEFIの設定でSMIを無効化する
# （ベンダー固有。多くは"System Management Mode"や"Intel TXT"の項目にある）

# Intel: MSRでSMIの回数を監視できる:
rdmsr -a 0x34  # MSR_SMI_COUNT（CPUごとのSMIカウンタ）
```

## RT設定の完全なチェックリスト {#full-rt-configuration-checklist}

```bash
# RTの前提条件をすべて確認する:
cat << 'EOF' > check_rt.sh
#!/bin/bash
echo "=== Kernel PREEMPT_RT ==="
uname -v | grep -qi preempt_rt && echo "OK: PREEMPT_RT active" || echo "FAIL: Not PREEMPT_RT"

echo "=== CPU Isolation ==="
[ -s /sys/devices/system/cpu/isolated ] && \
    echo "OK: isolated=$(cat /sys/devices/system/cpu/isolated)" || \
    echo "WARN: No CPU isolation"

echo "=== Frequency Scaling ==="
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor 2>/dev/null | \
    grep -q performance && echo "OK: performance governor" || \
    echo "WARN: Not performance governor"

echo "=== IRQBalance ==="
systemctl is-active irqbalance 2>/dev/null | \
    grep -q inactive && echo "OK: irqbalance stopped" || \
    echo "WARN: irqbalance running"

echo "=== Turbo/Boost ==="
[ "$(cat /sys/devices/system/cpu/intel_pstate/no_turbo 2>/dev/null)" = "1" ] && \
    echo "OK: turbo disabled" || echo "WARN: turbo may be enabled"
EOF
bash check_rt.sh
```

## 参考文献 {#further-reading}

- [プリエンプションモデル](preemption.md) — PREEMPT_RTの内部
- [IRQアフィニティとCPU分離](../interrupts/irq-affinity.md) — IRQの管理
- [CPUアフィニティ](cpu-affinity.md) — SCHED_FIFOとタスクのピン留め
- [SCHED_DEADLINE](deadline.md) — ハードリアルタイムのためのEDFスケジューリング
- [RTスケジューラ](rt-scheduler.md) — SCHED_FIFO/SCHED_RRの内部
- [スケジューリングティック](sched-tick.md) — ticklessなCPUのためのNOHZ_FULL
- `rtl/cyclictest` — レイテンシ測定ツール
