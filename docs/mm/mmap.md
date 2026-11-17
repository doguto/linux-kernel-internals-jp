# プロセスのアドレス空間 {#process-address-space}

> Linuxがプロセスの仮想メモリをどう管理するか

## プロセスのアドレス空間とは何か? {#what-is-the-process-address-space}

各プロセスは自身の仮想アドレス空間を持つ。これはメモリのプライベートな見え方であり、カーネルがそれを物理メモリに対応付ける。この分離により、プロセスは（明示的に共有しない限り）互いのメモリにアクセスできない。

```
Process A's View              Process B's View
┌─────────────────┐          ┌─────────────────┐
│ Stack      ↓    │          │ Stack      ↓    │
├─────────────────┤          ├─────────────────┤
│                 │          │                 │
│   (unmapped)    │          │   (unmapped)    │
│                 │          │                 │
├─────────────────┤          ├─────────────────┤
│ Heap       ↑    │          │ Heap       ↑    │
├─────────────────┤          ├─────────────────┤
│ BSS (zero)      │          │ BSS (zero)      │
├─────────────────┤          ├─────────────────┤
│ Data            │          │ Data            │
├─────────────────┤          ├─────────────────┤
│ Text (code)     │          │ Text (code)     │
└─────────────────┘          └─────────────────┘
   0x0                          0x0

Same virtual addresses, different physical memory
```

## 仮想メモリ領域（VMA） {#virtual-memory-areas-vmas}

アドレス空間はVMAと呼ばれる領域に分割される。各VMAは、一貫した属性を持つ連続した範囲を記述する。

### VMAの構造 {#vma-structure}

```c
struct vm_area_struct {
    unsigned long vm_start;      /* 開始アドレス */
    unsigned long vm_end;        /* 終了アドレス（この値は含まない） */
    pgprot_t vm_page_prot;       /* アクセス権限 */
    unsigned long vm_flags;      /* フラグ（VM_READ、VM_WRITEなど） */
    struct file *vm_file;        /* 裏付けとなるファイル（またはNULL） */
    unsigned long vm_pgoff;      /* ファイル内のオフセット */
    struct mm_struct *vm_mm;     /* 所有するmm_struct */
    /* ... */
};
```

### VMAの種類 {#vma-types}

| 種類 | vm_file | 説明 |
|------|---------|-------------|
| 無名 | NULL | ヒープ、スタック、mmap(MAP_ANONYMOUS) |
| ファイルバック | 設定あり | mmapされたファイル、実行ファイル、ライブラリ |
| 共有 | 設定あり + VM_SHARED | 共有メモリ領域 |

### VMAを見る {#viewing-vmas}

```bash
# プロセスのメモリマップを見る
cat /proc/<pid>/maps

# 出力例:
# address          perms offset   dev   inode   pathname
# 00400000-00452000 r-xp 00000000 08:01 1234    /bin/bash
# 00651000-00652000 r--p 00051000 08:01 1234    /bin/bash
# 00652000-0065b000 rw-p 00052000 08:01 1234    /bin/bash
# 0065b000-00660000 rw-p 00000000 00:00 0       [heap]
# 7f8a0000-7f8a2000 rw-p 00000000 00:00 0
# 7ffff000-80000000 rw-p 00000000 00:00 0       [stack]

# サイズ付きの詳細表示
cat /proc/<pid>/smaps
```

## mmap()システムコール {#the-mmap-system-call}

`mmap()`は新しいVMAを作成する。これはプロセスが仮想メモリを要求する手段である。

### 基本的な使い方 {#basic-usage}

```c
#include <sys/mman.h>

/* 無名マッピング（大きな割り当てに対するmallocのようなもの） */
void *ptr = mmap(NULL, size, PROT_READ | PROT_WRITE,
                 MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

/* ファイルマッピング */
int fd = open("file.dat", O_RDONLY);
void *ptr = mmap(NULL, size, PROT_READ,
                 MAP_PRIVATE, fd, 0);

/* 終わったらアンマップする */
munmap(ptr, size);
```

### mmapのフラグ {#mmap-flags}

| フラグ | 説明 |
|------|-------------|
| `MAP_PRIVATE` | コピーオンライト。変更はファイルに書き込まれない |
| `MAP_SHARED` | 変更は他のプロセスから見え、ファイルに書き込まれる |
| `MAP_ANONYMOUS` | ファイルに裏付けられず、ゼロで初期化される |
| `MAP_FIXED` | 正確なアドレスを使う（危険） |
| `MAP_POPULATE` | pageを事前にフォールトインする（後のフォールトを避ける） |
| `MAP_HUGETLB` | ヒュージページを使う |
| `MAP_LOCKED` | RAMにロックする（mlockと同様） |

### 保護フラグ {#protection-flags}

| フラグ | 説明 |
|------|-------------|
| `PROT_READ` | pageを読み取れる |
| `PROT_WRITE` | pageに書き込める |
| `PROT_EXEC` | pageを実行できる |
| `PROT_NONE` | pageにアクセスできない |

## アドレス空間のレイアウト {#address-space-layout}

### 従来のレイアウト（32ビット） {#traditional-layout-32-bit}

```
┌──────────────────┐ 0xFFFFFFFF (4GB)
│  Kernel Space    │
├──────────────────┤ 0xC0000000 (3GB) - TASK_SIZE
│  Stack     ↓     │
├──────────────────┤
│  mmap region ↓   │
├──────────────────┤
│                  │
├──────────────────┤
│  Heap       ↑    │ brk
├──────────────────┤
│  BSS             │
├──────────────────┤
│  Data            │
├──────────────────┤
│  Text            │
└──────────────────┘ 0x08048000
```

### 現代のレイアウト（64ビット、ASLRあり） {#modern-layout-64-bit-with-aslr}

```
┌──────────────────┐ 0xFFFFFFFFFFFFFFFF
│  Kernel Space    │
├──────────────────┤ 0x7FFFFFFFFFFF (128TB user limit)
│  Stack     ↓     │ (randomized)
├──────────────────┤
│                  │
├──────────────────┤
│  mmap region     │ (randomized)
│  (libs, anon)    │
├──────────────────┤
│                  │
├──────────────────┤
│  Heap       ↑    │ (randomized)
├──────────────────┤
│  BSS/Data/Text   │ (randomized)
└──────────────────┘ 0x0
```

### ASLR（Address Space Layout Randomization） {#aslr-address-space-layout-randomization}

ASLRはメモリレイアウトをランダム化してエクスプロイトを難しくする。

```bash
# ASLRの設定を確認する
cat /proc/sys/kernel/randomize_va_space
# 0 = 無効
# 1 = スタック/mmapをランダム化
# 2 = + ヒープもランダム化（完全なASLR）
```

詳細は[kernel sysctl docs](https://docs.kernel.org/admin-guide/sysctl/kernel.html)と[x86_64 memory map](https://docs.kernel.org/arch/x86/x86_64/mm.html)を参照してほしい。

## メモリ管理の構造体 {#memory-management-structures}

### mm_struct {#mm_struct}

各アドレス空間は1つの`mm_struct`で記述される（プロセス内のスレッドは`CLONE_VM`によって同じmm_structを共有する）。

```c
struct mm_struct {
    struct maple_tree mm_mt;     /* VMAツリー（v6.1でrb_rootを置き換えた） */
    unsigned long task_size;     /* ユーザーアドレス空間のサイズ */
    pgd_t *pgd;                  /* Page Global Directory */
    atomic_t mm_users;           /* このmmを共有するプロセス */
    atomic_t mm_count;           /* 構造体への参照 */
    unsigned long start_code, end_code;   /* テキストセグメント */
    unsigned long start_data, end_data;   /* データセグメント */
    unsigned long start_brk, brk;         /* ヒープ */
    unsigned long start_stack;            /* スタックの開始位置 */
    unsigned long mmap_base;              /* mmap領域のベース */
    /* ... */
};
```

### VMAの編成 {#vma-organization}

VMAはv6.1まで赤黒木に格納されていたが、性能向上のためにmaple treeに切り替えられた。

```
Pre-v6.1: Red-Black Tree     v6.1+: Maple Tree
       [VMA]                    [Maple Node]
      /     \                   /    |    \
   [VMA]   [VMA]             [VMA] [VMA] [VMA]
   /   \                      ...
[VMA] [VMA]
```

**コミット**: [d4af56c5c7c6](https://git.kernel.org/linus/d4af56c5c7c6)（"mm: start tracking VMAs with maple tree"） | [LKML](https://lore.kernel.org/linux-mm/20220822150128.1562046-1-Liam.Howlett@oracle.com/)

## デマンドページング {#demand-paging}

pageはアクセスされるまで割り当てられない。これはデマンドページングまたは遅延割り当てと呼ばれる。

```
mmap() called:
┌───────────────────────┐
│ VMA created           │
│ No physical pages yet │
└───────────────────────┘

First access:
┌───────────────────────┐
│ Page fault            │
│     ↓                 │
│ Allocate page         │
│     ↓                 │
│ Update page table     │
│     ↓                 │
│ Resume execution      │
└───────────────────────┘
```

### フォールトの種類 {#fault-types}

| フォールト | 原因 | 解決方法 |
|-------|-------|------------|
| マイナー | pageはメモリ上にあるが、PTEの更新が必要 | PTEを更新する（COWの破れ、アクセス/ダーティビットの更新を含む） |
| メジャー | pageがメモリ上にない | ファイル/スワップから読み込む |
| 不正 | 不正なアクセス（SIGSEGV） | プロセスをkillする |

## コピーオンライト（COW） {#copy-on-write-cow}

`fork()`が子プロセスを作成すると、pageは書き込まれるまで共有される。

```
After fork():
Parent          Child
  │               │
  └───────┬───────┘
          │
          ▼
    [Shared Page]  (read-only)

After child writes:
Parent          Child
  │               │
  ▼               ▼
[Original]    [Copy]
```

これにより`fork()`は高速になる。ページテーブルは複製される（同じ物理pageを指し、それらは読み取り専用とマークされる）が、実際のデータpageはコピーされない。どちらかのプロセスが書き込んだときにだけコピーが起こる。

エッジケース、性能への影響、COWがいつ破れるかについては[コピーオンライトの解説](cow.md)を参照してほしい。

## よく使う操作 {#common-operations}

### brk/sbrk（ヒープ） {#brksbrk-heap}

```c
/* ヒープを拡張する */
void *old_brk = sbrk(0);
brk(old_brk + 4096);  /* ヒープに4KB追加する */

/* mallocは通常、大きな割り当てにはmmapを使う */
```

### mprotect（権限の変更） {#mprotect-change-permissions}

```c
/* 領域を読み取り専用にする */
mprotect(ptr, size, PROT_READ);

/* 領域を実行可能にする（JIT） */
mprotect(ptr, size, PROT_READ | PROT_EXEC);
```

### mlock（RAMに固定する） {#mlock-pin-in-ram}

```c
/* pageがスワップされないようにする */
mlock(ptr, size);

/* ロックを解除する */
munlock(ptr, size);

/* 現在および将来のすべてのpageをロックする */
mlockall(MCL_CURRENT | MCL_FUTURE);
```

### mremap（マッピングのサイズ変更） {#mremap-resize-mapping}

```c
/* マッピングを拡大または縮小する */
void *new_ptr = mremap(ptr, old_size, new_size, MREMAP_MAYMOVE);
```

## 進化 {#evolution}

### 当初の設計（1991年） {#original-design-1991}

VMAの単純な線形リスト。小さなアドレス空間ではこれで機能した。

### 赤黒木（v2.4.10、2001年） {#red-black-tree-v2410-2001}

[rb-tree](https://docs.kernel.org/core-api/rbtree.html)を使うことで、VMAの検索がO(n)からO(log n)に変わった（Andrea Arcangeliによる実装で、gitの履歴より前のもの）。

### Maple Tree（v6.1、2022年） {#maple-tree-v61-2022}

**コミット**: [d4af56c5c7c6](https://git.kernel.org/linus/d4af56c5c7c6)（"mm: start tracking VMAs with maple tree"） | [LKML](https://lore.kernel.org/linux-mm/20220906194824.2110408-9-Liam.Howlett@oracle.com/)

**作者**: Liam R. Howlett

キャッシュの挙動を改善し、RCUで安全な検索を可能にするため、rb-treeをmaple treeに置き換えた。

### VMA単位ロック（v6.4、2023年） {#per-vma-locks-v64-2023}

**コミット**: [5e31275cc997](https://git.kernel.org/linus/5e31275cc997)（"mm: add per-VMA lock and helper functions to control it"） | [LKML](https://lore.kernel.org/all/20230109205336.3665937-1-surenb@google.com/)

**作者**: Suren Baghdasaryan

mm全体に対するmmap_lockの代わりにVMAごとの細粒度ロックを導入し、ページフォールト処理のスケーラビリティを改善した。

## デバッグ {#debugging}

### /procインターフェース {#proc-interface}

```bash
# メモリマップ
cat /proc/<pid>/maps

# RSS、PSSなどを含む詳細
cat /proc/<pid>/smaps

# 要約
cat /proc/<pid>/smaps_rollup

# メモリ統計
cat /proc/<pid>/status | grep -i vm
# VmPeak: 仮想メモリのピーク値
# VmSize: 現在の仮想メモリ
# VmRSS:  常駐セットサイズ
# VmData: データセグメント
# VmStk:  スタック
```

### トレーシング {#tracing}

```bash
# mmap呼び出しをトレースする
strace -e mmap,munmap,mprotect ./program

# ページフォールトをトレースする
echo 1 > /sys/kernel/debug/tracing/events/exceptions/page_fault_user/enable
```

## よくある問題 {#common-issues}

### 仮想メモリの枯渇 {#virtual-memory-exhaustion}

プロセスが`TASK_SIZE`の上限、または[`vm.max_map_count`](https://docs.kernel.org/admin-guide/sysctl/vm.html)に達する。

```bash
# プロセスあたりのVMA数の上限を表示/設定する
cat /proc/sys/vm/max_map_count
echo 65530 > /proc/sys/vm/max_map_count
```

### メモリの断片化 {#memory-fragmentation}

多数の小さなマッピングが大きな割り当てを妨げる。

**解決策**: より大きなマッピングを使い、mmap/munmapの頻繁な繰り返しを減らす。

### mmap_lockの競合 {#mmap_lock-contention}

複数のスレッドがmm->mmap_lockを奪い合う。

**解決策**: VMA単位ロックのためにv6.4以降にアップグレードするか、mmap操作を減らす。

## 参照 {#references}

### 主要なコード {#key-code}

| ファイル | 説明 |
|------|-------------|
| [`mm/mmap.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mmap.c) | mmapの実装 |
| [`mm/memory.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memory.c) | ページフォールト処理 |
| [`include/linux/mm_types.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mm_types.h) | mm_struct、vm_area_struct |

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/mm/process_addrs.rst`](https://docs.kernel.org/mm/process_addrs.html)

### 関連 {#related}

- [page-tables](page-tables.md): VMAがページテーブルによってどう裏付けられるか
- [reclaim](reclaim.md): 無名pageとファイルバックのpageの扱い
- [thp](thp.md): VMA内のヒュージページ

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [mm/mmap.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mmap.c): `do_mmap()`、`mmap_region()`、VMAの結合と分割
- [mm/vma.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vma.c): VMAのライフサイクルのヘルパー。`do_brk_flags()`、`vma_merge()`、`vma_expand()`
- [include/linux/mm_types.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mm_types.h): `struct mm_struct`と`struct vm_area_struct`の定義

### カーネルドキュメント {#kernel-documentation_1}

- [`Documentation/admin-guide/mm/concepts.rst`](https://docs.kernel.org/admin-guide/mm/concepts.html): VMA、デマンドページング、プロセスのアドレス空間についての公式の概説
- [`Documentation/arch/x86/x86_64/mm.rst`](https://docs.kernel.org/arch/x86/x86_64/mm.html): x86_64の仮想メモリマップの権威ある資料

### 関連ページ {#related-pages}

- [page-fault.md](page-fault.md): VMAが見つかった後に何が起こるか。ページフォールト処理とPTEの設定
- [per-vma-locks.md](per-vma-locks.md): Linux 6.4がVMA単位ロックでmmap_lockの競合をどう減らしたか
- [brk-vs-mmap.md](brk-vs-mmap.md): アロケータが`brk()`と`mmap()`のどちらをいつ、なぜ選ぶのか

### LWNの記事 {#lwn-articles}

- [LWN: Virtual Memory I — address spaces](https://lwn.net/Articles/829812/): Linuxの仮想メモリについての入門シリーズ
- [LWN: The maple tree](https://lwn.net/Articles/894061/): v6.1でVMAの赤黒木を置き換えた、RCUで安全なBツリー
