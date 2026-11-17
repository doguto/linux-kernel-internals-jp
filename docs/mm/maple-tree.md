# maple tree: VMA管理のためのデータ構造 {#maple-tree-vma-management-data-structure}

> Linux 6.1で、仮想メモリ領域の管理に使われていたrbtree＋連結リストを置き換えたBツリーの変種

## 主要なソースファイル {#key-source-files}

| ファイル | 役割 |
|------|---------|
| `include/linux/maple_tree.h` | 公開API: `struct maple_tree`、`struct ma_state`、すべてのマクロとインライン関数 |
| `lib/maple_tree.c` | 完全な実装、`maple_tree_init()`、`mas_walk()`、`mas_find()`、`mas_store()`、`mas_erase()` |
| `include/linux/mm_types.h` | `struct mm_struct`への埋め込み（`mm_mt`）、`struct vma_iterator`、`MM_MT_FLAGS` |
| `mm/mmap.c` | `find_vma()`、`find_vma_intersection()`、`__mt_destroy()`によるプロセスの破棄 |
| `mm/vma.c` | `vma_iter_store_gfp()`によるVMAの挿入、`vma_iter_clear()`による削除 |
| `kernel/fork.c` | `mm_init()`: `mt_init_flags()`によるmaple treeの初期化 |

---

## なぜmaple treeがrbtree＋連結リストを置き換えたか {#why-the-maple-tree-replaced-rbtree-linked-list}

Linux 6.1より前は、すべての`mm_struct`がVMAのために2つの冗長なデータ構造を保持していた。

- `vm_start`をキーとする **赤黒木**（`mm->mm_rb`）。アドレスによるO(log n)の検索を提供する。
- すべてのVMAをアドレス順につなぐ **双方向連結リスト**（`vma->vm_next` / `vma->vm_prev`）。順次走査に使われる。

VMAの挿入、削除、変更のたびに、`mmap_lock`の下で両方の構造をアトミックに更新しなければならなかった。コードの複雑さが増すだけでなく、具体的な性能コストが2つあった。

1. **順次走査時のキャッシュミス**。 連結リストをたどるには、VMAごとに1つずつ`vm_next`ポインタを追う必要があり、各ポインタは別のslabオブジェクト内の別のキャッシュラインを指している可能性がある。数千のマッピングを持つプロセスでは、これはポインタを追うロードの長い連鎖になる。

2. **メモリのオーバーヘッド**。 各`vm_area_struct`は3つのrbtreeノードフィールド（`rb_node`。ポインタに埋め込まれた親と色を含む）と2つのリストポインタを持っており、インデックス付けのためだけにVMAあたり約40バイトを費やしていた。

maple tree（v6.1でマージ。OracleのLiam HowlettとMatthew Wilcoxによる）は、両方の構造を単一のBツリーの変種に統合した。これは範囲をキーとする検索 *と* 効率的な順次走査の両方をネイティブにサポートする。Bツリーのノードは複数のピボットとスロットを256バイトのキャッシュラインに詰め込むため、1回のノードアクセスで複数のピボット比較ができる。比較のたびに別のノードをロードする必要があるrbtreeに比べて、キャッシュミスが劇的に減る。

---

## maple treeとは何か {#what-the-maple-tree-is}

maple treeは、RCU安全で範囲をキーとするBツリーである。unsigned longのキーの重ならない範囲を格納し、各範囲を不透明なポインタ値に対応付ける。VMA管理では、キーはアドレス範囲`[vm_start, vm_end - 1]`であり、値は`struct vm_area_struct *`である。

### ノードの種類とサイズ {#node-types-and-sizes}

3種類のノードが定義されている（`include/linux/maple_tree.h`より）。

```c
/* 64ビットでは: ノードあたり256バイト、256バイト境界にアライン */
#define MAPLE_NODE_SLOTS     31   /* 汎用ノードのスロット数 */
#define MAPLE_RANGE64_SLOTS  16   /* rangeノード: 15ピボット + 16スロット */
#define MAPLE_ARANGE64_SLOTS 10   /* alloc-rangeノード: 9ピボット + 10スロット + gap */
```

使われている2種類のノードは次のとおり。

**`maple_range_64` — 標準の内部/リーフノード**

```c
struct maple_range_64 {
    struct maple_pnode *parent;
    unsigned long pivot[MAPLE_RANGE64_SLOTS - 1]; /* 15個の範囲境界 */
    union {
        void __rcu *slot[MAPLE_RANGE64_SLOTS];    /* 16個の子または値のポインタ */
        struct {
            void __rcu *pad[MAPLE_RANGE64_SLOTS - 1];
            struct maple_metadata meta;            /* 終端オフセット + 最大のgap */
        };
    };
};
```

`maple_range_64`ノードは、256バイトに最大15個のピボットと16個のスロットを格納する。Bツリーの用語では、ピボットは各スロットに格納された範囲の上限（その値を含む）である。スロット *i* は範囲`(pivot[i-1], pivot[i]]`をカバーし、スロット0はノードの暗黙の最小値からカバーする。

**`maple_arange_64` — 割り当て範囲ノード**

```c
struct maple_arange_64 {
    struct maple_pnode *parent;
    unsigned long pivot[MAPLE_ARANGE64_SLOTS - 1]; /* 9個の範囲境界 */
    void __rcu *slot[MAPLE_ARANGE64_SLOTS];         /* 10個のポインタ */
    unsigned long gap[MAPLE_ARANGE64_SLOTS];        /* スロットごとのNULLの最大gap */
    struct maple_metadata meta;
};
```

割り当て範囲ノードは、さらに各サブツリー内のNULLエントリの最大の連続ギャップを追跡する。これにより`mas_empty_area()` / `mas_empty_area_rev()`がサポートされる。これらはツリー全体を走査することなく、指定サイズの空きアドレス範囲を見つける。`mmap()`のアドレス選択にとって極めて重要である。

### ノードの割り当て {#node-allocation}

すべてのノードは、起動時に`maple_tree_init()`によって作成される`maple_node`という名前の専用`kmem_cache`から割り当てられる。

```c
void __init maple_tree_init(void)
{
    struct kmem_cache_args args = {
        .align          = sizeof(struct maple_node),
        .sheaf_capacity = 32,
    };
    maple_node_cache = kmem_cache_create("maple_node",
            sizeof(struct maple_node), &args, SLAB_PANIC);
}
```

256バイトのアラインメントにより、ノードポインタの下位8ビットは常にゼロになる。maple treeはこれらのビットを使ってノードの種類とツリーのメタデータをポインタに直接エンコードし、余分なメモリアクセスを避けている。

### ツリーの最大の高さ {#maximum-tree-height}

```c
#define MAPLE_HEIGHT_MAX  31
```

`maple_range_64`ノードあたり16スロットであれば、高さ31のツリーは16^31個の異なる範囲を表現できる。これは現実的などのアドレス空間をもはるかに超える。

---

## struct maple_tree {#struct-maple_tree}

```c
/* include/linux/maple_tree.h */
struct maple_tree {
    union {
        spinlock_t          ma_lock;
#ifdef CONFIG_LOCKDEP
        struct lockdep_map *ma_external_lock;
#endif
    };
    unsigned int    ma_flags;
    void __rcu     *ma_root;
};
```

- **`ma_lock`** — 内部のspinlock。外部ロックが与えられない場合に使われる。`mm->mm_mt`ではこれは使われない。ツリーは`MT_FLAGS_LOCK_EXTERN`で構成され、`mmap_lock`が直列化のためのロックとして機能する。
- **`ma_flags`** — ツリー全体の性質を表すビットマスク（後述）。
- **`ma_root`** — ルートノードへのRCU保護されたポインタ（ツリーがインデックス0にちょうど1つのエントリしか持たない場合は、格納された単一の値）。

### フラグ {#flags}

```c
#define MT_FLAGS_ALLOC_RANGE   0x01  /* ノードごとに最大の空きgapを追跡 */
#define MT_FLAGS_USE_RCU       0x02  /* ノードはRCU経由で解放される */
#define MT_FLAGS_LOCK_IRQ      0x100 /* ma_lockをIRQセーフに取得 */
#define MT_FLAGS_LOCK_BH       0x200 /* ma_lockをBHセーフに取得 */
#define MT_FLAGS_LOCK_EXTERN   0x300 /* 外部ロック。ma_lockは未使用 */
```

### mm_structはmaple treeをどう埋め込んでいるか {#how-mm_struct-embeds-the-maple-tree}

```c
/* include/linux/mm_types.h */
struct mm_struct {
    ...
    struct maple_tree mm_mt;
    ...
    struct rw_semaphore mmap_lock;
    ...
};

/* mm->mm_mtの初期化時に使われるフラグ */
#define MM_MT_FLAGS  (MT_FLAGS_ALLOC_RANGE | MT_FLAGS_LOCK_EXTERN | \
                      MT_FLAGS_USE_RCU)
```

`mm->mm_mt`では3つのフラグすべてが有効である。

- `MT_FLAGS_ALLOC_RANGE` — ギャップの追跡を有効にし、`mmap()`が空き領域を効率的に見つけられるようにする。
- `MT_FLAGS_LOCK_EXTERN` — ツリー内部の`ma_lock` spinlockは迂回される。呼び出し側は、変更を行う前に`mmap_lock`を書き込み用に保持していなければならない。
- `MT_FLAGS_USE_RCU` — 削除されたノードは即座にではなく`call_rcu()`経由で解放され、ロックなしのRCUリーダーが可能になる（後述の[RCU安全性](#rcu-safety)を参照）。

ツリーは`mm_init()`（`kernel/fork.c`）で初期化され、`mmap_lock`に結び付けられる。

```c
static struct mm_struct *mm_init(struct mm_struct *mm, ...)
{
    mt_init_flags(&mm->mm_mt, MM_MT_FLAGS);
    mt_set_external_lock(&mm->mm_mt, &mm->mmap_lock);
    ...
}
```

---

## RCU安全性 {#rcu-safety}

maple treeは、ツリーから削除されたノードを、現在実行中のすべてのRCU読み出し側クリティカルセクションが完了するまで解放しないことで、ロックなしの読み出しを実現している。`MT_FLAGS_USE_RCU`がセットされていると、削除経路は即座に解放する代わりに`call_rcu()`を呼ぶ。

```c
/* lib/maple_tree.c — ノード削除の経路 */
if (mt_in_rcu(mt)) {
    mt_destroy_walk(enode, mt, false);
    call_rcu(&node->rcu, mt_free_walk);   /* 遅延解放 */
} else {
    mt_destroy_walk(enode, mt, true);     /* 即時解放 */
}
```

`struct maple_node`は、ノードがツリーから削除された後、その`slot[]`配列を`rcu_head`として再利用する（両者はunionに格納されている）。ノードがリンクから外される前に`slots[]`からポインタを読み込んだリーダーは、それを安全に参照し続けられる。そのメモリはRCUグレースピリオドが経過するまでアロケータに返されない。

### mmにおけるRCUモードのライフサイクル {#rcu-mode-lifecycle-in-the-mm}

`mm->mm_mt`はほとんどの時間、RCUを有効にして動作する。プロセスの破棄時と`dup_mmap()`の経路では、ノードが大量に破棄される際の遅延解放のオーバーヘッドを避けるため、カーネルは一時的にRCUモードを無効にする。

```c
/* mm/mmap.c — exit_mmap() */
mt_clear_in_rcu(&mm->mm_mt);   /* 破棄のため即時解放に切り替える */
...
__mt_destroy(&mm->mm_mt);      /* 残りのノードをすべて解放する */

/* mm/mmap.c — dup_mmap() — 一括複製の完了後 */
mt_set_in_rcu(vmi.mas.tree);   /* 新しいmmでRCUを再び有効にする */
```

`mt_set_in_rcu()`と`mt_clear_in_rcu()`は、ツリーのロックの下で`MT_FLAGS_USE_RCU`を切り替える。

### ロックなしのVMA検索（per-VMAロックの高速パス） {#lockless-vma-lookup-per-vma-lock-fast-path}

`rcu_read_lock()`だけを保持し、`mmap_lock`を保持していないリーダーでも、`mm->mm_mt`を安全にたどってVMAを見つけられる。これがper-VMAロックの高速パスの基盤である（[per-vma-locks.md](per-vma-locks.md)を参照）。

```c
/* mm/mmap_lock.c — lock_vma_under_rcu() */
MA_STATE(mas, &mm->mm_mt, address, address);

rcu_read_lock();
vma = mas_walk(&mas);          /* RCU下でのロックなしmaple treeウォーク */
if (!vma)
    goto inval;
vma = vma_start_read(mm, vma); /* per-VMA読み出しロックを試みる */
...
rcu_read_unlock();
```

`mas_walk()`は、`__rcu`アノテーション付きのポインタとピボットを読むだけで、ルートからリーフまでたどる。削除されたノードは少なくとも1回のRCUグレースピリオドの間は有効なままなので、ロックは不要である。

---

## MA_STATEとmaple stateカーソル {#ma_state-and-the-maple-state-cursor}

`struct ma_state`は、すべての高度なmaple tree APIのための操作カーソルである。ツリー内の現在位置を追跡するため、順次的な操作（例えば全VMAの走査）で呼び出しのたびにルートからやり直す必要がない。

```c
/* include/linux/maple_tree.h */
struct ma_state {
    struct maple_tree      *tree;         /* 操作対象のツリー */
    unsigned long           index;        /* 現在の操作の範囲の開始 */
    unsigned long           last;         /* 現在の操作の範囲の終了 */
    struct maple_enode     *node;         /* 現在のノード */
    unsigned long           min;          /* 現在のノードの暗黙の最小値 */
    unsigned long           max;          /* 現在のノードの暗黙の最大値 */
    struct slab_sheaf      *sheaf;        /* 事前割り当てされたノードプール */
    struct maple_node      *alloc;        /* 事前割り当てされた単一ノード（高速パス） */
    unsigned long           node_request; /* 事前割り当てするノード数 */
    enum maple_status       status;       /* ma_start / ma_active / ma_none / ma_error / ... */
    unsigned char           depth;        /* 書き込み時の降下中の深さ */
    unsigned char           offset;       /* 現在のノード内のスロット/ピボットのインデックス */
    unsigned char           mas_flags;
    unsigned char           end;          /* 現在のノードで最後に使われているスロット */
    enum store_type         store_type;   /* 必要な書き込みの種類 */
};
```

`status`フィールドは、次の操作がカーソルをどう扱うかを決める。

| ステータス | 意味 |
|--------|---------|
| `ma_start` | まだ位置が決まっていない。次の操作はルートからたどる |
| `ma_active` | カーソルは`node`/`offset`で有効。その場からの継続が安全 |
| `ma_none` | 検索完了。`index`にエントリは見つからなかった |
| `ma_root` | エントリはルートポインタ内で見つかった（ツリーのエントリが1つだけ） |
| `ma_pause` | ノードのデータが古い可能性がある。再開が必要（例: ロックを手放した） |
| `ma_overflow` | 検索の上限に達した |
| `ma_underflow` | 検索の下限に達した |
| `ma_error` | エラーが発生した。`node`に負のerrnoがエンコードされている |

`MA_STATE`マクロはスタック上にカーソルを初期化する。

```c
#define MA_STATE(name, mt, first, end)      \
    struct ma_state name = {                \
        .tree   = mt,                       \
        .index  = first,                    \
        .last   = end,                      \
        .node   = NULL,                     \
        .status = ma_start,                 \
        .min    = 0,                        \
        .max    = ULONG_MAX,                \
        ...                                 \
    }
```

操作が完了すると、`mas->index`と`mas->last`は、見つかったか格納されたエントリの範囲全体を保持する。

`mm/mmap.c`と`mm/vma.c`の全体で使われる`struct vma_iterator`は、薄いラッパーである。

```c
/* include/linux/mm_types.h */
struct vma_iterator {
    struct ma_state mas;
};

#define VMA_ITERATOR(name, __mm, __addr)    \
    struct vma_iterator name = {            \
        .mas = {                            \
            .tree   = &(__mm)->mm_mt,       \
            .index  = __addr,               \
            .status = ma_start,             \
        },                                  \
    }
```

---

## 主要な操作 {#key-operations}

### ツリーのライフサイクル: `mt_init()`と`mtree_destroy()` {#tree-lifecycle-mtinit-and-mtreedestroy}

```c
/* デフォルトのフラグ（外部ロックなし、RCUなし）で空のツリーを初期化する */
static inline void mt_init(struct maple_tree *mt)
{
    mt_init_flags(mt, 0);
}

/* 特定のフラグで初期化する（mm->mm_mtで使用） */
static inline void mt_init_flags(struct maple_tree *mt, unsigned int flags);

/* ツリーを破棄し、すべてのノードを解放する（内部でma_lockを取得） */
void mtree_destroy(struct maple_tree *mt);

/* ロックなしで破棄する — 呼び出し側が外部ロックを保持していること */
void __mt_destroy(struct maple_tree *mt);
```

`mm->mm_mt`については、`mm_init()`が`MM_MT_FLAGS`を指定して`mt_init_flags()`を使う。`exit_mmap()`と`dup_mmap()`のエラー経路での破棄では、呼び出し側がすでに`mmap_lock`を書き込み用に保持した後で`__mt_destroy(&mm->mm_mt)`を使う。

---

### `mas_walk()` — カーソルをindexに位置付ける {#mas_walk-position-cursor-at-index}

```c
void *mas_walk(struct ma_state *mas);
```

ツリー内で`mas->index`を検索する。戻ったとき、`mas->index`と`mas->last`は見つかったエントリの範囲に設定される（何も見つからなければ`0`と`ULONG_MAX`）。カーソルは進めない。同じindexに対して`mas_walk()`を2回呼ぶと、同じエントリが返る。

`rcu_read_lock()`だけで安全に呼べる（書き込みロックは不要）。per-VMAロックの高速パスはこれを直接使う。

```c
MA_STATE(mas, &mm->mm_mt, address, address);
rcu_read_lock();
vma = mas_walk(&mas);
```

---

### `mas_find()` — indexからの前方走査 {#mas_find-forward-iteration-from-index}

```c
void *mas_find(struct ma_state *mas, unsigned long max);
```

範囲が`[mas->index, max]`と重なる次のエントリを返す。正常に戻った後、`mas->index`と`mas->last`は返されたエントリの範囲を表し、カーソルはそこに位置付けられる。次の呼び出しは`mas->last + 1`から前方に走査を始める。

残りの範囲にエントリが存在しない場合は`NULL`を返す（ステータスを`ma_overflow`に設定する）。

`rcu_read_lock()`または書き込みロックを保持した状態で呼ばなければならない。

`mas_for_each()`マクロはこれを使っている。

```c
#define mas_for_each(__mas, __entry, __max) \
    while (((__entry) = mas_find((__mas), (__max))) != NULL)
```

---

### `mas_store()` / `mas_store_gfp()` — 範囲マッピングの挿入または更新 {#masstore-masstore_gfp-insert-or-update-a-range-mapping}

```c
/* [mas->index, mas->last]にentryを格納する。事前割り当てされたノードが必要。 */
void *mas_store(struct ma_state *mas, void *entry);

/* オンデマンドの割り当てを伴って格納する（失敗時は-ENOMEMを返す）。 */
int mas_store_gfp(struct ma_state *mas, void *entry, gfp_t gfp);
```

どちらの関数も、範囲`[mas->index, mas->last]`に対して`entry`をツリーに書き込む。この範囲は呼び出し前に設定しておかなければならない。既存のエントリが範囲と重なる場合、それは押しのけられる（必要に応じて分割、縮小、または置換される）。`mas_store()`は押しのけられた最初のエントリ（あれば）を返し、`mas_store_gfp()`は0または`-ENOMEM`を返す。

書き込みロックが必要である。

VMAのコードでは、`vma_iter_store_gfp()`が`mas_store_gfp()`をラップしている。

```c
/* mm/vma.h */
static inline int vma_iter_store_gfp(struct vma_iterator *vmi,
                                     struct vm_area_struct *vma, gfp_t gfp)
{
    __mas_set_range(&vmi->mas, vma->vm_start, vma->vm_end - 1);
    mas_store_gfp(&vmi->mas, vma, gfp);
    if (unlikely(mas_is_err(&vmi->mas)))
        return -ENOMEM;
    vma_mark_attached(vma);
    return 0;
}
```

---

### `mas_erase()` — 現在のindexのエントリを削除する {#mas_erase-remove-the-entry-at-the-current-index}

```c
void *mas_erase(struct ma_state *mas);
```

`mas->index`を含む範囲を見つけ、その範囲全体を消去し（NULLを格納し）、以前のエントリを返す。戻ったとき、`mas->index`と`mas->last`は消去された範囲に設定される。

書き込みロックが必要である。

VMAのコードでは、削除経路は`vma_iter_clear()`を使う。これはNULLを指定して`mas_store_prealloc()`を呼ぶ。

```c
/* mm/vma.h */
static inline void vma_iter_clear(struct vma_iterator *vmi)
{
    mas_store_prealloc(&vmi->mas, NULL);
}
```

---

### `mt_for_each()` — 単純なツリー全体の走査 {#mtforeach-simple-full-tree-iteration}

```c
/* include/linux/maple_tree.h */
#define mt_for_each(__tree, __entry, __index, __max)        \
    for (__entry = mt_find(__tree, &(__index), __max);      \
         __entry;                                            \
         __entry = mt_find_after(__tree, &(__index), __max))
```

`mt_for_each`は、明示的な`ma_state`ではなく、より単純な`mt_find()` / `mt_find_after()`インターフェース（呼び出しのたびに内部で`rcu_read_lock()`を取得・解放する）を使う。反復の間でカーソルの状態を保持する必要がない、読み出し専用の走査に便利である。

---

## コード例 {#code-examples}

### アドレスによるVMAの検索 {#vma-lookup-by-address}

```c
/* find_vma() — mm/mmap.c */
struct vm_area_struct *find_vma(struct mm_struct *mm, unsigned long addr)
{
    unsigned long index = addr;

    mmap_assert_locked(mm);
    return mt_find(&mm->mm_mt, &index, ULONG_MAX);
}
```

`mt_find()`は`index`から開始し、範囲が`addr`をカバーするか、`addr`の後に続く最初のエントリを返す。アドレス検索においては、これは「どのVMAがaddrをカバーしているか、なければ次のVMAはどれか」を問うことと等価である。

### mmap時のVMAの挿入 {#vma-insertion-during-mmap}

```c
/* mm/vma.c — 匿名マッピングの作成（簡略化） */
vma = vm_area_alloc(mm);
vma_set_range(vma, addr, addr + len, pgoff);
vm_flags_init(vma, vm_flags);
vma_start_write(vma);                    /* VMAを書き込みロック状態にする */

if (vma_iter_store_gfp(vmi, vma, GFP_KERNEL))
    goto mas_store_fail;

mm->map_count++;
```

### 全VMAの走査（例: /proc/PID/mapsのため） {#iterating-all-vmas-eg-for-procpidmaps}

```c
VMA_ITERATOR(vmi, mm, 0);
struct vm_area_struct *vma;

for_each_vma(vmi, vma) {
    /* vma->vm_start、vma->vm_end、vma->vm_flags ... */
}
```

`for_each_vma`は`vma_next()`の上に構築されている。これはルートからやり直すことなく、`vma_iterator`をmaple tree内で前方に進める。

### 空きアドレス範囲の探索（mmapのヒント探索） {#finding-a-free-address-range-mmap-hint-search}

```c
/* mas_empty_area() — [min, max]内で少なくとも`size`バイトのgapを見つける */
MA_STATE(mas, &mm->mm_mt, 0, 0);
mas_empty_area(&mas, mmap_min_addr, mmap_end - len, len);
addr = mas.index;
```

これは`maple_arange_64`ノードの`gap[]`配列を活用し、十分な大きさのギャップを含みえないサブツリーを飛ばす。これにより空き範囲の探索はO(n)ではなくO(log n)になる。

---

## rbtreeとの性能比較 {#performance-compared-to-rbtree}

| 性質 | rbtree＋連結リスト | maple tree |
|----------|---------------------|------------|
| 点検索 | O(log n) | O(log n) |
| 順次走査 | O(n)のポインタ追跡 | O(n)の順次的なノード読み出し |
| 空きギャップの探索 | O(n)の全走査 | ギャップのメタデータによるO(log n) |
| ノードあたりのキャッシュライン | 1（ノードあたりVMAポインタ1つ） | 256バイトのノードあたり最大16エントリ |
| VMAあたりのメモリ（インデックスのオーバーヘッド） | 約40バイト（rb_node＋リストポインタ） | 償却で約16バイト（共有ノード） |
| 読み出し時のロック | mmap_lockが必要 | RCU読み出しロックで十分 |
| 更新の複雑さ | 2つの構造の更新 | 1つのツリーの更新 |

順次走査（例えば`/proc/PID/maps`の読み出しや、大きな範囲にわたる`mprotect()`）における主な利点は、`maple_range_64`ノードが1つの256バイトの割り当てに16個のVMAポインタを保持することである。ノード全体の走査は、ノードをロードするための1回のキャッシュミスで最大16個の有用なエントリを得られる。連結リストをたどる場合は *エントリごとに* 1回のキャッシュミスが生じる。

数千のマッピングを持つアドレス空間（JVM、大規模アプリケーション、コンテナランタイム）では、このキャッシュ圧迫の減少は測定可能な差になる。

---

## デバッグ {#debugging}

### `/proc/PID/maps`と`/proc/PID/smaps` {#procpidmaps-and-procpidsmaps}

これらのファイルは`mm->mm_mt`を直接走査して生成される。読むために必要な変更はない。maple treeが正式な情報源である。

### `CONFIG_DEBUG_MAPLE_TREE` {#configdebugmaple_tree}

`lib/Kconfig.debug`で有効にする。

```
CONFIG_DEBUG_MAPLE_TREE=y   # requires CONFIG_DEBUG_KERNEL=y
```

このオプションを有効にすると次のようになる。

- `MT_BUG_ON()`、`MAS_BUG_ON()`、`MT_WARN_ON()`が各ツリー操作で追加の整合性チェックを行い、失敗時にツリー全体をダンプする。
- 各書き込みの前に、`mas_store()`内で追加の範囲検証が行われる。
- `mt_validate()`を明示的に呼び出して、任意のツリーの構造的な整合性を確認できる。

```c
void mt_validate(struct maple_tree *mt);
```

`mt_validate()`は、すべてのピボット配列がソートされていること、範囲が重なっていないこと、ギャップのメタデータが一貫していること、`ma_flags`に格納された高さが実際のツリーの深さと一致することを確認する。

### `mt_dump()` {#mt_dump}

```c
void mt_dump(const struct maple_tree *mt, enum mt_dump_format format);
```

ツリー全体を、10進数（`mt_dump_dec`）または16進数（`mt_dump_hex`）の形式でカーネルログに出力する。`CONFIG_DEBUG_MAPLE_TREE=y`のときにのみ利用できる。失敗時には`MT_BUG_ON()`と`MAS_BUG_ON()`から自動的に呼ばれる。

### `mas_dump()` {#mas_dump}

```c
void mas_dump(const struct ma_state *mas);
```

`ma_state`カーソルの現在の状態（index、last、ノードポインタ、status、min/max、depth、offset）を出力する。カーソルの状態がおかしいときのデバッグに有用である。

!!! tip "maple tree操作のトレーシング"
    maple treeには、`lib/maple_tree.c`の随所に組み込まれたtracepoint（`trace_ma_read`、`trace_ma_write`、`trace_ma_op`）がある。対応する`maple_tree`イベントグループを有効にすると`ftrace`で確認でき、`CONFIG_DEBUG_MAPLE_TREE`を必要とせずに、操作ごとにツリーの様子を可視化できる。

!!! note "RCUモードとデバッグ"
    `CONFIG_DEBUG_MAPLE_TREE`を`CONFIG_MAPLE_RCU_DISABLED`（`lib/maple_tree.c`の先頭にあるコンパイル時オプション）と組み合わせると、RCUによる解放が完全に抑止され、ノードは即座に解放される。これはツリーの利用者に潜むuse-after-freeのバグを表面化させるのに役立つが、本番環境では安全ではない。

---

## ほかのサブシステムとの関係 {#relationship-to-other-subsystems}

- **per-VMAロック**（`mm/mmap_lock.c`）は、`rcu_read_lock()`の下でロックなしのVMA検索を行うために`MT_FLAGS_USE_RCU`に依存している。[per-vma-locks.md](per-vma-locks.md)を参照。
- **`/proc/PID/maps`** は、`mmap_lock`を読み出し用に保持した状態で、`for_each_vma()`によって`mm->mm_mt`を走査する。
- **`dup_mmap()`** は`__mt_dup()`を使って親のmaple treeを子の`mm`へ一括コピーし、その後`VM_DONTCOPY`のエントリを`vma_iter_clear_gfp()`で置き換える。
- **`execve()`** は`exit_mmap()`を呼んでアドレス空間を破棄する。`exit_mmap()`は、すべてのページテーブルエントリをクリアした後で`__mt_destroy(&mm->mm_mt)`を呼ぶ。
- **vmalloc** は、カーネルの仮想アドレス空間の追跡に、maple treeではなく赤黒木（`struct rb_root`である`free_vmap_area_root`）を使う。

## 参考文献 {#further-reading}

- [LWN: The maple tree, a modern data structure for a complex problem](https://lwn.net/Articles/845507/) — Liam HowlettとMatthew Wilcoxによる2022年の記事。設計根拠、ノードのレイアウト、rbtree＋連結リストの組み合わせとのベンチマークを解説している
- [lib/maple_tree.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/maple_tree.c) — 完全な実装: `maple_tree_init()`、`mas_walk()`、`mas_find()`、`mas_store()`、`mas_erase()`、`mt_free_walk()`によるRCUでのノード解放、`CONFIG_MAPLE_RCU_DISABLED`デバッグモード
- [include/linux/maple_tree.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/maple_tree.h) — 公開API: `struct maple_tree`、`struct ma_state`、`MA_STATE()`、`mt_for_each()`、ノード種別の定数、すべてのフラグ定義
- コミット[54a611b60590](https://git.kernel.org/linus/54a611b60590) — `mm->mm_rb`とVMA連結リストを置き換えた、Linux 6.1でのmaple treeのマージ。マージメッセージには、パッチシリーズ全体とベンチマーク結果へのリンクがある
- [per-vma-locks.md](per-vma-locks.md) — `mmap_lock`を保持せずに`rcu_read_lock()`の下でロックなしのmaple treeウォークを行うために`MT_FLAGS_USE_RCU`に依存する、per-VMAロックの高速パス
- [LWN: Splitting VMAs](https://lwn.net/Articles/937943/) — Linux 6.7でのVMAツリーの再編作業を扱う。よりすっきりしたmaple tree APIによって可能になった、VMA操作ヘルパーの`mm/vma.c`への移動について
