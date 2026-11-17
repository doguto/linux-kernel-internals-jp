# ロック競合のデバッグ {#lock-contention-debugging}

> カーネル内のロックのホットスポットを見つけて修正する

## 問題の特定 {#identifying-the-problem}

ロック競合は、CPUが有用な仕事をする代わりに待機に時間を費やすという形で現れる。症状:

- CPU使用率が高いのにスループットが低い
- `perf top`で`_raw_spin_lock`や`mutex_lock`が上位に表示される
- `sar -u`で`%sys`が高いのに、実際の作業量が少ない
- `perf stat`で`lock:contention_begin`イベントが多数観測される

## /proc/lock_stat {#proclock_stat}

`CONFIG_LOCK_STAT=y`が必要である。ロッククラスごとの競合統計を提供する。

```bash
# ロック統計の収集を有効化する
echo 1 > /proc/sys/kernel/lock_stat

# 統計を表示する（> を使うと読み込み時にリセットされる）
cat /proc/lock_stat

# カウンタをリセットする
echo 0 > /proc/lock_stat
```

出力形式:
```
class name    con-bounces  contentions  waittime-min  waittime-max  waittime-total  acq-bounces  acquisitions  holdtime-min  holdtime-max  holdtime-total
&rq->lock:
                   234567       234567          0.20       4567.89    12345678.90       345678      23456789          0.10        234.56    23456789.12
```

主要な列:

- **contentions**: 呼び出し側が待たなければならなかった回数（ロックがすでに保持されていた）
- **waittime-max**: 最悪ケースの待ち時間（µs）。これがテールレイテンシである
- **holdtime-total**: ロックが保持されていた合計時間。高い場合はクリティカルセクションが長いことを意味する
- **con-bounces**: 直前の保持者が別のCPU上にいた競合（CPU間のバウンス）

```bash
# 最も競合の激しいロックを見つける
cat /proc/lock_stat | sort -k2 -rn | head -20

# 待ち時間の長いものに絞る
awk '$5 > 1000' /proc/lock_stat  # waittime-max > 1ms
```

## perf lock {#perf-lock}

`perf lock`はロックイベントをタイムスタンプ付きでトレースする。

```bash
# ロックイベントを記録する（ロックサポート付きのperfが必要）
perf lock record -a sleep 10

# 解析: 最も競合の激しいロックを表示する
perf lock report

# 要約ビュー
perf lock report --key acquired
```

出力:
```
                Name   acquired  contended  avg wait (ns)  total wait (ns)
        &rq->__lock:    1234567      45678          12345      12345678901
  &mm->mmap_lock(W):      23456       1234         234567       1234567890
```

## ロックのホットスポットを探すperf top {#perf-top-for-lock-hotspots}

```bash
# どの関数がロック待ちに最も時間を費やしているかを見つける
perf top -e lock:contention_begin,lock:contention_end

# ロック取得のオーバーヘッドをプロファイルする
perf record -e lock:contention_begin -ag sleep 10
perf report --stdio
```

## bpftraceによるロック解析 {#bpftrace-for-lock-analysis}

```bash
# ロックと呼び出し箇所ごとのロック待ち時間を追跡する
bpftrace -e '
kprobe:mutex_lock {
    @ts[tid] = nsecs;
    @caller[tid] = kstack;
}
kretprobe:mutex_lock /@ts[tid]/ {
    $wait = nsecs - @ts[tid];
    if ($wait > 1000000) {  /* > 1ms */
        printf("Long mutex wait: %lu ns\n  caller:\n%s\n",
               $wait, @caller[tid]);
    }
    delete(@ts[tid]);
    delete(@caller[tid]);
}'

# spinlockの保持時間のヒストグラム
bpftrace -e '
kprobe:_raw_spin_lock {
    @start[tid] = nsecs;
}
kprobe:_raw_spin_unlock /@start[tid]/ {
    @hold_ns = hist(nsecs - @start[tid]);
    delete(@start[tid]);
}'

# どのロックで最も競合が多いか（スピンループ）を見つける
bpftrace -e '
kprobe:__pv_queued_spin_lock_slowpath {
    @[kstack] = count();
}
interval:s:5 {
    print(@); clear(@);
}'
```

## lockdepによる競合の追跡 {#lockdep-contention-tracking}

```bash
# CONFIG_LOCK_STATとCONFIG_LOCKDEPがあれば、/proc/lockdep_statsを確認する
cat /proc/lockdep_stats
# lock-classes: 2341
# direct dependencies: 8234
# chain cache misses: 123
# hardirq-safe locks:  456
# softirq-safe locks:  789
```

## ftrace: 特定のロック箇所をトレースする {#ftrace-tracing-specific-lock-points}

```bash
# すべてのmutex_lock呼び出しをスタック付きでトレースする
echo mutex_lock > /sys/kernel/debug/tracing/set_ftrace_filter
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... 問題を再現する ...
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | head -100
```

## よくあるロック競合のパターンと修正 {#common-lock-contention-patterns-and-fixes}

### 1. 細粒度ロック {#1-fine-grained-locking}

1つの粗いロックを、多数のバケット単位またはオブジェクト単位のロックに置き換える。

```c
/* 変更前: グローバルなロック1つ */
static DEFINE_SPINLOCK(hash_lock);
static struct hlist_head hash_table[HASH_SIZE];

/* 変更後: バケット単位のロック */
struct hash_bucket {
    spinlock_t lock;
    struct hlist_head head;
} ____cacheline_aligned;
static struct hash_bucket hash_table[HASH_SIZE];
```

`____cacheline_aligned`属性は、隣接するバケット間のfalse sharingを防ぐ。

### 2. 読み書きの分離 {#2-read-write-splitting}

アクセスの大半が読み込み専用なら、rwsemかRCUを使う。

```c
/* 変更前: spinlockが読み手と書き手を直列化する */
spin_lock(&config_lock);
val = config->value;
spin_unlock(&config_lock);

/* 変更後: RCUにより並行読み込みが可能になる */
rcu_read_lock();
cfg = rcu_dereference(config);
val = cfg->value;
rcu_read_unlock();
```

### 3. per-CPUシャーディング {#3-per-cpu-sharding}

カウンタや統計には、per-CPU変数を使う。

```c
/* 変更前: 共有カウンタへのアトミックなインクリメント */
atomic_inc(&event_count);

/* 変更後: per-CPUカウンタ。同期は不要 */
this_cpu_inc(event_count);
```

### 4. ロック省略 / trylockパターン {#4-lock-elision-trylock-patterns}

場合によっては、低速パスにフォールバックすることで競合を回避できる。

```c
/* 取得を試み、ビジーなら作業をキューに入れる方式にフォールバックする */
if (!spin_trylock(&cache_lock)) {
    schedule_work(&cache_work);  /* 後でリトライする */
    return;
}
/* 高速パス */
spin_unlock(&cache_lock);
```

### 5. バッチ化 {#5-batching}

複数の更新を1つのクリティカルセクションにまとめることで、ロックの粒度を下げる。

```c
/* 変更前: 項目ごとに1回ロック */
for (item in items) {
    spin_lock(&list_lock);
    list_add(&item->node, &list);
    spin_unlock(&list_lock);
}

/* 変更後: バッチ */
spin_lock(&list_lock);
for (item in items)
    list_add(&item->node, &list);
spin_unlock(&list_lock);
```

## 参考文献 {#further-reading}

- [lockdep](lockdep.md) — 循環依存の検出
- [spinlock](spinlock.md) — spinlockが意味を持つ場面
- [RCU](rcu.md) — 読み込み中心のワークロード向け
- [per-CPU変数](percpu.md) — 共有カウンタの排除
