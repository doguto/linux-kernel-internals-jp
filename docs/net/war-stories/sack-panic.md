# TCP SACK Panic {#the-tcp-sack-panic}

> CVE-2019-11477 / CVE-2019-11478 / CVE-2019-11479 — リモートのピアが、注意深く細工したSACKブロックの列によって、任意のLinux TCPエンドポイントをクラッシュさせることができた

公開日
:   2019年6月17日

報告者
:   Jonathan Looney（Netflix）

CVSS
:   7.5 HIGH（3件とも）

修正
:   4.4.182、4.9.182、4.14.127、4.19.52、5.1.11

エクスプロイトツール
:   公開されたものはない — 細工したパケット列そのものが攻撃のすべてである

実際の悪用
:   確認された事例はない（CISA KEVには未掲載）

*[事件簿: ネットワークスタックのバグとCVE](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

RFC 2018で定義されたTCPの選択的確認応答（SACK）オプションは、送信側に最初の欠落以降のすべてを再送させる代わりに、受信側が順序外で受信したバイト範囲を正確に送信側へ伝えられるようにする。これを効率的に実装するため、Linuxの再送キューと順序外キューはデータを`struct sk_buff`（SKB）の連鎖で保持し、何千もの小さなバッファ断片を抱え込まないよう、SACK情報が届くたびに隣接するSKBを積極的に**結合（coalesce）**する。

各SKBは`struct tcp_skb_cb`制御ブロック（`include/net/tcp.h`）を持ち、そこではとりわけ`tcp_gso_segs`、つまりGeneric Segmentation Offload（GSO）の計上のためにそのSKBが表すMSSサイズのセグメント数が追跡される。このフィールドは**16ビットの符号なし整数**であり、単一のSKBが65,535を超えるセグメントを表さない限り問題はない。SKBは最大17個のフラグメント（`MAX_SKB_FRAGS`）を保持でき、各フラグメントはx86では最大32 KB（64 KBページを使う一部のPowerPC構成では64 KB）なので、最大限詰め込まれたSKBは500 KBを超えるペイロードを保持できる。MSSが非常に小さければ、それは膨大なセグメント数になる。

## きっかけ {#the-trigger}

LinuxはMSSの下限を強制しているが、脆弱なコードの時点では、その下限は`net/ipv4/tcp_output.c`の`__tcp_mtu_to_mss()`で48バイトにハードコードされていた。そしてTCPオプションがそのうち最大40バイトを消費しうるため、**セグメントあたりの実ペイロードはわずか8バイト**しか残らないことがある。NetflixのJonathan Looneyは、リモートのピアがこの最小のMSSを広告したうえで、注意深く細工したSACKブロックの列を送れることを発見した。これは3つのCVEすべての一次資料である[Netflix自身のアドバイザリ](https://github.com/Netflix/security-bulletins/blob/master/advisories/third-party/2019-001.md)（NFLX-2019-001）として公開された。SACK処理により、`net/ipv4/tcp_input.c`の`tcp_shift_skb_data()`が`skb_shift()`を通じて再送キューのSKBを結合するため、攻撃者は1つのSKBに蓄積されたセグメント数を65,535より先に押し上げ、16ビットの`tcp_gso_segs`フィールドを黙ってオーバーフローさせることができた。

## 観測された挙動 {#observed-behavior}

このオーバーフローにより、`tcp_shifted_skb()`は次の行に到達した:

```c
BUG_ON(tcp_skb_pcount(skb) < pcount);
```

`BUG_ON()`はカーネルパニックを引き起こす。リモートの、認証されていないピアが、SACKが有効な（デフォルトで有効）任意のLinux TCPエンドポイントをクラッシュさせることができた。これがCVE-2019-11477であり、3つの中で最も深刻なものである。関連する2つのバグが併せて報告された:

- **CVE-2019-11478**: パニックに到達しなくても、攻撃者は細工したSACKの列を使って`tcp_fragment()`（`net/ipv4/tcp_output.c`）に再送キューを非常に多数の小さなSKBへ分割させ、接続ごとのメモリ使用量を膨らませることができた。
- **CVE-2019-11479**: 最小MSSが48バイトにハードコードされていたため、攻撃者はSACKの小細工すら必要としなかった。単に非常に小さなMSSを広告するだけで、カーネルに、まともな下限なら許さないほど多くのバイトあたりの作業（より多くのヘッダ、より多くのSKB、より多くのCPU）を強いることができた。

LWNの記事（["The TCP SACK panic"、2019年6月](https://lwn.net/Articles/791409/)）は、これが「NetflixのJonathan Looneyが発見した4つのSACK関連バグの中で最も深刻なもの」であると指摘した（4つ目の関連バグはLinuxではなくFreeBSDのRACKスタックに影響するものだった）。

## なぜ起きたか {#why-it-happened}

3つの独立した弱点が組み合わさった:

1. `tcp_gso_segs`は`u16`であり、65,535という固い上限があるのに、結合の前にそれを強制するものが何もなかった。
2. `tcp_shift_skb_data()`は、結果のセグメント数をチェックせずに`skb_shift()`を直接呼び出していたため、結合によってフィールドが黙ってオーバーフローしえた。
3. MSSの下限（48バイト）が十分に低かったため、現実的な量の再送キューデータで65,535セグメントのSKBに到達でき、さらに`tcp_fragment()`には、ソケットバッファのサイズに対してキューを何回分割できるかの上限がなかった。

`tcp_shifted_skb()`の`BUG_ON()`は、セグメント数が不整合になることは決してないという前提で書かれた防御的なアサーションだった。リモートのピアがその前提の誤りを証明したのである。

このバグの影響範囲は、誰もセキュリティ上の性質として追跡していなかった形でも広がっていた。LWNの記事の下のコメントで、カーネル開発者のRichard Weinbergerは、古いカーネルでは`BUG_ON()`に到達できるのは、そのフローでGSO/GROオフロードが実際にネゴシエートされた場合だけだったと指摘した。彼の読みでは、それは[`0a6b2a1dc2a2`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0a6b2a1dc2a2105f178255fe495eb914b09cb37a)（"tcp: switch to GSO being always on"、2018年2月）で変わった。これもEric Dumazetが作成したもので、無関係な理由、つまりOleksandr Natalenkoが報告したBBRのペーシングの性能問題を修正するために、GSOを日和見的なオフロードではなくすべてのTCPソケットで必須にする目的でマージされた。その変更はセキュリティとは何の関係もなかったが、Weinbergerの説明によれば、SACK panicの被害範囲を「GSO/GROを使うフロー」から「すべてのLinux TCPエンドポイント」へと広げた。

同じコメントスレッドで、XFSのメンテナであるDave Chinner（"dgc"）は、このバグの形をはるかに古い事件から思い出した:「I remember back in late 2002 when a bug report for an Irix NFS server performance issue was nailed down to a serious SACK problem due to really small MSS windows being sent from a buggy NFS client implementation. The phrase 'SACK panic' triggered my memory immediately…」（訳: 2002年の終わり頃、Irix NFSサーバーの性能問題のバグ報告が、バグのあるNFSクライアント実装から送られてくる非常に小さなMSSウィンドウによる深刻なSACKの問題に突き止められたことを覚えている。「SACK panic」という言葉を見てすぐに記憶がよみがえった…）Chinnerのより大きな論点は、非難ではなく組織の記憶についてだった:「the OS networking community knew about these problems 15 years ago but that knowledge seems to have been lost and so we have repeated past mistakes. Which raises some interesting questions: where did that institutional knowledge go?」（訳: OSのネットワーキングコミュニティは15年前にこれらの問題を知っていたが、その知識は失われたようで、我々は過去の過ちを繰り返した。これは興味深い問いを投げかける。あの組織的な知識はどこへ行ったのか？）

## 解決 {#resolution}

3つの別々のコミットが3つのCVEに対処した:

- **CVE-2019-11477** — [`3b4929f65b0d`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3b4929f65b0d8249f19a50245cd88ed1a2f78cff)（"tcp: limit payload size of sacked skbs"）は、`skb_shift()`のラッパーである`tcp_skb_shift()`を導入する。これは、結果として`to->len`が`65535 * TCP_MIN_GSO_SIZE`バイトを超える場合、または`tcp_skb_pcount()`が65,535セグメントを超える場合にシフトを拒否する。また`BUG_ON()`を`WARN_ON_ONCE()`に格下げし、不整合があってもマシンがパニックしないようにする。
- **CVE-2019-11478** — [`f070ef2ac667`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f070ef2ac66716357066b683fb0baf55f8191a2e)（"tcp: tcp_fragment() should apply sane memory limits"）は、`sk_wmem_queued`が`sk_sndbuf`の2倍を超えたら（`(sk->sk_wmem_queued >> 1) > sk->sk_sndbuf`）、`tcp_fragment()`がパケットの分割を拒否するようにし、代わりに`-ENOMEM`を返して新しい`TCPWqueueTooBig` SNMPカウンタをインクリメントする。
- **CVE-2019-11479** — [`967c05aee439`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=967c05aee439e6e5d7d805e195b3a20ef5c433d6)は`tcp_mtu_probing()`で新しい`net.ipv4.tcp_min_snd_mss` sysctlを強制し、対になるコミット[`5f3e2bf008c2`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5f3e2bf008c2221478101ee72f5cb4654b9fc363)（"tcp: add tcp_min_snd_mss sysctl"）がsysctl自体を追加する。互換性のためにデフォルトは48のままだが、管理者はこれを引き上げられるようになった。

これらのパッチは公開の一環として2019年6月17日に[netdevに投稿され](https://lore.kernel.org/netdev/20190617170354.37770-1-edumazet@google.com/)、CVE-2019-11478の修正は直ちに実際のリグレッションを表面化させた。Christoph Paaschは、以前は通っていたpacketdrillのテストが無期限にストールするようになったと報告した。`SO_SNDBUF`を人為的に低く強制した接続は、`sk_wmem_queued`が新しい上限を超えると、再送キューをまったく分割できなくなったためである。

Eric Dumazetの最初の反応は「I guess it is WAI :)」（訳: 意図どおりの動作だと思う :)）というものだったが、それでも再送キューが空のとき（`!tcp_rtx_queue_empty(sk)`）は新しいチェックをスキップする簡易的なガードを提示し、Paaschはそれでテストが直ることを確認した。しかしPaaschは未解決の問いも提起した。`sk_wmem_queued`が十分に大きくなり、正当な再送すら分割できなくなった場合、接続が恒久的に固まってしまうことはないのか？ 当時のDumazetの答えは率直だった:「Really TCP can not work well with tiny sndbuf limits. There is really no point trying to be nice.」（訳: 本当に、TCPは極小のsndbuf制限ではうまく動かない。親切にしようとする意味は本当にない。）

メールで送られたパッチは一時しのぎにすぎなかった。4日後に実際に出荷された[`b6653b3629e5`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b6653b3629e5b88202be3c9abc44713973f5c4b4)（"tcp: refine memory limit test in tcp_fragment()"）は、より精密な条件を使った。再送キューが空かどうかをチェックするのではなく、すでに書き込みキューにあるskbの分割（`tcp_queue != TCP_FRAG_IN_WRITE_QUEUE`）を免除したのである。その理由は、`tcp_sendmsg()`が`SO_SNDBUF`を強制するのは64&nbsp;KBのskb境界でだけなので、アプリケーションが作った最初のskbが正当にもう1回分割を必要とすることがある、というものだった。

それでもまだ終わりではなかった。3週間後、MIT Lincoln Laboratory Supercomputing CenterのAndrew Proutは、Paaschが予言していた事態に遭遇した。ノンブロッキングの10&nbsp;Gbitファブリック上のTCP接続、つまりSamba越しに大きなファイルを取得するVPNユーザーや、TCP/IP上のMPIジョブが、恒久的にストールして二度と回復しなかったのである。bisectでCVE-2019-11478の修正に行き着き、`TCPWqueueTooBig`カウンタの増加によって確認された。

netdevで、Christoph Paaschは、`tcp_fragment()`が特に再送タイマーから呼ばれた場合には常に割り当てを許可することを提案した。元の攻撃は正当な再送ではなく偽造されたSACKブロックに依存していた、という理由である。Dumazetは常時許可を、同じ穴を大規模に再び開くものとして却下し（"Anything we add in TCP stack to overcome the SO_SNDBUF by twice the limit _will_ be exploited at scale"、訳: SO_SNDBUFを上限の2倍まで超えるためにTCPスタックに何かを追加すれば、それは大規模に悪用される _ことになる_）、代わりに`tcp_retrans_collapse`を1から3に引き上げて、よりよい畳み込みでキューを縮小する案を出した。

実際に出荷された[`b617158dc096`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b617158dc096709d8600c53b6052144d12b89fab)（"tcp: be more careful in tcp_fragment()"、2019年7月）は、折衷案を取った。メモリ上限に達していても、再送キューの最初と最後のskbは分割できるようにする。これは上限を全体的に撤廃することなく、ストールした接続を解放するのに十分である。さらに、`tcp_sendmsg()`/`tcp_sendpage()`が正当に生み出しうる最大64&nbsp;KBの超過分のために、明示的な余裕を加える。これでMIT Lincoln Labの報告は解決した。

その2か月後、Databricksは同じ修正に反対側からぶつかった[経緯を公開した](https://www.databricks.com/blog/2019/09/16/adventures-in-the-tcp-stack-performance-regressions-vulnerability-fixes.html)。AWS上でS3に書き込むSparkジョブが6倍遅くなったのである。原因は、`sk_sndbuf`が46,080バイトなのに`sk_wmem_queued`が正当に103,971まで増えていた接続だった。56&nbsp;KBの差があったのは、`tcp_sendmsg()`/`tcp_sendpage()`が`SO_SNDBUF`を64&nbsp;KBのskb境界でしか強制しないためであり、まさに`b617158dc096`がすでに許容するよう書かれていた超過分だった。彼らの記事は、すでに出荷された修正の根拠を裏付けるものであり、新たなカーネル作業を促した独立の報告ではない。

元の3つの修正はすべて、2019年6月17日に4.4.182、4.9.182、4.14.127、4.19.52、5.1.11のstableリリースで出荷され、`tcp_fragment()`に対する2つの追加の改良はその後1か月の間に入った。すぐにアップグレードできない運用者には暫定的な緩和策があった。CVE-2019-11477/78を鈍らせるためにMTUプロービングを無効にし（`net.ipv4.tcp_mtu_probing=0`）、SACKを無効にする（`net.ipv4.tcp_sack=0`）こと、あるいはCVE-2019-11479に対してファイアウォールで不当に小さな広告MSS値をフィルタすることである。

これについては、公開されたエクスプロイトツールもMetasploitモジュールも存在しない。リモートのカーネルパニックにはそれが必要ないからである。Netflixが自身のアドバイザリで記録した細工SACKの列が「エクスプロイト」のすべてである。その影響範囲の本当の尺度はベンダーの対応である。NVDには25を超える下流のアドバイザリが掲載されており、Cisco、Oracle、VMware、F5、Huawei、Aruba、Pulse Secure、McAfee、SonicWall、NetApp、Synologyにまたがる。さらに[Siemens ProductCERTのアドバイザリ](https://cert-portal.siemens.com/productcert/pdf/ssa-462066.pdf)と専用の[CISA ICSアドバイザリ](https://www.cisa.gov/news-events/ics-advisories/icsa-19-253-03)もあり、これがクラウドや企業向け機器だけでなく産業用制御システムにまで及んだことを意味している。

```bash
# 修正で追加されたsysctlが存在するか、またその設定値を確認する
sysctl net.ipv4.tcp_min_snd_mss
```

## 教訓 {#what-it-taught-us}

**固定幅のカウンタは固い上限であって、緩い目安ではない**。`tcp_gso_segs`が`u16`であることは、単一のSKBが現実的に65,535を超えるセグメントを表すことはないという前提のもとでは妥当な選択だった。その前提は、攻撃者がMSSを制御するまでは成り立っていた。

**攻撃者が影響を与えられる状態に対する`BUG_ON()`は、自ら招くサービス拒否である**。そのアサーションはプログラマのミスを捕捉するためのものであり、敵対的な入力のためのものではなかった。リモートのピアが引き起こした不変条件違反をカーネルパニックに変えることは、バグを要求に応じてクラッシュさせるプリミティブに変えてしまう。

**「妥当な最小値」にはそれ自体の下限が必要である**。48バイトのMSS最小値は、他のサイズ計算から暗黙のうちに頼られていた。それがセキュリティに関わるパラメータとして悪用されるまで、コードベースの中にそれをそう扱うものは何もなかった。

**エンバーゴ下で出荷されたリソース上限は、テストされていなかった正当なワークロードと衝突する**。CVE-2019-11478の修正は単独では正しかったが、普通ではないが実在する構成、つまり人為的に小さな`SO_SNDBUF`、ノンブロッキングの10&nbsp;Gbitファブリック、ゼロコピーの`sendfile()`経路に対して検証されたことはなかった。協調的な公開は、修正を出荷前にどれだけ広くテストできるかを制限するからである。後に続いた2つの事件はどちらも、新たな攻撃ではなく正当なトラフィックが新しい上限に引っかかったものだった。上限が実際の利用と衝突しなくなるまでには、さらに2回の改良（`b6653b3629e5`、`b617158dc096`）が必要だった。

!!! warning "注意すべきパターン"
    攻撃者が制御できる値（セグメント数、バッファサイズ、キュー長）から導出される接続ごと・バッファごとのカウンタには、すでに値がおかしくなった後に発火するアサーションだけでなく、値が使われる*前に*強制される明示的な上限が必要である。5.1.11/4.19.52/4.14.127/4.9.182/4.4.182より前のカーネルを動かしている場合は、`sysctl net.ipv4.tcp_min_snd_mss`を確認すること。それが存在しなければ、依然として脆弱である。

## 関連項目 {#see-also}

- [TCPの実装](../tcp.md) — このバグに関わるSACK、再送キュー、受信経路
- [TCP輻輳制御](../tcp-congestion.md) — MSSとセグメントサイズが送信経路とどう相互作用するか
- [SegmentSmack](segmentsmack.md)と[Challenge-ACKサイドチャネル](challenge-ack.md) — 同時期のTCP層の事件簿2件

## 外部参考資料 {#external-references}

- [Netflix: NFLX-2019-001](https://github.com/Netflix/security-bulletins/blob/master/advisories/third-party/2019-001.md) — Jonathan Looneyによる元のアドバイザリ。3つのCVEすべてについてのパッチと回避策を含む
- [NVD: CVE-2019-11477](https://nvd.nist.gov/vuln/detail/CVE-2019-11477)、[CVE-2019-11478](https://nvd.nist.gov/vuln/detail/CVE-2019-11478)、[CVE-2019-11479](https://nvd.nist.gov/vuln/detail/CVE-2019-11479) — TCP SACK panicのCVEレコード
- [git.kernel.org: 3b4929f65b0d](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3b4929f65b0d8249f19a50245cd88ed1a2f78cff) — "tcp: limit payload size of sacked skbs"
- [git.kernel.org: f070ef2ac667](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f070ef2ac66716357066b683fb0baf55f8191a2e) — "tcp: tcp_fragment() should apply sane memory limits"
- [git.kernel.org: 967c05aee439](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=967c05aee439e6e5d7d805e195b3a20ef5c433d6)と[5f3e2bf008c2](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5f3e2bf008c2221478101ee72f5cb4654b9fc363) — "tcp: enforce tcp_min_snd_mss in tcp_mtu_probing()"と、それを支えるsysctl
- [LWN: The TCP SACK panic](https://lwn.net/Articles/791409/) — Jake EdgeによるCVE-2019-11477/78/79の記事。2018年のGSO常時有効化の変更と、2002年のIrixにおけるSACK panicの先例についてのコメントを含む
- [lore.kernel.org: tcp: make sack processing more robust](https://lore.kernel.org/netdev/20190617170354.37770-1-edumazet@google.com/) — netdevでの公開スレッド。CVE-2019-11478の修正に対するChristoph Paaschのリグレッション報告と、Eric Dumazetの追加のガードを含む
- [git.kernel.org: 0a6b2a1dc2a2](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0a6b2a1dc2a2105f178255fe495eb914b09cb37a) — "tcp: switch to GSO being always on"（2018年2月）。後にSACK panicの影響範囲を広げた、無関係な性能修正
- [git.kernel.org: b6653b3629e5](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b6653b3629e5b88202be3c9abc44713973f5c4b4)と[b617158dc096](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b617158dc096709d8600c53b6052144d12b89fab) — 実環境でのリグレッション報告を受けた、CVE-2019-11478の`tcp_fragment()`修正に対する2つの追加の改良
- [Databricks: Adventures in the TCP stack](https://www.databricks.com/blog/2019/09/16/adventures-in-the-tcp-stack-performance-regressions-vulnerability-fixes.html) — CVE-2019-11478の修正における64&nbsp;KBの`SO_SNDBUF`超過許容に関わり、S3書き込みが6倍遅くなった事例の記録。原因は`tcp_fragment()`のメモリ上限チェックに突き止められた
- [Siemens ProductCERT: SSA-462066](https://cert-portal.siemens.com/productcert/pdf/ssa-462066.pdf)と[CISA ICS Advisory ICSA-19-253-03](https://www.cisa.gov/news-events/ics-advisories/icsa-19-253-03) — 産業用制御システム製品にまで影響が及んだことを示す証拠
