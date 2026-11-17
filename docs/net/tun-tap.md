# TUN/TAP仮想ネットワークデバイス {#tuntap-virtual-network-devices}

> ユーザー空間ネットワーキング: VPN、QEMU、コンテナネットワークの仕組み

## TUNとTAPとは何か {#what-are-tun-and-tap}

TUNとTAPは、完全にソフトウェアで実装された仮想ネットワークデバイスである。ユーザー空間のプログラムは、ファイルディスクリプタへの読み書きによって**ネットワークトラフィックを送受信**できる。

```
Normal NIC:             Wire (hardware)
     ↕                         ↕
  kernel network stack  ←→  NIC driver
     ↕
  Application (socket)

TUN/TAP:               Userspace program (VPN, VM, etc.)
     ↕                         ↕
  kernel network stack  ←→  /dev/net/tun (file descriptor)
     ↕
  Application (socket)
```

| 機能 | TUN | TAP |
|---------|-----|-----|
| 層 | レイヤ3（IP） | レイヤ2（Ethernet） |
| フレーム | 生のIPパケット | 完全なEthernetフレーム（MAC付き） |
| ユースケース | VPNトンネル（OpenVPN、WireGuard） | VMネットワーキング（QEMU）、ブリッジ |

## TUNインターフェースの作成 {#creating-a-tun-interface}

```c
#include <linux/if_tun.h>
#include <net/if.h>
#include <sys/ioctl.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

int tun_alloc(char *dev, int flags)
{
    struct ifreq ifr = {};
    int fd, err;

    /* クローンデバイスを開く */
    fd = open("/dev/net/tun", O_RDWR);
    if (fd < 0)
        return fd;

    /* IFF_TUN  = TUN（IP）デバイス */
    /* IFF_TAP  = TAP（Ethernet）デバイス */
    /* IFF_NO_PI = パケット情報ヘッダを先頭に付けない */
    ifr.ifr_flags = flags;  /* 例: IFF_TUN | IFF_NO_PI */
    if (*dev)
        strncpy(ifr.ifr_name, dev, IFNAMSIZ);

    /* TUN/TAP デバイスを登録する */
    err = ioctl(fd, TUNSETIFF, &ifr);
    if (err < 0) {
        close(fd);
        return err;
    }

    /* dev には割り当てられたインターフェース名（例: "tun0"）が入っている */
    strcpy(dev, ifr.ifr_name);
    return fd;  /* fd は tun インターフェースに結び付けられている */
}

/* 使い方: */
char tun_name[IFNAMSIZ] = "tun0";
int tun_fd = tun_alloc(tun_name, IFF_TUN | IFF_NO_PI);

/* インターフェースを設定する（ip link set tun0 up に相当） */
/* ip addr add 10.0.0.1/24 dev tun0 */
```

## パケットの読み書き {#reading-and-writing-packets}

```c
/* 読み込み: TUN インターフェースにパケットが届いた */
uint8_t buf[65536];
ssize_t n = read(tun_fd, buf, sizeof(buf));
/* buf[0..n-1] は生の IP パケット（IFF_NO_PI の場合） */

/* 書き込み: カーネルネットワークスタックにパケットを注入する */
ssize_t sent = write(tun_fd, packet, packet_len);
/* カーネルはこれを tun0 にパケットが届いたかのように扱う */
```

## 最小限のVPNサーバーの実装 {#minimal-vpn-server-implementation}

```c
/* 簡略版: TUN を使ったユーザー空間 VPN */

#define REMOTE_PORT 51820
#define MTU         1500

int main(void)
{
    /* TUN インターフェースを作成 */
    char tun_name[IFNAMSIZ] = "vpn0";
    int tun_fd = tun_alloc(tun_name, IFF_TUN | IFF_NO_PI);

    /* リモートの相手への UDP ソケット */
    int udp_fd = socket(AF_INET, SOCK_DGRAM, 0);
    struct sockaddr_in remote = {
        .sin_family = AF_INET,
        .sin_port = htons(REMOTE_PORT),
    };
    inet_pton(AF_INET, "192.168.1.1", &remote.sin_addr);

    uint8_t buf[MTU + 28];  /* IP ヘッダ用の余白 */

    /* メインの転送ループ */
    struct pollfd fds[2] = {
        { .fd = tun_fd, .events = POLLIN },
        { .fd = udp_fd, .events = POLLIN },
    };

    while (1) {
        poll(fds, 2, -1);

        if (fds[0].revents & POLLIN) {
            /* ローカルのアプリケーションからのパケット → 暗号化して相手に送る */
            ssize_t n = read(tun_fd, buf, MTU);
            /* encrypt(buf, n) */
            sendto(udp_fd, buf, n, 0,
                   (struct sockaddr *)&remote, sizeof(remote));
        }

        if (fds[1].revents & POLLIN) {
            /* リモートの相手からのパケット → 復号してローカルに注入する */
            ssize_t n = recv(udp_fd, buf, sizeof(buf), 0);
            /* decrypt(buf, n) */
            write(tun_fd, buf, n);
        }
    }
}
```

## WireGuardの仮想インターフェース {#wireguards-virtual-interface}

WireGuardは、汎用のTUNドライバ**ではなく**、独自のnetdev型
（`wireguard`）を登録するカーネルモジュールとして実装されている。自身のnet_deviceを
直接管理し、`/dev/net/tun`を完全に迂回する。

```bash
# WireGuard は独自の netdev 型で仮想インターフェース（wg0）を作成する
ip link add wg0 type wireguard
wg set wg0 private-key /etc/wireguard/privatekey \
           listen-port 51820 \
           peer <pubkey> allowed-ips 0.0.0.0/0 endpoint 1.2.3.4:51820
ip link set wg0 up
ip addr add 10.0.0.2/24 dev wg0

# 10.0.0.0/24 宛てのトラフィック → WireGuard が暗号化 → 1.2.3.4:51820 へ UDP
# 受信した UDP → WireGuard が復号 → wg0 に届いたように見える
```

## VMネットワーキングのためのTAP（QEMU） {#tap-for-vm-networking-qemu}

```bash
# QEMU は完全な L2 ネットワーキング（Ethernet フレーム）に TAP を使う:
ip tuntap add mode tap tap0
ip link set tap0 up

# tap0 を物理 eth0 とブリッジする:
ip link add br0 type bridge
ip link set eth0 master br0
ip link set tap0 master br0
ip link set br0 up

# QEMU は tap0 を使う:
qemu-system-x86_64 \
    -netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
    -device virtio-net-pci,netdev=net0 \
    ...
# ゲストは完全な Ethernet インターフェースを得て、ブリッジ上に現れる
# LAN から DHCP を受けたり、br0 経由で他の VM と通信したりできる
```

## カーネルの実装 {#kernel-implementation}

### struct tun_struct {#struct-tun_struct}

```c
/* drivers/net/tun.c */
struct tun_struct {
    struct tun_file __rcu   *tfiles[MAX_TAP_QUEUES]; /* ファイルディスクリプタ */
    unsigned int             numqueues;

    struct net_device       *dev;      /* tun0/tap0 の netdev */
    netdev_features_t        set_features;

    int                      vnet_hdr_sz;  /* vhost/virtio のゼロコピー用 */
    int                      sndbuf;
    unsigned long            flags;
    /* IFF_TUN, IFF_TAP, IFF_NO_PI, IFF_VNET_HDR, IFF_MULTI_QUEUE */

    struct sock_fprog        fprog;    /* 受信パケットに対する BPF フィルタ */
    bool                     filter_attached;

    u32                      msg_enable;
    spinlock_t               lock;
    struct hlist_head        flows[TUN_NUM_FLOW_ENTRIES]; /* フローテーブル */
    struct timer_list        flow_gc_timer;

    struct net               *net;     /* ネットワーク namespace */
};
```

### パケットの流れ: TUNのfdへの書き込み → カーネル {#packet-flow-write-to-tun-fd-kernel}

```c
/* ユーザー空間が TUN のファイルディスクリプタにパケットを書き込んだとき: */
static ssize_t tun_chr_write_iter(struct kiocb *iocb, struct iov_iter *from)
{
    struct tun_file *tfile = file->private_data;
    struct tun_struct *tun = tun_get(tfile);
    struct sk_buff *skb;

    /* ユーザー空間のデータから sk_buff を組み立てる */
    skb = tun_alloc_skb(tfile, align, len, gso.hdr_len, noblock);
    if (copy_from_iter(skb_put(skb, len), len, from) != len)
        goto drop;

    /* skb->protocol を設定する:
     * TAP: eth_type_trans() が Ethernet ヘッダを解析して EtherType を得る。
     * TUN: プロトコルは IP バージョンのバイトから導出される（0x40→AF_INET、
     *      0x60→AF_INET6）。TUN では eth_type_trans() は呼ばれない。 */
    if (tun->flags & IFF_TAP)
        skb->protocol = eth_type_trans(skb, tun->dev);
    else
        skb->protocol = (skb->data[0] & 0xf0) == 0x60 ?
                        htons(ETH_P_IPV6) : htons(ETH_P_IP);

    /* カーネルネットワークスタックに注入する */
    netif_rx(skb);  /* → IP スタックを上ってソケットに届く */

    return len;
}
```

### パケットの流れ: カーネル → TUNのfdからの読み込み {#packet-flow-kernel-read-from-tun-fd}

```c
/* パケットが TUN デバイスにルーティングされたとき（ユーザー空間に渡される）: */
static netdev_tx_t tun_net_xmit(struct sk_buff *skb, struct net_device *dev)
{
    struct tun_struct *tun = netdev_priv(dev);
    struct tun_file *tfile;

    /* 適切なキューを選ぶ（マルチキュー TUN の場合） */
    tfile = rcu_dereference(tun->tfiles[txq]);

    /* sk_buff をファイルディスクリプタの受信キューに入れる */
    if (ptr_ring_produce(&tfile->tx_ring, skb))
        goto drop;

    /* ユーザー空間を起床させる（poll/read が POLLIN を返す） */
    wake_up_interruptible_poll(&tfile->wq.wait, EPOLLIN | EPOLLRDNORM);

    return NETDEV_TX_OK;
drop:
    dev_kfree_skb(skb);
    return NETDEV_TX_OK;
}
```

## マルチキューTUN（性能向上のため） {#multiqueue-tun-for-performance}

```c
/* 複数の CPU キューを使うためにマルチキューを有効化する: */
ifr.ifr_flags = IFF_TUN | IFF_NO_PI | IFF_MULTI_QUEUE;

/* 複数の fd を開く。それぞれが別のキューになる: */
int fds[NUM_QUEUES];
for (int i = 0; i < NUM_QUEUES; i++) {
    char name[IFNAMSIZ] = "tun0";
    fds[i] = tun_alloc(name, IFF_TUN | IFF_NO_PI | IFF_MULTI_QUEUE);
    /* 各 fd は異なる CPU のスレッドに固定される */
}
/* カーネルはフローの5タプルに基づいてパケットをキューにハッシュする */
```

## vhost: QEMUのためのカーネル高速化TUN {#vhost-kernel-accelerated-tun-for-qemu}

VMでは、**vhost**がTUNのパケット転送をQEMUのユーザー空間からカーネルに移し、コンテキストスイッチを排除する:

```
Without vhost:
  Guest (VM) → virtio device → KVM VM exit → QEMU → write(tap_fd) → kernel

With vhost:
  Guest (VM) → virtio device → KVM VM exit → kernel vhost worker → tap_fd
  (no QEMU context switch for data path!)
```

```bash
# QEMU は virtio-net に対して自動的に vhost を使う:
qemu-system-x86_64 \
    -netdev tap,id=net0,vhost=on \
    -device virtio-net-pci,netdev=net0 \
    ...
```

## TUN/TAPの観察 {#observing-tuntap}

```bash
# TUN/TAP インターフェースを一覧表示:
ip tuntap list
# tun0: tun MULTI_QUEUE

# インターフェースの統計:
ip -s link show tun0

# TUN の書き込み（ユーザー空間からのパケット）をトレース:
bpftrace -e '
kprobe:tun_do_read {
    printf("TUN read: pid=%d dev=%s\n", pid,
           ((struct tun_struct *)arg0)->dev->name);
}'

# TUN インターフェースで tcpdump:
tcpdump -i tun0 -n

# vhost ワーカーの活動を確認:
ps aux | grep vhost
cat /proc/$(pgrep vhost)/status
```

## 参考文献 {#further-reading}

- [virtio](../virtualization/virtio.md) — TAPを使った準仮想化ネットワーキング
- [ネットワークnamespace](net-namespaces.md) — vethペアによる分離されたネットワークスタック
- [NAPI](napi.md) — 実NICの受信処理
- [sk_buff](sk-buff.md) — パケットのデータ構造
- [XDP](xdp.md) — カーネルスタックを完全に迂回する
- `drivers/net/tun.c` — TUN/TAPの完全な実装
