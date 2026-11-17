# sched_ext: BPFでプログラム可能なスケジューリング {#sched_ext-bpf-programmable-scheduling}

> プラガブルなスケジューラに対する長年の「ノー」が、Linux 6.12で「イエス」に変わった経緯

## sched_extとは {#what-it-is}

`sched_ext`（SCX）は6番目の[スケジューラクラス](scheduler-classes.md)である。他のクラスと違い、そのスケジューリングポリシーはカーネルにコンパイルされていない — 他のBPFオブジェクトと同様に、ユーザー空間からロード・アンロードされるBPFプログラムである。Linux向けに新しいCPUスケジューラを書くのに、もはやカーネルにパッチを当てて再ビルドする必要はない。決まったコールバック群に対してBPFプログラムを書き、それをロードすればよい。

ただし、優先度の面では`fair`、`rt`、`deadline`と対等ではない: カーネル自身のスケジューラクラスのリンク順（`include/asm-generic/vmlinux.lds.h`の`SCHED_DATA`）は、クラスを`stop → dl → rt → fair → ext → idle`の順に並べている。CPUが実行するタスクを`ext`に探しに行くのは、`deadline`、`rt`、`fair`のいずれにも実行可能なものがない場合だけである — sched_extは既存の3クラスと並ぶ位置ではなく、`idle`のすぐ上に位置する。

`kernel/sched/ext.c`はLinux v6.11のタグには存在せず、v6.12には存在する。これにより、このクラスが**Linux 6.12**（2024年11月）でマージされたことが確認できる。

## 長年の「ノー」 {#a-standing-no}

ロード可能でプラガブルなスケジューリングポリシーは新しいアイデアではない — Linus Torvaldsが以前に却下してきたものである。Tejun Heoが（David Vernet、Josh Don、Barret Rhodenとの共著で）sched_extのパッチセットを最初に投稿したとき、スケジューラのメンテナであるPeter Zijlstraの反応は率直だった:

> "I hate all of this. Linus NAK'ed loadable schedulers a number of times in the past and this is just that again -- with the extra downside of the whole BPF thing on top."
>
> — [Peter Zijlstra, replying to the first posted version](https://lwn.net/ml/linux-kernel/Y5b2btWFJeEfTyJg@hirez.programming.kicks-ass.net/)

（訳: 「こういうものは全部嫌いだ。Linusは過去に何度もロード可能なスケジューラにNAKを出しており、これもまたそれと同じだ — しかもBPF絡みという欠点まで上乗せされている。」）

この一文にそれまでの歴史がすべて詰まっている: これはプラガブルなスケジューリングの最初の試みではなく、それ以前の試みはすべてTorvalds自身によって却下されていた。反対の理由は実のところBPFではなかった — よりによってスケジューラというサブシステムを、サードパーティのポリシーに開放すること自体が問題だったのである。不正な動作をするスケジューラは、プログラムをクラッシュさせるだけでは済まない。マシン全体を固まらせることもありうる。

## 2024年の覆し {#the-2024-override}

それにもかかわらず、Torvaldsは — スケジューラのメンテナ自身の反対を押し切って — マージすることを決めた。LWNは2024年6月、彼が決断を下し、スケジューラのメンテナの判断を覆してマージするつもりであると報じ、Torvalds自身の理由づけを直接引用した:

> "I honestly see no reason to delay this any more. This whole patchset was the major (private) discussion at last year's kernel maintainer summit, and I don't find any value in having the same discussion (whether off-list or as an actual event) at the upcoming maintainer summit one year later, so to make any kind of sane progress, my current plan is to merge this for 6.11."

（訳: 「正直なところ、これ以上遅らせる理由はないと思う。このパッチセット全体は昨年のカーネルメンテナサミットにおける主要な（非公開の）議論だったし、1年後の次のメンテナサミットで（リスト外であれ実際のイベントとしてであれ）同じ議論をすることに価値を見出せない。だから、何らかのまともな進展を得るために、現時点では6.11でこれをマージするつもりだ。」）

Torvaldsは、これが正式な議題ではなく「（非公開の）」議論だったと明言している — そしてLWN自身が公開した[2023 Kernel Maintainers Summit](https://lwn.net/Articles/951847/)の記事もそれと整合する: その日に挙げられたセッションは、ファイルシステムのメンテナンス、Rustへのコミット、メンテナの燃え尽き、そしてメンテナの悩みに関するQ&Aであった。sched_extはその中にない。Torvaldsが言及している会話がどのようなものであれ、それはサミットの報告された議題の外で行われたものであり、「カーネルコミュニティ全体がメンテナサミットの正式な時間を丸ごとそれに費やした」という主張に比べれば弱い根拠である — それはオフレコの議論を繰り返したくないという彼自身の述べた理由であって、文書化された合意形成のプロセスではない。

しかも、この覆しは一度目ではうまくいかなかった。上のTorvaldsの発表は6.11を目標としていたが、[実際の6.11マージウィンドウに関するLWNの記事](https://lwn.net/Articles/982605/)は別の話を伝えている: "After that came a strongly worded discussion about whether that step was justified, or whether sched_ext needed more work first... sched_ext was not merged for 6.11."（訳: 「その後、その措置が正当だったのか、それともsched_extにはまずさらなる作業が必要だったのかについて、激しい議論が起こった……sched_extは6.11ではマージされなかった。」）この覆しは、よくある遅延ではなく、本物の実質的な反発を引き起こしたのである — `kernel/sched/ext.c`が入ったのは次のサイクル、2024年11月の6.12になってからだった。

## 実際の動作の仕組み {#how-it-actually-works}

BPFスケジューラは`struct sched_ext_ops`のインスタンスである — タスクのスケジューリングのライフサイクルの各時点でカーネルが呼び出すコールバックの集合である。`kernel/sched/ext.c`で定義されている実際の構造体には、（他のものに加えて）次のものが含まれる:

```c
/* kernel/sched/ext.c */
struct sched_ext_ops {
    s32  (*select_cpu)(struct task_struct *p, s32 prev_cpu, u64 wake_flags);
    void (*enqueue)(struct task_struct *p, u64 enq_flags);
    void (*dequeue)(struct task_struct *p, u64 deq_flags);
    void (*dispatch)(s32 cpu, struct task_struct *prev);
    void (*tick)(struct task_struct *p);
    void (*runnable)(struct task_struct *p, u64 enq_flags);
    void (*running)(struct task_struct *p);
    void (*stopping)(struct task_struct *p, bool runnable);
    void (*quiescent)(struct task_struct *p, u64 deq_flags);
    bool (*yield)(struct task_struct *from, struct task_struct *to);
    bool (*core_sched_before)(struct task_struct *a, struct task_struct *b);
    void (*set_weight)(struct task_struct *p, u32 weight);
    s32  (*init_task)(struct task_struct *p, struct scx_init_task_args *args);
    /* ... その間に、デバッグ用のdump()/dump_cpu()/dump_task()、cgroupのコールバック、
       cpu_online()/cpu_offline()などがある ... */
    s32  (*init)(void);
    void (*exit)(struct scx_exit_info *info);
};
```

1回の起床におけるライフサイクルは次のとおりである: `select_cpu()`がターゲットCPUを選び（スケジューラが独自のアイドルCPU探索を実装するならここである）、`enqueue()`がスケジューラの選んだ場所にタスクを置き、`dispatch()`はCPUのローカルディスパッチキューが空になるたびに呼ばれる — CPUが本当にアイドルになったときだけでなく、直前に実行していたタスクのタイムスライスが尽きたがまだ実行可能である場合も含む。スケジューラは自前のランキューのデータ構造を一から管理するわけではない — タスクを**ディスパッチキュー（DSQ）**、カーネル内の`struct scx_dispatch_q`の間で移動させるのである。DSQは厳密なFIFOでも優先度キューでもない。カーネル自身のドキュメントによれば、単一のDSQは同時に「FIFOとしても優先度キューとしても動作できる」。vtime順のエントリ（`scx_bpf_dispatch_vtime()`）は、常に通常のFIFOエントリの後に消費される。各CPUは、常にそこからタスクを実行する組み込みの**ローカルDSQ**（`SCX_DSQ_LOCAL`）を持ち、さらにデフォルトの置き場としてすべてのCPUが共有する組み込みの**グローバルDSQ**（`SCX_DSQ_GLOBAL`）がひとつある。スケジューラは、cgroupごとのキューやその他の独自のキューイングのために追加のDSQを自由に作成でき、それらは実際に実行されるためにCPUのローカルDSQへと「消費」される。

## 安全性モデル {#the-safety-model}

BPFプログラムにスケジューリングの判断を任せることが成り立つのは、壊れたプログラムがマシンをハングさせられない場合に限られる。それを強制する独立した仕組みが2つある:

**BPF検証器（verifier）**は、無制限にループしうるプログラム、アクセスすべきでないメモリにアクセスしうるプログラム、その他BPFの通常の安全性保証に違反しうるプログラムを、ロード時に拒否する — 他のすべてのBPFプログラムタイプを関門する検証器と同じものである。

**ウォッチドッグ**は、検証器では捕まえられない不正動作を捕捉する: BPFとしては完全に正当だが、実行可能なタスクをまったくディスパッチしないスケジューラである。`kernel/sched/ext.c`のウォッチドッグは、各ランキューの実行可能リストを定期的にたどり、各タスクがどれだけ待っているかを確認する。上限は`SCX_WATCHDOG_MAX_TIMEOUT` — 30秒である:

```c
/* kernel/sched/ext.c（簡略化） */
static bool check_rq_for_timeouts(struct rq *rq)
{
    list_for_each_entry(p, &rq->scx.runnable_list, scx.runnable_node) {
        if (time_after(jiffies, p->scx.runnable_at + scx_watchdog_timeout)) {
            scx_ops_error_kind(SCX_EXIT_ERROR_STALL,
                               "%s[%d] failed to run for %u.%03us", ...);
            return true;
        }
    }
    return false;
}
```

ストールが検出されても、即座に同期的にCFSへ切り替わるわけではない — `scx_ops_error_kind()`は無効化要求をカーネルスレッドのワークキューに積み（`kthread_queue_work()`）、そのワークが`scx_ops_disable_workfn()`を実行する。この関数の最初の実質的な処理は**バイパスモード**である: `scx_ops_bypass(true)`がすべてのランキューに`SCX_RQ_BYPASSING`の印を付け、sched_ext自身のディスパッチロジックはBPFスケジューラの判断を無視し、ハードコードされたグローバルFIFOを使い始める — これにより、他の何かのアンロードを待つことなく、積まれたワークが実行された時点で前進が保証される。同じ関数はその後、より時間のかかる完全な後始末を行う: SCXクラスに残っているすべてのタスクをたどり、コアスケジューラの`__setscheduler_class()`でスケジューラクラスを再計算する。この関数はまずデッドラインとRTの優先度を確認し、どちらにも該当せずSCX自体が取り壊されている場合にのみ`&fair_sched_class` — [EEVDF](eevdf.md) — にフォールスルーする。BPFスケジューラはアンロードされ、それが所有していたすべてのタスクは、新しいスケジューラがロードされるまでEEVDFに戻る。

ウォッチドッグにも検証器にもまったく依存しない、手動の非常口もある: `SysRq-S`キーシーケンスを押すと、実行中のBPFスケジューラが中止され、検出されたストールやスケジューラ起因のエラーと同様に、すべてのタスクが直ちにCFSに戻る。対になる`SysRq-D`は、スケジューラを終了させずにsched_extの内部デバッグ状態をダンプする。問題を失わずに診断するためのものである。

## 実世界での採用 {#real-world-adoption}

2024年のLinux Plumbers Conferenceで、Tejun Heoは仮定の話ではなく実際の導入例を紹介した: `scx_layered`はすでに**100万台を超えるマシン**で稼働しており、測定可能な性能向上をもたらしていた。`scx_lavd`はValveの**Steam Deck**への搭載に向かっていた。そして`scx_bpfland`は一般的なデスクトップ用途で有望な結果を示していた。2024年後半までに、sched_extのサポートはCachyOS、Arch Linux、Ubuntu、Fedora、Nix、openSUSEで提供されるようになった — これらのディストリビューションでは、別のスケジューラを試すのはカーネルの再ビルドではなく、パッケージのインストールとコマンド1つで済む。

## 今後の方向性 {#where-its-heading}

Heo自身が描く今後の作業は**組み合わせ可能性（composability）**である: ひとつのプログラムがすべての判断を握るのではなく、複数のBPFスケジューラを協調させることである。掲げられている目標は、あるスケジューラがタイムスライスのポリシーに専念し、別のスケジューラが負荷分散やパーティショニングを担当すること、そして最終的にはスケジューラをcgroup階層に沿って積み重ね、各階層を別々のスケジューラが担当できるようにすることである。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

リンクは**v6.12**タグに固定している — これらの名前のいくつかはその後改名されており（例: `scx_ops_bypass()` → `scx_bypass()`、`scx_ops_disable_workfn()` → `scx_disable_workfn()`。いずれもv6.16で初登場）、HEADを追うリンクでは上で説明したものとは異なる名前が表示されるためである。

- [kernel/sched/ext.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/ext.c?h=v6.12) — sched_extの実装: `struct sched_ext_ops`、ディスパッチキュー、ウォッチドッグ、バイパスモード、`SysRq-S`/`SysRq-D`のハンドラ
- [include/linux/sched/ext.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/sched/ext.h?h=v6.12) — `struct scx_dispatch_q`その他の公開sched_ext型
- [Documentation/scheduler/sched-ext.rst](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/scheduler/sched-ext.rst?h=v6.12) — カーネル自身の設計文書: DSQのセマンティクス、FIFOと優先度キューを兼ねる挙動、BPFスケジューラを中止させる3つの条件
- [include/asm-generic/vmlinux.lds.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/asm-generic/vmlinux.lds.h?h=v6.12) — `SCHED_DATA`、リンカによって強制されるスケジューラクラスの順序（`stop → dl → rt → fair → ext → idle`）

### LWNの記事 {#lwn-articles}

- [The extensible scheduler class](https://lwn.net/Articles/922405/)（2023年2月） — Jonathan Corbetによる2番目の投稿版のレビュー。最初の版に対するZijlstraの「I hate all of this」という反応も含む
- [Extensible scheduler class to be merged for 6.11](https://lwn.net/Articles/978007/)（2024年6月） — Torvalds自身の言葉による覆しの発表
- [The rest of the 6.11 merge window](https://lwn.net/Articles/982605/)（2024年8月） — 実際の結末: 「激しい議論」の後、「sched_extは6.11ではマージされなかった」
- [sched: Implement BPF extensible scheduler class](https://lwn.net/Articles/978911/) — Tejun Heoによるv7（最終版）パッチシリーズのカバーレター。6.11でのマージの試みに先立ち、2024年6月18日に投稿された
- [The 2023 Kernel Maintainers Summit](https://lwn.net/Articles/951847/)（2023年11月） — サミットの実際の議題についてLWNが公開した記事。sched_extは載っていない
- [Sched_ext at LPC 2024](https://lwn.net/Articles/991205/) — 実際の導入数（scx_layered、scx_lavd/Steam Deck）と組み合わせ可能性のロードマップ

### メーリングリスト {#mailing-list}

- [Peter Zijlstraの「I hate all of this」返信](https://lwn.net/ml/linux-kernel/Y5b2btWFJeEfTyJg@hirez.programming.kicks-ass.net/) — LWN自身のメーリングリストミラー経由。パッチセットの最初の投稿版に対する彼の反応

### 関連ページ {#related-pages}

- [スケジューラクラス](scheduler-classes.md) — sched_extが他の5つのクラスとどう並ぶか
- [EEVDF](eevdf.md) — sched_extが無効化されたとき、タスクのスケジューラクラスがどこに解決されるか
- [スケジューラの進化](scheduler-evolution.md) — このクラスと並ぶO(1) → CFS → EEVDFの歴史
