# /proc/meminfoを理解する {#understanding-procmeminfo}

> 各フィールドの意味と、それを診断に使う方法

## /proc/meminfoとは何か {#what-is-procmeminfo}

`/proc/meminfo` はシステムのメモリ状態についてのカーネルのレポートである。`fs/proc/meminfo.c` の [`meminfo_proc_show()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/proc/meminfo.c) によって生成され、複数のカーネルサブシステムから読み取って1つのスナップショットを作る。

```bash
$ cat /proc/meminfo
MemTotal:       16384000 kB
MemFree:          524288 kB
MemAvailable:    8912000 kB
Buffers:           65536 kB
Cached:          7168000 kB
SwapCached:        12288 kB
Active:          9216000 kB
Inactive:        5120000 kB
...
```

## 最も重要なフィールド {#the-fields-that-matter-most}

### MemTotal {#memtotal}

使用可能なRAMの合計。物理RAMから予約領域（カーネルコード、BIOSホール、ファームウェア）を引いたもの。

- **出所**: [`si_meminfo()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) が返す `totalram_pages()` で、ブート時に設定される。
- **よくある誤解**: MemTotalは搭載RAM量ではない。16GBのマシンでは通常、数百MB少なく表示される。

### MemFree {#memfree}

バディアロケータのフリーリストに載っているpage。何にもまったく使われていない。

- **出所**: `global_zone_page_state(NR_FREE_PAGES)`。
- **よくある誤解**: MemFreeが低いことはシステムがメモリ不足であることを**意味しない**。Linuxは空きRAMを積極的にページキャッシュに使う。健全なシステムでもMemFreeが非常に低いことはよくある。代わりにMemAvailableを見ること。

### MemAvailable {#memavailable}

スワップせずに新しいアプリケーションを起動するのに使えるメモリ量の推定値。「メモリが足りなくなりつつあるか？」を知るうえで唯一最も有用なフィールドである。

- **出所**: `mm/show_mem.c` の [`si_mem_available()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/show_mem.c)。
- **導入**: v3.14、Rik van Rielによる（[commit 34e431b0ae39](https://git.kernel.org/linus/34e431b0ae39)）。

**アルゴリズム**:

1. 空きpageから予約の合計を引いた値から始める（`NR_FREE_PAGES - totalreserve_pages`）
2. 回収可能なファイルキャッシュ（`Active(file) + Inactive(file)`）を加え、予備として `min(pagecache/2, wmark_low)` を引く
3. 回収可能なslabとその他の回収可能なカーネルメモリを加え、予備として `min(reclaimable/2, wmark_low)` を引く
4. 下限をゼロとする

```
MemAvailable ≈ (MemFree - totalreserve_pages)
             + pagecache - min(pagecache/2, wmark_low)
             + reclaimable_slab - min(reclaimable_slab/2, wmark_low)
```

`min()` はキャッシュの半分とlowウォーターマークの合計のうち*小さい方*を差し引く。これにより、推定値は過度に悲観的にならずに保守的なものになる。

**なぜこれが必要だったか**: かつてアプリケーションは利用可能なメモリを `MemFree + Cached` として推定していた。これは誤りである。`Cached` には解放できないメモリ（shmem/tmpfs）が含まれ、回収可能なslabが含まれないからだ。

## ページキャッシュのフィールド {#page-cache-fields}

### Buffers {#buffers}

ブロックデバイスのinodeに属するページキャッシュのpage。生のブロックデバイスへの読み書き（例: `dd`、パーティションへの直接I/O）である。

- **出所**: [`nr_blockdev_pages()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/bdev.c) がすべてのブロックデバイスinodeを走査し、キャッシュされたpageを合計する。
- **よくある誤解**: 「Buffers」はページキャッシュとは別のキャッシュではない。Linux 2.4以降、バッファキャッシュとページキャッシュは統合されている。Buffersはページキャッシュのうちブロックデバイスinodeに属するサブセットである。通常は小さい（数十MB）。

### Cached {#cached}

ファイルに裏付けられたページキャッシュのpageから、スワップキャッシュとバッファを除いたもの。

- **出所**: `NR_FILE_PAGES - total_swapcache_pages() - Buffers` として計算される。
- **よくある誤解**: Cachedには共有メモリ（shmem/tmpfs）が含まれる。これらのpageはディスクの裏付けがないため、スワップなしでは**回収できない**。`MemFree + Cached` が利用可能なメモリを過大に見積もるのはこのためである。

本当に回収可能なファイルキャッシュを得るには次のようにする。

```
Reclaimable cache ≈ Cached - Shmem
```

### SwapCached {#swapcached}

RAMとスワップの両方に同時に存在するpage。スワップアウトされたpageが読み戻されるとき、カーネルはスワップスロットを保持しておき、再び書き込まずに安価に再スワップできるようにする。

- **よくある誤解**: SwapCachedは余分なメモリ使用ではない。これらのpageは即座に回収できる（書き込み不要）。SwapCachedが大きいことは、システムが過去にメモリ逼迫を経験し、その後回復したことを意味する。

## LRUのフィールド {#lru-fields}

### Active / Inactive {#active-inactive}

LRU（Least Recently Used）リスト上のpage。activeなpageは最近アクセスされたもので、inactiveなpageはそうでないものである。

```
Active   = Active(anon)   + Active(file)
Inactive = Inactive(anon) + Inactive(file)
```

### Active(anon) / Inactive(anon) {#activeanon-inactiveanon}

LRU上の匿名page（ヒープ、スタック、プライベートmmap）。これらにはファイルの裏付けがなく、回収する唯一の方法はスワップである。

- `Active(anon)` の増加は、アプリケーションのメモリ消費が増えていることを示す。

### Active(file) / Inactive(file) {#activefile-inactivefile}

LRU上のファイルに裏付けられたpage。単に破棄するだけで回収できる（必要ならディスクから再読み込みする）。これは最も安価な回収経路である。

- MemAvailableが回収可能なキャッシュを推定するのに使うのはこれらである。
- GB単位のRAMを持つシステムで `Active(file) + Inactive(file)` がごく小さい場合、ページキャッシュが食いつぶされており、ディスク読み込みが遅くなる。

## スワップのフィールド {#swap-fields}

### SwapTotal / SwapFree {#swaptotal-swapfree}

設定されたスワップ領域の合計 / 現在未使用のスワップ領域。

- `SwapTotal - SwapFree` = 使用中のスワップ。
- `SwapTotal` が非ゼロで `SwapFree` がゼロなのは危険信号である。
- `SwapTotal = 0` はスワップが設定されていないことを意味し、匿名pageはまったく回収できない。

## Dirty / Writeback {#dirty-writeback}

### Dirty {#dirty}

ページキャッシュ内で変更されたが、まだディスクに書き込まれていないpage。

- **出所**: `NR_FILE_DIRTY`。
- Dirtyが高いことは、クラッシュ時のデータ損失リスクを意味する。Dirtyが高い状態が続く場合は、ストレージが遅いか、dirty制限（[`vm.dirty_ratio`](https://docs.kernel.org/admin-guide/sysctl/vm.html)）が緩すぎることを示唆する。

### Writeback {#writeback}

ライトバックスレッドによって現在ディスクに書き込まれているpage。

- DirtyとWritebackの両方が高い場合、ストレージサブシステムがボトルネックになっている。

## プロセスメモリのフィールド {#process-memory-fields}

### AnonPages {#anonpages}

ユーザーのページテーブルにマップされている匿名page（ファイルに裏付けられていないもの）。

- **出所**: `NR_ANON_MAPPED`。
- **注**: AnonPagesは `Active(anon) + Inactive(anon)` と等しく**ない**。LRUカウンタはLRU上のすべての匿名pageを数えるが、AnonPagesは少なくとも1つのプロセスのページテーブルに現在マップされているものだけを数える。

### Mapped {#mapped}

ユーザーのページテーブルにマップされている、ファイルに裏付けられたpage（共有ライブラリ、mmapされたファイル、実行コード）。

- **出所**: `NR_FILE_MAPPED`。
- `Cached` に対して `Mapped` が高いことは、プロセスがファイルマッピングを活発に使っていることを意味する。

### Shmem {#shmem}

共有メモリ（shmem、tmpfs、POSIX共有メモリ、devtmpfs）が使っているメモリの合計。

- **出所**: `NR_SHMEM`。
- ShmemはCachedに計上されるが、スワップなしでは**回収できない**。
- デスクトップで大きなShmemはGPUの共有メモリの場合がある。サーバーでは、tmpfsを多用するワークロード（Redisなど）がここに現れる。

## カーネルメモリのフィールド {#kernel-memory-fields}

### Slab / SReclaimable / SUnreclaim {#slab-sreclaimable-sunreclaim}

カーネルのslabアロケータのキャッシュが使っているメモリ（[v2.6.23](https://git.kernel.org/linus/a0acd820)以降はデフォルトでSLUB）。

```
Slab = SReclaimable + SUnreclaim
```

- **SReclaimable**: メモリ逼迫時にカーネルが解放できる（dentryキャッシュ、inodeキャッシュ）。
- **SUnreclaim**: 解放できない（使用中のカーネルオブジェクト）。
- `SUnreclaim` が時間とともに増加するのは、カーネルのメモリリークの可能性がある。`slabtop` または `/proc/slabinfo` で確認すること。

### KernelStack {#kernelstack}

カーネルスレッドのスタックに使われるメモリ。各スレッドは固定サイズのスタック（x86-64では通常16kB）を持つ。

- **出所**: `NR_KERNEL_STACK_KB`（kB単位で直接報告される）。
- スレッド数に比例する。プロセス数が安定しているのにKernelStackが増加する場合、スレッドリークを示している可能性がある。

### PageTables {#pagetables}

全プロセスについて仮想アドレスを物理アドレスにマップするページテーブルに使われるメモリ。

- **出所**: `NR_PAGETABLE`。
- マップされた仮想アドレス空間の合計とともに増加する。PageTablesが非常に高い場合、巨大で疎なmmap領域を持つプロセスがあることを示唆する。

## コミットアカウンティング {#commit-accounting}

### CommitLimit {#commitlimit}

厳格なオーバーコミット（`vm.overcommit_memory = 2`）のもとでコミットできるメモリの最大量。

- **計算式**: `(totalram - hugetlb) * overcommit_ratio/100 + swap`
- **出所**: `mm/util.c` の [`vm_commit_limit()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/util.c)。
- 厳格なオーバーコミット（モード2）でのみ意味を持つ。デフォルトのヒューリスティックモード（モード0）では無関係である。

### Committed_AS {#committed_as}

全プロセスが割り当てた（コミットした）仮想メモリの合計。

- **よくある誤解**: これは物理ではなく仮想である。プロセスは1GBをコミットしても10MBしか触らないことがある。デフォルトのヒューリスティックなオーバーコミット（モード0）では、Committed_ASはMemTotal + SwapTotalを超えうる。

## ヒュージページのフィールド {#huge-page-fields}

### HugePages_Total / HugePages_Free / Hugepagesize {#hugepagestotal-hugepagesfree-hugepagesize}

事前割り当てされたヒュージページ（hugetlbfs）、その空き数、およびデフォルトサイズ（x86-64では2MB）。

- **出所**: `mm/hugetlb.c` の [`hugetlb_report_meminfo()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/hugetlb.c)。
- **よくある誤解**: 事前割り当てされたヒュージページはMemFreeにもMemAvailableにも計上**されない**。これらはバディアロケータから完全に取り除かれる。4GBのヒュージページを割り当てると、他のすべての用途で利用可能なメモリが実質4GB減る。

## ダイレクトマップのフィールド {#direct-map-fields}

### DirectMap4k / DirectMap2M / DirectMap1G {#directmap4k-directmap2m-directmap1g}

各ページテーブルエントリのサイズを使ってカーネルにダイレクトマップされているメモリの量。

- **出所**: `arch/x86/mm/pat/set_memory.c` の `arch_report_meminfo()`。
- TLB効率のため、理想的にはメモリの大半がDirectMap1GかDirectMap2Mであるべきだ。DirectMap4kが大きい場合、カーネルのダイレクトマップが断片化している（多くはドライバからの `set_memory_*` 呼び出しによる）。

## すべて合計すると一致するか？ {#does-it-all-add-up}

一致しない。MemTotalは報告される全フィールドの合計と等しく**ない**。おおよその分解は次のとおりである。

```
MemTotal ≈ MemFree
         + Active + Inactive + Unevictable    (LRU-tracked pages)
         + Slab                                (SReclaimable + SUnreclaim)
         + KernelStack
         + PageTables
         + HugePages_Total * Hugepagesize      (if preallocated)
         + <unlabeled kernel allocations>
```

差の原因は次のとおりである。

- meminfoで個別に追跡されないカーネルの割り当て（percpuメモリ、vmallocの裏付けpage、カーネルのページテーブル）
- 競合状態 — `meminfo_proc_show()` は単一のロックなしでカウンタを読む
- MemFreeに含まれる、用途が制限されたCMAのpage
- 意図的に重複しているフィールド（ShmemはCachedの一部、MappedはCached + AnonPagesのサブセット）

## 診断パターン {#diagnostic-patterns}

### メモリが足りなくなりつつあるか？ {#am-i-running-out-of-memory}

**MemAvailable**を見ること。これがゼロに近づいたら、システムはメモリ逼迫状態にある。MemFreeを見てはならない。健全なシステムでは通常ゼロ近くにある。

副次的なシグナル:

- SwapFreeの減少（スワップが消費されている）
- Active(anon)の増加
- `/proc/pressure/memory` が高い `some` または `full` を示す（v4.20以降）

### ページキャッシュは機能しているか？ {#is-the-page-cache-working}

`Active(file) + Inactive(file)` をディスクI/Oレート（`iostat` から）と比較する。

- 大きなファイルキャッシュ + 少ないディスク読み込み = キャッシュは効果的
- 小さなファイルキャッシュ + 多いディスク読み込み = ワーキングセットが収まらず、キャッシュがスラッシングしている

### メモリリークを見つける方法 {#how-to-spot-a-memory-leak}

**ユーザー空間のリーク**:

- `AnonPages` と `Committed_AS` が時間とともに増加する
- プロセスごと: `/proc/<pid>/status` のVmRSSまたは `/proc/<pid>/smaps_rollup` を監視する

**カーネルのリーク**:

- `SUnreclaim` の増加 = slabリーク（`slabtop` を確認）
- `VmallocUsed` の増加 = vmallocリーク（注: このフィールドはv4.4でゼロ固定にされ（[commit a5ad88ce8c7f](https://git.kernel.org/linus/a5ad88ce8c7f)）、v5.3で復活した（[commit 97105f0ab7b8](https://git.kernel.org/linus/97105f0ab7b8)）。カーネル4.4〜5.2では代わりに `/proc/vmallocinfo` を確認すること）
- プロセス数が安定しているのに `KernelStack` が増加 = スレッドリーク
- 新しいプロセスがないのに `PageTables` が増加 = マッピングリーク

### BuffersとCachedの違いは？ {#what-is-buffers-vs-cached}

| フィールド | 中身 | 典型的なサイズ |
|-------|-----------|-------------|
| Buffers | ブロックデバイスのページキャッシュ（生のブロックI/O） | 数十MB |
| Cached | ファイルデータのページキャッシュ + shmem | GB規模 |

歴史的な「バッファキャッシュ対ページキャッシュ」の区別はLinux 2.4で廃止された。今日ではどちらもページキャッシュであり、Buffersはそのうちブロックデバイスのサブセットである。

## 試してみる {#try-it-yourself}

```bash
# メモリ状態の変化をリアルタイムで監視する
watch -n 1 'grep -E "MemTotal|MemFree|MemAvailable|Cached|SwapFree|Dirty|Active\(|Inactive\(" /proc/meminfo'

# 簡易ヘルスチェック: 利用可能量 対 合計
awk '/MemTotal/{t=$2} /MemAvailable/{a=$2} END{printf "%.0f%% available\n", a/t*100}' /proc/meminfo

# メモリがどこに使われているかを調べる
awk '/^(MemFree|Active|Inactive|Slab|KernelStack|PageTables|Shmem|AnonPages):/' /proc/meminfo

# 本当に回収可能なキャッシュ（shmem を除く）
awk '/^Cached/{c=$2} /^Shmem/{s=$2} END{print (c-s)" kB reclaimable file cache"}' /proc/meminfo
```

## 参考文献 {#further-reading}

- [`fs/proc/meminfo.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/proc/meminfo.c) — ソースコード
- [`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) — `si_meminfo()`
- [`mm/show_mem.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/show_mem.c) — `si_mem_available()`
- [MemAvailableのコミット](https://git.kernel.org/linus/34e431b0ae39) — Rik van Rielによる設計根拠
- [vm sysctlのドキュメント](https://docs.kernel.org/admin-guide/sysctl/vm.html) — チューニング可能なパラメータ
- [/proc/vmstat](proc-vmstat.md) — 補完的な診断ファイル
- [OOMログの読み方](reading-oom-log.md) — MemAvailableがゼロに達したとき
