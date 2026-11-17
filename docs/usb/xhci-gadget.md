# ホストコントローラ（xHCI）とガジェットモード {#host-controllers-xhci-and-gadget-mode}

> ホストスタックの最下層 — xHCIが実際にどのように転送を回線上にスケジュールするか — と、その鏡像として、Linuxがガジェットフレームワークを通じてどのようにUSB *デバイス*になるか

[概要](README.md)では、ホストコントローラを「回線を所有する」ブラックボックスとして扱った。このページではその中を開ける。**ホストコントローラドライバ（HCD）**が何をするのか、現代的な**xHCI**コントローラがどのような構造になっているのか、そしてまったく逆方向の、Linux*が*ペリフェラルになる**ガジェットモード**を見ていく。

## ホストコントローラドライバ {#the-host-controller-driver}

ホスト側のUSBスタックはすべて、最終的にHCDに行き着く。usbcoreは統一された`struct usb_hcd`インターフェース — このURBをキューに入れる、あのURBをキャンセルする、ルートハブの状態はどうか — を通じてHCDとやりとりするので、カーネルの他の部分がコントローラ固有の詳細を目にすることはない。系統を新しい順に挙げる:

- **xHCI**（eXtensible Host Controller Interface）— 現代のコントローラ。1つのxHCIでUSB 1、2、3以降のすべての速度を扱う。現在のハードウェアは事実上すべてこれを使っている。
- **EHCI**（USB 2のハイスピード）、**OHCI/UHCI**（USB 1）— レガシー。EHCIは歴史的に、自身が扱えないロースピード／フルスピードのデバイスを処理するために*コンパニオン*のOHCI/UHCIコントローラを必要とした。

ホストコントローラはほぼ常に[PCIeファンクション](../drivers/pci-driver.md)（あるいはSoCのプラットフォームデバイス）なので、HCDはコントローラをプローブしてそのレジスタをマップするPCI／プラットフォームドライバでもある。

## xHCIの内部: リング、TRB、コンテキスト {#inside-xhci-rings-trbs-and-contexts}

xHCIの設計上の要は、**転送スケジューリングをコントローラ側に押し込んだ**ことである。EHCIが*ドライバ*に編み上げさせるフレームリスト型のスケジュールを公開していたのに対し、xHCIは共有メモリ上の**リング**を中心に構成されている — [NVMe](../block/nvme.md)のサブミッション／コンプリーションキューと同じプロデューサ／コンシューマパターンである:

- **Transfer Request Block（TRB）**はリングのエントリである — それぞれが転送の1つの塊（データ、セットアップステージ、次のリングセグメントへのリンク）を記述する。
- **転送リング（transfer ring）** — アクティブなエンドポイントごとに1つ。ドライバがTRBを追加し、コントローラがそれを消費する。
- **コマンドリング（command ring）** — ドライバがコントローラへのコマンド（デバイススロットの有効化、エンドポイントの構成、アドレスの設定）を発行する。
- **イベントリング（event ring）** — コントローラが完了通知やポートイベントをここに投入し、ドライバがそれを消費する。通常は**MSI-X**割り込みを契機とする。
- **ドアベルレジスタ（doorbell register）** — ドライバがコントローラに「このリングに仕事を追加した」と伝える手段で、ポーリングを避けられる。
- **Device Context Base Address Array（DCBAA）** — デバイスごとの**デバイスコンテキスト**構造体への*ポインタ*の配列。各デバイスコンテキストは、そのデバイスの*スロット*コンテキストと*エンドポイント*コンテキスト群を保持し、エンドポイントコンテキストは各エンドポイントの状態（種別、最大パケットサイズ、リングのデキューポインタ）を持つ。コンテキストはコントローラが所有し、更新する。

その結果、URBをキューに入れる処理は「エンドポイントの転送リングにTRBを追加してドアベルを鳴らす」ことになり、完了はイベントとして届く — パケットごとにレジスタを叩くのではなく、ロックレスでメモリ上に置かれた受け渡しである。ドライバは`drivers/usb/host/xhci*`にある。

## ガジェットモード: USBデバイスとしてのLinux {#gadget-mode-linux-as-a-usb-device}

ここまでの内容はすべて**ホスト**側である。しかし、フラッシュドライブとして振る舞うスマートフォン、USB越しにシリアルコンソールを公開する開発ボード、`g_ether`によるネットワークリンク — これらはLinuxが**ペリフェラル**として動作している例である。デバイス側は3つの部分からなる別個のスタックである:

```
  Gadget function(s)      ← mass storage, serial (ACM), ethernet (ECM/RNDIS),
        │                   HID, or a userspace function via FunctionFS
  Composite framework     ← assembles functions into configurations,
        │                   builds the descriptors the host will enumerate
  UDC driver              ← drives the USB Device Controller silicon
        │                   (the device-side analogue of the HCD)
  UDC hardware  → the wire → some host enumerates *us*
```

- **UDC（USB Device Controller）ドライバ**はHCDのデバイス側の対応物である。ペリフェラルコントローラを駆動し、エンドポイントをガジェット層に提供する。
- **コンポジットフレームワーク**は、再利用可能な**ファンクション**ドライバからデバイスを組み立てられるようにする。各ファンクションのインターフェース／エンドポイントディスクリプタを、ホストが列挙時に読み取るコンフィギュレーションへとまとめ上げる — [列挙](enumeration.md)のページで扱ったのと同じディスクリプタを、今度は読む側ではなく*提供する*側として扱うわけである。
- ファンクションは`/sys/kernel/config/usb_gadget/`以下の**configfs**を通じて実行時にバインドできる。ガジェットを作成し、ベンダー／プロダクトIDを設定し、ファンクション（`mass_storage.0`、`acm.0`、`ecm.0`）をインスタンス化し、それらをコンフィギュレーションにリンクし、UDCにバインドすれば稼働状態になる。**FunctionFS**はさらに進んで、*ユーザー空間*のプログラムがファンクションのエンドポイントを実装できるようにする（Androidのadbはこの仕組みで動いている）。

## デュアルロールとOTG {#dual-role-and-otg}

USB-Cポートは**デュアルロール**（DRD）であることが多い。接続されたものに応じてホストにもデバイスにもなることができ、その役割はコネクタ上でネゴシエートされる。カーネルのロール切り替え機能は、役割の変化に合わせてポートをxHCI（ホスト）ドライバとUDC（ガジェット）ドライバの間で移し替える — 同じ物理ポートの背後に、まったく異なる2つのスタックがあるわけである。

## 参考文献 {#further-reading}

- [Kernel docs: USB gadget API](https://docs.kernel.org/driver-api/usb/gadget.html) — コンポジット／ファンクションのフレームワーク
- [Kernel docs: gadget configfs](https://docs.kernel.org/usb/gadget_configfs.html) — 実行時にガジェットを組み立てる
- [Kernel docs: FunctionFS](https://docs.kernel.org/usb/functionfs.html) — ユーザー空間からガジェットファンクションを実装する
- [USBの概要](README.md) — このページの内容がその下に位置するホストスタック · [PCIドライバ](../drivers/pci-driver.md) — ホストコントローラの接続方法
