# cgroup 事件簿 {#cgroup-war-stories}

> OOM kill の嵐、CPU スロットリングの思わぬ挙動、cgroup v1 からの移行の苦労

ここで取り上げるのは、cgroup が関わる本番環境のインシデントの、実在する5つのカテゴリである。障害のパターンはわかりにくい。cgroup はカーネルレベルでポリシーを強制するが、設定したポリシーとカーネルが実際に強制するポリシーとの相互作用は、しばしば期待どおりにならない。

---

## 1. CFS 帯域制御の thundering herd {#1-the-cfs-bandwidth-thundering-herd}

**状況**: ノードあたり数十個の Pod が動く Kubernetes クラスタ。各 Pod は spec で CPU の request/limit が設定されており、kubelet がそれを `cpu.max`（CFS クォータ）に変換していた。CFS の周期はデフォルトの 100ms だった。

**何が起きたか**: すべての Pod の CFS クォータは、同じ瞬間、つまり各 100ms 周期の `t=0` に補充される。CPU バウンドな Pod はクォータに達してスリープし、`t=100ms` で再び実行可能になる。同じノード上の全 Pod が一斉にである。スケジューラのランキューは 100ms ごとに、CPU を奪い合う数十個の実行可能タスクで急増した。アプリケーションレベルのリクエストレイテンシは、ちょうど 100ms 間隔でスパイクするのこぎり波のパターンを示した。

**カーネルの仕組み**: CFS 帯域制御（`kernel/sched/fair.c`）は、周期ごとのトークンバケットを使って cgroup ごとのクォータを管理する。`cpu.max` が `200000 100000`（100ms 周期あたり 200ms の CPU = 2 CPU）に設定されている場合、cgroup は周期あたり 200ms のクォータを得る。それを使い切ると、グループはスロットリングされる。`cpu.stat` の `throttled_usec` カウンタは、累積したスロットリング時間を示す:

```bash
cat /sys/fs/cgroup/system.slice/myapp.service/cpu.stat
# usage_usec 4523190
# user_usec 3210456
# system_usec 1312734
# nr_periods 45231
# nr_throttled 12048          ← 45k 周期のうち 12k 周期でスロットリングされた
# throttled_usec 1204800000   ← 累積 1.2 秒のスロットリング時間
```

**診断**: 鍵となったのはレイテンシスパイクの周期性である。`nr_throttled` / `nr_periods` でスロットリング率がわかる。高い比率（> 5-10%）と、ちょうど周期の境界でのレイテンシスパイクが組み合わさっていれば、CFS の thundering herd である。

**修正**:

選択肢1 — CFS の周期を長くして、同期の頻度を下げる。100ms から 500ms に変更する場合:
```bash
# cpu.max の書式: クォータ 周期（どちらもマイクロ秒）
echo "1000000 500000" > /sys/fs/cgroup/myapp/cpu.max   # 2 CPU、500ms 周期
```

選択肢2 — `cpu.max`（ハードなクォータ）の代わりに `cpu.weight`（比例配分）を使う。`cpu.weight` はスロットリングを一切行わず、システムが競合しているときに CPU の配分を減らすだけである。レイテンシに敏感なワークロードでは、ほぼ常に比例配分のほうがハードなクォータより優れている:
```bash
echo "200" > /sys/fs/cgroup/myapp/cpu.weight   # デフォルトの 2 倍の配分
```

Linux カーネル 5.14 のコミット `(39f450ac4b06)` では、バックグラウンドの cgroup で SCHED_IDLE をサポートするために `cpu.idle` が追加された。これも優先度の低いバッチ処理向けの別の選択肢である。

---

## 2. OOM kill のループ {#2-the-oom-kill-loop}

**状況**: `memory.max=512M` の cgroup 内で動作する、コンテナ化された Java アプリケーション。コンテナは、終了時に JVM を再起動するスーパーバイザプロセス（これも cgroup 内にある）によって管理されていた。

**何が起きたか**: JVM はヒープを徐々に約 510M まで拡大した。一時的なスパイク（GC が十分なメモリを解放できない、大きなデータセットのロードなど）によって cgroup が `memory.max` に達した。カーネルの OOM killer は `out_of_memory()` → `oom_kill_process()` を呼び出し、`oom_score_adj` で重み付けされた最大のプロセスとして JVM プロセスを選んで kill した。スーパーバイザが JVM を再起動した。JVM は再び 510M まで成長した。スパイクが再発した。また kill された。このループは約30秒ごとに6時間にわたって続き、誰も気づかなかった。コンテナ自体は一度も kill されず、内部の個々のプロセスだけが kill されたため、Kubernetes が Pod を再起動することはなかったのである。

**カーネルの仕組み**: cgroup が `memory.max` を超えると、`try_charge_memcg()` が `mem_cgroup_oom()` を呼び、それが `mem_cgroup_out_of_memory()` を呼び出す。デフォルトでは、OOM killer は cgroup 内で `oom_score` が最も高いプロセスを選ぶ（cgroup 全体ではない）。コンテナランタイムがコンテナの init プロセスにだけ `oom_score_adj` を設定している場合、（デフォルトの `oom_score_adj=0` の）JVM が代わりに選ばれることがある。cgroup は動き続け、OOM イベントは次の場所で確認できる:

```bash
# cgroup ごとの OOM イベント:
cat /sys/fs/cgroup/containers/c1/memory.events
# low 0
# high 0
# max 47            ← memory.max に 47 回達した
# oom 47            ← OOM イベントが 47 回
# oom_kill 47       ← OOM killer が 47 個のプロセスを kill した
# oom_group_kill 0  ← cgroup 全体として kill されたことはない
```

**診断**: 最初に見るべき場所は `memory.events` である。`oom` > 0 は、その cgroup で OOM kill が起きたことを意味する。`oom_group_kill` = 0 は、cgroup 全体（全プロセス）が kill されたことは一度もないことを意味する。

**修正**:

選択肢1 — `memory.high` をソフトな上限として使う。メモリ使用量が `memory.high` を超えると、カーネルは `memory.max` に達する前に割り当てをスロットリングし、積極的な回収を引き起こす。JVM は CPU サイクルを得られるが速度が落ち、GC が回収する時間が生まれる:
```bash
echo "400M" > /sys/fs/cgroup/containers/c1/memory.high   # ソフトな制限
echo "512M" > /sys/fs/cgroup/containers/c1/memory.max    # ハードな制限
```

選択肢2 — `memory.oom.group` を有効にする。`1` に設定すると、OOM イベント発生時に OOM killer は（最大のプロセスだけでなく）cgroup 内のすべてのプロセスを kill する。これによりスーパーバイザも kill され、再起動のループを防げる:
```bash
echo "1" > /sys/fs/cgroup/containers/c1/memory.oom.group
```

選択肢3 — スパイクに対応できるよう `memory.max` を増やすか、JVM の `-Xmx` を cgroup の制限より十分低く設定する。

---

## 3. cgroup v1 から v2 への移行の落とし穴 {#3-the-cgroup-v1-to-v2-migration-surprise}

**状況**: ある企業が、コンテナプラットフォームを Ubuntu 20.04 上の Docker 19（cgroup v1 ハイブリッドモード）から、Ubuntu 22.04 上の containerd + runc（cgroup v2 のみ）に移行した。既存のコンテナイメージ、監視エージェント、スクリプトは開発環境ではすべて問題なく動いたが、本番環境では壊れていた。

**何が起きたか**: 複数の種類の障害が同時に発生した:

**種類1 — インターフェースファイルの欠如**: 次の場所からメモリ使用量を読み取るスクリプトは:
```
/sys/fs/cgroup/memory/<container-id>/memory.usage_in_bytes   ← v1
```
そのパスが存在しないことに気づいた。cgroup v2 で同等のものは次のとおりである:
```
/sys/fs/cgroup/<container-id>/memory.current   ← v2
```

**種類2 — スワップのアカウンティングの欠如**: cgroup v1 には、メモリ+スワップの合算の制限を表す `memory.memsw.limit_in_bytes` が別途あった。cgroup v2 ではこれらが `memory.swap.max` の下にまとめられている:

| v1 | v2 |
|----|----|
| `memory.limit_in_bytes` | `memory.max` |
| `memory.soft_limit_in_bytes` | `memory.high` |
| `memory.memsw.limit_in_bytes` | 直接の同等物はない: v2 の `memory.max` は RAM のみを制限し、`memory.swap.max` はスワップのみを制限する（合算ではない） |
| `memory.usage_in_bytes` | `memory.current` |
| `memory.memsw.usage_in_bytes` | `memory.current` + `memory.swap.current`（計算で求める。単一のファイルではない） |

**種類3 — コントローラが有効になっていない**: いくつかのホストでは、`memory.max` は存在するのに、書き込みが黙って無視されていた。根本原因は、コンテナの親 cgroup の `cgroup.subtree_control` に memory コントローラが列挙されていなかったことである。cgroup v2 では、コントローラのインターフェースファイルが子に現れるためには、階層の各レベルでそのコントローラが有効になっていなければならない。v2 ではカーネルのブートパラメータ `cgroup_enable=memory` は不要だが（memory コントローラは組み込まれている）、subtree_control の連鎖は完全でなければならない:

```bash
# 各レベルでコントローラが有効になっていることを確認する:
cat /sys/fs/cgroup/cgroup.subtree_control
# cpuset cpu io memory hugetlb pids     ← "memory" を含んでいなければならない

cat /sys/fs/cgroup/system.slice/cgroup.subtree_control
# cpuset cpu io memory pids              ← ここでも "memory" を含んでいなければならない
```

**必要だった修正**:
1. すべての監視とツールを cgroup v2 のパスを使うように更新する
2. `/etc/systemd/system.conf` で systemd の `DefaultMemoryAccounting=yes` が設定されていることを確認する
3. `grep -r 'sys/fs/cgroup/memory' /usr/local/` を使って、v1 のパスを使うスクリプトを洗い出す
4. Docker / containerd の設定を、cgroupv2 ドライバを使うように更新する

`libcgroup` と `cgroup-tools` パッケージは、今でもデフォルトで v1 のパスを使う。ほとんどは更新か置き換えが必要だった。

---

## 4. pids コントローラと Java のスレッドの混乱 {#4-the-pids-controller-and-java-thread-confusion}

**状況**: Pod の spec で `resources.limits` を設定した、Java 17 アプリケーションを動かす Kubernetes の Pod。kubelet は `TasksMax` を Pod の cgroup の `pids.max=100` に変換していた。

**何が起きたか**: Java アプリケーションは正常に起動したが、負荷がかかるとスレッドプールが拡大した。スレッド数が100に達すると、`fork()` の呼び出し（サブプロセスを実行するための `Runtime.exec()` が引き起こすものを含む）が `-EAGAIN` で失敗し始めた。JVM は `clone()` からの `EAGAIN` を `IOException` として捕捉し、ログに記録して処理を続けた。しかしサブプロセスの実行は黙って失敗し、データパイプラインの障害を引き起こした。

`pids.max` のカウンタが数えるのは、プロセスではなく **タスク** である。Linux カーネルでは、スレッドはタスク（`struct task_struct`）である。100スレッドを持つ JVM は `pids.max=100` の100枠すべてを消費し、新しいスレッドやサブプロセスのための枠は残らない。運用者は、`pids.max` が制限するのはプロセス（PID）の数だけで、スレッドの数ではないと思い込んでいた。

**確認**:

```bash
# cgroup の現在のタスク数を確認する:
cat /sys/fs/cgroup/kubepods/pod-abc123/pids.current
# 98

# 制限を確認する:
cat /sys/fs/cgroup/kubepods/pod-abc123/pids.max
# 100

# ps はカーネルのタスク（スレッド）ではなくプロセスを表示するので、少なく数えてしまう:
ps aux | grep java | wc -l
# 3    ← JVM プロセスといくつかのラッパーしか表示されない

# /proc/PID/status は全スレッドを示す:
cat /proc/$(pgrep java)/status | grep Threads
# Threads: 97
```

**修正**: アプリケーションのスレッド数に対応できるよう `pids.max` を増やす。200スレッドのスレッドプールにシステムスレッドを加えた Java アプリケーションでは、`pids.max=250` 以上が必要になるかもしれない:

```bash
echo "500" > /sys/fs/cgroup/kubepods/pod-abc123/pids.max
```

Kubernetes では、`TasksMax` を設定するか、ノードの `--pods-per-core` の計算に頼る。`pids.events` ファイルは、制限に達した回数を示す:

```bash
cat /sys/fs/cgroup/kubepods/pod-abc123/pids.events
# max 47    ← 制限に 47 回達した
```

監視では、プロセスが失敗し始める前に検知できるよう、`pids.current / pids.max > 0.8` でアラートを出すとよい。

---

## 5. 委譲された cgroup での競合 {#5-the-delegated-cgroup-race}

**状況**: systemd のサービスユニットで `Delegate=yes` を使うコンテナランタイム。コンテナの起動時に、ランタイムは次のことを行っていた:
1. 委譲されたサブツリーの下に子の cgroup を作成する
2. `cgroup.procs` に書き込んで、コンテナの init プロセスを新しい cgroup に移す
3. `memory.max` に書き込んで、コンテナのメモリ制限を設定する

**何が起きたか**: 高負荷のノードでは、手順3がときどき数百ミリ秒遅れた。手順2と3の間、コンテナの init プロセスはメモリ制限なしで動いていた。委譲されたサブツリーのルートにいて、そこはランタイム自身の寛大な `memory.max` を継承していた（あるいはまったく制限がなかった）。その間に、メモリを大量に消費するコンテナの init スクリプトが、意図したコンテナの制限をはるかに超えて割り当てを行えた。

ほとんどの場合、この時間窓はマイクロ秒単位で問題にならなかった。しかし、コンテナランタイムの goroutine が2つの書き込みの間でプリエンプトされるような過負荷のノードでは、時間窓が数百ミリ秒に伸びた。256M を使うよう設計されたコンテナが、制限が書き込まれる前の init の間に 2G を割り当てることができた。

**カーネルの振る舞い**: `cgroup.procs` への書き込みは、そのプロセスの cgroup への所属に関してはアトミックだが、「cgroup の作成 + 制限の設定 + プロセスの移動」にまたがるトランザクションは存在しない。カーネルには、すべての制限をアトミックに設定してからプロセスを受け入れる仕組みがない。これは cgroup インターフェースの本質的な設計上の性質である。

**修正**: プロセスを cgroup に移す **前に**、必ずリソース制限を書き込む。正しい手順は次のとおりである:

```bash
# 1. cgroup ディレクトリを作成する
mkdir /sys/fs/cgroup/containerd.service/abc123/

# 2. 最初にすべての制限を書き込む（この手順内での順序は問わない）:
echo "268435456" > /sys/fs/cgroup/containerd.service/abc123/memory.max   # 256M
echo "0"         > /sys/fs/cgroup/containerd.service/abc123/memory.swap.max
echo "100"       > /sys/fs/cgroup/containerd.service/abc123/pids.max
echo "200000 1000000" > /sys/fs/cgroup/containerd.service/abc123/cpu.max

# 3. その後で初めてプロセスを移す:
echo $CONTAINER_INIT_PID > /sys/fs/cgroup/containerd.service/abc123/cgroup.procs
```

runc ランタイムはこの教訓を学び、現在では `clone()` を呼ぶ前や PID を移す前に制限を書き込むようになっている。OCI ランタイム仕様は、コンテナのプロセスが実行を開始する前にリソース制限を適用することをランタイムに要求している。

副次的な修正: 中間の cgroup（プロセスが移される前のもの）を、すでに厳しい制限を持つ制約的な親 cgroup の下に置く。そうすれば、競合が起きても影響が限定される。

---

## インシデントに共通するパターン {#patterns-across-incidents}

これら5つのインシデントには、共通の診断手順がある:

```bash
# 1. cgroup がスロットリングされているか確認する:
cat /sys/fs/cgroup/.../cpu.stat | grep throttled

# 2. OOM イベントを確認する:
cat /sys/fs/cgroup/.../memory.events

# 3. pids の制限に達したか確認する:
cat /sys/fs/cgroup/.../pids.events

# 4. 実際にアクティブなコントローラを確認する:
cat /sys/fs/cgroup/.../cgroup.controllers
cat /sys/fs/cgroup/.../cgroup.subtree_control

# 5. 現在のリソース使用量と制限を比較する:
cat /sys/fs/cgroup/.../memory.current
cat /sys/fs/cgroup/.../pids.current
```

そして共通の原則がある。**cgroup は単一の書き込みの中ではアトミックにポリシーを強制するが、複数の書き込みにまたがってはそうしない。** プロセスを追加する前に制限を設定すること。設定した内容だけでなく、カーネルが実際に何をしているかを観察するために、`memory.events` と `cpu.stat` を使うこと。

## 参考文献 {#further-reading}

- [cgroup v2 のアーキテクチャ](cgroup-v2.md) — 統一階層、subtree_control の伝播
- [リソースコントローラ](cgroup-controllers.md) — cpu.max、memory.max、pids.max のインターフェースファイル
- [systemd と cgroup の統合](systemd-cgroups.md) — 委譲、ユニットのプロパティ、実行中の変更
- [io コントローラ](io-cgroup.md) — io.latency の SLA とライトバックの帰属
- [コンテナの隔離](container-isolation.md) — runc、OCI 仕様、コンテナのライフサイクル
- `kernel/sched/fair.c` — CFS 帯域コントローラとスロットリングの実装
- `mm/memcontrol.c` — memory コントローラの OOM 処理
- `kernel/cgroup/pids.c` — pids コントローラの実装
