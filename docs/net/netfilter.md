# Netfilterのアーキテクチャ {#netfilter-architecture}

> カーネルのパケットフィルタリングとNATのフレームワーク: フック、テーブル、コネクショントラッキング

## Netfilterとは何か {#what-netfilter-is}

Netfilterは、IPスタック内の固定された地点に埋め込まれたフックのフレームワークである。ネットワークを通過するすべてのパケットはこれらのフックを通り、そこで登録されたコールバック（iptables、nftables、conntrackなどから）がパケットを検査・変更できる。

Netfilter自体はフレームワークにすぎない。ポリシー（ファイアウォールルール、NATなど）は、これらのフックに登録するモジュールによって実装される。

## 5つのフックポイント {#the-five-hook-points}

```
                          Routing Decision
                                │
NIC → PREROUTING ──────────────→ FORWARD → POSTROUTING → NIC
                   ↓                                 ↑
                LOCAL IN                         LOCAL OUT
                   ↓                                 ↑
               Local Socket ────────────────────────┘
```

```c
// include/uapi/linux/netfilter.h
enum nf_inet_hooks {
    NF_INET_PRE_ROUTING,   // L2 の後、ルーティング判断の前
    NF_INET_LOCAL_IN,      // ルーティングの後、ローカル宛てのパケット用
    NF_INET_FORWARD,       // 転送されるパケット用（自分宛てではない）
    NF_INET_LOCAL_OUT,     // ローカルで生成されたパケット、経路検索の後
    NF_INET_POST_ROUTING,  // ルーティングの後、送信の直前
};
```

| フック | タイミング | 主な用途 |
|------|------|-------------|
| PREROUTING | NICの後、ルーティングの前 | DNAT（ポートフォワーディング）、conntrack |
| LOCAL_IN | ソケットへの配送の前 | INPUTチェイン（ファイアウォール） |
| FORWARD | 受信と送信の間 | FORWARDチェイン（ルーターのファイアウォール） |
| LOCAL_OUT | ソケットの後、ルーティングの前 | OUTPUTチェイン |
| POSTROUTING | NICからの送信の前 | SNAT（マスカレード） |

## struct nf_hook_ops: フックの登録 {#struct-nf_hook_ops-registering-a-hook}

どのモジュールでも、これらのフックにコールバックを登録できる:

```c
// include/linux/netfilter.h
struct nf_hook_ops {
    nf_hookfn    *hook;       // コールバック: (priv, skb, state) → 判定
    struct net_device *dev;   // NULL = 全デバイス、または特定のデバイス
    void         *priv;       // フックに渡されるプライベートデータ
    u8           pf;          // プロトコルファミリ: NFPROTO_IPV4, NFPROTO_IPV6
    unsigned int hooknum;     // どのフックか: NF_INET_PRE_ROUTING など
    int          priority;    // 同じ地点にある複数のフック間の順序
                              // NF_IP_PRI_CONNTRACK = -200
                              // NF_IP_PRI_FILTER = 0
                              // NF_IP_PRI_NAT_SRC = +100
};
```

フックの判定（verdict）:

- `NF_ACCEPT` — 処理を続行する
- `NF_DROP` — パケットを破棄する
- `NF_STOLEN` — フックが所有権を引き取った（以降の処理はない）
- `NF_QUEUE` — ユーザー空間に送る（NFQUEUE用）
- `NF_REPEAT` — このフックをもう一度呼ぶ

```c
// 例: すべての TCP SYN パケットをログに出す単純なフック
static unsigned int syn_log_hook(void *priv, struct sk_buff *skb,
                                  const struct nf_hook_state *state)
{
    struct iphdr *iph = ip_hdr(skb);
    if (iph->protocol == IPPROTO_TCP) {
        struct tcphdr *th = tcp_hdr(skb);
        if (th->syn && !th->ack)
            pr_info("SYN from %pI4 to %pI4\n", &iph->saddr, &iph->daddr);
    }
    return NF_ACCEPT;
}
```

## iptablesとnftables {#iptables-vs-nftables}

どちらもNetfilterにルールを設定するユーザー空間のツールである。違いは、カーネル内部での表現にある:

| 観点 | iptables | nftables |
|--------|----------|----------|
| カーネルモジュール | ip_tables | nf_tables |
| ルールの格納 | テーブル/チェイン/ルールのリスト | バイトコードを伴うセットとマップ |
| 性能 | 線形走査 | セットの検索（hash/rbtree） |
| IPv4/IPv6 | 別々（iptables/ip6tables） | 統合（nft） |
| ARP | 別々（arptables） | 統合 |
| アトミックなルール更新 | なし（ルールごとに追加） | あり（トランザクション） |

どちらも内部的には、ルールをNetfilterのフックコールバックとして設定する。

### iptablesのチェインとテーブル {#iptables-chains-and-tables}

```bash
# iptables はルールをテーブルとチェインに整理する
iptables -L -n -v              # filter テーブルのルールを一覧表示
iptables -t nat -L -n -v      # NAT テーブル
iptables -t mangle -L -n -v   # mangle テーブル（QoS マーク）
iptables -t raw -L -n -v      # raw テーブル（conntrack のバイパス）

# テーブルと、それが動作するフック:
# filter: INPUT, FORWARD, OUTPUT
# nat:    PREROUTING (DNAT), INPUT (local DNAT), OUTPUT (local DNAT), POSTROUTING (SNAT)
# mangle: 5つすべてのフック
# raw:    PREROUTING, OUTPUT（conntrack より前に実行される）
```

### nftablesでの同等の操作 {#nftables-equivalent}

```bash
# nftables は統一されたコマンドを使う
nft list ruleset

# テーブルとチェインを作成
nft add table inet myfilter
nft add chain inet myfilter input { type filter hook input priority 0\; policy drop\; }
nft add rule inet myfilter input tcp dport 22 accept

# セットを使った効率的なマッチング（線形走査なし）
nft add set inet myfilter allowed_ips { type ipv4_addr\; }
nft add element inet myfilter allowed_ips { 10.0.0.1, 10.0.0.2 }
nft add rule inet myfilter input ip saddr @allowed_ips accept
```

## コネクショントラッキング（conntrack） {#connection-tracking-conntrack}

**conntrack**はステートフルなパケット検査の層である。カーネルを通過するすべてのコネクションを追跡し、次のことを可能にする:

- ステートフルなファイアウォールルール（`-m state --state ESTABLISHED,RELATED`）
- NAT（DNATもSNATも双方向を書き換える必要がある）
- コネクションを意識したアプリケーション（NFQUEUEやnf_conntrackイベント経由）

### struct nf_conn {#struct-nf_conn}

```c
// include/net/netfilter/nf_conntrack.h
struct nf_conn {
    struct nf_conntrack ct_general;   // 参照カウント

    u32 timeout;                      // 期限（jiffies）

    // 2つのタプル: 元の方向と応答の方向
    // TCP 10.0.0.1:54321 → 8.8.8.8:80 の場合:
    //   tuplehash[ORIGINAL]: src=10.0.0.1:54321 dst=8.8.8.8:80
    //   tuplehash[REPLY]:    src=8.8.8.8:80 dst=10.0.0.1:54321
    struct nf_conntrack_tuple_hash tuplehash[IP_CT_DIR_MAX];

    unsigned long status;   // IPS_CONFIRMED, IPS_SEEN_REPLY, IPS_ASSURED, ...

    u_int32_t mark;         // connmark（ファイアウォールルール / ルーティング用）
    u_int32_t secmark;      // SELinux のセキュリティマーク

    union nf_conntrack_proto proto;  // TCP の状態、UDP のタイムアウトなど
};
```

### conntrackのライフサイクル {#conntrack-lifecycle}

```
Packet arrives → PREROUTING → conntrack lookup
    New packet: create nf_conn entry (state: NEW)
    Known packet: update state (ESTABLISHED/RELATED)
    ↓
Filter hook: rule can match by state
    -m conntrack --ctstate NEW,ESTABLISHED
    ↓
POSTROUTING → NAT rewrites src/dst if needed
    → nf_conn stores the rewrite for the reply direction
```

### conntrackの状態を見る {#viewing-conntrack-state}

```bash
# 追跡中のすべてのコネクションを一覧表示
conntrack -L
# tcp      6 431999 ESTABLISHED src=10.0.0.1 dst=8.8.8.8 sport=54321 dport=80
#                               src=8.8.8.8 dst=10.0.0.1 sport=80 dport=54321

# 状態ごとにコネクション数を数える
conntrack -L | awk '{print $4}' | sort | uniq -c | sort -rn

# conntrack のイベントを監視
conntrack -E

# conntrack テーブルの統計
cat /proc/net/stat/nf_conntrack
```

### conntrackのチューニング {#conntrack-tuning}

```bash
# 追跡するコネクションの最大数
cat /proc/sys/net/netfilter/nf_conntrack_max
echo 524288 > /proc/sys/net/netfilter/nf_conntrack_max

# 現在の使用量
cat /proc/sys/net/netfilter/nf_conntrack_count

# TCP ESTABLISHED のタイムアウト（デフォルト: 5日!）
cat /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_established
echo 7200 > /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_established

# コネクション生成レートの高いサーバー向け: TIME_WAIT のタイムアウトを調整
cat /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_time_wait  # default: 120s
```

conntrackテーブルが満杯になると、新しいコネクションが破棄される。これは、高負荷のNATゲートウェイやコネクションの多いサーバーでよくある問題である。

## NAT: DNATとSNAT {#nat-dnat-and-snat}

NATは、パケットを変更し、その書き換えをconntrackエントリに保存するNetfilterフックとして実装されている:

```bash
# DNAT: ポート80を内部サーバーにリダイレクト（PREROUTING）
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.1.10:8080

# SNAT: 外向きトラフィックをマスカレード（POSTROUTING）
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# nftables の場合:
nft add rule inet nat prerouting tcp dport 80 dnat to 192.168.1.10:8080
nft add rule inet nat postrouting oifname "eth0" masquerade
```

conntrackエントリは双方向を保存しているため、応答パケットは逆方向に自動的に書き換えられる（DNATの応答はSNATになる、など）。明示的な逆方向のルールは不要である。

## NF_HOOKマクロ: フックの呼ばれ方 {#nf_hook-macro-how-hooks-are-called}

IPスタック内の各フックポイントでは次のようになっている:

```c
// ip_rcv() (net/ipv4/ip_input.c) からの例
return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING,
               net, NULL, skb, dev, NULL,
               ip_rcv_finish);
```

`NF_HOOK`は登録された各フックを優先度順に呼び出す。すべてが`NF_ACCEPT`を返せば、`ip_rcv_finish`が呼ばれる。どれか1つでも`NF_DROP`を返せば、処理は即座に止まる。

## 参考文献 {#further-reading}

- [nftablesとiptables](nftables-iptables.md) — ルールの構文と移行
- [コネクショントラッキング](conntrack.md) — conntrackの内部の詳細
- [パケットの一生（受信）](life-of-packet-rx.md) — 受信パスのどこにNetfilterフックが現れるか
- [IPルーティング](ip-routing.md) — ルーティングとNetfilterの相互作用
