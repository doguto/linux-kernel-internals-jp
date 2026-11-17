# `drm_sched_entity_kill_jobs_cb` のlockdepデッドロック {#the-drm_sched_entity_kill_jobs_cb-lockdep-deadlock}

> 割り込みコンテキストで実行されるフェンスシグナルのコールバックが、通常のプロセスコンテキストが割り込みを無効化せずに保持しているロックを取ろうとしていた。典型的な割り込み非安全なロックのバグであり、実際のユーザーのマシン上でlockdepが捕まえた

収録
:   2025年11月（stable 6.2以降にバックポート）

スケジューラ
:   `drm_sched`。共通DRMスケジューラを使うすべてのドライバで共有

作者
:   Pierre-Eric Pelloux-Prayer（AMD）

報告者
:   Mikhail Gavrilov、[Mesa issue #13908](https://gitlab.freedesktop.org/mesa/mesa/-/issues/13908) にて

修正
:   [`487df8b69834`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=487df8b698345dd5a91346335f05170ed5f29d4e)

CVEではない

*[事件簿: GPU/DRMのバグとリグレッション](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

`drm_sched` のエンティティが破棄されるとき（プロセスが終了する、コンテキストが破棄される）、そのエンティティにまだキューイングされているジョブは後始末される必要がある。それを担うのが `drm_sched_entity_kill_jobs_cb()` である。これはジョブの依存フェンスに取り付けられたコールバックとして実行され、エンティティに残っている `job->dependencies`（`xarray` に格納されている）を走査し、次の依存関係に対して後始末を再設定する。

`job->dependencies` は、xarray自身の内部spinlockである `xa_lock` で保護されている。そこに追加を行う通常のコード（`drm_sched_job_add_dependency()`、および反復処理を行う際の `drm_sched_entity_kill_jobs_cb()` 自身）は素の `xa_*` アクセサを使っており、これらは割り込みを無効化せずに `xa_lock` を取得する。

## きっかけ {#the-trigger}

`drm_sched_entity_kill_jobs_cb()` は `dma_fence` のコールバックとして登録されている。つまり `dma_fence_signal()` から、割り込みコンテキストで実行されうる。lockdepが実機上でその結果生じる割り込み非安全なロックのパターンを捕まえた後、Mikhail GavrilovがMesaに対してバグを報告した。トレースには完全なシナリオが記録されていた。

```
Possible interrupt unsafe locking scenario:

      CPU0                    CPU1
      ----                    ----
 lock(&xa->xa_lock#17);
                              local_irq_disable();
                              lock(&fence->lock);
                              lock(&xa->xa_lock#17);
 <Interrupt>
   lock(&fence->lock);

*** DEADLOCK ***
```

CPU0は、IRQ安全でない `xa_*` アクセサを通じて `job->dependencies` に触れる、通常のプロセスコンテキストの任意の関数である。CPU1は、割り込みコンテキストでフェンスのコールバックとして実行される `drm_sched_entity_kill_jobs_cb()` であり、CPU0がすでに保持している同じ `xa_lock` を必要とする。CPU0がロックを保持している最中に割り込みが入ると、CPU1は、割り込みハンドラが戻るまでCPU0が解放できないロックを待って永遠にスピンし、その割り込みハンドラ自体はCPU1を待っている。

## 観測された挙動 {#observed-behavior}

Christian Königは、xarrayのロックの割り込み安全性だけを修正したとしても、同じ関数がデッドロックしうる2つ目の独立した経路を特定した。`dma_fence_signal()` 自体がコールバックを呼び出す前に `fence.lock` を取得し、`drm_sched_entity_kill_jobs_cb()` 自身の反復処理は各依存関係に対して `dma_fence_add_callback()` を呼ぶ。これも*その*フェンスの `lock` を取得する。現在シグナルしているフェンスと、後始末の間に新たに待機対象となるフェンスが、たまたま同じ下位のspinlockを共有していた場合、コールバックはそれを呼び出したシグナルそのものに対してデッドロックすることになる。

```
dma_fence_signal() // f1.lock をロックする
-> drm_sched_entity_kill_jobs_cb()
-> foreach dependencies
   -> dma_fence_add_callback() // f2.lock をロックする
```

つまり、同じ設計判断に根ざした2つの独立したデッドロックの形があった。割り込みコンテキストで、すでにフェンスのロックを保持したまま実行されているコールバックの中で、xarrayの反復処理とフェンスのコールバック登録を直接行っていたことである。

## なぜ起きたか {#why-it-happened}

すべての `xa_*` 呼び出しを割り込み安全な `xa_*_irq` 版に切り替えれば、1つ目のデッドロックだけは修正できたが、2つ目は修正できなかった。フェンスのロックの入れ子問題は、xarrayのロックをどれほど注意深く割り込み安全にしても、そのロジックをシグナルのコールバック内で実行すること*自体*に構造的に伴うものである。部分的な修正（IRQ安全なxarrayアクセサのみ）は、その動機となったlockdepのトレースの下では完全に見えただろうが、Christian Königが見つけたフェンスのロックの入れ子の危険にはまったく対処しないままになっていた。

## 解決 {#resolution}

`487df8b69834` は、依存関係の反復処理と再設定のロジックをフェンスシグナルのコールバックから完全に取り出し、新しい `drm_sched_entity_kill_jobs_work()` に移す。これは `dma_fence_signal()` から直接呼び出されるコールバックとしてではなく、通常のワークとして実行される。これにより2つのデッドロックの形を同時に回避できる。workqueueのコンテキストは割り込みコンテキストではないので、xarrayのロックはもはや割り込み非安全ではない。また、どのフェンスの `lock` もすでに保持した状態で実行されるわけではないので、入れ子になった `dma_fence_add_callback()` の呼び出しが、コールバックを引き起こしたフェンスと衝突することはない。修正には `Cc: stable@vger.kernel.org # v6.2+` と、この後始末パスを最初に導入したコミットを示す `Fixes: 2fdb8a8f07c2`（"drm/scheduler: rework entity flush, kill and fini"）が付いている。

## 教訓 {#what-it-taught-us}

**lockdepが実際に報告したバグに範囲を絞った修正では、構造的に同一のバグが残ったままになりうる。** 割り込み非安全なxarrayのロックは発見されたものだった。フェンスのロックの入れ子は同じ関数における2つ目の独立したデッドロックであり、特定のlockdepの出力の先にある、周囲のコードの形を見る必要があった。

**`dma_fence` のシグナルコールバックとして実行されるコードは、一見必要とは思えない割り込みコンテキストやロックの入れ子の制約を引き継ぐ。** そのコードが元々は通常の後始末ロジックとして書かれ、周囲のコードの進化につれて後からシグナルパスから到達可能になった場合は特にそうである。このロジックを今の場所に置いたのは `2fdb8a8f07c2` の「rework」であり、デッドロックは最初から存在していたわけではない。

!!! warning "注意すべきパターン"
    `dma_fence` のコールバックとして呼び出されるロジック（`dma_fence_add_callback()` で直接登録されたものでも、そこから到達するものでも）は、フェンスがシグナルされるコンテキストで実行され、それは割り込みコンテキストでありうる。フェンスのコールバック関数の中で `_irq`/`_irqsave` 版を使っていないロック呼び出し（`xa_lock`、spinlock、mutex）をgrepし、コールバック自体が、シグナルしているフェンス自身のロックと入れ子になりうるフェンス機構を呼び返していないかを確認すること。

## 関連項目 {#see-also}

- [コマンド投入、`dma_fence`、GPUスケジューラ](../command-submission.md) — `dma_fence` のシグナル、コールバック、そしてこのバグの修正が今は守っているドライバ横断の契約
- [msm GPUの回復処理とshrinkerのデッドロック](msm-shrinker-deadlock.md) — 割り込み安全性の側ではなくメモリ回収の側で起きた、別の `dma_fence` 関連のデッドロック
- [RCU（Read-Copy-Update）](../../locking/rcu.md)と[アトミック操作とメモリバリア](../../locking/atomics.md) — 割り込みコンテキストにおけるロックの危険に関する一般的な背景

## 外部参考資料 {#external-references}

- [git.kernel.org: 487df8b69834](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=487df8b698345dd5a91346335f05170ed5f29d4e) — "drm/sched: Fix deadlock in drm_sched_entity_kill_jobs_cb"、完全なlockdepトレースと両方のデッドロックシナリオ
- [Mesa GitLab issue #13908](https://gitlab.freedesktop.org/mesa/mesa/-/issues/13908) — Mikhail Gavrilovによる最初の報告
