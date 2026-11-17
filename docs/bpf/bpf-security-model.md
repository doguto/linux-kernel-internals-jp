# BPFセキュリティモデル {#bpf-security-model}

> ケーパビリティ、非特権BPFのノブ、そしてセキュリティモジュールとしてのBPF

## 2つの異なるセキュリティ上の問い {#two-different-security-questions}

[Verifier](bpf-verifier.md)が答えるのは1つの問い、すなわち*あるプログラムのロードが許可されたとして、それは安全に振る舞うか?* である。このページが答えるのは別の問い、すなわち*そもそも誰がプログラムをロードしてよいのか、そしてBPFプログラム自身に何を強制させることを信頼できるのか?* である。両者は関連している — 健全なVerifierを持つことの要点はまさに、2つ目の問いへの答えを「root以外の人も」にできることにある — が、別々の仕組みである。そして[BPFの事件簿](war-stories.md)のページは、Verifierだけで十分に強固だという確信をカーネルが徐々に失っていった経緯を記録している。以下で述べるケーパビリティとsysctlの層がそもそも存在するのは、そのためである。

## ケーパビリティモデル {#the-capability-model}

BPFプログラムのロードもBPF mapの作成も、同じ中心的なチェックである`bpf_token_capable()`を通る。これは`kernel/bpf/token.c`で定義されている（`include/linux/bpf.h`では宣言のみ）。

```c
static bool bpf_ns_capable(struct user_namespace *ns, int cap)
{
	return ns_capable(ns, cap) || (cap != CAP_SYS_ADMIN && ns_capable(ns, CAP_SYS_ADMIN));
}

bool bpf_token_capable(const struct bpf_token *token, int cap)
{
	struct user_namespace *userns;

	/* BPFトークンはns_capable()レベルのケーパビリティを許可する */
	userns = token ? token->userns : &init_user_ns;
	if (!bpf_ns_capable(userns, cap))
		return false;
	if (token && security_bpf_token_capable(token, cap) < 0)
		return false;
	return true;
}
```

`CAP_SYS_ADMIN`は、BPFに固有のどのケーパビリティチェックに対しても常にフォールバックとして受け入れられる — `CAP_SYS_ADMIN`を持つプロセスは、さらに`CAP_BPF`を必要とはしない。`CAP_BPF`自体はより限定的な専用のケーパビリティであり、Linux 5.8で追加された（v5.7タグの`include/uapi/linux/capability.h`には存在せず、v5.8には存在することを確認済み）。BPFプログラムのロードとmapの作成に、はるかに広範な`CAP_SYS_ADMIN`を要求しないようにするためである。`CAP_PERFMON`は、よりリスクの高いVerifierの緩和 — `bpf_allow_ptr_leaks()`、`bpf_allow_uninit_stack()`、そしてSpectre v1/v4緩和策の回避（`bpf_bypass_spec_v1()`/`bpf_bypass_spec_v4()`）— について別途チェックされる。これらはすべて`include/linux/bpf.h`で`bpf_token_capable()`のすぐ隣に宣言されている。トークンがない場合、チェックは`init_user_ns`にフォールバックし、通常の`capable()`チェックと同じように振る舞う。トークンがある場合は、トークン自身のuser namespaceに対して評価され、さらに`security_bpf_token_capable()`によるLSMの拒否権が加わる。

### BPFトークン: ケーパビリティをnamespaceに委譲する {#bpf-tokens-delegating-capabilities-into-a-namespace}

上の`token`引数は飾りではない。`kernel/bpf/token.c`（v6.8には存在せず、v6.9には存在することを確認済み）を使うと、ホスト上で本物のケーパビリティを持つプロセスが、`delegate_cmds=`、`delegate_maps=`、`delegate_progs=`、`delegate_attachs=`オプション（`kernel/bpf/inode.c`）付きで`bpffs`インスタンスをマウントし、そのマウントのファイルディスクリプタを、別のuser namespaceにいる非特権プロセス — たとえばコンテナ — に渡すことができる。そのプロセスは`BPF_TOKEN_CREATE`を指定して`bpf()`を呼び、そのfdからトークンを発行し、以降の`BPF_MAP_CREATE`/`BPF_PROG_LOAD`呼び出しでそのトークンを渡す。通常のケーパビリティチェックが走る前に、`bpf_token_get_from_fd()`、`bpf_token_allow_cmd()`、`bpf_token_allow_prog_type()`、`bpf_token_allow_map_type()`（いずれも`kernel/bpf/token.c`）が、要求された特定のコマンド、プログラムタイプ、mapタイプをそのトークンがカバーしているかどうかを判定する。これは、マウントした側自身の信頼境界を緩めることなく、「コンテナがrootでなくても*この特定の*XDPプログラムをロードできる」ことを可能にする仕組みである。

## `unprivileged_bpf_disabled`: 2状態ではなく3状態 {#unprivileged_bpf_disabled-three-states-not-two}

`kernel.unprivileged_bpf_disabled` sysctl（`kernel/bpf/syscall.c`）は、`CAP_BPF`を持たないプロセスがそもそもBPF mapを作成したりBPFプログラムをロードしたりできるかどうかを制御する — `map_create_alloc()`（`BPF_MAP_CREATE`のハンドラである`map_create()`から呼ばれ、`sysctl_unprivileged_bpf_disabled && !bpf_token_capable(token, CAP_BPF)`をチェックする）と`bpf_prog_load()`（`sysctl_unprivileged_bpf_disabled && !bpf_cap`をチェックする）は、これが真であれば即座に拒否する。`map_create_alloc()`はmap作成パスを最近分割してできたものであり、古いカーネルでは同等のチェックが`map_create()`内に直接置かれている。ただし、この値はブール値ではない。3つの状態があり、そのうち2つの違いが重要である。

```c
/* kernel/bpf/syscall.c */
int sysctl_unprivileged_bpf_disabled __read_mostly =
	IS_BUILTIN(CONFIG_BPF_UNPRIV_DEFAULT_OFF) ? 2 : 0;
```

- **0** — 非特権BPFが許可される。
- **1** — 無効化され、かつ*ロックされる*。一度1に設定されると、カーネル自身の書き込みハンドラはそれを元に戻すことを拒否する（`bpf_unpriv_handler()`は`unpriv_enable == 1`を一方向のラッチとして扱い、以降の`1`以外の書き込みはすべて`-EPERM`を返す）。こうなると、稼働中のカーネルで非特権BPFを再び有効にする方法は再起動以外にない。
- **2** — デフォルトで無効（`CONFIG_BPF_UNPRIV_DEFAULT_OFF`による）だが、*元に戻せる*。管理者は`0`を書き戻して再び有効にできる。

これは仮定上の区別ではない。2つの別々のコミットが関わっている。`08389d888287`（2021年5月、Linux 5.13に取り込まれた）は`CONFIG_BPF_UNPRIV_DEFAULT_OFF` Kconfigオプションと3値のsysctl自体を追加したが、ノブをオンにはしなかった — このKconfigシンボルは5.13、5.14、5.15では`default`を持たなかったため、アップストリームのカーネルは依然として非特権BPFを許可した状態で出荷されていた。`8a03e56b253e`（2021年10月、Linux 5.16に取り込まれた）が実際に`default y`を追加したコミットであり、これによってデフォルトで無効（値2。元に戻せることが特に理由である）がアップストリームで出荷される振る舞いになった — その経緯については[非特権BPFのデフォルト無効化](war-stories/unprivileged-bpf-off.md)の変更を参照のこと。値1は、特定のマシンで扉を永久に閉ざすと決めた管理者のために存在する。

このsysctlへの書き込み自体には、現在の値や目標の値にかかわらず`CAP_SYS_ADMIN`が必要である。

## 非特権BPFがまだできること {#what-unprivileged-bpf-can-still-do}

非特権BPFをまだ許可しているカーネル（`unprivileged_bpf_disabled`が0 — 5.16以降のアップストリームの出荷時デフォルトではないが、`CONFIG_BPF_UNPRIV_DEFAULT_OFF`なしでビルドされたカーネルや、sysctlで明示的に再有効化されたカーネルでは今でも一般的である）では、`CAP_BPF`を持たないプロセスは、Verifierだけが課す制限をはるかに超えて制限される。

- **プログラムタイプ**: `CAP_BPF`なしでロードできるのは`BPF_PROG_TYPE_SOCKET_FILTER`と`BPF_PROG_TYPE_CGROUP_SKB`だけである（`kernel/bpf/syscall.c`。`unprivileged_bpf_disabled`のゲートの直後でチェックされる）。
- **プログラムサイズ**: 非特権プログラムは`BPF_MAXINSNS` — 4096命令（`include/uapi/linux/bpf_common.h`）— に制限される。これに対し、`CAP_BPF`を持つプログラムの上限は`BPF_COMPLEXITY_LIMIT_INSNS`、すなわち1,000,000である。これは`include/linux/bpf.h`で定義され、`kernel/bpf/verifier.c`でVerifier自身のパス探索の予算として使われる。
- **ポインタと投機実行による漏洩**: `bpf_allow_ptr_leaks()`と上述のSpectre回避チェックは、いずれも`CAP_PERFMON`をそれらを緩める1つの手段として受け入れる — だが、それぞれがケーパビリティとは無関係の独自の抜け道（JITが構造的に影響を受けないアーキテクチャ向けの`bpf_jit_bypass_spec_v1()`/`bpf_jit_bypass_spec_v4()`、あるいは`mitigations=off`での起動）も持つ。そのため、影響を受けるアーキテクチャでデフォルトの緩和策が有効なマシンでは、非特権プログラムにはVerifierの最も厳格で防御的なルールが適用され、ケーパビリティだけでそれを緩める方法はない。
- **アラインされていないアクセス**: アラインされていないアクセスを効率的に行えないアーキテクチャでは、`BPF_F_ANY_ALIGNMENT`にも`CAP_BPF`が必要である。

これらはいずれもVerifierの安全性証明によって強制されるものではない — それはVerifierの上に重ねられた、ケーパビリティによって制御される別のポリシー層であり、[事件簿のページ](war-stories.md)にあるVerifierのCVEを受けて実際に変化したのはこの層である。

## セキュリティモジュールとしてのBPF: BPF LSM {#bpf-as-a-security-module-bpf-lsm}

ここまでの内容はすべて、誰がBPFをロードできるかを制御する話であった。BPF LSMはこの関係を逆転させる。BPFプログラムが[LSM](../security/lsm.md)フックに直接アタッチされ、カーネル自身のアクセス制御判断の一部*となる*ことを可能にするのである。

`kernel/bpf/bpf_lsm.c`（Googleによる。Linux 5.7で追加 — v5.6には存在せず、v5.7には存在することを確認済み）は、BPFをアタッチできるすべてのLSMフックについて、弱シンボルの何もしない関数を宣言することで機能する。

```c
/* kernel/bpf/bpf_lsm.c */
#define LSM_HOOK(RET, DEFAULT, NAME, ...)	\
__weak noinline RET bpf_lsm_##NAME(__VA_ARGS__)	\
{						\
	return DEFAULT;				\
}
#include <linux/lsm_hook_defs.h>
#undef LSM_HOOK
```

各`bpf_lsm_<hookname>()`スタブはアタッチ可能なBTF関数として登録され（同じファイル内の`BTF_SET_START(bpf_lsm_hooks)` / `BTF_ID(func, bpf_lsm_##NAME)`）、`BPF_PROG_TYPE_LSM`（`include/uapi/linux/bpf.h`）タイプのBPFプログラムがそのいずれかにアタッチする — 通常のアクセス制御フックには`BPF_LSM_MAC`、cgroupスコープの変種には`BPF_LSM_CGROUP`を使う。アタッチ先が本物のLSMフックなので、プログラムの戻り値は、SELinuxやAppArmorと同じように実際に操作を拒否できる — 違いは、ポリシーが組み込みまたはロードされたセキュリティモジュールではなくBPFプログラムである点であり、そのため短命にしたり、実行時に生成したり、ちょうど1つのワークロードにスコープを限定したりできる。Verifierは、このプログラムタイプに固有の追加ルールを1つ課す。`bpf_lsm_verify_prog()`（`kernel/bpf/bpf_lsm.c`）はGPL互換でないLSMプログラムをすべて拒否し、プロプライエタリなポリシーがセキュリティ判断の地点にアタッチすることを許さない。

`CONFIG_BPF_LSM`（`kernel/bpf/Kconfig`）がビルド時のスイッチであり、そのヘルプテキストは意図された用途について率直である。"Enables instrumentation of the security hooks with BPF programs for implementing dynamic MAC and Audit Policies."（訳: 動的なMACおよび監査ポリシーを実装するために、BPFプログラムによるセキュリティフックの計装を有効にする。）これは`BPF_EVENTS`、`BPF_SYSCALL`、`SECURITY`、`BPF_JIT`がすべて有効であることに依存している — 最小構成のカーネルビルドで偶然手に入るものではない。ただし、組み込んでビルドすればそれで終わりではない。LSMは、`LSM` Kconfig文字列が有効にする順序付きリストに含まれている場合にのみ有効になる（`security/Kconfig`では、どのディストリビューション向けのデフォルトリストにも`bpf`が含まれている）。そしてそのリストは「ブート時に`lsm=`パラメータで制御できる」— `bpf`を含まないカスタムの`lsm=`ブート引数を指定すると、`CONFIG_BPF_LSM=y`でビルドされたカーネルであっても、BPF LSMは黙って無効になる。

LSMにアタッチされるBPFプログラムのロードも、別個のより寛容なパスではなく、上述の通常のケーパビリティルールによって制御される。`BPF_PROG_TYPE_LSM`は非特権ロードが許可される2つのプログラムタイプに含まれないので、そもそも`CONFIG_BPF_LSM`が組み込まれていることに加えて`CAP_BPF`が必要である — さらに、`BPF_PROG_TYPE_LSM`は`is_perfmon_prog_type()`（`kernel/bpf/syscall.c`）がカバーするプログラムタイプの1つでもあるため、ロードには`CAP_PERFMON`も必要になる。BPF LSMが変えるのは、*十分な特権を持つ*BPFプログラムが何に影響を与えてよいかであって、誰がそれをロードできるかではない。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/bpf/syscall.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/bpf/syscall.c) — `bpf_unpriv_handler()`、`unprivileged_bpf_disabled` sysctlのテーブルエントリ、プログラム/map作成時のケーパビリティチェック
- [include/linux/bpf.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/bpf.h) — `bpf_token_capable()`の宣言、`bpf_allow_ptr_leaks()`、`bpf_bypass_spec_v1()`/`bpf_bypass_spec_v4()`
- [kernel/bpf/token.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/bpf/token.c) — `bpf_token_capable()`の実際の定義と、トークン委譲のチェック（`bpf_token_get_from_fd()`、`bpf_token_allow_cmd()`、`bpf_token_allow_prog_type()`、`bpf_token_allow_map_type()`）
- [include/uapi/linux/bpf_common.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/bpf_common.h) — `BPF_MAXINSNS`
- [kernel/bpf/bpf_lsm.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/bpf/bpf_lsm.c) — `bpf_lsm_<hook>()`の弱シンボルの仕組みとBTFへの登録
- [kernel/bpf/Kconfig](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/bpf/Kconfig) — `CONFIG_BPF_LSM`とその依存関係
- [include/uapi/linux/bpf.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/bpf.h) — `BPF_PROG_TYPE_LSM`、`BPF_LSM_MAC`、`BPF_LSM_CGROUP`

### 関連ページ {#related-pages}

- [BPF Verifier](bpf-verifier.md) — このページのケーパビリティ層がその上に載っている安全性証明
- [事件簿](war-stories.md) — ケーパビリティモデルが存在する理由: VerifierのCVEと、2021年の非特権BPFデフォルト無効化の変更
- [LSMフレームワーク](../security/lsm.md) — BPF LSMがアタッチするフックのフレームワーク
- [Linuxケーパビリティ](../security/capabilities.md) — `CAP_BPF`、`CAP_PERFMON`、そしてより広範なケーパビリティの分割
