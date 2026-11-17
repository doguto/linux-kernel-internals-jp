# OOMのシナリオとデバッグガイド {#oom-scenarios-and-debugging-guide}

> メモリ不足によるキルを防ぎ、検出し、診断するための実践的なプレイブック

これはカーネル内部を扱う[メモリ不足](oom.md)の実践編である。このページでは、実際に何をすべきかに焦点を当てる。

## OOMを防ぐ {#preventing-oom}

### ワークロードに合わせてメモリを見積もる {#size-memory-for-your-workload}

```bash
# ベースライン: 現在の使用量と余裕を確認する
free -h
cat /proc/meminfo | grep -E "MemTotal|MemAvailable|SwapTotal|SwapFree"

# プロセスごとの RSS（上位の消費者）
ps aux --sort=-%mem | head -20

# 全プロセスの RSS の合計（おおよそのコミット済みメモリ）
awk '/Rss:/ {sum += $2} END {print sum/1024, "MB"}' /proc/[0-9]*/smaps_rollup 2>/dev/null
```

目安: ピーク負荷時に `MemAvailable` を `MemTotal` の10〜15%以上に保つこと。日常的にそれを下回るなら、RAMを増やすかワークロードを減らす。

### スワップを追加する {#add-swap}

スワップなしで運用することは、不要なOOMキルの唯一最大の原因である。小さなスワップでも、コールドな匿名pageのための圧力弁をカーネルに与える。

```bash
# 既存のスワップを確認する
swapon --show

# スワップファイルを作成する（4GB の例）
# 注: fallocate は Btrfs や reflink 付きの XFS では機能しないことがある。
# 移植性のある代替として dd を使う: dd if=/dev/zero of=/swapfile bs=1M count=4096
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# 永続化する
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# swappiness を調整する（デフォルト 60。低いほどページキャッシュの回収を優先）
sysctl vm.swappiness=30
```

カーネルがスワップをどう使うかの詳細は[スワップ中に何が起きるか](swapping.md)を参照。

### cgroupのメモリ制限を設定する {#set-cgroup-memory-limits}

cgroupの制限は、暴走したプロセスがシステム全体に圧力をかける前に封じ込める。全体像は[メモリcgroup](memcg.md)を参照。

```bash
# systemd サービス（推奨）
# ユニットファイルの [Service] セクションで:
#   MemoryMax=2G
#   MemoryHigh=1800M

# 手動の cgroup v2
echo 2G > /sys/fs/cgroup/myservice/memory.max
echo 1800M > /sys/fs/cgroup/myservice/memory.high
```

`memory.high` はソフトなスロットリングである。カーネルはハードな `memory.max` 制限に達する前に割り当てを遅くする。常に両方を設定すること。

### ウォーターマークをチューニングする {#tune-watermarks}

```bash
# kswapd が動く余地を増やす（デフォルトはサーバーには小さすぎることが多い）
sysctl vm.min_free_kbytes=262144    # 256 MB
sysctl vm.watermark_scale_factor=50 # 1 段階あたりゾーンメモリの 0.5%
```

[メモリ不足](oom.md#stage-1-watermarks-and-pressure-detection)のウォーターマークについての議論を参照。

## 迫りくるOOMを検出する {#detecting-approaching-oom}

OOMが発動する前にメモリ逼迫を捉える方が、事後にデバッグするよりはるかに良い。

### PSI（Pressure Stall Information）を監視する {#monitor-psi-pressure-stall-information}

PSI（カーネル4.20以降で利用可能）は、メモリ逼迫について唯一最良の指標である。

```bash
# 現在の逼迫度（some = 少なくとも 1 つのタスクがストール、full = すべてのタスクがストール）
cat /proc/pressure/memory
# some avg10=0.00 avg60=0.00 avg300=0.00 total=0
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0

# cgroup ごとの逼迫度
cat /sys/fs/cgroup/myservice/memory.pressure
```

運用上の指針（これらは経験則であり、カーネルが定義した制限ではない。ワークロードに合わせて調整すること）:

| `some avg10` | 意味 |
|-------------|---------|
| < 5%  | 健全 |
| 5-20% | 中程度の逼迫、調査すべき |
| > 20% | 深刻な逼迫、持続すればOOMの可能性が高い |

**自動対応のためのPSIトリガー**（cgroup v2）:

```bash
# PSI トリガーは fd を開いたままにする必要がある — シェルの echo では機能しない。
# fd を閉じるとトリガーが破棄されるからである（psi_fop_release() が
# psi_trigger_destroy() を呼ぶ）。fd を開いたまま保持し、
# poll()/epoll() を呼ぶプログラムを使うこと。トリガーの形式の例:
#   "some 500000 2000000"  （2 秒のウィンドウ内で 500ms のストール時間）
# カーネルドキュメントを参照: Documentation/accounting/psi.rst
```

### MemAvailableを追跡する {#track-memavailable}

```bash
# ワンライナー: 10 秒ごとに MemAvailable を記録する
while true; do
  date +%H:%M:%S | tr '\n' ' '
  awk '/MemAvailable/ {print $2/1024, "MB"}' /proc/meminfo
  sleep 10
done

# MemAvailable が 5% を下回ったら警告する
awk '/MemTotal/ {total=$2} /MemAvailable/ {avail=$2}
     END {pct=avail*100/total; if (pct < 5) print "WARNING: MemAvailable at " pct "%"}' /proc/meminfo
```

### vmstatのカウンタを監視する {#watch-vmstat-counters}

```bash
# リアルタイム表示（1 秒間隔）
vmstat 1

# 主な列:
#   si/so  = スワップイン/アウト（KB/s）-- 非ゼロは活発にスワップしていることを意味する
#   free   = 空きメモリ（KB）
#   buff/cache = 回収可能なメモリ

# 回収活動の詳細なカウンタ
watch -n 1 'grep -E "allocstall|pgscan_direct|pgsteal|oom_kill" /proc/vmstat'
```

| カウンタ | 意味 |
|---------|---------------|
| `allocstall_normal` | 直接回収のイベント（割り当てたプロセスがブロックされる） |
| `pgscan_direct` | 直接回収中にスキャンされたpage |
| `pgsteal_direct` | 直接回収で回収に成功したpage |
| `oom_kill` | 起動以降のOOMキルの総数 |

`allocstall` が増加しているなら、システムはメモリ逼迫状態にあり、OOMに向かっている。

## 事後にOOMを分析する {#analyzing-oom-after-the-fact}

### OOMキルのログを読む {#reading-the-oom-kill-log}

```bash
# カーネルログから OOM イベントを探す
dmesg -T | grep -i "out of memory"
dmesg -T | grep -i "oom"

# systemd のシステムでは（再起動をまたいで保持される）
journalctl -k --grep="Out of memory"
journalctl -k --grep="oom" --since="2024-01-01"
```

典型的なOOMメッセージは次のようになる。

```
[Thu Jan  4 03:14:15 2024] myapp invoked oom-killer: gfp_mask=0x100cca(GFP_HIGHUSER_MOVABLE), order=0, oom_score_adj=0
[...]
[Thu Jan  4 03:14:15 2024] Mem-Info:
[...]
[Thu Jan  4 03:14:15 2024] Out of memory: Killed process 12345 (myapp)
                           total-vm:4194304kB, anon-rss:3145728kB, file-rss:2048kB,
                           shmem-rss:0kB, UID:1000 pgtables:8192kB oom_score_adj:0
```

確認すべき主なフィールド:

| フィールド | 見るべき点 |
|-------|-----------------|
| `gfp_mask` | OOMを引き起こした割り当ての種類。`GFP_KERNEL` = カーネルの割り当て、`GFP_HIGHUSER_MOVABLE` = ユーザー空間のpage |
| `order` | 0 = 単一page（本当のOOM）。それより高いorderは本当のOOMではなく断片化の可能性がある |
| `anon-rss` | キルされたプロセスの匿名常駐メモリ |
| `oom_score_adj` | 犠牲者は保護されていたか、標的にされていたか？ |
| `Mem-Info` セクション | キル時点のシステム全体のメモリ状態 |

!!! tip "order > 0でのキル"
    `order` が0より大きい場合、システムは実際にはメモリ不足ではないかもしれない。断片化の問題である可能性がある。Mem-Infoダンプで `MemAvailable` がまだ妥当な値だったかを確認すること。詳細は[コンパクション](compaction.md)を参照。

### 監視データと突き合わせる {#correlate-with-monitoring}

OOMログはスナップショットしか捉えない。傾向を理解するには、外部の監視データと突き合わせること。

```bash
# cgroup が制限に達したかを確認する（システム全体の OOM ではない）
cat /sys/fs/cgroup/myservice/memory.events
# oom と oom_kill のカウンタが 0 より大きいかを見る

# ピーク使用量を確認する
cat /sys/fs/cgroup/myservice/memory.peak

# スワップは利用可能だったか？
dmesg -T | grep -A 50 "invoked oom-killer" | grep -i swap
```

答えるべき問い:

1. **システム全体か、cgroupスコープか？** OOMログが特定のcgroupに言及しているか確認する。
2. **スワップは利用可能だったか？** Mem-Infoダンプで `SwapFree` が0だったなら、スワップを追加していれば助けになっただろう。
3. **メモリは着実に増えていたか（リーク）、それとも急増したか？** 時系列の監視データと突き合わせる。
4. **実際にメモリを消費していたのはどのプロセスか？** キルされたプロセスは最大のものであって、リークしているものとは限らない。

## よくある設定ミス {#common-misconfigurations}

### スワップなし {#no-swap}

スワップがないと、匿名page（ヒープ、スタック）はまったく回収できない。カーネルはページキャッシュしか回収できず、それが尽きるとすぐにOOMになる。

```bash
# 確認する
swapon --show
# 空ならスワップがない -- 追加すること（上記参照）
```

SSDでもスワップの恩恵はある。カーネルは何分も何時間も触れられていないコールドなpageを優先的にスワップする。

### OOM保護されたプロセスが多すぎる {#too-many-oom-protected-processes}

複数のプロセスに `oom_score_adj=-1000` を設定すると、OOMキラーに良い犠牲者が残らなくなる。

```bash
# OOM 保護されたプロセスをすべて探す
for pid in /proc/[0-9]*; do
  adj=$(cat "$pid/oom_score_adj" 2>/dev/null)
  if [ "$adj" = "-1000" ]; then
    comm=$(cat "$pid/comm" 2>/dev/null)
    echo "PID $(basename $pid): $comm (oom_score_adj=$adj)"
  fi
done
```

キル可能なプロセスが小さなもの（RSSが少ないもの）だけだと、OOMキラーは十分なメモリを解放できずにループするか、システムがハングするかもしれない。本当に重要な1つか2つのプロセス（例: データベース、init）だけを保護すること。

### cgroupの制限が厳しすぎる {#cgroup-limits-too-tight}

通常の使用量に合わせて余裕なしに設定したcgroupの制限は、わずかな急増でOOMキルを引き起こす。

```bash
# cgroup が制限にどれだけ近いかを確認する
echo "Current: $(cat /sys/fs/cgroup/myservice/memory.current)"
echo "Max:     $(cat /sys/fs/cgroup/myservice/memory.max)"
echo "Peak:    $(cat /sys/fs/cgroup/myservice/memory.peak)"

# スロットリングのイベント（memory.high の超過）を確認する
cat /sys/fs/cgroup/myservice/memory.events | grep high
```

`memory.max` は典型的なピーク使用量より少なくとも20〜30%上に設定すること。実際の目標値には `memory.high` を使い、キルではなくスロットリング（緩やかな減速）が起きるようにする。

### オーバーコミットの設定ミス {#overcommit-misconfiguration}

```bash
# オーバーコミットポリシーを確認する
cat /proc/sys/vm/overcommit_memory
# 0 = ヒューリスティック（デフォルト、通常は問題ない）
# 1 = 常にオーバーコミット（危険: malloc は失敗せず、OOM キルになるだけ）
# 2 = 厳格なアカウンティング（割り当てを過度に拒否することがある）

cat /proc/sys/vm/overcommit_ratio  # モード 2 で使われる（RAM に対する割合）
```

詳しい説明は[メモリのオーバーコミット](overcommit.md)を参照。

## OOM処理をテストする {#testing-oom-handling}

### stress-ng: 制御されたメモリ逼迫 {#stress-ng-controlled-memory-pressure}

```bash
# 4 つのワーカーで利用可能なメモリの 90% を割り当てる
stress-ng --vm 4 --vm-bytes 90% --vm-keep --timeout 60s

# 意図的に OOM を起こす（利用可能量より多く割り当てる）
stress-ng --vm 2 --vm-bytes 120% --vm-keep --timeout 30s

# 段階的に逼迫を高める（監視と組み合わせると有用）
stress-ng --vm 1 --vm-bytes 50% --vm-method all --vm-keep --timeout 120s
```

### cgroupによる制御されたOOM {#cgroup-based-controlled-oom}

ホストを危険にさらさずに、cgroup内でOOMをテストする。

```bash
# 小さな制限のテスト用 cgroup を作成する
mkdir -p /sys/fs/cgroup/oom-test
echo 100M > /sys/fs/cgroup/oom-test/memory.max
echo $$ > /sys/fs/cgroup/oom-test/cgroup.procs

# 制限より多く割り当てる（cgroup の OOM を引き起こす）
stress-ng --vm 1 --vm-bytes 200M --vm-keep --timeout 10s

# cgroup 内で OOM が発動したことを確認する
cat /sys/fs/cgroup/oom-test/memory.events | grep oom

# 後片付け: シェルをルート cgroup に戻す
echo $$ > /sys/fs/cgroup/cgroup.procs
rmdir /sys/fs/cgroup/oom-test
```

### oom_score_adjの操作 {#oom_score_adj-manipulation}

```bash
# 特定のプロセスを OOM の標的にする
echo 1000 > /proc/$PID/oom_score_adj

# スコアを確認する
for pid in $(pgrep -x myapp); do
  echo "PID $pid: score=$(cat /proc/$pid/oom_score) adj=$(cat /proc/$pid/oom_score_adj)"
done
```

### アプリケーションがOOMを適切に処理するか確認する {#verify-your-application-handles-oom-gracefully}

```bash
# OOM の後にプロセスが再起動したかを確認する（systemd）
systemctl status myservice
# 探すもの: "Main process exited, code=killed, status=9/KILL"

# systemd が再起動することを確認する
systemctl show myservice | grep Restart=
```

## ユーザー空間のOOMハンドラ {#userspace-oom-handlers}

### systemd-oomd {#systemd-oomd}

`systemd-oomd`（[systemd 247](https://github.com/systemd/systemd/blob/v247/NEWS)で導入され、[248](https://github.com/systemd/systemd/blob/v248/NEWS)で安定化）は、カーネルのOOMキラーが発動する前に、PSIの逼迫度とスワップ使用量に基づいてプロセスをキルする。より早く、より予測可能に動作する。

```bash
# 動作しているか確認する
systemctl status systemd-oomd

# 設定を表示する
cat /etc/systemd/oomd.conf
# [OOM]
# SwapUsedLimit=90%
# DefaultMemoryPressureDurationSec=20s

# サービスごとの上書き（ユニットファイル内）
# [Service]
# ManagedOOMSwap=kill
# ManagedOOMMemoryPressure=kill
# ManagedOOMMemoryPressureLimit=80%
```

カーネルのOOMキラーとの違い:

| 観点 | カーネルのOOM | systemd-oomd |
|--------|-----------|--------------|
| トリガー | 回収を尽くした後の割り当て失敗 | PSIのしきい値またはスワップ使用量 |
| スコープ | システム全体またはcgroupごと | cgroupごと（systemdのスライス） |
| 反応時間 | 最後の手段（遅い） | 設定可能（早い） |
| 犠牲者の選択 | `oom_score` が最大のもの | cgroup内でメモリ使用量が最大のもの |
| 設定 | `/proc/*/oom_score_adj` | ユニットファイルのディレクティブ |

### その他のユーザー空間ハンドラ {#other-userspace-handlers}

**earlyoom** -- 軽量なデーモンで、デスクトップで人気がある:

```bash
# インストールして設定する
# earlyoom は MemAvailable% と SwapFree% を監視する
# デフォルト: 両方が 10% を下回るとキルする
earlyoom -m 5 -s 5    # メモリ残り 5%、スワップ残り 5% でキルする
```

**nohang** -- earlyoomよりも設定の自由度が高い代替で、PSIトリガーとプロセスごとのカスタムアクションをサポートする。

### ユーザー空間のOOMハンドラをいつ使うか {#when-to-use-userspace-oom-handlers}

- デスクトップ/ワークステーション: earlyoomやsystemd-oomdはシステムが応答しなくなるのを防ぐ。
- コンテナホスト: スライスごとのポリシーを持つsystemd-oomdは予測可能な挙動をもたらす。
- 重要なサービスを持つサーバー: ユーザー空間のハンドラはカーネルより賢いキルの判断を下せる。

## クイックリファレンス {#quick-reference}

### 1ページの診断チェックリスト {#one-page-diagnostic-checklist}

```bash
# 1. スワップは利用可能か？
swapon --show

# 2. 現在のメモリ状態
free -h

# 3. メモリ逼迫度（カーネル 4.20 以降）
cat /proc/pressure/memory

# 4. メモリ消費の上位
ps aux --sort=-%mem | head -10

# 5. 最近の OOM イベント
dmesg -T | grep -i "out of memory"

# 6. 直接回収の活動（増加 = 問題あり）
grep -E "allocstall|oom_kill" /proc/vmstat

# 7. cgroup の制限とイベント
for cg in /sys/fs/cgroup/*/; do
  if [ -f "$cg/memory.max" ]; then
    max=$(cat "$cg/memory.max")
    [ "$max" != "max" ] && echo "$(basename $cg): current=$(cat $cg/memory.current) max=$max"
  fi
done

# 8. OOM 保護されたプロセス
grep "^-1000$" /proc/*/oom_score_adj 2>/dev/null | head -20
```

## 参考文献 {#further-reading}

- [メモリ不足](oom.md) -- カーネルのOOMの内部（ウォーターマーク、回収の段階、OOMキラーのスコアリング）
- [メモリcgroup](memcg.md) -- cgroupのメモリ制限とアカウンティング
- [メモリのオーバーコミット](overcommit.md) -- Linuxが持っている以上のメモリをどう約束するか
- [スワップ中に何が起きるか](swapping.md) -- スワップサブシステムの詳細
- [ページ回収](reclaim.md) -- OOMに頼る前にカーネルがどうメモリを回収するか
