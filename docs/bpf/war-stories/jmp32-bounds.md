# jmp32の範囲追跡リグレッション {#the-jmp32-bounds-regression}

> CVE-2020-8835 — 精度向上のパッチが、それをバグ修正と誤認した機械学習の分類器によってstableカーネルに自動選択され、結果として権限昇格になった

公開日
:   2020年3月30日（oss-security）

実証者
:   Manfred Paul、ZDIのPwn2Own 2020コンテストにて（ZDI-CAN-10780）。修正のきっかけとなったハングはAnatoly Trosinenkoが、kBdyschハーネスによるファジングで報告した

CVSS
:   7.8 HIGH（`CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`、NVDの主評価）。Ubuntuの副次的なスコアも7.8だが、ベクトルが異なる（`AC:H`、`S:C`）

混入
:   [`581738a681b6`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=581738a681b6faae5725c2555439189ca81c0f1f)（v5.5）、5.4-stableにバックポート

修正
:   [`f2d67fec0b43`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f2d67fec0b43edce8c416101cdc52e71145b5fef)、メインライン v5.7-rc1。stable 5.6.1、5.5.14、5.4.29

エクスプロイトツール
:   Exploit-DBのエントリはない。NVDは後のoss-security投稿に`Exploit`タグを付けている

実際の悪用
:   確認された事例はない（[CISA KEVには未掲載](https://nvd.nist.gov/vuln/detail/CVE-2020-8835)）

*[事件簿: BPF verifierのバグとCVE](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

BPFには32ビットの条件ジャンプ（`jmp32`。LLVMに`-mcpu=v3`を指定すると生成される）があり、2019年11月、Yonghong Songの[`581738a681b6`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=581738a681b6faae5725c2555439189ca81c0f1f)（"bpf: Provide better register bounds after jmp32 instructions"）は、verifierがそこから情報を学べるようにした。パッチはbpfリストで[レビューを経た](https://lore.kernel.org/all/20191121045924.v77wb5zzfliln7ql@ast-mbp.dhcp.thefacebook.com/)が、Alexei Starovoitovの唯一のコメントは`__reg_bound_offset32()`の*スタイル*についてのもの（"may be make sense to do it as a helper?"、訳: ヘルパーとしてやったほうがよいのでは？）であり、2人目のレビュアーへの直接の質問 —「Ed, how would you simplify `__reg_bound_offset32` logic?」（訳: Ed、`__reg_bound_offset32`のロジックをどう単純化する？）— には回答がなかった。マスキングという前提そのものが問われることは一度もなかった。

verifierは各スカラーレジスタを2つの方法で追跡する。数値の区間（`umin_value` … `umax_value`）として、そして**tnum**、つまりどの個々のビットが既知でどれが未知かを記録する`(value, mask)`のペアとしてである。`581738a681b6`は、32ビット比較の後にtnumを絞り込むヘルパーを追加した:

```c
static void __reg_bound_offset32(struct bpf_reg_state *reg)
{
	u64 mask = 0xffffFFFF;
	struct tnum range = tnum_range(reg->umin_value & mask,
				       reg->umax_value & mask);
	struct tnum lo32 = tnum_cast(reg->var_off, 4);
	struct tnum hi32 = tnum_lshift(tnum_rshift(reg->var_off, 32), 32);

	reg->var_off = tnum_or(hi32, tnum_intersect(lo32, range));
}
```

最初の2つの文を読んでほしい。レジスタの64ビットの範囲を取り、両端点を下位32ビットにマスクし、その結果をレジスタの32ビットの範囲として扱っている。

当時、これは修正ではなく精度の向上だった。それまで拒否していた正しいプログラムをverifierが受け入れるようにするものである。この区別は、次に起きたことにとって重要である。

## きっかけ {#the-trigger}

両端点をマスクすることが正しいのは、2つの端点の上位32ビットが一致する場合だけである。

`umin = 0x2000000000`、`umax = 0x4000000000`で範囲が制限されたレジスタを考える。マスクすると`0x0`と`0x0`になるので、`tnum_range()`は範囲`[0, 0]`を生成し、verifierは下位半分が*確実にゼロ*だと結論づける。実際には、`[0x2000000000, 0x4000000000]`のどこかにある値は、下位32ビットとしてどんな値でも取りうる。正しい答えは`[0x0, 0xffffffff]`、つまり何の情報もない、である。

修正コミットは、レジスタ状態を一つひとつ追って説明している。2つの64ビットジャンプと、続く2つの32ビットジャンプでr1の範囲を制限した後:

```
Thus, after knowing r1 <= 0x4000000000 and r1 >= 0x2000000000 and
                    w1 <= 0x400        and w1 >= 0x200:

  max: 0b100000000000000000000000000000000000000 / 0x4000000000
  var: 0b111111100000000000000000000000000000000 / 0x7f00000000
  min: 0b010000000000000000000000000000000000000 / 0x2000000000
```

その`var`の行に対するBorkmannの判定:

> A outcome of `0x7f00000000` is not correct since it would contradict the earlier probed bounds where we know that the result should have been in `[0x200,0x400]` in u32 space. Therefore, tests with such info will lead to wrong verifier assumptions later on like falsely predicting conditional jumps to be always taken, etc.

（訳: `0x7f00000000`という結果は正しくない。結果がu32空間で`[0x200,0x400]`にあるはずだと分かっている、先に調べた範囲と矛盾するからである。したがって、そのような情報を使った判定は、後で条件ジャンプが常に成立すると誤って予測するなど、verifierの誤った仮定につながる。）

verifierはいまや、同じレジスタについて両方は真になりえない2つの言明を保持している。下流のすべてがその矛盾を受け継ぐ。

## 観測された挙動 {#observed-behavior}

これがどう表面化したかは、記録の中でもとりわけ印象的な点である。**カーネルが、ハングするプログラムを生成したのである**。

kBdyschハーネスでファジングしていたAnatoly Trosinenkoがハングを報告した。Borkmannの調査により、verifierがある条件ジャンプは常に成立すると思い込み、もう一方の分岐をデッドとしてマークし、それらの命令を`goto pc-1`、つまり自分自身への無条件ジャンプに書き換えていたことが分かった。そして現実はシミュレーションから乖離した:

> The verifier rewrote original instructions it recognized as dead code with 'goto pc-1', but reality differs from verifier simulation in that we're actually able to trigger a hang due to hitting the 'goto pc-1' instructions.

（訳: verifierはデッドコードと認識した元の命令を'goto pc-1'で書き換えたが、現実はverifierのシミュレーションと異なり、実際に'goto pc-1'命令に到達してハングを引き起こすことができる。）

これはこのバグの最も穏やかな現れ方である。verifierのデッドコード除去は、そのジャンプ予測と同程度にしか健全ではなく、誤ったジャンプ予測は生きたコードを無限ループに変えてしまう。

深刻な現れ方は、Manfred Paulが実証したものである。修正と同じ日に公開された、Steve Beattieによるoss-securityの告知:

> Manfred Paul, as part of the ZDI pwn2own competition, demonstrated that a flaw existed in the bpf verifier for 32bit operations. [...] The result is that register bounds were improperly calculated, allowing out-of-bounds reads and writes to occur.

（訳: Manfred PaulはZDIのpwn2ownコンテストにおいて、32ビット演算に関するbpf verifierの欠陥を実証した。[...] その結果、レジスタの範囲が不適切に計算され、範囲外の読み書きが可能になる。）

そして到達可能性について、同じ告知から:「This bpf functionality is available to unprivileged users unless the `kernel.unprivileged_bpf_disabled` sysctl is set to 1.」（訳: このbpfの機能は、`kernel.unprivileged_bpf_disabled` sysctlが1に設定されていない限り、非特権ユーザーが利用できる。）当時、デフォルトは0だった。

## なぜ起きたか {#why-it-happened}

ここには2つの別々の失敗があり、verifierの中にあるのはそのうち1つだけである。

**算術上の失敗**は、値ではなく区間に適用されたマスク操作である。`umin & 0xffffffff`と`umax & 0xffffffff`はそれぞれ明確に定義されているが、その間の区間は、元の区間が2³²の境界をまたがない限り、下位半分が取りうる値の区間ではない。これは符号付き／符号なしの切り詰めのバグと同じ形である。縮小変換を端点に適用し、それが端点の定義する範囲と可換であると仮定している。実際には可換ではない。

**プロセス上の失敗**のほうがより興味深く、Borkmannはコミットメッセージでそれを名指ししている:

> apparently `581738a681b6` got auto-selected by Sasha's ML system and misclassified as a fix, so it got sucked into v5.4 where it should never have landed.

（訳: どうやら`581738a681b6`はSashaのMLシステムによって自動選択され、修正と誤分類されたため、本来決して入るべきではなかったv5.4に取り込まれてしまったようだ。）

`581738a681b6`は、`-mcpu=v3`でコンパイルされたBPFプログラムのための精度向上だった。stableカーネル向けを意図したものでは決してなかった。カーネルの自動stableバックポート分類器、つまり作者がタグ付けし忘れた修正を探してメインラインのコミットを走査する機械学習モデルが、それを修正らしいと判断し、5.4の長期サポートシリーズにバックポートされた。影響を受けた導入環境の大部分はそこから来ている。oss-securityの告知によれば、バックポートはコミット`b4de258dede5`として5.4-stableに入った。

つまり、脆弱な期間に曝された範囲は、その機能の実際の利用者よりもはるかに広かった。5.4で誰も求めていなかった機能が、それでも5.4で出荷されたのである。

## 解決 {#resolution}

選ばれた修正は**修理ではなくリバート**であり、その理由はstableツリーの読者に向けて詳しく説明されている。

Borkmannはまず修理を試みた。その試みはコミットメッセージに記録されている。レジスタを一時変数に移し、その一時変数に`coerce_reg_to_size(&tmp, 4)`を実行して正しく導出された32ビットの範囲を得て、そこからtnumの範囲を構築するというものである。そして、なぜそれを断念したかを説明している:

> However, above new `__reg_bound_offset32()` has no effect on refining the knowledge of the register contents. Meaning, if the bounds in hi32 range mismatch we'll get the identity function [...] Likewise, if the bounds in hi32 range match, then we mask both bounds with `0xffffffff` [...] However, _prior_ called `__reg_bound_offset()` did already such intersection on the full reg and we therefore would only repeat the same operation on the lo32 part twice.

（訳: しかし、上記の新しい`__reg_bound_offset32()`は、レジスタの内容についての知識を絞り込む効果を持たない。つまり、hi32の範囲で範囲が一致しなければ恒等関数になる [...] 同様に、hi32の範囲で範囲が一致すれば、両方の範囲を`0xffffffff`でマスクする [...] しかし、_先に_ 呼ばれた`__reg_bound_offset()`がすでにレジスタ全体でそのような共通部分を取っているので、lo32部分で同じ操作を2回繰り返すだけになる。）

修正版は、verifierがすでに行っていなかったことを何もしない。そこで[`f2d67fec0b43`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f2d67fec0b43edce8c416101cdc52e71145b5fef)（"bpf: Undo incorrect `__reg_bound_offset32` handling"）は、ヘルパーと2組の呼び出し箇所を削除した。19行の純粋な削除であり、stableツリー向けの明示的な正当化が添えられている:

> Given this has no effect and the original commit had false assumptions, this patch reverts the code entirely which is also more straight forward for stable trees [...] A proper bounds refinement would need a significantly more complex approach which is currently being worked, but no stable material. Hence revert is best option for stable.

（訳: これには効果がなく、元のコミットは誤った仮定に基づいていたため、このパッチはコードを完全にリバートする。stableツリーにとってもそのほうが単純である [...] 適切な範囲の絞り込みには大幅に複雑なアプローチが必要で、現在取り組まれているが、stable向けの材料ではない。したがってstableにとってはリバートが最善の選択肢である。）

コミットメッセージには、リバート後のverifierがTrosinenkoの再現プログラムを正しく拒否する様子まで示されている。

**置き換えの実装は同じシリーズで届いた**。2020年3月30日に`bpf@vger.kernel.org`へ投稿された[3パッチのシリーズ](https://lore.kernel.org/all/20200330160324.15259-1-daniel@iogearbox.net/)は、Borkmannのリバートと、**Jann Horn**による2つのパッチだった。Hornの[`604dca5e3af1`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=604dca5e3af1db98bd123b7bfc02b017af99e3a0)（"bpf: Fix tnum constraints for 32-bit comparisons"）は、元の欠陥を1文で診断している:

> However, the implementation from `581738a681b6` didn't compute the tnum constraint based on the fixed operand, but instead derives it from the arithmetic-range-based tracking.

（訳: しかし、`581738a681b6`の実装は、固定のオペランドに基づいてtnumの制約を計算するのではなく、算術的な範囲に基づく追跡からそれを導出していた。）

そしてそれを、レジスタの既存の数値範囲からではなく、比較の実際の定数オペランドからtnumの範囲を構築する`set_upper_bound()`／`set_lower_bound()`ヘルパーで置き換えた。2つ目のパッチ（シリーズの3/3）は、`reg_set_min_max_inv()`を、オペコードを反転させた`reg_set_min_max()`に統合した。その非対称性はclassic BPFでは意味があったが、eBPFでは意味がないという理由である。

このシリーズに対するAlexei Starovoitovの公開の返信は、これがすべてだった:

> Applied. Thanks

**そしてその修正自体も、同じマージウィンドウの中で置き換えられた。Borkmannのリバートが「現在取り組まれている」とすでに注記していた作業によってである**。John Fastabendの[`3f50f132d840`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3f50f132d8400e129fc9eb68b5020167ef80a244)（"bpf: Verifier, do explicit ALU32 bounds tracking"）は同じ日に作成され、同じリリースv5.7-rc1に入った。これはHornの`set_upper_bound()`／`set_lower_bound()`を削除し、tnumを流用するアプローチを完全に捨て、32ビットの知識をtnumにエンコードする代わりに、すべてのレジスタに本物の2組目の32ビット範囲（`s32_min_value`、`s32_max_value`、`u32_min_value`、`u32_max_value`）を与えた。この設計は本格的なレビューを受けた。[v1](https://lore.kernel.org/all/158507130343.15666.8018068546764556975.stgit@john-Precision-5820-Tower/)で、Starovoitovは負の32ビット範囲に対する符号拡張ロジックの本物の正しさのバグを見つけた —「looks like above will not be correct for negative `s32_min`/`max`」（訳: 上の部分は負の`s32_min`/`max`に対して正しくないようだ）— Fastabendはそれを修正し、[v2](https://lore.kernel.org/all/158560409224.10843.3588655801186916301.stgit@john-Precision-5820-Tower/)がマージされた。

それは正しい設計であり、同時に1年後の[CVE-2021-3490](alu32-bitwise-bounds.md)と[CVE-2021-31440](alu32-unsigned-bounds.md)の`Fixes:`タグで名指しされたコミットでもある。直接の系譜は次のようになる。精度向上 → CVE → リバートと適切な実装 → より良い適切な実装 → さらに2つのCVE。

## 教訓 {#what-it-taught-us}

**stableへのバックポートの自動選択は、ある機能のバグに誰が曝されるかを変える**。`581738a681b6`は、狭い利用者、つまり最近のカーネルで`-mcpu=v3`を使ってBPFをコンパイルする人々にとっては十分に正しい最適化だった。それを修正として分類したことで、5.4を動かすすべての人の前に置かれた。分類器の誤りはパッチの品質の判断ではなく、その*カテゴリ*の判断にあった。そしてカテゴリこそが被害範囲を決める。

**「verifierを賢くするだけ」というのはセキュリティ上の主張である**。verifierにおける精度向上は中立ではない。verifierが引き出す推論が1つ増えるたびに、誤った推論を引き出す機会も1つ増え、安全性チェッカーにおける誤った推論は回避手段になる。精度向上のパッチを、セキュリティパッチに求められる厳密さでレビューすること、それが教訓である。それを定着させるにはPwn2Ownのデモが必要だった。

**リバートは正当な修正であり、ときにはstableに適した唯一の修正である**。Borkmannは動作する修理を手にしていながら、それを却下した。情報の利得がゼロのまま複雑さを増やすこと、そして本当の解決策はstableには侵襲的すぎることが理由である。これはメンテナが「コードはどうあるべきか」と「今日5.4のユーザーに何を出荷すべきか」を明示的に切り分けたものであり、コミットメッセージはその両方について過程を示す点で異例なほど優れている。

**マスキングは範囲演算ではない**。`[a & M, b & M]`が下位ビットの範囲になるのは、`a`と`b`が上位ビットを共有する場合だけである。区間の端点に縮小変換を適用する場所ではどこでも、その前提条件を明示的にチェックする必要がある。

!!! warning "注意すべきパターン"
    grepすべきものは2つある。1つ目は、結果が範囲として使われる`min & MASK`／`max & MASK`のペア（あるいは範囲の端点に対する`(u32)`、`(s32)`、`(u16)`のキャスト）である。これは捨てられる上位ビットが両端点で等しいと証明できる場合にのみ正しい。2つ目は、自分のサブシステムのstableブランチで、アップストリームでの件名が修正ではなく機能追加や最適化のように読めるバックポート済みコミットを探すことである。自動選択は無謬ではなく、長期サポートブランチ上の機能には、それがレビューされた際には想定されていなかった利用者がいる。

## 関連項目 {#see-also}

- [BPF Verifier](../bpf-verifier.md) — tnum、レジスタの範囲、そしてこのバグをハングに変えたデッドコード除去パス
- [ALU32ビット演算の範囲追跡](alu32-bitwise-bounds.md) — このバグの修正を置き換えた設計である`3f50f132d840`におけるCVE
- [32ビット符号なし範囲伝播のバグ](alu32-unsigned-bounds.md) — その置き換え設計におけるもう1つのCVE
- [非特権BPFのデフォルト無効化](unprivileged-bpf-off.md) — このバグが依存していたデフォルトと、それが1年後にどう変わったか
- [`check_alu_op()`における符号拡張](sign-extension.md) — 2年余り前の、同じ種類のバグ（verifierのモデルが実行時の値から乖離する）

## 外部参考資料 {#external-references}

- [NVD: CVE-2020-8835](https://nvd.nist.gov/vuln/detail/CVE-2020-8835) — CVSS 7.8 HIGH、2020年4月2日公開。影響を受けるバージョンとZDIのケース番号を記載
- [git.kernel.org: f2d67fec0b43](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f2d67fec0b43edce8c416101cdc52e71145b5fef) — "bpf: Undo incorrect `__reg_bound_offset32` handling"。リバート本体。再現プログラムの詳細な解説と却下された修理を含む
- [git.kernel.org: 581738a681b6](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=581738a681b6faae5725c2555439189ca81c0f1f) — "bpf: Provide better register bounds after jmp32 instructions"（v5.5）、バグを混入させたコミット
- [lore.kernel.org: bpf: provide better register bounds after jmp32 instructions](https://lore.kernel.org/all/20191121045924.v77wb5zzfliln7ql@ast-mbp.dhcp.thefacebook.com/) — 2019年11月のレビュースレッド。マスキングという前提そのものは一度も問われなかった
- [git.kernel.org: 604dca5e3af1](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=604dca5e3af1db98bd123b7bfc02b017af99e3a0) — "bpf: Fix tnum constraints for 32-bit comparisons"。同じシリーズに含まれたJann Hornによる置き換え実装
- [git.kernel.org: 3f50f132d840](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3f50f132d8400e129fc9eb68b5020167ef80a244) — "bpf: Verifier, do explicit ALU32 bounds tracking"。同じリリースで両者に取って代わった設計
- [lore.kernel.org: ALU32 bounds tracking support (v1)](https://lore.kernel.org/all/158507130343.15666.8018068546764556975.stgit@john-Precision-5820-Tower/) — 設計がマージされる前に、Starovoitovが本物の符号拡張のバグを見つけた場所
- [lore.kernel.org: `[PATCH bpf-next 0/3] Fix __reg_bound_offset32 handling`](https://lore.kernel.org/all/20200330160324.15259-1-daniel@iogearbox.net/) — 3パッチのスレッドと、Starovoitovの2語の返信
- [oss-security: CVE-2020-8835](https://www.openwall.com/lists/oss-security/2020/03/30/3) — Steve Beattieによる告知、2020年3月30日。Pwn2Ownでの帰属、5.4-stableへのバックポートのハッシュ、sysctlによる到達可能性についての注記
