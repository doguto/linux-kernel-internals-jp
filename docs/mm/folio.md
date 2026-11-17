# folio抽象化 {#the-folio-abstraction}

> `struct folio`は、継ぎはぎだらけのcompound pageに代わる第一級のカーネルオブジェクトであり、複数ページからなるメモリ単位を安全・明示的・効率的に扱えるようにする

## 主要なソースファイル {#key-source-files}

| ファイル | 役割 |
|------|---------|
| `include/linux/mm_types.h` | `struct folio`の定義とレイアウトのアサーション |
| `include/linux/page-flags.h` | `page_folio()`、`folio_page()`、フラグアクセサ |
| `include/linux/mm.h` | `folio_order()`、`folio_nr_pages()`、`folio_size()`、`folio_pfn()`、`folio_address()`、`folio_get()`、`folio_put()` |
| `include/linux/page_ref.h` | `folio_ref_count()`、`folio_try_get()` |
| `include/linux/pagemap.h` | `filemap_alloc_folio()`、`filemap_grab_folio()`、`readahead_folio()`、`filemap_add_folio()` |
| `include/linux/swap.h` | `folio_add_lru()` |
| `mm/folio-compat.c` | folio関数に処理を委譲するpage APIのシム |

---

## なぜfolioが存在するか {#why-folios-exist}

### compound pageの問題 {#the-compound-page-problem}

`struct page`はorder-0（単一の4KB）ページ向けに設計された。ヒュージページは後から*compound page*機構を使って追加された。これは連続したページ群を割り当て、最初のページを*head*、それ以降の各ページを*tail*とするものである。

tailは`page->compound_head`にheadへの逆ポインタを格納し、タグとしてLSBを立てる。そのため任意の`struct page *`を受け取るコードは、それがheadなのかtailなのかを常に確認しなければならない。tailであれば、まず逆ポインタをたどる必要がある。これを誤ると、headの参照カウント、フラグ、mappingポインタを気付かないうちに破壊してしまう。

その結果、次のようなパターンが至るところに現れる:

```c
/* 旧来のスタイル: すべての呼び出し元がこの手順を踏む必要がある */
page = compound_head(page);
```

これは誤りやすく、型システムからは見えず、数千もの呼び出し箇所に散らばっている。

### folioとは何か {#what-a-folio-is}

`struct folio`は、**物理的・仮想的・論理的に連続し、2のべき乗にアラインされたバイト集合**を表す第一級のカーネルオブジェクトである。サイズは常に`PAGE_SIZE`以上である。重要な保証は、`struct folio`へのポインタは常に*head*へのポインタであるということだ。tail folioというものは存在しない。

- order-0のfolioは単一の4KBページである。
- order-Nのfolioは物理的に連続した2^N個のページ（2^N × `PAGE_SIZE`バイト）である。
- サイズは文脈から推測されるのではなく、`folio_order()`を通じてオブジェクト自体にエンコードされている。

`struct folio`はMatthew WilcoxによってLinux 5.16（2022年）から導入され、それ以降のすべてのリリースで変換作業が続いている。

---

## struct folio {#struct-folio}

定義は`include/linux/mm_types.h`にある:

```c
struct folio {
    /* private: 無名unionはドキュメント化しない */
    union {
        struct {
            memdesc_flags_t flags;
            union {
                struct list_head lru;
                struct {
                    void *__filler;
                    unsigned int mlock_count;
                };
                struct dev_pagemap *pgmap;
            };
            struct address_space *mapping;
            union {
                pgoff_t index;
                unsigned long share;
            };
            union {
                void *private;
                swp_entry_t swap;
            };
            atomic_t _mapcount;
            atomic_t _refcount;
            /* ... CONFIG_MEMCG、WANT_PAGE_VIRTUAL など */
        };
        struct page page;   /* オフセット0のunion */
    };
    /* 2番目と3番目のページスロット: large folioのメタデータ */
    union { struct { ... }; struct page __page_1; };
    union { struct { ... }; struct page __page_2; };
    union { struct { ... }; struct page __page_3; };
};
```

設計上の要点は、`struct page page`が**無名unionの最初のメンバ**であり、オフセット0に位置することである。`struct folio *`を`struct page *`にキャストすると、常にheadページになる。カーネルはこれを、構造体定義の直後に置かれたコンパイル時の`static_assert`呼び出しで強制している:

```c
#define FOLIO_MATCH(pg, fl) \
    static_assert(offsetof(struct page, pg) == offsetof(struct folio, fl))
FOLIO_MATCH(flags,      flags);
FOLIO_MATCH(mapping,    mapping);
FOLIO_MATCH(_refcount,  _refcount);
/* ... */
```

レイアウトがずれれば、ビルドが失敗する。

### サイズとorderのアクセサ {#size-and-order-accessors}

これらはすべて`include/linux/mm.h`にある:

```c
/* 割り当てorder: 単一ページなら0、2^Nページなら N */
static inline unsigned int folio_order(const struct folio *folio);

/* このfolioがまたがるstruct pageの数: 1 << folio_order() */
static inline unsigned long folio_nr_pages(const struct folio *folio);

/* 総バイトサイズ: PAGE_SIZE << folio_order() */
static inline size_t folio_size(const struct folio *folio);

/* バイトサイズの2を底とする対数: PAGE_SHIFT + folio_order() */
static inline unsigned int folio_shift(const struct folio *folio);
```

order-0（一般的なケース）では、`folio_order()`は余分なフィールドを一切参照せずに0を返す。`folio_test_large()`のチェックで即座に短絡評価されるためである。

---

## page ↔ folioの変換 {#page-folio-conversion}

### page_folio() — pageからそれを所有するfolioへ {#page_folio-page-to-its-owning-folio}

`include/linux/page-flags.h`で定義されている:

```c
#define page_folio(p)   (_Generic((p),                          \
    const struct page *: (const struct folio *)_compound_head(p), \
    struct page *:       (struct folio *)_compound_head(p)))
```

これは`_Generic`マクロなので、const正当性が保たれる。`_compound_head()`は、LSBが立っていれば（tailページ）`compound_head`ポインタをたどり、すでにheadであればページ自体を返す。結果は常に`struct folio *`にキャストされたheadページである。

!!! note "参照なしでの競合"
    カーネルのドキュメントには次の注意がある: 呼び出し元がまだ参照を保持していない場合、`page_folio()`はfolioの分割と競合しうる。参照を取得した後で、そのfolioがまだ当該ページを含んでいることを必ず確認すること。

### folio_page() — folio内のn番目のページ {#folio_page-nth-page-within-a-folio}

```c
#define folio_page(folio, n)    (&(folio)->page + (n))
```

単純なポインタ演算である。`folio_page(folio, 0)`はheadページ、`folio_page(folio, folio_nr_pages(folio) - 1)`は最後のページである。境界チェックは行われない。呼び出し元が参照を保持していることが前提である。

### folio_file_page() — 指定したファイルインデックスに対応するページ {#folio_file_page-page-for-a-given-file-index}

large folioが複数のページキャッシュインデックスにまたがる場合、`folio_file_page()`はファイルオフセットを正しいサブページに対応付ける:

```c
/* include/linux/pagemap.h */
static inline struct page *folio_file_page(struct folio *folio, pgoff_t index);
```

### folio_pfn()とfolio_address() {#folio_pfn-and-folio_address}

```c
/* folio内の最初のページのページフレーム番号 */
static inline unsigned long folio_pfn(const struct folio *folio);

/* folioのカーネル仮想アドレス（ダイレクトマップ） */
static inline void *folio_address(const struct folio *folio);
```

対称的な`pfn_folio(pfn)`はPFNをfolioに変換する:

```c
static inline struct folio *pfn_folio(unsigned long pfn)
{
    return page_folio(pfn_to_page(pfn));
}
```

---

## 参照カウント {#reference-counting}

### モデル {#the-model}

参照カウントはfolio（headページ）の`_refcount`にある。tailページは独立した参照カウントを持たない。すべての参照操作はfolioを通じて行われる。

### folio_get()とfolio_put() {#folio_get-and-folio_put}

```c
/* include/linux/mm.h */

/* 参照カウントを増やす — すでに参照を保持していることが必要 */
static inline void folio_get(struct folio *folio);

/* 参照カウントを減らす — 0になるとfolioを解放する */
static inline void folio_put(struct folio *folio);
```

`get_page()`と`put_page()`は今も存在するが、ラッパーである:

```c
static inline void get_page(struct page *page)
{
    struct folio *folio = page_folio(page);
    /* ... */
    folio_get(folio);
}

static inline void put_page(struct page *page)
{
    struct folio *folio = page_folio(page);
    folio_put(folio);
}
```

### folio_ref_count() {#folio_ref_count}

```c
/* include/linux/page_ref.h */
static inline int folio_ref_count(const struct folio *folio);
```

生の参照カウントを返す。カーネルのドキュメントは`_refcount`に直接アクセスしないよう警告しており、常にこのアクセサを使うべきである。

### folio_try_get() {#folio_try_get}

まだ参照を保持しておらず、投機的に参照を取得する必要がある場合にこれを使う:

```c
/* include/linux/page_ref.h */
static inline bool folio_try_get(struct folio *folio);
```

参照を取得できた場合は`true`、folioがすでに0（解放済み、または分割/マイグレーションのために凍結済み）だった場合は`false`を返す。これは旧来の`get_page_unless_zero()`に相当するfolio版である。

!!! warning "folio_get()とfolio_try_get()の違い"
    `folio_get()`は、すでに参照を保持している場合（例: ロックを保持した状態でページキャッシュからfolioを受け取った場合）にのみ使う。データ構造を通じてfolioポインタを見つけ、ロックを取得する前にそれをピン留めする必要がある場合は`folio_try_get()`を使う。

---

## 現在folioが使われている場所 {#where-folios-are-used-today}

### ページキャッシュ {#page-cache}

`struct address_space`はキャッシュしたページをXArray（`i_pages`）に格納する。XArrayのエントリは`struct folio *`ポインタである。現代のページキャッシュ検索パスはすべてfolioを返す:

```c
/* 検索 — ロックされたfolioを返す。存在しなければ作成する */
static inline struct folio *filemap_grab_folio(
    struct address_space *mapping, pgoff_t index);

/* フラグ付きの低レベルな検索/作成 */
struct folio *__filemap_get_folio(struct address_space *mapping,
    pgoff_t index, fgf_t fgf_flags, gfp_t gfp);

/* folioをページキャッシュとLRUに挿入する */
int filemap_add_folio(struct address_space *mapping,
    struct folio *folio, pgoff_t index, gfp_t gfp);
```

### 先読み {#readahead}

`->readahead`アドレス空間操作は`struct readahead_control *`を受け取る。変換済みのドライバやファイルシステムは、ループ内で`readahead_folio()`を呼び出す:

```c
/* include/linux/pagemap.h */
static inline struct folio *readahead_folio(struct readahead_control *ractl);
```

各呼び出しは次に埋めるべきfolioを返し、制御構造体を`folio_nr_pages()`ページ分進める。

### ライトバック {#writeback}

ライトバックパスはfolioネイティブな操作を使う。`mm/folio-compat.c`には、旧来のページベースAPIをfolio版の対応関数に対応付けるシムが含まれている:

```c
/* これらのpage API関数は互換性シムである（folio-compat.c） */
void end_page_writeback(struct page *page)  /* → folio_end_writeback() */
bool set_page_dirty(struct page *page)      /* → folio_mark_dirty() */
void mark_page_accessed(struct page *page)  /* → folio_mark_accessed() */
```

`folio-compat.c`の先頭のコメントは明確である: *"All of the callers of these functions should be converted to use folios eventually."*（これらの関数の呼び出し元はすべて、最終的にfolioを使うように変換されるべきである。）

### LRUリスト {#lru-lists}

```c
/* include/linux/swap.h */
void folio_add_lru(struct folio *folio);
```

LRUの集計はfolio単位である。large folioはLRU上では`folio_nr_pages()`ページとして数えられるが、単一の追い出し単位である。カーネルはfolioを部分的に追い出すことはできない。

### ファイルシステムでの採用状況 {#filesystem-adoption}

ツリー内のほとんどのファイルシステムは、`->readahead`、`->read_folio`、`->writepages`メソッドを変換済みである。アドレス空間操作の構造体（`struct address_space_operations`）は現在、folioネイティブなスロット`dirty_folio`、`migrate_folio`、`error_remove_folio`を持つ。まだ変換されていないファイルシステムは互換性ラッパーにフォールバックする。変換作業は継続中である。

---

## large folio {#large-folios}

N ≥ 1であるorder-Nのfolioは*large folio*である。`folio_test_large()`はこれらに対してtrueを返す:

```c
/* include/linux/page-flags.h */
static inline bool folio_test_large(const struct folio *folio);
```

### 割り当て {#allocation}

ページキャッシュ用には`filemap_alloc_folio()`で割り当てる:

```c
/* include/linux/pagemap.h */
#define filemap_alloc_folio(gfp, order) \
    alloc_hooks(filemap_alloc_folio_noprof(gfp, order))
```

汎用のカーネル用途では、`folio_alloc()`（`include/linux/gfp.h`内）がバディアロケータをラップしている:

```c
#define folio_alloc(gfp, order) \
    alloc_hooks(folio_alloc_noprof(gfp, order))
```

### ファイルバックのlarge folioとTLB {#file-backed-large-folios-and-tlb}

ページキャッシュのfolioがPMDサイズにアラインされた複数のページにまたがる場合、それをマップするのに必要なページテーブルエントリは少なくなる。大きなファイルを順次readしたりmmapしたりするワークロードでは、large folioによって次のものが削減される:

- **ページフォールト回数**: 1回のフォールトでfolio全体が埋まる。
- **TLB圧迫**: アラインメントが許せば、連続した物理ページを単一のPMDエントリでカバーできる。
- **ライトバックのオーバーヘッド**: 1回の`writepages`呼び出しでfolio全体を単一のI/Oとして発行できる。

ページキャッシュのfolioの最大orderは`MAX_PAGECACHE_ORDER`によって決まり、これは`min(MAX_XAS_ORDER, PREFERRED_MAX_PAGECACHE_ORDER)`である。ほとんどの64ビット構成では、`PREFERRED_MAX_PAGECACHE_ORDER`は`HPAGE_PMD_ORDER`（order 9、2MB）であり、PMDサイズと一致する。

ファイルシステムは、サポートするfolioのorder範囲を次の関数で通知する:

```c
/* include/linux/pagemap.h */
static inline void mapping_set_folio_order_range(
    struct address_space *mapping,
    unsigned int min, unsigned int max);
```

### mTHPとの関係 {#relation-to-mthp}

ページキャッシュのlarge folioは、匿名メモリ向けのmulti-size THP（mTHP）に対応するファイルバック版である。匿名側については[mTHP](mthp.md)を参照。重要な違いは、ファイルバックのlarge folioはページキャッシュと`struct address_space`によって管理され、匿名のlarge folioはTHPの仕組みと`struct vm_area_struct`によって管理されるという点である。

---

## 進行中の変換 {#the-conversion-in-progress}

### パターン: folio-compat.c {#the-pattern-folio-compatc}

`mm/folio-compat.c`は、変換がどのように進むかを示す典型例である。このファイル内の各関数は同じ構造を持つ:

```c
/* 旧来のページベースAPI — 未変換の呼び出し元のためにまだエクスポートされている */
void unlock_page(struct page *page)
{
    return folio_unlock(page_folio(page));
}
EXPORT_SYMBOL(unlock_page);
```

page関数はfolioに変換して処理を委譲する。呼び出し元が`folio_unlock()`を直接呼ぶように変換されていくにつれ、pageのシムは最終的に呼び出し元がなくなり、削除できるようになる。

### 命名規則 {#naming-convention}

カーネルは一貫した命名規則を採用している:

| APIスタイル | 命名パターン | 例 |
|-----------|---------------|---------|
| folioネイティブ（新） | `folio_*` | `folio_lock()`、`folio_mark_dirty()` |
| ページベース（レガシー） | 元の名前 | `lock_page()`、`set_page_dirty()` |
| 変換ヘルパー | `page_folio()` | `page_folio(page)` |

関数名が`folio_`で始まる場合、それは`struct folio *`を受け取り、folio全体に対して操作する。`struct page *`を受け取る場合は`mm/folio-compat.c`を確認すること。おそらくシムである。

### 関数が変換済みかどうかを見分ける方法 {#how-to-tell-if-a-function-has-been-converted}

1. 引数の型を確認する: `struct folio *`なら変換済みである。
2. `mm/folio-compat.c`を見る: 旧名がそこにあれば、実装は`folio_*`の対応関数に移っている。
3. アドレス空間操作の構造体（`include/linux/fs.h`の`struct address_space_operations`）を調べる: `*_folio`という名前のスロットは変換済みで、まだ`*_page`という名前のスロットは未変換である。

---

## folioを意識したコードの書き方 {#writing-folio-aware-code}

### 判断ルール {#the-decision-rule}

| 扱っている対象… | 使うもの |
|----------------------|-----|
| ページキャッシュ（ファイルデータ） | 全面的にfolio API |
| LRUの追跡 | `folio_add_lru()` |
| ライトバック状態 | `folio_mark_dirty()`、`folio_start_writeback()`、`folio_end_writeback()` |
| 生のページ割り当て（`alloc_pages`） | `struct page *`を返すので、直ちに`page_folio()`を呼ぶ |
| `struct page *`を渡す既存コード | 境界で変換する。フラグに触れる前にfolioの参照を保持する |

### folioを意識した典型的な読み込みパス（概略） {#typical-folio-aware-read-path-sketch}

```c
struct folio *folio;

/* 1. ページキャッシュ内でfolioを検索するか作成する */
folio = filemap_grab_folio(mapping, index);
if (IS_ERR(folio))
    return PTR_ERR(folio);

/* folioはこの時点でロックされ、参照されている */

/* 2. すでに最新かどうかを確認する */
if (folio_test_uptodate(folio)) {
    folio_unlock(folio);
    return 0;
}

/* 3. folio全体に対してI/Oを発行する */
/* ... ファイルシステム固有の読み込み ... */

/* 4. 最新としてマークし、解放する */
folio_mark_uptodate(folio);
folio_unlock(folio);
folio_put(folio);
```

### folioを直接割り当てる {#allocating-a-folio-directly}

ページキャッシュ外でfolioが必要な場合（例: カーネル内のプライベートなバッファ）:

```c
/* 単一ページ（order-0）のfolioを割り当てる */
struct folio *folio = folio_alloc(GFP_KERNEL, 0);
if (!folio)
    return -ENOMEM;

/* 使用する ... */

folio_put(folio);   /* アロケータの参照を落とす。0になれば解放する */
```

### 事前の参照なしにfolio_get()を呼ばない {#do-not-call-folio_get-without-a-prior-reference}

```c
/* 誤り: ロックを保持せずにデータ構造から得たfolioポインタ */
folio_get(folio);   /* 解放と競合する可能性がある */

/* 正しい: 投機的に参照を取得する */
if (!folio_try_get(folio))
    return -ENOENT;  /* 取得する前にfolioが解放された */
```

---

## リファレンスまとめ {#reference-summary}

| 関数 | ヘッダ | 説明 |
|----------|--------|-------------|
| `page_folio(page)` | `page-flags.h` | 任意のpageをそれを所有するfolioに変換する |
| `folio_page(folio, n)` | `page-flags.h` | folio内のn番目のページを取得する |
| `folio_order(folio)` | `mm.h` | 割り当てorder（単一ページなら0） |
| `folio_nr_pages(folio)` | `mm.h` | ページ数: `1 << folio_order()` |
| `folio_size(folio)` | `mm.h` | バイトサイズ: `PAGE_SIZE << folio_order()` |
| `folio_pfn(folio)` | `mm.h` | 最初のページのPFN |
| `folio_address(folio)` | `mm.h` | カーネル仮想アドレス |
| `folio_get(folio)` | `mm.h` | 参照カウントを増やす（事前の参照が必要） |
| `folio_put(folio)` | `mm.h` | 参照カウントを減らす。0で解放する |
| `folio_try_get(folio)` | `page_ref.h` | 投機的に参照を取得する |
| `folio_ref_count(folio)` | `page_ref.h` | 生の参照カウント |
| `folio_test_large(folio)` | `page-flags.h` | order ≥ 1のfolioならtrue |
| `folio_add_lru(folio)` | `swap.h` | folioをLRUリストに追加する |
| `filemap_grab_folio(mapping, index)` | `pagemap.h` | ページキャッシュのfolioを検索または作成する |
| `filemap_alloc_folio(gfp, order)` | `pagemap.h` | ページキャッシュのfolioを割り当てる |
| `readahead_folio(ractl)` | `pagemap.h` | 先読みバッチから次のfolioを取得する |

## 参考文献 {#further-reading}

- [LWN: Folios](https://lwn.net/Articles/849538/) — folioの概念を紹介し、なぜ`struct page`を置き換える必要があったかを説明したMatthew Wilcoxの2021年の記事
- [LWN: Large folios for the page cache](https://lwn.net/Articles/893512/) — ファイルバックメモリとページキャッシュに対するlarge folioサポートについての続報
- [mm/folio-compat.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/folio-compat.c) — 互換性シム層。ここにある関数はすべて、`folio_*`の対応関数に処理を委譲するpage API呼び出しであり、変換を待っている
- [mm/filemap.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/filemap.c) — ページキャッシュの中核。`__filemap_get_folio()`と`filemap_add_folio()`が主要なfolioネイティブのエントリポイントである
- [include/linux/mm_types.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mm_types.h) — `FOLIO_MATCH`によるレイアウトのアサーションを含む、`struct folio`の正規の定義
- [mthp.md](mthp.md) — multi-size THP: ファイルバックのlarge folioに対応する匿名メモリ版。どちらも同じorder-Nのfolio表現を共有している
