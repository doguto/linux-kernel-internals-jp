# カーネルライブパッチ（KLP） {#kernel-live-patching-klp}

> 稼働中のカーネルに、再起動せずにセキュリティ修正やバグ修正を適用する

## KLPが行うこと {#what-klp-does}

KLPは、ftraceの関数エントリフックを使って、元の（バグのある）関数への呼び出しを実行時に置き換え先の（パッチ適用済みの）関数へリダイレクトする。

```
Before patch:           After patch:
  caller                  caller
    │                       │
    call orig_func          call orig_func
                              │ (function entry hook)
                              └─► NEW: jump to patched_func
                                        │
                                        └─► executes patch
```

元の関数の本体が書き換えられることはなく、そこにジャンプ命令が挿入されることもない。KLPは、コンパイラが関数エントリに既に出力しているftraceの呼び出し箇所（`-mfentry`、arm64では`-fpatchable-function-entry`による）を再利用し、`klp_ftrace_handler()`はコードにパッチを当てるのではなく、保存された命令ポインタを変更することで実行をリダイレクトする。

## アーキテクチャ {#architecture}

```c
/* include/linux/livepatch.h */

/* パッチ対象の関数1つを記述する */
struct klp_func {
    /* 外部（パッチ作成者が設定） */
    const char       *old_name;     /* パッチ対象の関数名 */
    void             *new_func;     /* 置き換え先の関数 */
    unsigned long     old_sympos;   /* 何番目の出現か（重複名の場合） */

    /* 内部 */
    void             *old_func;     /* パッチ適用時にkallsyms経由で解決される */
    struct kobject    kobj;
    struct list_head  node;         /* klp_object->func_list用のリストノード */
    struct list_head  stack_node;   /* klp_ops->func_stack内での位置 */
    unsigned long     old_size, new_size;  /* 古い/新しい関数のサイズ */
    bool              nop;          /* 元のコードを再び使うための一時的なパッチ。
                                       アトミックリプレースによって動的に
                                       割り当てられる（klp-cumulative.md） */
    bool              patched;      /* 現在有効 */
    bool              transition;   /* 一貫性の遷移中 */
};

/* 1つの完全なパッチを構成する関数の集合 */
struct klp_patch {
    /* 外部 */
    struct module     *mod;         /* ライブパッチモジュール */
    struct klp_object *objs;        /* オブジェクト（モジュール/vmlinux）の配列 */
    struct klp_state  *states;      /* パッチが変更しうるシステム状態（klp-state.md） */
    bool               replace;     /* アトミックリプレース: 他のパッチを置き換える（klp-cumulative.md） */

    /* 内部 */
    struct list_head    list;
    struct kobject       kobj;
    struct list_head    obj_list;
    bool                 enabled;
    bool                 forced;    /* 強制遷移に関与した。
                                        モジュールは二度とアンロードできない */
    struct work_struct   free_work;
    struct completion    finish;
};

/* パッチ対象のカーネルモジュール（またはvmlinux）ごとにまとめた関数群 */
struct klp_object {
    /* 外部 */
    const char           *name;      /* vmlinuxの場合はNULL */
    struct klp_func       *funcs;    /* パッチ対象の関数の配列 */
    struct klp_callbacks  callbacks; /* pre/post-(un)patchフック */

    /* 内部 */
    struct kobject       kobj;
    struct list_head     func_list;  /* funcエントリの動的リスト */
    struct list_head     node;
    struct module        *mod;       /* 解決済みの対象モジュール */
    bool                  dynamic;   /* 動的に追加されたfunc用 */
    bool                  patched;
};
```

## ライブパッチモジュールを書く {#writing-a-live-patch-module}

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/livepatch.h>

/*
 * 置き換え先の関数。プロトタイプは元の関数と完全に一致しなければならない:
 * include/linux/fs.hでの宣言は
 *   ssize_t vfs_read(struct file *, char __user *, size_t, loff_t *);
 * これを間違えても何も警告されない。.new_funcはvoid *なので、戻り値の型が
 * 一致しなくても（ここではint）警告なしにコンパイルされ、パッチ適用後の
 * 関数が呼び出し元に返すすべての結果が切り詰められてしまう。
 */
static ssize_t patched_vfs_read(struct file *file, char __user *buf,
                                size_t count, loff_t *pos)
{
    /* バグを修正した新しい実装 */
    if (!file || !file->f_op)
        return -EBADF;   /* 以前はここでNULL参照していた */

    /*
     * ... fs/read_write.cからそのままコピーしたvfs_read()の本体の残り。
     * ここでは省略する。KLPにはklp_call_orig()はない。元の関数へ
     * 連鎖して戻るためのサポートされた方法はないため、パッチモジュールは
     * 本体全体を持ち、自身のコピーの中で修正を適用する。
     */
}

/* パッチ記述子 */
static struct klp_func funcs[] = {
    {
        .old_name = "vfs_read",
        .new_func = patched_vfs_read,
    },
    {}  /* 番兵 */
};

static struct klp_object objs[] = {
    {
        .name  = NULL,  /* NULL = vmlinuxにパッチを当てる */
        .funcs = funcs,
    },
    {}  /* 番兵 */
};

static struct klp_patch patch = {
    .mod  = THIS_MODULE,
    .objs = objs,
};

static int __init livepatch_init(void)
{
    return klp_enable_patch(&patch);
}

static void __exit livepatch_exit(void)
{
    /* KLPモジュールはデフォルトで居座る。無効化した後でのみアンロードできる */
}

module_init(livepatch_init);
module_exit(livepatch_exit);
MODULE_INFO(livepatch, "Y");
MODULE_LICENSE("GPL");
```

### 元の関数を呼び出す {#calling-the-original-function}

KLPのAPIは`klp_call_orig()`ヘルパーを提供していない。ライブパッチの置き換え関数は、
関数を完全に再実装するか、追加の状態を持たせるために`klp_shadow_*`を使う必要がある。
元のロジックが必要なら、それをパッチモジュールにコピーし、
狙った修正だけを適用すること。

## ftraceベースのリダイレクト {#ftrace-based-redirection}

KLPはパッチ対象の関数ごとに`FTRACE_OPS_FL_IPMODIFY`（命令ポインタを変更する許可）付きの
`ftrace_ops`を登録し、`klp_ftrace_handler()`は対象関数の`func_stack`の最上位にあるパッチへ
実行をリダイレクトする。

```c
/* kernel/livepatch/patch.c */
static void notrace klp_ftrace_handler(unsigned long ip,
                                        unsigned long parent_ip,
                                        struct ftrace_ops *fops,
                                        struct ftrace_regs *fregs)
{
    struct klp_ops *ops;
    struct klp_func *func;
    int patch_state;
    int bit;

    ops = container_of(fops, struct klp_ops, fops);

    /* 再帰ガード。これはプリエンプションも無効化する。これは
       kernel/livepatch/transition.cのklp_synchronize_transition()が
       必要とするものである。klp_synchronize_transition()は
       synchronize_rcu()の代わりとなるschedule_on_each_cpu()ベースの処理で、
       livepatchはRCUが監視していない関数（例: user_exit()の前）にも
       パッチを当てる必要があるため、RCU自体には頼れないことから使われる */
    bit = ftrace_test_recursion_trylock(ip, parent_ip);
    if (WARN_ON_ONCE(bit < 0))
        return;

    /* func_stackの先頭は、この関数に最後に適用されたパッチ */
    func = list_first_or_null_rcu(&ops->func_stack, struct klp_func, stack_node);
    if (WARN_ON_ONCE(!func))
        goto unlock;

    if (unlikely(func->transition)) {
        patch_state = current->patch_state;
        WARN_ON_ONCE(patch_state == KLP_TRANSITION_IDLE);

        if (patch_state == KLP_TRANSITION_UNPATCHED) {
            /*
             * このタスクはまだ遷移していない。スタック上に残っている
             * 次に古いパッチ（あれば）にフォールバックする。それより古い
             * パッチがなければ、元の関数へそのまま進む。
             */
            func = list_entry_rcu(func->stack_node.next,
                                   struct klp_func, stack_node);
            if (&func->stack_node == &ops->func_stack)
                goto unlock;
        }
    }

    /* NOPは元のコードを復元する。命令ポインタには触れない */
    if (func->nop)
        goto unlock;

    ftrace_regs_set_instruction_pointer(fregs, (unsigned long)func->new_func);

unlock:
    ftrace_test_recursion_unlock(bit);
}
```

`func_stack`はスタックなので、これはパッチの**積み重ね**の仕組みでもある。
パッチAの上にパッチBが適用され、あるタスクがまだBへ遷移していない場合、
ハンドラはBを飛ばして元の関数ではなくAの版の関数へ進む。
そのため、Bの遷移中もAによって既にパッチ適用された挙動が
維持される。

## 一貫性モデル {#the-consistency-model}

KLPは単純に呼び出しを即座にリダイレクトすることはできない。あるタスクが古い関数を実行している最中かもしれないからである。**一貫性モデル**は、パッチが有効とみなされる前にすべてのタスクが遷移を終えていることを保証する。

```
State: KLP_TRANSITION_UNPATCHED → (patch applied) → KLP_TRANSITION_PATCHED

For each task:
  1. On return to userspace, klp_update_patch_state() marks it KLP_TRANSITION_PATCHED
  2. Tasks that stay in the kernel are switched by the ~1s transition workqueue's
     stack check, or by klp_sched_try_switch() in __schedule() — which only fires
     when the outgoing task is entering a freezable (TASK_FREEZABLE) sleep, not at
     every schedule point
  3. Until all tasks are KLP_TRANSITION_PATCHED, the patch is in "transition" state

During transition:
  - New task entries: use new (patched) function
  - Tasks already running in old function: continue until they return
  - Patch isn't "complete" until all tasks are transitioned
```

```bash
# 遷移の状態を確認する
cat /sys/kernel/livepatch/mypatch/transition
# 1 = まだ遷移中（パッチ未適用のタスクが残っている）
# 0 = 完了

# 完了を強制する（一貫性チェックを飛ばす。安全でない可能性がある！）
echo 1 > /sys/kernel/livepatch/mypatch/force
```

### 眠り続けるタスク {#sleepy-tasks}

古い関数の中で`D`状態（`TASK_UNINTERRUPTIBLE`）でブロックしているタスクは遷移を遅らせ、livepatchコアはその待ち時間を短縮できない。`klp_send_signals()`が届くのは割り込み可能なスリープ中のタスクだけだからである（kthreadには`wake_up_state(task, TASK_INTERRUPTIBLE)`、ユーザータスクには`set_notify_signal()`。後者もそれ自体`TASK_INTERRUPTIBLE`の起床にすぎない）。D状態のタスクは自力でその状態を抜けるしかなく、定期的なworkqueueは抜けるまで再チェックを繰り返すだけである。[KLP一貫性モデル](klp-consistency.md)を参照。

```bash
# 遷移が停滞している場合: 何が妨げているかを探す
cat /sys/kernel/livepatch/mypatch/transition
# 1

# 古い関数がコールスタックにあるタスクを探す
ps aux | grep ' D '  # 割り込み不可能なスリープ中のタスク
cat /proc/<pid>/stack  # 古い関数内にいるか確認する
```

## shadow変数 {#shadow-variables}

shadow変数は、既存のカーネルオブジェクトの構造を変更せずに追加データを付加する。パッチが状態を追加する必要がある場合に有用である。

```c
/* 構造体を変更せずにstruct fileへ"magic"フィールドを追加する */
#define KLP_SHADOW_MAGIC 0xdeadbeef

struct my_shadow_data {
    u32 magic;
    int new_state;
};

/* 初回使用時: shadowデータを割り当てて付加する */
struct my_shadow_data *data;
data = klp_shadow_get_or_alloc(file,  /* このオブジェクトに付加する */
                                 KLP_SHADOW_MAGIC,
                                 sizeof(*data), GFP_KERNEL,
                                 shadow_init_fn, NULL);
data->new_state = 42;

/* 以降の使用時: 取得する */
data = klp_shadow_get(file, KLP_SHADOW_MAGIC);

/* 後始末時（例: ファイルのclose）: 解放する */
klp_shadow_free(file, KLP_SHADOW_MAGIC, NULL);
```

shadowデータは、（オブジェクトポインタ, ID）をキーとするグローバルなハッシュテーブルに格納される。

## ライブパッチのビルド {#building-a-live-patch}

```bash
# カーネルのビルドツリーが必要
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# あるいは: kpatch-buildを使う（ライブパッチの自動作成）
kpatch-build -t vmlinux fix.patch

# インストール
insmod mypatch.ko
lsmod | grep livepatch

# 有効になっていることを確認する
cat /sys/kernel/livepatch/mypatch/enabled
# 1
```

## ライブパッチの観察 {#observing-live-patches}

```bash
# 有効なすべてのパッチ
for p in /sys/kernel/livepatch/*/; do
    echo "Patch: $(basename $p)"
    echo "  enabled: $(cat $p/enabled)"
    echo "  transition: $(cat $p/transition)"
done

# どのオブジェクト（vmlinuxまたはモジュール）にパッチが当たっているか
cat /sys/kernel/livepatch/*/*/patched
# 注: 関数ごとのディレクトリ（<object>/<function,sympos>/）は存在するが、
# 現在のところ独自の属性は何も公開していない。

# ライブパッチによるカーネルのtaint
cat /proc/sys/kernel/tainted
# ビット15がセットされる（TAINT_LIVEPATCH: KLPパッチ適用済み）

# dmesgで確認する
dmesg | grep livepatch
# livepatch: enabling patch 'mypatch'
# livepatch: 'mypatch': starting patching transition
# livepatch: 'mypatch': patching complete
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/livepatch.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/livepatch.h) — `struct klp_func`、`struct klp_object`、`struct klp_patch`、および`klp_shadow_*()`のプロトタイプ
- [kernel/livepatch/patch.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/livepatch/patch.c) — `klp_ftrace_handler()`、ftrace opsの登録（`FTRACE_OPS_FL_IPMODIFY`）、func-stackによるリダイレクトのロジック
- [kernel/livepatch/transition.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/livepatch/transition.c) — 一貫性モデルのステートマシン、`klp_try_complete_transition()`、`klp_send_signals()`
- [kernel/livepatch/shadow.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/livepatch/shadow.c) — shadow変数のハッシュテーブルと`klp_shadow_alloc()`/`klp_shadow_get()`/`klp_shadow_free()`の実装
- [samples/livepatch/livepatch-sample.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/samples/livepatch/livepatch-sample.c) — 標準的な最小のライブパッチモジュール（`cmdline_proc_show`にパッチを当てる）
- [Documentation/ABI/testing/sysfs-kernel-livepatch](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/ABI/testing/sysfs-kernel-livepatch) — `/sys/kernel/livepatch/`の属性の正式な一覧（`enabled`、`transition`、`force`、`replace`、`stack_order`、およびオブジェクトごとの`patched`）

### 関連ページ {#related-pages}

- [KLP一貫性モデル](klp-consistency.md) — タスクごとのパッチ状態、スタック検査、遷移workqueueの詳細
- [累積パッチとアトミックリプレース](klp-cumulative.md) — パッチの積み重ね、`.replace`、`struct klp_ops`
- [KLP State: 独自の一貫性チェック](klp-state.md) — スタック走査を超えた一貫性チェックのための`klp_state` API
- [kexec](kexec.md) — 新しいカーネルのロードと起動。KLPで表現できない変更に対するライブパッチの代替手段
- [カーネルモジュール](../modules/module-basics.md) — KLPパッチは通常のカーネルモジュールとして配布・ロードされる
- [トレーシング: ftrace](../tracing/ftrace.md) — KLPが基盤とする関数エントリフックの機構

### LWNの記事 {#lwn-articles}

- [Kernel Live Patching](https://lwn.net/Articles/619390/) — Seth Jenningsによる、ftraceベースのコアについての2014年の最初の解説。マージ前の`lp_patch`ベースの設計（マージ前に`klp_`へ改名）と、元のsysfsインターフェース
- [livepatch: hybrid consistency model](https://lwn.net/Articles/685464/) — タスクごとの遷移、スタックの信頼性チェック、`TIF_PATCH_PENDING`を導入したJosh Poimboeufの2016年のシリーズ
- [livepatch: introduce shadow variable API](https://lwn.net/Articles/731585/) — `klp_shadow_alloc()`/`klp_shadow_get()`/`klp_shadow_free()`を導入したJoe Lawrenceの2017年のパッチ

### 外部資料 {#external}

- [Livepatch — The Linux Kernel documentation](https://docs.kernel.org/livepatch/livepatch.html) — 公式ドキュメント: 動機、一貫性モデル、モジュールのライフサイクル、既知の制限
- [Shadow Variables — The Linux Kernel documentation](https://docs.kernel.org/livepatch/shadow-vars.html) — shadow変数APIの公式リファレンスと使用上の注意
