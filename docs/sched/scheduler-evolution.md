# スケジューラの進化: O(1) → CFS → EEVDF {#scheduler-evolution-o1-cfs-eevdf}

> Linuxのスケジューラは30年間で3度変わった — その経緯と理由

## すべてのスケジューラが解く問題 {#the-problem-every-scheduler-solves}

カーネルは1秒間に何千回も、ひとつの問いに答えなければならない: **次にどのタスクを実行すべきか？**

その答えは次の条件を満たさなければならない:

- **公平** — どのタスクも飢餓状態にしてはならない
- **高速** — タスクの選択は安価でなければならない
- **応答性** — 対話的なタスクには低いレイテンシが必要である
- **スケーラブル** — 数百のCPUで動作する

4つすべてを完璧に満たすスケジューラはない。Linuxスケジューラの各世代は、それぞれ異なるトレードオフを選んできた。

---

## 最初のスケジューラ（1991–2001） {#the-original-scheduler-19912001}

Linux 1.0は単純なO(n)スケジューラを使っていた: 次のタスクを選ぶために、実行可能なすべてのタスクを走査し、「goodness」スコアが最も高いものを選ぶ。（O(n)とは、タスクを選ぶ時間が実行可能タスク数に比例して線形に増えることを意味する — 100タスクなら1タスクの約100倍かかる。）

これはプロセスが数個なら問題なく動いた。数百になるとそうはいかなかった — スケジューリングの判断のたびに、実行可能タスク数に比例した時間がかかったのである。その欠点は、Ingo Molnár自身によるO(1)の発表で説明されている: [LWN, January 2002](https://lwn.net/2002/0110/a/scheduler.php3)。

---

## O(1)スケジューラ（2.6.0、2003年） {#the-o1-scheduler-260-2003}

**作者**: Ingo Molnar | **発表**: [LWN, January 2002](https://lwn.net/2002/0110/a/scheduler.php3)

Linux 2.6.0はO(1)スケジューラを導入した。鍵となる洞察は、**優先度を固定サイズのビットマップに事前計算しておく**ことである。

### 動作の仕組み {#how-it-worked}

CPUごとに2つの優先度配列（`active`と`expired`）があり、それぞれが140段階の優先度レベルと、どのレベルに実行可能タスクがあるかを示すビットマップを持つ:

```c
struct prio_array {
    unsigned int nr_active;
    DECLARE_BITMAP(bitmap, MAX_PRIO + 1);
    struct list_head queue[MAX_PRIO];
};
```

次のタスクを選ぶには:
1. `sched_find_first_bit(bitmap)` — O(1)で、空でない最高優先度のキューを見つける
2. そのキューから最初のタスクを取り出す

タスクがタイムスライスを使い切ると、新たに計算されたタイムスライスとともに`active`から`expired`へ移される。`active`が空になったら、ポインタを入れ替える。

### なぜ高速だったか {#why-it-was-fast}

走査がない。タスクの選択は、実行中のタスク数に関係なく、1回のビットマップ操作で済んだ。これは大規模システムにおいて本物の勝利だった。

### 問題: 公平性のヒューリスティクス {#the-problem-fairness-heuristics}

O(1)スケジューラは、対話的なタスクをブーストしCPUバウンドなタスクにペナルティを与えるために「対話性」スコアを計算していた。これらのヒューリスティクスは悪名高いほど脆かった:

- ゲームやマルチメディアアプリが、ときにCPUバウンドと誤分類された
- スリープしていたタスクが巨大な優先度ブーストを得て起床し、他のタスクを飢餓状態にすることがあった
- ヒューリスティクスは不透明だった — あるタスクがなぜ実行された（あるいはされなかった）のかを理解するには、数百行の複雑な推定コードを読む必要があった

Linuxのコントリビュータであり麻酔科医でもあるCon Kolivasは、これらの公平性の問題を修正するパッチを何年もかけて書いた。彼のRSDL（Rotating Staircase Deadline）スケジューラ（[LKML, March 2007](https://lkml.kernel.org/lkml/200703041800.53360.kernel@kolivas.org/)、[LWN](https://lwn.net/Articles/224654/)で紹介）は、根本的な問題を浮き彫りにした: **対話性の検出というアプローチ自体が間違っている**。スケジューラは近似的に公平であるのではなく、証明可能な形で公平であるべきなのだ。

---

## CFS: Completely Fair Scheduler（2.6.23、2007年10月） {#cfs-completely-fair-scheduler-2623-october-2007}

**作者**: Ingo Molnar（Con Kolivasの仕事に触発された） | **発表**: [LKML, April 2007](https://lkml.org/lkml/2007/4/13/180) | [LWNの記事](https://lwn.net/Articles/230574/)

**コミット**: [2bd8e7d04a0a](https://git.kernel.org/linus/2bd8e7d04a0a)（最初のCFSマージ）

CFSは、O(1)スケジューラのヒューリスティクスを、ひとつの洗練された概念 — **仮想実行時間（virtual runtime）** — で置き換えた。

### 中核となるアイデア {#the-core-idea}

すべてのタスクは`vruntime` — 消費したCPU時間を優先度で重み付けしたもの — を持つ。スケジューラは常に`vruntime`が最小のタスクを実行する。

これは証明可能な形で公平である: 時間が経てば、同じ優先度のすべてのタスクは等しいCPU配分に収束する。

### 仮想時間と重み {#virtual-time-and-weight}

すべてのタスクが平等なわけではない。nice -20のタスクは、nice +19のタスクより多くのCPUを得るべきである。CFSはこれを**重み**によって扱う:

```c
// kernel/sched/core.c
const int sched_prio_to_weight[40] = {
 /* -20 */     88761,     71755,     56483,     46273,     36291,
 /* -15 */     29154,     23254,     18705,     14949,     11916,
 /* -10 */      9548,      7620,      6100,      4904,      3906,
 /*  -5 */      3121,      2501,      1991,      1586,      1277,
 /*   0 */      1024,       820,       655,       526,       423,
 /*   5 */       335,       272,       215,       172,       137,
 /*  10 */       110,        87,        70,        56,        45,
 /*  15 */        36,        29,        23,        18,        15,
};
```

nice値が1段階変わるごとに、重みはおよそ1.25倍異なる。nice 0のタスクは重み1024、nice -1では重み1277（約25%多い）である。

vruntimeを更新するとき、時間は重みでスケーリングされる:

```c
// kernel/sched/fair.c
static u64 calc_delta_fair(u64 delta, struct sched_entity *se)
{
    if (unlikely(se->load.weight != NICE_0_LOAD))
        delta = __calc_delta(delta, NICE_0_LOAD, &se->load);
    return delta;
}
```

重みの大きいタスクのvruntimeは実時間に比べてゆっくり進み、重みの小さいタスクのvruntimeは速く進む。結果として、重みの大きいタスクは、同じ「vruntime最小」のルールで選ばれながらも、より多くの実CPU時間を得る。

### 赤黒木 {#the-red-black-tree}

実行可能なタスクは、vruntimeをキーとする赤黒木（`struct rb_root_cached`）に格納される。最左ノードは常にvruntimeが最小のタスク — 次に実行するタスク — である。

```
          [vruntime=100]
         /              \
  [vruntime=80]    [vruntime=130]
       /
 [vruntime=60]  ← pick this next
```

エンキュー、デキュー、最小値の検索はすべてO(log n)である。実際には、ランキューは十分に小さいので、事実上定数時間となる。

### 主要なデータ構造 {#key-data-structures}

```c
// include/linux/sched.h — タスクごと
struct sched_entity {
    struct load_weight  load;           // 重み（nice値から導出）
    u64                 vruntime;       // 仮想実行時間
    u64                 exec_start;     // update_curr()が最後に実行された時刻
    u64                 sum_exec_runtime; // 消費した実CPU時間の合計
    // ...（EEVDFのフィールドは後に追加された）
};

// kernel/sched/sched.h — CPUごとのCFSランキュー
struct cfs_rq {
    struct rb_root_cached tasks_timeline; // 実行可能タスクのRBツリー
    struct sched_entity  *curr;           // 現在実行中のエンティティ
    unsigned int          nr_queued;      // キュー内のエンティティ数
    // ...
};
```

### 更新ループ {#the-update-loop}

タイマーティックごと、およびコンテキストスイッチのたびに、`update_curr()`が実行される:

```c
// kernel/sched/fair.c
static void update_curr(struct cfs_rq *cfs_rq)
{
    struct sched_entity *curr = cfs_rq->curr;
    u64 now = rq_clock_task(rq_of(cfs_rq));
    u64 delta_exec = now - curr->exec_start;

    curr->exec_start = now;
    curr->sum_exec_runtime += delta_exec;
    curr->vruntime += calc_delta_fair(delta_exec, curr);

    // update_min_vruntime(cfs_rq);
}
```

### CFSの長所 {#cfs-strengths}

- **ヒューリスティクスがない**: 公平性は近似ではなく数学的な性質である
- **予測可能**: vruntimeによってスケジューリングの挙動を検証できる
- **ほとんどのワークロードに適する**: サーバー、デスクトップ、コンテナのワークロードすべてが恩恵を受ける

### CFSの短所 {#cfs-weaknesses}

時が経つにつれ、新たな問題が浮上した。CFSの「vruntime最小」というアプローチは公平性には最適化されているが、レイテンシには最適化されていない。起床してすぐに実行する必要があるタスクでも、技術的にvruntimeがより小さいタスクの後ろで待たされることがある。たとえそれらのタスクがしばらく実行し続けていたとしてもである。

リアルタイムやレイテンシに敏感なワークロード（ネットワークI/O、オーディオ、ゲーム）には、公平性の保証だけでなく**レイテンシの保証**を与えられるスケジューラが必要である。

---

## EEVDF: Earliest Eligible Virtual Deadline First（6.6、2023年10月） {#eevdf-earliest-eligible-virtual-deadline-first-66-october-2023}

**作者**: Peter Zijlstra（Intel） | **パッチシリーズ**: [lore.kernel.org, May 2023](https://lore.kernel.org/lkml/20230531115839.089944915@infradead.org/)

**コミット**: [`147f3efaa241`](https://git.kernel.org/linus/147f3efaa24182a21706bca15eab2f3f4630b5fe)（EEVDFの導入）、[`5e963f2bd465`](https://git.kernel.org/linus/5e963f2bd4654a202a8a05aa3a86cb0300b10e6c)（EEVDFへの無条件移行）

EEVDFは、Ion StoicaとHussein Abdel-Wahabによる1995年の学術論文（[CiteSeerX](https://citeseerx.ist.psu.edu/document?repid=rep1&type=pdf&doi=805acf7726282721504c8f00575d91ebfd750564)）に基づいている。これは公平性モデルに**仮想デッドライン**を追加する。

### 鍵となる洞察 {#the-key-insight}

CFSの問い: *最もCPUを使っていないのは誰か？*（vruntime最小）

EEVDFの問い: *実行する資格のある（eligibleな）タスクのうち、最も切迫したデッドラインを持つのは誰か？*

この違いはレイテンシにとって重要である。スリープ後に起床したタスクは短い仮想デッドラインを持つ — すぐに実行される必要がある。EEVDFは、他のタスクに対する公平性を損なうことなく、そのタスクにCPUを与えることができる。

### 適格性と仮想デッドライン {#eligibility-and-virtual-deadlines}

各タスクは次のものを持つ:

- `vruntime` — 消費した仮想時間（CFSと同じ）
- `deadline` — それまでに実行されなければならない仮想時刻
- `slice` — 要求するタイムスライス
- `vlag` — 理想的な公平配分に対して、タスクがどれだけ遅れている／進んでいるか

タスクの`vruntime`がランキューの現在の仮想時刻と同じかそれより遅れている — 「前借り」していない — 場合、そのタスクは**適格（eligible）**である:

```c
// kernel/sched/fair.c
static bool vruntime_eligible(struct cfs_rq *cfs_rq, u64 vruntime)
{
    struct sched_entity *curr = cfs_rq->curr;
    s64 avg = cfs_rq->avg_vruntime;
    long load = cfs_rq->avg_load;

    if (curr && curr->on_rq) {
        unsigned long weight = scale_load_down(curr->load.weight);
        avg += entity_key(cfs_rq, curr) * weight;
        load += weight;
    }

    return avg >= (s64)(vruntime - cfs_rq->min_vruntime) * load;
}
```

適格なタスクのうち、EEVDFは**最も早い仮想デッドライン**を持つものを選ぶ:

```c
// kernel/sched/fair.c
static struct sched_entity *pick_eevdf(struct cfs_rq *cfs_rq)
{
    return __pick_eevdf(cfs_rq);
}
```

デッドラインはタスクがエンキューされるときに設定される:

```c
// kernel/sched/fair.c
se->deadline = se->vruntime + calc_delta_fair(se->slice, se);
```

### 比較: CFSとEEVDF {#comparison-cfs-vs-eevdf}

| 特性 | CFS | EEVDF |
|----------|-----|-------|
| 選択基準 | vruntime最小 | 適格なタスクのうち最も早いデッドライン |
| 起床タスクのレイテンシ | 悪い（vruntimeが追いつくのを待つ必要がある） | 良い（すぐに適格になり、デッドラインが短い） |
| 公平性の保証 | あり（vruntimeの収束） | あり（適格性とデッドラインの組み合わせ） |
| タイムスライス | 動的、暗黙的 | 明示的（`se->slice`） |
| 基になったもの | Rotating Staircase Deadline（Con Kolivas） | Stoica & Abdel-Wahab, 1995 |

### 実際に何が変わったか {#what-changed-in-practice}

ほとんどのワークロードでは違いはわずかである — CFSはすでに十分良かった。EEVDFが主に役立つのは次の場合である:

- **ネットワーク負荷の高いサービス**: I/O後に起床するタスクへの応答が速くなる
- **オーディオ/マルチメディア**: スケジューリングの周期がより一定になる
- **混在ワークロード**: CPUバウンドなタスクがレイテンシに敏感なタスクを遅らせにくくなる

### `kernel.sched_latency_ns`ノブ {#the-kernelsched_latency_ns-knob}

EEVDFでは、既存の`sched_latency_ns` sysctlがタスクに割り当てられる`slice`に直接影響するようになり、より意味のあるものになった:

```bash
# デフォルト: 6ms（6000000 ns）
cat /proc/sys/kernel/sched_latency_ns

# 半分にするとタスクごとのタイムスライスが短くなる（レイテンシは下がり、オーバーヘッドは増える）
# 倍にすると応答性を犠牲にしてスループットが上がる
```

---

## まとめ {#summary}

| 時代 | スケジューラ | バージョン | 主な革新 | 主な弱点 |
|-----|-----------|---------|----------------|---------------|
| 1991–2003 | 最初のもの | 1.0–2.4 | 単純なgoodness指標 | O(n)の選択 |
| 2003–2007 | O(1) | 2.6.0–2.6.22 | ビットマップによるO(1)の選択 | 脆い対話性ヒューリスティクス |
| 2007–2023 | CFS | 2.6.23–6.5 | 仮想実行時間、証明可能な公平性 | 起床タスクのレイテンシが悪い |
| 2023– | EEVDF | 6.6+ | 仮想デッドライン + 適格性 | まだ発展途上（活発に開発中） |

## 参考文献 {#further-reading}

- [kernel/sched/fair.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/fair.c) — CFSとEEVDFの実装
- [LWN: An EEVDF CPU scheduler for Linux](https://lwn.net/Articles/925371/)（2023年）
- [LWN: Introducing the Completely Fair Scheduler](https://lwn.net/Articles/230501/)（2007年）
- EEVDFの原論文: Stoica & Abdel-Wahab, "Earliest Eligible Virtual Deadline First"（1995年）
