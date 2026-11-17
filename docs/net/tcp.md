# TCPの実装 {#tcp-implementation}

> カーネルは信頼性があり順序の保証されたバイトストリーム配送をどう実装しているか

## TCPの保証 {#tcps-guarantees}

TCPは次のものを提供する:
1. **信頼性**: 失われたパケットは再送される
2. **順序保証**: 順序の入れ替わったセグメントは、配送前に並べ直される
3. **フロー制御**: 送信側は受信側を圧倒しない
4. **輻輳制御**: 送信側はネットワークを圧倒しない

カーネルはこの4つすべてを同時に実装しており、TCPが複雑なのはそのためである。

## struct tcp_sock {#struct-tcp_sock}

接続ごとのTCPの状態は`struct tcp_sock`（`struct sock`を拡張したもの）にある:

```c
// include/linux/tcp.h
struct tcp_sock {
    // シーケンス番号
    u32  rcv_nxt;      // 次に受信を期待するシーケンス番号
    u32  snd_nxt;      // 次に送信するシーケンス番号
    u32  snd_una;      // 最も古い未確認のバイト（送信ウィンドウの開始）

    // 輻輳制御
    u32  snd_cwnd;     // 輻輳ウィンドウ（MSS単位）
    u32  snd_ssthresh; // スロースタート閾値
    u32  snd_cwnd_cnt; // 線形増加のカウンタ

    // RTTの測定
    u32  srtt_us;      // 平滑化RTT（SRTT）× 8、マイクロ秒単位
    u32  mdev_us;      // RTTの平均偏差（RTO用）
    u32  rttvar_us;    // RTTの分散

    // 受信ウィンドウ
    u32  rcv_wnd;      // 現在の受信ウィンドウサイズ
    u32  rcv_ssthresh; // 受信ウィンドウサイズの閾値（小さなソケットバッファ）
    u32  window_clamp; // 最大ウィンドウサイズ

    // SACKの状態
    struct tcp_sack_block selective_acks[4];  // 通知するSACKブロック
    struct tcp_sack_block recv_sack_cache[4]; // 受信したSACKのキャッシュ

    // 再送
    u32  high_seq;     // 輻輳の発生時点のsnd_nxt
    u32  undo_marker;  // 新しい回復エピソード開始時のsnd_una

    // 輻輳制御のプライベートデータ
    // tcp_sk(sk)->inet_conn.icsk_ca_priv経由でアクセスされる
    // （アルゴリズム固有の状態用に64バイト。例: BBRの帯域幅サンプル）
    // ...
};
```

## シーケンス番号とウィンドウ {#sequence-numbers-and-windows}

```
Send side:
  snd_una ─── snd_nxt
     │            │
     └── unacked ─┘  (in-flight, waiting for ACK)
         ↑
  snd_una + snd_cwnd = send window limit

Receive side:
  rcv_nxt = next expected byte
  rcv_nxt + rcv_wnd = advertised window limit (flow control)
```

## 再送: RTOとタイマー {#retransmission-rto-and-timer}

**再送タイムアウト（RTO）**は、RTTの測定値から計算される:

```
SRTT = α × SRTT + (1 - α) × RTT_sample   (α = 7/8)
RTTVAR = β × RTTVAR + (1 - β) × |SRTT - RTT|   (β = 3/4)
RTO = SRTT + 4 × RTTVAR   (minimum: 200ms)
```

セグメントが送信されると、RTOの時間で再送タイマーが設定される。RTO以内にACKが届かなければ:
1. セグメントが再送される
2. RTOが2倍になる（指数バックオフ）
3. `snd_cwnd`が減らされる（輻輳のシグナル）

```bash
# TCPの再送カウンタを表示
netstat -s | grep retransmit
cat /proc/net/snmp | grep ^Tcp
# TcpRetransSegs: 再送したセグメントの総数
```

## 高速再送とSACK {#fast-retransmit-and-sack}

順序の入れ替わったセグメントが到着すると、受信側は**重複ACK**（以前と同じ`ack_seq`で、受信済みの範囲を示すSACKブロック付き）を送る。

重複ACKが3つ → （RTOが満了する前に）高速再送:

```c
// net/ipv4/tcp_input.c
static void tcp_fastretrans_alert(struct sock *sk, u32 prior_snd_una, ...)
{
    // 重複ACK 3つ → Fast Recovery（TCP_CA_Recovery状態）に入る
    if (tp->sacked_out >= tp->reordering) {
        tcp_enter_recovery(sk, ...);
        // 失われたセグメントを即座に再送する
        tcp_retransmit_skb(sk, tcp_rtx_queue_head(sk), 1);
    }
}
```

**SACK（Selective ACK）**を使うと、受信側は順序が入れ替わって受信したセグメントがどれかを送信側に正確に伝えられる。送信側は欠けている部分だけを再送できる:

```bash
# SACKはデフォルトで有効
cat /proc/sys/net/ipv4/tcp_sack  # → 1
```

## 輻輳制御のフレームワーク {#congestion-control-framework}

カーネルは、プラガブルな輻輳制御のフレームワークを備えている:

```c
// include/net/tcp.h
struct tcp_congestion_ops {
    // 古典的な応答: ACKごとにcwndを更新する
    void (*cong_avoid)(struct sock *sk, u32 ack, u32 acked);

    // 独自の制御: cwndとペーシングを完全に制御する
    void (*cong_control)(struct sock *sk, u32 ack, int flag,
                         const struct rate_sample *rs);

    // ロスが検出されたときに呼ばれる: 新しいssthreshを返す
    u32  (*ssthresh)(struct sock *sk);

    // ロス回復の後に呼ばれる: cwndを復元する
    u32  (*undo_cwnd)(struct sock *sk);

    char name[TCP_CA_NAME_MAX];  // "cubic"、"bbr"、"reno"など
};
```

### 利用可能なアルゴリズム {#available-algorithms}

```bash
# 利用可能な輻輳制御アルゴリズムを一覧表示
cat /proc/sys/net/ipv4/tcp_available_congestion_control
# cubic reno bbr2 bbr

# 現在のデフォルト
cat /proc/sys/net/ipv4/tcp_congestion_control
# cubic

# グローバルに変更
echo bbr > /proc/sys/net/ipv4/tcp_congestion_control

# ソケットごと（アルゴリズムによってはCAP_NET_ADMINが必要）
setsockopt(fd, IPPROTO_TCP, TCP_CONGESTION, "bbr", 4);
```

### CUBIC（デフォルト） {#cubic-default}

CUBICは、最後の輻輳イベントからの経過時間の3次関数を使って輻輳ウィンドウを増やす:

```
W(t) = C × (t - K)³ + W_max

Where:
  W_max = cwnd at last congestion event
  K = time for cwnd to return to W_max
  C = scaling constant (0.4)
```

- **スロースタート**: cwndはssthreshまで指数的に増加する
- **輻輳回避**: cwndは3次関数に従って増加する
- **ロスを検出**: ssthresh = 0.7 × cwnd、cwnd = ssthresh

### BBR（Bottleneck Bandwidth and RTT） {#bbr-bottleneck-bandwidth-and-rtt}

BBR（Bottleneck Bandwidth and RTT）は、ロスに反応するのではなく、ネットワーク経路（帯域幅と伝搬遅延）をモデル化する:

```
delivery_rate = bytes_delivered / elapsed_time
BtlBw = max delivery_rate over recent window
RTprop = min RTT over recent window

pacing_rate = 2/ln(2) × BtlBw ≈ 2.885 × BtlBw  (STARTUP phase)
              BtlBw                               (drain/PROBE_BW steady state)
cwnd = BtlBw × RTprop × 2   (BDP × 2, during STARTUP)
```

BBRはボトルネック帯域幅に合わせた**ペーシングされたレート**で送信し、ロスベースのアルゴリズムが引き起こすバッファブロートを回避する:

```bash
# BBRは正確なペーシングのためにfq qdiscを必要とする
echo fq > /sys/class/net/eth0/queues/tx-0/tx_queue_len  # これは正しくない
tc qdisc replace dev eth0 root fq
echo bbr > /proc/sys/net/ipv4/tcp_congestion_control
```

## フロー制御: 受信ウィンドウ {#flow-control-receive-window}

受信側の`rcv_wnd`は、受信バッファにどれだけの空きが残っているかを送信側に伝える。`rcv_wnd = 0`なら送信側は停止する（ゼロウィンドウプローブの状況）:

```c
// net/ipv4/tcp_output.c
static u16 tcp_select_window(struct sock *sk)
{
    u32 cur_win = tcp_receive_window(tp);
    u32 new_win = __tcp_select_window(sk);

    // ウィンドウを縮小しない（送信側を混乱させうる）
    if (new_win < cur_win)
        new_win = cur_win;

    return new_win;
}
```

**TCPウィンドウスケーリング**（RFC 1323）は、SYN/SYN-ACKでネゴシエートしたシフト係数を使うことで、65535バイトを超えるウィンドウを可能にする:

```bash
cat /proc/sys/net/ipv4/tcp_window_scaling  # → 1（有効）
```

## NagleアルゴリズムとTCP_NODELAY {#nagle-algorithm-and-tcp_nodelay}

デフォルトでは、TCPは小さなパケットをまとめることを期待して送信を遅らせる（Nagleのアルゴリズム）。レイテンシに敏感なアプリケーションでは:

```c
// Nagleを無効化: セグメントが小さくても即座に送信する
int flag = 1;
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(flag));
```

```bash
# TCP_NODELAYが設定されているか確認
ss -tni | grep nodelay
```

## 主要なTCPのsysctl {#key-tcp-sysctls}

```bash
# 初期輻輳ウィンドウはsysctlではなく、ルートごとの設定である:
# ip route change default via <gw> initcwnd 10
# デフォルトはRFC 6928に従い10 MSS

# 諦めるまでの最大再送回数
cat /proc/sys/net/ipv4/tcp_retries2  # デフォルト: 15（約924秒のタイムアウト）

# TIME_WAITソケット: 高速な再利用を許可する
echo 1 > /proc/sys/net/ipv4/tcp_tw_reuse  # 新しい送信方向の接続に再利用する

# キープアライブのプローブ
cat /proc/sys/net/ipv4/tcp_keepalive_time     # 7200秒（2時間）
cat /proc/sys/net/ipv4/tcp_keepalive_probes   # 9回のプローブ
cat /proc/sys/net/ipv4/tcp_keepalive_intvl    # プローブ間隔75秒

# バッファの自動チューニング [min, default, max]
cat /proc/sys/net/ipv4/tcp_rmem
cat /proc/sys/net/ipv4/tcp_wmem
```

## TCP接続の調査 {#inspecting-tcp-connections}

```bash
# TCPソケットの詳細情報（輻輳状態、cwndなどを表示）
ss -tni

# 出力例:
# State Recv-Q Send-Q  Local Address:Port  Peer Address:Port
# ESTAB 0      0       10.0.0.1:22         10.0.0.2:54321
#  cubic wscale:7,7 rto:208 rtt:4.5/0.75 ato:40 mss:1448 pmtu:1500
#  rcvmss:931 advmss:1448 cwnd:10 ssthresh:2147483647 bytes_sent:12345
#  segs_out:456 segs_in:234 data_segs_out:123 send 25.7Mbps
#  lastrcv:100ms lastack:50ms pacing_rate 51.4Mbps delivery_rate 25.7Mbps

# 接続ごとの再送情報
ss -tin | grep retrans

# TCPの統計
netstat -s | grep -E "connection|segment|retransmit"
```

## 参考文献 {#further-reading}

- [TCP輻輳制御アルゴリズム](tcp-congestion.md) — CUBICとBBRの詳細（下記参照）
- [パケットの一生（送信）](life-of-packet-tx.md) — tcp_write_xmitがcwndをどう使うか
- [connect()したときに何が起きるか](connect.md) — ハンドシェイクと初期ウィンドウの設定
- [ソケット層の概要](socket-layer.md) — tcp_sendmsgにデータを供給するソケットバッファ
