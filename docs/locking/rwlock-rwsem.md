# rwlockとrwsem {#rwlock-and-rwsem}

> reader-writerロック: 読み込みは並行、書き込みは排他

## reader-writerパターン {#the-reader-writer-pattern}

多くのカーネルのデータ構造は、書き込まれるよりはるかに頻繁に読まれる。普通のspinlockやmutexは、衝突しない並行読み込みも含めて、すべてのアクセスを直列化してしまう。reader-writerロックは、書き手には排他アクセスを保証しつつ、複数の読み手が同時にアクセスすることを許す。

```
Multiple readers: OK
Reader + writer:  BLOCKED (writer waits)
Two writers:      BLOCKED (second waits)
```

カーネルは2つのreader-writerプリミティブを提供している。

| プリミティブ | スリープするか | IRQセーフか | 使う場面 |
|-----------|---------|-----------|----------|
| `rwlock_t` | しない（spinlock） | はい | 割り込みコンテキスト、短いクリティカルセクション |
| `rw_semaphore` | する（スリープ） | いいえ | プロセスコンテキスト、長めのクリティカルセクション |

## rwlock_t: スピンする版 {#rwlock_t-the-spinning-variant}

`rwlock_t`はreader-writer spinlockである。読み手も書き手も、スリープせずにスピンする。

```c
#include <linux/rwlock.h>

/* 初期化 */
DEFINE_RWLOCK(my_rwlock);  /* 静的 */
rwlock_t my_rwlock;
rwlock_init(&my_rwlock);   /* 動的 */

/* 読み側 */
read_lock(&my_rwlock);
/* 読み込みクリティカルセクション — 他の読み手と並行 */
read_unlock(&my_rwlock);

/* 書き側 */
write_lock(&my_rwlock);
/* 書き込みクリティカルセクション — 排他 */
write_unlock(&my_rwlock);

/* IRQセーフ版（ロックがIRQハンドラでも取られる場合） */
unsigned long flags;
read_lock_irqsave(&my_rwlock, flags);
read_unlock_irqrestore(&my_rwlock, flags);

write_lock_irqsave(&my_rwlock, flags);
write_unlock_irqrestore(&my_rwlock, flags);

/* BH版 */
read_lock_bh(&my_rwlock);
read_unlock_bh(&my_rwlock);
```

PREEMPT_RTカーネルでは、`spinlock_t`が`rt_mutex`に対応付けられるのと同様に、`rwlock_t`は`rwbase_rt`（スリープする）で実装される。

## rw_semaphore: スリープする版 {#rw_semaphore-the-sleeping-variant}

`rw_semaphore`は`rwlock_t`のスリープする対応物である。競合した待機者は、（mutexと同様に短い楽観的スピンの段階を経た後）スピンせずにスリープする。

```c
/* include/linux/rwsem.h */
struct rw_semaphore {
    atomic_long_t count;   /* 読み手の数 + RWSEM_WRITER_LOCKEDビット */
    atomic_long_t owner;   /* 書き手のtask_struct* */
    struct osq_lock osq;   /* 楽観的スピンを行う者 */
    raw_spinlock_t wait_lock;
    struct list_head wait_list;
};
```

`count`フィールドは、ロック状態と読み手の数の両方をエンコードする。下位ビットはフラグであり、読み手の数は`RWSEM_READER_SHIFT`（8）だけシフトされて上位ビットに格納される。

```
count = 0                       → unlocked
count & RWSEM_WRITER_LOCKED     → held by writer (bit 0)
count >> RWSEM_READER_SHIFT     → number of active readers
```

### API {#api}

```c
#include <linux/rwsem.h>

/* 初期化 */
DECLARE_RWSEM(my_rwsem);       /* 静的 */
struct rw_semaphore my_rwsem;
init_rwsem(&my_rwsem);         /* 動的 */

/* 読み側 */
down_read(&my_rwsem);          /* スリープしうる */
/* 読み込みクリティカルセクション */
up_read(&my_rwsem);

/* 割り込み可能な読み込みロック */
if (down_read_interruptible(&my_rwsem))
    return -ERESTARTSYS;
up_read(&my_rwsem);

/* trylock（ブロックしない） */
if (down_read_trylock(&my_rwsem)) {
    /* 取得できた */
    up_read(&my_rwsem);
}

/* 書き側 */
down_write(&my_rwsem);         /* スリープしうる、排他 */
/* 書き込みクリティカルセクション */
up_write(&my_rwsem);

/* 書き込みから読み込みへの格下げ（解放せずに） */
downgrade_write(&my_rwsem);    /* 排他 → 共有に降格 */
/* ここからは読み側クリティカルセクション */
up_read(&my_rwsem);
```

### downgrade_write {#downgrade_write}

`downgrade_write()`は、書き込みロックをアトミックに読み込みロックへ変換する。書き手が書き込みの段階を終え、他の読み手を受け入れながら読み込みを続けたい場合に使われる。

```c
down_write(&mm->mmap_lock);
/* vmaリストを変更する */
downgrade_write(&mm->mmap_lock);
/* ここからは読み込み専用。他の読み手も参加できる */
up_read(&mm->mmap_lock);
```

## 書き手の飢餓 {#writer-starvation}

素朴なreader-writerロックの重大な問題として、読み手が絶え間なく到着すると、書き手がいつまでもロックを取れない可能性がある。Linuxのrwsemは書き手をキューに入れることでこれを防ぐ。書き手が待機し始めると、新しい読み手もキューに入れられる（待機中の書き手を追い越さない）。

```
Timeline:
  R1 holds read lock
  W1 requests write lock → queues
  R2 requests read lock → also queues (behind W1)
  R1 releases → W1 gets lock
  W1 releases → R2 gets lock
```

## rwlock、rwsem、普通のmutex/spinlockの使い分け {#when-to-use-rwlock-vs-rwsem-vs-plain-mutexspinlock}

```
rwlock_t:
  ✓ IRQコンテキストの読み手または書き手
  ✓ 非常に短い読み込み/書き込みクリティカルセクション
  ✗ 書き側がまれなら使わない — 単なるspinlockを検討する

rw_semaphore:
  ✓ プロセスコンテキスト、長いクリティカルセクション
  ✓ 読み込み中心のワークロード（多数の並行する読み手が実際の利点を生む）
  ✗ 非常に短いクリティカルセクションには割に合わない（mutexのオーバーヘッドと同程度）

経験則: 読み込みが書き込みの5倍以上頻繁で、
読み込みが問題になるほど長いなら、rwsemが有利である。
mmサブシステムはこれを多用している（mm->mmap_lock）。
```

## 実際の使用例: mmap_lock {#real-world-usage-mmap_lock}

各プロセスのVMAリストは、`mm->mmap_lock`（`rw_semaphore`）で保護されている。

```c
/* VMAの読み込み（例: ページフォールト中） */
mmap_read_lock(mm);      /* = down_read(&mm->mmap_lock) */
vma = find_vma(mm, addr);
/* vmaを使う */
mmap_read_unlock(mm);    /* = up_read(&mm->mmap_lock) */

/* VMAの変更（例: mmap/munmapシステムコール） */
mmap_write_lock(mm);     /* = down_write(&mm->mmap_lock) */
/* VMAの追加/削除/変更 */
mmap_write_unlock(mm);   /* = up_write(&mm->mmap_lock) */
```

これにより、同じプロセス内の複数のスレッドが並行してページフォールトを処理でき、一方でmmap/munmap操作は直列化される。

## 参考文献 {#further-reading}

- [spinlockとraw_spinlock](spinlock.md) — 普通のspinlock
- [RCU](rcu.md) — さらに並行性の高い読み込み（読み手側のロックがまったくない）
- [VMA単位のロック](../mm/per-vma-locks.md) — mmap_lockに代わる細粒度の選択肢
- カーネルツリー内の`Documentation/locking/`
