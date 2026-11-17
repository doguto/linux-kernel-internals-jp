# 累積パッチとアトミックリプレース {#cumulative-patches-and-atomic-replace}

> ライブパッチを積み重ね、それらをアトミックに置き換える

## 蓄積の問題 {#the-accumulation-problem}

長期間稼働する本番環境では、カーネルに時間とともに複数のライブパッチが蓄積することがある。
パッチP1は`tcp_sendmsg`のバグを修正する。2週間後、パッチP2は別の関数`ip_output`を修正する。
その1か月後、パッチP3は`vfs_write`を修正する。

各パッチは独立しているが、すべて同時にロードされたままでなければならない。
システムは今や3つのパッチモジュール、3つの遷移の履歴を抱え、どの関数がどのモジュールによって
パッチされているかを追跡する認知的なオーバーヘッドを負う。
`tcp_sendmsg`を再び更新する4つ目のセキュリティパッチが届くと、事態は複雑になる。
P4はP1も有効であることを前提に書かなければならない。

*累積パッチ*はこれを解決する。累積パッチは、以前のパッチの上に重ねるのではなく、
単一のアトミックな操作で以前のすべてのパッチを置き換える。

## struct klp_patch: .replaceフラグ {#struct-klp_patch-the-replace-flag}

Linux 5.1で`struct klp_patch`に追加された`.replace`フィールドは、
パッチを累積パッチとしてマークする。

```c
/* include/linux/livepatch.h */
struct klp_patch {
    /* 外部（パッチモジュールの作者が設定する）: */
    struct module       *mod;       /* ライブパッチモジュール */
    struct klp_object   *objs;      /* パッチ対象オブジェクトの配列 */
    struct klp_state    *states;    /* 任意の一貫性状態（5.5以降） */
    bool                 replace;   /* true = 累積的な置き換え（5.1以降） */

    /* 内部（livepatchコアが管理する）: */
    struct list_head     list;
    struct kobject       kobj;
    struct list_head     obj_list;
    bool                 enabled;
    bool                 forced;
    struct work_struct   free_work;
    struct completion    finish;
};
```

累積パッチは、適用後にパッチされているべきすべての関数を宣言する。
以前のパッチが触れた関数も含めてである。
`.replace = true`のとき、KLPコアはパッチを1つ追加するだけでなく、
有効なパッチの集合全体を置き換える。

## 累積パッチを書く {#writing-a-cumulative-patch}

```c
/* 累積パッチ: P1（tcp_sendmsg）とP2（ip_output）を置き換える */

static int patched_tcp_sendmsg(struct sock *sk, struct msghdr *msg,
                                size_t size)
{
    /* 最新の修正: P1の修正と新たな変更を組み合わせる */
    ...
}

static int patched_ip_output(struct net *net, struct sock *sk,
                              struct sk_buff *skb)
{
    /* P2の修正を再実装する */
    ...
}

static struct klp_func net_funcs[] = {
    {
        .old_name = "tcp_sendmsg",
        .new_func = patched_tcp_sendmsg,
    },
    {
        .old_name = "ip_output",
        .new_func = patched_ip_output,
    },
    {}
};

static struct klp_object objs[] = {
    { .name = NULL, .funcs = net_funcs },
    {}
};

static struct klp_patch cumulative_patch = {
    .mod     = THIS_MODULE,
    .objs    = objs,
    .replace = true,    /* <-- これが累積パッチである */
};

static int __init cumulative_patch_init(void)
{
    return klp_enable_patch(&cumulative_patch);
}
```

`klp_atomic_replace()`という関数は存在しない。累積的な置き換えの仕組みは次のとおりである。

`patch->replace == true`のとき、`klp_init_patch()`は`klp_add_nops(patch)`を呼び、現在有効なパッチがカバーしているが新しい累積パッチが明示的にパッチしていないすべての関数について、`klp_func`の「nop」エントリを動的に割り当てる。これらのnopエントリは、有効なときには元の関数へそのまま呼び出しを通す。これにより、遷移が完了して古いパッチがfunc_stackから取り除かれたとき、以前パッチされていたすべての関数が引き続き（nopによって）カバーされることが保証される。

「アトミック」なのは、新しいパッチの有効化と古いパッチの削除という切り替え全体が、ユーザー空間から見てアトミックに起こるという点である。古いパッチは、遷移が終わった後に`klp_complete_transition()`の中で取り除かれる。

```c
/* 累積的な置き換えの流れ（簡略化）: */

/* 1. パッチ作成者がklp_patchで.replace = trueを設定する */
/* 2. klp_enable_patch() → klp_init_patch() → klp_add_nops()
 *    が、有効なパッチに含まれ新しいパッチがカバーしていない
 *    関数についてnop funcを割り当てる */
/* 3. 通常の遷移が始まる（他のパッチと同じ） */
/* 4. klp_complete_transition() → klp_unpatch_replaced_patches()
 *    → klp_unpatch_objects()（古いパッチごと）が、置き換えられた
 *    すべてのパッチをfunc_stackから取り除く */
```

## struct klp_ops: 関数ごとのフック {#struct-klp_ops-the-per-function-hook}

`struct klp_ops`は、その関数を対象とするパッチがいくつあっても、
（オブジェクト, 関数名）の組ごとにちょうど1つ存在する。

```c
/* kernel/livepatch/patch.h */
struct klp_ops {
    struct list_head  node;        /* グローバルなklp_opsリスト */
    struct list_head  func_stack;  /* klp_funcエントリ。最新のものが先頭 */
    struct ftrace_ops fops;        /* 単一のftrace登録 */
};
```

同じ関数に触れるすべてのパッチは1つの`klp_ops`を共有し、したがって1つの
ftraceフックを共有する。`func_stack`が裁定者であり、先頭のエントリが有効な
置き換え先である。新しいパッチの`klp_func`は、有効化されるときに常に先頭へ
プッシュされる。

```c
/* kernel/livepatch/patch.c — klp_patch_func() */
list_add_rcu(&func->stack_node, &ops->func_stack);
```

そして無効化されるときには、先頭（あるいはどこにあっても）から取り除かれる。

```c
/* kernel/livepatch/patch.c — klp_unpatch_func() */
list_del_rcu(&func->stack_node);
```

スタックが空になると、ftraceフックは登録解除され、元の関数は
トランポリンのオーバーヘッドなしに再び実行されるようになる。

## パッチの無効化 {#disabling-a-patch}

パッチを無効化すると*逆方向の遷移*が始まる。タスクは
`KLP_TRANSITION_PATCHED`から`KLP_TRANSITION_UNPATCHED`へ戻される。sysfsの`enabled`ファイルがこれを起動する。

```bash
echo 0 > /sys/kernel/livepatch/<patch>/enabled
```

カーネルは`klp_target_state = KLP_TRANSITION_UNPATCHED`を設定し、
`klp_transition_work`をキューに入れる。スタック検査は引き続き適用される。*新しい*（パッチ適用済みの）関数の
内部を実行中のタスクは、そこから戻るまで逆遷移できない。

無効化の遷移を観察する:

```bash
cat /sys/kernel/livepatch/mypatch/transition
# 1 = 逆方向の遷移が進行中

# 完了を待つ
while [ "$(cat /sys/kernel/livepatch/mypatch/transition)" = "1" ]; do
    sleep 1
done

cat /sys/kernel/livepatch/mypatch/enabled
# 0 = 完全に無効化された
```

## パッチの削除 {#removing-a-patch}

パッチが無効化されると、そのモジュールはアンロードできる。`klp_enable_patch()`は
livepatchモジュールが呼ぶ主要なエントリポイントだが、`EXPORT_SYMBOL_GPL`で
エクスポートされる公開livepatch APIはそれだけではない。`core.c`は
`klp_find_section_by_name()`も、`shadow.c`は`klp_shadow_get()`、
`klp_shadow_alloc()`、`klp_shadow_get_or_alloc()`、`klp_shadow_free()`、
`klp_shadow_free_all()`を、`state.c`は`klp_get_state()`と
`klp_get_prev_state()`をエクスポートしている。livepatchモジュールの`module_exit`は
空にするか完全に省略すべきである。後始末はパッチが無効化された時点で既に済んでいる。
`klp_complete_transition()`が`klp_unpatch_objects()`を呼び、これが各関数を
その`func_stack`から取り出し、スタックが空になったらftraceフックを登録解除する。
その後`klp_free_patch_async()`が`struct klp_patch`を解放し、
モジュール参照を手放す。`rmmod`が実行される時点では、
後始末すべきものは何も残っていない。

（`klp_module_going()`は、モジュールのロード/アンロード中に`kernel/module/main.c`から
モジュール通知チェーン経由ではなく直接呼ばれるもので、別の仕組みである。
これは、それ自体がアンロードされつつある*別の*対象モジュールに適用されたパッチを
元に戻すものであり、livepatchモジュール自身の後始末ではない。）

```c
/* livepatchモジュールのexit: 明示的な登録解除は不要。パッチが下記の方法で
 * 無効化されたとき、遷移が既にftraceフックを取り外し、パッチを解放している。
 * アンロードする前に、sysfs経由でパッチを無効化する:
 *   echo 0 > /sys/kernel/livepatch/<patch>/enabled
 * その後: rmmod <patch_module>
 */
static void __exit livepatch_exit(void) { }
module_exit(livepatch_exit);
```

ライブパッチモジュールは有効な間はアンロードできない。モジュールローダが
これをチェックする。

```bash
# パッチがまだ有効なら、これは失敗する:
rmmod mypatch
# ERROR: Module mypatch is in use

# 正しい手順:
echo 0 > /sys/kernel/livepatch/mypatch/enabled
# transition=0になるのを待つ
rmmod mypatch
```

## klp_patchのライフサイクル {#klp_patch-lifecycle}

`struct klp_patch`には状態を表すenumはない。状態はブール型のフィールド、`patch->enabled`（bool）と`patch->forced`（bool）によって追跡される。sysfsから観察できる状態で表すと、ライフサイクルは次のとおりである: 未ロード → `enabled=0`（ロード済み、まだ有効でない） → `enabled=1, transition=1`（遷移中） → `enabled=1, transition=0`（完全に有効） → `enabled=0`（無効化済み、アンロード可能）。

```
insmod mypatch.ko
  │
  ▼
klp_enable_patch()
  │  (registration, symbol resolution, ftrace hook install)
  ▼
enabled=1, transition=1 (transition in progress -- enabled is set
                          at the START of the transition, not the end)
  │
  ▼
enabled=1, transition=0 (fully active)
  │
  │  echo 0 > enabled
  ▼
enabled=0, transition=1 (reverse transition in progress)
  │
  ▼
enabled=0, transition=0 (disabled, can unload)
  │
  │  rmmod mypatch.ko
  ▼
(patch module unloaded; ftrace hooks were already removed when the patch was disabled)
```

## func_stackの観察 {#observing-the-func_stack}

sysfsの階層は、パッチ対象の各オブジェクトの状態を公開し、パッチ対象の各関数の
ディレクトリを列挙するが、関数ごとの`patched`ファイルは存在しない。
`patched`はオブジェクトレベルの属性のみである。

```bash
# vmlinuxオブジェクト（コアカーネル）がmypatchによってパッチされているか確認する
cat /sys/kernel/livepatch/mypatch/vmlinux/patched
# 1 = このオブジェクト内のこのパッチの関数が現在フックされている

# パッチ対象の各関数は、オブジェクトの下に"<function>,<sympos>"という
# 名前のディレクトリとして現れる。存在すること自体がシグナルであり、
# 独自の属性ファイルは持たない:
ls /sys/kernel/livepatch/mypatch/vmlinux/
# patched  tcp_sendmsg,1

# old_addrはsysfsでは公開されていない
# 元の関数のアドレスを知るには/proc/kallsymsを使う:
grep " tcp_sendmsg$" /proc/kallsyms

# 2つのパッチが同じ関数をカバーしている場合、裁定するのは（sysfsではなく）
# func_stackの順序である。各パッチの位置は"stack_order"を参照:
cat /sys/kernel/livepatch/p1/stack_order  # 1（古い。p2に覆い隠されている）
cat /sys/kernel/livepatch/p2/stack_order  # 2（新しい。有効）
```

## kpatchを使った実践的なワークフロー {#practical-workflow-with-kpatch}

`kpatch`は、累積ライブパッチのビルドと配布に最も広く使われているツールチェーンである。
セキュリティパッチをリリースする際の標準的なワークフローは次のとおり。

```bash
# 1. ソースのdiffから累積パッチをビルドする
#    kpatch-buildは、diffが触れるすべての関数にパッチを当てる.koを作成する
kpatch-build -t vmlinux security-fix.patch

# 2. 生成されたモジュールはdiffのすべての関数を含む
#    これがp1、p2に続く3つ目のパッチなら、モジュール内で.replace=trueを設定する

# 3. 累積パッチをロードする
insmod kpatch-security-fix.ko

# 4. 以前のパッチを置き換えたことを確認する
cat /sys/kernel/livepatch/kpatch-security-fix/transition
# 0になるのを待つ

cat /sys/kernel/livepatch/kpatch-security-fix/enabled
# 1

# 以前のパッチは無効化され、そのモジュールはアンロードできる:
echo 0 > /sys/kernel/livepatch/kpatch-prev-patch/enabled
# 待ってからrmmodする

# 5. リブート後も維持する（kpatchサービスを使う）
kpatch install kpatch-security-fix.ko
systemctl enable kpatch
```

## 累積パッチの順序規則 {#cumulative-patch-ordering-rules}

最も重要な順序規則は、オペレータではなくカーネルによって強制される。
一度に進行中にできる遷移は1つだけであり、有効化と無効化の経路はどちらも、
既に遷移が進行中であれば早期に処理を打ち切る。

```c
/* kernel/livepatch/core.c — __klp_enable_patch()と__klp_disable_patch()は
 * どちらも同じガードで早期に打ち切る */
if (klp_transition_patch)
        return -EBUSY;
```

したがって、別のパッチがまだ遷移中に累積パッチをロードしても、
`func_stack`が壊れることはない。新しいモジュールの`init`から呼ばれる`klp_enable_patch()`が
`-EBUSY`を返し、`insmod`は失敗し、モジュールはロードされない。同じガードは、
あるパッチが遷移中のときに*別の*パッチに向けた`echo 0 > .../enabled`も拒否する。
ただし遷移中のパッチ自身に向けたものは拒否しない。`enabled_store()`はそのケースを
`klp_reverse_transition()`へ回し、書き込みを拒否するのではなく、進行中の遷移を
意図的に逆転させる（[KLP State](klp-state.md)を参照。このページはまさにこの挙動に依存して
コールバックを再入可能にしている）。事前にチェックしても、本当に別のパッチに対する
デプロイ手順の失敗を1つ省けるだけである。

```bash
# 無用なEBUSYを避ける: 進行中の遷移が終わるのを待つ
for p in /sys/kernel/livepatch/*/; do
    t=$(cat "$p/transition")
    if [ "$t" = "1" ]; then
        echo "$(basename $p) is still transitioning — insmod would fail with EBUSY"
    fi
done
```

カーネルが強制*しない*規則は、累積パッチのつもりのパッチが実際に累積パッチであるか
どうかである。`klp_add_nops()`は`if (patch->replace)`の下でのみ実行され、
`klp_unpatch_replaced_patches()`は`klp_transition_patch->replace`がセットされている
ときにのみ実行される。そのため`.replace = true`を付け忘れたパッチは、
置き換えるはずだったパッチの上に黙って積み重なる。そして、それらのパッチが触れたが
新しいパッチが名指ししていない関数は、古い置き換え関数を実行し続ける。
デプロイのたびに結果を確認すること。

```bash
cat /sys/kernel/livepatch/mypatch/replace       # 1であるべき
cat /sys/kernel/livepatch/mypatch/stack_order   # 1であるべき。置き換えられたパッチは
                                                # klp_patchesから取り除かれる
ls /sys/kernel/livepatch/                       # mypatchだけであるべき
```

累積パッチが`.replace`なしで出荷され、置き換えられたはずの修正が元に戻らなかった
実際の事例については[事件簿](war-stories.md)を参照。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/livepatch.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/livepatch.h) — `struct klp_patch`、`struct klp_func`、`struct klp_object`、およびタスク状態の定数`KLP_TRANSITION_IDLE`/`KLP_TRANSITION_UNPATCHED`/`KLP_TRANSITION_PATCHED`
- [kernel/livepatch/core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/livepatch/core.c) — `klp_enable_patch()`、`klp_add_nops()`、`klp_unpatch_replaced_patches()`、`klp_module_going()`（アンロードされる対象モジュールへのパッチを元に戻す）、`__klp_enable_patch()`/`__klp_disable_patch()`が共有する`if (klp_transition_patch) return -EBUSY;`のガード、そして`/sys/kernel/livepatch/...`のsysfsレイアウト全体を説明するコメントブロック
- [kernel/livepatch/patch.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/livepatch/patch.c) — `klp_patch_func()`/`klp_unpatch_func()`: `list_add_rcu()`/`list_del_rcu()`による`ops->func_stack`への`klp_func`エントリのプッシュとポップ
- [kernel/livepatch/patch.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/livepatch/patch.h) — `struct klp_ops`の定義（`node`、`func_stack`、`fops`）
- [Documentation/livepatch/cumulative-patches.rst](https://docs.kernel.org/livepatch/cumulative-patches.html) — アトミックリプレースについてのアップストリームの利用ガイド。コールバックとshadow変数に関する制限も含む
- [`e1452b607c48` — livepatch: Add atomic replace](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e1452b607c48c642caf57299f4da83aa002f8533) — `struct klp_patch`に`.replace`フィールドを追加したJason Baronのコミット（Petr Mladekが手直し・分割）。Linux 5.1でマージされた

### manページ {#man-pages}

- [`rmmod(8)`](https://man7.org/linux/man-pages/man8/rmmod.8.html) — モジュールの削除。`-f`/`--force`オプションは、このページが危険だと警告している使用中の参照カウントのチェックを迂回する。この危険は、使用中のあらゆるモジュールと同様に、まだ有効なライブパッチにも当てはまる

### 関連ページ {#related-pages}

- [KLP一貫性モデル](klp-consistency.md) — タスクごとの状態、スタック検査、強制遷移
- [カーネルライブパッチ](klp.md) — struct klp_func/klp_patch、ftraceによるリダイレクト、shadow変数
- [KLP State](klp-state.md) — `struct klp_patch`の`.states`フィールドが参照する`struct klp_state`/`klp_get_state()` API
- [カーネルモジュール](../modules/module-basics.md) — KLPモジュールは標準的なモジュール基盤を使う
- [事件簿](war-stories.md) — `.replace = true`が欠けた「累積」パッチを出荷したことによる実際の事例

### LWNの記事 {#lwn-articles}

- [livepatch: Atomic replace feature](https://lwn.net/Articles/776290/) — アトミックリプレースを導入したPetr Mladekのv15パッチシリーズ（LWNのミラー、2019年1月9日）。カバーレターでは累積パッチの設計と、それが解消した問題（パッチの順序の依存関係、積み重ねたパッチを1つだけ元に戻す方法がないこと）を説明している
