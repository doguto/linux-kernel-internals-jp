# データベース向けのメモリチューニング {#tuning-memory-for-databases}

> なぜデータベースはカーネルのデフォルト設定とぶつかるのか、そしてそれをどう直すか

## なぜデータベースには特別なメモリチューニングが必要か {#why-databases-need-special-memory-tuning}

汎用のLinuxカーネルは、コードのコンパイル、Webページの配信、コンテナの実行といった多様なワークロードの混在を想定してチューニングされている。データベースは、そうしたデフォルト設定が前提とする仮定のほぼすべてを覆す。

データベースを特別なものにしている性質は3つある。

1. **巨大な共有バッファ** - 本番環境のPostgreSQLインスタンスは、RAMの25%（32GB以上）を単一の共有メモリ領域として割り当てることがある。MySQLのInnoDBバッファプールはさらに大きくなりうる。これらは長寿命で頻繁にアクセスされる割り当てであり、カーネルはこれを断片化させたりスワップアウトしたりしてはならない。

2. **ランダムI/Oパターン** - カーネルの先読み（readahead）やページキャッシュのヒューリスティクスは、シーケンシャルアクセスを前提としている。データベースは主キーによるランダムな検索、page間を飛び回るインデックススキャン、ハッシュ結合を行う。ページキャッシュは、決して読まれないデータを先読みしてメモリを浪費する。

3. **レイテンシへの敏感さ** - Webサーバーであれば、50msのストールは誰にも気付かれずに吸収できる。データベースが行ロックを保持したまま50msのコンパクションストールに陥ると、ロック待ちの連鎖、接続の滞留、そしてアプリケーションから見えるレイテンシ悪化を引き起こす。スループットよりも予測可能性の方が重要である。

```
General-purpose workload:              Database workload:
┌─────────────────────────┐            ┌─────────────────────────┐
│ Many small allocations  │            │ Few huge allocations    │
│ Sequential file access  │            │ Random page access      │
│ Throughput-oriented     │            │ Latency-sensitive       │
│ Short-lived processes   │            │ Long-lived daemon       │
│ Tolerates swap          │            │ Cannot tolerate swap    │
└─────────────────────────┘            └─────────────────────────┘
```

本ページの残りでは、これらの問題それぞれに対処するカーネルのチューニング可能なパラメータを、PostgreSQL、MySQL/MariaDB、Redisの実例とともに順に見ていく。

## 共有バッファ用のhuge page {#huge-pages-for-shared-buffers}

### 問題 {#the-problem}

32GBの共有バッファプールを4KBのpageで裏付けると、8,388,608個のページテーブルエントリが必要になる。バッファプールの検索中にTLBミスが起きるたびに、多段のページテーブルウォークが発生する。高負荷のOLTPシステムでは、共有バッファでのTLBミスが計測可能なオーバーヘッドとなる。

huge page（2MB）を使うと、ページテーブルエントリの数は1/512になる。32GBのバッファプールであれば、800万個ではなく16,384個のエントリで済む。

### hugetlbfs: 静的なhuge page {#hugetlbfs-static-huge-pages}

[THP](thp.md)とは異なり、hugetlbfsのpageはブート時に事前割り当てされ、断片化したり回収されたりすることがない。これこそがデータベースの求めるもの、すなわち共有バッファを確実かつ安定してhuge pageで裏付けることである。

#### huge pageを予約する {#reserve-huge-pages}

```bash
# 16384個のhuge pageを予約する（2MB pageで32GB）
echo 16384 > /proc/sys/vm/nr_hugepages

# あるいはブート時にカーネルコマンドラインで指定する（大きな予約ではこちらの方が確実）:
# hugepages=16384

# 割り当てを確認する:
cat /proc/meminfo | grep -i huge
# HugePages_Total:   16384
# HugePages_Free:    16384
# HugePages_Rsvd:        0
# Hugepagesize:       2048 kB
```

!!! warning "早めに予約する"
    システムがしばらく稼働した後では、物理メモリが断片化するため、huge pageの予約に失敗することがある。本番のデータベースでは、カーネルコマンドラインで、あるいはデータベースの起動前に実行されるsystemdユニットで、ブート時に予約すること。

カーネルのhugetlbサブシステムは[mm/hugetlb.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/hugetlb.c)で実装されている。`set_max_huge_pages()`関数が予約のロジックを担い、連続したメモリがすぐに得られない場合はコンパクションを試みる。

#### PostgreSQLとhugetlbfs {#postgresql-with-hugetlbfs}

PostgreSQLは共有バッファにSystem V共有メモリまたは`mmap`を使う。GUCパラメータ`huge_pages`はPostgreSQL 9.4で導入された（[リリースノート](https://www.postgresql.org/docs/9.4/release-9-4.html)）。

```bash
# 1. huge pageを予約する（shared_buffers / hugepageサイズ + 若干のオーバーヘッド）
#    shared_buffers = 32GB の場合: 32768MB / 2MB = 16384 page
#    内部構造用に約5%のオーバーヘッドを加える
echo 17000 > /proc/sys/vm/nr_hugepages

# 2. postgresユーザーがhuge pageを使えるようにする
#    方法A: sysctl（推奨）
echo "vm.hugetlb_shm_group = $(id -g postgres)" > /etc/sysctl.d/hugepages.conf
sysctl --system

#    方法B: /etc/security/limits.conf でmemlock上限を設定する
#    postgres  soft  memlock  unlimited
#    postgres  hard  memlock  unlimited

# 3. PostgreSQLを設定する
# postgresql.conf 内で:
#   shared_buffers = '32GB'
#   huge_pages = on        # 'on' = require, 'try' = best-effort, 'off' = disable
```

`huge_pages = on`の場合、PostgreSQLは`SHM_HUGETLB`付きで`shmget()`を呼び出す（新しいバージョンでは`MAP_HUGETLB`付きの`mmap`を使う）。カーネルがhugetlbプールから要求を満たせない場合、PostgreSQLは起動を拒否する。これは正しい振る舞いである。性能の劣化に後から気付くのではなく、起動時に知りたいからである。

#### Oracleとhugetlbfs {#oracle-with-hugetlbfs}

OracleのSGA（System Global Area）はhuge pageから大きな恩恵を受ける。Oracleはドキュメントでこれを「HugePages」と呼んでおり、Oracle 11g以来のベストプラクティスとなっている。

```bash
# 1. 必要なpage数を計算する: SGAターゲット / hugepageサイズ
#    sga_target = 64GB の場合: 65536MB / 2MB = 32768
echo 33000 > /proc/sys/vm/nr_hugepages

# 2. oracleユーザーのmemlockを設定する
#    /etc/security/limits.conf:
#    oracle  soft  memlock  unlimited
#    oracle  hard  memlock  unlimited

# 3. 次の条件を満たせば、OracleはSGAに自動的にhuge pageを使う:
#    - USE_LARGE_PAGES = ONLY（init.ora、推奨）
#    - 十分なhuge pageが利用可能
#    - memlock上限が設定されている
```

#### MySQL/InnoDBとhugetlbfs {#mysqlinnodb-with-hugetlbfs}

```bash
# 1. InnoDBバッファプール用にhuge pageを予約する
#    innodb_buffer_pool_size = 48GB の場合: 49152MB / 2MB = 24576
echo 25000 > /proc/sys/vm/nr_hugepages

# 2. mysqlユーザーのmemlockを設定する
#    /etc/security/limits.conf:
#    mysql  soft  memlock  unlimited
#    mysql  hard  memlock  unlimited

# 3. my.cnf で有効にする:
#    [mysqld]
#    large-pages

# 4. MySQLにはCAP_IPC_LOCKケーパビリティも必要である。あるいは十分な
#    memlock上限で実行すること。確認方法:
cat /proc/$(pidof mysqld)/status | grep HugetlbPages
```

## THP: なぜデータベースはしばしばこれを無効にするのか {#thp-why-databases-often-disable-it}

[Transparent Huge Pages](thp.md)は、hugetlbfsと同じ問題を解決するように聞こえる。しかし、問題が始まるのはまさに「transparent（透過的）」な部分である。

### コンパクションの問題 {#the-compaction-problem}

THPが有効な場合、カーネルは`khugepaged`を走らせて4KBのpageを2MBのhuge pageにまとめる（collapse）。そのためには、2MB境界に揃った連続空き領域を見つけるか作り出す必要がある。これが[コンパクション](compaction.md)、つまり物理メモリ内でpageを移動して断片化を解消する処理を引き起こす。

コンパクションはzoneロックを保持し、割り当てをストールさせることがある。毎秒数千のクエリを処理するデータベースにとって、ページフォールト中のコンパクションストールは次のことを意味する。

```
Query arrives
    │
    ▼
Buffer pool miss, need new page
    │
    ▼
Page fault → kernel tries 2MB allocation
    │
    ▼
No contiguous 2MB region → trigger compaction
    │
    ▼
Compaction holds zone lock (10-100ms+)
    │
    ▼
Query blocked, holds row locks
    │
    ▼
Other queries waiting on those locks pile up
    │
    ▼
Latency spike visible to application
```

これは多くのデータベースベンダーのガイドに記載されている。MongoDB、Redis、OracleはいずれもTHPの無効化を推奨している。PostgreSQLはTHPではなくhugetlbfsを推奨している。

コンパクションのコードは[mm/compaction.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/compaction.c)にある。ストールが起こるのは`compact_zone()`関数である。この関数はzoneロックを保持したままpageを移行し、移行の途中で中断することはできない。

### THPを無効にする {#disabling-thp}

```bash
# 即座に無効にする（実行時）
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# 確認する
cat /sys/kernel/mm/transparent_hugepage/enabled
# always madvise [never]

# カーネルコマンドラインで永続化する:
# transparent_hugepage=never
```

!!! note "THPのmadviseモード"
    THPを`never`ではなく`madvise`に設定すると、デフォルトの振る舞いを4KB pageのままにしつつ、アプリケーションが`madvise(MADV_HUGEPAGE)`でTHPをオプトインできるようになる。混在ワークロードを実行するシステムではこれが妥当な妥協点になりうるが、データベース専用サーバーの多くは単純さのために`never`を使っている。

### THPが許容されうる場合 {#when-thp-might-be-acceptable}

THPが常に悪いわけではない。カラムストアに対して大規模なシーケンシャルスキャンを行う分析系/OLAPデータベースは、アクセスパターンがhuge pageの境界と揃うため、THPの恩恵を受けることがある。問題となるのは、特にコンパクションのレイテンシが予測できないランダムアクセス主体のOLTPワークロードである。

## vm.swappiness {#vmswappiness}

### なぜデータベースはこれを低く設定するのか {#why-databases-set-this-low}

`vm.swappiness`は、カーネルがファイルバックのpage（ページキャッシュ）と比べて無名page（プロセスのメモリ）をどれだけ積極的に回収するかを制御する。値の範囲は0〜200で、デフォルトは60である。

データベースにとって、共有バッファプールは無名メモリである。カーネルには、それらの無名pageに毎秒数千回アクセスされるホットなBツリーのインデックスノードが入っていることを知る術がない。デフォルトのswappinessでは、カーネルはシーケンシャルなファイル読み出しによるページキャッシュのエントリのために場所を空けようと、バッファプールのpageをスワップアウトすると判断することがある。

swappinessの値は、[mm/vmscan.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c)にある回収スキャナのコスト計算に組み込まれる。値が大きいほどスキャナは無名pageをスキャン・回収しやすくなり、値が小さいほどファイルpageの回収に偏る。

```bash
# swappinessを1に設定する（ファイルpageの回収を強く優先する）
sysctl -w vm.swappiness=1

# 永続化する
echo "vm.swappiness = 1" >> /etc/sysctl.d/database.conf
```

**なぜ0ではなく1なのか？**

`vm.swappiness = 0`に設定すると、カーネルはシステムが深刻なメモリ圧迫状態（空きpage + ファイルpageがzoneのhighウォーターマークを下回る）にない限り、無名pageのスワップを避けるようになる。これは理想的に聞こえるが、実際にはカーネルが「スワップなし」から「OOMの圧迫」へと、穏やかな中間段階なしに一気に移行することを意味する。値を1にすると、最小限ではあるがゼロではないスワップの意思が残り、OOM状態に達する前の圧力逃がし弁をカーネルに与えることになる。

!!! info "独自のキャッシュを管理するデータベース"
    PostgreSQL、MySQL、Oracleはいずれも、LRUによる追い出し、遅延書き込み（write-behind）、先読みを備えた独自のバッファ管理を持っている。これらはホットな集合の管理において、カーネルの汎用LRUよりも優れている。`vm.swappiness`を低く保つことで、カーネルに横から口出しされることなく、データベース自身のキャッシュ管理がその役割を果たせるようになる。

## vm.dirty_ratioとvm.dirty_background_ratio {#vmdirty_ratio-and-vmdirty_background_ratio}

### ログ先行書き込みのパターン {#the-write-ahead-log-pattern}

データベースはログ先行書き込み（WAL）を使う。すべてのデータ変更はまずログファイルにシーケンシャルに書き込まれ、`fsync()`される。実際のデータファイルは後で非同期に書き込まれる（チェックポイント処理）。これにより、2つの異なるI/Oパターンが生まれる。

```
WAL writes:                         Data file writes:
┌──────────────────┐                ┌──────────────────┐
│ Sequential       │                │ Random           │
│ Small (8KB-64KB) │                │ Large (pages)    │
│ Frequent fsync() │                │ Periodic bursts  │
│ Latency-critical │                │ Throughput-ok    │
└──────────────────┘                └──────────────────┘
```

カーネルのダーティページのライトバックは、2つの比率によって制御される。

- **`vm.dirty_background_ratio`**（デフォルト: 10%）- ダーティページが利用可能メモリのこの割合を超えると、カーネルはデバイスごとのライトバックスレッド（`bdi_writeback`）によるバックグラウンドのライトバックを開始する。
- **`vm.dirty_ratio`**（デフォルト: 20%）- ダーティページがこの値を超えると、書き込みを行うプロセスはライトバックが追いつくまで**ブロック**される。これは同期的なストールである。

ライトバックのロジックは[mm/page-writeback.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page-writeback.c)にある。ダーティページがしきい値を超えたときにプロセスがスロットリングされるのは`balance_dirty_pages()`関数においてである。

### なぜデフォルト値がデータベースに害を与えるのか {#why-the-defaults-hurt-databases}

128GBのサーバーでは、20%は、プロセスがストールするまでに25GBのダーティページが溜まりうることを意味する。チェックポイントが25GBのダーティデータを一度にフラッシュすると、ディスクを飽和させてWALの書き込みをストールさせるI/Oの殺到（thundering herd）が生じる。

```bash
# データベースワークロード向けに比率を下げる
sysctl -w vm.dirty_background_ratio=3
sysctl -w vm.dirty_ratio=10

# あるいは、より予測可能な振る舞いのために絶対的なバイト値を使う:
sysctl -w vm.dirty_background_bytes=268435456   # 256MB
sysctl -w vm.dirty_bytes=1073741824              # 1GB

# 永続化する
cat >> /etc/sysctl.d/database.conf << 'EOF'
vm.dirty_background_ratio = 3
vm.dirty_ratio = 10
EOF
```

これらの値を下げると、カーネルはダーティページをより頻繁に、より小さな単位でライトバックするようになる。その結果、次の効果がある。

- チェックポイントのI/Oスパイクが減る
- WAL書き込みのレイテンシがより予測可能になる
- プロセスが`dirty_ratio`に達してストールするという崖を避けられる

!!! tip "バイト値と比率"
    大容量メモリのサーバー（256GB以上）では、3%でさえ約8GBのダーティデータになる。`vm.dirty_background_bytes`と`vm.dirty_bytes`を使うと、総RAM量に関係なく絶対値で制御できる。両方が設定されている場合、バイト値が比率の値より優先される。

## NUMAピニング {#numa-pinning}

### NUMA上でのデータベース共有バッファの問題 {#the-problem-with-database-shared-buffers-on-numa}

[NUMA](numa.md)システムでは、デフォルトのメモリ割り当てポリシーは「ローカル割り当て」、つまり要求したCPUが動いているノード上にメモリを割り当てるというものである。データベースの共有バッファプールにとって、これは次のことを意味する。

```
Node 0 (CPUs 0-15)           Node 1 (CPUs 16-31)
┌──────────────────┐          ┌──────────────────┐
│ Local memory     │          │ Local memory     │
│ 80% buffer pool  │          │ 20% buffer pool  │
│ (allocated by    │          │ (allocated by    │
│  startup thread) │          │  some queries)   │
└──────────────────┘          └──────────────────┘
```

PostgreSQLのpostmasterがNode 0で起動してそこに共有バッファを割り当てると、Node 1で動作するすべてのバックエンドは、バッファプールへのアクセスのたびにノード間のレイテンシのペナルティを払うことになる。バッファプールはデータベースで最も頻繁にアクセスされるデータ構造であるため、これはスループットにとって壊滅的である。

### numactl --interleave {#numactl-interleave}

解決策は、共有バッファの割り当てをすべてのNUMAノードにインターリーブすることである。そうすれば、平均的に各CPUのローカル/リモートアクセスが均等になる。

```bash
# メモリ割り当てをインターリーブしてPostgreSQLを起動する
numactl --interleave=all pg_ctl start -D /var/lib/postgresql/data

# MySQLの場合
numactl --interleave=all mysqld_safe &

# systemdサービスの場合は、ユニットファイルに次を追加する:
# [Service]
# ExecStart=/usr/bin/numactl --interleave=all /usr/bin/postgres -D /var/lib/postgresql/data
```

インターリーブはpageをノード間にラウンドロビンで分散させる。どのCPUも最適な局所性は得られないが、バッファプールの大部分に対して最悪のリモートアクセスとなるCPUもない。これは、すべてのCPUからアクセスされる共有データ構造にとって正しいトレードオフである。

Linux 6.3では、アクセスするCPUの方へpageを自動的に移行できるNUMAメモリバランシングの改良が導入された（[LWN: NUMA balancing gets some tweaks](https://lwn.net/Articles/920920/)を参照）。しかし、多数のCPUからアクセスされる大きな共有バッファについては、自動バランシングではすべてのアクセス元を同時に満足させることができないため、インターリーブが引き続き推奨される方法である。

### NUMAの分布を確認する {#verifying-numa-distribution}

```bash
# プロセスのノードごとのメモリ使用量を確認する
numastat $(pidof postgres) 2>/dev/null || numastat -p $(pidof postgres)

# ノード間でおおむね均等に分布しているかを見る:
#                  Node 0      Node 1
# Heap             15360       15680    <-- interleaved: good
# Stack               4           0
# Total            16200       16100

# 大きな偏り（1つのノードに90%以上）がある場合、インターリーブは機能していない
```

## vm.overcommit_memory {#vmovercommit_memory}

### なぜデータベースには厳格なオーバーコミットなのか {#why-strict-overcommit-for-databases}

[メモリオーバーコミット](overcommit.md)で説明したように、Linuxのデフォルトはヒューリスティックなオーバーコミット（モード0）である。これは一般的なワークロードには問題ないが、データベースには特有の問題がある。データベースは起動時に大きな共有バッファを割り当て、そのメモリがプロセスの生存期間中ずっと利用可能であることを期待する。

ヒューリスティックなオーバーコミットでは、カーネルは物理メモリが支えられる以上の総割り当てを許してしまうことがある。そうなると[OOM killer](oom.md)が起動して犠牲者を選ぶ。そして64GBの共有バッファプールを持つデータベースは、RSSが大きいために格好の標的となる。

```bash
# モード2: 厳格なアカウンティング
sysctl -w vm.overcommit_memory=2

# データベース + OSに十分な量を許すよう比率を設定する
# 128GBのサーバーで32GBのスワップがある場合:
# CommitLimit = (128GB * overcommit_ratio/100) + 32GB
sysctl -w vm.overcommit_ratio=80
# CommitLimit = (128 * 0.80) + 32 = 134.4GB

# 永続化する
cat >> /etc/sysctl.d/database.conf << 'EOF'
vm.overcommit_memory = 2
vm.overcommit_ratio = 80
EOF
```

アカウンティングのロジックは[mm/mmap.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mmap.c)（`__vm_enough_memory()`）にある。モード2では、すべての`mmap()`と`brk()`呼び出しがコミット量をコミット上限と照合する。新しい割り当てが上限を超える場合、システムコールは`ENOMEM`で失敗する。

**トレードオフ:** モード2では、大きなプロセスからの`fork()`が失敗しうる。copy-on-writeであっても、子プロセスのアドレス空間がコミット上限に対して計上されるからである。PostgreSQLは接続ごとにバックエンドをforkするため、`overcommit_ratio`は、forkされたすべてのバックエンドの見かけ上の（実際のではない）メモリコストを賄えるだけ高くしなければならない。`/proc/meminfo`の`Committed_AS`を監視して、余裕があることを確認すること。

!!! warning "コミットアカウンティングとfork()"
    `shared_buffers`が32GBで接続数が200のPostgreSQLサーバーでは、実際のRAM使用量よりはるかに大きな`Committed_AS`が表示されることがある。forkされた各バックエンドが、コミットアカウンティング上で親のアドレス空間を継承するためである。`overcommit_ratio`は控えめに設定し、`Committed_AS`と`CommitLimit`を比較して監視すること。

## vm.zone_reclaim_mode {#vmzone_reclaim_mode}

### なぜデータベースはzone reclaimを無効にすべきなのか {#why-databases-should-disable-zone-reclaim}

NUMAシステムでは、`vm.zone_reclaim_mode`は、リモートノードにフォールバックする前に、カーネルがローカルノードからメモリを回収しようとするかどうかを制御する。有効な場合、カーネルはリモートノードから割り当てる代わりに、ローカルzoneのページキャッシュを追い出したり、pageをスワップアウトしたりすることさえある。

これはデータベースにとってまさに間違った振る舞いである。データベースのバッファプールはページキャッシュではなく、再構築にコストのかかる無名メモリである。ページキャッシュについても、ローカル割り当てを満たすためにホットなWALセグメントのキャッシュを追い出すのは、ノード間のレイテンシを払うより悪い。

```bash
# zone reclaimを無効にする（最近のほとんどのディストリビューションではデフォルトだが、確認すること）
sysctl -w vm.zone_reclaim_mode=0

# 永続化する
echo "vm.zone_reclaim_mode = 0" >> /etc/sysctl.d/database.conf
```

zone reclaimのロジックは[mm/vmscan.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c)（`node_reclaim()`）にある。モードが0の場合、アロケータはローカルでの回収をスキップして直ちにリモートノードにフォールバックする。モードが0以外の場合は、まずローカルでの回収を試みるため、NUMA上のデータベースワークロードではレイテンシスパイクを引き起こすことがある。

カーネルはコミット[4f9b16a6 ("mm: disable zone_reclaim_mode by default")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4f9b16a64753d0bb607454347036dc997fd03b82)でデフォルト値を1から0に変更した。古い振る舞いはほとんどのワークロードにとって益より害の方が大きいと認められたためである。システムが妥当なデフォルト値になっていることを確認すること。

```bash
cat /proc/sys/vm/zone_reclaim_mode
# 0 であるべき
```

## 監視 {#monitoring}

### /proc/meminfoの主要フィールド {#key-procmeminfo-fields}

```bash
# データベースワークロードにとって必須のフィールド
cat /proc/meminfo | grep -E \
  'MemTotal|MemFree|MemAvailable|Buffers|Cached|SwapTotal|SwapFree|Dirty|Writeback|AnonPages|Mapped|Shmem|HugePages|Committed_AS|CommitLimit'
```

| フィールド | 何が分かるか | 警告サイン |
|-------|-------------------|--------------|
| `MemAvailable` | スワップせずに割り当て可能なメモリ量 | データベースのバッファプールサイズを下回る |
| `SwapFree` | 残りのスワップ領域 | チューニング済みのデータベースサーバーでは、スワップの使用は少しでも危険信号 |
| `Dirty` | 変更されたがまだディスクに書き込まれていないpage | 高い値が続くのは、ライトバックが追いついていないことを意味する |
| `Writeback` | 現在ディスクに書き込み中のpage | 非ゼロが続くのはI/Oのボトルネックを意味する |
| `AnonPages` | ファイルバックでないpage（バッファプールを含む） | データベースのウォームアップ後は安定しているべき |
| `Shmem` | 共有メモリ（PostgreSQLの共有バッファはこれを使う） | shared_buffersとおおむね一致するべき |
| `HugePages_Free` | 未使用のhuge page | データベースの起動後に減り、その後は安定しているべき |
| `Committed_AS` | コミット済みのアドレス空間の合計 | `CommitLimit`に近づくと、新しい割り当てが失敗する（モード2） |

### /proc/vmstatの主要フィールド {#key-procvmstat-fields}

```bash
# 回収とコンパクションのリアルタイム監視
watch -n 1 'cat /proc/vmstat | grep -E \
  "pgscan_direct|pgsteal_direct|pgscan_kswapd|pgsteal_kswapd|pswpin|pswpout|compact_stall|compact_fail|thp_fault_alloc|thp_fault_fallback|numa_hit|numa_miss|numa_foreign"'
```

| カウンタ | 何が分かるか | 警告サイン |
|---------|-------------------|--------------|
| `pgscan_direct` | 直接回収によるスキャン（プロセスがストールしている） | 通常負荷下での増加はすべて |
| `pgsteal_direct` | 直接回収によって回収されたpage | プロセスがメモリを待たなければならなかった |
| `pswpin` / `pswpout` | スワップイン/アウトされたpage | チューニング済みサーバーでの活動はすべて |
| `compact_stall` | コンパクションを待ってストールしたプロセス | 増加 = THPまたはhugepageの割り当ての問題 |
| `compact_fail` | 失敗したコンパクションの試行 | compact_stallに対する比率が高い = 断片化 |
| `thp_fault_fallback` | 4KBにフォールバックしたTHPの割り当て | THPが有効な場合、断片化を示す |
| `numa_hit` / `numa_miss` | ローカル対リモートのNUMA割り当て | `numa_miss`が高い = メモリポリシーの設定ミス |

### 実用的な監視スクリプト {#a-practical-monitoring-script}

```bash
#!/bin/bash
# db-memcheck.sh: データベースのメモリ設定の簡易ヘルスチェック

echo "=== Huge Pages ==="
grep -i huge /proc/meminfo

echo ""
echo "=== Swap Activity ==="
grep -E "SwapTotal|SwapFree" /proc/meminfo
echo "Swap I/O: $(grep -E 'pswpin|pswpout' /proc/vmstat)"

echo ""
echo "=== Dirty Pages ==="
grep -E "^Dirty|^Writeback" /proc/meminfo

echo ""
echo "=== Reclaim Pressure ==="
echo "Direct reclaim scans: $(awk '/pgscan_direct/ {sum+=$2} END {print sum}' /proc/vmstat)"
echo "Compaction stalls:    $(grep compact_stall /proc/vmstat | awk '{print $2}')"

echo ""
echo "=== NUMA Balance ==="
grep -E "numa_hit|numa_miss" /proc/vmstat

echo ""
echo "=== Commit Status ==="
grep -E "Committed_AS|CommitLimit" /proc/meminfo

echo ""
echo "=== THP Status ==="
echo "Enabled: $(cat /sys/kernel/mm/transparent_hugepage/enabled)"
echo "Defrag:  $(cat /sys/kernel/mm/transparent_hugepage/defrag)"
```

## 実践例 {#practical-examples}

### PostgreSQL {#postgresql}

PostgreSQL専用の128GBサーバー向けの、完全なsysctlと設定の例:

```bash
# /etc/sysctl.d/postgresql.conf

# huge page: 32GBのshared_buffers + オーバーヘッド分を予約
vm.nr_hugepages = 17000

# スワップ: 無名pageをRAMに保持することを強く優先
vm.swappiness = 1

# ダーティページ: 大きなバーストではなく頻繁で小さなフラッシュ
vm.dirty_background_ratio = 3
vm.dirty_ratio = 10

# NUMA: zone reclaimを無効化
vm.zone_reclaim_mode = 0

# オーバーコミット: 厳格なアカウンティング
vm.overcommit_memory = 2
vm.overcommit_ratio = 80
```

```ini
# postgresql.conf
shared_buffers = '32GB'
huge_pages = on
effective_cache_size = '80GB'    # プランナにOSキャッシュの量を伝えるもので、割り当てではない
wal_buffers = '64MB'
```

```bash
# systemdのオーバーライド: /etc/systemd/system/postgresql.service.d/numa.conf
[Service]
ExecStart=
ExecStart=/usr/bin/numactl --interleave=all /usr/bin/postgres -D /var/lib/postgresql/data
LimitMEMLOCK=infinity
```

```bash
# THPを無効にする
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# あるいはカーネルコマンドラインで（永続的）:
# transparent_hugepage=never
```

### MySQL / MariaDB {#mysql-mariadb}

MySQL専用の64GBサーバー向けの完全な設定例:

```bash
# /etc/sysctl.d/mysql.conf

# 40GBのInnoDBバッファプール用のhuge page
vm.nr_hugepages = 21000

vm.swappiness = 1
vm.dirty_background_ratio = 3
vm.dirty_ratio = 10
vm.zone_reclaim_mode = 0
vm.overcommit_memory = 2
vm.overcommit_ratio = 85
```

```ini
# /etc/my.cnf.d/server.cnf
[mysqld]
innodb_buffer_pool_size = 40G
innodb_buffer_pool_instances = 8    # mutex競合を減らす
large-pages                         # huge pageを有効にする

# InnoDBは独自にフラッシュを管理する。以下はsysctlの設定を補完する:
innodb_flush_method = O_DIRECT      # データファイルについてページキャッシュをバイパスする
innodb_io_capacity = 2000           # ストレージのIOPSに合わせる
innodb_io_capacity_max = 4000
```

InnoDBで`O_DIRECT`を使うことは重要である。InnoDBは独自のバッファプールを持っているので、`O_DIRECT`はデータファイルのI/Oについてページキャッシュをバイパスするようカーネルに指示する。これにより二重キャッシュを避け、メモリ圧迫を減らせる。`O_DIRECT`とページキャッシュの相互作用は[mm/filemap.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/filemap.c)で実装されている。

```bash
# NUMAインターリーブで起動する
numactl --interleave=all mysqld_safe &

# THPを無効にする
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

### Redis {#redis}

Redisは（シャードごとに）シングルスレッドで、データセット全体を無名メモリに保持する。チューニングの要件はディスクベースのデータベースとは異なるが、メモリ管理上の懸念は重なる部分がある。

```bash
# /etc/sysctl.d/redis.conf

# Redisは主要なデータ構造にhuge pageを使わない
# （小さな割り当てを行うjemallocを使う）が、大きな
# Redisインスタンスを使う場合はhuge pageが役立つこともある。
# vm.nr_hugepages = ...  # Redisでは通常不要

# 重要: RedisはBGSAVE/BGREWRITEAOFのためにforkする
# オーバーコミットの設定はfork()の成功を許さなければならない
vm.overcommit_memory = 1    # Redisはモード2ではなくモード1を推奨している!

vm.swappiness = 1
vm.zone_reclaim_mode = 0

# ダーティ比率を下げるとBGSAVEが速く完了しやすくなる
vm.dirty_background_ratio = 3
vm.dirty_ratio = 10
```

!!! warning "Redisとオーバーコミットのモード2"
    PostgreSQLやMySQLとは異なり、Redisは`vm.overcommit_memory = 1`（常にオーバーコミット）を推奨している。これはRedisがバックグラウンドの永続化（BGSAVE、BGREWRITEAOF）に`fork()`を使うためである。50GBのRAMを使うRedisインスタンスはforkする必要があり、50GBのアドレス空間を持つ子プロセスが作られる。モード2では、`CommitLimit`に50GBの余裕がない限りこのforkは失敗し、厳格なアカウンティングの意味がなくなってしまう。モード1ではforkが成功し、copy-on-writeによって実際に100GBを必要とすることを避ける。

```bash
# THPを無効にする: Redisはこれを強く推奨している
# THPが有効な場合、Redisは起動時に警告をログに出す
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

## 試してみる {#try-it-yourself}

### 現在の設定を確認する {#check-your-current-settings}

```bash
# 関連するsysctlの値をまとめて表示する
echo "=== Current Database Memory Tunables ==="
sysctl vm.swappiness vm.dirty_ratio vm.dirty_background_ratio \
       vm.overcommit_memory vm.overcommit_ratio vm.zone_reclaim_mode \
       vm.nr_hugepages
echo ""
echo "THP enabled: $(cat /sys/kernel/mm/transparent_hugepage/enabled)"
```

### メモリ圧迫下のスワップの振る舞いを観察する {#observe-swap-behavior-under-memory-pressure}

```bash
# 端末1: スワップの活動を監視する
watch -n 1 'grep -E "SwapTotal|SwapFree|SwapCached" /proc/meminfo; echo "---"; grep -E "pswpin|pswpout" /proc/vmstat'

# 端末2: メモリ圧迫を作り出す（サイズはシステムに合わせて調整する）
stress-ng --vm 2 --vm-bytes 80% --timeout 30s

# vm.swappiness=60 と vm.swappiness=1 で振る舞いを比較する
```

### NUMAの影響を測定する {#measure-numa-effects}

```bash
# 特定のノード上とインターリーブとでメモリを割り当て、
# numactlでアクセスレイテンシを測定する:
numactl --membind=0 -- stress-ng --vm 1 --vm-bytes 1G --timeout 10s --metrics
numactl --interleave=all -- stress-ng --vm 1 --vm-bytes 1G --timeout 10s --metrics

# 実行中のプロセスのメモリがどこにあるかを確認する:
numastat -p $(pidof postgres)
```

### ダーティページのライトバックを観察する {#watch-dirty-page-writeback}

```bash
# 端末1: ダーティページ数を監視する
watch -n 0.5 'grep -E "^Dirty|^Writeback" /proc/meminfo'

# 端末2: 書き込みを発生させる
dd if=/dev/zero of=/tmp/testfile bs=1M count=2048 conv=fdatasync

# ダーティページがdirty_background_ratioまで溜まってから
# ライトバックが始まる様子と、dirty_ratioで頭打ちになる様子を観察する
```

## 参考文献 {#further-reading}

- [docs.kernel.org: vm.overcommit_memory](https://docs.kernel.org/admin-guide/sysctl/vm.html#overcommit-memory) - vm sysctlのすべてのチューニング可能なパラメータに関するカーネルドキュメント
- [docs.kernel.org: hugetlbpage](https://docs.kernel.org/admin-guide/mm/hugetlbpage.html) - huge pageの設定に関するカーネルドキュメント
- [LWN: NUMA in a hurry](https://lwn.net/Articles/473440/) - LinuxにおけるNUMAメモリ管理の概要
- [LWN: Transparent huge pages](https://lwn.net/Articles/423584/) - THPの当初の設計とトレードオフ
- [LWN: Better active/inactive list balancing](https://lwn.net/Articles/495543/) - カーネルが何を回収するかをどう決めるか
- [mm/vmscan.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) - 回収、swappiness、zone reclaimのロジック
- [mm/page-writeback.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page-writeback.c) - ダーティページのライトバックとスロットリング
- [mm/hugetlb.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/hugetlb.c) - huge pageの予約と割り当て
