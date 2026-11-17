# 遅延 probe の起源 {#the-deferred-probe-origin-story}

> CVE ではない — OLPC ラップトップのカメラドライバは、カーネルに直接組み込んでコンパイルすると初期化に失敗したが、モジュールとしてなら問題なく動作した。その「カメラ」は実際には3つの別々のデバイスであり、カーネルの静的な組み込みのリンク順序では、それらの probe 順序を保証できなかったからである — 修正は2011年に、ドライバが `-EAGAIN` を返すものとして始まり、何度かの改訂を経て、カーネルが今日も使っている専用の `-EPROBE_DEFER` 機構に落ち着いた

マージ
:   Linux 3.4、2012年5月20日リリース

作者
:   Grant Likely（パッチ作者）。Greg Kroah-Hartman（コミッタ、ドライバコアのメンテナ）

起源
:   OLPC XO ラップトップのハードウェアで実際に起きたブート失敗。2011年7月に LWN で Jonathan Corbet が初めて公に説明した

CVE ではない

*[事件簿：デバイスドライバ](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

（後からモジュールとしてロードされるのではなく）カーネルに直接組み込んでコンパイルされたドライバは、ブート中に、カーネルのビルドが静的に決める順序で初期化される。まず initcall レベル（`arch_initcall`、`subsys_initcall`、`device_initcall` など）、次にそのレベル内でのリンク順序である。この順序づけは、ある組み込みドライバのデバイスが、別の組み込みドライバのデバイスが先に準備できていることに実際に依存しているかどうかを、何も知らないし気にもかけない — それは純粋に、各 `initcall` マクロがカーネルイメージのどこに置かれているかの関数であり、ブート時に実際のハードウェアが probe されるはるか前の、ビルド時に決まる。後からカーネルモジュールとしてロードされるドライバには、そのような制約はない。管理者や udev がそれをロードする頃には、依存しているものは通常すでに初期化を終えている。

## きっかけ {#the-trigger}

LWN の Jonathan Corbet は、進行中のパッチ提案を紹介する2011年7月7日の記事で、まさにこの種の具体的な失敗を説明した。"The developers working on the initial OLPC laptop ran into an interesting problem: the camera driver would fail to initialize if it was built into the kernel, but it worked just fine if built as a module."（訳: 初代 OLPC ラップトップに取り組んでいた開発者たちは興味深い問題に遭遇した。カメラドライバは、カーネルに組み込んでビルドすると初期化に失敗するが、モジュールとしてビルドすれば問題なく動作した）。その理由は、ハードウェアの観点から「カメラ」が実際に何であったかにさかのぼる。"The problem with the camera driver is a result of the fact that the 'camera' is, in reality, three devices working in concert: a DMA bridge, a sensor, and an I2C bus connecting the two."（訳: カメラドライバの問題は、「カメラ」が実際には協調して動く3つのデバイス — DMA ブリッジ、センサー、そして両者をつなぐ I2C バス — であることに起因する）。カーネルに直接組み込んだ場合、3つのドライバの `probe()` 関数がこの依存関係を尊重する順序で実行される保証はなかった。"If all of the drivers are built into the kernel, the bridge driver's probe() function may be called first; there will be no sensor, so everything fails."（訳: すべてのドライバがカーネルに組み込まれていると、ブリッジドライバの probe() 関数が最初に呼ばれることがある。そのときセンサーは存在しないので、すべてが失敗する）。モジュールとしてなら、同じ3つのドライバが、それぞれの `probe()` が実行される時点で依存関係がすでに満たされている順序でたまたまロードされた — 何かがその順序を強制していたからではなく、手動で、あるいは modprobe 自身の依存関係解決によって後からロードすることで、組み込みのリンク順序の問題を完全に回避していたからである。

## 観測された挙動 {#observed-behavior}

Corbet が2011年に説明した、Grant Likely のパッチによる仕組みは、汎用の再試行機能だった。"drivers which are unable to initialize their devices as the result of missing resources can request that the operation be retried at some point in the future. That request is a simple matter of returning **-EAGAIN** from the probe() function."（訳: リソースが欠けているためにデバイスを初期化できないドライバは、将来のある時点でその操作を再試行するよう要求できる。その要求は、単に probe() 関数から **-EAGAIN** を返すだけでよい）。ドライバコアは、これを要求したドライバを追跡する。"The driver core maintains a simple linked list of drivers that have requested this sort of deferral."（訳: ドライバコアは、この種の遅延を要求したドライバの単純な連結リストを保持する）。この初期版はカーネルの通常の `-EAGAIN` errno を使っていた — ノンブロッキング I/O からリソース競合まで、数十の無関係な文脈で、カーネル全体と POSIX を通じて「この操作を後でもう一度試せ」を意味するのに使われているのと同じ値である。この選択は、最終的にマージされた版までは生き残らなかった。

## なぜ起きたか {#why-it-happened}

根本的な問題は、OLPC の特定のカメラハードウェアをはるかに超えて一般化する。カーネルのビルド時の initcall 順序には、デバイス間の*機能的な*依存関係を表現する手段がない — ビルドシステムがたまたまオブジェクトをリンクした順序をエンコードしているだけである。完成した仕組みを説明する、後のツリー内のコメントは、無関係な例を使って一般的なケースを平易に述べている。"Sometimes driver probe order matters, but the kernel doesn't always have dependency information which means some drivers will get probed before a resource it depends on is available. For example, an SDHCI driver may first need a GPIO line from an i2c GPIO controller before it can be initialized. If a required resource is not available yet, a driver can request probing to be deferred by returning -EPROBE_DEFER from its probe hook."（訳: ドライバの probe 順序が重要になることがあるが、カーネルは常に依存関係の情報を持っているわけではなく、そのため一部のドライバは、依存するリソースが利用可能になる前に probe される。たとえば SDHCI ドライバは、初期化できるようになる前に、まず i2c GPIO コントローラからの GPIO ラインを必要とすることがある。必要なリソースがまだ利用できない場合、ドライバは probe フックから -EPROBE_DEFER を返すことで、probe の遅延を要求できる）。OLPC の3デバイス構成のカメラは、Corbet の記事がより広い読者にパッチを紹介するために使った具体的なケースにすぎない。Grant Likely の提案は、特に OLPC の報告に応えて書かれたのではなく、すでに出回っていた — Corbet の記事の3日前に v2 が送られており、それは順序の問題一般に対処するもので、パッチ自体には OLPC への言及はなかった。マージされたコードが自身の説明に用いている SDHCI/GPIO の例は、特定の1つのきっかけとなる報告とは無関係に、同じ形の問題がまったく異なるハードウェアで繰り返し現れることを示している。

## 解決 {#resolution}

この仕組みは、Corbet の2011年7月の記事からマージまでの間に何度か改訂を経た。最終的なコミットメッセージに埋め込まれたその v4 の変更履歴には、意図的な終盤の変更が記録されている。"Change -EAGAIN to -EPROBE_DEFER for drivers to trigger deferral"（訳: ドライバが遅延を起動するための値を -EAGAIN から -EPROBE_DEFER に変更）— おそらく、無関係な意味ですでに `-EAGAIN` を使っている他の多くのカーネルサブシステムと区別される、専用の曖昧さのないシグナルを遅延要求に与えるためである。`d1c3414c2a9d`（"drivercore: Add driver probe deferral mechanism"、Grant Likely、2012年3月5日作成、2012年3月8日に Greg Kroah-Hartman がコミット）は、今も使われている仕組みとしてこれを入れた。`include/linux/errno.h` には `#define EPROBE_DEFER 517 /* Driver requests probe retry */` が加わり、`drivers/base/dd.c` には `driver_deferred_probe_add()`、`driver_deferred_probe_del()`、`driver_deferred_probe_trigger()`、`deferred_probe_work_func()` が加わった。これらは2つのリスト — `deferred_probe_pending_list` と `deferred_probe_active_list` — に支えられ、`deferred_probe_mutex` で保護され、その場で再試行するのではなく専用の workqueue によって処理される。このコミットは2012年5月20日リリースの Linux 3.4 で初めて出荷された。

この仕組みは、後にファームウェア由来の自動的な probe 順序づけが直接その上に構築される再試行プリミティブでもある。[ほぼ10年後に追加された](fw-devlink-boot-breaking.md) fw_devlink の `DL_FLAG_AUTOPROBE_CONSUMER` モードは、consumer が宣言した supplier がまだ probe されていないときにはいつでも、ドライバコアが `-EPROBE_DEFER` に相当するものを自動的に発行することで機能する — 元々はドライバ自身が `-EAGAIN`/`-EPROBE_DEFER` を1つずつ返して呼び出さなければならなかった仕組みを、ドライバコアがすでに持っている依存関係のデータから、ドライバに代わって起動できるものに変えたのである。

## 得られた教訓 {#what-it-taught-us}

**他の変数を何も変えずに、組み込みとモジュールとでドライバの振る舞いが異なるなら、それはドライバ自身のロジックではなく、probe の*順序*に関する何かが荷重を支えていることを示す強いシグナルである。** カメラドライバのコード自体は、どちらのビルドでも間違っていなかった。組み込みのリンク順序が、モジュールのロードがたまたま満たしていた順序の制約を、たまたま満たしていなかっただけである。

**再試行の要求には、既存のものを流用するのではなく、専用のシグナルを与えるべきである。** この仕組み自身の改訂履歴は、それが `-EAGAIN` — 明白ですでに利用可能な「もう一度試せ」の errno — から始まり、1つのプラットフォームの回避策ではなく汎用のドライバコアの機能になった時点で、マージ前に専用の `-EPROBE_DEFER` に移ったことを示している。

!!! warning "注意すべきパターン"
    ドライバがモジュールとしては正しく初期化されるのに、カーネルに直接組み込んでコンパイルすると失敗する（あるいはその逆）なら、ドライバ自体のロジックのバグではなく、動作している構成が設計によってではなく偶然に満たしている暗黙の順序依存を疑うこと。`-EPROBE_DEFER` は、そうなったときにドライバが即座に失敗するのではなく「まだだ」と言えるようにするために存在する。

## 関連項目 {#see-also}

- [Linux デバイスモデル](../device-model.md) — `struct device`、`probe()`、そしてドライバ登録の中で遅延 probe が占める位置
- [fw_devlink=on：定着するまでに2度ブートを壊したデフォルト](fw-devlink-boot-breaking.md) — このページで説明した仕組みを、ほぼ10年後に自動化しデフォルトで有効にしたもの。その展開は、定着するまでに2度ブートを壊した（そして1度リバートされた）
- [デバイスツリー](../device-tree.md) — 数十年後、ドライバが自分で遅延を要求することだけに頼るのではなく、カーネルが依存関係グラフを導出できるようにしたファームウェアの記述形式

## 外部参考文献 {#external-references}

- [LWN: Deferred driver probing](https://lwn.net/Articles/450460/) — Jonathan Corbet、2011年7月7日。元の OLPC カメラの経緯と、`-EAGAIN` ベースの提案
- [LWN: Grant Likely's v2 patch, "drivercore: Add driver probe deferral mechanism"](https://lwn.net/Articles/450178/) — 2011年7月4日、Corbet の記事の3日前。OLPC への言及はない
- [LWN: Grant Likely's patch submission, "drivercore: Add driver probe deferral mechanism"](https://lwn.net/Articles/485194/) — LWN がアーカイブした lkml のパッチメール、2012年3月5日。`d1c3414c2a9d` としてマージされたものとテキスト上同じ内容
- [git.kernel.org: d1c3414c2a9d](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d1c3414c2a9d10ef7f0f7665f5d2947cd088c093) — "drivercore: Add driver probe deferral mechanism"、マージされたコミット。Linux 3.4 で初めて出荷
