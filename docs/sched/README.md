# スケジューラ {#scheduler}

> Linuxが次に実行するタスクをどう決めるか

Linuxスケジューラは、任意の時点で各CPU上でどのタスクを実行するかを決める役割を担う。スケジューラは相反する要求のバランスを取る。すなわち、タスク間の公平性、対話的な処理の低レイテンシ、バッチジョブの高スループット、そしてリアルタイムプロセスの決定性である。

## このセクションの構成 {#structure-of-this-section}

### 基礎 {#fundamentals}
- [スケジューラの進化](scheduler-evolution.md) — O(1)からCFS、そしてEEVDFへ
- [スケジューラクラス](scheduler-classes.md) — 5つのスケジューリングポリシーとその階層構造
- [CFS: Completely Fair Scheduler](cfs.md) — vruntime、赤黒木、重み付き公平性
- [EEVDFスケジューラ](eevdf.md) — 仮想デッドライン、適格性（eligibility）、そしてCFSの置き換え
- [sched_ext（BPFスケジューリング）](sched-ext.md) — 6番目のスケジューラクラス。BPFでプログラム可能なポリシーで、6.12でマージされた
- [ランキューとタスク選択](runqueues.md) — CPUごとのランキューと、タスクがどう選ばれるか

### ライフサイクル {#lifecycle}
- [コンテキストスイッチの一生](context-switch.md) — `__schedule()`の内部で何が起きるか
- [プロセスが起床するとき何が起きるか](wakeup.md) — 起床パスとスケジューラによる配置
- [fork()すると何が起きるか](sched-fork.md) — 新しいタスクがスケジューラに入る仕組み

### リアルタイム {#real-time}
- [RTスケジューラ](rt-scheduler.md) — SCHED_FIFO、SCHED_RR、RTランキュー
- [SCHED_DEADLINE](deadline.md) — CBSと受け入れ制御（admission control）
- [優先度逆転とPI mutex](pi-mutexes.md) — 問題とその解決策

### リソース制御 {#resource-control}
- [CPU cgroup v1とv2の違い](cpu-cgroup.md) — shares、quota、そして階層の違い
- [CPU帯域制御](cpu-bandwidth.md) — CFS帯域スロットリング
- [cpuset](cpuset.md) — cgroupによるCPUとNUMAノードのアフィニティ

### トポロジ {#topology}
- [スケジューラドメインと負荷分散](sched-domains.md) — SMT、LLC、NUMAの階層
- [CPUアフィニティとピン留め](cpu-affinity.md) — taskset、sched_setaffinity、そしてそれらを使うべき場面

### デバッグ {#debugging}
- [/proc/schedstatを理解する](schedstat.md) — CPUごと・タスクごとのスケジューラ統計
- [スケジューラのトレーシング](sched-tracing.md) — ftrace、perf sched、スケジューライベント
- [レイテンシ対スループットのチューニング](sched-tuning.md) — sysctlのつまみとそのトレードオフ

### 事件簿 {#war-stories}
- [スケジューラの事件簿](war-stories.md) — 実際のCVE、性能リグレッション、そして未解決のヒューリスティックの緊張関係

## 主要なソースファイル {#key-source-files}

| ファイル | 役割 |
|------|-------------|
| [`kernel/sched/core.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/core.c) | `__schedule()`、`context_switch()`、`wake_up_process()` |
| [`kernel/sched/fair.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/fair.c) | CFSとEEVDFの実装 |
| [`kernel/sched/rt.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/rt.c) | RTスケジューラ |
| [`kernel/sched/deadline.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/deadline.c) | SCHED_DEADLINE |
| [`kernel/sched/sched.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/sched.h) | 中核のデータ構造: `struct rq`、`struct sched_class`、`struct cfs_rq` |
| [`include/linux/sched.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/sched.h) | `struct task_struct`、`struct sched_entity` |
