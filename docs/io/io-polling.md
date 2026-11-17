# I/Oポーリング: HIPRI、IOPOLL、NOWAIT {#io-polling-hipri-iopoll-and-nowait}

> NVMe I/Oから割り込みとスレッド切り替えのオーバーヘッドを取り除く — IOCB_HIPRI、io_uringのIOPOLL、IOCB_NOWAITがカーネル内部でどのように動作するか

---

## なぜポーリングか {#why-polling}

従来のブロックI/Oは割り込みを使う。書き込みや読み出しが完了すると、NVMeコントローラはMSI-X割り込みを発行する。CPUは実行中の処理を中断して状態を保存し、割り込みハンドラを実行して完了を通知し、元の処理に戻る。数ミリ秒のレイテンシを持つ回転ディスクでは、このオーバーヘッドは無視できる。しかしレイテンシが100µsを切る最新のNVMe SSDでは、割り込み経路のコストが無視できなくなってくる。

50µsのNVMe読み出しで、割り込み経路上で何が起きているかを分解してみる。

```
Application thread submits I/O
  → syscall entry overhead                     ~200 ns
  → VFS, filesystem, block layer               ~1–2 µs
  → NVMe submission queue ring doorbell write  ~100 ns
  [NVMe controller processes request]          ~45–50 µs
  → MSI-X interrupt fires on some CPU
  → interrupt handler: nvme_irq()              ~1–2 µs
  → softirq: blk_done_softirq()               ~1–2 µs
  → bio end_io → kiocb completion             ~500 ns
  → wakeup sleeping thread                    ~2–5 µs
  → context switch back to application        ~2–5 µs
Total overhead (interrupt + wakeup path):     ~7–15 µs on top of device latency
```

レイテンシ50µsのデバイスでは、このオーバーヘッドは観測される総レイテンシの15〜30%に相当する。p99のテールレイテンシでは問題はさらに深刻になる。割り込みコアレッシング、CPUのCステート、スケジューラのジッタのすべてが起床経路を膨らませるからである。

ポーリングは割り込みを完全に取り除く。

```
Application thread submits I/O
  → syscall entry overhead                     ~200 ns
  → VFS, filesystem, block layer               ~1–2 µs
  → NVMe submission queue ring doorbell write  ~100 ns
  [NVMe controller processes request]          ~45–50 µs
  → kernel spins reading NVMe CQ              ~100–500 ns per check
  → CQ entry appears: completion inline        ~500 ns
Total overhead:                               ~2–4 µs on top of device latency
```

トレードオフはCPUである。ポーリングするコアは待機中にアイドル状態になれない。すべてのI/Oの間、1つのコアが100%ビジーになる。これは次のような場合には妥当なトレードオフである。

- **レイテンシに敏感なワークロード**: データベースのトランザクションコミット、キーバリューストア、リアルタイム分析
- **高IOPSのワークロード**: ポーリングするコアが常にすぐ完了を見つけるため、有用な仕事をしている
- **テールレイテンシのSLO**: p99やp999を膨らませる割り込みのジッタを取り除く

一方、次のような場合には割に合わない。

- **低IOPSのワークロード**: コアはほとんどの時間、何もせずにスピンする
- **共有システム**: ポーリングするコアが他のテナントから奪われる
- **回転メディアや低速なSSD**: レイテンシが十分に大きいため、スリープする方が常に有利

### 割り込み経路とポーリング経路 {#interrupt-path-vs-polling-path}

```
Interrupt path:
┌─────────────┐    doorbell     ┌──────────┐    MSI-X      ┌──────────────┐
│  submitter  │ ─────────────▶ │   NVMe   │ ────────────▶ │  interrupt   │
│   (asleep)  │                │ controller│               │   handler    │
└─────────────┘                └──────────┘               └──────┬───────┘
       ▲                                                          │ wakeup
       └──────────────────────────────────────────────────────────┘
       (context switch back)

Polling path:
┌─────────────┐    doorbell     ┌──────────┐
│  submitter  │ ─────────────▶ │   NVMe   │
│  (spinning) │ ◀── CQ check ─ │ controller│
│             │ ◀── CQ check ─ │          │
│             │ ◀── CQ found ─ │          │  ← completion visible in CQ
└─────────────┘                └──────────┘
(no interrupt, no context switch)
```

---

## HIPRI: カーネルのポーリングインターフェース {#hipri-the-kernel-polling-interface}

ポーリングのユーザーから見える入口は、`preadv2()`/`pwritev2()`の`RWF_HIPRI`フラグ、またはio_uringリングの`IORING_SETUP_IOPOLL`である。どちらも最終的にはkiocbの`ki_flags`に`IOCB_HIPRI`をセットする。

### ユーザー空間API {#userspace-api}

```c
/* RWF_HIPRI付きのpreadv2/pwritev2: 呼び出し単位のポーリング要求 */
#include <sys/uio.h>

struct iovec iov = { .iov_base = buf, .iov_len = 4096 };
/* RWF_HIPRI = 0x1: この呼び出しで高優先度ポーリングを要求する */
ssize_t n = preadv2(fd, &iov, 1, offset, RWF_HIPRI);

/* io_uring: リング全体のIOPOLLモード */
struct io_uring_params params = {};
params.flags |= IORING_SETUP_IOPOLL;
int ring_fd = io_uring_setup(128, &params);
```

どちらも`O_DIRECT`を必要とする。バッファードファイルに対するHIPRIは拒否される。ブロック層にはページキャッシュの充填をポーリングする手段がないからである。

### RWF_HIPRIがIOCB_HIPRIになるまで {#how-rwf_hipri-becomes-iocb_hipri}

`preadv2()`や`pwritev2()`に渡された`RWF_*`フラグは、そのまま`IOCB_*`フラグにコピーされる。値は意図的に同じにしてある。

```c
/* include/uapi/linux/fs.h */
#define RWF_HIPRI   ((__force __kernel_rwf_t)0x00000001)
#define RWF_NOWAIT  ((__force __kernel_rwf_t)0x00000008)

/* include/linux/fs.h */
#define IOCB_HIPRI  (__force int) RWF_HIPRI    /* = 0x1 */
#define IOCB_NOWAIT (__force int) RWF_NOWAIT   /* = 0x8 */
```

VFSの入口では次のようになる。

```c
/* fs/read_write.c */
static ssize_t do_iter_readv_writev(struct file *filp, struct iov_iter *iter,
                                     loff_t *ppos, int type, rwf_t flags)
{
    struct kiocb kiocb;
    init_sync_kiocb(&kiocb, filp);

    /* RWF_*フラグをIOCB_*にコピーする — 値は同一 */
    ret = kiocb_set_rw_flags(&kiocb, flags);
    if (ret)
        return ret;
    /* RWF_HIPRIが渡されていれば、kiocb.ki_flagsにはIOCB_HIPRIがセットされている */
    ...
}
```

### HIPRIはbioまで伝わる {#hipri-flows-through-to-the-bio}

kiocbに`IOCB_HIPRI`がセットされると、それはbioへ、さらにブロック層のリクエストへと伝播する。

```c
/* fs/iomap/direct-io.c — iomapのDIO経路 */
static void iomap_dio_submit_bio(const struct iomap_iter *iter,
        struct iomap_dio *dio, struct bio *bio, loff_t pos)
{
    struct kiocb *iocb = dio->iocb;

    atomic_inc(&dio->ref);

    /* 同期dioは確実にはポーリングできない */
    if ((iocb->ki_flags & IOCB_HIPRI) && !is_sync_kiocb(iocb)) {
        bio->bi_opf |= REQ_POLLED;         /* bioにポーリング対象の印を付ける */
        WRITE_ONCE(iocb->private, bio);    /* ->iopollが見つけられるよう保存しておく */
    }

    if (dio->dops && dio->dops->submit_io)
        dio->dops->submit_io(iter, bio, pos);
    else
        submit_bio(bio);
}
```

ブロック層では、`REQ_POLLED`のbioは**ポーリング用ハードウェアキュー**に振り分けられる。これは割り込みが割り当てられていない、別のハードウェアキューの集合（`HCTX_TYPE_POLL`）である。ドライバは自身のキューの一部をポーリング用に切り出すことでこれに参加する。たとえばNVMeは、キュープールの一部をポーリング用マップに割り当て、その数は`nvme.poll_queues`モジュールパラメータで制御される。ブロック層はデバイスごとに最大3つのキューマップ（`HCTX_TYPE_DEFAULT`、`HCTX_TYPE_READ`、`HCTX_TYPE_POLL`）を保持し、`REQ_POLLED`のリクエストはポーリング用マップにディスパッチされる。

ポーリング用キューを割り込みなしに保つことが、ポーリングを安全にしている。ポーリング用キューの完了リングには割り込みハンドラが一切触れないため、ポーリングするスレッドはIRQ経路と競合することなくリングを読み進められる。もしポーリングするI/OとしないI/Oがキューを共有していれば、ポーリングループと割り込みハンドラの両方が同じ完了を刈り取ろうとしかねない。

---

## iopollフック: ファイルシステムレベルのポーリング {#the-iopoll-hook-filesystem-level-polling}

`file_operations`には`iopoll`メソッドがあり、io_uringはスリープせずに完了を確認するためにこれを呼び出す。

```c
/* include/linux/fs.h */
struct file_operations {
    ...
    int (*iopoll)(struct kiocb *kiocb, struct io_comp_batch *iob,
                  unsigned int flags);
    ...
};
```

### iocb_bio_iopoll: 共通実装 {#iocb_bio_iopoll-the-common-implementation}

ext4とXFSはどちらもiomap層に処理を委ねる。iomap層は発行時にbioのポインタを`kiocb->private`に保存する。

```c
/* DIOの発行時に、元のbioがkiocbに保存される */
kiocb->private = bio;

/* block/blk-core.c — ext4、XFS、f2fsなどが共有する->iopollハンドラ */
int iocb_bio_iopoll(struct kiocb *kiocb, struct io_comp_batch *iob,
                    unsigned int flags)
{
    struct bio *bio;
    int ret = 0;

    rcu_read_lock();
    bio = READ_ONCE(kiocb->private);   /* I/Oが既に完了していればNULL */
    if (bio)
        ret = bio_poll(bio, iob, flags);
    rcu_read_unlock();

    return ret;
}
EXPORT_SYMBOL_GPL(iocb_bio_iopoll);
```

`bio_poll`関数はブロック層へと降りていく。

```c
/* block/blk-core.c */
int bio_poll(struct bio *bio, struct io_comp_batch *iob, unsigned int flags)
{
    blk_qc_t cookie = READ_ONCE(bio->bi_cookie);
    struct request_queue *q;
    int ret = 0;

    if (!bio->bi_bdev)
        return 0;
    q = bdev_get_queue(bio->bi_bdev);
    if (cookie == BLK_QC_T_NONE)         /* ポーリング付きの発行ではない */
        return 0;

    blk_flush_plug(current->plug, false);

    /* hctxとリクエストが有効なままとなるよう、キュー使用の参照を取る */
    if (!percpu_ref_tryget(&q->q_usage_counter))
        return 0;
    if (queue_is_mq(q))
        ret = blk_mq_poll(q, cookie, iob, flags);   /* NVMe: CQを確認する */
    blk_queue_exit(q);
    return ret;
}
```

`cookie`は、発行時にbioに記録されたハードウェアキューの識別子である。`blk_qc_t`の改修（5.16）以降、これは単にハードウェアキューの番号になっているため、`blk_mq_poll()`は正しいキュー、つまり正しいNVMe完了キューへ直接たどり着いて完了を確認できる。

---

## NVMeのポーリング: 完了キューを直接読む {#nvme-polling-reading-the-completion-queue-directly}

NVMeコントローラはキューごとに一対のリングを使う。ドライバがコマンドを書き込むサブミッションキュー（SQ）と、コントローラが完了を書き込む完了キュー（CQ）である。ドライバは**フェーズビット**を確認することで新しいCQエントリを検出する。これはコントローラがリングを一周するたびに反転させる1ビットである。

```c
/* drivers/nvme/host/pci.c */
static inline bool nvme_cqe_pending(struct nvme_queue *nvmeq)
{
    struct nvme_completion *hd = cq_head(nvmeq);
    return (le16_to_cpu(hd->status) & 1) == nvmeq->cq_phase;
}

static int nvme_poll(struct blk_mq_hw_ctx *hctx, struct io_comp_batch *iob)
{
    struct nvme_queue *nvmeq = hctx->driver_data;
    bool found;

    if (!test_bit(NVMEQ_POLLED, &nvmeq->flags) ||
        !nvme_cqe_pending(nvmeq))
        return 0;

    /*
     * ポーリングロック: 複数のポーリング呼び出し元（例: io_uringリングを
     * 共有するスレッド群）は、エントリの二重完了を避けるためにCQ処理を
     * 直列化しなければならない。
     */
    spin_lock(&nvmeq->cq_poll_lock);
    found = nvme_poll_cq(nvmeq, iob);
    spin_unlock(&nvmeq->cq_poll_lock);

    return found;
}
```

`nvme_poll_cq`は保留中のCQエントリをたどり、各リクエストを完了させ、ヘッドポインタを更新する。フェーズビットの確認は1回のメモリ読み出しにすぎず、MMIOもPCIトランザクションも伴わない。ポーリングがこれほど速いのはこのためである。完了の検出は、コントローラが書き込むDMAマップされたメモリ領域からのロード1回で済む。

### フェーズビットの仕組み {#phase-bit-mechanics}

```
CQ ring (4 entries):
                         ┌───────────┬───────────┬───────────┬───────────┐
  entries:               │  phase=1  │  phase=1  │  phase=0  │  phase=0  │
                         └───────────┴───────────┴───────────┴───────────┘
                                                 ▲
                                              cq_head (next to check)

  nvme_cqe_pending() checks: entry[cq_head].phase == nvmeq->cq_phase
  When controller posts completion: it writes status with toggled phase bit
  Driver detects it: phase bit at cq_head matches expected phase → new entry
```

割り込み経路でも、ドライバは同じ`nvme_poll_cq()`の処理を行う。ただし、ポーリングループではなく割り込みハンドラ（`nvme_irq()`）によって起動される。`nvme_poll_cq()`以降のコードパスは共通である。

---

## io_uringのIOPOLLモード {#io_uring-iopoll-mode}

io_uringの`IORING_SETUP_IOPOLL`は、最も高性能なポーリングインターフェースである。これは次の要素を組み合わせている。

1. 非同期の発行（I/Oごとのシステムコールオーバーヘッドがない）
2. カーネル側での完了ポーリング（割り込みも起床もない）
3. オプションの発行側ポーリング（`IORING_SETUP_SQPOLL`、後述）

### セットアップ {#setup}

```c
#include <liburing.h>

struct io_uring ring;
struct io_uring_params params = {};

params.flags = IORING_SETUP_IOPOLL;
/* オプション: サブミッションキューもポーリングする */
/* params.flags |= IORING_SETUP_SQPOLL; */

io_uring_queue_init_params(128, &ring, &params);

/* IOPOLLにはO_DIRECTが必須 */
int fd = open("/dev/nvme0n1", O_RDWR | O_DIRECT);

/* 固定バッファを登録する（IOPOLLでは強く推奨） */
struct iovec iov[1] = {{ .iov_base = buf, .iov_len = BUF_SIZE }};
io_uring_register_buffers(&ring, iov, 1);
```

### 発行と完了 {#submission-and-completion}

```c
/* 登録済み（固定）バッファを使って読み出しを発行する */
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read_fixed(sqe, fd, buf, BUF_SIZE, offset, 0 /* buf_index */);
io_uring_submit(&ring);

/* 完了を待つ: カーネルはio_do_iopoll()内でビジーウェイトする */
struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);   /* io_uring_enter(IORING_ENTER_GETEVENTS)を呼ぶ */
int result = cqe->res;
io_uring_cqe_seen(&ring, cqe);
```

### io_do_iopoll()の内部 {#inside-io_do_iopoll}

IOPOLLリングに対して`IORING_ENTER_GETEVENTS`付きで`io_uring_enter()`が呼ばれると、カーネルはポーリングループに入る。

```c
/* io_uring/rw.c */
int io_do_iopoll(struct io_ring_ctx *ctx, bool force_nonspin)
{
    DEFINE_IO_COMP_BATCH(iob);
    struct io_kiocb *req, *tmp;
    unsigned int poll_flags = 0;
    int nr_events = 0;

    if (ctx->poll_multi_queue || force_nonspin)
        poll_flags |= BLK_POLL_ONESHOT;

    /* 処理中の各リクエストのファイルをポーリングする（その->iopollメソッド） */
    list_for_each_entry(req, &ctx->iopoll_list, iopoll_node) {
        int ret;

        if (READ_ONCE(req->iopoll_completed))
            break;

        ret = io_uring_classic_poll(req, &iob, poll_flags);  /* → file->f_op->iopoll */
        if (unlikely(ret < 0))
            return ret;
        else if (ret)
            poll_flags |= BLK_POLL_ONESHOT;

        if (!rq_list_empty(&iob.req_list) ||
            READ_ONCE(req->iopoll_completed))
            break;
    }

    if (!rq_list_empty(&iob.req_list))
        iob.complete(&iob);

    /* 2回目のパス: 完了したリクエストを刈り取り、CQEをキューに入れる */
    list_for_each_entry_safe(req, tmp, &ctx->iopoll_list, iopoll_node) {
        if (!smp_load_acquire(&req->iopoll_completed))
            continue;
        list_del(&req->iopoll_node);
        wq_list_add_tail(&req->comp_list, &ctx->submit_state.compl_reqs);
        nr_events++;
    }

    if (nr_events)
        __io_submit_flush_completions(ctx);
    return nr_events;
}
```

このループは`ctx->iopoll_list`をたどり、要求されたすべての完了が届くまで各ファイルの`iopoll`メソッドを呼び出す。`io_uring_enter()`内の呼び出し元は、`min_complete`個のCQEが得られるまでこれを繰り返す。

```c
/* io_uring/io_uring.c */
static int io_iopoll_check(struct io_ring_ctx *ctx, unsigned int min_events)
{
    /* 既に完了が保留されているなら、スピンする必要はない */
    if (io_cqring_events(ctx))
        return 0;

    do {
        int ret = io_do_iopoll(ctx, !min_events);
        if (unlikely(ret < 0))
            return ret;
        if (need_resched())
            break;
    } while (io_cqring_events(ctx) < min_events);

    return 0;
}
```

### IOPOLLモードにおけるリクエストのライフサイクル {#request-lifecycle-in-iopoll-mode}

```
io_uring_submit()
  → io_submit_sqes()
    → io_issue_sqe()        ← submit the I/O with IOCB_HIPRI set
      → vfs_iocb_iter_read() / vfs_iocb_iter_write()
        → file->f_op->read_iter / write_iter
          → iomap_dio_rw()  ← submits bio with REQ_POLLED set
            → iocb->private = bio  ← stash bio for iopoll
      → req added to ctx->iopoll_list  ← tracked for polling

io_uring_enter(IORING_ENTER_GETEVENTS)
  → io_iopoll_check()
    → io_do_iopoll()
      → file->f_op->iopoll()  ← per-request: reads NVMe CQ
        → bio_poll()
          → nvme_poll()       ← reads phase bit, processes CQ entry
      → reap completed reqs   ← post CQEs, remove from iopoll_list
```

---

## SQPOLL: カーネル側での発行ポーリング {#sqpoll-kernel-side-submission-polling}

`IORING_SETUP_SQPOLL`はIOPOLLと組み合わせて使う機能で、発行側のシステムコールを取り除く。SQPOLLでは、専用のカーネルスレッドがSQリングをポーリングする。

```c
/* SQPOLL + IOPOLL: I/Oあたりのシステムコールはゼロ */
params.flags = IORING_SETUP_IOPOLL | IORING_SETUP_SQPOLL;
params.sq_thread_idle = 2000;  /* SQPOLLスレッドがスリープするまでのミリ秒 */
io_uring_queue_init_params(128, &ring, &params);
```

SQPOLLスレッド（`io_sq_thread()`）は、CPUにバインドされたリアルタイムのカーネルスレッドとして動作する。

```c
/* io_uring/sqpoll.c */
static int io_sq_thread(void *data)
{
    struct io_sq_data *sqd = data;
    struct io_ring_ctx *ctx;

    while (!kthread_should_stop()) {
        bool cap_entries = !list_is_singular(&sqd->ctx_list);

        /*
         * このSQPOLLスレッドに関連付けられたすべてのリングで新しい
         * SQEを確認する。見つかったものはすべて発行する。
         */
        list_for_each_entry(ctx, &sqd->ctx_list, sqd_list)
            io_submit_sqes(ctx, cap_entries ? IORING_SQPOLL_CAP_ENTRIES_VALUE
                                            : UINT_MAX);

        if (list_empty(&sqd->ctx_list) ||
            (!io_sqring_entries(ctx) && time_after(jiffies, timeout))) {
            /* 仕事がない: ユーザー空間がSQリングに書き込むまでスリープする */
            set_current_state(TASK_INTERRUPTIBLE);
            schedule_timeout(usecs_to_jiffies(sqd->sq_thread_idle * 1000));
        }
    }
    return 0;
}
```

SQPOLLとIOPOLLの両方を有効にすると、ユーザー空間のアプリケーションは次のように動く。

1. 共有SQリングにSQEを書き込む（システムコールなし）
2. SQPOLLカーネルスレッドがそれを拾い上げて発行する
3. `io_uring_enter(IORING_ENTER_GETEVENTS)`を呼んで完了を回収する（結果のバッチ1つにつきシステムコール1回）
4. あるいは、`IORING_FEAT_NODROP`がセットされていればCQリングから直接CQEを読む

### SQPOLL + IOPOLLの要件 {#requirements-for-sqpoll-iopoll}

| 要件 | 理由 |
|-------------|--------|
| `O_DIRECT` | バッファードI/Oにはポーリングフックがない |
| NVMe（または類似の）デバイス | ポーリングには`mq_ops->poll`の実装が必要 |
| 固定バッファ（`io_uring_register_buffers`） | I/Oごとの`get_user_pages`のオーバーヘッドを避ける |
| 固定ファイル（`io_uring_register_files`） | I/Oごとの`fdget`のオーバーヘッドを避ける |
| `CAP_SYS_NICE`または特権プロセス | SQPOLLスレッドには高い優先度が必要 |

---

## IOCB_NOWAIT: ブロックせずに即座に失敗する {#iocb_nowait-fail-fast-instead-of-block}

`IOCB_NOWAIT`は概念的に`IOCB_HIPRI`とは異なる。完了をポーリングするのではなく、NOWAITは「このI/Oが何らかの理由で待たなければならないなら、代わりに即座に`-EAGAIN`を返せ」と指示する。

```c
/* RWF_NOWAIT: 呼び出し単位のノンブロッキングI/O */
ssize_t n = preadv2(fd, &iov, 1, offset, RWF_NOWAIT);
if (n == -EAGAIN) {
    /* I/Oがブロックする — フォールバックするか、後で処理するためキューに入れる */
}

/* io_uringはO_DIRECTファイルに対して自動的にIOCB_NOWAITをセットする */
/* 特別なフラグは不要 — io_uringのデフォルトの挙動である */
```

### NOWAITがEAGAINを返す原因 {#what-causes-nowait-to-return-eagain}

呼び出し元のコンテキストがスリープしなければならなくなる操作はすべて、NOWAITの高速パスを発動させる。

```c
/* inodeロックの競合 */
if (iocb->ki_flags & IOCB_NOWAIT) {
    if (!inode_trylock_shared(inode))
        return -EAGAIN;   /* 誰かがi_rwsemを排他的に保持している */
}

/* ページがキャッシュにない（バッファード読み出し） */
if (iocb->ki_flags & IOCB_NOWAIT) {
    folio = filemap_get_folio(mapping, index);
    if (!folio || !folio_test_uptodate(folio))
        return -EAGAIN;   /* ページを埋めるにはディスクI/Oが必要 */
}

/* folioロックの競合（バッファード書き込み） */
if (iocb->ki_flags & IOCB_NOWAIT) {
    if (!folio_trylock(folio))
        return -EAGAIN;   /* 書き込み側がfolioロックを保持している */
}

/* ブロック層: リクエストキューが満杯 */
if (iocb->ki_flags & IOCB_NOWAIT) {
    bio->bi_opf |= REQ_NOWAIT;
    /* 利用可能なタグがなければblk-mqはBLK_STS_AGAINを返す */
}
```

### io_uringの高速パスにおけるNOWAIT {#nowait-in-io_urings-fast-path}

io_uringの性能はNOWAITに大きく依存している。各読み出しは、まず発行元のタスク上でノンブロッキングモードで発行される。この最初の発行でio_uringは`IOCB_NOWAIT`をセットするため、ファイルシステムはスリープする代わりに`-EAGAIN`を返す。

```c
/* io_uring/rw.c — io_read()/__io_read()を要約したもの */
static int __io_read(struct io_kiocb *req, unsigned int issue_flags)
{
    struct io_rw *rw = io_kiocb_to_cmd(req, struct io_rw);
    bool force_nonblock = issue_flags & IO_URING_F_NONBLOCK;

    /* 最初のノンブロッキング発行では、ファイルシステムにスリープしないよう伝える */
    if (force_nonblock)
        rw->kiocb.ki_flags |= IOCB_NOWAIT;

    ret = io_iter_do_read(rw, &iter);   /* → file->f_op->read_iter() */

    /*
     * 高速パスがブロックしそうな場合、read_iterは-EAGAINを返す。io_read()は
     * それをコアに伝播し、コアはリクエストをio-wqスレッドプールに
     * 再キューイングする。そこではIO_URING_F_NONBLOCKなしで再発行され、
     * スリープが許される。
     */
    return ret;
}
```

io-wqへのフォールバックがあるからこそ、高速パスで完了できない場合でもio_uringはあらゆるI/Oを扱える。しかし重要な指標はヒット率である。キューが満杯になることがまれなNVMeデバイス上のO_DIRECTワークロードでは、ほぼ100%の操作が高速パスで完了する。スレッドプールもコンテキストスイッチも使わない。

---

## ファイルシステムと操作ごとのNOWAITサポート {#nowait-support-by-filesystem-and-operation}

すべてのI/O種別が等しくNOWAITをサポートしているわけではない。サポートの程度は、その操作が必要とするロックやリソースによって決まる。

### O_DIRECT I/O {#o_direct-io}

```
ext4 O_DIRECT read:     Full NOWAIT support
ext4 O_DIRECT write:    Full NOWAIT support (no journal lock needed)
XFS  O_DIRECT read:     Full NOWAIT support
XFS  O_DIRECT write:    Full NOWAIT support
btrfs O_DIRECT read:    Full NOWAIT support (since ~5.16)
btrfs O_DIRECT write:   Partial (CoW may require allocation)
```

O_DIRECTの場合、主な失敗要因は次のとおりである。

- `i_rwsem`が排他的に保持されている（まれ、truncate/fallocateの間のみ）
- NVMeキューが満杯（通常の利用率ではまれ）
- エクステントマップがまだメモリ上にない（XFS: エクステントツリーの読み出しが必要になることがある）

### バッファードI/O {#buffered-io}

```
Buffered read, page in cache and uptodate:    NOWAIT succeeds
Buffered read, page not in cache:             NOWAIT returns EAGAIN
Buffered read, folio lock contended:          NOWAIT returns EAGAIN
Buffered write, page in cache and writable:   NOWAIT may succeed
Buffered write, page allocation needed:       NOWAIT returns EAGAIN
Buffered write, journal pressure (ext4):      NOWAIT returns EAGAIN
```

バッファードのNOWAIT読み出しは、ホットなワーキングセットを持つワークロードで有用である。ページがキャッシュにあれば（よくあるケース）、読み出しはシステムコールのオーバーヘッドなしに完了する。キャッシュミスの場合、NOWAITはio-wqに処理を引き渡し、io-wqがバックグラウンドでディスク読み出しを処理する。

### 操作サポートマトリクス {#operation-support-matrix}

| ファイルシステム | O_DIRECT読み出し | O_DIRECT書き込み | バッファード読み出し | バッファード書き込み |
|------------|--------------|----------------|---------------|----------------|
| ext4       | 完全         | 完全           | 部分的       | 部分的        |
| XFS        | 完全         | 完全           | 部分的       | 部分的        |
| btrfs      | 完全         | 部分的        | 部分的       | 限定的        |
| tmpfs      | N/A          | N/A            | 完全          | 完全          |
| nfs        | 完全         | 完全           | 限定的       | 限定的       |
| block dev  | 完全         | 完全           | N/A           | N/A            |

「部分的」とは、リソースが競合していなければ成功する（よくチューニングされたワークロードではこれが通常のケース）が、競合時にはio-wqにフォールバックすることを意味する。「限定的」とは、ファイルシステムの書き込み経路に構造的なブロッキングポイントがあるため、頻繁にフォールバックすることを意味する。

---

## HIPRI、IOPOLL、NOWAITの比較 {#hipri-vs-iopoll-vs-nowait-comparison}

これら3つのフラグはよく一緒に言及されるが、解決する問題は異なる。

| 機能 | HIPRI / IOPOLL | NOWAIT |
|---------|---------------|--------|
| 何をするか | IRQでスリープする代わりにNVMe CQをポーリングする | スリープする代わりにEAGAINを返す |
| 仕組み | カーネル内のビジーウェイトループ | trylock / ノンブロッキングな割り当て |
| CPUコスト | 高い — 待機中は1コアが占有される | 低い — エラー処理が変わるだけ |
| レイテンシ上の利点 | IRQ配送と起床を取り除く（~5–15µs） | スレッドのコンテキストスイッチを取り除く（~2–5µs） |
| 使える対象 | O_DIRECTのみ | バッファードI/OとO_DIRECT |
| フォールバック時の挙動 | なし — 呼び出し元は結果を得るか待つ | io_uringはio-wqスレッドにフォールバック |
| 最適な用途 | NVMeで一貫して低いp99 | 高並行性、混在ワークロード |
| 専用ハードウェアキューが必要 | はい | いいえ |
| ドライバのカーネルIOPOLLサポートが必要 | はい（nvme、io_uring、virtio-blk） | いいえ |
| io_uringとの組み合わせ | `IORING_SETUP_IOPOLL` | デフォルトの挙動、フラグ不要 |

### 適切なモードの選び方 {#choosing-the-right-mode}

```
High-performance NVMe, dedicated server, latency SLO:
  → IORING_SETUP_IOPOLL (+ SQPOLL for zero-syscall path)

High-concurrency application server, mixed I/O:
  → io_uring default (IOCB_NOWAIT + io-wq fallback)
  → No IOPOLL (too expensive to dedicate a core)

Database with large working set (mostly cached):
  → io_uring buffered NOWAIT: cache hits are free, misses fall back

Mixed read/write, NVMe, p99 matters:
  → IORING_SETUP_IOPOLL for O_DIRECT, NOWAIT for buffered
  → Separate rings: one IOPOLL ring for bulk I/O, one standard ring for metadata
```

---

## ポーリングの効果を測定する {#measuring-poll-effectiveness}

### NVMeドライバがポーリングをサポートしているか確認する {#check-that-the-nvme-driver-supports-polling}

```bash
# ブロックデバイスにポーリング用キューがあるか確認する
cat /sys/block/nvme0n1/queue/io_poll
# 0 = 無効（デフォルト）、1 = 有効

# デバイスのポーリングを有効にする
echo 1 > /sys/block/nvme0n1/queue/io_poll

# ハードウェアキューの数を確認する（通常はCPUごとに1つ）
cat /sys/block/nvme0n1/mq/*/nr_tags | head
# 各キューのタグ深さ（例: 1024）

# デバイスが複数のキューを持っているか確認する
ls /sys/block/nvme0n1/mq/
# 0  1  2  3  ...  (one directory per hw queue)
```

### fioベンチマーク: 割り込みとポーリングの比較 {#fio-benchmark-interrupt-vs-poll}

```bash
# ベースライン: 割り込みを使うO_DIRECT（デフォルト）
fio --name=baseline \
    --filename=/dev/nvme0n1 \
    --ioengine=io_uring \
    --iodepth=1 \
    --rw=randread \
    --bs=4k \
    --direct=1 \
    --runtime=30 \
    --output-format=json \
    --lat_percentiles=1

# ポーリングモード: IORING_SETUP_IOPOLL
fio --name=polled \
    --filename=/dev/nvme0n1 \
    --ioengine=io_uring \
    --iodepth=1 \
    --rw=randread \
    --bs=4k \
    --direct=1 \
    --hipri=1 \           # IORING_SETUP_IOPOLLをセットする
    --runtime=30 \
    --output-format=json \
    --lat_percentiles=1
# 2回の実行でp50、p99、p999のレイテンシを比較する
```

### bpftrace: 割り込みによる完了とポーリングによる完了を数える {#bpftrace-count-interrupt-vs-poll-completions}

```bash
# ブロック完了を、割り込みコンテキストとポーリングコンテキストのどちらから来たかで数える
bpftrace -e '
tracepoint:block:block_rq_complete
{
    /* softirqコンテキスト = 割り込み駆動、タスクコンテキスト = ポーリング */
    @completions[curtask->comm, in_softirq()] = count();
}
interval:s:5 { print(@completions); clear(@completions); }'

# NOWAITのフォールバックをトレースする: io_uringがio-wqにフォールバックする頻度
bpftrace -e '
kprobe:io_wq_enqueue
{
    @io_wq_fallbacks[comm] = count();
}
interval:s:5 { print(@io_wq_fallbacks); clear(@io_wq_fallbacks); }'
```

### perf: NVMeの割り込みレート {#perf-nvme-interrupt-rate}

```bash
# fio実行中のNVMe MSI-X割り込みを数える
# ポーリングが有効なら、この数は劇的に減るはずである
perf stat -e 'irq_vectors:irq_handler_entry' \
    -p $(pgrep fio) \
    -- sleep 10

# あるいは/proc/interruptsで数える
watch -n1 'grep nvme /proc/interrupts'
# IOPOLLが有効なら: NVMeの割り込みカウンタは増加しなくなるはずである
```

### io_uringの統計 {#io_uring-statistics}

```bash
# io_uringはリングごとの統計を/proc経由で公開している
cat /proc/$(pgrep myapp)/fdinfo/$(ls -1 /proc/$(pgrep myapp)/fd | head -1)
# 注目する項目: sq_entries, cq_entries, sq_head, sq_tail, cq_head, cq_tail

# io_uring内部のカーネルtracepoint（Linux 5.x以降）
bpftrace -e '
tracepoint:io_uring:io_uring_poll_arm   { @poll_armed = count(); }
tracepoint:io_uring:io_uring_complete   { @completed  = count(); }
interval:s:1 { print(@poll_armed); print(@completed);
               clear(@poll_armed); clear(@completed); }'
```

### NOWAITのヒット率を検証する {#verifying-nowait-hit-rate}

```bash
# NOWAITの高速パスとio-wqへのフォールバックを計測する
bpftrace -e '
/* 高速パスでの完了 */
kretprobe:vfs_iocb_iter_read /retval > 0/ {
    @nowait_hits = count();
}
/* io-wqへのフォールバック（NOWAITミス） */
kprobe:io_wq_enqueue {
    @io_wq_submissions = count();
}
interval:s:5 {
    printf("NOWAIT hits: %d, io-wq fallbacks: %d\n",
           @nowait_hits, @io_wq_submissions);
    clear(@nowait_hits); clear(@io_wq_submissions);
}'
# 目標: O_DIRECTのNVMeワークロードでNOWAITヒット率95%超
```

---

## 主要なソースファイル {#key-source-files}

| ファイル | 内容 |
|------|----------|
| `include/linux/fs.h` | `IOCB_HIPRI`、`IOCB_NOWAIT`、`struct kiocb`、`file_operations.iopoll` |
| `include/uapi/linux/fs.h` | `RWF_HIPRI`、`RWF_NOWAIT`（ユーザー空間から見えるフラグ） |
| `include/uapi/linux/io_uring.h` | `IORING_SETUP_IOPOLL`、`IORING_SETUP_SQPOLL` |
| `fs/read_write.c` | `kiocb_set_rw_flags()`、`do_iter_readv_writev()` |
| `fs/iomap/direct-io.c` | `iomap_dio_rw()`、`iocb_bio_iopoll()`、`REQ_HIPRI`付きのbio発行 |
| `block/bio.c` | `bio_poll()` |
| `block/blk-mq.c` | HIPRIリクエストの割り当て、ポーリング用キューの管理 |
| `drivers/nvme/host/pci.c` | `nvme_poll()`、`nvme_cqe_pending()`、フェーズビットの確認 |
| `io_uring/io_uring.c` | `io_do_iopoll()`、`io_iopoll_check()` |
| `io_uring/rw.c` | `io_read()`、`io_write()`、NOWAITの高速パスとio-wqへのフォールバック |
| `io_uring/sqpoll.c` | `io_sq_thread()`、SQPOLLカーネルスレッド |

---

## 参考文献 {#further-reading}

- [Direct I/O](direct-io.md) — HIPRI/IOPOLLにおけるO_DIRECTの必要性
- [非同期I/Oの進化](async-io.md) — io_uringのアーキテクチャとSQE/CQEモデル
- [ストレージのチューニング](tuning-storage.md) — fioとio_uringの実践的な性能チューニング
- `Documentation/block/blk-mq.rst` — ブロックマルチキューのアーキテクチャ
- `Documentation/admin-guide/iostats.rst` — `/proc/diskstats`の読み方
- NVMe仕様 §4.6 — Completion Queueエントリの形式とフェーズビットのセマンティクス
