# systemdとcgroupの統合 {#systemd-and-cgroup-integration}

> systemdがcgroup階層をどう管理するか: slice、scope、service、委譲

## 概要 {#overview}

systemdは、ほぼすべての現代的なLinuxディストリビューションでcgroupマネージャを務めている。systemd 232（2016年）以降、systemdはcgroup v2階層の排他的な所有権を前提としている。すべてのユニットに対してサブツリーを作成し、適切なインターフェースファイルにリソース制限を書き込み、あるサブツリーを管理するユーザー空間のエージェントは1つだけというルールを強制する。systemdが明示的にそのサブツリーを委譲していない限り、他のツールは `/sys/fs/cgroup` に直接書き込むべきではない。

この取り決めは、cgroup v1を悩ませていた問題を解決する。すなわち、複数のマネージャ（Docker、LXC、Kubernetes、systemd自身）が同じcgroupファイルへの書き込みを競い合い、しばしば互いの設定を上書きしていたという問題である。

## ユニットの種類とcgroupへの対応 {#unit-types-and-their-cgroup-mapping}

systemdは、cgroupに対応する3種類のユニットを提供する:

### serviceユニット（`.service`） {#service-units-service}

長時間動作するデーモンである。systemdは `system.slice/<name>.service/` にcgroupを作成し、起動時にデーモンのプロセスをそこへ移す。

```
/sys/fs/cgroup/system.slice/nginx.service/
├── cgroup.procs        ← contains nginx worker PIDs
├── cpu.max             ← set from CPUQuota=
├── memory.max          ← set from MemoryMax=
├── io.weight           ← set from IOWeight=
└── pids.max            ← set from TasksMax=
```

cgroupはサービスの起動時に作成され、停止時に（プロセスが空になった後で）削除される。サービスがクラッシュして再起動した場合は、同じcgroupパスが再利用される。

### scopeユニット（`.scope`） {#scope-units-scope}

外部で起動されたプロセスのグループを、D-Bus（`org.freedesktop.systemd1.Manager.StartTransientUnit`）経由でsystemdに登録したものである。コンテナランタイムは、自分で起動したコンテナを登録するためにscopeを使う。ユーザーのscopeのcgroupパスは `user.slice/user-<uid>.slice/<name>.scope/` である。

serviceと異なり、systemdはscope内のプロセスを起動も停止もしない。cgroupを作成して追跡するだけである。プロセスについては、scopeを登録したランタイムが責任を持つ。

### sliceユニット（`.slice`） {#slice-units-slice}

serviceとscopeをまとめる階層的なグループ化の入れ物である。sliceはプロセスを直接保持せず、子のslice、service、scopeを保持する。そのcgroupは、すべての子が共有するリソースの外枠を提供する。

デフォルトのsliceは次のとおりである:

- `system.slice` — すべてのシステムサービス
- `user.slice` — すべてのユーザーセッション
- `machine.slice` — `systemd-machined` が管理するVMとコンテナ

## デフォルトのcgroupレイアウト {#the-default-cgroup-layout}

```
/sys/fs/cgroup/
├── cgroup.controllers
├── cgroup.subtree_control
├── init.scope/            ← PID 1 (systemd) itself
│   └── cgroup.procs       ← contains 1
├── system.slice/          ← system services
│   ├── cgroup.subtree_control
│   ├── nginx.service/
│   │   ├── cgroup.procs
│   │   ├── cpu.max
│   │   └── memory.max
│   ├── sshd.service/
│   │   └── cgroup.procs
│   └── containerd.service/
│       └── cgroup.procs   ← contains containerd (not containers)
├── user.slice/            ← user sessions
│   └── user-1000.slice/
│       ├── session-1.scope/
│       │   └── cgroup.procs   ← login shell and its children
│       └── user@1000.service/ ← user systemd instance
│           └── cgroup.procs
└── machine.slice/         ← VMs / containers via machined
```

systemdは、`system.slice` と `user.slice` が常に存在すること、そして `cgroup.subtree_control` に `/etc/systemd/system.conf` の `DefaultCPUAccounting=`、`DefaultMemoryAccounting=` などに列挙されたコントローラが設定されていることを保証する。

## ユニットファイルでのリソース制限 {#resource-limits-in-unit-files}

ユニットファイルのプロパティは、cgroup v2のインターフェースファイルに直接対応する:

```ini
[Service]
# cpu.max = クォータ 周期（マイクロ秒）
CPUQuota=50%          # cpu.max = "500000 1000000"  (50% of one CPU)

# memory.max（ハードな制限。超過すると OOM kill が発生する）
MemoryMax=512M        # memory.max = 536870912

# memory.high（ソフトな制限。OOM の前に回収を引き起こす）
MemoryHigh=400M       # memory.high = 419430400

# memory.swap.max（スワップ使用量を制限する）
MemorySwapMax=0       # memory.swap.max = 0  (no swap)

# io.weight（比例配分の I/O スケジューリングの重み、範囲 1-10000）
IOWeight=100          # io.weight = 100

# io.max（デバイスごとのハードな帯域制限）
IOReadBandwidthMax=/dev/sda 50M   # io.max = "8:0 rbps=52428800"
IOWriteBandwidthMax=/dev/sda 25M  # io.max = "8:0 wbps=26214400"

# pids.max
TasksMax=100          # pids.max = 100

# cpuset.cpus（特定の CPU に固定する）
AllowedCPUs=0-3       # cpuset.cpus = "0-3"

# cpuset.mems（特定の NUMA ノードに固定する）
AllowedMemoryNodes=0  # cpuset.mems = "0"
```

systemdがこれらを書き込む際、実行中の変更にはD-Bus経由で `sd_bus_call_method()` を使い、ユニットの起動時には内部ヘルパー `cg_set_attribute()`（systemdソースの `src/basic/cgroup-util.c`）を介してcgroupのインターフェースファイルに直接書き込む。

## 実行中のプロパティ変更 {#live-property-changes}

`systemctl set-property` は、実行中のユニットを再起動せずにリソースの変更を適用する:

```bash
# nginx の CPU クォータを 25% に下げる
systemctl set-property nginx.service CPUQuota=25%

# メモリを制限する
systemctl set-property nginx.service MemoryMax=256M

# 変更は次のように行われる:
# 1. D-Bus 経由で PID 1（systemd）に送られる
# 2. systemd が値を /sys/fs/cgroup/system.slice/nginx.service/cpu.max に書き込む
# 3. 必要に応じて /etc/systemd/system/nginx.service.d/50-CPUQuota.conf に永続化される

# 変更が反映されたことを確認する:
cat /sys/fs/cgroup/system.slice/nginx.service/cpu.max
# 250000 1000000
```

`--runtime` フラグを付けると、変更は次の再起動までだけ適用される（`.conf` ファイルは作成されない）:

```bash
systemctl set-property --runtime nginx.service MemoryMax=256M
```

## 委譲 {#delegation}

ユニットファイルで `Delegate=yes` を設定すると、systemdはそのサービスのcgroupを委譲済みとしてマークする。systemdは引き続きcgroupを作成し、最上位の制限を設定するが、サブツリーの中には立ち入らない。つまり、`cgroup.subtree_control` や委譲ポイントより下のファイルには一切触れない。サービス自身が、自分のサブ階層を管理する責任を負う。

```ini
[Service]
Delegate=yes
# あるいは特定のコントローラだけを委譲する:
Delegate=cpu memory pids
```

これは次のものが使っている:

- **containerd** / **Docker** — `containerd.service/` の下にコンテナごとのサブcgroupを作成する
- **Kubernetes kubelet** — `kubelet.service/` または専用のsliceの下にPodごとのcgroupを作成する
- **ユーザーのsystemdインスタンス** — `user@1000.service` は委譲されており、ユーザーごとのsystemdが `user.slice/user-1000.slice/` を管理できる

`Delegate=yes` の場合、systemdは `chown(2)` でcgroupディレクトリの所有者をサービスの `User=` のUIDに変更する。これにより、非特権プロセスがrootなしで `cgroup.subtree_control` に書き込み、子のcgroupサブディレクトリを作成できるようになる。`cgroup.delegate` というカーネルのインターフェースファイルは存在しない。委譲は純粋にcgroupディレクトリのファイルシステム上の所有権の問題である。

委譲を使うコンテナランタイムの正しいパターンは次のとおりである:

```bash
# systemd が作成済み: /sys/fs/cgroup/system.slice/containerd.service/
# Delegate=yes は containerd がこのサブツリーを所有することを意味する。

# containerd がコンテナ abc123 用の cgroup を作成する:
mkdir /sys/fs/cgroup/system.slice/containerd.service/abc123/

# プロセスを cgroup に移す「前に」制限を書き込む:
echo "512M" > /sys/fs/cgroup/system.slice/containerd.service/abc123/memory.max
echo "100"  > /sys/fs/cgroup/system.slice/containerd.service/abc123/pids.max

# その後でコンテナの init プロセスを移す:
echo $CONTAINER_PID > /sys/fs/cgroup/system.slice/containerd.service/abc123/cgroup.procs
```

## 一時ユニット {#transient-units}

`systemd-run` は、ユニットファイルを書かずに、その場で一時的なscopeやserviceのcgroupを作成する:

```bash
# myslice.slice の下の一時 scope で、リソース制限付きでコマンドを実行する
systemd-run --scope \
    --slice=myslice.slice \
    --property=MemoryMax=256M \
    --property=CPUQuota=50% \
    -- stress --vm 2 --vm-bytes 200M

# oneshot のサービスを実行する（完了を待つ）
systemd-run --service-type=oneshot \
    --property=TasksMax=20 \
    -- /usr/local/bin/batch-job.sh

# 生成された cgroup を確認する:
systemd-cgls /sys/fs/cgroup/myslice.slice/
```

一時ユニットは、D-Bus経由の `org.freedesktop.systemd1.Manager.StartTransientUnit` で登録される。ユニットとそのcgroupは、プロセスが終了すると自動的に削除される。

## 可観測性 {#observability}

```bash
# systemd のユニット名を注記した cgroup ツリー全体を表示する:
systemd-cgls

# 出力例:
# Control group /:
# -.slice
# ├─system.slice
# │ ├─nginx.service
# │ │ ├─1234 nginx: master process /usr/sbin/nginx
# │ │ └─1235 nginx: worker process
# │ └─sshd.service
# │   └─5678 sshd: accepting connections
# └─user.slice
#   └─user-1000.slice
#     └─session-1.scope
#       ├─9012 -bash
#       └─9013 systemd-cgls

# cgroup ごとのリソース使用量をリアルタイム表示する（cgroup 版の top）:
systemd-cgtop

# ユニットのリソース設定を表示する:
systemctl show nginx.service | grep -E 'CPU|Memory|IO|Tasks'
# CPUQuota=50%
# MemoryMax=536870912
# TasksMax=100

# cgroup の生のインターフェースファイルを確認する:
cat /sys/fs/cgroup/system.slice/nginx.service/cpu.stat
cat /sys/fs/cgroup/system.slice/nginx.service/memory.current
cat /sys/fs/cgroup/system.slice/nginx.service/io.stat
```

## systemdがコントローラを有効にする方法 {#how-systemd-enables-controllers}

systemdは `/proc/cgroups` を読んで利用可能なコントローラを調べ、`cgroup.subtree_control` を使ってそれらを階層の下へ伝播させる。`/etc/systemd/system.conf` の `DefaultCPUAccounting=yes` / `DefaultMemoryAccounting=yes` / `DefaultIOAccounting=yes` の設定項目が、systemdがグローバルに有効にするコントローラを制御する。

```bash
# systemd が有効にしたコントローラを確認する:
cat /sys/fs/cgroup/cgroup.subtree_control
# cpuset cpu io memory hugetlb pids

# サービスの cgroup で利用可能なコントローラを確認する:
cat /sys/fs/cgroup/system.slice/cgroup.subtree_control
```

該当する階層の `cgroup.subtree_control` にコントローラが列挙されていない場合、`memory.max` のようなインターフェースファイルは子のcgroupに存在しない。これは、コントローラがデフォルトで無効になっている場合によくある混乱の原因である。

## 参考文献 {#further-reading}

- [cgroup v2のアーキテクチャ](cgroup-v2.md) — 統一階層、struct cgroup、subtree_control
- [リソースコントローラ](cgroup-controllers.md) — cpu、memory、io、pidsのインターフェースファイルの詳細
- [コンテナの隔離](container-isolation.md) — Docker/containerdが委譲をどう使うか
- [cgroup事件簿](war-stories.md) — 委譲における競合状態とv1→v2移行の思わぬ落とし穴
- `man systemd.resource-control(5)` — リソース制御プロパティの完全な一覧
- `man systemd-run(1)` — 一時ユニットの作成
- systemdソースの `src/core/cgroup.c` — cgroup管理の実装
- `Documentation/admin-guide/cgroup-v2.rst` — カーネルのcgroup v2ドキュメント
