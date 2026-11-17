# Tasklet {#tasklets}

> softirqの上に構築された、直列化されたper-CPUの遅延処理

## taskletとは何か {#what-are-tasklets}

taskletは、softirq（`TASKLET_SOFTIRQ`と`HI_SOFTIRQ`）の上に構築された、ドライバから利用できる遅延機構である。生のsoftirqにはない2つの性質を提供する。

1. **直列化**: ある1つのtaskletは同時に1つのCPU上でしか実行されない。SMPシステムでも、同じtasklet関数が並行して呼ばれることはない
2. **動的な登録**: ドライバは新しいsoftirqの種類を定義することなく、独自のtaskletを登録できる

taskletはスリープできない（softirqコンテキストで動作する）。スリープが必要な遅延処理には[workqueue](workqueues.md)を使う。

## tasklet_struct {#the-tasklet_struct}

```c
/* include/linux/interrupt.h */
struct tasklet_struct {
    struct tasklet_struct *next;  /* 保留リストのリンク */
    unsigned long state;          /* TASKLET_STATE_SCHED, TASKLET_STATE_RUN */
    atomic_t count;               /* 0 = 有効、>0 = 無効 */
    bool use_callback;            /* unionのどのメンバーが有効か */
    union {
        void (*func)(unsigned long data);         /* レガシーAPI */
        void (*callback)(struct tasklet_struct *t); /* 現代的なAPI */
    };
    unsigned long data;           /* レガシー: 旧形式のfuncに渡される */
};
```

`use_callback`と`func`/`callback`のunionそのものが、後述する5.9でのセキュリティを動機とした改修の一部である。そのパッチセット以前は、`.func`が唯一のシグネチャであり、`.data`がそれにコンテキストを渡す唯一の方法だった。

## API {#api}

```c
#include <linux/interrupt.h>

/* 現代的なAPI: コールバックはtaskletへのポインタを受け取る */
void my_tasklet_handler(struct tasklet_struct *t)
{
    struct my_device *dev = from_tasklet(dev, t, tasklet);
    /* 処理を行う */
}

/* 宣言する（静的） */
DECLARE_TASKLET(my_tasklet, my_tasklet_handler);

/* あるいは動的に初期化する */
struct tasklet_struct my_tasklet;
tasklet_setup(&my_tasklet, my_tasklet_handler);

/* スケジュールする（IRQハンドラから呼べる） */
tasklet_schedule(&my_tasklet);    /* 通常優先度（TASKLET_SOFTIRQ） */
tasklet_hi_schedule(&my_tasklet); /* 高優先度（HI_SOFTIRQ） */

/* 無効化/有効化 */
tasklet_disable(&my_tasklet);   /* 実行中のものを待ってから無効化する */
tasklet_disable_nosync(&my_tasklet);  /* 待たずに無効化する */
tasklet_enable(&my_tasklet);    /* 再度有効化し、保留中だったならスケジュールする */

/* taskletの終了を待ってから停止させる */
tasklet_kill(&my_tasklet);
```

## from_tasklet: デバイスのコンテキストへのアクセス {#from_tasklet-accessing-device-context}

現代的なコールバックは`struct tasklet_struct *`を受け取る。それを含むデバイス構造体を得るには`from_tasklet()`を使う。

```c
struct my_device {
    struct tasklet_struct rx_tasklet;
    void *rx_buffer;
    int rx_len;
};

static void my_rx_tasklet(struct tasklet_struct *t)
{
    struct my_device *dev = from_tasklet(dev, t, rx_tasklet);
    /* dev->rx_buffer[0..dev->rx_len-1]を処理する */
}

static irqreturn_t my_irq_handler(int irq, void *data)
{
    struct my_device *dev = data;

    /* ハードウェアからバッファへデータをコピーする */
    dev->rx_len = readl(dev->regs + RX_LEN);
    memcpy_fromio(dev->rx_buffer, dev->regs + RX_DATA, dev->rx_len);

    /* 処理をtaskletに遅延させる */
    tasklet_schedule(&dev->rx_tasklet);
    return IRQ_HANDLED;
}
```

## 直列化の保証 {#serialization-guarantee}

taskletが別のCPUですでに実行中のときに`tasklet_schedule()`が呼ばれると、taskletはその実行が終わった後に再び実行される。しかし並行して実行されることは決してない。

```
CPU 0: tasklet running...
CPU 1: tasklet_schedule() → sets SCHED bit

After CPU 0 finishes: sees SCHED bit, runs tasklet again
→ tasklet function never runs simultaneously on two CPUs
```

これが、同じハンドラが複数のCPU上で同時に実行されうる生のsoftirqに対する重要な利点である。

## 誰も始めるつもりがなく、そして終わらせられなかった非推奨化 {#the-deprecation-nobody-planned-to-start-then-couldnt-finish}

taskletは2.3開発系列から存在しており、周囲の他の遅延処理機構がすべて進化していく間も、その形は本質的に変わらなかった。現在`include/linux/interrupt.h`にある非推奨のコメント、*"This API is deprecated. Please consider using threaded IRQs instead"*（訳: このAPIは非推奨である。代わりにスレッド化IRQの使用を検討してほしい）は、Linux 5.9（2020年）で初めて登場したものであり、誰も物議を醸すとは思っていなかったパッチの副産物として始まった。

2019年、Romain Perierは（早い段階からKees Cookも関わって）、taskletのコールバックのシグネチャを現代化する、セキュリティを動機としたパッチセットをkernel-hardeningメーリングリストに投稿した。レガシーAPIはコールバックに`unsigned long data`を渡し、コールバックはそれを実際に必要な型にキャストし直す。そこには`container_of()`も型チェックもない。修正、すなわち`tasklet_struct *`そのものを渡す`DECLARE_TASKLET()`/`tasklet_setup()`と、上で示した`from_tasklet()`のパターンは、最終的に5.9でマージされ、最終的なシリーズにはPerierとCookのそれぞれのコミットが入った（[LWN](https://lwn.net/Articles/830964/)）。

それが引き起こした議論は狭い範囲に留まらなかった。Peter Zijlstraはこのスレッドを使って、taskletを完全に削除することを強く求めた。*"I would _MUCH_ rather see tasklets go the way of the dodo [...] Can't we stage an extinction event here instead?"*（訳: taskletにはドードーと同じ道をたどってほしいと *切に*思う［…］ここはいっそ絶滅イベントを起こせないだろうか？）Thomas Gleixnerは、その心情には同意しつつ、APIを現代化するパッチに「渋々」ackを与えた。*"I'd rather see tasklets vanish from the planet completely, but that's going to be a daring feat."*（訳: taskletにはこの惑星から完全に消えてほしいが、それは大胆な離れ業になるだろう）（[LWN](https://lwn.net/Articles/830964/)）Sebastian Andrzej Siewiorは置き換え先の1つとしてスレッド化IRQを提案した。taskletの本体は本質的にそのままスレッド化ハンドラの`thread_fn`に移り、これはプロセスコンテキストで動作するため、softirqコンテキストで動くtaskletの本体にはできなかったことを何でもできる。一方Dmitry Torokhovは、それ以外のケースには即座に満了するタイマーを提案した。

それはまさに「大胆な離れ業」となった。同じ議論の中で、その理由を示す2つの例が浮上した。AMDの`ccp`暗号ドライバはtaskletをDMAエンジンと組み合わせており、単一の置き換え先にきれいに対応付けられない。そしてIntelの`i915` GPUドライバは、慎重な性能テストなしには誰も触りたがらないパスでtaskletを使ってGPUタスクをスケジュールしている。削除はドライバごと、サブシステムごとに何年もかけて進められた（Takashi Iwaiは数週間後にサウンドサブシステムの変換の準備ができたと報告した）が、SiewiorとTorokhovの提案がカバーしていない高速パスのケースについては、専用の機械的な置き換え先は存在しなかった。

その隙間を埋めるのが`WQ_BH`である。2024年1月、気に入らないtaskletの変種の提案に返信する中で、Linus Torvaldsはもっと根本的なものを提案した。*"look at introducing a 'low-latency atomic workqueue' that looks exactly like a regular workqueue, but has the rule that it's per-cpu and functions on it cannot sleep... I think if we introduced a workqueue that worked more like a tasklet — in that it's run in softirq context — but doesn't have the interface mistakes of tasklets, a number of existing workqueue users might decide that that is exactly what they want."*（訳: 通常のworkqueueとまったく同じに見えるが、per-cpuであり、その上の関数はスリープできないという規則を持つ「低レイテンシのアトミックなworkqueue」の導入を検討してほしい……もしtaskletにより近い動作をする、つまりsoftirqコンテキストで実行されるが、taskletのインターフェースの誤りを持たないworkqueueを導入すれば、既存のworkqueue利用者のうち少なからぬ数が、それこそが自分の求めるものだと判断するかもしれない）。workqueueのメンテナであるTejun Heoはこれを`WQ_BH`として実装し、Linux 6.9でマージされた。`WQ_BH` workqueueに投入されたワークアイテムは、アトミック（softirq）コンテキストで、キューに積まれたのと同じCPU上で素早く実行される。これはworkqueue APIがこれまでに提供した中で、最もtaskletのそのままの置き換えに近いものである。当時LWNが取り上げた版では、まだ変換されていないtaskletに対する優先度逆転を避けるために、`WQ_BH`のワークアイテムを内部的にtaskletから実行していた（[LWN](https://lwn.net/Articles/960041/)）。長期的にsoftirqが生き残ることに依存しないものではなく、そもそもソフトウェア割り込みを使うという点がSiewiorの反対を招いた。彼はtaskletの利用者が代わりにスレッド化IRQか通常のworkqueueに移ることを望んでいた。Heoの答えは、そのどちらも可能な限り短いレイテンシを必要とするケースの助けにはならない、というものであり、まさにそれが、実際のtaskletを経由しなくなった後も`WQ_BH`がsoftirqベースのままである理由である。実際にマージされた版までに、実装はtaskletを一切経由せず、taskletのsoftirq自身のaction関数に直接フックする形に変わっていた。これはより低コストであり、taskletが完全になくなった後の最終的なコード構造にもより近い。数週間のうちに一括変換の取り組みが続いた（[LWN](https://lwn.net/Articles/966894/)、Allen Paisの`[PATCH 0/9] Convert Tasklets to BH Workqueues`）が、当時カーネル内にはtaskletの利用者が優に500を超えて存在しており、変換は完了しておらずいまだ進行中である。現在のメインラインでも、tasklet APIはまだ存在し、実際の利用者がいる。

この議論は今回が初めてですらない。[LWNは](https://lwn.net/Articles/239633/)2007年に本質的に同じ削除提案を取り上げている。Steven Rostedtのパッチはtaskletの実装を取り除き、内部でworkqueueに支えられた互換ラッパーで置き換えるもので、既存のtaskletを使うコードを変更する必要がないようにしていた。tasklet自体をなくすことへの反対はほとんどなかった（上で述べた割り込みレイテンシの議論はすでによく理解されていた）が、ラッパーそのものを気に入った人は「ほぼ誰も」いなかった。ある仕組みを別の仕組みの背後に隠す互換シムは、まさにカーネルの「安定した内部APIを持たない」という規範が避けようとするものだからである。そのため、本当の代替案はすべてのtasklet利用者を直接workqueueに変換することとなったが、これははるかに大きな作業であり当時は誰も引き受けなかった。おそらくそれが、その回の議論が何も生まなかった理由である。2019〜2020年と2024年の回で違うのは、誰もまだ行っていない変換作業の代役としてのラッパーではなく、直接的で機械的な代替手段（最初はスレッド化IRQとworkqueue、次に特に`WQ_BH`）が実際に存在し、成熟しているという点である。

**削除日がなくても非推奨化が定着した実際的な理由**:

- softirqコンテキストであるため、スリープできず、機能が限られる
- 直列化が現代のハードウェアには粗すぎる
- workqueueとスレッド化IRQは、より少ない落とし穴で同じ性質を提供する

新しいドライバはtaskletの代わりに`workqueue`（または`threaded IRQ`）を使うべきである。

```c
/* taskletの代わりにwork_structを使う */
struct my_device {
    struct work_struct rx_work;  /* taskletの代わり */
};

static void my_rx_work(struct work_struct *work)
{
    struct my_device *dev = container_of(work, struct my_device, rx_work);
    /* スリープやmutexの取得などが可能 */
}

/* IRQハンドラからスケジュールする */
schedule_work(&dev->rx_work);
```

## taskletの動作の観察 {#observing-tasklet-activity}

```bash
# taskletのsoftirq回数
cat /proc/softirqs | grep -E "TASKLET|HI"
#     HI:        4        2        1        3
# TASKLET:    12345    12234    12123    12234

# TASKLETの回数が多いことは、taskletが多用されていることを示す
# スケジューリングの問題を引き起こしているならworkqueueへの移行を検討する
```

## 参考文献 {#further-reading}

- [Softirq](softirq.md) — taskletの土台となっているもの
- [Workqueue](workqueues.md) — 推奨される現代的な代替手段
- [スレッド化IRQ](threaded-irq.md) — ドライバのIRQ処理のためのもう1つの現代的な代替手段
