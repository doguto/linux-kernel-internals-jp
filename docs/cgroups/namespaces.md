# Linuxのnamespace {#linux-namespaces}

> プロセスから見えるものを隔離する

## 8種類のnamespace {#the-eight-namespace-types}

| namespace | フラグ | 隔離するもの | カーネル |
|-----------|------|----------|--------|
| Mount | `CLONE_NEWNS` | マウントポイント、ファイルシステムのビュー | 2.4.19（2002年） |
| UTS | `CLONE_NEWUTS` | ホスト名、NISドメイン名 | 2.6.19 |
| IPC | `CLONE_NEWIPC` | SysV IPC、POSIXメッセージキュー | 2.6.19 |
| PID | `CLONE_NEWPID` | プロセスID | 2.6.24 [(LWN)](https://lwn.net/Articles/259217/) |
| Network | `CLONE_NEWNET` | ネットワークインターフェース、ルーティング、ソケット | 2.6.24 |
| User | `CLONE_NEWUSER` | ユーザー/グループID | 3.9（完成） [(LWN)](https://lwn.net/Articles/532593/) |
| Cgroup | `CLONE_NEWCGROUP` | cgroupルートのビュー | 4.6 |
| Time | `CLONE_NEWTIME` | CLOCK_MONOTONIC、CLOCK_BOOTTIMEのオフセット | 5.6 |

## namespaceの作成 {#creating-namespaces}

### clone() — 新しいnamespaceの中に子を作成する {#clone-create-child-in-new-namespace}

```c
#define _GNU_SOURCE
#include <sched.h>
#include <unistd.h>

/* 子のためのスタック */
static char child_stack[65536];

static int child_fn(void *arg)
{
    /* 新しい namespace の中で実行中 */
    printf("child PID: %d\n", getpid());  /* 新しい PID ns では 1 と表示される */
    return 0;
}

int main(void)
{
    pid_t child = clone(child_fn,
                        child_stack + sizeof(child_stack),
                        CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET | SIGCHLD,
                        NULL);
    waitpid(child, NULL, 0);
    return 0;
}
```

### unshare() — 現在のプロセスが新しいnamespaceに入る {#unshare-current-process-enters-new-namespace}

```c
/* 共有している namespace から切り離す */
unshare(CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC);

/* 同等のシェルコマンド */
/* unshare --mount --uts --ipc bash */
```

```bash
# シェル: 新しい namespace の中で bash を実行する
unshare --pid --mount --fork --mount-proc bash
# これで隔離された PID+mount namespace に入り、ps にはこの bash のサブツリーだけが表示される
```

### setns() — 既存のnamespaceに参加する {#setns-join-an-existing-namespace}

```c
#include <fcntl.h>
#include <sched.h>

/* 別のプロセスの network namespace に入る */
int nsfd = open("/proc/1234/ns/net", O_RDONLY);
setns(nsfd, CLONE_NEWNET);
close(nsfd);
/* これで pid 1234 の network namespace の中にいる */
```

```bash
# nsenter: 実行中のコンテナの namespace に入る
nsenter --target <pid> --mount --pid --net -- bash
```

## struct nsproxy: タスクのnamespaceの集合 {#struct-nsproxy-the-tasks-namespace-set}

各タスクは、自身のnamespaceの集合への参照を保持する:

```c
/* include/linux/nsproxy.h */
struct nsproxy {
    atomic_t          count;          /* 参照カウント */
    struct uts_namespace   *uts_ns;
    struct ipc_namespace   *ipc_ns;
    struct mnt_namespace   *mnt_ns;
    struct pid_namespace   *pid_ns_for_children;
    struct net             *net_ns;
    struct time_namespace  *time_ns;
    struct time_namespace  *time_ns_for_children;
    struct cgroup_namespace *cgroup_ns;
};

/* struct task_struct の中: */
struct nsproxy *nsproxy;
```

すべてのnamespaceを共有するプロセスは、同じ `nsproxy` を指す（参照カウント）。いずれかのnamespaceが分岐すると、新しいnsproxyが割り当てられる。

## PID namespace {#pid-namespace}

PID namespaceはLinux 2.6.24で導入された [(LWN)](https://lwn.net/Articles/259217/)。各PID namespaceは1から始まる独自の番号付けを持つ。1つのプロセスは、namespaceごとに異なるPIDを持つ:

```
Host PID namespace:        pid=1234 (systemd), pid=5678 (container init)
Container PID namespace:   pid=1 (container init), pid=100 (nginx)
```

```c
/* include/linux/pid_namespace.h */
struct pid_namespace {
    struct idr      idr;            /* pid → タスクの対応付け */
    struct rcu_head rcu;
    unsigned int    level;          /* ネストの深さ（0=init_pid_ns） */
    struct pid_namespace *parent;
    struct user_namespace *user_ns;
    struct ucounts *ucounts;
    struct ns_common ns;

    struct task_struct *child_reaper; /* この namespace の PID 1 */
    struct kmem_cache *pid_cachep;
    unsigned int    nr_hashed;
};
```

PID namespaceにおけるPID 1はinitである。これが終了すると、そのnamespace内のすべてのプロセスがkillされる。

```bash
# PID の対応を確認する
cat /proc/1234/status | grep NSpid
# NSpid:  5678   1    ← ホストの PID 5678 = コンテナの namespace における PID 1
```

## mount namespace {#mount-namespace}

mount namespace（Linux 2.4.19、2002年）は、カーネルに最初に追加されたnamespaceの種類である。そのため、フラグは `CLONE_NEWMNT` のような名前ではなく、汎用的な `CLONE_NEWNS` になっている。各mount namespaceはマウントツリーの独自のコピーを持つ。あるnamespaceでの変更は他に影響しない。

```c
/* fs/namespace.c */
struct mnt_namespace {
    struct ns_common    ns;
    struct mount       *root;           /* ルートマウント */
    struct list_head    list;           /* すべてのマウント */
    struct user_namespace *user_ns;
    u64                 seq;            /* イベントのシーケンス番号 */
    wait_queue_head_t   poll;
    u64                 event;
};
```

```bash
# 隔離された mount namespace を作成し、新しい PID ns 用に procfs をマウントする
unshare --mount --pid --fork --mount-proc bash

# このバインドマウントにより、新しい /proc は PID namespace の内容を表示する
# ホストからは古い /proc が見える
```

### マウント伝播の種類 {#mount-propagation-types}

```bash
# shared: マウントがピアに伝播する
mount --make-shared /mnt

# private: 伝播しない
mount --make-private /mnt

# slave: マスターからの伝播を受け取るが、送り返さない
mount --make-slave /mnt

# unbindable: バインドマウントできない
mount --make-unbindable /mnt
```

## network namespace {#network-namespace}

各network namespaceは、次のものを独自に持つ:

- ネットワークインターフェース（ただしループバックはnamespaceごとに別々）
- IPルーティングテーブル
- Netfilterのルール（iptables）
- ソケットテーブル
- `/proc/net/` のビュー

```c
/* include/net/net_namespace.h */
struct net {
    /* 最初のキャッシュライン: 頻繁にアクセスされるフィールド */
    refcount_t          passive;      /* パッシブ参照カウント */
    spinlock_t          rules_mod_lock;

    unsigned int        dev_unreg_count;
    unsigned int        dev_base_seq;

    struct list_head    list;         /* すべての net namespace のリスト */
    struct list_head    exit_list;
    struct llist_node   cleanup_list;

    struct user_namespace *user_ns;   /* 所有者の user namespace */
    struct ucounts      *ucounts;
    struct idr          netns_ids;

    struct ns_common    ns;
    struct ref_tracker_dir  refcnt_tracker;

    struct list_head    dev_base_head; /* ネットワークデバイスのリスト */
    struct proc_dir_entry *proc_net;
    struct proc_dir_entry *proc_net_stat;

    /* プロトコル固有の namespace 化された状態: */
    struct netns_ipv4   ipv4;
    struct netns_ipv6   ipv6;
    struct netns_unix   unx;
    struct netns_packet packet;
    struct netns_nftables nft;
    /* ... */
};
```

```bash
# network namespace を作成する
ip netns add myns

# namespace の中でコマンドを実行する
ip netns exec myns ip link show

# veth ペアを namespace に移す（典型的なコンテナのネットワーク構成）
ip link add veth0 type veth peer name veth1
ip link set veth1 netns myns
ip addr add 10.0.0.1/24 dev veth0
ip netns exec myns ip addr add 10.0.0.2/24 dev veth1
ip link set veth0 up
ip netns exec myns ip link set veth1 up

# network namespace を一覧表示する
ip netns list
ls /var/run/netns/
```

## user namespace {#user-namespace}

user namespaceは、Linux 3.9でEric W. Biedermanによって機能的に完成し、非特権ユーザーから使えるようになった [(commit)](https://git.kernel.org/linus/94f2f14234178f118545a0be60a6371ddeb229b7) [(LWN)](https://lwn.net/Articles/532593/)。user namespaceは内側と外側の間でUID/GIDをマップする:

```c
/* kernel/user_namespace.c */
struct user_namespace {
    struct uid_gid_map  uid_map;        /* uid のマッピング規則 */
    struct uid_gid_map  gid_map;
    struct uid_gid_map  projid_map;
    struct ucounts     *ucounts;
    struct user_namespace *parent;
    int                 level;
    kuid_t              owner;
    kgid_t              group;
    struct ns_common    ns;
    unsigned long       flags;
    struct list_head    keyring_name_list;
    struct key         *user_keyring_register;
    struct rw_semaphore keyring_sem;
};

struct uid_gid_map {
    u32                 nr_extents;     /* マッピング範囲の数 */
    union {
        struct uid_gid_extent extent[UID_GID_MAP_MAX_BASE_EXTENTS];
        struct {
            struct uid_gid_extent *forward;
            struct uid_gid_extent *reverse;
        };
    };
};
```

```bash
# 非特権ユーザーとして user namespace を作成する
unshare --user --map-root-user bash
# これで: 内側の "root" = 外側の uid 1000

# UID マッピングを確認する
cat /proc/self/uid_map
# 0       1000          1   ← 内側の開始 外側の開始 個数
# 内側の UID 0 = 外側の UID 1000、1 エントリ

# 複数範囲のマッピング（特権が必要）
echo "0 1000 10" > /proc/<pid>/uid_map
# 内側の UID 0-9 が外側の UID 1000-1009 にマップされる
```

重要な性質は、プロセスがuser namespaceの中ではUID 0（root）でありながら、その外側では何の特権も持たない、という点である。これによりrootlessコンテナが可能になる。

### ケーパビリティの範囲 {#capability-scope}

user namespace内のケーパビリティは、そのnamespaceとその子孫が所有するリソースに対してのみ特権を与える。user namespace内のrootは次のことができない:

- カーネルモジュールのロード
- namespace外のカーネルパラメータの変更
- ホスト上の他のユーザーが所有するファイルの読み取り

## UTS namespace {#uts-namespace}

```bash
# ホスト名を隔離する
unshare --uts bash
hostname container1
# ホストのホスト名は変わらない
```

```c
struct uts_namespace {
    struct new_utsname name;    /* nodename（ホスト名）を含む */
    struct user_namespace *user_ns;
    struct ucounts *ucounts;
    struct ns_common ns;
};
```

## namespaceのファイルディスクリプタ {#namespace-file-descriptors}

namespaceは、次のいずれかが成り立つ限り存続する:
1. プロセスがその中にいる、または
2. ファイルディスクリプタが開かれている、または
3. `/proc/<pid>/ns/<type>` や `/var/run/netns/` へのバインドマウントがある

```bash
# プロセスが終了しても network namespace を存続させる
touch /var/run/netns/myns
mount --bind /proc/<pid>/ns/net /var/run/netns/myns
# これで、作成したプロセスが終了した後も myns は存続する
```

## namespaceの観察 {#observing-namespaces}

```bash
# 全プロセスの namespace ID を確認する（一致 = 共有）
lsns
# NS TYPE   NPROCS   PID USER       COMMAND
# 4026531835 cgroup    102     1 root       /sbin/init
# 4026531836 pid       102     1 root       /sbin/init
# 4026531992 net        98     1 root       /sbin/init
# 4026532156 mnt         1  1234 user       bash

# コンテナが使っている namespace を確認する
lsns --task <container_pid>

# /proc/self/ns/ のシンボリックリンクを調べる
ls -la /proc/self/ns/

# 同じ network namespace にいるすべてのプロセスを見つける
lsns -t net
```

## 参考文献 {#further-reading}

- [コンテナの隔離](container-isolation.md) — namespaceとcgroupの組み合わせ方
- [cgroup v2のアーキテクチャ](cgroup-v2.md) — 隔離を補完するリソース制御
- `man 7 namespaces` — 全種類のnamespaceについてのカーネルドキュメント
- `man 2 clone`、`man 2 unshare`、`man 2 setns`
