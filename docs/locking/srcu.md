# SRCU: スリープ可能なRCU {#srcu-sleepable-rcu}

> ブロックやスリープをしうる読み側クリティカルセクションのためのRCU

## 従来のRCUの問題 {#the-problem-with-classic-rcu}

従来のRCU（`rcu_read_lock`）には1つの重要な制約がある。**スリープしてはならない**ことである。読み手はブロックせずにクリティカルセクション内にとどまらなければならない。

```c
rcu_read_lock();
p = rcu_dereference(global_ptr);
/* ここでスリープしてはならない！ mutexもブロッキングI/Oも不可 */
do_something(p);
rcu_read_unlock();
```

これにより多くのユースケースが除外される。VFS操作（ロックを取ることがある）、I/O、GFP_KERNELでのメモリ割り当て、あるいはスケジュールしうるあらゆる操作である。

**SRCU**（Sleepable RCU）は、読み側クリティカルセクション内でのスリープを許す。トレードオフとして、SRCUは従来のRCUより高コストである（静止状態の検出ではなくper-CPUカウンタを使う）。

## SRCUの使い方 {#srcu-usage}

```c
#include <linux/srcu.h>

/* モジュールごとのSRCUドメインを宣言する */
DEFINE_SRCU(my_srcu);
/* または: */
struct srcu_struct my_srcu;
init_srcu_struct(&my_srcu);  /* 動的初期化 */

/* ---- 読み側 ---- */
int idx = srcu_read_lock(&my_srcu);
/* ここでスリープしてよい（mutex、wait_eventなど） */
p = srcu_dereference(global_ptr, &my_srcu);
mutex_lock(&p->lock);     /* ← SRCUではこれは問題ない */
do_work(p);
mutex_unlock(&p->lock);
srcu_read_unlock(&my_srcu, idx);

/* ---- 更新側 ---- */
old = rcu_replace_pointer(global_ptr, new_ptr, 1);

/* 現在のすべての読み手の完了を待つ */
synchronize_srcu(&my_srcu);  /* スリープしうる（ブロックする） */
/* oldを解放しても安全 */
kfree(old);

/* 非同期: */
call_srcu(&my_srcu, &old->rcu, my_free_callback);
```

## struct srcu_struct {#struct-srcu_struct}

```c
/* include/linux/srcu.h */
struct srcu_struct {
    struct srcu_node    node[NUM_RCU_NODES];  /* 緊急処理用ノード */
    struct srcu_node   *level[RCU_NUM_LVLS + 1]; /* ノードの階層 */
    int                 srcu_size_state;       /* 下記のサイズ */
    struct mutex        srcu_cb_mutex;         /* コールバックの直列化 */
    spinlock_t          lock;
    struct mutex        srcu_gp_mutex;
    unsigned int        srcu_idx;             /* 現在のグレースピリオドのidx */
    bool                srcu_gp_running;
    bool                srcu_gp_waiting;
    struct srcu_data __percpu *sda;           /* per-CPUカウンタ */
    struct list_head    srcu_work_list;
    struct delayed_work work;                 /* 定期的なグレースピリオドのチェック */
    struct lockdep_map  dep_map;
    unsigned long       srcu_gp_seq;          /* グレースピリオドのシーケンス */
    unsigned long       srcu_gp_seq_needed;
    unsigned long       srcu_gp_seq_needed_exp;
};
```

### per-CPUカウンタ: 決定的な違い {#per-cpu-counters-the-key-difference}

従来のRCUは静止状態をグローバルに追跡する。SRCUは、読み手がインクリメント/デクリメントするper-CPUカウンタを使う。

```c
/* srcu_read_lock: */
int idx = srcu_read_lock(sp)
    → this_cpu_inc(sp->sda->srcu_lock_count[idx & 1])
    → return idx

/* srcu_read_unlock: */
srcu_read_unlock(sp, idx)
    → this_cpu_inc(sp->sda->srcu_unlock_count[idx & 1])
```

グレースピリオドは、全CPUにわたって`srcu_lock_count[old_idx] == srcu_unlock_count[old_idx]`となったときに完了する。これは、グレースピリオドの開始前に始まったすべての読み手が終了したことを意味する。

## 緊急SRCU（expedited SRCU） {#expedited-srcu}

`synchronize_srcu_expedited()`は、CPUを能動的にポーリングすることでグレースピリオドを短縮する。

```c
/* synchronize_srcuより速いが、CPU負荷が高い */
synchronize_srcu_expedited(&my_srcu);
```

待つことが許容できない、まれでレイテンシが重要な更新に使う。

## SRCUと従来のRCUの比較 {#srcu-vs-classic-rcu}

| 機能 | RCU | SRCU |
|---------|-----|------|
| 読み側のオーバーヘッド | バリア（ほぼゼロ） | per-CPUカウンタのインクリメント |
| 読み側でのスリープ | 不可 | 可 |
| グレースピリオド | 非同期、自然な静止状態 | per-CPUカウンタの能動的なポーリング |
| 複数ドメイン | グローバルに1つ | ドメインごとに1つ（struct srcu_struct） |
| 典型的な用途 | ロックフリーなデータ構造 | VFS、notifier、モジュールのアンロード |
| `call_rcu`相当 | あり | `call_srcu` |

## SRCUの実際の使用例 {#real-world-uses-of-srcu}

**VFSのパス探索**: カーネルは、読み側でロックが必要になりうるnamespaceにSRCUを使っている。

```c
/* fs/namespace.c */
/* マウントnamespace保護のためのSRCU */
DEFINE_STATIC_SRCU(mount_lock);

int path_mount(const char *dev_name, struct path *path, ...)
{
    int mnt_flags = 0;
    int retval;

    /* ... */
    retval = do_mount(dev_name, path->dentry, type_page, flags, data_page);
    /* ... */
}
```

**notifierチェーン**: 多くのカーネルのnotifierチェーンは、スリープするコールバックを許すためにSRCUを使っている。

**モジュールのアンロード**: SRCUを使うと、スリープ中もモジュールへの参照を保持でき、操作の途中でモジュールがアンロードされるのを防げる。

## SRCUの観測 {#observing-srcu}

```bash
# SRCUのグレースピリオド統計
cat /sys/kernel/debug/rcu/rcudata

# lockdep: SRCUのアノテーションはlockdepの出力に現れる
# 読み側クリティカルセクションが長すぎる場合:
dmesg | grep "SRCU stall"
# rcu: INFO: SRCU stall warning (5999ms)

# SRCUストールを強制する（テスト用、CONFIG_RCU_STALL_COMMONが必要）
# echo 1 > /sys/kernel/debug/rcu/rcu_urgent_qs
```

## 参考文献 {#further-reading}

- [RCU](rcu.md) — スリープしない読み側のための従来のRCU
- [mutex](mutex.md) — ブロックする相互排他
- [completionと待ちキュー](completions.md) — ブロックする同期プリミティブ
- `kernel/rcu/srcutree.c` — SRCUの実装
- カーネルツリー内の`Documentation/RCU/Design/`
