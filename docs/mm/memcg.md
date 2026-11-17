# メモリcgroup（memcg） {#memory-cgroups-memcg}

> コンテナのメモリ制限と計上

## memcgとは何か {#what-is-memcg}

[メモリcgroup](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory)（memcg）は、プロセスのグループに対してメモリ使用量の制限と追跡を可能にする。これはコンテナのメモリ制限（Docker、Kubernetes、systemd）の土台である。

```
Container A (limit: 1GB)      Container B (limit: 2GB)
┌─────────────────────┐      ┌─────────────────────┐
│  Process 1          │      │  Process 3          │
│  Process 2          │      │  Process 4          │
│  [shared memory]    │      │  [shared memory]    │
└─────────────────────┘      └─────────────────────┘
         │                            │
         └────────────┬───────────────┘
                      │
              Memory Controller
              (tracks & enforces)
```

## cgroup v1とv2の比較 {#cgroup-v1-vs-v2}

Linuxには2つのcgroup実装がある。

| 機能 | v1 | v2 |
|---------|----|----|
| 階層 | 複数（コントローラごとに1つ） | 単一の統合階層 |
| メモリとIOの連携 | 困難 | 統合されている |
| 逼迫の通知 | 限定的 | PSI（Pressure Stall Info） |
| デフォルト（現代のシステム） | レガシー | 推奨 |

新規の導入には**[v2が推奨される](https://lwn.net/Articles/679786/)**（v4.5で本番利用可能と宣言された）。このドキュメントはv2に焦点を当てる。

```bash
# どちらのバージョンがマウントされているか確認
mount | grep cgroup

# v1: /sys/fs/cgroup/memory, /sys/fs/cgroup/cpu など
# v2: /sys/fs/cgroup (統合)
```

## 基本操作 {#basic-operations}

### cgroupを作成する {#create-a-cgroup}

```bash
# 新しいcgroupを作成
mkdir /sys/fs/cgroup/mygroup

# プロセスをその中へ移動
echo $PID > /sys/fs/cgroup/mygroup/cgroup.procs

# メモリコントローラを有効化
echo "+memory" > /sys/fs/cgroup/cgroup.subtree_control
```

### メモリ制限を設定する {#set-memory-limit}

```bash
# 500MBの制限を設定 (cgroup v2)
echo 500M > /sys/fs/cgroup/mygroup/memory.max

# ソフトリミット (回収の目標) を設定
echo 400M > /sys/fs/cgroup/mygroup/memory.high
```

### 使用量を見る {#view-usage}

```bash
# 現在のメモリ使用量
cat /sys/fs/cgroup/mygroup/memory.current

# 詳細な統計
cat /sys/fs/cgroup/mygroup/memory.stat
```

## メモリ制限（v2） {#memory-limits-v2}

| ファイル | 説明 |
|------|-------------|
| `memory.max` | ハードリミット。超えるとOOM killされる。 |
| `memory.high` | ソフトリミット。スロットリングし、積極的に回収する。 |
| `memory.low` | 保護。逼迫時に[ベストエフォートで保持する](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory-interface-files)。 |
| `memory.min` | ハードな保護。[これ以下には決して回収しない](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory-interface-files)。 |

```
                    memory.max
                        │
        OOM kill ──────>│
                        │
                    memory.high
                        │
     Aggressive ───────>│
      reclaim           │
                        │
                    memory.low
                        │
    Protected ─────────>│ (best-effort)
                        │
                    memory.min
                        │
     Cannot ───────────>│ (hard guarantee)
      reclaim
```

### 例: コンテナの制限 {#example-container-limits}

```bash
# 本番コンテナの設定
echo 1G > /sys/fs/cgroup/container/memory.max    # ハードリミット
echo 800M > /sys/fs/cgroup/container/memory.high # スロットリング開始
echo 200M > /sys/fs/cgroup/container/memory.low  # この量を保護
```

## 計上されるもの {#whats-accounted}

memcgが追跡するもの:

| 種類 | 計上されるか | 備考 |
|------|-----------|-------|
| 匿名ページ | はい | ヒープ、スタック、mmap(MAP_ANONYMOUS) |
| ファイルキャッシュ | はい | ファイルの読み込みによるページ |
| slab（kmem） | はい（v2） | このcgroupのためのカーネルオブジェクト |
| huge page | 別扱い | `hugetlb`コントローラ経由 |
| カーネルスタック | はい（v2） | タスクごとのカーネルスタック |

### メモリの統計 {#memory-statistics}

```bash
cat /sys/fs/cgroup/mygroup/memory.stat

# 主なフィールド:
# anon        - 匿名メモリ
# file        - ファイルキャッシュ
# slab        - カーネルのslabオブジェクト
# sock        - ネットワークソケットのバッファ
# pgfault     - ページフォールト
# pgmajfault  - メジャーページフォールト (ディスクI/O)
```

## cgroupごとの回収 {#per-cgroup-reclaim}

cgroupが制限に近づくと、まずそのcgroupの中で回収が行われる。

```
Global memory OK
        │
        v
Cgroup A at memory.high
        │
        v
Reclaim from Cgroup A only
(other cgroups unaffected)
```

### memory.reclaim（[v5.19+](https://lwn.net/Articles/894849/)） {#memoryreclaim-v519}

先回りして回収をトリガする（[commit 94968384dde1](https://git.kernel.org/linus/94968384dde1)）。

```bash
# cgroupから100MBを回収
echo 100M > /sys/fs/cgroup/mygroup/memory.reclaim
```

次の用途に役立つ。

- 負荷のスパイクに備えた事前のウォームアップ
- マイグレーション前のメモリ削減
- 回収の挙動のテスト

## OOMの処理 {#oom-handling}

cgroupが`memory.max`を超えると、次のようになる。

1. カーネルがcgroup内で回収を試みる
2. 不十分なら、cgroup OOMをトリガする
3. OOM killerがcgroup内のプロセスを選ぶ
4. 選ばれたプロセスがkillされる

```bash
# OOMイベントを見る
cat /sys/fs/cgroup/mygroup/memory.events

# フィールド:
# oom        - OOMイベントの回数
# oom_kill   - OOMによってkillされたプロセス数
# max        - memory.maxに達した回数
# high       - memory.highに達した回数
```

### OOMグループkill（v4.19+） {#oom-group-kill-v419}

**コミット**: [3d8b38eb81ca](https://git.kernel.org/linus/3d8b38eb81ca) ("mm, oom: introduce memory.oom.group") | [LKML](https://lore.kernel.org/all/20180730180100.25079-4-guro@fb.com/)

**作者**: Roman Gushchin

単一のプロセスではなく、cgroup全体をkillする。

```bash
echo 1 > /sys/fs/cgroup/mygroup/memory.oom.group
```

1つのプロセスをkillすると他のプロセスが壊れた状態で残ってしまうコンテナで有用である。

## Pressure Stall Information（PSI） {#pressure-stall-information-psi}

PSIは、タスクがメモリを待って費やした時間を示す。

```bash
cat /sys/fs/cgroup/mygroup/memory.pressure

# avg10=0.00 avg60=0.00 avg300=0.00 total=12345
# some: 少なくとも1つのタスクがストールしていた時間の割合
# full: すべてのタスクがストールしていた時間の割合
```

### 用途 {#use-cases}

- **オートスケーリング**: 逼迫が強まったらスケールアップする
- **ヘルスチェック**: メモリが不足しているコンテナを検出する
- **ロードバランシング**: 逼迫しているノードからワークロードを移す

```bash
# システム全体の逼迫を監視
cat /proc/pressure/memory
```

## 階層的な制限 {#hierarchical-limits}

cgroupは階層的である。子の制限は親を超えられない。

```
root (limit: 8GB)
├── container-a (limit: 2GB)
│   ├── app (limit: 1GB)      <- effective: 1GB
│   └── sidecar (limit: 3GB)  <- effective: 2GB (parent limit)
└── container-b (limit: 4GB)
```

## スワップの制御 {#swap-control}

```bash
# スワップ使用量を制限 (v2)
echo 0 > /sys/fs/cgroup/mygroup/memory.swap.max    # スワップなし
echo 500M > /sys/fs/cgroup/mygroup/memory.swap.max # 500MBのスワップ

# スワップ使用量を見る
cat /sys/fs/cgroup/mygroup/memory.swap.current
```

## 歴史 {#history}

### メモリコントローラの導入（v2.6.25、2008年） {#memory-controller-introduction-v2625-2008}

**コミット**: [8cdea7c05454](https://git.kernel.org/linus/8cdea7c05454) ("Memory controller: cgroups setup")

**作者**: Balbir Singh

*注: lore.kernel.orgにある2009年以前のLKMLアーカイブはまばらである。設計はコミットメッセージに記されている。*

cgroup v1向けの最初のメモリcgroup実装。

### 統合階層（cgroup v2、v4.5、2016年） {#unified-hierarchy-cgroup-v2-v45-2016}

cgroup v2の統合階層は[v4.5で実験的扱いが外され](https://lwn.net/Articles/679786/)（[commit 67e9c74b8a87](https://git.kernel.org/linus/67e9c74b8a87)）、メモリコントローラは本番利用に耐える安定したものとみなされた。メモリコントローラはv1から大幅に作り直され、セマンティクスが整理され、pressure stall informationが統合された。

### カーネルメモリの計上（v4.5+） {#kernel-memory-accounting-v45}

v2では、カーネルメモリ（slab、スタック）が[デフォルトでmemory.currentに含まれる](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory)（[背景: 新しいslabメモリコントローラ](https://lwn.net/Articles/798605/)）。

### memory.reclaim（v5.19、2022年） {#memoryreclaim-v519-2022}

**コミット**: [94968384dde1](https://git.kernel.org/linus/94968384dde1) ("memcg: introduce per-memcg reclaim interface") | [LKML](https://lore.kernel.org/linux-mm/20220421234426.3494842-2-yosryahmed@google.com/)

**作者**: Shakeel Butt

プロアクティブな回収のインターフェース。

## 試してみる {#try-it-yourself}

### メモリを制限したシェルを作る {#create-a-memory-limited-shell}

```bash
# cgroupを作成
sudo mkdir /sys/fs/cgroup/test

# メモリコントローラを有効化 (必要なら)
echo "+memory" | sudo tee /sys/fs/cgroup/cgroup.subtree_control

# 100MBの制限を設定
echo 100M | sudo tee /sys/fs/cgroup/test/memory.max

# 現在のシェルをcgroupへ移動
echo $$ | sudo tee /sys/fs/cgroup/test/cgroup.procs

# 100MBを超える割り当てを試す
python3 -c "x = 'a' * 150_000_000"  # OOM killされる
```

### コンテナを監視する {#monitor-a-container}

```bash
# コンテナのcgroupを探す
CGROUP=$(cat /proc/<container-pid>/cgroup | cut -d: -f3)

# メモリ使用量を観察
watch -n 1 cat /sys/fs/cgroup/$CGROUP/memory.current

# 詳細な統計を見る
cat /sys/fs/cgroup/$CGROUP/memory.stat
```

### メモリ逼迫をシミュレートする {#simulate-memory-pressure}

```bash
# 低い制限でcgroupを作成
sudo mkdir /sys/fs/cgroup/pressure-test
echo 50M | sudo tee /sys/fs/cgroup/pressure-test/memory.max

# メモリを大量に使うプロセスを実行
echo $$ | sudo tee /sys/fs/cgroup/pressure-test/cgroup.procs
stress --vm 1 --vm-bytes 100M

# 逼迫を観察
cat /sys/fs/cgroup/pressure-test/memory.pressure
```

## よくある問題 {#common-issues}

### システムのメモリに空きがあるのにコンテナがOOMになる {#container-oom-despite-free-system-memory}

コンテナが自身の`memory.max`の制限に達した。

**デバッグ**: `memory.events`の`oom`のカウントを確認する。

**対策**:

- 制限を引き上げる
- アプリケーションのメモリ使用量を最適化する
- スワップと`memory.swap.max`を追加する

### コンテナの起動が遅い {#slow-container-startup}

起動中にメモリが回収されている。

**デバッグ**: `memory.pressure`を確認する

**対策**:

- `memory.high`を引き上げる
- `memory.reclaim`で事前にウォームアップする
- 制限が低すぎないか確認する

### カーネルメモリが際限なく増える（v1） {#kernel-memory-growing-unbounded-v1}

cgroup v1では、カーネルメモリはデフォルトで制限されていなかった。

**対策**: cgroup v2を使うか、v1で`memory.kmem.limit_in_bytes`を設定する。

## リファレンス {#references}

### 主要なコード {#key-code}

| ファイル | 説明 |
|------|-------------|
| [`mm/memcontrol.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c) | メモリコントローラの実装 |
| [`include/linux/memcontrol.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/memcontrol.h) | memcgのAPI |

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/admin-guide/cgroup-v2.rst`](https://docs.kernel.org/admin-guide/cgroup-v2.html) - cgroup v2の公式ドキュメント

### 関連 {#related}

- [reclaim](reclaim.md) - メモリがどのように回収されるか
- [page-allocator](page-allocator.md) - グローバルなメモリ割り当て
- [glossary](glossary.md) - cgroup、OOMの定義

## 参考文献 {#further-reading}

- [mm/memcontrol.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c) — メモリコントローラの実装全体：charge/unchargeのパス、制限の強制、PSIとの統合、`memory.reclaim`インターフェース
- `Documentation/admin-guide/cgroup-v2.rst` — すべての`memory.*`インターフェースファイルについて、セマンティクス、単位、必要なバージョンを含めて記した公式のカーネルリファレンス
- [LWN: The unified cgroup hierarchy in 4.5](https://lwn.net/Articles/679786/) — cgroup v2がなぜ再設計されたのか、そしてメモリコントローラがv1と比べて何が変わったのかを扱う
- [LWN: The new slab memory controller](https://lwn.net/Articles/798605/) — v2でカーネルメモリ（slab、スタック）がどのように統合された`memory.current`の計上に含まれるようになったかを説明する
- [LWN: Proactive reclaim for memory cgroups](https://lwn.net/Articles/894849/) — ユーザー空間主導のプロアクティブな回収のためのインターフェースである`memory.reclaim`の設計と動機
- [memcg-hierarchy](memcg-hierarchy.md) — cgroupが入れ子になったときに`memory.max`、`memory.high`、`memory.min`、`memory.low`がどう相互作用するか
- [memcg-oom](memcg-oom.md) — 回収が失敗した後にcgroupが`memory.max`を使い果たしたとき、カーネル内部で何が起きるか
- [memcg-swap](memcg-swap.md) — `memory.swap.max`と`memory.zswap.max`によるcgroupごとのスワップの計上
