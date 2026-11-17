# カーネルモジュールの作成とロード {#writing-and-loading-kernel-modules}

> ソースから実行中のコードに至るまでのカーネルモジュールのライフサイクル

## 最小のカーネルモジュール {#a-minimal-kernel-module}

```c
/* hello.c */
#include <linux/module.h>
#include <linux/init.h>
#include <linux/kernel.h>

static int __init hello_init(void)
{
    pr_info("Hello, kernel!\n");  /* カーネルログに出力する */
    return 0;                      /* 非0 = ロード失敗 */
}

static void __exit hello_exit(void)
{
    pr_info("Goodbye, kernel!\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Example Author");
MODULE_DESCRIPTION("A minimal hello world module");
MODULE_VERSION("1.0");
```

```makefile
# Makefile
obj-m := hello.o

# ソースがツリー外にある場合:
KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

```bash
# ビルド
make

# ロード
sudo insmod hello.ko

# 出力を確認
dmesg | tail -3
# [12345.678] Hello, kernel!

# アンロード
sudo rmmod hello

dmesg | tail -3
# [12346.789] Goodbye, kernel!
```

## モジュールのライフサイクル {#module-lifecycle}

```
insmod/modprobe                                rmmod
    │                                              │
    ▼                                              ▼
load_module()                                  delete_module()
    │                                              │
    ├── module_sig_check()                         ├── 参照カウントが0でなければならない
    ├── elf_validity_cache_copy()                  │   (try_stop_module)
    ├── early_mod_check()                          ├── mod->exit()
    │     └── .modinfoを解析し、vermagicを確認     │
    ├── layout_and_allocate()                      └── free_module()
    │     └── layout_sections()                        └── free_mod_mem()
    ├── add_unformed_module()                              └── module_memory_free()
    ├── module_augment_kernel_taints()                         └── execmem_free(mem->base)
    │     └── MODULE_LICENSEの確認（カーネルのtaintはここで設定される）
    ├── find_module_sections()   /* mod->syms, crcs, flagstab */
    ├── simplify_symbols() / apply_relocations()
    ├── complete_formation()
    │     └── MODULE_STATE_COMING: モジュールが/proc/modulesに
    │         見えるようになり、シンボルが他のモジュールにエクスポートされる
    │
    └── return do_init_module(mod);   /* load_module()の最後の文 */
        │  (mod->initを実行)
        ├── 0を返す: MODULE_STATE_LIVE
        └── -errnoを返す: モジュールは即座にアンロードされる
```

## モジュールのセクション {#module-sections}

モジュールの `.ko` ファイルは、特殊なセクションを持つELFである。

```bash
objdump -h hello.ko | grep -E "(\.init|\.exit|\.text|\.data|\.rodata|__param|\.modinfo)"
# .init.text   hello_init()を含む（モジュールのロード後に解放される）
# .exit.text   hello_exit()を含む（モジュールのアンロードまで保持される）
# .text        通常のコード
# .data        読み書き可能なデータ
# .rodata      読み取り専用データ
# __param      module_paramの定義
# .modinfo     MODULE_LICENSE、MODULE_AUTHORなど
```

`__init` 属性は、モジュールのinitが実行された後に解放すべき関数を示す（メモリの節約になる）。同様に `__exit` のコードは、アンロードされない常駐モジュールでは破棄できる。

## /proc/modules {#procmodules}

```bash
lsmod
# Module                  Size  Used by
# e1000e               262144  0
# ptp                   28672  1 e1000e

cat /proc/modules
# e1000e 262144 0 - Live 0xffffffffc0400000 （説明用の例にすぎない --
#   ディストリビューションが署名した標準のe1000eにはtaintフラグはない。
#   (OE)はツリー外の未署名ビルドを意味する）
# フィールド: 名前 サイズ 参照カウント 依存 状態 アドレス (フラグ)
# フラグ: O=ツリー外, E=未署名, F=強制ロード

# モジュールの依存関係を見る
cat /lib/modules/$(uname -r)/modules.dep | grep e1000e
```

## カーネル内のmodule構造体 {#the-module-struct-in-the-kernel}

```c
/* include/linux/module.h */
struct module {
    enum module_state state;       /* MODULE_STATE_LIVE/COMING/GOING/UNFORMED */
    struct list_head  list;        /* modulesリストにつながれる */
    char              name[MODULE_NAME_LEN];

    struct module_kobject mkobj;   /* /sys/module/name/ */
    struct module_attribute *modinfo_attrs;

    const char       *version;
    const char       *srcversion;  /* ソースのハッシュ */
    const char       *imported_namespaces;  /* MODULE_IMPORT_NS()のリスト */
    struct kobject   *holders_dir; /* /sys/module/name/holders/ */

    /* エクスポートされたシンボル — 通常のエクスポートとGPL専用の
     * エクスポートの両方を1つのテーブルで扱う。flagstab[i]はsyms[i]の
     * フラグバイト（KSYM_FLAG_GPL_ONLY）を保持する。gpl_syms/gpl_crcsの
     * ような別テーブルは存在しない。 */
    const struct kernel_symbol *syms;
    const u32                  *crcs;
    const u8                   *flagstab;
    unsigned int                num_syms;

    /* パラメータ: */
    struct kernel_param *kp;
    unsigned int num_kp;

    /* このモジュールがGPL専用シンボルを1つでも解決したら設定される: */
    bool using_gplonly_symbols;

    /* 起動関数（CONFIG_MODULE_UNLOADの配下ではない）: */
    int (*init)(void);

    /* モジュールのメモリ: 領域ごとに1エントリ — MOD_TEXT, MOD_DATA,
     * MOD_RODATA, MOD_RO_AFTER_INIT, MOD_INIT_TEXT, MOD_INIT_DATA,
     * MOD_INIT_RODATA。6.4で、以前のstruct module_layoutによる
     * core_layout/init_layoutの組を置き換えた。struct module_layoutは廃止された。 */
    struct module_memory mem[MOD_MEM_NUM_TYPES] __module_memory_align;

    /* tracepoint: */
#ifdef CONFIG_TRACEPOINTS
    unsigned int      num_tracepoints;
    tracepoint_ptr_t *tracepoints_ptrs;
#endif

#ifdef CONFIG_MODULE_UNLOAD
    struct list_head source_list;  /* 自分に依存するモジュール */
    struct list_head target_list;  /* 自分が依存するモジュール */
    void (*exit)(void);            /* 破棄関数 */
    atomic_t refcnt;               /* モジュールの参照カウント */
#endif
    /* ... */
};
```

## pr_* によるログ出力 {#pr_-logging}

```c
/* ログ出力マクロ（printkを直接使うよりこちらを推奨）: */
pr_emerg("...");   /* KERN_EMERG — システムが使用不能 */
pr_alert("...");   /* KERN_ALERT — 直ちに対処が必要 */
pr_crit("...");    /* KERN_CRIT */
pr_err("...");     /* KERN_ERR — エラー状態 */
pr_warn("...");    /* KERN_WARNING */
pr_notice("...");  /* KERN_NOTICE */
pr_info("...");    /* KERN_INFO */
pr_debug("...");   /* KERN_DEBUG — CONFIG_DYNAMIC_DEBUG（実行時に切り替え可能）またはDEBUGが定義されていない限りコンパイル時に除去される */

/* デバイス固有のログ出力（デバイス名を前置する）: */
dev_err(&pdev->dev, "failed to allocate: %d\n", ret);
dev_info(&pdev->dev, "initialized at %px\n", base);  /* 4.15以降、%pだけでは
                                                        ハッシュ化された役に立たない
                                                        値が出力される */

/* レート制限付きのログ出力: */
pr_info_ratelimited("too many events\n");

/* Dynamic debug: 実行時に選択的に有効化する */
/* echo "module hello +p" > /proc/dynamic_debug/control */
pr_debug("This only shows when enabled dynamically\n");
```

## モジュール署名 {#module-signing}

最近のカーネルは、モジュールが信頼された鍵で署名されていることを強制できる。

```bash
# カーネルが署名済みモジュールを要求するか確認する
cat /sys/module/module/parameters/sig_enforce  # Y = 未署名モジュールは拒否される

# 無関係なキルスイッチ: 一度設定するとモジュールのロード*と*アンロードをすべて禁止する
cat /proc/sys/kernel/modules_disabled  # 0=許可, 1=init_module/finit_module/delete_module不可（元に戻せない）

# モジュールに署名する
/usr/src/linux-headers-$(uname -r)/scripts/sign-file \
    sha256 signing_key.pem signing_key.x509 hello.ko

# モジュールの署名を確認する
modinfo hello.ko | grep sig

# カーネルのブートパラメータ:
# module.sig_enforce=1   — 署名を要求する（未署名のモジュールはすべて拒否される）
```

## モジュールのデバッグ {#module-debugging}

```bash
# Dynamic debug: モジュールのpr_debug()を有効にする
# 制御ファイル: /proc/dynamic_debug/control、またはdebugfsが有効なら
# 同等の/sys/kernel/debug/dynamic_debug/control
echo "module hello +p" > /proc/dynamic_debug/control
echo "file hello.c +p" > /proc/dynamic_debug/control
echo "func hello_init +p" > /proc/dynamic_debug/control

# dynamic debugの設定をすべて表示する
cat /proc/dynamic_debug/control

# メモリエラーにはKASAN（CONFIG_KASAN=y）
# モジュールをロードしてバグを起こす → KASANがuse-after-freeなどを報告する

# モジュールのkallsymsを確認する
grep hello /proc/kallsyms
# ffffffffc0401000 t hello_exit [hello]
# 注意: hello_initはここには表示されない。do_init_module()はinitの実行後に
# mod->kallsymsをcore_kallsymsへ切り替え、そのテーブルはinit系セクションに
# あるシンボルをすべて除外する — そのため__initシンボルはモジュールの
# ロードが完了した時点で消える。

# gdbによるクラッシュのデバッグ: セクションのベースアドレスはシンボルの
# アドレスではなくsysfsから得る（このモジュールの関数は__init/__exitの2つだけなので
# .text自体は空でsysfsのエントリもない -- 代わりに.exit.textを読む。
# 常駐し続けていてデバッグ対象なのはhello_exitだからである）
cat /sys/module/hello/sections/.exit.text
# 0xffffffffc0401000
gdb vmlinux
(gdb) add-symbol-file /path/to/hello.ko 0xffffffffc0401000
(gdb) list hello_exit
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/module/main.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/main.c) — ローダのコア: `load_module()`、`layout_sections()`、`do_init_module()`（モジュールの `init` 関数を呼ぶ）、およびアンロード経路の `free_module()`
- [include/linux/module.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/module.h) — `struct module` 本体と、hello worldの例で使った `module_init()`/`module_exit()` および `MODULE_LICENSE()`/`MODULE_AUTHOR()`/`MODULE_DESCRIPTION()`/`MODULE_VERSION()` マクロ
- [include/linux/init.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/init.h) — `__init` と `__exit` が `__section(".init.text")` と `__section(".exit.text")` として定義されている場所。上で説明した破棄可能セクションの挙動の背後にある属性である
- [include/linux/moduleparam.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/moduleparam.h) — `.modinfo` セクションに出力する `MODULE_INFO()` と、`__param` に出力する `module_param()`。上の `objdump` が示す2つのセクションである
- [kernel/module/procfs.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/procfs.c) — `/proc/modules` の行形式（名前、サイズ、参照カウント、依存リスト、`Live`/`Loading`/`Unloading` の状態、ベースアドレス、taintフラグ）を生成する
- [include/linux/printk.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/printk.h) — `pr_emerg()` … `pr_debug()` のファミリーと `pr_info_ratelimited()`

### manページ {#man-pages}

- [`insmod(8)`](https://man7.org/linux/man-pages/man8/insmod.8.html) — 「モジュールを挿入する単純なプログラム」。モジュールの依存関係を扱う `modprobe(8)` を参照するよう案内している
- [`rmmod(8)`](https://man7.org/linux/man-pages/man8/rmmod.8.html) — アンロードと、`-f`/`--force` に `CONFIG_MODULE_FORCE_UNLOAD` が必要な理由
- [`modprobe(8)`](https://man7.org/linux/man-pages/man8/modprobe.8.html) — `modules.dep.bin` による依存関係を考慮したロードと、モジュールとそれによって使われなくなった依存モジュールを削除する `-r`
- [`modinfo(8)`](https://man7.org/linux/man-pages/man8/modinfo.8.html) — `.modinfo` のフィールド（`license`、`author`、`description`、`parm`、`depends`、`alias`、`filename`）を出力する。kmodは `sig*` の署名フィールドも出力するが、このmanページには記載されていない点に注意
- [`lsmod(8)`](https://man7.org/linux/man-pages/man8/lsmod.8.html) — 上で示した `/proc/modules` を整形して表示する
- [`depmod(8)`](https://man7.org/linux/man-pages/man8/depmod.8.html) — `modprobe` が読む `modules.dep` ファイルを生成する

### 関連ページ {#related-pages}

- [モジュールロードの内部](module-loading-internals.md) — ELFが届いてから `init` が呼ばれるまでの間に `load_module()` が実際に何をしているか
- [パラメータ、シンボル、Kconfig](module-params.md) — `module_param()` と `EXPORT_SYMBOL`/`EXPORT_SYMBOL_GPL`
- [モジュール署名](module-signing.md) — 上の署名のセクションで概略を示した署名強制の仕組み
- [Kbuild](kbuild.md) — `obj-m` とツリー外の `M=$PWD` ビルドが実際にどう動くか
- [Dynamic Debug](dynamic-debug.md) — `echo "module hello +p"` の背後にある完全なクエリ構文
- [プラットフォームドライバ](../drivers/platform-driver.md) — ほとんどのドライバはモジュールである
- [BPF検証器](../bpf/bpf-verifier.md) — `.ko` のロード経路を使わずに、カーネルが信頼できない拡張コードを受け入れる方法: 署名やシンボルの確認の代わりに静的検証を行う

### LWNの記事 {#lwn-articles}

- [Two approaches to tightening restrictions on loadable modules](https://lwn.net/Articles/998221/) — Jonathan Corbet、2024年11月: `MODULE_LICENSE("GPL")` が形式的なものではなく実際に効力を持つ宣言である理由。GPLv3をGPLと宣言した件をめぐる論争や、モジュールごとのシンボルエクスポート名前空間を扱う
- [Yet another memory allocator for executable code](https://lwn.net/Articles/933867/) — Jonathan Corbet、2023年6月: モジュールのテキスト用メモリがどのように割り当てられるか、そして `module_alloc()` を共有の実行可能メモリアロケータで置き換える取り組み

### 外部資料 {#external}

- [Building External Modules — The Linux Kernel documentation](https://docs.kernel.org/kbuild/modules.html) — 上のMakefileで使った `obj-m` と `make -C $KDIR M=$PWD modules` のパターンに関するアップストリームのリファレンス
- [Kernel module signing facility — The Linux Kernel documentation](https://docs.kernel.org/admin-guide/module-signing.html) — `scripts/sign-file`、`module.sig_enforce=1`、`CONFIG_MODULE_SIG_FORCE`
- [Dynamic debug — The Linux Kernel documentation](https://docs.kernel.org/admin-guide/dynamic-debug-howto.html) — 実行時に `pr_debug()` を有効にするための `module`/`file`/`func` のクエリ言語と `+p` フラグ
- [Message logging with printk — The Linux Kernel documentation](https://docs.kernel.org/core-api/printk-basics.html) — 各 `pr_*()` マクロの背後にある `KERN_*` ログレベルと、モジュールのメッセージに接頭辞を付ける `pr_fmt()`
