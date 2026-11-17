# メモリ階層化 {#memory-tiering}

> 異種メモリの管理: DRAM、CXL、PMEM、HBM

## なぜメモリ階層化か {#why-memory-tiering}

現代のサーバーは、性能特性の異なる複数種類のメモリを搭載することが増えている。

| メモリの種類 | 帯域幅 | レイテンシ | 容量 | コスト |
|-------------|-----------|---------|----------|------|
| HBM（オンパッケージ） | 1+ TB/s | ~10ns | 32-128GB | 高 |
| DDR5 DRAM | 50-100 GB/s | ~80ns | 256GB-4TB | 中 |
| CXLメモリ | 30-50 GB/s | ~200ns | 1-16TB | やや低 |
| PMEM（Optane） | 20-40 GB/s | ~300ns | 1-8TB | 最低 |

カーネルは次のことを行わなければならない。
1. どのメモリノードがどの階層に属するかを **識別** する
2. ホットなpageを高速メモリに、コールドなpageを低速メモリに **配置** する
3. ホットになったコールドなpageを **昇格（promote）** させる
4. コールドになったホットなpageを（メモリプレッシャー下で） **降格（demote）** させる

## メモリ階層 {#memory-tiers}

```c
/* include/linux/memory-tiers.h */
struct memory_tier {
    struct list_head    list;
    struct list_head    memory_types;  /* この階層に属するメモリ種別（NUMAノード） */
    int                 adistance;     /* 抽象距離: 小さいほど高速 */
    struct device       dev;
};

/* デフォルトの階層割り当て: */
/* DRAM: adistance = DRAM_ADISTANCE (200) */
/* CXL:  adistance = CXL_ADISTANCE  (300) */
/* PMEM: adistance = PMEM_ADISTANCE  (400) */
```

### 階層の検出 {#tier-discovery}

```bash
# メモリ階層を表示する
cat /sys/devices/virtual/memory_tiering/memory_tierN/adistance
# 200   ← DRAM 階層

# NUMA ノードとその階層割り当て
for node in /sys/devices/system/node/node*/; do
    echo -n "$(basename $node): "
    cat "$node/memory_tier" 2>/dev/null || echo "none"
done

# numactl: メモリノードを表示する
numactl --hardware
# available: 4 nodes (0-3)
# node 0 cpus: 0-15
# node 0 size: 256000 MB     ← DRAM
# node 2 cpus:               ← CPU なし: CXL または PMEM のメモリノード
# node 2 size: 512000 MB
```

## pageの降格 {#page-demotion}

高速メモリがプレッシャー下にあるとき、カーネルはコールドなpageを低速メモリノードへ降格させる。

```c
/* mm/migrate.c */
int migrate_pages_to_node(struct list_head *pagelist, int node,
                           enum migrate_mode mode)
{
    /* リスト上のページを指定したNUMAノードへ移動する */
    return migrate_pages(pagelist, alloc_migration_target_page,
                          NULL, node, mode, MR_DEMOTION);
}

/* mm/vmscan.c: 高速メモリが不足するとkswapdが降格を行う */
static bool demote_page_list(struct list_head *demote_pages,
                              struct pglist_data *pgdat)
{
    int target_nid = next_demotion_node(pgdat->node_id);
    if (target_nid == NUMA_NO_NODE)
        return false;  /* より低速な階層がない */

    return migrate_pages_to_node(demote_pages, target_nid, MIGRATE_ASYNC) == 0;
}
```

```bash
# page の降格を監視する
cat /proc/vmstat | grep pgdemote
# pgdemote_kswapd   12345   ← kswapd によって降格した page
# pgdemote_direct   678     ← 直接回収中に降格した page
```

## pageの昇格 {#page-promotion}

低速メモリ上のpageが頻繁にアクセスされると、NUMA Balancingや階層マイグレーションデーモンがそれを高速メモリへ昇格させることがある。

```c
/* mm/numa_balancing.c */
static int numa_migrate_preferred(struct task_struct *p)
{
    unsigned long interval = msecs_to_jiffies(task_scan_period(p));
    /* このタスクが使うページを、タスクの優先NUMAノードへ移動する */
    /* 優先ノードがより高速な階層にあれば、これは昇格になる */
}
```

### memory_tier sysfsによる明示的な昇格 {#explicit-promotion-via-memory_tier-sysfs}

```bash
# page の NUMA ノード移動ポリシーを設定する
numactl --preferred=0 myprogram  # DRAM（ノード 0）を優先する

# プロセスのすべての page を DRAM へ移動する
numactl --membind=0 --pid=1234 migrate

# 現在の page 配置を確認する
numastat -p <pid>
# Per-node memory use of process:
# N0      N1      N2      N3
# 512.0   128.0   0.0     1024.0  (MB)
```

## 階層化されたメモリ割り当て {#tiered-memory-allocation}

カーネルはまず高速な階層から割り当て、足りなければ低速な階層へフォールバックする。

```c
/* mm/page_alloc.c */
static struct page *get_page_from_freelist(gfp_t gfp_mask, unsigned int order,
                                            int alloc_flags,
                                            const struct alloc_context *ac)
{
    struct zoneref *z;
    struct zone *zone;

    /* zonelistは順序付けられている: 高速なメモリのゾーンが先に並ぶ */
    for_next_zone_zonelist_nodemask(zone, z, ac->zonelist, ac->highest_zoneidx,
                                    ac->nodemask) {
        /* このゾーンからの割り当てを試みる */
        page = rmqueue(ac->preferred_zoneref->zone, zone, order,
                        gfp_mask, alloc_flags, ac->migratetype);
        if (page)
            return page;
    }
    return NULL;
}
```

zonelistの順序は、NUMA距離を考慮する `build_zonelists()` によって設定される。メモリ階層がある場合、距離300のCXLメモリは距離200のDRAMの後に並ぶ。

## CXL（Compute Express Link）メモリ {#cxl-compute-express-link-memory}

CXLはPCIeベースのメモリ拡張規格であり、オンパッケージのDDRより高いレイテンシでDRAMを接続できる。

```bash
# CXL デバイスを確認する
ls /sys/bus/cxl/devices/
# mem0  mem1  (CXL メモリデバイス)

# CXL メモリリージョン
cat /sys/bus/cxl/devices/mem0/size

# CXL メモリを NUMA ノードに結び付ける
# （ファームウェア/BIOS が行い、NUMA ノード N として公開される）
cat /sys/devices/system/node/node2/memory_tier
# memory_tier2    ← CXL は階層 2 に割り当てられている
```

## PMEM（Persistent Memory） {#pmem-persistent-memory}

Optane DIMMや将来の永続メモリデバイス:

```bash
# ndctl: NVDIMM の制御
ndctl list
# [{
#   "dev":"namespace0.0",
#   "mode":"fsdax",
#   "size":549755813888,
#   "sector_size":512
# }]

# PMEM を DAX（直接アクセス、ページキャッシュなし）としてマウントする
mount -o dax /dev/pmem0 /mnt/pmem

# PMEM をメモリ階層として使う（揮発性、永続性なし）
# カーネルブートパラメータで設定する:
# memmap=4G!4G  ← 4G から始まる 4G を PMEM として使う（DRAM でエミュレート）
```

## NUMAマイグレーションポリシー {#numa-migration-policies}

```bash
# システム全体の NUMA マイグレーションポリシー
cat /proc/sys/kernel/numa_balancing
# 1 = 有効

# プロセスごとのマイグレーション制御
cat /proc/<pid>/numa_maps | head -5
# 7f1234560000 default anon=4 dirty=4 N0=4 kernelpagesize_kB=4
#                                     ^^^ すべての page が NUMA ノード 0 上にある

# page を特定のノードに強制する
numactl --membind=2 ./myapp  # ノード 2（CXL）から割り当てる

# page の NUMA 配置を確認する
cat /proc/<pid>/numa_maps | awk '{print $2}' | sort | uniq -c
```

## 昇格のためのホットなpageの検出 {#hot-page-detection-for-promotion}

カーネルは、PTEのaccessedビットとページフォールトのサンプリングを使って、低速な階層上のホットなpageを識別する。

```c
/* mm/numa_balancing.c — NUMA balancingのフォールトハンドラ */
static int do_numa_page(struct vm_fault *vmf)
{
    struct page *page = vmf->page;
    int current_nid = page_to_nid(page);
    int target_nid  = numa_migrate_prep(page, vmf, vmf->address,
                                         current_nid, &flags);

    if (target_nid != current_nid) {
        /* 移動先がより高速な階層にある: このページを昇格させる */
        migrate_misplaced_page(page, vmf, target_nid);
    }
    return 0;
}
```

## メモリ階層化の観測 {#observing-memory-tiering}

```bash
# メモリ階層の統計
cat /sys/kernel/debug/memory_tiering/

# 昇格/降格の頻度
watch -n 1 'grep -E "pgpromote|pgdemote" /proc/vmstat'

# 階層ごとの割り当て統計
cat /sys/devices/system/node/node*/vmstat | grep -E "nr_free|nr_active|nr_inactive"

# numastat: ノードごとの割り当て統計
numastat
# Per-node process memory usage:
#                     Node 0    Node 2
#            Huge         0         0
#          Heap       1234.5   456.7
#           Stack        2.1      0.0

# CXL の帯域幅監視
perf stat -e uncore_imc/data_reads/,uncore_imc/data_writes/ -a sleep 5
```

## 参考文献 {#further-reading}

- [NUMA](numa.md) — NUMAトポロジーの基礎
- [CXLメモリ階層化](cxl-memory-tiering.md) — CXL固有の詳細
- [ページ回収](reclaim.md) — 降格が回収とどう統合されているか
- [MGLRU](mglru.md) — 世代ベースのLRU。コールドなpageの識別に重要
- カーネルツリーの `mm/memory-tiers.c` — 階層管理
- カーネルツリーの `Documentation/admin-guide/mm/memory-tiers.rst`
