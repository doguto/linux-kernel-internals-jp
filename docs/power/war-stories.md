# 電源管理の事件簿 {#power-management-war-stories}

> サスペンドのリグレッション、サーマルスロットリングの嵐、ランタイムPMのデッドロック

電源管理は、カーネルの中でも最もリグレッションが起きやすい領域の1つである。状態機械は複雑で、ハードウェアは多種多様であり、バグはタイミングやプラットフォームに依存することが多い。

!!! note "これらの記事の書き方について"
    ここに挙げるのは、1つのコミットやCVEに結びついた特定の名前付きインシデントの記録ではなく、実際に繰り返し起きている失敗パターンを例示したものである。参考文献にある日付付きのLWNの引用は、各インシデントが前提とする仕組み（`dev_pm_ops`のランタイムコールバック、サーマルガバナ、RAPLのpowercapインターフェース）を裏付けるものであり、インシデントそのものの出典ではない。このサイトの通常の形式、すなわち日付を特定できる具体的なインシデントで、その大半が（すべてではない — [wake_wide()](../sched/war-stories/wake-wide-heuristic.md)を参照）修正コミットやCVEに結びついているものについては、[BPF](../bpf/war-stories.md)、[スケジューラ](../sched/war-stories.md)、[割り込み](../interrupts/war-stories.md)の事件簿ページを参照のこと。

---

## 1. PMコールバックの欠落によるサスペンドのリグレッション {#1-the-suspend-regression-from-a-missing-pm-callback}

### 状況 {#setup}

あるPCIe NICドライバがアップストリームに投稿された。このドライバは`probe`、`remove`、データパスを実装していたが、`dev_pm_ops`構造体は空で、`suspend`や`resume`のコールバックがなかった。

```c
static const struct dev_pm_ops mynic_pm_ops = {
    /* ここには何もない */
};
```

このドライバは、サスペンドを使わない投稿者のテストマシンでは完璧に動作していた。

### 何が起きたか {#what-happened}

このドライバを動かすノートパソコンで:

1. ユーザーが蓋を閉じる。systemdが`echo mem > /sys/power/state`を呼ぶ。
2. PMコアが`dpm_suspend()`を呼び、デバイスリストを走査する。各デバイスについて、`device_suspend()`（`drivers/base/power/main.c`）は、`pm_domain`、`type`、`class`、`bus`の順に確認して最初に見つかったNULLでない`dev_pm_ops`を選び、どれもコールバックを提供しない場合にのみドライバ自身の`dev_pm_ops`にフォールバックする。PCIデバイスの場合、`dev->bus->pm`は`pci_dev_pm_ops`（`drivers/pci/pci-driver.c`。`PCI_PM_OPS_PTR`マクロで接続される）であり、`CONFIG_PM`が有効なビルドでは常にNULLではない。そのため、このデバイスのサスペンドは`mynic_pm_ops`の中身にかかわらず`pci_pm_suspend()`を通る。
3. `pci_pm_suspend()`の中で、バスレベルのコードは*ドライバの*コールバック自体を確認する: `if (pm->suspend) { pm->suspend(dev); ... }`。`mynic_pm_ops.suspend`はNULLなので、このブロックは単に飛ばされ、ドライバのコードは何も実行されない。しかし`pci_pm_suspend()`はそれでも0を返し、汎用のPCIサスペンド処理は先に進む。
4. 同じサスペンドの手順の後のほうで、`pci_pm_suspend_noirq()`は無条件に実行される。これは`pci_save_state()`を呼び、実際の`suspend`コールバックを持つドライバの場合とまったく同じようにデバイスをD3に移行させる。この呼び出しが保存するのは*汎用のPCIコンフィグ空間*の状態（BAR、コマンドレジスタ、MSI/MSI-Xの設定）である。NICのプライベートなハードウェア状態については何も知らない。それを保存するのはドライバの仕事であり、ドライバのコールバックは一度も実行されていないからである。PCIeホストコントローラは正しくサスペンドされ、バスの電源を落とす。
5. レジューム時、D3coldからの復帰は、PCIe仕様（r6.0 §6.6.1）で**Conventional Reset**と呼ばれるものを意味する。具体的には、ソフトウェアが開始するHot Reset（ブリッジのSecondary Bus Resetビット、§6.6.1）ではなく、電源投入の遷移が引き起こす種類であるFundamental Resetである。リセット後にコンフィグ空間へのアクセスを再試行する前に、PCIコアのコード（`pci_dev_wait()`、`drivers/pci/pci.c`）は固定の遅延を置くのではなく、直ちに指数バックオフでポーリングを始める。`PCI_RESET_WAIT`（1000ms）は、まだ待っていることをログに出し始める時点であって、リセット前の待機時間ではない。このレジュームパスがドライバに到達する頃には、デバイスはすでにそのチェックを通過しており、コンフィグ空間の読み取りに正常に応答している。
6. 汎用の*PCIコンフィグ空間*（標準のPCIヘッダ — BAR、コマンドレジスタ — とMSI/MSI-Xのケーパビリティ構造体）は、バスレベルのレジュームパスによって復元される。一方、それらのBARの背後にある、ドライバが所有するNIC固有のプライベートなMMIOレジスタは、Fundamental Resetの結果、電源投入時のデフォルト値になっている。通常はすべてゼロであり、サスペンド前に保持していた値ではない。ドライバにはこのことが一切知らされていない（`resume`コールバックも実行されていない）ため、メモリ上の管理情報 — DMAリングの先頭が現在どこにあると思っているか、どのディスクリプタが処理中か — は、黙って白紙に戻ってしまったハードウェアに対して古いものになっている。
7. ドライバの割り込みハンドラは、その古い管理情報に基づいて動作する。記憶している処理中の状態があるものと期待してDMAリングの先頭ポインタレジスタにアクセスするが、ハードウェアはそのことを何も覚えていない。ドライバによっては、期待と一致しない値を読み戻すか、ドライバ自身の整合性チェックが想定していない状態にリングのロジックを追い込む。ドライバはこの不一致を致命的なハードウェアエラーと解釈し、`BUG()`を呼ぶ。

### 診断 {#diagnosis}

カーネルパニックのコールトレースは、NICドライバの割り込みハンドラを指していた。`CONFIG_PM_DEBUG`と`CONFIG_PM_SLEEP_DEBUG`を有効にすると追加のログが出力され、デバイスが自身のサスペンドパスを通っていないことがわかった。正常に動作するドライバと比較したところ、PMコールバックの欠落が判明した。

### 修正 {#fix}

```c
static int mynic_suspend(struct device *dev)
{
    struct mynic_priv *priv = dev_get_drvdata(dev);

    /* DMAと割り込みを止める */
    mynic_disable_irq(priv);
    mynic_stop_dma(priv);

    /* 電源断を越えて保持すべきレジスタを保存する */
    mynic_save_state(priv);

    /* PCIコンフィグ空間の保存とD3への遷移はPCIコアに任せる */
    return 0;
}

static int mynic_resume(struct device *dev)
{
    struct mynic_priv *priv = dev_get_drvdata(dev);

    mynic_restore_state(priv);
    mynic_reset_hw(priv);
    mynic_init_rings(priv);
    mynic_enable_irq(priv);
    return 0;
}

/* DEFINE_SIMPLE_DEV_PM_OPSは、システムスリープのコールバック（suspend/resume/
   freeze/thaw/poweroff/restore）を同じ関数のペアで埋める。Runtime
   PMのコールバックにはSET_RUNTIME_PM_OPSが必要である。（古いSIMPLE_DEV_PM_OPS
   マクロも同じことをするが、こちらを使うよう非推奨になっている。） */
static DEFINE_SIMPLE_DEV_PM_OPS(mynic_pm_ops, mynic_suspend, mynic_resume);
```

`DEFINE_SIMPLE_DEV_PM_OPS`（`include/linux/pm.h`で定義）は、システムスリープのコールバック（`suspend`、`resume`、`freeze`、`thaw`、`poweroff`、`restore`）を同じ関数のペアで埋める便利なマクロである。注意: これはruntime PMのコールバック（`runtime_suspend`、`runtime_resume`）は**接続しない**。それらには別途`SET_RUNTIME_PM_OPS`が必要である。

**教訓**: ハードウェア状態を持つデバイスにとって、コールバックがないことは安全なデフォルトではない。開発中は`CONFIG_PM_DEBUG`を使い、ドライバを投稿する前に`echo mem > /sys/power/state`でテストすること。

---

## 2. ランタイムPMのデッドロック {#2-the-runtime-pm-deadlock}

### 状況 {#setup_1}

I2C接続のタッチコントローラ用のドライバがRuntime PMを使っていた。ハンドラはスリープしうるI2C転送を発行する必要があるため、ドライバはそれをスレッド化IRQハンドラとして登録した: `request_threaded_irq(irq, NULL, touch_irq, IRQF_ONESHOT, "mytouch", priv)`。これにより`touch_irq`は専用のカーネルスレッド（`irq/N-mytouch`という名前）で実行される。これは通常のプロセスコンテキストであり、スリープが許される。ハンドラはデバイスから座標を読み取り、イベントをキューに入れていた。

```c
static irqreturn_t touch_irq(int irq, void *data)
{
    struct touch_priv *priv = data;

    spin_lock(&priv->lock);

    /* デバイスがランタイムサスペンド中なら起こす */
    pm_runtime_get_sync(priv->dev);   /* バグ: spinlockを保持したままスリープしうる */

    read_touch_registers(priv);       /* I2C転送 */
    queue_event(priv);

    pm_runtime_put(priv->dev);
    spin_unlock(&priv->lock);
    return IRQ_HANDLED;
}
```

### 何が起きたか {#what-happened_1}

`pm_runtime_get_sync()`は使用カウンタをインクリメントする。デバイスが既にアクティブなら、すぐに戻る。しかし`runtime_status == RPM_SUSPENDED`であれば、ドライバの`->runtime_resume()`コールバックを呼び、それがハードウェアを低電力状態から戻すためにI2C転送を行う。I2C転送はスリープしうる（バストランザクションの完了を待つ）。そして数行後の`read_touch_registers()`自体も、runtime PMの状態にかかわらずスリープしうる。

`irq/N-mytouch`スレッドの中で実行されているので、本来ならここでスリープしても問題ない。バグは、この一連の処理全体を囲む`spin_lock(&priv->lock)`にある。`spin_lock()`はどのコンテキストから呼ばれてもプリエンプションを無効にし、スリープするにはスケジューラが別のタスクを実行できるようプリエンプションが有効になっている必要がある。結果として、`might_sleep()`（`CONFIG_DEBUG_ATOMIC_SLEEP`）によるBUGの出力、さらに悪ければ無言のデッドロックが起きる。

`CONFIG_DEBUG_ATOMIC_SLEEP`を有効にしていたため、`might_sleep()`がこれを直ちに捕捉した。

```
BUG: sleeping function called from invalid context at kernel/locking/mutex.c:580
in_atomic(): 1, irqs_disabled(): 0, non_block: 0, pid: 88, name: irq/86-mytouch
...
Call Trace:
  __might_sleep
  mutex_lock_nested
  i2c_transfer
  touch_runtime_resume
  rpm_resume
  pm_runtime_get_sync
  touch_irq
```

### 修正 {#fix_1}

本質的なバグは「間違ったPM呼び出し」ではない。I2C転送を含む一連の処理にわたって`spin_lock(&priv->lock)`が保持されていることであり、I2C転送は常にスリープしうる。以下の正しい修正はどちらも、スリープする処理（PMのレジューム*と*`read_touch_registers()`）を、いかなるspinlockのクリティカルセクションからも完全に外に出す。

**選択肢A** — スリープしうる処理をすべてworkqueueに先送りする。spinlockが保護するのは単純なフラグだけになる。

```c
static irqreturn_t touch_irq(int irq, void *data)
{
    struct touch_priv *priv = data;

    /* irq/N-mytouchスレッドで実行されるが、このハンドラ自体は
     * スリープしないようにしてI2Cの処理を引き渡す — ここでは
     * スリープをまたいでpriv->lockを保持する必要はない。 */
    spin_lock(&priv->lock);
    priv->irq_pending = true;
    spin_unlock(&priv->lock);

    schedule_work(&priv->irq_work);
    return IRQ_HANDLED;
}

static void touch_irq_work(struct work_struct *work)
{
    struct touch_priv *priv = container_of(work, struct touch_priv, irq_work);

    /* workqueueコンテキスト: スリープしてよい。pm_runtime_get_sync()と
     * read_touch_registers()はどちらもspinlockを保持せずに実行される。 */
    pm_runtime_get_sync(priv->dev);
    read_touch_registers(priv);        /* I2C転送 */

    spin_lock(&priv->lock);
    queue_event(priv);
    priv->irq_pending = false;
    spin_unlock(&priv->lock);

    pm_runtime_put(priv->dev);
}
```

**選択肢B** — `touch_irq`は既にスリープ可能な`irq/N-mytouch`スレッドで実行されているので、spinlockを回避するのではなくmutexに置き換える。ここではアトミックである必要のあるものは何もない。

```c
static irqreturn_t touch_irq(int irq, void *data)
{
    struct touch_priv *priv = data;

    /* priv->data_mutex（struct mutex）が、以前このデータを保護していた
     * spinlockを置き換える。mutexの下でのスリープは問題ないので、PMの
     * レジュームとI2C転送はここにインラインのまま置ける。 */
    mutex_lock(&priv->data_mutex);

    pm_runtime_get_sync(priv->dev);
    read_touch_registers(priv);        /* I2C転送 — mutexの下なら問題ない */
    queue_event(priv);
    pm_runtime_put(priv->dev);

    mutex_unlock(&priv->data_mutex);
    return IRQ_HANDLED;
}
```

選択肢Bが成り立つのは、`touch_irq`がhardirqやsoftirqのコンテキストから実行されることが決してないからにすぎない。このハンドラが他の場所でアトミックコンテキストから呼ばれる可能性があるなら、mutexも元のspinlockと同様に壊れており、選択肢Aの先送りだけが安全な選択となる。

**教訓**: `pm_runtime_get_sync()` — およびあらゆるI2C/SPI/regmapアクセス — はスリープする可能性のある関数である。spinlockを保持したまま、生のhardirqコンテキストから、あるいはその他のアトミックコンテキストから実行してはならない。スレッド化IRQハンドラはスリープを許されるが、その中でspinlockを取るとその利点を捨てることになる。スリープする呼び出しと一緒に触れるデータはmutexで保護するか、スリープする処理をworkqueueに先送りしてspinlockの範囲をスリープしない管理処理だけに限定すること。アトミックコンテキストから本当に抜け出せず、構造を変えられない場合には`pm_runtime_get_if_active()`を使う。

---

## 3. サーマルスロットリングの嵐 {#3-the-thermal-throttle-storm}

### 状況 {#setup_2}

モバイルSoCのベンチマークスイートが、持続的なCPUバウンドのワークロードを実行した。デバイスにはCPUクラスタをカバーする単一のサーマルゾーンがあり、`step_wise`ガバナと2つの冷却デバイス、すなわちCPU周波数（cpufreq cooling経由）とパッシブ通知を持っていた。

### 何が起きたか {#what-happened_2}

```
t=0s:   CPU at 2.4 GHz, temperature rising
t=10s:  Temperature hits passive trip (85°C), trend is THERMAL_TREND_RAISING
        step_wise: raise cooling_state by 1 → frequency drops to 2.1 GHz
t=12s:  Temperature still rising (display + GPU also hot, but not managed)
        step_wise: raise again → 1.8 GHz
t=14s:  Still rising
        step_wise: 1.5 GHz
...
t=30s:  CPU at 200 MHz (minimum), temperature has barely moved
        Benchmark reports 8% of expected throughput
```

`step_wise`の`get_target_state()`は、傾向が`THERMAL_TREND_RAISING`である評価のたびに`cooling_state`を1段階上げ（`cur_state + 1`。冷却デバイスの上限でクランプされる）、ゾーンが冷えるにつれて1段階ずつ下げる。ガバナはゾーン内のトリップポイントに結び付けられたすべての冷却デバイスを順に処理する（単一のデバイスに限定されてはいない）が、このサーマルゾーンに結び付けられたアクターはCPU周波数デバイス*だけ*だった。GPUとディスプレイは合わせてCPUより多くの電力を消費していたが、どちらもこのゾーンのどこにも冷却デバイスとして登録されていなかったため、ガバナはそれらについて何も知らなかった。CPUが主な熱源ではなかったため、CPUの周波数を下げてもほとんど効果がなかった。

さらに、`step_wise`には積分項がない。温度がトリップポイントをどれだけ上回っているか、どれだけ長くその状態が続いているかにかかわらず、各評価周期で傾向が示す方向にちょうど1つの固定ステップだけ進む。この固定のステップ幅こそが、上のタイムラインでCPUの冷却状態が数秒ごとに下限まで段階的に下がり続け、温度を安定させる状態に一度も収束しなかった理由である。

### 修正 {#fix_2}

**`power_allocator`に切り替える**: Intelligent Power Allocation（IPA）ガバナはPIDコントローラを使い、サーマルゾーン内のすべての冷却デバイスをまとめて管理する。各冷却デバイスに消費電力を問い合わせ、全体の予算を再配分する。

```bash
echo power_allocator > /sys/class/thermal/thermal_zone0/policy

# 持続可能な電力の総予算（mW）。デバイスごとに調整する
echo 4000 > /sys/class/thermal/thermal_zone0/sustainable_power

# PIDのゲイン（デバイスの熱容量に合わせて調整する）
echo 500 > /sys/class/thermal/thermal_zone0/k_po   # オーバーシュートゲイン
echo 500 > /sys/class/thermal/thermal_zone0/k_pu   # アンダーシュートゲイン
echo 0   > /sys/class/thermal/thermal_zone0/k_d    # 微分
```

**GPUとディスプレイを同じサーマルゾーンの冷却デバイスとして登録する**。すべてのアクターが管理されていれば、ガバナはCPU周波数と並行してディスプレイの輝度やGPU周波数を下げることができ、最小の熱源だけがスロットリングされるという状況を防げる。

**より早いパッシブトリップポイントを追加する**: トリップポイントが85°Cで発火しても、熱容量のせいで温度がさらに5秒間上がり続けるなら、ガバナは常に後手に回る。パッシブトリップを75°Cに移せば、コントローラにより多くの猶予ができる。

**教訓**: `step_wise`は、単純でよく分離されたサーマルゾーンに適している。相互に影響する複数の熱源を持つSoCでは、すべてのアクターを登録した`power_allocator`が閉ループ制御を提供する。熱設計とソフトウェアは協調して設計しなければならない。

---

## 4. RAPL電力制限の思わぬ落とし穴 {#4-the-rapl-power-limit-surprise}

### 状況 {#setup_3}

あるクラウド事業者が新しいサーバーを導入し、課金のためにサーバーごとの電力予算を強制する目的で、powercapのsysfsインターフェースを通じてRAPLの長期電力制限を設定した。

```bash
# パッケージ0の総電力制限を150 Wに設定する
echo 150000000 > /sys/class/powercap/intel-rapl:0/constraint_0_power_limit_uw
# "enabled"はゾーン単位の属性（ゾーン内のすべての制約で共有される）であり、
# 制約ごとではない — constraint_0_enabledというファイルは存在しない。
echo 1 > /sys/class/powercap/intel-rapl:0/enabled
```

### 何が起きたか {#what-happened_3}

ワークロードは読み込み中心の混合データベースで、CPU演算は中程度、メモリ帯域幅は高かった。監視では次のように見えた。

```
CPU utilization:  35%
Memory bandwidth: 85 GB/s  (high)
Server throughput: 30% of baseline
```

データベースチームはカーネルのリグレッションを疑った。大規模なプロファイリングの結果、CPUはメモリ読み込みで頻繁にストールしていることがわかった。DRAMが遅いからではなく、DRAMアクセスが*人為的にスロットリングされていた*からである。テスト機でDRAMのRAPL制限を一時的に引き上げると、同じワークロードは全帯域幅を出すために持続的に約45 WのDRAM電力を消費した。デフォルトの制限に戻すと、RAPLのDRAMドメインの電力制限に達していた。

```bash
# 2つのドメインのエネルギーカウンタを1秒間読む
cat /sys/class/powercap/intel-rapl:0:1/name
# "dram"
# （energy_ujは約20 Wに相当するレートで増加していた — DRAMドメインの上限
#  ちょうどであり、ワークロードが消費しようとした約45 Wを大きく下回っていた）
```

問題はこうである。事業者はパッケージレベルで150 Wの電力制限を設定していた。しかしDRAMドメインは、パッケージドメインとは独立して計測され、上限が設けられている。DRAMドメインは独自のエネルギーカウンタと制約を持つ別のRAPLドメインであり、パッケージの電力予算から切り出された一部ではない。そしてDRAMドメインは、プラットフォームで設定された約20 Wのデフォルトのまま放置されていた。高帯域幅のワークロードはDRAMの電力をその別個の20 W制限を大きく超えるまで押し上げようとしていたため、ハードウェアはそこに留めるためにメモリコントローラのスループットをスロットリングした。150 Wのパッケージ上限にどれだけ余裕があるかとは無関係にである。

パッケージの制限を設定しても、DRAMドメインの制限は変わらなかった。事業者は、このドメインが独立して上限を設けられていることに気づいていなかった。

### 修正 {#fix_3}

```bash
# すべてのゾーンと現在の制限を列挙する
for zone in /sys/class/powercap/intel-rapl:*/; do
    name=$(cat "$zone/name" 2>/dev/null)
    limit=$(cat "$zone/constraint_0_power_limit_uw" 2>/dev/null)
    echo "$name: $limit µW"
done
# package-0: 150000000 µW
# core:      (not capped)
# uncore:    (not capped)
# dram:      20000000 µW   ← これがボトルネックだった

# DRAMドメインの制限を、ワークロードの約45 Wの需要より上に引き上げる
echo 50000000 > /sys/class/powercap/intel-rapl:0:1/constraint_0_power_limit_uw
```

DRAMの制限をワークロードの約45 Wの需要を上回る50 Wに引き上げると、メモリ帯域幅のスロットリングは消え、スループットはベースラインに戻った。CPUコアの負荷は35%にすぎなかったため、それでも150 Wのパッケージ予算の範囲内に十分収まっていた。

**教訓**: RAPLのドメイン制限は互いに独立して設定・強制される。電力上限を設定する前に、`for zone in /sys/class/powercap/intel-rapl:*/; do cat $zone/name $zone/constraint_0_power_limit_uw; done`ですべてのドメインを列挙し、実際のボトルネックがどのドメインかを把握すること（globにコロンがあることに注意。コロンのない`intel-rapl*`は、ゾーンだけでなく`intel-rapl`コントロールタイプのディレクトリ自体にも一致してしまう）。ワークロードの特性評価の際には、すべてのドメインの`energy_uj`を監視すること。

---

## 5. S2idleのウェイクアップソースの謎 {#5-the-s2idle-wakeup-source-mystery}

### 状況 {#setup_4}

あるノートパソコンが、S2idle（suspend-to-idle、`echo s2idle > /sys/power/mem_sleep`）をデフォルトのスリープモードとして配備された。ユーザーからは、何かが接続されているかどうかにかかわらず、蓋を閉じてから約30秒後に毎回ノートパソコンが起きてしまうと報告された。

### 診断 {#diagnosis_1}

```bash
# 早すぎる起床の直後にdmesgを観察する
dmesg | grep -i "wakeup\|wake\|resume" | tail -20
# PM: Waking up from sleep state 'freeze'
# PM: resume from suspend-to-idle
# （メッセージにデバイス名がない — ファームウェアレベルのウェイクアップ）

# wakeup_sourcesで最近の活動を確認する
# 列（drivers/base/power/wakeup.c、print_wakeup_source_stats()）:
# name  active_count  event_count  wakeup_count  expire_count  active_since
# total_time  max_time  last_change  prevent_suspend_time
cat /sys/kernel/debug/wakeup_sources | sort -k4 -rn | head -10
# name                active_count  event_count  wakeup_count  ...
# i2c-touchpad        47            47            47           ...
# （wakeup_countが、蓋を閉じてスリープを試みた回数と一致する）
```

`event_count`（`include/linux/pm_wakeup.h`）は、ソースが報告するすべてのウェイクアップイベントを無条件に数える。`wakeup_count`はそれより狭く、`wakeup_source_report_event()`（`drivers/base/power/wakeup.c`）は`events_check_enabled`がセットされているときにのみこれをインクリメントする。このフラグはサスペンドが起きることとはまったく結びついていない。これをセットするのは`pm_save_wakeup_count()`であり、ユーザー空間の`/sys/power/wakeup_count`プロトコルの背後にある関数である。呼び出し側（通常はsystemd）はサスペンドを要求する前に現在の登録済みイベント数をそこに書き込み、カーネルはその間に何も変化していなかった場合にのみ`wakeup_count`の追跡を有効にする。このトレースではそのプロトコルが使われていたため、ここでは両方のカウンタが一緒に増えていった。I2Cタッチパッドの`wakeup_count`（と`event_count`）は、蓋を閉じるたびにそれぞれ1ずつ増えていた。タッチパッドのファームウェアは、蓋が閉じていて指が触れていない状態でも、アイドル中に偽の割り込みアサートを発生させていた。

`ftrace`で確認する。

```bash
echo 1 > /sys/kernel/debug/tracing/events/power/wakeup_source_activate/enable
cat /sys/kernel/debug/tracing/trace_pipe
# wakeup_source_activate: i2c-touchpad state=0x1
```

（このtracepointのフォーマット文字列は`"%s state=0x%lx"`であり、ソース名がそのまま出て、`name=`という接頭辞はない。`state`は`combined_event_count`（`drivers/base/power/wakeup.c`）であり、2つのカウンタを1ワードに詰め込んだもの — 上位ビットに登録済みイベント、下位ビットに現在処理中のイベント — であって、単独の処理中カウントではない。ここでの`state=0x1`は、1つのイベントが登録され、処理中のものは（まだ）ないことを意味する。）

ウェイクアップはI2Cタッチパッドデバイスから発生していた。このデバイスはACPIテーブルでデフォルトで`wakeup`ケーパビリティが有効になっていた。

### 修正 {#fix_4}

**当面の回避策** — このデバイスのウェイクアップケーパビリティを無効にする。

```bash
# I2Cデバイスのパスを特定する
ls /sys/bus/i2c/devices/
# i2c-0  i2c-1  i2c-2  ...

# タッチパッドを見つける
grep -r "ELAN\|i2c-hid\|touchpad" /sys/bus/i2c/devices/*/name 2>/dev/null
# /sys/bus/i2c/devices/i2c-2/name: ELAN0001:00

# このデバイスのウェイクアップを無効にする
echo disabled > /sys/bus/i2c/devices/i2c-2/power/wakeup

# udevルールで再起動後も維持する
echo 'ACTION=="add", SUBSYSTEM=="i2c", ATTR{name}=="ELAN0001:00", \
  ATTR{power/wakeup}="disabled"' > /etc/udev/rules.d/99-touchpad-wakeup.rules
```

**sysfsによる回避策が実際にどう効くのか** — `drivers/hid/i2c-hid/i2c-hid-core.c`の`i2c_hid_core_suspend()`は、デバイスを低電力の`I2C_HID_PWR_SLEEP`状態にし（ドライバが`I2C_HID_QUIRK_NO_SLEEP_ON_SUSPEND` quirkを持っていない限り）、クライアントのIRQラインに対して`disable_irq()`を呼ぶ。しかし、ドライバ固有の完全な電源遮断である`i2c_hid_core_power_down()`を呼ぶのは、そのデバイスについて`device_may_wakeup()`がfalseを返す場合だけである。sysfsの`power/wakeup`でウェイクアップを無効にすることは、まさに`device_may_wakeup()`にfalseを返させることであり、それによってタッチパッドはサスペンドのたびに完全な電源断のパスを通るようになる。これこそがファームウェアレベルの偽のIRQアサートを実際に止めるものであり、単なる「何もしない」フラグではない。

**教訓**: S2idleのウェイクアップのデバッグは、常に`/sys/kernel/debug/wakeup_sources`から始まる。`wakeup_count`または`active_count`でソートして犯人を見つける。`wakeup_source_activate`のftraceイベントが正確なソースを特定する。デバイスを見つけたら、sysfsでそのウェイクアップケーパビリティを無効にするか、アイドル中にハードウェアを静かにさせるようドライバを修正する。

---

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [drivers/base/power/main.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/base/power/main.c) — `device_suspend()`のコールバック選択順序（`pm_domain` → `type` → `class` → `bus` → ドライバの`pm`）と、`dpm_run_callback()`の`if (!cb) return 0;`。PCIデバイスでは`dev->bus->pm`が常に設定されているため、この特定のスキップは決して発生しない — 事例1
- [include/linux/pm.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/pm.h) — `DEFINE_SIMPLE_DEV_PM_OPS()`（現行のマクロ。`SIMPLE_DEV_PM_OPS()`は現在非推奨だが同等）と`SET_RUNTIME_PM_OPS()`。事例1の修正
- [drivers/pci/pci-driver.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pci/pci-driver.c) — `pci_pm_suspend()`。ドライバ自身のコールバックをラップし、それでもPCIコンフィグ空間の保存とD3への遷移を行うバスレベルの`dev_pm_ops` — 事例1
- [include/linux/pm_runtime.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/pm_runtime.h)と[drivers/base/power/runtime.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/base/power/runtime.c) — `pm_runtime_get_sync()`（スリープしうる）と`pm_runtime_get_if_active()`（決してスリープせず、3値を返す — [Runtime PM](runtime-pm.md#runtime-pm-in-interrupt-context)を参照）— 事例2
- [drivers/thermal/gov_step_wise.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/thermal/gov_step_wise.c)と[drivers/thermal/gov_power_allocator.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/thermal/gov_power_allocator.c) — 1段階ずつのスロットリングロジックとPIDコントローラ — 事例3
- [drivers/powercap/intel_rapl_common.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/powercap/intel_rapl_common.c) — `rapl_domain_names[]`（`"package"`、`"core"`、`"uncore"`、`"dram"`）。事例4の背後にある独立したサブドメイン
- [drivers/base/power/wakeup.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/base/power/wakeup.c) — `print_wakeup_source_stats()`、`/sys/kernel/debug/wakeup_sources`の列の配置、および`wakeup_source_activate` tracepoint — 事例5
- [drivers/hid/i2c-hid/i2c-hid-core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/hid/i2c-hid/i2c-hid-core.c) — `i2c_hid_core_suspend()`。`device_may_wakeup()`がfalseの場合にのみ`i2c_hid_core_power_down()`でデバイスの電源を落とす — 事例5

### 関連ページ {#related-pages}

- [Runtime PM](runtime-pm.md) — 使用カウント、コールバック、アトミックコンテキストでの制約
- [システムサスペンド](suspend.md) — PM notifier、フリーズの手順、ウェイクアップソース
- [熱管理](thermal.md) — step_wiseガバナとpower_allocatorガバナ
- [パワーキャッピングとRAPL](power-capping.md) — RAPLドメインの列挙と制限

### LWNの記事 {#lwn-articles}

- [Runtime power management](https://lwn.net/Articles/347573/)（2009年8月19日） — `dev_pm_ops`への最初の`runtime_suspend`/`runtime_resume`/`runtime_idle`の追加。事例2の背景
- [The power allocator thermal governor](https://lwn.net/Articles/602517/)（2014年6月17日） — 事例3の修正で切り替え先となるPIDコントローラの設計
- [Power Capping Framework and RAPL Driver](https://lwn.net/Articles/567620/)（2013年9月19日） — 最初のpowercapフレームワークとintel-raplドライバの投稿。事例4の背後にある統一的なsysfsインターフェース

### 外部資料 {#external}

- [Runtime Power Management Framework for I/O Devices](https://docs.kernel.org/power/runtime_pm.html) — Runtime PMの正式なドキュメントと関数の戻り値のリファレンス — 事例2
- [Basic PM Debugging](https://docs.kernel.org/power/basic-pm-debugging.html) — `/sys/power/pm_test`、`CONFIG_PM_DEBUG`、および事例1の診断で使ったドライバ切り分けの手法
- [Power Capping Framework](https://docs.kernel.org/power/powercap/powercap.html) — `constraint_N_power_limit_uw`/`enabled`のsysfs ABIと、事例4の背後にあるゾーン階層
- [System Sleep States](https://docs.kernel.org/admin-guide/pm/sleep-states.html) — suspend-to-idle（`s2idle`）、`/sys/power/mem_sleep`、およびウェイクアップ可能なデバイス — 事例5
