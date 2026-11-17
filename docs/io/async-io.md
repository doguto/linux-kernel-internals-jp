# 非同期I/Oの進化 {#async-io-evolution}

> POSIX AIOからLinux AIO、そしてio_uringへ

## 同期I/Oの問題 {#the-problem-with-synchronous-io}

同期的な`read()`/`write()`は、データが転送されるまで呼び出し元スレッドをブロックする。高スループットなサーバーにとって、これは次のいずれかを意味する:

- 多数のスレッド（それぞれがI/Oでブロックする）— メモリコストが高く、コンテキストスイッチのオーバーヘッドがある
- `epoll`によるノンブロッキングI/O — ソケットには有効だが、**epollは通常ファイルには機能しない**（通常ファイルは常に「ready」である）

非同期I/Oの目標は、I/Oを発行し、ほかの作業を行い、準備ができたら結果を回収することである。

## POSIX AIO（glibcのユーザー空間実装） {#posix-aio-glibc-userspace}

POSIX AIO（`aio_read`、`aio_write`）は標準APIである。しかしLinuxのglibc実装はスレッドを使ってこれを模倣している:

```c
#include <aio.h>

struct aiocb cb = {
    .aio_fildes = fd,
    .aio_buf    = buf,
    .aio_nbytes = 4096,
    .aio_offset = 0,
    /* シグナルで通知: */
    .aio_sigevent = {
        .sigev_notify = SIGEV_SIGNAL,
        .sigev_signo  = SIGUSR1,
    },
};

/* 非同期読み出しを発行 */
aio_read(&cb);

/* ... ほかの作業を行う ... */

/* 完了を確認 */
int err = aio_error(&cb);
if (err == 0) {
    ssize_t n = aio_return(&cb);  /* 転送されたバイト数 */
}

/* または: すべての完了を待つ */
const struct aiocb *list[] = { &cb };
aio_suspend(list, 1, NULL);
```

**問題点**: glibcはPOSIX AIOをスレッドプールで実装している。各`aio_read()`は、`pread()`を同期的に呼び出すワーカースレッドに作業を渡す。これにより次のものが加わる:

- スレッドの生成・破棄のオーバーヘッド
- スレッドごとのメモリ（スタック）
- ワーカースレッドとの間を行き来するコンテキストスイッチ
- カーネルのバイパスはない — スレッドごとに依然としてカーネルを経由する

## LinuxカーネルAIO {#linux-kernel-aio}

Linuxは2.6でネイティブな非同期I/O（`io_submit`、`io_getevents`）を追加した。POSIX AIOと異なり、これはカーネル内で動作し、スレッドを使わない。

```c
#include <linux/aio_abi.h>
#include <sys/syscall.h>

/* セットアップ: AIO コンテキストを作成 */
aio_context_t ctx = 0;
io_setup(128, &ctx);  /* 実行中の操作は最大 128 */

/* iocb を準備 */
struct iocb cb = {
    .aio_lio_opcode = IOCB_CMD_PREAD,
    .aio_fildes     = fd,
    .aio_buf        = (uint64_t)buf,
    .aio_nbytes     = 4096,
    .aio_offset     = 0,
};
struct iocb *cbs[] = { &cb };

/* 発行 */
io_submit(ctx, 1, cbs);

/* 完了を待つ */
struct io_event events[1];
struct timespec timeout = { .tv_sec = 1 };
int n = io_getevents(ctx, 1, 1, events, &timeout);
if (n == 1) {
    printf("result: %ld\n", events[0].res);
}

io_destroy(ctx);
```

### Linux AIOの制限 {#linux-aio-limitations}

Linux AIOは`O_DIRECT` I/O専用に設計された。深刻な制限がある:

| 制限 | 詳細 |
|-----------|---------|
| **O_DIRECTのみ** | バッファードI/Oは`io_submit`を使っても同期的に発行される |
| **ネットワークソケット非対応** | ブロック/ファイルI/Oのみで、ソケットやパイプには使えない |
| **io_submitがブロックしうる** | バッファードファイルの場合や発行リングが満杯の場合 |
| **固定バッファがない** | 操作ごとにユーザーバッファのマップ/アンマップが必要 |
| **チェーンがない** | 「Aを読み、次にBを書く」をユーザー空間との往復なしに表現できない |
| **fsyncがない** | `IOCB_CMD_FSYNC`は存在するが、非同期に動作することはまれ |

結果として、Linux AIOはすでに`O_DIRECT`を使っているデータベースにしか役立たない。一般的な非同期I/Oの問題は解決しない。

### Linux AIOのカーネルパス {#kernel-path-for-linux-aio}

```c
/* fs/aio.c */
long io_submit(aio_context_t ctx_id, long nr, struct iocb __user **iocbpp)
{
    struct kioctx *ctx = lookup_ioctx(ctx_id);

    for (i = 0; i < nr; i++) {
        struct iocb __user *user_iocb = iocbpp[i];
        ret = io_submit_one(ctx, user_iocb, &tmp, compat);
    }
}

static int io_submit_one(struct kioctx *ctx, struct iocb __user *user_iocb, ...)
{
    struct aio_kiocb *req = aio_get_req(ctx);

    switch (iocb.aio_lio_opcode) {
    case IOCB_CMD_PREAD:
        ret = aio_read(&req->rw, &iocb, false, compat);
        /* O_DIRECT の場合: bio を発行してすぐに戻る */
        /* バッファードの場合: ここ io_submit の中でブロックしうる! */
        break;
    case IOCB_CMD_PWRITE:
        ret = aio_write(&req->rw, &iocb, false, compat);
        break;
    }
}
```

## io_uring: 問題を正しく解決する {#io_uring-solving-the-problem-properly}

`io_uring`（Linux 5.1、2019年にマージ）は、Linux AIOの制限をすべて解消する。カーネルとユーザー空間の間で共有メモリ上の2つのリングバッファを使う:

```
Userspace                    Kernel
   │                            │
   │  SQ ring (submissions)     │
   │  ┌─────────────────────┐   │
   │  │ SQE SQE SQE SQE ... │──▶│ io_uring_enter()
   │  └─────────────────────┘   │     ↓
   │                         submits I/O
   │  CQ ring (completions)      │
   │  ┌─────────────────────┐   │
   │  │ CQE CQE CQE CQE ... │◀──│ kernel writes completions
   │  └─────────────────────┘   │
```

ロックは不要である。SQのtailはユーザー空間が、SQのheadはカーネルが書き込む。CQのtailはカーネルが、CQのheadはユーザー空間が書き込む。

### io_uringの基本的な使い方 {#basic-io_uring-usage}

```c
#include <liburing.h>

struct io_uring ring;
io_uring_queue_init(128, &ring, 0);

/* 読み出しを発行 */
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, 4096, offset);
sqe->user_data = 42;  /* 完了を識別するためのタグ */

io_uring_submit(&ring);

/* 完了を待つ */
struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);
printf("read %d bytes (tag=%llu)\n", cqe->res, cqe->user_data);
io_uring_cqe_seen(&ring, cqe);

io_uring_queue_exit(&ring);
```

### io_uringとLinux AIOの比較 {#io_uring-vs-linux-aio}

| 機能 | Linux AIO | io_uring |
|---------|-----------|---------|
| バッファードI/O | 発行時にブロック | 真に非同期 |
| ネットワークソケット | 非対応 | 対応（recv/send/accept/connect） |
| 固定バッファ | 非対応 | 対応（登録後はゼロコピー） |
| リクエストのチェーン | 非対応 | 対応（IOSQE_IO_LINK） |
| ポーリングI/O（IRQなし） | 非対応 | 対応（IORING_SETUP_IOPOLL） |
| カーネルスレッドによる発行 | 非対応 | 対応（IORING_SETUP_SQPOLL） |
| fsync | 部分的 | 対応 |
| splice/tee | 非対応 | 対応 |
| openat/statx/unlinkat | 非対応 | 対応 |
| syscallゼロ | 非対応 | 対応（SQPOLL使用時） |

### io_uring SQPOLL: syscallなしのI/O {#io_uring-sqpoll-zero-syscall-io}

`IORING_SETUP_SQPOLL`を指定すると、カーネルスレッドがSQリングをポーリングする:

```c
struct io_uring_params params = {
    .flags = IORING_SETUP_SQPOLL,
    .sq_thread_idle = 2000,  /* カーネルスレッドがスリープするまでの ms */
};
io_uring_queue_init_params(128, &ring, &params);

/* これで io_uring_submit() は省略可能 — カーネルが新しい SQE をポーリングする */
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, 4096, 0);
io_uring_sqe_set_flags(sqe, IOSQE_FIXED_FILE);

/* カーネルスレッドがまだポーリング中なら syscall は不要: */
io_uring_submit(&ring);  /* syscall なしで戻ることがある */
```

ポーリングキューを持つNVMe（`IORING_SETUP_IOPOLL` + `IORING_SETUP_SQPOLL`）で有用であり、syscallゼロで50〜100µsのレイテンシを実現する。

### io_uringのリクエストチェーン {#io_uring-request-chaining}

```c
/* チェーン: ファイルを読み → ソケットへ書く（読み出しが成功した場合のみ） */
struct io_uring_sqe *read_sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(read_sqe, file_fd, buf, 4096, 0);
io_uring_sqe_set_flags(read_sqe, IOSQE_IO_LINK);  /* 次へリンク */

struct io_uring_sqe *send_sqe = io_uring_get_sqe(&ring);
io_uring_prep_send(send_sqe, sock_fd, buf, 4096, 0);
/* IOSQE_IO_LINK なし: チェーンの終端 */

io_uring_submit(&ring);
/* カーネルは read を実行し、（read が成功すれば）send を実行する */
```

### io_uringのカーネル内部 {#io_uring-kernel-internals}

```c
/* io_uring/io_uring.c */
static int io_read(struct io_kiocb *req, unsigned int issue_flags)
{
    struct io_rw *rw = io_kiocb_to_cmd(req, struct io_rw);
    struct kiocb *kiocb = &rw->kiocb;

    /* まずインラインで試す（ノンブロッキング）: */
    ret = io_iter_do_read(rw, &iter);

    if (ret == -EAGAIN) {
        /* ブロックしそう: io-wq スレッドプールに回す */
        if (issue_flags & IO_URING_F_NONBLOCK)
            return -EAGAIN;
        /* または: 非同期ワーカーへ発行 */
        io_req_task_work_add(req);
    }
    /* データがページキャッシュにあればインラインで完了 */
    io_req_set_res(req, ret, 0);
    return IOU_OK;
}
```

io_uringはまず操作をインライン（ノンブロッキング）で試みる。ブロックしそうな場合は`io-wq`ワーカースレッドにフォールバックする（カーネルスレッドプールに似ているが、POSIX AIOのユーザー空間スレッドより軽量である）。

## ファイルI/Oに対するepollの制限 {#epoll-limitations-for-file-io}

補足として、`epoll`は非同期ファイルI/Oを解決しない:

```c
/* これは通常ファイルには機能しない: */
int fd = open("file.dat", O_RDONLY | O_NONBLOCK);
int epfd = epoll_create1(0);

struct epoll_event ev = { .events = EPOLLIN, .data.fd = fd };
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);

/* epoll_wait はすぐに戻る: ファイルは常に「ready」 */
epoll_wait(epfd, events, 10, -1);
/* データがページキャッシュになければ read() はやはりブロックする */
```

VFSにはページキャッシュミスが解消されたことを通知する仕組みがないため、通常ファイルはepollに対して常にreadyに見える。真のreadiness（準備完了）セマンティクスを持つのはソケットとパイプだけである。

## 適切な非同期I/Oの選び方 {#choosing-the-right-async-io}

| シナリオ | 推奨 |
|----------|---------------|
| O_DIRECTのデータベースI/O | io_uring（レガシーならLinux AIO） |
| ネットワークとファイルI/Oの併用 | io_uring |
| 高スループットなNVMe | io_uring + IOPOLL + SQPOLL |
| 移植性のあるPOSIXコード | POSIX AIO（スレッドのオーバーヘッドを受け入れる） |
| ソケットのみのサーバー | epoll + ノンブロッキングソケット |
| 単純なバッファードファイルI/O | 同期read/write + スレッドプール |

## 参考文献 {#further-reading}

- [Direct I/O](direct-io.md) — O_DIRECT、アライメント要件
- [io_uring: アーキテクチャ](../io-uring/io-uring-arch.md) — リングのセットアップ、SQE/CQEのレイアウト
- [io_uring: 操作](../io-uring/io-uring-ops.md) — 操作の完全なリファレンス
- `io_uring/io_uring.c` — io_uringのコア
- `fs/aio.c` — Linux AIOの実装
- [io_uring.pdf](https://kernel.dk/io_uring.pdf) — Jens Axboeによる設計論文
