# io_uringの操作と高度な機能 {#io_uring-operations-and-advanced-features}

> 基本的なread/writeからSQPOLL、リンクされたリクエストまで

## サポートされる操作 {#supported-operations}

io_uringは幅広い操作をサポートしており、そのすべてが非同期である。

### ファイルとソケットのI/O {#file-and-socket-io}

```c
/* 読み書き（pread64/pwrite64と同様） */
IORING_OP_READ        /* fdのオフセット位置から読み込む */
IORING_OP_WRITE       /* fdのオフセット位置に書き込む */
IORING_OP_READV       /* スキャッタ読み込み（struct iovec） */
IORING_OP_WRITEV      /* ギャザー書き込み */
IORING_OP_READ_FIXED  /* 事前登録したバッファに読み込む */
IORING_OP_WRITE_FIXED /* 事前登録したバッファから書き込む */

/* ソケット操作 */
IORING_OP_ACCEPT      /* 接続を受け付ける */
IORING_OP_CONNECT     /* アドレスに接続する */
IORING_OP_SEND        /* ソケットで送信する */
IORING_OP_RECV        /* ソケットから受信する */
IORING_OP_SENDMSG     /* sendmsg（スキャッタ/ギャザー、補助データ） */
IORING_OP_RECVMSG     /* recvmsg */
IORING_OP_SEND_ZC     /* ゼロコピー送信（カーネル6.0以降） */
IORING_OP_SENDMSG_ZC

/* ファイル操作 */
IORING_OP_OPENAT      /* ファイルを開く */
IORING_OP_OPENAT2     /* how構造体付きのopenat2 */
IORING_OP_CLOSE       /* fdを閉じる */
IORING_OP_STATX       /* statx */
IORING_OP_FADVISE     /* posix_fadvise */
IORING_OP_FALLOCATE   /* fallocate */
IORING_OP_FSYNC       /* fsync/fdatasync */
IORING_OP_SYNC_FILE_RANGE

/* ディレクトリ操作 */
IORING_OP_RENAMEAT    /* rename */
IORING_OP_UNLINKAT    /* unlink/rmdir */
IORING_OP_MKDIRAT     /* mkdir */
IORING_OP_LINKAT      /* link */
IORING_OP_SYMLINKAT   /* symlink */

/* splice */
IORING_OP_SPLICE      /* fd間のsplice */
IORING_OP_TEE         /* tee */

/* タイミング */
IORING_OP_TIMEOUT     /* 一定時間待ってからCQEを投稿する */
IORING_OP_TIMEOUT_REMOVE  /* タイムアウトをキャンセルする */
IORING_OP_LINK_TIMEOUT    /* 別のSQEにリンクされたタイムアウト */

/* ポーリング */
IORING_OP_POLL_ADD    /* fdをpollに追加する */
IORING_OP_POLL_REMOVE /* pollを削除する */

/* その他 */
IORING_OP_CANCEL      /* 保留中の操作をキャンセルする */
IORING_OP_ASYNC_CANCEL
IORING_OP_MSG_RING    /* 別のリングにCQEを送る */
IORING_OP_URING_CMD   /* ドライバ固有のコマンド */
IORING_OP_WAITID      /* 非同期のwaitid() */
IORING_OP_SOCKET      /* ソケットを作成する */
IORING_OP_EPOLL_CTL   /* 非同期のepoll_ctl */
IORING_OP_PROVIDE_BUFFERS /* バッファプールを登録する */
IORING_OP_REMOVE_BUFFERS
```

## liburing: 便利なラッパー {#liburing-the-convenience-wrapper}

ほとんどのアプリケーションは、生のシステムコールではなく`liburing`を使う。

```c
#include <liburing.h>

/* 256エントリでリングを初期化する */
struct io_uring ring;
io_uring_queue_init(256, &ring, 0);

/* またはフラグ付きで */
io_uring_queue_init_params(256, &ring, &(struct io_uring_params){
    .flags = IORING_SETUP_SQPOLL | IORING_SETUP_SQ_AFF,
    .sq_thread_cpu = 2,
    .sq_thread_idle = 2000,  /* スリープするまで2秒アイドル */
});
```

### バッチの例: 複数のreadを投入する {#batch-example-submit-multiple-reads}

```c
struct io_uring_sqe *sqe;
struct io_uring_cqe *cqe;

/* 8つのreadを一度に投入する */
for (int i = 0; i < 8; i++) {
    sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, fds[i], bufs[i], BUF_SIZE, 0);
    io_uring_sqe_set_data64(sqe, (uint64_t)i);  /* タグ */
}

/* 1回のシステムコールで8つすべてを投入する */
io_uring_submit(&ring);

/* 完了を回収する */
for (int i = 0; i < 8; i++) {
    io_uring_wait_cqe(&ring, &cqe);
    int idx = (int)io_uring_cqe_get_data64(cqe);
    if (cqe->res < 0)
        fprintf(stderr, "read[%d] error: %s\n", idx, strerror(-cqe->res));
    io_uring_cqe_seen(&ring, cqe);
}
```

## SQPOLL: システムコールを完全になくす {#sqpoll-eliminating-syscalls-entirely}

`IORING_SETUP_SQPOLL`を使うと、専用のカーネルスレッドがSQリングを継続的にポーリングする。ユーザー空間は`io_uring_enter`を呼ぶ必要がなく、SQEを書き込むだけで自動的に拾われる。

```c
io_uring_queue_init_params(256, &ring, &(struct io_uring_params){
    .flags = IORING_SETUP_SQPOLL,
    .sq_thread_idle = 2000,  /* カーネルスレッドは2秒アイドルが続くとスリープする */
});

/* システムコールなしで投入する: */
sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, len, 0);
io_uring_sqe_set_flags(sqe, 0);

/* SQのtailを更新するだけ。io_uring_enter()は不要 */
io_uring_submit(&ring);  /* liburing: SQPOLLを検出し、直接書き込む */

/* カーネルスレッドの起床が必要か確認する */
if (io_sq_ring_needs_enter(ring_ptr))
    io_uring_enter(ring_fd, 0, 0, IORING_ENTER_SQ_WAKEUP, NULL, 0);
```

カーネルのSQPOLLスレッドはCPU（`sq_thread_cpu`）にピン留めされ、通常のスケジューリング優先度（`SCHED_NORMAL`）で動作する。超低レイテンシのI/Oにおいて、システムコールの約1µsのオーバーヘッドを回避できる。

```bash
# SQPOLLスレッドを確認する
ps aux | grep io_uring-sq
# root 1234  99.9  0.0  0  0 [io_uring-sq]
```

## 固定ファイル（登録済みファイルディスクリプタ） {#fixed-files-registered-file-descriptors}

ファイルディスクリプタを登録すると、操作ごとのfd検索がなくなる。

```c
/* 最大4096個のfdをリングに登録する */
int fds[4096] = { fd0, fd1, fd2, -1, -1, ... };
io_uring_register_files(&ring, fds, 4096);

/* fdの代わりに固定ファイルのインデックスを使う */
sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, 0, buf, len, 0);  /* 0 = 固定ファイルのインデックス0 */
io_uring_sqe_set_flags(sqe, IOSQE_FIXED_FILE);

/* すべてを再登録せずに1スロットだけ更新する */
io_uring_register_files_update(&ring, slot_index, &new_fd, 1);

/* 登録解除 */
io_uring_unregister_files(&ring);
```

固定ファイルを使うと、操作のたびに`struct file`への参照を取得せずに済む。これは高IOPSのワークロードでは大きな効果がある。

## 固定バッファ（登録済みバッファ） {#fixed-buffers-registered-buffers}

事前登録したバッファを使うと、I/Oごとのページのピン留めが不要になる。

```c
/* 8個のバッファを登録する */
struct iovec iovecs[8];
for (int i = 0; i < 8; i++) {
    iovecs[i].iov_base = bufs[i];
    iovecs[i].iov_len  = BUF_SIZE;
}
io_uring_register_buffers(&ring, iovecs, 8);

/* バッファインデックスを指定してIORING_OP_READ_FIXEDを使う */
sqe = io_uring_get_sqe(&ring);
io_uring_prep_read_fixed(sqe, fd, bufs[0], BUF_SIZE, offset, 0);
/*                                                              ^ buf_index */

io_uring_unregister_buffers(&ring);
```

固定バッファは登録時に一度だけメモリにピン留めされる。I/Oごとのページのピン留め/解除（ページテーブルのウォークを必要とする）がなくなる。

## リンクされたリクエスト {#linked-requests}

SQEを連鎖させ、前のSQEが成功裏に完了したときにのみ次のSQEが開始されるようにする。

```c
/* 順序: connect → send → recv */
sqe = io_uring_get_sqe(&ring);
io_uring_prep_connect(sqe, fd, addr, addrlen);
io_uring_sqe_set_flags(sqe, IOSQE_IO_LINK);  /* 次にリンクする */

sqe = io_uring_get_sqe(&ring);
io_uring_prep_send(sqe, fd, request, req_len, 0);
io_uring_sqe_set_flags(sqe, IOSQE_IO_LINK);  /* 次にリンクする */

sqe = io_uring_get_sqe(&ring);
io_uring_prep_recv(sqe, fd, response, resp_len, 0);
/* IOSQE_IO_LINKなし: チェーンの終端 */

io_uring_submit(&ring);
/* connectが失敗すると → send/recvは-ECANCELEDのCQEを受け取る */
```

エラーを無視するハードチェーンには、代わりに`IOSQE_IO_HARDLINK`を使う。

## バッファ選択: 提供バッファ {#buffer-selection-provided-buffers}

バッファをSQEに事前に割り当てる代わりに、カーネルにプールから選ばせる（何バイト届くかわからない場合に便利）。

```c
/* バッファグループを登録する */
struct io_uring_buf_reg reg = {
    .ring_addr    = (uint64_t)buf_ring,  /* バッファの共有リング */
    .ring_entries = NUM_BUFS,
    .bgid         = 0,                   /* バッファグループID */
};
io_uring_register_buf_ring(&ring, &reg, 0);

/* グループにバッファを追加する */
for (int i = 0; i < NUM_BUFS; i++) {
    io_uring_buf_ring_add(buf_ring, bufs[i], BUF_SIZE, i, ring_mask, i);
}
io_uring_buf_ring_advance(buf_ring, NUM_BUFS);

/* recvでバッファ選択を使う */
sqe = io_uring_get_sqe(&ring);
io_uring_prep_recv(sqe, fd, NULL, 0, 0);  /* NULL = カーネルがバッファを選ぶ */
sqe->buf_group = 0;
io_uring_sqe_set_flags(sqe, IOSQE_BUFFER_SELECT);

/* どのバッファが使われたかはCQEが教えてくれる */
io_uring_wait_cqe(&ring, &cqe);
int buf_id = cqe->flags >> IORING_CQE_BUFFER_SHIFT;
int bytes = cqe->res;
void *data = bufs[buf_id];
/* data[0..bytes-1]を処理する */
/* バッファをプールに返す: */
io_uring_buf_ring_add(buf_ring, bufs[buf_id], BUF_SIZE, buf_id, ring_mask, 0);
io_uring_buf_ring_advance(buf_ring, 1);
```

## マルチショット操作 {#multishot-operations}

一部の操作は、1つのSQEから複数のCQEを投稿できる。

```c
/* マルチショットaccept: IORING_ACCEPT_MULTISHOTフラグ付きでIORING_OP_ACCEPTを使う */
sqe = io_uring_get_sqe(&ring);
io_uring_prep_multishot_accept(sqe, listen_fd, NULL, NULL, 0);
/* liburingはIORING_OP_ACCEPTにIORING_ACCEPT_MULTISHOTフラグを設定する */

/* 新しい接続ごとに、cqe->resに新しいfdを入れたCQEが投稿される */
/* マルチショットが有効な間はIORING_CQE_F_MOREがセットされる */
while (1) {
    io_uring_wait_cqe(&ring, &cqe);
    if (cqe->res >= 0) {
        int client_fd = cqe->res;
        handle_client(client_fd);
    }
    bool more = cqe->flags & IORING_CQE_F_MORE;
    io_uring_cqe_seen(&ring, cqe);
    if (!more) break;  /* マルチショットが終了した（例: listen_fdが閉じられた） */
}

/* ほかにも利用可能（別個のオペコードではなく、基本オペコードに付けるフラグ）: */
/* IORING_OP_RECV + IORING_RECV_MULTISHOTフラグ: ソケットで継続的にrecvする */
/* IORING_OP_POLL_ADD + IORING_POLL_ADD_MULTIフラグ: fdを繰り返しpollする */
```

## io_uringとBPF {#io_uring-and-bpf}

BPFプログラムをio_uringの操作にアタッチできる。

```c
/* BPFプログラムタイプ: BPF_PROG_TYPE_SYSCALLはio_uring_enterをインターセプトできる */
/* より一般的には: ネットワークパスではsocket_filter経由でio_uring + BPF */

/* io_uringのクレデンシャル: このSQEを別のuid/ケーパビリティで実行する */
sqe->personality = io_uring_register_personality(&ring);
/* SQEは登録されたアイデンティティで実行される */
```

## io_uringの観測 {#observing-io_uring}

```bash
# io_uringインスタンスを一覧表示する
ls /proc/*/fdinfo/ | xargs grep -l "sq entries" 2>/dev/null

# プロセスのio_uring統計
cat /proc/<pid>/fdinfo/<ring_fd>
# pos:     0
# flags:   0100002
# sq entries:     256
# cq entries:     512
# sq head:         42
# sq tail:         43
# sq mask:        255
# sq array off:    128
# cq head:         40
# cq tail:         42
# ...

# システム全体のio_uring統計
cat /proc/sys/fs/io_uring_groups
cat /sys/kernel/debug/io_uring_stats 2>/dev/null

# io_uringのイベントをトレースする
trace-cmd record -e io_uring:* sleep 1
trace-cmd report

# BPFによるトレーシング
bpftrace -e 'tracepoint:io_uring:io_uring_submit_sqe { printf("%s\n", comm); }'
```

## 性能比較 {#performance-comparison}

| インターフェース | セットアップ | 操作ごとのシステムコール | 最大IOPS（NVMe） |
|-----------|-------|----------------|-----------------|
| `read()`/`write()` | なし | あり（操作ごとに1回） | 約300K |
| `libaio` + O_DIRECT | AIOコンテキスト | `io_submit`（バッチ化） | 約700K |
| io_uring（SQPOLLなし） | リングの初期化 | `io_uring_enter`（バッチ化） | 約1.5M |
| io_uring + SQPOLL | リング + スレッド | なし | 約2M以上 |

## 参考文献 {#further-reading}

- [アーキテクチャとリング](io-uring-arch.md): リングレイアウトとSQE/CQE構造体
- [BPF/eBPF](../bpf/README.md): io_uringとBPFの統合
- [ブロック層: blk-mq](../block/blk-mq.md): io_uringの操作が最終的に行き着く場所
- カーネルツリーの`io_uring/`ディレクトリ
- `liburing`のGitHubリポジトリ: 高レベルのC API
