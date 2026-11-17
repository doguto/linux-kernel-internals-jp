# spinlockとraw_spinlock {#spinlock-and-raw_spinlock}

> スリープしないビジーウェイト型のロック。カーネルの同期の土台

## spinlockとは何か {#what-is-a-spinlock}

spinlockは、ロックが利用可能になるまでスレッドがタイトなループでスピンすることで保持するロックである。mutexと異なり、スレッドをスリープさせることは決してない。そのため、スリープが禁止されているコンテキスト、つまり割り込みハンドラ、NMIハンドラ、および`in_interrupt()`が真となるあらゆる場所でspinlockを使える。

代償として、スピンはCPUサイクルを浪費する。spinlockは非常に短いクリティカルセクション（通常はミリ秒ではなくマイクロ秒単位）だけを保護すべきである。

## 2種類のspinlock {#two-spinlock-types}

カーネルには、見た目は似ているがPREEMPT_RTカーネルでは異なる振る舞いをする2つの型がある。

```c
/* include/linux/spinlock_types.h */

/* 非RTカーネル: spinlock_tはraw_spinlock_tをラップする */
typedef struct spinlock {
    union {
        struct raw_spinlock rlock;
    };
} spinlock_t;

/* raw_spinlock_t: 常に本物のspinlockで、変換されることはない */
typedef struct raw_spinlock {
    arch_spinlock_t raw_lock;   /* アーキテクチャ固有。例: queued spinlock */
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map;
#endif
} raw_spinlock_t;
```

標準のカーネル（非RT）では、`spinlock_t`と`raw_spinlock_t`は同一の振る舞いをする。

PREEMPT_RTカーネルでは、RTタスクがロック保持者をプリエンプトできるように、`spinlock_t`はスリープするロック（`rt_mutex`が裏付け）になる。`raw_spinlock_t`はRTでも本物のspinlockのままである。

**ルール**: ほとんどのコードでは`spinlock_t`を使う。RTカーネルでも本物のspinlockが必要な場合に限り`raw_spinlock_t`を使う（例: スケジューラのランキューロック。スリープの仕組みそのものがスケジューラなので、これはスリープできない）。

## 基本API {#basic-api}

```c
#include <linux/spinlock.h>

/* 静的初期化 */
DEFINE_SPINLOCK(my_lock);

/* 動的初期化 */
spinlock_t my_lock;
spin_lock_init(&my_lock);

/* ロック / アンロック（プロセスコンテキスト、プリエンプション無効） */
spin_lock(&my_lock);
/* クリティカルセクション */
spin_unlock(&my_lock);

/* ロック/アンロック + ローカル割り込みの無効化 */
unsigned long flags;
spin_lock_irqsave(&my_lock, flags);
/* クリティカルセクション — IRQハンドラもこのロックを取りうる
   プロセスコンテキストから呼ばれても安全 */
spin_unlock_irqrestore(&my_lock, flags);

/* ロック/アンロック + softirq（ボトムハーフ）のみ無効化 */
spin_lock_bh(&my_lock);
spin_unlock_bh(&my_lock);

/* ブロックしないtrylock（成功で1、失敗で0を返す） */
if (spin_trylock(&my_lock)) {
    /* ロックを取得できた */
    spin_unlock(&my_lock);
}
```

## irqsaveと素のspin_lockの使い分け {#when-to-use-irqsave-vs-plain-spin_lock}

ルール: 同じロックがプロセスコンテキストと割り込みハンドラの両方から取得されるなら、`spin_lock_irqsave()`が必要である。

```c
/* 誤り: プロセスコンテキストがロックを取った後、IRQが発生して
   同じロックを取ろうとする → デッドロック */
static spinlock_t counter_lock;
static int counter;

void update_counter(void)       /* プロセスコンテキスト */
{
    spin_lock(&counter_lock);   /* ロックを取る */
    counter++;
    spin_unlock(&counter_lock); /* ロックを解放する */
}

irqreturn_t my_irq_handler(int irq, void *data)
{
    spin_lock(&counter_lock);   /* デッドロック: プロセスがロックを保持中にIRQが発生 */
    counter++;
    spin_unlock(&counter_lock);
    return IRQ_HANDLED;
}

/* 正しい: ロック保持中はIRQを無効化する */
void update_counter(void)
{
    unsigned long flags;
    spin_lock_irqsave(&counter_lock, flags);
    counter++;
    spin_unlock_irqrestore(&counter_lock, flags);
}
```

ロックが単一のIRQハンドラ内でのみ使われる場合（あるいは、すでにIRQがオフになっているプロセスコンテキストでのみ使われる場合）は、素の`spin_lock()`で十分である。

## 仕組み: queued spinlock {#how-it-works-queued-spinlocks}

現代のx86は**queued spinlock**を使う（MCSベースで、Waiman LongによってLinux 4.2で導入された。[`a33fda35e3a7`](https://git.kernel.org/linus/a33fda35e3a7655fb7df756ed67822afb5ed5e8d)、[LWN](https://lwn.net/Articles/590243/)）。すべての待機者が同じ変数上でスピンする（キャッシュラインのバウンスを引き起こす）代わりに、各CPUは自分専用のper-CPUノード上でスピンする。

```
Lock structure:
┌────────────────────────────────────────┐
│ locked byte | pending | tail (cpu+ctx) │  ← 32-bit word
└────────────────────────────────────────┘
  bits 0-7: locked byte (0=free, 1=locked)
  bit 8:    pending (a waiter is about to acquire)
  bits 16-31: tail (encodes CPU number + nesting context)

MCS queue node (per-CPU):
┌────────────────────────────┐
│ locked | next_cpu_ptr      │
└────────────────────────────┘
         ↑
         Each waiter spins on its own node's 'locked' field.
         No cache line bouncing.
```

CPUがロックを取得すると、次のCPUのノードの`locked`をクリアし、その待機者だけがスピンをやめる。これは数百個のCPUまでスケールする。

## raw_spinlockのAPI {#raw_spinlock-api}

`raw_spinlock_t`は同じ命名規則に`raw_`接頭辞を付けたものを使う。

```c
raw_spinlock_t raw_lock = __RAW_SPIN_LOCK_UNLOCKED(raw_lock);

raw_spin_lock(&raw_lock);
raw_spin_unlock(&raw_lock);

raw_spin_lock_irqsave(&raw_lock, flags);
raw_spin_unlock_irqrestore(&raw_lock, flags);

raw_spin_trylock(&raw_lock);       /* 成功で1を返す */
```

## デバッグ: lockdepが誤用を捕まえる {#debugging-lockdep-catches-misuse}

`CONFIG_DEBUG_SPINLOCK`と`CONFIG_LOCKDEP`があれば、カーネルは次を検出する。

- 二重ロック（すでに保持しているロックを取る）
- ロック順序の違反（ロッククラス間の潜在的なデッドロック）
- spinlockを保持したままのスリープ
- プロセスコンテキストでIRQを無効化せずに取られているロックを、hardirqコンテキストでも取る

```
BUG: spinlock already locked on CPU#1
spin_lock() at include/linux/spinlock.h:186
...
```

詳細は[lockdep](lockdep.md)を参照のこと。

## よくある誤り {#common-mistakes}

```c
/* してはならない: スリープする呼び出しをまたいでspinlockを保持する */
spin_lock(&lock);
kmalloc(size, GFP_KERNEL);  /* スリープしうる — バグ */
spin_unlock(&lock);

/* するべき: spinlock保持中はGFP_ATOMICを使う */
spin_lock(&lock);
kmalloc(size, GFP_ATOMIC);  /* 決してスリープしない */
spin_unlock(&lock);

/* してはならない: 同じロックに対してspin_lockを2回呼ぶ（再帰不可） */
spin_lock(&lock);
spin_lock(&lock);  /* デッドロック */

/* してはならない: アンロックせずに関数を抜ける */
spin_lock(&lock);
if (error)
    return -EINVAL;  /* ロックがリークする！ */
spin_unlock(&lock);
```

## 参考文献 {#further-reading}

- [mutexとrt_mutex](mutex.md) — スリープしても構わない場合
- [lockdep](lockdep.md) — カーネルがロックの使い方をどう検証するか
- [アトミック操作とメモリバリア](atomics.md) — spinlockの土台
- カーネルツリー内の`Documentation/locking/spinlocks.rst`
