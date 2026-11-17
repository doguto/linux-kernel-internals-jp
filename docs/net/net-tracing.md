# ネットワークスタックのトレーシング {#tracing-the-network-stack}

> パケットレベルのネットワークデバッグのためのftrace、perf、BPFツール

## ネットワークのトレースイベント {#network-trace-events}

カーネルはネットワークスタックの要所でtracepointを発行しており、ftraceやperfから利用できる:

```bash
# 利用可能なネットワークのトレースイベントを一覧表示
ls /sys/kernel/debug/tracing/events/net/
ls /sys/kernel/debug/tracing/events/tcp/
ls /sys/kernel/debug/tracing/events/skb/

# 主なイベント:
# net:net_dev_xmit           - デバイスがパケットを送信した
# net:net_dev_queue          - パケットが送信のためにキューに入れられた
# net:netif_rx               - パケットを受信した（旧NAPIパス）
# net:netif_receive_skb      - NAPIがパケットを処理した
# skb:consume_skb            - sk_buffが解放された（通常パス）
# skb:kfree_skb              - sk_buffが解放された（ドロップパス）
# tcp:tcp_send_reset         - RSTを送信した
# tcp:tcp_rcv_space_adjust   - 受信バッファがリサイズされた
# tcp:tcp_destroy_sock       - ソケットが破棄された
```

## kfree_skbによるパケットドロップのトレーシング {#tracing-packet-drops-with-kfree_skb}

`skb:kfree_skb`は、sk_buffがドロップされるたびに発火する。`reason`フィールドがその理由を示す:

```bash
# ドロップのトレーシングを有効化
echo 1 > /sys/kernel/debug/tracing/events/skb/kfree_skb/enable
cat /sys/kernel/debug/tracing/trace_pipe

# 出力例:
# <irq>-0 [000] ..... 123.456789: kfree_skb: skbaddr=0xffff... location=0xffffffff... protocol=2048 reason=NETFILTER_DROP

# あるいはtrace-cmdで:
trace-cmd record -e skb:kfree_skb sleep 5
trace-cmd report | grep kfree_skb
```

### ドロップ理由コード（Linux 5.17以降） {#drop-reason-codes-linux-517}

正確なドロップ箇所を特定するために`reason`フィールドが追加された:

```bash
# すべてのドロップ理由を見る
grep -r "SKB_DROP_REASON" /usr/include/linux/ 2>/dev/null | head -20
# SKB_DROP_REASON_NOT_SPECIFIED
# SKB_DROP_REASON_NO_SOCKET
# SKB_DROP_REASON_PKT_TOO_SMALL
# SKB_DROP_REASON_TCP_CSUM
# SKB_DROP_REASON_TCP_FILTER
# SKB_DROP_REASON_UDP_CSUM
# など
```

### dropwatch（BCC）を使う {#using-dropwatch-bcc}

```bash
# dropwatch: パケットのドロップ箇所とレートを表示
# BCCツールが必要
dropwatch -l kas  # カーネルアドレスのシンボル

# あるいはbpftraceで:
bpftrace -e '
tracepoint:skb:kfree_skb {
    @drops[kstack] = count();
}
interval:s:5 {
    print(@drops);
    clear(@drops);
}'
```

## TCPイベントのトレーシング {#tracing-tcp-events}

```bash
# TCPの状態変化とリセットをトレース
trace-cmd record -e tcp:tcp_send_reset \
                 -e tcp:tcp_receive_reset \
                 sleep 10

# RSTストーム（接続リセット）を監視
bpftrace -e '
tracepoint:tcp:tcp_send_reset {
    @resets[args->saddr, args->daddr] = count();
}
interval:s:5 { print(@resets); clear(@resets); }'

# 再送をトレース
bpftrace -e '
kprobe:tcp_retransmit_skb {
    @retrans[comm] = count();
}'
```

## ネットワークイベントのためのperf {#perf-for-network-events}

```bash
# システム全体でネットワークイベントを数える
perf stat -e \
    net:net_dev_xmit,\
    net:netif_receive_skb,\
    skb:kfree_skb \
    -a sleep 10

# パケットの流れを記録して分析
perf record -e net:netif_receive_skb -ag sleep 5
perf report

# ネットワーク処理のオーバーヘッド（CPUサイクル）
perf top -e net:net_dev_xmit
```

## bpftraceワンライナー {#bpftrace-one-liners}

```bash
# デバイスごとの受信レートを表示
bpftrace -e '
tracepoint:net:netif_receive_skb {
    @pkts[args->dev_name] = count();
    @bytes[args->dev_name] = sum(args->len);
}
interval:s:1 {
    print(@pkts); print(@bytes);
    clear(@pkts); clear(@bytes);
}'

# NICでの受信からソケットへの配送までのパケットレイテンシをトレース
bpftrace -e '
tracepoint:net:netif_receive_skb { @ts[args->skbaddr] = nsecs; }
kprobe:tcp_rcv_established {
    $key = (uint64)arg1;
    if (@ts[$key]) {
        @lat = hist(nsecs - @ts[$key]);
        delete(@ts[$key]);
    }
}'

# どのプロセスがUDPパケットを送信しているかを追跡
bpftrace -e '
kprobe:udp_sendmsg {
    @[comm, pid] = count();
}'

# TCP接続の確立にかかる時間を表示
bpftrace -e '
kprobe:tcp_connect { @start[tid] = nsecs; }
kretprobe:tcp_finish_connect {
    @connect_ms = hist((nsecs - @start[tid]) / 1000000);
    delete(@start[tid]);
}'
```

## Wireshark / tcpdump {#wireshark-tcpdump}

パケットレベルのキャプチャには:

```bash
# インターフェース上でキャプチャ
tcpdump -i eth0 -n

# ホストとポートで絞り込み
tcpdump -i eth0 -n host 8.8.8.8 and port 53

# Wireshark用にファイルへ保存
tcpdump -i eth0 -w /tmp/capture.pcap

# 特定のプロトコルをキャプチャ
tcpdump -i eth0 -n 'tcp[tcpflags] & tcp-syn != 0'  # SYNパケットのみ
tcpdump -i eth0 -n 'tcp[tcpflags] & tcp-rst != 0'  # RSTパケットのみ

# ファイルから読み込む
tcpdump -r /tmp/capture.pcap -nn

# 任意のインターフェース（ループバックを含む）でキャプチャ
tcpdump -i any -n
```

### AF_PACKETソケット（tcpdumpが使うもの） {#af_packet-sockets-what-tcpdump-uses}

tcpdumpはAF_PACKETソケットを使う。AF_PACKETソケットは、`__netif_receive_skb`のレベル、つまりパケットがネットワークスタックで処理される前に、すべてのパケットのコピーを受け取る。後でiptablesによってドロップされるパケットでもtcpdumpに見えるのはこのためである。

```bash
# 高負荷時にtcpdumpはパケットを取りこぼすことがある: ドロップを確認する
tcpdump -i eth0 -w /dev/null -v  # "X packets captured, Y dropped by kernel"を表示

# ドロップを減らすためにキャプチャバッファを増やす
tcpdump -i eth0 -B 65536 -w /dev/null  # 65MBのバッファ
```

## nftables/iptablesによるトレーシング {#tracing-with-nftablesiptables}

ファイアウォールルールをパケットレベルでデバッグするには:

```bash
# nftables: 特定のルールにマッチしたパケットをトレース
nft add table inet debug
nft add chain inet debug input { type filter hook input priority -200\; }
nft add rule inet debug input ip saddr 10.0.0.1 meta nftrace set 1
nft monitor trace

# iptables: LOGターゲット（カーネルログ / syslogに出力される）
iptables -I INPUT 1 -p tcp --dport 80 -j LOG --log-prefix "HTTP: " --log-level 4
journalctl -k | grep "HTTP: "
```

## net_ratelimitとカーネルログのドロップ {#net_ratelimit-and-kernel-log-drops}

カーネルは、ログの氾濫を防ぐために一部のネットワークメッセージをレート制限する。`net_ratelimit: X callbacks suppressed`のようなメッセージが見える場合、カーネルはログメッセージを捨てている:

```bash
# ログのレート制限を緩める
echo 5000 > /proc/sys/net/core/message_burst  # バースト
echo 10 > /proc/sys/net/core/message_cost     # インターバルあたりのトークン
```

## ネットワークデバッグへの体系的なアプローチ {#systematic-approach-to-network-debugging}

1. **まずカウンタを確認する**: `/proc/net/dev`、`ethtool -S eth0`、`/proc/net/snmp`
2. **ドロップ箇所を特定する**: `skb:kfree_skb` tracepointまたは`dropwatch`
3. **ソケットキューを確認する**: `ss -tn | awk '$2>0 || $3>0'`
4. **conntrackを確認する**: `conntrack -S`、`cat /proc/sys/net/netfilter/nf_conntrack_count`
5. **パケットキャプチャ**: プロトコルレベルの分析には`tcpdump`
6. **ファイアウォールのトレース**: nftablesの`meta nftrace`またはiptablesのLOG

## 参考文献 {#further-reading}

- [ssとipによるネットワークのデバッグ](net-debugging.md) — ソケットとルートの調査
- [/proc/net/snmpを理解する](proc-snmp.md) — プロトコルカウンタ
- [Netfilterのアーキテクチャ](netfilter.md) — ファイアウォールでのドロップがどこで起きるか
- [パケットの一生（受信）](life-of-packet-rx.md) — ドロップが起こりうる経路の全体
