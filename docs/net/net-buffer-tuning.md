# ネットワークバッファのチューニング {#network-buffer-tuning}

> 高スループットなネットワーキングのための、ソケットバッファ、リングバッファ、バックプレッシャーのチューニング

## バッファの階層 {#the-buffer-hierarchy}

ネットワークのデータはいくつものバッファリング層を流れ、そのどれもがボトルネックになりうる:

```
Application
    ↓ write()
Socket send buffer  (sk->sk_sndbuf)
    ↓ tcp_sendmsg
TCP write queue     (segments not yet sent or ACK'd)
    ↓ ip_queue_xmit
qdisc queue         (dev_queue_xmit)
    ↓ ndo_start_xmit
NIC TX ring buffer  (driver)
    ↓ DMA
Wire

Wire
    ↓ DMA
NIC RX ring buffer  (driver)
    ↓ NAPI
Softnet backlog     (sd->input_pkt_queue)
    ↓ __netif_receive_skb
Socket receive buffer (sk->sk_receive_queue)
    ↓ tcp_recvmsg
Application
```

## ソケットバッファのチューニング {#socket-buffer-tuning}

### TCPの自動チューニング {#tcp-auto-tuning}

デフォルトでは、TCPは観測された帯域幅遅延積に基づいてソケットバッファを自動チューニングする:

```bash
# TCPバッファの自動チューニング（推奨: 有効のままにする）
cat /proc/sys/net/ipv4/tcp_moderate_rcvbuf  # デフォルト: 1（有効）

# 自動チューニングの範囲 [min, default, max]（バイト単位）
cat /proc/sys/net/ipv4/tcp_rmem  # → 4096 131072 6291456
cat /proc/sys/net/ipv4/tcp_wmem  # → 4096 16384 4194304

# 高帯域・高レイテンシの経路の場合（例: 10Gbps、RTT 50ms）
# BDP = 10Gbps × 50ms = 62.5 MB → 64MB以上のバッファが必要
echo "4096 87380 67108864" > /proc/sys/net/ipv4/tcp_rmem
echo "4096 65536 67108864" > /proc/sys/net/ipv4/tcp_wmem
```

### 手動でのバッファサイズ設定 {#manual-buffer-sizing}

```bash
# グローバルな最大値（setsockoptはこれを超えられない）
cat /proc/sys/net/core/rmem_max  # デフォルト: 212992（208KB）
cat /proc/sys/net/core/wmem_max

# 大きなバッファを使うソケットのために増やす
echo 67108864 > /proc/sys/net/core/rmem_max  # 64MB
echo 67108864 > /proc/sys/net/core/wmem_max

# ソケットごと（カーネルが2倍にするため、アプリケーションはrmem_max/2まで設定できる）
setsockopt(fd, SOL_SOCKET, SO_RCVBUF, &size, sizeof(size));
setsockopt(fd, SOL_SOCKET, SO_SNDBUF, &size, sizeof(size));

# 実効バッファサイズを確認
getsockopt(fd, SOL_SOCKET, SO_RCVBUF, &actual, &len);
# actual = 2 × 要求値（カーネルが管理用に2倍にする）
```

### UDPのバッファサイズ設定 {#udp-buffer-sizing}

UDPにはフロー制御がない。アプリケーションの読み取りが遅いと、パケットはドロップされる:

```bash
# 高レートの送信元のためにUDPソケットの受信バッファを増やす
cat /proc/sys/net/core/rmem_default  # 全ソケットのデフォルト受信バッファ
echo 26214400 > /proc/sys/net/core/rmem_default  # 25MB

# アプリケーション: バインドする前にバッファを増やす
int buf_size = 25 * 1024 * 1024;
setsockopt(fd, SOL_SOCKET, SO_RCVBUF, &buf_size, sizeof(buf_size));
```

## NICリングバッファのチューニング {#nic-ring-buffer-tuning}

NICはTXとRXのためにDMAリングバッファを保持している。これが満杯になると、パケットはカーネルに届く前にドロップされる:

```bash
# 現在のリングバッファサイズを表示
ethtool -g eth0
# Ring parameters for eth0:
# Pre-set maximums:
# RX: 4096  TX: 4096
# Current hardware settings:
# RX: 256   TX: 256  ← 高トラフィックには小さすぎることが多い

# リングバッファを増やす（高スループットなワークロードでは極めて重要）
ethtool -G eth0 rx 4096 tx 4096

# 変更を確認
ethtool -g eth0
```

リングバッファでのドロップは次で確認できる:

```bash
ethtool -S eth0 | grep -i "drop\|miss\|discard"
# rx_missed_errors: RXリングが満杯だったためにNICがドロップしたパケット数
```

## softnetバックログ {#softnet-backlog}

カーネルのCPUごとの受信バックログ（`sd->input_pkt_queue`）は、NAPIとプロトコル処理の間に位置する。これが満杯になると、パケットはドロップされる:

```bash
# バックログキューの長さ（CPUごと）
cat /proc/sys/net/core/netdev_max_backlog  # デフォルト: 1000

# 高いパケットレートに対応するため増やす
echo 10000 > /proc/sys/net/core/netdev_max_backlog

# ドロップを確認（列2 = dropped、列3 = time_squeeze）
cat /proc/net/softnet_stat
# 形式: total dropped time_squeeze 0 0 0 0 0 0 cpu_collision received_rps flow_limit_count
```

### softnet_statの各列の説明 {#softnet_stat-columns-explained}

```bash
# CPUごとのsoftnet統計をラベル付きで表示
awk 'BEGIN{OFS="\t"; print "CPU","total","dropped","time_squeeze"}
     {print NR-1, strtonum("0x"$1), strtonum("0x"$2), strtonum("0x"$3)}
     ' /proc/net/softnet_stat | head -10
```

| 列 | 意味 | 対処 |
|--------|---------|-----|
| total | softirqが処理したパケットの総数 | 正常 |
| dropped | ドロップしたパケット（バックログが満杯） | `netdev_max_backlog`を増やす |
| time_squeeze | poll()がバジェットを使い切った | `netdev_budget`を増やす |
| cpu_collision | 複数のCPUがCPUキューを奪い合った | RPS/RSSを有効にする |

## TCPのメモリプレッシャー {#tcp-memory-pressure}

カーネルは、TCPソケットが使うメモリの総量を制限している:

```bash
# TCPのメモリ制限 [プレッシャーが始まるページ数, ハードリミット, 最大]
cat /proc/sys/net/ipv4/tcp_mem
# → 94011 125349 188022（4KBページ単位 = 最大で合計約368MB）

# 現在の使用量
cat /proc/net/sockstat | grep TCP
# TCP: inuse 234 orphan 0 tw 12 alloc 248 mem 45

# memがtcp_mem[2]に近づくと: dmesgに"TCP: out of memory"
# 対処: tcp_memを増やす（またはメモリリークを修正する/接続数を減らす）
echo "768432 1024576 1536864" > /proc/sys/net/ipv4/tcp_mem
```

## バッファの問題の診断 {#diagnosing-buffer-issues}

### RXのボトルネック（アプリケーションの読み取りが遅い） {#rx-bottleneck-application-reads-slowly}

```bash
# ssでRecv-Qが大きいのは、データがアプリケーションを待っていることを意味する
ss -tn | awk '$2 > 10000 {print $2, $5}'

# ソケットのメモリ使用量を監視
cat /proc/net/sockstat

# UDPのドロップを確認
netstat -s | grep "receive buffer"
# "X packets pruned from receive queue because of socket buffer overrun"
```

### TXのボトルネック（送信バッファが満杯） {#tx-bottleneck-send-buffer-full}

```bash
# Send-Qが大きいのは、データが送信を待っていることを意味する（ネットワークが遅いか受信側ウィンドウが満杯）
ss -tn | awk '$3 > 0'

# 受信側ウィンドウが制約になっていないか確認（ゼロウィンドウプローブ）
netstat -s | grep "zero window"
```

### NICリングバッファでのドロップ {#nic-ring-buffer-drops}

```bash
# NICレベルのドロップを確認
ethtool -S eth0 | grep -i drop
watch -n 5 'ethtool -S eth0 | grep -i "drop\|miss" | awk "$2>0"'
```

### qdiscでのドロップ（TXキューが小さすぎる） {#qdisc-drops-tx-queue-too-small}

```bash
# qdiscのドロップを確認
tc -s qdisc show dev eth0
# "dropped"が0でなければ、qdiscがパケットをドロップしている

# qdiscのキュー長を増やす
ip link set dev eth0 txqueuelen 10000  # デフォルト1000
```

## 性能チェックリスト（完全版） {#complete-performance-checklist}

高スループットなサーバー（10G以上）の場合:

```bash
# 1. NICリングバッファ
ethtool -G eth0 rx 4096 tx 4096

# 2. ソケットバッファ
echo 67108864 > /proc/sys/net/core/rmem_max
echo 67108864 > /proc/sys/net/core/wmem_max
echo "4096 87380 67108864" > /proc/sys/net/ipv4/tcp_rmem
echo "4096 65536 67108864" > /proc/sys/net/ipv4/tcp_wmem

# 3. softnetバックログ
echo 30000 > /proc/sys/net/core/netdev_max_backlog
echo 1000  > /proc/sys/net/core/netdev_budget
echo 8000  > /proc/sys/net/core/netdev_budget_usecs

# 4. コネクショントラッキング（NATを使う場合）
echo 1048576 > /proc/sys/net/netfilter/nf_conntrack_max

# 5. TCPメモリ
echo "768432 1024576 1536864" > /proc/sys/net/ipv4/tcp_mem

# 6. オフロード（高スループットのため有効になっていることを確認）
ethtool -k eth0 | grep -E "scatter-gather|tcp-segmentation|generic-segmentation|generic-receive"
# スループットのためにはすべて"on"にする。レイテンシ重視/デバッグ時は無効にする
```

## 参考文献 {#further-reading}

- [/proc/net/snmpを理解する](proc-snmp.md) — これらのドロップを反映するカウンタ
- [ネットワークのトレーシング](net-tracing.md) — ドロップ箇所の診断方法
- [ネットワークデバイスとNAPI](napi.md) — NAPIのバジェットとsoftnetのチューニング
- [TCPの実装](tcp.md) — TCPのウィンドウとcwndがバッファとどう相互作用するか
