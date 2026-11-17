# ASoC: 組み込みシステムのためのオーディオ {#asoc-audio-for-embedded-systems}

> なぜスマートフォンのオーディオパスは1つではなく3つのドライバをつなぎ合わせたものなのか、そしてDAPMはどのようにしてその大部分の電源を切ったままにしておくのか

## 問題: 「サウンドカード」は実はボード上の3つのチップである {#the-problem-the-sound-card-is-actually-three-chips-on-a-board}

ALSAの元々のドライバモデルは、サウンドカードが1つのデバイスであることを前提としている。PCIやUSBのサウンドカードには、DMAエンジン、コーデックのレジスタ、ミキサーコントロールを所有する単一の`snd_pcm`ドライバがある。物理的に1つのバスアドレスの背後にある1つのチップ（または1枚の小さなボード）だからである。`hw_params`/`trigger`/`pointer`のための`snd_pcm_ops`と、音量のための少数の`snd_kcontrol`があれば十分である。ほかに記述すべきものがないからだ。コーデックはDMAエンジンと同じシリコンにはんだ付けされており、ほかのカードに現れることは決してない。

組み込みやモバイルのオーディオはそうはなっていない。典型的なスマートフォンやシングルボードコンピュータには次のものがある。

- SoC自体の一部である**I2S/TDMコントローラ**。CPU、GPU、そのほかダイ上のすべてとシリコンを共有している。
- **コーデック**。ボードのI2CまたはSPIバス上にある独立したDAC/ADCチップ（Wolfson、Cirrus、Maxim、Realtek、TI...）であり、オーディオ用にはSoCのI2Sピンに、制御用にはまったく別のバスに配線されている。
- 場合によってはスピーカーを駆動する**独立したアンプ**。これも独自のイネーブルGPIOとゲインレジスタを持つ別個のチップである。

この3つはどれも同じシリコンではない。そして決定的なことに、*同じ*コーデックチップと*同じ*SoCのI2Sブロックが、互いに無関係な何十ものボードで異なる組み合わせで登場する。PCIサウンドカードのドライバが「このDMAエンジンはこのコーデックと通信する」とハードコードするように、「このI2Sコントローラはこのコーデックと通信する」とハードコードしたドライバは、ボードごとに書き直さなければならなくなる。`Documentation/sound/soc/overview.rst`は、ASoCが解決するために書かれた具体的な問題としてこれを挙げており、まさにこのコーデック再利用の例を使っている。

> *"Codec drivers were often tightly coupled to the underlying SoC CPU. This is not ideal and leads to code duplication - for example, Linux had different wm8731 drivers for 4 different SoC platforms."*（訳: 「コーデックドライバは、しばしば下位のSoC CPUと密結合していた。これは理想的ではなく、コードの重複につながる。たとえばLinuxには、4つの異なるSoCプラットフォーム向けに別々のwm8731ドライバがあった。」）

同じドキュメントは、ASoC以前の組み込みオーディオの制約をさらに2つ挙げている。ヘッドホンジャックの挿入のようなユーザー起因のイベントを通知する標準的な方法がなかったこと、そしてコーデックドライバが、再生や録音のたびにチップ*全体*の電源を入れていたことである。壁のコンセントにつながったPCなら問題ないが、バッテリーにとっては無駄である。

ASoCの答えは、「サウンドカード」を1つのドライバとして扱うのをやめ、ハードウェアにすでにある継ぎ目にちょうど沿って、再利用可能な3つの部品に分割することである。

> *"Codec class drivers: The codec class driver is platform independent... Platform class drivers: The platform class driver includes the audio DMA engine driver, digital audio interface (DAI) drivers... Machine class driver: The machine driver class acts as the glue that describes and binds the other component drivers together to form an ALSA 'sound card device'."*（訳: 「コーデッククラスドライバ: コーデッククラスドライバはプラットフォームに依存しない……プラットフォームクラスドライバ: プラットフォームクラスドライバには、オーディオDMAエンジンドライバ、デジタルオーディオインターフェース（DAI）ドライバが含まれる……マシンクラスドライバ: マシンドライバクラスは、ほかのコンポーネントドライバを記述し結び付けてALSAの『サウンドカードデバイス』を形成する接着剤として働く。」）
> — [`Documentation/sound/soc/overview.rst`](https://docs.kernel.org/sound/soc/overview.html)

このページの残りはその形に沿って進む。3種類のドライバがそれぞれ実際にコアに何を登録するのか、マシンドライバがそのうち2つをどのようにつないで動作する`dai_link`にするのか、そしてその結果できるオーディオパスに、なぜPCのサウンドカードでは気にする必要のなかった独自のきめ細かい電源モデル（DAPM）が必要なのかを扱う。

## 3つのドライバコンポーネント {#three-driver-components}

### CPU DAIドライバ: 配線のSoC側 {#cpu-dai-driver-the-soc-side-of-the-wire}

CPU DAIドライバは、SoCのデジタルオーディオインターフェースのハードウェアを所有する。すなわち、ビットクロックとフレーム同期のレートでビットをバス上に送り出すI2S/TDM/PCMコントローラと、リングバッファからそれにデータを供給するDMAの配管である。CPU DAIドライバは配線の反対側に何があるかを知らない。フォーマットとレートを設定され、開始を指示されればよいだけである。

CPU DAIドライバがASoCコアに公開するインターフェースは、[`include/sound/soc-dai.h`](https://raw.githubusercontent.com/torvalds/linux/master/include/sound/soc-dai.h)で定義されている`struct snd_soc_dai_driver`である。

```c
// include/sound/soc-dai.h
struct snd_soc_dai_driver {
	/* DAIの記述 */
	const char *name;
	unsigned int id;
	unsigned int base;
	struct snd_soc_dobj dobj;
	const struct of_phandle_args *dai_args;

	/* 操作 */
	const struct snd_soc_dai_ops *ops;
	const struct snd_soc_cdai_ops *cops;

	/* DAIの能力 */
	struct snd_soc_pcm_stream capture;
	struct snd_soc_pcm_stream playback;
	unsigned int symmetric_rate:1;
	unsigned int symmetric_channels:1;
	unsigned int symmetric_sample_bits:1;
};
```

`capture`と`playback`はそれぞれ`struct snd_soc_pcm_stream`であり、サポートするレートとフォーマットをビットマスク（`SNDRV_PCM_RATE_*`、`SNDRV_PCM_FMTBIT_*`）で、加えて`channels_min`/`channels_max`を素の整数で持つ。普通のALSA PCMドライバが使うのと同じ語彙である。実際の操作は`struct snd_soc_dai_ops`にあり、CPU DAIドライバが気にするのは、クロック/フォーマットの設定関数と標準的なPCMのtriggerシーケンスである。

```c
// include/sound/soc-dai.h（抜粋）
struct snd_soc_dai_ops {
	int (*probe)(struct snd_soc_dai *dai);
	int (*remove)(struct snd_soc_dai *dai);

	/* クロック設定 */
	int (*set_sysclk)(struct snd_soc_dai *dai, int clk_id, unsigned int freq, int dir);
	int (*set_pll)(struct snd_soc_dai *dai, int pll_id, int source,
		unsigned int freq_in, unsigned int freq_out);
	int (*set_clkdiv)(struct snd_soc_dai *dai, int div_id, int div);
	int (*set_bclk_ratio)(struct snd_soc_dai *dai, unsigned int ratio);

	/* フォーマット設定 */
	int (*set_fmt)(struct snd_soc_dai *dai, unsigned int fmt);
	int (*set_tdm_slot)(struct snd_soc_dai *dai,
		unsigned int tx_mask, unsigned int rx_mask, int slots, int slot_width);

	/* デジタルミュート */
	int (*mute_stream)(struct snd_soc_dai *dai, int mute, int stream);

	/* PCM操作。オーディオPCM操作の際にsoc-coreから呼ばれる */
	int (*startup)(struct snd_pcm_substream *, struct snd_soc_dai *);
	void (*shutdown)(struct snd_pcm_substream *, struct snd_soc_dai *);
	int (*hw_params)(struct snd_pcm_substream *, struct snd_pcm_hw_params *, struct snd_soc_dai *);
	int (*hw_free)(struct snd_pcm_substream *, struct snd_soc_dai *);
	int (*trigger)(struct snd_pcm_substream *, int, struct snd_soc_dai *);
};
```

I2Sプロトコルの詳細を運ぶのは`set_fmt`である。データの詰め方（I2S/左詰め/右詰め/DSP）、クロックの極性、そしてフォーマットビットの改修以降は、どちら側がビットクロックとフレーム同期を駆動するか、である。この最後の部分は以前は`_MASTER`/`_SLAVE`と表記されていたが、現在のヘッダではprovider/consumerという用語で表記され、ビットパターンはリンクのどちら側がどの信号を所有するかを明示している。

```c
// include/sound/soc-dai.h
#define SND_SOC_DAIFMT_CBP_CFP		(1 << 12) /* コーデックがクロックprovider & フレームprovider */
#define SND_SOC_DAIFMT_CBC_CFP		(2 << 12) /* コーデックがクロックconsumer & フレームprovider */
#define SND_SOC_DAIFMT_CBP_CFC		(3 << 12) /* コーデックがクロックprovider & フレームconsumer */
#define SND_SOC_DAIFMT_CBC_CFC		(4 << 12) /* コーデックがクロックconsumer & フレームconsumer */
```

RockchipのI2Sドライバは、CPU DAIドライバのコンパクトな実例である。コーデック固有のコードはどこにもなく、I2Sコントローラだけを扱っている。

```c
// sound/soc/rockchip/rockchip_i2s.c
static const struct snd_soc_dai_ops rockchip_i2s_dai_ops = {
	.probe = rockchip_i2s_dai_probe,
	.hw_params = rockchip_i2s_hw_params,
	.set_bclk_ratio	= rockchip_i2s_set_bclk_ratio,
	.set_sysclk = rockchip_i2s_set_sysclk,
	.set_fmt = rockchip_i2s_set_fmt,
	.trigger = rockchip_i2s_trigger,
};

static struct snd_soc_dai_driver rockchip_i2s_dai = {
	.ops = &rockchip_i2s_dai_ops,
	.symmetric_rate = 1,
};
```

（`playback`/`capture`の`snd_soc_pcm_stream`は静的初期化子には含まれていない。同じシリコンブロックがボードによって再生専用、録音専用、またはその両方として配線されるため、`rockchip_i2s_init_dai()`がprobe時にdevicetreeからそれらを埋める。）

ここで指摘しておく価値のある構造上の詳細がある。ASoC自身のドキュメントがいまだに古い3分割、つまりコーデッククラスドライバとは別の、DMAエンジン用の「プラットフォームクラスドライバ」を説明しているからである。その構造体はもう存在しない。現在のメインラインに対して`grep -rn "struct snd_soc_platform" include/sound/`を実行しても何も返らない。DMA/プラットフォームの役割は、コーデック側が使うのと同じコンポーネントモデル（後述）に統合された。CPU DAIドライバは自身の`snd_soc_dai_driver`を登録し、DMA側については、PCMコールバックを自分で実装する代わりに、通常は`devm_snd_dmaengine_pcm_register()`のようなヘルパーを呼ぶだけである。

```c
// sound/soc/rockchip/rockchip_i2s.c
ret = devm_snd_soc_register_component(&pdev->dev,
				      &rockchip_i2s_component,
				      dai, 1);
...
ret = devm_snd_dmaengine_pcm_register(&pdev->dev, NULL, 0);
```

### コーデックドライバ: DAC/ADCチップそのもの {#codec-driver-the-dacadc-chip-itself}

コーデックドライバはプラットフォームに依存しない半分である。特定のDAC/ADCチップとI2CやSPI経由でどう通信するか、そのミキサートポロジーがどうなっているか、内部の電源ドメインをどういう順序で操作するかは知っているが、どのSoCやどのボードに配線されているかについては何も知らない。これこそが、`overview.rst`が嘆くwm8731が4つある問題を解決可能にするものである。コーデックドライバを一度書けば、たまたまそのチップを使っているすべてのボードで再利用できる。

構造的には、コーデックドライバはCPU DAIドライバとほとんど同じに見える。I2Sリンクのコーデック側もDAIなので、コーデックドライバも`snd_soc_dai_driver`を登録する。異なるのは、それと一緒に登録する*コンポーネント*である。古いASoCのコードにはこのための独立した`struct snd_soc_codec`があったが、現在のメインラインにはない。`include/sound/`に対して同じように`struct snd_soc_codec`を`grep`しても、見つかるのは`snd_soc_codec_conf`（マシンドライバ側のコーデックごとの設定項目であり、ドライバ向けの構造体ではない）だけである。コーデックドライバ、CPU DAI/プラットフォームドライバ、さらにはDSP/圧縮オーディオのドライバまで、すべて同じ[`struct snd_soc_component_driver`](https://raw.githubusercontent.com/torvalds/linux/master/include/sound/soc-component.h)を通じて登録される。

```c
// include/sound/soc-component.h（抜粋）
struct snd_soc_component_driver {
	const char *name;

	const struct snd_kcontrol_new *controls;
	unsigned int num_controls;
	const struct snd_soc_dapm_widget *dapm_widgets;
	unsigned int num_dapm_widgets;
	const struct snd_soc_dapm_route *dapm_routes;
	unsigned int num_dapm_routes;

	int (*probe)(struct snd_soc_component *component);
	void (*remove)(struct snd_soc_component *component);

	unsigned int (*read)(struct snd_soc_component *component, unsigned int reg);
	int (*write)(struct snd_soc_component *component, unsigned int reg, unsigned int val);

	int (*set_sysclk)(struct snd_soc_component *component,
			  int clk_id, int source, unsigned int freq, int dir);
	int (*set_jack)(struct snd_soc_component *component,
			struct snd_soc_jack *jack, void *data);
	int (*set_bias_level)(struct snd_soc_component *component,
			      enum snd_soc_bias_level level);

	/* PCM操作 — DMA/プラットフォーム型のコンポーネントが使う */
	int (*hw_params)(struct snd_soc_component *component,
			 struct snd_pcm_substream *substream, struct snd_pcm_hw_params *params);
	int (*trigger)(struct snd_soc_component *component,
		       struct snd_pcm_substream *substream, int cmd);
	snd_pcm_uframes_t (*pointer)(struct snd_soc_component *component,
				     struct snd_pcm_substream *substream);

	unsigned int idle_bias_on:1;
	unsigned int endianness:1;
};
```

そして、その実行時の対応物は次のとおりである。

```c
// include/sound/soc-component.h（抜粋）
struct snd_soc_component {
	const char *name;
	const char *name_prefix;
	struct device *dev;
	struct snd_soc_card *card;

	unsigned int active;
	struct list_head dai_list;
	int num_dai;

	struct regmap *regmap;

	/*
	 * DO NOT use any of the fields below in drivers, they are
	 * temporary and are going to be removed again soon. If you
	 * use them in driver code the driver will be marked as
	 * BROKEN when these fields are removed.
	 */
	struct snd_soc_dapm_context *dapm;
	int (*init)(struct snd_soc_component *component);
};
```

*ドライバ*構造体の`controls`、`dapm_widgets`、`dapm_routes`こそが、コーデックドライバを自己記述的にしているものである。ドライバは自身のDAPMグラフ（後述）を同梱しており、マシンドライバは、毎回コーデック内部のミキサートポロジーを宣言し直すのではなく、ボード固有の部分（外部スピーカー、マイクジャック）だけを記述すればよい。

Maximのmax98090コーデックドライバは、このパターンを端から端まで示している。I2S側のための独自の`snd_soc_dai_driver`と、それ以外すべてのための`snd_soc_component_driver`である。

```c
// sound/soc/codecs/max98090.c
static struct snd_soc_dai_driver max98090_dai = {
	.name = "HiFi",
	.playback = {
		.stream_name = "HiFi Playback",
		.channels_min = 2,
		.channels_max = 2,
		.rates = MAX98090_RATES,
		.formats = MAX98090_FORMATS,
	},
	.capture = {
		.stream_name = "HiFi Capture",
		.channels_min = 1,
		.channels_max = 4,
		.rates = MAX98090_RATES,
		.formats = MAX98090_FORMATS,
	},
	.ops = &max98090_dai_ops,
};

static const struct snd_soc_component_driver soc_component_dev_max98090 = {
	.probe			= max98090_probe,
	.remove			= max98090_remove,
	.seq_notifier		= max98090_seq_notifier,
	.set_bias_level		= max98090_set_bias_level,
	.set_jack		= max98090_set_jack,
	.idle_bias_on		= 1,
	.use_pmdown_time	= 1,
	.endianness		= 1,
};
```

これはI2Cのprobe関数から、CPU DAI側が使ったのと同じ`devm_snd_soc_register_component()`で登録される。

```c
// sound/soc/codecs/max98090.c
ret = devm_snd_soc_register_component(&i2c->dev,
				      &soc_component_dev_max98090,
				      &max98090_dai, 1);
```

どちらの構造体もRockchipに言及しておらず、RockchipのI2SドライバもMaximに言及していない。これこそが、`overview.rst`が可能にしようとして書かれた再利用である。この2つを組み合わせるボードを扱うのは、マシンドライバの仕事である。

### マシンドライバ: DAIリンクとボードの接着剤 {#machine-driver-dai-links-and-board-glue}

マシンドライバは3つのうちで唯一、本当にボード固有のものであり、その中心的な仕事は小さい。どのCPU DAIがどのコーデックDAIに、どのバスフォーマットで接続されるかを宣言し、その組を`struct snd_soc_dai_link`としてコアに渡すことである。`Documentation/sound/soc/machine.rst`はその範囲を率直に述べている。

> *"The ASoC machine (or board) driver is the code that glues together all the component drivers (e.g. codecs, platforms and DAIs). It also describes the relationships between each component which include audio paths, GPIOs, interrupts, clocking, jacks and voltage regulators."*（訳: 「ASoCのマシン（またはボード）ドライバは、すべてのコンポーネントドライバ（コーデック、プラットフォーム、DAIなど）を接着するコードである。また、オーディオパス、GPIO、割り込み、クロック、ジャック、電圧レギュレータを含む、各コンポーネント間の関係も記述する。」）

`dai_link`は、CPU DAIとコーデックDAIを直接埋め込んで参照するわけではない。各側は`struct snd_soc_dai_link_component`の配列である。1つのリンクが片側に複数のDAIを持つことが正当にありうるからである（マルチコーデックのボードや、TDM分割されたCPUが複数のコーデックチャネルに供給する場合）。

```c
// include/sound/soc.h
struct snd_soc_dai_link_component {
	const char *name;
	struct device_node *of_node;
	const char *dai_name;
	const struct of_phandle_args *dai_args;
	unsigned int ext_fmt;
};

// struct snd_soc_dai_link（抜粋）
struct snd_soc_dai_link {
	const char *name;			/* コーデック名 */
	const char *stream_name;		/* ストリーム名 */

	struct snd_soc_dai_link_component *cpus;
	unsigned int num_cpus;

	struct snd_soc_dai_link_component *codecs;
	unsigned int num_codecs;

	struct snd_soc_dai_link_component *platforms;
	unsigned int num_platforms;

	unsigned int dai_fmt;			/* 初期化時に設定するフォーマット */

	/* コーデック/マシン固有の初期化 — 例: マシンのコントロールを追加する */
	int (*init)(struct snd_soc_pcm_runtime *rtd);

	const struct snd_soc_ops *ops;

	unsigned int playback_only:1;
	unsigned int capture_only:1;
	unsigned int no_pcm:1;			/* バックエンドリンク、ユーザー空間PCMなし */
	unsigned int dynamic:1;		/* フロントエンド、実行時にBEへルーティングする */
};
```

この構造体には`.cpu_dai_name`/`.codec_name`/`.platform_name`といった文字列フィールドはない。そのより平坦な、DPCM時代の形は現在のメインラインからは消えている（`machine.rst`でいまだに出回っている古い例はその形を示しているが、コピーしてはいけない）。リンクごとに3つのコンポーネント配列を手で埋めるのは十分に面倒なので、`soc.h`はそれらをコンパイル時に構築するためのマクロの組を用意している。

```c
// include/sound/soc.h
#define COMP_CPU(_dai)			{ .dai_name = _dai, }
#define COMP_CODEC(_name, _dai)		{ .name = _name, .dai_name = _dai, }
#define COMP_PLATFORM(_name)		{ .name = _name }

SND_SOC_DAILINK_DEFS(test,
	DAILINK_COMP_ARRAY(COMP_CPU("cpu_dai")),
	DAILINK_COMP_ARRAY(COMP_CODEC("codec", "codec_dai")),
	DAILINK_COMP_ARRAY(COMP_PLATFORM("platform")));

struct snd_soc_dai_link link = {
	...
	SND_SOC_DAILINK_REG(test),
};
```

max98090コーデックを搭載したボード向けのRockchipのマシンドライバは、まさにこのパターンの、現行の実例である。`cpus`と`platforms`の両方が`COMP_EMPTY()`を使っていることに注目してほしい。このSoCでは、I2SコントローラとDMAエンジンは名前ではなくdevicetreeのphandleによってランタイムに照合されるからである。

```c
// sound/soc/rockchip/rockchip_max98090.c
SND_SOC_DAILINK_DEFS(analog,
		     DAILINK_COMP_ARRAY(COMP_EMPTY()),
		     DAILINK_COMP_ARRAY(COMP_CODEC(NULL, "HiFi")),
		     DAILINK_COMP_ARRAY(COMP_EMPTY()));

static struct snd_soc_dai_link rk_max98090_dailinks[] = {
	{
		.name = "max98090",
		.stream_name = "Analog",
		.init = rk_init,
		.ops = &rk_aif1_ops,
		/* max98090をスレーブとして設定する */
		.dai_fmt = SND_SOC_DAIFMT_I2S | SND_SOC_DAIFMT_NB_NF |
			SND_SOC_DAIFMT_CBC_CFC,
		SND_SOC_DAILINK_REG(analog),
	},
};

static struct snd_soc_card rockchip_max98090_card = {
	.name = "ROCKCHIP-I2S",
	.owner = THIS_MODULE,
	.dai_link = rk_max98090_dailinks,
	.num_links = ARRAY_SIZE(rk_max98090_dailinks),
	.aux_dev = &rk_98090_headset_dev,
	.num_aux_devs = 1,
	.dapm_widgets = rk_max98090_dapm_widgets,
	.num_dapm_widgets = ARRAY_SIZE(rk_max98090_dapm_widgets),
	.dapm_routes = rk_max98090_audio_map,
	.num_dapm_routes = ARRAY_SIZE(rk_max98090_audio_map),
	.controls = rk_max98090_controls,
	.num_controls = ARRAY_SIZE(rk_max98090_controls),
};
```

`struct snd_soc_card`は、マシンドライバが`devm_snd_soc_register_card()`で登録する最上位のオブジェクトである。`dai_link`の配列と、コーデックドライバがまだ知らないボードレベルのDAPMウィジェット/ルート/コントロール（外部スピーカー、マイクジャック。後述）からなる。ここでの`dai_fmt`はI2Sの配線フォーマットをリンク上で直接指定している。標準的なI2Sの詰め方、通常の（反転しない）ビットクロックとフレーム、そしてビットクロックとフレーム同期の両方についてコーデックがクロックの*consumer*であること、つまりこのボードではRockchipのI2SコントローラがBCLK/LRCKを駆動する側であることを意味する。`aux_dev`/`num_aux_devs`は、どの`dai_link`にもまったく属さないコンポーネントのための、別の小さな仕組みである。このボードでは`rk_98090_headset_dev`を指しており、そのinitコールバックは（`snd_soc_card_jack_new_pins()`によって、ジャックが報告に使うボタンキーのマスクとDAPMピンとともに）`snd_soc_jack`を設定し、続いて`ts3a227e_enable_jack_detect()`を呼んで、そのジャックをts3a227eヘッドセット検出ICのドライバに登録する。実際に割り込み線を要求するのはIC自身のI2C probeであり、この呼び出しとは独立している。`aux_dev`が配線するのはジャックの報告経路であって、オーディオのデータパスではない。

## DAPM: PCのサウンドカードにはないものの電源管理 {#dapm-power-managing-what-a-pc-sound-card-doesnt-have}

PCのサウンドカードのコーデックは、固定のAC'97またはHD-Audioリンクの背後にあり、壁のコンセントから給電され、通常は支配的な電源状態が1つしかない。オンである。マシン自体が起きているかサスペンドしているかのどちらかだからだ。組み込みのコーデックは、電源にとって重要な2つの点で異なる。バッテリーで動作すること、そして通常は本当に独立したアナログブロックをいくつも含んでいることである。ヘッドホンアンプ、スピーカーアンプ、マイクバイアス電源、ADC、DACなどであり、実際に何が接続され何が再生されているかによって、その時点で必要なのはそれらの任意の部分集合でありうる。マイクバイアスからADCへのパスしか必要としない通話のためにチップ全体の電源を入れるのは電流の無駄遣いであり、ノートPCのサウンドカードでは考える必要のなかったことである。

Dynamic Audio Power Management（DAPM）がASoCの答えであり、このページのほかのすべてと違って、マシンドライバが能動的に管理しなければならないもの*ではない*。`Documentation/sound/soc/dapm.rst`は、マシンドライバはグラフを記述するだけで、残りはコアが行うと明言している。

> *"DAPM is also completely transparent to all user space applications as all power switching is done within the ASoC core. No code changes or recompiling are required for user space applications. DAPM makes power switching decisions based upon any audio stream (capture/playback) activity and audio mixer settings within the device."*（訳: 「すべての電源切り替えはASoCコア内で行われるため、DAPMはすべてのユーザー空間アプリケーションに対して完全に透過的でもある。ユーザー空間アプリケーションにはコードの変更も再コンパイルも必要ない。DAPMは、オーディオストリーム（録音/再生）の活動とデバイス内のオーディオミキサー設定に基づいて電源切り替えの判断を行う。」）

モデルは2種類のオブジェクトからなる有向グラフである。

> *"a **widget** is every part of the audio hardware that can be enabled by software when in use and disabled to save power when not in use... a **route** is an interconnection between widgets that exists when sound can flow from one widget to the other."*（訳: 「**ウィジェット**とは、使用中にはソフトウェアで有効化でき、未使用時には省電力のために無効化できる、オーディオハードウェアのあらゆる部分である……**ルート**とは、あるウィジェットから別のウィジェットへ音が流れうるときに存在する、ウィジェット間の相互接続である。」）

ウィジェットは`struct snd_soc_dapm_widget`である。型付きのノード（ミキサー、マルチプレクサ、PGA、ADC、DAC、マイク、スピーカー、電源、...）であり、その電源を切り替えるレジスタ/ビット/マスクと、グラフを辿るための標準的な管理情報を持つ。

```c
// include/sound/soc-dapm.h（抜粋）
struct snd_soc_dapm_widget {
	enum snd_soc_dapm_type id;
	const char *name;
	const char *sname;			/* ストリーム名。AIF/DAC/ADCウィジェット用 */

	int reg;				/* 負のreg = 直接のdapmなし */
	unsigned char shift;
	unsigned int mask;
	unsigned int on_val;
	unsigned int off_val;
	unsigned char power:1;			/* ブロックの電源状態 */
	unsigned char connected:1;		/* 接続されたコーデックピン */

	/* 外部イベント */
	unsigned short event_flags;
	int (*event)(struct snd_soc_dapm_widget*, struct snd_kcontrol *, int);

	int num_kcontrols;
	const struct snd_kcontrol_new *kcontrol_news;

	struct list_head edges[2];		/* ウィジェットの入力辺と出力辺 */
};
```

ルートははるかに単純な`struct snd_soc_dapm_route`である。`sink`、`source`、そして接続がマルチプレクサやスイッチで制御される場合のオプションの`control`名だけを持つ。

```c
// include/sound/soc-dapm.h
struct snd_soc_dapm_route {
	const char *sink;
	const char *control;
	const char *source;
	int (*connected)(struct snd_soc_dapm_widget *source,
			 struct snd_soc_dapm_widget *sink);
};
```

ウィジェットは型ごとの便利マクロ（`SND_SOC_DAPM_SPK`、`SND_SOC_DAPM_MIC`、`SND_SOC_DAPM_ADC`、`SND_SOC_DAPM_MIXER`など）で宣言され、その大半は正しい`id`とレジスタビットが埋められた`struct snd_soc_dapm_widget`のリテラルに展開される。ここでもRockchipのボードが、グラフのボードレベル側の半分について、現実の最小限の例を示している。コーデックドライバが知らない外部ジャックを、コーデックドライバが*エクスポートしている*ピンに配線するものである。

```c
// sound/soc/rockchip/rockchip_max98090.c
#define RK_MAX98090_WIDGETS \
	SND_SOC_DAPM_HP("Headphone", NULL), \
	SND_SOC_DAPM_MIC("Headset Mic", NULL), \
	SND_SOC_DAPM_MIC("Int Mic", NULL), \
	SND_SOC_DAPM_SPK("Speaker", NULL)

#define RK_MAX98090_AUDIO_MAP \
	{"IN34", NULL, "Headset Mic"}, \
	{"Headset Mic", NULL, "MICBIAS"}, \
	{"DMICL", NULL, "Int Mic"}, \
	{"Headphone", NULL, "HPL"}, \
	{"Headphone", NULL, "HPR"}, \
	{"Speaker", NULL, "SPKL"}, \
	{"Speaker", NULL, "SPKR"}
```

`IN34`、`HPL`/`HPR`、`SPKL`/`SPKR`、`MICBIAS`は、max98090コーデックドライバがすでに登録した内部ピンである。ただし、上で示した`snd_soc_component_driver`の静的初期化子を通じて登録されたのではない。`soc_component_dev_max98090`は`.dapm_widgets`/`.dapm_routes`をまったく設定していない。代わりにドライバは、`probe()`コールバックから、`snd_soc_dapm_new_controls()`/`snd_soc_dapm_add_routes()`を呼ぶヘルパーを介して、手続き的にウィジェットを登録する。

```c
// sound/soc/codecs/max98090.c
static int max98090_add_widgets(struct snd_soc_component *component)
{
	struct max98090_priv *max98090 = snd_soc_component_get_drvdata(component);
	struct snd_soc_dapm_context *dapm = snd_soc_component_to_dapm(component);

	snd_soc_add_component_controls(component, max98090_snd_controls,
		ARRAY_SIZE(max98090_snd_controls));

	if (max98090->devtype == MAX98091) {
		snd_soc_add_component_controls(component, max98091_snd_controls,
			ARRAY_SIZE(max98091_snd_controls));
	}

	snd_soc_dapm_new_controls(dapm, max98090_dapm_widgets,
		ARRAY_SIZE(max98090_dapm_widgets));

	snd_soc_dapm_add_routes(dapm, max98090_dapm_routes,
		ARRAY_SIZE(max98090_dapm_routes));
	...
}
```

静的な`.dapm_widgets`フィールドではなくこの方法で登録することで、ドライバは実行時に異なるウィジェットセットを選べる。上の`if (max98090->devtype == MAX98091)`の分岐はmax98091の派生品向けに2つ目のコントロール配列を追加しているが、これはドライバ構造体上の固定のコンパイル時配列では表現できない。いずれにせよ、マシンドライバはコーデックが内部で自分のヘッドホンアンプにどう電源を入れるかを知る必要はまったくない。「ボードのヘッドホンジャックはコーデックのHPL/HPRピンに配線されている」と言うだけでよく、残りはDAPMのグラフ走査が引き受ける。再生ストリームを開くとDACのエンドポイントがアクティブになり、コアはルートグラフを逆向きに辿って、接続されたアクティブなエンドポイントへのパス上にあるすべてのウィジェットの電源を入れる。アイドルな枝（何も録音していなければマイクのパス）はオフのままである。

`dapm.rst`は電源に関する判断を4つのドメインに分類しており、それぞれ異なる種類のイベント（ストリームの開始/停止、ミキサーのスイッチ、ジャックの挿入、コーデックのサスペンド/レジューム）によって引き起こされる。

| ドメイン | 対象 | 契機 |
|---|---|---|
| コーデックバイアス | VREF/VMID — コーデック/オーディオの中核電源 | probe、サスペンド/レジューム |
| プラットフォーム/マシン | 物理的に接続された入力/出力 | マシンドライバ、非同期イベント（ジャック挿入） |
| パス | ミキサー/マルチプレクサの信号経路 | ユーザーによるコントロールの変更（alsamixer） |
| ストリーム | DACとADC | ストリームの開始/停止（aplay/arecord） |

コーデックバイアスのドメインはそれ自体が小さな状態機械`enum snd_soc_bias_level`であり、その4つの状態は、速く制御されない電源遷移が引き起こすポップノイズやクリックノイズを避けるために、コーデックの`set_bias_level`コールバックがきれいに順を追って遷移することが期待されているものである。enumの本体自体には何の説明もなく、各値の説明はそのすぐ上のkernel-docブロックにある。

```c
// include/sound/soc-dapm.h
/*
 * Bias levels
 *
 * @ON:      Bias is fully on for audio playback and capture operations.
 * @PREPARE: Prepare for audio operations. Called before DAPM switching for
 *           stream start and stop operations.
 * @STANDBY: Low power standby state when no playback/capture operations are
 *           in progress. NOTE: The transition time between STANDBY and ON
 *           should be as fast as possible and no longer than 10ms.
 * @OFF:     Power Off. No restrictions on transition times.
 */
enum snd_soc_bias_level {
	SND_SOC_BIAS_OFF = 0,
	SND_SOC_BIAS_STANDBY = 1,
	SND_SOC_BIAS_PREPARE = 2,
	SND_SOC_BIAS_ON = 3,
};
```

## Devicetree: ドライバを書かずにボードのトポロジーを記述する {#devicetree-describing-the-board-topology-without-writing-a-driver}

ほとんどの組み込みボードは、専用のCのマシンドライバをまったく必要としない。トポロジーが「CPU DAIが1つ、コーデックDAIが1つ、固定のI2Sフォーマット、そしてボードのウィジェット/ルートの短いリスト」だけなら（これは現実のハードウェアのかなりの割合をカバーする）、汎用の**simple-audio-card**ドライバ（`sound/soc/generic/simple-card.c`）が、ボード固有のドライバコードなしに、devicetreeのプロパティから`snd_soc_card`全体を構築できる。DTノードはCPUとコーデックのDAIをphandleで指定し、専用のマシンドライバならCでハードコードするのと同じウィジェット/ルート情報を記述する。

```dts
sound {
    compatible = "simple-audio-card";
    simple-audio-card,name = "VF610-Tower-Sound-Card";
    simple-audio-card,format = "left_j";
    simple-audio-card,bitclock-master = <&dailink0_master>;
    simple-audio-card,frame-master = <&dailink0_master>;
    simple-audio-card,widgets =
            "Microphone", "Microphone Jack",
            "Headphone", "Headphone Jack",
            "Speaker", "External Speaker";
    simple-audio-card,routing =
            "MIC_IN", "Microphone Jack",
            "Headphone Jack", "HP_OUT",
            "External Speaker", "LINE_OUT";

    simple-audio-card,cpu {
        sound-dai = <&sh_fsi2 0>;
    };

    dailink0_master: simple-audio-card,codec {
        sound-dai = <&ak4648>;
        clocks = <&osc>;
    };
};
```
— [`Documentation/devicetree/bindings/sound/simple-card.yaml`](https://www.kernel.org/doc/Documentation/devicetree/bindings/sound/simple-card.yaml)

simple-cardの平坦な`cpu`/`codec`ノードの組ではきれいに表現できないトポロジー（TDM分割を通じて複数のDAIが共有コーデックに供給する場合や、3つ以上のコンポーネントが連なるボード）には、**audio-graph-card**（`sound/soc/generic/audio-graph-card.c`）がある。これはsimple-cardの平坦なプロパティリストの代わりに、汎用のdevicetree [OF graph](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/devicetree/bindings/sound/audio-graph.yaml)の`port`/`endpoint`/`remote-endpoint`バインディングを再利用する。

```dts
sound {
    compatible = "audio-graph-card";
    dais = <&cpu_port_a>;
};

cpu {
    port {
        cpu_endpoint: endpoint {
            remote-endpoint = <&codec_endpoint>;
            dai-format = "left_j";
        };
    };
};

codec {
    port {
        codec_endpoint: endpoint {
            remote-endpoint = <&cpu_endpoint>;
        };
    };
};
```
— [`Documentation/devicetree/bindings/sound/audio-graph-card.yaml`](https://raw.githubusercontent.com/torvalds/linux/master/Documentation/devicetree/bindings/sound/audio-graph-card.yaml)

どちらのドライバも、構造的にはそれ自体がただのマシンドライバである。devicetreeのプロパティを解析し、`rockchip_max98090.c`のような手書きのボードドライバがCで構築するのとまったく同じ`snd_soc_dai_link`/`snd_soc_card`構造体を埋め、同じ`devm_snd_soc_register_card()`を呼ぶ。本物のマシンドライバを書くか、simple-cardを使うか、audio-graph-cardを使うかの選択は、ボードが実際にどれだけのボード固有ロジック（ジャック検出のコールバック、GPIOで制御されるアンプ、自明でない`.init()`フック）を必要とするかだけで決まる。simple-cardやaudio-graph-cardがプロパティで表現できないものは、依然としてCのマシンドライバに頼ることになる。

## 全体の組み合わせ {#how-it-fits-together}

```
                          snd_soc_card                         machine driver
                    (e.g. "ROCKCHIP-I2S", or built
                     from DT by simple-card/
                     audio-graph-card)
                ┌────────────────────────────────┐
                │ dai_link[] ─┬─ .cpus[]         │
                │             ├─ .codecs[]       │
                │             ├─ .platforms[]    │
                │             └─ .dai_fmt        │
                │ board dapm_widgets/routes      │  ("Headphone" → codec's HPL/HPR)
                │ board controls                 │
                └────────────────┬───────────────┘
                                 │ binds cpus[i] <-> codecs[i]
              ┌──────────────────┴──────────────────┐
              ▼                                     ▼
  ┌──────────────────────────────┐            ┌──────────────────────────────┐
  │  CPU DAI driver              │  I2S/TDM   │  Codec driver                │
  │  (e.g. rockchip_i2s.c)       │◄──────────►│  (e.g. max98090.c)           │
  │                              │ BCLK/LRCK  │                              │
  │  snd_soc_component_driver    │  /DATA     │  snd_soc_component_driver    │
  │  snd_soc_dai_driver          │            │  snd_soc_dai_driver          │
  │  .ops: set_fmt, hw_params,   │            │  .ops: set_fmt, set_sysclk   │
  │        trigger               │            │  probe(): registers          │
  │  + DMA engine (component)    │            │    ADC/DAC/mixer/amp         │
  │                              │            │    widgets at runtime        │
  │                              │            │  I2C/SPI regmap I/O          │
  └──────────────┬───────────────┘            └──────────────┬───────────────┘
                 │                                           │
        part of the SoC die                    separate chip on the board,
                                                reused across many boards
```

CPU DAIとコーデックの箱は、ソース上で互いを名前で参照することは決してない。それらが*この特定の*ボード上で配線されていることを知っているのは、マシンドライバ（またはそれが生成される元となったdevicetree）だけである。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [`include/sound/soc.h`](https://raw.githubusercontent.com/torvalds/linux/master/include/sound/soc.h) — `snd_soc_dai_link`、`snd_soc_dai_link_component`、`snd_soc_card`、`SND_SOC_DAILINK_DEFS`/`SND_SOC_DAILINK_REG`マクロ
- [`include/sound/soc-dai.h`](https://raw.githubusercontent.com/torvalds/linux/master/include/sound/soc-dai.h) — `snd_soc_dai`、`snd_soc_dai_driver`、`snd_soc_dai_ops`、DAIフォーマットのビット定義
- [`include/sound/soc-component.h`](https://raw.githubusercontent.com/torvalds/linux/master/include/sound/soc-component.h) — `snd_soc_component`、`snd_soc_component_driver`（コーデック/プラットフォームの統一された登録ポイント）
- [`include/sound/soc-dapm.h`](https://raw.githubusercontent.com/torvalds/linux/master/include/sound/soc-dapm.h) — `snd_soc_dapm_widget`、`snd_soc_dapm_route`、`snd_soc_bias_level`、ウィジェットマクロ
- [`sound/soc/rockchip/rockchip_i2s.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/sound/soc/rockchip/rockchip_i2s.c) — 実際のCPU DAIドライバ
- [`sound/soc/codecs/max98090.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/sound/soc/codecs/max98090.c) — 実際のコーデックドライバ
- [`sound/soc/rockchip/rockchip_max98090.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/sound/soc/rockchip/rockchip_max98090.c) — 実際のマシンドライバ、DAIリンク、ボードのDAPMグラフ
- [`sound/soc/generic/simple-card.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/sound/soc/generic/simple-card.c)、[`sound/soc/generic/audio-graph-card.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/sound/soc/generic/audio-graph-card.c) — devicetree駆動の汎用マシンドライバ

### 関連ページ {#related-pages}

- [Device Tree: ARMのハードウェア記述](../drivers/device-tree.md) — `snd_soc_dai_link_component`とsimple-card/audio-graph-cardが土台とする`of_node`/phandleの仕組み
- [I2CとSPIのバスドライバ](../drivers/i2c-spi.md) — ほとんどのコーデックドライバが登録される制御バス
- [プラットフォームドライバ](../drivers/platform-driver.md) — CPU DAIドライバとマシンドライバの両方が使う`platform_device`/`platform_driver`パターン
- [ALSAの事件簿](war-stories.md) — ASoCの`dai_link`が最終的にその上に乗っているPCM/rawmidi/USBオーディオ層の事件記録

### 外部資料 {#external}

- [ALSA SoC Layer Overview](https://docs.kernel.org/sound/soc/overview.html) — 問題提起と、コーデック/プラットフォーム/マシンの分割
- [ASoC Machine Driver](https://docs.kernel.org/sound/soc/machine.html) — マシンドライバの役割（注: その中のコード例は現在の`cpus[]`/`codecs[]`/`platforms[]`配列の形より前のものである。このページの`SND_SOC_DAILINK_DEFS`形式が現在のメインラインを反映している）
- [ASoC Codec Class Driver](https://docs.kernel.org/sound/soc/codec.html) — コーデックドライバの責務
- [ASoC Digital Audio Interface (DAI)](https://docs.kernel.org/sound/soc/dai.html) — AC97/I2S/PCMの配線プロトコル
- [Dynamic Audio Power Management for Portable Devices](https://docs.kernel.org/sound/soc/dapm.html) — ウィジェット/ルートモデルと電源ドメイン
- [Simple Audio Card devicetree binding](https://www.kernel.org/doc/Documentation/devicetree/bindings/sound/simple-card.yaml)
- [Audio Graph Card devicetree binding](https://www.kernel.org/doc/Documentation/devicetree/bindings/sound/audio-graph-card.yaml)
