# 「自分にできる仕事はあるか？」と頻繁に問いすぎるコスト {#the-cost-of-asking-is-there-any-work-for-me-too-often}

> newidle バランスのコスト追跡をめぐる古い条件を削除したことで、ウェイクアップの多いワークロードが、ほとんど存在しない仕事を確認するだけで CPU 時間の5分の1を費やしうることが明らかになった — そしてそれに対する最初の修正は、自身のベンチマークでは効果的だったものの、データベースのワークロードで測定可能なリグレッションを引き起こし、別のアプローチが出荷される前にリバートされなければならなかった

報告者
:   Chris Mason（Meta）、`schbench` によるベンチマーク

最初の修正（後にリバート）
:   commit `155213a2aed4`（"sched/fair: Bump sd->max_newidle_lb_cost when newidle balance fails"）、メインライン Linux 6.17（2025年9月）

リバート
:   commit `d206fbad9328`（"sched/fair: Revert max_newidle_lb_cost bump"）、Peter Zijlstra。データベースワークロードのリグレッションに関する複数の独立した報告を受けて

修正（出荷版）
:   commit `33cf66d88306`（"sched/fair: Proportional newidle balance"）、メインライン Linux 6.19（2026年2月）

*[事件簿：スケジューラのバグ](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

CPU がアイドルに入ろうとするとき、`sched_balance_newidle()` はスケジューリングドメインの階層を上にたどり、各ドメインに、引っ張ってこられる実行可能なタスクがどこかにないかを問い合わせる — これが「newidle バランス」である。このチェックはレイテンシに敏感なアイドル突入パスで実行されるので、各ドメインは自身の newidle バランス試行のコストを `sd->max_newidle_lb_cost` で追跡し、その推定値を時間とともに減衰させる。そうすれば、高コストなドメインは、すべてのウェイクアップを遅らせるのではなく、やがてスキップされるようになる。

## きっかけ {#the-trigger}

Chris Mason は、まさにこのパスに負荷をかけるよう設計されたワークロードを使い、`schbench` のリグレッションを以前のコミット `c5b0a7eefc70`（"sched/fair: Remove sysctl_sched_migration_cost condition"、Vincent Guittot、メインライン Linux 5.16）までバイセクトした。`schbench -L -m 4 -M auto -t 256 -n 0 -r 0 -s 0` は4つのメッセージスレッドを CPU 0-3 に固定し、1024個のワーカースレッドを残りの CPU に分散させる。どのスレッドも実際の仕事はせず、ただ互いをできるだけ速く起こしては再びスリープに戻るだけである。バイセクトで特定したコミットをリバートすると、v6.9 カーネルでスループットが 3.4M リクエスト/秒から 5.4M RPS に戻った。このリグレッションは、この特定の形のワークロードでバイセクトされるまで、v5.16 以来 — 約3年半 — 潜在していた。

## 観測された挙動 {#observed-behavior}

`schedstat` は newidle バランスの試行が以前の約100倍になっていることを示し、プロファイリングでは、ワーカースレッドが CPU 時間の約20%を newidle バランスの中で費やしていることがわかった — その試行のほぼすべてが、仕事を引っ張ってくる忙しいグループを見つけられずに失敗していた。Guittot のコミットが削除した条件は、このコストの大部分を黙って抑えていた。それがなくなると、「自分にできる仕事はあるか？」という失敗するチェックの1つ1つが、設計上ほとんど本当に「はい」という答えが返らないワークロードの上で、フルコストで実行されるようになった。

## なぜ起きたか {#why-it-happened}

`sysctl_sched_migration_cost` は元々、粗く静的なしきい値として newidle バランスを制限していた。その条件の削除は、それ自体としては妥当に見える単純化だった — リバートされておらず、現在のメインラインでもこの条件は存在しないままである — が、同時に、CPU がアイドルになってまた実行可能になる時間スケールが、どのロードバランス間隔が想定していたよりもはるかに短いワークロードにおいて、newidle バランスの頻度を抑えていた唯一のものも取り除いてしまった。削除そのものに個別に誤りがあったわけではない。それが残したコストモデルには、仕事を見つけるのに失敗し続けるドメインにペナルティを科す仕組みがなく、成功はするが遅いことを続けるドメインにペナルティを科す仕組みしかなかったのである。

## 解決 {#resolution}

Mason の最初の修正 `155213a2aed4` は、newidle バランスの試行がタスクを引っ張ってこられなかったときには常に、`sched_balance_newidle()` が `sd->max_newidle_lb_cost` を積極的に引き上げる（`domain_cost = (3 * sd->max_newidle_lb_cost) / 2` によって）ように変更した。推定値が際限なく大きくならないよう、上限は `sysctl_sched_migration_cost` に小さなマージンを加えた値とした。`schbench` ではこれは意図どおりに機能した。しかしこれはツリーに残らなかった。何人かが、その直接の結果としてデータベースワークロードでのリグレッションを報告した — Adam Li（Ampere）は SpecJBB で6%のリグレッションを報告し、Joseph Salisbury（Oracle）、Dietmar Eggemann（Arm）、Hazem Mohamed Abuelfotoh（Amazon）からもさらに報告があった — そして Peter Zijlstra が `d206fbad9328` でこれをリバートした。

その代わりに、Zijlstra の `33cf66d88306` は別のアプローチを取った。コストベースのペナルティではなく、各ドメインの newidle バランスの*成功率*（`sd->newidle_ratio`）を追跡し、それを使ってバランスの試行そのものを確率的にスキップする — "throw a 1k sided dice; and only run newidle_balance according to the success rate"（訳: 1000面のサイコロを振り、成功率に応じてのみ newidle_balance を実行する）。これは新しい `NI_RANDOM` スケジューラ機能フラグの背後に置かれた。同じ `schbench` のケースでの Zijlstra 自身の測定では、Mason の修正をリバートして代わりにランダム化アプローチを加えたもの（2.18 Mrps/s）は、Mason の修正が達成した `schbench` 限定の数値（6.18-rc4、つまり Mason の修正がまだ適用された状態で 2.22 Mrps/s）に迫り、何も代わりを入れずにリバートした状態（2.04 Mrps/s）を明確に上回った — そしてパッチには Chris Mason 自身がテスターとしてクレジットされている。

## 現状 {#current-status}

話はそこでも終わらなかった。2026年1月、Zijlstra は `9fe89f022c05`（"sched/fair: More complex proportional newidle balance"）を投稿し、一部のワークロード（easyWave、fio）では newidle バランスの成功率がかなり低いにもかかわらず、それを試みることで大きな恩恵を受けると報告した — `NI_RANDOM` の純粋な確率モデルではうまく扱えないケースである。このパッチは `NI_RATE` 機能フラグの背後に置かれた、レートベースの2つ目の項を追加するもので、今も存在する `NI_RANDOM` と並んでメインライン Linux 7.0 にマージされた。Reported-by のクレジットには Mario Roy と Hazem Mohamed Abuelfotoh（Amazon）が含まれている — SpecJBB のリグレッションと同じ報告者であり、今回はチューニングのトレードオフの反対側からの報告である。

## 得られた教訓 {#what-it-taught-us}

**静的なしきい値を削除すると、たとえそのしきい値が単体では無駄な重荷に見えても、あるワークロードにとって唯一の防御を取り除くことがある。** `sysctl_sched_migration_cost` の制限条件は何も高度なことをしていなかった — しかしウェイクアップの嵐のようなワークロードにとっては、「高度なことは何もしないが、少なくともめったにやらない」ことが荷重を支えていた。

**成功した高コストな操作しか追跡しないコスト推定器には、安価だが絶えず失敗する操作に対する死角がある。** 元の `max_newidle_lb_cost` モデルは、避けるべきものは*遅い*バランス試行だと想定していた。実際のリグレッションは、単にあまりにも頻繁に実行される*速い*バランス試行から来ており、既存の計上の仕組みでは見ることのできない失敗モードだった。

**報告されたバグを明確に解決する修正でも、リグレッションした1つのドメインだけでなくすべてのドメインの挙動を変えるのであれば、誤った修正でありうる。** Mason のコスト引き上げはグローバルに適用され、たまたま、データベースワークロードが依存している、成功率は低いが価値のある newidle バランスをちょうど損なった — そのコストは `schbench` には決して現れなかった。`schbench` の newidle バランスは、ほとんど構造上、無価値だからである。

!!! warning "リバートは本物のシグナルであり、いつものスケジューラチューニングではない"
    このケースは、2つの独立した改善が相次いで入ったのではなく、本物の訂正である。Mason の修正は、別の、同じく実在するクラスのワークロードをリグレッションさせたという、まさにその理由でリバートされた。教訓は抽象的な「スケジューラのヒューリスティクスは反復改善されるものだ」ではない — 1つの合成的なウェイクアップ嵐ベンチマークで検証された newidle バランスの修正は、4つの別々のベンダーから本番環境での報告が届くまで、データベースワークロードへの影響について何も語っていなかった、ということである。

## 関連項目 {#see-also}

- [スケジューラの概要](../README.md) — ロードバランシングとスケジューリングドメインの階層
- [スケジューラの進化](../scheduler-evolution.md) — CFS のバランシングロジックが時間とともにどう変わってきたか

## 外部参考文献 {#external-references}

- [git.kernel.org: 155213a2aed4](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=155213a2aed42c85361bf4f5c817f5cb68951c3b) — "sched/fair: Bump sd->max_newidle_lb_cost when newidle balance fails"、Chris Mason の修正。`schbench` によるバイセクトの完全な説明付き
- [git.kernel.org: d206fbad9328](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d206fbad9328ddb68ebabd7cf7413392acd38081) — "sched/fair: Revert max_newidle_lb_cost bump"、Peter Zijlstra によるリバート。SpecJBB のリグレッション報告付き
- [git.kernel.org: 33cf66d88306](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=33cf66d88306663d16e4759e9d24766b0aaa2e17) — "sched/fair: Proportional newidle balance"、`NI_RANDOM` を導入した Peter Zijlstra の代替修正。`schbench` と SpecJBB の前後比較の測定付き
