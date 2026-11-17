# I/O優先度 {#io-priorities}

> `ioprio_set()`、優先度クラス、cgroupのI/Oウェイト、そしてブロックスケジューラが優先度をどう使うか

## I/O優先度とは何か {#what-io-priority-means}

CPUスケジューリングには強制的なプリエンプションがある。優先度の高いタスクは、優先度の低いタスクを命令の途中でCPUから追い出すことができる。I/O優先度はこれより弱く、**助言的（advisory）** なものである。ブロックスケジューラはそれを尊重しようとするが、BFQにおけるRTクラスを除いて厳密な保証はしない。

優先度が影響するもの:

- スケジューラのキューからの **リクエストのディスパッチ順序** — RTリクエストはBEより先に、BEはIDLEより先に処理される
- **帯域の割り当て** — BFQはプロセスごとのウェイトに比例してI/O帯域を割り当て、そのウェイトはioprioから導かれる
- **レイテンシ目標** — BFQのレイテンシモードは、実際のレイテンシが設定された目標を超えたcgroupを優遇（ブースト）する

優先度が影響 *しない* もの:

- `none`スケジューラはioprioを完全に無視する（スケジューラがないので、優先度の強制もない）
- `mq-deadline`はioprioを順序付けにのみ使い、帯域には使わない
- ページキャッシュのライトバックは、どのプロセスのioprioにも結びつかない（ライトバックはカーネルスレッドとして動作する）

---

## ioprio_set()とioprio_get() {#ioprioset-and-ioprioget}

glibcのラッパーはなく、これらは生のシステムコールである。

```c
#include <linux/ioprio.h>
#include <sys/syscall.h>

/* 呼び出し元プロセスの I/O 優先度を設定する */
int ret = syscall(SYS_ioprio_set,
                  IOPRIO_WHO_PROCESS,   /* 対象: プロセス */
                  0,                    /* 0 = 現在のプロセス */
                  IOPRIO_PRIO_VALUE(IOPRIO_CLASS_BE, 0));

/* 特定の PID の I/O 優先度を取得する */
int prio = syscall(SYS_ioprio_get, IOPRIO_WHO_PROCESS, pid);
int class = IOPRIO_PRIO_CLASS(prio);   /* クラスを取り出す */
int level = IOPRIO_PRIO_DATA(prio);    /* レベルを取り出す */
```

### 対象セレクタ {#target-selectors}

| 定数 | 意味 |
|---|---|
| `IOPRIO_WHO_PROCESS` | PIDで指定する単一のプロセス（またはスレッド） |
| `IOPRIO_WHO_PGRP` | プロセスグループ内の全プロセス |
| `IOPRIO_WHO_USER` | あるUIDが所有する全プロセス |

### 優先度クラス {#priority-classes}

```c
/* include/uapi/linux/ioprio.h */
#define IOPRIO_CLASS_NONE  0   /* CPU の nice 値から継承 */
#define IOPRIO_CLASS_RT    1   /* リアルタイム: 最初に処理される */
#define IOPRIO_CLASS_BE    2   /* ベストエフォート: デフォルト */
#define IOPRIO_CLASS_IDLE  3   /* アイドル: 他に保留中のものがないときのみ */
```

### 優先度値の組み立て {#building-a-priority-value}

```c
/* クラスとレベルを単一の u16 にエンコードする */
#define IOPRIO_PRIO_VALUE(class, data) \
    (((class) << IOPRIO_CLASS_SHIFT) | (data))

/* 例 */
IOPRIO_PRIO_VALUE(IOPRIO_CLASS_RT,   0)  /* RT、最高レベル */
IOPRIO_PRIO_VALUE(IOPRIO_CLASS_BE,   4)  /* BE、レベル 4 — デフォルト */
IOPRIO_PRIO_VALUE(IOPRIO_CLASS_IDLE, 0)  /* IDLE（レベルは無視される） */
```

RTとBEにおける優先度レベルは **0（最高）から7（最低）** までである。IDLEクラスには意味のあるレベルはない。

### ioniceコマンド {#the-ionice-command}

`ionice`は、これらのシステムコールのユーザー空間ラッパーである。

```bash
# PID の現在の I/O 優先度を確認する
ionice -p 1234

# 実行中のプロセスを BE クラス、レベル 0（BE で最高）に設定する
ionice -c 2 -n 0 -p $(pgrep postgres)

# プロセスを IDLE クラスで起動する
ionice -c 3 rsync -av /data /backup

# プロセスを RT クラス、レベル 0 で起動する（CAP_SYS_ADMIN が必要）
ionice -c 1 -n 0 bash -c 'exec ./latency-critical-job'

# シェルのサブツリー内の全プロセスに ioprio を設定する
ionice -c 2 -n 0 bash -c 'exec your-critical-job'
```

---

## 優先度クラスの詳細 {#priority-classes-in-detail}

### RT（リアルタイム） {#rt-real-time}

RTは最も高い優先度クラスである。ブロックスケジューラ（BFQまたはmq-deadline）は、同じキュー内のどのBEやIDLEのリクエストよりも先にRTリクエストをディスパッチする。

- **レベル**: 0〜7。0がRT内で最高
- **`CAP_SYS_ADMIN`が必要** — 非特権プロセスはRTクラスを設定できない
- **リスク**: RTプロセスが絶えず発行し続けると、RTは優先度の低いI/Oを完全に飢餓状態にしうる
- **典型的な用途**: リアルタイムのオーディオ再生、産業用制御システム、レイテンシの上限が必要なVMのストレージI/O

```bash
# プロセスに RT I/O を与える — root か CAP_SYS_ADMIN で実行する必要がある
ionice -c 1 -n 0 -p $(pgrep jackd)
```

BFQでは、RTキューはどのBEキューに順番が回るよりも前に、別個のバジェットラウンドで処理される。mq-deadlineでは、RTリクエストはBEやIDLEのFIFOより先に空にされる高優先度のFIFOに置かれる。

### BE（ベストエフォート） {#be-best-effort}

BEはすべてのプロセスのデフォルトのクラスである。明示的な`ioprio_set()`が呼ばれていなければ、プロセスは`IOPRIO_CLASS_BE`に属し、そのレベルはCPUのnice値から導かれる。

```
ioprio_level = (cpu_nice + 20) / 5
```

したがって`nice 0` → BEレベル4（中間点）、`nice -20` → BEレベル0、`nice 19` → BEレベル7となる。

- **レベル**: 0〜7（0 = I/O帯域が最も多い、7 = 最も少ない）
- BFQは帯域を比例配分する。レベル0のプロセスは、レベル7のプロセスより1ラウンドあたり多くのセクタを得る
- CFQ（Linux 5.0で削除）はBEプロセスをクラス内で厳密に順序付けていた。BFQは重み付き公平キューを使う

### IDLE {#idle}

IDLEクラスのプロセスは、RTやBEのI/Oが保留中でないときにのみ処理される。このため、フォアグラウンドのI/Oとまったく競合すべきでないバックグラウンドのワークロードに最適である。

```bash
# バックグラウンドのバックアップ — 他の何も遅くしない
ionice -c 3 nice -n 19 rsync -av /data /backup

# バックグラウンドタスクとしてのファイルシステムのスクラブ
ionice -c 3 btrfs scrub start /

# 本当のバックグラウンド作業には ionice -c 3 と CPU の nice を組み合わせる
ionice -c 3 nice -n 19 find / -name '*.log' -mtime +30 -delete
```

IDLEクラスはBFQにおいて、IDLEキューからディスパッチする前に、RTまたはBEの`bfq_queue`に保留中のリクエストがあるかどうかを確認することで実装されている。

---

## 優先度がカーネル内をどう流れるか {#how-priority-flows-through-the-kernel}

`ioprio_set()`で設定された優先度は、`task_struct`からブロックスケジューラまで運ばれなければならない。完全な経路は次のとおりである。

```
ioprio_set() syscall
  → stored in task_struct->ioprio

write() / read() syscall entry
  → init_sync_kiocb() / init_kiocb()
  → kiocb.ki_ioprio = get_current_ioprio()

bio submission (submit_bio / iomap_dio_rw)
  → bio->bi_ioprio = kiocb->ki_ioprio

blk-mq request allocation (blk_mq_get_request)
  → rq->ioprio = bio->bi_ioprio

Scheduler (BFQ) enqueue
  → bfq_queue->ioprio from rq->ioprio
  → RT queues served before BE before IDLE
  → weight within BE class derived from ioprio level
```

### get_current_ioprio() {#getcurrentioprio}

```c
/* include/linux/ioprio.h */
static inline int get_current_ioprio(void)
{
    return __get_task_ioprio(current);
}

static inline int __get_task_ioprio(struct task_struct *p)
{
    struct io_context *ioc = p->io_context;
    int prio;

    if (!ioc)   /* io_context がない: CPU の nice からクラス + レベルを導出する */
        return IOPRIO_PRIO_VALUE(task_nice_ioclass(p), task_nice_ioprio(p));

    prio = ioc->ioprio;
    if (IOPRIO_PRIO_CLASS(prio) == IOPRIO_CLASS_NONE)  /* クラス NONE: nice を使う */
        prio = IOPRIO_PRIO_VALUE(task_nice_ioclass(p), task_nice_ioprio(p));
    return prio;
}
```

プロセスが`io_context`を持ち、`ioprio_set()`で明示的なioprioが設定されていれば、その値が使われる。そうでなければioprioはCPUのnice値から導かれ、2つのスケジューリングの次元をゆるやかに結びつけている。

### bio->bi_ioprioの割り当て {#bio-bi_ioprio-assignment}

```c
/* block/blk-core.c — bio は発行元タスクから ioprio を継承する */
static void bio_set_ioprio(struct bio *bio)
{
    /* まだ誰も ioprio を設定していない? タスクの（nice 由来の）ioprio で初期化する */
    if (IOPRIO_PRIO_CLASS(bio->bi_ioprio) == IOPRIO_CLASS_NONE)
        bio->bi_ioprio = get_current_ioprio();
    blkcg_set_ioprio(bio);   /* その後 cgroup の I/O 優先度ポリシーを適用する */
}
```

ダイレクトI/Oでは、bioは発行元kiocbの`ki_ioprio`を運ぶ（`fs/iomap/direct-io.c`でDIOのbioが構築されるときにコピーされる）。バッファード書き込みでは、bioは後でライトバック経路によって作られるため、そのioprioは発行元プロセスではなくcgroup/ライトバックのコンテキストから来る。

---

## cgroup v2のI/Oウェイト {#cgroup-v2-io-weight}

**cgroup v2** では、I/O優先度はプロセス単位ではなくcgroup単位で管理される。cgroupの階層は、個々の`task_struct->ioprio`の値を上書きまたは補完する。

### io.weight {#ioweight}

```bash
# cgroup の I/O ウェイトを設定する（範囲 1〜10000、デフォルト 100）
echo "100" > /sys/fs/cgroup/myapp/io.weight

# デバイスごとのウェイト上書き（major:minor weight）
echo "8:16 200" > /sys/fs/cgroup/myapp/io.weight

# 現在のウェイトを読む
cat /sys/fs/cgroup/myapp/io.weight
# default 100
# 8:16 200

# 兄弟 cgroup に対する相対的なウェイトを設定する
# ウェイト 200 の cgroup は、ウェイト 100 の cgroup の 2 倍の I/O 帯域を得る
echo "200" > /sys/fs/cgroup/highprio/io.weight
echo "50"  > /sys/fs/cgroup/background/io.weight
```

`io.weight`はタスクごとのioprioレベルには変換 **されない**。これは別個の比例配分の仕組みである。その値（1〜10000、デフォルト100）はブロックI/Oコントローラ（BFQ、または他のスケジューラで`io.weight`を支える`blk-iocost`のコストモデル）によって直接消費され、兄弟cgroupの間でウェイトに比例してデバイスの帯域を分割する。ウェイト200のcgroupは、ウェイト100のcgroupのおよそ2倍の取り分を得る。タスクごとの`ioprio`（`ionice`による）とcgroupごとの`io.weight`は独立したつまみである。前者はBFQの優先度クラス内でリクエストを順序付け、後者はcgroup間で帯域を分割する。

### io.bfq.weight {#iobfqweight}

BFQは独自のウェイトインターフェースを直接公開している。

```bash
# BFQ のウェイト: 1〜1000（デフォルト 100）
echo "500" > /sys/fs/cgroup/myapp/io.bfq.weight

# デバイスごとの BFQ ウェイト
echo "8:16 500" > /sys/fs/cgroup/myapp/io.bfq.weight

# 現在の BFQ ウェイトを読む
cat /sys/fs/cgroup/myapp/io.bfq.weight
# default 100
# 8:16 500
```

`io.bfq.weight`は、`io.weight`が適用する中間変換なしに、BFQの内部ウェイトに直接対応する。

### プロセスをcgroupに移す {#moving-a-process-into-a-cgroup}

```bash
# cgroup を作成する
mkdir /sys/fs/cgroup/background

# 低い I/O ウェイトを設定する
echo "10" > /sys/fs/cgroup/background/io.weight

# プロセスをそこへ移す
echo $PID > /sys/fs/cgroup/background/cgroup.procs

# これ以降、そのプロセスが発行するすべてが cgroup のウェイトを使う
```

---

## io.latency: レイテンシ目標 {#iolatency-latency-targeting}

`io.latency`を使うと、cgroupは目標とするI/Oレイテンシを宣言できる。BFQはcgroupが目標を満たしているかを監視し、レイテンシが閾値を超えるとそのウェイトを一時的に引き上げる。

```bash
# デバイス 8:16 に 10ms の読み出しレイテンシ目標を設定する
echo "8:16 10000" > /sys/fs/cgroup/critical/io.latency

# 現在のレイテンシ目標を読む
cat /sys/fs/cgroup/critical/io.latency
# 8:16 10000

# 複数のデバイス
echo "8:0 5000"  > /sys/fs/cgroup/db/io.latency   # sda に 5ms
echo "8:16 5000" >> /sys/fs/cgroup/db/io.latency  # sdb に 5ms
```

レイテンシは **マイクロ秒** で指定する。

### io.latencyがBFQとどう連携するか {#how-iolatency-works-with-bfq}

```
Cgroup submits I/O request
  → BFQ records submission timestamp

I/O completes
  → BFQ computes actual latency
  → compares to io.latency target

If actual > target:
  → blkcg_iolatency_throttle() temporarily boosts cgroup weight
  → competing cgroups are throttled until latency recovers

If actual <= target consistently:
  → weight returns to nominal io.weight value
```

これは、大量のバックグラウンドワークロードとストレージを共有するデータベースで特に有用である。

```bash
# データベースの cgroup: 低いレイテンシ目標
mkdir /sys/fs/cgroup/db
echo "100" > /sys/fs/cgroup/db/io.weight
echo "8:0 5000" > /sys/fs/cgroup/db/io.latency  # 目標 5ms

# バックアップの cgroup: レイテンシ目標なし、低いウェイト
mkdir /sys/fs/cgroup/backup
echo "10" > /sys/fs/cgroup/backup/io.weight

# プロセスを移す
echo $DB_PID     > /sys/fs/cgroup/db/cgroup.procs
echo $BACKUP_PID > /sys/fs/cgroup/backup/cgroup.procs
```

これで、バックアップによってデータベースのレイテンシが5msを超えて跳ね上がると、BFQはデータベースが目標内に戻るまでバックアップのcgroupをスロットリングする。

---

## BFQ: Budget Fair Queueing {#bfq-budget-fair-queueing}

BFQは、ioprioとcgroupのI/Oウェイトを最も完全に尊重するスケジューラである。多くのデスクトップや汎用のLinuxシステムでデフォルトになっている。

```bash
# BFQ が有効か確認する
cat /sys/block/sda/queue/scheduler
# [bfq] mq-deadline none

# BFQ を有効にする
echo bfq > /sys/block/sda/queue/scheduler
```

### プロセスごとのキュー {#per-process-queues}

BFQは、プロセス（またはcgroupエンティティ）ごとに`bfq_queue`を保持する。

```c
/* block/bfq-iosched.h */
struct bfq_queue {
    struct bfq_data     *bfqd;           /* 所有者のスケジューラ */
    struct rb_root       sort_list;      /* セクタ順に並べたリクエスト */
    struct request      *next_rq;        /* 次にディスパッチするリクエスト */
    unsigned short       ioprio;         /* ioprio レベル */
    unsigned short       ioprio_class;   /* IOPRIO_CLASS_* */
    int                  max_budget;     /* このラウンドで処理するセクタ数 */
    unsigned long        budget_timeout;
    /* ... */
};
```

各`bfq_queue`は、1回のアクティベーションにつき最大`max_budget`セクタまで処理される。バジェットを使い切る（またはタイムアウトする）と、BFQは次のキューへ移る。`max_budget`自体はBFQによって自動調整され（`bfq_default_max_budget`、16Kセクタから開始）、ioprioレベルから導かれるものでは *ない*。ioprioレベルが制御するのはキューの **ウェイト** であり、それがBFQの比例配分（WF2Q+）スケジューラにおけるディスク時間の取り分を決める。

### ioprioレベルからウェイトへ {#from-ioprio-level-to-weight}

```
weight ← ioprio level  (block/bfq-wf2q.c):

  weight = (IOPRIO_NR_LEVELS − level) × BFQ_WEIGHT_CONVERSION_COEFF
         = (8 − level) × 10

  BE level 0 → weight 80
  BE level 4 → weight 40   ← default ioprio
  BE level 7 → weight 10
```

したがってBEレベル0のプロセスはBEレベル7のプロセスの8倍のウェイト（80対10）を持ち、それに応じてデバイスのより大きな取り分を受け取る。

### 優先度クラスの順序付け {#priority-class-ordering}

BFQは厳密なクラス階層、すなわちRTがBEより先、BEがIDLEより先という順序を強制する。この順序付けは単純な`if`の連鎖ではなく、BFQの階層型スケジューラから自然に生じるものである。`bfq_select_queue()`（`block/bfq-iosched.c`）は、`bfq_get_next_queue()`を介してBFQのサービスツリー階層をたどることで次に処理するキューを選び、その階層はクラスによって順序付けられている。保留中のI/Oを持つRTキューは常にどのBEキューよりも先に選ばれ、BEキューはIDLEより先に選ばれる。BEクラス内では、キューは（ioprioレベルから導かれる）キューごとのウェイトに従って重み付き公平な順序で処理される。

### ソフトリアルタイムの検出 {#soft-real-time-detection}

BFQは、アイドル期間（閾値を超える思考時間）で区切られた短いI/Oのバーストを観察することで、**インタラクティブ** なアプリケーション（Webブラウザ、テキストエディタ）をヒューリスティックに検出する。このようなキューはBEクラスにあっても一時的にブーストされる。これが「ソフトリアルタイム」への昇格である。内部的にはこれは **ウェイトレイジング（weight raising）** として実装されている。BFQは一定期間、キューのウェイトに係数を掛け（`bfqq->wr_coeff > 1`）、ブーストされたキューが通常のBEキューより先にスケジュールされるようにする。プロセス生成の「大規模バースト」の一部としてフラグが立てられたキュー（`bfq_bfqq_in_large_burst()`）は除外される。短命なプロセスの嵐は、このヒューリスティックが対象とするインタラクティブなパターンではないからである。

### シーク性の検出 {#seeky-detection}

BFQは、キューが **シーケンシャル** なI/Oを発行しているか **ランダム**（シーク多発）なI/Oを発行しているかを追跡し、平均シーク距離が大きい場合には`BFQQ_SEEKY()`マクロによってキューをシーク多発（seeky）と分類する。シーク多発のキューには異なるバジェットが与えられる。ランダムI/Oは長く途切れないサービス時間をうまく活用できないため、BFQはそのバジェットに上限を設け、シーク多発のキューがほとんど有用な仕事をせずにデバイスを占有できないようにする。これにより、持続的なアクセスの恩恵を受けるシーケンシャルなワークロードが、ランダムアクセスのキューによって飢餓状態にされることを防ぐ。

### io_uringとBFQ {#io_uring-and-bfq}

アプリケーションレベルのキューイングが使われると、io_uringはBFQのプロセスごとのモデルを混乱させることがある。SQPOLLや固定ファイルを使うと、複数のプロセスが単一のio_uringコンテキストを通じてI/Oを発行することがある。BFQにはプロセスごとのキューではなく単一のキューとして見え、プロセスごとのウェイトによる差別化が損なわれる可能性がある。cgroupベースの分離には、io_uringを使う場合、プロセスごとのioprioより`io.weight`のほうが信頼できる。

---

## mq-deadline: より単純な優先度サポート {#mq-deadline-simpler-priority-support}

`mq-deadline`は、多くのシステムでNVMeやSCSIデバイスのデフォルトのスケジューラである。その優先度サポートはBFQより単純で、順序付けのみを行い、帯域の比例配分は行わない。

```bash
# mq-deadline が有効か確認する
cat /sys/block/nvme0n1/queue/scheduler
# [mq-deadline] none bfq
```

### mq-deadlineがioprioをどう扱うか {#how-mq-deadline-handles-ioprio}

mq-deadlineは優先度クラスごとに別々のFIFOキューを保持する。

```c
/* block/mq-deadline.c */
enum dd_prio {
    DD_RT_PRIO   = 0,   /* IOPRIO_CLASS_RT */
    DD_BE_PRIO   = 1,   /* IOPRIO_CLASS_BE */
    DD_IDLE_PRIO = 2,   /* IOPRIO_CLASS_IDLE */
    DD_PRIO_MAX  = 2,
};

/* 優先度クラスごとに 1 組のキュー（DD_DIR_COUNT = 2: 読み出し、書き込み） */
struct dd_per_prio {
    struct rb_root       sort_list[DD_DIR_COUNT];  /* セクタ順（マージ用） */
    struct list_head     fifo_list[DD_DIR_COUNT];  /* デッドライン順（飢餓防止） */
    /* ... */
};

struct deadline_data {
    struct dd_per_prio   per_prio[DD_PRIO_COUNT];  /* RT、BE、IDLE（DD_PRIO_COUNT = 3） */
    /* ... */
};
```

ディスパッチの順序:
1. RTの読み出し / RTの書き込みリクエスト（デッドライン順、次にセクタ順）
2. BEの読み出し / BEの書き込みリクエスト
3. IDLEのリクエスト — RTとBEのFIFOが空のときのみ

クラス内では、mq-deadlineは **デッドライン**（飢餓防止のタイマー）順、次に **セクタ順**（シークを減らす）でディスパッチする。クラス内のレベルは区別 **しない**。BEレベル0とBEレベル7はmq-deadlineでは同一に扱われる。

```bash
# mq-deadline の優先度のチューニング
cat /sys/block/sda/queue/iosched/read_expire   # デフォルト 500ms
cat /sys/block/sda/queue/iosched/write_expire  # デフォルト 5000ms
cat /sys/block/sda/queue/iosched/prio_aging_expire  # 低優先度が繰り上がるまでの ms
```

---

## ioniceの実践 {#ionice-in-practice}

### バックグラウンドの保守ジョブ {#background-maintenance-jobs}

```bash
# ファイルシステムのバックアップ — IDLE クラス、フォアグラウンドの I/O に影響なし
ionice -c 3 rsync -av --progress /data /backup/

# CPU 優先度の引き下げと組み合わせる
ionice -c 3 nice -n 19 tar -czf /backup/archive.tar.gz /var/lib/pgsql/

# btrfs balance — 非常に I/O 負荷が高くなりうる
ionice -c 3 btrfs balance start -dusage=50 /

# 本番環境を乱さずに大きなファイルを見つけて処理する
ionice -c 3 find /var/log -name '*.log' -size +100M -exec gzip {} \;
```

### インタラクティブおよびデータベースのI/O {#interactive-and-database-io}

```bash
# 高優先度のデータベースバックアップ（RT には CAP_SYS_ADMIN が必要）
ionice -c 1 -n 0 pg_basebackup -D /backup/pgbase

# 実行中のデータベースの I/O を改善する（BE クラス、レベル 0）
ionice -c 2 -n 0 -p $(pgrep -x postgres | head -1)

# プロセスが現在どの ionice クラスを使っているか確認する
ionice -p $(pgrep mysqld)
# best-effort: prio 4
```

### 実行中のプロセスの調整 {#adjusting-a-running-process}

```bash
# 再起動せずに実行中のプロセスの I/O 優先度を下げる
PID=$(pgrep -x bacula-fd)
ionice -c 3 -p $PID        # IDLE に降格する
ionice -p $PID             # 確認する

# 時間がかかりすぎている停滞したバックアップの I/O 優先度を上げる
ionice -c 2 -n 0 -p $BACKUP_PID
```

### ioprio_setを直接使ったプログラム {#scripting-with-ioprio_set-directly}

```c
#include <sys/syscall.h>
#include <linux/ioprio.h>
#include <unistd.h>
#include <stdio.h>

int main(void)
{
    /* バックグラウンド作業の前に、このプロセスを IDLE クラスに設定する */
    int ret = syscall(SYS_ioprio_set,
                      IOPRIO_WHO_PROCESS,
                      0,
                      IOPRIO_PRIO_VALUE(IOPRIO_CLASS_IDLE, 0));
    if (ret < 0) {
        perror("ioprio_set");
        return 1;
    }

    /* ここから先の I/O はすべて IDLE クラス */
    do_background_scan();
    return 0;
}
```

---

## 優先度逆転 {#priority-inversion}

I/Oスタックにおける微妙な危険として、高優先度のプロセスが、低優先度のプロセスがリソースを解放するのを待ってブロックされることがある。

### pageロックの競合 {#page-lock-contention}

```
High-priority process:   wants to read page at offset X
  → page is locked (PG_locked set)
  → waits on page lock

Low-priority process:    holds PG_locked (doing writeback or readahead)
  → scheduled less frequently by I/O scheduler
  → high-priority process is effectively at low-priority
```

これはページキャッシュ層における優先度逆転の一形態である。高優先度のプロセスは *スケジューラで* ブロックされているのではなく、スケジューラより上のpageロックでブロックされているため、I/Oスケジューラは助けにならない。

### inodeとファイルシステムのロック {#inode-and-filesystem-locks}

同様に、低優先度のプロセスがtruncateや書き込みの間`inode->i_rwsem`を保持していると、同じファイルを読みたい高優先度のプロセスは、そのioprioにかかわらず、その間ブロックされる。

### 部分的な緩和策: IOCB_NOWAIT {#partial-mitigation-iocb_nowait}

`IOCB_NOWAIT`は、ロックやリソースがすぐに利用できない場合に、ブロックする代わりに`-EAGAIN`を返すようファイルシステムとブロック層に指示する。

```c
/* 待つ余裕のない高優先度のコード */
kiocb->ki_flags |= IOCB_NOWAIT;
ret = file->f_op->read_iter(kiocb, &iter);
if (ret == -EAGAIN) {
    /* リソースが一時的に利用できない — 処理するか後で再試行する */
    schedule_retry();
}
```

io_uringは、すべてのノンブロッキング操作でデフォルトで`IOCB_NOWAIT`を使い、`-EAGAIN`が返されたときにのみワーカースレッドにフォールバックする。これにより、低速パスでワーカースレッドを使うというコストと引き換えに、優先度逆転を回避する。

### 補償の仕組みとしてのcgroupのio.latency {#cgroup-iolatency-as-a-compensating-mechanism}

`io.latency`は間接的に役立つことがある。これは（BFQとは独立した）**blk-iolatency** コントローラによって実装されている。保護したいcgroupに目標レイテンシを設定すると、そのcgroupの観測レイテンシが目標を超えたとき、コントローラは保護されたcgroupが回復するまで *競合する* cgroupをスロットリングし、それらに許されるI/Oの深さを減らす。ロックに起因する逆転を解消することはできないが、低優先度のI/Oを絞ることで、レイテンシに敏感なcgroupの遅延を一定範囲に抑える。これは封じ込めの仕組みであり、予防ではない。

---

## カーネルのデータ構造のまとめ {#kernel-data-structures-summary}

```
task_struct
  └── io_context->ioprio (u16) ← set by ioprio_set(), read by get_current_ioprio()

struct kiocb
  └── ki_ioprio (u16)         ← copied from the task's ioprio at I/O submission

struct bio
  └── bi_ioprio (u16)         ← copied from kiocb; carries priority to block layer

struct request
  └── ioprio (u16)            ← copied from bio; seen by the scheduler

struct bfq_queue
  └── ioprio                  ← derived from request->ioprio on queue creation
  └── ioprio_class            ← RT / BE / IDLE
  └── weight                  ← (8 − level) × 10; sets share of disk time
  └── max_budget              ← sectors per round (auto-tuned, not weight-derived)
```

---

## 適切な仕組みの選び方 {#choosing-the-right-mechanism}

| シナリオ | 推奨されるアプローチ |
|---|---|
| 単一のバックグラウンドプロセス | `ionice -c 3`（IDLEクラス） |
| 競合する複数のアプリケーション | cgroup v2の`io.weight` |
| レイテンシSLAのあるデータベース | cgroup v2の`io.latency` |
| リアルタイムオーディオ / ハードウェア制御 | `ionice -c 1 -n 0`（RT、rootが必要） |
| クラウドVMのストレージ分離 | コンテナごとのcgroup v2 `io.weight` |
| NVMeのワークロード | BFQまたはmq-deadline。純粋なスループットには`none`を検討 |
| io_uringを多用するアプリケーション | プロセスごとのioprioではなくcgroupの`io.weight` |

---

## I/O優先度の動作を観測する {#observing-io-priorities-in-action}

```bash
# プロセスの全スレッドの ioprio を確認する
for tid in /proc/$(pgrep postgres)/task/*/; do
    pid=$(basename $tid)
    echo -n "tid $pid: "
    ionice -p $pid
done

# デバイスごとの BFQ キューの統計
cat /sys/block/sda/queue/iosched/stats

# blktrace: I/O トレース内の ioprio を見る
blktrace -d /dev/sda -o - | blkparse -i - | grep 'prio'

# ftrace: BFQ のディスパッチの判断をトレースする
echo 1 > /sys/kernel/debug/tracing/events/block/block_rq_issue/enable
cat /sys/kernel/debug/tracing/trace_pipe | grep -E 'prio|ioprio'

# cgroup の I/O ウェイトが適用されているか確認する
cat /sys/fs/cgroup/myapp/io.stat  # BFQ の cgroup ごとの統計
```

---

## 主要なソースファイル {#key-source-files}

| ファイル | 内容 |
|---|---|
| `include/uapi/linux/ioprio.h` | `IOPRIO_CLASS_*`定数、`IOPRIO_PRIO_VALUE()`、`IOPRIO_PRIO_CLASS()`マクロ |
| `fs/ioprio.c` | `ioprio_set()` / `ioprio_get()`システムコールの実装 |
| `kernel/sched/core.c` | `get_current_ioprio()` — `io_context`から読むか、niceから導出する |
| `block/bfq-iosched.c` | BFQスケジューラ: キューごとのバジェット、ウェイト、RT/BE/IDLEクラスのディスパッチ |
| `block/bfq-iosched.h` | `struct bfq_queue`、`struct bfq_data`の定義 |
| `block/mq-deadline.c` | mq-deadline: クラスごとのFIFOキューとデッドラインによるディスパッチ |
| `block/blk-ioprio.c` | cgroupのI/O優先度との統合 — `io.weight`をioprioに対応付ける |
| `block/blk-cgroup.c` | cgroup blkioの中核 — `blkcg_get_queue()`、ウェイトの継承 |

---

## 参考文献 {#further-reading}

- [I/Oスケジューラ](../block/io-schedulers.md) — BFQ、mq-deadline、none、Kyber
- [blk-mq](../block/blk-mq.md) — マルチキューブロック層、リクエストのライフサイクル
- [ダイレクトI/O](direct-io.md) — ioprioが最も確実にスケジューラへ届くO_DIRECT経路
- [非同期I/O](async-io.md) — io_uringとBFQのプロセスごとのモデルとの相互作用
- [cgroupsのI/O](../cgroups/io-cgroup.md) — io.weight、io.max、io.latencyのリファレンス
- [可観測性](observability.md) — 優先度の強制を検証するためのblktrace、ftrace、iostat
