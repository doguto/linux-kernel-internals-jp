# /proc/meminfoを理解する {#understanding-procmeminfo}

> Linuxのメモリ統計を読み解くための実践ガイド

*これはシナリオ別の診断ガイドである。フィールドごとのリファレンスは[/proc/meminfoリファレンス](proc-meminfo.md)を参照。*

## なぜ/proc/meminfoか {#why-procmeminfo}

サーバーにSSHでログインする。何かがおかしい — アプリケーションが遅い、監視ダッシュボードが赤くなっている、あるいはOOM killが起きたばかりだ。最初に思いつくのは`free -h`だが、これは要約しか表示しない。本当のデータは`/proc/meminfo`にある。

```bash
$ cat /proc/meminfo
MemTotal:       16384000 kB
MemFree:          204560 kB
MemAvailable:    8741200 kB
Buffers:          182400 kB
Cached:          7892000 kB
SwapCached:        12400 kB
Active:          6420000 kB
Inactive:        5840000 kB
Active(anon):    3210000 kB
Inactive(anon):  1120000 kB
Active(file):    3210000 kB
Inactive(file):  4720000 kB
...
```

このファイルは[`fs/proc/meminfo.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/proc/meminfo.c)によって生成される。すべてのフィールドはカーネルのデータ構造から取得されており、システムのメモリ状態に関する唯一の正しい情報源である。

## シナリオ1:「アプリケーションが遅い — メモリが原因か?」 {#scenario-1-my-application-is-slow-is-it-memory}

これは最もよくある疑問である。メモリのせいにする前に、2つのことを確認する。

**ステップ1: システムはスワップしているか?**

```bash
$ grep -E 'SwapTotal|SwapFree' /proc/meminfo
SwapTotal:       8388608 kB
SwapFree:        6000000 kB
```

`SwapTotal - SwapFree`が大きく、かつ増え続けているなら、システムはスワップを活発に使っている。これはページ回収が匿名pageをディスクへ押し出していることを意味し、レイテンシの大きな原因となる。

`vmstat 1`で確認する。

```bash
$ vmstat 1
procs -----------memory---------- ---swap--
 r  b   swpd   free   buff  cache   si   so
 3  1 238400 204560 182400 7892000   40  120
 2  2 239200 198000 182400 7890000   80  200
```

`si`（スワップイン）と`so`（スワップアウト）の列がゼロでなければ、pageがRAMとディスクの間で活発に移動している。これはメモリに起因する遅さの決定的な証拠である。

**ステップ2: MemAvailableは危険なほど低いか?**

```bash
$ grep MemAvailable /proc/meminfo
MemAvailable:    8741200 kB
```

`MemAvailable`は、スワップせずに新しいアプリケーションを起動するために使えるメモリ量の推定値である。これがゼロに近づくと、システムはメモリ逼迫状態にある。（カーネル自身はパーセンテージのしきい値ではなくゾーンのウォーターマークを使う — [ページ回収](reclaim.md)を参照。）

**ステップ3: 直接回収を確認する。**

```bash
$ grep -E 'allocstall|pgscan_direct' /proc/vmstat
allocstall_dma 0
allocstall_normal 142
allocstall_movable 38
pgscan_direct 284500
```

`allocstall`の値がゼロでなければ、プロセスが処理を止めて自らメモリを回収（直接回収）しなければならなかったことを意味し、これはすべての割り当てにレイテンシを加える。詳細は[ページ回収](reclaim.md)を参照。

**判定表:**

| 症状 | 意味 |
|---------|---------|
| MemAvailableが高く、スワップ活動なし | メモリは問題ない — 他を調べる |
| MemAvailableが低く、スワップ活動あり | メモリ逼迫 — 遅さの原因である可能性が高い |
| MemAvailableが低く、スワップなし | 回収は機能しているが余裕がない — RAMを増やすかワークロードを減らす |
| 直接回収（`allocstall`）が増加中 | 割り当てがストールしている — 深刻な逼迫 |

## シナリオ2:「メモリ使用率が95% — パニックすべきか?」 {#scenario-2-were-at-95-memory-usage-should-i-panic}

これはLinuxのメモリ管理で最もよくある誤解である。監視ツールは一般に、メモリ使用率を次のように計算する。

```
used% = (MemTotal - MemFree) / MemTotal * 100
```

健全で忙しいサーバーでは、`MemFree`はごくわずかになる。カーネルは「空き」メモリをページキャッシュに使う — 読み出しを高速化するためにファイルデータをRAMにキャッシュするのである（[ページキャッシュ](page-cache.md)を参照）。これは良いことだ。

**重要な数値は`MemFree`ではなく`MemAvailable`である。**

### MemFreeとMemAvailable {#memfree-vs-memavailable}

```
MemTotal:       16384000 kB
MemFree:          204560 kB    <-- 1.2% free — looks alarming
MemAvailable:    8741200 kB    <-- 53% available — actually fine
```

`MemFree`は完全に未使用のRAMであり、キャッシュにすら使われていない。よく活用されているシステムでは、これは小さいはずである。

`MemAvailable`は、スワップせずに割り当て可能なメモリ量の推定値である。次のものを考慮に入れている。

- `MemFree` — 当然利用可能
- 回収可能なページキャッシュ — 即座に破棄できるファイルバックのpage
- 回収可能なslab — 縮小できるカーネルキャッシュ（dentryキャッシュやinodeキャッシュなど）

`MemAvailable`の計算は、Rik van Rielによるコミット[`34e431b0ae39`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=34e431b0ae398fc54ea69ff85ec700722c9da773)（"proc: provide MemAvailable in /proc/meminfo"、Linux 3.14、2014年）で追加された。それ以前は、システム管理者が何が回収可能かを推定しなければならなかった。コミットメッセージは問題をこう説明している。

> Many load balancing and workload placing programs check /proc/meminfo to estimate how much free memory is available. They generally do this by adding up "free" and "cached", which was fine ten years ago, but is pretty much guaranteed to be wrong today.

（訳: 多くの負荷分散やワークロード配置のプログラムは、/proc/meminfoを確認して空きメモリがどれだけあるかを推定する。一般には「free」と「cached」を足し合わせるが、これは10年前なら問題なかったものの、今日ではほぼ確実に間違っている。）

カーネルは`MemAvailable`を[`si_mem_available()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/show_mem.c)で計算する（歴史的には`mm/page_alloc.c`にあったが、後のカーネルで移動した）。簡略化したロジックは次のとおり。

```
available = MemFree - low watermark reserve
available += reclaimable page cache (with a safety margin)
available += reclaimable slab (with a safety margin)
```

「安全マージン」は、逼迫時にすべてのキャッシュが回収可能なわけではないことを考慮したものである — カーネルはlowウォーターマーク分のpageを予約として確保している。

### 健全と不健全の比較 {#healthy-vs-unhealthy-a-side-by-side}

**健全なWebサーバー（16 GB RAM）:**

```
MemTotal:       16384000 kB
MemFree:          204560 kB    # Low — that's fine
MemAvailable:    8741200 kB    # 53% — plenty of room
Buffers:          182400 kB
Cached:          7892000 kB    # 7.5 GB of page cache — good
SwapTotal:       8388608 kB
SwapFree:        8388608 kB    # No swap used
Active(anon):    3210000 kB    # ~3 GB application memory
Active(file):    3210000 kB    # Actively used cache
Inactive(file):  4720000 kB    # Cache that can be reclaimed
```

このシステムは素朴な指標では「95%使用」に見えるが、8.7 GBが利用可能である。`Cached`の値が大きいということはファイルI/Oが速いことを意味する。`SwapFree`は`SwapTotal`と等しく、スワップは発生していない。

**不健全なアプリケーションサーバー（16 GB RAM）:**

```
MemTotal:       16384000 kB
MemFree:           52000 kB    # Very low
MemAvailable:     310000 kB    # 1.9% — critical
Buffers:           14000 kB
Cached:           420000 kB    # Almost no cache — reclaimed away
SwapTotal:       8388608 kB
SwapFree:        2100000 kB    # 6 GB of swap used
Active(anon):   12800000 kB    # ~12.2 GB — apps consuming almost everything
Active(file):     180000 kB    # Almost no file cache
Inactive(file):   160000 kB    # Nothing left to reclaim
```

このシステムは本当に問題を抱えている。アプリケーションメモリ（`Active(anon)`）がほぼすべてを消費している。ページキャッシュはほとんどゼロまで圧迫されている（ファイルI/Oは遅くなる）。6 GBがスワップにある。`MemAvailable`は2%未満である。

## シナリオ3:「メモリはどこに消えているのか?」 {#scenario-3-where-is-all-my-memory-going}

`/proc/meminfo`をカテゴリに分解する。

```
               MemTotal (16 GB)
                    │
        ┌───────────┼───────────┐
        │           │           │
   Application   Kernel     Page Cache
    Memory       Memory     + Buffers
        │           │           │
  ┌─────┴─────┐    │     ┌─────┴─────┐
  │           │    │     │           │
 Anon       Anon  Slab  File       File
Active    Inactive      Active    Inactive
```

### アプリケーションメモリ（匿名page） {#application-memory-anonymous-pages}

```bash
$ grep -E 'Active\(anon\)|Inactive\(anon\)|AnonPages' /proc/meminfo
Active(anon):    3210000 kB
Inactive(anon):  1120000 kB
AnonPages:       4280000 kB
```

これはアプリケーションが割り当てたメモリ — ヒープ、スタック、匿名`mmap()`領域 — である。ファイルによるバックがないため、アプリケーションが解放するか、スワップアウトされることでしか解放できない。

`AnonPages`は実際に使用中の匿名pageを数える。`Active(anon) + Inactive(anon)`はLRUリスト上の匿名pageを含み、共有メモリやtmpfsのpageも数える。

### ページキャッシュ（ファイルバックのpage） {#page-cache-file-backed-pages}

```bash
$ grep -E 'Active\(file\)|Inactive\(file\)|Cached|Buffers' /proc/meminfo
Buffers:          182400 kB
Cached:          7892000 kB
Active(file):    3210000 kB
Inactive(file):  4720000 kB
```

`Cached`はページキャッシュの総サイズ（`SwapCached`を除く）である。`Buffers`はブロックデバイスinodeのページキャッシュ — 生のブロックデバイスI/O（例: `/dev/sda`からの直接読み出し）— である。これは統合ページキャッシュの小さな部分集合であり、独立した構造ではない。

`Active(file)`のpageは最近アクセスされたもので、回収されにくい。`Inactive(file)`のpageはより古く、メモリが必要になったときに先に回収される。大きな`Inactive(file)`は良い予備であり、すばやく解放できるメモリがあることを意味する。

### カーネルメモリ {#kernel-memory}

```bash
$ grep -E 'Slab|SReclaimable|SUnreclaim|KernelStack|PageTables|VmallocUsed' /proc/meminfo
Slab:             620000 kB
SReclaimable:     480000 kB
SUnreclaim:       140000 kB
KernelStack:       24000 kB
PageTables:        86000 kB
VmallocUsed:       52000 kB
```

`Slab`はカーネルオブジェクトキャッシュが使うメモリである（[kmalloc（SLUB）](slab.md)を参照）。`SReclaimable`（dentryキャッシュ、inodeキャッシュ）は逼迫時に解放できる。`SUnreclaim`（タスク構造体、ネットワークバッファ）はできない。

`KernelStack`は全スレッドのカーネルスタックである。`PageTables`はページテーブルのメモリであり、プロセスが多い場合や大きなアドレス空間を持つ場合に大きくなりうる（[ページテーブル](page-tables.md)を参照）。`VmallocUsed`は`vmalloc()`によるメモリを追跡する（[vmalloc](vmalloc.md)を参照）。

### 共有メモリとtmpfs {#shared-memory-and-tmpfs}

```bash
$ grep -E 'Shmem|Mapped' /proc/meminfo
Shmem:            890000 kB
Mapped:           540000 kB
```

`Shmem`には共有メモリセグメント（`shmget`、`shm_open`）、tmpfs、`mmap(MAP_SHARED|MAP_ANONYMOUS)`が含まれる。これはファイルバック（tmpfs）であるため`Cached`に計上される。よくある驚きは、tmpfsのファイルがRAMを消費し、それが`Cached`に現れることである。

`Mapped`は`mmap()`でプロセスのアドレス空間にマップされたメモリの総量である。共有ライブラリ、メモリマップドファイル、共有メモリが含まれる。

## シナリオ4:「ページキャッシュは効いているか?」 {#scenario-4-is-my-page-cache-effective}

効果的なページキャッシュはディスクI/Oを減らす。ファイルバックのLRUリストを調べる。

```bash
$ grep -E 'Active\(file\)|Inactive\(file\)' /proc/meminfo
Active(file):    3210000 kB
Inactive(file):  4720000 kB
```

**見るべきポイント:**

| パターン | 意味 |
|---------|---------|
| 大きな`Active(file)` + 大きな`Inactive(file)` | 健全なキャッシュ、キャッシュされたファイルデータが豊富 |
| 大きな`Active(file)` + 小さな`Inactive(file)` | ワーキングセットがRAMに収まっており、回収対象は少ない |
| 小さな`Active(file)` + 小さな`Inactive(file)` | キャッシュが圧迫されている — アプリメモリが支配的 |
| 小さな`Active(file)` + 大きな`Inactive(file)` | キャッシュデータは多いが再利用が少ない（ストリーミング読み出し） |

**キャッシュの効果を時系列で監視する:**

```bash
# /proc/vmstat でページキャッシュのヒット率を観察する
$ grep -E 'pgpgin|pgpgout|pswpin|pswpout' /proc/vmstat
pgpgin 12483920      # KiB read from block devices (not pages)
pgpgout 8492100      # KiB written to block devices (not pages)
pswpin 0             # Pages swapped in (counted in pages, not KiB)
pswpout 0            # Pages swapped out (counted in pages, not KiB)
```

アプリケーションの読み出し量に比べて`pgpgin`が低ければ、キャッシュは役目を果たしている。`pswpin`/`pswpout`がゼロでなく増え続けているなら、システムはスワップしている — キャッシュの問題ではなく、RAMが足りない。

**キャッシュの逼迫と比較する:**

```bash
# リアルタイムで観察する（値は累積カウンタ）
$ watch -d -n 1 'grep -E "pgpgin|pgpgout|pswpin|pswpout|pgfault" /proc/vmstat'
```

`pgfault`の数が多く`pgpgin`が少ない場合、ほとんどのフォールトはキャッシュから満たされている（マイナーフォールト）。`pgpgin`が多い場合は、データがディスクから読まれている（メジャーフォールトまたはキャッシュミス）。

## シナリオ5:「メモリリークしているか?」 {#scenario-5-am-i-leaking-memory}

メモリリークは、ワークロードが増えていないのに匿名メモリが時間とともに着実に増加するという形で現れる。

**観察すべきもの:**

```bash
# 60秒ごとにサンプリングする
$ while true; do
    echo "$(date): $(grep -E 'MemAvailable|AnonPages|Shmem|Slab' /proc/meminfo | tr '\n' ' ')"
    sleep 60
  done
```

**リークの兆候:**

1. **`AnonPages`が着実に増加** — アプリケーションが割り当てたまま解放していない
2. **`MemAvailable`が着実に減少** — 利用可能なメモリが消費されている
3. **`Slab`が着実に増加** — カーネルのメモリリーク（まれだが、より深刻）
4. **`Shmem`が着実に増加** — 共有メモリやtmpfsが片付けられずに増えている

**健全なパターン（安定したワークロード）:**

```
14:00  AnonPages: 4280000 kB  MemAvailable: 8741200 kB  Slab: 620000 kB
14:10  AnonPages: 4310000 kB  MemAvailable: 8720000 kB  Slab: 621000 kB
14:20  AnonPages: 4275000 kB  MemAvailable: 8750000 kB  Slab: 619000 kB
14:30  AnonPages: 4290000 kB  MemAvailable: 8735000 kB  Slab: 620000 kB
```

値は一定の範囲内で変動するが、上昇傾向にはない。

**リークしているパターン:**

```
14:00  AnonPages: 4280000 kB  MemAvailable: 8741200 kB  Slab: 620000 kB
14:10  AnonPages: 4520000 kB  MemAvailable: 8500000 kB  Slab: 620000 kB
14:20  AnonPages: 4780000 kB  MemAvailable: 8240000 kB  Slab: 620000 kB
14:30  AnonPages: 5050000 kB  MemAvailable: 7970000 kB  Slab: 620000 kB
```

`AnonPages`が10分ごとに約250 MB増え、`MemAvailable`が同じ割合で減っている — 典型的なユーザー空間のリークである。`Slab`は安定しているので、カーネルは犯人ではない。

**リークしているプロセスを見つける:**

```bash
# RSS でプロセスをソートする
$ ps aux --sort=-%mem | head -10

# あるいはプロセスごとの匿名メモリを確認する
$ for pid in $(ls /proc | grep '^[0-9]'); do
    if [ -f /proc/$pid/status ]; then
        name=$(grep Name /proc/$pid/status | awk '{print $2}')
        anon=$(grep RssAnon /proc/$pid/status 2>/dev/null | awk '{print $2}')
        [ -n "$anon" ] && [ "$anon" -gt 100000 ] && echo "$anon kB  $name (pid $pid)"
    fi
  done 2>/dev/null | sort -rn | head -10
```

## フィールド別リファレンス {#field-by-field-reference}

すべての値の単位はkBである。出典: [`fs/proc/meminfo.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/proc/meminfo.c)。

### 中核フィールド {#core-fields}

| フィールド | 説明 | 注視すべき状況 |
|-------|-------------|---------------|
| `MemTotal` | 使用可能なRAMの総量（カーネル予約分を除く） | 基準値 — 変化しない |
| `MemFree` | 完全に未使用のRAM | 低い値は正常 — カーネルは積極的にキャッシュする |
| `MemAvailable` | スワップせずに利用可能なメモリの推定値 | キャパシティプランニングの**主要指標** |
| `Buffers` | ブロックデバイスinodeのページキャッシュ（生のブロックI/O） | 通常は小さい。回収可能分に計上される |
| `Cached` | ページキャッシュ（RAM上のファイルデータ）+ tmpfs | `Shmem`を含む。回収可能な部分は利用可能 |
| `SwapCached` | RAMにも存在するスワップpage（再読み込みを回避） | ゼロでなければ最近スワップが発生した |

### 匿名メモリ {#anonymous-memory}

| フィールド | 説明 | 注視すべき状況 |
|-------|-------------|---------------|
| `Active(anon)` | 最近アクセスされた匿名page | 増加 = アプリのワーキングセットが拡大中 |
| `Inactive(anon)` | 最近アクセスされていない匿名page | スワップアウトの候補 |
| `AnonPages` | ページテーブルにマップされた匿名pageの総量 | アプリケーションのメモリ使用量の主要指標 |
| `AnonHugePages` | 匿名のTransparent Huge Page | [Transparent Huge Pages](thp.md)を参照 |

### ファイルバックのメモリ {#file-backed-memory}

| フィールド | 説明 | 注視すべき状況 |
|-------|-------------|---------------|
| `Active(file)` | 最近アクセスされたファイルキャッシュpage | 高い = ワーキングセットがキャッシュされている |
| `Inactive(file)` | 最近アクセスされていないファイルキャッシュ | 回収可能な予備 |
| `Mapped` | アドレス空間にmmapされたファイル | 共有ライブラリ、mmapされたデータファイル |
| `Shmem` | 共有メモリ + tmpfs | `Cached`に計上される — よくある驚き |

### カーネルメモリ {#kernel-memory_1}

| フィールド | 説明 | 注視すべき状況 |
|-------|-------------|---------------|
| `Slab` | カーネルslabアロケータのメモリ総量 | 際限なく増加 = カーネルのリーク |
| `SReclaimable` | 回収可能なslab（dentry、inodeキャッシュ） | `MemAvailable`に計上される |
| `SUnreclaim` | 回収不能なslab | ネットワーク負荷の高いワークロードで増えうる |
| `KernelStack` | 全スレッドのカーネルスタック | スレッド数とともに増加 |
| `PageTables` | ページテーブルに使われるメモリ | プロセス数とアドレス空間サイズとともに増加 |
| `VmallocUsed` | vmalloc領域の使用量 | カーネルモジュール、一部のドライバ |
| `Percpu` | per-CPU割り当て | CPU数とともに増加 |

### スワップ {#swap}

| フィールド | 説明 | 注視すべき状況 |
|-------|-------------|---------------|
| `SwapTotal` | スワップ領域の総量 | ゼロ = スワップ未設定 |
| `SwapFree` | 未使用のスワップ | `SwapTotal - SwapFree` = 使用中のスワップ |
| `SwapCached` | まだRAMにあるスワップ済みpage | スワップインのコストを減らす |

### その他 {#misc}

| フィールド | 説明 | 注視すべき状況 |
|-------|-------------|---------------|
| `Dirty` | 変更されたがまだディスクに書き込まれていないpage | 大きい値 = ライトバックが追いついていない |
| `Writeback` | ディスクへ書き込み中のpage | 一時的であるべき。持続 = I/Oが遅い |
| `CommitLimit` | コミット可能な最大メモリ（モード2のみ） | [メモリオーバーコミット](overcommit.md)を参照 |
| `Committed_AS` | プロセスに「約束」されたメモリの総量 | モード2では`CommitLimit`を超えるとENOMEMになる |
| `HugePages_Total` | 事前割り当てされたhuge page（2 MB） | 静的割り当て。通常用途には使えない |
| `HugePages_Free` | 未使用のhuge page | 割り当てたが使われていなければ無駄 |
| `DirectMap4k` / `2M` / `1G` | カーネルのダイレクトマップのページサイズ | TLB性能のチューニングに関係する |

## よくある誤解 {#common-misconceptions}

**「MemFreeが低い、メモリが足りなくなっている。」**
違う。カーネルは意図的に空きメモリをページキャッシュに使っている。見るべき指標は`MemAvailable`である。`MemFree`が200 MBで`MemAvailable`が8 GBのシステムには十分なメモリがある。

**「キャッシュされたメモリは無駄だ。」**
逆である — キャッシュされたメモリはファイルI/Oを高速化する。アプリケーションが必要とすれば自動的に回収される。ページキャッシュのないシステムは、すべてをディスクから読まなければならない。

**「Buffers + Cachedがキャッシュの総量だ。」**
歴史的にはおおむね正しいが、`Cached`には回収可能なキャッシュではない`Shmem`（共有メモリ、tmpfs）が含まれる。実際の回収可能量は`Cached - Shmem + Buffers + SReclaimable`に近い。まさにこれが`MemAvailable`が追加された理由であり、自分で計算しなくて済むようにするためである。

**「SwapCachedがあるのはスワップしているということだ。」**
必ずしもそうではない。`SwapCached`は、スワップアウトされたpageがRAMにも保持されていることを意味する。システムはどこかの時点でスワップしたが、`vmstat`の`si`/`so`がゼロなら、現在活発にスワップしているわけではない。`SwapCached`は、それらのpageが再び必要になった場合にスワップから再読み込みするのを回避する。

**「全プロセスのRSSを足せば総メモリ使用量になる。」**
違う。RSSは共有page（共有ライブラリ、共有メモリ）を二重に数える。プロセスごとの公平な計上には`/proc/<pid>/smaps_rollup`のPSS（Proportional Set Size）を使うか、システム全体の合計には`/proc/meminfo`の`AnonPages` + `Cached` + `Slab`を見る。[仮想・物理・常駐](virtual-physical-resident.md)を参照。

## クイックリファレンスコマンド {#quick-reference-commands}

```bash
# 必須項目 — 1つのコマンドで
grep -E 'MemTotal|MemFree|MemAvailable|Cached|SwapTotal|SwapFree|AnonPages|Slab' /proc/meminfo

# システムは今スワップしているか?
vmstat 1 5   # si/so 列を見る

# メモリ逼迫の要約
cat /proc/pressure/memory   # （Linux 4.20+、PSI 有効時）

# ページキャッシュを破棄する（テスト専用 — 解決策ではない）
echo 3 > /proc/sys/vm/drop_caches

# メモリの傾向を観察する
watch -n 5 'grep -E "MemAvailable|AnonPages|Slab|Dirty" /proc/meminfo'

# プロセスごとのメモリ使用量（ソート済み）
ps aux --sort=-%mem | head -10

# プロセスごとの詳細な内訳
cat /proc/<pid>/smaps_rollup

# 人間向けの要約（MemAvailable を使う）
free -h
```

## free(1)と/proc/meminfoの対応 {#how-free1-maps-to-procmeminfo}

`free`コマンドは`/proc/meminfo`を読み、要約を表示する。

```
              total        used        free      shared  buff/cache   available
Mem:           15Gi       4.1Gi       195Mi       870Mi        11Gi       8.3Gi
Swap:         8.0Gi          0B       8.0Gi
```

| freeの列 | /proc/meminfoの出所 |
|-------------|---------------------|
| `total` | `MemTotal` |
| `free` | `MemFree` |
| `shared` | `Shmem` |
| `buff/cache` | `Buffers` + `Cached` + `SReclaimable` |
| `available` | `MemAvailable` |
| `used` | `MemTotal` - `MemFree` - `Buffers` - `Cached` - `SReclaimable` |

キャパシティに関する判断で重要なのは`available`列である。

## 参考文献 {#further-reading}

- [`fs/proc/meminfo.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/proc/meminfo.c) — これらの数値の出所
- [コミット`34e431b0ae39`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=34e431b0ae398fc54ea69ff85ec700722c9da773) — MemAvailableの追加
- [ページキャッシュ](page-cache.md) — ファイルキャッシュの仕組み
- [ページ回収](reclaim.md) — メモリが少なくなると何が起きるか
- [メモリオーバーコミット](overcommit.md) — CommitLimitとCommitted_ASの解説
- [仮想・物理・常駐](virtual-physical-resident.md) — RSS、VSZ、PSS
- [メモリ不足](oom.md) — メモリ不足からOOM killに至る経路
- [/proc/meminfoのドキュメント](https://www.kernel.org/doc/Documentation/filesystems/proc.rst) — カーネル公式ドキュメント
