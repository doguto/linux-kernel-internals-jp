# 階層的なメモリ制限 {#hierarchical-memory-limits}

> メモリの制限と保護がcgroup v2のツリーをどのように伝播するか

## フラットな制限の問題 {#the-problem-with-flat-limits}

単一のcgroupの制限は理解しやすい。`memory.max`を設定すれば、カーネルがそれを強制する。しかし実際のデプロイはフラットではない。Kubernetesのノードには、システムサービス、kubelet、そして何百ものpodがある。コンテナランタイムには、ランタイム全体のための親cgroup、コンテナごとの子cgroup、プロセスグループごとの孫cgroupがある。すべての階層で制限が設定されている場合、それらがどう相互作用するかのルールが極めて重要になる。

cgroup v2は明確なモデルを定義している。制限は下方向へ連鎖し、保護は比例配分される。このページでは、それが正確にどう機能するかを説明する。

## 4つのつまみ、再訪 {#the-four-knobs-revisited}

cgroup v2は4つのメモリ制御ファイルを公開している。2つは**制限**（消費量の上限）で、2つは**保護**（回収に対する保証）である。

| ファイル | 方向 | 種類 | 強制方法 |
|------|-----------|------|-------------|
| `memory.max` | 下方向の上限 | ハードリミット | 超過するとOOM kill |
| `memory.high` | 下方向の上限 | ソフトリミット | スロットリング + 積極的な回収 |
| `memory.low` | 上方向の保証 | ソフトな保護 | ベストエフォート。カーネルはこの値を下回る回収を避ける |
| `memory.min` | 上方向の保証 | ハードな保護 | 極度の圧迫下でも決して回収されない |

制限と保護では階層のルールが異なり、その違いを理解することがこのページの鍵である。

## 制限の連鎖: 実効制限 {#how-limits-cascade-the-effective-limit}

`memory.max`と`memory.high`について、子cgroupの**実効制限**は、自身の設定値と親の残りの余裕のうち小さいほうである。カーネルは子の設定を黙って上書きするわけではない。代わりに、各階層で独立して強制が行われる。

次の階層を考える。

```
root
└── service (memory.max = 4G)
    ├── container-a (memory.max = 3G)
    └── container-b (memory.max = 3G)
```

`container-a`と`container-b`はどちらも`memory.max = 3G`だが、親の`service`には4Gしかない。これは許可されている。cgroup v2は、親より大きな子の制限を書き込むことを妨げない。実際には次のようになる。

- `container-a`が3Gを使っていて`container-b`が2Gを使おうとすると、`container-b`自身の`memory.max`である3Gには達していなくても、親の4Gの制限によって`container-b`はスロットリングされるかOOMになる。
- どのcgroupの実効制限も、自身の制限と親ツリーが収容できる量のうち小さいほうになる。

カーネルは[`mm/memcontrol.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c)の`try_charge()`で、各階層で独立して制限を強制する。各チャージ操作は親ポインタの連鎖をたどってcgroupツリーを上に歩き、すべての祖先で`memory.max`と`memory.high`をテストする。

```
Charge path for a page allocation (simplified):
  try_charge() in mm/memcontrol.c
    → walks up cgroup tree
    → at each ancestor: compare usage against memory.max
    → if any ancestor is at limit: enter reclaim / OOM
```

## 保護の配分方法 {#how-protections-are-distributed}

保護（`memory.min`と`memory.low`）は制限とは異なる動作をする。グローバルな回収中に子の保護が考慮されるとき、カーネルは**実効保護**、つまり親が保証できる量を踏まえて子が実際に保持できる量を計算しなければならない。

この計算は[`mm/memcontrol.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c)の`mem_cgroup_calculate_protection()`で定義されている。[`Documentation/admin-guide/cgroup-v2.rst`](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory-interface-files)のカーネルドキュメントは、このモデルを次のように説明している。子の実効保護は、自身に設定された保護と親の保護のうち小さいほうであり、子の保護の合計が親の保護を超える場合は比例配分される。

### 比例配分の式 {#the-proportional-distribution-formula}

兄弟の保護の合計が親の保護を超える場合、各子は比例した取り分を受け取る。

```
effective_protection(child) =
    min(child.memory.min,
        parent_protection × (child.memory.min / sum_of_sibling_memory_mins))
```

これは公平な配分のためのヒューリスティックであり、厳密な数学的保証ではない。カーネルはこの式を回収の重みとして使うのであって、チャージ時に強制される厳格な予約として使うわけではない。

**例:**

```
parent (memory.min = 4G)
├── child-a (memory.min = 3G)
└── child-b (memory.min = 2G)
```

子の合計 = 5G、親の保護 = 4G。5G > 4Gなので、各子は比例した取り分を受け取る。

- `child-a`の実効保護 = 4G × (3G / 5G) = **2.4G**
- `child-b`の実効保護 = 4G × (2G / 5G) = **1.6G**

子の保護の合計が親の保護以下であれば、各子は設定値をそのまま受け取る。

## 3階層の例: システム → サービス → コンテナ {#a-three-level-hierarchy-system-service-container}

3階層の具体例を示す。システムには16GのRAMがあり、すべてのサービスのための`system.slice` cgroup、単一アプリケーションのための`myapp.service` cgroup、その中に`worker`と`sidecar`のコンテナがある。

```
system.slice   (memory.max = 12G, memory.min = 2G)
└── myapp.service  (memory.max = 8G,  memory.min = 3G, memory.high = 7G)
    ├── worker     (memory.max = 6G,  memory.min = 2G, memory.high = 5G)
    └── sidecar    (memory.max = 2G,  memory.min = 1G)
```

### この階層での制限の強制 {#enforcing-limits-from-this-hierarchy}

```
system.slice:   effective max = 12G
myapp.service:  effective max = min(8G, 12G)  = 8G
worker:         effective max = min(6G, 8G)   = 6G
sidecar:        effective max = min(2G, 8G)   = 2G
```

`worker`が6Gを使い、`sidecar`が3Gを割り当てようとすると、`sidecar`は親の階層ではなく自身の`memory.max`である2Gでスロットリングされる。しかし`worker`が6G、`sidecar`が2G（合計8G）を使っている場合、`myapp.service`には余裕がもうない。どちらの子からのさらなる割り当ても、`myapp.service`の階層で回収を引き起こす。

### この階層での保護の配分 {#protection-distribution-from-this-hierarchy}

```
myapp.service:  memory.min = 3G
  worker:       memory.min = 2G
  sidecar:      memory.min = 1G
  sum = 3G = parent protection, so each child gets its full value
```

合計が親とちょうど等しければ、比例によるスケーリングは不要である。次に、sidecarに`memory.min = 2G`を与えたとする。

```
  worker:  memory.min = 2G
  sidecar: memory.min = 2G
  sum = 4G > parent (3G)
  worker  effective protection = 3G × (2G/4G) = 1.5G
  sidecar effective protection = 3G × (2G/4G) = 1.5G
```

### 親に余裕があるのに子が制限を超えた場合 {#what-happens-when-a-child-exceeds-its-limit-but-the-parent-has-headroom}

親に余裕があっても、子の`memory.max`の発動は妨げられない。自身の制限に達した子は、親にまだ空き容量があってもスロットリングされるかOOM killされる。cgroupの階層は各階層で独立して制限を強制するため、子が親の未使用の余裕を「借りる」ことはできない。

```
myapp.service: usage = 3G, max = 8G    ← still 5G available
└── worker:    usage = 5.9G, max = 6G  ← worker at its own limit

A worker allocation that pushes usage to 6.1G will trigger
worker's OOM killer, even though myapp.service has 5G free.
```

子が親の余裕を使えるようにするには、子の制限を引き上げるか、制限を完全に取り除く（`memory.max = max`を設定する）。

## memory.statの階層的カウンタ {#hierarchical-counters-in-memorystat}

cgroup v2では、`memory.stat`の**すべて**のフィールドが階層的であり、サブツリー全体（そのcgroup自身とすべての子孫）を含む。一部のフィールドがcgroup単位のみだったcgroup v1とは異なる。

```bash
cat /sys/fs/cgroup/myapp.service/memory.stat
```

`memory.current`は常に階層的な合計を反映する。つまり、そのcgroupとすべての子にチャージされたメモリの合計である。`memory.stat`のすべてのフィールドについても同じことが言える。

あるcgroup自身のメモリだけ（子を除く）を見るには、親の`memory.current`から各子の`memory.current`を差し引く。

```bash
# 階層的な合計
cat /sys/fs/cgroup/myapp.service/memory.current
# 3145728000

# 子を差し引いてmyapp.service自身の使用量を求める
parent=$(cat /sys/fs/cgroup/myapp.service/memory.current)
worker=$(cat /sys/fs/cgroup/myapp.service/worker/memory.current)
sidecar=$(cat /sys/fs/cgroup/myapp.service/sidecar/memory.current)
echo $((parent - worker - sidecar))  # ローカルの使用量のみ
```

`memory.stat`ファイル自体に含まれる主要なフィールドの範囲は、cgroup v2では常に階層的なサブツリーである。[`Documentation/admin-guide/cgroup-v2.rst`](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory-interface-files)のドキュメントは、`memory.stat`の値が"account for the entire subtree"（サブツリー全体を計上する）と記している。

### memory.statの主要なフィールドとその意味 {#key-memorystat-fields-and-their-meaning}

```bash
cat /sys/fs/cgroup/myapp.service/worker/memory.stat

anon 1073741824          # 匿名ページ: ヒープ、スタック、mmap(MAP_ANONYMOUS)
file 209715200           # ファイルを裏付けとするページ: ページキャッシュ
kernel 12582912          # カーネルメモリ: slab、スタック、ページテーブル
kernel_stack 2097152     #   プロセスのカーネルスタック
pagetables 1048576       #   ページテーブルのページ
slab_reclaimable 6291456 #   回収可能なslab（dentry/inodeキャッシュ）
slab_unreclaimable 2097152 # 回収不能なslab（task_structなど）
slab 8388608             #   slabの合計（回収可能 + 回収不能）
sock 4194304             # ソケットの受信／送信バッファ
shmem 0                  # 共有メモリ（tmpfs、IPC shm）
file_mapped 104857600    # マップされているファイルページ（fileの部分集合）
file_dirty 0             # 書き込み待ちのファイルページ
file_writeback 0         # 現在書き込み中のファイルページ
inactive_anon 0          # inactive LRU上の匿名ページ
active_anon 1073741824   # active LRU上の匿名ページ
inactive_file 104857600  # inactive LRU上のファイルページ（回収候補）
active_file 104857600    # active LRU上のファイルページ
unevictable 0            # 回収できないページ（mlockされたものなど）
pgfault 458923           # ページフォールトの総数
pgmajfault 12            # メジャーフォールト（ディスクI/Oを要したもの）
pgrefill 1024            # active→inactiveのエージング中にスキャンされたページ
pgscan 2048              # 回収中にスキャンされたページ
pgsteal 1536             # 実際に回収されたページ
pgactivate 512           # inactiveからactiveへ移動したページ
workingset_refault_anon 0    # 回収後に再フォールトした匿名ページ
workingset_refault_file 64   # 回収後に再フォールトしたファイルページ
```

## memory.eventsで階層的な制限を監視する {#watching-hierarchical-limits-with-memoryevents}

各cgroupには`memory.events`ファイルがある。cgroup v2には、そのcgroupのみ（子を含まない）に帰属するイベントのための`memory.events.local`もある。

```bash
# myapp.serviceとそのすべての子に帰属するイベント
cat /sys/fs/cgroup/myapp.service/memory.events
# low 0        <- 子のメモリがmemory.lowを下回った回数
# high 45      <- memory.highを超過した回数（このcgroupまたはいずれかの子）
# max 3        <- memory.maxに達した回数
# oom 1        <- OOMイベント
# oom_kill 1   <- OOMによってkillされたプロセス
# oom_group_kill 0

# myapp.service自身のプロセスのみに帰属するイベント
cat /sys/fs/cgroup/myapp.service/memory.events.local
# （同じ形式だが、このcgroup自身のページチャージで起きたイベントのみを数える）
```

`memory.events.local`は、階層的なイベント伝播モデルとともに導入された。フィールドの全一覧については[`Documentation/admin-guide/cgroup-v2.rst`](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory-interface-files)を参照。

## 3階層の階層構造をセットアップする {#setting-up-a-three-level-hierarchy}

```bash
# 階層を作成する
sudo mkdir -p /sys/fs/cgroup/myapp.service/worker
sudo mkdir -p /sys/fs/cgroup/myapp.service/sidecar

# 各階層でメモリコントローラを有効化する
echo "+memory" | sudo tee /sys/fs/cgroup/cgroup.subtree_control
echo "+memory" | sudo tee /sys/fs/cgroup/myapp.service/cgroup.subtree_control

# 制限と保護を設定する
echo 8G  | sudo tee /sys/fs/cgroup/myapp.service/memory.max
echo 7G  | sudo tee /sys/fs/cgroup/myapp.service/memory.high
echo 3G  | sudo tee /sys/fs/cgroup/myapp.service/memory.min

echo 6G  | sudo tee /sys/fs/cgroup/myapp.service/worker/memory.max
echo 5G  | sudo tee /sys/fs/cgroup/myapp.service/worker/memory.high
echo 2G  | sudo tee /sys/fs/cgroup/myapp.service/worker/memory.min

echo 2G  | sudo tee /sys/fs/cgroup/myapp.service/sidecar/memory.max
echo 1G  | sudo tee /sys/fs/cgroup/myapp.service/sidecar/memory.min

# 設定を読み戻して実効制限を確認する
for cg in myapp.service myapp.service/worker myapp.service/sidecar; do
    echo "=== $cg ==="
    echo "max:  $(cat /sys/fs/cgroup/$cg/memory.max)"
    echo "high: $(cat /sys/fs/cgroup/$cg/memory.high)"
    echo "min:  $(cat /sys/fs/cgroup/$cg/memory.min)"
    echo "cur:  $(cat /sys/fs/cgroup/$cg/memory.current)"
done
```

## よくある間違い {#common-mistakes}

### 子の保護を親より大きく設定する {#setting-child-protections-larger-than-the-parent}

```
parent: memory.min = 1G
├── child-a: memory.min = 600M
└── child-b: memory.min = 600M
sum = 1.2G > 1G
```

これは許可されるが、各子の実効保護は500Mに縮小される。カーネルは警告を出さない。保証が本物であることを確かめるために、子の保護を親の保護と照らし合わせて監査すること。

### 親の余裕を子が使えると思い込む {#assuming-parent-headroom-is-available-to-children}

`memory.max = 1G`を与えられた子cgroupは、親cgroupに10Gの空き容量があっても1GでOOM killされる。制限はcgroupごとのものであり、「親の中で利用可能な量」ではない。

### memory.currentが階層的であることを忘れる {#forgetting-that-memorycurrent-is-hierarchical}

親cgroupの`memory.current`がその`memory.max`に等しければ、個々の子がどれも制限に達していなくても、親は制限に達している。すべての子の合計使用量が親の上限に達したのである。

## 主要なソースファイル {#key-source-files}

| ファイル | 関連する関数 |
|------|--------------------|
| [`mm/memcontrol.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c) | `mem_cgroup_calculate_protection()` — 実効保護を計算する。`mem_cgroup_margin()` — 制限までの余裕。`try_charge()` — チャージと制限強制のパス |
| [`include/linux/memcontrol.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/memcontrol.h) | `struct mem_cgroup`、インラインの`mem_cgroup_protection()` |
| [`Documentation/admin-guide/cgroup-v2.rst`](https://docs.kernel.org/admin-guide/cgroup-v2.html) | 配分モデルの権威ある説明 |

## 歴史 {#history}

### cgroup v2メモリコントローラ（v4.5、2016年） {#cgroup-v2-memory-controller-v45-2016}

階層的な保護モデルを含むcgroup v2メモリコントローラは、v4.5で本番利用可能と宣言された（[commit 67e9c74b8a87](https://git.kernel.org/linus/67e9c74b8a87)）。比例的な保護モデルはv1のより単純なアプローチを置き換えたもので、マルチテナントのワークロード向けに特に設計された。[LWN: The unified cgroup hierarchy](https://lwn.net/Articles/679786/)を参照。

### memory.lowとmemory.min（v4.18、2018年） {#memorylow-and-memorymin-v418-2018}

`memory.low`（ベストエフォートの保護）と`memory.min`（ハードな保護）は、[LWN: Putting cgroup memory protection back together](https://lwn.net/Articles/762472/)で説明されているメモリ保護モデルの一部として導入された。比例配分の計算を実装する`mem_cgroup_calculate_protection()`関数は、Tejun HeoとRoman Gushchinによって開発された。

### memory.events.local {#memoryeventslocal}

`memory.events.local`は、cgroupごとのイベントと階層的なイベントを区別するために追加された。これにより運用者は、親cgroupのOOMイベントが子によって引き起こされたのか、親自身のプロセスによるものなのかを判断できる。導入されたバージョンについては[`Documentation/admin-guide/cgroup-v2.rst`](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory-interface-files)を参照。

## 参考資料 {#references}

### 主要なコード {#key-code}

| ファイル | 説明 |
|------|-------------|
| [`mm/memcontrol.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c) | メモリコントローラ: 制限の強制、保護の計算 |
| [`include/linux/memcontrol.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/memcontrol.h) | memcgの構造体とAPI |

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/admin-guide/cgroup-v2.rst`](https://docs.kernel.org/admin-guide/cgroup-v2.html) — 保護の配分モデルを含む、権威あるcgroup v2のリファレンス

### LWNの記事 {#lwn-articles}

- [The unified cgroup hierarchy](https://lwn.net/Articles/679786/) — cgroup v2が本番利用可能になったこと
- [Putting cgroup memory protection back together](https://lwn.net/Articles/762472/) — memory.lowとmemory.minの設計

### 関連項目 {#related}

- [メモリcgroup](memcg.md) — memcgの基礎とv1とv2の比較
- [コンテナ向けのメモリチューニング](tuning-containers.md) — 実践的な制限設定ガイド
- [memory.statリファレンス](memory-stat.md) — memory.statの全フィールドの解説

## 参考文献 {#further-reading}

- [mm/memcontrol.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c) — 比例的な保護の式については`mem_cgroup_calculate_protection()`、cgroupツリーを上にたどる制限強制については`try_charge()`、余裕の計算については`mem_cgroup_margin()`
- `Documentation/admin-guide/cgroup-v2.rst` — 子と親の設定が相互作用するときに実効制限と実効保護がどう計算されるかについての正式な説明。比例配分モデルも含む
- [LWN: Putting cgroup memory protection back together](https://lwn.net/Articles/762472/) — `memory.low`（ベストエフォート）と`memory.min`（ハード）の保護の背後にある設計の根拠と議論、そして厳格な予約ではなく比例スケーリングが選ばれた理由
- [LWN: The unified cgroup hierarchy in 4.5](https://lwn.net/Articles/679786/) — cgroup v2が単一の統合階層に移行した理由と、それが多段の制限強制にとって何を意味するかの背景
- [memcg](memcg.md) — memcgの基礎: 4つの制限／保護のつまみ、基本操作、cgroup v1とv2の違い
- [memcg-oom](memcg-oom.md) — 階層の余裕が尽きた後にcgroupが`memory.max`に達したときに何が起きるか
- [reclaim](reclaim.md) — cgroupの制限に近づいたときにカーネルがどのようにページを回収するか
