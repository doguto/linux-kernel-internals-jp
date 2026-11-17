# pread / pwrite / preadv2 / pwritev2 {#pread-pwrite-preadv2-pwritev2}

> 位置指定I/O、スキャッタ・ギャザー、呼び出しごとのフラグ: pread64からRWF_ATOMICまでのAPI全体

## 並行I/Oにおけるread() / write()の問題 {#the-problem-with-read-write-for-concurrent-io}

`read()`と`write()`は`file->f_pos`、すなわちオープンファイル記述（open file description）に格納されたファイル位置を使う。オープンファイル記述は、それを参照するすべてのファイルディスクリプタ（`dup`、`dup2`、`fork`などの後）の間で共有されるため、並行アクセスは競合状態を生む。

```
Thread A                        Thread B
─────────────────────────────   ────────────────────────────────
lseek(fd, 0, SEEK_SET)         lseek(fd, 4096, SEEK_SET)
                                    ← context switch here
read(fd, buf, 4096)            ← reads from offset 4096, not 0
```

2つのスレッドがそれぞれ`open()`で独立に得た自分の`fd`を持っている場合でも、`lseek()` + `read()`はアトミックではない。同じスレッド上の別の`read()`が、シークと読み出しの間に割り込みうる。

カーネルは`f_pos`の更新を`file->f_pos_lock`（mutex）で直列化するが、この直列化は操作単位であり、シーク+読み出しを1つの単位としてアトミックにするものではない。

```
Thread A: fdget_pos() → acquires f_pos_lock
          f_pos = 4096
          fdput_pos() → releases f_pos_lock
Thread A: fdget_pos() → acquires f_pos_lock
          read from f_pos (4096)
          f_pos += bytes_read
          fdput_pos() → releases f_pos_lock
```

この2回のロック取得の間に、スレッドBは`f_pos`を変更できる。これはアプリケーションレベルでのTOCTOU問題である。

`pread()`と`pwrite()`はこれを解決する。これらは明示的な`offset`引数を受け取り、`f_pos`を読むことも書くこともない。オフセットは呼び出し側のスタック上にあるので、競合は起きない。

---

## pread() / pwrite() {#pread-pwrite}

```c
#include <unistd.h>

/* fd のオフセット offset から count バイトを buf に読む。f_pos は変わらない。 */
ssize_t pread(int fd, void *buf, size_t count, off_t offset);

/* buf から count バイトを fd のオフセット offset に書く。f_pos は変わらない。 */
ssize_t pwrite(int fd, const void *buf, size_t count, off_t offset);
```

### システムコールの実装 {#syscall-implementation}

カーネルのエントリポイントは`pread64`と`pwrite64`である（`64`という接尾辞は、オフセットがglibcの`off64_t`に対応する64ビットの`loff_t`であることを示す）。

```c
/* fs/read_write.c */
SYSCALL_DEFINE4(pread64, unsigned int, fd, char __user *, buf,
                size_t, count, loff_t, pos)
{
    struct fd f;
    ssize_t ret = -EBADF;

    f = fdget(fd);          /* 注意: fdget_pos ではなく fdget */
    if (f.file) {
        ret = -ESPIPE;
        if (f.file->f_mode & FMODE_PREAD)   /* ファイルはシーク可能でなければならない */
            ret = vfs_read(f.file, buf, count, &pos);
            /*
             * pos はカーネルスタック上のローカル変数である。
             * vfs_read() はそれを進めることがあるが、その結果が
             * f.file->f_pos に書き戻されることはない。
             */
        fdput(f);
    }
    return ret;
}
```

`FMODE_PREAD` / `FMODE_PWRITE`のモードフラグは、通常ファイルとブロックデバイスでセットされる。パイプ、ソケット、その他のシーク不可能なファイル種別はこれらのフラグをセットせず、`ESPIPE`を返す。

### fdgetとfdget_pos — 重要な違い {#fdget-vs-fdget_pos-the-key-difference}

`read()`は`fdget_pos()`を使う。

```c
/* fs/read_write.c */
ssize_t ksys_read(unsigned int fd, char __user *buf, size_t count)
{
    struct fd f = fdget_pos(fd);    /* f_pos_lock を取得する */
    ssize_t ret = -EBADF;

    if (f.file) {
        loff_t pos, *ppos = file_ppos(f.file);
        if (ppos) {
            pos = *ppos;
            ppos = &pos;
        }
        ret = vfs_read(f.file, buf, count, ppos);
        if (!IS_ERR_VALUE(ret)) {
            if (ppos)
                f.file->f_pos = pos;    /* 更新された位置を書き戻す */
        }
        fdput_pos(f);                   /* f_pos_lock を解放する */
    }
    return ret;
}
```

`pread()`は`fdget()`を使う（ロックなし、位置の更新なし）。

| | `read()` / `write()` | `pread()` / `pwrite()` |
|---|---|---|
| 取得するロック | `f_pos_lock`（mutex） | なし |
| 位置の取得元 | `file->f_pos` | スタック上のローカル変数 |
| 実行後に位置を更新するか | する | しない |
| 並行スレッドで安全か | 逐次的な場合のみ | 安全 |

`pread()`/`pwrite()`は`f_pos`に触れず`f_pos_lock`も取らないため、複数のスレッドが同じオープンファイル記述に対して、直列化のオーバーヘッドなしに並行して位置指定読み出しを発行できる。

### エラーケース {#error-cases}

```c
/* ESPIPE: fd がパイプまたはソケット */
pread(pipefd[0], buf, 4096, 0);     /* → -1, errno = ESPIPE */

/* EINVAL: オフセットが負 */
pread(fd, buf, 4096, -1);           /* → -1, errno = EINVAL */

/* 問題なく動作する: EOF を超えるオフセットは 0 を返す */
pread(fd, buf, 4096, 1ULL << 40);   /* → ファイルがより小さければ 0（EOF） */
```

---

## preadv() / pwritev() {#preadv-pwritev}

`readv()`と`writev()`はPOSIX.1-2001からスキャッタ・ギャザーI/Oを備えていたが、明示的なオフセットを指定する方法はなかった。Linux 2.6.30はこの穴を埋めるために`preadv()`と`pwritev()`を追加した。

```c
#include <sys/uio.h>

ssize_t preadv(int fd, const struct iovec *iov, int iovcnt, off_t offset);
ssize_t pwritev(int fd, const struct iovec *iov, int iovcnt, off_t offset);
```

これらは`pread()`/`pwrite()`のオフセットのセマンティクスと、`readv()`/`writev()`のスキャッタ・ギャザー機能を組み合わせたものである。ベクタでない版と同様に、`f_pos`に触れず、`FMODE_PREAD`/`FMODE_PWRITE`を必要とする。

### カーネル内の経路 {#kernel-path}

```c
/* fs/read_write.c */
SYSCALL_DEFINE5(preadv, unsigned long, fd,
                const struct iovec __user *, vec, unsigned long, vlen,
                unsigned long, pos_l, unsigned long, pos_h)
{
    loff_t pos = pos_from_hilo(pos_h, pos_l);
    return do_preadv(fd, vec, vlen, pos, 0);
}

static ssize_t do_preadv(unsigned long fd, const struct iovec __user *vec,
                          unsigned long vlen, loff_t pos, rwf_t flags)
{
    struct fd f;
    ssize_t ret = -EBADF;

    f = fdget(fd);
    if (f.file) {
        ret = -ESPIPE;
        if (f.file->f_mode & FMODE_PREAD) {
            struct iovec iovstack[UIO_FASTIOV];
            struct iovec *iov = iovstack;
            struct iov_iter iter;

            ret = import_iovec(READ, vec, vlen,
                               ARRAY_SIZE(iovstack), &iov, &iter);
            if (ret >= 0) {
                ret = vfs_iter_read(f.file, &iter, &pos, flags);
                kfree(iov);
            }
        }
        fdput(f);
    }
    return ret;
}
```

`pos`を`pos_l`と`pos_h`に分けているのは32ビットABIの名残である。32ビットアーキテクチャでは、64ビットの`loff_t`は2つの32ビットレジスタで渡される。64ビットアーキテクチャではABIが単一のレジスタを提供する。

### なぜデータベースはpreadvを使うか {#why-databases-use-preadv}

典型的なデータベースのpage読み出しは次のようになる。

```c
/*
 * 固定オフセットから B-tree の page を、ヘッダとデータの別々の
 * バッファへ読み込む。f_pos には触れず、中間バッファの割り当てもしない。
 */
struct page_header hdr;
uint8_t data[PAGE_SIZE - sizeof(hdr)];

struct iovec iov[2] = {
    { .iov_base = &hdr,  .iov_len = sizeof(hdr)  },
    { .iov_base = data,  .iov_len = sizeof(data)  },
};

ssize_t n = preadv(dbfile, iov, 2, page_no * PAGE_SIZE);
```

利点は2つある。
1. **memcpyが不要**: ヘッダとデータが別々の構造体へ直接入る。フラットなバッファに読み込んでから分割する必要はない。
2. **スレッドセーフ**: 複数のスレッドが、ユーザー空間でのロックなしに、同じ`fd`に対して異なるオフセットで同時に`preadv()`を呼べる。

---

## preadv2() / pwritev2(): 呼び出しごとのフラグ {#preadv2-pwritev2-per-call-flags}

Linux 4.6は`preadv()`/`pwritev()`を`flags`引数で拡張した。

```c
#include <sys/uio.h>

ssize_t preadv2(int fd, const struct iovec *iov, int iovcnt,
                off_t offset, int flags);
ssize_t pwritev2(int fd, const struct iovec *iov, int iovcnt,
                 off_t offset, int flags);
```

`flags`は`rwf_t`型（`__kernel_rwf_t`、つまり`int`として定義される）である。これにより、以前はファイルオープン時に`O_`フラグで設定するか、別途fcntl呼び出しを必要としたI/Oの振る舞いを、呼び出しごとに制御できる。

### RWF_* フラグ {#rwf_-flags}

| フラグ | 値 | カーネルバージョン | 説明 |
|------|-------|----------------|-------------|
| `RWF_HIPRI` | `0x01` | 4.6 | 高優先度I/O。完了をポーリングする（NVMeのポーリングキュー） |
| `RWF_DSYNC` | `0x02` | 4.7 | この書き込みにデータ同期セマンティクスを適用（呼び出し単位の`O_DSYNC`） |
| `RWF_SYNC` | `0x04` | 4.7 | この書き込みを完全同期にする（呼び出し単位の`O_SYNC`） |
| `RWF_NOWAIT` | `0x08` | 4.14 | I/Oがブロックする場合は直ちに`EAGAIN`を返す |
| `RWF_APPEND` | `0x10` | 4.16 | `offset`にかかわらずファイル末尾へアトミックに追記する |
| `RWF_NOAPPEND` | `0x20` | 6.9 | この呼び出しについて`O_APPEND`を上書きし、指定オフセットに書き込む |
| `RWF_ATOMIC` | `0x40` | 6.11 | ハードウェアが保証するアトミック書き込み |

フラグは`include/uapi/linux/fs.h`で定義されている。

### フラグがファイルシステムに届く仕組み: kiocb_set_rw_flags {#how-flags-reach-the-filesystem-kiocb_set_rw_flags}

カーネル内のすべてのI/O操作は`struct kiocb`（kernel I/O control block）で表される。`RWF_*`フラグは`IOCB_*`フラグに変換され、`kiocb.ki_flags`に格納される。

```c
/* include/linux/fs.h */
static inline int kiocb_set_rw_flags(struct kiocb *ki, rwf_t flags,
                                      int rw_type)
{
    int kiocb_flags = 0;

    /* 未知のフラグをチェックする */
    if (unlikely(flags & ~RWF_SUPPORTED))
        return -EOPNOTSUPP;

    if (flags & RWF_NOWAIT) {
        if (!(ki->ki_filp->f_mode & FMODE_NOWAIT))
            return -EOPNOTSUPP;
        kiocb_flags |= IOCB_NOIO;          /* I/O は発行せず、チェックのみ */
        kiocb_flags |= IOCB_NOWAIT;
    }
    if (flags & RWF_HIPRI)
        kiocb_flags |= IOCB_HIPRI;
    if (flags & RWF_DSYNC)
        kiocb_flags |= IOCB_DSYNC;
    if (flags & RWF_SYNC)
        kiocb_flags |= (IOCB_DSYNC | IOCB_SYNC);
    if (flags & RWF_APPEND)
        kiocb_flags |= IOCB_APPEND;
    if (flags & RWF_NOAPPEND)
        kiocb_flags |= IOCB_NOAPPEND;
    if (flags & RWF_ATOMIC)
        kiocb_flags |= IOCB_ATOMIC;
    if (iocb_is_dsync(ki))
        kiocb_flags |= IOCB_DSYNC;

    ki->ki_flags |= kiocb_flags;
    return 0;
}
```

`kiocb`に`IOCB_*`フラグがセットされると、そのI/Oを扱うすべての層、すなわちVFS、ファイルシステム（`ext4`、`xfs`）、ブロック層がこれらのフラグを確認して振る舞いを調整する。ファイルシステムはスリープするかもしれないロックを取る前に`iocb->ki_flags & IOCB_NOWAIT`を確認し、ブロック層は`IOCB_HIPRI`を確認してポーリング型のNVMeキューへ振り分ける。

### offset = -1: 現在のf_posを使う {#offset-1-use-current-f_pos}

`offset`が（`off_t`にキャストした）`-1`のとき、`preadv2()` / `pwritev2()`は`readv()` / `writev()`と同じように振る舞う。現在の`f_pos`が使われ、その後更新される。これにより、明示的なオフセットを持たずに`RWF_*`フラグを使うことができる。

```c
/* 現在のファイル位置で RWF_NOWAIT を使う */
ssize_t n = preadv2(fd, iov, 1, (off_t)-1, RWF_NOWAIT);
```

`offset == -1`の場合、システムコールは`fdget_pos()`にフォールバックして`f_pos_lock`を取る。つまり、`readv()`にフラグを加えたものとまったく同じように振る舞う。

---

## RWF_NOWAITの詳細 {#rwf_nowait-in-detail}

`RWF_NOWAIT`は最もよく使われる呼び出しごとのフラグである。これはカーネルに「ブロックせずに済む場合に限りこのI/Oを完了させよ。そうでなければ`EAGAIN`を返せ」と依頼する。

```c
/* ページキャッシュからのノンブロッキング読み出しを試みる */
ssize_t n = preadv2(fd, iov, 1, offset, RWF_NOWAIT);
if (n == -1 && errno == EAGAIN) {
    /* I/O はブロックしていたはず — 非同期で発行する */
    submit_async_io(fd, iov, offset);
    return;
}
if (n < 0) {
    perror("preadv2");
    return;
}
/* データはページキャッシュにあった — I/O なしで返された */
process_data(iov[0].iov_base, n);
```

### RWF_NOWAIT付きのpreadv2がEAGAINを返すとき {#when-preadv2-with-rwf_nowait-returns-eagain}

読み出しは次の場合に、I/Oを一切行わずに`EAGAIN`を返す。

**ページキャッシュミス**: 要求されたpageがページキャッシュになく、それを取得するにはブロックI/Oの発行が必要になる場合。`generic_file_read_iter`は`filemap_get_pages`の中で`IOCB_NOWAIT`を確認し、`read_folio`を呼んでbioを発行する代わりに`-EAGAIN`を返す。

**inodeロックの競合**: バッファード読み出しでは、`generic_file_read_iter`が`inode->i_rwsem`を読み取りモードで必要とすることがある。ロックが書き込み側に保持されていて`IOCB_NOWAIT`がセットされていれば、カーネルはロックでスリープする代わりに`-EAGAIN`を返す。

**ファイルシステムのジャーナルの混雑**: ext4のようなファイルシステムは、一部の経路では読み出しでもジャーナルのリソースを必要とすることがある。`IOCB_NOWAIT`がセットされていて必要なリソースが利用できなければ、`-EAGAIN`が返される。

**ダイレクトI/Oのエクステントマッピング**: `O_DIRECT`読み出しでは、ext4とXFSはエクステントマップを検索する必要がある。エクステントが存在せず（ホール）`IOCB_NOWAIT`がセットされている場合、XFSはエクステントの割り当てにジャーナルトランザクションが必要になるため`-EAGAIN`を返す。

**truncateやライトバックのロック**: 一部のファイルシステムはtruncateの間`i_rwsem`を排他的に保持する。共有ロックを取得できない並行の`RWF_NOWAIT`読み出しは`-EAGAIN`を返す。

注意: バッファード読み出しは、pageがすでにキャッシュにない限り、`RWF_NOWAIT`で成功することはまれである。頻繁に`EAGAIN`を受け取るワークロードでは、ポーリングではなく真の非同期I/Oを発行する`IORING_OP_READ`付きの`io_uring`のほうが効率的である。

### FMODE_NOWAITによる関門 {#the-fmode_nowait-gate}

すべてのファイル種別が`RWF_NOWAIT`をサポートしているわけではない。カーネルはフラグを尊重する前に、ファイルの`FMODE_NOWAIT`を確認する。

```c
if (flags & RWF_NOWAIT) {
    if (!(ki->ki_filp->f_mode & FMODE_NOWAIT))
        return -EOPNOTSUPP;
    ...
}
```

`FMODE_NOWAIT`は、ノンブロッキングI/Oをサポートするファイルシステム（ext4、XFS、btrfs）上で開かれた通常ファイルと、ブロックデバイスでセットされる。ソケット、パイプ、オプトインしていないファイルシステムではセットされない。

---

## RWF_DSYNCとRWF_SYNC {#rwf_dsync-and-rwf_sync}

`O_DSYNC`と`O_SYNC`は、オープン時に設定されるfdごとのフラグである。これらは各書き込みが戻る前に、データ（および場合によってはメタデータ）を安定ストレージへ書き込むよう強制する。呼び出しごとの同等機能を使えば、一部の書き込みだけが永続性を必要とする場合に、ファイルをこれらのフラグ付きで開くオーバーヘッドを避けられる。

```c
/*
 * トランザクションログのエントリを書く: 戻る前に永続化されていなければならない。
 * 通常のデータ書き込みにはこのオーバーヘッドは不要である。
 */
ssize_t n = pwritev2(logfd, iov, 1, log_offset, RWF_DSYNC);
if (n < 0) {
    perror("pwritev2 RWF_DSYNC");
}
/* データはこれで安定ストレージ上にある — トランザクションを確認応答してよい */
```

`RWF_DSYNC`は`IOCB_DSYNC`に対応し、書き込み完了後に`vfs_fsync_range()`を強制して、データpageとファイルのinodeをフラッシュする（ただしディレクトリエントリはフラッシュしない）。`RWF_SYNC`は`IOCB_DSYNC | IOCB_SYNC`に対応し、メタデータも強制する（`O_SYNC`相当）。

`do_iter_write`での実装は次のとおりである。

```c
/* fs/read_write.c */
static ssize_t do_iter_write(struct file *file, struct iov_iter *iter,
                              loff_t *pos, rwf_t flags)
{
    size_t tot_len;
    ssize_t ret = 0;

    /* ... 権限チェック ... */

    ret = __generic_file_write_iter(file_iocb, iter);  /* 実際の書き込み */

    if (ret > 0 && iocb_is_dsync(file_iocb)) {
        /* RWF_DSYNC または RWF_SYNC: データを安定ストレージへフラッシュする */
        ret = generic_write_sync(file_iocb, ret);
    }

    return ret;
}
```

---

## RWF_APPEND {#rwf_append}

`O_APPEND`は、fdへのすべての書き込みをアトミックな追記にする。各書き込みの前にinodeロックの下でファイル位置がEOFへ移動されるため、2つの並行する書き込み側が互いに上書きし合うことを防ぐ。しかし`O_APPEND`はオープンファイル記述の属性であり、`fcntl(F_SETFL)`なしに呼び出しごとに切り替えることはできない。

`RWF_APPEND`は同じアトミックな追記の保証を呼び出し単位で提供する。

```c
/*
 * ログレコードをアトミックに追記する。
 * fd は O_APPEND なしで開かれている。呼び出しごとに RWF_APPEND を使うので、
 * 同じ fd でランダムな読み書きも行える。
 */
ssize_t n = pwritev2(fd, iov, 1, /* オフセットは無視される */ 0, RWF_APPEND);
```

`RWF_APPEND`がセットされていると、`offset`引数は無視される。書き込み位置は、ファイルシステムによってinodeロックの下で、書き込み自体とアトミックに決定される。

実装: `IOCB_APPEND`により、VFSとファイルシステムは次を行う。
1. inodeを排他的にロックする。
2. `i_size`を読んで現在のEOFを得る。
3. EOFからデータを書き込む。
4. `i_size`をアトミックに更新する。
5. アンロックする。

これは`O_APPEND`と同じコードパスであり、ファイルモードではなく呼び出しごとのフラグによって起動されるだけである。

### RWF_NOAPPEND（Linux 6.9） {#rwf_noappend-linux-69}

逆の働きをするもので、ファイルが`O_APPEND`で開かれている場合に、`RWF_NOAPPEND`は1回の呼び出しについてそれを上書きし、代わりに指定された明示的な`offset`に書き込む。

```c
/* fd は O_APPEND で開かれているが、このレコードだけは固定オフセットに書く */
pwritev2(fd, iov, 1, fixed_offset, RWF_NOAPPEND);
```

---

## RWF_HIPRI: 高優先度のポーリングI/O {#rwf_hipri-high-priority-polled-io}

NVMeドライブは **ポーリング型I/Oキュー** をサポートする。I/O完了時にハードウェア割り込みを待つ代わりに、CPUがNVMeの完了キューをスピンして監視する。これによりCPU時間と引き換えにレイテンシが減る。`RWF_HIPRI`は、このI/Oについてポーリングによる完了を要求する。

```c
/* ポーリング I/O には O_DIRECT が必要 */
int fd = open("file", O_RDWR | O_DIRECT);

ssize_t n = preadv2(fd, iov, 1, offset, RWF_HIPRI);
```

`RWF_HIPRI`はkiocbに`IOCB_HIPRI`をセットする。ブロック層はこのフラグを確認し、ブロックデバイスがポーリングキュー（`BLK_MQ_F_POLL`）をサポートしていれば、bioをポーリングキューへ振り分け、完了割り込みを待つ代わりに`blk_poll()`を呼ぶ。

有効なのは`O_DIRECT`の場合のみである。バッファードI/Oの完了はライトバックスレッドを介して非同期に起こり、アプリケーションがポーリングすることはできない。

---

## RWF_ATOMIC（Linux 6.11） {#rwf_atomic-linux-611}

従来のディスク書き込みには、ハードウェアレベルでのアトミック性の保証がない。書き込みの途中で電源が落ちると、セクタが部分的に書かれたままになりうる。データベースはジャーナリングやコピーオンライトでこれを回避している。現代のストレージデバイス（Atomic Write Unit機能セットを持つNVMe）は、`N`バイトまでの書き込みが完全にコミットされるか、まったくコミットされないかのどちらかであることを保証できる。

Linux 6.11はこれを`RWF_ATOMIC`として公開した。

```c
#include <sys/stat.h>   /* statx */
#include <linux/stat.h> /* STATX_WRITE_ATOMIC */

/* ステップ 1: デバイスの能力を確認する */
struct statx stx;
if (statx(fd, "", AT_EMPTY_PATH, STATX_WRITE_ATOMIC, &stx) < 0)
    err(1, "statx");

printf("atomic write unit min: %u bytes\n", stx.stx_atomic_write_unit_min);
printf("atomic write unit max: %u bytes\n", stx.stx_atomic_write_unit_max);
printf("max atomic segments:   %u\n",       stx.stx_atomic_write_segments_max);

/* ステップ 2: アトミック書き込みを発行する */
/* バッファは stx_atomic_write_unit_min にアラインされていなければならない */
/* 長さは stx_atomic_write_unit_max を超えてはならない */
ssize_t n = pwritev2(fd, iov, 1, offset, RWF_ATOMIC);
if (n < 0 && errno == EINVAL) {
    /* アラインメントやサイズの制約違反、または FS が未サポート */
}
```

### RWF_ATOMICの要件 {#requirements-for-rwf_atomic}

- 書き込みサイズは、`stx_atomic_write_unit_min`と`stx_atomic_write_unit_max`の間の2のべき乗でなければならない。
- バッファアドレスとファイルオフセットは`stx_atomic_write_unit_min`にアラインされていなければならない。
- 書き込みは`stx_atomic_write_unit_max`にアラインされた境界をまたいではならない。
- ファイルシステムがサポートしていなければならない。XFSは6.11時点で`O_DIRECT`書き込みに対する`RWF_ATOMIC`をサポートしている。ext4のサポートは進行中である。
- ブロックデバイスがNVMeまたはSCSIインターフェースを通じてAtomic Write Unitを通知していなければならない。

### なぜこれがデータベースにとって重要か {#why-this-matters-for-databases}

`RWF_ATOMIC`なしの場合:
```
journaling overhead:
  write journal record → flush → update actual page → flush
  2× I/O amplification, 2× flush latency
```

対応ハードウェアで`RWF_ATOMIC`を使う場合:
```
  write page directly at target location → hardware guarantees atomicity
  1× I/O, 0 journal overhead
```

PostgreSQLとMySQLは、カーネルとファイルシステムのサポートが安定すれば、8 KB〜16 KBのpage書き込みに`RWF_ATOMIC`を採用すると見込まれている。

---

## f_posロックの内部 {#f_pos-locking-internals}

補足として、`read()` / `write()`で`f_pos_lock`がどう働くか、そして`pread()`がなぜそれを省略できるかを示す。

### ロックそのもの {#the-lock-itself}

`file->f_pos_lock`は`struct file`に埋め込まれた`mutex`である。

```c
/* include/linux/fs.h */
struct file {
    /* ... */
    spinlock_t          f_lock;
    fmode_t             f_mode;
    atomic_long_t       f_count;
    struct mutex        f_pos_lock;   /* read/write のために f_pos を保護する */
    loff_t              f_pos;
    /* ... */
};
```

### file_ppos: ストリームとシーク可能なファイル {#file_ppos-stream-vs-seekable}

すべてのファイル種別が意味のある位置を持つわけではない。`file_ppos()`はストリームファイル（パイプ、ソケット）に対して`NULL`を返し、ロックの経路を完全にスキップさせる。

```c
/* fs/read_write.c */
static inline loff_t *file_ppos(struct file *file)
{
    return file->f_mode & FMODE_STREAM ? NULL : &file->f_pos;
}
```

### fdget_pos / fdput_pos {#fdget_pos-fdput_pos}

```c
/* fs/read_write.c */
static struct fd fdget_pos(int fd)
{
    return __to_fd(__fdget_pos(fd));
}

/*
 * __fdget_pos: ファイルが位置を持ち、かつ複数のスレッドがこの
 * ファイル記述を共有している（f_count > 1）場合、f_pos_lock を取得して、
 * 並行する read/write 呼び出しが f_pos の更新を直列化するようにする。
 */

static inline void fdput_pos(struct fd f)
{
    if (f.flags & FDPUT_POS_UNLOCK)
        mutex_unlock(&f.file->f_pos_lock);
    fdput(f);
}
```

ロックは`f_count > 1`（複数のfdが同じオープンファイル記述を参照している）の場合にのみ取られる。シングルスレッドのプロセスが開き、`dup`/`fork`で共有されていないファイルが`f_pos_lock`で競合することはない。

### まとめ: f_pos_lockはいつ取られるか {#summary-when-does-f_pos_lock-fire}

| シナリオ | f_pos_lockを取るか |
|---|---|
| シングルスレッド、dupなし | 取らない（`f_count == 1`） |
| `dup(fd)`の後 | 取る |
| `fork()`の後 | 取る（親と子が記述を共有する） |
| パイプに対する`read()` | 取らない（`FMODE_STREAM` → `file_ppos`がNULLを返す） |
| 任意のファイルに対する`pread()` | 決して取らない |

---

## io_uringでの同等機能 {#io_uring-equivalents}

`io_uring`（Linux 5.1以降）は位置指定I/Oを第一級の操作として公開しており、`preadv2`/`pwritev2`のセマンティクスに直接対応する。

| io_uringのオペコード | 同等のシステムコール | 備考 |
|---|---|---|
| `IORING_OP_READ` | `pread()` | 単一バッファ、明示的なオフセット |
| `IORING_OP_WRITE` | `pwrite()` | 単一バッファ、明示的なオフセット |
| `IORING_OP_READV` | `preadv()` | ベクタ、明示的なオフセット |
| `IORING_OP_WRITEV` | `pwritev()` | ベクタ、明示的なオフセット |
| `IORING_OP_READ_FIXED` | `pread()` | 事前登録したバッファ（ユーザーからのコピーなし） |
| `IORING_OP_WRITE_FIXED` | `pwrite()` | 事前登録したバッファ（ユーザーからのコピーなし） |

`RWF_*`フラグは`sqe->rw_flags`を介して渡される。

```c
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);

io_uring_prep_readv(sqe, fd, iov, 1, offset);
sqe->rw_flags = RWF_NOWAIT;   /* preadv2 と同じ RWF_* フラグ */

io_uring_submit(&ring);
```

カーネル内部では、`io_uring`は`kiocb`を作成し、`sqe->rw_flags`の値を使って`kiocb_set_rw_flags()`を呼ぶ。これは`preadv2`が呼ぶのと同じ関数である。`IOCB_*`フラグは、まったく同じ方法でファイルシステムを伝播する。

`preadv2`に対する`io_uring`の主な利点は、呼び出しごとのフラグ（これらは同一である）ではなく **発行のバッチ化** である。多数のI/O操作が1回の`io_uring_enter()`システムコールで発行され、高IOPSのワークロードでのシステムコールのオーバーヘッドを減らす。

---

## 性能比較 {#performance-comparison}

| システムコール | `f_pos_lock`? | ベクタ? | 呼び出しごとのフラグ? | 最適な用途 |
|---------|:---:|:---:|:---:|---|
| `read` / `write` | あり | なし | なし | 単純なシーケンシャルアクセス |
| `pread` / `pwrite` | なし | なし | なし | ランダムアクセス、並行スレッド |
| `readv` / `writev` | あり | あり | なし | シーケンシャルなスキャッタ・ギャザー |
| `preadv` / `pwritev` | なし | あり | なし | ランダムアクセスのスキャッタ・ギャザー |
| `preadv2` / `pwritev2` | なし | あり | あり | 呼び出しごとの完全な制御 |
| `io_uring` | なし | あり | あり | 高い並行性、バッチ発行 |

複数のスレッドでランダム読み出しを発行するデータベースの場合:

- `read()` + `lseek()`: シーク+読み出しの組ごとに外部のmutexが必要。
- `pread()`: mutexなしで安全。各スレッドが自分のオフセットを渡す。
- `preadv()`: 安全で、pageのヘッダと本体が別々の構造体へ入る場合に余分な`memcpy`を避けられる。
- `preadv2(..., RWF_NOWAIT)`: ブロックせずにページキャッシュを確認できる。`EAGAIN`の場合は`io_uring`にフォールバックする。
- `IORING_OP_READ_FIXED`付きの`io_uring`: 持続的なランダムI/Oで最もオーバーヘッドが小さい。

---

## 主要なソースファイル {#key-source-files}

| ファイル | 内容 |
|------|----------|
| `fs/read_write.c` | `pread64`、`pwrite64`、`do_preadv`、`do_pwritev`、`ksys_read`、`fdget_pos`、`fdput_pos`、`file_ppos` |
| `include/linux/fs.h` | `struct file`、`f_pos_lock`、`FMODE_PREAD`、`FMODE_PWRITE`、`FMODE_NOWAIT`、`kiocb_set_rw_flags`、`IOCB_*`フラグ |
| `include/uapi/linux/fs.h` | `RWF_HIPRI`、`RWF_DSYNC`、`RWF_SYNC`、`RWF_NOWAIT`、`RWF_APPEND`、`RWF_NOAPPEND`、`RWF_ATOMIC`、`rwf_t` |
| `mm/filemap.c` | `generic_file_read_iter`、`filemap_get_pages` — バッファード読み出しで`IOCB_NOWAIT`が確認される場所 |
| `fs/inode.c` | `inode_init_always` — `i_rwsem`が初期化される場所（`RWF_NOWAIT`の経路で取得される） |
| `io_uring/rw.c` | io_uringの読み書きハンドラ。`sqe->rw_flags`を使って`kiocb_set_rw_flags`を呼ぶ |
| `block/blk-mq.c` | `blk_poll` — `IOCB_HIPRI`のためのポーリングによる完了経路 |
| `fs/xfs/xfs_file.c` | XFSの`read_iter` / `write_iter`。`RWF_ATOMIC`と`IOCB_NOWAIT`の処理 |

---

## 参考文献 {#further-reading}

- [ベクタI/O（readv/writev）](vectored-io.md) — `struct iovec`、`iov_iter`、スキャッタ・ギャザーの基礎
- [ダイレクトI/O](direct-io.md) — `O_DIRECT`のアラインメント要件。`RWF_HIPRI`には`O_DIRECT`が必要
- [バッファードI/Oとページキャッシュ](buffered-io.md) — `RWF_NOWAIT`が何に対して判定するか。`filemap_get_pages`
- [非同期I/Oの進化](async-io.md) — io_uringの`IORING_OP_READ` / `IORING_OP_WRITE`と`sqe->rw_flags`
- [fsync / fdatasync](fsync-fdatasync.md) — `RWF_DSYNC`と`RWF_SYNC`が内部で何を引き起こすか
