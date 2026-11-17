# ALU32ビット演算の境界追跡 {#alu32-bitwise-bounds-tracking}

> CVE-2021-3490 — 32ビットの条件をチェックしながら64ビットの保証に依存していたコメント

公開
:   2021年5月11日（oss-security）

報告者
:   RedRocket CTFチームのManfred Paul（@_manfp）（Trend MicroのZero Day Initiativeと協力、ZDI-CAN-13590）、およびCanonicalのThadeu Lima de Souza Cascardo。両者とも修正に`Reported-by:`として記載されている

CVSS
:   7.8 HIGH（`CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`、NVDの一次評価）。Ubuntuのセカンダリ評価も7.8だが、ベクトルが異なる（`AC:H`、`S:C`）

修正バージョン
:   [`049c4e13714e`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=049c4e13714ecbca567b4d5f6d563f05d431c80e)、メインライン v5.13-rc4。stable 5.12.4、5.11.21、5.10.37

エクスプロイトツール
:   あり — NVDは、公開されたPacket Stormのエントリ"Linux eBPF ALU32 32-bit Invalid Bounds Tracking Local Privilege Escalation"に`Exploit`タグを付けている。Exploit-DBのエントリはない

実際の悪用
:   確認された事例なし（[CISA KEVに未掲載](https://nvd.nist.gov/vuln/detail/CVE-2021-3490)）

*[事件簿: BPF verifierのバグとCVE](../war-stories.md)の一部。*

## 発生前の状態 {#before-state}

[`3f50f132d840`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3f50f132d8400e129fc9eb68b5020167ef80a244)（"bpf: Verifier, do explicit ALU32 bounds tracking"、v5.7-rc1）以降、verifierは算術演算とビット演算のALU操作を2回シミュレートする。1回は64ビットの境界に対して、もう1回は別個の32ビットの`{s,u}32_{min,max}_value`境界に対してである。ビット演算の場合、これは2つの並行した関数ファミリーを意味する。`scalar_min_max_and()` / `scalar_min_max_or()` / `scalar_min_max_xor()`と、それらの`scalar32_`版である。

32ビット版はそれぞれ近道から始まっていた。`scalar32_min_max_and()`から引用する。

```c
	/* scalar64_min_max_and が呼ばれることを前提としているので、
	 * 32 ビットが既知の場合はレジスタの更新をスキップしても安全である。
	 */
	if (src_known && dst_known)
		return;
```

その理屈はこうである。ソースとデスティネーションのサブレジスタが両方とも既知の定数であれば、演算の結果は既知の定数であり、直後に実行される64ビットの関数がレジスタをその定数に設定するので、32ビットの関数は何もする必要がない。

ほぼ同一のコメントと同じ早期リターンが`scalar32_min_max_or()`にもある。`xor`版はより新しい。[`2921c90d4718`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2921c90d471889242c24cff529043afb378937fa)（"bpf: Fix a verifier failure with xor"、v5.10-rc1）が数か月後に`scalar32_min_max_xor()`を追加し、それは`or`版のコメントを関数名だけ調整してコピーするところから始まっている。

そのコミット自身のレビューこそ、このCVEの正確な前提条件が検討され、安全だと宣言された場所である。Alexei Starovoitovは、このページが扱うのと同じ`if (src_known && dst_known) return;`の行を引用して[直接尋ねた](https://lore.kernel.org/all/20200826015836.2rlfvhoznylkabp6@ast-mbp.dhcp.thefacebook.com/): "why? I've looked at `_and()` and `_or()` variants that do the same and couldn't quite remember why it's ok to do so."（訳: なぜ？同じことをしている`_and()`と`_or()`版を見たが、なぜそうしてよいのかよく思い出せなかった。）Songの[返信](https://lore.kernel.org/all/f2056e3c-e300-6fa0-8b8e-fa19ed5580bd@fb.com/)は率直で、確信がなかった: "Yes, I copied what `_and()` and `_or()` did. What I thought is if both known, 64bit `scalar_min_max_xor()` handled this and did not go through the approximation below... John, could you confirm?"（訳: はい、`_and()`と`_or()`がしていることをコピーした。私が考えたのは、両方が既知なら64ビットの`scalar_min_max_xor()`がこれを処理し、下の近似を通らないということだ……John、確認してもらえるか？）John Fastabendはそれから、このCVEが後に悪用することになる正確なレジスタ状態（非定数の64ビットレジスタの中にある定数のサブレジスタ）を[一つずつたどり](https://lore.kernel.org/all/5f46dcd8c0156_50e8208f4@john-XPS-13-9370.notmuch/)、問題ないと結論づけた。汎用の後処理パスが"will use the `var_off`, previously updated. The 32-bit bounds are then updated using this `var_off` so they are correct even if less precise than we might expect."（訳: 以前に更新された`var_off`を使う。その後、32ビットの境界はこの`var_off`を使って更新されるので、期待するほど精密ではないかもしれないが正しい。）からである。彼が説明していたのは`__update_reg32_bounds()`、このページの「きっかけ」の節で説明するのと同じ、狭めることしかしない関数である。そして彼は間違っていた。この関数は狭めることしかできない。最大値を正しい値まで引き下げ、古い最小値はそのまま残した。それこそがCVE-2021-3490が悪用した`u32_min_value=1, u32_max_value=0`の状態である。彼は約束したテストを確かに書いた。[`99d4def4d085`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=99d4def4d08507474b250dad6345d14715a4726b)（"bpf: Add AND verifier test case where 32bit and 64bit bounds differ"）が1か月後に`verifier/and.c`に"check known subreg with unknown reg"ケースを追加した。それはプログラムがロードされて0を返すことをアサートしたが、結果の境界が自己矛盾していないことはアサートしなかった。そのため、7か月間脆弱なパスを実行しながら、それを検出することはなかった。

## きっかけ {#the-trigger}

この近道の前提は、それが委ねている関数と突き合わせると成り立たない。

`src_known`と`dst_known`は`tnum_subreg_is_const()`で計算される。これらは**下位32ビット**についての記述である。しかし`scalar_min_max_and()`が既知の定数を割り当てるのは、ソースとデスティネーションが**64ビット全体**で既知の場合だけである。2つの条件の間には隙間があり、そこに入るレジスタはどれも32ビットの境界が古いまま残される。`adjust_scalar_min_max_vals()`の最後にある汎用の後処理パス（`__update_reg32_bounds()`）が、その後それらを`var_off`と照合するが、狭めることしかできず広げることはできないので、最大値を正しい値まで引き下げ、古い最小値は置き去りにする。

修正コミットの例:

> consider a register R2 which has a tnum of `0xffffffff00000000`, meaning, lower 32 bits are known constant and in this case of value `0x00000001`. R2 is then and'ed with a register R3 which is a 64 bit known constant, here, `0x100000002`.

（訳: tnumが`0xffffffff00000000`であるレジスタR2を考える。これは、下位32ビットが既知の定数で、このケースでは値が`0x00000001`であることを意味する。次にR2は、64ビットの既知の定数（ここでは`0x100000002`）であるレジスタR3とandされる。）

R2の下位半分は既知（1）である。R3は完全に既知である。したがって`src_known && dst_known`が成り立ち、`scalar32_min_max_and()`はすぐにリターンする。しかしR2の上位半分は未知なので、`scalar_min_max_and()`はレジスタを定数とマークすることを見送り、32ビットの境界を更新するものは何もない。

コミット内のverifierのトレースが、その惨状を示している。

```
9: (5f) r2 &= r3
10: R2_w=inv(id=0,...,var_off=(0x0; 0x100000000),
             s32_min_value=1,s32_max_value=0,
             u32_min_value=1,u32_max_value=0)
```

`u32_min_value=1`で`u32_max_value=0`である。32ビット空間では`0x1 & 0x2 == 0`なので、正しい答えは4つすべてが0である。

## 観測された挙動 {#observed-behavior}

最小値が最大値を超えるレジスタは空集合を表す。どんな実際の値もとりえない状態であり、それはverifierがそこから導くその後のあらゆる推論が現実によって制約されないことを意味する。

2021年5月11日に公開されたCascardoによるoss-securityの告知は、その帰結を次のように述べている。

> Manfred Paul (@_manfp) of the RedRocket CTF team (@redrocket_ctf) working with Trend Micro's Zero Day Initiative discovered that this vulnerability could be turned into out-of-bounds reads and writes in the kernel.

（訳: Trend MicroのZero Day Initiativeと協力しているRedRocket CTFチーム（@redrocket_ctf）のManfred Paul（@_manfp）が、この脆弱性をカーネル内の範囲外読み書きに変えられることを発見した。）

NVDの説明もこれに同意し、さらに一歩進めている: "which could be turned into out of bounds reads and writes in the Linux kernel and therefore, arbitrary code execution."（訳: これはLinuxカーネル内の範囲外読み書きに変えることができ、したがって任意のコード実行に変えられる。）

動作するローカル権限昇格のエクスプロイトが公開されている。NVDの参照リストには、"Linux eBPF ALU32 32-bit Invalid Bounds Tracking Local Privilege Escalation"という題のPacket Stormのエントリが`Exploit`タグ付きで載っている。CISAのKEVカタログの時点では、実環境での悪用は確認されていない。これは[ネットワークスタックの事件簿](../../net/war-stories.md)の大半に通底する、「武器化された」と「使われた」の間の同じ隔たりである。

Cascardoの告知には、影響範囲を実質的に限定した一つの詳細が含まれている: "There has been no backport to any upstream LTS kernel."（訳: アップストリームのどのLTSカーネルにもバックポートされていない。）これはバックポートについての話である。5.10はそれ自体が長期サポート系列であり、バグをネイティブに持っていた（そのため上記の5.10.37の修正がある）が、`and`/`or`版は5.7-rc1で、`xor`版は5.10-rc1で導入されたため、より古い4.19と5.4の長期サポート系列はこれらのコミットを一切受け取らなかった。

## なぜ起きたか {#why-it-happened}

**不変条件を主張するコメントは不変条件ではない。** "Assuming `scalar64_min_max_and` will be called so its safe to skip updating register for known 32-bit case"（`scalar64_min_max_and`が呼ばれることを前提としているので、32ビットが既知の場合はレジスタの更新をスキップしても安全である）は、*別の関数の*振る舞いについての主張であり、この関数の中に書かれ、誰にもチェックされていなかった。それはほぼ真だった。完全に既知のレジスタ、つまり一般的なケースについては。そしてそれが偽になるケースは、一方の半分が定数で他方の半分が未知のレジスタであり、それはまさに悪意あるプログラムが意図的に構築できる状態である。

**2つの並行した実装はずれていき、そのずれはファイル単位では見えない。** 修正コミットはこれを直接指摘している: "Given `scalar32_min_max_*()` is intended to be designed as closely as possible to `scalar_min_max_*()`"（`scalar32_min_max_*()`は`scalar_min_max_*()`にできるだけ近く設計されることが意図されているので）。32ビットのファミリーは64ビットのファミリーを鏡写しにするように書かれていた。鏡写しは設計目標であり、同時に故障モードでもあった。「できるだけ近く」は「同一」ではなく、その違いはそれぞれが使う前提条件に現れるからである。

**コメントをコピーすると、その前提もコピーされる。** `xor`版はバグが持ち込まれた時点では存在しなかった。`2921c90d4718`自体がバグ修正であり、LLVM 11と12のもとでBPFセルフテストを壊していたverifierの失敗に対するものだった。そして欠けていた32ビットの`xor`パスを追加する際に、隣の`and`/`or`から委譲の近道を、欠陥のある理屈ごと複製した。それが、修正に`Fixes:`タグが**2つ**付いている理由である。

```
Fixes: 3f50f132d840 ("bpf: Verifier, do explicit ALU32 bounds tracking")
Fixes: 2921c90d4718 ("bpf: Fix a verifier failure with xor")
```

両者の間には5か月の隔たりがあり、2つ目は1つ目のバグを新しい呼び出し箇所に伝播させた修正である。

## 解決 {#resolution}

[`049c4e13714e`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=049c4e13714ecbca567b4d5f6d563f05d431c80e)（"bpf: Fix alu32 const subreg bound tracking on bitwise operations"、Daniel Borkmann）は、前提を真にしようとするのではなく、前提を削除する。同じ変更が3つの関数すべてに適用されている。

```diff
-	/* Assuming scalar64_min_max_and will be called so its safe
-	 * to skip updating register for known 32-bit case.
-	 */
-	if (src_known && dst_known)
+	if (src_known && dst_known) {
+		__mark_reg32_known(dst_reg, var32_off.value);
 		return;
+	}
```

64ビットのパスに委ねる代わりに、32ビットのパスが32ビットの境界を自ら設定するようになった。それが可能なのは、答えがすでにローカル変数にあるからである。`var32_off`は`tnum_subreg(dst_reg->var_off)`であり、コミットが記しているように、"This is possible given `var32_off` already holds the final value as `dst_reg->var_off` is updated before calling `scalar32_min_max_*()`."（訳: `scalar32_min_max_*()`を呼ぶ前に`dst_reg->var_off`が更新されるので、`var32_off`はすでに最終的な値を保持しており、これが可能である。）正しい値はずっと利用可能だった。コードがそれを使っていなかっただけである。

パッチには`Reviewed-by: John Fastabend`と`Acked-by: Alexei Starovoitov`が付いている。この特定のパッチには公開投稿への`Link:`トレーラーがない。これは非公開のチャネルで処理され、修正が出荷された時点でoss-securityで告知された報告であることと整合しており、[セキュリティサブシステムの事件簿](../../security/war-stories.md)全体で見られるのと同じパターンである。しかし、この修正が答えている設計上の問題は、9か月前に、上で説明した`xor`コミット自身のレビュースレッドで公に議論*されていた*。その記録がこのコミットに紐付いていないだけである。

修正はv5.13-rc4でメインラインに、そして5.12.4、5.11.21、5.10.37のstableリリースに入った。

## 教訓 {#what-it-taught-us}

**呼び出し元とその委譲先の間の前提条件の不一致は、静かなバグのクラスである。** この関数は32ビットの条件をチェックし、64ビットの条件に基づいて動作する関数に依存していた。どちらも単体では間違っていない。バグは完全に両者の間の空間に存在し、単一の関数のレビューでは浮かび上がらない。そして、一方の半分が定数で他方の半分が未知のレジスタを構築しようと誰かが思いつかない限り、どのテストもそれを捕捉しない。

**「もう一方のパスが処理する」は、もう一方のパスが変わるたびに、前提とするのではなくチェックする必要がある。** このコメントは、5か月後に別の作者によって追加されたまったく新しい関数を含む3つの別々の関数で、レビューを生き延びるほどもっともらしかった。

**ありえないレジスタ状態は検出可能な不変条件である。** `u32_min_value > u32_max_value`はどんな実際の値も表せない。その不整合は、誰かが見た瞬間にverifier自身のデバッグ出力で見えていた。そしてそれは、シミュレーションの各ステップで実行されるレジスタ状態に対する安価なアサーションがあれば、どの関数がそれを持ち込んだかにかかわらず、すぐに捕捉できた種類のものである。

!!! warning "注意すべきパターン"
    異なる幅、符号の有無、表現（32ビットと64ビットのパス、高速パスと低速パス、キャッシュありとなしの変種）にわたる同じロジックの2つの並行した実装を見つけたら、本体ではなく*早期リターン*を最初に読むこと。早期リターンは一方の実装がもう一方に委譲する場所であり、委譲は前提条件が静かに食い違う場所である。次に、下流の状態が自己矛盾に陥りうるか（`min > max`、`end < start`、`count > capacity`）を確認し、もしそうなら、すべての生成元が正しいことに頼るのではなく、それをアサートすること。

## 関連項目 {#see-also}

- [BPF Verifier](../bpf-verifier.md) — レジスタ境界の追跡、tnum、そしてALU操作がどのようにシミュレートされるか
- [32ビット符号なし境界の伝播バグ](alu32-unsigned-bounds.md) — 同じ導入コミットに対する2021年のもう一つのCVEで、同じ研究者が報告し、2週間半早く修正されたもの
- [The jmp32 Bounds Regression](jmp32-bounds.md) — その解決が`3f50f132d840`を生み出した2020年のCVE
- [Unprivileged BPF Off by Default](unprivileged-bpf-off.md) — このCVEが告知されたのと同じ日に投稿された
- [セキュリティの事件簿](../../security/war-stories.md) — 同じ「エンバーゴ後に告知」という公開パターンが繰り返し現れる場所

## 外部参考資料 {#external-references}

- [NVD: CVE-2021-3490](https://nvd.nist.gov/vuln/detail/CVE-2021-3490) — CVSS 7.8 HIGH、2021年6月4日公開。説明では両方の導入コミットと3つのstableリリースすべてが挙げられている
- [git.kernel.org: 049c4e13714e](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=049c4e13714ecbca567b4d5f6d563f05d431c80e) — "bpf: Fix alu32 const subreg bound tracking on bitwise operations"、修正コミットで、修正前後のverifierトレースを含む
- [git.kernel.org: 3f50f132d840](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3f50f132d8400e129fc9eb68b5020167ef80a244) — "bpf: Verifier, do explicit ALU32 bounds tracking"（v5.7-rc1）、1つ目の`Fixes:`の対象
- [git.kernel.org: 2921c90d4718](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2921c90d471889242c24cff529043afb378937fa) — "bpf: Fix a verifier failure with xor"（v5.10-rc1）、2つ目の`Fixes:`の対象で、欠陥のある近道を新しい関数にコピーしたもの
- [lore.kernel.org: bpf: fix a verifier failure with xor](https://lore.kernel.org/all/20200825064608.2017937-1-yhs@fb.com/) — CVE-2021-3490が悪用する正確な前提条件が、CVEの9か月前に検討され安全だと宣言された2020年8月のレビュースレッド
- [oss-security: CVE-2021-3490](https://www.openwall.com/lists/oss-security/2021/05/11/11) — Thadeu Lima de Souza Cascardoによる2021年5月11日の告知。Manfred Paulの功績を記し、導入コミットがより古いLTS系列には届かなかったことを記している
