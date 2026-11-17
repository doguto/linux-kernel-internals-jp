# splice、sendfile、copy_file_range {#splice-sendfile-and-copy_file_range}

> ファイルディスクリプタ間のゼロコピーデータ転送

## 問題: データ移動のオーバーヘッド {#the-problem-data-movement-overhead}

通常の`read()` + `write()`は、データをユーザー空間経由でコピーする:

```
Disk → kernel page cache → [copy] → userspace buffer → [copy] → kernel socket buffer → NIC
                                         ↑
                               Two copies through CPU
```

大量のデータ転送（HTTPでのファイル配信、プロセス間でのデータのパイプ処理）では、これらのコピーは無駄である。Linuxはいくつかのゼロコピーの代替手段を提供している。

## sendfile: ファイルからソケットへのゼロコピー {#sendfile-file-to-socket-zero-copy}

`sendfile()`はLinux 2.2で導入された[(man page)](https://www.man7.org/linux/man-pages/man2/sendfile.2.html)。ユーザー空間を経由してコピーすることなく、ファイルを直接ソケットへ送る:

```c
#include <sys/sendfile.h>

ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);

/* 例: HTTPでファイルを配信する */
int file_fd = open("file.dat", O_RDONLY);
int sock_fd = accept(listen_fd, NULL, NULL);

off_t offset = 0;
ssize_t sent = sendfile(sock_fd, file_fd, &offset, file_size);
```

sendfileでのデータ経路:
```
Disk → page cache → [DMA] → NIC (with scatter-gather DMA)
              ↑
    No copy through userspace
    No copy through socket buffer (with NIC scatter-gather)
```

### sendfileの制限 {#sendfile-limitations}

- `out_fd`はソケットでなければならない（一部のカーネルではパイプも可）
- `in_fd`は`mmap`のような操作をサポートしていなければならない（通常ファイル。ソケットは不可）
- ファイルからファイルへのコピーには使えない
- 真のゼロコピーにはNICのscatter-gatherが必要。そうでない場合はカーネル→カーネルのコピーが1回発生する

## splice: パイプベースのゼロコピー {#splice-pipe-based-zero-copy}

`splice()`は、Larry McVoyによる以前の構想に基づき、Jens AxboeによってLinux 2.6.17で導入された[(commit)](https://git.kernel.org/linus/5274f052e7b3dbd81935772eb551dfd0325dfa9d) [(LWN)](https://lwn.net/Articles/178199/)。ページ参照を使ってパイプとファイルディスクリプタの間でデータを移動する。データのコピーは発生しない:

```c
#define _GNU_SOURCE
#include <fcntl.h>

ssize_t splice(int fd_in, loff_t *off_in,
               int fd_out, loff_t *off_out,
               size_t len, unsigned int flags);
/* flags: SPLICE_F_MOVE, SPLICE_F_NONBLOCK, SPLICE_F_MORE */
```

`fd_in`と`fd_out`の少なくとも一方はパイプでなければならない。パイプは中継用のバッファとして働く:

```c
/* パイプ経由でファイルからソケットへ（ゼロコピー） */
int pipefd[2];
pipe(pipefd);

/* 段階1: ファイル → パイプ（ページ参照を移動、コピーなし） */
splice(file_fd, &file_offset, pipefd[1], NULL, len, SPLICE_F_MOVE);

/* 段階2: パイプ → ソケット（scatter-gather対応NICならゼロコピー） */
splice(pipefd[0], NULL, sock_fd, NULL, len, SPLICE_F_MOVE | SPLICE_F_MORE);
```

### tee: パイプのデータを複製する {#tee-duplicate-pipe-data}

`tee()`は、ソースから消費することなく、2つのパイプ間でデータを複製する:

```c
ssize_t tee(int fd_in, int fd_out, size_t len, unsigned int flags);

/* ストリームを複製する: ログと転送を同時に行う */
int log_pipe[2], fwd_pipe[2];
pipe(log_pipe); pipe(fwd_pipe);

/* ソース → log_pipe */
splice(source_fd, NULL, log_pipe[1], NULL, len, 0);

/* log_pipe → fwd_pipe に複製する（コピーなし、ページを共有） */
tee(log_pipe[0], fwd_pipe[1], len, 0);

/* 両方から独立に消費する */
splice(log_pipe[0], NULL, logfile_fd, NULL, len, 0);
splice(fwd_pipe[0], NULL, socket_fd, NULL, len, 0);
```

## copy_file_range: カーネル側でのファイルコピー {#copy_file_range-kernel-side-file-copy}

`copy_file_range()`は、ユーザー空間に触れることなく2つのファイル間でコピーする。最近のファイルシステムでは**サーバーサイドコピー**（例: NFS、SMB、Btrfsのreflink）を使うことができる:

```c
#define _GNU_SOURCE
#include <unistd.h>

ssize_t copy_file_range(int fd_in, loff_t *off_in,
                         int fd_out, loff_t *off_out,
                         size_t len, unsigned int flags);

/* 高速なファイルコピー（reflinkなら一瞬で終わることもある） */
int src = open("src.dat", O_RDONLY);
int dst = open("dst.dat", O_WRONLY | O_CREAT, 0644);

off_t off_in = 0, off_out = 0;
copy_file_range(src, &off_in, dst, &off_out, file_size, 0);
```

### Btrfsのreflink {#btrfs-reflink}

Btrfs（およびreflink対応のXFS）では、`copy_file_range`は**reflink**を行う。コピー先は、copy-on-writeのセマンティクスでコピー元と同じエクステントを共有する:

```
Before copy_file_range:
  src inode → [extent A] [extent B] [extent C]

After copy_file_range (reflink):
  src inode → [extent A] [extent B] [extent C]
  dst inode →     ↑           ↑          ↑
              (shared, COW on write)

No data copied — instantaneous regardless of file size!
```

```bash
# ファイルシステムがreflinkをサポートしているか確認する
cp --reflink=always src.dat dst.dat   # サポートされていなければ失敗する
cp --reflink=auto   src.dat dst.dat   # 必要なら通常のコピーにフォールバックする
```

## カーネルの実装 {#kernel-implementation}

### sendfileのカーネル内経路 {#sendfile-kernel-path}

```c
/* fs/read_write.c */
SYSCALL_DEFINE4(sendfile64, int, out_fd, int, in_fd,
                loff_t __user *, offset, size_t, count)
{
    struct fd in = fdget(in_fd);
    struct fd out = fdget(out_fd);

    /* do_sendfileに委譲する */
    return do_sendfile(out.file, in.file, ppos, count, 0);
}

/* fs/read_write.c */
static ssize_t do_sendfile(struct file *out_file, struct file *in_file, loff_t *ppos, size_t count, loff_t max)
{
    /* in_fileからページを得るためにfile->f_op->splice_readを使う */
    /* out_fileにはsock_sendpage（またはgeneric_file_splice_write）を使う */

    /* ソケットの場合: tcp_sendpage → skb_fill_page_descを呼ぶ */
    /* ページ参照がskbに追加される — NICがscatter-gatherを持っていればコピーなし */
}
```

### spliceのカーネル内経路 {#splice-kernel-path}

```c
/* fs/splice.c */
SYSCALL_DEFINE6(splice, int, fd_in, loff_t __user *, off_in,
                int, fd_out, loff_t __user *, off_out,
                size_t, len, unsigned int, flags)
{
    if (ipipe && opipe) {
        /* パイプからパイプ: pipe_bufferの参照を移動するだけ */
        return splice_pipe_to_pipe(ipipe, opipe, len, flags);
    }
    if (ipipe) {
        /* パイプからファイル: パイプのバッファをファイルへ書き込む */
        return do_splice_from(ipipe, out, &offset, len, flags);
    }
    if (opipe) {
        /* ファイルからパイプ: ページ参照をパイプに追加する */
        return do_splice_to(in, &offset, opipe, len, flags);
    }
}

/* pipe_buffer: ページへの参照。データのコピーはない */
struct pipe_buffer {
    struct page      *page;
    unsigned int     offset, len;
    const struct pipe_buf_operations *ops;
    unsigned int     flags;
};
```

### copy_file_rangeのカーネル内経路 {#copy_file_range-kernel-path}

```c
/* fs/read_write.c */
SYSCALL_DEFINE6(copy_file_range, ...)
{
    /* まずファイルシステム固有の実装を試す */
    if (out_file->f_op->copy_file_range) {
        ret = out_file->f_op->copy_file_range(file_in, pos_in,
                                               file_out, pos_out,
                                               len, flags);
        if (ret != -EOPNOTSUPP && ret != -EXDEV)
            return ret;
    }

    /* フォールバック: generic_copy_file_range（spliceベース） */
    return generic_copy_file_range(file_in, pos_in,
                                    file_out, pos_out, len, flags);
}

/* Btrfsの実装: */
static ssize_t btrfs_copy_file_range(...)
{
    /* btrfs_clone_file_range → 共有エクステントを作成する */
    ret = btrfs_clone(src, dst, off, len, len, destoff, 0);
}
```

## 比較 {#comparison}

| API | ゼロコピーか | 用途 | カーネル → ユーザー空間のコピー |
|-----|-----------|----------|------------------------|
| `read()` + `write()` | いいえ | 汎用 | 2回のコピー |
| `sendfile()` | はい（NICのSGがあれば） | ファイル → ソケット | コピー0回 |
| `splice()` | はい | 任意のfd ↔ パイプ | コピー0回 |
| `tee()` | はい | パイプの複製 | コピー0回 |
| `copy_file_range()` | はい（reflink） | ファイル → ファイル | コピー0回（reflink） |
| `mmap()` + `write()` | 部分的 | ファイル → ソケット | 1回のコピー（page → SKB） |

## 性能に関するヒント {#performance-tips}

```bash
# sendfileが使われていることを確認する（ファイルサーバーをstraceする）:
strace -e sendfile64,splice nginx -g 'daemon off;' 2>&1 | head -20

# NICのscatter-gatherサポートを確認する（真のゼロコピーに必要）:
ethtool -k eth0 | grep scatter-gather
# scatter-gather: on

# 大きなファイルでは: sendfileはread+writeより大幅に高性能
# 典型的には: 100MB以上のファイルでスループットが2〜4倍向上する
```

```c
/* io_uringのsplice: ブロックしないゼロコピー */
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_splice(sqe,
    file_fd, file_offset,
    pipe_fd[1], -1,
    len, SPLICE_F_MOVE);
io_uring_sqe_set_flags(sqe, IOSQE_IO_LINK);

/* チェーン: パイプ → ソケット */
sqe = io_uring_get_sqe(&ring);
io_uring_prep_splice(sqe,
    pipe_fd[0], -1,
    sock_fd, -1,
    len, SPLICE_F_MOVE);

io_uring_submit(&ring);
```

## 参考文献 {#further-reading}

- [ファイル操作](file-ops.md) — read/writeのVFS経路
- [ページキャッシュ](../mm/page-cache.md) — spliceにおけるページ参照
- [Direct I/O](../io/direct-io.md) — ページキャッシュを完全に迂回する
- [io_uringの操作](../io-uring/io-uring-ops.md) — `IORING_OP_SPLICE`
- `fs/splice.c` — splice/sendfile/teeの実装
- `include/linux/splice.h`
