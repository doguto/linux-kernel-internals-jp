# ソケット層の概要 {#socket-layer-overview}

> カーネルはユーザー空間のソケット呼び出しをプロトコル実装にどう対応付けるか

## ソケットという抽象化 {#the-socket-abstraction}

ソケットは双方向の通信エンドポイントである。ユーザー空間からは、単なるファイルディスクリプタに見える。内部的には、3つの構造体が積み重なったものである:

```
userspace:   fd (int)
             ↓
kernel VFS:  struct file → f_op = socket_file_ops
             ↓
socket:      struct socket (VFS view: state, type, proto_ops)
             ↓
sock:        struct sock (protocol view: receive queues, buffers, state machine)
             ↓
protocol:    struct proto (TCP/UDP/RAW-specific operations)
```

## struct socket — VFS側の層 {#struct-socket-the-vfs-facing-layer}

```c
// include/linux/net.h
struct socket {
    socket_state        state;   // SS_FREE、SS_UNCONNECTED、SS_CONNECTING、
                                  // SS_CONNECTED、SS_DISCONNECTING
    short               type;    // SOCK_STREAM、SOCK_DGRAM、SOCK_RAW、...
    unsigned long       flags;
    struct file         *file;   // 対応するファイル（fd → socketの検索用）
    struct sock         *sk;     // プロトコルレベルのソケット
    const struct proto_ops *ops; // VFSからプロトコルへのディスパッチテーブル
    struct socket_wq    *wq;     // poll/select用の待ちキュー
};
```

`proto_ops`は、ソケットのシステムコールをプロトコルのコードに対応付けるディスパッチテーブルである:

```c
// include/linux/net.h
struct proto_ops {
    int family;
    int (*bind)     (struct socket *, struct sockaddr *, int);
    int (*connect)  (struct socket *, struct sockaddr *, int, int);
    int (*accept)   (struct socket *, struct socket *, struct proto_accept_arg *);
    int (*listen)   (struct socket *, int);
    int (*sendmsg)  (struct socket *, struct msghdr *, size_t);
    int (*recvmsg)  (struct socket *, struct msghdr *, size_t, int);
    int (*setsockopt)(struct socket *, int, int, sockptr_t, unsigned int);
    int (*getsockopt)(struct socket *, int, int, char __user *, int __user *);
    int (*shutdown) (struct socket *, int);
    // ...
};
```

TCP/IPv4では`proto_ops = inet_stream_ops`。UDPでは`proto_ops = inet_dgram_ops`。AF_UNIXでは`proto_ops = unix_stream_ops`である。

## struct sock — プロトコル側の層 {#struct-sock-the-protocol-facing-layer}

```c
// include/net/sock.h
struct sock {
    struct sock_common __sk_common; // アドレス/ポート/ファミリ/状態
    // 共通のマクロがこれらのフィールドを公開する:
    // sk_daddr, sk_rcv_saddr  （IPv4アドレス）
    // sk_dport, sk_num        （ポート）
    // sk_state                （TCP_ESTABLISHED、TCP_CLOSE_WAIT、...）
    // sk_prot                 （struct protoを指す）

    // 受信パス
    struct sk_buff_head sk_receive_queue; // 受信済みだがまだ読まれていない
    struct {
        atomic_t rmem_alloc;
        struct sk_buff *head, *tail;
    } sk_backlog;               // ロック保持中に到着したパケット

    // メモリの上限
    int           sk_rcvbuf;    // 受信バッファサイズの上限
    int           sk_sndbuf;    // 送信バッファサイズの上限
    atomic_t      sk_rmem_alloc; // 受信用に現在割り当てられている量
    atomic_t      sk_wmem_alloc; // 送信用に現在割り当てられている量

    // コールバック（プロトコルが設定する）
    void    (*sk_data_ready)(struct sock *sk);    // データが到着した
    void    (*sk_write_space)(struct sock *sk);   // 送信バッファに空きができた
    void    (*sk_state_change)(struct sock *sk);  // TCPの状態が変わった
    void    (*sk_error_report)(struct sock *sk);  // エラー（ICMPなど）

    // プロトコル固有の拡張（TCPはtcp_sockを使う、など）
    // プロトコル層からcontainer_of()でアクセスされる
};
```

## struct proto — プロトコルの実装 {#struct-proto-the-protocol-implementation}

各プロトコルは、実際の処理を含む`struct proto`を登録する:

```c
// include/net/sock.h
struct proto {
    void  (*close)     (struct sock *, long timeout);
    int   (*connect)   (struct sock *, struct sockaddr *, int);
    int   (*disconnect)(struct sock *, int);
    struct sock *(*accept)(struct sock *, struct proto_accept_arg *);
    int   (*sendmsg)   (struct sock *, struct msghdr *, size_t);
    int   (*recvmsg)   (struct sock *, struct msghdr *, size_t, int, int *);
    int   (*backlog_rcv)(struct sock *, struct sk_buff *);  // バックログを処理する
    int   (*hash)      (struct sock *);   // プロトコルのハッシュテーブルに追加する
    void  (*unhash)    (struct sock *);
    int   (*get_port)  (struct sock *, unsigned short);     // ポートのバインド
    char  name[32];   // "TCP"、"UDP"、"RAW"
    // ...
};
```

TCP: `net/ipv4/tcp_ipv4.c`の`struct proto tcp_prot`
UDP: `net/ipv4/udp.c`の`struct proto udp_prot`

## ソケットの作成: socket(2) {#creating-a-socket-socket2}

```
socket(AF_INET, SOCK_STREAM, 0)
    → sys_socket()
        → sock_create()
            → __sock_create()
                → net_families[AF_INET]->create()  # = inet_create()
                    → allocate struct socket + struct sock (tcp_sock)
                    → set sock->sk_prot = &tcp_prot
                    → set sock->ops = &inet_stream_ops
                    → tcp_prot.init(sk)
        → sock_map_fd()  # install in fd table
```

## 2段階のディスパッチ {#the-two-dispatch-levels}

二重のディスパッチ（proto_ops → proto）が存在するのは、次の理由による:

- `proto_ops`は**VFSインターフェース**（ファイルディスクリプタのセマンティクス、ソケットの状態、アドレスファミリによるディスパッチ）を扱う
- `proto`は**プロトコルの実装**（TCPのステートマシン、UDPの振り分けなど）を扱う

1つの層で両方を担うことはできない。TCPソケットへの`sendmsg()`とUDPソケットへの`sendmsg()`は異なる動作を必要とするが、どちらも同じVFSのファイル操作を通るからである。

## 受信パス: データがソケットに届くまで {#receive-path-how-data-reaches-the-socket}

```
NIC interrupt → NAPI poll → __netif_receive_skb()
    → IP input → TCP/UDP demux → socket lookup
        → sk->sk_receive_queue or sk->sk_backlog
            → sk->sk_data_ready(sk)   # wake up reader
                → sock_def_readable() → wake_up_interruptible(&sk->sk_wq)
                    → read()/recv() returns data
```

`sk_backlog`は、ソケットのロックが保持されている間（例えば別のシステムコールの実行中）にパケットが到着した場合を扱う。パケットはバックログに保持され、ロックが解放されたときに処理される。

## ソケットバッファとメモリ {#socket-buffers-and-memory}

```bash
# デフォルトの受信/送信バッファサイズ
cat /proc/sys/net/core/rmem_default  # バイト単位
cat /proc/sys/net/core/wmem_default

# 最大値（setsockoptのSO_RCVBUF/SO_SNDBUFの上限）
cat /proc/sys/net/core/rmem_max
cat /proc/sys/net/core/wmem_max

# TCP固有の自動チューニング範囲 [min, default, max]
cat /proc/sys/net/ipv4/tcp_rmem
cat /proc/sys/net/ipv4/tcp_wmem
```

`sk->sk_rmem_alloc`は、`sk_receive_queue`に現在保持されているメモリ量を追跡する。これが`sk_rcvbuf`を超えると、新たに到着したパケットはカーネルによって黙って破棄される。TCPなら送信側が再送するが、UDPではデータは単に失われる。

## ソケットのステートマシン（TCP） {#socket-state-machine-tcp}

```
                   connect()
                      ↓
    CLOSE → SYN_SENT → ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSE
                                  ↘ CLOSE_WAIT → LAST_ACK → CLOSE

                                  accept() returns a new ESTABLISHED socket; listening socket remains in LISTEN
```

これは`sk->sk_state`（`sk->__sk_common.skc_state`経由でアクセスされる）が追跡する。TCPプロトコルの実装がこれらの状態を遷移させ、`sk->sk_state_change()`を呼んで待機中のポーラーに通知する。

## ソケットの調査 {#inspecting-sockets}

```bash
# すべてのTCPソケットを一覧表示
ss -tnap

# ソケットのメモリ使用量を表示
ss -tm

# ソケットのカーネル内部状態を表示（Linux 5.1以降）
ss --diag

# /proc/net/tcpはソケットテーブルを示す
cat /proc/net/tcp  # 16進: local_addr:port remote_addr:port state tx_queue rx_queue uid

# 状態ごとにソケットを数える
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn
```

## 参考文献 {#further-reading}

- [sk_buff](sk-buff.md) — 層の間で受け渡されるパケット構造体
- [ネットワークデバイスとNAPI](napi.md) — パケットがハードウェアからどう届くか
- [パケットの一生（受信）](life-of-packet-rx.md) — NICからソケットまでの全経路
- [TCPの実装](tcp.md) — TCPのステートマシンと輻輳制御
