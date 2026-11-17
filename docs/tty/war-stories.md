# 事件簿: TTY/シリアルのバグとリグレッション {#war-stories-ttyserial-bugs-and-regressions}

> TTY層、N_TTY、ptyサブシステムで起きた6件の事件。うち5件は合わせて6つのCVE識別子を持ち、残る1件はよく文書化されたCVEなしのライフタイムバグである。大半はロックとライフタイムの競合であり、その中には、書き込み側のパスが独立に起動される読み取り/エコー側のパスと競合し、しかも両者の間で管理情報の整合性を保たなければならないコードで起きた1組が含まれる

[TTY層](README.md)、[シリアルコアとUARTドライバ](serial.md)、[ラインディシプリン、termios、擬似端末](line-disciplines.md)は、現在動いているこのサブシステムの姿を説明している。このページは、そのアーキテクチャの背後にある事件の記録である。そしてこのサブシステムは、難解で何年も続くロックのバグを異例なほど多く抱えており、その一部は実際に解決するまでに複数回の試みを要した。

## 詳細解説 {#deep-dives}

### [n_ttyエコー競合: 修正からCISAの悪用確認リスト入りまで9年](war-stories/n-tty-echo-race.md) {#the-n_tty-echo-race-nine-years-from-fix-to-cisas-exploited-in-the-wild-list}
**2014年5月 · CVE-2014-0196 · CISA Known Exploited Vulnerabilitiesカタログ（2023年追加）**
ラインディシプリンが入力をエコーで送り返している最中にptyへ書き込むと、2つの書き込み主体が、間に何のロックもないまま同じttyバッファを破壊できた。2014年に修正されたが、9年後になってもAndroidのroot化エクスプロイト手法として現役で使われていた。

### [修正され、修正が取り消され、再び修正されたpty_write()のロック](war-stories/pty-write-lock-saga.md) {#the-pty_write-lock-that-was-fixed-un-fixed-and-fixed-again}
**2018年 → 2020年 → 2022年7月 · CVE-2022-1462**
2018年のデータ競合の修正でロックが追加された。2020年のデッドロック修正がその一部を取り除き、競合を新しい形で黙って再導入した。2022年のCVE報告でようやくそれが捕捉され、本当の修正には両方の性質を同時に満たすことが必要だった。

## 短い事例 {#quick-cases}

### 事例1: n_hdlcの二重解放 — CVE-2017-2636 {#case-1-the-n_hdlc-double-free-cve-2017-2636}

`n_hdlc.tbuf`は「送信エラー後に現在再送中のバッファ」を追跡する、同期されていない単一のポインタであり、HDLCラインディシプリン（`drivers/tty/n_hdlc.c`、`N_HDLC`でロード可能）においてこのバグより前から存在していた。2009年のコミットが`flush_tx_queue()`のパス（`TCFLSH` ioctlで起動される）を追加し、これが既存の`tbuf`ポインタと競合した。並行に走る`flush_tx_queue()`と`n_hdlc_send_frames()`（送信完了）が、どちらも`tbuf`経由で同じバッファを見て、両方がそれを空きバッファリストにキューイングしうる。これは二重解放であり、ローカルの非特権ユーザーから到達可能だった。研究者のAlexander Popovはこれを、SMEPを回避する実際に動作するroot権限取得エクスプロイトに仕立てた。

修正（[`82f2341c94d2`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=82f2341c94d270421f383641b7cd670e474db56b)、Alexander Popov、2017年2月）は、場当たり的なポインタ追跡の設計を丸ごと取り除き、バッファの管理をカーネル標準の`list_head`に変換した。そして、送信エラーとなったバッファを、同期されていない2つ目の参照で追跡する代わりに、単純にバッファリストの先頭へ再キューイングする。[NVD: CVE-2017-2636](https://nvd.nist.gov/vuln/detail/CVE-2017-2636)、CVSS 3.1 7.0 HIGH。

### 事例2: ptyペアで誤ったtty_structをロックしていた — CVE-2020-29660とCVE-2020-29661 {#case-2-the-wrong-tty_struct-locked-in-a-pty-pair-cve-2020-29660-and-cve-2020-29661}

Jann Hornが同じ日に報告した関連する2つのバグで、どちらも原因は`drivers/tty/tty_jobctrl.c`にある。`tiocspgrp()`は2つの`tty_struct`ポインタを受け取る。`tty`（ユーザー空間の`ioctl()`のfdが指すもの）と`real_tty`（実際に変更されるtty。ioctlがpty *マスター* のfdに対して実行された場合に異なる）である。ところが`real_tty->pgrp`を更新するコードは、`tty->ctrl_lock`、つまり *反対側の* ロックを取っていた。同じptyペアの両端で並行に`TIOCSPGRP`を呼ぶと競合が起き、下にある`struct pid`の参照カウントを破壊しうる。これはuse-after-freeである（CVE-2020-29661）。これとは別に、`tty->session`のロックがコードベース全体で一貫していなかった。大半の呼び出し箇所は保護していたが、`disassociate_ctty()`、`__do_SAK()`（`drivers/tty/tty_io.c`内）、`tiocspgrp()`、`tiocgsid()`は保護しておらず、`TIOCGSID`の読み取りが並行するセッションの解体と競合しえた（CVE-2020-29660）。

どちらもJann Hornが同じ日に修正した。[`54ffccbf053b`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=54ffccbf053b5b6ca4f6e45094b942fab92a25fc)は`tiocspgrp()`で`tty->ctrl_lock`の代わりに`real_tty->ctrl_lock`をロックする。[`c8bcd9c5be24`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c8bcd9c5be24fb9e6132e97da5a35e55a83e36b9)は`drivers/tty/tty_io.c`、`drivers/tty/tty_jobctrl.c`、`include/linux/tty.h`にまたがり、`->session`へのすべての書き込み側に一貫したロックを取らせ、`tiocgsid()`、`tiocspgrp()`のセッションチェック、`__do_SAK()`に欠けていたロックを追加する。[NVD: CVE-2020-29661](https://nvd.nist.gov/vuln/detail/CVE-2020-29661)（CVSS 3.1 7.8 HIGH、`PR:L` — 通常のローカルユーザー）、[NVD: CVE-2020-29660](https://nvd.nist.gov/vuln/detail/CVE-2020-29660)（CVSS 3.1 4.4 MEDIUM — `PR:H`と評価されており、起動には昇格したローカル権限が必要で、CVE-2020-29661の通常ローカルユーザーである`PR:L`より高いハードルである）。

### 事例3: ldiscセマフォのハングアップ時デッドロック — CVE-2015-4170 {#case-3-the-ldisc-semaphore-hangup-deadlock-cve-2015-4170}

2013年、ラインディシプリンの場当たり的な「mutexとステータスビット」によるロックを専用のリーダー/ライターセマフォ（`ld_semaphore`、`drivers/tty/tty_ldsem.c`）に置き換える作り直しが行われた。その際、内部ヘルパー`ldsem_cmpxchg()`は、compare-exchangeの後に *新しい* ロックカウント値ではなく *古い* 値をチェックしていた。シグナルを受けたリーダー/ライターがldiscの参照を解放するのと同時に、ハングアップが待機中であり、さらに新しいリーダー/ライターも獲得しようとしていると、待機中のハングアップへの起床が失われることがあった。その結果、ハングアップ処理は無期限に停止した。

修正（[`cf872776fc84`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=cf872776fc84128bb779ce2b83a37c884c3203ae)、Peter Hurley、2013年12月）は`ldsem_cmpxchg()`を作り直し、成功/失敗を正しく報告し、失敗時には呼び出し元のカウントを現在値で更新するようにして、失われていた起床のパスを回復した。[NVD: CVE-2015-4170](https://nvd.nist.gov/vuln/detail/CVE-2015-4170)、CVSS 3.0 4.7 MEDIUM。

### 事例4: 珍しいクローズ順序で起きるptyのdriver_dataのuse-after-free — CVEなし {#case-4-the-pty-driver_data-use-after-free-on-an-unusual-close-order-not-a-cve}

`drivers/tty/pty.c`のコーナーケースである。ptyのマスター/スレーブのペアに対する *最後の* 参照が、下にあるdevptsのinodeを所有する`/dev/ptmx`/`/dev/pts/N`のファイルを直接通じてではなく、以前に開かれていた`/dev/tty`のファイルディスクリプタ経由で解放されることがあった。その最後の`/dev/tty`のクローズが走る前に、`ptmx`/`pts/N`のハンドルがすべて閉じられ、それらが保持していたdevptsのinodeもすでに解放されていると、`tty->driver_data`（そのinodeを指していた）は古い値になっており、`pty_unix98_shutdown()`はすでに解放された後のそれを使用していた。

修正（[`2831c89f42dc`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2831c89f42dcde440cfdccb9fee9f42d54bbc1ef)、Herton R. Krzesinski、2016年1月）は、オープン時に`ihold()`でptmxのinodeに追加の参照を取り、`pty_unix98_shutdown()`がペアのどちら側からでも正しいinodeを特定して、その保持した参照を明示的に解放するようにした。有効性がクローズ順序に依存する`driver_data`ポインタを信用するのをやめたのである。

## 共通点 {#common-threads}

| パターン | n_ttyエコー競合 | pty_writeの顛末 | n_hdlc二重解放 | 誤ったttyのロック | ldsemハングアップ | pty driver_data UAF |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| 書き込み側のパスが、独立に起動される読み取り/エコー側のパスと競合する | あり | あり | — | — | — | — |
| 後の修正が、以前の修正が依存していた緩和策を取り除いた | — | あり | — | — | — | — |
| 対になった構造体の誤った側でロックを取っている | — | — | — | あり（29661のみ） | — | — |
| 特別な設定なしに任意のローカル非特権ユーザーから到達可能 | あり | あり | あり* | あり | あり | あり |
| 修正から現実の影響まで、または混入から修正までの期間が長い | あり（2014年修正 → 2023年CISA掲載） | あり（2018年修正 → 2022年に本当の修正） | あり（2009年混入 → 2017年修正） | — | — | — |

\* `N_HDLC`ラインディシプリンがロードされていれば（`CONFIG_N_HDLC`）、任意の非特権ユーザーから到達可能。これはモジュール/設定上の前提条件であって、追加の権限要件ではない。

**6件のうち2件はまったく同じ形をしている。書き込み側のパスと、別に起動される読み取りまたはエコー側のパスの両方が同じttyバッファを変更し、それを保護するはずのロックが2つのパスの片方しかカバーしていない**。n_ttyのエコー競合とpty_writeの顛末の第3幕はどちらもこのパターンであり、ttyスタックの異なる2箇所（N_TTYの書き込みパスと、ptyレイヤーの書き込みパス）で起きている。サブシステム全体に共通する根本原因は、「書き込みパス」と「バッファを変更しうるあらゆるもの」が同じ呼び出し箇所の集合ではなく、前者を対象としたロックでは後者を取りこぼす、ということのようである。

**pty_writeの顛末は、このページの中で、ある修正が1つの正しさの性質を別の性質と引き換えにし、しかもその引き換えが安全かどうか誰も確認しなかったことを最も鮮明に示す例である**。2020年の修正は、lockdepが検出したデッドロック（実在し、検証済みの問題）を、バッファのコミット手順をロックの外へ移すことで正しく解決した。しかし、コミット手順が、2018年の修正が暗黙の不変条件として確立していた無関係な理由（データ競合に対する安全性）のために、依然として挿入手順とアトミックである必要があるかどうかを再検証しなかった。これを[ALSAの事件簿のミキサー解体時UAF](../alsa/war-stories/mixer-teardown-use-after-free.md)や[Input/HIDの型混同バグ](../input/war-stories/hid-validate-values-type-confusion.md)と比べてみるとよい。これらは、ある問題への修正が副作用として別の問題を *持ち込んだ* 事例である。pty_writeの顛末はその変種であり、ある修正が、 *元の* 修正が解決していた問題への緩和策を *取り除いて* しまった。デッドロックに焦点を当てた修正は、その性質をまったく確認していなかったからである。

**事例2の「対になった構造体の誤った側」というバグは、それ自体として名前を付けておく価値のある、別個の根本原因である**。ptyには常に2つの`tty_struct`（マスターとスレーブ）があり、その一方を渡されたコードは、どの操作が *その* 構造体に適用され、どの操作がペアの相手（`real_tty`）を経由する必要があるかを知っていなければならない。誤った側をロックするのはロックの欠落のバグではない。ロックは取られていたが、対象のオブジェクトが間違っていたのであり、これはロックが完全に欠落している場合よりもレビューで見つけにくいことがある。

## 関連項目 {#see-also}

- [TTY層](README.md) — `tty_struct`、`tty_driver`、そしてこれらの事件が起きている3層スタック
- [シリアルコアとUARTドライバ](serial.md) / [ラインディシプリン、termios、擬似端末](line-disciplines.md) — 事例1〜4と両方の詳細解説が関わるバッファとptyの仕組み
- [ALSAの事件簿](../alsa/war-stories.md)と[Input/HIDの事件簿](../input/war-stories.md) — 兄弟関係にあるデバイスサブシステムの事件ページ。「あるバグの修正が別のバグを持ち込む」ことが、このサイトのデバイスドライバの記事全体でどれほど繰り返されるかを比較するために
- [ロック](../locking/README.md) — ロックの範囲、ロック順序、クリティカルセクションのバグに関する一般的な背景。このページの大半の背後にあるパターンである
