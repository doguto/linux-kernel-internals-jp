# SegmentSmack {#segmentsmack}

> CVE-2018-5390 — 意図的にばらまかれた小さな順序外のTCPセグメントによって、わずか約2 kppsのトラフィックでCPUコアを100%に張り付かせることができた

公開
:   2018年8月6日

報告者
:   Juha-Matti Tilli（Aalto University / Nokia Bell Labs）

CVSS
:   7.5 HIGH

修正バージョン
:   4.9.116、4.14.59、4.17.11、メインライン 4.18

エクスプロイトツール
:   公開されたものはない — 文書化されたトラフィックパターンが攻撃のすべてである

実際の悪用
:   確認された事例なし（CISA KEVに未掲載）

*[事件簿: ネットワークスタックのバグとCVE](../war-stories.md)の一部。*

## 発生前の状態 {#before-state}

TCPセグメントが順序外で到着すると、カーネルはそれをソケットごとの順序外（OFO）キューに保持する。これはシーケンス番号をキーとするrbtree（`tp->out_of_order_queue`。コミット[`9f5afeae5152`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9f5afeae51526b3ad7b7cb21ee8b145ce6ea7a7a) "tcp: use an RB tree for ooo receive queue"で導入）であり、後から来たセグメントをおよそO(log n)時間で挿入し、既存の範囲とマージできる。Eric Dumazetは2016年9月にYaogong Wangに代わって[この変換を投稿した](https://lore.kernel.org/netdev/1473284968.15733.8.camel@edumazet-glaptop3.roam.corp.google.com/)。それに続くレビューは完全に正しさについてであり、攻撃対象領域についてではなかった。Ilpo Järvinenは特に、ストリーム破損のテストが行われたかどうかを尋ね（"This kind of changes have some potential to cause some corruption to the stream content"（訳: この種の変更は、ストリームの内容に何らかの破損を引き起こす可能性がある））、Dumazetは最大10%のパケットロスのもとでGoogleの実際のフロントエンドでテストしたと答えた。そのスレッドの誰も、到着のロス率だけでなく*順序*を操作できる敵対者が新しいツリーに何をしうるかを考えなかった。それこそが、2年後にSegmentSmackがやって来た角度である。メモリ逼迫時には、関連する2つのルーチンがこのキューを管理する。

- `tcp_collapse_ofo_queue()`（`net/ipv4/tcp_input.c`）は、SKBごとのオーバーヘッドを減らすため、より少数のより大きなSKBにマージできる隣接範囲を探してrbtreeをたどる。
- `tcp_prune_ofo_queue()`は、ソケットの受信バッファがメモリ逼迫状態にあるときに、OFOキューからSKBをそのまま破棄する。

どちらも、「隣接範囲」は通常、扱える数の適度なサイズのセグメントであるという前提に基づいて設計されていた。

## きっかけ {#the-trigger}

Juha-Matti Tilli（Aalto University / Nokia Bell Labs）は、リモートの攻撃者が確立済みのコネクション上で、**意図的にランダム化されたシーケンスオフセットを持つ、小さな順序外のTCPセグメント**のストリームを送れることを発見した。各セグメントはrbtree内の別々の隣接しない位置に着地するため、`tcp_collapse_ofo_queue()`は受信パケットのたびに実質的にすべてのノードをたどって調べなければならず、しかもキューを実際に縮小できるほどの隣接性は決して見つからなかった。[Red Hatのアドバイザリ](https://access.redhat.com/articles/3553061)は、わずか約2 kpps（キロパケット/秒）の細工されたトラフィックで病的な挙動を引き起こすのに十分であること、そしてこの攻撃は送信元アドレスを偽装しては行えないため、安価ではあるが本物のTCPコネクションを必要とする（そして消費する）ことを記している。

[Aalto Universityによる発見の経緯の説明](https://www.aalto.fi/en/news/juha-matti-tilli-could-not-sleep-instead-he-had-an-idea-that-took-him-to-the-vulnerabilities)によれば、Tilliはこのバグを無関係な作業の副産物として発見した。彼の大学院での研究はIPフラグメントの再構成に関するもので、Linuxが（ほとんどのオープンソースのスタックと同様に）フラグメントの再構成に単純な連結リストをたどっていることを突き止めていた。これはそれ自体が計算量の弱点であり、後にFragmentSmack（[CVE-2018-5391](https://nvd.nist.gov/vuln/detail/CVE-2018-5391)）として別途追跡された。その連結リストを平衡探索木で置き換えるものをテストする中で、彼はLinuxのスタックとの速度比較を行い、その過程で"also noticed another vulnerability associated with the most common way to break information, TCP segmentation"（情報を分割する最も一般的な方法であるTCPセグメンテーションに関連する、別の脆弱性にも気づいた）。それが順序外キューの縮約ロジックである。彼は両方の発見をフィンランド通信規制当局とLinus Torvaldsに直接報告した。

## 観測された挙動 {#observed-behavior}

攻撃者が制御する持続的な**CPUの張り付き**である。単一の低帯域のコネクションが、順序外キューのO(n)以上のスキャンを繰り返し強制することで、CPUコアを100%の使用率に保つことができた。デフォルトの`tcp_rmem[2]`設定では、キューは数千ノードのオーダーを保持する（デフォルトの6 MBで約7,000ノード）。Red Hatとより広いコミュニティはこれを**SegmentSmack**と名付けた。LWNの続報（["CVE-2018-5390 and 'embargoes'", August 2018](https://lwn.net/Articles/762512/)）は、バグの内部ではなく公開のタイムラインを取り上げたが、実際の影響と「SegmentSmack」という命名を裏付けた。

## なぜ起きたか {#why-it-happened}

CVE-2018-5390のマージ修正は、[`1a4f14bab186`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1a4f14bab1868b443f0dd3c55b689a478f82e72e)（"Merge branch 'tcp-robust-ooo'"）を介してマージされた5コミットのシリーズとして入った。すべてEric Dumazetが作成し、Tilliが報告者となっている。根底にある問題は次のとおりである。

- `tcp_prune_ofo_queue()`はSKBを一度に1つずつ解放し、*ノードを1つ解放するたびに*`sk_mem_reclaim()`を呼んでメモリ逼迫を再チェックしていた。数千ノードを取り除く必要があるときには高コストである。
- `tcp_collapse_ofo_queue()`には早期終了がなかった。マージされる「範囲」が、決して有用なものに合体しえない単独の小さなパケットであっても、rbtree全体にわたってスキャンと縮約の試行を続けた。
- 特定の縮約/刈り込みのパスにどれだけの*作業*（*メモリ*ではなく）が費やされたかを追跡するものが何もなかったため、キューのメモリ使用量を増やせない攻撃者でも、カーネルにそれをたどるために際限なくCPUを費やさせることができた。

## 解決 {#resolution}

`tcp-robust-ooo`シリーズは、OFOキューの経済性を、単にメモリで制限されたものではなく、攻撃者に耐性のあるものにした。

- `tcp: free batches of packets in tcp_prune_ofo_queue()`は、刈り込みを、ノードごとに回収と再チェックを行う代わりに、**1回のパスで`sk_rcvbuf`容量のおよそ12.5%**（`goal = sk->sk_rcvbuf >> 3`）をパージするように変更した。
- `tcp: avoid collapses in tcp_prune_queue() if possible`は、`sk_rmem_alloc`がすでに`sk_rcvbuf`の範囲内にあれば`tcp_collapse_ofo_queue()`の呼び出しを完全にスキップし、新しく開かれたコネクションに対するO(N²)の攻撃対象領域を取り除く。
- `tcp: detect malicious patterns in tcp_collapse_ofo_queue()`は、蓄積された範囲がまだ小さな（`SK_MEM_QUANTUM`未満のサイズの）SKBで構成されている場合は縮約を試みず、スキップした小さな範囲の合計が`sk_rcvbuf >> 3`を超えたら縮約パス全体から抜け出す。これにより、決して有用に縮約されないキューに費やされるCPUに上限が設けられる。
- `tcp: call tcp_drop() from tcp_data_queue_ofo()`と`tcp: add tcp_ooo_try_coalesce() helper`は、破棄の計上（`sk->sk_drops`）を改善し、運用者が進行中の攻撃を実際に観測できるようにする。

David Millerの[netdevでシリーズを適用した際の返信](https://lore.kernel.org/netdev/20180723162821.11556-1-edumazet@google.com/)は、このコードに精通したメンテナにとってさえ修正がいかに自明でなかったかを捉えている: "Sucky... It took me a while to understand the sums_tiny logic, every time I read that function I forget that we reset all of the state and restart the loop after a coalesce inside the loop."（訳: ひどいね……sums_tinyのロジックを理解するのにしばらくかかった。この関数を読むたびに、ループ内で合体した後にすべての状態をリセットしてループを再開していることを忘れてしまう。）彼は同じ日にシリーズ全体を-stable向けにキューに入れた。

それでも5つ目のパッチ（`tcp_ooo_try_coalesce()`ヘルパー）は、当初4.9-stableへのバックポートから漏れた。David Woodhouseは11日後にそれが4.9.116にないことに気づき、Greg Kroah-Hartmanは単にきれいに適用できなかったのだと確認し（"Odds are it did not apply and so I didn't backport it"（訳: おそらく適用できなかったので、バックポートしなかったのだろう））、Woodhouseが動作するバックポートを送った。

マージに続く公開は修正そのものよりも混乱しており、それ自体が議論の的になった。`1a4f14bab186`のマージは2018年7月23日にLinusのツリーに入り、CERTは同じ問題（および関連するFreeBSDのバグ[CVE-2018-6922](https://nvd.nist.gov/vuln/detail/CVE-2018-6922)）についてディストリビューションへの非公開の通知を調整していた。しかしマージされたコミットはマージされた瞬間から公開されており、同日にそれへのリンクを貼ったgrsecurityのツイートと、stableのバックポートが出荷された7月28日の2つ目のツイートによって、CVEが正式に発表されるずっと前から修正の存在と目的が広く知られることになった。

CERTの公開ノートは8月6日まで出なかった。非公開のlinux-distrosリストでバグが議論された後に必要となる、oss-securityメーリングリストへのMatthew Garrettによる義務的な報告は、その2日後の8月8日に続いた。その詳細はあまりに乏しく、リストのメンバーであるStiepan A. Kovacが公にさらなる情報を求めたほどだった。

distros/linux-distros/oss-securityリストのモデレーターであるAlexander Peslyak（"Solar Designer"）は、完全なタイムラインを示し、それを擁護しなかった: "Of course, I am unhappy about this semi-embargo, and even more unhappy about the semi-violation of linux-distros list policy on only having non-public issues in there. However, with CERT involved and with related issues affecting more than just Linux, there was little I could do, short of playing full BOFH and breaking the semi-embargo for everyone."（訳: もちろん、私はこの半エンバーゴに不満であり、非公開の問題のみを扱うというlinux-distrosリストのポリシーへの半ば違反にはさらに不満である。しかし、CERTが関与し、関連する問題がLinux以外にも影響していた以上、完全なBOFHを演じて全員のために半エンバーゴを破ること以外に、私にできることはほとんどなかった。）彼はCERTのノートと義務的なoss-securityの投稿の間の空白についても同様に不満だった: "I've been pinging off-list to make this happen at all, and would have probably made the posting myself if it didn't happen for another day."（訳: そもそもこれを実現させるためにリスト外で催促していたのであり、あと1日起きなければおそらく自分で投稿していただろう。）

[LWNによるこの一件の記事](https://lwn.net/Articles/762512/)は、修正が公開されセキュリティ研究者に指し示されている時点で、それ以上の「エンバーゴ」による遅延は、netdevを読む誰にとってもコード自体がすでに悪用可能である一方で、主に防御側を暗闇に置き去りにするだけだという結論を導いた — "the horse is loose, so the state of the barn door is immaterial."（馬はもう逃げたのだから、納屋の扉の状態はどうでもいい。）

これについても、公開されたエクスプロイトツールやMetasploitモジュールは存在しない。攻撃は文書化されたトラフィックパターン（小さく散らばった順序外セグメント）であり、武器化を必要とするメモリ破壊のプリミティブではない。実世界での影響範囲を最も明確に示すのはベンダーの対応である。NVDは約30件の下流のアドバイザリを挙げており、専用の[Ciscoセキュリティアドバイザリ](https://sec.cloudapps.cisco.com/security/center/content/CiscoSecurityAdvisory/cisco-sa-20180824-linux-tcp)や産業用製品を対象とする[Siemens ProductCERTアドバイザリ](https://cert-portal.siemens.com/productcert/pdf/ssa-377115.pdf)に加え、Oracle、F5、Debian、そして複数回にわたるUbuntuのセキュリティ通知が含まれる。

```bash
# このパターンを示す OFO キューの刈り込み/破棄の動きを監視する
nstat -az | grep -i 'TcpExtOfoPruned\|TcpExtTCPOFOMerge'
```

## 教訓 {#what-it-taught-us}

**メモリで制限されていることは、CPUで制限されていることではない。** `sk_rcvbuf`を守るキューであっても、それをたどるアルゴリズムが自身の作業量に独立した上限を持たなければ、攻撃者は受信バイトあたり際限なくCPUを消費させることができる。

**隣接性を探すアルゴリズムには「諦める」条件が必要である。** 典型的な入力はきれいにマージされると想定する縮約/マージのロジックには、決してマージされないように敵対的に構成された入力に対する明示的な早期終了が必要である。

**安いトラフィックで高価な処理をさせるのは、典型的な計算量攻撃の形である。** 約2 kppsの細工されたセグメントでCPUコアを張り付かせるのは教科書どおりの計算量攻撃である。修正は「トラフィックをブロックする」ことではなく、攻撃者が制御する入力の単位あたりのカーネル自身の作業量に上限を設けることだった。

!!! warning "注意すべきパターン"
    `nstat`/`/proc/net/netstat`の`TcpExtOfoPruned`と`TcpExtTCPOFOMerge`を監視し、特定のピアと相関する異常なスパイクがないか見ること。それがこの種の圧力を受けている順序外キューの観測可能なシグネチャである。4.9.116/4.14.59/4.17.11（メインラインでは4.18）より前のカーネルは、`tcp-robust-ooo`がバックポートされていなければ露出している。

## 関連項目 {#see-also}

- [TCPの実装](../tcp.md) — このバグに関わる順序外キューと受信パス
- [The TCP SACK Panic](sack-panic.md)と[TCP Challenge-ACKサイドチャネル](challenge-ack.md) — 同時期の他の2つのTCP層の事件簿

## 外部参考資料 {#external-references}

- [NVD: CVE-2018-5390](https://nvd.nist.gov/vuln/detail/CVE-2018-5390) — SegmentSmackのCVEレコード
- [git.kernel.org: 9f5afeae5152](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9f5afeae51526b3ad7b7cb21ee8b145ce6ea7a7a) — "tcp: use an RB tree for ooo receive queue"。このバグが悪用した2016年のrbtreeへの変換
- [lore.kernel.org: tcp: use an RB tree for ooo receive queue](https://lore.kernel.org/netdev/1473284968.15733.8.camel@edumazet-glaptop3.roam.corp.google.com/) — 2016年9月のレビュースレッド。提起された疑問はパケットロス下でのストリームの正しさについてであり、敵対的な順序についてではなかった
- [git.kernel.org: 1a4f14bab186](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1a4f14bab1868b443f0dd3c55b689a478f82e72e) — "Merge branch 'tcp-robust-ooo'"、SegmentSmackの修正シリーズ
- [lore.kernel.org: tcp: more robust ooo handling](https://lore.kernel.org/netdev/20180723162821.11556-1-edumazet@google.com/) — netdevへの投稿スレッド。David Millerのマージ時の返信と、4.9-stableへのバックポート漏れについての続報を含む
- [LWN: CVE-2018-5390 and "embargoes"](https://lwn.net/Articles/762512/) — SegmentSmackの公開をめぐる論争についてのJake Edgeの記事。Alexander Peslyakによる完全なタイムラインを含む
- [Red Hat: SegmentSmack and FragmentSmack](https://access.redhat.com/articles/3553061) — CVE-2018-5390についての約2 kppsという攻撃トラフィックの数値と影響の概要
- [Aalto University: Juha-Matti Tilli could not sleep](https://www.aalto.fi/en/news/juha-matti-tilli-could-not-sleep-instead-he-had-an-idea-that-took-him-to-the-vulnerabilities) — 兄弟的な発見であるFragmentSmackと並ぶ、バグがどのように見つかったかについての報告者自身の説明
- [Cisco Security Advisory: cisco-sa-20180824-linux-tcp](https://sec.cloudapps.cisco.com/security/center/content/CiscoSecurityAdvisory/cisco-sa-20180824-linux-tcp)と[Siemens ProductCERT SSA-377115](https://cert-portal.siemens.com/productcert/pdf/ssa-377115.pdf) — 下流ベンダーの対応の広さを示す証拠
