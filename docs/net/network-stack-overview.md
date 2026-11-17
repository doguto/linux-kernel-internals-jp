# なぜネットワークスタックはこれほど複雑なのか {#why-is-the-network-stack-so-complex}

> Linuxネットワーキングサブシステムを理解するためのメンタルモデル

## 根本的な課題 {#the-fundamental-challenge}

ネットワーキングは単一の問題ではない。きれいに組み合わさらなければならない、何十もの階層化された問題である:

- 異なるハードウェア（Ethernet、WiFi、InfiniBand、ループバック）
- 異なるアドレスファミリ（IPv4、IPv6、AF_UNIX、AF_PACKET）
- 異なるトランスポートプロトコル（TCP、UDP、SCTP、DCCP）
- 異なるユースケース（Webサーバー、データベース、リアルタイムシステム、コンテナ）
- 厳しい要件: 正しさ、性能、セキュリティ、拡張性

Linuxネットワークスタックが複雑なのは、これらすべてを同時に解決しており、要件が加わるたびに30年かけて進化してきたからである。

## 階層モデル {#the-layer-model}

スタックはOSI/IPモデルを反映しているが、カーネルの用語で表すと次のようになる:

```
Application (userspace)
    │  read/write/send/recv  (system calls)
    ▼
Socket layer          struct socket, struct sock, proto_ops
    │  sendmsg/recvmsg
    ▼
Transport layer       TCP (net/ipv4/tcp.c), UDP (net/ipv4/udp.c)
    │  ip_queue_xmit / udp_rcv
    ▼
Network layer (IP)    net/ipv4/ip_output.c, net/ipv4/ip_input.c
    │  routing, fragmentation, NAT (via Netfilter hooks)
    ▼
Link layer            net/core/dev.c, driver-specific
    │  __netif_receive_skb (RX) / dev_queue_xmit (TX)
    ▼
Hardware              NIC driver, NAPI, DMA ring buffers
```

各層は同じ`sk_buff`構造体を使い、データポインタを前後に動かしてヘッダを付加・除去する。**層の間でデータはコピーされない**。同じバッファがスタック全体を通過する。

## 受信パスを1枚の図で {#the-receive-path-in-one-picture}

```
NIC DMA → ring buffer
    ↓ interrupt (or polling)
NAPI poll (NET_RX_SOFTIRQ)
    ├── XDP programs (if attached)  ← early drop/redirect (before skb allocation)
    ↓ skb = napi_build_skb()
__netif_receive_skb()
    ↓ protocol demux (Ethernet type)
    ├── Packet sockets (tcpdump)
    ├── Netfilter (PREROUTING hook)
    ↓
ip_rcv() → ip_rcv_finish()
    ↓ routing decision
    ├── Forward: ip_forward() → Netfilter (FORWARD) → ip_output()
    └── Local: ip_local_deliver()
              ↓ Netfilter (INPUT hook)
              ↓ transport demux (proto = TCP/UDP/ICMP)
              tcp_v4_rcv() / udp_rcv()
              ↓ socket lookup (hash table)
              sk->sk_receive_queue.push(skb)
              sk->sk_data_ready(sk) → wake up reader
```

## 送信パスを1枚の図で {#the-transmit-path-in-one-picture}

```
write()/sendmsg()
    ↓ tcp_sendmsg() / udp_sendmsg()
    ↓ sk_buff allocation + data copy
tcp_transmit_skb() / udp_send_skb()
    ↓ IP header construction
ip_queue_xmit() → ip_output()
    ↓ Netfilter (OUTPUT hook)
    ↓ routing: dst_output() → ip_finish_output()
    ↓ fragmentation if needed
    ↓ Netfilter (POSTROUTING hook)
    ↓ neigh_output() → ARP resolution
dev_queue_xmit()
    ↓ qdisc (traffic control, rate limiting)
    ↓ driver's ndo_start_xmit()
    ↓ DMA to NIC
```

## なぜ複数の抽象化層があるのか {#why-multiple-abstraction-layers}

### 1. コピーなしの柔軟性 {#1-flexibility-without-copying}

各層は同じ`sk_buff`を操作する。TCPは`skb_push()`を呼んでTCPヘッダを付加する。これは`data`ポインタを20バイト前に戻し、そこにヘッダを書き込む操作である。IPも自分のヘッダについて同じことを行う。Ethernetも同様である。ヘッダについては**層をまたいでコピーはゼロ**である。

### 2. プロトコル非依存性 {#2-protocol-independence}

ソケット層は、下にあるトランスポートがTCPかUDPかを知らない。`proto_ops->sendmsg()`が適切な実装にディスパッチする。新しいプロトコル（SCTP、カーネル内QUICなど）を追加するには、適切なインターフェースを実装するだけでよい。

### 3. フックによる拡張性 {#3-extensibility-via-hooks}

**Netfilter**はIPスタック内の固定された地点（PREROUTING、INPUT、FORWARD、OUTPUT、POSTROUTING）にフックを提供する。ファイアウォール、NAT、コネクショントラッキング、パケット改変はすべてこれらのフックに接続され、コアスタックを変更する必要はない。

**XDP**は、最大の性能を得るために、さらに早い段階、つまり`sk_buff`の割り当て前にフックする。

**eBPFソケットプログラム**は、フロー単位のポリシーのためにソケットレベルでフックする。

### 4. 高スループットのためのNAPI {#4-napi-for-high-throughput}

割り込み駆動の受信モデルはスケールしない。NAPIは高負荷時にポーリングに切り替え、softirqコンテキストでパケットをバッチ処理する。GROはセグメントを結合する。RSSは複数のCPUに処理を分散する。これらはすべて、コアの受信パスの上に積み重ねられている。

## 主要なデータ構造の一覧 {#key-data-structures-at-a-glance}

| 構造体 | 場所 | 目的 |
|-----------|-------|---------|
| `sk_buff` | あらゆる場所 | パケットそのもの。すべての層を通過する |
| `struct socket` | net/core | VFS側: fd → ソケットの対応付け |
| `struct sock` | net/core | プロトコル側: キュー、状態、コールバック |
| `struct proto` | プロトコルごと | TCP/UDPの実装 |
| `struct net_device` | ドライバ | NICの抽象化 |
| `struct napi_struct` | ドライバ | キューごとの受信ポーリング |
| `struct dst_entry` | ルーティング | ネクストホップのルーティングキャッシュエントリ |
| `struct nf_hook_ops` | netfilter | ファイアウォール/NATフックの登録 |

## 複雑さは偶然ではなく設計によるもの {#complexity-by-design-not-accident}

スタックの複雑さは、本物の要件を反映している:

- **TCP**はパケットロス、順序の入れ替わり、輻輳、フロー制御、コネクションの寿命を扱わなければならない。本質的に複雑である
- **NAT**は双方向を書き換えるためにコネクションの状態を追跡しなければならない。conntrackサブシステムが必要になる
- RSS、RPS、XPS、IRQアフィニティを備えた**マルチキューNIC**は、あらゆる層でCPUごとの状態を必要とする
- ネットワークnamespaceを使う**コンテナ**は、ネットワークスタック全体の仮想化を必要とする
- **セキュリティ**（nftables、seccomp、SELinuxのソケットラベル）は複数の地点でのフックを必要とする

どれか1つの部分（TCP、ルーティング、Netfilter、NAPI）を理解するのは手に負える範囲である。スタック全体が複雑なのは、それが手に負える多くの部品から成り、それらが完璧に組み合わさらなければならないからである。

## 次に読むもの {#where-to-go-next}

スタックを理解する最良の方法は、パケットを追いかけることである:

- [パケットの一生（受信）](life-of-packet-rx.md) — NICからアプリケーションまで、ステップごとに
- [パケットの一生（送信）](life-of-packet-tx.md) — アプリケーションから回線まで

あるいは主要な構造体から始める:

- [sk_buff](sk-buff.md) — あらゆる場所で使われるパケット構造体
- [ソケット層の概要](socket-layer.md) — socket/sock/protoの階層
- [ネットワークデバイスとNAPI](napi.md) — 受信ハードウェアのインターフェース

## 参考文献 {#further-reading}

### このリポジトリ内 {#in-this-repo}

- [sk-buff.md](sk-buff.md) — `sk_buff`構造体の詳細: headroom、tailroom、cloneとcopyの違い、GRO
- [socket-layer.md](socket-layer.md) — `struct socket` / `struct sock` / `struct proto`の階層とVFSとの統合
- [life-of-packet-rx.md](life-of-packet-rx.md) — NICの割り込みから`recv()`が返るまでの、注釈付きの完全な受信パス
- [life-of-packet-tx.md](life-of-packet-tx.md) — `send()`からDMAディスクリプタの引き渡しまでの、注釈付きの完全な送信パス
- [napi.md](napi.md) — NAPIポーリング、GRO、マルチキューのRSS/RPS/XPSの仕組み
- [tc-qdisc.md](tc-qdisc.md) — トラフィック制御: qdisc、クラス、フィルタ、HTB/FQスケジューラ

### カーネルソース {#kernel-source}

- [`net/core/`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/core) — ソケット層、`sk_buff`の管理、`dev.c`（TX/RXの中央ディスパッチ）、NAPI
- [`net/ipv4/`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4) — IPv4の入出力、TCP、UDP、ルーティング、Netfilterのフック箇所
- [`include/linux/skbuff.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/skbuff.h) — `sk_buff`の完全な定義と、各層で使われるインラインヘルパー
- [`include/linux/netdevice.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/netdevice.h) — `struct net_device`、`struct napi_struct`、`net_device_ops`の関数テーブル

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/networking/`](https://www.kernel.org/doc/html/latest/networking/index.html) — 公式のカーネルネットワーキングドキュメント: スケーリング、packet mmap、タイムスタンプなど
- [`Documentation/networking/scaling.rst`](https://www.kernel.org/doc/html/latest/networking/scaling.html) — RSS、RPS、RFS、XPS、aRFSの解説とチューニングの指針
- [`Documentation/networking/kapi.rst`](https://www.kernel.org/doc/html/latest/networking/kapi.html) — カーネルネットワーキングAPIリファレンス（ソケットバッファ、デバイス、プロトコルのAPI）

### LWN記事 {#lwn-articles}

- [Driver porting: Network drivers (2003)](https://lwn.net/Articles/30107/) — NAPIの導入と、純粋な割り込み駆動受信からの移行
- [JLS2009: Generic receive offload](https://lwn.net/Articles/358910/) — GROがハードウェアLROと同様に、ソフトウェアでセグメントを結合する仕組み
- [BPF: the universal in-kernel virtual machine (2014)](https://lwn.net/Articles/599755/) — BPFがパケットフィルタリング言語から、ネットワーキングをはるかに超えて使われる汎用のカーネル内仮想マシンへと成長した経緯
- [Early packet drop — and more — with BPF (2016)](https://lwn.net/Articles/682538/) — XDPの設計目標と、skbより前にフックするモデル
