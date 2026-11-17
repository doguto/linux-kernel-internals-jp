# XDP: eXpress Data Path {#xdp-express-data-path}

> ドライバレベルでのカーネルバイパス: 受信の高速パス上で動くBPFプログラム

## XDPとは何か {#what-is-xdp}

XDP（eXpress Data Path）は、カーネルネットワークスタックの中で可能な限り最も早い地点、つまりNICドライバの内部で、skbの割り当て**より前に**BPFプログラムを実行する。これにより、カーネル内にとどまりながらDPDKに近い速度を実現する:

```
Packet arrival:

Without XDP:
  NIC → DMA → driver alloc skb → kernel stack → socket → userspace
  Latency: ~1-5µs, throughput: limited by skb overhead

With XDP:
  NIC → DMA → XDP BPF program → (pass/drop/tx/redirect)
  Latency: ~100-200ns, throughput: 10-50+ Mpps (million packets/sec)

DPDK (comparison):
  NIC → userspace poll → DPDK app
  Latency: ~50-100ns, throughput: 60+ Mpps
  But: requires dedicated CPUs, no kernel network stack
```

## XDPのフックポイント {#xdp-hook-points}

```c
/* 3つのアタッチモード: */

/* 1. ネイティブ XDP（最速）: ドライバの RX パスで実行される */
/*    対応ドライバ: mlx5, i40e, ixgbe, bpfilter, veth など */
ip link set eth0 xdp obj xdp_prog.o sec xdp

/* 2. 汎用 XDP（skb ベース）: どのドライバでも動くが、遅い */
/*    skb の割り当て後、netif_receive_skb() の中で実行される */
ip link set eth0 xdpgeneric obj xdp_prog.o sec xdp

/* 3. HW オフロード: NIC 自体の上で実行される（対応 NIC はごく少数） */
ip link set eth0 xdpoffload obj xdp_prog.o sec xdp
```

## XDPの判定 {#xdp-verdicts}

```c
/* BPF プログラムは次のいずれかを返す: */

XDP_DROP      /* パケットを即座に破棄（skb の割り当てなし、通知なし） */
XDP_PASS      /* 通常のカーネル処理を続行（skb を割り当て、スタックを上へ渡す） */
XDP_TX        /* 同じインターフェースから再送信（例: 反射用） */
XDP_REDIRECT  /* 別のインターフェース、CPU、または AF_XDP ソケットへ転送 */
XDP_ABORTED   /* 破棄 + デバッグ用のトレースポイント（DROP と同様に扱われる） */
```

## XDPプログラムの構造 {#xdp-program-structure}

```c
/* xdp_prog.c */
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>
#include <linux/if_ether.h>
#include <linux/ip.h>

/* BPF マップ: 送信元 IP ごとのパケット数を記録 */
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __type(key, __u32);    /* 送信元 IP */
    __type(value, __u64);  /* パケット数 */
    __uint(max_entries, 65536);
} ip_counter SEC(".maps");

/* ブロックリストのマップ */
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __type(key, __u32);    /* ブロックする IP */
    __type(value, __u8);
    __uint(max_entries, 1024);
} blocklist SEC(".maps");

SEC("xdp")
int xdp_firewall(struct xdp_md *ctx)
{
    void *data     = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    /* Ethernet ヘッダを解析 */
    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return XDP_DROP;   /* 切り詰められたパケット */

    if (eth->h_proto != bpf_htons(ETH_P_IP))
        return XDP_PASS;   /* IPv4 ではないのでスタックに渡す */

    /* IP ヘッダを解析 */
    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end)
        return XDP_DROP;

    __u32 src_ip = ip->saddr;

    /* ブロックリストを確認 */
    if (bpf_map_lookup_elem(&blocklist, &src_ip))
        return XDP_DROP;   /* ブロック対象: 黙って破棄 */

    /* 送信元 IP ごとにパケットを数える */
    __u64 *count = bpf_map_lookup_elem(&ip_counter, &src_ip);
    if (count) {
        __sync_fetch_and_add(count, 1);
    } else {
        __u64 one = 1;
        bpf_map_update_elem(&ip_counter, &src_ip, &one, BPF_ANY);
    }

    return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
```

```bash
# コンパイル:
clang -O2 -target bpf -c xdp_prog.c -o xdp_prog.o

# アタッチ:
ip link set eth0 xdp obj xdp_prog.o sec xdp

# 確認:
ip link show eth0
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdp ...
#     xdp id 42    ← プログラムがロードされている

# デタッチ:
ip link set eth0 xdp off
```

## パケットの書き換え（NAT） {#packet-modification-nat}

```c
SEC("xdp")
int xdp_dnat(struct xdp_md *ctx)
{
    void *data     = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return XDP_PASS;
    if (eth->h_proto != bpf_htons(ETH_P_IP))
        return XDP_PASS;

    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end)
        return XDP_PASS;

    /* 宛先 IP を書き換える（チェックサムの差分計算のため、先に旧値を保存） */
    __u32 old_dst = ip->daddr;
    __u32 new_dst = bpf_htonl(0xc0a80101);  /* 192.168.1.1 */
    ip->daddr = new_dst;

    /* IP チェックサムをインラインで再計算する（bpf_l3_csum_replace/bpf_l4_csum_replace
     * と bpf_csum_diff は TC 専用のヘルパーで XDP プログラムでは使えないため、
     * RFC 1624 による差分チェックサム更新を使う）: */
    __u32 csum_diff = (~old_dst & 0xffff) + (~old_dst >> 16) +
                      (new_dst & 0xffff) + (new_dst >> 16);
    __u16 csum = (~ntohs(ip->check) & 0xffff) + (csum_diff & 0xffff) +
                  (csum_diff >> 16);
    ip->check = htons(~(csum + (csum >> 16)));

    return XDP_PASS;  /* または XDP_TX で送り返す */
}
```

## XDPのリダイレクト {#xdp-redirect}

### インターフェース間 {#between-interfaces}

```c
/* パケットを eth1 にリダイレクト: */
SEC("xdp")
int xdp_redirect_iface(struct xdp_md *ctx)
{
    int ifindex = 3;  /* eth1 の ifindex */
    return bpf_redirect(ifindex, 0);
}
```

### DEVMAP: バッチリダイレクト {#devmap-batch-redirect}

```c
/* devmap による高性能なリダイレクト */
struct {
    __uint(type, BPF_MAP_TYPE_DEVMAP);
    __uint(key_size, sizeof(int));
    __uint(value_size, sizeof(int));
    __uint(max_entries, 256);
} tx_ports SEC(".maps");

SEC("xdp")
int xdp_devmap_redirect(struct xdp_md *ctx)
{
    int port = 0;  /* devmap のキー */
    return bpf_redirect_map(&tx_ports, port, XDP_DROP);
}
```

```bash
# devmap を埋める（ユーザー空間）:
int map_fd = bpf_obj_get("/sys/fs/bpf/tx_ports");
int ifindex = if_nametoindex("eth1");
bpf_map_update_elem(map_fd, &key, &ifindex, BPF_ANY);
```

### CPUMAP: 特定のCPUへの振り分け {#cpumap-steer-to-specific-cpu}

```c
/* さらなる処理のために CPU へリダイレクトする（XDP の後） */
struct {
    __uint(type, BPF_MAP_TYPE_CPUMAP);
    __uint(key_size, sizeof(__u32));
    __uint(value_size, sizeof(struct bpf_cpumap_val));
    __uint(max_entries, 12);
} cpumap SEC(".maps");

SEC("xdp")
int xdp_cpumap_redirect(struct xdp_md *ctx)
{
    /* RSS ベースの振り分け: CPU へハッシュする */
    __u32 cpu = bpf_get_smp_processor_id() % 4;
    return bpf_redirect_map(&cpumap, cpu, 0);
}
```

## AF_XDP: ユーザー空間へのゼロコピー {#af_xdp-zero-copy-to-userspace}

AF_XDP（eXpress Data Pathソケット）は、カーネルによるコピーを一切行わずに、パケットを直接ユーザー空間のメモリへ移す:

```
NIC DMA → UMEM (userspace memory) → XSK socket → userspace app
          ↑ zero copy! kernel never touches packet data
```

### セットアップ {#setup}

```c
/* ユーザー空間: AF_XDP ソケットを作成 */
#include <linux/if_xdp.h>
#include <sys/socket.h>

/* 1. UMEM（パケットバッファプール）を割り当てる: */
void *umem_area = mmap(NULL, UMEM_SIZE, PROT_READ|PROT_WRITE,
                        MAP_PRIVATE|MAP_ANONYMOUS|MAP_HUGETLB, -1, 0);

struct xdp_umem_reg umem_reg = {
    .addr = (uint64_t)umem_area,
    .len  = UMEM_SIZE,
    .chunk_size = FRAME_SIZE,    /* フレームあたり 4096 バイト */
    .headroom   = 0,
};
setsockopt(xsk_fd, SOL_XDP, XDP_UMEM_REG, &umem_reg, sizeof(umem_reg));

/* 2. リング（FILL/COMPLETION/RX/TX）を作成: */
int ring_size = 2048;
setsockopt(xsk_fd, SOL_XDP, XDP_RX_RING, &ring_size, sizeof(ring_size));
setsockopt(xsk_fd, SOL_XDP, XDP_UMEM_FILL_RING, &ring_size, sizeof(ring_size));

/* 3. リングを mmap する: */
struct xdp_mmap_offsets off;
getsockopt(xsk_fd, SOL_XDP, XDP_MMAP_OFFSETS, &off, &optlen);

void *fill_ring = mmap(NULL, off.fr.desc + ring_size * sizeof(uint64_t),
                        PROT_READ|PROT_WRITE, MAP_SHARED|MAP_POPULATE,
                        xsk_fd, XDP_UMEM_PGOFF_FILL_RING);

/* 4. インターフェースのキューに bind する: */
struct sockaddr_xdp sxdp = {
    .sxdp_family   = AF_XDP,
    .sxdp_ifindex  = if_nametoindex("eth0"),
    .sxdp_queue_id = 0,           /* NIC のキュー 0 */
    .sxdp_flags    = XDP_ZEROCOPY, /* ネイティブ XDP のドライバサポートが必要 */
};
bind(xsk_fd, (struct sockaddr *)&sxdp, sizeof(sxdp));
```

### AF_XDPへリダイレクトするXDPプログラム {#xdp-program-for-af_xdp-redirect}

```c
/* AF_XDP ソケットの BPF マップ */
struct {
    __uint(type, BPF_MAP_TYPE_XSKMAP);
    __uint(key_size, sizeof(int));
    __uint(value_size, sizeof(int));
    __uint(max_entries, 64);
} xsks_map SEC(".maps");

SEC("xdp")
int xdp_sock_prog(struct xdp_md *ctx)
{
    int index = ctx->rx_queue_index;

    /* 同じキューの AF_XDP ソケットへリダイレクト */
    if (bpf_map_lookup_elem(&xsks_map, &index))
        return bpf_redirect_map(&xsks_map, index, 0);

    return XDP_PASS;
}
```

## XDPと代替手段の比較 {#xdp-vs-alternatives}

| | XDP | DPDK | カーネルTC | iptables |
|---|-----|------|-----------|----------|
| スループット | 50+ Mpps | 60+ Mpps | 10+ Mpps | 1-5 Mpps |
| レイテンシ | ~200ns | ~100ns | ~1µs | ~10µs |
| カーネルスタックへのアクセス | あり（XDP_PASS） | なし | あり | あり |
| CPUの専有 | なし | あり | なし | なし |
| プログラマビリティ | BPF | C | BPF/C | 限定的 |
| ユースケース | DDoS緩和、LB、ファイアウォール | 高頻度取引、NFV | QoS、TCフィルタ | 汎用ファイアウォール |

## 可観測性 {#observability}

```bash
# XDP プログラムの情報:
bpftool net show dev eth0
# xdp:
#         id 42  name xdp_firewall  flags 0x0

# XDP の統計（ドライバ経由）:
ethtool -S eth0 | grep xdp
# rx_xdp_drop: 1234567
# rx_xdp_pass: 9876543
# rx_xdp_tx:          0
# rx_xdp_redirect:    0

# BPF マップの検査:
bpftool map dump id 5   # ブロックリストのマップをダンプ

# XDP プログラムのトレーシング:
bpftrace -e '
tracepoint:xdp:xdp_exception
{ printf("XDP exception: ifindex=%d, prog_id=%d, act=%d\n",
         args->ifindex, args->prog_id, args->act); }'

# XDP の破棄に対する perf イベント:
perf record -e xdp:xdp_drop -ag -- sleep 10
perf script
```

## 参考文献 {#further-reading}

- [BPFネットワーキング](../bpf/bpf-networking.md) — TC BPF、sockmap
- [sk_buff: ネットワークバッファ](sk-buff.md) — XDP_PASSがsk_buffを作成する
- [NVMeドライバ](../block/nvme.md) — 同様のDMA/リングバッファのパターン
- [IRQアフィニティ](../interrupts/irq-affinity.md) — XDPのためのNICキューとCPUのアフィニティ
- `net/core/filter.c` — BPFネットワークフィルタの基盤
- `drivers/net/ethernet/intel/i40e/i40e_xsk.c` — i40eドライバにおけるXDP
- `samples/bpf/xdpsock_user.c` — AF_XDPのユーザー空間の例
- `tools/lib/bpf/xsk.c` — libbpfのAF_XDPヘルパー
