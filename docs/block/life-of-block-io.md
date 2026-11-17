# ブロックI/Oの一生 {#life-of-a-block-io}

> 読み込みや書き込みを`submit_bio()`からブロック層を通ってデバイスまで追跡し、完了時に再び上へ戻るまでをたどる

すべてのブロックI/Oには2つの半分がある。リクエストをハードウェアに向かって*下へ*運ぶ**発行パス**と、結果を要求元へ*上へ*運び戻す**完了パス**である。その間で、ブロック層は自らの存在を正当化する仕事を行う — マージ、バッチ化、スケジューリング、そしてハードウェアキューへのマッピングである。このページでは、1つのI/Oが両方の半分を通過する様子を追う。

以下の関数名は、特に断りがない限り[`block/blk-mq.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/blk-mq.c)と[`block/blk-core.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/blk-core.c)のものを指す。

## 出発点: bio {#the-starting-point-a-bio}

[`bio`](bio-request.md)は1つのI/Oを記述する。対象デバイス、開始セクタ、方向、そして読み込み先または書き込み元となるメモリセグメントのリストである。I/Oを必要とする者 — ファイルシステム、[ページキャッシュのライトバック](../io/page-cache-writeback.md)パス、スワップ、あるいは`O_DIRECT`の発行者 — は`bio`を埋めて`submit_bio()`を呼ぶ。

## 1. 発行: `submit_bio()` → `blk_mq_submit_bio()` {#1-submission-submit_bio-blk_mq_submit_bio}

`submit_bio()`は`submit_bio_noacct()`に流れ込み、そこで検証とI/Oの計上を行った後、`bio`を`blk_mq_submit_bio()`に渡す — これがマルチキュー機構への入口である。ここからブロック層が最初にとる本能的な行動は、新しいrequestを作ることでは*なく*、それを避けることである。

## 2. 新しいrequestを避けられるか？ マージ {#2-can-we-avoid-a-new-request-merging}

何かを割り当てる前に、ブロック層は`bio`を、まだディスパッチされていない既存のrequestに取り込もうとする。

- **`blk_attempt_plug_merge()`** — 現在のタスクは、保留中のrequestの短い*plug*リストを保持している（ステップ4を参照）。そのいずれかと連続している`bio`は、キューロックを取ることなく、その場でマージされる。
- **`blk_mq_sched_bio_merge()`** — I/Oスケジューラが接続されていれば、そのキューにすでに存在するrequestに`bio`をマージする機会が与えられる。

**バックマージ**（`bio`の開始セクタがrequestの終端と等しい）や**フロントマージ**は、単に既存のrequestを新しいセグメントまで覆うように拡張する。マージがあるからこそ、シーケンシャルな4 KBの書き込みの流れが、何千もの小さなrequestではなく、ひと握りの大きなrequestになる — これはブロック層がもたらす最大の効率化である。

## 3. 割り当て: bioがrequestになる {#3-allocation-a-bio-becomes-a-request}

何もマージできなかった場合、`blk_mq_get_new_requests()`は、ハードウェアキューのtag setから空き**タグ**を確保することで`request`を割り当てる。そのタグは、以後のライフタイムを通じてrequestの識別子となる — デバイスまで下っていき、完了時に戻ってきて、どのI/Oが終わったかを識別する。

タグの数は有限である（キュー深さ）。空きがなければ、発行者は返却されるまで**ブロック**する。これがブロック層の自然なバックプレッシャーである。追いつけないデバイスは、明示的なレート制限なしに、自動的に発行者をスロットリングする。

## 4. plug: スループットのためのバッチ化 {#4-plugging-batching-for-throughput}

即座にディスパッチするのではなく、新しいrequestは通常タスクの**plug**に追加される。I/Oのバーストを発行しようとするコードは、それを`blk_start_plug()` / `blk_finish_plug()`で囲む — 例えばライトバックパスは、フラッシュ処理の実行の周りでこれを行う。plugが保持されている間はrequestが溜まっていくので、マージ（ステップ2）の材料が増え、ハードウェアを叩くコストが償却される。plugがフラッシュされると、そのrequestはディスパッチへと向かう。

## 5. スケジューリング: 順序と公平性 {#5-scheduling-order-and-fairness}

[I/Oスケジューラ](io-schedulers.md) — BFQ、mq-deadline、Kyber — が接続されていれば、requestはそのキューで待ち、スケジューラが**ディスパッチ順序**を決める。テールレイテンシを抑える、cgroup間で帯域を公平に分配する、回転メディアでシークを減らす、といった目的である。高速なSSDではスケジューラは**`none`**に設定されることが多く、requestはハードウェアキューに直行する。デバイスにシークのペナルティがなく、数百万IOPSの余裕があるときには、並べ替えで得られるものはほとんどないからである。

## 6. ディスパッチ: ハードウェアキューへ {#6-dispatch-into-the-hardware-queue}

`blk_mq_run_hw_queue()` → `blk_mq_dispatch_rq_list()`は準備のできたrequestをたどり、それぞれをドライバの`->queue_rq()`コールバックに渡す — NVMeでは`nvme_queue_rq()`（[`drivers/nvme/host/pci.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/nvme/host/pci.c)）であり、SCSIやvirtio-blkにも相当するものがある。ドライバは汎用的なrequestをデバイス固有のコマンドに変換し、それを**ハードウェアのサブミッションキュー**に置き（requestのタグで識別される）、デバイスのドアベルを鳴らす。制御は発行者に戻り、I/Oはこれで処理中（in flight）となる。

## 7. 完了: スタックを上へ戻る {#7-completion-back-up-the-stack}

やがてデバイスが処理を終え、それを通知する — 通常は**割り込み**であり、レイテンシが重要なI/Oでは**[ポーリング](../io/io-polling.md)**である。ドライバは完了を読み取り、タグでrequestを探し出し、`blk_mq_complete_request()`を呼ぶ。データのキャッシュを温かく保つため、完了処理はしばしば（IPIを介して）I/Oを発行したCPUに戻され、そこで`blk_mq_end_request()` → `bio_endio()`が実行される。

`bio_endio()`は`bio`の`->bi_end_io`コールバックを呼び出す — *元の*発行者が設定したフックである。各呼び出し元にとっての物語はここで終わる。同期的な読み込み側が起こされる、ページが最新（up-to-date）とマークされてアンロックされる、あるいはライトバックの完了処理がデータが永続化されたことを記録する。タグは解放され、それによってステップ3でブロックしていた発行者が解放されることもある。

## 往復 {#the-round-trip}

```
 submit_bio()                                     bi_end_io() callback
     │  submit_bio_noacct()                              ▲  (wake / mark up-to-date)
     ▼                                                   │
 blk_mq_submit_bio()                              bio_endio()
     │  try merge (plug / scheduler)                     ▲
     ▼  else allocate request (claim a tag)      blk_mq_end_request()
 plug → (I/O scheduler) → hardware queue                 ▲
     │  blk_mq_dispatch_rq_list()               blk_mq_complete_request()
     ▼  ->queue_rq()                                     ▲  (IRQ or poll)
 device  ───────────────  in flight  ──────────────────  device
```

## 試してみる {#try-it-yourself}

`blktrace`は、稼働中のデバイスについて、このパスのすべての段階 — `Q`（キューに入った）、`M`（マージされた）、`G`（リクエストの取得）、`I`（挿入された）、`D`（ディスパッチされた）、`C`（完了した） — を記録する。

```bash
blktrace -d /dev/nvme0n1 -o - | blkparse -i -
```

`bpftrace`は、ブロックのtracepoint（`block:block_bio_queue`、`block:block_rq_issue`、`block:block_rq_complete`）にアタッチすることで、同じ遷移を観察できる。

## 参考文献 {#further-reading}

- [Kernel docs: block layer](https://docs.kernel.org/block/index.html)
- [`block/blk-mq.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/blk-mq.c) — 発行と完了の機構
- [`320ae51feed5`](https://git.kernel.org/linus/320ae51feed5) — このマルチキューパスを確立したblk-mqの導入（Linux 3.13）
