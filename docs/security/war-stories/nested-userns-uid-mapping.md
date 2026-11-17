# ネストしたuser namespaceのUID/GIDマッピング {#nested-user-namespace-uidgid-mapping}

> CVE-2018-18955 — 性能最適化のためにIDマップのソート済みコピーを2つ持つようにしたが、カーネルはその片方しか変換しておらず、デスクトップユーザーが`/etc/shadow`を読めてしまった

公開日
:   2018年11月16日（修正の作成は2018年11月5日、メインラインはv4.20-rc2）

報告者
:   Jann Horn, Google Project Zero（issue 1712）

CVSS
:   7.0 HIGH（NVDの主スコア、`CVSS:3.0/AV:L/AC:H/PR:L/UI:N/S:U/C:H/I:H/A:H`）

バグの存在期間
:   4.15以降（コミット`6397fac4915a`、2017年10月）

修正バージョン
:   4.18.19、4.19.2（メインラインは`d2f007dbe7e4`）

エクスプロイトツール
:   あり — Exploit-DB 45886（Project Zero PoC）および45915（Metasploitモジュール）

実際の悪用
:   確認された事例なし（CISA KEVには未掲載）

*[事件簿: LinuxセキュリティのバグとCVE](../war-stories.md)の一部。*

## 発生前の状態 {#before-state}

user namespaceのIDマッピングは、エクステント（extent）の小さな配列である。各エクステントは3つ組 — namespace内の開始ID、*親*namespaceでの対応する開始ID、そして個数 — であり、すべての`make_kuid()`/`from_kuid()`変換はこの配列を走査する。[User Namespaceとuidマッピング](../user-namespaces.md)では`/proc/<pid>/uid_map`がどのように書き込まれ、2つの方向が何を意味するかを扱い、[クレデンシャルとUser Namespace](../credentials.md)では`struct cred`の中で`kuid_t`がどこに位置するかを扱っている。要するに、互いに逆方向の2種類のルックアップがある:

- `map_id_down()` — namespace内ID → カーネルID。`extent->first`と照合し、`make_kuid()`が使う。
- `map_id_up()` — カーネルID → namespace内ID。`extent->lower_first`と照合し、`from_kuid()`と、決定的に重要なことに`kuid_has_mapping()`が使う。

4.15まで、この配列は5要素で、両方のルックアップは同じソートされていない配列に対する線形走査だった。5個は実際のコンテナツールには少なすぎることが判明した。Christian Braunerは2017年10月に[パッチを投稿](https://lore.kernel.org/all/20171004112009.22642-1-christian.brauner@ubuntu.com/)し、その冒頭で具体的なLXDのケースを挙げた — あるユーザーが「999以外のすべてと、1000000000の範囲のうち1001」をマップするよう求めると、利用可能な5つのエクステントをちょうど使い切り、余地がなくなるというものである。

上限の引き上げには性能面の約束という制約があった。`map_id_up()`は`stat()`のパス上にあるため、`struct uid_gid_map`は1キャッシュラインに収まり続ける必要があった。Braunerの最初の試みはファイルシステム流の間接ポインタと二重間接ポインタを使い、ルックアップは線形のままだった。Eric Biedermanは[10月11日の返信](https://lore.kernel.org/all/87d15t1klz.fsf@xmission.com/)で漸近的な計算量に異議を唱え — 「I am concerned about an implementation that scales linearly with the number of mappings in the large. When it is know we should be able to scale at log(N) of the number of mappings」（訳: 大規模な場合にマッピング数に対して線形にスケールする実装を懸念している。マッピング数のlog(N)でスケールできることがわかっているのだから）— 代替案を自ら素描した。union、エクステントの`kmalloc`を1回、そして二分探索である。Braunerが「forward」と「reverse」は何を意味するのかと尋ねると、Biedermanは[率直に答えた](https://lore.kernel.org/all/87lgkgz43i.fsf@xmission.com/):

> No. Intricacies missed. The sort for forward and reverse is different so we need two different arrays. One sorted on first the other sorted on lower_first.

（訳: いいえ。細部が見落とされている。forwardとreverseでソート順が異なるので、2つの別々の配列が必要だ。一方はfirstでソートし、もう一方はlower_firstでソートする。）

この設計は[`6397fac4915a`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6397fac4915ab3002dc15aae751455da1a852f25)（"userns: bump idmap limits to 340"）としてv4.15で出荷された:

```c
/* include/linux/user_namespace.h */
#define UID_GID_MAP_MAX_BASE_EXTENTS 5
#define UID_GID_MAP_MAX_EXTENTS 340

struct uid_gid_map { /* 64バイト -- 1キャッシュライン */
	u32 nr_extents;
	union {
		struct uid_gid_extent extent[UID_GID_MAP_MAX_BASE_EXTENTS];
		struct {
			struct uid_gid_extent *forward;
			struct uid_gid_extent *reverse;
		};
	};
};
```

エクステントが5個以下なら`extent[]`が使われ、両方向が同じエントリを走査する。5個を超えると、unionは*同じ*エクステントを2つの異なるソート順で保持する2つのヒープ配列に切り替わり、両方のルックアップが`bsearch()`になる。`sort_idmaps()`は1つ目の配列をコピーして2つ目の配列を作っていた:

```c
/* kernel/user_namespace.c、修正前 */
static int sort_idmaps(struct uid_gid_map *map)
{
	if (map->nr_extents <= UID_GID_MAP_MAX_BASE_EXTENTS)
		return 0;

	/* forward配列をソートする。 */
	sort(map->forward, map->nr_extents, sizeof(struct uid_gid_extent),
	     cmp_extents_forward, NULL);

	/* forwardから実際に必要な分のメモリだけをコピーする。 */
	map->reverse = kmemdup(map->forward, ...);
	...
	/* reverse配列をソートする。 */
	sort(map->reverse, map->nr_extents, sizeof(struct uid_gid_extent),
	     cmp_extents_reverse, NULL);
	return 0;
}
```

`map_write()`についてもう1つ知っておくべきことは、`lower_first`が2つの異なる時点で2つの異なる意味を持つという点である。ユーザーが`/proc/<pid>/uid_map`に書き込んだバッファからマップを解析している間、`lower_first`は*親namespaceにおける*IDを保持する。その後になって初めて、ループが各値を親のマップを通してカーネル全体のIDに変換する:

```c
	/* 親user namespaceの下位IDを
	 * カーネル全体のID空間にマップする。
	 */
	for (idx = 0; idx < new_map.nr_extents; idx++) {
		...
		lower_first = map_id_range_down(parent_map, e->lower_first, e->count);
		...
		e->lower_first = lower_first;
	}
```

修正前、`sort_idmaps()`はこのループの**前に**実行され、ループは`new_map.forward`だけを走査していた。

## トリガー {#the-trigger}

`map->reverse`は変換前に取られた`map->forward`の`kmemdup()`であり、変換ループはforward側のコピーしか書き換えなかったため、reverse配列の`lower_first`フィールドには親namespaceのIDがずっと残り続けた。`map_id_up()` — カーネルIDからnamespace内IDへの方向 — はその配列を検索し、カーネルIDを古い親namespaceの値と比較する。

トップレベルのnamespaceではこれは表に出ない。親は`init_user_ns`なので、親のIDは*そのまま*カーネルIDであり、欠けている変換は恒等変換だからである。バグは1段下のnamespaceで、しかもエクステントが5個を超えた場合にのみ現れる。Jann Hornの[報告](https://project-zero.issues.chromium.org/issues/42450783)は最小の手順を示している — 1つ目のnamespace `NS1`に

```
0 100000 1000
```

を設定し、次に`NS1`の内側から、6つのエクステントを持つネストした`NS2`を作る:

```
0 0 1
1 1 1
2 2 1
3 3 1
4 4 1
5 5 995
```

`NS2`は5エクステントのしきい値を超えるので、2配列のパスを使う。その`lower_first`の値は`0..999` — `NS1`におけるID — であり、本来はカーネルID `100000..100999`に変換されるべきである。forward配列にはその変換が適用されるが、reverse配列には適用されない。Hornの言葉では:

> then make_kuid(NS2, ...) will work properly, but from_kuid(NS2) will be an identity mapping for UIDs in the range 0..1000.

（訳: その場合make_kuid(NS2, ...)は正しく動作するが、from_kuid(NS2)は範囲0..1000のUIDについて恒等マッピングになる。）

ここに到達するには複数のIDがマップされたnamespaceが必要だが、それはまさに`/etc/subuid`、`/etc/subgid`、そしてsetuidされた`newuidmap`/`newgidmap`ヘルパーが、`uidmap`パッケージをインストールしたあらゆるデスクトップユーザーに与えるものである — HornはUbuntu 18.04でのテストのためにこのパッケージを明示的にインストールしており、Metasploitモジュールも前提条件としてこれを挙げている。Hornの概念実証（PoC）は1段目に`newuidmap`を使い、その後、すでにnamespace内の`CAP_SYS_ADMIN`を持っている`NS1`の内側から6エクステントのマップを直接書き込んだ。NVDはこのベクタを`AC:H`と評価しているが、難しいのはその形を*見つける*ことであって、再現することではない。

## 観測された挙動 {#observed-behavior}

`from_kuid()`の呼び出し元の大半は表示用のコードである — `ls -l`の出力の数字が間違っているのはバグであって脆弱性ではない。`kuid_has_mapping()`は表示用のコードではない。これは「このカーネルIDはそもそもそのnamespaceに存在するか」に答えるもので、2つのアクセスチェック、`inode_owner_or_capable()`と`privileged_wrt_inode_uidgid()`がその上に構築されている。その帰結についてのHornの要約は次のとおりである:

> Most users of from_kuid() are relatively boring, but kuid_has_mapping() is used in inode_owner_or_capable() and privileged_wrt_inode_uidgid(); so you can abuse this to gain the ability to override DAC security controls on files whose IDs aren't mapped in your namespace.

（訳: from_kuid()の利用者の大半は比較的退屈なものだが、kuid_has_mapping()はinode_owner_or_capable()とprivileged_wrt_inode_uidgid()で使われている。そのため、これを悪用すれば、自分のnamespaceにIDがマップされていないファイルに対してDACのセキュリティ制御を無視する能力を得られる。）

マップされていないカーネルUIDとは、namespace内のrootが何の権限も持たないはずのUIDである — それこそが、そもそも非特権ユーザーにnamespace内で`CAP_DAC_OVERRIDE`を持たせてよい根拠のすべてである。reverseマップが未変換のままだったため、ホストのUID `0..999`はすべて`NS2`でマップされているように見え、その結果`NS2`のnamespace内rootは、実際のホストのrootが所有するファイルに対して権限を持つように見えた。

Hornの実行記録はその結果を具体的に示している。`NS1`の内側では`/etc/shadow`は`nobody`/`nogroup`の所有として表示され、読み取りは失敗する。`NS2` — 構造上*より権限の低い*namespace — の内側では、同じファイルが`root shadow`として表示され、そして:

```
nobody@ubuntu-18-04-vm:~/userns_4_15$ head -n1 /etc/shadow
root:!:17696:0:99999:7:::
```

これは、`uidmap`がインストールされsubuidの範囲が割り当てられた通常のアカウントから、標準のUbuntu 18.04カーネル上で非特権のローカルユーザーがshadowファイルを読んでいるということである。このCVEは4.15から4.19.1までのすべてのカーネルに影響する — およそ1年分のリリースであり、Ubuntu 18.04 LTS、Kubuntu 18.04、Linux Mint 19、Fedora Workstation 28で出荷されたカーネルを含む。これらはすべて[Metasploitモジュール](https://www.rapid7.com/db/modules/exploit/linux/local/nested_namespace_idmap_limit_priv_esc/)がテスト済みの対象として挙げている。

公開ツールは2週間以内に続いた。Project ZeroのPoCは公開日に[Exploit-DB 45886](https://www.exploit-db.com/exploits/45886)としてミラーされ、`exploit/linux/local/nested_namespace_idmap_limit_priv_esc`（Jann Hornとbcolesのクレジット）が2018年11月29日に[Exploit-DB 45915](https://www.exploit-db.com/exploits/45915)として登場した。にもかかわらず、このCVEはCISAのKnown Exploited Vulnerabilitiesカタログに追加されたことは一度もなく — [NVDのレコード](https://nvd.nist.gov/vuln/detail/CVE-2018-18955)にKEVの掲載は見当たらない — 実環境での悪用は確認されていない。

## なぜ起きたか {#why-it-happened}

根本原因は**元データが完成する前に取られた派生コピー**である。`map->reverse`は`kmemdup()`が実行された瞬間には正しく、数行後に変換ループが元の配列を書き換えた時点で誤りになった。コードのどこにも2つの配列の関係は表現されていなかったため、片方が古くなったことを指摘するものは何もなかった。

その根底には、より捉えにくい危険がある。`lower_first`は単一の`u32`フィールドで、解析中は親namespaceのIDを、設定後はカーネル全体のIDを保持する。全体を通じて型も名前も構造体メンバーも同じであり — コンパイラから見える目印のない、時間的な型の変化である。forward配列はその境界を越えたが、reverse配列は越えなかった。

これがレビューされていないコードではなかったことは明確にしておく価値がある。340エクステントのシリーズは3週間にわたって複数のリビジョンを経ており、Serge HallynとEric Biedermanがスレッドで返信していた（Tycho AndersenはCcに入っていたが投稿はしていない）。さらにBiedermanによる[5パッチのフォローアップシリーズ](https://lore.kernel.org/all/871sliubhj.fsf_-_@xmission.com/)があり — こちらではPeter Zijlstra、Nikolay Borisov、Joe Perchesがレビューした — そこでは別種の本物の正しさのバグが見つかった。マップの書き込み中に`nr_extents`が2回読まれるため、「it could be 0 the first time and > 5 the second time, which would lead to misinterpreting the union fields」（訳: 1回目は0で2回目は5より大きいということがあり得て、unionのフィールドを誤解釈することにつながる）というものである。そのレビューは注意深かった。ただし注意が向けられていたのは並行性、キャッシュライン、メモリバリアであり、危険がありそうに見えたのはそこだった。書き込みパスにおける一度きりの初期化順序の誤りは、誰も探していたリスクの種類ではなかった。

最後に、バグのあるパスは偶然にはほぼ到達不能だった。6個以上のエクステントは珍しく、*ネストしたnamespaceで*6個以上のエクステントとなれば珍しさの二乗である。多数のエクステントを使うコンテナランタイムは通常それを最も外側のnamespaceで行い、そこでは親が`init_user_ns`なので欠けている変換は何もしないのと同じである。そのためこのバグは1年間、機能上の苦情を一切生まなかった — `ls`の出力がおかしくなることも、コンテナが壊れることも、bisectする対象もなく — 誰かが意図的に探しに行くまでは。

## 解決 {#resolution}

[`d2f007dbe7e4`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d2f007dbe7e4c9583eea6eb04d60001e85c6f1bd)（"userns: also map extents in the reverse map to kernel IDs"）は2018年11月5日にJann Hornが作成したもので、8行の追加と4行の削除である。`sort_idmaps()`を変換ループの前から後へ移動させる:

```diff
-	ret = sort_idmaps(&new_map);
-	if (ret < 0)
-		goto out;
-
 	ret = -EPERM;
 	/* Map the lower ids from the parent user namespace to the
 	 * kernel global id space.
 	 */
 	for (idx = 0; idx < new_map.nr_extents; idx++) {
 		...
 		e->lower_first = lower_first;
 	}

+	/*
+	 * If we want to use binary search for lookup, this clones the extent
+	 * array and sorts both copies.
+	 */
+	ret = sort_idmaps(&new_map);
+	if (ret < 0)
+		goto out;
```

この順序は両方向で決定的に重要であり、コミットメッセージがその理由を述べている:

> To fix it, we have to make sure that the "lower_first" members of extents in both arrays are translated; and we have to make sure that the reverse map is sorted *after* the translation (since otherwise the translation can break the sorting).

（訳: 修正するには、両方の配列のエクステントの"lower_first"メンバーが変換されるようにしなければならない。また、reverseマップが変換の*後に*ソートされるようにしなければならない（そうでないと変換によってソートが壊れうるため）。）

先にソートして後で変換するのは機能しない。reverse配列のソートキーは*まさに*`lower_first`だからである。ソート後にその場で変換すれば、`bsearch()`が依存する順序を黙って破壊してしまう。変換後に複製すれば両方の性質が同時に得られる。コピーは変換済みの値を受け継ぎ、各配列はその後二度と変わらないキーでソートされる。forward配列はどちらの場合も影響を受けない — ソートキーは`first`であり、変換はそれに触れないからである。

このコミットには`Fixes: 6397fac4915a`、`Cc: stable@vger.kernel.org`、そしてEric Biedermanによる`Tested-by:`と`Reviewed-by:`の両方が付いており、適用したのも彼である。2018年11月13日にタグ付けされた4.18.19と4.19.2のstableリリースで出荷され、ディストリビューションの更新がそれに続いた（Ubuntu向けには[USN-3832-1](https://usn.ubuntu.com/3832-1/)、3833-1、3835-1、3836-1/-2）。

**議論について: 公の場では議論はなかった。** これはごまかさずに率直に述べておく価値がある。このパッチには`Link:`トレーラーがなく、メーリングリストのアーカイブを検索しても、適用前にLKML、`linux-security-module`、containersリストのいずれかに投稿された形跡は見つからない — エンバーゴのもとで`security@kernel.org`を経由し、直接マージされた。最初の公の登場は、下記のstableへの投稿より1日早い、Biedermanの[`[GIT PULL] namespace fixes for v4.20-rc2`](https://lore.kernel.org/lkml/87pnvcls3n.fsf@xmission.com/)（2018年11月10日）であり、そのshortlogには"Jann Horn (1): userns: also map extents in the reverse map to kernel IDs"とクレジットされ、本文ではいくつかの"simple obviously correct bug fixes"（単純で明らかに正しいバグ修正）の1つと呼ばれている。返信は自動のマージトラッカーによる受領通知2件と、ボットのメール重複に対するkernel.orgのKonstantin Ryabitsevからの謝罪だけで — 人間によるレビューはない。その後2018年11月11日にGreg Kroah-Hartmanのstableレビュー投稿（[`[PATCH 4.18 349/350]`](https://lore.kernel.org/all/20181111221722.721113385@linuxfoundation.org/)と[`[PATCH 4.19 359/361]`](https://lore.kernel.org/all/20181111221702.182204786@linuxfoundation.org/)）に登場したが、どちらもこのパッチについての返信はつかなかった。記録に残る公の議論は、stableリリースの3日後に行われたHornの[oss-securityへの投稿](https://www.openwall.com/lists/oss-security/2018/11/16/1)と、そこからリンクされたProject Zeroのissueだけであり — レビュースレッドではない。メーリングリストだけからこのバグの歴史を再構成しようとする人は、バグを持ち込んだ2017年のスレッドを見つけるだけで、修正については中身のあるものを何も見つけられないだろう。

本物のフォローアップスレッドが1つあるが、それは遅れてやってきた。修正から2年8か月後の2021年7月、SUSEのRichard Palethorpeが[LTPに再現テストを投稿した](https://lore.kernel.org/all/20210712162208.2396-1-rpalethorpe@suse.com/)。`userns08`で、`{"linux-git", "d2f007dbe7e4"}`と`{"CVE", "CVE-2018-18955"}`のタグが付けられ、6エクステントの内側マップを持つ同じ2段のnamespaceを構築して、制限されたファイルを開くと`EACCES`で失敗することを確認する。Cyril Hrubisが定型的なスタイル上の指摘（`TST_CHECKPOINT_WAKE_AND_WAIT()`を使うこと、チェックポイントは0から番号を振ること、`SAFE_OPEN(..., O_CREAT, 0700)`に`O_WRONLY`が欠けていること）とともにレビューし、v2でそれらが対応された。興味深いのはそのタイミングである。3年近くの間、ここでのリグレッションを検出できた唯一の実行可能な成果物はHornのエクスプロイトだった。

```bash
# このカーネルは、あるプロセスについて2配列のパスを使っているか？
wc -l < /proc/self/uid_map     # 5行を超える => forward/reverseのbsearch配列を使用中

# 到達可能性を左右する設定（名前はディストリビューションにより異なる）
sysctl kernel.unprivileged_userns_clone   # 存在する場合、0なら非特権のuserns作成をブロック
cat /proc/sys/user/max_user_namespaces    # 0なら汎用的にブロック
ls -l /usr/bin/newuidmap                  # 複数IDの1段目マップを与えるsetuidヘルパー
```

## 得られた教訓 {#what-it-taught-us}

**コピーは守り続けなければならない約束である。** `kmemdup()`が作った2つ目の配列は、元の配列が変わるまでのほんの数行の間だけ正しかった。性能改善のために派生表現 — ソート済みの複製、キャッシュされたインデックス、逆引きマップ — を導入するたびに、「この2つは一致している」という不変条件は保守上の義務となり、型システムはそれを助けてくれない。元データが確定した後、*最後に*派生させるほうが、2つのものを同期させ続けるより安上がりである。

**関数の途中で意味が変わるフィールドは、ラベルのない型変更である。** `lower_first`は入口では親namespaceのIDを、出口ではカーネル全体のIDを保持していた。どちらも`u32`で、綴りも同じであり、両者を分けているのは1つのループだけである。同じ記憶領域が互換性のない2つの解釈を持つ場合、すべてのコードパスについて、それが変化のどちら側にあるかを確認しなければならない — そしてreverse配列のパスは誰も確認しなかったものだった。

**レビューは、探しているリスクの種類しか捉えない。** このバグを持ち込んだシリーズは複数のリビジョンを経て、サブシステムのメンテナによる専用のフォローアップシリーズも受けており、そこではunionを誤解釈しかねない本物の二重読み取りの競合が見つかった。レビュアーが並行性とキャッシュラインのレイアウトについて考えていたのは、それがパッチの*主題*だったからである。書き込みパスでの初期化順序は誰のリストにもなく、ある軸に沿って厳しくレビューされたことは、他の軸についてはほとんど何も語らない。

**「ネストによってのみ到達可能」は「攻撃者によってのみ到達可能」を意味する。** reverseが未変換のバグは、トップレベルのnamespaceでは無害で、通常のコンテナ構成ではすべて沈黙しており、1年間ユーザーに見える症状を何も生まなかった。正規のソフトウェアが決して生成しない構成にはバグ報告は来ない — 来るのはセキュリティ研究者である。ネストの深さ、通常とは異なる個数、境界的な構成は、他の何もそれらを実行しないからこそ、意図的なテストに値する。

!!! warning "注意すべきパターン"
    次の2つが揃っている箇所を探すこと: ルックアップ速度のために複製されたデータ構造と、その複製が作られた後にその場で書き換えられるフィールド。どちらか一方だけなら問題ない。両方が揃うと、一方のコピーが変換前の値を黙って保持することになる。同じ関数内で後から変更される構造体の`kmemdup()`/`memcpy()`をgrepし、後続の各読み手がどちらのコピーを参照しているかを問うこと。運用面では、非特権のuser namespaceをネストできるカーネルはすべて、namespace単位のアクセスチェック — `kuid_has_mapping()`、`inode_owner_or_capable()`、`privileged_wrt_inode_uidgid()` — が、任意のローカルユーザーから到達可能なセキュリティ境界となっているカーネルである。

## 関連項目 {#see-also}

- [User Namespaceとuidマッピング](../user-namespaces.md) — `uid_map`/`gid_map`、`make_kuid()`/`from_kuid()`、そしてこのバグが悪用したネストのモデル
- [クレデンシャルとUser Namespace](../credentials.md) — `struct cred`の中で`kuid_t`がどこにあり、ID変換がどのようにアクセスチェックに届くか
- [ケーパビリティ](../capabilities.md) — `CAP_DAC_OVERRIDE`と`CAP_SYS_ADMIN`の背後にあるケーパビリティモデルと、`capable()`/`ns_capable()`チェックの書き方
- [Netfilter x_tablesのヒープオーバーフロー](../../net/war-stories/netfilter-xtables.md) — 非特権user namespace内から到達可能であることで深刻度が高まった、もう1つのバグ

## 外部参考資料 {#external-references}

- [NVD: CVE-2018-18955](https://nvd.nist.gov/vuln/detail/CVE-2018-18955) — CVSS 3.0基本値7.0 HIGH、`AV:L/AC:H/PR:L/UI:N/S:U/C:H/I:H/A:H`、2018年11月16日公開
- [git.kernel.org: d2f007dbe7e4](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d2f007dbe7e4c9583eea6eb04d60001e85c6f1bd) — "userns: also map extents in the reverse map to kernel IDs"。修正であり、自らCVE-2018-18955であると明記している
- [lore.kernel.org: [GIT PULL] namespace fixes for v4.20-rc2](https://lore.kernel.org/lkml/87pnvcls3n.fsf@xmission.com/) — 修正の実際の最初の公の登場。stableレビュー投稿の1日前
- [git.kernel.org: 6397fac4915a](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6397fac4915ab3002dc15aae751455da1a852f25) — "userns: bump idmap limits to 340"。`Fixes:`トレーラーで名指しされたコミット
- [Project Zero issue 1712](https://project-zero.issues.chromium.org/issues/42450783) — Jann Hornによる元の報告、分析、再現コード（旧`bugs.chromium.org/p/project-zero/issues/detail?id=1712`）
- [oss-security: broken uid/gid mapping for nested user namespaces with >5 ranges](https://www.openwall.com/lists/oss-security/2018/11/16/1) — Hornによる公開開示の投稿、2018年11月16日
- [lore.kernel.org: user namespaces: bump idmap limits](https://lore.kernel.org/all/20171004112009.22642-1-christian.brauner@ubuntu.com/) — forward/reverseの2配列設計が提案され合意された2017年10月のレビュースレッド
- [lore.kernel.org: userns: bump idmap limits, fixes & tweaks](https://lore.kernel.org/all/871sliubhj.fsf_-_@xmission.com/) — Eric Biedermanのフォローアップシリーズ。`nr_extents`二重読み取りの修正を含む
- [lore.kernel.org: userns08, CVE-2018-18955: Broken id mappings in nested namespaces](https://lore.kernel.org/all/20210712162208.2396-1-rpalethorpe@suse.com/) — 2021年7月に追加されたLTPのリグレッションテストと、Cyril Hrubisによるレビュー
- [Exploit-DB 45886](https://www.exploit-db.com/exploits/45886)と[45915](https://www.exploit-db.com/exploits/45915) — Project Zeroの概念実証と、それから派生したMetasploitモジュール
- [Rapid7: exploit/linux/local/nested_namespace_idmap_limit_priv_esc](https://www.rapid7.com/db/modules/exploit/linux/local/nested_namespace_idmap_limit_priv_esc/) — モジュールのドキュメント。テスト済みのディストリビューションと`uidmap`パッケージの前提条件を含む
- [Ubuntu: USN-3832-1](https://usn.ubuntu.com/3832-1/) — このCVEに対して発行された5件のUbuntuアドバイザリの1つ
