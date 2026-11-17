# fs_contextのレガシーパラメータのオーバーフロー {#the-fs_context-legacy-parameter-overflow}

> CVE-2022-0185 — 符号なし算術で`PAGE_SIZE - 2 - size`と書かれた境界チェックにより、`unshare -Urm`と`fsconfig()`呼び出しのループで4 KBのslabオブジェクトの終端を越えて書き込めるようになり、Googleの強化されたKubernetes CTFクラスタを31,337ドルの報奨金に変えた

公開日
:   2022年1月18日

報告者
:   CTFチームCrusaders of Rust — Alec Petridis、Hrvoje Mišetić、Isaac Badipe、Jamie Hill-Daniel、Philip Papurt、William Liu。これに先立ち、syzbot（2021年12月31日、Androidツリー上）とStarLabsがそれぞれ独立に発見していた

CVSS
:   8.4 HIGH（`CVSS:3.1/AV:L/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`、NVDの主評価）

バグの存在期間
:   5.1-rc1（2019年）以降

修正
:   5.4.173、5.10.93、5.15.16、5.16.2（いずれも2022年1月20日）、メインライン5.17-rc1

エクスプロイトツール
:   あり — 独立した2つの公開エクスプロイト。Exploit-DBやMetasploitのエントリはない

実際の悪用
:   あり — 2024年8月21日にCISAのKEVカタログに追加（[NVD](https://nvd.nist.gov/vuln/detail/CVE-2022-0185)による）

*[事件簿: LinuxセキュリティのバグとCVE](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

Linux 5.1は、David Howellsのファイルシステムコンテキストの作業を、[`3e1aeb00e6d1`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3e1aeb00e6d132efc151dacc062b38269bc9eccc)（"vfs: Implement a filesystem superblock creation/configuration context"）としてマージして出荷した。これが`fs_context`の基盤と`legacy_parse_param()`である。続いてLinux 5.2は、それを実際にユーザー空間から駆動する新しいマウントAPIのシステムコール、`fsopen()`、`fsconfig()`、`fsmount()`、`move_mount()`を追加した。`mount()`の時点で1つの不透明なオプション文字列をカーネルに渡す代わりに、ユーザー空間は*コンテキスト*を開き、パラメータを1つずつ、それぞれ別々のシステムコールとして渡すようになった。

新しいインターフェースに一度に変換されたファイルシステムはほとんどなかった。レガシー経路が7.0で削除されるまで、`alloc_fs_context()`にはフォールバックについてのHowellsの元のコメントが残っていた:

```c
/* TODO: すべてのファイルシステムが無条件にこれをサポートするようにする */
init_fs_context = fc->fs_type->init_fs_context;
if (!init_fs_context)
    init_fs_context = legacy_init_fs_context;
```

`init_fs_context`を実装していないすべてのファイルシステム（5.1から5.17で変換されるまでのext4を含む）では、パラメータは`fs/fs_context.c`の`legacy_parse_param()`に流れ込んだ。その役割は、古い`->mount()`フックが依然として期待するカンマ区切りのオプション文字列を再構築することである。この関数はその文字列を単一の`kmalloc(PAGE_SIZE)`バッファに蓄積し、書き込んだ量を`ctx->data_size`で追跡していた:

```c
static int legacy_parse_param(struct fs_context *fc, struct fs_parameter *param)
{
	struct legacy_fs_context *ctx = fc->fs_private;
	unsigned int size = ctx->data_size;
	size_t len = 0;
	...
	if (len > PAGE_SIZE - 2 - size)
		return invalf(fc, "VFS: Legacy: Cumulative options too large");
	...
	if (!ctx->legacy_data) {
		ctx->legacy_data = kmalloc(PAGE_SIZE, GFP_KERNEL);
		...
	}
	ctx->legacy_data[size++] = ',';
	len = strlen(param->key);
	memcpy(ctx->legacy_data + size, param->key, len);
	size += len;
	if (param->type == fs_value_is_string) {
		ctx->legacy_data[size++] = '=';
		memcpy(ctx->legacy_data + size, param->string, param->size);
		size += param->size;
	}
	ctx->legacy_data[size] = '\0';
	ctx->data_size = size;
```

`- 2`は区切りのカンマと末尾のNULのための領域を確保している（文字列値のパラメータでは、`=`はすでに`len`自体に含まれている）。`size`は`unsigned int`、`PAGE_SIZE`はunsigned long、`len`は`size_t`である。減算が実行される前にすべてのオペランドがunsigned longに変換されるため、結果は決して負にならない。

## きっかけ {#the-trigger}

追記が成功するたびに`size`は`len + 1`だけ進み、チェックは`len <= PAGE_SIZE - 2 - size`を許可する。この算術を限界まで進めると、`size`はちょうど**4095**に到達することが許される。バッファより1バイト少なく、NULは最後の有効なバイトに置かれる。これは、チェック自体が生み出す、正当で範囲内の状態である。

次の呼び出しでは、`PAGE_SIZE - 2 - size`は`4096 - 2 - 4095`と評価される。符号なし算術では、これは`-1`ではなく`ULONG_MAX`である。どんな`len`もそれを超えられないので、ガードは決して発火しない。`legacy_data[4095]`へのカンマの書き込みはまだ範囲内に収まり、その次に書き込まれるバイト、オフセット4096が最初の範囲外書き込みになる。`size`は増え続けるので、以降の`fsconfig()`呼び出しはそのたびに終端をさらに越えて書き込む。繰り返し可能で、位置も内容も攻撃者が決められるヒープ書き込みであり、1回の呼び出しあたりの上限は、`fs/fsopen.c`の`strndup_user(_key, 256)`と`strndup_user(_value, 256)`によるキーと値それぞれの長さの制限だけである。

ちょうど4095に着地させるのに必要なのは運ではなく算術である。発見者たちのPoCは空のキーと33バイトの値を使うので、各呼び出しは`1 + 0 + 1 + 33 = 35`バイトを追加し、117回の呼び出しで`data_size`はぴったり`35 × 117 = 4095`になる:

```c
fd = fsopen("ext4", 0);
for (int i = 0; i < 117; i++)
	fsconfig(fd, FSCONFIG_SET_STRING, "\x00", pat, 0);
/* 118回目の呼び出しが範囲外に書き込む */
```

そもそも`fsopen()`に到達するには`CAP_SYS_ADMIN`が必要である。ただしそれは、呼び出し元のマウント名前空間を所有するユーザー名前空間の中だけでよい（`ns_capable(current->nsproxy->mnt_ns->user_ns, CAP_SYS_ADMIN)`）。[oss-securityの告知](https://www.openwall.com/lists/oss-security/2022/01/18/7)が述べたとおりである:「An unprivileged user can use `unshare(CLONE_NEWNS|CLONE_NEWUSER)` to enter a namespace with the CAP_SYS_ADMIN permission, and then proceed with exploitation to root the system.」（訳: 非特権ユーザーは`unshare(CLONE_NEWNS|CLONE_NEWUSER)`を使ってCAP_SYS_ADMIN権限を持つ名前空間に入り、そこから悪用を進めてシステムのroot権限を奪取できる。）非特権ユーザー名前空間を提供するディストリビューション、つまりUbuntu、Debian、そしてそれに依存するすべてのコンテナランタイムでは、実質的な特権要件はない。ローカルのバグであるにもかかわらずNVDがこれを`PR:N`と評価しているのはそのためである。

## 観測された挙動 {#observed-behavior}

カーネルから見える症状は、`kmalloc-4k`キャッシュにおけるKASANのslab-out-of-bounds書き込みであり、公開につながったCrusaders of Rustのファジングで観測されたものは次のとおりである:

```
BUG: KASAN: slab-out-of-bounds in legacy_parse_param+0x450/0x640 fs/fs_context.c:569
Write of size 1 at addr ffff88802d7d9000 by task syz-executor.12/386100
...
 vfs_parse_fs_param+0x1fd/0x390 fs/fs_context.c:146
 vfs_fsconfig_locked+0x177/0x340 fs/fsopen.c:265
 __x64_sys_fsconfig+0x6a6/0x7a0 fs/fsopen.c:314
The buggy address is located 0 bytes to the right of
 4096-byte region [ffff88802d7d8000, ffff88802d7d9000)
```

これをクラッシュではなく権限昇格のバグにしたのは、その書き込みがどこに着地するかである。[彼らの解説記事](https://www.willsroot.io/2022/01/cve-2022-0185.html)で、William LiuとCrusaders of Rustは、レガシーデータの割り当ての直後に`msg_msg`オブジェクトを配置し、そのサイズフィールドを破壊し、`MSG_COPY`を使って範囲外を読み出す方法を説明している。これにより`open("/proc/self/stat")`でスプレーした`seq_operations`のポインタを漏洩させ、KASLRを破る。Ubuntu向けエクスプロイトでの任意書き込みのために、彼らは`usercopy`を途中で停止させる必要があった。5.11以降、非特権の`userfaultfd`はデフォルトで無効になっているため、代わりに自前のFUSEファイルシステムを使い、その読み込みハンドラを単にハングさせた。そして最後に`modprobe_path`を上書きした。GoogleのkCTFコンテナは`/dev`が削ぎ落とされていてFUSEも`userfaultfd`もなかったため、脱出用のエクスプロイトは代わりに`pipe_buffer`を狙った。破壊した`msg_msg`をリストから外して`pipe_buffer`のopsポインタに直接書き込もうとする試みは失敗した。`CONFIG_DEBUG_LIST`が有効で、`__list_del_entry_valid()`が偽造したリンクを拒否したからである。しかし`do_msgrcv()`はそのチェックが失敗しても解放処理を行うため、任意解放のプリミティブが得られた。標的の`pipe_buffer`を含むチャンクを意図的にずらしたアドレスで解放し、その上に`msg_msg`オブジェクトをスプレーすることで、opsポインタをスタックピボットのガジェットに向け、`commit_creds(prepare_kernel_cred(NULL))`と`switch_task_namespaces(find_task_by_vpid(1), init_nsproxy)`をROPで実行できた。これによりコンテナの`init`をホストの名前空間に移し、最後の`setns()`で脱出を完了した。

解説記事は、記録しておく価値のある皮肉を指摘している。悪用対策のハードニング変更が、エクスプロイトを*より*確実にしたのである。5.7以降、SLUBはフリーリストのポインタを各オブジェクトの先頭ではなく中央に格納しているため、次のチャンクの最初の`0x30`バイト、まさに`msg_msg`が気にする領域を通る文字列状のオーバーフローは、もはやslabのメタデータを破壊しない。そのため、成功するまで新しい`fsopen()`コンテキストでオーバーフローを繰り返すことができた。

彼らはこれを使ってGoogleの強化されたkCTF Kubernetesコンテナから脱出し、別のpodのフラグを読み取って、31,337ドルの報奨金を得た。最高額から減額されたのは、n0psledbyteと名乗る研究者が同じバグを先に見つけていながら、適切に報告も公開もしていなかったためであり、Crusaders of Rustはその重複を自分たちの報告の後で初めて知った。1週間後、Alejandro Guerreroは、カーネル5.11のUbuntu 21.04を標的とする[独立したエクスプロイトを](https://www.openwall.com/lists/oss-security/2022/01/25/14)oss-securityで公開した。その2年半後、CISAはCVE-2022-0185を既知の悪用された脆弱性（KEV）カタログに追加した（2024年8月21日）。これは[NVD](https://nvd.nist.gov/vuln/detail/CVE-2022-0185)で確認でき、そのSSVCデータは悪用状況を**active**と記録している。Exploit-DBのエントリもMetasploitモジュールもない。2022年1月19日に作成され、動作するエクスプロイトが1月25日に公開された[Crusaders of Rustのリポジトリ](https://github.com/Crusaders-of-Rust/CVE-2022-0185)がエクスプロイトの配布経路であり、NVDもそのようにタグ付けしている。

## なぜ起きたか {#why-it-happened}

チェックが欠けていたわけではない。正しいチェックを、間違った方向に計算していたのである。`len > PAGE_SIZE - 2 - size`は、まず残りの空き領域を計算することで「このパラメータは残りの領域に収まるか？」を問う。そして*残りを計算すること*こそが、型が符号なしの場合に誤りうる手順である。`size + len + 2 > PAGE_SIZE`は、アンダーフローしうる値を一度も作らずに同じ問いを問う。

これを教科書的なアンダーフロー以上のものにしているのは、ガード自身の成功条件が、`size`をガードが壊れる値まで進めてしまうことである。バグを仕込むのに不正な入力は必要ない。チェックがそれぞれ正しく受け入れる、117回の完全に正当な`fsconfig()`呼び出しが、118回目の呼び出しがチェックされない唯一の状態にコンテキストを残す。チェックが実際に維持していた不変条件（`size <= PAGE_SIZE - 1`）と、チェック自身の算術が必要としていた不変条件（`size <= PAGE_SIZE - 2`）は、ちょうど到達可能な1ステップ分ずれていた。

影響範囲も「1つのレガシーなコードパス」よりはるかに広かった。`legacy_parse_param()`は2018年に互換性のためのシムとして書かれたもので、当時それに到達する唯一の方法は、一枚岩のオプション文字列を伴う`mount()`だった。5.2のマウントAPIのシステムコールは、ユーザー空間にそのシムをパラメータ1つずつ、蓄積される長さをバイト単位で制御しながら駆動する手段を与えた。元のコードは、そのような粒度で入力を与えられることを想定して設計されていなかった。誰も気づかないまま3年が過ぎ、その後2つのファザーが6日違いでそれを見つけた。Liu自身がコメントで述べたとおりである:「It was interesting to see how both of our fuzzer and syzbot's found this bug a few days apart in 2022 while the bug existed since 2019 — perhaps some recent change in either the kernel or syzkaller made it easier to trigger.」（訳: バグは2019年から存在していたのに、2022年に私たちのファザーとsyzbotのファザーの両方が数日違いでこのバグを見つけたのは興味深かった。カーネルかsyzkallerのどちらかの最近の変更が、引き起こしやすくしたのかもしれない。）

ただし「誰も気づかなかった」というのは正確ではない。CVEの3年半前に、ある人物がまさにこの関数の算術を見て、同じ短い関数の中の1行を修正している。2018年8月、`legacy_parse_param()`がまだDavid Howellsのブランチで開発中だった頃、Andrei Vaginは、欠陥のあるガードの数文下、`size`を進める最後の行に対する[1行の修正を投稿した](https://lore.kernel.org/linux-fsdevel/20180802072810.3126-1-avagin@openvz.org/)。`size += len;`は`size += param->size;`であるべきだった。`len`は数行前に*キー*の長さに再代入されており、もはやコピーされる値を表していなかったからである。Howellsは同じ日にそれを取り込んだ。そのパッチはアンダーフローが依存する累積値に触れていながら、アンダーフローを見逃した。これは、このバグがすでに教えている教訓の2つ目の例である。長さ計算の1行を正しくレビューしても、それを制限するチェックが健全かどうかについては何も分からない。

Crusaders of Rustより早い別の発見こそが、居心地の悪い部分である。[syzbotは2021年12月31日にこれを報告していた](https://groups.google.com/g/syzkaller-android-bugs/c/hWixpJ22kc8/m/MrIgyhQvEAAJ)。件名は"KASAN: slab-out-of-bounds Write in legacy_parse_param"、バグIDは`02b637bea12b3944b0b9`で、同じ関数における別のクラッシュ（CoRの実行での5.14上の`fs/fs_context.c:569`ではなく、`android12-5.4`ツリー上の`fs/fs_context.c:603`）であり、syzbotは15分後に同じスレッドにsyzとCの再現プログラムを投稿していた。それはVFSのリストではなく`syzkaller-android-bugs`に投稿され、CVEの告知から8日後の2022年1月26日にLee Jonesがすでに公開されていたアップストリームの修正をスレッドに投稿するまで、人間は誰も返信しなかった。

## 解決 {#resolution}

2022年1月18日にJamie Hill-Danielが作成した[`722d94847de2`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=722d94847de29310e8aa03fcbdb41fc92c521756)（"vfs: fs_context: fix up param length parsing in legacy_parse_param"）は、1行の変更である:

```c
-	if (len > PAGE_SIZE - 2 - size)
+	if (size + len + 2 > PAGE_SIZE)
		return invalf(fc, "VFS: Legacy: Cumulative options too large");
```

そのコミットメッセージは規則をはっきりと述べている:「The 'PAGE_SIZE - 2 - size' calculation in legacy_parse_param() is an unsigned type so a large value of 'size' results in a high positive value instead of a negative value as expected. Fix this by getting rid of the subtraction.」（訳: legacy_parse_param()の'PAGE_SIZE - 2 - size'の計算は符号なし型なので、'size'の値が大きいと、期待どおりの負の値ではなく大きな正の値になる。減算をなくすことでこれを修正する。）

興味深いのはトレーラーである。このパッチには、Hill-DanielとWilliam Liuの`Signed-off-by`、Salvatore Bonaccorso（Debian）とThadeu Lima de Souza Cascardo（Canonical）の`Tested-by`、Dan CarpenterとAl Viroの両者の`Acked-by`が付き、その後にGreg Kroah-HartmanとLinus Torvaldsのサインオフが続く。これは公開メーリングリストのレビューの連鎖ではなく、ディストリビューションのセキュリティレビューの連鎖であり、それははっきり表れている。**このパッチには、マージ前のアップストリームへの投稿もレビュースレッドも存在しない**。コミットには`Link:`トレーラーがなく、取り込まれる前に設計の議論のためにlinux-fsdevelやLKMLに投稿されたことは一度もない。バグは非公開で報告され（Red Hatの[Bugzilla 2040358](https://bugzilla.redhat.com/show_bug.cgi?id=2040358)は2022年1月13日に起票された）、エンバーゴ下でレビューされ、公開の当日にそのままメインラインに投入された。公開の議論は別の場所で行われた。1月18日の[oss-securityの告知](https://www.openwall.com/lists/oss-security/2022/01/18/7)、同じ時間帯にCVE-2022-0185を割り当てたJohn Haxbyの返信、そして1週間後のGuerreroのエクスプロイト投稿である。修正そのものが後にメーリングリストに現れる唯一の場所は下流である。Greg Kroah-Hartmanの5.16.2-rc1のstableレビューシリーズに含まれる28パッチのうちの1つとしてである。この修正の*設計*の経緯をloreだけから再構成しようとする人は、そのstableレビューの投稿と、兄弟のバックポートしか見つけられない。

それらのバックポートは速かった。5.4.173、5.10.93、5.15.16、5.16.2はすべて、公開の2日後の2022年1月20日にリリースされ、4つすべてに修正が含まれていた。すぐに再起動できない運用者には、本当の緩和策が1つあった。このバグは`CAP_SYS_ADMIN`を与える名前空間なしでは到達できないからである:

```bash
# Debian/Ubuntu系のノブ: 0にすると非特権ユーザー名前空間の作成をブロックする
sysctl kernel.unprivileged_userns_clone
# 4.9以降アップストリームにある: 任意のユーザーが作成できるuserns数の固い上限
sysctl user.max_user_namespaces
```

どちらもrootlessコンテナや、正当にユーザー名前空間をunshareするその他のものを壊す。それこそが、コンテナプラットフォームが2022年の初めに議論を重ねたトレードオフである。

## 教訓 {#what-it-taught-us}

**境界チェックは差ではなく和として書くこと**。`a + b > LIMIT`と`b > LIMIT - a`は数学的には等価だが、Cでは等価ではない。左辺で定数から攻撃者が影響を与えられる量を引くチェックはすべて、大きな入力1つで何もしないものになりうる。そして真実が失われるのが*減算*である以上、*比較*にどれだけ注意を払っても役に立たない。

**自身の前提条件の破れに到達することを許すガードはガードではない**。このチェックは、`size`がまさにチェックが機能しなくなる値まで増えることを許していた。この形のバグがレビューをすり抜けるのは、個々のステップがすべて正しく見えるからである。失敗は連続した操作の中にしか存在せず、レビュアーが読むのは関数であって、操作の連続ではない。

**新しいシステムコールは、古いコードを1行も変えずに武器に変えることができる**。`legacy_parse_param()`は5.1でも5.2でも同じように正しかった。変わったのは、`fsconfig()`によって、攻撃者が1つの文字列を渡す代わりに35バイトずつそれを駆動できるようになったことである。既存のサブシステムにより細かい粒度のインターフェースを追加すると、その背後にある互換性のためのシムは、作者が想定しなかった方法で到達可能になる。

**「CAP_SYS_ADMINが必要」は2013年頃に緩和要因ではなくなった**。NVDはこのバグを`PR:N`（必要な特権: なし）と評価した。Linuxで最も強力なケイパビリティで明示的にゲートされたコードパスであるにもかかわらずである。デフォルトのUbuntuインストールでは、`unshare -Urm`が任意のユーザーにそのケイパビリティを与えるからである。ケイパビリティのチェックを信頼境界として読む深刻度のトリアージは、この種のバグ全体を系統的に過小評価することになる。

!!! warning "注意すべきパターン"
    自分のコードで、いずれかのオペランドが符号なしで攻撃者の影響を受ける`LIMIT - x - y`形式の境界チェック、たとえば`PAGE_SIZE - hdr - len`、`sizeof(buf) - offset`、`count - consumed`をgrepすること。いずれも同じ書き換えの候補である。次にパターンの後半をチェックすること。つまり、*正当な*操作を繰り返すことで累積値を境界の値まで進められるかどうかである。運用面では、非特権ユーザー名前空間を明示的に無効にしていない限り、`fsopen()`/`fsconfig()`から到達可能なすべての経路、そして`CAP_SYS_ADMIN`でゲートされたすべての経路を、非特権で到達可能なものとして扱うこと。

## 関連項目 {#see-also}

- [ユーザー名前空間とクレデンシャルのマッピング](../user-namespaces.md) — `unshare(CLONE_NEWUSER)`が新しい名前空間の中で完全なケイパビリティセットを与える仕組み。このエクスプロイトは完全にこれに依存している
- [Linuxケイパビリティ](../capabilities.md) — とりわけ`CAP_SYS_ADMIN`が権限昇格の連鎖に繰り返し現れるケイパビリティである理由を含む
- [ファイルシステムの登録とマウント](../../vfs/mounting.md) — `struct fs_context`、`init_fs_context()`、そしてこのバグが存在するマウント経路
- [カーネルハードニング](../kernel-hardening.md) — KASLR、hardened usercopy、SMAP/SMEP。公開されたエクスプロイトはそのすべてを回避する必要があった

## 外部参考資料 {#external-references}

- [NVD: CVE-2022-0185](https://nvd.nist.gov/vuln/detail/CVE-2022-0185) — CVSS 8.4 HIGH、影響を受ける範囲、NVDが`Exploit`とタグ付けした参照。2024年8月21日に追加されたCISA KEVへの掲載、CWE-190、連邦機関向けの是正期限2024年9月11日を確認できる
- [git.kernel.org: 722d94847de2](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=722d94847de29310e8aa03fcbdb41fc92c521756) — "vfs: fs_context: fix up param length parsing in legacy_parse_param"。1行の修正とそのレビュートレーラー
- [git.kernel.org: 3e1aeb00e6d1](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3e1aeb00e6d132efc151dacc062b38269bc9eccc) — David Howellsのファイルシステムコンテキストのシリーズ。5.1-rc1で`legacy_parse_param()`を導入した
- [oss-security: Linux kernel: Heap buffer overflow in fs_context.c since version 5.1](https://www.openwall.com/lists/oss-security/2022/01/18/7) — William Liuによる2022年1月18日の公開。アンダーフローの分析と`unshare`の要件を含む
- [oss-security: CVE-2022-0185: Linux kernel slab out-of-bounds write: exploit and writeup](https://www.openwall.com/lists/oss-security/2022/01/25/14) — Alejandro GuerreroによるUbuntu 21.04向けの独立したエクスプロイトと、発見に関する完全なクレジット一覧
- [Will's Root: CVE-2022-0185 — Winning a $31337 Bounty after Pwning Ubuntu and Escaping Google's KCTF Containers](https://www.willsroot.io/2022/01/cve-2022-0185.html) — 発見者による一次の解説記事。syzkallerの設定、`msg_msg`とFUSEの手法、kCTFコンテナからの脱出
- [GitHub: Crusaders-of-Rust/CVE-2022-0185](https://github.com/Crusaders-of-Rust/CVE-2022-0185) — 公開されたエクスプロイト。`exploit_kctf.c`を含む
- [syzkaller-android-bugs: KASAN: slab-out-of-bounds Write in legacy_parse_param](https://groups.google.com/g/syzkaller-android-bugs/c/hWixpJ22kc8/m/MrIgyhQvEAAJ) — `android12-5.4`に対するsyzbotの2021年12月31日の報告。公開後まで返信がなかった
- [Red Hat Bugzilla 2040358](https://bugzilla.redhat.com/show_bug.cgi?id=2040358) — 非公開のベンダー報告、2022年1月13日起票
