# Towelroot：ペアの requeue 側に欠けていたチェック {#towelroot-the-missing-check-on-the-requeuers-half-of-the-pair}

> CVE-2014-3153 — `futex_requeue()` は PI-requeue の移動元と移動先が本当に別の futex かどうかを一切チェックしておらず、その2年前に*兄弟*関数へ入った同一のチェックでは、この問題を捕捉できなかった

公開日
:   2014年6月2日（Kees Cook が Red Hat に報告）。2014年6月5日にメインラインで修正

報告者
:   Pinkie Pie（仮名）。Google の Kees Cook 経由で報告

CVSS
:   7.8 HIGH / 7.2（CVSS v2）（`CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`）

バグの混入時期
:   `FUTEX_CMP_REQUEUE_PI` のサポート導入時、2009年頃

修正
:   commit `e9c243a5a6de`、メインライン Linux 3.15（2014年6月8日）

エクスプロイトツール
:   あり — George Hotz の「Towelroot」。公開されたワンクリックの Android root 化ツール

実際の悪用
:   あり — 2022年5月25日に CISA KEV に追加。Google 自身の2014年のテレメトリでは、この年に最も深刻に悪用された Android カーネルバグとして名指しされた

*[事件簿：ロックと同期のバグ](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

`FUTEX_CMP_REQUEUE_PI` は、ある futex で待機しているスレッドを、先に起床させることなく、PI 対応の2つ目の futex へ移す。これは glibc が優先度継承 mutex 上の `pthread_cond_wait()` を効率よく実装するために使う仕組みである。待機側スレッド（T1）は `FUTEX_WAIT_REQUEUE_PI(uaddr, uaddr2)` を呼んでブロックし、その際 `struct rt_mutex_waiter` を自分自身のカーネルスタック上に割り当てる。続いて2つ目のスレッド（T2）が `FUTEX_CMP_REQUEUE_PI(uaddr, uaddr2)`、すなわち `futex_requeue()` を呼ぶ。これは移動元 futex の waiter リストをたどり、一致したものごとに `rt_mutex_start_proxy_lock()` を呼んで、T1 のスタック上にある waiter を `uaddr2` の実際の PI mutex に、T1 に代わって、完全に T2 のコンテキストから接続する。このプロトコル全体は、明文化されていない1つの不変条件に依存している。`uaddr` と `uaddr2` が本当に別の futex であること — 一方は通常の futex、もう一方は PI 対応の futex — である。`futex_requeue()` の中には、それをチェックするものが何もなかった。

## きっかけ {#the-trigger}

兄弟関数の `futex_wait_requeue_pi()` には、まさにこの穴に対するパッチがすでに入っていた — 2年前に。2012年、Dave Jones の syscall ファザー `trinity` が、`uaddr == uaddr2` で `FUTEX_WAIT_REQUEUE_PI` を呼ぶと NULL ポインタ参照が起きることを発見した。移動元と移動先が同一だと `q.key` と `key2` が等しいと判定され、早期起床のチェックが誤動作し、コードは実際には一度もセットアップされていない `pi_mutex` を参照してしまう。Darren Hart の修正 `6f7b0a2a5c0f` は `if (uaddr == uaddr2) return -EINVAL;` を追加した — ただし、ペアの*待機*側である `futex_wait_requeue_pi()` にだけである。実際に `FUTEX_CMP_REQUEUE_PI` の移し替えを行う*requeue*側の `futex_requeue()` は、同一の不変条件を持つにもかかわらず、同等のガードを一度も受け取らなかった。

## 観測された挙動 {#observed-behavior}

ガードのない requeue 側のパスに `uaddr1 == uaddr2` を強制的に通すと、requeue 機構の帳簿と、`rt_mutex_start_proxy_lock()` が対象 mutex の waiter 構造体に実際に行ったこととの同期が崩れる。このバグに関する公開された技術解析によれば、実際の結果として、`futex_wait_requeue_pi()` が PI mutex の待機リストから `rt_waiter` を一度も取り除かないまま終了しうる。`rt_waiter` は T1 のカーネルスタック上にある — その syscall が戻った瞬間に再利用される領域である。PI mutex 自身の waiter 追跡構造体は、もはや古くなったスタックメモリを指すポインタを保持したまま残される。その後の syscall に、まさにそのスタックスロットを攻撃者が制御するデータで再利用させることができれば、攻撃者は事実上、偽の `rt_mutex_waiter` — タスクポインタやリストのリンクまで含めて — を、カーネル自身の PI ロック受け渡しコードが後で正当なものとして参照するアドレスに偽造できる。この破損ポインタのプリミティブこそが、公開されたエクスプロイト解説（後に移植された Exploit-DB 上の PoC を含む）が、任意のカーネルメモリ読み書きに変え、さらにそこからカーネルのクレデンシャル構造体を上書きして完全な root シェルを得る、と説明しているものである。

## なぜ起きたか {#why-it-happened}

2012年の修正は、`trinity` が実際に引き起こしたクラッシュ — `futex_wait_requeue_pi()` 内の NULL 参照 — に範囲を絞ったものだった。それは実在し再現されたバグに対する、正しく狭い修正だった。当時、誰もこの不変条件を第一原理から導き直し、「このペアのすべての関数に同じチェックが必要ではないか」と問わなかった。requeue 側も同一の前提条件を持ち、それが破られたときの失敗モードも同一だったが、最初のバグを見つけたファジングの実行中にそれ自体でクラッシュすることはなかった — ファザーが2度目に偶然たどり着くのを待つのではなく、誰かが鏡像の穴を意図的に探しに行くまで、さらに2年間ガードのないまま放置されていた。

## 解決 {#resolution}

`e9c243a5a6de`（"futex: Forbid uaddr == uaddr2 in futex_requeue(..., requeue_pi=1)"）は、欠けていた `if (uaddr1 == uaddr2) return -EINVAL;` を `futex_requeue()` に追加し、`futex_wait_requeue_pi()` がすでに持っていたチェックと揃えた。Thomas Gleixner のレビューは、生のポインタ比較では実際には不十分であることを見抜いた。異なる2つの仮想アドレス（同一プロセス内、あるいは共有 futex ならプロセスをまたいで）が、同じ下位の page とオフセットに解決されうるからである。このコミットは2つ目のチェックとして、*解決後の* `union futex_key` の値を `match_futex()` で比較する処理を追加し、さらに同じキー比較を、それまで生のポインタしか比較していなかった `futex_wait_requeue_pi()` の2012年の元の修正にも後付けで適用した。

## 得られた教訓 {#what-it-taught-us}

**ファザーが見つけたクラッシュの修正は、ファザーが偶然突き当たった不変条件の修正と同じではない。** `trinity` は、「移動元と移動先は異なっていなければならない」という条件に対する1つの関数の違反を見つけた。この不変条件は、誰もファジングでクラッシュさせなかった2つ目の関数にも等しく当てはまっていた — そして、範囲を狭く絞った修正には、同じ前提条件が別の場所にも存在することを浮かび上がらせる仕組みがない。

**ポインタの等価性は、それが代理している性質よりも弱いチェックである。** 最終的な完全な修正でさえ、2回目の見直しを必要とした。`uaddr == uaddr2` の比較で明らかなケースは捕まえられるが、本当の不変条件はアドレスが解決される先の *futex の同一性*に関するものであり、完全に塞ぐにはポインタではなくキーを比較する必要があった。

!!! warning "注意すべきパターン"
    2つの関数が同じプロトコルの反対側をそれぞれ実装している場合（ここでは PI-futex requeue の待機側と requeue 側）、一方の側で前提条件が発見され修正されたことは、もう一方の側に同一の穴がないか監査すべき強いシグナルである — 目の前のクラッシュを直して先へ進むだけで終わらせてはならない。鏡像の関数はしばしばまったく同じ不変条件を共有しており、コンパイラもファザーもその2つを結び付けてはくれない。

## 関連項目 {#see-also}

- [ロックの概要](../README.md) — futex、rt_mutex、そしてこのバグが存在する PI-requeue プロトコル
- [PI-Mutex の起源](pi-mutex-origin.md) — そもそもなぜ PI futex と `FUTEX_LOCK_PI`/`FUTEX_CMP_REQUEUE_PI` が存在するのか
- [恒久的なフォールトへの答えを持たなかった PI-Futex の修正処理](pi-futex-fixup-owner-uaf.md) — requeue 側ではなくフォールト処理側で起きた、2つ目の PI-futex の use-after-free

## 外部参考文献 {#external-references}

- [git.kernel.org: e9c243a5a6de](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e9c243a5a6de0be8e584c604d353412584b592f8) — "futex: Forbid uaddr == uaddr2 in futex_requeue(..., requeue_pi=1)"、修正
- [git.kernel.org: 6f7b0a2a5c0f](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6f7b0a2a5c0fb03be7c25bd1745baa50582348ef) — "futex: Forbid uaddr == uaddr2 in futex_wait_requeue_pi()"、兄弟関数だけをカバーした2012年の修正
- [Debian DSA-2949-1](https://lists.debian.org/debian-security-announce/2014/msg00130.html) — "Pinkie Pie discovered an issue in the futex subsystem that allows a local user to gain ring 0 control via the futex syscall"
- [Google Android Security 2014 Year in Review](https://source.android.com/docs/security/reports/Google_Android_Security_2014_Report_Final.pdf) — CVE-2014-3153 を、その年に最も深刻に悪用された Android のローカル権限昇格バグとして挙げている
- [NVD: CVE-2014-3153](https://nvd.nist.gov/vuln/detail/CVE-2014-3153) — CVE レコード。2022年5月25日に CISA KEV に掲載
