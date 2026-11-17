# 時間サブシステムの事件簿 {#time-subsystem-war-stories}

> TSCの不安定性、clocksourceウォッチドッグ、うるう秒、タイマーのバグ

## マルチソケットシステムでのTSCドリフト {#tsc-drift-on-multi-socket-systems}

初期のx86 NUMAシステム（Nehalem以前の時代）には根本的な問題があった。各ソケットが独自の発振器を持っており、ソケット間のTSC（タイムスタンプカウンタ）は共通の基準に同期していなかった。ソケット0のTSCが、ソケット1のTSCよりわずかに速くインクリメントされることがあった。差は小さく、典型的には毎秒数十マイクロ秒程度だったが、それが蓄積していった。

その結果、ソケット0でスケジュールされたタスクがvDSO（TSCを読む）を介して`CLOCK_MONOTONIC`を読み、スケジューラによってソケット1に移動され、その後*より小さな*TSC値を読む、ということが起こりえた。`clock_gettime(CLOCK_MONOTONIC)`が逆戻りしたように見えるのである。モノトニックな時刻は実際にモノトニック（単調）であると仮定していたアプリケーション（名前からすれば妥当な仮定である）は誤動作した。

カーネルの対策は**clocksourceウォッチドッグ**（`kernel/time/clocksource.c`、`clocksource_watchdog()`）である。周期的なタイマーがTSCの読み取り値を基準のclocksource（HPETまたはACPI PMタイマー）と比較する。TSCと基準の差がしきい値を超えると、TSCは`CLOCK_SOURCE_UNSTABLE`とマークされ、カーネルは次善のclocksource（典型的にはHPET。レーティングは250で、TSCの固定レーティング300より低い）に格下げする。

現代のIntelおよびAMDのCPUは**Invariant TSC**機能を公開している。`CPUID`リーフ`0x80000007`、EDXのビット8（`TSC_INVARIANT`）である。invariant TSCは次の性質を持つ。

- CPU周波数スケーリング（Pステート）や電力状態（C1までのCステート）にかかわらず一定のレートで動作する。
- パッケージ内のすべてのコアで同期している。
- RESET同期を使うIntelプラットフォームでは、すべてのソケットで同期している。

invariant TSC機能が検出された場合でも、カーネルは依然としてウォッチドッグを必要とし、TSCに`CLOCK_SOURCE_MUST_VERIFY`をセットしたままにする。検証を完全に回避するための`X86_FEATURE_TSC_RELIABLE`フラグは、特定のプラットフォーム（Intel Goldmont SoCなど）でのみセットされる。

```bash
# invariant TSC を確認する
grep ' constant_tsc' /proc/cpuinfo
grep ' nonstop_tsc' /proc/cpuinfo

# どのクロックソースが有効かを見る
cat /sys/devices/system/clocksource/clocksource0/current_clocksource

# ウォッチドッグがTSCを不安定とマークする — dmesgでこれを探す
dmesg | grep -i 'clocksource.*unstable\|tsc.*watchdog'
```

## うるう秒によるthundering herd（2012年） {#the-leap-second-thundering-herd-2012}

2012年6月30日23:59:59 UTCに、うるう秒が挿入された。世界中のLinuxサーバーで同時にCPU飽和が発生した。原因は負荷ではなく、カーネルのバグだった。

timexのステータスに`STA_INS`がセットされていると、カーネルはUTCの午前0時の境界でうるう秒の挿入をスケジュールする。当時の実装には、`CLOCK_REALTIME`に対して`TIMER_ABSTIME`でスリープしているタスクのための`hrtimer_interrupt()`の扱いに欠陥があった。

何が起きたかは次のとおりである。

1. アプリケーション（JVM、MySQL、Javaベースのサービス）には、`CLOCK_REALTIME`の絶対時刻、たとえば「00:00:00.000に起こして」まで眠っているスレッドがあった。
2. うるう秒が挿入された。カーネルはクロックを23:59:59に余分に1秒とどめ、それから00:00:00にジャンプした。
3. スレッドの起床時刻は、カーネルから見て「過去」になった。
4. カーネルはそれらを即座に起こした。
5. スレッドは新しい絶対時刻を指定して再び`clock_nanosleep()`を呼んだ。
6. その新しい時刻もまた「過去」だった（NTPで調整されたクロックが混乱した状態にあった）。
7. スレッドは即座にまた起こされ、再びスリープし、また起こされ……と、CPUを100%消費するタイトなループで空回りした。

その結果、Reddit、Mozilla、LinkedIn、Qantas、その他何百ものLinuxベースのサービスが、世界中でまったく同じ瞬間にCPU使用率が100%に跳ね上がったと報告した。

根本原因: うるう秒を処理するhrtimerは、`clock_was_set()`を介したクロックのステップ変更を一度も通知されなかったため、`CLOCK_REALTIME`/`TIMER_ABSTIME`のタイマーが連続したループの中で即座に発火し続けた。その後カーネルには、うるう秒挿入中の`hrtimer`の挙動を修正するバックポートパッチが適用された。

**当時使われた回避策:**

- 午前0時の前にシステムクロックをわずかに進めておき、うるう秒が通常のティックとして吸収されるようにして、混乱した状態をまるごと避ける。
- `ntpd`の`leapsmear`オプション（chrony、NTPD 4.2.6以降）を使い、1秒の調整をより長い時間枠（12〜24時間）に分散させて、どの1秒にも不連続が生じないようにする。これにより空回りを引き起こした急激な不連続を防ぐ。現在では多くのNTPサーバー運用者やクラウドプロバイダがデフォルトでスミアリングを行っている。カーネル自体はスミアリングを実装しておらず、時刻デーモンが行う。
- カーネルの修正を適用する: うるう秒挿入中の`hrtimer`の挙動を修正したバックポートパッチが、影響を受けるカーネルバージョン向けに提供された。

この事件をきっかけに、単調増加する実時間クロックを必要とするアプリケーションで`CLOCK_TAI`が広く採用されるようになった。TAIにはうるう秒がなく、決してジャンプしない。

## clocksourceウォッチドッグの誤検知 {#the-clocksource-watchdog-false-positive}

KVMハイパーバイザ上の本番Kubernetesクラスタで、`clock_gettime()`が不規則な挙動を示し始めた。プロセスからは`CLOCK_MONOTONIC`の進みがあまりに遅く見え、分解能は期待される1ナノ秒ではなく1ミリ秒だった。影響を受けたゲストVMの`dmesg`には次のように表示されていた。

```
clocksource: timekeeping watchdog on CPU 0: Marking clocksource tsc unstable due to frequency skew
clocksource: Switched to clocksource jiffies
```

根本原因は、フォールバックの連鎖だった。

1. KVMゲストでは、性能上の理由からハイパーバイザでHPETが無効化されていた（ホストのカーネルコマンドラインに`no_hpet`）。HPETのレジスタはアクセスが遅いためである。
2. HPETがないため、ウォッチドッグは基準のclocksourceとして**ACPI PMタイマー**にフォールバックした。
3. ホストは過負荷だった。KVMゲストからPMタイマーのレジスタを読むにはVMEXITが必要だった。負荷がかかった状態では、ハイパーバイザはVMEXITの完了を数百マイクロ秒にわたってスケジュールしなかった。これは想定されるPMタイマーの読み取りレイテンシよりはるかに長い。
4. ウォッチドッグはTSCの差分を遅延したPMタイマーの差分と比較し、TSCは正確だったにもかかわらず、TSCが数百マイクロ秒ドリフトしたと結論づけた。
5. カーネルはTSCを`CLOCK_SOURCE_UNSTABLE`とマークし、HPET（利用不可）にフォールバックし、さらに`jiffies`にフォールバックした。その結果、分解能は1msになった。

影響を受けたゲストでは、すべての`clock_gettime()`呼び出しが粗い粒度になった。高分解能の計時に依存するアプリケーション（プロファイリング、ネットワークペーシング、HTTP/2の期限）はすべて性能が劣化した。

**修正と緩和策:**

```bash
# 影響を受けたゲストでの即時の緩和策:
echo tsc > /sys/devices/system/clocksource/clocksource0/current_clocksource

# 恒久的な対策: TSCが信頼できるとわかっているならウォッチドッグを無効にする
# カーネルコマンドラインに追加:
tsc=reliable

# より良い対策: KVMゲストが kvm-clock を使うよう設定する
# kvm-clock は準仮想化されており、steal time を正しく考慮する
grep kvm-clock /sys/devices/system/clocksource/clocksource0/available_clocksource
echo kvm-clock > /sys/devices/system/clocksource/clocksource0/current_clocksource
```

KVMゲストにとって正しいclocksourceは`kvm-clock`である。これはハイパーバイザが書き込む共有メモリページ（`pvclock`ページ）を使い、steal time（ゲストのvCPUがスケジュールされていなかった時間）を考慮する。ハイパーバイザがこれを公開している場合、KVM上で動作するカーネルはデフォルトでこれを使う。

## timer_deleteとtimer_delete_syncの競合 {#timer_delete-vs-timer_delete_sync-race}

あるネットワークドライバは、タイムアウト内に確認応答が届かなければパケットを再送するためのタイマーを登録していた。タイマーのコールバックは、ドライバの`struct device_state`のフィールドを読んでいた。ドライバの後始末の際、クリーンアップ関数は`timer_delete()`を呼び、続いて`kfree(state)`を呼んでいた。

```c
/* バグのある後始末 */
static void driver_remove(struct platform_device *pdev)
{
    struct device_state *state = platform_get_drvdata(pdev);
    timer_delete(&state->retransmit_timer);  /* バグ: 待たない */
    kfree(state);                          /* コールバックが実行中かもしれないのに解放される */
}
```

競合の流れ:

1. CPU 0: `timer_delete()`が呼ばれる。タイマーはホイール内で保留中である。`timer_delete()`はそれを取り除き、1（保留中だった）を返す。
2. CPU 1: CPU 0の`timer_delete()`が実行される前に、タイマーはすでに`__run_timers()`によってキューから外されていた。コールバックは現在実行中である。
3. CPU 0: `timer_delete()`が戻る。`kfree(state)`が呼ばれる。`state`が解放される。
4. CPU 1: タイマーのコールバックは実行を続け、すでに解放された`state`のフィールドを読む。use-after-freeである。

`timer_delete()`はタイマーをホイールから取り除くが、現在実行中のコールバックが終了したことは何も保証しない。`timer_delete_sync()`は、どのCPU上でもコールバックが実行されていない状態になるまでスピンしてから戻る。

**修正:**

```c
/* 正しい後始末 */
static void driver_remove(struct platform_device *pdev)
{
    struct device_state *state = platform_get_drvdata(pdev);
    timer_delete_sync(&state->retransmit_timer);  /* コールバックを待つ */
    kfree(state);                               /* 安全 */
}
```

Linux 6.2以降、後始末の後にタイマーを二度と発火させてはならない場合は、`timer_shutdown_sync()`が推奨される関数である。

```c
timer_shutdown_sync(&state->retransmit_timer);
/* 以降の add_timer() / mod_timer() 呼び出しは黙って無視される */
kfree(state);
```

これにより、`timer_delete_sync()`が戻る前にコールバックがタイマーを再設定してしまう再設定の競合を防げる。`timer_shutdown_sync()`はタイマーを恒久的に非アクティブとしてマークする。

LOCKDEPは、softirqコンテキストから呼ばれた`timer_delete_sync()`について警告する（待つためにスリープできないため）。その場合は`timer_delete()`を使って`kfree()`をworkqueueに遅延させるか、RCUを使ってデータ構造を保護すること。

## 32ビットjiffiesのラップアラウンド {#the-32-bit-jiffies-wraparound}

あるストレージドライバは、コマンドのタイムアウトを次のように実装していた。

```c
struct command {
    unsigned long submit_jiffies;
    /* ... */
};

/* 投入時: */
cmd->submit_jiffies = jiffies;

/* 完了パスで: */
unsigned long elapsed = jiffies - cmd->submit_jiffies;
if (elapsed > msecs_to_jiffies(TIMEOUT_MS)) {
    report_timeout(cmd);
}
```

このコードは正しいように見え、テストにも合格した。HZ=1000の本番システムでは49日間は確実に動作したが、その後、コマンドが誤ってタイムアウトしたと報告されるようになった。

HZ=1000では、`jiffies`は約49.7日後に`ULONG_MAX`（32ビットカーネルでは0xFFFFFFFF、64ビットカーネルでもドライバが`u32`にキャストしていれば`jiffies`の下位32ビット）から0に戻る。

ラップアラウンド後: `cmd->submit_jiffies`は大きな数（例: 0xFFFF0000）である。現在の`jiffies`は小さな数（例: 0x00001000）である。減算`jiffies - cmd->submit_jiffies`は符号なしラップアラウンドによって巨大な正の数を生み、誤って非常に長い経過時間に見えてしまう。

このバグの2つ目の変種は、生の比較を使うものである。

```c
/* これも誤り: */
if (jiffies > cmd->submit_jiffies + msecs_to_jiffies(TIMEOUT_MS)) {
```

ラップアラウンド後、`cmd->submit_jiffies + timeout`はラップして小さな数になり、`jiffies > small_number`は即座に真になる。

**修正: 常にラップアラウンドに安全なマクロを使うこと。**

```c
/* 正しい経過時間チェック */
if (time_after(jiffies, cmd->submit_jiffies + msecs_to_jiffies(TIMEOUT_MS))) {
    report_timeout(cmd);
}

/* time_after(a, b) は減算を unsigned long の幅で行い、
 * その結果を long にキャストする: (long)((b) - (a)) < 0
 * 符号なしの結果を signed long として解釈することで、LONG_MAX までの
 * ラップアラウンドした差を正しく扱える。このマクロは、間隔が
 * LONG_MAX/HZ 秒未満（32ビット・HZ=1000のシステムで約24日）で
 * あることを前提とする。 */
```

`include/linux/jiffies.h`にあるマクロの一覧:

| マクロ | 意味 |
|-------|---------|
| `time_after(a, b)` | `a`は`b`より後（a > b、ラップアラウンドに安全） |
| `time_before(a, b)` | `a`は`b`より前 |
| `time_after_eq(a, b)` | `a`は`b`より後または等しい |
| `time_before_eq(a, b)` | `a`は`b`より前または等しい |
| `time_in_range(a, b, c)` | `a`は`[b, c]`の範囲内 |

64ビットカーネルでは、`jiffies`は64ビット値であり、`jiffies_64`が完全な64ビットカウンタを直接提供する。オーバーフローには何億年もかかるため問題にはならない。しかし、ラップアラウンドに安全なマクロは依然として正しく、明確さと移植性のためにいずれにせよ使うべきである。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/time/clocksource.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/clocksource.c) — `clocksource_watchdog()`による安定性検証、`CLOCK_SOURCE_UNSTABLE`のマーク、リトライしきい値のロジック
- [kernel/time/timekeeping.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/timekeeping.c) — `second_overflow()`とうるう秒の状態遷移。タイマー再設定のループを防ぐ
- [kernel/time/timer.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/timer.c) — 並行実行と再設定の競合を防ぐ`timer_delete_sync()`と`timer_shutdown_sync()`の実装
- [include/linux/jiffies.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/jiffies.h) — `time_after()`、`time_before()`、ラップアラウンドに安全な算術
- [arch/x86/kernel/tsc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/tsc.c) — `tsc_init()`、invariant TSCのCPUID検証、`mark_tsc_unstable()`
- [arch/x86/kernel/kvmclock.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/kvmclock.c) — ゲストの時刻管理のための、準仮想化された`kvm-clock` clocksourceの統合

### manページ {#man-pages}

- [`clock_gettime(2)`](https://man7.org/linux/man-pages/man2/clock_gettime.2.html) — モノトニッククロックとリアルタイムクロックの問い合わせのセマンティクス
- [`adjtimex(2)`](https://man7.org/linux/man-pages/man2/adjtimex.2.html) — うるう秒挿入のフラグ（`STA_INS`、`STA_DEL`）と状態の報告

### 関連ページ {#related-pages}

- [clocksourceとclockeventのドライバ](clocksource.md) — ハードウェアclocksource、ウォッチドッグによる検証、レーティングの階層
- [時刻管理とクロックソース](timekeeping.md) — 時刻管理のアーキテクチャ、うるう秒、vDSOの高速パス
- [タイマーホイール](timer-wheel.md) — タイマーのライフサイクル、`timer_delete_sync()`、jiffiesの算術
- [NTPとクロック規律](ntp.md) — PLLによる同期、うるう秒、chrony/ntpdのスミアリング

### LWN記事 {#lwn-articles}

- [The leap second bug](https://lwn.net/Articles/504657/) — Jonathan Corbet、2012年7月: 2012年のうるう秒によるhrtimerのライブロックの詳細な事後分析
- [The trouble with the TSC](https://lwn.net/Articles/388188/) — Jake Edge、2010年5月: invariant TSCの保証、周波数スケーリングによるドリフト、ウォッチドッグによる検証
- [Reinventing the timer wheel](https://lwn.net/Articles/646950/) — Jonathan Corbet、2015年6月: タイマーホイールの再設計とタイマーキャンセルの保証の改善

### 外部資料 {#external}

- [Timekeeping and Timers in Linux](https://docs.kernel.org/core-api/timekeeping.html) — 時刻管理とタイマーのサブシステムに関するコアカーネルドキュメント
- [The kernel's command-line parameters](https://docs.kernel.org/admin-guide/kernel-parameters.html) — `tsc=reliable`、`clocksource=`、`no_hpet`パラメータのリファレンス
