# ユーザーページの取得（GUP） {#getting-user-pages-gup}

> `get_user_pages()`対`pin_user_pages()`、FOLL_LONGTERM、fast GUP、そしてなぜピン留めがコピーオンライトやページマイグレーションと衝突するのか

カーネル（あるいはその背後にあるデバイス）が、プロセスのページテーブルを経由するのではなく物理的な実体として*ユーザー*メモリのpageに触れる必要があるとき、カーネルは**get_user_pages**ファミリー（一般に「GUP」と呼ばれる）を呼び出す。これはメモリ管理サブシステムで最も影響が大きく、かつ最もバグを生みやすいインターフェースの1つである。物理pageという安定した世界と、ユーザー仮想メモリという移ろう世界がちょうど出会う場所に位置しており、その契約を誤ったことがLinuxの最も有名な脆弱性のいくつかを生んできた。

このページは*なぜ*を扱う。なぜGUPが存在しなければならないのか、なぜほとんど同じに見える2つのAPIに分割されたのか、そしてなぜ「pageをピン留めするだけ」がカーネルの交わす約束の中で最も難しいものの1つになるのか。コピーオンライトへの影響については特に[コピーオンライト](cow.md)と[CVE事例研究](war-stories-cves.md)を、高速パスの性能面については[io_uringの固定バッファ](../io-uring/fixed-buffers.md)を読んでほしい。

---

## GUPが解決する問題 {#the-problem-gup-solves}

ユーザー仮想アドレスは*借地権*であって、*権利証*ではない。それが指すpageは、ほとんどいつでも次のような状態になりうる。

- **まだ存在しない**: 一度もフォールトインされていないか、スワップへ回収された（[ページフォールトハンドラ](page-fault.md)）。
- **再配置される**: コンパクション、NUMAバランシング、CMAによって別の物理フレームへ移動される（[コンパクション](compaction.md)）。
- **共有かつ読み取り専用**: 次の書き込みで複製されるコピーオンライトのpage（[コピーオンライト](cow.md)）。

この不安定さは、CPUが自分のページテーブルを通して自分のメモリに触れる分には問題ない。MMUとフォールトハンドラが透過的に取り繕ってくれるからだ。しかし、**物理アドレスによって、所有プロセスのページテーブルウォークの外側から**pageに到達する必要がある3つの重要な利用者にとっては問題になる。

1. **ハードウェアDMA**: 物理（またはIOMMU）アドレスをプログラムされたNIC、GPU、NVMeコントローラは、そのフレームを直接読み書きする。転送の途中でカーネルがそれをマイグレーションしたり解放したりすると、デバイスはその時点でそこにあるものを何であれ破壊してしまう。
2. **カーネルが別のアドレス空間にアクセスする場合**: `process_vm_readv()`、ptrace、共有マッピング上のfutex、`vmsplice()`。
3. **ダイレクトI/O**: ブロック層がページキャッシュを迂回してユーザーバッファへ直接DMAする。

GUPはその橋渡しである。`mm_struct`とユーザー仮想アドレスの範囲を渡すと、そのプロセスのページテーブルをウォークし（**必要ならpageをフォールトインし**）、各アドレスを`struct page`に解決し、pageをその場に留めておく参照を取得して、`struct page *`の配列を返す。呼び出し元は、参照を解放するまで、DMAしたり`kmap()`したりできる物理的に安定したハンドルを手にする。

```c
/* 概形 — mm/gup.c */
long pin_user_pages(unsigned long start, unsigned long nr_pages,
                    unsigned int gup_flags, struct page **pages);
/* ... ページテーブルをウォークし、欠けているものをフォールトインし、pageごとに参照を取る ... */
/* 後で: */
unpin_user_pages(pages, nr_pages);
```

呼び出し元が指定する`gup_flags`（`FOLL_WRITE`、`FOLL_LONGTERM`、…）は意図を表し、後述の歴史が示すように、このインターフェースの安全性はすべて、その意図を正しく表明できるかどうかにかかっている。（`FOLL_PIN`自体は`pin_user_pages*()`ラッパーが内部で設定するもので、呼び出し元が渡すものではない。）

---

## getとpin: 同じに見えて正反対の意味を持つ2つのAPI {#get-vs-pin-two-apis-that-look-identical-and-mean-opposite-things}

Linux 5.6以降、互換性があるように見える2つのファミリーが存在する。

| | `get_user_pages()`（FOLL_GET） | `pin_user_pages()`（FOLL_PIN） |
|---|---|---|
| **区別の要点: 呼び出し元はpageの*データ*にアクセスするか?** | **しない**: `struct page`のメタデータを操作するだけで、それが管理するメモリには触れない | **する**: pageの実際の内容を読み書きする（例: DMA） |
| 取得される参照 | 通常のpage参照カウント+1 | ベースpage: 参照カウント += `GUP_PIN_COUNTING_BIAS`（1024）。大きなfolioは別個の`_pincount`を使う |
| DMAピンとして検出可能か? | 不可: 他の参照と区別できない | 可能: `folio_maybe_dma_pinned()` |
| 解放方法 | `put_page()` | `unpin_user_pages()` |

主軸は**期間ではなくデータアクセス**である。カーネルのドキュメントは、「FOLL_GET is for `struct page` manipulation, without affecting the data」（FOLL_GETはデータに影響を与えない`struct page`の操作用）である一方、FOLL_PINは「the caller will access the page data」（呼び出し元がpageのデータにアクセスする）場合のためのものだと明言している。（ピンを*どれだけ長く*保持するかは別の軸であり、後述の`FOLL_LONGTERM`が担う。）この分割は、微妙で高くついた教訓から生まれた。すなわち、**通常の参照カウントでは、pageがDMAに使われていることをメモリ管理のコアに伝えられない**。参照カウントが例えば3であっても、それは3つの一時的な参照なのか、乱してはならない進行中のDMA転送1つなのか区別がつかない。「このpageをマイグレーションしても安全か、デバイスがまだ書き込んでいるかもしれない間にこのダーティなfolioをライトバックしてよいか」という判断を下す必要のあるコードには、それを問う手段がなかった。

John Hubbardによる`FOLL_PIN`の再設計（[LWN: Explicit pinning of user-space pages](https://lwn.net/Articles/807108/)）は、DMAの意図を*page自体の中に見える*ようにする。[カーネルのドキュメント](https://docs.kernel.org/core-api/pin_user_pages.html)によれば、その仕掛けはカウントにバイアスをかけることである。ピンは参照カウントに1ではなく`GUP_PIN_COUNTING_BIAS`（1024）を加えるので、次のようになる。

```c
/* 「このpageはDMAのためにピン留めされているか?」に答えられるようになる */
if (folio_maybe_dma_pinned(folio))
        /* マイグレーションしてはならない。ライトバックで書き込み手が終わると仮定してはならない */
```

このバイアス方式は実用本位のハックである。巨大な複合pageではカウントが共有されていたため、1 GBのpageは数回ピン留めしただけでオーバーフローしかねず、それが大きなfolioに*別個の*`_pincount`フィールドが追加された理由である。しかし重要なのは概念的な部分だ。**以前は文脈が暗に示すだけだった契約を、今やAPIが担うようになった。** 指針は率直である。pageのデータに触れる新しいコードは`pin_user_pages()`を使うべきであり、`get_user_pages()`は`struct page`のメタデータを操作するだけの場合のためのレガシーである。

---

## FOLL_LONGTERM: メモリ管理全体と衝突するピン {#foll_longterm-the-pin-that-fights-the-whole-memory-manager}

短いピンは厄介ごとにすぎないが、*長い*ピンは構造的な問題である。`FOLL_LONGTERM`は、数秒、数分、あるいはRDMA登録の存続期間中ずっと続く可能性のあるピンを示す。そして、いつまでも移動できないpageは、どのpageも再配置できるという前提で設計された2つのサブシステムと対立する。

- **コンパクション / 断片化防止**: カーネルは移動可能なpageをマイグレーションして寄せ集めることで、物理的に連続した空き領域を確保する。長期ピンはpageをその場に釘付けにし、その周囲の領域を恒久的に断片化させる（[コンパクション](compaction.md)）。
- **CMAと`ZONE_MOVABLE`**: これらの領域は、誰かが連続メモリを必要としたときにそのpageをいつでもマイグレーションで退避できるようにすることを*特に目的として*存在する。そこでピン留めされたpageは、その領域が存在する理由そのものを無にしてしまう（[CMA](cma.md)）。

カーネルの答えは**ピン留め前のマイグレーション**である。`FOLL_LONGTERM`の要求は、まず対象pageを移動可能なゾーンやCMA領域から通常のメモリへ*移し出そう*とし、その後でピン留めする。マイグレーションに失敗すれば、ピン留めも失敗する（[LWN: Preserving the mobility of ZONE_MOVABLE](https://lwn.net/Articles/843326/)、[prohibit pinning pages in ZONE_MOVABLE](https://lwn.net/Articles/843143/)）。典型的なCMA割り当て失敗が「ここにあるpageが`get_user_pages()`によってピン留めされている」となるのはこのためである。デバイスドライバのDMAバッファがCMAのpageを人質に取っているのだ。この緊張関係は根本的なものである。**移動可能性と長期ピン留めは構造上両立しない**のであり、カーネルはpageごとにどちらを優先するかを選ぶことしかできない。

---

## Fast GUP: ロックなしでページテーブルをウォークする {#fast-gup-walking-page-tables-without-the-locks}

素直なGUPパスは`mmap_lock`を取得し、その下でページテーブルをウォークする。毎秒数百万回のピン留めを行う高IOPSのダイレクトI/Oやio_uringのワークロードにとって、そのロックと完全なウォークは純粋なオーバーヘッドである（[io_uringの固定バッファ](../io-uring/fixed-buffers.md)はまさにそれを償却して消すために存在する）。そこで**ロックレスの高速パス**、`get_user_pages_fast()` / `pin_user_pages_fast()`が存在する。これは2.6.27でのNick Pigginによる元々のx86向けの作業である（[LWN: Toward better direct I/O scalability](https://lwn.net/Articles/275808/)）。

- **`mmap_lock`を取得せずに**ページテーブルをウォークし、代わりにfast GUPのウォーカーが**割り込み禁止**で動作するという事実に依拠する。x86では、これによりページテーブルの解放時にブロードキャストされるTLBシュートダウンのIPIがブロックされるため、ウォークの最中にテーブルが消えることはない。ハードウェアによるTLBブロードキャストを持つアーキテクチャでは、代わりにページテーブルを**RCUで解放**し、割り込み禁止の区間がRCUの読み取り側クリティカルセクションとして機能する（[LWN: a general RCU get_user_pages_fast](https://lwn.net/Articles/616211/)。[メモリ管理におけるRCU](rcu-mm.md)、`follow_page_mask()`、`mm/gup.c`の`gup_fast`の仕組みを参照）。
- 参照カウントを投機的に増やしてからPTEを**再検証**する。何かが変わっていれば撤退し、ロックを取る低速パスにフォールバックする。

これはロックレス読み取りの教科書的なパターンである。楽観的に進め、後で検証し、衝突時にはフォールバックする。そしてこれが、GUPがコピーオンライトと非常に微妙な相互作用をする理由でもある。高速パスのウォーカーとフォールト中の書き込み手が、まさに同じPTEをめぐって競合しうるのだ。

---

## なぜGUPはバグを呼び寄せるのか: コピーオンライトとの衝突 {#why-gup-is-a-bug-magnet-the-copy-on-write-collision}

悪名高いGUPのバグのほとんどすべては同じ形をしている。**あるpageがある目的のためにピン留めされている間に、コピーオンライトがそのpageの*意味*を変えてしまった**のだ。代表的な例を2つ挙げる（詳細は[コピーオンライト](cow.md)と[CVE事例研究](war-stories-cves.md)で述べている）。

- **Dirty COW（CVE-2016-5195）**: 11年間潜んでいた競合で、`__get_user_pages()`が失敗した書き込みフォールトを`FOLL_WRITE`を再度表明*せずに*リトライしたため、COWのコピーではなく元の読み取り専用pageを返してしまい、特権のないプロセスが読み取り専用ファイルに書き込めた。[19be0eaffa3a](https://git.kernel.org/linus/19be0eaffa3a)で修正され、すべてのリトライにわたって書き込みの意図が保持されるようになった。
- **vmsplice/COWのリーク**: GUPで共有COWのpageをピン留めした子プロセスが、その後親プロセスにそこへ書き込ませると、自身のPTEをアンマップした後でも、古いピンを通じて親の後のデータを観測できてしまう（[LWN: Patching until the COWs come home](https://lwn.net/Articles/849638/)）。

根本的な修正はパッチではなくアーキテクチャによるものだった。`FOLL_PIN`の分割と`PG_anon_exclusive`は、COWの仕組みが長期ピンを*認識*し、そもそもピン留めされたpageを共有しないようにするために存在する。GUPはまさにこの種のバグの罠として評判を得たのであり、再設計の主題全体は**契約を文脈から推論させるのではなくAPIで明示する**ことにある。失敗し続けたのは推論だったからだ。

---

## 呼び出し元の分類 {#the-caller-taxonomy}

[カーネルのドキュメント](https://docs.kernel.org/core-api/pin_user_pages.html)はGUPの呼び出し元をいくつかのケースに分類しており、これが正しい呼び出しを選ぶ最も手早い方法である。

| ケース | 呼び出し元 | 正しい呼び出し |
|------|--------|--------------|
| 1 | **ダイレクトI/O**: 短命なバッファ、DMAはすぐに完了する | `pin_user_pages_fast()` |
| 2 | **RDMA / 長命なDMA**: バッファが無期限に登録される | `pin_user_pages(..., FOLL_LONGTERM)` |
| 3 | **MMU notifierの利用者**: マッピングを追跡し、無効化時に再取得する | ピン留めは不要なことが多い |
| 4 | **`struct page`の操作のみ**: pageの*データ*には一切触れない | `get_user_pages()` |
| 5 | **pageのデータへの書き込み** | `pin_user_pages()` |

`FOLL_PIN`を自分で渡すことは決してない。これはGUPの内部のものであり、「should not appear at the gup call sites」（GUPの呼び出し箇所に現れるべきではない）。`pin_user_pages*()`ラッパーを選べば、それが`FOLL_PIN`を設定してくれる。ドキュメントが推す経験則は次のとおりだ。*データ*にアクセスするならピンを使い、アクセスがシステムコールより長く続くなら`FOLL_LONGTERM`を追加して、先にマイグレーションするコストを受け入れる。

---

## ピンの観測 {#observing-pins}

```bash
# VmPin = 移動不可としてピン留めされたpage。長期ピンを計上するドライバ
# （RDMA、VFIO、io_uringの登録バッファ）はここでpinned_vmに課金する。
# 注意: その場限りのget_user_pages()によるピンは体系的には公開されない —
# 明示的に計上する呼び出し元だけが現れる。
grep -i "VmLck\|VmPin" /proc/$$/status

# デバイスドライバのDMAピン留めをライブで観察する（VFIO、RDMA、io_uring）:
sudo bpftrace -e 'kprobe:pin_user_pages_remote,
                  kprobe:pin_user_pages_fast { @[probe, comm] = count(); }'

# ダイレクトI/O / io_uringワークロードのカーネルCPUプロファイルでGUPを見る —
# アプリが登録/固定バッファを使っていなければ、get_user_pages_fast / unpin_user_pages
# が目立って現れる:
sudo perf top -e cycles --sort comm,symbol | grep -iE "user_pages"

# 領域内のpageがピン留めされているためにCMA割り当てが失敗している?
dmesg | grep -iE "alloc_contig|cma:.*fail|migrat"
```

---

## よくある問題 {#common-issues}

**メモリ逼迫時に「CMA allocation failed」が出る。** ほぼ常に、CMA領域内のpageが処理中の`get_user_pages()`/DMAによってピン留めされており、移し出せないことが原因である。修正はチューニングではなく、ドライバ側の根本的な対処（バウンスバッファを使う、あるいはCMAのpageを長期ピン留めしない）である。

**空きメモリがあるのにRDMA/VFIOの登録が`-ENOMEM`で失敗する。** `FOLL_LONGTERM`のピン留め前マイグレーションがpageを`ZONE_MOVABLE`の外へ再配置できなかったか、`RLIMIT_MEMLOCK`に達している。長期ピンは回収不能であるからこそ、ロック済みメモリの上限に計上される。

**fork後にデバイスが古い/ゴミのデータを読む。** ピンとCOWの古典的な相互作用である。`fork()`の前にピン留めされたpageは、COWが働き始めると子プロセスから見えるものと食い違いうる。現代のカーネルはこれを緩和している。ピン留めされた無名pageはfork時にCOWで共有されるのではなく*即座に*コピーされ（Peter Xuによるピン留めpageの早期COW、5.9）、5.19の`PG_anon_exclusive`がGUPとCOWの関係を全般的に信頼できるものにした。しかし根底にある危険性があるため、長期のDMAバッファはこの点を念頭に置いて設定すべきである。

---

## バージョンに関する注記 {#version-notes}

| 変更 | Linuxバージョン | ここで重要な理由 |
|--------|--------------|---------------------|
| `get_user_pages_fast()`のロックレスパス | 2.6.27（[8174c430e445](https://git.kernel.org/linus/8174c430e445)） | 高スループットのDMA/ダイレクトI/O向け高速パス |
| Dirty COWの修正（書き込みの意図をリトライ間で保持） | 4.8.3 / 4.9（[19be0eaffa3a](https://git.kernel.org/linus/19be0eaffa3a)） | CVE-2016-5195を塞いだ（2016年10月マージ） |
| `pin_user_pages()` + `FOLL_PIN` | 5.6（[eddb1c228f79](https://git.kernel.org/linus/eddb1c228f79)） | ピンのバイアスによりDMAの意図が見えるようになった |
| DMA用に`pin_user_pages_remote()`が`get_user_pages_remote()`を置き換え | 5.6 | VFIO/RDMAがピンAPIへ移行 |
| `ZONE_MOVABLE`での長期ピンを禁止、先にマイグレーション | 5.11–5.12 | CMA/移動可能ゾーンの移動可能性を保つ |
| fork時のピン留めpageの早期COW | 5.9 | ピン留めされた無名pageを共有せず即座にコピー |
| `PG_anon_exclusive`: COWとピンの信頼できる関係 | 5.19（[LWN](https://lwn.net/Articles/887178/)） | GUP/COW戦争のアーキテクチャ上の決着 |

---

## 参考文献 {#further-reading}

### カーネルソースとドキュメント {#kernel-source-documentation}

- [Documentation/core-api/pin_user_pages.rst](https://docs.kernel.org/core-api/pin_user_pages.html): 権威ある契約。FOLL_GET対FOLL_PIN、呼び出し元のケース、カウントのバイアス
- [mm/gup.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/gup.c): `get_user_pages*`、`pin_user_pages*`、`follow_page_mask()`、そして`gup_fast`のロックレスウォーク
- [include/linux/mm.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mm.h): `FOLL_*`フラグの定義

### 関連ページ {#related-pages}

- [コピーオンライト](cow.md): GUP/COWの相互作用の全体像と`PG_anon_exclusive`
- [CVE事例研究](war-stories-cves.md): インシデントの歴史としてのDirty COWとFOLL_PINの再設計
- [io_uringの固定バッファ](../io-uring/fixed-buffers.md): 一度ピン留めしてI/OごとのGUPコストを消す
- [VFIOの内部構造](../iommu/vfio-internals.md): デバイスDMAのための`pin_user_pages_remote()`
- [CMA](cma.md): なぜピン留めされたpageがCMA割り当て失敗の最大の原因なのか
- [メモリ管理におけるRCU](rcu-mm.md): fast GUPが依拠するロックレスなページテーブルウォーク
- [カーネルアドレス空間](kernel-address-space.md): GUPが返すpageが実際にどこにあるか（ダイレクトマップ）

### LWNの記事 {#lwn-articles}

- [Explicit pinning of user-space pages](https://lwn.net/Articles/807108/): John Hubbardが`pin_user_pages()`とカウントのバイアスを紹介
- [Patching until the COWs come home (part 1)](https://lwn.net/Articles/849638/): Vlastimil Babkaによるピン対COWのデータリーク類
- [Preserving the mobility of ZONE_MOVABLE](https://lwn.net/Articles/843326/): ピン留め前のマイグレーションと、なぜ長期ピンが移動可能ゾーンを脅かすのか
- [Reliable GUP pins of anonymous pages](https://lwn.net/Articles/887178/): David Hildenbrandによる`PG_anon_exclusive`とGUP/COW戦争の終結
- [Toward better direct I/O scalability](https://lwn.net/Articles/275808/): ロックレスな`get_user_pages_fast()`の2008年の起源
- [A general RCU get_user_pages_fast](https://lwn.net/Articles/616211/): IPIによるTLBシュートダウンを持たないアーキテクチャのためのRCU解放されたページテーブル
