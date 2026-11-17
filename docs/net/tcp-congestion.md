# TCPの輻輳制御 {#tcp-congestion-control}

> カーネルが送信速度をどう決めるか: CUBIC、BBR、cc_opsインターフェース

## 輻輳制御が行うこと {#what-congestion-control-does}

TCPの輻輳制御は**輻輳ウィンドウ（cwnd）**、つまり送信中で未確認応答のバイト数の上限を管理する:

```
Throughput ≈ min(rwnd, cwnd) / RTT

rwnd: receiver's advertised window (flow control)
cwnd: sender's estimate of network capacity (congestion control)
```

輻輳制御器は送信側で動作し、次の事象に応答する:

- **ACKの受信**: cwndを増やす（ネットワークに余裕がある）
- **ロスの検出**: cwndを減らす（輻輳）
- **ECN（Explicit Congestion Notification）**: ロスが起きる前にcwndを減らす

## struct tcp_congestion_ops: 輻輳制御のインターフェース {#struct-tcp_congestion_ops-the-cc-interface}

すべての輻輳制御アルゴリズムはこのインターフェースを実装する:

```c
/* include/net/tcp.h */
struct tcp_congestion_ops {
    /* 必須 */
    u32  (*ssthresh)(struct sock *sk);      /* ロス後に ssthresh を設定する */
    void (*cong_avoid)(struct sock *sk,
                        u32 ack, u32 acked); /* ACK 受信時に cwnd を増やす */

    /* 任意 */
    void (*set_state)(struct sock *sk, u8 new_state);
    void (*cwnd_event)(struct sock *sk, enum tcp_ca_event ev);
    void (*in_ack_event)(struct sock *sk, u32 flags);
    u32  (*undo_cwnd)(struct sock *sk);
    void (*pkts_acked)(struct sock *sk,
                        const struct ack_sample *sample);
    u32  (*min_tso_segs)(struct sock *sk);
    void (*cong_control)(struct sock *sk, const struct rate_sample *rs);
    u32  (*get_info)(struct sock *sk, u32 ext,
                      int *attr, union tcp_cc_info *info);

    char         name[TCP_CA_NAME_MAX];
    struct module *owner;
    u32           key;
    u32           flags;
};
```

### アルゴリズムの登録 {#registering-an-algorithm}

```c
/* net/ipv4/tcp_cubic.c */
static struct tcp_congestion_ops cubictcp __read_mostly = {
    .init        = bictcp_init,
    .ssthresh    = bictcp_recalc_ssthresh,
    .cong_avoid  = bictcp_cong_avoid,
    .set_state   = bictcp_state,
    .undo_cwnd   = tcp_reno_undo_cwnd,
    .cwnd_event  = bictcp_cwnd_event,
    .pkts_acked  = bictcp_acked,
    .owner       = THIS_MODULE,
    .name        = "cubic",
};

static int __init cubictcp_register(void)
{
    BUILD_BUG_ON(sizeof(struct bictcp) > ICSK_CA_PRIV_SIZE);
    return tcp_register_congestion_control(&cubictcp);
}
```

### ソケットごとの輻輳制御データ {#per-socket-congestion-control-data}

各ソケットは、アルゴリズム固有の状態のために`ICSK_CA_PRIV_SIZE`バイト（128バイト = 16 × u64）を持つ:

```c
struct tcp_sock *tp = tcp_sk(sk);
struct bictcp *ca = inet_csk_ca(sk);  /* icsk_ca_priv を指す */
```

## TCP Reno: 基準となるアルゴリズム {#tcp-reno-the-baseline}

Renoは最も単純な正しいアルゴリズムであり、すべてのTCP実装に必須である:

```
Slow Start:
  cwnd starts at IW (≈10 segments)
  Each ACK: cwnd += 1 MSS  (exponential growth)
  Until cwnd >= ssthresh

Congestion Avoidance (AIMD):
  Each RTT: cwnd += 1 MSS  (linear growth, "AI" = Additive Increase)

Loss detected (triple duplicate ACKs):
  ssthresh = cwnd / 2
  cwnd = ssthresh  (fast recovery)
  ("MD" = Multiplicative Decrease)

Timeout (RTO):
  ssthresh = cwnd / 2
  cwnd = 1 MSS  (slow start restart)
```

```c
/* net/ipv4/tcp_cong.c - Reno の実装 */
void tcp_reno_cong_avoid(struct sock *sk, u32 ack, u32 acked)
{
    struct tcp_sock *tp = tcp_sk(sk);

    if (!tcp_is_cwnd_limited(sk))
        return;

    if (tcp_in_slow_start(tp)) {
        /* スロースタート: 確認応答されたセグメント数だけ増やす */
        acked = tcp_slow_start(tp, acked);
        if (!acked)
            return;
    }
    /* 輻輳回避: RTT ごとに cwnd += 1 MSS */
    tcp_cong_avoid_ai(tp, tp->snd_cwnd, acked);
}
```

## CUBIC: Linuxのデフォルト {#cubic-the-linux-default}

CUBICは、Renoの線形増加を、最後の輻輳イベントからの経過時間の**3次関数**に置き換える。これにより、帯域幅遅延積が大きい場合にウィンドウをはるかに速く拡大できる。

```
CUBIC window function:
  W(t) = C × (t - K)³ + Wmax

  t    = time since last congestion event
  K    = time to reach Wmax (origin point)
  Wmax = window at last congestion
  C    = scaling factor (typically 0.4)

    cwnd
      │        ╭─── new Wmax
  Wmax│───╮   ╭
      │    ╰─╯  concave/convex cubic curve
      │
      └────────────── time
           K
```

3次関数の形状により、CUBICは次のように振る舞う:

- Wmaxより十分下では速く増加する（ロス後の追い上げ）
- Wmaxの近くでは増加が緩やかになる（慎重に探る）
- Wmaxが受信側によって制限されていた場合は、それを超えて増加する

```c
/* net/ipv4/tcp_cubic.c */
struct bictcp {
    u32 cnt;           /* この数の ACK の後に cwnd を 1 増やす */
    u32 last_max_cwnd; /* Wmax: 直近の最大 snd_cwnd */
    u32 last_cwnd;
    u32 last_time;
    u32 bic_origin_point; /* bic 関数の原点 */
    u32 bic_K;            /* 原点までの時間 */
    u32 delay_min;        /* 最小遅延（msec << 3） */
    u32 epoch_start;      /* エポックの開始 */
    u32 ack_cnt;          /* ACK の数 */
    u32 tcp_cwnd;         /* 推定 TCP cwnd */
    u16 unused;
    u8  sample_cnt;       /* curr_rtt を決めるためのサンプル数 */
    u8  found;            /* 終了点が見つかったか */
    u32 round_start;      /* 各ラウンドの開始 */
    u32 end_seq;          /* ラウンドの end_seq */
    u32 last_ack;         /* ACK 間隔が詰まっていた直近の時刻 */
    u32 curr_rtt;         /* 現在のラウンドの最小 RTT */
};

static void bictcp_cong_avoid(struct sock *sk, u32 ack, u32 acked)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct bictcp *ca = inet_csk_ca(sk);

    if (!tcp_is_cwnd_limited(sk))
        return;

    if (tcp_in_slow_start(tp)) {
        if (hystart && after(ack, ca->end_seq))
            bictcp_hystart_reset(sk);  /* HyStart: 遅延が増えたら
                                          スロースタートを抜ける */
        acked = tcp_slow_start(tp, acked);
        if (!acked)
            return;
    }
    bictcp_update(ca, tp->snd_cwnd, acked);
    tcp_cong_avoid_ai(tp, ca->cnt, acked);
}
```

### HyStart++: 安全なスロースタートの終了 {#hystart-safe-slow-start-exit}

CUBICはHyStartを使い、実際にロスが起きる前にスロースタートを抜ける:

- **遅延の増加**: RTTがしきい値を超えて増加したら抜ける（ACKトレインの遅延）
- **ACKトレインの終端**: ACKの間隔からパイプが満杯だとわかったら抜ける

```bash
# HyStart の確認/設定
sysctl net.ipv4.tcp_cubic_hystart
```

## BBR: 帯域幅ベースの輻輳制御 {#bbr-bandwidth-based-congestion-control}

BBR（Bottleneck Bandwidth and Round-trip propagation time）は根本的に異なるアプローチをとる。**ロス**に反応する代わりに、**帯域幅**と**RTT**を推定して送信レートを直接設定する。

```
BBR's model:
  BtlBw  = bottleneck bandwidth (bytes/sec)
  RTprop = round-trip propagation delay (min RTT)

  Optimal operating point:
    Inflight = BtlBw × RTprop
    Rate = BtlBw

BBR phases:
  STARTUP:       probe bandwidth (double rate each RTT, like slow start)
  DRAIN:         drain the queue STARTUP filled
  PROBE_BW:      cycle pacing_gain (1.25/0.75/1.0...) to probe bandwidth
  PROBE_RTT:     reduce cwnd to 4 MSS to measure min RTT (every 10s)
```

```c
/* net/ipv4/tcp_bbr.c */
struct bbr {
    u32  min_rtt_us;           /* 最小 RTT（usec） */
    u32  min_rtt_stamp;        /* min_rtt_us のタイムスタンプ */
    u32  probe_rtt_done_stamp; /* BBR_PROBE_RTT モードの終了時刻 */
    struct minmax bw;          /* 直近の最大配送レート */
    u32  rtt_cnt;              /* 経過したパケット計時ラウンドの数 */
    u32  next_rtt_delivered;   /* ラウンド終了時の scb->tx.delivered */
    u64  cycle_mstamp;         /* このサイクルフェーズの開始時刻 */
    u32  mode:3,               /* 現在の BBR モード */
         prev_ca_state:3,      /* 前回の ACK 時の CA 状態 */
         packet_conservation:1,/* パケット保存を使うか */
         round_start:1,        /* パケット計時の tx.delivered ラウンドの開始か */
         idle_restart:1,       /* アイドル後の再開か */
         probe_rtt_round_done:1, /* 4 パケットでの BBR_PROBE_RTT ラウンドか */
         unused:13,
         lt_is_sampling:1,     /* 現在、長期（"LT"）サンプルを取っているか */
         lt_rtt_cnt:7,         /* 長期区間内のラウンドトリップ数 */
         lt_use_bw:1;          /* lt_bw を帯域幅推定値として使うか */
    u32  lt_bw;                /* LT の推定配送レート（pkts/uS << 24） */
    u32  lt_last_delivered;    /* LT 区間の開始: tp->delivered */
    u32  lt_last_stamp;        /* LT 区間の開始: tp->delivered_mstamp */
    u32  lt_last_lost;         /* LT 区間の開始: tp->lost */
    u32  pacing_gain:10,       /* 送信レートの現在のゲイン */
         cwnd_gain:10,         /* cwnd の現在のゲイン */
         full_bw_reached:1,    /* BBR_FULL_BW_THRESH に達したか */
         full_bw_cnt:2,        /* しきい値に達したラウンド数 */
         cycle_idx:3,          /* pacing_gain サイクル配列の現在のインデックス */
         has_seen_rtt:1,       /* RTT サンプルをすでに見たか */
         unused_b:5;
    u32  prior_cwnd;           /* ロス回復に入る前の cwnd */
    u32  full_bw;              /* パイプが満杯かを推定するための直近の帯域幅 */
};
```

### BBRのペーシング {#bbr-pacing}

BBRはバースト送信ではなく**ペーシング**（パケットを時間的に均等に分散させること）を使う:

```c
/* ペーシングレートを設定: BtlBw × pacing_gain */
static void bbr_set_pacing_rate(struct sock *sk, u32 bw, int gain)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct bbr *bbr = inet_csk_ca(sk);
    u64 rate = bw;

    rate = bbr_rate_bytes_per_sec(sk, rate, gain);
    rate = min_t(u64, rate, sk->sk_max_pacing_rate);
    if (bbr->mode != BBR_STARTUP || rate > sk->sk_pacing_rate)
        WRITE_ONCE(sk->sk_pacing_rate, rate);
}
```

TCPのペーシングは、パケットスケジューラ（FQ/fq_codelのqdisc）によって、あるいはペーシング用のqdiscがなければhrtimerによって実装される。

```bash
# BBR のために FQ ペーシングを有効化する（推奨）:
tc qdisc add dev eth0 root fq
sysctl -w net.ipv4.tcp_congestion_control=bbr
```

## BBRとCUBICの比較 {#bbr-vs-cubic}

| 観点 | CUBIC | BBR |
|--------|-------|-----|
| 反応する対象 | ロス | 帯域幅 + RTT |
| ペーシング | なし（バースト） | あり（BtlBwのレート） |
| キューの充填 | ボトルネックのキューを満たす | BtlBw×RTpropを目標とする |
| 公平性 | 他のCUBICに対して公平 | ロスベースのものに対して不公平になりうる |
| ロス時の性能 | 良好 | 高ロス環境ではより良い |
| レイテンシ | 高い（キューのため） | 低い（最小RTTを目標とする） |
| カーネルバージョン | 2.6.19でデフォルト | 4.9 |

## 輻輳制御の選択と観察 {#selecting-and-observing-congestion-control}

```bash
# システムのデフォルト
sysctl net.ipv4.tcp_congestion_control

# システムのデフォルトを変更
sysctl -w net.ipv4.tcp_congestion_control=bbr

# ソケットごと（非標準のアルゴリズムには CAP_NET_ADMIN が必要）:
setsockopt(fd, IPPROTO_TCP, TCP_CONGESTION, "bbr", 4);

# 利用可能なアルゴリズムを一覧表示
sysctl net.ipv4.tcp_available_congestion_control
# または:
cat /proc/sys/net/ipv4/tcp_available_congestion_control

# コネクションごとの状態を観察する（ss -i は cwnd、rtt などを表示）
ss -tni
# 出力:
#  cubic wscale:7,7 rto:204 rtt:0.748/0.374 ato:40
#  mss:1448 pmtu:1500 rcvmss:1448 advmss:1448 cwnd:10 ssthresh:7
#  bytes_sent:1234 bytes_acked:1234 segs_out:10 segs_in:10

# BBR 固有の情報
ss -tni | grep -A2 bbr
#  bbr wscale:7,7 rto:204 rtt:1.234/0.617
#  bw:1.23Gbps pacing_rate:1.50Gbps
```

## ECN: 明示的輻輳通知 {#ecn-explicit-congestion-notification}

ECNにより、ルーターはパケットを破棄する前に、IPヘッダのビットで輻輳を通知できる:

```
IP header: ECT(0), ECT(1) — ECN capable transport
TCP: CWR (congestion window reduced), ECE (ECN echo)

Flow:
  1. Sender marks packets as ECN-capable (ECT)
  2. Router sets CE (Congestion Experienced) when queue fills
  3. Receiver echoes CE back to sender via ECE flag
  4. Sender reduces cwnd (like a loss signal) and sets CWR
```

```bash
# ECN を有効化
sysctl -w net.ipv4.tcp_ecn=1  # 両端がサポートしていれば有効化
# 0: 無効, 1: ECN を有効化（SYN で開始する）, 2: ECN を受け入れるが自分からは開始しない
```

## 参考文献 {#further-reading}

- [TCPの実装](tcp.md) — TCPソケットの状態機械、高速再送
- [ネットワークバッファのチューニング](net-buffer-tuning.md) — cwndの初期値のチューニング
- [TCとqdisc](tc-qdisc.md) — BBRのペーシングのためのFQ qdisc
- `net/ipv4/tcp_cubic.c` — CUBICの実装
- `net/ipv4/tcp_bbr.c` — BBRの実装
- `net/ipv4/tcp_cong.c` — cc_opsの登録とディスパッチ
