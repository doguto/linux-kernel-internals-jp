# IPルーティング {#ip-routing}

> カーネルはパケットの行き先をどう決めるか: ルーティングテーブル、検索、そしてFIB

## ルーティングが行うこと {#what-routing-does}

受信・送信を問わずすべてのパケットについて、カーネルは「これはどこへ行くべきか?」と問う。ルーティングサブシステムはこう答える:

- **ローカル配送**: このパケットは自分宛てなので、ソケットへ上げる
- **転送**: このパケットは他者宛てなので、ネクストホップを見つけて送信する
- **到達不能**: 宛先へのルートがないので、ICMP unreachableを送る

この答えは**FIB（Forwarding Information Base）**、すなわちカーネルのルーティングテーブルから得られる。

## ルーティングテーブル {#the-routing-tables}

Linuxは（ポリシールーティングのために）複数のルーティングテーブルをサポートする:

```bash
# mainテーブル（テーブル254）— 標準のルート
ip route show

# すべてのテーブル
ip route show table all

# 特定の宛先を検索
ip route get 8.8.8.8
# → 8.8.8.8 via 192.168.1.1 dev eth0 src 192.168.1.100

# ルートを追加
ip route add 10.0.0.0/8 via 192.168.1.254 dev eth0

# ポリシールール: 送信元IP、マーク、インターフェースによってルーティングする
ip rule show
```

## FIBの内部: LC-trie {#fib-internals-lc-trie}

カーネルは、最長プレフィックス一致を効率的に行うため、FIBを**LC-trie**（Level Compressed trie。Patricia trieとも呼ばれる）として実装している:

```c
// include/net/ip_fib.h
struct fib_table {
    struct hlist_node   tb_hlist;
    u32                 tb_id;      // テーブルID（254 = main、255 = local）
    int                 tb_num_default;
    struct rcu_head     rcu;
    unsigned long       *tb_data;   // trie本体
};

struct fib_result {
    __be32          prefix;
    unsigned char   prefixlen;
    struct fib_info *fi;    // ネクストホップ情報: ゲートウェイ、デバイス、メトリック
    struct fib_nh_common *nhc; // 選択されたネクストホップ（ECMP用）
    // ...
};
```

検索は`fib_table_lookup()`で行う:

```c
// net/ipv4/fib_trie.c:1420
int fib_table_lookup(struct fib_table *tb, const struct flowi4 *flp,
                     struct fib_result *res, int fib_flags)
{
    // ルートからLC-trieをたどる
    // 各ノードで宛先アドレスからビットを取り出す
    // 一致する子をたどる
    // 葉に到達: 最長プレフィックス一致が見つかった
    // resにfib_info（ゲートウェイ、インターフェースなど）を格納する
}
```

## ルート検索: ip_route_input_noref() {#route-lookup-ip_route_input_noref}

受信時には、ルーティングは`ip_rcv_finish()`から呼ばれる:

```c
// net/ipv4/route.c
int ip_route_input_noref(struct sk_buff *skb, __be32 daddr, __be32 saddr,
                          dscp_t dscp, struct net_device *dev)
{
    // 1. FIB検索（Linux 3.6以降ルートキャッシュはない）
    err = fib_lookup(net, &fl4, &res, 0);

    // 2. 入力時の動作を決定する
    if (res.type == RTN_LOCAL) {
        // 自分宛てのパケット — input = ip_local_deliverでrtableを作成
        // skb->_skb_refdstに&rt->dstを設定
    } else if (res.type == RTN_UNICAST) {
        // パケットは転送されるべき
        ip_route_input_slow(skb, daddr, saddr, ...);
    } else {
        // RTN_UNREACHABLE、RTN_BLACKHOLE、RTN_PROHIBIT
        goto martian_destination;
    }
}
```

## ルートキャッシュ: dst_entry {#route-cache-dst_entry}

ルーティングの判断は`struct dst_entry`にキャッシュされる:

```c
// include/net/dst.h
struct dst_entry {
    struct net_device *dev;         // 出力デバイス
    struct  dst_ops   *ops;         // ip_dst_opsなど
    void (*output)(struct net *, struct sock *, struct sk_buff *);
    // → ローカルならip_output()、転送ならip_forward()

    unsigned long     _metrics;     // MTU、RTTなど
    unsigned long     expires;      // キャッシュの有効期限
};
```

送信パスでは、ルートは`sk->sk_dst_cache`にキャッシュされる:

```c
// ip_queue_xmit()はまずこれを確認する:
rt = (struct rtable *)__sk_dst_check(sk, 0);
if (!rt) {
    rt = ip_route_output_ports(net, fl4, sk, daddr, saddr, ...);
    sk_setup_caps(sk, &rt->dst);  // キャッシュする
}
```

## ポリシールーティング {#policy-routing}

Linuxは複数のルーティングテーブルと、どのテーブルを使うかを選択する**ルール**をサポートする:

```bash
# ルールを追加: 10.0.0.0/24からのパケットはテーブル100を使う
ip rule add from 10.0.0.0/24 table 100
ip route add default via 192.168.2.1 table 100

# ルールは優先度順に評価される
ip rule show
# 0:      from all lookup local   (localテーブル: localhost、ブロードキャスト)
# 32766:  from all lookup main    (mainテーブル: 通常のルート)
# 32767:  from all lookup default (defaultテーブル: 通常は空)
```

ルールは次のものにマッチできる:

- 送信元アドレス（`from`）
- 宛先アドレス（`to`）
- 受信インターフェース（`iif`）
- ファイアウォールマーク（`fwmark`）
- TOS/DSCPフィールド

## ECMP: 等コストマルチパス {#ecmp-equal-cost-multipath}

同じ宛先に対してメトリックが等しいルートが複数ある = ECMP:

```bash
# 2つの等コストルートを追加（トラフィックはeth0とeth1に分散される）
ip route add 10.0.0.0/8 nexthop via 192.168.1.1 dev eth0 weight 1 \
                         nexthop via 192.168.2.1 dev eth1 weight 1

# デフォルトのハッシュ: 5タプル（送信元/宛先IP、送信元/宛先ポート、プロトコル）
# L4ハッシュにより、同じフローは常に同じ経路を通る
```

```bash
# ECMPのハッシュアルゴリズムを確認
cat /proc/sys/net/ipv4/fib_multipath_hash_policy
# 0 = L3（送信元+宛先IP）
# 1 = L4（5タプル）
# 2 = L3+L4
```

## 近隣サブシステム: ARP {#neighbor-subsystem-arp}

IPルーティングはネクストホップのIPを解決するが、送信にはMACアドレスが必要である。**近隣サブシステム**がARPによってIP → MACの対応付けを行う:

```c
// net/core/neighbour.c
struct neighbour {
    struct neighbour __rcu *next;
    struct neigh_table  *tbl;       // IPv4ではarp_tbl
    struct neigh_parms  *parms;
    unsigned long       confirmed;  // 最後に到達可能性を確認した時刻
    u8                  flags;
    u8                  nud_state;  // NUD_REACHABLE、NUD_STALE、NUD_PROBE、...
    u8                  ha[ALIGN(MAX_ADDR_LEN, sizeof(unsigned long))]; // MACアドレス
    struct hh_cache     hh;         // 高速パス用にキャッシュされたL2ヘッダ
    int (*output)(struct neighbour *, struct sk_buff *);
};
```

NUD（Neighbor Unreachability Detection）の状態:

```
INCOMPLETE → REACHABLE (ARP reply received)
REACHABLE → STALE (no recent confirmation)
STALE → PROBE (send unicast ARP probe)
PROBE → REACHABLE (got reply) or FAILED (no reply)
```

```bash
# ARPテーブルを表示
ip neigh show

# 静的ARPエントリを追加
ip neigh add 10.0.0.5 lladdr 52:54:00:12:34:56 dev eth0

# staleエントリをフラッシュ
ip neigh flush dev eth0

# ARPキャッシュの寿命を調整
cat /proc/sys/net/ipv4/neigh/default/gc_stale_time  # デフォルト: 60秒
```

## ルーティング問題の診断 {#diagnosing-routing-issues}

```bash
# ルーティング判断をシミュレート
ip route get 8.8.8.8

# ルーティングテーブルの変更を監視
ip monitor route

# ルートキャッシュのミス数を数える（該当する場合）
cat /proc/net/rt_cache_stat  # （新しいカーネルでは削除済み）

# FIBテーブルの統計
cat /proc/net/fib_triestat
# Aver depth: trieの平均の深さ
# Max depth: 最悪ケースの検索の深さ

# ICMP unreachableメッセージ（ルーティングの失敗）
netstat -s | grep "destination unreachable"
```

## 参考文献 {#further-reading}

- [パケットの一生（受信）](life-of-packet-rx.md) — ip_rcv_finishの中でルーティングがどこに位置するか
- [パケットの一生（送信）](life-of-packet-tx.md) — ip_queue_xmitがルーティングをどこで使うか
- [Netfilterのアーキテクチャ](netfilter.md) — ルーティング判断と並んで実行されるフック
- [TCPの実装](tcp.md) — TCPが送信元IPの選択にルーティングをどう使うか
