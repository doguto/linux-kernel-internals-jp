# /proc/net/snmpとnetstatを理解する {#understanding-procnetsnmp-and-netstat}

> TCP、UDP、IPの問題を診断するためのプロトコルレベルのカウンタ

## /proc/net/snmp {#procnetsnmp}

`/proc/net/snmp`は、SNMPのRFCに由来するMIB（Management Information Base）カウンタを公開する。各カウンタは、IP、TCP、UDPのいずれかのレベルにおける特定のプロトコルイベントについて教えてくれる。

```bash
cat /proc/net/snmp
# Ip: Forwarding DefaultTTL InReceives InHdrErrors ...
# Ip: 2 64 1234567 0 ...        ← 値
# Icmp: InMsgs InErrors InCsumErrors ...
# Icmp: 123 0 0 ...
# Tcp: RtoAlgorithm RtoMin RtoMax MaxConn ActiveOpens ...
# Tcp: 1 200 120000 -1 5678 ...
# Udp: InDatagrams NoPorts InErrors OutDatagrams ...
# Udp: 12345 0 0 23456 ...
```

## 主要なIPカウンタ {#key-ip-counters}

```bash
# IPの統計を抽出
awk '/^Ip:/ && /[0-9]/' /proc/net/snmp | head -1

# 重要なフィールド:
# InReceives       - IP層が受信したパケット
# InHdrErrors      - IPヘッダのチェックサムエラー
# InAddrErrors     - 宛先アドレスが不正（このホストにそのIPがない）
# ForwDatagrams    - 転送したパケット（ルーターのみ）
# InDiscards       - ヘッダは正しいがドロップされた（バッファの逼迫）
# InDelivers       - トランスポート層に配送したパケット
# OutRequests      - ローカルで発生したパケット
# OutDiscards      - 送信キューのオーバーフローでドロップしたパケット
# OutNoRoutes      - 宛先へのルートがないためドロップしたパケット
# ReasmOKs         - 再構築に成功したフラグメント
# ReasmFails       - フラグメント再構築の失敗
# FragOKs          - フラグメント化した送信パケット
# FragFails        - フラグメント化の失敗（DFビットが立っているがMTUを超えた）
```

```bash
# ワンライナー: IPカウンタをラベル付きで表示
paste <(head -1 /proc/net/snmp | tr ' ' '\n') \
      <(sed -n '2p' /proc/net/snmp | tr ' ' '\n') | grep -v "^Ip"
```

## 主要なTCPカウンタ {#key-tcp-counters}

```bash
# TCPの統計をラベル付きで抽出（わかりやすさのためpythonを使う）
python3 -c "
data = open('/proc/net/snmp').read()
lines = [l.split() for l in data.split('\n') if l.startswith('Tcp')]
for k, v in zip(lines[0][1:], lines[1][1:]):
    if int(v) > 0:
        print(f'{k}: {v}')
"
```

| カウンタ | 意味 | よくある原因 |
|---------|---------------|--------------|
| `ActiveOpens` | 開始したTCP接続（connect()） | 通常の送信方向の接続 |
| `PassiveOpens` | 受け付けたTCP接続（accept()） | 通常の受信方向の接続 |
| `AttemptFails` | 確立処理中に失敗した接続 | 接続拒否、SYNのタイムアウト |
| `EstabResets` | リセットされたESTABLISHED接続 | アプリケーションのクラッシュ、ネットワークの問題 |
| `CurrEstab` | 現在確立している接続 | 現在の負荷 |
| `InSegs` | 受信したセグメントの総数 | トラフィック量 |
| `OutSegs` | 送信したセグメントの総数 | トラフィック量 |
| `RetransSegs` | 再送したセグメント | **ネットワーク品質の問題** |
| `InErrs` | エラー（不正なチェックサム）付きで受信したセグメント | ハードウェア/NICの問題 |
| `OutRsts` | 送信したRSTセグメント | 接続の拒否 |
| `InCsumErrors` | チェックサムエラー | NICオフロードのバグ、ハードウェアの問題 |

`RetransSegs/OutSegs`の比率が上昇している（1〜2%超）なら、ネットワークでパケットロスが起きていることを示す:

```bash
# 再送率を計算
python3 -c "
import re
data = open('/proc/net/snmp').read()
tcp = dict(zip(re.findall(r'Tcp: (.*)', data)[0].split(),
               re.findall(r'Tcp: (.*)', data)[1].split()))
ratio = int(tcp['RetransSegs']) / max(int(tcp['OutSegs']), 1) * 100
print(f'Retransmit ratio: {ratio:.2f}%')
print(f'RetransSegs: {tcp[\"RetransSegs\"]}')
print(f'OutSegs: {tcp[\"OutSegs\"]}')
"
```

## 主要なUDPカウンタ {#key-udp-counters}

| カウンタ | 意味 |
|---------|---------------|
| `InDatagrams` | 受信して配送したUDPデータグラム |
| `NoPorts` | listenしているソケットのないUDPを受信した → ICMP port unreachableを送信 |
| `InErrors` | バッファオーバーフロー、チェックサムエラー |
| `OutDatagrams` | 送信したUDPデータグラム |
| `RcvbufErrors` | ソケットの受信バッファが満杯だったためドロップした |
| `SndbufErrors` | ソケットの送信バッファが満杯だったためドロップした |

`NoPorts`が大きいなら、何かがリスナーのいないUDPポートへ送信している。`RcvbufErrors`が大きいなら、アプリケーションの読み取りが追いついていない。

## /proc/net/netstat: 拡張TCP統計 {#procnetnetstat-extended-tcp-stats}

より詳細なTCPカウンタ（SNMP MIBには含まれない）:

```bash
cat /proc/net/netstat
# TcpExt: SyncookiesSent SyncookiesRecv SyncookiesFailed EmbryonicRsts ...
```

主要な`TcpExt`カウンタ:

| カウンタ | 意味 |
|---------|---------------|
| `SyncookiesSent` | 送信したSYN cookie（SYNキューが満杯だった） |
| `SyncookiesRecv` | 受信した正当なSYN cookieのACK |
| `SyncookiesFailed` | 不正だったSYN cookie（攻撃の可能性） |
| `EmbryonicRsts` | SYN_RECV状態の接続に送信したRST |
| `ListenDrops` | acceptキューが満杯だったためドロップした接続 |
| `ListenOverflows` | acceptキューのオーバーフロー回数 |
| `TCPHPHits` | ヘッダ予測のヒット（確立済み接続の高速パス） |
| `TCPPureAcks` | 受信した純粋なACK |
| `TCPHPAcks` | ヘッダ予測の高速パスで処理したACK |
| `TCPRenoRecovery` | Reno方式のロス回復の回数 |
| `TCPSackRecovery` | SACKによるロス回復の回数 |
| `TCPFastRetrans` | 発生した高速再送 |
| `TCPTimeouts` | RTOタイムアウト（遅い回復） |
| `TCPLostRetransmit` | 再送したセグメントが再び失われた |
| `TCPDSACKOldSent` | 古いデータに対する重複SACK |
| `TCPAbortOnTimeout` | 再試行の上限によって中断された接続 |
| `TCPAbortOnData` | クローズ中の接続にデータが到着したため中断された接続 |

```bash
# TCPの問題をリアルタイムで監視
watch -n 2 'awk "/^TcpExt:/ && NR==2 {
    split(\$0, a, \" \"); for(i=1;i<=NF;i++) print a[i]
}" /proc/net/netstat | paste - <(
awk "/^TcpExt:/ && NR==1 {
    split(\$0, a, \" \"); for(i=1;i<=NF;i++) print a[i]
}" /proc/net/netstat) | grep -v "^0"'
```

## netstat（ツール） {#netstat-the-tool}

`netstat`は`/proc/net/`から読み取り、カウンタを読みやすい形式で表示する:

```bash
# プロトコル統計（/proc/net/snmpと/proc/net/netstatから）
netstat -s

# TCPの統計に絞り込む
netstat -s | grep -i -E "retransmit|error|fail|drop|reset"

# アクティブな接続
netstat -tn          # ss -tnと同様

# listenソケット
netstat -tnl
```

接続の一覧表示については`netstat`は非推奨で`ss`が推奨されるが、`netstat -s`は統計を見るのに引き続き有用である。

## カウンタによる診断: よくある問題 {#diagnosing-with-counters-common-issues}

### パケットロス {#packet-loss}

```bash
# RetransSegsの上昇はロスを示す
watch -n 5 'cat /proc/net/snmp | grep ^Tcp | awk "NR==2 {print \"Retrans:\", \$13, \"OutSegs:\", \$12}"'

# NICのドロップも確認する
ethtool -S eth0 | grep -i drop
```

### acceptキューのオーバーフロー {#accept-queue-overflow}

```bash
# ListenOverflows: backlogが満杯だったためドロップした接続
netstat -s | grep "listen"
# 0でない場合:
#   listen()のbacklogを増やす: listen(fd, 4096)
#   /proc/sys/net/core/somaxconnを増やす
```

### SYNフラッドの検出 {#syn-flood-detection}

```bash
# SyncookiesSent > 0はSYNキューが満杯だったことを意味する
netstat -s | grep "syncookies"
# あるいはSYN-RECVの数が多くないかを監視する:
ss -tn state syn-recv | wc -l
```

### UDPバッファのドロップ {#udp-buffer-drops}

```bash
# RcvbufErrors: アプリケーションの読み取りが追いついていない
netstat -s | grep "receive buffer"
# 対処: ソケットバッファ（SO_RCVBUF）を増やすか、読み取り頻度を上げる
```

## 参考文献 {#further-reading}

- [ssとipによるネットワークのデバッグ](net-debugging.md) — 接続ごとの調査
- [ネットワークのトレーシング](net-tracing.md) — イベントベースのデバッグ
- [TCPの実装](tcp.md) — これらのカウンタが内部的に何を測っているか
- [ネットワークバッファのチューニング](net-buffer-tuning.md) — バッファオーバーフロー問題の解決
