# NVMeドライバ {#nvme-driver}

> PCIe SSDのアーキテクチャ、サブミッション/コンプリーションキュー、blk-mqとの統合

## NVMeの概要 {#nvme-overview}

NVMe（Non-Volatile Memory Express）は、PCIe接続SSDのためのプロトコルである。NVMe 1.0仕様は2011年3月に承認された。Linuxカーネルのドライバは、Matthew Wilcox（Intel）によって書かれ、Linux 3.3（2012年3月）で初めて登場した — [`b60503ba432b`](https://git.kernel.org/linus/b60503ba432b16fc84442a84e29a7aad2c0c363d)。低レイテンシと高い並列性のために設計されている。

```
CPU → PCIe bus → NVMe controller → NAND flash

Key differences from SATA/SAS:
  SATA: 1 command queue, 32 depth
  NVMe: up to 65535 I/O queues, 65535 depth each
  NVMe latency: ~100µs (vs ~100ms HDD, ~1ms SATA SSD)
```

## NVMeのキューアーキテクチャ {#nvme-queue-architecture}

### 管理キューとI/Oキュー {#admin-queue-and-io-queues}

```
NVMe controller
├── Admin queue (Q0): management commands (identify, firmware, etc.)
└── I/O queues (Q1-QN): data read/write commands
    ├── Q1: CPU 0  (submission queue + completion queue)
    ├── Q2: CPU 1
    ├── Q3: CPU 2
    └── ...  (one queue per CPU, up to controller's maximum)
```

各キューは、CPUとNVMeコントローラの間の**共有メモリリング**である。

```
Submission Queue (SQ):
  CPU writes commands here, advances SQ tail doorbell
  Controller reads commands, advances SQ head

Completion Queue (CQ):
  Controller writes completions here, advances CQ tail
  CPU reads completions, advances CQ head doorbell
```

### NVMeサブミッションキューエントリ（SQE） {#nvme-submission-queue-entry-sqe}

```c
/* include/linux/nvme.h */
struct nvme_command {
    __u8    opcode;           /* NVM_OP_READ, NVM_OP_WRITE など */
    __u8    flags;
    __u16   command_id;       /* 完了時にCQEのcommand_idと照合される */
    __le32  nsid;             /* namespace ID（1始まり） */
    __u64   rsvd2;
    __le64  metadata;         /* メタデータポインタ */
    union nvme_data_ptr dptr; /* データポインタ: PRPリストまたはSGL */
    union {
        struct nvme_rw_command rw; /* 読み込み/書き込み用 */
        struct nvme_identify   identify;
        /* ... その他のコマンド種別 ... */
    };
};

/* 読み込み/書き込みコマンドのフィールド: */
struct nvme_rw_command {
    __le16  flags;
    __le16  control;    /* FUA、PRINFOなど */
    __le32  dsmgmt;
    __le32  reftag;
    __le16  apptag;
    __le16  appmask;
    __le64  slba;       /* 開始論理ブロックアドレス */
    __le16  length;     /* 0始まりの論理ブロック数 - 1 */
};
```

### NVMeコンプリーションキューエントリ（CQE） {#nvme-completion-queue-entry-cqe}

```c
struct nvme_completion {
    __le32  result;         /* コマンド固有の結果 */
    __u32   rsvd;
    __le16  sq_head;        /* SQのheadポインタ（コントローラが更新） */
    __le16  sq_id;          /* サブミッションキューID */
    __u16   command_id;     /* SQEのcommand_idと一致 */
    __le16  status;         /* 成功またはエラーコード */
    /* ビット0: フェーズタグ（CQが一周するたびに0/1が交互に変わる） */
};
```

**フェーズタグ**により、別個の「使用済みエントリ数」カウンタなしに完了をポーリングできる。CPUはCQが一周するたびに、期待するフェーズビットを反転させる。

## LinuxのNVMeドライバ: struct nvme_dev {#linux-nvme-driver-struct-nvme_dev}

```c
/* drivers/nvme/host/pci.c */
struct nvme_dev {
    struct nvme_queue   *queues;     /* I/Oキューの配列 */
    u32                 q_depth;     /* キュー深さ */
    u32                 db_stride;   /* ドアベルレジスタのストライド */
    void __iomem       *bar;         /* BAR0 MMIOのベース */
    unsigned long       bar_mapped_size;

    /* 管理キュー */
    struct nvme_queue   adminq;

    /* 割り込み */
    struct msix_entry  *entry;       /* MSI-Xエントリ */
    int                 num_vecs;    /* MSI-Xベクタの数 */

    struct nvme_ctrl    ctrl;        /* 汎用NVMeコントローラ */
    /* ... */
};

struct nvme_queue {
    struct nvme_dev    *dev;
    struct nvme_command *sq_cmds;   /* サブミッションキュー（DMA） */
    volatile struct nvme_completion *cqes; /* コンプリーションキュー（DMA） */
    dma_addr_t          sq_dma_addr;
    dma_addr_t          cq_dma_addr;
    u32 __iomem        *q_db;       /* ドアベルレジスタ */
    u16                 q_depth;
    u16                 cq_head;
    u16                 cq_phase;   /* フェーズビット */
    u16                 sq_tail;
    u16                 last_sq_tail;
    int                 cq_vec;     /* MSI-Xベクタ */
    struct blk_mq_tags *tags;
};
```

## blk-mqとの統合 {#blk-mq-integration}

NVMeドライバはブロック層の`blk-mq`（マルチキュー）フレームワークにフックする。

```c
/* drivers/nvme/host/pci.c */
static const struct blk_mq_ops nvme_mq_admin_ops = {
    .queue_rq = nvme_queue_rq,
    .complete = nvme_pci_complete_rq,
    .init_hctx = nvme_admin_init_hctx,
    .exit_hctx = nvme_admin_exit_hctx,
    .init_request = nvme_init_request,
    .timeout = nvme_timeout,
};

static const struct blk_mq_ops nvme_mq_ops = {
    .queue_rq     = nvme_queue_rq,     /* requestをNVMeのSQに発行する */
    .complete     = nvme_pci_complete_rq, /* 完了時に呼ばれる */
    .init_hctx    = nvme_init_hctx,    /* ハードウェアキューごとの初期化 */
    .init_request = nvme_init_request,
    .map_queues   = nvme_pci_map_queues, /* CPU → キューのマッピング */
    .poll         = nvme_poll,          /* ポーリングによる完了（IOPOLL） */
    .timeout      = nvme_timeout,
};
```

### nvme_queue_rq: requestの発行 {#nvme_queue_rq-submitting-a-request}

```c
static blk_status_t nvme_queue_rq(struct blk_mq_hw_ctx *hctx,
                                    const struct blk_mq_queue_data *bd)
{
    struct nvme_ns *ns = hctx->queue->queuedata;
    struct nvme_queue *nvmeq = hctx->driver_data;
    struct request *req = bd->rq;
    struct nvme_iod *iod = blk_mq_rq_to_pdu(req);
    struct nvme_command cmnd;
    blk_status_t ret;

    /* requestのバッファをDMAにマッピングする */
    ret = nvme_setup_cmd(ns, req, &cmnd);
    if (ret)
        return ret;

    ret = nvme_map_data(dev, req, &cmnd);
    if (ret)
        goto out_free_cmd;

    /* コマンドをSQリングに書き込む */
    nvme_submit_cmd(nvmeq, &cmnd, bd->last);
    return BLK_STS_OK;
}

static void nvme_submit_cmd(struct nvme_queue *nvmeq,
                              struct nvme_command *cmd, bool write_sq)
{
    u16 tail = nvmeq->sq_tail;

    /* コマンドをSQのスロットにコピーする */
    memcpy(nvmeq->sq_cmds + tail, cmd, sizeof(*cmd));

    /* tailを進める */
    if (++tail == nvmeq->q_depth)
        tail = 0;
    nvmeq->sq_tail = tail;

    if (write_sq) {
        /* ドアベルを鳴らす: 新しいSQエントリがあることをコントローラに伝える */
        writel(tail, nvmeq->q_db);
    }
}
```

### nvme_irq: 完了の処理 {#nvme_irq-processing-completions}

```c
static irqreturn_t nvme_irq(int irq, void *data)
{
    struct nvme_queue *nvmeq = data;
    return nvme_process_cq(nvmeq);
}

static blk_status_t nvme_process_cq(struct nvme_queue *nvmeq)
{
    u16 head = nvmeq->cq_head;
    u8  phase = nvmeq->cq_phase;

    /* フェーズビットが一致するCQエントリをポーリングする */
    while ((nvmeq->cqes[head].status & 1) == phase) {
        struct nvme_completion *cqe = &nvmeq->cqes[head];
        u16 status = le16_to_cpu(cqe->status) >> 1;

        /* requestを完了させる */
        struct request *req = nvme_find_rq(nvmeq, cqe->command_id);
        nvme_complete_rq(req, status);

        /* headを進める */
        if (++head == nvmeq->q_depth) {
            head = 0;
            phase ^= 1;   /* 一周するとフェーズビットが反転する */
        }
    }

    nvmeq->cq_head  = head;
    nvmeq->cq_phase = phase;

    /* CQのheadドアベルを鳴らす */
    writel(head, nvmeq->q_db + nvmeq->dev->db_stride);
    return IRQ_HANDLED;
}
```

## NVMeのnamespace {#nvme-namespaces}

NVMeは「namespace」を使ってSSDを論理ユニット（SCSIのLUNのようなもの）に分割する。

```bash
# NVMeのnamespaceを一覧表示する
nvme list-ns /dev/nvme0

# 各namespaceはブロックデバイスとして現れる:
ls /dev/nvme0n1    # nvme0 = コントローラ0、n1 = namespace 1
ls /dev/nvme0n2    # namespace 2

# コントローラを識別する:
nvme id-ctrl /dev/nvme0 | head -20
# mn: Samsung SSD 990 PRO 2TB
# sn: S1234567890
# fr: 4B2QFXO7
# tnvmcap: 2000398934016   # 2TB

# namespace 1を識別する:
nvme id-ns /dev/nvme0n1
# nsze: 3907029168  # 総セクタ数
# ncap: 3907029168  # フォーマット済み容量
# lbads: 9           # 論理ブロックサイズ = 2^9 = 512B（12なら4096B）
```

## NVMeのポーリング（IRQなしのパス） {#nvme-polling-no-irq-path}

超低レイテンシのために、NVMeは割り込みの代わりに**ポーリング**を使える。

```bash
# NVMeのioポーリングを有効にする
echo 1 > /sys/block/nvme0n1/queue/io_poll
echo 0 > /sys/block/nvme0n1/queue/io_poll_delay  # -1 = スピン、0 = ハイブリッド

# IORING_SETUP_IOPOLL付きのio_uring:
struct io_uring_params params = {
    .flags = IORING_SETUP_IOPOLL,
};
io_uring_queue_init_params(128, &ring, &params);
```

ポーリングでは完了割り込みが無効になり、カーネルは発行後にCQをビジーポーリングする。NVMe SSDでは、レイテンシが約100µsから約15〜20µsに下がることがある。

## NVMeの観察 {#observing-nvme}

```bash
# NVMeの統計
nvme smart-log /dev/nvme0
# Critical Warning: 0x00
# Temperature: 35 C
# Available Spare: 100%
# Data Units Read: 12,345,678
# Data Units Written: 9,876,543
# Host Read Commands: 234,567,890
# Host Write Commands: 123,456,789

# I/Oキューの統計
cat /sys/block/nvme0n1/queue/iostats

# キュー深さとディスパッチの統計（blk-mq）
cat /sys/block/nvme0n1/mq/0/nr_tags

# レイテンシのヒストグラム
bpftrace -e '
tracepoint:block:block_rq_issue
/strncmp(args->devname, "nvme", 4) == 0/
{
    @start[args->sector] = nsecs;
}
tracepoint:block:block_rq_complete
/@start[args->sector]/
{
    @lat_us = hist((nsecs - @start[args->sector]) / 1000);
    delete(@start[args->sector]);
}'
```

## 参考文献 {#further-reading}

- [blk-mq（マルチキュー）](blk-mq.md) — NVMeが使うブロックマルチキューフレームワーク
- [bioとrequestの構造体](bio-request.md) — NVMeが処理するstruct request
- [I/Oスケジューラ](io-schedulers.md) — NVMeはスケジューラをバイパスする（noneスケジューラ）
- [PCIドライバ](../drivers/pci-driver.md) — NVMeはPCIドライバである
- [ダイレクトI/O](../io/direct-io.md) — データベース向けのO_DIRECT + NVMe
- `drivers/nvme/host/pci.c` — NVMe PCIドライバ
- `drivers/nvme/host/core.c` — NVMeコア（namespace、管理コマンド）
