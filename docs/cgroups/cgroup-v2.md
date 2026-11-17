# cgroup v2のアーキテクチャ {#cgroup-v2-architecture}

> 統一されたリソース制御の階層

## なぜcgroup v2が存在するのか {#why-cgroup-v2-exists}

### cgroup v1: コントローラごとに1つの階層 {#cgroup-v1-one-hierarchy-per-controller}

cgroup v1（Linux 2.6.24、2008年） [(LWN)](https://lwn.net/Articles/236437/) では、各リソースコントローラ（`cpu`、`memory`、`blkio`、`cpuset` など）がそれぞれ独立した階層を持つことができた。プロセス1234を、CPUのアカウンティングのためには `/sys/fs/cgroup/cpu/webserver/` に、メモリ制限のためには `/sys/fs/cgroup/memory/tier-a/` に置くことができ、それらは完全に独立したツリーだった。

これは柔軟に聞こえるが、根本的な問題を生んだ。**コントローラ間で「プロセスがどのグループに属するか」の共通の定義がなかった**のである。`cpu` コントローラはタスクをある見方で捉え、`memory` は別の見方で捉え、`blkio` はまた独自の見方を持っていた。systemdやコンテナランタイムが、CPU *と* メモリ *と* I/Oの制限を持つ一貫した「単位」（コンテナやサービス）を作ろうとすると、複数の階層の中に並行して位置を保持し、それらを手作業で同期させ続けなければならなかった。

この問題を象徴的に示したのが、ライトバックの帰属の問題である。プロセスがファイルに書き込むと、ディスクへの実際のI/Oは後でカーネルのワーカースレッド（`kworker`）から行われる。v1では、`blkio` コントローラはこのI/Oを、ページをダーティにしたプロセスではなく `kworker` に課金していた。グループへの所属に関する統一された概念がなければ、これを修正する方法はなかった。`memory` コントローラと `blkio` コントローラが、ダーティページをどのcgroupが所有するかについて合意する必要があったのである。

### cgroup v2: 統一階層 {#cgroup-v2-unified-hierarchy}

Tejun Heoは、単一のツリーを中心にcgroup v2を設計した（Linux 4.5、2016年にマージ） [(commit)](https://git.kernel.org/linus/34a9304a96d6351c2d35dcdc9293258378fc0bd8) [(LWN)](https://lwn.net/Articles/679786/)。すべてのコントローラが同じ階層の上で動作する。プロセスはちょうど1つのcgroupに属し、すべてのコントローラがそのcgroupに適用される。

これにより、適切なライトバックの帰属が可能になった。`memory` と `io` の両コントローラが同じ階層を共有するので、ダーティページが `kworker` によってフラッシュされる場合でも、正しいcgroupに帰属させることができる。

「内部プロセスなし」のルール（cgroupはプロセスと子のcgroupを同時に持てない）は、明快な委譲のセマンティクスを強制する。コンテナランタイムはあるサブツリーを所有し、そのサブツリー内に子のcgroupを作成できる。親のcgroupにいるプロセスが、子のリソースアカウンティングに誤って干渉することはない。

## cgroup v1とv2の比較 {#cgroup-v1-vs-v2}

| 機能 | v1 | v2 |
|---------|----|----|
| 階層 | コントローラごとに1つ | 単一の統一階層 |
| コントローラのアタッチ | 任意のcgroup | 親で有効にする必要がある |
| スレッドかプロセスか | スレッド単位の制御 | プロセス単位（スレッドモードはオプトイン） |
| 委譲 | 複雑なACL | 明快なサブツリー委譲 |
| BPFとの統合 | 限定的 | cgroup BPFプログラム |

cgroup v2（統一階層）は4.5でマージされ [(commit)](https://git.kernel.org/linus/34a9304a96d6351c2d35dcdc9293258378fc0bd8)、現在では現代的なディストリビューションのデフォルトになっている。ほとんどのサブシステムはv2のみをサポートする。

## 単一の階層 {#the-single-hierarchy}

cgroup v2のすべてのリソースは、`/sys/fs/cgroup` にある単一のファイルシステムにぶら下がっている:

```
/sys/fs/cgroup/                    ← root cgroup
├── cgroup.controllers             ← available controllers
├── cgroup.subtree_control         ← controllers delegated to children
├── cpu.stat
├── memory.current
├── system.slice/                  ← systemd service cgroup
│   ├── nginx.service/
│   │   ├── cgroup.procs           ← PIDs in this cgroup
│   │   ├── cpu.max                ← CPU quota
│   │   ├── memory.max             ← memory limit
│   │   └── io.max                 ← I/O bandwidth limit
│   └── sshd.service/
└── user.slice/
    └── user-1000.slice/
        └── session-1.scope/
```

## 主要なデータ構造 {#key-data-structures}

### struct cgroup {#struct-cgroup}

```c
/* kernel/cgroup/cgroup-internal.h */
struct cgroup {
    struct cgroup_subsys_state self;  /* cgroup_subsys 自身の CSS */
    unsigned long   flags;
    int             level;            /* ルートからの深さ（root=0） */
    int             max_depth;        /* サブツリーの深さの上限 */
    int             nr_descendants;   /* 空でない子孫の数 */

    struct cgroup  *dom_cgrp;         /* ドメイン cgroup（v2 用） */
    struct cgroup  *old_dom_cgrp;

    struct kernfs_node *kn;           /* cgroup の kernfs ディレクトリノード */
    struct cgroup_file  procs_file;   /* cgroup.procs のハンドル */
    struct cgroup_file  events_file;  /* cgroup.events のハンドル */

    u16 subtree_control;              /* 子に対して有効なコントローラ */
    u16 subtree_ss_mask;

    struct list_head    self_css_set_list;
    struct cgroup       **ancestors;   /* 高速検索用の祖先配列 */

    struct cgroup_bpf   bpf;          /* ここにアタッチされた BPF プログラム */

    atomic_t            nr_dying_descendants;
    struct cgroup_rstat_cpu __percpu *rstat_cpu;

    /* サブシステムごとのアクティブな状態: */
    struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];
};
```

### struct cgroup_subsys_state（CSS） {#struct-cgroup_subsys_state-css}

各コントローラは、CSSを介してcgroupごとの状態をアタッチする:

```c
struct cgroup_subsys_state {
    struct cgroup        *cgroup;     /* この CSS が属する cgroup */
    struct cgroup_subsys *ss;         /* コントローラのサブシステム */
    struct percpu_ref     refcnt;     /* 参照カウント */
    struct list_head      sibling;    /* 親の下の兄弟 CSS のリスト */
    struct list_head      children;   /* 子の CSS のリスト */
    struct list_head      rstat_css_node;

    u64     id;                       /* 一意な ID */
    unsigned int flags;               /* CSS_NO_REF, CSS_ONLINE, CSS_DYING */
    struct work_struct destroy_work;
    struct rcu_work destroy_rwork;
    struct cgroup_subsys_state *parent;
};
```

コントローラ（cpu、memory、ioなど）は、最初のフィールドとして `cgroup_subsys_state` を埋め込む:

```c
/* mm/memcontrol.c */
struct mem_cgroup {
    struct cgroup_subsys_state css;  /* 必ず先頭に置く */
    /* ... memory コントローラの状態 ... */
    struct page_counter memory;
    struct page_counter swap;
    unsigned long       soft_limit;
    /* ... */
};

/* アクセスパターン: CSS → mem_cgroup */
struct mem_cgroup *memcg = container_of(css, struct mem_cgroup, css);
```

### struct cgroup_subsys {#struct-cgroup_subsys}

各コントローラは自身を登録する:

```c
struct cgroup_subsys {
    struct cgroup_subsys_state *(*css_alloc)(struct cgroup_subsys_state *parent);
    int (*css_online)(struct cgroup_subsys_state *css);
    void (*css_offline)(struct cgroup_subsys_state *css);
    void (*css_free)(struct cgroup_subsys_state *css);
    void (*css_reset)(struct cgroup_subsys_state *css);

    int (*can_attach)(struct cgroup_taskset *tset);
    void (*attach)(struct cgroup_taskset *tset);  /* タスクがここに移された */
    void (*post_attach)(void);
    int (*can_fork)(struct task_struct *task, struct css_set *cset);
    void (*cancel_fork)(struct task_struct *task, struct css_set *cset);
    void (*fork)(struct task_struct *task);        /* 新しいプロセス */
    void (*exit)(struct task_struct *task);
    void (*release)(struct task_struct *task);
    void (*bind)(struct cgroup_subsys_state *root_css);

    bool        early_init:1;
    bool        implicit_on_dfl:1;
    bool        threaded:1;
    int         id;
    const char  *name;
    const char  *legacy_name;
    struct cgroupfs_root *root;
    struct idr  css_idr;
    struct list_head cfts;           /* cftype（インターフェースファイル） */
    struct cftype *dfl_cftypes;      /* デフォルトの v2 ファイル */
    struct cftype *legacy_cftypes;   /* v1 ファイル */
    unsigned int depends_on;
};
```

## コントローラの有効化 {#enabling-controllers}

コントローラはサブツリーで明示的に有効にする必要がある:

```bash
# ルートで利用可能なものを確認する
cat /sys/fs/cgroup/cgroup.controllers
# cpuset cpu io memory hugetlb pids rdma misc

# ルートの子に対してコントローラを有効にする
echo "+cpu +memory +io +pids" > /sys/fs/cgroup/cgroup.subtree_control

# cgroup を作成する
mkdir /sys/fs/cgroup/myapp

# myapp の子に対してコントローラを有効にする
echo "+cpu +memory" > /sys/fs/cgroup/myapp/cgroup.subtree_control

# プロセスを追加する（プロセスとその全スレッドをアトミックに移動する）
echo $$ > /sys/fs/cgroup/myapp/cgroup.procs
```

**内部プロセスなしの制約**: v2では、cgroupはプロセスと子のcgroupを同時に持つことができない。プロセスを直接持てるのは、リーフcgroup（子を持たないcgroup）とルートcgroupだけである。

## css_set: タスクとcgroupのリンク {#css_set-the-task-to-cgroup-link}

各タスクは `css_set` へのポインタを持ち、`css_set` はアクティブなコントローラごとに1つのCSSへの参照を保持する:

```c
struct css_set {
    struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];
    refcount_t   refcount;          /* 同じ CSS の組み合わせを持つタスク間で共有される */
    struct hlist_node hlist;        /* 全 css_set のハッシュテーブル */
    struct list_head tasks;         /* このセット内のタスク */
    struct list_head mg_tasks;      /* 移行中のタスク */
    struct list_head dying_tasks;
    struct list_head task_iters;
    struct list_head e_cset_node[CGROUP_SUBSYS_COUNT]; /* サブシステムごと */
    struct list_head threaded_csets;
    struct list_head threaded_csets_node;
    struct rcu_head  rcu_head;
    struct list_head mg_preload_node;
    struct list_head mg_node;
    struct cgroup   *mg_src_cgrp;
    struct cgroup   *mg_dst_cgrp;
    struct css_set  *mg_dst_cset;
    bool             dead;
    struct work_struct release_work;
};
```

CSSセットは共有される。2つのタスクがまったく同じcgroupの組（コントローラごとに1つ）に属しているなら、それらは `css_set` を共有する。これによりcss_setオブジェクトの数を少なく保てる。

## cgroup間でのプロセスの移動 {#moving-processes-between-cgroups}

```bash
# PID のプロセスを cgroup に移動する
echo <pid> > /sys/fs/cgroup/myapp/cgroup.procs
```

内部では次のように処理される:
```c
/* kernel/cgroup/cgroup.c: cgroup_procs_write() */
cgroup_attach_task()
  → cgroup_taskset_migrate()
      → for each controller:
          css->ss->can_attach(tset)    /* 権限チェック */
      → move task to new css_set
      → for each controller:
          css->ss->attach(tset)        /* コントローラに通知 */
```

## cgroup.eventsと通知 {#cgroupevents-and-notifications}

```bash
# cgroup が空/非空に遷移するのを監視する
cat /sys/fs/cgroup/myapp/cgroup.events
# populated 1
# frozen 0

# 変更を監視する（ファイルに対する inotify または poll）
inotifywait -e modify /sys/fs/cgroup/myapp/cgroup.events
```

## cgroup上のBPFプログラム {#bpf-programs-on-cgroups}

```bash
# BPF プログラムを cgroup にアタッチする（例: ネットワークフィルタ）
bpftool cgroup attach /sys/fs/cgroup/myapp/ ingress id <prog_id>
bpftool cgroup show /sys/fs/cgroup/myapp/
```

```c
/* ソケットのフィルタリングのために cgroup にアタッチされた BPF プログラム */
SEC("cgroup/skb")
int cgroup_filter(struct __sk_buff *skb)
{
    /* ソケット/コネクションの属性に基づいて許可または拒否する */
    return 1;  /* 1=許可、0=拒否 */
}
```

cgroupでサポートされるBPFのアタッチタイプ:

- `BPF_CGROUP_INET_INGRESS` / `BPF_CGROUP_INET_EGRESS`
- `BPF_CGROUP_INET_SOCK_CREATE` / `BPF_CGROUP_SOCK_OPS`
- `BPF_CGROUP_DEVICE` — デバイスアクセス制御
- `BPF_CGROUP_SYSCTL` — sysctlの読み書きの制御
- `BPF_CGROUP_INET4_CONNECT` / `BPF_CGROUP_INET6_CONNECT`
- `BPF_CGROUP_INET4_BIND` / `BPF_CGROUP_INET6_BIND`

## 参考文献 {#further-reading}

- [リソースコントローラ](cgroup-controllers.md) — cpu、memory、io、pidsの詳細
- [コンテナの隔離](container-isolation.md) — cgroupとnamespaceの組み合わせ方
- `Documentation/admin-guide/cgroup-v2.rst` — 包括的なカーネルドキュメント
