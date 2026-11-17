# DRM: Direct Rendering Manager {#drm-the-direct-rendering-manager}

> Linuxが1つのGPUを多数のプロセスでどう共有するか — 特権が必要なディスプレイ操作とメモリ管理操作はカーネル内に置き、ハードウェアが実際に実行するコマンドバッファはユーザー空間が組み立てる

## はじめに {#getting-started}

GPUは、1台しかない高価でステートフルなハードウェアであり、無関係な多数のプロセスが同時に触る必要がある。デスクトップをスキャンアウトするコンポジタ、WebGLを描画するブラウザのタブ、描画コールを発行するゲーム、固定機能ブロックで動画をデコードするメディアプレーヤーなどである。`drivers/gpu/drm/` 以下にある **DRM（Direct Rendering Manager）** は、この共有を調停するカーネルサブシステムである。DRMは2つの部分に分かれる。

- **KMS（Kernel Mode Setting）** — *出力*側。ディスプレイモードを選択し、フレームバッファを物理コネクタに結び付け、HDMI/DisplayPort/eDP経由でピクセルを送り出す。
- **レンダリングとアクセラレーション** — *計算／描画*側。GPUから見えるメモリを割り当て、コマンドバッファを投入し、その処理をシステムの他の部分と同期させる。

### DRMが解決する問題 {#the-problem-drm-solves}

DRM以前は、Xサーバー（XFree86）がユーザー空間からグラフィックスハードウェアを直接駆動していた。rootとして動作し、自らハードウェアレジスタを叩いていたのである。つまり、特権を持ちクラッシュしやすい単一のプロセスが、チップセットごとの独自ドライバ群の中でrootレベルのハードウェアアクセスを内部的に重複して抱えていた。そこにバグがあればマシン全体をハングさせたり破壊したりしかねず、2つのクライアントが同じディスプレイやGPUメモリを奪い合うのを止めるカーネル側の調停者も存在しなかった。初期のDRMは複数クライアント間の*レンダリング*を調停するだけで、モード設定自体はXの手に残っていた。それをカーネルに移したのが **KMS**（カーネルモード設定）であり、[`f453ba046074`](https://git.kernel.org/linus/f453ba0460742ad027ae0c4c7d61e62817b3e7ef)（"DRM: add mode setting support"。DRMメンテナのDave AirlieがSigned-offし、Linux 2.6.29の開発サイクルの一部として2008年12月にメインラインにコミットされた）によって実現した。

DRMの答えは、カーネルでおなじみのパターンである。信頼が必要な部分はカーネルに置き、そうでない部分はユーザー空間に残す。カーネル側は、モード設定、GPUメモリ管理、コマンド投入の管理情報（あるジョブがどのバッファに触れるか、いつそれらを回収しても安全か）を検証し、所有する。ユーザー空間ドライバ — 主に **Mesa** — は、OpenGL/Vulkanの呼び出しをハードウェア固有のコマンドバッファ形式に変換する方法を実際に知っている部分であり、特権を持たないユーザー空間でそのバッファを組み立て、ioctlを通じてカーネルに渡す。カーネルはコマンドストリーム自体は検査しない。分離は代わりにコンテキストごとのGPUページテーブル（GPU自身のMMU）によって実現される。カーネルは投入が参照するバッファだけをそのコンテキストのページテーブルにバインドするため、あるクライアントのコマンドは別のクライアントのメモリをそもそもアドレス指定できない。

### アーキテクチャ: コア＋ドライバ、カーネル＋ユーザー空間 {#architecture-core-drivers-kernel-userspace}

```
  Application (game, browser, compositor)
          │  OpenGL / Vulkan calls
          ▼
  Mesa (userspace GPU driver)   ← builds command buffers, talks to libdrm
          │  ioctls via libdrm
          ▼
  DRM core (drivers/gpu/drm/)  ← generic KMS objects, GEM, dma-buf plumbing, scheduler
          │
          ▼
  Per-hardware DRM driver       ← i915 / xe (Intel), amdgpu (AMD),
          │                        nouveau (NVIDIA), many SoC display drivers
          ▼
  GPU hardware
```

**DRMコア**は共有の仕組み — KMSオブジェクトモデル、GEMバッファのライフサイクル、dma-buf/fenceの配管、汎用GPUスケジューラ — を提供する。これにより各ハードウェアドライバは、共通のioctlとuAPIの下にあるハードウェア固有の部分だけを実装すればよい。その上では `libdrm` が生のioctlを安定したC APIでラップしている。Mesaは歴史的に完全にこれの上に構築されてきたが、現在ではいくつかのMesaドライバ（Intelのiris/anvなど）が一部のioctlを直接発行している。

### 2つのデバイスノード、2つの特権レベル {#two-device-nodes-two-privilege-levels}

DRMは、信頼レベルの異なるキャラクタデバイスを通じてGPUを公開する。ディスプレイ専用ドライバは1つだけを公開するが、ほとんどのGPUドライバは2つを公開する。

- **`/dev/dri/card0`** — **プライマリノード**。これは特権インターフェースであり、モード設定を担い、歴史的にはすべてのレンダリングも担っていた。ディスプレイサーバーやコンポジタは、出力を設定するためにこれを開く。
- **`/dev/dri/renderD128`** — **レンダーノード**。オフスクリーンレンダラやGPGPUクライアント — ヘッドレスの計算ジョブや動画トランスコーダ — が、ディスプレイサーバーへの認証*なしに*、かつモード設定のアクセス権を一切持たずにレンダリング処理を投入できるように導入された。レンダーノードではDRM-Masterの概念が完全に取り除かれている。グローバルな、あるいは特権を要するioctlは一切許されず、許可されるのは非グローバルなレンダリングコマンドとPRIME（dma-buf）によるバッファ共有だけである。アクセス制御は、単にノードに対するファイルシステムのパーミッションで行われる。

### このセクションの残り {#the-rest-of-this-section}

以降のページでは、上で概要を述べた各要素をさらに掘り下げる。

- **[KMSモード設定](kms.md)** — オブジェクトモデル（CRTC、plane、connector、encoder）と、アトミックモード設定がディスプレイ構成全体を1つの不可分なトランザクションとしてコミットする仕組み。かつての、プロパティごとに設定し、間違えればちらつくインターフェースに代わるものである。
- **[GEMバッファオブジェクトとdma-buf](gem-dmabuf.md)** — GPUメモリがどう割り当てられ参照カウントされるか（GEM）、そしてバッファがデバイスやサブシステムを*またいで*どう共有されるか — GPUからディスプレイへ、V4L2カメラがキャプチャしたフレームをGPUへ — をdma-bufを通じて説明する。
- **[コマンド投入、fence、GPUスケジューラ](command-submission.md)** — GPU処理のバッチがドライバ固有のioctlから `drm_sched` を経てハードウェア実行に至るまでの流れ、非同期でパイプライン化されたGPU処理をまたいで「このバッファはまだ使用中か」をカーネルが `dma_fence` でどう追跡するか、そしてジョブがハングしたときに何が起こるか。
- **[事件簿](war-stories.md)** — 上で扱った `drm_sched`/TDR/`dma_fence` の仕組みから生じた、実際のハング、デッドロック、スケジューラの公平性に関する5つのインシデント。いずれもCVEではない。

### 前提知識と関連分野 {#prerequisites-and-neighbors}

デスクトップやサーバーでは、GPUは通常[PCIeデバイス](../drivers/pci-driver.md)である。SoCでは、GPUとディスプレイコントローラは通常、同じドライバコアを通じてバインドされるプラットフォームデバイスである。いずれにせよ、GPUが読み書きするバッファオブジェクトは、他のDMA対応デバイスと同じ[DMA](../mm/dma.md)のコヒーレンシ規則に従う。読む順序は、このページ、次に[KMS](kms.md)、次に[GEM/dma-buf](gem-dmabuf.md)、次に[コマンド投入](command-submission.md)、そして[事件簿](war-stories.md)である。各ページは、前のページのデバイス／メモリ管理の概念の上に積み上がっている。

## 参考文献 {#further-reading}

- [カーネルドキュメント: GPU driver developer's guide](https://docs.kernel.org/gpu/index.html) — DRMに関するカーネルドキュメント全体のトップレベル索引
- [カーネルドキュメント: DRM contribution guide](https://docs.kernel.org/gpu/introduction.html) — スタイルガイドライン、貢献プロセス、グラフィックススタックに関する講演の厳選リスト
- [カーネルドキュメント: kernel mode setting (KMS)](https://docs.kernel.org/gpu/drm-kms.html) — CRTC/plane/connector/encoderのオブジェクトモデルとアトミックモード設定
- [カーネルドキュメント: memory management](https://docs.kernel.org/gpu/drm-mm.html) — GEM、TTM、バッファオブジェクトのライフサイクル
- [カーネルドキュメント: DRM internals](https://docs.kernel.org/gpu/drm-internals.html) — DRMコアのドライバ向け内部構造
- [カーネルドキュメント: DRM uAPI](https://docs.kernel.org/gpu/drm-uapi.html) — プライマリ／レンダーノードの分割を含む、ユーザー空間向けのioctlインターフェース
- [PCIドライバ](../drivers/pci-driver.md) · [DMA](../mm/dma.md) — ほとんどのDRMドライバが載っているバス層とメモリ層
