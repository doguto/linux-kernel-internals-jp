# Linuxのプリエンプションモデル {#linux-preemption-model}

> カーネルはいつプリエンプトされうるのか？ Voluntary、Full、そしてPREEMPT_RT

## プリエンプションとは何か？ {#what-is-preemption}

**プリエンプション**とは、現在実行中のタスクを止めて、より優先度の高いタスクを実行することである。Linuxには複数のプリエンプションモデルがあり、ビルド時にKconfigで選択する:

```
CONFIG_PREEMPT_NONE        — Server: only voluntary preemption
CONFIG_PREEMPT_VOLUNTARY   — Desktop: voluntary + explicit preemption points
CONFIG_PREEMPT             — Full: preempt anywhere except spinlocks
CONFIG_PREEMPT_RT          — Real-time: preempt almost everywhere
```

モデルの選択は、**レイテンシ**（高優先度タスクがどれだけ速く実行されるか）と**スループット**（システム全体の効率）のバランスに影響する。

## preempt_count: プリエンプションの関門 {#preempt_count-the-preemption-gate}

すべてのタスクはthread_info内に`preempt_count`フィールドを持つ。この値が0でない間、プリエンプションは無効になる:

```c
/* include/linux/preempt.h */
/* preempt_countのレイアウト（CPUごと）: */
/*
 *         PREEMPT_MASK  (bits 0-7):   プリエンプション無効化のネスト深さ
 *         SOFTIRQ_MASK  (bits 8-15):  softirqのネスト深さ
 *         HARDIRQ_MASK  (bits 16-19): hardirq（割り込み）のネスト深さ
 *         NMI_MASK      (bit 20):     NMIコンテキスト
 */

/* プリエンプト可能かを確認する */
#define preemptible()   (preempt_count() == 0 && !irqs_disabled())

/* プリエンプションを無効化/有効化する */
preempt_disable();   /* preempt_count++ */
preempt_enable();    /* preempt_count--; 0になれば保留中の再スケジュールを確認する */

/* 低オーバーヘッド版（有効化時に再スケジュールを確認しない）: */
preempt_disable_notrace();
preempt_enable_notrace();
```

### preempt_countとコンテキスト {#preempt_count-contexts}

```c
/* 割り込みコンテキスト内にいるか？ */
in_interrupt()    /* HARDIRQ_MASK | SOFTIRQ_MASK | NMI_MASK */
in_irq()          /* HARDIRQ_MASKのみ */
in_softirq()      /* SOFTIRQ_MASKのみ */
in_nmi()          /* NMI_MASKのみ */

/* アトミックコンテキスト（スリープ不可）にいるか？ */
in_atomic()       /* preempt_count != 0 */
/* 注: hardirq、softirq、spinlock、preempt_disableを含む */
```

## CONFIG_PREEMPT_NONE: サーバーモデル {#config_preempt_none-server-model}

明示的な`schedule()`呼び出し以外ではプリエンプションが起きない。カーネルのコードパスは、自発的にCPUを明け渡すまで実行され続ける:

```
High-priority task wakes → must wait for current kernel path to finish
  ↓
Current task calls schedule() or returns to userspace
  ↓
High-priority task runs
```

**レイテンシ**: ミリ秒〜秒（カーネルパスが長い場合）
**スループット**: 最も高い（不要なコンテキストスイッチがない）
**用途**: サーバー、バッチワークロード（HPC、データベース）

## CONFIG_PREEMPT_VOLUNTARY: デスクトップモデル {#config_preempt_voluntary-desktop-model}

カーネル内の長いループの要所に、明示的なプリエンプションポイント（`might_sleep()`、`cond_resched()`）を追加する:

```c
/* mm/filemap.c — プリエンプションポイントを持つ長いループ */
ssize_t filemap_read(struct kiocb *iocb, struct iov_iter *iter, ...)
{
    do {
        /* ... ページを読む ... */
        cond_resched();  /* 高優先度タスクが待っていればCPUを明け渡す */
    } while (iov_iter_count(iter));
}

/* kernel/sched/fair.c */
static void __sched_yield(void)
{
    cond_resched();
}
```

`cond_resched()`は`TIF_NEED_RESCHED`を確認し、セットされていれば`schedule()`を呼ぶ — ただしプリエンプションが無効化されていない（`preempt_count == 0`）場合に限る。

**レイテンシ**: 数十ミリ秒
**用途**: デスクトップシステム

## CONFIG_PREEMPT: 完全プリエンプション {#config_preempt-full-preemption}

カーネルは、明示的なyieldポイントだけでなく、`preempt_count == 0`である任意の地点でプリエンプトされうる。`TIF_NEED_RESCHED`がセットされると、タスクは次のプリエンプション可能な時点でプリエンプトされる:

```c
/* arch/x86/kernel/entry_64.S */
/*
 * 割り込みからの復帰パス:
 * カーネルモードへ戻る場合、プリエンプションが必要かを確認する
 */
.Lreturn_from_interrupt:
    testb $3, CS-ORIG_RAX(%rsp)  /* ユーザー空間へ戻るか？ */
    jnz .Lreturn_to_userspace

    /* カーネルへ戻る: preempt_countとTIF_NEED_RESCHEDを確認する */
    call preempt_schedule_irq    /* 必要ならプリエンプトする */
```

```c
/* kernel/sched/core.c */
asmlinkage __visible void __sched preempt_schedule_irq(void)
{
    /* TIF_NEED_RESCHEDがセットされ、かつpreempt_count == 0のとき */
    /* 割り込みからの復帰時に呼ばれる */
    do {
        preempt_disable();       /* 再帰的なプリエンプションを防ぐ */
        local_irq_enable();
        __schedule(SM_PREEMPT);  /* コンテキストスイッチ */
        local_irq_disable();
        sched_preempt_enable_no_resched();
    } while (need_resched());
}
```

**レイテンシ**: 数百マイクロ秒
**用途**: デスクトップ、組み込み、オーディオシステム

## CONFIG_PREEMPT_RT: リアルタイム {#config_preempt_rt-real-time}

PREEMPT_RT（6.12で全アーキテクチャ向けに完全にマージされた — [マージコミット`baeb9a7d8b60`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=baeb9a7d8b60b021d907127509c44507539c15e5)、[LWN](https://lwn.net/Articles/989212/)）は、ほぼすべてのspinlockをスリープ可能なmutexに変換し、「spinlock」を保持している間でもプリエンプションを可能にする:

```
Non-RT:                          RT:
  spin_lock → disable preempt     spin_lock → sleeping mutex
  [not preemptible]               [can be preempted]
  spin_unlock                     spin_unlock

  Result: RT task must wait       Result: RT task preempts
  for lock holder to finish       lock holder immediately
```

### PREEMPT_RTで何が変わるか {#what-changes-in-preempt_rt}

```c
/* PREEMPT_RTでは、大半のspinlockがrt_mutexになる: */
spinlock_t lock;
spin_lock(&lock);   /* PREEMPT_RT: スリープ可能なmutex_lock(&rt_mutex) */
spin_unlock(&lock); /* PREEMPT_RT: mutex_unlock */

/* 例外（RTでもraw spinlockのまま）: */
raw_spinlock_t raw_lock;
raw_spin_lock(&raw_lock);   /* RTでも決してスリープしない */
/* 用途: スケジューラ内部、IRQサブシステム、短いクリティカルセクション */

/* softirqはCPUごとの専用スレッドで実行される（RT）: */
/* ksoftirqd/0は通常のkthreadであり、プリエンプト可能 */

/* カーネル内タイマーはhrtimerのsoftirqスレッドから発火する（プリエンプト可能） */
```

### 優先度逆転の防止 {#priority-inversion-prevention}

PREEMPT_RTは、RT mutexに対して**優先度継承**の仕組みを有効にする:

```c
/* 優先度継承なしの場合: */
Low-prio task holds lock → High-prio RT task waits → Medium-prio task runs
  → RT task is effectively blocked by medium-prio task (priority inversion!)

/* RT mutexの優先度継承ありの場合: */
Low-prio task holds lock → High-prio RT task waits
  → Low-prio task temporarily runs at HIGH priority
  → Medium-prio task does NOT run
  → Low-prio finishes, releases lock
  → High-prio RT task runs immediately
```

詳細は[優先度逆転とPI mutex](pi-mutexes.md)を参照。

### PREEMPT_RTのレイテンシ {#preempt_rt-latency}

```
CONFIG_PREEMPT_NONE:       ~1-100ms worst-case
CONFIG_PREEMPT_VOLUNTARY:  ~1-10ms
CONFIG_PREEMPT:            ~100-1000µs
CONFIG_PREEMPT_RT:         ~10-100µs (with proper tuning)
```

## 実際のpreempt_disable {#preempt_disable-in-practice}

### spinlockは暗黙にプリエンプションを無効化する {#spinlocks-implicitly-disable-preemption}

```c
spinlock_t lock;

spin_lock(&lock);
/* preempt_count++（非RTの場合） */
/* この時点で: スリープ不可、プリエンプト不可 */

/* ... クリティカルセクション ... */

spin_unlock(&lock);
/* preempt_count-- */
/* TIF_NEED_RESCHEDがセットされていれば、ここでschedule()が呼ばれる */
```

### CPUごとの操作にはプリエンプション無効化が必要 {#per-cpu-operations-require-preemption-disabled}

```c
/* CPUごとの変数は、プリエンプションが無効な間だけ安全である */
/* （そうでなければ、操作の途中で別のCPUへマイグレーションしうる） */

preempt_disable();
this_cpu_inc(my_per_cpu_counter);
preempt_enable();

/* あるいはヘルパーを使う（暗黙にプリエンプションを無効化する）: */
this_cpu_inc(my_per_cpu_counter);  /* 常に安全: 暗黙にプリエンプションを無効化する */
```

### ドライバでのコンテキスト確認 {#checking-context-in-a-driver}

```c
void my_function(void) {
    if (in_interrupt()) {
        /* IRQから呼ばれた: スリープ不可、スリープするロックも取れない */
        /* アトミックな割り当てを使う: GFP_ATOMIC */
        ptr = kmalloc(size, GFP_ATOMIC);
    } else if (in_atomic()) {
        /* プリエンプション無効中（spinlockなど）に呼ばれた */
        /* こちらもスリープ不可 */
    } else {
        /* 通常のプロセスコンテキスト: スリープ可能 */
        ptr = kmalloc(size, GFP_KERNEL);
        mutex_lock(&my_mutex);
    }
}
```

## might_sleep: アトミックコンテキストでのスリープをデバッグする {#might_sleep-debugging-sleeping-in-atomic-context}

```c
/* include/linux/kernel.h */
#define might_sleep() \
    do { __might_sleep(__FILE__, __LINE__); } while (0)

void __might_sleep(const char *file, int line)
{
    if (in_atomic() || irqs_disabled()) {
        /* スタックトレース付きでWARNを出力する */
        pr_warn("BUG: sleeping function called from invalid context at %s:%d\n",
                 file, line);
        dump_stack();
    }
}

/* 次のものから自動的に呼ばれる: */
/* mutex_lock, down(), copy_to_user, kmalloc(GFP_KERNEL), ... */
```

割り込みハンドラからうっかり`mutex_lock()`を呼ぶと「BUG: sleeping function called from invalid context」が出るのはこのためである。

## プリエンプションモデルの確認 {#checking-the-preemption-model}

```bash
# どのプリエンプションモデルが有効かを確認する
zcat /proc/config.gz | grep PREEMPT
# CONFIG_PREEMPT_NONE=y      (or VOLUNTARY, PREEMPT, PREEMPT_RT)

# 実行時に確認する
uname -v | grep PREEMPT
# #1 SMP PREEMPT_RT ...

# プリエンプションの統計
cat /proc/sched_debug | grep -A5 "preempt"

# レイテンシのトレーシング（ftrace）
echo preemptirqsoff > /sys/kernel/debug/tracing/current_tracer
# プリエンプション無効期間の最長区間をトレースする
cat /sys/kernel/debug/tracing/trace | head -50
```

## 参考文献 {#further-reading}

- [コンテキストスイッチ](context-switch.md) — schedule()が何をするか
- [プロセスが起床するときに何が起きるか](wakeup.md) — TIF_NEED_RESCHEDの設定
- [優先度逆転とPI mutex](pi-mutexes.md) — RTの優先度継承
- [spinlockとraw_spinlock](../locking/spinlock.md) — spinlockにおけるプリエンプション無効化
- [softirq](../interrupts/softirq.md) — PREEMPT_RTにおけるsoftirqのスレッド化
- `kernel/sched/core.c` — preempt_schedule_irq, __schedule
- `include/linux/preempt.h` — preempt_countのマクロ
