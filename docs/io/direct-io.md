# Direct I/O {#direct-io}

> O_DIRECT: 予測可能なI/Oレイテンシのためにページキャッシュをバイパスする

## O_DIRECTが行うこと {#what-o_direct-does}

デフォルトでは、`read()`と`write()`は**ページキャッシュ**を経由する。カーネルはディスクのデータをDRAMのpageに読み込み、読み出しはそこから応答する。書き込みはダーティpageに対して行われ、それが非同期にフラッシュされる。

O_DIRECTはページキャッシュをバイパスする:

```
Buffered I/O:
  write() → page cache (dirty page) → [async] → disk
  read()  ← page cache ← [lazy] ← disk

O_DIRECT:
  write() → disk (synchronous, no page cache)
  read()  ← disk (synchronous, no page cache)
```

## なぜO_DIRECTを使うのか {#why-use-o_direct}

**データベース（PostgreSQL、MySQL InnoDB、Oracle）**: データベースは独自のバッファプールを実装している。I/OをデータベースのバッファとOSのページキャッシュの両方に通すとDRAMが無駄になる。O_DIRECTはデータベースに完全な制御を与える:

```
Without O_DIRECT:
  DB buffer (8GB) + OS page cache (partial duplication) = wasted RAM

With O_DIRECT:
  DB buffer (8GB) = all RAM goes to the database
```

**予測可能なレイテンシ**: ページキャッシュのライトバックは、ダーティpageがフラッシュされる際に思わぬレイテンシスパイクを引き起こすことがある。O_DIRECTの書き込みは、データがドライブに到達した時点で完了する。

**NUMAを考慮したメモリ**: O_DIRECTではNUMAローカルなバッファをカーネルに渡すことができ、NUMAノードをまたぐキャッシュの影響を避けられる。

## アライメント要件 {#alignment-requirements}

O_DIRECTには厳格なアライメント要件がある。違反すると`EINVAL`が返る:

```c
#define SECTOR_SIZE 512    /* 大半のドライブでの最小値 */
#define BLOCK_SIZE  4096   /* ファイルシステムのブロックサイズ */

/* 次の3つはすべて論理ブロックサイズ（通常512Bまたは4096B）にアラインされていなければならない: */
/* 1. ファイルオフセット */
/* 2. バッファアドレス */
/* 3. 転送長 */

/* 正しい例: すべて4096バイトアライン */
void *buf;
posix_memalign(&buf, 4096, 4096);   /* 4096バイトアラインのバッファ */

int fd = open("file", O_RDWR | O_DIRECT);
pread(fd, buf, 4096, 0);    /* offset=0, len=4096 → OK */
pread(fd, buf, 4096, 512);  /* offset=512, len=4096 → 多くのFSでEINVAL */
pread(fd, buf, 512, 0);     /* len=512 → 動く可能性あり（blksize=512の場合） */
```

### 必要なアライメントの確認 {#checking-the-required-alignment}

```c
#include <sys/ioctl.h>
#include <linux/fs.h>

int blksz;
ioctl(fd, BLKSSZGET, &blksz);   /* 論理ブロックサイズ */
/* または */
struct stat st;
fstat(fd, &st);
/* st.st_blksize = 推奨I/Oブロックサイズ */

/* ブロックデバイスの場合: */
ioctl(fd, BLKPBSZGET, &blksz);  /* 物理ブロックサイズ（4096の場合あり） */
```

## カーネル内のO_DIRECT {#o_direct-in-the-kernel}

```c
/* fs/read_write.c */
static ssize_t do_loop_readv_writev(struct file *filp, struct iov_iter *iter,
                                     loff_t *ppos, int type, rwf_t flags)
{
    /* ブロックアラインされたI/Oのための O_DIRECT パス */
    if (filp->f_flags & O_DIRECT) {
        return filp->f_op->read_iter(kiocb, iter);
        /* → generic_file_direct_write → filemap_write_and_wait_range +
             filp->f_mapping->a_ops->direct_IO */
    }
    /* ... バッファードパス ... */
}

/* ファイルシステムごとの direct_IO 実装: */
/* ext4_direct_IO → iomap_dio_rw → bio 発行 */
static ssize_t ext4_direct_IO(struct kiocb *iocb, struct iov_iter *iter)
{
    struct inode *inode = iocb->ki_filp->f_mapping->host;
    size_t count = iov_iter_count(iter);

    /* 範囲内のダーティ page をフラッシュする必要がある（一貫性: ページキャッシュの方が新しいデータを持っている可能性がある） */
    ret = filemap_write_and_wait_range(inode->i_mapping, offset,
                                        offset + count - 1);
    if (ret)
        return ret;

    /* DIO を発行: ブロック層へ直接送られる */
    return iomap_dio_rw(iocb, iter, &ext4_iomap_ops,
                         &ext4_dio_write_ops, 0, 0, 0);
}
```

## iomap DIO: 現代のdirect I/Oパス {#iomap-dio-modern-direct-io-path}

4.10以降、ほとんどのファイルシステムはdirect I/Oに`iomap`層を使用している:

```c
/* fs/iomap/direct-io.c */
ssize_t iomap_dio_rw(struct kiocb *iocb, struct iov_iter *iter,
                      const struct iomap_ops *ops,
                      const struct iomap_dio_ops *dops,
                      unsigned int dio_flags,
                      void *private, size_t done_before)
{
    struct iomap_dio *dio;

    /* ファイルブロックを物理アドレスへマップする */
    while (iov_iter_count(iter)) {
        ret = iomap_apply(inode, pos, count, flags, ops, dio,
                           iomap_dio_iter);
    }

    /* すべての bio の完了を待つ（同期 DIO） */
    if (!is_sync_kiocb(iocb))
        return -EIOCBQUEUED;  /* 非同期: 呼び出し側が待つ */

    wait_for_completion_io(&dio->done);
    return dio->size;
}
```

## O_DIRECTとmmapの比較 {#o_direct-vs-mmap}

| 観点 | O_DIRECT | mmap |
|--------|----------|------|
| ページキャッシュ | バイパスする | 使用する |
| ゼロコピー | DMAで可能（バッファを直接アドレス指定） | ページキャッシュからコピー |
| ランダムアクセス | `pread(fd, buf, len, offset)` | `memcpy(buf, mmap_ptr + offset, len)` |
| CPUコスト | I/Oごとにsyscall | フォールト後はロード命令のみ |
| アライメント | 厳格なセクタアライメント | なし（page粒度） |
| スキャッタ・ギャザー | `preadv(fd, iov, n, offset)` | 自然に可能（ポインタ演算） |
| 典型的な用途 | データベース | 検索エンジン、キーバリューストア |

## O_DSYNCとO_SYNC: 書き込みの永続性 {#odsync-and-osync-write-durability}

```c
/* O_SYNC: 毎回の書き込み後に fsync()（メタデータ + データが永続化） */
int fd = open("file", O_WRONLY | O_SYNC);

/* O_DSYNC: データのみ永続化（性能のためメタデータは同期しない） */
int fd = open("file", O_WRONLY | O_DSYNC);

/* fsync() を手動で: バッファリングされた書き込みをディスクへフラッシュ */
write(fd, data, len);
fsync(fd);  /* データのみなら fdatasync(fd) */
```

## O_DIRECTの性能向上のヒント {#o_direct-performance-tips}

```c
/* O_DIRECT で最高性能を得るには io_uring の IORING_OP_READ_FIXED を使う */
/* バッファを一度だけ登録: */
struct iovec iov[1] = {{ .iov_base = buf, .iov_len = BUF_SIZE }};
io_uring_register_buffers(ring, iov, 1);

/* 登録済みバッファで発行（操作ごとのマッピングが不要） */
struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
io_uring_prep_read_fixed(sqe, fd, buf, BUF_SIZE, offset, 0);
io_uring_submit(ring);
```

## 参考文献 {#further-reading}

- [非同期I/Oの進化](async-io.md) — O_DIRECT向けのAIO、io_uring
- [io_uring: 操作](../io-uring/io-uring-ops.md) — IORING_OP_READ_FIXED
- [ブロック層: bioとrequest](../block/bio-request.md) — DIOからのbio発行
- [メモリ管理: ページキャッシュ](../mm/page-cache.md) — O_DIRECTがバイパスするもの
- `fs/iomap/direct-io.c` — 現代のDIO実装
- `Documentation/filesystems/dax.txt` — PMEM向けのDAX
