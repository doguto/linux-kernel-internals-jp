# 時間namespace {#time-namespaces}

> コンテナごとのクロックオフセット: CLOCK_MONOTONICとCLOCK_BOOTTIMEの分離

## 動機 {#motivation}

Linuxのnamespaceは、カーネル資源をコンテナごとに分離する。ほとんどのnamespace（ネットワーク、マウント、PID）は、それを使うプロセス自身の中から`unshare()`や`clone()`を呼び出して作成できる。時間namespace（`CLONE_NEWTIME`）は別の問題を解決する。それは、チェックポイント/リストアのサイクルをまたいでも`CLOCK_MONOTONIC`と`CLOCK_BOOTTIME`が連続しているように見せることである。

時間namespaceがなければ、t=1000でチェックポイントされ3600秒後にリストアされたコンテナからは、`CLOCK_MONOTONIC`が1000から4600へジャンプして見える。この3600秒は、コンテナが凍結されていた間に実際に経過した時間である。チェックポイント前にモノトニッククロックを基準に計算されたタイムアウトや期限は、リストア後には大きく狂ってしまう。時間namespaceにより、CRIUや同様のツールは、中断中に時間が経過しなかったかのような錯覚をコンテナに与えられる。

## CLONE_NEWTIME（Linux 5.6） {#clone_newtime-linux-56}

```c
/* 新しい時間namespaceを作成する */
unshare(CLONE_NEWTIME);

/* あるいはプロセス作成時に */
clone(child_fn, stack, CLONE_NEWTIME | SIGCHLD, NULL);
```

`CLONE_NEWTIME`はLinux 5.6で導入された。これはnamespaceの種類の中で、ある重要な点で特異である。**`/proc/<pid>/timens_offsets`のオフセットは、いずれかのプロセスが新しい時間namespaceに入る前に書き込まなければならない**。`unshare(CLONE_NEWTIME)`を呼ぶと、*将来の子プロセス*のための新しい時間namespace（`time_ns_for_children`）が作られるが、呼び出したプロセス自身は元の時間namespaceにとどまる。最初の子プロセスがforkまたはspawnされると、`timens_on_fork()`がnamespaceを確定させ、namespaceごとのvvarページを設定し、`frozen_offsets = true`を恒久的にセットする。プロセスがnamespaceに入った後で`timens_offsets`に書き込もうとすると、`-EACCES`で失敗する。

## namespace化されるもの・されないもの {#what-is-and-is-not-namespaced}

| クロック | namespace化されるか |
|-------|-------------|
| `CLOCK_MONOTONIC` | される — オフセットが適用される |
| `CLOCK_BOOTTIME` | される — オフセットが適用される |
| `CLOCK_REALTIME` | されない — 常にグローバルなUTC |
| `CLOCK_TAI` | されない — 常にグローバルなTAI |
| `CLOCK_PROCESS_CPUTIME_ID` | されない — プロセスごとのCPU時間 |
| `CLOCK_THREAD_CPUTIME_ID` | されない — スレッドごとのCPU時間 |

`CLOCK_REALTIME`は意図的にnamespace化されていない。壁時計時刻はホスト上のすべてのコンテナで共有される。namespaceごとのオフセットが与えられるのは、ある基準点からの経過時間を測るモノトニッククロックとブート時間クロックだけである。

## struct time_namespace {#struct-time_namespace}

```c
/* include/linux/time_namespace.h */
struct time_namespace {
    struct user_namespace   *user_ns;
    struct ucounts          *ucounts;
    struct ns_common         ns;
    struct timens_offsets    offsets;   /* クロックの調整値 */
#ifdef CONFIG_TIME_NS_VDSO
    struct page             *vvar_page; /* namespaceごとのvDSOデータページ */
#endif
    bool                     frozen_offsets; /* いずれかのプロセスがnamespaceに入った後はtrue */
} __randomize_layout;
```

`offsets`フィールドは、`CLOCK_MONOTONIC`と`CLOCK_BOOTTIME`の読み取りに加算される値を保持する。

```c
struct timens_offsets {
    struct timespec64 monotonic; /* CLOCK_MONOTONIC の読み取りに加算 */
    struct timespec64 boottime;  /* CLOCK_BOOTTIME の読み取りに加算 */
};
```

これらは単純な`timespec64`値であり、ユーザー空間に返す前にベースのクロック値に秒とナノ秒が加算される。

## vDSOとの統合 {#vdso-integration}

`clock_gettime(CLOCK_MONOTONIC)`はvDSO（virtual dynamic shared object）で実装されている。vDSOはすべてのプロセスのアドレス空間にカーネルがマップするページであり、システムコールなしで呼び出しを実行できるようにする。時間namespaceをvDSOと透過的に連携させるため、各時間namespaceは独自のvvar（vDSO変数）ページを持つ。

プロセスが新しい時間namespaceに入ると、カーネルはそのnamespace固有のvvarページをプロセスのアドレス空間にマップする。vDSOのクロック関数はこのnamespaceごとのページからクロックデータを読む。このページには、ベース時刻にnamespaceのオフセットがあらかじめ組み込まれた`basetime`フィールドが含まれている。オフセットはnamespace設定時にvvarページへ適用されるため、vDSOのパスは実行時に、namespace化されていない場合と比べて余分なコストを払わない。

つまり、時間namespaceを持つコンテナ内の`clock_gettime(CLOCK_MONOTONIC)`は引き続きvDSO呼び出し（システムコールなし）のままであり、カーネルの関与なしにオフセット調整済みの時刻を返す。

## オフセットの設定 {#setting-offsets}

オフセットは、いずれかの子プロセスが新しいnamespaceに入る前に、namespaceの外から`/proc/<pid>/timens_offsets`に書き込んで設定する。

```
# Format: <clock_name> <seconds> <nanoseconds>
monotonic <sec> <nsec>
boottime  <sec> <nsec>
```

例 — コンテナに、1000秒前に起動したかのような錯覚を与える:

```bash
# 将来の子プロセスのために時間namespaceをunshareする
unshare --time bash -c '
    # 親は将来の子のためのオフセットを自身の timens_offsets に書き込む
    echo "monotonic 1000 0" > /proc/self/timens_offsets
    echo "boottime 1000 0"  > /proc/self/timens_offsets

    # ここで子を新しいnamespaceにforkする
    # timens_on_fork() がfork時にオフセットを凍結する
    python3 -c "import time; print(time.monotonic())"
'
```

いずれかのプロセスがnamespaceに入ると（例: `unshare(CLONE_NEWTIME)`後の`fork()`時）、`timens_on_fork()`がnamespaceを確定させ、オフセットを凍結する。それ以降の`timens_offsets`への書き込みは`EACCES`を返す。

オフセットは負の値にもできる（コンテナからはホストのモノトニッククロックより早い時刻に見える。コンテナがホストのモノトニックの基準点より前に作られていた場合のCRIUリストアで有用である）。

## CRIUのユースケース {#criu-use-case}

CRIU（Checkpoint/Restore In Userspace）は時間namespaceの主要な利用者である。CRIUがチェックポイントをリストアするときは次のようになる。

1. CRIUのリストアプロセスが`unshare(CLONE_NEWTIME)`を呼ぶ。
2. 対象のコンテナプロセスをforkする前に、CRIUは適切なオフセットを`/proc/self/timens_offsets`に書き込む。オフセットは、チェックポイント時点のコンテナのモノトニック時刻と、ホストの現在のモノトニック時刻との差を表す。
3. CRIUは対象プロセスを新しいnamespaceにforkする（`timens_on_fork()`がオフセットを確定させ、namespaceごとのVVARページをマップする）。リストアされたプロセスからは`CLOCK_MONOTONIC`が連続して見え、中断中に起きた壁時計時刻のジャンプは隠される。

時間namespaceがなければ、チェックポイント前にコンテナが`CLOCK_MONOTONIC`を基準に計算したタイマー、タイムアウト、期限は、リストア後に即座に満了するか、残り時間が大きく狂ってしまう。

## timerfdとの相互作用 {#timerfd-interaction}

時間namespace内で`CLOCK_MONOTONIC`を指定して作成した`timerfd`は、namespace化されたクロックを使う。

```c
/* コンテナ内 */
int fd = timerfd_create(CLOCK_MONOTONIC, 0);
struct itimerspec its = {
    .it_value = { .tv_sec = 5, .tv_nsec = 0 },
};
timerfd_settime(fd, TFD_TIMER_ABSTIME, &its, NULL);
/* namespace化された CLOCK_MONOTONIC が今から5秒後に達したときに発火 */
```

カーネルはtimerfdの満了をnamespace調整済みのクロックに対して追跡する。CRIUのリストア後、まだ満了していなかったタイマーは、元の相対位置から正しくカウントダウンを続ける。

## 現在の制限 {#current-limitations}

- オフセットが適用されるのは`CLOCK_MONOTONIC`と`CLOCK_BOOTTIME`だけである。その他のクロックはnamespace化されない。
- オフセットは整数（秒+ナノ秒）であり、namespaceごとの周波数補正やドリフト補償はない。
- プロセスが同時に属せる時間namespaceは高々1つであり、入れ子（スタッキング）はない。
- 時間namespaceはRTC、NTP同期、`adjtimex()`には影響しない。これらはグローバルである。
- `/proc/uptime`はnamespace内の`CLOCK_BOOTTIME`オフセットの影響を受けるため、コンテナ内の`uptime`はnamespace調整済みのブート時刻を表示する。

## 時間namespaceの調査 {#inspecting-time-namespaces}

```bash
# プロセスがどの時間namespaceにいるかを見る
ls -la /proc/<pid>/ns/time

# 2つのプロセスを比較する
readlink /proc/1/ns/time
readlink /proc/$$/ns/time

# プロセスのオフセットを表示する
cat /proc/<pid>/timens_offsets
# monotonic 1000 0
# boottime  1000 0

# 内側からクロックの分離を確認する
# （nsenter またはnamespace内での実行が必要）
nsenter --time=/proc/<pid>/ns/time -- \
    python3 -c "import time; print(time.monotonic())"
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/time_namespace.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/time_namespace.h) — `struct time_namespace`、`struct timens_offsets`、`timens_on_fork()`フックの定義
- [kernel/time/namespace.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/namespace.c) — 時間namespaceのライフサイクル、`/proc/[pid]/timens_offsets`の解析、権限チェック、クロックオフセットの変換（`do_timens_ktime_to_host()`）
- [kernel/time/namespace_vdso.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/namespace_vdso.c) — namespaceごとのVVARページの設定（`timens_setup_vdso_clock_data()`）、VVARページフォールトの処理、`timens_commit()`

### manページ {#man-pages}

- [`time_namespaces(7)`](https://man7.org/linux/man-pages/man7/time_namespaces.7.html) — Linuxの時間namespaceの概要、`/proc/[pid]/timens_offsets`の構文、オフセット凍結の規則
- [`unshare(2)`](https://man7.org/linux/man-pages/man2/unshare.2.html) — 実行コンテキストの分離と、`CLONE_NEWTIME`による新しい時間namespaceの作成
- [`setns(2)`](https://man7.org/linux/man-pages/man2/setns.2.html) — ファイルディスクリプタを介した既存の時間namespaceへの参加
- [`clock_gettime(2)`](https://man7.org/linux/man-pages/man2/clock_gettime.2.html) — 時間namespace内での`CLOCK_MONOTONIC`と`CLOCK_BOOTTIME`の読み取り

### 関連ページ {#related-pages}

- [時刻管理とクロックソース](timekeeping.md) — 基本となる`CLOCK_MONOTONIC`と`CLOCK_BOOTTIME`の仕組みと、vDSOデータのレイアウト
- [POSIXタイマーとtimerfd](posix-timers.md) — 時間namespace内でのtimerfd満了時刻の調整
- [vDSO高速システムコール](../mm/vdso.md) — カーネルとユーザー間の共有データページと、VVARマッピングのアーキテクチャ

### LWN記事 {#lwn-articles}

- [Time namespaces](https://lwn.net/Articles/766089/) — Jonathan Corbet、2018年9月: CRIUのチェックポイント/リストアと仮想化のための、コンテナ化されたクロックオフセット
- [The 5.6 merge window opens](https://lwn.net/Articles/810780/) — Jonathan Corbet、2020年1月30日: 時間namespace（`CLONE_NEWTIME`）のメインラインLinuxへの正式なマージ

### 外部資料 {#external}

- [Namespaces in the Linux Kernel](https://docs.kernel.org/admin-guide/namespaces/index.html) — namespaceと仮想化を扱う公式カーネルドキュメント
