# io_uring vs epoll {#io_uring-vs-epoll}

> イベントループを置き換えるべきとき、維持すべきとき、そして移行の方法

## 両者が解決する問題 {#the-problem-both-solve}

現代のサーバーは、スピンもブロックもせずに多数のファイルディスクリプタを同時に待つ必要がある。素朴なアプローチ、つまり接続ごとに1スレッドという方式は、コンテキストスイッチのオーバーヘッドとメモリ（各スレッドにスタックが必要）のために、負荷がかかると破綻する。

Unixの答えは3世代を経て進化してきた。

| インターフェース | 導入 | 仕組み | スケーラビリティの限界 |
|-----------|-----------|-----------|-------------------|
| `select(2)` | BSD 4.2 | ビットマップの走査 | O(maxfd)、fdの上限は1024 |
| `poll(2)` | SVR3 | `pollfd[]`の線形走査 | 呼び出しごとにO(N) |
| `epoll(7)` | Linux 2.5.44 | 赤黒木 + readyリスト | イベントごとにO(1) |
| `io_uring` | Linux 5.1 | 共有メモリのリング | システムコールO(0)（SQPOLL） |

`epoll`は**ネットワークソケット**についてスケーラビリティの問題を解決した。カーネルは関心のあるfdを赤黒木で管理し、readyになったfdを連結リストに入れ、`epoll_wait`がそれを取り出す。待機時にコストがかかるのは*アクティブな*fdだけである。

残された課題:

- epollでは通常ファイルは*常に*readyである。`EPOLLIN`は即座に発火し、実際のデータは後続の`read()`で同期的に取得される。
- すべての操作に依然として少なくとも1回のシステムコールが必要である。登録に`epoll_ctl`、ブロックに`epoll_wait`、そしてI/Oを行うのに`read`/`write`/`accept`。
- 異なる種類の操作にまたがって投入や結果をバッチ化する方法がない。
- カーネル側のポーリングも、ゼロコピーも、統合されたタイマーもない。

`io_uring`は、ネットワークI/O、ファイルI/O、タイマー、キャンセルをカバーする単一の統一されたサブミッション/コンプリーションのインターフェースを追加する。SQPOLLのもとでは、ユーザー空間は共有メモリに書き込むだけで処理を投入できる。システムコールはゼロである。

---

## epollの内部 {#epoll-internals}

### セットアップ {#setup}

```c
/* epollインスタンスを作成する。size引数は2.6.8以降無視されるが、> 0でなければならない */
int epfd = epoll_create1(EPOLL_CLOEXEC);

struct epoll_event ev = {
    .events  = EPOLLIN | EPOLLET,  /* エッジトリガの読み込み関心 */
    .data.fd = client_fd,
};
epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev);
```

```c
/* イベントが届くまで（またはtimeout_msが経過するまで）ブロックする */
struct epoll_event events[MAX_EVENTS];
int n = epoll_wait(epfd, events, MAX_EVENTS, timeout_ms);
for (int i = 0; i < n; i++) {
    handle_fd(events[i].data.fd, events[i].events);
}
```

### カーネルのデータ構造 {#kernel-data-structures}

```
epollインスタンス（struct eventpoll）
  ├─ rbr: 赤黒木
  │    各ノードはstruct epitem（登録されたfdごとに1つ）
  │    epoll_ctl ADD/MOD/DEL = O(log N)の木操作
  │
  └─ rdllist: 双方向連結のreadyリスト
       ソケットが読み込み可能になると、VFSのwqコールバックが発火し、
       epitemをrdllistに載せる。
       epoll_waitはrdllistをユーザー空間のevents[]配列に取り出す。
```

`epoll_wait`のコストは、登録されているfdの数に関係なく、*アクティブな*fdの数に対してO(1)である。これがepollがもたらした重要な洞察である。

### エッジトリガ vs レベルトリガ {#edge-triggered-vs-level-triggered}

- **レベルトリガ（デフォルト）**: データが利用可能である限り、`epoll_wait`はそのfdを返す。標準的なブロッキング読み込みでも安全である。
- **エッジトリガ（`EPOLLET`）**: *状態の変化*（新しいデータの到着）のときにのみ発火する。ユーザー空間はノンブロッキング読み込みでfdを完全に読み切らなければならない。より慎重なバッファリングと引き換えに、不要な起床を減らすために使われる。

### EPOLLONESHOT {#epolloneshot}

`EPOLLONESHOT`は、1回のイベントの後にそのfdを無効化する。ユーザー空間は処理後に`EPOLL_CTL_MOD`で再設定しなければならない。マルチスレッドのサーバーで、2つのスレッドが同じfdで起床するのを避けるための一般的なパターンである。

```c
ev.events = EPOLLIN | EPOLLONESHOT;
epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev);  /* 処理後に再設定する */
```

---

## epollが不十分な点 {#where-epoll-falls-short}

### 通常ファイル {#regular-files}

```c
/* これは常に即座に返る。通常ファイルは常に「ready」である */
ev.events = EPOLLIN;
epoll_ctl(epfd, EPOLL_CTL_ADD, open("big_file", O_RDONLY), &ev);
/* epoll_waitは発火するが、read()はページフォールトのパスで依然としてブロックしうる */
```

カーネルの通常ファイル向けpoll実装は、無条件に`POLLIN | POLLOUT`を返す。epollは「このディスク読み込みが終わったら起こしてくれ」を表現できない。実際のI/Oは、epollイベントの後の`read()`呼び出しの中で同期的に行われる。

### バッチあたりのシステムコールコスト {#syscall-cost-per-batch}

I/Oの往復には、最低でも次が含まれる。

```
epoll_wait()          — システムコール1: ブロック/待機
read() / recv()       — システムコール2: データを取得
write() / send()      — システムコール3: 応答を送信
epoll_ctl(MOD/ADD)    — システムコール4: EPOLLONESHOTなら再設定
```

100 K RPSでは、これは毎秒400 K回以上のシステムコールになる。各システムコールはユーザー/カーネルの境界をまたぎ、TLBの状態を部分的にフラッシュし、緩和策（KPTI、Retpoline）が有効なシステムでは100〜300 nsかかる。

### サブミッションのバッチ化がない {#no-submission-batching}

`epoll_ctl`は一度に1つのfdしか受け取らない。起動時に1000個のfdを登録するには、1000回の個別のシステムコールが必要になる。

### 統合されたタイマーがない {#no-integrated-timers}

タイムアウトには、別途`timerfd_create` + `EPOLL_CTL_ADD`を使うか、`epoll_wait`のタイムアウト引数を注意深く使う必要がある。どちらも操作単位の期限モデルにはきれいに統合されない。

### キャンセルがない {#no-cancellation}

保留中の`epoll_wait`をキャンセルするには、fdを閉じるかシグナルを送る以外に方法がない。

---

## io_uringの統一モデル {#io_urings-unified-model}

io_uringは、*すべての*非同期操作に対して1つのリングを提供する。同じサブミッションパスが次を扱う。

```
Network:  IORING_OP_ACCEPT, IORING_OP_RECV, IORING_OP_SEND
Files:    IORING_OP_READ, IORING_OP_WRITE, IORING_OP_FSYNC
Poll:     IORING_OP_POLL_ADD, IORING_OP_POLL_REMOVE
Timers:   IORING_OP_TIMEOUT, IORING_OP_TIMEOUT_REMOVE
Other:    IORING_OP_CONNECT, IORING_OP_OPENAT, IORING_OP_CLOSE,
          IORING_OP_STATX, IORING_OP_SPLICE, IORING_OP_SEND_ZC
```

### IORING_OP_POLL_ADD: epoll_waitの置き換え {#ioring_op_poll_add-replacing-epoll_wait}

```c
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_poll_add(sqe, fd, POLLIN);
sqe->user_data = (uintptr_t)conn;  /* コンテキストポインタ */
io_uring_submit(&ring);

/* 後で完了を回収する */
struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);
if (cqe->res & POLLIN) {
    handle_readable((struct conn *)cqe->user_data);
}
io_uring_cqe_seen(&ring, cqe);
```

デフォルトではワンショットである。`EPOLLONESHOT`と同じく、再設定するにはSQEを再投入する。

### IORING_POLL_ADD_MULTI: 永続的なfdの監視 {#ioring_poll_add_multi-persistent-fd-monitoring}

```c
sqe->len = IORING_POLL_ADD_MULTI;  /* マルチショット: 自動キャンセルしない */
```

`IORING_POLL_ADD_MULTI`を使うと、1回の投入で、fdがreadyになるたびにCQEを生成し続ける。再設定不要のレベルトリガのepollに相当する。`IORING_OP_POLL_REMOVE`でキャンセルする。

```c
/* マルチショットpollをキャンセルする */
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_poll_remove(sqe, user_data_tag);
io_uring_submit(&ring);
```

### パイプライン化: 1つのリングでACCEPT + RECV {#pipelining-accept-recv-in-one-ring}

リンクされたSQE（`IOSQE_IO_LINK`）を使うと、一連の操作がステップ間でユーザー空間に戻ることなく順番に完了する。

```c
/* SQE 0: accept */
sqe = io_uring_get_sqe(&ring);
io_uring_prep_multishot_accept(sqe, listen_fd, NULL, NULL, 0);
sqe->user_data = TAG_ACCEPT;

/* SQE 1: recv（SQE 0のIOSQE_IO_LINKによりacceptにリンクされる） */
sqe = io_uring_get_sqe(&ring);
io_uring_prep_recv(sqe, IORING_FILE_INDEX_ALLOC, buf, sizeof(buf), 0);
sqe->flags |= IOSQE_FIXED_FILE;
sqe->user_data = TAG_RECV;

io_uring_submit(&ring);
```

### スレッドなしのファイルI/O {#file-io-without-threads}

```c
/* 通常ファイルの読み込み。O_NONBLOCKによるごまかしではなく、本当に非同期 */
sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, file_fd, buf, 4096, offset);
sqe->user_data = TAG_FILE_READ;
io_uring_submit(&ring);
/* このfdに対するepoll_waitなら即座に「ready」を返し、
   その後read()でブロックしていただろう。io_uringは実際のI/Oを非同期に行う。 */
```

---

## 機能比較 {#feature-comparison}

| 機能 | epoll | io_uring |
|---------|-------|---------|
| ネットワークソケットのレディネス | 可 | 可（`IORING_OP_POLL_ADD`） |
| ネットワークソケットのI/O | イベント後に`recv`/`send`で | `IORING_OP_RECV` / `IORING_OP_SEND` |
| 通常ファイルのI/O | 不可（常にready） | 可（`IORING_OP_READ` / `IORING_OP_WRITE`） |
| パイプとFIFO | 可 | 可 |
| `eventfd` / `timerfd` | 可 | 可（`IORING_OP_POLL_ADD`） |
| ゼロコピー送信 | 不可 | 可（`IORING_OP_SEND_ZC`、カーネル6.0以上） |
| カーネル側のポーリング | 不可 | 可（`IORING_SETUP_SQPOLL`） |
| バッチ化された投入 | 不可（`epoll_ctl`ごとに1つのfd） | 可（リングを埋めて1回投入） |
| バッチ化された完了の回収 | 可（`epoll_wait`がN個返す） | 可（CQリングを読み切る） |
| リンク/連鎖した操作 | 不可 | 可（`IOSQE_IO_LINK`、`IOSQE_IO_HARDLINK`） |
| マルチショット（永続的） | 可（デフォルトのレベルトリガ） | 可（`IORING_POLL_ADD_MULTI`、マルチショットaccept/recv） |
| 統合されたタイムアウト | 部分的（`timerfd`による回避策） | 可（`IORING_OP_TIMEOUT`、タイムアウトをリンクした操作） |
| キャンセル | 不可 | 可（`IORING_OP_CANCEL`） |
| 固定バッファ / 登録済みfd | 不可 | 可（`IORING_REGISTER_BUFFERS` / `IORING_REGISTER_FILES`） |
| POSIXの移植性 | Linuxのみ | Linuxのみ（5.1以降） |
| BSD / macOSでの相当機能 | `kqueue` | 相当するものなし |
| 最低カーネルバージョン | 2.5.44 | 5.1（実用的には5.10以降） |
| ユーザー空間ライブラリ | 不要 | `liburing`（強く推奨） |

---

## システムコールのオーバーヘッド {#syscall-overhead}

### epollのパス（リクエストごと、エコーサーバーの例） {#epoll-path-per-request-echo-server-example}

```
クライアントがデータを送信:
  1. epoll_wait()             ← EPOLLINで起床
  2. recv(fd, buf, len, 0)    ← データを読む
  3. send(fd, buf, len, 0)    ← 応答を書く
  [4. epoll_ctl(MOD, ...)     ← EPOLLONESHOTなら再設定]

= リクエストあたり3〜4回のシステムコール
```

### io_uringのパス（SQPOLLなし） {#io_uring-path-without-sqpoll}

```
クライアントがデータを送信:
  1. io_uring_enter()         ← RECV + SEND SQEを投入し、CQEを待つ
                                 （またはio_uring_submit + io_uring_wait_cqe）

= N個のリクエストのバッチあたり1回のシステムコール
```

### io_uringのパス（SQPOLLあり） {#io_uring-path-with-sqpoll}

```
クライアントがデータを送信:
  （システムコールなし。カーネルのSQPOLLスレッドが共有メモリ内の新しいSQEを見つける）
  完了がCQリングに現れる
  ユーザー空間はCQリングを直接読む

= リクエストあたり0回のシステムコール（リングがアクティブな間）
```

### 実環境での数値 {#numbers-from-the-wild}

高接続数のワークロードでのベンチマーク（出典: io_uring作者の講演、Cloudflareのブログ、Nginxのio_uring実験）:

- 100 K接続では、epollサーバーは通常、CPU時間の15〜25%をシステムコールのオーバーヘッドだけに費やす。
- SQPOLL付きのio_uringは、エコーサーバーのマイクロベンチマークでシステムコール数を約80%削減する。
- バッチ化した`io_uring_enter`（32個のSQEを投入し、32個のCQEを待つ）により、現代のCPUでは操作あたりのシステムコールコストが10 ns未満になる。
- ゼロコピー送信（`IORING_OP_SEND_ZC`）は、大きなペイロードでのユーザー→カーネルのバッファコピーをなくし、4 KBを超えるメッセージではさらに10〜15%のスループット向上をもたらす。

---

## 並べて比較: エコーサーバー {#side-by-side-echo-server}

### epoll版 {#epoll-version}

```c
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>

#define MAX_EVENTS 64
#define BUF_SIZE   4096

static void set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main(void) {
    int listen_fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, 0);
    int opt = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port   = htons(8080),
        .sin_addr   = { .s_addr = INADDR_ANY },
    };
    bind(listen_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(listen_fd, 128);

    int epfd = epoll_create1(EPOLL_CLOEXEC);

    /* listenソケットを登録する */
    struct epoll_event ev = { .events = EPOLLIN, .data.fd = listen_fd };
    epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);

    struct epoll_event events[MAX_EVENTS];
    char buf[BUF_SIZE];

    for (;;) {
        int n = epoll_wait(epfd, events, MAX_EVENTS, -1); /* システムコール1 */
        for (int i = 0; i < n; i++) {
            int fd = events[i].data.fd;

            if (fd == listen_fd) {
                /* 新しい接続を受け付ける */
                int conn = accept4(listen_fd, NULL, NULL,
                                   SOCK_NONBLOCK | SOCK_CLOEXEC); /* システムコール2 */
                ev.events  = EPOLLIN | EPOLLET;
                ev.data.fd = conn;
                epoll_ctl(epfd, EPOLL_CTL_ADD, conn, &ev); /* システムコール3 */
            } else {
                /* 読み込んでエコーする */
                ssize_t r;
                while ((r = recv(fd, buf, sizeof(buf), 0)) > 0) { /* システムコール4 */
                    send(fd, buf, r, 0);                           /* システムコール5 */
                }
                if (r == 0 || (r < 0 && errno != EAGAIN)) {
                    epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);      /* システムコール6 */
                    close(fd);                                      /* システムコール7 */
                }
            }
        }
    }
}
```

### io_uring版 {#io_uring-version}

```c
#include <liburing.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>
#include <string.h>

#define QUEUE_DEPTH  256
#define BUF_SIZE     4096

/* CQEの種類を識別するためにuser_dataにエンコードするタグ */
#define TAG_ACCEPT  0x0100000000000000ULL
#define TAG_RECV    0x0200000000000000ULL
#define TAG_SEND    0x0300000000000000ULL

struct conn {
    int  fd;
    char buf[BUF_SIZE];
};

static void submit_accept(struct io_uring *ring, int listen_fd) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
    /* マルチショット: 新しい接続ごとにCQEを生成し続ける */
    io_uring_prep_multishot_accept(sqe, listen_fd, NULL, NULL, 0);
    sqe->user_data = TAG_ACCEPT;
}

static void submit_recv(struct io_uring *ring, struct conn *c) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
    io_uring_prep_recv(sqe, c->fd, c->buf, BUF_SIZE, 0);
    sqe->user_data = TAG_RECV | (uintptr_t)c;
}

static void submit_send(struct io_uring *ring, struct conn *c, int len) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
    io_uring_prep_send(sqe, c->fd, c->buf, len, 0);
    sqe->user_data = TAG_SEND | (uintptr_t)c;
}

int main(void) {
    int listen_fd = socket(AF_INET, SOCK_STREAM | SOCK_CLOEXEC, 0);
    int opt = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port   = htons(8080),
        .sin_addr   = { .s_addr = INADDR_ANY },
    };
    bind(listen_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(listen_fd, 128);

    struct io_uring ring;
    io_uring_queue_init(QUEUE_DEPTH, &ring, 0);

    /* マルチショットacceptを一度だけ投入する。再設定は不要 */
    submit_accept(&ring, listen_fd);
    io_uring_submit(&ring); /* 1回限りのセットアップ用システムコール */

    for (;;) {
        struct io_uring_cqe *cqe;
        io_uring_wait_cqe(&ring, &cqe); /* すべてのイベントに対して1回のシステムコール */

        uint64_t tag  = cqe->user_data & 0xff00000000000000ULL;
        void    *ptr  = (void *)(uintptr_t)(cqe->user_data & ~0xff00000000000000ULL);

        if (tag == TAG_ACCEPT) {
            if (cqe->res >= 0) {
                /* 新しい接続のfdはcqe->resにある */
                struct conn *c = calloc(1, sizeof(*c));
                c->fd = cqe->res;
                submit_recv(&ring, c); /* すぐにRECVを投稿する */
                io_uring_submit(&ring);
            }
            /* マルチショット: accept SQEは有効なまま。再設定不要 */

        } else if (tag == TAG_RECV) {
            struct conn *c = ptr;
            if (cqe->res > 0) {
                submit_send(&ring, c, cqe->res); /* エコーバックする */
                io_uring_submit(&ring);
            } else {
                close(c->fd);
                free(c);
            }

        } else if (tag == TAG_SEND) {
            struct conn *c = ptr;
            if (cqe->res > 0) {
                submit_recv(&ring, c); /* 次のメッセージを待つ */
                io_uring_submit(&ring);
            } else {
                close(c->fd);
                free(c);
            }
        }

        io_uring_cqe_seen(&ring, cqe);
    }
}
```

2つの版の主な違い:

- io_uring版は`epoll_ctl`、`recv`、`send`を直接呼ばない。それらはSQEとして表現される。
- `multishot_accept`は再設定なしで新しい接続ごとにCQEを発火する。同等のepollコードでは、ループ内で`accept4`を呼び、さらにクライアントごとに`epoll_ctl ADD`を呼ぶ必要がある。
- 高スループット時には、次の投入の前に1回の`io_uring_wait_cqe`ループで複数のCQEを読み切れるため、投入のシステムコールのコストが多数の完了に分散される。

---

## 移行パターン {#migration-patterns}

### ステップ1: `epoll_wait`をCQEの読み切りに置き換える {#step-1-replace-epoll_wait-with-cqe-drain}

```c
/* 移行前 */
int n = epoll_wait(epfd, events, MAX, -1);
for (int i = 0; i < n; i++) { handle(events[i]); }

/* 移行後 */
struct io_uring_cqe *cqe;
unsigned head;
io_uring_for_each_cqe(&ring, head, cqe) {
    handle_cqe(cqe);
}
io_uring_cq_advance(&ring, count);  /* headをまとめて進める */
```

### ステップ2: `epoll_ctl ADD`を`IORING_OP_POLL_ADD`に置き換える {#step-2-replace-epoll_ctl-add-with-ioring_op_poll_add}

```c
/* 移行前 */
struct epoll_event ev = { .events = EPOLLIN, .data.fd = fd };
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);

/* 移行後 */
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_poll_add(sqe, fd, POLLIN);
sqe->user_data = (uintptr_t)conn;
/* 他のSQEとバッチ化し、io_uring_submitを1回呼ぶ */
```

### ステップ3: イベント後の`recv`を`IORING_OP_RECV`に置き換える {#step-3-replace-post-event-recv-with-ioring_op_recv}

```c
/* 移行前（2段階: epollがreadyを通知し、その後recv） */
/* epoll_waitが発火する */
ssize_t r = recv(fd, buf, len, 0);

/* 移行後（1段階: RECVを投入し、結果をCQEで受け取る） */
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_recv(sqe, fd, buf, len, 0);
sqe->user_data = TAG_RECV | ctx;
/* 結果はcqe->resとして届く。別途recv()を呼ぶ必要はない */
```

### ステップ4: 必要に応じてSQPOLLを有効にする {#step-4-optionally-enable-sqpoll}

```c
struct io_uring_params params = {
    .flags          = IORING_SETUP_SQPOLL,
    .sq_thread_idle = 10000, /* SQPOLLスレッドがスリープするまでのミリ秒 */
};
io_uring_queue_init_params(QUEUE_DEPTH, &ring, &params);
/* これ以降、io_uring_submit()は任意。呼ぶ前に
   sq_ring->flagsのIORING_SQ_NEED_WAKEUPを確認する */
```

5.11未満のカーネルでは`CAP_SYS_NICE`が必要であるか、`IORING_SETUP_SQPOLL` + 特権プロセスが必要である。5.11以降では、非特権のSQPOLLが許可される。

### 段階的な移行戦略 {#incremental-migration-strategy}

```
フェーズ1: epollと並行してリングを導入する
  ├─ 起動時にio_uringリングを作成する
  ├─ 新しいファイルI/Oをio_uring経由にする（pread/pwriteスレッドを置き換える）
  └─ ソケットイベントには既存のepollループを維持する

フェーズ2: ソケットイベントを移行する
  ├─ 新しい接続に対してEPOLL_CTL_ADDをIORING_OP_POLL_ADDに置き換える
  ├─ epollイベント後のrecv/sendをIORING_OP_RECV / IORING_OP_SENDに置き換える
  └─ fd数がゼロになるにつれてepollインスタンスを縮小する

フェーズ3: epollを完全に取り除く
  ├─ 残りのepoll_ctl呼び出しを置き換える
  └─ 正しさが確認できたら、必要に応じてSQPOLLを有効にする
```

---

## epollを維持すべきとき {#when-to-keep-epoll}

**移植性が必要な場合**。epollはLinux専用だが、2.5.44から存在している。コードベースがFreeBSD、macOS、OpenBSD（`kqueue`が相当機能）でもコンパイルできなければならないなら、Linuxではepoll、それ以外ではkqueueを使う抽象化レイヤー（libevent、libuv、uv__io_t）のほうがio_uringより適している。

**カーネルバージョンの制約**。io_uringはカーネルバージョンを重ねて安定してきた。

```
5.1   — 最初のリリース（不完全、多数のバグ）
5.6   — IORING_OP_SPLICE、固定バッファが安定
5.10  — IOSQE_BUFFER_SELECT、安定性の良いベースライン
5.11  — 非特権のSQPOLL
5.19  — マルチショットrecv、send_zcのプロトタイプ
6.0   — IORING_OP_SEND_ZCが安定
6.1   — マルチショットrecvが安定
```

デプロイ先がRHEL 8（カーネル4.18）や5.10より古いカーネルの場合、io_uringは利用できないか、重要な機能が欠けている。

**小さく単純なイベントループ**。10個のソケットを監視し、毎秒数千リクエストを処理するシングルスレッドのデーモンは、io_uringのバッチ化の仕組みから何も得られない。追加される複雑さに見合わない。

**`eventfd` / `signalfd` / `timerfd`を多用する場合**。これらはすべてepollの正当な監視対象である。io_uringでもそれらに`POLL_ADD`できるが、既存のepollとの統合は成熟しており、よく理解されている。ワークロードがすでにこれらの特殊なfdに支配されているなら、移行による利得はごくわずかである。

**監査とデバッグのツール**。strace、perf、BPFベースの可観測性フックなどのツールには、何十年にもわたるepollのトレーシングのサポートがある。io_uringのシステムコールのパターンは大きく異なるため、素朴な`epoll_wait`ベースのプロファイリングは機能しなくなる。`io_uring_register(IORING_REGISTER_ENABLE_RINGS)`や`io_uring_enter`のBPFトレーシングは機能するが、更新されたツールが必要である。

**成熟したepollロジックを持つ既存の大規模コードベース**。イベントループが本番システム（Nginx、HAProxy、Redis）に深く組み込まれている場合、特定のボトルネックが計測されていない限り、移行と再テストのコストが利得を上回る。

---

## io_uringが勝るとき {#when-io_uring-wins}

**新しい高性能ネットワークサーバー**。ゼロから始めてLinux 5.10以降をターゲットにするなら、io_uringが正しいデフォルトである。システムコールの節約は本物であり、規模が大きくなるほど効いてくる。

**50 Kを超える同時接続**。この規模では、epollのイベントごとのシステムコールコストが計測可能になる。io_uringのバッチ化された投入/完了ループは、そのコストをシステムコールあたり数百のイベントに分散する。

**ファイルとネットワークが混在するI/O**。同じループの中でファイルを読み、ネットワーククライアントに応答するデータベース、オブジェクトストア、プロキシが最も恩恵を受ける。epollでは、ファイルI/Oにはスレッドプール（ファイルは常に「ready」なので）かLinux AIO（`libaio`）が必要で、どちらも統合性が劣る。io_uringは両方を同一のSQE/CQEセマンティクスで扱う。

**データベースにおけるLinux AIOの置き換え**。`libaio`は`O_DIRECT`の読み書きしか扱えず、64 KBのバッファアラインメントという固定の要件があり、バッチ化が最適でない`io_submit`/`io_getevents`を使う。io_uringはこれに取って代わる。PostgreSQL、RocksDB、ScyllaDBはいずれもio_uringのパスに移行済みか、移行中である。

**レイテンシが決定的に重要なパス**。SQPOLLと登録済みのfd/バッファを使えば、データパス全体を共有メモリ上に置ける。ホットパスでシステムコールによる境界越えがない。カーネル内SQPOLLを使うワークロードでは、同等のepollサーバーと比べてp99レイテンシが20〜40%改善したと報告されている。

**大きなペイロードのゼロコピー送信**。`IORING_OP_SEND_ZC`は、`send`でのユーザー空間→カーネルのバッファコピーを回避する。4 KB以上のペイロード（典型的なHTTPレスポンスボディ、ストリーミングメディア）では、送信ごとの大きなmemcpyがなくなる。

**I/Oのスレッドモデルを単純にしたい場合**。epollベースのサーバーは、イベントループのスレッドとは別に、ブロッキングなファイルI/O専用のスレッドプールを持つことが多い。io_uringは両方を1つのリングに統一し、スレッドプールとイベントループ間の同期オーバーヘッドをなくす。

---

## `io_uring_prep_poll_add`と`IORING_POLL_ADD_MULTI`の詳細 {#io_uring_prep_poll_add-and-ioring_poll_add_multi-in-depth}

最小限の移行、つまり「io_uringをより良いepollとして使う」ことを望むチームにとって、`IORING_OP_POLL_ADD`が入り口になる。

### ワンショットpoll（epollのEPOLLONESHOTに相当） {#one-shot-poll-epoll-epolloneshot-equivalent}

```c
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_poll_add(sqe, fd, POLLIN | POLLRDHUP);
io_uring_sqe_set_data(sqe, conn);

io_uring_submit(&ring);

/* fdが読み込み可能になるとCQEが届く */
struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);
struct conn *c = io_uring_cqe_get_data(cqe);
/* cqe->resには発火したpollマスクが入っている */
io_uring_cqe_seen(&ring, cqe);

/* fdを再び監視するには再投入しなければならない（EPOLLONESHOT + EPOLL_CTL_MODと同様） */
```

### マルチショットpoll（epollのレベルトリガに相当） {#multishot-poll-epoll-level-triggered-equivalent}

```c
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_poll_multishot(sqe, fd, POLLIN);
io_uring_sqe_set_data(sqe, conn);
io_uring_submit(&ring);

/* fdがreadyである限りCQEが届き続ける。
   cqe->flagsのIORING_CQE_F_MOREを確認する: */
struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);
if (!(cqe->flags & IORING_CQE_F_MORE)) {
    /* pollがキャンセルされたかエラー。必要なら再投入する */
}
io_uring_cqe_seen(&ring, cqe);
```

`IORING_CQE_F_MORE`がセットされていれば、マルチショットpollはカーネル内でまだ有効である。これがない場合（キャンセル、エラー、または`IORING_OP_POLL_REMOVE`）、そのfdはもう監視されていない。

### マルチショットpollのキャンセル {#cancelling-a-multishot-poll}

```c
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_poll_remove(sqe, conn);  /* user_dataで照合する */
io_uring_submit(&ring);
/* res == 0のCQEがキャンセルを確認する */
```

### pollマスクのリファレンス {#poll-mask-reference}

```c
POLLIN      — 読み込み可能なデータがある          （EPOLLINに相当）
POLLOUT     — 書き込み可能な空きがある            （EPOLLOUTに相当）
POLLRDHUP   — 相手が書き込み側を閉じた            （EPOLLRDHUPに相当）
POLLHUP     — ハングアップ（接続が閉じた）        （EPOLLHUPに相当）
POLLERR     — エラー状態                          （EPOLLERRに相当）
POLLPRI     — 緊急 / 帯域外データ                 （EPOLLPRIに相当）
```

---

## 参考文献 {#further-reading}

- [io_uringのアーキテクチャとリング](io-uring-arch.md): SQ/CQリングのレイアウト、SQE/CQE構造体、サブミッションとコンプリーションの流れ
- [io_uringの操作と高度な機能](io-uring-ops.md): 全操作の一覧、SQPOLL、固定バッファ、登録済みファイル
- `liburing`のソース: `src/`には、上で説明したSQEフィールドに直接対応する`io_uring_prep_*`ヘルパーがある
- カーネルツリーの`io_uring/poll.c`: `IORING_OP_POLL_ADD`とマルチショットのロジックの実装
- Jens Axboeのio_uringノート: `kernel.dk/io_uring.pdf`（最初の設計文書）
- `man 7 epoll`、`man 2 epoll_ctl`、`man 2 epoll_wait`: epollのリファレンス
