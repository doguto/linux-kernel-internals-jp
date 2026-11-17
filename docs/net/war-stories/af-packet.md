# AF_PACKET TPACKET_V3 の権限昇格 {#af_packet-tpacket_v3-privilege-escalation}

> CVE-2017-7308 — 符号なしの減算を囲むたった1つの `(int)` キャストが、リングバッファの境界チェックを無力化した。syzkaller によって発見され、Project Zero によってローカル root に仕立てられた

公開日
:   2017年3月29日

報告者
:   Andrey Konovalov、Google Project Zero（syzkaller + KASAN による）

CVSS
:   7.8 HIGH

修正
:   2017年3月にメインラインで修正、4.10.x およびそれ以前の stable ブランチにバックポート

エクスプロイトツール
:   あり（Exploit-DB に2件、[Metasploit モジュール](https://www.exploit-db.com/exploits/44654)を含む）

実際の悪用
:   確認された事例はない（CISA KEV には掲載されていない）

*[事件簿：ネットワークスタックのバグと CVE](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

`AF_PACKET` ソケット（`tcpdump`、Wireshark、その他あらゆる生パケットキャプチャツールが使う）は、`PACKET_RX_RING`/`PACKET_TX_RING` によってメモリマップされたリングバッファを使うよう設定でき、キャプチャしたパケットごとのコピーを避けられる。`net/packet/af_packet.c` の `packet_set_ring()` は、リングを割り当ててマップする前に、要求されたリングのジオメトリ — ブロックサイズ、ブロック数、フレームサイズ — を検証する。`TPACKET_V3` のリング形式では、各ブロックはパケットフレームに加えて小さなプライベートヘッダ（`tp_sizeof_priv`）用の領域を予約し、これは `BLK_PLUS_PRIV()` マクロで計算される。

この計算を守る妥当性チェックは、次のようになっていた。

```c
if (po->tp_version >= TPACKET_V3 &&
    (int)(req->tp_block_size - BLK_PLUS_PRIV(req_u->req3.tp_sizeof_priv)) <= 0)
        goto out;
```

## きっかけ {#the-trigger}

`tp_block_size` も `BLK_PLUS_PRIV()` の結果も符号なしである。両者の差をゼロと比較する前に `(int)` にキャストするのは安全ではない。[Project Zero の Andrey Konovalov](https://projectzero.google/2017/05/exploiting-linux-kernel-via-packet.html) は、syzkaller（カバレッジガイド型の syscall ファザー）と KASAN を組み合わせて、最上位ビットを立てた `tp_sizeof_priv` の値を与えると、符号なしの減算がラップアラウンドして大きな値になり、それを符号付き `int` として再解釈すると正の値に見えることを発見した — 大きすぎるプライベートヘッダの要求を拒否するはずだったチェックを、黙って通過してしまうのである。

これにより `blk_sizeof_priv` が攻撃者の選んだ値に設定されることになり、`init_prb_bdqc()` と `prb_open_block()` で計算されるブロックレイアウトが破壊され、さらにその下流で `__packet_lookup_frame_in_block()` が使う `max_frame_len` の計算も破壊された — カーネルが認識しているリングのフレームレイアウトと、マップされたブロックの実際のサイズとが食い違う状態が生まれた。

## 観測された挙動 {#observed-behavior}

計算されたフレームレイアウトと実際のブロック境界との不一致により、呼び出し側は、カーネルがリングバッファのフレームを埋める際に**ヒープの範囲外書き込み**を引き起こせた — このローカルな整数の符号のバグを、Project Zero の[解説記事](https://projectzero.google/2017/05/exploiting-linux-kernel-via-packet.html)「Exploiting the Linux kernel via packet sockets」は、動作するローカル root エクスプロイトに仕立てた。

`AF_PACKET` ソケットを作成するにはそもそも `CAP_NET_RAW` が必要である — 通常なら意味のある障壁だが、解説記事が指摘するように、"if a kernel has unprivileged user namespaces enabled, then an unprivileged user is able to create packet sockets"（訳: カーネルで非特権ユーザー namespace が有効になっていれば、非特権ユーザーがパケットソケットを作成できる）。そして Ubuntu は `CONFIG_USER_NS=y` かつその使用に何の制限もない状態で出荷されており、これは当時一般的なデフォルトだった（解説記事は、Android が信頼されないコードによる `AF_PACKET` ソケットの作成を一切許可していないことを明記している）。

Konovalov は解説記事とともに自身の動作するエクスプロイトを公開し（[Exploit-DB 41994](https://www.exploit-db.com/exploits/41994)、2017年5月）、1年後には Brendan Coles がそれを [Metasploit モジュール](https://www.exploit-db.com/exploits/44654)（`exploit/linux/local/af_packet_packet_set_ring_priv_esc`）に移植し、研究上のバグを、フレームワークを動かす誰もが使えるワンコマンドのローカル root ツールに変えた。実際の標的に対する悪用で捕捉されたという公開の証拠はない（CISA の KEV カタログには掲載されていない）が、「確認された攻撃はない」と「攻撃する現実的な方法がない」は異なる主張である — このバグについては、後者が開示から数か月以内に閉じられた。

## なぜ起きたか {#why-it-happened}

攻撃者が影響を与えられる2つの符号なしの値の減算に対して適用された、たった1つの符号なしから符号付きへのキャストが、境界チェックを無力化するのに十分だった。チェックの作者は、「負またはゼロ」がそのまま「小さすぎる」を意味する符号付きの領域で比較が行われているかのように推論しており、符号なしの大きなラップアラウンドが `int` にキャストされたときにどう再解釈されるかを考慮していなかった。

## 解決 {#resolution}

Andrey Konovalov が作成した [`2b6867c2ce76`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2b6867c2ce76c596676bec7d2d525af525fdc6e2)（"net/packet: fix overflow in check for priv area size"）は、符号付きのキャストを取り除き、代わりに比較の前に `tp_sizeof_priv` を `u64` にキャストする — 両辺を、ラップアラウンドがもはや起こりえない、十分に幅が広く符号の扱いが正しい領域で比較するのである。

```c
if (po->tp_version >= TPACKET_V3 &&
    req->tp_block_size <=
          BLK_PLUS_PRIV((u64)req_u->req3.tp_sizeof_priv))
        goto out;
```

修正は2017年3月にメインラインに入り、stable カーネルにバックポートされた。ディストリビューションはその後まもなく、4.10.x およびそれ以前の stable ブランチでこれを出荷した。

Konovalov は[当初この修正を](https://lore.kernel.org/netdev/cover.1490709552.git.andreyknvl@google.com/)、AF_PACKET のリングバッファのコード全体にわたる複数のオーバーフローと符号の問題に対処する5パッチのシリーズの一部として投稿した。netdev でレビューした Willem de Bruijn は、その範囲に異議を唱えた。"These are a lot of changes to backport to stable kernels. Can we separate the minimal patch set needed to address known overflow to send to net... and follow up with the larger cleanup to net-next."（訳: stable カーネルにバックポートするには変更が多すぎる。既知のオーバーフローに対処するのに必要な最小限のパッチセットを分けて net に送り……より大きなクリーンアップは後から net-next に送れないか）。Konovalov は同意し、5つのパッチのうち2つ — `tp_frame_size` のチェックと並べ替えのクリーンアップ — をシリーズから外し（"remove cleanup patches, will send in a separate patchset"）、翌日、実際に stable に届く必要のある最小限の `net` 向け投稿として、[3パッチに絞った v2 を再投稿した](https://lore.kernel.org/netdev/cover.1490796500.git.andreyknvl@google.com/)。David Miller はその翌日にそれを適用した。

## 得られた教訓 {#what-it-taught-us}

**減算を囲む符号付き/符号なしのキャストは、繰り返し現れるバグクラスである。** 「2つの符号なしの値の差を `int` にキャストして符号をチェックする」というのは、正しく見え自然に読めるパターンだが、実際に機能するのは差が決してラップアラウンドしない場合だけである — そしてその性質は、どちらかのオペランドが攻撃者の影響を受ける場合には常に、仮定するのではなく証明しなければならない。

**ケーパビリティチェックの強さは、その到達可能性の強さで決まる。** `CAP_NET_RAW` は、非特権ユーザー namespace のない世界では、実在する意味のある関門だった。非特権での namespace 作成が一般的になると、同じケーパビリティチェックは「信頼された呼び出し側」を意味しなくなった。

**リングバッファのジオメトリのバグは、本質的にサイズ不一致のバグである。** サイズを別途検証し、後でそのサイズに依存してオフセットを計算するコードはどれも、検証と後の使用とが演算の領域（符号付きか符号なしか、ビット幅）についてビット単位で一致していなければ、まさにこのクラスのバグに対して脆弱である。

!!! warning "注意すべきパターン"
    境界チェックが後に続く場所で、2つの `unsigned`/`size_t`/`u32`/`u64` の値の減算を囲む `(int)` キャストを grep すること — まさにこの形（`(int)(a - b) <= 0`）が CVE-2017-7308 を引き起こしたものであり、カーネル全体で繰り返し現れる。あわせて確認する価値があるのは `sysctl kernel.unprivileged_userns_clone`（またはディストリビューションでの同等の設定）である — 非特権での namespace 作成が有効なら、`CAP_NET_RAW`/`CAP_NET_ADMIN` で守られたすべてのローカルバグを、コンテナのワークロードからリモートに到達可能なものとして扱うこと。

## 関連項目 {#see-also}

- [AF_PACKET 生ソケット](../packet-socket.md) — このバグの背後にあるリングバッファの仕組み
- [ネットワーク namespace](../net-namespaces.md) — 非特権のユーザー+ネットワーク namespace が、`CAP_NET_RAW` で守られたコードパスをどのように露出させるか
- [Netfilter x_tables のヒープオーバーフロー](netfilter-xtables.md) — コンテナエスケープのプリミティブになった、ケーパビリティで守られたもう1つのローカルバグ
- [UDP UFO のパス切り替えによる破損](udp-ufo.md) — 同時期に syzkaller で発見されたもう1つのバグ

## 外部参考文献 {#external-references}

- [NVD: CVE-2017-7308](https://nvd.nist.gov/vuln/detail/CVE-2017-7308) — AF_PACKET TPACKET_V3 の CVE レコード
- [git.kernel.org: 2b6867c2ce76](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2b6867c2ce76c596676bec7d2d525af525fdc6e2) — "net/packet: fix overflow in check for priv area size"
- [lore.kernel.org: net/packet: fix multiple overflow issues in ring buffers (v1)](https://lore.kernel.org/netdev/cover.1490709552.git.andreyknvl@google.com/) — 最小限のオーバーフロー修正を、より広範なリングバッファのクリーンアップから分離するよう Willem de Bruijn が求めたことを示すカバーレターのスレッド
- [lore.kernel.org: v2、3パッチに絞ったシリーズ](https://lore.kernel.org/netdev/cover.1490796500.git.andreyknvl@google.com/) — David Miller が実際にマージした再投稿
- [Project Zero: Exploiting the Linux kernel via packet sockets](https://projectzero.google/2017/05/exploiting-linux-kernel-via-packet.html) — Andrey Konovalov による CVE-2017-7308 の解説記事
- [Exploit-DB 41994](https://www.exploit-db.com/exploits/41994) と [44654](https://www.exploit-db.com/exploits/44654) — Project Zero の解説記事とともに公開された Konovalov 自身の C エクスプロイトと、Brendan Coles によるその Metasploit への移植
