# USB: Universal Serial Busサブシステム {#usb-the-universal-serial-bus-subsystem}

> LinuxがUSBをどう駆動するか — ホストが*すべての*転送を開始するホストスケジュール型でホットプラグ可能なバスであり、デバイスはディスクリプタを通じて自身を記述し、ドライバはデバイス全体ではなくインターフェースにバインドする

## はじめに {#getting-started}

USBは、ほぼすべての周辺機器が話すバスである: キーボード、マウス、ストレージ、ウェブカメラ、ネットワークドングル、プリンタ、スマートフォン。カーネル側から見ると、USBにはサブシステム全体の形を決める3つの決定的な性質がある:

- **ホストスケジュール型。** マスターは1つ — ホストコントローラだけである。回線上の*すべての*転送はホストが開始し、デバイスが促されずに話すことは決してない。「インタラプト」エンドポイント（マウスが移動を報告する）でさえ、実際にはホストが一定間隔でデバイスを*ポーリング*しているのである。これはPCIのような割り込み駆動型のバスとは正反対である。
- **ホットプラグ可能で自己記述的。** デバイスはいつでも現れたり消えたりしうる。現れたときには、**ディスクリプタ**のツリーを通じて自分が何者であるかをホストに正確に伝える — 静的な設定も、デバイスツリーのエントリも不要である。
- **階層型スター・トポロジ。** デバイスはハブにぶら下がり、ハブはさらに別のハブにぶら下がり、そのすべてがホストコントローラの*ルートハブ*を根としている。1つのバスあたり最大127アドレスである。

## スタックの3つの層 {#the-three-layers-of-the-stack}

```
  Class / interface drivers   ← usb-storage, usbhid, cdc-acm, uvcvideo, ...
        │   bind to an interface, submit URBs
        ▼
  usbcore  (drivers/usb/core/) ← the hub driver, device model, enumeration,
        │                         URB machinery — the bus midlayer
        ▼
  Host Controller Driver (HCD) ← xHCI (USB3), EHCI (USB2), OHCI/UHCI (USB1)
        │   schedules transactions on the physical bus
        ▼
  Host controller hardware  (usually a PCIe function) → the wire → devices
```

- **ホストコントローラとそのドライバ（HCD）**が回線を所有する。今日主流なのは**xHCI**（eXtensible Host Controller Interface）で、USB 1からUSB 3以降までを1つのコントローラで扱う。**EHCI/OHCI/UHCI**はそれより古いUSB 2／1のコントローラである。コントローラ自体はほぼ常に[PCIデバイス](../drivers/pci-driver.md)である。
- **usbcore**は、すべてのデバイスとドライバが通過する中間層である。**ハブドライバ**（接続／切断を検出し、列挙を駆動する）、[ドライバコア](../drivers/device-model.md)の上に重ねられたUSBの**デバイスモデル**、そして**URB**エンジン（後述）を含む。ドライバに対しては、コントローラに依存しないすっきりしたデバイスの見え方を提供する。
- **クラス／インターフェースドライバ**は、あるデバイスの「USBドライバ」として思い浮かべるものである — フラッシュドライブ用の`usb-storage`、キーボード用の`usbhid`、シリアルアダプタ用の`cdc-acm`など。決定的に重要なのは、これらがデバイス全体ではなく**インターフェース**にバインドするという点である。

## デバイスモデル: デバイス → コンフィギュレーション → インターフェース → エンドポイント {#the-device-model-device-configuration-interface-endpoint}

USBデバイスは単一のアドレス指定可能なものではない。デバイスがディスクリプタを通じて記述する、小さなツリーである:

```
Device  (idVendor, idProduct — identifies the product)
└── Configuration  (a power/feature profile; usually just one, and one is active)
    ├── Interface 0  (one *function* — e.g. the video half of a webcam)
    │   ├── Endpoint 1 IN   (isochronous — video frames)
    │   └── Endpoint 0 (shared control endpoint)
    └── Interface 1  (another function — e.g. the audio half)
        └── Endpoint 2 IN   (isochronous — audio)
```

重要な帰結は、**ドライバはデバイスではなくインターフェースにバインドする**ということである。オーディオデバイスであると同時にHIDの音量コントロールでもあるUSBヘッドセットは、*同じ*物理デバイスの*2つの*インターフェースにバインドした*2つの*別々のドライバによって駆動される。`struct usb_driver`の`->probe()`が`usb_interface`を受け取るのも、マッチングがベンダー／プロダクトIDだけでなくインターフェースクラスでも行われるのが普通なのも、このためである。

## エンドポイントと4つの転送タイプ {#endpoints-and-the-four-transfer-types}

**エンドポイント**とは、デバイス上の一方向のデータの送出元または受け取り先である（IN = ホストへ向かう、OUT = ホストから来る）。各エンドポイントは、データの必要に応じて選ばれた4つの転送タイプのうちちょうど1つを持つ:

| タイプ | 保証 | 用途 |
|---|---|---|
| **コントロール** | 高信頼、双方向。エンドポイント0はすべてのデバイスに存在する | 列挙、構成、小さなコマンド |
| **バルク** | 高信頼（再送あり）、タイミング保証なし。余った帯域を使う | マスストレージ、プリンタ、バルク型ネットワーク |
| **インタラプト** | 一定間隔でのホストによるポーリングでレイテンシに上限がある。小さなペイロード | キーボード、マウス、HID |
| **アイソクロナス** | 帯域とタイミングが保証され、**再送はない** | 音声、映像（遅れたフレームは役に立たないので、再送せず破棄する） |

エンドポイント0は特別である。すべてのデバイスが持たなければならない**コントロールエンドポイント**であり、ドライバがバインドされる*前*、つまり列挙の間にホストがデバイスとやりとりする手段である。

## URB: USB I/Oの単位 {#the-urb-the-unit-of-usb-io}

実際のデータ移動はすべて**URB**（USB Request Block）として表現される — ブロック層の[`bio`](../block/bio-request.md)のUSB版である。ドライバはURBに対象エンドポイント、データバッファ、完了コールバックを設定し、非同期に投入する:

```c
usb_fill_bulk_urb(urb, dev, pipe, buffer, len, my_complete, ctx);
usb_submit_urb(urb, GFP_KERNEL);   /* すぐに戻る */
/* ... 後で、HCD の完了パスから: */
static void my_complete(struct urb *urb) { /* urb->status, urb->actual_length */ }
```

URBはHCDのキューに入れられ、バスが運べるときに回線上にスケジュールされ、完了時（あるいはエラー時やunlink時）にそのコールバックが実行される。単純な単発の転送のために、usbcoreは同期ラッパー — `usb_control_msg()`、`usb_bulk_msg()` — を提供しており、これらはURBを構築し、投入し、待機する。

## デバイスが動き出すまで（列挙の概略） {#how-a-device-comes-to-life-enumeration-in-brief}

デバイスが接続されると、ハブドライバがポートの変化を検知し、ホストがポートを**リセット**し、デバイスに**アドレス**を割り当て、エンドポイント0経由でディスクリプタを読み取る。コンフィギュレーションを選択すると、usbcoreは各インターフェースを登録済みドライバと（ベンダー／プロダクトIDまたはUSBクラスによって）**照合**し、選ばれたドライバの`->probe()`を呼ぶ。ディスクリプタのやりとりの全体は列挙のページで扱う。

## USB *デバイス*としてのLinux: ガジェットモード {#linux-as-a-usb-device-gadget-mode}

ここまでの内容はすべて**ホスト**側である。Linuxは自らUSBペリフェラルに*なる*こともできる — ストレージとして振る舞うスマートフォンや、USB越しにシリアルコンソールやEthernetを公開する組み込みボードなどである。それが**ガジェット**側である。USB Device Controller（UDC）ドライバとガジェットファンクションからなり、通常はconfigfsを通じて組み立てられる。これは別個のスタックであり、ホストコントローラのページで併せて扱う。

## このセクションの残り {#the-rest-of-this-section}

1. **[列挙とディスクリプタ](enumeration.md)** — デバイスがどのように自己申告し、ドライバがバインドされるか
2. **[URBと転送タイプ](urbs.md)** — USB I/Oの単位と、4つの転送タイプそれぞれが保証すること
3. **[ホストコントローラとガジェットモード](xhci-gadget.md)** — xHCIの内部と、USB *デバイス*として動作するLinux
4. **[事件簿](war-stories.md)** — BadUSB、MIDIのdouble-free、そしてデバイス側からのホストスタックのファジング

## 参考文献 {#further-reading}

- [Kernel docs: USB driver API](https://docs.kernel.org/driver-api/usb/index.html) — usbcoreとHCDに関する正式なリファレンス
- [Kernel docs: the USB core API](https://docs.kernel.org/driver-api/usb/usb.html) — デバイスモデル、インターフェース、ドライバインターフェース
- [Kernel docs: USB Request Block (URB)](https://docs.kernel.org/driver-api/usb/URB.html) — URBのライフサイクルの詳細
- [Linuxデバイスモデル](../drivers/device-model.md) — USBが土台とするドライバコア · [PCIドライバ](../drivers/pci-driver.md) — ホストコントローラの接続方法
