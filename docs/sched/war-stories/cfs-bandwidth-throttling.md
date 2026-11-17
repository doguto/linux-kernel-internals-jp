# 使われる前に期限切れになったCFS帯域スライス {#the-cfs-bandwidth-slice-that-expired-before-it-was-spent}

> `cpu.cfs_quota_us` の下で動くマルチスレッドの非CPUバウンドなアプリケーションが、クォータを実際には使い切っていないのにスロットリングされることがあった。各CPUが持つクォータのローカルスライスが、使われたかどうかにかかわらず毎周期の終わりに期限切れになっていたからである

報告者
:   Kubernetesユーザー（[kubernetes/kubernetes#67577](https://github.com/kubernetes/kubernetes/issues/67577) として公開で追跡）。根本原因の特定と修正はDave Chiluk（Indeed）

バグの存在期間
:   CFS帯域制御が最初に導入されたLinux 3.2の時点からのCPUごとのスライス期限切れ設計。2014年から無関係な修正（`51f2176d74ac`）によって覆い隠され、Linux 4.18でコミット `512ac999d275`（"sched/fair: Fix bandwidth timer clock drift condition"）によって再び露呈した

修正
:   コミット `de53fd7aedb1`（"sched/fair: Fix low cpu usage with high throttling by removing expiration of cpu-local slices"）、メインラインLinux 5.4（2019年11月）

修正の効果の測定値
:   人工的な80 CPUのベンチマーク（10ms/100msのクォータ）でスループットが約30倍向上

*[事件簿: スケジューラのバグ](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

CFS帯域制御（`cpu.cfs_quota_us`/`cpu.cfs_period_us`）は、cgroupのクォータ全体を最初に要求したCPUに渡すわけではない。スケジューリング判断のたびにグローバルなロックを取るのを避けるため、クォータは必要に応じて小さなスライスに分けて各CPUのローカルなランキュー（`cfs_rq`）に配られ、`cfs_rq->runtime_remaining` で追跡される。各スライスは、それが引き出された帯域周期に結び付いた期限のタイムスタンプ `cfs_rq->runtime_expires` を持っていた。未使用のスライスが周期をまたいで際限なく蓄積されるのを許すべきではない、という考えである。

## きっかけ {#the-trigger}

2014年にLinux 3.16向けにマージされたコミット `51f2176d74ac` は、無関係なロックなし読み出しの競合を修正するため、この期限切れロジックの周りにチェックを追加した。

```c
if (cfs_rq->runtime_expires != cfs_b->runtime_expires) {
    /* ローカルの期限を延長する。ドリフトは上限2ティックに収まる */
    cfs_rq->runtime_expires += TICK_NSEC;
```

この条件分岐には、実際にはスライスが事実上決して期限切れにならなくなるという副作用があった。それが5年近く続いたが、コミット `512ac999d275`（"sched/fair: Fix bandwidth timer clock drift condition"）が*その*バグを修正し、本当の周期ごとの期限切れを復活させた。正しい期限切れを復活させたことで、根底にある設計上の問題が露呈した。個々のスレッドが少しずつ仕事をしてはスリープする、多数のスレッドを持つアプリケーションは、クォータの消費を多数のCPUに薄く分散させる。各CPUは小さなスライスを引き出し、その一部を使い、スライスに時間を残したまま周期の境界に達する。そのスライスは今や本当に期限切れとなって破棄され、直後にそれを必要とするスレッドが使えるようにはならなかった。

## 観測された挙動 {#observed-behavior}

スライスが再び実際に期限切れになるようになると、CPUクォータの下でマルチスレッドの非CPUバウンドなワークロードを実行しているKubernetesやMesosのユーザーは、高い割合の周期でスロットリングが起きるのを目にし始めた。`cpuacct` の計上では、コンテナが割り当てられたクォータの消費に近づいたことは一度もないと示されていたにもかかわらずである。ワークロードはCPUバウンドではなく対話的だった。これはまさに、使われなかった実行時間を多数の異なるCPUのローカルスライスに同時に置き去りにしやすいパターンである。

## なぜ起きたか {#why-it-happened}

2014年の修正と、本当の期限切れを再有効化した2019年の修正は、どちらも作者の目の前にあった実際のバグ（ロックなし読み出しの競合、そしてクロックドリフトの条件）に対する個別には正しい対応だった。どちらのパッチも、CPUごとのスライスがそもそも期限切れになるべきかという根底の設計判断を評価してはいなかった。それぞれ対処する特定のバグに狭くスコープを絞っていた。設計上の欠陥、すなわち多数のCPUのローカルスライスに断片化したクォータは、cgroup全体としては上限に近づいてもいないのに周期の境界で黙って消えうる、という欠陥は、5年間潜在し、最初のバグによって事実上覆い隠されていた。そして2つ目のバグ修正が偶然の回避策を取り除いて初めて、本番環境で見えるようになった。

## 解決 {#resolution}

Dave Chilukの修正であるコミット `de53fd7aedb1` は、期限切れロジックにさらにパッチを当てるのではなく、CPUローカルなスライスの期限切れを完全に取り除く。CPUのローカルな `cfs_rq` 上の未使用クォータは、破棄されずに周期の境界を越えて持ち越せるようになった。上限はCPUごとに `min_cfs_rq_runtime`（1ms）なので、ある1周期におけるグループの総クォータの最悪の超過は小さく、自己制限的である。Chiluk自身の再現（10ms/100msのクォータを持つ合成的な80 CPUのテストケースで、[fibtest](https://github.com/indeedeng/fibtest) として公開）では、変更後にスループットが約30倍向上し、より長い時間枠ではクォータ制限が正しく保たれたままだった。

## 教訓 {#what-it-taught-us}

**性能バグの修正は、設計上の欠陥に対する偶然の緩和策を黙って取り除くことがある。** 2014年のロックなし読み出しの修正は間違っていなかったが、本当のスライス期限切れを無効化するという副作用が、帯域の断片化問題のコストを5年間ひそかに吸収していた。クロックドリフトのバグを正しく修正したことが、古い設計上の欠陥を再び見えるようにしたのである。

**「クォータを使い切っていないのにスロットリングされた」は、上限が間違っているのではなく、計上の粒度が間違っている兆候である。** ここでの修正は誰かのクォータを引き上げることではなかった。間の悪い瞬間にたまたま間違ったCPUの台帳に載っていたクォータの断片を破棄するのをやめることだった。

!!! warning "注意すべきパターン"
    ロックのスケーラビリティのために、リソースが多数の独立した追跡単位（ここでは、cgroupごと・周期ごとのCPUのローカル実行時間スライス）に細分化されている場合、細分化単位ごとに適用される期限切れやリセットのポリシーは、推論やベンチマークが最も容易なCPUバウンドで単一のホットCPUのケースだけでなく、すべての細分化単位に同時に使用量を薄く分散させるワークロードに対しても評価する必要がある。

## 関連項目 {#see-also}

- [スケジューラの概要](../README.md) — CFSと帯域制御
- [cgroup v2のCPUコントローラ](../../cgroups/cgroup-v2.md) — `cpu.max`/`cpu.cfs_quota_us` がこの仕組みにどう対応するか

## 外部参考資料 {#external-references}

- [git.kernel.org: de53fd7aedb1](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=de53fd7aedb100f03e5d2231cfce0e4993282425) — "sched/fair: Fix low cpu usage with high throttling by removing expiration of cpu-local slices"、修正。Dave Chilukによる根本原因の詳細な解説付き
- [git.kernel.org: 512ac999d275](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=512ac999d275) — "sched/fair: Fix bandwidth timer clock drift condition"、本当の期限切れを再有効化し設計上の欠陥を露呈させた修正
- [LWN: sched/fair: Fix low cpu usage with high throttling...](https://lwn.net/Articles/792268/) — Dave Chilukが投稿したパッチ（v5）、LWNによるアーカイブ
- [kubernetes/kubernetes#67577](https://github.com/kubernetes/kubernetes/issues/67577) — 根本原因の調査を後押しした公開のユーザー報告
