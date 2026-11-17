# lockdep: ロック依存関係検証器 {#lockdep-lock-dependency-validator}

> カーネルの実行時ロック正当性チェッカ

## lockdepが行うこと {#what-lockdep-does}

lockdepは、実行時にロックの取得順序を追跡し、実際に起きる前に潜在的なデッドロックを検出するカーネルサブシステム（`CONFIG_PROVE_LOCKING`で有効化される）である。次のものを捕まえる。

- **循環したロック依存**: A → B → A（古典的なデッドロック）
- **ロックの逆転**: あるコードパスではAの次にBを取り、別のコードパスではBの次にAを取る
- **不正なIRQコンテキスト**: 割り込みコンテキストでスリープするロックを取る
- **再帰的なロック**: 解放せずに同じロックを2回取る
- **アノテーションの欠落**: サブクラスに依存するロック順序

lockdepは、デッドロックにつながりうる新しいロック順序を初めて見た時点で「possible deadlock」を報告する。実際のデッドロックがまだ一度も起きていなくてもである。本番環境で本物のデッドロックが起きるのを待つより、はるかに良い。

## ロッククラス {#lock-classes}

lockdepは個々のロックインスタンスではなく、**ロッククラス**を追跡する。クラスは「このキーで初期化されたすべてのロック」を表す。たとえば、`struct inode->i_mutex`型のすべての`mutex`インスタンスは、inodeが何百万個あっても同じクラスに属する。

```c
/* 静的に初期化されたロックは、それぞれ自動的に独自のクラスを得る */
DEFINE_MUTEX(my_mutex);  /* クラスキー = my_mutexのアドレス */

/* 動的なロック: 静的なキーを与えなければならない */
struct my_struct {
    struct mutex lock;
};

void init_my_struct(struct my_struct *s)
{
    /* すべてのmy_struct::lockインスタンスが1つのクラスキーを共有する */
    static struct lock_class_key my_lock_key;
    mutex_init(&s->lock);
    /* または明示的なクラスアノテーションで: */
    lockdep_set_class(&s->lock, &my_lock_key);
}
```

## 依存グラフ {#dependency-graph}

別のロックを保持したままロックが取得されるたびに、lockdepは自身の有向グラフに辺A → Bを追加する。循環依存（A → B → A）は潜在的なデッドロックを意味する。

```
Scenario:
  Thread 1: lock(A), lock(B)    → adds edge A→B
  Thread 2: lock(B), lock(A)    → adds edge B→A
                                 → CIRCULAR! lockdep reports a warning
```

警告は、スレッド2のパターンが初めて観測された瞬間に出る。スレッド1と2が実際には一度も並行して実行されなくてもである。

## lockdepの警告の読み方 {#reading-a-lockdep-warning}

```
======================================================
WARNING: possible circular locking dependency detected
6.5.0 #1 Not tainted
------------------------------------------------------
thread/1234 is trying to acquire lock:
ffffffff82345678 (&a_lock){+.+.}-{3:3}, at: func_a+0x12/0x34

but task is already holding:
ffffffff82345690 (&b_lock){+.+.}-{3:3}, at: func_b+0x56/0x78

which lock already depends on the new lock.

the existing dependency chain (in reverse order) is:
-> #1 (&b_lock){+.+.}-{3:3}:
       lock_acquire at kernel/locking/lockdep.c:5543
       _raw_spin_lock at kernel/locking/spinlock.c:151
       func_b+0x56/0x78
-> #0 (&a_lock){+.+.}-{3:3}:
       lock_acquire at kernel/locking/lockdep.c:5543
       _raw_spin_lock at kernel/locking/spinlock.c:151
       func_a+0x12/0x34
```

主要なフィールド:

- ロック種別のアノテーション`{+.+.}`は、どこで使われているかを示す。`+` = IRQ有効状態で使用、`-` = IRQ無効状態で使用、`.` = そのコンテキストでは未使用
- 依存チェーンは、その依存関係を作った正確な呼び出しパスを示す

## ロックのアノテーション {#lock-annotations}

lockdepは、コード内で使うアノテーションを提供している。

```c
/* ロックが保持されていることを表明する（保持されていなければWARN） */
lockdep_assert_held(&my_mutex);

/* 読み込み用に保持されていることを表明する */
lockdep_assert_held_read(&my_rwsem);

/* 書き込み用に保持されていることを表明する */
lockdep_assert_held_write(&my_rwsem);

/* ロックが1つも保持されていないことを表明する（スリープする関数向け） */
might_sleep();    /* アトミックコンテキストなら警告する */

/* スリープしてはならないコードを示す */
cant_sleep();

/* RCU読み側クリティカルセクション内にいることを示す */
lockdep_assert_in_rcu_read_lock();
```

これらはlockdepが無効（`CONFIG_PROVE_LOCKING=n`）のときは何もしないので、本番のコードに残しておいても安全である。

## ネストしたロックのためのサブクラス {#subclasses-for-nested-locks}

同じクラスの2つのロックを正当な理由で順に取る場合（例: ツリー走査で親をロックしてから子をロックする）、これが意図的でありデッドロックではないことをlockdepが理解するには、サブクラスのアノテーションが必要である。

```c
/* 各ノードが同じクラスの独自のロックを持つツリー */
struct tree_node {
    struct mutex lock;
    struct tree_node *child;
};

void lock_parent_then_child(struct tree_node *parent,
                             struct tree_node *child)
{
    /* サブクラスがないと、lockdepはこれを循環として報告する */
    mutex_lock(&parent->lock);
    mutex_lock_nested(&child->lock, 1);  /* subclass=1 = 子の階層 */
    /* ... */
    mutex_unlock(&child->lock);
    mutex_unlock(&parent->lock);
}
```

`_nested`版（`mutex_lock_nested`、`spin_lock_nested`、`down_read_nested`など）はサブクラス引数を取り、lockdepはそれを使ってツリーの異なる深さでのロック順序を区別する。

## /proc/lockdep {#proclockdep}

lockdepが有効なとき、そのデータが公開される。

```bash
# 既知のすべてのロッククラス
cat /proc/lockdep

# 依存グラフ
cat /proc/lockdep_chains

# 統計（使用中のロッククラス、依存チェーンなど）
cat /proc/lockdep_stats
# lock-classes: 2341
# direct dependencies: 8234
# indirect dependencies: 12456
# all direct dependencies: 23456
```

## ロック統計（/proc/lock_stat） {#lock-stats-proclock_stat}

`CONFIG_LOCK_STAT`があれば、カーネルはロッククラスごとの競合を追跡する。

```bash
cat /proc/lock_stat
# class name    con-bounces   contentions   waittime-min   waittime-max   waittime-total  acq-bounces   acquisitions  holdtime-min   holdtime-max   holdtime-total
# &mm->mmap_lock:
#                      5623        5623              0.26        1234.56       123456.78      234567         2345678          0.12         456.78       23456789.12
```

列:

- `contentions`: 取得時にロックがすでに保持されていた回数
- `waittime-*`: 呼び出し側が待った時間（マイクロ秒）
- `acquisitions`: ロック取得の合計回数
- `holdtime-*`: ロックが保持されていた時間

`contentions`と`waittime-max`が高い場合は、ロックのホットスポットを示している。

## 性能への影響 {#performance-impact}

lockdepは高コストである。完全な依存グラフをメモリ上に保持し、すべてのロック取得を検証する。本番用ではなく、開発・テスト用のカーネルを意図したものである。

- メモリ: ロッククラスのデータ構造に約60MB
- CPU: ロックの取得/解放のたびにlockdepの検証を通る

本番用には`CONFIG_PROVE_LOCKING=n`でコンパイルする。`lockdep_assert_held()`アノテーションは何もしないものとして残る。

## 参考文献 {#further-reading}

- [ロック競合のデバッグ](lock-debugging.md) — lock_statとperfを使ってホットスポットを見つける
- [spinlock](spinlock.md) — lockdepが検証する対象
- `Documentation/locking/lockdep-design.rst` — lockdepの内部
