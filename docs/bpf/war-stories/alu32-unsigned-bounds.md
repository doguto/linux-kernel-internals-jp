# 32ビット符号なし境界の伝播バグ {#the-32-bit-unsigned-bounds-propagation-bug}

> CVE-2021-31440 — 4か月半前にマージされた修正は、2行からなるパターンの符号付きの側を直したが、符号なしの側は残したままだった

公開
:   2021年5月3日（[ZDI-21-503](https://www.zerodayinitiative.com/advisories/ZDI-21-503/)、ZDI-CAN-13661。NVDレコードの公開は2021年5月21日）

報告者
:   Manfred Paul（修正コミットの`Reported-by:`タグによる）、Trend MicroのZero Day Initiative経由

CVSS
:   7.0 HIGH（`CVSS:3.1/AV:L/AC:H/PR:L/UI:N/S:U/C:H/I:H/A:H`、NVDの一次評価）。セカンダリ評価者としてのZDIによる評価は8.8 HIGH（`CVSS:3.0/AV:L/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H`）

修正バージョン
:   [`10bf4e83167c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=10bf4e83167cc68595b85fd73bb91e8f2c086e36)、メインラインv5.13-rc1

エクスプロイトツール
:   Exploit-DBには登録なし。NVDには`Exploit`タグの付いた参照はない

実際の悪用
:   確認された事例なし（[CISA KEVに未掲載](https://nvd.nist.gov/vuln/detail/CVE-2021-31440)）

*[事件簿: BPF verifierのバグとCVE](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

[`3f50f132d840`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3f50f132d8400e129fc9eb68b5020167ef80a244)（"bpf: Verifier, do explicit ALU32 bounds tracking"、v5.7-rc1）以降、verifier内のすべてのスカラーレジスタは*2組*の数値境界を持つ。64ビットの`{s,u}{min,max}_value`と、下位半分を表す別個の32ビットの`{s,u}32_{min,max}_value`である。

境界が2組あるということは、それらを一貫した状態に保たなければならないということである。`__reg_combine_64_into_32()`は64ビットの境界から32ビットの境界を導出する方向の処理で、それを端点ごとに1つずつ行っていた。

```c
/* kernel/bpf/verifier.c、修正前 */
if (__reg64_bound_u32(reg->umin_value))
	reg->u32_min_value = (u32)reg->umin_value;
if (__reg64_bound_u32(reg->umax_value))
	reg->u32_max_value = (u32)reg->umax_value;
```

`__reg64_bound_u32(a)`は単に`a > U32_MIN && a < U32_MAX`、つまり「この値は32ビットの範囲の厳密に内側にある」というものである（厳密な比較により0と`U32_MAX`自体は除外される）。

## きっかけ {#the-trigger}

各端点を独立にテストすることこそがバグである。修正コミットは、破られていたルールを次のように述べている。

> That is, really only set the `u32_{min,max}_value` when /both/ `{umin,umax}_value` safely fit in 32 bit space. For example, the register with a `umin_value == 1` does /not/ imply that `u32_min_value` is also equal to 1, since `umax_value` could be much larger than 32 bit subregister can hold, and thus `u32_min_value` is in the interval `[0,1]` instead.

（訳: つまり、`u32_{min,max}_value`を設定するのは、`{umin,umax}_value`の/両方/が32ビット空間に安全に収まる場合に限るべきである。例えば、`umin_value == 1`であるレジスタは、`u32_min_value`も1に等しいことを/意味しない/。`umax_value`は32ビットのサブレジスタが保持できるよりもずっと大きいかもしれず、その場合`u32_min_value`は代わりに区間`[0,1]`に入る。）

64ビットの範囲が`[1, 0xffffffff00000001]`であるレジスタは`0x100000000`を保持でき、その下位32ビットは0である。したがって`u32_min_value = 1`は、正しい事実から導出された、レジスタについての誤った記述である。

コミットは、その結果として生じるverifierのトレースを示している。goto側のパスで`if r2 >= 0x1`と`if w2 <= 0x1`を経て、その後`w2 = w2`（上位半分を捨てる32ビットのmove）を行うと次のようになる。

```
10: R0=inv1337 R1=ctx(id=0,off=0,imm=0) R2_w=inv1 R10=fp0
```

`R2_w=inv1`は、verifierがこのレジスタはちょうど定数1であると確信していることを意味する。実際にはそうではない。誤った`u32_min_value = 1`が、誤った既知定数へと昇格してしまったのである。

## 観測された挙動 {#observed-behavior}

その時点から、verifierは実行時の実際の値をもはや知らないレジスタについて推論していることになる。NVDのCVEの説明は、影響を率直に述べている。

> The specific flaw exists within the handling of eBPF programs. The issue results from the lack of proper validation of user-supplied eBPF programs prior to executing them. An attacker can leverage this vulnerability to escalate privileges and execute arbitrary code in the context of the kernel.

（訳: この欠陥はeBPFプログラムの処理に存在する。この問題は、ユーザーが提供したeBPFプログラムを実行する前に適切な検証が欠けていることに起因する。攻撃者はこの脆弱性を利用して権限を昇格し、カーネルのコンテキストで任意のコードを実行できる。）

2つの評価者は、それがどれほど難しいかについて意味のある形で意見が分かれている。NVDの一次評価は`AC:H`かつスコープ不変で7.0とし、ZDIは`AC:L`かつ`S:C`で8.8と評価した。意見の相違は攻撃の複雑さと影響範囲についてであって、バグが何をするかについてではない。

公開エクスプロイトがPacket Stormに登録されている兄弟バグ[CVE-2021-3490](alu32-bitwise-bounds.md)とは異なり、こちらについてはパッケージ化された公開エクスプロイトは存在しない。NVDの参照リストには`Exploit`タグの付いたエントリはなく、Exploit-DBのエントリもない。

## なぜ起きたか {#why-it-happened}

このまったく同じ形のコードが修正されたのは、これが初めてではない。

2020年12月8日（この修正の4か月半前）、Alexei Starovoitovは、Jean-Philippe Bruckerの報告による[`b02709587ea3`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b02709587ea3d699a608568ee8157d8db4fd8cae)（"bpf: Fix propagation of 32-bit signed bounds from 64-bit bounds."）をコミットした。その論拠は、符号の有無が違うだけで一言一句同じ議論である。

> The 64-bit signed bounds should not affect 32-bit signed bounds unless the verifier knows that upper 32-bits are either all 1s or all 0s. For example the register with `smin_value==1` doesn't mean that `s32_min_value` is also equal to 1, since `smax_value` could be larger than 32-bit subregister can hold.

（訳: verifierが上位32ビットがすべて1かすべて0であることを知っている場合を除き、64ビットの符号付き境界は32ビットの符号付き境界に影響すべきではない。例えば、`smin_value==1`であるレジスタは、`s32_min_value`も1に等しいことを意味しない。`smax_value`は32ビットのサブレジスタが保持できるよりも大きいかもしれないからである。）

そしてその差分は、隣接する2つのブロックのうちちょうど1つだけを修正している。

```diff
-	if (__reg64_bound_s32(reg->smin_value))
+	if (__reg64_bound_s32(reg->smin_value) && __reg64_bound_s32(reg->smax_value)) {
 		reg->s32_min_value = (s32)reg->smin_value;
-	if (__reg64_bound_s32(reg->smax_value))
 		reg->s32_max_value = (s32)reg->smax_value;
+	}
 	if (__reg64_bound_u32(reg->umin_value))
 		reg->u32_min_value = (u32)reg->umin_value;
 	if (__reg64_bound_u32(reg->umax_value))
 		reg->u32_max_value = (u32)reg->umax_value;
```

符号なしのブロックは差分のコンテキストのまさにそこ、変更箇所のすぐ下に同一の構造で存在している。そしてそれは手つかずのまま残された。報告されたのが符号付きの変種だったので、修正されたのも符号付きの変種だったのである。

両方のブロックは同じ親コミット`3f50f132d840`にさかのぼる。これはv5.7-rc1で明示的なALU32トラッキングを導入したコミットである。このコミットは、この修正が書かれた18日後に公開された[CVE-2021-3490](alu32-bitwise-bounds.md)の`Fixes:`の対象でもある。レジスタの値について2つ目の並行した表現を追加したことで、「2つを同期させておく」という義務の一群がまるごと生まれ、バグは一つずつやって来た。

## 解決 {#resolution}

[`10bf4e83167c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=10bf4e83167cc68595b85fd73bb91e8f2c086e36)（"bpf: Fix propagation of 32 bit unsigned bounds from 64 bit bounds"、Daniel Borkmann）は、符号なしのブロックに同じ形の修正を適用する。

```diff
-	if (__reg64_bound_u32(reg->umin_value))
+	if (__reg64_bound_u32(reg->umin_value) && __reg64_bound_u32(reg->umax_value)) {
 		reg->u32_min_value = (u32)reg->umin_value;
-	if (__reg64_bound_u32(reg->umax_value))
 		reg->u32_max_value = (u32)reg->umax_value;
+	}
```

このコミットは注目に値することをもう一つ行っている。`__reg64_bound_u32()`を`if/return true/return false`の形から単純な`return a > U32_MIN && a < U32_MAX;`に書き換え、`b02709587ea3`がすでに`__reg64_bound_s32()`のスタイルを変更していたのに合わせたのである。コミット自身の言葉では: "Also, align `__reg64_bound_u32()` similarly to `__reg64_bound_s32()` as done in `b02709587ea3` to make them uniform again."（訳: また、`b02709587ea3`で行われたのと同様に`__reg64_bound_u32()`を`__reg64_bound_s32()`に揃え、再び統一する。）これは、見た目の修正も含めて、以前の修正がやり残した仕事を明示的に完了させる修正である。

このパッチには`Reviewed-by: John Fastabend`と`Acked-by: Alexei Starovoitov`が付いているが、公開投稿への`Link:`トレーラーはない。これは公開前に非公開で処理された報告であることと整合する。コミットもNVDの参照リストもメーリングリストへの投稿を指していないため、公開記録はCVEと修正自体である。

差分の1行が、どの文章よりも挙動の変化をよく記録している。verifierが特定のメッセージでプログラムを拒否することをアサートしていたBPFセルフテストが更新された。

```diff
-	.errstr = "invalid access to map value, value_size=48 off=44 size=8",
+	.errstr = "R0 unbounded memory access",
```

修正前、verifierはその特定のオフセットについて文句を言えるほど正確にオフセットを知っていると信じていた。修正後、verifierはその値をまったく知らないことを正しく認める。

同じ境界伝播のロジックは、6か月後にもう一度、公の場に戻ってきた。2021年11月、Starovoitovは`Fixes: 10bf4e83167c`、つまりこのページの修正を名指しするさらなる修正を[投稿した](https://lore.kernel.org/all/20211101222153.78759-1-alexei.starovoitov@gmail.com/)。これはセキュリティバグではなかった: "Before commit `10bf4e83167c` the umin was incorrectly ignored. The commit `10bf4e83167c` fixed the correctness issue, but pessimized propagation of 64-bit min max into 32-bit min max and corresponding var_off."（訳: コミット`10bf4e83167c`以前は、uminが誤って無視されていた。コミット`10bf4e83167c`は正しさの問題を修正したが、64ビットのmin/maxから32ビットのmin/maxおよび対応するvar_offへの伝播を悲観的にしてしまった。）付随するパッチが*符号付き*の側で同等の変更を行った。そちら側の不正確さはこの修正ではなく、その1年前の`3f50f132d840`自体にまでさかのぼるものだった。そのコミットメッセージは、自身の`Fixes:`タグが単なるヒントに過ぎないことを明言していた: "The Fixes tag is a hint. There is no security bug here. The verifier was too conservative."（訳: Fixesタグはヒントである。ここにセキュリティバグはない。verifierが保守的すぎたのである。）セキュリティ修正が出荷され、それが犠牲にした精度を見直す時間ができてから、同じシリーズが両方の側を一度に修復した。

## 教訓 {#what-it-taught-us}

**バグ報告が1つの変種を名指ししているとき、修正の範囲は同じブロック内のすべての兄弟である。** 符号付きと符号なしの伝播パスは隣接し、同一の構造を持ち、同じコミットから派生していた。一方を修正して出荷するのは、バグ報告に対する応答としては擁護できるが、バグの*クラス*に対する応答としては擁護できない。2つの側の間には、4か月半と1つのCVEが横たわっていた。

**同じ事実の冗長な表現は、どのコンパイラもチェックしない同期の義務を生み出す。** `3f50f132d840`がすべてのレジスタに2組目の境界を与えたのには正当な理由があった。Cで書かれた実際のBPFプログラムは32ビット演算を行い、すべてを64ビットの境界に押し込めるとverifierが正しいプログラムを拒否してしまったのである。その代償は、レビュアーの頭の中にしか存在しない一連の不変条件（「32ビットの境界は64ビットの境界から導かれなければならず、その逆も同様」）だった。

**片側だけの範囲テストは範囲テストではない。** `if (endpoint_fits) narrow(endpoint)`は、一方の端点を狭めることが区間全体について何かを主張することになる場合には常に誤りである。正しい述語は端点についてではなく区間についてのものであり、それこそが修正の`&&`が表現しているものである。

!!! warning "注意すべきパターン"
    範囲、エクステント、あるいは開始/終了のペアを共同で定義する、対になった`if (cond_a) x = ...;` / `if (cond_b) y = ...;`ブロックをgrepすること。一方の端だけを狭めることで実際の値を除外する範囲が生じうるなら、2つのテストは1つの条件にまとめるべきである。そして`Fixes:`タグ付きのパッチをレビューするときは、パッチが触れていない構造的に同一のブロックがないか、周囲のハンクのコンテキストを読むこと。このケースでは、脆弱なコードは修正自身の差分の中で、修正箇所のすぐ下の後続コンテキストに変更されないまま見えていた。

## 関連項目 {#see-also}

- [BPF Verifier](../bpf-verifier.md) — レジスタ境界の追跡と、verifierがスカラー値についてどう推論するか
- [ALU32 Bitwise Bounds Tracking](alu32-bitwise-bounds.md) — 同じ導入コミットに対する2021年のもう一つのCVEで、同じ研究者が報告したもの
- [The jmp32 Bounds Regression](jmp32-bounds.md) — その余波がそもそも`3f50f132d840`を生み出すことになった2020年のCVE
- [Unprivileged BPF Off by Default](unprivileged-bpf-off.md) — 一連のverifier CVEに対するハードニングの対応

## 外部参考資料 {#external-references}

- [NVD: CVE-2021-31440](https://nvd.nist.gov/vuln/detail/CVE-2021-31440) — CVSS 7.0 HIGH（NVD）/ 8.8 HIGH（ZDI）、2021年5月21日公開
- [git.kernel.org: 10bf4e83167c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=10bf4e83167cc68595b85fd73bb91e8f2c086e36) — "bpf: Fix propagation of 32 bit unsigned bounds from 64 bit bounds"、修正コミット
- [git.kernel.org: b02709587ea3](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b02709587ea3d699a608568ee8157d8db4fd8cae) — "bpf: Fix propagation of 32-bit signed bounds from 64-bit bounds."（2020年12月）、同じバグの符号付きの側で、先に修正されたもの
- [git.kernel.org: 3f50f132d840](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3f50f132d8400e129fc9eb68b5020167ef80a244) — "bpf: Verifier, do explicit ALU32 bounds tracking"（v5.7-rc1）、2021年の両CVEが名指しする導入コミット
- [lore.kernel.org: bpf: Fix propagation of bounds from 64-bit min/max into 32-bit and var_off](https://lore.kernel.org/all/20211101222153.78759-1-alexei.starovoitov@gmail.com/) — `Fixes: 10bf4e83167c`を持つ2021年11月の後続パッチで、このページの修正が犠牲にした精度を回復した。その符号付き側の付随パッチは、セキュリティバグを修正したものではないことを明言している
