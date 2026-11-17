# 古いptracerのクレデンシャル {#stale-ptracer-credentials}

> CVE-2019-13272 — `PTRACE_TRACEME`は、実際に関係を要求した子ではなく、トレーサーとして引き込んだ*親*のクレデンシャルを記録していた。そのため非特権の子が、特権を持つコードが一度も要求していない、rootとしてマークされたptrace関係を手に入れ、その親が特権を落として攻撃者の制御下に入った後もそれを保持できた

公開日
:   2019年7月17日（修正の作成は2019年7月4日）

報告者
:   Jann Horn、Google Project Zero（issue 1903）

CVSS
:   7.8 HIGH（`CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H` — NVDの主評価とCISA-ADPの副次評価が一致。CVSS 2.0: 7.2 HIGH）

バグの存在期間
:   4.10以降（コミット`64b875f7ac8a`、2016年11月）

修正
:   アップストリーム5.2。stable 5.1.17、4.19.58、4.14.133、4.9.185、4.4.185（いずれも2019年7月10日告知）、3.16.71

エクスプロイトツール
:   あり — Exploit-DBに4件（PoC 3件、Metasploitモジュール1件）

実際の悪用
:   あり — 2021年12月10日にCISAのKEVカタログに追加（[NVD](https://nvd.nist.gov/vuln/detail/CVE-2019-13272)による）

*[事件簿: LinuxセキュリティのバグとCVE](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

あるプロセスが別のプロセスをトレースするとき、カーネルは*関係が形成された時点でトレーサーがどれだけの特権を持っていたか*を覚えておく必要がある。後でトレーサーの現在のクレデンシャルを再チェックするだけでは済まない。トレーサーはアタッチ後に正当に特権を落とすことがあるからである。かといってこの問いを無視することもできない。その答えが、トレースされているプロセスがsetuidの`execve()`を完全な特権で完了してよいかを決めるからである。

Linuxはその答えを*トレース対象（tracee）*の単一のフィールドに格納する。これは2016年に[`64b875f7ac8a`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=64b875f7ac8a5d60a4e191479299e931ee949b67)（"ptrace: Capture the ptracer's creds not PT_PTRACE_CAP"）によって`struct task_struct`に追加された:

```c
/* include/linux/sched.h */
const struct cred __rcu *ptracer_cred; /* アタッチ時のトレーサーのクレデンシャル */
```

このコミットは、古い単一ビット`PT_PTRACE_CAP`を置き換えた。そのビットの問題は、トレーサーが特権を持っていた*かどうか*だけを記録し、*どのユーザー名前空間で*持っていたかを記録しないことだった。Eric Biedermanのコミットメッセージは「this has already allowed one mistake through insufficient granularity」（訳: これは粒度の不足によってすでに1つの誤りを許している）と述べ、さらにこのビットが最初に導入されたときに`PTRACE_TRACEME`の経路が「overlooked」（見落とされていた）と記している。完全な`struct cred`ポインタを格納することで両方が修正され、新しいヘルパーが必要に応じて問いに答えた:

```c
/* kernel/capability.c */
bool ptracer_capable(struct task_struct *tsk, struct user_namespace *ns)
{
	int ret = 0;  /* トレーサーがいなければ制限は追加されない */
	const struct cred *cred;
	rcu_read_lock();
	cred = rcu_dereference(tsk->ptracer_cred);
	if (cred)
		ret = security_capable_noaudit(cred, ns, CAP_SYS_PTRACE);
	rcu_read_unlock();
	return (ret == 0);
}
```

5.2の開発サイクルの時点で、そのフィールドを設定するコードは次のようになっていた（`kernel/ptrace.c`）:

```c
void __ptrace_link(struct task_struct *child, struct task_struct *new_parent,
		   const struct cred *ptracer_cred)
{
	BUG_ON(!list_empty(&child->ptrace_entry));
	list_add(&child->ptrace_entry, &new_parent->ptraced);
	child->parent = new_parent;
	child->ptracer_cred = get_cred(ptracer_cred);
}

static void ptrace_link(struct task_struct *child, struct task_struct *new_parent)
{
	rcu_read_lock();
	__ptrace_link(child, new_parent, __task_cred(new_parent));
	rcu_read_unlock();
}
```

`ptrace_link()`ラッパーの呼び出し元はちょうど2つであり、バグのすべてはその2つの違いの中にある:

```c
ptrace_link(task, current);                    /* ptrace_attach()  */
ptrace_link(current, current->real_parent);    /* ptrace_traceme() */
```

`PTRACE_ATTACH`では、`new_parent`は`current`、つまり関係を要求しているプロセス*そのもの*なので、`__task_cred(new_parent)`は正しいプロセスを指す。`PTRACE_TRACEME`では、`new_parent`は`current->real_parent`であり、関係を要求しているプロセスは子である。ラッパーはどちらの場合も`new_parent`のクレデンシャルを記録していた。クレデンシャルを*要求者*からではなく*トレーサー*の引数から導出していたからである。

`struct cred`、そのコピーオンライトの規律、そして`execve()`をまたいでクレデンシャルに何が起きるかの背景については、[クレデンシャルとユーザー名前空間](../credentials.md)を参照。このバグが損なう特権放棄のパターン、つまりrootを取得し、特権が必要な作業を行い、`setuid()`で権限を戻し、それから信頼度の低い何かを`execve()`する、というパターンは[Linuxケイパビリティ](../capabilities.md#capability-based-privilege-dropping)で扱っている。

## きっかけ {#the-trigger}

`PTRACE_TRACEME`はptraceのリクエストの中でも特殊である。*トレース対象*が発行し、トレーサーを指定しない。カーネルがトレーサー、つまり実の親を代わりに選び、唯一の権限チェックはLSMフック`security_ptrace_traceme(current->parent)`である。これは*親*が子をトレースしてよいかを問うものであり、子が親を引き込むことを許されるべきかは決して問わない:

```c
/* kernel/ptrace.c, ptrace_traceme() */
write_lock_irq(&tasklist_lock);
/* すでにトレースされているか？ */
if (!current->ptrace) {
	ret = security_ptrace_traceme(current->parent);
	if (!ret && !(current->real_parent->flags & PF_EXITING)) {
		current->ptrace = PT_PTRACED;
		ptrace_link(current, current->real_parent);
	}
}
write_unlock_irq(&tasklist_lock);
```

したがって非特権プロセスは、`ptracer_cred`を*その瞬間に親がたまたま保持しているクレデンシャルのスナップショット*に一方的に設定させることができる。親をトレースする権限は必要ない。親の協力も必要ない。親には通知すらされない。

悪用の窓は、特権を持つ親の生存期間のうち「rootである」と「もはやrootではない」の間の隙間である。Jann Hornの[Project Zeroへの元の報告](https://project-zero.issues.chromium.org/issues/42450993)（issue 1903）は、polkitの`pkexec`に対してそれを実証した。攻撃者プロセスAがBをforkする。Bは子Cをforkし、*その後で*`pkexec`を`execve()`するので、Cの実の親はrootとして動作していることになる。Cが`PTRACE_TRACEME`を呼ぶと、記録される`ptracer_cred`はrootを所有するBのスナップショットになる。次にCがsetuidバイナリを`execve()`する。カーネルは`ptracer_cred`を参照するが、そこにあるのは関係を実際に要求したプロセスであるCのものではなく*B*のrootクレデンシャルなので、カーネルはそのptrace関係を特権付きとして扱い、setuidの遷移を抑止しない。その結果、Cはトレースされたまま完全なrootとして動作することになる。その後でようやくBが特権を落としてdumpableになり、その時点でAはBに`PTRACE_ATTACH`し、Bを通じてroot特権のCを操ることができる。Exploit-DBにある2つの派生PoCツール（bcolesによる、Hornのオリジナルをヘルパーを自動で標的にするようにした版で、20余りのディストリビューションでテストされたもの、そして後にUjas Dhamiがbcolesの版を3つのディストリビューション向けに改変したもの）は、まさにこのpolkitヘルパーの手法を自動化した。Rapid7は2019年10月にそれを[Metasploitモジュール](https://raw.githubusercontent.com/rapid7/metasploit-framework/master/modules/exploits/linux/local/ptrace_traceme_pkexec_helper.rb)として出荷した。これら後発の3つのツールはいずれも同じローカルセッションの前提条件、つまりリモートでないログイン（`loginctl … Remote=no`）の`$XDG_SESSION_ID`をテストするが、それに基づいて動作を変えるのはMetasploitモジュールだけである。その`check`は`loginctl`が`Remote=yes`を報告すると`CheckCode::Safe`を返し、これが「cannot be executed over ssh」（ssh越しには実行できない）と文書化されている理由である。2つのC言語のPoCは警告を表示するだけである。それらだけがテストするSELinuxの`deny_ptrace`ブール値についても同様であり、`check_env()`の戻り値が`main()`で捨てられているからである。Yamaの`kernel.yama.ptrace_scope = 3`なら`PTRACE_TRACEME`を完全にブロックできる。スコープ2は親が`CAP_SYS_PTRACE`を持たない場合にだけそれを拒否するが、rootの`pkexec`の親はそれを持っている。しかし3つのツールのどれもそのsysctlを一切チェックしない。どちらの制限もLinuxのデフォルトではなく、それゆえこのバグは当時のほとんどのデスクトップおよびサーバーのディストリビューションで、そのままの状態で悪用可能だった。

## 観測された挙動 {#observed-behavior}

このバグの特権に関わる半分は、クラッシュもメモリ破壊も、ファザーから見える痕跡も生まなかった。唯一の症状は、権限の判断が誤った結果になることである。（同じ報告のうち、オブジェクトの寿命に関わる別の半分はもっと騒がしい。Hornのもう1つのPoCは`setresuid()`と`PTRACE_TRACEME`を競合させ、`put_cred_rcu()`のパニックを引き起こす。）トレースされているプロセスは、カーネルが、そのプロセスが自ら差し出したわけではないクレデンシャルを根拠に特権付きとマークした「トレーサー」によって操作されえた（レジスタの書き換え、`/proc/<pid>/mem`への書き込み、システムコールの横取り）。そして操作が行われる頃には、そのトレーサー役の背後にある生身のプロセスは`setuid()`/`setgid()`で特権を落とし、攻撃者が到達可能なコードの実行に移っていた。NVDはこれをCVSS 3.1 **7.8 HIGH**（`CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`）と評価している。ローカル、低い複雑さ、低い特権が必要、機密性・完全性・可用性への完全な影響であり、最終状態がrootとしての任意コード実行だからである。

このバグはLinux **4.10**から存在していた。2016年11月に作成された[`64b875f7ac8a`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=64b875f7ac8a5d60a4e191479299e931ee949b67)が`ptracer_cred`を導入したのがこのバージョンであり、Jann Hornの報告のほぼ3年前である。修正とCVEが公開されると、各ディストリビューションはそれぞれのスケジュールで追随した。DebianのDSA-4484は7月20日、Red HatのRHSA-2019:2405は8月7日に続き、Ubuntuの5つのカーネルアドバイザリ（USN-4093-1からUSN-4118-1まで、外部参考資料を参照）は8月13日と9月2日の2回に分けて出た。同じ週の協調リリースではなく、7週間にわたる長い裾野だった。

## なぜ起きたか {#why-it-happened}

根本原因は、**2つの呼び出し元を持つヘルパー関数の内部での主体と客体の取り違え**である。`ptrace_link()`は、`PTRACE_ATTACH`（呼び出し元とトレーサーが同じプロセス）と`PTRACE_TRACEME`（そうではない）の両方に、常に同じであると仮定した単一のコードパスで対応するように書かれていた。`__task_cred(new_parent)`は`new_parent == current`の場合には正しいクレデンシャルの源である。しかし`PTRACE_TRACEME`では、それは黙って*誤った*主体、つまり何も要求していない親を選んでしまう。そして型システムにも関数のシグネチャにも、2つの呼び出し箇所の一方でその仮定が破られていることを示すものは何もなかった。

`ptracer_cred`を導入したコミット`64b875f7ac8a`自体がセキュリティ修正だった。より粗い`PT_PTRACE_CAP`ビットを置き換えたのは、まさにそのビットがトレーサーが*どの*ユーザー名前空間で特権を持っていたかを区別しなかったからである。Eric Biedermanはその2016年のコミットのメッセージで、`PT_PTRACE_CAP`が最初に追加されたときに`PTRACE_TRACEME`の経路が一度「overlooked」（見落とされた）ことをすでに指摘していた。ビットを完全なクレデンシャルポインタで置き換えたときに、同じ呼び出し箇所が二度目の誤りを犯した。今回は飛ばされたのではなく、誤った主体と結び付けられたのである。`ptrace_link()`の単一のクレデンシャルの源は、一般的なケース（`PTRACE_ATTACH`）では正しく、ここでは黙って誤っていた。

2016年のパッチは"Fixing ptrace vs exec vs userns interactions"と題された3パッチのシリーズの一部として[公開でレビューされ](https://lore.kernel.org/lkml/87oa1eavfx.fsf_-_@xmission.com/)、そのレビューは、別の行に向けられたものではあったが、後のバグまであと1つの質問というところまで迫っていた。`fs/exec.c`の`check_unsafe_exec()`のハンク、つまり実際の取り違えがあった`ptrace_link()`の取得箇所ではなく、新しいフィールドの*利用側*をレビューしていたAndy Lutomirskiは、次のように尋ねた:「IIRC `PT_PTRACE_CAP` was added to prevent TOCTOU races. What prevents that type of race now? For that matter, what guarantees that we've already switched to new creds here and will continue to do so in the future?」（訳: 記憶が正しければ、`PT_PTRACE_CAP`はTOCTOUの競合を防ぐために追加された。今は何がその種の競合を防いでいるのか？ ついでに言えば、ここですでに新しいクレデンシャルに切り替わっていて、将来もそうであり続けることを何が保証しているのか？）Biedermanの答えは、新しいフィールドが*いったん設定されれば*安全である理由を説明していた:「Because instead of capturing a single bit we now capture tracers entire credentials in `tsk->ptracer_cred`. As such `tsk->ptracer_cred` never changes except when ptracing begins or ends, and we remain safe for TOCTOU races.」（訳: 単一のビットを取得する代わりに、今はトレーサーのクレデンシャル全体を`tsk->ptracer_cred`に取得しているからだ。そのため`tsk->ptracer_cred`はptraceの開始時と終了時を除いて決して変わらず、TOCTOUの競合に対して安全なままである。）その答えは正しく、それはクレデンシャルが*いつ*取得されるかについてのものだった。隣にある問い、つまり*どの*呼び出し元について*誰の*クレデンシャルなのか、という問いこそが、`PTRACE_TRACEME`の取り違えを3年早く捕捉できたはずのものだった。誰もそれを問わなかった。修正が併せて取り込んでいる、二次的でより小さなバグもある。`ptrace_link()`は親のクレデンシャルへのRCU参照を取得し、そのポインタを`get_cred()`に渡していた。修正のコミットメッセージにおけるJann Horn自身の言葉によれば、「the object lifetime rules for things like struct cred do not permit unconditionally turning an RCU reference into a stable reference」（訳: struct credのようなもののオブジェクト寿命の規則は、RCU参照を無条件に安定した参照に変えることを許さない）。これは、誰のクレデンシャルを読んでいたかとは無関係に、古いコードパスが不健全だった2つ目の理由である。

## 解決 {#resolution}

2019年7月4日にJann Hornが作成した[`6994eefb0053`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6994eefb0053799d2e07cd140df6c2ea106c41ee)（"ptrace: Fix ->ptracer_cred handling for PTRACE_TRACEME"）は、`ptrace_link()`を1行に縮める:

```c
static void ptrace_link(struct task_struct *child, struct task_struct *new_parent)
{
	__ptrace_link(child, new_parent, current_cred());
}
```

`current_cred()`、つまりptrace関係を要求するシステムコールを実際に実行しているプロセスのクレデンシャルが、無条件に`__task_cred(new_parent)`を置き換える。`PTRACE_ATTACH`では、`current`と`new_parent`はすでに同じプロセスだったので、これは何も変えない。`PTRACE_TRACEME`では、これが本当の修正である。記録されるクレデンシャルは、トレースしてもらおうとしている親ではなく、要求している子のものになるので、後で親の特権が変化しても、子自身が非特権のときに確立したptrace関係が遡って格上げされることはもうない。また、`current_cred()`はRCU参照外しをまったく必要としないため、RCU/`get_cred()`の寿命の問題も回避される。

このコミットは`Fixes: 64b875f7ac8a`と`Acked-by: Oleg Nesterov <oleg@redhat.com>`を持つ（Nesterovはptraceの長年のメンテナの一人である）が、**`Link:`トレーラーはなく**、このパッチのマージ前の公開のLKML/lore.kernel.orgでのレビュースレッドは存在しない。公開リストへの初登場は7月8日で、マージの4日後に、Greg Kroah-Hartmanの5つのstableレビューシリーズのそれぞれに含まれる1パッチとしてだった。それは非公開で扱われ、直接マージされた。Linusは7月4日にメインラインにコミットし、7月7日のv5.2リリースで出荷され、Jann HornのProject Zeroの報告は7月17日に公開された。修正そのものは、CVEとディストリビューションのアドバイザリが追いつく2週間近く前に、公開のgit履歴に存在していたのである。

修正は5.2向けにメインラインに入り、1週間以内に5.1.17、4.19.58、4.14.133、4.9.185、4.4.185に、後に3.16.71にもバックポートされた。stableツリーへのこの広い展開は、バグがいかに長く存在していたかを反映している（メインラインには4.10から存在していたが、`64b875f7ac8a`には`Cc: stable`が付いていたため、脆弱なコードは4.4、4.9、3.16のstableツリーにも届いていた）。

```bash
# パッチ未適用時の暫定緩和策: PTRACE_TRACEMEを無条件にブロックするのはスコープ3のみ
sysctl -w kernel.yama.ptrace_scope=3    # 0/1ではこのバグは止められない（1 = Ubuntu/Debianのデフォルト、
                                        # PTRACE_ATTACHのみを制限）。2は親がCAP_SYS_PTRACEを持たない
                                        # 場合にのみTRACEMEを拒否する。3の設定は再起動まで元に戻せない。
                                        # それより低い値はどちらの方向にも変更可能。
```

## 教訓 {#what-it-taught-us}

**2つの呼び出し箇所に仕えるヘルパー関数は、一般的なケースを満たすだけでなく、実際の前提条件を明示する必要がある**。`ptrace_link()`のバグは、`PTRACE_ATTACH`の経路だけを追っている限り見えない。「*もう一方の*呼び出し元にとって`new_parent`は何を意味するのか？」と問うて初めて現れる。関数自身のシグネチャはその問いを促さなかった。

**一度誤った呼び出し箇所には、修正だけでなく監査がふさわしい**。`ptracer_cred`を導入したコミットは、15年前に`PT_PTRACE_CAP`が最初に追加されたとき、`PTRACE_TRACEME`がすでに一度見落とされていたことを明記していた。2016年の書き換えはその隙間を閉じ、反対側から再び開いた。ついに`PTRACE_TRACEME`のクレデンシャルを記録するようになったが、それは誤ったプロセスのものだった。脆いことが分かっている呼び出し箇所を修理することは、それを監査することと同じではない。

**クレデンシャルは要求者から取得しなければならず、関連するプロセスから推測してはならない**。`current`自身のシステムコールが引き起こすアクセス制御の判断において、`current_cred()`は常に正しい主体である。「関係がありそうな誰か」のために別のtask_structへ手を伸ばすことは、このバグが悪用した、まさにその種の混乱した代理人（confused deputy）の隙間を再び持ち込む。

**特権放棄のパターン（`setuid()`の後に`execve()`）は、放棄の前に形成されたすべての関係と同程度にしか安全ではない**。そのような関係はptraceだけではない。開いているファイルディスクリプタ、保留中のシグナル、名前空間の所属もすべて同じ境界をまたぐ。しかしこのバグは、ある特定の関係が、それを終わらせるはずだった放棄をまたいで、古い特権を黙って持ち越した具体的な事例である。

!!! warning "注意すべきパターン"
    「この関係の*相手側*が作成時にどれだけの特権を持っていたか」をスナップショットするカーネルオブジェクトはすべて、都合で選んだ関連プロセスではなく、要求を行っているプロセスのクレデンシャルを記録する必要がある。また、それを設定するコードの、最も一般的なものだけでなく、異なる呼び出し規約のそれぞれについてテストが必要である。

## 関連項目 {#see-also}

- [クレデンシャルとユーザー名前空間](../credentials.md) — `struct cred`、そのコピーオンライトの規律、そして`execve()`をまたいでクレデンシャルに何が起きるか
- [Linuxケイパビリティ](../capabilities.md) — このバグが損なった特権放棄のパターン（`setuid()`の後に`execve()`）
- [入れ子のユーザー名前空間におけるUID/GIDマッピング](nested-userns-uid-mapping.md) — 同時期にJann Hornが報告した、別のクレデンシャル／アイデンティティのバグ

## 外部参考資料 {#external-references}

- [NVD: CVE-2019-13272](https://nvd.nist.gov/vuln/detail/CVE-2019-13272) — CVSS 3.1 7.8 HIGH、2019年7月17日公開
- [git.kernel.org: 6994eefb0053](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6994eefb0053799d2e07cd140df6c2ea106c41ee) — "ptrace: Fix ->ptracer_cred handling for PTRACE_TRACEME"
- [git.kernel.org: 64b875f7ac8a](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=64b875f7ac8a5d60a4e191479299e931ee949b67) — "ptrace: Capture the ptracer's creds not PT_PTRACE_CAP"（2016年）。このバグの修正が訂正した`PTRACE_TRACEME`の処理を持っていたコミット
- [lore.kernel.org: Fixing ptrace vs exec vs userns interactions](https://lore.kernel.org/lkml/87oa1eavfx.fsf_-_@xmission.com/) — 2016年のレビュースレッド。Andy LutomirskiのTOCTOUに関する質問とEric Biedermanの回答を含む
- [Google Project Zero: Issue 1903](https://project-zero.issues.chromium.org/issues/42450993) — Jann Hornによる元の報告。polkitの`pkexec`を使った悪用経路を含む（旧URLは`bugs.chromium.org/p/project-zero/issues/detail?id=1903`）
- [Exploit-DB 47133](https://www.exploit-db.com/exploits/47133)、[47163](https://www.exploit-db.com/exploits/47163)、[50541](https://www.exploit-db.com/exploits/50541) — Hornの元の報告とPoC（47133）、および2つの派生版。bcolesによる、ヘルパーを自動で標的にする版（47163、2019-07-24）と、Ujas Dhamiによるbcoles版の改変（50541、2021-11-23）
- [Rapid7 Metasploitモジュール: ptrace_traceme_pkexec_helper](https://raw.githubusercontent.com/rapid7/metasploit-framework/master/modules/exploits/linux/local/ptrace_traceme_pkexec_helper.rb) — Exploit-DB 47543、2019年10月
- [Debian DSA-4484](https://www.debian.org/security/2019/dsa-4484)、[Ubuntu USN-4093-1](https://usn.ubuntu.com/4093-1/)（およびUSN-4094-1、4095-1、4117-1、4118-1）、[Red Hat RHSA-2019:2405](https://access.redhat.com/errata/RHSA-2019:2405) — 代表的な下流ベンダーのアドバイザリ、2019年7〜9月
