# KLP State: 独自の一貫性チェック {#klp-state-custom-consistency-checks}

> スタック走査だけでは足りないパッチのためのklp_state API

## スタック走査だけでは足りない問題 {#the-problem-with-stack-scanning-alone}

標準的なKLPの一貫性モデル、つまりすべてのタスクのスタックを走査し、
古い関数を実行しているタスクがなくなるまで待つという方式は、ほとんどのパッチにとって十分である。
単一の関数内のロジックエラーを修正するだけのパッチは、スタックに基づく遷移に完全に
頼ることができる。古い関数本体の実行途中にいるタスクがなくなれば、
全員を新しい関数へ切り替えても安全である。

しかし一部のパッチは、スタック走査では検出できないものを変更する。

- 共有構造体を保護するロックを変更するパッチ。古いコードは`spinlock_t`を取り、
  新しいコードはクリティカルセクションでスリープできるよう`mutex`を保持したい。
  一貫性モデルは最後のタスクが切り替わるまで意図的に古いコードと新しいコードを並行して
  実行させ、その期間中はどちらのロックも他方を排除しない。
- 古いコードと新しいコードの両方がアクセスする構造体に新しいフィールドを追加するパッチ。
  すべての呼び出し元が新しいコードパスを使うようになるまで、一部の呼び出し元は
  新しいフィールドを読み、他の呼び出し元はまだそれを無視する。
- 参照カウントされるオブジェクトの意味を変更するパッチ。古いコードはあるカウンタを
  インクリメントし、新しいコードは別のカウンタをインクリメントする。この2つは
  安全に共存できない。

これらすべてのケースで、パッチ作成者には遷移の瞬間に独自のロジックを実行する手段が
必要である。カーネルが自動的に推論できないロジックである。
Linux 5.5で追加された`klp_state` APIは、まさにそれを提供する。

## struct klp_state {#struct-klp_state}

```c
/* include/linux/livepatch.h */
struct klp_state {
    unsigned long  id;        /* この状態の、0でない一意な識別子 */
    unsigned int   version;   /* 累積パッチとの互換性のため */
    void          *data;      /* パッチ固有のポインタ。パッチが所有する */
};
```

パッチは、`id`が0のエントリで終端された`klp_state`エントリの配列を宣言する。

```c
static struct klp_state my_states[] = {
    {
        .id      = 1,        /* 任意の0でないid。バージョン間で不変 */
        .version = 1,        /* 状態の形式が変わったらインクリメントする */
    },
    { }                      /* 終端 */
};
```

`data`ポインタは、遷移中に必要な状態ごとの任意の情報を格納するために
パッチ作成者が使える。

## パッチに状態を付加する {#attaching-states-to-a-patch}

`states`配列は`struct klp_patch`のフィールドである。

```c
/* include/linux/livepatch.h */
struct klp_patch {
    struct module     *mod;
    struct klp_object *objs;
    struct klp_state  *states;   /* 任意。使わない場合はNULL */
    bool               replace;  /* 累積パッチではtrue */
    /* ... 内部フィールド ... */
};
```

`klp_state`を使うパッチは両方のフィールドを設定する。

```c
static struct klp_patch my_patch = {
    .mod     = THIS_MODULE,
    .objs    = my_objs,
    .states  = my_states,
    .replace = false,
};
```

## struct klp_objectの遷移コールバック {#transition-callbacks-in-struct-klp_object}

独自の一貫性ロジックが実行されるフックは、`struct klp_object`の中でオブジェクトごとに
宣言される。

```c
/* include/linux/livepatch.h */
struct klp_object {
    const char           *name;      /* vmlinuxの場合はNULL */
    struct klp_func      *funcs;
    struct klp_callbacks  callbacks; /* pre/post patch/unpatchフックを含む */
    /* ... 内部フィールド ... */
};

/* include/linux/livepatch_external.h */
struct klp_callbacks {
    int  (*pre_patch)(struct klp_object *obj);
    void (*post_patch)(struct klp_object *obj);
    void (*pre_unpatch)(struct klp_object *obj);
    void (*post_unpatch)(struct klp_object *obj);
    bool post_unpatch_enabled;
};
```

順方向の遷移（パッチの適用）における呼び出し順序:

1. `pre_patch(obj)` — ftraceフックがインストールされる前に呼ばれる。これが
   0でないエラーコードを返すと、`klp_enable_patch()`は中断してその
   エラーを返す。フックは一切インストールされない。
2. ftraceフックがインストールされ、タスクが新しい関数を見始める。
3. 一貫性の遷移が実行される（スタック走査、タスクごとの状態更新）。
4. `post_patch(obj)` — 遷移が完了し、パッチが完全に有効になった後に
   呼ばれる。上のシグネチャに注意。これは`void`を返す。この時点で
   パッチは適用済みであり、取り消す仕組みはない。そのため
   `post_patch`コールバックには失敗を報告する手段がなく、
   失敗しえないように書かなければならない。

逆方向の遷移（パッチの無効化）における呼び出し順序:

1. `pre_unpatch(obj)` — `__klp_disable_patch()`
   （`kernel/livepatch/core.c`）から`klp_start_transition()`の*前*に呼ばれる。
   そのため、すべてのタスクがまだパッチ適用済みのコードを実行している間に実行される。
2. 逆方向の遷移が実行される。
3. ftraceフックが取り除かれる。`klp_complete_transition()`が
   `klp_unpatch_objects()`（`kernel/livepatch/transition.c`）を呼ぶ。
4. `post_unpatch(obj)` — `klp_complete_transition()`の最後、
   ステップ3の後に呼ばれる。そのため実行される時点で、パッチ適用済みのコードには既に
   到達できない。これは、このオブジェクトの`pre_patch`コールバックが以前に
   成功していた場合にのみ呼ばれる。`klp_pre_patch_callback()`がそれを
   `post_unpatch_enabled`フラグ（`kernel/livepatch/core.h`）に記録する。

## klp_get_state()とklp_get_prev_state() {#klpgetstate-and-klpgetprev_state}

2つのヘルパーが、コールバック内から状態データへのアクセスを提供する。

```c
/* include/linux/livepatch.h（ここで宣言。実装はkernel/livepatch/state.c） */
struct klp_state *klp_get_state(struct klp_patch *patch, unsigned long id);
struct klp_state *klp_get_prev_state(unsigned long id);
```

`klp_get_state(patch, id)`は、`patch->states`から指定された`id`を持つ`klp_state`を
返す。そのidを持つ状態が存在しなければ`NULL`を返す。

`klp_get_prev_state(id)`は、*既にインストールされている*パッチの中から、指定された`id`を
持つ`klp_state`を探す。グローバルなパッチリストを順にたどり、
現在遷移中のパッチ（`klp_transition_patch`）に到達したところで止まり、
途中で見つかった**最後の**一致を返す。
単純に「1つ前のパッチ」を返すわけではない。複数の古いパッチが同じidを宣言していることもある。
`kernel/livepatch/state.c`のkernel-docは、「同じシステム状態が複数の非累積livepatchによって
変更されうる」こと、そして「最新のlivepatchが最も新しい情報を持っていることが
期待される」ことを述べている。
累積パッチは、置き換え対象のパッチから状態（カウンタ、フラグ、
割り当て済みデータ）を継承するためにこれを使う。

どちらの関数も`klp_mutex`を取得したり表明したりはしない。実際の前提条件は
それぞれ異なる。`klp_get_state()`は`patch->states`を単純にたどるだけであり、その
kernel-docは「pre/post (un)patchコールバックからでも、livepatchが追加したカーネルコードからでも
呼び出せる」と述べている。`klp_get_prev_state()`はより厳格である。グローバルな
`klp_transition_patch`を読み、冒頭で`WARN_ON_ONCE(!klp_transition_patch)`を実行し、
それが未設定なら`NULL`を返す。したがって、実際に遷移が進行中のときにしか
呼び出せない。実際には、コールバックの内部から呼ぶことになり、実行時にパッチ適用済みの
コードから呼ぶことはできない。

## 例: サブシステムをspinlockからmutexへ切り替える {#example-switching-a-subsystem-from-a-spinlock-to-a-mutex}

共有テーブルが`subsys_spinlock`で保護されているサブシステムを考える。ある
パッチは、クリティカルセクションでスリープできるよう、代わりに`mutex`を使いたい。2つのロックの
規律を単純に入れ替えることはできない。遷移が実行されている間は、
一部のタスクは古いコード上に、一部は新しいコード上にいるので、`subsys_spinlock`を保持する
古いコードの呼び出し元と`subsys_mutex`を保持する新しいコードの呼び出し元が、
それぞれ自分がテーブルを所有していると思い込んでしまう。

どんなコールバックもこの期間を消すことはできない。混在期間は、一貫性モデルが
設計どおりに動いている結果である。コールバックに*できる*のは、新しいコードが
その期間に対処できるようにし、その後でそれを終わらせることである。これが
`Documentation/livepatch/system-state.rst`の§4が規定するパターンであり、Nicolai
StangeのL1TF/KPTI向けライブパッチが使ったパターンである。

1. すべてのアクセサにパッチを当て、古い意味と新しい意味の**両方**を扱えるようにし、
   実行時にフラグで選択する。フラグがクリアされている間、新しい
   アクセサは古いロック規則に従い続ける。
2. 遷移を完了させる。その後で初めて、`post_patch()`からフラグをセットする。
   一貫性モデルはその時点で古いコード上にタスクがいないことを保証するので、
   古いコードの呼び出し元が新しい意味を観測することは決してない。
3. 取り外す際には、`pre_unpatch()`からフラグをクリアする。これは逆方向の遷移が始まる
   *前*、すべてのタスクがまだ新しいコード上にいる間に実行される。そのため、
   新しい意味がまだ生きている間にタスクが古いコードへ戻っていることはありえない。

フラグ自体は、それが調停する呼び出し元を実際に排除するロックの下で
読み書きしなければならない。ここではそのロックは`subsys_mutex`であり、パッチ適用済みの
アクセサは、*どちらの*モードでも、フラグの読み出しとテーブルへのアクセスの両方にまたがって
それを保持する。そのため、切り替えがクリティカルセクションの途中で起こることはない。

これはまた、`subsys_do_work()`のすべての呼び出し元がスリープを許されていることを前提としている。
そうでなければ、そもそもそこでmutexを使いたいということ自体が
意味をなさない。

```c
#include <linux/livepatch.h>
#include <linux/mutex.h>
#include <linux/spinlock.h>

/* id=1は「サブシステムのテーブルのロック規律」を識別する */
#define SUBSYS_LOCK_STATE_ID  1UL

/*
 * 前方宣言: 下のコールバックはklp_get_state(&subsys_patch, ...)で
 * 自身のパッチの状態を参照するが、完全なstruct klp_patchの定義
 * （コールバックとオブジェクトが先に定義されている必要がある）は
 * ファイルの最後でようやく揃う。これは実際のlivepatchモジュールが使う、
 * ファイル末尾で構造体を定義する順序と同じである（ほとんどのサンプル、
 * 例えばsamples/livepatch/livepatch-callbacks-demo.cは、コールバックが
 * 自身のパッチ構造体をアドレスで参照しないので前方宣言を必要としないが、
 * klp_get_state(&this_patch, ...)を呼ぶklp_stateの利用者には必要になる）。
 */
static struct klp_patch subsys_patch;

/*
 * パッチモジュールがKLPリロケーション経由で到達する、既存のvmlinuxシンボル。
 * このページの後半でsubsys_patch_v2を省略しているのと同様に、
 * 簡潔さのためここでは定義を省略する。
 */
extern spinlock_t subsys_spinlock;
extern void subsys_update_table(void);       /* 既存の、アトミックな更新 */
extern void subsys_update_table_slow(void);  /* スリープしうる。これがパッチの目的 */

/* 以下はすべて新規で、パッチモジュール内に置かれる。 */
static DEFINE_MUTEX(subsys_mutex);

/*
 * false = 混在モード。パッチ未適用のコードがまだ実行中かもしれず、
 *         それはテーブルをsubsys_spinlockだけで保護しているので、
 *         パッチ適用済みのアクセサもそれを取らなければならず、スリープしてはならない。
 * true  = 順方向の遷移が完了し、パッチ適用済みのコードだけが生きているので、
 *         subsys_mutexだけで十分であり、クリティカルセクションはスリープしてよい。
 *
 * subsys_mutexの下でのみ読み書きする。このリストの後の注記を参照。
 */
static bool subsys_use_mutex;

/* subsys_do_work()の置き換え: 両方の意味を実装する。 */
static void patched_subsys_do_work(void)
{
    mutex_lock(&subsys_mutex);

    if (!subsys_use_mutex) {
        /* 混在モード: まだ生きている元のコードと排他する。 */
        spin_lock(&subsys_spinlock);
        subsys_update_table();
        spin_unlock(&subsys_spinlock);
    } else {
        /* パッチ適用済みの呼び出し元しか残っていない。mutexだけで十分。 */
        subsys_update_table_slow();
    }

    mutex_unlock(&subsys_mutex);
}

static int pre_patch_subsys(struct klp_object *obj)
{
    struct klp_state *state;

    state = klp_get_state(&subsys_patch, SUBSYS_LOCK_STATE_ID);
    if (!state)
        return -EINVAL;

    /*
     * pre_patch()は準備をする。まだ意味を変えてはならない。
     * 元のコードがまだ生きていて、mutexだけのロックには対処できないからである。
     * 引き継ぎがまだ行われていないことだけを記録する。
     */
    state->data = (void *)0UL;

    return 0;
}

/*
 * post_patch()とpre_unpatch()は、NULLチェックなしで参照結果を
 * デリファレンスできる。pre_patch()がエラーを返したオブジェクトに対しては
 * どちらも実行されず（kernel/livepatch/core.cとcore.h）、状態が欠けていれば
 * pre_patch_subsys()が既にパッチを拒否しているからである。
 */
static void post_patch_subsys(struct klp_object *obj)
{
    struct klp_state *state = klp_get_state(&subsys_patch,
                                            SUBSYS_LOCK_STATE_ID);

    /*
     * 遷移は完了した: 元のsubsys_do_work()を実行しているタスクはもういないので、
     * テーブルをsubsys_spinlockだけで保護しているものはもう存在しえない。
     * subsys_mutexを取ることで、既にクリティカルセクションの中にいる
     * パッチ適用済みの呼び出し元の終了を待つ。そのため切り替えが、
     * 呼び出し元のフラグ読み出しとテーブルアクセスの間に入り込むことはない。
     * どちらも1回のmutex保持の中で起こるからである。
     *
     * post_patch()はプロセスコンテキストで、有効化を行うタスクまたは
     * 遷移workqueueのどちらかの上でklp_complete_transition()から実行されるので、
     * ここでmutexを待ってスリープすることは許される。
     */
    mutex_lock(&subsys_mutex);
    subsys_use_mutex = true;
    state->data = (void *)1UL;   /* 引き継ぎ完了 */
    mutex_unlock(&subsys_mutex);
}

static void pre_unpatch_subsys(struct klp_object *obj)
{
    struct klp_state *state = klp_get_state(&subsys_patch,
                                            SUBSYS_LOCK_STATE_ID);

    /*
     * post_patch_subsys()と対称であり、*ここで*実行しなければならない:
     * __klp_disable_patch()は、すべてのタスクがまだパッチ適用済みのコード上に
     * いる間に、klp_start_transition()の前にpre_unpatchを呼ぶ。今フラグを
     * クリアすれば、いずれかのタスクが元のspinlockだけのコードに戻る時点では、
     * 残っているパッチ適用済みの呼び出し元もすべて再びsubsys_spinlockを
     * 取っていることになる。
     */
    mutex_lock(&subsys_mutex);
    subsys_use_mutex = false;
    state->data = (void *)0UL;
    mutex_unlock(&subsys_mutex);
}

/*
 * post_unpatchコールバックは意図的に置いていない。klp_complete_transition()は
 * post_unpatchコールバックを呼ぶ前にklp_unpatch_objects()を呼んで
 * ftraceのリダイレクトを取り外すので、その時点ではpatched_subsys_do_work()を
 * 実行しているものは何もなく、そこでのフラグ書き込みはデッドコードになる。
 * 元に戻す処理はpre_unpatch_subsys()で既に行われている。これはまさに
 * Documentation/livepatch/system-state.rstが述べる対称性である: pre_unpatch()は
 * post_patch()を鏡写しにし、post_unpatch()は「何もしないことを意味するかもしれない」。
 */

static struct klp_state subsys_states[] = {
    { .id = SUBSYS_LOCK_STATE_ID, .version = 1 },
    { }
};

static struct klp_func subsys_funcs[] = {
    {
        .old_name = "subsys_do_work",
        .new_func = patched_subsys_do_work,
    },
    { }
};

static struct klp_object subsys_objs[] = {
    {
        .name  = NULL,   /* vmlinux */
        .funcs = subsys_funcs,
        .callbacks = {
            .pre_patch   = pre_patch_subsys,
            .post_patch  = post_patch_subsys,
            .pre_unpatch = pre_unpatch_subsys,
            /* .post_unpatchは意図的に未設定。上を参照 */
        },
    },
    { }
};

static struct klp_patch subsys_patch = {
    .mod    = THIS_MODULE,
    .objs   = subsys_objs,
    .states = subsys_states,
};
```

このプロトコル全体は1つの規則に依存している。`subsys_use_mutex`は
`subsys_mutex`の下でのみ読み書きされ、アクセサは読み出しとテーブルアクセスの両方に
またがってそのmutexを保持する。切り替えを別のロックの下で公開したり、
フラグをmutexの外で読んだりすると、まさにこの仕組みが閉じるために存在する競合が
再び開いてしまう。ある呼び出し元が`false`を読み、`spin_lock()`に到達する前に
プリエンプトされ、切り替えの後に再開する一方で、`true`を読んだ2つ目の呼び出し元が
既にmutexだけのクリティカルセクションの中にいる、という状況である。
そうなると両者が同時にテーブルに触れることになる。切り替えは、アクセサが
クリティカルセクション全体にわたって保持するのと同じmutexを取得しなければならないので、
そのような交錯は起こりえない。

この設計は、逆転した遷移にも耐える。これはコールバックが何もせずに
得られる性質である。順方向の遷移が完了前に逆転された場合（`transition`がまだ1を返す間に
`echo 0 > .../enabled`した場合）、`post_patch`は一度も実行されていないので、
フラグはセットされておらず、元に戻すものは何もない。*無効化*が逆転されて
パッチ適用に戻った場合、`pre_unpatch`は既にフラグをクリアしており、
`klp_complete_transition()`は戻る途中で再び`post_patch`を実行し、
フラグを再設定する。それはmutexの下で、一貫性モデルが古いコード上にタスクがいないことを
改めて確認した後に行われる。

`pre_patch_subsys`が0でない値を返した場合（例えば、
`klp_get_state()`が期待する状態エントリを見つけられなかった場合）、
`klp_enable_patch()`はそのエラーを`insmod`に伝播し、フックは一切
インストールされない。システムは部分的な状態を残さずに古いコードパスにとどまる。

## klp_stateと累積パッチ {#klp_state-and-cumulative-patches}

累積パッチが、同じく`klp_state`を使っていた以前のパッチを置き換えるとき、
新しいパッチは`klp_get_prev_state()`経由で前のパッチの状態データを継承できる。
これにより、後続の各パッチが前任者の確立した状態の上に積み上げていく、
複数バージョンにわたる移行が可能になる。

継承処理自体は`pre_patch()`ではなく`post_patch()`に置くべきである。
`system-state.rst`の§4は、「互換性がある場合は前のlivepatchから*state->data*をコピーする」を
`post_patch()`の項目に挙げている。メインの例がそこでフラグを切り替えるのと同じ理由である。
遷移が完了するまでは、古いパッチのコードがまだ生きていて、それが記述している
状態をまだ所有している。
`pre_patch()`は割り当てを行う場所である。ロードをまだ拒否できる
唯一のコールバックだからである。

```c
/* subsys_patch_v2: 置き換える側の新しいパッチ自身のstruct klp_patch。
 * 上のsubsys_patchと同じ方法で宣言される。簡潔さのため
 * ここでは省略する。 */
static int pre_patch_v2(struct klp_object *obj)
{
    struct klp_state *cur;

    cur = klp_get_state(&subsys_patch_v2, SUBSYS_LOCK_STATE_ID);
    if (!cur)
        return -EINVAL;

    /*
     * 新しい状態形式が必要とするものはここで割り当てる。これはロードを
     * まだ失敗させられる唯一のコールバックである。前のパッチの状態は
     * まだ引き継がないこと: そのコードはまだ実行中である。
     */
    return 0;
}

static void post_patch_v2(struct klp_object *obj)
{
    struct klp_state *prev, *cur;

    cur  = klp_get_state(&subsys_patch_v2, SUBSYS_LOCK_STATE_ID);
    prev = klp_get_prev_state(SUBSYS_LOCK_STATE_ID);

    /*
     * klp_get_prev_state()はここでもまだ機能する: klp_complete_transition()が
     * klp_transition_patchをクリアするのはpost_patchコールバックを実行した
     * *後*であり、置き換えられたパッチはそれが戻るまで解放されない。
     */
    if (prev)
        cur->data = prev->data;
}
```

`prev->data`をそのままコピーして安全なのは、前の例のエンコードされたフラグのように、
それが*値*を運んでいる場合だけである。古いパッチがポインタを格納していた場合は、
所有権を明示的に決着させなければならない。アトミックリプレースが完了すると、
古いパッチは無効化され、そのモジュールはアンロードできる。そのモジュール自身の
記憶領域を指す`state->data`はダングリングポインタになり、そのモジュールが割り当てた
メモリは、新しいパッチだけがまだ解放できる立場にあるリークになる。
これが、`system-state.rst`の`post_patch()`の残りの項目
（「Free *state->data* from replaces livepatches when they are not longer
needed」）が求めていることである。堅牢な形は、`pre_patch_v2()`で割り当て
（そこならまだ失敗でロードを中止できる）、`post_patch_v2()`で*内容*をコピーし、
そこで古いパッチの割り当てを解放することである。

メインの例と異なり、こちらは`post_unpatch_v2()`を必要とする。
`pre_patch_v2()`が割り当てを行い、その後`post_patch_v2()`が実行される前に
有効化が逆転された場合、その割り当てには他に解放する場所がない。
`system-state.rst`が`post_unpatch()`は「通常`pre_patch()`と対称的な操作を行う」と
述べているのはまさにこの理由による。`pre_patch_v2()`が割り当てたものはここで解放し、
エラー時に「自分の散らかしたものを片付ける」という`pre_patch_subsys()`の義務を
鏡写しにすること。

`version`フィールドは、カーネルが互換性を判断するのに使うものである。
それは自動的に、パッチ自身のコードが一切実行される前に行われる。パッチが
ロードされると、`klp_enable_patch()`は`klp_is_patch_compatible()`
（`kernel/livepatch/state.c`）を呼び、現在インストールされているすべてのパッチが
既に変更したすべての状態を走査する。そうした各状態について、新しいパッチも
その状態idを宣言しているなら、その`version`は既にインストールされている
バージョン以上（`>=`）でなければならない。新しいパッチがその状態idをまったく宣言していないなら、
新しいパッチが非累積の場合にのみ互換となる（累積の、`.replace =
true`のパッチは、置き換えるパッチが既に触れたすべての状態を考慮しなければならない）。
チェックに失敗すると、`klp_enable_patch()`は直ちに`-EINVAL`を返す。
上の`pre_patch_v2()`は呼ばれることすらなく、`insmod`はあっさり失敗する。
パッチ自身のコールバックはこの判断に関与できない。それらが実行されるのは、
カーネルが既に互換性を確認した後だけである。

## 状態の観察 {#observing-state}

`klp_state`のデータを公開するsysfsファイルはない。状態はパッチモジュールの
内部にある。遷移の進行状況を観察するには、標準的なsysfsファイルを使う。

```bash
# パッチはまだ遷移中か？
cat /sys/kernel/livepatch/<patch>/transition
# 1 = 進行中、0 = 完了

# 停滞した遷移を、一貫性チェックの残りを飛ばして直ちに終わらせる。
# この属性は書き込み専用であり、過去の遷移が強制されたかどうかを
# 読み出して知る方法はない。それは内部でのみ
# 追跡される（struct klp_patch.forced）。
echo 1 > /sys/kernel/livepatch/<patch>/force
```

遷移の強制は最後の手段である。すべてのタスクのペンディング中パッチフラグを
問答無用でクリアし、印を付けたパッチモジュールを永久に固定してしまう。
どのパッチに印が付くかは、何を強制したかによって決まる。
`klp_force_transition()`（`kernel/livepatch/transition.c`）が
`forced = true`をセットするのは次の場合である。

- *無効化*（unpatch）の遷移が強制されたときは、無効化されるパッチ。
- 累積（`.replace = true`）パッチの有効化が強制されたときは、インストール済みの
  **他の**すべてのパッチ（置き換えられるもの）。新しいパッチ自身には
  セットされない。
- 非累積パッチの有効化が強制されたときは、何にもセットされない。

その後`klp_free_patch_finish()`（`kernel/livepatch/core.c`）は、`forced`が
セットされたパッチについて`module_put()`を飛ばす。そのため、そのモジュールの参照
カウントは決して0に戻らず、そのモジュールに対する`rmmod`は稼働中のカーネルが
動いている間ずっと失敗する。

`pre_patch`コールバックがエラーを返すと、`insmod`は0でない
ステータスで終了し、`dmesg`には次のような行が含まれる。

```
livepatch: pre-patch callback failed for object 'vmlinux'
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/livepatch.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/livepatch.h) — `struct klp_state`、`struct klp_object`、`struct klp_patch`、および`klp_get_state()`/`klp_get_prev_state()`の宣言
- [include/linux/livepatch_external.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/livepatch_external.h) — `struct klp_callbacks`: `pre_patch`/`post_patch`/`pre_unpatch`/`post_unpatch`のフックテーブル
- [kernel/livepatch/state.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/livepatch/state.c) — `klp_get_state()`と`klp_get_prev_state()`の実装、およびパッチのロード時に実行される自動のバージョン互換性チェック（`klp_is_patch_compatible()`）
- [kernel/livepatch/core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/livepatch/core.c) — `klp_enable_patch()`: ftraceフックのインストール前に`pre_patch`が呼ばれる場所と、0でない戻り値が遷移を中断する仕組み

### 関連ページ {#related-pages}

- [KLP一貫性モデル](klp-consistency.md) — スタック走査、タスクごとの状態、強制遷移
- [累積パッチとアトミックリプレース](klp-cumulative.md) — `.replace=true`、文脈の中での`klp_get_prev_state()`
- [カーネルライブパッチ](klp.md) — `struct klp_func`、`struct klp_patch`、ftraceによるリダイレクト

### LWNの記事 {#lwn-articles}

- [LWN: Live patching for CPU vulnerabilities](https://lwn.net/Articles/775264/) — Nicolai StangeによるL1TF/KPTI向けライブパッチの解説。これらは遷移の途中でグローバルなページテーブルの意味を切り替える必要があり、まさにこのページで説明しているpre/post-patchコールバックのパターンを使った

### 外部資料 {#external}

- [System State Changes — The Linux Kernel documentation](https://docs.kernel.org/livepatch/system-state.html) — `klp_state` APIとそのバージョン互換性規則についてのアップストリームのドキュメント
- [(Un)patching Callbacks — The Linux Kernel documentation](https://docs.kernel.org/livepatch/callbacks.html) — `pre_patch`/`post_patch`/`pre_unpatch`/`post_unpatch`の意味と用途についてのアップストリームのドキュメント
