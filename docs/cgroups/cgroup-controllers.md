# cgroup v2 のリソースコントローラ {#cgroup-v2-resource-controllers}

> CPU、メモリ、I/O、プロセス数の制御

## cpu コントローラ {#cpu-controller}

CPU帯域の割り当てを制御する。

### インターフェースファイル {#interface-files}

```bash
# CPU 時間の重み（相対値、デフォルト 100、範囲 1–10000）
cat /sys/fs/cgroup/myapp/cpu.weight
echo 200 > /sys/fs/cgroup/myapp/cpu.weight  # 通常の 2 倍の優先度

# CPU 帯域のクォータ: <quota_us> <period_us>
# "200000 1000000" = CPU 1 個の 20%
# "max 1000000"    = 無制限
cat /sys/fs/cgroup/myapp/cpu.max
echo "200000 1000000" > /sys/fs/cgroup/myapp/cpu.max

# CPU ごとのバースト: 一時的にクォータ超過を許可する（µs 単位の単一値）
echo "100000" > /sys/fs/cgroup/myapp/cpu.max.burst

# CPU の統計
cat /sys/fs/cgroup/myapp/cpu.stat
# usage_usec 1234567        ← 使用した CPU 時間の合計（µs）
# user_usec 987654
# system_usec 246913
# nr_periods 1234           ← 経過した期間の数
# nr_throttled 56           ← スロットリングされた回数
# throttled_usec 12345      ← スロットリングされた時間の合計
```

### cpu.max の内部動作 {#how-cpumax-works-internally}

```c
/* kernel/sched/core.c: スケジューラが帯域を確認する */
struct cfs_bandwidth {
    ktime_t         period;       /* クォータの補充周期 */
    u64             quota;        /* 周期あたりのクォータ（ns） */
    u64             burst;        /* クォータを超えて許可されるバースト */
    u64             runtime;      /* この周期の残り実行時間 */
    u64             runtime_expires;

    short           idle;         /* 待機中のタスクなし */
    short           period_active;
    struct hrtimer  period_timer;  /* 周期ごとに runtime を補充する */
    struct hrtimer  slack_timer;   /* 未使用の runtime を返却する */

    struct list_head throttled_cfs_rq; /* runtime がゼロになった rq */
    int             throttled;
    int             nr_periods;
    int             nr_throttled;
    int             nr_burst;
    u64             burst_time;
    u64             throttled_time;
};
```

cfs_rq がクォータを使い切ると、`throttled_cfs_rq` に追加されてランキューから外され、周期タイマーが発火して `runtime` が補充されるまでそのままになる。

### cpu.weight と EEVDF {#cpuweight-and-eevdf}

`cpu.weight` は EEVDF スケジューラの `se.weight` に対応する。重み200のプロセスは、両方が実行可能なとき、重み100のプロセスの2倍のCPU配分を得る。

```c
/* 変換: cpu.weight → スケジューラの重み */
static u32 sched_weight_from_cgroup(unsigned long cgroup_weight)
{
    /* cgroup は 1–10000、スケジューラは 1–1048576 を使う */
    return scale_load_down(cgroup_weight *
                           (SCHED_WEIGHT_SCALE / CGROUP_WEIGHT_DFL));
}
```

## memory コントローラ {#memory-controller}

メモリ使用量を制御する。アカウンティングのオーバーヘッドのため、最も複雑なコントローラである。

### インターフェースファイル {#interface-files_1}

```bash
# ハードなメモリ制限（超過すると OOM kill）
echo "512M" > /sys/fs/cgroup/myapp/memory.max

# ソフトな制限（メモリ回収の圧力のヒント）
echo "400M" > /sys/fs/cgroup/myapp/memory.high

# OOM 通知: SIGKILL を送るか ENOMEM を返すか
# デフォルトでは、cgroup の OOM はそれを引き起こしたプロセスを kill する

# スワップの制限（スワップ使用量のみ。メモリとは合算しない）
echo "1G" > /sys/fs/cgroup/myapp/memory.swap.max

# 現在の使用量
cat /sys/fs/cgroup/myapp/memory.current    # 使用バイト数
cat /sys/fs/cgroup/myapp/memory.peak       # 過去のピーク

# 詳細な統計（フィールドの説明は mm/memory-stat.md を参照）
cat /sys/fs/cgroup/myapp/memory.stat

# OOM イベント
cat /sys/fs/cgroup/myapp/memory.events
# low 0
# high 12          ← memory.high を超えた回数
# max 0            ← memory.max を超えた回数（リトライあり）
# oom 2            ← 発生した OOM の回数
# oom_kill 3       ← OOM で kill されたプロセス数
# oom_group_kill 0
```

### memory.high: 回収によるスロットリング {#memoryhigh-the-reclaim-throttle}

`memory.high` は、kill する代わりに回収を引き起こすソフトな制限である:

```c
/* mm/memcontrol.c: mem_cgroup_handle_over_high() */
/* 使用量 > memory.high の場合、各ページ割り当ての後に呼ばれる */
void mem_cgroup_handle_over_high(gfp_t gfp_mask)
{
    unsigned long penalty_jiffies;
    unsigned long overage;

    overage = mem_find_max_overage(current->nsproxy->cgroup_ns);
    if (!overage)
        return;

    /* スロットリング: high をどれだけ超えたかに比例してスリープする */
    penalty_jiffies = calculate_high_delay(memcg, nr_pages, overage);
    schedule_timeout_killable(penalty_jiffies);

    /* 回収も試みる */
    mem_cgroup_reclaim(memcg, gfp_mask, 0);
}
```

### ページのアカウンティング {#page-accounting}

各ページは `page_counter` を介して1つの `mem_cgroup` に課金される:

```c
/* include/linux/page_counter.h */
struct page_counter {
    atomic_long_t   usage;       /* 現在の使用量（ページ数） */
    unsigned long   min;         /* memory.min（保護） */
    unsigned long   low;         /* memory.low（保護） */
    unsigned long   high;        /* memory.high（ソフトな制限） */
    unsigned long   max;         /* memory.max（ハードな制限） */
    struct page_counter *parent; /* cgroup 階層における親 */
    unsigned long   watermark;   /* ピーク使用量 */
    unsigned long   failcnt;     /* max に達した回数 */
};
```

ページが割り当てられるとき:
```c
mem_cgroup_charge(page, mm, gfp)
  → get_mem_cgroup_from_mm(mm)      /* この mm が属する memcg を見つける */
  → charge_memcg(memcg, nr_pages)
      → page_counter_try_charge(&memcg->memory, nr_pages, &counter)
      → if over max: try to reclaim → if still over: OOM
```

### memory.min と memory.low: 保護 {#memorymin-and-memorylow-protection}

```bash
# この cgroup のために少なくとも 256MB を確保する（これ以下には決して回収しない）
echo "256M" > /sys/fs/cgroup/myapp/memory.min

# 少なくとも 256MB を維持しようとする（グローバルな圧力下では最後に回収する）
echo "256M" > /sys/fs/cgroup/myapp/memory.low
```

## io コントローラ {#io-controller}

BFQ または mq-deadline スケジューラとの統合により、ブロックI/Oの帯域とIOPSを制御する。

### インターフェースファイル {#interface-files_2}

```bash
# I/O スケジューラ向けの相対的な重み（デフォルト 100、範囲 1–10000）
echo "8:0 200" > /sys/fs/cgroup/myapp/io.weight  # major:minor 重み

# 絶対的な制限: <major>:<minor> rbps=N wbps=N riops=N wiops=N
echo "8:0 rbps=10485760 wbps=5242880" > /sys/fs/cgroup/myapp/io.max
# デバイス 8:0 で読み込み 10MB/s、書き込み 5MB/s

# デバイスごとの I/O 統計
cat /sys/fs/cgroup/myapp/io.stat
# 8:0 rbytes=1048576 wbytes=524288 rios=128 wios=64 dbytes=0 dios=0

# I/O の圧力（PSI）
cat /sys/fs/cgroup/myapp/io.pressure
# some avg10=0.00 avg60=0.00 avg300=0.00 total=0
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

### io.max の仕組み {#how-iomax-works}

io コントローラはブロック層のスロットリングパスを使う:

```c
/* block/blk-cgroup.c: スロットリング用の blkcg_policy_ops */
struct blkcg_policy blkcg_policy_throttl = {
    .pd_alloc_fn  = tg_pd_alloc,
    .pd_init_fn   = tg_pd_init,
    .pd_free_fn   = tg_pd_free,
    /* ... */
};

/* bio が発行されたとき: */
blk_throtl_bio(bio)
  → find tg (throtl_grp) for current task's blkcg
  → check against rbps/wbps/riops/wiops limits
  → if over limit: queue bio in tg, schedule timer to dispatch later
```

## pids コントローラ {#pids-controller}

プロセスとスレッドの総数を制限する。

### インターフェースファイル {#interface-files_3}

```bash
# プロセス/スレッドの合計を 100 に制限する
echo 100 > /sys/fs/cgroup/myapp/pids.max

# 現在の数
cat /sys/fs/cgroup/myapp/pids.current   # 現在の PID 数

# イベント
cat /sys/fs/cgroup/myapp/pids.events
# max 3  ← pids.max によって fork() が拒否された回数
```

### pids.max の強制方法 {#how-pidsmax-is-enforced}

```c
/* kernel/fork.c: copy_process() */
static struct task_struct *copy_process(...)
{
    /* ... */
    retval = cgroup_can_fork(p, args);
    /* pids_can_fork() は次を確認する: */
    if (atomic64_read(&pids_cgrp->counter) >= pids_max)
        return -EAGAIN;  /* fork() は EAGAIN を返す */
    /* ... */
}
```

## cpuset コントローラ {#cpuset-controller}

プロセスを特定のCPUとNUMAノードに固定する:

```bash
# CPU 0-3 のみ許可する
echo "0-3" > /sys/fs/cgroup/myapp/cpuset.cpus

# メモリは NUMA ノード 0 からのみ確保する
echo "0" > /sys/fs/cgroup/myapp/cpuset.mems

# 排他: 他の cgroup はこれらの CPU を使わない
echo 1 > /sys/fs/cgroup/myapp/cpuset.cpus.exclusive
```

## cgroup のリソース使用量の監視 {#monitoring-cgroup-resource-usage}

```bash
# CPU の圧力（PSI — Pressure Stall Information）
cat /sys/fs/cgroup/myapp/cpu.pressure
# some avg10=1.23 avg60=0.45 avg300=0.12 total=12345678
# "some" = 少なくとも 1 つのタスクがストールしている
# "full" = すべてのタスクがストールしている（CPU では表示されない）

# メモリの圧力
cat /sys/fs/cgroup/myapp/memory.pressure

# systemd-cgtop で cgroup の統計を監視する
systemd-cgtop

# cgroupv2 のツールで監視する
cat /sys/fs/cgroup/myapp/cpu.stat
cat /sys/fs/cgroup/myapp/memory.current
```

## rootless コンテナのための委譲 {#delegation-for-rootless-containers}

```bash
# 非特権ユーザーにサブツリーの所有権を与える
chown -R user:user /sys/fs/cgroup/myapp/

# これでユーザーはサブ cgroup を作成し、自分のプロセスを移動できる
# ただし、親で有効になっていないコントローラを有効にすることはできない
```

`nsdelegate` マウントオプションはコンテナからの脱出（breakout）を防ぐ:
```bash
mount -o remount,nsdelegate /sys/fs/cgroup
# これで: namespace 内のタスクは自身のサブツリーしか制御できない
```

## 参考文献 {#further-reading}

- [cgroup v2 のアーキテクチャ](cgroup-v2.md) — 階層とコアのデータ構造
- [メモリ cgroup](../mm/memcg.md) — memcg のアカウンティングの詳細
- [PSI](../mm/psi.md) — Pressure Stall Information
- `Documentation/admin-guide/cgroup-v2.rst` — コントローラの完全なリファレンス
