# UDPソケットの内部 {#udp-socket-internals}

> struct udp_sock、sendmsg/recvmsgのパス、UDPセグメンテーションオフロード、マルチキャスト

## カーネル内のUDP {#udp-in-the-kernel}

UDPはコネクションレスである。ハンドシェイクも再送もフロー制御もない。カーネルの仕事は最小限で、UDPヘッダを付加してパケットをIPに渡すだけである。

```
Application:
    sendto(fd, buf, len, 0, &dest, sizeof(dest))
         │
    ┌────▼────────────────────────────────────────────────────┐
    │ sock_sendmsg → udp_sendmsg                              │
    │   1. Build sk_buff                                      │
    │   2. Set UDP header (src port, dst port, len, checksum) │
    │   3. ip_make_skb → route lookup                         │
    │   4. udp_send_skb → ip_send_skb                         │
    └─────────────────────────────────────────────────────────┘
         │
    IP layer → routing → Ethernet → wire
```

## struct udp_sock {#struct-udp_sock}

```c
/* include/linux/udp.h */
struct udp_sock {
    /* inet_sock は先頭でなければならない: */
    struct inet_sock inet;   /* 最初のメンバとして struct sock sk を含む */

    int       pending;       /* 保留中のフレームがあるか */
    unsigned int corkflag;   /* UDP_CORK ソケットオプションによる cork */
    __u8      encap_type;    /* UDP カプセル化ソケットか */
    unsigned char no_check6_tx:1, no_check6_rx:1;
    unsigned char encap_enabled:1;
    unsigned char gro_enabled:1;
    unsigned char accept_udp_l4:1;
    unsigned char accept_udp_fraglist:1;

    __u16     len;           /* 保留中のフレームの合計長 */
    __u16     gso_size;      /* UDP GSO のフラグメントサイズ */

    /* 受信パス: */
    int       (*encap_rcv)(struct sock *sk, struct sk_buff *skb);
    int       (*encap_err_rcv)(struct sock *sk, struct sk_buff *skb, int err,
                               __be16 port, u32 info, u8 *payload);
    void      (*encap_destroy)(struct sock *sk);
    struct sk_buff *(*gro_receive)(struct sock *sk, struct list_head *head,
                                   struct sk_buff *skb);
    int       (*gro_complete)(struct sock *sk, struct sk_buff *skb, int nhoff);
};
```

## UDPの送信パス {#udp-send-path}

### udp_sendmsg {#udp_sendmsg}

```c
/* net/ipv4/udp.c */
int udp_sendmsg(struct sock *sk, struct msghdr *msg, size_t len)
{
    struct inet_sock *inet = inet_sk(sk);
    struct udp_sock  *up   = udp_sk(sk);
    struct flowi4     fl4;
    struct rtable    *rt = NULL;
    struct sk_buff   *skb;

    /* 1. 宛先アドレスとポートを決定する */
    if (msg->msg_name) {
        struct sockaddr_in *usin = (struct sockaddr_in *)msg->msg_name;
        daddr = usin->sin_addr.s_addr;
        dport = usin->sin_port;
    } else {
        /* 接続済みソケット: キャッシュされた dst を使う */
        daddr = inet->inet_daddr;
        dport = inet->inet_dport;
    }

    /* 2. 経路検索 */
    flowi4_init_output(&fl4, sk->sk_bound_dev_if, sk->sk_mark,
                       tos, RT_SCOPE_UNIVERSE, IPPROTO_UDP,
                       sk->sk_flags, daddr, saddr, dport, sport, ...);
    rt = ip_route_output_flow(net, &fl4, sk);

    /* 3. UDP ヘッダ + データを持つ sk_buff を組み立てる */
    skb = ip_make_skb(sk, &fl4, getfrag, msg, ulen,
                      sizeof(struct udphdr), &ipc, &rt,
                      msg->msg_flags);

    /* 4. UDP ヘッダを付加する */
    struct udphdr *uh = udp_hdr(skb);
    uh->source = inet->inet_sport;
    uh->dest   = dport;
    uh->len    = htons(ulen);
    uh->check  = 0;  /* チェックサムは下で計算する */

    /* 5. チェックサムを計算する（またはハードウェアにオフロード） */
    udp4_hwcsum(skb, fl4.saddr, fl4.daddr);

    /* 6. IP に渡す */
    return udp_send_skb(skb, &fl4, &cork.base);
}
```

### UDPのcork（MSG_MORE） {#udp-corking-msg_more}

```c
/* cork: 送信前にデータを蓄積する（TCP_CORK と同様） */
setsockopt(fd, IPPROTO_UDP, UDP_CORK, &one, sizeof(one));
/* または各 sendmsg に MSG_MORE フラグを付ける: */

/* 複数の小さな sendmsg 呼び出し → 1つの UDP パケット: */
sendmsg(fd, &msg1, MSG_MORE);  /* バッファされる */
sendmsg(fd, &msg2, MSG_MORE);  /* バッファされる */
sendmsg(fd, &msg3, 0);         /* フラッシュ: 3つすべてが1つの UDP パケットで送られる */
```

## UDPの受信パス {#udp-receive-path}

### 受信パケットの振り分け {#incoming-packet-routing}

```c
/* net/ipv4/udp.c */
int udp_rcv(struct sk_buff *skb)
{
    return __udp4_lib_rcv(skb, &udp_table, IPPROTO_UDP);
}

static int __udp4_lib_rcv(struct sk_buff *skb, struct udp_table *udptable,
                            int proto)
{
    struct udphdr *uh = udp_hdr(skb);
    __be32 saddr = ip_hdr(skb)->saddr;
    __be32 daddr = ip_hdr(skb)->daddr;
    __u16  ulen  = ntohs(uh->len);

    /* 1. UDP ヘッダとチェックサムを検証する */
    if (udp4_csum_init(skb, uh, proto))
        goto csum_error;

    /* 2. 宛先ポート/アドレスでソケットを検索する（ハッシュテーブル） */
    sk = __udp4_lib_lookup_skb(skb, uh->source, uh->dest, udptable);
    if (sk)
        return udp_unicast_rcv_skb(sk, skb, uh);

    /* 3. ソケットが見つからない — ICMP port unreachable を送る */
    icmp_send(skb, ICMP_DEST_UNREACH, ICMP_PORT_UNREACH, 0);
    return 0;
}
```

### ソケットの受信キューへの配送 {#delivering-to-socket-receive-queue}

```c
static int udp_queue_rcv_one_skb(struct sock *sk, struct sk_buff *skb)
{
    struct udp_sock *up = udp_sk(sk);

    /* BPF ソケットフィルタが設定されていれば実行する */
    if (sk_filter_trim_cap(sk, skb, sizeof(struct udphdr)))
        goto drop;

    /* sk->sk_receive_queue にエンキューする */
    if (likely(sk_receive_skb(sk, skb, 1) == NET_RX_SUCCESS))
        return 0;

    return -1;
}

/* recvmsg: ユーザー空間がキューから読む */
int udp_recvmsg(struct sock *sk, struct msghdr *msg, size_t len, ...)
{
    struct sk_buff *skb;

    /* パケットを待ってデキューする */
    skb = __skb_recv_udp(sk, flags, &off, &err);
    if (!skb)
        return err;

    /* ユーザー空間へコピーする */
    copied = skb->len - sizeof(struct udphdr) - off;
    err = skb_copy_datagram_msg(skb, sizeof(struct udphdr) + off, msg, copied);

    /* 要求されていれば送信元アドレスを返す（MSG_NAME） */
    if (msg->msg_name) {
        struct sockaddr_in *sin = (struct sockaddr_in *)msg->msg_name;
        sin->sin_addr.s_addr = ip_hdr(skb)->saddr;
        sin->sin_port        = udp_hdr(skb)->source;
    }

    return copied;
}
```

## UDPソケットのハッシュテーブル {#udp-socket-hash-table}

```c
/* ソケットはローカルポートをキーとするハッシュテーブルに格納される: */
struct udp_table {
    struct udp_hslot    *hash;      /* ポートによるハッシュ */
    struct udp_hslot    *hash2;     /* ポート+アドレスによるハッシュ（4タプル） */
    unsigned int         mask;
    unsigned int         log;
};

/* 検索: */
/* 1. {dest_port} でハッシュ → 候補リスト */
/* 2. 各候補についてアドレスの一致を確認 */
/* 3. 複数一致した場合は接続済みソケット（4タプル一致）を優先 */
```

## UDP GSO（Generic Segmentation Offload） {#udp-gso-generic-segmentation-offload}

高スループットのUDP（例: QUIC、ゲームサーバー）向けに、UDP GSOは1回のシステムコールで複数のUDPペイロードを送信できるようにする:

```c
/* 1400 バイトの UDP データグラム 10 個を1回のシステムコールで送る: */
char buf[10 * 1400];
fill_data(buf, sizeof(buf));

struct msghdr msg = {};
struct iovec iov = { .iov_base = buf, .iov_len = sizeof(buf) };
msg.msg_iov    = &iov;
msg.msg_iovlen = 1;

/* 宛先: */
struct sockaddr_in dest = { .sin_family = AF_INET, .sin_port = htons(9000) };
inet_pton(AF_INET, "192.168.1.1", &dest.sin_addr);
msg.msg_name = &dest;
msg.msg_namelen = sizeof(dest);

/* GSO 制御メッセージ: セグメントサイズ */
char cmsg_buf[CMSG_SPACE(sizeof(uint16_t))];
struct cmsghdr *cm = (struct cmsghdr *)cmsg_buf;
cm->cmsg_level = SOL_UDP;
cm->cmsg_type  = UDP_SEGMENT;
cm->cmsg_len   = CMSG_LEN(sizeof(uint16_t));
*(uint16_t *)CMSG_DATA(cm) = 1400;  /* セグメントサイズ */

msg.msg_control    = cmsg_buf;
msg.msg_controllen = sizeof(cmsg_buf);

sendmsg(fd, &msg, 0);
/* カーネルは 10 個の別々の UDP パケットを送る。システムコールは1回! */
```

### UDP GRO（Generic Receive Offload） {#udp-gro-generic-receive-offload}

```c
/* 受信側で UDP GRO を有効化する: */
int one = 1;
setsockopt(fd, SOL_UDP, UDP_GRO, &one, sizeof(one));

/* MSG_TRUNC 付きの recvmsg で結合されたバッファ全体を得る */
/* cmsg で UDP_GRO のセグメントサイズを確認する */
char cmsg_buf[256];
msg.msg_control    = cmsg_buf;
msg.msg_controllen = sizeof(cmsg_buf);

ssize_t n = recvmsg(fd, &msg, 0);
for (struct cmsghdr *cm = CMSG_FIRSTHDR(&msg); cm;
     cm = CMSG_NXTHDR(&msg, cm)) {
    if (cm->cmsg_level == SOL_UDP && cm->cmsg_type == UDP_GRO) {
        uint16_t seg_size = *(uint16_t *)CMSG_DATA(cm);
        /* n バイトを受信。各セグメントは seg_size バイト */
    }
}
```

## マルチキャスト {#multicast}

```c
/* マルチキャストグループに参加する: */
struct ip_mreq mreq = {
    .imr_multiaddr.s_addr = inet_addr("239.0.0.1"),
    .imr_interface.s_addr = INADDR_ANY,
};
setsockopt(fd, IPPROTO_IP, IP_ADD_MEMBERSHIP, &mreq, sizeof(mreq));

/* マルチキャストのポートに bind する: */
struct sockaddr_in addr = {
    .sin_family = AF_INET,
    .sin_port   = htons(5000),
    .sin_addr.s_addr = inet_addr("239.0.0.1"),  /* または INADDR_ANY */
};
bind(fd, (struct sockaddr *)&addr, sizeof(addr));

/* マルチキャストグループに送信する: */
struct sockaddr_in dest = {
    .sin_family = AF_INET,
    .sin_port   = htons(5000),
    .sin_addr.s_addr = inet_addr("239.0.0.1"),
};
sendto(fd, data, len, 0, (struct sockaddr *)&dest, sizeof(dest));
```

### カーネルのマルチキャスト配送 {#kernel-multicast-delivery}

```c
/* net/ipv4/udp.c: マルチキャストグループ内の複数のソケットに配送する */
static int __udp4_lib_mcast_deliver(struct net *net, struct sk_buff *skb,
                                     struct udphdr *uh, ...)
{
    /* このグループ/ポートを購読しているすべてのソケットを見つける */
    struct hlist_head *head = &udptable->hash[hash];
    struct sock *sk;

    sk_for_each_from(sk) {
        /* 確認: ポートが一致し、マルチキャストグループに参加し、インターフェースが一致 */
        if (!inet_mc_hash_match(sk, skb))
            continue;
        /* 受信ソケットごとに skb を clone する */
        struct sk_buff *skb2 = skb_clone(skb, GFP_ATOMIC);
        udp_queue_rcv_one_skb(sk, skb2);
    }
}
```

## UDPの性能チューニング {#udp-performance-tuning}

```bash
# 受信バッファを増やす（高負荷時の破棄を減らす）:
sysctl -w net.core.rmem_max=26214400        # 最大 25 MB
sysctl -w net.core.rmem_default=26214400
sysctl -w net.ipv4.udp_mem="102400 873800 16777216"

# 送信バッファを増やす:
sysctl -w net.core.wmem_max=26214400

# UDP の破棄の統計を確認:
cat /proc/net/snmp | grep Udp
# UdpInDatagrams, UdpNoPorts, UdpInErrors, UdpOutDatagrams
# UdpRcvbufErrors: ソケットの受信バッファが満杯で破棄された

netstat -su | grep -i error
ss -upe  # 破棄を含む UDP ソケットの統計を表示

# ソケットごとの統計:
ss -u -n -e | grep ":9000"

# UDP の破棄をトレース:
bpftrace -e '
tracepoint:udp:udp_fail_queue_rcv_skb {
    @drops = count();
}'
```

## 参考文献 {#further-reading}

- [TCP](tcp.md) — コネクション指向のトランスポート
- [ソケット層](socket-layer.md) — socket()の作成、bind、sendmsg
- [sk_buff](sk-buff.md) — パケットのデータ構造
- [NAPI](napi.md) — 受信のスケーリング
- [XDP](xdp.md) — 極限のUDPスループットのためにカーネルを迂回する
- `net/ipv4/udp.c` — UDPの実装
