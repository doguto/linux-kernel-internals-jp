# コネクショントラッキング（conntrack） {#connection-tracking-conntrack}

> ステートフルなファイアウォールとNATのために、カーネルはネットワーク接続の状態をどう追跡するか

## conntrackが行うこと {#what-conntrack-does}

コネクショントラッキングは、アクティブなすべてのネットワークフローのテーブルを維持する。カーネルはパケットごとにこのテーブルでそのフローを検索し、状態（new、established、related、invalid）を判定する。これにより次のことが可能になる:

- **ステートフルなファイアウォール**: 明示的なルールなしに、送信方向の接続への応答を受け入れる
- **NAT**: 書き換えられた接続の両方向を追跡する
- **ヘルパーモジュール**: 関連する接続（FTPのデータチャネル、SIPのRTPなど）を追跡する

conntrackは`net/netfilter/nf_conntrack_core.c`に実装されており、PREROUTINGとOUTPUT（ルーティングの前、NATの前）にフックする。

## struct nf_conn {#struct-nf_conn}

追跡される各接続は`struct nf_conn`である:

```c
// include/net/netfilter/nf_conntrack.h
struct nf_conn {
    struct nf_conntrack ct_general;  // 参照カウント

    u32 timeout;  // 有効期限（jiffies32）

    // 2つのタプル: 各方向からどう見えるか
    // TCP接続の場合: クライアント10.0.0.1:54321 → サーバー8.8.8.8:80
    //   tuplehash[ORIGINAL]: src=10.0.0.1:54321 dst=8.8.8.8:80  proto=TCP
    //   tuplehash[REPLY]:    src=8.8.8.8:80 dst=10.0.0.1:54321  proto=TCP
    struct nf_conntrack_tuple_hash tuplehash[IP_CT_DIR_MAX];

    unsigned long status;  // IPS_*のビットマスク

    u_int32_t mark;    // connmark（iptablesのCONNMARKターゲットで設定できる）
    u_int32_t secmark; // SELinuxのセキュリティマーク

    union nf_conntrack_proto proto;  // プロトコルごとの状態（TCP: ステートマシン）
};
```

### 接続のステータスビット（IPS_*） {#connection-status-bits-ips_}

```c
IPS_EXPECTED      // ヘルパーによって予期されていた（例: FTPのデータ）
IPS_SEEN_REPLY    // 両方向のトラフィックを観測した
IPS_ASSURED       // メモリ逼迫時のGCでも生き残る（長寿命）
IPS_CONFIRMED     // ハッシュテーブルに挿入された（パケット処理を生き延びた）
IPS_SRC_NAT       // 送信元アドレスがNATされた
IPS_DST_NAT       // 宛先アドレスがNATされた
IPS_DYING         // 削除中
IPS_FIXED_TIMEOUT // タイムアウトが明示的に設定された
```

### TCP固有の状態 {#tcp-specific-state}

```c
// 接続ごとに追跡されるTCPの状態
union nf_conntrack_proto {
    struct ip_ct_tcp {
        enum tcp_conntrack state;  // TCP_CONNTRACK_SYN_SENT、ESTABLISHEDなど
        enum tcp_conntrack last_dir;
        u_int32_t seen[2];         // 両方向のウィンドウ情報
    } tcp;
    // UDP、ICMP、SCTPの状態...
};
```

## conntrackのタプル {#conntrack-tuple}

**タプル**は、一方向のパケットを一意に識別する:

```c
struct nf_conntrack_tuple {
    struct nf_conntrack_man {
        union nf_inet_addr u3;    // 送信元IP（IPv4またはIPv6）
        union nf_conntrack_man_proto {
            __be16 port;  // TCP/UDPの送信元ポート
            // ICMPのidなど
        } u;
    } src;

    struct {
        union nf_inet_addr u3;    // 宛先IP
        union {
            __be16 port;           // TCP/UDPの宛先ポート
        } u;
        u_int8_t protonum;         // IPPROTO_TCP、IPPROTO_UDP、...
        u_int8_t dir;              // ORIGINALまたはREPLY
    } dst;
};
```

`nf_conn`ごとに2つのタプルが保存される。元の方向のものと、応答方向のもの（元の方向を反転したもの）である。NATはこれによって機能する。NATは一方または両方のタプルを変更し、conntrackは応答タプルを使って戻りのトラフィックをマッチさせる。

## conntrackのライフサイクル {#conntrack-lifecycle}

### 1. 最初のパケット（NEW） {#1-first-packet-new}

```
Packet arrives → PREROUTING hook → nf_conntrack_in()
    → Lookup hash table by packet's tuple
    → Not found: create new nf_conn (state = UNCONFIRMED)
    → Hash table insertion deferred (to avoid early allocation)
    → Attach nf_conn to skb->_nfct
```

### 2. 確定（POSTROUTINGまたはLOCAL_INで） {#2-confirmed-at-postrouting-or-local_in}

```
nf_conntrack_confirm()
    → Hash the tuple
    → Insert both tuplehash entries into conntrack hash table
    → Status: IPS_CONFIRMED
```

### 3. 後続のパケット（ESTABLISHED） {#3-subsequent-packets-established}

```
Packet arrives → lookup by tuple → nf_conn found
    → Update state, refresh timeout
    → Set IPS_SEEN_REPLY when reply direction seen
    → Set IPS_ASSURED when connection is long-lived
```

### 4. タイムアウトとクリーンアップ {#4-timeout-and-cleanup}

パケットが観測されなければ、接続はタイムアウトする。タイムアウトはプロトコルと状態によって異なる:

```bash
# TCPのタイムアウト
cat /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_established  # 432000秒（5日）
cat /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_syn_sent      # 120秒
cat /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_time_wait     # 120秒
cat /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_close          # 10秒

# UDPのタイムアウト（状態はなく、有効期限のみ）
cat /proc/sys/net/netfilter/nf_conntrack_udp_timeout                # 30秒
cat /proc/sys/net/netfilter/nf_conntrack_udp_timeout_stream         # 180秒

# ICMP
cat /proc/sys/net/netfilter/nf_conntrack_icmp_timeout               # 30秒
```

## ファイアウォールルールのためのconntrackの状態 {#conntrack-states-for-firewall-rules}

iptables/nftablesから見える状態:

| 状態 | 意味 |
|-------|---------|
| `NEW` | 新しい接続の最初のパケット（TCPならSYN） |
| `ESTABLISHED` | 応答を観測した。接続は双方向に通信している |
| `RELATED` | 既存の接続に関連する新しい接続（FTPのデータ、ICMPエラー） |
| `INVALID` | どの接続にもマッチしない。不正な形式の可能性がある |
| `UNTRACKED` | `NOTRACK`/`raw`テーブルによって明示的にバイパスされた |

```bash
# iptables: established/relatedを受け入れ、invalidをドロップする
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -m conntrack --ctstate INVALID -j DROP

# nftables
nft add rule inet filter input ct state established,related accept
nft add rule inet filter input ct state invalid drop
```

## conntrackヘルパー: 関連する接続の追跡 {#conntrack-helpers-tracking-related-connections}

プロトコルによっては、二次的な接続（FTPのデータチャネル、SIPのRTPメディア）を開くものがある。**conntrackヘルパー**はプロトコルのメッセージを解析して、これらの二次的な接続を`EXPECT`する:

```bash
# FTPヘルパーをロード
modprobe nf_conntrack_ftp
# これでFTPのデータ接続は自動的にRELATEDとなり受け入れられる
```

ヘルパーは**エクスペクテーション**を作成する。これは、予期される二次的な接続にマッチするプレースホルダである:

```c
// FTPサーバーが"PORT x,x,x,x,a,b"を送ったとき:
// ヘルパーはエクスペクテーションを作成する: src=server dst=client dport=(a*256+b)
// データ接続が到着すると → エクスペクテーションにマッチ → state=RELATED
```

```bash
# 現在のエクスペクテーションを表示
cat /proc/net/nf_conntrack | grep RELATED
conntrack -L expect
```

## conntrackのマーク（connmark） {#conntrack-marks-connmark}

マークによって、ファイアウォールルールをルーティングやQoSと関連付けられる:

```bash
# すべてのHTTP接続にマークを付ける
iptables -t mangle -A POSTROUTING -p tcp --dport 80 \
    -j CONNMARK --set-mark 0x1

# conntrackのマークをパケットマークに復元する（後続のパケットで）
iptables -t mangle -A PREROUTING -j CONNMARK --restore-mark

# マークをルーティングで使う（ポリシールーティングのテーブル100）
ip rule add fwmark 0x1 table 100
```

## 監視とデバッグ {#monitoring-and-debugging}

```bash
# すべての接続を一覧表示
conntrack -L
conntrack -L --proto tcp --state ESTABLISHED

# 状態ごとに接続数を数える
conntrack -L 2>/dev/null | awk '{print $4}' | sort | uniq -c

# イベントをリアルタイムで監視
conntrack -E

# CPUごとの統計（検索、挿入、発見などの回数）
conntrack -S
# または
cat /proc/net/stat/nf_conntrack

# conntrackテーブルのサイズを出力
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# 手動で削除
conntrack -D -p tcp --src 10.0.0.1 --sport 54321

# テーブル全体をフラッシュ（警告: すべてのアクティブな接続の状態が失われる）
conntrack -F
```

## 接続レートの高い環境向けのチューニング {#tuning-for-high-connection-rate-environments}

```bash
# テーブルサイズを増やす（メモリと引き換えに）
echo 1048576 > /proc/sys/net/netfilter/nf_conntrack_max
# エントリあたりのメモリ: 約300バイト → 100万エントリで約300MB

# ハッシュテーブルのサイズ（モジュールのロード時に設定）
# nf_conntrackのhashsize = nf_conntrack_max / 8（典型的な値）
modprobe nf_conntrack hashsize=131072

# TIME_WAITエントリを減らす（混雑したサーバーでの主要な消費源）
echo 60 > /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_time_wait

# SYNフラッド対策: ハーフオープン接続を追跡しない（rawテーブル）
iptables -t raw -A PREROUTING -p tcp --syn -j NOTRACK
iptables -A INPUT -p tcp --syn -m conntrack --ctstate UNTRACKED -j ACCEPT
```

## 参考文献 {#further-reading}

- [Netfilterのアーキテクチャ](netfilter.md) — conntrackが属するフックフレームワーク
- [nftables対iptables](nftables-iptables.md) — conntrackルールの書き方
- [パケットの一生（受信）](life-of-packet-rx.md) — 受信パスのどこでconntrackが実行されるか
- [ssとipによるネットワークのデバッグ](net-debugging.md) — 接続を調査するためのツール
