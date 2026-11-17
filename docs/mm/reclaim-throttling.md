# 回収のスロットリングとバランシング {#reclaim-throttling-and-balancing}

> 回収が追いつかないとき、カーネルはどのようにアロケータを減速させるか

メモリ逼迫が激しいとき、2つのことが同時に起きてはならない。すなわち、
割り当てを行う単一のプロセスが回収に没頭してCPUを独占してはならず、
またアロケータが突き進み続ける間に回収が圧倒されてもならない。
カーネルはスロットリングによって両方の問題を解決する。割り当てを行うプロセスを
一時的に停止させて回収を進行させ、kswapdと直接回収を行う側（direct reclaimer）の
間を調整して負荷を分散させるのである。

## 主要なソースファイル {#key-source-files}

| ファイル | 関連する関数 |
|------|--------------------|
| [`mm/vmscan.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) | `reclaim_throttle()` — スロットリングの中核となるスリープ; `throttle_direct_reclaim()` — pfmemallocリザーブのスロットリング; `consider_reclaim_throttle()` — NOPROGRESSパス; `balance_pgdat()` — kswapdのノードバランシング; `shrink_node()` — ノードごとの回収; `allow_direct_reclaim()` — 起床条件; `pgdat_balanced()` — ウォーターマークのチェック; `writeback_throttling_sane()` — dirtyスロットリングの利用可否 |
| [`mm/memcontrol.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c) | `__mem_cgroup_handle_over_high()` — memory.highの強制; `reclaim_high()` — cgroupの回収ループ; `calculate_high_delay()` — 比例ペナルティ; `mem_find_max_overage()` / `swap_find_max_overage()` — 祖先の超過量スキャン |
| [`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) | `__alloc_pages_slowpath()` — 割り当ての低速パス; `__alloc_pages_direct_reclaim()` / `__perform_reclaim()` — 直接回収の入口; `try_to_free_pages()`の呼び出し箇所 |
| [`include/linux/mmzone.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mmzone.h) | `enum vmscan_throttle_state`、`enum pgdat_flags`、`enum lruvec_flags`、`reclaim_wait[]`待ちキュー、`pfmemalloc_wait`待ちキュー |

---

## なぜスロットリングが存在するか {#why-throttling-exists}

スロットリングがなければ、高速にメモリを割り当てるアプリケーションは
直接回収を延々と繰り返し、pageを解放しようとしてCPUコアを丸ごと消費しかねない。
他のプロセスは飢餓状態に陥り、実際の仕事の進行は停滞する。より微妙な点として、
回収自体が回復できない状態に陥ることがある。見つかるpageがすべてライトバック中であれば、
より積極的にスキャンしてもCPUを浪費するだけである。正しい対応は、一時停止して待つことである。

スロットリングは2つの目的を達成する:

1. **公平性** — 1つのプロセスが回収パスを独占することを防ぐ。
2. **バックプレッシャー** — 回収が追いつかないとき、無駄な作業でスピンするのではなくアロケータを減速させる。

カーネルはスロットリングの理由をいくつか区別しており、それぞれが独自の
待ちキューとタイムアウトを持つ:

```
enum vmscan_throttle_state {
    VMSCAN_THROTTLE_WRITEBACK,   /* pageがライトバックより速くLRUを循環している */
    VMSCAN_THROTTLE_ISOLATED,    /* pageを隔離している並列回収者が多すぎる */
    VMSCAN_THROTTLE_NOPROGRESS,  /* 高い優先度で回収がまったく進まない */
    VMSCAN_THROTTLE_CONGESTED,   /* 輻輳したデバイスを背後に持つdirty page */
    NR_VMSCAN_THROTTLE,
};
```

`pg_data_t`構造体（NUMAノードごとに1つ）は、スロットリングの理由ごとに1つの待ちキューを持ち、
さらに直接回収のpfmemallocスロットリング用に別のキューを持つ:

```c
/* include/linux/mmzone.h */
wait_queue_head_t reclaim_wait[NR_VMSCAN_THROTTLE];
wait_queue_head_t pfmemalloc_wait;
```

---

## 直接回収とkswapd {#direct-reclaim-vs-kswapd}

カーネルには2つの回収の担い手がある:

```
Allocation path                     Background
───────────────                     ──────────
alloc_pages()                       kswapd (per-node kernel thread)
    │                                   │
    └─ __alloc_pages_slowpath()         └─ balance_pgdat()
           │                                    │
           └─ __alloc_pages_direct_reclaim()    └─ kswapd_shrink_node()
                  │                                     │
                  └─ __perform_reclaim()                └─ shrink_node()
                         │
                         └─ try_to_free_pages()
                                │
                                └─ shrink_zones()
                                       │
                                       └─ shrink_node()
```

**kswapd**はバックグラウンドで動作し、各ゾーンの空きpageが`high`ウォーターマークを
上回るまでpageを回収する。割り当てを行うプロセスをブロックすることはない。

**直接回収**は、`__alloc_pages_slowpath()`が既存の空きpageから割り当てを満たせないときに、
割り当てを行うプロセス自身のコンテキストで実行される。`alloc_pages()`を呼び出したプロセスは
`try_to_free_pages()`に入り、そこでスロットリングされることがある。

!!! note "優先度の段階的上昇"
    どちらのパスも、`DEF_PRIORITY`（12）から始まり0に向かって下がっていく
    回収優先度を使う。優先度*p*では、各LRUリストのスキャンは
    `list_size >> p`個のpageを対象とする。優先度が低いほどメモリのより大きな割合をスキャンし、
    優先度0でリスト全体が調べられる。

---

## `reclaim_throttle()` — 中核となるスリープ関数 {#reclaim_throttle-core-sleep-function}

`mm/vmscan.c`の`reclaim_throttle()`は、pfmemalloc以外のすべてのスロットリングが
通過する唯一の関数である:

```c
void reclaim_throttle(pg_data_t *pgdat, enum vmscan_throttle_state reason)
```

この関数は`pgdat->reclaim_wait[reason]`から適切な待ちキューを選び、
呼び出し元タスクを`TASK_UNINTERRUPTIBLE`で、理由に応じたタイムアウトの間スリープさせる:

| スロットリングの理由 | タイムアウト | 発動条件 |
|-----------------|---------|----------------|
| `VMSCAN_THROTTLE_WRITEBACK` | `HZ/10`（100 ms） | ライトバックが掃き出せるより速くpageがLRUを循環している（`sc->nr.immediate`が非ゼロ） |
| `VMSCAN_THROTTLE_ISOLATED` | `HZ/50`（20 ms） | 多すぎる並列回収者が同時にLRUからpageを隔離している |
| `VMSCAN_THROTTLE_NOPROGRESS` | 1 jiffy | 優先度1の直接回収で回収されたpageがゼロ |
| `VMSCAN_THROTTLE_CONGESTED` | 1 jiffy | すべてのdirty pageが輻輳したデバイスを背後に持つ（`LRUVEC_CGROUP_CONGESTED`または`LRUVEC_NODE_CONGESTED`がセットされている） |

!!! info "カーネルスレッドは免除される"
    `reclaim_throttle()`は、`PF_USER_WORKER`または`PF_KTHREAD`がセットされた
    タスク（kswapd自身を除く）に対してはスロットリングをスキップする。ジャーナリングスレッド、
    ライトバックワーカー、その他のカーネルスレッドは、回収が前進するために依存しているので、
    自由に実行できなければならない。

`VMSCAN_THROTTLE_WRITEBACK`では、コードはスロットリング開始以降にライトバックを完了した
page数（`pgdat->nr_reclaim_start`）も追跡する。これにより、`__acct_reclaim_writeback()`は
十分なpageがcleanになった時点で、常にタイムアウトいっぱいまで待つのではなく、
スロットリングされたタスクを早めに起こすことができる。

**スロットリングされたタスクを誰が起こすのか**?  kswapdの`balance_pgdat()`は、
`allow_direct_reclaim()`が真になると`pfmemalloc_wait`キューを起こす。
`reclaim_wait[]`キューは、回収効率が12%を超えたとき（スキャン数に対する回収数の比が`> 1/8`）に
`consider_reclaim_throttle()`によって起こされ、またkswapdが輻輳状態をクリアしたときにも起こされる。

---

## `throttle_direct_reclaim()` — pfmemallocリザーブのスロットリング {#throttle_direct_reclaim-pfmemalloc-reserve-throttle}

これは目的の異なる別のスロットリングパスである。カーネルは、ネットワーク越しのスワップに必要な
ネットワークスタックの割り当てのために、低位ゾーンのpageの一部（`pfmemalloc`リザーブ）を
予約している。これらのリザーブが枯渇した場合、さらなる直接回収を許すと逆効果になる。
kswapdが妨げられずに処理すべきである。

```c
static bool throttle_direct_reclaim(gfp_t gfp_mask,
                                    struct zonelist *zonelist,
                                    nodemask_t *nodemask)
```

この関数は、回収作業が始まる前に`try_to_free_pages()`の先頭で呼ばれる。
次の場合は直ちに（スロットリングせずに）戻る:

- 呼び出し元タスクに`PF_KTHREAD`がセットされている（カーネルスレッド）。
- 致命的なシグナルが保留中である（プロセスを終了させ、自然にメモリを
  解放させる）。
- `allow_direct_reclaim()`が真を返す — `ZONE_NORMAL`以下のすべてのゾーンにわたる空きpageの合計が、それらのゾーンにわたって累積したpfmemallocリザーブの半分を超えている。

スロットリングが適用されると、プロセスは`pgdat->pfmemalloc_wait`でスリープする:

```c
/* 呼び出し元はファイルシステムに入れない — ジャーナルロックを保持しているかもしれない */
if (!(gfp_mask & __GFP_FS))
    wait_event_interruptible_timeout(pgdat->pfmemalloc_wait,
                                     allow_direct_reclaim(pgdat), HZ);
else
    /* kswapd が前進するまでブロックする */
    wait_event_killable(pgdat->pfmemalloc_wait,
                        allow_direct_reclaim(pgdat));
```

スロットリングのイベントごとに`PGSCAN_DIRECT_THROTTLE` vmstatカウンタがインクリメントされる
（`/proc/vmstat`では`pgscan_direct_throttle`として見える）。

`allow_direct_reclaim()`は最下位から`ZONE_NORMAL`までのすべてのゾーンを走査し、空きpageとpfmemallocリザーブの両方を累積したうえで、`free_pages > pfmemalloc_reserve / 2`かどうかを確認する。ノードが絶望的な状態（`kswapd_test_hopeless()`）にある場合、直接回収は常に許可される。

!!! warning "致命的シグナルの扱い"
    `throttle_direct_reclaim()`が`true`を返した場合、スリープ中に致命的なシグナルが
    届いたことを意味する。その場合`try_to_free_pages()`は1を返し、
    ページアロケータはOOM killerをスキップする。プロセスはすでに死につつあるからである。

---

## `consider_reclaim_throttle()` — 進捗なしのスロットリング {#consider_reclaim_throttle-no-progress-throttle}

直接回収中の各`shrink_zones()`パスの後、
`consider_reclaim_throttle()`がスロットリングが妥当かどうかを評価する:

```c
static void consider_reclaim_throttle(pg_data_t *pgdat,
                                      struct scan_control *sc)
```

- 回収効率が12.5%を上回る場合（`sc->nr_reclaimed > sc->nr_scanned >> 3`）、
  `VMSCAN_THROTTLE_NOPROGRESS`ですでにスリープしているタスクを起こす。
- 優先度1（LRU全体をスキャンする1段階手前）にあり、
  `sc->nr_reclaimed`がまだゼロであれば、
  `reclaim_throttle(pgdat, VMSCAN_THROTTLE_NOPROGRESS)`を呼ぶ。
- kswapdとcgroupの回収は免除される。これらはI/Oで停滞したときに代わりに
  `VMSCAN_THROTTLE_WRITEBACK`を使う。

---

## メモリcgroupのスロットリング — `memory.high`パス {#memory-cgroup-throttling-the-memoryhigh-path}

### 比例遅延 {#the-proportional-delay}

cgroup v2の`memory.high`はソフトリミットである。これを超えても、cgroupが直ちに
killされたり強制停止されたりはしない。代わりに、超過したプロセスは、
cgroup（またはその祖先のいずれか）がリミットをどれだけ超えたかに比例して減速させられる。

割り当てによってcgroupが`memory.high`を超えると、チャージパスは
`current->memcg_nr_pages_over_high`をセットする。ユーザー空間に戻る際に（あるいは
カウントが`MEMCG_CHARGE_BATCH`を超えた場合はそれより早く）、
`__mem_cgroup_handle_over_high()`が呼ばれる:

```
Allocation exceeds memory.high
           │
           v
    memcg_nr_pages_over_high += batch
           │
           v (on return to userspace or if count > MEMCG_CHARGE_BATCH)
    __mem_cgroup_handle_over_high()
           │
           ├── reclaim_high()              ← try_to_free_mem_cgroup_pages()
           │       walking up cgroup tree
           │
           ├── calculate_high_delay()      ← exponential penalty in jiffies
           │
           └── schedule_timeout_killable(penalty_jiffies)
```

`reclaim_high()`はcgroup階層を上へたどり、自身の`memory.high`リミットを超えている
各祖先から回収する。最大`MAX_RECLAIM_RETRIES`（16）回の回収を試みた後も
使用量がリミットを超えていれば、`calculate_high_delay()`がスリープ時間を計算する。

### 遅延の計算式 {#the-delay-formula}

ペナルティは`calculate_high_delay()`によって計算される:

```c
penalty_jiffies = max_overage * max_overage * HZ;
penalty_jiffies >>= MEMCG_DELAY_PRECISION_SHIFT;   /* 20ビット */
penalty_jiffies >>= MEMCG_DELAY_SCALING_SHIFT;     /* 14ビット */

/* MEMCG_CHARGE_BATCH に対する nr_pages の比でスケールする */
return penalty_jiffies * nr_pages / MEMCG_CHARGE_BATCH;
```

`max_overage`は、cgroup（またはその最悪の祖先）が`memory.high`をどれだけ超えているかを示す
固定小数点の比率であり、次のように計算される:

```c
overage = (usage - high) << MEMCG_DELAY_PRECISION_SHIFT;
overage = overage / high;   /* リミットに対する割合 */
```

2乗の項があるため、ペナルティは超過量とともに急速に増大する。リミットの2倍を超えた
cgroupは、わずかに超えただけのものよりはるかに重いペナルティを受ける。
最終結果はユーザー空間への復帰1バッチあたり`MEMCG_MAX_HIGH_DELAY_JIFFIES`（2秒）に
制限されるので、アプリケーションが永久にハングすることはなく、
非常に遅く動くだけである。

!!! note "メモリとスワップのペナルティは別々"
    `calculate_high_delay()`は2回呼ばれる。1回は`mem_find_max_overage()`の結果
    （メモリの超過量）で、もう1回は`swap_find_max_overage()`の結果（スワップの超過量）で
    呼ばれる。階層内の祖先はそれぞれ独立にたどられ、ペナルティは加算される。

!!! note "わずかな超過は無料"
    `penalty_jiffies <= HZ/100`（10 ms）であれば、タスクはまったくスリープさせられない。
    これにより、リミットをわずかに超えただけで、すでに回収に成功しつつあるかもしれない
    cgroupにペナルティを課すことを避ける。

---

## 書き込みのスロットリングと回収の停滞 {#write-throttling-and-reclaim-stalls}

### `PGDAT_WRITEBACK`とライトバックのスロットリング {#pgdat_writeback-and-the-writeback-throttle}

`shrink_node()`が、調べたすべてのpageがライトバック中であること
（`sc->nr.writeback == sc->nr.taken`）を見つけると、ノードに`PGDAT_WRITEBACK`フラグを
セットする。これはkswapdに対し、dirtyのライトバックがLRUスキャンと同じペースで進んでおり、
pageがストレージの掃き出し速度よりも速くリストを循環していることを知らせる。

`sc->nr.immediate`が非ゼロ — すなわちpageがライトバック中で*かつ*すでに
非アクティブLRUの末尾に戻ってきている — の場合、`shrink_node()`は次を呼ぶ:

```c
reclaim_throttle(pgdat, VMSCAN_THROTTLE_WRITEBACK);
```

これは最大100 ms、または十分なpageがライトバックを完了した後に
`__acct_reclaim_writeback()`がキューを起こすまでスリープする。

### `LRUVEC_NODE_CONGESTED`と`VMSCAN_THROTTLE_CONGESTED` {#lruvec_node_congested-and-vmscan_throttle_congested}

スキャンしたすべてのdirty pageがdirtyかつ輻輳している場合
（`sc->nr.dirty == sc->nr.congested`）、ノードは`LRUVEC_NODE_CONGESTED`（kswapdがセット）
または`LRUVEC_CGROUP_CONGESTED`（cgroupの回収がセット）によって輻輳状態とマークされる。
いずれかのフラグを検出した直接回収者は、次を呼ぶ:

```c
reclaim_throttle(pgdat, VMSCAN_THROTTLE_CONGESTED);
```

kswapd自身はここではスロットリングされない。kswapdを停滞させると回収の進行がすべて
止まってしまうため、kswapdは自由にスキャンを続けられる。スリープするのは
ユーザーコンテキストの直接回収者だけである。

### `balance_dirty_pages()` — 上流側のスロットリング {#balance_dirty_pages-the-upstream-throttle}

`mm/page-writeback.c`の書き込みスロットリングパスは、ライトバックが掃き出せるより速く
pageをdirtyにするプロセスを減速させる主要なメカニズムである。`balance_dirty_pages()`は
ファイルシステムの書き込みパスから呼ばれ、回収パスから直接呼ばれることはない。
しかし両者は相互作用する。`balance_dirty_pages()`が利用できない場合（たとえばライトバックと
memcgが切り離されているレガシーなcgroup v1メモリコントローラ）、
`writeback_throttling_sane()`は`false`を返し、`shrink_folio_list()`はライトバックI/Oで
直接停滞する方法にフォールバックする。この方法には、通常のdirtyスロットリングが持つ
公平性や帯域比例性がない。

---

## 逼迫下での公平な回収 {#fair-reclaim-under-pressure}

### 回収優先度とcgroup階層 {#reclaim-priority-and-the-cgroup-hierarchy}

複数のcgroupがすべてメモリ逼迫下にあるとき、カーネルは単純に最も多くのメモリを持つ
cgroupから回収するわけではない。`shrink_node_memcgs()`はノードのlruvecツリー内の
すべてのmemcgを走査し、それぞれに対して`shrink_lruvec()`を呼ぶ。各cgroupからスキャンされる量は、
ノード上の回収可能なpageに占めるそのcgroupの割合に比例し、設定された`swappiness`によって
調整される。

回収優先度は`DEF_PRIORITY`（12）から始まり、逼迫が強まるにつれて0に向かって
デクリメントされる。各優先度レベルで、スキャンは各LRUリストの`1 >> priority`の割合を訪れる。
優先度が低いほど大きな割合をスキャンし、
優先度0ではリスト全体をスキャンする。

### `reclaim_high()`でのcgroup階層の走査 {#cgroup-hierarchy-walk-in-reclaim_high}

`memory.high`の強制において、`reclaim_high()`は直接超過したcgroupから回収するだけではない。
`parent_mem_cgroup()`を介してルートまでたどり、自身の`memory.high`も超えている各祖先から
回収する。これにより、直接の親がたまたま自身のリミット内に収まっているというだけで、
深くネストしたcgroupが祖先のリミットから逃れることを防ぐ。

```c
do {
    if (page_counter_read(&memcg->memory) > READ_ONCE(memcg->memory.high))
        nr_reclaimed += try_to_free_mem_cgroup_pages(...);
} while ((memcg = parent_mem_cgroup(memcg)) &&
         !mem_cgroup_is_root(memcg));
```

`mem_find_max_overage()`関数は、ペナルティの遅延を計算する際に同様の祖先の走査を行い、
最悪の（最もリミットを超えている）祖先を選んでペナルティの大きさを決める。

---

## `balance_pgdat()` — kswapdのノードバランシング {#balance_pgdat-kswapd-node-balancing}

kswapdのメインループは`balance_pgdat()`を呼び、少なくとも1つのゾーンがバランスするまで
ノード上のpageを回収する。ゾーンは、`pgdat_balanced()`による判定で空きpageが
`high`ウォーターマークを超えたときにバランスしたとみなされる:

```c
static bool pgdat_balanced(pg_data_t *pgdat, int order, int highest_zoneidx)
```

この関数はゾーンを下位から上位へ確認し、いずれか1つのゾーンが`high_wmark_pages()`の
しきい値で`__zone_watermark_ok()`を満たした時点で`true`を返す。

`pgdat_balanced()`が真になると、kswapdは`prepare_kswapd_sleep()`を呼び、これは次を行う:

1. `pgdat->pfmemalloc_wait`でスリープしているプロセスを起こす（それらは
   pfmemallocリザーブが補充されるのを待っていた）。
2. `clear_pgdat_congested()`を呼んで`PGDAT_WRITEBACK`、
   `LRUVEC_NODE_CONGESTED`、`LRUVEC_CGROUP_CONGESTED`をクリアする。

`balance_pgdat()`は、`allow_direct_reclaim()`が真になるたびにループの途中でも
`pfmemalloc_wait`を起こすので、スロットリングされたアロケータはkswapdがノードを
完全にバランスさせるまで待たされることはない。

---

## スロットリングの観察 {#observing-throttling}

### PSI — 最も直接的なシグナル {#psi-the-most-direct-signal}

Pressure Stall Informationは、スロットリングが対処する現象、すなわちメモリを待って
停滞しているタスクをそのまま捉える。詳しい説明は[PSI](psi.md)を参照。回収のスロットリング中、
`reclaim_throttle()`や`throttle_direct_reclaim()`で待っているタスクは、
`psi_memstall_enter()` / `psi_memstall_leave()`によってメモリストールとして計上される。

```bash
# スナップショット — "full" は実行可能なすべてのタスクが停滞していたことを意味する
cat /proc/pressure/memory
# some avg10=8.42 avg60=3.11 avg300=1.05 total=12450183
# full avg10=2.17 avg60=0.84 avg300=0.27 total=3241885
```

`full`のメモリプレッシャーの上昇は、スロットリングが作動しており、システムが十分な速さで
回収できていないことの信頼できる指標である。

### `/proc/vmstat`のカウンタ {#procvmstat-counters}

```bash
grep -E 'pgscan|pgsteal|allocstall|throttle' /proc/vmstat
```

| カウンタ | 意味 |
|---------|---------|
| `pgscan_kswapd` | kswapdがスキャンしたpage数 |
| `pgsteal_kswapd` | kswapdが回収したpage数 |
| `pgscan_direct` | 直接回収中にスキャンされたpage数 |
| `pgsteal_direct` | 直接回収中に回収されたpage数 |
| `pgscan_direct_throttle` | `throttle_direct_reclaim()`がプロセスをブロックした回数 |
| `allocstall_*` | ゾーンクラスごとに`try_to_free_pages()`に入った回数 |

**比率の解釈**:

- 健全なシステムでは、`pgscan_kswapd`が`pgscan_direct`を大きく上回る。kswapdが
  バックグラウンドで逼迫を処理しており、直接回収はめったに作動しない。
- `pgscan_kswapd`に対して`pgscan_direct`が高い場合は、kswapdが追いつけず、
  割り当てを行うプロセスが自ら回収を行っていることを意味する。これは
  持続的なメモリ逼迫の兆候である。
- `pgscan_direct_throttle`の上昇は、pfmemallocリザーブが繰り返し枯渇していることを意味する。
  システムはkswapdが処理できる限界に近いかもしれない。
- `allocstall_*`のカウントが高いことは、`try_to_free_pages()`に頻繁に入っていることを示す。
  各インクリメントは、プロセスが直接回収に入ったことに対応する
  （スロットリングのイベントそのものではなく、回収への進入である）。

!!! tip "steal/scanの比率に注目する"
    `pgsteal / pgscan`は回収効率である。この比率がおよそ50%を下回る場合、
    回収は解放できない多くのpageをスキャンしていることを意味する。原因は多くの場合、参照された
    page、ピン留めされたpage、または激しい書き込み負荷である。この低い効率こそが
    `VMSCAN_THROTTLE_NOPROGRESS`を引き起こす。

---

## チューニング {#tuning}

### `vm.swappiness` {#vmswappiness}

カーネルが匿名（スワップバック）pageの回収とファイルバックpageの回収に割り当てる
相対的なコストを制御する。内部変数は`vm_swappiness`で、デフォルトは60である。

- **0** — ファイルキャッシュの回収を強く優先する。匿名pageは他に選択肢がない場合にのみ
  スワップされる（注: Linux 3.5以降、スワップを完全に無効化するわけではない）。
- **60** — デフォルトのバランス。匿名pageとファイルpageの両方が候補となり、
  ファイルがやや優先される。
- **100** — 匿名pageとファイルpageの回収コストを等しく扱う。
- **200** — ファイルキャッシュが回収可能な場合でも匿名pageを積極的にスワップする
  （プロアクティブな回収シナリオのために5.8で導入された）。

`swappiness`を低くすると、より多くのファイルキャッシュを破棄する代償として、スワップによるI/Oが減る。
スワップが豊富でディスクが遅いシステムでは、ファイル書き込みが支配的な場合に
`VMSCAN_THROTTLE_WRITEBACK`イベントが増えることがある。

```bash
# 現在の値を読む
sysctl vm.swappiness

# 一時的に設定する
sysctl -w vm.swappiness=10

# 再起動後も維持する
echo "vm.swappiness = 10" >> /etc/sysctl.d/99-memory.conf
```

### `vm.vfs_cache_pressure` {#vmvfs_cache_pressure}

ページキャッシュに対して、カーネルがdentryとinode（VFSのslabキャッシュ）をどれだけ積極的に
回収するかを制御する。内部変数は`fs/dcache.c`の`sysctl_vfs_cache_pressure`で、
デフォルトは100である。

- **< 100** — カーネルはslabキャッシュをページキャッシュほど積極的には回収しない。
  メタデータの多いワークロード（多数の小さなファイル）は、低い値の恩恵を受ける。
- **100** — デフォルト。slabとページキャッシュは等しく重み付けされる。
- **> 100** — slabキャッシュがより積極的に回収される。メタデータが追い出されたときの
  VFSルックアップのオーバーヘッドが増える代わりに、dentry/inodeのメモリが減る。

```bash
sysctl vm.vfs_cache_pressure
sysctl -w vm.vfs_cache_pressure=50
```

### `vm.dirty_ratio`と`vm.dirty_background_ratio` {#vmdirty_ratio-and-vmdirty_background_ratio}

これらは、主要な書き込みスロットリングである`balance_dirty_pages()`がいつ作動するかを制御する。
回収がライトバック中のpageに遭遇するかどうか、したがって`VMSCAN_THROTTLE_WRITEBACK`が
どれだけ頻繁に発動するかに直接影響する。

| sysctl | 内部変数 | デフォルト | 意味 |
|--------|------------------|---------|---------|
| `vm.dirty_background_ratio` | `dirty_background_ratio` | 10% | `flusher`スレッドがバックグラウンドライトバックを開始する、dirty化可能なメモリの割合 |
| `vm.dirty_ratio` | `vm_dirty_ratio` | 20% | 書き込みを行うプロセスが`balance_dirty_pages()`でスロットリングされる割合 |

- `dirty_background_ratio`を下げるとバックグラウンドライトバックが早く始まり、
  dirtyプールが小さく保たれ、ライトバックによる回収の停滞が減る。
- `dirty_ratio`を下げると書き込み側が早くスロットリングされるが、
  アプリケーションの書き込みレイテンシが高くなる。
- 大容量RAM（数百GB）を持つシステムでは、割合ベースの比率では巨大なdirtyプールを
  許してしまうことがある。代わりに絶対値の制限として`vm.dirty_bytes`と
  `vm.dirty_background_bytes`の使用を検討するとよい。

```bash
# 256 GB のシステムで dirty データを上限内に保つ
sysctl -w vm.dirty_background_bytes=1073741824   # 1 GB
sysctl -w vm.dirty_bytes=4294967296              # 4 GB
```

---

## まとめ: 逼迫の段階的エスカレーション {#putting-it-together-a-pressure-escalation-sequence}

```
1. Allocation demand rises; kswapd wakes and runs balance_pgdat().
   └── pgscan_kswapd increases; system healthy.

2. kswapd falls behind; free pages drop below min watermark.
   └── __alloc_pages_slowpath() → __alloc_pages_direct_reclaim()
   └── try_to_free_pages() entered; allocstall_* increments.

3. Direct reclaim finds pages under writeback.
   └── sc->nr.immediate > 0
   └── reclaim_throttle(VMSCAN_THROTTLE_WRITEBACK) — 100 ms sleep.
   └── PSI memory stall accounting begins.

4. Reclaim makes no progress at priority 1.
   └── consider_reclaim_throttle() → VMSCAN_THROTTLE_NOPROGRESS.

5. pfmemalloc reserves depleted.
   └── throttle_direct_reclaim() → pfmemalloc_wait sleep.
   └── pgscan_direct_throttle increments.
   └── kswapd woken (allow_direct_reclaim → wake_up kswapd_wait).

6. kswapd recovers the node; pgdat_balanced() becomes true.
   └── wake_up_all(&pgdat->pfmemalloc_wait).
   └── clear_pgdat_congested().
   └── Direct reclaimers wake and retry allocations.
```

ステップ3以降では、PSIの`memory.full`プレッシャーが上昇する。この一連の流れを
`/proc/vmstat`のカウンタの差分とPSIで監視すれば、システムがどのスロットリング段階に達し、
そこでどれだけの時間を費やしているかを正確に把握できる。

## 参考文献 {#further-reading}

- [mm/vmscan.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) — `reclaim_throttle()`、`throttle_direct_reclaim()`、`consider_reclaim_throttle()`、`balance_pgdat()`、`allow_direct_reclaim()`、`pgdat_balanced()` — スロットリングとkswapdのバランシングの完全な実装
- [mm/memcontrol.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c) — `__mem_cgroup_handle_over_high()`、`reclaim_high()`、`calculate_high_delay()`、`mem_find_max_overage()`、`swap_find_max_overage()` — `memory.high`の比例ペナルティパス
- `Documentation/admin-guide/cgroup-v2.rst` — `memory.high`のセマンティクス、ソフトリミット超過時のスロットリング動作、`memory.max`との関係
- [LWN: Throttling memory-hungry cgroups](https://lwn.net/Articles/758781/) — `memory.high`の比例遅延の元となった設計議論。2乗のペナルティ式と、単純な強制停止が却下された理由を説明している
- [回収](reclaim.md) — 回収の全体像: kswapd、直接回収、ウォーターマーク、LRUリスト、swappiness
- [memcg](memcg.md) — 運用者の視点から見た`memory.high`と`memory.max`: リミットの設定方法と`memory.events`の解釈
