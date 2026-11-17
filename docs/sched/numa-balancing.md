# NUMA自動バランシング {#numa-automatic-balancing}

> スケジューラがNUMAローカリティの不一致を検出し、修正する仕組み

## 問題 {#the-problem}

NUMAシステムでは、メモリアクセスのレイテンシは、物理ページをどのNUMAノードが保持しているかによって決まる:

```
Node 0:                 Node 1:
  CPU 0, CPU 1            CPU 2, CPU 3
  Memory 0 (fast)         Memory 1 (fast)
       │                        │
       └──────── QPI/UPI ────────┘
                (slow: ~2× penalty)

Task running on CPU 2 reading pages on Node 0:
  → every cache miss → remote memory access → 2× latency!
```

**NUMA自動バランシング**（CONFIG_NUMA_BALANCING）はこの状況を検出し、ページを実際に使われているノードへマイグレーションする。Linux 3.13でMel Gorman（SUSE）らによって導入された — [`10fc05d0e551`](https://git.kernel.org/linus/10fc05d0e551146ad6feb0ab8902d28a2d3c5624)。

## 動作の仕組み {#how-it-works}

### フェーズ1: ページをアクセス不能にする（PROT_NONE） {#phase-1-mark-pages-as-inaccessible-prot_none}

NUMAバランサは定期的にプロセスのページテーブルを走査し、アクセスされたページのPTEを`PROT_NONE`に変更する（解放せずにアンマップする）:

```c
/* kernel/sched/fair.c */
static void task_numa_work(struct callback_head *work)
{
    struct task_struct *p = current;
    unsigned long start = p->numa_scan_offset;

    /* VA空間の一部を走査し、PTEにNUMAフォールトを仕掛ける: */
    change_prot_numa(p->mm, start, start + NUMA_SCAN_SIZE);
    /* PTEを_PAGE_PROTNONE付きのPROT_NONEに設定する（3.17より前は_PAGE_NUMA） */

    p->numa_scan_offset = start + NUMA_SCAN_SIZE;
    if (p->numa_scan_offset >= p->mm->highest_vm_end)
        p->numa_scan_offset = 0;  /* 先頭に戻る */
}
```

### フェーズ2: NUMAページフォールトを捕捉する {#phase-2-catch-numa-page-faults}

タスクがPROT_NONEのページにアクセスすると、ページフォールトが発生する:

```c
/* mm/memory.c: do_numa_page() */
static vm_fault_t do_numa_page(struct vm_fault *vmf)
{
    struct page *page = vm_normal_page(vmf->vma, vmf->address, vmf->orig_pte);
    int page_nid    = page_to_nid(page);   /* ページの現在のNUMAノード */
    int this_nid    = numa_node_id();       /* CPUのNUMAノード */

    /* 統計のためにアクセスを記録する */
    task_numa_fault(last_cpupid, page_nid, 1, flags);

    /* このページをマイグレーションすべきか？ */
    if (page_nid != this_nid) {
        /* ページをthis_nidへマイグレーションする */
        migrate_misplaced_page(page, vmf->vma, this_nid);
    }

    /* PTEを復元する（再びアクセス可能にする） */
    pte_t pte = pte_modify(vmf->orig_pte, vma->vm_page_prot);
    set_pte_at(vmf->vma->vm_mm, vmf->address, vmf->pte, pte);

    return 0;
}
```

### フェーズ3: タスクのマイグレーション {#phase-3-task-migration}

タスクのページの大部分がNode 1にあるのにNode 0で実行されている場合、スケジューラはページの代わりに（あるいはページに加えて）**タスク**をマイグレーションすることがある:

```c
/* kernel/sched/fair.c */
static bool task_numa_compare(struct task_numa_env *env,
                               long taskimp, long groupimp, bool maymove)
{
    /*
     * 比較: preferred_nodeで実行するタスク vs current_nodeで実行するタスク
     * taskimp: タスクを移動した場合の改善スコア
     * groupimp: タスクグループ全体を移動した場合の改善
     */
    long imp = env->p->numa_group ? groupimp : taskimp;
    if (imp <= env->best_imp)
        return false;
    /* NUMAローカリティが改善 → 最良候補を更新する */
    env->best_imp  = imp;
    env->best_task = env->dst_task;
    env->best_cpu  = env->dst_cpu;
    return true;
}
```

## タスクごとのNUMA統計 {#numa-statistics-per-task}

```c
/* task_structのNUMA関連フィールド: */
struct task_struct {
    /* ... */
    int             numa_preferred_nid;   /* 優先NUMAノード */
    unsigned long   numa_migrate_retry;
    u64             node_stamp;           /* 最後のNUMAバランシング処理 */
    u64             last_task_numa_placement;
    u64             last_sum_exec_runtime;
    struct callback_head    numa_work;

    struct numa_group __rcu *numa_group;  /* 共有メモリのためのタスクグループ */

    unsigned long   *numa_faults;         /* ノードごと・アクセス種別ごとのフォールト数 */
    /* [2 * nr_nodes]: 各ノードについて[ローカルフォールト, リモートフォールト] */
    /* [2]: CPUアクセス vs メモリアクセス */
};
```

## NUMAグループ: 共有メモリのバランシング {#numa-groups-shared-memory-balancing}

複数のタスクがメモリを共有する場合（例: マルチスレッドプログラム）、それらは**NUMAグループ**を形成する:

```c
struct numa_group {
    refcount_t          refcount;
    spinlock_t          lock;
    int                 nr_tasks;
    pid_t               gid;
    int                 active_nodes;

    /* ノードごとのフォールト統計: */
    unsigned long       total_faults;
    unsigned long       max_faults_cpu;

    unsigned long       *faults;       /* faults[node][type] */
    unsigned long       *faults_cpu;

    struct rcu_head     rcu;
    unsigned long       nodes[];       /* 参加しているオンラインノード */
};

/* グループ内のタスクは、共有メモリのローカリティを保つために一緒にマイグレーションされる */
```

## バランシングの走査レート {#balancing-scan-rate}

走査レートは、見つかったNUMAフォールトの数に応じて適応的に変化する:

```c
/* 走査頻度（1秒あたりのページ数）: */
/* 制御: /proc/sys/kernel/numa_balancing_scan_period_min/max */

static void update_task_scan_period(struct task_struct *p,
                                     unsigned long shared, unsigned long private)
{
    /* フォールトが多い → より積極的に走査する（周期を短く） */
    /* フォールトが少ない → 走査頻度を下げる（周期を長く） */

    unsigned int period = p->numa_scan_period;
    if (faults > p->numa_faults_locality[2])
        period = max(period / 2, task_scan_min(p));  /* 速める */
    else
        period = min(period * 2, task_scan_max(p));  /* 遅くする */

    p->numa_scan_period = period;
}
```

## 設定 {#configuration}

```bash
# NUMA自動バランシングを有効化/無効化する:
echo 1 > /proc/sys/kernel/numa_balancing   # 有効化（デフォルト）
echo 0 > /proc/sys/kernel/numa_balancing   # 無効化

# 走査周期（ミリ秒） — 各タスクのVASをどのくらいの頻度で走査するか:
cat /proc/sys/kernel/numa_balancing_scan_period_min  # デフォルト 1000
cat /proc/sys/kernel/numa_balancing_scan_period_max  # デフォルト 60000

# 走査サイズ — 1回の間隔あたりに走査するバイト数:
cat /proc/sys/kernel/numa_balancing_scan_size  # デフォルト 256

# データベースワークロード向けのチューニング（積極的）:
echo 100  > /proc/sys/kernel/numa_balancing_scan_period_min
echo 5000 > /proc/sys/kernel/numa_balancing_scan_period_max
echo 1024 > /proc/sys/kernel/numa_balancing_scan_size
```

## NUMAバランシングの観測 {#observing-numa-balancing}

```bash
# プロセスごとのNUMA統計:
cat /proc/<pid>/sched  # numa_faults, numa_preferred_nid, ...
cat /proc/<pid>/numa_maps
# 7f1234000000 default anon=12 dirty=12 N0=8 N1=4
#                                         ↑N0  ↑N1 = 各ノード上のページ数

# システム全体のNUMAバランシング統計:
cat /proc/vmstat | grep numa
# numa_hint_faults      12345   ← 発生したPROT_NONEフォールト
# numa_hint_faults_local 8000  ← すでに優先ノード上にあったフォールト
# numa_pages_migrated    4345  ← マイグレーションされたページ
# numa_pte_updates       50000 ← PROT_NONEにされたPTE

# perfでのNUMAミス:
perf stat -e dtlb_load_misses.miss_causes_a_walk,node_loads,node_stores \
          -p <pid> sleep 5

# numastat: ノードごとの割り当て統計
numastat <pid>
numastat -m  # ノードごとのシステムメモリ

# NUMAページフォールトとマイグレーションをトレースする:
bpftrace -e '
kprobe:do_numa_page {
    @numa_faults[tid] = count();
}
kprobe:migrate_misplaced_page {
    @migrations = count();
}'

# NUMAトポロジを確認する:
numactl --hardware
# available: 2 nodes (0-1)
# node 0 cpus: 0 1 2 3
# node 1 cpus: 4 5 6 7
# node distances:
# node   0   1
#   0:  10  21
#   1:  21  10
```

## NUMAバランシングを無効にすべき場合 {#when-to-disable-numa-balancing}

```bash
# 逆効果になるワークロードでは無効にする:
# 1. すでにNUMAピン留めされているワークロード:
numactl --cpunodebind=0 --membind=0 ./myapp
# （明示的な配置は自動バランシングに勝る）

# 2. レイテンシに敏感なワークロード:
# NUMAフォールトは1回あたり約10µsを追加する — RTタスクでは避ける
echo 0 > /proc/sys/kernel/numa_balancing

# 3. tmpfs/huge pageのワークロード:
# tmpfsやhuge page上のページはNUMAバランサに走査されない
# 有効にしても利点がない

# 4. ほぼ逐次的な単一ノードアクセス:
# アプリがすでにローカルノード上のデータにアクセスしているなら、バランシングは
# 利点なしにオーバーヘッド（走査 + フォールト）を追加するだけである
```

## NUMAを意識したメモリ割り当て {#numa-aware-memory-allocation}

NUMAを明示的に制御したいアプリケーション向け:

```c
/* -lnumaとリンクする */
#include <numa.h>

/* ノード0上に割り当てる: */
void *mem = numa_alloc_onnode(size, 0);

/* ローカルノード（このスレッドが動作しているノード）上に割り当てる: */
void *mem = numa_alloc_local(size);

/* 以降の割り当てに対するメモリポリシーを設定する: */
struct bitmask *mask = numa_bitmask_alloc(numa_num_configured_nodes());
numa_bitmask_setbit(mask, 0);
numa_set_membind(mask);   /* ノード0からのみ割り当てる */
```

## 参考文献 {#further-reading}

- [NUMA](../mm/numa.md) — NUMAアーキテクチャとzonelist
- [NUMA Distance](../mm/numa-distance.md) — ACPI SRATと距離行列
- [NUMA Reclaim](../mm/numa-reclaim.md) — ノードごとの回収ポリシー
- [CFS Load Balancing](load-balancing.md) — CPUスケジューラの負荷分散
- [NUMA Zonelist](../mm/numa-zonelist.md) — メモリ割り当てのフォールバック
- `kernel/sched/fair.c` — task_numa_work
- `mm/memory.c` — do_numa_page
- `mm/migrate.c` — migrate_misplaced_page
