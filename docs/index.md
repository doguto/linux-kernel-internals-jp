---
hide:
  - navigation
---

# Linux Kernel Internals {#linux-kernel-internals}

Linuxカーネルを理解するためのコミュニティハブ — 設計判断、内部構造、そしてコントリビュートに至る道のりについてのドキュメントと議論の場。

## これは何か {#what-this-is}

**ドキュメント**: カーネルのサブシステムを深く掘り下げ、APIだけでなく、物事が*なぜ*そのように動くのかを説明する。

**ディスカッション**: 質問をしたり、発見を共有したり、カーネルの理解を互いに助け合ったりする場。

## なぜか？ {#why}

カーネルには充実したAPIドキュメントがあるが、その*根拠*を理解するには、メーリングリストのアーカイブ、散在する記事、そして暗黙知を掘り起こす必要がある。我々は、より手に取りやすいリソースを作っている。

## ドキュメント {#documentation}

### 全般 {#general}

- [Linuxの進化](linux-evolution.md) — 趣味のプロジェクトから世界のインフラへ

### サブシステム {#subsystems}

**基礎**

- [カーネル内部（kernel/）](kernel/README.md) — すべてのサブシステムをつなぎとめる接着剤: ブート、ロギング、パラメータ、パニック、モジュール初期化
- [アーキテクチャ（arch/）](arch/arm64/README.md) — カーネルがハードウェアと接する場所: ブート、CPU機能、例外、アーキテクチャ固有の事項（x86、arm64）
- [システムコール（syscalls/）](syscalls/README.md) — ユーザー空間がカーネルへ入り、また戻る仕組み
- [モジュール（modules/）](modules/README.md) — カーネルコードを実行時にロード、リンク、アンロードする
- [時間（time/）](time/README.md) — クロック、タイマー、ティック、そしてカーネルが時刻を保持・計測する仕組み

**メモリ**

- [メモリ管理（mm/）](mm/README.md) — カーネルが物理メモリと仮想メモリを割り当て、マップし、回収する仕組み
- [IOMMU（iommu/）](iommu/README.md) — デバイスとメモリの間のアドレス変換と分離

**スケジューリングと並行性**

- [スケジューラ（sched/）](sched/README.md) — 次にどのタスクをいつ実行するかをカーネルが決める仕組み
- [ロック（locking/）](locking/README.md) — 並行コードの正しさを保つプリミティブ: spinlock、mutex、RCU
- [割り込み（interrupts/）](interrupts/README.md) — ハードウェアイベントの処理と、作業の安全な遅延実行
- [IPC（ipc/）](ipc/README.md) — プロセス間の通信方法: パイプ、シグナル、共有メモリ、メッセージキュー

**ストレージとI/O**

- [VFS（vfs/）](vfs/README.md) — 1つのAPIですべてのファイルシステムを駆動できるようにする抽象化
- [ファイルシステム（filesystems/）](filesystems/README.md) — ディスク上のフォーマットとページキャッシュがバイト列をファイルに変える仕組み
- [ブロック層（block/）](block/README.md) — ファイルシステムの要求から物理デバイスまでの経路
- [I/Oパターン（io/）](io/README.md) — バッファードI/OとダイレクトI/O、同期と非同期、そしてそれらの間のトレードオフ
- [io_uring（io-uring/）](io-uring/README.md) — 高性能な非同期I/Oのためのリングベースのインターフェース

**デバイスとドライバ**

- [ドライバ（drivers/）](drivers/README.md) — デバイスモデルと、ドライバがハードウェアにバインドする仕組み
- [USB（usb/）](usb/README.md) — ホストがスケジューリングするホットプラグ対応の周辺機器バス: ディスクリプタ、エンドポイント、URB
- [GPU / DRM（drm/）](drm/README.md) — グラフィックスとディスプレイのスタック: KMSによるモード設定、GEMバッファ、コマンド投入

**ネットワークとBPF**

- [ネットワーク（net/）](net/network-stack-overview.md) — ソケットから回線まで、スタックを通るパケットの旅
- [BPF（bpf/）](bpf/README.md) — トレーシング、ネットワーク、セキュリティのためにサンドボックス化されたプログラムをカーネル内で実行する

**分離とセキュリティ**

- [Cgroups（cgroups/）](cgroups/README.md) — プロセスのグループごとにリソースを計上・制限する。コンテナの基盤
- [セキュリティ（security/）](security/README.md) — LSM、ケーパビリティ、seccomp、そしてカーネルのアクセス制御機構
- [仮想化（virtualization/）](virtualization/README.md) — KVMと、カーネルがゲストマシンを実行する仕組み
- [暗号（crypto/）](crypto/README.md) — カーネルの暗号APIとハードウェアアクセラレーション
- [Livepatch（livepatch/）](livepatch/README.md) — 実行中のカーネルを再起動せずにパッチする

**可観測性**

- [トレーシング（tracing/）](tracing/README.md) — ftrace、tracepoint、perf: カーネルが何をしているかを見る
- [デバッグ（debugging/）](debugging/README.md) — カーネルの問題を診断するためのツールと手法
- [電源管理（power/）](power/README.md) — サスペンド、レジューム、cpufreq、アイドル状態

## コミュニティ {#community}

- コントリビュータとして始めるには[コントリビューションガイド](contributing.md)を参照
- 質問や会話には[GitHub Discussions](https://github.com/laveeshb/linux-kernel-internals/discussions)
- トピックの提案や問題の報告には[GitHub Issues](https://github.com/laveeshb/linux-kernel-internals/issues)

## 免責事項 {#disclaimer}

これはコミュニティの学習リソースであり、決定版のリファレンスではない。Linuxカーネルは複雑で、絶えず進化している。我々は正確さに努め、一次資料（コミット、LKML）へのリンクを示しているが、誤りが存在する可能性はある。疑わしい場合は、[公式カーネルドキュメント](https://docs.kernel.org/)とソースコードを参照してほしい。貢献や訂正を歓迎する。

## ライセンス {#license}

- ドキュメント: [CC BY-SA 4.0](https://github.com/laveeshb/linux-kernel-internals/blob/main/LICENSE-CC-BY-SA-4.0)
- コードスニペット: [GPL-2.0](https://github.com/laveeshb/linux-kernel-internals/blob/main/LICENSE-GPL-2.0)
- Tuxロゴ: Larry Ewing（オリジナル）、Simon Budig（SVG）
