# シリアルコア: あらゆるUARTを覆う1つの抽象化 {#serial-core-one-abstraction-over-every-uart}

> `drivers/tty/serial/serial_core.c`は、1980年代初頭の8250の流れを汲む16550のレジスタセットと、現代のSoCのメモリマップドUARTの両方を、同じく`/dev/ttyS0`として見せる層である

## 問題: UARTといってもさまざまである {#the-problem-a-uart-is-not-a-uart}

どのUARTチップも1度に1ビットずつ回線上でバイトを運ぶが、「あらゆるUART」は1つのチップではなく、無秩序に広がった多数のチップである。由緒ある8250/16550ファミリー（と、そのレジスタ互換の多くの子孫）は、PC形式のシリアルポートやPCIeのマルチポートカードの背後にあるものとして、いまも最も一般的である。しかし、ほぼすべてのSoCベンダーが独自のUART IPブロックも出荷している。Broadcomのもの、Freescale/NXPの`lpuart`、Qualcommの`msm_serial`/`geni`、Samsungのもの、ARMのPL011、Xilinxの`uartlite`、さらに数十種類があり、それぞれが独自のレジスタレイアウト、独自のFIFOの深さ、そしてモデム制御線がどう配線されているか（RS-232のシグナリングを一切持たないものではまったく配線されていないこともある）についての独自の癖を持つ。`drivers/tty/serial/`に現在、チップごとの個別ドライバのソースファイルがおよそ70個（共有のコアファイルや、8250ファミリーの変種と増設カード用に`drivers/tty/serial/8250/`以下にあるさらに約45個のファイルは数えずに）あるのは、まさにこの理由による。

そうした多様性のどれも、汎用のTTY層が気にすべき問題ではない。`drivers/tty/tty_io.c`とその仲間が知っているのは、ラインディシプリン、`termios`、ジョブ制御、キャラクタデバイスとしての`/dev/ttyS0`である。これらの概念は、下にあるバイトが8250から来たものでも、BluetoothのRFCOMMチャネルから来たものでも、仮想コンソールから来たものでも、まったく同じように適用される。TTY層が知らずに済むべきなのは、`IER`レジスタをどういじるか、あるいはチップ固有のFIFOトリガーレベルのエンコーディングをどうデコードするか、といったことである。

`serial_core.c`はその間にある層である。ハードウェア側の契約を1つ（すべてのUARTドライバが埋めるコールバックの表である`struct uart_ops`）と、登録モデルを1つ（チップファミリーのための`struct uart_driver`と、その物理インスタンス1つのための`struct uart_port`）定義する。70余りのドライバはどれも、同じ一握りのコールバックを実装する。`serial_core.c`は、それらのコールバックをTTYコアが駆動する汎用の`tty_operations`テーブルに変換し、TTYコアからの呼び出し（open、write、set_termios、ioctl）を正しいポートの正しい`uart_ops`コールバックに変換するものである。`.startup`、`.shutdown`、`.set_termios`、`.start_tx`と、他のわずかなものを実装したドライバ作者は、`/dev/ttyS*`をただで手に入れる。ラインディシプリンの処理、`termios`のネゴシエーション、`poll()`、`TIOCM*`のモデム制御ioctl、magic SysRq、カーネルコンソールのサポート、そのすべてを、TTY層のコードを1行も書かずに得られるのである。

## コアの構造体 {#core-structures}

### `struct uart_ops`: ハードウェアとの契約 {#struct-uart_ops-the-hardware-contract}

`struct uart_ops`（`include/linux/serial_core.h`）は、すべてのUARTドライバが構築し、`uart_port.ops`経由で`serial_core`に渡すvtableである。これが、ドライバが汎用層に組み込まれるために実装しなければならないインターフェースのすべてである。

```c
// include/linux/serial_core.h
struct uart_ops {
	unsigned int	(*tx_empty)(struct uart_port *);
	void		(*set_mctrl)(struct uart_port *, unsigned int mctrl);
	unsigned int	(*get_mctrl)(struct uart_port *);
	void		(*stop_tx)(struct uart_port *);
	void		(*start_tx)(struct uart_port *);
	void		(*throttle)(struct uart_port *);
	void		(*unthrottle)(struct uart_port *);
	void		(*send_xchar)(struct uart_port *, char ch);
	void		(*stop_rx)(struct uart_port *);
	void		(*start_rx)(struct uart_port *);
	void		(*enable_ms)(struct uart_port *);
	void		(*break_ctl)(struct uart_port *, int ctl);
	int		(*startup)(struct uart_port *);
	void		(*shutdown)(struct uart_port *);
	void		(*flush_buffer)(struct uart_port *);
	void		(*set_termios)(struct uart_port *, struct ktermios *new,
				       const struct ktermios *old);
	void		(*set_ldisc)(struct uart_port *, struct ktermios *);
	void		(*pm)(struct uart_port *, unsigned int state,
			      unsigned int oldstate);
	const char	*(*type)(struct uart_port *);
	void		(*release_port)(struct uart_port *);
	int		(*request_port)(struct uart_port *);
	void		(*config_port)(struct uart_port *, int);
	int		(*verify_port)(struct uart_port *, struct serial_struct *);
	int		(*ioctl)(struct uart_port *, unsigned int, unsigned long);
#ifdef CONFIG_CONSOLE_POLL
	int		(*poll_init)(struct uart_port *);
	void		(*poll_put_char)(struct uart_port *, unsigned char);
	int		(*poll_get_char)(struct uart_port *);
#endif
};
```

日常的な負荷のほぼすべてを担うのは、このうちのいくつかである。

- **`.startup`/`.shutdown`** — ポートの最初の`open()`と最後の`close()`が行われるときに呼ばれる。IRQを要求し、ハードウェアFIFOをリセットして有効化し、割り込み許可レジスタを設定する。そしてそのすべてを元に戻す。
- **`.set_termios`** — ユーザー空間がボーレート、パリティ、ストップビット、フロー制御を変更するたびに（`stty`、または`tcsetattr()`に渡された`termios`構造体によって）呼ばれる。ドライバがボーレートの分周値を計算して設定するのはここである。
- **`.start_tx`/`.stop_tx`** — 送信側を開始または停止する。割り込み駆動のドライバでは、`.start_tx`は通常「THR空」割り込みのマスクを外し、割り込みハンドラにFIFOへの供給を任せるだけである。`.stop_tx`はそれを再びマスクする。
- **`.stop_rx`** — 受信データを上位に渡すのを止める（`!CREAD`とスロットリングに使われる）。
- **`.set_mctrl`/`.get_mctrl`** — モデム制御線（RTS、DTR）を設定し、ステータス線（CTS、DSR、DCD、RI）を読み戻す。それぞれ`TIOCM_*`のビットとして扱う。
- **`.tx_empty`** — 送信パス（FIFOとシフトレジスタ）が完全に空になったかどうかを報告する。`TIOCSER_TEMT`/`tcdrain()`の裏付けとなる。
- **`.type`/`.request_port`/`.release_port`/`.config_port`/`.verify_port`** — `setserial`時代のポート設定と自動検出の仕組み。人間が読めるチップ名を報告し、I/OまたはMMIOの領域を確保/解放し、正確なチップの変種を自動検出し、ユーザーが要求した`struct serial_struct`の変更を検証する。

### `struct uart_port`: 1つの物理ポート {#struct-uart_port-one-physical-port}

`struct uart_port`はインスタンスごとのオブジェクトであり、物理UARTごとに1つ、ドライバによって割り当てられて埋められる。ttyS/ttyUL/ttyAMA/...のマイナー番号ごとに1つである。レジスタアクセスの方式、ネゴシエートされたハードウェアの状態、そして汎用層への逆ポインタを保持する。

```c
// include/linux/serial_core.h
struct uart_port {
	spinlock_t		lock;			/* ポートのロック */
	unsigned long		iobase;			/* in/out[bwl] */
	unsigned char __iomem	*membase;		/* read/write[bwl] */
	u32			(*serial_in)(struct uart_port *, unsigned int offset);
	void			(*serial_out)(struct uart_port *, unsigned int offset, u32 val);
	void			(*set_termios)(struct uart_port *,
				               struct ktermios *new,
				               const struct ktermios *old);
	void			(*set_ldisc)(struct uart_port *,
					     struct ktermios *);
	unsigned int		(*get_mctrl)(struct uart_port *);
	void			(*set_mctrl)(struct uart_port *, unsigned int);
	unsigned int		(*get_divisor)(struct uart_port *,
					       unsigned int baud,
					       unsigned int *frac);
	void			(*set_divisor)(struct uart_port *,
					       unsigned int baud,
					       unsigned int quot,
					       unsigned int quot_frac);
	int			(*startup)(struct uart_port *port);
	void			(*shutdown)(struct uart_port *port);
	void			(*throttle)(struct uart_port *port);
	void			(*unthrottle)(struct uart_port *port);
	int			(*handle_irq)(struct uart_port *);
	void			(*pm)(struct uart_port *, unsigned int state,
				      unsigned int old);
	void			(*handle_break)(struct uart_port *);
	int			(*rs485_config)(struct uart_port *,
						struct ktermios *termios,
						struct serial_rs485 *rs485);
	int			(*iso7816_config)(struct uart_port *,
						  struct serial_iso7816 *iso7816);
	unsigned int		ctrl_id;		/* 任意: シリアルコアのコントローラID */
	unsigned int		port_id;		/* 任意: シリアルコアのポートID */
	unsigned int		irq;			/* IRQ番号 */
	unsigned long		irqflags;		/* IRQフラグ */
	unsigned int		uartclk;		/* UARTのベースクロック */
	unsigned int		fifosize;		/* 送信FIFOのサイズ */
	unsigned char		x_char;			/* XON/XOFF文字 */
	unsigned char		regshift;		/* レジスタオフセットのシフト量 */
	unsigned char		quirks;			/* 内部的な癖 */
	enum uart_iotype	iotype;			/* I/Oアクセス方式 */
	unsigned int		read_status_mask;	/* ドライバ固有 */
	unsigned int		ignore_status_mask;	/* ドライバ固有 */
	struct uart_state	*state;			/* 親のstateへのポインタ */
	struct uart_icount	icount;			/* 統計 */
	struct console		*cons;			/* struct console（あれば） */
	upf_t			flags;
	upstat_t		status;
	bool			hw_stopped;		/* ソフトウェア支援のCTSフロー状態 */
	bool			cons_flow;		/* ユーザー指定のコンソールフロー制御 */
	unsigned int		mctrl;			/* 現在のモデム制御設定 */
	unsigned int		frame_time;		/* フレーム時間（ns） */
	unsigned int		type;			/* ポート種別 */
	const struct uart_ops	*ops;
	unsigned int		custom_divisor;
	unsigned int		line;			/* ポートのインデックス */
	unsigned int		minor;
	resource_size_t		mapbase;		/* ioremap用 */
	resource_size_t		mapsize;
	struct device		*dev;			/* シリアルポートの物理的な親デバイス */
	struct serial_port_device *port_dev;		/* シリアルコアのポートデバイス */
	unsigned long		sysrq;			/* sysrqのタイムアウト */
	u8			sysrq_ch;		/* sysrq用の文字 */
	unsigned char		has_sysrq;
	unsigned char		sysrq_seq;		/* sysrq_toggle_seq内のインデックス */
	unsigned char		hub6;			/* これは8250ドライバにあるべき */
	unsigned char		suspended;
	unsigned char		console_reinit;
	const char		*name;			/* ポート名 */
	struct attribute_group	*attr_group;		/* ポート固有の属性 */
	const struct attribute_group **tty_groups;	/* 全属性（シリアルコア専用） */
	struct serial_rs485     rs485;
	struct serial_rs485	rs485_supported;	/* serial_rs485のサポートマスク */
	struct gpio_desc	*rs485_term_gpio;	/* RS485バス終端を有効にする */
	struct gpio_desc	*rs485_rx_during_tx_gpio; /* TX中のRS485 RXの状態を設定する出力GPIO */
	struct serial_iso7816   iso7816;
	void			*private_data;		/* 汎用のプラットフォームデータポインタ */
};
```

特筆に値するフィールドをいくつか挙げる。

- **`serial_in`/`serial_out`** — レジスタアクセスの間接化。すべてのレジスタアクセスを`#ifdef`で分けるのではなく、これらの関数ポインタを差し替えることで、1つのドライバが複数のバス幅/エンディアン（8/16/32ビット、ビッグ/リトルエンディアンのMMIO、あるいはレガシーなポートI/O）をサポートできるようにする。
- **`iotype`**（`enum uart_iotype`: `UPIO_PORT`、`UPIO_MEM`、`UPIO_MEM32`、`UPIO_MEM32BE`、`UPIO_MEM16`と、いくつかのチップ固有の変種 — `UPIO_HUB6`、`UPIO_AU`、`UPIO_TSI`） — `serial_in`/`serial_out`がどのアクセス方式を実装しているか。
- **`line`** — このポートのインデックス。`uart_port`を、その`uart_driver`のstate配列（後述）の特定のスロットに結びつけるのがこれである。
- **`state`** — `struct uart_state`への逆ポインタ。このハードウェアポートを動作中の`tty_port`に結びつけるオブジェクトである。
- **`ops`** — 上で説明した`uart_ops`のvtable。
- **`private_data`** — コールバックが必要とするインスタンスごとの状態（クロックのハンドル、DMAチャネル、デバイスツリーから解析したチップ固有の設定）のための、ドライバが所有するポインタ。

### `struct uart_driver`: 登録レベルのオブジェクト {#struct-uart_driver-the-registration-level-object}

`uart_port`が1つの物理ポートであるのに対し、`struct uart_driver`はチップファミリー全体の登録である。インスタンスごとに1つではなく、ドライバモジュールごとに1つである。

```c
// include/linux/serial_core.h
struct uart_driver {
	struct module		*owner;
	const char		*driver_name;
	const char		*dev_name;
	int			 major;
	int			 minor;
	int			 nr;
	struct console		*cons;

	/*
	 * これらはプライベートである。低レベルドライバはこれらに
	 * 触れてはならない。NULLに初期化しておくこと
	 */
	struct uart_state	*state;
	struct tty_driver	*tty_driver;
};
```

`driver_name`/`dev_name`は、`setserial`から見える名前と、`/dev`の名前の語幹（`"ttyS"`、`"ttyUL"`、...）である。`nr`はこのドライバファミリーが登録しうるポートの最大数であり、`uart_register_driver()`が割り当てる`state`配列の大きさを決める。`cons`がセットされていれば、このドライバをカーネルコンソールとして接続する（カーネルコマンドラインの`console=ttyS0`）。`state`と`tty_driver`はまさにコメントのとおりで、`serial_core`が埋め、ドライバは触れない。

## TTY層への橋渡し {#bridging-to-the-tty-layer}

### `struct uart_state`: オープンをまたいで存続するオブジェクト {#struct-uart_state-the-persistent-across-opens-object}

`struct uart_state`は、個々の`open()`/`close()`のサイクルをまたいで存続する小さなオブジェクトであり、`uart_driver.state[]`はこれを、ありうるポートごとに1つずつ並べた配列である。

```c
// include/linux/serial_core.h
struct uart_state {
	struct tty_port		port;

	enum uart_pm_state	pm_state;

	atomic_t		refcount;
	wait_queue_head_t	remove_wait;
	struct uart_port	*uart_port;
};
```

`port`は汎用の`struct tty_port`であり、open/close/hangupのセマンティクスを持つすべてのキャラクタデバイスが埋め込む、TTYコア側のオブジェクトである。`uart_port`は、現在このstateスロットに接続されている動作中のハードウェアポートである（ドライバが`uart_register_driver()`でstate配列を登録したものの、特定のインスタンスがまだ現れていない場合は`NULL`になりうる。UARTがホットプラグ可能な増設カードであるプラットフォームではよくあることである）。

### 登録: `uart_register_driver()`と`uart_add_one_port()` {#registration-uart_register_driver-and-uart_add_one_port}

2つの呼び出しがすべてを接続し、すべてのUARTドライバのモジュール初期化は両方を行う。

`uart_register_driver()`（`drivers/tty/serial/serial_core.c`）は`drv->state`（`drv->nr`個のエントリ）を割り当て、汎用の`struct tty_driver`を割り当て、そして（これが鍵となる行である）その汎用`tty_driver`を、単一の共有された`tty_operations`テーブルに向ける。

```c
// drivers/tty/serial/serial_core.c — uart_register_driver()、抜粋
drv->state = kzalloc_objs(struct uart_state, drv->nr);
...
normal = tty_alloc_driver(drv->nr, TTY_DRIVER_REAL_RAW | TTY_DRIVER_DYNAMIC_DEV);
...
drv->tty_driver = normal;
normal->driver_name	= drv->driver_name;
normal->name		= drv->dev_name;
...
tty_set_operations(normal, &uart_ops);

for (i = 0; i < drv->nr; i++) {
	struct uart_state *state = drv->state + i;
	struct tty_port *port = &state->port;

	tty_port_init(port);
	port->ops = &uart_port_ops;
}
```

この`uart_ops`は、上のハードウェアvtableである`struct uart_ops`と同じ型では *ない*。これは`serial_core.c`内のファイルローカルな`static const struct tty_operations`であり、チップにかかわらず、すべてのUARTドライバファミリーが変更なしで共有する。

```c
// drivers/tty/serial/serial_core.c
static const struct tty_operations uart_ops = {
	.install	= uart_install,
	.open		= uart_open,
	.close		= uart_close,
	.write		= uart_write,
	.put_char	= uart_put_char,
	.flush_chars	= uart_flush_chars,
	.write_room	= uart_write_room,
	.chars_in_buffer= uart_chars_in_buffer,
	.flush_buffer	= uart_flush_buffer,
	.ioctl		= uart_ioctl,
	.throttle	= uart_throttle,
	.unthrottle	= uart_unthrottle,
	.send_xchar	= uart_send_xchar,
	.set_termios	= uart_set_termios,
	.set_ldisc	= uart_set_ldisc,
	.stop		= uart_stop,
	.start		= uart_start,
	.hangup		= uart_hangup,
	.break_ctl	= uart_break_ctl,
	.wait_until_sent= uart_wait_until_sent,
	...
	.tiocmget	= uart_tiocmget,
	.tiocmset	= uart_tiocmset,
	.set_serial	= uart_set_info_user,
	.get_serial	= uart_get_info_user,
	.get_icount	= uart_get_icount,
	...
};
```

2つの構造体、同じ識別子`uart_ops`、2つの異なる型である。一方はドライバごとのハードウェアコールバックテーブル（`serial_core.h`の`struct uart_ops`で、個々のUARTドライバが選ぶもの）であり、もう一方は、シリアルコアベースのすべての`tty_driver`が共有する単一の汎用`struct tty_operations`テーブル（`serial_core.c`の中で一度だけ、プライベートに定義される）である。小さいが実在する名前の衝突であり、ソースを読むときには意識しておく価値がある。

`uart_add_one_port()`は2つ目の呼び出しであり、ドライバが見つけた物理ポートごとに1回行われる（DTで列挙されるプラットフォームドライバなら`.probe()`から、マルチポートカードのドライバならカード上のポートごとに1回）。

```c
// include/linux/serial_core.h
int uart_add_one_port(struct uart_driver *reg, struct uart_port *port);
void uart_remove_one_port(struct uart_driver *reg, struct uart_port *port);
```

これは、ドライバが埋めたばかりの`uart_port`を`drv->state[port->line]`に関連づける（`serial_core.c`全体で使われるインデックスの慣例は、文字どおり`state = drv->state + uport->line`である）。そして、`tty_port_register_device()`相当の登録が必要なら、`/dev`ノードを立ち上げる。（最近のカーネルは内部の配管を、`serial_ctrl.c`/`serial_port.c`にある小さなシリアルコアの「コントローラ」/「ポート」サブデバイスモデルを通すようにしており、電源管理やデバイスモデルとの統合のために`uart_add_one_port()`の下に層を成している。ドライバ作者が知る必要のあるエントリポイントと`drv->state[port->line]`の関連づけは変わっていない。）

### TTYコアからの呼び出しがハードウェアに届くまで {#how-a-tty-core-call-reaches-hardware}

登録が済めば、共有された`uart_ops`テーブル内のすべての`tty_operations`コールバックは汎用のシリアルコアのロジックであり、実際にハードウェアに触れる必要がある時点で`uport->ops`を通じて呼び出しを行う。`serial_core.c`から直接取った、具体的な呼び出しの連鎖をいくつか示す。

- ポートを開くと、最終的に`uart_port_startup()`が呼ばれ、それが`uport->ops->startup(uport)`、すなわちドライバの`.startup`を呼ぶ。
- `tcsetattr()`/`stty`は共有の`.set_termios = uart_set_termios`に到達する。これはttyコアが`tty->termios_rwsem`を保持した状態で呼ばれる。`tty_ioctl.c`の`tty_set_termios()`が、どのドライバの`set_termios`フックを呼び出すよりも前に`down_write()`でこれを取るので、これはttyのロックであって、`uart_set_termios()`自身が獲得するものではない（`uart_set_termios()`は内部で独自に`state->port.mutex`を取る）。`uart_set_termios()`は次に`uart_change_line_settings()`を呼び、それが`uport->ops->set_termios(uport, termios, old_termios)`、すなわちドライバの`.set_termios`を呼ぶ。
- データの書き込み（`uart_write()`）は`__uart_start()`を直接呼び、それが`port->ops->start_tx(port)`、すなわちドライバの`.start_tx`を呼ぶ。割り込み駆動のドライバでは、通常ここで送信空割り込みのマスクを外し、残りをISRに任せるだけである（後述）。（`uart_start()`は別のエントリポイントで、`.start`のttyフックであり、`uart_flush_chars()`経由でも到達する。これも同じ`__uart_start()`を呼ぶだけである。）
- RTS/DTRの設定（`TIOCMSET`、またはopen/close時の`CLOCAL`/`HUPCL`の処理）は`uart_update_mctrl()`を呼び、それが`port->ops->set_mctrl(port, port->mctrl)`を呼ぶ。

このロジックのどれもチップ固有ではなく、8250でも`uartlite`でも同一である。この4つが最もよくあるホットパスの呼び出しだが、これらだけでは到底ない。実際には、`serial_core.c`はそのロジックのどこかで、およそ20数種類の異なる`uart_ops`メンバを呼び出している。`release_port`、`get_mctrl`、`tx_empty`、`stop_rx`、`start_rx`、`request_port`、`verify_port`、`type`、`shutdown`、`set_ldisc`、`send_xchar`、`pm`、`ioctl`、`flush_buffer`、`enable_ms`、`config_port`、`break_ctl`、`throttle`、`unthrottle`、そしてpoll系のフックなどであり、それぞれがそれを必要とする特定のTTYコアまたは`ioctl`のパスから呼び出される。

## 8250/16550ドライバ: 基準となるUART {#the-825016550-driver-the-reference-uart}

`drivers/tty/serial/8250/`は8250/16550互換ファミリーのドライバであり、他のあらゆるシリアルドライバがこれを基準に比較される。理由は歴史的で単純である。16550（とそのUART互換の後継）は、1981年のオリジナルのPCシリアルポートが使ったチップである8250のレジスタ互換の子孫であり、したがってこの系譜はツリー内で最も古く、最もよく使われ、最も徹底的に鍛えられたUARTドライバなのである。新しいチップ固有のドライバは日常的にその一部を再利用する。`drivers/tty/serial/8250/`以下の`8250_omap.c`、`8250_dw.c`、`8250_pci1xxxx.c`などのファイルはどれも、共有の8250コア（`8250_port.c`）を包む薄いラッパーであり、FIFO/レジスタのロジックを一から再実装するのではなく、その上にチップ固有のDMA、クロック、エラッタの処理を追加している。

### レジスタモデルの概要 {#the-register-model-briefly}

16550互換のUARTは、バイト幅のレジスタの小さなバンクを公開している。そのいくつかは *エイリアス* されており、同じオフセットが方向や制御ビットの状態によって異なる意味を持つ。これこそが、`struct uart_ops`がその上のすべてから隠すために存在する、厄介でチップ固有の詳細である。

| オフセット | 読み取り | 書き込み | 目的 |
|---|---|---|---|
| 0 | RBR（受信バッファ） | THR（送信保持） | FIFOとの間で1バイトを出し入れする |
| 1 | IER（割り込み許可） | IER | どの割り込み要因のマスクを外すか |
| 2 | IIR（割り込みID） | FCR（FIFO制御） | 割り込みが発生した理由 / FIFOのリセットとトリガーレベル |
| 3 | LCR（ライン制御） | LCR | データビット、ストップビット、パリティ、DLABラッチビット |
| 4 | MCR（モデム制御） | MCR | RTS/DTRを駆動する（およびループバックモード） |
| 5 | LSR（ラインステータス） | — | データ準備完了、オーバーラン/パリティ/フレーミングエラー、THR空、breakのフラグ |
| 6 | MSR（モデムステータス） | — | CTS/DSR/DCD/RIの現在の状態と変化ビット |

`LCR`のDLABビットがセットされている間、オフセット0と1はまったく別の意味（ボーレート分周ラッチの下位バイトと上位バイト、DLL/DLM）になる。そのため`.set_termios`は、一時的にDLABを切り替え、分周値を設定し、それからLCRを通常の意味に戻さなければならない。`struct uart_port.serial_in`/`serial_out`と`include/uapi/linux/serial_reg.h`のオフセット定数は、この種のビット操作を1つのドライバの中に閉じ込めておくために存在する。`uart_ops`より上のものは、`IIR`の読み取りと`FCR`の書き込みが同じアドレスを共有していることを知る必要がまったくない。

### `struct uart_8250_port`: 8250の`uart_port`サブクラス {#struct-uart_8250_port-8250s-uart_port-subclass}

8250ドライバのポートごとのオブジェクトは、素の`uart_port`を最初のメンバとして埋め込み、その周りに8250固有の状態を追加している。Linuxの古典的な「埋め込みによるサブクラス化」パターンである。

```c
// include/linux/serial_8250.h
struct uart_8250_port {
	struct uart_port	port;
	struct timer_list	timer;		/* 「IRQなし」用タイマー */
	struct list_head	list;		/* このIRQ上のポート */
	u32			capabilities;	/* ポートの機能 */
	u16			bugs;		/* ポートのバグ */
	unsigned int		tx_loadsz;	/* 送信FIFOのロードサイズ */
	unsigned char		acr;
	unsigned char		fcr;
	unsigned char		ier;
	unsigned char		lcr;
	unsigned char		mcr;
	unsigned char		cur_iotype;	/* 実行中のI/O種別 */
	unsigned int		rpm_tx_active;
	unsigned char		canary;		/* no_console_suspendの場合、
						 *   システムスリープ中は非ゼロ
						 */
	unsigned char		probe;
	struct mctrl_gpios	*gpios;
#define UART_PROBE_RSA	(1 << 0)

	u16			lsr_saved_flags;
	u16			lsr_save_mask;
	unsigned char		msr_saved_flags;

	struct uart_8250_dma	*dma;
	const struct uart_8250_ops *ops;

	/* 8250固有のコールバック */
	u32			(*dl_read)(struct uart_8250_port *up);
	void			(*dl_write)(struct uart_8250_port *up, u32 value);

	struct uart_8250_em485 *em485;
	void			(*rs485_start_tx)(struct uart_8250_port *up, bool toggle_ier);
	void			(*rs485_stop_tx)(struct uart_8250_port *up, bool toggle_ier);

	/* シリアルポートのオーバーラン時のバックオフ */
	struct delayed_work overrun_backoff;
	u32 overrun_backoff_time_ms;
};
```

`ier`/`fcr`/`lcr`/`mcr`はハードウェアレジスタのシャドーコピーである。4つのうち本当に書き込み専用なのは`FCR`だけで（同じオフセットを読むと`IIR`が返る）、`IER`/`LCR`/`MCR`は読み取り可能である。それでもドライバが4つすべてをシャドーしているのは、読み戻せないからではなく、read-modify-writeの競合を避け、読み取りに副作用のあるレジスタを回避するためである。これを`serial_core`に組み込むのは`serial8250_init_port()`（`drivers/tty/serial/8250/8250_port.c`）であり、`port->ops = &serial8250_pops`、すなわちドライバの`struct uart_ops`実装を設定する。

```c
// drivers/tty/serial/8250/8250_port.c
static const struct uart_ops serial8250_pops = {
	.tx_empty	= serial8250_tx_empty,
	.set_mctrl	= serial8250_set_mctrl,
	.get_mctrl	= serial8250_get_mctrl,
	.stop_tx	= serial8250_stop_tx,
	.start_tx	= serial8250_start_tx,
	.throttle	= serial8250_throttle,
	.unthrottle	= serial8250_unthrottle,
	.stop_rx	= serial8250_stop_rx,
	.enable_ms	= serial8250_enable_ms,
	.break_ctl	= serial8250_break_ctl,
	.startup	= serial8250_startup,
	.shutdown	= serial8250_shutdown,
	.flush_buffer	= serial8250_flush_buffer,
	.set_termios	= serial8250_set_termios,
	.set_ldisc	= serial8250_set_ldisc,
	.pm		= serial8250_pm,
	.type		= serial8250_type,
	.release_port	= serial8250_release_port,
	.request_port	= serial8250_request_port,
	.config_port	= serial8250_config_port,
	.verify_port	= serial8250_verify_port,
#ifdef CONFIG_CONSOLE_POLL
	.poll_get_char = serial8250_get_poll_char,
	.poll_put_char = serial8250_put_poll_char,
#endif
};
```

## 割り込み駆動のTX/RXパス {#the-interrupt-driven-txrx-path}

ここで抽象化が端から端まで報われる。割り込みが発生すると、固定された汎用の手順によって、生のレジスタの読み取りがラインディシプリン（そして最終的にはユーザー空間）が読めるバイトに変わり、あるいはバッファされたwrite()のデータがレジスタへの書き込みに変わる。

### RX: ハードウェア → `uart_insert_char()` → `tty_flip_buffer_push()` {#rx-hardware-uart_insert_char-tty_flip_buffer_push}

`serial8250_handle_irq_locked()`（`drivers/tty/serial/8250/8250_port.c`）は8250のISRの連鎖の心臓部である（登録されたIRQハンドラ`serial8250_interrupt()`から`port->handle_irq()`経由で到達する）。受信側では`LSR`を読み、データが準備できていれば`serial8250_rx_chars()`を呼ぶ。

```c
// drivers/tty/serial/8250/8250_port.c — serial8250_handle_irq_locked()、抜粋
status = serial_lsr_in(up);
...
if (status & (UART_LSR_DR | UART_LSR_BI) && !skip_rx) {
	...
	if (!up->dma || handle_rx_dma(up, iir))
		status = serial8250_rx_chars(up, status);
}
```

`serial8250_rx_chars()`は`serial8250_read_char()`でFIFOを1バイトずつ吸い出し、それからバッチ全体を1回の呼び出しでラインディシプリンに押し出す。

```c
// drivers/tty/serial/8250/8250_port.c
u16 serial8250_rx_chars(struct uart_8250_port *up, u16 lsr)
{
	struct uart_port *port = &up->port;
	int max_count = 256;

	do {
		serial8250_read_char(up, lsr);
		if (--max_count == 0)
			break;
		lsr = serial_in(up, UART_LSR);
	} while (lsr & (UART_LSR_DR | UART_LSR_BI));

	tty_flip_buffer_push(&port->state->port);
	return lsr;
}
```

そして`serial8250_read_char()`は、生のレジスタのバイトが、適切にフラグ付けされた（`TTY_NORMAL`/`TTY_BREAK`/`TTY_PARITY`/`TTY_FRAME`）文字となって汎用層に渡される場所である。

```c
// drivers/tty/serial/8250/8250_port.c — serial8250_read_char()、抜粋
if (likely(lsr & UART_LSR_DR))
	ch = serial_in(up, UART_RX);
else
	ch = 0;

port->icount.rx++;
...
uart_insert_char(port, lsr, UART_LSR_OE, ch, flag);
```

`uart_insert_char()`（`drivers/tty/serial/serial_core.c`）は`serial_core`の汎用RXエントリポイントであり、より特化したものを必要としないすべてのドライバが、受信したバイトごとに1回これを呼ぶ。

```c
// drivers/tty/serial/serial_core.c
void uart_insert_char(struct uart_port *port, unsigned int status,
		      unsigned int overrun, u8 ch, u8 flag)
{
	struct tty_port *tport = &port->state->port;

	if ((status & port->ignore_status_mask & ~overrun) == 0)
		if (tty_insert_flip_char(tport, ch, flag) == 0)
			++port->icount.buf_overrun;

	/*
	 * オーバーランは特別である。即座に報告されるので、
	 * 現在の文字には影響しない。
	 */
	if (status & ~port->ignore_status_mask & overrun)
		if (tty_insert_flip_char(tport, 0, TTY_OVERRUN) == 0)
			++port->icount.buf_overrun;
}
```

これは、バイトを保持する価値があるかを判断する前にポートの`ignore_status_mask`（`.set_termios`が`IGNPAR`/`IGNBRK`のような`termios`フラグから設定する）をチェックし、それから`tty_insert_flip_char()`を呼ぶ。これは汎用TTYのフリップバッファへの挿入であり、バイトとそのフラグをキューに入れるだけで、まだ誰も起床させない。ドライバ自身のループは、ハードウェアFIFO内のバイトごとに`uart_insert_char()`を呼び続ける。バッチを吸い出し終えて初めて、`tty_flip_buffer_push(&port->state->port)`（`include/linux/tty_flip.h`）を呼んでバッチ全体を一度にラインディシプリンに渡し、読み手を起床させる。このように起床をまとめること（バイトごとに1回ではなく、割り込みごとに1回の`tty_flip_buffer_push()`）は、偶然ではなく、スループットのための意図的な選択である。

### TX: バッファされたデータ → `uart_fifo_get()` → ハードウェア {#tx-buffered-data-uart_fifo_get-hardware}

同じハンドラの送信側の半分は、`LSR`のTHR空ビットがセットされ、送信空割り込みが有効になっているときに実行される。

```c
// drivers/tty/serial/8250/8250_port.c — serial8250_handle_irq_locked()、抜粋
if ((status & UART_LSR_THRE) && (up->ier & UART_IER_THRI)) {
	if (!up->dma || up->dma->tx_err)
		serial8250_tx_chars(up);
	else if (!up->dma->tx_running)
		__stop_tx(up);
}
```

`serial8250_tx_chars()`は、ポートの送信用`kfifo`（`write()`/`tty_write()`が埋めたバッファ）から`uart_fifo_get()`で1バイトずつ取り出し、それを`THR`レジスタに直接書き込む。FIFOのロードサイズを上限として、バッファが空になるかロードサイズに達するまで続ける。

```c
// drivers/tty/serial/8250/8250_port.c — serial8250_tx_chars()、抜粋
if (kfifo_is_empty(&tport->xmit_fifo)) {
	__stop_tx(up);
	return;
}

count = up->tx_loadsz;
do {
	unsigned char c;

	if (!uart_fifo_get(port, &c))
		break;

	serial_out(up, UART_TX, c);
	...
} while (--count > 0);

if (kfifo_len(&tport->xmit_fifo) < WAKEUP_CHARS)
	uart_write_wakeup(port);
```

`uart_fifo_get()`（`include/linux/serial_core.h`）は、TX側における`uart_insert_char()`の汎用の対応物である。`port->state->port.xmit_fifo`から1文字を取り出し、`port->icount.tx`を増やす。

```c
// include/linux/serial_core.h
static inline unsigned int uart_fifo_get(struct uart_port *up,
		unsigned char *ch)
{
	struct tty_port *tport = &up->state->port;
	unsigned int chars;

	chars = kfifo_get(&tport->xmit_fifo, ch);
	up->icount.tx += chars;

	return chars;
}
```

FIFOが空になると、`.stop_tx`が`UART_IER_THRI`を再びマスクし、次の`write()`が`__uart_start()` → `.start_tx`を呼んでマスクを外すまでポートはアイドルになる。この開始/供給/停止のサイクル（TTYコアを1バイトずつ往復するのではなく）こそが、深さ16バイトの16550のFIFOが、バイトごとの割り込みなしに高いボーレートを維持できる理由である。

## 列挙: UARTはどのようにプローブされるか {#enumeration-how-a-uart-gets-probed}

UARTは、デバイスモデルから見れば、ただのプラットフォーム（あるいはPCI、USB、I2C/SPI接続の）デバイスの1つであり、他のプラットフォームドライバと同じ方法でドライバとマッチングされる。デバイスツリーに記述されたメモリマップドなSoCのUARTの場合、それは`of_device_id`のマッチテーブルを持つ`struct platform_driver`である。

```c
// drivers/tty/serial/uartlite.c、抜粋 — 実際のファイルではulite_of_match[]と
// ulite_platform_driverは約130行離れており、
// マッチテーブルは#if defined(CONFIG_OF) / #endifで囲まれている
#if defined(CONFIG_OF)
static const struct of_device_id ulite_of_match[] = {
	{ .compatible = "xlnx,opb-uartlite-1.00.b", },
	{ .compatible = "xlnx,xps-uartlite-1.00.a", },
	{}
};
MODULE_DEVICE_TABLE(of, ulite_of_match);
#endif /* CONFIG_OF */

...

static struct platform_driver ulite_platform_driver = {
	.probe = ulite_probe,
	.remove = ulite_remove,
	.driver = {
		.name  = "uartlite",
		.of_match_table = of_match_ptr(ulite_of_match),
		.pm = &ulite_pm_ops,
	},
};
```

カーネルがデバイスツリーをたどり、`compatible`プロパティがこれらの文字列のいずれかに一致するノードを見つけると、ドライバコアは、そのノードのリソース（MMIOの範囲、IRQ番号、クロックの参照、そして`uartlite`の場合はボーレート、パリティ、データビットといったベンダー固有のプロパティ）を持つ`struct platform_device`を引数に`.probe()`を呼ぶ。ACPIで記述されたUARTも同じ形に従い、`of_device_id`の代わりに（あるいはそれと並んで）`acpi_device_id`のテーブルを使う。DesignWareの8250ラッパー（`drivers/tty/serial/8250/8250_dw.c`）は、`dw8250_of_match[]`と`dw8250_acpi_match[]`の両方を同じ`platform_driver`に登録しているので、ファームウェアがポートをデバイスツリーとACPIのどちらで記述していても、同一のドライバがバインドされる。また、別のより単純な8250プラットフォームドライバ（`drivers/tty/serial/8250/8250_platform.c`）と、デバイスツリー専用のもの（`8250_of.c`）もある。複数の別個のプラットフォームドライバが、その下ではすべて同じ`serial8250_pops`/`uart_add_one_port()`のコアに流れ込んでいるのである。`8250_platform.c`はレガシーなISA形式のポートを、`acpi_platform_serial_table[]`ではなく名前でマッチする素の`"serial8250"`プラットフォームデバイスとして登録する。そのACPIテーブルには実質的なエントリが1つしかなく、RISC-Vの汎用16550Aのマッチ（`"RSCV0003"`）である。ファームウェアのインターフェースや、これらのフロントエンドドライバのどれが関わるかにかかわらず、`.probe()`の仕事は常に同じである。リソースを読み、`uart_port`を埋め、`uart_add_one_port()`を呼ぶ。

## 実例: `drivers/tty/serial/uartlite.c` {#worked-example-driversttyserialuartlitec}

Xilinxの`uartlite`ドライバが良い実例なのは、まさにそれが8250互換 *ではない* からである。FIFOのトリガーレベルもなく、エイリアスされたレジスタもなく、分周ラッチもない（このソフトIPコアではボーレートは論理合成時に固定される）。あるのは4つのレジスタ（RX、TX、ステータスレジスタ、制御レジスタ）と最小限の`uart_ops`実装だけであり、そのおかげで8250のレジスタの癖に邪魔されずに`serial_core`の配管が見やすくなる。

### `.startup`/`.shutdown` {#startupshutdown}

```c
// drivers/tty/serial/uartlite.c
static int ulite_startup(struct uart_port *port)
{
	struct uartlite_data *pdata = port->private_data;
	int ret;

	ret = clk_enable(pdata->clk);
	if (ret) {
		dev_err(port->dev, "Failed to enable clock\n");
		return ret;
	}

	ret = request_irq(port->irq, ulite_isr, IRQF_SHARED | IRQF_TRIGGER_RISING,
			  "uartlite", port);
	if (ret)
		return ret;

	uart_out32(ULITE_CONTROL_RST_RX | ULITE_CONTROL_RST_TX,
		ULITE_CONTROL, port);
	uart_out32(ULITE_CONTROL_IE, ULITE_CONTROL, port);

	return 0;
}

static void ulite_shutdown(struct uart_port *port)
{
	struct uartlite_data *pdata = port->private_data;

	uart_out32(0, ULITE_CONTROL, port);
	uart_in32(ULITE_CONTROL, port); /* ダミー */
	free_irq(port->irq, port);
	clk_disable(pdata->clk);
}
```

`.startup`がハードウェアの立ち上げのすべてである。クロックを有効にし、IRQを要求し、両方のFIFOをリセットし、それから制御レジスタの「割り込み許可」ビットを立てる。`.shutdown`はその正確な鏡像である。8250形式のドライバと比べて何がないかに注意してほしい。ここにはボーレートの設定がない。`uartlite`のボーレートはハードウェアで固定されているからである。このデバイスの`.set_termios`（次節）は、固定のレートを *報告* し返すだけで、設定することは決してない。

### `.set_termios` {#set_termios}

```c
// drivers/tty/serial/uartlite.c — ulite_set_termios()、抜粋
static void ulite_set_termios(struct uart_port *port,
			      struct ktermios *termios,
			      const struct ktermios *old)
{
	unsigned long flags;
	struct uartlite_data *pdata = port->private_data;

	/* termiosをハードウェアがサポートする値に設定する */
	termios->c_iflag &= ~BRKINT;
	termios->c_cflag &= ~(CSTOPB | PARENB | PARODD | CSIZE);
	termios->c_cflag |= pdata->cflags & (PARENB | PARODD | CSIZE);
	tty_termios_encode_baud_rate(termios, pdata->baud, pdata->baud);

	uart_port_lock_irqsave(port, &flags);

	port->read_status_mask = ULITE_STATUS_RXVALID | ULITE_STATUS_OVERRUN
		| ULITE_STATUS_TXFULL;
	...
```

ここでの`.set_termios`はネゴシエートするのではなく、呼び出し元が要求したものを、ハードウェアの実際の値（`pdata->baud`と、この合成されたコアがどのパリティ/データビットで作られたか）で *上書き* し、それから`tty_termios_encode_baud_rate()`を呼んで、それを汎用の`ktermios`を通じて報告し返す。`port->read_status_mask`は`uart_insert_char()`が見るものではない。あのヘルパーが参照するのは`ignore_status_mask`だけである。`read_status_mask`はドライバ自身が適用する。`ulite_receive()`はステータスビットに基づいて動作する前に`stat &= port->read_status_mask`を行う（8250ドライバも`lsr &= port->read_status_mask`で同等のことを行う）。したがって、ここでどのステータスビットが意味を持つかを決めるのは`uart_insert_char()`ではなく、このドライバ側のマスク処理である。

### 割り込みハンドラとRX/TXのコールバック {#interrupt-handler-and-the-rxtx-callbacks}

```c
// drivers/tty/serial/uartlite.c
static irqreturn_t ulite_isr(int irq, void *dev_id)
{
	struct uart_port *port = dev_id;
	int stat, busy, n = 0;
	unsigned long flags;

	do {
		uart_port_lock_irqsave(port, &flags);
		stat = uart_in32(ULITE_STATUS, port);
		busy  = ulite_receive(port, stat);
		busy |= ulite_transmit(port, stat);
		uart_port_unlock_irqrestore(port, flags);
		n++;
	} while (busy);

	/* 仕事をしたか? */
	if (n > 1) {
		tty_flip_buffer_push(&port->state->port);
		return IRQ_HANDLED;
	} else {
		return IRQ_NONE;
	}
}
```

8250のISRがどのステータスビットがセットされているかで分岐するのに対し、`uartlite`のISRは起床するたびに`ulite_receive()`と`ulite_transmit()`の両方を呼び、どちらもそれ以上の仕事を報告しなくなるまでループするだけである。これは、プログラム可能なFIFOトリガーレベルを持たないUARTに合った、より単純でレジスタ数に左右されにくい形である。`uartlite`は実際に深さ16のハードウェアFIFOを持っている（`port->fifosize = 16`で、RXフル/TXフルのステータスビットと、`.startup`でのFIFOリセットがある）。8250と違って欠けているのは、そのFIFOをバッチで、トリガーレベルを意識して吸い出したり詰め込んだりする手段である。各割り込みは、8250形式の`tx_loadsz`単位のバーストではなく、ステータス/RX/TXのレジスタを通して1バイトずつデータを動かす。`ulite_receive()`は、`uart_insert_char()`を経由せず、`tty_insert_flip_char()`で直接TTY層に到達する。

```c
// drivers/tty/serial/uartlite.c — ulite_receive()、抜粋
if (stat & ULITE_STATUS_RXVALID) {
	port->icount.rx++;
	ch = uart_in32(ULITE_RX, port);
	...
}
...
if (stat & ULITE_STATUS_RXVALID)
	tty_insert_flip_char(tport, ch, flag);
```

そして`ulite_transmit()`は8250の`uart_fifo_get()`のパターンを正確になぞるが、8250形式のバッチでのロードではなく、割り込みごとに1バイトだけをハードウェアFIFOに移す。

```c
// drivers/tty/serial/uartlite.c — ulite_transmit()、抜粋
if (!uart_fifo_get(port, &ch))
	return 0;

uart_out32(ch, ULITE_TX, port);

/* 起床させる */
if (kfifo_len(&tport->xmit_fifo) < WAKEUP_CHARS)
	uart_write_wakeup(port);
```

`uart_insert_char()`も、ドライバ自身による直接の`tty_insert_flip_char()`呼び出しも、どちらも正当で、実在する、ツリー内のパターンである。`uart_insert_char()`は、ほとんどのドライバが欲しがる`ignore_status_mask`/オーバーランの管理をただで追加してくれる。このドライバのように、より単純なステータスのセマンティクスを持つドライバは、TTY層のプリミティブを直接呼ぶだけのこともある。

### 登録 {#registration}

```c
// drivers/tty/serial/uartlite.c
static struct uart_driver ulite_uart_driver = {
	.owner		= THIS_MODULE,
	.driver_name	= "uartlite",
	.dev_name	= ULITE_NAME,
	.major		= ULITE_MAJOR,
	.minor		= ULITE_MINOR,
	.nr		= ULITE_NR_UARTS,
#ifdef CONFIG_SERIAL_UARTLITE_CONSOLE
	.cons		= &ulite_console,
#endif
};
```

そして（`.probe()`から呼ばれる）`ulite_assign()`の中に、特定の`uart_port`をそのドライバに結びつける2つの呼び出しがある。

```c
// drivers/tty/serial/uartlite.c — ulite_assign()、抜粋
port = &ulite_ports[id];

spin_lock_init(&port->lock);
port->fifosize = 16;
port->regshift = 2;
port->iotype = UPIO_MEM;
port->iobase = 1; /* ポートを使用中としてマークする */
port->mapbase = base;
port->membase = NULL;
port->ops = &ulite_ops;
port->irq = irq;
port->flags = UPF_BOOT_AUTOCONF;
port->dev = dev;
port->type = PORT_UNKNOWN;
port->line = id;
port->private_data = pdata;

/* ポートを登録する */
rc = uart_add_one_port(&ulite_uart_driver, port);
```

`uart_register_driver(&ulite_uart_driver)`は、どのポートも現れる前に、モジュール初期化から1回だけ実行される。`uart_add_one_port()`は、`.probe()`が見つけた物理ポートごとに1回実行される。`uartlite`の場合、合成された`uartlite`コアはそれぞれが別個のデバイスなので、これはデバイスツリーのノードごとに1回ということになる。

## ハードウェアからユーザー空間までの全経路 {#the-full-path-hardware-to-userspace}

```
 UART hardware (8250, uartlite, PL011, ...)
        │  RBR/THR, LSR, IER, ... registers  ▲
        │  RX byte ready (IRQ)               │  TX register write
        ▼                                    │
 ┌────────────────────────────────────────────────┐
 │  driver interrupt handler                      │
 │  (serial8250_handle_irq_locked / ulite_isr)    │
 │                                                │
 │  RX: serial8250_read_char() / ulite_receive()  │
 │        → uart_insert_char() or                 │
 │          tty_insert_flip_char() directly       │
 │        → tty_flip_buffer_push()                │
 │                                                │
 │  TX: uart_fifo_get() drains xmit_fifo          │
 │        → serial_out()/uart_out32() to hw       │
 └───────────────────┬────────────────────────────┘
                     │  both directions go through struct uart_ops
                     │  (.startup/.shutdown/.set_termios/.start_tx/
                     │   .stop_tx/.set_mctrl/.get_mctrl/.tx_empty/...)
                     ▼
        serial_core.c (drivers/tty/serial/serial_core.c)
        ┌──────────────────────────────────────────────────────────────┐
        │ struct uart_port  ←──ops──→ struct uart_ops (driver's table) │
        │      │ state                                                 │
        │      ▼                                                       │
        │ struct uart_state { tty_port, uart_port }                    │
        └────────────────────────┬─────────────────────────────────────┘
                                 │  drv->state[port->line], shared
                                 │  static const struct tty_operations uart_ops
                                 ▼
              generic tty_operations (.open/.close/.write/.set_termios/
              .ioctl/.throttle/.tiocmget/.tiocmset/...)
                                 ▼
                    TTY core (drivers/tty/tty_io.c)
                                 ▼
                  line discipline (n_tty by default —
                  see line-disciplines.md)
                                 ▼
                    userspace: read()/write() on
                    /dev/ttyS0, /dev/ttyUL0, ...
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [`include/linux/serial_core.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/serial_core.h) — `struct uart_ops`、`struct uart_port`、`struct uart_driver`、`struct uart_state`、`uart_insert_char()`、`uart_fifo_get()`
- [`drivers/tty/serial/serial_core.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/tty/serial/serial_core.c) — `uart_register_driver()`、共有の`tty_operations`、`uart_start()`/`uart_port_startup()`/`uart_change_line_settings()`、`uart_insert_char()`
- [`drivers/tty/serial/serial_port.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/tty/serial/serial_port.c) — `uart_add_one_port()`/`uart_remove_one_port()`
- [`include/linux/serial_8250.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/serial_8250.h) — `struct uart_8250_port`
- [`drivers/tty/serial/8250/8250_port.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/tty/serial/8250/8250_port.c) — `serial8250_pops`、`serial8250_handle_irq_locked()`、`serial8250_rx_chars()`/`serial8250_read_char()`、`serial8250_tx_chars()`
- [`include/uapi/linux/serial_reg.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/serial_reg.h) — 8250/16550のレジスタオフセットとビット定義（`UART_IER`、`UART_FCR`、`UART_LCR`、`UART_MCR`、`UART_LSR`、`UART_MSR`、...）
- [`drivers/tty/serial/uartlite.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/tty/serial/uartlite.c) — 上の実例で扱ったドライバ
- [`include/linux/tty_flip.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/tty_flip.h) — `tty_insert_flip_char()`、`tty_flip_buffer_push()`

### 関連ページ {#related-pages}

- [TTYサブシステム](README.md) — 汎用のTTYコア、`tty_struct`、そしてラインディシプリンとドライバがそれぞれどこに位置するか
- [ラインディシプリン](line-disciplines.md) — `tty_flip_buffer_push()`がバイトを`n_tty`や他のldiscに渡した後、そのバイトに何が起きるか
- [TTY/シリアルの事件簿](war-stories.md) — このページで説明する層で実際に起きた事件

### 外部資料 {#external}

- [Serial drivers](https://docs.kernel.org/driver-api/serial/driver.html) — `uart_ops`、`uart_port`、シリアルドライバが使うヘルパー関数に関する、カーネル自身のドライバAPIリファレンス
