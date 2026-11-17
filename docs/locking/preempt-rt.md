# PREEMPT_RT: メインラインまでの20年 {#preempt_rt-twenty-years-to-mainline}

> 2004年に始まり、20年を要して、ついにLinux 6.12で完全なメインラインのカーネルオプションとなったパッチセット。その理由は、長らく特定されていたある1つの障害がついに解消されたことにある

## 概要 {#the-short-version}

`CONFIG_PREEMPT_RT`は、Linuxを、平均的なケースで良好な性能を出すだけでなく、上限のある**最悪ケースの応答時間**を目指すカーネルに変える。その上限が実測レイテンシの数値でどのようなものかは、[リアルタイムLinuxチューニングガイド](../sched/rt-tuning.md#what-real-time-means-on-linux)を参照のこと。これは、spinlockをプリエンプト可能で優先度継承を行う`rt_mutex`に置き換え（[mutexとrt_mutex](mutex.md#rt_mutex-priority-inheritance-mutex)を参照）、割り込みハンドラをプリエンプト可能なスレッドとして実行させ、カーネルに残るわずかな長いプリエンプト不可区間を分割することで実現される。この仕組み自体は何年も前から成熟しており、部分的にはツリー内に存在していた。カーネル**6.12**（2024年11月リリース）で変わったのは、「PREEMPT_RTがついに動くようになった」よりも範囲が狭く、そしてより興味深いことである。それは、長らく特定されていたある1つの障害、`printk()`がついに解消された話であり、これによって初めて実際のアーキテクチャでこのオプションを有効にできるようになった。

## 20年の年表 {#the-twenty-year-timeline}

プロジェクトの年齢は誇張ではない。カーネル開発者のSebastian Andrzej Siewiorは、メインラインで`PREEMPT_RT`をついに有効にするパッチシリーズを2024年9月6日に投稿した。その投稿に関するLWNの記事は、次の一文で始まる。["Work on realtime preemption for the Linux kernel got its start almost exactly 20 years ago (though it had its roots in earlier work, of course)"](https://lwn.net/Articles/989212/)（訳: Linuxカーネルのリアルタイムプリエンプションの作業は、ほぼちょうど20年前に始まった（もちろん、その根はさらに以前の作業にあるが））。これにより、プロジェクトの始まりはおよそ2004年となる。

その間の年月は、LWNによってリアルタイムに記録されている。[2009年のThomas GleixnerとIngo Molnarへのインタビュー](https://lwn.net/Articles/319544/)は、その時点ですでに取り組み開始から何年も経っていたが、リアルタイム作業のメインライン化の進捗が「ここ1年ほど」「鈍化していた」ことに触れるところから始まる。その後2人は「新しいリアルタイムプリエンプションツリーと、新たに活性化した開発の取り組み」を携えて再び姿を現し、Gleixnerは遅れの理由を、先にx86アーキテクチャの統合を整えることに時間を費やしていたためだと説明している。GleixnerとSiewiorはいずれも、当時も今も**Linutronix**のエンジニアである。LinutronixはリアルタイムLinuxのコンサルティング企業で、プロジェクトの歴史の大部分を通じてPREEMPT_RTのコアチームの多くを雇用してきた。2024年のパッチ投稿には`linutronix.de`のアドレスがそのまま見える。

RTパッチセットの多くは、「PREEMPT_RTをマージする」という単一のイベントを待たずにメインラインに到達した。その一部は20年の間に少しずつ入っていった。スレッド化された割り込みハンドラ、汎用の`rt_mutex`基盤（その作業から生まれた具体的な歴史上のインシデントについては[PI-mutexの起源の物語](war-stories/pi-mutex-origin.md)を参照）、futexの優先度継承などがそれにあたる。2024年のマージの時点では、*ダウンストリームのツリー*としてのRTパッチセットは十分に実用的になっており、ベンダーはすでにその上で製品やディストリビューションを出荷していた。2024年の発表に対するLWNのコメントスレッドには、職場のARMボードでPREEMPT_RTを約6年使ってきたと報告するユーザーもいる。ツリー外に残っていたのは、縮小しつつも頑固な障害の集まりであり、それが*メインライン自体で*完全な構成を有効にすることを妨げていた。

## 最後の障害: printk {#the-last-blocker-printk}

カーネルメーリングリストに投稿されたパッチのカバーレターにあるSiewior自身の言葉は、何が足止めしていたかについて率直である。["The printk bits required for PREEMPT_RT are sitting in linux-next. This was the last known roadblock for PREEMPT_RT"](https://lwn.net/Articles/989211/)（訳: PREEMPT_RTに必要なprintkの部分はlinux-nextに入っている。これがPREEMPT_RTにとって既知の最後の障害だった）。

printkが引き起こした問題は偶発的なものではなく構造的なものであり、修正されるずっと前から障害として特定されていた。2022年のLinux Plumbers Conferenceでのprintkの議論に関するLWNの記事は、「printk()は、RT_PREEMPTパッチが完全にマージされる前に変更が必要な最後のピースである」とはっきり述べている。それは実際にそうなる2年前のことである（[LWN](https://lwn.net/Articles/909980/)）。リアルタイムプリエンプションの前提そのものは、どのコードパスも、カーネルパニックがバックトレースを出力しようとする場合のような緊急時に起動されるパスでさえも、高優先度タスクを無期限にブロックできてはならないということである。しかし`printk()`は歴史的にまさにそれを行っていた。任意のCPUが単一のグローバルなコンソールロックを取り、上限のない時間をかけて出力を行うことができた。その中には、RTのルールでは何も長時間実行したりスリープしたりしてはならないとされるコンテキストからのものも含まれていた。

John Ognessがその2022年の議論で提示し、Thomas Gleixnerとともに開発した設計は、既存のグローバルなコンソールロックにはまったく手を付けなかった。代わりに、新しい（「nbcon」）モデルに参加したコンソールだけが使う、新しい別個のコンソールごとのロックをその横に追加した。nbconコンソールを通じて出力したい各CPUは優先度（`include/linux/console.h`で定義された`NBCON_PRIO_NORMAL`、`NBCON_PRIO_EMERGENCY`、`NBCON_PRIO_PANIC`）を宣言する。そして引き継ぎチェック（出荷版では`nbcon_can_proceed()`、2022年の元の提案では`console_can_proceed()`と呼ばれていた）により、高優先度のメッセージは低優先度のメッセージの終了を待たず、出力の途中でそれを横取りできる。一方、上限のないI/O作業そのものは、その場でのインライン出力ではなく、能動的に出力を行うカーネルスレッドが担う（[LWN](https://lwn.net/Articles/909980/)）。nbconを採用しないレガシーなコンソールは、元のグローバルロック（`kernel/printk/printk.c`の`console_sem`で、6.12でもまだ存在する）を使い続ける。6.12では`printk_legacy_kthread`も追加され、すべてのドライバを一斉に移行させることなく、そのロックを最もRTと相性の悪いパスから外せるようになった。

そのピースがついに揃ったことで、Siewiorのメッセージは直接的で具体的な効果を正確に述べた。["With the printk bits merged, PREEMPT_RT could be enabled on X86, ARM64 and Risc-V"](https://lwn.net/Articles/989211/)（訳: printkの部分がマージされたことで、PREEMPT_RTはX86、ARM64、Risc-Vで有効にできるようになった）。これが実際の出来事である。新しい機能が書かれたのではなく、必要な他のすべてをすでに済ませていた最後の3つのアーキテクチャに対して、Kconfigのゲートが外れたのである。

## カーネルツリーで実際に切り替わったもの {#what-actually-flipped-in-the-kernel-tree}

`PREEMPT_RT`のKconfigオプション自体は6.12で新しく入ったものではない。何年も前から存在しており、`kernel/Kconfig.preempt`の中で`depends on EXPERT && ARCH_SUPPORTS_RT`によってゲートされていた（カーネル5.10まで遡って変化がないことを確認済み）。変わったのは、どのアーキテクチャが`ARCH_SUPPORTS_RT`を宣言するかである。6.11と6.12のタグの間で`arch/x86/Kconfig`、`arch/arm64/Kconfig`、`arch/riscv/Kconfig`の差分を取ると、3つのアーキテクチャすべてが6.12で初めて`select ARCH_SUPPORTS_RT`の行を追加していることがわかる。そのリリース以前は、このオプションはソースツリーに存在してはいたが、メインラインではどのアーキテクチャでも実際には有効にできなかった。これはSiewior自身の説明と完全に一致する。printkの作業が、これら3つのアーキテクチャがサポートを宣言できなかった残り1つの理由を取り除いたのである。

すべてのアーキテクチャが一度にそこへ到達したわけではない。同じパッチのカバーレターには、「ARM and POWERPC have a few essential patches left」（ARMとPOWERPCには不可欠なパッチがいくつか残っている）こと、そしてSiewiorが「lost track of MIPS」（MIPSの状況を把握できなくなっていた）ことが記されている。これは、6.12における「PREEMPT_RTのマージ」が特にx86、arm64、RISC-Vを意味し、全アーキテクチャで利用可能になったわけではないことを思い出させる。その一覧はその後増えている。LoongArchは6.13で`select ARCH_SUPPORTS_RT`を追加した。32ビットARMは開発ツリーにキューされているが、本稿執筆時点ではタグ付きリリースにはまだ入っていない。powerpcとMIPSはまだ持っていない。

## これが意味しないこと {#what-this-doesnt-mean}

メインラインに入ったからといって、`PREEMPT_RT`がカーネルのデフォルトのプリエンプションモデルになるわけでも、ほとんどのユーザーが`CONFIG_PREEMPT_RT=y`を設定するようになるわけでもない。これは依然として`EXPERT`でゲートされた明示的なビルド時の選択である。最悪ケースのレイテンシと引き換えにスループットを差し出すものであり、ほとんどの汎用システムが望まないトレードオフだからである。RTカーネルをビルドした後、実際に設定して運用することが何を伴うかについては[リアルタイムLinuxチューニングガイド](../sched/rt-tuning.md)を、RTワークロードが通常`PREEMPT_RT`と組み合わせるスケジューリング側については[RTスケジューラ: SCHED_FIFOとSCHED_RR](../sched/rt-scheduler.md)を参照のこと。

6.12のマージで話が終わったわけでもない。`PREEMPT_LAZY`が6.13で別のプリエンプションモデルの選択肢（`kernel/Kconfig.preempt`）として入った。これは「基本的には完全プリエンプションと同様だが、`SCHED_NORMAL`タスクのプリエンプトにはそれほど積極的でない」もので、`PREEMPT_RT`そのものを置き換えるのではなく、完全プリエンプションで失われたスループットの一部を取り戻すことを狙っている。

## 参考文献 {#further-reading}

### LWNの記事 {#lwn-articles}

- [The realtime preemption end game — for real this time](https://lwn.net/Articles/989212/) — 2024年9月のパッチシリーズに関するJonathan Corbetの記事。「almost exactly 20 years」という表現を含む
- [Allow to enable PREEMPT_RT.](https://lwn.net/Articles/989211/) — Sebastian Andrzej Siewiorがカーネルメーリングリストに投稿した元のパッチのカバーレター（2024年9月6日）
- [Interview: the return of the realtime preemption tree](https://lwn.net/Articles/319544/) — プロジェクトの歴史とペースに関する、Thomas GleixnerとIngo Molnarへの2009年のインタビュー
- [A discussion on printk()](https://lwn.net/Articles/909980/) — 2022年のLinux Plumbers Conferenceでの議論に関するJake Edgeの記事。この議論から、PREEMPT_RTの障害を取り除いたコンソールごとのロックと優先度による引き継ぎの再設計が生まれた

### カーネルソース {#kernel-source}

- [kernel/Kconfig.preempt](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/Kconfig.preempt) — `PREEMPT_RT`シンボルと、その`depends on EXPERT && ARCH_SUPPORTS_RT`ゲート
- [arch/Kconfig](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/Kconfig) — 汎用の`ARCH_SUPPORTS_RT`シンボル自体が定義されている場所
- [arch/x86/Kconfig](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/Kconfig)、[arch/arm64/Kconfig](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/Kconfig)、[arch/riscv/Kconfig](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/riscv/Kconfig) — カーネル6.12時点でいずれも`select ARCH_SUPPORTS_RT`を持つ
- [kernel/printk/nbcon.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/printk/nbcon.c) — printk修正の背後にある引き継ぎチェック、`nbcon_can_proceed()`
- [include/linux/console.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/console.h) — `enum nbcon_prio`（`NBCON_PRIO_NORMAL`/`EMERGENCY`/`PANIC`）
- [kernel/printk/printk.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/printk/printk.c) — nbcon以外のコンソールが今も使う元のグローバルなコンソールロックである`console_sem`と、`printk_legacy_kthread`

### 関連ページ {#related-pages}

- [mutexとrt_mutex](mutex.md) — RTがspinlockの置き換えに頼る優先度継承mutexの仕組み
- [優先度逆転とPI mutex](../sched/pi-mutexes.md) — 優先度継承の背後にある仕組みの全体
- [リアルタイムLinuxチューニングガイド](../sched/rt-tuning.md) — `PREEMPT_RT`カーネルを実際に運用・チューニングする
- [RTスケジューラ: SCHED_FIFOとSCHED_RR](../sched/rt-scheduler.md) — RTワークロードが使うスケジューリングクラス
- [PI-mutexの起源の物語](war-stories/pi-mutex-origin.md) — このプロジェクトが生んだ優先度継承の作業にまつわる歴史上のインシデント
