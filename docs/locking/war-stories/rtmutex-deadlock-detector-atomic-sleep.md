# アトミックコンテキストでスケジュールしたデッドロック検出器 {#the-deadlock-detector-that-scheduled-while-atomic}

> CVE-2024-46829 — rt_mutex自身のサイクル検出器が本物のデッドロックを確認したとき、その対応は警告を出してから `schedule()` を呼び続ける無限ループに入ることだった。しかも、同じロックの他のすべての待機者が前進するために必要なraw spinlockを保持したままである

公開
:   2024年9月27日（NVD、Linuxカーネル自身のCVEツールから自動生成）

報告者
:   Roland Xu、LKMLへのパッチ投稿による

CVSS
:   5.5 MEDIUM（`CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:N/I:N/A:H`）

バグの存在期間
:   `rt_mutex_handle_deadlock()` が導入された3.16（2014年）以降

修正
:   コミット `d33d26036a02`、メインラインLinux 6.11（2024年9月15日）

エクスプロイトツール
:   なし — ファザーやクラッシュ報告ではなく、コードレビュー的な修正

実際の悪用
:   確認された事例なし（CISA KEVには未掲載）

*[事件簿: ロックと同期のバグ](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

rt_mutexの低速パスでのロック取得は、このロックを取ると優先度継承の依存グラフにサイクルができるかどうかを調べるチェーンウォークの間、`rt_mutex::wait_lock` を保持する。これはraw spinlockであり、`PREEMPT_RT` の下でもプリエンプトされず、スリープもしない。`3d5c9340d194`（"rtmutex: Handle deadlock detection smarter"、2014年）は、そのチェーンウォークが、呼び出し元が明示的に検出を求めたものだけでなく、見つけた本物のデッドロックをすべて報告するようにし、確認されたサイクルに対応する `rt_mutex_handle_deadlock()` を導入した。警告を出力し、問題のタスクを `TASK_INTERRUPTIBLE` にして無限の `while (1) { schedule(); }` ループの中に留め置くのである。本物の循環待ちの一部であることが証明されたばかりのタスクを、まともに巻き戻す方法はないからである。

## きっかけ {#the-trigger}

`rt_mutex_handle_deadlock()` は低速パスのロック関数の内部から呼ばれており、制御フローのその時点ではまだ `wait_lock` を保持していた。2014年のコミットには、留め置きループに入る前にそれを解放する処理がなかった。関数は単に `WARN(1, "rtmutex deadlock detected\n")` を発し、raw spinlockを保持したまま `schedule()` を呼んでいた。デッドロック検出は、すべてのロック取得で実行されるわけではない。非デバッグカーネルでは明示的な `RT_MUTEX_FULL_CHAINWALK` フラグの背後に置かれているが、PI-futexのプロキシロックパス（glibcの `PTHREAD_PRIO_INHERIT` mutexを支える `__rt_mutex_start_proxy_lock()`）は常にこのフラグを渡す。同じ2014年のコミットのコメントにあるとおり、"We enforce deadlock detection for futexes"（訳: futexについてはデッドロック検出を強制する）である。`CONFIG_DEBUG_RT_MUTEXES=y` でビルドされたカーネルでは、PI-futexかどうかにかかわらず、すべてのrt_mutexの取得で実行される。

## 観測された挙動 {#observed-behavior}

raw spinlockを保持したまま `schedule()` を呼ぶことは文書化された不変条件の違反であり、スケジューラ自身がデバッグビルドだけでなく本番カーネルでも、呼び出しのたびにそれをチェックしている。`__schedule()` の `schedule_debug()` は `in_atomic_preempt_off()` をテストし、真であれば `"BUG: scheduling while atomic: %s/%d/0x%08x\n"` を出力し、保持中のロックをダンプし、システムに `panic_on_warn` が設定されていれば即座にパニックする。診断メッセージ以外にも、実際の障害は複合的に悪化する。確認されたデッドロックに遭遇したタスクは今やCPUから外れて割り込み可能なスリープで留め置かれているが、`wait_lock` は保持されたままである。同じ `wait_lock` を必要とする他のCPU（同じrt_mutexの別の待機者や、PIの管理処理を行うロック自身の所有者）は、raw spinlockのビジーウェイトで無期限にスピンする。スリープしている保持者がそれを解放するために再びスケジュールされる保証は何もないからである。あるタスクで確認され、正しく検出されたデッドロックが、まさにデッドロックを安全に報告することを役割とするコードパスを通じて、実際のシステム全体のロックアップに変わる。

## なぜ起きたか {#why-it-happened}

2014年のコミット自身の説明は、その目的を正確に示している。検出されたサイクルに対して黙って `0` を返すのをやめ、代わりに警告を出してタスクを留め置くことである。そのコミットの差分で新しい留め置きループの上に残されたコメントは、もっと率直に（タイプミスも含めて）こう述べていた。"Yell lowdly and stop the task right here."（訳: 大声で叫んで、タスクをここで止めよ。）「前進する方法がない」ことへの対応としては妥当である。しかし「タスクをここで止める」は無限のスケジューリングループとして実装され、当時は誰も、「ここ」がそのループの間どのロックがまだ保持されていることを意味するのかを追跡しなかった。バグはタスクを留め置くという判断にあるのではない。留め置きのコードがクリティカルセクションの内部に書かれ、そこに到達した時点で関数がまだどの状態を保持しているかを再検討しなかったことにある。

## 解決 {#resolution}

`d33d26036a02`（"rtmutex: Drop rt_mutex::wait_lock before scheduling"）は、`WARN()` と留め置きループの直前に `raw_spin_unlock_irq(&lock->wait_lock)` を追加する。スリープしうる処理の後ではなく、前にロックを解放するのである。Thomas GleixnerがLinusへのプルリクエストで書いた修正の要約は、バグを率直に述べている。"The deadlock detection code drops into an infinite scheduling loop while still holding rt_mutex::wait_lock, which rightfully triggers a 'scheduling in atomic' warning. Unlock it before that."（訳: デッドロック検出コードは rt_mutex::wait_lock を保持したまま無限のスケジューリングループに入っており、当然ながら「scheduling in atomic」警告を引き起こす。その前にアンロックする。）投稿されたパッチでは当初、アンロックは `WARN()` の*後*に置かれていた。Gleixnerはマージの際にそれを前に移し、警告自体もアトミックコンテキストから発火しないようにした。

## 教訓 {#what-it-taught-us}

**「タスクを安全に止める」と「何がまだロックされているかを確認せずにタスクを止める」は別の修正であり、前者を書いたつもりで後者を出荷してしまうのはたやすい。** 2014年のコミットの意図（検出されたデッドロックで黙って成功を返すのではなく、大声で知らせて動かなくする）は正しかった。バグは完全に、タスクを*どのように*動かなくしたかという仕組みの側にあり、そうするという判断の側にはなかった。

**致命的な状態を報告するためだけに存在するコードパスも、他のすべてのパスと同じロックの規律を守らなければならない。** すでに回復不能なほど壊れている分岐を、「どうせタスクは止まっている」のだから通常の注意から免除されると扱いたくなる。しかし、その分岐に持ち込まれたロックは、まだそれを待っている他のすべてのタスクにとって重要であることに変わりはない。

!!! warning "注意すべきパターン"
    「致命的な状態を検出したので、このタスクを留め置く」というコードパスは、通常の制御フローとまったく同じロック監査に値する。通常のテストでは実行されないほどまれにしか到達しないことが多いので、むしろそれ以上に値すると言える。エラー処理やパニックに近い分岐の中で、ブロックやスリープしうる呼び出し（`schedule()`、`msleep()`、待機しうるあらゆる割り当て）を追加する前に、成功パスの場合と同じように、すぐ外側のスコープがまだ何を保持しているかを確認すること。

## 関連項目 {#see-also}

- [ロックの概要](../README.md) — spinlock、rt_mutex、そしてこのバグが存在するデッドロック検出機構
- [PI-mutex誕生の経緯](pi-mutex-origin.md) — rt_mutexの優先度継承のチェーンウォーク。このバグの留め置きループが応答していたサイクル検出を行う仕組み
- [永続的なフォールトに答えを持たなかったPI-futexの修正処理](pi-futex-fixup-owner-uaf.md) — フォールト処理の側における、3つ目のrt_mutex/PI-futexの正しさのバグ

## 外部参考資料 {#external-references}

- [git.kernel.org: d33d26036a02](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d33d26036a0274b472299d7dcdaa5fb34329f91b) — "rtmutex: Drop rt_mutex::wait_lock before scheduling"、修正
- [git.kernel.org: 3d5c9340d194](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3d5c9340d1949733eb37616abd15db36aef9a57c) — "rtmutex: Handle deadlock detection smarter"、無条件の留め置きループを導入した2014年のコミット
- [lore.kernel.org: 最初のパッチ投稿](https://lore.kernel.org/all/ME0P300MB063599BEF0743B8FA339C2CECC802@ME0P300MB0635.AUSP300.PROD.OUTLOOK.COM/) — Roland Xuによる "Avoid schedule while atomic if meeting the early deadlock"
- [NVD: CVE-2024-46829](https://nvd.nist.gov/vuln/detail/CVE-2024-46829) — CVEレコード、CVSS 5.5 MEDIUM、2024年9月27日公開
