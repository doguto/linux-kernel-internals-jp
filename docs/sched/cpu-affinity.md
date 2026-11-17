# CPUアフィニティ {#cpu-affinity}

> タスクごとのCPUピン留め: sched_setaffinity()と、それがcpusetとどう相互作用するか

## CPUアフィニティが制御するもの {#what-cpu-affinity-controls}

CPUアフィニティを使うと、タスク自身（または管理者）が、特定のタスクがどのCPU上で実行してよいかを制限できる。cgroup全体に適用されるcpusetとは異なり、アフィニティは**タスクごと**である。

一般的な用途:

- **キャッシュ局所性**: L3キャッシュを共有するCPUにタスクをピン留めする
- **割り込みとの結び付け**: タスクを、それが処理するNIC割り込みと同じCPU上に保つ
- **ベンチマーク**: 1つのコアにピン留めして、CPU間のばらつきを排除する
- **リアルタイム**: レイテンシに敏感なタスクのためにコアを確保する

## システムコール {#the-syscall}

```c
// タスクを特定のCPUに制限する
int sched_setaffinity(pid_t pid, size_t cpusetsize, const cpu_set_t *mask);

// タスクの現在のアフィニティマスクを読み取る
int sched_getaffinity(pid_t pid, size_t cpusetsize, cpu_set_t *mask);
```

コマンドラインからは次のようにする:

```bash
# CPU 0と1にピン留めしてコマンドを実行する
taskset -c 0,1 my_program

# 既存のプロセスをピン留めする
taskset -cp 0,1 $PID

# 現在のアフィニティを表示する
taskset -p $PID
# → pid 1234's current affinity mask: f  (4コアのシステムでCPU 0-3)
```

## カーネルの実装 {#kernel-implementation}

### システムコールのパス {#the-syscall-path}

```c
// kernel/sched/syscalls.c:1256
SYSCALL_DEFINE3(sched_setaffinity, pid_t, pid, unsigned int, len,
                unsigned long __user *, user_mask_ptr)
{
    cpumask_var_t new_mask;
    get_user_cpu_mask(user_mask_ptr, len, new_mask);  // ユーザー空間からコピー
    return sched_setaffinity(pid, new_mask);
}
```

`sched_setaffinity()`はマスクを検証し、権限をチェックしてから`__sched_setaffinity()`を呼び出し、それが`set_cpus_allowed_ptr()`を呼び出す。

### task_structのアフィニティ関連フィールド {#affinity-fields-in-task_struct}

```c
// include/linux/sched.h
struct task_struct {
    // ...
    int            nr_cpus_allowed;  // cpus_ptrのpopcount
    const cpumask_t *cpus_ptr;       // 実効アフィニティ（cpus_maskを指すか、
                                     // マイグレーション中は一時的なマスクを指す）
    cpumask_t      *user_cpus_ptr;   // ユーザーが設定したアフィニティ（未設定ならNULL）
    cpumask_t       cpus_mask;       // cpus_ptrの格納先（通常）
    // ...
};
```

- `cpus_ptr`はスケジューラが読むものである。通常は`cpus_mask`を指す。
- 一時的なCPU制限の間（例: マイグレーションのヘルパー）、`cpus_ptr`は別の場所を指すことがある。
- `user_cpus_ptr`は、cpusetによって課される制限とは別に、ユーザーが要求したマスクを追跡する。

### アフィニティの読み戻し {#reading-back-affinity}

```c
// kernel/sched/syscalls.c:1275
long sched_getaffinity(pid_t pid, struct cpumask *mask)
{
    // ...
    cpumask_and(mask, &p->cpus_mask, cpu_active_mask);
    // 注: アクティブなCPUとの積集合を取り、オフライン/ホットプラグで外されたCPUを除外する
}
```

返されるマスクは常に`cpu_active_mask`との積集合が取られる。オフラインのCPUは、たとえ`cpus_mask`に含まれていても除外される。

## アフィニティとcpusetの積集合 {#affinity-and-cpuset-intersection}

タスクがcpusetに属している場合、実効アフィニティは次のとおりである:

```
effective = user_affinity ∩ cpuset.effective_cpus
```

cpusetの外にあるCPUを含むマスクで`sched_setaffinity()`を呼び出すと、それらのCPUは黙って除外される。タスクはアフィニティを通じてcpusetから逃れることはできない。

```bash
# 例: CPU 4-7に制限されたcpuset内のタスク
taskset -cp 0-7 $PID      # すべてのCPUを使おうとする
taskset -p $PID            # 実効: 4-7のみ（cpusetとの積集合）
```

カーネルは両方のマスクを別々に保持する:

- `user_cpus_ptr`: ユーザーが要求したもの
- `cpus_mask`: cpusetとの積集合（実際に強制されるもの）

この分離により、cpusetが変化したときに、ユーザーの元の意図を失うことなく、カーネルは実効マスクを再計算できる。

## マイグレーション: アフィニティが変わると何が起きるか {#migration-what-happens-when-affinity-changes}

`set_cpus_allowed_ptr()`が呼び出されると:

1. タスクの現在のCPUがまだ新しいマスクに含まれている場合 → すぐには何も変わらない
2. 現在のCPUが除外された場合 → タスクは許可されたCPUへマイグレーションされる
3. マイグレーションは`migration_cpu_stop()`を通じて行われる。これはタスクを安全に移動させるためにターゲットCPU上で実行されるstop-machineの仕組みである

```bash
# アフィニティによって引き起こされたマイグレーションを観察する
perf stat -e migrations ./my_program
```

## カーネルスレッドとアフィニティ {#kernel-threads-and-affinity}

カーネルスレッドは、すべてのオンラインCPU（`cpu_all_mask`）に設定されたアフィニティで開始する。これを制限することもできる:

```bash
# カーネルのワーカースレッド（例: kworker）をピン留めする
taskset -cp 0 $(pgrep "kworker/0:1")
```

一部のカーネルスレッドは自分のアフィニティを明示的に設定しており（例: 割り込みスレッド、CPUごとのkthread）、外部からの変更を受け付けない。`/proc/PID/status`の`Cpus_allowed`を確認すること。

## NUMAを意識したアフィニティ {#numa-aware-affinity}

NUMAシステムでは、特定のCPUへのピン留めはメモリ割り当てにも影響する:

```bash
# NUMAノード1のCPUにピン留めする（CPU 4-7がノード1にあると仮定）
taskset -c 4-7 ./my_program

# より良い方法: numactlを使ってCPUとメモリの両方のアフィニティを設定する
numactl --cpunodebind=1 --membind=1 ./my_program
```

`numactl`はCPUアフィニティ（`sched_setaffinity`）とNUMAメモリポリシー（`set_mempolicy`）を組み合わせる。NUMAに敏感なワークロードでは、両方を設定することが不可欠である。メモリをピン留めせずにCPUだけをピン留めしても、依然としてリモートメモリアクセスが発生しうる。

## /procでアフィニティを確認する {#checking-affinity-in-proc}

```bash
# 16進ビットマスクとしてのCPUアフィニティ
cat /proc/$PID/status | grep Cpus_allowed
# → Cpus_allowed: 000f  (CPU 0-3)
# → Cpus_allowed_list: 0-3  (人間が読める形式)

# プロセス内のスレッドについて
for tid in /proc/$PID/task/*/; do
    echo -n "TID $(basename $tid): "
    cat ${tid}/status | grep Cpus_allowed_list
done
```

## アフィニティ対cpuset対スケジューリングドメイン {#affinity-vs-cpuset-vs-scheduling-domains}

| 仕組み | 粒度 | 範囲 | 厳格な制限か？ |
|-----------|-------------|-------|-------------|
| `sched_setaffinity` | タスクごと | 選択したCPU | Yes — タスクは他の場所では決して実行されない |
| cpuset | cgroupごと | CPUのグループ | Yes — すべてのタスクと積集合が取られる |
| スケジューリングドメイン | グローバル | トポロジのレベル | No — 負荷分散の優先度に影響する |
| `nice`/優先度 | タスクごと | CPU時間の配分 | No — スケジューリングの優先度に影響するだけ |

## 参考文献 {#further-reading}

- [cpuset](cpuset.md) — グループ単位のCPU制限。タスクごとのアフィニティと積集合が取られる
- [スケジューリングドメイン](sched-domains.md) — 負荷分散がトポロジをどう考慮するか
- [CPU帯域制御](cpu-bandwidth.md) — cgroupのクォータによる厳格なCPU時間の制限
- [NUMA](../mm/numa.md) — CPUアフィニティと並んでNUMAノードのアフィニティが重要な理由
