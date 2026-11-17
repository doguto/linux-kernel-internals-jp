# mutexとrt_mutex {#mutex-and-rt_mutex}

> プロセスコンテキスト用のスリープするロック。クリティカルセクションがブロックしうる場合に適した道具

## mutexとは何か {#what-is-a-mutex}

mutex（相互排他ロック）はスリープするロックである。スレッドがmutexを取得できないとき、そのスレッドはスリープし、ロックが利用可能になると起床させられる。これはビジーウェイトするspinlockとは異なる。

mutexの取得はスリープしうるため、mutexは**プロセスコンテキスト**でしか使えない。割り込みハンドラ、softirq、taskletでは決して使えない。

## mutexの構造体 {#the-mutex-structure}

```c
/* include/linux/mutex_types.h */
struct mutex {
    atomic_long_t   owner;      /* オーナーのtask_struct* + 下位ビットのフラグ */
    raw_spinlock_t  wait_lock;  /* wait_listを保護する */
    struct osq_lock osq;        /* 楽観的スピンのキュー（MCS） */
    struct list_head wait_list; /* 待機中タスクのリスト */
};
```

`owner`フィールドは、オーナーへのポインタと状態フラグを1つのアトミック変数に詰め込んでいる。

```c
/* kernel/locking/mutex.h */
#define MUTEX_FLAG_WAITERS  0x01  /* wait_listに待機者がいる */
#define MUTEX_FLAG_HANDOFF  0x02  /* ロックを先頭の待機者に引き渡し中 */
#define MUTEX_FLAG_PICKUP   0x04  /* 先頭の待機者がロックを受け取り中 */
#define MUTEX_FLAGS         0x07
```

タスクへのポインタは`owner & ~MUTEX_FLAGS`に格納される（`task_struct`は少なくとも8バイトにアラインされているので、下位3ビットは常にゼロであり、安全である）。

## 基本API {#basic-api}

```c
#include <linux/mutex.h>

/* 静的初期化 */
DEFINE_MUTEX(my_mutex);

/* 動的初期化 */
struct mutex my_mutex;
mutex_init(&my_mutex);

/* ロック（スリープしうる） */
mutex_lock(&my_mutex);
/* クリティカルセクション */
mutex_unlock(&my_mutex);

/* 割り込み可能なロック（シグナルが届くと-EINTRを返す） */
if (mutex_lock_interruptible(&my_mutex))
    return -ERESTARTSYS;
/* クリティカルセクション */
mutex_unlock(&my_mutex);

/* killableなロック（致命的なシグナルでのみ中断される） */
if (mutex_lock_killable(&my_mutex))
    return -EINTR;

/* ブロックしないtrylock（成功で1、ロック済みなら0を返す） */
if (mutex_trylock(&my_mutex)) {
    /* 取得できた */
    mutex_unlock(&my_mutex);
}

/* ロックされているかを確認する（参考情報であり、同期には使えない） */
mutex_is_locked(&my_mutex);
```

## 3つの取得戦略 {#three-acquisition-strategies}

`mutex_lock()`が呼ばれ、ロックが保持されている場合、カーネルは3つの戦略を順に試す。

```
1. Fastpath (no contention)
   ─────────────────────────
   atomic_long_try_cmpxchg(owner, 0, current)
   → succeeds immediately, no waiting

2. Midpath (owner is running on another CPU)
   ──────────────────────────────────────────
   Optimistic spinning via OSQ (MCS queue)
   → spins briefly, hoping owner finishes quickly
   → avoids the overhead of sleep/wake
   → bails out if owner is preempted

3. Slowpath (must sleep)
   ──────────────────────
   Add self to wait_list, set TASK_UNINTERRUPTIBLE, schedule()
   → woken by mutex_unlock() when lock is released
```

この3段階のアプローチにより、ロックがすぐに解放される場合（負荷のかかったマルチプロセッサシステムでよくあるケース）にスリープを避けられる。

## 厳格なセマンティクス {#strict-semantics}

mutexは、spinlockにはない厳格な所有権のルールを強制する。

- **オーナーだけがアンロックできる**: 別のタスクから`mutex_unlock()`を呼ぶのはバグである
- **再帰的なロックは不可**: mutexを保持しているタスクが同じmutexに対して再び`mutex_lock()`を呼んではならない。デッドロックする
- **ロックを保持したままの終了は不可**: タスクがmutexを保持したまま終了すると、システムはそれを検出する（`CONFIG_DEBUG_MUTEXES`使用時）
- **割り込みコンテキストでの使用は不可**: 使おうとするとカーネルが警告する

```c
/* してはならない: 別のタスクからアンロックする */
void thread_a(void) { mutex_lock(&m); /* thread_bに引き渡す */ }
void thread_b(void) { mutex_unlock(&m); /* バグ: オーナーではない */ }

/* してはならない: 再帰的なロック */
mutex_lock(&m);
mutex_lock(&m);  /* デッドロック */
```

## rt_mutex: 優先度継承mutex {#rt_mutex-priority-inheritance-mutex}

PREEMPT_RTカーネルでは、高優先度のRTタスクがロック保持者をプリエンプトできるように、`spinlock_t`は`struct rt_mutex`を使って実装される。優先度継承が明示的に必要な場合には、`struct rt_mutex`を直接使うこともできる。

```c
/* include/linux/rtmutex.h */
struct rt_mutex_base {
    raw_spinlock_t  wait_lock;
    struct rb_root_cached waiters;  /* 優先度順に並べた待機者 */
    struct task_struct *owner;
};
```

決定的な違いは次の点である。rt_mutexでは、高優先度のタスクがロックを待っている場合、ロック保持者はロックを解放するまでその高い優先度を**継承**する。これにより**優先度逆転**（低優先度のタスクが高優先度のタスクを無期限にブロックすること）が防がれる。

```
Without PI:
  RT task (prio=90)    waits on lock held by...
  Normal task (prio=20)   ... but gets preempted by...
  Medium task (prio=50)   ... which runs for a long time → RT task starves

With rt_mutex:
  RT task (prio=90)    waits on lock held by...
  Normal task (prio=20)   ... is boosted to prio=90, runs immediately
  → RT task gets the lock quickly
```

詳しくは[優先度逆転とPI mutex](../sched/pi-mutexes.md)を参照のこと。

## mutexとspinlockの選択 {#choosing-between-mutex-and-spinlock}

```
mutexを使うべき場合:
  ✓ クリティカルセクションがブロックしうる（例: kmalloc(GFP_KERNEL)、copy_from_user）
  ✓ クリティカルセクションが長い（ミリ秒単位）
  ✓ プロセスコンテキストにいる

spinlockを使うべき場合:
  ✓ クリティカルセクションが決してブロックしない
  ✓ クリティカルセクションが非常に短い（マイクロ秒単位）
  ✓ 割り込みコンテキストにいる
  ✓ IRQからの保護が必要（spin_lock_irqsave）
```

## 設計の歴史 {#design-history}

### セマフォ（2.6.16以前） {#semaphores-pre-2616}

`struct mutex`が存在する前、カーネルの標準的なスリープするロックは`struct semaphore`であった。セマフォはカウントを持つ。`down()`はそれをデクリメントし、カウントがゼロに達するとスリープする。`up()`はそれをインクリメントし、待機者を起床させる。

カウンティングセマフォをmutexとして使うことの問題は、`up()`が（`down()`を呼んだタスクに限らず）*任意の*タスクから呼べるため、カーネルが所有権を強制できないことであった。これにより次のことができなかった。

- 再帰的なロック（デッドロックしうる）の検出
- 優先度継承（誰がロックを保持しているのか？ 不明）
- 「タスク終了時にロックが保持されていないか」をチェックするデバッグツール
- ロック順序の静的解析

Ingo Molnárは、明示的な単一オーナーのセマンティクスを持つ`struct mutex`をLinux 2.6.16（2006年）で導入した。[`6053ee3b32e3`](https://git.kernel.org/linus/6053ee3b32e3437e8c1e72687850f436e779bd49)であり、[LWN](https://lwn.net/Articles/164802/)で発表された。mutexはセマフォよりもセマンティクスが厳格であり、それが上記のすべてを可能にする。カーネルのコードはその後数年かけて`struct semaphore`から`struct mutex`へと徐々に移行された。新しいコードは、カウンティングセマフォが本当に必要でない限り、常に`struct mutex`を使うべきである。

### Big Kernel Lock（BKL） {#the-big-kernel-lock-bkl}

BKL（`lock_kernel()` / `unlock_kernel()`）は、カーネルを素早くSMPセーフにするためにLinux 2.0で導入された、単一のグローバルな再帰的spinlockであった。そのアプローチは、すべてのデータ構造を細粒度のロックに変換する代わりに、大きなサブシステムを1つの粗いロックで包むというものだった。

BKLには変わった性質があった。`schedule()`で自動的に解放され（そのため保持者は他の保持者をデッドロックさせずにスリープできた）、かつ再帰的であった。これらの性質のおかげで導入は簡単だったが、取り除くのは難しかった。取り除くには、保護されていたコードがBKLなしでも安全であることを証明する必要があったからである。

開発者たちはサブシステムごとに、BKLの区間を適切なサブシステム単位のmutexやspinlockに置き換えていった。この過程には10年以上かかった。

- VFS: 2004〜2007年頃に変換（big_kernel_lock → i_mutexなど）
- TTY層: 2009年頃に変換
- 残りのネットワークドライバ、サウンド: 2011〜2013年頃に変換
- BKLはLinux 2.6.39（2011年5月）で完全に削除された。Arnd Bergmannによる[`4ba8216cd905`](https://git.kernel.org/linus/4ba8216cd90560bc402f52076f64d8546e8aefcb)（"BKL: That's all, folks"）— [LWNの背景記事](https://lwn.net/Articles/384855/)

### PREEMPT_RTとrt_mutex {#preempt_rt-and-rt_mutex}

`CONFIG_PREEMPT_RT`カーネル（リアルタイムLinuxの導入環境で使われる）では、`spinlock_t`自体が`rt_mutex`の上に実装される。spinlockは、完全な優先度継承を伴い、より高優先度のRTタスクにプリエンプトされうるスリープするロックになる。ネストしたロックのシナリオでも、PIチェーンをたどることで優先度の上限は正しく保たれる。

つまり、PREEMPT_RTシステムでは、ほぼすべてのカーネルのロックが同じ優先度継承の仕組みを使うため、優先度逆転は事実上起こりえなくなる。その代償として、素のspinlockのモデルに比べてスループットがいくらか低下する。

## 参考文献 {#further-reading}

- [spinlockとraw_spinlock](spinlock.md) — ビジーウェイトする代替手段
- [lockdep](lockdep.md) — デッドロックとロック順序違反の検出
- [優先度逆転とPI mutex](../sched/pi-mutexes.md) — rt_mutexが重要になる場面
- カーネルツリー内の`Documentation/locking/mutex-design.rst`
