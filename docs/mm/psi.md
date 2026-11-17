# Pressure Stall Information（PSI） {#pressure-stall-information-psi}

> 使用率だけでなく、リソースの圧迫を定量化する

## PSIとは何か {#what-is-psi}

CPU使用率、空きメモリ、I/Oスループットといった従来のシステム指標は、リソースがどれだけ忙しいかは教えてくれるが、ワークロードが実際に*苦しんでいる*かどうかは教えてくれない。CPU使用率が90%でも、すべてのタスクが進捗していれば、システムはまったく健全であり得る。逆に、CPUが30%のシステムでも、タスクがメモリの回収を待って止まっていれば問題を抱えていることがある。

PSI（Pressure Stall Information）は、タスクが*ストール*している時間、つまり有用な仕事をする代わりにリソースを待っている時間を測定する。これが使用率（リソースのどれだけが使われているか）と圧迫（リソースの不足によってどれだけの仕事が遅れているか）の違いである。

```
Traditional metrics:          PSI:
"CPU is 85% used"             "Tasks lost 12% of productive time
                               waiting for CPU"

"500MB free memory"           "Tasks were fully stalled on memory
                               for 3% of the last 10 seconds"

"Disk is 60% busy"           "No tasks were stalled on I/O"
```

## 追跡される3つのリソース {#three-resources-tracked}

PSIは、タスクが奪い合う3つの基本的なリソースについて圧迫を追跡する。

| リソース | 圧迫ファイル | ストールの原因 |
|----------|--------------|-------------------|
| CPU | `/proc/pressure/cpu` | 実行可能なタスクがCPUコアを待っている |
| メモリ | `/proc/pressure/memory` | タスクがページ回収、スワップI/O、リフォールトを待っている |
| I/O | `/proc/pressure/io` | タスクがブロックデバイスのI/Oを待っている |

```bash
# すべての圧迫指標を表示する
cat /proc/pressure/cpu
cat /proc/pressure/memory
cat /proc/pressure/io
```

## "some"と"full"の圧迫 {#some-vs-full-pressure}

各圧迫ファイルは2つのレベルのストールを報告する。

```
$ cat /proc/pressure/memory
some avg10=0.00 avg60=0.00 avg300=0.00 total=456789
full avg10=0.00 avg60=0.00 avg300=0.00 total=123456
```

**some**: 少なくとも1つのタスクがこのリソースでストールしている。他のタスクはまだ生産的に動いているかもしれない。これは競合を示しており、ワークロードは部分的に劣化している。

**full**: アイドルでない*すべて*のタスクが同時にストールしている。生産的な仕事はまったく行われていない。これははるかに深刻な状態であり、システム（またはcgroup）は完全に行き詰まっている。

```
                Time ──────────────────────>

Task A:    [running][stalled ][running ][stalled ]
Task B:    [running][running ][stalled ][stalled ]
                         │                   │
                    "some" pressure     "full" pressure
                  (one task stalled)   (all tasks stalled)
```

!!! note
    `/proc/pressure/cpu`は`some`のみを報告し、`full`は報告しない。理由は次のとおりである。すべてのタスクがCPUを待っているなら、どのタスクも実行されていないことになり、それはCPUがアイドルであることを意味する。これは矛盾した状態である。CPUの圧迫とは、実行可能なタスクが時間を奪い合っていることを意味するので、少なくとも1つは常に進捗している。

## 圧迫ファイルの読み方 {#reading-the-pressure-files}

各行には3つの移動平均と累積合計が含まれる。

```
some avg10=4.67 avg60=2.13 avg300=0.85 total=2345678
```

| フィールド | 意味 |
|-------|---------|
| `avg10` | 直近10秒間でタスクがストールしていた時間の割合 |
| `avg60` | 直近60秒間でストールしていた時間の割合 |
| `avg300` | 直近5分間でストールしていた時間の割合 |
| `total` | ブート以降の累積ストール時間（マイクロ秒） |

平均値は指数加重移動平均（ロードアベレージと同様）で、パーセント（0.00〜100.00）で表される。`total`カウンタは単調増加し、任意の間隔での正確なストール率を計算するのに役立つ。

```bash
# 5秒間のウィンドウでの正確なストール率を計算する
T1=$(awk '/some/ {print $NF}' /proc/pressure/memory | cut -d= -f2)
sleep 5
T2=$(awk '/some/ {print $NF}' /proc/pressure/memory | cut -d= -f2)
STALL_US=$((T2 - T1))
echo "Memory stall: ${STALL_US}us over 5 seconds"
```

## PSIトリガ: ユーザー空間からの監視 {#psi-triggers-userspace-monitoring}

ポーリングで圧迫ファイルを読むのはダッシュボードには十分だが、リアルタイムのリソース管理にはイベント駆動の通知が必要である。PSIは圧迫ファイルに対するpoll/epollをサポートしており、ユーザー空間は閾値を設定して、圧迫がそれを超えたときにだけ起こされるようにできる。

### トリガの設定 {#setting-up-a-trigger}

圧迫ファイルにトリガの指定を書き込み、`poll()`または`epoll()`でイベントを待つ。

```c
#include <fcntl.h>
#include <poll.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

int main(void) {
    int fd = open("/proc/pressure/memory", O_RDWR | O_NONBLOCK);

    /* 任意の1秒間のウィンドウ内で"some"のストール時間が
       100msを超えたら発火する（値はマイクロ秒） */
    const char *trigger = "some 100000 1000000";
    write(fd, trigger, strlen(trigger) + 1);

    struct pollfd fds = { .fd = fd, .events = POLLPRI };

    for (;;) {
        int ret = poll(&fds, 1, -1);  /* 閾値に達するまでブロックする */
        if (ret > 0) {
            printf("Memory pressure threshold exceeded!\n");
            /* 対処する: キャッシュを破棄する、低優先度の仕事をkillする、など */
        }
    }
}
```

トリガの形式は次のとおりである。

```
<some|full> <stall_us> <window_us>
```

- `some 100000 1000000`は、任意の1秒間のウィンドウ（window_us）内で少なくとも100msのストール時間（stall_us）が発生したら発火する、という意味である。
- ウィンドウは500msから10秒の間でなければならない。非特権ユーザーはさらに、2秒の倍数のウィンドウに制限される。完全な仕様については[`Documentation/accounting/psi.rst`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/accounting/psi.rst)を参照。

PSIトリガのサポートは、v5.2でSuren Baghdasaryanによって追加された（[commit 0e94682b73bf](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0e94682b73bf)）。

## 実践でのPSI {#psi-in-practice}

PSIは単なる監視上の珍しい機能ではない。本番システムにおける現代的なout-of-memory管理の基盤である。

### systemd-oomd {#systemd-oomd}

[systemd-oomd](https://www.freedesktop.org/software/systemd/man/latest/systemd-oomd.html)は、PSIのメモリ圧迫を使って、カーネルのOOM killerが発動する前にcgroupをkillするタイミングを判断する。各cgroupの`memory.pressure`を監視し、圧迫が閾値を超えるとメモリ消費量が最も多いものをkillする。これにより、大雑把なグローバルOOM killerが、対象を絞った早期の介入に置き換えられる。

```ini
# /etc/systemd/oomd.conf
[OOM]
SwapUsedLimit=90%
DefaultMemoryPressureLimit=60%
DefaultMemoryPressureDurationUSec=30s
```

### Metaのoomd {#metas-oomd}

Meta（Facebook）は、systemd-oomdの着想の元となったPSIベースのOOMデーモン[oomd](https://github.com/facebookincubator/oomd)を最初に開発した。Metaのフリートでは、PSIベースのkillがカーネルのOOM killerに取って代わり、より速い応答時間と、何をkillすべきかについてのより良い判断をもたらした。これはPSI開発の主な動機であった。Johannes Weinerは、Metaにおける実際のフリート管理の問題を解決するためにPSIを作った。

PSIが解決しようとした問題についてのより詳しい背景は、[pressure-stall informationの追跡に関するLWNの記事](https://lwn.net/Articles/759658/)を参照。

### Android LMKD {#android-lmkd}

AndroidのLow Memory Killer Daemon（LMKD）は、PSIトリガを使ってメモリ圧迫を検出し、システムが応答しなくなる前にバックグラウンドのアプリをkillする。Androidはカーネル内のlowmemorykillerからPSIベースのユーザー空間のアプローチに移行し、メモリに制約のあるモバイルデバイスでの応答性を向上させた。

### 大規模なリソース管理 {#resource-management-at-scale}

OOMの処理以外にも、PSIは次のことを可能にする。

- **オートスケーリング**: 生の使用率に反応するのではなく、ワークロードが苦しんでいることを圧迫が示したときにコンテナやVMをスケールアップする。
- **ワークロードの配置**: スケジューラは、使用率の低いノードではなく圧迫の低いノードを優先できる。
- **SLA監視**: 恣意的なリソースの閾値ではなく、「タスクがどれだけの時間を失っているか」という観点で健全性を定義する。

## cgroupごとのPSI {#per-cgroup-psi}

PSIはシステム全体の指標に限られない。各cgroupは`memory.pressure`、`cpu.pressure`、`io.pressure`を通じて自身の圧迫情報を公開する。

```bash
# 特定のcgroupのメモリ圧迫を表示する
cat /sys/fs/cgroup/myservice/memory.pressure

# CPUの圧迫を表示する
cat /sys/fs/cgroup/myservice/cpu.pressure

# I/Oの圧迫を表示する
cat /sys/fs/cgroup/myservice/io.pressure
```

cgroupごとのPSIこそが、systemd-oomdやoomdのようなツールを実用的にしている。システム全体の圧迫は問題なくても、単一のコンテナがスラッシングしていることがある。cgroupごとの圧迫はこれを捉える。

PSIトリガはcgroupごとの圧迫ファイルでも機能するので、デーモンは個々のcgroupに対して閾値通知を登録できる。

```c
int fd = open("/sys/fs/cgroup/myservice/memory.pressure",
              O_RDWR | O_NONBLOCK);
write(fd, "some 50000 1000000", 19);  /* 1秒のウィンドウあたり50msのストール */
/* poll(fd) ... */
```

## カーネルがPSIを計算する方法 {#how-the-kernel-calculates-psi}

### タスク状態の追跡 {#task-state-tracking}

PSIは、各時点ですべてのタスクが何をしているかを追跡することで動作する。スケジューラはタスクが実行中か、スリープ中か、待機中かをすでに知っている。PSIはこれらの状態遷移にフックしてストール時間を積算する。

各タスクは、PSIに関係するいくつかの状態のいずれかになり得る。

```
┌────────────────────────────────────────────────────────────┐
│  TSK_IOWAIT          - Waiting for I/O completion          │
│  TSK_MEMSTALL        - Waiting for memory (reclaim, swap)  │
│  TSK_RUNNING         - On a runqueue (may or may not       │
│                        have a CPU)                         │
│  TSK_MEMSTALL_RUNNING - Running but inside a memory stall  │
│                        (e.g. synchronous reclaim)          │
│  TSK_ONCPU           - Currently executing on a CPU        │
└────────────────────────────────────────────────────────────┘
```

これらのフラグは[`include/linux/psi_types.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/psi_types.h)で定義されている。`TSK_MEMSTALL_RUNNING`は、同期回収のパスにいる間にCPU上で実行しているタスクを正しく計上するために追加された。CPUはアイドルではないので、これらは`some`の圧迫には寄与するが`full`の圧迫には寄与しない。

タスクがストール状態に入ると（例: メモリを回収する必要がある、スワップI/Oを待つ）、カーネルは`psi_task_change()`を呼び出して遷移を記録する。ストール状態を抜けるときにも、再び遷移が記録される。

### 集約 {#aggregation}

PSIはCPUごとのレベルで時間を追跡し、それを集約する。各CPUについて、少なくとも1つのタスクがストールしていた時間（`some`）と、すべてのタスクがストールしていた時間（`full`）を記録する。これらのCPUごとのサンプルが集約され、圧迫ファイルで見られる指数加重平均になる。

```
Per-CPU tracking:
  CPU 0: some=150us  full=30us   (in last period)
  CPU 1: some=80us   full=0us
  CPU 2: some=200us  full=100us
  CPU 3: some=0us    full=0us
         │
         v
  Aggregation → weighted averages → /proc/pressure/*
```

集約は`psi_avgs_work()`で行われる。これは移動平均を更新するために定期的に（2秒ごとに）実行される遅延ワーク関数である。ユーザー空間がPSIトリガを登録している場合は、トリガのウィンドウの粒度で、より高速なポーリングのパスが実行される。

### 増分ベースの追跡 {#growth-based-tracking}

重要な最適化（[commit df77430639c9](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=df77430639c9)）として、PSIはロックではなくCPUごとの「増分（growth）」カウンタを使う。各CPUは独立して累積ストール時間を追跡する。集約のステップでこれらのカウンタを読み取って差分を計算することで、ホットパスでの競合を避けている。

## 歴史 {#history}

### 導入（v4.20、2018年） {#introduction-v420-2018}

**コミット**: [eb414681d5a0](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=eb414681d5a0)（"psi: pressure stall information for CPU, memory, and IO"）

**作者**: Johannes Weiner（Facebook/Meta）

PSIはLinux v4.20（2018年12月）でマージされた。[コミットメッセージ](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=eb414681d5a0)と[LWNの記事](https://lwn.net/Articles/759658/)に記されているとおり、その動機は、既存の指標（ロードアベレージ、空きメモリ、I/O使用率）が、ワークロードが実際にリソース不足の影響を受けているかどうかを示す指標としては不十分だったことにある。Facebookは、フリート全体のリソース管理の判断を駆動するために、単なるリソース消費量ではなく、失われた仕事を定量化する指標を必要としていた。

### PSIトリガ（v5.2、2019年） {#psi-triggers-v52-2019}

**コミット**: [0e94682b73bf](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0e94682b73bf)（"psi: introduce psi monitor"）

**作者**: Suren Baghdasaryan

PSIファイルに対するpoll/epollのサポートを追加し、イベント駆動のユーザー空間からの監視を可能にした。これはoomdのようなデーモンでPSIを実用的にするために不可欠だった。ループでファイルをポーリングするのは高コストで、レイテンシも生じる。

### cgroupごとのPSI（v4.20） {#per-cgroup-psi-v420}

cgroupレベルの圧迫は当初から中心的なユースケースだったため、cgroupごとのPSIは最初のPSIパッチセットの一部であった。

### 広範な採用 {#broader-adoption}

- **Android**: Android 10（2019年）でLMKDにPSIを採用
- **systemd**: systemd-oomdが[systemd v247で導入](https://lwn.net/Articles/837034/)（2020年）
- **Meta**: OOMの防止とワークロードのスケジューリングのため、フリート全体でPSIを使用

## 主要なソースファイル {#key-source-files}

| ファイル | 説明 |
|------|-------------|
| [`kernel/sched/psi.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/psi.c) | PSIのコア実装: 状態の追跡、集約、トリガ |
| [`include/linux/psi.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/psi.h) | PSIのAPIとインライン関数 |
| [`include/linux/psi_types.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/psi_types.h) | PSIのデータ構造の定義 |
| [`kernel/cgroup/cgroup.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/cgroup/cgroup.c) | cgroupごとのPSIファイルの公開 |
| [`Documentation/accounting/psi.rst`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/accounting/psi.rst) | PSIのカーネルドキュメント |

## 試してみる {#try-it-yourself}

### システムの圧迫を表示する {#view-system-pressure}

```bash
# PSIが有効か確認する（最近のカーネルの多くではデフォルトで有効）
cat /proc/pressure/cpu
cat /proc/pressure/memory
cat /proc/pressure/io
```

ファイルが存在しない場合、PSIは無効化されている可能性がある。カーネルのブートパラメータ`psi=1`で有効化できる。一部のディストリビューションは、性能上の理由からデフォルトで無効にしている（オーバーヘッドは最小限だがゼロではない）。

### メモリ圧迫を発生させて観察する {#generate-memory-pressure-and-observe}

```bash
# ターミナル1: メモリ圧迫を監視する
watch -n 1 cat /proc/pressure/memory

# ターミナル2: メモリ圧迫を発生させる
# 利用可能な量より多くのメモリを割り当てる（サイズはシステムに合わせて調整する）
stress --vm 4 --vm-bytes $(awk '/MemAvailable/ {print int($2*0.3)"k"}' \
    /proc/meminfo) --timeout 30
```

メモリが圧迫されている間、`some`と`full`の両方の行で`avg10`が上昇するのが見えるはずである。

### cgroupごとの圧迫の監視 {#per-cgroup-pressure-monitoring}

```bash
# メモリ制限付きのcgroupを作成する
sudo mkdir /sys/fs/cgroup/psi-test
echo "+memory" | sudo tee /sys/fs/cgroup/cgroup.subtree_control
echo 100M | sudo tee /sys/fs/cgroup/psi-test/memory.max

# シェルをcgroupに移す
echo $$ | sudo tee /sys/fs/cgroup/psi-test/cgroup.procs

# cgroup固有の圧迫を監視する
watch -n 1 cat /sys/fs/cgroup/psi-test/memory.pressure

# 同じシェルで圧迫を発生させる
python3 -c "x = bytearray(80 * 1024 * 1024)"  # 100MBのcgroupで80MB
```

### totalカウンタを使った精密な測定 {#using-the-total-counter-for-precise-measurements}

```bash
# 任意の間隔での正確なストール時間を測定する
read_total() {
    awk '/^some/ {for(i=1;i<=NF;i++) if($i~/^total=/) print substr($i,7)}' \
        /proc/pressure/memory
}

START=$(read_total)
# ここでワークロードを実行する
sleep 10
END=$(read_total)

STALL_US=$((END - START))
STALL_PCT=$(echo "scale=2; $STALL_US / 10000000 * 100" | bc)
echo "Memory stall: ${STALL_US}us (${STALL_PCT}% of 10s)"
```

### 後片付け {#clean-up}

```bash
# テスト用cgroupを削除する（先にプロセスを外に移す）
echo $$ | sudo tee /sys/fs/cgroup/cgroup.procs
sudo rmdir /sys/fs/cgroup/psi-test
```

## 参考資料 {#references}

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/accounting/psi.rst`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/accounting/psi.rst) — 権威あるPSIのドキュメント

### LWNの記事 {#lwn-articles}

- [Tracking pressure-stall information](https://lwn.net/Articles/759658/) — PSIパッチセットの最初の紹介記事（2018年）
- [Monitoring memory pressure with PSI](https://lwn.net/Articles/837034/) — systemd-oomdとPSIトリガ

### 関連項目 {#related}

- [メモリcgroup](memcg.md) — cgroupごとのメモリ制限と計上
- [ページ回収](reclaim.md) — メモリのストールを引き起こすもの
- [メモリ不足](oom.md) — PSIが回避を助けるOOM killer
- [用語集](glossary.md) — PSI、cgroup、OOMの定義

## 参考文献 {#further-reading}

### カーネルドキュメント {#kernel-documentation_1}

- `Documentation/accounting/psi.rst` — 圧迫ファイルの形式、トリガの構文、ウィンドウの制約、cgroupとの統合の完全な仕様
- [kernel/sched/psi.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/psi.c) — PSIのコア実装: CPUごとの状態追跡、`some`/`full`の計上、指数加重平均、トリガの通知

### LWNの記事 {#lwn-articles_1}

- [Tracking pressure-stall information](https://lwn.net/Articles/759658/) — Johannes Weinerによる最初のPSIパッチセットについてのTejun Heoの記事。動機と`some`/`full`の区別を説明している（2018年）
- [Monitoring memory pressure with PSI](https://lwn.net/Articles/837034/) — systemd-oomdがPSIトリガを使って、カーネルのOOM killerを対象を絞った早期のcgroup退避に置き換える方法

### 関連ドキュメント {#related-docs}

- [memcg.md](memcg.md) — cgroupごとのメモリ制限。systemd-oomdやAndroid LMKDが監視するのは、cgroupごとのPSIファイル（`memory.pressure`、`cpu.pressure`、`io.pressure`）である
- [oom.md](oom.md) — カーネルのOOM killer。PSIベースのデーモンは、カーネルがそれを呼び出す必要が生じる前に介入することを目指す
- [reclaim.md](reclaim.md) — ページ回収は、PSIが測定するメモリのストールの主な原因である
- [damon.md](damon.md) — DAMON_RECLAIMは、回収クォータの自動チューニングのフィードバックとしてPSIのメモリ圧迫を使える
