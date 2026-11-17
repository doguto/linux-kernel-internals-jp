# I/Oスケジューラ {#io-schedulers}

> ブロック層が効率のためにリクエストをどのように並べ替えるか

## I/Oスケジューラが行うこと {#what-io-schedulers-do}

I/Oスケジューラ（エレベータとも呼ばれる）は、ブロック層の発行パスとハードウェアへのディスパッチの間に位置する。次のことができる。

1. 隣接するリクエストを1つに**マージ**する（I/O操作が減る）
2. HDDのシーク距離を減らしたり公平性を改善したりするために、リクエストを**並べ替える**
3. 特定の種類のリクエスト（メタデータ、対話的なI/O）を**優先する**
4. I/Oを使いすぎているプロセスからのリクエストを**スロットリングする**

現代的なblk-mqハードウェア（NVMe、virtio-blk）では、スケジューラはオプションである — 多数のキューを持つNVMeでは、"none"（スケジューラなし）が最適であることが多い。

### スケジューラがblk-mqにどう組み込まれるか {#how-a-scheduler-plugs-into-blk-mq}

blk-mqのスケジューラは、少数のフックを公開する`elevator_type`である — 主なものは`bio_merge`（到着したbioのマージを試みる）、`insert_requests`（スケジューラ自身のキューにリクエストを受け入れる）、`dispatch_request`（ハードウェアキューに空きがあるときに次のリクエストを渡す）である。スケジューラに挿入されたリクエストは、ハードウェアに直行するのではなく*そこで待つ*。リクエストが出ていく順序はスケジューラの`dispatch_request`が決める。デバイスが持つスケジューラは最大1つであり、sysfsを通じて実行時に切り替えられる — ライブのI/Oに対してアトミックに行われるが、この性質は歴史的にuse-after-freeバグの発生源となってきた。

重要な洞察は、スケジューラが元を取れるのは、並べ替えの価値がそのオーバーヘッドを上回るときだけだということである。回転ディスクでは、並べ替えによってミリ秒単位のシーク時間を節約できる。シークのペナルティがなく数百万IOPSをこなすNVMeデバイスでは、スケジューラのリクエストごとの管理処理が節約分よりも多くの*コストがかかる*ことがある — だからこそ`none`が存在し、それがしばしば正解となるのである。

## 利用可能なスケジューラ {#available-schedulers}

```bash
# デバイスで利用可能なスケジューラを確認する
cat /sys/block/sda/queue/scheduler
# [mq-deadline] none bfq kyber
# 括弧内が現在のスケジューラ

# スケジューラを変更する
echo mq-deadline > /sys/block/sda/queue/scheduler
echo none > /sys/block/nvme0n1/queue/scheduler  # NVMe向け
```

### none {#none}

スケジューリングを行わない。リクエストは発行順にディスパッチされる。最適なのは次の場合である。

- ハードウェアキューを持つNVMe SSD（シークレイテンシがなく、並べ替えによる利点がない）
- スケジューラのオーバーヘッドが問題になる、非常に低レイテンシなワークロード

```bash
echo none > /sys/block/nvme0n1/queue/scheduler
```

### mq-deadline {#mq-deadline}

**ほとんどのブロックデバイスのデフォルト。** 各リクエストにデッドラインを与えることで、リクエストの飢餓を防ぐ。読み込みと書き込みの2つの別々のキューを使う。Jens AxboeによりLinux 4.11で導入された — [`945ffb60c11d`](https://git.kernel.org/linus/945ffb60c11dfb228130f3f2bdde961cecb76671)。

アルゴリズム:
1. リクエストはエレベータ（セクタ順にソートされたrb-tree）でソートされる
2. 各リクエストには**デッドライン**も与えられる: 読み込みはデフォルト500ms、書き込みは5000ms
3. スケジューラは通常、セクタ順にディスパッチする（シークが減る）
4. いずれかのリクエストがデッドラインに達すると、即座にディスパッチされる

```c
/* block/mq-deadline.c */
struct deadline_data {
    struct rb_root sort_list[DD_DIR_COUNT];  /* セクタ順にソート */
    struct list_head fifo_list[DD_DIR_COUNT]; /* デッドライン順にソート */
    sector_t last_sector[DD_DIR_COUNT];
    unsigned int batching;    /* 現在のバッチ数 */
    unsigned int starved;     /* 書き込み飢餓のカウント */
};
```

**判断の仕方。** 各方向は同じリクエストを*2つの*構造で保持する。**セクタ**順にソートされた赤黒木と、**デッドライン**順にソートされたFIFOリストである。通常、mq-deadlineはrb-treeからセクタ順にディスパッチし、シークを最小化するためにエレベータのように前方へ掃引する。各バッチの前にFIFOの先頭をのぞき、最も古いリクエストがデッドラインを過ぎていれば、デッドライン順に切り替えてそのリクエストを処理する。これにより、どのリクエストも飢餓状態にならないことが保証される。`fifo_batch`は、方向とデッドラインを再評価するまでに1回の掃引でディスパッチするリクエスト数を設定する。読み込みは書き込みより優先される — 読み込みは通常同期的（何かがそれを待っている）であるのに対し、書き込みはバッファリングされるからである — そして`writes_starved`は、書き込みバッチが強制されるまでに実行できる読み込みバッチの数を制限する。後のバージョンではI/O優先度のサポートが追加され、RT / BE / IDLEの各クラスが独自のキューを持つようになったので、デッドラインの枠組みの中で優先度の高いI/Oが先に処理される。

チューニング:
```bash
# 読み込みのデッドライン（ms、デフォルト500）
cat /sys/block/sda/queue/iosched/read_expire
echo 100 > /sys/block/sda/queue/iosched/read_expire

# 書き込みのデッドライン（ms、デフォルト5000）
cat /sys/block/sda/queue/iosched/write_expire

# 書き込みに切り替える前に、読み込みリクエストをいくつディスパッチするか
cat /sys/block/sda/queue/iosched/writes_starved  # デフォルト2

# 方向を切り替える前にバッチ化するリクエスト数
cat /sys/block/sda/queue/iosched/fifo_batch  # デフォルト16
```

### BFQ（Budget Fair Queueing） {#bfq-budget-fair-queueing}

**デスクトップ/対話的なワークロードに最適。** Paolo ValenteによりLinux 4.12で導入された — [`aee69d78dec0`](https://git.kernel.org/linus/aee69d78dec0ffdf82e35d57c626e80dddc314d5)、[LWN](https://lwn.net/Articles/720675/)。比例配分のI/Oスケジューリングを提供し、重いバックグラウンドI/Oの下でも対話的なアプリケーション（ブラウザ、デスクトップUI）の低レイテンシを重視する。

BFQは、各プロセスまたはグループに処理すべきセクタの**バジェット**を割り当てる。バジェットを使い切ると、スケジューラは次のプロセスに移る。バジェットはワークロードの特性に基づいて動的にサイズが決まる。

主な特性:

- 対話的なプロセスは優先度がブーストされる（短いバーストが報われる）
- cgroupとの統合: `/sys/block/sda/queue/iosched/`がグループごとのウェイトを公開する
- `CONFIG_BFQ_GROUP_IOSCHED`がcgroupベースのI/O分離を有効にする

**内部の仕組み。** BFQは**WF2Q+**の上に構築された比例配分スケジューラである。WF2Q+は正確なフェアキューイングアルゴリズムで、キューを*仮想時間*の順に処理し、任意の区間で各プロセスやcgroupがウェイトに比例した帯域を受け取るようにする。各キューにはセクタ単位の**バジェット**が与えられる。BFQはバジェットを使い切るかキューがアイドルになるまでそのキューを処理し、その後仮想時間によって次のキューを選ぶ。この公平性の核の上に、BFQはヒューリスティックを重ねている。**対話的**および**ソフトリアルタイム**なアプリケーション — 思考時間を挟んだ短いI/Oバースト — を検出し、それらのウェイトを一時的に倍増させる（「weight raising」）ことで、重いバックグラウンドコピーの最中でもデスクトップの応答性を保つ。この正確さにはすべて代償がある。リクエストごとの仮想時間とヒューリスティックの管理処理は高価であり、まさにそれゆえにBFQはHDDや対話的なデスクトップでは輝くが、高IOPSのNVMeではスループットのボトルネックになる。

```bash
echo bfq > /sys/block/sda/queue/scheduler

# BFQのチューニング可能なパラメータ
cat /sys/block/sda/queue/iosched/slice_idle    # リクエスト間のアイドル時間（µs）
cat /sys/block/sda/queue/iosched/back_seek_max  # 許容される後方シークの最大値

# プロセスごとのI/O優先度（ionice）
ionice -c 2 -n 4 rsync source/ dest/   # best-effort、優先度4
ionice -c 3 rsync source/ dest/         # idleクラス
```

### kyber {#kyber}

**マルチキューの低レイテンシデバイス（NVMe、SSD）向け。** Omar SandovalによりLinux 4.12で導入された — [`00e043936e9a`](https://git.kernel.org/linus/00e043936e9a1c274c29366c7ecd9e17c79418e6)、[LWN](https://lwn.net/Articles/720675/)。操作の種類ごと（読み込み、書き込み、discard）のキューを使い、トークンベースでレイテンシ目標を狙う。

並べ替えを行う代わりに、kyberは次のようにする。

- 目標レイテンシを達成するために、カテゴリごとの処理中リクエスト数を制限する
- マージを行わない（ハードウェアに任せる）
- オーバーヘッドが非常に低い

**内部の仕組み。** Kyberはエレベータではなく、フィードバック制御器である。リクエストを種類（読み込み、書き込み、discard、その他）ごとにグループ化し、種類ごとの処理中リクエスト数 — その「トークン」 — に上限を設ける。各種類が実際に達成しているレイテンシを継続的に*計測*し、それぞれを目標（`read_lat_nsec`、`write_lat_nsec`）近くに保つようにトークン数を上げ下げする。トークンが少なければキューは浅くなり、スループットをいくらか犠牲にしてレイテンシが下がる。逆もまた然りであり、Kyberは自動的にバランスを見つける。セクタでソートすることがなく、管理処理もほとんど行わないため、オーバーヘッドはごくわずかである — 目的は、すでに高速なデバイスでのレイテンシの*制御*であって、シークの削減ではない。

```bash
echo kyber > /sys/block/nvme0n1/queue/scheduler

# レイテンシ目標（ns）
cat /sys/block/nvme0n1/queue/iosched/read_lat_nsec    # デフォルト2000000（2ms）
cat /sys/block/nvme0n1/queue/iosched/write_lat_nsec   # デフォルト10000000（10ms）
```

## スケジューラの選び方 {#choosing-a-scheduler}

| ワークロード | 推奨スケジューラ |
|----------|----------------------|
| NVMe SSD | `none`または`kyber` |
| SATA SSD | `mq-deadline`または`none` |
| HDD（回転ディスク） | `mq-deadline`または`bfq` |
| デスクトップ/対話的 | `bfq` |
| データベースサーバー | `mq-deadline`または`none` |
| コンテナのI/O分離 | `bfq`（cgroupと併用） |

```bash
# 回転ディスクかどうかを確認する（0=SSD、1=HDD）
cat /sys/block/sda/queue/rotational

# 起動時の自動スケジューラ選択（udevルール）
# /usr/lib/udev/rules.d/60-io-scheduler.rules
ACTION=="add|change", KERNEL=="sd[a-z]*", ATTR{queue/rotational}=="0",
  ATTR{queue/scheduler}="mq-deadline"
ACTION=="add|change", KERNEL=="nvme[0-9]*",
  ATTR{queue/scheduler}="none"
```

## I/O性能の監視 {#monitoring-io-performance}

```bash
# iostat: スループット、IOPS、レイテンシ
iostat -x 1 /dev/sda
# Device    r/s   w/s  rkB/s  wkB/s  await  svctm  %util
# sda       0.5  50.2   8.0  801.0    4.5    1.2   6.1

# await: 平均合計レイテンシ（キュー待ち + サービス時間）
# svctm: デバイスでの平均サービス時間
# %util: デバイスの利用率

# blktrace: リクエスト単位の詳細なトレーシング
blktrace -d /dev/sda -o trace
blkparse -i trace.blktrace.0 | head -50

# biolatency（BCC）: I/Oレイテンシのヒストグラム
biolatency -d sda 10 1
```

## 参考文献 {#further-reading}

- [blk-mq](blk-mq.md) — スケジューラの下にあるディスパッチ層
- [bioとrequestの構造体](bio-request.md) — スケジューラが並べ替えるもの
- カーネルツリー内の`Documentation/block/`
