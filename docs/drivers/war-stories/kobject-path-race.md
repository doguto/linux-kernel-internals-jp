# ロックではなくリトライが必要だったkobjectパスの競合 {#the-kobject-path-race-that-needed-a-retry-not-a-lock}

> CVE-2023-45863 — `kobject_get_path()`はsysfsパスの長さを1回目のパスで計算し、同期のない2回目のパスでバッファを埋めていた。その2つのパスの間にネットワークインタフェースのリネームが割り込むと、名前が十分に長くなって小さすぎるバッファをあふれ、そのままヒープのslabに書き込むことがあった

公開
:   2023年10月14日（NVDでCVE-2023-45863が公開。バグの発見・報告・修正はそれより数か月前の2022年12月〜2023年1月に行われた。後述）

報告者
:   Wang Hai, Huawei（KASANによる再現）

CVSS
:   6.4 MEDIUM（CVSS 3.1、`AV:L/AC:H/PR:H/UI:N/S:U/C:H/I:H/A:H`）

バグの混入時期
:   少なくともLinux 2.6.12-rc2（2005年4月。カーネルのgitで追跡されている履歴の最も古い時点）から。`kobject_get_path()`の長さ計算/埋め込みの2パス設計は、git履歴の最初の時点ですでに存在していた

修正バージョン
:   コミット`3bb2a01caa81`、2023年1月11日にマージ。stableへはLinux 6.2.3（2023年3月10日）としてバックポートされた。メインラインに初めて入ったのはLinux 6.3（2023年4月23日）

エクスプロイトツール
:   公開された概念実証は見つかっていない

実際の悪用
:   確認された事例なし（CISA KEVに未掲載）

*[事件簿: デバイスドライバ](../war-stories.md)の一部。*

## 発生前の状態 {#before-state}

カーネルがユーザー空間に送るすべてのuevent（`add`、`remove`、`change`、`move`）は`DEVPATH`フィールドを持つ。これはsysfsツリーにおけるデバイスの位置で、例えば`/devices/pci0000:00/.../net/eth0`のようなものである。`udevd`/`systemd-udevd`はこの文字列を`/sys`と連結して、イベントがどのsysfsノードを指しているかを正確に知り、さらに属性を読み、命名規則や権限規則を適用し、対応する`/dev`ノードを作成する。`kobject_get_path()`はこの文字列を、kobjectの`parent`チェーンをルートまでたどることで構築するが、それを2つの別々のパスで行う。`get_kobj_path_length()`はチェーンを1回たどり、すべての祖先について`strlen(kobject_name(parent)) + 1`を合計して、必要なバッファサイズを正確に計算する。続いて`kobject_get_path()`がちょうどそのサイズのバッファを割り当てて`fill_kobj_path()`を呼び出し、これが*同じ*チェーンをもう一度たどって、各祖先の名前を右から左へバッファに書き込む。2つのパスをまたいでロックを保持するものは何もない。コードは、チェーンとその中のすべての名前が2つのパスの間で変わらないことを信頼している。

## きっかけ {#the-trigger}

ネットワークドライバをロードしている`ixgbe_probe()`は、`device_add()`を介して子kobject（NICのMIIバス）を登録する。これは、その子自身の`add` ueventのためにsysfsパスを構築する必要がある。CPU 0では、`get_kobj_path_length()`が子の親（netdev自身のkobjectで、例えば`eth0`という名前）をたどり、その名前に基づいて長さを計算する。同時にCPU 1では、`systemd-udevd`が（少し前にnetdev自体について送られた`add` ueventに反応して）リネームを発行する。例えば、`eth0`を`enp3s0f0`のようなより長い名前に変える永続的なインタフェース命名規則を適用する場合である。`kobject_rename()`は`kobj->name`を新しい、より長い文字列を指すように差し替える。そのドキュメントコメントは、同期の責任を完全に呼び出し元に負わせている: "It is the responsibility of the caller to provide mutual exclusion between two different calls of `kobject_rename` on the same kobject."（訳: 同じkobjectに対する2つの異なる`kobject_rename`の呼び出しの間で相互排他を提供するのは、呼び出し元の責任である。）ネットワークインタフェースの場合、その相互排他は呼び出し元（`dev_change_name()`）がすでに保持しているRTNLロックによってもたらされる。これはリネーム同士を直列化するが、RTNLやその他のいかなるロックも一切取らない`get_kobj_path_length()`/`fill_kobj_path()`のような無関係な読み手とリネームとの直列化には何の役にも立たない。リネームが2つのパスの間に入ると、`fill_kobj_path()`の2回目のたどりはその祖先について*新しい*、より長い名前を見る一方で、*古い*、より短い名前から計算された長さを使って処理することになる。これを修正したコミットは、このインターリーブを直接図示している。

```
cpu0                                         cpu1
ixgbe_probe
 register_netdev(netdev)
  netdev_register_kobject
   device_add
    kobject_uevent // Sending ADD events
                                             systemd-udevd // rename netdev
                                              dev_change_name
                                               device_rename
                                                kobject_rename
 ixgbe_mii_bus_init                             |
  mdiobus_register                              |
   __mdiobus_register                           |
    device_register                             |
     device_add                                 |
      kobject_uevent                            |
       kobject_get_path                         |
        len = get_kobj_path_length // old name  |
        path = kzalloc(len, gfp_mask);          |
                                                kobj->name = name;
                                                /* name length becomes
                                                 * longer
                                                 */
        fill_kobj_path /* kobj path length is
                        * longer than path,
                        * resulting in out of
                        * bounds when filling path
                        */
```

`fill_kobj_path()`は独自の境界を持っていなかった。渡された`length`引数を信頼し、各祖先の`kobject_name()`がその時点で返す名前の`strlen()`を盲目的に差し引いていき、小さすぎるバッファの先頭を越えて、その開始位置より前に`memcpy`していた。

## 観測された挙動 {#observed-behavior}

これは、書き込みオフセットと内容の両方がリネームされたインタフェースの新しい名前によって決まるヒープバッファオーバーフローである。単なる読み取りではなく、slab割り当ての先頭より前への書き込みである。`rmmod ixgbe; sleep 0.5; modprobe ixgbe; sleep 0.5`というストレステストのループから得られたWang HaiのKASANレポートは、これを直接捕捉した。

```
BUG: KASAN: slab-out-of-bounds in fill_kobj_path+0x50/0xc0
Write of size 7 at addr ff1100090573d1fd by task kworker/28:1/673

 Workqueue: events work_for_cpu_fn
 Call Trace:
 <TASK>
 dump_stack_lvl+0x34/0x48
 print_address_description.constprop.0+0x86/0x1e7
 print_report+0x36/0x4f
 kasan_report+0xad/0x130
 kasan_check_range+0x35/0x1c0
 memcpy+0x39/0x60
 fill_kobj_path+0x50/0xc0
 kobject_get_path+0x5a/0xc0
 kobject_uevent_env+0x140/0x460
 device_add+0x5c7/0x910
 __mdiobus_register+0x14e/0x490
 ixgbe_probe.cold+0x441/0x574 [ixgbe]
 local_pci_probe+0x78/0xc0
 work_for_cpu_fn+0x26/0x40
 process_one_work+0x3b6/0x6a0
 worker_thread+0x368/0x520
 kthread+0x165/0x1a0
 ret_from_fork+0x1f/0x30
```

## なぜ起きたか {#why-it-happened}

`get_kobj_path_length()`と`fill_kobj_path()`は、暗黙の前提の下で書かれていた。kobjectの祖先チェーンとすべての祖先の名前が、長さの計算とそれに続く埋め込みの間で変わらないという前提である。この前提は一般的なケースでは妥当である（ほとんどのkobjectはリネームされない）が、コード内でそれを強制するものは何もなかった。そして、この同じprobeシーケンスが送った*最初の*ueventに対するudev自身の反応として引き起こされるネットワークインタフェースのリネームは、まさに2パス設計では考慮しようのない種類の並行イベントだった。このバグは、競合の両側が現実的ではあるが特殊ではない特権を持つことを必要とした。再現手順で使われたprobe/リロードのパスを引き起こすには`CAP_SYS_MODULE`（モジュールのロード/アンロード）が必要で、rtnetlinkまたはレガシーな`SIOCSIFNAME` ioctlでインタフェースのリネームを発行するには`CAP_NET_ADMIN`が必要である。これらはrootとして動作する`systemd-udevd`が当然のこととしてすでに持っているケーパビリティである。Wang Haiがマージの判断を催促したとき、Greg Kroah-Hartmanの返信はまさにこの点を比較考量していた: "It's in my 'to review' queue that I am working on. As this is not anything that a normal user can trigger, it's not that high of a priority, right?"（訳: 今取り組んでいる「レビュー予定」キューに入っている。これは一般ユーザーが引き起こせるものではないから、それほど優先度は高くないよね？）これは、バグのトリガー面がrootに限られることが、バグが本物かどうかではなく、どれだけ急いで扱われるかを左右したことを明示的に認めたものである。

## 解決 {#resolution}

`3bb2a01caa81`（"kobject: Fix slab-out-of-bounds in fill_kobj_path()"、Wang Hai）は、2つのパスをまたぐロックを追加しない。代わりに、`fill_kobj_path()`自身に境界チェックをさせ、チェーンが足元で変わった場合は最初からリトライさせる。

```diff
-static void fill_kobj_path(const struct kobject *kobj, char *path, int length)
+static int fill_kobj_path(const struct kobject *kobj, char *path, int length)
 {
 	const struct kobject *parent;
 	--length;
 	for (parent = kobj; parent; parent = parent->parent) {
 		int cur = strlen(kobject_name(parent));
 		length -= cur;
+		if (length <= 0)
+			return -EINVAL;
 		memcpy(path + length, kobject_name(parent), cur);
 		*(path + --length) = '/';
 	}
 	...
+	return 0;
 }

 char *kobject_get_path(const struct kobject *kobj, gfp_t gfp_mask)
 {
+retry:
 	len = get_kobj_path_length(kobj);
 	...
 	path = kzalloc(len, gfp_mask);
 	...
-	fill_kobj_path(kobj, path, len);
+	if (fill_kobj_path(kobj, path, len)) {
+		kfree(path);
+		goto retry;
+	}
 	return path;
 }
```

`fill_kobj_path()`のすべての書き込みの前に、`length`が0以下になっていないかのチェックが入るようになった。そうなりそうになった時点で、関数は範囲外に書き込む代わりに`-EINVAL`で抜け出し、`kobject_get_path()`はそれを割り当てたバッファが古くなったというシグナルとして扱い、バッファを解放して、長さの計算から埋め込みまでの全体を`retry:`からやり直す。依然としてロックはない。十分に執拗なリネームの連続があれば、原理的には何度もリトライし続ける可能性がある。レビュアーのMichal Swiatkowskiはまさにそれを尋ねた: "I wonder if there is no case we end up with infinite loop (fill_kobj_path always returning error). Do You know?"（訳: 無限ループ（fill_kobj_pathが常にエラーを返す）に陥るケースはないのだろうか。ご存じだろうか？）Wang Haiの回答: "It should only be possible to have an infinite loop if name or parent keeps changing. The probability of this is extremely low. If necessary, change it to only retry 3 times?"（訳: 無限ループになりうるのは、名前か親が変わり続ける場合だけのはずだ。その確率は極めて低い。必要なら、リトライを3回までに変更しようか？）リトライ回数の上限は、マージされた修正には結局追加されなかった。`kobject_get_path()`は不一致を検出するたびに無条件にリトライする。

## 教訓 {#what-it-taught-us}

**「測ってから埋める」2パスのパターンには、両方のパスをまたぐロックか、埋め込みパスが測定値の古さを検出する手段のどちらかが必要である。測定値が有効なままであることを単純に信頼することはできない。** ここで選ばれた修正はロックではなかった。きれいに失敗して呼び出し元にリトライさせる自己チェック型の埋め込みパスによって、すでに多くのコンテキストから到達可能な関数に新しいロックを導入することなく、同等の安全性を実現した。

**バグの現実的な緊急度とその正しさは別の問題であり、メンテナは後者を否定することなく前者を比較考量していることを明言している。** `dev->driver`に隣接するバグがrootでしか起こせないということは、このバグが本物のヒープ範囲外書き込みであることを何ら損なうものではなかった。それが影響したのはレビューされる速さであり、修正が必要かどうかではなかった。

!!! warning "注意すべきパターン"
    同じ可変データに対して、1回目のパスでバッファサイズを計算し、別のパスでそのバッファを埋める関数は、その間に何も変わらないという前提と同程度にしか安全ではない。データが並行して変わりうるのに両方のパスをまたいでロックを取るのが現実的でないなら、埋め込みパスは古い測定値を検出して安全に失敗する必要がある。渡された数値を信頼してはならない。

## 関連項目 {#see-also}

- [Linuxデバイスモデル](../device-model.md) — `struct kobject`、sysfs、およびこのバグが存在するueventの仕組み
- [The uevent_show() Fix That Deadlocked Driver Detach](uevent-show-deadlock.md) — 同じuevent生成のコードパスにある別のドライバコアの競合。そのデッドロックは以前の競合を修正するために追加されたロックに起因し、最終的にはそのロックを取り除いてRCUに置き換えることで解決された
- [Netfilter x_tablesのヒープオーバーフロー](../../net/war-stories/netfilter-xtables.md) — 別のサブシステムにおける、未チェックの長さを介したヒープオーバーフローという比較可能なバグ

## 外部参考資料 {#external-references}

- [git.kernel.org: 3bb2a01caa81](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3bb2a01caa813d3a1845d378bbe4169ef280d394) — "kobject: Fix slab-out-of-bounds in fill_kobj_path()"。修正コミットで、コミットメッセージに競合の全体図とKASANレポートを含む
- [lore.kernel.org: v2パッチスレッド](https://lore.kernel.org/r/20221220012143.52141-1-wanghai38@huawei.com/) — Wang Haiの投稿と、Michal Swiatkowski / Greg Kroah-Hartmanとの議論
- [NVD: CVE-2023-45863](https://nvd.nist.gov/vuln/detail/CVE-2023-45863) — CVEレコード。CVSS 3.1 6.4 MEDIUM、2023年10月14日公開
- [Debian LTSアドバイザリ DLA-3710-1](https://lists.debian.org/debian-lts-announce/2024/01/msg00004.html) — この欠陥についての独立した説明
