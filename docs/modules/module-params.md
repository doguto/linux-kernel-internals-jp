# モジュールパラメータ、シンボル、Kconfig {#module-parameters-symbols-and-kconfig}

> ロード時にモジュールを設定し、カーネル全体でシンボルを共有する

## module_param: 実行時の設定 {#module_param-runtime-configuration}

```c
/* モジュールパラメータを宣言する */
static int debug = 0;
static unsigned int timeout_ms = 100;
static char *device_name = "mydevice";
static bool enable_feature = false;

module_param(debug, int, 0644);
MODULE_PARM_DESC(debug, "Debug level (0=off, 1=basic, 2=verbose)");

module_param(timeout_ms, uint, 0644);
MODULE_PARM_DESC(timeout_ms, "Timeout in milliseconds (default: 100)");

module_param(device_name, charp, 0444);  /* ロード後は読み取り専用 */
MODULE_PARM_DESC(device_name, "Device name string");

module_param(enable_feature, bool, 0644);
MODULE_PARM_DESC(enable_feature, "Enable experimental feature");
```

3番目の引数は `sysfs` のパーミッションである。

- `0644` — 所有者は読み書き可能、グループ/その他は読み取り専用 → 参照も変更もできる
- `0444` — 全員に対して読み取り専用
- `0` — sysfsに公開されない（ロード時のみ）

```bash
# ロード時に設定する
sudo modprobe mymodule debug=2 timeout_ms=500

# あるいはinsmodで
sudo insmod mymodule.ko debug=2

# sysfs経由で読み書きする（パーミッションが許す場合）
cat /sys/module/mymodule/parameters/debug
echo 3 > /sys/module/mymodule/parameters/debug
```

### 配列 {#arrays}

```c
static int irq_nums[4];
static unsigned int irq_count;

module_param_array(irq_nums, int, &irq_count, 0444);
MODULE_PARM_DESC(irq_nums, "IRQ numbers (comma-separated)");
```

カウント変数は `int` ではなく `unsigned int` でなければならない。`module_param_array()`
はこのポインタを `struct kparam_array` に格納し、その `.num` フィールドは
`unsigned int *` と宣言されているため、`int *` を渡すと互換性のないポインタ型のエラーとなり、
カーネルのビルドで拒否される。カウントが不要なら代わりに `NULL` を渡す。

```bash
sudo insmod mymodule.ko irq_nums=5,6,7  # カーネルがirq_countを3に設定する
```

### パラメータ変更時のコールバック {#callbacks-on-parameter-change}

```c
static int my_debug = 0;

static int debug_set(const char *val, const struct kernel_param *kp)
{
    int n;
    int ret = kstrtoint(val, 10, &n);
    if (ret)
        return ret;
    if (n < 0 || n > 3)
        return -EINVAL;
    my_debug = n;
    pr_info("debug level changed to %d\n", n);
    return 0;
}

static const struct kernel_param_ops debug_ops = {
    .set = debug_set,
    .get = param_get_int,
};
/* 別の名前にしている点に注意: "debug" は上の
 * module_param(debug, ...) ですでに使われており、1つのモジュールで
 * 同じ名前のパラメータを2つ登録することはできない。
 */
module_param_cb(debug_level, &debug_ops, &my_debug, 0644);
```

## EXPORT_SYMBOL: モジュール間でのシンボル共有 {#export_symbol-sharing-symbols-between-modules}

```c
/* モジュールA（またはコアカーネル）内: */
int shared_function(int arg)
{
    return arg * 2;
}
EXPORT_SYMBOL(shared_function);             /* どのモジュールからも利用可能 */

int shared_function_gpl(int arg)
{
    return arg * 3;
}
EXPORT_SYMBOL_GPL(shared_function_gpl);     /* GPLモジュールのみ */

int namespaced_function(int arg)
{
    return arg + 1;
}
/* 名前空間付きエクスポート（5.4以降）。名前空間は文字列リテラル。 */
EXPORT_SYMBOL_NS(namespaced_function, "MYNS");
```

各シンボルはちょうど1回だけエクスポートされる — `EXPORT_SYMBOL`、`EXPORT_SYMBOL_GPL`、
`EXPORT_SYMBOL_NS` は同じシンボルに対する選択肢であり、重ねて
適用する層ではない。

```c
/* モジュールB内: */
extern int shared_function(int arg);  /* あるいはヘッダをインクルードするだけ */

static int __init moduleB_init(void)
{
    int result = shared_function(21);  /* 動作する: シンボルがエクスポートされている */
    pr_info("result = %d\n", result);
    return 0;
}
```

### EXPORT_SYMBOLとEXPORT_SYMBOL_GPL {#exportsymbol-vs-exportsymbol_gpl}

`EXPORT_SYMBOL_GPL` は、`MODULE_LICENSE("GPL")` または互換ライセンスを持つモジュールだけがそのシンボルを使えるようにする。これにより、プロプライエタリなモジュールがカーネル内部のインターフェースを使えないことが強制される。

モジュールがGPL専用シンボルをインポートする場合:
```bash
modinfo mymodule.ko | grep license
# license: GPL
```

モジュールがGPL非互換のライセンスを宣言しているのにGPL専用シンボルを
参照している場合、違反は2回検出される。まず**ビルド**時に
`modpost` によって:

```
ERROR: modpost: GPL-incompatible module mymodule.ko uses GPL-only symbol 'shared_function_gpl'
```

そして、そのようなモジュールがそれでもロードされた場合（たとえばビルド済みの
`.ko`）、**ロード**時にもう一度検出される。カーネルには「GPLではない」ことを示す専用のエラーはない。
`find_exported_symbol_in_section()` は、ロードするモジュールのライセンスがGPL互換でない場合に
GPL専用シンボルを単に読み飛ばすので、そのシンボルはまったく存在しないかのように
見える:

```
$ sudo insmod mymodule.ko
insmod: ERROR: could not insert module mymodule.ko: Unknown symbol in module

$ dmesg | tail -1
mymodule: Unknown symbol shared_function_gpl (err -2)
```

### シンボルのバージョニング（CRC） {#symbol-versioning-crc}

カーネルが `CONFIG_MODVERSIONS=y` でビルドされている場合、エクスポートされた各シンボルには
その型シグネチャのCRCも付与される。シンボルの定義が変わるとCRCも
変わり、古いインターフェースに対してコンパイルされたモジュールはロードを拒否される:

```
# ロード時、kernel/module/version.cのcheck_version()から:
mymodule: disagrees about version of symbol tcp_sendmsg
```

ビルド時の警告は `modpost` から出る。`modpost` はモジュールが参照するすべての
シンボルについてCRCを記録する:

```
WARNING: modpost: "tcp_sendmsg" [drivers/net/mydriver.ko] has no CRC!
```

この警告は、参照されたシンボルに記録すべき有効なCRCがなかったことを意味する。通常は
エクスポート側がバージョン情報なしでビルドされたことが原因である。この警告には
`CONFIG_MODVERSIONS=y` が必要だが、2つの異なる `modpost` の関数がまったく同じ
文字列を出力する。`add_versions()` は `CONFIG_BASIC_MODVERSIONS` が設定されていなければ
直ちに戻り（これは `MODVERSIONS` の下ではデフォルトで `y`）、`add_extended_versions()`
は代わりに `CONFIG_EXTENDED_MODVERSIONS` で保護されている — そのため、ビルドがどちらのバージョン形式を
出力するかに応じて、警告はどちらの経路からも出うる。
`CONFIG_MODVERSIONS=n` ではCRCが存在しないので、CRCの警告もロード時の
バージョンチェックも一切ない。

## /proc/kallsyms: すべてのカーネルシンボル {#prockallsyms-all-kernel-symbols}

```bash
# 関数の場所を探す（rootで — 後述のアドレスに関する注記を参照）
sudo grep -E "tcp_sendmsg|mymodule_reset" /proc/kallsyms
# ffffffff81a12345 T tcp_sendmsg              ← T = グローバルなコード。「エクスポート済み」ではない
# ffffffffc0401000 t mymodule_reset [mymodule] ← t = モジュールローカルなコード、[モジュール名]

# アドレス → 関数名
sudo awk '/ffffffff81a12345/{print $3}' /proc/kallsyms

# モジュールのすべてのシンボル
sudo grep "\[mymodule\]" /proc/kallsyms
```

アドレスは非特権の読み手からは隠される。`/proc/kallsyms` はファイルを開いた者の
資格情報に対して `kallsyms_show_value()` を呼び、それがfalseを返すと
`s_show()` はNULLアドレスを出力する。`CAP_SYSLOG` がない場合
（そして `kptr_restrict` の値によっては）、すべての行が上に示した実際のアドレスではなく
`0000000000000000` となる — これが `sudo` を付けている理由である。

シンボルの種類:

- `T`/`t` — コード（.text） — 大文字=グローバル、小文字=ローカル
- `D`/`d` — データ（.data）
- `R`/`r` — 読み取り専用データ（.rodata）
- `B`/`b` — BSS（ゼロ初期化）
- `W`/`w` — weak、`V`/`v` — weakなデータオブジェクト
- `A`/`a` — 絶対値。モジュールのシンボルの場合はper-CPU変数も含む
- `G`/`g` と `S`/`s` — small dataとsmall BSS。small-dataセクションを持つ
  アーキテクチャで使われる

大文字・小文字の規則は、ファイルの前半と後半でわずかに異なる。
vmlinuxのシンボルでは、文字はビルド時に埋め込まれた `nm` の出力をそのまま使っており、
大文字はグローバル（外部）リンケージを意味する — `T` はそのシンボルが
`EXPORT_SYMBOL` に渡されたかどうかについては何も示さない。モジュールのシンボルでは、
`s_show()` 自身が文字の大文字・小文字を付け直す: エクスポートされていれば大文字、
そうでなければ小文字である。

`nm` の `U`（未定義）は、モジュールのロードが完了すると表示されない。未定義シンボルは
モジュールのシンボルテーブルが切り詰められるときに除外される — `is_core_symbol()`
は `st_shndx == SHN_UNDEF` のものに対してfalseを返す — そのため、インポートされたシンボルは
独自の種類で表示されるのではなく、一覧から単に姿を消す。

`__init` 関数がないのも同じ理由である。`is_core_symbol()` はinit系セクションに
あるものもすべて拒否し、モジュールのinit関数が実行されると、
`do_init_module()` が `mod->kallsyms` を切り詰められた `core_kallsyms` のコピーに
切り替える。そのため、ロードが完了したモジュールで `mymodule_init` をgrepしても
何も見つからない — モジュールのコアメモリに残ったシンボルだけが
一覧に表示される。

## Kconfig: コンパイル時の設定 {#kconfig-compile-time-configuration}

```kconfig
# drivers/mydriver/Kconfig

config MY_DRIVER
    tristate "My example driver"
    depends on PCI
    select DMA_ENGINE
    help
      Enable support for the MyDriver hardware.

      If unsure, say N.
      To compile as a module, say M.

config MY_DRIVER_DEBUG
    bool "Enable MyDriver debug output"
    depends on MY_DRIVER
    default n
    help
      Enable verbose debug logging for MyDriver.
```

`tristate` は `y`（組み込み）、`m`（モジュール）、`n`（無効）のいずれかを意味する。

```makefile
# drivers/mydriver/Makefile
obj-$(CONFIG_MY_DRIVER) += mydriver.o
mydriver-y := mydriver_main.o mydriver_pci.o

# 条件付きコンパイル: 複合モジュールの*中に*オブジェクトを追加する
mydriver-$(CONFIG_MY_DRIVER_DEBUG) += mydriver_debug.o
```

条件付きの行の接頭辞が `obj-` ではなく `mydriver-` である点に注意。`obj-y +=
mydriver_debug.o` とすると、そのオブジェクトは `built-in.a` に直接リンクされる一方、
`mydriver-y` リストからビルドされる `mydriver.ko` は依然として
`debug_dump()` を参照し、それを解決できずに失敗する。`MY_DRIVER_DEBUG` は `bool` なので
`y` か `n` にしかならない。`CONFIG_MY_DRIVER=m` の場合、2つの部分は
モジュール境界の反対側に分かれてしまう。`<module>-$(CONFIG_FOO) += file.o` は
複合モジュールにオブジェクトを任意で取り込むための定石であり、`=y` のビルドでも
正しく動作する。

```c
/* ソース内: */
#ifdef CONFIG_MY_DRIVER_DEBUG
void debug_dump(void) { /* ... */ }
#else
static inline void debug_dump(void) {}
#endif
```

```bash
# 設定する
make menuconfig
# 自分のドライバのセクションに移動する

# 何が有効になっているか確認する
grep MY_DRIVER .config
# CONFIG_MY_DRIVER=m
# # CONFIG_MY_DRIVER_DEBUG is not set

# このディレクトリだけをビルドする（ツリー内の単一ターゲット）
make drivers/mydriver/

# あるいは特定のオブジェクト/モジュール1つ
make drivers/mydriver/mydriver.ko
```

`M=` はツリー内向けの同等手段では*ない*。これは `make -C $KDIR M=$PWD` のように、**外部**
モジュールのディレクトリを指定するものである。カーネルツリー内のコードには、上の
単一ターゲットの形式を使う — 末尾に `/` を付けると、そのディレクトリ以下のすべてが
ビルドされる。

## モジュールの依存関係 {#module-dependencies}

`modprobe` は依存関係情報を読み、必要なモジュールを自動的にロードする。

```bash
# 依存関係データベースを再生成する（新しいモジュールをインストールした後）
depmod -a

# mymoduleが何を必要とするか確認する
modinfo mymodule.ko | grep depends
# depends: ptp,i2c-algo-bit

# 依存関係ツリー全体を見る
modprobe --show-depends e1000e
# insmod /lib/modules/.../ptp.ko
# insmod /lib/modules/.../e1000e.ko

# モジュールをブラックリストに入れる: エイリアスによる*今後の*自動ロードを止める
# （明示的な `modprobe nouveau` には影響せず、すでにロードされている
#  ものをアンロードすることもない）
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf

# アンロードは別個の、即時の操作である
modprobe -r nouveau

# ブラックリストがブート時の自動ロードに効くのは、initramfsを
# 再生成した後である（ディストリビューション依存）。例:
# sudo dracut -f      /  sudo update-initramfs -u
```

```bash
# /lib/modules/$(uname -r)/ ディレクトリ
ls /lib/modules/$(uname -r)/
# build           kernel          modules.alias      modules.builtin
# modules.dep     modules.order   modules.softdep    modules.symbols

# modules.dep: depmodが生成する
cat /lib/modules/$(uname -r)/modules.dep | grep e1000e
# kernel/drivers/net/ethernet/intel/e1000e/e1000e.ko: kernel/drivers/ptp/ptp.ko
```

## モジュールロードのフック {#module-loading-hooks}

udev/systemd-udevdは、デバイスの検出に基づいてモジュールを自動的にロードする。

```c
/* ハードウェアIDに一致させるため、ソースにMODULE_ALIASを追加する: */
MODULE_ALIAS("pci:v00008086d00001234*");  /* Intelのデバイス0x1234 */
```

```bash
# depmodがそれをmodules.aliasに収集する:
grep 00001234 /lib/modules/$(uname -r)/modules.alias
# alias pci:v00008086d00001234* e1000e
```

ドライバコアは、デバイスが検出されたときに `modprobe` を呼び出すわけではない。（他の
サブシステムは `request_module()` と `CONFIG_MODPROBE_PATH` を介してそれを起動する — しかし
この経路ではない。）デバイスが現れると、カーネルはデバイスのIDから組み立てた
`MODALIAS=` 変数を含むueventを発行する:

```bash
cat /sys/bus/pci/devices/0000:00:1f.6/modalias
# pci:v00008086d00001234sv...sd...bc02sc00i00
```

`systemd-udevd` はそのueventを受け取り、自身のルールと照合する。
標準のルールは `80-drivers.rules` にある:

```
ACTION!="add", GOTO="drivers_end"
ENV{MODALIAS}=="?*", RUN{builtin}+="kmod load"
```

`RUN{builtin}+=` は、そのイベントに対してビルトインを実行するようキューに入れる（
デバイスのプロパティをインポートするためにビルトインを実行する `IMPORT{builtin}=` とは異なる）。
`load` の後に引数がない点に注意: モジュール名が与えられない場合、`kmod` ビルトインは
イベントからデバイス自身の `MODALIAS` プロパティを読み取る。

ビルトインは次に、そのエイリアスを `modules.alias` を通じて解決し — `modprobe` が行うのと
同じlibkmodによる検索である — `e1000e` をロードする。その過程で
`modprobe.d` のブラックリストを適用するので、モジュールをブラックリストに入れると
この自動ロード経路は抑止される一方、明示的な `modprobe nouveau` は
動作し続ける。

## シンボル名前空間（5.4以降） {#symbol-namespaces-54}

大規模なサブシステムでは、シンボル名前空間によって内部シンボルの誤用を防ぐ。

```c
/* 名前空間付きでエクスポートする */
EXPORT_SYMBOL_NS(my_internal_func, "MY_SUBSYSTEM");

/* それを使うモジュールは名前空間をインポートしなければならない */
MODULE_IMPORT_NS("MY_SUBSYSTEM");

extern int my_internal_func(void);
```

名前空間の引数は**文字列リテラル**である。5.4でこの機能が導入された当初は裸のトークンだったが、
コミット `cdd30ebb1b9f`（v6.13）がマクロとツリー内のすべての呼び出し元を
引用符付き文字列に変換した。現在のカーネルでコンパイルできるのは引用符付きの形式だけである。

`MODULE_IMPORT_NS` がないと、まず `modpost` でビルドが失敗する:

```
ERROR: modpost: module mymodule uses symbol my_internal_func from namespace MY_SUBSYSTEM, but does not import it.
```

そして `verify_namespace_is_imported()` が、ロード時に同じモジュールを拒否する:

```
mymodule: module uses symbol (my_internal_func) from namespace MY_SUBSYSTEM, but does not import it.
```

`CONFIG_MODULE_ALLOW_MISSING_NAMESPACE_IMPORTS` は両方を警告に格下げする —
カーネルは `pr_err()` の代わりに `pr_warn()` を使うようになり、`modpost` には `-N` が
渡される。`make nsdeps` を使えば、不足している `MODULE_IMPORT_NS()` の行を
追加してくれる。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/moduleparam.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/moduleparam.h) — 上で使ったすべてのマクロ。`module_param()` は `module_param_named()` の薄いラッパーで、これは `param_check_<type>()` + `module_param_cb()` + `__MODULE_PARM_TYPE()` に展開される。`MODULE_PARM_DESC()` は単なる `MODULE_INFO(parm, ...)` である。ほかに `struct kernel_param_ops`（`.flags`、`.set`、`.get`、`.free`）、`struct kparam_array`（その `.num` フィールドは `unsigned int *`）、各 `struct kernel_param` を `__param` セクションに配置する `__module_param_call()` もある
- [kernel/params.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/params.c) — `int`/`uint`/`charp`/`bool` の型名の背後にある型ごとの `kernel_param_ops`（数値型は `STANDARD_PARAM_DEF()` マクロで生成され、`param_ops_charp` と `param_ops_bool` は手書きされている）に加えて、`param_array_ops`（`5,6,7` を解析し、要素数を `nump` 経由で書き戻すコード）、および `/sys/module/<name>/parameters/` を構築する `add_sysfs_param()`/`module_param_sysfs_setup()`。`perm == 0` を読み飛ばす処理は呼び出し側にある点に注意 — `module_param_sysfs_setup()` と `param_sysfs_builtin()` はどちらもそのようなパラメータを `continue` で飛ばす — 一方で `add_sysfs_param()` 自体は冒頭で `BUG_ON(!kp->perm)` を行い、`perm` に書き込みビットが立っている場合にのみ `store` ハンドラを設定する
- [include/linux/sysfs.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/sysfs.h) — `VERIFY_OCTAL_PERMISSIONS()`。`__module_param_call()` がパーミッション引数に適用するコンパイル時チェックで、誰でも書き込み可能なモードや、グループまたはその他が所有者より緩いモードを拒否する。そのため `0666` はビルドできない
- [include/linux/export.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/export.h) — `EXPORT_SYMBOL()`、`EXPORT_SYMBOL_GPL()`、`EXPORT_SYMBOL_NS()`/`EXPORT_SYMBOL_NS_GPL()`、`DEFAULT_SYMBOL_NAMESPACE`、そしてより新しい `EXPORT_SYMBOL_FOR_MODULES()`。ライセンスと名前空間は文字列として `.export_symbol` セクションに出力される。これが名前空間の引数が引用符付きの文字列リテラルである理由である
- [kernel/module/main.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/main.c) — `resolve_symbol()` と `find_exported_symbol_in_section()`。ここでGPL専用シンボルは、ライセンスがGPL互換でないモジュールからは単に*見えなく*される（その後ロードは `Unknown symbol` で失敗する）。さらに、`MODULE_IMPORT_NS` 不足エラーの発生源である `verify_namespace_is_imported()` もある
- [kernel/module/version.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/version.c) — `check_version()`: `CONFIG_MODVERSIONS` の背後にあるロード時のCRC比較であり、`disagrees about version of symbol` や `no symbol version for` のメッセージの出どころ
- [scripts/mod/modpost.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/mod/modpost.c) — 同じチェックのビルド時側: `add_versions()` が `has no CRC!` の警告を出し、GPLと名前空間の違反は `GPL-incompatible module ... uses GPL-only symbol` および `module ... uses symbol ... from namespace ..., but does not import it` として報告される
- [kernel/module/kallsyms.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/kallsyms.c) — `/proc/kallsyms` に表示される1文字のシンボル種別（`t`/`d`/`r`/`b`、加えてweakの `w`/`v` と絶対値の `a`）を割り当てる `elf_type()` と、モジュールのどのシンボルをそもそも一覧に載せるかを決める `is_core_symbol()`
- [kernel/kallsyms.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/kallsyms.c) — `/proc/kallsyms` のseq_file: `kallsyms_open()` が開いた者の資格情報を記録し、それが通らなければ `s_show()` がアドレスを空にする。判定関数そのものである `kallsyms_show_value()` — 非特権の `cat` ですべてのアドレスが `0000000000000000` に見える理由である `kptr_restrict`/`CAP_SYSLOG` のロジック — は隣の `kernel/ksyms_common.c` で定義されている

### manページ {#man-pages}

- [`modprobe(8)`](https://man7.org/linux/man-pages/man8/modprobe.8.html) — モジュール名の後に `param=value` を渡す方法、削除用の `-r`、`--show-depends`
- [`insmod(8)`](https://man7.org/linux/man-pages/man8/insmod.8.html) — 上の `insmod mymodule.ko debug=2` の例で使った、依存関係を考慮しないローダ
- [`modinfo(8)`](https://man7.org/linux/man-pages/man8/modinfo.8.html) — `MODULE_PARM_DESC()` が生成する `parm:` の行、`depends`、`license`、`alias` のフィールド、そしてそのうち1つを抽出する `-F`
- [`depmod(8)`](https://man7.org/linux/man-pages/man8/depmod.8.html) — `modules.dep`/`modules.dep.bin` と、`modules.alias` を含むマップファイルを生成する
- [`modules.dep(5)`](https://man7.org/linux/man-pages/man5/modules.dep.5.html) — 上に示した `modules.dep` の行の正確な形式
- [`modprobe.d(5)`](https://man7.org/linux/man-pages/man5/modprobe.d.5.html) — `blacklist` の背後にある設定ファイルと、モジュールパラメータのデフォルトを永続的に設定する `options` ディレクティブ

### 関連ページ {#related-pages}

- [カーネルモジュールの作成とロード](module-basics.md) — `module_init()`/`module_exit()`、`MODULE_LICENSE()`、そしてこれらのパラメータが関わるモジュールのライフサイクル
- [モジュールロードの内部](module-loading-internals.md) — シンボル解決、再配置、modversionsのチェックが `load_module()` 内のどこで行われるか
- [Kbuildビルドシステム](kbuild.md) — 上のMakefile断片の背後にある `obj-y`/`obj-m`/`<module>-y` の仕組みとKconfigのフロントエンド
- [カーネルモジュール署名](module-signing.md) — `.ko` が通過しなければならないもう1つのロード時の関門
- [プラットフォームドライバ](../drivers/platform-driver.md) — ドライバとしてのモジュールと、実際のドライバのマッチテーブルの文脈での `MODULE_DEVICE_TABLE()`
- [Linuxデバイスモデル](../drivers/device-model.md) — `/sys/module/<name>/parameters/` のようなディレクトリの土台となるkobject/sysfsの仕組みの背景

### LWNの記事 {#lwn-articles}

- [module_param() 1/3](https://lwn.net/Articles/16550/) — Rusty Russell、2002年11月26日: `MODULE_PARM` を置き換えるために `module_param()` を導入したパッチの投稿。パーミッション引数に関する当初の説明（"for exposing parameters in sysfs (if non-zero)"）を含む
- [Module parameters in sysfs](https://lwn.net/Articles/85443/) — Jonathan Corbet、2004年5月18日: `/sys/module/<name>/parameters/` を作ったパッチ。パラメータは `perm` が非0の場合にのみ現れ、`perm` がファイルのモードになることを確立した。このページのコールバックの節が扱う注意点も指摘している: パラメータが書き込まれたときの組み込みの通知はない
- [Kernel symbol namespacing](https://lwn.net/Articles/760045/) — Jonathan Corbet、2018年7月18日: `EXPORT_SYMBOL_NS()` と `MODULE_IMPORT_NS()` の当初の提案と、30,000個のフラットなエクスポート済みシンボルが解決に値する問題だった理由
- [A new version of modversions](https://lwn.net/Articles/986892/) — Jonathan Corbet、2024年8月26日: 上で説明したシンボルCRCの歴史。1.1.85（1995年1月）の最初のmodversionsから、`genksyms` のCパーサを経て、Sami Tolvanenによる代替の `gendwarfksyms` まで — これはCを解析する代わりにコンパイラのDWARF出力を読む。`genksyms` がRustを扱えないためである。その成果が、`include/linux/export.h` から参照される `CONFIG_GENDWARFKSYMS` の経路である

### 外部資料 {#external}

- [The kernel's command-line parameters](https://docs.kernel.org/admin-guide/kernel-parameters.html) — パラメータを設定するもう1つの方法: ブートコマンドラインでの `modulename.param=value`。これは `=y` で組み込まれたコードのパラメータを設定する唯一の方法である（付随させる `modprobe` の呼び出しがないため）。また実行時の形式 `echo -n value > /sys/module/${modulename}/parameters/${parm}` も扱う
- [Symbol Namespaces](https://docs.kernel.org/core-api/symbol-namespaces.html) — 現在の権威ある構文: 引用符付きの名前空間文字列を使う `EXPORT_SYMBOL_NS(usb_stor_suspend, "USB_STORAGE")` と `MODULE_IMPORT_NS("USB_STORAGE")`、`DEFAULT_SYMBOL_NAMESPACE`、不足しているインポートを自動的に追加する `make nsdeps` の使い方
- [Kconfig Language](https://docs.kernel.org/kbuild/kconfig-language.html) — `bool`/`tristate` のリファレンスと、`depends on` と `select`（選択されたシンボル自身の依存関係を無視する逆依存）の正確な意味
- [Linux Kernel Makefiles](https://docs.kernel.org/kbuild/makefiles.html) — `obj-$(CONFIG_FOO)` によるゴール定義と、複数のオブジェクトからビルドされる複合モジュール
- [Building External Modules](https://docs.kernel.org/kbuild/modules.html) — `M=` が実際に何を意味するかと、ツリー外ビルドのための `make -C $KDIR M=$PWD modules` のパターン
