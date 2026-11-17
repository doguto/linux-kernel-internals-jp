# 明示的なhugetlbfsと透過的ヒュージページ（THP） {#explicit-hugetlbfs-vs-transparent-huge-pages}

> Linuxにおけるヒュージページの2つのアプローチ: 自ら要求するものと、自動的に得られるもの

## なぜヒュージページか {#why-huge-pages}

現代のCPUは、仮想アドレスから物理アドレスへの変換をTLB（Translation Lookaside Buffer）にキャッシュする。標準の4KBページでは、数ギガバイトのメモリに触れるワークロードは数十万のページテーブルエントリを必要とし、TLBが保持できる数をはるかに超える。TLBミスのたびに、コストの高い多段のページテーブルウォークが発生する。

ヒュージページ（x86-64では2MBまたは1GB）はTLBへの圧力を劇的に減らす。

| ページサイズ | 1GBに必要なページ数 | 必要なTLBエントリ数 |
|-----------|---------------|--------------------|
| 4KB       | 262,144       | 262,144            |
| 2MB       | 512           | 512                |
| 1GB       | 1             | 1                  |

Linuxはヒュージページを使うための2つの異なるメカニズムを提供している。どちらも同じハードウェアサポート（PMDレベルとPUDレベルのページテーブルエントリ）を利用するが、管理の仕方が根本的に異なる。

## 2つのアプローチ {#the-two-approaches}

### hugetlbfs: 明示的で事前割り当てされるヒュージページ {#hugetlbfs-explicit-preallocated-huge-pages}

hugetlbfsは元祖のヒュージページメカニズムである。管理者がブート時または実行時にヒュージページのプールを予約し、アプリケーションがそれを明示的に要求する。自動的に行われることは何もない。

```
Administrator                          Application
     │                                      │
     ├── Reserve pool                       │
     │   (boot param or sysctl)             │
     │                                      │
     ├── Mount hugetlbfs                    │
     │   mount -t hugetlbfs ...             │
     │                                      │
     │                                      ├── mmap() with MAP_HUGETLB
     │                                      │   or open file on hugetlbfs
     │                                      │
     │                                      ├── Use memory
     │                                      │
     │                                      └── munmap() / close
     │                                      │
     └── Pages return to pool               │
```

カーネルはこれらのページを割り当て時に予約し、通常のページアロケータから完全に取り除く。それらは[`mm/hugetlb.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/hugetlb.c)が管理する別のプールに置かれ、他の用途には使えない。

### THP: 透過的でオンデマンドなヒュージページ {#thp-transparent-on-demand-huge-pages}

透過的ヒュージページ（Transparent Huge Pages）は、アプリケーションの変更を一切必要としない。カーネルは可能なときには4KBページを2MBのヒュージページに自動的に昇格させ、必要なときには分割して元に戻す。

```
Application                            Kernel
     │                                      │
     ├── malloc() / mmap()                  │
     │   (normal allocation)                │
     │                                      │
     ├── Touch memory                       ├── Page fault
     │                                      │   Can allocate 2MB?
     │                                      │   ├── Yes: map huge page
     │                                      │   └── No: fall back to 4KB
     │                                      │
     │                                      ├── khugepaged scans later
     │                                      │   Collapse 512 x 4KB → 2MB
     │                                      │
     └── free() / munmap()                  └── Pages return to buddy
```

THPのページは通常のバディアロケータから得られる。カーネルは、2MBアラインされた連続領域をオンデマンドで見つける（あるいはコンパクションで作り出す）。中核の実装は[`mm/huge_memory.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/huge_memory.c)を参照のこと。

## 並べて比較 {#side-by-side-comparison}

| 特徴 | hugetlbfs | THP |
|---------|-----------|-----|
| **ページサイズ** | 2MBと1GB（x86-64）。アーキテクチャ依存 | 2MB（PMD）。mTHPでは16KB〜1MB（v6.8以降） |
| **アプリケーションの変更** | 必要: `MAP_HUGETLB`、hugetlbfs上のファイル、またはlibhugetlbfs | 不要（または任意で`MADV_HUGEPAGE`ヒント） |
| **割り当て** | 事前割り当てのプール。ブート時／実行時に予約 | バディアロケータからオンデマンドで |
| **スワップ可能か** | 不可 - ページはRAMに固定される | 可能（v4.13以降、1単位としてスワップされる） |
| **断片化のリスク** | 低い - プールを前もって予約する | 高い - フォールト時に連続メモリが必要 |
| **オーバーコミット** | 不可 - プールが枯渇すると割り当てが失敗する | 可能 - 4KBページにフォールバックする |
| **NUMAへの対応** | `hugepages_*` sysfsによるノードごとのプール | 割り当てはローカルノードを優先する。NUMAバランシングがスラッシングを引き起こすことがある |
| **メモリのアカウンティング** | 別のプールとして`/proc/meminfo`に表示される | 通常の匿名メモリの一部 |
| **ページキャッシュのサポート** | あり（hugetlbfs上のファイル） | 匿名メモリはあり。ファイルTHPは限定的 |
| **分割／collapse** | なし - 常にヒュージ | あり - カーネルが必要に応じて分割・collapseする |
| **レイテンシの予測可能性** | 高い - コンパクションもフォールバックもない | 低い - コンパクションによるストールがあり得る |
| **メモリが無駄になるリスク** | 消費されなければプールが未使用のまま残る | 2MBの粒度のため、小さな割り当てでメモリを無駄にし得る |

## hugetlbfsを使うべきとき {#when-to-use-hugetlbfs}

アプリケーションを自分で制御でき、ヒュージページを確実に得る必要があり、割り当て失敗やレイテンシのジッタを許容できない場合には、hugetlbfsが正しい選択である。

### データベース（Oracle、PostgreSQL、MySQL） {#databases-oracle-postgresql-mysql}

データベースの共有バッファは長寿命で、大きく、頻繁にアクセスされる。hugetlbfsはそれらに、決して分割もスワップもされない固定のヒュージページを与える。

```bash
# PostgreSQL: 共有バッファにヒュージページを使う
# postgresql.conf
huge_pages = on   # または通常ページへのフォールバックを許す "try"
```

PostgreSQLは共有メモリセグメントに`MAP_HUGETLB`付きの`mmap()`を使う。Oracleは2000年代初頭から、SGA（System Global Area）にhugetlbfsを使ってきた。

### DPDKとネットワークパケット処理 {#dpdk-and-network-packet-processing}

DPDK（Data Plane Development Kit）は、パケットバッファをhugetlbfs上にマップし、アドレスがわかっている固定の物理メモリを得る。これはネットワークハードウェアとのDMAにとって極めて重要である。

```bash
# DPDK は通常 1GB のヒュージページを必要とする
echo 4 > /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages
mount -t hugetlbfs -o pagesize=1G nodev /dev/hugepages-1G
```

ここでは1GBページが特に有用である。1つのTLBエントリでパケットバッファプール全体をカバーできる。

### KVM/QEMU仮想マシン {#kvmqemu-virtual-machines}

VMはゲストメモリにヒュージページを使うことで恩恵を受ける。TLBエントリが少なくなれば、ネストしたアドレス変換（EPT/NPT）によるオーバーヘッドも減る。

```bash
# hugetlbfs をバックエンドとするゲストメモリで QEMU を起動する
qemu-system-x86_64 -m 4G \
    -mem-path /dev/hugepages \
    -mem-prealloc ...
```

## THPを使うべきとき {#when-to-use-thp}

THPは、アプリケーションを変更せずにヒュージページの恩恵を得たく、ときおり4KBページへフォールバックすることを許容できるワークロードに最も適している。

### 一般的なサーバーワークロード {#general-server-workloads}

変更や再コンパイルができないアプリケーションに対して、THPは自動的に恩恵をもたらす。

```bash
# システム全体で THP を有効にする
echo always > /sys/kernel/mm/transparent_hugepage/enabled
```

カーネルは、すべての匿名メモリに対して機会があればヒュージページを使う。

### 大きなヒープを持つアプリケーション {#applications-with-large-heaps}

Java、Python、その他のマネージドランタイムのアプリケーションは、コードを変更せずにTHPの恩恵を受ける。特にJVMは、大きなヒープでのTLB圧力の低減により大きな効果が得られる。

アプリケーションを変更せずにきめ細かく制御するには:

```bash
# madvise モード + libhugetlbfs や LD_PRELOAD によるプロセスごとのヒントを使う
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
```

### KVM（透過モード） {#kvm-transparent-mode}

THPはもともとKVMのために設計された。最初のTHPコミット[71e3aac0724f](https://git.kernel.org/linus/71e3aac0724f)より:

> "Lately I've been working to make KVM use hugepages transparently without the usual restrictions of hugetlbfs."

（訳: 最近、hugetlbfsにつきものの制約なしに、KVMがヒュージページを透過的に使えるようにする作業をしてきた。）

VMはhugetlbfsのプールを予約することなく、ゲストメモリに自動的にヒュージページを得る。

## THPを無効にすべきとき {#when-to-disable-thp}

THPを使うと性能が悪化するワークロードもある。根本的な問題は、THPがバックグラウンドの作業（コンパクション、khugepagedの走査、ページの分割）を追加し、それが予測不能なレイテンシを引き起こし得ることである。

### レイテンシに敏感なアプリケーション（Redis、Memcached） {#latency-sensitive-applications-redis-memcached}

マイクロ秒単位でリクエストに応答するインメモリストアでは、カーネルがヒュージページの割り当てを満たすためにメモリコンパクションを実行すると、数ミリ秒のストールが発生し得る。

```bash
# Redis と Memcached のドキュメントはどちらも次を推奨している:
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

Redisのドキュメントは、THPをレイテンシとメモリ使用量の問題の原因として明示的に挙げている。

### 疎なメモリアクセスパターン {#sparse-memory-access-patterns}

大きな領域を割り当てるものの、2MBの領域ごとに数バイトしか触れないアプリケーションは、メモリを無駄にする。4KBしか使われなくても、カーネルは2MBのページを丸ごと割り当てる。

### リアルタイムおよび決定論的なワークロード {#real-time-and-deterministic-workloads}

スループットよりも最悪ケースのレイテンシが重要なワークロードでは、THPを無効にすべきである。コンパクションの所要時間には上限がない。

```bash
# 中間的な選択肢: 要求したアプリだけ THP を使う
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
echo madvise > /sys/kernel/mm/transparent_hugepage/defrag
```

## 設定 {#configuration}

### hugetlbfsのセットアップ {#hugetlbfs-setup}

#### プールの予約 {#reserve-the-pool}

```bash
# ブート時（カーネルコマンドライン）- 大きなプールには最も確実
hugepagesz=2M hugepages=1024
hugepagesz=1G hugepages=4

# 実行時（メモリが断片化していると失敗することがある）
echo 1024 > /proc/sys/vm/nr_hugepages                          # 2MB ページ
echo 4 > /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages  # 1GB ページ

# 現在の状態を確認する
cat /proc/meminfo | grep -i huge
# HugePages_Total:    1024
# HugePages_Free:     1024
# HugePages_Rsvd:        0
# HugePages_Surp:        0
# Hugepagesize:       2048 kB
```

#### hugetlbfsのマウント {#mount-hugetlbfs}

```bash
# デフォルトの 2MB ページサイズ
mount -t hugetlbfs nodev /dev/hugepages

# 1GB ページサイズを明示する
mount -t hugetlbfs -o pagesize=1G nodev /dev/hugepages-1G
```

ほとんどのディストリビューションは、systemdによってデフォルトのhugetlbfsを自動的にマウントする。

#### アプリケーションからの利用 {#use-from-applications}

```c
#include <sys/mman.h>

/* 方法 1: MAP_HUGETLB フラグ（最近のカーネルではマウント不要） */
void *ptr = mmap(NULL, size, PROT_READ | PROT_WRITE,
                 MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);

/* 方法 2: 1GB ページ */
void *ptr = mmap(NULL, size, PROT_READ | PROT_WRITE,
                 MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB | MAP_HUGE_1GB,
                 -1, 0);

/* 方法 3: hugetlbfs 上のファイル */
int fd = open("/dev/hugepages/myfile", O_CREAT | O_RDWR, 0600);
void *ptr = mmap(NULL, size, PROT_READ | PROT_WRITE,
                 MAP_SHARED, fd, 0);
```

#### NUMAを意識したプールの設定 {#numa-aware-pool-configuration}

```bash
# 特定の NUMA ノードにページを予約する
echo 512 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
echo 512 > /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages
```

### THPのセットアップ {#thp-setup}

```bash
# グローバルモード
echo always  > /sys/kernel/mm/transparent_hugepage/enabled   # すべての匿名メモリ
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled   # MADV_HUGEPAGE の領域のみ
echo never   > /sys/kernel/mm/transparent_hugepage/enabled   # 無効

# defrag の挙動（ヒュージページのためにいつメモリをコンパクションするか）
echo always        > /sys/kernel/mm/transparent_hugepage/defrag  # フォールト時に同期的に（ストールする）
echo defer         > /sys/kernel/mm/transparent_hugepage/defrag  # khugepaged により非同期に
echo defer+madvise > /sys/kernel/mm/transparent_hugepage/defrag  # 大半は遅延、ヒント付きは同期
echo madvise       > /sys/kernel/mm/transparent_hugepage/defrag  # MADV_HUGEPAGE の場合のみ
echo never         > /sys/kernel/mm/transparent_hugepage/defrag  # THP のためにコンパクションしない

# khugepaged のチューニング
echo 10000 > /sys/kernel/mm/transparent_hugepage/khugepaged/scan_sleep_millisecs
echo 4096  > /sys/kernel/mm/transparent_hugepage/khugepaged/pages_to_scan
```

すべてのチューニング可能なパラメータについては[THPのカーネルドキュメント](https://docs.kernel.org/admin-guide/mm/transhuge.html)を参照のこと。

## 監視 {#monitoring}

### hugetlbfs: /proc/meminfo {#hugetlbfs-procmeminfo}

```bash
$ grep -i huge /proc/meminfo
AnonHugePages:    204800 kB    # THP の使用量（hugetlbfs ではない!）
ShmemHugePages:        0 kB
ShmemPmdMapped:        0 kB
HugePages_Total:    1024       # hugetlbfs プールのサイズ
HugePages_Free:      900       # プール内の未使用ページ
HugePages_Rsvd:       50       # 予約済みだがまだフォールトしていない
HugePages_Surp:        0       # nr_hugepages を超える余剰ページ
Hugepagesize:       2048 kB    # デフォルトのヒュージページサイズ
```

主要な関係:

- **HugePages_Total - HugePages_Free** = 現在アプリケーションがマップしているページ
- **HugePages_Rsvd** = アプリケーションに約束されたが、まだフォールトインしていないページ
- **利用可能** = HugePages_Free - HugePages_Rsvd

```bash
# ノードごとの hugetlbfs の統計
cat /sys/devices/system/node/node0/hugepages/hugepages-2048kB/free_hugepages
cat /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
```

### THP: /sys/kernel/mm/transparent_hugepage/と/proc/vmstat {#thp-syskernelmmtransparent_hugepage-and-procvmstat}

```bash
$ grep thp /proc/vmstat
thp_fault_alloc 1234          # ページフォールト時に割り当てられたヒュージページ
thp_fault_fallback 56         # フォールト時に 4KB へフォールバックした
thp_collapse_alloc 789        # khugepaged が collapse したページ
thp_collapse_alloc_failed 12  # khugepaged の collapse 失敗
thp_split_page 45             # 4KB に分割し直されたヒュージページ
thp_zero_page_alloc 3         # 割り当てられたゼロヒュージページ
thp_swpout 0                  # （1 単位として）スワップアウトされたヒュージページ
thp_swpout_fallback 0         # スワップ時に先に分割するようフォールバックした
```

```bash
# プロセスごとの THP 使用量
grep AnonHugePages /proc/<pid>/smaps_rollup

# 現在の THP モード
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never
```

### 簡易ヘルスチェック {#quick-health-check}

```bash
#!/bin/bash
echo "=== hugetlbfs ==="
grep -i huge /proc/meminfo

echo ""
echo "=== THP ==="
echo "Mode: $(cat /sys/kernel/mm/transparent_hugepage/enabled)"
echo "Defrag: $(cat /sys/kernel/mm/transparent_hugepage/defrag)"
echo ""
echo "Allocations:"
grep -E "thp_fault_alloc|thp_fault_fallback|thp_collapse|thp_split" /proc/vmstat
```

## 歴史 {#history}

### hugetlbfs（v2.5.36、2002年） {#hugetlbfs-v2536-2002}

**コミット**: hugetlbfsはLinux 2.5の開発シリーズ中（git以前の時代、2002年）に追加された。

実装は[`fs/hugetlbfs/inode.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/hugetlbfs/inode.c)と[`mm/hugetlb.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/hugetlb.c)にある。もともとの動機はデータベースベンダー（特にOracleとIBM）からもたらされた。彼らは、ヒュージページを長らくサポートしていたプロプライエタリなUnixシステム（AIX、Solaris、HP-UX）の性能に匹敵するため、Linuxでヒュージページを必要としていた。

**LWNの記事**: [Huge pages part 1: Introduction](https://lwn.net/Articles/374424/)が、このメカニズムとその歴史の概要を説明している。

*注: Linux 2.5はgitより前の時代である（カーネルがgitに移行したのは2005年4月の2.6.12）。最初のhugetlbfsパッチにはコミットIDが存在しない。*

hugetlbfsの進化における主要なマイルストーン:

- **v2.6.32（2009年）**: mmapフラグとして`MAP_HUGETLB`が追加され、hugetlbfsのマウントポイントなしでヒュージページのマッピングが可能になった。コミット[e5ff2159a434](https://git.kernel.org/linus/e5ff2159a434)が使用例を追加している
- **v3.8（2013年）**: コミット[42d7395feb56](https://git.kernel.org/linus/42d7395feb56)（"mm: support more pagesizes for MAP_HUGETLB/SHM_HUGETLB"）- mmapフラグにページサイズをエンコードする（`MAP_HUGE_1GB`による1GBページを可能にする）

### THP（v2.6.38、2011年） {#thp-v2638-2011}

**コミット**: [71e3aac0724f](https://git.kernel.org/linus/71e3aac0724f)（"thp: transparent hugepage core"）

**作者**: Andrea Arcangeli（Red Hat）

最初のTHP実装は、KVM仮想マシン向けの匿名メモリを対象としていた。コミットメッセージは、その動機をhugetlbfsの制約を取り除きつつTLBの恩恵を維持することだと説明している。

**LWNの記事**: [Transparent huge pages](https://lwn.net/Articles/423584/)が、設計と初期の反応を扱っている。

THPの進化における主要なマイルストーン:

- **v2.6.38（2011年）**: VMAごとの制御のための`MADV_HUGEPAGE` / `MADV_NOHUGEPAGE`が、最初のTHPコミットで追加された
- **v4.13（2017年）**: コミット[38d8b4e6bdc8](https://git.kernel.org/linus/38d8b4e6bdc8)（"mm, THP, swap: delay splitting THP during swap out"）- THPのスワップサポート（Huang Ying）
- **v5.4（2019年）**: defragモードの改善（`defer+madvise`）
- **v6.8（2024年）**: コミット[19eaf44954df](https://git.kernel.org/linus/19eaf44954df)（"mm: thp: support allocation of anonymous multi-size THP"）- 4KBと2MBの間のページサイズを可能にするマルチサイズTHP（mTHP）（Ryan Roberts）

### 設計上の緊張関係 {#the-design-tension}

hugetlbfsとTHPは、システム設計における古典的なトレードオフを体現している。すなわち、**明示的な制御か自動最適化か**である。

hugetlbfsは管理者に完全な制御を与える。ヒュージページがいくつ存在し、どのアプリケーションがそれを使っているかを正確に把握でき、予想外のことは起きない。その代償は、運用の複雑さと、プールが大きすぎた場合のメモリの無駄である。

THPは運用の負担を取り除くが、予測不能性をもたらす。カーネルは、多くのワークロードに役立つ一方で一部のワークロードには害となり得る判断を下す。長年にわたるバグ修正とチューニング用のつまみ（defragモード、madviseヒント、mTHP）は、自動的なアプローチを確実に機能させるための継続的な取り組みを示している。

どちらのアプローチもなくなることはない。カーネル開発コミュニティは両方の改善を続けている。hugetlbfsはより優れたNUMAサポートとアカウンティングを得て、THPはより細かい粒度（mTHP）とより優れたヒューリスティクスを得ている。

## 自分で試す {#try-it-yourself}

```bash
# === hugetlbfs ===

# ヒュージページを 10 個予約する（各 2MB = 20MB）
echo 10 > /proc/sys/vm/nr_hugepages

# プールを確認する
grep HugePages /proc/meminfo

# hugetlbfs をマウントする（まだマウントされていない場合）
mkdir -p /mnt/huge
mount -t hugetlbfs nodev /mnt/huge

# MAP_HUGETLB を使う小さな C プログラムを書き、次で確認する:
grep HugePages /proc/meminfo   # HugePages_Free が減るはず

# === THP ===

# 現在の THP モードを確認する
cat /sys/kernel/mm/transparent_hugepage/enabled

# 大きな匿名領域を割り当てるプログラムを実行する:
#   void *p = mmap(NULL, 64*1024*1024, PROT_READ|PROT_WRITE,
#                  MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
#   memset(p, 0, 64*1024*1024);

# 実行中に次を確認する:
grep AnonHugePages /proc/<pid>/smaps_rollup
grep thp_fault_alloc /proc/vmstat

# === レイテンシを比較する ===

# THP defrag=always でレイテンシに敏感なワークロードを実行し、ストールを観察する:
echo always > /sys/kernel/mm/transparent_hugepage/defrag
# 次に以下を試す:
echo never > /sys/kernel/mm/transparent_hugepage/defrag
# テールレイテンシを比較する

# === khugepaged を観察する ===
# 1 つの端末で:
grep thp_collapse /proc/vmstat
# ワークロードを実行し、10 秒以上待ってから再度確認する
grep thp_collapse /proc/vmstat
# khugepaged がページを collapse するにつれてカウンタが増えるはず
```

## リファレンス {#references}

### 主要なコード {#key-code}

| ファイル | 説明 |
|------|-------------|
| [`mm/hugetlb.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/hugetlb.c) | hugetlbfsのプール管理とフォールト処理 |
| [`fs/hugetlbfs/inode.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/hugetlbfs/inode.c) | hugetlbfsファイルシステムの実装 |
| [`mm/huge_memory.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/huge_memory.c) | THPの中核実装 |
| [`mm/khugepaged.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/khugepaged.c) | THPのcollapseデーモン |

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/admin-guide/mm/hugetlbpage.rst`](https://docs.kernel.org/admin-guide/mm/hugetlbpage.html) - hugetlbfsの管理者ガイド
- [`Documentation/admin-guide/mm/transhuge.rst`](https://docs.kernel.org/admin-guide/mm/transhuge.html) - THPの管理者ガイド

### LWNの記事 {#lwn-articles}

- [Huge pages part 1: Introduction](https://lwn.net/Articles/374424/)（2010年）- Linuxにおけるヒュージページの概要
- [Huge pages part 2: Interfaces](https://lwn.net/Articles/375096/)（2010年）- hugetlbfsとlibhugetlbfs
- [Huge pages part 3: Administration](https://lwn.net/Articles/376606/)（2010年）- プール管理とNUMA
- [Transparent huge pages](https://lwn.net/Articles/423584/)（2011年）- THPの設計と導入
- [Multi-size THP](https://lwn.net/Articles/937239/)（2023年）- mTHPの動機と設計

### 関連項目 {#related}

- [透過的ヒュージページ](thp.md) - THPの内部、バグ、チューニングの詳解
- [コンパクション](compaction.md) - THPが依存するメモリのデフラグメンテーション
- [ページアロケータ](page-allocator.md) - 両方のメカニズムに供給するバディアロケータ
- [NUMA](numa.md) - ノードごとのヒュージページプールとTHPのNUMAとの相互作用
- [ページテーブル](page-tables.md) - PMDレベルとPUDレベルのヒュージページマッピング

## 参考文献 {#further-reading}

- [thp.md](thp.md) — THPの内部の詳解: `khugepaged`デーモン、フォールトベースの割り当て、defragモード、COWの競合バグ、NUMAとの相互作用
- [hugepages-1gb.md](hugepages-1gb.md) — 1GBのPUDレベルのヒュージページ: 実行時の割り当てがほぼ必ず失敗する理由、ノードごとの予約、そして512倍のTLB効果が運用コストに見合うワークロード（データベース、DPDK、HPC）
- [mthp.md](mthp.md) — マルチサイズTHP（Linux 6.10）: 4KBへのフォールバックと2MBのオーバーヘッドの間を埋める、16KBから512KBまでのPMD未満のサイズ。ARM64で特に有用
- `Documentation/admin-guide/mm/hugetlbpage.rst` — hugetlbfsの公式管理者ガイド: プールの予約、sysfsのつまみ、mmapフラグ、ノードごとのNUMA制御。[docs.kernel.org](https://docs.kernel.org/admin-guide/mm/hugetlbpage.html)でレンダリング版を閲覧できる
- `Documentation/admin-guide/mm/transhuge.rst` — THPの公式管理者ガイド: `enabled`と`defrag`のモード、khugepagedのチューニング、サイズごとのmTHP制御。[docs.kernel.org](https://docs.kernel.org/admin-guide/mm/transhuge.html)でレンダリング版を閲覧できる
- [LWN: Transparent huge pages](https://lwn.net/Articles/423584/) — THPの導入と、hugetlbfsに代わる透過的な選択肢というもともとの動機を扱った2011年の記事
- [LWN: Huge pages part 1: Introduction](https://lwn.net/Articles/374424/) — Linuxにおけるヒュージページの歴史と、hugetlbfsがデータベースベンダーに端を発することの概要
- [LWN: Multi-size THP](https://lwn.net/Articles/937239/) — mTHPの動機と、それが2つのメカニズムの間に残る運用上の差をどう縮めるか
- コミット[71e3aac0724f](https://git.kernel.org/linus/71e3aac0724f) — 最初のTHPコミット（v2.6.38）。そのメッセージはTHPを「the usual restrictions of hugetlbfs」（hugetlbfsにつきものの制約）に対する解決策として明確に位置づけている
