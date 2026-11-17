# NTPとクロック規律 {#ntp-and-clock-discipline}

> カーネルはどのようにクロックを調整するか: adjtimex、PLL/FLL、TAIオフセット

## 問題 {#the-problem}

ハードウェアの発振器はドリフトする。補正しなければ、システムクロックはUTCから1日あたり数秒ずれていく。クロック規律（clock discipline）の目的は、クロックを逆方向にステップさせたり、アプリケーションを混乱させる不連続を生じさせたりすることなく、外部の基準（GPS、NTPサーバー、PPS信号）に向けてクロックを継続的かつ滑らかに導くことである。

Linuxカーネルはこれを`kernel/time/ntp.c`の制御ループで実装しており、ユーザー空間のデーモン（`ntpd`、`chrony`）からの`adjtimex()`呼び出しによって駆動される。

## PLLモードとFLLモード {#pll-and-fll-modes}

カーネルは2つの調整戦略をサポートする。

**位相同期ループ（PLL）** — デーモンはローカル時刻と基準との間のオフセットを測定し、オフセットがゼロに向かうよう、小さく継続的な周波数調整を行うようカーネルに依頼する。補正量は測定されたオフセットに比例し、適用前に`±MAXPHASE`（500ms）にクランプされる（`ntp_update_offset()`、`kernel/time/ntp.c`）。このクランプはどちらのモードが有効でも行われるものであり、PLLとFLLのどちらを選ぶかを決めるものではない。これはよく同期したクロックにおける定常状態のモードであり、更新が十分頻繁に（互いに`MAXSEC`以内の間隔で）届き、かつ`STA_FLL`がセットされていない場合に常に使われる。

**周波数同期ループ（FLL）** — オフセットが大きいとき、またはクロックが長い間同期されていなかったときに使われる。カーネルは、オフセットそのものではなくオフセットが変化する速さに基づいて、より積極的に周波数を調整する。カーネルがFLL補正（`ntp_update_offset_fll()`）に切り替わるのは、デーモンが明示的に`STA_FLL`ステータスフラグをセットした場合か、前回の更新から時間（`MAXSEC`）が経ちすぎて純粋なPLL補正が意味をなさなくなった場合である。

オフセットの規律そのものを有効にするのは`STA_PLL`である。これがセットされていなければ`ntp_update_offset()`はただちに戻る。`STA_FLL`はそれとは独立には機能しない。`STA_PLL`と一緒にセットされると、更新間隔が十分長い場合に補正をFLL寄りにする（後述の`ntp_update_offset_fll()`の`MINSEC`/`MAXSEC`チェックを参照）。デーモンは`adjtimex()`を呼ぶ際に`timex.status`でこれらのビットを設定する。

## adjtimex() {#adjtimex}

`adjtimex()`はカーネルの時刻規律インターフェースである。システムコールのエントリポイント（`SYSCALL_DEFINE1(adjtimex, ...)`）とその下にある`do_adjtimex()`は、それぞれ`kernel/time/time.c`と`kernel/time/timekeeping.c`にある。`kernel/time/ntp.c`は、`do_adjtimex()`が呼び出すPLL/FLLの計算そのもの（`ntp_adjtimex()`、`ntp_update_frequency()`）を保持している。

```c
int adjtimex(struct timex *txc);
```

これはglibc向けのプロトタイプである。実際のシステムコール（`SYSCALL_DEFINE1(adjtimex, struct __kernel_timex __user *, txc_p)`）は`struct __kernel_timex`を受け取る。これは、`include/uapi/linux/timex.h`が、カーネルから`struct timex`を隠す`#ifndef __KERNEL__`ブロックの外で定義している、アーキテクチャ非依存の別のレイアウトである。単に名前を変えたコピーではない。glibcの`struct timex`で`__kernel_long_t`（アーキテクチャ依存の幅）だったすべてのフィールドは、ここでは固定の`long long`になり、32ビットと64ビットのビルドで同じようにアラインされるよう明示的な`int :32`パディングが追加されている。`modes`、`status`、`shift`、`tai`は素の`int`のまま変わらない。また、埋め込まれた`struct __kernel_timex_timeval`は64ビットでY2038に安全な`tv_sec`を使う。glibcのラッパーは、システムコールを発行する前に自身の`struct timex`をこの形に変換する。

`struct timex`（`include/uapi/linux/timex.h`で定義）:

```c
struct timex {
    unsigned int modes;          /* ADJ_OFFSET, ADJ_FREQUENCY, ADJ_MAXERROR,
                                    ADJ_ESTERROR, ADJ_STATUS, ADJ_TIMECONST,
                                    ADJ_TAI, ADJ_SETOFFSET, ADJ_NANO, ... */
    __kernel_long_t offset;      /* 時刻オフセット（STA_NANO がセットされていればns、そうでなければus） */
    __kernel_long_t freq;        /* 周波数オフセット（スケーリングされたppm: ppm * 2^16） */
    __kernel_long_t maxerror;    /* 最大誤差の推定値（us） */
    __kernel_long_t esterror;    /* 推定誤差（us） */
    int  status;                 /* STA_PLL, STA_FLL, STA_NANO, STA_UNSYNC,
                                    STA_INS, STA_DEL, STA_PPSFREQ, STA_PPSTIME,
                                    STA_PPSJITTER, STA_PPSWANDER, STA_PPSERROR,
                                    STA_CLOCKERR, ... */
    __kernel_long_t constant;    /* PLLの時定数（ポーリング間隔のlog2） */
    __kernel_long_t precision;   /* クロック精度（us、読み取り専用） */
    __kernel_long_t tolerance;   /* クロック周波数の許容誤差（読み取り専用） */
    struct timeval time;         /* 現在時刻（ADJ_SETOFFSET の場合を除き読み取り専用） */
    __kernel_long_t tick;        /* クロックティック間のus */
    __kernel_long_t ppsfreq;     /* PPS周波数（読み取り専用、スケーリングされたppm） */
    __kernel_long_t jitter;      /* PPSジッタ（読み取り専用、nsまたはus） */
    int  shift;                  /* PPS間隔の長さ（秒、読み取り専用） */
    __kernel_long_t stabil;      /* PPS安定度（読み取り専用、スケーリングされたppm） */
    __kernel_long_t jitcnt;      /* PPSジッタが上限を超えた回数（読み取り専用） */
    __kernel_long_t calcnt;      /* PPS較正間隔（読み取り専用） */
    __kernel_long_t errcnt;      /* PPS較正エラー（読み取り専用） */
    __kernel_long_t stbcnt;      /* PPS安定度が上限を超えた回数（読み取り専用） */
    int  tai;                    /* 秒単位の TAI - UTC オフセット（modes に ADJ_TAI が
                                    セットされていない限り読み取り専用） */

    int :32; int :32; int :32; int :32;  /* 将来の使用のための予約パディング */
    int :32; int :32; int :32; int :32;
    int :32; int :32; int :32;
};
```

重要な`modes`フラグ:

| フラグ | 効果 |
|------|--------|
| `ADJ_OFFSET` | `offset`を位相補正として適用する |
| `ADJ_FREQUENCY` | `freq`（ppm × 2^16単位の周波数オフセット）を設定する |
| `ADJ_STATUS` | `status`フラグ（STA_PLL、STA_FLLなど）を更新する |
| `ADJ_TIMECONST` | `constant`（PLLの帯域幅。収束速度に影響する）を設定する |
| `ADJ_TAI` | TAI − UTCオフセットを設定する |
| `ADJ_SETOFFSET` | 現在時刻に（`offset`ではなく）`time`を加えてクロックをステップさせる（大きな補正のためにchronyが使う） |
| `ADJ_NANO` | `offset`をナノ秒単位として解釈する（そうでなければマイクロ秒） |

### clock_adjtime() {#clock_adjtime}

`clock_adjtime()`は、`clockid_t`を受け取る、現代的でPOSIX互換の変種である。

```c
int clock_adjtime(clockid_t clk_id, struct timex *txc);
```

`chrony`は古い`adjtimex()`よりも`clock_adjtime(CLOCK_REALTIME, ...)`を好んで使う。`CLOCK_REALTIME`については両者は等価である（`kernel/time/posix-timers.c`の`do_clock_adjtime()`は、`adjtimex()`が到達するのと同じ`.clock_adj`ハンドラにディスパッチするだけである）。ただし、`clock_adjtime()`はこれを`CLOCK_TAI`には拡張しない。`clock_tai`の`k_clock`構造体には`.clock_adj`ハンドラがまったくないため、`CLOCK_TAI`を指定して呼ぶと`-EOPNOTSUPP`が返る。TAIオフセットは、後述のように`CLOCK_REALTIME`と`ADJ_TAI`を使って設定する。

## NTPの状態機械 {#the-ntp-state-machine}

`struct timex`の`status`フィールドは状態機械を駆動する。

```
STA_UNSYNC
  │
  │  daemon calls adjtimex() with STA_PLL set and a valid offset
  ▼
STA_PLL          ← steady state: continuous frequency steering
  │
  │  PPS signal available and stable
  ▼
STA_PPSFREQ | STA_PPSTIME   ← PPS-disciplined (highest accuracy)
```

`STA_UNSYNC`は、最近有効な時刻更新を受け取っていないときにカーネルがセットする。ユーザー空間のデーモンは、基準へのロックを開始する際に`STA_UNSYNC`をクリアする。`adjtimex()`の戻り値は、現在の同期状態を`TIME_OK`、`TIME_INS`、`TIME_DEL`、`TIME_OOP`、`TIME_WAIT`、`TIME_ERROR`のいずれかとして報告する。

## ntp_tick_length() {#ntp_tick_length}

ティックごとに、`timekeeping_adjust()`と`__timekeeping_advance()`（いずれも`kernel/time/timekeeping.c`）がNTPのコードを呼び出し、適用する補正を取得する。

```c
u64 ntp_tick_length(unsigned int tkid);
```

この関数はtimekeeperのID（`tkid`）を受け取り、現在のPLL/FLL周波数補正を組み込んだ、加算すべきティック長を返す。ただし素のナノ秒ではない。値は`NTP_SCALE_SHIFT`（32ビット）だけ左にスケーリングされており、つまり`ns << 32`である。これは`struct timekeeper`自身の`ntp_tick`/`ntp_error`フィールドが使うのと同じ固定小数点表現である（`tkr_mono.xtime_nsec`アキュムレータは、代わりにclocksource固有の別の`shift`を使う）。スケーリング前の基本値は`NSEC_PER_SEC / HZ`である。PLL/FLL補正（`ntp_update_frequency()`、`kernel/time/ntp.c`）は、システム管理者や同期デーモンが周波数を調整するたびに、この値をわずかに上下させる。

## うるう秒 {#leap-seconds}

うるう秒とは、UTCをUT1（地球の自転）に合わせ続けるために、UTCの1日の終わりに挿入（または削除）される1秒の調整である。カーネルは`adjtimex()`のステータスフラグを通じてうるう秒を扱う。

| フラグ | 意味 |
|------|---------|
| `STA_INS` | 次のUTC午前0時の切り替わりでうるう秒を挿入する |
| `STA_DEL` | 次のUTC午前0時の切り替わりでうるう秒を削除する |

`STA_INS`がセットされていると、UTC午前0時の境界でカーネル（`kernel/time/ntp.c`の`second_overflow()`）は次のことを行う。

1. `xtime.tv_sec`を23:59:59の値に2秒間とどめ（「23:59:60」のうるう秒）、`tai_offset`を1増やす。TAIは常にこのオフセットだけUTCより進んでいる（2024年時点で37秒）。
2. 内部の`time_state`を`TIME_INS` → `TIME_OOP` → `TIME_WAIT`と遷移させる。これは`adjtimex()`の戻り値に反映されるため、同期デーモンはうるう秒が起きていることを観測できる。
3. `STA_INS`を自分ではクリア*しない*。このビットは、うるう秒を要求したデーモンが別の`adjtimex()`呼び出しで明示的にクリアするまでセットされたままである。カーネルが`TIME_OK`に戻るのは、`STA_INS`/`STA_DEL`がすでにクリアされている場合だけである。

`CLOCK_TAI`はTAIクロックを読む。TAIクロックにはうるう秒がなく、秒を単調にカウントする。単調増加する実時間クロックを必要とするアプリケーション（例: 金融システム）は、`CLOCK_REALTIME`ではなく`CLOCK_TAI`を使うべきである。

!!! warning "うるう秒のバグ"
    うるう秒の挿入時にカーネルのバグによって世界中のLinuxサーバーでCPU飽和が起きた2012年のうるう秒事件については、[事件簿](war-stories.md#the-leap-second-thundering-herd-2012)のページを参照。

## TAIオフセット {#tai-offset}

TAI（国際原子時）は、これまでに挿入されたうるう秒の累計に等しい整数秒だけUTCより進んでいる。2024年時点で、TAI − UTC = 37秒である。

TAIの読み取り:

```c
struct timespec64 ts;
clock_gettime(CLOCK_TAI, &ts);
/* ts.tv_sec = CLOCK_REALTIME + tai_offset */
```

TAIオフセットの設定（アプリケーションではなく時刻デーモンが行う）:

```c
struct timex txc = {
    .modes = ADJ_TAI,
    .tai   = 37,
};
adjtimex(&txc);
```

## 11分モード（RTC同期） {#the-11-minute-mode-rtc-sync}

クロックが同期している（`STA_UNSYNC`がクリアされている）とき、カーネルは11分ごとに定期的に現在時刻をハードウェアRTCに書き込む。これは`sync_hw_clock()`（workqueueから呼ばれる）で実装されており、NTP補正を持たないRTCが再起動をまたいでUTCに近い状態を保つようにする。この関数は、従来のCMOS/RTCハードウェアに加えて現代のRTCクラスデバイスもサポートすることを反映して、`sync_cmos_clock()`から`sync_hw_clock()`に名前が変更された。

11分の間隔はハードコードされており（`SYNC_PERIOD_NS`、`kernel/time/ntp.c`）、設定できない。`sync_hw_clock()`にはVM固有のチェックはない。単に`update_persistent_clock64()`（従来のCMOSパス）と`update_rtc()`（現代のRTCクラスのパス）を順に試し、両方が`-ENODEV`を返すと自身のタイマーの再設定をやめる。これは恒久的なものではない。`ntp_notify_cmos_timer()`は、次に`adjtimex()`がクロックを同期済みと報告したときに同期処理を再びキューに入れるため、後からRTCを得たゲスト（あるいは後の`adjtimex()`呼び出し）が再試行を引き起こしうる。実際には、両方のパスからの`-ENODEV`は、ゲストにRTCデバイスが公開されていないVMで最もよく見られるが、ロジック自体は「現時点ではどちらの種類のRTCも見つからなかった」というだけであり、仮想化のチェックではない。

## ntpd / chronyのワークフロー {#ntpd-chrony-workflow}

時刻同期デーモンは次のことを行う。

1. 1つ以上のNTPサーバーへの接続を開く（UDPポート123）。
2. 往復時間を測定し、NTPのon-wireプロトコルを使ってクロックオフセットを計算する。
3. `adjtimex()`（または`clock_adjtime()`）を繰り返し呼び出し、カーネルのPLLをオフセットゼロへと駆動する。
4. ポーリング間隔に基づいて`constant`（PLLの時定数）を調整する。ポーリング間隔が長いほど、安定性のために大きな時定数を使う。

`chrony`はいくつかの点で`ntpd`と異なる。

- オフセットが大きいとき（例: 初回起動時）は、ゆっくりスルー（slew）させるのではなく、`ADJ_SETOFFSET`を使って高速なステップ補正を行う。
- PPSやPTPのソースとともに使うことでサブマイクロ秒の精度を得るため、`SO_TIMESTAMPING`によるハードウェアタイムスタンプをサポートする。
- 複数の基準ソースを追跡し、ジッタと距離によって重み付けする。

## 同期状態の確認 {#checking-synchronization-status}

```bash
# 現在のtimexの状態を表示する（util-linuxより）
adjtimex

# systemdから見た状態
timedatectl show
timedatectl status
# 探すもの: NTP synchronized: yes

# chronyの詳細なトラッキング
chronyc tracking
# Reference ID、stratum、システム時刻のオフセット、周波数誤差

# ntpdのピアの状態
ntpq -p
# '*' の接頭辞 = 選択されたピア

# カーネルのtimexの状態をCから直接見る
struct timex tx = {};
int state = adjtimex(&tx);
/* state: TIME_OK=0, TIME_INS=1, TIME_DEL=2, TIME_OOP=3,
          TIME_WAIT=4, TIME_ERROR=5 */
/* tx.status & STA_UNSYNC: 非ゼロ = 同期していない */
```

`STA_UNSYNC`ビットがクリアされている（ゼロである）ことは、カーネルがクロックを同期済みとみなしていることを示す。`timedatectl`はこれを「NTP synchronized: yes」と表示する。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/time/ntp.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/ntp.c) — PLL/FLLによる時刻規律の制御ループ、うるう秒の状態機械、`ntp_tick_length()`、`sync_hw_clock()`
- [kernel/time/time.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/time.c) — `adjtimex()`/`adjtimex_time32()`システムコールのエントリポイント
- [kernel/time/timekeeping.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/timekeeping.c) — `do_adjtimex()`と、NTPのティック補正を`struct timekeeper`に適用する`timekeeping_advance()`/`timekeeping_adjust()`
- [include/uapi/linux/timex.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/timex.h) — `struct timex`、`ADJ_*`モード、`STA_*`ステータスフラグ、`TIME_*`戻りコードのユーザー空間向け定義
- [include/linux/timex.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/timex.h) — カーネル内部のtimex操作、スケーリングされたppmの変換、PPSインターフェースのフック

### manページ {#man-pages}

- [`adjtimex(2)`](https://man7.org/linux/man-pages/man2/adjtimex.2.html) — カーネルの時刻管理パラメータを読み取り、調整するためのシステムコールインターフェース
- [`clock_adjtime(2)`](https://man7.org/linux/man-pages/man2/clock_adjtime.2.html) — POSIXのクロック別調整システムコール。メインラインで`.clock_adj`ハンドラを持つのは`CLOCK_REALTIME`だけ（`kernel/time/posix-timers.c`）なので、上述のとおり`CLOCK_TAI`は`-EOPNOTSUPP`を返す
- [`adjtime(3)`](https://man7.org/linux/man-pages/man3/adjtime.3.html) — クロックを徐々にスルーさせるための従来のCライブラリインターフェース
- [`timedatectl(1)`](https://man7.org/linux/man-pages/man1/timedatectl.1.html) — NTP同期状態とシステムクロックの設定を問い合わせるsystemdのユーティリティ

### 関連ページ {#related-pages}

- [時刻管理とクロックソース](timekeeping.md) — timekeeperのアーキテクチャ、`struct timekeeper`、ハードウェアカウンタとの統合
- [clocksourceとclockeventのドライバ](clocksource.md) — ハードウェアカウンタの較正と周波数乗数の計算
- [時間サブシステムの事件簿](war-stories.md) — 実世界のうるう秒のバグとclocksourceウォッチドッグの誤検知

### LWN記事 {#lwn-articles}

- [ptp: IEEE 1588 hardware clock support](https://lwn.net/Articles/421436/) — Richard Cochranのパッチシリーズ、2010年12月、LWNのカーネルパッチダイジェストで紹介: PTPハードウェアクロックのサポートとともに、`clock_adjtime()`システムコールと`ADJ_SETOFFSET`モードビットを導入する
- [The leap second bug](https://lwn.net/Articles/504657/) — Jonathan Corbet、2012年7月: 2012年のカーネルのうるう秒によるライブロックとその解決の分析

### 外部資料 {#external}

- [Timekeeping and Timers in Linux](https://docs.kernel.org/core-api/timekeeping.html) — 時刻管理、NTPによる規律、ハードウェアクロックの同期に関するカーネルドキュメント
- [RFC 5905: Network Time Protocol Protocol and Algorithm Specification](https://datatracker.ietf.org/doc/html/rfc5905) — NTPv4のon-wireプロトコルとクロック規律アルゴリズムの権威ある仕様
