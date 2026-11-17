# Runtime PM {#runtime-pm}

> デバイスレベルの電源管理: システムの稼働中にアイドル状態のデバイスをサスペンドする

## Runtime PMとは何か {#what-is-runtime-pm}

Runtime PMは、システムの他の部分が動作を続けている間に、個々のデバイスを低電力状態に移行させる仕組みである。ディスプレイが接続されていなければGPUの電源を切ることができ、USBデバイスはアイドル時にサスペンドでき、PCIe NICはバーストとバーストの合間にPHYの電源を切ることができる。

```
System running (S0)
├── CPU: P-state varies with load
├── GPU:  [active] → [runtime suspended] after 2 seconds idle
├── NVMe: [active] → [runtime suspended] after 1 second idle
├── USB:  [runtime suspended] (nothing plugged in)
└── NIC:  [active]
```

## PMコールバック: struct dev_pm_ops {#pm-callbacks-struct-dev_pm_ops}

どのドライバも電源管理のコールバックを登録できる。

```c
/* include/linux/pm.h — 抜粋。システムスリープ中に呼ばれる_late/_early
   および_noirqの変種（さらに12個のコールバック）についてはシステムサスペンドを参照 */
struct dev_pm_ops {
    /* システムスリープ: システムのサスペンド/レジューム時に呼ばれる */
    int (*prepare)(struct device *dev);
    void (*complete)(struct device *dev);
    int (*suspend)(struct device *dev);
    int (*resume)(struct device *dev);
    int (*freeze)(struct device *dev);    /* ハイバネートのスナップショット */
    int (*thaw)(struct device *dev);
    int (*poweroff)(struct device *dev);
    int (*restore)(struct device *dev);
    /* ... suspend_late/resume_early、freeze_late/thaw_early、
       poweroff_late/restore_early、および6つの*_noirq変種 ... */

    /* Runtime PM: デバイスがアイドルになるか再開するときに呼ばれる */
    int (*runtime_suspend)(struct device *dev);
    int (*runtime_resume)(struct device *dev);
    int (*runtime_idle)(struct device *dev);
};
```

ドライバは次のようにしてこれらを登録する。

```c
static const struct dev_pm_ops mydriver_pm_ops = {
    SET_SYSTEM_SLEEP_PM_OPS(mydriver_suspend, mydriver_resume)
    SET_RUNTIME_PM_OPS(mydriver_runtime_suspend,
                       mydriver_runtime_resume,
                       mydriver_runtime_idle)
};

static struct platform_driver mydriver = {
    .driver = {
        .name   = "mydriver",
        .pm     = &mydriver_pm_ops,
    },
    /* ... */
};
```

## 使用カウント: getとput {#usage-counting-get-and-put}

Runtime PMの中核は**使用カウンタ**である。カウンタが0より大きい間、デバイスはアクティブなままであり、0に達するとサスペンドされうる。

```c
#include <linux/pm_runtime.h>

/* 使用カウントをインクリメントし、デバイスがアクティブになるのを待つ。
   デバイスがサスペンドされていれば、まずレジュームする。 */
int ret = pm_runtime_get_sync(dev);
if (ret < 0) {
    /* デバイスをレジュームできなかった — エラーを返す */
    return ret;
}

/* ... デバイスを使う ... */

/* 使用カウントをデクリメントする。カウントが0に達したらアイドルチェックをキューに入れる
   （runtime_idleを実行し、それが許せばruntime_suspendを実行する）。 */
pm_runtime_put(dev);

/* または: デクリメントして、autosuspendの遅延後にサスペンドをキューに入れる */
pm_runtime_put_autosuspend(dev);
```

`pm_runtime_put`の各変種の違いは次のとおりである。

| 関数 | 振る舞い |
|----------|---------- |
| `pm_runtime_put_sync` | カウントが0に達したらアイドルチェックを同期的に実行する — `runtime_idle()`を呼び、拒否されなければ直ちに`runtime_suspend()`を呼ぶ |
| `pm_runtime_put` | カウントが0に達したら非同期のアイドルチェックをキューに入れる |
| `pm_runtime_put_autosuspend` | カウントが0に達したら、autosuspendの遅延後のサスペンドをキューに入れる |
| `pm_runtime_put_noidle` | デクリメントするがサスペンドは引き起こさない |

## Autosuspend {#autosuspend}

autosuspendは、サスペンドの前に設定可能な遅延を追加する。これにより、活動のバーストの合間にデバイスが短時間アイドルになったときのスラッシングを防ぐ。

```c
/* probe内: 2秒の遅延でautosuspendを有効にする */
pm_runtime_set_autosuspend_delay(dev, 2000 /* ms */);
pm_runtime_use_autosuspend(dev);

/* デバイスを初期状態でアクティブとしてマークする（probeで起動したばかりなので）。
   これはpm_runtime_enable()より前に行わなければならない: runtime PMがそのデバイスで
   既に有効になっていると、__pm_runtime_set_status()は
   （状態を変更せずに）-EAGAINを返す。 */
pm_runtime_get_noresume(dev);
pm_runtime_set_active(dev);
pm_runtime_enable(dev);
pm_runtime_put_autosuspend(dev);
```

autosuspendの遅延はユーザー空間にもエクスポートされている。

```bash
# デバイスのautosuspend遅延を読み書きする（ms、-1 = 無効）
cat /sys/bus/usb/devices/1-1/power/autosuspend_delay_ms
echo 1000 | sudo tee /sys/bus/usb/devices/1-1/power/autosuspend_delay_ms
```

## Runtime PMの状態機械 {#runtime-pm-state-machine}

`RPM_IDLE`という状態は存在しない。アイドルは*コールバック*であって、デバイスが
留まる状態ではない。`runtime_idle()`コールバックが実行されている間、
デバイスはずっと`RPM_ACTIVE`のままである。そのコールバック（または呼び出し元）が
続けてサスペンドを要求した場合にのみ、状態が変化する。

```
           pm_runtime_enable()
                   │
                   ▼
              [RPM_ACTIVE]
              usage_count > 0
                   │
         pm_runtime_put() → count=0
                   │
                   ▼
      still [RPM_ACTIVE] — runtime_idle() callback fires
                   │
    idle CB (or no callback) allows suspend → pm_runtime_suspend()
                   │
                   ▼
           [RPM_SUSPENDING]
      runtime_suspend() callback
                   │
                   ▼
           [RPM_SUSPENDED]
                   │
         pm_runtime_get_sync()
                   │
                   ▼
           [RPM_RESUMING]
      runtime_resume() callback
                   │
                   ▼
              [RPM_ACTIVE]
```

実際の状態（`include/linux/pm.h`の`enum rpm_status`）は、
`RPM_INVALID`、`RPM_ACTIVE`、`RPM_RESUMING`、`RPM_SUSPENDED`、
`RPM_SUSPENDING`、`RPM_BLOCKED`である。

### アイドルコールバック {#the-idle-callback}

アイドルコールバックは、使用カウントが0に達したときに呼ばれる。実際にサスペンドするかどうかを決定する。

```c
static int mydriver_runtime_idle(struct device *dev)
{
    struct mydata *priv = dev_get_drvdata(dev);

    /* 保留中の作業があればサスペンドしない */
    if (!list_empty(&priv->pending_list))
        return -EBUSY;  /* サスペンドしない */

    /* PMコアにruntime_suspendを呼ばせる */
    return pm_runtime_autosuspend(dev);
    /* または: return 0（PMコアが直ちにruntime_suspendを呼ぶ） */
}
```

## struct deviceの電源関連フィールド {#struct-device-power-fields}

```c
/* include/linux/pm.h — 抜粋。実際の構造体には約55個のフィールドがある */
struct dev_pm_info {
    pm_message_t        power_state;    /* 現在の電源状態 */
    bool                can_wakeup:1;   /* デバイスがウェイクアップイベントを生成できる */
    bool                async_suspend:1;
    bool                in_dpm_list:1;

    /* Runtime PMのフィールド（#ifdef CONFIG_PM の内側） */
    struct hrtimer      suspend_timer;  /* autosuspendタイマー */
    u64                 timer_expires;
    struct work_struct  work;

    wait_queue_head_t   wait_queue;

    atomic_t            usage_count;    /* get/putのカウンタ */
    atomic_t            child_count;    /* アクティブな子の数 */

    unsigned int        disable_depth:3; /* pm_runtime_disable()呼び出しの深さ */
    int                 runtime_error;

    enum rpm_status     runtime_status; /* 現在のruntime PM状態 */
};
```

## 電源ドメイン {#power-domains}

電源ドメインとは、独立して電源を切ることができるハードウェアブロックである。複数のデバイスが1つのドメインを共有することがあり、ドメインはその中のすべてのデバイスがサスペンドされるまでオンのままである。

```c
/* include/linux/pm_domain.h */
struct generic_pm_domain {
    struct device          dev;
    struct dev_pm_domain domain;    /* 埋め込み — dev_pm_opsを持つ */
    struct list_head      gpd_list_node;

    const char           *name;
    atomic_t              sd_count;  /* 電源が「オン」のサブドメインの数 */
    enum gpd_status       status;    /* GENPD_STATE_ON / GENPD_STATE_OFF */

    unsigned int          device_count;
    unsigned int          device_id;       /* 一意なデバイスID */
    unsigned int          suspended_count;
    unsigned int          prepared_count;
    unsigned int          performance_state; /* 集約された最大パフォーマンス状態 */

    struct dev_power_governor *gov;

    struct list_head      parent_links; /* 親ドメイン */
    struct list_head      child_links;  /* 子ドメイン */
    struct list_head      dev_list;     /* このドメイン内のデバイス */

    int (*power_off)(struct generic_pm_domain *domain);
    int (*power_on)(struct generic_pm_domain *domain);
};
```

### genpdの使い方（ARM SoCの例） {#genpd-usage-arm-soc-example}

```c
/* SoCの電源ドメインのセットアップ（プラットフォームコードまたはデバイスツリー内） */
static struct generic_pm_domain gpu_pd = {
    .name      = "gpu",
    .power_off = gpu_domain_power_off,
    .power_on  = gpu_domain_power_on,
};

/* デバイスをドメインに接続する */
pm_genpd_add_device(&gpu_pd, &gpu_dev);

/* これで: gpu_pd内のすべてのデバイスがランタイムサスペンドされると、
   power_off()が自動的に呼ばれる */
```

## 割り込みコンテキストでのRuntime PM {#runtime-pm-in-interrupt-context}

`pm_runtime_get_sync`は（レジュームを待つために）スリープすることがある。これは割り込みコンテキストでは許されない。ノンブロッキングの変種を使う。

```c
/* 割り込みハンドラ内: getを試みるが、スリープはしない。
   pm_runtime_get_if_active()は3値を返す: 1 = アクティブ（カウントが増えた）、
   0 = アクティブでない、-EINVAL = このデバイスではruntime PMが無効。
   「アクティブでない」と「無効」のどちらの場合もデバイスをスキップしなければならない —
   使用カウントを増やしたのはret == 1の場合だけなので、put()を呼んでよいのもその場合だけである。 */
ret = pm_runtime_get_if_active(dev);
if (ret <= 0) {
    /* デバイスはサスペンド中（またはruntime PMが無効） — 作業をスケジュールする */
    schedule_work(&priv->deferred_work);
    return IRQ_HANDLED;
}

/* デバイスはアクティブ */
handle_interrupt(dev);
pm_runtime_put(dev);
return IRQ_HANDLED;
```

## Runtime PMの観測 {#observing-runtime-pm}

```bash
# すべてのPCIデバイスのRuntime PM状態
for d in /sys/bus/pci/devices/*/; do
    echo -n "$d: "
    cat "$d/power/runtime_status" 2>/dev/null || echo "N/A"
done

# 使用カウントとautosuspend
cat /sys/bus/pci/devices/0000:00:02.0/power/runtime_usage
cat /sys/bus/pci/devices/0000:00:02.0/power/autosuspend_delay_ms

# デバイスのruntime PMを有効にする（一部のドライバはユーザー空間からの有効化が必要）
echo auto | sudo tee /sys/bus/usb/devices/1-1/power/control
echo on   | sudo tee /sys/bus/usb/devices/1-1/power/control  # 無効化

# PMのtracepoint（rpm_suspend、rpm_resume、rpm_idle、rpm_usage、
# rpm_return_int、rpm_status — include/trace/events/rpm.h）
echo 1 > /sys/kernel/tracing/events/rpm/enable
cat /sys/kernel/tracing/trace_pipe
# 0000:01:00.0 flags-0 cnt-0 dep-0 auto-1 p-0 irq-0 child-0
# 0000:01:00.0 flags-4 cnt-0 dep-0 auto-1 p-0 irq-0 child-0

# powertopはデバイスのruntime PMの活動を表示する
sudo powertop --html=powertop.html
```

## よくある落とし穴 {#common-pitfalls}

### pm_runtime_enableを忘れる {#forgetting-pm_runtime_enable}

```c
/* 誤り: runtime PMはデフォルトで無効になっている */
static int mydriver_probe(struct platform_device *pdev)
{
    /* ... セットアップ ... */
    /* 欠落: pm_runtime_enable(&pdev->dev) */
    return 0;
}
/* pm_runtime_enable()がないと、disable_depthはpm_runtime_init()が
 * 設定した値（1）のままになる。その後このデバイスに対して
 * pm_runtime_get_sync()を呼ぶと、rpm_resume()を通ってdisable_depth > 0を見つけ、
 * -EACCESを返す。 */
```

### get/putの釣り合いが取れていない {#not-balancing-getput}

```c
/* 誤り: エラーパスでputを飛ばしている */
static int mydriver_do_io(struct device *dev)
{
    pm_runtime_get_sync(dev);
    if (error_condition)
        return -EIO;  /* getがリークしている! usage_countがデクリメントされない */
    pm_runtime_put(dev);
    return 0;
}

/* 正しい: 常にputする */
static int mydriver_do_io(struct device *dev)
{
    int ret;
    pm_runtime_get_sync(dev);
    ret = do_actual_io(dev);
    pm_runtime_put(dev);
    return ret;
}
```

### アトミックコンテキストでget_syncを呼ぶ {#calling-get_sync-in-atomic-context}

```c
/* 誤り: pm_runtime_get_syncはスリープすることがある */
spin_lock_irqsave(&lock, flags);
pm_runtime_get_sync(dev);  /* スリープしうる! バグ */
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/pm.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/pm.h) — `struct dev_pm_ops`、`struct dev_pm_info`、および`SET_RUNTIME_PM_OPS()`/`SET_SYSTEM_SLEEP_PM_OPS()`マクロ
- [include/linux/pm_runtime.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/pm_runtime.h) — `pm_runtime_get*()`/`pm_runtime_put*()`のインラインラッパーとautosuspendのヘルパー
- [drivers/base/power/runtime.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/base/power/runtime.c) — runtime PMのコア: `__pm_runtime_suspend()`、`__pm_runtime_resume()`、`pm_runtime_get_if_active()`、および`RPM_*`の状態機械
- [include/linux/pm_domain.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/pm_domain.h) — `struct generic_pm_domain`と`enum gpd_status`
- [drivers/pmdomain/core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pmdomain/core.c) — `pm_genpd_add_device()`とgenpdのコア（ドライバコアの再編成の際に`drivers/base/power/domain.c`からここへ移された）

### 関連ページ {#related-pages}

- [電源ドメインとgenpd](power-domains.md) — `struct generic_pm_domain`、ガバナ、デバイスツリーバインディングの詳しい解説
- [cpufreq](cpufreq.md) — CPUレベルの周波数/電圧スケーリング
- [システムサスペンド](suspend.md) — システム全体のスリープ状態
- [デバイスドライバ: プラットフォームドライバ](../drivers/platform-driver.md) — devm_とprobeの流れ

### LWNの記事 {#lwn-articles}

- [Runtime power management](https://lwn.net/Articles/347573/) — Jonathan Corbet、2009年8月19日。Rafael Wysockiによる最初のruntime PMパッチセットと、上で説明した`runtime_suspend`/`runtime_resume`/`runtime_idle`コールバック

### 外部資料 {#external}

- [Documentation/power/runtime_pm.rst](https://docs.kernel.org/power/runtime_pm.html) — runtime PM API、使用カウントの規則、ロックに関するアップストリームのリファレンス
