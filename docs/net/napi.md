# ネットワークデバイスとNAPI {#network-device-and-napi}

> カーネルはハードウェアからパケットをどう受け取るか: 割り込みの緩和とポーリング

## NAPIが解決する問題 {#the-problem-napi-solves}

NAPI（New API）以前は、到着したパケットごとにハードウェア割り込みが発生していた。トラフィックのレートが低いうちは、これでうまく動く。しかし高負荷時、例えば64バイトのパケットで10 Gbpsとなると、NICは毎秒数十万回の割り込みを発生させる。カーネルはパケットの処理よりも割り込みの処理に多くの時間を費やすことになる。これが**割り込みライブロック**である。

NAPIは、トラフィックが多いときにポーリングモデルへ切り替えることでこれを解決する:

1. パケットが到着する → NICが割り込みを発生させる
2. カーネルがそれ以降のNIC割り込みを無効にする
3. カーネルがNICのリングバッファをポーリングして、さらなるパケットを取り出す
4. バジェット分のパケットを処理したら、割り込みを再び有効にする

これにより処理がバッチ化され、高いパケットレートでの割り込みのオーバーヘッドが劇的に減る。

## struct napi_struct {#struct-napi_struct}

各ネットワークデバイス（またはマルチキューNICの各キュー）は、1つの`napi_struct`を持つ:

```c
// include/linux/netdevice.h
struct napi_struct {
    struct list_head    poll_list;    // このCPUのsoftirqポーリングリスト上
    unsigned long       state;        // NAPI_STATE_SCHED、NAPI_STATE_DISABLE、...
    int                 weight;       // poll()呼び出し1回あたりの最大パケット数（バジェット）
    int                 (*poll)(struct napi_struct *, int);  // ドライバのpoll関数

    struct net_device   *dev;
    struct hrtimer      timer;        // スレッド化NAPIや遅延処理用

    u32                 napi_id;      // ビジーポーリング用の一意なID
    int                 index;        // デバイス上のキューのインデックス

    // GRO（Generic Receive Offload）の状態
    struct gro_node     gro;
};
```

## NAPIの受信フロー {#the-napi-receive-flow}

```
1. Packet arrives in NIC ring buffer
2. NIC raises interrupt
        ↓
3. Hard IRQ handler (driver's irq_handler):
   - napi_schedule(&napi)    → sets NAPI_STATE_SCHED
   - disable NIC interrupt    (no more hardware interrupts)
   - return IRQ_HANDLED
        ↓
4. Soft IRQ (NET_RX_SOFTIRQ) runs on the same CPU:
   - net_rx_action() polls all scheduled NAPI instances
        ↓
5. Driver's poll() function:
   for each packet (up to budget):
       skb = napi_build_skb(rx_buf, frag_size)
       napi_gro_receive(&dev->napi, skb)  → GRO + route up the stack
   if (processed < budget):
       napi_complete_done(&napi, processed)  → re-enable interrupt
   else:
       return budget  → more work, stay scheduled
```

## ドライバの実装パターン {#driver-implementation-pattern}

```c
// 割り込みハンドラ
static irqreturn_t my_nic_interrupt(int irq, void *dev_id)
{
    struct my_nic *nic = dev_id;
    // このNICキューの割り込みを無効にする
    my_nic_disable_irq(nic);
    // NAPIのpollをスケジュールする
    napi_schedule(&nic->napi);
    return IRQ_HANDLED;
}

// NAPIのpoll関数
static int my_nic_poll(struct napi_struct *napi, int budget)
{
    struct my_nic *nic = container_of(napi, struct my_nic, napi);
    int processed = 0;

    while (processed < budget) {
        struct sk_buff *skb = my_nic_receive_one(nic);
        if (!skb)
            break;
        napi_gro_receive(napi, skb);  // GRO + ネットワークスタックへ渡す
        processed++;
    }

    if (processed < budget) {
        // パケットがもうない: ポーリングモードを抜け、割り込みを再び有効にする
        napi_complete_done(napi, processed);
        my_nic_enable_irq(nic);
    }

    return processed;
}
```

## NAPIのバジェット {#napi-budget}

バジェット（デフォルト: poll呼び出し1回あたり64パケット）は公平性のための仕組みであり、高負荷の1つのNICがsoftirqのCPUを独占できないようにする。64パケットを処理すると、softirqハンドラは譲って他の処理を実行させる。

```bash
# NAPIのバジェットを調整（デバイスごと）
ethtool -G eth0 rx 4096  # リングバッファサイズを増やす

# グローバルなバジェットは/proc/sys/net/core/netdev_budgetにある
cat /proc/sys/net/core/netdev_budget  # デフォルト: 300
echo 600 > /proc/sys/net/core/netdev_budget  # 高スループットなワークロード向け

# 1回のポーリングサイクルの時間制限
cat /proc/sys/net/core/netdev_budget_usecs  # デフォルト: 2000（2ms）
```

## GRO: Generic Receive Offload {#gro-generic-receive-offload}

GROは、スタックの上位へ渡す前に複数の小さなTCPセグメントを1つの大きな`sk_buff`に統合し、パケットごとのオーバーヘッドを削減する:

```c
// ドライバはnetif_receive_skb()の代わりにnapi_gro_receive()を呼ぶ
// GROは同じTCPフローのセグメントを蓄積する
// 完了すると、統合された大きなskbをスタックの上位へ渡す
napi_gro_receive(napi, skb);
```

```bash
# GROが有効か確認
ethtool -k eth0 | grep generic-receive-offload
# generic-receive-offload: on

# GROを無効化（デバッグ時やGROがレイテンシを増やす場合）
ethtool -K eth0 gro off
```

## struct net_device {#struct-net_device}

```c
// include/linux/netdevice.h（主なフィールド）
struct net_device {
    char            name[IFNAMSIZ];  // "eth0"、"lo"など
    unsigned long   state;           // __LINK_STATE_STARTなど
    struct netdev_queue *_tx;        // 送信キュー（配列）
    struct Qdisc    *qdisc;          // トラフィック制御のqdisc

    const struct net_device_ops *netdev_ops;  // ドライバの操作

    unsigned int    flags;           // IFF_UP、IFF_BROADCAST、IFF_PROMISC、...
    unsigned int    mtu;             // 最大転送単位

    // 統計
    struct net_device_stats stats;   // rx/txのパケット数、バイト数、エラー
    // ...
};
```

## スレッド化NAPI {#threaded-napi}

予測可能なレイテンシが必要なワークロードでは、NAPIをsoftirqではなくカーネルスレッドで実行できる:

```bash
# すべてのNICでスレッド化NAPIを有効化
echo 1 > /sys/class/net/eth0/threaded

# カーネルは"napi/eth0-0"などのkthreadを作成する
# キャッシュ局所性を高めるため、これらをCPUにピン留めできる
taskset -cp 4 $(pgrep "napi/eth0-0")
```

スレッド化NAPIでは、poll関数がスリープでき（例えばGROのフラッシュタイマーのため）、ポーリングを行うCPUの配置をスケジューラが完全に制御できる。

## ビジーポーリング {#busy-polling}

超低レイテンシのアプリケーションでは、ビジーポーリングがsoftirqをバイパスし、ユーザーコンテキストから直接NICをポーリングする:

```bash
# ビジーポーリングを有効化
echo 50 > /proc/sys/net/core/busy_poll    # 50µsポーリングする
echo 50 > /proc/sys/net/core/busy_read    # recv()でも行う

# ソケットごと（SO_BUSY_POLLオプション）
setsockopt(fd, SOL_SOCKET, SO_BUSY_POLL, &timeout_us, sizeof(timeout_us));
```

これはCPUサイクルとレイテンシのトレードオフである。アプリケーションは、スリープしてsoftirqに起こされる代わりに、スピンしながらNICをポーリングする。

## RPS/RFS: 複数CPUでの受信スケーリング {#rpsrfs-multi-cpu-receive-scaling}

シングルキューのNICは、すべてのパケットを1つのCPUに届ける。**RPS（Receive Packet Steering）**は、ソフトウェアでパケットを複数のCPUに分散させる:

```bash
# 受信処理をCPU 0〜7に分散
echo ff > /sys/class/net/eth0/queues/rx-0/rps_cpus

# RFS（Receive Flow Steering）: アプリケーションが動作しているCPUへ
# パケットを振り向ける（キャッシュ局所性が向上する）
echo 32768 > /proc/sys/net/core/rps_sock_flow_entries
echo 2048  > /sys/class/net/eth0/queues/rx-0/rps_flow_cnt
```

RSS（Receive Side Scaling）を備えた現代的なマルチキューNICは、これをハードウェアで行い、フローをハードウェアキューに分散させる。キューごとに1つのNAPIインスタンスがある。

## 受信性能の診断 {#diagnosing-receive-performance}

```bash
# RXのドロップカウンタ
ethtool -S eth0 | grep -i drop

# softirqの統計（NET_RX_SOFTIRQのオーバーラン）
cat /proc/net/softnet_stat
# 列: total、dropped、time_squeeze、[cpu_collision]、[received_rps]、[flow_limit_count]
# time_squeeze: pollがバジェットを使い切った（netdev_budgetの増加を検討する）
# dropped: バックログのオーバーフローでパケットがドロップされた

# CPUごとの割り込み
cat /proc/interrupts | grep eth0

# GROの統合の統計を確認
ethtool -S eth0 | grep gro
```

## 参考文献 {#further-reading}

- [sk_buff](sk-buff.md) — 受信時に構築されるパケット構造体
- [パケットの一生（受信）](life-of-packet-rx.md) — NAPIより先の受信パス全体
- [ソケット層の概要](socket-layer.md) — ネットワーク層の処理の後、パケットがどこへ行くか
- [XDP](xdp.md) — さらに早い段階、sk_buffの割り当て前でのパケット処理
