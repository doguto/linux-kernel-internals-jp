# dentryキャッシュのウォークのライブロック {#the-dentry-cache-walk-livelock}

> CVE-2014-8559 — `d_walk()` のリネーム検出による再スタートは、「リネームが起きた」と「兄弟がkillされた」を区別できなかった。そのため、並行するdentryの刈り込みによって、ウォーカーが自分自身とデッドロックし、マシン上のすべてのリネームをハングさせることがありえた

公開日
:   2014年11月10日（NVD）。修正の作成は2014年10月26日

報告者
:   Prasad Pandit、Red Hat（RHBZ #1159313、2014年10月31日）— ただし同一のバグは、その8か月前にJaegeuk Kimが独立に診断していた

CVSS
:   5.5 MEDIUM（`CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:N/I:N/A:H`）

バグの混入時期
:   v3.12頃（2013年11月）、`d_walk()` がdentryツリーをたどるコードを統合したとき

修正
:   commit `ca5358ef75fc`、2014年10月26日作成。約7週間後に3.18.yのstableブランチ（3.18.1、2014年12月）に入り、古いLTSブランチ（3.10.y、3.14.y）には2015年半ばまで入らず、メインラインそのものにはLinux 3.19（2015年2月）で入った

エクスプロイトツール
:   なし — 公開のエクスプロイトツールは存在しない。これは細工した入力による攻撃ではなく、通常のファイルシステム負荷で到達可能な並行性のバグである

実際の悪用
:   確認された事例はない（CISA KEVには掲載されていない）— ローカルなサービス拒否のみ

*[事件簿：VFSのバグとリグレッション](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

`d_walk()`（`fs/dcache.c`）は、ディレクトリ配下のすべてのdentryを訪れる必要がある操作 — キャッシュの縮小、アンマウント、`d_invalidate()` など — に使われる、汎用の深さ優先のツリーウォーカーである。これは一度に最大1つのdentryの `d_lock` しか保持せず、親のロックを取る前に子のロックを手放す。そのため、ウォーカーにとっての本当の問題はグローバルな整合性になる。ウォークの途中で別のスレッドがツリーの一部をリネームするのを止めるものは何もない。それはseqlockである `rename_lock` の役目である。ウォーカーは開始前に `rename_lock` のシーケンスカウンタを取得する。終了時までにカウンタが変化していれば、`need_seqretry()` はウォークが安全でないと報告し、ウォークは先頭から再スタートする。今度は実際に `rename_lock` の書き込み側を保持するので、完了が保証される。

サブツリーを終えて親に戻るときには、3つの条件による再検証が行われていた。

```c
if (this_parent != child->d_parent ||
     (child->d_flags & DCACHE_DENTRY_KILLED) ||
     need_seqretry(&rename_lock, seq)) {
        spin_unlock(&this_parent->d_lock);
        rcu_read_unlock();
        goto rename_retry;
}
```

このうち実際に `rename_lock` で保護されているのは、3つ目の節 `need_seqretry()` だけである。他の2つ — 親ポインタの変化、あるいは `DCACHE_DENTRY_KILLED` フラグ — は、リネームが一切起きていなくても、それぞれ独立に真になりえた。

## きっかけ {#the-trigger}

`DCACHE_DENTRY_KILLED` は `__dentry_kill()` によってセットされる。これはdentryが最終的に解放されるときに必ず呼ばれる — メモリ逼迫時のdcacheの縮小、`rmdir()`、最後の参照を手放す `dput()` などである。これらのどれも `rename_lock` には触れない。そのため、リネームがどこにも見当たらないのに、ウォーカーの足元からdentryが刈り込まれるだけで、本物のリネームとまったく同じ再スタートのパスが起動された。

Jaegeuk Kimは2014年2月、3.14-rc3カーネル上で `fsstress`（標準的なファイルシステムのストレスツール）を10スレッドで実行していてこれに独立に遭遇し、lockdepの報告をメーリングリストに投稿した。

```
[ INFO: possible recursive locking detected ]
(rename_lock){+.+...}, at: [<...>] d_walk+0x4d/0x3c0
but task is already holding lock:
(rename_lock){+.+...}, at: [<...>] d_walk+0x4d/0x3c0
*** DEADLOCK ***
```

彼はすでに問題の形を見抜いていた。"I suspect that the upper conditions can trigger rename_retry even though rename_retry was done once before."（訳: 上の条件は、rename_retryが一度すでに行われていても、rename_retryを起動しうるのではないかと疑っている）。Waiman Longは同じ日に、事実上、修正が最終的にとったアプローチで返信した。"It seems like the rename_lock may not be able to fully protect against the setting of the DCACHE_DENTRY_KILLED flag... I am 100% sure if we could just release the lock and let it try again without causing infinite loop."（訳: rename_lockはDCACHE_DENTRY_KILLEDフラグのセットに対して完全には保護できないようだ……無限ループを起こさずに、ロックを解放してもう一度試させることができるかどうか、100%確信している）。そのスレッドにはAl Viroからの公開の返信は見当たらず、実際の修正は8か月後に別途入った。それはRed HatのPrasad PanditがRHBZ #1159313を起票したことにクレジットされている — 2つの報告は、一方が他方を引き起こしたのではなく、独立に同じ根本原因に収束したように見える。

## 観測された挙動 {#observed-behavior}

再スタートは意図的にエスカレートする。最初の `rename_retry` は `seq = 1` をセットしてもう一周し、奇数の `seq` は、`need_seqretry()` の土台にある `read_seqbegin_or_lock()` に、単にサンプリングするのではなく `rename_lock` の実際の書き込みロックを取得させる — これは前進を保証する仕組みである。活発に変化するツリーに対するロックなしの再試行ループは、そうしなければ永遠に回り続けうるからである。しかしdentryのkillも `rename_lock` で制限されていなかったので、その今やロックされたパスの最中に*2度目*の並行するkillが起きると、まったく同じ分岐が再び起動されうる — ウォーカーは、最初のエスカレーションで取得した書き込みロックをすでに保持したまま、`rename_retry` に送り返される。Linuxのspinlockは再入可能ではない。タスクは、すでに保持しているロックを再取得しようとして永遠にスピンした。

`rename_lock` の書き込み側は、すべての `d_move()`/`vfs_rename()` 呼び出しが必要とするものでもあるので、このように1つのCPUが詰まると、起動したスレッドがハングするだけでなく、マシン上のその後のすべてのリネームが凍結した。Kimの元の報告は、この両方の側面をとらえていた。一方のCPUでのlockdepの警告と、別のCPUでの、`vfs_rename()` 内の `_raw_spin_lock` でスピンしている、まったく別個の "`BUG: soft lockup - CPU#0 stuck for 22s!`" である。Debianのアドバイザリは後に、実際の起動条件を独立に同じ言葉で説明している。"kernel functions that iterate over a directory tree can dead-lock or live-lock in case some of the directory entries were recently deleted or dropped from the cache."（訳: ディレクトリツリーを反復するカーネル関数は、ディレクトリエントリの一部が最近削除されたかキャッシュから落とされた場合に、デッドロックまたはライブロックしうる）

## なぜ起きたか {#why-it-happened}

このバグは、本当に異なる2つのイベント — 「ツリー構造が、自分の位置を無効にするような形で変化した」（リネーム。seqlockで正しく保護されている）と、「これから訪れようとしているエントリがもう存在しない」（kill。はるかに頻繁で、まったく無関係なイベント）— を混同していた。ウォーカーの内側からは、どちらも「自分の下にあるものはもう信頼できない」ように見えたからである。頻繁でありふれたイベントを、まれで厳密に同期されたイベントのために作られた同じ回復パスに折り込んだことで、日常的なdcacheの縮小が自ら招いたデッドロックに変わった。

## 解決 {#resolution}

`ca5358ef75fc`（"deal with deadlock in d_walk()"、Al Viro）は、killされた兄弟を、ウォーク全体を再スタートする理由として扱うのをやめた。親に戻るときに `rename_retry` が起動されるのは、本物の `need_seqretry()` の失敗の場合だけになった — これはretryラベルに新たに置かれた `BUG_ON(seq & 1)`、つまりすでにロックされたモードにある間は再スタートが決して起きてはならないという強いアサーションで補強されている。killされた子は、代わりにその場でスキップされる。新しい内側のループが兄弟リストを前方へたどり（`child = list_entry(next, struct dentry, d_child); next = next->next;`）、再開できる生きているdentryを見つけるか、末尾に達してさらに1レベル上に戻るまで進む。

このスキップループは、同じコミット内の2つ目の変更に依存している。`__dentry_kill()` において、`list_del()` — ノードをリンクから外す*と同時に*、削除後の使用バグを捕まえるために自身の `next`/`prev` ポインタを毒化する — が、リンクから外すことだけを行う `__list_del_entry()` に変わった。killされたばかりのdentryの `d_child.next` は依然として以前の次の兄弟を指しており、まさにそれによって、新しいループはkillされたdentryの連なりを安全に追い越して、生きているリストが再開する場所を見つけられる。同じ日の別のコミット `946e51f2bf37` は、これをRCUのもとで安全にした。`d_rcu` を、それまで `d_child` と共有していたunionから外し、代わりに `d_alias` と重ねるようにしたのである。これにより、dentryのRCUによる遅延解放が、スキップループがまだたどる必要のある `d_child` のリンクを上書きしなくなった。

## 教訓 {#what-it-taught-us}

**あるクラスの並行変更のために作られた回復パスが、まったく別の、はるかに頻繁なクラスの変更でも黙って発動し始めることがある**。両方の変更が、たまたま同じフラグや条件チェックを起動する場合である。`rename_lock` はdentryの寿命を調停するためのものではなく、dentryの位置だけを調停するためのものだった — しかし親に戻るロジックは、その両方をさせようとした。

**再試行をより強いロックへエスカレートさせて前進が保証されるのは、そのより強いロックを保持した時点で、*2度目*の再試行を起動しうるすべてのイベントがそれ自体排除されている場合だけである。** 修正は「より大きなロックを取る」ではなかった — そもそも何が再試行を起動してよいかを狭め、エスカレートしたパスが本当に再起動されえないようにすることだった。

!!! warning "注意すべきパターン"
    再試行してからロックにエスカレートするパターン（seqlock をサンプリングし、ロックなしで1回再試行し、書き込み側を保持してもう一度再試行する）がライブロックしないのは、*再試行を起動しうるイベントの集合*が、そのロックをすでに保持している間に起こりうるあらゆることと交わらないことが証明されている場合だけである。ロックが実際に保護しているものと無関係な何かによって再試行条件が起動されうるなら、エスカレートしたパスは、同じタスクがすでに保持しているロックに対して再起動しうる。

## 関連項目 {#see-also}

- [VFSの概要](../README.md) — dentryキャッシュと、パス解決における `rename_lock` の役割
- [RCU（Read-Copy-Update）](../../locking/rcu.md) — この修正のスキップループが依存している、RCUで保護されたたどり
- [Seqlock](../../locking/seqlock.md) — `rename_lock` が使う、ロックへのフォールバックを伴うseqlockのパターン

## 外部参考資料 {#external-references}

- [git.kernel.org: ca5358ef75fc](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ca5358ef75fc69fee5322a38a340f5739d997c10) — "deal with deadlock in d_walk()"、修正
- [git.kernel.org: 946e51f2bf37](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=946e51f2bf37f1656916eb75bd0742ba33983c28) — "move d_rcu from overlapping d_child to overlapping d_alias"、同じ日の対となる構造上の変更
- [NVD: CVE-2014-8559](https://nvd.nist.gov/vuln/detail/CVE-2014-8559) — CVEレコード。CVSS 5.5 MEDIUM、2014年11月10日公開
