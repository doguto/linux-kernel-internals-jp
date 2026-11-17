# cpuset: CPUとメモリのピン留め {#cpuset-cpu-and-memory-pinning}

> cgroupが、タスクの使えるCPUとNUMAノードをどう制限するか

## cpusetが制御するもの {#what-cpuset-controls}

`cpuset`は、タスクのグループが使えるCPUとNUMAメモリノードを制限するcgroupサブシステムである。CPUの帯域や重みとは独立して動作する:

- **帯域**（cpu.max）: グループがどれだけのCPU時間を得るか
- **重み**（cpu.weight/shares）: CPU時間をグループ間でどう分けるか
- **cpuset**: そもそもどのCPUが対象になるか — 除外されたCPU上でタスクが実行されることは決してない

これが重要になるのは次のような場面である:

- **NUMA局所性**: タスクを、そのメモリに近いCPUにピン留めする
- **コアの隔離**: レイテンシに敏感なワークロードのために特定のコアを確保する
- **ハードウェアの分割**: 専用のコンテナにCPUを割り当てる

## インターフェース {#interface}

```bash
# v2（統合階層）
ls /sys/fs/cgroup/mygroup/
# cpuset.cpus              - 要求したCPUのリスト
# cpuset.mems              - 要求したNUMAノードのリスト
# cpuset.cpus.effective    - 親との積集合を取った後の実際のCPU
# cpuset.mems.effective    - 親との積集合を取った後の実際のNUMAノード
# cpuset.cpus.exclusive    - このcgroup専用に確保されたCPU
# cpuset.cpus.partition    - パーティションルートの制御（member/root/isolated）

# cgroupをCPU 0-3とNUMAノード0にピン留めする
echo "0-3" > /sys/fs/cgroup/mygroup/cpuset.cpus
echo "0"   > /sys/fs/cgroup/mygroup/cpuset.mems

# v1（別個のマウント）
echo "0-3" > /sys/fs/cgroup/cpuset/mygroup/cpus
echo "0"   > /sys/fs/cgroup/cpuset/mygroup/mems
```

## struct cpuset {#struct-cpuset}

各cgroupは、`kernel/cgroup/cpuset-internal.h`の`cpuset`に対応する:

```c
// kernel/cgroup/cpuset-internal.h
struct cpuset {
    struct cgroup_subsys_state css;  // 埋め込まれたcgroupの状態

    unsigned long flags;             // CS_CPU_EXCLUSIVE、CS_MEM_EXCLUSIVEなど

    // ユーザーが設定したマスク（制御ファイル経由で書き込まれる）
    cpumask_var_t cpus_allowed;
    nodemask_t    mems_allowed;

    // 実効マスク（タスクに適用される実際の制限）
    // effective = 設定値 & 親のeffective
    cpumask_var_t effective_cpus;
    nodemask_t    effective_mems;

    // 排他的なCPU（v2のパーティションサポート）
    cpumask_var_t effective_xcpus;  // 実際に与えられた排他的CPU
    cpumask_var_t exclusive_cpus;   // ユーザーが要求した排他的CPU

    int partition_root_state;       // PRS_MEMBER/ROOT/ISOLATED、負の値は無効
    int nr_deadline_tasks;          // このcpuset内のSCHED_DEADLINEタスク数

    // ...
};
```

### フラグ {#flags}

```c
typedef enum {
    CS_CPU_EXCLUSIVE,      // 兄弟cgroupはこれらのCPUと重複できない
    CS_MEM_EXCLUSIVE,      // 兄弟cgroupはこれらのNUMAノードと重複できない
    CS_MEM_HARDWALL,       // ユーザーの割り当ては他のノードを使えない（フォールバックとしても）
    CS_MEMORY_MIGRATE,     // mems_allowedが変わったときにページをマイグレーションする
    CS_SCHED_LOAD_BALANCE, // cpusetのCPU間で負荷分散を有効にする
    CS_SPREAD_PAGE,        // ページキャッシュを許可されたすべてのノードに分散させる
    CS_SPREAD_SLAB,        // slabキャッシュを許可されたすべてのノードに分散させる
} cpuset_flagbits_t;
```

## 実効マスク: 親との積集合 {#effective-masks-intersection-with-parent}

cpusetの重要な不変条件は、子の**effective_cpus**は親の**effective_cpus**の部分集合にしかなりえないということである。親の実効集合に含まれないCPUを含むCPUリストを書き込むと、それらのCPUは黙って除外される。

```
root cpuset: effective_cpus = {0,1,2,3,4,5,6,7}
    └── mygroup: cpus_allowed = {4,5,6,7,8,9}
                 effective_cpus = {4,5,6,7}  ← 親との積集合
```

```c
// kernel/cgroup/cpuset.c（簡略化）
static void cpuset_cpus_allowed(struct task_struct *p, struct cpumask *pmask)
{
    struct cpuset *cs = task_cs(p);
    // 空でない積集合が見つかるまで階層をたどって上がる
    while (!cpumask_intersects(cs->effective_cpus, pmask))
        cs = parent_cs(cs);
    cpumask_and(pmask, pmask, cs->effective_cpus);
}
```

## タスクがどう制約されるか {#how-tasks-are-constrained}

タスクがcgroupに追加されたとき（`cgroup.procs`）、またはcpusetのeffective_cpusが変化したとき、カーネルはタスクのアフィニティを更新する:

1. `cpuset_change_task_nodemask()` — NUMAノードのポリシーを更新する
2. `set_cpus_allowed_ptr()` — タスクを`effective_cpus`に制限する
3. タスクが現在、除外されたCPU上で実行中であれば、マイグレーションされる

cpuset内のタスクは（`sched_setaffinity()`を通じて）自分のアフィニティを設定することもできるが、その結果は常にcpusetのeffective_cpusとの**積集合**が取られる。アフィニティを通じてcpusetから逃れることはできない。

## パーティションルート（v2） {#partition-roots-v2}

v2のcpusetは**パーティションルート**をサポートする。これは親のスケジューリングドメインからCPUを完全に切り出す仕組みである:

```bash
# パーティションを作成する: CPU 4-7を排他的に取得する
echo "4-7" > /sys/fs/cgroup/mygroup/cpuset.cpus
echo "root" > /sys/fs/cgroup/mygroup/cpuset.cpus.partition
# CPU 4-7は親のスケジューリングドメインから取り除かれ、
# mygroup用の隔離されたドメインを形成する

# 隔離パーティション: パーティション内でも負荷分散を行わない
echo "isolated" > /sys/fs/cgroup/mygroup/cpuset.cpus.partition
```

パーティションの状態:

- `member`（デフォルト）: 通常のcpusetで、親のスケジューリングドメインを共有する
- `root`: CPUを専用のスケジューリングドメインに切り出す
- `isolated`: rootと同様だが、パーティション内の負荷分散も無効にする

## cpusetとスケジューラドメイン {#cpuset-and-scheduler-domains}

パーティションルートが作成されたり、CPUが排他的に割り当てられたりすると、カーネルは**スケジューリングドメインを再構築する**:

```c
// kernel/cgroup/cpuset.c
// effective_cpusが変化した後:
rebuild_sched_domains();  // または rebuild_sched_domains_locked()
```

これはコストが高い。再構築のたびに、CPUごとのスケジューラドメインの階層を解体して作り直すからである。本番環境では、cpusetのCPUを頻繁に変更するのは避けること。

## NUMA: mems_allowed {#numa-mems_allowed}

`effective_mems`は、メモリ割り当てにどのNUMAノードを使えるかを制御する:

```bash
# ノード0のみに制限する
echo "0" > /sys/fs/cgroup/mygroup/cpuset.mems

# ノード0と1を許可する
echo "0-1" > /sys/fs/cgroup/mygroup/cpuset.mems
```

`CS_MEM_HARDWALL`が設定されていると、許可されたノードが満杯でも、タスクは他のノードにフォールバックできない。設定されていなければ、メモリ圧迫下でカーネルが他のノードから割り当てることがある。

`CS_MEMORY_MIGRATE`が設定されていると、`mems_allowed`が変化したときに既存のページが新しいノード集合へマイグレーションされる。

## cpusetの制約を診断する {#diagnosing-cpuset-constraints}

```bash
# タスクのcpusetを確認する
cat /proc/$PID/cpuset
# → /mygroup/container1

# cgroupの実効CPU
cat /sys/fs/cgroup/mygroup/cpuset.cpus.effective

# タスクのアフィニティがcpusetによって制約されているか確認する
taskset -p $PID  # ユーザーのアフィニティとcpusetの積集合を表示する

# cpusetのイベント（CPUホットプラグ、パーティションの変更）を監視する
cat /sys/kernel/debug/tracing/events/cpuset/
```

```bash
# よくある間違い: 親の実効集合にないCPUを書き込む
echo "0-7" > /sys/fs/cgroup/nested/cpuset.cpus
cat /sys/fs/cgroup/nested/cpuset.cpus.effective
# → 0-3   （黙って親の範囲に切り詰められる）
```

## cpusetとsched_setaffinity {#cpuset-vs-sched_setaffinity}

| 仕組み | 誰が制御するか | 範囲 | 継承されるか |
|-----------|-------------|-------|-------------|
| cpuset | cgroupの管理者 | cgroup内のすべてのタスク | Yes — 子が継承する |
| sched_setaffinity | プロセス（他のプロセスに対してはCAP_SYS_NICE） | 1つのタスク | No |

実効アフィニティは常に`user_affinity ∩ cpuset.effective_cpus`である。どちらか一方でもあるCPUを制限していれば、タスクはそこで実行できない。

## 参考文献 {#further-reading}

- [CPU cgroup: v1とv2](cpu-cgroup.md) — CPUの帯域と重みの制御
- [スケジューリングドメイン](sched-domains.md) — cpusetのパーティションルートが負荷分散にどう影響するか
- [CPUアフィニティ](cpu-affinity.md) — sched_setaffinityによるタスクごとのCPU制限
- [NUMA](../mm/numa.md) — NUMAノードへのピン留めが性能にとって重要な理由
