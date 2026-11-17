# ストリーミングおよび高スループットワークロード向けのI/Oチューニング {#tuning-io-for-streaming-and-high-throughput-workloads}

> 動画、バックアップ、ETL、データパイプラインのワークロードにおけるシーケンシャルな読み書きのスループットを最大化する

## ストリーミングI/Oとはどのようなものか {#what-streaming-io-looks-like}

ストリーミングワークロードは、データを高いレートでシーケンシャルに読み書きする。動画のエンコード/デコード、バックアップエージェント、ログ集約、ETLパイプライン、Kafkaブローカー、Hadoop/Sparkなどである。そのI/Oパターンは次のとおりである:

```
Streaming write pattern:
Offset: 0     4MB    8MB    12MB   16MB   ...
        ████  ████  ████  ████  ████  →  sequential, large, aligned

Streaming read pattern:
Offset: 0     4MB    8MB    12MB   16MB   ...
        ████→ ████→ ████→ ████→ ████→    sequential scan, one pass

Contrast with random I/O:
Offset: 12MB  1MB   7MB   3MB   15MB   ...
        ████  ████  ████  ████  ████       random jumps, small sizes
```

カーネルのデフォルト設定は、先読みとライトバックの結合によってすでにシーケンシャルI/Oを優遇している — しかし、*もっぱら*シーケンシャルなワークロードにとっては、デフォルトのままでは相当なスループットを取りこぼしている。

---

## 先読み: ストリーミング読み出しの主要なつまみ {#readahead-the-primary-lever-for-streaming-reads}

カーネルの先読みの仕組みは、現在の読み出し位置より先のデータをディスクからプリフェッチする。ストリーミング読み出しでは、積極的な先読みがそのままスループットにつながる。アプリケーションが以前に取得したデータを処理している間も、デバイスは読み出しで忙しく動き続ける。

```bash
# 現在の先読みサイズ
cat /sys/block/sda/queue/read_ahead_kb    # デフォルト: 128KB

# ストリーミング読み出しでは大幅に増やす
echo 4096 > /sys/block/sda/queue/read_ahead_kb   # 4MB の先読み
echo 8192 > /sys/block/nvme0n1/queue/read_ahead_kb  # NVMe では 8MB

# 最適な値は次に依存する:
# - ストレージのレイテンシ（レイテンシが高いほど、デバイスを忙しく保つために大きな先読みが必要）
# - キャッシュに使えるメモリ
# - シークする前にどれだけのデータをシーケンシャルに読むか
```

**最適な先読みウィンドウの計算:**

先読みウィンドウは、アプリケーションが処理している間もデバイスを忙しく保てる大きさにすべきである。2GB/sを出し、128KBのリクエストあたり0.1msのレイテンシを持つデバイスの場合:

```
Without readahead (synchronous reads):
  App requests 128KB → device serves in 64µs → app processes → repeat
  Throughput: 128KB / 64µs ≈ 2GB/s (device-limited, but only if app is fast enough)

With 4MB readahead:
  Device is 4MB ahead of the app at all times
  App can process for up to 4MB / 2GB/s = 2ms before needing more data
  This allows substantial CPU processing between reads without stalling
```

**`posix_fadvise`によるファイルごとの先読み:**

```c
/* このファイルがシーケンシャルに読まれることをカーネルに伝える — 大きな先読みを引き起こす */
posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL);

/* 読んだ後にページキャッシュに残す必要のないファイル（ストリーミング/一度きり）の場合 */
posix_fadvise(fd, 0, 0, POSIX_FADV_DONTNEED);

/* ストリーミングの一度きりの読み出し向けの組み合わせ: 積極的な先読み、キャッシュ汚染なし */
posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL);
/* ... チャンクを読む ... */
posix_fadvise(fd, processed_offset, chunk_size, POSIX_FADV_DONTNEED);
```

処理済みのデータに対して`POSIX_FADV_DONTNEED`を使うと、ページキャッシュが一度きりのデータで埋まり、ほかのプロセスのワーキングセットのpageが追い出されるのを防げる。これは、ストリーミングワークロードとほかのサービスの両方を動かしているサーバーでは極めて重要である。

---

## 書き込みバッファリング: ページキャッシュにバーストを吸収させる {#write-buffering-letting-the-page-cache-absorb-bursts}

ストリーミング書き込みでは、ページキャッシュが書き込みバッファとして働く。書き込みは即座にページキャッシュに入り、ライトバックスレッドによって非同期にストレージへフラッシュされる。アプリケーションは、ダーティの上限までは、デバイスが吸収できるより速く書き込むことができる。

```bash
# ストリーミング書き込みワークロード向けのチューニング

# スロットリングが始まるまでにメモリ上に置けるダーティデータを増やす
# デフォルト: RAM の 20%。ストリーミング専用サーバーでは 40〜60%
echo 40 > /proc/sys/vm/dirty_ratio

# このしきい値でバックグラウンドのライトバックが始まる
# デフォルト: 10%。ストリーミングでは 20〜30%
echo 20 > /proc/sys/vm/dirty_background_ratio

# フラッシュまでダーティデータを長く置いておく（小さなバッチによる
# 書き込み増幅を減らす）。デフォルト: 3000（30s）。ストリーミングでは 6000（60s）
echo 6000 > /proc/sys/vm/dirty_expire_centisecs

# ライトバックの実行頻度を下げる（フラッシュあたりのバッチを大きくする）
# デフォルト: 500（5s）。ストリーミングでは 1000（10s）
echo 1000 > /proc/sys/vm/dirty_writeback_centisecs
```

!!! warning "ダーティ比率とクラッシュのリスク"
    `dirty_ratio`が高いということは、メモリ上にあってまだディスクに載っていないデータが多いということである。RAMの40%がダーティな状態でシステムクラッシュや電源障害が起きると、数ギガバイトのデータが失われうる。高いダーティ比率を使うのは、データ損失が許容できるワークロード（一時的なデータ、容易に再生成できるデータ）か、ストレージがバッテリーバックアップ付きのライトキャッシュを持つ場合に限ること。

---

## 大きなI/Oサイズとアライメント {#large-io-size-alignment}

ストリーミングワークロードは、大きくアラインされたチャンク単位で書き込むべきである。カーネルは大きなシーケンシャルI/Oを1つのBIO（ブロックI/Oリクエスト）として発行でき、デバイスの効率を最大化できる。

```bash
# デバイスが効率よく処理できる最大リクエストサイズを確認
cat /sys/block/nvme0n1/queue/max_sectors_kb   # 例: 512
cat /sys/block/sda/queue/max_hw_sectors_kb    # ハードウェアの最大値

# ストリーミング書き込みでは、I/O サイズをデバイスが好むチャンクに合わせる:
# NVMe: 256KB〜1MB のチャンク
# SATA SSD: 128KB〜256KB のチャンク
# HDD: 1MB〜8MB のチャンク（回転待ちレイテンシの償却）
```

**アプリケーションコードでは:**

```c
/* ストリーミング書き込み: 大きなアラインされたバッファを使う */
#define STREAM_BUF_SIZE (1 * 1024 * 1024)  /* 1MB */

void *buf;
posix_memalign(&buf, 4096, STREAM_BUF_SIZE);

while (has_data()) {
    size_t n = fill_buffer(buf, STREAM_BUF_SIZE);
    write(fd, buf, n);  /* カーネルが大きな BIO にまとめる */
}
```

io_uringでは、`IORING_OP_WRITEV`によるベクタ書き込みを使って複数の256KBバッファを一度に発行し、syscallのオーバーヘッドを減らす。

---

## 一度きりのストリーミングではページキャッシュをバイパスする {#bypass-the-page-cache-for-single-pass-streaming}

データが一度だけ読み書きされる場合（バックアップダンプ、ログローテーション、動画エンコードの入力）、ページキャッシュはRAMを無駄にし、何の利益もなく有用なデータを追い出す。

**一度きりの読み出しの場合:**

```c
/* O_DIRECT: ページキャッシュをバイパスし、ユーザー空間のバッファへ直接読み込む */
int fd = open(path, O_RDONLY | O_DIRECT);

/* または: バッファード I/O を使いつつ、キャッシュしないよう助言する */
int fd = open(path, O_RDONLY);
posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL | POSIX_FADV_NOREUSE);
/* 各ブロックを読んだ後: */
posix_fadvise(fd, offset, block_size, POSIX_FADV_DONTNEED);
```

**一度きりの書き込みの場合（バックアップ、ダンプ）:**

```c
/* ページキャッシュでバッファリングせずにデバイスへ書き込む */
int fd = open(path, O_WRONLY | O_DIRECT);
/* 512 バイトまたは 4096 バイトにアラインされたバッファが必要 */
```

**共有サーバーでこれが重要な理由:**

バッファードI/Oで実行される100GBのバックアップは、二度と読まれない100GBのデータでページキャッシュを埋め、データベース、Webサーバー、その他の実行中プロセスのワーキングセットを追い出す。64GBのサーバーでは、キャッシュを埋め尽くすバックアップの後、ほかのサービスのワーキングセットがディスクから再びフォールトインされるまでの10〜20分間、それらのサービスに40〜60%の性能低下が生じることがある。

---

## 並列I/Oストリーム {#parallel-io-streams}

最新のNVMeデバイスは複数のハードウェアキューを持ち、並列I/O向けに設計されている。単一のシーケンシャルストリームではデバイスを飽和させられないことがある:

```bash
# NVMe: ハードウェアキューの数
cat /sys/block/nvme0n1/mq/     # キューごとに 1 つのディレクトリ

# テスト: 単一ストリームと複数ストリームの比較
# 単一ストリーム:
fio --name=single --rw=read --bs=1M --direct=1 \
    --numjobs=1 --iodepth=4 --filename=/dev/nvme0n1 \
    --size=10G --time_based --runtime=30

# 複数の並列ストリーム（NVMe では 2〜4 倍速いことが多い）:
fio --name=parallel --rw=read --bs=1M --direct=1 \
    --numjobs=4 --iodepth=4 --filename=/dev/nvme0n1 \
    --size=10G --time_based --runtime=30 --group_reporting
```

高速なNVMe（定格7GB/s）では、CPU/PCIeのボトルネックにより、単一ストリームでは4〜5GB/sしか出ないことがある。並列ストリームは複数のCPUコアとPCIeレーンに分散する。

---

## KafkaブローカーのI/Oチューニング {#kafka-broker-io-tuning}

Kafkaは典型的なストリーミングワークロードである。プロデューサーはパーティションログにシーケンシャルに書き込み、コンシューマーは同じログの異なるオフセットから同時に読み出す。

```bash
# Kafka の I/O パターン:
# - プロデューサー: パーティションファイルへのシーケンシャルな追記
# - コンシューマー: さまざまなオフセットからのシーケンシャル読み出し（先読みが不可欠）
# - レプリケーション: 最近のデータのシーケンシャル読み出し → 先読みの恩恵を共有

# Kafka ブローカー向けのカーネル設定:

# 大きなダーティバッファ（Kafka の書き込みはバースト的かつシーケンシャル）
echo 40 > /proc/sys/vm/dirty_ratio
echo 15 > /proc/sys/vm/dirty_background_ratio

# ダーティの有効期限を長めに（Kafka のセグメントは長寿命で、flush 設定による fsync が頻繁）
echo 3000 > /proc/sys/vm/dirty_expire_centisecs

# コンシューマーの読み出し向けに積極的な先読み
echo 4096 > /sys/block/nvme0n1/queue/read_ahead_kb

# これに合わせた Kafka アプリケーションの設定:
# log.flush.interval.messages=Long.MAX_VALUE  # フラッシュは OS に任せる
# log.flush.interval.ms=Long.MAX_VALUE        # 永続性はレプリケーションに頼る
# log.retention.bytes=107374182400            # パーティションあたり 100GB
```

**Kafkaがページキャッシュに頼る理由:**

Kafkaは意図的に独自のバッファプールを持たない。プロデューサーのすぐ後ろを追うコンシューマーの読み出し（よくあるケース: 最近書き込まれた同じデータ）は、ディスク読み出しなしにページキャッシュから直接応答される。カーネルの先読みは後続のセグメントを先回りしてキャッシュに取り込む。プロデューサーからコンシューマーへのこのゼロコピー経路がKafkaの性能の秘密である — Kafkaの性能に関するドキュメントと[splice/sendfileの内部](splice-sendfile.md)を参照。

---

## HDFS DataNodeのI/Oチューニング {#hdfs-datanode-io-tuning}

HadoopのDataNodeは、クライアントから大きなブロック（128MB〜256MB）を受け取り、ローカルストレージに書き込む。パターンは、シーケンシャルな書き込みの後に複数のクライアントからのシーケンシャル読み出しが続くというものである。

```bash
# DataNode のストレージ: 大きなシーケンシャル I/O、複数の同時ストリーム
# 通常はノードあたり 4〜12 台のドライブ（RAID ではなく JBOD）

# ディスクごとの先読み: レプリケーションの読み出しのため高く設定
for disk in /sys/block/sd*; do
    echo 8192 > $disk/queue/read_ahead_kb
done

# 大きなダーティバッファを許可（DataNode は相当な RAM を持つ）
echo 40 > /proc/sys/vm/dirty_ratio
echo 20 > /proc/sys/vm/dirty_background_ratio

# HDFS アプリケーションの設定:
# dfs.datanode.data.dir — ディスクごとに 1 つのディレクトリ（並列ストリーム）
# dfs.datanode.handler.count — ディスク数 × 2 に合わせる
# dfs.client.read.shortcircuit — ローカルクライアントの読み出し向けに有効化（ゼロコピー）
```

---

## 動画ストリーミングサーバーのI/Oチューニング {#video-streaming-server-io-tuning}

動画ストリーミングサーバーは大きなファイルをシーケンシャルに読み、クライアントへ送信する。ワーキングセットは大きく（アクティブな動画カタログ）、アクセスはファイル単位ではややランダムだが、各ファイル内ではシーケンシャルである。

```bash
# 中程度の先読み: ファイルごとのシーケンシャルアクセスに適する
# ただし大きすぎないように — 同時に開くファイルが多すぎると RAM を消費する
echo 2048 > /sys/block/nvme0n1/queue/read_ahead_kb  # ファイルあたり 2MB

# アプリケーションで各動画ファイルに posix_fadvise を使う:
# SEQUENTIAL: ファイルごとのより大きな先読みを有効化
# 送信後に DONTNEED: ストリーミング後にキャッシュを解放（ファイルが大きく一度きりの場合）

# ネットワークソケットへのゼロコピー配信には sendfile()
# カーネルはファイルから読み → NIC へ直接 DMA し、ユーザー空間へのコピーをバイパスする
ssize_t sent = sendfile(client_sockfd, file_fd, &offset, chunk_size);

# HTTP の範囲リクエスト（動画内のシーク）の場合:
posix_fadvise(fd, range_start, range_end - range_start, POSIX_FADV_WILLNEED);
```

---

## スループットの計測と検証 {#throughput-measurement-and-validation}

```bash
# fio: ストリーミングのベンチマーク
fio --name=stream_read \
    --rw=read \
    --bs=1M \
    --direct=1 \
    --numjobs=1 \
    --iodepth=32 \
    --filename=/dev/nvme0n1 \
    --size=20G \
    --time_based \
    --runtime=60 \
    --output-format=normal \
    --group_reporting

# 目標とするメトリクス:
# NVMe Gen4 シーケンシャル: 読み出し 5〜7 GB/s、書き込み 4〜6 GB/s
# SATA SSD シーケンシャル: 読み出し 400〜550 MB/s、書き込み 350〜500 MB/s
# HDD シーケンシャル: 読み出し 100〜200 MB/s、書き込み 80〜150 MB/s

# 先読みの有無で比較:
echo 0 > /sys/block/nvme0n1/queue/read_ahead_kb
fio --name=no_ra --rw=read --bs=4k --direct=0 ...   # 先読みなしのバッファード

echo 4096 > /sys/block/nvme0n1/queue/read_ahead_kb
fio --name=ra --rw=read --bs=4k --direct=0 ...      # 先読みありのバッファード
```

---

## クイックリファレンス: ストレージの種類ごとのストリーミングチューニング {#quick-reference-streaming-tuning-by-storage-type}

| 設定 | HDD | SATA SSD | NVMe |
|---------|-----|----------|------|
| `read_ahead_kb` | 8192 | 4096 | 2048〜4096 |
| `dirty_ratio` | 40% | 30% | 20〜30% |
| `dirty_background_ratio` | 20% | 15% | 10〜15% |
| `scheduler` | `mq-deadline` | `mq-deadline` | `none` |
| `nr_requests` | 256 | 256 | 1024 |
| I/Oサイズ | 1〜8MB | 256KB〜1MB | 256KB〜1MB |
| 並列ストリーム数 | 1〜2 | 2〜4 | 4〜8 |

---

## 関連ページ {#related-pages}

- [先読み](readahead.md) — カーネルの先読みアルゴリズムの仕組み
- [splice、sendfile、ゼロコピー](splice-sendfile.md) — ネットワークストリーミングのためのゼロコピー
- [ストレージI/Oのチューニング](tuning-storage.md) — sysctlの完全なリファレンス
- [バッファードI/Oとページキャッシュ](buffered-io.md) — バッファード書き込みの仕組み
