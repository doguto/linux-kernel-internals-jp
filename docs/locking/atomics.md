# アトミック操作とメモリバリア {#atomic-operations-and-memory-barriers}

> より高レベルのあらゆる同期の土台となるプリミティブ

## アトミック操作はどんな問題を解決するか {#what-problem-do-atomics-solve}

現代のCPUでは、単純な`x++`は3つの操作にコンパイルされる。`x`のロード、1の加算、`x`へのストアである。2つのCPUがこれを同時に実行すると、両方が同じ値をロードし、一方のインクリメントが失われることがある。

```
CPU 0:  load x (gets 5)                store x = 6
CPU 1:              load x (gets 5)              store x = 6
Result: x = 6 (should be 7)
```

アトミック操作は、読み込み・変更・書き込みを単一の不可分な単位として実行するハードウェア命令である。他のCPUが中間状態を見ることはできない。

## atomic_tとatomic64_t {#atomict-and-atomic64t}

```c
/* include/linux/atomic.h */
typedef struct { int counter; } atomic_t;
typedef struct { s64 counter; } atomic64_t;

/* 基本操作 */
atomic_t a = ATOMIC_INIT(0);

atomic_read(&a);              /* 値を読む */
atomic_set(&a, 5);            /* 値を設定する */

atomic_inc(&a);               /* a++ */
atomic_dec(&a);               /* a-- */
atomic_add(n, &a);            /* a += n */
atomic_sub(n, &a);            /* a -= n */

/* 新しい値を返す */
int v = atomic_inc_return(&a);   /* ++a */
int v = atomic_dec_return(&a);   /* --a */
int v = atomic_add_return(n, &a);
int v = atomic_sub_return(n, &a);

/* 古い値を返す */
int old = atomic_fetch_add(n, &a);
int old = atomic_fetch_sub(n, &a);

/* テストして返す */
bool zero = atomic_dec_and_test(&a);  /* 結果 == 0 なら真を返す */
bool neg  = atomic_add_negative(n, &a); /* 結果 < 0 なら真を返す */

/* ビット演算 */
atomic_or(mask, &a);
atomic_and(mask, &a);
atomic_xor(mask, &a);
int old = atomic_fetch_or(mask, &a);
```

## compare-and-swap（cmpxchg） {#compare-and-swap-cmpxchg}

最も強力なアトミック操作である。値を期待値とアトミックに比較し、等しければ入れ替える。

```c
/* *ptr == expected なら成功し（old == expected を返す）、*ptr = new に設定する */
old = atomic_cmpxchg(&a, expected, new);

/* 入れ替えが起きたら真を返す */
bool success = atomic_try_cmpxchg(&a, &expected, new);
```

これはロックフリーなデータ構造の基本構成要素である。たとえば、ブロックしないpushは次のようになる。

```c
struct node {
    struct node *next;
    int value;
};
static atomic_long_t head;  /* 実際には: struct node __rcu *head */

void push(struct node *node)
{
    struct node *old_head;
    do {
        old_head = (struct node *)atomic_long_read(&head);
        node->next = old_head;
    } while (atomic_long_cmpxchg(&head,
                                  (long)old_head,
                                  (long)node) != (long)old_head);
}
```

## atomic_long_t {#atomiclongt}

`atomic_long_t`はポインタサイズのアトミック型であり、すべてのアーキテクチャでポインタに収まる必要がある64ビット値に便利である。

```c
atomic_long_t v = ATOMIC_LONG_INIT(0);
atomic_long_read(&v);
atomic_long_set(&v, val);
atomic_long_add(n, &v);
atomic_long_cmpxchg(&v, old, new);
```

## READ_ONCEとWRITE_ONCE {#readonce-and-writeonce}

2つのCPUが同時に1つの変数にアクセスすることは、Cでは未定義動作である。`READ_ONCE`と`WRITE_ONCE`は、コンパイラが次のことをするのを防ぐ。

- アクセスのテアリング（ワードサイズの読み書きを複数に分割すること）
- アクセスの統合や並べ替え
- 関数呼び出しをまたいで値をレジスタにキャッシュすること

```c
/* READ_ONCE/WRITE_ONCEなし: コンパイラがキャッシュ、並べ替え、テアリングをしうる */
while (flag) { ... }  /* コンパイラはflagを一度だけロードして永久にループするかもしれない */

/* READ_ONCEあり: 毎回の反復で新たにロードする */
while (READ_ONCE(flag)) { ... }

/* WRITE_ONCE: 書き込みが単一の操作として見えることを保証する */
WRITE_ONCE(flag, 0);
```

これらは、共有変数へのあらゆるアクセスに必要な最低限のものである。CPU間の順序保証は提供しない。それはメモリバリアの役目である。

## メモリバリア {#memory-barriers}

現代のCPUとコンパイラは、性能のためにメモリアクセスを並べ替える。単一のCPU上ではこれは見えないが、CPU間では同期を壊しうる。

防ぐべき並べ替えには3種類ある。

```
Store-store: CPU may reorder two stores
Load-load:   CPU may reorder two loads
Store-load:  CPU may reorder a store followed by a load (most common)
```

### バリアの種類 {#barrier-types}

```c
/* フルバリア: この地点をまたぐ並べ替えをしない */
smp_mb();

/* 読み込みバリア: この地点をまたぐロードの並べ替えをしない */
smp_rmb();

/* 書き込みバリア: この地点をまたぐストアの並べ替えをしない */
smp_wmb();

/* acquire: これより後のロード/ストアはこれより前に移動されない */
smp_load_acquire(ptr);   /* ロード + acquireバリア */

/* release: これより前のロード/ストアはこれより後に移動されない */
smp_store_release(ptr, val);  /* releaseバリア + ストア */
```

### acquire-releaseパターン {#acquire-release-pattern}

最も一般的なバリアのパターンはacquire/releaseであり、生産者から消費者へデータを渡すのに使われる。

```c
/* 生産者 */
data = compute_result();        /* (1) データを準備する */
smp_store_release(&ready, 1);  /* (2) 公開: ready=1 より前にdataが見える */

/* 消費者 */
if (smp_load_acquire(&ready)) { /* (3) チェック: dataを読む前に ready=1 */
    use(data);                   /* (4) ここではdataが見えることが保証される */
}
```

(2)のreleaseバリアは、(1)が(2)より前に順序付けられることを保証する。(3)のacquireバリアは、(3)が(4)より前に順序付けられることを保証する。これらが合わさってhappens-before関係を形成する。

### アトミック操作の順序付けの種類 {#atomic-ordering-variants}

アトミック操作には4種類の順序付けがある。

```c
/* relaxed: 順序付けなし、アトミック性のみ */
atomic_add_relaxed(1, &counter);

/* acquire: これより後のロードは、対となるreleaseより前のストアを見る */
atomic_add_acquire(1, &counter);

/* release: これより前のストアは、対となるacquireより前に見える */
atomic_add_release(1, &counter);

/* full（デフォルト）: acquireとreleaseの両方のセマンティクス */
atomic_add(1, &counter);
```

順序付けが不要なカウンタにはrelaxedを使う。生産者・消費者の同期にはacquire/releaseを使う。デフォルト（`atomic_add`など）はフルバリアであり、安全だが必要以上に遅い可能性がある。

```c
/* 非アトミック操作の前後: アトミック操作に対する順序付けを保証する */
smp_mb__before_atomic();
/* 非アトミック操作 */
atomic_op();
/* 非アトミック操作 */
smp_mb__after_atomic();
```

## よくあるパターン {#common-patterns}

### 参照カウント（atomic_t） {#reference-counting-atomic_t}

```c
/* 単純なケースではkrefより簡単 */
static atomic_t refcount = ATOMIC_INIT(1);

void get(void) { atomic_inc(&refcount); }
void put(void) {
    if (atomic_dec_and_test(&refcount))
        free_object();
}
```

カーネルは、アトミック操作をよりわかりやすいセマンティクスでラップした`kref`（カーネルの参照カウント）を提供している。

### フラグ（ビットフィールドとしてのatomic_t） {#flags-atomic_t-as-bitfield}

```c
#define FLAG_BUSY    0
#define FLAG_READY   1

atomic_t flags = ATOMIC_INIT(0);

/* フラグを立てる */
atomic_or(BIT(FLAG_READY), &flags);

/* アトミックにテストしてクリアする */
if (atomic_fetch_andnot(BIT(FLAG_BUSY), &flags) & BIT(FLAG_BUSY)) {
    /* ビジーだったが、クリアされた */
}
```

ビット単位のアトミック操作として、カーネルは`unsigned long`に対する`set_bit()`、`clear_bit()`、`test_and_set_bit()`なども提供している。

## 参考文献 {#further-reading}

- [spinlock](spinlock.md) — アトミック操作とメモリバリアの上に構築されている
- [per-CPU変数](percpu.md) — CPUごとのデータについてアトミック操作を完全に避ける
- カーネルツリー内の`Documentation/memory-barriers.txt` — 決定版のリファレンス
- `Documentation/atomic_t.txt` — atomic_tのセマンティクス
