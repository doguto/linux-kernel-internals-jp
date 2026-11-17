# OverlayFS: マッピングされていないUIDを信頼したcopy-up {#overlayfs-the-copy-up-that-trusted-an-unmapped-uid}

> CVE-2023-0386 — OverlayFSのcopy-upパスは、下位ファイルの所有者UID/GIDがマウントした者のユーザー名前空間に実際にマッピングを持っているかを一度も確認していなかった。そのため攻撃者は、ケーパビリティを持つファイルを非特権のoverlayマウントを通じて持ち込み、最終的にrootを得ることができた

公開
:   2023年3月22日（NVDレコード）

CVSS
:   7.8 HIGH（`CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`）

バグの存在期間
:   `459c7c565ac3`（"ovl: unprivieged mounts"）によって非特権ユーザーがユーザー名前空間内でOverlayFSをマウントできるようになった5.11（2021年2月）以降

修正
:   コミット `4f11ada10d0a`、メインライン6.2、2023年2月。stableにバックポート済み

エクスプロイトツール
:   あり — 公開PoC（`xkaneiki/CVE-2023-0386`、`ovlcap` エクスプロイト）がrootへの権限昇格を実証

実際の悪用
:   あり — 2025年6月17日にCISA KEVに追加

*[事件簿: VFSのバグとリグレッション](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

`459c7c565ac3`（"ovl: unprivieged mounts"、Miklos Szeredi、Linux 5.11）は、非特権ユーザーが自身のユーザー名前空間内でOverlayFSマウントを作成できるようにした。これは「rootlessコンテナ」の利用者層のために明示的に作られた機能であり、OverlayFSの文書化された権限モデルはそれを安全にするよう設計されていた。マウントした者が自身の資格情報で下位のレイヤーに対してすでにできることが、overlayマウントでできることの上限であり、それ以上はできない、というものである。

読み取り専用の下位レイヤーにしか存在しないファイルに書き込むと、**copy-up** が発生する。OverlayFSは下位ファイルの `stat` データ（マウントした者のユーザー名前空間を通して見た所有者UIDとGIDを含む）を読み、`fs/overlayfs/copy_up.c` の `ovl_copy_up_one()` を通じて、書き込み可能な上位レイヤーに対応するファイルを作成する。

## きっかけ {#the-trigger}

ユーザー名前空間内でまったくマッピングを持たないUIDやGIDは、`stat(2)` にエラーとして報告されるのではなく、**オーバーフローUID**（デフォルトでは `65534`、`nobody`）として報告される。`ovl_copy_up_one()` はその `stat` の結果を額面どおりに受け取り、実際の下位のUIDが `current_user_ns()` で実際にマッピングを持っているかどうかにかかわらず、copy-upを進めていた。

これは、修正のコミットメッセージが指摘しているまさにその不整合を生んだ。オーバーフローUIDの `65534` 自体がたまたまマウントした者の名前空間で本当のマッピングを持っていた場合、copy-upは成功し、`65534` がマッピングされた先のユーザーが所有するファイルを作成する。下位ファイルの*元の*UIDは実際には一度もマッピングされておらず、同じ非特権ユーザーが通常の権限チェック付きのパスを通じて手動で `cp -a` を実行していれば、まさにこの時点で失敗していたはずであるにもかかわらずである。

## 観測された挙動 {#observed-behavior}

公開された概念実証（`xkaneiki/CVE-2023-0386`。以前のOverlayFSのバグ群「GameOver(lay)」のパターンを踏襲している）は、攻撃者が完全に制御するFUSEベースの下位ファイルシステムと、Linuxのファイルケーパビリティを持つファイルを組み合わせる。攻撃者のFUSEレイヤーは、マッピングされていないUIDとオーバーフローUIDの不整合を突く形でそのファイルの所有権を報告し、そのケーパビリティを持つファイルの書き込み可能な上位レイヤーへのcopy-upを引き起こす。同じ権限での手動コピーなら拒否されていたはずのところを、ケーパビリティが尊重される場所に置いてしまうのである。Red Hatのアドバイザリは、結果として得られるプリミティブを率直に要約している。"unauthorized access to the execution of the setuid file with capabilities... how a user copies a capable file from a nosuid mount into another mount."（訳: ケーパビリティ付きsetuidファイルの実行への不正アクセス……ユーザーがケーパビリティを持つファイルをnosuidマウントから別のマウントへコピーする方法。）そこからPoCの `getshell` ステップが、不正に得たケーパビリティを完全なrootシェルに変える。OverlayFSの非特権ユーザー名前空間マウントは主要なディストリビューション（Ubuntuを含む）でデフォルトで有効になっており、一般的なコンテナランタイムでも使われているため、このバグの実際の影響範囲は、影響を受けるカーネルを実行しているすべてのデスクトップと多くのコンテナホストに及んだ。CISAは2025年6月17日にCVE-2023-0386をKnown Exploited Vulnerabilitiesカタログに追加し、最初のパッチから2年以上経った後も現実世界での悪用があることを確認した。

## なぜ起きたか {#why-it-happened}

`459c7c565ac3` は、マウントした者自身のユーザー名前空間の権限チェックがoverlayでできることを制限するという前提の上に、非特権のOverlayFSマウントを構築した。マウントした者の資格情報が操作の時点で実際に再チェックされるところでは、どこでも正しい前提である。copy-upの所有権の扱いは、その前提が黙って成り立っていなかった場所の1つだった。それは、マウントした者のUID/GIDマッピングを*すでに*通過した `stat` データを読んでいた（そのため「安全」に、ただの数値に見えた）が、その数値が、マッピングされていないIDがすべて収束する曖昧なオーバーフロー値ではなく、本当に有効なマッピングを反映しているかを別途確認していなかった。本当にマッピングされた `nobody` からの `65534` と、「このIDはまったくマッピングされていない」の代わりとしての `65534` は、`stat` に届いた時点で区別できない。そして書き込みパスだけが、等価でないそれらを等価なものとして扱っていた。

## 解決 {#resolution}

`4f11ada10d0a`（"ovl: fail on invalid uid/gid mapping at copy up"）は、欠けていたまさにそのチェックを追加する。copy-upを進める直前の4行である。

```c
if (!kuid_has_mapping(current_user_ns(), ctx.stat.uid) ||
    !kgid_has_mapping(current_user_ns(), ctx.stat.gid))
	return -EOVERFLOW;
```

下位ファイルの本当のUIDやGIDがマウントした者のユーザー名前空間で有効なマッピングを持たない場合、copy-upは `-EOVERFLOW` で即座に失敗するようになった。これは同じユーザーによる手動の `cp -a` が行っていたであろう動作と一致する。これにより、コミットメッセージが指摘する、`stat(2)` のオーバーフローUIDの挙動とPOSIX ACLとの間のまさにその不整合が解消された。POSIX ACLはすでに、不正なUID/GIDに対して黙ってプレースホルダーで置き換えるのではなく、`-1` を返してきれいに失敗するようになっていた。

## 教訓 {#what-it-taught-us}

**セキュリティ上重要な変換をすでに1つ通過した値は、さらにチェックが必要だと知らない下流のコードには「安全」に見えうる。** copy-upがUIDを目にする時点で、それはすでにマウントした者の名前空間を通してマッピングされていた。その段階だけで普通の信頼できる数値に見えたが、実際に必要だったのは、マッピングが曖昧なプレースホルダーに黙って劣化したのではなく*成功した*ことのチェックだった。

**現実世界の複数の異なる状態を1つの表現に収束させるオーバーフロー値や番兵値は、まさにこの種のバグを常に招いている。** `65534` が「本当にnobody」と「マッピングされておらず、実際にはわからない」の両方を意味することは、`stat(2)` のユーザー向けの契約としては問題ない。しかし、まず2つ目の意味を排除せずにその数値に基づいて動作するセキュリティ上重要なコードにとっては問題である。

!!! warning "注意すべきパターン"
    ID（UID、GID、その他名前空間付きの識別子）がマッピングの境界をまたぎ、その結果が普通の値であるかのように消費される場所ではどこでも、それに基づいて動作する前にオーバーフロー／番兵のケースを明示的にチェックすること。特にその動作（ここでのcopy-upのように）が、そもそもその値を生み出した読み出しを超えるセキュリティ上の帰結を持つ場合に注意すること。

## 関連項目 {#see-also}

- [VFSの概要](../README.md) — OverlayFSがその上に重なる汎用のsuperblock/inode操作
- [OverlayFS: ケーパビリティxattrのバイパス](overlayfs-capability-bypass.md) — 同じ「ケーパビリティが、生き残るべきでないcopy-upを生き残る」種類のバグの、より初期の変種
- [セキュリティ: LSM、ケーパビリティ、seccomp](../../security/README.md) — OverlayFSのcopy-upパスが依存するケーパビリティチェックの機構

## 外部参考資料 {#external-references}

- [git.kernel.org: 4f11ada10d0a](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4f11ada10d0ad3fd53e2bd67806351de63a4f9c3) — "ovl: fail on invalid uid/gid mapping at copy up"、修正
- [git.kernel.org: 459c7c565ac3](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=459c7c565ac36ba09ffbf24231147f408fde4203) — "ovl: unprivieged mounts"、このバグが必要としたマウント構成を可能にしたコミット
- [NVD: CVE-2023-0386](https://nvd.nist.gov/vuln/detail/CVE-2023-0386) — CVEレコード、CVSS 7.8 HIGH。2025年6月17日にCISA KEVに掲載
- [Red Hat: CVE-2023-0386](https://access.redhat.com/security/cve/cve-2023-0386) — ベンダーのアドバイザリと技術的な要約
