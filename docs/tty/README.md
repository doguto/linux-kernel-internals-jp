# TTYサブシステム: 実在・仮想を問わずあらゆる端末を1つのモデルで {#the-tty-subsystem-one-model-for-every-terminal-real-or-virtual}

> Linuxはいまも数十年前のテレタイプライタのモデル（低レベルドライバとユーザー空間の間にラインディシプリンが位置する）を使い、物理シリアルポート、擬似端末、システムコンソールを1つの`tty_struct`/`tty_driver`オブジェクトモデルの下に統一している

## はじめに {#getting-started}

あらゆる端末セッション（組み込みボード上の物理シリアルコンソール、`ssh`ログイン、GUIのターミナルエミュレータのウィンドウ、さらにはスクロールしていくカーネル自身のブートメッセージまで）は、カーネルから見ればすべて **TTY** である。この名前は実際のハードウェアに由来する。*テレタイプライタ* とは、送られてきた文字を印字し、人間がタイプしたものを何であれ送り返す電気機械式の装置である。Linuxは（それ以前のあらゆるUnixと同様に）、今世紀に本物のテレタイプをコンピュータにつないだ人などほとんどいないにもかかわらず、文字指向で行バッファリングされるすべての端末I/Oの形として、いまもその数十年前のモデルを使っている。カーネル自身のドキュメントは率直にこう述べている。"Teletypewriter (TTY) layer takes care of all those serial devices. Including the virtual ones like pseudoterminal (PTY)."（訳: テレタイプライタ（TTY）層は、それらすべてのシリアルデバイスを扱う。擬似端末（PTY）のような仮想的なものも含めて。）

`drivers/tty/`は、カーネルの中でも最も古く、そしてそれを保守している人々自身が認めるとおり、最も扱いの難しい一角の1つでもある。TTYのリグレッションに関するLWNの回顧記事は、"there are dark areas of the kernel where only the bravest hackers dare to tread... arguably, no part of the kernel is darker and scarier than the serial terminal (TTY) code."（訳: カーネルには最も勇敢なハッカーしか足を踏み入れようとしない暗い領域がある……おそらく、シリアル端末（TTY）のコードほど暗く恐ろしい部分はカーネルにない。）という一文で始まっている。この評判は、このようなページにとって有用な背景である。以下で説明するオブジェクトモデルは安定しており、よく文書化されているが、それを実装するコードは数十年にわたって積み重なってきたものであり、慎重さが報われる。

### TTYサブシステムが解決する問題 {#the-problem-the-tty-subsystem-solves}

まったく異なる3つのものが、どれもユーザー空間からは「端末」に見える必要がある。

- **物理シリアルポート** — UARTチップの背後にある実際のRS-232回線。サーバー（帯域外管理用）や組み込みボードでいまも一般的である。
- **擬似端末（pty）** — 背後にハードウェアがまったくない、純粋にソフトウェアだけのマスター/スレーブのペア。*あらゆる* ターミナルエミュレータ（xterm、GNOME Terminal、VS Codeの統合ターミナル）、あらゆる`ssh`セッション、あらゆる`screen`/`tmux`のペインがこの上で動いている。
- **システムコンソール** — カーネル自身のテキスト出力（ブートメッセージ、`printk()`、あれば対話シェル）が現在向けられているデバイス。これ自体が物理シリアル回線、画面上の仮想コンソール、ptyのいずれでもありうる。

共通のモデルがなければ、これらはそれぞれ、あらゆる端末が扱わなければならない事柄を独自に再実装する必要がある。生のバイトストリームを編集可能な行に変換すること（バックスペースで最後の文字を消す）、入力を1行ずつバッファするか1バイトずつ渡すかを決めること、タイプされた文字をユーザーにエコーし返すこと、ユーザーが割り込みキーを押したときにシグナルを生成すること、そしてウィンドウサイズのネゴシエーションである。LinuxのTTY層は、`drivers/tty/`を根とし、データ構造を`include/linux/tty.h`、`include/linux/tty_driver.h`、`include/linux/tty_ldisc.h`に持つ。これを3つの層に分解することで、デバイスファミリーごとに書く必要があるのは最下層（実際のUART、ptyの相手、または画面との間でバイトをやり取りする部分）だけになる。それより上はすべて共有される。

### 3層スタック {#the-three-layer-stack}

1. **低レベルドライバ。** 実際にバイトを動かす層である。実シリアルハードウェア用のUARTドライバ（`drivers/tty/serial/`）、ptyのマスター/スレーブ実装（`drivers/tty/pty.c`）、または画面用の仮想コンソール/フレームバッファコンソールドライバ（`drivers/tty/vt/`）がこれにあたる。これらはそれぞれ`struct tty_driver`を登録し、`struct tty_operations`を実装する。これはttyコアが`open()`、`write()`、`ioctl()`などのために呼び出すvtableである。
2. **ラインディシプリン。** ドライバと、ユーザー空間から見えるキャラクタデバイスインターフェースとの間に位置し、ラインディシプリン（`struct tty_ldisc_ops`）が生のバイトストリームを解釈する。デフォルトであり、ほぼすべての対話端末が使うのが **N_TTY** である（`N_TTY`は`0`で、`include/uapi/linux/tty.h`で定義されている）。N_TTYはカノニカル（「cooked」）モードとrawモード、文字のエコー、制御文字からのシグナル生成（最も有名なのはCtrl-Cを`SIGINT`に変換すること）を実装する。新しく開かれたttyはすべて自動的にN_TTYを得る。`drivers/tty/tty_ldisc.c`の`tty_ldisc_init()`は無条件に`tty_ldisc_get(tty, N_TTY)`を呼び、また別のラインディシプリンへの切り替えで問題が起きたときにttyが戻るフォールバック先でもある。プロセスは`TIOCSETD` ioctlで別のラインディシプリンに差し替えられる。カーネルにはそのほかに20数種類（`N_PPP`、`N_HDLC`、`N_SLIP`、Bluetooth UART用の`N_HCI`、`N_GSM0710`など）が同梱されており、これらはシリアル回線を、人間向けの端末ではなくプロトコルのフレーマとして転用する。
3. **`tty_struct`と`tty_driver`。** 他の2つを結びつけるコアのカーネルオブジェクトである。`struct tty_driver`はドライバファミリー（例えば「シリアルドライバ」や「ptyスレーブドライバ」）を表し、`struct tty_struct`は開かれた1つのttyインスタンスを表す。後者は、現在のラインディシプリンへのポインタ（`ldisc`）、ドライバ（`driver`、`ops`）、端末設定（`termios`）、その他の動作中の状態を保持する。

### コアの構造体 {#core-structures}

**`struct tty_struct`**（`include/linux/tty.h`）は開かれたttyごとに割り当てられ、（埋め込まれた`kref`による参照カウントで）最後の参照がなくなるまで存在する。主なフィールドは次のとおり。

```c
struct tty_struct {
	struct kref kref;
	int index;
	struct device *dev;
	struct tty_driver *driver;
	struct tty_port *port;
	const struct tty_operations *ops;

	struct tty_ldisc *ldisc;
	struct ld_semaphore ldisc_sem;
	/* ... ロック: atomic_write_lock, legacy_mutex, throttle_mutex,
	 *              termios_rwsem, winsize_mutex ... */

	struct ktermios termios, termios_locked;
	char name[64];
	unsigned long flags;
	int count;
	unsigned int receive_room;
	struct winsize winsize;

	struct {
		spinlock_t lock;
		bool stopped;
		bool tco_stopped;
	} flow;

	struct {
		struct pid *pgrp;
		struct pid *session;
		spinlock_t lock;
		unsigned char pktstatus;
		bool packet;
	} ctrl;

	/* ... hw_stopped, closing, flow_change ... */
	struct tty_struct *link;
	void *disc_data;
	void *driver_data;
	/* ... */
};
```

このうちいくつかは特筆に値する。`name`は`tty_line_name()`によって組み立てられる（例: `ttyS3`）。`flags`は`TTY_THROTTLED`、`TTY_IO_ERROR`、`TTY_OTHER_CLOSED`（相手側が閉じたptyに設定される）、`TTY_HUPPED`といった状態のビットマスクである。`receive_room`は、現在のラインディシプリンが1回の`receive_buf()`呼び出しでドライバから受け入れるつもりのバイト数であり、ラインディシプリン自身が管理する（N_TTYは入力を消費するにつれて`n_tty_receive_buf_common()`で動的に更新し、ttyコアは`tty_set_termios_ldisc()`でこれをゼロにする。`n_gsm.c`や`n_hdlc.c`のような他のいくつかのラインディシプリンは、`.open()`で固定値を設定するだけである）。`ctrl.pgrp`/`ctrl.session`は、このttyをジョブ制御（`setpgrp(2)`、`setsid(2)`）のための *制御* 端末とするプロセスグループとセッションである。そして`link`は、ptyの2つの`tty_struct`（マスターとスレーブ）を互いに結びつける。

**`struct tty_driver`**（`include/linux/tty_driver.h`）は、個々の開かれたttyではなくドライバを表す。

```c
struct tty_driver {
	struct kref kref;
	struct cdev **cdevs;
	struct module *owner;
	const char *driver_name;
	const char *name;
	int name_base;
	int major;
	int minor_start;
	unsigned int num;
	enum tty_driver_type type;
	enum tty_driver_subtype subtype;
	struct ktermios init_termios;
	unsigned long flags;
	/* ... proc_entry, other, flip_wq ... */

	struct tty_struct **ttys;
	struct tty_port **ports;
	struct ktermios **termios;
	void *driver_state;

	const struct tty_operations *ops;
	/* ... */
};
```

`name`は`/dev`ノードを組み立てるのに使われるもの（`"ttyS"`、`"pty"`、`"pts"`）である。`type`/`subtype`（`enum tty_driver_type`: `TTY_DRIVER_TYPE_SYSTEM`、`_CONSOLE`、`_SERIAL`、`_PTY`、`_SCC`、`_SYSCONS`）は、これがおおよそどんな種類のドライバかをコアに伝える。`ttys`/`ports`/`termios`は回線ごとの配列で、標準の`tty_standard_install()`パスがマイナー番号で既存のttyを検索するのに使う。ドライバは`tty_alloc_driver()`でセットアップされ、`tty_set_operations()`で`ops`が取り付けられ、`tty_register_driver()`で稼働を始める。

**`struct tty_operations`**（これも`tty_driver.h`）は、ttyコアと特定のドライバをつなぐvtableである。カーネルのドキュメントコメントは`open()`と`close()`の両方に「Required method」と記しているが、実際に実行時に欠落が強制されるのは`open()`だけである（ドキュメントコメント: "if this routine is not filled in, the attempted open will fail with `ENODEV`"（訳: このルーチンが埋められていなければ、オープンの試みは`ENODEV`で失敗する））。残りは任意だが、ほとんどのドライバは要となるものを実装している。出力用の`write()`/`write_room()`/`chars_in_buffer()`、端末設定の変更に反応する`set_termios()`、フロー制御用の`throttle()`/`unthrottle()`と`stop()`/`start()`、`hangup()`、`ioctl()`、BREAKを送る`break_ctl()`、そして標準の`ttys[]`配列による検索を使わないドライバ（ptyやコンソールのドライバなど）のための`install()`/`lookup()`/`remove()`である。

**`struct tty_ldisc_ops`**（`include/linux/tty_ldisc.h`）はラインディシプリンのvtableであり、カーネルのドキュメントコメントはそのフックを2つの方向に分けている。`[TTY]`と記されたフックはttyコアから下向きに呼ばれるもので、`open()`、`close()`、`read()`、`write()`、`ioctl()`、`set_termios()`、`poll()`、`hangup()`である。`[DRV]`と記されたフックは低レベルドライバから上向きに呼ばれ、受信データを流し込む。`receive_buf()`とその新しい変種`receive_buf2()`（存在すればこちらが優先される。自動フロー制御のために、何バイト消費したかを報告し返せるからである）、`write_wakeup()`（ドライバに出力の余地ができた）、`dcd_change()`（`N_PPS` Pulse-Per-Secondディシプリンが使う）、`lookahead_buf()`（ソフトウェアフロー制御のバイトのように、通常の`receive_buf()`呼び出しが到達する前に処理する必要がある文字のための高速パス）である。N_TTYの`receive_buf()`/`receive_buf2()`の実装こそが、カノニカルモードの行編集、エコー、`ISIG`のシグナル生成が実際に行われる場所である。ローカルフラグ`ISIG`（`c_lflag`）が設定されていて、到着した文字が`INTR_CHAR(tty)`（端末に設定された割り込み文字。通常はCtrl-C）と一致すると、`drivers/tty/n_tty.c`は`n_tty_receive_signal_char(tty, SIGINT, c)`を呼び、ttyのフォアグラウンドプロセスグループに`SIGINT`を発生させる。

### コンソール: printk、ブートメッセージ、tty層 {#the-console-printk-boot-messages-and-the-tty-layer}

カーネル自身のテキスト出力（初期のブートメッセージ、`printk()`、シェルが起動する前に、そしてしばしば起動後にも、シリアルコンソールや画面に表示されるすべて）は、関連はあるが別個の仕組みを通る。`struct console`（`include/linux/console.h`）、登録されたコンソールドライバ、そして`CON_BOOT`（初期コンソール。本物のコンソールが登録されると置き換えられる）や`CON_PRINTBUFFER`（引き継ぎの間に何も失われないよう、バッファされたメッセージを再生する）のようなフラグである。これはttyコアそのものではないが、両者は意図的に結びつけられている。`struct console`には`device`コールバックがあり、

```c
struct tty_driver *(*device)(struct console *co, int *index);
```

その役割は、あるコンソールの背後にどの`tty_driver`があるかを伝えることである。これはまさに、`/dev/console`が開かれたときに使われる仕組みである。`drivers/tty/tty_io.c`の`tty_lookup_driver()`は`console_device()`を呼んで、システムコンソールとして現在登録されている`tty_driver`を見つけ、他のttyと同じようにそれを通じてttyを開く。テキスト画面を駆動する仮想コンソールドライバ（`drivers/tty/vt/`。`/dev/tty1`〜`/dev/tty63`と、現在アクティブなVTの別名`/dev/tty0`の背後にある）自体も、`type == TTY_DRIVER_TYPE_CONSOLE`を持つ普通の`struct tty_driver`である。シリアルコンソールを持つヘッドレスシステムでは、同じ`device`コールバックが、そのシリアルポートの背後にある`tty_driver`を指すだけである。この`struct console`/printkのルーティングの側（NBCONノンブロッキングコンソールの作り直し、複数のコンソールドライバの同時使用など）はそれ自体が1つのサブシステムであり、このページの範囲外である。ここでの要点は、「コンソール」は、その下では、他のあらゆるものが使うのと同じ`tty_driver`/`tty_struct`モデルの利用者の1つにすぎない、ということだけである。

### デバイスノードの命名 {#device-node-naming}

- **`/dev/ttyS0`、`/dev/ttyS1`、...** — 実際のシリアルポート。ドライバが列挙するUART回線ごとに1つ（最も一般的なケースでは`drivers/tty/serial/8250/`以下の8250/16550系ドライバ）。`ttyS(4)`のmanページは簡潔にこう述べている。"ttyS[0-3] are character devices for the serial terminal lines."（訳: ttyS[0-3]はシリアル端末回線用のキャラクタデバイスである。）
- **`/dev/pts/N`** — UNIX 98擬似端末のスレーブ。ユーザー空間は、`drivers/tty/pty.c`の`ptmx_open()`が処理するクローンデバイス`/dev/ptmx`を開いて新しいptyマスターを割り当てる。その後`devpts`ファイルシステムが、対応する`/dev/pts/N`スレーブノードを作成する。あらゆるターミナルエミュレータ、あらゆる`ssh`セッション、あらゆる`tmux`/`screen`のペインが、こうしたペアの1つである。カーネル側では、これは2つの別々の`struct tty_driver`である。`ptm_driver`（`driver_name = "pty_master"`、`name = "ptm"`、マスター側）と`pts_driver`（`driver_name = "pty_slave"`、`name = "pts"`、スレーブ側）で、`tty_driver.other`を通じて相互にリンクされている。`ptmx_open()`は`tty_init_dev(ptm_driver, index)`でマスターを割り当てる。これは、`CONFIG_LEGACY_PTYS`の下で`legacy_pty_init()`によって別途セットアップされるレガシーなBSD形式の`pty_driver`（`name = "pty"`）とは別物である。後者は`/dev/ptmx`ではなく、古い`/dev/ptyXX`/`/dev/ttyXX`のペアの背後にある。
- **`/dev/tty`** — 独立したデバイスファミリーではなく、固定の別名である。デバイス番号は`MKDEV(TTYAUX_MAJOR, 0)`、つまりメジャー`TTYAUX_MAJOR`（`5`）、マイナー`0`である。`tty(4)`のmanページによれば、"a character file with major number 5 and minor number 0... a synonym for the controlling terminal of a process, if any."（訳: メジャー番号5、マイナー番号0のキャラクタファイル……プロセスの制御端末があれば、その同義語。）これを開くと`tty_open_current_tty()`が呼ばれ、`get_current_tty()`で呼び出し元プロセスの制御ttyを検索して、それを開き直す。
- **`/dev/console`** — システムコンソールの別名。デバイス番号は`MKDEV(TTYAUX_MAJOR, 1)`（メジャー`5`、マイナー`1`）で、上で説明したのと同じく`console_device()`経由で解決される。システムがどのようにブートされたか（`console=`カーネルパラメータ）と、現在何が登録されているかに応じて、シリアルポート、仮想コンソール、あるいは（あまり一般的ではないが）ptyを指しうる。

### アーキテクチャ: ハードウェアからユーザー空間まで {#architecture-from-hardware-to-userspace}

```
  Hardware / peer object
  UART chip (8250/16550, ...)   pty peer (other half     VT/fbcon screen, or a
                                 of the master/slave       serial line acting as
                                 pair)                     the system console
          │                             │                            │
          ▼                             ▼                            ▼
  drivers/tty/serial/*          drivers/tty/pty.c            drivers/tty/vt/*
  struct tty_driver "ttyS"      tty_driver "ptm"/"pts"       console_driver
  + struct tty_operations       + struct tty_operations      + struct tty_operations
          │                             │                            │
          └─────────────────────────────┼────────────────────────────┘
                                         ▼
                 Line discipline: struct tty_ldisc_ops
                 N_TTY (default) — canonical/raw mode, echo,
                 signal generation (INTR_CHAR -> SIGINT), ...
                 swappable per-tty via TIOCSETD (N_PPP, N_HDLC, N_SLIP, ...)
                                         │
                                         ▼
                              struct tty_struct
                ldisc, termios, ops, driver, port, receive_room,
                flow{stopped}, ctrl{pgrp,session}, link (pty pairing)
                                         │
                                         ▼
            character device: open()/read()/write()/ioctl()/poll()
                                         │
          ┌──────────────────────────────┼──────────────────────────────┐
          ▼                              ▼                              ▼
    /dev/ttyS0                     /dev/pts/N                    /dev/tty1..ttyN
    real serial hw                 ptmx-cloned pty; every         virtual consoles
                                    terminal emulator/SSH          (VT layer)
                                    session runs on one

  Two more names are resolved at open() time to whichever tty above is
  currently appropriate, rather than being separate device families:

    /dev/tty      -> the calling process's controlling terminal
                     (tty_open_current_tty(), device number MKDEV(TTYAUX_MAJOR, 0))
    /dev/console  -> whichever tty_driver is the registered system console
                     (console_device(), device number MKDEV(TTYAUX_MAJOR, 1))
```

### このセクションの残りのページ {#the-rest-of-this-section}

- **[シリアルコア: あらゆるUARTを覆う1つの抽象化](serial.md)** — 実際のUARTチップのドライバが`serial_core`、`tty_driver`、`tty_port`にどう組み込まれるかを、8250/16550と`uartlite`ドライバを通じて解説する。
- **[ラインディシプリン、termios、擬似端末](line-disciplines.md)** — カノニカルモードとrawモード、`termios`、シグナル生成、ジョブ制御、擬似端末を、主にN_TTYを通じて解説する（N_TTY以外のディシプリンは簡単な概観のみ）。
- **[事件簿](war-stories.md)** — TTY/シリアルサブシステムで実際に起きた事件。

### 前提知識と隣接分野 {#prerequisites-and-neighbors}

あらゆる`/dev/tty*`、`/dev/pts/*`、`/dev/ttyS*`ノードは、その下では普通の[キャラクタデバイス](../drivers/chardev.md)であり、他のドライバと同じ`cdev`/`file_operations`の仕組みを通じて登録される。tty層の`struct tty_operations`は、その1段上に端末固有のvtableとして位置する。シリアルドライバやコンソールドライバは、普通の[Linuxデバイスモデル](../drivers/device-model.md)の一員である（SoC上のUARTならプラットフォームデバイス、増設シリアルカードならPCIデバイス）。読む順序: まずこのページを読み、次に実際のUARTハードウェアを扱うなら[シリアルコア: あらゆるUARTを覆う1つの抽象化](serial.md)を、N_TTYの内部を知りたいなら[ラインディシプリン、termios、擬似端末](line-disciplines.md)を読む。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/tty.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/tty.h) — `struct tty_struct`、`TTY_*`フラグビット、termiosのアクセサマクロ
- [include/linux/tty_driver.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/tty_driver.h) — `struct tty_driver`、`struct tty_operations`、`tty_alloc_driver()`/`tty_register_driver()`
- [include/linux/tty_ldisc.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/tty_ldisc.h) — `struct tty_ldisc_ops`、`struct tty_ldisc`、`tty_register_ldisc()`/`tty_set_ldisc()`
- [include/uapi/linux/tty.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/tty.h) — `N_*`ラインディシプリン番号（`N_TTY`、`N_PPP`、`N_HDLC`、...）
- [include/linux/console.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/console.h) — `struct console`、`CON_BOOT`/`CON_PRINTBUFFER`、コンソールをその`tty_driver`に結びつける`device`コールバック
- [drivers/tty/tty_io.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/tty/tty_io.c) — ttyコア: `tty_open()`、`tty_lookup_driver()`、`/dev/tty`と`/dev/console`の処理
- [drivers/tty/tty_ldisc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/tty/tty_ldisc.c) — ラインディシプリンの接続/切り離し/再初期化、N_TTYへのフォールバックのロジック
- [drivers/tty/n_tty.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/tty/n_tty.c) — デフォルトのN_TTYラインディシプリン: カノニカルモード、エコー、シグナル生成
- [drivers/tty/pty.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/tty/pty.c) — ptyのマスター/スレーブドライバ、`/dev/ptmx`、UNIX 98 ptyの割り当て

### manページ {#man-pages}

- [`tty(4)`](https://man7.org/linux/man-pages/man4/tty.4.html) — 制御端末の別名である`/dev/tty`
- [`ttyS(4)`](https://man7.org/linux/man-pages/man4/ttyS.4.html) — シリアル端末回線のデバイス
- [`pty(7)`](https://man7.org/linux/man-pages/man7/pty.7.html) — 擬似端末のマスター/スレーブのペア、UNIX 98 ptyとBSD pty、`/dev/ptmx`と`/dev/pts/*`
- [`termios(3)`](https://man7.org/linux/man-pages/man3/termios.3.html) — カノニカルモードと非カノニカルモード、`c_iflag`/`c_oflag`/`c_cflag`/`c_lflag`フィールド、`ISIG`の特殊文字

### 関連ページ {#related-pages}

- [シリアルコア: あらゆるUARTを覆う1つの抽象化](serial.md) · [ラインディシプリン、termios、擬似端末](line-disciplines.md) · [事件簿](war-stories.md) — このセクションの残りのページ
- [キャラクタデバイスとmiscデバイス](../drivers/chardev.md) · [Linuxデバイスモデル](../drivers/device-model.md) — TTYサブシステムが載っているドライバコアの層

### LWN記事 {#lwn-articles}

- [A tempest in a tty pot](https://lwn.net/Articles/343828/) — TTYコードの複雑さに関する評判と、一時的にメンテナ不在の状態をもたらした保守をめぐる論争について

### 外部資料 {#external}

- [カーネルドキュメント: TTY driver-apiの索引](https://docs.kernel.org/driver-api/tty/index.html) — `tty_driver`、`tty_port`、`tty_struct`、ラインディシプリン、コンソールに関するカーネル自身の概要
- [カーネルドキュメント: TTY Line Discipline](https://docs.kernel.org/driver-api/tty/tty_ldisc.html) — `tty_ldisc_ops`、デフォルトのN_TTY、ディシプリンが失敗したときのN_NULLフォールバック
- [カーネルドキュメント: TTY Driver and TTY Operations](https://docs.kernel.org/driver-api/tty/tty_driver.html) — `tty_driver`/`tty_operations`の完全なリファレンス
- [カーネルドキュメント: Console](https://docs.kernel.org/driver-api/tty/console.html) — `struct console`、ブートから実行時へのコンソールの引き継ぎ、コンソールが`tty_driver`にどう対応づけられるか
