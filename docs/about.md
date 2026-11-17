# このサイトについて {#about}

[kernel-internals.org](https://kernel-internals.org)は、Linuxカーネルの背後にある*設計上の理由*のためのリファレンスである — 「これが構造体で、これが関数だ」ではなく、「これが問題で、最初に何が試され、なぜ現在の設計が勝ち残り、その代償は何か」を扱う。

## なぜこのサイトがあるのか {#why-this-exists}

カーネルは徹底的に文書化されているが、その資料は妙に分断されている。[docs.kernel.org](https://docs.kernel.org)は権威あるリファレンスである — それはあるものが*何*であり、*どう*使うかを教えてくれるが、*なぜ*そのように作られているかは教えてくれない。[LWN](https://lwn.net)は*なぜ*について書かれたものとしてはどこよりも優れているが、ニュースの形をとっている: 特定のパッチシリーズに結びつき、サブシステムごとではなく出来事が起きた時期ごとに整理されている。設計の背後にある理由付け — トレードオフ、却下された代替案、決着をつけたメーリングリストでの議論 — は、コミットやスレッドのあちこちに散らばってしまう。

このサイトはその理由付けを一つにまとめ、サブシステムごとに整理する。それにより、腰を据えて、カーネルがどう動くかだけでなく、なぜそのように動くのかを理解できるようにする。docs.kernel.orgと競合するのではなく、それを補完することを意図している。

## 誠実さを保つ方法 {#how-it-stays-honest}

- **すべての主張は一次資料にたどれる** — [git.kernel.org](https://git.kernel.org)上の特定のコミット、[LWN](https://lwn.net)の記事、あるいは[lore.kernel.org](https://lore.kernel.org)のスレッドである。出典を示せない主張はサイトに載せるべきではなく、何かを要約している箇所では原典へワンクリックでたどれる。
- **相互参照は腐らない** — 継続的インテグレーションは、内部リンクやアンカーが1つでも壊れていればビルドを失敗させる。そのため、サイトが成長してもページ間のリンクは正しく保たれる。
- **内容は特定のカーネルスナップショットを反映する** — 各ページは、ある時点のコミット（現在は[`f5bbbfec59b4`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f5bbbfec59b4e2fb7520a91de3df8a6174325d6a)、2026年8月、v7.2-rc7）時点のLinuxメインラインに対してファクトチェックされており、定期的に更新される。個々のページは、各主張がどのカーネルバージョンの時点で正しいかを明記している。したがって、ページが現在のメインラインからわずかに遅れていても誤りになるわけではない — 最新の開発内容が欠けている可能性があるだけである。

## 参加するには {#getting-involved}

このサイトはオープンソースである — 全ソースと履歴は**[GitHub](https://github.com/laveeshb/linux-kernel-internals)**にある。貢献を歓迎する:

- **訂正** — *なぜ*の説明が間違っていたら、ぜひ指摘してほしい。自信満々の誤った説明は、説明がないことよりも悪い。
- **リクエスト** — 取り上げてほしいサブシステムやトピックがあるか？ [ディスカッションを始めよう](https://github.com/laveeshb/linux-kernel-internals/discussions)。
- **貢献** — [コントリビューションガイド](contributing.md)を参照のこと。[オープンなissue](https://github.com/laveeshb/linux-kernel-internals/issues)で次に取り組むことを管理している。

## ライセンス {#license}

- ドキュメント: [CC BY-SA 4.0](https://github.com/laveeshb/linux-kernel-internals/blob/main/LICENSE-CC-BY-SA-4.0)
- コードスニペット: [GPL-2.0](https://github.com/laveeshb/linux-kernel-internals/blob/main/LICENSE-GPL-2.0)

---

[Laveesh Bansal](https://laveeshb.com)が作成・保守している。
