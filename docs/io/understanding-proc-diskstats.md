# /proc/diskstatsを理解する {#understanding-procdiskstats}

> Linuxのブロックデバイス統計を読み解くための、フィールドごとのガイド

## `/proc/diskstats`から分かること {#what-procdiskstats-tells-you}

`/proc/diskstats`は、カーネルのブロック層が管理するブロックデバイスごとのI/Oカウンタを公開している。`blk-mq`を通過するすべてのI/Oリクエストがここで集計される — 読み出し、書き込み、discard、フラッシュ、キュー待ち時間、サービス時間である。

```bash
cat /proc/diskstats
#  8       0 sda 91234 1023 3456789 45678 56789 2345 9876543 234567 0 123456 280245 0 0 0 0 1234 5678
#  8       1 sda1 85432 ...
# 259       0 nvme0n1 ...
```

カウンタは起動時からの累積値である。レートを得るには、2回の読み取り値の差分を取る。

---

## フィールドリファレンス {#field-reference}

```
Field  1: major device number
Field  2: minor device number
Field  3: device name
Field  4: reads completed successfully
Field  5: reads merged
Field  6: sectors read
Field  7: time spent reading (ms)
Field  8: writes completed
Field  9: writes merged
Field 10: sectors written
Field 11: time spent writing (ms)
Field 12: I/Os currently in progress
Field 13: time spent doing I/Os (ms)
Field 14: weighted time spent doing I/Os (ms)
Field 15: discards completed (kernel 4.18+)
Field 16: discards merged (kernel 4.18+)
Field 17: sectors discarded (kernel 4.18+)
Field 18: time spent discarding (ms) (kernel 4.18+)
Field 19: flush requests completed (kernel 5.5+)
Field 20: time spent flushing (ms) (kernel 5.5+)
```

---

## フィールド4〜7: 読み出し {#fields-47-reads}

### フィールド4: `reads_completed` {#field-4-reads_completed}

正常に完了した読み出しI/Oの数。デバイスから（上位のキャッシュからではなく）満たされたI/Oが含まれる。発行前に既存のキュー内リクエストにマージされた読み出しは**含まない**（それらはフィールド5で数えられる）。

```bash
# レート: 1 秒あたりの読み出し数
prev=$(awk '/^sda /{print $4}' /proc/diskstats); sleep 1
curr=$(awk '/^sda /{print $4}' /proc/diskstats)
echo "$((curr - prev)) reads/sec"
```

### フィールド5: `reads_merged` {#field-5-reads_merged}

デバイスに発行される前に、隣接する実行中のリクエストとマージされた読み出しリクエストの数。隣接するセクタへの2つの読み出しは1つの大きな読み出しにマージでき、デバイスレベルでのIOPSが減る。

マージ率が高いことは、I/Oの局所性が良く、I/Oスケジューラが効率的であることを示す。マージ率が低い（0に近い）のは、`none`スケジューラを使うNVMeや、純粋にランダムなワークロードでは正常である。

```
reads_merged / (reads_completed + reads_merged) = merge ratio
```

### フィールド6: `sectors_read` {#field-6-sectors_read}

読み出したセクタ（各512バイト）の総数。メガバイトにするには2048で割るか、次のようにする:

```bash
# 読み出しスループット（MB/s）
awk 'BEGIN{prev=0} /^sda /{
    if (prev > 0) printf "%.1f MB/s read\n", ($6 - prev) * 512 / 1024 / 1024;
    prev = $6
}' <(cat /proc/diskstats; sleep 1; cat /proc/diskstats)
```

### フィールド7: `ms_reading` {#field-7-ms_reading}

読み出しI/Oに費やされた合計ミリ秒。各読み出しリクエストがデバイスに発行されてから完了するまでの時間の合計である。

```
average read latency = ms_reading / reads_completed   (in ms per read)
```

`ms_reading / reads_completed`がデバイスの定格レイテンシよりはるかに高い場合（たとえば0.1msのはずのNVMeで10ms）、I/Oがデバイスに届く前にスケジューラのキューで時間を費やしているか、デバイスが激しい競合下にある。

---

## フィールド8〜11: 書き込み {#fields-811-writes}

書き込みのフィールドは読み出しのフィールドと対になっている:

- **フィールド8**（`writes_completed`）: 正常に完了した書き込み
- **フィールド9**（`writes_merged`）: 発行前にマージされた書き込み
- **フィールド10**（`sectors_written`）: 書き込んだセクタ数（× 512バイト）
- **フィールド11**（`ms_writing`）: 書き込みI/Oに費やされた合計ms

```bash
# 平均書き込みレイテンシ
awk '/^nvme0n1 /{
    if ($8 > 0) printf "avg write latency: %.2f ms\n", $11 / $8
}' /proc/diskstats
```

**書き込みレイテンシと読み出しレイテンシ**: HDDやSSDでは、ライトキャッシュのため、デバイスレベルでは書き込みの方が読み出しより速いことが多い。しかし`fsync()`を伴う書き込みは、デバイスのキャッシュをフラッシュしなければならないため遅くなる — `ms_writing`カウンタにはフラッシュの待ち時間が含まれる。

---

## フィールド12: `ios_in_progress` {#field-12-ios_in_progress}

現在デバイスに対して実行中のI/Oの数 — ドライバに発行されたがまだ完了していないもの。これは**スナップショット**（累積値ではない）であり、瞬間的なキュー深度を反映する。

```bash
# キュー深度をリアルタイムで観察
watch -n 0.5 "awk '/^nvme0n1 /{print \"in-flight:\", \$12}' /proc/diskstats"
```

**`ios_in_progress`の解釈:**

- 常に0: デバイスがアイドルであるか、I/Oが非常にバースト的である（バーストの合間にサンプリングしている）。
- 1〜8: 軽い負荷で、デバイスは飽和していない。
- `nr_requests`（NVMeのデフォルトは1023）に近い: デバイスは飽和しており、新しいI/Oはソフトウェア層でキューに入る。
- `iostat`の`%util`が高いのに0: `iostat`は`%util`にフィールド12ではなくフィールド13（`ms_doing_io`）を使う。デバイスがサンプリングできるより速くI/Oを処理している可能性がある。

---

## フィールド13: `ms_doing_io` {#field-13-ms_doing_io}

少なくとも1つのI/Oが実行中だった合計ミリ秒。これは`iostat`の`%util`メトリクスの元データである:

```
%util = (delta ms_doing_io) / (measurement_interval_ms) × 100
```

`%util`が100%に達するということは、計測期間全体にわたってデバイスに少なくとも1つのI/Oが実行中だった — デバイスが一度もアイドルにならなかった — ことを意味する。HDDでは、これは飽和を示す。NVMeでは、100%の`%util`は正常であり、飽和を示さない（NVMeは利用率100%でも多数の並列I/Oを処理できる）。

```bash
# %util を手動で計算
prev=$(awk '/^sda /{print $13}' /proc/diskstats)
sleep 1
curr=$(awk '/^sda /{print $13}' /proc/diskstats)
echo "util: $(( (curr - prev) ))%"   # 間隔がちょうど 1000ms の場合
```

---

## フィールド14: `ms_weighted_io`（I/Oの重み付き時間） {#field-14-ms_weighted_io-weighted-time-in-ios}

このフィールドは、BIOがキューに入れられるか完了するたびに`(current_ios_in_progress × elapsed_ms)`だけ増加する。キュー深度で重み付けされた時間 — キュー深度の時間積分 — である。

```
ms_weighted_io / ms_doing_io ≈ average queue depth during active periods
```

これは`iostat`が`aqu-sz`（平均キューサイズ）を計算するのに使われる:

```
aqu-sz = delta(ms_weighted_io) / measurement_interval_ms
```

`aqu-sz`が高く`await`が低いのは問題ない（NVMeが大きな並列キューを効率よく処理している）。`aqu-sz`が高く`await`も高い場合は、デバイスがキューに追いつけていない — 真の飽和 — ことを示す。

---

## フィールド15〜18: discard（カーネル4.18以降） {#fields-1518-discards-kernel-418}

discard（TRIM/UNMAP）はデバイスに解放された領域を通知し、SSDが未使用ブロックを事前にガベージコレクションできるようにする。discardのフィールドは読み出し/書き込みの構造と対になっている:

- **フィールド15**（`discards_completed`）: 完了したdiscard操作
- **フィールド16**（`discards_merged`）: マージされたdiscard（実際にはまれ）
- **フィールド17**（`sectors_discarded`）: discardされたセクタ数
- **フィールド18**（`ms_discarding`）: discardの処理に費やされたms

```bash
# discard は動いているか?（データベースやコンテナは定期的な TRIM の恩恵を受ける）
awk '/^nvme0n1 /{print "discards:", $15, "sectors:", $17}' /proc/diskstats

# fstrim で定期的な TRIM を有効にする（cron から毎週実行）
fstrim -v /  # ルートファイルシステムの未使用ブロックを discard
```

discardのレートが高いことは、領域の回収が活発であることを示す — 大きなファイルを削除・再作成するシステム（コンテナ、ログをローテートするデータベース）では想定どおりである。discardのレートが非常に高いと、デバイスのガベージコレクタが追いつかなくなった場合にSSDの書き込み増幅を引き起こすことがある。

---

## フィールド19〜20: フラッシュリクエスト（カーネル5.5以降） {#fields-1920-flush-requests-kernel-55}

フラッシュリクエストはデバイスにキャッシュフラッシュコマンド（`REQ_PREFLUSH`または`REQ_FUA`）を送り、ライトキャッシュ上のデータを永続メディアにコミットするよう指示する。これらは、ジャーナリングファイルシステムがジャーナルのコミット前に生成するほか、明示的な`fsync()`呼び出しによっても生成される。

- **フィールド19**（`flush_requests`）: 完了したフラッシュコマンド
- **フィールド20**（`ms_flushing`）: フラッシュ操作に費やされた合計ms

```bash
# フラッシュレート: デバイスがどれくらいの頻度でフラッシュされているか
prev_f=$(awk '/^sda /{print $19}' /proc/diskstats); sleep 10
curr_f=$(awk '/^sda /{print $19}' /proc/diskstats)
echo "$(( (curr_f - prev_f) / 10 )) flushes/sec"

# 平均フラッシュレイテンシ（フラッシュあたりの ms）
awk '/^sda /{if ($19 > 0) printf "avg flush latency: %.1f ms\n", $20/$19}' /proc/diskstats
```

**なぜフラッシュレイテンシがデータベースにとって重要か:**

ダーティデータを持つファイルに対する`fsync()`や`fdatasync()`は、それぞれ少なくとも1回のフラッシュコマンドを引き起こす。フラッシュレイテンシは最大トランザクションレートの上限を直接決める:

```
max_tps ≤ 1000 / avg_flush_latency_ms

NVMe (0.1ms flush): max ~10,000 TPS from flush alone
SATA SSD (1ms flush): max ~1,000 TPS
HDD (5ms flush): max ~200 TPS
```

`ms_flushing / flush_requests`を監視すれば、使用しているストレージでの実際のフラッシュレイテンシが分かる。

---

## 実践的な分析レシピ {#practical-analysis-recipes}

### 飽和したデバイスを検出する {#detect-a-saturated-device}

```bash
# diskstats を 2 回サンプリングし、派生メトリクスを計算
{
  read line1 < <(grep '^sda ' /proc/diskstats)
  sleep 5
  read line2 < <(grep '^sda ' /proc/diskstats)

  set -- $line1; r1=$4; w1=$8; ms_io1=$13; ms_w1=$14
  set -- $line2; r2=$4; w2=$8; ms_io2=$13; ms_w2=$14

  interval=5000  # ms
  echo "reads/sec:    $(( (r2-r1)/5 ))"
  echo "writes/sec:   $(( (w2-w1)/5 ))"
  echo "%util:        $(( (ms_io2-ms_io1)*100/interval ))%"
  echo "avg queue:    $(echo "scale=1; (${ms_w2}-${ms_w1})/${interval}" | bc)"
}
```

### 書き込みレイテンシが最も高いデバイスを探す {#find-the-device-with-the-highest-write-latency}

```bash
awk '
NF >= 14 && $8 > 100 {
    lat = $11 / $8
    if (lat > max_lat) { max_lat = lat; max_dev = $3 }
}
END { printf "slowest writer: %s at %.1f ms avg\n", max_dev, max_lat }
' /proc/diskstats
```

### データベースのチェックポイント中のフラッシュ活動を観察する {#watch-flush-activity-during-a-database-checkpoint}

```bash
# チェックポイント前
awk '/nvme0n1/{print $19, $20}' /proc/diskstats

# チェックポイントを実行（PostgreSQL）:
# psql -c "CHECKPOINT"

# チェックポイント後
awk '/nvme0n1/{print $19, $20}' /proc/diskstats
# 差分: チェックポイントが生成したフラッシュ数と、費やされた合計 ms
```

---

## `iostat`の出力との関係 {#relationship-to-iostat-output}

| `iostat -x`の列 | `/proc/diskstats`からの導出 |
|--------------------|-----------------------------|
| `r/s` | `delta(reads_completed) / interval` |
| `w/s` | `delta(writes_completed) / interval` |
| `rkB/s` | `delta(sectors_read) × 512 / 1024 / interval` |
| `wkB/s` | `delta(sectors_written) × 512 / 1024 / interval` |
| `rrqm/s` | `delta(reads_merged) / interval` |
| `wrqm/s` | `delta(writes_merged) / interval` |
| `r_await` | `delta(ms_reading) / delta(reads_completed)` |
| `w_await` | `delta(ms_writing) / delta(writes_completed)` |
| `aqu-sz` | `delta(ms_weighted_io) / (interval × 1000)` |
| `%util` | `delta(ms_doing_io) / (interval × 10)` |

`iostat`は監視のためのより便利なインターフェースであり、`/proc/diskstats`はプログラムからのアクセス、アラート、独自のメトリクスパイプラインのための信頼できる情報源である。

---

## 関連ページ {#related-pages}

- [ストレージI/Oのチューニング](tuning-storage.md) — diskstatsが明らかにしたことへの対処
