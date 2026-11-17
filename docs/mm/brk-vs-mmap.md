# brk対mmap: メモリを得る2つの方法 {#brk-vs-mmap-two-ways-to-get-memory}

> カーネルは2つの仕組みを提供し、ユーザー空間のアロケータがその間で選択する

## カーネルの仕組みとユーザー空間のポリシー {#kernel-mechanisms-vs-userspace-policy}

カーネルはメモリを得るための2つのシステムコールを提供している。

1. **`brk()`** - 単一の連続したヒープ領域を拡張する
2. **`mmap()`** - 任意の場所に任意のメモリマッピングを作る

カーネルはどちらが使われるかを気にしない。両方を実装し、ユーザー空間に決めさせるだけである。このドキュメントでは、カーネルの仕組みとglibcがそれをどう使うかの両方を、明確に分けて扱う。

## 2つのアプローチ {#the-two-approaches}

### brk(): ヒープ {#brk-the-heap}

`brk()` は「プログラムブレーク」を移動する。歴史的には「データセグメント」の終端と呼ばれてきたが、現代のシステムでは上方向に伸びる単なるVMAの1つにすぎない（[mm/vma.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vma.c) の `do_brk_flags()` を参照）。

```
┌─────────────────┐ High addresses
│      Stack      │
│        ↓        │
│                 │
│                 │
│        ↑        │
│      Heap       │
├─────────────────┤ ← Program break (brk)
│   BSS (zeros)   │
│   Data (init)   │
│   Text (code)   │
└─────────────────┘ Low addresses
```

```c
// brk システムコールはプログラムブレークを設定する
#include <unistd.h>
#include <sys/syscall.h>

unsigned long current_brk = syscall(SYS_brk, 0);  // 現在のブレークを取得
syscall(SYS_brk, current_brk + 4096);             // 4KB 拡張する
// 新しいメモリは古いブレークと新しいブレークの間にある

// 注: sbrk() は libc のラッパーであり、brk が実際のシステムコール
```

**特徴:**

- 単一の連続した領域
- 上方向に伸びる
- 途中のメモリを解放できない
- mmapよりオーバーヘッドが小さい（ただしシステムコールであることに変わりはなく、VMAの更新、セキュリティチェック、rlimitチェックを伴う）

### mmap(): 匿名マッピング {#mmap-anonymous-mappings}

`mmap()` はアドレス空間の任意の場所に新しい領域を作る。

```
┌─────────────────┐
│      Stack      │
│                 │
│  ┌───────────┐  │
│  │  mmap #2  │  │  ← Independent regions
│  └───────────┘  │
│                 │
│  ┌───────────┐  │
│  │  mmap #1  │  │
│  └───────────┘  │
│                 │
│      Heap       │
├─────────────────┤
│   Text/Data     │
└─────────────────┘
```

```c
void *p = mmap(NULL, size,
               PROT_READ | PROT_WRITE,
               MAP_PRIVATE | MAP_ANONYMOUS,
               -1, 0);
```

カーネルの実装については [`do_mmap()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mmap.c) を参照。

**特徴:**

- アドレス空間の任意の位置
- 各マッピングは独立している
- 個別に解放（`munmap`）できる
- 割り当てごとのオーバーヘッドが大きい

## ユーザー空間のポリシー: glibcはどう選ぶか {#userspace-policy-how-glibc-chooses}

ここではユーザー空間が独自に判断を下す。カーネルは関与しない。

glibcのmallocは、割り当てパターンに基づいて調整される**動的しきい値**を使う。

```
Allocation size < threshold → brk (main arena heap)
Allocation size ≥ threshold → mmap
```

しきい値は128KBから始まり、大きな割り当てが解放されるのをアロケータが観測するにつれて、最大32MB（64ビットシステムの場合）まで増加しうる。この動的な挙動は、一貫して大きな割り当てを行うプログラムのmmap/munmapシステムコールのオーバーヘッドを減らすために追加された。

**注:** これらのしきい値は `glibc` 固有のものである。他のCライブラリ（`musl`、`bionic`）は異なる戦略を使い、一部のディストリビューションは `glibc` のデフォルトにパッチを当てている。

```c
// 簡略化した glibc のロジック
void *malloc(size_t size) {
    if (size < mmap_threshold) {
        return allocate_from_arena();  // メインアリーナは brk を使う
    } else {
        return mmap(...);              // 直接 mmap
    }
}
```

しきい値の確認または上書き:
```bash
# デフォルトは 128KB（131072 バイト）から始まり、動的に増加する
# 動的な挙動を無効にして固定しきい値を設定できる:
$ MALLOC_MMAP_THRESHOLD_=65536 ./myprogram

# あるいはコード内で:
mallopt(M_MMAP_THRESHOLD, 65536);
```

詳細は[glibc malloc tunables](https://www.gnu.org/software/libc/manual/html_node/Memory-Allocation-Tunables.html)を参照。

## なぜしきい値があるのか {#why-the-threshold}

### 小さな割り当て → brk（ヒープ） {#small-allocations-brk-heap}

**利点:**

- システムコールのオーバーヘッドが小さい（glibcはヒープを大きな塊で拡張し、1回のbrk呼び出しで多数のmallocに応える）
- 局所性が良い（割り当てが隣接する）
- フリーリストからの高速な割り当て

**欠点:**

- 時間とともにメモリが断片化する
- OSにメモリを返しにくい（ヒープの先頭に空き領域が必要）

### 大きな割り当て → mmap {#large-allocations-mmap}

**利点:**

- 解放時にメモリが即座にOSに返される（`munmap()`）
- 大きなブロックによるヒープの断片化がない
- 予測可能な後始末

**欠点:**

- 割り当てごとのシステムコールのオーバーヘッド
- TLBへの負荷（各マッピングにエントリが必要）
- アドレス空間が断片化する可能性

## 断片化の問題 {#the-fragmentation-problem}

brkのみで割り当てる次のシナリオを考える。

```
Initial heap:
[  A  ][  B  ][  C  ][  D  ]
        └─ free B ─┘

After free(B):
[  A  ][hole][  C  ][  D  ]
       └─ Can't return to OS!

Even if C and D are freed:
[  A  ][          free           ]
       └─ Still can't shrink brk past A
```

大きな割り当てにmmapを使う場合:
```
Heap: [  A  ][ B ][ C ]   (small allocations)

mmap regions:
[ Large D ]   ← munmap() returns directly to OS
[ Large E ]   ← Independent of heap
```

## 実際の例 {#real-world-example}

```c
#include <stdlib.h>
#include <stdio.h>

int main() {
    // 小さな割り当て - ヒープ（brk）に行く
    void *small = malloc(1024);
    printf("Small (1KB): %p\n", small);

    // 大きな割り当て - mmap を使う
    void *large = malloc(256 * 1024);
    printf("Large (256KB): %p\n", large);

    // アドレスの違いに注目
    // small: ヒープの近く（低いアドレス）
    // large: mmap 領域（高く、別の場所）

    free(large);  // glibc が munmap() を呼ぶ → OS に返る
    free(small);  // malloc のフリーリストに行く（システムコールなし）

    return 0;
}
```

システムコールをトレースする:
```bash
$ strace -e brk,mmap,munmap ./test
brk(NULL)                               = 0x55a8b8c49000
brk(0x55a8b8c6a000)                     = 0x55a8b8c6a000  # Heap setup
mmap(NULL, 266240, ..., MAP_ANONYMOUS)  = 0x7f8b12345000  # Large alloc
munmap(0x7f8b12345000, 266240)          = 0              # Large free
```

## glibcがメモリを返すとき {#when-glibc-returns-memory}

### ヒープ（brk）のメモリ: {#heap-brk-memory}

- 解放されたブロックは再利用のためmallocのフリーリストに入る
- ヒープ先頭の空き領域が `M_TRIM_THRESHOLD`（デフォルト128KB）を超えると、`glibc` は自動的にヒープをトリムする
- `malloc_trim()` でトリムを強制できるが、解放されるのはヒープ先頭のメモリだけである
- ヒープの途中にあるメモリはOSに返せない

### mmapのメモリ: {#mmap-memory}

- 解放時に `munmap()` → 即座にOSに返る
- RSSが目に見えて減る
- 割り当て順序にかかわらずきれいに解放される

```c
// ヒープのトリムを強制する（ヒープの先頭にのみ影響する）
#include <malloc.h>
malloc_trim(0);  // ヒープ先頭の空きメモリを OS に返す

// あるいは、特定の範囲に MADV_DONTNEED を使う
madvise(addr, len, MADV_DONTNEED);  // page を解放し、マッピングは保持する
```

## しきい値のチューニング {#tuning-the-threshold}

```c
#include <malloc.h>

// しきい値を上げる: mmap が減り、ヒープが大きくなる
mallopt(M_MMAP_THRESHOLD, 512 * 1024);  // 512KB

// しきい値を下げる: mmap が増え、ヒープが小さくなる
mallopt(M_MMAP_THRESHOLD, 64 * 1024);   // 64KB
```

あるいは環境変数で:
```bash
# 64KB 以上のすべての割り当てが mmap を使う
export MALLOC_MMAP_THRESHOLD_=65536
./myprogram
```

## トレードオフのまとめ {#trade-offs-summary}

| 要素 | brk（ヒープ） | mmap |
|--------|------------|------|
| システムコールのオーバーヘッド | 小さい | 割り当てごと |
| OSへのメモリ返却 | 難しい | 即時 |
| 断片化 | 蓄積しうる | 領域ごとのみ |
| 局所性 | 良い | 散らばる |
| TLBへの負荷 | 小さい | 大きい |
| 適した用途 | 多数の小さな割り当て | 大きな割り当て |

## 歴史的背景 {#historical-context}

この2つのシステムコールは異なる時代に生まれ、異なる目的のために設計された。

### brk(): 元祖（1979年） {#brk-the-original-1979}

`brk()` は[**Version 7 AT&T Unix（1979年）**](https://www.tuhs.org/cgi-bin/utree.pl?file=V7/usr/man/man2/brk.2)で登場した。最初期のUnixシステムコールの1つである。1970年代、アドレス空間はごく小さかった（合計64KBということも多かった）。「プログラムブレーク」は文字どおりプログラムのコードとデータを分けるものであり、この境界を上に動かすことでヒープを拡張した。

10年以上にわたり、`brk()` はアプリケーションがヒープメモリを得る**唯一の**方法だった。すべての `malloc()` 実装がこれを使っていた。

この呼び出しは今では歴史的遺物とみなされている。

- Single UNIX Specification v2で**LEGACY**とされた
- POSIX.1-2001から**削除**された
- Linuxは互換性のために残しているが、現代のアロケータはあまり使わなくなっている

### mmap(): 最初はファイルマッピング（1983〜1988年） {#mmap-file-mapping-first-1983-1988}

`mmap()` はまったく異なる目的のために設計された。**メモリマップドファイル**である。

年表:

- **1983年（4.2BSD）**: APIが設計・文書化されたが、実装はされなかった
- **1988年頃（SunOS 4.0）**: Sun Microsystemsによる最初の動作する実装
- **4.3BSD-Reno**: BSDの実装（Sunがコードの共有を拒んだため、Mach VMをベースにした）
- **4.4BSD**: mmapを含む公式なBSDリリース

元々の用途は、ファイルの内容を直接メモリにマップすることだった。`read()` を呼ぶ代わりにメモリアドレスにアクセスしてファイルを読むのである。これはヒープ割り当てとは何の関係もなかった。

### MAP_ANONYMOUS: ゲームチェンジャー {#map_anonymous-the-game-changer}

`mmap()` をヒープ割り当てに使えるようにした機能が **`MAP_ANONYMOUS`**（または `MAP_ANON`）である。

- **どのファイルにも裏付けられていない**メモリマッピングを可能にする
- これが存在する前は、`mmap("/dev/zero", ...)` が回避策だった
- **`MAP_PRIVATE` と組み合わせた `MAP_ANONYMOUS`**: 初期のLinuxから利用可能（概念は[BSDの `MAP_ANON`](https://man.freebsd.org/cgi/man.cgi?query=mmap&sektion=2)に由来）

#### 何年もかかった「怪物」 {#the-monstrosity-that-took-years}

**`MAP_SHARED` と組み合わせた `MAP_ANONYMOUS`** には論争の歴史がある。

**1996年7月**: A.N. KuznetsovがLinux 2.0にこの機能がないことを[発見した](https://lkml.iu.edu/hypermail/linux/kernel/9607.3/0055.html)。
> *"Unpleasant discovery, 2.0 has no shared anonymous mmap ??? I do not believe to my eyes..."*（訳: 不愉快な発見だ。2.0には共有匿名mmapがない？？？　自分の目が信じられない…）

**Linus Torvaldsは冷ややかに[返信した](https://lkml.iu.edu/hypermail/linux/kernel/9607.3/0107.html)**。
> *"a monstrosity (even the name is a oxymoron [sic]), and it's hard to implement to boot"*（訳: 怪物だ（名前すら撞着語法だ [原文ママ]）。おまけに実装も難しい）
> *"definitely 2.1, if even that"*（訳: やるとしても間違いなく2.1だ）

彼は代わりにSysV共有メモリを推奨し、「共有匿名mmapを完全に避けたい」と考えていた。

**2000〜2001年**: **Christoph Rohland**（SAP AG）が2.3/2.4の開発サイクルで、**[`shmem`/`tmpfs`](https://docs.kernel.org/filesystems/tmpfs.html)** ファイルシステムを作ることでついに解決策を実装した（[`mm/shmem.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/shmem.c) を参照）。巧妙な秘訣は、Linuxが「RAMベースのファイルシステムを使って匿名pageに人工的なファイルの裏付けを作る」ことである。共有匿名マッピングは本当の意味で匿名なのではなく、目に見えない `tmpfs` ファイルに裏付けられているのだ。

*注: Linux 2.4はgit以前のものである（カーネルがgitに移行したのは2005年4月、2.6.12のとき）。git以前の歴史にはコミットIDが存在しない。*

`MAP_ANONYMOUS` が存在するようになると、アロケータは大きな割り当てに `mmap()` を使えるようになった。その重要な利点は、ヒープを先頭からしか縮められない `brk()` と異なり、`munmap()` がメモリを即座にOSに返すことである。

### なぜ両方が生き残っているのか {#why-both-survive}

今日2つの仕組みがあるのは、それぞれが異なる目的のために進化してきたからである。

| 時代 | 仕組み | 目的 |
|-----|-----------|---------|
| 1979 | `brk()` | 単一のヒープを伸ばす（唯一の選択肢） |
| 1983-88 | `mmap()` | ファイルをメモリにマップする |
| 1990年代以降 | `mmap()` + `MAP_ANONYMOUS` | ヒープ用の `brk()` の代替 |

`glibc` の `malloc` のような現代のアロケータは**両方**を使う。メインアリーナには `brk()`（小さな割り当ては局所性の恩恵を受ける）、大きな割り当てには `mmap()`（OSへのきれいな返却）である。これはエレガントな設計ではなく、歴史的な偶然が実用的なエンジニアリングになったものである。

## 試してみる {#try-it-yourself}

```bash
# brk と mmap の動作を観察する
strace -e brk,mmap,munmap ./myprogram

# ヒープのサイズを見る
cat /proc/<pid>/maps | grep heap

# すべての匿名マッピングを見る
cat /proc/<pid>/maps | grep anon

# 大きな割り当ての解放前後で RSS を比較する
watch -n 0.5 "grep -E 'VmRSS|VmData' /proc/<pid>/status"
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [mm/mmap.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mmap.c) — `do_mmap()` と匿名マッピングの実装
- [mm/vma.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vma.c) — `do_brk_flags()`: ヒープ拡張のカーネル側
- [mm/shmem.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/shmem.c) — `MAP_SHARED | MAP_ANONYMOUS` マッピングのtmpfsによる裏付け

### manページ {#man-pages}

- [`brk(2)`](https://man7.org/linux/man-pages/man2/brk.2.html) — システムコールのインターフェース。POSIXでのレガシーという位置付けについての記述がある
- [`mmap(2)`](https://man7.org/linux/man-pages/man2/mmap.2.html) — `MAP_ANONYMOUS`、`MAP_POPULATE`、`MAP_HUGETLB` を含む全フラグのリファレンス
- [`mallopt(3)`](https://man7.org/linux/man-pages/man3/mallopt.3.html) — glibcのチューニング: `M_MMAP_THRESHOLD`、`M_TRIM_THRESHOLD`

### 関連ページ {#related-pages}

- [mmap.md](mmap.md) — VMA構造体、`mm_struct`、アドレス空間のレイアウト全体
- [overcommit.md](overcommit.md) — 物理RAMが足りなくても `mmap()` が成功しうる理由

### LWNの記事 {#lwn-articles}

- [LWN: Fun with `mmap()`](https://lwn.net/Articles/657338/) — 匿名マッピングとファイルに裏付けられたマッピングの詳解

### 外部資料 {#external}

- [glibc malloc internals](https://sourceware.org/glibc/wiki/MallocInternals) — アリーナの設計、スレッドごとのキャッシュ、動的なmmapしきい値のアルゴリズム
