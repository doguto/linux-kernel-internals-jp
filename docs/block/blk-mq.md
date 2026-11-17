# blk-mq: マルチキューブロック層 {#blk-mq-multi-queue-block-layer}

> 高性能ストレージのための現代的なブロック層インターフェース

## なぜblk-mqか {#why-blk-mq}

従来のブロック層は、デバイスごとに単一のリクエストキューを持ち、それをグローバルなロックで保護していた。これはHDD（どのみちシーケンシャルである）には問題なかったが、複数のハードウェアキューにまたがって数百万IOPSを処理できるNVMe SSDにとってはボトルネックになった。

blk-mq（Linux 3.13で導入）は次のものを使う。

- ロック競合をなくすための**複数のソフトウェアキュー**（CPUごと、またはCPUグループごとに1つ）
- **複数のハードウェアキュー**（NVMeのキュー、SCSIホストアダプタなどにマッピングされる）
- requestオブジェクトのための**CPUごとのタグ割り当て**

## 進化: シングルキューからblk-mqへ {#the-evolution-from-single-queue-to-blk-mq}

### 元のブロック層（Linux 2.4〜2.6） {#the-original-block-layer-linux-2426}

元のブロック層は、デバイスごとに1つの`request_queue`を持ち、それを単一のspinlock（`q->queue_lock`）で保護していた。そのキューの中にはエレベータスケジューラ（CFQ、deadline、anticipatory）が置かれ、HDDのシーク時間を最小化するためにリクエストをマージし並べ替えていた。

これはHDDにとっては理にかなっていた。回転ディスクにはシークレイテンシ（3〜10ms）があるため、シリンダアドレスによってリクエストをバッチ化し並べ替える（「エレベータ」の比喩）ことでスループットを倍増させられた。CFQ（Completely Fair Queuing）がデフォルトであり、プロセスごとのキューを交互に処理することで、各プロセスにディスク帯域の公平な取り分を与えていた。

```
Per-device request_queue
  │
  ├── q->queue_lock (single spinlock — ALL CPUs contend here)
  ├── elevator (CFQ/deadline/anticipatory)
  └── hw_queue → one hardware submission path
```

### SSDが欠陥を露呈させた {#ssds-exposed-the-flaw}

SSDが登場すると（2007〜2010年頃）、このモデル全体が崩壊した。

1. **シークコストがない** — アドレスによるリクエストの並べ替えは無意味になった。エレベータは何の利点もなく、CPUオーバーヘッドとレイテンシを増やすだけだった。
2. **高い並列性** — SSDは複数の内部フラッシュダイから数千のI/Oを同時に処理できたが、カーネルはそれらすべてを1つのキューで直列化していた。
3. **割り込みのボトルネック** — 完了割り込みは1つのCPUに届き、そのCPUが他のCPU上の要求元を起こさなければならなかった。

NVMe（2011年以降）はこの危機を深刻なものにした。NVMeコントローラは最大65,535個のハードウェアキューを持ち、それぞれに65,535個のスロットがある。しかし`request_queue`は単一のロックを持つ単一のオブジェクトだったため、カーネルはそのうち1つしか使えなかった。

初期のNVMeドライブでのベンチマークでは、カーネルは約700K IOPSで`queue_lock` spinlockを飽和させていた — ハードウェアの約1M+ IOPSの能力を大きく下回る。

### blk-mq（Linux 3.13、2014年） {#blk-mq-linux-313-2014}

Jens Axboeは2つの重要なアイデアを中心にブロック層を再設計した — [`320ae51feed5`](https://git.kernel.org/linus/320ae51feed5c2f13664aa05a76bec198967e04d)（"blk-mq: new multi-queue block IO queueing mechanism"）、[LWN](https://lwn.net/Articles/552683/)。

1. **CPUごとのソフトウェアキュー** — 各CPUは独自のロックを持つ独自の`blk_mq_ctx`を持つ。I/O発行中にCPU同士が競合することはない。requestはCPUごとのキューでバッチ化され、その後ハードウェアキューにディスパッチされる。

2. **複数のハードウェアキュー** — `blk_mq_hw_ctx`は実際のハードウェアのサブミッションキューにマッピングされる。NVMeでは、各ハードウェアキューはNVMeのSQ/CQペアにマッピングされる。キューが1つのSCSIアダプタでは、blk-mqは1つのハードウェアキューを提示する。

エレベータスケジューラは、CPUごとのキューとハードウェアキューの*間に*収まるオプションのプラグインとして再設計された。NVMeはデフォルトで`none`（スケジューラなし）を使い、requestはそのまま通り抜ける。HDDは今でも`mq-deadline`や`bfq`の恩恵を受ける。

```
Before blk-mq:    1 CPU → q->queue_lock → 1 hw queue
After blk-mq:     N CPUs → N per-CPU queues → M hw queues (N ≫ M possible)
```

その結果、現代のNVMeドライブはLinux上で3〜7M IOPSで飽和するようになり、その制約はカーネルのロック競合ではなくPCIeの帯域となった。

## 2段階のキューアーキテクチャ {#two-level-queue-architecture}

```
CPUs
 CPU 0    CPU 1    CPU 2    CPU 3
   │        │        │        │
   ▼        ▼        ▼        ▼
Software Queue (blk_mq_ctx)  ← per-CPU, no contention between CPUs
   │        │        │        │
   └────────┴────────┘        │
           │                  │
    ┌──────▼──────┐   ┌───────▼──────┐
    │  Hardware   │   │  Hardware    │
    │  Queue 0    │   │  Queue 1     │  ← maps to NVMe submission queues
    └─────────────┘   └──────────────┘
```

### ソフトウェアキュー（blk_mq_ctx） {#software-queues-blkmqctx}

CPUごとに1つ。requestは最初、ロックなしでここにキューイングされる（各CPUが独自のキューを持つ）。

```c
/* CPUごとのソフトウェアキューのコンテキスト */
struct blk_mq_ctx {
    struct {
        spinlock_t      lock;
        struct list_head rq_lists[HCTX_MAX_TYPES]; /* 保留中のrequest */
    } ____cacheline_aligned_in_smp;

    unsigned int        cpu;        /* これが属するCPU */
    unsigned short      index_hw[HCTX_MAX_TYPES]; /* ハードウェアキューのインデックス */
    struct blk_mq_hw_ctx *hctxs[HCTX_MAX_TYPES];
    struct request_queue *queue;
};
```

### ハードウェアキュー（blk_mq_hw_ctx） {#hardware-queues-blkmqhw_ctx}

ハードウェアキュー（NVMeのnamespace、virtioのキューなど）ごとに1つ。

```c
/* ハードウェアキューのコンテキスト */
struct blk_mq_hw_ctx {
    spinlock_t          lock;         /* dispatchリストを保護する */
    struct list_head    dispatch;     /* ディスパッチ可能なrequest */
    unsigned long       state;        /* BLK_MQ_S_* */
    struct blk_mq_tags *tags;         /* このハードウェアキューのタグ割り当て */
    cpumask_var_t       cpumask;      /* このハードウェアキューにマッピングされたCPU */
    unsigned long       flags;        /* BLK_MQ_F_* */
};
```

## blk-mqドライバの実装 {#implementing-a-blk-mq-driver}

### 1. 操作を定義する {#1-define-the-operations}

```c
static const struct blk_mq_ops my_mq_ops = {
    .queue_rq     = my_queue_rq,     /* requestをハードウェアに発行する */
    .complete     = my_complete_rq,  /* 割り込みからの完了時に呼ばれる */
    .timeout      = my_timeout,      /* requestがタイムアウトしたときに呼ばれる */
    .poll         = my_poll,         /* ポーリングI/O用（割り込みなし） */
};
```

### 2. tag_setを設定する {#2-set-up-tag_set}

```c
struct blk_mq_tag_set tag_set;

memset(&tag_set, 0, sizeof(tag_set));
tag_set.ops        = &my_mq_ops;
tag_set.nr_hw_queues = num_online_cpus();  /* またはNVMeキューの数 */
tag_set.queue_depth  = 128;               /* 処理中requestの最大数 */
tag_set.numa_node    = NUMA_NO_NODE;
tag_set.cmd_size     = sizeof(struct my_request);  /* ドライバ固有のrequestごとのデータ */
tag_set.flags        = BLK_MQ_F_SHOULD_MERGE;

ret = blk_mq_alloc_tag_set(&tag_set);
```

### 3. ディスクを割り当てる {#3-allocate-disk}

```c
struct gendisk *disk = blk_mq_alloc_disk(&tag_set, NULL, my_dev);
disk->major       = 0;  /* カーネルに割り当てさせる */
disk->first_minor = 0;
disk->minors      = 1;
disk->fops        = &my_block_ops;
snprintf(disk->disk_name, DISK_NAME_LEN, "myblk%d", index);

/* 容量を設定する（512バイトセクタ単位） */
set_capacity(disk, total_bytes >> 9);

/* 登録する */
add_disk(disk);
```

### 4. queue_rq: ハードウェアへの発行 {#4-queue_rq-submitting-to-hardware}

```c
static blk_status_t my_queue_rq(struct blk_mq_hw_ctx *hctx,
                                  const struct blk_mq_queue_data *bd)
{
    struct request *rq = bd->rq;
    struct my_request *my_req = blk_mq_rq_to_pdu(rq);

    /* requestを開始済みとしてマークする */
    blk_mq_start_request(rq);

    /* ハードウェアコマンドを組み立てる */
    my_req->sector = blk_rq_pos(rq);     /* 開始セクタ */
    my_req->count  = blk_rq_sectors(rq); /* セクタ数 */
    my_req->dir    = rq_data_dir(rq);    /* READまたはWRITE */

    /* DMA用にscatter-gatherリストをマッピングする */
    my_req->nsegs = blk_rq_map_sg(hctx->queue, rq, my_req->sglist);

    /* ハードウェアに発行する */
    if (my_hw_submit(hctx->driver_data, my_req) != 0) {
        blk_mq_requeue_request(rq, true);
        return BLK_STS_RESOURCE;
    }

    return BLK_STS_OK;
}
```

### 5. 割り込みからの完了 {#5-completion-from-interrupt}

```c
/* ハードウェアが完了を通知したときにIRQハンドラから呼ばれる */
static void my_irq_handler(void *data)
{
    struct my_request *my_req = data;
    struct request *rq = blk_mq_rq_from_pdu(my_req);

    /* ステータスを直接渡す — struct requestには'result'フィールドがない */
    blk_mq_end_request(rq, BLK_STS_OK);  /* エラー時はBLK_STS_IOERR */
}

/* あるいは、完了を2つのステップに分ける: */
static void my_complete_rq(struct request *rq)
{
    blk_mq_end_request(rq, BLK_STS_OK);
    /* これによりbioと、そのファイルシステム/アプリケーションに通知される */
}
```

## タグの割り当て {#tag-allocation}

処理中の各requestには**タグ**が与えられる — 未完了の間、それを一意に識別する小さな整数である。ドライバはタグを使って、完了イベントとrequestを対応づける。

```c
/* タグの範囲: 0からqueue_depth-1 */
int tag = rq->tag;    /* queue_rqを呼ぶ前にblk-mqが設定する */

/* 完了時にタグからrequestを取り出す */
struct request *rq = blk_mq_tag_to_rq(hctx->tags, tag);
```

## ポーリングI/O {#polled-io}

超低レイテンシのNVMeでは、完了をスピンして待つほうが割り込みを待つより速い。

```c
/* ポーリングフラグ付きで発行する */
bio->bi_opf |= REQ_POLLED;
submit_bio(bio);

/* 完了をポーリングする */
blk_poll(q, cookie, true);
```

これはCPUを1つ消費する代わりに、割り込みのオーバーヘッド（約1〜2µs）を回避する。レイテンシ目標が10µs未満のときに有用である。

## blk-mqの観察 {#observing-blk-mq}

```bash
# キュー深さと処理中のrequest
cat /sys/block/nvme0n1/queue/nr_requests      # 最大キュー深さ
cat /sys/block/nvme0n1/inflight               # 現在処理中のrequest

# ハードウェアキューごとの統計（ドライバが公開していれば）
ls /sys/block/nvme0n1/mq/
cat /sys/block/nvme0n1/mq/0/nr_tags

# blktrace: 個々のI/Oイベントをトレースする
blktrace -d /dev/nvme0n1 -o - | blkparse -i -
# C = 完了、D = ドライバ、I = 挿入、Q = キューに入った

# iostat: スループットとレイテンシ
iostat -x 1 /dev/nvme0n1
```

## 参考文献 {#further-reading}

- [bioとrequestの構造体](bio-request.md) — blk-mqがディスパッチするもの
- [I/Oスケジューラ](io-schedulers.md) — blk-mqのディスパッチの前に動くもの
- [ブロック層の概要](block-overview.md) — 全体像
