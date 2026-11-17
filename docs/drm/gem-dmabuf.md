# GEMバッファオブジェクトとdma-buf: GPUメモリの割り当てと共有 {#gem-buffer-objects-and-dma-buf-allocating-and-sharing-gpu-memory}

> 参照カウント付きGPUメモリのために全ドライバのバッファ型が埋め込む1つの基底オブジェクトと、そのメモリをコピーせずにまったく別のデバイスへ渡すための1つのフレームワーク — dma-buf

## 問題: GPUメモリにはライフサイクルが必要であり、GPUの外に出ていく必要もある {#the-problem-gpu-memory-needs-a-lifecycle-and-it-needs-to-leave-the-gpu}

GPUが描画したフレームはGPUにとどまらない。ディスプレイコントローラによってスキャンアウトされるが、ハイブリッドグラフィックスのノートPCでは、それは2つ目のまったく別のドライバかもしれない。カメラがキャプチャしたフレームもカメラドライバにとどまらない。コンポジタはそれを、間にCPUコピーを挟まずに、拡大縮小や色変換のためにGPUへ直接渡したい。どちらのケースも同じ2つのものを必要とする。GPUから見えるメモリの塊を表す、カーネルが追跡する参照カウント付きのオブジェクトと、エクスポートする側のドライバが何も知らないドライバ境界を越えて、そのメモリへの*参照*を安全かつコピーなしに渡す方法である。

**GEM（Graphics Execution Manager）** は1つ目の部品である。すべてのDRMドライバのメモリアロケータが土台とする基底バッファオブジェクト型であり、ユーザー空間がそれを参照するために使うプロセスごと／デバイスごとのハンドルの仕組みである。**dma-buf** は2つ目である。DRM固有ではないサブシステム横断のフレームワークで、任意のカーネルバッファをファイルディスクリプタとしてエクスポートし、別のドライバがそれにアタッチし、自身のDMAのためにマップし、同期をとれるようにする。その別のドライバが2つ目のGPUであっても、ディスプレイコントローラであっても、V4L2の動画キャプチャデバイスであってもよい。DRM自身のdma-bufへのバインディングは **PRIME** と呼ばれる。

## GEM: 基底バッファオブジェクト {#gem-the-base-buffer-object}

`struct drm_gem_object`（`include/drm/drm_gem.h`）は、ドライバ固有のすべてのバッファ型が最初のメンバとして埋め込む汎用部分である。ピクセルフォーマットやハードウェア上の配置については何も知らない — それはドライバや、その上に構築されたヘルパー層に任される。扱うのは参照カウント、mmapの管理情報、そしてGEMオブジェクトをエクスポート可能にする配管だけである。

```c
// include/drm/drm_gem.h
struct drm_gem_object {
	struct kref refcount;              // 参照カウント。drm_gem_object_get()/_put()
	unsigned handle_count;              // 開かれているプロセスごとのハンドルの数

	struct drm_device *dev;             // 所有するDRMデバイス

	struct file *filp;                  // shmemのバッキングファイル。ドライバ独自のストレージならNULL

	struct drm_vma_offset_node vma_node; // mmapオフセットの管理情報

	size_t size;                        // オブジェクトの生存期間中は不変

	int name;                           // GEM_FLINK/GEM_OPEN のグローバル名。0 = 名前なし

	struct dma_buf *dma_buf;            // このオブジェクトのエクスポート／インポートに使われるdma-buf
	struct dma_buf_attachment *import_attach; // 他所からインポートされたオブジェクトでのみ設定される

	struct dma_resv *resv;              // 予約オブジェクト。通常は == &_resv
	struct dma_resv _resv;              // インポートされていないオブジェクト用の予約オブジェクト本体

	const struct drm_gem_object_funcs *funcs; // free/open/close/export/pin/vmap/mmap/evict/...
	...
};
```

このレイアウトから直接読み取れる、注目すべき点がいくつかある。

- **参照カウントは素の `struct kref` である。** `drm_gem_object_get()` は `kref_get()` を呼び、`drm_gem_object_put()` は `drm_gem_object_free()` を解放関数として `kref_put()` を呼ぶ。`drm_gem_object_free()` はドライバの `funcs->free` にディスパッチする。
- **`resv` と `_resv` は意図的に別のフィールドになっている。** 通常のローカルに割り当てられたオブジェクトでは、`resv` は自身に埋め込まれた `_resv` を指すだけである。別のデバイスのdma-bufからインポートされたオブジェクトでは、`resv` は代わりに*エクスポータ*の予約オブジェクトを指す。こうしてfenceの追跡は、直近にインポートしたドライバではなく、基盤となるバッファに従う。これこそが、2つの異なるGPUドライバが共有バッファを誰がまだ使っているかについて合意できる理由である（後述の[fenceと `dma_resv`](#fences-and-dma_resv)を参照）。
- **`dma_buf` と `import_attach` は、dma-bufとの2つの結合点である。** `dma_buf` は、GEMオブジェクトがエクスポートされた場合でもインポートされた場合でも設定される（`drm_gem.h` は、これを "either through importing or exporting"、つまりインポートかエクスポートのいずれかを通じてオブジェクトに関連付けられたものと説明している）。インポート方向に固有なのは `import_attach`（他者のdma-bufへのこのデバイスのアタッチメントを指す）であり、まさにそれゆえに `drm_gem_is_imported()` は、2つのケースを区別できない `dma_buf` ではなく `!!obj->import_attach` を調べている。
- **`name` は非推奨のグローバル名である** — その理由は後述する。

### GEMオブジェクトを参照する3つの方法と、そのうち2つが廃れた理由 {#three-ways-to-reference-a-gem-object-and-why-two-of-them-are-obsolete}

GEMオブジェクトには、ユーザー空間がioctlで受け渡しできる何らかの整数が必要である。生のカーネルポインタをシステムコールの境界越しに渡すことはできない。「どの整数か」に対して、DRMにはおおむねこの順で導入された3つの答えがあり、それぞれが前のものの制限やセキュリティ問題を修正している。

**1. ハンドル** — オブジェクトを作成またはインポートしたioctl（ドライバのdumb bufferやGEM作成のioctl、あるいは `DRM_IOCTL_PRIME_FD_TO_HANDLE`）が返す `u32` であり、1つの `struct drm_file`、つまり `/dev/dri/card0` や `/dev/dri/renderD128` 上の1つのオープンファイル記述にスコープされる。そのファイルの `object_idr` に格納される（`include/drm/drm_file.h`: "Mapping of mm object handles to object pointers. Used by the GEM subsystem."、訳: mmオブジェクトのハンドルからオブジェクトポインタへの対応表。GEMサブシステムが使用する）。あるプロセスのfdから得たハンドルは別のプロセスにとっては何の意味も持たず、ここにはプロセス間共有の仕組みは一切ない。`DRM_IOCTL_GEM_CLOSE` で解放される。

**2. グローバル名（`GEM_FLINK`/`GEM_OPEN`）— 非推奨。** `DRM_IOCTL_GEM_FLINK` はハンドルを、DRMノードを開ける*任意の*プロセスから見える小さなグローバル整数（`struct drm_gem_object.name`）に変換する。`DRM_IOCTL_GEM_OPEN` はその整数を呼び出し側自身の名前空間のハンドルに戻す。問題はその名のとおりである。flink名は小さく、ほぼ連番で割り当てられる整数であり、**アクセス制御がない**。同じシステム上の無関係な2つ目のプロセスが、推測や総当たりで得た名前で `GEM_OPEN` を呼び、渡されたこともないバッファのハンドルを得ることを止めるものは何もない。`docs.kernel.org` のDRM uAPIドキュメントは、その帰結について率直である。特権を持たないクライアントがレンダリング処理を投入できるように導入されたレンダーノードは、"the buffer-leaks, which occur if clients guess the flink names"（訳: クライアントがflink名を推測した場合に起こるバッファの漏洩）を避けるよう明示的に設計されており、**"New clients must not use the insecure FLINK interface."**（訳: 新しいクライアントは安全でないFLINKインターフェースを使ってはならない）と断言している。

**3. dma-bufファイルディスクリプタ（DRM PRIME経由）— 現代的な仕組み。** `DRM_IOCTL_PRIME_HANDLE_TO_FD` は、ハンドルを `struct dma_buf` を裏付けとするfdとしてエクスポートする。fdは通常のUnixファイルディスクリプタのセキュリティセマンティクスを持つ。推測できず、手に入れる方法は明示的に渡されることだけである — Unixドメインソケット上の `SCM_RIGHTS` 制御メッセージで渡されるか、`fork()` をまたいで継承されるかで、どちらも送り手が共有を選択している必要がある。`drivers/gpu/drm/drm_prime.c` にあるDRM自身のkerneldocは、この比較を直接述べている。PRIMEのfdは "offer additional security: as file descriptors must be explicitly sent over UNIX domain sockets to be shared between applications, they can't be guessed like the globally unique GEM names."（訳: 追加のセキュリティを提供する。ファイルディスクリプタはアプリケーション間で共有するためにUNIXドメインソケットで明示的に送られなければならないので、グローバルに一意なGEM名のように推測されることはない）。また、これは3つのうちデバイスをまたいで機能する*唯一*の方法でもある。名前やハンドルは、特定の1つのDRMドライバ自身のオブジェクト名前空間でしか意味を持たないが、dma-bufのfdはまったく別のドライバに渡すことができる（後述の[DRM PRIME](#drm-prime-drms-dma-buf-binding)を参照）。

### バッキングメモリ: shmemヘルパーとTTM {#backing-memory-shmem-helpers-vs-ttm}

`drm_gem_object` 自体は、バイトが*どこに*あるかについて何も言わない。それはその上に重ねられたバッキングメモリのヘルパーの仕事であり、ドライバがどちらを使うかは、そのGPUが専用のビデオメモリを持つかどうかに直接対応している。

**`struct drm_gem_shmem_object`**（`include/drm/drm_gem_shmem_helper.h`）は、単純でよくあるケースである: `struct drm_gem_shmem_object { struct drm_gem_object base; struct page **pages; ... struct sg_table *sgt; void *vaddr; ... }`。バッキングストレージは通常のshmemファイル（`drm_gem_object.filp`）、つまりスワップ可能なシステムRAMであり、他のプロセスメモリと同じプールから来る。統合GPUや単純なSoCのディスプレイ／レンダードライバはこれを使う。管理すべき2つ目のメモリプールがなく、システムRAMがGPUがアドレス指定できる唯一のメモリだからである。

ディスクリートGPUにはその余裕がない。オンボードのVRAMを持っており、それはPCIe越しにシステムメモリへアクセスするより高速だが、サイズははるかに限られている。そのためカーネルは何をどこに置くかを決め、メモリ圧迫に応じて両者の間でバッファを移動しなければならない。それが **TTM（Translation Table Manager）** であり、GEMより前から存在する、DRM本来の古いメモリマネージャである。別個の `drm_gem_ttm_object` 型はない。代わりに `struct ttm_buffer_object`（`include/drm/ttm/ttm_bo.h`）が `struct drm_gem_object base` を直接埋め込んでおり、shmemヘルパーと同じパターンである。

```c
// include/drm/ttm/ttm_bo.h
struct ttm_buffer_object {
	struct drm_gem_object base;
	struct ttm_device *bdev;
	enum ttm_bo_type type;
	...
	struct ttm_resource *resource;   // 現在の配置: VRAMの範囲、またはシステムメモリ
	struct ttm_tt *ttm;              // "TT"（システムメモリ）の場合のページ配列
	...
};
```

`drm_gem_ttm_helper.h` は、TTMを裏付けとするオブジェクトがDRMの他の部分に対して通常の `drm_gem_object_funcs` vtableを提示できるようにする糊（`drm_gem_ttm_vmap()`、`drm_gem_ttm_mmap()`、`container_of` マクロ `drm_gem_ttm_of_gem()`）を提供する。カーネル自身のメモリ管理ドキュメントは、この分担を次のように位置付けている。TTMはVRAMとシステムメモリにまたがる配置と追い出しについて "tri[es] to be a one-size-fits-them all solution"（訳: すべてに合う万能の解決策であろうとする）一方、GEMは単純なUMAのケースについて "identified common code between drivers and created a support library to share it"（訳: ドライバ間の共通コードを特定し、それを共有するためのサポートライブラリを作った）。amdgpuやnouveauがTTMの上にあり、統合GPUやSoCのドライバのほとんどがshmemヘルパーの上にあるのはこのためである。TTMの内部の追い出しと配置の仕組みはそれ自体が大きなサブシステムであり、ここでは扱わない。このページにとって重要なのは、どちらのバッキング型も下層は `drm_gem_object` であるため、上で述べたすべて — ハンドル、名前、dma-bufのエクスポート、`resv` — がドライバがどちらを選んだかに関係なく同じように動作するということだけである。

## dma-buf: ドライバ境界を越えたバッファの共有 {#dma-buf-sharing-a-buffer-across-driver-boundaries}

### なぜ存在するか {#why-it-exists}

dma-buf以前は、V4L2のキャプチャドライバがフレームを持っていて、別のDRMドライバのGPUがそれに触れる必要がある場合 — 拡大縮小、色変換、合成 — 選択肢は実際のメモリコピーか、その特定の2つのドライバを結合する場当たり的なハックしかなかった。どちらもスケールしない。コピーは毎フレーム帯域とレイテンシを消費し、ドライバの組み合わせごとの糊コードでは、プロデューサとコンシューマの新しい組み合わせごとに専用のコードが必要になる。

dma-bufは2011年のLinaro／カーネルのミニサミットで提案され、GPU固有ではなく、サブシステムに依存しない汎用フレームワークとして取り込まれた。基礎となるコミット [`d15bd7ee445d`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d15bd7ee445d0702ad801fdaece348fdb79e6581)（"dma-buf: Introduce dma buffer sharing mechanism"、Sumit Semwal、2011年12月26日作成、Linux 3.3のマージウィンドウ向けに2012年1月6日にDave Airlieがコミット）は、先行する成果の上に構築していることを明示している — "based on design suggestions from many people at the mini-summits... most notably from Arnd Bergmann, Rob Clark and Daniel Vetter"（訳: ミニサミットでの多くの人々からの設計上の提案、とりわけArnd Bergmann、Rob Clark、Daniel Vetterからの提案に基づく）— そして、この仕組み全体はまずTomasz Stanislawskiによって2つのV4L2デバイス間で実証されたことも明記している。その設計が固まるまでには3回の公開RFCを要した。2011年12月2日に投稿された[その2回目](https://lore.kernel.org/dri-devel/1322816252-19955-1-git-send-email-sumit.semwal@ti.com/)で、サブシステムに依存しない形の大部分が議論された。同じスレッドには、Arnd Bergmann、Daniel Vetter、Rob Clark、Dave Airlie、Alan Cox、NVIDIAのRobert Morell、Inki Dae、そしてメディア側からはメンテナのMauro Carvalho ChehabとV4L2開発者のHans Verkuil、Sakari Ailusが意見を寄せた。当時のLWNの記事 [**"DMA buffer sharing in 3.3"**](https://lwn.net/Articles/474819/)（Jonathan Corbet、2012年1月11日）は、意図された用途を、このページの冒頭とほぼ同じ言葉で述べている: *"The initial target use is sharing buffers between producers and consumers of video streams; a camera device, for example, could acquire a stream of frames into a series of buffers that are shared with the graphics adapter, enabling the capture and display of the data with no copying in the kernel."*（訳: 当初の対象用途は、動画ストリームのプロデューサとコンシューマの間でバッファを共有することである。たとえばカメラデバイスは、グラフィックスアダプタと共有される一連のバッファにフレームのストリームを取り込むことができ、カーネル内でのコピーなしにデータのキャプチャと表示が可能になる。）

その上に構築されたDRM自身のバインディングである **PRIME** は、それから間もなく続いた。[`3248877ea179`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3248877ea1796915419fba7c89315fdbf00cb56a)（"drm: base prime/dma-buf support (v5)"、Dave Airlie、2011年11月25日作成、Linux 3.4のマージウィンドウ向けに2012年3月30日コミット）は、GEMハンドルとdma-buf fdを相互変換する層を追加した。これは、その後nouveau、i915などで続いたドライバ対応のための "agreed upon starting point"（合意された出発点）であると明示されていた。v5の投稿は、2012年3月27日にDave Airlieが投稿してから約16時間以内に、Daniel Vetter、Rob Clark、Sumit Semwal、Inki Daeからの `Reviewed-by` タグと、[Ben Widawskyからの `Acked-by`](https://lore.kernel.org/dri-devel/20120327183151.0523bf8d@bwidawsk.net/)を集めた。Airlieはその3日後にコミットした。

### `struct dma_buf` {#struct-dma_buf}

`include/linux/dma-buf.h` は、エクスポートされるオブジェクトそのものを定義している。

```c
// include/linux/dma-buf.h
struct dma_buf {
	size_t size;                       // バッファの生存期間中は不変
	struct file *file;                 // ユーザー空間が保持するfd。参照カウントにも使われる
	struct list_head attachments;      // アタッチされたデバイスごとに1エントリ。resvで保護される
	const struct dma_buf_ops *ops;     // エクスポータのvtable
	...
	const char *exp_name;              // エクスポートしたドライバの名前。デバッグ用
	struct dma_resv *resv;             // 予約オブジェクト: このバッファのfence
	wait_queue_head_t poll;            // ユーザー空間のpoll()サポート
	...
};
```

`ops` はエクスポートするドライバが埋める `struct dma_buf_ops` であり — `attach`/`detach`、`map_dma_buf`/`unmap_dma_buf`、`pin`/`unpin`、`mmap`、`vmap`/`vunmap`、`release` — すべてのdma-bufインポータは、エクスポータのコールバックを直接呼ぶことは決してなく、汎用の `dma_buf_*()` APIを通じて間接的にこのvtableとやり取りする。

### attachとmapの手順 {#the-attachmap-dance}

「fdを持っている」状態から「このバッファにDMAできる」状態に至るまでは、意図的に1段階ではなく2段階になっている。

1. **`dma_buf_attach(struct dma_buf *dmabuf, struct device *dev)`** — インポータは、特定のデバイスからバッファを使う*意図*を宣言し、`struct dma_buf_attachment` を作成する。これはエクスポータの任意の `dma_buf_ops.attach` コールバックを呼び、エクスポータにとっては拒否する機会となる。そのコールバックのkerneldocはまさにその理由を説明している — "exporters which support buffer objects in special locations like VRAM or device-specific carveout areas should check whether the buffer could be move [sic] to system memory (or directly accessed by the provided device), and otherwise need to fail the attach operation."（訳: VRAMやデバイス固有のカーブアウト領域のような特殊な場所にあるバッファオブジェクトをサポートするエクスポータは、バッファをシステムメモリに移動できるか（または提供されたデバイスから直接アクセスできるか）を確認し、そうでなければattach操作を失敗させる必要がある。）あるGPUのVRAMにあるバッファは、まず移行されるかシステムメモリ経由でバウンスされない限り、2つ目のGPUや無関係なキャプチャデバイスからは物理的に到達できないかもしれない。誰かがマッピングを確定する前に、それを交渉するのが `attach()` である。
2. **`dma_buf_map_attachment(struct dma_buf_attachment *attach, enum dma_data_direction dir)`** — ここで実際に `struct sg_table` を得る。これは特に*この*アタッチメントのデバイス向けにDMAマップされたものである。`map_dma_buf` のkerneldocは、これが "may sleep, e.g. when the backing storage first needs to be allocated, or moved to a location suitable for all currently attached devices"（訳: スリープすることがある。たとえばバッキングストレージを最初に割り当てる必要がある場合や、現在アタッチされているすべてのデバイスに適した場所へ移動する必要がある場合）と記しており、さらにバッファに対する最初の呼び出しでは、エクスポータは現在の*すべての*アタッチメントを見渡し、それらすべてを同時に満たすバッキングストレージを自由に選べるとしている。

2段階に分かれているのは、GPUと、たとえばカメラのキャプチャエンジンとでは、DMAのアドレス指定の制約が本当に異なりうるからである — 異なるIOMMUドメイン、異なる到達可能な物理範囲、異なるアラインメント要件。エクスポータには、新しいデバイスがバッファに触れることを許される*前に*、そのデバイスで機能する場所へバッファを移行またはピン留めする機会が必要であり、アクセスの途中で配置が間違っていたと判明するのでは困るのである。

### DRM PRIME: DRMのdma-bufバインディング {#drm-prime-drms-dma-buf-binding}

PRIMEは、あるドライバの名前空間にあるGEMハンドルを、dma-bufのfdを往復させることで、まったく別のドライバの名前空間にあるGEMハンドルにする仕組みである。2つのioctlと、その下で実際に起こること（`drivers/gpu/drm/drm_prime.c`）は次のとおり。

- **`DRM_IOCTL_PRIME_HANDLE_TO_FD`** — エクスポート。ハンドラ `drm_prime_handle_to_fd_ioctl()` は `drm_gem_prime_handle_to_fd()` を呼び、それが `drm_gem_prime_handle_to_dmabuf()` を呼ぶ。これはハンドルに対応するGEMオブジェクトを探し、ドライバ独自の `drm_gem_object_funcs.export` か、汎用のデフォルトである **`drm_gem_prime_export()`** のどちらかを呼ぶ。いずれにせよ結果は `struct dma_buf` であり、`dma_buf_fd()` 相当の配管（予約済みのfdに対する `fd_install()`）が、`drm_prime_handle.fd` でユーザー空間にfdを返す。
- **`DRM_IOCTL_PRIME_FD_TO_HANDLE`** — インポート。ハンドラ `drm_prime_fd_to_handle_ioctl()` は `drm_gem_prime_fd_to_handle()` を呼ぶ。これはdma-bufへの参照を取得し（`dma_buf_get()`）、*この*ドライバがまったく同じdma-bufを以前にインポートしたことがあるかを確認する（`drm_file` ごとのルックアップキャッシュであり、同じバッファを2回インポートしても重複したオブジェクトを作らずに同じハンドルが返る）。なければ、ドライバの `gem_prime_import` コールバックか汎用の **`drm_gem_prime_import()`** を呼ぶ。これは `import_attach` が設定され、`resv` がエクスポータの予約オブジェクトを指す新しいGEMオブジェクトを作成する。続いてそのオブジェクトの新しいハンドルが、*インポートする側*のドライバ自身の `object_idr` に作成される。ローカルに割り当てられたオブジェクトとまったく同じハンドルの仕組みであり、ただこのドライバが割り当てたのではないメモリを包んでいるだけである。

これが、このページの冒頭で挙げた2つのシナリオの背後にある仕組みそのものである。コンポジタが描画済みフレームのGEMハンドルをfdとしてエクスポートし、そのfdを（Unixソケット、たとえばWaylandプロトコル経由で）別のディスプレイコントローラのドライバの `PRIME_FD_TO_HANDLE` に渡すケース。あるいは、V4L2ドライバがキャプチャしたフレームを同じ方法でエクスポートし、GPUドライバがそれをインポートして、ゼロコピーで中身が入った通常のGEMオブジェクトとして扱うケースである。

## fenceと `dma_resv` {#fences-and-dma_resv}

バッファを共有する2つのドライバはどちらも、「まだ誰かがこれを使っているか」に対する答えを必要とする。しかも、どちらも相手のハードウェアがどう動くかや、そのコマンドキューがどうなっているかを一切理解せずにである。dma-bufは、協調する2つの部品でこれに答える。

### `struct dma_fence`: 一度きりの完了シグナル {#struct-dma_fence-a-one-shot-completion-signal}

`struct dma_fence`（`include/linux/dma-fence.h`）がそのプリミティブである。

```c
// include/linux/dma-fence.h
struct dma_fence {
	...
	const struct dma_fence_ops __rcu *ops;
	struct list_head cb_list;   // シグナル時に実行するコールバック（シグナル後は @timestamp に置き換わる）
	u64 context;                // このfenceが属するタイムライン／エンジン
	u64 seqno;                  // そのコンテキスト内での順序
	unsigned long flags;
	struct kref refcount;
	int error;                  // 単なる成功ではなく、エラーで完了した場合に設定される
};
```

fenceは、GPUの特定の1つの処理 — コマンドバッファの投入、ページテーブルの更新、コピー — が完了することを表し、それ以上のものではない。どのエンジンがそれを生成したかは知らないし、気にもしない。他のコードは、それに対して `dma_fence_wait()`（シグナルされるまでブロックする）したり、`dma_fence_add_callback()`（非同期に通知を受ける。たとえばCPUが一度もブロックすることなく、後でスケジュールされたGPUジョブのブロックを解除するため）したりできる。`dma_fence_ops.enable_signaling` コールバックは、実装がハードウェアの完了を実際に検出する割り込みやポーリングの仕組みを接続する場所である。GPUスタックにおけるあらゆる完了待ち — バッファを再利用するために待つドライバ、スキャンアウトする前に待つディスプレイコントローラ、別のジョブからの依存関係を待つGPUジョブ — は、どのドライバやどのシリコンが生成したかに関係なく、1つ以上の `dma_fence` オブジェクトを待つことに帰着する。

### `struct dma_resv`: どのfenceがバッファに属するか、そしてその理由 {#struct-dma_resv-which-fences-belong-to-a-buffer-and-why}

`struct dma_resv`（`include/linux/dma-resv.h`）は、バッファ — `dma_buf.resv`、ひいては `drm_gem_object.resv` — に付けられ、どのfenceがそのバッファに適用されるかを追跡するコンテナである。

```c
// include/linux/dma-resv.h
struct dma_resv {
	struct ww_mutex lock;             // 更新側のロック。dma_resv_lock()/_unlock() を参照
	struct dma_resv_list __rcu *fences; // 現在アタッチされているfence
};
```

現在のカーネルは、古いドキュメントが説明しているように「1つの書き込みfence」と「共有の読み込みfence群」を2つの別々のフィールドに分けてはいない。そのモデルは進化した。代わりに、`dma_resv_add_fence()` で追加されるすべてのfenceは `enum dma_resv_usage` のタグを持ち、それは「それと並行して安全に実行できるものがどれだけあるか」の厳密な昇順になっている。

```c
// include/linux/dma-resv.h
enum dma_resv_usage {
	DMA_RESV_USAGE_KERNEL,    // カーネル内のメモリ管理（移動、クリア）— 常にこれらを待つ
	DMA_RESV_USAGE_WRITE,     // 暗黙的な書き込みアクセス — 新しい書き込みは先行する読み込み＋書き込みを待つ必要がある
	DMA_RESV_USAGE_READ,      // 暗黙的な読み込みアクセス — 新しい読み込みは先行する書き込みだけを待つ
	DMA_RESV_USAGE_BOOKKEEP,  // 暗黙的な同期は一切なし（例: 明示的に同期される投入）
};
```

ある使用レベルでfenceを要求すると（`dma_resv_get_fences()`、`dma_resv_wait_timeout()` など）、そのリストにある*より低い*使用レベルのfenceもすべて返される。`WRITE` のfenceを要求すると暗黙的に `KERNEL` のfenceが含まれ、`READ` を要求すると `WRITE` と `KERNEL` の両方が含まれる。これが、2つの別々の格納場所を必要とせずに、おなじみのreader/writerの規則（多数の並行するreaderは問題ないが、writerは全員を待たなければならない）を表現している。新しい読み込みは既存の*書き込み*fenceを待つだけでよいが、新しい書き込みは既存の読み込みと既存の書き込みの両方を待たなければならない。まさにこれを実装しているのが、`dma_resv_usage_rw()` の一見逆に見える対応付け（`write ? DMA_RESV_USAGE_READ : DMA_RESV_USAGE_WRITE`）である。「新しい書き込みは何を待つべきか」を問うと、READ以下の集合が返る。

### 具体的になぜ重要か {#why-this-matters-concretely}

2つのハードウェアが、一方が他方の物理的な完了を待たずに同じバッファに触れうる場所はどこでも、スキャンアウト後の使用（use-after-scanout）やフレームのティアリングのバグの温床である。そしてこれらのデバイスはどれも、それがいつなのかを知るために互いのコマンドキューを検査することはできない。そこで代わりに、全員が `dma_resv` を通すことに合意する。バッファのメモリを追い出したり再利用したりしようとするGPUドライバは、最後にそれを書いたジョブが完了したと仮定するのではなく、その `dma_resv` にあるfenceを確認する（または待つ）。描画の後でバッファを初めてスキャンアウトしようとするディスプレイドライバは、CRTCをプログラムする前に同じことをする。動的なdma-bufインポータ（`dma_buf_attach_ops.invalidate_mappings` コールバックに従い、移行に参加するもの）は、`dma-buf.h` に文書化された規則により、"obey the write fences and wait for them to signal before allowing access to the buffer's underlying storage through the device"（訳: 書き込みfenceに従い、デバイスを通じてバッファの基盤ストレージへのアクセスを許す前に、それらがシグナルされるのを待つ）ことが求められる。これらのいずれも、ディスプレイドライバがGPUのコマンドバッファのセマンティクスを理解することや、GPUドライバがディスプレイコントローラのタイミングを理解することを必要としない。どちらもただfenceを待つだけである。

## 実例: カメラのフレームがゼロコピーでGPUテクスチャになる {#worked-example-a-camera-frame-becomes-a-gpu-texture-zero-copy}

V4L2のキャプチャドライバがキャプチャしたフレームでバッファを埋めたところで、コンポジタはそれを、どちらの方向にもCPUコピーなしで、拡大縮小／色変換のためのGPUテクスチャとして使いたい。

**1. キャプチャ側がエクスポートする。** V4L2ドライバは、キャプチャバッファのdma-bufエクスポータとして振る舞い（`dma_buf_ops` を実装する任意のドライバが担うのと同じ役割である — V4L2自身のバッファ共有APIはまさにこの仕組みの薄いラッパーであり、実際にdma-bufの最初に実証された用途でもあった）、キャプチャしたばかりのフレームのdma-buf fdをユーザー空間に渡す。

**2. コンポジタがそれをGPUドライバにインポートする。** そのfdを、GPUのDRM fdに対する `DRM_IOCTL_PRIME_FD_TO_HANDLE` に渡す。カーネル側では、`drm_gem_prime_fd_to_handle()` がdma-bufへの参照を取得し、このファイルのインポートキャッシュにそれが見つからない（このバッファを見るのは初めて）ので、GPUドライバの `gem_prime_import` を呼ぶ。これは最終的に `drm_gem_prime_import()` / `drm_gem_prime_import_dev()` に行き着く。これがGPUドライバ内に新しい `drm_gem_object` を作成する。`import_attach` は（GPUのデバイスに対する内部の `dma_buf_attach()` から）設定され、`resv` は新しいものではなく*V4L2ドライバの*予約オブジェクトを指す — このGEMオブジェクトと元のV4L2バッファは、今やfenceの追跡を共有している。そのハンドルは、コンポジタのDRMファイルディスクリプタの `object_idr` に置かれる。

**3. 実際のGPUアクセスのためのマッピング。** `drm_gem_prime_import_dev()` は、手順2の間にコンポジタに代わってattach/mapの作業をすでに行っている。GPUのアタッチメントに対して `dma_buf_attach()` を呼び、続いて `dma_buf_map_attachment_unlocked()`（まだ `dma_resv` のロックを保持していない呼び出し元向けの変種）を呼んで、GPU自身のアドレス空間／IOMMUドメイン向けにDMAマップされた `sg_table` を得る。V4L2のキャプチャエンジンとGPUのレンダーエンジンは、到達可能な物理範囲が異なることがよくあり、まさにそのケースのためにattach/mapの分離が存在する。

**4. fenceが安全を保つ — ただしV4L2は文書化された例外である。** インポートされたGEMオブジェクトの `resv` は*エクスポータの* `dma_resv` を指しているので、エクスポートするドライバが付けたfenceは自動的にGPUドライバからも見え、暗黙的同期のもとでGPUジョブの依存関係に組み込まれる（あるいは明示的に待たれる）。しかし `dma-buf.h` の暗黙的同期の規則は、参加しないドライバとしてV4L2を名指ししている: "Some drivers only expose a synchronous userspace API with no pipelining across drivers. These do not set any fences for their access. An example here is v4l."（訳: 一部のドライバは、ドライバをまたいだパイプライン化のない同期的なユーザー空間APIしか公開しない。これらは自身のアクセスに対してfenceを一切設定しない。その例がv4lである。）V4L2は代わりに `VIDIOC_DQBUF` から順序付けを得ている。キャプチャされたバッファは、キャプチャDMAがすでに完了するまでユーザー空間に返されない（したがってdma-buf fdとして渡されることもない）ので、コンポジタがそれをインポートする時点では、fenceで守るべき実行中の処理は何も残っていない。共有 `resv` によるfenceのパスが依存関係を運ぶのは、両側が非同期にパイプライン処理するGPUからGPUへのケースである — 以下を参照。

**5. 後片付け。** コンポジタは最終的に、GPU側のハンドルに対して `DRM_IOCTL_GEM_CLOSE` を呼ぶ。GPUドライバのGEMオブジェクトへの参照を落とすのは、dma-buf fdそのものではなくハンドルの解放である。基盤となるdma-bufは、誰かがまだ参照を保持している限り — コンポジタ自身のdma-buf fd、あるいは他のインポータの `get_dma_buf()` による参照 — 生き続ける。V4L2側は予想とは逆である。バッファのエクスポートは*基盤となるvb2バッファに対する*参照を取得したので、そのキャプチャバッファのメモリは、dma-buf自体が最終的に解放されるまで解放できない。その逆ではない。

同じエクスポート／インポートの形 — あるドライバのGEMハンドルに対する `PRIME_HANDLE_TO_FD`、別のドライバのDRM fdに対する `PRIME_FD_TO_HANDLE` — は、ハイブリッドグラフィックスのノートPCで、レンダリング用GPUが合成済みフレームを物理的に別のディスプレイ出力用GPUに渡す方法でもある。ただしそこではfenceが実際に働く。レンダリング用GPUは自身の投入に対して `dma_resv_add_fence()` で `DMA_RESV_USAGE_WRITE` のfenceを付け、ディスプレイドライバはインポートしたオブジェクトの `resv` を通じて同じ `dma_resv` を見て、フリップの前にそのfenceを拾う。GEMを裏付けとするドライバのデフォルトの `prepare_fb` フックである `drm_gem_plane_helper_prepare_fb()` は、`dma_resv_get_singleton(obj->resv, DMA_RESV_USAGE_WRITE, ...)` を呼んで結果をplaneの状態に格納し、`drm_atomic_helper_wait_for_fences()` が、コミットが実際にハードウェアをプログラムする前にそれを待つ。変わるのは、どちらの側が「エクスポータ」でどちらが「インポータ」かだけである。

## 参考文献 {#further-reading}

- [カーネルドキュメント: DRM Memory Management (GEM, TTM)](https://docs.kernel.org/gpu/drm-mm.html) — GEM/TTMの設計とAPIに関する正式なリファレンス
- [カーネルドキュメント: Buffer Sharing and Synchronization (dma-buf)](https://docs.kernel.org/driver-api/dma-buf.html) — dma-buf、dma-bufのアタッチメント、暗黙的fenceと明示的fence
- [カーネルドキュメント: DRM uAPI](https://docs.kernel.org/gpu/drm-uapi.html) — "New clients must not use the insecure FLINK interface" という明示的な指針を含む
- [`d15bd7ee445d`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d15bd7ee445d0702ad801fdaece348fdb79e6581) — "dma-buf: Introduce dma buffer sharing mechanism"、Sumit Semwal、Linux 3.3向けにマージ（2012年初め）
- [lore.kernel.org: [RFC v2] Introduce DMA buffer sharing mechanism](https://lore.kernel.org/dri-devel/1322816252-19955-1-git-send-email-sumit.semwal@ti.com/) — サブシステムに依存しない設計が議論された2回目（2011年12月）のRFC
- [`3248877ea179`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3248877ea1796915419fba7c89315fdbf00cb56a) — "drm: base prime/dma-buf support (v5)"、Dave Airlie、Linux 3.4向けにマージ（2012年春）。DRMのGEMハンドル⇔dma-buf fdのバインディング
- [LWN: "DMA buffer sharing in 3.3"](https://lwn.net/Articles/474819/) — Jonathan Corbet、2012年1月11日。マージされたフレームワークと、その動機となったカメラからGPUへの用途について
- `include/drm/drm_gem.h`、`include/drm/drm_gem_shmem_helper.h`、`include/drm/drm_gem_ttm_helper.h` — GEMオブジェクトとバッキングメモリヘルパーの定義
- `include/linux/dma-buf.h`、`include/linux/dma-resv.h`、`include/linux/dma-fence.h` — dma-buf、予約オブジェクト、fenceの構造体
- `drivers/gpu/drm/drm_prime.c` — DRM PRIMEによるエクスポート／インポートの実装
- [DRM: Direct Rendering Manager](README.md) · [カーネルモード設定（KMS）](kms.md) — GEMバッファが画面に映ったときの行き先
