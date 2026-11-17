# コンテナのためのメモリチューニング {#tuning-memory-for-containers}

> コンテナのメモリ制限を正しく設定する -- そして誤っていたときにデバッグする

## なぜコンテナのメモリは異なるのか {#why-container-memory-is-different}

ベアメタルサーバーでは、すべてのプロセスがカーネルの管理する単一のメモリプールを共有する。コンテナはこのモデルを根本的に変える。各コンテナは、カーネルの **cgroupメモリコントローラ** によって強制される独自のメモリ境界を持つ。このコントローラの仕組みを理解しているかどうかが、安定した本番デプロイと、午前3時の謎のOOM killの違いになる。

このページは **cgroup v2** に焦点を当てる。cgroup v2は、すべての新規デプロイに対する[推奨されるcgroupバージョン](https://lwn.net/Articles/679786/)である（カーネルv4.5で本番利用可能と宣言された）。まだcgroup v1を使っている場合、概念は似ているがインターフェースファイルが異なる。v1とv2の違いについては[メモリcgroup](memcg.md)を参照。

## 4つのメモリノブ {#the-four-memory-knobs}

cgroup v2のメモリコントローラは、プロセスのグループが使えるメモリ量を制御する4つのファイルを公開している。これらは、強制力のある上限から緩い保護までの階層を構成する。

```
                    memory.max      <- Hard ceiling. Exceed this and die.
                        |
                    memory.high     <- Soft ceiling. Exceed this and get throttled.
                        |
                        |  (normal operation zone)
                        |
                    memory.low      <- Soft floor. Protected from reclaim (best-effort).
                        |
                    memory.min      <- Hard floor. Absolutely protected from reclaim.
```

| ファイル | 種類 | 何が起きるか | デフォルト |
|------|------|-------------|---------|
| [`memory.max`](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory-interface-files) | ハード制限 | 超えるとOOM killerが起動される | `max`（制限なし） |
| [`memory.high`](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory-interface-files) | ソフト制限 | プロセスがスロットリングされ、積極的に回収される | `max`（制限なし） |
| [`memory.low`](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory-interface-files) | ソフト保護 | これ未満のメモリは回収から保護される（ベストエフォート） | `0` |
| [`memory.min`](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory-interface-files) | ハード保護 | これ未満のメモリは決して回収されない | `0` |

### memory.max: ハード制限 {#memorymax-the-hard-limit}

cgroupのメモリ使用量が`memory.max`に達すると、カーネルはそのcgroup内で回収を試みる。回収で十分なメモリを解放できなければ、cgroupのOOM killerが起動され、cgroup内のプロセスが1つkillされる。ホストシステムには影響しない -- これはグローバルなOOMではなく、cgroupスコープのOOMである。

```bash
# 1GBのハード制限を設定する
echo 1G > /sys/fs/cgroup/mycontainer/memory.max
```

これはコンテナが乗り越えられない壁である。単一のコンテナがシステムの残りを飢えさせることを防ぐために存在する。

### memory.high: ソフト制限 {#memoryhigh-the-soft-limit}

`memory.high`はもっと繊細である。使用量がこの閾値を超えても、カーネルは何もkillしない。その代わりにcgroupから積極的にメモリを回収し、割り当てを行うプロセスを **スロットリングする**。それらのプロセスは、回収によって十分なページが解放されるまでスリープさせられる。

```bash
# 800MBでスロットリングし、1GBでkillする
echo 800M > /sys/fs/cgroup/mycontainer/memory.high
echo 1G > /sys/fs/cgroup/mycontainer/memory.max
```

なぜ両方が必要なのか。`memory.high`はバックプレッシャーを生むからである。壁にぶつかって死ぬまで全速力で走る代わりに、アプリケーションは徐々に減速する。これにより監視が問題を検出する時間が生まれ、OOM killよりはるかに穏やかである。

!!! tip "制限速度と崖のようなものと考える"
    `memory.high`は制限速度の標識である -- そこに達したら減速する。`memory.max`は崖の縁である -- 越えたら終わりである。

### memory.lowとmemory.min: 保護の保証 {#memorylow-and-memorymin-protection-guarantees}

これらのノブは逆方向に働く。cgroupが使えるメモリ量を制限する代わりに、システム全体のメモリ圧迫の際に保持できる量を保証する。

- **`memory.low`**: ベストエフォートの保護。この閾値未満のメモリは、システム内のどこにもほかに回収可能なメモリがない場合を除いて回収されない。「どうしても必要でない限り、このメモリは取らないでほしい」と考えるとよい。
- **`memory.min`**: ハードな保護。この閾値未満のメモリは、何があっても回収されない。極度の圧迫下でも、カーネルはこのcgroupの保護されたメモリを回収するよりも、ほかのものをOOM killする。

```bash
# データベースコンテナが少なくとも2GBを保持することを保証する
echo 2G > /sys/fs/cgroup/database/memory.min

# 可能なら4GBを保持することを優先する
echo 4G > /sys/fs/cgroup/database/memory.low
```

**保護は階層的である**: 子cgroupの実効的な保護は、親の保護によって制限される。親の`memory.min`が4GBの場合、子の`memory.min`の合計は4GBを超えてもよいが、実効的な保護は親の4GBまでの範囲で比例配分される。これは[cgroup v2の配分モデル](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory-interface-files)に記載されている。

```
parent (memory.min: 4GB)
├── child-a (memory.min: 3GB)  <- effective: ~2.4GB (3/5 of 4GB)
├── child-b (memory.min: 2GB)  <- effective: ~1.6GB (2/5 of 4GB)
```

## なぜコンテナは誤ったメモリ量を見るのか {#why-containers-see-wrong-memory-numbers}

典型的なコンテナの中で`free`を実行すると、コンテナの制限ではなく **ホストの** 総メモリが表示される。

```bash
# 1GB制限のコンテナ内で
$ free -h
              total        used        free
Mem:           62Gi        45Gi        17Gi    # <- これはホストのメモリ!
```

これは、`/proc/meminfo`がシステム全体のメモリを報告するグローバルなファイルだからである。コンテナのcgroup制限はcgroupファイルシステムにのみ存在し、`/proc`には存在しない。多くのアプリケーション（特にJVM、データベース、言語ランタイム）は`/proc/meminfo`を読んで自身を自動チューニングする。1GBではなく62GBの総メモリを見ると、それらはあまりにも多くを割り当て、OOM killされる。

### 解決策 {#the-solutions}

**LXCFS**: `/proc/meminfo`、`/proc/cpuinfo`などのファイルの読み出しを横取りし、代わりにcgroupを考慮した値を返すFUSEファイルシステム。[LXCFS](https://linuxcontainers.org/lxcfs/)はコンテナ内の`/proc`の上にマウントされるため、既存のツールが修正なしで動作する。

**ランタイム固有のフラグ**: 現在では多くのランタイムがcgroup制限をネイティブに検出する。

```bash
# Java 10以降はcgroup制限を自動的に検出する
java -XX:+UseContainerSupport -jar app.jar

# Java 8u191以降もサポートしている
java -XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -jar app.jar
```

**cgroupを考慮したツール**: 最近のバージョンの`systemd-cgtop`、`cadvisor`、`kubectl top`は、`/proc/meminfo`ではなくcgroupファイルシステムから直接読む。

## よくある問題 {#common-problems}

### ホストに空きメモリがあるのにコンテナがOOMになる {#container-oom-when-host-has-free-memory}

これは最も頻繁に誤診されるコンテナの問題である。ホストは17GBの空きを示しているのに、コンテナがkillされた。

原因は単純である。コンテナが **cgroupのメモリ制限**（`memory.max`）に達し、cgroupスコープのOOM killが発動したのである。グローバルな空きメモリは関係ない -- cgroupは独自の計上を持つ。

```bash
# cgroupの制限に達したことを確認する
cat /sys/fs/cgroup/mycontainer/memory.events
# oom 3          <- OOMが3回発動した
# oom_kill 3     <- 3つのプロセスがkillされた
# max 47         <- memory.maxに47回達した
```

**対処法**:

- 制限が低すぎるなら`memory.max`を引き上げる
- OOMの前にバックプレッシャーを生むよう、`memory.max`より下に`memory.high`を追加する
- アプリケーションのメモリ使用量を最適化する
- `memory.swap.max`でスワップを許可し、バッファを設ける

### ページキャッシュが制限に計上される {#page-cache-counting-against-the-limit}

cgroup v2では、**ページキャッシュはファイルを読んだcgroupに課金される**。これは多くの人の不意を突く。大きなファイルを読むコンテナは、カーネルがそのキャッシュをいつでも回収できるにもかかわらず、ページキャッシュがメモリ制限に計上される。

```bash
# キャッシュと匿名メモリがそれぞれどれだけか確認する
cat /sys/fs/cgroup/mycontainer/memory.stat | grep -E '^(anon|file) '
# anon 524288000     <- 実際の作業用メモリ
# file 209715200     <- ページキャッシュ（回収可能）
```

キャッシュは回収可能なので、それ自体がOOMを引き起こすことはない。しかし合計の使用量がソフト制限を超えれば、`memory.high`のスロットリングは引き起こす。これが問題になる場合は、ページキャッシュの余裕を考慮して`memory.high`のサイズを決めること。

### メモリ圧迫下でのコンテナの起動の遅さ {#slow-container-start-under-memory-pressure}

コンテナの起動が遅い場合、`memory.high`が実際の使用量に近すぎる値に設定されているのかもしれない。すべての割り当てが回収とスロットリングを引き起こす。確認するには`memory.pressure`を見る。

```bash
cat /sys/fs/cgroup/mycontainer/memory.pressure
# some avg10=45.00 avg60=30.00 avg300=12.00 total=890234
# full avg10=20.00 avg60=10.00 avg300=5.00 total=234567
```

これらの数値が高ければ、コンテナはメモリ待ちにかなりの時間を費やしている。

## Kubernetes: requestsとlimits {#kubernetes-requests-vs-limits}

Kubernetesはコンテナごとに2つのメモリ設定を公開しており、それらはcgroupのノブに直接対応する。

| Kubernetes | cgroup v2 | 挙動 |
|-----------|-----------|----------|
| `resources.limits.memory` | `memory.max` | ハード制限。超えるとコンテナはOOM killされる。 |
| `resources.requests.memory` | `memory.min`（MemoryQoSフィーチャーゲートが必要） | スケジューリングの保証。QoS分類に使われる。`memory.min`への対応付けには、アルファ版の[MemoryQoSフィーチャーゲート](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)（k/enhancements#2570）が必要。 |

```yaml
# Podのspec
resources:
  requests:
    memory: "512Mi"   # 512Miが利用可能なノードにスケジュールし、この量を保護する
  limits:
    memory: "1Gi"     # ハードな上限。これを超えるとOOM kill
```

### QoSクラスがメモリに与える影響 {#how-qos-classes-affect-memory}

Kubernetesは、requestsとlimitsに基づいてQuality of Serviceクラスを割り当てる。

- **Guaranteed**（requests == limits）: コンテナは要求したとおりの量を得る。MemoryQoSが有効な場合、kubeletは`memory.min`をlimitと等しく設定し、ハードな保護を提供する。
- **Burstable**（requests < limits）: コンテナはlimitまで、要求を超えて使える。MemoryQoSが有効な場合、`memory.min`がrequestの値に設定される。
- **BestEffort**（requestsもlimitsもない）: cgroupのメモリ制限は設定されない。ノードの圧迫下で最初にkillされる。

!!! note "MemoryQoSフィーチャーゲート"
    上で述べた`memory.min`への対応付けには`MemoryQoS`フィーチャーゲートが必要である。これはKubernetes 1.22以降でアルファ版であり、デフォルトでは有効になっていない。標準的なKubernetesのデプロイでは、`resources.requests.memory`はスケジューリングとQoSクラスにのみ影響し、cgroupの`memory.min`は設定しない。

```
Node (16GB)
├── Guaranteed pod (memory.min=1G, memory.max=1G)  <- always safe
├── Burstable pod  (memory.min=512M, memory.max=2G) <- protected up to 512M
└── BestEffort pod (no limits)                       <- first to die
```

!!! warning "limitsを設定しない場合でもrequestsは設定すること"
    `requests.memory`がないと、PodはBestEffortクラスになり、ノードのメモリが不足したときに最初に退避させられる。少なくともrequestsは常に設定すること。

## Docker: --memoryと--memory-reservation {#docker-memory-and-memory-reservation}

Dockerはコマンドラインフラグを通じてコンテナのメモリ設定を公開している。

| Dockerフラグ | cgroup v2 | 挙動 |
|------------|-----------|----------|
| `--memory`（または`-m`） | `memory.max` | ハード制限。超えるとOOM kill。 |
| `--memory-reservation` | `memory.low` | システムの圧迫下でのソフトな保護。 |
| `--memory-swap` | `memory.swap.max` + `memory.max` | メモリ＋スワップの合計制限。 |
| `--oom-kill-disable` | OOM killerを無効化 | **危険**: コンテナとホストがハングしうる。 |

```bash
# 1GBの制限と750MBの予約でコンテナを実行する
docker run -d \
  --memory=1g \
  --memory-reservation=750m \
  --name myapp myimage

# 等価なcgroup設定:
# memory.max = 1073741824
# memory.low = 786432000
```

!!! warning "Dockerの--memory-swapのセマンティクス"
    `--memory-swap`は、スワップだけでなくメモリ＋スワップの **合計** の制限を設定する。したがって`--memory=1g --memory-swap=2g`は、1GBのRAM＋1GBのスワップを意味する。スワップを無制限にするには`--memory-swap=-1`を設定する。

## コンテナのメモリの監視 {#monitoring-container-memory}

### 主要なファイル {#key-files}

```bash
CGROUP=/sys/fs/cgroup/mycontainer

# 現在の使用量（バイト）
cat $CGROUP/memory.current
# 734003200

# ピーク使用量（サイズ決めに有用）
cat $CGROUP/memory.peak       # v5.19以降
# 891289600

# 詳細な内訳
cat $CGROUP/memory.stat
# anon 524288000              # ヒープ、スタック、匿名mmap
# file 209715200              # ページキャッシュ
# slab 12582912               # カーネルのslabオブジェクト
# sock 4194304                # ソケットバッファ
# shmem 0                     # 共有メモリ（tmpfs）
# pgfault 1234567             # ページフォールト
# pgmajfault 42               # メジャーフォールト（ディスクI/O）

# イベント（発生した問題）
cat $CGROUP/memory.events
# low 0                       # memory.lowを下回った回数
# high 152                    # memory.highを超えた回数
# max 3                       # memory.maxに達した回数
# oom 1                       # OOMイベント
# oom_kill 1                  # OOM killされたプロセス
# oom_group_kill 0            # グループOOM kill
```

### Pressure Stall Information（PSI） {#pressure-stall-information-psi}

PSIは、コンテナがメモリに制約されているかどうかについて、最も行動に結びつけやすいシグナルを与えてくれる。

```bash
cat $CGROUP/memory.pressure
# some avg10=2.50 avg60=1.00 avg300=0.50 total=45678
# full avg10=0.00 avg60=0.00 avg300=0.00 total=1234
```

- **some**: cgroup内の少なくとも1つのタスクがメモリ待ちでストールしていた時間の割合。これが常にゼロより大きければ、コンテナは圧迫下にある。
- **full**: すべてのタスクがストールしていた時間の割合。これがゼロより大きければ、コンテナはメモリで完全にブロックされている。

PSIは[v4.20](https://lwn.net/Articles/759658/)でcgroupレベルの機能として導入され、Metaのユーザー空間OOMデーモン[oomd](https://facebookincubator.github.io/oomd/)の基盤となっている。oomdはPSIを使い、カーネルのOOM killerが発動する前に先回りしてワークロードをkillする。

```bash
# 圧迫が閾値を超えたときに通知を受けるPSIトリガーを設定する
# （オートスケーリングの構築に有用）
echo "some 50000 1000000" > $CGROUP/memory.pressure
# 1秒のウィンドウあたり"some"が50msを超えたら発火する
```

## コンテナにおけるスワップ {#swap-in-containers}

cgroup v2のデフォルトでは、スワップはシステムが提供する分まで許可される。これはcgroupごとに制御できる。

```bash
# このコンテナのスワップを完全に無効にする
echo 0 > /sys/fs/cgroup/mycontainer/memory.swap.max

# 最大500MBのスワップを許可する
echo 500M > /sys/fs/cgroup/mycontainer/memory.swap.max

# 現在のスワップ使用量を確認する
cat /sys/fs/cgroup/mycontainer/memory.swap.current
```

### なぜコンテナでスワップを許可するのか {#why-allow-swap-in-containers}

「コンテナではスワップを無効にせよ」という通説は、スワップの計上が高コストで信頼性に欠けていたcgroup v1に由来する。cgroup v2では、スワップの計上は効率的で、よく統合されている。

スワップは「memory.highのスロットリング」と「memory.maxのOOM kill」の間のバッファとなる。スワップがなければ、メモリ使用量の急増はそのままOOMにつながる。スワップがあれば急増はスワップに吸収され、検出して対応する時間が生まれる。

### memory.zswap.max（v5.19以降） {#memoryzswapmax-v519}

[zswap](https://docs.kernel.org/admin-guide/mm/zswap.html)は、ページをスワップに書き出す前にメモリ内で圧縮する。`memory.zswap.max`ファイルは、cgroupがzswapプールに格納できる圧縮データの量を制限する。

```bash
# このcgroupのzswap使用量を200MBに制限する
echo 200M > /sys/fs/cgroup/mycontainer/memory.zswap.max

# zswapの使用量を確認する
cat /sys/fs/cgroup/mycontainer/memory.zswap.current
```

これにより、単一のコンテナがzswapプール全体を埋め尽くし、ほかのコンテナのページを低速なディスクスワップへ書き出させることを防げる。cgroupごとのzswap制限は[commit f4840ccfca25](https://git.kernel.org/linus/f4840ccfca25)（"zswap: memcg accounting"）で導入された。

## ベストプラクティス {#best-practices}

### 制限のサイズ決め {#sizing-limits}

1. **まずプロファイルする**: 現実的な負荷でアプリケーションを実行し、`memory.peak`（v1では`memory.max_usage_in_bytes`）を観察する。`memory.max`を観測したピークの1.25〜1.5倍に設定する。

2. **memory.highをmemory.maxより下に設定する**: `memory.high`と`memory.max`の間に10〜20%の差を設けると、カーネルはkillする前にスロットリングと回収を行う余地を得る。

    ```bash
    # アプリのピークが800MBの場合:
    echo 1200M > /sys/fs/cgroup/mycontainer/memory.max   # ピークの1.5倍
    echo 1000M > /sys/fs/cgroup/mycontainer/memory.high  # スロットリング領域
    ```

3. **ページキャッシュを考慮する**: ワークロードがI/O中心なら、ページキャッシュのための余裕を加える。負荷をかけた状態で`memory.stat`の`file`フィールドを確認する。

4. **重要なサービスにはmemory.lowを使う**: コンテナを回収させてはならない場合（例えばデータベースのバッファプール）は、`memory.low`または`memory.min`を設定する。

### OOMを穏やかに処理する {#handling-oom-gracefully}

**`memory.oom.group`を使う**（[commit 3d8b38eb81ca](https://git.kernel.org/linus/3d8b38eb81ca)、"mm, oom: introduce memory.oom.group"）: コンテナのOOMが起きたとき、デフォルトの挙動では単一のプロセスがkillされる。マルチプロセスのコンテナ（例えばワーカーを持つWebサーバー）では、1つのプロセスをkillするとコンテナが壊れた状態で残ることが多い。`memory.oom.group`はcgroup内のすべてのプロセスをまとめてkillし、クリーンな再起動を保証する。

```bash
# OOM時にcgroup内のすべてのプロセスをkillする（v4.19以降）
echo 1 > /sys/fs/cgroup/mycontainer/memory.oom.group
```

**systemdやコンテナランタイムの再起動ポリシーを使う**: OOMは常に防げるわけではないので、コンテナがクリーンに再起動できるようにしておくこと。

```bash
# Docker: OOM時に常に再起動する
docker run --restart=always --memory=1g myimage

# Kubernetes: デフォルトのrestartPolicyはAlways
```

**`memory.events`を監視する**: `oom`と`oom_kill`カウンタにアラートを設定する。カウントが増えていれば、制限が低すぎるか、アプリケーションにリークがある。

### 本番環境の完全な設定例 {#a-complete-production-setup}

```bash
CGROUP=/sys/fs/cgroup/mycontainer

# ハード制限: 観測したピークの1.5倍
echo 1500M > $CGROUP/memory.max

# OOMの前にスロットリングする
echo 1200M > $CGROUP/memory.high

# 重要なワーキングセットを保護する
echo 500M > $CGROUP/memory.min

# バッファとしていくらかのスワップを許可する
echo 512M > $CGROUP/memory.swap.max

# OOM時にすべてのプロセスをまとめてkillする
echo 1 > $CGROUP/memory.oom.group
```

## 自分で試す {#try-it-yourself}

### memory.highのスロットリングを観察する {#observe-memoryhigh-throttling}

```bash
# 低いソフト制限を持つcgroupを作成する
sudo mkdir -p /sys/fs/cgroup/throttle-test
echo "+memory" | sudo tee /sys/fs/cgroup/cgroup.subtree_control
echo 50M | sudo tee /sys/fs/cgroup/throttle-test/memory.high
echo 100M | sudo tee /sys/fs/cgroup/throttle-test/memory.max

# ゆっくりメモリを割り当てるプロセスを実行する
echo $$ | sudo tee /sys/fs/cgroup/throttle-test/cgroup.procs
python3 -c "
import time
chunks = []
for i in range(80):
    chunks.append(b'x' * 1_000_000)  # 1MB per iteration
    print(f'Allocated {i+1}MB')
    time.sleep(0.1)
"
# 50MB（memory.high）付近で劇的に遅くなり、
# 100MB（memory.max）付近でkillされることに注目
```

### memory.lowによる保護を比較する {#compare-memorylow-protection}

```bash
# 親と2つの子を作成する
sudo mkdir -p /sys/fs/cgroup/parent
echo "+memory" | sudo tee /sys/fs/cgroup/cgroup.subtree_control
echo 200M | sudo tee /sys/fs/cgroup/parent/memory.max
echo "+memory" | sudo tee /sys/fs/cgroup/parent/cgroup.subtree_control

sudo mkdir /sys/fs/cgroup/parent/protected
sudo mkdir /sys/fs/cgroup/parent/unprotected

# 一方の子を保護する
echo 100M | sudo tee /sys/fs/cgroup/parent/protected/memory.low

# 両方でメモリを割り当て、親が200Mの制限に近づいたときに
# どちらが回収されるかを観察する
```

### Dockerコンテナのメモリを監視する {#monitor-a-docker-containers-memory}

```bash
# コンテナを起動する
docker run -d --name memtest --memory=256m nginx

# cgroupのパスを見つける
CGPATH=$(docker inspect memtest --format '{{.HostConfig.CgroupParent}}')
# またはsystemdベースのシステムでは:
PID=$(docker inspect memtest --format '{{.State.Pid}}')
CGROUP=$(cat /proc/$PID/cgroup | grep -o '/.*')

# メモリをリアルタイムで監視する
watch -n 1 "echo '--- usage ---' && \
  cat /sys/fs/cgroup${CGROUP}/memory.current && \
  echo '--- pressure ---' && \
  cat /sys/fs/cgroup${CGROUP}/memory.pressure && \
  echo '--- events ---' && \
  cat /sys/fs/cgroup${CGROUP}/memory.events"

# 後片付け
docker rm -f memtest
```

## 歴史 {#history}

### memory.highのスロットリング（v4.0以降、v4.5で安定） {#memoryhigh-throttling-v40-stable-in-v45}

**作者**: Johannes Weiner

`memory.high`ノブはv4.0でcgroup v2メモリコントローラの一部として追加された（[commit 241994ed8649](https://git.kernel.org/linus/241994ed8649)）が、cgroup v2はv4.5で[本番利用可能と宣言される](https://lwn.net/Articles/679786/)までは実験的な扱いだった。プロセスをkillする代わりに減速させるソフト制限のスロットリングモデルによって、コンテナのメモリ管理ははるかに予測しやすくなった。

### memory.oom.group（v4.19、2018年） {#memoryoomgroup-v419-2018}

**コミット**: [3d8b38eb81ca](https://git.kernel.org/linus/3d8b38eb81ca)（"mm, oom: introduce memory.oom.group"）

**作者**: Roman Gushchin

OOM時にcgroup内のすべてのプロセスをまとめてkillできるようにした。部分的なkillが壊れた状態を残すコンテナベースのデプロイにとって不可欠である。

### cgroupごとのPSI（v4.20、2018年） {#per-cgroup-psi-v420-2018}

**コミット**: [eb414681d5a0](https://git.kernel.org/linus/eb414681d5a0)（"psi: pressure stall information for CPU, memory, and IO"） | [LWN](https://lwn.net/Articles/759658/)

**作者**: Johannes Weiner

cgroupごとのきめ細かな圧迫監視を可能にし、v1の大雑把な`memory.failcnt`カウンタに取って代わった。

### memory.reclaim（v5.19、2022年） {#memoryreclaim-v519-2022}

**コミット**: [94968384dde1](https://git.kernel.org/linus/94968384dde1)（"memcg: introduce per-memcg reclaim interface"） | [LWN](https://lwn.net/Articles/894849/)

**作者**: Shakeel Butt（最初の作者、Google）とYosry Ahmed（Google、その後の改訂を推進）

プロアクティブな回収: メモリ圧迫を待たずに、ユーザー空間がcgroup内で回収を起動できるようにする。MetaとGoogleが先回りのメモリ管理に使っている。

## 参考資料 {#references}

### 主要なコード {#key-code}

| ファイル | 説明 |
|------|-------------|
| [`mm/memcontrol.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c) | メモリコントローラの実装 |
| [`mm/vmpressure.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmpressure.c) | メモリ圧迫の通知 |
| [`kernel/sched/psi.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/psi.c) | Pressure Stall Information |

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/admin-guide/cgroup-v2.rst`](https://docs.kernel.org/admin-guide/cgroup-v2.html) - cgroup v2の正式なリファレンス
- [`Documentation/admin-guide/mm/zswap.rst`](https://docs.kernel.org/admin-guide/mm/zswap.html) - zswapのドキュメント

### LWN記事 {#lwn-articles}

- [Controlling memory with cgroup v2](https://lwn.net/Articles/662925/) - memory.highの設計と動機
- [PSI - Pressure Stall Information](https://lwn.net/Articles/759658/) - cgroupごとの圧迫の追跡
- [User-space OOM handling](https://lwn.net/Articles/894849/) - memory.reclaimとプロアクティブな管理
- [The unified cgroup hierarchy](https://lwn.net/Articles/679786/) - cgroup v2が本番利用可能になるまで

### 関連項目 {#related}

- [メモリcgroup](memcg.md) - memcgの内部の詳細とv1とv2の違い
- [メモリ不足](oom.md) - グローバルなOOM killの経路
- [ページ回収](reclaim.md) - カーネルがメモリをどのように回収するか
- [スワップ](swap.md) - スワップサブシステムの内部
- [用語集](glossary.md) - cgroup、OOM、PSIの定義

## 参考文献 {#further-reading}

- [コンテナのメモリ制限](container-memory-limits.md) — `memory.high`のスロットリングからcgroupのOOM kill、そして回復までの一連の流れ
- [memory.statフィールドリファレンス](memory-stat.md) — `/sys/fs/cgroup/*/memory.stat`のすべてのフィールドの説明と診断のレシピ
- [メモリcgroup](memcg.md) — memcgの課金モデルの仕組みとv1とv2の違い
- [OOMログの読み方](reading-oom-log.md) — コンテナがOOM killされたときのdmesg出力の解析方法
- [`Documentation/admin-guide/cgroup-v2.rst`](https://docs.kernel.org/admin-guide/cgroup-v2.html) — `memory.high`、`memory.max`、`memory.min`、`memory.events`を含む、cgroup v2のすべてのインターフェースファイルに関するカーネルの正式なリファレンス
- [LWN: Controlling memory with cgroup v2](https://lwn.net/Articles/662925/) — `memory.high`とソフト制限のスロットリングモデルの設計根拠
- [LWN: PSI — Pressure Stall Information](https://lwn.net/Articles/759658/) — cgroupごとの`memory.pressure`がどのように計算され、どう対応すべきか
- [LWN: User-space OOM handling](https://lwn.net/Articles/894849/) — `memory.reclaim`と、オーケストレータからのプロアクティブなメモリ管理
- [`mm/memcontrol.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c) — メモリコントローラの実装: 課金、スロットリング、OOMの経路
