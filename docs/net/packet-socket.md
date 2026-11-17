# AF_PACKET: rawソケットによるパケットキャプチャ {#af_packet-raw-socket-packet-capture}

> tcpdump、Wireshark、BPFフィルタリングはパケットキャプチャ層でどう動作するか

## AF_PACKETとは何か? {#what-is-af_packet}

`AF_PACKET`ソケットは、ユーザー空間にネットワークデバイスのフレーム層（レイヤ2）への直接アクセスを与える。アプリケーションは、カーネルのネットワークスタックが処理する前の生のEthernetフレーム（ヘッダを含む）を受け取る。

使用例:

- `tcpdump`と`Wireshark` — パケットキャプチャ
- `arping`、`arpwatch` — ARPユーティリティ
- ネットワークブートのエージェント（DHCP、PXE）
- 独自プロトコルの実装
- ネットワーク侵入検知システム

## AF_PACKETソケットの作成 {#creating-an-af_packet-socket}

```c
#include <sys/socket.h>
#include <linux/if_packet.h>
#include <net/ethernet.h>

/* SOCK_RAW: ヘッダ付きの生のEthernetフレームを受信する */
/* ETH_P_ALL: すべてのプロトコルをキャプチャする */
int sock = socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL));

/* 特定のインターフェースにバインドする: */
struct sockaddr_ll sll = {
    .sll_family   = AF_PACKET,
    .sll_protocol = htons(ETH_P_ALL),
    .sll_ifindex  = if_nametoindex("eth0"),
};
bind(sock, (struct sockaddr *)&sll, sizeof(sll));

/* プロミスキャスモードを設定: ユニキャスト以外も含むすべてのフレームを受信する: */
struct packet_mreq mreq = {
    .mr_ifindex = if_nametoindex("eth0"),
    .mr_type    = PACKET_MR_PROMISC,
};
setsockopt(sock, SOL_PACKET, PACKET_ADD_MEMBERSHIP, &mreq, sizeof(mreq));
```

## パケットの読み取り {#reading-packets}

```c
uint8_t buf[65536];
struct sockaddr_ll src_addr;
socklen_t addr_len = sizeof(src_addr);

ssize_t n = recvfrom(sock, buf, sizeof(buf), 0,
                     (struct sockaddr *)&src_addr, &addr_len);

/* buf[0..13]: Ethernetヘッダ
   buf[0..5]:  宛先MAC
   buf[6..11]: 送信元MAC
   buf[12..13]: EtherType（0x0800=IPv4、0x0806=ARP、0x86DD=IPv6）
   buf[14..n-1]: ペイロード */

struct ethhdr *eth = (struct ethhdr *)buf;
printf("src: %02x:%02x:%02x:%02x:%02x:%02x → "
       "dst: %02x:%02x:%02x:%02x:%02x:%02x "
       "proto: 0x%04x\n",
       eth->h_source[0], eth->h_source[1], eth->h_source[2],
       eth->h_source[3], eth->h_source[4], eth->h_source[5],
       eth->h_dest[0], eth->h_dest[1], eth->h_dest[2],
       eth->h_dest[3], eth->h_dest[4], eth->h_dest[5],
       ntohs(eth->h_proto));
```

## classic BPFソケットフィルタ {#classic-bpf-socket-filters}

`tcpdump`はフィルタをBPFバイトコードにコンパイルしてソケットにアタッチする。そのため、カーネルはマッチしたパケットだけをユーザー空間にコピーする:

```c
#include <linux/filter.h>

/* tcpdumpは"tcp port 80"を次のようなBPFバイトコードにコンパイルする: */
/* （tcpdump -d tcp port 80で生成） */
struct sock_filter filter[] = {
    /* IPプロトコルをロード（Ethernetフレームからのオフセット23） */
    { BPF_LD  | BPF_B   | BPF_ABS, 0, 0, 23 },
    /* TCP（6）か? */
    { BPF_JMP | BPF_JEQ | BPF_K, 0, 7, IPPROTO_TCP },
    /* 送信元ポートをロード */
    { BPF_LD  | BPF_H   | BPF_ABS, 0, 0, 34 },
    /* ポート80か? */
    { BPF_JMP | BPF_JEQ | BPF_K, 2, 0, 80 },
    /* 宛先ポートをロード */
    { BPF_LD  | BPF_H   | BPF_ABS, 0, 0, 36 },
    /* ポート80か? */
    { BPF_JMP | BPF_JEQ | BPF_K, 0, 3, 80 },
    /* マッチ: パケット全体を返す */
    { BPF_RET | BPF_K, 0, 0, 0xffffffff },
    /* マッチしない: ドロップ */
    { BPF_RET | BPF_K, 0, 0, 0 },
};

struct sock_fprog fprog = {
    .len    = sizeof(filter) / sizeof(filter[0]),
    .filter = filter,
};
setsockopt(sock, SOL_SOCKET, SO_ATTACH_FILTER, &fprog, sizeof(fprog));
/* これでTCPポート80のフレームだけがrecvfrom()に届く */
```

## PACKET_MMAP: ゼロコピーのリングバッファ {#packet_mmap-zero-copy-ring-buffer}

高性能なキャプチャのために、`PACKET_MMAP`はカーネルとユーザー空間で共有するリングバッファをマップする:

```c
/* TX/RXリングをセットアップ: */
struct tpacket_req3 tp = {
    .tp_block_size = 1 << 22,   /* ブロックあたり4MB */
    .tp_block_nr   = 64,         /* 64ブロック = 合計256MB */
    .tp_frame_size = 2048,       /* 最大フレームサイズ */
    .tp_frame_nr   = (1<<22) / 2048 * 64,
    /* パケットごとに1フレーム */

    /* V3の機能: */
    .tp_retire_blk_tov = 60,     /* 60ms後にブロックをフラッシュ */
    .tp_sizeof_priv    = 0,
    .tp_feature_req_word = TP_FT_REQ_FILL_RXHASH,
};
setsockopt(sock, SOL_PACKET, PACKET_RX_RING, &tp, sizeof(tp));

/* リングバッファをmmapする: */
size_t ring_size = tp.tp_block_size * tp.tp_block_nr;
void *ring = mmap(NULL, ring_size, PROT_READ | PROT_WRITE,
                  MAP_SHARED | MAP_LOCKED, sock, 0);

/* ブロックを処理する: */
uint8_t *block = ring;  /* ring[0]はブロック0 */
struct tpacket_block_desc *bh;

while (1) {
    bh = (struct tpacket_block_desc *)block;

    /* ブロックが準備できるのを待つ: */
    while (!(bh->hdr.bh1.block_status & TP_STATUS_USER))
        poll(&pfd, 1, -1);

    /* このブロック内のフレームをたどる: */
    struct tpacket3_hdr *ppd = (void *)block + bh->hdr.bh1.offset_to_first_pkt;
    for (int i = 0; i < bh->hdr.bh1.num_pkts; i++) {
        uint8_t *pkt = (uint8_t *)ppd + ppd->tp_mac;
        size_t  len  = ppd->tp_snaplen;
        /* pkt[0..len-1]のパケットを処理する */

        ppd = (void *)ppd + ppd->tp_next_offset;
    }

    /* ブロックをカーネルに返す: */
    bh->hdr.bh1.block_status = TP_STATUS_KERNEL;

    block += tp.tp_block_size;
    if (block >= (uint8_t *)ring + ring_size)
        block = ring;
}
```

## rawパケットの送信 {#sending-raw-packets}

```c
/* SOCK_RAW: Ethernetヘッダを手動で用意しなければならない */
uint8_t frame[1500];
struct ethhdr *eth = (struct ethhdr *)frame;

memcpy(eth->h_dest,   dest_mac, ETH_ALEN);
memcpy(eth->h_source, src_mac,  ETH_ALEN);
eth->h_proto = htons(ETH_P_IP);

/* frame[14..]にIP + トランスポートヘッダを書き込む */
/* ... */

struct sockaddr_ll dst = {
    .sll_family  = AF_PACKET,
    .sll_ifindex = if_nametoindex("eth0"),
    .sll_halen   = ETH_ALEN,
};
memcpy(dst.sll_addr, dest_mac, ETH_ALEN);

sendto(sock, frame, frame_len, 0,
       (struct sockaddr *)&dst, sizeof(dst));

/* SOCK_DGRAM: カーネルがEthernetヘッダを自動的に埋める */
int dgram_sock = socket(AF_PACKET, SOCK_DGRAM, htons(ETH_P_IP));
/* sll_addrに宛先MACを設定してsendto()する */
```

## カーネルの実装 {#kernel-implementation}

### パケットソケットの登録 {#packet-socket-registration}

```c
/* net/packet/af_packet.c */

static const struct proto_ops packet_ops = {
    .family   = PF_PACKET,
    .bind     = packet_bind,
    .recvmsg  = packet_recvmsg,
    .sendmsg  = packet_sendmsg,
    .poll     = packet_poll,
    .mmap     = packet_mmap,
    /* ... */
};

/* パケットが（netif_receive_skbから）到着したとき: */
static int packet_rcv(struct sk_buff *skb, struct net_device *dev,
                       struct packet_type *pt, struct net_device *orig_dev)
{
    struct sock *sk = pt->af_packet_priv;
    struct packet_sock *po = pkt_sk(sk);

    /* BPFフィルタを実行する: */
    if (po->has_vnet_hdr || skb_copy_datagram_from_iov(...)) {
        if (run_filter(skb, sk, snaplen) == 0)
            goto drop;
    }

    /* sk_buffをソケットの受信キューにコピーする */
    if (po->has_rx_ring) {
        /* PACKET_MMAPのパス: リングバッファに直接書き込む */
        packet_rcv_ring(skb, sk, snaplen);
    } else {
        /* 通常のパス: sk->sk_receive_queueにエンキューする */
        __skb_queue_tail(&sk->sk_receive_queue, skb);
        sk->sk_data_ready(sk);
    }
    return NET_RX_SUCCESS;
}
```

## tcpdumpの内部 {#tcpdump-under-the-hood}

```bash
# tcpdumpが何をしているかを見る:
strace tcpdump -i eth0 tcp port 80 2>&1 | head -30
# socket(AF_PACKET, SOCK_RAW|SOCK_CLOEXEC, 768) = 3  (ETH_P_ALL=0x0003; 768=htons(ETH_P_ALL) in little-endian)
# setsockopt(3, SOL_SOCKET, SO_ATTACH_FILTER, ...) = 0
# bind(3, {sa_family=AF_PACKET, sll_protocol=...}) = 0
# setsockopt(3, SOL_PACKET, PACKET_VERSION, ...) = 0
# setsockopt(3, SOL_PACKET, PACKET_RX_RING, ...) = 0
# mmap(NULL, ..., PROT_READ|PROT_WRITE, MAP_SHARED, 3, 0) = 0x7f...
# poll([{fd=3, events=POLLIN|POLLERR}], 1, -1) = 1

# BPFフィルタをコンパイルしてバイトコードを表示:
tcpdump -d tcp port 80
# (000) ldh      [12]
# (001) jeq      #0x86dd  jt 2  jf 8
# ...

# libpcapを直接使う場合:
# pcap_open_live() → socket(AF_PACKET, SOCK_RAW, ...)
# pcap_setfilter()  → setsockopt(SO_ATTACH_FILTER)
# pcap_loop()       → poll() + mmapリング
```

## 性能 {#performance}

```bash
# パケットのドロップを確認:
cat /proc/net/packet
# sk       RefCnt Type Proto  Iface   R Rmem   User   Inode
# ...

# ソケットごとの統計（受信キューが満杯によるドロップ）:
ss -p | grep packet

# PACKET_STATISTICSを使う:
struct tpacket_stats stats;
socklen_t len = sizeof(stats);
getsockopt(sock, SOL_PACKET, PACKET_STATISTICS, &stats, &len);
printf("received: %u, dropped: %u\n", stats.tp_packets, stats.tp_drops);

# 高性能: PACKET_MMAP v3リングを使う（パケットごとのシステムコールを回避）
# さらに高性能: カーネルバイパスのためにAF_XDPを使う
```

## 参考文献 {#further-reading}

- [XDP](xdp.md) — より高性能なカーネルバイパス
- [AF_XDP](af-xdp.md) — ゼロコピーのユーザー空間パケット処理
- [Netfilter](netfilter.md) — より上位の層でのパケットフィルタリング
- [sk_buff](sk-buff.md) — パケットのデータ構造
- [BPFネットワーキング](../bpf/bpf-networking.md) — tc/cgroup BPF
- `net/packet/af_packet.c` — AF_PACKETの実装
- `man 7 packet` — 詳細なmanページ
