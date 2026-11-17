# fw_devlink=on: 定着するまでに2度ブートを壊したデフォルト {#fw_devlinkon-the-default-that-broke-boot-twice-before-it-stuck}

> CVEではない — ファームウェアが記述するデバイスグラフに基づくprobe順序の自動強制をデフォルトで有効にしたところ、3か月のうちに2度、複数のプラットフォームでブートが壊れ、リバートと再挑戦のサイクルを一巡させることになった。しかもリバートコミット自身の「5.12向け」という位置付けは意味をなさなかった。Linux 5.12は終始古いデフォルトのまま出荷され、新しいデフォルトが初めてユーザーに届いたのは5.13だった

最初の試み
:   コミット`e590474768f1`、2020年12月18日 — `fw_devlink=on`をデフォルトに設定

最初のリバート
:   コミット`3e4c982f1ce7`、2021年2月18日 — プラットフォームのブートリグレッションを受けて「5.12向け」にリバート

再有効化（"Take II"）
:   コミット`ea718c699055`、作成は2021年3月2日、driver-coreツリーへのコミットは2021年3月23日

実際にユーザーへ出荷されたリリース
:   Linux 5.13、2021年6月27日リリース — Linux 5.12（2021年4月25日）ではない。5.12はリバートの「5.12向け」という文言にもかかわらず古いデフォルトのまま出荷された

CVEではない
:   特定のファームウェア構成下の特定のプラットフォームに影響するブート信頼性のリグレッションであり、セキュリティ脆弱性ではない

*[事件簿: デバイスドライバ](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

`fw_devlink`は、ファームウェア記述にすでに存在する依存関係の情報、つまり`clocks`、`interrupts`、`dmas`、`power-domains`といったDevice Treeのプロパティや、ACPIの`_DEP`を、単なる情報用のメタデータではなく、probeをゲートする実際のカーネルの`device_link`に変換する。`fw_devlink_parse_fwtree()`はファームウェアノードのツリーを辿り、各fwnodeの`add_links`操作を呼び出してこれらの関係を発見する。発見された依存関係の両端が`struct device`を持つと、`fw_devlink_create_devlink()`がそれを実際のリンクに変える。作成されたリンクが何を意味するかは、2つのフラグセットが制御する:

```c
#define FW_DEVLINK_FLAGS_PERMISSIVE (DL_FLAG_INFERRED | DL_FLAG_SYNC_STATE_ONLY)
#define FW_DEVLINK_FLAGS_ON         (DL_FLAG_INFERRED | DL_FLAG_AUTOPROBE_CONSUMER)
```

`permissive`のもとでは、推論されたリンクはサプライヤの`sync_state()`コールバックをゲートするだけであり、サプライヤがprobeされる前にコンシューマがprobeされることを妨げない。`on`のもとでは、リンクは`DL_FLAG_AUTOPROBE_CONSUMER`を持つ。サプライヤがまだprobeされていないコンシューマは、ドライバコアの既存の`-EPROBE_DEFER`による再試行の仕組みを通じて自動的に延期され、それを要求するドライバのコードは不要である。fw_devlinkがデフォルトの切り替えを試みたのはこれが初めてでもなかった。ちょうど1年前、`fw_devlink=permissive`そのものが一時的にデフォルトにされ（`c442a0d18744`、2020年3月21日）、Raspberry Pi 3Bと4のハードウェアで黙ってブートをフリーズさせ、6日後にリバートされた（`18555cb6db23`）。その後、そのコミット自身のメッセージが指し示す根本的な処理のバグが修正されると（`00b247557858`、"driver core: Fix handling of fw_devlink=permissive"）、約1か月後に再有効化された（`c78c31b374a6`、2020年4月28日）。`on`の顛末は、これと同じパターンを一段上で繰り返したものであり、仕組みは異なり（sync-stateの順序付けではなくprobeのブロック）、解決までの道のりははるかに長い。

`fw_devlink_relax_cycle()`は、推論された依存関係が循環を成すグラフを扱う。コンシューマになろうとするデバイス自身のコンシューマの連鎖を辿り、サプライヤになろうとするデバイスがすでに推移的にそのコンシューマに依存していると判明した場合、その循環に含まれるfw_devlinkが推論したすべてのリンクを、probeを無期限にデッドロックさせたままにするのではなく、sync-stateのみのリンクに格下げする。有効化コミット自身のメッセージは、デフォルト切り替えの直前にマージされたこの機能が最後の障害だったと明言している:「Cyclic dependencies in some firmware was one of the last remaining reasons fw_devlink=on couldn't be set by default. Now that cyclic dependencies don't block probing, set fw_devlink=on by default.」（訳: 一部のファームウェアにおける循環依存は、fw_devlink=onをデフォルトにできない最後に残った理由のひとつだった。循環依存がprobeをブロックしなくなったので、fw_devlink=onをデフォルトにする。）

## きっかけ {#the-trigger}

`e590474768f1`（"driver core: Set fw_devlink=on by default"、Saravana Kannan、2020年12月18日）は、1行のデフォルトを切り替え、期待される利点をコミットメッセージ自体に列挙している:「Significantly cuts down deferred probes. Device probe is effectively attempted in graph order. Makes it much easier to load drivers as modules without having to worry about functional dependencies between modules (depmod is still needed for symbol dependencies).」（訳: 延期されるprobeを大幅に削減する。デバイスのprobeは事実上グラフの順序で試行される。モジュール間の機能的な依存関係を気にせずに、ドライバをモジュールとしてロードすることがはるかに容易になる（シンボルの依存関係のためにdepmodは引き続き必要）。）また、実際にその後に起きた障害の形も予期している:「If this patch prevents some devices from probing, it's very likely due to the system having one or more device drivers that 'probe'/set up a device (DT node with compatible property) without creating a struct device for it.」（訳: このパッチによって一部のデバイスがprobeされなくなった場合、それはおそらく、システムに、struct deviceを作成せずにデバイス（compatibleプロパティを持つDTノード）を「probe」／セットアップするデバイスドライバが1つ以上存在するためである。）

## 観測された挙動 {#observed-behavior}

その予期された障害の形こそが、その後の2か月にわたり、無関係なプラットフォームで繰り返し表面化したものだった。Renesas R-Car Gen3ボードは完全にブートしなくなった。Geert UytterhoevenがShimoda-sanからの報告を中継し、bisectした:「Shimoda-san reported that next-20210111 and later fail to boot on Renesas R-Car Gen3 platforms. No output is seen, unless earlycon is enabled. I have bisected this to commit e590474768f1cc04...」（訳: Shimoda-sanから、next-20210111以降がRenesas R-Car Gen3プラットフォームでブートに失敗するとの報告があった。earlyconを有効にしない限り何も出力されない。bisectした結果、コミットe590474768f1cc04...に行き着いた。）根本原因は、R-CarのSystem Controllerが`early_initcall()`からパワードメインを登録しながら、自身の`struct device`を一切作成しないことだった。そのためfw_devlinkは、コンシューマが宣言したそれへの依存関係を、永久に満たされないものと見なした。最初の修正（`2dfc564bda4a`、"soc: renesas: rcar-sysc: Mark device node OF_POPULATED after init"）だけでは不十分だった。Geertは、その後も特定の構成でR-Car Gen2/Gen3がハングし続けると報告し（`CONFIG_RCAR_DMAC=n hangs: supplier e7310000.dma-controller not ready`）、さらなる追加対応が必要になった。

これとは別に、長年カーネルのブートCIを保守してきたGuenter Roeckが、QEMU上でnios2がハングすることを、同じコミットを指すbisectログとともに報告し、単一プラットフォームの報告では伝わらない問題の規模を捉えた一文を添えた:「It may also break a variety of other boot tests, but with 115 of 430 boot tests failing in -next it is difficult to identify all culprits.」（訳: 他のさまざまなブートテストも壊しているかもしれないが、-nextでは430のブートテストのうち115が失敗しているので、すべての原因を特定するのは難しい。）同じ時期に、他のプラットフォーム向けの修正も続いた。`e2c1b0ff38c9`はi.MXのAVIC割り込みコントローラを`IRQCHIP_DECLARE`に変換し、fw_devlinkがそれが`struct device`になるのを待つこと自体をやめるようにした（"boot issues on imx25 with fw_devlink=on that were reported by Martin"、訳: Martinが報告した、fw_devlink=onでのimx25のブート問題）。`f265f06af194`はDevice Treeの`interrupt-map`プロパティに対するfw_devlinkの処理を修正し、Marek SzyprowskiがSamsung Exynosのハードウェアでテストした。Szyprowskiは2021年2月の強化パッチ群のほぼ全体にテスターとして名を連ねており、R-Carとnios2に並ぶ、リグレッションを報告した2番目の主要なプラットフォームだった。

## なぜ起きたか {#why-it-happened}

これらのリグレッションはすべて同じ形をしていた。fw_devlinkの循環を断ち切るロジックは準備できていたが、仕組み全体の根底にあるより基本的な前提、つまりファームウェア記述で参照されているデバイスはいずれ対応する`struct device`を登録する、という前提が普遍的には成り立たなかったのである。ドライバコアの通常のデバイス登録経路を一切通らず、ブート初期のinitcallで自身を構成するプラットフォーム固有のコントローラは、その前提からは見えず、それはどれだけ循環検出のロジックを用意しても解決しない。fw_devlinkは、決して現れない`struct device`を永遠に待ち続けるだけである。これはまさに、有効化コミット自身のメッセージが抽象的に予言していた障害の形である。問題は、それを実際に引き起こすハードウェアの組み合わせが、デフォルトが切り替わり、はるかに広い範囲の実際のブート構成が新しいコードパスを通り始めるまで、完全には表面化しなかったことだった。

## 解決 {#resolution}

Greg Kroah-Hartmanは2021年2月18日にデフォルトをリバートした（`3e4c982f1ce7`）。そのメッセージは撤退というより計画として読める:「While things are _almost_ there and working for almost all systems, there are still reported regressions happening, so let's revert this default for 5.12. We can bring it back in linux-next after 5.12-rc1 is out to get more testing...」（訳: _ほぼ_ 完成していてほとんどすべてのシステムで動いているが、まだリグレッションが報告されているので、5.12向けにはこのデフォルトをリバートしよう。5.12-rc1が出た後にlinux-nextに戻して、さらにテストすればよい...）Saravana Kannanの"Take II"シリーズは予定どおりに続き、カバーレターは2021年3月2日に投稿された:「This series fixes the last few remaining issues reported when fw_devlink=on by default... As far as I know, there shouldn't have any more issues you reported that are still left unfixed after this series.」[sic]（訳: このシリーズは、fw_devlink=onをデフォルトにした際に報告された最後のいくつかの残りの問題を修正する... 私の知る限り、このシリーズの後に、報告された問題で未修正のまま残っているものはもうないはずだ。）`ea718c699055`は同じ日にデフォルトを再び切り替え、2021年3月23日にGregのdriver-coreツリーにコミットされた。

ここに「5.12向け」という位置付けが予期していなかった細部がある。Take IIは、Linux 5.12に間に合う形でLinusのメインラインツリーに入ることは一度もなかった。実際のv5.12のソースをv5.13と直接比較するとそれが確認できる。最終的なv5.12タグの`drivers/base/core.c`は依然として`fw_devlink_flags = FW_DEVLINK_FLAGS_PERMISSIVE`となっており、`ea718c699055`はv5.12の祖先ではない（`git compare`は両者を分岐したものとして報告する）が、v5.13-rc1の祖先では*ある*。Linux 5.12は2021年4月25日に、終始*古い*デフォルト、つまり2月のリバートがすでに元に戻していたまさにそのデフォルトでリリースされた。`fw_devlink=on`が初めてユーザーに届いたのは、2021年5月9日にタグ付けされたv5.13-rc1と、2021年6月27日の正式なv5.13リリースだった。

"Take II"が書かれてから実際に出荷されるまでのその隔たりは、実務上重要だった。v5.12が出荷された翌日の2021年4月26〜28日、Florian Fainelliは、まだメインラインに入っていないTake IIパッチをすでに取り込んだ5.12ベースの社内ビルドをテストしていて、BroadcomのARCH_BRCMSTBプラットフォームでの新たなリグレッションを報告した:「This change breaks booting on SCMI-based platforms such as ARCH_BRCMSTB... the SCMI clock provider was never probed which means that our UART driver never got a chance to get its clock and we have no console -> timeout.」（訳: この変更は、ARCH_BRCMSTBのようなSCMIベースのプラットフォームでのブートを壊す... SCMIのクロックプロバイダが一度もprobeされず、つまりUARTドライバがクロックを取得する機会がなく、コンソールがない -> タイムアウト。）Cristian MarussiとJim Quinlanによる診断で、そのボードのDevice TreeはSCMIノードを、動作するSMCトランスポート*と*、後方互換性のためだけに残された、恒久的に`status = "disabled"`のメールボックスノードへのレガシーな`mboxes` phandleの両方を持つ形で宣言していることが分かった。fw_devlinkはその古い参照を忠実にprobeの強い依存関係に変換した。無効化されたメールボックスデバイスにドライバがバインドされることはないため、SCMIプロバイダと、UARTのクロックを含むその下流のすべてが永遠に延期された。

今回の解決は、明示的にカーネルコアの変更ではなかった。Saravana Kannanはスレッドの中で利用可能な回避策に順位を付けた:「1. Fix the DT sent to the kernel. 2. If deferred_probe_timeout=1 doesn't break anything else, use that... 3. Geert's early boot quirk suggestion. 4. fw_devlink=permissive (least preferred because this might mask issues with fw_devlink=on in your future changes). Changing the SCMI driver itself won't help fw_devlink.」（訳: 1. カーネルに渡すDTを修正する。2. deferred_probe_timeout=1が他に何も壊さないなら、それを使う... 3. Geertのブート初期のquirkの提案。4. fw_devlink=permissive（将来の変更でfw_devlink=onに関する問題を覆い隠すかもしれないので最も好ましくない）。SCMIドライバ自体を変更してもfw_devlinkの助けにはならない。）Fainelliは、ボードのDTから古い`mboxes` phandleを削除すると直ることを確認した。この場合のfw_devlinkは、ファームウェア記述自体が主張している依存関係を正しく強制していたのである。バグはDTにあり、古いpermissiveのデフォルトは、それを露呈させるほど厳格ではなかったにすぎない。Broadcom STBのSCMIノードにおけるSMCとメールボックスの同じ曖昧さは、数年後に別のきっかけ、つまり無関係な2024年のSCMIトランスポートのリファクタリングのもとで再浮上し、最終的には`db8f0b808886`（"firmware: arm_scmi: Give SMC transport precedence over mailbox"、2024年10月）によってドライバ自体で対処された。そのコミット自身の`Fixes:`タグは2021年のfw_devlinkの顛末ではなく2024年のリファクタリングを指しているため、この2021年の特定の出来事に対する遅れた修正というより、同じ種類のファームウェア記述の曖昧さが新しい条件のもとで再浮上したものと読むのが最も適切である。

## 教訓 {#what-it-taught-us}

**プラットフォーム全体でprobe順序の意味を変えるデフォルトを切り替えると、レビューでは事前に捕捉できない個々の隅のケースが長い裾野として露呈する。ひとつのハードウェアプラットフォーム、ひとつのバス種別、ひとつのレガシーな互換性文字列ずつ**。R-Carのearly-initcallによる電源コントローラ、i.MXのAVIC、Broadcomの古いメールボックスphandleは、同じ特定の隙間が無関係に現れた3つの事例である。グラフ上は存在するはずなのに、実際には`struct device`を一切登録しないデバイスである。Samsung Exynosの`interrupt-map`の修正は、同じ展開の中の別種の隙間だった。fw_devlink自身がDTの間接参照を通じて割り込みサプライヤを解決する方法におけるパースのバグであり、デバイス欠落のケースではまったくない。仕組みは違っても教訓は同じである。システム全体のデフォルト変更は、レビューのプロセスがたまたま試さなかったものを表面化させ、しかもそれを1プラットフォームずつ表面化させる。

**リバートコミットが述べる対象リリースは計画であって、何が出荷されるかについての事実ではない。誰かがそれを書いた日付ではなく、変更が実際に入ったタグで検証すること**。「5.12向けにこのデフォルトをリバートする」という文言は、そのコミットメッセージだけで修正を追っていた人には、5.12が新しいデフォルトを含むという保証のように読めた。マージウィンドウのタイミングがそれを覆した。再挑戦のコミットは、メインラインにpullされるまで数週間サブシステムメンテナのツリーに存在しており、それが実際に初めて出荷されたリリースは、リバート自身の文言ではなく、そのpullのタイミングによって決まった。

!!! warning "注意すべきパターン"
    カーネルの変更が、順序付けや依存関係の強制を変えるシステム全体のデフォルトを切り替えるとき、実際のテストの組み合わせは、レビュー中に試されたプラットフォームではなく「現在アクティブにテストされているすべてのプラットフォーム」であり、切り替えの後に少しずつ発見されていくものと考えること。また、特定の修正やデフォルト変更が「出荷された」かどうかを追跡する際は、変更を行ったコミットの日付ではなく、それが祖先となっている実際のタグ付きリリースで確認すること。リリースのかなり前に作成され、メンテナのツリーにコミットまでされた修正でも、そのリリースのマージウィンドウを完全に逃すことがありうる。

## 関連項目 {#see-also}

- [Linuxデバイスモデル](../device-model.md) — `struct device`、バス、そしてfw_devlinkが土台とするprobe／リンクの仕組み
- [Device Tree](../device-tree.md) — fw_devlinkが依存関係を発見するためにパースするファームウェア記述フォーマット
- [Deferred Probeの起源](deferred-probe-origin.md) — fw_devlinkの`on`モードが自動的に駆動する`-EPROBE_DEFER`の再試行の仕組み
- [ドライバのデタッチをデッドロックさせたuevent_show()の修正](uevent-show-deadlock.md) — 同じサブシステムで起きた、はるかに短い時間スケールの別のドライバコアのリグレッション

## 外部参考資料 {#external-references}

- [git.kernel.org: c78c31b374a6](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c78c31b374a68be79cb4a03ef5b6c187f034e903) — "Revert 'Revert...'"、それ以前の`fw_devlink=permissive`デフォルトの2020年の再有効化
- [git.kernel.org: e590474768f1](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e590474768f1cc04852190b61dec692411b22e2a) — "driver core: Set fw_devlink=on by default"、最初の有効化
- [git.kernel.org: 3e4c982f1ce7](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3e4c982f1ce75faf5314477b8da296d2d00919df) — "Revert 'driver core: Set fw_devlink=on by default'"
- [git.kernel.org: ea718c699055](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ea718c699055c8566eb64432388a04974c43b2ea) — "Revert 'Revert...'"、Take II
- [git.kernel.org: 2dfc564bda4a](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2dfc564bda4a31bc4439315448bd4da5182cb397) — R-Car SYSCのOF_POPULATED修正
- [git.kernel.org: e2c1b0ff38c9](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e2c1b0ff38c961d49ce34efda48fa45eb1cb5f19) — i.MX AVICのIRQCHIP_DECLARE修正
- [git.kernel.org: f265f06af194](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f265f06af1948c90007c78fe9f2fa93d6cea8800) — fw_devlinkの`interrupt-map`修正、Exynosでテスト済み
- [git.kernel.org: db8f0b808886](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=db8f0b8088865150e4c9a8b8ffc9abdfd58bc4f7) — 2024年のBroadcom SCMIのSMC優先修正。同じDTの曖昧さが数年後に再浮上したもの
- [lore.kernel.org: 最初の有効化パッチのスレッド](https://lore.kernel.org/r/20201218031703.3053753-6-saravanak@google.com/) — R-Carとnios2のリグレッション報告
- [lore.kernel.org: Take IIのスレッド](https://lore.kernel.org/r/20210302211133.2244281-4-saravanak@google.com/) — ARCH_BRCMSTB/SCMIのリグレッションと、DTのバグとしての解決
