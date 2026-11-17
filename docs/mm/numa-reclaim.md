# メモリ回収に対するNUMAの影響 {#numa-effects-on-memory-reclaim}

> ノードごとのウォーターマーク、ノードごとのkswapdスレッド、NUMAバランシングがどのように相互作用してマルチソケットシステムのメモリ回収を形作るのか、そしてリモートノードに圧力がかかったときに何がうまくいかなくなり得るのか。

## 背景: NUMAのメモリはノードローカルである {#background-numa-memory-is-node-local}

NUMAシステムでは、物理CPUソケット（**ノード**）がそれぞれメモリバンクを所有する。カーネルは各ノードを`pg_data_t`（`pgdat`とも呼ばれる）として表現する。各`pgdat`は1つ以上のzone（`struct zone`）を含み、各zoneは固有のウォーターマーク閾値を管理する。

```
System with 2 NUMA nodes
────────────────────────────────────────────────────────────
  Node 0 (pgdat 0)                Node 1 (pgdat 1)
  ┌────────────────────┐          ┌────────────────────┐
  │ zone: ZONE_DMA     │          │ zone: ZONE_DMA     │
  │ zone: ZONE_DMA32   │          │ zone: ZONE_DMA32   │
  │ zone: ZONE_NORMAL  │          │ zone: ZONE_NORMAL  │
  │                    │          │                    │
  │  kswapd0 thread    │          │  kswapd1 thread    │
  └────────────────────┘          └────────────────────┘
```

kswapdカーネルスレッドは、システム全体で1つではなく**ノードごとに1つ**存在する。回収のアカウンティング、ウォーターマーク、LRUリストはすべてノード単位である。これが、本ドキュメントの他のすべての内容を左右する根本的な事実である。

---

## ノードごとのウォーターマーク {#per-node-watermarks}

すべての`struct zone`は固有のウォーターマーク配列を格納している。

```c
/* include/linux/mmzone.h */
enum zone_watermarks {
    WMARK_MIN,
    WMARK_LOW,
    WMARK_HIGH,
    WMARK_PROMO,        /* メモリ階層化の NUMA バランシングが有効なときに使われる */
    NR_WMARK
};

struct zone {
    /* ... */
    unsigned long _watermark[NR_WMARK];
    unsigned long watermark_boost;   /* 断片化イベント後の一時的なブースト */
    /* ... */
};
```

アクセサマクロは、格納された値に`watermark_boost`を加算する。

```c
static inline unsigned long wmark_pages(const struct zone *z,
                                         enum zone_watermarks w)
{
    return z->_watermark[w] + z->watermark_boost;
}

static inline unsigned long min_wmark_pages(const struct zone *z)  { ... }
static inline unsigned long low_wmark_pages(const struct zone *z)  { ... }
static inline unsigned long high_wmark_pages(const struct zone *z) { ... }
static inline unsigned long promo_wmark_pages(const struct zone *z){ ... }
```

ウォーターマークは2つの異なる回収パスを制御する。

| 条件 | 動作 |
|---|---|
| 空きpage < `low_wmark_pages(zone)` | `wakeup_kswapd()` — そのノードのkswapdを起床させる |
| 空きpage < `min_wmark_pages(zone)` | 直接回収 — 割り当て中のプロセスがブロックする |

各ノードの各zoneが固有のウォーターマークを持つため、**Node 0が`WMARK_HIGH`を余裕で上回っている一方で、Node 1が`WMARK_MIN`に達していることがあり得る**。カーネルはノード間でウォーターマークを平均化しない。

---

## ノードごとのkswapd {#per-node-kswapd}

### pgdatごとに1スレッド {#one-thread-per-pgdat}

`kswapd_run()`はブート時にノードごとに1つのカーネルスレッドを作成する。

```c
/* mm/vmscan.c */
void __meminit kswapd_run(int nid)
{
    pg_data_t *pgdat = NODE_DATA(nid);
    /* ... */
    pgdat->kswapd = kthread_create_on_node(kswapd, pgdat, nid,
                                            "kswapd%d", nid);
    /* ... */
    wake_up_process(pgdat->kswapd);
}
```

各スレッドは自分のノードに固定される（`kthread_create_on_node`）ので、kswapd自身が行う割り当ては正しいローカルプールから得られる。`pg_data_t::kswapd`に格納されたポインタが、そのノードのスレッドのハンドルである。

### kswapdはどう起床するか {#how-kswapd-wakes-up}

`wakeup_kswapd()`は、空きpageが`WMARK_LOW`を下回っているzoneに対して割り当てが試みられるたびに、ページアロケータ（`mm/page_alloc.c`）から呼ばれる。

```c
/* mm/vmscan.c */
void wakeup_kswapd(struct zone *zone, gfp_t gfp_flags, int order,
                   enum zone_type highest_zoneidx)
{
    pg_data_t *pgdat = zone->zone_pgdat;   /* その zone を所有するノード */
    /* ... */
    if (READ_ONCE(pgdat->kswapd_order) < order)
        WRITE_ONCE(pgdat->kswapd_order, order);
    /* ... */
    wake_up_interruptible(&pgdat->kswapd_wait);
}
```

起床させられるのは**そのzoneを所有するノードのkswapd**だけである。Node 1での不足が、Node 0のkswapdを自動的に起床させることはない。

### balance_pgdatが行うこと {#what-balance_pgdat-does}

起床すると、kswapdスレッドは`balance_pgdat()`を呼ぶ。

```c
/* mm/vmscan.c */
static int balance_pgdat(pg_data_t *pgdat, int order, int highest_zoneidx)
```

`balance_pgdat()`はノードのzoneを優先順位の順にループし、管理対象のすべてのzoneがバランスするまで`shrink_node()`を呼ぶ。「バランスしている」かどうかは`pgdat_balanced()`で判定される。

```c
/* mm/vmscan.c */
static bool pgdat_balanced(pg_data_t *pgdat, int order, int highest_zoneidx)
{
    /* ... */
    for_each_managed_zone_pgdat(zone, pgdat, i, highest_zoneidx) {
        if (sysctl_numa_balancing_mode & NUMA_BALANCING_MEMORY_TIERING)
            mark = promo_wmark_pages(zone);
        else
            mark = high_wmark_pages(zone);
        /* 空き page >= mark かを確認する ... */
    }
}
```

kswapdは、**自分のノード**のすべてのzoneが`WMARK_HIGH`を上回るまで回収する。リモートノードから回収する義務はなく、リモートノードのウォーターマークも見ない。

### kswapdと直接回収 {#kswapd-vs-direct-reclaim}

| | kswapd（バックグラウンド） | 直接回収 |
|---|---|---|
| きっかけ | 空きpage < `WMARK_LOW` | 空きpage < `WMARK_MIN` |
| 誰が回収するか | 専用のカーネルスレッド | 割り当て中のプロセス自身 |
| プロセスのレイテンシ | なし（非同期） | プロセスがブロックする |
| エントリポイント | `balance_pgdat()` | `try_to_free_pages()` → `do_try_to_free_pages()` |
| ノードの範囲 | 自ノードのLRU | 割り当ての`zonelist`に従う |

!!!note
    `pg_data_t`の`kswapd_failures`は、回収したpageがゼロだった連続実行の回数を数える。`MAX_RECLAIM_RETRIES`回失敗すると、`kswapd_test_hopeless()`がtrueを返し、そのノードは「見込みなし（hopeless）」とみなされる。kswapdは手を引き、代わりに直接回収がそのノードを処理するに任せる。

---

## NUMAバランシングと回収の相互作用 {#numa-balancing-and-its-interaction-with-reclaim}

### NUMAヒントフォールトの仕組み {#how-numa-hint-faults-work}

`CONFIG_NUMA_BALANCING`が有効な場合、カーネルは`task_numa_work()`（`kernel/sched/fair.c`）でタスクのVMAを定期的に走査し、配置を評価したいpageにPROT_NONEのPTEを設定する。そのようなpageへの次のアクセスはページフォールトとしてトラップされ、`mm/memory.c`の`do_numa_page()`に到達する。

`do_numa_page()`は`numa_migrate_check()`を呼び、フォールトしたpageをローカルノードへ移動すべきかを判断する。マイグレーションが妥当であれば、`migrate_misplaced_folio_prepare()`と`migrate_misplaced_folio()`を経て処理を進める。

```c
/* mm/memory.c */
static vm_fault_t do_numa_page(struct vm_fault *vmf)
{
    /* ... */
    target_nid = numa_migrate_check(folio, vmf, vmf->address, &flags,
                                     writable, &last_cpupid);
    if (target_nid == NUMA_NO_NODE)
        goto out_map;               /* その場に留まる */

    if (migrate_misplaced_folio_prepare(folio, vma, target_nid)) {
        flags |= TNF_MIGRATE_FAIL;
        goto out_map;
    }
    if (!migrate_misplaced_folio(folio, target_nid)) {
        nid = target_nid;
        flags |= TNF_MIGRATED;      /* ローカルノードへ移動した */
        task_numa_fault(last_cpupid, nid, nr_pages, flags);
        return 0;
    }
    flags |= TNF_MIGRATE_FAIL;
    /* ... */
}
```

同じパターンが透過的ヒュージページにも適用され、`mm/huge_memory.c`で処理される。

### 昇格パスと回収パス {#the-promotion-path-vs-the-reclaim-path}

リモートのpageがアクセスされたとき:

```
Remote page accessed on Node 0 by CPU on Node 0
               │
               ▼
         NUMA hint fault fires
         (do_numa_page called)
               │
   ┌───────────┴───────────┐
   │                       │
   ▼                       ▼
Migration succeeds     Migration fails
(page moves to         (page stays on
 Node 0 — promotion)    Node 1 — remote)
               │                   │
               ▼                   ▼
     Page now on Node 0's    Page stays on Node 1's
     LRU — ages normally      LRU — aged and potentially
                               reclaimed by kswapd1
```

マイグレーションできないpage（たとえばNode 0自体のメモリが少ない場合や、pageが共有されていてマイグレーションが阻まれる場合）は、Node 1のLRUに留まる。Node 1のkswapdは、フォールトしたプロセスがNode 0でまだそれを活発に使っているかどうかにかかわらず、最終的にそれらをコールドなpageとして回収する。

これが核心的な緊張関係である。**回収パスと昇格パスは競合する**。kswapd1がNode 1の圧力を和らげるために回収するpageは、NUMAバランシングがより良い局所性のためにNode 0へ昇格させようとしていたpageかもしれない。

### WMARK_PROMOとメモリ階層化 {#wmark_promo-and-memory-tiering}

`sysctl_numa_balancing_mode`に`NUMA_BALANCING_MEMORY_TIERING`が設定されている場合（CXL／永続メモリの階層で使われる）、`pgdat_balanced()`は目標として`high_wmark_pages(zone)`の代わりに`promo_wmark_pages(zone)`を使う。これにより高速な階層により多くの余裕が確保され、低速なメモリからの昇格が、すぐに回収を引き起こすことなく進められる。

---

## zone_reclaim_mode {#zonereclaimmode}

### それは何だったか {#what-it-was}

`vm.zone_reclaim_mode`（カーネル変数: `node_reclaim_mode`）はsysctlであり、非ゼロのとき、ページアロケータはリモートノードのメモリにフォールバックする**前に**ローカルでの回収を試みる。これは、強制的な回収によってキャッシュがあふれるコストよりも、NUMAの局所性のほうが重要なワークロードを想定したものだった。

ビットは`include/uapi/linux/mempolicy.h`で定義されている。

```c
#define RECLAIM_ZONE    (1<<0)  /* zone/node 回収を有効にする */
#define RECLAIM_WRITE   (1<<1)  /* 回収中にダーティ page をライトバックする */
#define RECLAIM_UNMAP   (1<<2)  /* 回収中にマップされた page をアンマップする */
```

`node_reclaim_enabled()`は、これらのビットのいずれかが設定されているとtrueを返す。

```c
/* mm/internal.h */
static inline bool node_reclaim_enabled(void)
{
    return node_reclaim_mode & (RECLAIM_ZONE|RECLAIM_WRITE|RECLAIM_UNMAP);
}
```

有効な状態で割り当てが優先zoneのウォーターマークを満たせないと、`mm/page_alloc.c`の`get_page_from_freelist()`は、リモートのzoneに進む前にローカルノードに対して`node_reclaim()`を呼ぶ。

```c
/* mm/page_alloc.c */
if (!node_reclaim_enabled() ||
    !zone_allows_reclaim(zonelist_zone(ac->preferred_zoneref), zone))
    continue;

ret = node_reclaim(zone->zone_pgdat, gfp_mask, order);
```

`zone_allows_reclaim()`はNUMA距離によるゲートを課す。回収が試みられるのは、対象zoneのノードが優先zoneのノードから`node_reclaim_distance`以内にある場合だけである（デフォルト: `include/linux/topology.h`の`RECLAIM_DISTANCE = 30`）。これにより、実際には近くにあるリモートノードを避けるためにアロケータがコストの高いローカル回収を行うことを防いでいる。

### なぜレイテンシの問題を引き起こしたか {#why-it-caused-latency-problems}

リモートからの割り当ての前にローカルでの回収を強制すると、次のようになる。

- 空のウォーターマークバッファを保つために、有用なホットpageがローカルメモリから追い出される。
- ワーキングセットが複数ノードにまたがる、あるいはノード間でメモリを共有するワークロードでは、zone_reclaimがリモートノードの空き容量を一切見ないため、不釣り合いな追い出しが発生する。
- ライトバックのパス（`RECLAIM_WRITE`）が、本来ならリモートのDRAMを使って素早く完了するはずの割り当てパスにI/Oのレイテンシを加える。

汎用的なワークロード（ほとんどのデータベースやインメモリキャッシュを含む）では、温まったキャッシュpageを失う損失が、リモートアクセスのNUMAペナルティを避ける利益を上回る。

### 現状 {#current-state}

`zone_reclaim_mode`は**デフォルトで無効**（ゼロ）である。カーネルドキュメント（`Documentation/admin-guide/sysctl/vm.rst`）は明確にこう述べている。

> *zone_reclaim_mode is disabled by default. For file servers or workloads that benefit from having their data cached, zone_reclaim_mode should be left disabled as the caching effect is likely to be more important than data locality.*

（訳: zone_reclaim_modeはデフォルトで無効である。ファイルサーバーや、データがキャッシュされることで恩恵を受けるワークロードでは、キャッシュの効果のほうがデータの局所性よりも重要である可能性が高いため、zone_reclaim_modeは無効のままにしておくべきである。）

これが今でも有用なのは、各プロセスのワーキングセットが単一ノードに完全に収まり、リモートアクセスのレイテンシが性能上の支配的な懸念となる、厳密に分割されたHPCやNUMAを意識したリアルタイムのワークロードに限られる。

!!!warning
    共有メモリを使うワークロードや、ワーキングセットが1つのNUMAノードより大きいワークロードで`zone_reclaim_mode`を有効にすると、通常はスループットが損なわれる。対象のワークロードで、有効化の前後でベンチマークを取ること。

---

## リモートのメモリ圧力がローカルの回収を引き起こす {#remote-memory-pressure-causing-local-reclaim}

Node 1の割り当てがローカルメモリを使い果たしつつある、2ノードのシステムを考える。

```
Node 0: 20 GB free (comfortably above WMARK_HIGH)
Node 1:  1 GB free (below WMARK_LOW, kswapd1 running hard)
```

`zone_reclaim_mode = 0`（デフォルト）のとき、Node 1で実行中のプロセスには何が起きるか。

1. Node 1のページアロケータはNode 1のzoneを順に調べ、それらがウォーターマークを下回っていることを見つけ、`zonelist`を確認する。これにはフォールバック先としてNode 0のzoneが含まれている。
2. `node_reclaim_enabled()`がfalseなので、アロケータは**`node_reclaim()`を呼ばない**。
3. 割り当てはNode 0からただちに成功する。プロセスはリモートのメモリを得るが、ストールはしない。
4. その間、kswapd1はバックグラウンドでNode 1自身のLRUから回収を続け、Node 1を`WMARK_HIGH`より上に戻そうとする。

**決定的なのは、kswapd1はNode 0の空きメモリにかかわらずNode 1のLRUから回収するという点である。** Node 0の余裕を「借りる」仕組みは持っていない。その結果、Node 0がほとんど遊んでいるのに、Node 1は活発に使われているpageをNode 1から回収することがある。

これはバグではなく、汎用的なワークロードにとって正しい動作である。代替案（リモートのメモリに空きがあるからといって回収を止める）では、リモートノードのメモリが際限なく流出し、最終的にはリモートのメモリも枯渇してしまう。

!!!note
    Node 1に圧力がかかっても、Node 1上のプロセスが**Node 0へ自動的に移動することはない**。プロセスの配置はスケジューラの関心事であり、メモリ回収の関心事ではない。プロセスの配置を明示的に制御するには`numactl --cpunodebind`や`taskset`を使うか、スケジューラの負荷分散に頼ること（これはNUMAトポロジを考慮するが、メモリ圧力のかかったノードからプロセスを移動させる保証はない）。

---

## ノードごとの回収の観測 {#observing-per-node-reclaim}

### /proc/vmstat {#procvmstat}

`/proc/vmstat`はシステム全体のカウンタを報告する。回収に最も関係するもの:

```bash
grep -E 'pgsteal|pgscan|kswapd' /proc/vmstat
```

主要なフィールド（文字列名は`mm/vmstat.c`より）:

| カウンタ | 意味 |
|---|---|
| `pgsteal_kswapd` | kswapdが回収したpage（全ノードの合計） |
| `pgsteal_direct` | 直接回収で回収されたpage |
| `pgscan_kswapd` | kswapdが走査したpage |
| `pgscan_direct` | 直接回収で走査されたpage |
| `pgscan_direct_throttle` | 直接回収がスロットリングされた回数 |
| `kswapd_low_wmark_hit_quickly` | kswapdが完全な走査なしにlowウォーターマークを回復した回数 |
| `kswapd_high_wmark_hit_quickly` | kswapdがhighウォーターマークに素早く到達した回数 |
| `zone_reclaim_success` | 十分なpageを回収できた`node_reclaim()`の呼び出し |
| `zone_reclaim_failed` | 十分に回収できなかった`node_reclaim()`の呼び出し |

`zone_reclaim_success`と`zone_reclaim_failed`が非ゼロになるのは、`zone_reclaim_mode != 0`のときだけである。

### ノードごとのvmstat {#per-node-vmstat}

各ノードはsysfsを通じて固有のvmstatカウンタを公開している。

```bash
# ノードごとの空き page
cat /sys/devices/system/node/node0/vmstat
cat /sys/devices/system/node/node1/vmstat

# ここで見えるフィールドの例:
# nr_inactive_anon, nr_active_anon, nr_inactive_file, nr_active_file
# pgsteal_kswapd, pgscan_kswapd, pgsteal_direct, pgscan_direct
```

ノード間で`pgsteal_kswapd`を比較すると、どのノードが回収の負担を担っているかがわかる。大きな非対称性は、メモリ配置の問題を示していることが多い。

### NUMA割り当てのカウンタ {#numa-allocation-counters}

```bash
numastat          # ノードごとのヒット/ミスのカウンタ（/sys/devices/system/node/nodeN/numastat を読む）
numastat -p <pid> # プロセスごとの NUMA マッピング
```

`/sys/devices/system/node/nodeN/numastat`にある、関連するノードごとのカウンタ:

| sysfsでの名前 | `/proc/vmstat`での名前 | 意味 |
|---|---|---|
| `numa_hit` | `numa_hit` | 意図したノードに置かれた割り当て |
| `numa_miss` | `numa_miss` | 別のノードに置かれた割り当て |
| `numa_foreign` | `numa_foreign` | このノード向けだったが別の場所に置かれた割り当て |
| `local_node` | `numa_local` | このノードにローカルなCPUによる割り当て |
| `other_node` | `numa_other` | リモートノード上のCPUによる割り当て |
| `interleave_hit` | `numa_interleave` | インターリーブポリシーで意図したノードに置かれた割り当て |

Node 1で`numa_miss`が増加し、同時にNode 1でkswapdの回収が活発に行われているなら、それはNode 1がメモリ圧力にさらされ、リモートからの割り当てにフォールバックしていることを示す典型的な兆候である。

### NUMAバランシングのカウンタ {#numa-balancing-counters}

```bash
grep numa /proc/vmstat
```

| カウンタ | 意味 |
|---|---|
| `numa_hint_faults` | PROT_NONEフォールトの発生（NUMAヒントのpageが走査され、アクセスされた） |
| `numa_hint_faults_local` | pageがすでにローカルだったヒントフォールト |
| `numa_pages_migrated` | NUMAバランシングによって昇格／マイグレーションに成功したpage |

`numa_pages_migrated` / `numa_hint_faults`の比率が低い場合、ほとんどのヒントフォールトがマイグレーションに失敗していることを意味する。多くの場合、移動先のノードが満杯すぎることが原因である。

---

## チューニング {#tuning}

### ノードごとのホットスポットを減らすためのMPOL_INTERLEAVE {#mpol_interleave-to-reduce-per-node-hotspots}

大きな共有メモリのデータセット（例: データベースのバッファプール、共有キュー）がすべて1つのノードに割り当てられると、そのノードにかかるメモリ圧力によってデータセットの一部が追い出される。インターリーブを使うと、割り当て — そして圧力 — がすべてのノードに分散される。

```c
/* アプリケーションのコード */
#include <numaif.h>

unsigned long nodemask = 0x3;  /* ノード 0 と 1 */
mbind(addr, length, MPOL_INTERLEAVE, &nodemask, 2, 0);
```

```bash
# プロセス起動時に
numactl --interleave=all ./my_application
```

トレードオフ: 個々のアクセスがリモートになることがあり、平均レイテンシが増加する。通常これは、データセットが1つのノードに収まらないほど大きい場合や、NUMAローカルな複数のプロセスがすべて同じデータを必要とする場合には割に合う。

### 透過的ヒュージページとNUMA {#transparent-huge-pages-and-numa}

THPの割り当ては常にまずローカルノードから試みられる。ローカルノードが2MBの連続した割り当てを満たせず、リモートノードなら満たせる場合、カーネルはリモートからの割り当てか、ベースページの割り当てにフォールバックする。THPの要求を複数のノードに分割することはない。ローカルのメモリ圧力下では、kswapdのコンパクション（`kcompactd`）が連続した空き範囲を作るために動作するが、このコンパクションもノード単位である。

NUMAの固定とTHPを併用するアプリケーションでは:

- 両方のノードにまたがる領域への`MADV_HUGEPAGE`は、各VMA範囲がマップされているノードでTHPを試みる。
- `numactl --membind=0 --huge-pages`はTHPの割り当てをNode 0に制限する。Node 0に連続したメモリがなければ、割り当てはベースページにフォールバックするか、ブロックする。

!!!tip
    システム全体には空きメモリがあるのに、あるノードのkswapdが他より常に活発な場合、まず確認すべきは、単一のプロセスや共有セグメントがそのノードに割り当てを集中させていないかである。`numastat -p <pid>`は、プロセスの仮想マッピングがノード間でどう分布しているかを示す。

### node_reclaim_distanceの調整 {#adjusting-nodereclaimdistance}

AMD EPYCや類似のマルチダイプラットフォームでは、2ホップのNUMA距離が32と報告され（デフォルトの`RECLAIM_DISTANCE`である30を上回る）、ソケット内のファブリックを介したアクセスは比較的高速であるにもかかわらず、カーネルがそれをまたいだローカル回収を拒むことがある。距離の閾値は調整できる。

`mm/page_alloc.c`の`node_reclaim_distance`変数のデフォルトは`RECLAIM_DISTANCE`（30）である。これはsysctlとして**公開されていない** — 変更するにはカーネルソースを修正するか、カスタムのカーネルモジュールを使う必要がある。回避策として、これらのプラットフォームでは`vm.zone_reclaim_mode = 0`に設定してローカル回収を完全に無効にするか、リモートからの割り当てを優先するようにNUMAメモリポリシーを調整する。

---

## 主要なソースファイル {#key-source-files}

| ファイル | 含まれるもの |
|---|---|
| `mm/vmscan.c` | `kswapd()`、`kswapd_run()`、`balance_pgdat()`、`pgdat_balanced()`、`wakeup_kswapd()`、`node_reclaim()`、`node_reclaim_mode`変数 |
| `mm/page_alloc.c` | `get_page_from_freelist()`、`zone_watermark_ok()`、`__zone_watermark_ok()`、`zone_allows_reclaim()`、`node_reclaim_distance` |
| `include/linux/mmzone.h` | `struct zone`（ウォーターマークのフィールド）、`struct pglist_data`（kswapdのフィールド）、`enum zone_watermarks`、ウォーターマークのアクセサインライン関数 |
| `mm/migrate.c` | `migrate_misplaced_folio()`、`migrate_misplaced_folio_prepare()`、`migrate_pages()` |
| `mm/memory.c` | `do_numa_page()`、`numa_migrate_check()` |
| `mm/internal.h` | `node_reclaim_enabled()` |
| `include/uapi/linux/mempolicy.h` | `RECLAIM_ZONE`、`RECLAIM_WRITE`、`RECLAIM_UNMAP`、`MPOL_INTERLEAVE` |
| `include/linux/topology.h` | `RECLAIM_DISTANCE`のデフォルト値（30）、`node_reclaim_distance`の宣言 |
| `include/linux/vm_event_item.h` | `PGSTEAL_KSWAPD`、`PGSCAN_KSWAPD`、`NUMA_HINT_FAULTS`、`NUMA_PAGE_MIGRATE`の列挙値 |
| `mm/vmstat.c` | カウンタの文字列名（`pgsteal_kswapd`、`numa_hit`、`numa_miss`、`zone_reclaim_success`など） |
| `kernel/sched/fair.c` | `task_numa_work()` — NUMAヒントPTEのスキャナ |

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [mm/vmscan.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) — `kswapd()`、`kswapd_run()`、`balance_pgdat()`、`pgdat_balanced()`、`wakeup_kswapd()`、`node_reclaim()`、`node_reclaim_mode`
- [mm/page_alloc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) — `get_page_from_freelist()`、`zone_allows_reclaim()`、`node_reclaim_distance`、およびフォールバック用zonelistの走査
- [mm/memory.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memory.c) — `do_numa_page()`と`numa_migrate_check()`: pageのマイグレーションを駆動するNUMAヒントフォールトのパス
- [mm/migrate.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/migrate.c) — `migrate_misplaced_folio()`と`migrate_misplaced_folio_prepare()`

### カーネルドキュメント {#kernel-documentation}

- `Documentation/admin-guide/sysctl/vm.rst` — `zone_reclaim_mode`、`min_free_kbytes`、および関連する回収のつまみを説明している（[レンダリング版](https://docs.kernel.org/admin-guide/sysctl/vm.html)）

### LWNの記事 {#lwn-articles}

- [Per-node reclaim and kswapd](https://lwn.net/Articles/461294/) — ノードごとのkswapdの起床ロジックと、メモリ圧力下でのノード間の相互作用の分析
- [NUMA balancing and page migration](https://lwn.net/Articles/568870/) — 自動NUMAバランシングにおいて、ヒントフォールト、マイグレーションの判断、昇格パスがどう機能するか
- [Zone reclaim mode considered harmful](https://lwn.net/Articles/432224/) — 汎用的なワークロードで`zone_reclaim_mode`を無効にすべき理由

### 関連ドキュメント {#related-docs}

- [NUMAのメモリ管理](numa.md) — NUMAの概要、メモリポリシー、自動バランシング、監視
- [zone回収ポリシー](zone-reclaim.md) — `vm.zone_reclaim_mode`、ウォーターマークブースト、DMA zoneへの圧力の詳細
- [NUMAのzonelist構築とフォールバック順序](numa-zonelist.md) — リモートノードへのフォールバックがどう順序付けられ、`MPOL_BIND`がそれをどう防ぐか
- [回収](reclaim.md) — 回収の仕組みの全体: LRUリスト、`shrink_node()`、直接回収
