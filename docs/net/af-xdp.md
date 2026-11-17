# AF_XDPソケット {#af_xdp-sockets}

> ユーザー空間との間でパケットをゼロコピーで直接送受信する

## AF_XDPとは何か {#what-af_xdp-is}

AF_XDPはソケットのアドレスファミリの一つで、ユーザー空間のアプリケーションがXDPプログラムから直接パケットを受け取れるようにする。対象のパケットについては、カーネルのネットワークスタックを完全にバイパスする。XDPで動作するBPFプログラムが、マッチしたパケットをAF_XDPソケットにリダイレクトし、アプリケーションは共有メモリ上のリングバッファからそれを読み取る。

これにより、カーネルのセキュリティモデルの範囲内にとどまりながら（カーネルバイパスではなく、ハードウェアは引き続きカーネルが所有する）、DPDKに近い性能を実現する。

## アーキテクチャ {#the-architecture}

```
NIC hardware → DMA → ring buffer
    ↓ (NAPI poll)
XDP program runs:
    if packet matches: bpf_redirect_map(&xsk_map, queue_id, 0) → XDP_REDIRECT
    else: XDP_PASS → normal kernel stack
    ↓
Shared UMEM (User Memory)
    ↓
AF_XDP socket (xsk) → userspace application reads/writes directly
```

`sk_buff`は割り当てられない。パケットのDMAバッファは、ユーザー空間が読み取るのと同じメモリである。

## UMEM: 共有メモリ領域 {#umem-the-shared-memory-region}

AF_XDPは**UMEM**を中心に構築されている。UMEMはユーザー空間が割り当てるメモリ領域で、固定サイズのフレームに分割され、カーネルとユーザー空間の双方で共有される:

```
UMEM:
┌──────┬──────┬──────┬──────┬──────┬──────┐
│frame0│frame1│frame2│frame3│frame4│frame5│ ...
└──────┴──────┴──────┴──────┴──────┴──────┘
Each frame: configurable size (default 4096 bytes)
```

4つのリングバッファが、カーネルとユーザー空間の間でフレームの所有権を仲介する:

| リング | 方向 | 生産者 | 消費者 |
|------|-----------|--------------|--------------|
| FILL | RX: フレームをカーネルに渡す | ユーザー空間 | カーネル（DMAの書き込み先） |
| RX | RX: カーネルがパケットを届ける | カーネル | ユーザー空間 |
| TX | TX: ユーザー空間がパケットを送る | ユーザー空間 | カーネル |
| COMPLETION | TX: カーネルが送信完了を通知する | カーネル | ユーザー空間 |

## AF_XDPソケットのセットアップ {#setting-up-an-af_xdp-socket}

```c
// 1. UMEM（共有メモリ）を作成
int umem_size = 4096 * NUM_FRAMES;  // 各4096バイトのフレームをNUM_FRAMES個
void *umem_area = mmap(NULL, umem_size, PROT_READ|PROT_WRITE,
                       MAP_PRIVATE|MAP_ANONYMOUS|MAP_HUGETLB, -1, 0);

struct xdp_umem_reg mr = {
    .addr = (uint64_t)umem_area,
    .len  = umem_size,
    .chunk_size = 4096,
    .headroom   = 0,
};

// 2. AF_XDPソケットを作成
int xsk_fd = socket(AF_XDP, SOCK_RAW, 0);

// 3. UMEMをソケットに登録
setsockopt(xsk_fd, SOL_XDP, XDP_UMEM_REG, &mr, sizeof(mr));

// 4. リングサイズを設定
int ring_size = 2048;
setsockopt(xsk_fd, SOL_XDP, XDP_RX_RING,   &ring_size, sizeof(ring_size));
setsockopt(xsk_fd, SOL_XDP, XDP_TX_RING,   &ring_size, sizeof(ring_size));
setsockopt(xsk_fd, SOL_XDP, XDP_UMEM_FILL_RING,      &ring_size, sizeof(ring_size));
setsockopt(xsk_fd, SOL_XDP, XDP_UMEM_COMPLETION_RING,&ring_size, sizeof(ring_size));

// 5. リングをユーザー空間にマップ
struct xdp_mmap_offsets off;
getsockopt(xsk_fd, SOL_XDP, XDP_MMAP_OFFSETS, &off, &optlen);

void *fill_ring = mmap(NULL, off.fr.desc + ring_size * sizeof(__u64),
                       PROT_READ|PROT_WRITE, MAP_SHARED|MAP_POPULATE,
                       xsk_fd, XDP_UMEM_PGOFF_FILL_RING);
// rx_ring、tx_ring、completion_ringも同様

// 6. インターフェースとキューにバインド
struct sockaddr_xdp sxdp = {
    .sxdp_family    = AF_XDP,
    .sxdp_ifindex   = ifindex,
    .sxdp_queue_id  = queue_id,
    .sxdp_flags     = XDP_COPY,  // 最適な性能にはXDP_ZEROCOPY
};
bind(xsk_fd, (struct sockaddr *)&sxdp, sizeof(sxdp));
```

## ゼロコピーモード {#zero-copy-mode}

**ゼロコピー**モード（`XDP_ZEROCOPY`）では、NICがパケットをUMEMのフレームに直接DMA書き込みする。NIC、カーネル、ユーザー空間の間でデータのコピーは一切発生しない。必要条件:

- ドライバのサポート（i40e、mlx5、ixgbeなど）
- UMEMのフレームがメモリ上にピン留めされていること

**コピーモード**（`XDP_COPY`）では、カーネルが内部バッファからUMEMへパケットデータをコピーする。どのNICでも動作するが、コピーが1回加わる。

## パケットの受信（RXループ） {#receiving-packets-rx-loop}

```c
// 空きフレームのアドレスでfillリングを埋める（DMA用にカーネルへ渡す）
for (int i = 0; i < NUM_FRAMES; i++) {
    fill_ring->addrs[fill_prod_idx % ring_size] = i * FRAME_SIZE;
    fill_prod_idx++;
}
// fillリングのプロデューサインデックスを更新
__atomic_store_n(fill_ring->producer, fill_prod_idx, __ATOMIC_RELEASE);

// 受信パケットをpollで待つ
struct pollfd pfd = { .fd = xsk_fd, .events = POLLIN };
poll(&pfd, 1, -1);

// RXリングから読み取る
uint32_t rx_idx = *rx_ring->consumer;
uint32_t avail  = *rx_ring->producer - rx_idx;

for (uint32_t i = 0; i < avail; i++) {
    struct xdp_desc *desc = &rx_ring->descs[(rx_idx + i) % ring_size];
    void *pkt = umem_area + desc->addr;
    uint32_t len = desc->len;
    // pkt[0..len-1]のパケットを処理する
    // その後、フレームをfillリングに戻す
}
*rx_ring->consumer = rx_idx + avail;
```

## XDPプログラム（カーネル側） {#the-xdp-program-kernel-side}

```c
struct {
    __uint(type, BPF_MAP_TYPE_XSKMAP);
    __uint(max_entries, 64);
    __type(key, __u32);    // キューのインデックス
    __type(value, __u32);  // xskソケットのfd
} xsk_map SEC(".maps");

SEC("xdp")
int xdp_redirect_to_xsk(struct xdp_md *ctx)
{
    // このキューのすべてのパケットをAF_XDPソケットにリダイレクト
    return bpf_redirect_map(&xsk_map, ctx->rx_queue_index, XDP_PASS);
    // フォールバックとしてのXDP_PASS: このキューにxskが登録されていなければスタックへ渡す
}
```

```bash
# xskソケットをXSKMAPに登録
bpftool map update id <xskmap_id> key 0 value <xsk_fd>
```

## libxdp / libbpfヘルパーライブラリ {#libxdp-libbpf-helper-library}

本番用途では、`libxdp`（xdp-toolsの一部）と`libbpf`が、リング管理、UMEMのセットアップ、XDPプログラムのロードを扱う抽象化を提供する:

```c
// libxdpを使う場合
#include <xdp/xsk.h>

struct xsk_socket_config cfg = {
    .rx_size     = 2048,
    .tx_size     = 2048,
    .libxdp_flags = XSK_LIBXDP_FLAGS__INHIBIT_PROG_LOAD,
    .xdp_flags   = XDP_FLAGS_UPDATE_IF_NOEXIST,
    .bind_flags  = XDP_ZEROCOPY,
};

struct xsk_umem *umem;
struct xsk_socket *xsk;

xsk_umem__create(&umem, umem_area, umem_size, fill_ring, comp_ring, NULL);
xsk_socket__create(&xsk, ifname, queue_id, umem, rx_ring, tx_ring, &cfg);
```

## ユースケース {#use-cases}

- **アプリケーションプロトコルのためのカーネルバイパス**: DNS/HTTPのパケット解析をユーザー空間でラインレートで処理する
- **パケットキャプチャ**: 高レートのキャプチャにおける、libpcapのゼロコピーな代替
- **独自のロードバランサ**: ユーザー空間の判断ロジックと、カーネル並みの速度のI/O
- **ネットワーク機能仮想化**: 完全なDPDKなしでの仮想スイッチやファイアウォール

## 参考文献 {#further-reading}

- [XDP](xdp.md) — AF_XDPソケットにパケットを供給するXDPフレームワーク
- [ネットワークデバイスとNAPI](napi.md) — XDPプログラムがドライバのどこにフックするか
- [TCとqdisc](tc-qdisc.md) — スタック通過後のパケット振り分けのための代替手段
