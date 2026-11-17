# 更新履歴 {#whats-new}

サイトの大きな変更 — 新しいセクション、大規模な拡充、注目すべき訂正 — を新しい順に並べる。これは重要なマイルストーンを*厳選*した一覧であり、完全なコミットログではない。完全な履歴は[リポジトリ](https://github.com/laveeshb/linux-kernel-internals/commits/main)にある。

## 2026年10月 {#october-2026}

- **罫線図をサイト全体で修正。** コードブロックは、フォールバックが罫線文字をカバーしていなかったWebフォントの代わりに、ネイティブのシステム等幅フォントを使うようになった。これにより、サイト全体でガタガタになっていたASCII図が修正された。
- **フォント修正にとどまらない、図の位置合わせの全面監査。** 数十の図は、それ自体のソーステキストにずれがあった — 箱の辺が異なる列に置かれている、矢印が対象を越えた位置に着地している、内容が枠からはみ出している — これらをメモリ管理、ファイルシステム、ネットワーク、TTY/シリアル、ALSA、仮想化、暗号などで発見し修正した。

## 2026年9月 {#september-2026}

- **正確性監査をサイト全体に拡大。** syscalls、カーネル内部、デバッグ、電源、時間、暗号、IOMMU、仮想化、IPCにわたる数十のページで、誤った、あるいは古くなった技術的詳細を現在のカーネルソースと照らし合わせて訂正した。これは8月に[virtualization/](virtualization/README.md)で始まった作業の継続である。
- **新しいページ: [PREEMPT_RT](locking/preempt-rt.md)、[sched_ext](sched/sched-ext.md)、[kGraft vs. kpatch](livepatch/kgraft-vs-kpatch-history.md)、そして2つのBPF詳細解説**（[セキュリティモデル](bpf/bpf-security-model.md)、[ヘルパー/kfuncs/JIT](bpf/bpf-helpers-kfuncs-jit.md)）。
- **事件簿はさらに増加** — [スケジューラ](sched/war-stories.md)と[割り込み](interrupts/war-stories.md)の新しいページ、USB、XFS、net/schedにわたる、独立に出典を確認したCVE解説がさらに5本（最後の1本は、それを修正したアップストリームの報告とは独立に発見されたもの）、そしてどの事件簿ページが実際の個別に出典のあるインシデントではなく例示的なものかを明確にする編集上の見直し。
- **2つの事件簿ページを新しい物語形式のテンプレートに作り直した** — [act_apiのuse-after-free CVE](net/war-stories/act-api-uaf.md)には3つの新しい図、実際の修正前/修正後のコード比較、そして破棄された最初の修正の試みに関する新たな調査が加わった。[Dirty Pipe](security/war-stories/dirty-pipe.md)は、最も強力なセクションを埋もれさせるのではなく冒頭に置くよう再構成され、その過程でいくつかの不正確な点が修正された。
- **[割り込み](interrupts/README.md)に本物の設計の歴史を追加** — このセクションで最も浅かったページが、仕組みを説明するだけでなく、カーネルが*なぜ*5つの異なる遅延作業の仕組みを持つのか、そしてそれぞれがどのように生まれたのかを説明するようになった。

## 2026年8月 {#august-2026}

- **デバイス、ドライバ、ストレージにわたる新コンテンツ。** [USB](usb/README.md)が完全な新セクションとして追加された。デバイスとドライバのカテゴリは[GPU/DRM](drm/README.md)、[サウンド/ALSA](alsa/README.md)、[入力/HID](input/README.md)、[TTY/シリアル](tty/README.md)で充実した。そして[ファイルシステム](filesystems/README.md)、[ブロック層](block/README.md)、[トレーシング](tracing/README.md)がいずれも深化した。
- **新規: [ハードウェアの進化](hardware-evolution.md)**。ハードウェアの変化がカーネル設計の変更を強いた6つの局面をたどる、姉妹編の年表。
- **事件簿をサイトの大部分に拡大**。各インシデントは記憶から要約するのではなく、実際のコミット、CVE、LWN記事と照らし合わせて独立に検証した。
- **引用の整合性の一掃**により、壊れたリンクや誤った出典リンクを修正し、引用がまったくなかったサブシステムを解消した — これにより、古い例示用コードサンプルの一部に誤った技術的詳細があることが浮かび上がり、現在も続く正確性監査が始まった。
- **ナビゲーションの全面改修。** サブシステムは、ランディングページを持つテーマ別のカテゴリタブにまとめられ、より読みやすいレイアウトになった。

## 2026年7月 {#july-2026}

- **CIでビルドの整合性を強制。** プルリクエストは、内部リンクとアンカーの検証を伴う厳格なビルドを通過しなければならなくなった。そのため、壊れた相互参照がマージされることはもうない。
- **メモリ管理をさらに深化。** [物理メモリモデル](mm/memory-model.md)、[アドレス空間のカーネル側半分](mm/kernel-address-space.md)、[GUP（Getting User Pages）](mm/gup.md)を追加した。
- **出典付けの作業。** [アーキテクチャ](arch/arm64/README.md)ドキュメント全体の参考文献セクションに出典を付け、セクションのランディングページを刷新した。

## 2026年4月 — 深化の作業 {#april-2026-depth-pass}

- **スタブだったセクションを網羅的な内容に拡充。** 詳細な拡充が[arch/x86](arch/x86/README.md)と[コアカーネル](kernel/README.md)、[I/Oとio_uring](io/README.md)、[IOMMU](iommu/README.md)と[暗号](crypto/README.md)、[livepatch](livepatch/README.md)、[syscalls](syscalls/README.md)、[モジュール](modules/README.md)、[時間](time/README.md)、[仮想化](virtualization/README.md)、[デバッグ](debugging/README.md)、[cgroups](cgroups/README.md)、[IPC](ipc/README.md)、[電源](power/README.md)、そして[arch/arm64](arch/arm64/README.md)にわたって加わった。

## 2026年3月 — メモリからカーネル全体へ {#march-2026-from-memory-to-the-whole-kernel}

- **サイトはメモリ管理のリファレンスからカーネル全体のリファレンスへと拡大した。** 20数個の新しいサブシステムセクションが一度に追加された: [スケジューラ](sched/README.md)、[ネットワーク](net/README.md)、[VFS](vfs/README.md) / [ファイルシステム](filesystems/README.md) / [ストレージ](block/README.md) / [IPC](ipc/README.md) / [syscalls](syscalls/README.md)、[ロック](locking/README.md) / [割り込み](interrupts/README.md) / [BPF](bpf/README.md) / [トレーシング](tracing/README.md)、そして[アーキテクチャ](arch/arm64/README.md) / [セキュリティ](security/README.md) / [ドライバ](drivers/README.md) / [仮想化](virtualization/README.md)。
- **メモリ管理を深化。** NUMAの高度なトピック、回収の内部、モダンなmm（folio、maple tree、TLB、RCU）、最初のmmの事件簿、そしてブート/DMAのカバー。
- **最初のサイト全体の引用監査。** 多くのサブシステムにわたって一次資料の引用を追加し、あわせて85のドキュメントにわたる正確性レビューを実施した。

## 2026年1月 — 公開 {#january-2026-launch}

- **サイトが公開された**。深い[メモリ管理](mm/README.md)のリファレンスとして、[Linuxの進化](linux-evolution.md)の歴史と対にして。
- 独自ドメイン、自動デプロイ、[コントリビューションガイドライン](contributing.md)を整備した。

---

*追いかけたい？* 更新情報を受け取るには[GitHubリポジトリ](https://github.com/laveeshb/linux-kernel-internals)をWatchまたはStarしてほしい。
