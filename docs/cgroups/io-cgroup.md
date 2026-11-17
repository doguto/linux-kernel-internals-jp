# io cgroupコントローラ {#io-cgroup-controller}

> ブロックI/O帯域のスロットリング、重みベースのスケジューリング、レイテンシSLA

## 概要 {#overview}

cgroup v2の `io` コントローラは、プロセスのブロックI/Oリソースを管理する。次の機能を提供する:

- **io.max**: ハードなレート制限（バイト/秒、IOPS）
- **io.weight**: 比例配分スケジューリング（WFQ）
- **io.latency**: レイテンシSLAの強制（レイテンシを守るために過剰なI/Oを排除する） [(commit)](https://git.kernel.org/linus/d70675121546c35feaceebf7ed9caed8716640f3) [(LWN)](https://lwn.net/Articles/758963/)

```bash
# cgroup で io コントローラを有効にする:
echo "+io" > /sys/fs/cgroup/mycgroup/cgroup.subtree_control

# サポートされている機能を確認する:
cat /sys/block/sda/queue/scheduler
# [mq-deadline] kyber bfq none
# BFQ は io.weight による比例配分スケジューリングをサポートする
# mq-deadline は io.latency をサポートする
```

## io.max: ハードなレート制限 {#iomax-hard-rate-limits}

```bash
# 書式: MAJ:MIN rbps=N wbps=N riops=N wiops=N
echo "8:0 rbps=52428800" > /sys/fs/cgroup/mycgroup/io.max  # 読み込み 50MB/s
echo "8:0 wbps=26214400" > /sys/fs/cgroup/mycgroup/io.max  # 書き込み 25MB/s
echo "8:0 riops=1000"    > /sys/fs/cgroup/mycgroup/io.max  # 読み込み 1000 IOPS
echo "8:0 wiops=500"     > /sys/fs/cgroup/mycgroup/io.max  # 書き込み 500 IOPS

# 複数の制限を一度に設定:
echo "8:0 rbps=52428800 wbps=26214400 riops=1000 wiops=500" \
    > /sys/fs/cgroup/mycgroup/io.max

# 制限を解除:
echo "8:0 rbps=max wbps=max riops=max wiops=max" \
    > /sys/fs/cgroup/mycgroup/io.max

# 現在の制限を読む:
cat /sys/fs/cgroup/mycgroup/io.max
# 8:0 rbps=52428800 wbps=26214400 riops=1000 wiops=500
```

### カーネルでの実装 {#kernel-implementation}

```c
/* block/blk-throttle.c */
/* bio が発行されると、blk-throttle は cgroup が制限を超えているかを確認する: */
static bool throtl_tg_is_idle(struct throtl_grp *tg)
{
    /* バイト数と IOPS の両方の制限を確認 */
    return (tg->bytes_disp[READ] + tg->bytes_disp[WRITE]) == 0;
}

static void tg_dispatch_one_bio(struct throtl_grp *tg, bool rw)
{
    struct throtl_data *td = tg->td;
    struct bio *bio;

    /* 制限内かどうかを確認 */
    if (tg_bps_limit(tg, rw)) {
        /* レート制限: bio のディスパッチを遅らせる */
        tg->disptime = jiffies + tg->slice_remaining[rw];
        throtl_schedule_pending_timer(tg->service_queue, tg->disptime);
        return;
    }
    /* 通常どおりディスパッチ */
    bio_list_pop(&tg->service_queue->bio_lists[rw]);
    throtl_dispatch_bio(td, tg, bio);
}
```

## io.weight: 比例配分スケジューリング {#ioweight-proportional-scheduling}

BFQ I/Oスケジューラが必要である（Kyberは重みをサポートせず、mq-deadlineとnoneもサポートしない）:

```bash
# 重みを設定（1-10000、デフォルト 100）:
echo "default 200" > /sys/fs/cgroup/high-prio/io.weight  # デフォルトの 2 倍
echo "default 50"  > /sys/fs/cgroup/low-prio/io.weight   # デフォルトの 0.5 倍

# デバイスごとの重み:
echo "8:0 200" > /sys/fs/cgroup/high-prio/io.weight  # /dev/sda のみ
echo "default 100" >> /sys/fs/cgroup/high-prio/io.weight  # その他はデフォルト

# BFQ の下では io.weight は WFQ（Weighted Fair Queuing）を実装する
# high-prio は帯域の 200/(200+50+100) = 57% を得る
# low-prio は帯域の 50/(200+50+100) = 14% を得る
# normal は帯域の 100/(200+50+100) = 29% を得る
```

### BFQの重みベースのスケジューリング {#bfq-weight-based-scheduling}

BFQ（Budget Fair Queuing）は、重みベースの公平性を実装する比例配分型のI/Oスケジューラである:

```c
/* block/bfq-iosched.c */
struct bfq_group {
    struct bfq_entity entity;  /* スケジューリングエンティティ */
    struct bfq_sched_data sched_data;  /* アクティブ/アイドルのキュー */
    /* ... */
};

/* 重みは cgroup の io.weight 経由で設定される: */
static void bfq_pd_init(struct blkg_policy_data *pd)
{
    struct bfq_group *bfqg = blkg_to_bfqg(pd->blkg);
    bfqg->entity.weight = pd->cgroup_weight; /* io.weight から */
    bfqg->entity.new_weight = bfqg->entity.weight;
}
```

## io.latency: レイテンシSLA {#iolatency-latency-sla}

`io.latency` はレイテンシベースのI/O保護を提供する。Linux 4.19でJosef Bacikによって導入された [(commit)](https://git.kernel.org/linus/d70675121546c35feaceebf7ed9caed8716640f3) [(LWN)](https://lwn.net/Articles/758963/)。cgroupのレイテンシが目標値を超えると、コントローラは競合するcgroupをスロットリングする:

```bash
# cgroup の目標レイテンシを設定:
echo "8:0 target=10ms" > /sys/fs/cgroup/important/io.latency

# カーネルはこの cgroup のレイテンシを監視する。
# p99 レイテンシが 10ms を超えると、競合する cgroup をスロットリングする。

# io.latency は SLA であり、ハードな制限ではない:
# - 保護された cgroup の I/O が優先される
# - 競合する cgroup は io.max の予算が残っていてもスロットリングされる

# スロットリングが起きているか確認:
cat /sys/fs/cgroup/important/io.stat
# 8:0 rbytes=... wbytes=... rios=... wios=... dbytes=... dios=...
# rbytes=完了した読み込みバイト数
# dbytes=discard バイト数（TRIM/DISCARD 操作）
```

### io.latencyの仕組み {#how-iolatency-works}

```c
/* block/blk-iolatency.c */
/* iolatency モニタは cgroup ごとに I/O 単位のレイテンシを追跡する: */

struct iolatency_grp {
    struct blkg_rwstat stats;         /* 方向別の統計 */
    struct blk_iolatency *blkiolat;
    u64     min_lat_nsec;             /* io.latency で設定した目標値 */
    u64     cur_win_nsec;             /* 現在の監視ウィンドウ */
    atomic64_t window_start;
    atomic_t  scale_cookie;          /* 競合する cgroup 用のスロットリング cookie */
    /* ... */
};

/* I/O が完了したとき: */
static void blkcg_iolatency_done_bio(struct rq_qos *rqos, struct bio *bio)
{
    u64 now = ktime_get_ns();
    u64 bio_issue_time = bio->bi_issue.value;
    u64 lat = now - bio_issue_time;

    /* ウィンドウ内のレイテンシを記録 */
    iolatency_record_time(iolat, lat);

    /* 目標値を超えたか確認 */
    if (lat > iolat->min_lat_nsec)
        iolatency_check_latencies(iolat, now);
}
```

## io.stat: I/O統計 {#iostat-io-statistics}

```bash
# cgroup のデバイスごとの I/O 統計:
cat /sys/fs/cgroup/mycgroup/io.stat
# 8:0 rbytes=1234567890 wbytes=987654321 rios=12345 wios=9876 \
#     dbytes=0 dios=0 wait_usec=123456789 io_service_bytes=... \
#     io_wait_time=...

# フィールド:
# rbytes/wbytes: 完了した読み込み/書き込みバイト数
# rios/wios:     読み込み/書き込み I/O 回数
# dbytes/dios:   discard のバイト数/回数
# wait_usec:     キューでの待ち時間の合計（µs）
```

## ライトバックの帰属 {#writeback-attribution}

ダーティページのライトバックは、そのページをダーティにしたcgroupに帰属させることができる:

```bash
# ライトバックの統計を確認:
cat /sys/fs/cgroup/mycgroup/io.stat
# dbytes/dios = discard（TRIM）のバイト数/回数。ライトバックは wbytes に計上される

# ライトバックのスロットリングを調整:
echo 80 > /proc/sys/vm/dirty_ratio           # RAM の 80% をダーティのしきい値にする
echo "8:0 wbps=10485760" > /sys/fs/cgroup/mycgroup/io.max  # ライトバックをスロットリング
```

```c
/* mm/page-writeback.c: ライトバックは inode の所有関係によって帰属が決まる */
/* inode がある cgroup によって作成された場合、そのライトバックはその cgroup に計上される */
```

## コンテナのI/O隔離 {#container-io-isolation}

Docker/containerdは、コンテナの隔離にio.maxとio.weightを使う:

```bash
# Docker: デバイスの読み込みレートを設定
docker run --device-read-bps /dev/sda:50mb --device-write-bps /dev/sda:25mb \
    my-image

# Docker: I/O の重みを設定
docker run --blkio-weight 500 my-image   # 500/1000 = デフォルトの 0.5 倍

# Kubernetes: リソースの requests/limits は内部で io.max を使う:
# resources:
#   limits:
#     blkio.throttle.read_bps_device: "52428800"  # 50MB/s
```

## 参考文献 {#further-reading}

- [cgroup v2のアーキテクチャ](cgroup-v2.md) — cgroup v2の階層とコントローラ
- [リソースコントローラ](cgroup-controllers.md) — memory、cpu、ioの各コントローラ
- [I/Oスケジューラ](../block/io-schedulers.md) — io.weightが使うBFQスケジューラ
- [NVMeドライバ](../block/nvme.md) — ioコントローラの下にあるブロックデバイス
- `block/blk-throttle.c` — io.maxのスロットリング実装
- `block/blk-iolatency.c` — io.latencyの実装
- `block/bfq-iosched.c` — 重みをサポートするBFQ I/Oスケジューラ
