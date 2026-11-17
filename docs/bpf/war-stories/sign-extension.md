# `check_alu_op()`における符号拡張 {#sign-extension-in-check_alu_op}

> CVE-2017-16995 — verifierは、マシンがゼロ拡張する32ビット即値を符号拡張していたため、レジスタについてのverifierのモデルとレジスタそのものが最初の命令から食い違っていた

公開
:   2017年12月21日（oss-security）

報告者
:   Jann Horn, Google Project Zero

CVSS
:   7.8 HIGH（`CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`、NVDの一次評価）

混入コミット
:   [`484611357c19`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=484611357c19f9e19ef742ebef4505a07d243cc9)（"bpf: allow access into map value arrays"、v4.9）

修正バージョン
:   [`95a762e2c8c9`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=95a762e2c8c942780948091f8f2a4f32fce1ac6f)、メインライン v4.15-rc5

エクスプロイトツール
:   あり — Exploit-DBに3件のエントリがあり、[Metasploitモジュール](https://www.exploit-db.com/exploits/45058)を含む

実際の悪用
:   確認された事例なし（[CISA KEVに未掲載](https://nvd.nist.gov/vuln/detail/CVE-2017-16995)）

*[事件簿: BPF verifierのバグとCVE](../war-stories.md)の一部。*

## 発生前の状態 {#before-state}

BPF命令セットには2つのクラスのALU操作がある。64ビットのレジスタ全体に作用する`BPF_ALU64`と、下位32ビットに作用し上位をゼロで埋める`BPF_ALU`である。どちらのクラスにも32ビット即値オペランドを取る`MOV`があり、その2つは実行時に異なる振る舞いをする。

- `BPF_ALU64 | BPF_MOV | BPF_K` — 32ビット即値を64ビットに**符号拡張**してロードする。
- `BPF_ALU | BPF_MOV | BPF_K` — 32ビット即値を64ビットに**ゼロ埋め**してロードする。

`insn->imm`は符号付き32ビットのフィールドである。`check_alu_op()`の中で、verifierは結果のレジスタ値を、2つのケースを区別しない1行で記録していた。

```c
/* kernel/bpf/verifier.c、修正前 */
regs[insn->dst_reg].type = SCALAR_VALUE;
__mark_reg_known(regs + insn->dst_reg, insn->imm);
```

`__mark_reg_known()`は`u64`を受け取る。負の`s32`を渡すと符号拡張を伴って昇格される。したがって*両方*の命令クラスについて、verifierは符号拡張された値を記録していた。

## きっかけ {#the-trigger}

`BPF_ALU64`ではそれは正しい。`BPF_ALU`ではそうではない。

負の即値を持つ32ビット`MOV`を書くと、verifierは、上位32ビットが立った大きな64ビットの量であると信じるレジスタを記録する。実行時には、インタプリタとすべてのJITは代わりにゼロ埋めを行うため、レジスタは実際には2³²未満の小さな正の数を保持している。

Jann Hornの修正はその区別を述べるだけで、それ以上は何も述べていない。それ以上は何も必要ないからである。

> Distinguish between `BPF_ALU64|BPF_MOV|BPF_K` (load 32-bit immediate, sign-extended to 64-bit) and `BPF_ALU|BPF_MOV|BPF_K` (load 32-bit immediate, zero-padded to 64-bit); only perform sign extension in the first case.

（訳: `BPF_ALU64|BPF_MOV|BPF_K`（32ビット即値を64ビットに符号拡張してロード）と`BPF_ALU|BPF_MOV|BPF_K`（32ビット即値を64ビットにゼロ埋めしてロード）を区別し、前者の場合にのみ符号拡張を行う。）

verifierの安全性の論拠全体は、レジスタごとの値のモデルがマシンと一致していることに依存している。ここでは、定数をロードする時点で両者が食い違う。それは言語の中で最も単純な命令であり、その後のあらゆる境界推論がその上に構築される命令である。

## 観測された挙動 {#observed-behavior}

NVDの説明は型どおりの要約である — "allows local users to cause a denial of service (memory corruption) or possibly have unspecified other impact by leveraging incorrect sign extension."（訳: 誤った符号拡張を利用することで、ローカルユーザーがサービス拒否（メモリ破壊）を引き起こしたり、その他の不特定の影響を及ぼしたりできる可能性がある。）2017年12月21日のHorn自身の[oss-security投稿](https://www.openwall.com/lists/oss-security/2017/12/21/2)は、このクラスについてもっと率直である: "A few BPF verifier bugs in the Linux kernel, most of which can be used for controlled memory corruption, have been fixed over the last days."（訳: LinuxカーネルのいくつかのBPF verifierのバグが、ここ数日で修正された。そのほとんどは制御されたメモリ破壊に利用できる。）

Hornが報告と同時に公開した概念実証は、その投稿の中で、非カノニカルアドレスへの書き込みを試みるクラッシャーとして説明されており、4.14専用に書かれたものであることが明記されている。これはverifierのモデルが誤っていることの実証であって、武器化されたエクスプロイトではない。

武器化は別のところから、しかも素早くやって来た。

- **[Exploit-DB 44298](https://www.exploit-db.com/exploits/44298)** — "Linux Kernel < 4.4.0-116 (Ubuntu 16.04.4) — Local Privilege Escalation"、Bruce Leidl、2018年3月16日。
- **[Exploit-DB 45010](https://www.exploit-db.com/exploits/45010)** — "Linux Kernel < 4.13.9 (Ubuntu 16.04 / Fedora 27) — Local Privilege Escalation"、rlarabee、2018年7月10日。
- **[Exploit-DB 45058](https://www.exploit-db.com/exploits/45058)** — "Linux — BPF Sign Extension Local Privilege Escalation"、2018年7月19日。Metasploitモジュール（`exploit/linux/local/bpf_sign_extension_priv_esc`）であり、このバグを、フレームワークを動かす誰にとってもコマンド一つでローカルrootが取れるものに変えた。

公開から動作する公開エクスプロイトまで3か月、Metasploitモジュールまで7か月。このシリーズの他のインシデントと同様に、CISAのKEVカタログは実環境での悪用を確認していない。

到達可能性についての注意書きは、修正コミット自体に書かれている。

> Starting with v4.14, this is exploitable by unprivileged users as long as the `unprivileged_bpf_disabled` sysctl isn't set.

（訳: v4.14以降、`unprivileged_bpf_disabled` sysctlが設定されていない限り、これは非特権ユーザーによって悪用可能である。）

このバグはv4.9で[`484611357c19`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=484611357c19f9e19ef742ebef4505a07d243cc9)（"bpf: allow access into map value arrays"、Josef Bacik、2016年9月）によって混入したが、修正コミットによれば、非特権ユーザーによって悪用可能になったのはv4.14以降であり、それがverifierの欠陥を権限昇格に変えた。

## なぜ起きたか {#why-it-happened}

**符号付きのフィールドと符号なしのパラメータ、そしてその間の暗黙の昇格。** `insn->imm`は符号付きで、`__mark_reg_known()`は`u64`を受け取る。この変換は正当なCであり、何の警告もなく、2つの呼び出し元のうちちょうど一方にとってだけ正しい。言語レベルでは何も間違っていないので、警告は出ない。コードは意味することを正確に述べており、その意味することが半分の場合にだけ正しいのである。

**意味の異なる2つの命令クラスが1つのコードパスを共有している。** `check_alu_op()`は`BPF_ALU`と`BPF_ALU64`をまとめて扱い、振る舞いが異なるところで`BPF_CLASS(insn->code)`によって分岐する。ここは振る舞いが異なるのに分岐が欠けていた箇所の一つだった。

**これは8つのうちの1つだった。** Hornの2017年12月のoss-security投稿は、1回の調査で見つかった8つの異なるverifierのバグを列挙する単一のメッセージであり、それぞれに独自の修正コミットがある。このバグと並んで、`BPF_RSH`の誤った符号付き境界、レジスタサイズの切り詰めの誤った追跡（CVE-2017-16996）、インタプリタとJITが32ビット演算を行う一方で32ビットALU操作の検証が64ビットの数値に対して行われていたこと、`check_stack_boundary()`におけるエラーリターンの欠落、スタックポインタに対する厳密なアラインメントチェックの欠落、スカラーがポインタに置き換えられたときの分岐の枝刈り、そしてオフセット演算における一連の整数オーバーフローがあった。符号拡張のバグは孤立したミスではない。敵対的な視点を持つ誰によってもまだ監査されていなかった算術モデルを持つサブシステムからの一つのサンプルであり、それを体系的に行った最初の存在がProject Zeroだったのである。

## 解決 {#resolution}

修正はif/elseである。

```diff
 			regs[insn->dst_reg].type = SCALAR_VALUE;
-			__mark_reg_known(regs + insn->dst_reg, insn->imm);
+			if (BPF_CLASS(insn->code) == BPF_ALU64) {
+				__mark_reg_known(regs + insn->dst_reg,
+						 insn->imm);
+			} else {
+				__mark_reg_known(regs + insn->dst_reg,
+						 (u32)insn->imm);
+			}
```

`(u32)`キャストが`BPF_ALU`に対してゼロ埋めのパスを強制し、verifierのモデルをインタプリタとJITが実際に行うことと一致させる。

`95a762e2c8c9`には`Acked-by: Edward Cree`と、Alexei StarovoitovおよびDaniel BorkmannのSigned-off-byが付いている。Hornのoss-security投稿は、告知の時点で修正が"in the net tree of the Linux kernel [...] but not in Linus' tree yet."（Linuxカーネルのnetツリーには入っているが、Linusのツリーにはまだ入っていない）と記している。パッチはv4.15-rc5でメインラインに到達した。

ここで引用すべき設計上の議論はなく、その理由はコミット自身のリビジョンノートに見て取れる。

```
v3:
 - add CVE number (Ben Hutchings)
```

パッチは少なくとも3回目のリビジョンに達したが、コミットメッセージが残している唯一のリビジョン変更は、Debianが割り当てたCVE識別子の追加である。修正について議論した者はいなかった。議論すべきことが何もなかったからである。命令セットは2つの振る舞いを定義しており、コードはそのうち1つを実装していた。

構造的に変わったのは、この一連のバグが、verifierの算術が正しさのレビュー対象からセキュリティのレビュー対象へと変わった時点を画しているということである。このシリーズの残りで記録されている一連の境界追跡CVE（[2020年](jmp32-bounds.md)、[2021年](alu32-bitwise-bounds.md)、[再び2021年](alu32-unsigned-bounds.md)）は、すべてその転換の下流にある。

## 教訓 {#what-it-taught-us}

**仕様が2つの振る舞いを定義しているところでは、実装には2つの分岐が必要である。そして型システムはそれを思い出させてはくれない。** BPF命令セットは、`BPF_ALU`がゼロ埋めし`BPF_ALU64`が符号拡張することを明示している。verifierのコードパスは1つだった。このバグは一度言葉にされれば微妙なものではないが、問題の行が正しいコードとまったく同じように読めるため、レビューでは見えない。

**関数境界での符号付きから符号なしへの昇格は、デフォルトが正しい場合でも明示的なキャストに値する。** `__mark_reg_known(reg, insn->imm)`は、`imm`が`s32`であること、パラメータが`u64`であること、そして符号拡張が意図されていることを読み手が知っていることに依存している。意図した変換を書き出すこと（修正が一方の側で`(u32)`を使って行っているように）で、2つのケースを言語規格のレベルではなく呼び出し箇所で区別できるようになる。

**verifierのバグの深刻度は、非特権プログラムがその上に何を構築できるかによって決まる。** この欠陥はv4.9で出荷され、v4.14から権限昇格になった。修正コミットはその境界を説明なしに述べている。欠陥自体は、v4.14でのverifierの値追跡の全面的な書き直しを生き延びた。v4.9は`insn->imm`を直接`u64 max_value`に代入しており、Edward Creeの[`b03c9f9fdc37`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b03c9f9fdc37dab81ea04d5dacdc5995d4c224c2)はそれを、同じ`s32`を`__mark_reg_known()`の`u64`パラメータに渡す呼び出しに置き換えた。別の行がまったく同じ誤りを犯していたのである。これら5つのリリースの間に変わったのは露出であって、バグではなかった。この関係こそ、3年半後に[非特権BPFのデフォルト無効化](unprivileged-bpf-off.md)の変更が最終的に対処することになるものである。

**体系的な敵対的レビューは、バグではなくバグの*クラス*を見つける。** 1人の研究者による1回の調査で、1つの告知に8つのverifierのバグ。個々の修正はそれぞれ数行である。重要だった発見は、verifierの整数演算が、それを積極的に壊そうとする者によって一度も検証されたことがなかったということである。

!!! warning "注意すべきパターン"
    デコーダ、インタプリタ、またはverifierがオペコードのファミリーを共有コードで扱っているすべての箇所を監査すること。オペコードファミリーが変化させる各フィールド（オペランド幅、符号の有無、アドレッシングモード、エンディアン）について、分岐があるか、振る舞いが本当に同一であることの証明があるかを確認すること。危険な形は、多数派のケースでは正しく少数派のケースでは黙って誤っている単一の文である。それはどのレビューでも正しく読めてしまうからである。それとは別に、実行時の値をモデル化することが仕事であるコードでは、`s32`を`u64`の引数に渡すことをレビュー上の警告サインとして扱うこと。

## 関連項目 {#see-also}

- [BPF Verifier](../bpf-verifier.md) — verifierがレジスタ値をどうモデル化するか、そして誤った定数がなぜ下流のすべてを汚染するのか
- [アーキテクチャとプログラムタイプ](../bpf-overview.md) — このバグの`MOV`が属するeBPF命令フォーマットとレジスタモデル
- [The jmp32 Bounds Regression](jmp32-bounds.md) — 2年余り後の同じ故障モード（verifierのモデルが実行時の値から乖離する）
- [Unprivileged BPF Off by Default](unprivileged-bpf-off.md) — このバグの悪用可能性が依存していたsysctl
- [カーネルハードニング](../../security/kernel-hardening.md) — このようなローカル権限昇格が突破しなければならない緩和策の層

## 外部参考資料 {#external-references}

- [NVD: CVE-2017-16995](https://nvd.nist.gov/vuln/detail/CVE-2017-16995) — CVSS 7.8 HIGH、2017年12月27日公開
- [git.kernel.org: 95a762e2c8c9](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=95a762e2c8c942780948091f8f2a4f32fce1ac6f) — "bpf: fix incorrect sign extension in check_alu_op()"、修正コミット（7行追加、1行削除）
- [git.kernel.org: 484611357c19](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=484611357c19f9e19ef742ebef4505a07d243cc9) — "bpf: allow access into map value arrays"（v4.9）、修正の`Fixes:`タグが名指しするコミット
- [oss-security: Linux >=4.9: eBPF memory corruption bugs](https://www.openwall.com/lists/oss-security/2017/12/21/2) — 8つのverifierバグすべてについての、Jann Hornによる2017年12月21日の告知。バグごとの技術的説明を含む
- [Exploit-DB 44298](https://www.exploit-db.com/exploits/44298)、[45010](https://www.exploit-db.com/exploits/45010)、[45058](https://www.exploit-db.com/exploits/45058) — 登録済みの3つのローカルrootエクスプロイト。45058はMetasploitモジュール`exploit/linux/local/bpf_sign_extension_priv_esc`である
