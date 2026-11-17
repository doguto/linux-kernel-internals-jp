# kGraftとkpatch: 2つのライバルはいかにして1つのアップストリームになったか {#kgraft-vs-kpatch-how-two-rivals-became-one-upstream}

> SUSEとRed Hatは2014年、本質的に異なる技術モデルに基づくライブパッチ機構をそれぞれ構築した。マージされた結果は両方を取り込み、実際に完成するのは2017年までかかった

## 数週間違いで現れた、同じ問いへの2つの答え {#two-answers-to-the-same-question-weeks-apart}

両プロジェクトが解決しようとした問題は新しいものではなかった。Kspliceのパッチセットは2008年からカーネルのライブパッチを提供していたが、メインラインに入ることはなく、2011年にOracleがその開発元企業を買収した後はLinuxエコシステムからほぼ姿を消し、根本的なニーズは満たされないまま残された（[LWN, "The initial kGraft submission"](https://lwn.net/Articles/596854/)）。2014年、SUSEとRed Hatはその穴を埋める新たな機構をそれぞれ独自に構築してオープンソース化したが、両者に互換性はなかった。

先に発表したのはSUSEの**kGraft**である。[LWNの記事は2014年2月3日付](https://lwn.net/Articles/584016/)で、SUSE自身の説明として「ftraceと、関数ヘッダ内のmcountベースの予約領域、jumplabelでも使われるINT3/IPI-NMIによるパッチ適用、そしてカーネルを停止させる必要のないRCU風のコード更新」の上に構築したと引用している。実際のコードはその約3か月後に続いた。Jonathan Corbetによる[「the initial kGraft submission」](https://lwn.net/Articles/596854/)の記事（2014年4月30日）は作者について明快で、「KGraftはJiří KosinaとJiří Slabyの仕事であり、2人ともSUSEに勤務している」と書いている。基本的なkGraftの機構は、Corbet自身の留保付きの表現を借りれば「わずか600行のパッチ」であった。これはその最初のパッチ自体についての記述であり、kGraftが後に含むことになるすべてについての記述ではない。

Red Hatの**kpatch**は数日のうちに続いた。Corbetによる[「the first kpatch submission」](https://lwn.net/Articles/597407/)の記事（2014年5月7日）は冒頭でそのタイミングに直接触れ、「先週、SUSEのkGraftライブパッチ機構が投稿された。その直後、Red Hatの開発者たちが競合するkpatch機構を携えて名乗り出た」と述べ、2人の作者を「kpatch developer Josh Poimboeuf」「kGraft developer Jiri Kosina」と直接名指ししている。Poimboeuf自身のカバーレター（[「\[RFC PATCH 0/2\] kpatch: dynamic kernel patching」](https://lkml.iu.edu/hypermail/linux/kernel/1405.0/00278.html)、2014年5月1日にカーネルメーリングリストへ投稿）は「JiriがkGraftのパッチを投稿したので、kpatchという別のライブパッチソリューションを共有したいと思った」という一文で始まる。（対照的に、後述するSeth Jenningsのコミットは、後に両プロジェクトのコードが流れ込んだ、別個の*マージされた*livepatchコアであり、kpatchそのものではない。）両機構とも同じ基盤プリミティブ、すなわちftraceの関数エントリフック（[このサイトのKLPページで詳しく扱っている](klp.md)のと同じ機構）を使っていたが、難しい問題、つまり*タスクを古い関数から新しい関数へ切り替えて安全なのはいつか*を、本質的に異なる方法で解いていた。

## 「いつ切り替えれば安全か」への2つの異なる答え {#two-different-answers-to-when-is-it-safe-to-switch}

統合の取り組みを振り返ったLWNの後の記事は、技術的な違いを正確に説明している。

> As originally developed, kpatch worked by calling `stop_machine()` to bring the entire system to a halt. It then would check the stack of every process in the system to ensure that none are running within the function(s) to be patched; if the affected functions are not currently running, the patch can proceed, otherwise the operation fails. KGraft, instead, used a "two-universe" model where every process in the system is switched from the old code to the new at a "safe" point. The most common safe point is exit from a system call; at that point, the process cannot be running in any kernel code.
>
> （訳: 当初開発されたkpatchは、`stop_machine()`を呼んでシステム全体を停止させることで動作した。その後、システム内の全プロセスのスタックを検査し、パッチ対象の関数内で実行中のものがないことを確認する。対象関数が現在実行中でなければパッチを進められ、そうでなければ操作は失敗する。一方kGraftは「2つの宇宙（two-universe）」モデルを使い、システム内の各プロセスを「安全な」地点で古いコードから新しいコードへ切り替えた。最も一般的な安全地点はシステムコールからの復帰であり、その時点でプロセスがカーネルコードのどこかを実行していることはありえない。）

— [LWN, "A rough patch for live patching"](https://lwn.net/Articles/634649/)

どちらのアプローチも厳密に優れていたわけではなく、それぞれが相手にはない現実の失敗モードを抱えていた。kpatchのスタック検査は精密さをもたらした。タスクは安全であることが証明できた瞬間にパッチを適用できる。その代償は、全体を止めるstop-the-worldの一時停止と、明確なハード失敗のケースであった。Corbetの言葉では「カーネルコードが対象関数のいずれかの中で実行中であれば、kpatchは単に失敗する」。そして一部の関数（例として挙げられたのは`schedule()`、`do_wait()`、`irq_thread()`）は事実上常にカーネルのどこかで実行中であり、kpatch単独ではそれらに触れるパッチを決して適用できないことを意味した。ただしCorbetは実際の影響範囲も示しており、「典型的なシステムでは、このようにライブパッチを阻む関数はおそらく数十個程度だろう。カーネル内の何千もの関数のうちのごく小さな部分集合である」と述べている（[LWN](https://lwn.net/Articles/597407/)）。kGraftのタスクごと・システムコール復帰時の移行はこのハード失敗を回避したが、その代償は単なる管理の手間にとどまらなかった。タスクはシステムコール復帰のような安全地点でしか遷移しないため、「すべてのプロセスを安全な場所で捕まえる過程には際限のない時間がかかりうり、その間システムは奇妙な中間状態に置かれる」（[LWN, "A rough patch for live patching"](https://lwn.net/Articles/634649/)）。この保証には実装上の現実的なコストもあった。kGraftは`kthread_should_stop()`を手作業で修正し、その呼び出しを行わないカーネルスレッドには`kgr_task_safe()`呼び出しを手で挿入していた。さらに未解決のエッジケースもあった。I/Oでブロックしているプロセスは「長時間カーネル内にとどまる可能性があり、graft操作の後始末を妨げる」が、投稿されたパッチセットには強制遷移の機構は実際には存在しなかった（[LWN, "The initial kGraft submission"](https://lwn.net/Articles/596854/)）。

## 統合: LPC 2014と、マージ前の提案 {#the-unification-lpc-2014-and-a-proposal-before-the-merge}

2つのプロジェクトは、ドイツのデュッセルドルフで10月15〜17日に開催された[2014 Linux Plumbers Conference](https://blog.linuxplumbersconf.org/2014/)で公の場で収束した。最終的にマージされたパッチセットに付けたSeth Jennings自身のカバーレターが、そのことを直接述べている。「このソリューションはLPC 2014のLive Patching Mini-conferenceで議論された」（[LWN, "Kernel Live Patching"](https://lwn.net/Articles/619390/)、2014年11月6日付のパッチ投稿のミラー）。

この11月の投稿（JenningsからJosh Poimboeuf、Jiri Kosina、Vojtech Pavlik、そしてftraceメンテナのSteven Rostedtに送られた）こそが実際の統合提案であり、その範囲についての記述は、何をして何をしないかについて率直である。「これはkpatch \[...\]とkGraft \[...\]の最大公約数的な機能セットを表しており、どちらの方法でビルドされたパッチも受け付けられる」。また、この最初のバージョンが一貫性モデルをまったく実装していないことも明示している。同じ但し書きは、パッチが実際にマージされた際のコミットメッセージに一字一句そのまま引き継がれており、以下に全文を引用する。

## 最初に実際に入ったもの: 一貫性モデルはまったくなし {#what-actually-landed-first-no-consistency-model-at-all}

アップストリームの`kernel/livepatch/`となるコードは、2014年12月16日にSeth Jenningsが作成し、その6日後の2014年12月22日に、SUSEのJiri Kosinaがコミッタとして`jikos/livepatching`サブシステムツリーへコミットした。パッチ自身のABIドキュメントには`KernelVersion: 3.19.0`と記されており、当時開発中だったリリースを狙っていたことがわかる。ただし、すぐにメインラインに入ったわけではない。Linusが`jikos/livepatching`ツリーをpullしたのは2015年2月10日、4.0のマージウィンドウ中であり、そのため`kernel/livepatch/core.c`はLinux **v3.19**タグには存在せず、**v4.0**（2015年4月12日リリース）には存在する。コミットメッセージ自体が、この最初の版が受け入れた現実的な制限を明示している。

> This first version does not implement any consistency mechanism that ensures that old and new code do not run together. In practice, ~90% of CVEs are safe to apply in this way, since they simply add a conditional check. However, any function change that can not execute safely with the old version of the function can *not* be safely applied in this version.
>
> （訳: この最初のバージョンは、古いコードと新しいコードが同時に実行されないことを保証する一貫性機構を一切実装していない。実際には、CVEの約90%は単に条件チェックを追加するだけなので、この方法で安全に適用できる。しかし、古いバージョンの関数と一緒に安全に実行できない関数変更は、このバージョンでは安全に適用*できない*。）

— [commit `b700e7f03df5`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b700e7f03df5d92f85fa5247fe1f557528d3363d), Seth Jennings

このコミットに付けられたKosina自身の注記は、パッチの作者がJennings（Red Hat）であるにもかかわらずSUSEが共同著作権者になった理由を説明している。「Seth Jenningsのこの元のパッチに取り込まれた貢献の数を考慮し、メールで議論したとおり、SUSEの著作権表示も追加する」。Kosina（SUSE）はコミッタとして記録されている。コミットのトレーラーは順に、SUSEとRed Hatの共同名簿のように読める。まずJenningsとPoimboeuf（Red Hat、kpatch）のSigned-off-by、次にMiroslav Beneš、Petr Mladek（ともにSUSE）、Masami Hiramatsu（日立。SUSEでもRed Hatでもない）のReviewed-by、そしてBeneš、Mladek、Kosina（全員SUSE）による2回目のSigned-off-byが続く。BenešとMladekは2回登場しており、パッチのレビューとサインオフの両方を行っている。

## 本当のハイブリッドが入るまでには2年以上かかった {#the-real-hybrid-took-over-two-years-to-land}

問題を先送りするのではなく、両プロジェクトの手法を実際に組み合わせた本格的な一貫性モデルは、2015年2月9日に提案された。Josh PoimboeufがRFCパッチセットを投稿したのは、livepatchコア自体が実際にメインラインに入る*前日*である。LWNは、このパッチセットが「kGraftの2つの宇宙モデルを保持しつつ、プロセスを新しいコードへ切り替える作業を加速するためにkpatchのスタックトレース検査を使う」と報じた（[LWN, "A rough patch for live patching"](https://lwn.net/Articles/634649/)）。しかし、これをアップストリームに入れるには最初のマージよりはるかに長い時間がかかった。

このタスクごとの一貫性モデルを実装するファイルである`kernel/livepatch/transition.c`は、Linux **v4.11**タグには存在せず、**v4.12**には存在する。つまりこの機構が実際に出荷されたのは2017年7月2日リリースのカーネル4.12であり、RFCと4.0でのメインラインマージ（この2つの出来事はわずか1日違い）の両方から約2年半後のことであった。最終的なコミットのメッセージは、その系譜を曖昧さなく述べている。

> This code stems from the design proposal made by Vojtech \[1\] in November 2014. It's a hybrid of kGraft and kpatch: it uses kGraft's per-task consistency and syscall barrier switching combined with kpatch's stack trace switching.
>
> （訳: このコードは、2014年11月にVojtechが行った設計提案 \[1\] に由来する。kGraftとkpatchのハイブリッドであり、kGraftのタスクごとの一貫性とシステムコール境界での切り替えを、kpatchのスタックトレースによる切り替えと組み合わせて使う。）

— [commit `d83a7cb375ee`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d83a7cb375eec21f04c83542395d08b2f6641da2), "livepatch: change to a per-task consistency model," Josh Poimboeuf, 2017年2月13日作成

この脚注「\[1\]」は特定の文書を指している。Vojtech Pavlik自身の設計提案で、[2014年11月7日にカーネルメーリングリストへ投稿された](https://lkml.kernel.org/r/20141107140458.GA21774@suse.cz)ものである。これはその1日前のJenningsによる統合カバーレターとは別の投稿であり、両者は混同しやすいが別のことを述べている（Jenningsは既存の2つの機構をそのまま統合することを提案し、Pavlikは`d83a7cb375ee`が2年半後に実際に実装したハイブリッド設計を提案した）。注目すべきことに、最終コミットには`Acked-by: Ingo Molnar <mingo@kernel.org> # for the scheduler changes`も付いている。後述するように、ライブパッチという前提そのものが誤っていると主張したのと同じMolnarである。

具体的には、出荷された機構（`kernel/livepatch/transition.c`の`klp_try_switch_task()`、`klp_check_stack()`）はまずkpatchのスタック検査を試みる。スリープ中のタスクのスタックフレームがどれもパッチ対象関数内になければ、そのタスクは即座に切り替えられる。そして予測可能な形でスリープしないタスクについては、kGraftのシステムコール復帰・割り込み復帰という安全地点にフォールバックする。[このサイトのKLP一貫性モデルのページ](klp-consistency.md)では、この機構（`patch_state`フィールド、遷移状態、フォールバックの規則）を技術的に詳しく扱っている。このページは、それが単一の発明ではなく、競合する2つの設計の統合としていかにして生まれたかという物語である。

## なぜ空白期間があったのか: まだ誰もスタックアンワインダを信用していなかった {#why-the-gap-nobody-trusted-the-stack-unwinder-yet}

2年半の空白は惰性によるものではなかった。Poimboeufのハイブリッド設計はほぼ即座に、現実的で具体的な技術的反論に直面し、それらは交渉で片付くものではなく、実際に解決するまでに何年もかかった。2015年2月のCorbetによるこの提案の記事は、反論を正確に挙げている。kpatchのスタックトレース検査を採用するということは、スタックアンワインダにデバッグ出力のためだけでなく*正しさ*のために依存することを意味し、Peter Zijlstraはそれを率直に指摘した。

> So far stack unwinding has basically been a best effort debug output kind of thing, you're wanting to make the integrity of the kernel depend on it. You require an absolute 100% correctness of the stack unwinder — where today it is; as stated above; a best effort debug output thing. That is a *big* change.
>
> （訳: これまでスタックアンワインドは基本的にベストエフォートのデバッグ出力のようなものだった。あなたはカーネルの完全性をそれに依存させようとしている。スタックアンワインダに絶対的な100%の正しさを要求している。現状は、上で述べたとおり、ベストエフォートのデバッグ出力に過ぎないのに。これは*大きな*変更だ。）

— [LWN, "A rough patch for live patching"](https://lwn.net/Articles/634649/)

Ingo Molnarも別の角度から同じ懸念を表明した。カーネル空間で実行中のプロセスから真に信頼できるスタックトレースを得ることは難しいと知られており、バグとアーキテクチャごとの癖の現実の歴史があった。そして彼はさらに踏み込み、両プロジェクトの設計を完全に放棄することを提案した。「両者は実装においても設計においても根本的に誤っており、それによって（不本意ながら）セキュリティシアターの延長された腕になっていると思う」と述べ、一貫性モデルそのものの代替として、状態を保持したkexec（システム状態を保存し、新しいカーネルを起動し、状態を復元する）を提案したのである（[LWN](https://lwn.net/Articles/634649/)）。この提案は採用されなかったが、意見の相違が表面的なものではなかったことを裏付けている。尊敬されるカーネル開発者が、どちらのプロジェクトの手法を残すかについてのささいな言い争いではなく、ライブパッチという前提そのものが誤ったアプローチだと主張していたのである。

だからこそ、2017年の最終コミットはスタック検査の経路全体を、名前の付いた特定のケーパビリティの背後に置いている。「このオプションは、アーキテクチャが信頼できるスタック（`HAVE_RELIABLE_STACKTRACE`）を持つ場合にのみ利用できる」（[commit `d83a7cb375ee`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d83a7cb375eec21f04c83542395d08b2f6641da2)）。スタックアンワインドを、単なるベストエフォートではなく証明可能な形で信頼できるものにすること。それが「一貫性モデル」という項目の陰に隠れた、何年もかかる実際の前提条件だったのであり、何を作るべきかについて合意がなかったわけではない。ただし4.12のマージ時点で、実際にその基準を満たしていたのはx86_64だけであった。`arch/x86/Kconfig`は`HAVE_RELIABLE_STACKTRACE`を`X86_64 && FRAME_POINTER && STACK_VALIDATION`の条件の背後に置いており、arm64、powerpc、s390がサポートを宣言したのはその何年も後である。「2017年に完成した」とは、機構とそのx86_64向けの条件がその時に入ったという意味であり、初日からすべてのアーキテクチャで使えたという意味ではない。

4.12のマージは、この機構自体の進化の終わりでもなかった。同機構とともに出荷された`immediate`フラグ（一貫性モデルを必要としないパッチ向けに、一貫性モデルなしでパッチを適用するための抜け道）はLinux 4.16で削除された。停滞したタスクに対して管理者が手動で`SIGSTOP`/`SIGCONT`を送る回避策は、4.15で「偽シグナル（fake signal）」機構に置き換えられた。そして、このページで引用したコミットがまだ必要としていた個別の登録ステップを不要にしたアトミックリプレースは、5.1で入った。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/livepatch/core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/livepatch/core.c) — 最初にマージされたインターフェース（カーネル4.0）
- [kernel/livepatch/transition.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/livepatch/transition.c) — タスクごとの一貫性モデル（カーネル4.12）: `klp_try_switch_task()`、`klp_check_stack()`
- [commit b700e7f03df5](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b700e7f03df5d92f85fa5247fe1f557528d3363d) — 最初のマージ、Seth Jennings、2014年12月
- [commit d83a7cb375ee](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d83a7cb375eec21f04c83542395d08b2f6641da2) — タスクごとの一貫性モデル、Josh Poimboeuf、2017年

### LWNの記事 {#lwn-articles}

- [kGraft — live kernel patching from SUSE](https://lwn.net/Articles/584016/) — kGraftについての最初の記事、2014年2月
- [The first kpatch submission](https://lwn.net/Articles/597407/) — kpatchについての最初の記事、2014年5月
- [Kernel Live Patching](https://lwn.net/Articles/619390/) — Seth Jenningsによる2014年11月の統合提案（LWNのミラー）
- [A rough patch for live patching](https://lwn.net/Articles/634649/) — kpatchのスタック検査とkGraftの2つの宇宙モデルの技術的比較、およびPoimboeufのハイブリッド提案
- [livepatch: consistency model](https://lwn.net/Articles/632582/) — Josh Poimboeufによる2015年2月のRFCパッチセット（LWNのミラー）
- [Vojtech Pavlikの2014年11月の設計提案](https://lkml.kernel.org/r/20141107140458.GA21774@suse.cz) — `d83a7cb375ee`が実装したハイブリッド設計の直接の祖先

### 関連ページ {#related-pages}

- [カーネルライブパッチ（KLP）](klp.md) — kGraftとkpatchの両方が基盤としたftraceベースのリダイレクト機構
- [KLP一貫性モデル](klp-consistency.md) — このページの歴史がたどり着いた、出荷済みのタスクごとの一貫性機構
- [累積パッチとアトミックリプレース](klp-cumulative.md) — 同じフレームワークのその後の進化
