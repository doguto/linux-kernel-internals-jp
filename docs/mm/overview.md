# メモリ管理: その物語 {#memory-management-the-story}

## 問題（1991年） {#the-problem-1991}

Linus TorvaldsがLinuxを始めたとき、彼は根本的な問いに直面した。**限られたRAMを複数のプログラムでどう共有させるか?**

どのプログラムも、コンピュータのメモリ全体を自分だけのものだと思っている。しかし物理RAMには限りがある。メモリ管理（mm）サブシステムは、2つの重要な仕掛けでこれを解決する。

1. **仮想メモリ**: 各プログラムに自分専用の「偽の」アドレス空間を与える
2. **ページング**: 実際に使われているものだけをRAMに置いておく

このドキュメントは、Linuxのメモリアロケータが30年以上にわたってどう進化してきたかの物語を語る。

---

## 全体像 {#the-big-picture}

Linuxのメモリアロケータが互いにどう関係しているか:

```
                           +---------------------------------------------+
                           |              User Space                     |
                           |  malloc() / mmap() / brk()                  |
                           +-----------------------+---------------------+
                                                   |  system calls
                           +-----------------------v---------------------+
                           |              Kernel                         |
                           |                                             |
  +------------------------+---------------------------------------------+
  |                        |                                             |
  |   +--------------------v--------------------+                        |
  |   |             kmalloc / kfree             | <--- Small objects     |
  |   |         (physically contiguous)         |      (<page size)      |
  |   +--------------------+--------------------+                        |
  |                        |                                             |
  |   +--------------------v--------------------+                        |
  |   |           SLUB Allocator                | <--- Object caches     |
  |   |    (carves pages into fixed objects)    |      (inodes, tasks)   |
  |   +--------------------+--------------------+                        |
  |                        |                                             |
  |   +--------------------v--------------------+   +-----------------+  |
  |   |          Buddy Allocator                |<--|    vmalloc      |  |
  |   |   (manages physical page frames)        |   |  (virtual only) |  |
  |   |        mm/page_alloc.c                  |   |  mm/vmalloc.c   |  |
  |   +--------------------+--------------------+   +--------+--------+  |
  |                        |                                 |           |
  +------------------------+---------------------------------+-----------+
                           |                                 |
                           v                                 v
              +------------------------------------------------------------+
              |                  Physical RAM                              |
              |   [page][page][page][page][page][page][page][page]         |
              |     ^     ^     ^                 ^           ^            |
              |     |     |     |                 |           |            |
              |     +-----+-----+                 +-----------+            |
              |      contiguous                    scattered               |
              |      (kmalloc)                     (vmalloc)               |
              +------------------------------------------------------------+
```

**重要な洞察**: すべてのアロケータは最終的にバディアロケータからpageを得る。違いは、呼び出し元にメモリをどう見せるかにある。

- **kmalloc/SLUB**: 物理的に連続、高速、小さな割り当て
- **vmalloc**: 仮想的に連続、断片化したメモリからでも大きな要求を満たせる

---

## 第1章: ページアロケータ（土台） {#chapter-1-the-page-allocator-the-foundation}

### 元々の問題 {#the-original-problem}

物理メモリは有限である。その塊を配り、そして回収する方法が必要になる。

*注: バディアロケータはLinux 0.01（1991年）にまでさかのぼる。これはLKMLより前のことであり、初期のLinux開発はcomp.os.minixや、よいアーカイブを持たない初期のメーリングリストで行われていた。この時代のmmに関する詳細な議論は、ほとんどが歴史の中に失われている。*

> **歴史的背景: Linuxはどう始まったか**
>
> 1991年8月25日、Linus Torvaldsは[comp.os.minixに投稿した](https://www.cs.cmu.edu/~awb/linux.history.html)。*"I'm doing a (free) operating system (just a hobby, won't be big and professional like gnu) for 386(486) AT clones."*（訳: 386(486) ATクローン向けの（フリーの）オペレーティングシステムを作っている（ただの趣味で、gnuのように大きくてプロフェッショナルなものにはならない）。）彼はヘルシンキの21歳の学生で、MINIX（教育用OS）を自由に改変できないことに不満を抱いていた。Linux 0.01には基本的なメモリ管理、すなわち物理page用の単純なバディアロケータがあった。数か月のうちに、他の人々が貢献し始めた。
>
> **モノリシック対マイクロカーネル論争（1992年）**
>
> 1992年1月、Andrew Tanenbaum（MINIXの作者、OSの教授）はcomp.os.minixで[「Linux is obsolete」（Linuxは時代遅れだ）と宣言した](https://www.oreilly.com/openbook/opensources/book/appa.html)。彼の主張は、Linuxのモノリシックカーネル（mm/、ファイルシステム、ドライバがすべてカーネル空間で動く）は「1970年代への大きな後退」だというものだった。（MINIXのような）マイクロカーネル、つまり最小限のカーネルで、それ以外はすべてユーザー空間に置く方式こそが未来だというのである。
>
> Torvaldsは翌日反論した。モノリシックは実用的な選択だった。マイクロカーネルには理論上の美しさがあるが、コンテキストスイッチとメッセージパッシングによる現実の性能オーバーヘッドがあった。彼はアーキテクチャの純粋さより、実際のハードウェア（386）で物事を動かすことを優先した。歴史は彼が正しかったことを証明した。Linuxは電話からスーパーコンピュータまであらゆる場所で動いている一方、純粋なマイクロカーネルはニッチなままである。
>
> **これがmm/にとってなぜ重要か**: Linuxのモノリシックな設計により、メモリ管理はハードウェアに直接アクセスできるカーネル空間で動作する。メッセージパッシングのオーバーヘッドはない。`kmalloc()`は関数呼び出しであってIPCではない。これが以下で論じるすべてのアロケータを形作った。どれもこの緊密な統合に向けて最適化されている。

### 解決策: バディシステム {#the-solution-buddy-system}

Linuxは（より古いシステムから借用した）「バディアロケータ」を使う。その巧妙なアイデアは次のとおりである。

```
Imagine you have 16 pages of RAM:

[________________]  <- 16 pages (order 4)

Someone asks for 2 pages. You split:

[________][______]  <- two 8-page blocks
[____][__][______]  <- split again: 4, 4, 8
[__][_][_][______]  <- split again: 2, 2, 4, 8
    ^^
    Given to requester

When they free those 2 pages, you merge them back with their "buddy":
[__][_][_] -> [____][_] -> [________] -> etc.
```

**なぜうまくいくのか**: 割り当てが高速（特定のorderのフリーリストから`O(1)`）で、自動的な結合によって断片化が減る。

**できないこと**: 例えば3 pageを渡すこと。4 page（次の2のべき乗）が渡される。

---

## 第2章: slabアロケータ（v2.0、1996年頃） {#chapter-2-the-slab-allocator-v20-1996}

### 問題 {#the-problem}

ページアロケータはpageサイズ（`4KB`）の塊で動作する。しかしカーネルのオブジェクトは小さいことが多い。

- inodeは? 約600バイト
- dentryは? 約200バイト
- ソケットバッファは? 約200バイト

200バイトのオブジェクトのために`4KB`のpageを丸ごと渡すのはメモリの無駄である。

### 解決策: slab {#the-solution-slab}

（SunOSから借用した）「slabアロケータ」は、pageを固定サイズのオブジェクトキャッシュに切り分ける。

```
Page for 200-byte objects:
[obj][obj][obj][obj][obj][obj]...[obj]  <- ~20 objects fit

Page for 600-byte objects:
[object][object][object][object]...     <- ~6 objects fit
```

オブジェクトの種類ごとに専用のキャッシュを持つ。割り当ては高速で（キャッシュから取ってくる）、解放も高速である（キャッシュに返す）。

### 進化 {#the-evolution}

**SLAB（元祖、1996年頃）**: SunOSから移植され、[Bonwickの1994年のUSENIX論文](https://people.eecs.berkeley.edu/~kubitron/courses/cs194-24-S14/hand-outs/bonwick_slab.pdf)に基づく。複雑である。何段階ものキャッシュを持つ。デバッグ機能は優れている。しかし保守が難しい。

*注: SLABはよいLKMLアーカイブより前のものである。Bonwickの論文が標準的な設計の参考文献である。*

**SLUB（v2.6.22、2007年）**: Christoph Lameterによる、より単純な再設計。

- **コミット**: [81819f0fc828](https://git.kernel.org/linus/81819f0fc828)
- 複雑なキューイングを削除
- メモリオーバーヘッドが小さい
- NUMAのサポートが優れている
- 同じAPIで、内部はより単純

*注: コミットメッセージに設計の根拠が記されている。2008年以前のLKMLアーカイブはまばらである。*

**なぜSLUBが勝ったのか**: カーネル開発者は、単純なコードのほうが保守もデバッグも容易だと学んだ。SLABの複雑さは、わずかな性能向上に見合うものではなかった。

**SLOB（組み込み向け、v6.4で削除）**: 小さなシステム向けの最小限のアロケータ。SLUBが組み込みでも十分に効率的になったため削除された。

**API**（`kmalloc`/`kfree`）:
```c
ptr = kmalloc(size, GFP_KERNEL);  // メモリを得る
kfree(ptr);                        // 返す
```

---

## 第3章: vmalloc（1993年 → 現在） {#chapter-3-vmalloc-1993-present}

### 問題 {#the-problem_1}

`kmalloc`は*物理的に連続した*メモリを与える。システムが動き続けると、物理メモリは断片化していく。やがて、空きメモリの合計が`100MB`あっても、連続した`1MB`の塊を見つけられなくなる。

### 解決策: 仮想的な連続性 {#the-solution-virtual-contiguity}

メモリが利用者から見て連続しているように*見え*さえすればよいとしたら?

```
Virtual addresses:     Physical pages:
0xFFFF0000 ──────────→ Page at 0x1234000
0xFFFF1000 ──────────→ Page at 0x8765000  (scattered!)
0xFFFF2000 ──────────→ Page at 0x2222000
```

プログラムからは0xFFFF0000〜0xFFFF3000が連続して見えるが、物理pageは散らばっている。これが`vmalloc()`である。

### コスト {#the-cost}

ページテーブルを設定しなければならない。TLB（変換キャッシュ）のエントリが消費される。kmallocより遅い。しかし、kmallocでは満たせない大きな割り当てを満たすことができる。

### 主要な進化 {#major-evolution}

**v2.6.28（2008年）: スケーラビリティの危機**

**問題**: vunmapはTLBエントリをフラッシュする必要があった。マルチコアシステムでは、これは*すべての*CPUへのIPI（割り込み）を意味した。しかもグローバルロックの下で。コア数が4から64以上に増えるにつれ、これは二次関数的な速度低下になった。

**修正**: Nick Pigginがvmallocをゼロから書き直した。

- **コミット**: [db64fe02258f](https://git.kernel.org/linus/db64fe02258f)
- 遅延TLBフラッシュ: すぐにはフラッシュせず、複数のアンマップをまとめて処理する
- アドレス検索にRBTree: `O(n)`のリスト走査の代わりに`O(log n)`
- CPUごとのフロントエンド: ロック競合を減らす

*注: 遅延TLBフラッシュとRCUについての詳細な設計の根拠は、コミットメッセージに記されている。*

**v5.13（2021年）: ヒュージページ**

**問題**: 巨大なvmallocバッファ（BPFプログラム、モジュール）では、`4KB`のpageごとにTLBエントリが必要だった。TLBには限りがある。TLBミスが多発した。

**修正**: 可能な場合はヒュージページ（`2MB`）を使う。

- **コミット**: [121e6f3258fe](https://git.kernel.org/linus/121e6f3258fe) | [LKML](https://lore.kernel.org/linux-mm/1616036421.amjz2efujj.astroid@bobo.none/)
- 必要なTLBエントリが少なくなる
- トレードオフ: 内部断片化が増える

**v6.12（2024年）: vrealloc**

**問題**: vmallocバッファがあり、それをリサイズしたい。以前は、新しく割り当て、コピーし、古いものを解放しなければならなかった。

**修正**: `vrealloc()`。可能な場合はその場でリサイズする。

- **コミット**: [3ddc2fefe6f3](https://git.kernel.org/linus/3ddc2fefe6f3)
- Rustのアロケータの必要性（`Vec`のリサイズ）が動機
- 物語の全体と見つかったバグについては[vrealloc](vrealloc.md)を参照

*Danilo Krummrichによる[LKMLのパッチシリーズ](https://lore.kernel.org/all/20240722163111.4766-1-dakr@kernel.org/)を参照。*

---

## 第4章: 物事が壊れたとき {#chapter-4-when-things-broke}

現実のバグは完璧なコードより多くを教えてくれる。教訓的な失敗を紹介する。

### KASAN対vrealloc（v6.12） {#kasan-vs-vrealloc-v612}

**何が壊れたか**: vreallocは既存のメモリを再利用したが、KASAN（メモリサニタイザ）の注釈を更新しなかった。その結果、偽陽性の「use-after-free」報告が出た。

**教訓**: メモリ安全性のツールを使っているなら、それを実際のメモリの状態と同期させ続けなければならない。注釈はコメントではなくコードである。

**修正**: [d699440f58ce](https://git.kernel.org/linus/d699440f58ce)

### 戻り値のバグ（v6.13） {#the-return-value-bug-v613}

**何が壊れたか**: vreallocは、その場での拡大をサポートするようにリファクタリングされた。コードはすべての処理を正しく行った……そして間違った変数を返した。拡大する割り当ては、黙って低速パスにフォールバックしていた。

**教訓**: 複雑なリファクタリングの後は、return文を再確認すること。想定した正常系が、実際に実行されるパスとは限らない。

**影響**: BPF検証器の性能のリグレッション。検証器はvreallocを多用している。

**修正**: [f7a35a3c36d1](https://git.kernel.org/linus/f7a35a3c36d1)。1行の変更。

### 冗長なゼロ埋め（v6.13） {#redundant-zeroing-v613}

**何が壊れたか**: `init_on_alloc`が有効なとき、vreallocは拡大時と縮小時の両方でメモリをゼロ埋めしていた。しかし拡大時には、新しいメモリはすでにアロケータによってゼロ埋めされていた。

**教訓**: 上流のコードがすでに何をしたかを理解すること。すべてを自分でやる必要があると思い込まないこと。

**修正**: [70d1eb031a68](https://git.kernel.org/linus/70d1eb031a68)

---

## 第5章: 正しいアロケータを選ぶ {#chapter-5-choosing-the-right-allocator}

これまでの歴史を踏まえた、実践的な指針は次のとおりである。

| 状況 | 使うもの | 理由 |
|-----------|-----|-----|
| 小さなオブジェクト（< page） | `kmalloc()` | 高速、オーバーヘッドが小さい |
| 大きなバッファ（page以上） | `vmalloc()` | 物理的な連続性を必要としない |
| サイズが事前にわからない | `kvmalloc()` | kmallocを試し、vmallocにフォールバックする |
| DMAバッファ | `dma_alloc_coherent()` | ハードウェアのために物理アドレスが必要 |
| リサイズ可能なバッファ | `krealloc()`/`vrealloc()` | 効率的なリサイズ |

---

## 自分で試す {#try-it-yourself}

### バディアロケータを探る {#explore-the-buddy-allocator}

```bash
# ゾーンごとのバディアロケータの状態を見る
cat /proc/buddyinfo

# 出力例:
# Node 0, zone   Normal  1024  512  256  128   64   32   16    8    4    2    1
#                        ^     ^    ^    ^     ^    ^    ^     ^    ^    ^    ^
#                        |     |    |    |     |    |    |     |    |    |    order 10 (4MB blocks)
#                        order 0 (4KB pages)

# 各数値 = そのorderの空きブロック数
# Order 0 = 4KB, Order 1 = 8KB, ... Order 10 = 4MB
```

### slabアロケータを探る {#explore-the-slab-allocator}

```bash
# すべてのslabキャッシュを見る
cat /proc/slabinfo

# より良い方法: slabtopでライブ監視する
slabtop

# 主要な列:
# OBJS     - オブジェクトの総数（アクティブ + 非アクティブ）
# ACTIVE   - 現在使用中のオブジェクト
# USE      - 使用率（パーセント）
# OBJ SIZE - 各オブジェクトのサイズ
# SLABS    - slabの数

# 最大のメモリ消費者を見つける
slabtop -s c  # キャッシュサイズでソート
```

### メモリ全体を確認する {#check-overall-memory}

```bash
# 高レベルのメモリ概要
cat /proc/meminfo

# mm/にとって重要なフィールド:
# MemTotal      - 使用可能なRAMの合計
# MemFree       - まったく使われていないpage
# Buffers       - ブロックデバイスのキャッシュ
# Cached        - ページキャッシュ（ファイルデータ）
# Slab          - カーネルのslabアロケータの合計
# SReclaimable  - 回収可能なslabメモリ
# VmallocUsed   - vmallocの使用量
```

### カーネルのメモリテストを実行する {#run-kernel-memory-tests}

```bash
# テストモジュールをビルドする（CONFIG_TEST_*を有効にしたカーネルツリーで）
make M=lib/ lib/test_vmalloc.ko
make M=mm/ mm/test_hmm.ko

# vmallocのテストを実行する
modprobe test_vmalloc run_test_mask=0xFFFF
dmesg | grep test_vmalloc

# virtme-ng経由で実行する（テストに推奨）
virtme-ng --kdir . --append 'test_vmalloc.run_test_mask=0xFFFF'
```

### 主要なコードの場所 {#key-code-locations}

| サブシステム | ファイル | 見るべきもの |
|-----------|------|------------------|
| バディアロケータ | [`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) | `__alloc_pages()`、`free_pages()` |
| SLUBアロケータ | [`mm/slub.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/slub.c) | `kmem_cache_alloc()`、`kmalloc()` |
| vmalloc | [`mm/vmalloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmalloc.c) | `vmalloc()`、`vfree()` |
| メモリ回収 | [`mm/vmscan.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) | `shrink_node()`、`kswapd()` |
| OOM killer | [`mm/oom_kill.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/oom_kill.c) | `out_of_memory()`、`oom_badness()`。[overcommit](overcommit.md)を参照 |

---

## 年表のまとめ {#timeline-summary}

| 年 | カーネル | 出来事 |
|------|--------|---------------|
| 1991 | 0.01 | page用のバディアロケータ |
| 1996年頃 | 2.0 | 小さなオブジェクト用のSLABアロケータ |
| 1993-2002 | 2.x | vmallocの進化（Linus、その後Christoph Hellwig） |
| 2007 | [v2.6.22](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tag/?h=v2.6.22) | SLUBがSLABに代わってデフォルトに |
| 2008 | [v2.6.28](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tag/?h=v2.6.28) | vmallocの書き直し（RBTree、遅延TLB） |
| 2021 | [v5.13](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tag/?h=v5.13) | vmallocのヒュージページサポート |
| 2023 | [v6.4](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tag/?h=v6.4) | SLOBの削除 |
| 2024 | [v6.12](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tag/?h=v6.12) | vreallocの導入 |
| 2025 | [v6.13](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tag/?h=v6.13)以降 | vreallocのバグ修正、縮小の最適化 |

---

## 参考文献 {#further-reading}

- [vmalloc](vmalloc.md): 仮想メモリ割り当ての詳細
- [vrealloc](vrealloc.md): vmallocのリサイズの物語の全体
- [page-allocator](page-allocator.md): バディシステムの詳細
- [slab](slab.md): SLUBの内部構造
- [glossary](glossary.md): 用語のリファレンス

### 解説記事 {#explainers}

- [virtual-physical-resident](virtual-physical-resident.md): VSZ対RSS対PSSの解説
- [overcommit](overcommit.md): なぜオーバーコミットが唯一の正気なデフォルトなのか
- [brk-vs-mmap](brk-vs-mmap.md): カーネルからメモリを得る2つの方法
- [contiguous-memory](contiguous-memory.md): なぜ空きメモリがあるのに大きな割り当てが失敗するのか
- [cow](cow.md): コピーオンライトのエッジケースと、それがいつ破れるか
