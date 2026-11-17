# ssとipによるネットワークのデバッグ {#network-debugging-with-ss-and-ip}

> ソースコードを見ずにネットワークの状態を調べるための必須ツール

## ss: ソケット統計 {#ss-socket-statistics}

`ss`（socket statistics）は`netstat`の現代的な代替である。`/proc/net/tcp`をパースする代わりにカーネルのデータ構造から直接読み取るため、大幅に高速である。

### 基本的な使い方 {#basic-usage}

```bash
# すべてのTCPソケット
ss -tn

# プロセス情報付き
ss -tnp

# すべての種類のソケット
ss -a

# listen中のソケットのみ
ss -tnl

# UDPソケット
ss -un

# Unixドメインソケット
ss -x
```

### 状態とアドレスによる絞り込み {#filter-by-state-and-address}

```bash
# 特定の状態
ss -tn state established
ss -tn state time-wait
ss -tn state syn-sent

# 複数の状態
ss -tn state established state close-wait

# ポートで絞り込み
ss -tn sport = :22         # 送信元ポート22
ss -tn dport = :80         # 宛先ポート80
ss -tn '( dport = :443 or sport = :443 )'

# アドレスで絞り込み
ss -tn dst 8.8.8.8
ss -tn src 10.0.0.0/8

# TIME_WAITソケットを数える（よくある性能問題）
ss -tn state time-wait | wc -l
```

### ソケットの詳細情報 {#detailed-socket-info}

```bash
# TCPの内部情報を表示（輻輳状態、cwnd、RTTなど）
ss -tni

# ESTABLISHED接続の出力例:
# ESTAB 0 0 10.0.0.1:22 10.0.0.2:54321
#  cubic wscale:7,7 rto:208 rtt:4.2/0.5 ato:40 mss:1448 pmtu:1500
#  rcvmss:931 advmss:1448 cwnd:10 ssthresh:2147483647
#  bytes_sent:12345 bytes_acked:12345 bytes_received:67890
#  segs_out:456 segs_in:234
#  send 27.6Mbps pacing_rate 55.2Mbps delivery_rate 27.6Mbps
#  delivered:234 app_limited
#  reord_seen:0 lost:0 retrans:0/0 dsack_dups:0
#  rcv_rtt:4.2 rcv_space:14600 rcv_ssthresh:87380 minrtt:3.8

# 主なフィールド:
#   rtt/rtt_var: 平滑化RTTとその分散
#   cwnd: 輻輳ウィンドウ（セグメント数）
#   ssthresh: スロースタート閾値
#   retrans: 累積/現在の再送数
#   delivery_rate: 実際の送信レート
```

### ソケットのメモリ使用量 {#socket-memory-usage}

```bash
# 送信/受信キューのサイズとメモリを表示
ss -tm

# 出力の列:
# Recv-Q: 受信済みだがアプリケーションがまだ読んでいないバイト数
# Send-Q: 送信済みだがまだACKされていないバイト数

# Recv-Qが大きい: アプリケーションの読み取りが追いついていない
# Send-Qが大きい: ネットワークが遅い / 受信側ウィンドウが小さい
```

## ip: ネットワークの設定と状態 {#ip-network-configuration-and-state}

### インターフェース情報 {#interface-information}

```bash
# 統計付きで全インターフェースを表示
ip -s link show
# → インターフェースごとのrx/txパケット数、バイト数、エラー、ドロップを表示

# 特定のインターフェースを表示
ip link show eth0

# IPアドレスを表示
ip addr show
ip addr show dev eth0

# インターフェースのイベント（リンクのup/down、アドレス変更）を監視
ip monitor link
ip monitor address
```

### ルートの調査 {#route-inspection}

```bash
# ルーティングテーブルを表示
ip route show
ip route show table all  # すべてのルーティングテーブル

# ルーティング検索（カーネルが行う処理をシミュレート）
ip route get 8.8.8.8
# → 8.8.8.8 via 192.168.1.1 dev eth0 src 192.168.1.100 uid 0

# ルートの変更を監視
ip monitor route

# キャッシュされたルートを表示
ip route show cache
```

### ARP/近隣テーブル {#arpneighbor-table}

```bash
# ARPテーブルを表示
ip neigh show
# → 192.168.1.1 dev eth0 lladdr 52:54:00:12:34:56 REACHABLE

# 状態で絞り込み
ip neigh show nud reachable  # 確認済みのエントリのみ
ip neigh show nud stale      # 再確認が必要なエントリ

# staleエントリをフラッシュ
ip neigh flush dev eth0

# 静的エントリを追加
ip neigh add 10.0.0.5 lladdr 52:54:00:11:22:33 dev eth0

# ARPイベントを監視
ip monitor neigh
```

## /proc/net: ソケットテーブルを読む {#procnet-reading-socket-tables}

```bash
# TCPソケットテーブル（16進アドレス）
cat /proc/net/tcp
# local_address rem_address state tx_queue rx_queue uid inode
# 0101007F:0035 00000000:0000 0A ...  ← 127.0.0.1:53 listen中
# 010011AC:01BB 02010A0A:E76C 01 ...  ← ESTABLISHED

# 16進のアドレス:ポートを変換
python3 -c "import socket; print(socket.inet_ntoa(bytes.fromhex('010011AC')[::-1]))"
# → 172.17.0.1

# UDPソケット
cat /proc/net/udp

# ネットワークインターフェースの統計
cat /proc/net/dev
# eth0: 12345678 23456 0 0 0 0 0 0  98765432 12345 0 0 0 0 0 0
#       ↑rx bytes ↑rx pkts ↑err     ↑tx bytes ↑tx pkts

# ARPテーブル
cat /proc/net/arp
```

## 手早い診断パターン {#quick-diagnostic-patterns}

```bash
# 状態ごとの接続数を数える（全体的なヘルスチェック）
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn

# 接続数の多いリモートIP上位10件
ss -tn | awk '{print $5}' | grep -v Peer | sed 's/:[0-9]*$//' | sort | uniq -c | sort -rn | head

# 何がlistenしているかを調べる（プロセス名付き）
ss -tnlp

# ポートが使用中か確認する
ss -tnlp | grep ':8080'

# ソケットキューの蓄積をリアルタイムで監視
watch -n 1 'ss -tn | awk "NR==1 || \$2>0 || \$3>0"'

# 送信キューが大きいソケット（ネットワークの問題の可能性）
ss -tn | awk '$3 > 0'  # Send-Q > 0

# 受信キューが大きいソケット（アプリケーションが遅い）
ss -tn | awk '$2 > 10000'  # Recv-Q > 10KB
```

## 参考文献 {#further-reading}

- [/proc/net/snmpを理解する](proc-snmp.md) — プロトコルカウンタの詳細
- [ネットワークのトレーシング](net-tracing.md) — パケットレベルのデバッグのためのftraceとBPF
- [TCPの実装](tcp.md) — ss -tniの各フィールドが内部的に何を意味するか
- [ソケット層の概要](socket-layer.md) — ssが読み取るソケット構造体
