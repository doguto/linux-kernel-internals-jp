# 非特権BPFのデフォルト無効化 {#unprivileged-bpf-off-by-default}

> CVEではない。長年のCVEに対するハードニング上の回答であり、「非特権でも安全なBPF」がサブシステムとして追求をやめた目標であることを認めたものである

投入日
:   2021年5月11日

作者
:   Daniel Borkmann

マージ先
:   [`08389d888287`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=08389d888287c3823f80b0216766b71e17f0aba5)、メインライン v5.13-rc4

仕組み
:   `CONFIG_BPF_UNPRIV_DEFAULT_OFF`

背景
:   [LWN 796328](https://lwn.net/Articles/796328/)、"Reconsidering unprivileged BPF"（Jonathan Corbet、2019年8月）

*[事件簿: BPF verifierのバグとCVE](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

BPFの当初の野心には、具体的で、全体を支える重要な主張が含まれていた。それは、verifierによって*非特権*ユーザーにカーネルへプログラムをロードさせても安全にできる、というものである。[このシリーズ](../war-stories.md)の他のすべての事件は、この主張を守るために作られた仕組みのバグである。

その実行時スイッチは`/proc/sys/kernel/unprivileged_bpf_disabled`であり、一方向のラッチだった。sysctlテーブルのエントリにはコメントでそう書かれている:

```c
/* kernel/sysctl.c, before the change */
	/* only handle a transition from default "0" to "1" */
	.proc_handler	= proc_dointvec_minmax,
	.extra1		= SYSCTL_ONE,
	.extra2		= SYSCTL_ONE,
```

`extra1 == extra2 == 1`は、ハンドラが受け付ける値が1だけであることを意味する。非特権BPFを無効にすることはできたが、再起動せずに再び有効に戻すことは決してできなかった。そしてコンパイル時のデフォルトは0、つまり有効だった。

部分的な回答はその1年前にすでに投入されていた。[`2c78ee898d8f`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2c78ee898d8f10ae6fb2fa23a3fbaec96b1b7366)（"bpf: Implement CAP_BPF"、2020年5月マージ）は、verifierの単一の`allow_ptr_leaks`フラグを`allow_ptr_leaks`、`bypass_spec_v1`、`bypass_spec_v4`、`bpf_capable`の4つに分割した。これにより、プロセスに対して、投機的実行の緩和策を回避する能力まで与えることなく、BPFプログラムをロードする能力だけを付与できるようになった。これは管理者に「root」と「誰でも」の中間の設定を与えた。しかし、誰も何も設定していないマシンで起きることは変わらなかった。デフォルト無効化のノブとは違い、`CAP_BPF`は公開の場で十分に議論された。2019年8月の[Starovoitovの最初の投稿](https://lore.kernel.org/all/20190827205213.456318-1-ast@kernel.org/)には数十の返信が付き、設計はその後の投稿を通じて作り直され、約8か月半後にマージされた。

## きっかけ {#the-trigger}

この変更を生んだ議論は変更そのものより古く、LWNはパッチのほぼ2年前、2019年8月に[Reconsidering unprivileged BPF](https://lwn.net/Articles/796328/)でそれを記録している。Corbetの捉え方は次のとおりである:

> A recent discussion has made it clear, though, that the goal of opening up BPF to unprivileged users has been abandoned as unachievable, and that further work in that direction will not be accepted by the BPF maintainer.

（訳: しかし最近の議論で、BPFを非特権ユーザーに開放するという目標は達成不可能として放棄されており、その方向へのさらなる作業はBPFメンテナに受け入れられないことが明らかになった。）

Andy Lutomirskiの反論は記録しておく価値がある。この意見の相違は実際には中身の議論で決着したことはなく、ノブを出荷することで決着したからである:

> I hope not. There are a couple setsockopt uses right now, and and seccomp will surely want it someday. And the bpf-inside-container use case really is unprivileged bpf -- containers are, in many (most?) cases, explicitly not trusted by the host.

（訳: そうでないことを願う。現在すでにsetsockoptでの用途がいくつかあり、seccompもいつか必ずそれを必要とするだろう。そしてコンテナ内BPFのユースケースは実際に非特権BPFである。コンテナは多くの（大半の？）場合、明示的にホストから信頼されていない。）

BPFメンテナのAlexei Starovoitovは、Corbetの要約によれば「Linuxはシングルユーザーシステムになった」、つまりコード実行がひとつでもあればそれがrootへの経路になる、と返答した。彼はさらに続けた:

> When we say 'unprivileged bpf' we really mean arbitrary malicious bpf program. It's been a constant source of pain. The constant blinding, randomization, verifier speculative analysis, all spectre v1, v2, v4 mitigations are simply not worth it. It's a lot of complex kernel code without users. There is not a single use case to allow arbitrary malicious bpf program to be loaded and executed.

（訳: 「非特権BPF」と言うとき、実際には任意の悪意あるBPFプログラムを意味している。それは絶え間ない苦痛の種だった。定数ブラインディング、ランダム化、verifierの投機的解析、spectre v1、v2、v4のすべての緩和策は、単純に割に合わない。ユーザーのいない大量の複雑なカーネルコードだ。任意の悪意あるBPFプログラムのロードと実行を許すべきユースケースはひとつもない。）

Lutomirskiの再反論は「非特権eBPFの主要なユーザーがいないのは、カーネルのサポートがないからだ」というものだった。

Corbetは2019年の記事をどちらの側にも立たずに締めくくっているが、何が懸かっているかは正確に言い当てている:

> At its core, it's a fundamental difference of opinion over whether a Linux system can ever be truly hardened against an unprivileged user. If the answer is "no", then there is little point in maintaining a lot of complex code in the BPF subsystem to try to effect that hardening. Accepting that answer, though, is tantamount to saying that the Linux privilege model just doesn't work in the end: the combination of software bugs and hardware vulnerabilities will always undermine it, so we might as well just give up. That would be a discouraging conclusion to say the least.

（訳: その核心にあるのは、Linuxシステムを非特権ユーザーに対して本当にハードニングできるのかどうかについての根本的な意見の相違である。答えが「いいえ」なら、そのハードニングを実現しようとしてBPFサブシステムに大量の複雑なコードを保守する意味はほとんどない。しかしその答えを受け入れることは、Linuxの特権モデルは結局のところ機能しない、と言うに等しい。ソフトウェアのバグとハードウェアの脆弱性の組み合わせが常にそれを損なうのだから、諦めたほうがよい、と。控えめに言っても気の滅入る結論である。）

そして2021年が来た。5週間のうちに、サブシステムは[CVE-2021-31440](alu32-unsigned-bounds.md)（4月23日）、[CVE-2021-3490](alu32-bitwise-bounds.md)（5月10日、公表は5月11日）、[CVE-2021-33624](spectre-verifier.md)（5月28日）の修正を作成した。いずれもデフォルト設定のカーネル上で非特権プロセスから到達可能なものだった。

## 観測された挙動 {#observed-behavior}

Borkmannがノブを投稿したのは**2021年5月11日**で、CVE-2021-3490がoss-securityで公表されたのと同じ日である。

それは`bpf@vger.kernel.org`への2パッチのシリーズとして送られた。パッチ1/2（"bpf, kconfig: Add consolidated menu entry for bpf with core options"）は`kernel/bpf/Kconfig`を作成し、BPFのオプションを`init/Kconfig`と`net/Kconfig`から単一の`General setup → BPF subsystem`メニューにまとめた。理由は「BPFのコア関連オプションは、主に歴史的な理由から、さまざまなKconfigの場所に散らばっている」からである。パッチ2/2はその新しいメニューにノブを追加した。

その[スレッド](https://lore.kernel.org/all/f23f58765a4d59244ebd8037da7b6a6b2fb58446.1620765074.git.daniel@iogearbox.net/)には公開の返信がひとつも付かなかった。この話題で2年間議論が続いた後、デフォルトを変えるパッチは返信なしで取り込まれた。

## なぜ起きたか {#why-it-happened}

2019年の膠着状態がすべての説明であり、それは誤りではなく設計上の意見の相違である。

BPFの非特権安全性という目標は、verifierが敵対的なプログラム作者に対する健全なセキュリティ境界であることを要求した。投機的実行攻撃と、算術追跡のバグが絶え間なく見つかることによって、その境界を維持するコストは高くなり、Starovoitovの判断では維持する価値がなくなった。一方で、BPFの実際のユーザー、つまりトレーシングツール、ネットワークのデータプレーン、systemd、cgroupフックは、いずれにせよすべて特権で動いている。

つまり保証のコストはサブシステムに集中し、その恩恵は、まだ存在しないか、より狭いケイパビリティで対応できるユースケースに集中していた。2020年の`CAP_BPF`がその狭いケイパビリティを作った。このパッチは、何も設定しないときにトレードオフのどちら側に落ちるかを変えたのである。

この変更が具体的に何で*ない*かに注意してほしい。非特権BPFを削除するものではなく、既存の無効化スイッチを永続的にするものでもない。デフォルトを変え、スイッチを双方向にするものである。

`CONFIG_BPF_UNPRIV_DEFAULT_OFF`オプション自体は、当初はビルドする側がオプトインしなければならないものだった。このパッチがマージされた時点では`kernel/bpf/Kconfig`に`default`が付いておらず、configで明示的に有効にしない限りオフのままだった。それは5か月後に変わった。[`8a03e56b253e`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8a03e56b253e9691c90bc52ca199323d71b96204)（"bpf: Disallow unprivileged bpf by default"、Pawan Gupta、2021年10月29日、メインライン v5.16）がこのオプションに`default y`を追加した。明示的に「多くのディストリビューションがすでに適用しているものと同期するため」である。v5.16以降、非特権BPFが無効（値2）であることは、上書きしないすべてのカーネルにとってアップストリームのデフォルト*そのもの*であり、単にディストリビュータがその上に重ねた選択ではない。

## 解決 {#resolution}

[`08389d888287`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=08389d888287c3823f80b0216766b71e17f0aba5)（"bpf: Add kconfig knob for disabling unpriv bpf by default"、v5.13-rc4）は3つのことを行う。

kconfigオプションを追加する:

```
config BPF_UNPRIV_DEFAULT_OFF
	bool "Disable unprivileged BPF by default"
	depends on BPF_SYSCALL
	help
	  Disables unprivileged BPF by default by setting the corresponding
	  /proc/sys/kernel/unprivileged_bpf_disabled knob to 2. An admin can
	  still reenable it by setting it to 0 later on, or permanently
	  disable it by setting it to 1 (from which no other transition to
	  0 is possible anymore).
```

それをsysctlの初期値に結び付ける:

```c
int sysctl_unprivileged_bpf_disabled __read_mostly =
	IS_BUILTIN(CONFIG_BPF_UNPRIV_DEFAULT_OFF) ? 2 : 0;
```

そして3つ目の状態を導入する。sysctlはもはやラッチ付きのブール値ではなく、3値の設定である。同じパッチで`Documentation/admin-guide/sysctl/kernel.rst`に文書化されている:

| 値 | 意味 |
| --- | --- |
| 0 | `bpf()`の非特権呼び出しは有効 |
| 1 | `bpf()`の非特権呼び出しは無効で、**元に戻せない** |
| 2 | `bpf()`の非特権呼び出しは無効 |

1と2の区別がこの設計の微妙な点のすべてであり、汎用の`proc_dointvec_minmax`ではなく専用のハンドラによって強制される:

```c
/* kernel/sysctl.c, added by 08389d888287 */
static int bpf_unpriv_handler(struct ctl_table *table, int write,
			      void *buffer, size_t *lenp, loff_t *ppos)
{
	int ret, unpriv_enable = *(int *)table->data;
	bool locked_state = unpriv_enable == 1;
	struct ctl_table tmp = *table;

	if (write && !capable(CAP_SYS_ADMIN))
		return -EPERM;

	tmp.data = &unpriv_enable;
	ret = proc_dointvec_minmax(&tmp, write, buffer, lenp, ppos);
	if (write && !ret) {
		if (locked_state && unpriv_enable != 1)
			return -EPERM;
		*(int *)table->data = unpriv_enable;
	}
	return ret;
}
```

`locked_state`は、すでに1を書き込んでいた人のために、従来の一方向の挙動を保つ。いったん永続的に無効化されれば、永続的に無効化されたままである。しかし2で起動したカーネルは、何かのために非特権BPFを必要とする管理者が0に変更でき、また1に変更して永久にロックダウンすることもできる。コミットの要約は次のとおり:「This still allows a transition of 2 -> {0,1} through an admin. Similarly, this also still keeps 1 -> {1} behavior intact, so that once set to permanently disabled, it cannot be undone aside from a reboot.」（訳: これにより、管理者による2 -> {0,1}の遷移は引き続き可能である。同様に、1 -> {1}の挙動もそのまま維持されるため、いったん永続的な無効化に設定されると、再起動以外では元に戻せない。）

`extra1`/`extra2`の範囲は`SYSCTL_ONE`/`SYSCTL_ONE`から`SYSCTL_ZERO`/`two`に変わる。これこそが、0と2の書き込みをそもそも可能にしている部分である。

コミットはまた、サブシステムがすでに作っていた中間の道も指し示している:「Either way, as an additional alternative, applications can make use of CAP_BPF that we added a while ago.」（訳: いずれにせよ、追加の代替手段として、アプリケーションは少し前に追加したCAP_BPFを利用できる。）

## 教訓 {#what-it-taught-us}

**誰も維持できないセキュリティ保証は、正直なデフォルトよりも悪い**。2019年の議論が読んでいて居心地が悪いのは、まさにStarovoitovが非特権BPFは難しいと主張しているのではなく、やる価値がないと主張しており、それを安全にするために書かれたコードは「ユーザーのいない大量の複雑なカーネルコード」だと主張しているからである。賛成するかどうかにかかわらず、その結末、つまりデフォルトで無効にし、求める人にはケイパビリティを残すという結末は、メンテナが内心では信じていない境界を出荷するよりも正直である。

**双方向のスイッチは使われ、一方向のスイッチは避けられる**。元のラッチは善意によるものだった。いったん無効にすれば、攻撃者はそれを再び有効にできない。しかし実際には、後でBPFを必要とするかもしれない管理者は、無効化が再起動なしでは元に戻せないため、有効のままにしておかざるをえなかった。値2、つまり無効だが回復可能な状態を追加するコストは、小さな専用のsysctlハンドラひとつだけであり、それによって安全なデフォルトが採用可能になった。元に戻せない状態は、それを望む人のために引き続き利用できた。

**ケイパビリティは正しい粒度だが、それはデフォルトが正しい場合に限る**。`CAP_BPF`は2020年5月にマージされ、BPFのロードを`CAP_SYS_ADMIN`から、またSpectre緩和策の回避から分離した。しかし設定されていないマシンを1台も助けなかった。デフォルトはほぼ全員にとってのセキュリティ態勢であり、ケイパビリティはドキュメントを読む人にとってのセキュリティ態勢である。

!!! warning "注意すべきパターン"
    自分のカーネルが実際にどう動くかを確認すること。v5.16以降は無効状態（2）がアップストリームのデフォルトだが、`CONFIG_BPF_UNPRIV_DEFAULT_OFF`なしでビルドされたカーネルや、古いリリースのカーネルは、依然として非特権BPFが有効な状態で出荷される:

    ```bash
    # 0 = 非特権bpf()を許可、1 = 無効で元に戻せない、2 = 無効だが管理者が再有効化可能
    sysctl kernel.unprivileged_bpf_disabled
    ```

    より一般的には、サブシステムが挙動を直接変えるのではなくハードニングの*オプション*を追加した場合、オプション自体がデフォルトで有効になるまで（もしなるとすれば）、セキュリティ作業がビルドを設定する人に委ねられたままの期間が生じる。今回の場合は5か月だった。自分のカーネルがそれ以前のものである、あるいはそれなしでビルドされたこのようなオプションを監査し、ケイパビリティを分割するパッチ（`CAP_BPF`、`CAP_PERFMON`、`CAP_CHECKPOINT_RESTORE`）にはすべて「誰も何も設定しなかったらどうなるか？」という問いが付随しているものとして扱うこと。

## 関連項目 {#see-also}

- [BPF Verifier](../bpf-verifier.md) — 特権ローダーと非特権ローダーでverifierが何を変えるかを含む
- [Linuxケイパビリティ](../../security/capabilities.md) — `CAP_BPF`が`bpf()`のロードを切り出す元となったモデル
- [BPF verifierにおけるSpectre](spectre-verifier.md) — 2021年のverifier CVEの3つ目。このノブの投稿から2週間半後に修正された
- [ALU32ビット演算の範囲追跡](alu32-bitwise-bounds.md) — このパッチが投稿されたのと同じ日に公表された
- [カーネルハードニング](../../security/kernel-hardening.md) — これと併せて監査する価値のある、その他のビルド時ハードニングオプション（KASLR、スタックプロテクタ、`HARDENED_USERCOPY`）
- [seccomp BPF](../../security/seccomp.md) — Lutomirskiが実在する非特権BPFのユースケースとして挙げたclassic BPFのユーザー

## 外部参考資料 {#external-references}

- [git.kernel.org: 08389d888287](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=08389d888287c3823f80b0216766b71e17f0aba5) — "bpf: Add kconfig knob for disabling unpriv bpf by default"。kconfigオプション、3値のsysctl、新しいハンドラ
- [git.kernel.org: 8a03e56b253e](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8a03e56b253e9691c90bc52ca199323d71b96204) — "bpf: Disallow unprivileged bpf by default"（Pawan Gupta、2021年10月、v5.16）。`CONFIG_BPF_UNPRIV_DEFAULT_OFF`に`default y`を付けたコミット
- [lore.kernel.org: 2パッチのシリーズ](https://lore.kernel.org/all/f23f58765a4d59244ebd8037da7b6a6b2fb58446.1620765074.git.daniel@iogearbox.net/) — 2021年5月11日に`bpf@vger.kernel.org`へ投稿。公開の返信なし
- [git.kernel.org: 2c78ee898d8f](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2c78ee898d8f10ae6fb2fa23a3fbaec96b1b7366) — "bpf: Implement CAP_BPF"（2020年5月）。`allow_ptr_leaks`を4つの別々のverifier権限に分割した
- [LWN: Reconsidering unprivileged BPF](https://lwn.net/Articles/796328/) — Jonathan Corbet、2019年8月16日。この変更が最終的に決着させたStarovoitovとLutomirskiのやり取り
- [kernel.org: `unprivileged_bpf_disabled`](https://docs.kernel.org/admin-guide/sysctl/kernel.html) — このパッチが書き換えたsysctlのドキュメント
