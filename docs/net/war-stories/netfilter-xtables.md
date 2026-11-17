# Netfilter x_tablesのヒープオーバーフロー {#netfilter-x_tables-heap-overflow}

> CVE-2021-22555 — iptablesのcompat層にあった15年前からの境界チェックの欠落は、syzbotの受信箱で8か月間手つかずのまま放置され、"Turning \x00\x00 into 10000$"となり、Kubernetesのpodエスケープとなった

公開
:   2021年7月（修正は2021年4月）

報告者
:   syzbot（2020年8月、返答なし）およびAndy Nguyen, Google（2021年4月）

CVSS
:   8.3 HIGH（GoogleのCNAスコア）/ 7.8 HIGH（NVDの一次評価）

バグの混入時期
:   2.6.19-rc1（2006年）

実際の悪用
:   あり — 2025年10月にCISAのKEVカタログに追加された（[NVD](https://nvd.nist.gov/vuln/detail/CVE-2021-22555)による）

*[事件簿: ネットワークスタックのバグとCVE](../war-stories.md)の一部。*

## 発生前の状態 {#before-state}

`iptables`/`ip6tables`/`arptables`のルールセットはユーザー空間で構築され、`setsockopt()`を介してカーネルに渡される。64ビットカーネルでは、32ビットのユーザー空間ツール（compatモード）は**compat**レイアウトでルールを提出し、カーネルは使用前にそれをネイティブの64ビット構造体に変換しなければならない。この変換は`translate_compat_table()`（`net/ipv4/netfilter/ip_tables.c`、`net/ipv6/netfilter/ip6_tables.c`、`net/ipv4/netfilter/arp_tables.c`のそれぞれにある）で行われ、ネイティブレイアウト用のサイズの新しいバッファを割り当ててから、`net/netfilter/x_tables.c`の`xt_compat_match_from_user()` / `xt_compat_target_from_user()`を呼び出して、各match/targetエントリをそこにコピーして展開する。

ネイティブの`xt_entry_match`/`xt_entry_target`構造体はアラインされたサイズ（`XT_ALIGN()`）にパディングされるが、カーネルが32ビットのユーザー空間から受け取るcompat版は必ずしも同じようにアラインされているわけではない。`xt_compat_match_from_user()`/`xt_compat_target_from_user()`の修正前のコードは、個々のエントリごとにパディングの隙間だけをゼロにしようとしていた。

```c
pad = XT_ALIGN(match->matchsize) - match->matchsize;
if (pad > 0)
    memset(m->data + match->matchsize, 0, pad);
```

## きっかけ {#the-trigger}

このエントリごとのパディングのゼロ埋めは、各match/targetごとに独立に計算されたオフセット（`m->data + match->matchsize`）を使っており、それを変換後のルールセット*全体*のために割り当てられた書き込み先バッファの実際のサイズと照合し直していなかった。`matchsize`/`targetsize`が8バイト境界にアラインされていないtargetを選ぶことで（[Andy Nguyenのエクスプロイト解説](https://google.github.io/security-research/pocs/linux/cve-2021-22555/writeup.html)はそのようなtargetの一つとして`NFLOG`を挙げており、制御可能な範囲外オフセットによってバッファの最大`0x4C`バイト先まで到達できる）、そのようなcompatルールセットをロードできるローカルプロセスは、`memset()`に**割り当てられたルールセットのblobの末尾を越えて**少数のゼロバイトを書き込ませることができた。ヒープの範囲外書き込みである。

このコードパスに到達するには、32ビットのcompatコンテキストから`setsockopt(IPT_SO_SET_REPLACE)`（またはIPv6/ARPの同等物）を呼び出せればよく、それには`CAP_NET_ADMIN`が必要である。これは、ユーザー名前空間*かつ*ネットワーク名前空間の中にいる非特権ユーザーが日常的に利用できるケーパビリティである。

## 観測された挙動 {#observed-behavior}

このバグは**2.6.19-rc1**（2006年）から、およそ15年間存在しており、`net/netfilter/x_tables.c`を介して到達可能だった。カーネルには[syzbot](https://syzkaller.appspot.com/bug?extid=cfc0247ac173f597aaaa)と、独立に[Andy Nguyen](https://google.github.io/security-research/pocs/linux/cve-2021-22555/writeup.html)（Google）によって報告された。

その深刻度は、バグ自体（ほんの数バイトのゼロの書き込み）よりも、そのプリミティブが可能にしたことに由来していた。Andy Nguyenの公開解説[`CVE-2021-22555: Turning \x00\x00 into 10000$`](https://google.github.io/security-research/pocs/linux/cve-2021-22555/writeup.html)は、4バイトのゼロの範囲外書き込みを使い、ヒープスプレーによって使用中の`msg_msg`オブジェクトを破壊し、use-after-freeを構築し、カーネルのヒープとコードのアドレスをリークし、`pipe_buffer`の関数ポインタテーブルを乗っ取り、最終的に`commit_creds()`を呼んで名前空間を切り替えるROPチェーンを実行することを示している。小さなOOB書き込みを完全なカーネルコード実行に変えたのである。NguyenはこのエクスプロイトをGoogleのkCTFクラスタでKubernetesのpodの隔離から脱出するのに使った。これはLWNで取り上げられた（["CVE-2021-22555: Turning \x00\x00 into 10000$", July 2021](https://lwn.net/Articles/862955/)）。

影響はその2021年の実証にとどまらなかった。CISAは2025年10月にCVE-2021-22555をKnown Exploited Vulnerabilitiesカタログに追加しており、これは[NVD](https://nvd.nist.gov/vuln/detail/CVE-2021-22555)で確認できる。元の修正から4年以上後のことであり、研究用のkCTFクラスタだけでなく、パッチが広く利用可能になってからずっと後に、実際の標的に対する能動的な悪用が観測されたことを裏付けている。このページの7つのインシデントのうち、CISAが能動的に悪用されているとして掲載したことがあるのはこれだけである。

syzbotの報告は、実際にはNguyenの報告より8か月近く前にさかのぼる。2020年8月17日、syzbotは"KASAN: slab-out-of-bounds Write in xt_compat_target_from_user"という題の[KASAN slab-out-of-boundsレポートをメールで送った](https://lore.kernel.org/all/00000000000022934305ad166be3@google.com/)。宛先は`netdev@`、`netfilter-devel@`、netfilterコアチームのアドレス、David Miller、Florian Westphalで、`net/netfilter/x_tables.c:1129`を特定する完全なクラッシュトレースと、すぐに実行できるCの再現プログラムが付いていた。誰も返信しなかった。

そのレポートの[syzbotダッシュボード](https://syzkaller.appspot.com/bug?extid=cfc0247ac173f597aaaa)は、スレッド上で人間によるフォローアップがゼロであったことを示しており、原因と修正のbisectもともに失敗していた。それは、Nguyenが`security@kernel.org`を通じて独立に行ったエクスプロイト駆動の報告をきっかけとする2021年4月のWestphalの修正が手作業で紐付けられるまで、未対応のまま開いたバグとして放置されていた。出荷されたコミットには`Reported-by: syzbot+cfc0247ac173f597aaaa@syzkaller.appspotmail.com`と`Reported-by: Andy Nguyen <theflow@google.com>`の両方が付いている。自動レポートには、メンテナがバグを再現して修正するのに必要なものが8か月前の時点ですべて揃っていた。欠けていたのは検出ではなくトリアージだった。

## なぜ起きたか {#why-it-happened}

根本原因は、**欠けているグローバルな境界チェックの代わりを、ローカルなエントリごとの境界チェックが務めていた**ことである。各match/targetのパディング書き込みは、それ単体では「正しかった」（そのエントリのデータの先にちょうど`pad`バイトを書き込んでいた）が、ルールセット内のすべてのエントリを変換した累積的な結果が、最初に割り当てた単一のバッファ内に収まることを検証するものは何もなかった。15年前からの前提（compatモードのエントリサイズの合計がネイティブの割り当てを超えることはない）は、特定のtargetの種類では誤りであることが判明した。

## 解決 {#resolution}

[`b29c457a6511`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b29c457a6511435960115c0f548c4360d5f4801d)（"netfilter: x_tables: fix compat match/target pad out-of-bound write"）は、エントリごとのパディングの`memset()`を完全に取り除き、代わりにエントリが変換されて入る前に、**書き込み先バッファ全体を最初にゼロにする**。

```c
newinfo = xt_alloc_table_info(size);
memset(newinfo->entries, 0, size);   /* ip_tables.c、ip6_tables.c、arp_tables.c の translate_compat_table() に追加 */
```

これにより、境界追跡の問題は完全に回避される。エントリ固有のデータがコピーされる前にバッファのすべてのバイトがすでにゼロになっているので、バッファの外に着地しうる別個のエントリごとの書き込みはもはや存在しない。この修正は2021年4月にアップストリームにマージされ、stableカーネル全体にバックポートされた。

[パッチはnetfilter-develで何の反対も受けなかった](https://lore.kernel.org/netfilter-devel/20210407193857.21120-1-fw@strlen.de/)。Pablo Neira AyusoはWestphalの投稿を1週間以内に適用し、単に"Applied."とだけ返信した。

とはいえ、修正自体で議論が終わったわけではなかった。2週間後、Westphalは[後続パッチを投稿した](https://lore.kernel.org/netfilter-devel/20210426101440.25335-1-fw@strlen.de/)。"netfilter: allow to turn off xtables compat layer"で、`CONFIG_NETFILTER_XTABLES_COMPAT`というKconfigのノブ（互換性のためデフォルトは`y`）を追加し、32ビットのcompat変換コードを完全にコンパイルから外せるようにするものである。彼が述べた理由は率直だった: "The compat layer needs to parse untrusted input (the ruleset) to translate it to a 64bit compatible format. We had a number of bugs in this department in the past, so allow users to turn this feature off."（訳: compat層は、信頼できない入力（ルールセット）を解析して64ビット互換の形式に変換する必要がある。この分野では過去に多くのバグがあったので、ユーザーがこの機能をオフにできるようにする。）

Jan Engelhardtはこのアイデアをさらに推し進め、compat固有の構造体レイアウトを必要とするmatchはもう1つ（`xt_limit`の`struct xt_rateinfo`）しか残っていないのだから、compat層を単にオプションにするのではなく完全に削除できないかと尋ねた: "Perhaps the remaining one... could be respecified as a v1, with the plan to ditch the v0. Then the entire xtables_compat code could go as well."（訳: 残っている1つは……v0を廃止する計画のもとでv1として仕様を定め直せるかもしれない。そうすればxtables_compatのコード全体もなくせる。）Pablo Neira Ayusoは原則として同意した — "If the remaining matches and targets that rely on this get a new revision to fix their structure layout issues, then this entire layer could be peeled off"（訳: これに依存している残りのmatchとtargetが、構造体レイアウトの問題を修正する新しいリビジョンを得れば、この層全体を剥がせる）— が、Westphalは、その時点でも完全な削除がそれほど単純ではない理由を説明した。compatとネイティブのルールのblobは本質的に異なるアラインメント規則を持ち（`ip(6)t_entry`/match/targetの構造体は32ビットと64ビットのレイアウトの間で1対1に対応しない）、`ipt_get_entries`にはi686にはない構造体途中のパディングがx86_64にはあり、`compat_to_user`コールバックは`xt_rateinfo`のようなmatchごとの構造体だけでなく、標準targetそのものにも必要なのである。

彼は代わりにこのアイデアのより限定的なバージョンを提案した。`xt_limit`と`ebt_mark`がその唯一の残りのユーザーであるため、matchごとの`xt_match`のcompatコールバックだけを削除し、コアの変換層はそのまま残すというものである。出荷されたのは最初に投稿されたとおりのKconfigトグルだった。Pabloは同じ日にそれを適用した: "Applied, thanks."

```bash
# compat netfilter には 32 ビットの setsockopt() パス経由でしか到達できない。
# 関係するノブは CONFIG_NETFILTER_XTABLES とユーザー名前空間の利用可否である
sysctl kernel.unprivileged_userns_clone   # 存在する場合（一部のディストリビューション）、0 にすると userns 経由の CAP_NET_ADMIN パスを塞げる
```

## 教訓 {#what-it-taught-us}

**項目ごとの境界チェックはバッファの境界チェックではない。** 各`memset()`呼び出しは、それが処理しているエントリに対しては個別に正しかった。バグは集合の中にあり、それはどの単一の呼び出し箇所からも見えなかった。

**「最初に全部ゼロにする」は「必要なところを、必要な分だけ正確にゼロにする」に勝る。** この修正は、精密なエントリごとの計算を、より粗いが安全性を証明できるバッファ全体のゼロ埋めと引き換えにした。精密なバージョンが多くの呼び出し箇所間の相互作用についての推論を必要とする場合には、良い一般的な直感である。

**小さなプリミティブと決意あるエクスプロイト作者が揃えば、完全な侵害になる。** 範囲外に書き込まれた4バイトのゼロだけで、完全なカーネルコード実行のチェーンを構築するのに十分だった。「ここから何に到達できるか」ではなく「何バイトか」に基づく深刻度評価は、このようなバグを過小評価する。

**返答のない自動レポートは、却下されたレポートではない。トリアージされていないレポートである。** syzbotは、バグが独立に再発見されて武器化される8か月前に、完全なクラッシュトレースと動作する再現プログラムをメンテナに手渡していた。欠けていたのは検出ではなくトリアージだった。ファザーがバグを見つけることと人間がそれに対処することは2つの別々の出来事であり、その間の隙間こそ、深刻度が低そうに見えるクラッシュが、誰かがそのためのエクスプロイトチェーンを構築するまで放置されうる場所である。

!!! warning "注意すべきパターン"
    ユーザー名前空間から到達可能な`CAP_NET_ADMIN`/`CAP_NET_RAW`のコードパスは、特に入念に精査する価値がある。かつては「信頼されたroot相当の呼び出し元」を意味していたケーパビリティが、今ではコンテナ内の非特権ユーザーに日常的に適用される。`kernel.unprivileged_userns_clone`（またはディストリビューションの同等のノブ）が有効なら、`CAP_NET_ADMIN`で制限されたパスのどのバグも、単なるローカルのバグではなく、潜在的なコンテナエスケープのプリミティブである。

## 関連項目 {#see-also}

- [Netfilterのアーキテクチャ](../netfilter.md) — x_tables、フック、そしてこのバグの背後にあるcompat変換層
- [ネットワーク名前空間](../net-namespaces.md) — 非特権のユーザー+ネットワーク名前空間が、`CAP_NET_ADMIN`で制限されたコードパスをどう露出させるか
- [AF_PACKETの権限昇格](af-packet.md) — ケーパビリティで制限されたローカルのバグが、コンテナエスケープのプリミティブになったもう一つの例

## 外部参考資料 {#external-references}

- [NVD: CVE-2021-22555](https://nvd.nist.gov/vuln/detail/CVE-2021-22555) — Netfilter x_tablesヒープオーバーフローのCVEレコード
- [git.kernel.org: b29c457a6511](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b29c457a6511435960115c0f548c4360d5f4801d) — "netfilter: x_tables: fix compat match/target pad out-of-bound write"
- [lore.kernel.org: netfilter: x_tables: fix compat match/target pad out-of-bound write](https://lore.kernel.org/netfilter-devel/20210407193857.21120-1-fw@strlen.de/) — netfilter-develに投稿されたFlorian Westphalのパッチ
- [syzbot: KASAN: slab-out-of-bounds Write in xt_compat_target_from_user](https://syzkaller.appspot.com/bug?extid=cfc0247ac173f597aaaa) — 2020年8月の元のレポートのsyzbotダッシュボード。8か月後に修正がマージされるまで、レポートのスレッドに返信がなかったことを示している
- [lore.kernel.org: syzbotの元のレポート](https://lore.kernel.org/all/00000000000022934305ad166be3@google.com/) — netdev、netfilter-devel、netfilterメンテナにメールで送られ、返答のなかった2020年8月17日のKASANレポート
- [lore.kernel.org: netfilter: allow to turn off xtables compat layer](https://lore.kernel.org/netfilter-devel/20210426101440.25335-1-fw@strlen.de/) — `CONFIG_NETFILTER_XTABLES_COMPAT`によるオプトアウトか、compat層の完全な削除かを議論した後続スレッド
- [Andy Nguyen: CVE-2021-22555: Turning \x00\x00 into 10000$](https://google.github.io/security-research/pocs/linux/cve-2021-22555/writeup.html) — 一次資料であるエクスプロイト解説。kCTFでのコンテナエスケープへの利用を含む
- [LWN: CVE-2021-22555: Turning \x00\x00 into 10000$](https://lwn.net/Articles/862955/) — Nguyenの解説についてのLWNの短報
