# 優先度逆転とPI mutex {#priority-inversion-and-pi-mutexes}

> リアルタイムにおける古典的なロックの問題と、Linuxによる解決策

## 優先度逆転の問題 {#the-priority-inversion-problem}

優先度逆転は、高優先度タスクが低優先度タスクの保持するリソースを待ってブロックされているときに、中優先度タスクが低優先度タスクの実行を妨げることで発生する。結果として、高優先度タスクは中優先度タスクの後ろで際限なく待たされる — 事実上、優先度が逆転するのである。

```
Priority:  HIGH        MEDIUM      LOW
           ┌───┐       ┌───┐       ┌───┐
           │ H │       │ M │       │ L │
           └───┘       └───┘       └───┘

Timeline:
  t=0:  L starts, acquires mutex
  t=1:  H wakes, tries to acquire mutex → BLOCKED on L
  t=2:  M wakes (higher priority than L) → PREEMPTS L
  t=3:  M runs freely (doesn't need mutex)
        L can't run (preempted by M)
        H is stuck waiting for L to release mutex
        H < M effectively, even though H > M nominally!
```

これは理論上の話ではない。Mars Pathfinderミッション（1997年）では優先度逆転が発生し、技術者が該当するmutexで優先度継承を有効にする修正をアップロードするまで、システムが周期的にリセットされた。（[Mike Jones, "What Really Happened on Mars?", 1997](https://www.cs.cornell.edu/courses/cs614/1999sp/papers/pathfinder.html)）

## 優先度継承: 解決策 {#priority-inheritance-the-solution}

タスクHがLの保持するmutexでブロックすると、カーネルはLの優先度を一時的にHのレベルまで引き上げる（ブーストする）。これにより、LはMをプリエンプトしてmutexを素早く解放できるようになる。

```
After PI boost:
  t=0:  L starts, acquires mutex
  t=1:  H blocks on mutex → L's priority BOOSTED to H's level
  t=2:  M wakes → cannot preempt L (L now has H's priority!)
  t=3:  L runs, finishes critical section, releases mutex
  t=4:  L's priority RESTORED to original
        H unblocks, runs (has highest priority)
        M runs after H
```

## rt_mutex: PI対応のロック {#rt_mutex-the-pi-aware-lock}

Linuxは、PI対応のロックとして`kernel/locking/rtmutex.c`に`rt_mutex`を提供している:

```c
// include/linux/rtmutex.h
struct rt_mutex_base {
    raw_spinlock_t          wait_lock;  // 待機者ツリーと所有者を保護する
    struct rb_root_cached   waiters;    // ブロックされたタスクの優先度順ツリー
    struct task_struct      *owner;     // 現在のロック所有者（LSB = 待機者あり）
};
```

待機者はFIFOキューではなく、**優先度順のRBツリー**に格納される。最も優先度の高い待機者がルートにあり、それが所有者のブースト後の優先度を決める。

## task_structにおける優先度の追跡 {#priority-tracking-in-task_struct}

```c
// include/linux/sched.h (CONFIG_RT_MUTEXES)
struct task_struct {
    // ...
    raw_spinlock_t          pi_lock;        // PI状態の変更を保護する
    struct rb_root_cached   pi_waiters;     // このタスクの優先度をブーストしているタスク
    struct task_struct      *pi_top_task;   // キャッシュ: 最高優先度のブースト元
    struct rt_mutex_waiter  *pi_blocked_on; // このタスクがブロックされているロック
};
```

`pi_waiters`は、このタスクが保持するロックを待っているタスクのRBツリーである。`pi_top_task`は、素早くアクセスできるよう最高優先度の待機者をキャッシュする。

## ブーストの仕組み {#the-boost-mechanism}

タスクHがLの保持するmutexを獲得しようとしたとき:

```c
// kernel/locking/rtmutex.c（簡略化）
int rt_mutex_lock(struct rt_mutex *lock)
{
    // 獲得を試みる: lock->ownerに対するCAS
    if (try_acquire(lock))
        return 0;

    // ロックが競合している: Hをlock->waitersに追加する
    // Lの優先度をHの優先度までブーストする
    rt_mutex_adjust_prio_chain(H, lock, ...)
}
```

`rt_mutex_adjust_prio_chain()`はロックの依存チェーンをたどる:

```c
// kernel/locking/rtmutex.c
static int rt_mutex_adjust_prio_chain(struct task_struct *task,
                                       enum rtmutex_chainwalk chwalk,
                                       struct rt_mutex_base *orig_lock,
                                       struct rt_mutex_base *next_lock,
                                       struct rt_mutex_waiter *orig_waiter,
                                       struct task_struct *top_task)
{
    // ロックチェーンをたどる: Lも別のロックでブロックされていれば、
    // ブーストを推移的に伝播させる
    while (task) {
        // タスクの優先度をブーストする
        rt_mutex_adjust_prio(task);

        // タスクが別のロックでブロックされていれば、チェーンを上へたどり続ける
        if (task->pi_blocked_on)
            task = task->pi_blocked_on->lock->owner;
        else
            break;
    }
}
```

### 連鎖的な優先度継承 {#chained-priority-inheritance}

PIはチェーンを扱わなければならない:

```
H waits on lock_1 held by M2
M2 waits on lock_2 held by L
→ L must be boosted to H's priority
→ M2 must also be boosted to H's priority (to run and release lock_2 first)
```

チェーンのたどりはこれを推移的に処理する。カーネルは、最悪ケースでO(n)となる経路がシステムをDoS状態にしないよう、チェーンの深さを制限している。

## ブーストの適用: rt_mutex_setprio() {#the-boost-application-rt_mutex_setprio}

```c
// kernel/sched/core.c
void rt_mutex_setprio(struct task_struct *p, struct task_struct *pi_task)
{
    // pi_taskは最高優先度の待機者（ブースト解除ならNULL）
    int prio = pi_task ? pi_task->prio : p->normal_prio;

    // pの優先度を更新する
    p->pi_top_task = pi_task;
    __setscheduler_prio(p, prio);

    // pが現在実行中なら、再スケジュールが必要になる場合がある
    if (p->on_rq)
        resched_curr(rq);
}
```

ブーストされたタスクはPIタスクの優先度で実行される — ロックを解放するまで、まるでその高優先度タスクであるかのようにスケジューリングされる。

## ロック解放時のブースト解除 {#unboost-on-lock-release}

Lがmutexを解放するとき:

1. LがHの`pi_waiters`から取り除かれる（あるいはHの優先度がLのブーストから取り除かれる）
2. `rt_mutex_setprio(L, new_pi_top)`が呼ばれる — ここで`new_pi_top`は次に優先度の高い待機者（いなければNULL）である
3. Lの優先度は`normal_prio`に戻る
4. Hが起床し、ロックを獲得する

```c
// kernel/locking/rtmutex.c
static void mark_wakeup_next_waiter(struct rt_wake_q_head *wqh,
                                     struct rt_mutex_base *lock)
{
    // 所有者のpi_waitersから先頭の待機者をデキューする
    rt_mutex_dequeue_pi(current, waiter);

    // ブースト解除: 所有者の優先度を次に高い待機者（またはnormal_prio）に戻す
    rt_mutex_adjust_prio(current);

    // 先頭の待機者（H）を起床させる
    rt_wake_q_add(wqh, waiter->task);
}
```

## ユーザー空間のrt_mutex: PTHREAD_PRIO_INHERIT {#userspace-rt_mutex-pthread_prio_inherit}

POSIX pthread mutexのプロトコル`PTHREAD_PRIO_INHERIT`は、カーネルの`rt_mutex`基盤を使う:

```c
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
pthread_mutex_init(&my_mutex, &attr);
```

Linuxでは、`PTHREAD_PRIO_INHERIT` mutexは`FUTEX_LOCK_PI`操作を伴うfutexで実装されており、それが内部で`rt_mutex`を使っている。

```bash
# futexのPI操作を確認する
strace -e futex ./my_rt_app 2>&1 | grep FUTEX_LOCK_PI
```

## PI mutexと通常のmutexの比較 {#pi-mutex-vs-regular-mutex}

| 特性 | pthread_mutex（デフォルト） | pthread_mutex（PRIO_INHERIT） |
|----------|------------------------|------------------------------|
| ロック所有者のブースト | なし | あり |
| オーバーヘッド | 低い | やや高い |
| RTに必要か | 不要 | 非RTタスクと共有するなら必要 |
| カーネル側の実装 | futex (FUTEX_LOCK) | futex (FUTEX_LOCK_PI / rt_mutex) |

すべてのスレッドが同じ優先度である純粋なRTアプリケーションでは、通常のmutexで問題ない。PI mutexが必要になるのは、RTタスクが非RTタスクとリソースを共有する場合である。

## SCHED_DEADLINEとPI {#sched_deadline-and-pi}

`SCHED_DEADLINE`タスクも優先度逆転を経験しうる。DLタスクが通常タスクまたはRTタスクの保持するrt_mutexでブロックすると、所有者はDLタスクから借りた仮想的なDLエンティティで実行されるようブーストされる:

```c
// kernel/sched/deadline.c
static inline bool is_dl_boosted(struct sched_dl_entity *dl_se)
{
    return pi_of(dl_se) != dl_se;  // ブーストされていればpi_seはドナーを指す
}
```

ブーストされたタスクは一時的にSCHED_DEADLINEのセマンティクスで実行され、任意のRTタスクをプリエンプトして、DLタスクのデッドラインより前にロックを解放できることが保証される。

## PREEMPT_RT拡張 {#the-preempt_rt-extension}

完全な`PREEMPT_RT`パッチセットは、カーネルのspinlockの大部分をrt_mutexに変換し、PIをほぼすべてのカーネルロックに拡張する。これによりカーネルコードの完全なプリエンプションが可能になり、上限のないレイテンシの発生源の大半が取り除かれる — その代償は、より複雑なロックとわずかに高いオーバーヘッドである。

カーネル6.x時点で、PREEMPT_RTの大部分はアップストリームに入っている。サポートされているアーキテクチャでは`CONFIG_PREEMPT_RT`を選択できる。

## PIの観測 {#observing-pi}

```bash
# rt_mutexでブロックされているタスクを確認する（wchanに表示される）
cat /proc/$PID/wchan   # PI mutexでブロックされていれば"rt_mutex_wait"と表示される

# PIブーストのイベントをトレースする
trace-cmd record -e rtmutex:rtmutex_acquire \
                 -e rtmutex:rtmutex_release \
                 ./workload

# perf: PIロックの獲得回数を数える
perf stat -e rtmutex:rtmutex_acquire -a sleep 10
```

## 参考文献 {#further-reading}

- [RTスケジューラ](rt-scheduler.md) — SCHED_FIFO/RRと、RTタスクにとってPIが重要な理由
- [SCHED_DEADLINE](deadline.md) — DLがPIロックとどう相互作用するか
- [kernel/locking/rtmutex.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/locking/rtmutex.c) — PIの完全な実装
- [LWN: Priority inheritance in the kernel](https://lwn.net/Articles/178253/)（2006年） — Thomas GleixnerとIngo Molnarによる最初の投稿
