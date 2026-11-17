# システムサスペンドとハイバネート {#system-suspend-and-hibernate}

> システム全体の電源状態: suspend-to-RAM、ハイバネート、そしてPMの状態機械

## スリープ状態 {#sleep-states}

Linuxは、ACPIのS-stateとカーネル内部のモデルで記述される、複数のシステムレベルのスリープ状態をサポートする。

| カーネルの状態 | ACPI | 説明 |
|---|---|---|
| `freeze` | S0（アイドル） | ユーザープロセスをフリーズし、デバイスをサスペンドする。CPUには電源が供給されたまま。レジュームは非常に速い。 |
| `standby` | S1 | 浅いスリープ。CPUの電源は落ちるが状態は保持される。 |
| `mem` | S3 | **Suspend-to-RAM（STR）**。すべての状態はDRAMに置かれ、それ以外の電源は切られる。ノートパソコンの一般的なスリープ。 |
| `disk` | S4 | **ハイバネート（STD）**。メモリイメージをスワップ/ファイルに書き出し、電源を完全に切る。 |

```bash
# 利用可能な状態を確認する
cat /sys/power/state
# freeze mem disk

# suspend-to-RAMに入る
echo mem | sudo tee /sys/power/state
```

## サスペンドの状態機械 {#the-suspend-state-machine}

```
/sys/power/state write "mem"
        │
        ▼
kernel/power/suspend.c: pm_suspend()
        │
        ├─ 1. Sync filesystems
        │
        ├─ 2. Freeze userspace processes
        │      freeze_processes() → freeze_task() per task (kernel/freezer.c):
        │      tasks blocked in an interruptible sleep are switched directly
        │      to the TASK_FROZEN scheduler state via __freeze_task(); tasks
        │      that can't be switched in place are sent a fake signal wakeup
        │      (or woken, for kthreads) so they run until they hit a
        │      try_to_freeze() checkpoint and enter the refrigerator
        │      (__refrigerator()), which also lands them in TASK_FROZEN
        │      [replaces the older pre-6.1 TIF_SIGPENDING-only refrigerator loop]
        │
        ├─ 3. Suspend devices (reverse probe order)
        │      dpm_suspend_start() → each driver's .suspend callback
        │      (deepest devices in tree suspended first)
        │
        ├─ 4. Disable non-boot CPUs (CPU hotplug)
        │
        ├─ 5. Suspend syscore (timers, IRQ chips, clocksource)
        │      syscore_suspend() — last things before hardware off
        │
        ├─ 6. Enter hardware sleep
        │      acpi_suspend_enter() or platform-specific
        │      ──────────────── ASLEEP ────────────────
        │
        └─ 7. Resume (reverse order):
               syscore_resume()
               Enable CPUs
               dpm_resume_end() → each driver's .resume callback
               Thaw processes
```

## PM notifier {#pm-notifiers}

notifierを使うと、サブシステムがサスペンド/レジュームのイベントに反応できる。

```c
#include <linux/suspend.h>

static int mysubsys_pm_notify(struct notifier_block *nb,
                               unsigned long action, void *data)
{
    switch (action) {
    case PM_SUSPEND_PREPARE:
        /* サスペンド直前: キャッシュをフラッシュし、バックグラウンド処理を止める */
        flush_workqueue(mysubsys_wq);
        break;
    case PM_POST_SUSPEND:
        /* レジュームした: バックグラウンド処理を再開する */
        queue_delayed_work(mysubsys_wq, &mysubsys_work, HZ);
        break;
    case PM_HIBERNATION_PREPARE:
        break;
    case PM_POST_HIBERNATION:
        break;
    case PM_RESTORE_PREPARE:   /* ハイバネートイメージを読み込む前 */
        break;
    case PM_POST_RESTORE:
        break;
    }
    return NOTIFY_OK;
}

static struct notifier_block mysubsys_pm_nb = {
    .notifier_call = mysubsys_pm_notify,
};

/* 登録 */
register_pm_notifier(&mysubsys_pm_nb);

/* 登録解除 */
unregister_pm_notifier(&mysubsys_pm_nb);
```

## デバイスのサスペンドコールバック {#device-suspend-callbacks}

システムサスペンドにおけるドライバのPMコールバックの完全な順序は次のとおりである。

```
dpm_prepare()      → driver .prepare()         (optional; prepare for suspend)
dpm_suspend()      → driver .suspend()          (save state, stop DMA)
dpm_suspend_late() → driver .suspend_late()     (last operations before power off)
dpm_suspend_noirq()→ driver .suspend_noirq()    (IRQs disabled; final operations)
─────────────── hardware suspended ───────────────
dpm_resume_noirq() → driver .resume_noirq()     (IRQs still disabled)
dpm_resume_early() → driver .resume_early()     (restore critical state)
dpm_resume()       → driver .resume()           (full restore)
dpm_complete()     → driver .complete()         (post-resume cleanup)
```

ほとんどのドライバでは、`.suspend`と`.resume`だけを実装すればよい。`_noirq`と`_late/_early`の変種は、非常に遅い/早いタイミングでのアクセスを必要とするハードウェア向けである。

### 最小限のサスペンド実装 {#minimal-suspend-implementation}

```c
static int mydriver_suspend(struct device *dev)
{
    struct mydata *priv = dev_get_drvdata(dev);

    /* ハードウェアの動作を停止する */
    mydriver_stop_hw(priv);

    /* ハードウェアが保持しないレジスタを保存する */
    priv->saved_config = readl(priv->base + CONFIG_REG);
    priv->saved_irq_mask = readl(priv->base + IRQ_MASK_REG);

    /* クロック/電源を無効にする */
    clk_disable_unprepare(priv->clk);

    return 0;
}

static int mydriver_resume(struct device *dev)
{
    struct mydata *priv = dev_get_drvdata(dev);
    int ret;

    /* クロック/電源を再び有効にする */
    ret = clk_prepare_enable(priv->clk);
    if (ret)
        return ret;

    /* 保存したレジスタを復元する */
    writel(priv->saved_config,   priv->base + CONFIG_REG);
    writel(priv->saved_irq_mask, priv->base + IRQ_MASK_REG);

    /* ハードウェアを再初期化する */
    mydriver_init_hw(priv);

    return 0;
}

/* SET_マクロを使った共有のPM ops */
static const struct dev_pm_ops mydriver_pm_ops = {
    SET_SYSTEM_SLEEP_PM_OPS(mydriver_suspend, mydriver_resume)
    SET_RUNTIME_PM_OPS(mydriver_runtime_suspend,
                       mydriver_runtime_resume, NULL)
};
```

## ウェイクアップソース {#wakeup-sources}

**ウェイクアップソース**とは、システムをスリープから起こすことができるハードウェアイベントである。よくあるウェイクアップソースは、キーボード、タッチパッド、NIC（Wake-on-LAN）、RTCアラーム、USBである。

```c
/* ドライバ: ウェイクアップソースを登録する */
static int mydriver_probe(struct platform_device *pdev)
{
    struct mydata *priv;
    /* ... */

    device_init_wakeup(&pdev->dev, true);  /* このデバイスはシステムを起こせる */

    /* サスペンド中にウェイクアップ割り込みをアサートするようハードウェアを設定する */
    /* ... */
    return 0;
}

/* サスペンド前: ウェイクアップ割り込みを準備する */
static int mydriver_suspend(struct device *dev)
{
    if (device_may_wakeup(dev))
        enable_irq_wake(priv->irq);  /* IRQをウェイクアップソースとして配線する */
    /* ... */
}

static int mydriver_resume(struct device *dev)
{
    if (device_may_wakeup(dev))
        disable_irq_wake(priv->irq);
    /* ... */
}
```

### ウェイクアップソースの追跡 {#wakeup-source-tracking}

```c
#include <linux/pm_wakeup.h>

/* カーネルはアクティブなウェイクアップソースを追跡する */
struct wakeup_source {
    const char           *name;
    int                    id;
    struct list_head      entry;
    spinlock_t            lock;
    struct wake_irq      *wakeirq;
    struct timer_list     timer;
    unsigned long         timer_expires;
    ktime_t               total_time;
    ktime_t               max_time;
    ktime_t               last_time;
    ktime_t               start_prevent_time;
    ktime_t               prevent_sleep_time;
    unsigned long         event_count;
    unsigned long         active_count;
    unsigned long         relax_count;
    unsigned long         expire_count;
    unsigned long         wakeup_count;
    struct device        *dev;
    bool                   active:1;
    bool                   autosleep_enabled:1;
};
```

```bash
# どのウェイクアップソースがアクティブか（サスペンドを妨げているか）?
cat /sys/kernel/debug/wakeup_sources
# name          active_count  event_count  wakeup_count  expire_count  active_since  total_time  max_time  last_change  prevent_suspend_time
# NETLINK       0             1234         0             0             0             0           0         12345        0
# battery       1             5678         1             0             12345         67890       12345     67890        12345

# どのIRQがシステムを起こせるか?
cat /proc/interrupts | grep -i wake

# 最後のウェイクアップ理由（レジューム後）: システムを起こしたIRQ番号。
# 記録がなければ0ではなく"No data available"（-ENODATA）と表示される -
# pm_wakeup_irq_show()（kernel/power/main.c）はIRQ 0を「なし」として扱う。
cat /sys/power/pm_wakeup_irq
```

## ハイバネート（suspend-to-disk） {#hibernate-suspend-to-disk}

ハイバネーションは、メモリイメージ全体をディスクに保存してから電源を切る。レジューム時にはイメージが読み戻され、中断した場所から正確に実行が再開される。

```
echo disk | sudo tee /sys/power/state
```

### ハイバネートの状態機械 {#hibernate-state-machine}

```
1. Freeze processes
2. Create memory snapshot (swsusp_save):
   - Walk all pages, copy to free pages or swap
   - The "hibernation image" = all in-use memory pages
3. Write image to swap partition or hibernation file
4. Power off
─────────── powered off ──────────
5. Boot (normal cold boot; bootloader has no special role)
6. Kernel: check resume= parameter or /sys/power/resume device for hibernation image
7. swsusp_read(): read the image back from swap
8. hibernation_restore(): overwrite running kernel pages with the saved image
9. Resume execution at hibernation point
```

### ハイバネートの保存先の設定 {#configuring-hibernate-destination}

```bash
# 特定のスワップパーティションを使う
echo /dev/sda2 | sudo tee /sys/power/resume

# またはスワップファイル: /sys/power/resume_offsetには、パーティション内での
# スワップファイルのヘッダの生のオフセットを<PAGE_SIZE>単位で指定する（FIBMAP
# ioctlで求める — Documentation/power/swsusp-and-swap-files.rstを参照）。その後
# /sys/power/resumeにそのファイルを含むパーティションを指定する
echo 34816 | sudo tee /sys/power/resume_offset
echo /dev/sda1 | sudo tee /sys/power/resume  # スワップファイルを含むパーティション

# カーネルコマンドラインで同等の指定をする場合:
# resume=/dev/sda1 resume_offset=34816

# 現在のハイバネートデバイスを確認する
cat /sys/power/resume
```

## フリーズ（s2idle / S0ix） {#freeze-s2idle-s0ix}

現代のノートパソコンは**s2idle**（suspend-to-idle。S0ixや「connected standby」と呼ばれることもある）を使う。S3とは異なり、CPUは非常に浅いスリープ状態に留まるため、次のことが可能になる。

- Bluetooth/WiFiが接続されたままでいられる
- バックグラウンドタスクが低いレートで実行される
- プラットフォーム全体の電源断と再初期化を元に戻す必要がないため、S3より起床が速い — どの程度速いかはプラットフォームとファームウェアによって大きく異なる

```bash
# s2idleを強制する（プラットフォームのS3サポートを無視する）
echo s2idle | sudo tee /sys/power/mem_sleep

# あるいは利用可能ならS3を優先する
echo deep | sudo tee /sys/power/mem_sleep

cat /sys/power/mem_sleep
# s2idle [deep]   (brackets = current selection)
```

## サスペンド失敗の診断 {#diagnosing-suspend-failures}

```bash
# 詳細なサスペンドログ
echo 1 | sudo tee /sys/power/pm_debug_messages
echo 1 | sudo tee /sys/power/pm_print_times

# サスペンドの試行が失敗した後
dmesg | grep -E "PM:|suspend|freeze" | tail -50

# どのデバイスがサスペンドに失敗したか?
dmesg | grep "error during suspend"

# サスペンド/レジュームの成功・失敗カウンタ: 統計ごとに1ファイルを持つ
# sysfsディレクトリ（kernel/power/main.c）であり、単一のdebugfsファイルではない
cat /sys/power/suspend_stats/success
# 42
cat /sys/power/suspend_stats/fail
# 2
cat /sys/power/suspend_stats/failed_suspend       # ← ここでドライバが失敗した
# 1
cat /sys/power/suspend_stats/last_failed_dev      # 最後に失敗したデバイスの名前
cat /sys/power/suspend_stats/last_failed_step     # どのフェーズで失敗したか
ls /sys/power/suspend_stats/
# success  fail  failed_freeze  failed_prepare  failed_suspend
# failed_suspend_late  failed_suspend_noirq  failed_resume
# failed_resume_early  failed_resume_noirq  last_failed_dev
# last_failed_errno  last_failed_step  last_hw_sleep
# total_hw_sleep  max_hw_sleep

# 各ドライバのサスペンド/レジュームコールバックにかかる時間: （上で有効にした）
# pm_print_timesによりdmesgに表示される — それ自体は統計ファイルではない
dmesg | grep "PM: .* took"

# 詳細なタイムラインのためにPMのtracepointを有効にする
echo 1 > /sys/kernel/tracing/events/power/device_pm_callback_start/enable
echo 1 > /sys/kernel/tracing/events/power/device_pm_callback_end/enable
cat /sys/kernel/tracing/trace_pipe > /tmp/suspend_trace &
echo mem | sudo tee /sys/power/state
# バックグラウンドのcatをCtrl-Cで止め、/tmp/suspend_traceを調べる
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/power/suspend.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/power/suspend.c) — `pm_suspend()`、`enter_state()`、`suspend_devices_and_enter()`: S1-S3/s2idleへの進入経路
- [kernel/power/hibernate.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/power/hibernate.c) — `hibernate()`、`hibernation_restore()`、および`/sys/power/resume`/`resume_offset`属性
- [kernel/power/snapshot.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/power/snapshot.c) — `swsusp_save()`: すべてのページを走査してハイバネーションイメージを構築する
- [include/linux/suspend.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/suspend.h) — `PM_SUSPEND_PREPARE`/`PM_POST_SUSPEND`/`PM_HIBERNATION_PREPARE`のnotifier定数、`register_pm_notifier()`
- [include/linux/pm_wakeup.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/pm_wakeup.h) — `struct wakeup_source`、`device_init_wakeup()`、`device_may_wakeup()`
- [drivers/base/power/main.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/base/power/main.c) — `dpm_prepare()`/`dpm_suspend()`/`dpm_suspend_late()`/`dpm_suspend_noirq()`と、それらに対応するレジューム側の関数

### 関連ページ {#related-pages}

- [cpufreq](cpufreq.md) — CPU周波数スケーリング
- [Runtime PM](runtime-pm.md) — デバイスレベルの電源管理
- [デバイスドライバ: プラットフォームドライバ](../drivers/platform-driver.md) — dev_pm_opsとの統合
- [割り込み: タイマー](../interrupts/timers.md) — `timer_list`、`hrtimer`、そしてサスペンド/レジュームをまたぐ時刻管理が依存するclocksourceの仕組み

### LWNの記事 {#lwn-articles}

- [A new suspend/hibernate infrastructure](https://lwn.net/Articles/274008/) — Jonathan Corbet、2008年3月19日。サスペンドとハイバネーションのデバイスコールバック経路を分離した、Rafael Wysockiによる再設計
- [PM / Sleep: Introduce new phases of device suspend/resume](https://lwn.net/Articles/475730/) — Rafael Wysockiによるlinux-pmのパッチシリーズ（2012年1月16日投稿。LWNのスタッフ記事ではなく、メーリングリストのスレッドとしてLWNにアーカイブされたもの）で、上で説明した`.suspend_late`/`.resume_early`フェーズを追加した。`_noirq`フェーズは上記の2008年の再設計の時点で既に存在しており、このシリーズで導入されたものではない。
- [Waking systems from suspend](https://lwn.net/Articles/429925/) — John Stultz、2011年3月2日。多数のユーザー空間アラームをtimerqueueによって単一のハードウェアRTCアラームに多重化するalarmtimerサブシステムと、それをどう公開するかというAPI設計上の問題（`CLOCK_REALTIME_ALARM`のようなPOSIXのアラームクロックID対Androidの`/dev/alarm` ioctlインターフェース）

### 外部資料 {#external}

- [Documentation/driver-api/pm/devices.rst](https://docs.kernel.org/driver-api/pm/devices.html) — デバイスのサスペンド/レジュームのフェーズモデル全体と、デバイス階層全体にわたる順序の保証
- [Documentation/admin-guide/pm/sleep-states.rst](https://docs.kernel.org/admin-guide/pm/sleep-states.html) — `freeze`/`standby`/`mem`/`disk`というラベルと、それらのACPI S-stateへの対応
- [Documentation/admin-guide/pm/suspend-flows.rst](https://docs.kernel.org/admin-guide/pm/suspend-flows.html) — suspend-to-idle、standby、suspend-to-RAMのコードフロー図
- [Documentation/power/swsusp-and-swap-files.rst](https://docs.kernel.org/power/swsusp-and-swap-files.html) — スワップパーティションの代わりにスワップファイルをハイバネーション先にする方法
