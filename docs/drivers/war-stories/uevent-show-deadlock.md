# ドライバのデタッチをデッドロックさせたuevent_show()の修正 {#the-uevent_show-fix-that-deadlocked-driver-detach}

> CVE-2024-44952（却下）— 本物の`dev->driver` NULLポインタ競合に対する2024年の修正は、sysfsの読み取りパスに`device_lock()`を追加し、デッドロックを再び生み出した。`device_lock()`はlockdepのサイクル検出器から意図的に除外されているためlockdepにはそれが見えず、デバイスのmutexに独自のローカルlockdepキーを持つサブシステムだけがそれを捕捉できた

公開
:   2024年9月4日（"Linux" CNAがCVE-2024-44952を公開。後に却下された。後述）

報告者
:   syzbot（ファザーによる発見）。根本原因の特定はTetsuo Handa

CVSS
:   5.5 MEDIUM（CVSS 3.1、`AV:L/AC:L/PR:L/UI:N/S:U/C:N/I:N/A:H`。却下前に短期間表示されていたNVD算出のスコアで、現在のNVDレコードには存在しない）

バグの混入時期
:   コミット`c0a40097f0bc`、2024年5〜6月、Linux 6.10で初めて出荷

修正バージョン
:   コミット`15fffc6a5624`、2024年7月、Linux 6.11で初めて出荷

CVEのステータス
:   割り当てた"Linux" CNAによって2024年11月9日に却下された。却下レコードには定型文（"withdrawn by its CVE Numbering Authority"）しかなく、バグ固有の理由は示されていない

エクスプロイトツール
:   なし。影響はカーネルのデッドロック（サービス拒否）であり、メモリ破壊やコード実行ではない

実際の悪用
:   確認された事例なし（CISA KEVに未掲載）

*[事件簿: デバイスドライバ](../war-stories.md)の一部。*

## 発生前の状態 {#before-state}

`device_lock()`は`dev->mutex`のラッパーで、デバイスのコア状態を保護する。その中には、どのドライバが（もしあれば）現在デバイスを所有しているかを記録する`dev->driver`ポインタも含まれる。このロックはドライバコア、sysfs、個々のバス/サブシステムのコードにわたってさまざまな順序で取得され、カーネル自身が認めているように、lockdepの通常のサイクル検出にはなじまない。`dev->mutex`は`lockdep_set_novalidate_class()`で初期化されており、これはlockdepに対し、このロックについては通常の取得順序の記録を完全にスキップするよう指示する。したがって、`device_lock()`に関連するロック順序の問題のほとんどは、設計上lockdepからは見えない。ただし、サブシステムが特定のデバイスのmutexに独自の明示的なローカルlockdepキーを割り当て、そのデバイスクラスだけを検証対象に戻す場合は別である。CXLのACPIドライバのコードは、バインドするプラットフォームデバイスに対してまさにそれを行っている（`drivers/cxl/acpi.c`の`device_lock_set_class(&pdev->dev, &cxl_root_key)`）。この話のデッドロックが原因不明のハングではなくlockdepのスプラットを生み出した唯一の理由はそれである。

`uevent_show()`はデバイスの`uevent`属性を支えるsysfsの読み取りハンドラである。これは`dev_uevent()`を呼び出し、カーネルのuevent netlinkソケットを通じてユーザー空間に送られるのと同じkey=value形式の環境（`DEVPATH=`、`DEVTYPE=`など）を構築する。その中には、ドライバがバインドされている場合に`dev->driver->name`から取られる`DRIVER=%s`も含まれる。`dev->driver`自体は、ここで重要な2か所で書き込まれる。`really_probe()`はprobeが成功したときにそれを設定し、probeが失敗したときには`device_unbind_cleanup()`を介して`NULL`に戻す。そして`device_release_driver_internal()`は通常のドライバのデタッチ時にそれをクリアする。どちらも`device_lock()`の下で行われる。

## きっかけ {#the-trigger}

`device_lock()`を保持せずに`dev->driver`を読むことは、本物の、現に存在するバグだった。`dev_uevent()`は`if (dev->driver)`をチェックしてから、同期なしに`dev->driver->name`を参照していたため、別のCPU上で並行して行われる失敗したprobeのクリーンアップが、チェックと参照の間に`dev->driver`を`NULL`にクリアしうる。`c0a40097f0bc`（"drivers: core: synchronize really_probe() and dev_uevent()"、Dirk Behme、2024年5月）は、これを直接的な方法で修正した。`uevent_show()`から`dev_uevent()`への呼び出しを`device_lock()`/`device_unlock()`で囲み、`really_probe()`が自身の書き込みの周りですでに保持していたロックに合わせたのである。コミットメッセージは、まさにこのクラスの競合が2015年に一度報告され、そのときは修正されなかったことに触れている。また、*初期化*側（`dev->driver = drv`。これは`NULL`から非`NULL`への遷移しかしない）での関連していそうなNULL参照についての未解決のsyzbotレポートにも言及し、そのレポートは"can be considered to be false-positives"（偽陽性とみなせる）が、同じ変更によって付随的に修正されるはずだと述べている。

コミットメッセージが触れておらず、どうやらそれを確認した誰も気づかなかったのは、`device_lock()`が`uevent_show()`の呼び出しスタックの中で*もう一方*の方向、すなわちドライバの削除からすでに取得されていたことである。`device_release_driver_internal()`は（モジュールのアンロード時に`driver_detach()`経由で呼ばれる）、`device_del()`の呼び出しをまたいで`device_lock()`を保持しており、`device_del()`は`kernfs_remove_by_name_ns()`を呼び出してデバイスのsysfsエントリを削除する。その中にはデバイス自身の`uevent`属性ファイルも含まれる。kernfsノードの削除は、そのノードに対する実行中の読み取りが終わるのを待たなければならない。`uevent`属性に対する実行中の読み取りとはまさに`uevent_show()`のことであり、それは今や、ドライバのデタッチがその読み取りの完了を待ちながら保持している、まさにその`device_lock()`の取得を待っているのである。

## 観測された挙動 {#observed-behavior}

結果は、デバイスの`device_lock()`とkernfs内部のノードごとの`kn->active`参照カウントとの間の典型的なAB-BAデッドロックであり、CXLハードウェアでの再現とlockdepのスプラット（`Tainted: G OE N`、カーネル`6.10.0-rc7+`）によって捕捉された。

```
======================================================
WARNING: possible circular locking dependency detected
6.10.0-rc7+ #275 Tainted: G           OE    N
------------------------------------------------------
modprobe/2374 is trying to acquire lock:
ffff8c2270070de0 (kn->active#6){++++}-{0:0}, at: __kernfs_remove+0xde/0x220

but task is already holding lock:
ffff8c22016e88f8 (&cxl_root_key){+.+.}-{3:3}, at: device_release_driver_internal+0x39/0x210

which lock already depends on the new lock.

the existing dependency chain (in reverse order) is:

-> #1 (&cxl_root_key){+.+.}-{3:3}:
       __mutex_lock+0x99/0xc30
       uevent_show+0xac/0x130
       dev_attr_show+0x18/0x40
       sysfs_kf_seq_show+0xac/0xf0
       seq_read_iter+0x110/0x450
       vfs_read+0x25b/0x340
       ksys_read+0x67/0xf0
       do_syscall_64+0x75/0x190
       entry_SYSCALL_64_after_hwframe+0x76/0x7e

-> #0 (kn->active#6){++++}-{0:0}:
       __lock_acquire+0x121a/0x1fa0
       lock_acquire+0xd6/0x2e0
       kernfs_drain+0x1e9/0x200
       __kernfs_remove+0xde/0x220
       kernfs_remove_by_name_ns+0x5e/0xa0
       device_del+0x168/0x410
       device_unregister+0x13/0x60
       devres_release_all+0xb8/0x110
       device_unbind_cleanup+0xe/0x70
       device_release_driver_internal+0x1c7/0x210
       driver_detach+0x47/0x90
       bus_remove_driver+0x6c/0xf0
       cxl_acpi_exit+0xc/0x11 [cxl_acpi]
       __do_sys_delete_module.isra.0+0x181/0x260
       do_syscall_64+0x75/0x190
       entry_SYSCALL_64_after_hwframe+0x76/0x7e
```

`cxl_root_key`は、CXLがlockdepに推論させるためだけにルートデバイスのmutexに割り当てるローカルlockdepキーである。これはドライバコアのほとんどが選択しないオプトインである。というのも、`dev->mutex`は、ツリー全体で`device_lock()`が他のロックに対してどれほど一貫性なく順序付けられているかという理由から、まさに`lockdep_set_novalidate_class()`を持っているからである。そのローカルキーがなければ、まったく同じ形のデッドロックは他のどのデバイスクラスでも同じように現実のものでありながら、lockdepの依存関係グラフからは黙って欠落し、`rmmod`の下での原因不明のハングとしてしか再現できなかっただろう。

## なぜ起きたか {#why-it-happened}

独立にはそれぞれ妥当な2つの修正が衝突した。`really_probe()`は`dev->driver`を書き込むときにすでに`device_lock()`を保持していた。唯一の同期されていない読み手に`device_lock()`を追加するのは、本物のNULLポインタ競合に対する自然で範囲の狭い修正であり、「これは説明されている競合を修正するか」を確認するレビュアーが、呼び出しスタックの他の場所ですでにそのロックを保持している他のすべての呼び出し元を導き直す必要なく承認するような種類の変更である。`device_release_driver_internal()`が`device_del()`をまたいで`device_lock()`を使っているのは、この変更よりずっと前からのことで、目的も無関係である。ドライバのデタッチは、デバイスを解体する間、デバイスのドライババインディング状態の一貫性を保つためにこのロックを必要とする。どちらのコミットの作者も、それ単体で見れば自分の修正について間違ってはいなかった。デッドロックは完全に両者の相互作用の中に存在しており、それはノード削除時のkernfs自身の内部的な読み手待ちロジックによって媒介される。これは、`device_lock()`がnovalidateであるせいで、誰もlockdepが自動的に浮かび上がらせてくれることを当てにできない依存関係だった。

## 解決 {#resolution}

`15fffc6a5624`（"driver core: Fix uevent_show() vs driver detach race"、Dan Williams、2024年7月）は、`uevent_show()`から`device_lock()`/`device_unlock()`を完全に取り除き、ロック付きの読み取りをRCUで保護された読み取りに置き換える。`dev_uevent()`は今では`rcu_read_lock()`の下で`READ_ONCE()`を使ってドライバポインタを読む。

```c
 static int dev_uevent(const struct kobject *kobj, struct kobj_uevent_env *env)
 {
 	const struct device *dev = kobj_to_dev(kobj);
+	struct device_driver *driver;
 	int retval = 0;
 	...
-	if (dev->driver)
-		add_uevent_var(env, "DRIVER=%s", dev->driver->name);
+	/* module_remove_driver() と同期する */
+	rcu_read_lock();
+	driver = READ_ONCE(dev->driver);
+	if (driver)
+		add_uevent_var(env, "DRIVER=%s", driver->name);
+	rcu_read_unlock();
```

そして`uevent_show()`は、呼び出しの周りの`device_lock()`/`device_unlock()`のペアを完全に取り除く。書き込み側には、ドライバ/モジュールの解体時に実行される`module_remove_driver()`に対応する`synchronize_rcu()`が追加され、ドライバオブジェクト自体が消える前に、`dev->driver`に対する実行中のRCU保護された読み手がすべて終了していることを保証する。

```c
 void module_remove_driver(const struct device_driver *drv)
 {
 	if (!drv)
 		return;
 
+	/* dev_uevent() と同期する */
+	synchronize_rcu();
+
 	sysfs_remove_link(&drv->p->kobj, "module");
```

これにより、元の修正の目的（どの読み手も壊れた、あるいは解放された`dev->driver`を観測できない）を、sysfsの読み取りパスから`device_lock()`を一切取らずに維持できる。そのため、ドライバのデタッチのkernfs削除に対するAB-BAの順序付けは単純にもう存在しない。コミット自身の論拠は、そのコストについて現実的である。ドライバオブジェクトは長寿命でめったに登録解除されないので、モジュールの削除1回ごとに`synchronize_rcu()`のレイテンシを払うことは、まれで範囲の狭い競合をロックなしで安全にするためのトレードオフとして許容できる、というものである。

## 教訓 {#what-it-taught-us}

**lockdepの検証から除外されたロックは、デッドロックしうることをやめるわけではない。それについて教えてくれることをやめるのである。** `device_lock()`の`lockdep_set_novalidate_class()`が存在するのは、このロックがドライバコア全体であまりに一貫性なく使われているため、汎用的なサイクル検出では偽陽性を避けられないからだが、そのトレードオフは、その一貫性のない使い方のどこかで新たに持ち込まれた本物のデッドロックがデフォルトでは見えないことを意味する。今回のものが表面化したのは、CXLのドライバコードが独自にローカルlockdepキーで自分のデバイスクラスを検証対象に戻していたからにすぎない。ドライバコアのほとんどはそのようなキーを持たず、他の場所で同じ形の欠陥があっても、原因不明のハングとしてしか現れないだろう。

**ロックを追加して競合を修正することが安全なのは、周囲のコールグラフの中ですでにそのロックを保持している他のすべてのものを知っている場合に限られる。** 2024年のNULLポインタ修正は、対象とした競合については正しかったが、ロックを追加しても安全だという点では誤っていた。`device_lock()`は、修正の作者がわざわざ探しに行く明白な理由のない、まったく別のコールパス（ドライバのデタッチ）ですでに重要な役割を担っていたからである。

!!! warning "注意すべきパターン"
    データ競合を修正するためにロックの取得を追加する前に、修正しようとしているパス上ですでにそのロックを保持しているものだけでなく、コードベースの他の場所で、新しい取得箇所に到達しうるコードへの呼び出しをまたいでそのロックを保持している*他の*ものも確認すること。特に`device_lock()`のように、自動的なサイクル検出から意図的に除外されているロックではそうである。自分のサブシステムだけにスコープを限定したローカルlockdepキーは、ドライバコア自体が検証してくれないロックについて本物のシグナルを得るための一つの方法である。

## 関連項目 {#see-also}

- [Linuxデバイスモデル](../device-model.md) — `struct device`、`device_lock()`、そしてこのバグが存在するドライバコアのprobe/デタッチの仕組み
- [ロックではなくリトライが必要だったkobjectパスの競合](kobject-path-race.md) — 同じくsysfsに隣接するパスにある別のドライバコアの競合で、新しいロックを追加せずに修正されたもの
- [The Deadlock Detector That Scheduled While Atomic](../../locking/war-stories/rtmutex-deadlock-detector-atomic-sleep.md) — 別のサブシステムのロック規律のバグで、可視性の点ではこれとちょうど正反対のもの。あちらのバグはどの本番カーネルでも"scheduling while atomic"という派手なスプラットを引き起こしたが、こちらはサブシステムローカルのlockdepキーがなければ見えないままだった

## 外部参考資料 {#external-references}

- [git.kernel.org: c0a40097f0bc](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c0a40097f0bc81deafc15f9195d1fb54595cd6d0) — "drivers: core: synchronize really_probe() and dev_uevent()"。本物のNULLポインタ競合を修正する過程でデッドロックを持ち込んだ修正
- [git.kernel.org: 15fffc6a5624](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=15fffc6a5624b13b428bb1c6e9088e32a55eb82c) — "driver core: Fix uevent_show() vs driver detach race"。RCUベースの修正で、コミットメッセージにlockdepスプラットの全文を含む
- [NVD: CVE-2024-44952](https://nvd.nist.gov/vuln/detail/CVE-2024-44952) — 却下ステータスを示す現在のレコード
- [Wayback Machine: NVD CVE-2024-44952, October 22, 2024](https://web.archive.org/web/20241022050517/https://nvd.nist.gov/vuln/detail/CVE-2024-44952) — 却下前のCVSS 3.1 5.5 MEDIUMのスコアを示すアーカイブスナップショット
- [カーネルドキュメント: driver model overview](https://docs.kernel.org/driver-api/driver-model/overview.html) — `struct device`と、ドライバコアにおける`device_lock()`の役割
