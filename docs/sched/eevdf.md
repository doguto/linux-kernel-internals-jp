# EEVDFスケジューラ {#eevdf-scheduler}

> Earliest Eligible Virtual Deadline First: 公平性にレイテンシの保証を加える

## CFSの問題点 {#the-problem-with-cfs}

CFSは、`vruntime`が最も小さいタスクを選ぶ。これは公平ではあるが、デッドラインを意識していない。

I/Oから起床したばかりのタスクを考えてみよう。そのvruntimeは他のタスクより遅れているかもしれないので、すぐに実行される。しかし、CFSには*緊急性*という概念がない。1msでの応答が必要なネットワークパケットを処理するタスクも、たまたま少しの間スリープしていただけのバッチタスクも、スケジューリング上の優先度は同じである。

EEVDFは、CFSの「最小のvruntime」による選択を**「適格なタスクの中で最も早いデッドライン」**による選択に置き換えることで、これを解決する。

## 背景 {#background}

EEVDFは、Ion StoicaとHussein Abdel-Wahab（William & Mary）による1995年の技術レポート["Earliest Eligible Virtual Deadline First: A Flexible and Accurate Mechanism for Proportional Share Resource Allocation"](https://citeseerx.ist.psu.edu/document?repid=rep1&type=pdf&doi=805acf7726282721504c8f00575d91ebfd750564)に基づいている。

Linux向けにはPeter Zijlstra（Intel）が実装し、カーネル6.6（2023年10月）でマージされた。**パッチシリーズ**: [lore.kernel.org, May 2023](https://lore.kernel.org/lkml/20230531115839.089944915@infradead.org/)

**コミット**:

- [`147f3efaa241`](https://git.kernel.org/linus/147f3efaa24182a21706bca15eab2f3f4630b5fe) — `sched/fair: Implement an EEVDF-like scheduling algorithm`
- [`5e963f2bd465`](https://git.kernel.org/linus/5e963f2bd4654a202a8a05aa3a86cb0300b10e6c) — `sched/fair: Commit to EEVDF`（`pick_next_entity`でEEVDFを無条件に使うようにする）

## 2つの概念: 適格性と仮想デッドライン {#two-concepts-eligibility-and-virtual-deadline}

### 適格性 {#eligibility}

タスクが現在の仮想時間までに、CPUの公平な取り分を超えて消費していなければ、そのタスクは**適格（eligible）**である。くだけた言い方をすれば、実行しすぎているタスクは不適格であり、他のタスクをプリエンプトすべきではない。

```c
// kernel/sched/fair.c
static bool vruntime_eligible(struct cfs_rq *cfs_rq, u64 vruntime)
{
    struct sched_entity *curr = cfs_rq->curr;
    s64 avg = cfs_rq->avg_vruntime;
    long load = cfs_rq->avg_load;

    if (curr && curr->on_rq)
        avg += entity_key(cfs_rq, curr) * curr->load.weight;

    return vruntime * load < avg;
}
```

タスクの`vruntime`がランキューの重み付き平均vruntime以下であれば、そのタスクは適格である。

### 仮想デッドライン {#virtual-deadline}

各タスクは`deadline`を持つ。これは、そのタスクがそれまでに実行されているべき仮想時間である。タスクがランキューに配置されるとき、そのデッドラインは次のように設定される:

```c
// kernel/sched/fair.c
se->deadline = se->vruntime + calc_delta_fair(se->slice, se);
```

これは「現在のvruntimeから始めて、`slice`分の仮想時間だけ実行したい」という意味である。デッドラインはその時間窓の終わりである。

`slice`が短いタスクは早いデッドラインを得て、より早く実行される。`slice`が長いタスクは遅いデッドラインを得て、待たされることがある。

## sched_entityの新しいフィールド {#the-new-fields-in-sched_entity}

EEVDFは`struct sched_entity`（`include/linux/sched.h`）にいくつかのフィールドを追加した:

```c
struct sched_entity {
    // ... 既存のCFSのフィールド ...
    u64 vruntime;       // 仮想実行時間（依然として中心的）
    u64 deadline;       // この要求の仮想デッドライン
    u64 slice;          // 要求したタイムスライス
    u64 vlag;           // 仮想lag: 理想からどれだけ遅れている/進んでいるか
    // ...
};
```

### vlag {#vlag}

`vlag`は、理想的な公平配分スケジュールに対するタスクのlagを追跡する。正のlagは、タスクが理想の取り分より遅れている（十分なCPUを得ていない）ことを意味し、負のlagは進んでいることを意味する。

タスクが起床するとき、`vlag`はその`vruntime`を適切に配置するために使われる。これにより、スリープしていたタスクが飢餓状態に陥る（何も得られない）ことも、バーストする（長いスリープの後にCPUを独占する）ことも防がれる。

## 選択: __pick_eevdf() {#selection-__pick_eevdf}

中核となる選択関数:

```c
// kernel/sched/fair.c
static struct sched_entity *
__pick_eevdf(struct cfs_rq *cfs_rq)
{
    struct rb_node *node = cfs_rq->tasks_timeline.rb_root.rb_node;
    struct sched_entity *se, *curr, *best = NULL;

    while (node) {
        se = __node_2_se(node);

        // このタスクは適格か？
        if (entity_eligible(cfs_rq, se)) {
            // 適格なタスクの中で最も早いデッドラインか？
            if (!best || deadline_gt(best->deadline, se->deadline))
                best = se;
        }

        // 部分木のmin_deadlineを使って探索を枝刈りする
        // （すべての部分木を探索する必要はない）
        if (node->rb_left &&
            __node_2_se(node->rb_left)->min_deadline <= se->deadline)
            node = node->rb_left;
        else
            node = node->rb_right;
    }

    return best;
}

static struct sched_entity *pick_eevdf(struct cfs_rq *cfs_rq)
{
    return __pick_eevdf(cfs_rq);
}
```

このアルゴリズムは赤黒木をたどって適格なタスクをすべて見つけ、その中でデッドラインが最も早い（最も小さい）ものを返す。適格でないタスクはスキップされる。効率のため、木の走査は部分木ごとにキャッシュされた`min_deadline`の値を使って枝刈りされる。

## EEVDFがレイテンシをどう改善するか {#how-eevdf-improves-latency}

### シナリオ: バッチタスク + 対話型タスク {#scenario-batch-interactive-task}

CFSの場合（簡略化）:
```
Time:    0   1   2   3   4   5   6   7   8ms
Batch:   ████████░░░░░░░░████████
Network: ░░░░░░░░████████
                ↑ woke here, had to wait for batch's vruntime to be overtaken
```

EEVDFの場合:
```
Time:    0   1   2   3   4   5   6   7   8ms
Batch:   ████████░░░░░░░░░░░░░░░░
Network: ░░░░░░░░████░░░░░░░░░░░░
                ↑ eligible + earliest deadline → runs immediately
```

ネットワークタスクは早いデッドラインを持ち（要求したスライスが小さい）、適格である（起床したばかり）。EEVDFは、vruntimeがずれていくのを待たずに、それをすぐにスケジュールする。

## タイムスライスとsched_base_slice_ns {#time-slice-and-sched_base_slice_ns}

EEVDFが6.6でマージされたとき、古い`sched_latency_ns`/`sched_min_granularity_ns`のチューニング可能なパラメータは、`sched_base_slice_ns`に置き換えられる形で非推奨になった（[`e4ec3318a17f`](https://git.kernel.org/linus/e4ec3318a17f5dcf11bc23b2d2c1da4c1c5bb507)、Peter Zijlstra）。これらのsysctlはまだ存在するが、今では`sched_base_slice_ns`が主要なつまみである。基本タイムスライスは現在、単一のつまみで制御される:

```bash
# 基本タイムスライス（デフォルト: 3ms = 3000000ns）
cat /proc/sys/kernel/sched_base_slice_ns
```

各タスクの`slice`は、`sched_base_slice_ns`を平均に対するそのタスクの重みでスケールした値に設定される。要求するスライスが大きいタスクは遅いデッドラインを得る。スライスが小さいタスクは早いデッドラインを得て、レイテンシが低くなる。

アプリケーションは、周波数スケーリングのために`SCHED_FLAG_UTIL_CLAMP_MIN`/`SCHED_FLAG_UTIL_CLAMP_MAX`を付けた`sched_setattr()`で特定のレイテンシのヒントを要求できるが、ユーザー空間からスライスを直接制御する手段はまだ公開されていない。

## CFSとEEVDFの選択の比較 {#comparing-cfs-and-eevdf-selection}

| 観点 | CFS | EEVDF |
|--------|-----|-------|
| 選択基準 | `min(vruntime)` — 赤黒木の最左ノード | 適格なタスクの中での`min(deadline)` |
| 適格性のチェック | なし | `vruntime ≤ avg_vruntime` |
| 起床したタスクのレイテンシ | vruntimeの差に依存する | 適格かつデッドラインが早ければ即座 |
| 飢餓の防止 | vruntimeの収束による | 適格性とデッドラインの組み合わせによる |
| 最悪ケースの選択 | O(1)（最左ノード） | O(log n)（枝刈りされた木の走査） |
| タイムスライス | 暗黙的、レイテンシの目標から導かれる | 明示的（`se->slice`） |

## 変わらなかったもの {#what-didnt-change}

EEVDFが置き換えたのは`fair_sched_class`内の*選択*である。vruntimeの仕組み、重みの計算、グループスケジューリング、負荷分散、エンキュー/デキューのパスはほとんど変わっていない。`sched_prio_to_weight`テーブル、`calc_delta_fair()`、`update_curr()`、cgroupとの統合はすべてCFSから引き継がれている。

つまり、EEVDFはCFSの置き換えではなく進化である。実際にはEEVDFであるものを「CFS」と呼ぶのはますます誤称になりつつあるが、`fair_sched_class`は今もその名前を保っている。

## EEVDFの挙動を確認する {#checking-eevdf-behavior}

```bash
# タスクごとのsched情報（vruntime、deadline、sliceを含む）
cat /proc/$PID/sched

# スケジューラの起床とスイッチのレイテンシ
perf sched record -a sleep 10
perf sched latency --sort=max

# スケジューリングの判断をリアルタイムに監視する
trace-cmd record -e sched:sched_switch -e sched:sched_wakeup ./workload
trace-cmd report
```

## 参考文献 {#further-reading}

- [CFS](cfs.md) — EEVDFが土台にしているvruntimeの基盤
- [ランキュー](runqueues.md) — EEVDFが動作するCPUごとのコンテキスト
- [LWN: An EEVDF CPU scheduler for Linux](https://lwn.net/Articles/925371/) — Peter Zijlstraによる解説
- [kernel/sched/fair.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/fair.c) — 1010行目の`__pick_eevdf()`
