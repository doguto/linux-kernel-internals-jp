# Unixドメインソケット {#unix-domain-sockets}

> ファイルディスクリプタの受け渡し、抽象名前空間、SOCK_SEQPACKETを備えたローカルIPC

Unixドメインソケット（`AF_UNIX`）は、全二重で、コネクション指向またはデータグラム型のIPCを完全にカーネル内で提供する。ネットワークスタックは関与しない。D-Bus、systemdのソケットアクティベーション、Dockerの制御ソケット、その他無数のローカルデーモンの基盤となっている。

## ソケットタイプ {#socket-types}

| タイプ | 意味論 | 境界の保持 | 用途 |
|------|-----------|--------------------|----------|
| `SOCK_STREAM` | 信頼性のあるバイトストリーム | なし | 高スループットのパイプ、データベース |
| `SOCK_DGRAM` | 信頼性のないデータグラム | あり | 短い送りっぱなしのメッセージ |
| `SOCK_SEQPACKET` | 信頼性があり順序の保たれたメッセージ | **あり** | RPC、特権分離 |

`SOCK_SEQPACKET`は構造化されたIPCにとって最適な選択肢である。`SOCK_STREAM`の信頼性と、`SOCK_DGRAM`のメッセージ境界の保持を兼ね備えている。各`send()`は相手側のちょうど1回の`recv()`に対応する。

```c
#include <sys/socket.h>
#include <sys/un.h>

/* サーバー側 */
int srv = socket(AF_UNIX, SOCK_SEQPACKET | SOCK_CLOEXEC, 0);
```

## struct sockaddr_un {#struct-sockaddr_un}

```c
#include <sys/un.h>

struct sockaddr_un {
    sa_family_t sun_family;   /* AF_UNIX */
    char        sun_path[108]; /* ソケットのパスまたは抽象名 */
};
```

`sun_path`は全体で108バイト（`UNIX_PATH_MAX`）である。抽象名前空間のソケットでは、最初のバイトが`\0`で、残りの107バイトが名前を構成する。

## ファイルシステム名前空間 {#filesystem-namespace}

ファイルシステムのパスにbindすると、VFS内にソケットのinodeが作成される:

```c
struct sockaddr_un addr = {
    .sun_family = AF_UNIX,
    /* NUL終端されたパス */
};
strncpy(addr.sun_path, "/run/myservice.sock", sizeof(addr.sun_path) - 1);

bind(srv, (struct sockaddr *)&addr, sizeof(addr));
listen(srv, SOMAXCONN);
```

特性:

- パーミッションはファイルのモードビットによって強制される（`chmod 0660 /run/myservice.sock`）
- ソケットファイルはサーバーの終了後も残る。再びbindする前に`unlink()`で削除する必要がある
- ファイルシステム名前空間によって制御される: 別々のマウント名前空間を持つコンテナは、互いのソケットファイルを見ることができない

## 抽象名前空間 {#abstract-namespace}

抽象ソケット名はヌルバイト（`\0`）で始まる。カーネルはbindをメモリ内で追跡し、ファイルシステムのエントリは存在しない:

```c
struct sockaddr_un addr = { .sun_family = AF_UNIX };
/* 最初のバイトは \0、続いて名前 */
memcpy(addr.sun_path, "\0myservice", 10);

/* sun_path の長さには先頭の \0 が含まれる */
socklen_t addrlen = offsetof(struct sockaddr_un, sun_path) + 10;
bind(srv, (struct sockaddr *)&addr, addrlen);
```

特性:

- それを参照する最後のファイルディスクリプタが閉じられると自動的に片付けられる。`unlink()`は不要
- 名前はCの文字列ではなく任意のバイト列であり、先頭以外にもヌルバイトを含められる
- 同じネットワーク名前空間内からのみ見える（`ip netns`やコンテナの名前空間が分離を提供する）
- `ss -xlp`は、アドレス列で抽象ソケットに`@`の接頭辞を付けて表示する

## socketpair() {#socketpair}

`socketpair()`は、bindやlistenなしに接続済みのソケットのペアを作成する。親プロセスと子プロセスの間に双方向チャネルを作る最も簡単な方法である:

```c
int sv[2];
socketpair(AF_UNIX, SOCK_SEQPACKET | SOCK_CLOEXEC, 0, sv);
/* sv[0] と sv[1] は接続済み */

if (fork() == 0) {
    close(sv[0]);
    /* 子は sv[1] を使う */
} else {
    close(sv[1]);
    /* 親は sv[0] を使う */
}
```

双方向IPCにはパイプよりもこちらが望ましく、構造化された（メッセージ境界を保持する）通信には匿名ソケットペアよりもこちらが望ましい。

## SCM_RIGHTSによるファイルディスクリプタの受け渡し {#file-descriptor-passing-with-scm_rights}

`SCM_RIGHTS`制御メッセージを使うと、プロセスはUnixソケット越しにオープン済みのファイルディスクリプタを送信できる。カーネルはファイル記述（file description）を受信者のファイルディスクリプタテーブルに複製する。受信者は、オフセット、フラグ、アクセスモードを含め、同じ下位のオープンファイルを指す新しいfd番号を得る。

```c
#include <sys/socket.h>
#include <sys/un.h>

/* --- 送信側 --- */
int fd_to_pass = open("/etc/passwd", O_RDONLY);

struct msghdr msg = {};
char buf[CMSG_SPACE(sizeof(int))];  /* 制御メッセージバッファ */
struct iovec iov = { .iov_base = "x", .iov_len = 1 }; /* 1バイト以上のデータを送る必要がある */

msg.msg_iov        = &iov;
msg.msg_iovlen     = 1;
msg.msg_control    = buf;
msg.msg_controllen = sizeof(buf);

struct cmsghdr *cmsg = CMSG_FIRSTHDR(&msg);
cmsg->cmsg_level = SOL_SOCKET;
cmsg->cmsg_type  = SCM_RIGHTS;
cmsg->cmsg_len   = CMSG_LEN(sizeof(int));
memcpy(CMSG_DATA(cmsg), &fd_to_pass, sizeof(int));

sendmsg(sock, &msg, 0);
close(fd_to_pass);  /* 送信側ではもう不要 */

/* --- 受信側 --- */
char data[1];
char cbuf[CMSG_SPACE(sizeof(int))];
struct iovec riov = { .iov_base = data, .iov_len = sizeof(data) };
struct msghdr rmsg = {
    .msg_iov        = &riov,
    .msg_iovlen     = 1,
    .msg_control    = cbuf,
    .msg_controllen = sizeof(cbuf),
};

recvmsg(sock, &rmsg, 0);

struct cmsghdr *rcmsg = CMSG_FIRSTHDR(&rmsg);
if (rcmsg && rcmsg->cmsg_type == SCM_RIGHTS) {
    int received_fd;
    memcpy(&received_fd, CMSG_DATA(rcmsg), sizeof(int));
    /* received_fd はすぐに使える */
}
```

カーネルの`unix_stream_sendmsg()` / `unix_scm_to_skb()`のパス（`net/unix/af_unix.c`内）が、ファイル参照をソケットバッファに付加する。受信側では、`unix_detach_fds()`がそれらをskbから取り出し、続いて`scm_detach_fds()`（`net/core/scm.c`内）が`scm_recv_one_fd()`を介して各fdをインストールする。これは単独の`receive_fd()`ヘルパー（`fs/file.c`）が1つのfdに対して使うのと同じfdテーブルへのインストール処理であり、ここではまとめて一括でインストールするためにインライン化されている。

### リークのリスク {#leak-risk}

受け取ったfdは、エラーパスも含めてすべて閉じなければならない。受信者がfdを閉じなければ、それはプロセスのファイルディスクリプタテーブルに目に見えない形で蓄積していく。`lsof -p <pid>`で明らかになるが、`EMFILE`に達した時点ですでに被害は出ている。

*ソケット*に`SOCK_CLOEXEC`を指定して、ソケットのfd自体が`exec()`をまたいでリークするのを防ぐこと。受け取ったfdについては、受信直後に`fcntl(received_fd, F_SETFD, FD_CLOEXEC)`で`O_CLOEXEC`を設定する。

## SCM_CREDENTIALSによるピアの資格情報の受け渡し {#peer-credential-passing-with-scm_credentials}

`SCM_CREDENTIALS`を使うと、送信者は自身の`pid`、`uid`、`gid`をメッセージに付加できる。カーネルはその資格情報を送信者の実際の値と照合して検証する。プロセスは別のuidを偽装できない（rootでない限り）。

```c
/* ソケットで資格情報の受信を有効にする */
int enable = 1;
setsockopt(sock, SOL_SOCKET, SO_PASSCRED, &enable, sizeof(enable));

/* 送信側: 資格情報を付加する */
struct ucred cred = {
    .pid = getpid(),
    .uid = getuid(),
    .gid = getgid(),
};
char cbuf[CMSG_SPACE(sizeof(struct ucred))];
struct msghdr msg = { .msg_control = cbuf, .msg_controllen = sizeof(cbuf) };
struct cmsghdr *cmsg = CMSG_FIRSTHDR(&msg);
cmsg->cmsg_level = SOL_SOCKET;
cmsg->cmsg_type  = SCM_CREDENTIALS;
cmsg->cmsg_len   = CMSG_LEN(sizeof(struct ucred));
memcpy(CMSG_DATA(cmsg), &cred, sizeof(cred));
/* sendmsg() の前に msg.msg_iov/msg_iovlen も設定する必要がある --
   SCM_RIGHTS と同様、SCM_CREDENTIALS にも1バイト以上のデータが必要。 */
```

接続済みのソケットでは、より簡単な代替手段として`SO_PEERCRED`がある。これはメッセージごとの制御メッセージを必要とせずに、*接続先の*ピアの資格情報を返す:

```c
struct ucred peer;
socklen_t len = sizeof(peer);
getsockopt(sock, SOL_SOCKET, SO_PEERCRED, &peer, &len);
printf("peer pid=%d uid=%d gid=%d\n", peer.pid, peer.uid, peer.gid);
```

`SO_PEERCRED`は、systemd、D-Bus、polkitが特権操作を許可する前に呼び出し元プロセスを認証するために使われている。

## 性能 {#performance}

UnixドメインソケットはTCP/IPスタック全体を迂回する。データはカーネル内のソケット送信バッファと受信バッファの間で直接コピーされるか、構成によっては`sk_buff`を介したゼロコピーの工夫が使われる。実際のスループットはCPU、メッセージサイズ、カーネルバージョンに大きく依存するため、ここでは具体的な数値は示さない。ただし、以下の相対的な順序は成り立つ。各段階が、前の段階よりもコピーやプロトコルのオーバーヘッドを多く取り除くからである:

1. `AF_INET`ループバック（`127.0.0.1`）— 物理NICがないだけで、TCP/IPプロトコルスタック全体のコストを払う
2. `AF_UNIX SOCK_STREAM` — TCP/IPスタックを完全にスキップするが、データは依然としてカーネルを通してコピーされる
3. 共有メモリ — メッセージごとのカーネル内コピーがまったくない。カーネルが必要なのはマッピングのセットアップと同期のためだけである

`SOCK_DGRAM`の`AF_UNIX`は接続確立のオーバーヘッドを完全に回避するため、同じホスト上のプロセス間での短い送りっぱなしの制御メッセージに有用である。

## カーネルでの実装 {#kernel-implementation}

実装は`net/unix/af_unix.c`にある。主要な構造体:

```c
/* include/net/af_unix.h */
struct unix_sock {
    /* 警告: sk は最初のメンバーでなければならない */
    struct sock     sk;
    struct unix_address *addr;      /* bindされたアドレス */
    struct path     path;           /* ソケットファイルのパス（ファイルシステムソケット） */
    struct mutex    iolock, bindlock;
    struct sock    *peer;           /* 接続先ピア（SOCK_STREAM/SEQPACKET） */
    struct sock    *listener;
    struct unix_vertex *vertex;     /* GCグラフにおけるこのソケットのノード */
    spinlock_t      lock;
    struct socket_wq peer_wq;
    wait_queue_entry_t peer_wake;
    struct scm_stat scm_stat;       /* このソケットのSCM統計 */
    int             inq_len;
    bool            recvmsg_inq;
    bool            scm_rights_notrunc;
    /* struct sk_buff *oob_skb; -- CONFIG_AF_UNIX_OOB の場合のみ */
};
```

`SOCK_DGRAM`と`SOCK_SEQPACKET`のメッセージは、`sk->sk_receive_queue`内に`sk_buff`エントリとして格納される。`SOCK_STREAM`では、`unix_stream_sendmsg()`がデータをピアの受信キューに直接コピーする。

`struct unix_sock`には、ソケットごとの送信中（in-flight）fdカウンタは存在しない（以前のカーネル設計では`struct list_head link`/`atomic_long_t inflight`のペアでこれを追跡していたが、どちらも削除された）。送信中の`SCM_RIGHTS`のアカウンティングは、現在は`user_struct`単位（`user->unix_inflight`）で行われる。ガベージコレクタ（`net/unix/garbage.c`）は、ソケットとそのfd参照の明示的なグラフを構築する。各ソケットは`struct unix_vertex`（上記の`unix_sock.vertex`からリンクされる）、各送信中fd参照はエッジ（`struct unix_edge`）である。そして単純な送信中カウンタのチェックではなく、そのグラフ上でTarjan方式の強連結成分探索を実行し、参照の循環を見つけて解放する。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [net/unix/af_unix.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/unix/af_unix.c) — `AF_UNIX`の中核実装: `unix_stream_sendmsg()`、`unix_dgram_sendmsg()`、`unix_attach_fds()` / `unix_detach_fds()`、`unix_scm_to_skb()`
- [net/unix/garbage.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/unix/garbage.c) — `SCM_RIGHTS`の参照循環を回収するガベージコレクタ（`struct unix_vertex`、`struct unix_edge`）
- [net/core/scm.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/core/scm.c) — 汎用の補助データ処理: `scm_detach_fds()`、`scm_check_creds()`（`SCM_CREDENTIALS`のuid/gidチェック）
- [include/net/af_unix.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/net/af_unix.h) — `struct unix_sock`の定義
- [include/uapi/linux/un.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/un.h) — `struct sockaddr_un`と`UNIX_PATH_MAX`

### manページ {#man-pages}

- [`unix(7)`](https://man7.org/linux/man-pages/man7/unix.7.html) — 正式なリファレンス: ソケットタイプ、`sockaddr_un`、抽象名前空間、`SCM_RIGHTS`、`SCM_CREDENTIALS`、`SO_PEERCRED`
- [`socket(2)`](https://man7.org/linux/man-pages/man2/socket.2.html) — `socket()`システムコールと`SOCK_STREAM` / `SOCK_DGRAM` / `SOCK_SEQPACKET`タイプ
- [`sendmsg(2)`](https://man7.org/linux/man-pages/man2/sendmsg.2.html) — `sendmsg()`/`recvmsg()`、`struct msghdr`、補助データ
- [`cmsg(3)`](https://man7.org/linux/man-pages/man3/cmsg.3.html) — `CMSG_FIRSTHDR`/`CMSG_DATA`/`CMSG_LEN`/`CMSG_SPACE`マクロ

### 関連ページ {#related-pages}

- [パイプとFIFO](pipes.md) — 単方向のバイトストリームIPC
- [共有メモリ](shared-memory.md) — ゼロコピーのデータ交換
- [eventfdとsignalfd](eventfd-signalfd.md) — poll可能な通知fd
- [IPC事件簿](war-stories.md) — 特権分離デーモンで実際に起きた`SCM_RIGHTS`のfdリーク事例
- [ソケット層の概要](../net/socket-layer.md) — `AF_UNIX`が組み込まれる汎用の`struct socket`/`struct sock`/`proto_ops`スタック

### LWN記事 {#lwn-articles}

- [io_uring, SCM_RIGHTS, and reference-count cycles](https://lwn.net/Articles/779472/)（Jonathan Corbet、2019年2月）— `io_uring`のファイル登録が、`AF_UNIX`ガベージコレクタが断ち切るために存在するfd参照カウント循環の新たな変種をどのように生み出したか
