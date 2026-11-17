# cgroup BPFプログラム {#cgroup-bpf-programs}

> ネットワークポリシー、デバイス制御、リソースアカウンティングのためにeBPFプログラムをcgroupにアタッチする

## 概要 {#overview}

BPFプログラムはcgroupディレクトリにアタッチでき、そのcgroupとその子孫に属するすべてのプロセスに対して実行される。これにより、カーネルにパッチを当てたりカーネルモジュールを書いたりせずに、プログラム可能なcgroup単位のポリシーを実現できる。cgroup BPFプログラムは、明確に定義されたフックポイント（ソケットの作成、パケットの到着、デバイスノードのオープンなど）でカーネル内で実行され、何をすべきかを決定する。

従来のcgroupコントローラ（固定されたインターフェースファイルの集合を公開する）とは異なり、cgroup BPFでは、検証済みでサンドボックス化されたプログラムで任意のポリシーを表現できる。同じ仕組みが、KubernetesのNetworkPolicyの強制、コンテナのデバイスのホワイトリスト、サービスごとのTCPチューニングを支えている。

## アタッチタイプ {#attach-types}

アタッチタイプは `include/uapi/linux/bpf.h` で `enum bpf_attach_type` として定義されている。cgroupに関連するものは次のとおりである:

### ネットワークパケットのフィルタリング {#network-packet-filtering}

`BPF_CGROUP_INET_INGRESS` / `BPF_CGROUP_INET_EGRESS`

cgroup内のプロセスが所有するソケットが送受信するすべてのIPv4/IPv6パケットに対して呼ばれる。プログラムは `__sk_buff` コンテキストを受け取り、次を返す:

- `1` — パケットを通す
- `0` — パケットを破棄する

```c
/* egress: この cgroup から 10.0.0.0/8 宛てのパケットをすべて破棄する */
SEC("cgroup_skb/egress")
int block_internal(struct __sk_buff *skb)
{
    void *data     = (void *)(long)skb->data;
    void *data_end = (void *)(long)skb->data_end;
    struct iphdr *ip = data;

    if (ip + 1 > data_end)
        return 1;

    /* 10.0.0.0/8 宛てのパケットを破棄する */
    if ((bpf_ntohl(ip->daddr) & 0xff000000) == 0x0a000000)
        return 0;
    return 1;
}
```

### ソケット作成の制御 {#socket-creation-control}

`BPF_CGROUP_INET_SOCK_CREATE`

cgroup内のプロセスが `socket()` を呼び出したときに呼ばれる。プログラムは `struct bpf_sock` を受け取り、`0` を返すことでソケットの作成を拒否できる:

```c
SEC("cgroup/sock_create")
int restrict_raw_sockets(struct bpf_sock *sk)
{
    /* raw ソケット（SOCK_RAW）の作成を拒否する — STREAM と DGRAM のみ許可 */
    if (sk->type == SOCK_RAW)
        return 0;
    return 1;
}
```

これは `socket()` に対するseccompのフィルタリングよりも軽量である。BPFプログラムは、システムコールの引数をデコードしなくてもソケットのメタデータにアクセスできるからである。

### TCPのライフサイクルフック {#tcp-lifecycle-hooks}

`BPF_CGROUP_SOCK_OPS`

TCPのライフサイクルにおける複数のイベント（コネクション確立、タイムアウト、再送、RTOの変化）で呼ばれ、cgroupごとにソケットオプションを照会・設定できる。プログラムは `struct bpf_sock_ops` を受け取る:

```c
SEC("sockops")
int set_buffer_sizes(struct bpf_sock_ops *skops)
{
    switch (skops->op) {
    case BPF_SOCK_OPS_ACTIVE_ESTABLISHED_CB:
        /* コネクション確立（クライアント側）: TCP の送信バッファを増やす */
        bpf_setsockopt(skops, SOL_SOCKET, SO_SNDBUF, &(int){4 * 1024 * 1024},
                       sizeof(int));
        break;
    case BPF_SOCK_OPS_PASSIVE_ESTABLISHED_CB:
        /* コネクション確立（サーバー側） */
        bpf_setsockopt(skops, SOL_SOCKET, SO_RCVBUF, &(int){4 * 1024 * 1024},
                       sizeof(int));
        break;
    }
    return 1;
}
```

これをサービスのcgroupにアタッチすると、ホスト上の全プロセスに影響するグローバルな `sysctl` の値に触れることなく、そのサービスが開くすべてのソケットにTCPチューニングを適用できる。

### デバイスアクセス制御 {#device-access-control}

`BPF_CGROUP_DEVICE`

cgroup内のプロセスがデバイスノードを `open()`、`mknod()`、`access()` しようとしたときに呼ばれる。プログラムは、デバイスのmajor/minorと要求されたアクセス種別（`BPF_DEVCG_ACC_READ`、`BPF_DEVCG_ACC_WRITE`、`BPF_DEVCG_ACC_MKNOD`）を記述した `struct bpf_cgroup_dev_ctx` を受け取り、許可なら `1`、拒否なら `0` を返す:

```c
SEC("cgroup/dev")
int device_policy(struct bpf_cgroup_dev_ctx *ctx)
{
    /* /dev/null (1:3) と /dev/zero (1:5) へのアクセスのみ許可する */
    if (ctx->major == 1 && (ctx->minor == 3 || ctx->minor == 5))
        return 1;
    /* 端末エミュレータ用に /dev/pts/* (136:*) を許可する */
    if (ctx->major == 136)
        return 1;
    return 0;
}
```

`BPF_CGROUP_DEVICE` は、レガシーなcgroup v1の `devices` コントローラをcgroup v2で置き換えるものである。v1のコントローラは `devices.allow` / `devices.deny` ファイルを公開していたが、cgroup v2では `devices` コントローラが完全に廃止され、代わりに `BPF_CGROUP_DEVICE` に依存している。runcのようなコンテナランタイムは、OCI仕様の `linux.resources.devices` リストから `BPF_CGROUP_DEVICE` プログラムを生成してロードする。

### sysctlの傍受 {#sysctl-interception}

`BPF_CGROUP_SYSCTL`

プロセスが `/proc/sys` 配下のsysctlを読み書きしたときに呼ばれる。プログラムは `struct bpf_sysctl` を受け取り、値を検査・変更したり、操作を拒否したりできる:

```c
SEC("cgroup/sysctl")
int sysctl_filter(struct bpf_sysctl *ctx)
{
    /* cgroup 内からの net.core.somaxconn への書き込みを拒否する */
    char name[64] = {};
    bpf_sysctl_get_name(ctx, name, sizeof(name), 0);
    if (ctx->write && __builtin_memcmp(name, "net/core/somaxconn", 18) == 0)
        return 0;
    return 1;
}
```

### ソケットオプションの傍受 {#socket-option-interception}

`BPF_CGROUP_GETSOCKOPT` / `BPF_CGROUP_SETSOCKOPT`

`getsockopt(2)` や `setsockopt(2)` が発行されたときに呼ばれる。プログラムは、アプリケーションから見えるオプション値を上書きしたり（cgroupごとのデフォルト値を返すのに便利）、特定のソケットオプションを完全にブロックしたりできる。

## アタッチフラグ {#attach-flags}

`bpf(BPF_PROG_ATTACH, ...)` でプログラムをアタッチする際、`flags` フィールドが階層での振る舞いを制御する:

| フラグ | 意味 |
|------|---------|
| （なし / `0`） | 排他: アタッチタイプごとにcgroupあたり1つのプログラムのみ。子のcgroupは継承するが上書きできない |
| `BPF_F_ALLOW_OVERRIDE` | 子のcgroupがこのプログラムを自身のプログラムで置き換えられる |
| `BPF_F_ALLOW_MULTI` | 複数のプログラムをアタッチできる。親と子の両方のプログラムが実行される（親が先） |

実際に最もよく使われるのは `BPF_F_ALLOW_MULTI` である。ルートcgroupにプラットフォームレベルのポリシーを置き、子のcgroupにコンテナごとの細かな調整を置いて、両方を実行させることができる。

## システムコールのインターフェース {#the-syscall-interface}

```c
/* プログラムを cgroup にアタッチする */
union bpf_attr attr = {
    .target_fd   = cgroup_fd,           /* open("/sys/fs/cgroup/c1", O_RDONLY) */
    .attach_bpf_fd = prog_fd,           /* ロード済みの BPF プログラムの fd */
    .attach_type = BPF_CGROUP_INET_EGRESS,
    .attach_flags = BPF_F_ALLOW_MULTI,
};
bpf(BPF_PROG_ATTACH, &attr, sizeof(attr));

/* デタッチ */
attr.attach_bpf_fd = prog_fd;
bpf(BPF_PROG_DETACH, &attr, sizeof(attr));

/* cgroup にアタッチされたプログラムを照会する */
attr.query.target_fd  = cgroup_fd;
attr.query.attach_type = BPF_CGROUP_INET_EGRESS;
bpf(BPF_PROG_QUERY, &attr, sizeof(attr));
```

`libbpf` を使う場合、より高レベルのAPIは `bpf_prog_attach()` / `bpf_prog_detach()` である。

## 実践例: コンテナごとのegressレート制限 {#practical-example-per-container-egress-rate-limiting}

cgroup BPFは組み込みのレートリミッタを提供しない（それはブロックI/Oなら `io` コントローラ、ネットワークならTCの役目である）。定石は、`BPF_CGROUP_INET_EGRESS` プログラムと、コンテナの `veth` インターフェース上のTC `qdisc`（例: `tbf` や `fq`）を組み合わせることである:

```bash
# 1. コンテナ用の veth ペアを作成する
ip link add veth0 type veth peer name eth0

# 2. veth0（ホスト側）に TBF qdisc をアタッチする — 10Mbit/s
tc qdisc add dev veth0 root tbf rate 10mbit burst 32kbit latency 400ms

# 3. コンテナの cgroup に、パケットにマークを付ける BPF プログラムをアタッチする
#    （skb->mark を設定し、TC の分類器が cgroup ごとのフローを識別できるようにする）
bpftool prog load mark_egress.bpf.o /sys/fs/bpf/mark_egress
bpftool cgroup attach /sys/fs/cgroup/containers/c1/ egress \
    pinned /sys/fs/bpf/mark_egress allow_multi
```

BPFプログラム自体は `bpf_skb_store_bytes()` を使うか、`skb->mark` にコンテナIDを設定する。そしてTCのflower/u32分類器がそのマークにマッチして、コンテナごとの `tbf` qdiscを適用する。

純粋にBPFだけで実現する方法としては、`BPF_MAP_TYPE_ARRAY` で（アトミック操作を使って）独自のトークンバケットを実装し、バケットが空のときにパケットを破棄することで、`BPF_CGROUP_INET_EGRESS` フックから直接パケットのレートを制限できる。

## アタッチされたプログラムの調査 {#inspecting-attached-programs}

```bash
# cgroup のサブツリーにアタッチされたすべての BPF プログラムを表示する:
bpftool cgroup tree /sys/fs/cgroup/containers/c1/

# 単一の cgroup ディレクトリにアタッチされたプログラムを表示する:
bpftool cgroup show /sys/fs/cgroup/

# 出力例:
# /sys/fs/cgroup/containers/c1
# ID       AttachType      AttachFlags     Name
# 42       egress          multi           block_internal
# 43       device          multi           device_policy
```

`bpftool cgroup tree` は階層を再帰的にたどり、アタッチされたすべてのプログラムをID、名前、フラグとともに一覧表示する。特定のプログラムのバイトコード、マップ、BTF情報を調べるには `bpftool prog show id 42` を使う。

## カーネルでの実装 {#kernel-implementation}

カーネルは、ネットワーク、ソケット、デバイスの各サブシステムのホットパスからcgroup BPFプログラムを実行する。内部のエントリポイントには `__cgroup_bpf_run_filter_*()` 系の関数（ダブルアンダースコアのプレフィックス）が使われ、これらは `include/linux/bpf-cgroup.h` で定義され、`BPF_CGROUP_RUN_PROG_*` ラッパーマクロを介して呼び出される:

```c
/* include/linux/bpf-cgroup.h（シグネチャは簡略化） */
int __cgroup_bpf_run_filter_skb(struct sock *sk, struct sk_buff *skb,
                                 enum cgroup_bpf_attach_type atype);
int __cgroup_bpf_run_filter_sk(struct sock *sk,
                                enum cgroup_bpf_attach_type atype);
int __cgroup_bpf_run_filter_sock_ops(struct sock *sk,
                                     struct bpf_sock_ops_kern *sock_ops,
                                     enum cgroup_bpf_attach_type atype);
int __cgroup_bpf_check_dev_permission(short dev_type, u32 major, u32 minor,
                                       short access,
                                       enum cgroup_bpf_attach_type atype);
```

これらは次の場所からマクロ経由で呼び出される:

- `net/ipv4/ip_output.c`、`net/ipv6/ip6_output.c` → `BPF_CGROUP_RUN_PROG_INET_EGRESS`
- `net/ipv4/ip_input.c`（`ip_rcv`） → `BPF_CGROUP_RUN_PROG_INET_INGRESS`
- `net/socket.c`（`__sock_create`） → `BPF_CGROUP_RUN_PROG_INET_SOCK`
- `security/device_cgroup.c`（`devcgroup_check_permission`） → `BPF_CGROUP_RUN_PROG_DEVICE_CGROUP`

`struct cgroup` に埋め込まれた `struct cgroup_bpf` が、アタッチされたプログラムを保持する:

```c
/* include/linux/bpf-cgroup-defs.h */
struct cgroup_bpf {
    /*
     * effective[] はこの cgroup に対して有効なプログラムの集合を保持する —
     * ここにアタッチされたプログラムと、祖先から継承したプログラム（平坦化済み）。
     */
    struct bpf_prog_array __rcu *effective[MAX_CGROUP_BPF_ATTACH_TYPE];

    /* この cgroup に直接アタッチされたプログラム */
    struct hlist_head     progs[MAX_CGROUP_BPF_ATTACH_TYPE];
    u8                    flags[MAX_CGROUP_BPF_ATTACH_TYPE];

    struct list_head      storages;
    struct bpf_prog_array *inactive; /* RCU による更新用 */
};
```

プログラムがアタッチされると、（`__cgroup_bpf_attach()` から呼ばれる）`update_effective_progs()` がルートから対象のcgroupまでをたどり、そのcgroupとすべての子孫の `effective[]` プログラム配列を再構築する。`cgroup_bpf_inherit()` は別の関数で、cgroupの作成時にのみ呼ばれ、親から継承したプログラムをコピーする。ホットパスでは `BPF_PROG_RUN_ARRAY()` を介して `effective[]` を参照するだけであり、実行時に階層をたどることはない。

## 参考文献 {#further-reading}

- [cgroup v2のアーキテクチャ](cgroup-v2.md) — `struct cgroup` と統一階層
- [コンテナの隔離](container-isolation.md) — BPFのデバイスポリシーがseccompやケーパビリティとどう組み合わさるか
- [BPFネットワーキング（TC、cgroup、sockmap）](../bpf/bpf-networking.md) — cgroup BPFと併用するTCベースのレート制限
- [BPFベリファイア](../bpf/bpf-verifier.md) — BPFプログラムがアタッチ前にどのように安全性検査を受けるか
- `kernel/bpf/cgroup.c` — アタッチ/デタッチ/実行の実装
- `include/uapi/linux/bpf.h` — `enum bpf_attach_type` とすべての `BPF_CGROUP_*` 定数
- `tools/bpf/bpftool/cgroup.c` — bpftoolのcgroupサブコマンドの実装
- `Documentation/bpf/prog_cgroup_skb.rst` — アップストリームのドキュメント
