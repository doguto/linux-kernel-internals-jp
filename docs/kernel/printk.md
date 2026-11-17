# printk: カーネルロギングの内部構造 {#printk-kernel-logging-internals}

> カーネルのロギングシステム: ログバッファ、レート制限、dev_printk、dmesg

## printk API {#the-printk-api}

```c
/* 基本的な使い方: */
printk(KERN_INFO "my_driver: initialized, irq=%d\n", irq);

/* 便利マクロ（推奨）: */
pr_emerg("...");    /* KERN_EMERG   = KERN_SOH "0" */
pr_alert("...");    /* KERN_ALERT   = KERN_SOH "1" */
pr_crit("...");     /* KERN_CRIT    = KERN_SOH "2" */
pr_err("...");      /* KERN_ERR     = KERN_SOH "3" */
pr_warn("...");     /* KERN_WARNING = KERN_SOH "4" */
pr_notice("...");   /* KERN_NOTICE  = KERN_SOH "5" */
pr_info("...");     /* KERN_INFO    = KERN_SOH "6" */
pr_debug("...");    /* KERN_DEBUG   = KERN_SOH "7"（DEBUG が定義されている場合のみ） */

/* デバイスを意識したラッパー（出力にデバイス名/アドレスを含める）: */
dev_err(&pdev->dev, "failed to map IO: %d\n", ret);
dev_info(&client->dev, "probe OK, addr=0x%02x\n", client->addr);
dev_warn(&pdev->dev, "timeout, retrying\n");
dev_dbg(&pdev->dev, "IRQ status: 0x%08x\n", status);
```

`KERN_SOH`（`include/linux/kern_levels.h`）はリテラルのASCII SOH制御文字（`"\001"`）であり、
整形された`dmesg`の行に表示される山括弧ではない — 山括弧は、dmesgがメッセージからSOH接頭辞付きのログレベルバイトを
パースして取り出した後に、`<level>`接頭辞としてdmesg自身が描画したものである。

## ログレベルとコンソール {#log-levels-and-console}

```c
/*
 * メッセージは、そのログレベルが現在の console_loglevel より
 * 小さい場合にコンソール（シリアル、VT）へ出力される:
 */
cat /proc/sys/kernel/printk
# 4  4  1  7
# current  default  min  boot-time-default

/* レベル:
   0 EMERG   — システムが使用不能
   1 ALERT   — 直ちに対処が必要
   2 CRIT    — 致命的な状態
   3 ERR     — エラー状態
   4 WARNING — 警告状態（多くのディストリビューションは console_loglevel を 4 に設定する。
                アップストリームの Kconfig デフォルト CONFIG_CONSOLE_LOGLEVEL_DEFAULT は 7）
   5 NOTICE  — 正常だが重要
   6 INFO    — 情報
   7 DEBUG   — デバッグメッセージ（デフォルトレベルでは決して表示されない）
*/

/* DEBUG を含むすべてのメッセージを一時的に表示する: */
echo 8 > /proc/sys/kernel/printk
```

## 循環ログバッファ {#circular-log-buffer}

```c
/*
 * リングバッファはメタデータ付きのレコードを格納する。
 * サイズは CONFIG_LOG_BUF_SHIFT で制御される（デフォルト 17 = 128KB）
 * ブートパラメータ: log_buf_len=4M
 */

/* kernel/printk/printk_ringbuffer.h */
struct printk_info {
    u64     seq;         /* シーケンス番号 */
    u64     ts_nsec;     /* ナノ秒単位のタイムスタンプ */
    u16     text_len;    /* メッセージテキストの長さ */
    u8      facility;    /* syslog ファシリティ */
    u8      flags:5;     /* 内部レコードフラグ */
    u8      level:3;     /* syslog レベル、0-7 */
    u32     caller_id;   /* スレッド ID またはプロセッサ ID */

    struct dev_printk_info dev_info;  /* サブシステム/デバイス。dev_printk() の呼び出し元用 */
};
```

`struct dev_printk_info`（`include/linux/dev_printk.h`）は単に2つの固定長char配列 —
`subsystem`/`device` — であり、`dev_err()`/`dev_info()`などによって埋められる。これにより`/dev/kmsg`の読み手は、
整形済みテキストをパースすることなく、メッセージがどのデバイスから来たかを復元できる。より新しい、`CONFIG_PRINTK_EXECUTION_CTX`で
有効化されるビルドでは、`struct printk_info`に`caller_id2`と`comm[TASK_COMM_LEN]`フィールドも追加される（上には示していない）—
まだ一般的なケースではない。

```c
/*
 * リングバッファはロックフリーの multi-producer multi-consumer 設計である。
 * 読み手（dmesg、/dev/kmsg）が書き手（printk）をブロックすることはない。
 * バッファが満杯になると古いメッセージが上書きされる。
 */
```

## printkの流れ {#printk-flow}

```c
/* kernel/printk/printk.c — 簡略化したもの。実際の関数は、このコアシーケンスに
 * 到達する前に、パニック CPU のメッセージ抑制や、スケジューラコンテキスト
 * （LOGLEVEL_SCHED）のケースも処理する */
asmlinkage int vprintk_emit(int facility, int level,
                             const struct dev_printk_info *dev_info,
                             const char *fmt, va_list args)
{
    /* 1. メッセージを整形してリングバッファに格納する
     *    （vscnprintf + prb_reserve/memcpy/prb_commit が内部で行われる） */
    int printed_len = vprintk_store(facility, level, dev_info, fmt, args);

    /* 2. 今すぐコンソールへフラッシュするかどうか、どのようにするかを決める。
     *    実際の判断は struct console_flush_type によって駆動され、
     *    従来の console-lock パス、より新しい "nbcon"
     *   （non-blocking console）パス、そしてここで直接出力するのが
     *    安全でない場合の kthread や irq_work へのオフロードを扱う。 */

    /* 3. 直接フラッシュしている者がいなければ、コンソールの書き手
     *   （klogd、/dev/kmsg の読み手）を起床させ、メッセージを拾わせる */
    wake_up_klogd();

    return printed_len;
}
```

## カーネルログを読む {#reading-kernel-logs}

```bash
# dmesg: /dev/kmsg を読む
dmesg
dmesg -T      # 人間が読めるタイムスタンプ
dmesg -l err  # エラー以上のみ
dmesg -w      # 追従（tail -f のように）
dmesg --clear # リングバッファをクリア

# /dev/kmsg: 生の構造化アクセス
cat /dev/kmsg
# 6,1234,123456789,-;e1000: eth0: Intel(R) PRO/1000 Network Connection
# 形式: level,seq,timestamp(us),flags;message

# journalctl はカーネルメッセージを取り込んだ journal から読む:
journalctl -k              # 前回ブート以降のカーネルメッセージ
journalctl -k -b-1         # 1つ前のブートのカーネルメッセージ
journalctl -k -p err       # カーネルのエラー
```

## レート制限 {#rate-limiting}

```c
/* ログの氾濫を避ける: */
printk_ratelimited(KERN_WARNING "my_driver: spurious interrupt\n");
/* 5 秒あたり 1 メッセージを許可し、抑制された件数を出力する */

pr_warn_ratelimited("Too many requests, dropping\n");
dev_warn_ratelimited(&dev, "error %d, continuing\n", ret);

/* 制御するもの: */
/* /proc/sys/kernel/printk_ratelimit        （デフォルト 5 秒） */
/* /proc/sys/kernel/printk_ratelimit_burst  （デフォルト 10 メッセージ） */

/* カスタムのレート制限: */
static DEFINE_RATELIMIT_STATE(my_rs, 10 * HZ, 5);
if (__ratelimit(&my_rs))
    pr_warn("...");
```

## pr_fmt: モジュール接頭辞 {#pr_fmt-module-prefix}

```c
/* ソースファイルの先頭で: */
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

/* 以降のすべての pr_*() 呼び出しに自動的にモジュール名が含まれる: */
pr_info("driver loaded\n");
/* 出力: "my_module: driver loaded" */
```

## dynamic debug {#dynamic-debug}

```bash
# dev_dbg() と pr_debug() は動的に有効化しない限り何もしない:
# ファイル内のすべてのデバッグメッセージを有効化:
echo "file drivers/net/e1000.c +p" > /sys/kernel/debug/dynamic_debug/control

# モジュール内のすべてのデバッグメッセージを有効化:
echo "module e1000 +p" > /sys/kernel/debug/dynamic_debug/control

# フラグ付きで有効化（p=出力, f=関数名, l=行, m=モジュール, t=スレッド, s=ソースファイル名）:
echo "file net/ipv4/tcp.c +pflmts" > /sys/kernel/debug/dynamic_debug/control

# すべての dynamic debug サイトを一覧表示:
cat /sys/kernel/debug/dynamic_debug/control | grep "e1000"
# drivers/net/ethernet/intel/e1000e/netdev.c:2345 [e1000e]e1000_intr =p

# 起動時に有効化するブートパラメータ:
# dyndbg="module e1000 +p"
```

## pstore: 再起動をまたいでメッセージを保持する {#pstore-persist-messages-across-reboot}

```bash
# CONFIG_PSTORE が有効な場合（EFI 変数、RAM、MTD）:
ls /sys/fs/pstore/      # 前回ブートのクラッシュログ
cat /sys/fs/pstore/dmesg-ramoops-0

# ramoops: 永続的なカーネルログ専用の RAM 領域
# EFI pstore: UEFI の不揮発性ストレージに保存する
# 初期ブートのクラッシュやパニックの嵐をデバッグするのに有用
```

## printkを観測する {#observing-printk}

```bash
# すべての printk 呼び出しをトレースする:
bpftrace -e '
kprobe:vprintk_emit {
    printf("printk level=%d: %s\n", (int)arg1, str(arg3));
}'

# シリアルコンソールの速度ボトルネック:
# 大量のロギングはリアルタイムタスクを停滞させうる
# 解決策: 非同期コンソール（ASYNC_CONSOLE）か pstore を使う
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/printk/printk.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/printk/printk.c) — `vprintk_emit()`、`wake_up_klogd()`、`console_flush_all()`: コア実装
- [kernel/printk/printk_ringbuffer.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/printk/printk_ringbuffer.h) — `struct printk_info`、`struct printk_ringbuffer`、`prb_reserve()`/`prb_commit()`: ロックフリーのリングバッファ
- [include/linux/printk.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/printk.h) — `pr_*()`便利マクロと`printk_ratelimited()`
- [include/linux/ratelimit_types.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/ratelimit_types.h) — `DEFINE_RATELIMIT_STATE()`と`__ratelimit()`: レート制限のプリミティブ
- [include/linux/dev_printk.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/dev_printk.h) — `dev_err()`、`dev_info()`、`dev_warn()`、`dev_dbg()`: デバイスを意識したラッパー

### manページ {#man-pages}

- [`dmesg(1)`](https://man7.org/linux/man-pages/man1/dmesg.1.html) — カーネルログの読み取りとフィルタリング。`-T`、`-w`、`-l`、`--clear`を含む
- [`syslog(2)`](https://man7.org/linux/man-pages/man2/syslog.2.html) — `dmesg`/`klogctl`の背後にあるシステムコールと、`/proc/sys/kernel/printk`を通じて公開されるconsole-loglevelのセマンティクス

### 関連ページ {#related-pages}

- [Oops解析](../debugging/oops-analysis.md) — カーネルクラッシュ出力の読み方
- [kdump](../debugging/kdump.md) — カーネルクラッシュダンプ
- [Dynamic Debug](../modules/dynamic-debug.md) — `pr_debug()`/`dev_dbg()`と`dynamic_debug/control`コマンド言語の詳しい解説

### LWN記事 {#lwn-articles}

- [Reimplementing printk()](https://lwn.net/Articles/780556/) — 現在の`printk_ringbuffer`が実装している、John Ognessによるロックレスなリングバッファの再設計
- [Why printk() is so complicated (and how to fix it)](https://lwn.net/Articles/800946/) — 現在の設計を形作った、コンソール出力・ログレベル・信頼性の間の緊張関係
- [Keeping printk() under control](https://lwn.net/Articles/66091/) — 元祖の`printk_ratelimit()`機構

### 外部資料 {#external}

- [Message logging with printk](https://docs.kernel.org/core-api/printk-basics.html) — `pr_*()`マクロ、`KERN_*`ログレベル、`pr_fmt()`
- [dynamic-debug-howto](https://docs.kernel.org/admin-guide/dynamic-debug-howto.html) — `/sys/kernel/debug/dynamic_debug/control`の完全なコマンド言語とフラグのリファレンス
- [sysctl/kernel.rst: printk](https://docs.kernel.org/admin-guide/sysctl/kernel.html#printk) — `/proc/sys/kernel/printk`の4つの値と、`printk_ratelimit`および`printk_ratelimit_burst`のデフォルト値
- [ramoops.rst](https://docs.kernel.org/admin-guide/ramoops.html) — RAMをバックエンドとするpstore/ramoopsの永続ロギング機構
