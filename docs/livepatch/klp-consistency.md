# KLP一貫性モデル {#klp-consistency-model}

> パッチを有効化する前に、カーネルはすべてのタスクが安全な状態にあることをどのように保証するか

## 問題 {#the-problem}

ライブパッチの適用は瞬時には終わらない。パッチがロードされた瞬間、
一部のタスクは古い関数の内部を実行しているかもしれない。スタックの途中にいて、
ローカル変数が、新しい関数では扱い方が異なるかもしれないデータ構造を参照している。
その瞬間に盲目的に呼び出しをリダイレクトすれば、そうしたタスクは
一貫性のない状態に置かれてしまう。

KLPの一貫性モデルは、システム内のすべてのタスクを追跡し、
すべてのタスクが安全な地点に到達して初めてパッチを完全に有効とみなすことでこれを解決する。
安全な地点とは、古い関数から戻ったこと、freezableなスリープに入ったこと、
あるいはカーネルからユーザー空間へ戻る経路に到達したことである。

## タスクごとのパッチ状態 {#per-task-patch-state}

各タスクは`struct task_struct`（`include/linux/sched.h`）に`patch_state`フィールドを
持つ。ライブパッチの遷移中、このフィールドは`include/linux/livepatch.h`で定義された
3つの値のいずれかを保持する。

```c
/* include/linux/livepatch.h — 「タスクのパッチ状態」 */
#define KLP_TRANSITION_IDLE       -1   /* 遷移は進行していない */
#define KLP_TRANSITION_UNPATCHED   0   /* タスクは元の関数を呼ぶべき */
#define KLP_TRANSITION_PATCHED     1   /* タスクはパッチ適用済みの関数を呼ぶべき */
```

`KLP_TRANSITION_IDLE`は、遷移が進行していないときの定常値である。
`klp_init_transition()`は、遷移が可視になる前に、すべてのタスクに実際の開始状態
（`KLP_TRANSITION_UNPATCHED`または`KLP_TRANSITION_PATCHED`のうち目標でない方）を
割り当てる。そのため、遷移の途中で生きているタスクが`KLP_TRANSITION_IDLE`の状態で
観測されることは実際にはない。`kernel/livepatch/patch.c`のftraceハンドラは
`WARN_ON_ONCE(patch_state ==
KLP_TRANSITION_IDLE)`でこれを表明してさえいる。

遷移の方向は、順方向（パッチの有効化:
`KLP_TRANSITION_UNPATCHED` → `KLP_TRANSITION_PATCHED`）と逆方向（パッチの無効化:
`KLP_TRANSITION_PATCHED` → `KLP_TRANSITION_UNPATCHED`）のどちらもありうる。

### klp_update_patch_state()の仕組み {#how-klpupdatepatch_state-works}

`klp_update_patch_state()`は、単一のタスクをある状態から次の状態へ移す
関数である。

```c
/* kernel/livepatch/transition.c */
void klp_update_patch_state(struct task_struct *task)
{
    preempt_disable_notrace();

    /*
     * TIF_PATCH_PENDINGがセットされていた場合にのみ、それをクリアして
     * patch_stateを更新する。これが仕組みである: 状態の比較ではなく、
     * ペンディングフラグの検査とクリアである。
     */
    if (test_and_clear_tsk_thread_flag(task, TIF_PATCH_PENDING))
        task->patch_state = READ_ONCE(klp_target_state);

    preempt_enable_notrace();
}
```

グローバルな`klp_target_state`は、遷移の機構によって、有効化時には
`KLP_TRANSITION_PATCHED`に、無効化時には`KLP_TRANSITION_UNPATCHED`に設定される。
`klp_update_patch_state()`は、`TIF_PATCH_PENDING`がセットされているときは常に、
カーネルからユーザー空間へ戻る経路（`kernel/entry/common.c`の`__exit_to_user_mode_loop()`）から
`current`に対して呼ばれる。また強制遷移の際には`klp_force_transition()`から
すべてのタスクに対して直接呼ばれる。

## スタック検査: klp_check_stack() {#stack-checking-klpcheckstack}

`klp_update_patch_state()`はスタック検査を**行わない**。これは無条件の
フラグ検査とクリアである。カーネルからユーザー空間へ戻る経路ではそれで安全である。
まさにユーザー空間へ戻ろうとしているタスクが、パッチ対象のカーネル関数の本体の
実行途中にいることはありえないからである。

他の2つの経路はスタック検査を*行う*。どちらも
`klp_check_stack()`を通る。

- **workqueue経路**（`klp_try_complete_transition()`。後述）:
  およそ1秒に1回実行され、スリープ中やブロック中のものも含めて
  システム内のすべてのタスクを走査する。
- **スケジューラ経路**: 遷移の進行中、`__schedule()`は
  `klp_sched_try_switch(prev)`（`kernel/sched/core.c`）を呼ぶ。これは、遷移の間
  `klp_resched_enable()`/`klp_resched_disable()`が切り替えるstatic keyで
  ゲートされており、すべてのコンテキストスイッチではなく、切り替えで出ていくタスクが
  freezableなスリープ状態（`TASK_FREEZABLE`）に入ろうとしているコンテキストスイッチで
  `klp_try_switch_task(current)`を呼ぶ。この経路の目的はカーネルスレッドである。
  kthreadはユーザー空間へ戻ることがないので、カーネルからユーザー空間へ戻る経路が
  どれほど頻繁に実行されても、それを捕まえることはできない。しかし多くのkthreadは、
  作業項目の合間にfreezableなスリープで待機する。そして`TASK_FREEZABLE`はまさに、
  freezerが既に依存している「既知の安全な状態でスリープしている」という性質である。
  そこでスケジューラのフックが代わりにその場所でそれらを切り替える。逆のケースは
  カバーされて*いない*ことに注意。決してスリープしない本当にCPUバウンドなkthreadは
  どちらの経路にも到達せず、定期的なworkqueueの検査は実際にCPU上で実行中の
  タスクに対して`-EBUSY`を返す。そのためそのようなタスクは、スリープするか
  遷移が強制されるまで遷移を停滞させる。

`kernel/livepatch/transition.c`の`klp_sched_try_switch_key`の上にあるコメントは、
いまだにこのstatic keyを「CPUバウンドなkthreadにパッチを当てるのを助ける」ものとして
説明している。この表現はコミット
[`676e8cf70cb0`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=676e8cf70cb0)
（"sched,livepatch: Untangle cond_resched() and live-patching"、2025-05-09）より前のものである。
このコミットはフックを`cond_resched()`（ここでは譲歩するビジーループを本当に捕まえられた）から
`__schedule()`内の`TASK_FREEZABLE`スリープへ移した。
コメントではなく、`klp_sched_try_switch()`（`include/linux/livepatch_sched.h`）の
`TASK_FREEZABLE`の判定を読むこと。

いずれにせよ、タスクをスタック検査を行う経路で切り替える前に、
カーネルは古い（パッチ未適用の）関数、あるいはパッチが別のパッチの上に既に
積み重なっている場合は以前にパッチされた関数が、そのタスクのコールスタック上に
ないことを確認しなければならない。もしあれば、そのタスクはその関数の実行途中であり、
まだ安全に遷移させることはできない。

```c
/* kernel/livepatch/transition.c */
static int klp_check_stack_func(struct klp_func *func, unsigned long *entries,
                                unsigned int nr_entries)
{
    unsigned long func_addr, func_size, address;
    struct klp_ops *ops;
    int i;

    if (klp_target_state == KLP_TRANSITION_UNPATCHED) {
        /*
         * パッチを外される予定の関数
         * （func自身）を検査する。
         */
        func_addr = (unsigned long)func->new_func;
        func_size = func->new_size;
    } else {
        /*
         * パッチを当てられる予定の関数
         * （前のfunc）を検査する。
         */
        ops = klp_find_ops(func->old_func);

        if (list_is_singular(&ops->func_stack)) {
            /* 元の関数 */
            func_addr = (unsigned long)func->old_func;
            func_size = func->old_size;
        } else {
            /* 以前にパッチされた関数 */
            struct klp_func *prev;

            prev = list_next_entry(func, stack_node);
            func_addr = (unsigned long)prev->new_func;
            func_size = prev->new_size;
        }
    }

    for (i = 0; i < nr_entries; i++) {
        address = entries[i];

        if (address >= func_addr && address < func_addr + func_size)
            return -EAGAIN;
    }
    return 0;
}
```

パッチの積み重ねを考慮した分岐に注目してほしい。（パッチを外すのではなく）*パッチを当てて*
安全かを検査する際、この関数は単に`func->old_func`を探すのではない。
別のパッチが既にこの関数を置き換えている（`func_stack`に2つ以上のエントリがある）場合、
*以前にパッチされた*関数（`list_next_entry(func, stack_node)`の`new_func`）を探す。
それが今まさにスタック上で実行されているものだからである。後述の「パッチの積み重ね」を参照。

この関数は単一のアドレスではなく、`stack_trace_save_tsk_reliable()`が取得した
スタックフレーム配列全体を受け取る。

```c
/* kernel/livepatch/transition.c */
#define MAX_STACK_ENTRIES  100
static DEFINE_PER_CPU(unsigned long[MAX_STACK_ENTRIES], klp_stack_entries);

static int klp_check_stack(struct task_struct *task, const char **oldname)
{
    unsigned long *entries = this_cpu_ptr(klp_stack_entries);
    struct klp_object *obj;
    struct klp_func *func;
    int ret, nr_entries;

    ret = stack_trace_save_tsk_reliable(task, entries, MAX_STACK_ENTRIES);
    if (ret < 0)
        return -EINVAL;
    nr_entries = ret;

    klp_for_each_object(klp_transition_patch, obj) {
        if (!obj->patched)
            continue;
        klp_for_each_func(obj, func) {
            ret = klp_check_stack_func(func, entries, nr_entries);
            if (ret) {
                *oldname = func->old_name;
                return -EADDRINUSE;
            }
        }
    }

    return 0;
}
```

`klp_check_stack()`が走査するのは、現在遷移中の単一のパッチ
（`klp_transition_patch`、`klp_for_each_object()`経由）だけであり、有効なすべての
パッチではない。一度に遷移の途中にいられるパッチは1つだけだからである。スタック
バッファ自体はper-CPU配列（`klp_stack_entries`。サイズは`KLP_MAX_STACK_ENTRIES`定数ではなく、
非公開の`#define`である`MAX_STACK_ENTRIES`で決まる）であり、
プリエンプションを無効化することで保護されている。

`klp_check_stack()`は、スタックトレース自体が信頼できない場合
（`stack_trace_save_tsk_reliable()`が失敗した場合）は`-EINVAL`を返し、
スタックはきれいにアンワインドできたがパッチを当てる予定またはパッチを外す予定の関数が
その上に見つかった場合は`-EADDRINUSE`を返す。いずれの場合も、そのタスクは今回の遷移ラウンドでは
スキップされる。遷移の機構は次のworkqueueの起動時に再試行する。

## 遷移workqueue {#the-transition-workqueue}

単一の作業項目`klp_transition_work`が遷移ループを駆動する。

```c
/* kernel/livepatch/transition.c */
static void klp_transition_work_fn(struct work_struct *work)
{
    mutex_lock(&klp_mutex);

    if (klp_transition_patch)
        klp_try_complete_transition();

    mutex_unlock(&klp_mutex);
}

static DECLARE_DELAYED_WORK(klp_transition_work, klp_transition_work_fn);
```

`klp_try_complete_transition()`内の実際の呼び出しチェーンは`klp_try_complete_transition()` → `klp_try_switch_task(task)` → `klp_check_and_switch_task()`である。`klp_check_and_switch_task()`関数（current以外のタスクに対しては`task_call_func()`経由で使われる）は`klp_check_stack()`を実行し、それが0を返せば`TIF_PATCH_PENDING`をクリアして`task->patch_state = klp_target_state`を直接設定する。前節のフラグ検査・クリア関数である`klp_update_patch_state()`が、`klp_try_complete_transition()`のメインループから呼ばれることはない。その関数はカーネルからユーザー空間へ戻る経路と`klp_force_transition()`専用である。

```c
/* kernel/livepatch/transition.c — 簡略化した構造 */
void klp_try_complete_transition(void)
{
    unsigned int cpu;
    struct task_struct *g, *task;
    bool complete = true;

    /*
     * klp_try_switch_taskはtask_call_funcを使って、対象タスク上で
     * klp_check_and_switch_taskを安全に実行する。成功時
     * （タスクがklp_target_stateになった）にtrueを返す。
     */
    for_each_process_thread(g, task)
        if (!klp_try_switch_task(task))
            complete = false;

    /* idle（"swapper"）タスクも同じ方法で検査する */
    for_each_possible_cpu(cpu) {
        task = idle_task(cpu);
        if (!klp_try_switch_task(task))
            complete = false;
    }

    if (!complete) {
        /* まだ切り替わっていないタスクがある。後で再試行する */
        schedule_delayed_work(&klp_transition_work, round_jiffies_relative(HZ));
        return;
    }

    /* すべてのタスクが遷移した */
    klp_complete_transition();
}
```

定期的な再チェックはおよそ1秒ごと（1 HZ）に実行される。

## klp_send_signals(): ブロック中のタスクを促す {#klpsendsignals-nudging-blocked-tasks}

`klp_send_signals()`はPOSIXシグナルを**一切送らない**。実際の仕組みは次のとおり。

- **kthread**（`task->flags & PF_KTHREAD`）に対して: `wake_up_state(task, TASK_INTERRUPTIBLE)`を呼ぶ。スリープ中のkthreadに対するスケジューラの直接の起床である
- **ユーザータスク**に対して: `set_notify_signal(task)`を呼ぶ。`TIF_NOTIFY_SIGNAL`をセットし、タスクが次のシグナルチェック地点でユーザー空間へ戻るようにする

どちらの経路も`TASK_UNINTERRUPTIBLE`（D状態）のタスクを起こすことはできない。D状態のタスクは、遷移がそれを含められるようになる前に、自然にその状態を抜けなければならない。これがlivepatchの遷移が長時間停滞しうる理由であり、`force`機構はまさにD状態のタスクを遷移させられない状況のために存在する。

`klp_send_signals()`は、遷移が停滞してからは`klp_try_complete_transition()`から
およそ15秒ごとに呼ばれる（上述の1ラウンドあたり約1秒の間隔で、未完了の再試行ラウンドの
`SIGNALS_TIMEOUT`回ごと）。

## パッチの積み重ね: func_stackとstruct klp_ops {#patch-stacking-funcstack-and-struct-klpops}

複数のライブパッチが同時に有効で、それぞれが同じ関数にパッチを当てている場合、
カーネルはどの置き換え関数が現在有効なのかを知る必要がある。これは
`struct klp_ops`とその`func_stack`によって管理される。

```c
/* kernel/livepatch/patch.h */
struct klp_ops {
    struct list_head  node;        /* klp_opsリスト内のエントリ */
    struct list_head  func_stack;  /* klp_funcのスタック。最新のものが先頭 */
    struct ftrace_ops fops;        /* この関数用のftraceフック */
};
```

`klp_ops`は（オブジェクト, 関数名）の組ごとにちょうど1つ存在し、すべてのパッチで
共有される。2つ目のパッチが同じ関数を対象にすると、その`klp_func`は
`func_stack`の先頭にプッシュされる。ftraceハンドラは常に先頭を使う。

```c
/* kernel/livepatch/patch.c */
func = list_first_or_null_rcu(&ops->func_stack, struct klp_func, stack_node);
```

したがって、最後に有効化されたパッチが勝つ。パッチを無効化するとその
`klp_func`がスタックから取り除かれ、前のパッチ（スタックが空なら元の
関数）が復元される。

```
func_stack (head → tail):

  [P3: patched_tcp_sendmsg]  ← active (P3 enabled last)
  [P1: patched_tcp_sendmsg]  ← shadowed by P3
```

*パッチの積み重ね*と呼ばれるこの仕組みにより、パッチは
互いを意識する必要がない。スタックが順序を
自動的に処理する。

## 強制遷移 {#forced-transitions}

遷移が停滞して完了できない場合（例えば、古い関数の内部で永遠にループする
kthread）、強制遷移を起動できる。

```bash
# 遷移を強制する — すべてのタスクについてスタック検査を飛ばす
echo 1 > /sys/kernel/livepatch/<patch>/force
```

遷移の強制は**安全ではない**。強制した瞬間に古い関数の内部を実行していたタスクは
古い関数のコードを実行し続けるが、パッチ状態は`KLP_TRANSITION_PATCHED`に
設定される。新しい関数がデータレイアウトや前提を変更していれば、それらのタスクは
一貫性のない状態にアクセスしうる。

その後どのパッチにフラグが立つかは、見た目より限定的である。
`klp_force_transition()`は`struct klp_patch`の`forced`フィールドを次のように
設定する。

```c
/* kernel/livepatch/transition.c — klp_force_transition()の末尾 */
/* 取り除かれるパッチにforcedフラグをセットする。 */
if (klp_target_state == KLP_TRANSITION_UNPATCHED)
    klp_transition_patch->forced = true;
else if (klp_transition_patch->replace) {
    klp_for_each_patch(patch) {
        if (patch != klp_transition_patch)
            patch->forced = true;
    }
}
```

つまりケースは3つあり、フラグを立てるのはそのうち2つだけである。

- **無効化**の強制（`klp_target_state == KLP_TRANSITION_UNPATCHED`）は、
  無効化されるパッチに`forced`をセットする。そのパッチは二度とアンロードできない。
- **`.replace`（累積）パッチの有効化**の強制は、*他の*すべてのパッチ（このパッチが
  置き換えるもの）に`forced`をセットし、強制的に有効化されるパッチ自身には
  セット**しない**。置き換えられたモジュールは永久にアンロード不能になるが、
  新しいモジュールはそうならない。
- `replace`でないパッチの単純な**有効化**の強制は、まさに上で述べた
  シナリオ（順方向の遷移が待っている間、kthreadが古い関数の内部で永遠にループしている）
  であるが、**何にも**`forced`をセットしない。遷移は安全でない形で
  完了するが、どのパッチにも印は付かず、強制したことは`forced`フラグの
  痕跡として一切残らない。

フラグがセット*される*場合も、それは内部専用であり、読み出すためのsysfsファイルはない。
外部から見える唯一の効果は、フラグの立ったパッチのモジュールが
二度とアンロードできないことである。`klp_free_patch_finish()`（`kernel/livepatch/core.c`）は
`if (!patch->forced) module_put(patch->mod);`で終わるので、forcedのパッチは
取得したモジュール参照を決して返さず、その参照カウントは決して
0にならない。この帰結は上の規則でフラグが立ったモジュールに付くものであり、
`force`ファイルに書き込んだモジュールとは限らないことに注意。

`TAINT_LIVEPATCH`は、すべてのlivepatchモジュールに対して**モジュールのロード時**に適用される。強制遷移の有無によって条件付きで付くものではない。すべてのライブパッチ適用は`TAINT_LIVEPATCH`（ビット15）でカーネルをtaintする。`klp_force_transition()`自体は追加のtaintを付けない。`TAINT_FORCED_MODULE`は無関係である。これはモジュールが`insmod -f`で強制ロードされたときにセットされるものであり、livepatchの遷移が強制されたときではない。

強制遷移は、影響を受けるタスクが新しい関数の依存するコールパス上にないことを
`/proc/<pid>/stack`を読んで確認したうえで、最後の手段としてのみ
使うこと。

## klp_complete_transition(): パッチの確定 {#klpcompletetransition-finalizing-the-patch}

すべてのタスクが遷移を終えると、`klp_complete_transition()`が呼ばれる。

```c
/* kernel/livepatch/transition.c */
static void klp_complete_transition(void)
{
    struct klp_object *obj;
    struct klp_func *func;
    struct task_struct *g, *task;
    unsigned int cpu;

    /* 累積（replace）パッチの場合: 置き換えられたすべてのパッチを外す。
     * これによりそれらのfuncがfunc_stackから取り除かれる。 */
    if (klp_transition_patch->replace && klp_target_state == KLP_TRANSITION_PATCHED) {
        klp_unpatch_replaced_patches(klp_transition_patch);
        klp_discard_nops(klp_transition_patch);
    }

    if (klp_target_state == KLP_TRANSITION_UNPATCHED) {
        /*
         * すべてのタスクがKLP_TRANSITION_UNPATCHEDへ遷移したので、
         * 新しい関数をfunc_stackから取り除ける。
         */
        klp_unpatch_objects(klp_transition_patch);
        klp_synchronize_transition();
    }

    /* funcごとの遷移フラグをクリアする */
    klp_for_each_object(klp_transition_patch, obj)
        klp_for_each_func(obj, func)
            func->transition = false;

    /* klp_ftrace_handler()がKLP_TRANSITION_IDLE状態を見ないようにする */
    if (klp_target_state == KLP_TRANSITION_PATCHED)
        klp_synchronize_transition();

    /* idleタスクを含め、タスクごとの遷移状態をクリアする */
    read_lock(&tasklist_lock);
    for_each_process_thread(g, task)
        task->patch_state = KLP_TRANSITION_IDLE;
    read_unlock(&tasklist_lock);

    for_each_possible_cpu(cpu)
        idle_task(cpu)->patch_state = KLP_TRANSITION_IDLE;

    /* 遷移が終わったので、post-(un)patchコールバックを実行する */
    klp_for_each_object(klp_transition_patch, obj) {
        if (!klp_is_object_loaded(obj))
            continue;
        if (klp_target_state == KLP_TRANSITION_PATCHED)
            klp_post_patch_callback(obj);
        else if (klp_target_state == KLP_TRANSITION_UNPATCHED)
            klp_post_unpatch_callback(obj);
    }

    klp_target_state = KLP_TRANSITION_IDLE;
    klp_transition_patch = NULL;
}
```

`klp_complete_transition()`が戻ると、sysfsの`transition`ファイルは
`0`を返し、パッチは完全に有効になる。

## 遷移の観察 {#observing-the-transition}

```bash
# パッチはまだ遷移中か？
cat /sys/kernel/livepatch/<patch>/transition
# 1 = 進行中、0 = 完了

# 遷移が強制されたことがあるかどうかは内部専用（struct klp_patch.forced）で、
# sysfsでは公開されていない。観察できる唯一の副作用は、フラグの立った
# モジュールに対して`rmmod`が永久に拒否されることである。そしてそれは強制した
# モジュールとは限らない: 無効化を強制すると無効化されるパッチにフラグが立ち、
# .replaceパッチの有効化を強制すると（自身ではなく）置き換えられたパッチにフラグが立ち、
# replaceでない単純な有効化を強制すると何にもフラグは立たない。

# どのタスクが遷移を妨げているか？
# （古い関数がスタック上にあるタスク）
for pid in $(ls /proc/ | grep -E '^[0-9]+$'); do
    stack=$(cat /proc/$pid/stack 2>/dev/null)
    if echo "$stack" | grep -q "old_function_name"; then
        echo "PID $pid is blocking transition"
        cat /proc/$pid/stack
    fi
done

# 遷移中のカーネルログ
dmesg | grep livepatch
# livepatch: 'mypatch': starting patching transition
# livepatch: 'mypatch': patching complete
```

## 遷移のステートマシン {#transition-state-machine}

```
klp_enable_patch()  →  __klp_enable_patch()
       │
       ▼
  klp_init_transition():  klp_target_state = KLP_TRANSITION_PATCHED
                          func->transition = true
  klp_start_transition(): set TIF_PATCH_PENDING on every task
  patch->enabled = true   ← set HERE, at the start of the transition,
                            not when it completes
       │
       ▼ (first call is immediate; then periodic, every ~1s)
  klp_try_complete_transition()
       │
       ├── for each task (and each idle task):
       │     klp_try_switch_task() → klp_check_and_switch_task()
       │       task unsafe to switch (-EBUSY: running; -EINVAL / -EADDRINUSE from klp_check_stack())?
       │         yes: skip (try again next round)
       │         no:  task->patch_state = KLP_TRANSITION_PATCHED
       │
       ├── all tasks KLP_TRANSITION_PATCHED?
       │     no:  klp_send_signals(), reschedule work
       │     yes: klp_complete_transition()
       │             │
       │             ▼
       │         func->transition = false
       │         task->patch_state = KLP_TRANSITION_IDLE (all tasks)
       │         klp_target_state = KLP_TRANSITION_IDLE
       │         klp_transition_patch = NULL
       │         transition sysfs = 0
       │         (enabled was already 1 throughout — it is not
       │          touched by klp_complete_transition())
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/livepatch/transition.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/livepatch/transition.c) — `klp_check_stack()`、`klp_try_complete_transition()`、`klp_update_patch_state()`、`klp_force_transition()`、および遷移のステートマシンの残りの部分
- [kernel/livepatch/patch.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/livepatch/patch.c) — `func_stack`をたどって有効な置き換え関数を選ぶ`klp_ftrace_handler()`、および`klp_patch_func()`/`klp_unpatch_func()`
- [kernel/livepatch/core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/livepatch/core.c) — sysfs属性`/sys/kernel/livepatch/<patch>/{transition,force}`
- [include/linux/livepatch.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/livepatch.h) — `struct klp_func`、`struct klp_object`、`struct klp_patch`、および遷移状態の定数

### 関連ページ {#related-pages}

- [カーネルライブパッチ](klp.md) — struct klp_func/klp_patch、ftraceによるリダイレクト、shadow変数
- [累積パッチとアトミックリプレース](klp-cumulative.md) — パッチの積み重ねと.replace=true
- [KLP State: 独自の一貫性チェック](klp-state.md) — スタック走査だけでは検証できない変更のための`klp_state` API
- [ftrace](../tracing/ftrace.md) — KLPが使うftraceフック

### LWNの記事 {#lwn-articles}

- [LWN: livepatch: consistency model](https://lwn.net/Articles/632582/) — `kernel/livepatch/transition.c`に実装された、タスクごと・スタック検査型の遷移モデルについての2015年の最初の設計解説
- [LWN: An update on live kernel patching](https://lwn.net/Articles/734765/) — 遅延移行とスタック検査のハイブリッドモデル、および信頼できるスタック検査のためにORCアンワインダが必要だった理由を扱う2017年の状況報告

### 外部資料 {#external}

- [Kernel docs: Livepatch](https://docs.kernel.org/livepatch/livepatch.html) — 一貫性モデル、sysfsの`force`・`transition`ファイル、`func_stack`によるパッチの積み重ねについてのアップストリームの説明
- [Kernel docs: Reliable Stacktrace](https://docs.kernel.org/livepatch/reliable-stacktrace.html) — livepatchがスタック検査経路のために（ベストエフォートではなく）信頼できるスタックアンワインダを必要とする理由
