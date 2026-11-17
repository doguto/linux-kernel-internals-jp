# RCU（Read-Copy-Update） {#rcu-read-copy-update}

> グレースピリオドによるロックフリーな読み込み。カーネルで最も高性能な同期

## 基本的な考え方 {#the-core-idea}

RCUは、Dipankar Sarma（IBM）とPaul E. McKenneyによってカーネル2.5.43（2002年10月）でLinuxに追加された。これは["What is RCU, Fundamentally?"](https://lwn.net/Articles/262464/)（McKenney & Walpole、LWN 2007）で確認できる。*"RCU added to the Linux kernel in October 2002."*（訳: RCUは2002年10月にLinuxカーネルに追加された。）

RCUは特定の問題を解決する。書き手が安全に変更できるようにしつつ、多数の読み手が同期のオーバーヘッドゼロでデータ構造にアクセスできるようにするにはどうすればよいか、という問題である。

洞察は次のとおりである。**読み手はデータを決して変更しない**。したがって、古いデータを見た可能性のあるすべての読み手が終了するまで書き手が古いデータを解放しない限り、読み手は真に並行して実行できる。

```
Writer algorithm:
  1. Make a copy of the data
  2. Modify the copy
  3. Atomically publish the new version (pointer swap)
  4. Wait for a "grace period" — until all readers that could
     have seen the old version have finished
  5. Free the old version

Reader algorithm:
  1. Enter RCU read-side critical section (very cheap)
  2. Dereference the published pointer
  3. Use the data
  4. Exit RCU read-side critical section
```

決定的な性質は、**読み手がほとんど何も支払わない**ことである。プリエンプト不可のカーネルでは、`rcu_read_lock()`は文字どおり、プリエンプションを無効化した状態でのコンパイラバリアにすぎない。アトミック操作もキャッシュラインのバウンスもない。

## グレースピリオド {#the-grace-period}

**グレースピリオド**とは、書き手が新しいポインタを公開した後、古いものを解放するまでに待つ時間である。グレースピリオドは、すべてのCPUが**静止状態**（RCU読み側クリティカルセクションが進行中ではありえない地点）を通過したときに終わる。

静止状態には次のものがある。

- コンテキストスイッチ（CPUがプリエンプトされた）
- アイドル（CPUに仕事がない）
- ユーザーモードでの実行

```
CPU 0: [reader using old data]──────────────────[end read]
CPU 1:                         [context switch]
CPU 2:                                    [context switch]
                         ↑                           ↑
                   grace period starts          grace period ends
                                                (old data safe to free)
```

プリエンプト不可のカーネルでは、すべてのCPUが少なくとも1回スケジュールを行った時点でグレースピリオドが完了する。これが、コンテキストスイッチがRCUの静止状態である理由である。

## API {#api}

```c
#include <linux/rcupdate.h>

/* 読み側: クリティカルセクションに入る/出る */
rcu_read_lock();
/* RCUで保護されたデータにアクセスする。スリープしてはならない（!PREEMPT_RCUの場合）。 */
rcu_read_unlock();

/* RCUで保護されたポインタを読む */
struct my_data *p = rcu_dereference(rcu_ptr);
/* rcu_dereferenceはデータ依存バリアを追加する — 必須！ */

/* 新しいポインタを公開する（代入の前にメモリバリアを発行する） */
rcu_assign_pointer(rcu_ptr, new_data);

/* グレースピリオドの完了を待つ（ブロックする） */
synchronize_rcu();

/* グレースピリオド後に実行するコールバックを登録する（ブロックしない） */
call_rcu(&old_data->rcu_head, my_free_callback);

/* コールバック内で: */
static void my_free_callback(struct rcu_head *head)
{
    struct my_data *data = container_of(head, struct my_data, rcu_head);
    kfree(data);
}
```

### rcu_headの埋め込み {#embedding-rcu_head}

RCU経由で解放されるオブジェクトは、すべて`struct rcu_head`を埋め込まなければならない。

```c
struct my_data {
    int value;
    char name[32];
    struct rcu_head rcu;  /* call_rcu / kfree_rcu用 */
};

/* 割り当てて公開する */
struct my_data *new_data = kmalloc(sizeof(*new_data), GFP_KERNEL);
new_data->value = 42;
rcu_assign_pointer(global_ptr, new_data);

/* 取り除いて解放する（書き手側） */
struct my_data *old = rcu_dereference_protected(global_ptr,
                                                 lockdep_is_held(&my_lock));
rcu_assign_pointer(global_ptr, NULL);
synchronize_rcu();
kfree(old);

/* あるいはcall_rcuで（非同期） */
call_rcu(&old->rcu, my_free_callback);
```

### kfree_rcuによる省略形 {#kfree_rcu-shorthand}

グレースピリオド後に`kfree()`するだけでよいオブジェクトには、`kfree_rcu()`という省略形があり、コールバックを書かずに済む。

```c
/* call_rcu + kfreeを呼ぶコールバックの代わりに: */
kfree_rcu(old_data, rcu);  /* 'rcu'はrcu_headフィールドの名前 */

/* 引数なしの形式（Linux 5.9以降）: rcu_headが不要 */
kfree_rcu(old_data);
```

## 完全な例: RCUで保護されたルックアップテーブル {#a-complete-example-rcu-protected-lookup-table}

```c
/* グローバルなポインタ。更新はまれ */
static struct config __rcu *global_config;
static DEFINE_SPINLOCK(config_lock);  /* 書き手を保護する */

/* 読み手: 頻繁に呼ばれ、高速でなければならない */
int get_timeout(void)
{
    int timeout;
    rcu_read_lock();
    timeout = rcu_dereference(global_config)->timeout;
    rcu_read_unlock();
    return timeout;
}

/* 書き手: まれに呼ばれる */
int update_config(int new_timeout)
{
    struct config *new_cfg, *old_cfg;

    new_cfg = kmalloc(sizeof(*new_cfg), GFP_KERNEL);
    if (!new_cfg)
        return -ENOMEM;

    spin_lock(&config_lock);
    old_cfg = rcu_dereference_protected(global_config,
                                         lockdep_is_held(&config_lock));
    new_cfg->timeout = new_timeout;
    rcu_assign_pointer(global_config, new_cfg);
    spin_unlock(&config_lock);

    /* どの読み手もold_cfgを見られなくなるまで待ってから解放する */
    synchronize_rcu();
    kfree(old_cfg);
    return 0;
}
```

## RCUで保護されたリスト {#rcu-protected-lists}

カーネルは、リンクリスト用に`rculist.h`を提供している。

```c
#include <linux/rculist.h>

/* 書き手: リストに追加する（リストのロックを保持していなければならない） */
spin_lock(&list_lock);
list_add_rcu(&new->list, &my_list);
spin_unlock(&list_lock);

/* 書き手: リストから削除する */
spin_lock(&list_lock);
list_del_rcu(&entry->list);
spin_unlock(&list_lock);
synchronize_rcu();  /* 解放する前に待つ */
kfree(entry);

/* 読み手: リストを走査する */
rcu_read_lock();
list_for_each_entry_rcu(entry, &my_list, list) {
    /* entryを使う */
}
rcu_read_unlock();
```

## RCUのフレーバー {#rcu-flavors}

コンテキストに応じて、主に3つのRCUフレーバーがある。

| フレーバー | 読み側のロック | 静止状態 | 使う場面 |
|--------|---------------|-----------------|----------|
| `rcu` | `rcu_read_lock()` | コンテキストスイッチ、アイドル、ユーザーモード | ほとんどのカーネルコード |
| `rcu_bh` | `rcu_read_lock_bh()` | BHが無効な任意の領域 | レガシー、BHコンテキスト |
| `rcu_sched` | `rcu_read_lock_sched()` | プリエンプション無効の領域 | スケジューラの内部 |

Linux 5.0以降、3つはすべて統合されている。`synchronize_rcu()`は3つすべてを同時に待つ。

## SRCU: スリープ可能なRCU {#srcu-sleepable-rcu}

標準のRCUの読み側クリティカルセクションはスリープできない。SRCU（Sleepable RCU）は、読み側のオーバーヘッドが高くなる代わりにこの制限を取り払う。

```c
#include <linux/srcu.h>

DEFINE_SRCU(my_srcu);  /* またはDEFINE_STATIC_SRCU */

/* 読み手（スリープしうる） */
int idx = srcu_read_lock(&my_srcu);
/* ... ここでschedule()を呼んでもよい ... */
srcu_read_unlock(&my_srcu, idx);

/* 書き手 */
synchronize_srcu(&my_srcu);
```

SRCUは、notifierチェーン、ファイルシステムの操作、BPFプログラムの呼び出しなど、読み手がスリープする必要がある場面で使われる。

## RCUを使うべき場面 {#when-to-use-rcu}

```
RCUが理想的な場合:
  ✓ 読み込みが書き込みよりはるかに頻繁である
  ✓ 読み側の性能が重要である（読み込みパスにアトミック操作がない）
  ✓ データがポインタ経由でアクセスされる
  ✓ 書き手がsynchronize_rcu()やcall_rcu()のコストを許容できる

RCUが向かない場合:
  ✗ 書き込み中心のワークロード（synchronize_rcuは高コスト）
  ✗ その場で更新されるデータ（ポインタベースでない）
  ✗ 小さなCPUごとのデータ（代わりにper-CPU変数を使う）
```

## Tree RCU: グレースピリオド検出の仕組み {#tree-rcu-grace-period-detection-mechanism}

実際のシステムでは、`synchronize_rcu()`が単純に「すべてのCPUがコンテキストスイッチするのを待つ」わけにはいかない。負荷のかかったシステムでは呼び出し側を無期限にブロックしてしまうからである。Tree RCUは、効率的な階層型アルゴリズムでこれを解決する。

```
Tree RCU node hierarchy (for 16 CPUs, FANOUT=8):
          [root rcu_node]
         /               \
   [node 0-7]         [node 8-15]
   /  |  |  \  \       ...
 CPU0 CPU1 ... CPU7

Each node tracks:
  qsmask:   bitmask of children that haven't reported QS yet
  qsmaskinit: initial value at grace period start
```

### グレースピリオドの段階 {#grace-period-phases}

```c
/* kernel/rcu/tree.c */

/* 段階1: RCU GP kthreadが新しいグレースピリオドを開始する */
static void rcu_gp_init(void)
{
    /* オンラインのCPUに基づいて全ノードのqsmaskinitを設定する */
    /* GPシーケンス番号を進め、静止状態を報告させるためにCPUを起こす */
}

/* 段階2: 各CPUが自身の静止状態を報告する */
/* コンテキストスイッチ時: */
void rcu_note_context_switch(bool preempt)
{
    /* 親ノードに静止状態を報告する */
    rcu_qs();
}

/* 段階3: ルートのqsmaskの全ビットがクリアされたら、GPが終わる */
static void rcu_gp_cleanup(void)
{
    /* call_rcu()で登録されたすべてのコールバックを呼び出す */
    /* グレースピリオドのシーケンスカウンタを進める */
    rcu_advance_cbs(rdp);
}
```

### 緊急グレースピリオド（expedited grace period） {#expedited-grace-periods}

`synchronize_rcu_expedited()`は、すべてのCPUにIPIを送って即座に静止状態を報告させることで、高速なグレースピリオドを強制する。

```c
/* 通常のGPの約10msに対し、約100µsで完了する */
synchronize_rcu_expedited();
/* 控えめに使うこと: IPIの嵐によるオーバーヘッドがあり、リアルタイムのレイテンシを損なう */

/* カーネルは自動的に次を切り替える:
   - 通常のGP: 低オーバーヘッド。急いで待っている者がいないときに使われる
   - 緊急GP: synchronize_rcu()の呼び出し側がしきい値を超えて溜まった後 */
```

### Tasks RCU {#tasks-rcu}

「タスク」（非アトミックでスリープ可能なコンテキスト）に対して、Tasks RCUは静止状態を別の方法で追跡する。

```c
/* Tasks RCU: BPFプログラムの更新、トランポリンのコードパッチに使われる */
synchronize_rcu_tasks();   /* 古いトランポリン内にいるタスクがなくなるまで待つ */
call_rcu_tasks(&rcu_head, callback);

/* Tasks RCUの静止状態: 自発的なスケジューリング（cond_resched） */
/* 使用者: BPF、ftrace、ライブパッチ */
```

## RCUの拷問テスト {#rcu-torture-testing}

カーネルには、大規模なRCUのストレステストが含まれている。

```bash
# RCUのストレステストのためにrcutortureモジュールをロードする:
modprobe rcutorture

# 結果を見る:
dmesg | grep -E "rcu_torture|Readers Stall"
# rcu_torture: Readers stall PASSED: n=0
# rcu_torture: 0 expedited-grace-period failures

# rcuscale: 性能測定（Linux 5.10でrcuperfから改名 — 4e88ec4a9eb1）
modprobe rcuscale perf_type="rcu"
dmesg | grep "rcuscale"
```

## カーネル内のRCU: 主要な使用箇所 {#rcu-in-the-kernel-key-usage-sites}

```c
/* ルーティングテーブルの検索（ホットパス）: */
rcu_read_lock();
rt = rcu_dereference(dev->ip_ptr->rt_cache);  /* ロック不要 */
/* ... rtを使う ... */
rcu_read_unlock();

/* ネットワークインターフェースのリスト: */
rcu_read_lock();
list_for_each_entry_rcu(dev, &net->dev_base_head, dev_list) {
    if (dev->flags & IFF_UP)
        do_something(dev);
}
rcu_read_unlock();

/* プロセスのクレデンシャル（非常にホットなパス）: */
rcu_read_lock();
uid = rcu_dereference(current->cred)->uid;
rcu_read_unlock();
/* または最適化されたヘルパ: */
uid = current_uid();  /* 内部でrcu_read_lockを使う */
```

## RCUのデバッグ {#debugging-rcu}

```bash
# RCUストール（読み側クリティカルセクションで止まったCPU）をチェックする:
# カーネルは検出して次を出力する: "INFO: rcu_sched detected stalls on CPUs/tasks"
dmesg | grep "rcu.*stall"

# lockdepが誤用をチェックする:
# rcu_read_lock()はタスクに注釈を付け、スリープするとlockdepが警告する

# CONFIG_PROVE_RCU=y: lockdep方式のRCU正当性チェックを有効にする
# 警告対象: rcu_read_lockの外でのrcu_dereference()、読み側でのスリープ

# rcustat BPFツール:
bpftrace -e '
kprobe:synchronize_rcu { @[kstack] = count(); }' &
# 誰がsynchronize_rcu（高コストな操作）を呼んでいるかを示す
```

## 参考文献 {#further-reading}

- [メモリ管理におけるRCU](../mm/rcu-mm.md) — mmサブシステムでRCUがどう使われているか
- [spinlock](spinlock.md) — 書き込み中心の短いクリティカルセクション向け
- [rwsem](rwlock-rwsem.md) — スリープするreader-writerロック向け
- [SRCU](srcu.md) — スリープが必要なコンテキストのためのスリープ可能なRCU
- カーネルツリー内の`Documentation/RCU/` — Paul McKenneyによる充実したRCUのドキュメント
- `kernel/rcu/tree.c` — Tree RCUの実装
- `kernel/rcu/tasks.h` — Tasks RCU
