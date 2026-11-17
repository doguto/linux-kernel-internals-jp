# Xe TDR回復のリグレッション連鎖 {#the-xe-tdr-recovery-regression-chain}

> Intel Lunar LakeにおけるTDRのバグの修正が2度目のリグレッションを起こし、*その*リグレッションの修正がカーネルをクラッシュさせた。3つのコミット、20か月、1つのドライバ

収録
:   2024年11月 – 2026年7月（3コミットの連鎖）

ドライバ
:   Intel Xe（Lunar Lake以降）

作者
:   Matthew Brost、Rodrigo Vivi（Intel）

実際にハングを引き起こしていたか
:   はい、影響を受けるハードウェア上で最終修正まで — CVEではない

*[事件簿: GPU/DRMのバグとリグレッション](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

[コマンド投入、`dma_fence`、GPUスケジューラ](../command-submission.md)では、TDR（timeout detection and recovery、タイムアウト検出と回復）の仕組みを扱っている。ジョブのタイムアウトが経過すると `drm_sched_job_timedout()` が発火し、ドライバの `timedout_job()` コールバックが次に何が起こるかを決める。ハードウェアをリセットするか、スケジューラに `DRM_GPU_SCHED_STAT_NO_HANG` を伝えてジョブの実行を続けさせるかである。Xeの `guc_exec_queue_timedout_job()` は、IntelのGuCファームウェアスケジューラを通じて投入されたジョブに対するそのコールバックである。

そのコールバックの以前のバージョンは、実際には一度も実行を開始していないジョブを、スキップしてよい軽いケースとして扱っていた。GuCがその実行を始めてさえいないなら、リセットするものは何もないので、ハンドラはそれ以上何もせずにタイムアウトをやり過ごせばよい、というわけである。

## きっかけ {#the-trigger}

Lunar Lakeでは、GuCがGGTT（Global Graphics Translation Table）のページ更新を取りこぼし、ジョブをまったくスケジュールしないことがありえた。コールバックは開始していないジョブを明示的にスキップしていたため、この障害モードでは回復動作がまったく行われなかった。ジョブは永遠に放置され、ドライバのどこもそれに気づかなかった。

Matthew Brostの修正 [`fe05cee4d953`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=fe05cee4d9533892210e1ee90147175d87e7c053)（"drm/xe: Don't short circuit TDR on jobs not started"、2024年10月）は、この短絡を取り除いた。開始していないジョブに対して、TDRはジョブを動かすためにスケジューリングを切り替え、警告を出し、2回目のTDRの発火でもまだ開始していなければ本当にジョブをタイムアウトさせるようになった。これで元のハングは解消されたが、次のハングが生まれた。

## 観測された挙動 {#observed-behavior}

修正後のハンドラは、ハードウェアを実際に回復させるためのGTリセットを一度も起動せずに、開始していないジョブを黙ってエラーにするようになった。約20か月後のRodrigo Viviの修正 [`770031ec2312`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=770031ec2312bfab307d05db5469f24fd297e758)（"drm/xe: fix job timeout recovery for unstarted jobs and kernel queues"、2026年6月10日）は、問題を率直に述べている。"A job that GuC never scheduled (never started) indicates a GuC scheduling failure; previously such jobs were silently errored out instead of triggering a GT reset to recover."（訳: GuCが一度もスケジュールしなかった（開始しなかった）ジョブは、GuCのスケジューリング障害を示している。以前はそのようなジョブは、回復のためにGTリセットを起動する代わりに、黙ってエラーにされていた。）カーネルキュー（ユーザーの投入ではなく、ドライバ内部の作業）については、修正はさらに踏み込む。コミット自身の言葉では、それらは "always recovered this way and wedge the device once recovery attempts are exhausted, since kernel work must not silently fail"（訳: 常にこの方法で回復され、回復の試みが尽きたらデバイスをwedgeする。カーネルの作業は黙って失敗してはならないからである）であり、BANされたユーザーキューの回復とは違って無条件である。

その修正自体にも、レビューで見つかった危うい点があった。*すでに*BANされたキュー上の開始していないジョブは、そのままにしておく必要があった。意図的にBANされたキューのBANを解除したりGTリセットを強制したりすると、カーネルが意図的に殺したユーザー空間の作業を復活させてしまい、1つの不良キューがGTリセットの嵐に変わりうる。v3のリビジョンで、その例外が明示的に追加された。

Rodrigo Viviが `770031ec2312` を書いた2日後、それ自体がクラッシュを起こした。[`d42df9dce7b3`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d42df9dce7b374079c5c41691bd62d8765768a80)（"drm/xe: wedge from the timeout handler only after releasing the queue"、2026年6月12日）はフォールトを直接引用している。

```
Oops: general protection fault ... 0x6b6b6b6b6b6b6c3b
RIP: guc_exec_queue_timedout_job+...
```

`0x6b6b6b6b6b6b6c3b` は解放済みメモリに対するSLUBのポイズンパターンである。ハンドラは、もはや存在しないジョブを参照外ししていた。

## なぜ起きたか {#why-it-happened}

回復の試みを使い果たしたカーネルジョブは、`guc_exec_queue_timedout_job()` の内部から直接 `xe_device_declare_wedged()` を呼んでいた。その間、同じハンドラはタイムアウトしたジョブとそのスケジューラをまだ保持し、積極的に使っていた。Xeのデフォルトのwedgedモードでは、デバイスをwedgedと宣言すると、すべてのキューのスケジューラ（TDRハンドラがまだ操作していたものを含む）を停止する破壊的なパスを通り、投入処理を破棄して実行中のフェンスをシグナルする。

その後制御はハンドラに戻り、ハンドラはもう古くなったジョブとスケジューラを使い続けた。もはや停止状態ではないスケジューラに対して `drm_sched_for_each_pending_job()` を呼び（`WARN_ON` を引き起こした）、その反復処理は破棄処理がすでに解放したジョブを参照外しした。wedgeの呼び出しは単独で見れば正しかった。カーネルジョブを回復できないときには、それが*正しい*行動である。しかし、まさに破棄しようとしている状態をまだ所有しているハンドラの内部からそれを呼ぶのは正しくなかった。

## 解決 {#resolution}

`d42df9dce7b3` は、wedgeの呼び出しを、ハンドラがキューの操作を終えた後、`DRM_GPU_SCHED_STAT_NO_HANG` を返す直前にだけ発火するよう移動した。これにより、破棄処理がハンドラ自身によるジョブとスケジューラの使用と競合することはなくなった。3つのコミットはすべて同じ関数、`drivers/gpu/drm/xe/xe_guc_submit.c` の `guc_exec_queue_timedout_job()` に触れている。後の2つはそれぞれ、この連鎖の直前のコミットを指す `Fixes:` タグを持つ（最初のコミット自身の `Fixes:` タグはさらに遡り、この連鎖全体が回避しようとしている、元のジョブタイムアウトのサンプリングのコミットを指している）。

## 教訓 {#what-it-taught-us}

**「壊れていないものはリセットするな」と「本当に壊れているものは必ずリセットせよ」はどちらも正しく、その境界線こそが、ここで2度リグレッションした場所である。** 元の短絡は、時には本当に回復を必要とすることが判明したケース（開始していないジョブ）の回復をスキップしていた。それに対する修正は今度は、カーネルが意図的にBANしたために開始していなかったものも含めて、開始していない*すべての*ジョブを無条件に回復させた。それらを回復させること自体がバグだった。

**状態を破棄するハンドラは、その後もその状態を使い続けてはならない。** 最後のクラッシュは、デバイスを*いつ*wedgeするかの論理エラーではなかった。wedgeによって解放されるものへの参照をまだ保持しているコードとの相対で、*どこで*wedgeするかという寿命のエラーだった。

!!! warning "注意すべきパターン"
    より広範な破棄（デバイスのwedge、GTリセット、スケジューラの停止）を自ら引き起こしうるタイムアウト／回復のコールバックは、その破棄を引き起こす*前に*自身のローカルな状態の使用を終えるか、戻った後まで破棄を遅らせる必要がある。`timedout_job()` 風のコールバックの内部で、その呼び出しより下でローカルなjob/queue/schedulerのポインタがまだスコープ内にある状態で行われている `_declare_wedged()`、`_reset()` などの破壊的な呼び出しをgrepすること。

## 関連項目 {#see-also}

- [コマンド投入、`dma_fence`、GPUスケジューラ](../command-submission.md) — このリグレッションが存在するTDRの仕組みと、このドライバが最終的に返す `DRM_GPU_SCHED_STAT_NO_HANG` ステータス
- [`NO_HANG` の誤検知](no-hang-false-positive.md) — 同じステータス値についての、別のドライバでの別の話
- [カーネルモード設定（KMS）のオブジェクトモデルとアトミックモード設定](../kms.md) — 同じサブシステムの表示側

## 外部参考資料 {#external-references}

- [git.kernel.org: fe05cee4d953](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=fe05cee4d9533892210e1ee90147175d87e7c053) — "drm/xe: Don't short circuit TDR on jobs not started"
- [git.kernel.org: 770031ec2312](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=770031ec2312bfab307d05db5469f24fd297e758) — "drm/xe: fix job timeout recovery for unstarted jobs and kernel queues"
- [git.kernel.org: d42df9dce7b3](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d42df9dce7b374079c5c41691bd62d8765768a80) — "drm/xe: wedge from the timeout handler only after releasing the queue"、完全なGPFトレースを含む
