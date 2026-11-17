# vmalloc {#vmalloc}

> 散らばった物理pageから、仮想的に連続したメモリを割り当てる

## vmallocとは何か {#what-is-vmalloc}

`vmalloc()` は、*仮想*アドレス空間では連続しているが、物理的には連続していないpageで裏付けられうるメモリを割り当てる。このため、物理的な連続性が不要な大きな割り当てに適している。（vmalloc領域がダイレクトマップやvmemmapに対してどこに位置するか、そしてそもそもなぜ別の領域なのかについては、[カーネルアドレス空間](kernel-address-space.md)を参照。）

```c
void *vmalloc(unsigned long size);
void vfree(const void *addr);
```

## 歴史と変遷 {#history-evolution}

### 起源（1993年） {#origins-1993}

vmallocは最初のリリースの2年後、1993年にLinuxに追加された。

[`mm/vmalloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmalloc.c) より:
```
Copyright (C) 1993  Linus Torvalds
Support of BIGMEM added by Gerhard Wichert, Siemens AG, July 1999
SMP-safe vmalloc/vfree/ioremap, Tigran Aivazian, May 2000
Major rework to support vmap/vunmap, Christoph Hellwig, SGI, August 2002
```

Hellwigによる改修は[LWNで解説されている](https://lwn.net/Articles/7473/)。

### v2.6.28: 大規模な書き直し - 赤黒木と遅延TLB {#v2628-major-rewrite-rbtree-lazy-tlb}

**コミット**: [db64fe02258f](https://git.kernel.org/linus/db64fe02258f) ("mm: rewrite vmap layer")
**カーネル**: v2.6.28
**作者**: Nick Piggin

*注: lore.kernel.org上の2009年以前のLKMLアーカイブはまばらである。コミットメッセージに設計の根拠が記されている。*

**問題**: vunmapはカーネル全体のグローバルなTLBフラッシュ（全CPUへのブロードキャストIPI）を必要とし、そのすべてが高速パスで書き込み用に取得される単一のグローバルなrwlockの下で行われていた。CPU数が増えるにつれ、スケーラビリティは二次関数的に悪化した。

**解決策**:
1. **遅延TLBフラッシュ**: vunmapですぐにはフラッシュしない。アドレスは再割り当てされるまで再利用されないので、複数のアンマップを1回のフラッシュにまとめる。
2. **vmap_areaの赤黒木**: 線形リストのスキャンの代わりに高速な `O(log n)` のルックアップ。
3. **percpuのフロントエンド**: 小さなvmapのための高速でスケーラブルな割り当て。

**トレードオフ**: XENとPATはエイリアシングの問題のため、即時のTLBフラッシュを必要とする。これらは必要なときに `vm_unmap_aliases()` を呼んでフラッシュを強制する。

*注: コミットメッセージには、遅延TLBフラッシュとRCUについての詳細な設計根拠が記されている。*

### v5.13: ヒュージページのサポート {#v513-huge-page-support}

**コミット**: [121e6f3258fe](https://git.kernel.org/linus/121e6f3258fe) ("mm/vmalloc: hugepage vmalloc mappings") | [LKML](https://lore.kernel.org/linux-mm/1616036421.amjz2efujj.astroid@bobo.none/)
**カーネル**: v5.13
**作者**: Nicholas Piggin

**何が変わったか**: vmallocが大きな割り当てに `PMD` サイズのヒュージページを使えるようになった。

**仕組み**:

- 割り当てが `PMD` サイズ以上なら、まずヒュージページを試す
- ヒュージページの割り当てが失敗したら小さなpageにフォールバックする
- `VM_NOHUGE` フラグで小さなpageを強制する（厳格なrwxを要するモジュールの割り当てに必要）
- ブートオプション `nohugevmalloc` でグローバルに無効化する

**トレードオフ**: 内部断片化は増える（`1.1MB` が必要なときに `2MB` を割り当てる）が、必要なTLBエントリは減る。

### v6.12: vreallocの導入 {#v612-vrealloc-introduction}

**コミット**: [3ddc2fefe6f3](https://git.kernel.org/linus/3ddc2fefe6f3) ("mm: vmalloc: implement vrealloc()") | [LKML](https://lore.kernel.org/linux-mm/20240717222427.2211-2-dakr@kernel.org/)
**カーネル**: v6.12
**作者**: Danilo Krummrich

詳しい歴史は[vrealloc](vrealloc.md)を参照。

## vmallocをいつ使うか {#when-to-use-vmalloc}

**vmallocを使うべき場合:**

- 割り当てが大きい（複数page）
- 物理的な連続性が不要
- メモリをDMAに使わない

**vmallocを使うべきでない場合:**

- 小さな割り当て（kmallocを使う。オーバーヘッドが小さい）
- 物理的な連続性が必要（kmallocまたはalloc_pagesを使う）
- DMA操作（物理アドレスが必要）
- 性能が重要なホットパス（ページテーブルのオーバーヘッド、TLBミスの可能性）

## 仕組み {#how-it-works}

### 割り当ての流れ {#allocation-flow}

1. **仮想アドレス範囲を見つける**: vmap_areaの赤黒木から空き領域を探す
2. **物理pageを割り当てる**: バディアロケータから個々のpageを得る
3. **ページテーブルを構築する**: 仮想アドレスを物理pageにマップする
4. **ポインタを返す**: 呼び出し元からは連続した仮想メモリに見える

```
Virtual Address Space          Physical Memory
+---------------+              +---------------+
|   vmalloc     |------------->|   Page A      |
|   buffer      |              +---------------+
|   (contiguous |              :               :
|    virtual)   |------------->|   Page B      | (scattered!)
|               |              +---------------+
|               |              :               :
|               |------------->|   Page C      |
+---------------+              +---------------+
```

### 解放の流れ（遅延TLB） {#free-flow-lazy-tlb}

1. **領域を解放対象としてマークする**: 遅延解放リストに追加する
2. **TLBをすぐにはフラッシュしない**: アドレスはまだ再利用されない
3. **まとめてフラッシュする**: しきい値に達したとき、または `vm_unmap_aliases()` が呼ばれたとき
4. **pageを返す**: 物理pageをバディアロケータに解放する

## 主要なデータ構造 {#key-data-structures}

vmallocが割り当てをどう追跡するか:

```
  vmap_area RBTree                      vm_struct                Physical Pages
  (fast lookup)                      (allocation info)           (actual memory)

+------------------+              +-------------------+        +---------+
| vmap_area        |              | vm_struct         |        | struct  |
|------------------|              |-------------------|        | page    |
| va_start --------|--+           | addr = 0xFFFF0000 |        +---------+
| va_end           |  |           | size = 12288      |            ^
| rb_node ------+  |  |           | nr_pages = 3      |            |
| vm -----------)--|--+---------->| pages[] ----------|--+---------+
+------------------+  |           | flags = VM_ALLOC  |  |     +---------+
        |             |           +-------------------+  |     | struct  |
        v             |                                  +---->| page    |
   +----+----+        |                                  |     +---------+
   |         |        |                                  |         ^
   v         v        |                                  |         |
 [left]   [right]     |                                  |     +---------+
                      |                                  +---->| struct  |
                      v                                        | page    |
              Virtual Address                                  +---------+
              0xFFFF0000-0xFFFF3000
```

### vm_struct {#vm_struct}

vmalloc領域を記述する。

```c
struct vm_struct {
    void *addr;              /* 仮想アドレス */
    unsigned long size;      /* ガードページを含むサイズ */
    struct page **pages;     /* 裏付けとなる page の配列 */
    unsigned int nr_pages;   /* page の数 */
    phys_addr_t phys_addr;   /* 物理アドレス（ioremap 用） */
    unsigned long flags;     /* VM_ALLOC、VM_MAP、VM_IOREMAP など */
    unsigned long requested_size; /* 実際に要求されたサイズ（v6.13 以降） */
};
```

### vmap_area {#vmap_area}

vmallocのアドレス空間の使用状況を追跡する。

```c
struct vmap_area {
    unsigned long va_start;  /* 仮想範囲の開始 */
    unsigned long va_end;    /* 仮想範囲の終了 */
    struct rb_node rb_node;  /* 高速ルックアップ用の赤黒木ノード */
    struct list_head list;   /* 連結リスト */
    struct vm_struct *vm;    /* 関連付けられた vm_struct */
};
```

## バリエーション {#variants}

| 関数 | 説明 | 導入 |
|----------|-------------|-------|
| `vmalloc()` | 標準の割り当て | 当初から |
| `vzalloc()` | ゼロ初期化 | v2.6.37 |
| `vmalloc_node()` | NUMAを意識 | v2.6.16 |
| `vmalloc_huge()` | ヒュージページを優先 | v5.13 |
| `vmalloc_no_huge()` | 小さなpageを強制 | v5.13 |
| `vrealloc()` | 割り当てのサイズ変更 | v6.12 |
| `vfree()` | 割り当ての解放 | 当初から |

## 設計判断 {#design-decisions}

### なぜガードページか {#why-guard-pages}

各vmalloc領域の末尾には（マップされていない）ガードページがある。バッファオーバーフローはガードに当たって即座にページフォールトを起こし、隣接する割り当てを黙って破壊することがない。

**コミット**: ガードページを望まない特殊なケースのために `VM_NO_GUARD` フラグが存在するが、[bd1a8fb2d43f](https://git.kernel.org/linus/bd1a8fb2d43f) ("mm/vmalloc: don't allow VM_NO_GUARD on vmap()") がその使用を制限している。（[LKML](https://lore.kernel.org/lkml/YUMfdA36fuyZ+%2Fxt@hirez.programming.kicks-ass.net/)）

### なぜ遅延TLBフラッシュか {#why-lazy-tlb-flushing}

TLBフラッシュは高コストである。

- 全CPUへのIPI（Inter-Processor Interrupt）
- 各CPUが自身のTLBをフラッシュしなければならない
- すべてロックの下で行われる

複数のvunmapを1回のフラッシュにまとめることで、このコストを償却する。カーネルは `vmap_lazy_nr` を追跡し、それがしきい値を超えたときにフラッシュする。

### なぜvmap_areaに赤黒木か {#why-rbtree-for-vmap_area}

v2.6.28より前、vmallocは単純な連結リストを使っていた。システムが大きくなるにつれ、線形の `O(n)` 探索がボトルネックになった。赤黒木は `O(log n)` のルックアップ、挿入、削除を提供する。

## よくある問題 {#common-issues}

### TLBフラッシュの嵐 {#tlb-flush-storms}

vmalloc/vfreeの負荷が高いと、遅延フラッシュが「フラッシュの嵐」を引き起こすことがある。しきい値に達したときに多数のIPIが一斉に発生するのである。

### アドレス空間の枯渇 {#address-space-exhaustion}

vmallocのアドレス空間（`VMALLOC_START` から `VMALLOC_END` まで）には限りがある。32ビットシステムでは、これはわずか `128MB` であることが多かった。BPF、モジュール、ドライバのようなヘビーユーザーはこれを使い果たしうる。

### NUMAの局所性 {#numa-locality}

vmallocは常にNUMAを意識していたわけではない。現代のカーネルは、可能な場合は `vmalloc_node()` によって同じノードからpageを割り当てようとする。

## 試してみる {#try-it-yourself}

### vmallocの使用状況を調べる {#inspect-vmalloc-usage}

稼働中のシステムですべてのvmalloc割り当てを表示する。

```bash
# すべての vmalloc 領域をアドレス、サイズ、呼び出し元とともに一覧表示する
cat /proc/vmallocinfo

# 出力例:
# 0xffffc90000000000-0xffffc90000201000 2101248 bpf_prog_alloc+0x5e/0x100 pages=512 vmalloc N0=512
# 0xffffc90000201000-0xffffc90000206000   20480 pcpu_alloc+0x3fa/0x7e0 pages=4 vmalloc N0=4
```

主なフィールド:

- **アドレス範囲**: 仮想アドレスの開始-終了
- **サイズ**: バイト単位
- **呼び出し元**: 割り当てた関数（リークのデバッグに有用）
- **pages=N**: この割り当てを裏付ける物理pageの数
- **vmalloc**: 割り当ての種類（ioremap、vmapと区別される）
- **N0=N**: NUMAノードごとの分布

### vmallocの統計を確認する {#check-vmalloc-statistics}

```bash
# 使用中の vmalloc page の合計
grep VmallocUsed /proc/meminfo

# vmalloc のアドレス空間の上限
grep Vmalloc /proc/meminfo
# VmallocTotal:   34359738367 kB  (アドレス空間のサイズ)
# VmallocUsed:           0 kB     (現代のカーネルでは 0 と表示されることがある)
# VmallocChunk:          0 kB     (最大の連続した空きブロック)
```

### テストモジュールを実行する {#run-the-test-module}

カーネルにはvmallocの機能をテストするための `test_vmalloc` が含まれている。

```bash
# テストモジュールをビルドする（カーネルツリー内で）
make M=lib/ lib/test_vmalloc.ko

# 特定のテストを指定してロードする（QEMU またはテストマシン上で）
# テストマスクのビット: すべてのオプションは lib/test_vmalloc.c を参照
modprobe test_vmalloc run_test_mask=0xFFFF

# あるいは手早くテストするなら virtme-ng 経由で
virtme-ng --append 'test_vmalloc.run_test_mask=0xFFFF'

# 結果を確認する
dmesg | grep test_vmalloc
```

### 主要なコードの場所 {#key-code-locations}

| 関数 | ファイル:行 | 役割 |
|----------|-----------|--------------|
| `vmalloc()` | [`mm/vmalloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmalloc.c) | メインのエントリポイント |
| `__vmalloc_node_range()` | [`mm/vmalloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmalloc.c) | 割り当ての中核ロジック |
| `vfree()` | [`mm/vmalloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmalloc.c) | vmallocメモリを解放する |
| `find_vmap_area()` | [`mm/vmalloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmalloc.c) | 赤黒木のルックアップ |
| `vrealloc()` | [`mm/vmalloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmalloc.c) | 割り当てのサイズ変更 |

*注: 行番号はカーネルバージョンによって異なる。ファイル内で関数名を検索すること。*

### 簡単なテストを書く {#write-a-simple-test}

```c
// カーネルモジュール内、または BPF 経由で
#include <linux/vmalloc.h>

void test_vmalloc_basic(void)
{
    void *ptr;

    // 仮想的に連続した 1MB を割り当てる
    ptr = vmalloc(1024 * 1024);
    if (!ptr) {
        pr_err("vmalloc failed\n");
        return;
    }

    // 使う...
    memset(ptr, 0, 1024 * 1024);

    // 解放する
    vfree(ptr);
}
```

---

## リファレンス {#references}

### 主要なコミット {#key-commits}
| コミット | カーネル | 説明 |
|--------|--------|-------------|
| [1da177e4c3f4](https://git.kernel.org/linus/1da177e4c3f4) | [v2.6.12](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tag/?h=v2.6.12) | gitの歴史の始まり |
| [db64fe02258f](https://git.kernel.org/linus/db64fe02258f) | [v2.6.28](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tag/?h=v2.6.28) | 赤黒木 + 遅延TLBへの書き直し |
| [121e6f3258fe](https://git.kernel.org/linus/121e6f3258fe) | [v5.13](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tag/?h=v5.13) | ヒュージページのサポート |
| [3ddc2fefe6f3](https://git.kernel.org/linus/3ddc2fefe6f3) | [v6.12](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tag/?h=v6.12) | vreallocの導入 |

### コード {#code}
- [`mm/vmalloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmalloc.c) - 実装
- [`include/linux/vmalloc.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/vmalloc.h) - API

### 関連 {#related}
- [vrealloc](vrealloc.md) - 割り当てのサイズ変更
- [overview](overview.md) - メモリ管理の概要

## 参考文献 {#further-reading}

- [vrealloc.md](vrealloc.md) — v6.12で追加された、vmallocベースのサイズ変更プリミティブ `vrealloc()`
- [slab.md](slab.md) — `kvmalloc()` はまずkmallocを試し、大きな割り当てではvmallocにフォールバックする
- [page-allocator.md](page-allocator.md) — vmallocに個々のpageを供給するバディアロケータ
- [page-tables.md](page-tables.md) — vmallocの仮想から物理へのマッピングの基礎となるページテーブルの仕組み
- [mm/vmalloc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmalloc.c) — 実装全体。`__vmalloc_node_range()` が中核のアロケータで、`vmap_area` の赤黒木は冒頭にある
- [include/linux/vmalloc.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/vmalloc.h) — `vmalloc()`、`vfree()`、`vmap()` やバリエーション関数を含む公開API
- [lib/test_vmalloc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/test_vmalloc.c) — 割り当て、マッピング、vreallocのシナリオを網羅するツリー内のテストモジュール
- [LWN: A deep dive into vmalloc](https://lwn.net/Articles/828313/) — vmap_areaの赤黒木、遅延TLBフラッシュ、ヒュージページのサポートについての詳細な解説
- [LWN: vmap() improvements](https://lwn.net/Articles/7473/) — vmap/vunmapをvmallocの内部から分離した、Christoph Hellwigによる2002年の改修
- [db64fe02258f](https://git.kernel.org/linus/db64fe02258f) — 赤黒木と遅延TLBフラッシュのバッチ化を導入した、Nick Pigginによるv2.6.28の書き直し
- `Documentation/admin-guide/mm/concepts.rst` — vmalloc領域と、ダイレクトマッピングと並ぶその役割を扱うカーネルドキュメント
