# OverlayFS: 1つの呼び出し元だけが行っていたケーパビリティチェック {#overlayfs-the-capability-check-that-only-one-caller-made}

> CVE-2021-3493 — `security.capability` xattrを対象のユーザー名前空間に照らして検証するコードは、すべてのファイルシステムが信頼できるはずの汎用の `vfs_setxattr()` ではなく、`setxattr(2)` システムコールハンドラの中にあった。そしてOverlayFSのcopy-upパスは後者を呼んでいた

公開
:   2021年4月15日（CanonicalのSteve Beattieがoss-securityへ）

報告者
:   独立したセキュリティ研究者、SSD Secure Disclosureプログラム経由

CVSS
:   8.8 HIGH、Ubuntu Security Teamの評価（`CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H`）。7.8 HIGH、NVDの主評価（`.../S:U/...`）

修正
:   コミット `7c03e2cda4a5`、Linux 5.11の開発サイクル中、2020年12月にマージ

エクスプロイトツール
:   あり — 影響を受けるUbuntuカーネルでのroot権限昇格を実証する公開PoCが2021年に広く出回った

実際の悪用
:   あり — 2022年10月20日にCISA KEVに追加

*[事件簿: VFSのバグとリグレッション](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

Linuxのファイルケーパビリティ（`security.capability` xattr）を使うと、特定のバイナリを完全なsetuid-rootにすることなく、昇格した権限で実行させられる。`cap_setuid`、`cap_net_bind_service` などを、プロセス全体に付与するのではなくファイルに付けるのである。ケーパビリティは名前空間に相対的である（あるユーザー名前空間で意味を持つケーパビリティが、別の名前空間で自動的に尊重されるべきではない）ため、そのxattrの設定は `cap_convert_nscap()` を通る。この関数は2つのことを同時に行う。呼び出し元プロセスがこの種の名前空間付きケーパビリティを設定することを実際に許可されているかを検証し、ディスク上のxattrの値を名前空間付き（`vfs_ns_cap_data`）形式に書き換える。

`7c03e2cda4a5` 以前、その `cap_convert_nscap()` の呼び出しは、`fs/xattr.c` の `setxattr()` システムコールハンドラ、つまり特定の1つの呼び出し元の中にあった。すべてのファイルシステムの内部xattr設定ロジックが直接呼び出して信頼できると期待される汎用エントリポイント `vfs_setxattr()` の中ではなかったのである。

## きっかけ {#the-trigger}

OverlayFSのcopy-upパスは、新たに作成した上位ファイルにxattrを設定する際に `vfs_setxattr()` を直接呼ぶ。内部のファイルシステム操作の中ですでにカーネル内のdentryを扱っているので、ユーザー空間の `setxattr(2)` システムコールパスを経由する理由がないからである。つまり、OverlayFSによってcopy-upされるすべての `security.capability` xattrは、`cap_convert_nscap()` を完全にスキップしていた。関係するユーザー名前空間でそのケーパビリティを付与することが妥当かどうかの権限チェックも、名前空間付き形式への変換も行われなかった。

NVD自身の要約は、悪用可能性の条件を正確に述べている。"the overlayfs implementation in the linux kernel did not properly validate with respect to user namespaces the setting of file capabilities on files in an underlying file system. Due to the combination of unprivileged user namespaces along with a patch carried in the Ubuntu kernel to allow unprivileged overlay mounts, an attacker could use this to gain elevated privileges."（訳: Linuxカーネルのoverlayfs実装は、下位ファイルシステム上のファイルへのファイルケーパビリティの設定を、ユーザー名前空間に関して適切に検証していなかった。非特権ユーザー名前空間と、非特権のoverlayマウントを可能にするためにUbuntuカーネルが保持していたパッチの組み合わせにより、攻撃者はこれを使って昇格した権限を得ることができた。）アップストリームのメインラインのOverlayFSは、5.11で `459c7c565ac3` がマージされるまで非特権マウントをまったくサポートしていなかった。Ubuntuのカーネルは、それがアップストリームに入る何年も前から、非特権のoverlayマウントを可能にする独自のツリー外パッチを保持していた。そのため、根底にある `vfs_setxattr()` のバグ自体が修正されるよりずっと前から、この特定の隙間には非特権のローカルユーザーが到達できたのである。

## 観測された挙動 {#observed-behavior}

Ubuntuが保持していた非特権overlayマウントのパッチを利用できる非特権ユーザーは、`security.capability` xattrを持つファイルを含む下位レイヤーを構築し、OverlayFSにそのファイルをcopy-upさせ（copy-upパスは欠けていた権限チェックを一度も通さない）、その結果できた上位レイヤーのファイルを実行して、付与されたケーパビリティをそのまま得ることができた。選んだケーパビリティが十分に強力なら、rootへの直接の道となる（たとえば `cap_setuid` があれば容易にrootシェルが得られる）。悪用に必要なのは通常のユーザー名前空間とoverlayマウントへのアクセスだけで、特別なものは何もなく、どちらも影響を受けるUbuntuリリースではデフォルトで有効だったため、非特権のローカルシェルから到達可能だった。CISAは2022年10月20日にCVE-2021-3493をKnown Exploited Vulnerabilitiesカタログに追加し、2021年の最初の公開とパッチからかなり経った後も、現実世界での悪用が続いていることを確認した。

## なぜ起きたか {#why-it-happened}

`cap_convert_nscap()` の二重の役割（権限チェック*と*形式変換）は、唯一の呼び出し元が `setxattr(2)` システムコールパスであると想定して書かれていた。当時は、それが `security.capability` xattrを設定できる唯一の方法だったからである。他のどの呼び出し元から見ても、`vfs_setxattr()` は正しく汎用的で、すでに権限チェック済みのエントリポイントに見えた。コミットメッセージもそう述べている。これは「どのみち `vfs_foo()` がやるべきこと」なのである。隙間はチェックの欠落というより、関数の名前とVFSのコールグラフにおける位置から当然そこにあると思われる場所より、一段上のレイヤーにチェックが置かれていたことだった。

## 解決 {#resolution}

`7c03e2cda4a5`（"vfs: move cap_convert_nscap() call into vfs_setxattr()"、Miklos Szeredi）は、`cap_convert_nscap()` の呼び出しを `setxattr()` システムコールハンドラから `vfs_setxattr()` 自体の中へ移す。これにより、すべての呼び出し元（システムコールパス、OverlayFSのcopy-up、eCryptfsなどその他のカーネル内呼び出し元）が同じ権限チェックと名前空間付き形式への変換を受けるようになり、「汎用」関数を直接呼ぶことでそれを迂回する方法はなくなった。

## 教訓 {#what-it-taught-us}

**すべての呼び出し元が信頼すると期待される共有関数ではなく、特定の1つの呼び出し元に置かれたセキュリティチェックは、新しい呼び出し元が付け直すべきだと知りようのないチェックである。** `vfs_setxattr()` の名前とVFSレイヤーにおける位置は、将来のすべての呼び出し元に対して、直接呼んでも安全だと告げていた。VFSの他の部分が従うあらゆる慣習に照らして、それは正しい。権限チェックが、インターフェース自身の契約が示す場所にただ存在しなかっただけである。

**ユーザー空間向けのシステムコールのラッパーを迂回するカーネル内の呼び出し元は、そのラッパーにだけ存在した検証を、黙って取りこぼす。** OverlayFSがシステムコールを経由せずに `vfs_setxattr()` を呼んだのは、何ら特別なことではない。それこそが汎用VFSレイヤーを持つ意味のすべてである。バグは、`vfs_setxattr()` が約束していたことと実際に行っていたことの食い違いにあった。

!!! warning "注意すべきパターン"
    ファイルシステムに依存しない汎用エントリポイント（`vfs_*()` 関数やコアのヘルパー）に、追加の検証を行う兄弟のシステムコールラッパーがある場合、新しいカーネル内の呼び出し元がそれを正しく引き継ぐと信頼する前に、その検証がラッパーではなく汎用関数自体の中にあることを確かめること。チェックがシステムコールハンドラを読まなければ見つからないなら、システムコールを経由しない呼び出し元はチェックもスキップする。

## 関連項目 {#see-also}

- [VFSの概要](../README.md) — このバグが存在した汎用の `vfs_setxattr()` インターフェース
- [OverlayFS: UIDマッピングの混乱](overlayfs-uid-confusion.md) — ケーパビリティではなくUID/GIDマッピングの側で起きた、2つ目のcopy-up関連の権限昇格
- [セキュリティ: LSM、ケーパビリティ、seccomp](../../security/README.md) — Linuxのファイルケーパビリティと、このバグが迂回した名前空間相対の権限モデル

## 外部参考資料 {#external-references}

- [git.kernel.org: 7c03e2cda4a5](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7c03e2cda4a584cadc398e8f6641ca9988a39d52) — "vfs: move cap_convert_nscap() call into vfs_setxattr()"、修正
- [NVD: CVE-2021-3493](https://nvd.nist.gov/vuln/detail/CVE-2021-3493) — CVEレコード。2022年10月20日にCISA KEVに掲載
