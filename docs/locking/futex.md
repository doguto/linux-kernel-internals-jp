# futexの内部 {#futex-internals}

> ユーザー空間のmutexや条件変数の実装がカーネル内でどう動くか

## なぜfutexが存在するか {#why-futexes-exist}

futex以前は、POSIX mutexのすべての操作がカーネルを経由していた。`pthread_mutex_lock()`は`sem_wait()`などのシステムコールを呼び、`pthread_mutex_unlock()`は`sem_post()`を呼んでいた。システムコールのコストは現代のハードウェアで約100〜300 nsである。細粒度のmutexを短時間だけ保持するプログラム（たとえば、数回のキャッシュアクセスの間だけ保持するハッシュテーブルのロック）は、実際の仕事よりもカーネル内で多くの時間を費やしかねなかった。

Hubertus Franke（IBM）、Rusty Russell、Matthew Kirkwoodは、このオーバーヘッドをなくすために2002年にfutexを設計した。Ottawa Linux Symposium 2002において["Fuss, Futexes and Furwocks: Fast Userlevel Locking in Linux"](https://www.kernel.org/doc/ols/2002/ols2002-pages-479-495.pdf)として発表されている。（Ulrich Drepperは後に、glibcの`pthread_mutex_t`をfutexの*上に*設計した。Ingo Molnárは初期の競合状態の修正に貢献した。）重要な観察は、**ロックが高コストになるのは競合があるときだけ**ということである。他に待っているスレッドがいなければ、ロック/アンロックの操作全体をユーザー空間での2つのアトミック命令で済ませられる。カーネルを呼ぶ必要があるのは、スレッドを待機させたり再開させたりするときだけである。

futex(2)はLinux 2.6.0（2003年12月）で導入された。[`man 2 futex`](https://www.man7.org/linux/man-pages/man2/futex.2.html)を参照のこと。今日では、glibcの`pthread_mutex_t`、`pthread_cond_t`、`pthread_rwlock_t`、`sem_t`、そしてC++の`std::mutex`はすべてfutexの上に実装されている。

## futexとは何か {#what-is-a-futex}

**futex**（fast userspace mutex）は、ユーザー空間のロックプリミティブ（pthreadsのmutex、条件変数、セマフォ）を効率的にするカーネルの仕組みである。重要な洞察は、競合のないよくあるケースでは、ロックとアンロックはカーネルを一切関与させずに完全にユーザー空間で行われるべきだということである。カーネルが呼ばれるのは、実際に競合があるときだけである。

futexシステムコール（`futex(2)`）は、2つの中核となる操作を提供する。

- `FUTEX_WAIT`: futexワードが指定された値ならスリープする
- `FUTEX_WAKE`: 1つ以上の待機者を起床させる

## 競合のない高速パス {#the-uncontended-fast-path}

たとえばpthreadのmutexは、ユーザー空間メモリ上の32ビット整数の上に実装されている。

```c
/* 簡略化したpthread mutex */
int mutex_word = 0;  /* 0 = アンロック、1 = ロック、2 = 待機者ありでロック */

void pthread_mutex_lock(int *mutex_word)
{
    /* 高速パス: アトミックなCASでロックの取得を試みる */
    if (atomic_cmpxchg(mutex_word, 0, 1) == 0)
        return;  /* 取得できた。カーネル呼び出しなし */

    /* 低速パス: 競合があるので、カーネルを呼ぶ */
    do {
        /* 待機者がいることを示す */
        atomic_set(mutex_word, 2);
        /* 値が != 2 になるまでスリープする */
        futex(mutex_word, FUTEX_WAIT, 2, NULL, NULL, 0);
    } while (atomic_cmpxchg(mutex_word, 0, 2) != 0);
}

void pthread_mutex_unlock(int *mutex_word)
{
    /* 高速パス: 待機者なし */
    if (atomic_xchg(mutex_word, 0) == 1)
        return;  /* クリアするだけ。カーネル呼び出しなし */

    /* 低速パス: 待機者がいるので、1つ起床させる */
    futex(mutex_word, FUTEX_WAKE, 1, NULL, NULL, 0);
}
```

カーネルが呼ばれるのは次の場合だけである。
1. ロック時: ロックがすでに保持されている（誰かがスリープする必要がある）
2. アンロック時: 起床させる必要のある待機者がいる

競合の少ないmutexでは、ほとんどのロック/アンロック操作は、2つのアトミック命令によって完全にユーザー空間で完了する。

## カーネル側のデータ構造 {#kernel-side-data-structures}

```c
/* kernel/futex/futex.h */

/* 特定のfutex（メモリ上のどこにあるか）を識別する */
union futex_key {
    struct {
        u64 ptr;          /* プライベートfutexの場合: mmポインタ */
        unsigned long word;  /* ページオフセット + ワードオフセット */
    } private;
    struct {
        u64 ptr;          /* 共有futexの場合: mappingポインタ */
        unsigned long pgoff; /* ファイル内のページオフセット */
        unsigned int offset;
    } shared;
    struct {
        u64 ptr;
        unsigned long word;
    } both;
};

/* スリープ中の待機者ごとに1エントリ */
struct futex_q {
    struct plist_node list;       /* ハッシュバケット内で優先度順に並ぶ */
    struct task_struct *task;     /* スリープ中のタスク */
    spinlock_t *lock_ptr;         /* ハッシュバケットのロック */
    union futex_key key;          /* 待っているfutex */
    struct futex_pi_state *pi_state; /* 優先度継承の状態 */
    u32 bitset;                   /* FUTEX_WAIT_BITSET用 */
};

/* ハッシュテーブルのバケット: 同じハッシュを持つすべてのfutexがバケットを共有する */
struct futex_hash_bucket {
    atomic_t waiters;
    spinlock_t lock;
    struct plist_head chain;      /* futex_qエントリのリスト */
} ____cacheline_aligned_in_smp;
```

ハッシュテーブルは`1 << CONFIG_FUTEX_HASHBITS`個のバケットを持つ（ほとんどの構成でデフォルト256）。複数のfutexが1つのバケットを共有することがあり、起床時にはキーを使って特定のfutexを識別する。

## プライベートfutexと共有futex {#private-vs-shared-futexes}

```c
/* プライベートfutex（プロセス内専用。glibcのデフォルト） */
futex(&word, FUTEX_WAIT | FUTEX_PRIVATE_FLAG, expected, timeout, NULL, 0);

/* 共有futex（共有メモリ上に置けて、プロセスをまたいでアクセス可能） */
futex(&word, FUTEX_WAIT, expected, timeout, NULL, 0);
```

プライベートfutexは、キーが単なる仮想アドレスであるため、わずかに高速である。ページテーブルのウォークが不要だからである。共有futexは、同じページをマップする複数のプロセスが同じfutexを見られるように、背後にあるページの物理的な位置（`page->mapping` + オフセット）を使う。

## FUTEX_WAIT: スリープに入る {#futex_wait-going-to-sleep}

```c
/* 簡略化したカーネルのFUTEX_WAIT実装 */
int futex_wait(u32 __user *uaddr, u32 val, ...)
{
    struct futex_q q;
    struct futex_hash_bucket *hb;

    /* 1. キーを計算する */
    ret = get_futex_key(uaddr, &q.key);

    /* 2. ハッシュバケットをロックする */
    hb = hash_futex(&q.key);
    spin_lock(&hb->lock);

    /* 3. バケットのロック下で値をチェックする */
    ret = get_user(uval, uaddr);
    if (uval != val) {
        /* ここに来る前に値が変わった: スリープせずに戻る */
        spin_unlock(&hb->lock);
        return -EAGAIN;
    }

    /* 4. 自分をキューに入れる */
    plist_add(&q.list, &hb->chain);
    spin_unlock(&hb->lock);

    /* 5. スリープする */
    set_current_state(TASK_INTERRUPTIBLE);
    schedule();

    /* 6. 起床した: キューから外す */
    return 0;
}
```

決定的なステップは、**ハッシュバケットのロック下で**値をチェックすること（ステップ3）である。これにより次のような競合を防ぐ。

- ユーザー空間がロックの競合を検知してFUTEX_WAITを呼ぶ
- キューに入る前に、ロック保持者がアンロックしてFUTEX_WAKEを呼ぶ
- ロック下でのチェックがなければ、永遠にスリープしてしまう

## FUTEX_WAKE: スリープ中の者を起床させる {#futex_wake-waking-sleepers}

```c
int futex_wake(u32 __user *uaddr, int nr_wake)
{
    struct futex_hash_bucket *hb;
    struct futex_key key;
    struct futex_q *q;

    get_futex_key(uaddr, &key);
    hb = hash_futex(&key);

    spin_lock(&hb->lock);
    /* キーが一致する待機者を見つけて起床させる */
    plist_for_each_entry_safe(q, ..., &hb->chain, list) {
        if (futex_match(&q->key, &key)) {
            wake_up_process(q->task);
            if (--nr_wake == 0)
                break;
        }
    }
    spin_unlock(&hb->lock);
}
```

## PI futex {#pi-futexes}

優先度継承futex（`FUTEX_LOCK_PI`、`FUTEX_UNLOCK_PI`）は、ユーザー空間のmutexに優先度継承プロトコルを実装する。高優先度のタスクが低優先度のタスクの保持するPI futexでブロックすると、低優先度のタスクの実効優先度が引き上げられる。

カーネルは`futex_pi_state`構造体を管理し、これが`futex_q`を`rt_mutex`に結び付ける。これはカーネル内の`rt_mutex_t`が使うものと同じ優先度継承の仕組みである。

```c
/* ユーザー空間のPI mutex（簡略化） */
void pi_mutex_lock(int *uaddr)
{
    /* 取得を試みる: ワードに自分のTIDを設定する */
    int zero = 0;
    if (cmpxchg(uaddr, 0, gettid()) == 0)
        return;  /* 取得できた */
    /* ブロックする: カーネルがオーナーの優先度を引き上げる */
    futex(uaddr, FUTEX_LOCK_PI, 0, timeout, NULL, 0);
}
```

## 条件変数（FUTEX_WAIT + FUTEX_WAKE） {#condition-variables-futexwait-futexwake}

`pthread_cond_wait`は次のように実装されている。
1. 関連付けられたmutexを解放する（ユーザー空間のアトミック操作）
2. 条件変数のワードに対して`FUTEX_WAIT`する
3. 起床時: mutexを再取得する

`pthread_cond_signal` / `pthread_cond_broadcast`は、`FUTEX_WAKE`（signal）または`FUTEX_REQUEUE`（broadcast。待機者を全員起床させずに、mutexのキューへ効率よく移すため）を使う。

### なぜFUTEX_REQUEUEが存在するか {#why-futex_requeue-exists}

`pthread_cond_broadcast`の素朴な実装は、大きなカウントを指定した`FUTEX_WAKE`、つまり全待機者の起床である。問題は、N個の待機者全員が同時に起床し、全員が関連付けられたmutexの再取得を試み、そのうちN-1個がすぐにスリープに戻ることである。これが**thundering herd**である。

`FUTEX_CMP_REQUEUE`（Linux 2.6.7、2004年に追加。[`man 2 futex`](https://www.man7.org/linux/man-pages/man2/futex.2.html)）は、スリープ中のスレッドを*起床させずに*、条件変数のキューからmutexのキューへ移す。1つの待機者だけが（mutexを取得するために）起床させられ、残りはrequeueされる。最初の待機者がmutexを解放すると、requeueされた次の待機者が起床し、以下同様に続く。N個の待機者は、全員が一斉に競合するのではなく、順番に処理される。

## 参考文献 {#further-reading}

- [mutexとrt_mutex](mutex.md) — カーネル側のスリープするロック
- [優先度逆転とPI mutex](../sched/pi-mutexes.md) — rt_mutexのPIの仕組み
- `man 2 futex` — システムコールのインターフェース
- カーネルツリー内の`Documentation/locking/`
