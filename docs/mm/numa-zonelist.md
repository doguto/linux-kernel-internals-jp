# NUMAのzonelistの構築とフォールバック順序 {#numa-zonelist-construction-and-fallback-ordering}

> NUMAノードのメモリが尽きたとき、次の割り当てがどこに行くかはカーネルが事前に構築したフォールバックリストによって決まる。そしてメモリポリシーはそのリストを完全に上書きできる。

## zonelistとは何か {#what-is-a-zonelist}

すべてのNUMAノード（`pg_data_t`、`pgdat`とも呼ばれる）は、zonelistの配列を持っている。zonelistは`zoneref`エントリの順序付きの列であり（(ゾーン, ノード)の組ごとに1エントリ）、アロケータは要求を満たす際にこれを優先順位の順にたどる。

```c
/* include/linux/mmzone.h */

/* zonelist上のゾーンの最大数 */
#define MAX_ZONES_PER_ZONELIST (MAX_NUMNODES * MAX_NR_ZONES)

enum {
    ZONELIST_FALLBACK,      /* フォールバックありのzonelist */
#ifdef CONFIG_NUMA
    ZONELIST_NOFALLBACK,    /* フォールバックなしのzonelist（__GFP_THISNODE） */
#endif
    MAX_ZONELISTS
};

/*
 * この構造体はzonelist内のゾーンに関する情報を保持する。大きな構造体への
 * 参照外しやテーブルの検索を避けるためにここに格納される
 */
struct zoneref {
    struct zone *zone;   /* 実際のゾーンへのポインタ */
    int zone_idx;        /* zone_idx(zoneref->zone) */
};

/*
 * 1つの割り当て要求は1つのzonelistに対して動作する。zonelistは
 * ゾーンのリストであり、先頭が割り当ての「目標」で、それ以外の
 * ゾーンは優先度の降順に並んだフォールバックゾーンである。
 */
struct zonelist {
    struct zoneref _zonerefs[MAX_ZONES_PER_ZONELIST + 1];
};
```

各`pgdat`は2つのzonelist（`node_zonelists[MAX_ZONELISTS]`）を持つ。

| インデックス | 名前 | 目的 |
|---|---|---|
| `ZONELIST_FALLBACK` | フォールバックリスト | 通常の割り当て。優先ノードが先頭で、その後NUMA距離の順に並ぶ |
| `ZONELIST_NOFALLBACK` | フォールバックなしリスト | `__GFP_THISNODE`が設定されている場合に使われる。ローカルノードのみで、他のノードには決して越境しない |

ヘルパー`node_zonelist(nid, gfp_flags)`は、GFPフラグに`__GFP_THISNODE`が含まれているかどうかに基づいて適切なリストを選択する。

```c
/* include/linux/gfp.h */
static inline struct zonelist *node_zonelist(int nid, gfp_t flags)
{
    return NODE_DATA(nid)->node_zonelists + gfp_zonelist(flags);
}
```

ここで`gfp_zonelist()`は、`__GFP_THISNODE`が設定されていれば`ZONELIST_NOFALLBACK`を、そうでなければ`ZONELIST_FALLBACK`を返す。

---

## zonelistの構築方法 {#how-zonelists-are-built}

### エントリポイント: `build_all_zonelists()` {#entry-point-buildallzonelists}

zonelistはカーネルの生存期間中の2つの時点で構築される。

1. **ブート時** — メモリの初期化中に`build_all_zonelists()`が呼ばれる。システムはまだ`SYSTEM_BOOTING`の状態にあるので、`build_all_zonelists_init()`に処理を委ね、それが`__build_all_zonelists(NULL)`を呼んで全ノードを再構築する。
2. **メモリホットプラグ時** — ノードがオンラインになると、新しいノードの`pgdat`を引数にして`build_all_zonelists(pgdat)`が再び呼ばれる。そのノードがまだオンラインでなければ、そのノード自身のzonelistだけが再構築される。そうでなければ、フォールバックリストが新しいトポロジを反映するよう全ノードが再構築される。

```c
/* mm/page_alloc.c */
void __ref build_all_zonelists(pg_data_t *pgdat)
{
    if (system_state == SYSTEM_BOOTING) {
        build_all_zonelists_init();
    } else {
        __build_all_zonelists(pgdat);
    }
    ...
}
```

再構築中は書き込み側のseqlock（`zonelist_update_seq`）が保持される。これにより、並行して動作するアロケータ（`GFP_ATOMIC`でIRQレベルで動くこともある）は、再構築が進行中であることを検出してzonelistを読み直せる。

### ノードの順序付け: `build_zonelists()` {#node-ordering-build_zonelists}

各`pgdat`について、`build_zonelists()`は、メモリを持つすべてのノードを訪れるまで`find_next_best_node()`を繰り返し呼び出すことで`ZONELIST_FALLBACK`リストを構築する。

```c
/* mm/page_alloc.c */
static void build_zonelists(pg_data_t *pgdat)
{
    static int node_order[MAX_NUMNODES];
    int node, nr_nodes = 0;
    nodemask_t used_mask = NODE_MASK_NONE;
    int local_node, prev_node;

    local_node = pgdat->node_id;
    prev_node = local_node;

    while ((node = find_next_best_node(local_node, &used_mask)) >= 0) {
        if (node_distance(local_node, node) !=
            node_distance(local_node, prev_node))
            node_load[node] += 1;

        node_order[nr_nodes++] = node;
        prev_node = node;
    }

    build_zonelists_in_node_order(pgdat, node_order, nr_nodes);
    build_thisnode_zonelists(pgdat);
    ...
}
```

カーネルはブート時に、結果の順序を`Fallback order for Node N: 0 1 2 ...`のように出力する。

### 次に最適なノードの選択: `find_next_best_node()` {#choosing-the-next-best-node-findnextbest_node}

`find_next_best_node()`は、未訪問のすべてのノードを複合的な指標でスコア付けする。

1. **NUMA距離** — 主要な指標で、`node_distance()`を通じてACPIのSLITテーブルから読み取られる。近いノードほどスコアが低い。
2. **CPUの存在によるペナルティ** — CPUを持つノード（`PENALTY_FOR_NODE_WITH_CPUS`）は、CPUを持たないメモリのみのノードよりもわずかにスコアが高くなる。メモリのみのノードはカーネル自身からの割り当て圧力が少ないので、溢れた分の受け皿として適している。
3. **負荷分散** — 新しい距離グループの最初のノードに小さな`node_load`の加算が適用され、等距離のノード間で割り当てがラウンドロビンで分散される。

優先（ローカル）ノードは常に先頭に置かれる。

### `numa_zonelist_order` sysctl {#the-numazonelistorder-sysctl}

カーネルは`/proc/sys/vm/numa_zonelist_order`を公開している。現在のカーネルでは`"Node"`順のみが実装されている。以前のゾーンベースの順序付けモードは、実際には役に立たなかったため削除された。このsysctlは互換性のために残されているが、`'d'`、`'D'`、`'n'`、`'N'`のいずれかで始まらない値（`"Default"`と`"Node"`に一致するもの以外）は無視し、その場合は警告を出す。

```bash
$ cat /proc/sys/vm/numa_zonelist_order
Node
```

---

## ノード内のゾーンの順序 {#zone-ordering-within-a-node}

単一のノード内では、`build_zonerefs_node()`が最も高いゾーンインデックスから0までゾーンを走査し、実体のある各ゾーンをリストに追加する。

```c
/* mm/page_alloc.c */
static int build_zonerefs_node(pg_data_t *pgdat, struct zoneref *zonerefs)
{
    struct zone *zone;
    enum zone_type zone_type = MAX_NR_ZONES;
    int nr_zones = 0;

    do {
        zone_type--;
        zone = pgdat->node_zones + zone_type;
        if (populated_zone(zone)) {
            zoneref_set_zone(zone, &zonerefs[nr_zones++]);
            check_highest_zone(zone_type);
        }
    } while (zone_type);

    return nr_zones;
}
```

ゾーン種別のenumを昇順に並べると次のとおりである。

```c
/* include/linux/mmzone.h */
enum zone_type {
    ZONE_DMA,       /* CONFIG_ZONE_DMA    — DMA可能な小さな範囲 */
    ZONE_DMA32,     /* CONFIG_ZONE_DMA32  — 32ビットDMAの範囲 */
    ZONE_NORMAL,    /* 主要なアドレス指定可能メモリ */
    ZONE_MOVABLE,   /* マイグレーションまたはオフライン化できるページ */
    __MAX_NR_ZONES
};
```

ループは`MAX_NR_ZONES - 1`から*下に向かって*数えるので、典型的なx86-64のノードのゾーンはzonelistに次の順で現れる。

```
ZONE_MOVABLE → ZONE_NORMAL → ZONE_DMA32 → ZONE_DMA
```

**なぜこの順序なのか？** アロケータは容量の最も大きいゾーンを先に試す（`ZONE_NORMAL`と`ZONE_MOVABLE`を合わせるとRAMの大部分を占める）。`ZONE_DMA32`と`ZONE_DMA`は、アドレス指定に厳しい制約を持つハードウェアデバイスのために温存される。通常の割り当てに通常のメモリを使うことで、こうした限られたDMA可能な領域が守られる。

---

## フォールバックのアルゴリズム: zonelistの走査 {#fallback-algorithm-iterating-the-zonelist}

### `for_each_zone_zonelist`ファミリ {#the-foreachzone_zonelist-family}

アロケータの高速パス（`get_page_from_freelist()`）は、次のものを使ってzonelistを走査する。

```c
/* include/linux/mmzone.h */

/* highest_zoneidx以下のすべてのゾーンを走査する */
#define for_each_zone_zonelist(zone, z, zlist, highidx) \
    for_each_zone_zonelist_nodemask(zone, z, zlist, highidx, NULL)

/* 同じだが、ノードがnodemaskに含まれるゾーンに絞り込む */
#define for_each_zone_zonelist_nodemask(zone, z, zlist, highidx, nodemask) \
    for (z = first_zones_zonelist(zlist, highidx, nodemask), zone = zonelist_zone(z); \
         zone; \
         z = next_zones_zonelist(++z, highidx, nodemask), \
             zone = zonelist_zone(z))
```

`z`はzonelistの`_zonerefs`配列を指すカーソル（`struct zoneref *`）である。`++z`で進めると、先頭から再スキャンすることなく次のエントリに移る。これは1ステップあたりO(1)である。

### ウォーターマークによる判定 {#watermark-gating}

各ゾーンで、アロケータは割り当てを試みる前にウォーターマークをチェックする。

```c
/* mm/page_alloc.c — get_page_from_freelist()の内部 */
mark = wmark_pages(zone, alloc_flags & ALLOC_WMARK_MASK);
if (!zone_watermark_fast(zone, order, mark,
                         ac->highest_zoneidx, alloc_flags,
                         gfp_mask)) {
    /* ウォーターマークの判定に失敗 — このゾーンを飛ばしてループを続ける */
    ...
    continue;
}
/* ウォーターマークの判定に合格 — このゾーンから割り当てる */
```

`zone_watermark_ok()`は、`free_pages - requested > watermark + lowmem_reserve[highest_zoneidx]`であることをチェックする。`lowmem_reserve`配列はゾーンごとのバッファを提供し、上位ゾーン向けの割り当てがDMA可能なゾーンを完全に使い果たすのを防ぐ。

!!! note "アロケータはリスト全体を再試行しない"
    あるゾーンでウォーターマークのチェックに失敗すると、そのゾーンは飛ばされ、ループは単に次のエントリに進む。先頭からの再スキャンはない。フォールバックリスト内のすべてのゾーンがウォーターマークのチェックに失敗すると、割り当ては`__alloc_pages_slowpath()`に落ち、回収、コンパクションを引き起こし、メモリを解放できなければ最終的にOOMになる。

---

## メモリポリシーがzonelistの走査をどう変えるか {#how-memory-policies-modify-zonelist-traversal}

メモリポリシー（`struct mempolicy`）は`mempolicy.c`で割り当てパスに介入し、優先ノードか、zonelistのイテレータを絞り込むnodemaskのいずれかを変更できる。

```c
/* include/linux/mempolicy.h */
struct mempolicy {
    atomic_t refcnt;
    unsigned short mode;   /* MPOL_DEFAULT, MPOL_BIND, MPOL_PREFERRED, ... */
    unsigned short flags;  /* MPOL_F_STATIC_NODES, MPOL_F_RELATIVE_NODES, ... */
    nodemask_t nodes;      /* interleave/bind/preferredなど */
    int home_node;         /* MPOL_BINDとMPOL_PREFERRED_MANYのホームノード */
    union {
        nodemask_t cpuset_mems_allowed;
        nodemask_t user_nodemask;
    } w;
    struct rcu_head rcu;
};
```

`mempolicy.c`の`policy_nodemask()`関数は、ポリシーのモードを、コアのアロケータが使う`(preferred_nid, nodemask)`の組に変換する。

```c
/* mm/mempolicy.c */
static nodemask_t *policy_nodemask(gfp_t gfp, struct mempolicy *pol,
                                   pgoff_t ilx, int *nid)
{
    nodemask_t *nodemask = NULL;

    switch (pol->mode) {
    case MPOL_PREFERRED:
        *nid = first_node(pol->nodes);
        break;
    case MPOL_BIND:
        if (apply_policy_zone(pol, gfp_zone(gfp)) &&
            cpuset_nodemask_valid_mems_allowed(&pol->nodes))
            nodemask = &pol->nodes;
        ...
        break;
    case MPOL_INTERLEAVE:
        *nid = (ilx == NO_INTERLEAVE_INDEX) ?
            interleave_nodes(pol) : interleave_nid(pol, ilx);
        break;
    ...
    }

    return nodemask;
}
```

### `MPOL_DEFAULT` {#mpol_default}

ポリシーオブジェクトは設定されていない（`current->mempolicy == NULL`）。アロケータは現在のCPUのノードを優先ノードとして使い、nodemaskによる絞り込みは行わない。フォールバックリスト全体が利用可能である。

### `MPOL_PREFERRED` {#mpol_preferred}

`nodes`フィールドはちょうど1つのノードを保持する。`policy_nodemask()`は`*nid`をそのノードに設定し、そこを優先的な開始点にする。nodemaskによる絞り込みは`NULL`のままなので、優先ノードが使い果たされると、アロケータは`ZONELIST_FALLBACK`リスト全体を通じてフォールバックする。

### `MPOL_BIND` {#mpol_bind}

`policy_nodemask()`はnodemaskの絞り込みとして`&pol->nodes`を返す。`for_each_zone_zonelist_nodemask`は、ノードがそのマスクに含まれないゾーンをすべて飛ばす。したがってアロケータはバインドされたノードだけを考慮する。**バインドされたノードがすべて使い果たされても、他のノードへのフォールバックは起こらない**。割り当ては失敗し、バインドされていないノードに溢れ出す代わりにOOMを引き起こす。

!!! warning "MPOL_BINDとOOM"
    `MPOL_BIND`で満杯のノードにバインドされたプロセスは、リモートメモリにフォールバックする代わりにOOM killerによってkillされる。これは意図的なものである。アプリケーションは、ローカルメモリを必要とすることを明示的に宣言しているからである。

### `MPOL_INTERLEAVE` {#mpol_interleave}

`interleave_nodes()`は`current->il_prev`を`pol->nodes`内の次のノードに進め、それをこの割り当ての優先ノードとして返す。zonelistにnodemaskの絞り込みは適用されないので、インターリーブ先のノードが一時的に使い果たされていても通常のフォールバックは利用可能である。多数の割り当てを通じて、ページは許可されたノード間にラウンドロビンで分散される。

### `MPOL_LOCAL` {#mpol_local}

現在のCPUのNUMAノードから割り当てる。`policy_nodemask()`は`MPOL_LOCAL`の場合に`*nid`を変更しない（defaultケースに落ちる）ので、`numa_node_id()`が使われる。通常のフォールバックが適用される。

### まとめの表 {#summary-table}

| ポリシー | 優先ノード | nodemaskによる絞り込み | 使い果たしたとき |
|---|---|---|---|
| `MPOL_DEFAULT` | 現在のCPUのノード | なし | 通常どおりフォールバック |
| `MPOL_PREFERRED` | 指定したノード | なし | 通常どおりフォールバック |
| `MPOL_BIND` | マスク内の最初のノード（または`home_node`） | `pol->nodes` | OOM — フォールバックなし |
| `MPOL_INTERLEAVE` | `pol->nodes`間でラウンドロビン | なし | 通常どおりフォールバック |
| `MPOL_LOCAL` | 現在のCPUのノード | なし | 通常どおりフォールバック |

---

## 実際の効果: 2ノードの例 {#practical-effect-two-node-example}

ノード0（32GB）とノード1（32GB）を持つシステムを考える。NUMA距離はローカル = 10、リモート = 20である。ノード0の`ZONELIST_FALLBACK`リストは次のとおりである。

```
[ZONE_NORMAL/node0] → [ZONE_DMA32/node0] → [ZONE_NORMAL/node1] → [ZONE_DMA32/node1]
```

**シナリオ1 — デフォルトポリシー、ノード0が満杯:**
明示的なポリシーなしでノード0上で動くタスクが、ノード0の`ZONE_NORMAL`を使い切る。`get_page_from_freelist()`は`ZONE_NORMAL/node0`でウォーターマークのチェックに失敗し、`ZONE_DMA32/node0`を飛ばし（ウォーターマークのチェックにやはり失敗するか、`lowmem_reserve`が拒否する）、`ZONE_NORMAL/node1`で成功する。割り当てはリモートのレイテンシでノード1から提供される。`numastat`はノード0に`numa_miss`を、ノード1に`numa_foreign`を記録する。

**シナリオ2 — ノード0への`MPOL_BIND`、ノード0が満杯:**
nodemaskの絞り込みにより、イテレータはノード0のエントリのみに制限される。`ZONE_NORMAL/node0`と`ZONE_DMA32/node0`はどちらもウォーターマークのチェックに失敗する。他のゾーンはnodemaskの絞り込みを通過しない。低速パスは十分なメモリを回収できず、OOM killerが呼び出される。タスクはノード1にフォールバック**しない**。

**シナリオ3 — ノード0への`MPOL_PREFERRED`、ノード0が満杯:**
優先nidはノード0に上書きされるが、nodemaskの絞り込みは適用されない。ノード0のゾーンがウォーターマークのチェックに失敗すると、イテレータは`ZONE_NORMAL/node1`へと進み、割り当ては成功する。タスクがたまたまノード1にスケジュールされていてもノード0が最初に試される点を除けば、挙動はデフォルトポリシーと同じである。

---

## 実際にフォールバックを観察する {#observing-fallback-in-practice}

### `numastat` {#numastat}

```bash
numastat
```

| カウンタ | 意味 |
|---|---|
| sysfsでの名前 | `/proc/vmstat`での名前 | 意味 |
|---|---|---|
| `numa_hit` | `numa_hit` | 意図した（優先）ノードで割り当てられた |
| `numa_miss` | `numa_miss` | 優先ノードが満杯だったため別のノードで割り当てられた |
| `numa_foreign` | `numa_foreign` | リモートのタスクがこのノードから割り当てた |
| `interleave_hit` | `numa_interleave` | インターリーブポリシーが意図したノードに当たった |
| `local_node` | `numa_local` | ローカルノードで割り当てられた（CPUとメモリが同じ場所にある） |
| `other_node` | `numa_other` | ローカルでないノードで割り当てられた |

`numa_miss`のカウントが高い場合、優先ノードでメモリが圧迫されており、リモートメモリへのフォールバックが頻繁に起きていることを示す。

### `/proc/zoneinfo` {#proczoneinfo}

```bash
grep -A 20 "Node 0, zone" /proc/zoneinfo
```

ゾーンごとの空きページ数と3つのウォーターマーク（`min`、`low`、`high`）をここで確認できる。`free`が`min`に近づくと、そのゾーンはほとんどの割り当てで`zone_watermark_ok()`に失敗し、アロケータはzonelistの次のエントリへ飛ぶ。

### カーネルのブートログ {#kernel-boot-log}

ブート時に、カーネルは各ノードのフォールバック順序を出力する。

```
Fallback order for Node 0: 0 1
Fallback order for Node 1: 1 0
```

これは`build_zonelists()`から直接出力されている。

```c
pr_info("Fallback order for Node %d: ", local_node);
for (node = 0; node < nr_nodes; node++)
    pr_cont("%d ", node_order[node]);
```

---

## 主要なソースファイル {#key-source-files}

| ファイル | 注目すべき箇所 |
|---|---|
| `include/linux/mmzone.h` | `struct zoneref`、`struct zonelist`、`enum zone_type`、`ZONELIST_FALLBACK`、`ZONELIST_NOFALLBACK`、`MAX_ZONES_PER_ZONELIST`、`for_each_zone_zonelist`、`for_each_zone_zonelist_nodemask`、`enum numa_stat_item` |
| `include/linux/mempolicy.h` | `struct mempolicy` — `mode`、`flags`、`nodes`、`home_node` |
| `include/uapi/linux/mempolicy.h` | `MPOL_DEFAULT`、`MPOL_PREFERRED`、`MPOL_BIND`、`MPOL_INTERLEAVE`、`MPOL_LOCAL`、`MPOL_PREFERRED_MANY` |
| `include/linux/gfp.h` | `node_zonelist()`、`gfp_zonelist()` |
| `mm/page_alloc.c` | `build_all_zonelists()`、`__build_all_zonelists()`、`build_zonelists()`、`build_zonelists_in_node_order()`、`build_thisnode_zonelists()`、`build_zonerefs_node()`、`find_next_best_node()`、`get_page_from_freelist()`、`zone_watermark_ok()`、`__zone_watermark_ok()` |
| `mm/mempolicy.c` | `policy_nodemask()`、`alloc_pages_mpol()`、`interleave_nodes()`、`mempolicy_slab_node()` |

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [mm/page_alloc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) — `build_all_zonelists()`、`build_zonelists()`、`find_next_best_node()`、`get_page_from_freelist()`、`zone_watermark_ok()`
- [mm/mempolicy.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mempolicy.c) — `policy_nodemask()`、`alloc_pages_mpol()`、およびすべてのメモリポリシーモードの実装
- [include/linux/mmzone.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mmzone.h) — `struct zonelist`、`struct zoneref`、`for_each_zone_zonelist_nodemask`、`ZONELIST_FALLBACK`、`ZONELIST_NOFALLBACK`

### カーネルドキュメント {#kernel-documentation}

- `Documentation/admin-guide/mm/numa_memory_policy.rst` — すべての`MPOL_*`ポリシーモードと、それらがzonelistとどう相互作用するかのリファレンス（[HTML版](https://docs.kernel.org/admin-guide/mm/numa_memory_policy.html)）

### LWNの記事 {#lwn-articles}

- [GFP flags and memory zones](https://lwn.net/Articles/629925/) — GFPフラグがどのようにゾーンを選択し、フォールバックリストと相互作用するか

### 関連ドキュメント {#related-docs}

- [NUMAのメモリ管理](numa.md) — メモリポリシーの概要: `set_mempolicy()`、`mbind()`、`numactl`、自動NUMAバランシング
- [NUMA距離とソケット間レイテンシ](numa-distance.md) — `node_distance()`の値が`find_next_best_node()`のスコア付けをどう左右するか
- [NUMAがメモリ回収に及ぼす影響](numa-reclaim.md) — ノードごとのkswapd、ウォーターマーク、そして`MPOL_BIND`がOOMとどう相互作用するか
- [ページアロケータ](page-allocator.md) — `__alloc_pages()`からzonelistを経てバディアロケータに至る割り当てパス全体
