# ラインディシプリン、termios、擬似端末 {#line-disciplines-termios-and-pseudo-terminals}

> UART（またはpty）とシェルの間に位置するインタプリタ。生のバイトを編集済みの行に、Ctrl-CをSIGINTに、バックグラウンドからの書き込みをSIGTTOUに変える

## 問題: 生のバイトはシェルが読みたいものではない {#the-problem-raw-bytes-are-not-what-a-shell-wants-to-read}

シリアルポートやptyドライバの仕事は、バイトを動かすことだけである。バイトは回線から（あるいはptyペアのもう一方から）到着し、バイトは出ていく。その層には、バックスペースが何を意味するのか、Ctrl-Cでフォアグラウンドのプログラムを終了させるべきか、`read()`を呼んでいるプロセスがバックグラウンドで動いているせいでそもそもいまこの入力を受け取ってよいのか、といったことを知っているものは何もない。端末を使うすべてのプログラムが、生のデバイスバイトの上に行編集、シグナル生成、ジョブ制御のポリシーを再実装しなければならないとしたら、あらゆるシェル、あらゆる`readline`もどきが、同じ数百行の細かいコードを重複して持つことになり、しかも細部で食い違うことになる。

Linuxのtty層は、低レベルドライバと読み書きするプロセスの間に **ラインディシプリン** を挿入することでこれを解決する。ラインディシプリンは差し替え可能なプロトコルハンドラである。ドライバは到着した生のバイトをそれに渡し、ラインディシプリンは処理済みのバイトを`read()`に渡す。逆方向では、プロセスが`write()`したものをドライバに渡す前に変換できる。圧倒的に最もよく使われるのは **N_TTY** である。これは普通の対話端末の振る舞い、すなわちカノニカルな行編集、エコー、シグナル生成を実装するデフォルトのディシプリンである。このページは主にN_TTYと、そのすぐ隣にある2つの仕組みを扱う。プロセスがN_TTYの振る舞いを設定するのに使う構造体`termios`と、そもそも誰が特定の端末を読み書きしてよいかを決めるプロセスグループの管理であるジョブ制御である。最後に擬似端末（pty）を扱う。ptyはマスター/スレーブのデバイスのペアであり、N_TTYとジョブ制御を物理シリアルポートだけでなく、ネットワークログインやターミナルエミュレータにも適用できるようにするものである。

## ラインディシプリン: `struct tty_ldisc_ops` {#line-disciplines-struct-tty_ldisc_ops}

ラインディシプリンは`struct tty_ldisc_ops`（`include/linux/tty_ldisc.h`）を`tty_register_ldisc()`で登録する。この構造体自身のkernel-docコメントは、コールバックの一覧を真ん中で区切っている。一部のフックは「上から」（ttyに対して`read()`/`write()`/`ioctl()`を行うプロセスから）呼ばれ、それ以外は「下から」（ハードウェアで受信したバイトを流し込むttyドライバから）呼ばれる。

```c
// include/linux/tty_ldisc.h
struct tty_ldisc_ops {
	char	*name;
	int	num;

	/*
	 * 以下のルーチンは上から呼ばれる。
	 */
	int	(*open)(struct tty_struct *tty);
	void	(*close)(struct tty_struct *tty);
	void	(*flush_buffer)(struct tty_struct *tty);
	ssize_t	(*read)(struct tty_struct *tty, struct file *file, u8 *buf,
			size_t nr, void **cookie, unsigned long offset);
	ssize_t	(*write)(struct tty_struct *tty, struct file *file,
			 const u8 *buf, size_t nr);
	int	(*ioctl)(struct tty_struct *tty, unsigned int cmd,
			unsigned long arg);
	int	(*compat_ioctl)(struct tty_struct *tty, unsigned int cmd,
			unsigned long arg);
	void	(*set_termios)(struct tty_struct *tty, const struct ktermios *old);
	__poll_t (*poll)(struct tty_struct *tty, struct file *file,
			     struct poll_table_struct *wait);
	void	(*hangup)(struct tty_struct *tty);

	/*
	 * 以下のルーチンは下から呼ばれる。
	 */
	void	(*receive_buf)(struct tty_struct *tty, const u8 *cp,
			       const u8 *fp, size_t count);
	void	(*write_wakeup)(struct tty_struct *tty);
	void	(*dcd_change)(struct tty_struct *tty, bool active);
	size_t	(*receive_buf2)(struct tty_struct *tty, const u8 *cp,
				const u8 *fp, size_t count);
	void	(*lookahead_buf)(struct tty_struct *tty, const u8 *cp,
				 const u8 *fp, size_t count);

	struct  module *owner;
};
```

`read`/`write`/`ioctl`/`open`/`close`はプロセス向けの半分である。ttyのファイルディスクリプタに対する`read()`は最終的にldiscの`.read`を呼び、`write()`は`.write`を呼ぶ。`receive_buf`/`receive_buf2`はドライバ向けの半分である。UARTの割り込みハンドラ（ptyの場合はペアのもう一方）は、新しいバイトを得るたびに、読み取りバッファに直接触れるのではなく、現在のldiscの`receive_buf`/`receive_buf2`を呼び出す。`.set_termios`は、`tcsetattr()`によるtermiosの変更の後に発火する通知フックであり、ldiscはこれで内部状態を調整できる（N_TTYはこれを使って、rawモードが適用されるかどうかといった派生フラグを再計算する）。`tty->ldisc`が指す実行時オブジェクトである`struct tty_ldisc`自体は、opsポインタと、所有者であるttyへの逆参照にすぎない。

```c
// include/linux/tty_ldisc.h
struct tty_ldisc {
	const struct tty_ldisc_ops *ops;
	struct tty_struct *tty;
};
```

N_TTYの登録を見ると、実際のディシプリンがこれらのうちどれを埋めているかがわかる。`.compat_ioctl`はなく、`.dcd_change`もない（これはN_PPS専用である）。そして`.lookahead_buf`は、メインの受信パスが走る前にフロー制御文字を探すヘルパーを指している。

```c
// drivers/tty/n_tty.c
static struct tty_ldisc_ops n_tty_ops = {
	.owner		 = THIS_MODULE,
	.num		 = N_TTY,
	.name            = "n_tty",
	.open            = n_tty_open,
	.close           = n_tty_close,
	.flush_buffer    = n_tty_flush_buffer,
	.read            = n_tty_read,
	.write           = n_tty_write,
	.ioctl           = n_tty_ioctl,
	.set_termios     = n_tty_set_termios,
	.poll            = n_tty_poll,
	.receive_buf     = n_tty_receive_buf,
	.write_wakeup    = n_tty_write_wakeup,
	.receive_buf2	 = n_tty_receive_buf2,
	.lookahead_buf	 = n_tty_lookahead_flow_ctrl,
};
```

### ラインディシプリンはN_TTYだけではない {#its-not-the-only-line-discipline}

`include/uapi/linux/tty.h`は31個の`N_*`番号を予約しており、プロセスは`ioctl(fd, TIOCSETD, &ldisc_num)`で実行時にttyのディシプリンを差し替える。昔ながらのPPPやSLIPはまさにこうして動いていた。ユーザー空間のデーモン（`pppd`、`slattach`）がシリアルポートを普通のttyとして開き、`TIOCSETD`を実行してN_TTYから **N_PPP** や **N_SLIP** に切り替える。その時点から、回線上のすべてのバイトは端末入力ではなく、フレーム化されたPPP/SLIPのトラフィックになる。行編集もCtrl-Cももはやなく、独自のフレーミングを持つただのバイトのパイプである。

```c
// include/uapi/linux/tty.h
#define N_TTY		0
#define N_SLIP		1
#define N_MOUSE		2
#define N_PPP		3
...
#define N_HCI		15	/* Bluetooth HCI UART */
...
#define N_GSM0710	21	/* GSM 0710 多重化 */
...
#define N_MCTP		28	/* シリアル上のMCTP */
#define N_CAN327	30	/* ELM327ベースのOBD-IIインターフェース */
```

この一覧の残りの大半は、同じ手口を、UARTに載る他のフレーム化されたプロトコルに適用したものである。N_HCIはBluetoothコントローラのHCIパケットをデコードし、N_GSM0710はGSMモデムのために1つのシリアルポートを複数の論理チャネルに多重化し、N_CAN327は安価なELM327 OBD-IIアダプタと通信する。どれも人間がタイプする対象ではない。いずれも「ttyのバイトストリームを端末以外のプロトコルのために横取りする」ものであり、より特化したものが回線を確保していなければ、すぐにN_TTYが元の位置に戻る。このページの残りはすべてN_TTYについてである。

## カノニカルモードとrawモード {#canonical-vs-raw-mode}

N_TTYの最も目に見える仕事は、`read()`がデータを返す前にどれだけの編集を行うかを決めることであり、それは`c_lflag`の`ICANON`という1ビットで制御される。

- **カノニカル（cooked）モード** — `ICANON`がセットされている。これがデフォルトである。入力は1行ずつバッファされる。行が読み手に渡される前に、N_TTYはその場で編集を行う。消去文字（デフォルトは`DEL`/`Ctrl-?`）は最後の文字を削除し、kill文字（デフォルトは`Ctrl-U`）は行全体を先頭まで消去し、（`IEXTEN`があれば）単語消去（`Ctrl-W`）は最後の単語を削除する。カノニカルモードの`read()`は、改行、`EOF`、または`EOL`文字で終端された完全な1行が得られるまで何も返さない。
- **raw（非カノニカル）モード** — `ICANON`がクリアされている。行バッファリングも編集も一切ない。バイトは基本的に到着したとおりに`read()`から読めるようになり（後述の`VMIN`/`VTIME`のタイミング制御に従う）、消去/killのロジックに一切手を加えられない。全画面プログラム（エディタ、ページャ、`ssh`そのもの）が端末をこのモードにするのは、（本来なら消去文字になるものも含めて）すべてのキー入力をデータとして見る必要があるからである。

カノニカルモードの編集ロジックは、`drivers/tty/n_tty.c`の`eraser()`という1つの関数にあり、きっかけとなった文字が消去文字、単語消去文字、kill文字のどれかによって分岐する。

```c
// drivers/tty/n_tty.c（eraser()のシグネチャと分岐）
static void eraser(u8 c, const struct tty_struct *tty)
{
	...
	enum { ERASE, WERASE, KILL } kill_type;
	...
	if (c == ERASE_CHAR(tty))
		kill_type = ERASE;
	else if (c == WERASE_CHAR(tty))
		kill_type = WERASE;
	...
}
```

`ERASE_CHAR(tty)`とその仲間は、`tty->termios.c_cc[]`を包む薄いマクロである（後述）。

非カノニカルモードでは、さらに2つの`c_cc[]`スロットの意味がまったく変わる。`VMIN`と`VTIME`は`read()`がどのようにブロックするかを制御する。`VMIN`はreadを満たすのに必要な最小バイト数であり、`VTIME`は10分の1秒単位のタイムアウトである。4つの組み合わせ（両方ゼロ、`VMIN`のみ、`VTIME`のみ、両方非ゼロ）が4つの異なるブロッキングの振る舞いを生み、その詳細は`termios(3)`に記されている。

## `termios`: 設定のインターフェース {#termios-the-configuration-surface}

開かれたttyはそれぞれ`struct ktermios`（`tty->termios`）を持ち、ユーザー空間は`tcgetattr(3)`/`tcsetattr(3)`でそれを読み書きする。これらは`TCGETS`/`TCSETS`系のioctlを包む薄いラッパーである。カーネルのUAPI構造体である`struct termios`（`tcgetattr()`が実際に埋めるもの）と、カーネル内部の`struct ktermios`は、現代のLinuxでは同じフィールドレイアウトを共有している（`ktermios`はさらに、速度を`c_cflag`に詰め込む代わりに明示的な`c_ispeed`/`c_ospeed`フィールドを持つ）。

```c
// include/uapi/asm-generic/termbits.h
#define NCCS 19
struct termios {
	tcflag_t c_iflag;		/* 入力モードフラグ */
	tcflag_t c_oflag;		/* 出力モードフラグ */
	tcflag_t c_cflag;		/* 制御モードフラグ */
	tcflag_t c_lflag;		/* ローカルモードフラグ */
	cc_t c_line;			/* ラインディシプリン */
	cc_t c_cc[NCCS];		/* 制御文字 */
};

struct ktermios {
	tcflag_t c_iflag;		/* 入力モードフラグ */
	tcflag_t c_oflag;		/* 出力モードフラグ */
	tcflag_t c_cflag;		/* 制御モードフラグ */
	tcflag_t c_lflag;		/* ローカルモードフラグ */
	cc_t c_line;			/* ラインディシプリン */
	cc_t c_cc[NCCS];		/* 制御文字 */
	speed_t c_ispeed;		/* 入力速度 */
	speed_t c_ospeed;		/* 出力速度 */
};
```

`c_line`はldiscの番号（上の`N_*`値のいずれか）であり、`TIOCSETD`が読み書きするのと同じフィールドである。4つのフラグワードは互いに独立したビットマスクの名前空間で、パイプラインの段階ごとに1つずつある。

| フィールド | 制御対象 | 具体的なフラグの例 |
|---|---|---|
| `c_iflag` | 入力処理。ldiscがバイトを「行データ」として見る前に行われる | `ICRNL`（入力時にCR→NLに変換）、`IXON`/`IXOFF`（ソフトウェアフロー制御）、`ISTRIP`、`BRKINT` |
| `c_oflag` | 出力処理。プロセスが書き込んだものがドライバに届く前に適用される | `OPOST`（出力処理そのものを有効化する。`OPOST`がクリアされていると、このワードの他のフラグはすべて何もしない）、`ONLCR`（出力時にNL→CR-NLに変換） |
| `c_cflag` | ハードウェア回線そのもの | `CS8`（8ビットの文字サイズ。`CSIZE`マスクの一部）、`CREAD`、`PARENB`/`PARODD`、`CLOCAL`、ボーレートのビット |
| `c_lflag` | 「ローカル」な振る舞い。このページが主に扱う、ldiscレベルのポリシーのつまみ | `ICANON`、`ISIG`、`ECHO`、`IEXTEN`、`TOSTOP` |

`include/uapi/asm-generic/termbits.h`と`include/uapi/asm-generic/termbits-common.h`から直接確認した、最もよく参照されるフラグのビット値は次のとおりである。

```c
// include/uapi/asm-generic/termbits-common.h
#define ICRNL	0x100			/* 入力時にCRをNLに変換 */
#define OPOST	0x01			/* 出力処理を行う */

// include/uapi/asm-generic/termbits.h
#define CS8		0x00000030	/* CSIZEマスクの一部 */

/* c_lflagのビット */
#define ISIG	0x00001
#define ICANON	0x00002
#define ECHO	0x00008
```

`c_cc[]`配列（`NCCS` = 19スロット）は特殊文字が置かれる場所であり、プロセスは`stty`や直接の`tcsetattr()`呼び出しでこれらのバイトを再割り当てできる。配列は固定のバイト位置ではなく、シンボル定数でインデックスされる。

```c
// include/uapi/asm-generic/termbits.h
#define VINTR		 0
#define VQUIT		 1
#define VERASE		 2
#define VKILL		 3
#define VEOF		 4
#define VTIME		 5
#define VMIN		 6
#define VSWTC		 7
#define VSTART		 8
#define VSTOP		 9
#define VSUSP		10
#define VEOL		11
#define VREPRINT	12
#define VDISCARD	13
#define VWERASE		14
#define VLNEXT		15
#define VEOL2		16
```

`include/linux/tty.h`は、カーネル自身がこれらを読むのに使うマクロを定義している。`INTR_CHAR(tty)`は`tty->termios.c_cc[VINTR]`に展開され、上のすべての項目について同様である。Linuxでは、`VMIN`/`VTIME`と`VEOF`/`VEOL`は互いに重ならない4つの別々のインデックス（上の表のとおり4、5、6、11）である。これらが`VMIN`/`VTIME`としての *意味* を持つのは非カノニカルモードのときだけだが、スロット自体が使い回されることはない。（SPARCの`termbits.h`は歴史的な例外で、古いUnixのレイアウトを受け継いで文字どおり`VMIN VEOF`と`VTIME VEOL`を`#define`している。これはSPARC特有のものであって、Linux/glibc一般の規則ではない。）

## シグナル生成: Ctrl-CはどのようにSIGINTになるか {#signal-generation-how-ctrl-c-becomes-sigint}

`c_lflag`に`ISIG`がセットされている場合（デフォルト）、N_TTYは到着するすべてのバイトについて3つの特殊文字を監視し、それぞれをデータとして通す代わりにシグナルに変える。この振り分けが行われるのは`n_tty_receive_char_special()`であり、通常の行編集ロジックが走る前にチェックされる。ただし`n_tty_receive_char_special()`自体は、まず`I_IXON`によるソフトウェアフロー制御（`Ctrl-S`/`Ctrl-Q`）をチェックし、それがバイトを消費しなかった場合にのみ、以下の`ISIG`ブロックに到達する。

```c
// drivers/tty/n_tty.c
if (L_ISIG(tty)) {
	if (c == INTR_CHAR(tty)) {
		n_tty_receive_signal_char(tty, SIGINT, c);
		return;
	} else if (c == QUIT_CHAR(tty)) {
		n_tty_receive_signal_char(tty, SIGQUIT, c);
		return;
	} else if (c == SUSP_CHAR(tty)) {
		n_tty_receive_signal_char(tty, SIGTSTP, c);
		return;
	}
}
```

デフォルトの割り当ては、`VINTR`が`Ctrl-C` → `SIGINT`、`VQUIT`が`Ctrl-\` → `SIGQUIT`、`VSUSP`が`Ctrl-Z` → `SIGTSTP`である。これらはそれぞれ`isig()`を通る。`isig()`はシグナルを送るプロセスグループを解決し、`NOFLSH`がセットされていなければ、シグナル配送の一環としてldiscの保留中の入力/エコー/出力をフラッシュする（そのため、`SIGINT`の後に古い先行入力がバッファに残って次のフォアグラウンドプログラムに渡ってしまうことがない）。シグナル配送の部分（`tty->ctrl.pgrp`を解決して`kill_pgrp()`を呼ぶ）は、以下に示す内部ヘルパー`__isig()`に切り出されている。`NOFLSH`のチェックと実際のフラッシュは`isig()`自体にあり、まず`__isig()`を呼び、それから`NOFLSH`がセットされていなければエコー/出力/入力のバッファをクリアする。

```c
// drivers/tty/n_tty.c
static void __isig(int sig, struct tty_struct *tty)
{
	struct pid *tty_pgrp = tty_get_pgrp(tty);
	if (tty_pgrp) {
		kill_pgrp(tty_pgrp, sig, 1);
		put_pid(tty_pgrp);
	}
}
```

`tty_get_pgrp()`は`tty->ctrl.pgrp`（ttyのフォアグラウンドプロセスグループ。次節で扱う）への参照を返し、`kill_pgrp()`はそのグループ内のすべてのプロセスにシグナルを配送する。シェルでのCtrl-Cが1つのプロセスだけでなく、フォアグラウンドのパイプライン全体（例えば`sort | uniq | less`）に届くのはこのためである。これは完全にN_TTY独自のロジックであり、`ISIG`やシグナル生成に関するものは汎用のttyコアには一切ない。別のラインディシプリンは、これをまったく行わなくてもかまわない（実際PPP/SLIP系のディシプリンは行わない。フレーム化されたネットワークトラフィックを運ぶバイトストリームで`Ctrl-C`を解釈するのは無意味だからである）。

## ジョブ制御: 誰が読み書きしてよいか {#job-control-whos-allowed-to-read-and-write}

`ISIG`は、フォアグラウンドのプロセスがどのようにシグナルを受け取るかを説明する。ジョブ制御はその上の層であり、そもそも *どの* プロセスグループをフォアグラウンドとみなすか、そしてそれ以外のプロセスに何が起きるかを決める。

### `tty_struct`の制御フィールド {#tty_structs-control-fields}

関連する状態は、`struct tty_struct`の中の小さな`ctrl`サブ構造体に、専用のspinlockとともに置かれている。

```c
// include/linux/tty.h（struct tty_struct、ctrlフィールド）
struct {
	struct pid *pgrp;
	struct pid *session;
	spinlock_t lock;
	unsigned char pktstatus;
	bool packet;
} ctrl;
```

`ctrl.pgrp`は端末の現在のフォアグラウンドプロセスグループであり、`ctrl.session`はこのttyが制御端末と *なっている* セッションである。どちらも生のpidではなく`struct pid *`であり、書き込み側はどちらも`ctrl.lock`で保護される（構造体のkernel-docによれば、読み取り側は`ctrl.lock`か、ttyのより広い範囲を守る`legacy_mutex`のどちらか一方を保持していれば済む）。`tty_get_pgrp(tty)`は上で`isig()`が使っていたアクセサであり、`tty_jobctrl.c`には対応する設定側のパスである`tiocspgrp()`があり、`TIOCSPGRP` ioctlを通じて到達する。

### 制御端末になる {#becoming-a-controlling-terminal}

ttyがセッションの制御端末になる方法は2つある。1つは暗黙的な方法で、制御ttyを持たないセッションリーダーが、まだ制御セッションを持たないttyデバイスを開いたときである（`tty_open()`から呼ばれる`tty_open_proc_set_tty()`で処理される）。もう1つは明示的な方法で、`TIOCSCTTY` ioctl、すなわち`tiocsctty()`による。

```c
// drivers/tty/tty_jobctrl.c（tiocsctty()、中核ロジック）
static int tiocsctty(struct tty_struct *tty, struct file *file, int arg)
{
	...
	/*
	 * プロセスはセッションリーダーであり、かつ
	 * まだ制御ttyを持っていてはならない。
	 */
	if (!current->signal->leader || current->signal->tty) {
		ret = -EPERM;
		goto unlock;
	}

	if (tty->ctrl.session) {
		/* すでに別のセッショングループの制御ttyになっている! */
		if (arg == 1 && capable(CAP_SYS_ADMIN)) {
			/* 奪い取る */
			session_clear_tty(tty->ctrl.session);
		} else {
			ret = -EPERM;
			goto unlock;
		}
	}
	...
	proc_set_tty(tty);
	...
}
```

これはまさに`setsid()` + `open()` + `TIOCSCTTY`が行う手順である。ターミナルエミュレータ、`sshd`、`screen`/`tmux`はいずれも、新たに開いたptyスレーブを新しいセッションに渡すときに、この標準的なイディオムを使う。まず`setsid()`で、継承した制御端末から切り離され、制御端末を持たないセッションリーダーになる。次にスレーブを開いて`TIOCSCTTY`を発行し、それを自分のものにする。`proc_set_tty()`は、ttyの新しいpgrp/sessionとして呼び出し元の現在のプロセスグループとセッションを（`task_pgrp(current)`/`task_session(current)`経由で）記録し、それとは別に`current->signal->tty`（プロセス自身が持つ、自分の制御端末へのポインタ）を設定する。

### SIGTTINとSIGTTOU {#sigttin-and-sigttou}

セッションがフォアグラウンドの`ctrl.pgrp`を持つ制御ttyを得ると、*バックグラウンド* のグループに属するプロセスがその端末から`read()`しようとしたり、（`TOSTOP`によっては）その端末に`write()`しようとしたりすると、そのまま進むことは許されず停止させられる。この仕組みは`__tty_check_change()`であり、そのドキュメントコメントがポリシーを直接述べている。

```c
// drivers/tty/tty_jobctrl.c
/**
 *	__tty_check_change	-	POSIXの端末変更をチェックする
 *	@tty: チェックするtty
 *	@sig: 送るシグナル
 *
 *	端末に書き込もうとするか、端末の状態を設定しようとしたときに、
 *	フォアグラウンドにいなければSIGTTOUを送る。シグナルがブロックまたは
 *	無視されている場合は、そのまま操作を実行する。（POSIX 7.2）
 */
int __tty_check_change(struct tty_struct *tty, int sig)
{
	...
	if (tty_pgrp && pgrp != tty_pgrp) {
		if (is_ignored(sig)) {
			if (sig == SIGTTIN)
				ret = -EIO;
		} else if (is_current_pgrp_orphaned())
			ret = -EIO;
		else {
			kill_pgrp(pgrp, sig, 1);
			set_thread_flag(TIF_SIGPENDING);
			ret = -ERESTARTSYS;
		}
	}
	...
}
```

2つの呼び出し箇所が、これをそれぞれ読み取りパスと書き込みパスに組み込んでいる。`drivers/tty/n_tty.c`の`job_control()`は`n_tty_read()`の先頭で実行され、呼び出し元がフォアグラウンドのグループにいなければ`SIGTTIN`を要求する。

```c
// drivers/tty/n_tty.c
static int job_control(struct tty_struct *tty, struct file *file)
{
	/* ジョブ制御のチェック -- 開始時と、スリープの後に
	   毎回行わなければならない（POSIX.1 7.1.1.4）。 */
	...
	/* /dev/consoleでは停止しない */
	if (file->f_op->write_iter == redirected_tty_write)
		return 0;

	return __tty_check_change(tty, SIGTTIN);
}
```

そしてエクスポートされている`tty_check_change()`（常に`SIGTTOU`）は、書き込みパスと、ttyの状態を変更する一握りのioctl（`TIOCSETD`、break関連のioctl、`TIOCSPGRP`）が、処理を進める前に呼ぶものである。結果として、バックグラウンドジョブの`read()`は`SIGTTIN`によってそのプロセスグループ全体を止める（デフォルトではグループを停止させる。バックグラウンドに回した`cat`がstdinを読もうとした瞬間にエラー終了するのではなく固まるのはこのためである）。そして、`c_lflag`に`TOSTOP`がセットされている場合に限り、`write()`にも`SIGTTOU`経由で同じことが起きる。`TOSTOP`がなければバックグラウンドからの書き込みは通される。バックグラウンドで走らせたビルドのはぐれたログ行がシェルのプロンプトに混ざってしまうのは、たいていこれが理由である。

## 擬似端末: pty {#pseudo-terminals-ptys}

ここまでの内容はすべて、プロセスが実際の端末デバイスと通信していることを前提としていた。今日動いているものの中で実際にそうなっているものはほとんどない。SSHセッション、ターミナルエミュレータのウィンドウ、`tmux`、`expect`スクリプト、コンテナランタイムの`-t`フラグは、どれも **擬似端末** の上に構築されている。擬似端末では、一方の端の「ハードウェア」は単なる別のプロセスである。

### 1つのlinkを共有するマスターとスレーブ {#master-and-slave-sharing-one-link}

ptyは、上の`struct tty_struct`にすでに見えている`link`フィールド（`struct tty_struct *link;` — 「別のptyへのリンク（マスター → スレーブ、およびその逆）」）を通じて互いを直接参照し合う、`tty_struct`の *ペア* である。このペアが実際に構築されるのは`drivers/tty/pty.c`の`pty_common_install()`であり、マスターが最初に開かれたときに行われる。

```c
// drivers/tty/pty.c（pty_common_install()、抜粋）
static int pty_common_install(struct tty_driver *driver, struct tty_struct *tty,
		bool legacy)
{
	struct tty_struct *o_tty;
	...
	o_tty = alloc_tty_struct(driver->other, idx);
	...
	/* 双方向のリンクを確立する */
	tty->link   = o_tty;
	o_tty->link = tty;
	tty_port_init(ports[0]);
	tty_port_init(ports[1]);
	...
	o_tty->port = ports[0];
	tty->port = ports[1];
	...
}
```

2つの`tty_struct`、2つの`tty_port`、そして双方向に1本ずつの`link`ポインタである。一方の側に書き込まれたデータは、どのドライバのハードウェアにも行かない。`pty_write()`はバイトを相手側の入力バッファに直接詰め込むだけである。

```c
// drivers/tty/pty.c
static ssize_t pty_write(struct tty_struct *tty, const u8 *buf, size_t c)
{
	struct tty_struct *to = tty->link;

	if (tty->flow.stopped || !c)
		return 0;

	return tty_insert_flip_string_and_push_buffer(to->port, buf, c);
}
```

スレーブ側は、物理端末であるかのようにまったく普通のラインディシプリン（デフォルトではN_TTY）を動かす。カノニカル編集、`ISIG`、ジョブ制御のすべてがそのまま適用される。マスター側は *もう一方の* プログラム（ターミナルエミュレータ、`sshd`、`tmux`のサーバー）が読み書きする側であり、取り立てて言うほどのラインディシプリン処理は持たない。マスターから見れば、スレーブのldiscが生成するものとの間でバイトを流しているだけだからである。

ここは、ptyのシグナル配送に追加の経路がある場所でもある。`TIOCSIG`はマスターだけが発行できるioctlであり、マスター側の制御プロセスが、ldiscの文字照合を一切経由せずに、スレーブのフォアグラウンドグループに`SIGINT`/`SIGQUIT`/`SIGTSTP`を直接注入できるようにする。これは、シグナル要求をストリーム中のリテラルなバイトとしてではなく帯域外で受け取るターミナルエミュレータや`sshd`（SSHの`~C`のようなエスケープ、あるいはGUIの「breakを送る」操作）にとって有用である。

```c
// drivers/tty/pty.c
static int pty_signal(struct tty_struct *tty, int sig)
{
	struct pid *pgrp;

	if (sig != SIGINT && sig != SIGQUIT && sig != SIGTSTP)
		return -EINVAL;

	if (tty->link) {
		pgrp = tty_get_pgrp(tty->link);
		if (pgrp)
			kill_pgrp(pgrp, sig, 1);
		put_pid(pgrp);
	}
	return 0;
}
```

### `/dev/ptmx`と`/dev/pts/N`: Unix98インターフェース {#devptmx-and-devptsn-the-unix98-interface}

現代のあらゆるプログラムが使うインターフェースはUnix98のptyモデルである。単一の多重化デバイス`/dev/ptmx`と、割り当てられた各スレーブを`/dev/pts/N`として公開する仮想ファイルシステム`devpts`からなる。`/dev/ptmx`を開くと（`drivers/tty/pty.c`の`ptmx_open()`）、新しいマスター/スレーブのペアが割り当てられ、`devpts_new_index()`で未使用のインデックスが選ばれ、`devpts_pty_new()`を通じて対応する`/dev/pts/N`エントリが作成される。これが置き換えたレガシーなBSD形式の`/dev/ptyXX`/`/dev/ttyXX`のペアとは異なり、探し回るべき既存のデバイスノードはない。

ユーザー空間からの標準的な手順は4つの呼び出しからなり、glibcの`openpty()`はその4つすべてを代わりに行ってくれる便利なラッパーである。

1. **`posix_openpt(O_RDWR | O_NOCTTY)`** — `/dev/ptmx`を開き、マスターのファイルディスクリプタを返す。
2. **`grantpt(fd)`** — 対応するスレーブデバイスの所有者/パーミッションを整える（伝統的にはsetuidヘルパーを実行して行っていたが、Linuxでは代わりに`ptmxmode`/`gid=tty`のような`devpts`のマウントオプションでほぼ処理される）。
3. **`unlockpt(fd)`** — ptyのロックフラグ（`TIOCSPTLCK`）をクリアし、スレーブを実際に開けるようにする。新しいptyはすべてロックされた状態で始まる（`ptmx_open()`は`set_bit(TTY_PTY_LOCK, &tty->flags); /* LOCK THE SLAVE */`によって、無条件に *マスター* のフラグに`TTY_PTY_LOCK`をセットする。`ptmx_open()`内の`tty`は`tty = tty_init_dev(ptm_driver, index)`から得たマスターである。このコメントは実際上の効果を述べている。スレーブを開くパスである`pty_open()`は、`tty->link->flags`を通じてこのビットをチェックするからである）。
4. **`ptsname(fd)`**（または競合のないカーネル内の`TIOCGPTPEER`/`ptm_open_peer()`パス） — スレーブの`/dev/pts/N`パス、または直接のfdを解決する。

ロックが解除されれば、返されたスレーブのパスを開くと、N_TTYが動いている普通のttyファイルディスクリプタが得られる。そこから、スレーブを開いたプロセスは通常`setsid()`を行い、それに対して`TIOCSCTTY`を発行する。これはまさに、あらゆるターミナルエミュレータ（`xterm`、GUIのターミナル、`tmux`）とあらゆる`sshd`セッションが子シェルをセットアップする方法である。シェルの制御端末はptyの *スレーブ* であり、エミュレータや`sshd`のプロセスが *マスター* 側を保持して、「ユーザーのマウス/キーボード/ネットワークソケットが生成するもの」と「本物の端末なら送っていたもの」との間を変換する。

## 図 {#diagrams}

実際のシリアル回線での、キー入力からフォアグラウンドの`read()`まで:

```
 keyboard keystroke
        │
        ▼
 UART hardware / interrupt handler
        │  driver calls tty_insert_flip_char() + tty_flip_buffer_push()
        ▼
 tty_struct->ldisc  (N_TTY, the default)
        │
        │  .receive_buf() / .receive_buf2()
        ▼
 n_tty_receive_buf() → n_tty_receive_char_special() / n_tty_receive_char()
        │
        ├─ ISIG match (Ctrl-C/Ctrl-\/Ctrl-Z) ──► isig() ──► kill_pgrp(tty->ctrl.pgrp, SIG*)
        │                                                    delivered to every process in
        │                                                    the terminal's foreground group
        │
        ├─ ICANON editing (erase/kill/werase via eraser())
        │
        ├─ ECHO (echo_char(), if L_ECHO(tty))
        │
        ▼
 read buffer (struct n_tty_data.read_buf)
        │
        │  process calls read(fd, ...)
        ▼
 job_control(): __tty_check_change(tty, SIGTTIN)
   - caller in tty->ctrl.pgrp?  → proceed, n_tty_read() returns data
   - caller in a background pgrp? → SIGTTIN to that pgrp, read() blocks/restarts
```

ptyのマスター/スレーブのペア、ターミナルエミュレータとシェル:

```
 ┌─────────────────────────┐                    ┌─────────────────────────┐
 │   Terminal emulator     │                    │   Shell (bash, etc.)    │
 │   (or sshd)             │                    │                         │
 │   holds the MASTER fd   │                    │   controlling terminal  │
 └───────────┬─────────────┘                    │   is the SLAVE          │
             │                                  └──────────────────┬──────┘
             │ open("/dev/ptmx")                                   │ open("/dev/pts/N")
             ▼                                                     ▼
   ┌─────────────────────────┐   tty->link  /  o_tty->link    ┌─────────────────────────┐
   │  master tty_struct      │◄──────────────────────────────►│  slave tty_struct       │
   │  (no ldisc processing;  │                                │  ldisc = N_TTY          │
   │   pty_write() just      │   pty_write(): bytes written   │  (canonical editing,    │
   │   pushes bytes to       │   on one side land straight    │   ISIG, job control     │
   │   the other side's      │   in the other side's flip     │   all apply here)       │
   │   flip buffer)          │   buffer, no wire involved     │                         │
   └─────────────────────────┘                                └─────────────────────────┘
             ▲                                                     │
             │ user types in the emulator window /                 │ shell's stdin/stdout/
             │ bytes arrive over the SSH network connection        │ stderr are all fds on
             │                                                     │ the slave
             └─────────────── userspace I/O ───────────────────────┘
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [`include/linux/tty.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/tty.h) — `struct tty_struct`（`ctrl.pgrp`/`ctrl.session`を含む）、`c_cc[]`のアクセサマクロ（`INTR_CHAR()`、`ERASE_CHAR()`、...）、`L_*`/`I_*`/`O_*`/`C_*`のフラグ判定マクロ
- [`include/linux/tty_ldisc.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/tty_ldisc.h) — `struct tty_ldisc_ops`、`struct tty_ldisc`
- [`include/uapi/linux/tty.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/tty.h) — `N_*`ラインディシプリン番号の完全な一覧
- [`include/uapi/asm-generic/termbits.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/asm-generic/termbits.h)、[`termbits-common.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/asm-generic/termbits-common.h) — `struct termios`/`struct ktermios`、`c_cc[]`の`V*`インデックス、`c_iflag`/`c_oflag`/`c_cflag`/`c_lflag`のビット定義
- [`drivers/tty/n_tty.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/tty/n_tty.c) — N_TTYラインディシプリン: `eraser()`、`isig()`、`n_tty_receive_char_special()`、`job_control()`
- [`drivers/tty/pty.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/tty/pty.c) — `pty_common_install()`、`pty_write()`、`pty_signal()`、`ptmx_open()`、レガシーBSDとUnix98のドライバのセットアップ
- [`drivers/tty/tty_jobctrl.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/tty/tty_jobctrl.c) — `tiocsctty()`、`__tty_check_change()`、`tiocgpgrp()`/`tiocspgrp()`、`disassociate_ctty()`

### manページ {#man-pages}

- [`termios(3)`](https://man7.org/linux/man-pages/man3/termios.3.html) — `struct termios`のフィールドの完全なリファレンス、すべての`c_cc[]`特殊文字、カノニカル/非カノニカルでの`VMIN`/`VTIME`のセマンティクス
- [`ioctl_tty(2)`](https://man7.org/linux/man-pages/man2/ioctl_tty.2.html) — 端末/ジョブ制御のioctl（`TIOCSCTTY`、`TIOCGPGRP`/`TIOCSPGRP`、`TIOCGSID`、`TIOCSIG`）。man-pagesプロジェクトでは、このページが古い`tty_ioctl(4)`という名前に取って代わった
- [`pty(7)`](https://man7.org/linux/man-pages/man7/pty.7.html) — マスター/スレーブのモデル、`/dev/ptmx` + `/dev/pts/N`、`posix_openpt()`/`grantpt()`/`unlockpt()`、レガシーなBSD形式のインターフェース

### 関連ページ {#related-pages}

- [TTYとシリアルのサブシステム](README.md) — このページのラインディシプリンとジョブ制御の仕組みが載っている、ドライバ/`tty_struct`/`tty_port`の層
- [シリアルドライバとUART](serial.md) — このページで説明するラインディシプリンにバイトを流し込む、ハードウェア側
- [TTY/シリアルの事件簿](war-stories.md) — ここで扱ったラインディシプリン、ジョブ制御、ptyのコードで起きた事件

### LWN記事 {#lwn-articles}

- [Containers, pseudo TTYs, and backward compatibility](https://lwn.net/Articles/688809/) — レガシーなBSDの`/dev/ptyXX`/`/dev/ttyXX`のptyモデルと、それを置き換えたUnix98の`/dev/ptmx`+`devpts`との比較、そして古いシングルトンの前提がなぜいまもコンテナで問題を起こすのか
