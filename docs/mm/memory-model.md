# 物理メモリモデル {#the-physical-memory-model}

> カーネルがRAMのすべてのpageをどう追跡しているか — FLATMEM、SPARSEMEM、vmemmap — そして `struct page` 自体がいかにしてメモリディスクリプタへと解体されつつあるか

カーネルはマシン内のすべての物理ページフレームについて `struct page` を1つずつ保持しており、**物理フレーム番号**（PFN）からその `struct page` へ、そしてその逆へと、アロケータの最もホットなパスで絶えず変換する必要がある。*`struct page` の巨大な配列をどのように格納するか*、それが「メモリモデル」の意味するところである。これは何度も書き直されてきた。書き直しはいずれも、物理メモリがより乱雑になったことに迫られたものだった。そしてその中心にある `struct page` は今、解体されつつある。

このページはその抽象化について扱う。なぜそれが存在するのか、カーネルがこれまで使ってきたモデル、そして全体がどこへ向かっているのか。結果としてできる配列がカーネルアドレス空間のどこに実際に置かれるかについては[カーネルアドレス空間](kernel-address-space.md)を、データ構造としての `struct page` 自体については[pageの一生](life-of-page.md)を参照。

---

## そもそもなぜ「モデル」があるのか {#why-theres-a-model-at-all}

2つの事実が衝突する。

1. **`pfn_to_page()` と `page_to_pfn()` は高速でなければならない。** これらはすべての割り当て、解放、回収スキャン、ページテーブルウォークで実行される。理想的には単一の配列インデックスであるべきだ。
2. **物理メモリはきれいな配列ではない。** ファームウェアは穴を開け、メモリマップドデバイスはウィンドウを占有し、NUMAノードはアラインされた数テラバイトのオフセットから始まり、[ホットプラグ](memory-hotplug.md)は実行時にRAMを出現させたり消したりする。

「PFNごとに1つの `struct page` を、PFNでインデックスする」という素朴な配列は高速な選択肢だが、穴ごとに64バイトの `struct page` を浪費する。物理配置が疎なマシンでは何ギガバイトにもなる。**メモリモデル**は、「`struct page` 配列はどこにあり、穴の代償を払わずにどうやって*高速に*インデックスするか」に対するカーネルの答えである。[Documentation/mm/memory-model.rst](https://docs.kernel.org/mm/memory-model.html)によれば、各モデルは単に独自の `pfn_to_page()` / `page_to_pfn()` を定義している。

---

## FLATMEM: 1つのフラットな配列 {#flatmem-one-flat-array}

最も単純なモデル。単一のグローバルな `mem_map[]` 配列がPFN範囲全体をカバーする。

```c
/* 本質的にはこれがモデルのすべて */
#define __pfn_to_page(pfn)  (mem_map + ((pfn) - ARCH_PFN_OFFSET))
#define __page_to_pfn(page) ((unsigned long)((page) - mem_map) + ARCH_PFN_OFFSET)
```

減算1回、インデックス1回。これ以上ないほど高速である。FLATMEMは**物理メモリが本質的に連続している場合**に正しく効率的であり、そのため今でも多くの小規模システムや組み込みシステムで使われるモデルである。弱点はまさに穴である。範囲内のすべてのPFNについて、存在するか否かにかかわらず `struct page` が割り当てられるため、物理マップの隙間はRAMの浪費という形で代償を払うことになる。

---

## DISCONTIGMEM: ノードごとの配列（知っておく価値のある行き止まり） {#discontigmem-per-node-arrays-a-dead-end-worth-knowing}

穴を扱う最初の試みはNUMAから生まれた。大規模なNUMAマシンにはノード*間*に巨大な隙間があるため、FLATMEMの単一配列ではノード間の空白に何ギガバイトも浪費してしまう。**DISCONTIGMEM**は各NUMAノードに独自の `mem_map`（`pg_data_t.node_mem_map`）を与え、実際にメモリのあるノードだけがメモリを消費するようにした。`pfn_to_page()` はまず所属するノードを見つけ、そのノード内でインデックスする。

これが置き換えられたのは、「どのNUMAノードか」と「メモリのどの部分が存在するか」という2つの*異なる*問題を1つの仕組みに混同していたからである。1つのノード自体が疎であることもあり、DISCONTIGMEMはそれをうまく扱えなかった。SPARSEMEM（後述）はNUMAとは独立に穴を直接解決し、DISCONTIGMEMは無用の長物となった。最後の利用者（alpha、arc、m68k）はLinux 5.11までに移行され、モデルは2021年に完全に削除された（[LWN: Remove the DISCONTIGMEM memory model](https://lwn.net/Articles/858333/)）。ここで取り上げるのは、SPARSEMEMの設計がその欠点への直接的な対応だからにすぎない。

---

## SPARSEMEM: セクション {#sparsemem-sections}

事実上すべての現代の64ビットシステムで使われるモデル。物理アドレス空間を固定サイズの**セクション**に分割し、*存在する*各セクションに個別に割り当てた `struct page` の部分配列を与える。

- セクションの幅は `SECTION_SIZE_BITS` である。**x86-64では27ビット、つまり128MB**である（[arch/x86/include/asm/sparsemem.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/sparsemem.h)）。
- `struct mem_section` の配列（大規模マシンでは2段の基数木）が、どのセクションが存在し、その `struct page` 部分配列がどこにあるかを記録する。`mem_section` 構造体の詳細は[メモリホットプラグ](memory-hotplug.md)を参照。ホットプラグはSPARSEMEMの最大の受益者である。DIMMの追加や取り外しはセクションの追加や削除そのものだからだ。

代償はルックアップにある。「古典的な」SPARSEMEMの `pfn_to_page()` はもはや単なるインデックスではない。PFNからセクションを取り出し、そのセクションの `struct page` のベースを見つけ、その中でインデックスする。

```c
/* 古典的な sparsemem（概念的なもの） */
struct mem_section *ms = __pfn_to_section(pfn);
struct page *base = __section_mem_map_addr(ms);
return base + pfn;   /* base はあらかじめオフセット済みなので、これで正しくインデックスされる */
```

*すべての* `pfn_to_page()`（ホットパスである）に加わるこの余分な間接参照が、穴に `struct page` を浪費しないためにSPARSEMEMが払う代償である。これは**2.6.13**（2005年）でマージされた。もともとは[メモリホットプラグ](memory-hotplug.md)を可能にするためだった。

---

## SPARSEMEM_VMEMMAP: O(1)を取り戻す {#sparsemem_vmemmap-getting-o1-back}

今日主流の構成であり、x86-64とほとんどの64ビットアーキテクチャでデフォルトとなっているのは、FLATMEMのメモリ浪費*なしに*FLATMEMの単一インデックスの速度を取り戻す巧妙な手法である。

秘訣は安価な**仮想**アドレス空間を使うことにある。PFNで直接インデックスされる、仮想的に連続した `struct page` の `vmemmap` 配列をマップするが、裏付けとなるpageは実際に存在するセクションについてのみ用意する。

```c
/* include/asm-generic/memory_model.h — CONFIG_SPARSEMEM_VMEMMAP */
#define __pfn_to_page(pfn)  (vmemmap + (pfn))
#define __page_to_pfn(page) ((unsigned long)((page) - vmemmap))
```

`pfn_to_page()` は再び加算1回になる。穴のコストはマップされていない仮想アドレス空間だけであり、64ビットマシンではそれはタダである。フラット配列の速度と疎な配列の密度という両方の長所を兼ね備えており、だからこそ `vmemmap` 領域は[カーネルアドレス空間](kernel-address-space.md#vmemmap-the-struct-page-array-as-an-address-space-trick)に専用の場所を持っている（そのページはそれがどこに置かれ、どうサイズが決まるかを扱い、このページはなぜこのモデルが存在するかを扱う）。これは**2.6.24**（2008年）で登場し、`ZONE_DEVICE` の基盤でもある。`ZONE_DEVICE` はデバイスのアドレス範囲についてvmemmapを用意することで、デバイスメモリに `struct page` を提供する。

| モデル | `struct page` 配列 | `pfn_to_page()` | 穴でメモリを浪費するか | ホットプラグに対応するか |
|-------|--------------------|-----------------|------------------------|------------------|
| FLATMEM | 1つのフラットな `mem_map[]` | インデックス（最速） | **する** | しない |
| DISCONTIGMEM *(削除済み)* | NUMAノードごとの配列 | ノードを探してからインデックス | ノード間のみ | しない |
| SPARSEMEM | セクションごとの部分配列 | セクションのルックアップ後にインデックス | しない | **する** |
| SPARSEMEM_VMEMMAP | 疎に裏付けられた仮想配列 | インデックス（最速） | しない | **する** |

（厳密には、`SPARSEMEM_VMEMMAP` は4つ目のモデルではなくSPARSEMEMのデフォルトの*構成*である。セクションの仕組みは同じで、`pfn_to_page()` が異なるだけだ。しかしコストの差が大きいため、別に挙げている。）

FLATMEM → DISCONTIGMEM → SPARSEMEM → vmemmapという歴史的な流れは、Mike Rapoportの[Memory: the flat, the discontiguous, and the sparse](https://lwn.net/Articles/789304/)によくまとめられている。

---

## 問題のもう半分: `struct page` 自体 {#the-other-half-of-the-problem-struct-page-itself}

メモリモデルは*配列がどこにあるか*を決める。もう1つのコストは*各要素がどれだけ大きいか*であり、現在圧力がかかっているのは `struct page` の方である。

各 `struct page` は**64バイト**で、RAMの4KBごとに1つある。これは物理メモリを記述するだけで**全物理メモリの約1.6%**を固定的に費やすことを意味する。1TBのマシンでは、1バイトも何かに使われる前に約16GBが `struct page` に使われる。

さらに悪いことに、`struct page` は1995年以来さまざまな意味を蓄積してきたunionのunionである。同じ64バイトが、そのpageが匿名か、ファイルに裏付けられているか、slabオブジェクトか、ページテーブルのpageか、バディアロケータで空いているか、デバイスメモリかによってまったく異なる意味を持ち、重なり合うフィールドは文脈によってのみ区別される。窮屈で型安全でなく、これが[folio](folio.md)が存在する理由である。folioは第一歩であり、ファイルメモリと匿名メモリに、生の `struct page` の代わりに独立した正しく型付けされたハンドルを与えた。

---

## 向かう先: メモリディスクリプタ（memdesc） {#where-its-going-memory-descriptors-memdesc}

folioの作業から生まれ、Matthew Wilcoxが推進する長期目標は、`struct page` を完全に解体することである。その代わりに、pageごとに**8バイトの「メモリディスクリプタ」**（memdesc）を置く。これは型付きポインタであり、下位ビットがそのpageが*どの種類*かをエンコードし、残りのビットが型固有の構造体（folio、slab、バディの記録、ページテーブルディスクリプタなど）を指す。

この移行は理論上の話ではなく、すでに大きく進んでいる。

- 型固有のディスクリプタが**1つずつ `struct page` から切り出されている**。[`struct slab`](https://lwn.net/Articles/881039/)（5.17）、`struct ptdesc`（ページテーブルのpage）、netmemディスクリプタはすでに分離されており、移行中はそれぞれが `struct page` に重ねて配置されている。
- 難しいのは「**ロングテール**」である。ツリー全体のカーネルコードが `struct page` の任意のフィールドに手を伸ばしており、unionを実際に縮小する前にそうした箇所をすべて変換しなければならない。
- 見返り: `struct page` のメモリ税はRAMの約1.6%からその何分の一かへと下がり、型の混乱はなくなり、メモリ管理はより柔軟になる。

遠い最終形ではあるが、具体的なものである。Jonathan Corbetの[Separating memory descriptors from struct page](https://lwn.net/Articles/1073425/)が現状のスナップショットであり、[The proper time to split struct page](https://lwn.net/Articles/937839/)と[Fleshing out memory descriptors](https://lwn.net/Articles/974937/)が設計と議論を扱っている。

---

## モデルを観察する {#observing-the-model}

```bash
# このカーネルはどのメモリモデルでビルドされているか？
zcat /proc/config.gz 2>/dev/null | grep -E "CONFIG_(FLATMEM|SPARSEMEM|SPARSEMEM_VMEMMAP)="
# あるいは、稼働中のディストリビューションカーネルなら:
grep -E "FLATMEM|SPARSEMEM" /boot/config-$(uname -r)

# struct page 税 — RAM を記述するためにどれだけの RAM が使われているか。
# 大まかな見積もり: MemTotal / 4096 * 64 バイト ≈ RAM の 1.6%。
grep MemTotal /proc/meminfo
# 6.11 以降では、page ごとのメタデータが直接計上される:
grep -i memmap /proc/vmstat

# セクションサイズ（SECTION_SIZE_BITS）はコンパイル時のアーキテクチャ定数であり、
# x86-64 では 27（128MB セクション）。

# カーネルプロファイルで pfn_to_page を多用する処理を観察する — vmemmap では
# 単なる加算なので、ほとんど現れない（それが狙いである）:
sudo perf top -e cycles --sort symbol | grep -iE "pfn_to|page_to_pfn"
```

---

## よくある問題 {#common-issues}

**「なぜRAMの約1.5〜2%がブート時に…消えているのか？」** それは `struct page` 配列（とその他のpageごとのメタデータ）である。使用量ではなくRAMの総量に比例し、メモリモデルが払う税である。上述のmemdescの作業が長期的な解決策である。

**巨大なマシンとセクションのオーバーヘッド。** 巨大だが疎な物理マップを持つシステムでは、`SECTION_SIZE_BITS` の選択が粒度（セクションが小さいほど穴での浪費が少ない）とセクション配列のサイズとのトレードオフになる。これはアーキテクチャ/構成の問題であり、手動でチューニングされることはまれである。

**`ZONE_DEVICE` / 永続メモリには `struct page` が必要。** 通常のmmの仕組み（DAX、pmemに対する `get_user_pages`）に参加したいデバイスやpmemの範囲は `struct page` を持たなければならない。つまりデバイス範囲についてvmemmapを用意する必要があり、そのメタデータをデバイスメモリ自体に格納することもある（`altmap`）。これはSPARSEMEM_VMEMMAPでのみ機能する。

---

## バージョンに関する注記 {#version-notes}

| 変更 | Linuxバージョン | ここで重要な理由 |
|--------|--------------|---------------------|
| FLATMEM | 当初から | 元祖の単一配列モデル |
| SPARSEMEM | 2.6.13（2005年） | セクションごとの配列。ホットプラグを可能にした |
| SPARSEMEM_VMEMMAP | 2.6.24（[8f6aac419bd5](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8f6aac419bd5)） | 穴で浪費せずにO(1)の `pfn_to_page()`。今日のデフォルト |
| DISCONTIGMEM削除 | 2021年（最後の利用者は5.11までに消滅） | NUMA時代のモデルが完全に引退 |
| folio | 5.16以降 | 生の `struct page` に代わる、初の正しく型付けされた代替 |
| メモリディスクリプタ（`struct slab`、`ptdesc` など） | 進行中 | `struct page` が8バイトの型付きディスクリプタへと解体されつつある |

---

## 参考文献 {#further-reading}

### カーネルソースとドキュメント {#kernel-source-documentation}

- [Documentation/mm/memory-model.rst](https://docs.kernel.org/mm/memory-model.html) — FLATMEMとSPARSEMEMの公式な説明と、`pfn_to_page()` の定義
- [include/asm-generic/memory_model.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/asm-generic/memory_model.h) — 各モデルの実際の `pfn_to_page()`/`page_to_pfn()` マクロ
- [include/linux/mmzone.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mmzone.h) — `struct mem_section` とセクションの仕組み

### 関連ページ {#related-pages}

- [カーネルアドレス空間](kernel-address-space.md) — `vmemmap` 配列が物理的にどこに置かれ、どうサイズが決まるか
- [メモリホットプラグ](memory-hotplug.md) — 実際に動くSPARSEMEMのセクション: 実行時のメモリの追加と削除
- [pageの一生](life-of-page.md) — このモデル全体がその配列であるところの `struct page`
- [folioの抽象化](folio.md) — `struct page` → memdesc移行の第一歩

### LWNの記事 {#lwn-articles}

- [Memory: the flat, the discontiguous, and the sparse](https://lwn.net/Articles/789304/) — メモリモデルの歴史
- [Separating memory descriptors from struct page](https://lwn.net/Articles/1073425/) — memdesc移行の現状
- [The proper time to split struct page](https://lwn.net/Articles/937839/) — memdescの背後にある設計上の議論
- [Fleshing out memory descriptors](https://lwn.net/Articles/974937/) — 8バイトの型付きディスクリプタがどう機能する想定か
