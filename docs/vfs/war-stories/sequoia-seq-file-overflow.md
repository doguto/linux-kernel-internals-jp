# Sequoia：seq_file のサイズ切り詰めによるオーバーフロー {#sequoia-the-seq_file-size-truncation-overflow}

> CVE-2021-33909 — `dentry_path()` が書かれた日から黙って32ビットの `int` に縮小されていた `size_t` のバッファサイズは、無関係な割り当て失敗のバグに対する2014年の修正によって、それを起動できるほど大きなバッファを育てられるようになるまで、到達不能のままだった — そしてその後さらに7年間、誰かが探しに行くまで実質的に到達不能のまま眠っていた。それは「非常に長いディレクトリパスを作る」ことを、攻撃者が選んだ正確な範囲外書き込みに変えた

公開日
:   2021年7月20日（Red Hat、カーネルセキュリティリスト、linux-distros と協調）

報告者
:   Qualys Research Team

CVSS
:   7.8 HIGH（`CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`）

バグの混入時期
:   3.16（2014年7月に vmalloc フォールバックがマージされた。3.16 リリース自体は8月）。5.13.3 まで悪用可能

修正
:   commit `8cae8cd89f05`。5.13.4 の stable ポイントリリースに優先的に取り込まれ、Linux 5.14 の開発サイクルの一部として Linus のメインラインツリーに入った

エクスプロイトツール
:   あり — Qualys は動作するローカル root エクスプロイトを作成して非公開でデモした。クラッシュさせる PoC はアドバイザリとともに公開された

実際の悪用
:   確認された事例はない（CISA KEV には掲載されていない）

*[事件簿：VFS のバグとリグレッション](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

`single_open()` とその仲間は、`seq_file` に、出力全体を1つのバッファで生成する機会をちょうど1回だけ与える — 段階的な分割はない。出力が収まらないと、`fs/seq_file.c` はバッファを2倍にして再試行する。元々これは `kmalloc()` を意味しており、メモリ断片化のもとでは、満たしにくい大きな割り当ては単純に失敗していた — これは実際の問題だった。[`058504edd026`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=058504edd02667eef8fac9be27ab3ea74332e9b4)（"fs/seq_file: fallback to vmalloc allocation"、Heiko Carstens、2014年7月）は、断片化したシステムで実際に起きていた `/proc/stat` の読み出し失敗を、フォールバックを追加することで修正した。`kmalloc()` が失敗し、要求が1 page より大きければ `vmalloc()` にフォールバックする。`vmalloc()` は非連続な page をつなぎ合わせることで、はるかに大きな要求を満たせる。

```c
static void *seq_buf_alloc(unsigned long size)
{
	void *buf;

	buf = kmalloc(size, GFP_KERNEL | __GFP_NOWARN);
	if (!buf && size > PAGE_SIZE)
		buf = vmalloc(size);
	return buf;
}
```

この1つの変更こそが、数ギガバイトの `seq_file` バッファの割り当てをそもそも*可能*にしたものである。倍増のロジックの他の部分は、それを考慮して変更されることはなかった。

## きっかけ {#the-trigger}

再試行のたびに倍増するバッファサイズ `m->size` は `size_t` であり、現代のどのカーネルでも64ビットである。`/proc/self/mountinfo` の読み出し（`show_mountinfo()` → `seq_dentry()`）は、そのサイズをそのまま `dentry_path()` に渡す。

```c
char *dentry_path(struct dentry *dentry, char *buf, int buflen)
```

`buflen` は普通の32ビット符号付き `int` である。`m->size` がちょうど 2 GiB に達すると、呼び出し箇所での暗黙の縮小変換は `INT_MIN` — `-2147483648` — を生む。エラーでもクランプでもなく、黙って再解釈された負の数である。

Qualys のアドバイザリは残りを正確にたどっている。`dentry_path()` は、unlink された dentry に対して `p = buf + buflen` を計算する — `buflen` が今や `INT_MIN` なので、`p` は vmalloc されたバッファの先頭のおよそ 2 GiB *手前*を指す — そして `prepend(&p, &buflen, "//deleted", 10)` を呼んでそのリテラルの接尾辞を書き込む。`prepend()` は（まだ負の、演算後には巨大な正の値として解釈される）`buflen` とポインタをデクリメントし、書き込む。

## 観測された挙動 {#observed-behavior}

書き込みは小さい — 10バイト、リテラル文字列 `"//deleted"` — が、境界チェックがまったくないまま、攻撃者が選んだオフセットに着地する。Qualys 自身の言葉では、"exactly -2GB-10B below the beginning of a vmalloc()ated kernel buffer"（訳: vmalloc() で割り当てられたカーネルバッファの先頭からちょうど 2GB+10B 手前）である。2 GiB のしきい値に到達するには、現実的だが達成可能なセットアップが必要である。パスの全長が 1 GiB を超える深くネストしたディレクトリツリーを作り、それを非特権のユーザー namespace 内で bind マウントし、そのマウントの `mountinfo` を何かがまだ読んでいる間に、dentry が unlink されるように（`"//deleted"` 接尾辞のパスを起動するために）それを `rmdir()` する。`seq_file` 自身のパス要素のエスケープ処理は `\` をそれぞれ4バイトの `\134` シーケンスに変えるので、ディレクトリ名をバックスラッシュで埋める攻撃者は4倍の長さの増幅を得る — Qualys は、実際に必要なディレクトリ数が、`NAME_MAX` 256バイトでの単純な計算が示す400万ではなく、およそ100万個のネストしたディレクトリになることを見出した — これは約 5 GB のメモリと100万個の inode で到達可能である。

その制御された1回の10バイト書き込みから、Qualys は完全なローカル root エクスプロイトを構築した。`userfaultfd()` または FUSE を使って、BPF 検証の途中のスレッドを、検証器が小さな eBPF プログラムを承認した後かつ JIT コンパイルの前で停止させ、範囲外書き込みのタイミングを使ってカーネルの状態を破壊し、検証されたものとは異なるバイトコードに差し替える — 任意のカーネル読み書きを得て、次に `modprobe_path[]` を上書きして root 相当のコード実行を得る。彼らは、この完全なチェーンが Ubuntu 20.04、20.10、21.04、Debian 11、Fedora 34 Workstation のデフォルトインストールで無修正のまま動作することを確認し、他のディストリビューションについては "certainly vulnerable, and probably exploitable"（訳: 確実に脆弱で、おそらく悪用可能）と評価した。

## なぜ起きたか {#why-it-happened}

脆弱な縮小変換 — 64ビットのサイズが32ビット符号付きのパラメータに流れ込む — は、`dentry_path()` のシグネチャが書かれた瞬間から存在していた。単に起動できなかっただけである。`vmalloc()` のサポートが存在する前は、`m->size` を数キロバイトより先に育てられるものが何もなかったからである。2014年に vmalloc フォールバックを追加したことは、実在する割り当て失敗のバグに対する、正しく範囲を絞った修正だった — その作者には、`seq_file` のバッファサイズのすべての下流の利用者について `int` 幅の仮定を監査する理由はなかった。そして切り詰めは、誰かがまさにこのクラスのバグをツリー全体で特に探しに行くまで、7年間実質的に到達不能なまま眠っていた。

## 解決 {#resolution}

Qualys の報告を受けて Al Viro が提案し、Eric Sandeen が書いた修正は3行である。

```c
static void *seq_buf_alloc(unsigned long size)
{
	if (unlikely(size > MAX_RW_COUNT))
		return NULL;

	return kvmalloc(size, GFP_KERNEL_ACCOUNT);
}
```

`MAX_RW_COUNT` は `INT_MAX` を page 境界に切り捨てた値 — 2 GiB をわずかに下回る — であり、まさにこのクラスの int オーバーフローの危険に備えて、通常の read/write のサイズを制限するためにすでに使われている、カーネル全体の既存の定数である。修正は `dentry_path()` のシグネチャには触れず、縮小変換そのものも直さない。単に、それを起動できるほど大きく `seq_file` のバッファが育つことを拒否するだけである。LWN の記事はこれを "something of a band-aid to simply reject seq_buf allocations that get 'too large,'"（訳: 「大きすぎる」seq_buf の割り当てを単に拒否するのは、いささか応急処置的である）と評し、"how many other size_t-to-int problems of that sort still linger in the kernel."（訳: その種の size_t から int への問題が、カーネルにほかにいくつ残っているのか）と問いかけた。

## 得られた教訓 {#what-it-taught-us}

**正しく、範囲を適切に絞った修正が、何年も後に眠っていたバグクラスをひっそりと再び開くことがある**。その修正が、無関係な潜在バグ（`dentry_path()` における `int` の切り詰め）をたまたま覆い隠していた制限（ここでは、巨大な要求に対する `kmalloc()` の確実な失敗）を取り除く場合である。2014年のパッチは、それがやろうとしたことについて間違っていたことは一度もない。

**`size_t` から `int` への縮小は、まさに静かであるがゆえに、繰り返し現れる有名なバグクラスである** — デフォルトではコンパイラの警告もなく、変換箇所でのクラッシュもなく、呼び出し側が 2^31 を超えて押し上げることに成功して初めて意味のある誤りとなる値があるだけである。攻撃者が、最終的に32ビット符号付きの境界を越えるサイズに影響を与えられるコードパスは、どれもまさにこの精査に値する。

!!! warning "注意すべきパターン"
    呼び出し側が影響を与えられ、際限なく増える値が渡されるあらゆる場所で、（`size_t`、`ssize_t`、`unsigned long` ではなく）`int` として宣言されたサイズ/長さのパラメータを grep すること — バッファの倍増ロジック、累積カウンタ、「ひたすら増え続ける」ものはどれも、`INT_MAX` を越えた途端に、静かな縮小変換を攻撃者が制御する正確な範囲外オフセットに変える、まさにその形をしている。

## 関連項目 {#see-also}

- [VFS の概要](../README.md) — このバグが存在する汎用の `seq_file` インターフェイス
- [/proc を理解する](../../mm/understanding-proc-meminfo.md) — `seq_file` が支える `/proc` の基盤
- [Dirty Pipe](../../security/war-stories/dirty-pipe.md) — 小さく静かな見落としから生じ、何年も休眠していた、もう1つのローカル root バグ

## 外部参考文献 {#external-references}

- [git.kernel.org: 8cae8cd89f05](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8cae8cd89f05f6de223d63e6d15e31c8ba9cf53b) — "seq_file: disallow extremely large seq buffer allocations"、修正
- [git.kernel.org: 058504edd026](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=058504edd02667eef8fac9be27ab3ea74332e9b4) — "fs/seq_file: fallback to vmalloc allocation"、バグを到達可能にしたコミット
- [oss-security: CVE-2021-33909](https://www.openwall.com/lists/oss-security/2021/07/20/1) — Qualys の元のアドバイザリ。悪用の仕組みの全容を含む
- [LWN: The Sequoia seq_file vulnerability](https://lwn.net/Articles/863729/) — Jake Edge、2021年7月21日
- [NVD: CVE-2021-33909](https://nvd.nist.gov/vuln/detail/CVE-2021-33909) — CVE レコード。CVSS 7.8 HIGH、2021年7月20日公開
