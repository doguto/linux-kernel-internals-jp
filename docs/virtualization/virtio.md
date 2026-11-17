# virtio {#virtio}

> KVMゲストのための準仮想化I/O

## なぜ準仮想化か {#why-paravirtualization}

QEMUの中で実際のハードウェア（例: Intel e1000 NIC、IDEディスク）をエミュレートする方式は動作はするが遅い。ゲストによるデバイスレジスタへのアクセスのたびにVM exitが発生し、QEMUがアクセスをデコードし、エミュレートされた状態を更新して戻る。これがパケット1つにつき何千回も起こる。

virtioは、エミュレートされたハードウェアレジスタを共有メモリのプロトコルに置き換える。ゲストは自分がVMの中にいることを知っていて協調する。VM exitははるかに少なく、スループットはずっと高い。

```
Emulated hardware path:
  Guest write → VM exit → QEMU decode → emulate register → VM resume
  (per I/O register access)

virtio path:
  Guest fills ring buffer → notify host (one VM exit per batch)
  Host processes batch → notifies guest
```

## virtioのアーキテクチャ {#virtio-architecture}

```
Guest kernel                          Host (QEMU / vhost)
┌────────────────────────────┐        ┌────────────────────────────┐
│  virtio-net / virtio-blk   │        │  virtio-net backend        │
│  device driver             │        │  (QEMU or vhost-net)       │
│           │                │        │           ▲                │
│    virtqueue (vring)       │        │    virtqueue (vring)       │
│    ┌──────────────────┐    │        │    ┌──────────────────┐    │
│    │ desc  table      │◄───┼────────┼───►│ desc  table      │    │
│    │ avail ring       │    │        │    │ avail ring       │    │
│    │ used  ring       │    │        │    │ used  ring       │    │
│    └──────────────────┘    │        │    └──────────────────┘    │
│           │                │        │                            │
│    kick (PCI notify)  ─────┼────────┼──► process descriptors     │
│    ◄─── interrupt ─────────┼────────┼─── add to used ring        │
└────────────────────────────┘        └────────────────────────────┘
```

## virtqueue / vring {#the-virtqueue-vring}

virtqueueは、共有メモリ上の3つの領域からなる分割型のリングバッファである。

```c
/* include/uapi/linux/virtio_ring.h */

/* 1つのディスクリプタ: バッファのセグメントを指す */
struct vring_desc {
    __virtio64 addr;   /* バッファの物理アドレス（GPA） */
    __virtio32 len;    /* バッファの長さ */
    __virtio16 flags;  /* VRING_DESC_F_NEXT | VRING_DESC_F_WRITE | VRING_DESC_F_INDIRECT */
    __virtio16 next;   /* 次のディスクリプタのインデックス（NEXTフラグがセットされている場合） */
};

/* availableリング: ドライバ（ゲスト）がここに追加し、デバイス（ホスト）が読む */
struct vring_avail {
    __virtio16 flags;  /* VRING_AVAIL_F_NO_INTERRUPT: 割り込みを抑制する */
    __virtio16 idx;    /* ドライバが次のエントリを置く位置 */
    __virtio16 ring[]; /* ディスクリプタチェーンの先頭インデックス */
};

/* usedリング: デバイス（ホスト）がここに追加し、ドライバ（ゲスト）が読む */
struct vring_used_elem {
    __virtio32 id;     /* 使用済みディスクリプタチェーンのインデックス */
    __virtio32 len;    /* 書き込まれたバイト数（読み込みの場合。書き込みでは0） */
};

struct vring_used {
    __virtio16 flags;  /* VRING_USED_F_NO_NOTIFY: キックを抑制する */
    __virtio16 idx;    /* デバイスが次のエントリを置く位置 */
    struct vring_used_elem ring[];
};
```

これは**split ring**のレイアウトであり、3つの別々の領域（ディスクリプタテーブル、availリング、usedリング）からなる、従来型/デフォルトのvirtqueueの形式である。`VIRTIO_F_RING_PACKED`をネゴシエートすると（下の機能ビットを参照）、代わりに**packed ring**のレイアウトに切り替わる。これは`struct vring_packed_desc`エントリからなる単一の統合されたリングで、ディスクリプタ、利用可能状態、完了状態を1つのキャッシュラインに優しい構造にまとめており、split ringの3領域にまたがる間接参照を避ける。

### virtqueueのライフサイクル {#virtqueue-lifecycle}

```
Guest (driver side):                   Host (device side):

1. Allocate descriptors:
   desc[0].addr = buf_gpa
   desc[0].len  = 1500
   desc[0].flags = VRING_DESC_F_WRITE

2. Post to available ring:
   avail.ring[avail.idx] = 0 (desc #0)
   wmb()
   avail.idx++

3. Kick the host:
   iowrite16(queue_idx, notify_addr)
   → VM exit (PCI MMIO write)

                                       4. Host sees avail.idx changed
                                          Read desc[0] → DMA into buf

                                       5. Host posts to used ring:
                                          used.ring[used.idx] = {0, 1500}
                                          used.idx++
                                          send interrupt to guest

6. Guest interrupt handler:
   while (last_used != used.idx) {
       process used.ring[last_used]
       last_used++
   }
```

### ディスクリプタのチェーン {#descriptor-chaining}

大きなバッファ（例: ヘッダとペイロードからなる64KBのネットワークパケット）は、チェーンとして表現できる。

```
desc[0]: header   → flags = NEXT, next = 1
desc[1]: payload  → flags = WRITE, next = 0  (end of chain)
```

ホストはdesc[0]を読み、`next`をたどってdesc[1]に進み、両方を1つの論理的なバッファとして処理する。

## virtio-net {#virtio-net}

```c
/* drivers/net/virtio_net.c */
struct virtnet_info {
    struct virtio_device    *vdev;
    struct virtqueue        *cvq;           /* 制御用virtqueue */
    struct net_device       *dev;
    struct send_queue       *sq;            /* TXキューごとに1つ */
    struct receive_queue    *rq;            /* RXキューごとに1つ */
    unsigned int            max_queue_pairs;

    /* ホストとネゴシエートしたオフロード機能 */
    bool                    mergeable_rx_bufs;
    bool                    has_rss;
};

/* TXの経路: パケットをキューに入れる（概念的なスケッチ — drivers/net/virtio_net.cの
 * 実際のxmit_skb()は、ヘッダのバリアント一式、トンネリング、ヘッダpushの
 * 最適化を扱うように拡張されている。下を参照） */
static int xmit_skb(struct send_queue *sq, struct sk_buff *skb)
{
    struct virtio_net_hdr *hdr;
    struct virtnet_info *vi = sq->vq->vdev->priv;
    int num_sg;

    /* virtio_netヘッダを先頭に付ける */
    hdr = skb_push(skb, vi->hdr_len);
    memset(hdr, 0, vi->hdr_len);

    /* skb自身の状態からチェックサム/GSOオフロードのフィールドを埋める */
    virtio_net_hdr_from_skb(skb, hdr, virtio_is_little_endian(vi->vdev), false, 0);

    /* skbのフラグメントからスキャッタギャザーリストを構築する */
    num_sg = skb_to_sgvec(skb, sq->sg, 0, skb->len);

    return virtqueue_add_outbuf(sq->vq, sq->sg, num_sg, skb, GFP_ATOMIC);
}
```

実際の`xmit_skb()`（現在は`orphan`フラグも受け取る）は、上に示したチェックサム/GSOのロジックをインラインで手書きしているわけではない。それは`virtio_net_hdr_from_skb()`ヘルパー（`include/linux/virtio_net.h`）に切り出されており、実際に`skb->ip_summed == CHECKSUM_PARTIAL`のときに`VIRTIO_NET_HDR_F_NEEDS_CSUM`/`csum_start`/`csum_offset`を設定し、skbがGSOパケットのときにGSOのフィールドを埋めるのはこのヘルパーである。実際の関数はまた、ネゴシエートされた機能に応じて複数のヘッダのバリアント、つまり素の`struct virtio_net_hdr`、下のmergeable-rx-buffersのバリアント、あるいは（新しいハッシュ/トンネルのオフロード向けの）`struct virtio_net_hdr_v1_hash_tunnel`から選択し、`virtqueue_add_outbuf()`を直接呼ぶのではなく`virtnet_add_outbuf()`ラッパーを通じてキューに入れる。上のスケッチは、ヘッダのバリアントのディスパッチとTXのバッチ処理のロジックを省き、中核となる考え方、すなわちヘッダを先頭に付け、skb自身のチェックサム/GSOの状態でそれを決め、バッファをリングに渡すという点だけを残している。

### virtio-netヘッダ {#virtio-net-header}

```c
/* include/uapi/linux/virtio_net.h */
struct virtio_net_hdr {
    __u8       flags;       /* VIRTIO_NET_HDR_F_NEEDS_CSUMなど */
    __u8       gso_type;    /* VIRTIO_NET_HDR_GSO_TCPV4/6など */
    __virtio16 hdr_len;     /* イーサネット+IP+TCPのヘッダ長 */
    __virtio16 gso_size;    /* GSOの最大セグメントサイズ */
    __virtio16 csum_start;  /* チェックサムの開始位置へのオフセット */
    __virtio16 csum_offset; /* チェックサムを置く、csum_startからのオフセット */
};

/* VIRTIO_NET_F_MRG_RXBUF（下記）をネゴシエートすると、その上に1つのフィールドが加わる —
 * 受信側のバッファのマージにおいて、現在も使われている最新のワイヤーフォーマットである: */
struct virtio_net_hdr_mrg_rxbuf {
    struct virtio_net_hdr hdr;
    __virtio16 num_buffers; /* このパケットがまたがるマージされたRXバッファの数 */
};
```

これらの16ビットのフィールドは`__virtio16`であり、エンディアンが固定された型ではない。virtioのワイヤー上のバイトオーダーは、どのVIRTIO_F_VERSION_1モードがネゴシエートされたかに依存する（モダンなデバイスは常にリトルエンディアン、レガシーなデバイスはゲストのネイティブなオーダーを使う）ため、ヘッダ自体は型のレベルで`__le16`に決め打ちできない。このヘッダにより、ホストとゲストはハードウェアオフロード、すなわちチェックサム、セグメンテーション（TSO）、受信側の結合（LRO）をネゴシエートできる。

## virtio-blk {#virtio-blk}

```c
/* include/uapi/linux/virtio_blk.h */
struct virtio_blk_outhdr {
    __virtio32 type;    /* VIRTIO_BLK_T_IN / T_OUT / T_FLUSH / T_DISCARD */
    __virtio32 ioprio;
    __virtio64 sector;  /* 512バイトセクタの番号 */
};

/* 1つの完全なI/Oリクエストのディスクリプタチェーン:
   [0] virtio_blk_outhdrヘッダ  （デバイスが読み取り可能）
   [1] データバッファ           （readではデバイスが書き込み可能、writeではデバイスが読み取り可能）
   [2] ステータスバイト         （デバイスが書き込み可能: VIRTIO_BLK_S_OK=0、S_IOERR=1、S_UNSUPP=2） */
```

### なぜvirtio-blkは速いか {#why-virtio-blk-is-fast}

- vCPUごとに1つのvirtqueue（マルチキューのサポート）
- ゲストはキックする前に複数のリクエストをバッチにまとめる
- ホストはバッチ全体を処理し、すべての完了を通知する
- `VIRTIO_BLK_F_FLUSH`により、明示的なフラッシュの順序付けが可能

## 機能のネゴシエーション {#feature-negotiation}

virtioデバイスは、使用前に機能をネゴシエートする。

```c
/* ゲストはホストがサポートする機能を読む */
u64 host_features = vdev->config->get_features(vdev);

/* ゲストは何を使いたいかを決める */
u64 guest_features = 0;
if (host_features & (1ULL << VIRTIO_NET_F_CSUM))
    guest_features |= (1ULL << VIRTIO_NET_F_CSUM);   /* チェックサムオフロード */
if (host_features & (1ULL << VIRTIO_NET_F_MRG_RXBUF))
    guest_features |= (1ULL << VIRTIO_NET_F_MRG_RXBUF); /* マージ可能なRX */
if (host_features & (1ULL << VIRTIO_F_RING_PACKED))
    guest_features |= (1ULL << VIRTIO_F_RING_PACKED); /* packed ringのレイアウト */

/* 受け入れた機能を書き戻す */
vdev->config->finalize_features(vdev);
```

主要な機能ビット:
```c
VIRTIO_F_VERSION_1         /* モダンなvirtio（新しいデバイスでは必須） */
VIRTIO_F_RING_PACKED       /* packed ringのレイアウト（avail/usedの分割を避ける） */
VIRTIO_F_IN_ORDER          /* ホストはリクエストを順番どおりに完了する */
VIRTIO_NET_F_CSUM          /* チェックサムオフロード */
VIRTIO_NET_F_HOST_TSO4     /* TCPセグメンテーションオフロード（IPv4） */
VIRTIO_NET_F_MRG_RXBUF     /* マージ可能な受信バッファ */
VIRTIO_NET_F_RSS           /* receive-side scaling（マルチキュー） */
VIRTIO_BLK_F_SEG_MAX       /* リクエストあたりの最大セグメント数 */
VIRTIO_BLK_F_SIZE_MAX      /* 最大セグメントサイズ */
VIRTIO_BLK_F_DISCARD       /* TRIM/discardのサポート */
VIRTIO_BLK_F_WRITE_ZEROES  /* write-zeroesコマンド */
```

## vhost-net: カーネル空間による高速化 {#vhost-net-kernel-space-acceleration}

vhost-netは、virtio-netのバックエンドをQEMUのユーザー空間からカーネルへ移し、パケットごとのQEMU → ホストカーネルの往復をなくす。

```
Without vhost-net:
  Guest → VM exit → KVM → wake QEMU → QEMU reads vring →
  QEMU writes to TAP fd → kernel TAP → network

With vhost-net:
  Guest → VM exit → KVM → vhost worker (kernel thread) →
  reads vring → writes to TAP directly (no QEMU in data path)
```

### vhostのカーネル内部 {#vhost-kernel-internals}

```c
/* drivers/vhost/net.c */
struct vhost_net {
    struct vhost_dev dev;           /* 汎用のvhostコア（ワーカー、iotlb、...） */
    struct vhost_net_virtqueue vqs[VHOST_NET_VQ_MAX];
    /* ... */
};

struct vhost_dev {
    struct mm_struct *mm;           /* GPA→HVA変換のためのゲストのmm */
    struct mutex mutex;
    struct vhost_virtqueue **vqs;
    int nvqs;
    struct xarray worker_xa;        /* このデバイスのワーカーの登録簿 */
    bool use_worker;
    /* ... */
};

/* 説明用のみ — 実在する関数ではない。vhostワーカーの一般的な
 * パターン: ゲストのavailリングからディスクリプタを読み、I/Oを行い、
 * 完了をusedリングに通知し、ゲストにシグナルを送る。
 * drivers/vhost/net.cでは、これはhandle_tx()（薄いディスパッチャ）、
 * get_tx_bufs()、handle_tx_copy()/handle_tx_zerocopy()に分散しており、
 * このスケッチが省いているTXのバッチ処理、XDP、ビジーポーリングが加わる: */
static void vhost_worker_tx_sketch(struct vhost_net *net, struct vhost_virtqueue *vq)
{
    unsigned int head;

    while ((head = vhost_get_vq_desc(vq, ...)) != vq->num) {
        /* ゲストのmmを使ったGPA → HVA変換 */
        /* ソケット / TAPのfdに書き込む */
        vhost_add_used(vq, head, 0);
    }
    vhost_signal(&net->dev, vq);  /* ゲストに割り込む */
}
```

`struct vhost_dev`はかつて単一の`struct task_struct *worker`を直接所有していた。その後リファクタリングされ、デバイスが複数のワーカー（`worker_xa`。virtqueueごとのワーカー割り当てのようなケースのためにキーが付けられる）を登録できるようになり、ワーカーは素のkthreadではなく`vhost_task_create()`で作成されるようになった。ワーカーの`comm`は依然として`vhost-<owner-pid>`なので、（後述の）`pgrep vhost`で今でも見つけられる。

### vhost-user: ユーザー空間のvhost {#vhost-user-userspace-vhost}

vhost-userは、制御にUnixソケット、リングに共有メモリを使い、バックエンドを別のユーザー空間プロセス（例: DPDK、OVS）に移す。

```
Guest (KVM) ──► vring (shared memory) ◄── vhost-user process (DPDK/OVS)
                 ▲
          eventfd for kick/interrupt
```

Open vSwitch + DPDKで、カーネルを介さずにラインレートのスイッチングを行うために使われる。

## virtio-mmioとPCIトランスポート {#virtio-mmio-vs-pci-transport}

virtioデバイスは2種類のトランスポートで公開される。

| トランスポート | 用途 | 検出方法 |
|-----------|-------|-----------|
| PCI | x86のVM、標準 | PCIコンフィギュレーション空間のスキャン |
| MMIO | ARM/組み込みのVM、コンテナ | デバイスツリー / ACPI |

```bash
# Linuxゲスト内で: virtioデバイスを確認する
lspci | grep -i virtio
# 00:01.0 Ethernet controller: Red Hat, Inc. Virtio network device
# 00:02.0 SCSI storage controller: Red Hat, Inc. Virtio block device

# またはvirtio-mmio経由で:
cat /proc/device-tree/virtio_mmio@*/compatible
# virtio,mmio
```

## virtioの性能の観測 {#observing-virtio-performance}

```bash
# ゲスト内で: virtio-netの統計
ethtool -S eth0 | grep -E "queue|vq"

# virtio-blkデバイスの統計
cat /sys/block/vda/stat

# vhost-netの統計（ホスト上で）
cat /proc/net/dev   # TAPインターフェイスのスループット

# キック/割り込みの頻度（低い = バッチ処理、高い = レイテンシ重視）:
# virtio/vringにはこのためのdebugfsやtracepointのインターフェイスがないので、
# 代わりに実際のキック/割り込みのエントリポイントの呼び出しを直接数える:
perf probe -a virtqueue_notify
perf probe -a vring_interrupt
perf stat -e probe:virtqueue_notify,probe:vring_interrupt -a sleep 5

# perf: vhostワーカーのCPU使用率
perf top -p $(pgrep vhost)

# 割り込みの結合: 割り込みが抑制されているか確認する
# （VRING_AVAIL_F_NO_INTERRUPTはホストにゲストへ割り込まないよう伝える）
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [drivers/virtio/](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/virtio) — virtioのコアバス: デバイスの登録と機能のネゴシエーション（`virtio_add_status()`）
- [drivers/virtio/virtio_ring.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/virtio/virtio_ring.c) — `virtqueue_add_outbuf()`と`virtqueue_kick()`: 上のvringの図の背後にあるsplit ringとpacked ringの実装
- [include/uapi/linux/virtio_ring.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/virtio_ring.h) — `struct vring_desc`、`vring_avail`、`vring_used`: ワイヤー上のディスクリプタ/available/usedリングのレイアウト
- [drivers/net/virtio_net.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/net/virtio_net.c) — virtio-netドライバ: `struct virtnet_info`、`xmit_skb()`
- [include/linux/virtio_net.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/virtio_net.h) — `virtio_net_hdr_from_skb()`: `xmit_skb()`がヘッダに書き込むチェックサム/GSOオフロードのフィールドを埋める
- [include/uapi/linux/virtio_net.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/virtio_net.h) — `struct virtio_net_hdr`、`struct virtio_net_hdr_mrg_rxbuf`、`VIRTIO_NET_F_*`機能ビット
- [drivers/block/virtio_blk.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/block/virtio_blk.c) — virtio-blkドライバ
- [include/uapi/linux/virtio_blk.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/virtio_blk.h) — `struct virtio_blk_outhdr`と`VIRTIO_BLK_F_*`機能ビット（`DISCARD`、`WRITE_ZEROES`、`FLUSH`）
- [drivers/vhost/net.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/vhost/net.c) — vhost-netのカーネルバックエンド: `struct vhost_net`、`handle_tx_copy()`/`handle_tx_zerocopy()`にディスパッチする`handle_tx()`
- [drivers/vhost/vhost.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/vhost/vhost.h) — `struct vhost_dev`: vhost-net、vhost-scsi、vhost-vsockが共有する汎用のvhostコア

### 関連ページ {#related-pages}

- [KVMアーキテクチャ](kvm-arch.md) — VM exit、ハイパーコール
- [メモリ仮想化](kvm-memory.md) — EPT、バルーンドライバ

### LWNの記事 {#lwn-articles}

- [An API for virtual I/O: virtio](https://lwn.net/Articles/239238/) — Rusty Russellによるvirtioの導入を扱った2007年のLWNの記事: virtqueueとなった`add_buf()`/`sync()`/`get_buf()`の操作ベクタと、ブロックドライバおよびネットワークドライバの実例
- [Standardizing virtio](https://lwn.net/Articles/580186/) — Jonathan Corbet、2014年: virtioがOASISでの標準化に移った理由と、1.0仕様で何が変わったか（必須のバージョン機能ビット、リトルエンディアンに固定されたバイトオーダー、柔軟なvirtqueueのメモリレイアウト）

### 外部資料 {#external}

- [Virtual I/O Device (VIRTIO) Version 1.2](https://docs.oasis-open.org/virtio/virtio/v1.2/virtio-v1.2.html) — OASISの委員会仕様: splitとpackedのvirtqueue形式、デバイスタイプ（net、blockなど）、PCI/MMIO/CCWトランスポート
- [Virtio on Linux](https://docs.kernel.org/driver-api/virtio/virtio.html) — virtioサブシステムのカーネルドキュメント: コアバス、virtqueue、トランスポートドライバ
- [Writing Virtio Drivers](https://docs.kernel.org/driver-api/virtio/writing_virtio_drivers.html) — `virtqueue_add_outbuf()`、`virtqueue_add_inbuf()`、`virtqueue_kick()`、ドライバのprobe/removeのライフサイクルを扱うカーネルドキュメント
