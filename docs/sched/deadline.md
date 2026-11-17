# SCHED_DEADLINE {#sched_deadline}

> 受け入れ制御を備えた、帯域ベースのリアルタイムスケジューリング

## SCHED_DEADLINEが提供するもの {#what-sched_deadline-provides}

`SCHED_DEADLINE`は、Linuxで最も優先度の高い通常のスケジューリングポリシーである（SCHED_FIFO/RRより上）。固定の優先度の代わりに、各タスクは自身の計算上の要件を記述する:

- **実行時間**（`sched_runtime`）: 周期ごとにどれだけのCPU時間が必要か
- **デッドライン**（`sched_deadline`）: いつまでに終えなければならないか
- **周期**（`sched_period`）: どれくらいの頻度で繰り返すか（未設定ならデッドラインと同じ）

カーネルは**Constant Bandwidth Server**（CBS）を使ってこれらの範囲を強制する。各タスクは宣言したとおりの帯域をちょうど得て、それ以上は得られない。タスクが実行時間の予算を使い切ると、次の周期までスロットリングされる。

```
Task declares: 5ms runtime, 10ms deadline, 10ms period
= 50% CPU utilization

Timeline:
|----5ms---|---throttled---|----5ms---|---throttled---|
0         5              10         15              20ms
          ↑ budget exhausted          ↑ replenished
          deadline must be met ←------↑
```

## なぜSCHED_FIFOではなくSCHED_DEADLINEなのか {#why-sched_deadline-over-sched_fifo}

| 性質 | SCHED_FIFO | SCHED_DEADLINE |
|----------|------------|----------------|
| 保証の種類 | 優先度（相対的） | 帯域（絶対的） |
| 受け入れ制御 | なし | あり — カーネルは実現不可能なタスクを拒否する |
| 最悪ケースのレイテンシ | 同じ優先度のタスクについては上限なし | デッドラインで上限が決まる |
| 超過時の挙動 | 飢餓 | スロットリングされ、次の周期で補充される |
| 用途 | 単純なRTタスク | タイミング要件のある周期的タスク |

SCHED_DEADLINEは、タイミング要件を証明できるタスクに適している。例えば、ビデオエンコードのフレーム処理、レート保証付きのネットワークパケット処理、産業用の制御ループなどである。

## データ構造 {#data-structures}

### struct sched_dl_entity {#struct-sched_dl_entity}

`task_struct`に埋め込まれ、タスクごとのデッドラインの状態を追跡する:

```c
// include/linux/sched.h
struct sched_dl_entity {
    struct rb_node  rb_node;        // dl_rqの赤黒木内の位置
    struct hrtimer  dl_timer;       // CBSの補充タイマー

    u64 dl_runtime;     // 宣言した実行時間（r_i）
    u64 dl_deadline;    // 宣言した相対デッドライン（d_i）
    u64 dl_period;      // 宣言した周期（p_i）。0は = dl_deadline を意味する
    u64 dl_bw;          // 帯域 = dl_runtime / dl_period
    u64 dl_density;     // 密度 = dl_runtime / dl_deadline

    s64 runtime;        // 現在のインスタンスの残り実行時間
    u64 deadline;       // 現在のインスタンスの絶対デッドライン

    unsigned int dl_throttled : 1;  // この周期の予算を使い切った
    // ...
    struct hrtimer  inactive_timer; // GRUBの非アクティブ時間の追跡
};
```

### struct dl_rq {#struct-dl_rq}

CPUごとのデッドラインランキュー:

```c
// kernel/sched/sched.h
struct dl_rq {
    struct rb_root_cached root;              // 絶対デッドライン順の赤黒木
    unsigned int          dl_nr_running;     // 実行可能なDLタスクの数
    struct {
        u64 curr, next;
    } earliest_dl;                           // キャッシュされた最も早いデッドライン

    u64 running_bw;   // 現在実行中のタスクの帯域
    u64 this_bw;      // 割り当てられた帯域の合計（実行中 + ブロック中）
    u64 extra_bw;     // 回収したアイドル帯域（GRUB）
    u64 max_bw;       // 回収可能な最大帯域
    u64 bw_ratio;     // GRUBの計算用の1/Umax
};
```

## 受け入れ制御 {#admission-control}

タスクが`SCHED_DEADLINE`になる前に、カーネルはシステムがそれを受け入れられるかどうかをチェックする:

```c
// kernel/sched/deadline.c
static int sched_dl_overflow(struct task_struct *p, int policy,
                              const struct sched_attr *attr)
{
    // チェック: すべてのDL帯域の合計 + 新しいタスク <= 1.0（CPUあたり）
    // 容量を超える場合: -EBUSYを返す
}
```

このチェックは、スケジューリングドメイン内のすべてのCPUにわたるグローバルなものである。新しいタスクを受け入れると合計の使用率が100%を超える場合、`sched_setattr()`は`-EBUSY`を返す:

```c
struct sched_attr attr = {
    .size           = sizeof(attr),
    .sched_policy   = SCHED_DEADLINE,
    .sched_runtime  = 5000000,   // 5ms
    .sched_deadline = 10000000,  // 10ms
    .sched_period   = 10000000,  // 10ms
};
if (sched_setattr(0, &attr, 0) < 0) {
    if (errno == EBUSY)
        // システムはこのタスクを受け入れられない — 使用中のDL帯域が多すぎる
}
```

これは、システムの容量にかかわらずいくつでもタスクを作成できるSCHED_FIFO/RRとは異なる。

## CBS: 予算の強制と補充 {#cbs-budget-enforcement-and-replenishment}

### 予算の枯渇 {#budget-exhaustion}

タスクが実行されるたびに、残りの`runtime`は減少する。それがゼロになると:

1. タスクに`dl_throttled = 1`が設定される
2. タスクがdl_rqからデキューされる
3. 次の補充時刻に向けてhrtimer（`dl_timer`）が設定される
4. 補充されるまで、タスクはもう実行できない

### dl_task_timer()による補充 {#replenishment-via-dl_task_timer}

```c
// kernel/sched/deadline.c
static enum hrtimer_restart dl_task_timer(struct hrtimer *timer)
{
    struct sched_dl_entity *dl_se = container_of(timer, ...);
    struct task_struct *p = dl_task_of(dl_se);

    // 次の周期のために予算を補充する
    replenish_dl_entity(dl_se);

    // タスクを再エンキューする
    enqueue_task(rq, p, ENQUEUE_REPLENISH);

    // プリエンプションをチェックする
    wakeup_preempt(rq, p, 0);
}
```

`replenish_dl_entity()`は次のように設定する:

- `dl_se->runtime = dl_se->dl_runtime`（予算を満タンに戻す）
- `dl_se->deadline += dl_se->dl_period`（絶対デッドラインを進める）

### EDFによる選択 {#edf-selection}

実行可能なDLタスクが複数ある場合、スケジューラは常に**絶対デッドラインが最も早い**ものを選ぶ（EDF — Earliest Deadline First）:

```c
// kernel/sched/deadline.c
static struct sched_dl_entity *pick_next_dl_entity(struct dl_rq *dl_rq)
{
    struct rb_node *left = rb_first_cached(&dl_rq->root);
    return rb_entry(left, struct sched_dl_entity, rb_node);
}
```

赤黒木は`deadline`順に並んでいるので、最左ノードが常に最も早いデッドラインを持つ。

## GRUB: 使われていない帯域の回収 {#grub-reclaiming-unused-bandwidth}

予算を使い切らないDLタスクはCPU時間を無駄にしてしまう。予約された帯域を他のタスクが使えないからである。**GRUB**（Greedy Reclamation of Unused Bandwidth）がこれを解決する。

DLタスクが非アクティブ（スリープ中）のとき、その使われていない帯域は追跡され、`extra_bw`を通じて他のタスクが利用できるようになる。これにより、タスクがアクティブなときにはすべてのデッドラインを守りつつ、厳密な帯域予約から想定されるより多くの仕事をシステムで実行できる。

```bash
# CPUごとのDL帯域の合計を確認する
cat /proc/sched_debug | grep -A 5 "dl_rq"
```

## SCHED_DEADLINEを使う {#using-sched_deadline}

### ポリシーの設定 {#setting-the-policy}

```c
#include <linux/sched/types.h>

struct sched_attr attr = {
    .size           = sizeof(struct sched_attr),
    .sched_policy   = SCHED_DEADLINE,
    .sched_flags    = 0,
    .sched_runtime  =  5000000,  // ナノ秒単位で5ms
    .sched_deadline = 10000000,  // 10ms
    .sched_period   = 10000000,  // 10ms（= 使用率50%）
};

if (sched_setattr(0, &attr, 0) < 0)
    perror("sched_setattr");
```

`CAP_SYS_NICE`が必要である。

### SCHED_DEADLINEでのsched_yield() {#sched_yield-for-sched_deadline}

DLタスクは、仕事を早く終えたときに`sched_yield()`を呼び出すべきである。これにより、タスクがこの周期の仕事を終えたこと、そして残りの予算を回収できることがCBSに伝わる:

```c
while (1) {
    do_periodic_work();  // dl_runtimeより前に終わる
    sched_yield();       // 残りの予算を解放し、次の周期までスリープする
}
```

`sched_yield()`を呼ばないと、タスクは予算を使い切るまで実行され、CPUサイクルを無駄にする。

### 制約 {#constraints}

- `SCHED_DEADLINE`タスクからの`fork()`は成功するが、子は`SCHED_NORMAL`に降格される
- DLタスク内での`pthread_create()`は動作するが、新しいスレッドは`SCHED_NORMAL`を継承する
- DLタスクは、CPU制限のあるcgroupに所属できない

```bash
# 現在のDLタスクを確認する
chrt -p $PID  # SCHED_DEADLINEとパラメータを表示する

# システムのDL帯域の使用量を表示する
cat /proc/sched_debug | grep running_bw
```

## RTタスクとの相互作用 {#interaction-with-rt-tasks}

クラス階層において、`dl_sched_class`は`rt_sched_class`の上に位置する。SCHED_DEADLINEタスクは、RT優先度にかかわらず、常にSCHED_FIFO/RRタスクをプリエンプトする。

DLタスクが必要とするmutexをRTタスクが保持している場合、優先度継承によってRTタスクの優先度がブーストされる。SCHED_DEADLINEも`is_dl_boosted()`によるブーストをサポートしている。DLタスクがブーストされると、その実効デッドラインはドナータスクから導かれる。

## 参考文献 {#further-reading}

- [RTスケジューラ](rt-scheduler.md) — SCHED_FIFO/RR、DLの下にあるクラス
- [優先度逆転とPI mutex](pi-mutexes.md) — DLがロックとどう相互作用するか
- [スケジューラクラス](scheduler-classes.md) — 階層の中でのDLの位置
- [kernel/sched/deadline.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/deadline.c) — CBSの完全な実装
