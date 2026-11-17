# L1TF：不在のページテーブルエントリが十分に不在ではなかったとき {#l1tf-when-a-not-present-page-table-entry-wasnt-not-present-enough}

> CVE-2018-3646 — 不在の EPT エントリに対するターミナルなページフォールトであっても、CPU は L1 データキャッシュにたまたま置かれていた古いデータを投機的にフォワードしてしまった。そして仮想化されたゲストでは、そのキャッシュはコア上のすべての兄弟ハイパースレッドで、ホストもゲストも区別なく共有されている

公開日
:   2018年8月14日（複数ベンダーによる協調的開示）

報告者
:   Intel 自身の社内研究チーム。それに先行する学術的な Foreshadow/SGX の研究（CVE-2018-3615）を発展させたもの

CVSS
:   5.6（CVSS 3.1、`AV:L/AC:H/PR:L/UI:N/S:C/C:H/I:N/A:N`）

バグの混入時期
:   ハードウェアの投機実行の振る舞いであり、どのカーネルの変更によって持ち込まれたものでもない

修正
:   「l1tf-final」シリーズ、マージコミット `958f338e96f87`、メインライン Linux 4.19（2018年8月14日）

エクスプロイトツール
:   兵器化された公開ツールはない

実際の悪用
:   確認された事例はない（CISA KEV には掲載されていない）

*[事件簿：仮想化（KVM）のバグとエスケープ](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

L1TF は、「Foreshadow-NG」という総称のもとで一緒に開示された3つの関連する CVE の1つである — SGX エンクレーブについての CVE-2018-3615、OS/SMM のケースについての CVE-2018-3620、そして仮想化についての CVE-2018-3646 で、いずれも同じ根本的な CPU の振る舞いに由来する。カーネル自身のドキュメントは、このメカニズムを正確に述べている。"If an instruction accesses a virtual address for which the relevant page table entry (PTE) has the Present bit cleared or other reserved bits set, then speculative execution ignores the invalid PTE and loads the referenced data if it is present in the Level 1 Data Cache, as if the page referenced by the address bits in the PTE was still present and accessible."（訳: 命令が、対応するページテーブルエントリ（PTE）の Present ビットがクリアされているか、他の予約ビットがセットされている仮想アドレスにアクセスした場合、投機実行は無効な PTE を無視し、参照されたデータが L1 データキャッシュに存在すれば、PTE のアドレスビットが指す page がまだ存在してアクセス可能であるかのように、そのデータをロードする）。その命令はリタイア時に最終的にページフォールトを起こす — しかしその時点では、投機的にロードされたデータはすでに他の投機的命令から観測可能になっており、サイドチャネルの窓が開いている。

## きっかけ {#the-trigger}

通常のホスト側の攻撃にとって、これは Meltdown ファミリーの一変種である。仮想化に関しては、それとは異なる、より悪い問題になる。"the speculation bypasses the extended page table (EPT) protection mechanism"（訳: 投機が拡張ページテーブル（EPT）の保護機構をバイパスする）、それも完全にである。ゲストは、古い物理アドレスビットがホスト物理メモリのどこでも指すような不在の EPT エントリを構築でき、CPU の投機的な L1D 検索は、そのアドレスがゲストに属するかどうかを一切チェックしない — そのアドレスで L1D にたまたま存在しているものを何でも返すだけである。バイパスすべき権限チェックすら存在しない。ハードウェアは、そもそもそれを参照するところまで進まないからである。

## 観測された挙動 {#observed-behavior}

マルチテナントの側面こそが、これをクラウドホスティングにとって特に危険なものにしている。同じ物理コア上の兄弟ハイパースレッドは L1 データキャッシュを共有する。カーネルのドキュメントは、その到達範囲について明示的である。"a malicious guest running on one Hyperthread can attack the data which is brought into the L1D by the context which runs on the sibling Hyperthread of the same physical core. This context can be host OS, host user space or a different guest."（訳: あるハイパースレッド上で動く悪意のあるゲストは、同じ物理コアの兄弟ハイパースレッド上で動くコンテキストが L1D に持ち込んだデータを攻撃できる。このコンテキストはホスト OS、ホストのユーザー空間、あるいは別のゲストでありうる）。悪意のある VM が他のテナントのデータを読むのに、ハイパーバイザのバグは必要ない — 必要なのは、適切な瞬間にそのテナントと物理コアを共有することだけであり、それはクラウドスケジューラの通常の vCPU 配置が、何の設定ミスもなしに生み出しうる状況である。

## なぜ起きたか {#why-it-happened}

これは従来の意味でのソフトウェアの欠陥ではない — 脆弱な振る舞いは CPU の投機実行ロジックの中にあり、カーネルやハイパーバイザが何をしようとハードウェアに存在する。カーネルの緩和策の選択肢はすべて、ソフトウェアが制御できる特定の窓を閉じることに関わる。ゲストがターミナルフォールトで競争を仕掛けられる瞬間に L1D に何が存在しているか、そしてその窓が開いている間に、ゲストが異なるセキュリティコンテキストとコアを共有することがありうるかどうか、である。

## 解決 {#resolution}

KVM の主要な防御は、VM entry 時の L1D キャッシュのフラッシュである — ゲストが次の投機的なターミナルフォールトの探査を実行する前に、*ホスト*がキャッシュに置いたものを一掃する。カーネルは2つのモードを提供する。`always` は最大限の保護のために VM entry のたびにフラッシュするが、実際のコストがかかる（カーネルのドキュメントは、VM exit の頻度とゲストのワークロードに応じて "performance degradation in the range of 1% to 50%"（訳: 1%から50%の範囲の性能低下）を挙げている）。デフォルトの `cond` は、その間に通ったホストのコードパスが秘密を露出しないと監査済みの VM exit の後ではフラッシュを省略する — ただしドキュメントは、その監査済みのパスも "can [still] leak information about the address space layout of the hypervisor"（訳: ハイパーバイザのアドレス空間レイアウトに関する情報を［依然として］漏らしうる）と注意深く記しており、より狭いが実在する残存リークである。フラッシュに加えて、PTE 反転が、不在の PTE に格納された物理アドレスビットを反転させることで、スワップアウトされたホストの page を無条件に保護する。これにより、スワップ関連の PTE に対するターミナルフォールトの探査を、選んだ物理アドレスに向けることができなくなる — その代償として、使用可能なスワップは約16TBに制限される。

フラッシュだけでは兄弟ハイパースレッドのケースは解決しない。カーネルのドキュメントによれば、"L1D flush does not prevent the SMT problem because the sibling thread will also bring back its data into the L1D which makes it attackable again"（訳: L1D のフラッシュは SMT の問題を防がない。兄弟スレッドもまた自分のデータを L1D に持ち戻し、それを再び攻撃可能にするからである）。マルチテナントのホストに対する完全な緩和策は、SMT（ハイパースレッディング）を完全に無効にするか、兄弟ハイパースレッドが異なるセキュリティドメインを同時に実行しないようにスケジューリングすることである — これは実際の処理能力のコストであり、カーネルは意図的にこれをデフォルトでは課さず、`l1tf=` と `kvm-intel.vmentry_l1d_flush=` というコマンドラインおよびモジュールのオプションを通じて、トレードオフを管理者に委ねている。ネストされた仮想化も独自の注意深い扱いを受ける。KVM がベアメタル（L0）のハイパーバイザとして、自身のゲスト（L2）を実行するネストされたハイパーバイザ（L1）を実行している場合、L0 はネストされたハイパーバイザとネストされた VM の間の切り替えのたびに、両方向で L1D をフラッシュし、同時にネストされたハイパーバイザには*さらに*フラッシュしないよう明示的に指示する — ネストされた遷移のたびに冗長な二重フラッシュが起きるのを避けるためである。

## 得られた教訓 {#what-it-taught-us}

**保護機構（EPT）は、そもそもそれを参照しない CPU の振る舞いによって完全にバイパスされうる。** EPT の保証 — ゲストはハイパーバイザがそれのためにマップしたメモリにしかアクセスできない — は、CPU がその境界を強制するために実際に EPT をたどることを前提としている。ターミナルフォールトの投機は、そのたどりを完全に短絡させるので、このパスでは保護そのものの正しさが実際に試されることは一度もなかった。

**共有されたマイクロアーキテクチャの状態（ここでは兄弟ハイパースレッド間の L1D）は、ソフトウェアスタックが想定して設計していなかったセキュリティ境界であり、それを閉じるには実際の処理能力のコストがかかる。** SMT をデフォルトでは無効にしないというカーネル自身の選択は、完全に安全な構成と完全に高性能な構成が別々の構成であり、そのトレードオフは分離要件を持つ者が明示的に行わなければならない、ということを意図的に認めたものである。

!!! warning "注意すべきパターン"
    ハードウェアやソフトウェアの機構が、権限構造（ページテーブル、EPT、ケーパビリティチェック）を参照することで分離を強制している場合は、その参照に到達*しない*すべてのパス — エラーパス、投機的パス、チェックがすでに別の場所で行われたと仮定する高速パス — で何が起きるかを問うこと。到達するすべてのパスで正しい権限構造は、それを必要とするすべてのパスで実際に到達される権限構造と同じではない。

## 関連項目 {#see-also}

- [メモリ仮想化](../kvm-memory.md) — EPT/NPT と、このバグがバイパスする2段目のアドレス変換
- [ネストされた仮想化](../nested-virt.md) — 上記のネストされたゲストの緩和策で言及した L0/L1/L2 モデル
- [MDS/KVM の VERW タイミングの隙間](mds-verw-timing-gap.md) — KVM における2つ目の投機実行ハードウェア緩和策。そのタイミングも、L1TF のフラッシュするかしないかの選択と同じくらい重要であることが判明した

## 外部参考文献 {#external-references}

- [git.kernel.org: 958f338e96f87](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=958f338e96f874a0d29442396d6adf9c1e17aa2d) — KVM の緩和策をメインラインに入れた、マージされた「l1tf-final」シリーズ
- [カーネルドキュメント: L1TF - L1 Terminal Fault](https://docs.kernel.org/admin-guide/hw-vuln/l1tf.html) — カーネル自身の技術的説明と緩和策のリファレンス。上記のすべての直接引用の出典
- [CERT/CC VU#982149](https://www.kb.cert.org/vuls/id/982149) — Foreshadow-NG ファミリーの3つの CVE すべてをカバーする協調的開示のまとめ。謝辞の節には "L1TF OS/SMM and L1TF VMM were found internally by Intel researchers after expanding on the research of L1TF SGX" とクレジットされている
- [Foreshadow attack research](https://foreshadowattack.eu/) — 元の SGX に焦点を当てた Foreshadow の研究（CVE-2018-3615）。2つのチーム（KU Leuven/imec-DistriNet と、別途 Technion、University of Michigan、University of Adelaide/CSIRO の Data61）によって独立かつ同時に発見され、その成果を Intel が社内で発展させて、このページで扱う OS/SMM と VMM の変種を特定した
- [NVD: CVE-2018-3646](https://nvd.nist.gov/vuln/detail/CVE-2018-3646) — CVE レコード。CVSS 3.1 5.6、2018年8月14日公開
