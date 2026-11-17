# Shrinker API {#shrinker-api}

> サブシステムがメモリ回収に参加するためのコールバック機構 — カーネルがメモリを必要とするとき、登録されたshrinkerにキャッシュの解放を求める。

## shrinkerとは何か {#what-are-shrinkers}

カーネルは多くのインメモリキャッシュを保持している。ディレクトリエントリ（dentry）キャッシュ、inodeキャッシュ、DMA-bufヒープ、ドライバ固有のオブジェクトプール、zswapプールなどである。これらのキャッシュは価値がある — 高コストな再計算やI/Oを回避してくれる — が、システムが逼迫しているときにはメモリを明け渡さなければならない。

shrinker APIは、サブシステムがメモリ管理のコアに登録して「解放を頼んでもらえるオブジェクトがある」と伝えるための手段である。回収が走ると、1回のスイープにつき登録済みのすべてのshrinkerを2回呼ぶ。まずどれだけ解放できるかを尋ね（カウントパス）、次に実際にその一部を解放させる（スキャンパス）。

ツリー内のshrinkerの例は次のとおり。

- `fs/super.c` — dentry LRUとinode LRUをカバーするsuperblockごとのshrinker（`super_cache_scan` / `super_cache_count`）
- `mm/huge_memory.c` — 遅延分割THPのshrinker
- `mm/zswap.c` — zswapプールのshrinker
- `mm/workingset.c` — ワーキングセットのシャドウエントリのshrinker
- `fs/xfs/xfs_icache.c` — XFSのinode GCのshrinker
- `mm/vmalloc.c` — vmapノードのshrinker

## `struct shrinker` {#struct-shrinker}

`include/linux/shrinker.h`より:

```c
struct shrinker {
    unsigned long (*count_objects)(struct shrinker *,
                                   struct shrink_control *sc);
    unsigned long (*scan_objects)(struct shrinker *,
                                  struct shrink_control *sc);

    long batch;     /* 回収のバッチサイズ、0 = デフォルト */
    int seeks;      /* オブジェクトの再作成に要するシーク数 */
    unsigned flags;

    /*
     * この shrinker の参照カウント。登録済みの shrinker は初期参照カウント 1
     * を持ち、その後ルックアップ操作は shrinker_try_get() を通じてそれを使う
     * ことができる。登録解除時に初期参照カウントが破棄され、参照カウントが 0
     * に達した後、shrinker は RCU を介して非同期に解放される。
     */
    refcount_t refcount;
    struct completion done;  /* 参照カウントが 0 になるのを待つために使う */
    struct rcu_head rcu;

    void *private_data;

    /* 内部用 */
    struct list_head list;
#ifdef CONFIG_MEMCG
    int id;          /* shrinker_idr 内の ID */
#endif
#ifdef CONFIG_SHRINKER_DEBUG
    int debugfs_id;
    const char *name;
    struct dentry *debugfs_entry;
#endif
    /* 削除保留中のオブジェクト、ノードごと */
    atomic_long_t *nr_deferred;
};

#define DEFAULT_SEEKS 2
```

### フィールドガイド {#field-guide}

| フィールド | 説明 |
|-------|-------------|
| `count_objects` | 解放*できる*オブジェクトの数を返す。実際には何も解放せずに呼ばれる。 |
| `scan_objects` | 最大`sc->nr_to_scan`個のオブジェクトを解放する。解放した数、`SHRINK_STOP`、または`SHRINK_EMPTY`を返す。 |
| `batch` | `scan_objects`の1回の呼び出しでスキャンするオブジェクト数。`0`なら内部のデフォルト値128を使う。大きなバッチは呼び出しのオーバーヘッドを減らし、小さなバッチはレイテンシを改善する。 |
| `seeks` | オブジェクト1つを再作成する相対コスト（ディスクシークを単位とする）。優先度スケーリングの式で使われる。迷ったら`DEFAULT_SEEKS`（2）を使う。再作成にI/Oを必要としないオブジェクトには0を設定する — これにより、より積極的に削減されるようになる。 |
| `flags` | `SHRINKER_NUMA_AWARE`、`SHRINKER_MEMCG_AWARE`、`SHRINKER_NONSLAB` — 後述する。 |
| `private_data` | shrinkerの実装のための不透明なポインタ。例えば`fs/super.c`では`super_block *`。 |
| `nr_deferred` | 計算されたがまだ完了していないスキャン作業（`batch`の丸めや競合による）のノードごとの数。次回の呼び出しに持ち越される。コアによって管理される。 |

## `struct shrink_control` {#struct-shrink_control}

カーネルは呼び出しのたびに、両方のコールバックに`struct shrink_control`を渡す。

```c
struct shrink_control {
    gfp_t gfp_mask;

    /* 現在縮小中のノード（NUMA 対応 shrinker 用） */
    int nid;

    /*
     * scan_objects がスキャンして回収を試みるべきオブジェクト数。
     * 毎回の呼び出しの前にリセットされる。
     */
    unsigned long nr_to_scan;

    /*
     * scan_objects はいくつのオブジェクトを処理したか?
     * デフォルトは nr_to_scan。呼び出された側が実際の進捗を反映するよう更新すべき。
     */
    unsigned long nr_scanned;

    /* 現在縮小中の memcg（memcg 対応 shrinker 用） */
    struct mem_cgroup *memcg;
};
```

### フィールドのセマンティクス {#field-semantics}

**`gfp_mask`** — 回収を引き起こした割り当てのコンテキスト。shrinkerはこれを尊重しなければならない。`__GFP_FS`がクリアされているなら、shrinkerはファイルシステムに入ってはならない（例えば、その場合`super_cache_scan`は即座に`SHRINK_STOP`を返す）。`__GFP_IO`がクリアされているなら、I/Oの開始を避ける。

**`nid`** — 縮小対象のNUMAノード。`SHRINKER_NUMA_AWARE`が設定されているときにのみ意味を持つ。それ以外のshrinkerは常にノード0を見る。

**`nr_to_scan`** — `scan_objects`を呼ぶ前にコアが設定する。shrinkerはこの数のオブジェクトの解放を試みるべきである。変更しても安全である（コアは毎回の呼び出しの前にリセットする）。

**`nr_scanned`** — コアによって`nr_to_scan`があらかじめ設定される。shrinkerは、（解放した数だけでなく）実際に調べたオブジェクト数を反映するようこれを更新すべきである。コアは`nr_deferred`を更新する際に、実施された作業の計上にこれを使う。

**`memcg`** — 対象となるメモリcgroup。memcgごとのshrinkerを走査する際にコアが設定する。グローバル（ルートmemcg）経路では`NULL`である。`SHRINKER_MEMCG_AWARE`が設定されているときにのみ意味を持つ。

## `count_objects`と`scan_objects` {#countobjects-vs-scanobjects}

この2つのコールバックは、2段階の契約を実装している。

### `count_objects` — 解放可能な量を見積もる {#count_objects-estimate-freeable-capacity}

```c
unsigned long (*count_objects)(struct shrinker *, struct shrink_control *sc);
```

キャッシュから解放*できる*オブジェクトの数を返す。戻り値は、そもそも`scan_objects`が呼ばれるかどうかを制御する。

| 戻り値 | 意味 |
|---|---|
| `SHRINK_EMPTY`（`~0UL - 1`） | キャッシュは完全に空である。コアはこのshrinkerのmemcgごとのshrinkerビットをクリアし、新しいオブジェクトが追加されるまで以降のパスでスキップする。 |
| `0` | 現時点では解放可能なものがない（例: すべてのオブジェクトがピン留めされている、あるいは数がしきい値を下回っている）。このスイープでは`scan_objects`は呼ばれない。 |
| `N > 0` | およそN個の解放可能なオブジェクトがある。コアはこれを使って、`nr_to_scan`で要求するオブジェクト数を計算する。 |

**ロックの要件はない。** `count_objects`は、回収とデッドロックしうるロックを取ってはならない。カウントは古くてもよい — コアは、`count_objects`と`scan_objects`が実際に解放する数の食い違いを許容する。これは設計上のものであり、カウントは比例スケーリングに使われるのであって、ハードな制限としてではない。

### `scan_objects` — 実際にオブジェクトを解放する {#scan_objects-actually-free-objects}

```c
unsigned long (*scan_objects)(struct shrinker *, struct shrink_control *sc);
```

キャッシュを走査し、最大`sc->nr_to_scan`個のオブジェクトを解放し、`sc->nr_scanned`を更新して、解放したオブジェクト数を返す。

| 戻り値 | 意味 |
|---|---|
| `N >= 0` | この呼び出しの間に解放されたオブジェクト数。 |
| `SHRINK_STOP`（`~0UL`） | 現時点では進捗できない — 必要となるロックが、現在の回収コンテキストでデッドロックを引き起こしうる。コアは、現在の回収の呼び出しの間、このshrinkerの`scan_objects`を再び呼ぶことはない。 |
| `SHRINK_EMPTY`（`~0UL - 1`） | キャッシュにオブジェクトが残っていない。 |

```c
/* include/linux/shrinker.h で定義されている */
#define SHRINK_STOP  (~0UL)
#define SHRINK_EMPTY (~0UL - 1)
```

## shrinkerのフラグ {#shrinker-flags}

```c
#define SHRINKER_NUMA_AWARE   BIT(2)
#define SHRINKER_MEMCG_AWARE  BIT(3)
#define SHRINKER_NONSLAB      BIT(4)
```

**`SHRINKER_NUMA_AWARE`** — shrinkerはNUMAノードごとに1回呼ばれる。`sc->nid`には回収対象のノードが設定される。キャッシュがノードごとに分割されている場合（例: ノードごとのLRU）に使う。このフラグがなければ、どのノードが逼迫しているかにかかわらず、shrinkerは`nid = 0`で1回だけ呼ばれる。

**`SHRINKER_MEMCG_AWARE`** — shrinkerはmemcgごとの回収に参加する。memcgが制限を超えると、コアは`MEMCG_AWARE`なshrinkerを、`sc->memcg`に対象cgroupを設定して呼ぶ。shrinkerは、そのmemcgにチャージされたオブジェクトだけを解放しなければならない。これには、LRUが`list_lru_init_memcg()`で初期化されている必要がある。このフラグがなければ、shrinkerはグローバル（ルートcgroup）回収の間にのみ呼ばれる。

**`SHRINKER_NONSLAB`** — `SHRINKER_MEMCG_AWARE`と組み合わせたときにのみ意味を持つ。shrinkerがslab以外のメモリを管理していることを示す。memcgのkmem計上が無効な場合、`NONSLAB`でないshrinkerはmemcgごとの回収の間スキップされるが、`NONSLAB`のshrinkerは常に呼ばれる。slab割り当てではないがmemcgごとに回収されるべきDMA-bufのアタッチメントのようなオブジェクトに使われる。

## カーネルはshrinkerをどのように呼び出すか {#how-the-kernel-invokes-shrinkers}

### 呼び出し経路 {#call-path}

```
Memory pressure (allocation fails or watermark crossed)
    │
    ├── kswapd (background) ──────────────────────────────┐
    │                                                      │
    └── direct reclaim (allocating process blocks) ────────┤
                                                           │
                                                           v
                                              shrink_node() in mm/vmscan.c
                                                           │
                                                           v
                                              shrink_slab(gfp_mask, nid,
                                                          memcg, priority)
                                              in mm/shrinker.c
                                                           │
                                     ┌─────────────────────┴──────────────────────┐
                                     │ (non-root memcg)                           │ (root)
                                     v                                            v
                          shrink_slab_memcg()                    list_for_each_entry_rcu
                          iterates per-memcg bitmap               over shrinker_list
                                     │                                            │
                                     └──────────────┬─────────────────────────────┘
                                                    v
                                          do_shrink_slab(sc, shrinker, priority)
                                                    │
                                          1. call count_objects()
                                          2. compute nr_to_scan via priority
                                          3. loop: call scan_objects() in
                                             batches until total_scan exhausted
                                             or SHRINK_STOP returned
```

### `do_shrink_slab`における優先度スケーリング {#priority-scaling-in-doshrinkslab}

`priority`パラメータ（`mm/vmscan.c`の`sc->priority`由来）は、shrinkerをどれだけ積極的に呼び出すかを制御する。優先度が高いほど、メモリ逼迫が大きいことを意味する。

```c
/* mm/shrinker.c の do_shrink_slab() より */
if (shrinker->seeks) {
    delta = freeable >> priority;  /* priority 12 では delta は非常に小さい */
    delta *= 4;
    do_div(delta, shrinker->seeks);
} else {
    delta = freeable / 2;          /* seeks=0: 無条件に半分を削減する */
}

total_scan = nr >> priority;   /* 遅延された作業。低優先度では縮小される */
total_scan += delta;
total_scan = min(total_scan, (2 * freeable));
```

`priority = DEF_PRIORITY`（12、低い逼迫）では、`freeable >> 12`はごくわずかであり、shrinkerはほとんど触られない。priorityが0に近づくにつれ（極度の逼迫）、遅延カウント全体とより大きな`delta`が適用され、1回の呼び出しでより多くのオブジェクトがスキャンされる。`total_scan`は過剰なスキャンを防ぐため`2 * freeable`で上限が設けられる。

`do_shrink_slab`はその後、`total_scan`を使い切るか`SHRINK_STOP`が返されるまで、`batch_size`個ずつループで`scan_objects`を呼ぶ。計算されたが（バッチの丸めのために）行われなかった作業は`nr_deferred`に保存され、次の呼び出しに加算される。

### 並行性とロック {#concurrency-and-locking}

shrinkerのリストは、変更（登録／登録解除）については`shrinker_mutex`で、走査についてはRCUで保護される。個々のshrinkerのルックアップは`shrinker_try_get()`（`refcount_inc_not_zero`）を使い、`do_shrink_slab`の実行中は参照を保持する。`shrinker_free()`は、メモリを解放する前に`wait_for_completion(&shrinker->done)`を呼んで、処理中のすべての参照を排出する。

これは、`scan_objects`が（異なるノードやmemcgに対して）異なるCPU上で並行に実行されうることを意味する。shrinkerの実装は並行呼び出しに対して安全でなければならない。

## 登録とライフサイクル {#registration-and-lifecycle}

### 3段階のライフサイクル {#the-three-step-lifecycle}

```c
/* 1. 割り当てて設定する */
struct shrinker *shrinker;

shrinker = shrinker_alloc(SHRINKER_NUMA_AWARE | SHRINKER_MEMCG_AWARE,
                           "mydriver-%s", device_name);
if (!shrinker)
    return -ENOMEM;

shrinker->count_objects = mydriver_count_objects;
shrinker->scan_objects  = mydriver_scan_objects;
shrinker->seeks         = DEFAULT_SEEKS;
shrinker->private_data  = my_private_ctx;

/* 2. 登録する（shrinker が回収から見えるようになる） */
shrinker_register(shrinker);

/* ... デバイス／モジュールの使用中 ... */

/* 3. 解放する（登録解除し、参照を排出してから、RCU を介して非同期に解放する） */
shrinker_free(shrinker);
```

### `shrinker_alloc(flags, fmt, ...)` {#shrinker_allocflags-fmt}

`struct shrinker`を割り当てて初期化する。`fmt`引数は`printf`形式の名前で、（`CONFIG_SHRINKER_DEBUG`が設定されている場合に）`/sys/kernel/debug/shrinker/`で使われる。`SHRINKER_MEMCG_AWARE`が要求されたが実行時にmemcgが無効な場合、このフラグは黙ってクリアされ、shrinkerは非memcgモードにフォールバックする。

割り当てに失敗すると`NULL`を返す。

### `shrinker_register(shrinker)` {#shrinker_registershrinker}

shrinkerを`shrinker_list`に追加し、`SHRINKER_REGISTERED`を設定し、`refcount`を1に初期化する（これにより回収から見えるようになる）。`shrinker_alloc()`が返したshrinkerに対してのみ呼ばなければならない。

### `shrinker_free(shrinker)` {#shrinker_freeshrinker}

登録解除と解放の両方を1回の呼び出しで処理する。

1. 登録済みであれば、初期参照カウントを落とし、処理中のすべての`do_shrink_slab`呼び出しが終わるのを待つ（`wait_for_completion`）。
2. `shrinker_mutex`の下で`shrinker_list`から削除する。
3. `SHRINKER_MEMCG_AWARE`であれば、memcgのIDRから削除する。
4. 構造体自体を`call_rcu`経由で`kfree`するようスケジュールする。

`shrinker_free()`が戻った後は、shrinkerを保持している構造体（例: `super_block`やドライバのデバイス構造体）を解放しても安全である。

### それぞれをいつ呼ぶか {#when-to-call-each}

| 状況 | 登録のタイミング | 解放のタイミング |
|-----------|-----------------|--------------|
| モジュール全体のキャッシュ | `module_init` / `subsys_initcall` | `module_exit` |
| デバイスごとのキャッシュ | キャッシュの初期化後の`probe()` | `remove()` / `probe()`のエラー経路 |
| superblockごとのキャッシュ | マウント完了前の`sget()` / `alloc_super()` | `kill_sb()` / `deactivate_super()` |

## 実例: superblockのshrinker {#real-example-the-superblock-shrinker}

マウントされたすべてのファイルシステムは、dentry LRUとinode LRUの両方をカバーするsuperblockごとのshrinkerを持つ。これは`alloc_super()`（`fs/super.c`）で設定される。

```c
/* fs/super.c: alloc_super() */
s->s_shrink = shrinker_alloc(SHRINKER_NUMA_AWARE | SHRINKER_MEMCG_AWARE,
                              "sb-%s", type->name);

s->s_shrink->scan_objects  = super_cache_scan;
s->s_shrink->count_objects = super_cache_count;
s->s_shrink->batch         = 1024;
s->s_shrink->private_data  = s;
```

shrinkerは後で、superblockが完全に設定されて`SB_BORN`とマークされた後に登録される。

```c
/* fs/super.c: vfs_get_tree() / mount_fs() の経路 */
shrinker_register(s->s_shrink);
```

**`super_cache_count`**は、dentryとinodeのLRUに対して`list_lru_shrink_count()`を問い合わせ、`s_op->nr_cached_objects()`によるファイルシステム固有のオブジェクト数を加え、合計がゼロなら`SHRINK_EMPTY`を返す。

**`super_cache_scan`**は、`sc->nr_to_scan`をdentry、inode、FS固有のオブジェクトに比例配分する。まず`prune_dcache_sb()`を呼び（dentryはinodeをピン留めしているため）、次に`prune_icache_sb()`、次に`s_op->free_cached_objects()`を呼ぶ。`sc->gfp_mask`に`__GFP_FS`が設定されていなければ、ロックを保持している可能性のあるファイルシステムへの再入を避けるため、即座に`SHRINK_STOP`を返す。

## よくある間違い {#common-mistakes}

!!! warning "`gfp_mask`を確認せずに`scan_objects`内でブロックやスリープをする"
    `scan_objects`は、任意のコンテキストで直接回収から呼ばれうる。呼び出し元がshrinkerも必要とするロックを保持している場合や、回収がファイルシステムのロックの下で行われている場合、そのロックを再び取るとデッドロックする。ファイルシステムに入ったりI/Oを開始したりする前に、必ず`sc->gfp_mask`の`__GFP_FS`と`__GFP_IO`を確認すること。安全に進められない場合は`SHRINK_STOP`を返す。

!!! warning "キャッシュが実際には空なのに`count_objects`から0を返す"
    `0`は「現時点では解放可能なものがない」（例: オブジェクトがピン留めされている）を意味する。`SHRINK_EMPTY`は「キャッシュが完全に空である」を意味する。この2つは置き換え可能ではない。キャッシュが空のときに`0`を返しても、グローバル回収の経路ではコストはかからないが、memcgごとの経路では、カーネルがmemcgビットマップのshrinkerのビットをクリアできなくなり、将来の不要な起動を引き起こす。

!!! warning "`nr_scanned`の誤った扱い"
    `sc->nr_scanned`は`sc->nr_to_scan`にあらかじめ設定されている。`scan_objects`が要求より少ないオブジェクトしか調べなかった場合（例: trylockに失敗して一部をスキップした）、実際に調べた数を反映するよう`sc->nr_scanned`を更新すること。コアはこれを使って`next_deferred`を計算する。実際のスキャン数が少ないのに`nr_to_scan`のままにしておくと、その作業が完了したと誤ってマークされ、遅延作業が過少に報告される。

!!! warning "エラー経路で`shrinker_free`の呼び出しを忘れる"
    `shrinker_register`を呼んだ後にデバイスやモジュールの初期化が失敗した場合、エラーを返す前に`shrinker_free`を呼ばなければならない。解放済みまたは初期化途中のデータを指す登録済みshrinkerは、次の回収サイクルで呼ばれ、use-after-freeやNULLデリファレンスを引き起こす。`shrinker_free`は未登録の場合（まだ登録されていなければ何もしない）と登録済みの場合の両方を処理するので、`shrinker_alloc`が成功した後のどの後始末経路でも無条件に呼んで安全である。

!!! warning "`scan_objects`の並行呼び出し"
    カーネルは、（`SHRINKER_NUMA_AWARE`なら）異なるノード、または（`SHRINKER_MEMCG_AWARE`なら）異なるmemcgに対して、`scan_objects`を並行に呼ぶことがある。LRUの走査やカウンタは並行アクセスに対して安全でなければならない。独自のリストを作るのではなく、`list_lru`（ノードごと・memcgごとのロックが組み込まれている）を使うこと。

!!! warning "意図せず`seeks = 0`を使う"
    `seeks`を0に設定すると、`do_shrink_slab`の積極的な分岐`delta = freeable / 2`が発動する。これは、どの優先度でも、shrinkerが1回のパスで解放可能なオブジェクトの半分のスキャンを要求することを意味する。これは再投入が安価な（I/Oを伴わない）キャッシュには意図されたものだが、高コストなキャッシュに誤って使うと、逼迫時に過剰な追い出しを引き起こす。

## 主要なソースファイル {#key-source-files}

| ファイル | 説明 |
|------|-------------|
| [`include/linux/shrinker.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/shrinker.h) | `struct shrinker`、`struct shrink_control`、`SHRINK_STOP`、`SHRINK_EMPTY`、フラグ、`shrinker_alloc`/`shrinker_register`/`shrinker_free`の宣言 |
| [`mm/shrinker.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/shrinker.c) | `shrinker_alloc`、`shrinker_register`、`shrinker_free`、`do_shrink_slab`、`shrink_slab`、`shrink_slab_memcg`、memcgビットマップの管理 |
| [`mm/vmscan.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) | `shrink_node`、`shrink_lruvec` — 回収中に`shrink_slab()`が呼ばれる場所 |
| [`fs/super.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/super.c) | `super_cache_scan`、`super_cache_count`、`alloc_super`でのsuperblockごとのshrinkerの設定 |
| [`fs/dcache.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/dcache.c) | `prune_dcache_sb` — dentry LRUを縮小するために`super_cache_scan`から呼ばれる |
| [`fs/inode.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/inode.c) | `prune_icache_sb` — inode LRUを縮小するために`super_cache_scan`から呼ばれる |
| [`include/linux/list_lru.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/list_lru.h) | `list_lru_init_memcg`、`list_lru_shrink_count`、`list_lru_shrink_walk` — shrinkerとともに使われるNUMA/memcg対応のLRU |

## 参考文献 {#further-reading}

- [mm/shrinker.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/shrinker.c) — `shrinker_alloc()`、`shrinker_register()`、`shrinker_free()`、`do_shrink_slab()`（優先度スケーリングとバッチループ）、`shrink_slab()`、およびmemcgごとのビットマップを使う`shrink_slab_memcg()`
- [mm/vmscan.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) — LRUリストのスキャン後に`shrink_slab()`を呼ぶ回収のエントリポイント、`shrink_node()`と`shrink_lruvec()`
- [LWN: Smarter shrinkers](https://lwn.net/Articles/550463/) — 古い単一コールバックのモデルを置き換えた、`count_objects`/`scan_objects`の2段階分割の動機と設計
- [LWN: Shrinker reorganization](https://lwn.net/Articles/966451/) — shrinkerのコードを`mm/vmscan.c`から専用の`mm/shrinker.c`ファイルへ切り出したことと、それに伴って導入されたRCUベースの参照カウントのライフサイクル
- [reclaim](reclaim.md) — `shrink_slab()`がLRUスキャン、kswapd、直接回収とともにどこに位置づけられるかを示す、回収の全体像
- [memcg](memcg.md) — cgroupごとの回収コンテキスト: `sc->memcg`がいつ設定されるか、`SHRINKER_MEMCG_AWARE`なshrinkerがcgroupのメモリ制限とどう相互作用するか
