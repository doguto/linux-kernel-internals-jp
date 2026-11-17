# パイプとFIFO {#pipes-and-fifos}

> 単方向のバイトストリームIPC

## パイプとは何か {#what-pipes-are}

パイプは、書き込み側の端と読み出し側の端をつなぐ、カーネル管理の循環バッファである。あるプロセスが書き込んだデータを別のプロセスが読み出す。ファイルシステムは関与しない。

```
Process A               Kernel              Process B
─────────              ────────             ─────────
write(pipefd[1], data)  →  pipe buffer  →  read(pipefd[0], buf)
                             (64KB)
```

特性:

- **単方向**: データは書き込み→読み出しの方向にのみ流れる
- **バイトストリーム**: メッセージ境界がない
- **ブロッキング**: 満杯なら書き込みがブロックし、空なら読み出しがブロックする
- **アトミックな書き込み**: PIPE_BUF（4096）バイト以下の書き込みはアトミックである（混ざり合わない）
- **SIGPIPE**: 読み手のいないパイプへの書き込み → `SIGPIPE` + `EPIPE`

## パイプの作成 {#creating-pipes}

```c
/* 無名パイプ: 2つのfdとして返される */
int pipefd[2];
pipe2(pipefd, O_CLOEXEC | O_NONBLOCK);
/* pipefd[0] = 読み出し端、pipefd[1] = 書き込み端 */

/* 古典的なパイプ */
pipe(pipefd);

/* 典型的な fork + pipe のパターン */
if (fork() == 0) {
    /* 子: パイプから読む */
    close(pipefd[1]);              /* 書き込み端を閉じる */
    read(pipefd[0], buf, sizeof(buf));
    exit(0);
} else {
    /* 親: パイプに書く */
    close(pipefd[0]);              /* 読み出し端を閉じる */
    write(pipefd[1], "hello", 5);
    close(pipefd[1]);              /* 子にEOFを伝える */
    wait(NULL);
}
```

## カーネルのパイプデータ構造 {#kernel-pipe-data-structures}

```c
/* include/linux/pipe_fs_i.h -- fs/pipe.c ではない */
struct pipe_inode_info {
    struct mutex        mutex;          /* パイプを保護する */
    wait_queue_head_t   rd_wait;        /* データを待つ読み手 */
    wait_queue_head_t   wr_wait;        /* 空きを待つ書き手 */

    /* head/tail は独立した2つのフィールドではなく、
     * union にまとめられている（64ビットでは1つの unsigned long の head_tail） */
    union pipe_index {
        unsigned long head_tail;
        struct {
            pipe_index_t head;         /* 生産者インデックス */
            pipe_index_t tail;         /* 消費者インデックス */
        };
    };

    unsigned int        max_usage;      /* バッファスロット数（デフォルト16 = 64KB） */
    unsigned int        ring_size;      /* 2のべき乗のバッファサイズ */
    unsigned int        nr_accounted;   /* アカウンティング用 */
    unsigned int        readers;        /* 読み出し端をopenした数 */
    unsigned int        writers;        /* 書き込み端をopenした数 */
    unsigned int        files;          /* readers + writers の合計 */
    unsigned int        r_counter;      /* POLLHUPチェック用の読み出しカウンタ */
    unsigned int        w_counter;
    bool                pseudo_edgetrigger; /* EPOLLET消費者向けに書き込みごとに起床させる */
    struct anon_pipe_prealloc prealloc; /* 事前割り当てページの小さなプール
                                            （単なる tmp_page ポインタ/配列ではない） */
    struct fasync_struct *fasync_readers;
    struct fasync_struct *fasync_writers;
    struct pipe_buffer  *bufs;          /* pipe_buffer[ring_size] のリング */
    struct user_struct  *user;
    /* このほかに watch_queue ポインタと note_loss フラグがあるが、
     * どちらも CONFIG_WATCH_QUEUE の場合のみ */
};

struct pipe_buffer {
    struct page *page;          /* このデータ塊を保持するページ */
    unsigned int offset;        /* ページ内のバイトオフセット */
    unsigned int len;           /* バイト数 */
    const struct pipe_buf_operations *ops;
    unsigned int flags;         /* PIPE_BUF_FLAG_* */
    unsigned long private;
};
```

パイプバッファは`pipe_buffer`スロットのリングであり、各スロットが1つのページを指す。デフォルトの容量は16ページ × 4096 = 65,536バイトである。

## パイプの読み出し・書き込みパス {#pipe-read-and-write-paths}

```c
/* パイプ書き込みパスの簡略版 -- 実際の関数は anon_pipe_write()（fs/pipe.c） */
static ssize_t anon_pipe_write(struct kiocb *iocb, struct iov_iter *from)
{
    struct pipe_inode_info *pipe = /* ... */;
    size_t total_len = iov_iter_count(from);

    mutex_lock(&pipe->mutex);

    while (total_len > 0) {
        unsigned int head = pipe->head;

        /* パイプが満杯なら待つ */
        if (pipe_full(head, pipe->tail, pipe->max_usage)) {
            mutex_unlock(&pipe->mutex);
            wait_event_interruptible(pipe->wr_wait,
                !pipe_full(pipe->head, pipe->tail, pipe->max_usage));
            mutex_lock(&pipe->mutex);
            continue;
        }

        /* head バッファにデータをコピー */
        struct pipe_buffer *buf = &pipe->bufs[head & (pipe->ring_size-1)];
        /* ... ユーザー空間から buf->page へコピー ... */

        pipe->head = head + 1;
        wake_up_interruptible(&pipe->rd_wait);
    }

    mutex_unlock(&pipe->mutex);
    return written;
}
```

## パイプ容量の変更 {#changing-pipe-capacity}

```c
/* パイプバッファを増やす（高スループットのロギングに有用） */
int new_size = 1 << 20;  /* 1MB */
fcntl(pipefd[1], F_SETPIPE_SZ, new_size);

/* 現在のサイズを問い合わせる */
int size = fcntl(pipefd[1], F_GETPIPE_SZ);
```

```bash
# システム全体のパイプサイズの上限
cat /proc/sys/fs/pipe-max-size  # デフォルト 1MB
echo 4194304 > /proc/sys/fs/pipe-max-size  # 4MBに増やす
```

## splice: ゼロコピーのパイプI/O {#splice-zero-copy-pipe-io}

`splice`は、ユーザー空間へのコピーなしにファイルディスクリプタとパイプの間でデータを移動する:

```c
/* ファイル → パイプへコピー（ゼロコピー読み出し） */
ssize_t n = splice(file_fd, &offset,
                   pipe_fd,  NULL,
                   count, SPLICE_F_MORE);

/* パイプ → ソケットへコピー（ゼロコピー送信） */
n = splice(pipe_fd, NULL,
           socket_fd, NULL,
           count, 0);

/* 典型的なパターン: コピーなしでファイルをネットワークソケットへsplice */
splice(file_fd, NULL, pipe_fd, NULL, file_size, SPLICE_F_MORE);
splice(pipe_fd, NULL, socket_fd, NULL, file_size, 0);
```

`splice`は`pipe_buffer`のページ参照を移動することで動作し、データはコピーされない。これはフラグに関係なくそうである。`SPLICE_F_MOVE`（コピーの代わりにページを移動するヒントで、元の2.6.17の`splice()`と同時に導入されたため後述のLWNの引用でも言及されている）は、Linux 2.6.21以降no-opになっている。最初の実装にバグがあったため、カーネルはこのフラグに基づく動作をやめ、現在の`fs/splice.c`にはこのフラグを読むコードすら存在しない。互換性のために今も受け付けられるが、渡しても何も変わらない。`sendfile`は内部的にこの方法で実装されている。

## vmsplice: ユーザー空間メモリをパイプにマップする {#vmsplice-mapping-userspace-memory-into-a-pipe}

```c
/* ユーザー空間バッファをパイプにマップする（gift: パイプが所有権を得る） */
struct iovec iov = { .iov_base = buf, .iov_len = len };
vmsplice(pipefd[1], &iov, 1, SPLICE_F_GIFT);

/* SPLICE_F_GIFT なし: パイプは参照を得るので、buf を変更してはならない */
vmsplice(pipefd[1], &iov, 1, 0);
```

## tee: パイプデータの複製 {#tee-duplicating-pipe-data}

```c
/* pipe1 から消費せずに pipe1 から pipe2 へデータをコピー */
tee(pipe1_read_fd, pipe2_write_fd, count, SPLICE_F_NONBLOCK);
/* これでデータは pipe1 と pipe2 の両方から読める */
```

`tee(1)`（stdout → ファイル + stdout）を効率的に実装するのに有用である。

## FIFO（名前付きパイプ） {#fifos-named-pipes}

FIFOはファイルシステム経由でアクセスできるパイプである。パスを知っていればどのプロセスでも開ける:

```bash
# FIFOを作成
mkfifo /tmp/myfifo

# プロセスA: 書き込み
echo "hello" > /tmp/myfifo  # 読み手が開くまでブロックする

# プロセスB: 読み出し
cat /tmp/myfifo  # 書き手が開くまでブロックする
```

```c
/* プログラムからFIFOを作成 */
mkfifo("/tmp/myfifo", 0666);

/* ブロックせずに開く（O_NONBLOCK） */
int fd = open("/tmp/myfifo", O_RDONLY | O_NONBLOCK);
/* すぐに戻る。まだ書き手が開いていなくても成功する */

/* 標準のブロッキングopen */
int fd = open("/tmp/myfifo", O_RDONLY);
/* 書き手が書き込み端を開くまでブロックする */
```

openの規則:

- 読み出し専用のopenは、書き手が開くまでブロックする（`O_NONBLOCK`でない限り）
- 書き込み専用のopenは、読み手が開くまでブロックする（`O_NONBLOCK`の場合は`ENXIO`を返す）
- 読み書き両用のopenは決してブロックしない

## パイプとソケットの比較 {#pipe-vs-socket}

| | パイプ/FIFO | Unixソケット（SOCK_STREAM） |
|---|-----------|--------------------------|
| 方向 | 単方向 | 双方向 |
| アドレス | 無名またはパス | パスまたはautobind |
| スキャッタ/ギャザ | なし（writevは使える） | あり（sendmsg） |
| 帯域外データ | なし | あり（MSG_OOB） |
| 資格情報 | なし | あり（SCM_CREDENTIALS） |
| FD受け渡し | なし | あり（SCM_RIGHTS） |
| オーバーヘッド | 低い | やや高い |

## パイプの観察 {#observing-pipes}

```bash
# パイプバッファの使用状況を確認
ls -la /proc/self/fd/
# lrwx 0 -> pipe:[12345678]

# システム内で開かれているすべてのパイプを表示
lsof | grep "^.\{8\}pipe"

# fdinfo にはパイプ固有の "pipe-size" フィールドはなく、汎用の
# pos/flags/mnt_id/ino 行だけがある。現在の容量を読むには F_GETPIPE_SZ を使う。
cat /proc/$(pgrep myproc)/fdinfo/4
# pos:     0
# flags:   0100001
# mnt_id:  12
# ino:     54321

# splice/vmsplice の使用状況を確認
perf trace -e splice,vmsplice -- myprocess
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [fs/pipe.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/pipe.c) — パイプ/FIFOの読み出し・書き込みパス（`anon_pipe_read()`、`anon_pipe_write()`、`fifo_open()`）と、`F_SETPIPE_SZ`/`F_GETPIPE_SZ`の処理
- [include/linux/pipe_fs_i.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/pipe_fs_i.h) — `struct pipe_inode_info`と`struct pipe_buffer`の定義、`PIPE_DEF_BUFFERS`
- [fs/splice.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/splice.c) — `splice()`、`vmsplice()`、`tee()`システムコールの実装
- [include/linux/splice.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/splice.h) — `SPLICE_F_*`フラグの定義（`MOVE`、`NONBLOCK`、`MORE`、`GIFT`）
- [Splice — The Linux Kernel documentation](https://docs.kernel.org/filesystems/splice.html) — カーネル内部のsplice/パイプAPIリファレンス（`pipe_buffer`ヘルパー、`splice_to_pipe()`など）

### manページ {#man-pages}

- [`pipe(2)`](https://man7.org/linux/man-pages/man2/pipe.2.html) — `pipe()`/`pipe2()`による作成、`O_DIRECT`パケットモード
- [`pipe(7)`](https://man7.org/linux/man-pages/man7/pipe.7.html) — パイプ/FIFOの概要、容量、`PIPE_BUF`のアトミック性
- [`fifo(7)`](https://man7.org/linux/man-pages/man7/fifo.7.html) — `O_RDONLY`/`O_WRONLY`/`O_RDWR`に対するFIFOの`open()`のブロッキング意味論
- [`splice(2)`](https://man7.org/linux/man-pages/man2/splice.2.html) — `splice()`のフラグと「少なくとも一方のfdはパイプでなければならない」という要件
- [`vmsplice(2)`](https://man7.org/linux/man-pages/man2/vmsplice.2.html) — `SPLICE_F_GIFT`の意味論
- [`tee(2)`](https://man7.org/linux/man-pages/man2/tee.2.html) — パイプデータを消費せずに複製する
- [`F_SETPIPE_SZ(2const)`](https://man7.org/linux/man-pages/man2/F_SETPIPE_SZ.2const.html) — パイプ容量の変更/問い合わせ、丸め規則、Linux 2.6.35以降

### 関連ページ {#related-pages}

- [splice-sendfile.md](../vfs/splice-sendfile.md) — splice/tee/sendfileのカーネルパスと、`sendfile()`が同じ仕組みの上にどう構築されているかをさらに掘り下げる
- [shared-memory.md](shared-memory.md) — バイトストリームの順序付けを持たない代替手段
- [unix-sockets.md](unix-sockets.md) — 上で比較した、双方向で資格情報の受け渡しができる対応物
- [war-stories.md](war-stories.md) — 「パイプ容量の意外な落とし穴」: 消費者が停滞している間に満杯のパイプで生産者がブロックする
- [file-ops.md](../vfs/file-ops.md) — パイプ/FIFOの`read_iter`/`write_iter`がフックされる`file_operations` vtable

### LWN記事 {#lwn-articles}

- [Two new system calls: splice() and sync_file_range()](https://lwn.net/Articles/178199/) — Jonathan Corbet、2006年4月3日。`splice()`（Larry McVoyによる以前の提案に基づくJens Axboeの2.6.17実装）と`SPLICE_F_MOVE`フラグを紹介している
