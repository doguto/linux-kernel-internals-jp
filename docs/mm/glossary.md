# 用語集 {#glossary}

Linuxのメモリ管理で出会う用語を、初学者向けに説明する。

---

## A {#a}

### アドレス空間（Address Space） {#address-space}
プロセスが使用できるメモリアドレスの範囲。仮想メモリシステムでは、各プロセスが独自のアドレス空間を持ち、それがページテーブルを介して物理メモリにマップされる。

### 割り当て（Allocation） {#allocation}
システムにメモリを要求すること。カーネルは用途に応じてさまざまなアロケータ（kmalloc、vmalloc）を提供している。

---

## B {#b}

### バディアロケータ（Buddy Allocator） {#buddy-allocator}
Linuxのページアロケータ。物理pageを2のべき乗サイズの「バディ」にまとめる。pageを解放すると、そのバディも空いていれば両者は結合する。

**例**: 隣接する2つの4ページブロックは、1つの8ページブロックに結合できる。

---

## C {#c}

### キャッシュ（CPU） {#cache-cpu}
CPUとRAMの間にある高速なメモリ。最近アクセスされたデータを保持する。L1が最も高速かつ小さく、L3が最も低速かつ大きい。

### キャッシュ（slab） {#cache-slab}
同じ型のオブジェクトをあらかじめ割り当てておくプール。毎回新しいメモリを割り当てるより高速である。

### 連続（Contiguous） {#contiguous}
アドレス空間上で隣接していること。「物理的に連続」は実際のRAM上で隣接していることを意味する。「仮想的に連続」は仮想アドレス上で隣接していることを意味する（物理RAM上では散らばっているかもしれない）。

---

## D {#d}

### DMA（Direct Memory Access） {#dma-direct-memory-access}
ハードウェアがCPUを介さずに直接メモリにアクセスすること。DMAバッファには仮想アドレスではなく物理アドレスが必要である。

---

## F {#f}

### folio {#folio}
1つ以上の連続したpageを表す新しい抽象化（[v5.16+](https://git.kernel.org/linus/7b230db3b8d3)）。複合ページに対する `struct page` の曖昧な使い方を置き換える。folioは常に複合ページの先頭（head）か、単一のpageである。

```c
struct folio *folio = page_folio(page);
```

**存在理由**: `struct page` は多義的だった。単一のpageである場合も、複合ページのtail pageである場合も、head pageである場合もあった。folioはAPIをより明確にし、将来の最適化を可能にする。

### 断片化（Fragmentation） {#fragmentation}
空きメモリは存在するのに、小さく非連続な断片に分かれているため使えない状態。

**外部断片化**: 空きpageがメモリ全体に散らばっている状態。
**内部断片化**: 必要以上に割り当てられた状態（`5KB` を要求して `8KB` を得た）。

---

## G {#g}

### GFPフラグ（Get Free Pages） {#gfp-flags-get-free-pages}
割り当て関数に渡し、その振る舞いを指定するフラグ:

- `GFP_KERNEL`: 通常の割り当て。スリープできる
- `GFP_ATOMIC`: スリープできない（割り込み内で使う）
- `__GFP_ZERO`: 返す前にメモリをゼロクリアする

---

## H {#h}

### ヒュージページ（Huge Pages） {#huge-pages}
デフォルトの `4KB` より大きいpage（典型的には `2MB` または `1GB`）。大きな割り当てに対するTLBへの負荷を減らすが、内部断片化を増やす。

---

## I {#i}

### IPI（Inter-Processor Interrupt） {#ipi-inter-processor-interrupt}
あるCPUから別のCPUへ送られるシグナル。TLBフラッシュに使われる。あるCPUがページテーブルを変更したとき、他のCPUにTLBエントリを無効化するよう伝えなければならない。

**なぜ重要か**: IPIは高コストである。IPIを減らすために操作をバッチ化するのは一般的な最適化である。

---

## K {#k}

### kmalloc {#kmalloc}
カーネルの汎用メモリアロケータ。物理的に連続したメモリを返す。小さな割り当てに対して高速である。

```c
void *ptr = kmalloc(size, GFP_KERNEL);
kfree(ptr);
```

### kvmalloc {#kvmalloc}
まずkmallocを試み、それが（断片化により）失敗するとvmallocにフォールバックする。両者の良いところ取りである。

---

## L {#l}

### 遅延（Lazy / Deferred）操作 {#lazy-deferred-operations}
必要になるまで作業を先送りすること。例: lazy TLBフラッシュは、アドレスが再利用されるまでフラッシュを遅らせ、複数の操作をバッチ化する。

---

## M {#m}

### memblock {#memblock}
ブート初期のメモリアロケータ。バディアロケータが初期化される前、カーネルはmemblockを使ってメモリを追跡・割り当てる。単純（予約済み領域と空き領域を追跡するだけ）だが、ブートには十分である。

```bash
# memblock の領域を表示する（debugfs が有効な場合）
cat /sys/kernel/debug/memblock/memory
```

### memcg（Memory Cgroup） {#memcg-memory-cgroup}
メモリコントロールグループ。コンテナやプロセスのグループごとにメモリ使用量を制限・追跡できる。コンテナのメモリ制限の基盤である。

### MMU（Memory Management Unit） {#mmu-memory-management-unit}
ページテーブルを使って仮想アドレスを物理アドレスに変換するハードウェア。すべてのメモリアクセスはMMUを通る。

---

## N {#n}

### NUMA（Non-Uniform Memory Access） {#numa-non-uniform-memory-access}
どのCPUがどのメモリバンクにアクセスするかによってメモリアクセス時間が変わるアーキテクチャ。各CPUには「ローカル」メモリ（高速）と「リモート」メモリ（低速）がある。

**なぜ重要か**: アロケータはローカルノードのメモリを割り当てようとする。

---

## O {#o}

### OOM（Out of Memory） {#oom-out-of-memory}
システムのメモリが尽きた状態。OOM killerは終了させるプロセスを選び、メモリを解放する。

### オーダー（Page Order） {#order-page-order}
割り当てサイズを示す2のべき指数。オーダー0 = 1ページ（`4KB`）、オーダー1 = 2ページ（`8KB`）、オーダー2 = 4ページ（`16KB`）、以下同様。

---

## P {#p}

### page {#page}
メモリ管理の基本単位。x86では通常 `4KB`。MMUがマップできる最小単位である。

### ページテーブル（Page Table） {#page-table}
仮想アドレスを物理アドレスにマップするデータ構造。空間を節約するため多段構成になっている（x86-64では `PGD` -> `PUD` -> `PMD` -> `PTE`）。

### PFN（Page Frame Number） {#pfn-page-frame-number}
物理ページフレームのインデックス。pageが物理アドレス `0x1234000` にあり、ページサイズが `4KB` なら、PFN = `0x1234000 / 4096 = 0x1234` である。

```c
/* PFNと物理アドレスの相互変換 */
pfn = phys_addr >> PAGE_SHIFT;
phys_addr = pfn << PAGE_SHIFT;

/* PFNとstruct pageの相互変換 */
struct page *page = pfn_to_page(pfn);
pfn = page_to_pfn(page);
```

### 物理アドレス（Physical Address） {#physical-address}
RAMハードウェア上の実際の位置。メモリコントローラから見えるアドレスである。

---

## R {#r}

### RBTree（赤黒木） {#rbtree-red-black-tree}
自己平衡二分探索木。vmallocで `O(log n)` のアドレス探索に使われる。大きなデータセットでは連結リストより優れている。

---

## S {#s}

### slabアロケータ（Slab Allocator） {#slab-allocator}
pageを固定サイズのオブジェクトキャッシュに切り分ける。小さなオブジェクトの内部断片化を避ける。現在のデフォルト実装はSLUBである。

### SLUB {#slub}
現在のデフォルトslabアロケータ（v2.6.22以降）。元のSLABより単純で、オーバーヘッドが小さく、NUMAのサポートが優れている。

### struct page {#struct-page}
物理ページフレームに対するカーネルの記述子。システム内のすべての物理pageに対して1つの `struct page` が存在する（`mem_map` 配列またはvmemmapに格納される）。

```c
struct page {
    unsigned long flags;        /* ページ状態フラグ（PG_locked, PG_dirty など） */
    atomic_t _refcount;         /* 参照カウント */
    atomic_t _mapcount;         /* ページテーブルからのマッピング数 */
    struct address_space *mapping;  /* 所有者（file, swap, anon） */
    pgoff_t index;              /* マッピング内のオフセット */
    struct list_head lru;       /* 回収用のLRUリスト */
    /* ... unionを介したさらに多くのフィールド */
};
```

**注**: 新しいコードでは、複数ページの操作に `struct folio` を使うほうが望ましい。

---

## T {#t}

### TLB（Translation Lookaside Buffer） {#tlb-translation-lookaside-buffer}
ページテーブルエントリのキャッシュ。TLBがなければ、メモリアクセスのたびに複数回のページテーブル参照が必要になる。TLBによって仮想メモリは実用的なものになっている。

**TLBミス**: アドレスがTLBになく、ページテーブルを走査しなければならない状態。高コストである。
**TLBフラッシュ**: TLBエントリを無効化すること。ページテーブルが変更されたときに必要になる。

---

## V {#v}

### 仮想アドレス（Virtual Address） {#virtual-address}
プロセスが使うアドレス。MMUとページテーブルによって物理アドレスに変換される。各プロセスは、同じ仮想アドレスを異なる物理メモリにマップできる。

### vmalloc {#vmalloc}
物理的には散らばっている可能性のあるpageから、仮想的に連続したメモリを割り当てる。kmallocより低速だが、大きな割り当てに応えられる。

```c
void *ptr = vmalloc(size);
vfree(ptr);
```

### vmap_area {#vmap_area}
vmallocのアドレス空間を追跡するデータ構造。高速に探索できるようRBTreeに格納される。

### vm_struct {#vm_struct}
vmalloc領域を記述する: アドレス、サイズ、裏付けとなるpage、フラグ。

### vrealloc {#vrealloc}
vmallocによる割り当てのサイズを変更する。その場で縮小する（pageを解放する）ことも、拡大する（新たな割り当てが必要になることがある）こともできる。

---

## Z {#z}

### ゾーン（Zone） {#zone}
ハードウェアの制約による物理メモリの区分。ゾーンはアーキテクチャによって異なる。

**x86-64:**

- **`ZONE_DMA`**: 先頭16MB（レガシーな16ビットDMA）
- **`ZONE_DMA32`**: 16MB - 4GB（32ビットDMAデバイス）
- **`ZONE_NORMAL`**: 4GB以上（通常の割り当て）

**32ビットx86:**

- **`ZONE_DMA`**: 先頭16MB
- **`ZONE_NORMAL`**: 16MB - 約896MB（ダイレクトマップされる）
- **`ZONE_HIGHMEM`**: 約896MB以上（ダイレクトマップされず、kmapが必要）

---

## 関連項目 {#see-also}

- [概要](overview.md) - 全体像はここから
- [vmalloc](vmalloc.md) - 仮想メモリの割り当て
- [vrealloc](vrealloc.md) - 割り当てのサイズ変更
