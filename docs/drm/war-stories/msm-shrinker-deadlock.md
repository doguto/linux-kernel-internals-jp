# msm GPUの回復処理とshrinkerのデッドロック {#the-msm-gpu-recoveryshrinker-deadlock}

> ハングしたGPUの回復ワーカーが、ジョブ実行スレッドが必要とするまさにそのロックを保持したままメモリを割り当て、その割り当てが、止まったジョブがシグナルするはずだったフェンスを待つshrinkerを起動した

収録
:   2026年4月（Linux 7.1）

ドライバ
:   msm（Qualcomm Adreno）

作者
:   Sergey Senozhatsky（Google/ChromeOS）

修正
:   [`4625fe5bbdac`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4625fe5bbdaccd45be274c30ff0a42e30d4e38cf)

CVEではない

*[事件簿: GPU/DRMのバグとリグレッション](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

[コマンド投入、`dma_fence`、GPUスケジューラ](../command-submission.md)では、ドライバ横断の `dma_fence` シグナル契約を説明している。`dma_fence` の待機はメモリ回収の奥深くまで重要な役割を担っており、ページアロケータ自体が前進するためにGPUのフェンスを待つことになりうる。そのため、フェンスのシグナルをブロックしうるコードは、そのページの言葉を借りれば「`dma_resv` ロックを取ってはならず、`GFP_KERNEL` で割り当ててはならず、`GFP_NOFS`/`GFP_NOIO` で割り当ててもならない」。この事件は同じ危険の一段離れた変種である。シグナルパス上で直接割り当てるのではなく、シグナルするスレッド自身が必要とするロックを保持したまま割り当てを行ったのである。

msmのGPU hangcheck機構は、GPUが最後に完了したフェンスと、最後に投入を求められたフェンスを比較することで、止まったリングを検出する。hangcheckが発火すると、回復ワーカーは `gpu->lock` を取得し、デバッグ用のクラッシュ状態ダンプを取得し、リングのリセットと再起動を試みる。

## きっかけ {#the-trigger}

そのクラッシュ状態ダンプの取得では、ダンプバッファを割り当てるために `kvmalloc_node()` を呼ぶ。これは `gpu->lock` を保持したまま行われる素のカーネル割り当てである。十分なメモリプレッシャーの下では、その割り当てが直接回収に入り、msm自身のGEM shrinker（`msm_gem_shrinker_scan()`）を呼び出し、それがさらに `drm_gem_lru_scan()` → `active_purge()` を呼んだ。`active_purge()` は、未完了のフェンスのページを回収する前に、それらがシグナルされるのを `dma_resv_wait_timeout()` で待つ。

## 観測された挙動 {#observed-behavior}

Sergey Senozhatskyの修正コミットは、この状態で止まったデバイスから実際の症状を直接引用している。hangcheckが繰り返し発火し、決して解消しなかった。

```
[..]
msm_dpu ae01000.display-controller: [drm:hangcheck_handler] *ERROR* (IPv4: 1): hangcheck detected gpu lockup rb 0!
msm_dpu ae01000.display-controller: [drm:hangcheck_handler] *ERROR* (IPv4: 1): completed fence: 7840161
msm_dpu ae01000.display-controller: [drm:hangcheck_handler] *ERROR* (IPv4: 1): submitted fence: 7840162
msm_dpu ae01000.display-controller: [drm:hangcheck_handler] *ERROR* (IPv4: 1): hangcheck detected gpu lockup rb 0!
msm_dpu ae01000.display-controller: [drm:hangcheck_handler] *ERROR* (IPv4: 1): completed fence: 7840162
msm_dpu ae01000.display-controller: [drm:hangcheck_handler] *ERROR* (IPv4: 1): submitted fence: 7840163
[..]
```

コミットメッセージには、ブロックされた両方のスレッドのhung-taskダンプも含まれている。ジョブ実行のカーネルスレッドは、回復ワーカーが保持するmutexで2分以上ブロックされていた。

```
INFO: task ring0:155 blocked for more than 122 seconds.
...
task:ring0 state:D stack:0 pid:155 ppid:2 flags:0x00000008
Call trace:
...
__mutex_lock_slowpath+0x28/0x40
mutex_lock+0x5c/0x90
msm_job_run+0x9c/0x140
drm_sched_main+0x514/0x938
...
```

そして回復ワーカー自身は、その同じmutexを保持したまま、メモリ回収の中でフェンスを待ってブロックされていた。

```
...
task:gpu-worker state:D stack:0 pid:154 ppid:2 flags:0x00000008
Call trace:
...
dma_fence_default_wait+0x108/0x218
dma_fence_wait_timeout+0x6c/0x1c0
dma_resv_wait_timeout+0xe4/0x118
active_purge+0x34/0x98
drm_gem_lru_scan+0x1d0/0x388
msm_gem_shrinker_scan+0x1cc/0x2e8
shrink_slab+0x228/0x478
...
__vmalloc_node_range+0x1c0/0x420
kvmalloc_node+0xe8/0x108
msm_gpu_crashstate_capture+0x1e4/0x280
recover_worker+0x1c0/0x638
...
```

循環は、`dma_fence` 契約が警告しているとおりの形で閉じる。ジョブスレッドは前進してフェンスをシグナルするために `gpu->lock` を必要とした。回復ワーカーは `gpu->lock` を保持しており、割り当てを終えてロックを解放するために（shrinkerの回収待ちを通じて）その同じフェンスを必要とした。どちらの側も進めなかった。コミット自身の言葉では "So no one can make any further progress"（訳: そのため誰もそれ以上前進できない）である。

## なぜ起きたか {#why-it-happened}

クラッシュダンプの割り当ては、`dma_fence` やメモリ回収のことをまったく考えずに書かれた普通のコードだった。表面上、`msm_gpu_crashstate_capture()` はフェンスとは何の関係もないからである。これは診断用のツールであり、高速パスではない。危険はもっぱら、その割り当てがメモリプレッシャーの下で推移的に引き起こしうることにあった。メモリ回収、そこからGPU自身のshrinker、そこからフェンスの待機へと至るが、その間、割り当てを行うスレッド自身が保持しているロックこそが、そのフェンスをシグナルするはずのスレッドが前進するために必要としていたものだった。割り当ての呼び出し箇所を単独で見ても、危険に見えるところは何もない。危険は負荷下で到達可能なコールグラフの中でしか見えない。だからこそドライバ横断の `dma_fence` 契約は、ドライバごとに再発見されるのに任せず、メモリ回収に関わる割り当てを明示的に取り上げているのである。

## 解決 {#resolution}

`4625fe5bbdac`（"drm: gpu: msm: forbid mem reclaim from reset"）は、recover/faultワーカーが `gpu->lock` を保持したままメモリ回収に入ることを禁止し、ロックやshrinkerではなく割り当ての時点で循環を断ち切る。msmのメンテナであるRob Clarkがレビューした。

## 教訓 {#what-it-taught-us}

**割り当てをまたいで保持されたロックは、割り当てを行うコードがフェンスに直接触れなくても、`dma_fence` のデッドロックを生みうる。** `msm_gpu_crashstate_capture()` はどの `dma_fence` APIも呼ばない。デッドロックは、その割り当てが引き起こしうるもの（メモリ回収、そこからドライバ自身のshrinker）から生じた。その間、それが保持していたロックは、フェンスをシグナルすることでその回収のブロックを解けた唯一のスレッドをブロックしていた。

**診断／エラー処理のコードパスこそ、この規則が見落とされる場所である。** それらは何かがすでにおかしくなった*後に*実行されるよう書かれており、そこでは性能より正しさが重要だという前提がある。それは正しいが、回復しようとしている対象のコードのロックと割り当ての制約から免除されるわけではない。

!!! warning "注意すべきパターン"
    ジョブ完了やフェンスシグナルのパスも必要とするロックを保持したまま行う割り当ては、割り当てを行うコードがフェンスと直接の関係を持っていなくても、メモリ回収によって引き起こされるデッドロックの候補である。割り当てが通常のケースで何をするかだけでなく、メモリプレッシャーの下で推移的に何を引き起こしうるか（shrinker、メモリ回収、他ドライバのGEM/TTM shrinker）を追跡すること。

## 関連項目 {#see-also}

- [コマンド投入、`dma_fence`、GPUスケジューラ](../command-submission.md) — このバグが違反した、ドライバ横断の `dma_fence` シグナル契約
- [GEMバッファオブジェクトとdma-bufの共有](../gem-dmabuf.md) — GEM shrinkerと、それが相互作用する `dma_resv` のフェンスリスト
- [Xe TDR回復のリグレッション連鎖](xe-tdr-recovery-regression.md) — 同じTDR/hangcheck系統における、別のドライバの回復パスのバグ

## 外部参考資料 {#external-references}

- [git.kernel.org: 4625fe5bbdac](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4625fe5bbdaccd45be274c30ff0a42e30d4e38cf) — "drm: gpu: msm: forbid mem reclaim from reset"、両方のhung-taskトレースを含む
