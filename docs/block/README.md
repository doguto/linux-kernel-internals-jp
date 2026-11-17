# ブロック層 {#block-layer}

> カーネルがストレージI/Oをファイルシステムからデバイスまでどのように運び、その途中でマージ、並べ替え、スケジューリングを行うか

## はじめに {#getting-started}

ブロック層は、ストレージI/Oを*生み出す*カーネルの部分 — ファイルシステム、ページキャッシュのライトバックパス、スワップ、ダイレクトI/Oの発行者 — と、実際のデバイス（NVMe、SCSI、virtio-blk、...）と通信するドライバとの間に位置する。その仕事は、`bio`の発行の流れを効率的なデバイスリクエストの列に変換することである。隣接するI/Oのマージ、発行のバッチ化、I/Oスケジューラの適用、そしてデバイスのハードウェアキューへの作業のマッピングを行う。

このドキュメントはAPIだけでなく設計も説明する — なぜブロック層が独立した層として存在するのか、なぜ現代のSSDのために複数のキュー（`blk-mq`）を中心に書き直されたのか、そしてその結果として生まれた構造（`bio`、`request`、`request_queue`、ソフトウェアキューとハードウェアキュー）が実際に何をしているのか。

```
Filesystem / writeback / swap / O_DIRECT
    ↓ submit_bio()
Block layer core   ──►  (optional) I/O scheduler: BFQ · mq-deadline · Kyber · none
    ↓ blk-mq: software queue (per-CPU) → hardware queue (per device queue)
Device driver: NVMe · SCSI · virtio-blk · ...
    ↓ DMA
Storage hardware
```

### 略史: なぜblk-mqか {#a-brief-history-why-blk-mq}

元のブロック層は、デバイスごとに単一のリクエストキューを使い、それを単一のロックで保護していた — 数百IOPSをこなす回転ディスクには問題なかったが、数百万IOPSをこなすSSDにとってはスケーリングの壁だった。これを解決するために、Linux 3.13でマルチキューブロック層（`blk-mq`）が導入された（[`320ae51feed5`](https://git.kernel.org/linus/320ae51feed5)、"blk-mq: new multi-queue block IO queueing mechanism"）。各CPUがデバイスのハードウェアキューにマッピングされる独自のソフトウェアキューを持ち、共有ロックがなくなった。ドライバはその後数年かけて移行し、ついにblk-mqが*唯一の*パスとなった。レガシーなシングルキューのI/Oスケジューラ（CFQ、deadline）はLinux 5.0で削除され（[`f382fb0bcef4`](https://git.kernel.org/linus/f382fb0bcef4)、"block: remove legacy IO schedulers"）、古いリクエストパスもそれとともに消えた。今日のブロックドライバはすべてblk-mqである。

### 前提知識 {#prerequisites}

- **Cと基本的なOSの概念** — カーネルはCで書かれている。ドライバ、DMA、カーネル/ユーザー空間の境界といった考え方に慣れておくこと。
- **ページキャッシュとライトバック**を知っていると役立つ。ほとんどのブロックI/Oはそこから発生するからである — [ページキャッシュ](../mm/page-cache.md)と[ページキャッシュとライトバック](../io/page-cache-writeback.md)を参照。

### ソースの入手 {#getting-the-source}

```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
```

関連するディレクトリは`block/`（コア、blk-mq、スケジューラ）、`drivers/nvme/`、`drivers/scsi/`、`drivers/block/`（virtio-blk、loop、`null_blk`）である。

### 実験する {#experimenting}

`null_blk`は、ハードウェアなしでブロック層を動かす最も簡単な方法である — 設定可能なインメモリのブロックデバイスを提供する。`blktests`はブロック層のテストスイートであり、`fio`はI/Oパターンを生成・計測する標準的なツールである。

### 推奨される読む順番 {#suggested-reading-order}

1. **[ブロック層の概要](block-overview.md)** — 主要な構造体と、発行パスの端から端まで
2. **[ブロックI/Oの一生](life-of-block-io.md)** — 1つのI/Oを`submit_bio()`からデバイスまで下り、また戻ってくるまで追跡する
3. **[bioとrequestの構造体](bio-request.md)** — I/O記述子オブジェクトと、`bio`がどのように`request`になるか
4. **[blk-mq: マルチキューブロック層](blk-mq.md)** — ソフトウェア/ハードウェアキュー、タグの割り当て、ドライバインターフェース
5. **[I/Oスケジューラ](io-schedulers.md)** — BFQ、mq-deadline、Kyber: それぞれが何を最適化するか
6. **[ブロックcgroup（I/O制御）](blk-cgroup.md)** — デバイスの帯域をcgroup間で分配する
7. **[NVMeドライバ](nvme.md)** — 実際の現代的なドライバが、サブミッション/コンプリーションキューを通じてどのようにblk-mqに組み込まれるか
8. **[Device Mapper: dm-verity](dm-verity.md)** — 整合性検証のために仮想ブロックデバイスをスタックする
9. **[可観測性](block-observability.md)** — カウンタからblktraceまで、ブロック層が何をしているかを見る
10. **[事件簿](war-stories.md)** — ブロック層の実際のバグと、それが教えてくれたこと

### 学べること {#what-youll-learn}

| 教科書的な考え方 | Linuxの現実 |
|---|---|
| 「OSがディスクに読み書きを送る」 | I/Oは[`bio`](bio-request.md)で記述され、`request`としてマージ・キューイングされ、[I/Oスケジューラ](io-schedulers.md)によって並べ替えられ、ハードウェアキューにディスパッチされる |
| 「ディスクごとに1つのリクエストキュー」 | [blk-mq](blk-mq.md)は各CPUに、デバイスのハードウェアキューにマッピングされるソフトウェアキューを与える — 共有ロックはない |
| 「I/Oスケジューラが処理を速くする」 | 高速なSSDでは最良のスケジューラは*none*であることが多い。スケジューリングは主にレイテンシを抑え、公平性を強制するためのものである（[スケジューラ](io-schedulers.md)） |
| 「`write()`はディスクに行く」 | 通常は[ページキャッシュ](../io/page-cache-writeback.md)に行く。ブロック層がそれを目にするのは後のライトバック時である |

## ドキュメント {#documentation}

| ドキュメント | 学べること |
|---|---|
| [ブロック層の概要](block-overview.md) | 主要な構造体（`bio`、`request`、`request_queue`）と発行パス |
| [ブロックI/Oの一生](life-of-block-io.md) | 1つのI/Oを発行から完了まで端から端まで追跡する |
| [bioとrequestの構造体](bio-request.md) | I/O記述子オブジェクトとそのライフサイクル |
| [blk-mq: マルチキューブロック層](blk-mq.md) | ソフトウェア/ハードウェアキュー、タグ、現代的なドライバインターフェース |
| [I/Oスケジューラ](io-schedulers.md) | BFQ、mq-deadline、Kyber |
| [ブロックcgroup（I/O制御）](blk-cgroup.md) | cgroupごとのデバイス帯域の分配（`io.max` / `io.weight` / `io.latency`） |
| [NVMeドライバ](nvme.md) | PCIe SSDのアーキテクチャ、サブミッション/コンプリーションキュー、blk-mqとの統合 |
| [Device Mapper: dm-verity](dm-verity.md) | Merkleツリーによるブロック整合性検証（Android、ChromeOS） |
| [可観測性](block-observability.md) | ブロックI/Oのための`/proc/diskstats`、`blktrace`、BPF |
| [事件簿](war-stories.md) | ブロック層の実際のインシデントとその教訓 |

## 参考文献 {#further-reading}

- [Kernel docs: block layer](https://docs.kernel.org/block/index.html) — ブロックサブシステムの権威あるリファレンス
- [`320ae51feed5`](https://git.kernel.org/linus/320ae51feed5) — blk-mqの導入（Linux 3.13）
- [`f382fb0bcef4`](https://git.kernel.org/linus/f382fb0bcef4) — レガシーなシングルキュースケジューラの削除（Linux 5.0）
