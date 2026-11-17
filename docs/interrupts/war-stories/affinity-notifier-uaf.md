# 自身の参照カウントより長生きしたアフィニティ通知 {#the-affinity-notifier-that-outlived-its-own-reference-count}

> コールバックを差し替えて古いものを解放する処理は、参照カウントがゼロになった時点で完結しているように見えた。ところが古いコールバックはまだworkqueueに入っていて、解放されたばかりのメモリ上で実行されようとしていることがあった

収録
:   Linux 5.2（2019年7月）· 修正の作成は2019年3月24日

作者
:   Prasad Sodagudi（作者）、Thomas Gleixner（修正を適用）

ヒューリスティック/APIの導入者
:   Ben Hutchings、`irq_set_affinity_notifier()`、2011年1月（Linux 2.6.39）。この修正の約8年前

CVEではない

*[事件簿: 割り込みとIRQアフィニティのバグ](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

`irq_set_affinity_notifier()` を使うと、ドライバは割り込みのCPUアフィニティが変わるたびに発火するコールバックを登録できる。通知自体はアフィニティが実際に変更されたコンテキストの外で行う必要があるため、通知のコールバックはインラインでは実行されない。カーネルは `struct work_struct` をキューに入れ、後で通常のプロセスコンテキストでそれを呼び出す。割り込みサブシステムと、まだ保留中のワークアイテムの両方が同時に参照を保持しうるため、notifierオブジェクトは `kref` で参照カウントされている。

## きっかけ {#the-trigger}

ドライバが `irq_set_affinity_notifier()` を再び呼んで*新しい*notifierをインストールし、以前に登録されていたものを置き換えると、関数は*古い*notifierの参照カウントを正しく減らす。その減算でカウントがゼロになると、`kref_put()` はnotifier自身の `release` コールバックを呼び出し、古いnotifierが占めていたメモリを解放する。この一連の処理のどこにも、古いnotifierのワークアイテムがまだworkqueueに入って実行を待っているかどうかを確認するものはなかった。

## 観測された挙動 {#observed-behavior}

参照カウントがゼロになった瞬間に古いnotifierのコールバックのワークがまだキューに入っていると、workqueueのスレッドがその中のコードを実行する保留中のジョブをまだ持っている間に、ワークが指すメモリが解放された。そのジョブが最終的に実行されると、すでに解放されたメモリ上で動作した。これはuse-after-freeであり、その後そのメモリが何に再割り当てされたかによっては、解放されたnotifierのメモリ配置とたまたま重なったworkqueue自身の内部リスト管理状態を破壊することもありえた。

## なぜ起きたか {#why-it-happened}

参照カウント自体が隙間だったわけではない。まだキューに入っているワークアイテムは本当に参照を1つ保持していた。`irq_set_affinity_locked()` は `schedule_work()` を呼ぶ前に `kref_get(&desc->affinity_notify->kref)` を呼び、コールバックである `irq_affinity_notify()` は実行を終えると対応する `kref_put()` で終わる。欠けていたのは、*その*ライフサイクルと、ドライバが置き換えのnotifierをインストールするときに `irq_set_affinity_notifier()` が使うライフサイクルとの間の同期だった。置き換えパスは、古いnotifierのワークがまだworkqueueに入っているかどうかを先に確認したり待ったりせずに、古いnotifierに対する自身の参照を手放していた。それはカウントをゼロにして解放する参照だったかもしれない。修正のコミットメッセージ自身が問題を率直に述べている。"nothing ensures that the old notifier is not longer queued in the work list. If it is queued this results in a use after free and possibly in work list corruption."（訳: 古いnotifierがもはやワークリストに入っていないことを保証するものが何もない。入っていればuse-after-freeとなり、ワークリストが破壊される可能性もある。）（タイプミスも含めてコミットの原文どおり。）正しくカウントされた参照がオブジェクトを守るのは、何かがそれを保持し続けている間だけである。最後の保持者が、別の場所でまだ未完了の使用に対して*いつ*手放すかについては何も語らない。その隙間を塞ぐには、より良いカウントではなく明示的な待機が必要だった。

## 解決 {#resolution}

修正は2行を追加する。古いnotifierに対して `kref_put()` を呼ぶ前に、`irq_set_affinity_notifier()` はまず `cancel_work_sync(&old_notify->work)` を呼ぶようになった。`cancel_work_sync()` は、ワークアイテムがすでに完了まで実行されたか、まったく実行されないかのどちらかが保証されるまでブロックする。つまり `kref_put()` が実行される時点では、解放されるnotifierに対して保留中のものは何も残っていない。

これで話が完全に終わったわけではなかった。1年後、コミット [`df81dfcfd699`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=df81dfcfd6991d547653d46c051bac195cd182c1)（Edward Cree、Linux 5.7でマージ）は、この修正がuse-after-freeを参照リークと引き換えにしていたことを発見した。`cancel_work_sync()` が未完了のワークを実際にキャンセルした場合、そのワーク自身の `kref_get()` に対応する `kref_put()` は呼ばれない。それを解放するはずだったコールバックが実行される機会を得なかったからである。現在のメインラインは、`cancel_work_sync()` の戻り値を確認し、実際の保留中ワークをキャンセルしたと報告された場合にのみ余分な参照を解放することでこれに対処している。
```c
if (old_notify) {
        if (cancel_work_sync(&old_notify->work)) {
                /* 保留中のワークが参照を持っていたので、それも解放する */
                kref_put(&old_notify->kref, old_notify->release);
        }
        kref_put(&old_notify->kref, old_notify->release);
}
```

## 教訓 {#what-it-taught-us}

**遅延実行されるワークのキャンセルと参照の解放は2つの別々の操作であり、その相互作用を正しくするのに2回の試みを要した。** 最初の修正は、参照を手放す前にワークをキャンセルすることでuse-after-freeを正しく塞いだが、*キャンセルされたワーク自身が保持していた*参照を考慮していなかった。クラッシュを緩やかなリークと引き換えにしたのである。

**「保留中のワークを、それが触れるものを解放する前にキャンセルする」は修正の一般的な形であり、このnotifier以外にも広く一般化できる。ただし、キャンセル自体の管理情報も考慮に入れた場合に限る。** 参照カウントに参加し、かつまだキューに入っている遅延コールバックの対象になりうるオブジェクトは、すべて「キャンセルしてから解放する」順序を必要とする。さらに、コールバック自体が参照を保持していて、キャンセルに成功するとその参照が宙に浮いてしまわないかを問う必要がある。

!!! warning "注意すべきパターン"
    ワークアイテム、タイマー、taskletがまだ参照している可能性のあるオブジェクトを解放する（あるいは `kref` に解放させる）前に、その遅延機構がキャンセルされたかどうかを問うこと。そして解放の前に、キャンセルを要求するだけでなく*同期的に*キャンセルすること。その上で逆方向も確認すること。遅延機構自体がカウントされた参照を保持していた場合、キャンセルに成功したということは、それを解放するはずだったコールバックが二度と実行されないため、その参照を明示的に解放する必要がある。

## 関連項目 {#see-also}

- [workqueue](../workqueues.md) — notifierのコールバックが実行される遅延実行の仕組み

## 外部参考資料 {#external-references}

- [git.kernel.org: 59c39840f5ab](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=59c39840f5ab) — "genirq: Prevent use-after-free and work list corruption"、修正コミット
- [git.kernel.org: df81dfcfd699](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=df81dfcfd6991d547653d46c051bac195cd182c1) — "genirq: fix reference leaks on irq affinity notifiers"、最初の修正が持ち込んだ参照リークに対するEdward Creeの追加修正
