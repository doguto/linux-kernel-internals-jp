# 事件簿: 入力/HIDのバグとリグレッション {#war-stories-inputhid-bugs-and-regressions}

> 入力コアとHIDサブシステムで起きた6件の事件。6件すべてがCVEであり、以下の4つのクイックケースのうち2つはそれ自体がペアになっている。同じチェックの欠落、あるいは同じ形のバグが、何年も隔ててツリーの別の場所で再発したものである

[入力サブシステム](README.md)と[HIDサブシステム](hid.md)では、入力コア、evdev、HIDレポートのパースが現在どのように動作するかを説明している。このページは、そのアーキテクチャの背後にある事件の記録である。大半はパーサのバグ（デバイスが提供するレポートディスクリプタ、あるいはデバイスが提供する空のリストを、先に検証せずに読んでしまうもの）と、タイマー破棄時の競合（周期タイマーが、それが触るメモリが解放される前に同期的に停止されないもの）である。

## 詳細解説 {#deep-dives}

### [初期化されないHIDレポートバッファによるカーネルメモリリーク](war-stories/uninitialized-report-buffer-leak.md) {#the-uninitialized-hid-report-buffer-kernel-memory-leak}
**2024年10月（CVE公開は2024年11月）· CVE-2024-50302 · CISA Known Exploited Vulnerabilities カタログ**
1語の修正（`kmalloc` は `kzalloc` であるべきだった）で、ヒープのリークが塞がれた。Googleは、このリークがフォレンジック機器メーカーのハードウェアが携帯電話のロック解除チェーンで使った複数のゼロデイの1つであった可能性が高いと評価しており、その経緯はAmnesty InternationalのSecurity Labが文書化している。

### [2014年のバグ修正が持ち込んだ `hid_validate_values()` の型混同](war-stories/hid-validate-values-type-confusion.md) {#the-hid_validate_values-type-confusion-that-a-2014-bug-fix-introduced}
**2023年1月（CVE公開は2023年3月）· CVE-2023-1073**
壊れていたレポートID 0の検索を直す2014年の修正が、安全なハッシュテーブルの読み出しを、チェックのないリスト走査に置き換えた。その副作用として型混同が持ち込まれ、それはメモリレイアウトの運だけで、ほぼ9年間悪用されずに残っていた。

## クイックケース {#quick-cases}

### ケース1: Bluetooth HIDPのアイドルタイマーの競合 — CVE-2023-54120 {#case-1-the-bluetooth-hidp-idle-timer-race-cve-2023-54120}

`net/bluetooth/hidp/core.c` のセッション破棄処理は、接続のアイドルタイムアウト用タイマーを `del_timer()` で停止していた。これは非同期であり、タイマーを保留中タイマーのリストから外すが、別のCPUですでに実行中のコールバックの完了は待たない。問題のウィンドウはこうである。`hidp_idle_timeout()` が別の場所ですでに実行中の間に `hidp_del_timer()` が走り、破棄処理はそのまま `hidp_session_put()` に進んでセッションを解放する。するとまだ実行中のタイマーコールバックが、解放済みメモリに触れる。NVDのkernel.org由来のスコアでは CVSS 3.1 8.8（HIGH）と評価されている。

修正（[`c95930abd687`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c95930abd687fcd1aa040dc4fe90dff947916460)、Min Li、2023年3月）は1行の変更である。`del_timer(&session->timer)` → `del_timer_sync(&session->timer)` とし、実行中のコールバックが実際に完了するまでブロックしてから戻るようにした。[NVD: CVE-2023-54120](https://nvd.nist.gov/vuln/detail/CVE-2023-54120)。

### ケース2: フォースフィードバックのタイマーのuse-after-free — CVE-2019-19524 {#case-2-the-force-feedback-timer-use-after-free-cve-2019-19524}

`drivers/input/ff-memless.c` のデバイス破棄パスは、*再生中の*フォースフィードバックエフェクトをすべて停止していた（`ml_ff_destroy()` より上流で、`input_device_flush()` が `input_ff_flush()` を呼ぶことによる）。しかし `ml_ff_destroy()` 自体は、その下にある周期タイマー `ml->timer` を一度も停止しておらず、数行後にドライバのプライベートデータを解放する時点で、このタイマーがまだ保留中または実行中である可能性があった。悪意のある、あるいは誤動作するUSBフォースフィードバックデバイスは、タイマーが生きている間に破棄パスを起こすことができた。

修正（[`fa3a5a1880c9`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=fa3a5a1880c91bb92594ad42dfe9eedad7996b86)、Oliver Neukum、2019年11月）は、`ml_ff_destroy()` の `kfree()` の前に `del_timer_sync(&ml->timer)` を追加し、別の場所でエフェクトをフラッシュしてもタイマーは止まらないことを明記したコメントを添えている。ケース1と同じ種類の穴が、別のサブシステムにあったわけである。[NVD: CVE-2019-19524](https://nvd.nist.gov/vuln/detail/CVE-2019-19524)。

### ケース3: リストが空かのチェックを欠いた14個のフォースフィードバックドライバ — CVE-2019-19532 {#case-3-14-force-feedback-drivers-missing-a-list-emptiness-check-cve-2019-19532}

syzbotが見つけた一括のバグである。14個の独立したHIDフォースフィードバック用quirkドライバ（`hid-axff.c`、`hid-dr.c`、`hid-emsff.c`、`hid-gaff.c`、`hid-holtekff.c`、`hid-lg2ff.c`、`hid-lg3ff.c`、`hid-lg4ff.c`、`hid-lgff.c`、`hid-logitech-hidpp.c`、`hid-microsoft.c`、`hid-sony.c`、`hid-tmff.c`、`hid-zpff.c`）がそれぞれ、probe時に `&hid->inputs` が空でないことを先に確認せずに `list_first_entry()` または等価な `list_entry(hid->inputs.next, ...)` を呼んでいた。前者を使っていたのが3つのドライバ、残りの11が後者である。入力レポートを1つも提示しない悪意のあるデバイスは、[`hid_validate_values()` のバグ](war-stories/hid-validate-values-type-confusion.md)と同じ種類の、型が混同された不正なポインタを生み出す。空かもしれないリストに `list_entry` 系を使うという同じ誤りが、HIDコアではなくドライバごとのprobe層で、約3年早く起きていたのである。

修正（[`d9d4b1e46d95`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d9d4b1e46d9543a82c23f6df03f4ad697dab361b)、Alan Stern、2019年10月）は、14個のドライバすべてで `list_first_entry()` 呼び出しの前に明示的な `if (list_empty(&hid->inputs)) { hid_err(...); return -ENODEV; }` ガードを追加した。[NVD: CVE-2019-19532](https://nvd.nist.gov/vuln/detail/CVE-2019-19532)。

### ケース4: GTCOタブレットドライバ、同じ関数、2年足らずを隔てた2つのバグ — CVE-2017-16643 と CVE-2019-13631 {#case-4-the-gtco-tablet-driver-the-same-function-two-bugs-under-two-years-apart-cve-2017-16643-and-cve-2019-13631}

GTCOデジタイザ/タブレットドライバ（`drivers/input/tablet/gtco.c`）の `parse_hid_report_descriptor()` は、デバイスが提供するHIDレポートディスクリプタを1バイトずつ走査する。この同じ関数に、メカニズムの上では無関係な2つの別々のバグがあった。

- **2017年**: ループは各反復の前に少なくとも1バイトが残っていることを確認していたが、HIDディスクリプタの各アイテムはプレフィックスバイトに応じて1、2、4バイトを消費しうる。そのため残り1バイトのチェックではマルチバイトの読み出しの安全性が保証されず、ディスクリプタバッファを越える範囲外読み出しが起きた。[`a50829479f58`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a50829479f58416a013a4ccca791336af3c584c7)（Dmitry Torokhov、2017年10月）で修正された。switch文の上に引き上げた `if (i + size > length)` チェックを1つ追加し、3つの読み出し幅（1、2、4バイト）すべてを一度にカバーしている。[NVD: CVE-2017-16643](https://nvd.nist.gov/vuln/detail/CVE-2017-16643)。
- **2019年**: 同じ関数のまったく別のバグである。HIDのCollection/End Collectionのネストの深さを追跡するデバッグメッセージ用のインデントカウンタに上限がなく、深くネストした、あるいは対応の取れていないコレクションを持つディスクリプタによって、固定長10バイトのインデント文字列配列の末尾を越えて書き込むことができた。同じドライバに対する以前のsyzkallerの報告を受けたコードレビューで発見された。[`2a017fd82c54`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2a017fd82c5402b3c8df5e3d6e5165d9e6147dc1)（Grant Hernandez、2019年7月）で修正された。この修正は2種類のチェックを追加している。インデントカウンタをインクリメントする前の `MAX_COLLECTION_LEVELS` による上限チェック（深すぎるネストのコレクションを拒否する）と、デクリメントする前の別個のゼロ下限チェック（`indent == 0`）（カウンタをアンダーフローさせる対応の取れないEnd Collectionを拒否する）である。[NVD: CVE-2019-13631](https://nvd.nist.gov/vuln/detail/CVE-2019-13631)。

## 共通点 {#common-threads}

| パターン | レポートバッファのリーク | `hid_validate` の型混同 | BT HIDPタイマー | ff-memlessタイマー | 14ドライバのリストチェック | GTCO（両方） |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| レポートディスクリプタのパース（攻撃者が入力を制御する） | — | あり | — | — | あり | あり |
| 空かもしれないリストへの `list_entry()`/`container_of()` | — | あり | — | — | あり | — |
| 破棄前にタイマーを非同期に停止（または停止しない） | — | — | あり | あり | — | — |
| 同じ形のバグが別の場所で何年も隔てて再発 | — | あり（→ ケース3） | あり（↔ ケース2） | あり（↔ ケース1） | あり（← 型混同） | あり（それ自体の中で） |
| 実環境での悪用が文書化されている | あり | — | — | — | — | — |

**6件のうち2件は「空になりえないかのようにリストを走査する」という同じ誤りで、約3年を隔てて2つの異なる層で起きている。** [`hid_validate_values()` の型混同](war-stories/hid-validate-values-type-confusion.md)（2023年）は、HIDコアがレポートリストに対して、空でないことを確認せずに `list_entry()` を呼ぶものである。ケース3の14個のフォースフィードバックドライバ（2019年）は、probe時に `hid->inputs` に対して同一のことを、約3年*早く*行っていた。呼び出し箇所は異なるが、同じサブシステムにおけるカーネルのリストプリミティブの使い方に、同じ根本的な穴があった。2019年の事例を塞いだことは、HIDコア自身にある2023年の事例を見つけられたはずの監査にはつながらなかったようである。

**6件のうち2件は、無関係な2つのサブシステムで破棄時のタイマー停止を誤ったもので、単に同じカテゴリに属するのではなく、同じ形のバグの再発である。** BT HIDPのセッション破棄（ケース1）と `ff-memless` のデバイス破棄（ケース2）はどちらも、タイマーのコールバックが触るメモリを解放する前に、非同期のタイマー停止を呼ぶか、停止そのものを欠いていた。つまり `del_timer_sync()` ではなく `del_timer()` を使うか、停止呼び出しがまったくなかった。どちらの修正も1行である。同期版に差し替えるか、欠けていた呼び出しを追加するかである。

**GTCOドライバは、他のケースとの対ではなく、内部でペアになっている唯一の事件である。** 構造的に無関係な2つのバグ（マルチバイト読み出しの境界チェックの欠落と、上限のないインデントカウンタ）が同じ関数の中にあり、2年足らずを隔てて表面化した。1つ目を修正しても、2つ目を見つけられるほど関数の残りを丁寧に読むことにはつながらなかった。

**実環境での悪用が文書化されているのはレポートバッファのリークだけであり、それはこのページで単体として最も深刻なバグだからではない。** CVE-2023-54120（CVSS 8.8）とCVE-2023-1073（型混同）は、どちらも単体のプリミティブとしてはおそらくより危険である。このリークが注目されたのは、GoogleのThreat Analysis Groupが、修正が出荷される前に商用フォレンジック機器メーカーのハードウェアが実際に動作する携帯電話ロック解除エクスプロイトで使った、少なくとも3つのゼロデイの1つである可能性が高いと評価したためである。これはAmnesty InternationalのSecurity Labが文書化しており、その報告自体も、成功したチェーンにどの具体的なデバイスが含まれていたかは不明だと記している。「単体で最も深刻」と「実際に使われる可能性が最も高い」は別の順位付けだということを思い出させる事例である。

## 関連項目 {#see-also}

- [入力サブシステム](README.md) — `struct input_dev`、イベントモデル、evdev
- [HIDサブシステム](hid.md) — レポートディスクリプタ、パース、そして上記のすべての事件が存在するドライバモデル
- [ALSAの事件簿](../alsa/war-stories.md) — 似た形をした兄弟分のデバイスサブシステムの事件ページ。大半がCVEで、大半がディスクリプタのパースと破棄時のロックのバグである
- [タイマーとタイマーホイール](../interrupts/timers.md) — `del_timer()` と `del_timer_sync()` の違い、およびケース1と2の背後にある非同期と同期の破棄の区別
- [ロック](../locking/README.md) — ロックの範囲とクリティカルセクションのバグに関する一般的な背景
