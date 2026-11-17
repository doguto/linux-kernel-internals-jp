# I/Oハングのデバッグ {#debugging-io-hangs}

> I/Oが単に遅いのではなく完全に止まってしまったとき — LinuxにおけるI/Oハングの特定、切り分け、復旧の方法

## I/Oハングとは何か {#what-is-an-io-hang}

**D状態**（割り込み不可能なスリープ）にあるプロセスは、I/Oの完了を待っている。通常この状態はマイクロ秒からミリ秒程度しか続かない。I/Oハングは、プロセスが数秒、数分、あるいは無期限にD状態にとどまるときに発生する。典型的な原因は次のとおりである:

- ストレージデバイスが応答しなくなった（ハードウェア故障、ファームウェアのハング、ケーブルの問題）
- ネットワークファイルシステム（NFS、CIFS）がサーバーとの接続を失った
- ブロック層がエラーを検出し、無制限にリトライしている
- D状態のプロセスがカーネルのロックを保持しており、依存するプロセスの連鎖を引き起こしている

カーネルはデフォルトで120秒後に「hung task」警告をログに出力する:

```
[12345.678901] INFO: task kworker/u8:2:1234 blocked for more than 120 seconds.
[12345.678902]       Not tainted 6.8.0 #1
[12345.678903] "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
[12345.678904] task:kworker/u8:2    state:D stack:    0 pid: 1234 ppid:     2 flags:0x00004000
[12345.678905] Call Trace:
[12345.678906]  <TASK>
[12345.678907]  __schedule+0x2d4/0x8a0
[12345.678908]  schedule+0x46/0xb0
[12345.678909]  io_schedule+0x42/0x70
[12345.678910]  wait_for_completion_io+0x6e/0x110
```

---

## ステップ1: どのプロセスがハングしているかを特定する {#step-1-identify-which-processes-are-hung}

```bash
# D 状態（割り込み不可能なスリープ）のプロセスをすべて探す
ps aux | awk '$8 ~ /^D/ { print $0 }'

# あるいはより詳細に
for pid in $(ls /proc | grep '^[0-9]'); do
    state=$(cat /proc/$pid/status 2>/dev/null | grep '^State' | awk '{print $2}')
    if [ "$state" = "D" ]; then
        comm=$(cat /proc/$pid/comm 2>/dev/null)
        wchan=$(cat /proc/$pid/wchan 2>/dev/null)
        echo "PID $pid ($comm): blocked in $wchan"
    fi
done
```

**`/proc/<pid>/wchan`の読み方:**

| `wchan`の値 | プロセスが待っているもの |
|---------------|--------------------------------|
| `io_schedule` | 一般的なブロックI/O |
| `jbd2_log_wait_commit` | ext4のジャーナルコミット |
| `nfs_wait_bit_killable` | NFS操作 |
| `wait_on_page_writeback` | 特定のpageがフラッシュされるのを待っている |
| `balance_dirty_pages_ratelimited` | ダーティスロットリング（通常はすぐに解消する） |
| `blk_execute_rq` | 同期ブロックリクエストの処理中 |
| `md_flush_request` | md/RAIDのフラッシュ |

---

## ステップ2: カーネルスタック全体を取得する {#step-2-get-the-full-kernel-stack}

```bash
# CAP_SYS_ADMIN（root）が必要
cat /proc/<pid>/stack

# ext4 の書き込みが止まっている場合の出力例:
# [<0>] jbd2_log_wait_commit+0xb4/0x110
# [<0>] jbd2__journal_start+0x18c/0x340
# [<0>] __ext4_journal_start_sb+0x6c/0xf0
# [<0>] ext4_dirty_inode+0x34/0x60
# [<0>] __mark_inode_dirty+0x1cc/0x4b0
# [<0>] generic_write_end+0xb4/0x100
# [<0>] ext4_write_end+0x68/0x1c0
# [<0>] generic_perform_write+0x124/0x1c0
# [<0>] ext4_buffered_write_iter+0x58/0x100
# [<0>] vfs_write+0x298/0x3e0
# [<0>] __x64_sys_write+0x5c/0x100
```

スタックトレースは、プロセスがカーネル内のどこでブロックしているかを正確に示す。最も内側のフレームから上へたどっていく。

**すべてのD状態プロセスのスタックを一度に取得する（インシデント対応中に有用）:**

```bash
# D 状態のスタックをすべてダンプするスクリプト
for pid in $(ls /proc | grep '^[0-9]'); do
    state=$(awk '/^State/{print $2}' /proc/$pid/status 2>/dev/null)
    if [ "$state" = "D" ]; then
        echo "=== PID $pid ($(cat /proc/$pid/comm 2>/dev/null)) ==="
        cat /proc/$pid/stack 2>/dev/null
        echo ""
    fi
done

# あるいは sysrq を使ってすべてのスタックを dmesg にダンプする
echo l > /proc/sysrq-trigger  # 全 CPU のスタックをダンプ
# その後: dmesg | tail -200
```

---

## ステップ3: デバイスの健全性を確認する {#step-3-check-device-health}

プロセスがブロックI/Oで止まっている場合は、デバイス自体が応答しなくなっていないかを確認する。

```bash
# カーネルログでデバイスエラーを確認
dmesg | grep -E '(error|timeout|reset|offline|failed|EIO|SCSI|ata|nvme)' | tail -30

# NVMe の健全性
nvme smart-log /dev/nvme0

# SATA/SAS の健全性
smartctl -a /dev/sda

# デバイスがまだ I/O に応答しているか確認（デバイスがハングしているとブロックする）
# シェルがハングしないよう timeout を使う
timeout 5 dd if=/dev/sda of=/dev/null bs=512 count=1 iflag=direct 2>&1
# これがハングする場合: デバイスは I/O に応答していない

# ブロックデバイスのキュー状態を確認
cat /sys/block/sda/queue/in_flight    # 現在デバイスに対して実行中の I/O
cat /sys/block/sda/queue/nr_requests  # 最大キュー深度
```

**dmesgに現れる典型的なデバイスハングのシグネチャ:**

```
# NVMe timeout:
nvme nvme0: I/O 23 QID 1 timeout, aborting

# SCSI timeout:
sd 0:0:0:0: [sda] tag#0 FAILED Result: hostbyte=DID_TIMEOUT driverbyte=DRIVER_OK

# ATA timeout:
ata1.00: exception Emask 0x10 SAct 0x0 SErr 0x4010000 action 0x6
ata1.00: hard resetting link

# I/O error returned to filesystem:
EXT4-fs error (device sda1): ext4_find_entry:1455: inode #2: comm bash: reading directory lblock 0
```

---

## ステップ4: NFSのハングを確認する {#step-4-check-for-nfs-hangs}

NFSのハングは、D状態プロセスのよくある原因である。NFSサーバーに到達できなくなったとき、あるいはクライアントの接続がタイムアウトしたときに発生する。

```bash
# NFS のマウントオプションを確認
mount | grep nfs
# 確認する項目: hard か soft か、timeo=、retrans=、intr

# NFS の統計でエラーを確認
nfsstat -c  # クライアント側の統計
cat /proc/net/rpc/nfs  # NFS クライアント RPC の生の統計

# 保留中の RPC 呼び出しを確認
cat /proc/net/rpc/nfsd  # このマシンがサーバーの場合のサーバー側
```

**hardマウントとsoftマウント:**

`hard`（デフォルト）でマウントすると、サーバー障害時に無期限にリトライする — サーバーが復帰するまでプロセスはD状態でハングする。`soft`マウントは、`retrans`回のリトライ × `timeo`のタイムアウトの後に`EIO`を返す。

```bash
# 無限のハングを避けるため soft と timeo で再マウント
mount -o remount,soft,timeo=30,retrans=3 /nfs/mountpoint

# あるいは止まった NFS マウントを強制アンマウント（遅延アンマウント）
umount -l /nfs/mountpoint   # lazy: 即座に切り離し、参照がなくなったら後始末
umount -f /nfs/mountpoint   # force: 使用中のファイルがあっても即時アンマウントを試みる
```

!!! warning "強制アンマウントによるデータ損失のリスク"
    ダーティデータを持つNFSファイルシステムを強制アンマウントすると、まだサーバーに届いていない書き込みが失われる可能性がある。`-f`は、データ損失の可能性を受け入れてでもシステムを復旧する必要があるときにのみ使うこと。

---

## ステップ5: ライトバックのハングを確認する {#step-5-check-for-writeback-hangs}

ライトバックのハングは、ダーティpageをフラッシュできないときに発生する。フラッシャーのkworkerが止まっているか、デバイスがエラーを返してリトライが起きている状態である。

```bash
# ライトバックの状態を確認
grep -E '(nr_dirty|nr_writeback|nr_dirty_threshold|nr_dirty_background_threshold)' /proc/vmstat

# kworker スレッドが止まっていないか確認
ps aux | grep kworker
# ライトバック中に D 状態になった kworker は、スタックに次が現れる:
# writeback_sb_inodes / wb_writeback / wb_do_writeback

# デバイスごとの BDI（Backing Device Info）の状態を確認
ls /sys/class/bdi/
cat /sys/class/bdi/8:0/max_ratio          # このデバイスの最大ダーティ比率
cat /sys/class/bdi/8:0/read_ahead_kb

# デバイスにエラーがあると、ライトバックはリトライする:
dmesg | grep -E '(writeback|EIO|write error)' | tail -20
```

**ライトバックの停滞を強制的に解消する:**

```bash
# クリーンなキャッシュ page をすべて破棄（ライトバックがエラーで止まっている場合は役に立たない）
echo 1 > /proc/sys/vm/drop_caches

# すべてのファイルシステムの sync を試みる（ライトバックがハングしているとブロックする）
timeout 30 sync

# sync がハングする場合、blktrace でどのデバイスが問題を起こしているかを確認
blktrace -d /dev/sda -o /tmp/btrace &
sleep 5
kill %1
blkparse -i /tmp/btrace.blktrace.0 | tail -50
```

---

## ステップ6: ハングの原因となるロック競合を確認する {#step-6-check-for-lock-contention-causing-hangs}

プロセスがI/Oを行っているように見えても、実際にはD状態のI/O待ちプロセスが保持しているカーネルのロックを待っていることがある。これにより、止まったプロセスの連鎖が生じる。

```bash
# lockdep の出力（カーネル設定で有効な場合）
cat /proc/lockdep_stats

# プロセスのスタックで mutex/rwsem の競合を確認
# 次のようなフレームを探す:
#   mutex_lock_slowpath
#   rwsem_down_read_slowpath
#   down_read
```

**`perf lock`でロック競合を見つける:**

```bash
# ロック競合イベントを 10 秒間記録
perf lock record -a -- sleep 10

# 分析: ロックごとの競合を表示
perf lock report --key=wait

# あるいは bpftrace で mutex の競合をトレース
bpftrace -e '
kprobe:mutex_lock_slowpath {
    @[kstack] = count();
}
interval:s:5 {
    print(@);
    clear(@);
}'
```

---

## ステップ7: 再起動せずにデバイスのハングから復旧する {#step-7-recover-from-a-device-hang-without-rebooting}

単一のデバイスがハングしているだけで、システムのそれ以外の部分は機能している場合、再起動せずにデバイスをリセットできることがある。

**SCSI/SATAデバイスのリセット:**

```bash
# SCSI デバイスのリセットを起こす
echo 1 > /sys/block/sda/device/delete   # デバイスをシステムから取り除く
# その後、再スキャンして再追加:
echo "- - -" > /sys/class/scsi_host/host0/scan

# あるいは sg_reset でバスリセットを送る
sg_reset --device /dev/sda
sg_reset --bus /dev/sda
```

**NVMeのリセット:**

```bash
# NVMe コントローラをリセット
nvme reset /dev/nvme0

# あるいは sysfs 経由で取り除いて再追加
echo 1 > /sys/bus/pci/devices/<pci-id>/remove
echo 1 > /sys/bus/pci/rescan
```

**md/RAIDの復旧:**

```bash
# RAID の状態を確認
cat /proc/mdstat

# 故障したドライブを faulty としてマークし、取り除く
mdadm /dev/md0 --fail /dev/sdb
mdadm /dev/md0 --remove /dev/sdb

# 交換後に再追加
mdadm /dev/md0 --add /dev/sdb
```

---

## 予防的な設定 {#preventive-configuration}

```bash
# より早く警告を得るため hung_task のタイムアウトを短くする
echo 30 > /proc/sys/kernel/hung_task_timeout_secs

# hung task でパニックさせる（ハングが再起動より悪いサーバー向け）
echo 1 > /proc/sys/kernel/hung_task_panic

# ハードロックアップを検出するため NMI ウォッチドッグを有効にする
echo 1 > /proc/sys/kernel/nmi_watchdog

# NFS マウントを適度なタイムアウトの soft に設定
# /etc/fstab に追加: nfsserver:/export /mnt nfs soft,timeo=30,retrans=3 0 0

# ブロックデバイスのエラー処理を設定
# `max_sectors_kb` とエラーポリシーはデバイス固有
cat /sys/block/sda/device/timeout    # SCSI タイムアウトまでの秒数
echo 30 > /sys/block/sda/device/timeout  # 必要ならデフォルトの 30 秒から短くする
```

