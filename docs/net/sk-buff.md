# sk_buff: ネットワークバッファ {#sk_buff-the-network-buffer}

> Linuxネットワークスタックの中心となるデータ構造

## sk_buffとは何か {#what-sk_buff-is}

`struct sk_buff`（ソケットバッファ）は、カーネルがネットワークパケットを表現する方法である。すべてのパケットは、ハードウェアから届いた瞬間からソケットに配送される瞬間まで、`sk_buff`の中で生きている。

この構造体は2つの部分からなる:
1. **メタデータヘッダ**: ポインタ、長さフィールド、プロトコル情報、タイムスタンプ
2. **データバッファ**: 実際のパケットのバイト列（埋め込まれているのではなく、ポインタで指される）

すべての層が`sk_buff`を操作するため、`sk_buff`を理解することはネットワークスタック全体を理解するための基礎となる。

## メモリレイアウト {#memory-layout}

```
sk_buff metadata:
┌──────────────────────────────────────┐
│ next, prev  (list linkage)           │
│ sk          (owning socket)          │
│ dev         (network device)         │
│ len, data_len                        │
│ protocol, cloned, ...                │
│ head, data, tail, end (data ptrs)    │
│ cb[48]      (layer-private scratch)  │
│ ...                                  │
└──────────────────────────────────────┘

Packet data buffer (allocated separately):
 ┌─────────────┬───────────────────────────────────────┬────────┐
 │  headroom   │  packet data (data ... tail)          │  end   │
 └─────────────┴───────────────────────────────────────┴────────┘
 ↑ head        ↑ data                           ↑ tail           ↑ end

As headers are added (push down the stack):
  data moves left: skb_push()
As headers are consumed (pop up the stack):
  data moves right: skb_pull()
```

`headroom`は`data`の手前に予約された領域である。パケットがスタックを下っていくにつれ、下位の層（Ethernet、IP）はこのheadroomに自分のヘッダを押し込む。これによりコピーが避けられる。

## struct sk_buffの主要なフィールド {#struct-sk_buff-key-fields}

```c
// include/linux/skbuff.h
struct sk_buff {
    // リストのリンク（キュー、ハッシュチェイン内）
    struct sk_buff  *next;
    struct sk_buff  *prev;

    struct sock     *sk;        // 所有するソケット（転送パケットでは NULL）
    struct net_device *dev;     // このパケットが届いた / 出ていくネットワークデバイス

    ktime_t         tstamp;     // パケットのタイムスタンプ

    // 層ごとの作業領域: 各層が自分のデータをここに格納する。
    // パケットがその層にある間だけ有効。
    char            cb[48] __aligned(8);

    unsigned int    len;        // 全体の長さ（リニア + ページ化データ）
    unsigned int    data_len;   // ページ化データの長さ（scatter-gather）
    __u16           mac_len;    // MAC ヘッダの長さ
    __u16           hdr_len;    // clone されたヘッダの長さ

    __u8            cloned:1;   // 別の skb とデータを共有している
    __u8            head_frag:1; // head がページフラグメント（kmalloc ではない）
    __u8            pp_recycle:1; // page_pool が所有している

    // プロトコル情報
    __be16          protocol;   // Ethernet タイプ（ETH_P_IP, ETH_P_IPV6, ...）
    __u16           transport_header;  // L4 ヘッダへのオフセット
    __u16           network_header;    // L3 ヘッダ（IP）へのオフセット
    __u16           mac_header;        // L2 ヘッダ（Ethernet）へのオフセット

    // データポインタ
    sk_buff_data_t  tail;       // リニアデータの終端
    sk_buff_data_t  end;        // バッファの終端（ハードリミット）
    unsigned char   *head;      // バッファの先頭
    unsigned char   *data;      // 有効なデータの先頭

    unsigned int    truesize;   // ソケットに課金される総メモリ量

    // ...
    // 注: ページ化されたフラグメントは struct sk_buff 自体の中にはない。
    // それらは skb->end にある struct skb_shared_info に置かれる:
    //   skb_shinfo(skb)->frags[0..nr_frags-1]
};
```

## データポインタの操作 {#data-pointer-manipulation}

```c
// 先頭にデータを押し込む（スタックを下るときにヘッダを付加する）
void *skb_push(struct sk_buff *skb, unsigned int len)
{
    skb->data -= len;
    skb->len += len;
    return skb->data;
}

// 先頭からデータを取り除く（スタックを上るときにヘッダを消費する）
void *skb_pull(struct sk_buff *skb, unsigned int len)
{
    skb->data += len;
    skb->len -= len;
    return skb->data;
}

// 末尾にデータを追加する
void *skb_put(struct sk_buff *skb, unsigned int len)
{
    void *tmp = skb_tail_pointer(skb);
    skb->tail += len;
    skb->len += len;
    return tmp;
}
```

## ヘッダアクセス用マクロ {#header-access-macros}

```c
// L2/L3/L4 ヘッダへのポインタを得る
static inline struct ethhdr *eth_hdr(const struct sk_buff *skb)
{
    return (struct ethhdr *)skb_mac_header(skb);
}

static inline struct iphdr *ip_hdr(const struct sk_buff *skb)
{
    return (struct iphdr *)skb_network_header(skb);
}

static inline struct tcphdr *tcp_hdr(const struct sk_buff *skb)
{
    return (struct tcphdr *)skb_transport_header(skb);
}

// 例: 受信パケットから宛先 IP を読む
struct iphdr *iph = ip_hdr(skb);
__be32 dst = iph->daddr;
```

## cb[]作業領域 {#the-cb-scratch-area}

各層は`cb[]`に48バイトのプライベートな記憶領域を持つ。これは重要な設計であり、層ごとのメタデータのための別個の割り当てを避けられる:

```c
// TCP は cb[] を使って TCP 固有の情報を格納する
struct tcp_skb_cb {
    union {
        struct inet_skb_parm    h4;
        struct inet6_skb_parm   h6;
    } header;
    __u32   seq;            // 開始シーケンス番号
    __u32   end_seq;        // SEQ + FIN + SYN + datalen
    __u8    tcp_flags;      // TCP ヘッダのフラグ
    __u8    sacked;         // SACK ビットの状態
    // ...
};
#define TCP_SKB_CB(__skb)    ((struct tcp_skb_cb *)&((__skb)->cb[0]))

// 使い方:
TCP_SKB_CB(skb)->seq = tcp_seq;
```

## cloneと共有 {#cloning-and-sharing}

`skb_clone()`は、同じデータバッファを共有する新しい`sk_buff`を作成する。両方に`cloned`ビットがセットされる。同じパケットを複数の消費者に送る（例: パケットソケット＋転送）場合に便利である:

```c
struct sk_buff *clone = skb_clone(skb, GFP_ATOMIC);
// clone->data と skb->data は同じバッファを指す
// ヘッダを変更するには、先に skb_copy_expand() が必要
```

`skb_copy()`は完全に独立したコピーを作成する。

## フラグメント化されたパケット（scatter-gather） {#fragmented-packets-scatter-gather}

大きなパケット（例: TCPオフロード）では、データが複数のページにまたがることがある:

```c
/* ページ化された部分は skb->end にある skb_shared_info に置かれる: */
struct skb_shared_info {
    skb_frag_t frags[MAX_SKB_FRAGS];
    __u8       nr_frags;
    // ...
};
// アクセス方法: skb_shinfo(skb)->frags[i]
// data_len = ページ化された部分の合計長
// len = リニア + ページ化

// アクセス可能なデータの全体:
// skb->data ... skb->tail  （リニア）
// + frags[0] ... frags[nr_frags-1]  （ページ化）
```

`skb_linearize()`のような関数は、ドライバがscatter-gatherをサポートしていない場合に、ページ化データをリニア部分にまとめる。

## 割り当てと解放 {#allocation-and-freeing}

```c
// ヘッダ用の headroom を持つ新しい sk_buff を割り当てる
struct sk_buff *alloc_skb(unsigned int size, gfp_t priority);

// ネットワーク用アロケータから割り当てる（RX パスでは page_pool を使う）
struct sk_buff *napi_alloc_skb(struct napi_struct *napi, unsigned int length);

// sk_buff を解放する（参照カウントを減らし、0 になれば解放）
void kfree_skb(struct sk_buff *skb);

// エラー計上なしで解放する（通常のパス）
void consume_skb(struct sk_buff *skb);
```

`truesize`はソケットの受信バッファ（`sk->sk_rmem_alloc`）に課金される。ソケットバッファが満杯になると、新しいパケットは破棄される。

## なぜこの設計なのか {#why-this-design}

### リニア部分とページ化部分の分割 {#the-linear-paged-split}

初期のsk_buffの実装は、パケットデータを単一の連続した`kmalloc()`バッファに格納していた。これは、2000年代初頭にNICがscatter-gather DMAの能力を獲得するまではうまく機能していた。

scatter-gatherに対応したNICは、受信パケットを非連続な物理ページにDMAで書き込める。ペイロードはページキャッシュのページへ、ヘッダは小さなslab割り当てへ入る。すべてを1つの連続したバッファにコピーすれば、メモリとCPUサイクルが無駄になる。カーネルには、その両方を保持できる表現が必要だった。

その解決策が、リニア＋フラグメントのモデルである:

- **リニア部分**（`head`…`tail`）はパケットヘッダを保持する。小さく、常に連続しており、アクセスが速い。
- **ページ化フラグメント**（`skb_shinfo(skb)->frags[]`）はペイロードを保持する。大きく、DMAから直接来た非連続なページであることもある。

TCP Segmentation Offload（TSO、2002年頃）がこれを確固たるものにした。TSOに対応したNICは、単一の大きなバッファ（例: 64KB）を受け取り、MTUサイズのEthernetフレームに分割できる。カーネルはTCPペイロード全体をページ化フラグメント領域に持つsk_buffを1つ送り、NICが分割を担当する。ページ化フラグメントがなければ、カーネルはすべてをソフトウェアで事前に分割しなければならなかっただろう。

### なぜ`skb_shared_info`は`skb->end`に置かれるのか {#why-skb_shared_info-lives-at-skb-end}

```c
/* net/core/skbuff.c: skb のデータレイアウト */
/*
 *  head
 *   ↓
 *  [headroom][  packet data  ][ skb_shared_info ]
 *            ↑               ↑                  ↑
 *           data            tail               end
 */
static inline struct skb_shared_info *skb_shinfo(const struct sk_buff *skb)
{
    return (struct skb_shared_info *)skb_end_pointer(skb);
}
```

`skb_shared_info`をリニアデータバッファの直後（`skb->end`）に置くことで、フラグメント配列のための別個のヒープ割り当てを避けられる。リニアバッファとフラグメントのメタデータは、1回の`kmalloc()`呼び出しでまとめて割り当てられる。これはホットパスにとって重要である。受信するすべてのパケットがsk_buffを割り当てるため、それを1回の割り当てに減らすことで、高いパケットレートでは大きなオーバーヘッドが削減される。

### cloneと参照カウント方式の設計 {#cloning-and-the-reference-counted-design}

`skb_clone()`は、同じデータバッファを共有する新しいsk_buffヘッダを作成する。これにより、同じパケットを複数の消費者に送ることができる。パケットソケットのリスナーと通常の転送パスの両方が、パケットデータをコピーすることなく同じsk_buffを「持つ」ことができる。

`cloned`ビットはデータバッファが共有されていることを示す。ヘッダを変更したい層は、まず`skb_unshare()`か`pskb_expand_head()`を呼んで自分専用のコピーを得なければならない。このコピーオンライトの規律によって、あるサブシステムが、別のサブシステムが読んでいるデータを破壊することを防いでいる。

## 参考文献 {#further-reading}

- [ソケット層の概要](socket-layer.md) — sk_buffがソケットとどうつながるか
- [ネットワークデバイスとNAPI](napi.md) — 受信時にsk_buffがどう割り当てられるか
- [パケットの一生（受信）](life-of-packet-rx.md) — sk_buffを使った完全な受信パス
- [パケットの一生（送信）](life-of-packet-tx.md) — 完全な送信パス
