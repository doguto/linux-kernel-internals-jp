# bioとrequestの構造体 {#bio-and-request-structures}

> ブロック層でI/O操作を記述するオブジェクト

## struct bio: 発行の単位 {#struct-bio-the-submission-unit}

`bio`（Block I/O）は基本となるI/O記述子である。scatter-gather I/O操作、つまりブロックデバイス上の連続したセクタにマッピングされた1つ以上の物理メモリセグメントを記述する。

```c
/* include/linux/blk_types.h */
struct bio {
    struct bio          *bi_next;     /* request内でbioを連結するため */
    struct block_device *bi_bdev;     /* 対象のブロックデバイス */
    blk_opf_t            bi_opf;      /* 操作 + フラグ (REQ_OP_READ/WRITE など) */
    blk_status_t         bi_status;   /* 完了ステータス */

    /* scatter-gatherリスト */
    struct bio_vec      *bi_io_vec;   /* (page, offset, len) の配列 */
    struct bvec_iter     bi_iter;     /* bi_io_vec内の現在位置 */

    bio_end_io_t        *bi_end_io;   /* 完了コールバック */
    void                *bi_private;  /* 呼び出し元のプライベートデータ */
    atomic_t             __bi_remaining; /* 分割用: 残っているサブbioの数 */
};
```

### struct bio_vec: 1つのscatter-gatherセグメント {#struct-bio_vec-one-scatter-gather-segment}

```c
/* include/linux/bvec.h */
struct bio_vec {
    struct page  *bv_page;    /* データを含むpage */
    unsigned int  bv_len;     /* バイト数 */
    unsigned int  bv_offset;  /* page内のバイトオフセット */
};
```

各`bio_vec`はメモリの連続した塊（1ページ以下）を記述する。複数の`bio_vec`を持つbioはscatter-gather操作である。

```
bio->bi_io_vec:
  [0]: page A, offset 0, len 4096    → sectors 1024..1031
  [1]: page B, offset 2048, len 2048  → sectors 1032..1035
  [2]: page C, offset 0, len 4096    → sectors 1036..1043

All map to contiguous sectors on disk.
```

### bio_vecの反復処理 {#bio_vec-iteration}

```c
/* bio内のすべてのセグメントを反復処理する */
struct bvec_iter iter;
struct bio_vec bvec;

bio_for_each_segment(bvec, bio, iter) {
    /* bvec.bv_page, bvec.bv_offset, bvec.bv_len */
    void *addr = page_address(bvec.bv_page) + bvec.bv_offset;
    /* addrにあるbvec.bv_lenバイトを処理する */
}
```

## bioの操作（bi_opf） {#bio-operations-bi_opf}

```c
/* 操作の種類 */
REQ_OP_READ        /* デバイスからデータを読み込む */
REQ_OP_WRITE       /* デバイスにデータを書き込む */
REQ_OP_FLUSH       /* デバイスの書き込みキャッシュをフラッシュする */
REQ_OP_DISCARD     /* セクタを破棄する（SSDのTRIM） */
REQ_OP_ZONE_RESET  /* ゾーンデバイスのゾーンをリセットする */
REQ_OP_WRITE_ZEROES /* セクタをゼロで埋める */

/* 操作を修飾するフラグ */
REQ_SYNC      /* 呼び出し元が完了を待っている */
REQ_META      /* メタデータI/O（優先度が高い） */
REQ_PRIO      /* 優先I/O */
REQ_NOMERGE   /* 隣接するrequestとマージしない */
REQ_IDLE      /* ヒント: キューがアイドルのときに発行 */
REQ_FUA       /* force unit access（書き込みキャッシュをバイパス） */
REQ_PREFLUSH  /* この書き込みの前にフラッシュを発行 */
REQ_RAHEAD    /* 先読み。逼迫時には破棄されうる */
```

## bioの割り当てと発行 {#allocating-and-submitting-a-bio}

```c
#include <linux/bio.h>

/* bioを割り当てる: Linux 5.18以降、bdevとopを直接渡す */
struct bio *bio = bio_alloc(bdev, nr_vecs, REQ_OP_WRITE, GFP_KERNEL);

bio->bi_iter.bi_sector = sector;   /* 開始セクタ（512バイト単位） */
bio->bi_end_io = my_bio_complete;  /* 完了コールバック */
bio->bi_private = my_data;

/* bioにページを追加する */
bio_add_page(bio, page, len, offset);

/* 発行 */
submit_bio(bio);

/* 完了コールバック */
static void my_bio_complete(struct bio *bio)
{
    if (bio->bi_status != BLK_STS_OK)
        handle_error(bio->bi_status);
    /* リソースを解放する */
    bio_put(bio);
}
```

## struct request: スケジューリングの単位 {#struct-request-the-scheduled-unit}

`bio`がファイルシステムからの発行単位であるのに対し、`struct request`はI/Oスケジューラが扱う単位である。ディスク上で隣接している複数のbioは、1つのrequestに**マージ**できる。

```c
/* include/linux/blkdev.h */
struct request {
    struct request_queue *q;        /* キュー */
    struct blk_mq_ctx    *mq_ctx;   /* ソフトウェアキューのコンテキスト */
    struct blk_mq_hw_ctx *mq_hctx; /* ハードウェアキューのコンテキスト */
    blk_opf_t             cmd_flags; /* REQ_OP_* + フラグ */
    struct bio            *bio;      /* チェーン内の最初のbio */
    struct bio            *biotail;  /* チェーン内の最後のbio */
    unsigned int          __data_len; /* 合計バイト数 */
    sector_t              __sector;  /* 開始セクタ */
    unsigned int          nr_phys_segments; /* scatter-gatherセグメント数 */
    /* ... tag, timeout, stats ... */
};
```

### bioのマージ {#bio-merging}

I/Oスケジューラは、ディスパッチ前に隣接するbioを1つのrequestにマージしようとする。

```
Before merging:
  Request A: sectors 1000-1007 (write)
  Bio:        sectors 1008-1015 (write to same device)

After merging:
  Request A: sectors 1000-1015 (merged write)
  → fewer interrupts, larger DMA transfers, better throughput
```

マージはハードウェアの制限によって制約される。

- `queue->limits.max_sectors` — 最大リクエストサイズ
- `queue->limits.max_segments` — scatter-gatherセグメントの最大数
- `queue->limits.max_segment_size` — セグメントあたりの最大バイト数

## bioの分割 {#bio-splitting}

bioがハードウェアの制限を超えると、ブロック層はそれを分割する。

```c
/* bio_split: 先頭から'sectors'セクタの位置でbioを分割する */
struct bio *split = bio_split(bio, sectors, GFP_NOIO, &q->bio_split);
bio_chain(split, bio);   /* 両者をリンクする: splitはbioより先に完了する */
submit_bio(split);
submit_bio(bio);
```

分割はファイルシステムからは透過的である — 完了コールバックは、すべてのサブbioが完了したときにだけ呼ばれる。

## 参考文献 {#further-reading}

- [ブロック層の概要](block-overview.md) — bioがスタックのどこに位置するか
- [blk-mq](blk-mq.md) — requestがどのようにハードウェアにディスパッチされるか
- [DMA](../mm/dma.md) — bioのページがコントローラ向けにどのようにDMAマッピングされるか
