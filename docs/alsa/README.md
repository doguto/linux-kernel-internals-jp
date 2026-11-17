# ALSA: Advanced Linux Sound Architecture {#alsa-the-advanced-linux-sound-architecture}

> Linuxがサウンドチップをどのように`/dev/snd/pcmC0D0p`に変えるか。すべてのサウンドカードが共有する1つのカーネルレベルのオブジェクトモデルがあり、再生、ミキシング、MIDIはそれぞれ独立に番号付けされた別々のデバイスファイルとして現れる

## はじめに {#getting-started}

オーディオハードウェアは外から見ると単純に見える。DACがあり、ミキサーのつまみがいくつかあり、MIDIポートがあるかもしれない、という程度だ。しかし汎用OSは、互いに無関係な何百ものチップ（PCI、USB、そして組み込みボードに載っている何十種類ものI2S/SoCコーデック）を1つの予測可能なユーザー空間インターフェースの背後でサポートしなければならない。同時に、1つのデバイスにつき複数の論理ストリームが存在できるようにする必要もある。再生と録音は別のデータパスであり、カードのミキサー状態は、何かが再生中かどうかとは無関係に参照・制御できなければならない。`sound/`以下にある**ALSA（Advanced Linux Sound Architecture）**が、これを担うカーネルサブシステムである。ALSAは1つの共有オブジェクトモデル、すなわち*カード*を定義し、それ以外のすべて（PCMストリーム、ミキサーコントロール、MIDIポート、タイマー）を、それぞれ独自のデバイスノード番号を持つコンポーネントデバイスとしてカードにぶら下げる。

### ALSAが解決する問題 {#the-problem-alsa-solves}

Linuxの最初のサウンドサブシステムは**OSS（Open Sound System）**であり、初期のSoundBlasterドライバの上に直接構築されていた。OSSのAPIでは、あらゆるオーディオサービス（サンプルレートやフォーマットの変換、ルーティング、複数ストリームを1つのデバイスへミックスダウンすること）をカーネルドライバ*内部*で実装する必要があった。インターフェースがユーザー空間に与えるのが、デバイスノードあたり固定フォーマットの単一ストリームだけだったからである。1998年までには、サウンドハードウェアと用途が多様化するにつれてこれが現実の制約となり、OSSの設計への不満の高まりから、Jaroslav Kyselaらが再設計されたカーネルAPIの開発に着手した。これがALSAになる。ALSAはカーネル側をハードウェア制御、バッファ管理、ミキサー状態にとどめ、フォーマット変換とミキシングのポリシーはユーザー空間/ライブラリ層に押し出した。2001年末までにALSAはLinuxの公式オーディオシステムとして採用され（[LWN: "LPC: The past, present, and future of Linux audio," Jake Edge, 2009](https://lwn.net/Articles/355542/)）、2002年初頭の2.5開発系列（2.5.4〜2.5.5）でメインラインカーネルにマージされ、Linux 2.6（2003年12月リリース）でOSSに代わる*デフォルト*のサウンドシステムとなった。古いOSS APIはその後、レガシーなバイナリのためのオプションのエミュレーション層（`snd-pcm-oss`、`snd-mixer-oss`、`snd-seq-oss`）としてのみ残された。

その結果、カーネルAPIは意図的に狭く保たれている。ハードウェアパラメータのネゴシエーション、オーディオバッファの移動、ミキサーコントロールの公開、MIDIバイトのルーティングだけである。サンプルレート変換、ストリームのミキシング、ネットワーク透過性はユーザー空間の仕事である。まずはALSA自身の`libasound`プラグインチェーンがそれを担い、現代のほとんどのデスクトップでは、その上に重ねられたオーディオサーバー（PulseAudioまたはPipeWire）が担う。

### コアオブジェクトモデル: カードとデバイス {#the-core-object-model-cards-and-devices}

どのALSAドライバの仕事も、結局は1つの`struct snd_card`を構築し、そこにコンポーネントオブジェクトを取り付けることに帰着する。`struct snd_card`（[`include/sound/core.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/sound/core.h)で定義）は、カーネルドキュメント自身の言葉を借りれば「サウンドカードの本部」である。カードの識別文字列（`id`、`driver`、`shortname`、`longname`）、取り付けられたすべてのコンポーネントの`struct list_head devices`、ミキサー/コントロールの状態（`controls`、`controls_rwsem`、`last_numid`）、カードの`/proc/asound/cardN`ツリー（`proc_root`）、そしてsysfsとドライバコアへの登録に使われる、埋め込み`struct device`である`card_dev`を所有する。

ドライバの`probe()`は決まった手順に従う。

1. **`snd_card_new()`**（またはdevres管理版の`snd_devm_card_new()`）が`snd_card`を割り当て、カードインデックスを確保する。インデックスはモジュールの`index`パラメータで要求されたもの、または`SNDRV_CARDS`（デフォルトは8、動的マイナー番号が有効な場合は`CONFIG_SND_MAX_CARDS`）未満の最初の空きスロットである。
2. ドライバは必要に応じて**コンポーネントオブジェクト**を作成する。PCM（`snd_pcm_new()`）、rawmidiポート（`snd_rawmidi_new()`）、タイマー（`snd_timer_new()`）などである。これらはそれぞれ`snd_device_new()`を通じて`struct snd_device`としてカードに自身を取り付け、`enum snd_device_type`（`SNDRV_DEV_PCM`、`SNDRV_DEV_RAWMIDI`、`SNDRV_DEV_TIMER`、...）のタグと、`dev_free`/`dev_register`/`dev_disconnect`コールバックからなる`struct snd_device_ops`を持つ。作成されたばかりのデバイスは`SNDRV_DEV_BUILD`状態から始まる。割り当て済みだが、まだユーザー空間からは到達できない状態である。ミキサーコントロールは扱いが異なる。`SNDRV_DEV_CONTROL`の`snd_device`はカードごとにちょうど1つであり、`snd_card_new()`内で`snd_ctl_create()`によって自動的に作成されるので、ドライバは何もしなくてもそれを得られる。個々の`struct snd_kcontrol`（`snd_ctl_new1()`で構築）は独自の`snd_device`を持たない。`snd_ctl_add()`（`sound/core/control.c`）は単に各コントロールをカードの`controls`リストにつなぐだけであり、単一のコントロールデバイスが1つのioctlインターフェースを通じてそれらすべてを扱う。
3. **`snd_card_register()`**がスイッチを入れる。カード自身の`card_dev`に対して`device_add()`を呼び、次に`snd_device_register_all()`を呼ぶ。後者は`card->devices`を辿って各コンポーネントの`dev_register`コールバックを呼び出し、`SNDRV_DEV_BUILD`から`SNDRV_DEV_REGISTERED`に移行させる。この呼び出しの後で初めて、カードのデバイスノードを実際に開けるようになる。カーネル自身のドライバ作成ガイドによれば、「`snd_card_register()`が呼ばれる前は、コンポーネントは外部から安全にアクセスできない状態にある」（[docs.kernel.org: Writing an ALSA Driver](https://docs.kernel.org/sound/kernel-api/writing-an-alsa-driver.html)）。

解体はその鏡像である。`snd_card_free()`（devres管理のカードでは、登録時に追加される自動の`devm`アクション）が、カード自体を解放する前に、取り付けられたすべての`snd_device`を切断して解放する。そのため、ドライバがいくつのPCM、コントロール、MIDIポートを作成していても、1回の呼び出しでコンポーネントツリー全体を巻き戻せる。

```
  Hardware (PCI, USB, or an ASoC I2S/SoC codec)
          │
          ▼
  Low-level driver's probe()      e.g. snd_hda_intel, snd_usb_audio,
          │                          or an ASoC machine driver
          ▼
  snd_card_new() / snd_devm_card_new()
          │  allocates struct snd_card, claims a card index (card->number)
          │  snd_ctl_create() → the one SNDRV_DEV_CONTROL snd_device for this card
          │
          ├── snd_pcm_new()      → SNDRV_DEV_PCM      (playback/capture streams)
          ├── snd_rawmidi_new()  → SNDRV_DEV_RAWMIDI   (MIDI byte stream)
          └── snd_timer_new()    → SNDRV_DEV_TIMER     (sample-accurate scheduling)
          │      each wraps itself in a struct snd_device, state = SNDRV_DEV_BUILD
          │      (linked onto card->devices)
          │
          snd_ctl_new1() + snd_ctl_add()  → struct snd_kcontrol linked onto card->controls
          │      (no snd_device of its own; served by the one SNDRV_DEV_CONTROL device above)
          ▼
  snd_card_register()
          │  device_add(&card->card_dev)
          │  snd_device_register_all(card)  → each snd_device: BUILD → REGISTERED
          ▼
  Device nodes go live under /dev/snd/*  and  /proc/asound/cardN/*
```

### ALSAが公開する4つのサブシステム {#the-four-subsystems-alsa-exposes}

どのALSAカードも、最大4種類のコンポーネントから構成される。それぞれが独自のカーネルデータ構造、マイナー番号の範囲、デバイスノードの命名規則を持つ。

- **PCM（Pulse-Code Modulation）** — 再生と録音のストリーム。`struct snd_pcm`（`sound/core/pcm.c`、`include/sound/pcm.h`）は最大2つの`struct snd_pcm_str`を保持する。方向（`SNDRV_PCM_STREAM_PLAYBACK` / `_CAPTURE`）ごとに1つであり、それぞれが複数substreamを持つハードウェアのために`struct snd_pcm_substream`のリストを所有する。ハードウェアパラメータのネゴシエーション（`hw_params`）、リングバッファのレイアウト、割り込み駆動の位置更新はここにある。専用の詳細解説は[PCM: 再生と録音](pcm.md)にある。
- **Control** — ミキサー。`struct snd_kcontrol`（`include/sound/control.h`）は、名前/インデックス/get/put/infoからなる汎用の組である。音量スライダー、録音ソースのスイッチ、S/PDIFステータスビット、そしてオーディオ以外の設定（カード内部のルーティングなど）は、すべて`info`/`get`/`put`コールバックが異なるだけのkcontrolである。コントロールデバイス（`controlC*`）はカードごとにちょうど1つで、カード自体と一緒に自動的に作成される。あらゆるミキサーアプリケーション（`amixer`、`alsamixer`、PulseAudio/PipeWireのミキサーバックエンド）は、同じioctlインターフェースを通じてそれとやり取りする。
- **MIDI** — 2つの層に分かれる。**Rawmidi**（`struct snd_rawmidi`、`include/sound/rawmidi.h`）は、ハードウェアのMIDIポートやUSB MIDIアダプタが直接公開する低レベルのバイトストリームインターフェースである。その上にある**ALSAシーケンサ**（`include/sound/seq_kernel.h`、`sound/core/seq/`）は、独自のクライアント/ポート/キューモデルを持つカーネル空間のMIDIイベントルーターである。これにより、シンセサイザー、ハードウェアポート、ユーザー空間のシーケンサアプリケーション（DAW、`aconnect`）が、エンドポイントのあらゆる組み合わせを直接接続しなくても、MIDIイベントストリームを互いにつなぎ合わせられる。
- **Timer** — `struct snd_timer`（`include/sound/timer.h`）はALSAのサンプル精度のスケジューリング基盤である。内部的にはPCMのperiod経過通知やシーケンサのイベントタイミングの駆動に使われ、ハードウェアに同期したウェイクアップを必要とするアプリケーション向けにユーザー空間（`/dev/snd/timer`）にも公開されている。

一般消費者向けのオーディオハードウェアのほとんどは、PCMとControlしか持たない。MIDIとTimerのデバイスは、実際にMIDIポートやタイマー機能を持つハードウェアのカードにのみ現れる。SoC/組み込みオーディオでも、ユーザー空間に公開されるのは上記の4つのサブシステムだが、ドライバ側は通常**ASoC（ALSA System on Chip）**に対して書かれる。ASoCはドライバを、再利用可能な*コーデック*ドライバ、*プラットフォーム*/DMAドライバ、そして特定のボード向けにその2つを結び付ける小さな*マシン*ドライバに分割する層であり、[ASoC](asoc.md)で扱う。

### ユーザー空間に向けたデバイスノード {#userspace-facing-device-nodes}

登録されたすべてのコンポーネントは`/dev/snd/`以下のキャラクタデバイスとして現れる。これらはすべてメジャー番号116（`CONFIG_SND_MAJOR`）を共有し、カードとカード内のデバイスはマイナー番号にエンコードされる（[`include/sound/minors.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/sound/minors.h)の`SNDRV_MINOR_CARD()`/`SNDRV_MINOR_DEVICE()`）。この固定の`card << 5 | device`というマイナー番号レイアウトはデフォルトのものであり、`CONFIG_SND_DYNAMIC_MINORS`を有効にすると、マイナー番号は平坦な範囲からオンデマンドで払い出され、このようには分解できなくなる。命名は各サブシステムの登録パスで`dev_set_name()`によって設定され、決まったパターンに従う。

| ノード | 意味 | 設定箇所 |
|---|---|---|
| `controlC0` | カード0のミキサー/コントロールデバイス | `sound/core/control.c` |
| `pcmC0D0p` | カード0、PCMデバイス0、**p**layback（再生） | `sound/core/pcm.c` |
| `pcmC0D0c` | カード0、PCMデバイス0、**c**apture（録音） | `sound/core/pcm.c` |
| `midiC0D0` | カード0、rawmidiデバイス0 | `sound/core/rawmidi.c` |
| `seq` | シーケンサ。全カードで共有される1つのノード | `sound/core/seq/` |
| `timer` | タイマーデバイス。同じく共有される | `sound/core/timer.c` |

2つの独立したオーディオパス（たとえばアナログ出力とHDMI出力）を持つカードは、`pcmC0D0p`と`pcmC0D1p`として現れる。同じ*カード*上の異なるPCM*デバイス*であって、異なるカードではない。

これらのノードと直接やり取りするのは、`libasound`（ALSAのユーザー空間ライブラリ）とオーディオサーバー以外にはほとんどない。現代のほとんどのディストリビューションでは、アプリケーションは**PipeWire**（古い/単純な構成では**PulseAudio**）を経由する。これらはデバイスを排他的に所有し、代わりにミックス済みでフォーマット変換済み、ネットワーク透過なストリームをアプリケーションに提供する。PipeWire自身のALSA互換機能は、いまだにALSAデバイスを直接開く（減りつつある）アプリケーションのために`libasound`呼び出しを横取りするプラグインである。PipeWireはさらに、PulseAudio互換のソケットとJACK互換のクライアントライブラリもそれぞれのエコシステム向けに実装している。それらすべての下で、実際の最下層であり続けるのはカーネルのALSAノードである（[LWN: "PipeWire: The Linux audio/video bus," Ahmed S. Darwish, 2021](https://lwn.net/Articles/847412/)）。このページはその境界で止まる。PulseAudio/PipeWireの内部はカーネルではなくユーザー空間のものであり、ここでは扱わない。

### カードとデバイスの番号付け、そして`/proc/asound` {#card-and-device-numbering-and-procasound}

カードインデックスは`snd_card_new()`の時点（`sound/core/init.c`の`snd_card_init()`）で割り当てられる。ドライバは`index`モジュールパラメータで特定のインデックスを要求できる。設定されていない場合、カーネルはまずグローバルな`slots=`モジュールパラメータで、そのドライバのモジュール名用に予約されたスロットがないかを確認し（`module_slot_match()`）、それもなければ最初の空きスロットにフォールバックする。上限は`SNDRV_CARDS`枚である。このインデックス（`card->number`）が、`card0`、`controlC0`、`pcmC0D0p`といったすべてのデバイス名に現れる。そして、何か（`index=`を固定するモジュールの`options`行、`slots=`配列、udev/systemdの永続命名ルール）がそれを固定しない限り、再起動をまたいでprobe順と一致することは*保証されない*。

`/proc/asound/`は、現在のカード/デバイスの状態を、特別なツールなしで調べられるように映し出している。

- **`/proc/asound/cards`**は、登録されたすべてのカードを、カードごとに1行（加えてドライバ/長い名前の継続行）で一覧表示する。形式は`sound/core/init.c`の`snd_card_info_read()`が書き出すとおり、`" N [id             ]: driver - shortname"`の次の行に`longname`が続く。
- **`/proc/asound/cardN/`**はそのカード自身のツリー（`id`、`pcm0p/`、`pcm0c/`など）を保持し、ハードウェアパラメータ、バッファ状態、ドライバ固有のデバッグ情報をプレーンテキストのファイルとして公開している。ドライバに計装を入れずに、ドライバの`hw_params`や`codec`の状態が実際にどうなっているかを確認するには、これが最速の方法であることが多い。

### このセクションの残り {#the-rest-of-this-section}

以降のページでは、上で概説した各要素をさらに掘り下げる。

- **[PCM: 再生と録音](pcm.md)** — ハードウェアパラメータのネゴシエーション、リングバッファ、割り込み駆動のperiod更新、そしてユーザー空間の`write()`/`mmap()`呼び出しがどのようにDMAになるか。
- **[ASoC](asoc.md)** — ほぼすべての組み込みおよびSoCオーディオで使われるコーデック/プラットフォーム/マシンのドライバ分割と、DAPMがオーディオパス内の電源ドメインをどのように管理するか。
- **[事件簿](war-stories.md)** — ALSAで実際に起きた6件の事件。ほぼ21年間潜伏していたUSBミキサーのuse-after-freeから、PCMバッファのロック競合まで。

### 前提知識と関連分野 {#prerequisites-and-neighbors}

ALSAドライバは、ごく普通の[Linuxデバイスモデル](../drivers/device-model.md)の住人である。デスクトップ/サーバーのサウンドカードの大半は[PCIデバイス](../drivers/pci-driver.md)かUSBオーディオクラスのデバイスであり、組み込みオーディオはほぼ常にASoCが組み立てる[プラットフォームデバイス](../drivers/platform-driver.md)のツリーである。`/dev/snd/*`ノード自体は、ほかのドライバと同じ`cdev`/`file_operations`の仕組みで登録される、普通の[キャラクタデバイス](../drivers/chardev.md)である。読む順番: このページ、次にデータパスについて[PCM](pcm.md)、組み込み/SoCオーディオに取り組んでいるなら続いて[ASoC](asoc.md)。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/sound/core.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/sound/core.h) — `struct snd_card`、`struct snd_device`、およびカード/デバイス登録API
- [sound/core/init.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/sound/core/init.c) — `snd_card_new()`、`snd_card_register()`、カードインデックスの割り当て、`/proc/asound/cards`
- [sound/core/device.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/sound/core/device.c) — `snd_device_new()`/`snd_device_register_all()`と、`SNDRV_DEV_BUILD` → `SNDRV_DEV_REGISTERED`の状態機械
- [include/sound/pcm.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/sound/pcm.h) · [sound/core/pcm.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/sound/core/pcm.c) — `struct snd_pcm`、substream、`pcmC%iD%i%c`のデバイス命名
- [include/sound/control.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/sound/control.h) · [sound/core/control.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/sound/core/control.c) — `struct snd_kcontrol`とミキサーのコントロールデバイス
- [include/sound/rawmidi.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/sound/rawmidi.h) — `struct snd_rawmidi`
- [include/sound/timer.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/sound/timer.h) — `struct snd_timer`
- [include/sound/minors.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/sound/minors.h) — `/dev/snd/*`のマイナー番号レイアウト

### 関連ページ {#related-pages}

- [PCM: 再生と録音](pcm.md) — データパスの詳細
- [ASoC](asoc.md) — 組み込みオーディオ向けのコーデック/プラットフォーム/マシンドライバ
- [事件簿](war-stories.md) — 上記サブシステムで実際に起きた事件
- [Linuxデバイスモデル](../drivers/device-model.md) · [PCIドライバ](../drivers/pci-driver.md) · [プラットフォームドライバ](../drivers/platform-driver.md) · [キャラクタデバイス](../drivers/chardev.md) — ALSAが乗っているドライバコアの各層

### LWN記事 {#lwn-articles}

- [LPC: The past, present, and future of Linux audio](https://lwn.net/Articles/355542/) — ALSAがどのように、そしてなぜOSSに代わってLinuxのデフォルトのサウンドシステムになったか
- [PipeWire: The Linux audio/video bus](https://lwn.net/Articles/847412/) — ALSA、PulseAudio、JACKに対してPipeWireがどこに位置するか

### 外部資料 {#external}

- [Kernel docs: Writing an ALSA Driver](https://docs.kernel.org/sound/kernel-api/writing-an-alsa-driver.html) — カード/デバイスモデルについての、ドライバ作者向けの標準的なガイド
- [Kernel docs: ALSA configuration guide](https://docs.kernel.org/sound/alsa-configuration.html) — モジュールパラメータ、デバイスファイルの対応、`/proc/asound`のレイアウト
- [Kernel docs: OSS emulation](https://docs.kernel.org/sound/designs/oss-emulation.html) — レガシーなOSS APIが現在ALSAの上でどのようにエミュレートされているか
- [Kernel docs: Sound subsystem documentation index](https://docs.kernel.org/sound/index.html) — ALSA SoCとHD-Audioのセクションを含む、最上位の索引
