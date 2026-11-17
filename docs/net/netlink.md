# Netlink {#netlink}

> ネットワーク設定ツールのためのカーネルとユーザー空間の通信

## Netlinkとは何か {#what-is-netlink}

Netlinkは、ユーザー空間とカーネルの間で通信するための、ソケットベースのIPC機構である。ほとんどのネットワーク設定ツールはこれを使って動作している:

| ツール | Netlinkファミリ | 目的 |
|------|---------------|---------|
| `ip link`, `ip addr` | `NETLINK_ROUTE` (rtnetlink) | インターフェース/アドレス/経路の管理 |
| `ss`, `netstat` | `NETLINK_SOCK_DIAG` | ソケットの統計 |
| `ethtool` | `NETLINK_GENERIC` (ethtool_nl) | NICの設定 |
| `iw`, `nmcli` | `NETLINK_GENERIC` (nl80211) | WiFiの設定 |
| `tc` | `NETLINK_ROUTE` | トラフィック制御 |
| `nftables` | `NETLINK_NETFILTER` | ファイアウォールルール |
| `audit` | `NETLINK_AUDIT` | 監査サブシステム |
| `uevent` | `NETLINK_KOBJECT_UEVENT` | デバイスのホットプラグイベント |

## Netlinkソケットの基本 {#netlink-socket-basics}

```c
#include <linux/netlink.h>
#include <sys/socket.h>

/* Netlink ソケットを開く */
int fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE);

/* bind: 自分の PID を登録する */
struct sockaddr_nl addr = {
    .nl_family = AF_NETLINK,
    .nl_pid    = getpid(),   /* 0 = カーネルが割り当てる */
    .nl_groups = 0,          /* マルチキャストグループ（後述） */
};
bind(fd, (struct sockaddr *)&addr, sizeof(addr));
```

## Netlinkのメッセージ形式 {#netlink-message-format}

すべてのNetlinkメッセージは固定のヘッダを持つ:

```c
/* include/uapi/linux/netlink.h */
struct nlmsghdr {
    __u32 nlmsg_len;    /* ヘッダを含む全体の長さ */
    __u16 nlmsg_type;   /* メッセージタイプ（ファミリ固有） */
    __u16 nlmsg_flags;  /* フラグ: NLM_F_REQUEST, NLM_F_ACK, NLM_F_DUMP, ... */
    __u32 nlmsg_seq;    /* シーケンス番号（応答との対応付け用） */
    __u32 nlmsg_pid;    /* 送信元 PID（0 = カーネル） */
    /* この後にペイロードが続く: ファミリ固有の構造体 + 属性 */
};
```

よく使われるフラグ:

- `NLM_F_REQUEST`: カーネルへの要求であることを示す
- `NLM_F_ACK`: 確認応答を要求する
- `NLM_F_DUMP`: この種類のオブジェクトをすべてダンプする
- `NLM_F_CREATE`: 存在しなければ作成する（追加操作）
- `NLM_F_EXCL`: 存在すれば失敗する（追加操作）

## rtnetlink: ネットワークインターフェースの管理 {#rtnetlink-network-interface-management}

`NETLINK_ROUTE`（rtnetlink）は、`ip`がインターフェース、アドレス、経路を管理するために使う。

### すべてのインターフェースを一覧表示する（GETLINKダンプ） {#list-all-interfaces-getlink-dump}

```c
#include <linux/rtnetlink.h>
#include <net/if.h>

/* RTM_GETLINK のダンプ要求を組み立てる */
struct {
    struct nlmsghdr nlh;
    struct ifinfomsg ifi;
} req = {
    .nlh = {
        .nlmsg_len   = NLMSG_LENGTH(sizeof(struct ifinfomsg)),
        .nlmsg_type  = RTM_GETLINK,
        .nlmsg_flags = NLM_F_REQUEST | NLM_F_DUMP,
        .nlmsg_seq   = 1,
    },
    .ifi = {
        .ifi_family = AF_UNSPEC,
    },
};

send(fd, &req, req.nlh.nlmsg_len, 0);

/* 応答を読む（複数のメッセージ、インターフェースごとに1つ） */
char buf[8192];
while (1) {
    ssize_t n = recv(fd, buf, sizeof(buf), 0);
    struct nlmsghdr *nlh = (struct nlmsghdr *)buf;

    for (; NLMSG_OK(nlh, n); nlh = NLMSG_NEXT(nlh, n)) {
        if (nlh->nlmsg_type == NLMSG_DONE)
            goto done;
        if (nlh->nlmsg_type == RTM_NEWLINK) {
            struct ifinfomsg *ifi = NLMSG_DATA(nlh);
            /* 属性を解析する */
            struct rtattr *rta = IFLA_RTA(ifi);
            int len = IFLA_PAYLOAD(nlh);
            for (; RTA_OK(rta, len); rta = RTA_NEXT(rta, len)) {
                if (rta->rta_type == IFLA_IFNAME)
                    printf("interface: %s\n", (char *)RTA_DATA(rta));
                if (rta->rta_type == IFLA_MTU)
                    printf("mtu: %u\n", *(uint32_t *)RTA_DATA(rta));
            }
        }
    }
}
done:;
```

### Netlinkの属性（rtattr / nlattr） {#netlink-attributes-rtattr-nlattr}

NetlinkはTLV（Type-Length-Value）形式の属性を使う:

```c
/* rtattr の形式: */
struct rtattr {
    unsigned short rta_len;   /* ヘッダを含む全体の長さ */
    unsigned short rta_type;  /* 属性タイプ */
    /* この後にデータが続く */
};

/* 解析: RTA_DATA(rta) は属性データを指す */
/* ネスト: RTA_DATA(rta) 自体が rtatrr のリストになる */

/* 現代の generic netlink は同じレイアウトの nlattr を使う: */
struct nlattr {
    __u16 nla_len;
    __u16 nla_type;  /* 上位2ビット: NLA_F_NESTED, NLA_F_NET_BYTEORDER */
};
```

### RTM_NEWROUTE: 経路の追加 {#rtm_newroute-add-a-route}

```c
/* 経路を追加: 10.0.0.0/8 via 192.168.1.1 dev eth0 */
struct {
    struct nlmsghdr  nlh;
    struct rtmsg     rtm;
    char             attrs[256];
} req = {};

req.nlh.nlmsg_len   = NLMSG_LENGTH(sizeof(req.rtm));
req.nlh.nlmsg_type  = RTM_NEWROUTE;
req.nlh.nlmsg_flags = NLM_F_REQUEST | NLM_F_CREATE | NLM_F_ACK;
req.nlh.nlmsg_seq   = 2;

req.rtm.rtm_family   = AF_INET;
req.rtm.rtm_dst_len  = 8;   /* プレフィックス長 */
req.rtm.rtm_table    = RT_TABLE_MAIN;
req.rtm.rtm_protocol = RTPROT_STATIC;
req.rtm.rtm_scope    = RT_SCOPE_UNIVERSE;
req.rtm.rtm_type     = RTN_UNICAST;

/* 宛先を追加: RTA_DST */
uint32_t dst = inet_addr("10.0.0.0");
addattr_l(&req.nlh, sizeof(req), RTA_DST, &dst, 4);

/* ゲートウェイを追加: RTA_GATEWAY */
uint32_t gw = inet_addr("192.168.1.1");
addattr_l(&req.nlh, sizeof(req), RTA_GATEWAY, &gw, 4);

/* インターフェースインデックスを追加: RTA_OIF */
int ifidx = if_nametoindex("eth0");
addattr32(&req.nlh, sizeof(req), RTA_OIF, ifidx);

send(fd, &req, req.nlh.nlmsg_len, 0);
/* ACK を読んで成功を確認する */
```

## Generic Netlink（genetlink） {#generic-netlink-genetlink}

Generic Netlinkはマルチプレクサであり、カーネルのサブシステムが予約済みの`NETLINK_*`プロトコル番号を使わずに独自のメッセージファミリを定義できるようにする。

```
AF_NETLINK + NETLINK_GENERIC
    ↓
genetlink dispatcher
    ├── family "ethtool"   → ethtool_nl handlers
    ├── family "nl80211"   → cfg80211 handlers
    ├── family "nlctrl"    → list families
    └── family "devlink"   → devlink handlers
```

### libnl3の利用 {#using-libnl3}

ほとんどのツールは生のソケットではなく`libnl3`を使う:

```c
#include <netlink/netlink.h>
#include <netlink/genl/genl.h>
#include <netlink/genl/ctrl.h>

struct nl_sock *sock = nl_socket_alloc();
genl_connect(sock);

/* ファミリ名を ID に解決する */
int family = genl_ctrl_resolve(sock, "nl80211");

/* メッセージを組み立てて送信する */
struct nl_msg *msg = nlmsg_alloc();
genlmsg_put(msg, NL_AUTO_PORT, NL_AUTO_SEQ, family, 0, 0,
            NL80211_CMD_GET_INTERFACE, 0);
nla_put_u32(msg, NL80211_ATTR_IFINDEX, if_nametoindex("wlan0"));

nl_send_auto(sock, msg);
nlmsg_free(msg);

/* 応答を受信して解析する */
nl_recvmsgs_default(sock);
```

## カーネル側: genetlinkファミリの登録 {#kernel-side-registering-a-genetlink-family}

```c
/* net/wireless/nl80211.c（簡略版） */
#include <net/genetlink.h>

static const struct genl_ops nl80211_ops[] = {
    {
        .cmd    = NL80211_CMD_GET_INTERFACE,
        .doit   = nl80211_get_interface,
        .dumpit = nl80211_dump_interface,
        .policy = nl80211_policy,
    },
    {
        .cmd    = NL80211_CMD_SET_INTERFACE,
        .doit   = nl80211_set_interface,
        .flags  = GENL_ADMIN_PERM,  /* CAP_NET_ADMIN が必要 */
    },
    /* ... ほかにも多数 ... */
};

static struct genl_family nl80211_fam __ro_after_init = {
    .name     = NL80211_GENL_NAME,   /* "nl80211" */
    .version  = 1,
    .maxattr  = NL80211_ATTR_MAX,
    .policy   = nl80211_policy,
    .module   = THIS_MODULE,
    .ops      = nl80211_ops,
    .n_ops    = ARRAY_SIZE(nl80211_ops),
    .mcgrps   = nl80211_mcgrps,      /* マルチキャストグループ */
    .n_mcgrps = ARRAY_SIZE(nl80211_mcgrps),
};

static int __init nl80211_init(void)
{
    return genl_register_family(&nl80211_fam);
}
```

## Netlinkマルチキャスト: イベント通知 {#netlink-multicast-event-notifications}

Netlinkマルチキャストにより、カーネルは関心のあるユーザー空間プロセスにイベントをプッシュできる:

```c
/* 経路変更イベントを購読する */
struct sockaddr_nl addr = {
    .nl_family = AF_NETLINK,
    .nl_groups = RTMGRP_LINK        /* インターフェースの変更 */
              | RTMGRP_IPV4_IFADDR  /* アドレスの変更 */
              | RTMGRP_IPV4_ROUTE,  /* 経路の変更 */
};
bind(fd, (struct sockaddr *)&addr, sizeof(addr));

/* これで recv() は、要求を送らなくても RTM_NEWLINK、RTM_NEWADDR、
   RTM_NEWROUTE のイベントを受信するようになる */

/* systemd-networkd、NetworkManager、dhclient はこれを使って
   ケーブルの抜き差しや DHCP リースの変更などを検出する */
```

### カーネル側: マルチキャスト通知の送信 {#kernel-side-sending-a-multicast-notification}

```c
/* net/core/rtnetlink.c */
/* インターフェースが up になったときに呼ばれる: */
void rtmsg_ifinfo(int type, struct net_device *dev, unsigned int change,
                   gfp_t flags)
{
    struct sk_buff *skb = rtmsg_ifinfo_build_skb(type, dev, change,
                                                    0, flags, NULL, 0, 0);
    if (skb) {
        /* RTNLGRP_LINK マルチキャストグループに送る */
        rtnl_notify(skb, dev_net(dev), 0, RTNLGRP_LINK, NULL, flags);
    }
}
```

## NETLINK_SOCK_DIAG: ソケットの統計 {#netlink_sock_diag-socket-statistics}

`ss`が`/proc/net/tcp`より高速にソケットを一覧表示するために使う:

```c
/* TCP ソケットのダンプを要求する */
struct {
    struct nlmsghdr nlh;
    struct inet_diag_req_v2 req;
} request = {
    .nlh = {
        .nlmsg_len   = sizeof(request),
        .nlmsg_type  = SOCK_DIAG_BY_FAMILY,
        .nlmsg_flags = NLM_F_REQUEST | NLM_F_DUMP,
    },
    .req = {
        .sdiag_family   = AF_INET,
        .sdiag_protocol = IPPROTO_TCP,
        .idiag_states   = (1 << TCP_ESTABLISHED) | (1 << TCP_LISTEN),
        .idiag_ext      = (1 << (INET_DIAG_INFO - 1)),  /* tcp_info を取得 */
    },
};
send(fd, &request, sizeof(request), 0);
/* 受信: ソケットごとに1つの INET_DIAG_INFO */
```

## 参考文献 {#further-reading}

- [ソケット層の概要](socket-layer.md) — AF_NETLINKソケットの実装
- [procfsとsysfs](../vfs/procfs-sysfs.md) — カーネルとユーザー空間の間の別のインターフェース
- [XDPとAF_XDP](af-xdp.md) — ソケットベースのパケット処理
- [ネットワークのデバッグ](net-debugging.md) — `ss`と`ip`コマンドの内部
- `net/netlink/` — netlinkのコア実装
- `net/core/rtnetlink.c` — rtnetlink
- `net/wireless/nl80211.c` — nl80211のgenetlinkファミリ
- `libnl3`のドキュメント — ユーザー空間のnetlinkライブラリ
