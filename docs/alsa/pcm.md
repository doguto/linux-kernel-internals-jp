# ALSA PCMのデータパス: リングバッファ、period、hw_params、mmap {#the-alsa-pcm-data-path-ring-buffer-periods-hw_params-and-mmap}

> ドライバのDMAエンジンとユーザー空間が共有する1つのリングバッファ、その埋まり具合を追跡する2つのポインタ、そして両者に辻褄を合わせさせる1つの割り込み

## 問題: ドライバ境界を越えてオーディオサンプルを途切れなく受け渡す {#the-problem-getting-audio-samples-across-a-driver-boundary-continuously}

サウンドカードはデータを一度だけ要求するわけではない。再生や録音が続く限り、ずっと、まさに自分のサンプルレートでサンプルが到着（または出発）し続けることを求める。ユーザー空間はそのようには動かない。起床し、ほかのことをして、あとで戻ってきてバッファを補充する。固定の歩調でサンプルを消費または生成するハードウェアのDMAエンジンと、スケジューラに駆動されてバーストでバッファを満たしたり排出したりするプロセスとを、どちらの側もバッファが空（アンダーラン、聞こえるグリッチ）や未読（オーバーラン、サンプルの欠落）の状態を見ることがないように調停する何かが必要である。

ALSAの答えは`snd_pcm`である。USBヘッドセットからHDMIコーデック、ハードウェアDSPに至るまで、あらゆるサウンドドライバが実装の対象とするPCM（"Pulse-Code Modulation"）層だ。これはストリームの方向ごとに1つのリングバッファであり、そのバッファを割り込みの粒度を決める**period**に固定的に分割したものと、2つの位置、すなわちハードウェアがどこまで進んだかとアプリケーションがどこまで進んだかをそれぞれ追跡する`hw_ptr`と`appl_ptr`からなる。このページではその仕組みを扱う。ドライバとユーザー空間が実際に触れる構造体（`struct snd_pcm_substream`、`struct snd_pcm_runtime`）、periodサイズとバッファサイズがどのようにネゴシエートされるか、ユーザー空間がどのようにバッファのゼロコピーな`mmap()`ビューを得るか、そしてドライバの割り込みハンドラがどのように全体を前に進めるかである。

## `struct snd_pcm_substream`と`struct snd_pcm_runtime` {#struct-snd_pcm_substream-and-struct-snd_pcm_runtime}

`struct snd_pcm`はサウンドカードのPCMデバイス（たとえばカードのデフォルトのオーディオデバイスである「デバイス0」）であり、再生方向、録音方向、またはその両方を、それぞれ`struct snd_pcm_str`として公開できる。デバイスの再生方向または録音方向を開くと、プロセスは`struct snd_pcm_substream`（`include/sound/pcm.h`）を得る。同時に開かれたストリームごとに1つである。

```c
// include/sound/pcm.h
struct snd_pcm_substream {
	struct snd_pcm *pcm;
	struct snd_pcm_str *pstr;
	void *private_data;		/* copied from pcm->private_data */
	int number;
	char name[32];			/* substream name */
	int stream;			/* stream (direction) */
	struct pm_qos_request latency_pm_qos_req; /* pm_qos request */
	size_t buffer_bytes_max;	/* limit ring buffer size */
	struct snd_dma_buffer dma_buffer;
	size_t dma_max;
	/* -- hardware operations -- */
	const struct snd_pcm_ops *ops;
	/* -- runtime information -- */
	struct snd_pcm_runtime *runtime;
	/* -- timer section -- */
	struct snd_timer *timer;		/* timer */
	unsigned timer_running: 1;	/* time is running */
	long wait_time;	/* time in ms for R/W to wait for avail */
	/* -- next substream -- */
	struct snd_pcm_substream *next;
	/* -- linked substreams -- */
	struct list_head link_list;
	struct snd_pcm_group self_group;
	struct snd_pcm_group *group;
	/* -- assigned files -- */
	int ref_count;
	atomic_t mmap_count;
	unsigned int f_flags;
	void (*pcm_release)(struct snd_pcm_substream *);
	struct pid *pid;
	...
};
```

`substream->ops`はドライバのvtable（`struct snd_pcm_ops`、後述）であり、ドライバのコードがこの仕組みに差し込まれる唯一の場所である。それ以外のほぼすべては`substream->runtime`にある。`struct snd_pcm_runtime`は、`Documentation/sound/kernel-api/writing-an-alsa-driver.rst`にあるカーネルドキュメント自身の表現によれば「PCM情報の宝箱」である。`snd_pcm_attach_substream()`（`sound/core/pcm.c`、`snd_pcm_open()`のオープンパスから呼ばれる）で新たに割り当てられ、クローズ時に解放されるので、ストリームが開いている間だけ存在する。そのフィールドは、このページの残りで順に見ていくカテゴリにきれいに分かれる。

```c
// include/sound/pcm.h
struct snd_pcm_runtime {
	/* -- Status -- */
	snd_pcm_state_t state;		/* stream state */
	snd_pcm_state_t suspended_state; /* suspended stream state */
	struct snd_pcm_substream *trigger_master;
	struct timespec64 trigger_tstamp;	/* trigger timestamp */
	bool trigger_tstamp_latched;
	int overrange;
	snd_pcm_uframes_t avail_max;
	snd_pcm_uframes_t hw_ptr_base;	/* Position at buffer restart */
	snd_pcm_uframes_t hw_ptr_interrupt; /* Position at interrupt time */
	unsigned long hw_ptr_jiffies;	/* Time when hw_ptr is updated */
	unsigned long hw_ptr_buffer_jiffies; /* buffer time in jiffies */
	snd_pcm_sframes_t delay;	/* extra delay; typically FIFO size */
	u64 hw_ptr_wrap;                /* offset for hw_ptr due to boundary wrap-around */

	/* -- HW params -- */
	snd_pcm_access_t access;	/* access mode */
	snd_pcm_format_t format;	/* SNDRV_PCM_FORMAT_* */
	snd_pcm_subformat_t subformat;	/* subformat */
	unsigned int rate;		/* rate in Hz */
	unsigned int channels;		/* channels */
	snd_pcm_uframes_t period_size;	/* period size */
	unsigned int periods;		/* periods */
	snd_pcm_uframes_t buffer_size;	/* buffer size */
	snd_pcm_uframes_t min_align;	/* Min alignment for the format */
	...
	unsigned int frame_bits;
	unsigned int sample_bits;
	unsigned int info;
	...

	/* -- SW params; see struct snd_pcm_sw_params for comments -- */
	int tstamp_mode;
	unsigned int period_step;
	snd_pcm_uframes_t start_threshold;
	snd_pcm_uframes_t stop_threshold;
	snd_pcm_uframes_t silence_threshold;
	snd_pcm_uframes_t silence_size;
	snd_pcm_uframes_t boundary;
	...

	/* -- mmap -- */
	struct snd_pcm_mmap_status *status;
	struct snd_pcm_mmap_control *control;

	/* -- locking / scheduling -- */
	snd_pcm_uframes_t twake;
	wait_queue_head_t sleep;	/* poll sleep */
	wait_queue_head_t tsleep;	/* transfer sleep */
	struct snd_fasync *fasync;
	bool stop_operating;
	struct mutex buffer_mutex;
	atomic_t buffer_accessing;

	/* -- private section -- */
	void *private_data;
	void (*private_free)(struct snd_pcm_runtime *runtime);

	/* -- hardware description -- */
	struct snd_pcm_hardware hw;
	struct snd_pcm_hw_constraints hw_constraints;

	/* -- timer -- */
	unsigned int timer_resolution;	/* timer resolution */
	int tstamp_type;		/* timestamp type */

	/* -- DMA -- */
	unsigned char *dma_area;	/* DMA area */
	dma_addr_t dma_addr;		/* physical bus address (not accessible from main CPU) */
	size_t dma_bytes;		/* size of DMA area */

	struct snd_dma_buffer *dma_buffer_p;	/* allocated buffer */
	...
};
```

`runtime->private_data`はドライバが所有するポインタである。ほとんどのドライバは、`.open`でそこにストリームごとの小さな構造体（タイマーの状態、DMAチャネルのハンドル、ハードウェアのバックエンドが必要とするものなら何でも）を置き、`.close`で解放する。`runtime->hw`（`struct snd_pcm_hardware`）は、ドライバがオープン時に返す能力の記述である。`runtime->status`/`runtime->control`は、`hw_ptr`と`appl_ptr`をユーザー空間に運ぶ2つのmmapされた構造体であり、後述のmmapの節で扱う。

## リングバッファモデル: periodとフレーム {#the-ring-buffer-model-periods-and-frames}

DMAバッファ（`runtime->dma_area`、サイズは`runtime->dma_bytes`）は単一の連続したリングである。データは**フレーム**単位で計られる。フレームはチャネルごとに1サンプルなので、16ビットステレオのフレームは4バイトである。24ビット6チャネルの場合は、どの24ビット*フォーマット*を使うかによる。隙間なく詰め込まれた`S24_3LE`（1サンプル3バイト）なら18バイトだが、はるかに一般的な`S24_LE`（24ビット値を4バイトのコンテナの下位3バイトに格納する）なら24バイトである。（`S24_LE`/`S24_3LE`は`SNDRV_PCM_FORMAT_*`の別々のエントリであり、単一フォーマットの`subformat`フィールドの変種ではない。`subformat`はそれ自体が別個の、ほとんど使われないhw_paramsの次元である。`SNDRV_PCM_SUBFORMAT_*`を参照。）フレーム数そのものはフォーマットやチャネル数とは独立しており、だからこそPCMコアの残りの部分は、バイトに実際に何が入っているかを気にせずにバッファ位置を扱える。

バッファは、それぞれ`runtime->period_size`フレームからなる`runtime->periods`個の等しいサイズの塊に分割されるので、`buffer_size == period_size * periods`となる。periodは**割り込みの粒度**である。ドライバのハードウェアは、1 period分のフレームを消費（再生）または生成（録音）し終えるたびに1回割り込みを上げることが期待されており、その割り込みがPCMコアに「periodが経過した」ことを伝える。後述の[`snd_pcm_period_elapsed()`](#the-interrupt-driven-update-snd_pcm_period_elapsed)を参照。これはOSSが「フラグメント」と呼んでいたのと同じ設計である。

periodのサイズはレイテンシとオーバーヘッドの直接的なトレードオフであり、データパス全体の中心的な調整つまみである。

- **periodを小さくする**と割り込みが頻繁になるので、ハードウェアがデータを消費してから、カーネルが「もっと書き込める」（または「読み出せるデータが増えた」）ことに気づくまでが早くなる。ユーザー空間はウェイクアップの間隔を乗り切るためにそれほど多くのデータを事前にバッファしておく必要がないので、レイテンシが下がる。代償は1秒あたりの割り込みの増加である。IRQハンドラで費やすCPU時間が増え、スケジューラのウェイクアップが増え、アイドルでいたいものすべてにとって電力特性が悪化する。
- **periodを大きくする**と割り込みは少なく安価になり、スケジュールが遅れたユーザー空間プロセスにとっての余裕も増える。電力にとっても、バースト的で時間的制約の緩い処理とCPUを共有するものにとっても好ましい。代償は、アンダーランを避けるためにより多くのデータを事前にバッファしておく必要があることであり、これはまさに実際にはレイテンシが高いということを意味する。

`buffer_size`そのものは、それがperiodにどう分割されるかとは独立に、外側の上限を決める。再生では、`write()`がブロックするか`mmap()`されたリングが満杯になるまでにどれだけのオーディオをキューに入れられるかであり、録音では、未読のバッファがあふれるまでにハードウェアがどれだけの時間データを生成し続けられるかである。ドライバは`struct snd_pcm_hardware`を通じて両方の次元を制約し、ユーザー空間は`hw_params`の間にその制約内の特定の点をネゴシエートする。これが次の節である。

## hw_paramsのネゴシエーション {#hw_params-negotiation}

### ドライバが宣言する能力: `struct snd_pcm_hardware` {#the-drivers-declared-capability-struct-snd_pcm_hardware}

`.open`の時点で、ドライバは`runtime->hw`に自身のハードウェアが実際にできることを書き込む。

```c
// include/sound/pcm.h
struct snd_pcm_hardware {
	unsigned int info;		/* SNDRV_PCM_INFO_* */
	u64 formats;			/* SNDRV_PCM_FMTBIT_* */
	u32 subformats;			/* for S32_LE, SNDRV_PCM_SUBFMTBIT_* */
	unsigned int rates;		/* SNDRV_PCM_RATE_* */
	unsigned int rate_min;		/* min rate */
	unsigned int rate_max;		/* max rate */
	unsigned int channels_min;	/* min channels */
	unsigned int channels_max;	/* max channels */
	size_t buffer_bytes_max;	/* max buffer size */
	size_t period_bytes_min;	/* min period size */
	size_t period_bytes_max;	/* max period size */
	unsigned int periods_min;	/* min # of periods */
	unsigned int periods_max;	/* max # of periods */
	size_t fifo_size;		/* fifo size in bytes */
};
```

`info`は、`SNDRV_PCM_INFO_MMAP`（このストリームがそもそも`mmap()`可能か。後述）や`SNDRV_PCM_INFO_INTERLEAVED`/`NONINTERLEAVED`（バッファ内のチャネル配置）といった能力フラグを運ぶ。`rates`/`formats`はビットマスク（`SNDRV_PCM_RATE_44100`、`SNDRV_PCM_FMTBIT_S16_LE`など）であり、`rate_min`/`rate_max`/`channels_min`/`channels_max`は連続範囲の境界を定める。ドライバはさらに、平坦な`snd_pcm_hardware`構造体では表現できない関係（たとえば「このチャネル数はこれらの特定のレートでのみ有効」）のために、`hw_constraints`のルール（`snd_pcm_hw_constraint_*()`）を登録できる。

### ネゴシエーションそのもの: 絞り込み、そして確定 {#the-negotiation-itself-refine-then-commit}

ユーザー空間のALSA-lib層（libasoundの`snd_pcm_hw_params()`。カーネル側の同名の関数と混同しないこと）は、`struct snd_pcm_hw_params`を繰り返し絞り込むことで動作する。これは、マスク（列挙型のようなパラメータ: アクセスモード、フォーマット、サブフォーマット）と区間（範囲を持つパラメータ: レート、チャネル数、periodサイズ、period数、バッファサイズ）からなる固定サイズの配列であり、`SNDRV_PCM_HW_PARAM_ACCESS`、`SNDRV_PCM_HW_PARAM_FORMAT`、`SNDRV_PCM_HW_PARAM_RATE`、`SNDRV_PCM_HW_PARAM_PERIOD_SIZE`、`SNDRV_PCM_HW_PARAM_PERIODS`、`SNDRV_PCM_HW_PARAM_BUFFER_SIZE`などのパラメータIDでインデックスされる。各往復は`SNDRV_PCM_IOCTL_HW_REFINE`であり、`snd_pcm_hw_refine()`（`sound/core/pcm_native.c`）に到達する。

```c
// sound/core/pcm_native.c
int snd_pcm_hw_refine(struct snd_pcm_substream *substream,
		      struct snd_pcm_hw_params *params)
{
	int err;

	params->info = 0;
	params->fifo_size = 0;
	...
	err = constrain_mask_params(substream, params);
	if (err < 0)
		return err;

	err = constrain_interval_params(substream, params);
	if (err < 0)
		return err;

	err = constrain_params_by_rules(substream, params);
	if (err < 0)
		return err;

	params->rmask = 0;
	return 0;
}
```

`constrain_mask_params()`と`constrain_interval_params()`は、要求された各マスク/区間を`runtime->hw`および登録された`hw_constraints`ルールと交差させ、範囲をその場で縮める。`constrain_params_by_rules()`はパラメータ間のルール（たとえば、フォーマットがサンプルビット数を制約し、それがフレームビット数を制約する）を適用する。ユーザー空間は、すべてのマスクが単一の値になり、すべての区間が単一の点になる（完全に指定された構成になる）まで、一度に1つのパラメータを絞り込みながらこれを繰り返し呼ぶ。

実際の確定は`SNDRV_PCM_IOCTL_HW_PARAMS`、すなわち`pcm_native.c`の`snd_pcm_hw_params()`である。これは最終的なパラメータに対して`snd_pcm_hw_refine()`を再実行し、`snd_pcm_hw_params_choose()`で具体的な値を選び、ドライバが管理された割り当てを使う場合はDMAバッファを割り当て、そして、ほかのすべてが成功した後でのみ、ドライバ自身の`.hw_params`コールバックを呼んで、フォーマット/レート/チャネルのレジスタを設定させる。成功すると、ネゴシエートされた値を上で見たruntimeのフィールド（`runtime->access`、`format`、`channels`、`rate`、`period_size`、`periods`、`buffer_size`、...）にコピーし、デフォルトのソフトウェアパラメータも設定する。

```c
// sound/core/pcm_native.c — snd_pcm_hw_params(), abridged
runtime->access = params_access(params);
runtime->format = params_format(params);
runtime->subformat = params_subformat(params);
runtime->channels = params_channels(params);
runtime->rate = params_rate(params);
runtime->period_size = params_period_size(params);
runtime->periods = params_periods(params);
runtime->buffer_size = params_buffer_size(params);
...
/* Default sw params */
runtime->tstamp_mode = SNDRV_PCM_TSTAMP_NONE;
runtime->period_step = 1;
runtime->control->avail_min = runtime->period_size;
runtime->start_threshold = 1;
runtime->stop_threshold = runtime->buffer_size;
runtime->silence_threshold = 0;
runtime->silence_size = 0;
runtime->boundary = runtime->buffer_size;
while (runtime->boundary * 2 <= LONG_MAX - runtime->buffer_size)
	runtime->boundary *= 2;
```

ストリームは`SNDRV_PCM_STATE_OPEN`から`SNDRV_PCM_STATE_SETUP`に移行する。`boundary`については一言述べておく価値がある。これはバッファサイズではない。`LONG_MAX`より十分下に収まる、`buffer_size`の2のべき乗倍のうち最大のもの（上の倍々ループ）であり、`hw_ptr`と`appl_ptr`が（`buffer_size`そのものではなく）それで折り返す法である。これによってコアは、ポインタに対して単純な算術を行いながらも、「バッファがN回折り返した」と「ポインタが等しい」とを区別できる。

### sw_params: 能力ではなくポリシー {#sw_params-policy-not-capability}

`hw_params`がハードウェアのサポートするものを記述するのに対し、`SNDRV_PCM_IOCTL_SW_PARAMS`（`struct snd_pcm_sw_params`、`snd_pcm_sw_params()`が処理する）は、そのハードウェアの周りで*コア*がどう振る舞うかを調整する。いつストリームを自動開始するか（`start_threshold`。再生が自動的に始まる前にキューに入っているべき`hw_avail`フレームの最小数）、アンダーラン時にいつ自動停止するか（`stop_threshold`）、そして`avail_min`である。`avail_min`は`poll()`/`select()`のウェイクアップを引き起こす利用可能フレーム数の最小値であり、periodによる割り込みが、ブロックしているユーザー空間プロセスを実際にどれだけ積極的に起こすか、それとも先にもっと溜まるのを待つかを決めるつまみである。`silence_threshold`/`silence_size`は関連するが別個の組である。これらはアンダーランが起きるのを待たない。`snd_pcm_playback_silence()`（`sound/core/pcm_lib.c`）はhw_ptrが更新されるたびに走り、`hw_ptr`と`appl_ptr`の間隔（「ノイズ距離」、すなわちまだ上書きされていない本物のデータがどれだけキューに残っているか）が`silence_threshold`を下回るたびに、`hw_ptr`より*先*のバッファに最大`silence_size`フレームの無音を先回りして書き込む。これにより、実際のxrunがすでに起きたかどうかとは無関係に、ユーザー空間が遅れてもハードウェアが古い/ゴミのサンプルを読むことは決してなくなる。

## mmapパス: リングバッファのゼロコピーなビュー {#the-mmap-path-a-zero-copy-view-of-the-ring-buffer}

`SNDRV_PCM_INFO_MMAP`を通知するドライバでは、ユーザー空間は`read()`/`write()`をまったく使わずに、リングバッファを直接`mmap()`し、サンプルフレームをその場で書き込む（または読み出す）ことができる。カーネル側は`snd_pcm_mmap_data()`（`sound/core/pcm_native.c`）であり、要求されたオフセットがstatus/controlページ（後述）でないと判断された後に、PCMデバイスの`mmap`ファイル操作を通じて到達する。

```c
// sound/core/pcm_native.c
int snd_pcm_mmap_data(struct snd_pcm_substream *substream, struct file *file,
		      struct vm_area_struct *area)
{
	struct snd_pcm_runtime *runtime;
	long size;
	unsigned long offset;
	size_t dma_bytes;
	int err;

	if (substream->stream == SNDRV_PCM_STREAM_PLAYBACK) {
		if (!(area->vm_flags & (VM_WRITE|VM_READ)))
			return -EINVAL;
	} else {
		if (!(area->vm_flags & VM_READ))
			return -EINVAL;
	}
	runtime = substream->runtime;
	if (runtime->state == SNDRV_PCM_STATE_OPEN)
		return -EBADFD;
	if (!(runtime->info & SNDRV_PCM_INFO_MMAP))
		return -ENXIO;
	if (runtime->access == SNDRV_PCM_ACCESS_RW_INTERLEAVED ||
	    runtime->access == SNDRV_PCM_ACCESS_RW_NONINTERLEAVED)
		return -EINVAL;
	size = area->vm_end - area->vm_start;
	offset = area->vm_pgoff << PAGE_SHIFT;
	dma_bytes = PAGE_ALIGN(runtime->dma_bytes);
	if ((size_t)size > dma_bytes)
		return -EINVAL;
	if (offset > dma_bytes - size)
		return -EINVAL;

	area->vm_ops = &snd_pcm_vm_ops_data;
	area->vm_private_data = substream;
	if (substream->ops->mmap)
		err = substream->ops->mmap(substream, area);
	else
		err = snd_pcm_lib_default_mmap(substream, area);
	if (!err)
		atomic_inc(&substream->mmap_count);
	return err;
}
```

この関数は、`mmap()`をインターリーブ/非インターリーブの`read`/`write`アクセスモードと混在させることを拒否し（ストリームは`hw_params`の時点で1つのアクセスモデルを選び、それを使い続ける）、要求されたVMAを実際のDMAバッファサイズに対して境界チェックし、そのうえでドライバ自身の`.mmap`操作を呼ぶか、`snd_pcm_lib_default_mmap()`にフォールバックする。後者は、バッファのページを直接マップする（`dma_alloc_coherent()`で確保されたバッファでは、これは多くの場合`snd_dma_buffer_mmap()`による素直な`remap_pfn_range()`的なマッピングである）か、個々のページをオンデマンドで解決するページフォールトハンドラ（`snd_pcm_mmap_data_fault()`）をインストールする。後者は、線形なカーネル仮想DMAマッピングを持たないドライバがたどるパスであり、フォールトごとに`substream->ops->page()`を呼ぶ。

### 残り2つのmmap領域: statusとcontrol {#the-other-two-mmap-regions-status-and-control}

リングバッファ自体は、PCMファイルディスクリプタを`mmap()`できる3つの対象のうちの1つである。残りの2つは固定オフセット・固定サイズの構造体（`SNDRV_PCM_MMAP_OFFSET_STATUS`/`_CONTROL`）であり、位置ポインタを運ぶ。そのため、ユーザー空間は「あとどれだけ空きがあるか」を確認するためだけにioctlの往復をする必要がない。uapiヘッダの「基本」構造体タグは実際には`struct __snd_pcm_mmap_status`/`__snd_pcm_mmap_control`であり、それぞれ6個と2個のフィールドを持つ、32ビットタイムスタンプの簡略化されたレイアウトである。しかし`include/uapi/sound/asound.h`は`__KERNEL__`が設定されていれば常に`__SND_STRUCT_TIME64`を定義し、そのマクロの下で、より大きな`*64`版である`struct __snd_pcm_mmap_status64`/`__snd_pcm_mmap_control64`を、`#define`によって素の名前`snd_pcm_mmap_status`/`snd_pcm_mmap_control`に改名する。したがってカーネル内（およびy2038安全な64ビットユーザー空間）では、それらの素の名前は実際にはより大きなパディング付きの構造体に解決され、`runtime->status`/`runtime->control`が指すのは小さいほうではなくこちらである。

```c
// include/uapi/sound/asound.h — the struct kernel builds actually get for
// "snd_pcm_mmap_status" / "snd_pcm_mmap_control", via the __SND_STRUCT_TIME64
// macro-rename of __snd_pcm_mmap_status64 / __snd_pcm_mmap_control64
struct snd_pcm_mmap_status {
	snd_pcm_state_t state;		/* RO: state - SNDRV_PCM_STATE_XXXX */
	__u32 pad1;			/* Needed for 64 bit alignment */
	__pad_before_uframe __pad1;
	snd_pcm_uframes_t hw_ptr;	/* RO: hw ptr (0...boundary-1) */
	__pad_after_uframe __pad2;
	struct __snd_timespec64 tstamp;	/* Timestamp; __kernel_timespec under __KERNEL__ */
	snd_pcm_state_t suspended_state; /* RO: suspended stream state */
	__u32 pad3;			/* Needed for 64 bit alignment */
	struct __snd_timespec64 audio_tstamp; /* from sample counter or wall clock */
};

struct snd_pcm_mmap_control {
	__pad_before_uframe __pad1;
	snd_pcm_uframes_t appl_ptr;	/* RW: appl ptr (0...boundary-1) */
	__pad_before_uframe __pad2;
	__pad_before_uframe __pad3;
	snd_pcm_uframes_t avail_min;	/* RW: min available frames for wakeup */
	__pad_after_uframe __pad4;
};
```

`struct snd_pcm_runtime`の`runtime->status`と`runtime->control`は、これと同じ構造体を指している。カーネルとユーザー空間のmmapされたビューは同じメモリである。`hw_ptr`はカーネルだけが書き込む（ドライバの`.pointer`コールバックから更新される。次の節を参照）。`appl_ptr`はユーザー空間が（mmapでない`read()`/`write()`のストリームでは、アプリケーションに代わってカーネルが）書き込み、どこまで消費または生成したかを示す。

### `appl_ptr`と`hw_ptr`: 生産者と消費者の位置 {#appl_ptr-and-hw_ptr-producer-and-consumer-position}

どちらも単調増加するフレームオフセットであり、`buffer_size`ではなく`runtime->boundary`で折り返す。リング内の実際の位置は`pointer % buffer_size`だが、カウンタをバッファサイズを超えても折り返さずに保つことで、コアは「ハードウェアが開始位置から1.5バッファ先にいる」と「ハードウェアがちょうど1バッファ位置先にいる」とを区別できる。これにより、健全に動作しているストリームと、気づかないうちに余分に1回折り返したストリームとを見分けられる。再生では、`hw_ptr`はハードウェアがバッファをどこまで*消費*（再生）したかであり、`appl_ptr`はユーザー空間がどこまで*書き込んだ*かである。その間隔はまさにキューに入っているがまだ再生されていないデータであり、`snd_pcm_playback_avail()`（`include/sound/pcm.h`）ではその逆、つまりまだ書き込める空き領域として計算される。

```c
// include/sound/pcm.h
static inline snd_pcm_uframes_t snd_pcm_playback_avail(struct snd_pcm_runtime *runtime)
{
	snd_pcm_sframes_t avail = runtime->status->hw_ptr + runtime->buffer_size - runtime->control->appl_ptr;
	if (avail < 0)
		avail += runtime->boundary;
	else if ((snd_pcm_uframes_t) avail >= runtime->boundary)
		avail -= runtime->boundary;
	return avail;
}
```

録音では関係が逆転する。`hw_ptr`はハードウェアがどこまで*生成*（録音）したかであり、`appl_ptr`はユーザー空間がどこまで*読んだ*かであり、`snd_pcm_capture_avail()`は単に`hw_ptr - appl_ptr`（boundaryを法とする）、すなわち新たに録音されてまだ読まれていないデータ量である。どちらの方向でも、2つのポインタの差は最大`buffer_size`フレームまでしか許されない。消費者が追いつかないままその限界に達することが、まさにxrun（再生ではアンダーラン、録音ではオーバーラン）である。

## リングバッファの図 {#ring-buffer-diagram}

```
                     buffer_size = period_size * periods  (here: 4 periods)

        0        period_size   2*period_size  3*period_size   buffer_size
        |--------------|--------------|--------------|--------------|
        | period 0     | period 1     | period 2     | period 3     |
        |--------------|--------------|--------------|--------------|
                              ^ hw_ptr                      ^ appl_ptr
                              (mod buffer_size)              (mod buffer_size)
                              |<------ queued, not yet -------->|
                                    played (playback)/
                                    read (capture)

  hw_ptr, appl_ptr: monotonically increasing counters, wrap at `boundary`
  (a large multiple of buffer_size), not at buffer_size itself.

  playback: hw_ptr advances as hardware consumes each period and fires
            an interrupt; appl_ptr advances as userspace writes/mmap-fills
            more frames. hw_ptr chases appl_ptr; if it catches up = underrun.

  capture:  hw_ptr advances as hardware fills each period and fires an
            interrupt; appl_ptr advances as userspace reads/mmap-drains
            frames. appl_ptr chases hw_ptr; if hw_ptr laps it = overrun.

  One IRQ per period boundary crossed -> snd_pcm_period_elapsed()
```

## 割り込み駆動の更新: `snd_pcm_period_elapsed()` {#the-interrupt-driven-update-snd_pcm_period_elapsed}

ループのドライバ側はこうである。DMAエンジンがperiodを終えるとIRQが発生し、ドライバのハンドラが`snd_pcm_period_elapsed()`（`sound/core/pcm_lib.c`）を呼ぶ。

```c
// sound/core/pcm_lib.c
void snd_pcm_period_elapsed(struct snd_pcm_substream *substream)
{
	if (snd_BUG_ON(!substream))
		return;

	guard(pcm_stream_lock_irqsave)(substream);
	snd_pcm_period_elapsed_under_stream_lock(substream);
}
EXPORT_SYMBOL(snd_pcm_period_elapsed);
```

これはsubstreamのロックの下で`snd_pcm_update_hw_ptr0()`を呼ぶ。この関数こそが、ドライバの`.pointer`コールバックが実際に呼び出される場所である。

```c
// sound/core/pcm_lib.c — snd_pcm_update_hw_ptr0(), abridged
old_hw_ptr = runtime->status->hw_ptr;
pos = substream->ops->pointer(substream);
...
if (pos == SNDRV_PCM_POS_XRUN) {
	__snd_pcm_xrun(substream);
	return -EPIPE;
}
if (pos >= runtime->buffer_size) {
	/* ... treated as a driver bug, position reset to 0 with a rate-limited log ... */
}
pos -= pos % runtime->min_align;
hw_base = runtime->hw_ptr_base;
new_hw_ptr = hw_base + pos;
...
/* new_hw_ptr might be lower than old_hw_ptr in case when */
/* pointer crosses the end of the ring buffer */
if (new_hw_ptr < old_hw_ptr) {
	hw_base += runtime->buffer_size;
	if (hw_base >= runtime->boundary) {
		hw_base = 0;
		crossed_boundary++;
	}
	new_hw_ptr = hw_base + pos;
}
```

これがドライバの`.pointer`コールバックに課す契約に注意してほしい。コールバックは*バッファ内の*生の位置、つまり`0 .. buffer_size - 1`（ハードウェアがすでにxrunしている場合は`SNDRV_PCM_POS_XRUN`）を返す。そして、その折り返された、ハードウェア相対の位置を、`snd_pcm_playback_avail()`/`snd_pcm_capture_avail()`が使う単調増加でboundaryで折り返す`hw_ptr`に変換する責任は、`hw_ptr_base`を追跡し折り返しを自分で検出することによって、すべてコアが負う。ドライバは通常`runtime->status->hw_ptr`に直接書き込むことはない。例外は、明示的なハードウェア位置リセットのioctlを持つ少数のドライバであり、RME9652ファミリ（`sound/pci/rme9652/{hdsp,rme9652,hdspm}.c`）のように、そのリセットハンドラ内で`snd_pcm_update_hw_ptr0()`を経由せずに、自身の位置読み戻しから`runtime->status->hw_ptr`を直接代入するものである。

位置が更新されると、`snd_pcm_update_hw_ptr0()`自身が`snd_pcm_update_state()`に処理を渡す。十分なフレームが利用可能になったときに、`runtime->sleep`/`runtime->tsleep`で眠っているもの（ブロックしている`read()`/`write()`や、`avail_min`を超えた`poll()`）を実際に起こすのはこれである。`snd_pcm_period_elapsed_under_stream_lock()`に戻ると、`CONFIG_SND_PCM_TIMER`が有効なら、その方法でperiod経過イベントを購読しているもののために、PCMに関連付けられたALSAタイマーも刻む。XRUNとDRAINの処理があるのもまさにこの地点である。`.pointer`が`SNDRV_PCM_POS_XRUN`を報告すると、`__snd_pcm_xrun()`がまさにここで、割り込みコンテキストのまま、ストリームを`SNDRV_PCM_STATE_XRUN`に遷移させ、ユーザー空間は次にストリームに触れたときにそれを知る。

## ドライバ側のコールバック操作: `struct snd_pcm_ops` {#driver-side-callback-ops-struct-snd_pcm_ops}

ドライバがPCMコアに差し込むものはすべて、1つのvtable、`substream->ops`を通る。

```c
// include/sound/pcm.h
struct snd_pcm_ops {
	int (*open)(struct snd_pcm_substream *substream);
	int (*close)(struct snd_pcm_substream *substream);
	int (*ioctl)(struct snd_pcm_substream * substream,
		     unsigned int cmd, void *arg);
	int (*hw_params)(struct snd_pcm_substream *substream,
			 struct snd_pcm_hw_params *params);
	int (*hw_free)(struct snd_pcm_substream *substream);
	int (*prepare)(struct snd_pcm_substream *substream);
	int (*trigger)(struct snd_pcm_substream *substream, int cmd);
	int (*sync_stop)(struct snd_pcm_substream *substream);
	snd_pcm_uframes_t (*pointer)(struct snd_pcm_substream *substream);
	int (*get_time_info)(struct snd_pcm_substream *substream,
			     struct timespec64 *system_ts, struct timespec64 *audio_ts,
			     struct snd_pcm_audio_tstamp_config *audio_tstamp_config,
			     struct snd_pcm_audio_tstamp_report *audio_tstamp_report);
	int (*fill_silence)(struct snd_pcm_substream *substream, int channel,
			    unsigned long pos, unsigned long bytes);
	int (*copy)(struct snd_pcm_substream *substream, int channel,
		    unsigned long pos, struct iov_iter *iter, unsigned long bytes);
	struct page *(*page)(struct snd_pcm_substream *substream,
			     unsigned long offset);
	int (*mmap)(struct snd_pcm_substream *substream, struct vm_area_struct *vma);
	int (*ack)(struct snd_pcm_substream *substream);
};
```

データパスそのものの中核となるものは次のとおりである。

- **`.open`/`.close`** — ストリームごとのドライバ状態（一般に`runtime->private_data`に置かれる）を割り当て/解放し、この特定のデバイスインスタンスがサポートするものを記述するよう`runtime->hw`を設定する。
- **`.hw_params`/`.hw_free`** — ネゴシエートされた構成に合わせてハードウェアのフォーマット/レート/チャネルのレジスタを設定し、またそれを元に戻す。コアがすでにパラメータを検証・記録した後に、`snd_pcm_hw_params()`/`do_hw_free()`から呼ばれる。
- **`.prepare`** — ドライバ内部の位置/状態をバッファの先頭にリセットし、次の`.trigger(START)`に備える。`hw_params`の後と、`SNDRV_PCM_STATE_XRUN`から回復するたびに呼ばれる。
- **`.trigger`** — リアルタイム性が問われるもの。`cmd`（`SNDRV_PCM_TRIGGER_START`、`_STOP`、`_PAUSE_PUSH`、`_PAUSE_RELEASE`、`_SUSPEND`、`_RESUME`、`_DRAIN`）によって選ばれる、開始、停止、一時停止、再開、またはドレイン開始である。少なくとも一部の呼び出しパスでは、ストリームロックを保持し割り込みを無効にした状態で呼ばれるので、高速でなければならない。DMAが実際に起動または停止されるのはここである。
- **`.pointer`** — バッファ内の現在のハードウェア位置をフレーム単位で報告する。上で示したとおり、`snd_pcm_update_hw_ptr0()`が呼ぶのはこれである。
- **`.copy`/`.fill_silence`/`.page`** — ドライバが単純な線形のカーネルマップされたDMAバッファを公開できない場合（散在するページ、ハードウェア固有の転送要件）にのみ必要である。ほとんどのドライバはこれらを未設定のままにし、コアのデフォルトの線形バッファパス（`memcpy`ベースの`read`/`write`、直接のmmap）に任せる。
- **`.mmap`** — デフォルトのバッファmmapロジック（`snd_pcm_lib_default_mmap()`）を完全に置き換える。普通のDMAコヒーレントなメモリを使うほとんどのドライバはこれをNULLのままにする。

## 実例: `sound/drivers/dummy.c`のsystimerバックエンド {#worked-example-sounddriversdummycs-systimer-backend}

`sound/drivers/dummy.c`は、実ハードウェアをまったく持たないPCMデバイスを実装している。ALSA PCMコアを単独でテスト・デモするために存在しており、そのため、実際のDMA駆動の実装ではなく、最小限の`.pointer`/`.trigger`実装の、すっきりとした、本物の、ツリー内の例になっている。「ハードウェアの進行」のために選択可能なバックエンドが2つある。`jiffies`駆動のシステムタイマーと、（`CONFIG_HIGH_RES_TIMERS`が有効なら）`hrtimer`である。最初の例として読むにはsystimerのほうが単純である。

substreamごとの状態:

```c
// sound/drivers/dummy.c
struct dummy_systimer_pcm {
	/* ops must be the first item */
	const struct dummy_timer_ops *timer_ops;
	spinlock_t lock;
	struct timer_list timer;
	unsigned long base_time;
	unsigned int frac_pos;	/* fractional sample position (based HZ) */
	unsigned int frac_period_rest;
	unsigned int frac_buffer_size;	/* buffer_size * HZ */
	unsigned int frac_period_size;	/* period_size * HZ */
	unsigned int rate;
	int elapsed;
	struct snd_pcm_substream *substream;
};
```

`dummy_systimer_update()`は、前回の更新から経過したjiffies分だけ小数部を含む位置を進め、それをストリームの`rate`でフレームに換算し、いくつのperiod境界を越えたかを数える。

```c
// sound/drivers/dummy.c
static void dummy_systimer_update(struct dummy_systimer_pcm *dpcm)
{
	unsigned long delta;

	delta = jiffies - dpcm->base_time;
	if (!delta)
		return;
	dpcm->base_time += delta;
	delta *= dpcm->rate;
	dpcm->frac_pos += delta;
	while (dpcm->frac_pos >= dpcm->frac_buffer_size)
		dpcm->frac_pos -= dpcm->frac_buffer_size;
	while (dpcm->frac_period_rest <= delta) {
		dpcm->elapsed++;
		dpcm->frac_period_rest += dpcm->frac_period_size;
	}
	dpcm->frac_period_rest -= delta;
}
```

`struct timer_list`のコールバックはその更新を実行し、次に予想されるperiod境界に向けてタイマーを再設定し、少なくとも1つのperiodが実際に経過した場合にのみ`snd_pcm_period_elapsed()`を呼ぶ。これはまさに上で説明した仕組みであり、本物のドライバのDMA完了IRQハンドラが行うことの代役を務めている。

```c
// sound/drivers/dummy.c
static void dummy_systimer_callback(struct timer_list *t)
{
	struct dummy_systimer_pcm *dpcm = timer_container_of(dpcm, t, timer);
	int elapsed = 0;

	scoped_guard(spinlock_irqsave, &dpcm->lock) {
		dummy_systimer_update(dpcm);
		dummy_systimer_rearm(dpcm);
		elapsed = dpcm->elapsed;
		dpcm->elapsed = 0;
	}
	if (elapsed)
		snd_pcm_period_elapsed(dpcm->substream);
}
```

`.pointer`は、現在の小数部を含む位置をフレームに戻して報告するだけである。

```c
// sound/drivers/dummy.c
static snd_pcm_uframes_t
dummy_systimer_pointer(struct snd_pcm_substream *substream)
{
	struct dummy_systimer_pcm *dpcm = substream->runtime->private_data;

	guard(spinlock)(&dpcm->lock);
	dummy_systimer_update(dpcm);
	return dpcm->frac_pos / HZ;
}
```

そして`.trigger`/`.prepare`は、このバックエンドの`start`/`stop`/`prepare`への薄いディスパッチである。`dummy_pcm_trigger()`は`SNDRV_PCM_TRIGGER_START`/`_RESUME`を`start()`（`base_time`を記録してタイマーを設定する）に、`_STOP`/`_SUSPEND`を`stop()`（タイマーを解除する）に対応付ける。

```c
// sound/drivers/dummy.c
static int dummy_pcm_trigger(struct snd_pcm_substream *substream, int cmd)
{
	switch (cmd) {
	case SNDRV_PCM_TRIGGER_START:
	case SNDRV_PCM_TRIGGER_RESUME:
		return get_dummy_ops(substream)->start(substream);
	case SNDRV_PCM_TRIGGER_STOP:
	case SNDRV_PCM_TRIGGER_SUSPEND:
		return get_dummy_ops(substream)->stop(substream);
	}
	return -EINVAL;
}

static const struct snd_pcm_ops dummy_pcm_ops = {
	.open =		dummy_pcm_open,
	.close =	dummy_pcm_close,
	.hw_params =	dummy_pcm_hw_params,
	.prepare =	dummy_pcm_prepare,
	.trigger =	dummy_pcm_trigger,
	.pointer =	dummy_pcm_pointer,
};

static const struct snd_pcm_ops dummy_pcm_ops_no_buf = {
	.open =		dummy_pcm_open,
	.close =	dummy_pcm_close,
	.hw_params =	dummy_pcm_hw_params,
	.prepare =	dummy_pcm_prepare,
	.trigger =	dummy_pcm_trigger,
	.pointer =	dummy_pcm_pointer,
	.copy =		dummy_pcm_copy,
	.fill_silence =	dummy_pcm_silence,
	.page =		dummy_pcm_page,
};
```

この6エントリの`snd_pcm_ops`（`open`/`close`/`hw_params`/`prepare`/`trigger`/`pointer`）は、動作するPCMデバイスの実用上の最小限に近い。それ以外のすべて（`.copy`、`.fill_silence`、`.page`、`.mmap`）は未設定のままにされ、コアのデフォルトの線形DMAバッファパスが処理する。これはまさに上で述べた「ほとんどのドライバはこれらを未設定のままにする」場合である。ただし`dummy.c`は、カード作成時に`fake_buffer`モジュールパラメータ（`static bool fake_buffer = 1;`、つまりデフォルトで*オン*）に基づいて上の2つの構造体のどちらかを選ぶ。

```c
// sound/drivers/dummy.c
if (fake_buffer)
	ops = &dummy_pcm_ops_no_buf;
else
	ops = &dummy_pcm_ops;
```

したがって、そのままの状態で`fake_buffer`がデフォルトの1なら、`dummy.c`が実際に読み込むのは*9*エントリの`dummy_pcm_ops_no_buf`である。渡すべき本物のDMAで裏付けられたバッファがないので、サンプルデータを運ぶために`.copy`/`.fill_silence`/`.page`を自分で実装しなければならないからだ。上の素の6エントリの`dummy_pcm_ops`は最小限のvtableの例示であり、本物の線形DMAバッファを持つドライバが使うものだが、`dummy.c`自身がそれにフォールバックするのは`fake_buffer=0`で読み込まれたときだけである。本物のハードウェアドライバの`.pointer`はjiffiesから導いた小数の代わりにDMA位置レジスタを読み、その`.trigger`はソフトウェアタイマーを設定する代わりに実際のDMA開始ビットを切り替える。しかし形、すなわち位置を進め、period境界に気づき、「ハードウェアが進んだ」ことの代わりとなる何らかの割り込み源から`snd_pcm_period_elapsed()`を呼ぶという形は同一である。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [`include/sound/pcm.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/sound/pcm.h) — `struct snd_pcm_substream`、`struct snd_pcm_runtime`、`struct snd_pcm_hardware`、`struct snd_pcm_ops`
- [`include/uapi/sound/asound.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/sound/asound.h) — uAPIの構造体: `snd_pcm_hw_params`、`snd_pcm_sw_params`、`snd_pcm_mmap_status`、`snd_pcm_mmap_control`、ioctl番号
- [`sound/core/pcm_native.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/sound/core/pcm_native.c) — `snd_pcm_hw_refine()`、`snd_pcm_hw_params()`、`snd_pcm_sw_params()`、`snd_pcm_mmap_data()`
- [`sound/core/pcm_lib.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/sound/core/pcm_lib.c) — `snd_pcm_period_elapsed()`、`snd_pcm_update_hw_ptr0()`、hw_paramsの制約の仕組み
- [`sound/drivers/dummy.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/sound/drivers/dummy.c) — 上の実例のドライバ: 完全で最小限の`snd_pcm_ops`実装

### 関連ページ {#related-pages}

- [ALSA: サウンドサブシステム](README.md) — カード/デバイスモデルの中でPCM層が占める位置
- [ALSAの事件簿](war-stories.md) — 事例1: このページのネゴシエーションとロックの仕組みが、それをカバーできるだけの広さのクリティカルセクションを持たずに出荷された`hw_params`/`hw_free`競合（CVE-2022-1048）

### 外部資料 {#external}

- [Writing an ALSA Driver: PCM Interface](https://docs.kernel.org/sound/kernel-api/writing-an-alsa-driver.html#pcm-interface) — `snd_pcm_runtime`、period、そしてすべての`snd_pcm_ops`コールバックについての、ドライバ作者向けの標準的なガイド
- [ALSA PCM Timestamping](https://docs.kernel.org/sound/designs/timestamping.html) — `hw_ptr`の更新が、`SNDRV_PCM_IOCTL_STATUS`/`SYNC_PTR`と一緒に報告されるタイムスタンプとどう関係するか
