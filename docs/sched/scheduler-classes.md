# スケジューラクラス {#scheduler-classes}

> Linuxが5つのスケジューリングポリシーをひとつのフレームワークに階層化する仕組み

## なぜvtableの階層なのか？ {#why-a-vtable-hierarchy}

### sched_class以前: ひとつの一枚岩のスケジューラ {#before-sched_class-one-monolithic-scheduler}

Linux 2.4のスケジューラと初期2.6のO(1)スケジューラ（Ingo Molnár、2003年）は、リアルタイムタスクと通常タスクを、条件分岐を持つひとつの関数で扱っていた: タスクがリアルタイム優先度を持っていれば優先度でインデックスされたビットマップに入れ、そうでなければnice値と最近のスリープ/実行の履歴から動的な優先度を計算する。

**O(1)スケジューラの発表**: [LWN, January 2002](https://lwn.net/2002/0110/a/scheduler.php3) — Ingo Molnár、Linux 2.6.0（2003年12月）で出荷。

これは動作したが、根本的に異なるスケジューリングアルゴリズムをひとつの塊に結合していた。SCHED_FIFOとSCHED_RRのルールは単純である: ブロックするか、より高優先度のRTタスクにプリエンプトされるまで実行する。通常タスクには複雑な公平性のヒューリスティクスが必要である。スケジューラが両方を扱おうとすればするほど、どちらについても推論するのが難しくなった。

### CFSとvtable設計（Linux 2.6.23、2007年） {#cfs-and-the-vtable-design-linux-2623-2007}

Ingo MolnárがO(1)スケジューラをCFSで置き換えたとき、彼は抽象化レイヤーとして`struct sched_class`も導入した。動機は明確だった: CFSは仮想実行時間順の赤黒木を使う — RTタスクにとっては意味をなさないデータ構造である。RTタスクは優先度ビットマップを使う — 公平スケジューリングには無関係なデータ構造である。2つのアルゴリズムには、異なるデータ構造、異なるエンキュー/デキューのロジック、異なる次タスク選択のセマンティクスが必要だった。

**CFSの発表**: [LKML, April 13 2007](https://lkml.org/lkml/2007/4/13/180) — Ingo Molnár, "[Announce] [patch] Modular Scheduler Core and Completely Fair Scheduler [CFS]"

**CFSのマージコミット**（2007年7月9日、Ingo Molnár）:

- [`bf0f6f24a1ec`](https://git.kernel.org/linus/bf0f6f24a1ece8988b243aefe84ee613099a9245) — `sched: cfs core, kernel/sched_fair.c`
- [`dd41f596cda0`](https://git.kernel.org/linus/dd41f596cda0d7d6e4a8b139ffdfabcefdd46528) — `sched: cfs core code`（`struct sched_class`をコアスケジューラに組み込む）

スケジューラ全体に`if (rt_task(p)) { ... } else { ... }`を織り込む代わりに、vtableによって各アルゴリズムがそれぞれ独立したきれいな実装を持てるようになる。コアスケジューラのループは単純になる: 最高優先度から最低優先度へとクラスを順にたどり、それぞれに`pick_next_task()`を問い合わせ、最初の非NULLの結果を返す。

この設計はすぐに成果をもたらした。`SCHED_DEADLINE`（帯域の計上を伴うEDFスケジューリング、3.14でマージ）は、`sched_class`インターフェースを実装することで新しいクラスとして追加された。vtableがなければ、既存のコードに3つ目のスケジューリングアルゴリズムを追加するのは侵襲的な作業になっていただろう。vtableのおかげで、`kernel/sched/deadline.c`は自己完結している。

**SCHED_DEADLINEのマージ**（[patch series v9, LKML Nov 2013](https://lore.kernel.org/all/1360608656-7969-1-git-send-email-juri.lelli@gmail.com/) — Juri Lelli、Dario Faggioliら）: [`aab03e05e8f7`](https://git.kernel.org/linus/aab03e05e8f7e26f51dee792beddcb5cca9215a5) — `sched/deadline: Add SCHED_DEADLINE structures & implementation`、v3.14-rc1（2014年1月）にマージされた。

`stop`クラス（CPUマイグレーション用スレッドのためのもの）は別の利点を示している: このクラスは`SCHED_FIFO`の優先度99のタスクを含む*すべて*をプリエンプトする必要がある。クラス階層があれば、これは`stop`をたどる順序の先頭に置くだけで済む。コアスケジューラに特別扱いのコードはない。

**stopクラスのコミット**: [`34f971f6f798`](https://git.kernel.org/linus/34f971f6f7988be4d014eec3e3526bee6d007ffa) — `sched: Create special class for stop/migrate work`（Peter Zijlstra、v2.6.37にマージ、2010年10月）。

### 高速パスの最適化 {#the-fast-path-optimization}

性能のための唯一の妥協点は、すべてのタスクが`fair`クラスにある場合（デスクトップ/サーバーシステムでの一般的なケース）に`for_each_class()`ループを省略することである:

```c
if (likely(!sched_class_above(prev->sched_class, &fair_sched_class) &&
           rq->nr_running == rq->cfs.h_nr_running))
    return pick_next_task_fair(rq, prev, rf);
```

RTタスクもデッドラインタスクもないシステムでは、NULLを返すだけのvtable関数を3つ呼ぶオーバーヘッドがなくなる。

## 階層モデル {#the-layered-model}

Linuxのスケジューラはひとつではない — 5つあり、それぞれが異なるクラスのタスクを扱う。これらは優先度順に積み重ねられている: カーネルが次に実行するタスクを選ぶとき、最高から最低へと各クラスに順に問い合わせ、準備のできたタスクを持つクラスが見つかるまで続ける。

```
┌─────────────────────────────────────────┐
│  stop     (migration, hotplug)          │  ← highest priority
├─────────────────────────────────────────┤
│  deadline (SCHED_DEADLINE)              │
├─────────────────────────────────────────┤
│  rt       (SCHED_FIFO, SCHED_RR)        │
├─────────────────────────────────────────┤
│  fair     (SCHED_NORMAL, SCHED_BATCH,   │
│            SCHED_IDLE)                  │
├─────────────────────────────────────────┤
│  idle     (swapper/idle thread)         │  ← lowest priority
└─────────────────────────────────────────┘
```

デッドラインタスクを実行しているCPUは、そのデッドラインタスクがブロックするか予算を使い切るまで、決してfairタスクを実行しない。これにより、コアスケジューラに特別扱いを設けることなく、リアルタイムタスクとデッドラインタスクに厳格な優先権が与えられる。

Linux 6.12以降、これら5つに加えて6番目のクラスがある: [sched_ext](sched-ext.md)（`ext_sched_class`）である。これは、カーネルにコンパイルされたクラスの代わりに、BPFプログラムがスケジューリングポリシーを提供できるようにする。

## struct sched_class {#struct-sched_class}

各スケジューラクラスは、`kernel/sched/sched.h`で定義された共通インターフェースを実装する:

```c
// kernel/sched/sched.h
struct sched_class {
    void (*enqueue_task)(struct rq *rq, struct task_struct *p, int flags);
    bool (*dequeue_task)(struct rq *rq, struct task_struct *p, int flags);

    struct task_struct *(*pick_task)(struct rq *rq, struct rq_flags *rf);
    struct task_struct *(*pick_next_task)(struct rq *rq,
                                          struct task_struct *prev,
                                          struct rq_flags *rf);
    void (*put_prev_task)(struct rq *rq, struct task_struct *p,
                          struct task_struct *next);
    void (*set_next_task)(struct rq *rq, struct task_struct *p, bool first);

    void (*task_tick)(struct rq *rq, struct task_struct *p, int queued);
    void (*prio_changed)(struct rq *this_rq, struct task_struct *task,
                         int oldprio);
    void (*update_curr)(struct rq *rq);
    // ...（その他のフック）
};
```

主要なコールバック:

| コールバック | 呼ばれるタイミング |
|----------|-----------------|
| `enqueue_task` | タスクが実行可能になった（起床、新規タスク） |
| `dequeue_task` | タスクがブロックまたは終了した |
| `pick_next_task` | スケジューラが次に実行するタスクを必要としている |
| `put_prev_task` | 現在実行中のタスクがプリエンプトされようとしている |
| `task_tick` | タイマー割り込み — タスクをプリエンプトすべきかを確認する |
| `prio_changed` | タスクの優先度が変わった（例: `nice()`） |
| `update_curr` | 現在のタスクの計上を更新する |

## クラスの登録 {#class-registration}

各クラスは`DEFINE_SCHED_CLASS`マクロで登録される。このマクロは構造体を、順序の決まった特別なリンカセクションに配置する:

```c
// kernel/sched/sched.h
#define DEFINE_SCHED_CLASS(name) \
const struct sched_class name##_sched_class \
    __aligned(__alignof__(struct sched_class)) \
    __section("__sched_classes")
```

クラスはセクション内で最高から最低の優先度の順に並ぶ。`for_each_class()`マクロがそれらをたどる:

```c
// kernel/sched/sched.h
#define for_each_class(class) \
    for (class = __sched_class_highest; \
         class <= __sched_class_lowest; \
         class++)
```

## 5つのクラス {#the-five-classes}

### 1. stop（`stop_sched_class`） {#1-stop-stop_sched_class}

**ファイル**: `kernel/sched/stop_task.c` | **導入**: [`34f971f6f798`](https://git.kernel.org/linus/34f971f6f7988be4d014eec3e3526bee6d007ffa)（Peter Zijlstra、v2.6.37）

stopクラスはすべての中で最も高い優先度を持つ。CPUマイグレーションとホットプラグ操作を実装する、CPUごとの「stopper」スレッドに使われる。

stopperスレッドは何でも — 優先度99のSCHED_FIFOタスクでさえ — プリエンプトできる。このクラスを直接扱うことはない。完全にカーネル内部のものである。

```c
// kernel/sched/stop_task.c
DEFINE_SCHED_CLASS(stop) = {
    .enqueue_task   = enqueue_task_stop,
    .dequeue_task   = dequeue_task_stop,
    .pick_next_task = pick_next_task_stop,
    // ...
};
```

### 2. deadline（`dl_sched_class`） {#2-deadline-dl_sched_class}

**ファイル**: `kernel/sched/deadline.c` | **導入**: [`aab03e05e8f7`](https://git.kernel.org/linus/aab03e05e8f7e26f51dee792beddcb5cca9215a5)（Dario Faggioli、Juri Lelliら、v3.14）

`SCHED_DEADLINE`はConstant Bandwidth Server（CBS）スケジューリングを実装する。各タスクは最悪実行時間（`runtime`）と周期を宣言する。カーネルはその予算の範囲内でタスクを実行し、どのデッドラインタスクを次に実行するかの選択にはEDF（Earliest Deadline First）の赤黒木を使う。

```c
// タスクをSCHED_DEADLINEに設定する
struct sched_attr attr = {
    .sched_policy   = SCHED_DEADLINE,
    .sched_runtime  = 5000000,   // 周期あたり5msの実行
    .sched_deadline = 10000000,  // 10ms以内に終える必要がある
    .sched_period   = 10000000,  // 周期は10ms
};
sched_setattr(0, &attr, 0);
```

カーネルはアドミッション制御を強制する: 受け入れるとシステムが実行不能になる（CPUあたりの合計使用率が1.0を超える）場合、`SCHED_DEADLINE`タスクを拒否する。

### 3. rt（`rt_sched_class`） {#3-rt-rt_sched_class}

**ファイル**: `kernel/sched/rt.c`

2つのポリシーがRTクラスを共有する:

- **`SCHED_FIFO`**: ブロックするか、より高優先度のタスクが起床するまで実行する。タイムスライスはない。
- **`SCHED_RR`**: FIFOと同じだが、同じ優先度のタスクには固定のタイムスライス（デフォルト100ms）が与えられ、それを使い切るとその優先度のキューの末尾に回る。

RTタスクの優先度は1–99である。RTランキューは100段階の優先度レベルのビットマップを使う — 旧スケジューラと同じO(1)ビットマップのアイデアであり、RTワークロードは通常小さく有界であるため、ここでは今でも適切である。

```c
// タスクを優先度50のSCHED_FIFOに設定する
struct sched_param param = { .sched_priority = 50 };
sched_setscheduler(pid, SCHED_FIFO, &param);
```

**危険**: 決してスリープしないRTタスクは、すべての通常タスクを飢餓状態にする。カーネルは`rt_throttle`でこれを防ぐ: デフォルトでは、RTタスクは1秒の時間窓あたり最大95%のCPU時間しか使えない（`/proc/sys/kernel/sched_rt_runtime_us`と`sched_rt_period_us`）。

### 4. fair（`fair_sched_class`） {#4-fair-fair_sched_class}

**ファイル**: `kernel/sched/fair.c`

fairクラスは3つのポリシーを扱う:

| ポリシー | 用途 |
|--------|----------|
| `SCHED_NORMAL` | すべてのユーザープロセスのデフォルト |
| `SCHED_BATCH` | CPUバウンドなバッチジョブ — プリエンプションが少なく、対話性ブーストなし |
| `SCHED_IDLE` | バックグラウンドタスク。どの`SCHED_NORMAL`タスクよりも優先度が低い |

CFSとEEVDFが存在するのはここである。詳細は[CFS](cfs.md)と[EEVDF](eevdf.md)を参照。

### 5. idle（`idle_sched_class`） {#5-idle-idle_sched_class}

**ファイル**: `kernel/sched/idle.c`

idleクラスは、CPUごとのアイドルスレッド（`swapper/N`）を実行する。他のどのクラスにも実行可能なタスクがない場合にのみ選ばれる。

アイドルスレッドは`cpu_idle_loop()`を呼び、それがCPUのアイドル命令（x86では`hlt`、ARMでは`wfi`）を実行する。さらに次のことも担当する:

- CPU周波数スケーリングのコールバック
- RCUの静止状態（quiescent state）の報告
- スリープ前の新しい仕事のポーリング

```c
// kernel/sched/idle.c
DEFINE_SCHED_CLASS(idle) = {
    .enqueue_task   = enqueue_task_idle,
    .dequeue_task   = dequeue_task_idle,
    .pick_next_task = pick_next_task_idle,
    // ...
};
```

## タスクのポリシーとクラスの対応 {#task-policy-and-class-mapping}

タスクのスケジューリングポリシーは`task_struct->policy`に格納される。クラスへのポインタは`task_struct->sched_class`にキャッシュされる:

```c
// include/uapi/linux/sched.h — ポリシーの値
#define SCHED_NORMAL    0
#define SCHED_FIFO      1
#define SCHED_RR        2
#define SCHED_BATCH     3
#define SCHED_IDLE      5
#define SCHED_DEADLINE  6
```

`sched_setscheduler()`や`sched_setattr()`でタスクのポリシーを設定すると、`task_struct->sched_class`も対応するクラス構造体を指すよう更新される。

## pick_next_taskの動作 {#how-pick_next_task-works}

中核のスケジューリング関数はクラス階層をたどる:

```c
// kernel/sched/core.c（簡略化）
static struct task_struct *
pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
    const struct sched_class *class;

    // 最適化された高速パス: すべてのタスクがfairクラスならループを省略する
    if (likely(!sched_class_above(prev->sched_class, &fair_sched_class) &&
               rq->nr_running == rq->cfs.h_nr_running))
        return pick_next_task_fair(rq, prev, rf);

    // 一般パス: 最高優先度のクラスから最低優先度のクラスへたどる
    for_each_class(class) {
        p = class->pick_next_task(rq, prev, rf);
        if (p)
            return p;
    }
    BUG(); // idleクラスは常に何かを返す
}
```

高速パスの効果は大きい: 通常タスクだけを実行しているシステム（一般的なケース）では、カーネルはstop、deadline、rtを先に確認することなく、`fair_sched_class`へ直行する。

## 優先度の範囲 {#priority-ranges}

```
Policy          Priority range   Nice range   Who can set
──────────────────────────────────────────────────────────
SCHED_DEADLINE  n/a              n/a          CAP_SYS_NICE
SCHED_FIFO      1–99             n/a          CAP_SYS_NICE
SCHED_RR        1–99             n/a          CAP_SYS_NICE
SCHED_NORMAL    0                -20 to +19   any user (raise only)
SCHED_BATCH     0                -20 to +19   any user
SCHED_IDLE      0                n/a          CAP_SYS_NICE
```

## 参考文献 {#further-reading}

- [スケジューラの進化](scheduler-evolution.md) — なぜこの設計になったのか
- [CFS](cfs.md) — fairクラスの詳細
- [RTスケジューラ](rt-scheduler.md) — RTクラスの詳細
- [SCHED_DEADLINE](deadline.md) — CBSとアドミッション制御
