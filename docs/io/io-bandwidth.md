# I/O帯域幅: 計測と最適化 {#io-bandwidth-measurement-and-optimization}

> 実際のI/O帯域幅をどう計測するか、限界はどこから来るのか、そしてどう限界に近づくか

## I/O帯域幅を制限するものは何か {#what-limits-io-bandwidth}

I/O帯域幅 — ストレージとメモリの間を移動する1秒あたりのバイト数 — は、経路の中で最も弱い部分によって上限が決まる:

```
Application → VFS → Page Cache → Block Layer → PCIe Bus → Storage Device
                                                    ↑
                                              The bottleneck
                                              is usually here
                                              or the device itself

Typical limits:
  SATA III link:       600 MB/s
  PCIe 3.0 ×4 (NVMe): ~3.5 GB/s
  PCIe 4.0 ×4 (NVMe): ~7 GB/s
  PCIe 5.0 ×4 (NVMe): ~14 GB/s
  DDR5 memory bus:     ~77 GB/s per channel
```

実際には、限界はハードウェアインターフェースではなく、ソフトウェアの経路にあることが多い。CPUのオーバーヘッド、キューの直列化、ファイルシステムのオーバーヘッド、あるいはメモリサブシステムが十分な速さでデータを供給できるかどうかである。

---

## デバイスの帯域幅を計測する {#measuring-device-bandwidth}

### `fio`: 信頼できるベンチマーク {#fio-the-authoritative-benchmark}

```bash
# シーケンシャル読み出しのスループット（O_DIRECT、ページキャッシュをバイパス）
fio --name=seq_read \
    --rw=read \
    --bs=1M \
    --direct=1 \
    --numjobs=1 \
    --iodepth=32 \
    --filename=/dev/nvme0n1 \
    --size=20G \
    --time_based \
    --runtime=30 \
    --group_reporting

# シーケンシャル書き込みのスループット
fio --name=seq_write \
    --rw=write \
    --bs=1M \
    --direct=1 \
    --numjobs=1 \
    --iodepth=32 \
    --filename=/dev/nvme0n1 \
    --size=20G \
    --time_based \
    --runtime=30 \
    --group_reporting

# 重要な出力行:
# READ: bw=5123MiB/s (5373MB/s), 5123MiB/s-5123MiB/s (5373MB/s-5373MB/s), io=150GiB (161GB), run=30001-30001msec
```

**なぜ`bs=1M`と`iodepth=32`なのか?** シーケンシャル帯域幅は、大きなI/Oサイズ（デバイスは1MBのリクエストを1回の操作で処理できる）と、前のI/Oが完了する間もデバイスを忙しく保つのに十分なキュー深度によって最大化される。

```bash
# デバイスに最適なブロックサイズを探す
for bs in 4k 16k 64k 256k 1M 4M; do
    bw=$(fio --name=test --rw=read --bs=$bs --direct=1 \
              --iodepth=32 --numjobs=1 \
              --filename=/dev/nvme0n1 --size=4G \
              --time_based --runtime=10 --output-format=terse 2>/dev/null \
              | awk -F';' '{print $6}')
    echo "bs=$bs: ${bw}KB/s"
done
```

### `dd`: 手早いシーケンシャル計測 {#dd-quick-sequential-measurement}

```bash
# 読み出しスループット（O_DIRECT、デバイスから）
dd if=/dev/nvme0n1 of=/dev/null bs=1M count=4096 iflag=direct 2>&1 | tail -1
# 4294967296 bytes (4.3 GB, 4.0 GiB) copied, 0.843 s, 5.1 GB/s

# 書き込みスループット（O_DIRECT、デバイスへ）
dd if=/dev/zero of=/dev/nvme0n1 bs=1M count=4096 oflag=direct 2>&1 | tail -1
```

`dd`は信頼できるベンチマークではない（シングルスレッドで、キュー深度を制御できない）が、高速でどこでも使える。

### ライブのスループットを見るための`/proc/diskstats` {#procdiskstats-for-live-throughput}

```bash
# 読み書きのライブスループット（MB/s）
awk '
BEGIN { getline line1 < "/proc/diskstats"; close("/proc/diskstats") }
/nvme0n1/ { split(line1, a); split($0, b)
  printf "read: %.1f MB/s  write: %.1f MB/s\n",
    (b[6]-a[6])*512/1024/1024,
    (b[10]-a[10])*512/1024/1024
  exit }
' <(grep nvme0n1 /proc/diskstats; sleep 1; grep nvme0n1 /proc/diskstats)
```

---

## I/O帯域幅におけるCPUのボトルネック {#the-cpu-bottleneck-in-io-bandwidth}

最新のNVMeデバイス（5〜7 GB/s）では、ストレージデバイスがボトルネックでないことが多く、CPUがボトルネックになる。デバイスとメモリの間でデータを移動するには次のものが必要である:

1. **割り込み処理**: I/O完了ごとに1回の割り込み（割り込みコアレッシングを使えばバッチごと）
2. **メモリコピー**: データがユーザー空間に届く前にカーネルバッファを経由する場合
3. **DMAの管理**: IOMMUのプログラミング、pageのピン留め、完了の確認

```bash
# シーケンシャル I/O 中の CPU 使用率を確認
iostat -xz 1 &
mpstat -P ALL 1 &

dd if=/dev/nvme0n1 of=/dev/null bs=1M count=10240 iflag=direct

# 確認する点: NVMe 割り込みを処理している CPU で %irq や %soft が高いか
# 1 つの CPU が I/O 割り込みで 100% に張り付いているなら、CPU がボトルネック
```

**CPUで制限された帯域幅の症状:**

- `iostat %util`が100%未満なのに、帯域幅が約3〜4 GB/sで頭打ちになる
- 1つのCPUコアが100%になっている（`mpstat`で`%irq`または`%soft`が高い）
- スループットが並列ジョブ数に比例して増加する（CPUが飽和するまで）

**解決策:**

```bash
# 1. NVMe の IRQ を複数の CPU に分散させる
# NVMe の IRQ 番号を探す
grep nvme /proc/interrupts | awk '{print $1}' | tr -d ':'

# すべての CPU に分散するようアフィニティを設定
for irq in $(grep nvme0 /proc/interrupts | awk '{print $1}' | tr -d ':'); do
    echo ff > /proc/irq/$irq/smp_affinity  # ビットマスクで 8 CPU すべて
done

# 2. io_uring と IOPOLL を使って割り込みのオーバーヘッドをなくす
# IOPOLL: アプリケーションが割り込みを待つ代わりに完了をポーリングする
# CPU サイクルと引き換えに割り込みのレイテンシを減らす

# 3. 割り込みコアレッシングを増やす（完了をバッチ化し、割り込みを減らす）
# NVMe: 完了キューの割り込みコアレッシングを設定
nvme set-feature /dev/nvme0 -f 8 -v 0x0064  # 100 エントリまたは 1ms をまとめる
```

---

## キュー深度と並列性 {#queue-depth-and-parallelism}

シーケンシャルI/Oでは、帯域幅はキュー深度に応じて伸びる — デバイス内部の並列性の限界までである:

```bash
# 帯域幅とキュー深度の関係（1 ジョブ、iodepth を変化）
for qd in 1 2 4 8 16 32 64 128; do
    bw=$(fio --name=qd_test --rw=read --bs=128k --direct=1 \
              --numjobs=1 --iodepth=$qd \
              --filename=/dev/nvme0n1 --size=10G \
              --time_based --runtime=10 --output-format=terse 2>/dev/null \
              | awk -F';' '{print $6}')
    echo "iodepth=$qd: $((bw/1024)) MB/s"
done
```

NVMeにおける帯域幅とキュー深度の典型的な曲線:

```
QD=1:   ~1.5 GB/s  (device latency limits throughput)
QD=4:   ~3.8 GB/s
QD=8:   ~5.5 GB/s
QD=16:  ~6.8 GB/s  (approaching device max)
QD=32:  ~7.0 GB/s  (saturated — diminishing returns)
QD=64:  ~7.0 GB/s  (no improvement, higher latency)
```

関係式は`IOPS = queue_depth / latency`であり、これにブロックサイズを掛けるとスループットになる。レイテンシが100µs（0.1ms）なら、キュー深度1では10,000 IOPS = 4KBで40 MB/sとなる。キュー深度32では320,000 IOPS = 4KBで1.25 GB/sとなる。

---

## 上限としてのメモリ帯域幅 {#memory-bandwidth-as-a-ceiling}

大きなシーケンシャルI/Oでは、データはデバイスとDRAMの間を移動しなければならない。DRAMの帯域幅が絶対的な上限となる:

```bash
# stream ベンチマークで DRAM 帯域幅を計測
# https://www.cs.virginia.edu/stream/
./stream
# Triad: 85432.3 MB/s  (typical DDR5 on a desktop)

# 7 GB/s の NVMe なら: DRAM 帯域幅はボトルネックではない
# 各 7 GB/s の NVMe 8 台による RAID = 56 GB/s: DRAM の限界に近づく
```

高帯域幅のストレージシステム（NVMe-oF、大規模RAIDアレイ）を構築する場合、DRAMの帯域幅が制限要因となる。解決策には次のものがある:

- **NUMAを考慮したI/O**: I/Oバッファをデバイスと同じNUMAノードに置く
- **PMEMへの直接DMA**: 永続メモリが転送先の場合はDRAMをバイパスする
- **カーネルバイパス**: SPDK（Storage Performance Development Kit）はカーネルのオーバーヘッドを完全に回避する

---

## 帯域幅に対するファイルシステムのオーバーヘッド {#filesystem-overhead-on-bandwidth}

生のブロックデバイスは定格どおりの帯域幅を発揮する。その上にファイルシステムを載せるとオーバーヘッドが加わる:

```bash
# 生デバイスの帯域幅
fio --name=raw --rw=read --bs=1M --direct=1 --iodepth=32 \
    --filename=/dev/nvme0n1 --size=10G --runtime=10 --time_based

# 同じデバイスに ext4 ファイルシステムを作成した場合
mkfs.ext4 /dev/nvme0n1
mount /dev/nvme0n1 /mnt
fio --name=fs --rw=read --bs=1M --direct=1 --iodepth=32 \
    --filename=/mnt/testfile --size=10G --runtime=10 --time_based
```

`O_DIRECT`によるシーケンシャルI/Oでの典型的なファイルシステムのオーバーヘッド:

- **ext4、XFS**: 3〜8%のオーバーヘッド（エクステントの検索、inodeのロック）
- **Btrfs**: 5〜15%のオーバーヘッド（ツリー操作、チェックサム）
- **tmpfs**: 0%（メモリベースで、デバイスI/Oがない）

バッファードI/O（ページキャッシュ経由）では、ページキャッシュによってコピーが1回増えるが、先読みとキャッシュも有効になるため、再読み込みの多いワークロードではデバイスより高い帯域幅を実現できる。

---

## 本番環境での帯域幅の監視 {#monitoring-bandwidth-in-production}

```bash
# iostat: 最も簡単な監視
iostat -xz 1 | grep -E '(nvme|Device)'

# 帯域幅に関する重要な列:
# rkB/s: 読み出し KB/s
# wkB/s: 書き込み KB/s

# bpftrace: プロセスごとの I/O 帯域幅をライブで
bpftrace -e '
tracepoint:block:block_rq_complete {
    @bytes_by_comm[comm] = sum(args->nr_sector * 512);
}
interval:s:1 {
    print(@bytes_by_comm);
    clear(@bytes_by_comm);
}'

# 書き込み帯域幅がしきい値を超えたら警告（監視システム向け）
THRESHOLD=$((400 * 1024))  # 400 MB/s を KB/s で
while true; do
    wbw=$(awk '/nvme0n1/{print $9}' /proc/diskstats)
    sleep 1
    wbw2=$(awk '/nvme0n1/{print $9}' /proc/diskstats)
    rate=$(( (wbw2 - wbw) * 512 / 1024 ))
    if [ $rate -gt $THRESHOLD ]; then
        echo "WARN: write bandwidth ${rate}KB/s exceeds ${THRESHOLD}KB/s"
    fi
done
```

---

## クイックリファレンス: ストレージの種類ごとの帯域幅特性 {#quick-reference-bandwidth-characteristics-by-storage-type}

| ストレージ | シーケンシャル読み出し | シーケンシャル書き込み | 備考 |
|---------|----------|-----------|-------|
| SATA HDD | 80〜200 MB/s | 70〜180 MB/s | 回転型。シーケンシャル ≫ ランダム |
| SATA SSD | 400〜560 MB/s | 350〜520 MB/s | SATA IIIインターフェースの限界 |
| NVMe Gen3 ×4 | 2〜3.5 GB/s | 1.5〜3 GB/s | PCIe 3.0 ×4の限界: 3.94 GB/s |
| NVMe Gen4 ×4 | 4〜7 GB/s | 3〜6.5 GB/s | PCIe 4.0 ×4の限界: 7.88 GB/s |
| NVMe Gen5 ×4 | 8〜14 GB/s | 6〜12 GB/s | PCIe 5.0 ×4の限界: 15.75 GB/s |
| Intel Optane | 2.5〜7 GB/s | 2〜7 GB/s | 非常に低レイテンシ（10µs）。帯域幅はNVMe Gen5に劣る |
| NVMe-oF（TCP） | 最大25 Gb/sのNIC | | ネットワークで制限される |

---

## 関連ページ {#related-pages}

- [ストリーミングワークロード向けのI/Oチューニング](tuning-streaming.md) — シーケンシャルスループットの最大化
- [I/Oポーリング（HIPRIとIOPOLL）](io-polling.md) — 割り込みのオーバーヘッドの排除
- [/proc/diskstatsを理解する](understanding-proc-diskstats.md) — diskstatsからの帯域幅の計測
