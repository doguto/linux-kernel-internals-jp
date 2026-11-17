# パケットの一生（受信パス） {#life-of-a-packet-receive-path}

> NICのDMAからアプリケーションのread()まで、ステップごとに

## 概要 {#overview}

サーバーに到着したTCPパケットは、アプリケーションがデータを受け取るまでに、おおよそ8つの異なるフェーズを通過する。各フェーズは異なるカーネルサブシステムが担当し、同じ`sk_buff`がデータのコピーなしにそのすべてを通り抜ける。

```
NIC hardware
    ↓ DMA
Ring buffer
    ↓ interrupt / NAPI poll
sk_buff allocation
    ↓
Ethernet demux (__netif_receive_skb)
    ↓
IP input (ip_rcv)
    ↓ Netfilter: PREROUTING
    ↓ routing decision
    ↓ Netfilter: INPUT
TCP input (tcp_v4_rcv)
    ↓ socket lookup
    ↓ TCP state machine
Socket receive queue
    ↓ data_ready wakeup
Application: read() / recv()
```

## フェーズ1: NICのDMAと割り込み {#phase-1-nic-dma-and-interrupt}

NICは、受信したパケットをDMAによって事前に割り当てられた**リングバッファ**に置く。リングの各エントリ（ディスクリプタ）はカーネルのバッファを指している:

```c
// ドライバがバッファを事前に割り当てる（RXリングのセットアップ）
for each ring slot:
    buf = page_pool_alloc_pages(pool, GFP_ATOMIC);
    ring->desc[i].dma_addr = dma_map_page(dev, buf, ...);
```

パケットが到着すると:
1. NICがパケットデータをDMAバッファに書き込む
2. NICがリングディスクリプタの"done"ビットを立てる
3. NICがハードウェア割り込みを発生させる

## フェーズ2: NAPIポーリング {#phase-2-napi-polling}

ハードIRQハンドラは、それ以降のNIC割り込みを無効にしてNAPIをスケジュールする:

```c
// ドライバの割り込みハンドラ
static irqreturn_t my_nic_irq(int irq, void *data)
{
    napi_schedule(&nic->napi);   // softirqのpollをスケジュールする
    my_nic_mask_irq(nic);        // NIC割り込みを無効にする
    return IRQ_HANDLED;
}
```

`NET_RX_SOFTIRQ` softirqが（同じCPU上で）発火し、`net_rx_action()`を呼ぶ:

```c
// net/core/dev.c
static __latent_entropy void net_rx_action(void)
{
    // このCPU上でスケジュールされた各napiについて:
    while (!list_empty(&sd->poll_list)) {
        napi->poll(napi, budget);  // ドライバのpoll関数
    }
}
```

ドライバのpoll関数は、リングバッファのエントリから`sk_buff`を構築する:

```c
static int my_nic_poll(struct napi_struct *napi, int budget)
{
    while (pkts < budget && ring has packets) {
        skb = napi_build_skb(rx_buf, headroom);
        skb->protocol = eth_type_trans(skb, dev); // Ethernetタイプを設定する
        napi_gro_receive(napi, skb);              // GRO → スタックへ渡す
    }
    if (pkts < budget)
        napi_complete_done(napi, pkts);  // 割り込みを再び有効にする
    return pkts;
}
```

## フェーズ3: Ethernetの振り分け — __netif_receive_skb() {#phase-3-ethernet-demux-__netif_receive_skb}

`napi_gro_receive()`は、パケットを既存のGROフローに統合するか、`__netif_receive_skb()`へ渡す:

```c
// net/core/dev.c
static int __netif_receive_skb_core(struct sk_buff *skb, ...)
{
    // 1. パケットソケット（tcpdump、AF_PACKET）へ配送する
    list_for_each_entry_rcu(ptype, &ptype_all, list)
        ptype->func(skb, ...);

    // 2. XDP（このデバイスにアタッチされている場合）— 早期のドロップ/リダイレクト
    // [NAPI内でこの時点より前に処理される]

    // 3. skb->protocolに基づくプロトコルの振り分け
    // ptype_base[ntohs(skb->protocol) & PTYPE_HASH_MASK]を検索する
    // IPならip_rcv()
    // ARPならarp_rcv()
    // など
    deliver_ptype_list_skb(skb, &pt_prev, orig_dev, skb->protocol, ...);
}
```

## フェーズ4: IP入力 — ip_rcv() {#phase-4-ip-input-ip_rcv}

```c
// net/ipv4/ip_input.c:564
int ip_rcv(struct sk_buff *skb, struct net_device *dev, ...)
{
    // IPヘッダを検証する: チェックサム、バージョン、最小長
    iph = ip_hdr(skb);
    if (ip_fast_csum((u8 *)iph, iph->ihl))
        goto drop;  // 不正なチェックサム

    // Netfilter: NF_INET_PRE_ROUTINGフック（IPヘッダはまだそのまま）
    // （conntrackやDNATはここで行われる）
    return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, ..., ip_rcv_finish);
}

static int ip_rcv_finish(struct sk_buff *skb)
{
    // ルーティング: 自分宛てか、それとも転送すべきか?
    ip_route_input_noref(skb, iph->daddr, iph->saddr, iph->tos, dev);

    // dstが決める: ローカル配送か転送か
    return dst_input(skb);
    // → ローカルのパケットならip_local_deliver()
    // → 転送されるパケットならip_forward()
}
```

## フェーズ5: NetfilterのINPUTフックとローカル配送 {#phase-5-netfilter-input-hook-and-local-delivery}

```c
// net/ipv4/ip_input.c
int ip_local_deliver(struct sk_buff *skb)
{
    // Netfilter: NF_INET_LOCAL_INフック
    // （iptablesのINPUTルール、ファイアウォールなど）
    return NF_HOOK(NFPROTO_IPV4, NF_INET_LOCAL_IN, ..., ip_local_deliver_finish);
}

static int ip_local_deliver_finish(struct sk_buff *skb)
{
    // トランスポートプロトコルの振り分け
    // iph->protocol = IPPROTO_TCP (6)、IPPROTO_UDP (17)など
    ipprot = rcu_dereference(inet_protos[iph->protocol]);
    ipprot->handler(skb);
    // → TCPならtcp_v4_rcv()
    // → UDPならudp_rcv()
}
```

## フェーズ6: TCP受信 — tcp_v4_rcv() {#phase-6-tcp-receive-tcp_v4_rcv}

```c
// net/ipv4/tcp_ipv4.c:2147
int tcp_v4_rcv(struct sk_buff *skb)
{
    // 1. TCPヘッダを検証する
    th = tcp_hdr(skb);
    if (skb->len < th->doff * 4)
        goto drop;

    // 2. ソケット検索: (送信元IP, 送信元ポート, 宛先IP, 宛先ポート)に一致するsockを探す
    sk = __inet_lookup_skb(&tcp_hashinfo, skb, th->source, th->dest, ...);
    if (!sk)
        goto no_tcp_socket;  // RSTを送る

    // 3. ソケットのロックが保持されていれば、バックログにキューイングする
    if (!sock_owned_by_user(sk)) {
        tcp_v4_do_rcv(sk, skb);
    } else {
        sk_add_backlog(sk, skb, ...);
    }
}

static int tcp_v4_do_rcv(struct sock *sk, struct sk_buff *skb)
{
    // TCPのステートマシン: SYN、ACK、データ、FIN、RSTを扱う
    // ESTABLISHED状態の場合:
    tcp_rcv_established(sk, skb);
}
```

## フェーズ7: TCPのステートマシンとソケットキュー {#phase-7-tcp-state-machine-and-socket-queue}

データを受信しているESTABLISHED接続の場合:

```c
// net/ipv4/tcp_input.c（簡略化）
static void tcp_rcv_established(struct sock *sk, struct sk_buff *skb)
{
    // 高速パス: 期待どおりのシーケンス番号、オプションなし
    // 低速パス: 順序の入れ替わり、SACK、緊急データ、オプション

    // 1. シーケンス番号の検証
    // 2. 受信キューへのデータの配送
    eaten = tcp_queue_rcv(sk, skb, &fragstolen);

    // 3. ACKを送る（遅延される場合がある）
    tcp_event_data_recv(sk, skb);

    // 4. 読み手を起こす
    sk->sk_data_ready(sk);  // = sock_def_readable()
}

static int tcp_queue_rcv(struct sock *sk, struct sk_buff *skb, bool *fragstolen)
{
    // sk->sk_receive_queueに追加する
    __skb_queue_tail(&sk->sk_receive_queue, skb);
    sk_mem_charge(sk, skb->truesize);  // sk_rcvbufに対して課金する
}
```

## フェーズ8: アプリケーションのread() {#phase-8-application-read}

アプリケーションは`read()`または`recv()`を呼ぶ。`sk_receive_queue`が空であれば、（TASK_INTERRUPTIBLEで）ブロックする。`sk_data_ready()`が発火すると、タスクが起床する:

```c
// net/ipv4/tcp.c
int tcp_recvmsg(struct sock *sk, struct msghdr *msg, size_t len, int flags, int *addr_len)
{
    // ソケットをロックする
    lock_sock(sk);

    while (len > 0) {
        // sk_receive_queueから取り出す
        skb = skb_peek(&sk->sk_receive_queue);
        if (!skb) {
            // 何もない: 待つ
            sk_wait_data(sk, &timeo, last);
            continue;
        }

        // データをユーザー空間（msg->msg_iter）へコピーする
        copied = skb_copy_datagram_msg(skb, offset, msg, chunk);
        // オフセットを進め、消費したskbを解放する
    }

    // ロック保持中に蓄積されたバックログを処理する
    release_sock(sk);
}
```

`release_sock()`はバックログを処理する。ソケットがロックされている間に到着したパケットは、ここで`tcp_v4_do_rcv()`によって処理される。

## 注視すべき主要なカウンタ {#key-counters-to-watch}

```bash
# 各段階のドロップカウンタ
ethtool -S eth0 | grep -i drop   # NICリングでのドロップ
cat /proc/net/softnet_stat        # softirqバックログでのドロップ（列2）
ss -s                             # ソケットバッファでのドロップ
cat /proc/net/snmp | grep -E "Tcp|Ip"  # IP/TCPプロトコルのカウンタ

# 特定のTCPカウンタ
netstat -s | grep -E "segments|retransmit|error"

# ソケットの受信キューの深さ（RX-Q列）
ss -tn  # RX-Qが0でなければ、ソケットバッファに未読のデータがある
```

## 参考文献 {#further-reading}

- [sk_buff](sk-buff.md) — この経路全体で使われるパケット構造体
- [ネットワークデバイスとNAPI](napi.md) — フェーズ1〜2の詳細
- [ソケット層の概要](socket-layer.md) — struct sockと受信キュー
- [パケットの一生（送信）](life-of-packet-tx.md) — 送信方向の経路
- [TCPの実装](tcp.md) — TCPのステートマシンと輻輳制御
