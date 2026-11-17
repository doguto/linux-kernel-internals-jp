# TCP Challenge-ACKサイドチャネル {#the-tcp-challenge-ack-side-channel}

> CVE-2016-5696 — 内部の実装詳細のつもりだったグローバルなレート制限カウンタが、オフパス攻撃者にとって被害者のTCPコネクションを覗く窓になった

公開
:   2016年8月

報告者
:   Yue Cao, Zhiyun Qian, et al.（UC Riverside、US Army Research LaboratoryのLisa Marvelと共同）

CVSS
:   4.8 MEDIUM

修正バージョン
:   メインライン4.7

エクスプロイトツール
:   あり（[`mountain_goat`](https://github.com/Gnoxter/mountain_goat) PoC）

実際の悪用
:   確認された事例なし（CISA KEVに未掲載）

*[事件簿: ネットワークスタックのバグとCVE](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

RFC 5961は、ブラインドなウィンドウ内パケット注入に対してTCPを強化する。ウィンドウ内の（ただし正確に正しいわけではない）シーケンス番号を持つ偽装SYNやRSTが届いたとき、黙って受け入れたりコネクションをリセットしたりする代わりに、受信側は**challenge ACK**を送る。これは*正確な*次の期待シーケンス番号を運ぶもので、受信側は本物のピアがそれを確認するか無視するかを待つ。これにより、オフパス攻撃者（トラフィックを見られない攻撃者）は受信ウィンドウ内に収めるだけでなく正確なシーケンス番号を推測しなければならなくなり、これははるかに難しい問題である。

challenge ACKはリソースを消費し、RFC 5961はそれがまれにしか発生しないと想定していたため、Linuxはそれを単一の**グローバルカウンタ**`challenge_count`でレート制限していた。このカウンタは1秒に1回リセットされ、`sysctl_tcp_challenge_ack_limit`（デフォルトは100/秒）で上限が設けられており、`net/ipv4/tcp_input.c`の`tcp_send_challenge_ack()`にあった。

## きっかけ {#the-trigger}

UC RiversideのYue Caoらは（US Army Research LaboratoryのLisa Marvelと共同で）、このカウンタが共有されたグローバルなものであるという性質が、それを**サイドチャネル**に変えてしまうことを発見した。オフパス攻撃者は次のことができる。

1. 特定またはハイジャックしようとしている被害者コネクションに対して、challenge ACKを最大数引き起こすように設計した偽装RST/SYNパケットのバーストを送る。
2. 同時に、同じサーバーへの自分自身の正当なコネクション上でプローブトラフィックを送る。
3. *自分自身の*コネクションが受け取ったchallenge ACKの数を数える。数が期待される最大値に届かなければ、グローバルな割り当て枠の一部が被害者に送られたchallenge ACKによって消費されたことになる。つまり、攻撃者の偽装した推測がウィンドウ内に入った（あるいはそもそも生きているコネクションと一致した）ということである。

異なる推測でこれを繰り返すことで、攻撃者は与えられた4タプルが生きているコネクションを表すかどうかを絞り込み、最終的にはウィンドウ内のシーケンス番号まで絞り込む。そのすべてを、**本物の通信のパケットを一つも見ることなく**行える。

## 観測された挙動 {#observed-behavior}

LWNの記事（["The TCP 'challenge ACK' side channel", August 2016](https://lwn.net/Articles/696868/)）は論文の結果を次のようにまとめた: "it takes only 10 seconds to successfully infer whether they are communicating. If there is a connection, subsequently, it takes also only tens of seconds to infer the TCP sequence numbers."（訳: 通信しているかどうかの推定に成功するまでわずか10秒しかかからない。コネクションが存在すれば、その後TCPシーケンス番号を推定するのにもわずか数十秒しかかからない。）

研究者らは、ブラインドなコネクションリセット（SSHやTorのコネクションに対して）と、より深刻なものとして、動画ストリームのような長寿命コネクションへのブラインドなデータ注入の両方を実証した。これには、暗号化されていないWebセッションへの攻撃者が制御するJavaScriptの注入も含まれる。この論文はUSENIX Security 2016で発表された。

この手法は論文の中だけにとどまらなかった。オフパス推定攻撃を実装したスタンドアロンの概念実証ツール[`mountain_goat`](https://github.com/Gnoxter/mountain_goat)が別途公開された。このページの他のローカルで武器化されたバグと同様に、実際の標的に対する能動的な悪用で捕捉されたという公開記録はない（CISA KEVへの掲載も、確認されたインシデント報告もない）が、このサイドチャネルは理論上存在するだけでなく、実際に悪用可能であることが実証されていた。

当時、LinuxはRFC 5961のchallenge-ACKの挙動を忠実に実装した唯一の主要OSであったため、この特定のサイドチャネルに対して脆弱な唯一のOSでもあった。

## なぜ起きたか {#why-it-happened}

根本原因は、典型的な共有リソースのサイドチャネルである。内部の実装詳細（レート制限）のつもりだった単一のグローバルカウンタが、攻撃者が共有の割り当て枠を*消費*し、かつ別の視点からその枯渇を*測定*できるようになった時点で、外部から観測可能なシグナルになった。RFC 5961自体はレート制限を推奨していたが、制限値が正確で予測可能な低い値（100/秒）であり、それが厳格なリセット境界（1秒に1回）と組み合わさることで、カウンタを安価に枯渇させ容易に観測できるようになることまでは想定していなかった。

## 解決 {#resolution}

[`75ff39ccc1bd`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=75ff39ccc1bd5d3c455b6822ab09e533c551f758)（"tcp: make challenge acks less predictable"）は、Linus Torvaldsの分析に基づき、2つの変更を行った。デフォルトの制限を100から1000 challenge ACK/秒（`sysctl_tcp_challenge_ack_limit`）に引き上げ、さらに1秒ごとの割り当て枠の消費方法を**ランダム化**した。ゼロから数え上げて固定の制限と比較する代わりに、毎秒`challenge_count`を制限値の0.5倍から1.5倍の間のランダムな値で初期化し、*カウントダウン*するようにした。これにより、攻撃者は残りの割り当て枠がどれだけあるかを安価に判断できなくなった。

```c
u32 half = (sysctl_tcp_challenge_ack_limit + 1) >> 1;
challenge_timestamp = now;
WRITE_ONCE(challenge_count, half + prandom_u32_max(sysctl_tcp_challenge_ack_limit));
```

この修正は4.7でメインラインに入った。既存のソケット単位のレート制限（`tcp_oow_rate_limited()`）はそのまま残された。Jason Baron（Akamai）は数日後に[`083ae308280d`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=083ae308280d13d187512b9babe3454342a7987e)（"tcp: enable per-socket rate limiting of all 'challenge acks'"）で追従した。これは、"as Eric Dumazet has suggested"（Eric Dumazetが提案したように）、最終的にグローバル制限を完全に取り除くための一歩として[明示的に位置づけられていた](https://lore.kernel.org/netdev/1468510720-322-1-git-send-email-jbaron@akamai.com/)。修正のないカーネルを使う運用者には、文書化された暫定的な緩和策があった。`net.ipv4.tcp_challenge_ack_limit`を非常に大きな値（例: `999999999`）に引き上げ、カウンタを実質的に無制限にし、それによって攻撃者にとって情報を与えないものにするというものである。

この修正は、マージされるまでにnetdev上でYue Caoと実際にやり取りを重ねた。Dumazetの[最初の投稿](https://lore.kernel.org/netdev/1467995586.30694.34.camel@edumazet-glaptop3.roam.corp.google.com/)は、1秒ごとの*リセット境界*（0.5秒から1.5秒の間のウィンドウ）をランダム化するだけだった。Caoは、1つのウィンドウ内に確実に収まる短いバーストを送り、境界ケースを解決するために推測を繰り返すことで攻撃を適応させられると返信した。これがきっかけとなって、代わりに*カウント*をランダム化する["v2"パッチ](https://lore.kernel.org/netdev/1468137842.30694.58.camel@edumazet-glaptop3.roam.corp.google.com/)が生まれ、それが出荷されたバージョンとなった。

Caoはさらに、v2に対する攻撃の改良版（1,000をはるかに超えるプローブを送り、返ってきたchallenge ACKの数を使って正しい推測を推定する）を説明し、FreeBSDもWindowsもグローバル制限を課していないことを指摘したうえで、グローバル制限を単純に取り除くべきではないかと尋ねた。Dumazetの返信は、それでもカーネルが制限を維持した理由を説明していた。この制限は、3.6でレート制限が追加される以前にサーバーを悩ませていた、"buggy firewalls and appliances"（バグのあるファイアウォールやアプライアンス）が引き起こす"ACK storms"を抑えるために存在するのである。彼は、それと比べれば残るサイドチャネルは"small nuisance"（小さな厄介事）だと判断し、改良版の攻撃が必要とするおよそ500本のコネクションを確立すること自体、実際のサーバーに対して目立たずに行うのは難しいと指摘した。セッションハイジャックのリスクは、タイミングチャネルの最後の1ビットまで塞ぐよりもTLSで対処するほうがよい、というわけである。

## 教訓 {#what-it-taught-us}

**レートリミッタはコバートチャネルでもある。** 攻撃者が摂動を与えることも観測することもできる共有カウンタは、たとえ間接的であっても、また保護されたデータを一切読まなくても、攻撃者が推定できるべきでない状態についての情報を漏らしうる。

**予測可能なリセットはサイドチャネルを増幅する。** 固定された1秒ごとのリセット境界により、攻撃者はプローブをカウンタのライフサイクルに同期させることができ、ノイズの多いシグナルを信頼できるシグナルに変えられた。

**唯一の準拠実装であることは、唯一の露出した実装であることを意味しうる。** RFC 5961を忠実に実装したというLinuxの正しさこそが、このサイドチャネルを生み出した。RFCのレート制限を実装していなかった他のスタックには、攻撃対象となるこの特定のカウンタがそもそも存在しなかった。

!!! warning "注意すべきパターン"
    セキュリティに関わる応答（challenge ACK、エラー応答、リトライ制限）を制御する、グローバルに共有され攻撃者が摂動を与えられるカウンタは、摂動を直接引き起こしていない者がその状態を推定できるなら、潜在的なサイドチャネルである。`sysctl net.ipv4.tcp_challenge_ack_limit`を確認すること。少なくとも修正後のデフォルトである1000であるべきで、4.7より古いカーネルは露出しているものとして扱うべきである。

## 関連項目 {#see-also}

- [TCPの実装](../tcp.md) — シーケンス番号の追跡とchallenge-ACKのパス
- [The TCP SACK Panic](sack-panic.md)と[SegmentSmack](segmentsmack.md) — 同時期の他の2つのTCP層の事件簿

## 外部参考資料 {#external-references}

- [NVD: CVE-2016-5696](https://nvd.nist.gov/vuln/detail/CVE-2016-5696) — TCP challenge-ACKサイドチャネルのCVEレコード
- [git.kernel.org: 75ff39ccc1bd](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=75ff39ccc1bd5d3c455b6822ab09e533c551f758) — "tcp: make challenge acks less predictable"
- [lore.kernel.org: tcp: make challenge acks less predictable (v1)](https://lore.kernel.org/netdev/1467995586.30694.34.camel@edumazet-glaptop3.roam.corp.google.com/)と[(v2)](https://lore.kernel.org/netdev/1468137842.30694.58.camel@edumazet-glaptop3.roam.corp.google.com/) — 報告者Yue Caoとのnetdevスレッド。v1からv2への反復と、残るサイドチャネルを受け入れたDumazetの論拠を示している
- [lore.kernel.org: tcp: enable per-socket rate limiting of all "challenge acks"](https://lore.kernel.org/netdev/1468510720-322-1-git-send-email-jbaron@akamai.com/) — グローバル制限の除去に向けたJason Baronの明示的な追従パッチ
- [LWN: The TCP "challenge ACK" side channel](https://lwn.net/Articles/696868/) — Jake Edgeによる記事。USENIX Security 2016論文の詳細を含む
- [Gnoxter/mountain_goat](https://github.com/Gnoxter/mountain_goat) — オフパス推定攻撃を実装した、公開済みの概念実証エクスプロイト
