# パケットの一生（送信パス） {#life-of-a-packet-transmit-path}

> アプリケーションのwrite()からNICのDMAまで、ステップごとに

## 概要 {#overview}

アプリケーションがTCPソケットにデータを書き込むと、カーネルはそのデータを1つ以上のIPパケットに変換して送信する。`write()`から回線までのパスは次のとおり:

```
Application: write() / send()
    ↓
tcp_sendmsg()         copy data into sk_buff chain
    ↓
tcp_write_xmit()      TCP segmentation, window check
    ↓
tcp_transmit_skb()    build TCP header
    ↓
ip_queue_xmit()       build IP header, routing lookup
    ↓ Netfilter: OUTPUT
    ↓ Netfilter: POSTROUTING
neigh_output()        ARP resolution → Ethernet header
    ↓
dev_queue_xmit()      qdisc (traffic control)
    ↓
driver ndo_start_xmit() DMA to NIC
    ↓
NIC transmits packet
```

## フェーズ1: tcp_sendmsg() — ユーザー空間からのデータ {#phase-1-tcp_sendmsg-data-from-userspace}

```c
// net/ipv4/tcp.c:1460
int tcp_sendmsg(struct sock *sk, struct msghdr *msg, size_t size)
{
    lock_sock(sk);
    err = tcp_sendmsg_locked(sk, msg, size);
    release_sock(sk);
    return err;
}

int tcp_sendmsg_locked(struct sock *sk, struct msghdr *msg, size_t size)
{
    while (msg_data_left(msg)) {
        // 送信用 sk_buff を取得、または割り当てる
        skb = tcp_write_queue_tail(sk);
        if (!skb || skb_tailroom(skb) == 0) {
            skb = sk_stream_alloc_skb(sk, 0, sk->sk_allocation, ...);
            tcp_add_write_queue_tail(sk, skb);
        }

        // ユーザー空間から skb にデータをコピー
        // copy_from_iter() を使う — 設定されていれば MSG_ZEROCOPY でゼロコピー
        copy = skb_copy_to_page_nocache(sk, &msg->msg_iter, skb, ...);
        copied += copy;
    }

    // すぐに送信を試みる
    tcp_push(sk, flags, mss_now, ...);
}
```

データは、TCPが送信を決めるまで`sk->sk_write_queue`に保持される。

## フェーズ2: TCPの書き出し: tcp_write_xmit() {#phase-2-tcp-write-tcp_write_xmit}

TCPは次の要素に基づいて*いつ*送信するかを決める:

- 輻輳ウィンドウ（`tp->snd_cwnd`）
- 相手が広告した受信ウィンドウ（`tp->snd_wnd`）
- Nagleアルゴリズム（小さなパケットを遅らせる）
- 送信バッファの空き

```c
// net/ipv4/tcp_output.c
static bool tcp_write_xmit(struct sock *sk, unsigned int mss_now, ...)
{
    while ((skb = tcp_send_head(sk))) {
        // 輻輳ウィンドウを確認
        cwnd_quota = tcp_cwnd_test(tp, skb);
        if (!cwnd_quota)
            break;

        // 受信ウィンドウを確認
        if (!tcp_snd_wnd_test(tp, skb, mss_now))
            break;

        // skb が MSS より大きければ分割する
        if (skb->len > mss_now)
            tcp_mtu_probe(sk) || tcp_fragment(sk, skb, mss_now, ...);

        tcp_transmit_skb(sk, skb, 1, GFP_ATOMIC);
    }
}
```

## フェーズ3: TCPヘッダの構築 — tcp_transmit_skb() {#phase-3-build-tcp-header-tcp_transmit_skb}

```c
// net/ipv4/tcp_output.c
static int tcp_transmit_skb(struct sock *sk, struct sk_buff *skb, int clone_it, ...)
{
    // skb を clone する（元の skb は再送のために書き込みキューに残る）
    if (clone_it)
        skb = skb_clone(skb, gfp_mask);

    // TCP ヘッダ用の領域を確保
    skb_push(skb, tcp_header_size);
    skb_reset_transport_header(skb);

    // TCP ヘッダを埋める
    th = tcp_hdr(skb);
    th->source = inet->inet_sport;
    th->dest   = inet->inet_dport;
    th->seq    = htonl(tcb->seq);
    th->ack_seq = htonl(tp->rcv_nxt);
    th->doff   = tcp_header_size >> 2;
    th->window = htons(tcp_select_window(sk));
    // ... オプション（SACK、タイムスタンプなど）

    // チェックサムを計算（またはハードウェアオフロードに任せる）
    tcp_set_skb_tso_segs(skb, mss_now);

    // IP 層に渡す
    icsk->icsk_af_ops->queue_xmit(sk, skb, &inet->cork.fl);
    // → ip_queue_xmit()
}
```

## フェーズ4: IP層 — ip_queue_xmit() {#phase-4-ip-layer-ip_queue_xmit}

```c
// net/ipv4/ip_output.c:546
int ip_queue_xmit(struct sock *sk, struct sk_buff *skb, struct flowi *fl)
{
    // 経路検索（sk->sk_dst_cache にキャッシュされる）
    rt = (struct rtable *)__sk_dst_check(sk, 0);
    if (!rt) {
        rt = ip_route_output_ports(net, fl4, sk, ...);
        sk_setup_caps(sk, &rt->dst);
    }
    skb_dst_set_noref(skb, &rt->dst);

    // IP ヘッダを構築
    skb_push(skb, sizeof(struct iphdr));
    skb_reset_network_header(skb);
    iph = ip_hdr(skb);
    iph->version  = 4;
    iph->ihl      = 5;
    iph->tos      = inet->tos;
    iph->tot_len  = htons(skb->len);
    iph->id       = htons(inet->inet_id++);
    iph->ttl      = ip_select_ttl(inet, &rt->dst);
    iph->protocol = sk->sk_protocol;
    iph->saddr    = fl4->saddr;
    iph->daddr    = fl4->daddr;
    ip_send_check(iph);  // チェックサムを計算

    // Netfilter: NF_INET_LOCAL_OUT フック（OUTPUT チェイン）
    return NF_HOOK(NFPROTO_IPV4, NF_INET_LOCAL_OUT, sk, skb, NULL,
                   rt->dst.dev, dst_output);
}
```

## フェーズ5: NetfilterのOUTPUTとルーティング {#phase-5-netfilter-output-and-routing}

`dst_output()`は`ip_output()`を呼ぶ:

```c
// net/ipv4/ip_output.c:428
int ip_output(struct net *net, struct sock *sk, struct sk_buff *skb)
{
    struct net_device *dev = skb_dst(skb)->dev;
    skb->dev = dev;
    skb->protocol = htons(ETH_P_IP);

    // Netfilter: NF_INET_POST_ROUTING フック（POSTROUTING チェイン）
    // （SNAT はここで行われる）
    return NF_HOOK_COND(NFPROTO_IPV4, NF_INET_POST_ROUTING, sk, skb,
                        NULL, dev, ip_finish_output, ...);
}

static int ip_finish_output(struct net *net, struct sock *sk, struct sk_buff *skb)
{
    // 必要ならフラグメンテーション（skb->len > dev->mtu）
    if (skb->len > ip_skb_dst_mtu(sk, skb))
        return ip_fragment(net, sk, skb, mtu, ip_finish_output2);

    return ip_finish_output2(net, sk, skb);
}
```

## フェーズ6: ARPとEthernet — neigh_output() {#phase-6-arp-and-ethernet-neigh_output}

`ip_finish_output2()`はネクストホップのMACアドレスを解決する:

```c
// net/ipv4/ip_output.c
static int ip_finish_output2(struct net *net, struct sock *sk, struct sk_buff *skb)
{
    struct neighbour *neigh;

    // 近隣エントリの検索（ARP テーブル）
    neigh = ip_neigh_for_gw(rt, skb, &is_v6gw);

    // MAC が既知なら neigh->output = neigh_hh_output()（高速パス）
    // → Ethernet ヘッダを付加し dev_queue_xmit() を呼ぶ
    // 近隣が未解決なら ARP を発行し、パケットをキューに入れる
    return neigh_output(neigh, skb, is_v6gw);
}
```

ARPエントリが存在しない場合、ARP要求が送信され、応答が届くまでパケットはキューに入れられる。

## フェーズ7: トラフィック制御 — dev_queue_xmit() {#phase-7-traffic-control-dev_queue_xmit}

```c
// net/core/dev.c
int dev_queue_xmit(struct sk_buff *skb)
{
    struct net_device *dev = skb->dev;
    struct netdev_queue *txq;

    // 送信キューを選択（XPS: Transmit Packet Steering）
    txq = netdev_core_pick_tx(dev, skb, ...);

    // qdisc を通過させる（トラフィック制御、レート制限）
    // デフォルト: pfifo_fast（優先度付き FIFO）
    // カスタム: tbf、htb、fq_codel など
    rc = __dev_xmit_skb(skb, q, dev, txq);
    // q->enqueue() → q->dequeue() → dev_hard_start_xmit()
}
```

qdiscは次のことができる:

- パケットを破棄する（TBFやポリシングによるレート制限）
- パケットの順序を入れ替える（FQスケジューリング）
- パケットを遅延させる（テスト用のnetem）
- トラフィックをシェーピングする（HTB、TBF）

## フェーズ8: ドライバのDMA — ndo_start_xmit() {#phase-8-driver-dma-ndo_start_xmit}

```c
// ドライバの実装
static netdev_tx_t my_nic_xmit(struct sk_buff *skb, struct net_device *dev)
{
    // skb のデータを DMA 用にマップする
    dma_addr = dma_map_single(dev, skb->data, skb->len, DMA_TO_DEVICE);

    // TX リングにディスクリプタを書き込む
    ring->desc[tail].dma_addr = dma_addr;
    ring->desc[tail].len = skb->len;
    ring->desc[tail].cmd = CMD_EOP | CMD_IFCS;  // パケット末尾、FCS を挿入

    // リングの tail を進める
    ring->tail = (tail + 1) % ring->size;

    // ドアベルを鳴らす: 新しいディスクリプタを NIC に通知
    writel(ring->tail, ring->tail_reg);

    return NETDEV_TX_OK;
}
```

NICはパケットを送信した後、TX完了割り込みを発生させる。ドライバはそこで`dev_consume_skb_any(skb)`を呼んで`sk_buff`を解放する。

## TX完了と書き込み可能領域の通知 {#tx-completion-and-write-space-wakeup}

```c
// TX 完了割り込みハンドラ
static void my_nic_tx_clean(struct my_nic_ring *ring)
{
    while (ring->next_to_clean != ring->next_to_use) {
        skb = ring->tx_buf[ring->next_to_clean].skb;
        dma_unmap_single(...);
        dev_consume_skb_any(skb);  // sk_buff を解放
        ring->next_to_clean++;
    }
    // ソケットの送信バッファが空いたら、書き込み側を起床させる
    netif_wake_queue(ring->dev);
}
```

`netif_wake_queue()`は最終的に`sk->sk_write_space(sk)`を呼び、送信バッファの空きを待って`sendmsg()`でブロックしているタスクを起床させる。

## 送信バッファとバックプレッシャー {#send-buffer-and-backpressure}

カーネルは、送信バッファが許す以上のデータをコピーしない:

```bash
# TCP 送信バッファのデフォルトサイズ [min, default, max]
cat /proc/sys/net/ipv4/tcp_wmem
# → 4096 16384 4194304

# ソケットごとの上書き
setsockopt(fd, SOL_SOCKET, SO_SNDBUF, &size, sizeof(size));

# 現在の使用量を確認（Send-Q 列）
ss -tn  # Send-Q が 0 以外 = 送信バッファ内で送信待ちのデータがある
```

`sk->sk_wmem_alloc >= sk->sk_sndbuf`の場合、`tcp_sendmsg()`はTX完了によって空きができるまでブロックする。

## 参考文献 {#further-reading}

- [sk_buff](sk-buff.md) — tcp_sendmsgで構築されるパケット構造体
- [パケットの一生（受信）](life-of-packet-rx.md) — 受信側のパス
- [TCPの実装](tcp.md) — 輻輳制御とウィンドウ管理
- [ソケット層の概要](socket-layer.md) — sendmsgのディスパッチの連鎖
- [TCとqdisc](tc-qdisc.md) — フェーズ7のトラフィック制御
