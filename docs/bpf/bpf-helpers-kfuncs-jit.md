# ヘルパー、kfunc、JIT {#helpers-kfuncs-and-the-jit}

> BPFプログラムはどのようにカーネルを呼び出し、ロードされた後に実際にどう実行されるのか

[アーキテクチャとプログラムタイプ](bpf-overview.md)では、ヘルパーを呼び出し可能な関数の平坦なリストとして扱った。このページではもう一段深く掘り下げる。そのリストの背後にある実際の呼び出し規約の仕組み、それと並行して成長してきたより新しいkfuncの仕組み、そして検証済みのプログラムがどのようにしてCPUが実際に実行するコードになるのかを扱う。

## ヘルパー: 固定された、番号付きのABI {#helpers-a-fixed-numbered-abi}

BPFプログラムは任意のカーネルシンボルを呼び出せない — コンパイラにはカーネルコードに対してリンクする手段がなく、Verifierには任意の関数本体について推論する手段がない。その代わりに、カーネルは固定されたヘルパー関数の集合を公開しており、各ヘルパーはコンパイル時に命令ストリームに埋め込まれる安定した番号で識別される。

```c
/* include/uapi/linux/bpf.h — ヘルパーIDのリストは、ヘルパーごとに
 * 1回ずつ呼び出される単一のマクロから組み立てられる。例: */
FN(map_lookup_elem, 1, ##ctx)
FN(map_update_elem, 2, ##ctx)
FN(map_delete_elem, 3, ##ctx)
/* ... */
```

`bpf_map_lookup_elem()`はヘルパー1である — 最も古く、最もよく使われるヘルパーであり、番号付きリストという慣習に名前が必要になる以前から存在する。コンパイルされたプログラム内の`BPF_CALL`命令はシンボルを参照しない。この番号を即値オペランドとして持つのである。

### 5レジスタの呼び出し規約 {#the-five-register-calling-convention}

すべてのヘルパーは — 実際のシグネチャにかかわらず — ちょうど5つの64ビットレジスタ（`r1`〜`r5`）を通して呼び出される。`BPF_CALL`命令が渡せるのがそれだけだからである。ヘルパーの実装は通常の型付きC関数のように見えるが、本当のレジスタ型のエントリポイントを生成するマクロでラップされている。

```c
/* include/linux/filter.h */
#define BPF_CALL_x(x, attr, name, ...)					\
	static __always_inline						\
	u64 ____##name(__BPF_MAP(x, __BPF_DECL_ARGS, __BPF_V, __VA_ARGS__)); \
	typedef u64 (*btf_##name)(__BPF_MAP(x, __BPF_DECL_ARGS, __BPF_V, __VA_ARGS__)); \
	attr u64 name(__BPF_REG(x, __BPF_DECL_REGS, __BPF_N, __VA_ARGS__));	\
	attr u64 name(__BPF_REG(x, __BPF_DECL_REGS, __BPF_N, __VA_ARGS__))	\
	{								\
		return ((btf_##name)____##name)(__BPF_MAP(x,__BPF_CAST,__BPF_N,__VA_ARGS__));\
	}								\
	static __always_inline						\
	u64 ____##name(__BPF_MAP(x, __BPF_DECL_ARGS, __BPF_V, __VA_ARGS__))

#define BPF_CALL_2(name, ...)	BPF_CALL_x(2, __NOATTR, name, __VA_ARGS__)
```

`BPF_CALL_2(bpf_map_lookup_elem, struct bpf_map *, map, void *, key)`は、2つの型付き引数を取る本物の関数（`____bpf_map_lookup_elem`）と、5つの`u64`を受け取り、最初の2つを`struct bpf_map *`/`void *`にキャストして呼び出す、レジスタ型の2つ目のラッパー（`bpf_map_lookup_elem`）に展開される。VerifierとJITが目にするのはレジスタ型のラッパーだけであり、ヘルパーのロジックを実際に書くのは型付きの内側の関数である。

### ヘルパー呼び出しの型チェック {#type-checking-a-helper-call}

Verifierは、各ヘルパーの引数と戻り値が実際に何を意味するのかを知る必要がある — `u64`のレジスタは、mapへのポインタかもしれないし、生の整数かもしれないし、プログラムが触れることを許されないポインタかもしれない。そのメタデータは、ヘルパーごとに1つある別の構造体に格納されている。

```c
/* include/linux/bpf.h */
struct bpf_func_proto {
	u64 (*func)(u64 r1, u64 r2, u64 r3, u64 r4, u64 r5);
	bool gpl_only;
	bool pkt_access;
	bool might_sleep;
	/* ... allow_fastcallはここでは省略 ... */
	enum bpf_return_type ret_type;
	union {
		struct {
			enum bpf_arg_type arg1_type;
			enum bpf_arg_type arg2_type;
			enum bpf_arg_type arg3_type;
			enum bpf_arg_type arg4_type;
			enum bpf_arg_type arg5_type;
		};
		enum bpf_arg_type arg_type[MAX_BPF_FUNC_ARGS];
	};
	/* ... ポインタ/サイズ引数ペアのためのBTF IDとサイズのメタデータ ... */
};
```

各プログラムタイプは、ヘルパーの数値IDをその`bpf_func_proto`に対応付ける`get_func_proto()`コールバックを持つ — これは、たとえばXDPプログラムがトレーシングでのみ有効なヘルパーを呼べないことをVerifierが強制する方法でもある。番号付けは恒久的である — ヘルパーのIDはユーザー空間に面したBPF ABIの一部である — ため、あるカーネル向けにコンパイルされたプログラムは、そのIDを認識し続けるどの新しいカーネルでも動き続ける。

とはいえ、新しいヘルパーはもう実質的に追加されない。`include/uapi/linux/bpf.h`自身が、ヘルパーリストの最後のエントリの直後でそう述べている（Linux **6.15**で追加されたコメント。タグ間のdiffで確認済み — v6.14には存在せず、v6.15には存在する）。

```c
/* This helper list is effectively frozen. If you are trying to
 * add a new helper, you should add a kfunc instead which has
 * less stability guarantees. See Documentation/bpf/kfuncs.rst
 */
```

（訳: このヘルパーリストは事実上凍結されている。新しいヘルパーを追加しようとしているなら、代わりに、安定性の保証がより弱いkfuncを追加すべきである。Documentation/bpf/kfuncs.rstを参照のこと。）

これは最近の方針転換というより、何年にもわたる慣行にコメントが追いついたものである。Linux 7.2の時点でもヘルパー211（`cgrp_storage_delete`）がリストの最後であり、これは6.19、6.12、6.6のときと同じである。その期間に追加された、BPFから呼び出し可能な新しいカーネル関数はすべて、代わりにkfuncとして入った。

## kfunc: 現在、新しいBPF機能が実際に追加される方法 {#kfuncs-how-new-bpf-functionality-actually-gets-added-now}

ヘルパーリストはカーネル全体で単一のenumである。あらゆるサブシステムのあらゆるヘルパーが同じリストに追加され、同じ恒久的なABI保証を受け、たとえ1つのサブシステムにしか役立たないものであってもBPFメンテナのレビューを必要とする。これでは、個々のサブシステム（あるいは個々のカーネルモジュール）が、自分たち専用の狭い、場合によっては不安定な関数をBPFに公開できるようにするところまでスケールしない — そして、上の凍結コメントにある通り、新しい機能はもはやこの方法ではまったく追加されない。kfuncはヘルパーABIの脇にある裏口ではない。BPFプログラムが呼び出せるものを拡張するための、唯一サポートされた経路なのである。

この仕組み自体は凍結より何年も前から存在する。そもそもBPFからカーネル関数を呼ぶこと — `BPF_PSEUDO_KFUNC_CALL`という命令エンコーディングのビット — はLinux **5.13**で導入された（`include/uapi/linux/bpf.h`のタグ間diffで確認済み: v5.12には存在せず、v5.13には存在する）。以下に示す登録API — `register_btf_kfunc_id_set()`、`struct btf_kfunc_id_set` — はそれより後のLinux **5.18**で登場した（`include/linux/btf.h`のタグ間diffで確認済み: v5.17には存在せず、v5.18には存在する）。これは、v5.16とv5.17の`include/linux/btf.h`に存在した、以前のより扱いにくいリスト単位のAPIである`struct kfunc_btf_id_set` / `register_kfunc_btf_id_set()`を置き換えたものである。

Linux 7.2時点の`kernel/bpf/helpers.c`にある実際のkfuncセットを示す（`_impl`の付かない`bpf_obj_new`/`bpf_obj_drop`のエントリと`KF_IMPLICIT_ARGS`フラグは最近の追加である — v7.0には存在せず、v7.1には存在することを確認済み。それ以前は`_impl`のエントリしかなく、呼び出し側はそれらを直接使っていた）。

```c
/* kernel/bpf/helpers.c */
BTF_KFUNCS_START(generic_btf_ids)
BTF_ID_FLAGS(func, bpf_obj_new, KF_ACQUIRE | KF_RET_NULL | KF_IMPLICIT_ARGS)
BTF_ID_FLAGS(func, bpf_obj_new_impl, KF_ACQUIRE | KF_RET_NULL)
BTF_ID_FLAGS(func, bpf_obj_drop, KF_RELEASE | KF_IMPLICIT_ARGS)
BTF_ID_FLAGS(func, bpf_obj_drop_impl, KF_RELEASE)
/* ... */
BTF_KFUNCS_END(generic_btf_ids)

static const struct btf_kfunc_id_set generic_kfunc_set = {
	.owner = THIS_MODULE,
	.set   = &generic_btf_ids,
};
```

このセットは、それを見せるべきプログラムタイプごとに登録される。

```c
/* kernel/bpf/helpers.c */
ret = register_btf_kfunc_id_set(BPF_PROG_TYPE_TRACING, &generic_kfunc_set);
ret = ret ?: register_btf_kfunc_id_set(BPF_PROG_TYPE_SCHED_CLS, &generic_kfunc_set);
ret = ret ?: register_btf_kfunc_id_set(BPF_PROG_TYPE_XDP, &generic_kfunc_set);
```

`KF_*`フラグ（`include/linux/btf.h`）は、`bpf_func_proto`による引数型チェックに代わるVerifierの手段であり、kfuncが実際に必要とする言葉で表現されている。`KF_ACQUIRE`は、プログラムが最終的に解放しなければならない参照を返す関数を示す。`KF_RELEASE`はそれを解放する関数を示す。`KF_RET_NULL`は、戻り値を使用前にNULLチェックしなければならないことをVerifierに伝える。ヘルパーの恒久的な数値IDとは異なり、kfuncはロード時にBTFを通じて名前で解決される — そして`Documentation/bpf/kfuncs.rst`は、これには確固たる安定性の約束がまったく伴わないと明言している。"A kfunc will never have any hard stability guarantees. BPF APIs cannot and will not ever hard-block a change in the kernel purely for stability reasons."（訳: kfuncには確固たる安定性の保証が与えられることは決してない。BPF APIが、純粋に安定性の理由だけでカーネルの変更を厳格に阻むことはできないし、今後もそうすることはない。）ある特定のkfuncが呼び出し側の知らないうちに実際に変わるかどうかは、メンテナがケースごとに下す判断であり、サブシステムが前もって約束できるものではない — この恒久的な契約がないことが、追加に中央のレビューを必要としないことの代償なのである。

## JIT: バイトコードからネイティブコードへ {#the-jit-from-bytecode-to-native-code}

検証済みのBPFプログラムは固定長命令の列である — 実行しても安全だが、先に実際のマシンコードにコンパイルしない限り、インタプリタが1命令ずつディスパッチしなければならないバイトコードのままである。LinuxはeBPFが存在する以前からBPF用のJITを持っていた。Eric Dumazetによる、classic（ソケットフィルタ）BPF向けのx86-64 JITである。これは2011年4月にLWNでJonathan Corbetが取り上げており（[LWN](https://lwn.net/Articles/437981/)）、`arch/x86/net/bpf_jit_comp.c`のタグ間diffによってLinux **3.0**（2011年7月）で取り込まれたことが確認されている（v2.6.39には存在せず、v3.0には存在する）。2014年のeBPFへの書き直しでは他のすべてとともにこのJITも置き換えられたが、同じファイルと同じ考え方 — 実行のたびにプログラムを解釈し直すのではなく、プログラム全体を一度ネイティブ命令に変換する — は引き継がれた。

Linux 7.2の時点では、各アーキテクチャが1つの関数の背後に独自のJITを提供しているが、その関数は直接呼ばれるわけではない — 先にプログラムを書き換えうるステップでラップされている。このラッパーは最近のものである。ラッパーと、それが呼び出す2引数の`bpf_int_jit_compile(env, prog)`シグネチャはどちらもLinux **7.1**で導入された（`kernel/bpf/core.c`のタグ間diffで確認済み — v7.0およびそれ以前のすべてのリリース、現行のすべてのLTSを含め、呼び出し側は1引数の素の`bpf_int_jit_compile(fp)`であり、定数ブラインディングは各アーキテクチャ自身の`bpf_int_jit_compile()`の中で、`bpf_jit_blind_constants()`を介して別途処理されている）。

```c
/* kernel/bpf/core.c — 定数ブラインディングを扱う実際のラッパー。
 * bpf_jit_harden sysctlで制御される: 無条件に有効なのではなく、JITスプレーを
 * 緩和するためにJIT前に即値を書き換える（Documentation/admin-guide/
 * sysctl/net.rst）。 */
static struct bpf_prog *bpf_prog_jit_compile(struct bpf_verifier_env *env, struct bpf_prog *prog)
{
#ifdef CONFIG_BPF_JIT
	struct bpf_prog *orig_prog;

	if (!bpf_prog_need_blind(prog))
		return bpf_int_jit_compile(env, prog);

	orig_prog = prog;
	prog = bpf_jit_blind_constants(env, prog);
	/*
	 * If blinding was requested and we failed during blinding, we must fall
	 * back to the interpreter.
	 */
	if (IS_ERR(prog))
		goto out_restore;

	prog = bpf_int_jit_compile(env, prog);
	/* ... */
#endif
	return prog;
}
```

`bpf_int_jit_compile(struct bpf_verifier_env *env, struct bpf_prog *prog)`（`include/linux/filter.h`で宣言され、アーキテクチャごとに定義される）が本当のコンパイラである。サポートされているすべてのアーキテクチャが、この同じ関数の独自版を定義している。`arch/x86/net/bpf_jit_comp.c`だけでも4,000行を超え、固定長のBPFオペコードそれぞれを対応するx86-64命令列に変換している。これには、JITされたBPF関数を他の関数と同じように呼び出せるようにするためのレジスタ割り当てと呼び出し規約の処理も含まれる。これをまったく実装していないアーキテクチャは、`kernel/bpf/core.c`にある`__weak`スタブにフォールバックする。このスタブはプログラムを変更せずに返すだけであり、そこで実際に動くのはインタプリタのパスである。JITを試みるかどうか自体も、次に述べる常時オン/オフの判断とは別に、`net.core.bpf_jit_enable` sysctl（`fp->jit_requested = ebpf_jit_enabled()`）によって制御される。

### JITかインタプリタかの選択 {#choosing-jit-or-interpreter}

コンパイルは無条件に行われるわけではない — 検証済みプログラムの最終処理の一部として行われ、失敗した場合のフォールバックが文書化されている。

```c
/* kernel/bpf/core.c（簡略化） */
struct bpf_prog *__bpf_prog_select_runtime(struct bpf_verifier_env *env,
					    struct bpf_prog *fp, int *err)
{
	bool jit_needed = fp->jit_required;

	if (!bpf_prog_select_interpreter(fp))
		jit_needed = true;

	/*
	 * eBPF JITs can rewrite the program in case constant
	 * blinding is active. However, in case of error during
	 * blinding, bpf_int_jit_compile() must always return a
	 * valid program, which in this case would simply not
	 * be JITed, but falls back to the interpreter.
	 */
	fp = bpf_prog_jit_compile(env, fp);
	if (!fp->jited && jit_needed) {
		*err = -ENOTSUPP;
		return fp;
	}
	/* ... */
	return fp;
}
```

このコメントは実在する安全性の性質を述べている。アーキテクチャがJITコードを生成できない（あるいは定数ブラインディングによって生成しないことを選ぶ）場合でも、インタプリタは依然として完全に有効なフォールバックである — 通常は。`jit_required`は最初`IS_ENABLED(CONFIG_BPF_JIT_ALWAYS_ON)`から設定されるが、Verifier（`kernel/bpf/verifier.c`）も、インタプリタではそもそも実行できない特定の形のプログラムについてはこれを強制的にオンにする。kfunc呼び出しを行うすべてのプログラム — 上述のkfuncの仕組みにはインタプリタ側の実装がない — に加え、BPF arena mapや`BPF_MAP_TYPE_INSN_ARRAY` mapタイプを使うプログラムである。`CONFIG_BPF_JIT_ALWAYS_ON`はこれらの中で最も広範なものであり、カーネルからインタプリタを完全に取り除く。その目的は特に「インタプリタによるBPF命令の投機実行を避けるため」("to avoid speculative execution of BPF instructions by the interpreter")であり、Spectre系の緩和策である。JITされていないBPFプログラムは、実際の（投機実行されうる）カーネルコードによって1命令ずつたどられるからである。kfunc/arena/insn-arrayの場合はより限定的で、それらの特定の機能を使うプログラムだけがフォールバックを失う。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/filter.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/filter.h) — `BPF_CALL_x`マクロ群と`bpf_int_jit_compile()`の宣言
- [include/linux/bpf.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/bpf.h) — `struct bpf_func_proto`、Verifierのヘルパーごとの型チェック用メタデータ
- [include/uapi/linux/bpf.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/bpf.h) — `___BPF_FUNC_MAPPER`のヘルパー名と番号の対応リストから生成される`enum bpf_func_id`
- [include/linux/btf.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/btf.h) — `register_btf_kfunc_id_set()`、`struct btf_kfunc_id_set`、および`KF_ACQUIRE`/`KF_RELEASE`/`KF_RET_NULL`フラグ
- [include/linux/btf_ids.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/btf_ids.h) — `BTF_KFUNCS_START`/`BTF_ID_FLAGS`/`BTF_KFUNCS_END`
- [kernel/bpf/helpers.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/bpf/helpers.c) — `generic_btf_ids`、`bpf_obj_new`/`bpf_obj_drop`などを含む実際のkfuncセット
- [kernel/bpf/core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/bpf/core.c) — `__bpf_prog_select_runtime()`、JITかインタプリタかの判断と定数ブラインディングのフォールバック
- [arch/x86/net/bpf_jit_comp.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/net/bpf_jit_comp.c) — x86-64 JIT

### LWNの記事 {#lwn-articles}

- [A JIT for packet filters](https://lwn.net/Articles/437981/) — Jonathan Corbet、2011年4月。Linux 3.0へのマージに先立ち、Eric Dumazetによるx86-64向けclassic BPF JITを取り上げた記事

### 関連ページ {#related-pages}

- [BPFのアーキテクチャとプログラムタイプ](bpf-overview.md) — プログラムから使うヘルパーのリストと、`bpf()`システムコールのロードパス
- [BPF Verifier](bpf-verifier.md) — `bpf_func_proto`とBTF型情報がどのように安全性証明に使われるか
- [BTFとCO-RE](btf-core.md) — kfuncの登録が土台とするBTF型エンコーディングの仕組み
