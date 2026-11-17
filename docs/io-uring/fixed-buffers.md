# 固定バッファと固定ファイル {#fixed-buffers-and-fixed-files}

> ページのピン留めとfd検索のコストをI/O操作全体に分散（償却）する

## 操作ごとの税 {#the-per-operation-tax}

すべての`read()`や`write()`、そしてすべての素の`IORING_OP_READ`は、見落とされがちな2つのコストを支払っている。

**ページのピン留め**。DMAを開始する前に、カーネルは転送の間、対象のページが物理メモリに常駐し続けることを保証しなければならない。つまり`get_user_pages()`を呼んでページテーブルをウォークし、ページの参照カウントを増やし、各ページを回収から守るためにピン留めする必要がある（これが土台とするget対pinの仕組みについては[ユーザーページの取得（GUP）](../mm/gup.md)を参照）。I/Oが完了したら、ページのピン留めを解除しなければならない（`unpin_user_pages()`）。1M IOPSを出せるNVMeデバイスで、4 KBのバッファが1ページに収まる場合、これは1秒あたり1M回のピン留め/解除のサイクルになる。その1回1回がTLBに優しくないページテーブルのウォークである。

**ファイルディスクリプタの検索**。カーネルは操作のたびに`fdget()`を通じて整数の`fd`を`struct file *`に解決する。`fdget()`はファイルへの参照を取得し、終了時にそれを手放さなければならない。高IOPSでは、ファイルディスクリプタテーブルに対するロック競合が計測可能なレベルになる。

固定バッファと固定ファイルは、登録時にコストを一度だけ支払い、それを後続のすべてのI/Oに分散させることで、これらの問題を解決する。

## 固定バッファ（`IORING_REGISTER_BUFFERS`） {#fixed-buffers-ioring_register_buffers}

### 登録 {#registration}

```c
#include <sys/uio.h>
#include <liburing.h>

#define NR_BUFS  8
#define BUF_SIZE (64 * 1024)  /* バッファあたり64 KB */

static char bufs[NR_BUFS][BUF_SIZE];

struct iovec iovecs[NR_BUFS];
for (int i = 0; i < NR_BUFS; i++) {
    iovecs[i].iov_base = bufs[i];
    iovecs[i].iov_len  = BUF_SIZE;
}

/* すべてのバッファを1回の呼び出しで登録する */
int ret = io_uring_register_buffers(&ring, iovecs, NR_BUFS);
if (ret < 0) {
    perror("io_uring_register_buffers");
    return ret;
}
```

内部では、`io_uring_register_buffers`は次を発行する。

```c
io_uring_register(ring_fd, IORING_REGISTER_BUFFERS, iovecs, nr_iovecs);
```

カーネルのハンドラ（`io_uring/rsrc.c`の`io_sqe_buffers_register`）は各`iovec`を走査し、`pin_user_pages_fast()`を呼んですべてのページをピン留めし（高速パスでない場合は`pin_user_pages()`にフォールバックする）、得られた`struct page **`配列を`io_mapped_ubuf`構造体の中に格納する。io_uringのバッファ登録は`FOLL_GET`（`get_user_pages_fast()`）ではなく`FOLL_PIN`のセマンティクス（`pin_user_pages_fast()`経由）を使う。これはDMAのライフタイムにとって重要である。ピン留めされたページは、ピンが保持されている間は解放も移動もできず、これが長期間にわたるDMA利用を安全に行うために必要なのである。

```c
/* io_uring/rsrc.c（簡略化） */
struct io_mapped_ubuf {
    u64         ubuf;         /* ユーザー空間の仮想アドレスの開始 */
    u64         ubuf_end;     /* 終端（この値を含まない） */
    unsigned    nr_bvecs;     /* struct bio_vecエントリの数 */
    struct bio_vec bvec[];    /* ページごとに1つ。直接DMAで使う */
};
```

これらの`io_mapped_ubuf`ポインタは`io_rsrc_data`テーブルにまとめられ、`io_ring_ctx`に格納される。

```c
/* io_uring/io_uring.h */
struct io_ring_ctx {
    /* ... */
    struct io_rsrc_data     *buf_data;   /* 登録済みバッファのテーブル */
    struct io_rsrc_data     *file_data;  /* 登録済みファイルのテーブル */
    /* ... */
};
```

ページはバッファが登録解除されるまでピン留めされたままであり、`bio_vec`テーブルも有効なままである。I/Oの投入パスではページテーブルのウォークは発生しない。

### SQEで固定バッファを使う {#using-fixed-buffers-in-sqes}

`IORING_OP_READ_FIXED` / `IORING_OP_WRITE_FIXED`を使い、SQEの`buf_index`に登録済みバッファのインデックスを設定する。

```c
/* 関係するSQEのフィールド */
struct io_uring_sqe {
    __u8   opcode;     /* IORING_OP_READ_FIXEDまたはIORING_OP_WRITE_FIXED */
    __u64  addr;       /* 登録済みバッファ内を指すポインタ */
    __u32  len;        /* 転送するバイト数 */
    __u64  off;        /* ファイルオフセット */
    __u16  buf_index;  /* 登録済みバッファテーブルのインデックス */
    /* ... */
};
```

liburingを使う場合:

```c
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);

/* 登録済みバッファ2に、そのバッファの0バイト目から読み込む */
io_uring_prep_read_fixed(sqe, fd, bufs[2], BUF_SIZE, /*offset=*/0, /*buf_index=*/2);
io_uring_sqe_set_data64(sqe, 2);  /* タグ = バッファのインデックス。CQEの照合用 */

io_uring_submit(&ring);
```

`addr`は範囲`[iovecs[buf_index].iov_base, iov_base + iov_len)`内になければならない。カーネルは投入時にこれを検証し、範囲外であれば`-EFAULT`を返す。

### 登録解除 {#unregistering}

```c
io_uring_unregister_buffers(&ring);
/* 次と等価:
   io_uring_register(ring_fd, IORING_UNREGISTER_BUFFERS, NULL, 0); */
```

ピン留めされたページはすべて解放される。登録解除が成功するには、バッファを参照している実行中の操作がすべて完了していなければならない。

## 固定ファイル（`IORING_REGISTER_FILES`） {#fixed-files-ioring_register_files}

### 登録 {#registration_1}

```c
int fds[MAX_CLIENTS];
/* fds[0..n-1]をオープンしたファイルディスクリプタで埋める。空きスロットには-1を使う */
fds[0] = open("data.bin", O_RDONLY | O_DIRECT);
fds[1] = accept(listen_fd, NULL, NULL);
/* 残りのスロットは事前に-1で埋めておく */
for (int i = 2; i < MAX_CLIENTS; i++) fds[i] = -1;

int ret = io_uring_register_files(&ring, fds, MAX_CLIENTS);
if (ret < 0) {
    perror("io_uring_register_files");
    return ret;
}
```

生のシステムコールは次のとおり。

```c
io_uring_register(ring_fd, IORING_REGISTER_FILES, fds, nr_fds);
```

カーネルは各fdをその`struct file *`に解決し、`fget()`を1回呼んでカウント付きの参照を取得し、そのポインタを`io_ring_ctx`上の`io_rsrc_data`ファイルテーブルに格納する。後続の操作は格納されたポインタを直接使い、投入パスでは`fdget()`は呼ばれない。

### SQEで固定ファイルを使う {#using-fixed-files-in-sqes}

`sqe->flags`に`IOSQE_FIXED_FILE`を設定し、登録した配列のインデックスを`sqe->fd`として使う。

```c
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);

/* インデックス0の固定ファイルから読み込む。インデックス0の登録済みバッファを使う */
io_uring_prep_read_fixed(sqe, /*fd=*/0, bufs[0], BUF_SIZE, 0, /*buf_index=*/0);
io_uring_sqe_set_flags(sqe, IOSQE_FIXED_FILE);
io_uring_sqe_set_data64(sqe, 0);

io_uring_submit(&ring);
```

固定ファイルと固定バッファは独立した機能であり、同じSQE内で自由に組み合わせられる。

### 全体を再登録せずにスロットを更新する {#updating-slots-without-full-re-registration}

クライアントが切断し、新しいクライアントが接続してきたときは、テーブル全体を登録解除せずに、該当するスロットをアトミックに更新する。

```c
int new_fd = accept(listen_fd, NULL, NULL);
unsigned slot = find_free_slot();

/* スロット`slot`をnew_fdで置き換える */
int ret = io_uring_register_files_update(&ring, slot, &new_fd, 1);
if (ret < 0)
    perror("io_uring_register_files_update");
```

これは`IORING_REGISTER_FILES_UPDATE`に対応し、実行時にスロット単位で更新できる。スロットを`-1`に設定すると、そのスロットの参照が解放される。

### 登録解除 {#unregistering_1}

```c
io_uring_unregister_files(&ring);
```

## 性能の数値 {#performance-numbers}

固定リソースによる性能上の利点は、IOPSが高くなるほど大きくなる。

| ワークロード | 素の操作 | 固定バッファ | 固定ファイル | 両方 |
|----------|-----------|---------------|-------------|------|
| 4 KBシーケンシャル読み込み、NVMe | ベースライン | スループット+8〜12% | スループット+3〜5% | スループット+12〜18% |
| 4 KBランダム読み込み、NVMe、深さ32 | ベースライン | 操作あたりCPU −20〜30 µs | 操作あたりCPU −5〜10 µs | 操作あたりCPU −30〜40 µs |
| 高並行のソケットrecv | ベースライン | 無視できる程度 | +5〜10% | +8〜15% |

これらの数値は3つの異なる効果によってもたらされる。

1. **TLBへの圧力がなくなる**。`get_user_pages()`中のページテーブルのウォークはTLBエントリを汚し、アプリケーションが使っているキャッシュラインを追い出す。一度だけピン留めすれば、リングのライフタイムの間、この干渉がなくなる。

2. **カーネルのCPU時間が減る**。負荷の高いio_uringワークロードを`perf`でプロファイルすると、カーネルのCPUプロファイルに`get_user_pages_fast`と`unpin_user_pages`が目立って現れる。固定バッファを使うと、これらのシンボルはプロファイルから完全に消える。

3. **アトミック操作が減る**。`fdget()`はアトミックな`atomic_long_inc`でファイルの参照カウントを増やす。100の並行接続で1M IOPSの場合、ファイル参照カウントに対する競合がCPUサイクル数に表れる。固定ファイルは、操作ごとのアトミック操作を登録時の1回の`fget`で置き換える。

この効果が最もよく見えるのは、高速なNVMeやメモリ上のファイルシステム（tmpfs、ramfs）での小さなI/O操作である。そこでは実際のI/O時間に比べてカーネルのオーバーヘッドが大きい。回転ディスクではI/Oのレイテンシが支配的なので、節約は相対的に小さくなる。

## バッファリング（Linux 5.19以降） {#buffer-rings-linux-519}

固定バッファは、アプリケーションが各I/Oに特定のバッファを割り当てる場合にはうまく機能する。しかしソケットの`recv`など、到着するデータのサイズが事前にわからない操作では、アプリケーションはSQEの投入時に安全にバッファを割り当てられない。バッファが十分な大きさでないかもしれないし、数百の実行中の受信がバッファプールを使い果たすかもしれない。

**提供バッファリング**（`IORING_REGISTER_PBUF_RING`、Linux 5.19で追加）は、データが到着した瞬間にカーネルがプールからバッファを選べるようにすることで、この問題を解決する。

### バッファリングのセットアップ {#setting-up-a-buffer-ring}

```c
#include <liburing.h>

#define NUM_BUFS  64
#define BUF_SIZE  4096

/* 共有バッファリングと、その背後のバッファを割り当てる */
struct io_uring_buf_ring *buf_ring;
unsigned ring_mask = io_uring_buf_ring_mask(NUM_BUFS);

/* io_uring_setup_buf_ringはmmapでbuf_ringを割り当てて登録する */
int bgid = 0;  /* バッファグループID */
buf_ring = io_uring_setup_buf_ring(&ring, NUM_BUFS, bgid, 0, &ret);
if (!buf_ring) {
    fprintf(stderr, "io_uring_setup_buf_ring: %s\n", strerror(-ret));
    return ret;
}

/* バッファ本体の背後にあるストレージ */
char *bufs = malloc((size_t)NUM_BUFS * BUF_SIZE);

/* すべてのバッファをリングに追加する */
for (int i = 0; i < NUM_BUFS; i++) {
    io_uring_buf_ring_add(buf_ring,
                          bufs + (size_t)i * BUF_SIZE, /* バッファへのポインタ */
                          BUF_SIZE,                     /* 長さ */
                          i,                            /* バッファID（bid） */
                          ring_mask,
                          i);                           /* リング内のオフセット */
}
io_uring_buf_ring_advance(buf_ring, NUM_BUFS);
```

`io_uring_buf_ring`は、単一生産者（ユーザー空間）/単一消費者（カーネル）のリングである。ユーザー空間は`io_uring_buf_ring_add`でバッファを追加し、`io_uring_buf_ring_advance`でそれをコミットする。カーネルはtailから取り出す。

### バッファ選択付きでrecvを投入する {#submitting-a-recv-with-buffer-selection}

```c
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);

/* バッファはNULL、長さは0: カーネルが選ぶ */
io_uring_prep_recv(sqe, client_fd, NULL, 0, 0);
sqe->buf_group = bgid;  /* どのバッファグループから取るか */
io_uring_sqe_set_flags(sqe, IOSQE_BUFFER_SELECT);
io_uring_sqe_set_data64(sqe, (uint64_t)client_fd);

io_uring_submit(&ring);
```

### 完了の消費とバッファの再利用 {#consuming-completions-and-recycling-buffers}

```c
struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);

if (cqe->res < 0) {
    fprintf(stderr, "recv error: %s\n", strerror(-cqe->res));
} else {
    /* カーネルはどのバッファを選んだか? */
    int bid   = cqe->flags >> IORING_CQE_BUFFER_SHIFT;
    int bytes = cqe->res;
    char *data = bufs + (size_t)bid * BUF_SIZE;

    process_data(data, bytes);

    /* 再利用のためにバッファをリングに返す */
    io_uring_buf_ring_add(buf_ring, data, BUF_SIZE, bid, ring_mask, 0);
    io_uring_buf_ring_advance(buf_ring, 1);
}
io_uring_cqe_seen(&ring, cqe);
```

バッファが選択された場合は常に`cqe->flags`に`IORING_CQE_F_BUFFER`がセットされる。セットされていない場合（例: エラー時）、`bid`は意味を持たず、バッファは消費されていない。

下位の登録システムコールは次のとおり。

```c
struct io_uring_buf_reg reg = {
    .ring_addr    = (uint64_t)buf_ring,
    .ring_entries = NUM_BUFS,           /* 2の累乗でなければならない */
    .bgid         = bgid,
};
io_uring_register(ring_fd, IORING_REGISTER_PBUF_RING, &reg, 1);
```

登録解除するには:

```c
io_uring_free_buf_ring(&ring, buf_ring, NUM_BUFS, bgid);
/* IORING_UNREGISTER_PBUF_RINGと等価 */
```

## 完全なCの例 {#complete-c-example}

次の例では、バッファとファイルの両方を登録し、`IORING_OP_READ_FIXED`を発行し、後始末を行う。

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <liburing.h>

#define BUF_SIZE  (64 * 1024)   /* 64 KB */
#define NR_BUFS   4
#define QUEUE_DEPTH 8

int main(int argc, char *argv[])
{
    if (argc < 2) {
        fprintf(stderr, "usage: %s <file>\n", argv[0]);
        return 1;
    }

    /* --- ファイルを開く --- */
    int fd = open(argv[1], O_RDONLY | O_DIRECT);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    /* --- リングを初期化する --- */
    struct io_uring ring;
    int ret = io_uring_queue_init(QUEUE_DEPTH, &ring, 0);
    if (ret < 0) {
        fprintf(stderr, "io_uring_queue_init: %s\n", strerror(-ret));
        return 1;
    }

    /* --- バッファを登録する（O_DIRECT用にアラインする） --- */
    static char raw[NR_BUFS][BUF_SIZE] __attribute__((aligned(4096)));
    struct iovec iovecs[NR_BUFS];
    for (int i = 0; i < NR_BUFS; i++) {
        iovecs[i].iov_base = raw[i];
        iovecs[i].iov_len  = BUF_SIZE;
    }
    ret = io_uring_register_buffers(&ring, iovecs, NR_BUFS);
    if (ret < 0) {
        fprintf(stderr, "io_uring_register_buffers: %s\n", strerror(-ret));
        return 1;
    }

    /* --- ファイルを登録する --- */
    ret = io_uring_register_files(&ring, &fd, 1);
    if (ret < 0) {
        fprintf(stderr, "io_uring_register_files: %s\n", strerror(-ret));
        return 1;
    }
    /* これでfdは固定ファイルのインデックス0としてアクセスできる。整数のfdは
       I/Oにはもう不要である（ただしclose()には依然として必要）。 */

    /* --- 固定readを投入する --- */
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    if (!sqe) {
        fprintf(stderr, "ring full\n");
        return 1;
    }

    /*
     * io_uring_prep_read_fixed(sqe, fd_or_index, buf, nbytes, offset, buf_index)
     *   - fd_or_index: 0 = 固定ファイルのインデックス0
     *   - buf:         登録済みバッファ0内を指すポインタ
     *   - buf_index:   0 = 登録済みバッファ0
     */
    io_uring_prep_read_fixed(sqe,
                             /*fd*/        0,
                             /*buf*/       raw[0],
                             /*nbytes*/    BUF_SIZE,
                             /*offset*/    0,
                             /*buf_index*/ 0);
    io_uring_sqe_set_flags(sqe, IOSQE_FIXED_FILE);
    io_uring_sqe_set_data64(sqe, 42);   /* 任意のユーザータグ */

    ret = io_uring_submit(&ring);
    if (ret < 0) {
        fprintf(stderr, "io_uring_submit: %s\n", strerror(-ret));
        return 1;
    }

    /* --- 完了を待つ --- */
    struct io_uring_cqe *cqe;
    ret = io_uring_wait_cqe(&ring, &cqe);
    if (ret < 0) {
        fprintf(stderr, "io_uring_wait_cqe: %s\n", strerror(-ret));
        return 1;
    }

    if (cqe->res < 0) {
        fprintf(stderr, "read error: %s\n", strerror(-cqe->res));
    } else {
        printf("read %d bytes (tag=%" PRIu64 ")\n",
               cqe->res, io_uring_cqe_get_data64(cqe));
        /* raw[0][0..cqe->res-1]にデータが入っている */
    }
    io_uring_cqe_seen(&ring, cqe);

    /* --- 後始末 --- */
    io_uring_unregister_buffers(&ring);
    io_uring_unregister_files(&ring);
    io_uring_queue_exit(&ring);
    close(fd);
    return 0;
}
```

コンパイル方法:

```bash
gcc -O2 -o fixed_read fixed_read.c -luring
```

この例の要点:

- `sqe->fd`が本物のファイルディスクリプタではなく登録済みファイルのインデックスである場合は、必ず`IOSQE_FIXED_FILE`を設定しなければならない。
- `io_uring_prep_read_fixed`の`buf_index`は、`io_uring_register_buffers`に渡した`iovecs[]`配列のインデックスと一致しなければならない。
- `addr`（`buf`引数）は`buf_index`の登録済み領域内になければならない。`iovecs[0].iov_base == raw[0]`なので、`buf_index=0`で`raw[0]`を渡せばこれを満たす。
- 登録解除の順序は問わないが、どちらも`io_uring_queue_exit`の前に行うべきである。

## 参考文献 {#further-reading}

- [アーキテクチャとリング](io-uring-arch.md): リングレイアウト、SQE/CQE構造体、mmapのセットアップ
- [操作と高度な機能](io-uring-ops.md): サポートされるオペコード、SQPOLL、リンクされたリクエスト
- カーネルツリーの`io_uring/rsrc.c`: `io_sqe_buffers_register`、`io_sqe_files_register`、`io_mapped_ubuf`
- `io_uring/rsrc.h`: `io_rsrc_data`、`io_rsrc_node`構造体
- `liburing/src/register.c`: すべての`io_uring_register`操作のユーザー空間ラッパー
- `liburing/test/fixed-buffers.c`: 固定バッファのエッジケースに関するliburingのテストスイートの例
