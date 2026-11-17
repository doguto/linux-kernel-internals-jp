# ブロック層の概要 {#block-layer-overview}

> 主要な構造体とI/O発行パス

## 主要なデータ構造 {#key-data-structures}

ブロック層は3つの主要な構造体を使う。

```
gendisk                    ← ディスク全体を表す (/dev/sda)
  │
  ├── block_device         ← パーティションまたはディスク全体を表す (/dev/sda1)
  │
  └── request_queue        ← I/Oキュー + スケジューラの状態
        │
        └── blk_mq_tag_set ← ハードウェアキューのマッピング (blk-mq)
```

### struct gendisk {#struct-gendisk}

`gendisk`は物理ディスクまたは仮想ブロックデバイスを表す。

```c
/* include/linux/blkdev.h */
struct gendisk {
    int                    major;         /* デバイスのメジャー番号 */
    int                    first_minor;   /* 最初のマイナー番号 */
    int                    minors;        /* マイナー番号の数（パーティション数） */
    char                   disk_name[DISK_NAME_LEN]; /* "sda", "nvme0n1" */
    struct xarray          part_tbl;      /* パーティションテーブル */
    struct block_device   *part0;         /* ディスク全体のblock_device */
    const struct block_device_operations *fops;  /* ドライバの操作 */
    struct request_queue  *queue;         /* I/Oキュー */
    void                  *private_data;  /* ドライバのプライベートデータ */
    int                   flags;          /* GENHD_FL_* */
};
```

### struct block_device {#struct-block_device}

パーティション（またはディスク全体）ごとに1つの`block_device`がある。ファイルシステムが使うのはこれである。

```c
struct block_device {
    sector_t           bd_start_sect;  /* パーティションの開始セクタ */
    sector_t           bd_nr_sectors;  /* 512バイトセクタ単位のサイズ */
    struct gendisk    *bd_disk;        /* 物理ディスク */
    struct inode      *bd_inode;       /* bdevファイルシステム内のinode */
    struct super_block *bd_super;      /* マウントされていればsuperblock */
    unsigned int       bd_openers;     /* オープン数 */
    struct mutex       bd_mutex;       /* open/closeのmutex */
};
```

### struct request_queue {#struct-request_queue}

`request_queue`はI/O管理の中心となるオブジェクトである。

```c
struct request_queue {
    const struct blk_mq_ops  *mq_ops;   /* ドライバの操作 */
    struct blk_mq_tag_set    *tag_set;   /* ハードウェアキューのマッピング */
    struct elevator_queue    *elevator;  /* I/Oスケジューラ */
    struct queue_limits       limits;    /* セクタサイズ、最大リクエストサイズなど */
    unsigned long             queue_flags; /* QUEUE_FLAG_* */
    /* ... */
};
```

## submit_bio()のパス {#the-submit_bio-path}

```c
/* ファイルシステムまたはダイレクトI/O: */
submit_bio(bio)
    ↓
submit_bio_noacct(bio)
    ↓
blk_mq_submit_bio(bio)    ← blk-mqパス（すべての現代的なドライバ）
    │
    ├── plugリスト内の既存requestとのマージを試みる
    │
    ├── tag setからrequestを割り当てる
    │
    ├── I/Oスケジューラがあれば: elevator->type->ops.insert_requests()
    │     mq-deadline: デッドライン順にソートされたdeadlineキューに追加
    │     BFQ: BFQのプロセスごとのキューに追加
    │     none: ハードウェアキューに直接追加
    │
    └── blk_mq_run_hw_queue()
          → ドライバの.queue_rq()が呼ばれる
          → DMAマッピング
          → ハードウェアへのコマンド発行
```

## ブロックデバイスの/procと/sys {#proc-and-sys-for-block-devices}

```bash
# ブロックデバイスを一覧表示
lsblk
# NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
# sda      8:0    0  100G  0 disk
# ├─sda1   8:1    0   99G  0 part /
# └─sda2   8:2    0    1G  0 part [SWAP]
# nvme0n1 259:0   0  500G  0 disk
# └─nvme0n1p1 259:1 0 500G 0 part /data

# ブロックデバイスの統計
cat /proc/diskstats
# 8  0 sda 123456 7890 12345678 12345 234567 8901 23456789 23456 0 12345 23456
# Fields: major minor name reads_completed reads_merged sectors_read ms_reading
#         writes_completed writes_merged sectors_written ms_writing io_in_progress
#         ms_io ms_weighted_io

# キューごとのsysfs
ls /sys/block/sda/queue/
cat /sys/block/sda/queue/scheduler          # 現在のスケジューラ
cat /sys/block/sda/queue/nr_requests        # キュー深さ
cat /sys/block/sda/queue/read_ahead_kb      # 先読みサイズ
cat /sys/block/sda/queue/rotational         # 1=HDD, 0=SSD/NVMe
cat /sys/block/sda/queue/logical_block_size # 通常は512または4096
```

## I/Oのplugとunplug {#io-plug-and-unplug}

スループットのために、ブロック層は発行前に短時間キューを「plug（栓をする）」してリクエストを溜める。

```c
/* 多数のbioを発行するドライバ/ファイルシステムのコード */
struct blk_plug plug;
blk_start_plug(&plug);

submit_bio(bio1);
submit_bio(bio2);
submit_bio(bio3);
/* requestはplug->mq_listに溜まる */

blk_finish_plug(&plug);
/* すべてのrequestが一度にディスパッチされる → マージが効きやすく、割り込みが減る */
```

これはTCPのNagleアルゴリズムに似ている。小さな操作をまとめて大きな操作にするのである。

## I/OスケジューラとNVMeの問題 {#io-schedulers-and-the-nvme-problem}

`request_queue`内の`elevator_queue`はオプションである。I/Oスケジューラはbioの発行とハードウェアへのディスパッチの間に位置し、リクエストの並べ替えとマージを行う。HDDではこれが不可欠だった — エレベータスケジューラはランダムな書き込みをシーケンシャルI/Oに変え、シーク時間を減らすことでスループットを倍増させることができた。

NVMeは計算を変える。

- シークレイテンシがない: リクエストの順序は性能に無関係である。
- ハードウェアがNANDダイ間で独自の内部スケジューリングを行う。
- ソフトウェアスケジューラを追加すると、利点ゼロでレイテンシが増える（リクエストがスケジューラのキューで待つ）。

そのため、NVMeデバイスはデフォルトで`none`スケジューラを使う。スケジューラの基盤はSATA SSDやHDDのために`request_queue`にまだ存在しているが、NVMeのパスでは完全にバイパスされる。

これがどのようにblk-mqの再設計につながったかの全容は[blk-mq](blk-mq.md)にある。

## 参考文献 {#further-reading}

- [bioとrequestの構造体](bio-request.md) — I/O記述子オブジェクト
- [blk-mq](blk-mq.md) — 現代的なマルチキューのディスパッチ機構
- [I/Oスケジューラ](io-schedulers.md) — リクエストがどのように並べ替えられるか
