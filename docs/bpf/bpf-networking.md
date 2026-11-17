# BPFネットワーキングプログラム {#bpf-networking-programs}

> ネットワーク制御のためのTC、cgroup、ソケットBPFプログラム

## ネットワーキング向けのBPFプログラムタイプ {#bpf-program-types-for-networking}

| プログラムタイプ | アタッチ箇所 | パケットを変更できるか | ドロップできるか | ユースケース |
|-------------|-------------|-----------------|---------|---------|
| `XDP` | NICドライバ | 可 | 可 | DDoS緩和、ロードバランシング |
| `SCHED_CLS` (tc) | トラフィック制御qdisc | 可 | 可 | L3/L4ポリシー、カプセル化 |
| `SCHED_ACT` (tc action) | TCアクション | 可 | 可 | ダイレクトアクションモード |
| `CGROUP_SKB` | cgroupのingress/egress | 限定的 | 可 | コンテナのネットワークポリシー |
| `CGROUP_SOCK` | ソケット作成 | 不可 | 可 | ソケットレベルの制御 |
| `SK_SKB` | sockmap | 可 | 可 | ソケットリダイレクト |
| `SK_MSG` | sockmap (sendmsg) | 可 | 可 | メッセージレベルのリダイレクト |
| `SOCKET_FILTER` | rawソケット | 読み取りのみ | 可 | パケットキャプチャ（tcpdump） |

## TC BPF: トラフィック制御プログラム {#tc-bpf-traffic-control-programs}

TC BPFプログラムは`clsact` qdisc（classless act）にアタッチされる。このqdiscは、ネットワークスタックの前後にあるingressとegressのフックでBPFを実行できる。

```bash
# TC ingressにBPFプログラムをアタッチする
ip link add dev eth0 clsact
tc filter add dev eth0 ingress bpf da obj my_prog.o sec tc_ingress
# "da" = ダイレクトアクションモード（BPFがTC_ACT_*の判定を直接返す）

# egressにアタッチする
tc filter add dev eth0 egress bpf da obj my_prog.o sec tc_egress

# アタッチされたプログラムを一覧表示する
tc filter show dev eth0 ingress
tc filter show dev eth0 egress
```

### TC BPFプログラムの構造 {#tc-bpf-program-structure}

```c
#include <vmlinux.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>
#include <linux/pkt_cls.h>  /* TC_ACT_* */

/* 判定コード */
/* TC_ACT_OK      (0): パケットを通す */
/* TC_ACT_SHOT    (2): パケットをドロップする */
/* TC_ACT_REDIRECT(7): 別のインターフェースへリダイレクトする */
/* TC_ACT_UNSPEC (-1): 次のフィルタに委ねる */

SEC("tc")
int tc_ingress(struct __sk_buff *skb)
{
    void *data     = (void *)(long)skb->data;
    void *data_end = (void *)(long)skb->data_end;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return TC_ACT_OK;

    /* IPv4のみ扱う */
    if (eth->h_proto != bpf_htons(ETH_P_IP))
        return TC_ACT_OK;

    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end)
        return TC_ACT_OK;

    /* 特定の送信元IPからのトラフィックをドロップする */
    if (ip->saddr == bpf_htonl(0xC0A80101))  /* 192.168.1.1 */
        return TC_ACT_SHOT;

    return TC_ACT_OK;
}
```

### TC BPFによるパケットの変更 {#packet-modification-with-tc-bpf}

TC BPFプログラムはヘルパーを使ってパケットを変更できる。

```c
SEC("tc")
int tc_nat(struct __sk_buff *skb)
{
    void *data     = (void *)(long)skb->data;
    void *data_end = (void *)(long)skb->data_end;

    struct iphdr *ip = data + sizeof(struct ethhdr);
    if ((void *)(ip + 1) > data_end)
        return TC_ACT_OK;

    /* 宛先IPを書き換える（DNAT） */
    __be32 new_daddr = bpf_htonl(0x0A000001);  /* 10.0.0.1 */
    if (ip->daddr != bpf_htonl(0x0A000002))
        return TC_ACT_OK;

    /* IPヘッダを更新し、チェックサムを修正する */
    bpf_l3_csum_replace(skb, ETH_HLEN + offsetof(struct iphdr, check),
                          ip->daddr, new_daddr, sizeof(new_daddr));
    bpf_skb_store_bytes(skb, ETH_HLEN + offsetof(struct iphdr, daddr),
                         &new_daddr, sizeof(new_daddr), 0);

    return TC_ACT_OK;
}
```

### 別のインターフェースへのTCリダイレクト {#tc-redirect-to-another-interface}

```c
struct {
    __uint(type, BPF_MAP_TYPE_DEVMAP);
    __uint(max_entries, 128);
    __uint(key_size, sizeof(u32));
    __uint(value_size, sizeof(u32));
} tx_port SEC(".maps");

SEC("tc")
int tc_redirect(struct __sk_buff *skb)
{
    /* パケットをインターフェースインデックス5へリダイレクトする */
    return bpf_redirect(5, 0);  /* ifindex, flags */
}
```

## cgroup BPF: コンテナごとのネットワークポリシー {#cgroup-bpf-per-container-network-policy}

cgroup BPFプログラムはcgroupにアタッチされ、そのcgroup階層内のすべてのソケット/パケットに作用する。コンテナランタイム（Kubernetes、containerd）がネットワークポリシーに使っている。

### BPF_CGROUP_INET_INGRESS/EGRESS: パケットフィルタリング {#bpf_cgroup_inet_ingressegress-packet-filtering}

```c
SEC("cgroup_skb/ingress")
int cgroup_ingress(struct __sk_buff *skb)
{
    /* このcgroup内の任意のソケットが受信したすべてのパケットについて呼ばれる */
    /* 1を返す: 通す、0: ドロップ */

    /* ポート8080へのインバウンドトラフィックをブロックする */
    if (skb->protocol == bpf_htons(ETH_P_IP)) {
        struct iphdr *ip = (void *)(long)skb->data + ETH_HLEN;
        if ((void *)(ip + 1) > (void *)(long)skb->data_end)
            return 1;

        if (ip->protocol == IPPROTO_TCP) {
            struct tcphdr *tcp = (void *)(ip + 1);
            if ((void *)(tcp + 1) > (void *)(long)skb->data_end)
                return 1;
            if (tcp->dest == bpf_htons(8080))
                return 0;  /* ドロップ */
        }
    }
    return 1;  /* 通す */
}
```

```bash
# cgroupにアタッチする
bpftool prog load cgroup_filter.o /sys/fs/bpf/cgroup_filter

bpftool cgroup attach /sys/fs/cgroup/system.slice/mycontainer.scope \
    ingress pinned /sys/fs/bpf/cgroup_filter

# アタッチされたプログラムを一覧表示する
bpftool cgroup tree /sys/fs/cgroup/system.slice/
```

### BPF_CGROUP_SOCK_OPS: TCPソケットオプションの制御 {#bpf_cgroup_sock_ops-tcp-socket-option-control}

```c
SEC("sockops")
int bpf_sockops(struct bpf_sock_ops *skops)
{
    /* TCPの状態遷移とオプションのネゴシエーション時に呼ばれる */

    switch (skops->op) {
    case BPF_SOCK_OPS_TCP_CONNECT_CB:
        /* このcgroup内のすべてのコネクションにTCP_NODELAYを設定する */
        bpf_sock_ops_cb_flags_set(skops, BPF_SOCK_OPS_STATE_CB_FLAG);
        break;

    case BPF_SOCK_OPS_ACTIVE_ESTABLISHED_CB:
        /* ソケットバッファのサイズを設定する */
        bpf_setsockopt(skops, SOL_SOCKET, SO_SNDBUF, &(int){512*1024}, 4);
        /* 初期輻輳ウィンドウを増やす */
        bpf_setsockopt(skops, IPPROTO_TCP, TCP_CONGESTION, "bbr", 4);
        break;
    }
    return 1;
}
```

```bash
bpftool cgroup attach /sys/fs/cgroup/myapp sock_ops pinned /sys/fs/bpf/sockops
```

## sockmap: ソケットリダイレクト {#sockmap-socket-redirection}

sockmapを使うと、ユーザー空間を経由せずに、カーネルの速度でソケット間のデータをリダイレクトできる。

```
Normal:                         Sockmap redirect:
  Client → kernel → recv() →   Client → kernel (BPF) → Backend
  userspace → send() → kernel  (no userspace copy)
  → Backend
```

### sk_skb: 受信データのリダイレクト {#sk_skb-redirect-incoming-data}

```c
struct {
    __uint(type, BPF_MAP_TYPE_SOCKMAP);
    __uint(max_entries, 65536);
    __uint(key_size, sizeof(u32));
    __uint(value_size, sizeof(u32));
} sock_map SEC(".maps");

/* ストリームパーサ: ストリームをどうメッセージに分割するかを決める */
SEC("sk_skb/stream_parser")
int bpf_parser(struct __sk_buff *skb)
{
    /* 次のメッセージの長さを返す */
    return skb->len;  /* skbごとに1メッセージ */
}

/* ストリーム判定: 各メッセージをどう扱うかを決める */
SEC("sk_skb/stream_verdict")
int bpf_verdict(struct __sk_buff *skb)
{
    /* キーで宛先ソケットを検索する */
    u32 key = 0;
    return bpf_sk_redirect_map(skb, &sock_map, key, 0);
    /* データはユーザー空間へのコピーなしにターゲットソケットへ配送される */
}
```

```bash
# パーサと判定のプログラムをアタッチする
bpftool prog load sk_skb.o /sys/fs/bpf/sk_skb

bpftool map update id <sockmap_id> key 0 value <target_socket_fd>
```

### sk_msg: sendmsgへの介入 {#sk_msg-intercept-sendmsg}

```c
struct {
    __uint(type, BPF_MAP_TYPE_SOCKHASH);
    __uint(max_entries, 65536);
    __uint(key_size, sizeof(struct sock_key));
    __uint(value_size, sizeof(u32));
} sock_ops_map SEC(".maps");

SEC("sk_msg")
int bpf_redir_proxy(struct sk_msg_md *msg)
{
    struct sock_key key = {};
    /* 逆方向のキーを組み立てる: ピアへリダイレクトする */
    key.sip4 = msg->remote_ip4;
    key.dip4 = msg->local_ip4;
    key.sport = msg->remote_port;
    key.dport = (bpf_htonl(msg->local_port) >> 16);

    /* msgをピアのソケットへ直接リダイレクトする */
    return bpf_msg_redirect_hash(msg, &sock_ops_map, &key, BPF_F_INGRESS);
}
```

Ciliumは`sk_msg`と`sk_skb`を使い、ホットパスにサイドカープロキシを置かない透過的なサービスメッシュのデータプレーンを実装している。

## libbpf: ネットワークプログラムのアタッチ {#libbpf-attaching-network-programs}

```c
/* libbpf経由のTCアタッチ */
struct bpf_tc_hook hook = {
    .sz      = sizeof(hook),
    .ifindex = if_nametoindex("eth0"),
    .attach_point = BPF_TC_INGRESS,
};
bpf_tc_hook_create(&hook);

struct bpf_tc_opts opts = {
    .sz   = sizeof(opts),
    .prog_fd = bpf_program__fd(skel->progs.tc_ingress),
};
bpf_tc_attach(&hook, &opts);

/* デタッチ */
bpf_tc_detach(&hook, &opts);
bpf_tc_hook_destroy(&hook);
```

```c
/* libbpf経由のcgroupアタッチ */
int cgroup_fd = open("/sys/fs/cgroup/myapp", O_RDONLY);
bpf_prog_attach(bpf_program__fd(skel->progs.cgroup_ingress),
                cgroup_fd,
                BPF_CGROUP_INET_INGRESS,
                BPF_F_ALLOW_MULTI);  /* 既存のプログラムと連鎖させる */
```

## ネットワークBPFプログラムの観察 {#observing-network-bpf-programs}

```bash
# すべてのBPFプログラムを一覧表示する
bpftool prog list

# 特定のプログラムを表示する
bpftool prog show id 42 --pretty

# バイトコード / JITコンパイルされた命令をダンプする
bpftool prog dump xlated id 42
bpftool prog dump jited id 42

# インターフェース上のTC BPFフィルタを表示する
tc filter show dev eth0 ingress

# cgroup BPFプログラムを表示する
bpftool cgroup tree

# 統計情報
bpftool prog show id 42 | grep run_cnt
```

## 参考文献 {#further-reading}

- [XDP (eXpress Data Path)](../net/xdp.md) — ドライバレベルのBPF、Linux 4.8で導入 [(LWN)](https://lwn.net/Articles/682538/)
- [AF_XDPソケット](../net/af-xdp.md) — ゼロコピーのパケット処理
- [TCとqdisc](../net/tc-qdisc.md) — TCのアーキテクチャ
- [BPFアーキテクチャ](bpf-overview.md) — プログラムタイプの概要
- [BPF Verifier](bpf-verifier.md) — パケットアクセスに対する安全性の制約
- [libbpfとスケルトン](libbpf.md) — libbpfによるTC/cgroupアタッチ
- `net/core/filter.c` — sk_filter、ソケットフィルタの実装
- `net/sched/cls_bpf.c` — TC BPF分類器
