# URBと4つの転送タイプ {#urbs-and-the-four-transfer-types}

> URBはUSBのデータ移動の唯一の単位である — それがどのように構築・投入・完了されるのか、そして4つの転送タイプごとにその形がどう変わるのか

USBドライバが動かすすべてのバイトは**URB**（USB Request Block）に乗って運ばれる。これはブロック層の[`bio`](../block/bio-request.md)のUSB版である。つまり、ドライバがコアに手渡す自己完結したI/Oリクエストであり、コアはそれをバス上にスケジュールし、完了をコールバック経由で非同期に通知する。URBのライフサイクルを理解することが、USBドライバの書き方を理解することの大部分を占める。

## URBの構造 {#anatomy-of-a-urb}

```c
struct urb {
    struct usb_device *dev;          /* 対象デバイス                      */
    unsigned int   pipe;             /* エンドポイント + 方向 + タイプ     */
    void          *transfer_buffer;  /* DMA可能なデータバッファ           */
    u32            transfer_buffer_length;
    u32            actual_length;     /* 完了時に設定される               */
    int            status;            /* 完了時に0または負のerrno         */
    usb_complete_t complete;          /* 完了時に呼ばれるコールバック      */
    void          *context;           /* ドライバ用のクッキー              */
    int            interval;          /* ポーリング周期（周期型タイプ）    */
    int            number_of_packets; /* アイソクロナスのみ                */
    struct usb_iso_packet_descriptor iso_frame_desc[0];
    /* ... */
};
```

**pipe**は、対象全体 — デバイス、エンドポイント番号、方向、転送タイプ — を1つの整数にエンコードしたもので、`usb_sndbulkpipe(dev, ep)`（ホスト→デバイスのバルク）や`usb_rcvintpipe(dev, ep)`（デバイス→ホストのインタラプト）のようなヘルパーで組み立てる。「snd/rcv」は常に*ホスト*から見た向きである。

## ライフサイクル {#the-lifecycle}

```c
urb = usb_alloc_urb(0, GFP_KERNEL);                 /* 1. 割り当て       */
usb_fill_bulk_urb(urb, dev, pipe, buf, len,         /* 2. 設定           */
                  my_complete, ctx);
usb_submit_urb(urb, GFP_KERNEL);                    /* 3. 投入（非同期） */
/* すぐに戻る。転送は後でバス上で行われる                                */

static void my_complete(struct urb *urb)            /* 4. 完了           */
{
    if (urb->status == 0)
        /* urb->actual_length バイトが転送された */ ;
    /* 連続ストリームの場合は、ここで GFP_ATOMIC を使って再投入する */
}
/* usb_free_urb(urb);  使い終わったら（参照カウント付き）  */
```

ステップ4については2つの点が重要である。完了コールバックは**アトミック（割り込み）コンテキスト**で実行される — スリープしてはならず、その中から再投入する場合は`GFP_ATOMIC`を使う。また、投入されたURBはコールバックが呼ばれるまで**参照カウントされ、コアが所有する**。キャンセルするには、ドライバは`usb_kill_urb()`（完了を待つ）または`usb_unlink_urb()`（非同期）を呼ぶのであって、決して単に解放してはならない。

ブロックしても構わない単発の転送のために、usbcoreは内部で構築・投入・待機を行う同期ラッパーを提供している: **`usb_control_msg()`**、**`usb_bulk_msg()`**、**`usb_interrupt_msg()`**。

## 4つの転送タイプ {#the-four-transfer-types}

エンドポイントのタイプ（その[ディスクリプタ](enumeration.md)から得られる）によって、コアがURBをどのようにスケジュールし、どのような保証を与えるかが決まる。

### コントロール — セットアップ、構成、小さなコマンド {#control-setup-configuration-small-commands}

コントロール転送は唯一の*双方向*タイプであり、存在が保証されている唯一のタイプでもある（エンドポイント0）。各転送は最大3つのステージを持つ: **SETUP**パケット（8バイトのリクエスト）、省略可能な**DATA**ステージ、そして**STATUS**ハンドシェイクである。列挙のすべてと、「このデバイスにこのコマンドを送る」操作の大半はコントロール転送であり、通常は`usb_control_msg()`経由で発行される。

### バルク — 大容量、高信頼、空きがあるときに {#bulk-large-reliable-whenever-theres-room}

バルク転送はエラー検査と再送が行われる（信頼性のある配送）が、**タイミングや帯域の保証はない** — ホストは周期型タイプの後に残った帯域でのみそれをスケジュールする。「大量のデータを、正しく、いずれは運ぶ」用途に最適である: マスストレージ、プリンタ、バルク型のネットワークアダプタなど。

### インタラプト — 小容量、上限のあるレイテンシ、ホストによるポーリング {#interrupt-small-bounded-latency-host-polled}

名前に反して、ホストに割り込むものは何もない。デバイスは希望するポーリング間隔（`bInterval`）を宣言し、ホストは少なくともその頻度でエンドポイントを**ポーリング**する。デバイスは「新しいものはない」と答えるか、小さなペイロードを渡す。これにより最悪ケースのレイテンシに上限が設けられ、それこそがキーボードやマウスに必要なものである。ドライバは1つのインタラプトURBを常に送信中の状態に保ち、完了コールバックから再投入し続ける。

### アイソクロナス — 帯域保証あり、再送なし {#isochronous-guaranteed-bandwidth-no-retransmission}

アイソクロナス転送は構成時に**保証された帯域**を予約し、固定スケジュールで配送するが、**決して再送されない** — 壊れたパケットや遅れたパケットは単に失われる。これは音声や映像のストリーミングにとって正しいトレードオフである。古いフレームには価値がなく、再送すればタイミングを壊すだけだからである。アイソクロナスURBは*多数の*パケットを一度に運ぶ（`number_of_packets`。各パケットはそれぞれ独自のステータスと長さを持つ`iso_frame_desc[]`エントリで記述される）。フレームごとに1つのURBをスケジュールしていては、オーバーヘッドが大きすぎるからである。

| タイプ | 信頼性 | タイミング | 帯域 | 典型的な用途 |
|---|---|---|---|---|
| コントロール | あり | なし | ベストエフォート | 列挙、コマンド |
| バルク | あり | なし | 残り帯域 | ストレージ、プリンタ |
| インタラプト | あり | 上限のあるレイテンシ | 予約（小） | HID |
| アイソクロナス | **なし** | 固定スケジュール | **予約** | 音声、映像 |

## 帯域、DMA、エラー {#bandwidth-dma-and-errors}

- **帯域予約。** 2つの周期型タイプ（インタラプトとアイソクロナス）は、インターフェースまたはオルタネートセッティングが選択された時点でバス帯域を予約する。バスが要求を満たせない場合、黙って品質を落とすのではなく`SET_INTERFACE`/`usb_submit_urb`が**失敗する** — 混雑したバス上でウェブカメラが高解像度のオルタネートセッティングを拒否することがあるのはこのためである。
- **DMA。** `transfer_buffer`はDMAでアドレス指定可能でなければならない。HCDがそれを[DMAエンジン](../mm/dma.md)用にマップする。ホットパスでは、`usb_alloc_coherent()`がすでにマップ済みのバッファを返してくれる。
- **エラー**は負の`urb->status`として届く: `-EPIPE`（エンドポイントが**ストール／ホルト**した — `usb_clear_halt()`で解除する）、`-EOVERFLOW`（**バブル**: デバイスがエンドポイントの最大パケットサイズを超えて送信した）、`-ENOENT`/`-ECONNRESET`（URBがkill／unlinkされた）、`-ESHUTDOWN`（デバイスが切断された）。

## 参考文献 {#further-reading}

- [Kernel docs: USB Request Block (URB)](https://docs.kernel.org/driver-api/usb/URB.html) — URBのライフサイクルに関する正式なリファレンス
- [Kernel docs: the USB core API](https://docs.kernel.org/driver-api/usb/usb.html) — 投入／完了のインターフェース
- [USBの概要](README.md) · [列挙とディスクリプタ](enumeration.md) — エンドポイントとそのタイプがどこから来るか
