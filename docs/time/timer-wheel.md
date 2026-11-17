# タイマーホイール {#the-timer-wheel}

> jiffies、struct timer_list、そして低分解能タイマーのための階層型タイマーホイール

## 2つのタイマーシステム {#two-timer-systems}

Linuxは、異なるユースケースに対応する2つの別個のタイマー実装を保持している。

| システム | 精度 | データ構造 | 挿入コスト | 典型的な用途 |
|--------|-----------|---------------|-------------|-------------|
| **hrtimer** | 約ナノ秒 | CPUごとの赤黒木 | O(log n) | `sleep()`、オーディオ、ネットワークペーシング、リアルタイム |
| **timer_list**（タイマーホイール） | 約1/HZ（1〜10ms） | CPUごとの階層型ホイール | O(1) | TCP再送、ウォッチドッグ、遅延実行される処理 |

hrtimerは正確なウェイクアップのためにハードウェアのclockeventを直接プログラムする。タイマーホイールは、満了前に頻繁にキャンセルされるタイムアウト（例: ACKが届いた時点でキャンセルされるTCP再送タイマー）向けに設計されている。O(1)の挿入・キャンセルコストこそが主な利点であり、timer_listのタイマーのほとんどは実際には一度も発火しない。

## jiffiesとHZ {#jiffies-and-hz}

`jiffies`は、ハードウェアのティックごとに1つずつインクリメントされるグローバルなカウンタである。これはタイマーホイールにおける時間の基本単位である。

```c
/* include/linux/jiffies.h */
extern unsigned long volatile jiffies;  /* ULONG_MAX でラップする */
extern u64 jiffies_64;                  /* 64ビット、実用上ラップアラウンドしない */
```

`HZ`は1秒あたりのjiffies数であり、コンパイル時に設定される。

| CONFIG_HZ | ティック周期 | 典型的な用途 |
|-----------|-------------|-------------|
| 100 | 10ms | サーバー、組み込み（CONFIG_HZ_100） |
| 250 | 4ms | 多くのディストリビューションでのデスクトップのデフォルト |
| 1000 | 1ms | 低レイテンシのデスクトップ、リアルタイム |

変換ヘルパー:

```c
msecs_to_jiffies(500)       /* ms → jiffies。HZ非依存のコードで安全 */
jiffies_to_msecs(jiffies)   /* jiffies → ms */
usecs_to_jiffies(100)       /* us → jiffies（切り上げ） */
nsecs_to_jiffies(1000000UL) /* ns → jiffies */
```

`HZ * seconds`を直接計算するのではなく、常にこれらのヘルパーを使うこと。ヘルパーは丸めを正しく扱い、HZの設定をまたいで移植性がある。

## struct timer_list {#struct-timer_list}

```c
/* include/linux/timer_types.h */
struct timer_list {
    struct hlist_node   entry;    /* ホイールのバケット内のリンク */
    unsigned long       expires;  /* jiffies単位の絶対満了時刻 */
    void              (*function)(struct timer_list *); /* コールバック */
    u32                 flags;    /* エンコードされたCPUアフィニティ + TIMER_* フラグ */
};
```

主なフィールド:

- `expires` — タイマーが発火すべき絶対jiffies値。`add_timer()`または`mod_timer()`で設定する。
- `function` — タイマーが固定されたCPU上で、softirqコンテキスト（TIMER_SOFTIRQ）から呼ばれる。スリープしてはならない。
- `flags` — 上位ビットに対象CPUを、下位ビットにタイマーフラグをエンコードする。

タイマーの初期化:

```c
struct timer_list my_timer;

/* 静的初期化 */
DEFINE_TIMER(my_timer, my_callback);

/* 動的初期化 */
timer_setup(&my_timer, my_callback, 0);
```

`timer_setup()`は、古い`init_timer()`+`setup_timer()`のパターンを置き換えた（Linux 4.14で変更）。コールバックは、不透明な`unsigned long`のデータ引数ではなく、`struct timer_list *`を直接受け取るようになった。

## 公開API {#the-public-api}

### add_timer() {#add_timer}

```c
void add_timer(struct timer_list *timer);
```

タイマーをホイールに挿入する。呼び出す前に`timer->expires`を設定しておかなければならない。タイマーが`expires`より早く発火することはないが、CPUがビジーであればわずかに遅れて発火することがある。

```c
timer_setup(&my_timer, my_callback, 0);
my_timer.expires = jiffies + msecs_to_jiffies(500);
add_timer(&my_timer);
```

### mod_timer() {#mod_timer}

```c
int mod_timer(struct timer_list *timer, unsigned long expires);
```

保留中のタイマーの満了時刻をアトミックに変更する。タイマーが保留中でなければ追加される。タイマーが保留中だった場合は1を、そうでなければ0を返す。1回の呼び出しでタイマーの追加と変更の両方を行う推奨の方法である。

### timer_reduce() {#timer_reduce}

```c
int timer_reduce(struct timer_list *timer, unsigned long expires);
```

`mod_timer()`と同様だが、満了時刻を*早める*方向にしか動かさない。タイマーがすでに`expires`以前に満了するよう設定されていれば、この呼び出しは何もしない。Linux 4.15で追加された。これにより、より早く発火するようすでにスケジュールされていたタイマーを、ドライバが誤って遅らせることを防げる。複数のコードパスが同じタイマーを設定しうる場合に有用である。

### timer_delete()とtimer_delete_sync() {#timer_delete-vs-timer_delete_sync}

```c
int timer_delete(struct timer_list *timer);
int timer_delete_sync(struct timer_list *timer);
```

`timer_delete()`はタイマーをホイールから取り除く。コールバックが別のCPU上で実行中の場合、`timer_delete()`は即座に戻り、コールバックは実行を続ける。`timer_delete()`の直後にコールバックが触るデータを解放すると、use-after-freeを引き起こしうる。

`timer_delete_sync()`は、すべてのCPU上でコールバックが終了するまで待ってから戻る。タイマーのコールバックがアクセスする資源を解放する前には、常に`timer_delete_sync()`を使うこと。

```c
/* 安全な後始末 */
timer_delete_sync(&my_timer);
kfree(my_data);   /* 安全: コールバックは完了している */
```

Linux 6.2以降、`timer_shutdown_sync()`はタイマーを停止した後、恒久的に無効なものとしてマークするため、以降の`add_timer()`や`mod_timer()`の呼び出しは黙って無視される。タイマーを二度と発火させてはならない場合（例: ドライバの削除中）にこれを使う。

## 階層型ホイール（Linux 4.8以降） {#the-hierarchical-wheel-linux-48}

Linux 4.8は、古い5レベルのカスケード型ホイールを階層型ホイールの設計（`kernel/time/timer.c`）に置き換えた。古い設計では、ホイールの針がレベルの境界を越えて進むたびに高コストなカスケード処理が必要だった。新しい設計は、より単純でキャッシュに優しい構造を使う。

### struct timer_base {#struct-timer_base}

```c
/* kernel/time/timer.c（内部） */
struct timer_base {
    raw_spinlock_t      lock;
    struct timer_list  *running_timer;   /* 現在実行中のコールバック */
#ifdef CONFIG_PREEMPT_RT
    spinlock_t          expiry_lock;
    atomic_t            timer_waiters;
#endif
    unsigned long       clk;            /* 最後に処理したjiffies値 */
    unsigned long       next_expiry;    /* 保留中の最も早い満了時刻 */
    unsigned int        cpu;
    bool                next_expiry_recalc;
    bool                is_idle;
    bool                timers_pending;
    DECLARE_BITMAP(pending_map, WHEEL_SIZE);
    struct hlist_head   vectors[WHEEL_SIZE]; /* バケット群 */
} ____cacheline_aligned;
```

`CONFIG_NO_HZ_COMMON`を有効にしたSMPシステムでは、各CPUが3つの`timer_base`インスタンスを持つ（`NR_BASES = 3`）。`BASE_LOCAL`（固定されたタイマー用）、`BASE_GLOBAL`（固定されていない／移動可能なタイマー用）、`BASE_DEF`（遅延可能タイマー用）である。

### ホイールの形状 {#wheel-geometry}

レベル数は設定された`HZ`に依存する。

- **HZ > 100**（例: HZ=250またはHZ=1000。デスクトップ/サーバー向けカーネルのデフォルト）: `LVL_DEPTH = 9`で、**9 × 64 = 576バケット**となる。
- **HZ ≤ 100**（例: 組み込み/低消費電力向けカーネル）: `LVL_DEPTH = 8`で、**8 × 64 = 512バケット**となる。

次の表は、HZ=250（`LVL_DEPTH = 9`）における形状を示す。各レベルは64バケット（`LVL_SIZE = 64`）を持つ。

| レベル | 粒度 | カバーする範囲 |
|-------|-------------|---------------|
| 0 | 1 jiffy | 0 – 63 jiffies |
| 1 | 8 jiffies | 64 – 511 jiffies |
| 2 | 64 jiffies | 512 – 4,095 jiffies |
| 3 | 512 jiffies | 4,096 – 32,767 jiffies |
| 4 | 4,096 jiffies | 32,768 – 262,143 jiffies |
| 5 | 32,768 jiffies | 262,144 – 2,097,151 jiffies |
| 6 | 262,144 jiffies | 2,097,152 – 16,777,215 jiffies |
| 7 | 2,097,152 jiffies | 16,777,216 – 134,217,727 jiffies |
| 8 | 16,777,216 jiffies | HZ=250で最大約4,294,967,295 jiffies（約17,179,869秒、約198.8日） |

各レベルは1つ前のレベルより8倍粗い。タイマーが挿入されると、カーネルは現在時刻と`expires`の差に基づいて、それが属するレベルとバケットを計算する。これは純粋なビットマスク演算であり、どの範囲でも挿入はO(1)である。

ホイール全体の容量: HZ=250では576バケット（9 × 64）。`pending_map`ビットマップにより、カーネルはすべてのエントリを走査することなく、次の空でないバケットをすばやく見つけられる。

### __run_timers() {#__run_timers}

```c
static void __run_timers(struct timer_base *base);
```

ティックごとに、`run_timer_softirq()`（TIMER_SOFTIRQのハンドラ）から`run_timer_softirq()` → `run_timer_base()` → `__run_timer_base()` → `__run_timers()`という呼び出しの連鎖を経て間接的に呼ばれる。`base->clk`を現在のjiffies値まで進め、期限に達したすべてのバケットを特定し、それらのコールバックを実行する。`CONFIG_NO_HZ_COMMON`を有効にしたSMPシステムでは、`run_timer_softirq()`は`BASE_LOCAL`、`BASE_GLOBAL`、`BASE_DEF`を処理し、タイマー移行の階層が有効なときは`tmigr_handle_remote()`を呼び出す。

softirqはティック割り込みハンドラによって発生させられる。NOHZ（ティックレス）システムでは、CPUがアイドルのときティックが完全にスキップされることがあり、CPUが起床したときに`__run_timers()`が遅れを取り戻す。

## 遅延可能タイマー {#deferrable-timers}

```c
timer_setup(&my_timer, my_callback, TIMER_DEFERRABLE);
```

遅延可能（deferrable）タイマーは`BASE_DEF`（遅延可能ベース）に格納される。これらのタイマーはCPUがすでに動作しているときにのみ満了し、アイドル状態のCPUを起こさない。満了時刻を過ぎてもCPUがアイドルのままであれば、タイマーは何らかの理由でCPUが次に起床したときに発火する。

遅延可能タイマーは、時間的にクリティカルでない周期的な保守作業（例: 統計の収集、バッファのフラッシュ）に使う。これらを使うと不要なウェイクアップが減り、NOHZシステムでの電力効率が向上する。

## workqueueとの関係 {#relationship-to-workqueues}

`schedule_delayed_work()`は、タイマーホイールとworkqueueの仕組みを組み合わせたものである。

```c
/* 内部的には: */
struct delayed_work {
    struct work_struct  work;
    struct timer_list   timer;  /* 遅延時間の後に発火する */
    /* ... */
};

schedule_delayed_work(&my_dwork, msecs_to_jiffies(100));
```

タイマーが発火すると、そのコールバックは`queue_work()`を呼び出して`work`をシステムworkqueueに投入する。実際の処理はsoftirqコンテキストではなく、workqueueのスレッド（プロセスコンテキストで、スリープ可能）で実行される。遅延させる処理がスリープしたりメモリを割り当てたりする必要がある場合は`schedule_delayed_work()`を使い、コールバックが短くスリープできない場合にのみ素の`timer_list`を使うこと。

## タイマーの観察 {#observing-timers}

保留中のタイマーホイールのエントリを調べるための直接的なprocfsインターフェースはない。代わりにカーネルのトレーシングを使う。

```bash
# ftraceで __run_timers の発火をトレースする
echo '__run_timers' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
cat /sys/kernel/debug/tracing/trace_pipe

# bpftraceでタイマーコールバックを関数ごとに数える
bpftrace -e '
  kprobe:call_timer_fn {
    @[ksym(arg1)] = count();
  }
  interval:s:5 { print(@); clear(@); exit(); }
'

# CPUごとのタイマーsoftirqの回数を表示する
grep TIMER /proc/softirqs

# 概観: タイマー満了レイテンシの分布を見る
perf stat -e irq:softirq_entry,irq:softirq_exit -a -- sleep 5
```

## よくある落とし穴 {#common-pitfalls}

### jiffiesのラップアラウンド {#jiffies-wraparound}

`jiffies`は`unsigned long`であり、`ULONG_MAX`でラップする。HZ=1000の32ビットカーネルでは、これは約49日ごとに起きる。jiffiesを生の`>`や`<`で比較してはならない。

```c
/* 誤り: ラップアラウンド時に壊れる */
if (jiffies > my_timer.expires) { ... }

/* 正しい: ラップアラウンドに安全なマクロを使う */
if (time_after(jiffies, my_timer.expires)) { ... }
if (time_before(jiffies, deadline)) { ... }
if (time_after_eq(jiffies, start + HZ)) { ... }
```

`time_after(a, b)`は以下のように定義されている。減算はまず`unsigned long`の幅で行われ、その結果が`long`にキャストされる。これは各オペランドを個別にキャストするのとは異なる。

```c
/* include/linux/jiffies.h */
/* 減算は unsigned long の幅で行われ、結果が long にキャストされる */
#define time_after(a,b)     \
    (typecheck(unsigned long, a) && \
     typecheck(unsigned long, b) && \
     ((long)((b) - (a)) < 0))
```

符号なし減算の結果を符号付きの値として解釈することで、符号なしのラップアラウンドを正しく扱える。

### タイマーコールバック内でのスリープ {#sleeping-in-a-timer-callback}

タイマーコールバックはsoftirqコンテキストで実行される。スリープしたり、`schedule()`を呼んだり、スリープしうるロック（例: mutex）を取得したりしてはならない。遅延させる処理がスリープする必要がある場合は`schedule_delayed_work()`を使うこと。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/timer_types.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/timer_types.h) — `struct timer_list`の公開API: `timer_setup()`、`add_timer()`、`mod_timer()`、`timer_reduce()`、`timer_delete_sync()`、`timer_shutdown_sync()`
- [include/linux/jiffies.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/jiffies.h) — `jiffies`カウンタ、時間変換ヘルパー（`msecs_to_jiffies()`）、ラップアラウンドに安全な比較マクロ（`time_after()`、`time_before()`）
- [kernel/time/timer.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/timer.c) — 階層型タイマーホイールの実装、`__run_timers()`、レベルとバケットの計算、softirqでの実行
- [kernel/time/timer_migration.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/timer_migration.c) — アイドルなCPUクラスタをまたいでグローバルタイマーを管理する、階層型タイマーのpull型移行（`tmigr_handle_remote()`）

### manページ {#man-pages}

- [`time(7)`](https://man7.org/linux/man-pages/man7/time.7.html) — 時間の概念、jiffies、カーネルタイマーの粒度の概要

### 関連ページ {#related-pages}

- [高分解能タイマー（hrtimers）](hrtimers.md) — ナノ秒精度の赤黒木タイマーと粗いタイマーホイールの比較
- [clocksourceとclockeventのドライバ](clocksource.md) — ハードウェアによるティック生成とclockevent割り込みの処理
- [時間サブシステムの事件簿](war-stories.md) — 実世界での後始末時のuse-after-free競合と、32ビットjiffiesのオーバーフローのバグ
- [workqueue: 並行性管理workqueue](../interrupts/workqueues.md) — タイマーホイールとプロセスコンテキストのworkqueueを組み合わせる`delayed_work`の仕組み

### LWN記事 {#lwn-articles}

- [Reinventing the timer wheel](https://lwn.net/Articles/646950/) — Jonathan Corbet、2015年6月: カスケード型ホイールを非カスケードの階層に置き換えた、Thomas Gleixnerによる再設計
- [The high-resolution timer API](https://lwn.net/Articles/167897/) — Jonathan Corbet、2006年1月: 従来型タイマーホイールと並ぶhrtimerサブシステムの設計と統合

### 外部資料 {#external}

- [Timekeeping and Timers in Linux](https://docs.kernel.org/core-api/timekeeping.html) — タイマーシステムと時刻管理に関する公式カーネルガイド
- [Timers Subsystem Documentation](https://docs.kernel.org/timers/index.html) — タイマーホイールの内部、高分解能タイマー、NO_HZ動作を扱うカーネルドキュメント
