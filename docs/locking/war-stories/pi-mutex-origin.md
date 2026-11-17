# PI-Mutex の起源 {#the-pi-mutex-origin-story}

> 優先度の低いロック保持者は、そのロックとはまったく無関係な中優先度のタスクによって、無期限に飢餓状態にされうる — 優先度継承 mutex が存在するのは、通常の mutex にはこれを止める手段がないからであり、カーネルがそれでもメインラインの実装を抱えるまでには、Linus Torvalds からの率直な公開反論を経る必要があった

マージ
:   Linux 2.6.18（2006年9月20日）

作者
:   Ingo Molnar、Thomas Gleixner（rt_mutex コアと PI-futex の統合）。Steven Rostedt（設計ドキュメント）

起源
:   ツリー外の `-rt`（PREEMPT_RT）パッチシリーズ

CVE ではない

*[事件簿：ロックと同期のバグ](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

通常の mutex は排他を強制するが、スケジューリングについては何も規定しない。カーネル自身の rt_mutex 設計ドキュメントは、この失敗モードを正確に述べている。A（高優先度）、B（中優先度）、C（低優先度）の3つのプロセスがあり、C が保持するロックで A がブロックしたとき、"B executes, and since B is of a higher priority than C, it preempts C, but by doing so, it is in fact preempting A which is a higher priority process. ... B is a CPU hog and will never give C a chance to release the lock."（訳: B が実行され、B は C より優先度が高いので C をプリエンプトするが、そうすることで実際にはより優先度の高いプロセスである A をプリエンプトしていることになる。……B は CPU を占有し、C にロックを解放する機会を決して与えない）。これが*非有界の*優先度逆転である — A の待ち時間は C のクリティカルセクションの長さではなく、A が実際に欲しいロックとは何の関係もない B の振る舞いによってしか制限されない。常に実行可能な最高優先度のタスクを走らせるスケジューラが、正しく、設計どおりに動作してこの結果を生む — バグはスケジューラにあるのではない。

## きっかけ {#the-trigger}

2006年の時点で、優先度継承 mutex は新しいアイデアではなかった — Ingo Molnar と Thomas Gleixner がメインライン実装のマージを提案した頃には、この仕組みはリアルタイムシステムの文献やツリー外の Linux `-rt` パッチに何年も前から存在していた。Linus Torvalds の反対は率直かつ公開のもので、2005年12月16日の LKML 投稿でこう述べている。"Friends don't let friends use priority inheritance. Just don't do it. If you really need it, your system is broken anyway."（訳: 友人には優先度継承を使わせるな。とにかくやるな。本当に必要なら、どのみちそのシステムは壊れている）。Jonathan Corbet による2006年4月の LWN 記事は、再開したマージの動きを紹介し、"too late for 2.6.17, but, if no real opposition develops, the PI-futex code might just find its way into a subsequent kernel."（訳: 2.6.17 には間に合わないが、本格的な反対が起きなければ、PI-futex のコードはその後のカーネルに入るかもしれない）と記している。

## 観測された挙動 {#observed-behavior}

最終的にマージされた仕組みは、C の*実効*優先度を、C が保持するロックを待っている者に、C がそれを保持している間だけ追従させることで逆転を解決する。A が C のロックでブロックすると、C は即座に A の優先度を継承するので、ブーストされた C より本当に優先度が低い B は、もはや C をプリエンプトできない。C がロックを解放した瞬間にブーストは取り除かれ、C は自分本来の優先度に戻る。カーネルの rt_mutex 設計ドキュメントは、これを単一のホップではなく*チェーン*として説明している。C 自身が D の保持するロックでブロックしていれば D も推移的に継承し、"if process G has the highest priority in the chain, then all the tasks up the chain... must have their priorities increased to that of G"（訳: プロセス G がチェーン内で最も高い優先度を持つなら、チェーンをさかのぼるすべてのタスクの優先度を G のそれまで引き上げなければならない）。チェーンは合流（複数の待機者が同じロックに集まる）しうるが、分岐することはない。タスクは一度に1つのロックでしかブロックできないからである。同じチェーンウォーク関数は逆方向も扱う。解放された、タイムアウトした、あるいはシグナルを受けた待機者は、別の特殊ケースではなく、同一のロジックを起動してデブーストを行う。

## なぜ起きたか {#why-it-happened}

これは誤りについての話ではない — リアルタイムシステムにおける、実在しよく理解された問題についての話であり、その問題には同じくよく理解された*ユーザー空間のリアルタイム*上の答えがあった。争点は特に、*汎用カーネル*がそもそもその仕組みを抱えるべきかどうかにあった。Torvalds の反論は、優先度継承が機能しないというものではなかった。それに手を伸ばすシステムの多くは、PI が単に取り繕うだけの設計上の問題を抱えており、それをカーネルに組み込めばまさにそうした取り繕いを招く、という判断だった。この機能は結局マージされたが、具体的なユーザー空間の利用者を中心に狭く範囲を絞ったものだった。glibc の `pthread_mutex` の優先度継承サポートであり、これは完全にユーザー空間のポーリングだけで組み立てたものではなく、本物のカーネルレベルの PI プリミティブを必要としていた。

## 解決 {#resolution}

`23f78d4a03c5`（"[PATCH] pi-futex: rt mutex core"、Ingo Molnar）は2006年6月27日に `kernel/rtmutex.c` を導入し、同日に futex 側の半分である `c87e2837be82`（"[PATCH] pi-futex: futex_lock_pi/futex_unlock_pi support"）が続いた。これにより `FUTEX_LOCK_PI`/`FUTEX_UNLOCK_PI` 操作が追加され、ユーザー空間の `pthread_mutex` に本物のカーネル裏付けの優先度継承プリミティブが与えられた。これは、同じ年の早い時期に別途マージされた「robust futex」（所有者の死亡処理）の作業とは別物であり、それを補完するものである。どちらも 2.6.17 の直後に開いたマージウィンドウで入り、2006年9月20日リリースの **Linux 2.6.18** で出荷された — LWN の記事が「その後のカーネル」と正しく予想してから約5か月後である。

カーネル自身の `pi-futex.rst` ドキュメントは、結果として得られた高速パスを平易に説明している。"in the user-space fastpath a PI-enabled futex involves no kernel work (or any other PI complexity) at all"（訳: ユーザー空間の高速パスでは、PI 対応 futex はカーネルの処理（やその他の PI の複雑さ）を一切伴わない）— 通常のアトミックな compare-and-swap であり、競合がなければ非 PI の futex とコストは同一である。カーネルの rt_mutex 機構に触れるのは、低速パス（実際の競合）だけである。

## 得られた教訓 {#what-it-taught-us}

**メンテナがある仕組みを全面的に拒絶することと、その仕組みが最終的にマージされることは矛盾しない — それらはしばしば、同じ設計プロセスが正しく機能している姿である。** 2.6.18 で出荷されたものは、汎用の「優先度逆転を直すハンマー」ではなく、1つの具体的な利用者 — glibc の PI 対応 `pthread_mutex` サポート — を中心に厳密に範囲を絞ったものであり、それが実装が現在のように狭いままである大きな理由である。

**非有界の優先度逆転を解決すると、「優先度」の意味が、ちょうど1つの境界で変わる。より高優先度のタスクがあなたを待ち始めた瞬間である。** rt_mutex 設計のチェーンウォークは、そのブーストにかかる作業量を慎重に制限している（チェーンをたどる間に同時に保持するロックは最大2つ）— これはスケジューリングの正しさに関する修正であると同時に、単なるアルゴリズム上のアイデアにとどまらず、独自の資源上限のエンジニアリングを伴う必要があった。

!!! warning "注意すべきパターン"
    優先度逆転は、どこにもバグがなくても起こる — 個々のコンポーネント（スケジューラ、ロック、無関係な中優先度タスク）がすべて設計どおりに動いていても、システムは高優先度タスクを無期限に飢餓状態にする。リアルタイムあるいはレイテンシに敏感なワークロードが低優先度のコードとロックを共有しており、システム上に無関係な中優先度の作業が少しでも存在するなら、近くの何かが「壊れている」かどうかではなく、そのロックに優先度継承が必要かどうかを問うこと。

## 関連項目 {#see-also}

- [ロックの概要](../README.md) — mutex、spinlock、そしてその中で rt_mutex が占める位置
- [恒久的なフォールトへの答えを持たなかった PI-Futex の修正処理](pi-futex-fixup-owner-uaf.md) — このページで構築過程を説明した機構における正しさのバグ
- [Towelroot：ペアの requeue 側に欠けていたチェック](towelroot-futex-requeue.md) — requeue パスにおける、2つ目の PI-futex の正しさのバグ

## 外部参考文献 {#external-references}

- [git.kernel.org: 23f78d4a03c5](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=23f78d4a03c53cbd75d87a795378ea540aa08c86) — "[PATCH] pi-futex: rt mutex core"、`kernel/rtmutex.c` を導入したコミット
- [git.kernel.org: c87e2837be82](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c87e2837be82df479a6bae9f155c43516d2feebc) — "[PATCH] pi-futex: futex_lock_pi/futex_unlock_pi support"、futex 側の統合
- [LWN: Priority inheritance in the kernel](https://lwn.net/Articles/178253/) — Jonathan Corbet、2006年4月3日
- [LWN: Linus Torvalds の2005年12月の LKML 投稿のミラー](https://lwn.net/Articles/178258/) — "Friends don't let friends use priority inheritance"
- [カーネルドキュメント: RT-mutex implementation design](https://docs.kernel.org/locking/rt-mutex-design.html) — Steven Rostedt の設計ドキュメント。上記の優先度逆転と PI チェーンの説明の出典
- [カーネルドキュメント: Lightweight PI-futexes](https://docs.kernel.org/locking/pi-futex.html) — 上記の高速パスの引用の出典
