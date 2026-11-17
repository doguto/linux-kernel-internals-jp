# 入力サブシステム: あらゆるキーボード、マウス、タッチスクリーンのための1つのモデル {#the-input-subsystem-one-model-for-every-keyboard-mouse-and-touchscreen}

> LinuxがPS/2キーボード、USBマウス、I2Cタッチスクリーンをどのように同じ `struct input_event` ストリームに変えるか。HIDクラスのすべてのデバイスが共有するカーネルレベルの1つのオブジェクトモデルがあり、`/dev/input/eventN` がユーザー空間への統一された受け渡し点となる

## はじめに {#getting-started}

あらゆる入力デバイスは、突き詰めれば離散的な状態変化のストリームである。キーが押された、ホイールが2クリック回った、指が (412, 900) に触れた、といったものだ。しかし、それらの変化を生み出すハードウェアは極めて多様である。PS/2コントローラ、USB HIDデバイス、I2C/SPIタッチコントローラ、Bluetooth周辺機器、GPIOに配線されたボタンなどがある。もしドライバごとに独自のioctlセットと独自のデバイスノードの規約を発明していたら、あらゆるウィンドウシステムとあらゆるアプリケーションが、それぞれに対してドライバ固有のバックエンドを必要とすることになる。`drivers/input/` 以下にある**入力サブシステム**は、それを防ぐために存在する。すべてのドライバに1つの共有オブジェクト（`struct input_dev`）、1つのイベント構造体（`struct input_event`）、そしてイベント型の小さな語彙を1つ与える。これにより、ユーザー空間から見ると、キーボード、マウス、ジョイスティック、タッチスクリーンはすべて同じ種類のもの、つまり型とコードとタイムスタンプを持つイベントのストリームを発するキャラクタデバイスに見える。

### 入力サブシステムが解決する問題 {#the-problem-the-input-subsystem-solves}

このモデルが存在する前は、個々のドライバサブシステム（コンソールキーボードドライバ、PS/2マウスドライバ、ジョイスティックドライバ）がそれぞれ独自のインターフェースを公開しており、キーボードやポインタの入力を欲しいアプリケーションは、どれと話しているのかを知っている必要があった。入力サブシステムのコアは `drivers/input/input.c` にあり、データ構造は [`include/linux/input.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/input.h) と [`include/uapi/linux/input.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/input.h) で宣言されている。入力サブシステムは問題を3つの層に分割する。

- **デバイスドライバ**（`drivers/input/keyboard/`、`drivers/input/mouse/`、`drivers/input/touchscreen/`、`drivers/hid/` ほか多数）は特定のハードウェアとのやりとりの仕方を知っており、生の割り込みやレポートを型付きのイベントに変換する。デバイス固有のドライバの作者が書くのはこの層である。
- **入力コア**は `struct input_dev` と `struct input_handler` を所有し、両者の間の交換台となる。ドライバが報告するすべてのイベントは、そのデバイスに現在アタッチされているすべてのハンドラにディスパッチされる。
- **入力ハンドラ**はイベントストリームを消費する。ほぼすべてのものが関心を持つのは **evdev** で、生のイベントストリームをキャラクタデバイスとしてユーザー空間に公開する。より古く用途の狭い2つのハンドラ、**mousedev**（`/dev/input/mice`、合成されたPS/2マウスプロトコルのストリーム）と **joydev**（`/dev/input/jsN`、レガシーなジョイスティックAPI）も互換性のために今も存在するが、現代的なものはすべて汎用インターフェースであるevdevの上に構築されている。

その結果、ドライバの作者は1つのAPIを学ぶだけで、新しいデバイスをシステム上のあらゆる入力対応プログラムから使えるようにでき、ユーザー空間のプログラムは1つのイベント形式を学ぶだけで、下にあるバスやプロトコルに関係なく、あらゆる入力デバイスから読み取れる。

### コアのオブジェクトモデル: `struct input_dev` {#the-core-object-model-struct-input_dev}

ドライバの仕事は、`struct input_dev` を1つ作り、それが生成できるイベントの種類を記述して、登録することである。3つの呼び出しでライフサイクルのほぼ全体がまかなわれる。

1. **`input_allocate_device()`**（またはdevresで管理される **`devm_input_allocate_device(dev)`**。こちらは入力デバイスの寿命を所有者の `struct device` に結び付け、明示的な登録解除を必要としない）は、`struct input_dev` を割り当ててゼロ初期化し、そのmutexと `event_lock` spinlockを初期化し、`dev_set_name(&dev->dev, "input%lu", ...)` を一度呼んで恒久的なsysfs上の識別名を与える。この `inputN` の番号は `input_register_device()` によって変更されることはなく、デバイスの生涯を通じて固定される。ドライバの人間が読めるラベルは別のフィールド `dev->name`（`/proc/bus/input/devices` の `N: Name=` 行）にあり、登録処理がこれを使って何かをリネームすることはない。その `inputN` エントリがsysfsのどこに現れるかは、デバイスに親があるかどうかにも依存する。`devm_input_allocate_device(dev)` は `input->dev.parent = dev` を設定するので、この方法で登録されたハードウェアに裏付けられたデバイスは、物理的な親のsysfsパスの下に現れる（`/sys/class/input/` からのシンボリックリンクが付く）。`/sys/devices/virtual/input/` の直下に置かれるのは、親を持たないデバイスだけである。
2. ドライバは**ケーパビリティ**、つまりデバイスが生成できるイベント型と、各型の中のコードを、デバイスのケーパビリティビットマップにビットを立てることで宣言する。これは直接行うこともできる。

   ```c
   __set_bit(EV_KEY, input_dev->evbit);
   __set_bit(KEY_A, input_dev->keybit);
   __set_bit(EV_REL, input_dev->evbit);
   __set_bit(REL_X, input_dev->relbit);
   ```

   あるいはヘルパー `input_set_capability(dev, type, code)` を通じて行うこともでき、こちらは型ごとのビット（`keybit`、`relbit`、`absbit`、`mscbit`、`swbit`、`ledbit`、`sndbit`、`ffbit`）*と* `dev->evbit` の対応するビットを1回の呼び出しで立てる。絶対軸については、`input_set_abs_params(dev, axis, min, max, fuzz, flat)` がさらにその軸の `struct input_absinfo` を割り当て、`minimum`、`maximum`、`fuzz`、`flat` を埋める。`value` はゼロのままにされ（ドライバが後で通常の `input_report_abs()` 呼び出しで報告する）、`resolution` はデバイスが提供する場合、別途 `input_abs_set_res(dev, axis, res)` 呼び出しが必要である。すべての `EV_ABS` 軸は `absinfo` エントリを必要とし、`input_register_device()` は、`EV_ABS` ケーパビリティを宣言しているのに `absinfo` 配列を持たないデバイスの登録を拒否する。
3. **`input_register_device()`** はデバイスを確定させる。`EV_SYN` ビットを無条件に立て（ドライバが宣言を覚えていたかどうかにかかわらず、すべての入力デバイスは同期イベントを発することが求められる）、予約済みの `KEY_RESERVED` キーコードを除外し、`device_add()` を呼んでデバイスをsysfsで見えるようにし、登録済みの入力ハンドラのリストを走査して、`match()`/`id_table` がこのデバイスを受け入れるものをすべてアタッチする。evdevのノードが現れるのはこのためである。

`struct input_dev` 自体（`include/linux/input.h` で定義）は、ほぼ全体が `input-event-codes.h` の対応する `*_CNT` 定数でサイズが決まるケーパビリティビットマップでできている。`evbit[BITS_TO_LONGS(EV_CNT)]`、`keybit[BITS_TO_LONGS(KEY_CNT)]`、`relbit`、`absbit`、`mscbit`、`ledbit`、`sndbit`、`ffbit`、`swbit` である。これに加えて、識別用フィールド（`name`、`phys`、`uniq`、`id`）、現在の状態のミラー（`key`、`led`、`snd`、`sw`。押されている/点灯している/アクティブな現在の状態のビットマップ）、最初/最後のハンドラがデバイスをオープン/クローズしたときにコアが呼ぶ `open()`/`close()` のペア（これによりドライバは、ポーリングスレッドの開始やIRQの要求を、実際に何かがイベントを欲しがるまで遅らせることができる）、そしてドライバモデルの一級市民にするための埋め込みの `struct device dev` がある。

### イベントモデル {#the-event-model}

カーネル内でもユーザー空間へ渡す際でも、すべてのイベントは1つの `struct input_event`（`include/uapi/linux/input.h`）である。

```c
struct input_event {
	struct timeval time;
	__u16 type;
	__u16 code;
	__s32 value;
};
```

上の `time` フィールドはユーザー空間向けの形であり、ほぼすべてのユーザー空間プログラムが実際に目にするのはこれである。64ビットビルドはどれも無条件にこの形になり、通常の32ビットビルドも同様である。カーネル内のコード、あるいはY2038対策の64ビット `time_t` ABI（`__USE_TIME_BITS64`。libcがアプリケーションに代わって定義するマクロ）を明示的に選択した32ビットのユーザー空間ビルドだけが、代わりに2つの単純な `__kernel_ulong_t` フィールド `__sec` と `__usec` を見る。これらには `time.tv_sec`/`time.tv_usec` ではなく `ev.input_event_sec`/`ev.input_event_usec`（関数呼び出しではなく、これらのフィールドの別名となるオブジェクト形式マクロ）としてアクセスする。フィールドのレイアウトは `include/uapi/linux/input.h` のプリプロセッサ条件分岐で選ばれ、構造体の受け渡し上のサイズが読み取り側プロセスの期待と一致するようになっている。

`type` は固定された、比較的小さなイベントクラスの集合のいずれかである（`include/uapi/linux/input-event-codes.h`）。

| 型 | 意味 | コードの例 |
|---|---|---|
| `EV_SYN` | 同期マーカー。同時に起きた変化の1つのパケットを次のパケットから区切る | `SYN_REPORT`、`SYN_MT_REPORT`、`SYN_DROPPED` |
| `EV_KEY` | キーとボタン。値 `0` = 離す、`1` = 押す、`2` = オートリピート。`input_report_key()` を通じて報告するドライバは `0`/`1` しか送れない。このラッパー内の `!!value` 正規化により、ドライバのコードから `2` には到達できない。オートリピートのイベントは、代わりに入力コア自身のリピートタイマーが合成する | `KEY_A`、`BTN_LEFT`、`BTN_TOUCH` |
| `EV_REL` | 相対軸の動き。マウスの移動のような*差分* | `REL_X`、`REL_Y`、`REL_WHEEL` |
| `EV_ABS` | 絶対軸の位置。`[minimum, maximum]` の範囲内の新しい値 | `ABS_X`、`ABS_Y`、`ABS_MT_*` マルチタッチコード |
| `EV_MSC` | 他に当てはまらない雑多なイベント。例えば生のスキャンコード | `MSC_SCAN` |
| `EV_SW` | 状態を持つ2値のスイッチ（キーのような瞬間的なものではない） | `SW_LID`、`SW_TABLET_MODE` |
| `EV_LED` | LEDの状態の設定/問い合わせ。デバイス*へ*流れる | `LED_NUML`、`LED_CAPSL` |
| `EV_SND` | 単純なサウンドイベント。これもデバイスへ流れる | `SND_CLICK`、`SND_BELL` |
| `EV_REP` | オートリピートの設定（遅延/周期）。`EVIOCSREP` によってデバイス*へ*流れる | `REP_DELAY`、`REP_PERIOD` |
| `EV_FF` | フォースフィードバックのエフェクト制御 | フォースフィードバックエフェクトのアップロード/再生/消去 |
| `EV_PWR` | 電源管理イベント。一握りのドライバが使う | ドライバ固有 |
| `EV_FF_STATUS` | フォースフィードバックのエフェクト状態の報告。デバイス*から*流れる | エフェクトの再生状態 |

`code` はその型の特定のメンバー（どのキーか、どの軸か）を選び、`value` はペイロードである。その意味は上の表が示すとおり `type` に依存する。

ドライバは、`include/linux/input.h` で定義された、コアのディスパッチャ `input_event()` を包む小さなインラインラッパー群でイベントを報告する。

```c
input_report_key(input_dev, BTN_LEFT, 1);   /* -> input_event(dev, EV_KEY, BTN_LEFT, !!1) */
input_report_rel(input_dev, REL_X, -3);     /* -> input_event(dev, EV_REL, REL_X, -3)   */
input_report_abs(input_dev, ABS_X, 512);    /* -> input_event(dev, EV_ABS, ABS_X, 512)  */
input_sync(input_dev);                      /* -> input_event(dev, EV_SYN, SYN_REPORT, 0) */
```

`input_event()` は、デバイスが与えられた型を実際に `evbit` で宣言しているかを確認し（宣言されていないイベントは黙って捨てられる。ケーパビリティの `set_bit` を忘れたドライバは、自分のイベントがどこにも届かないことに気づくことになる）、それから `dev->event_lock` を取って内部のディスパッチパスに引き渡す。ほとんどの型について、ディスパッチパスはデバイスの現在の状態のビットマップとの重複除去も行う。`EV_KEY`、`EV_SW`、`EV_LED`、`EV_SND` の報告で、値がすでに記録された状態（`key`/`sw`/`led`/`snd`）と一致するものは転送されずに捨てられる。そのため、同じ値で `input_report_key()` を2回続けて呼んでも、ハンドラ側では1つのイベントにしかならない（オートリピート、`value == 2` は意図的な例外で、常に通過する）。チェックを通過したイベントは、現在の状態のビットマップ（`EV_ABS` の場合は現在の `absinfo[axis].value`）を更新し、アタッチされているすべてのハンドラのためにキューに入れられる。

**`input_sync()`** は見た目以上に重要である。個々の `input_report_*()` 呼び出しは*変化*を記述するもので、複数の変化が同じ物理的な瞬間に属することがある。マウスを斜めに動かすのは、同時に起きた `REL_X` と `REL_Y` のイベントである。`EV_SYN`/`SYN_REPORT` は、「直前の `SYN_REPORT` 以降のすべてが1つのアトミックな更新として起きた」ことを読み手に伝えるマーカーである。ドライバは、関連する報告のバッチごとに一度 `input_sync()` を呼ぶことが期待されており、報告1つごとに呼ぶのではない。

### evdevのユーザー空間インターフェース {#the-evdev-userspace-interface}

**evdev**（`drivers/input/evdev.c`）は、生のイベントストリームを実質的にフィルタリングせずにユーザー空間へ渡す入力ハンドラである。evdevがアタッチする各入力デバイスには、キャラクタデバイスノード `/dev/input/eventN` が与えられ、マイナー番号は `EVDEV_MINOR_BASE`（64）から割り当てられる。docs.kernel.orgの入力の概要は、これを64〜95のレガシーなevdevマイナー番号範囲として説明しており、32個を超える入力デバイスを持つシステム向けに、それを超える動的マイナー番号も利用できる。

ユーザー空間のプログラムはノードをオープンして `read()` を呼ぶ。各 `read()` は、1つ以上の完全な `struct input_event` レコードを返す（部分的なレコードは決して返さない。`evdev_read()` は1イベントのサイズより小さいバッファを `-EINVAL` で拒否する。ただし `count == 0` の呼び出しは拒否されず、何もしないプローブとして通される）。これらは、下にあるデバイスがイベントを生成するたびにカーネルが埋める、オープンしたファイルディスクリプタごとのリングバッファから取り出される。比較されるサイズは単純な `sizeof(struct input_event)` ではなく `input_event_size()` であり、呼び出し元が64ビットカーネル上のcompatモード（例えば32ビット）のプロセスである場合には、代わりに32ビット時刻のcompat用 `struct input_event_compat` のサイズを返す。これにより、境界がそのプロセスが実際に読むイベントレイアウトと一致する。読み出しはデフォルトでブロックし、`O_NONBLOCK` の下ではバッファが空なら `-EAGAIN` を返す。`read()` 以外に、evdevは一連の `ioctl()` をサポートする。デバイスのケーパビリティビットマップを問い合わせる `EVIOCGBIT`、絶対軸の `input_absinfo` を読み書きする `EVIOCGABS`/`EVIOCSABS`、デバイスのイベントストリームの排他的な所有権を取る `EVIOCGRAB`（コンポジタが自身のキー入力を、聞いているかもしれない他の何かに漏らさないために使う）などがあり、すべて `include/uapi/linux/input.h` で宣言されている。

`/dev/input/eventN` と直接やりとりするものは、まさにその目的のために作られた2つのユーザー空間の層を除いてほとんどない。**libevdev** は、`ioctl()` の定型処理を引き受けてデバイス状態のローカルなミラーを保持する薄いCラッパーである。**libinput** はlibevdevの上に構築されており、生のイベントを、Waylandコンポジタや Xサーバーが消費するための、より高レベルなセマンティクス（ポインタの加速、タップでのクリック、ジェスチャー認識、デバイスのquirkの回避策）に変換する。このページはevdevまでで止める。libevdev/libinputの内部はカーネルではなくユーザー空間のものであり、ここでの対象外である。

### デバイスの列挙 {#device-enumeration}

イベントデータを読まずに、どんな入力デバイスが存在するかを調べられる場所が3つある。

- **`/proc/bus/input/devices`** — プレーンテキストのダンプで、登録された `struct input_dev` ごとに1つのスタンザがあり、`drivers/input/input.c` の `input_devices_seq_show()` が生成する。各スタンザには固定のプレフィックス文字がある。`I:` はバス/ベンダー/製品/バージョン、`N:` はデバイスの `name`、`P:` はその `phys` パス、`S:` はsysfsパス、`U:` は `uniq` 文字列、`H:` はアタッチされているハンドラのリスト（例: `sysrq kbd event3`）で、その後に一連の `B:` ビットマップ行が続き、それぞれが示すコードの16進ビットマップである。`B: PROP=...`（入力プロパティ、例: `INPUT_PROP_POINTER`）と `B: EV=...`（`evbit` マスクそのもの）は常に出力される。さらに `B: KEY=...`/`REL=...`/`ABS=...`/`MSC=...`/`LED=...`/`SND=...`/`FF=...`/`SW=...` の行は、デバイスが `evbit` で実際に宣言したイベント型についてのみ出力される。
- **`/sys/class/input/`** — 登録されたすべての入力デバイスと、ハンドラが作成したすべてのノード（`inputN`、`eventN`、`mouseN`、`jsN`）が、ドライバモデルのデバイスツリーへのシンボリックリンクとしてここに現れ、カーネルの `input` デバイスクラス（`drivers/input/input.c` の `const struct class input_class`、`.name = "input"`）の下に登録される。これは `/proc/bus/input/devices` が示すのと同じ情報だが、平坦なテキストダンプではなく、通常のsysfs属性ファイルとしてたどれる。
- **udevによる命名** — udevは上記のsysfsツリーを消費し、ユーザー空間のツールが実際に使う、安定した人間が読めるシンボリックリンクを `/dev/input/by-id/` と `/dev/input/by-path/` の下に作成する。素の `/dev/input/eventN` の番号は、再起動やホットプラグのイベントをまたいで同じ物理デバイスに結び付いたままである保証がないためである。

### 絶対デバイスと相対デバイス、そしてマルチタッチ {#absolute-vs-relative-devices-and-multitouch}

`EV_REL`/`EV_ABS` の区別が存在するのは、「ポインタはどこにあるか」がハードウェアによって本当に異なる意味を持つからである。マウスは**相対**的な動き（`REL_X`、`REL_Y`）を報告する。マウスはカーソルが画面のどこにあるかを知らず、前回の報告からどれだけ動いたかだけを知っており、受け取る側（入力コア、次いでコンポジタ）がそれらの差分を位置へと積算する。タッチスクリーンやペンタブレットは**絶対**位置（`ABS_X`、`ABS_Y`）を報告する。ハードウェアは自身の座標空間内のどこに接触があるかを正確に知っており、各報告は差分ではなく新たな正式な値である。

1点のタッチなら `ABS_X`/`ABS_Y` にきれいに収まるが、複数の指を同時に感知できるタッチスクリーンやトラックパッドはそうはいかない。1組の絶対軸では一度に複数の位置を表せないからである。カーネルの**マルチタッチ（MT）プロトコル**は、一連の `ABS_MT_*` コード（`ABS_MT_POSITION_X`、`ABS_MT_POSITION_Y`、`ABS_MT_TOUCH_MAJOR`、`ABS_MT_PRESSURE`、`ABS_MT_TRACKING_ID` など、`ABS_MT_SLOT` から始まるもの）と**スロット**モデルでこれを解決する。`ABS_MT_SLOT` は、続く `ABS_MT_*` イベントがどの接触（スロット）を記述するかを選び、`ABS_MT_TRACKING_ID` はその接触に、触れている間ずっと続く永続的なIDを与える。非負の値はアクティブな接触を意味し、`-1` はそのスロットが空になったことを意味する。ドライバはスロットを順にたどり、`input_mt_slot(dev, slot)` を呼んで1つを選び、その接触の `ABS_MT_*` 軸を、他の絶対軸を報告するのとまったく同じように報告する。このスロットベースの方式は公式に**タイプB**と呼ばれる。匿名の接触ごとのデータを `SYN_MT_REPORT` マーカーで区切って送る、より古い**タイプA**の方式もプロトコルの歴史にはまだ存在し、廃止扱いとされている。カーネル自身のドキュメントは、ツリー内のすべてのドライバがタイプBに変換済みだと主張しているが、これは正確ではない。`drivers/input/touchscreen/auo-pixcir-ts.c`、`drivers/input/touchscreen/usbtouchscreen.c`、`drivers/hid/hid-ntrig.c` を含む一握りのドライバは、今も `input_mt_sync()` を呼んでタイプAの報告を発している。大多数のドライバはタイプBを使っており、新しいドライバが目指すべきなのはこの方式である。スロットとトラッキングIDの仕組みの詳細は [Kernel docs: Multi-touch (MT) protocol](https://docs.kernel.org/input/multi-touch-protocol.html) を参照。[HID: Human Interface Deviceレイヤー](hid.md)では、USB、Bluetooth、I2C HID経由で届くデバイス（マルチタッチのタッチスクリーンを含む）について、そもそもHIDレポートディスクリプタがどのようにパースされ、`input_dev` のケーパビリティに対応付けられるかを扱う。

### アーキテクチャ: ハードウェアからユーザー空間まで {#architecture-from-hardware-to-userspace}

```
  Hardware (PS/2 controller, USB HID device, I2C/SPI touch controller, ...)
          │
          ▼
  Device driver's probe()        e.g. atkbd, usbhid, a touchscreen driver
          │
          ▼
  input_allocate_device() / devm_input_allocate_device()
          │  allocates struct input_dev
          │
          ├── set_bit(EV_KEY, dev->evbit); set_bit(KEY_A, dev->keybit); ...
          │      or input_set_capability() / input_set_abs_params()
          │      declares the capability bitmaps (evbit/keybit/relbit/absbit/...)
          │
          ▼
  input_register_device()
          │  device_add(); EV_SYN forced on; attaches every matching
          │  registered input_handler to this device
          ▼
  Input core (drivers/input/input.c)
          │  driver calls input_report_key()/_rel()/_abs() + input_sync()
          │  → input_event() → dispatch under dev->event_lock
          │
          ├── evdev handler   → /dev/input/eventN   struct input_event stream
          ├── mousedev        → /dev/input/mice      (legacy PS/2-style protocol)
          └── joydev          → /dev/input/jsN        (legacy joystick API)
          ▼
  Userspace: libevdev / libinput
          │  read()s struct input_event records; libinput adds pointer
          │  acceleration, gesture recognition, device quirks, etc.
          ▼
  Wayland compositor / X server / plain read() by a console tool
```

### このセクションの残り {#the-rest-of-this-section}

- **[HID: Human Interface Deviceレイヤー](hid.md)** — USB/Bluetooth/I2C HIDのレポートディスクリプタがどのようにパースされ、`input_dev` のケーパビリティに対応付けられるか、HIDドライバモデル、そしてトランスポートの階層化。
- **[事件簿](war-stories.md)** — 入力/HIDサブシステムで実際に起きた事件。

### 前提知識と関連分野 {#prerequisites-and-neighbors}

入力ドライバは通常の[Linuxデバイスモデル](../drivers/device-model.md)の市民である。大半は（HIDレイヤー経由の）[USBデバイス](../usb/README.md)、I2C/SPIの[プラットフォームデバイス](../drivers/platform-driver.md)、あるいは古いハードウェアでは、バスの抽象化をまったく持たないISA/PS2デバイスである。`/dev/input/*` ノードは、他のドライバと同じ `cdev`/`file_operations` の仕組みで登録される、普通の[キャラクタデバイス](../drivers/chardev.md)である。読む順番: このページ、次にUSB/Bluetoothの入力ハードウェアを扱うなら[HID](hid.md)。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/input.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/input.h) — `struct input_dev`、`struct input_handler`、`struct input_handle`、およびドライバ向けの登録/報告API
- [include/uapi/linux/input.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/input.h) — `struct input_event`、`struct input_absinfo`、`EVIOC*` ioctl
- [include/uapi/linux/input-event-codes.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/input-event-codes.h) — すべての `EV_*`、`KEY_*`、`REL_*`、`ABS_*`、`SYN_*` 定数
- [include/linux/input/mt.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/input/mt.h) — `struct input_mt`、`input_mt_slot()`、`input_mt_report_slot_state()`
- [drivers/input/input.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/input/input.c) — 入力コア: `input_register_device()`、`input_event()`/ディスパッチ、`/proc/bus/input/devices`、`input` デバイスクラス
- [drivers/input/evdev.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/input/evdev.c) — evdevハンドラ: `/dev/input/eventN`、`evdev_read()`、クライアントごとのイベントリングバッファ
- [drivers/input/mousedev.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/input/mousedev.c) · [drivers/input/joydev.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/input/joydev.c) — レガシーな `/dev/input/mice` と `/dev/input/jsN` のハンドラ

### 関連ページ {#related-pages}

- [HID: Human Interface Deviceレイヤー](hid.md) — レポートディスクリプタ、パース、HIDドライバモデル、USB/Bluetooth/I2Cのトランスポートの階層化
- [事件簿](war-stories.md) — 上記のサブシステムで実際に起きた事件
- [Linuxデバイスモデル](../drivers/device-model.md) · [キャラクタデバイスとmiscデバイス](../drivers/chardev.md) · [プラットフォームドライバ](../drivers/platform-driver.md) — 入力サブシステムが載っているドライバコアの層
- [USB](../usb/README.md) — ほとんどのHID入力デバイスが接続されるバス

### LWN記事 {#lwn-articles}

- [An update on the input stack](https://lwn.net/Articles/801767/) — libinputの進化、高解像度スクロールイベント、そしてevdevの上に構築されたユーザー空間の層

### 外部資料 {#external}

- [Kernel docs: Input subsystem introduction](https://docs.kernel.org/input/input.html) — `input_event` 構造体、evdev、そしてドライバ/コア/ハンドラのアーキテクチャ
- [Kernel docs: Event codes](https://docs.kernel.org/input/event-codes.html) — すべての `EV_*` 型の完全なセマンティクス
- [Kernel docs: Multi-touch (MT) protocol](https://docs.kernel.org/input/multi-touch-protocol.html) — タイプAとタイプBの違い、スロット、`ABS_MT_TRACKING_ID`
- [Kernel docs: Input subsystem driver API](https://docs.kernel.org/driver-api/input.html) — ドライバ向け関数の完全なリファレンス
