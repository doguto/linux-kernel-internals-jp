# MDS: バッファを早すぎるタイミングでフラッシュしていた緩和策 {#mds-the-mitigation-that-flushed-the-buffers-too-early}

> CVE-2018-12130 — Microarchitectural Data Samplingに対するKVMの当初の防御は、実際にゲストへ入る数命令前にVERWでCPUバッファをフラッシュしており、その間にあるレジスタのスピルやスタックへのプッシュはどれもそれ自体がメモリアクセスで、それらのバッファを新しいホストデータで再び満たしうるものだった。この隙間は5年間検証されないまま放置され、日常的なハードニング作業が副作用として塞ぐまで残っていた

公開
:   2019年5月14日（複数ベンダーによる協調公開。関連する3つのCVEと同時）

CVSS
:   5.9（CVSS 3.1、`AV:L/AC:H/PR:N/UI:N/S:C/C:H/I:N/A:N`）

バグの混入時期
:   コミット`650b68a0622f9`。KVMの当初のMDS緩和策で、2019年3月にエンバーゴ下でコミットされ、リリースされたカーネルにはLinux 5.2（2019年7月）で初めて入った

修正バージョン
:   コミット`43fb862de8f62`。2024年2月にコミットされ、Linux 6.8（2024年3月10日リリース）で出荷された。当初の緩和策から5年後であり、この特定の隙間に対する修正であると明示的に特定されたのは2025年後半の後続コミットになってからだった

エクスプロイトツール
:   公開されたエクスプロイトツールはない。確認された悪用が起きる前に隙間は塞がれた

実際の悪用
:   確認された事例なし（CISA KEVに未掲載）

*[事件簿: 仮想化（KVM）のバグとエスケープ](../war-stories.md)の一部。*

## 発生前の状態 {#before-state}

MDS（Microarchitectural Data Sampling）は、関連する一群のCPUの欠陥（MSBDS/CVE-2018-12126、MFBDS/CVE-2018-12130、MLPDS/CVE-2018-12127、MDSUM/CVE-2019-11091）を指す。カーネル自身のドキュメントによれば、これらの欠陥では通常のストア、ロード、L1リフィル操作の際に"processors write data into temporary microarchitectural structures (buffers)"（プロセッサが一時的なマイクロアーキテクチャ構造（バッファ）にデータを書き込む）、そのデータは上書きされる前に他のコードによって投機的にサンプリングされうる。この緩和策のために転用された`VERW`命令は、副作用としてそれらのバッファをクリアする。カーネルの修正は、信頼境界が変わる2つの境界で`VERW`を実行することだった。ユーザー空間への復帰時と、ここで問題となるケースであるVMエントリの直前である。後者により、ゲストは直前に動いていたホストコードがバッファに残したホストデータをサンプリングできなくなる。

## きっかけ {#the-trigger}

`650b68a0622f9`（"x86/kvm/vmx: Add MDS protection when L1D Flush is not active"、2019年3月にエンバーゴ下でコミットされ、Linux 5.2で初めて出荷）は、ゲストエントリ時のVERW呼び出しを、`__vmx_vcpu_run()`内の通常のC関数呼び出し`mds_clear_cpu_buffers()`として追加した。これは、実際のレジスタ保存/復元とVMLAUNCH/VMRESUMEを行う大きなインライン`asm()`ブロックの直前に置かれていた。ソースレベルでの直前は、機械語レベルでの直前と同じではない。その呼び出しと実際のVMエントリ命令との間には、コンパイラが生成する量の分からないコードと、asmブロック自身の「ホストレジスタを保存する」プロローグがあった。このプロローグは、文字どおりレジスタをスタックにプッシュする。そのどれもがメモリ書き込みである。そして、この脆弱性クラスに関するカーネル自身の説明によれば、メモリ書き込みはまさにMDSが標的とするマイクロアーキテクチャのフィルバッファを埋める種類の操作である。

## 観測された挙動 {#observed-behavior}

実際の帰結はこうである。VERWがバッファをきれいにし、その後、コンパイラとasmブロック自身のレジスタ保存プロローグが生成する通常のメモリトラフィック（実際のプロローグで見られる具体例の一つが`%rbp`のプッシュである）が、CPUが実際にゲストへ制御を渡す前に、それらのバッファを新しいホストデータで再び満たしうる。その時点以降にMFBDSを悪用するゲストは、VERWがクリアするはずだったフラッシュ前の内容ではなく、*新たに*バッファされたホストデータをサンプリングすることになる。5年後、`43fb862de8f62`（"KVM/VMX: Move VERW closer to VMentry for MDS mitigation"、2024年2月）は、まさにこの観点でリスクを説明した: "After VERW, any memory access like register push onto stack may put host data in MDS affected CPU buffers. A guest can then use MDS to sample host data. Although likelihood of secrets surviving in registers at current VERW callsite is less, but it can't be ruled out."（訳: VERWの後、レジスタのスタックへのプッシュのようなメモリアクセスはどれも、MDSの影響を受けるCPUバッファにホストデータを置きうる。するとゲストはMDSを使ってホストデータをサンプリングできる。現在のVERW呼び出し箇所でレジスタに秘密が残っている可能性は低いが、排除はできない。）この修正は、バッファクリアを手書きのアセンブリスタブ自体の中に移した。`CLEAR_CPU_BUFFERS`マクロとして、ゲストのRAXがロードされた直後、VMLAUNCHまたはVMRESUMEへの分岐の直前に挿入したのである。これにより、その間でCレベルの呼び出し、スピル、メモリ書き込みが起きることは物理的に不可能になった。

さらに後のコミット`e6ff1d61de51e`（2025年11月、Sean Christopherson）は、関連するMMIO固有の緩和策を作り直すもので、その経緯を率直に述べ、実際に何が起きていたのかに名前を与えた: "the flaw goes back to the introduction of the MDS mitigation. The MDS mitigation was inadvertently fixed by commit 43fb862de8f6 ... but previous kernels that flush CPU buffers in vmx_vcpu_enter_exit() are affected (though it's unlikely the flaw is meaningfully exploitable even older kernels)."（訳: この欠陥はMDS緩和策の導入にまでさかのぼる。MDS緩和策はコミット43fb862de8f6によって意図せず修正されていた……が、vmx_vcpu_enter_exit()でCPUバッファをフラッシュする以前のカーネルは影響を受ける（古いカーネルであっても、この欠陥が意味のある形で悪用可能である可能性は低いが）。）"inadvertently"（意図せず）という語がここで重要な意味を持っている。2024年のコミットの作者たちは、言語化はできたが2019年の元の設計まではたどっていなかった*理論上の*懸念に対して呼び出し箇所をハードニングしていたのであり、2024年の変更がすでに当初の修正にあった本物の（おそらく深刻度は低いが）隙間を塞いでいたことに気づくには、1年以上後に同じコードの隣接部分に取り組んでいた3人目の開発者を必要とした。

## なぜ起きたか {#why-it-happened}

2019年の当初の緩和策は、現実の時間的プレッシャーの下で書かれた。MDSは同時に公開された4つのCVEの一つであり、ドキュメント、sysfsでの報告、コマンドラインでの制御、そしてユーザー空間とKVMの両方に対する緩和策を同時にカバーする23コミットのシリーズの一部だった。VERW呼び出しを通常のC関数呼び出しとして置くのは、既存のCコードに緩和策を追加するうえで自然で機械的に最も簡単な方法だった。その特定のパッチに関わった誰も、その呼び出しと本物のVMエントリ命令との間にコンパイラ生成コードが正確に何バイトあるのかを明示的に検討したり、それらのバイトが実際に何をしているのかを監査したりはしなかったようである。この隙間は却下されたトレードオフではなかった。その場の誰もまだ問うていなかった問いだったのであり、2024年にこの同じMDS呼び出し箇所に対するハードニング作業が、関連はするが別の懸念のためにたまたまコードを移動したことでようやく塞がれ、さらにその1年後に隣接するコードに取り組んでいた3人目の開発者によって、この特定の隙間を修正していたと認識されたのである。

## 解決 {#resolution}

`43fb862de8f62`は、`vmx.c`の`vmx_vcpu_enter_exit()`からMDS固有の`mds_clear_cpu_buffers()`分岐を削除し、`CLEAR_CPU_BUFFERS`マクロ（alternativeでパッチされる生の`VERW`）を`vmenter.S`の`__vmx_vcpu_run`アセンブリルーチンに直接挿入した。その位置は、VMLAUNCHまたはVMRESUMEを実行する分岐の直前の最後の命令である。その時点ではゲストのレジスタはすべてすでに各GPRにロードされており、CPUがゲストへのエントリを確定するまでにメモリに触れるものは何もない。これにより、MDSのウィンドウは単に狭められるのではなく完全に閉じられた。無関係なMMIO Stale Data緩和策のための別の`mds_clear_cpu_buffers()`呼び出しは、`vmx_vcpu_enter_exit()`にCレベルのまま残り、同じ早すぎるタイミングの隙間を抱えていた。2025年後半の`e6ff1d61de51e`がそれも塞いだ。関連するMMIO Stale Dataのバッファクリアを、共有の`ALTERNATIVE_2`ブロックを介して同じアセンブリ上の位置に移し、緩和策のコードパスを統合すると同時に、そのコミットメッセージによれば、特定の緩和策設定の組み合わせのもとで"a mostly-benign flaw where KVM wouldn't do any clearing/flushing"（KVMがクリア/フラッシュを一切行わない、ほぼ無害な欠陥）を修正した。これは同じ近辺にあった2つ目の小さな隙間であり、1つ目を片付ける過程で見つかったものである。

## 教訓 {#what-it-taught-us}

**締め切りのプレッシャーの下で追加されたセキュリティ緩和策は、その掲げた目的においては正しくても、周囲のコードにどう組み込まれているかという点に、検証されていない隙間を残しうる。** 2019年の修正は意図どおりゲストエントリ時にVERWを追加した。ただ、フラッシュとそれが保護するエントリとの間で何もメモリに触れられないことを検証していなかったのである。

**修正は、まだ誰も名前を付けていない本物の脆弱性を塞ぐことがある。** 2024年のコミットの作者たちは、自分たちの推論の中にあった理論上の懸念に対してハードニングしていたのであり、5年前のCVEを意識的にパッチしていたわけではない。実際に何が修正されていたのかを認識し文書化するには、1年以上後に隣接するコードに取り組んでいた第三者を必要とした。

!!! warning "注意すべきパターン"
    セキュリティ上重要な命令（キャッシュフラッシュ、バッファクリア、権限チェック）が、信頼境界を越える前の「最後に起きること」でなければならない場合、その配置をCレベルだけでなくアセンブリレベルで検証すること。コンパイラは、C文と論理的にその後に続く機械語命令との間に、スピル、スタック調整、その他のメモリトラフィックを自由に挿入でき、そのようなトラフィックのどれもが、セキュリティ命令が提供するはずだった保証を台無しにしうる。

## 関連項目 {#see-also}

- [L1TF: When a Not-Present Page Table Entry Wasn't Not-Present Enough](l1tf-foreshadow-vmm.md) — 同じKVMのVMエントリパスにある兄弟的な投機実行緩和策で、独自のSMT共有に関する注意点を持つ
- [KVMのアーキテクチャ](../kvm-arch.md) — VMエントリ/エグジットと、この緩和策が保護する`vmlaunch`/`vmresume`境界
- [The rt_mutex Deadlock Detector's Atomic-Sleep Bug](../../locking/war-stories/rtmutex-deadlock-detector-atomic-sleep.md) — ロジックではなくコードの配置こそが実際の欠陥だった、比較可能な事例

## 外部参考資料 {#external-references}

- [git.kernel.org: 650b68a0622f9](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=650b68a0622f933444a6d66936abb3103029413b) — "x86/kvm/vmx: Add MDS protection when L1D Flush is not active"、2019年の当初の緩和策
- [git.kernel.org: 43fb862de8f62](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=43fb862de8f628c5db5e96831c915b9aebf62d33) — "KVM/VMX: Move VERW closer to VMentry for MDS mitigation"、2024年の修正
- [git.kernel.org: e6ff1d61de51e](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e6ff1d61de51ec5fe94c5fb79544a93f494104eb) — 元の欠陥に明示的に名前を付け、関連する隙間を塞いだ2025年のコミット
- [カーネルドキュメント: MDS - Microarchitectural Data Sampling](https://docs.kernel.org/admin-guide/hw-vuln/mds.html) — MDS CVEファミリとVERWベースの緩和策に関するカーネル自身の技術的説明
- [NVD: CVE-2018-12130](https://nvd.nist.gov/vuln/detail/CVE-2018-12130) — CVEレコード。CVSS 3.1 5.9、2019年5月30日公開
