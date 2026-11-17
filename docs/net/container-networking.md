# コンテナネットワーキング {#container-networking}

> Docker、Kubernetes、コンテナランタイムはネットワーク分離をどう実装しているか

## 構成要素 {#the-building-blocks}

コンテナネットワーキングは、3つのLinuxプリミティブから構築される:

1. **ネットワークnamespace** — 分離されたネットワークスタック（インターフェース、ルート、iptables）
2. **vethペア** — 2つのnamespaceをつなぐ仮想Ethernetケーブル
3. **Linuxブリッジ** — vethペアをつなぐソフトウェアスイッチ

```
Host network namespace                 Container namespace
┌─────────────────────────┐            ┌──────────────────────┐
│  eth0 (physical)        │            │  eth0 (veth1's peer) │
│  docker0 (bridge)       │            │  10.0.0.2/24         │
│  ├── veth0 (10.0.0.1)   │════════════│  (sees only this)    │
│  └── veth2              │════════════│                      │
│  iptables MASQUERADE    │            └──────────────────────┘
└─────────────────────────┘
```

## vethペア {#veth-pairs}

vethペアは、背中合わせに接続された2つの仮想インターフェースである。一方の端に送られたパケットは、もう一方の端から出てくる:

```bash
# vethペアを作成
ip link add veth0 type veth peer name veth1

# veth0はホストのnamespaceに残る
# veth1はコンテナのネットワークnamespaceに移動する
ip link set veth1 netns <container_pid>

# 両端を設定
ip addr add 10.0.0.1/24 dev veth0
ip link set veth0 up

# コンテナ内で:
nsenter -t <container_pid> -n ip addr add 10.0.0.2/24 dev veth1
nsenter -t <container_pid> -n ip link set veth1 up
nsenter -t <container_pid> -n ip route add default via 10.0.0.1
```

### カーネル内のveth {#veth-in-the-kernel}

```c
/* drivers/net/veth.c */
struct veth_priv {
    struct net_device __rcu *peer;      /* もう一方の端 */
    atomic64_t               dropped;
    struct veth_rq          *rq;        /* RXキュー */
    unsigned int             requested_headroom;
    bool                     rx_notify_masked;
    struct ptr_ring         *xdp_ring;  /* XDPリングバッファ */
};

/* TX: 単にピアのRXへ届けるだけ */
static netdev_tx_t veth_xmit(struct sk_buff *skb, struct net_device *dev)
{
    struct veth_priv *rcv_priv, *priv = netdev_priv(dev);
    struct net_device *rcv;

    rcv = rcu_dereference(priv->peer);
    if (!rcv)
        goto drop;

    /* ピアのsoftirq RXパスへ直接届ける */
    if (veth_forward_skb(rcv, skb, rq, use_napi) == NET_RX_DROP) {
        /* ... */
    }
    return NETDEV_TX_OK;
}
```

実際のハードウェアは一切関与しない。パケットは`netif_rx()`を介して、あるnamespaceから別のnamespaceへ渡る。

## Linuxブリッジ {#linux-bridge}

ブリッジはソフトウェアのL2スイッチとして動作する。vethのホスト側インターフェースは、ブリッジのポートとして接続される:

```bash
# ブリッジを作成（docker0に相当）
ip link add docker0 type bridge
ip addr add 172.17.0.1/16 dev docker0
ip link set docker0 up

# vethのホスト側をブリッジに接続
ip link set veth0 master docker0
ip link set veth2 master docker0

# これで、docker0に接続されたコンテナ同士が通信でき、
# ホストのルーティングテーブルを経由した通信もできる
```

### ブリッジの転送 {#bridge-forwarding}

```c
/* net/bridge/br_forward.c */
/* ブリッジのポートにフレームが到着したとき: */
void br_forward(const struct net_bridge_port *to, struct sk_buff *skb,
                bool local_rcv, bool local_orig)
{
    /* FDB（転送データベース）で宛先MACを検索する */
    struct net_bridge_fdb_entry *dst = br_fdb_find_rcu(to->br, dest, vid);
    if (dst) {
        /* ユニキャスト: 特定のポートへ転送する */
        __br_forward(dst->dst, skb, local_orig);
    } else {
        /* 未知のユニキャスト: すべてのポートへフラッディングする */
        br_flood(to->br, skb, BR_PKT_UNICAST, local_rcv, local_orig);
    }
}
```

ブリッジは、受信パケットから学習してMAC→ポートの転送データベース（FDB）を維持する。`bridge fdb show`でこれを表示できる。

## NAT / マスカレード {#nat-masquerade}

コンテナはインターネットに到達する必要がある。ホストは送信トラフィックをNATする:

```bash
# iptablesのNATマスカレード（Dockerが追加するもの）
iptables -t nat -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE

# nftablesでの同等の設定
nft add rule ip nat POSTROUTING ip saddr 172.17.0.0/16 oif != "docker0" masquerade
```

コンテナ`172.17.0.2`が`8.8.8.8`へ送信するとき:
1. パケットはコンテナから`eth0` → veth → ブリッジ → ホストのルーティングを経て出ていく
2. ホストは送信元`172.17.0.2`を見る → MASQUERADEルールにマッチする
3. コネクショントラッキングが対応関係を記録する: `172.17.0.2:54321 → host_ip:RANDOM`
4. 送信元IPがホストのIPに書き換えられる
5. 応答はホストのIPに戻ってくる → conntrack → 逆変換される → コンテナへ転送される

## ポートの公開 {#port-publishing}

```bash
# Docker: ホストのポート8080 → コンテナのポート80に対応付ける
docker run -p 8080:80 nginx

# iptablesのルールに変換される:
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
iptables -t filter -A DOCKER -d 172.17.0.2 -p tcp --dport 80 -j ACCEPT
```

## オーバーレイネットワーク（VXLAN） {#overlay-networks-vxlan}

複数ホストにまたがるコンテナネットワーキング（Kubernetes、Docker Swarm）では、オーバーレイネットワークがコンテナのトラフィックをUDP/VXLANでカプセル化する:

```
Node 1 (10.0.0.1)                    Node 2 (10.0.0.2)
Container A (192.168.0.1)            Container B (192.168.0.2)
   │                                        │
   veth                                     veth
   │                                        │
   VTEP (VXLAN Tunnel Endpoint)             VTEP
   │  encapsulate in UDP/VXLAN              │  decapsulate
   │  outer: 10.0.0.1→10.0.0.2              │
   └────────── physical network ────────────┘
```

```bash
# VXLANオーバーレイを作成（Flannel/Calico/Ciliumはこれを自動的に行う）
ip link add vxlan0 type vxlan id 100 \
    local 10.0.0.1 \
    remote 10.0.0.2 \
    dstport 4789

ip link set vxlan0 up
ip addr add 192.168.0.1/24 dev vxlan0

# コンテナB向けのルートを追加
ip route add 192.168.0.2/32 via 10.0.0.2 dev vxlan0
```

### カーネル内のVXLAN {#vxlan-in-the-kernel}

```c
/* drivers/net/vxlan/vxlan_core.c */
static netdev_tx_t vxlan_xmit(struct sk_buff *skb, struct net_device *dev)
{
    struct vxlan_dev *vxlan = netdev_priv(dev);

    /* 宛先のVTEPを検索する */
    struct vxlan_fdb *f = vxlan_find_mac(vxlan, eth_hdr(skb)->h_dest, vni);

    /* カプセル化: 内側のEthernetフレーム → VXLANヘッダ → UDP → 外側のIP */
    vxlan_encap_bypass(skb, vxlan, dst, vni, ...);
    udp_tunnel_xmit_skb(vxlan->vn4_sock, rt, skb,
                         src, remote_ip, ...);
    return NETDEV_TX_OK;
}
```

## ネットワークポリシー（iptables対eBPF） {#network-policies-iptables-vs-ebpf}

従来のコンテナのファイアウォールは、ランタイムが挿入するiptablesルールを使う。Podが数千になると、これがボトルネックになる（O(n)のルール走査）:

```bash
# kube-proxy（従来型）: 数千のiptablesルール
iptables -L -n | wc -l
# 5000+  ← サービスのエンドポイントごとに1ルール

# Cilium（eBPFベース）: O(1)のBPFマップ検索
# iptablesを一切使わない — ポリシーはXDP/TCのBPFフックで適用される
bpf_redirect_map(&cilium_lb4_backends, backend_id, 0)
```

Ciliumやその他のeBPFベースのCNIプラグイン（eBPFを使うCalico、VXLAN-on-eBPFを使うFlannel）は、iptablesをBPFマップで置き換える。ルール数にかかわらず定数時間で検索できる。

## コンテナネットワーキングの観察 {#observing-container-networking}

```bash
# すべてのネットワークnamespaceを一覧表示
lsns -t net

# コンテナのネットワークnamespaceに入る
ip netns exec <ns_name> ip addr show
# または
nsenter -t <container_pid> -n ip addr show

# vethのピア関係を表示
ip link show | grep -A1 "veth"
# あるいは:
for dev in /sys/class/net/veth*; do
    peer_idx=$(cat "$dev/ifindex")  # 誤り — 実際にはiflinkである
    echo "$(basename $dev) → peer ifindex $(cat $dev/iflink)"
done

# ブリッジのFDB（MAC学習テーブル）
bridge fdb show dev docker0

# ブリッジを通るパケットの流れを監視
tcpdump -i docker0 -n

# NATのコネクショントラッキング
conntrack -L -p tcp | head -20

# vethごとの統計
ip -s link show veth0

# VXLANトンネルの統計
ip -s link show vxlan0
```

## 参考文献 {#further-reading}

- [Netfilterのアーキテクチャ](netfilter.md) — iptables/nftablesのフック
- [コネクショントラッキング](conntrack.md) — NATの状態追跡
- [XDP](xdp.md) — eBPFによるネットワーキングの高速化
- [AF_XDP](af-xdp.md) — ゼロコピーのユーザー空間パケット処理
- [cgroup: namespace](../cgroups/namespaces.md) — ネットワークnamespaceの内部
- [cgroup: コンテナの分離](../cgroups/container-isolation.md) — コンテナの完全なセットアップ
