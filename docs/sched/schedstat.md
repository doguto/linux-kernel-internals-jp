# スケジューラ統計 {#scheduler-statistics}

> /proc/schedstat、/proc/PID/schedstat、そしてsched_statistics構造体

## schedstatが追跡するもの {#what-schedstat-tracks}

カーネルは`CONFIG_SCHEDSTATS`の下で、タスクごととCPUごとのスケジューリング統計を蓄積する。これにより、タスクがどれだけ待ったか、どれだけスリープしたか、どれくらいの頻度でマイグレーションしたか、そしてロードバランサで何が起きているかがわかる。

schedstatは、スケジューリングの異常、すなわち過大な待ち時間、予期しないマイグレーション、負荷の不均衡を診断するときに最初に見るべき場所である。

## schedstatを有効にする {#enabling-schedstat}

```bash
# 有効かどうかを確認する（4.6以降は実行時に切り替え可能）
cat /proc/sys/kernel/sched_schedstats
# 0 = 無効（デフォルト — スケジューラのオーバーヘッドを約5%節約）
# 1 = 有効

# 実行時に有効にする
echo 1 > /proc/sys/kernel/sched_schedstats

# あるいは schedstats=enable を付けてブートする
```

`CONFIG_SCHEDSTATS`がない場合、フィールドは存在するが常にゼロである。ほとんどの本番カーネルは`CONFIG_SCHEDSTATS`を組み込んでいるが、sysctlによってデフォルトで無効になっている。

## タスクごとの統計: /proc/PID/schedstat {#per-task-statistics-procpidschedstat}

```bash
cat /proc/$PID/schedstat
# スペース区切りの3つのフィールド:
# 1. CPU時間（ns）: CPU上で実行していた合計時間
# 2. 待ち時間（ns）: 実行可能だがCPUを待っていた合計時間
# 3. タイムスライス: タスクがスケジュールされた回数
```

例:

```bash
cat /proc/$$/schedstat
# 1234567890 98765432 1042
# → 1.23秒実行し、98.7ms待ち、1042回スケジュールされた
```

### sched_statistics構造体 {#the-sched_statistics-struct}

これらの統計は`struct sched_entity`（`task_struct`に埋め込まれている）に格納される:

```c
// include/linux/sched.h (CONFIG_SCHEDSTATS)
struct sched_statistics {
    // ランキューでの待ち時間
    u64  wait_start;      // タスクが最後にランキューに入ったときのタイムスタンプ
    u64  wait_max;        // 観測された最大待ち時間
    u64  wait_count;      // タスクがランキューで待った回数
    u64  wait_sum;        // 合計待ち時間（ns）
    u64  iowait_count;    // I/O待ちの回数
    u64  iowait_sum;      // 合計I/O待ち時間（ns）

    // スリープ時間（自発的なブロック）
    u64  sleep_start;
    u64  sleep_max;
    s64  sum_sleep_runtime;

    // ブロック時間（非自発的、例: ページフォールト）
    u64  block_start;
    u64  block_max;
    s64  sum_block_runtime;

    // スケジューリングスライスの統計
    s64  exec_max;        // 1回の実行バーストの最長時間（ns）
    u64  slice_max;       // 使われたスケジューラスライスの最長値

    // マイグレーションの統計
    u64  nr_migrations_cold;            // マイグレーションしたがキャッシュはコールドだった
    u64  nr_failed_migrations_affine;   // アフィニティのためマイグレーションできなかった
    u64  nr_failed_migrations_running;  // ターゲットが実行中だった
    u64  nr_failed_migrations_hot;      // タスクがキャッシュホットだったので放置した
    u64  nr_forced_migrations;          // アフィニティが上書きされた

    // 起床の統計
    u64  nr_wakeups;
    u64  nr_wakeups_sync;     // WF_SYNCによる起床（Producer-Consumer）
    u64  nr_wakeups_migrate;  // 起床がマイグレーションを引き起こした
    u64  nr_wakeups_local;    // ローカルCPU上で起床した
    u64  nr_wakeups_remote;   // リモートCPU上で起床した
    u64  nr_wakeups_affine;   // アフィニティのヒントによりそのCPU上で起床した
};
```

## CPUごとの統計: /proc/schedstat {#per-cpu-statistics-procschedstat}

```bash
cat /proc/schedstat
# version 15
# timestamp 12345678901234
# cpu0 0 0 0 0 0 0 123456789 987654321 42
# cpu1 0 0 0 0 0 0 ...
# domain0 00000003 0 0 0 0 0 0 ...
```

### CPU行のフィールド（v15形式） {#cpu-line-fields-v15-format}

```
cpu<N> yld_count sched_count sched_goidle ttwu_count ttwu_local \
       rq_cpu_time run_delay pcount
```

| フィールド | 説明 |
|-------|-------------|
| `yld_count` | sched_yield()が呼び出された回数 |
| `sched_count` | __schedule()が呼び出された回数 |
| `sched_goidle` | __schedule()がアイドルに入った回数 |
| `ttwu_count` | try_to_wake_up()の呼び出し回数 |
| `ttwu_local` | ローカルCPU上での起床回数 |
| `rq_cpu_time` | ランキュー上の合計時間（ns） |
| `run_delay` | 実行前の合計待ち時間（ns） |
| `pcount` | スケジュールされたタスク数 |

### ドメイン行のフィールド {#domain-line-fields}

各`domain<N>`行は、そのCPU上のそのスケジューリングドメインについてのロードバランサの統計を示す:

```
domain<N> cpumask lb_count[idle] lb_count[busy] lb_count[newly_idle] \
          lb_failed[...] lb_balanced[...] lb_imbalance[...] \
          lb_gained[...] lb_hot_gained[...] lb_nobusyg[...] lb_nobusyq[...]
          alb_count alb_failed alb_pushed
          sbe_count sbe_balanced sbe_pushed
          sbf_count sbf_balanced sbf_pushed
          ttwu_wake_remote ttwu_move_affine ttwu_move_balance
```

これらは負荷分散の判断の背後にある生のカウンタであり、ロードバランサが機能しているかどうかを診断するのに役立つ。

## /proc/PID/sched {#procpidsched}

より詳細なタスクごとの統計（常に利用可能で、schedstatsによって制限されない）:

```bash
cat /proc/$PID/sched
# Task: myprocess (pid: 1234)
# ...
# se.exec_start                    :  1234567890.123456
# se.vruntime                      :         456789.012
# se.sum_exec_runtime              :       12345.678901
# nr_switches                      :               1042
# nr_voluntary_switches            :                998
# nr_involuntary_switches          :                 44
# se.load.weight                   :               1024
# se.runnable_weight               :               1024
# policy                           :                  0
# prio                             :                120
# clock-delta                      :                 42
# mm->numa_scan_seq                :                  0
```

主なフィールド:

| フィールド | 意味 |
|-------|---------|
| `se.vruntime` | 現在の仮想実行時間（ns） |
| `se.sum_exec_runtime` | 消費したCPU時間の合計（ns） |
| `nr_voluntary_switches` | タスク自身が起こしたコンテキストスイッチ（I/O、スリープ） |
| `nr_involuntary_switches` | プリエンプション — スケジューラがタスクをCPUから強制的に降ろした回数 |
| `prio` | 実効優先度（100=RTの最大、120=nice 0、139=nice 19） |

`nr_voluntary_switches`に対して`nr_involuntary_switches`が多い場合、タスクが頻繁にプリエンプトされていることを示唆する。割り当てで許される以上のCPUを使っているか、より高い優先度のタスクと競合している可能性がある。

## 待ち時間の解釈 {#interpreting-wait-time}

```bash
# スケジューラの待ち時間が多い上位タスクを表示するスクリプト
for pid in /proc/[0-9]*/schedstat; do
    task=$(dirname $pid | xargs -I{} cat {}/comm 2>/dev/null)
    read cpu_time wait_time slices < $pid
    echo "$wait_time $task $pid"
done | sort -rn | head -20
```

`cpu_time`に対して`wait_time`が大きいことは、タスクが実行可能なのにCPUを待っていることが多いことを意味する。考えられる原因:

- システムが過負荷である（CPUより多くの実行可能タスクがある）
- タスクの優先度が低い（nice値またはcgroupの重み）
- CPUアフィニティの制限が厳しすぎる

## perfとschedstatを併用する {#perf-and-schedstat-together}

```bash
# ワークロードのスケジューラオーバーヘッドを測定する
perf stat -e context-switches,cpu-migrations,sched:sched_switch ./workload

# 非自発的なコンテキストスイッチ（プリエンプション）を数える
perf stat -e sched:sched_switch -a sleep 5 2>&1 | grep sched_switch
```

## 参考文献 {#further-reading}

- [スケジューラのトレーシング](sched-tracing.md) — スケジューリングをリアルタイムに観察するためのftraceイベント
- [スケジューラのチューニング](sched-tuning.md) — スケジューラの挙動を調整するsysctlのつまみ
- [CFS](cfs.md) — vruntimeと仮想デッドラインがスケジューリングの判断にどう関係するか
- [ランキューとタスク選択](runqueues.md) — これらの統計が更新される場所
