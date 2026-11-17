# UDPフラグメンテーションオフロードのパス切り替えによる破壊 {#udp-fragmentation-offload-path-switch-corruption}

> CVE-2017-1000112 — データグラム単位の不変条件であるべきだった呼び出し単位のヒューリスティックのせいで、2つの`send()`呼び出しがUDPデータグラムの構築方法について食い違うことがあった。syzkallerで発見され、1週間でエンバーゴ付きの公開に至った

公開
:   2017年8月10日

報告者
:   Andrey Konovalov（syzkaller経由）

CVSS
:   7.0 HIGH

バグの混入時期
:   2005年10月

エクスプロイトツール
:   あり（Konovalov自身のPoC + [Metasploitモジュール](https://www.exploit-db.com/exploits/45147)）

実際の悪用
:   確認された事例なし（CISA KEVに未掲載）

*[事件簿: ネットワークスタックのバグとCVE](../war-stories.md)の一部。*

## 発生前の状態 {#before-state}

パスMTUより大きいUDPデータグラムは、`MSG_MORE`を使って段階的に構築できる。カーネルが実際に送信する前に、複数の`send()`呼び出しにわたってデータを追加していくのである。下位のNICがUFO（`NETIF_F_UFO`、UDP向けのスキャッタギャザーによるセグメンテーションオフロード）を提供しているかどうかに応じて、`net/ipv4/ip_output.c`の`__ip_append_data()`（およびそのIPv6版である`net/ipv6/ip6_output.c`の`__ip6_append_data()`）は、送信SKBを構築するための2つのパスのうち1つを選ぶ。1つは`ip_ufo_append_data()`を介するUFOパスで、フラグメンテーションをNIC/ソフトウェアGSOに委ねた単一のGSO SKBを構築する。もう1つは通常のフラグメンテーションパスで、データを自らMTUサイズのSKBに分割し、その過程で各フラグメントの`copy`長を計算する。

UFOのスキャッタギャザー設計自体が、本物の意見の対立の結果だった。[Neterionによる2005年の最初の投稿](https://lore.kernel.org/netdev/20050526232006.60E6365005@linux.site/)は、オフロードのデータパスを`sock_append_data()`経由で構築していた。David Millerは、フラグメントを単にデバイスに渡すのと比べてこれは"seems like a lot of wasted work"（多くの無駄な作業に思える）と反論し、彼が念頭に置いていたまさにその種のフラグメントリストのオフロードを表現するために`NETIF_F_FRAGLIST`がすでに存在していることを指摘した。一方の設計を選ぶ代わりに、2つのアプローチは競合するパッチとしてプロトタイプが作られた — Miller: "I think minimizing driver specific work is probably going to make the SG approach more desirable, but we'll see."（訳: ドライバ固有の作業を最小化することで、おそらくSGアプローチのほうが望ましくなると思うが、様子を見よう。）出荷されたのはスキャッタギャザー版で、`e89e9cf539a2`としてである。

## きっかけ {#the-trigger}

どちらのパスを使うかの判断は、構築中のデータグラムの存続期間中は固定されるのではなく、`MSG_MORE`シーケンス内の**すべての**`send()`呼び出しで、その時点の条件に基づいて再評価されていた。Andrey Konovalovはsyzkallerを使って、同じ大きすぎるデータグラムを構築している同じソケット上の2つの`send()`呼び出しの間で、追加パスがデータグラムの途中で**UFOから非UFOに切り替わる**ことがあることを発見した。

その後、非UFOパスが、UFOパスの規則のもとですでにMTUを超えて大きくなったSKBに追加を続けると、その`copy = maxfraglen - skb->len`の計算が負になりうる。これがフラグメンテーションの分岐を引き起こし、そこで`fraggap = skb_prev->len - maxfraglen`が計算される。これ自体もMTUを超えうるものであり、そこから`copy = datalen - transhdrlen - fraggap`も負になりうる。

## 観測された挙動 {#observed-behavior}

負の`copy`長はそのまま`skb_copy_and_csum_bits()`に渡され、それをコピーサイズとして使う**範囲外書き込み**が行われる。[NVDのエントリ](https://nvd.nist.gov/vuln/detail/CVE-2017-1000112)は、IPv6のコードパスにも同じ欠陥が存在すると説明している。このバグは、2005年10月の元のUFOスキャッタギャザー実装であるコミット[`e89e9cf539a2`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e89e9cf539a28df7d0eb1d0a545368e9920b34ac)（"[IPv4/IPv6]: UFO Scatter-gather approach"）にまでさかのぼった。つまり、syzkallerが見つけるまでおよそ12年間存在していたことになる。

Konovalovは報告だけにとどまらなかった。彼は自身の[ローカル権限昇格の概念実証エクスプロイト](https://github.com/xairy/kernel-exploits/tree/master/CVE-2017-1000112)を公開し、そのPoCは後にh00dieとBrendan Colesによって[Metasploitモジュール](https://www.exploit-db.com/exploits/45147)（`exploit/linux/local/ufo_privilege_escalation`）に移植された。AF_PACKETと同様に、実際の標的に対する能動的な悪用で捕捉されたという公開された証拠はない（CISAのKEVカタログには載っていない）が、この範囲外書き込みは単なるクラッシュではなく、完全なローカルrootに武器化できることが実証されていた。

## なぜ起きたか {#why-it-happened}

このバグは**複数回の呼び出しにまたがるAPIにおける状態の一貫性のバグ**である。`sendmsg(MSG_MORE)`のシーケンスは、複数の呼び出しにわたって1つの論理的なデータグラムを構築するためのものだが、UFOかフラグメンテーションかの判断はデータグラム単位の不変条件ではなく、呼び出し単位のヒューリスティックだった。最初の`send()`呼び出しがSKBを「GSO、UFOに委ねる」形に確定させた後でも、同じシーケンス内の後続の`send()`呼び出しは、追加先のSKBがすでに異なる前提のもとで構築されていたにもかかわらず、非UFOのフラグメンテーションが適用されると独立に判断することができた。

このバグが依存している複数回呼び出しの機能自体が、12年前のレビューで下された意図的な設計判断だった。Ravinandan Arakaliによる元のUFOスキャッタギャザーパッチは、ソケットにすでにキューに入ったSKBがある場合にはUFOパスを一切使わなかった。`if (skb_queue_len(&sk->sk_write_queue))`が`-EOPNOTSUPP`を返し、エラーを表に出すのではなく黙ってソフトウェアフラグメンテーションにフォールバックしていたのである。そして[David Millerはレビューでそのガードを指摘した](https://lore.kernel.org/netdev/20050719.142320.52167011.davem@davemloft.net/)。それは"breaks NFS, and other things using MSG_MORE and UDP_CORK"（NFSや、MSG_MOREとUDP_CORKを使う他のものを壊す）というのである。Millerは選択肢を明示的に示した。UFOの追加パスにcorkされたソケットのサポートを追加するか、その問題を持たない別の実装（"the frag_list patch"）を採用するかである: "I prefer the frag_list patch from a cleanliness perspective, however I remember you saying that the sock_append_data() approach obtained better performance."（訳: きれいさの観点ではfrag_listパッチのほうが好ましいが、sock_append_data()アプローチのほうが性能が良かったとあなたが言っていたのを覚えている。）（実際には、Arakaliは同じ設計論争の以前のラウンドで、両者の間に測定可能な差はなく、スキャッタギャザー版はドライバでの結合作業を最小化するだけだと報告していた。）Neterionはスキャッタギャザーのアプローチを維持し、それを捨てる代わりに複数回呼び出しのサポートを追加した。それこそが、シーケンス途中でのUFO/フラグメンテーションの*パス切り替え*という別の故障モードについて監査されないまま残され、CVE-2017-1000112を生み出したコードパスである。2005年のスレッドはセキュリティではなくNFSの正しさについてのものだったが、それはこのバグが依存している複数回呼び出しの機能が意図的に設計に残された瞬間だった。

## 解決 {#resolution}

Willem de Bruijnが作成し、Andrey Konovalovが報告者となっている[`85f1bd9a7b5a`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=85f1bd9a7b5a79d5baa8bf44af19658f7bf77bfa)（"udp: consistently apply ufo or fragmentation"）は、`__ip_append_data()`と`__ip6_append_data()`の両方でこの不変条件を明示的に強制する。**SKBがいったんGSO（`skb_is_gso(skb)`）になったら、そのデータグラムの残りについては、サイズ/MTUのヒューリスティックが本来何を選ぶかにかかわらず、常にUFOパス経由で追加を続ける**。

```c
if ((skb && skb_is_gso(skb)) ||
    (((length + (skb ? skb->len : fragheaderlen)) > mtu) &&
    (skb_queue_len(queue) <= 1) &&
    (sk->sk_protocol == IPPROTO_UDP) &&
    (rt->dst.dev->features & NETIF_F_UFO) && !dst_xfrm(&rt->dst) &&
    (sk->sk_type == SOCK_DGRAM) && !sk->sk_no_check_tx)) {
        err = ip_ufo_append_data(...);
```

`skb_queue_len(queue) <= 1`の条件は逆方向を扱う。データグラムがフラグメンテーションパスによってすでに複数のSKBに分割されていれば、UFOはもはや考慮されないので、途中でUFO*へ*切り替わることもない。`udp_send_skb()`での関連する変更により、パケットがすでにGSOである場合には`sk->sk_no_check_tx`（チェックサム無効化）も尊重されなくなった。GSO SKBは部分チェックサムを持たなければならないからである。この修正は2017年8月にメインラインに入り、stableカーネルにバックポートされた。DebianはDSA-3981としてそれを出荷した。

David Millerは、[netdevに投稿されたその日に](https://lore.kernel.org/netdev/20170810162919.50577-1-willemdebruijn.kernel@gmail.com/)パッチを適用した（"Applied and queued up for -stable"）。数日後、Vasily Averin（Virtuozzo）がリスト上で、新しいロジックが今度はUDP以外のトラフィックを`ip_ufo_append_data()`経由で流すのではないかと尋ねた。de Bruijnは、`__ip_append_data()`のGSO分岐に到達できるのはUDPソケットだけであると明確にした。TCPやその他のセグメント化可能なプロトコルは代わりに`ip_queue_xmit()`を呼ぶのでそこには到達せず、唯一の狭い例外（`ip_send_unicast_reply()`）もいずれにせよGSO SKBを生成することはないと彼は指摘した。これにより、修正が変更の影響を受けうるコードを意図せず広げていないことが確認された。

修正方法について以前に公開でのやり取りがないのは、修正前に公開の段階がなかったからである。Konovalov自身の[公開の経緯についての説明](https://xairy.io/articles/cve-2017-1000112)によれば、バグは2017年8月3日に`security@kernel.org`へ、翌日に`linux-distros@`へ直接送られ、パッチが公開の`netdev`リストに届いたのは8月10日、マージされ発表されたのと同じ日だった。この圧縮された1週間のエンバーゴ付きタイムラインこそ、netdevのスレッドが議論されたパッチではなく完成したパッチとして読める理由である。議論があったとしても、それはリスト外で行われた。

Konovalovの[公開メール](https://seclists.org/oss-sec/2017/q3/277)が実際に明らかにしているのは、的を絞った修正と並行して走っていた、より広範な2つ目のトラックである: "David has also sent an RFC series to remove UFO completely, which should be merged in 4.14."（訳: DavidはUFOを完全に削除するRFCシリーズも送っており、4.14でマージされるはずである。）そのシリーズ — このCVEが報告される数週間前の2017年7月7日にDavid Millerが投稿した[`[PATCH v2 RFC 0/13] Remove UDP Fragmentation Offload support`](https://lore.kernel.org/netdev/20170707.104326.2165746493532974466.davem@davemloft.net/) — は、このバグへの対応ではなかった。Millerが述べた理由は"Very few devices support this operation, it's usefullness is quesitonable at best, and it adds a non-trivial amount of complexity to our data paths."（訳: この操作をサポートするデバイスはごくわずかで、その有用性はよく言っても疑わしく、データパスに少なからぬ複雑さを加えている。）[原文ママ]というものだった。

しかしそれは、カーネルが同じ根底の機能について2つの修正を同時に進めていたことを意味した。1つは、UFO/フラグメンテーションの不変条件を修復する、最小限でstableにバックポート可能なパッチ（すぐに`-stable`に入れられたため、このCVEに対して実際に出荷されたもの）。もう1つは、ソフトウェアUFOオフロードを完全に削除する、`net-next`限定のより進みの遅い取り組みで、これは大きすぎ、既存のユーザーを壊す可能性も高すぎた（UFOサポートは後に、古いゲストとのライブマイグレーションの互換性を保つためにtuntap/`AF_PACKET`向けに部分的に復活した）ため、緊急のセキュリティ修正の候補にはならなかった。不変条件の修正は、そのより大きなクリーンアップを待つことも、それに依存することもなく、目の前のバグに対処した。

## 教訓 {#what-it-taught-us}

**複数回呼び出しのAPIには、呼び出しごとの正しさだけでなく、呼び出しをまたいで保たれる不変条件が必要である。** 個々の`send(MSG_MORE)`呼び出しのパス選択ロジックは局所的には妥当だった。バグは、それぞれが局所的には正しいが互いに一貫しない判断を下す2つの呼び出しの間の遷移にのみ存在していた。

**12年前のコードパスは検証済みのコードパスではない。ファジングされていないコードパスである。** 2005年からのUFOのスキャッタギャザーロジックは、syzkallerが見つけた特定のインターリーブによって実行されたことが単に一度もなかった。組み合わせ的にまれな入力シーケンスについては、古さは正しさの証拠にならない。

**syzkallerのようなファジングは、まさにこの形のバグを見つける。** 珍しいシステムコールのシーケンス（ここでは、特定のサイズ/MTUの関係を持つ2つの`send()`呼び出し）によって引き起こされる状態機械の不整合は、カバレッジガイド付きのカーネルファジングがまさに得意とするものであり、コードレビューだけでは見つけるのが難しい。

!!! warning "注意すべきパターン"
    1つの論理的なオブジェクトを構築する一連の呼び出しにわたって、呼び出しごとのヒューリスティックな判断（バッファ戦略、オフロードパス、アルゴリズムの選択）が変わりうるコードパスは、このバグクラスの候補である。ワークロードがMTUサイズ付近またはそれ以上でUDPソケットに`MSG_MORE`を使っているなら、`85f1bd9a7b5a`を含むカーネル（2017年8月以降のstable）であることを確認すること。

## 関連項目 {#see-also}

- [UDPソケットの内部](../udp.md) — このバグの背後にある送信パスとUFO/セグメンテーションの挙動
- [AF_PACKETの権限昇格](af-packet.md) — 同時期にsyzkallerで見つかった別のバグ

## 外部参考資料 {#external-references}

- [NVD: CVE-2017-1000112](https://nvd.nist.gov/vuln/detail/CVE-2017-1000112) — UDP UFOパス切り替えのCVEレコード
- [git.kernel.org: e89e9cf539a2](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e89e9cf539a28df7d0eb1d0a545368e9920b34ac) — "[IPv4/IPv6]: UFO Scatter-gather approach"（2005年）、このバグの起源である元の実装
- [lore.kernel.org: UFOの元の設計投稿](https://lore.kernel.org/netdev/20050526232006.60E6365005@linux.site/) — Millerが`NETIF_F_FRAGLIST`による代替案を推し、2つのアプローチが競合するプロトタイプとして用意された2005年5〜6月のスレッド
- [lore.kernel.org: Arakaliの2005年6月のv2投稿](https://lore.kernel.org/netdev/20050603004106.BAB6A7B990@linux.site/)と、複数回呼び出しの`MSG_MORE`/`UDP_CORK`ガードがNFSを壊すと指摘した[Millerの2005年7月のレビュー](https://lore.kernel.org/netdev/20050719.142320.52167011.davem@davemloft.net/) — このバグが後に悪用した機能が、捨てられずに残された瞬間
- [git.kernel.org: 85f1bd9a7b5a](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=85f1bd9a7b5a79d5baa8bf44af19658f7bf77bfa) — "udp: consistently apply ufo or fragmentation"
- [lore.kernel.org: udp: consistently apply ufo or fragmentation](https://lore.kernel.org/netdev/20170810162919.50577-1-willemdebruijn.kernel@gmail.com/) — netdevスレッド。David Millerによる即日のマージと、Vasily Averinによる正しさについての後続の質問を含む
- [Andrey Konovalov: CVE-2017-1000112](https://xairy.io/articles/cve-2017-1000112) — 報告者自身による解説と公開のタイムライン
- [xairy/kernel-exploits: CVE-2017-1000112](https://github.com/xairy/kernel-exploits/tree/master/CVE-2017-1000112) — Konovalov自身が公開したローカルrootの概念実証
- [Exploit-DB 45147](https://www.exploit-db.com/exploits/45147) — 同じバグに対するMetasploitモジュール
- [oss-security: CVE-2017-1000112 disclosure](https://seclists.org/oss-sec/2017/q3/277) — Konovalovによる公開告知。David Millerが並行して進めていたUFO削除のRFCへの言及を含む
- [lore.kernel.org: [PATCH v2 RFC 0/13] Remove UDP Fragmentation Offload support](https://lore.kernel.org/netdev/20170707.104326.2165746493532974466.davem@davemloft.net/) — ソフトウェアUFOを完全に削除するDavid MillerのRFCシリーズで、このCVEが報告される数週間前から進行中だった。[v1](https://lore.kernel.org/netdev/20170705.160402.80726683432003025.davem@davemloft.net/)もアーカイブされている
