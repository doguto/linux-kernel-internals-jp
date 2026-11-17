# カーネルブートパラメータ {#kernel-boot-parameters}

> コマンドラインのパース、`__setup`マクロ、`early_param`

## カーネルがコマンドラインを受け取る仕組み {#how-the-kernel-receives-its-command-line}

カーネルのコマンドラインはブートローダに由来する。たとえばGRUBは、コマンド文字列をブートプロトコルの`boot_params.hdr.cmd_line_ptr`フィールド（物理アドレス）に置く。アーキテクチャのセットアップコード（`setup_arch()`）は、ブートの早い段階でこれをカーネルのアドレス空間にコピーする。

`start_kernel()`が始まる時点で、コマンドラインは2つの静的配列にコピーされている:

```c
/* init/main.c */
char __initdata boot_command_line[COMMAND_LINE_SIZE];
char *saved_command_line __ro_after_init;
```

`boot_command_line`はブートローダからの生の文字列をそのまま保持する。`saved_command_line`には`setup_arch()`による追加分（たとえばUEFIスタブが付け加えたパラメータ）が含まれる場合がある。作業用の変更可能なコピーが、各パース段階によってパースされ消費される。

```bash
# ユーザー空間からコマンドラインを見る:
cat /proc/cmdline
# BOOT_IMAGE=/vmlinuz-6.8.0 root=/dev/sda1 ro quiet splash
```

---

## パース段階 {#parsing-stages}

カーネルはコマンドラインを3つの異なるフェーズでパースする。フェーズによって、どのパラメータがいつ見えるかが決まる。

### フェーズ1: `parse_early_param()` — 非常に早い段階 {#phase-1-parse_early_param-very-early}

メモリアロケータが初期化される前に`start_kernel()`から呼ばれる。このフェーズでは、ブートの残りを進める前に知っておく必要があるパラメータ — たとえば使用可能なRAMの量や、どのコンソールを使うか — を処理する。

`early_param()`で登録されたパラメータはここで処理される。

### フェーズ2: `parse_args()` — カーネル初期化の本体 {#phase-2-parse_args-main-kernel-init}

`setup_arch()`とper-CPU領域のセットアップの後に`start_kernel()`から呼ばれる。各トークンは順に、まず**組み込みモジュールパラメータ**（`module.param=value`形式）と照合され、次に — 一致しなかった場合に限り — `__setup()`ハンドラと照合される（具体的な仕組みは後述の[`parse_args()`の内部](#parse_args-internals)を参照）。

どちらにも一致しなかったものは、次のいずれかとして扱われる:

- initプロセスに渡す環境変数
- 未知のパラメータ（警告ではなく`pr_notice()`でログに記録される）

### フェーズ3: モジュールパラメータ — モジュールのロード時 {#phase-3-module-parameters-at-module-load-time}

ロード可能モジュールの場合、`insmod`/`modprobe`のコマンドラインで渡されたパラメータは、モジュールのロード時に`kernel/params.c`を通じてパースされる。組み込みモジュールの場合、これらはカーネルコマンドライン上に`module.param=value`として現れ、フェーズ2で処理される。

---

## `early_param()`マクロ {#early_param-macro}

```c
/* include/linux/init.h */
#define early_param(str, fn) \
    __setup_param(str, fn, fn, 1)
```

`1`は、これがearlyパラメータであることを示す。`parse_early_param()`は`parse_args()`自体の内部でフィルタリングを行うわけではない — 空のパラメータ配列と、`unknown`コールバックとしての`do_early_param()`を指定して`parse_args()`を呼び出す。そして`.init.setup`を走査して`early`フラグが立っていないエントリをすべてスキップするのは`do_early_param()`である（後述の[`parse_args()`の内部](#parse_args-internals)を参照）。

### 例: `mem=`ハンドラ {#example-mem-handler}

```c
/* arch/x86/kernel/e820.c */
static int __init parse_memopt(char *p)
{
    if (!p)
        return -EINVAL;
    /* ... p をバイト数にパースし、e820 マップをそこで切り詰める ... */
    return 0;
}
early_param("mem", parse_memopt);
```

`arch/x86/kernel/setup.c`は、`setup_arch()`の残りの大部分を抱えているにもかかわらず、独自の`__setup()`/`early_param()`ハンドラを1つも登録していない。`mem=`や後述の他のメモリマップオプションは、代わりに`arch/x86/kernel/e820.c`にある。`early_param()`ハンドラは、後で説明する`__setup()`の戻り値規約ではなく、*モジュールパラメータ*の戻り値規約に従う: 成功なら`0`を返し、カーネルに警告を出力させるには非ゼロ（通常は負のerrno）を返す — この値は、`__setup()`ハンドラの戻り値のように「処理した／処理していない」を意味するわけではない。

`early_param()`で処理される一般的なパラメータ:

| パラメータ | ハンドラの場所 | 目的 |
|-----------|-----------------|---------|
| `mem=N` | `arch/x86/kernel/e820.c` | 使用可能なRAMを制限する |
| `memmap=N@S` | `arch/x86/kernel/e820.c` | メモリ領域をマークする |
| `console=` | `kernel/printk/printk.c`（`early_param()`ではなく`__setup()` — 下の注記を参照） | 初期コンソールの設定 |
| `earlyprintk=` | `arch/x86/kernel/early_printk.c` | 非常に早い段階のコンソール（`console_init()`より前） |
| `loglevel=N` | `init/main.c` | 初期ログレベルを設定する |
| `ignore_loglevel` | `kernel/printk/printk.c` | すべてのログレベルをコンソールに出力する |

`console=`は`early_param()`ではなく`__setup("console=", console_setup)`で登録されている — フェーズ1ではなくフェーズ2でパースされる。

---

## `__setup()`マクロ {#__setup-macro}

```c
/* include/linux/init.h */
#define __setup(str, fn) \
    __setup_param(str, fn, fn, 0)

#define __setup_param(str, unique_id, fn, early)             \
    static const char __setup_str_##unique_id[] __initconst  \
        __aligned(1) = str;                                   \
    static struct obs_kernel_param __setup_##unique_id       \
        __used __section(".init.setup")                       \
        __aligned(__alignof__(struct obs_kernel_param))       \
    = { __setup_str_##unique_id, fn, early }
```

各`__setup()`呼び出しは、`struct obs_kernel_param`のエントリを`.init.setup`セクションに配置する。`early_param()`は同じマクロを`early=1`で使う — 両者の唯一の違いはこのフラグであり、`do_early_param()`（フェーズ1の`unknown`コールバック。後述）が`.init.setup`を走査する際にこれをチェックする:

```c
struct obs_kernel_param {
    const char *str;
    int (*setup_func)(char *);
    int early;
};
```

リンカは、すべての`.init.setup`エントリを`__setup_start`と`__setup_end`の間の連続した配列にまとめる。`parse_args()`自身がこの配列を走査することはない — 走査するのは`unknown`コールバック（フェーズ1では`do_early_param()`、フェーズ2では`obsolete_checksetup()`）であり、`str`が一致するエントリを見つけると対応する`setup_func`を呼び出す（全体像は後述の[`parse_args()`の内部](#parse_args-internals)を参照）。

### `__setup()`ハンドラを書く {#writing-a-__setup-handler}

```c
static int __init myopt_setup(char *str)
{
    if (!str)
        return 0;

    if (*str == '=')
        str++;

    if (kstrtoint(str, 0, &myopt_value))
        return 0;

    return 1;  /* 1 = パース成功、0 = 処理していない */
}
__setup("myopt=", myopt_setup);
```

ハンドラは、パラメータの値部分（`myopt=`より後ろのすべて）へのポインタを受け取る。パラメータを認識して処理した場合は`1`を、そうでない場合は`0`を返す（後者の場合、他のハンドラがそれを試せる）。セットアップ文字列に`=`を含めると値を必須にでき、省略すると値のないフラグになる。

### ブール値フラグの例 {#boolean-flag-example}

```c
static bool myfeature_enabled __initdata = false;

static int __init myfeature_setup(char *str)
{
    myfeature_enabled = true;
    return 1;
}
__setup("myfeature", myfeature_setup);
```

---

## `parse_args()`の内部 {#parse_args-internals}

`parse_args()`（`kernel/params.c`）は汎用的なもので、`.init.setup`についてはまったく関知しない。
空白区切りの各トークンを最初の`=`で分割し、`parse_one()`を呼び出す。`parse_one()`は次のことを行う:

1. **呼び出し元が渡した**`struct kernel_param`配列を線形に走査して名前の一致を探す。これが
   `parse_one()`自身が照合する唯一の対象である。
2. 何も一致しなければ（または呼び出し元が空の配列を渡していれば）、呼び出し元が与えた`unknown`コールバックを
   呼び出す — そのコールバックが次に何をするかは、どのフェーズから呼ばれているかに完全に依存する。

`.init.setup`リンカセクションの配列（`__setup_start`/`__setup_end`）を走査するのは、`parse_one()`自身ではなく、
2つの*異なる*`unknown`コールバックである:

- **フェーズ1**（`parse_early_param()` → `parse_early_options()`）は空の`params`配列を渡す — すべての
  トークンはそのままコールバック`do_early_param()`に落ちる。`do_early_param()`は`.init.setup`を
  `early`フラグ付きのエントリに絞って走査し、一致すれば`setup_func()`を直接呼び出す。
- **フェーズ2**のトップレベルの`parse_args()`呼び出し（`start_kernel()`から）は`__start___param`..`__stop___param`を渡す
  — これは**組み込みモジュールパラメータ**の配列である（`modname.paramname`形式の名前がコンパイル時に
  埋め込まれて登録されている。例: `net_core.netdev_budget` — `.`を検出する特別なロジックはなく、`parameq()`は
  単純な文字列比較である）。この配列で一致しなかった場合に限り、そのコールバック`unknown_bootoption()`が
  `obsolete_checksetup()`を実行する — これはまったく同じ`.init.setup`配列に対する*2回目*の別個の走査であり、今度は
  early以外の`__setup()`エントリが対象となる:

```c
/* init/main.c（簡略化） */
static bool __init obsolete_checksetup(char *line)
{
    const struct obs_kernel_param *p = __setup_start;

    do {
        int n = strlen(p->str);
        if (parameqn(line, p->str, n)) {
            if (p->early) {
                /* parse_early_param() で処理済み。記録するだけ。 */
            } else if (p->setup_func(line + n))
                return true;
        }
        p++;
    } while (p < __setup_end);

    return false;
}
```

つまり、フェーズ2のトークンはまず組み込みモジュールパラメータと、次に`.init.setup`と照合され、両方の走査で
一致しなかった場合にのみ、真に未知の／init環境のトークンとなる。

```bash
# 認識されないパラメータに対するカーネルログ:
# Unknown kernel command line parameters "mxcpus=4", will be passed to userspace
```

---

## 組み込みモジュールのモジュールパラメータ {#module-parameters-for-built-in-modules}

カーネルに組み込まれたモジュール（Kconfigで`m`ではなく`y`にしたもの）もパラメータを公開するが、それらは`insmod`のコマンドラインではなくカーネルコマンドラインからパースされる:

```c
/* kernel/rcu/tree.c */
static long qhimark = DEFAULT_RCU_QHIMARK;
module_param(qhimark, long, 0444);
```

これをカーネルコマンドラインから設定するには:

```
rcutree.qhimark=20000
```

モジュール名の接頭辞（`rcutree`）は、ソースファイルのコンパイル時のモジュール名から導出され、コンパイル時に登録パラメータ名へ直接埋め込まれる — 実行時に「これはモジュールパラメータらしいか」を検出する処理は存在しない。`parse_one()`は、組み込みモジュールパラメータ配列にあるものに対して、ドット付き接頭辞も含めてリテラルな文字列一致を行うだけである（前述の[`parse_args()`の内部](#parse_args-internals)を参照）。

（まさにこのような見た目のパラメータ — 読者が`net_core.netdev_budget=`で設定できる単純な`module_param()` — が、かつてネットワークポーリング用に存在した: `net/core/dev.c`の`netdev_budget`である。これはその後**sysctl**の`net.core.netdev_budget`に移行し、コマンドラインで設定可能な`module_param()`としても、`/sys/module/`の下にも、もはや一切現れない。）

### sysfsからのアクセス {#accessing-from-sysfs}

ロード済みモジュール（または`0644`パーミッションの組み込みモジュール）のモジュールパラメータは、`/sys/module/`の下に公開される:

```bash
cat /sys/module/rcutree/parameters/qhimark
10000
```

`0444`と指定されたパラメータ（上の`qhimark`など）はブート後は読み取り専用である — ファイルは読めるが書き込めない。書き込み可能なものは`0644`を使う。例: `echo 600 > /sys/module/<name>/parameters/<param>`。`0000`と指定されたパラメータはsysfsにまったく現れない。

---

## 重要なカーネルパラメータ {#important-kernel-parameters}

以下は、ハンドラをソースレベルで理解しておく価値のあるパラメータである:

| パラメータ | ハンドラ | ソースファイル |
|-----------|---------|------------|
| `root=/dev/sda1` | `root_dev_setup()` | `init/do_mounts.c` |
| `init=/bin/sh` | `init_setup()` | `init/main.c` |
| `ro` / `rw` | `readonly()` / `readwrite()` | `init/do_mounts.c` |
| `quiet` | `quiet_kernel()`（early_param） | `init/main.c` |
| `debug` | `debug_kernel()`（early_param） | `init/main.c` |
| `panic=N` | `core_param()`経由の`panic_timeout` | `kernel/panic.c` |
| `maxcpus=N` | `maxcpus()`（early_param） | `kernel/smp.c` |
| `nosmp` | `nosmp()`（early_param） | `kernel/smp.c` |
| `nokaslr` | `cmdline_find_option_bool("nokaslr")` — `.init.setup`が存在すらしない段階で、ブートデコンプレッサ内で直接チェックされる。これに対する`__setup()`/`early_param()`ハンドラは一切ない | `arch/x86/boot/compressed/kaslr.c` |
| `nopti` | `pti_parse_cmdline_nopti()`（early_param） | `arch/x86/mm/pti.c` |
| `crashkernel=N` | 登録された`early_param`のエントリポイントは`parse_crashkernel_dummy()`であり、`parse_crashkernel()`に処理を委ねる | `kernel/crash_reserve.c` |
| `initcall_debug` | `initcall_debug_enable()` | `init/main.c` |
| `printk.devkmsg=on/off/ratelimit` | — | `kernel/printk/printk.c` |

`core_param()`（`include/linux/moduleparam.h`）は、`__setup()`/`early_param()`と並ぶ第3のパース機構ではない — これは`module_param()`自身の登録プリミティブである`__module_param_call()`を、ファイルの実際のモジュール名の代わりに空のモジュール名接頭辞（`""`）で呼び出したものである。だからこそ`panic=10`は、`kernel.panic=10`のようなものを必要とせず、接頭辞なしでコマンドライン上で機能する: 通常の`module_param()`呼び出しが格納するのとまったく同じ組み込みモジュールパラメータ配列（`__start___param`..`__stop___param`）に、ドット付き接頭辞なしで入るだけである。

### initバイナリの選択 {#selecting-an-init-binary}

```
init=/bin/sh          # drop to a root shell (recovery)
init=/sbin/init       # explicit sysvinit
init=/lib/systemd/systemd  # explicit systemd (usually the default)
rdinit=/bin/sh        # shell in initramfs (before pivot_root)
```

### CPUの制限 {#limiting-cpus}

```
maxcpus=1    # boot as if uniprocessor (secondary CPUs parked)
nosmp        # equivalent; disables SMP entirely
```

### KASLRの制御 {#kaslr-control}

```
nokaslr      # disable kernel address space layout randomization
             # useful for debugging: makes addresses reproducible across boots
```

---

## カーネルコードからパラメータを調べる {#inspecting-parameters-from-kernel-code}

```c
#include <linux/init.h>

/* 保存されたコマンドラインにアクセスする */
extern char *saved_command_line;

/* パラメータがコマンドラインのどこかに現れるかを調べる */
static bool __init has_param(const char *name)
{
    return strstr(saved_command_line, name) != NULL;
}

/* コマンドラインから数値をパースする */
static unsigned long __init get_param_ulong(const char *name,
                                             unsigned long defval)
{
    char *p = strstr(saved_command_line, name);
    if (!p)
        return defval;
    p += strlen(name);
    if (*p != '=')
        return defval;
    return simple_strtoul(p + 1, NULL, 0);
}
```

本番コードでは、手作業の文字列検索よりも`early_param()`や`__setup()`を使うべきである — これらは正しいライフタイム管理、適切なエラー報告、そしてブート完了時の`.init.setup`セクションのクリーンアップを提供する。

---

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/init.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/init.h) — `early_param()`、`__setup()`、`__setup_param()`マクロと`struct obs_kernel_param`
- [kernel/params.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/params.c) — `parse_args()`と`parse_one()`: トークン分割ループとトークンごとのディスパッチ
- [init/main.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/init/main.c) — `boot_command_line`、`saved_command_line`、`parse_early_param()`/`do_early_param()`、`unknown_bootoption()`/`obsolete_checksetup()`（`__setup_start`/`__setup_end`リンカセクションの走査）
- [init/do_mounts.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/init/do_mounts.c) — `root_dev_setup()`、`readonly()`/`readwrite()`: `root=`/`ro`/`rw`に対する実際の`__setup()`ハンドラの実例
- [include/linux/moduleparam.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/moduleparam.h) — 組み込みモジュールとロード可能モジュールが使う`module_param()`マクロ群、および同じ`__module_param_call()`プリミティブを空の接頭辞で使う`core_param()`
- [arch/x86/kernel/e820.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/e820.c) — `parse_memopt()`/`parse_memmap_opt()`: 実際の`mem=`/`memmap=`ハンドラ
- [kernel/panic.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/panic.c) — `core_param(panic, panic_timeout, ...)`
- [arch/x86/boot/compressed/kaslr.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/boot/compressed/kaslr.c) — `.init.setup`が存在する前にブートデコンプレッサ内でチェックされる`cmdline_find_option_bool("nokaslr")`
- [arch/x86/mm/pti.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/pti.c) — `pti_parse_cmdline_nopti()`
- [kernel/crash_reserve.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/crash_reserve.c) — `parse_crashkernel_dummy()`（登録された`early_param`のエントリポイント）と`parse_crashkernel()`
- [kernel/rcu/tree.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/rcu/tree.c) — `qhimark`/`module_param(qhimark, ...)`: このページで取り上げた組み込みモジュールパラメータの実例

### manページ {#man-pages}

- [`proc_cmdline(5)`](https://man7.org/linux/man-pages/man5/proc_cmdline.5.html) — `/proc/cmdline`を説明している

### 関連ページ {#related-pages}

- [初期ブートとstart_kernel()](early-boot.md) — 各パースフェーズがいつ実行されるか
- [モジュール初期化とinitcall](initcalls-modules.md) — ロード可能モジュールにおける`module_param()`

### 外部資料 {#external}

- [The kernel's command-line parameters](https://docs.kernel.org/admin-guide/kernel-parameters.html) — `__setup()`、`early_param()`、`core_param()`、`module_param()`が認識するすべてのパラメータの、自動生成された正式な一覧
