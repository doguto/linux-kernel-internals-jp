# `NO_HANG` の誤検知 {#the-no_hang-false-positive}

> 長年、`drm_sched` のタイムアウトハンドラは「GPUはハングしたか」に対してただ1つの答えしか持っていなかった。リセットすることである。3つのドライバは「実はハングしていない」と言う必要があったが、スケジューラにはメモリをリークさせずにそれを伝える手段がなかった

収録
:   Linux 6.17（2025年7月）

影響を受けたドライバ
:   v3d、Etnaviv、Xe

作者
:   Maíra Canal（Igalia）

メカニズム
:   `DRM_GPU_SCHED_STAT_NO_HANG`

CVEではない

*[事件簿: GPU/DRMのバグとリグレッション](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

[コマンド投入、`dma_fence`、GPUスケジューラ](../command-submission.md)では `drm_sched` のタイムアウトパスを扱っている。ジョブのタイムアウトが経過すると、`drm_sched_job_timedout()` はドライバの `timedout_job()` コールバックを呼ぶ。そしてこの変更以前は、ドライバのコールバックがハードウェアを回復させるか、いずれにせよジョブは本当に死んでいるという前提で、スケジューラは常に `sched->pending_list` からジョブを取り除いていた。`sched->ops->free_job()` は、そのリストにまだ載っているジョブに対してしか実行されなかった。

## きっかけ {#the-trigger}

Maíra Canalによる [`0b1217bfdfdd`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0b1217bfdfddf664c15954d1d51ee18ed88a2ccf)（"drm/sched: Allow drivers to skip the reset and keep on running"、2025年7月）のコミットメッセージは、スケジューラのタイムアウトの発火がGPUが実際にハングしたことを意味しない2つの具体的なシナリオを挙げている。

1. **ジョブはまだ進行しているが、タイムアウトが許すより遅いだけ。** 一部のドライバ（v3d、Etnaviv、Xeが明示的に挙げられている）は、ソフトウェアのタイムアウトが経過した後でもハードウェアがまだ進行しているかを確認するGPU固有の方法を持っており、正しく動作しているハードウェアに強制的にリセットをかけるよりは、ジョブを保留中として扱い続けるようスケジューラに伝えたい。
2. **タイムアウトとfree-jobワーカーが競合した。** 終了したばかりのジョブを別のワーカーがすでに解放しているちょうどその時にスケジューラのタイムアウトが発火すると、実際にはまったくタイムアウトしていないジョブに対して `sched->ops->timedout_job()` が呼ばれる。そのジョブは自身の完了との競合に負けただけである。

## 観測された挙動 {#observed-behavior}

どちらのシナリオも同じ構造上の問題に行き当たる。ドライバの `timedout_job()` コールバックが実行されるより前に、ジョブはすでに `sched->pending_list` から取り除かれていた。ドライバがジョブは実際にはハングしていないと正しく判断し、その後ジョブが本当に完了しても、スケジューラにはそれを解放すべきことを知る手段がなかった。`free_job()` は保留中リストでまだ追跡されているジョブに対してしか発火しないからである。ドライバがこうした正当な「実はハングしていない」ケースのどちらかに遭遇するたびに、ジョブのメモリが黙ってリークした。

## なぜ起きたか {#why-it-happened}

スケジューラのタイムアウト処理が扱える情報は1ビット（タイムアウトが経過した）だけで、それに応じて取れる行動も1つ（ジョブをもはやスケジューラが追跡しないものとして扱う）だけだった。「タイムアウトが経過した」が確実に「ジョブがハングした」を意味する限りはそれで十分であり、ハードウェアの進行を独立に検証する手段を持たないドライバではそれが成り立っていた。しかし、複数のドライバがソフトウェアのタイムアウトより信頼できるGPU固有の生存信号を持つようになり、さらにfree-jobの競合によって、ハングしていないジョブさえ実際のハングではなく純粋なタイミングで同じコードパスを引き起こせるようになると、それは成り立たなくなった。

## 解決 {#resolution}

`0b1217bfdfdd` は新しい `drm_gpu_sched_stat` の値 `DRM_GPU_SCHED_STAT_NO_HANG` を追加する。ドライバの `timedout_job()` コールバックはこれを返すことで、スケジューラに対し、リセット済みで消えたものとして扱うのではなく、このジョブを `sched->pending_list` に再挿入して通常どおり完了まで実行を続けさせるよう伝えられる。同じシリーズの3つの後続コミットが、特定のドライバでそれを使うよう配線した。v3d向けの [`6b37fbacd087`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6b37fbacd087fbd517b6b276ca8bebd1dc052fb7)（Tvrtko Ursulinがレビュー）、Etnaviv向けの [`8902c2b17a6e`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8902c2b17a6ec723ab7924bc4113bef47603c0dc)（Lucas Stachがレビュー）、Xe向けの [`53dcd0eaa271`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=53dcd0eaa271e870ca5d0b203be67b468214c1bc)（Matthew Brostがレビュー）である。それぞれが、ドライバ固有の「このジョブは実際にはまだ進行しているのか、それともタイムアウトがfree-jobワーカーと競合しただけなのか」というチェックを、リセットを強制する代わりに `NO_HANG` を返すものに変換している。シリーズ全体は `20250714-sched-skip-reset-v6-...@igalia.com` として投稿された。

Xeの `NO_HANG` の戻り値は、ほぼ1年後、[Xe TDR回復のリグレッション連鎖](xe-tdr-recovery-regression.md)の解決で再び登場するのと同じものである。そのページの2026年6月の修正は、Xeのハンドラが破壊的なwedgeを呼ぶ*タイミング*を、まさにこの `NO_HANG` の戻り値との相対で動かすものであり、Xeがそもそも `NO_HANG` を使い始めた時期を変えるものではない。

## 教訓 {#what-it-taught-us}

**1つの二値信号（「ハングしたか否か」）を中心に作られたスケジューラAPIは、いずれその信号だけが提供する以上の情報を持つドライバに出会う。** v3d、Etnaviv、Xeはそれぞれ、「ハングしたからタイムアウトした」と「遅いだけでタイムアウトしたが、まだ動いていることを証明できる」を区別する、実際のドライバ固有の方法を持っていた。この変更が語彙を与えるまで、スケジューラには2つ目のケースを表す語彙がなかった。

**同じコードパスに対する2つの独立したトリガー（本当のタイムアウトと完了との競合）の間の競合は、どちらか一方のトリガーと同じくらい本物のバグになりうる。** 「ドライバのほうがよく知っている」ケースだけを修正し、「タイムアウトが完了と競合した」ケースもカバーしなかったら、ドライバの知識とは無関係な、単なるタイミングによるリソースリークが残っていただろう。

!!! warning "注意すべきパターン"
    「タイムアウトが発火した」を無条件に「リソースを破棄／リセットすべき」として扱うタイムアウト検出パスは、タイムアウトとリソース自身の通常の完了パスとの間に競合がないか、また計時対象がまだ進行していることを証明する手段を持っているのに、タイムアウト機構がそれを問い合わせる方法を知らないということがないかを、確認する価値がある。

## 関連項目 {#see-also}

- [コマンド投入、`dma_fence`、GPUスケジューラ](../command-submission.md) — TDR／タイムアウトの仕組みと、このページの修正が拡張する `drm_gpu_sched_stat` の戻り値
- [Xe TDR回復のリグレッション連鎖](xe-tdr-recovery-regression.md) — 同じく `DRM_GPU_SCHED_STAT_NO_HANG` を返す、まったく別のバグに関するXe固有のTDRリグレッション

## 外部参考資料 {#external-references}

- [git.kernel.org: 0b1217bfdfdd](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0b1217bfdfddf664c15954d1d51ee18ed88a2ccf) — "drm/sched: Allow drivers to skip the reset and keep on running"、`DRM_GPU_SCHED_STAT_NO_HANG` を追加した中核の変更
- [git.kernel.org: 6b37fbacd087](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6b37fbacd087fbd517b6b276ca8bebd1dc052fb7) — "drm/v3d: Use DRM_GPU_SCHED_STAT_NO_HANG to skip the reset"
- [git.kernel.org: 8902c2b17a6e](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8902c2b17a6ec723ab7924bc4113bef47603c0dc) — "drm/etnaviv: Use DRM_GPU_SCHED_STAT_NO_HANG to skip the reset"
- [git.kernel.org: 53dcd0eaa271](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=53dcd0eaa271e870ca5d0b203be67b468214c1bc) — "drm/xe: Use DRM_GPU_SCHED_STAT_NO_HANG to skip the reset"、同じシリーズにおけるXeの変換
