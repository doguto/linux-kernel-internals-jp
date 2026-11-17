# connect()を呼ぶと何が起きるか {#what-happens-when-you-connect}

> カーネルの視点から見たTCPの3ウェイハンドシェイク

## 概要 {#overview}

クライアントが`connect()`を呼ぶと、カーネルはTCPの3ウェイハンドシェイクを開始する。この呼び出しは、コネクションが確立するまで（あるいはタイムアウトかエラーになるまで）ブロックすることがある。全体の流れは次のとおり:

```
Client                                    Server
  │                                          │
connect() →                                  │ (listening on accept())
  tcp_v4_connect()                           │
  → route lookup                             │
  → ephemeral port selection                 │
  → TCP state: SYN_SENT                      │
  → send SYN packet                          │
  │────────────── SYN ─────────────────────→ │
  │                                     tcp_v4_rcv()
  │                                     tcp_rcv_state_process()
  │                                     TCP state: SYN_RECEIVED
  │                                     send SYN-ACK
  │ ←──────────── SYN-ACK ─────────────── │
tcp_v4_rcv()                                 │
tcp_rcv_synsent_state_process()              │
→ TCP state: ESTABLISHED                     │
→ wake up connect()                          │
→ send ACK                                   │
  │────────────── ACK ─────────────────────→ │
  │                                     TCP state: ESTABLISHED
  │                                     sk queued on listen backlog
  │                                     accept() returns new socket
```

## フェーズ1: connect()システムコール → tcp_v4_connect() {#phase-1-connect-syscall-tcp_v4_connect}

```c
// ユーザー空間
int fd = socket(AF_INET, SOCK_STREAM, 0);
connect(fd, (struct sockaddr *)&addr, sizeof(addr));
```

システムコールのパス:

```
connect()
  → inet_stream_connect()       [af_inet.c]
    → __inet_stream_connect()
      → sk->sk_prot->connect()  = tcp_v4_connect()
```

## フェーズ2: tcp_v4_connect() {#phase-2-tcp_v4_connect}

```c
// net/ipv4/tcp_ipv4.c:222
int tcp_v4_connect(struct sock *sk, struct sockaddr *uaddr, int addr_len)
{
    struct sockaddr_in *usin = (struct sockaddr_in *)uaddr;

    // 1. 経路検索: 宛先への到達方法を見つける
    rt = ip_route_connect(fl4, daddr, inet->inet_saddr, ...);

    // 2. 送信元IPの選択（bindされていない場合）
    // fl4->saddr はルーティングサブシステムが埋める

    // 3. エフェメラルポートの選択（bindされていない場合）
    err = inet_hash_connect(&tcp_death_row, sk);
    // ip_local_port_range から送信元ポートを選び、衝突がないか確認する

    // 4. SYN_SENT 状態へ遷移
    tcp_set_state(sk, TCP_SYN_SENT);

    // 5. connect タイマー（再送）を開始
    inet->inet_id = get_random_u16();

    // 6. SYN を送信
    err = tcp_connect(sk);
}
```

## フェーズ3: tcp_connect() — SYNの送信 {#phase-3-tcp_connect-sending-the-syn}

```c
// net/ipv4/tcp_output.c:4296
int tcp_connect(struct sock *sk)
{
    struct tcp_sock *tp = tcp_sk(sk);

    // シーケンス番号を初期化（セキュリティのためランダム）
    tp->write_seq = secure_tcp_seq(inet->inet_saddr, inet->inet_daddr,
                                   inet->inet_sport, inet->inet_dport);

    // SYN 用の sk_buff を割り当てる
    buff = sk_stream_alloc_skb(sk, 0, sk->sk_allocation, true);

    // SYN パケットを組み立てる
    tcp_init_nondata_skb(buff, tp->write_seq++, TCPHDR_SYN);

    // TCP Fast Open: 要求されていれば SYN にデータを付ける
    if (tp->fastopen_req)
        err = tcp_send_syn_data(sk, buff);
    else
        tcp_transmit_skb(sk, buff, 1, sk->sk_allocation);

    // 再送タイマーを開始
    inet_csk_reset_xmit_timer(sk, ICSK_TIME_RETRANS, inet_csk(sk)->icsk_rto, ...);

    return 0;
}
```

SYNパケットに設定されるSYNオプション:

- **MSS**（Maximum Segment Size）: こちらが受け入れる最大セグメントサイズをサーバーに伝える
- **WSCALE**: 大きなウィンドウのためのウィンドウスケーリング係数
- **SACK_PERM**: Selective ACKをサポートしていることを示す
- **Timestamp**: RTT測定とPAWS（Protection Against Wrapped Sequences）のため

## フェーズ4: サーバーがSYNを処理する → tcp_rcv_state_process() {#phase-4-server-processes-syn-tcp_rcv_state_process}

サーバー側では:

```c
// net/ipv4/tcp_input.c
int tcp_rcv_state_process(struct sock *sk, struct sk_buff *skb)
{
    // LISTEN 状態のサーバーが SYN を受信:
    case TCP_LISTEN:
        if (th->syn) {
            // リクエストソケットを作成（軽量で、まだ完全な sock ではない）
            req = inet_reqsk_alloc(rsk_ops, sk, !want_cookie);

            // クライアントの ISN、MSS、オプションを記録
            tcp_parse_options(net, skb, &opt_rx, ...);

            // SYN タイムアウトタイマーを開始（ハーフオープンのコネクションを扱う）
            inet_csk_reqsk_queue_hash_add(sk, req, TCP_TIMEOUT_INIT);

            // SYN-ACK を送信
            af_ops->send_synack(sk, dst, &fl, req, ...);
            // → tcp_send_synack() → tcp_transmit_skb()
        }
}
```

サーバーは**リクエストソケット**（`struct request_sock`）を作成する。これはハーフオープンのコネクションを表す軽量なエントリである。完全な`struct sock`は、3番目のACKが届いたときに初めて割り当てられる。

## フェーズ5: クライアントがSYN-ACKを処理する → ESTABLISHED {#phase-5-client-processes-syn-ack-established}

```c
// net/ipv4/tcp_input.c
static int tcp_rcv_synsent_state_process(struct sock *sk, struct sk_buff *skb, ...)
{
    // SYN_SENT 状態のクライアントが SYN-ACK を受信:
    if (th->ack) {
        // ACK 番号を検証: 自分の SYN に対する ACK でなければならない
        if (TCP_SKB_CB(skb)->ack_seq != tp->snd_nxt)
            goto reset_and_undo;

        // サーバーの ISN を受け入れる
        tp->rcv_nxt = TCP_SKB_CB(skb)->seq + 1;

        // サーバーの MSS、ウィンドウスケール、SACK サポートを記録
        tcp_process_options(...);

        // ESTABLISHED へ遷移
        tcp_set_state(sk, TCP_ESTABLISHED);

        // 輻輳制御を初期化
        tcp_init_congestion_control(sk);

        // ACK を送信（3ウェイハンドシェイクが完了する）
        tcp_send_ack(sk);

        // connect() を起床させる — これで return できる
        sk->sk_state_change(sk);
        // → wake_up_interruptible(sk->sk_wq)
    }
}
```

`connect()`のブロックが解除され、0を返す。

## フェーズ6: サーバーがACKを処理する → accept()が返る {#phase-6-server-processes-ack-accept-returns}

```c
// サーバーが最後の ACK を受信:
// tcp_rcv_state_process() → TCP_SYN_RECV のケース
//   → inet_csk(sk)->icsk_af_ops->syn_recv_sock()
//       → tcp_v4_syn_recv_sock()
//           → 新しいコネクション用に完全な struct sock を割り当てる
//           → request_sock キューから accept キューへ移す
//   → sk_acceptq_added()  (accept キューの長さを増やす)

// サーバー側の accept():
accept(listenfd, &addr, &addrlen)
  → inet_accept()
    → sk->sk_prot->accept()  = inet_csk_accept()
      → accept キューにコネクションが来るのを待つ
      → デキューして新しい sock を返す
```

## SYNバックログとacceptキュー {#syn-backlog-and-accept-queue}

サーバーは2つのキューを保持する:

```
SYN queue (incomplete connections):
  SYN received → request_sock added
  SYN-ACK sent
  ACK received → moved to accept queue

Accept queue (complete connections):
  accept() dequeues from here
```

これらは次の設定で制御される:

```bash
# accept キューの最大長（SOMAXCONN）
cat /proc/sys/net/core/somaxconn  # default: 4096

# TCP の SYN バックログ
cat /proc/sys/net/ipv4/tcp_max_syn_backlog  # default: 512

# listen() の第2引数がソケットごとの accept キューの上限を設定する
listen(fd, backlog);  # capped at somaxconn
```

acceptキューが満杯 → 新しいACKが破棄される → クライアントが再試行する（エラーではなく、コネクションの遅さとして現れる）。SYNキューが満杯 → SYN cookieか、SYNの破棄となる。

## SYN cookie {#syn-cookies}

SYNフラッド攻撃を受けると、SYNキューが満杯になる。SYN cookieはコネクションの状態をISNにエンコードすることで、SYNキューのエントリを使わずに済ませる:

```bash
# SYN cookie を有効化（SYN フラッド時に自動的に作動する）
cat /proc/sys/net/ipv4/tcp_syncookies  # 1 = enabled (default)
```

SYN cookieを使う場合、サーバーはSYNを受信してもrequest_sockを作成しない。代わりに、コネクションのパラメータをSYN-ACKのシーケンス番号にエンコードする。有効なcookieを含むACKが届くと、SYNキューに一度も入ることなくコネクションが確立される。

## エフェメラルポートの選択 {#ephemeral-port-selection}

```bash
# エフェメラル（送信側）ポートの範囲
cat /proc/sys/net/ipv4/ip_local_port_range
# → 32768 60999  (default)

# 使用中のポート数を数える
ss -tn | wc -l

# ポートが枯渇しそうなら: 範囲を広げるか SO_REUSEPORT を使う
echo "1024 65535" > /proc/sys/net/ipv4/ip_local_port_range
```

## コネクションの観察 {#observing-the-connection}

```bash
# すべての TCP 状態のコネクションを見る
ss -tn state syn-sent
ss -tn state syn-recv  # サーバー側: ハーフオープンのコネクション

# 状態ごとにコネクション数を数える
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn

# ハンドシェイクをトレースする
trace-cmd record -e tcp:tcp_rcv_state_process \
                 -e tcp:tcp_send_syn \
                 -e tcp:tcp_send_synack \
                 sleep 5
```

## 参考文献 {#further-reading}

- [パケットの一生（送信）](life-of-packet-tx.md) — SYNパケットがどのように送信されるか
- [パケットの一生（受信）](life-of-packet-rx.md) — SYN-ACKがどのように受信されるか
- [TCPの実装](tcp.md) — 輻輳制御、フロー制御、再送
- [ソケット層の概要](socket-layer.md) — ソケットの状態機械
