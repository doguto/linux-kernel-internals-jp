# 1GBヒュージページ {#1gb-huge-pages}

> 2MBページでさえTLB圧迫が残るワークロード向けの、PUDレベルのヒュージページ

## 1GBページとは何か {#what-are-1gb-pages}

x86-64では、CPUのページテーブルウォーカーは3つのレベルでアドレス変換を終端できる:

```
PGD → PUD → PMD → PTE → 4KB page
PGD → PUD → PMD →        2MB huge page  (PMD-level)
PGD → PUD →               1GB huge page  (PUD-level)
```

**1GBヒュージページ**は、1ギガバイト全体を単一のPUDエントリでマップする。1つのTLBエントリで1GBの仮想アドレス空間をカバーでき、これは2MBページの512倍、4KBページの262,144倍のカバー範囲である。

これはx86-64でサポートされる最大のページサイズである。明示的なCPUサポートと明示的な予約が必要であり、何も自動的には行われない。

### CPU要件 {#cpu-requirement}

1GBページには`pdpe1gb` CPUIDフラグ（カーネルの機能名は`X86_FEATURE_GBPAGES`）が必要である。システムを確認するには:

```bash
grep pdpe1gb /proc/cpuinfo
```

おおよそ2010年以降に製造されたx86-64 CPU（Intel Westmere、AMD Barcelona以降）であればサポートしている。カーネルは`arch/x86/mm/hugetlbpage.c`で、このフラグに基づいて1GBページのサポートを制御している:

```c
bool __init arch_hugetlb_valid_size(unsigned long size)
{
    if (size == PMD_SIZE)
        return true;
    else if (size == PUD_SIZE && boot_cpu_has(X86_FEATURE_GBPAGES))
        return true;
    else
        return false;
}
```

## TLBカバー範囲: 中心的な動機 {#tlb-coverage-the-core-motivation}

TLBが保持できる仮想アドレスから物理アドレスへの変換の数には限りがある。TLBが一杯になると、CPUはメモリ上のページテーブルをウォークしなければならない。これは複数ステップの操作であり、最新のハードウェアではミス1回あたり50〜200サイクルかかることがある。

1GBのワーキングセットに必要なTLBエントリ数:

| ページサイズ | 1GBに必要なTLBエントリ数 |
|-----------|---------------------|
| 4KB       | 262,144             |
| 2MB       | 512                 |
| 1GB       | 1                   |

100GBのデータベースバッファプール（大規模なデータセットを持つOracle、SAP HANA、Redisでは現実的な数字）の場合:

| ページサイズ | 100GBに必要なTLBエントリ数 |
|-----------|-----------------------|
| 4KB       | ~26,200,000           |
| 2MB       | 51,200                |
| 1GB       | 100                   |

Skylake CPUの典型的なL2 TLB（STLB）は、4KBページ用に1,536エントリ、ラージページ用に32エントリを保持する（出典: [WikiChip Skylake server](https://en.wikichip.org/wiki/intel/microarchitectures/skylake_(server)#Memory_Hierarchy)、[7-cpu.com](https://www.7-cpu.com/cpu/Skylake.html)）。100GBのワーキングセットを2MBページでカバーするには51,200のTLBエントリが必要だが、ハードウェアにはラージページ専用のTLBエントリが32しかない。1GBページなら、100エントリでプール全体をカバーできる。

TLBの仕組みと2MBページでのTLBの計算について詳しくは、[Transparent Huge Pages](thp.md)と[hugetlbfs vs THP](hugetlbfs-vs-thp.md)を参照。

## 割り当て: （実用上は）ブート時のみ {#allocation-boot-time-only-practically-speaking}

### なぜ1GBページは実行時に割り当てられないか {#why-1gb-pages-cannot-be-allocated-at-runtime}

1GBページには、1GBの物理境界にアラインされた1GBの**物理的に連続した**メモリが必要である。メモリの断片化がない起動直後のシステムであれば、これは実現可能である。しかし稼働を続けているシステム、つまりカーネル、ドライバ、ファイルキャッシュ、アプリケーションがすでに物理メモリ全体に断片化して配置されているシステムでは、空いていて連続し、かつアラインされた1GBの領域を見つけることはほぼ不可能である。

カーネルはこのサイズのページを「gigantic」と呼ぶ。その割り当てorder（`PUD_SHIFT - PAGE_SHIFT` = 30 - 12 = order-18）が`MAX_PAGE_ORDER`（通常10または11）を超えるためである。バディアロケータはこれほど大きなブロックをまったく管理しない。

`CONFIG_CONTIG_ALLOC`が有効であれば、実行時の割り当ても技術的には可能である。カーネルは`alloc_contig_pages()`またはCMA（Contiguous Memory Allocator）を使い、既存のページをマイグレーションして退かすことで連続した領域を作り出せる。実際には、これはメモリが断片化する前の起動直後にしか成功しない。負荷のかかった本番システムでは、ほぼ常に何も言わずに失敗する。`nr_hugepages`に数を書き込んでも、`free_hugepages`は0のままである。

### ブート時の予約 {#boot-time-reservation}

メモリアロケータが何も触れる前に、カーネルコマンドラインで1GBページを予約する:

```
# Method 1: explicit size
hugepagesz=1G hugepages=N

# Method 2: set 1G as the default huge page size
default_hugepagesz=1G hugepages=N
```

`hugepagesz`と`default_hugepagesz`はどちらも`mm/hugetlb.c`で`early_param`としてパースされる（関数`hugepagesz_setup`と`default_hugepagesz_setup`）。これらはページアロケータが完全に初期化される前に実行されるため、実行時の予約が失敗する状況でもブート時の予約は成功する。

コマンドラインでサイズを組み合わせれば、2MBと1GBの両方のプールを予約できる:

```
hugepagesz=1G hugepages=16 hugepagesz=2M hugepages=1024
```

!!! warning "順序が重要"
    `hugepages=N`は、直前に指定された`hugepagesz`に適用される。どの`hugepagesz=`よりも前に置かれた`hugepages=`は、デフォルトのページサイズ（通常は2MB）に適用される。常に明示的にペアで指定すること。

## 実行時の失敗: どのように見えるか {#runtime-failure-what-it-looks-like}

起動後、1GBのプールは次の場所にある:

```
/sys/kernel/mm/hugepages/hugepages-1048576kB/
```

（1048576 kB = 1GB）

ブート時の予約をしていない起動済みシステムで、実行時の割り当てを試みた場合:

```bash
# 実行時に1GBページを4つ割り当てようとする
echo 4 > /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages

# 実際に何が起きたかを確認する
cat /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages   # 実際のプールサイズを表示（0〜4の可能性あり）
cat /sys/kernel/mm/hugepages/hugepages-1048576kB/free_hugepages # プール内の未使用ページを表示
```

`nr_hugepages`は、要求した目標値ではなく、プール内の実際のヒュージページ数を示す。4つを要求しても（断片化のために）カーネルが2つしか割り当てられなければ、`nr_hugepages`は2を示す。`free_hugepages`は、プールのページのうち現在どのプロセスにもマップされていないものの数を示す。負荷のかかったシステムでは、1GBページの実行時割り当てはほぼ常に完全に失敗する。

!!! note "CONFIG_CONTIG_ALLOC"
    一部のディストリビューションは`CONFIG_CONTIG_ALLOC`を有効にしており、これにより`alloc_contig_pages()`を介した実行時のgiganticページ割り当てが可能になる。メモリがまだ断片化していない起動直後であれば成功することもあるが、本番環境での運用で頼りにすべきではない。ブート時の予約が唯一の信頼できる方法である。

## NUMAへの影響 {#numa-implications}

各1GBページは特定のNUMAノード上に割り当てられる。ノード0で動作するプロセスがノード1上の1GBページにアクセスすると、**すべてのアクセス**でNUMAのリモートメモリペナルティを支払うことになる。1GBのページ境界より細かい粒度でのマイグレーションはない。

4KBや2MBのページであれば、カーネルのNUMAバランシングが個々のページや2MBのチャンクを、アクセスするCPUの近くにマイグレーションできる。1GBページにはNUMAバランシングは適用されない。ページはhugetlbプール内に固定されており、マイグレーションされることはない。

### ノードごとの予約 {#per-node-reservation}

1GBページは必ず、ワークロードが動作する特定のNUMAノード上に予約すること:

```bash
# ノード0に1GBページを8つ予約する
echo 8 > /sys/devices/system/node/node0/hugepages/hugepages-1048576kB/nr_hugepages

# ノード1に1GBページを8つ予約する
echo 8 > /sys/devices/system/node/node1/hugepages/hugepages-1048576kB/nr_hugepages
```

あるいは同等のことを、アプリケーション起動時に`numactl`で行う:

```bash
numactl --membind=0 --cpunodebind=0 ./myapp
```

ノードごとの1GBページの使用状況を確認する:

```bash
# ノードごとに割り当てられたページ
cat /sys/devices/system/node/node0/hugepages/hugepages-1048576kB/free_hugepages
cat /sys/devices/system/node/node1/hugepages/hugepages-1048576kB/free_hugepages

# 全ノードの集計
numastat -m | grep -i huge
```

## 1GBページが2MBページより性能で勝るとき {#when-1gb-pages-outperform-2mb-pages}

2MBページはすでに、4KBページに比べてTLB圧迫を512分の1に減らしている。1GBページに切り替えると、さらに512分の1に減る。ただしこれが意味を持つのは、2MBに切り替えた後もTLBミスがボトルネックであり続ける場合だけである。

**恩恵を受けるワークロード:**

- **インメモリデータベース** — Oracle Database、SAP HANA、および約100GBを超えるバッファプールを持つ同様のシステム。バッファプールはランダムにアクセスされるため、ワーキングセットがTLB容量を超えると、2MB粒度ではキャッシュヒットのたびにTLBミスが発生する。
- **大規模なモデル重みを扱うML学習** — 大規模言語モデルの学習では、繰り返しアクセスされる数ギガバイトの重みテンソルをロードする。1GBページなら、transformer層の重みテンソル全体がほんの数個のTLBエントリに収まる。
- **密行列を扱うHPC** — 大きな行列に対して密な線形代数演算（DGEMM、FFT）を行う科学技術計算コードでは、行列の次元によってワーキングセットが数百ギガバイトに達する場合、1GBページによって測定可能な高速化が得られる。
- **DPDKと高速パケット処理** — DPDKは長年、メモリプールに1GBページを推奨してきた。単一のTLBエントリでパケットバッファプール全体をカバーでき、パケットディスクリプタの参照でTLBミスがゼロになることはスループットとして測定できる。

**経験則:** 次の場合に1GBページを検討する:
1. ワーキングセットが約100GBより大きい、**かつ**
2. `perf stat -e dTLB-load-misses`でTLBミス率を測定でき、それが実行時間の無視できない割合を占めている、**かつ**
3. 2MBページ（THPまたはhugetlbfs）をすでに使っているのに、TLBミスが解消しない

```bash
# 1GBページへの切り替え前後でTLBミス率を測定する
perf stat -e cycles,dTLB-load-misses,dTLB-load-misses:u ./workload
```

## 1GBページを使うべきでないとき {#when-not-to-use-1gb-pages}

1GBページは確約である。多くのワークロードでは使えない大きな欠点がある。

**メモリの無駄。** 100MBのデータセットのために割り当てた1GBページは、900MBの物理RAMを無駄にする。それはアプリケーションが終了してページがプールに戻るまで、恒久的に続く。2MBページであれば、無駄は割り当てごとに最大2MBである。

**スワップされない。** hugetlbページ（2MB・1GBとも）は決してスワップされない。アプリケーションがマップしているもののほとんどアクセスしない1GBページは、物理RAMに無期限に居座る。メモリ圧迫のあるマシンでは、これが他のワークロードをスワップやOOMに追いやることがある。

**プール自体の断片化。** 各1GBページは、どのアプリケーションにもマップされていないときでもプールに確保されたままである。予約済みだがマップされていない1GBページが10個あれば、10GBのRAMが遊んでいることになる。

**コンテナとオーバーコミット。** コンテナ環境は通常、メモリのオーバーコミット、すなわち実在する以上のメモリを約束し、典型的な使用量が予約総量より少ないことを当てにする仕組みに依存している。hugetlbページはオーバーコミットできず、プールは厳格な上限となる。また1GBという粒度では、メモリ要件がまちまちなコンテナに合わせてプールを正確にサイジングすることも不可能である。

**小規模または中規模のワークロード。** 4GBのワーキングセットを持つプロセスは、1GBページなら4つのTLBエントリを使う。同じプロセスが2MBページなら2,048のTLBエントリを使うが、これは最新のSTLBの容量内に十分収まる。小さなワーキングセットでは、1GBページのオーバーヘッドに見合うTLB上の利点はない。

!!! tip "ほとんどのワークロードでは: まず2MBから始める"
    まだ2MBのhugetlbfsページを導入していないなら、まずそれを行うこと。運用上のオーバーヘッドがはるかに低く（実行時の割り当てが可能）、より複雑な1GBのセットアップに踏み切る前に、TLB上の利点がどれだけ得られるかを測定できる。2MBページのセットアップと比較については[hugetlbfs vs THP](hugetlbfs-vs-thp.md)を参照。

## アプリケーションから1GBページを使う {#using-1gb-pages-from-applications}

### hugetlbfsのマウント {#hugetlbfs-mount}

```bash
# 1GBページ用のマウントポイントを作成する
mkdir -p /dev/hugepages-1G

# ページサイズを明示してマウントする
mount -t hugetlbfs -o pagesize=1G nodev /dev/hugepages-1G
```

### MAP_HUGE_1GBを指定したmmap() {#mmap-with-map_huge_1gb}

```c
#include <sys/mman.h>

/* 1GBヒュージページを使って2GBをマップする（MAP_HUGE_1GB = 30U << 26 = 0x78000000） */
void *ptr = mmap(NULL, 2UL * 1024 * 1024 * 1024,
                 PROT_READ | PROT_WRITE,
                 MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB | MAP_HUGE_1GB,
                 -1, 0);
if (ptr == MAP_FAILED) {
    perror("mmap");  /* プールが枯渇しているか、予約されていない */
    exit(1);
}
```

`MAP_HUGE_1GB`は`include/uapi/linux/mman.h`で`HUGETLB_FLAG_ENCODE_1GB`として定義されており、これは`(30U << HUGETLB_FLAG_ENCODE_SHIFT)` = `(30U << 26)` = `0x78000000`である。値30はlog2(1GB) = 30をエンコードしている。

### hugetlbfs上のファイル（プロセス間の共有メモリ） {#file-on-hugetlbfs-shared-memory-between-processes}

```c
/* プロセス1: 裏付けとなるファイルを作成する */
int fd = open("/dev/hugepages-1G/shm-buffer", O_CREAT | O_RDWR, 0600);
ftruncate(fd, 4UL * 1024 * 1024 * 1024);  /* 4GB = 1GBページ × 4 */

void *ptr = mmap(NULL, 4UL * 1024 * 1024 * 1024,
                 PROT_READ | PROT_WRITE,
                 MAP_SHARED, fd, 0);

/* プロセス2: 同じファイルにアタッチする */
int fd2 = open("/dev/hugepages-1G/shm-buffer", O_RDWR);
void *ptr2 = mmap(NULL, 4UL * 1024 * 1024 * 1024,
                  PROT_READ | PROT_WRITE,
                  MAP_SHARED, fd2, 0);
```

hugetlbfsファイルを使う手法は、OracleのSGAや、DPDKの共有パケットメモリで使われている。

## 1GBページの使用状況を観測する {#observing-1gb-page-usage}

### /proc/meminfo {#procmeminfo}

`/proc/meminfo`は**デフォルト**のヒュージページサイズのプールしか表示しない。デフォルトが2MBのシステム（一般的なケース）では、`HugePages_Total`は1GBプールではなく2MBプールを反映する:

```bash
$ grep -i huge /proc/meminfo
AnonHugePages:    204800 kB    # THP (not hugetlbfs)
HugePages_Total:    1024       # default pool (2MB if default_hugepagesz=2M)
HugePages_Free:      900
HugePages_Rsvd:       50
HugePages_Surp:        0
Hugepagesize:       2048 kB    # confirms default is 2MB
```

デフォルト設定にかかわらず1GBプールを見るには、サイズ別のsysfsディレクトリを使う:

```bash
$ ls /sys/kernel/mm/hugepages/hugepages-1048576kB/
free_hugepages  nr_hugepages  nr_hugepages_mempolicy  nr_overcommit_hugepages  resv_hugepages  surplus_hugepages

$ cat /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages
16

$ cat /sys/kernel/mm/hugepages/hugepages-1048576kB/free_hugepages
4
```

### numastatによるノードごとの確認 {#per-node-via-numastat}

```bash
# 全ヒュージページサイズのノードごとの内訳
numastat -m

# 出力例（抜粋）:
# Node 0  Node 1   Total
# HugePages_Total   8.00    8.00   16.00
# HugePages_Free    2.00    2.00    4.00
```

### sysfsによるノードごとの確認 {#per-node-via-sysfs}

```bash
for node in /sys/devices/system/node/node*/hugepages/hugepages-1048576kB; do
    echo "$node:"
    echo "  nr:   $(cat $node/nr_hugepages)"
    echo "  free: $(cat $node/free_hugepages)"
done
```

### プロセスごと {#per-process}

```bash
# 1GBヒュージページを使うマッピングは、smapsでKernelPageSize: 1048576 kBとして表示される
grep -A5 "KernelPageSize" /proc/<pid>/smaps | grep -B1 "1048576"
```

## 主要なソースファイル {#key-source-files}

| ファイル | 説明 |
|------|-------------|
| [`mm/hugetlb.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/hugetlb.c) | hugetlbプールの管理: `hugepagesz_setup`、`default_hugepagesz_setup`、`alloc_gigantic_frozen_folio`、`hugetlb_gigantic_pages_alloc_boot` |
| [`include/linux/hugetlb.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/hugetlb.h) | `hstate_is_gigantic()`、`hugepage_migration_supported()` |
| [`arch/x86/mm/hugetlbpage.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/hugetlbpage.c) | `arch_hugetlb_valid_size()` — `X86_FEATURE_GBPAGES`に基づいて1GBサポートを制御する。`gigantic_pages_init()` — `CONFIG_CONTIG_ALLOC`が設定されているときにPUDレベルのhstateを登録する |
| [`arch/x86/include/asm/cpufeatures.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/cpufeatures.h) | `X86_FEATURE_GBPAGES`（`"pdpe1gb"`） — CPUの機能フラグ |
| [`include/uapi/linux/mman.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/mman.h) | `MAP_HUGE_1GB` — 1GBページ用のmmapフラグ |
| [`include/uapi/asm-generic/hugetlb_encode.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/asm-generic/hugetlb_encode.h) | `HUGETLB_FLAG_ENCODE_1GB` = `(30U << 26)` |

## 歴史 {#history}

### 1GBページのサポート（v2.6.26、2008年） {#1gb-page-support-v2626-2008}

x86-64での1GBページのサポートはLinux 2.6.26で導入された。AMD Barcelona（Family 10h）は`PDPE1GB` CPUIDビットを実装した最初のx86 CPUであり、このビットはLinuxでは`X86_FEATURE_GBPAGES`（cpufeature文字列`"pdpe1gb"`）として公開されている。

**LWNの記事**: [Huge pages part 3: Administration](https://lwn.net/Articles/376606/)は、1GBページのサポートに伴うNUMAプールの管理を扱っている。

### MAP_HUGE_1GBフラグ（v3.8、2013年） {#map_huge_1gb-flag-v38-2013}

**コミット**: [42d7395feb56](https://git.kernel.org/linus/42d7395feb56) ("mm: support more pagesizes for MAP_HUGETLB/SHM_HUGETLB")

このコミット以前は、1GBページを得るにはアプリケーションが`pagesize=1G`のhugetlbfsマウント上のファイルを開く必要があった。このコミットはmmapフラグにページサイズのエンコーディングを追加し、マウントポイントなしで`MAP_HUGETLB | MAP_HUGE_1GB`を使えるようにした。

## 参考資料 {#references}

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/admin-guide/mm/hugetlbpage.rst`](https://docs.kernel.org/admin-guide/mm/hugetlbpage.html) — ブートパラメータとsysfsインターフェースを含むhugetlbfsの管理者ガイド

### LWNの記事 {#lwn-articles}

- [Huge pages part 1: Introduction](https://lwn.net/Articles/374424/)（2010年）
- [Huge pages part 2: Interfaces](https://lwn.net/Articles/375096/)（2010年） — mmapフラグとlibhugetlbfs
- [Huge pages part 3: Administration](https://lwn.net/Articles/376606/)（2010年） — プール管理とNUMA

### 関連ページ {#related-pages}

- [hugetlbfs vs THP](hugetlbfs-vs-thp.md) — 2MBページの比較、プール管理、hugetlbfsとTHPの使い分け
- [Transparent Huge Pages](thp.md) — 2MBのPMDレベルTHPの内部とチューニング
- [ページテーブル](page-tables.md) — PUDレベルのページテーブルエントリの構造
- [NUMA](numa.md) — ノードごとのメモリ割り当てとNUMAトポロジ
- [ページアロケータ](page-allocator.md) — なぜバディアロケータは実行時に1GBの割り当てを満たせないか

## 参考文献 {#further-reading}

- `Documentation/admin-guide/mm/hugetlbpage.rst` — hugetlbfsに関するカーネルの正式な管理者ガイド。ブートパラメータ（`hugepagesz=`、`default_hugepagesz=`）、sysfsインターフェース、ノードごとの予約、`MAP_HUGETLB` / `MAP_HUGE_1GB`フラグを扱う。[docs.kernel.org](https://docs.kernel.org/admin-guide/mm/hugetlbpage.html)で閲覧できる
- [mm/hugetlb.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/hugetlb.c) — hugetlbプールの管理。`hugepagesz_setup()`、`default_hugepagesz_setup()`、`alloc_gigantic_frozen_folio()`、およびearly_paramによる予約パスのための`hugetlb_gigantic_pages_alloc_boot()`
- [arch/x86/mm/hugetlbpage.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/hugetlbpage.c) — `X86_FEATURE_GBPAGES`に基づいて1GBサポートを制御する`arch_hugetlb_valid_size()`と、PUDレベルのhstateを登録する`gigantic_pages_init()`
- [LWN: Huge pages part 1: Introduction](https://lwn.net/Articles/374424/) — なぜヒュージページがLinuxに追加されたか、そして当初のデータベースベンダーの動機についての背景
- [LWN: Huge pages part 3: Administration](https://lwn.net/Articles/376606/) — NUMAプールの管理とノードごとの予約。1GBページのサポートが成熟しつつあった時期に書かれた
- コミット[42d7395feb56](https://git.kernel.org/linus/42d7395feb56) — v3.8での`MAP_HUGE_1GB`とmmapフラグへのページサイズエンコーディングの追加。`pagesize=1G`のhugetlbfsマウントの必要性をなくした
- [hugetlbfs-vs-thp.md](hugetlbfs-vs-thp.md) — 2MBのTHPやmTHPではなく1GBのhugetlbfsを選ぶべきとき。事前予約プールの運用上のトレードオフを含む
- [thp.md](thp.md) — 2MBのPMDレベルTHP: 1GBページに踏み切る前の、オーバーヘッドの低い出発点
