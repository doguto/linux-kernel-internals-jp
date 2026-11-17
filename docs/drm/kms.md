# カーネルモード設定（KMS）: オブジェクトモデルとアトミックモード設定 {#kernel-mode-setting-kms-the-object-model-and-atomic-modesetting}

> ディスプレイパイプラインをモデル化する4つのオブジェクトと、それらすべてをまとめて変更するか、まったく変更しないかのどちらかを行う1つのioctl

## 問題: ディスプレイパイプラインをカーネル内でどう記述するか {#the-problem-describing-a-display-pipeline-in-the-kernel}

フレームバッファを画面上の光子に変えるのは1つの操作ではなく、操作の連鎖である。ピクセルデータの矩形を選び、拡大縮小して配置し、画面を共有する他のレイヤーとブレンドし、その結果をケーブルやパネルが理解できる信号に変換し、ディスプレイがゴミを表示しないよう正しいタイミングでその信号を物理ポートから送り出す。[KMS（Kernel Mode Setting）](README.md)は、この連鎖のDRMにおけるモデルである。各段階をカーネルオブジェクト — `drm_plane`、`drm_crtc`、`drm_encoder`、`drm_connector` — として表現し、それらをディスプレイごとのパイプラインとして結び付け、ユーザー空間には1つのioctl `DRM_IOCTL_MODE_ATOMIC` を提供する。これにより、パイプライン構成全体を、ハードウェアに一切触れる前に提案して検証できる。

KMS自体は2008年12月にさかのぼる。このとき [`f453ba046074`](https://git.kernel.org/linus/f453ba0460742ad027ae0c4c7d61e62817b3e7ef) がモード設定をXからカーネルへと移した（その歴史については[DRMの概要](README.md)を参照）。このページが扱うのはその後のことである。KMSが導入したオブジェクトモデルと、6年後にKMS本来の「一度に1プロパティずつ」のインターフェースを置き換えたアトミックioctlである。

## 4つのオブジェクト {#the-four-objects}

### `drm_plane` — ハードウェア合成レイヤー {#drm_plane-a-hardware-compositing-layer}

planeは、ディスプレイハードウェアがスキャンアウトして他のplaneとブレンドできるピクセルデータの1レイヤーである。すべてディスプレイコントローラに内蔵された固定機能のコンポジタで行われ、GPUの3Dエンジンは関与しない。`include/drm/drm_plane.h` は3つの種類を定義している。

```c
enum drm_plane_type {
	DRM_PLANE_TYPE_OVERLAY,  /* プライマリでもカーソルでもない任意のplane（"sprites"） */
	DRM_PLANE_TYPE_PRIMARY,  /* 単独でCRTCを点灯させる可能性が最も高いplane */
	DRM_PLANE_TYPE_CURSOR,   /* drm_mode_config.cursor_{width,height} に合わせたサイズの小さなplane */
};
```

`struct drm_plane` はハードウェアに依存しない状態 — `format_types`（受け付けるピクセルフォーマット）、`possible_crtcs`（どのCRTCに供給できるか）、現在バインドされている `fb`/`crtc` — を保持する。コミットごとのジオメトリは `struct drm_plane_state` にある。

```c
// include/drm/drm_plane.h
struct drm_plane_state {
	struct drm_plane   *plane;
	struct drm_crtc     *crtc;
	struct drm_framebuffer *fb;   // このplaneがスキャンアウトしているバッファ
	...
	int32_t  crtc_x,  crtc_y;     // CRTC上の位置、整数ピクセル
	uint32_t crtc_w,  crtc_h;     // CRTC上のサイズ、整数ピクセル
	uint32_t src_x,   src_y;      // ソースfb内の位置、16.16固定小数点
	uint32_t src_w,   src_h;      // サンプリングされる領域のサイズ、16.16固定小数点
	unsigned int rotation;
	unsigned int zpos;            // 他のplaneに対するブレンド順序
	struct drm_rect src, dst;     // 導出された整数矩形
	bool visible;
	...
};
```

`src_*`（固定小数点で、フレームバッファ自身の座標空間にある — これによりplaneはクロップや拡大縮小ができる）と `crtc_*`（整数で、出力の座標空間にある — 結果が配置される場所）の区別こそが、planeを単なるバッファへのポインタではなく、本物の合成プリミティブにしている。

マルチplaneハードウェアが重要なのは、ディスプレイコントローラが合成を無償で行うか、GPUが合成結果を描画しなければならないかの違いになるからである。ウィンドウ内で4K映画を表示する動画プレーヤーは、3Dエンジンに一切触れる必要がない。バッファにデコードし、そのバッファをウィンドウの上にサイズと位置を合わせたoverlay planeに直接渡せば、ディスプレイコントローラがそれをデスクトップのprimary planeとハードウェアで、走査線ごとに、GPUコストゼロかつ余分なメモリコピーなしでブレンドする。overlay planeが失われると — ヘッドレスなSoCディスプレイや、空いているplaneを見つけられないコンポジタの場合 — 同じ動画を毎フレームGPUがデスクトップのフレームバッファに合成しなければならなくなる。

### `drm_crtc` — スキャンアウトエンジン {#drm_crtc-the-scanout-engine}

CRTC（名前はCRT「Cathode Ray Tube Controller」ハードウェアの名残である）は、合成された画像を `drm_display_mode` に合わせたタイミングで、1行ずつリアルタイムにencoderへ送り出すエンジンである。`struct drm_crtc` は2つの組み込みplaneポインタを持つ。すべてのCRTCは少なくとも `primary` planeを持ち、通常は `cursor` planeも持つ。

```c
// include/drm/drm_crtc.h
struct drm_crtc {
	...
	struct drm_plane *primary;
	struct drm_plane *cursor;
	...
	struct drm_display_mode mode;    // 現在の、レガシーパスのタイミング
	struct drm_display_mode hwmode;  // 調整後に実際にプログラムされた値
	...
	struct drm_crtc_state *state;    // 現在のアトミック状態
};
```

モード自体である `struct drm_display_mode`（`include/drm/drm_modes.h`）は、CRT由来の完全なタイミング記述である。kHz単位のピクセル `clock` と、有効ピクセル、同期パルス、ブランキング期間を定義する水平／垂直の `display`/`sync_start`/`sync_end`/`total` の値からなる。アトミックコミットが何が変わったかを実際に記録する場所は `struct drm_crtc_state` である。

```c
// include/drm/drm_crtc.h
struct drm_crtc_state {
	struct drm_crtc *crtc;
	bool enable;
	bool active;
	bool planes_changed     : 1;
	bool mode_changed       : 1;   // @mode または @enable が変化した — 本物のモード設定が必要
	bool active_changed     : 1;
	bool connectors_changed : 1;
	...
	u32 plane_mask, connector_mask, encoder_mask;
	struct drm_display_mode mode, adjusted_mode;
	...
};
```

`mode_changed` は `active_changed` および `connectors_changed` とともに `drm_atomic_crtc_needs_modeset()` によって組み合わされ、アトミックコアはその結果を `DRM_MODE_ATOMIC_ALLOW_MODESET` と照合する。これについては後述する。

### `drm_encoder` — 信号変換、次第に形式的な存在に {#drm_encoder-signal-conversion-increasingly-a-formality}

`include/drm/drm_encoder.h` は、encoderの役割を1行で説明している。"CRTCs drive pixels to encoders, which convert them into signals appropriate for a given connector."（訳: CRTCはピクセルをencoderに送り、encoderはそれを特定のconnectorに適した信号に変換する。）`encoder_type` は生成するフォーマットを示す — DVI/HDMI/eDP用の `DRM_MODE_ENCODER_TMDS`、VGA用の `DRM_MODE_ENCODER_DAC`、パネル用の `DRM_MODE_ENCODER_DSI`/`DPI`、VMディスプレイ用の `DRM_MODE_ENCODER_VIRTUAL`。`possible_crtcs`/`possible_clones` は登録時に固定されるビットマスクで、どのCRTCがこのencoderを駆動できるか、どの兄弟encoderと並んでクローン出力できるかを記述する。

KMSが設計された時代の実ハードウェアでは、encoderは独立したブロックだった。TMDSトランスミッタチップ、つまりスキャンアウトエンジンの下流で信号変換を行う別個のシリコンブロックである。現代のSoCや統合GPUでは、そのブロックはCRTCと同じハードウェアに統合されているか、固定のマルチプレクスに縮小されていることが多く、カーネル自身のヘッダも今ではそう述べている。`drm_encoder.crtc` は "only really meaningful for non-atomic drivers. Atomic drivers should instead check `drm_connector_state.crtc`"（訳: 非アトミックドライバでのみ実質的に意味がある。アトミックドライバは代わりに `drm_connector_state.crtc` を確認すべきである）と文書化されている。アトミックドライバは、どのCRTCが自分を駆動しているかを*connector*に問い合わせるよう誘導されており、encoderは推論すべき実体のあるオブジェクトではなく、管理上の記録として扱われる。単純な現代のドライバの多くは、純粋にオブジェクトモデルを満たすためだけに、connectorごとにちょうど1つのencoderを生成している。

### `drm_connector` — 物理出力とホットプラグ {#drm_connector-the-physical-output-and-hotplug}

connectorは、マシンの背面で指し示せるものに結び付いた唯一のオブジェクトである。HDMIジャック、DisplayPortコネクタ、eDPパネルの固定内部リンクなどだ。connectorは接続の可変状態を所有する。今何かが接続されているか、接続されているものがどのモードをサポートしているか、である。

```c
// include/drm/drm_connector.h
struct drm_connector {
	...
	int connector_type;                 // DRM_MODE_CONNECTOR_HDMIA, ...
	enum drm_connector_status status;
	struct list_head modes;             // drm_display_mode のリスト、EDID由来
	struct drm_display_info display_info;
	const struct drm_connector_funcs *funcs;
	const struct drm_connector_helper_funcs *helper_private;
	u32 possible_encoders;
	struct drm_encoder *encoder;
	...
	struct drm_connector_state *state;
};
```

検出はコアのロジックではなく、ドライバのコールバックである。`struct drm_connector_funcs` と、`include/drm/drm_modeset_helper_vtables.h` にあるそのアトミック対応版がそれにあたる。

```c
// drm_connector_funcs
enum drm_connector_status (*detect)(struct drm_connector *connector, bool force);

// drm_connector_helper_funcs — modesetロックを意識した版
int (*detect_ctx)(struct drm_connector *connector,
		   struct drm_modeset_acquire_ctx *ctx, bool force);
```

`force` は、これが安価な定期ポーリングなのか（falseのままにし、高コストで破壊的なプローブは省略する）、それともユーザーが開始した再確認であり、実際の、場合によっては妨げになる検出サイクルを行ってよいのかをドライバに伝える。ホットプラグイベント — HDMIにモニタが接続された、DPドックが接続された — は割り込みを発生させ、ドライバはそれを `drm_kms_helper_hotplug_event()` の呼び出しに変える。これがconnectorの再プローブをスケジュールする。ユーザー空間（DRMのfdで待ち受けるコンポジタ、あるいはudev経由）は、その結果のueventを受け取り、`DRM_IOCTL_MODE_GETCONNECTOR` でconnectorの状態を読み直す。

何かが検出されると、`helper_private->get_modes()` がconnectorのモードリストを埋める。通常はI2C/DDCまたはAUXチャネル経由でEDIDを読み、`drm_add_edid_modes()`/`drm_helper_probe_add_cmdline_mode()` のようなヘルパーで `drm_display_mode` のエントリに解析する。4Kパネルのネイティブタイミングや、モニタがサポートするリフレッシュレートは、実際にはここから来ている。

### パイプラインと、フレームバッファが接続される場所 {#the-pipeline-and-where-the-framebuffer-attaches}

```
  drm_framebuffer                                                   physical
  (GEM buffer +          drm_plane(s)      drm_crtc      drm_encoder    display
   format/pitch)   ──►   (composite)  ──►  (scanout,  ──►  (signal  ──► drm_connector
                         primary/            timing)        convert)    (HDMI/eDP/…)
                         cursor/overlay
```

`drm_framebuffer` はGEMバッファオブジェクトの上のメタデータ — ピクセル `format`、planeごとの `pitches`/`offsets`、`width`/`height` — であり、それ自体は特定のplaneに属さない。コミットの一部として `drm_plane_state.fb` に割り当てることで接続される。1つ以上のplaneがCRTCに供給され、CRTCはそれらを合成された形で読み込み、`drm_display_mode` のタイミングを適用し、結果をバインドされたencoderへ送る（直接、あるいはSoC内部のMIPI-eDP変換器のようなものについては `drm_bridge` チェーンを経由して）。encoderはconnectorの信号フォーマットに変換し、connectorがディスプレイへの配線となる。4種類のオブジェクトすべてとフレームバッファはmodesetオブジェクト（`struct drm_mode_object`）であり、plane、CRTC、connectorはそれぞれ独自のプロパティセットを持つ。それこそが、これらを1つずつ変更することをあれほど危険にしていた理由であり、次に扱う。

## なぜレガシーAPIは廃止されなければならなかったか {#why-the-legacy-api-had-to-go}

2008年の最初のKMSインターフェースは、X自身のXRandR時代のモデルを忠実に模していた。CRTCをモード、connector、フレームバッファに1回の呼び出しでバインドする `DRM_IOCTL_MODE_SETCRTC`（`drm_mode_setcrtc()`）があり、必要が増えるにつれて別個のioctlが後から追加された。planeの更新用の `DRM_IOCTL_MODE_SETPLANE`、そしてプロパティが導入されてからは、1回の呼び出しで1つのオブジェクトの1つのプロパティを変更する `DRM_IOCTL_MODE_OBJ_SETPROPERTY`（`drm_mode_obj_set_property_ioctl()`）である。これらのioctlエントリはすべて、レガシーなユーザー空間のために、今日でも `drivers/gpu/drm/drm_ioctl.c` に残っている。

障害モードは偶発的なものではなく構造的なものだった。ホットプラグに反応するコンポジタ — たとえばノートPCの外部モニタが現れ、両方を同期させるために内蔵パネルのモード変更を強いられる場合 — は、一連の別々のioctlを発行しなければならなかった。あるCRTCを無効化し、別のCRTCに新しいモードを設定し、planeの向き先を変え、再び有効化する。別のプロセス（あるいは次のvblankでの同じコンポジタ）がシーケンスの途中でディスプレイを観測することを妨げるものは何もなく、手順3が実ハードウェアで無効だと判明した場合にシーケンス*全体*を事前に拒否する手段もカーネルにはなかった。シーケンスの途中でエラーを受け取り、画面がすでに中途半端に更新された状態で残されて初めてそれがわかるのである。新しい構成の一部を実際に適用せずに「この構成はそもそも動作するか」を問い合わせる方法はなかった。

修正は、2014年にかけて2人のDRM開発者によって提案され、推進された。Rob Clarkは2014年5月、ロックと状態追跡のインフラを17パッチのシリーズ ["prepare for atomic/nuclear modeset/pageflip"](https://lwn.net/Articles/600369/) としてdri-develに投稿した。これは明示的にインフラのみで、まだioctl自体ではなかった。その後Daniel Vetterがコアの状態追跡オブジェクトを作り直し、[`cc4ceb484b37`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=cc4ceb484b37b9369e0d4e8682b7ae1849ae4579)（"drm: Global atomic state handling"、2014年7月）として取り込んだ。そのコミットメッセージは、これを "Rob's patches" からの分岐であり後継であると明示的に述べている。そのシリーズに対するVetterの[カバーレター](https://lore.kernel.org/dri-devel/1406497308-30733-1-git-send-email-daniel.vetter@ffwll.ch/)は、分岐した理由を説明している。ソフトウェア側の状態更新を同期的にすることで "all the tricky locking dances can be abolished"（訳: 厄介なロックのダンスをすべて廃止できる）こと、そしてチェックのみの操作が前の更新の完了を待ってブロックする必要がないこと。なぜなら "compositors need to do [this] right after having completed the current frame"（訳: コンポジタは現在のフレームを完了した直後に［これを］行う必要がある）からである。Rob Clarkは同じスレッドで自身の設計を擁護した。彼の参照カウント付き状態は、状態を連鎖させることでチェックのみのパスがロックをまったく必要としないようにする意図であり、"in cases where drivers have to fwd one property to another object or other sort of side effect, I think my way was nicer, imho ;-)"（訳: ドライバがあるプロパティを別のオブジェクトに転送しなければならない場合や、その他の副作用がある場合には、私のやり方のほうが良かったと思う、私見だけど ;-)）と述べた。出荷されたのはVetterの設計である。

その後Rob Clarkは、ユーザー空間向けのエントリポイントそのものを [`d34f20d6e2f2`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d34f20d6e2f21bd3531b969dc40913181a8ae31a)（"drm: Atomic modeset ioctl"、2014年12月、Daniel VetterとSean Paulがレビュー）で取り込んだ。これは `DRM_IOCTL_MODE_ATOMIC` を追加し、レガシーなユーザー空間が偶然これに踏み込まないよう `DRM_CLIENT_CAP_ATOMIC` ケーパビリティフラグの背後に置いた。uAPI自体が議論されたのは、マージ版の[2日前のRFC](https://lore.kernel.org/dri-devel/1418771141-16954-1-git-send-email-robdclark@gmail.com/)である。Michel Dänzerは、変更が有効になる*時点*を示す明示的なパラメータをioctlが取るべきだと主張し、さらに可変リフレッシュレートのディスプレイを見越して、それはフレームカウンタではなくタイムスタンプであるべきだとした。Pekka Paalanenは、Wayland自身のPresentation拡張の経験に基づき、そのようなキューイングモデルがもたらす複雑さに反論した。1時間先にスケジュールされたコミットをどうやってキャンセルするのか、と。出荷されたioctlはどちらも採用しなかった。コミットは同期的か、ノンブロッキングかつ即時のどちらかであり、将来のコミットをスケジュールするという概念は一切ない。このインターフェースと、それに依存できるだけの数の移行済みドライバは、2015年半ばのLinux 4.2で出荷された。LWNの2部構成の設計解説、[パート1](https://lwn.net/Articles/653071/)と[パート2](https://lwn.net/Articles/653466/)（Daniel Vetter、2015年8月）は、推進力となった動機を、Wayland時代のコンポジタが「すべてのフレームが完璧である」セマンティクスを必要としたこと、そして電力に敏感なモバイル／組み込みハードウェアが、1プロパティずつレジスタに触れるのではなく状態変更をまとめる必要があったことだと位置付けている。

## アトミックコミットは実際にどう動くか {#how-an-atomic-commit-actually-works}

### コンテナ: `drm_atomic_state` → `drm_atomic_commit` {#the-container-drmatomicstate-drmatomiccommit}

アトミックコアは、何かに触れる前に、提案された構成を1つのコンテナオブジェクトにまとめ上げる。2014年からLinux 7.1まで、この構造体は `struct drm_atomic_state` という名前だった。上で挙げたLWNの記事や、現在も広く展開されているドライバコードの大半で目にするのはこの名前である。2026年4月、[`5164f7e7ff8e`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5164f7e7ff8ec7d41065d3862630c2ba09854328)（"drm: Rename struct drm_atomic_state to drm_atomic_commit"、Maxime Ripard）によって `struct drm_atomic_commit` に改名された。コミットメッセージが挙げる理由は、`drm_atomic_state` が*オブジェクトごと*の状態構造体（`drm_plane_state`、`drm_crtc_state` など）と混同されやすいが、実際にはデバイスの「状態」ではなく、新旧のオブジェクトごとの状態を参照する進行中のコミットだから、というものである。このページでは現在の名前を使う。2026年以前のカーネルソースやドキュメントを読む場合は、頭の中で `drm_atomic_state` に置き換えてほしい。同じオブジェクトである。

```c
// include/drm/drm_atomic.h（現在のmaster）
struct drm_atomic_commit {
	struct kref ref;
	struct drm_device *dev;

	bool allow_modeset       : 1;   // DRM_MODE_ATOMIC_ALLOW_MODESET が設定されていた
	bool legacy_cursor_update : 1;
	bool async_update        : 1;
	bool duplicated          : 1;
	bool checked             : 1;   // ->atomic_check が実行済み

	struct __drm_planes_state     *planes;
	struct __drm_crtcs_state      *crtcs;
	int num_connector;
	struct __drm_connnectors_state *connectors;
	int num_private_objs;
	struct __drm_private_objs_state *private_objs;

	struct drm_modeset_acquire_ctx *acquire_ctx;
	struct drm_crtc_commit *fake_commit;
	struct work_struct commit_work;
	...
};
```

オブジェクトごとの配列の各エントリは、そのオブジェクトの古い状態（現在有効なもの）と提案された新しい状態の両方を保持する。これらは `drm_atomic_get_crtc_state()` / `drm_atomic_get_plane_state()` / `drm_atomic_get_connector_state()` によって取得される。これらのいずれかを呼び出すことで、オブジェクトがコミットに取り込まれ、ドライバが変更できるようにその現在の状態が複製される。

### チェックとコミットの分離 {#the-checkcommit-split}

2つのエントリポイントがきれいに分離されている。

- **`drm_atomic_check_only(struct drm_atomic_commit *state)`**（`drivers/gpu/drm/drm_atomic.c`）— 提案された状態を `config->funcs->atomic_check` に照らして検証する。さらに、いずれかのCRTCが完全なモード設定を必要とする（`drm_atomic_crtc_needs_modeset()`: `mode_changed`、`active_changed`、`connectors_changed` のいずれか）にもかかわらず `state->allow_modeset` がfalseであれば、`-EINVAL` でコミットを拒否する。ハードウェアには一切触れない。`atomic_check` を実装するドライバは、通常、最初のステップとして汎用の **`drm_atomic_helper_check(struct drm_device *dev, struct drm_atomic_commit *state)`**（`drivers/gpu/drm/drm_atomic_helper.c`）を呼ぶ。これは、ハードウェア固有のチェックの前に、plane/CRTC/connectorにまたがる共通の相互検証（有効なすべてのCRTCがモードと少なくとも1つのconnectorを持つか、planeのジオメトリが収まるか、など）を実行する。
- **`drm_atomic_commit(struct drm_atomic_commit *state)`** / **`drm_atomic_nonblocking_commit(struct drm_atomic_commit *state)`** — どちらも内部で最初にチェックを呼ぶと文書化されており（元の設計コミットによれば "checking will be internally enforced by always calling `->atomic_check` before `->atomic_commit`"、訳: `->atomic_commit` の前に常に `->atomic_check` を呼ぶことで、チェックは内部的に強制される）、その後で状態をハードウェアに適用する。どちらも、ww-mutexのロック（`state->acquire_ctx`、`drm_modeset_acquire_ctx`）がバックオフのケースに当たった場合は `-EDEADLK` を返しうる。呼び出し側はロックを解放して再試行することが期待されており、ioctlハンドラはまさにそれを `retry:` ループで行っている。

### uAPIフラグ {#uapi-flags}

ioctlの構造体とそのフラグは、最初の `d34f20d6e2f2` 以来 `include/uapi/drm/drm_mode.h` で定義されており、それ以来変わっていない。

```c
#define DRM_MODE_ATOMIC_TEST_ONLY      0x0100
#define DRM_MODE_ATOMIC_NONBLOCK       0x0200
#define DRM_MODE_ATOMIC_ALLOW_MODESET  0x0400

struct drm_mode_atomic {
	__u32 flags;
	__u32 count_objs;
	__u64 objs_ptr;           // オブジェクトIDの配列
	__u64 count_props_ptr;    // オブジェクトごとのプロパティ数の配列
	__u64 props_ptr;          // プロパティIDのフラットな配列
	__u64 prop_values_ptr;    // プロパティ値のフラットな配列
	__u64 reserved;
	__u64 user_data;          // 完了イベントでそのまま返される
};
```

- **`DRM_MODE_ATOMIC_TEST_ONLY`** — `drm_atomic_check_only()` を実行してその結果を返す。コミットは決して呼ばない。これはレガシーAPIに相当するものがなかった「これは動作するか」のドライランである。コンポジタは、モード／plane／プロパティの変更の任意の組み合わせを試し、目に見える副作用ゼロで確定的なyes/noを得られる。
- **`DRM_MODE_ATOMIC_ALLOW_MODESET`** — これがないと、本物のモード設定を必要とする変更（`drm_atomic_crtc_needs_modeset()`）はすべて拒否される。コンポジタはこれを実際のモード変更のときだけ設定し、フレームごとの定常的なplane／プロパティの更新では設定しない。これにより、バグのあるコミットが誤って完全で目に見えるモード設定を強制することがなくなる。
- **`DRM_MODE_ATOMIC_NONBLOCK`** — `drm_atomic_commit()` の代わりに `drm_atomic_nonblocking_commit()` に振り向ける。ioctlは、ハードウェアが実際にフリップするのを待たずに、コミットが検証されキューに入った時点で戻る。
- **`DRM_MODE_PAGE_FLIP_EVENT`**（レガシーなページフリップioctlと共有）— 完了イベントを要求する。`TEST_ONLY` とは相互排他的である（ドライランには完了を通知すべき対象がないため、`drm_mode_atomic_ioctl()` で明示的にチェックされる）。

### 完了: vblankに同期したイベント {#completion-vblank-synced-events}

`DRM_MODE_PAGE_FLIP_EVENT` が設定されている場合、ioctlハンドラは影響を受ける各CRTCの `crtc_state->event` に `struct drm_pending_vblank_event` を付加する。ドライバのアトミックコミットワーカーが新しい状態を実際にハードウェアに書き込み、次のvblankでディスプレイがフリップすると、ドライバは **`drm_crtc_send_vblank_event(struct drm_crtc *crtc, struct drm_pending_vblank_event *e)`**（`drivers/gpu/drm/drm_vblank.c`）を呼ぶ。これはイベントに現在のvblankシーケンス番号とタイムスタンプを刻み、配送のためにキューに入れる。ユーザー空間はそれをDRMファイルディスクリプタから `DRM_EVENT_FLIP_COMPLETE` として読み戻す。`struct drm_crtc_commit`（進行中のコミットごと、CRTCごとに1つあり、`drm_crtc_state.commit` から参照される）は、これをドライバやヘルパーが待機できる3つの `completion` 段階として公開する。`flip_done`（ハードウェアがフリップした。イベントが発火するのと同じ瞬間）、`hw_done`（コミットのためのレジスタ書き込みがすべて出された。*無効化*されるCRTCの場合は `flip_done` より遅くなることがある）、`cleanup_done`（古いバッファが解放された）である。これにより、`DRM_MODE_ATOMIC_NONBLOCK` のコミットが競合することなく、正しく互いの後ろに並ぶことができる。

## 実例: ホットプラグ、続いてoverlayでの動画コミット {#worked-example-hotplug-then-an-overlay-video-commit}

**1. ディスプレイが現れる。** ドライバのホットプラグ割り込みハンドラが `drm_kms_helper_hotplug_event()` を呼ぶ。コンポジタはueventを受け取り、`DRM_IOCTL_MODE_GETCONNECTOR` でconnectorを読み直し、`status == connector_status_connected` で新たにモードリストが埋められた新しいconnectorを見つける。それを点灯させるため、コンポジタは1つの `DRM_IOCTL_MODE_ATOMIC` リクエストを組み立てる。オブジェクトIDはconnector、そのCRTC、CRTCのprimary planeのもの。プロパティは、connectorには `CRTC_ID`、CRTCには `MODE_ID`（選択した `drm_display_mode` を参照するblobプロパティ）と `ACTIVE=1`、primary planeには `CRTC_ID`/`FB_ID`/`CRTC_X`/`CRTC_Y`/`CRTC_W`/`CRTC_H`/`SRC_*` である。`DRM_MODE_ATOMIC_ALLOW_MODESET`（これは本物のモード変更である）を設定し、最初のドライランとして `DRM_MODE_ATOMIC_TEST_ONLY` も設定する。

カーネル側では、`drm_mode_atomic_ioctl()`（`drivers/gpu/drm/drm_atomic_uapi.c`）が `objs_ptr`/`props_ptr`/`prop_values_ptr` を走査し、各オブジェクトを `drm_mode_object_find()` で解決し、各プロパティを `drm_atomic_set_property()` に通す。これが該当するオブジェクトをコミットに取り込み（`drm_atomic_get_crtc_state()` など）、提案された `drm_crtc_state`/`drm_plane_state`/`drm_connector_state` を変更する。`TEST_ONLY` が設定されているので、`drm_atomic_check_only()` だけを呼ぶ。これはドライバの `atomic_check`（通常は `drm_atomic_helper_check()` から始まる）を実行し、モード設定による拒否はスキップし（ここでは `drm_atomic_crtc_needs_modeset()` はtrueだが、`allow_modeset` が設定されていた）、レジスタ書き込みを1つも行わずに0またはエラーを返す。コンポジタは同じリクエストを `TEST_ONLY` なしで（必要に応じて `DRM_MODE_PAGE_FLIP_EVENT` を追加して）繰り返し、実際に適用する。今度は `drm_atomic_commit()` が実行され、ハードウェアが再プログラムされ、新しいモードで実際にスキャンアウトが始まると `drm_crtc_send_vblank_event()` が完了を通知する。

**2. 動画アプリがダイレクトスキャンアウトのoverlayを要求する。** フルスクリーンの動画プレーヤーは、デコードしたフレームにGPU合成を完全にバイパスさせたい。デスクトップと同じCRTCで使える `DRM_PLANE_TYPE_OVERLAY` 型のplaneを見つけ（`DRM_IOCTL_MODE_GETPLANERESOURCES` / `GETPLANE` で列挙し、`possible_crtcs` でフィルタする）、そのplaneだけに触れるアトミックコミットを発行する。デスクトップのCRTCにバインドする `CRTC_ID`、最新のデコード済みバッファを指す `FB_ID`、動画ウィンドウの位置を示す `CRTC_X`/`CRTC_Y`/`CRTC_W`/`CRTC_H`、ソースのクロップを示す `SRC_X`/`SRC_Y`/`SRC_W`/`SRC_H`（16.16固定小数点）、そしてデスクトップのprimary planeより上で、どのカーソルよりも下の `zpos` である。`ALLOW_MODESET` はない。ここにはモード設定を必要とするものは何もないはずなので、コミットはそれなしで通る。仮にバグによってCRTCがモード設定を必要とするようになったとしても、カーネルは画面を点滅させるのではなくコミットを拒否する。これはまさに、コンポジタがフレームごとの更新に求める安全性である。`DRM_MODE_ATOMIC_NONBLOCK | DRM_MODE_PAGE_FLIP_EVENT` により、プレーヤーは次のフレームのコミットをブロックせずにキューに入れられ、前のフレームのフリップ完了イベントを待ってから次を投入することでペースが調整される。ページフリップのループがずっと持っていたのと同じリズムであり、それが今ではplane固有のioctlではなく、フレームごとに1つのアトミックコミットとして表現されているだけである。

## 参考文献 {#further-reading}

- [カーネルドキュメント: kernel mode setting (KMS)](https://docs.kernel.org/gpu/drm-kms.html) — オブジェクトモデルとアトミック設計に関する正式なリファレンス。`Documentation/gpu/drm-kms.rst` と同期が保たれている
- [`f453ba046074`](https://git.kernel.org/linus/f453ba0460742ad027ae0c4c7d61e62817b3e7ef) — "DRM: add mode setting support"。2008年12月の最初のKMSマージ
- [`cc4ceb484b37`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=cc4ceb484b37b9369e0d4e8682b7ae1849ae4579) — "drm: Global atomic state handling"、Daniel Vetter、2014年7月。コアのアトミック状態オブジェクト
- [lore.kernel.org: atomic, remixed](https://lore.kernel.org/dri-devel/1406497308-30733-1-git-send-email-daniel.vetter@ffwll.ch/) — 2014年7月の設計スレッド。Rob Clarkによる自身の競合アプローチの擁護を含む
- [`d34f20d6e2f2`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d34f20d6e2f21bd3531b969dc40913181a8ae31a) — "drm: Atomic modeset ioctl"、Rob Clark、2014年12月。`DRM_IOCTL_MODE_ATOMIC` のエントリポイント
- [lore.kernel.org: Atomic Properties (RFC: drm: Atomic modeset ioctl)](https://lore.kernel.org/dri-devel/1418771141-16954-1-git-send-email-robdclark@gmail.com/) — uAPIの議論。却下されたタイムスタンプベースのスケジューリング案を含む
- [`5164f7e7ff8e`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5164f7e7ff8ec7d41065d3862630c2ba09854328) — "drm: Rename struct drm_atomic_state to drm_atomic_commit"、Maxime Ripard、2026年4月
- [LWN: "prepare for atomic/nuclear modeset/pageflip"](https://lwn.net/Articles/600369/) — 移行の始まりとなった、Rob Clarkによる2014年5月のインフラシリーズ
- [LWN: Atomic mode setting design overview, part 1](https://lwn.net/Articles/653071/) — Daniel Vetter、2015年8月。レガシーインターフェースの歴史と限界について
- [LWN: Atomic mode setting design overview, part 2](https://lwn.net/Articles/653466/) — Daniel Vetter、2015年8月。チェック／コミットの分離とロックについて
- カーネルツリー内の `drivers/gpu/drm/drm_atomic.c`、`drivers/gpu/drm/drm_atomic_uapi.c`、`drivers/gpu/drm/drm_atomic_helper.c` — 現在の実装
- [DRM: Direct Rendering Manager](README.md) — より広いGPUスタックの中でのオブジェクトモデルの位置付け
