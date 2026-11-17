# ネットワークnamespace {#network-namespaces}

> namespaceごとのネットワークスタック、デバイスの移動、コンテナのネットワーク分離

## ネットワークnamespaceが提供するもの {#what-network-namespaces-provide}

各**ネットワークnamespace**は、次のものを独自に持つ:

- ネットワークインターフェース（lo、eth0など）
- IPアドレスとルーティングテーブル
- iptables/nftablesのルール
- netfilterのコネクショントラッキング
- Unixドメインソケットの名前空間
- ポート番号（独立した0〜65535）
- `/proc/net/`のビュー

```bash
# コンテナA（namespace ns1）:            コンテナB（namespace ns2）:
ip addr: 172.17.0.2/16                ip addr: 172.17.0.3/16
route: default via 172.17.0.1         route: default via 172.17.0.1
iptables: (container rules)           iptables: (container rules)
# 完全に分離されている — namespace間で互いに見えない
```

## ネットワークnamespaceの作成と使用 {#creating-and-using-network-namespaces}

```bash
# 名前付きネットワークnamespaceを作成:
ip netns add myns

# namespaceを一覧表示:
ip netns list
# myns (id: 0)

# namespace内でコマンドを実行:
ip netns exec myns ip addr
# 1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN
#     link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

# 対話的にnamespaceに入る:
ip netns exec myns bash
# 以降すべてのネットワークコマンドはmyns内で動作する

# ループバックを設定:
ip netns exec myns ip link set lo up
ip netns exec myns ip addr add 127.0.0.1/8 dev lo

# namespaceを削除:
ip netns del myns
```

## vethペア: namespace同士をつなぐ {#veth-pairs-connecting-namespaces}

```bash
# vethペアを作成:
ip link add veth0 type veth peer name veth1

# 一端をnamespaceに移動:
ip link set veth1 netns myns

# 両端を設定:
ip addr add 10.0.0.1/24 dev veth0
ip link set veth0 up

ip netns exec myns ip addr add 10.0.0.2/24 dev veth1
ip netns exec myns ip link set veth1 up

# ホストとnamespaceの間でping:
ping 10.0.0.2      # ホスト → namespace
ip netns exec myns ping 10.0.0.1  # namespace → ホスト
```

## カーネルの内部: struct net {#kernel-internals-struct-net}

```c
/* include/net/net_namespace.h */

struct net {
    /* 参照カウント */
    refcount_t          passive;    /* 受動的な利用者: アクティブな処理なし */
    spinlock_t          rules_mod_lock;
    atomic64_t          net_cookie; /* 一意なID */
    struct list_head    list;       /* すべてのネットワークnamespace */

    /* ネットワークデバイス */
    struct list_head    dev_base_head;  /* このnsのすべてのnet_device */
    struct hlist_head  *dev_name_head;  /* 名前によるハッシュ */
    struct hlist_head  *dev_index_head; /* ifindexによるハッシュ */
    unsigned int        dev_base_seq;
    unsigned int        ifindex;        /* 次に割り当てるifindex */

    /* ルーティング */
    struct netns_ipv4   ipv4;       /* IPv4のテーブル、ルーティング */
    struct netns_ipv6   ipv6;       /* IPv6のテーブル */

    /* ソケットとプロトコル */
    struct sock        *rtnl;       /* rtnetlinkソケット */
    struct sock        *genl_sock;  /* generic netlink */

    /* iptables / nftables */
    struct netns_nf     nf;
    struct netns_xt     xt;

    /* コネクショントラッキング */
    struct netns_ct     ct;

    /* procとsysctl */
    struct proc_dir_entry *proc_net;
    struct ctl_table_set sysctls;

    struct user_namespace  *user_ns;   /* 所有者のuser namespace */
    struct ucounts         *ucounts;
    struct idr              netns_ids;  /* netnsid用（rtnetlinkが使う） */
    struct ns_common        ns;         /* namespace共通のフィールド */
    /* ... */
};

/* 初期（ホスト）ネットワークnamespace: */
extern struct net init_net;

/* 現在のタスクのネットワークnamespaceを取得: */
static inline struct net *sock_net(const struct sock *sk)
{
    return read_pnet(&sk->sk_net);
}
```

## namespaceのライフサイクル {#namespace-lifecycle}

```c
/* 新しいネットワークnamespaceの作成（clone_newnet）: */
/* kernel/nsproxy.c → net/core/net_namespace.c */

static int setup_net(struct net *net, struct user_namespace *user_ns)
{
    /* 1. namespaceごとのデータ構造を初期化する */
    list_for_each_entry(ops, &pernet_list, list) {
        error = ops->init(net);  /* 各サブシステムが初期化する */
    }
    /* 2. ループバックデバイス（lo）を作成する */
    /* 3. ルーティングテーブルを初期化する */
    /* 4. conntrackを初期化する */
    return 0;
}

/* namespaceが解放されるときのクリーンアップ: */
static void cleanup_net(struct work_struct *work)
{
    /* すべてのpernet_operationsのexitコールバックを逆順に実行する */
    list_for_each_entry_reverse(ops, &pernet_list, list) {
        if (ops->exit)
            ops->exit(net);
    }
}
```

### pernet_operations: サブシステムのフック {#pernet_operations-subsystem-hooks}

namespaceごとの状態を持つすべてのサブシステムは、クリーンアップ/初期化のフックを登録する:

```c
/* 例: namespaceごとのルーティングテーブル */
static int __net_init ip_fib_net_init(struct net *net)
{
    /* namespaceごとのルーティングテーブルを割り当てる */
    net->ipv4.fib_table_hash = kzalloc(...);
    return 0;
}

static void __net_exit ip_fib_net_exit(struct net *net)
{
    /* namespaceごとのルーティングテーブルを解放する */
    kfree(net->ipv4.fib_table_hash);
}

static struct pernet_operations fib_net_ops = {
    .init = ip_fib_net_init,
    .exit = ip_fib_net_exit,
};
register_pernet_subsys(&fib_net_ops);
```

## namespace間でのネットワークデバイスの移動 {#moving-network-devices-between-namespaces}

```bash
# 物理/仮想デバイスをnamespaceに移動:
ip link set eth1 netns myns

# 元に戻す:
ip netns exec myns ip link set eth1 netns 1  # 1 = init_netのnsid
```

```c
/* net/core/dev.c */
int dev_change_net_namespace(struct net_device *dev,
                              struct net *net, const char *pat)
{
    /* 1. 現在のnetから切り離す: dev_close、unregister_netdevice */
    dev_close(dev);
    /* 2. netポインタを変更する: */
    dev_net_set(dev, net);
    /* 3. 新しいnetに登録する: */
    register_netdevice(dev);
    return 0;
}
```

## ip netns execの実装 {#ip-netns-exec-implementation}

```bash
# ip netns exec myns COMMAND
# は次と等価:
# 1. open("/var/run/netns/myns")    → nsへのファイルディスクリプタを得る
# 2. setns(fd, CLONE_NEWNET)        → そのnsに切り替える
# 3. exec(COMMAND)
```

```c
/* ip netns execはsetns(2)を使う: */
int ns_fd = open("/var/run/netns/myns", O_RDONLY);
setns(ns_fd, CLONE_NEWNET);   /* ネットワークnamespaceに参加する */
execv(argv[0], argv);
```

## ネットワークnamespaceの統計 {#network-namespace-statistics}

```bash
# namespaceごとのネットワーク統計:
ip netns exec myns cat /proc/net/dev
# Inter-|   Receive                                                |  Transmit
#  face |bytes    packets errs drop fifo frame compressed multicast|bytes    packets
#     lo:       0       0    0    0    0     0          0         0        0       0
#  veth1:  123456    1000    0    0    0     0          0         0    65432     800

# namespaceごとのソケット統計:
ip netns exec myns ss -tulnp

# namespaceごとのすべてのオープン中のソケット:
ip netns exec myns cat /proc/net/tcp

# ネットワークnamespace ID（コンテナでのns間ルーティング用）:
ip netns list-id
# nsid 0（現在のnsから見たinit_netのiproute2 ID）
```

## 可観測性 {#observability}

```bash
# プロセスのネットワークnamespaceを確認:
readlink /proc/1234/ns/net
# net:[4026531992]  ← inode番号がnsを一意に識別する

# あるnamespace内のすべてのプロセスを見つける:
ls -la /proc/*/ns/net | grep "4026531992"

# システム上のすべての一意なネットワークnamespaceを一覧表示:
find /proc -maxdepth 3 -name 'net' -path '*/ns/*' -print0 | \
    xargs -0 -I{} readlink {} | sort -u

# BPF: namespaceの作成をトレース:
bpftrace -e '
kprobe:setup_net
{ printf("new net ns: pid=%d, comm=%s\n", pid, comm); }'
```

## 参考文献 {#further-reading}

- [コンテナの分離](../cgroups/container-isolation.md) — コンテナにおけるnamespace
- [namespace](../cgroups/namespaces.md) — Linuxの全namespaceの概要
- [コンテナネットワーキング](container-networking.md) — veth、ブリッジ、オーバーレイ
- [Netfilter](netfilter.md) — namespaceごとのiptables
- [sk_buff](sk-buff.md) — namespaceのタグが付いたネットワークバッファ
- `net/core/net_namespace.c` — ネットワークnamespaceのライフサイクル
- `man ip-netns` — ip netnsコマンドのリファレンス
