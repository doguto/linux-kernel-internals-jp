# ネットワーキング {#networking}

> socket()から回線まで: パケットがLinuxネットワークスタックをどう通り抜けるか

## スタックの概観 {#the-stack-at-a-glance}

```
Application: send(fd, buf, len)
        │
        ▼
   Socket layer          ← struct socket、プロトコル非依存のAPI
        │
        ▼
   TCP / UDP             ← 輻輳制御、再送、ポート
        │
        ▼
   IP / routing          ← FIB検索、netfilterフック、フラグメンテーション
        │
        ▼
   Traffic control       ← qdisc、シェーピング、優先度付け
        │
        ▼
   Device driver         ← NAPI、リングバッファ、オフロード
        │
        ▼
      NIC                ← DMA、割り込み / ポーリング
```

すべてのパケットは[sk_buff](sk-buff.md)によって運ばれる。1ページだけ読むなら、そこから始めるとよい。

## このセクションのページ {#pages-in-this-section}

### 基礎 {#fundamentals}

| ページ | 扱う内容 |
|------|----------------|
| [ネットワークスタックの概要](network-stack-overview.md) | 各層と、それらがどうつながるか |
| [ソケット層](socket-layer.md) | struct socket、ファイルディスクリプタ、プロトコルファミリ |
| [sk_buff](sk-buff.md) | 他のすべてが共有するパケットのデータ構造 |
| [NAPI](napi.md) | 割り込みの緩和: IRQストームからポーリングへ |

### パケットのライフサイクル {#packet-lifecycle}

| ページ | 扱う内容 |
|------|----------------|
| [パケットの一生: RX](life-of-packet-rx.md) | NICからアプリケーションまで、ステップごとに |
| [パケットの一生: TX](life-of-packet-tx.md) | アプリケーションからNICまで、ステップごとに |
| [connect()で何が起きるか](connect.md) | カーネル側から見たスリーウェイハンドシェイク |

### TCP/IP {#tcpip}

| ページ | 扱う内容 |
|------|----------------|
| [TCPの内部](tcp.md) | ステートマシン、バッファ、タイマー |
| [TCP輻輳制御](tcp-congestion.md) | CUBIC、BBR、プラガブルなアルゴリズム |
| [UDP](udp.md) | 薄いパス、そしてそれでもなお驚かされる点 |
| [IPルーティング](ip-routing.md) | FIB、ルール、マルチパス |
| [epoll](epoll.md) | 大規模なレディネス通知 |

### フィルタリングとトラフィック制御 {#filtering-traffic-control}

| ページ | 扱う内容 |
|------|----------------|
| [Netfilter](netfilter.md) | フック、テーブル、チェーン |
| [nftables対iptables](nftables-iptables.md) | なぜ置き換えが起きたか |
| [コネクショントラッキング](conntrack.md) | conntrackエントリ、NAT、ヘルパー |
| [トラフィック制御（tc/qdisc）](tc-qdisc.md) | キューイングディシプリン、シェーピング、fq_codel |

### 高性能化とオフロード {#high-performance-offload}

| ページ | 扱う内容 |
|------|----------------|
| [XDP](xdp.md) | スタックより前でのパケット処理 |
| [AF_XDP](af-xdp.md) | ゼロコピーのユーザー空間パケットI/O |
| [kTLS](ktls.md) | カーネル内でのTLSレコード処理 |
| [パケットソケット](packet-socket.md) | AF_PACKET、tcpdumpから見た世界 |

### namespaceと仮想化 {#namespaces-virtualization}

| ページ | 扱う内容 |
|------|----------------|
| [ネットワークnamespace](net-namespaces.md) | namespaceごとのスタック、vethペア |
| [コンテナネットワーキング](container-networking.md) | ブリッジ、NAT、オーバーレイの基礎 |
| [TUN/TAP](tun-tap.md) | ユーザー空間のネットワークデバイス、VPN |

### インターフェースと可観測性 {#interfaces-observability}

| ページ | 扱う内容 |
|------|----------------|
| [Netlink](netlink.md) | カーネルのネットワーク設定API |
| [ネットワークのデバッグ](net-debugging.md) | ss、ethtool、ドロップの診断 |
| [ネットワークのトレーシング](net-tracing.md) | データパス上のtracepointとBPF |
| [バッファのチューニング](net-buffer-tuning.md) | ソケットバッファ、rmem/wmem、自動チューニング |
| [/proc/net/snmpリファレンス](proc-snmp.md) | プロトコルカウンタの読み方 |

### 事件簿 {#war-stories}

| ページ | 扱う内容 |
|------|----------------|
| [ネットワークの事件簿](war-stories.md) | 6件の実際のCVEを、根本原因を突き止め、各修正の背後にある実際のメーリングリストでの議論をたどって解説する |
