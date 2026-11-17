# コマンド投入、fence、GPUスケジューラ {#command-submission-fences-and-the-gpu-scheduler}

> GPU処理のバッチがioctlからハードウェア実行に至るまでの流れ — そして、それが決して戻ってこないときに何が起こるか

## 問題: カーネルはコマンドストリームを読めないが、それでもスケジュールしなければならない {#the-problem-the-kernel-cant-read-the-command-stream-but-it-still-has-to-schedule-it}

[DRMの概要](README.md)は分担を明確にしている。ユーザー空間（Mesa）がハードウェア固有のコマンドバッファを組み立て、カーネルはそれを検査しない。しかし「検査しない」ことは「管理しない」ことと同じではない。ioctlとハードウェアの間には、カーネルにしかできないことが長いリストをなしている。競合する複数のクライアントのうちどれに次にリングを使わせるかを決めること、投入が読み込むバッファへの他者の投入による書き込みが終わる前にその投入が開始しないようにすること、投入が触れるバッファをそれが終わるまで常駐させておくこと、そして投入が何秒もハードウェア上に居座り、決して終わりそうにないことに気づくことである。

この仕組みは3つの部品に分かれ、このページはそれを順に追う。

- **コマンド投入** — ユーザー空間が組み立てたコマンドバッファを、カーネルが追跡する**ジョブ**に変えるドライバ固有のioctl。
- **`struct dma_fence`** — GPUスタックにおけるあらゆる待機が帰着する、一度きりの完了プリミティブ。[GEMとdma-buf](gem-dmabuf.md)では、バッファの `dma_resv` に格納されるものとして紹介した。ここでは、fenceが実際にどうシグナルされるかに踏み込む。
- **`drm_sched`** — `drivers/gpu/drm/scheduler/` にある共有GPUスケジューラ。「ユーザー空間がジョブを投入した」から「ドライバがそれをハードウェアに送った」までの間のキューを所有し、ハードウェアが応答しなくなったときに発火するタイムアウトも含む。

このページのコードと構造体のレイアウトは、特に断りがない限り **Linux 7.1** のものである。スケジューラは活発に再設計が進んでおり、最後のセクションでは、それ以降メインラインですでに変わった点を示す。

## コマンド投入: 設計上ドライバ固有 {#command-submission-driver-specific-by-design}

汎用の「レンダリング投入」ioctlは存在しない。投入の形はハードウェアの性質であるため、各ドライバが独自に定義する。独立したエンジンをいくつ持つか、GPU仮想アドレスを受け取るのかバッファハンドルを受け取るのか、明示的な常駐リストが必要か、などである。LWNの["The Linux graphics stack in a nutshell, part 1"](https://lwn.net/Articles/955376/)（Thomas Zimmermann、2023年12月19日）は、レンダーパスについて端的にこう述べている: *"It's again all specific to the hardware and provided as `ioctl()` operations by each DRM driver individually. As with buffer allocation, the hardware driver within Mesa invokes the DRM driver's `ioctl()` operations."*（訳: これもまたすべてハードウェア固有であり、各DRMドライバが個別に `ioctl()` 操作として提供する。バッファ割り当てと同様に、Mesa内のハードウェアドライバがDRMドライバの `ioctl()` 操作を呼び出す。）

amdgpuのものは `DRM_IOCTL_AMDGPU_CS`（"command submission"）であり、スケジューラが元々そのために書かれたドライバなので、よい代表例である。uAPI（`include/uapi/drm/amdgpu_drm.h`）は意図的に拡張可能になっている。小さな固定ヘッダと、タグ付きの可変長**チャンク**の配列である。

```c
// include/uapi/drm/amdgpu_drm.h
struct drm_amdgpu_cs_in {
	__u32		ctx_id;           /* レンダリングコンテキストID */
	__u32		bo_list_handle;   /* このCSに関連付けられたリソースリスト */
	__u32		num_chunks;
	__u32		flags;
	__u64		chunks;           /* CSチャンクを指す __u64 * を指す */
};

struct drm_amdgpu_cs_chunk {
	__u32		chunk_id;         /* AMDGPU_CHUNK_ID_* */
	__u32		length_dw;
	__u64		chunk_data;
};

struct drm_amdgpu_cs_out {
	__u64 handle;                     /* 後で待機するためのシーケンス番号 */
};
```

実際の意味を運ぶのはチャンクIDである。

```c
#define AMDGPU_CHUNK_ID_IB			0x01
#define AMDGPU_CHUNK_ID_FENCE			0x02
#define AMDGPU_CHUNK_ID_DEPENDENCIES		0x03
#define AMDGPU_CHUNK_ID_SYNCOBJ_IN		0x04
#define AMDGPU_CHUNK_ID_SYNCOBJ_OUT		0x05
#define AMDGPU_CHUNK_ID_BO_HANDLES		0x06
#define AMDGPU_CHUNK_ID_SCHEDULED_DEPENDENCIES	0x07
#define AMDGPU_CHUNK_ID_SYNCOBJ_TIMELINE_WAIT	0x08
#define AMDGPU_CHUNK_ID_SYNCOBJ_TIMELINE_SIGNAL	0x09
#define AMDGPU_CHUNK_ID_CP_GFX_SHADOW		0x0a
```

**IBチャンク**（`struct drm_amdgpu_cs_chunk_ib`）はコマンドバッファそのものだが、実際に何を含むかに注目してほしい。`va_start`（実行を開始するGPU*仮想*アドレス）、`ib_bytes`、そしてどのハードウェアエンジンで実行するかを示す `ip_type`/`ip_instance`/`ring` である。グラフィックスと計算のリングでは、カーネルはそのアドレスにあるパケットを決して読まない。必要なのはバッファが常駐していることとページテーブルが正しいことだけであり、これはまさに[概要](README.md)が説明する分離モデルである。（動画とJPEGのリングは例外である。下の呼び出し順で5番目にある `amdgpu_cs_patch_jobs()` は、そのハードウェアが必要とするため、それらのコマンドストリームを実際に解析したり、その場でパッチしたりする。ツリー内のコメントは今でも "Only for UVD/VCE VM emulation" と書いているが、7.1では3世代のVCN（1.0、3.0、4.0）が `patch_cs_in_place` を設定し、5.0.2を除くすべてのJPEG世代が `parse_cs` を設定している。）**`BO_HANDLES`** は常駐リストである。**`DEPENDENCIES`** は、待機するfenceを `(ctx_id, ip_type, ip_instance, ring, handle)` で指定する。これはamdgpu独自のコンテキストごとのシーケンス番号であり、以前の投入が `cs_out.handle` で受け取ったのと同じ値である。**`SYNCOBJ_*`** チャンクはドライバに依存しない同等物である。`SYNCOBJ_IN` は待機するfence、`SYNCOBJ_OUT` はシグナルするfenceを指定し、`TIMELINE_` 版はタイムライン `drm_syncobj` のための `point` を追加する。

`amdgpu_cs_ioctl()`（`drivers/gpu/drm/amd/amdgpu/amdgpu_cs.c`）は全体を固定のパイプラインとして実行し、関数名はまるで目次のように読める。

```c
	r = amdgpu_cs_parser_init(&parser, adev, filp, data);   /* ctxを探し、guiltyなものは拒否 */
	r = amdgpu_cs_pass1(&parser, data);                     /* チャンクをコピーし、ジョブを割り当てる */
	r = amdgpu_cs_pass2(&parser);                           /* 各チャンクを解釈する */
	r = amdgpu_cs_parser_bos(&parser, data);                /* BOリストをロックして検証する */
	r = amdgpu_cs_patch_jobs(&parser);                      /* IBを解析／パッチする -- 動画／JPEGのみ */
	r = amdgpu_cs_vm_handling(&parser);                     /* ページテーブルの更新 */
	r = amdgpu_cs_sync_rings(&parser);                      /* 依存関係を収集する */
	r = amdgpu_cs_submit(&parser, data);                    /* armしてスケジューラにpushする */
```

2パスに分かれているのは、パス1が何かを割り当てる前に、投入がいくつの異なるエンジンを対象としているかを知る必要があるからである。`amdgpu_cs_job_idx()` は各IBチャンクの `ip_type`/`ip_instance`/`ring` をスケジューラのエンティティに対応付け、既出のエンティティと重複を除き、「ギャング」を拡大する。ギャングは1つのioctlから最大 `AMDGPU_CS_GANG_SIZE` 個のジョブからなり、エンジンごとに1つで、そのうち1つが**ギャングリーダー**に指定される。そのループの後で初めて、パス1は各ジョブについて `amdgpu_job_alloc()` を呼ぶ。

### 「ジョブ」とは何か {#what-a-job-is}

`amdgpu_job_alloc()` は最終的に `drm_sched_job_init()` に行き着く。ここで、ドライバ独自の投入がスケジューラから見えるオブジェクトになる。覚えておく価値のあるジョブの定義は次のとおりである。**ジョブとは、スケジューラがドライバの `run_job()` コールバックに渡す1単位の処理であり、自身の依存fenceのリストを持ち、その完了は他のコードが待機できるfenceで表される。**（非推奨の `drm_sched_resubmit_jobs()` は同じジョブに対して `run_job()` を2回呼ぶことがある — `run_job()` のkerneldocは、そうすることは "violates dma_fence rules"（dma_fenceの規則に違反する）と警告している。後述の[タイムアウトの検出とリカバリ](#timeout-detection-and-recovery)を参照。）ハードウェア固有のものはすべて、ドライバの外側の構造体にとどまる（amdgpuの `struct amdgpu_job` は最初のメンバとして `struct drm_sched_job base` を埋め込んでいる。GEMがバッファオブジェクトで使っているのと同じ埋め込みパターンである）。

## `struct dma_fence` の詳細 {#struct-dma_fence-in-detail}

[GEMとdma-buf](gem-dmabuf.md)では、`dma_fence` を、バッファの `dma_resv` に置かれ、2つのドライバが誰がまだメモリを使っているかについて合意できるようにするものとして紹介した。コマンド投入には、その1つ下の層が必要である。fenceがどう順序付けられ、どうシグナルされ、カーネルがそれについて何を保証するかである。

### コンテキストとシーケンス番号 {#contexts-and-sequence-numbers}

```c
// include/linux/dma-fence.h
struct dma_fence {
	union {
		spinlock_t *extern_lock;
		spinlock_t inline_lock;
	};
	const struct dma_fence_ops __rcu *ops;
	union {
		struct list_head cb_list;   /* dma_fence_signal() で @timestamp に置き換わる */
		ktime_t timestamp;
		struct rcu_head rcu;
	};
	u64 context;
	u64 seqno;
	unsigned long flags;
	struct kref refcount;
	int error;
};
```

`context` と `seqno` の組が順序付けのモデルであり、`dma-fence.c` 自身の概要がその規則を述べている。fenceは *"associated with a context, allocated through `dma_fence_context_alloc()`, and all fences on the same context are fully ordered."*（訳: `dma_fence_context_alloc()` で割り当てられたコンテキストに関連付けられ、同じコンテキスト上のすべてのfenceは全順序付けされている。）コンテキスト内では、より大きい `seqno` は「より後」を意味する — ただし比較自体はドライバの選択である。`__dma_fence_is_later()` はデフォルトではラップアラウンドを考慮した32ビット比較を行い、fenceが `DMA_FENCE_FLAG_SEQNO64_BIT` を設定していれば64ビット全体を比較する。コンテキストをまたぐと順序はまったくない。だからこそ、2つの異なるエンジン間でシーケンス番号を比較しても意味がない。`dma_fence_is_later()` は、コンテキストを共有する2つのfenceに対してのみ有効である。

この性質が、fenceの*重複排除*を安価にしている。`drm_sched_job_add_dependency()` はジョブの既存の依存配列を走査し、同じ `context` のものが見つかれば、2つのうち後のものだけを残す。コードのコメントがその効果を説明している: *"This lets the size of the array of deps scale with the number of engines involved, rather than the number of BOs."*（訳: これにより依存配列のサイズは、BOの数ではなく関与するエンジンの数に比例するようになる。）500個のバッファに触れる投入でも、それらがすべて同じエンジンによって最後に書き込まれていれば、依存関係は500個ではなく1個になる。

### シグナル {#signaling}

`struct dma_fence_ops` は小さく、必須のコールバックは2つだけである（`get_driver_name` と `get_timeline_name`。どちらも情報提供用で、デバッグ出力に加えて `SYNC_IOC_FILE_INFO` でも公開される）。興味深いのは任意のものである。

```c
// include/linux/dma-fence.h
struct dma_fence_ops {
	const char * (*get_driver_name)(struct dma_fence *fence);
	const char * (*get_timeline_name)(struct dma_fence *fence);
	bool (*enable_signaling)(struct dma_fence *fence);
	bool (*signaled)(struct dma_fence *fence);
	signed long (*wait)(struct dma_fence *fence, bool intr, signed long timeout);
	void (*release)(struct dma_fence *fence);
	void (*set_deadline)(struct dma_fence *fence, ktime_t deadline);
};
```

`enable_signaling` が存在するのは、割り込みにはコストがかかるからである。そのkerneldocが意図を説明している。ハードウェア間のシグナルが可能な実装は、*"implement this op to enable the necessary interrupts, or insert commands into cmdstream, etc, to avoid these costly operations for the common case where only hw->hw synchronization is required."*（訳: このopを実装して必要な割り込みを有効化したり、コマンドストリームにコマンドを挿入したりすることで、hw->hwの同期だけが必要なよくあるケースでは、これらのコストの高い操作を避けられる。）これを直接呼ぶものはない。誰かが実際にソフトウェア通知を必要とした最初の時点で、`__dma_fence_enable_signaling()` によって遅延的に呼び出される。それは `dma_fence_add_callback()` から、あるいは `dma_fence_enable_sw_signaling()` 経由で起こり、後者は `dma_fence_wait_timeout()` が待機を始める前に無条件に呼ぶ。コールバックが `false` を返した場合 — fenceがすでに完了しているか、シグナルを有効化できなかったことを意味する — `__dma_fence_enable_signaling()` は、ドライバが設定した `fence->error` のまま、即座にそれをシグナルする。

amdgpuのハードウェアfenceは、このパターンを縮図として示している。`amdgpu_fence_ops` は `enable_signaling` を、フォールバックタイマーを仕掛けるだけのものとして実装している。

```c
// drivers/gpu/drm/amd/amdgpu/amdgpu_fence.c
static bool amdgpu_fence_enable_signaling(struct dma_fence *f)
{
	if (!timer_pending(&to_amdgpu_fence(f)->ring->fence_drv.fallback_timer))
		amdgpu_fence_schedule_fallback(to_amdgpu_fence(f)->ring);

	return true;
}
```

実際のシグナルは、リングの割り込みハンドラから呼ばれる `amdgpu_fence_process()` から来る。これはGPUがメモリに書き込んだ最後に完了したシーケンス番号を読み（`amdgpu_fence_read()`）、以前に把握していた `last_seq` からそこまでのすべてのfenceをシグナルする。フォールバックタイマーは、割り込みの取りこぼしに対する保険である。

待機する側では、`dma_fence_add_callback()` がスケジューラの頼るノンブロッキングのプリミティブであり、身に付けておくべき鋭い落とし穴が1つある。**fenceがすでにシグナル済みの場合、`-ENOENT` を返し、コールバックを呼ばない。** すべての呼び出し側は `-ENOENT` を「すでに完了している、今すぐ行動せよ」として扱わなければならず、`drm_sched_run_job_work()` は `run_job()` がハードウェアfenceを返した後でまさにそうしている。そのkerneldocはさらに、*"the callback can be called from an atomic context or irq context"*（訳: コールバックはアトミックコンテキストやirqコンテキストから呼ばれうる）と警告している。コールバックはスリープしてはならない。

### ドライバ横断の契約 {#the-cross-driver-contract}

あるドライバが作成したfenceを、まったく別のドライバが待機しうるため、`dma-fence.c` はすべての実装を制約する一連の規則を **"fence cross-driver contract"** として文書化している。最初のものが、このページで最も重要なものである。

> *Fences must complete in a reasonable time. Fences which represent kernels and shaders submitted by userspace, which could run forever, must be backed up by timeout and gpu hang recovery code. Minimally that code must prevent further command submission and force complete all in-flight fences... Ideally the driver supports gpu recovery which only affects the offending userspace context, and no other userspace submissions.*

（訳: fenceは妥当な時間内に完了しなければならない。ユーザー空間が投入した、永遠に実行されうるカーネルやシェーダーを表すfenceは、タイムアウトとGPUハングのリカバリコードによって裏打ちされなければならない。最低限、そのコードはそれ以降のコマンド投入を阻止し、実行中のすべてのfenceを強制的に完了させなければならない……理想的には、ドライバは問題を起こしたユーザー空間コンテキストだけに影響し、他のユーザー空間の投入には影響しないGPUリカバリをサポートする。）

残りの規則は、主にそこから導かれるロックの制約である。ドライバは `dma_resv` のロックを保持したまま、シュリンカのコールバックから、そしてMMU notifierから `dma_fence_wait()` を呼んでよい。したがって、`dma_fence_signal()` に至る経路上のコードは、`dma_resv` のロックを取ってはならず、`GFP_KERNEL` で割り当ててはならず、`GFP_NOFS`/`GFP_NOIO` でも割り当ててはならない。`dma_fence_begin_signalling()`/`dma_fence_end_signalling()` は、まさにこれをlockdepに監視させるために存在する。

この規則の裏返しが、dma-bufドキュメントの[**"Indefinite DMA Fences"**](https://docs.kernel.org/driver-api/dma-buf.html)である。これは、完了がユーザー空間の制御下にあるfenceを許したであろう提案（future fence、proxy fence、ユーザー空間fence、長時間実行される計算バッチ）を列挙し、それらが却下される理由を説明している: *"Mixing indefinite fences with normal in-kernel DMA fences does not work, even when a fallback timeout is included to protect against malicious userspace."*（訳: 無期限のfenceを通常のカーネル内DMA fenceと混ぜることはうまくいかない。悪意あるユーザー空間から守るためのフォールバックタイムアウトを含めたとしてもである。）GPUハングが単に待ち切られるのではなく*回復*されなければならない理由がこの契約である — メモリ回収はfenceでブロックすることが許されている。

### デッドラインのヒント {#deadline-hints}

スケジューラのfence opsに現れるため、知っておく価値のある後発の追加機能が1つある。`dma_fence_set_deadline()` は、待機する側がシグナルする側に、fenceを*いつ*完了してほしいかを絶対値の `ktime` で伝えられるようにする。`deadline hints` のドキュメントは動機となったケースを挙げている。ダブルバッファリングのレンダラがvblankを逃すと、*より*アイドルになる（まるまる1vblank期間余分に待つ）ため、利用率ベースのdevfreqガバナはアイドル時間を見てGPUクロックを下げてしまう — まさに逆効果である。これは明示的に助言にすぎない: *"The deadline hint is just that, a hint. The driver that created the fence may react by increasing frequency, making different scheduling choices, etc. Or doing nothing at all."*（訳: デッドラインのヒントはあくまでヒントである。fenceを作成したドライバは、周波数を上げたり、異なるスケジューリングの選択をしたりして反応するかもしれない。あるいは何もしないかもしれない。）

## `drm_sched`: 共有GPUスケジューラ {#drm_sched-the-shared-gpu-scheduler}

### なぜ共有されているか {#why-its-shared}

スケジューラは共通基盤として設計されたわけではない。[`a72ce6f84109`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a72ce6f84109c1dec1ab236d65979d3250668af3)（"drm/amd: add basic scheduling framework"。2015年5月にJammy Zhouが作成し、Shaoyun LiuとChunming ZhouがSigned-off、Linux 4.3向けにマージ）でAMD専用のコードとして登場し、`drivers/gpu/drm/amd/scheduler/` に置かれていた。そのコミットメッセージはすでに、今日まで本質的に変わらず残っているオブジェクトモデルを説明している。*ランキュー*（"a set of entities scheduling commands for the same ring... implements the scheduling policy that selects the next entity to emit commands from..."、訳: 同じリングに対してコマンドをスケジュールするエンティティの集合……次にコマンドを発行するエンティティを選ぶスケジューリングポリシーを実装する……）と*エンティティ*（"a wrapper around a job queue... Entities take turns emitting jobs from their job queue to the corresponding hardware ring"、訳: ジョブキューのラッパー……エンティティは順番に、自身のジョブキューから対応するハードウェアリングへジョブを発行する）である。それを導入した[31パッチのシリーズ](https://lore.kernel.org/dri-devel/1438381367-24980-1-git-send-email-alexander.deucher@amd.com/)はAMD社内で開発され、完成したまとまりとしてdri-develに投稿された — 返信はゼロだった。AMD外部の誰も、出荷前にその設計をレビューしなかった。当時、AMD外部の誰もそれを必要としていなかったからである。

2年半後、Lucas Stachが [`1b1f42d8fde4`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1b1f42d8fde4fef1ed7873bf5aa91755f8c3de35)（"drm: move amd_gpu_scheduler into common location"、2017年12月作成、Linux 4.16向けにマージ）でそれを丸ごと移動した: *"This moves and renames the AMDGPU scheduler to a common location in DRM in order to facilitate re-use by other drivers. This is mostly a straight forward rename with no code changes."*（訳: 他のドライバによる再利用を容易にするため、AMDGPUスケジューラをDRM内の共通の場所に移動し、名前を変更する。これはほぼ単純な名前の変更であり、コードの変更はない。）移動を告知する際、Stachは理由を明言した: *"this is the first step to make the marvelous AMDGPU scheduler useable for other drivers. I have a (mostly) working prototype of Etnaviv using the scheduler."*（訳: これは素晴らしいAMDGPUスケジューラを他のドライバでも使えるようにするための第一歩だ。このスケジューラを使うEtnavivの（ほぼ）動作するプロトタイプがある。）AMDのMonk Liuは、単独の所有権を手放すドライバにとって最も重要な問いを投げかけた — *"I'm wondering if GPU reset still work after this home move"*（訳: この引っ越しの後もGPUリセットは動くのだろうか）— そしてStachの答えが、移動が安全である一行の理由だった: *"this is just a code move that doesn't change anything about the inner workings of the scheduler."*（訳: これは単なるコードの移動であり、スケジューラの内部動作は何も変えない。）AMD以外の最初のユーザーは1サイクル後に続いた — [`e93b6deeb45a`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e93b6deeb45a781489f4ceaa97f9545a3cbebb81)（"drm/etnaviv: hook up DRM GPU scheduler"、Lucas Stach、4.17向けにマージ）であり、そのコミットメッセージは、etnaviv独自のretireワーカーを維持しなくて済むという直接の利点を記している: *"Allows to get rid of the retire worker, as this is now driven by the scheduler."*（訳: これは今やスケジューラによって駆動されるので、retireワーカーを取り除ける。）Etnaviv独自のハングチェックのコードは、その2日後に [`6d7a20c07760`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6d7a20c0776036115c6e22bc673d645d524c4b8a)（"drm/etnaviv: replace hangcheck with scheduler timeout"）で削除された。そのコミットメッセージは、実際の動作上のトレードオフを指摘している: *"we don't allow jobs to run indefinitely as long as they make progress anymore, as this introduces quality of service issues when multiple processes are using the GPU."*（訳: 進捗している限りジョブを無期限に実行させることはもう許さない。複数のプロセスがGPUを使っているときにサービス品質の問題を引き起こすからである。）

これこそが、そもそも共有スケジューラが存在する理由である。リングバッファを持つすべてのドライバが、同じ3つのものを独立に書いていた。ハードウェアキューの手前にある順序付きのソフトウェアキュー、入力が準備できる前にジョブが開始しないようにする依存関係の追跡、そしてハングチェックのタイマーである。Linux 7.1では13のドライバが `DRM_SCHED` をselectしている。`drivers/gpu/drm/` 以下のamdgpu、etnaviv、imagination、lima、msm、nouveau、panfrost、panthor、v3d、xe、そして `drivers/accel/` 以下のamdxdna、ethosu、rocketである。

### 4つのオブジェクト {#the-four-objects}

```
  drm_sched_entity           drm_sched_rq              drm_gpu_scheduler        hardware
  (one per client            (one per priority,        (one per hw ring)          ring
   context+engine)            per scheduler)
  ┌──────────────┐           ┌─────────────┐          ┌──────────────────┐
  │ job_queue    │──push──►  │  entities   │──select► │ work_run_job     │──run_job()──►
  │ (spsc queue) │           │  (rr / fifo)│          │ pending_list     │
  │ dependency   │           └─────────────┘          │ work_tdr         │
  └──────────────┘                                    └──────────────────┘
```

**`struct drm_gpu_scheduler`** はハードウェアリングごとに1つある（`sched_main.c` の概要: *"Each hw run queue has one scheduler"*、訳: 各hwランキューは1つのスケジューラを持つ）。ランキューの配列、workqueue、現在ハードウェア上にあるジョブのリスト、そしてタイムアウトを所有する。

```c
// include/drm/gpu_scheduler.h
struct drm_gpu_scheduler {
	const struct drm_sched_backend_ops	*ops;
	u32				credit_limit;
	atomic_t			credit_count;
	long				timeout;
	const char			*name;
	u32				num_rqs;
	struct drm_sched_rq		**sched_rq;
	...
	struct workqueue_struct		*submit_wq;
	struct workqueue_struct		*timeout_wq;
	struct work_struct		work_run_job;
	struct work_struct		work_free_job;
	struct delayed_work		work_tdr;
	struct list_head		pending_list;   /* ハードウェアに渡されたジョブ */
	spinlock_t			job_list_lock;
	int				hang_limit;
	atomic_t			*score;
	...
};
```

**`struct drm_sched_rq`** は優先度レベル（`DRM_SCHED_PRIORITY_KERNEL`、`HIGH`、`NORMAL`、`LOW`）ごとに1つのランキューであり、処理が保留中のエンティティの集合を保持する。**`struct drm_sched_entity`** はクライアントごとのキューであり、そのkerneldoc自身が *"A wrapper around a job queue (typically attached to the DRM file_priv)"*（訳: ジョブキューのラッパー（通常はDRMのfile_privに付随する））と呼んでいる。amdgpuでは、`amdgpu_ctx_init_entity()` がコンテキストごとに `(ハードウェアIPの種類, リング)` の組ごとに1つのエンティティを作成する — それらは `ctx->entities[hw_ip][ring]` に置かれる — ので、2つのGPUコンテキストを持ち、グラフィックスと計算の両方を使うプロセスは、少なくとも4つの独立したエンティティを持つことになる。エンティティは、ジョブの単一プロデューサ／単一コンシューマのキュー（`struct spsc_queue job_queue`）と、全体を結び付けるfenceの識別情報を保持する。

```c
// include/drm/gpu_scheduler.h — struct drm_sched_entity の主なフィールド
	struct spsc_queue		job_queue;
	atomic_t			fence_seq;
	uint64_t			fence_context;
	struct dma_fence		*dependency;   /* 先頭のジョブが何でブロックされているか */
	struct dma_fence_cb		cb;
	atomic_t			*guilty;
	struct dma_fence __rcu		*last_scheduled;
```

`drm_sched_entity_init()` は `dma_fence_context_alloc(2)` を呼ぶ — エンティティごとに1つではなく**2つ**のコンテキストである。すべてのジョブは2つのfenceを生成し、それらは独立に順序付けられなければならないからである。

**`struct drm_sched_job`** は、ドライバが埋め込む単位である。

```c
struct drm_sched_job {
	ktime_t				submit_ts;
	struct drm_gpu_scheduler	*sched;    /* drm_sched_job_arm() で設定される */
	struct drm_sched_fence		*s_fence;
	struct drm_sched_entity		*entity;
	enum drm_sched_priority		s_priority;
	u32				credits;
	unsigned int			last_dependency;
	atomic_t			karma;     /* このジョブのせいにされたハングの数 */
	struct spsc_node		queue_node;
	struct list_head		list;      /* スケジューラの pending_list 上 */
	union {
		struct dma_fence_cb	finish_cb;
		struct work_struct	work;
	};
	struct dma_fence_cb		cb;
	struct xarray			dependencies;
};
```

### ジョブごとに2つのfence、そしてその理由 {#two-fences-per-job-and-why}

`struct drm_sched_fence` はパイプライン全体を機能させる部品であり、*2つ*の `dma_fence` と、3つ目へのポインタを含む。

```c
// include/drm/gpu_scheduler.h
struct drm_sched_fence {
	struct dma_fence		scheduled;  /* ジョブがハードウェアにpushされた */
	struct dma_fence		finished;   /* ジョブが完了した */
	ktime_t				deadline;
	struct dma_fence		*parent;    /* run_job() からのハードウェアfence */
	struct drm_gpu_scheduler	*sched;
	spinlock_t			lock;
	void				*owner;
	uint64_t			drm_client_id;
};
```

`drm_sched_fence_init()` はそれらに隣接するコンテキストを与える。`scheduled` は `entity->fence_context` を、`finished` は `entity->fence_context + 1` を得て、どちらも同じ `seqno` を持つ。

2つある理由は、`finished` fenceのkerneldocが直接述べている鶏と卵の問題である: *"When setting up an out fence for the job, you should use this, since it's available immediately upon `drm_sched_job_init()`, and the fence returned by the driver from `run_job()` won't be created until the dependencies have resolved."*（訳: ジョブの出力fenceを設定するときはこれを使うべきである。これは `drm_sched_job_init()` の直後から利用可能だが、ドライバが `run_job()` から返すfenceは依存関係が解決するまで作成されないからである。）投入には、ユーザー空間に返し、触れるすべてのバッファの `dma_resv` に入れるためのfenceが*ioctlの時点で*必要である。しかしハードウェアfenceは、ジョブが実際にハードウェアに到達するまで存在しない。それは数秒後かもしれない。そこで `finished` が代役となる。スケジューラは、`parent`（本物のハードウェアfence）がシグナルされたときにそれをシグナルし、その際にエラーコードを引き継ぐ。

`scheduled` fenceがもたらすものは別で、パイプライン化である。あるジョブが*同じスケジューラ上の*別のジョブに依存する場合、`drm_sched_entity_add_dependency_cb()` は、依存先の `finished` fenceを、ひそかにその `scheduled` fenceに置き換える。

```c
	s_fence = to_drm_sched_fence(fence);
	if (!fence->error && s_fence && s_fence->sched == sched &&
	    !test_bit(DRM_SCHED_FENCE_DONT_PIPELINE, &fence->flags)) {
		/*
		 * fenceは同じスケジューラからのものなので、
		 * スケジュールされるのを待つだけでよい
		 */
		fence = dma_fence_get(&s_fence->scheduled);
		dma_fence_put(entity->dependency);
		entity->dependency = fence;
	}
```

両方のジョブが同じリングに行き、リングは順番に実行するので、依存する側のジョブは、先行ジョブがpushされた時点ですぐにハードウェアにpushできる — 完了を待つCPUの往復は不要である。`DRM_SCHED_FENCE_DONT_PIPELINE`（`dma_fence` のユーザーフラグ）は、それが安全でないケースのためのオプトアウトである。同じ関数は、エンティティ*自身の*fenceへの依存も完全に省略する — エンティティのジョブは、そのSPSCキューによってすでに順序付けられているからである。

### パイプライン {#the-pipeline}

**投入側**、必須の順序で:

1. **`drm_sched_job_init(job, entity, credits, owner, drm_client_id)`** — `drm_sched_fence` を割り当て、依存関係のxarrayを初期化する。この時点ではまだスケジューラを選ば*ない*。
2. 依存関係が付けられる。生のfenceには `drm_sched_job_add_dependency()`、`drm_syncobj` ハンドルとタイムラインポイントには `drm_sched_job_add_syncobj_dependency()`、指定した `dma_resv_usage` で `dma_resv` を走査するには `drm_sched_job_add_resv_dependencies()`、あるいは `drm_sched_job_add_implicit_dependencies(job, obj, write)` を使う。最後のものは `dma_resv_usage_rw(write)` を使う `dma_resv` 版にすぎず、つまり[GEM/dma-buf](gem-dmabuf.md#fences-and-dma_resv)の暗黙的同期の規則をまさにジョブの依存関係として表現したものである。
3. **`drm_sched_job_arm(job)`** — スケジューラを選び（`drm_sched_entity_select_rq()`）、`job->sched` と `job->s_priority` を記録し、2つのfenceを本物のシーケンス番号で初期化する。`drm_sched_job_cleanup()` のkerneldocはこれを *"a point of no return since it initializes the fences and their sequence number"*（訳: fenceとそのシーケンス番号を初期化するので、後戻りできない地点）と呼んでいる。armされたら、ジョブは**必ず**pushされなければならない。他のコードがすでにその `finished` fenceを保持しているかもしれないからである。
4. **`drm_sched_entity_push_job(job)`** — ジョブにタイムスタンプを付け、エンティティのSPSCキューにpushし、それが最初のジョブであれば、エンティティをそのランキューに追加して `drm_sched_wakeup()` を呼ぶ。

amdgpuの `amdgpu_cs_submit()` は、ギャングという一ひねりを加えてこれらすべてを示している。ギャング内のすべてのジョブをarmし、ギャングリーダーを他のすべてのメンバーの **`scheduled`** fenceに依存させ、リーダーの `finished` fenceをロックされたすべてのバッファの `dma_resv` に公開し（リーダーには `DMA_RESV_USAGE_WRITE`、残りには `DMA_RESV_USAGE_READ`）、その後で初めてそれらすべてをpushする。

**実行側** — `submit_wq` 上のワークアイテムである `drm_sched_run_job_work()`:

1. **`drm_sched_select_entity()`** は `DRM_SCHED_PRIORITY_KERNEL` から下に向かってランキューを走査し、最初に準備ができたエンティティを返す。重要なのは、あるランキューに準備のできたエンティティがあるのにスケジューラのクレジットが尽きている場合、ランキューごとの選択ヘルパーは `NULL` ではなく `ERR_PTR(-ENOSPC)` を返し、優先度のループは停止し、`drm_sched_select_entity()` は `NULL` を返すことである — これにより、忙しいリングで低優先度の処理がブロックされた高優先度の処理を追い越すことがなくなる。
2. **`drm_sched_entity_pop_job()`** は依存関係を解決する。`drm_sched_job_dependency()` をループで呼び、これはジョブのxarrayから次のfenceを返す（すでにシグナル済みのものは停滞させない。コールバックの登録が `-ENOENT` を返し、ループは次の依存関係に進む）。xarrayを使い切ると、ドライバの任意の `prepare_job()` コールバックを呼び、これはさらに別のfenceを返すことがある — amdgpuはこれをVMIDの取得に使っている。それぞれについて、`drm_sched_entity_add_dependency_cb()` が `drm_sched_entity_wakeup` を登録し、ワーカーは戻る。エンティティは、そのfenceがシグナルされたときに再検査される。
3. ジョブは `sched->pending_list` に追加され、`drm_sched_job_begin()` によってタイムアウトが（再）設定される。
4. **`sched->ops->run_job(job)`** — ドライバが実際にリングに書き込む。これは**ハードウェアfence**を返し、スケジューラは呼び出し側の参照を*継承*する: *"the scheduler expects to 'inherit' its own reference to this fence from the callback. It does not invoke an extra `dma_fence_get()` on it."*（訳: スケジューラは、このfenceへの自身の参照をコールバックから「継承」することを期待している。追加の `dma_fence_get()` は呼ばない。）
5. `drm_sched_fence_scheduled()` がハードウェアfenceを `s_fence->parent` として格納し、`scheduled` をシグナルする。ハードウェアfenceに対する `dma_fence_add_callback()` が `drm_sched_job_done_cb()` を仕掛ける。`-ENOENT` の場合（すでにシグナル済み）は、`drm_sched_job_done()` を直接呼ぶことで処理される。
6. ハードウェアfenceがシグナルされると、`drm_sched_job_done()` がジョブのクレジットを返却し、ハードウェアfenceのエラーコードとともに `finished` をシグナルし、`work_free_job` をキューに入れる。これが最終的に `sched->ops->free_job()` を呼ぶ。

### フロー制御: クレジット {#flow-control-credits}

`drm_sched` はジョブを数えるのではなく、**クレジット**を数える。ジョブごとにドライバが定義する量を、スケジューラごとの `credit_limit` と照らし合わせる。`sched_main.c` は、これがキューではなく上限である理由を文書化している: *"If by executing one more job the scheduler's credit count would exceed the scheduler's credit limit, the job won't be executed."*（訳: もう1つジョブを実行するとスケジューラのクレジット数がその上限を超える場合、そのジョブは実行されない。）要点はリングバッファの容量である。Matthew Brostのworkqueue移行コミットは、Xeが頼っている工夫を説明している: *"in Xe submissions are done via programming a ring buffer (circular buffer), a drm_gpu_scheduler provides a limit on number of jobs, if the limit of number jobs is set to RING_SIZE / MAX_SIZE_PER_JOB we get flow control on the ring for free."*（訳: Xeでは投入はリングバッファ（循環バッファ）をプログラムすることで行われ、drm_gpu_schedulerはジョブ数の上限を提供するので、ジョブ数の上限をRING_SIZE / MAX_SIZE_PER_JOBに設定すれば、リングのフロー制御がただで手に入る。）amdgpuは単純にしている — `credit_limit = ring->num_hw_submission` で、すべてのジョブのコストは1クレジットである。`drm_sched_can_queue()` は、クレジットが上限全体を超えるジョブを（`dev_WARN` とともに）上限まで切り詰める。これは、不可能なジョブでデッドロックするのではなく、前進を保証するためである。

### スケジューリングポリシー、3つの時代 {#scheduling-policy-three-eras}

ランキュー上のエンティティから選択するポリシーは2回書き直されており、どの書き直しも、測定された飢餓の問題によって推進された。

- **ラウンドロビン**（2015〜2022年）— `drm_sched_rq_select_entity_rr()` は、前回止まったところからエンティティリストを走査する。Luben Tuikovによる、そこから移行するためのコミットメッセージは、その障害を図で示している。キューの深さが大きく異なるエンティティがある場合、厳密なラウンドロビンは1回の走査でエンティティごとに1ジョブずつ処理するため、深いキューの最も古いジョブが、浅いキューからの多数の新しいジョブの後ろに置かれ続けることがある。
- **FIFO**（Linux 6.2以降）— [`08fb97de03aa`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=08fb97de03aa2205c6791301bd83a095abc1949c)（Andrey Grodzovsky、"drm/sched: Add FIFO sched policy to run queue"）が `drm_sched_rq_select_entity_fifo()` を追加した。これはエンティティを `oldest_job_waiting` をキーとする `rb_root_cached` に保持し、常に先頭のジョブが最初に到着したエンティティを選ぶ — 選択はO(1)、更新はO(log N)である。[`977d97f18b5b`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=977d97f18b5b8efb7a94da84724113f15ae6cc2d)（Luben Tuikov）が同じリリースでそれをデフォルトにした。7.1では、スケジューラの `sched_policy` モジュールパラメータで両方のポリシーを選択できる。
- **フェア**（Linux 7.2向けにマージ済み、執筆時点では未リリース）— Tvrtko Ursulinの [`2fa4d8e2c109`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2fa4d8e2c1091189064c5b93222a23ded2d881ba)（"drm/sched: Add fair scheduling policy"）は、両方をCFS風の仮想実行時間による順序付けで置き換える: *"entity run queue is sorted by the virtual GPU time consumed by entities in a way that the entity with least vruntime runs first... It is able to avoid total priority starvation, which is one of the problems with FIFO, and it also does not need for per priority run queues."*（訳: エンティティのランキューは、エンティティが消費した仮想GPU時間でソートされ、vruntimeが最も少ないエンティティが最初に実行される……FIFOの問題の1つである優先度による完全な飢餓を避けることができ、優先度ごとのランキューも必要としない。）[`45c211ddf92a`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=45c211ddf92a1f9b4214ffadaf70d9037f53aaf6) がそれをデフォルトにし、続いて [`77a6809f1dc3`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=77a6809f1dc39376116f8d769a0d2630dc95ad79) がFIFO、ラウンドロビン、そして優先度ごとのランキュー配列全体を削除し、`drm_gpu_scheduler.sched_rq[]` を埋め込みの単一の `struct drm_sched_rq rq` にまとめた。7.1ではなくメインラインを読んでいるなら、これが上で示したものとの最大の構造的な違いである。

古いコードを読むときに知っておく価値のある構造変更がもう1つある。Linux 6.8までは、各スケジューラは専用の **kthread** を実行していた。[`a6149f039369`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a6149f0393699308fb00149be913044977bceb56)（Matthew Brost、"drm/sched: Convert drm scheduler to use a work queue rather than kthread"）が、それを上で示した `work_run_job`/`work_free_job` ワークアイテムに置き換えた。動機はXeから来ていた。Xeは1つのスケジューラを1つのエンティティに対応付ける。そのファームウェアスケジューラ（GuC）が投入を並べ替えたりプリエンプトしたりしうるため、それが必要だったのである — *"if a using shared drm_gpu_scheduler across multiple drm_sched_entity, the TDR falls apart as the TDR expects submission order == completion order"*（訳: 共有のdrm_gpu_schedulerを複数のdrm_sched_entityにまたがって使うと、TDRは投入順序 == 完了順序を前提としているため破綻する）— そして、エンティティごとに1つのkthreadはスケールしない。

## タイムアウトの検出とリカバリ {#timeout-detection-and-recovery}

### タイマー {#the-timer}

`sched->work_tdr` は `drm_sched_job_timedout()` を実行する `delayed_work` であり、`pending_list` が空でなく `sched->timeout != MAX_SCHEDULE_TIMEOUT` であるときは常に、`drm_sched_start_timeout()` によって設定される。ジョブが完了するたびに再設定されるので、タイムアウトは投入バッチごとではなく、先頭にあるジョブごとである。2つの関数がこれを短絡させる。`drm_sched_fault()` は、ドライバの割り込みハンドラがハードウェア障害を検出したときに即座にそれを発火させ、`drm_sched_tdr_queue_imm()` は `sched->timeout = 0` を設定して再設定する。

ハンドラ自体は短い。

```c
// drivers/gpu/drm/scheduler/sched_main.c
static void drm_sched_job_timedout(struct work_struct *work)
{
	...
	spin_lock(&sched->job_list_lock);
	job = list_first_entry_or_null(&sched->pending_list,
				       struct drm_sched_job, list);

	if (job) {
		/*
		 * 並行する &struct drm_sched_backend_ops.free_job によって
		 * 解放されないよう、問題のジョブを取り除く。スケジューラの
		 * ワークアイテムがキャンセルされた後、安全になった時点で
		 * 再挿入される。
		 */
		list_del_init(&job->list);
		spin_unlock(&sched->job_list_lock);

		status = job->sched->ops->timedout_job(job);
		...
		if (status == DRM_GPU_SCHED_STAT_NO_HANG)
			drm_sched_job_reinsert_on_false_timeout(sched, job);
	} else {
		spin_unlock(&sched->job_list_lock);
	}

	if (status != DRM_GPU_SCHED_STAT_ENODEV)
		drm_sched_start_timeout_unlocked(sched);
}
```

これが*しない*ことに注目してほしい。汎用のリセットはない。コアの貢献は「`pending_list` の先頭が長く居座りすぎている — ほら、これだ、あとは任せた」ということだけである。リカバリは `timedout_job()` であり、ステータスを返すドライバのコールバックである。

```c
enum drm_gpu_sched_stat {
	DRM_GPU_SCHED_STAT_NONE,     /* 予約済み、使用しないこと */
	DRM_GPU_SCHED_STAT_RESET,    /* GPUがハングし、リセットに成功した */
	DRM_GPU_SCHED_STAT_ENODEV,   /* デバイスがなくなった */
	DRM_GPU_SCHED_STAT_NO_HANG,  /* GPUはハングしておらず、まだ動作している */
};
```

`DRM_GPU_SCHED_STAT_NO_HANG` はこれらの中で最も新しく、実際のリークを修正するためにMaíra Canalが [`0b1217bfdfdd`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0b1217bfdfddf664c15954d1d51ee18ed88a2ccf)（"drm/sched: Allow drivers to skip the reset and keep on running"、Linux 6.17）で追加した。ハンドラは `timedout_job()` を呼ぶ*前に*ジョブを `pending_list` から取り除くので、「実はGPUは大丈夫で、このジョブが遅いだけだ」と判断したドライバには、それを戻す手段がなかった — そのためジョブが実際に完了しても、それに対して `free_job()` が呼ばれることはなかった。このコミットは、カバーする2つのケースを挙げている。ハードウェア固有の仕組みでGPUの前進を観測するドライバ（*"This happens in v3d, Etnaviv, and Xe"*、訳: これはv3d、Etnaviv、Xeで起こる）と、*"Timeout has fired before the free-job worker."*（訳: free-jobワーカーより先にタイムアウトが発火した）という単純な競合である。

### 全員を巻き添えにせずにリセットする {#resetting-without-killing-everyone}

`gpu_scheduler.h` にある `timedout_job` のkerneldocは推奨される手順を詳述しており、興味深いのは、そのどれだけが巻き添え被害を*避ける*ためにあるかである。1つの `drm_gpu_scheduler` が1つのリング上で多数のエンティティに対応するハードウェアスケジューラの場合、スケジューラを破棄することは許容できない。*"this would effectively also affect innocent userspace processes which did not submit faulty jobs."*（訳: それは事実上、問題のあるジョブを投入していない無実のユーザー空間プロセスにも影響を与えてしまう。）規定された順序は次のとおりである。影響を受けるすべてのスケジューラを `drm_sched_stop()` する → 問題のジョブが来たエンティティを殺す → ドライバ固有のリングリセットを発行する → 生き残ったものを再投入する → `drm_sched_start()`。

`drm_sched_stop(sched, bad)` は繊細な管理作業を行う。ワークアイテムを一時停止し、次に `pending_list` を*逆向きに*走査し、各ジョブについて、ハードウェアfenceのコールバックを取り除く（ジョブは完了していない。`s_fence->parent` を落とし、クレジットを返却して、再投入できるようにする）か、コールバックがすでに実行中であれば、`finished` fenceを待ち切ってジョブを解放する。guiltyなジョブは意図的に生かしておかれる — タイムアウトハンドラが後でそれを解放するように、ヒントとして `sched->free_guilty` が設定される。その後 `drm_sched_start(sched, errno)` が生き残ったものにコールバックを再び付け、親fenceがなくなったものはすべて `errno ?: -ECANCELED` で強制完了させる — これが、待機が永遠にハングするのではなく、エラーがユーザー空間に届く仕組みである。

責任は**karma**で追跡される。`drm_sched_increase_karma(bad)` はジョブの `karma` をインクリメントし、所有するエンティティの `guilty` フラグを設定する。すると `drm_sched_entity_pop_job()` は、そのエンティティからの後続のすべてのジョブを、その `finished` fenceに `-ECANCELED` を設定することで汚染する。コアはそれでもそのジョブに対して `run_job()` を呼ぶ。チェックするのはドライバである — amdgpuの `amdgpu_job_run()` は、`finished->error < 0` のとき "Skip scheduling IBs in ring(%s)" とログを出し、ハードウェアfenceを返さない。カーネルのジョブは設計上除外される — コメントは、GPUハングはカーネルのVM更新ジョブも壊しうると記しつつ、*"but keep in mind that kernel jobs always considered good."*（訳: ただし、カーネルのジョブは常に正常とみなされることに留意せよ）としている。（関連する `hang_limit`/`drm_sched_invalidate_job()` のしきい値の仕組みは、現在 `drm_sched_init_args` でDEPRECATEDとマークされており、ドライバにはそれを0に設定するよう指示している。）

`drm_sched_resubmit_jobs()` には異例なほど率直な非推奨の注記があり、コミュニティがここで何を学んだかのよい要約になっている。

> *Re-submitting jobs was a concept AMD came up as cheap way to implement recovery after a job timeout. This turned out to be not working very well. First of all there are many problem with the dma_fence implementation and requirements. Either the implementation is risking deadlocks with core memory management or violating documented implementation details of the dma_fence object.*

（訳: ジョブの再投入は、ジョブのタイムアウト後のリカバリを安価に実装する方法としてAMDが考え出した概念である。これはあまりうまく機能しないことが判明した。まず第一に、dma_fenceの実装と要件に多くの問題がある。実装は、コアのメモリ管理とのデッドロックの危険を冒すか、dma_fenceオブジェクトの文書化された実装上の詳細に違反するかのどちらかである。）

他のドライバがすでに参照を保持しているかもしれない `dma_fence` を再初期化することは、一度きりの契約を破る。そしてリセット中にメモリを割り当てると、シグナルしようとしているまさにそのfenceを待つメモリ回収とデッドロックしうる。推奨される代替策は、ドライバがスケジューラを停止した後で `drm_sched_for_each_pending_job()` を反復し、独自のリカバリを行うことである。

### amdgpuが実際にすること {#what-amdgpu-actually-does}

`amdgpu_job_timedout()` は「エスカレーションはできるだけ少なく」のよい例である。順に:

1. デバイスが取り外されていれば（`drm_dev_enter()` が失敗する）、即座に `DRM_GPU_SCHED_STAT_ENODEV` を返す。
2. リカバリが状態をかき乱す前にそれを捕捉するため、すぐにdevcoredumpを取る（`amdgpu_job_core_dump()`）。
3. **ソフトリカバリ**を試みる — `amdgpu_ring_soft_recovery()` はタイムアウトしたfenceを `-ENODATA` とマークし、fenceがシグナルされることを期待して、ASICごとの `ring->funcs->soft_recovery(ring, vmid)` コールバックで最大10msスピンする。何もリセットされない。成功すれば、`"ring %s timeout, but soft recovered"` とログを出して終了する。
4. **キューごとのリングリセット**を試みる — `drm_sched_wqueue_stop()`、`amdgpu_ring_reset()`、`drm_sched_wqueue_start()`。成功すれば、ユーザー空間がcoredumpを回収できるように `drm_dev_wedged_event(..., DRM_WEDGE_RECOVERY_NONE, info)` を発行する。
5. それ以外の場合は `dma_fence_set_error(&s_job->s_fence->finished, -ETIME)` を行い、デバイス全体のリセットである `amdgpu_device_gpu_recover()` へと進む。

デバイスが取り外された場合を除くすべてのパスは、関数末尾の単一の `return DRM_GPU_SCHED_STAT_NO_HANG` に行き着き、そこには `/* This is needed to add the job back to the pending list */` という注釈がある。amdgpuは「回復した」と「そもそもハングしていなかった」を区別するのではなく、`NO_HANG` の再挿入パスに無条件に頼り、コアにジョブを戻させて、それに対して `free_job()` が依然として実行されるようにしている。

デフォルトのタイムアウトは、`amdgpu_device_init_schedulers()` で `adev->gfx_timeout` / `compute_timeout` / `sdma_timeout` / `video_timeout` からエンジンの種類ごとに設定される。`amdgpu_device_get_job_timeout_settings()` は4つすべてを2秒に初期化し、`amdgpu.lockup_timeout=` パラメータで上書きできる（負の値はタイムアウトを無効化し、カーネルを `TAINT_SOFTLOCKUP` でtaintする）。注目すべきことに、amdgpuは `timeout_wq` として `adev->reset_domain->wq` を渡している — `timedout_job` のkerneldocは、別個のハードウェアキューを持つがグローバルにリセットしなければならないGPUに対して、異なるスケジューラのタイムアウトハンドラが1つずつ実行されるよう、順序付きworkqueueを明示的に推奨している。

### ユーザー空間に伝える {#telling-userspace}

リカバリは仕事の半分にすぎない。コンテキストを殺されたクライアントはそれを知る必要がある。[カーネルドキュメント: DRM uAPI](https://docs.kernel.org/gpu/drm-uapi.html) はこれに "Device reset" のセクションを割いており、その位置付けは、エラーの伝播は *"goes in the opposite direction of the usual flow of commands"*（訳: 通常のコマンドの流れとは逆方向に進む）というものである — そのため、ベンダーに依存しない仕組みは、fenceをシグナルする前にそれに対して `dma_fence_set_error()` を行うことであり、スケジューラはエラーを *"from the hardware fence to the scheduler fence to bubble up errors to the higher levels of the stack and eventually userspace."*（訳: ハードウェアfenceからスケジューラfenceへと転送し、エラーをスタックの上位レベル、そして最終的にユーザー空間へと浮かび上がらせる）。ドライバは、リセットをコンテキストごとに追跡し（`drm_sched_entity_error()` が提供されているヘルパーである）、**影響を受けたコンテキストからの新しい投入を拒否する**よう指示されている — これはまさに、`guilty` フラグが設定されたコンテキストに対して `amdgpu_cs_parser_init()` が `-ECANCELED` を返すときに行っていることである。ユーザー空間はそれを `GL_ARB_robustness` のリセットステータスや、Vulkanの `VK_ERROR_DEVICE_LOST` として表面化させる。同じドキュメントは、これはアプリケーションがそもそもrobustnessをオプトインしている場合にのみ機能し、そうでない場合には *"there is no strong community consensus on what the userspace driver should do"*（訳: ユーザー空間ドライバが何をすべきかについて、コミュニティの強い合意はない）と率直に認めている。

## 実例: 1回の `AMDGPU_CS` 呼び出しを端から端まで {#worked-example-one-amdgpu_cs-call-end-to-end}

Vulkanアプリケーションが、別のキューが書き込んだばかりのテクスチャをサンプリングするレンダーパスを投入し、その結果を表示する。

**1. ユーザー空間がリクエストを組み立てる。** Mesaのradvは、すでに所有しているGPUから見えるバッファにコマンドパケットを書き込み、次に `union drm_amdgpu_cs` に `ctx_id`、投入が触れるすべてのバッファを示す `bo_list_handle`、そしてチャンクを埋める。チャンクは、`AMDGPU_CHUNK_ID_IB` が1つ（それらのパケットの `va_start`、`ip_type = AMDGPU_HW_IP_GFX`）、もう一方のキューがシグナルするsyncobjを示す `AMDGPU_CHUNK_ID_SYNCOBJ_IN` が1つ、コンポジタが待機するfenceのための `AMDGPU_CHUNK_ID_SYNCOBJ_OUT` が1つである。

**2. パス1がジョブを割り当てる。** `amdgpu_cs_pass1()` はチャンク配列をコピーし、IBチャンクについて `amdgpu_cs_job_idx()` → `amdgpu_ctx_get_entity()` を呼び、`(GFX, instance 0, ring 0)` を、以前に `amdgpu_ctx_init_entity()` が作成した `drm_sched_entity` に解決する。エンジンは1つなので `gang_size == 1` である。`amdgpu_job_alloc()` → `drm_sched_job_init()` が `drm_sched_fence` を作成する — この瞬間、`finished` fenceは存在し、シグナルされておらず、最終的にどのハードウェアリングがそれを実行するかも知らない。

**3. パス2がチャンクを依存関係に変える。** `SYNCOBJ_IN` チャンクは `amdgpu_syncobj_lookup_and_add()` を通り、これがハンドルを `dma_fence` に解決して投入の同期セットに追加する。`amdgpu_cs_parser_bos()` はBOリストのすべてのバッファを `drm_exec`（`ww_acquire_ctx` のラッパーであり、ロック順序の衝突はバックオフして再試行する）でロックし、その配置を検証する。続いて `amdgpu_cs_sync_rings()` が*暗黙的*な半分を行う。ロックされたすべてのオブジェクトの `dma_resv` を `amdgpu_sync_resv()` で走査し、その結果を `amdgpu_sync_push_to_job()` で各ジョブにpushする。これは最終的に `drm_sched_job_add_dependency()` に行き着く。明示的な依存関係と暗黙的な依存関係は、同じxarrayに入る。

**4. armとpush。** `amdgpu_cs_submit()` が `drm_sched_job_arm()` を呼ぶ — *ここで* `drm_sched_entity_select_rq()` がスケジューラを選び（エンティティに複数が与えられていれば、`drm_sched_pick_best()` が `score` で最も負荷の低いリングを選ぶ）、2つのfenceが本物のシーケンス番号を得る。`finished` fenceは、ロックされたすべてのバッファの `dma_resv` に `DMA_RESV_USAGE_WRITE` で追加される。これにより、*次の*投入の暗黙的同期の走査 — あるいは[GEM/dma-buf](gem-dmabuf.md#worked-example-a-camera-frame-becomes-a-gpu-texture-zero-copy)にあるように、ディスプレイドライバの `prepare_fb` — が、このレンダリングを自動的に待つようになる。`amdgpu_ctx_add_fence()` は、それを `cs->out.handle` で返されるシーケンス番号のもとに記録する。次に `drm_sched_entity_push_job()` を行い、ioctlは戻る。まだ何もリングに触れていない。

**5. スケジューラがそれを実行する。** `drm_sched_run_job_work()` がエンティティを選び、`drm_sched_entity_pop_job()` はテクスチャ書き込みの依存関係がシグナルされていないことを見つける — そこでコールバックを登録し、何も実行せずに戻る。もう一方のキューのジョブが完了してシグナルすると、`drm_sched_entity_wakeup()` がワークアイテムを再びキューに入れる。今度は依存関係を使い切っているので、`prepare_job()`（amdgpu: `amdgpu_vmid_grab()`）がVMIDのためのfenceをもう1つ返すかもしれない。それが解消されると、ジョブは `pending_list` に入り、2秒のGFXタイムアウトが設定され、`amdgpu_job_run()` がリングに書き込む。これは `amdgpu_fence` を返す。スケジューラはそれを `s_fence->parent` として格納し、`scheduled` をシグナルし、それに `drm_sched_job_done_cb()` をぶら下げる。

**6a. 通常の完了。** GPUはシーケンス番号をメモリに書き込み、リング割り込みを上げる。`amdgpu_fence_process()` はそのseqnoまでのすべてのfenceをシグナルする。`drm_sched_job_done()` はクレジットを返却し、`finished` をシグナルし（エラーコードはハードウェアfenceから引き継がれる）、`work_free_job` をキューに入れる。`SYNCOBJ_OUT` のfence — これは `finished` fenceである — がシグナルされ、コンポジタの待機が完了し、フレームは[KMS](kms.md)のアトミックコミットに回される。

**6b. 代わりにシェーダーが永遠にループする。** 割り込みは来ない。2秒後、`drm_sched_job_timedout()` がジョブを `pending_list` から取り出し、`amdgpu_job_timedout()` を呼ぶ。coredumpが取られ、ソフトリカバリが10ms試みられる。fenceがまだシグナルされていなければ、キューごとのリングリセットが試みられる。それも失敗すれば、`finished` に `-ETIME` が設定され、デバイス全体がリセットされる。その完全リセットのパスでは、`drm_sched_increase_karma()` がエンティティをguiltyとマークするので、このジョブの後ろにまだキューに入っているすべてのジョブは、実行されるのではなく `-ECANCELED` で完了させられる。アプリケーションの次の `AMDGPU_CS` は `amdgpu_cs_parser_init()` で `-ECANCELED` で拒否され、radvはそれを `VK_ERROR_DEVICE_LOST` に変える。そして — システムの残りにとって決定的に重要なことに — そのsyncobjに対するコンポジタの待機は、永遠にハングするのではなくエラーで完了する。ドライバ横断のfence契約が、それ以外の何も許さないからである。

## 参考文献 {#further-reading}

- [カーネルドキュメント: DRM Memory Management](https://docs.kernel.org/gpu/drm-mm.html) — "GPU Scheduler" の章（概要、フロー制御、生成された `drm_sched` APIリファレンス）と "DRM Sync Objects" を含む
- [カーネルドキュメント: Buffer Sharing and Synchronization (dma-buf)](https://docs.kernel.org/driver-api/dma-buf.html) — DMA fenceの概要、ドライバ横断の契約、シグナルのアノテーション、デッドラインのヒント、"Indefinite DMA Fences" の論拠
- [カーネルドキュメント: DRM uAPI](https://docs.kernel.org/gpu/drm-uapi.html) — ハングをユーザー空間に伝播させることに関する "Device reset"、"Robustness"、"Device Wedging" のセクション
- [`a72ce6f84109`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a72ce6f84109c1dec1ab236d65979d3250668af3) — "drm/amd: add basic scheduling framework"、Linux 4.3（2015年）。最初のAMD専用スケジューラ
- [lore.kernel.org: Add GPU scheduler to amdgpu](https://lore.kernel.org/dri-devel/1438381367-24980-1-git-send-email-alexander.deucher@amd.com/) — それを導入した31パッチのシリーズ。完成したまとまりとして投稿され、返信はゼロだった
- [`1b1f42d8fde4`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1b1f42d8fde4fef1ed7873bf5aa91755f8c3de35) — "drm: move amd_gpu_scheduler into common location"、Lucas Stach、Linux 4.16（2018年）
- [lore.kernel.org: Move scheduler out of AMDGPU](https://lore.kernel.org/dri-devel/20171201152901.3626-1-l.stach@pengutronix.de/) — レビュースレッド。Monk LiuのGPUリセットに関する質問を含む
- [`e93b6deeb45a`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e93b6deeb45a781489f4ceaa97f9545a3cbebb81) — "drm/etnaviv: hook up DRM GPU scheduler"、Linux 4.17。AMD以外の最初のユーザー
- [`08fb97de03aa`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=08fb97de03aa2205c6791301bd83a095abc1949c) · [`977d97f18b5b`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=977d97f18b5b8efb7a94da84724113f15ae6cc2d) — FIFOスケジューリングポリシーの追加とデフォルト化、Linux 6.2
- [`a6149f039369`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a6149f0393699308fb00149be913044977bceb56) — "drm/sched: Convert drm scheduler to use a work queue rather than kthread"、Matthew Brost、Linux 6.8
- [`0b1217bfdfdd`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0b1217bfdfddf664c15954d1d51ee18ed88a2ccf) — "drm/sched: Allow drivers to skip the reset and keep on running"、Maíra Canal、Linux 6.17。`DRM_GPU_SCHED_STAT_NO_HANG`
- [`2fa4d8e2c109`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2fa4d8e2c1091189064c5b93222a23ded2d881ba) · [`77a6809f1dc3`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=77a6809f1dc39376116f8d769a0d2630dc95ad79) — "Add fair scheduling policy" と "Remove FIFO and RR and simplify to a single run queue"、Tvrtko Ursulin、Linux 7.2
- [LWN: "The Linux graphics stack in a nutshell, part 1"](https://lwn.net/Articles/955376/) — Thomas Zimmermann、2023年12月19日。Mesaがどこで終わり、ドライバ固有のioctlがどこで始まるかについて
- `include/drm/gpu_scheduler.h`、`drivers/gpu/drm/scheduler/{sched_main.c,sched_entity.c,sched_fence.c}` — スケジューラ本体
- `include/linux/dma-fence.h`、`drivers/dma-buf/dma-fence.c` — fenceプリミティブと、その文書化された契約
- `include/uapi/drm/amdgpu_drm.h`、`drivers/gpu/drm/amd/amdgpu/{amdgpu_cs.c,amdgpu_job.c,amdgpu_fence.c}` — 実際の完全な投入パス
- [DRM: Direct Rendering Manager](README.md) · [GEMバッファオブジェクトとdma-buf](gem-dmabuf.md) · [カーネルモード設定（KMS）](kms.md) — ジョブが操作するバッファと、その出力の行き先
- [workqueue](../interrupts/workqueues.md) — `submit_wq`、`timeout_wq`、`work_tdr` が土台としている遅延処理の仕組み
