# SLUBアロケータの内部 {#slub-allocator-internals}

> struct kmem_cache、freelistの仕組み、CPUごとのキャッシュ、slabのデバッグ

## なぜslabアロケータか {#why-a-slab-allocator}

`kmalloc` は割り当てのたびにページアロケータを呼び出すこともできるが、それは無駄が多い。

- ページアロケータの最小粒度: 4096バイト
- 典型的なカーネルの割り当て: 32〜512バイト
- 同じ型のオブジェクトが繰り返し作られ、解放される

slabアロケータはこれを次のように解決する。
1. あらかじめ分割されたpage（「slab」）からオブジェクトを割り当てる
2. 解放されたオブジェクトをキャッシュして即座に再利用する（ページアロケータとの往復なし）
3. コンストラクタのパターンを活用する: オブジェクトは同じ型であることが多い

Linuxは2.6.23以降 **SLUB**（「unqueued」slabアロケータ）を使っている。

## アーキテクチャの概要 {#architecture-overview}

```
kmalloc(128, GFP_KERNEL)
         │
         ▼
kmalloc-128 cache (struct kmem_cache)
         │
    ┌────┴─────────────────────────────────┐
    │ Per-CPU slab (struct kmem_cache_cpu) │  ← current CPU
    │   freelist: →obj1 →obj2 →obj3        │  ← hot, no lock
    └────┬─────────────────────────────────┘
         │ (per-CPU depleted)
    ┌────▼───────────────────────────────────────────┐
    │ Per-node partial list (struct kmem_cache_node) │
    │   partial slabs with some free objects         │
    └────┬───────────────────────────────────────────┘
         │ (no partial slabs)
         ▼
    Page allocator (get a new slab page)
```

## struct kmem_cache {#struct-kmem_cache}

```c
/* include/linux/slub_def.h */
struct kmem_cache {
    struct kmem_cache_cpu __percpu *cpu_slab; /* CPUごとの高速パス */

    /* 割り当てフラグと設定: */
    slab_flags_t    flags;
    unsigned long   min_partial;  /* ノード内の部分slabの最小数 */
    unsigned int    size;         /* オブジェクトサイズ（メタデータ込み） */
    unsigned int    object_size;  /* 実際のオブジェクトサイズ */
    struct reciprocal_value reciprocal_size;
    unsigned int    offset;       /* freelistポインタへのオフセット */
    struct kmem_cache_order_objects oo; /* オーダーとslabあたりのオブジェクト数 */
    struct kmem_cache_order_objects max;
    struct kmem_cache_order_objects min;
    gfp_t           allocflags;   /* slab割り当て用のgfpフラグ */
    int             refcount;
    void            (*ctor)(void *);  /* コンストラクタ */

    unsigned int    inuse;        /* メタデータへのオフセット */
    unsigned int    align;
    unsigned int    red_left_pad; /* レッドゾーンによるデバッグ用 */
    const char      *name;
    struct list_head list;        /* slab_cachesリスト内 */

    struct kmem_cache_node *node[MAX_NUMNODES];
};
```

## struct kmem_cache_cpu（CPUごとの高速パス） {#struct-kmem_cache_cpu-per-cpu-fast-path}

```c
struct kmem_cache_cpu {
    void        **freelist;     /* 最初の空きオブジェクトへのポインタ */
    unsigned long tid;          /* トランザクションID（ABA防止） */
    struct slab *slab;          /* 現在使用中のslab */
    /* ... */
};
```

freelistは **埋め込み連結リスト** である。各空きオブジェクトの先頭バイトには、次の空きオブジェクトへのポインタ（リストの末尾ではNULL）が入っている。

```
Slab page layout (objects = 128 bytes, SLUB):
┌────────┬────────┬────────┬────────┬──────────────────┐
│ obj[0] │ obj[1] │ obj[2] │ obj[3] │ ... until end    │
│ free   │ used   │ free   │ free   │                  │
│ →obj[2]│ [data] │ →obj[3]│ NULL   │                  │
└────────┴────────┴────────┴────────┴──────────────────┘
                     ↑
cpu_slab->freelist = &obj[0] → obj[2] → obj[3] → NULL
```

## kmallocの経路（alloc_cache_obj） {#kmalloc-path-alloc_cache_obj}

```c
/* mm/slub.c（簡略化） */
static __always_inline void *__kmalloc(size_t size, gfp_t flags)
{
    struct kmem_cache *s;

    /* 次のkmalloc-Nサイズに切り上げる: */
    s = kmalloc_caches[kmalloc_type(flags, _RET_IP_)][kmalloc_index(size)];

    return slab_alloc(s, size, flags, _RET_IP_);
}

static __always_inline void *slab_alloc(struct kmem_cache *s,
                                         size_t orig_size, gfp_t gfpflags,
                                         unsigned long addr)
{
    void *object;
    struct kmem_cache_cpu *c;
    unsigned long tid;

    /* 高速パス: CPUごとのfreelist */
    c = this_cpu_ptr(s->cpu_slab);
    tid = c->tid;
    barrier();

    object = c->freelist;
    if (unlikely(!object || !c->slab)) {
        /* CPUごとのslabを使い切った → 低速パス */
        object = __slab_alloc(s, gfpflags, NUMA_NO_NODE, addr, c, orig_size);
    } else {
        /* freelistポインタを進める（埋め込まれたnextポインタ）: */
        void *next = get_freepointer_safe(s, object);

        /* 同じfreelistからIRQが横取りする競合を防ぐためのCAS: */
        if (unlikely(!__update_cpu_freelist_fast(s, object, next, tid))) {
            object = __slab_alloc(s, gfpflags, NUMA_NO_NODE, addr, c, orig_size);
        }
    }
    return object;
}
```

## kfreeの経路 {#kfree-path}

```c
void kfree(const void *object)
{
    struct slab *slab;
    struct kmem_cache *s;

    /* pageからslab（ひいてはキャッシュ）を得る: */
    slab = virt_to_slab(object);
    if (unlikely(!slab)) {
        /* 大きな割り当て（複合ページ）かもしれない */
        free_large_kmalloc(page, object);
        return;
    }

    s = slab->slab_cache;
    slab_free(s, slab, object, NULL, &object, 1, _RET_IP_);
}

static void slab_free(struct kmem_cache *s, struct slab *slab,
                       void *head, void *tail, void **p, int cnt,
                       unsigned long addr)
{
    struct kmem_cache_cpu *c = this_cpu_ptr(s->cpu_slab);

    /* 現在のslabへの解放なら、freelistの先頭に追加するだけ: */
    if (slab == c->slab) {
        /* 高速パス: オブジェクトのnextポインタを現在のfreelistの先頭に設定する */
        set_freepointer(s, head, c->freelist);
        c->freelist = head;
        c->tid = next_tid(c->tid);
        return;
    }

    /* 低速パス: 別のslab → ロックして処理する */
    __slab_free(s, slab, head, tail, cnt, addr);
}
```

## kmem_cache_create（名前付きキャッシュ） {#kmem_cache_create-named-caches}

```c
/* 特定の型を持ち、頻繁に割り当てられるオブジェクト向け: */
struct kmem_cache *my_cache;

/* モジュール初期化時: */
my_cache = kmem_cache_create(
    "my_struct",               /* 名前（/proc/slabinfoに表示される） */
    sizeof(struct my_struct),  /* オブジェクトサイズ */
    __alignof__(struct my_struct), /* アラインメント */
    SLAB_HWCACHE_ALIGN |       /* キャッシュラインにアラインする */
    SLAB_PANIC,                /* 作成に失敗したらBUG() */
    my_struct_ctor             /* 任意のコンストラクタ */
);

/* 割り当てと解放: */
struct my_struct *obj = kmem_cache_alloc(my_cache, GFP_KERNEL);
kmem_cache_free(my_cache, obj);

/* モジュール終了時: */
kmem_cache_destroy(my_cache);
```

## SLUBのデバッグ {#slub-debugging}

```bash
# ブート時に SLUB デバッグを有効にする（オーバーヘッドが大きい！）:
# config で SLUB_DEBUG を有効にしたうえで:
# slub_debug=FZP (FreeZone, Poison, track Padding)
# またはキャッシュ単位で: slub_debug=FZP,my_struct

# /proc/slabinfo: キャッシュの統計
cat /proc/slabinfo
# name            <active_objs> <num_objs> <objsize> <objperslab> ...
# kmalloc-128        12453       13000       128         32        ...
# my_struct             123         256       256         16        ...

# より詳しく: slabtop（slab 版の top）
slabtop
# Active / Total Objects (% used)    : 1234567 / 1500000 (82.3%)
# Active / Total Slabs (% used)      : 12345 / 15000 (82.3%)
# ...

# あるポインタがどのキャッシュに属するかを調べる:
# （crash または vmlinux を読み込んだ gdb で）:
# crash> kmem -s <pointer>

# キャッシュの SLUB デバッグ情報:
ls /sys/kernel/slab/
cat /sys/kernel/slab/kmalloc-128/alloc_fastpath   # CPU ごとの割り当て回数
cat /sys/kernel/slab/kmalloc-128/free_fastpath
cat /sys/kernel/slab/kmalloc-128/slabs            # slab ページの総数
cat /sys/kernel/slab/kmalloc-128/objects          # オブジェクトの総数
cat /sys/kernel/slab/kmalloc-128/object_size      # 128

# KASAN: slab における use-after-free と範囲外アクセスを検出する:
# CONFIG_KASAN=y にすると SLUB はレッドゾーンを挿入し、メモリをポイズニングする
```

## メモリのポイズニング {#memory-poisoning}

`CONFIG_SLUB_DEBUG` が有効な場合、解放されたオブジェクトはポイズニングされる。

```
Poison values:
  0x6b ('k') — object content after free (POISON_FREE)
  0xbb        — red zone bytes while object is free (SLUB_RED_INACTIVE)
  0xcc        — red zone bytes while object is allocated (SLUB_RED_ACTIVE)

On the next allocation, SLUB checks that POISON_FREE pattern is intact.
If not: use-after-free detected, kernel dumps the offending allocation.
```

## 参考文献 {#further-reading}

- [kmallocとslab](slab.md) — 利用者向けのAPI
- [ページアロケータ](page-allocator.md) — SLUBはここからpageを得る
- [KASAN](kasan.md) — slabメモリのエラー検出
- [KFENCE](kfence.md) — サンプリングに基づく軽量なslabのエラー検出
- `mm/slub.c` — SLUBの実装（約10,000行）
- `Documentation/mm/slub.rst` — SLUBのドキュメント
