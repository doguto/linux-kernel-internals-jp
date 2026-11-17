# BPF verifierにおけるSpectre {#spectre-in-the-bpf-verifier}

> CVE-2021-33624 — verifierは実際に実行されうる経路しか辿らないため、分岐予測器が辿る経路を一度も見ていなかった

公開日
:   2021年6月21日（oss-security。NVDのレコード公開は2021年6月23日）

報告者
:   Adam MorrisonとOfek Kirzner、および独立にBenedict SchlueterとPiotr Krysiuk（修正コミットの`Reported-by:`タグによる）

CVSS
:   4.7 MEDIUM（`CVSS:3.1/AV:L/AC:H/PR:L/UI:N/S:U/C:H/I:N/A:N`、NVDの主評価）

修正
:   [`9183671af6db`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9183671af6dbf60a1219371d4ed73e23f43b49db)、メインラインv5.13-rc7。stable 5.12.13および5.10.46

エクスプロイトツール
:   複数の概念実証（PoC）が`security@kernel.org`に非公開で送られた。NVDの参照リストは公開のoss-security投稿に`Exploit`タグを付け、公開PoCリポジトリへリンクしている。Exploit-DBのエントリはない

実際の悪用
:   確認された事例はない（[CISA KEVには未掲載](https://nvd.nist.gov/vuln/detail/CVE-2021-33624)）

*[事件簿: BPF verifierのバグとCVE](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

BPF verifierの仕事は、プログラムの実行を許可する前に、そのプログラムが触れてはならないメモリに触れられないことを証明することである。それは[抽象解釈](../bpf-verifier.md)によって行われる。プログラムが取りうるすべての制御フロー経路を辿り、レジスタごとに、その経路上で値が何になりうるかを追跡するのである。

Spectreはその根底にある前提を壊した。2018年1月、Alexei Starovoitovの[`b2157399cc98`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b2157399cc9898260d6031c5bfe45fe137c1fbe7)（"bpf: prevent out-of-bounds speculation"、v4.15-rc8）は、問題を2文で述べるところから始まっている:

> Under speculation, CPUs may mis-predict branches in bounds checks. Thus, memory accesses under a bounds check may be speculated even if the bounds check fails, providing a primitive for building a side channel.

（訳: 投機実行下では、CPUは境界チェックの分岐を予測ミスすることがある。そのため、境界チェックが失敗する場合でも、境界チェック配下のメモリアクセスが投機的に実行されることがあり、サイドチャネルを構築するためのプリミティブを提供してしまう。）

そのパッチの答えはマスキングだった。すべての配列マップは`roundup_pow_of_two(max_entries) - 1`の`index_mask`を保持し、Cの経路、つまり`array_map_lookup_elem()`、per-CPU版、更新の経路は、すべてのインデックスを無条件にそれとANDする。非特権プロセスが作成したマップについては、さらに2つのことが起きる。割り当てそのものが`index_mask + 1`に切り上げられ、マスクされたインデックスが常に割り当て済みメモリに収まるようになる。また、インライン化された命令列（`array_map_gen_lookup()`、およびverifierによる`bpf_tail_call`のパッチ）もANDを出力する:

```c
/* kernel/bpf/arraymap.c, after b2157399cc98 */
return array->value + array->elem_size * (index & array->index_mask);
```

`bpf_tail_call()`も同じ扱いを受け、verifierによって明示的な`if (index >= max_entries) goto out; index &= array->index_mask;`の命令列にパッチされた。これにより、各JITを個別に修正しなくても、すべてのJITがマスクを受け継いだ。コミットは何をカバーし、何をカバーしないかをはっきりと述べている: *"That fixes bpf side of 'Variant 1: bounds check bypass (CVE-2017-5753)' on all architectures with and without JIT."*（訳: これにより、JITの有無を問わずすべてのアーキテクチャで、「Variant 1: bounds check bypass (CVE-2017-5753)」のBPF側が修正される。）

つまり、それはSpectre v1の*配列インデックス*という形を修正するものである。どのコードが到達可能かについてのverifier自身のモデルについては何も述べていない。

## きっかけ {#the-trigger}

3年半後、その隙間がバグになった。

verifierは、アーキテクチャ上のマシンで到達可能な経路だけを列挙する。ある分岐が常に成立すると証明できれば、成立する側を辿り、もう一方を捨てる。投機実行下では、CPUはそれでも捨てられた側を実行しうる。

Daniel Borkmannの修正コミットは、これを具体的に示す6行のプログラムから始まる:

```
// r0 = pointer to a map array entry
// r6 = pointer to readable stack slot
// r9 = scalar controlled by attacker
1: r0 = *(u64 *)(r0) // cache miss
2: if r0 != 0x0 goto line 4
3: r6 = r9
4: if r0 != 0x1 goto line 6
5: r9 = *(u8 *)(r6)
6: // leak r9
```

3行目は`r0 == 0`の場合にのみ実行される。5行目は`r0 == 1`の場合にのみ実行される。両者は互いに排他的なので、攻撃者が制御するスカラーを`r6`に代入し、*その後で*それを参照外しするような実際の実行は存在しない。verifierは両方の経路を辿り、両方を含む経路がないことを確認して、プログラムを受け入れる。

落とし穴は1行目にある。キャッシュミスを起こすよう仕組まれたロードである。CPUはストールする代わりに両方の分岐を予測して先へ進む。そして両方についてフォールスルーを予測すれば、verifierが同時には起こりえないと証明した命令のペアを、まさに投機的に実行してしまう。

## 観測された挙動 {#observed-behavior}

互いに排他的な2つの分岐を誤訓練するのは不可能に聞こえるが、コミットメッセージはその点に正面から取り組んでいる。分岐予測は値ベースではなく履歴ベースであり、分岐アドレスのハッシュをキーとしてCPUのパターン履歴テーブル（PHT）を引く。そこで攻撃者は、ユーザー空間で*異なる、排他的でない*分岐を訓練し、それらをPHT上でBPFプログラムの分岐と衝突するアドレスに配置する:

> A non-privileged attacker could simply brute force such collisions in the PHT until observing the attack succeeding.

（訳: 非特権の攻撃者は、攻撃が成功するのを観測するまで、PHT内のそのような衝突を単純に総当たりで探すことができる。）

コミットはさらに、PHT衝突の総当たりを完全に避ける、より確実な2つ目の構成を記録している。約190命令のプログラムで、制御用マップからロードされる「訓練／攻撃フェーズ」のセレクタ、興味のある分岐を現在の実行フローから引き離すための長いダミー条件ジャンプの連鎖、そしてverifierに対しては既知のゼロにマスクされる（`r3 &= 1; r3 &= 2;`）一方で、実行時には実際のロードのレイテンシ依存を保持し続ける遅い（キャッシュミスする）ロードから成る。

影響は情報漏洩であって、破壊ではない。LWNの[Spectre revisits BPF](https://lwn.net/Articles/860597/)（Jonathan Corbet、2021年6月24日）は、アドバイザリを要約して次のように述べている:

> These attacks can read out any memory in the kernel's address space; given that all of physical memory is contained therein, there are no real limits to what can be exfiltrated.

（訳: これらの攻撃はカーネルのアドレス空間内の任意のメモリを読み出すことができる。物理メモリのすべてがそこに含まれていることを考えると、持ち出せるものに実質的な制限はない。）

Corbetはまた、BPFをSpectreの標的として独特に魅力的にしている点も指摘した。ほとんどの投機的実行攻撃では、カーネルバイナリにすでに存在する適切なガジェットを見つける必要があるが、「BPFはユーザー空間からカーネルコンテキスト内で動くコードをロードできるようにするために存在する。これにより攻撃者は自前のコード断片を作成でき、カーネルコードをくまなく探すという退屈な作業を避けられる」。

LWNによれば、問題が報告された際に複数の概念実証が`security@kernel.org`に送られ、そのうちいくつかは分岐予測器の訓練ステップをまったく必要としなかった。

## なぜ起きたか {#why-it-happened}

verifierの健全性の議論は、常にアーキテクチャ上のマシンについてのものだった。*ISAが許すすべての実行について、プログラムは安全である*。投機実行は、ISAが許さない実行を追加する。

`b2157399cc98`はすでにその点を認めていたが、それはひとつの形についてだけだった。配列インデックスのマスキングは、CPUが投機的に通り過ぎる境界チェックに対しては防御になる。しかしCPUが投機的に入り込む*経路*に対しては何もしない。マスキングのロジックが適用される前に、verifierがその経路を到達不能としてすでに捨てていたからである。

言い換えれば、2018年の修正はverifierが追跡する値を強化した。2021年のバグは、verifierがそもそもどの経路を追跡するかにあった。

同時期のパッチからは、もうひとつの目立たない要因も見て取れる。BPFにおける投機実行への防御は、個別の対策の積み重ねとして後付けされている。インデックスのマスキング（[`b2157399cc98`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b2157399cc9898260d6031c5bfe45fe137c1fbe7)）、ポインタ算術のサニタイズ、そしてこの修正の1か月前の[`801c6058d14a`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=801c6058d14a82179a7ee17a4b532cac6fad067f)（"bpf: Fix leakage of uninitialized bpf stack under speculation"、これもPiotr Krysiukによる報告）である。最後のものは、投機的なポインタ算術が中間オフセットを辿り、プログラムが書き込む前の512バイトのBPFスタックを読めてしまう隙間を塞いだ。それぞれは正しいが、どれも一般的な論証ではないため、攻撃の新しい形が現れるたびに新しい対策が必要になる。

## 解決 {#resolution}

修正は、成立しないことが証明された分岐に対するverifierの扱いを反転させる。それらを捨てる代わりに、明示的に投機的とマークしたうえで検証スタックに積む:

```c
/* kernel/bpf/verifier.c, added by 9183671af6db */
static struct bpf_verifier_state *
sanitize_speculative_path(struct bpf_verifier_env *env,
			  const struct bpf_insn *insn,
			  u32 next_idx, u32 curr_idx)
{
	struct bpf_verifier_state *branch;
	struct bpf_reg_state *regs;

	branch = push_stack(env, next_idx, curr_idx, true);
	if (branch && insn) {
		regs = branch->frame[branch->curframe]->regs;
		if (BPF_SRC(insn->code) == BPF_K) {
			mark_reg_unknown(env, regs, insn->dst_reg);
		} else if (BPF_SRC(insn->code) == BPF_X) {
			mark_reg_unknown(env, regs, insn->dst_reg);
			mark_reg_unknown(env, regs, insn->src_reg);
		}
	}
	return branch;
}
```

`check_cond_jmp_op()`では、`pred == 1`と`pred == 0`の両方のケースでこれが呼ばれるようになった。ただし`!env->bypass_spec_v1`を条件としているため、すでに投機的解析をオプトアウトしている特権ローダーは何のコストも払わない。条件に関わるレジスタは投機的な分岐上で不明とマークされる。予測ミスの後では、その内容についてのいかなる仮定も成り立たないからである。

コミットは、積んだ分岐の状態をラベルなしで積むのではなく*投機的*とマークする理由を2つ挙げている。デッドコード除去が後でそれらの命令を`jmp-1`でサニタイズできるようにするため、そして非投機的ドメインで辿られる経路が、先に行われた投機的ドメインの探索によって枝刈りされないようにするためである。

**コミットは、検討して却下した代替案も記録しており**、これはセキュリティ修正としては異例なほど率直である。もうひとつの選択肢は、`BPF_JMP_TAKEN`状態と方向のエンコーディング（常にgoto、常にフォールスルー、不明）を`aux->alu_state`に記録し、方向が混在するプログラムを拒否するというものだった。これは`if (...) { x = 0; } else { x = 1; }`の後に`if (x == 1)`が続くような普通の構文を拒否してしまっただろう。Borkmannは2つの欠点を挙げている。ポインタ算術を行わない正当なプログラムが拒否される可能性があること、そして非特権プログラムに対して経路の枝刈りを無効にしなければならないことである。投機的な分岐を積む方式なら両方を避けられる。

LWNの記事におけるCorbetの変更ログへの評価は次のとおり: *「このパッチの変更ログは、脆弱性とその修正を文書化する方法の傑出した例である。全文を読む価値がある」*。

コストは公然と受け入れられている。LWN:

> This change has the potential to block the loading of correct programs that could be run before, though it is hard to imagine real-world, non-malicious code that would include this kind of pattern. It will, of course, slow the verification process to force it to examine paths that cannot occur in normal program execution, but that's the speculative world we live in.

（訳: この変更は、以前は実行できていた正しいプログラムのロードを妨げる可能性があるが、この種のパターンを含む現実の悪意のないコードは想像しにくい。もちろん、通常のプログラム実行では起こりえない経路を調べさせるので検証処理は遅くなるが、それが我々の生きる投機的な世界である。）

修正はv5.13-rc7でメインラインに入り、LWNによれば5.12.13と5.10.46のstable更新にも入ったが、その記事の執筆時点では、それより前のstableシリーズには入っていなかった。古いLTSブランチには2か月後に入り、それは単一のパッチではなかった。Ovidiu Panaitが修正一式を手作業でバックポートし、[5.4向けに6パッチ](https://lore.kernel.org/all/20210805155343.3618696-1-ovidiu.panait@windriver.com/)、4.19向けに4パッチを作成した。それぞれをbpfリストに投稿して健全性チェックを求め（"the fixes were manually adjusted to apply to 5.4, so copying bpf@ to see if there are any concerns"、訳: 修正は5.4に適用できるよう手作業で調整したので、懸念がないか確認するためbpf@にもCCする）、その後Greg Kroah-Hartmanがキューに入れた。

## 教訓 {#what-it-taught-us}

**健全性の証明は、それが前提とするマシンモデルにおいてのみ健全である**。verifierは、ISAに従ってプログラム順に命令を実行するマシンについての性質を証明する。実際のCPUはそのマシンではなく、verifierが「これは起こりえない」と言うあらゆる場所は、2つのマシンの差が攻撃面になる場所である。経路の到達可能性はそのひとつだと判明した。最後のひとつである可能性は低い。

**個別の緩和策を組み合わせても保証にはならない**。インデックスのマスキング、ポインタ算術のサニタイズ、未初期化スタックのサニタイズは、それぞれ特定の投機的な形に対する正しい修正である。どれも次のものを含意しない。BPFサブシステムは2018年から2021年の間に3つの別々のSpectre対策を積み上げ、それでも4つ目が必要になった。

**攻撃者が作成したコードをカーネルにロードできることは、投機的攻撃の経済性を変える**。通常のSpectreのガジェット探しは探索問題である。非特権BPFがあれば、攻撃者がガジェットを書く。この非対称性こそが、[非特権BPFのデフォルト無効化](unprivileged-bpf-off.md)のノブが同じリリースサイクルで投入された理由であり、また`AC:H`と`C:H/I:N/A:N`によって上限が抑えられたCVSSの4.7 MEDIUMが、すでにローカルでのコード実行権を持つ者にとってこのバグが持つ価値を過小評価している理由でもある。

!!! warning "注意すべきパターン"
    到達不能と証明できる分岐を*枝刈り*する静的解析はすべて、投機的なマシンではなくアーキテクチャ上のマシンについての主張をしている。セキュリティに関わるコード（verifier、JIT、サンドボックスのポリシーコンパイラ）でそのような枝刈りのステップを見つけたら、その分岐がそれでも実行されたら何が起きるかを問うこと。また、2018年の修正の逆も監査すること。*値*を強化する対策（マスキング、クランプ、飽和）は、解析器が一度も探索しなかった*経路*に対しては何もしない。その逆も同様である。

## 関連項目 {#see-also}

- [BPF Verifier](../bpf-verifier.md) — 経路探索、レジスタ状態の追跡、デッドコード除去。このバグはそのすべての内側にある
- [非特権BPFのデフォルト無効化](unprivileged-bpf-off.md) — 同じリリースサイクルで投入されたハードニング上の対応
- [ALU32ビット演算の範囲追跡](alu32-bitwise-bounds.md) — 6週間前に公開された、投機実行とは無関係のverifierのバグ
- [カーネルハードニング](../../security/kernel-hardening.md) — これが位置する、より広い緩和策の全体像
- [BPFマップ](../bpf-maps.md) — 配列マップ。そのインデックス処理を2018年のマスキング修正が書き換えた

## 外部参考資料 {#external-references}

- [NVD: CVE-2021-33624](https://nvd.nist.gov/vuln/detail/CVE-2021-33624) — CVSS 4.7 MEDIUM（`CVSS:3.1/AV:L/AC:H/PR:L/UI:N/S:U/C:H/I:N/A:N`）、2021年6月23日公開
- [git.kernel.org: 9183671af6db](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9183671af6dbf60a1219371d4ed73e23f43b49db) — "bpf: Fix leakage under speculation on mispredicted branches"。修正本体。細工した2つのプログラムと、却下された代替設計を含む
- [git.kernel.org: b2157399cc98](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b2157399cc9898260d6031c5bfe45fe137c1fbe7) — "bpf: prevent out-of-bounds speculation"（v4.15-rc8）。Spectre v1のBPF側に対する2018年のインデックスマスキング修正
- [git.kernel.org: 801c6058d14a](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=801c6058d14a82179a7ee17a4b532cac6fad067f) — "bpf: Fix leakage of uninitialized bpf stack under speculation"（2021年4月）。隣接する投機的漏洩の修正
- [LWN: Spectre revisits BPF](https://lwn.net/Articles/860597/) — Jonathan Corbet、2021年6月24日。脆弱性、修正、そのコストの解説
- [lore.kernel.org: bpf: backport fixes for CVE-2021-33624 (5.4)](https://lore.kernel.org/all/20210805155343.3618696-1-ovidiu.panait@windriver.com/) — メインラインの2か月後に5.4 LTSブランチへ修正を持ち込んだ、6パッチの手作業バックポート
- [NVD: CVE-2017-5753](https://nvd.nist.gov/vuln/detail/CVE-2017-5753) — Spectre Variant 1。2018年のBPFマスキングパッチが対処したハードウェア脆弱性
