# lockdepの実践 {#lockdep-in-practice}

> ロック依存関係の追跡: splatの読み方、誤検知へのアノテーション、ありえないはずのデッドロックの追跡

## このページの目的 {#why-this-page-exists}

`docs/locking/lockdep.md`ではlockdepの基礎、つまりlockdepとは何か、依存関係グラフがどう機能するか、ロッククラスについて扱っている。本ページは**実践的なデバッグ**に焦点を当てる: 実際のsplatの読み方、誤検知の解消、正しいアノテーションの書き方、そしてlockdepの限界の理解である。

---

## lockdep splatの構造 {#anatomy-of-a-lockdep-splat}

lockdepはデッドロックの可能性を検出すると、`WARNING: possible circular locking dependency detected`というメッセージを出力する。以下に完全な注釈付きの例を示す:

```
======================================================
WARNING: possible circular locking dependency detected
6.8.0-rc3 #1 Not tainted
------------------------------------------------------
kworker/u4:2/345 is trying to acquire lock:             ← (1) タスク + 獲得しようとしているロック
ffff888100ab1230 (&sb->s_type->i_mutex_key#3){+.+.}-{3:3}, at: inode_lock+0x1d/0x30

but task is already holding:                            ← (2) すでに保持しているロック
ffff888100ab1278 (&ei->i_data_sem){++++}-{3:3}, at: ext4_write_begin+0x1f0/0x8c0

which lock already depends on the new lock.             ← (3) 衝突の検出

the existing dependency chain (in reverse order) is:   ← (4) 循環
-> #1 (&ei->i_data_sem){++++}-{3:3}:
       lock_acquire+0xd8/0x390
       down_write+0x52/0x120
       ext4_write_begin+0x1f0/0x8c0
       generic_perform_write+0x10a/0x2a0

-> #0 (&sb->s_type->i_mutex_key#3){+.+.}-{3:3}:
       lock_acquire+0xd8/0x390
       down_write+0x52/0x120
       inode_lock+0x1d/0x30
       do_truncate+0x91/0x160

other info that might help us debug this:              ← (5) 「誰が何を保持しているか」
 Possible unsafe locking scenario:
   CPU0                        CPU1
   ----                        ----
   lock(&ei->i_data_sem);
                               lock(&sb->s_type->i_mutex_key#3);
                               lock(&ei->i_data_sem);
   lock(&sb->s_type->i_mutex_key#3);
   *** DEADLOCK ***
```

**4つの部分**:

1. **タスク + 獲得しようとしているもの** — タスク名、PID、およびロックのアドレス + クラス名 + アノテーション + 呼び出し箇所
2. **現在保持しているロック** — このタスクがすでに保持しているロックの連なり（複数の場合もある）
3. **衝突の宣言** — この新たな獲得が循環を生むことをlockdepが述べている
4. **逆順の依存関係チェーン** — `#1`が循環の最初のロック、`#0`が獲得しようとしているロックであり、両者で循環経路を示す
5. **CPU0/CPU1のシナリオ** — デッドロックを起こす具体的なインターリーブをlockdepが書き出している

---

## splat中のロックのアノテーション {#lock-annotations-in-the-splat}

splat中の各ロックには`{+.+.}-{3:3}`のようなアノテーションが付く:

```
{+.+.}  = 4つのコンテキストでの使用状況（hardirq-on、hardirq-off、softirq-on、softirq-off）
           + = このコンテキストで使用された
           . = このコンテキストでは使用されていない
           - = このコンテキストでIRQ無効状態で使用された

-{3:3}  = ロックの深さクラス: 3 = mutex（spinlockでもrwsemの読み取り側でもない）
           書式は{hard-depth:soft-depth}
```

よく見るアノテーション:

- `{+.+.}` — IRQ有効のプロセスコンテキストで取得されたmutex
- `{-.-.}` — IRQ無効で取得されたspinlock
- `{++++}` — 4つのコンテキストすべてで取得されたrwsem（例: i_data_sem）

---

## ロッククラスと`lock_class_key` {#lock-classes-and-lock_class_key}

lockdepは個々のインスタンスではなく**クラス**を追跡する。同じ種類のロックのインスタンスはすべて1つのクラスを共有し、lockdepはクラス間のエッジだけを追跡する。

```c
/* 静的なロックは、ロック自身のアドレスからクラスキーを得る: */
static DEFINE_SPINLOCK(my_lock);
/* → クラスキー = &my_lock、クラス名 = "my_lock" */

/* 動的に割り当てられたロック: すべてのインスタンスが1つのクラスキーを共有する */
struct my_obj {
    spinlock_t lock;
};

void init_my_obj(struct my_obj *obj)
{
    spin_lock_init(&obj->lock);
    /* すべてのobj->lockインスタンスは単一のクラスを共有する。そのクラスは
       マクロ展開でspin_lock_init()に埋め込まれた静的な__keyで識別される */
}
```

クラスキーはカーネルの`.data`セクションに置かれる`struct lock_class_key`である。これは`spin_lock_init()`や`mutex_init()`などによって、マクロを介して自動的に生成される（ここでは非`PREEMPT_RT`、`CONFIG_DEBUG_SPINLOCK=y`の経路を示す。実際の初期化呼び出しは単なる`__spin_lock_init()`ではなく`__raw_spin_lock_init()`であり、ロック待ちの種類を示すタグも渡す）:

```c
/* include/linux/spinlock.h: */
#define spin_lock_init(lock)                                   \
do {                                                           \
    static struct lock_class_key __key;                        \
    __raw_spin_lock_init(spinlock_check(lock), #lock, &__key,   \
                          LD_WAIT_CONFIG);                      \
} while (0)
```

この`static struct lock_class_key __key`がクラスキーであり、インスタンスごとではなく、ソースコード上の呼び出し箇所ごとに1つ存在する。

---

## クラスを分割して誤検知を解消する {#splitting-classes-to-resolve-false-positives}

同じ種類のロックの2つのインスタンスが、正当に異なる有効な順序を持つために、lockdepが誤検知を報告することがある。典型例はファイルシステムにおける**入れ子になったディレクトリロック**である:

```
Thread 1: rename(parent_dir/a → parent_dir/b)
  → lock(parent_inode), then lock(child_inode_a), lock(child_inode_b)

Thread 2: rename(child_dir/x → parent_dir/y)
  → lock(child_inode_x), then lock(parent_inode)
```

lockdepには、ある順序での`i_mutex` → `i_mutex`と、逆順での`i_mutex` → `i_mutex`が見える。これは誤検知である。「常に親を子より先に」という不変条件は安全だからである。

**解決策1: `lockdep_set_subclass()`**

```c
/* このロックインスタンスがサブクラスNであることをlockdepに伝える（0 = デフォルト）: */
lockdep_set_subclass(&inode->i_rwsem, I_MUTEX_PARENT);   /* サブクラス0 */
lockdep_set_subclass(&inode->i_rwsem, I_MUTEX_CHILD);    /* サブクラス1 */

/* これでlockdepはPARENT→CHILDを許可し、CHILD→PARENTを報告する */
/* サブクラスはinclude/linux/fs.hで定義されている: */
enum {
    I_MUTEX_NORMAL,
    I_MUTEX_PARENT,
    I_MUTEX_CHILD,
    I_MUTEX_XATTR,
    I_MUTEX_NONDIR2,
    /* ... */
};
```

**解決策2: 別のキーを使った`lockdep_init_map()`**

```c
static struct lock_class_key dir_lock_key;
static struct lock_class_key file_lock_key;

void init_inode(struct inode *inode, bool is_dir)
{
    struct lock_class_key *key = is_dir ? &dir_lock_key : &file_lock_key;
    lockdep_init_map(&inode->i_rwsem.dep_map, "i_rwsem", key, 0);
}
/* これでディレクトリとファイルは別々のロッククラスを持つ */
```

明確な入れ子レベル（親/子）がある場合は`lockdep_set_subclass()`が好まれる。2つのロック群がまったく独立した順序要件を持つ場合は、別々のキーを使う。

---

## 実行時アサーション {#runtime-assertions}

### `lockdep_assert_held()` {#lockdep_assert_held}

現在のタスクが特定のロックを保持していることをアサートする。呼び出し元がロックを保持している必要がある関数の先頭で使う:

```c
/* fs/inode.c */
static void inode_pin_lru_isolating(struct inode *inode)
{
    lockdep_assert_held(&inode->i_lock);   /* 呼び出し元はi_lockを保持していなければならない */
    WARN_ON(inode_state_read(inode) & (I_LRU_ISOLATING | I_FREEING | I_WILL_FREE));
    inode_state_set(inode, I_LRU_ISOLATING);
}
```

アサーションが失敗すると（ロックが保持されていないと）、lockdepは`WARNING`とスタックトレースを出力する。`CONFIG_PROVE_LOCKING=n`の場合、このマクロはコンパイル時に消える。

### `lockdep_assert_not_held()` {#lockdep_assert_not_held}

ロックが保持されて**いない**ことをアサートする。特定のロックの下では呼び出せない関数に使う:

```c
/* mmセマフォを保持したまま呼び出してはならない — この関数は内部でそれを
   獲得しようとする可能性があるため、デッドロックする: */
void some_mm_operation(struct mm_struct *mm)
{
    lockdep_assert_not_held(&mm->mmap_lock);
    /* ... */
}
```

### `lockdep_assert_held_write()` / `lockdep_assert_held_read()` {#lockdep_assert_held_write-lockdep_assert_held_read}

rwsemの場合は、特定のロックモードをアサートする:

```c
/* mm/vma.c — vma_adjust()はもう存在しない。VMAのマージ/分割/調整はすべて
   現在はvma_merge_existing_range()とその兄弟関数を経由する: */
static __must_check struct vm_area_struct *vma_merge_existing_range(
        struct vma_merge_struct *vmg)
{
    mmap_assert_write_locked(vmg->mm);
    /* ... マージ/調整のロジック ... */
}
```

`mmap_assert_write_locked()`（`include/linux/mmap_lock.h`）自体は薄いラッパーにすぎない。`mmap_lock`は普通のrwsemで実装されているので、単に`rwsem_assert_held_write(&mm->mmap_lock)`を呼ぶだけである。

---

## lockdepに対応した独自プリミティブの実装 {#implementing-custom-primitives-with-lockdep}

独自のロックプリミティブ（例: 手動でメモリ順序を管理するロックフリー構造や、別の何かをラップするオブジェクトごとのロック）を書く場合、それをlockdepから見えるようにできる:

```c
struct my_lock {
    atomic_t state;
    struct lockdep_map dep_map;   /* lockdepのメタデータを埋め込む */
};

static struct lock_class_key my_lock_key;

void my_lock_init(struct my_lock *l, const char *name)
{
    atomic_set(&l->state, 0);
    lockdep_init_map(&l->dep_map, name, &my_lock_key, 0);
}

void my_lock_acquire(struct my_lock *l)
{
    /* lockdepに伝える: 排他的な獲得、入れ子なし、リターンアドレス */
    lock_acquire(&l->dep_map, 0, 0, 0, 1, NULL, _RET_IP_);

    /* ... 実際の獲得ロジック ... */
    while (atomic_cmpxchg(&l->state, 0, 1) != 0)
        cpu_relax();
}

void my_lock_release(struct my_lock *l)
{
    atomic_set(&l->state, 0);
    lock_release(&l->dep_map, _RET_IP_);
}
```

`lock_acquire()`と`lock_release()`は、`spin_lock()`などが内部で（`kernel/locking/lockdep.c`において）呼び出すlockdepのフックである。これらを直接使うと、独自のプリミティブが依存関係グラフに組み込まれる。

---

## 誤検知の抑制（控えめに使うこと） {#suppressing-false-positives-use-sparingly}

### `lockdep_set_novalidate_class()` {#lockdep_set_novalidate_class}

ロッククラスを検証の対象外としてマークする。**どうしても必要な場合以外は避けること**。誤検知とともに本物のバグも隠してしまう:

```c
lockdep_set_novalidate_class(&my_lock);
```

### `lockdep_off()` / `lockdep_on()` {#lockdep_off-lockdep_on}

正当に順序に違反するコード領域（例: 意図的に循環を作るセルフテスト）で、lockdepを一時的に停止する:

```c
lockdep_off();
/* lockdepの誤った警告を引き起こすコード */
lockdep_on();
```

これらはプロセスローカルである（タスクごとのカウンタを調整する）。本物のロックのバグを取り繕うために使ってはならない。

---

## 設定 {#configuration}

```bash
CONFIG_PROVE_LOCKING=y          # lockdepを有効化（必須）
CONFIG_DEBUG_LOCK_ALLOC=y       # 初期化されていないロックの使用を検出
CONFIG_LOCKDEP=y                # PROVE_LOCKINGによって自動的に選択される
CONFIG_DEBUG_LOCKDEP=y          # lockdep自身の追加チェック（非常に遅い）
CONFIG_LOCK_STAT=y              # /proc/lock_statを有効化
```

---

## lockdepの上限 {#lockdep-limits}

lockdepはコンパイル時に上限が決まるデータ構造を管理している。`MAX_LOCKDEP_KEYS`は固定の上限であり、それ以外はKconfigで調整できる（`CONFIG_LOCKDEP_BITS`、`CONFIG_LOCKDEP_CHAINS_BITS`、`CONFIG_LOCKDEP_STACK_TRACE_BITS`）。それぞれのデフォルトは以下に示す値である（`CONFIG_KASAN`のもとでは`MAX_STACK_TRACE_ENTRIES`のデフォルトはより大きい2Mになる）:

```c
/* include/linux/lockdep_types.h, kernel/locking/lockdep_internals.h */
#define MAX_LOCKDEP_KEYS        8192   /* ロッククラスの最大数（固定） */
#define MAX_LOCKDEP_ENTRIES     (1UL << CONFIG_LOCKDEP_BITS)         /* デフォルト32768 */
#define MAX_LOCKDEP_CHAINS      (1UL << CONFIG_LOCKDEP_CHAINS_BITS)  /* デフォルト65536 */
#define MAX_STACK_TRACE_ENTRIES (1UL << CONFIG_LOCKDEP_STACK_TRACE_BITS)  /* デフォルト524288 */
```

テーブルが一杯になると、新しいロッククラスは黙って捨てられ、lockdepはそれらを追跡しなくなる。実行時に使用状況を確認するには:

```bash
cat /proc/lockdep_stats
```

```
lock-classes:                          3456 [max: 8192]
direct dependencies:                  12345 [max: 32768]
indirect dependencies:                56789
all direct dependencies:              23456
dependency chains:                     8901 [max: 65536]
stack-trace entries:                 234567 [max: 524288]
```

`lock-classes`が上限に近い場合は、ドライバの少ない設定で起動するか、ソース中の`MAX_LOCKDEP_KEYS`を増やすことを検討する。

---

## ロック競合の統計 {#lock-contention-statistics}

`CONFIG_LOCK_STAT=y`はロックごとの競合カウンタを有効にする。実行時に有効化するには:

```bash
echo 1 > /proc/sys/kernel/lock_stat
```

統計を読むには:

```bash
cat /proc/lock_stat
```

```
lock_stat version 0.4
---------------------------------------------------------------
                              class name    con-bounces    contentions   waittime-min   waittime-max waittime-total   acq-bounces   acquisitions   holdtime-min   holdtime-max holdtime-total
---------------------------------------------------------------
                        &sb->s_type->i_mutex_key#3:            0             23           0.27          143.50        1423.10           5432        1234567          0.10           5.30        8901.20
```

主な列:

- **con-bounces** — ロックが競合し、獲得から解放までの間にCPUが入れ替わった
- **contentions** — タスクがこのロックを待たなければならなかった総回数
- **waittime-min/max/total** — 待ちに費やした時間（マイクロ秒）
- **acq-bounces** — ロックは獲得されたが、保持者が解放前に別のCPUへ移動した
- **acquisitions** — 獲得に成功した総回数

統計をリセットするには:

```bash
echo 0 > /proc/lock_stat
```

特定のロッククラスで`contentions`が多い場合はボトルネックを示す。`waittime-max`が大きい場合は、まれだが非常に長いストール（優先度逆転や長いクリティカルセクションが原因であることが多い）を示す。

---

## ワークフロー: lockdep splatのデバッグ {#workflow-debugging-a-lockdep-splat}

```bash
# 1. splat全体を取得する（長くなることがある）:
dmesg | grep -A 80 "possible circular locking"

# 2. 循環に含まれる2つのロッククラスを特定する:
#    チェーン中の"-> #1"と"-> #0"を探す

# 3. 各ロックが獲得される場所を見つける:
#    各チェーンエントリにはコールスタックがある — addr2lineを使う:
./scripts/faddr2line vmlinux ext4_write_begin+0x1f0/0x8c0

# 4. 依存関係グラフを紙に描く:
#    クラスA → クラスB（どこかでこの順序で獲得される）
#    クラスB → クラスA（別の場所でこの順序で獲得される）
#    逆転している箇所を見つける

# 5. 修正の選択肢:
#    a. どこでも一貫した順序で獲得するよう並べ替える
#    b. もう一方を獲得する前に一方のロックを解放する（安全なら）
#    c. ブロックを避けるためtrylock + リトライループを使う
#    d. lockdep_set_subclass()でクラスを分割する（誤検知の場合）
#    e. lockdep_assert_held()を使って正しい呼び出し順序を強制する

# 6. 検証: リビルドして起動する。splatが再び現れなければよい
#    （lockdepは一意な循環ごとに起動あたり最大1回しか出力しない）
```

---

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/locking/lockdep.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/locking/lockdep.c) — バリデータの実装。`lock_acquire()`と`lock_release()`は、`spin_lock()`/`mutex_lock()`などが内部で呼ぶフックである
- [kernel/locking/lockdep_internals.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/locking/lockdep_internals.h) — `MAX_LOCKDEP_ENTRIES`、`MAX_LOCKDEP_CHAINS`、`MAX_STACK_TRACE_ENTRIES`。現在のカーネルでは、これらはコンパイル時に固定されるのではなく、`CONFIG_LOCKDEP_BITS`/`CONFIG_LOCKDEP_CHAINS_BITS`/`CONFIG_LOCKDEP_STACK_TRACE_BITS`のKconfigオプションから導出される
- [include/linux/lockdep_types.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/lockdep_types.h) — ロッククラス数の固定上限である`MAX_LOCKDEP_KEYS_BITS`/`MAX_LOCKDEP_KEYS`
- [kernel/locking/lockdep_proc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/locking/lockdep_proc.c) — `/proc/lockdep_stats`と`/proc/lock_stat`の出力整形（`lock-classes`、`con-bounces`、`contentions`、`waittime-min`など）
- [include/linux/fs.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/fs.h) — `lockdep_set_subclass()`とともに使われる`I_MUTEX_NORMAL`/`I_MUTEX_PARENT`/`I_MUTEX_CHILD`/`I_MUTEX_XATTR`/`I_MUTEX_NONDIR2`サブクラスのenum
- [include/linux/spinlock.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/spinlock.h) — 呼び出し箇所ごとの`lock_class_key`を確保する`spin_lock_init()`
- [fs/inode.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/inode.c) — `inode_pin_lru_isolating()`。実際の`lockdep_assert_held(&inode->i_lock)`の呼び出し箇所
- [mm/vma.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vma.c) — `vma_merge_existing_range()`。`mmap_assert_write_locked()`をアサートする現在のVMAマージ経路
- [include/linux/mmap_lock.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mmap_lock.h) — `mmap_assert_write_locked()`/`mmap_assert_locked()`
- [scripts/faddr2line](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/faddr2line) — splat中の`func+0x123/0x456`形式のオフセットをソース行に解決する
- [Documentation/locking/lockdep-design.rst](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/locking/lockdep-design.rst) — lockdep自身の設計文書
- [Documentation/locking/lockstat.rst](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/locking/lockstat.rst) — `/proc/lock_stat`の設計文書

### 関連ページ {#related-pages}

- [lockdepの基礎](../locking/lockdep.md) — 依存関係グラフ、ロッククラス、基本的なsplatの読み方
- [ロック競合のデバッグ](../locking/lock-debugging.md) — `/proc/lock_stat`、perf lock
- [KASANとKFENCE](kasan-kfence.md) — 相補的なメモリエラー検出器
- [KCSAN](kcsan.md) — データ競合検出

### LWN記事 {#lwn-articles}

- [Enhancing lockdep with crossrelease](https://lwn.net/Articles/709849/) — Byungchul Parkが、crossrelease拡張を提案する前に、lockdepの依存関係グラフの基礎（ロッククラス、循環検出）を説明している（2016年12月）
- [The dependency tracker for complex deadlock detection](https://lwn.net/Articles/1036222/) — lockdepの代替として提案されたDEPTに関するJonathan Corbetの記事。lockdepが捕まえられるもの・捕まえられないものを説明している（2025年9月）

### 外部資料 {#external}

- [docs.kernel.org: Runtime locking correctness validator](https://docs.kernel.org/locking/lockdep-design.html) — lockdep-design.rstのレンダリング版
- [docs.kernel.org: Lock Statistics](https://docs.kernel.org/locking/lockstat.html) — lockstat.rstのレンダリング版
