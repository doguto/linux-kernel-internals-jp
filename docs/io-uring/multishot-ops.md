# io_uringのマルチショット、リンクされたリクエスト、プローブ {#io_uring-multishot-linked-requests-and-probing}

> 1つのSQEから多数のCQE。永続的な操作、連鎖したI/O、機能の検出

## マルチショットとは何か {#what-multishot-means}

通常のio_uringの投入は1対1のモデルに従う。1つのSQEはちょうど1つのCQEを生成する。SQEは消費され、操作は1回実行され、結果がCQに現れる。

**マルチショット**はこの約束を破る。1つのSQEがカーネル内で有効（armed）なまま残り、明示的にキャンセルされるか致命的なエラーに遭遇するまで、イベントごとに1つずつCQEを生成し続ける。`cqe->flags`の`IORING_CQE_F_MORE`フラグは、同じSQEからまだ完了が来ることを示すシグナルである。

```c
/* cqe->flagsの解釈 */
if (cqe->flags & IORING_CQE_F_MORE) {
    /* SQEはまだ有効。再投入してはならない */
} else {
    /* SQEは終了した。続けたい場合は再投入する */
}
```

`IORING_CQE_F_MORE`がない場合、そのマルチショットSQEは無効化（disarm）されている。これはキャンセル時、特定のエラー時、または下位のリソースがなくなったときに起こる。再開したい場合、アプリケーションは再投入しなければならない。

マルチショットモデルは、接続を受け付けるたび、あるいはメッセージを受信するたびに新しいSQEを再投入する往復をなくす。これは接続レートの高いサーバーにとって決定的に重要である。

## マルチショットaccept（`IORING_OP_ACCEPT`） {#multishot-accept-ioringopaccept}

**追加**: Linux 5.19。`io_uring/net.c`

`IORING_ACCEPT_MULTISHOT`を使うと、1つの`IORING_OP_ACCEPT` SQEが無期限に接続を受け付ける。到着した各接続は、ブロッキングの`accept()`が成功したときに返すのとまったく同じように、新しいクライアントのファイルディスクリプタを`res`フィールドに持つCQEを生成する。

```c
#include <liburing.h>
#include <netinet/in.h>
#include <stdio.h>
#include <string.h>

#define QUEUE_DEPTH 64
#define TAG_ACCEPT  1ULL

int run_accept_loop(int listen_fd)
{
    struct io_uring ring;
    struct io_uring_sqe *sqe;
    struct io_uring_cqe *cqe;
    int ret;

    io_uring_queue_init(QUEUE_DEPTH, &ring, 0);

    /* マルチショットacceptを1つ投入する。キャンセルするまで有効なまま残る */
    sqe = io_uring_get_sqe(&ring);
    io_uring_prep_multishot_accept(sqe, listen_fd,
                                   NULL,   /* addr: 非NULLならCQEごとに埋められる */
                                   NULL,   /* addrlen */
                                   0);     /* accept4()に渡されるフラグ */
    io_uring_sqe_set_data64(sqe, TAG_ACCEPT);
    io_uring_submit(&ring);

    for (;;) {
        ret = io_uring_wait_cqe(&ring, &cqe);
        if (ret < 0) {
            perror("io_uring_wait_cqe");
            break;
        }

        if (io_uring_cqe_get_data64(cqe) == TAG_ACCEPT) {
            bool more = cqe->flags & IORING_CQE_F_MORE;

            if (cqe->res >= 0) {
                int client_fd = cqe->res;
                handle_client(client_fd);   /* 接続ごとのハンドラ */
            } else if (cqe->res != -ECANCELED) {
                fprintf(stderr, "accept error: %s\n", strerror(-cqe->res));
            }

            io_uring_cqe_seen(&ring, cqe);

            if (!more) {
                /* SQEが無効化された。受け付けを続けるために再投入する */
                sqe = io_uring_get_sqe(&ring);
                io_uring_prep_multishot_accept(sqe, listen_fd, NULL, NULL, 0);
                io_uring_sqe_set_data64(sqe, TAG_ACCEPT);
                io_uring_submit(&ring);
            }
        } else {
            /* その他の完了を処理する */
            io_uring_cqe_seen(&ring, cqe);
        }
    }

    io_uring_queue_exit(&ring);
    return 0;
}
```

マルチショットacceptをキャンセルするには、同じ`user_data`を対象とする`IORING_OP_ASYNC_CANCEL`を投入する。

```c
sqe = io_uring_get_sqe(&ring);
io_uring_prep_cancel64(sqe, TAG_ACCEPT, 0);
io_uring_submit(&ring);
/* マルチショットacceptはres == -ECANCELEDの最後のCQEを投稿し、
   IORING_CQE_F_MOREは付かない */
```

カーネルの実装は`io_uring/net.c`（`io_accept_multishot()`）にある。`inet_csk_accept()`が返るたびに、カーネルはCQEを投稿してただちに待機を再設定する。接続と接続の間にユーザー空間への往復は必要ない。

## マルチショットrecv（`IORING_OP_RECV`） {#multishot-recv-ioringoprecv}

**追加**: Linux 5.20 / 6.0。`io_uring/net.c`

`IORING_OP_RECV`に`IORING_RECV_MULTISHOT`を付けると、ソケットからの読み込みを続け、データが到着するたびにCQEを投稿する。カーネルは各メッセージの大きさを事前に知らないため、マルチショットrecvは提供バッファ（`IOSQE_BUFFER_SELECT`）を**必要とする**。カーネルは受信ごとに事前登録されたプールからバッファを選び、CQEの`flags`フィールドが`IORING_CQE_BUFFER_SHIFT`を介してバッファIDを運ぶ。

```c
#define NUM_BUFS   64
#define BUF_SIZE   4096
#define BUF_GID    0

/* ステップ1: バッファリングを割り当てて登録する */
struct io_uring_buf_ring *buf_ring;
void *bufs[NUM_BUFS];
unsigned ring_mask = io_uring_buf_ring_mask(NUM_BUFS);

buf_ring = io_uring_setup_buf_ring(&ring, NUM_BUFS, BUF_GID, 0, &ret);

for (int i = 0; i < NUM_BUFS; i++) {
    bufs[i] = malloc(BUF_SIZE);
    io_uring_buf_ring_add(buf_ring, bufs[i], BUF_SIZE,
                          i, ring_mask, i);
}
io_uring_buf_ring_advance(buf_ring, NUM_BUFS);

/* ステップ2: マルチショットrecvを1つ投入する。到着するすべてのデータにわたって有効なまま残る */
sqe = io_uring_get_sqe(&ring);
io_uring_prep_recv_multishot(sqe, client_fd, NULL, 0, 0);
sqe->buf_group = BUF_GID;
io_uring_sqe_set_flags(sqe, IOSQE_BUFFER_SELECT);
io_uring_sqe_set_data64(sqe, TAG_RECV);
io_uring_submit(&ring);

/* ステップ3: CQEを回収する */
io_uring_wait_cqe(&ring, &cqe);
if (cqe->res > 0) {
    int buf_id = cqe->flags >> IORING_CQE_BUFFER_SHIFT;
    int bytes  = cqe->res;
    process_data(bufs[buf_id], bytes);

    /* バッファをプールに返す */
    io_uring_buf_ring_add(buf_ring, bufs[buf_id], BUF_SIZE,
                          buf_id, ring_mask, 0);
    io_uring_buf_ring_advance(buf_ring, 1);
} else if (cqe->res == 0) {
    /* EOF: 相手が接続を閉じた */
} else {
    fprintf(stderr, "recv error: %s\n", strerror(-cqe->res));
}
bool more = cqe->flags & IORING_CQE_F_MORE;
io_uring_cqe_seen(&ring, cqe);
/* !moreなら: 再投入する */
```

バッファプールが枯渇すると、マルチショットrecvは自動的に無効化され、`res == -ENOBUFS`で`IORING_CQE_F_MORE`のないCQEを投稿する。バッファリングを補充して再投入すること。

## リンクされたリクエスト（`IOSQE_IO_LINK` / `IOSQE_IO_HARDLINK`） {#linked-requests-iosqeiolink-iosqeiohardlink}

リンクされたリクエストを使うと、ステップ間にアプリケーションが一切関与することなく、SQE間の**順序の依存関係**を表現できる。カーネルはチェーンを逐次実行する。SQE Aが完了するまでSQE Bは開始されない。

次のSQEが後に続くべき各SQEに`IOSQE_IO_LINK`を設定する。チェーンの最後のSQEにはリンクフラグを付けない。

### `IOSQE_IO_LINK`: ソフトリンク {#iosqeiolink-soft-link}

リンクされたSQEが失敗する（`res`が0以外になる）と、チェーン内の後続のSQEはすべて`res == -ECANCELED`のCQEを受け取る。チェーンは切断される。

### `IOSQE_IO_HARDLINK`: ハードリンク {#iosqeiohardlink-hard-link}

リンクされたSQEが成功しても失敗しても、チェーンは継続する。各SQEは無条件に順番に実行される。エラーの後でも実行しなければならない後始末のステップが必要な場合に有用である。

### 例: writeしてからfsync {#example-write-then-fsync}

永続的な書き込みのための一般的なパターンは、データを書き込んでからフラッシュすることである。リンクにより、`fsync`はwriteの完了後にのみ実行されることが保証され、両方が1回の`io_uring_submit`呼び出しで投入される。

```c
#include <liburing.h>

int durable_write(struct io_uring *ring, int fd,
                  const void *buf, size_t len, off_t offset)
{
    struct io_uring_sqe *sqe;
    struct io_uring_cqe *cqe;
    int ret;

    /* SQE 1: write。SQE 2にリンクされる */
    sqe = io_uring_get_sqe(ring);
    io_uring_prep_write(sqe, fd, buf, len, offset);
    io_uring_sqe_set_flags(sqe, IOSQE_IO_LINK);
    io_uring_sqe_set_data64(sqe, 1);   /* タグ: write */

    /* SQE 2: fsync。writeが成功した場合のみ実行される */
    sqe = io_uring_get_sqe(ring);
    io_uring_prep_fsync(sqe, fd, 0);   /* 0 = fsync（fdatasyncではない） */
    /* IOSQE_IO_LINKなし: チェーンの終端 */
    io_uring_sqe_set_data64(sqe, 2);   /* タグ: fsync */

    /* 両方を1回のシステムコールで投入する */
    io_uring_submit(ring);

    /* 両方の完了を回収する */
    for (int i = 0; i < 2; i++) {
        ret = io_uring_wait_cqe(ring, &cqe);
        uint64_t tag = io_uring_cqe_get_data64(cqe);
        if (cqe->res < 0) {
            fprintf(stderr, "%s failed: %s\n",
                    tag == 1 ? "write" : "fsync",
                    strerror(-cqe->res));
        }
        io_uring_cqe_seen(ring, cqe);
    }
    return 0;
}
```

writeが失敗すると、fsyncのCQEは`res == -ECANCELED`で届く。fsyncの結果を個別に確認する必要はない。どんな場合でもfsyncを強制したい場合は、`IOSQE_IO_LINK`を`IOSQE_IO_HARDLINK`に置き換える。

### リンクタイムアウト {#link-timeouts}

`IORING_OP_LINK_TIMEOUT` SQEをチェーンに追加すると、その全体の所要時間に上限を設けられる。リンクされた操作がタイムアウト内に完了しなければ、その操作はキャンセルされる。

```c
/* 5秒のタイムアウト付きでconnectする */
sqe = io_uring_get_sqe(ring);
io_uring_prep_connect(sqe, fd, addr, addrlen);
io_uring_sqe_set_flags(sqe, IOSQE_IO_LINK);

sqe = io_uring_get_sqe(ring);
io_uring_prep_link_timeout(sqe, &(struct __kernel_timespec){
    .tv_sec = 5, .tv_nsec = 0
}, 0);
```

## `IORING_OP_POLL_ADD`のマルチショット {#ioringoppoll_add-multishot}

**追加**: Linux 5.13。`io_uring/poll.c`

通常の`IORING_OP_POLL_ADD`は1回発火すると消費される。`IORING_POLL_ADD_MULTI`を追加すると永続的になる。SQEは登録されたまま残り、fdでpoll対象のイベントが発生するたびにCQEが投稿される。

```c
sqe = io_uring_get_sqe(&ring);
io_uring_prep_poll_add(sqe, fd, POLLIN);
sqe->len |= IORING_POLL_ADD_MULTI;   /* マルチショットpoll */
io_uring_sqe_set_data64(sqe, TAG_POLL);
io_uring_submit(&ring);

/* fdが読み込み可能になるたびにCQEが届く:
 *   cqe->res  = 発生したpollイベント（POLLIN、POLLHUP、...）
 *   cqe->flags & IORING_CQE_F_MORE: まだ有効
 */
```

これは、io_uringのイベントループ内でfdのレディネスを永続的に通知するための、`epoll`に代わる手段を提供する。すべてのイベントが同じCQを通るため、`epoll_wait`へのコンテキストスイッチを避けられる。

キャンセルするには:

```c
sqe = io_uring_get_sqe(&ring);
io_uring_prep_poll_remove(sqe, TAG_POLL);
io_uring_submit(&ring);
```

## 機能のプローブ（`io_uring_probe`） {#probing-capabilities-iouringprobe}

io_uringには新しいオペコードや機能が頻繁に追加されており、すべてのカーネルがすべての操作をサポートしているわけではない。サポートされていないオペコードを呼ぶと`EINVAL`または`EOPNOTSUPP`になり、使い方の誤りと区別するのは難しい。プローブAPIを使うと、コードパスを確定する前にカーネルの機能を問い合わせられる。

### `io_uring_register`による生のプローブ {#raw-probe-via-iouringregister}

```c
#include <linux/io_uring.h>
#include <stdlib.h>

struct io_uring_probe *probe_kernel(int ring_fd)
{
    /* probe構造体 + オペコードエントリ分の領域を割り当てる */
    int nr_ops = IORING_OP_LAST;
    struct io_uring_probe *probe = calloc(1,
        sizeof(*probe) + nr_ops * sizeof(struct io_uring_probe_op));

    int ret = io_uring_register(ring_fd,
                                IORING_REGISTER_PROBE,
                                probe,
                                nr_ops);
    if (ret < 0) {
        free(probe);
        return NULL;
    }
    return probe;
}

/* 1つのオペコードを確認する */
bool op_supported(const struct io_uring_probe *probe, unsigned op)
{
    if (op > probe->last_op)
        return false;
    return probe->ops[op].flags & IO_URING_OP_SUPPORTED;
}
```

### liburingのヘルパー {#liburing-helper}

```c
#include <liburing.h>

struct io_uring ring;
io_uring_queue_init(8, &ring, 0);

struct io_uring_probe *probe = io_uring_get_probe_ring(&ring);

if (io_uring_opcode_supported(probe, IORING_OP_ACCEPT)) {
    /* マルチショットacceptは使えるか? カーネルバージョンかprobeのフラグを確認する */
}
if (io_uring_opcode_supported(probe, IORING_OP_SEND_ZC)) {
    printf("zero-copy send supported\n");
}

io_uring_free_probe(probe);
io_uring_queue_exit(&ring);
```

`io_uring_get_probe_ring()`は内部で`io_uring_register(IORING_REGISTER_PROBE)`を呼び、ヒープに割り当てた`io_uring_probe`を返す。`io_uring_free_probe()`で解放する。

### なぜプローブが重要か {#why-probing-matters}

| アプローチ | 操作がない場合の結果 |
|----------|--------------------------|
| 無条件に使う | 実行時に`EINVAL` / `EOPNOTSUPP`となり、診断が難しい |
| バージョンチェック（`uname`） | 脆い: バックポート、ディストリビューションのカーネル |
| 機能フラグのチェック（`params.features`） | リングレベルの機能しかカバーせず、オペコード単位ではない |
| `io_uring_register(IORING_REGISTER_PROBE)` | 実行中のカーネルからのオペコード単位の確定的な答え |

起動時に1回プローブし、結果を保存し、それに基づいて分岐する。プローブのシステムコールは安価で、`io_uring_register`を1回呼ぶだけである。

## まとめ: マルチショットacceptとリンクされた読み込み {#putting-it-together-multishot-accept-with-linked-reads}

マルチショットaccept、接続ごとのマルチショットrecv、ログの永続化のためのリンクされたwrite+fsyncを組み合わせたサーバーの概略:

```
listen_fd
    │
    └─▶ multishot ACCEPT (one SQE, never resubmitted unless disarmed)
              │
              ▼  CQE: cqe->res = client_fd
         per-connection:
              │
              └─▶ multishot RECV + IOSQE_BUFFER_SELECT
                        │
                        ▼  CQE: buf_id = cqe->flags >> IORING_CQE_BUFFER_SHIFT
                   process data
                        │
                        └─▶ linked WRITE → FSYNC (one io_uring_submit)
```

3種類の操作はすべて同じCQに流れ込む。単一のイベントループが`user_data`タグを確認して各CQEを適切なハンドラにディスパッチすることで、それらを回収する。epollもスレッド・パー・コネクションも、イベントごとのシステムコールもない。

## 参考文献 {#further-reading}

- [アーキテクチャとリング](io-uring-arch.md): SQE/CQE構造体、リングレイアウト、`io_uring_enter`
- [操作と高度な機能](io-uring-ops.md): SQPOLL、固定ファイル、提供バッファ
- カーネルツリーの`io_uring/net.c`: `io_accept_multishot()`、`io_recv_multishot()`
- カーネルツリーの`io_uring/poll.c`: `io_poll_multishot()`
- `include/uapi/linux/io_uring.h`: `IORING_ACCEPT_MULTISHOT`、`IORING_RECV_MULTISHOT`、`IORING_POLL_ADD_MULTI`、`IORING_CQE_F_MORE`
- `liburing`のGitHubリポジトリ: `io_uring_prep_multishot_accept()`、`io_uring_get_probe_ring()`
