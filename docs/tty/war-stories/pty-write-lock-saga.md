# 修正され、修正が取り消され、再び修正されたpty_write() のロック {#the-pty_write-lock-that-was-fixed-un-fixed-and-fixed-again}

> データ競合に対する2018年の修正はロックを追加した。lockdepが検出したデッドロックに対する2020年の修正はその一部を取り除いた — そして元の競合を新しい形で黙って再導入した。2022年のCVE報告がついに再導入されたバグを捕捉し、本当の修正は、一方を他方と引き換えにするのではなく、両方の性質を同時に満たした。

マージ（最終修正）
:   Linux、2022年7月（CVE-2022-1462）

ドライバ
:   PTYコア（`drivers/tty/pty.c`、`drivers/tty/tty_buffer.c`）

報告者（2022年）
:   oss-security経由の "一只狗"

メカニズム
:   同期されていないttyフリップバッファのコミット。4年間にわたる3度の改訂

CVE-2022-1462

*[事件簿：TTY/シリアルのバグとリグレッション](../war-stories.md)の一部。*

## 第1幕（2018年）：元の競合 {#act-1-2018-the-original-race}

`pty_write()` は、受信側のttyフリップバッファに `tty_insert_flip_string_fixed_flag()` で到着したバイトを挿入し、その後 `tty_flip_buffer_push()` を呼んでそれをコミットしていたが、どちらの呼び出しもロックで囲まれていなかった。2つのスレッドが、まったく異なる呼び出しチェーンを通って、同じptyの `pty_write()` に同時に到達しうる — 修正の動機となったKASANの報告は、一方のスレッドが `n_hdlc_send_frames()` → `pty_write()` を経て（`n_tty_ioctl_helper()`/`__start_tty()` 呼び出しからのウェイクアップによって起動されて）到着し、もう一方が `tty_send_xchar()` → `pty_write()` を経て（もう一方のCPU上の別の `n_tty_ioctl_helper()` 呼び出しによって起動されて）到着する様子を示している — そして両者がフリップバッファの末尾とコミット状態を同時に変更し、それを破壊しうる。slabの範囲外書き込みである。

コミット `b6da31b2c07c46f2dcad1d86caa835227a16d9ff`（"tty: Fix data race in tty_insert_flip_string_fixed_flag"、DaeRyong Jeong、2018年4月）は、これを素直な方法で修正する。挿入とその後の `tty_flip_buffer_push()` の両方を `to->port->lock` で囲むのである。

## 第2幕（2020年）：修正がデッドロックを生む {#act-2-2020-the-fix-creates-a-deadlock}

`tty_flip_buffer_push()` の間 `port->lock` を保持することには、2018年の修正が予期していなかったコストがあることが判明した。`tty_flip_buffer_push()` は最終的に（`queue_work_on()`/`__queue_work()` を通じて）workqueueの `pool->lock` を直接取得し、その依存辺が、依存グラフの別の場所で `console_owner` と8250コンソール自身の `port->lock` を通るサイクルを閉じてしまう — `console_unlock()` は `console_owner` を取得したまま `serial8250_console_write()` を呼び、それがUARTの `port->lock` を取得する。一方で別途、workqueueの `pool->lock` を保持しているタスク（`printk()` でworkqueueの状態をダンプしている）が `console_unlock()` を呼んで `console_owner` でブロックしうる。lockdepは、これら3つのロックにまたがる結果としての循環的なロック順序の依存関係を指摘した — 適切なインターリーブのもとでは実際のデッドロックの危険であり、`tty_flip_buffer_push()` 自身が直接取得するロックの問題ではなかった。

コミット `71a174b39f10b4b93223d374722aa894b5d8a82e`（"pty: do tty_flip_buffer_push without port->lock in pty_write"、Artem Savkov、2020年9月）は、`tty_flip_buffer_push()` を `port->lock` の**外**に戻すことでlockdepのサイクルを解消する — ただし挿入自体はロックされたままである。これは第1幕を丸ごとリバートするより狭く、より外科的な修正に見える。だがそうではない。`tty_flip_buffer_push()` は、ライン規約に渡す準備ができたデータ量を示すために、バッファの `commit` フィールドに（対になるacquireロードから書き込みが見えるように `smp_store_release()` で）*書き込む*。その書き込みがロックの外に移ると、並行する挿入によって、`commit` が、後で独立に読み出す側が観測するものと比べて不整合な状態に置かれうる — 2018年の競合が、生の挿入ではなくコミットを追跡するフィールドで生まれ変わったのである。その後の読み出しは、下の第3幕で説明するworkqueueのワーカー `flush_to_ldisc()` で行われ、それは自身の `smp_load_acquire(&head->commit)` を、今やロックされていないこの書き込みと対にしている。

## 第3幕（2022年）：生まれ変わった競合にCVEが付く {#act-3-2022-the-reborn-race-gets-a-cve}

ある研究者がoss-securityで、`flush_to_ldisc()` が**負の**バイト数を計算しうることを報告した — 競合する挿入と読み出しが不整合な状態を観測すると `smp_load_acquire(&head->commit) - head->read` が負になり、範囲外読み出しとカーネルのslabメモリの漏洩につながる — Jiri Slabyの修正コミットは、同じ不整合な末尾の競合を、より広く "can lead to out of bounds writes and other issues"（訳: 範囲外書き込みやその他の問題にもつながりうる）ものとして説明している。NVD自身のCVEの説明によれば、この競合は、ペアの反対側で `pty_write()` と `ioctl(TIOCSTI)` および `ioctl(TCXONC)` を競合させることで再現された — どちらも `pty_write()` の呼び出しパスの外で、同じttyバッファに文字を挿入する。これがCVE-2022-1462であり、CVSS 3.1 6.3（MEDIUM）と評価されている。

コミット `a501ab75e7624d133a5a3c7ec010687c8b961d23`（"tty: use new tty_insert_flip_string_and_push_buffer() in pty_write()"、Jiri Slaby、2022年7月）は、自身のコミットメッセージに `Fixes: 71a174b39f10` を含んでいる — これを壊したのは2020年の修正だという明示的な認定である。本当の解決は新しいヘルパー `tty_insert_flip_string_and_push_buffer()` を導入する。これは挿入**と**バッファのコミットを一緒に `port->lock` の下で行い — 2018年/2022年の競合を閉じる — その後の `queue_work()` 呼び出しをロックの*外*に遅らせる。これこそが実際に2020年のデッドロックを避けるものである。2018年と2020年の修正がそれぞれ半分ずつしか得られなかった2つの性質 — データ競合がないこと、ロック順序のサイクルがないこと — は、どちらの操作をロックの下で行うかを二者択一で選ぶのでなければ、両方とも達成できることが判明したのである。

## なぜ起きたか {#why-it-happened}

最初の2つの修正は、それぞれが対象とした特定の問題に対してはローカルに正しかった。2018年の修正は本当にデータ競合を閉じた。2020年の修正は本当にlockdepが検出したデッドロックを閉じた。どちらの修正の作者も、自分が見ているバグについて間違っていたわけではない — 2020年の修正の作者は、実在し検証済みのロック順序の問題を解決しようとしており、それを捕捉したツール（lockdep）はデータ競合の安全性については知らないし気にもかけない。ロックの獲得順序だけを見る。バッファのコミットをロックの外に移すことは、lockdepが報告した特定のサイクルを断ち切るための、正しく最小限に見える方法だった。それがしなかったのは、挿入はロックされたままで読み出しがそうでなくなったときに、*ロックされていない*コミットの読み出しがなお安全かどうかを再検討することだった — それは2018年の修正が両操作にまたがる不変条件として確立していた性質であり、2020年の修正は誰もそれを再検証することなく、その性質を壊したのである。

## 解決 {#resolution}

上の第3幕を参照のこと。`tty_insert_flip_string_and_push_buffer()` は挿入とコミットを `port->lock` の下でアトミックに行い、（ロックに依存しない）workをキューに入れるステップの前でのみロックを解放する — 2018年の修正のデータ競合安全性と、2020年の修正のlockdepサイクルからの解放を、同じコミットで両立させている。

## 教訓 {#what-it-taught-us}

***検出された*問題（lockdepの警告、特定のクラッシュ）を解決する修正は、前の修正が依存していた*検出されていない*不変条件を、それ自体が書き留められたりテストされたりしていなければ、黙って取り消しうる。** 2018年の修正は「挿入とコミットはアトミックに一緒に行われる」を暗黙の不変条件として確立した。その不変条件を第一級の性質として強制したり文書化したりするものは何もなかった — そのため、lockdepを満足させることだけに集中していた2020年の修正には、挿入とコミットをロック境界をまたいで分割するとそれが破られる、というシグナルが何もなかった。

**互いにトレードオフの関係にあるように見える2つの正しさの性質（データ競合がないこと対デッドロックがないこと）は、両方とも達成できる場合がある — 見かけのトレードオフは、根本的な対立ではなく、修正を*どう*構成したかの産物でありうる。** 最終的な修正は、ロックを増やすか減らすかを選んだのではない。*何を*一緒にロックの下に置くか（挿入 + コミット、ただしその後のworkqueue呼び出しは含めない）を変えることで、両方の性質を同時に得たのである。

!!! warning "注意すべきパターン"
    デッドロック/ロック順序の問題に対する修正が、以前はアトミックだった複数ステップの操作の一部をロックの外に移すことで対応しており、そもそもそれらをロックの下に置いた動機とは*別の*理由（ここではデータ競合の安全性）で、それらのステップがなお互いにアトミックに行われる必要があるかどうかを再検証していない場合。

## 関連項目 {#see-also}

- [シリアルコアとUARTドライバ](../serial.md) / [ライン規約、termios、擬似端末](../line-disciplines.md) — この事件が起きているptyのmaster/slaveモデルとttyバッファの仕組み
- [n_ttyのエコー競合](n-tty-echo-race.md) — pty層ではなくN_TTYにおける別のttyバッファのロックのバグ。同じくらい長い尾を引いた（修正からCISA KEV掲載まで9年）

## 外部参考資料 {#external-references}

- [NVD: CVE-2022-1462](https://nvd.nist.gov/vuln/detail/CVE-2022-1462) — CVSS 3.1 6.3 MEDIUM
- [oss-security: 元の報告](https://seclists.org/oss-sec/2022/q2/155)
- [git.kernel.org: b6da31b2c07c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b6da31b2c07c46f2dcad1d86caa835227a16d9ff) — 第1幕（2018年）：挿入とコミットをロックする
- [git.kernel.org: 71a174b39f10](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=71a174b39f10b4b93223d374722aa894b5d8a82e) — 第2幕（2020年）：lockdepのサイクルを直すため、コミットをロックの外に移す
- [git.kernel.org: a501ab75e762](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a501ab75e7624d133a5a3c7ec010687c8b961d23) — 第3幕（2022年）：本当の修正。自身のコミットメッセージに `Fixes: 71a174b39f10` を含む
