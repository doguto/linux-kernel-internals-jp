# 降格時に継承を忘れたDEADLINEロック保持者 {#the-deadline-lock-holder-that-forgot-to-inherit-on-the-way-down}

> CVE-2026-23371 — 優先度継承mutexを保持しているSCHED_DEADLINEタスクが、保持中により低いスケジューリングクラスへ降格されると、待機しているドナーのデッドラインパラメータの継承を完全にスキップしてしまい、カーネル自身の帯域計上を破壊することがあった

報告者
:   Bruno Goncalves（Red Hat）。大規模マルチCPUマシン上のPREEMPT_RTカーネルで `stress-ng --schedpolicy 0` を実行して発覚

修正
:   コミット `d658686a1331`（"sched/deadline: Fix missing ENQUEUE_REPLENISH during PI de-boosting"）、Juri Lelli（Red Hat）。6.18.34と6.19.7にバックポートされ、メインラインはLinux 7.0

バグの存在期間
:   Linux 5.10以降、コミット `2279f540ea7d` による（これもJuri Lelli）

*[事件簿: スケジューラのバグ](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

SCHED_DEADLINEの優先度継承プロトコルでは、mutexでブロックされているタスクが、そのmutexを現在保持しているタスクに自身のデッドラインパラメータを一時的に譲渡（donate）できる。これにより保持者は、待機しているタスクと少なくとも同じくらい厳しいデッドラインで実行され、通常の優先度逆転がデッドラインミスにまで発展するのを防ぐ。ただし、DEADLINEタスクが常に待機者のパラメータを継承する必要があるわけではない。保持者*自身*のデッドラインが待機しているドナーのものよりすでに短ければ、継承するものはなく、カーネルは正しくそのままにしておく。

## きっかけ {#the-trigger}

この隙間が現れるのは、継承の必要がまったくなかったロック保持者が、mutexを保持したまま `sched_setscheduler()` によって*別の*、より優先度の低いスケジューリングクラスに変更されたときである。

1. あるDEADLINEタスク（「ドナー」）が、2つ目のDEADLINEタスク（「保持者」）が保持するPI mutexでブロックする。
2. 保持者自身のデッドラインはすでにドナーのものより厳しかったため、保持者はドナーのパラメータを継承しなかった。上記の規則に従えば正しい。
3. その後 `sched_setscheduler()` が、mutexを保持したままの保持者を `SCHED_DEADLINE` からより優先度の低いクラスに変更する。

この時点で保持者はブーストされた状態として認識されるべきである。まさに自力ではDEADLINEタスクでなくなろうとしているからこそ、*今*ドナーのDEADLINEパラメータを継承する必要があり、帯域計上が正しく扱えるよう `ENQUEUE_REPLENISH` フラグ付きで再エンキューされる必要がある。しかしステップ2で保持者の継承状態が一度も確立されていなかったため、`sched_setscheduler()` の既存ロジックのどこも、クラス変更の時点でその継承を発動させなかった。

## 観測された挙動 {#observed-behavior}

大規模マシン上のPREEMPT_RTカーネルで `stress-ng --schedpolicy 0` を実行すると（報告のトレースではCPU 93が挙げられている）、カーネル警告が発生した。

```
sched: DL de-boosted task PID 22725: REPLENISH flag missing
WARNING: CPU: 93 PID: 0 at kernel/sched/deadline.c:239 dequeue_task_dl+0x15c/0x1f8
```

続いて、`dequeue_task_dl → dequeue_task → deactivate_task → push_dl_task → dl_task_timer → __hrtimer_run_queues → hrtimer_interrupt` を経由して `running_bw` のアンダーフローが起きた。`ENQUEUE_REPLENISH` フラグが欠けていたため、`enqueue_task_dl()` はそもそも降格された保持者をブーストされたタスクとして認識せず、そのCPUに対するカーネル自身のDEADLINE帯域の管理情報が破壊されたままになった。

## なぜ起きたか {#why-it-happened}

DEADLINEタスクのクラス変更を扱う `sched_setscheduler()` の既存ロジックは、*すでに*ブーストされているものとして追跡されているタスク、つまり過去にどこかのドナーからパラメータを継承したタスクの扱い方しか知らなかった。正当な理由で何も継承する必要がなかった保持者（自身のデッドラインがすでに十分厳しかったため）は、そのロジックから完全に外れていた。継承しなかった*理由*は、`SCHED_DEADLINE` から完全に降格された瞬間に意味を失っていたにもかかわらずである。「このタスクは今継承する必要があるか」の条件はクラス切り替えの瞬間に黙って変わり、それを再評価するものは何もなかった。

## 解決 {#resolution}

修正は `__setscheduler_dl_pi()` を導入する。これは、`sched_setscheduler()` がDEADLINEタスク（ブーストされているかどうかを問わない）がより優先度の低いクラスに移されることを検出したときに限って呼ばれる。この新しい関数は、まさにその瞬間にタスクにPIドナーからDEADLINEパラメータを継承させ、続くエンキューで `ENQUEUE_REPLENISH` をセットする。これにより、以前に確立された継承状態が今も正しいと仮定するのではなく、今まさにブーストされつつあるタスクについて帯域計上が正しくやり直される。

## 教訓 {#what-it-taught-us}

**タスクの生涯のある時点で正しかった前提条件は、そのタスクの何か別のものが変わった瞬間に誤りになりうる。前提条件が依存する状態に直接触れるものが何もなくてもである。** 保持者が継承しないという判断は、下された時点では正しかった。`sched_setscheduler()` がタスクを `SCHED_DEADLINE` から外した瞬間にそれは正しくなくなり、その遷移で継承チェックを再実行するものは何もなかった。

**スケジューリングクラスの遷移は、ブースト／継承の状態が食い違いやすい自然な場所である。継承を管理するコードパスとクラス変更を管理するコードパスは、通常は別々に書かれ、別々に考察されるからだ。** 修正のスコープはまさにその交差点、つまり汎用の `sched_setscheduler()` パス内のDEADLINE固有のフックに限定されている。継承ロジック自体は終始それ自体としては正しかったので、そちらは変更していない。

!!! warning "注意すべきパターン"
    ある状態（「このタスクはドナーのパラメータを継承したか」）がある条件の下で確立され、別の条件の下で消費される場合は、その状態に直接触れるコードパスだけでなく、*条件*を変えうるすべてのコードパスについて、条件が変わる前の古いスナップショットのまま放置するのではなく状態を導出し直す必要がないかを確認すること。

## 関連項目 {#see-also}

- [スケジューラの概要](../README.md) — SCHED_DEADLINEとCBSの帯域計上
- [優先度逆転とPI mutex](../pi-mutexes.md) — このバグにおけるドナーと保持者の関係が依存する優先度継承プロトコル

## 外部参考資料 {#external-references}

- [git.kernel.org: security/vulns — CVE-2026-23371](https://git.kernel.org/pub/scm/linux/security/vulns.git/plain/cve/published/2026/CVE-2026-23371.mbox) — カーネルCVEチームの公式アナウンス。完全なトレース、根本原因の解説、各stableブランチにおける修正コミットを含む
- [git.kernel.org: d658686a1331](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d658686a1331db3bb108ca079d76deb3208ed949) — "sched/deadline: Fix missing ENQUEUE_REPLENISH during PI de-boosting"、Juri Lelliによる修正
