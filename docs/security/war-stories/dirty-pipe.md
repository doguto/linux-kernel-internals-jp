# Dirty Pipe {#dirty-pipe}

> CVE-2022-0847 — `buf->flags = 0;`が1行欠けていたために、ファイルを*読める*ユーザーなら誰でも、イミュータブルなファイル、読み取り専用のbtrfsスナップショット、読み取り専用マウントを含め、そのファイルをページキャッシュ上で上書きできた

公開日
:   2022年3月7日（修正のマージは2022年2月22日）

報告者
:   Max Kellermann, CM4all GmbH / IONOS SE

CVSS
:   7.8 HIGH（`AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H` — NVDの主スコア。[access.redhat.com](https://access.redhat.com/security/cve/cve-2022-0847)によれば、Red Hat独自のCNA評価も一致している）

バグの存在期間
:   4.9以降（2016年）

悪用可能になった時期
:   5.8以降（2020年）

修正バージョン
:   5.16.11、5.15.25、5.10.102（いずれも2022年2月23日リリース）

エクスプロイトツール
:   あり — 報告者が開示文書そのものの中で動作するPoCを公開した

実際の悪用
:   あり — 2022年4月25日にCISAのKEVカタログに追加（[NVD](https://nvd.nist.gov/vuln/detail/CVE-2022-0847)による）

*[事件簿: LinuxセキュリティのバグとCVE](../war-stories.md)の一部。*

## 何が起きたか {#what-happened}

欠けていた初期化 — `buf->flags = 0;` — は、パイプバッファを手作業で埋める`lib/iov_iter.c`の2つの関数の中で4年間眠っていた。その後、無関係で十分に考え抜かれたリファクタリングによって、それはカーネル史上最も深刻なローカル権限昇格バグの1つ、CVE-2022-0847になった。ケーパビリティも書き込み権限も一切なしに悪用可能なバグである。

これを見つけたのはホスティング企業のエンジニアで、カーネルのバグを探していたのではなく、顧客のログファイルで起きていたありふれたデータ破損に見えるものを調査している最中だった — そして仕組みを理解してしまえば、それを動作するエクスプロイトに仕立てるのは、決まった順序の5つのシステムコールで済み、競合も特権も必要なかった。

## どのように発見されたか {#how-it-was-found}

1年近くの間、誰もこれがセキュリティバグだとは知らなかった — 最初のサポートチケットは2021年4月29日に届き、破損がカーネルのバグだと特定されたのは2022年2月19日のことだった。その前段の謎について、Kellermann自身の説明はお決まりのハードウェアの容疑者を順にたどっている: "Is this caused by flaky hardware? Bad RAM? Bad storage? Cosmic rays?"（訳: 不安定なハードウェアのせいか？ 壊れたRAM？ 壊れたストレージ？ 宇宙線？）

CM4allのホスティングプラットフォームは、日々のWebアクセスログをzlibで圧縮し、1か月分を1つの連結された`.gz`として — Windowsユーザー向けにはZIPコンテナに包んで — 提供しており、`splice()`を使ってファイルデータを直接HTTP接続に送り込んでいた。2021年4月から、ダウンロードしたログがCRC検証に失敗するという報告が顧客から時折寄せられるようになった。最初のチケットは、CRCを手作業で修正することで解決扱いになった。

破損には特徴があった。影響を受けたすべてのファイルの最後の8バイトが同じ誤ったバイト列に置き換わっており、16進ダンプの末尾にそれが見える:

```
000005f0  81 d6 94 39 8a 05 b0 ed  e9 c0 fd 07 00 00 ff ff
00000600  03 00 50 4b 01 02 1e 03  14 00
```

`50 4b`は`PK`である。`01 02`はZIPのセントラルディレクトリのファイルヘッダである。ログファイルを上書きしていたバイト列は、*Webサービス*がすべての日次ファイルをspliceした後に自分のパイプに書き込んだZIPヘッダだった — 別のユーザーとして動作し、それらのファイルへの書き込み権限を持たず、そもそも書き込み用に開いたことすらないプロセスである。

ディスク全体をスキャンすると、3か月で37個の破損ファイルが見つかり、それらは各月の最終日に強く集中していた — ダウンロードのループは日付順にファイルを送るので、最終日のファイルは常に`PK`ヘッダの直前に来るからである。影響を受けたのは実際にHTTPダウンロードを提供しているサーバーだけで、同じログ分割ジョブを実行していた待機系のサーバーでは破損はゼロだった。

Kellermannの最小再現コードは2つの小さなプログラムである。1つはファイルに`write(1, "AAAAA", 5)`をループで書き込み、もう1つはそのファイルからパイプへの`splice()`と、それに続く`write(1, "BBBBB", 5)`をループで実行する。するとファイルの中に"BBBBB"が現れ始めた。v4.19からv5.10までの185,011個のコミットにわたるbisect — git bisectにかかればおよそ17ステップに縮まる — は`f6dd975583bd`にたどり着いた。

当初の見立ては、並行して書き込む特権を持つプロセスが必要で、競合に勝つことに依存するというものだった。実際の仕組みを理解すると、穴は一気に大きく広がった。書き込み手も競合も不要で、攻撃者が読めるあらゆるファイルの、ほぼ任意のオフセットに任意のデータを書ける。`/etc/passwd`、setuidバイナリ、`authorized_keys` — 何でもである。

## 影響 {#the-impact}

上書きのある性質が、運用面でこれを特にたちの悪いものにしている。ページをdirtyにしないので、ライトバックが一切行われないのである。変更はそのファイルを読むすべてのプロセスに対して有効だが、ディスクに一度も触れることなく、再起動やメモリ回収で消えてしまう。Kellermannの控えめな注記 — "This allows interesting attacks without leaving a trace on hard disk"（訳: これにより、ハードディスクに痕跡を残さない興味深い攻撃が可能になる）— は、ファイルシステムに何のフォレンジック上の痕跡も残さないインシデントを表している。

実環境での悪用はすぐに続き、よく文書化されている:

- **CISA**は**2022年4月25日**にCVE-2022-0847をKnown Exploited Vulnerabilitiesカタログに追加した。[NVD](https://nvd.nist.gov/vuln/detail/CVE-2022-0847)で確認でき、開示から7週間後のことで、連邦機関向けの対応期限は2022年5月16日だった。
- **Googleの[Android Security Bulletin（2022年5月）](https://source.android.com/docs/security/bulletin/2022-05-01)**はCVE-2022-0847（コンポーネント: `pipes`、EoP、High、バグA-220741611）を掲載し、"There are indications that the following may be under limited, targeted exploitation."（訳: 以下は限定的・標的型の悪用を受けている可能性を示す兆候がある。）の下に挙げている。Kellermannは報告の翌日にGoogle Pixel 6でこのバグを再現していた。
- **Exploit-DB**にはエントリ[50808](https://www.exploit-db.com/exploits/50808)、"Linux Kernel 5.8 < 5.16.11 - Local Privilege Escalation (DirtyPipe)"が、公開開示の翌日の2022年3月8日に掲載されている。NVDの参考資料リストはさらに3件のPacket Stormのエントリに`Exploit`のタグを付けており、うち1件はSUIDバイナリ乗っ取りの変種、残り2件は同じバグの汎用的なローカル権限昇格の解説である。

大まかな類例はCVE-2016-5195、Dirty COWであり、Kellermannも名前の中でそれに言及している — しかし彼はこちらのほうが"is easier to exploit"（悪用が容易だ）と明言しており、実際そのとおりである。Dirty COWは競合の窓を必要としたが、こちらは決まった順序の5つのシステムコールで済む。

## コードはどのように脆弱だったか {#how-the-code-was-vulnerable}

その穴がどうしてこれらすべてを生んだのかを見るには、パイプが実際にどう動作するかから始める。Linuxのパイプは`struct pipe_buffer`エントリのリングであり、各エントリが1つのpageを指している。リングの仕組みについては[パイプとFIFO](../../ipc/pipes.md)を、`splice()`がコピーせずにpageをそのリングに移す方法については[splice、sendfile、ゼロコピー](../../io/splice-sendfile.md)を参照のこと。

関係する最適化は**マージ**である。パイプへの直前の書き込みがそのpageを埋めきっていなければ、次の`write()`は新しいpageを割り当てる代わりに同じpageに追記できる。これは*匿名*パイプバッファについては安全である。そのpageはパイプが完全に所有しているからだ。しかしファイルから`splice()`で作られたバッファについては断じて安全ではない。それらは**ページキャッシュ**を直接指しているからである — そのpageはパイプではなくファイルのものであり、それに追記することはファイルに書き込むことになる。

そのため、カーネルには常に「このバッファにマージしてよいか？」というチェックがあった。年月を経て変わったのは、*そのチェックがどこから答えを読むか*である:

- **2.6.17（2006年）** — [`5274f052e7b3`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5274f052e7b3dbd81935772eb551dfd0325dfa9d)が`splice()`を導入し、同時に`page_cache_pipe_buf_ops`を導入した。これは`can_merge = 0`を持つ最初の`struct pipe_buf_operations`である。答えは**opsベクタ**にあった。
- **5.1（2019年）** — [`01e7187b4119`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=01e7187b41191376cee8bea8de9f907b001e87b4)（"pipe: stop using ->can_merge"、Jann Horn、Al Viroの提案による）はこのフラグを廃止し、ポインタ比較に置き換えた。それを設定していたopsベクタは1つしかなかったからである。これはHornが見つけて[2018年10月に報告した](https://lore.kernel.org/linux-fsdevel/CAG48ez1wN=oC_uWkHHhboDvfVt8p9O98ZMFZyh=AK6D=eHU7MA@mail.gmail.com/)本物のバグにさかのぼる。部分的なpageのバッファを持つパイプから`tee()`すると、2つの独立したパイプが同じpageを共有し — そして破損させ — うるというものだった。彼の[最初の修正の試み](https://lore.kernel.org/linux-fsdevel/20181015150420.2096-1-jannh@google.com/)はEric Biggersによるレビューを一巡し、その3か月後に[改訂版v2](https://lore.kernel.org/linux-fsdevel/20190123141918.238286-2-jannh@google.com/)が取り込まれた。答えは依然として**opsポインタ**から得られていた:

    ```c
    static bool pipe_buf_can_merge(struct pipe_buffer *buf)
    {
        return buf->ops == &anon_pipe_buf_ops;
    }
    ```

- **5.8（2020年）** — [`f6dd975583bd`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f6dd975583bd8ce088400648fd9819e4691c8958)（"pipe: merge anon_pipe_buf*_ops"、Christoph Hellwig）は、`anon_pipe_buf_ops`、`anon_pipe_buf_nomerge_ops`、`packet_pipe_buf_ops`がバイト単位で同一の関数テーブルを持ち、振る舞いを符号化するためだけに存在していることに気づいた。3つのうち2つを削除し、答えを**バッファごとのフラグ**に移した:

    ```c
    #define PIPE_BUF_FLAG_CAN_MERGE	0x10	/* バッファをマージできる */
    ```

    ```c
    /* fs/pipe.c、f6dd975583bd以降のpipe_write() */
    if ((buf->flags & PIPE_BUF_FLAG_CAN_MERGE) &&
        offset + chars <= PAGE_SIZE) {
    ```

Hellwigのコミットメッセージは、古い`nomerge`の特殊ケースの意味を反転させることが"actually allows for much nicer code"（実際にずっと良いコードにできる）と述べており、実際そうなった — diffは48行を削除し11行を追加している。

これは単独の投稿ではなく、カーネルの通常のメンテナツリー経由でメインラインに到達した。[Al Viroの"assorted splice cleanups"プルリクエスト](https://lore.kernel.org/lkml/20200603192615.GY23230@ZenIV.linux.org.uk/)に含まれる7つのパッチの1つで、メーリングリスト上でパッチごとに議論されたのではなく、サブシステムのメンテナとしてのViroによってレビューされた。そのスレッドの唯一の返信は自動のマージトラッカーによる受領通知である。

それとは別に — そしてそれより前、2016年に — まったく別のファイルに実際のバグが入り込んでいた。[`241699cd72a8`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=241699cd72a8489c9446ae3910ddd243e9b9061b)（"new iov_iter flavour: pipe-backed"、Al Viro、4.9、2016年）が`lib/iov_iter.c`に2つの関数を追加しており、それらはリングから新しい`pipe_buffer`を割り当てて手作業で埋めていた:

```c
/* copy_page_to_iter_pipe()、lib/iov_iter.c、修正前、2022年時点の状態 */
buf->ops = &page_cache_pipe_buf_ops;
get_page(page);
buf->page = page;
buf->offset = offset;
buf->len = bytes;
```

`ops`、`page`、`offset`、`len` — すべて設定されている。`flags` — 一度も触れられておらず、そのため、そのリングスロットを*以前に*使っていたものが残した値をそのまま保持していた。

このパッチは公の場で、しかも注意深くレビューされていた。linux-fsdevelへの[2回目の投稿](https://lore.kernel.org/linux-fsdevel/20160924040117.GP2356@ZenIV.linux.org.uk/)で、Miklos Szeredi — 後述するように、5か月後に同じ種類のバグを他の2つの呼び出し元で修正することになる人物 — は次のように書いた: "This is the hardest part of the whole set. I've been trying to understand it, but the modular arithmetic makes it really tricky to read... Specific comments inline"（訳: これはシリーズ全体で最も難しい部分だ。理解しようとしてきたが、剰余演算のせいで本当に読みにくい……具体的なコメントはインラインで）。そして4つの抜粋を引用してインラインでコメントしたが、その大半は無関係な点についてだった — `copy_page_to_iter_pipe()`と`push_pipe()`における`EFAULT`処理の欠如と予期しない0の戻り値、`pipe_advance()`の無駄な代入、`iov_iter_npages()`におけるpageの勘定である。

脆弱な関数から取られた2つの抜粋は、どちらもフィールドごとの代入ブロックのちょうど1文手前で終わっている（それぞれ`pipe->nrbufs++;`が間に入る）ため、そのブロックと、そこから欠けている`flags`の行は、スレッド内の引用コードのどこにも一度も現れなかった。スレッドの中に、そのブロックがそれ自体として精査されるべきだったことを示唆するものは何もない — 単に話題に上らなかったのである。

2016年から2020年まで、これは潜在的で無害なバグだった。当時存在していたフラグ（`PIPE_BUF_FLAG_LRU`、`_ATOMIC`、`_GIFT`、`_PACKET`）はどれも、Kellermannの言葉では"rather boring"（かなり退屈）なものであり、マージの判定は初期化*されていた*`ops`から読まれていた。初期化されていないフィールドは、まだ単に問題にならなかったのである。

## トリガー {#the-trigger}

`f6dd975583bd`は、マージの判定を、この2つの関数が書き忘れていたまさにそのフィールドに移すことで、それを問題にした。再利用されたリングスロットから引き継がれた古い`PIPE_BUF_FLAG_CAN_MERGE`ビットが、今や`pipe_write()`に対して、ページキャッシュのpageに追記しても安全だと告げるようになった。

[開示文書](https://dirtypipe.cm4all.com/)で全文が公開されたKellermannの概念実証は5ステップで、ケーパビリティは一切必要ない:

1. パイプを作成する。
2. 任意のデータでパイプを完全に満たす — これでリング内のすべての`pipe_buffer`に`PIPE_BUF_FLAG_CAN_MERGE`が設定される。通常の匿名書き込みパスがそうするからである。
3. パイプを空にする。バッファは解放されるが、リングスロットは`flags`の値を保持したままである。
4. 対象ファイル — `O_RDONLY`で開いたもの — から、上書きしたいオフセットの直前の位置で1バイトをパイプに`splice()`する。`copy_page_to_iter_pipe()`はページキャッシュへの参照を埋め、古いマージビットをそのまま残す。
5. データをパイプに`write()`する。新しいバッファを割り当てる代わりに、ページキャッシュのpageにマージされる。

`open()`呼び出しに付けられたエクスプロイト自身のコメント — `open(path, O_RDONLY); // yes, read-only! :-)` — は、この脆弱性全体を1行で表している。実際の制約は3つある。オフセットはpage境界上には置けない（その1バイト前をspliceしなければならないため）、書き込みはpage境界をまたげない（残りは新しい匿名バッファに入ってしまうため）、そしてファイルを拡張できない（パイプはファイルが大きくなったことをページキャッシュに決して伝えないため）。PoC自身のコメントは最初の2つを"major limitations"（主要な制限）として挙げており、3つ目は明示的な"cannot enlarge the file"（ファイルを拡張できない）のチェックで強制している。Kellermannの開示文書はそれを他の制約と並べて挙げている。

Kellermannの言葉では、これは"not only works without write permissions, it also works with immutable files, on read-only btrfs snapshots and on read-only mounts (including CD-ROM mounts). That is because the page cache is always writable (by the kernel), and writing to a pipe never checks any permissions."（訳: 書き込み権限なしで動作するだけでなく、イミュータブルなファイル、読み取り専用のbtrfsスナップショット、読み取り専用マウント（CD-ROMマウントを含む）でも動作する。ページキャッシュは（カーネルにとっては）常に書き込み可能であり、パイプへの書き込みは一切の権限チェックを行わないからだ。）

## なぜ気づかれなかったか {#why-it-went-unnoticed}

直接の原因は2つの欠けた初期化である。興味深い原因は、*なぜ6年間誰もそれに気づかなかったのか*である。

**安全性チェックが、常に初期化されるフィールドから、そうでないフィールドへ移った。** マージチェックのどのバージョンも、書かれた時点では正しかった。`copy_page_to_iter_pipe()`と`push_pipe()`は常に`buf->ops`を正しく設定していたので、答えが`ops`から導かれている限り、それらは問題なかった — 偶然に。`f6dd975583bd`は、マージの判定を代わりに`flags`から読むようにした、クリーンで十分に考え抜かれたデッドコードの整理だった。`lib/iov_iter.c`の2つの関数 — 別のファイル、別のサブシステム、4年前に別の作者によって追加されたもの — がたまたま`pipe_buffer`を手作業で埋めており、そのメンバー1つを飛ばしていたことを、このコミットが知り得たはずはない。KellermannはPoC自身のコメントで明確に述べている: "The commit did not introduce the bug, it was there before, it just provided an easy way to exploit it."（訳: このコミットがバグを持ち込んだのではない。バグはそれ以前からあり、このコミットはそれを悪用する簡単な方法を提供しただけだ。）

**極性がフェイルクローズからフェイルオープンに反転した。** 古い方式では、未設定やゴミの`flags`値は無害だった。マージには`ops == &anon_pipe_buf_ops`という肯定的な一致が必要だったからである。新しい方式では、マージはビットが*設定されている*ことによって許可されるので、たまたま`0x10`を含むゴミ値が許可を与えてしまう。0の値が「拒否」を意味する構造体フィールドはずさんな初期化を許容するが、設定された値が「許可」を意味するものは許容しない。リファクタリングのどこにも、重要な呼び出し箇所でこの違いを見えるようにするものはなかった。

**構造体をフィールドごとに手作業で埋めることには、コンパイラによる歯止めがない。** 脆弱な2つの関数はどちらも、指示付き初期化子やmemsetを使わずにメンバーを個別に代入しているので、省略されたメンバーは何の警告も生まない — 再利用されたリングスロットが保持していたものを黙って引き継ぐ。`struct pipe_buffer`は意図的に再利用される長寿命のリングの中にあり、それが「未初期化」を「ランダム」ではなく「攻撃者の影響下にある」ものに変える。

**まさにこの種類のバグは、5年前に2つの兄弟の呼び出し元ですでに表面化し、修正されていた — しかも2つ目の修正は1つ目の数分以内、同じ日に取り込まれていた。** 2017年2月、Miklos Szerediは[`84588a93d097`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=84588a93d097bace24b9233930f82511d4f34210)をコミットした。そのタイトルは — 一字一句 — "fuse: fix uninitialized flags in pipe_buffer"である。その1行のdiffは、`fuse_dev_splice_read()`の中の`bufs[page_nr].ops = ...`の代入の直後に`bufs[page_nr].flags = 0;`を追加するもので、その`Fixes:`タグは[`d82718e348fe`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d82718e348fee15dbce8f578ff2588982b7cc7ca)（"fuse_dev_splice_read(): switch to add_to_pipe()"）を指している。これは`lib/iov_iter.c`の箇所を持ち込んだのと同じ、4.9時代のpipe-backed iov_iterへの変換の一部である。fuseの修正は単独の投稿にはならず — Szerediが同じ日に送った[プルリクエスト](https://lore.kernel.org/linux-fsdevel/20170216164335.GB30656@veci.piliscsaba.szeredi.hu/)の中に含まれてLinusに届いた。

その5分後、[Szerediは2つ目の修正を投稿した](https://lore.kernel.org/linux-fsdevel/20170216164902.GC30656@veci.piliscsaba.szeredi.hu/)。`splice_to_pipe()`に対するほぼ同一の1行修正で、そのコミットメッセージは、未初期化のflagsが"appears to have been there from the introduction of the splice syscall"（spliceシステムコールの導入時から存在していたようだ）— つまり2006年から — と記している。同じパターンの2つの呼び出し元が同じ日に監査され修正されたが、同じ2016年の変換で書かれた`lib/iov_iter.c`の2つはそうならなかった。当時はそれは何の違いも生まなかった。3年後、それがすべての違いを生んだ。

## どのように修正されたか {#how-it-was-fixed}

修正[`9d2231c5d74e`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9d2231c5d74e13b2a0546fee6737ee4446017903)（"lib/iov_iter: initialize \"flags\" in new pipe_buffer"、Max Kellermann）は2行である:

```diff
@@ -414,6 +414,7 @@ static size_t copy_page_to_iter_pipe(struct page *page, size_t offset, size_t by
 	buf->ops = &page_cache_pipe_buf_ops;
+	buf->flags = 0;
 	get_page(page);
@@ -577,6 +578,7 @@ static size_t push_pipe(struct iov_iter *i, size_t size,
 		buf->ops = &default_pipe_buf_ops;
+		buf->flags = 0;
 		buf->page = page;
```

このコミットには`Fixes: 241699cd72a8 ("new iov_iter flavour: pipe-backed")`と`Cc: stable@vger.kernel.org`が付いており、この欠落を、それを武器化した5.8のリファクタリングではなく、2016年のコミットに正しく帰属させている。

**公開メーリングリスト上では議論はなかった — 意図的に。** [LKMLのスレッド](https://lore.kernel.org/lkml/20220221100313.1504449-1-max.kellermann@ionos.com/)はちょうど2通のメッセージしかない。Kellermannは2022年2月21日にパッチを投稿し、その本文は"The functions `copy_page_to_iter_pipe()` and `push_pipe()` can both allocate a new `pipe_buffer`, but the 'flags' member initializer is missing."（訳: 関数`copy_page_to_iter_pipe()`と`push_pipe()`はどちらも新しい`pipe_buffer`を割り当てうるが、'flags'メンバーの初期化が欠けている。）とだけ述べていた。5時間後、Al Viroの返信は次がすべてだった:

> Applied, will push to Linus...

（訳: 適用した。Linusにpushする……）

その簡潔さこそが物語であって、物語の不在ではない。ここから先の部分は公開のLKMLアーカイブには一切なく — [Kellermann自身の開示文書](https://dirtypipe.cm4all.com/)の時系列による。バグ報告、エクスプロイト、パッチは2月20日に非公開の`security@kernel.org`リストに送られ、翌日のLKMLへの投稿は**"(without vulnerability details) as suggested by Linus Torvalds, Willy Tarreau and Al Viro"**（脆弱性の詳細なしで、Linus Torvalds、Willy Tarreau、Al Viroの提案どおりに）行われた — 些細な初期化漏れの整理に見えるパッチとして、それ自体の妥当性によってマージされ、悪用可能性についての議論は修正が出荷されるまで公開アーカイブから遠ざけられた。このCVEについての設計上の議論をloreで探す人は何も見つけられないだろうが、それは[カーネルのセキュリティバグ対応プロセス](https://docs.kernel.org/process/security-bugs.html)が意図した結果であって、記録の欠落ではない。

Kellermannの開示文書による、残りの時系列:

| 日付 | 出来事 |
| --- | --- |
| 2021-04-29 | 破損したログファイルについての最初のサポートチケット |
| 2022-02-19 | 破損がカーネルのバグであり、悪用可能であると特定される |
| 2022-02-20 | 報告、エクスプロイト、パッチを`security@kernel.org`に送付 |
| 2022-02-21 | Google Pixel 6で再現。Androidセキュリティチームに報告 |
| 2022-02-21 | 脆弱性の詳細なしでパッチをLKMLに投稿。Al Viroが適用 |
| 2022-02-23 | stableリリース[5.16.11](https://lore.kernel.org/stable/1645618039140207@kroah.com/)、[5.15.25](https://lore.kernel.org/stable/164561803311588@kroah.com/)、5.10.102が修正を含めて出荷 |
| 2022-02-24 | GoogleがAndroid共通カーネルに修正をマージ |
| 2022-02-28 | `linux-distros`に通知 |
| 2022-03-07 | 公開開示 |

非公開の報告から修正済みstableカーネルの出荷まで3日、公開開示まで15日。修正は振る舞い上のリスクのない2行の初期化なので、取り消すべき後続のリグレッションはなかった — 修正パッチ自体に修正が要らなかった珍しいケースである。

意味のある実行時の緩和策はない。`splice()`と`pipe()`は普通のソフトウェアが使う非特権の基本的なシステムコールであり、このバグはケーパビリティもnamespaceも必要としない。[seccomp BPF](../seccomp.md)でシステムコールをブロックしても、それを選択したサンドボックス化されたプロセスしか守れない。パッチ適用が唯一の現実的な答えであり、だからこそCISAはこれに3週間という連邦機関向けの期限を付けたのである。

```bash
# 脆弱な範囲は5.8から5.16.10 / 5.15.24 / 5.10.101まで。
# ディストリビューションのカーネルはバックポートするので、アップストリームのバージョンだけでは決定的でない —
# ベンダーの変更履歴でCVE IDを確認すること。
uname -r
```

## 得られた教訓 {#what-it-taught-us}

**判定が答えを読む場所をリファクタリングすると、どの初期化が決定的に重要かの範囲が黙って変わる。** `f6dd975583bd`は良い整理だった。同一の2つのopsベクタを削除し、コードを短く明快にした。その唯一の欠陥はそれ自身のdiffからは見えなかった — マージの判定を`ops`から`flags`に移したことで、別のサブシステムで長く眠っていた初期化漏れが権限昇格に格上げされたのである。チェックが参照する構造体メンバーが変わるとき、その構造体を手作業で構築するすべての箇所が新たに関係するようになる。リファクタリングが一切触れなかった箇所も含めて。

**ビットが設定されていると許可を与えるフラグは、ビットが設定されていると拒否するフラグよりもはるかに寛容でない。** 14年間、マージの判定はフェイルクローズだった。追記するには肯定的な一致が必要だった。`PIPE_BUF_FLAG_CAN_MERGE`はそれを反転させ、意味の反転は「これを0にし忘れた」を見た目だけの欠陥から認可の回避へと変える。再利用される、カーネルが所有する構造体のケーパビリティビットは拒否として表現するか、あるいは安全な状態がデフォルトになるよう、構造体を割り当て時に0にすべきである。

**「今日は無害」は現在のコードベースについての主張であって、コードそのものについての主張ではない。** 初期化漏れは2016年から2020年まで本当に何の影響もなく、当時そのように扱ったのは妥当だった。しかし潜在的なバグは朽ちていかない — その深刻度は、それを見たこともない人々が何年も後に書くコードによって決まる。2017年のfuseの修正は、この種類のバグが既知であり、1つの呼び出し元では修正されたにもかかわらず、同じ変換で追加された兄弟たちは監査されないままだったことを示している。未初期化フィールドのバグを修正するときは、クラッシュを報告した1箇所ではなく、その構造体を構築するすべての箇所を監査することこそが修正である。

**CVSS基本値は攻撃ベクタの記述であって、被害範囲の記述ではない。** 7.8 HIGHは正しい — ローカルの非特権アカウントが必要なので、`AV:L`と`PR:L`がそこを上限にする。しかしその制約の範囲内では、このバグは本質的に無制限である。ケーパビリティも、namespaceも、競合も、タイミングも不要で、完全に公開された5システムコールのエクスプロイトがあり、読み取り専用マウント、イミュータブルなファイル、読み取り専用スナップショットを打ち破る。CISAは実際に悪用されているものとして掲載し、Googleは数週間以内にAndroidでの標的型の悪用を指摘した。スコアだけで優先順位を付けると、これはまったく似ていない数多くのローカルバグと同じ順位になってしまう。

!!! warning "注意すべきパターン"
    長寿命で再利用されるプール — パイプバッファのリング、skbの制御ブロック、リクエスト構造体、per-CPUキャッシュ — から取り出したカーネル構造体を、メンバーを1つずつ代入して構築するコードはすべて監査すること。そこで省略されたメンバーは0としてではなく、直前の利用者が残したものとして読まれる。そして攻撃者が直前の利用者を制御していたなら、「未初期化」の値は攻撃者が選んだものになる。フィールドごとの代入よりも、指示付き初期化子（`*buf = (struct pipe_buffer){ ... };`）や明示的なmemsetを好むこと。そして`Fixes:`タグの付いた未初期化フィールドのパッチはすべて、閉じたチケットとしてではなく、兄弟の呼び出し箇所で同じパターンをgrepするきっかけとして扱うこと。

## 関連項目 {#see-also}

- [Linuxケーパビリティ](../capabilities.md) — このバグが完全に迂回した特権モデル。大半のローカル権限昇格と異なり、Dirty Pipeはケーパビリティを一切必要とせず、読み取りアクセスだけで済んだ
- [カーネルハードニング](../kernel-hardening.md) — メモリ破壊を必要としないバグに対して、エクスプロイト緩和策（KASLR、スタックプロテクタ）が何の役にも立たない理由
- [seccomp BPF](../seccomp.md) — `splice()`ベースの攻撃に対する唯一の実用的な封じ込め手段。ただしそれを選択したプロセスに対してのみ
- [Linux Security Module（LSM）フレームワーク](../lsm.md) — アクセス制御フックが何の異議も唱えなかった理由。エクスプロイト中のすべてのシステムコール（読み取り専用の`open()`、`splice()`、パイプへの`write()`）は個々には正当である
- [パイプとFIFO](../../ipc/pipes.md) — このバグの中心にある`pipe_buffer`のリングとマージ最適化
- [splice、sendfile、ゼロコピー](../../io/splice-sendfile.md) — `splice()`がページキャッシュへの参照をどのようにパイプに入れるか
- [ページキャッシュ](../../mm/page-cache.md) — dirtyにされない上書きされたpageが、なぜディスク上に痕跡を残さないのか

## 外部参考資料 {#external-references}

- [The Dirty Pipe Vulnerability](https://dirtypipe.cm4all.com/) — Max Kellermannの開示文書。仕組み、1年にわたる破損の調査、概念実証エクスプロイト、時系列についての一次資料
- [NVD: CVE-2022-0847](https://nvd.nist.gov/vuln/detail/CVE-2022-0847) — CVEレコード、CVSS 7.8 HIGH（`CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`）、2022年3月10日公開。CISA KEVへの掲載（2022年4月25日追加、対応期限2022年5月16日）を確認できる
- [git.kernel.org: 9d2231c5d74e](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9d2231c5d74e13b2a0546fee6737ee4446017903) — "lib/iov_iter: initialize \"flags\" in new pipe_buffer"。2行の修正
- [lore.kernel.org: LKMLのパッチスレッド](https://lore.kernel.org/lkml/20220221100313.1504449-1-max.kellermann@ionos.com/) — 公の議論のすべて。Kellermannのパッチと、Al Viroの"Applied, will push to Linus..."
- [git.kernel.org: 241699cd72a8](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=241699cd72a8489c9446ae3910ddd243e9b9061b) — "new iov_iter flavour: pipe-backed"（4.9、2016年）。修正の`Fixes:`タグが発端として名指ししているコミット
- [lore.kernel.org: new iov_iter flavour: pipe-backed](https://lore.kernel.org/linux-fsdevel/20160924040117.GP2356@ZenIV.linux.org.uk/) — 2016年のレビュースレッド。初期化が欠けていることが後に判明した2つの関数に対する、Miklos Szerediのインラインレビューを含む
- [git.kernel.org: f6dd975583bd](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f6dd975583bd8ce088400648fd9819e4691c8958) — "pipe: merge anon_pipe_buf*_ops"（5.8、2020年）。マージチェックを`flags`に移し、眠っていたバグを悪用可能にした
- [lore.kernel.org: Al Viroの"assorted splice cleanups"プルリクエスト](https://lore.kernel.org/lkml/20200603192615.GY23230@ZenIV.linux.org.uk/) — `f6dd975583bd`がメインラインに到達した経路。アーカイブにはこのパッチの個別の投稿は見当たらない
- [git.kernel.org: 01e7187b4119](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=01e7187b41191376cee8bea8de9f907b001e87b4) — "pipe: stop using ->can_merge"（5.1、2019年）。opsのフラグをポインタ比較に置き換えた中間のリファクタリング
- [lore.kernel.org: sys_tee()のバグ報告](https://lore.kernel.org/linux-fsdevel/CAG48ez1wN=oC_uWkHHhboDvfVt8p9O98ZMFZyh=AK6D=eHU7MA@mail.gmail.com/)と[v2の修正](https://lore.kernel.org/linux-fsdevel/20190123141918.238286-2-jannh@google.com/) — Jann Hornによる元のバグ報告と、それから生まれたパッチ
- [git.kernel.org: 84588a93d097](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=84588a93d097bace24b9233930f82511d4f34210) — "fuse: fix uninitialized flags in pipe_buffer"（2017年）。同じ種類のバグが5年前に兄弟の呼び出し元で見つかり修正されたもの
- [lore.kernel.org: vfs: fix uninitialized flags in splice_to_pipe()](https://lore.kernel.org/linux-fsdevel/20170216164902.GC30656@veci.piliscsaba.szeredi.hu/) — fuseの修正を含むプルリクエストの5分後にSzerediが投稿した、ほぼ同一の2つ目の修正
- [Android Security Bulletin（2022年5月）](https://source.android.com/docs/security/bulletin/2022-05-01) — CVE-2022-0847（`pipes`、EoP、High）を掲載し、"limited, targeted exploitation"（限定的・標的型の悪用）の兆候に言及している
- [Exploit-DB 50808](https://www.exploit-db.com/exploits/50808) — "Linux Kernel 5.8 < 5.16.11 - Local Privilege Escalation (DirtyPipe)"、2022年3月8日公開
- [lore.kernel.org: Linux 5.16.11](https://lore.kernel.org/stable/1645618039140207@kroah.com/)と[Linux 5.15.25](https://lore.kernel.org/stable/164561803311588@kroah.com/) — 修正を含むGreg Kroah-Hartmanのstableリリースのアナウンス、2022年2月23日
- [Red Hat Bugzilla 2060795](https://bugzilla.redhat.com/show_bug.cgi?id=2060795) — CNAのトラッキングバグと影響分析
- [kernel.org: Security bugs](https://docs.kernel.org/process/security-bugs.html) — 修正が出荷されるまで悪用可能性の議論を公開リストから遠ざけた`security@kernel.org`のプロセス
