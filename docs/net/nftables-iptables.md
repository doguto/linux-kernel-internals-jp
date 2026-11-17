# nftables対iptables {#nftables-vs-iptables}

> Linuxの2つのパケットフィルタリングフロントエンド: 違い、移行、共存

## Netfilterとの関係 {#the-relationship-to-netfilter}

`iptables`も`nftables`も、カーネルのNetfilterフレームワークにルールをインストールする**ユーザー空間のツール**である。違いは、それらのルールをどのカーネルモジュールが処理するかにある:

- **iptables** → `ip_tables`カーネルモジュール（レガシー、テーブルごとのモジュール）
- **nftables** → `nf_tables`カーネルモジュール（統合された、現代的なもの）

根底にあるフックの仕組みは同じである。変わるのは、ルールの表現、マッチング、適用のされ方である。

## なぜnftablesが作られたか {#why-nftables-was-created}

iptablesは大きな技術的負債を抱えていた:

1. **ファミリごとに別々のツール**: `iptables`（IPv4）、`ip6tables`（IPv6）、`arptables`、`ebtables`。それぞれが独自のコードベースを持つ
2. **線形なルール走査**: すべてのパケットがチェーン内のすべてのルールを走査する
3. **アトミックな更新がない**: ルールを1つ追加するのに、`iptables-restore`でルールセット全体を再適用する必要がある
4. **セットのサポートが貧弱**: 大量のIPリストとのマッチングには、数千のルールか`ipset`拡張が必要になる

nftables（Linux 3.13でマージされた）は、単一の統合されたフレームワークでこれらすべてを解決する。

## 並べて比較する {#side-by-side-comparison}

### 基本的な構文 {#basic-syntax}

```bash
# iptables: 特定のIPからのSSHをブロック
iptables -A INPUT -s 10.0.0.5 -p tcp --dport 22 -j DROP

# nftablesでの同等の設定
nft add rule inet filter input ip saddr 10.0.0.5 tcp dport 22 drop
```

### テーブルとチェーン {#tables-and-chains}

```bash
# iptables: 暗黙のテーブル（filter、nat、mangle、raw、security）
# 新しいテーブルは作成できない

# nftables: テーブルとチェーンを明示的に作成する
nft add table inet myfilter
nft add chain inet myfilter input { type filter hook input priority 0\; policy accept\; }

# iptablesのテーブルはnftablesのtypeに対応する:
# filterテーブル  → type filter
# natテーブル     → type nat
# mangleテーブル  → type filter（type routeはOUTPUTフックでの再ルーティング専用）
```

### ルールの一覧表示 {#rule-listing}

```bash
# iptables
iptables -L -n -v --line-numbers
iptables -t nat -L -n -v

# nftables
nft list ruleset            # すべて
nft list table inet filter  # 特定のテーブル
nft list chain inet filter input  # 特定のチェーン
```

### カウンタ {#counters}

```bash
# iptables: ルールごとのカウンタ（パケット数、バイト数）
iptables -L -n -v
# → Chain INPUT: 12345 packets, 1234567 bytes

# nftables: ルールごとのカウンタはオプション
nft add rule inet filter input ip protocol tcp counter
nft list ruleset  # カウンタの値をインラインで表示
```

## セットベースのマッチング {#set-based-matching}

ここがnftablesの真価を発揮するところで、大きなセットに対するマッチングを効率的に行える:

```bash
# iptables: 10,000個のIPには10,000個のルールが必要（線形走査）
# （またはipset拡張）

# nftables: ハッシュセット、O(1)の検索
nft add set inet filter blocked_ips {
    type ipv4_addr\;
    flags dynamic, timeout\;
    timeout 1h\;   # エントリは1時間後に期限切れになる
}

nft add rule inet filter input ip saddr @blocked_ips drop

# セットにIPを追加
nft add element inet filter blocked_ips { 10.0.0.1, 10.0.0.2, 192.168.1.0/24 }

# 名前付きマップ: IP → アクションのマッチング
nft add map inet filter client_policy {
    type ipv4_addr : verdict\;
}
nft add element inet filter client_policy { 10.0.0.1 : accept, 10.0.0.2 : drop }
nft add rule inet filter input ip saddr vmap @client_policy
```

## アトミックなルール更新 {#atomic-rule-updates}

```bash
# iptables: 非アトミック — フラッシュと再ロードの間に競合の窓がある
iptables -F INPUT
iptables-restore < /etc/iptables/rules.v4

# nftables: アトミックなトランザクション
nft -f /etc/nftables.conf  # アトミックに適用されるか、全体が失敗する

# nftables.confの例:
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;
        ct state established,related accept
        iif lo accept
        tcp dport { 22, 80, 443 } accept
    }
}
```

## nftablesによるNAT {#nat-with-nftables}

```bash
# DNAT: ポート80を内部サーバーへリダイレクト
nft add table ip nat
nft add chain ip nat prerouting { type nat hook prerouting priority -100\; }
nft add chain ip nat postrouting { type nat hook postrouting priority 100\; }

nft add rule ip nat prerouting tcp dport 80 dnat to 192.168.1.10:8080
nft add rule ip nat postrouting oifname "eth0" masquerade
```

## conntrackとの統合 {#conntrack-integration}

```bash
# iptables: -m conntrack（または-m state）
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# nftables: ct state式
nft add rule inet filter input ct state established,related accept
nft add rule inet filter input ct state invalid drop

# コネクションマークでマッチ
nft add rule inet filter input ct mark 0x1 accept
```

## 優先度の値 {#priority-values}

```bash
# フックの優先度は、同じフックに複数のルール/テーブルがあるときの順序を決める
# 数値が小さいほど先に実行される

# 標準の優先度:
#  -400: NF_IP_PRI_CONNTRACK_DEFRAG （IPのデフラグメンテーション）
#  -300: NF_IP_PRI_RAW              （rawテーブル）
#  -200: NF_IP_PRI_CONNTRACK        （conntrack）
#   -50: NF_IP_PRI_SELINUX_FIRST    （SELinux）
#     0: NF_IP_PRI_FILTER           （filterテーブル）
#    50: NF_IP_PRI_SELINUX_LAST
#   100: NF_IP_PRI_NAT_SRC          （POSTROUTINGでのSNAT）
#   300: NF_IP_PRI_CONNTRACK_HELPER

nft add chain inet myapp input { type filter hook input priority -50\; }
# このチェーンは標準のfilterルールより前に実行される
```

## 共存と移行 {#coexistence-and-migration}

iptablesとnftablesは1つのシステム上で共存できるが、（現代的なディストリビューションで利用可能な）`iptables-nft`は、iptablesの構文をラップしつつ`nf_tables`バックエンドを使う:

```bash
# iptablesがどのバックエンドを使っているか確認
iptables --version
# iptables v1.8.7 (nf_tables)  ← nf_tablesバックエンドを使用
# iptables v1.8.7 (legacy)     ← ip_tablesバックエンドを使用

# iptablesのルールをnftablesに変換
iptables-translate -A INPUT -p tcp --dport 22 -j ACCEPT
# → nft add rule ip filter INPUT tcp dport 22 counter accept

# ルールセット全体を変換
iptables-restore-translate -f /etc/iptables/rules.v4 > /etc/nftables.conf
```

## デバッグとロギング {#debugging-and-logging}

```bash
# nftables: マッチしたパケットをログに記録
nft add rule inet filter input tcp dport 22 log prefix "SSH: " level info

# ルールにマッチしたパケットを数える（counterキーワード）
nft add rule inet filter input tcp dport 80 counter accept

# ルールを通るパケットをトレース（nftables 0.9.1以降）
nft add table inet trace_debug
nft add chain inet trace_debug input { type filter hook input priority -200\; }
nft add rule inet trace_debug input ip saddr 10.0.0.1 meta nftrace set 1
nft monitor trace

# iptables: LOGターゲット
iptables -A INPUT -p tcp --dport 22 -j LOG --log-prefix "SSH: " --log-level 4
```

## 性能比較 {#performance-comparison}

単純なケース（小さなルールセット）では、性能は同程度である。大きなルールセットでは:

- **iptables**: パケットあたりO(n)（n = ルール数）
- **セットを使うnftables**: セットのメンバーシップ判定はO(1)、区間セットはO(log n)

```bash
# ipset（iptablesの拡張）対nftablesのネイティブセット
# ipsetは10万個のIPを、ルール1つとほぼ同じ時間でマッチできる
# nftablesのセットは統合されており、別のツールは不要

# 高性能なファイアウォールには、次も検討するとよい:
# - カーネルのネットワークスタックより前で早期にドロップするXDP
# - eBPFソケットフィルタ
```

## 参考文献 {#further-reading}

- [Netfilterのアーキテクチャ](netfilter.md) — 両ツールの基盤となるフックフレームワーク
- [コネクショントラッキング](conntrack.md) — 両ツールが使うconntrackモジュール
- [XDP](xdp.md) — 最大の性能を得るための、スタック前でのパケットフィルタリング
