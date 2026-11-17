# スケジューラのチューニング {#scheduler-tuning}

> sysctlのつまみ、SCHED_FEATフラグ、よくあるチューニングのパターン

## スケジューラのsysctl {#scheduler-sysctls}

スケジューラのチューニング用のつまみはすべて`/proc/sys/kernel/`の下にある:

```bash
ls /proc/sys/kernel/sched_*
```

### 中核となるタイミングのつまみ {#core-timing-knobs}

```bash
# 最小スケジューリング粒度（ns）
# タスクはプリエンプトされる前に少なくともこの時間は実行される
# デフォルト: 750000（750µs）、CPU数に応じてスケールする
cat /proc/sys/kernel/sched_min_granularity_ns

# 基本スケジューリングスライス（ns） — 周期あたりタスクあたりの目標時間
# EEVDFはこれをタスクごとのデフォルトのスライスとして使う
# デフォルト: 700000（700µs）
cat /proc/sys/kernel/sched_base_slice_ns

# 起床時プリエンプションの粒度（ns）
# 起床したタスクが現在のタスクをプリエンプトするには、これだけ「緊急度」が高くなければならない
# デフォルト: 1000000（1ms）
cat /proc/sys/kernel/sched_wakeup_granularity_ns
```

両者の関係: `sched_latency_ns ≥ nr_runnable × sched_min_granularity_ns`により、実行可能な各タスクがレイテンシ周期ごとに少なくとも1つのスライスを得ることが保証される。

### マイグレーションと負荷分散 {#migration-and-balancing}

```bash
# タスクのマイグレーションコスト（ns）
# タスクがこの時間窓内に現在のCPU上で実行されていれば「キャッシュホット」とみなされ、
# ロードバランサはそのタスクをマイグレーションするのを避ける
# デフォルト: 500000（500µs）
cat /proc/sys/kernel/sched_migration_cost_ns

# 1回の負荷分散呼び出しでマイグレーションするタスクの最大数
# デフォルト: 32
cat /proc/sys/kernel/sched_nr_migrate

# CFS帯域スライス（µs）
# CPUがグローバルな帯域プールから一度に取得するクォータの量
# デフォルト: 5000（5ms）
cat /proc/sys/kernel/sched_cfs_bandwidth_slice_us
```

### RTスロットリング {#rt-throttling}

```bash
# RTタスクの周期（µs）: RTタスクのクォータが補充される間隔
# デフォルト: 1000000（1秒）
cat /proc/sys/kernel/sched_rt_period_us

# RTタスクの実行時間（µs）: 周期のうちRTタスクが消費できる量
# デフォルト: 950000（950ms = 周期の95%）
# -1 = 無制限（RTスロットリングを無効にする — 危険）
cat /proc/sys/kernel/sched_rt_runtime_us
```

### schedstat {#schedstat}

```bash
# タスクごとのスケジューリング統計を有効にする（わずかなオーバーヘッドあり）
# デフォルト: 0（無効）
echo 1 > /proc/sys/kernel/sched_schedstats
```

## SCHED_FEAT: コンパイル時の機能トグル {#sched_feat-compile-time-feature-toggles}

`SCHED_FEAT`フラグは、スケジューラの特定の挙動を制御する真偽値である。`CONFIG_SCHED_DEBUG`を有効にしたカーネルでは、実行時に切り替えられる:

```bash
# すべての機能とその現在の状態を表示する
cat /sys/kernel/debug/sched/features
# PLACE_LAG PLACE_DEADLINE_INITIAL PLACE_REL_DEADLINE RUN_TO_PARITY
# PREEMPT_SHORT PICK_BUDDY CACHE_HOT_BUDDY DELAY_DEQUEUE DELAY_ZERO
# WAKEUP_PREEMPTION ...

# 機能を切り替える
echo NO_WAKEUP_PREEMPTION > /sys/kernel/debug/sched/features  # 無効にする
echo WAKEUP_PREEMPTION > /sys/kernel/debug/sched/features      # 有効にする
```

主な機能:

| 機能 | デフォルト | 効果 |
|---------|---------|--------|
| `WAKEUP_PREEMPTION` | on | 起床したタスクのデッドラインが早ければ、現在のタスクをプリエンプトできる |
| `PLACE_LAG` | on | 起床したタスクのvruntimeを、そのlag（どれだけCPUの借りがあるか）に基づいて調整する |
| `PLACE_DEADLINE_INITIAL` | on | 新しくエンキューされたタスクに初期のデッドライン上の優位を与える |
| `CACHE_HOT_BUDDY` | on | キャッシュホットなタスクを、実行し続けているというだけでプリエンプトしない |
| `NEXT_BUDDY` | off | プリエンプションの後、直前に起床したタスクを実行しようとする |
| `HRTICK` | off | タイムスライスの満了時に正確にプリエンプトするためにhrtimerを使う（より正確だが、オーバーヘッドがやや高い） |
| `TTWU_QUEUE` | arch | CPUをまたぐ起床をキューに入れる（IPIの嵐を減らす） |
| `RT_RUNTIME_SHARE` | on | RTタスクが兄弟CPUから実行時間を借りることを許可する |

## よくあるチューニングのシナリオ {#common-tuning-scenarios}

### レイテンシに敏感なワークロード（ネットワーク、トレーディング、オーディオ） {#latency-sensitive-workloads-network-trading-audio}

```bash
# 最小粒度を小さくする: タスクのスライスが小さくなり、最大レイテンシが下がる
echo 500000 > /proc/sys/kernel/sched_min_granularity_ns     # 500µs（デフォルト750µs）
echo 700000 > /proc/sys/kernel/sched_base_slice_ns           # デフォルトのまま

# 起床粒度を小さくする: 起床したタスクがより積極的にプリエンプトする
echo 500000 > /proc/sys/kernel/sched_wakeup_granularity_ns  # 500µs（デフォルト1ms）

# マイグレーションコストを小さくする: マイグレーションを増やす（キャッシュホットの保護を弱める）
echo 250000 > /proc/sys/kernel/sched_migration_cost_ns      # 250µs（デフォルト500µs）
```

### スループット重視のワークロード（バッチ処理、HPC） {#throughput-oriented-workloads-batch-processing-hpc}

```bash
# 最小粒度を大きくする: タスクはプリエンプトされるまでより長く実行される
echo 4000000 > /proc/sys/kernel/sched_min_granularity_ns   # 4ms

# マイグレーションコストを大きくする: タスクをキャッシュホットに保ち、マイグレーションを減らす
echo 5000000 > /proc/sys/kernel/sched_migration_cost_ns    # 5ms

# 負荷分散1回あたりにマイグレーションできるタスクを増やす
echo 64 > /proc/sys/kernel/sched_nr_migrate
```

### リアルタイムのワークロード（SCHED_FIFO/RRタスク） {#real-time-workloads-sched_fiforr-tasks}

```bash
# RTタスクがCFSより優先されることを確認する（デフォルト: 95%）
cat /proc/sys/kernel/sched_rt_runtime_us  # 950000のはず

# 隔離されたRTコアの場合（cpusetによる隔離と組み合わせる）:
# 専用コアでRTスロットリングを無効にする — RTタスクには保証された時間が必要
# 警告: RTタスクが行儀よく動作し、CPUが完全に専用である場合にのみ安全
echo -1 > /proc/sys/kernel/sched_rt_runtime_us  # スロットリングをグローバルに無効にする

# より良い方法: cgroupごとのRT制限を使う
echo 950000 > /sys/fs/cgroup/rt_group/cpu.rt_runtime_us   # 実行時間（µs）
echo 1000000 > /sys/fs/cgroup/rt_group/cpu.rt_period_us   # 周期（µs）
```

### コンテナ/cgroupによる隔離 {#containercgroup-isolation}

```bash
# コンテナに2 CPUを与える（クォータ200ms / 周期100ms）
echo "200000 100000" > /sys/fs/cgroup/containers/myapp/cpu.max

# スロットリングの応答を細かくするために帯域スライスを小さくする（ロック競合が増える代償あり）
echo 1000 > /proc/sys/kernel/sched_cfs_bandwidth_slice_us  # 1ms（デフォルト5ms）
```

## RT/低レイテンシのためのCPU隔離 {#cpu-isolation-for-rtlow-latency}

真にレイテンシに敏感なタスクのためには、CPUをカーネルのスケジューラから完全に隔離する:

```bash
# ブートパラメータ: CPU 4-7を一般のスケジューリングから外す
# /etc/default/grubに追加: GRUB_CMDLINE_LINUX="isolcpus=4-7 nohz_full=4-7 rcu_nocbs=4-7"

# 実行時: タスクを隔離されたCPUにピン留めする
taskset -c 4 ./latency_sensitive_app

# 隔離の状態を確認する
cat /sys/devices/system/cpu/isolated  # → 4-7
cat /sys/devices/system/cpu/nohz_full # → 4-7
```

`isolcpus`はCPUをロードバランサから外す。`nohz_full`はそれらのCPU上のスケジューラティックを無効にする（タスクは周期的な割り込みなしに実行される）。`rcu_nocbs`はRCUコールバックを他のCPUにオフロードする。

## チューニングの効果を診断する {#diagnosing-tuning-effects}

```bash
# 前後でスケジューリングレイテンシを測定する
perf sched latency -- sleep 10

# プリエンプション（非自発的なスイッチ）を数える
perf stat -e cs ./workload 2>&1 | grep "context-switches"

# ランキュー長を監視する（大きい = タスクが待っている）
runqlen 10  # BCCツールキットに含まれる。10msごとにサンプリングする

# タスクがスロットリングされているか確認する（CFS帯域）
watch -n 1 'cat /sys/fs/cgroup/myapp/cpu.stat | grep throttled'
```

## 可観測性のチェックリスト {#observability-checklist}

チューニングの前に、ベースラインを確立する:

1. **タスクが長く待っているか？** → `perf sched latency`または`runqlat`
2. **マイグレーションが過剰か？** → `perf stat -e cpu-migrations`または`schedstat`
3. **CFS帯域スロットリングが起きているか？** → `cpu.stat nr_throttled/nr_periods`
4. **RTタスクがCPUを独占しているか？** → RT%が高いことを示す`htop`や`perf top`
5. **NUMAの非効率があるか？** → `numastat`、`perf stat -e cache-misses`

## 参考文献 {#further-reading}

- [スケジューラ統計](schedstat.md) — チューニングの前に何を測定すべきか
- [スケジューラのトレーシング](sched-tracing.md) — リアルタイムの観察ツール
- [CPU帯域制御](cpu-bandwidth.md) — CFSのクォータ/スロットリングのチューニング
- [cpuset](cpuset.md) — 低レイテンシのワークロードのためのCPU隔離
- [EEVDF](eevdf.md) — スライス/デッドラインのパラメータがタスク選択にどう影響するか
