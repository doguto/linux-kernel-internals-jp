# スケジューラの公平性の進化 {#the-scheduler-fairness-evolution}

> ラウンドロビンは長いジョブキューを持つエンティティを飢餓状態にした。それを置き換えた FIFO ポリシーは、負荷の下で低優先度のエンティティを飢餓状態にした。FIFO を置き換えた fair ポリシーは、その答えを CFS から借りてきた

マージ
:   FIFO: Linux 6.2（2022年）· Fair: Linux 7.2（2026年）

スケジューラ
:   `drm_sched`。共通 DRM スケジューラを使うすべてのドライバで共有される

作者
:   Andrey Grodzovsky、Luben Tuikov（FIFO）· Tvrtko Ursulin（Fair）

CVE ではない

*[事件簿：GPU/DRM のバグとリグレッション](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

[コマンド投入、`dma_fence`、GPU スケジューラ](../command-submission.md)では、`drm_sched` のランキューを*エンティティ*の集合として説明している — エンティティはユーザー空間の投入コンテキストごとに1つあり、それぞれが自分のジョブキューを持ち、順番にハードウェアリングへ作業を送り出す。スケジューラの歴史の大半において、「順番に」とはラウンドロビンを意味していた。エンティティを順にたどり、順番が回ってきたエンティティからジョブを1つ取り出し、次のエンティティへ移る。

## きっかけ {#the-trigger}

Andrey Grodzovsky の [`08fb97de03aa`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=08fb97de03aa2205c6791301bd83a095abc1949c)（"drm/sched: Add FIFO sched policy to run queue"、2022年9月）は、観測された症状から書き始めている。"When many entities are competing for the same run queue on the same scheduler, we observe an unusually long wait times and some jobs get starved. This has been observed on GPUVis"（訳: 多くのエンティティが同じスケジューラ上の同じランキューを奪い合うと、異常に長い待ち時間が観測され、一部のジョブが飢餓状態になる。これは GPUVis で観測された）— GPUVis は AMD 自身の GPU ワークロード可視化ツールである。そのメカニズムは構造的なもので、従来の意味でのバグではなかった。ラウンドロビンは1パスごとにエンティティ1つにつきジョブを1つ取り出す。*そのエンティティのキューがどれほど長いか、最も古いジョブがどれほど古いかに関係なく*、である。長く古いバックログを持つエンティティも、短く新しいバックログを持つエンティティも、同じだけ順番を得た。同時に存在するエンティティが十分に多いと、あるジョブは、すでにどれだけ長く待っていようと、任意の数の他エンティティの順番の後ろに置かれうる — 需要の実際の不公平さではなく、エンティティの数に比例して拡大する飢餓である。

## 観測された挙動 {#observed-behavior}

修正は FIFO 順序だった。エンティティごとに順番を回すのではなく、`drm_sched_rq_select_entity_fifo()` はエンティティを、各エンティティの待機中の最も古いジョブのタイムスタンプをキーとする赤黒木に保持し、常に先頭ジョブが最も早く到着したエンティティを選ぶ — 選択は O(1)、更新は O(log N) である。Luben Tuikov の [`977d97f18b5b`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=977d97f18b5b8efb7a94da84724113f15ae6cc2d) は同じリリースでこれをスケジューラのデフォルトにし、"about 1% faster performance over the Round Robin algorithm"（訳: ラウンドロビンアルゴリズムより約1%高速）と測定したうえで、具体例を挙げて意図を説明した。複数のエンティティにまたがって A から J までのジョブがその時系列順に到着した場合、FIFO はそれらを到着順 — A、B、C、D…… — に実行する。順序から外れるのは、古いジョブが本当にまだ準備できていない場合だけで、その場合はより新しい準備済みのジョブが実行され、古いジョブは準備ができた瞬間にキューの先頭へ割り込む。

FIFO はラウンドロビンの飢餓のケースを解決した。だが別のものを持ち込んだ。厳密な時系列順序では、ジョブを投入し続けるエンティティは、キューを空にしない限り、低優先度エンティティのより古いジョブよりも自分のジョブを無期限に前に置き続けられる。Tvrtko Ursulin の [`2fa4d8e2c109`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2fa4d8e2c1091189064c5b93222a23ded2d881ba)（"drm/sched: Add fair scheduling policy"、2026年4月）は、FIFO が避けられなかったこの失敗モードを "total priority starvation"（訳: 完全な優先度飢餓）と名付けている。

## なぜ起きたか {#why-it-happened}

どちらのポリシーも、ある性質に最適化していたが、その性質は重要なものの唯一ではないことが判明した。ラウンドロビンはエンティティごとの順番回しに最適化しており、ジョブの古さが見えていなかった。FIFO はジョブの古さに最適化しており、優先度が見えておらず、バースト的な低優先度クライアントと、継続的に投入する GPU 負荷の高い高優先度クライアントとの非対称性も見えていなかった。どちらの問題もコーディングの誤りではなかった。各ポリシーは前任者の特定の失敗モードに対する本物の改善であり、それぞれの新しい失敗モードは、古いものが修正され、ワークロードが新しい隙間を突いて初めて見えるようになった。

## 解決 {#resolution}

`2fa4d8e2c109` は両方を CFS 風の仮想実行時間方式に置き換える。コミット自身の言葉では、"entity run queue is sorted by the virtual GPU time consumed by entities in a way that the entity with least vruntime runs first."（訳: エンティティのランキューは、エンティティが消費した仮想 GPU 時間でソートされ、vruntime が最も小さいエンティティが最初に実行される）。低優先度エンティティの実際の GPU 時間は、vruntime として計上される前に指数的な係数で拡大されるので、同じ実時間を費やしても通常優先度のエンティティより速く仮想実行時間が蓄積する — それにより、低優先度エンティティを飢餓状態にする（十分な数の競合エンティティがあるときのラウンドロビン）のでも、優先度を完全に踏みにじらせる（高優先度の負荷が持続するときの FIFO）のでもなく、比例的にランキューの順序の後ろへ押し下げる。このコミットは特に、オーバーサブスクリプションのワークロード — 深いジョブキューを投入するクライアントと並んで動く、短くバースト的なクライアント — を、fair スケジューリングがどちらの前任者よりもうまく扱うケースとして挙げている。

[`45c211ddf92a`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=45c211ddf92a1f9b4214ffadaf70d9037f53aaf6) は fair スケジューリングを新しいデフォルトにし、続いて [`77a6809f1dc3`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=77a6809f1dc39376116f8d769a0d2630dc95ad79) はラウンドロビンと FIFO の両方を完全に削除した — Ursulin の理由は、fair スケジューリングが "in general better than FIFO and almost as good as round-robin in interactive use cases"（訳: 一般に FIFO より優れ、対話的なユースケースではラウンドロビンとほぼ同等）であり、ラウンドロビンはいずれにせよ何年もデフォルトではなかったので、3つのポリシーを生かしておくことはスケジューラの複雑さの面で、捨てられたポリシーがまだもたらしているどんな利益よりも高くつく、というものだった。これにより、スケジューラの優先度ごとのランキューの配列も、単一の埋め込みランキューにまとめられた。fair スケジューリングは、優先度を別々のキューではなく vruntime のスケーリングによって表現するからである。

## 得られた教訓 {#what-it-taught-us}

**スケジューリングポリシーとは、どの公平性の性質が最も重要かに対する賭けであり、どの賭けにも死角がある。** ラウンドロビンはエンティティの順番回しに賭け、キュー長の偏りに負けた。FIFO は到着順に賭け、優先度に負けた。fair スケジューリングは連続的にスケールされる仮想時間の指標に賭ける — CFS がすでに CPU スケジューリングで行ったのと同じ賭けである — まさに、他を犠牲にして守るべき1つの次元（古さ、優先度、エンティティ数）を選ぶ必要がないからである。

**「これで私たちが見ている飢餓は直る」と「これで飢餓はなくなる」は異なる主張であり、その間の隙間こそが次のポリシーの生まれる場所である。** これまでの2つの移行はいずれも、理論上の懸念ではなく、実在する具体的な観測された症状 — GPUVis で見えたエンティティの飢餓、続いてオーバーサブスクリプション下の優先度飢餓 — によって推進された。どちらの症状も、前の修正がすでに広く使われるようになるまで見えなかった。

!!! warning "注意すべきパターン"
    1つの観測された飢餓パターンを動機とするスケジューリングポリシーの修正は、飢餓が一般に解決された証拠ではない — *その*パターンが解決された証拠である。新しいポリシーが明示的に考慮していない次の軸（優先度、バースト性、エンティティ数、キューの深さ）に注意すること。

## 関連項目 {#see-also}

- [コマンド投入、`dma_fence`、GPU スケジューラ](../command-submission.md) — このページが前提とするランキュー、エンティティ、スケジューリングポリシーの仕組み。アーキテクチャの要約は「Scheduling policy, three eras」（スケジューリングポリシーの3つの時代）を参照
- [スケジューラの進化](../../sched/scheduler-evolution.md) — CPU スケジューリング側の CFS と vruntime ベースの公平性。`drm_sched` の fair ポリシーが借りてきたモデル

## 外部参考文献 {#external-references}

- [git.kernel.org: 08fb97de03aa](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=08fb97de03aa2205c6791301bd83a095abc1949c) — "drm/sched: Add FIFO sched policy to run queue"、GPUVis で観測された飢餓の報告
- [git.kernel.org: 977d97f18b5b](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=977d97f18b5b8efb7a94da84724113f15ae6cc2d) — "drm/scheduler: Set the FIFO scheduling policy as the default"、時系列順序の具体例付き
- [git.kernel.org: 2fa4d8e2c109](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2fa4d8e2c1091189064c5b93222a23ded2d881ba) — "drm/sched: Add fair scheduling policy"
- [git.kernel.org: 45c211ddf92a](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=45c211ddf92a1f9b4214ffadaf70d9037f53aaf6) — "drm/sched: Switch default policy to fair"
- [git.kernel.org: 77a6809f1dc3](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=77a6809f1dc39376116f8d769a0d2630dc95ad79) — "drm/sched: Remove FIFO and RR and simplify to a single run queue"
