# モジュール初期化とinitcall {#module-init-and-initcalls}

> カーネルモジュールと組み込みサブシステムはどのように初期化されるか

## `module_init()`と`module_exit()` {#module_init-and-module_exit}

すべてのカーネルモジュールや組み込みサブシステムは、2つのマクロを使って初期化関数とクリーンアップ関数を宣言する:

```c
#include <linux/module.h>
#include <linux/init.h>

static int __init mydriver_init(void)
{
    pr_info("mydriver: initializing\n");
    /* ... デバイスを登録し、リソースを割り当てる ... */
    return 0;  /* 0 = 成功、負の errno = 失敗 */
}

static void __exit mydriver_exit(void)
{
    pr_info("mydriver: exiting\n");
    /* ... リソースを解放する ... */
}

module_init(mydriver_init);
module_exit(mydriver_exit);
```

### これらのマクロが展開される内容 {#what-these-macros-expand-to}

**ロード可能モジュール**（`.ko`ファイル）の場合、`module_init()`はエイリアスを作成する:

```c
/* include/linux/module.h（ロード可能モジュールの場合） */
#define module_init(initfn)                         \
    static inline initcall_t __maybe_unused         \
    __inittest(void) { return initfn; }             \
    int init_module(void) __copy(initfn)            \
        __attribute__((alias(#initfn)));
```

こうしてできた`init_module`シンボルが、`.ko`をロードした後にカーネルのモジュールローダが呼び出すものである。

**組み込みモジュール**（Kconfigで`y`）の場合、`module_init()`は直接`device_initcall()`に展開されるわけではない —
間に間接層が1つ挟まる:

```c
#define module_init(x)     __initcall(x);
#define __initcall(fn)      device_initcall(fn)
#define device_initcall(fn) __define_initcall(fn, 6)
```

`__initcall(fn)`自体は`device_initcall(fn)`の単なるエイリアスである（動作上の違いはない — どちらも
initcallレベル6に入る）が、`module_init()`が実際に呼び出すのはこのマクロである。`__define_initcall(fn, 6)`が
関数ポインタを`.initcall6.init` ELFセクションに配置し、それがブート中に`do_initcalls()`によって呼び出される。initcallレベルの
完全な表は[初期ブートとstart_kernel()](early-boot.md)を参照。

組み込みコードに対する`module_exit()`は何も展開しない — 組み込みコードは決してアンロードできないので、exit関数は不要である（そしてリンカによって`.exit.text`から破棄される）。

---

## `struct module` {#struct-module}

ロードされた各モジュールは、モジュールメモリに割り当てられた`struct module`で表現される:

```c
/* include/linux/module.h（簡略化） */
struct module {
    enum module_state state;        /* COMING、LIVE、GOING、UNFORMED */

    struct list_head list;          /* modules リストにつながれる */

    char name[MODULE_NAME_LEN];     /* モジュール名（例: "e1000e"） */

    /* エクスポートされたシンボル（GPL と非 GPL は 1 つの統合テーブルを共有する。
     * 各エントリのライセンスは別の配列ではなく flagstab に保持される） */
    const struct kernel_symbol *syms;
    const u32 *crcs;
    const u8 *flagstab;
    unsigned int num_syms;

    /* init 関数と exit 関数 */
    int (*init)(void);
    void (*exit)(void);

    /* 参照カウント */
    atomic_t refcnt;

    /* モジュールの依存関係 */
    struct list_head source_list;   /* このモジュールを使うモジュール（依存元） */
    struct list_head target_list;   /* このモジュールが依存するモジュール */

    /* ソースセクション情報（/sys/module/<name>/sections/ 用） */
    struct module_sect_attrs *sect_attrs;

    /* パラメータ（/sys/module/<name>/parameters/ 用） */
    struct kernel_param *kp;
    unsigned int num_kp;

    /* ビルド ID */
    unsigned char build_id[BUILD_ID_SIZE_MAX];
};
```

モジュールの状態（`enum module_state`）:

| 状態 | 意味 |
|-------|---------|
| `MODULE_STATE_LIVE` | 完全にロードされ、動作中 |
| `MODULE_STATE_COMING` | ロード中。`init()`はまだ呼ばれていない |
| `MODULE_STATE_GOING` | アンロード中。`exit()`はまだ呼ばれていない |
| `MODULE_STATE_UNFORMED` | 部分的に構築された状態（初期の割り当ては完了） |

---

## モジュールのロードの流れ {#module-loading-flow}

カーネルはモジュールをロードするための2つのシステムコールを公開している:

- `init_module(buf, len, params)` — ユーザー空間メモリ上のバッファからモジュールイメージをロードする
- `finit_module(fd, params, flags)` — ユーザー空間のバッファではなく、すでにオープンされたファイルディスクリプタからモジュールをロードする（3.8以降、特にコンテナランタイムやsystemdで推奨される）。どちらのシステムコールも同じ`load_module()`に集約されるので、`module_sig_check()`はどちらの場合も同じように実行される — `finit_module`は署名の扱いを変えるのではなく、モジュールイメージがカーネルに届く方法を変えるものである

`insmod`は`init_module`を、`modprobe`は`finit_module`を使う。

### `load_module()`: ステップごとの解説 {#load_module-step-by-step}

`kernel/module/main.c`の`load_module()`が主な処理を担う:

```
load_module()
    1. module_sig_check()       [if CONFIG_MODULE_SIG]
       - Runs FIRST, before the ELF is even parsed
       - Verify the module's signature against the kernel's built-in
         public key (not PKCS#7 — see "Module signing" below for the
         real, non-standard signature format)
       - Fail if CONFIG_MODULE_SIG_FORCE and signature invalid

    2. elf_validity_cache_copy()
       - Verify ELF magic, architecture, section headers

    3. layout_and_allocate()
       - Compute section sizes
       - module_alloc(): allocate memory in module space
         (vmalloc region, or close to kernel text for 32-bit)

    4. add_unformed_module()
       - Adds the module to the global modules list, state =
         MODULE_STATE_UNFORMED, so a concurrent finit_module() of the
         same module is detected and rejected early
       - Runs right after allocation — well before init() is ever
         called, not after it

    5. Apply ELF relocations
       - apply_relocations(): resolve symbol references
       - Symbols resolved against:
         a. __ksymtab (unified table; each entry carries its own
            GPL/non-GPL license string, not a separate table)
         b. Other loaded modules' exported symbols

    6. post_relocation() -> module_finalize()
       - Set up alternatives (CPU feature patching)
       - Set up jump labels
       - Set up ORC unwind info

    7. complete_formation()
       - verify_exported_symbols(): reject a module that redefines an
         existing exported symbol name
       - Enable RO/NX/ROX memory protections on the module's sections
       - Set state = MODULE_STATE_COMING (do_init_module(), next, does
         NOT set this — it's already set by the time init() runs)

    8. mod_sysfs_setup()
       - Module directory created in /sys/module/

    9. do_init_module()
       - call module->init()   <-- driver's __init function runs
       - If init() returns 0: state = MODULE_STATE_LIVE
       - If init() returns non-zero: module loading fails,
         module freed, error returned to userspace
```

### モジュールのメモリレイアウト {#module-memory-layout}

モジュールのコードは、メインのカーネルテキストとは別の特別な仮想アドレス範囲にロードされる。x86-64では、これは`0xffffffffc0000000`以上の範囲（モジュール空間）である。この分離により、スタックトレースでモジュールがコアカーネルのコードと混同されることがなくなり、アンロード時にモジュールメモリを解放しやすくなる。

---

## モジュールのアンロード {#module-unloading}

```
rmmod mydriver
    → delete_module("mydriver", flags) syscall
    → kernel/module/main.c: free_module()

    1. Check refcnt: module_refcount() must be 0
       - If non-zero: EWOULDBLOCK
       - --force flag bypasses this (dangerous)

    2. set state = MODULE_STATE_GOING

    3. Call module->exit() if it exists

    4. free_module(): mod_sysfs_teardown() first
       - Remove from /sys/module/ (before the list unlink below,
         so nothing can look the module up mid-teardown)

    5. Remove from modules list and /proc/modules
       - list_del_rcu(&mod->list): a concurrent kallsyms walk of the
         list may still be in progress

    6. Synchronize with RCU (synchronize_rcu(), not rcu_barrier())
       - Waits out that grace period before the module's memory
         (which the list node lives inside) can be freed

    7. free_mod_mem(): return memory to vmalloc allocator
```

モジュールは、その中で何らかのコードが実行中である間（`refcnt`で追跡される）や、他のモジュールがそれに依存している間（`source_list`で追跡される）はアンロードできない。

---

## シンボルのエクスポート {#symbol-export}

ロード可能モジュールからアクセスできるのは、明示的にエクスポートされたシンボルだけである。どちらのマクロも同じ下位の`__EXPORT_SYMBOL()`に集約され、違いは各エントリが持つライセンス文字列だけである:

```c
/* include/linux/export.h */
#define _EXPORT_SYMBOL(sym, license)  __EXPORT_SYMBOL(sym, license, DEFAULT_SYMBOL_NAMESPACE)
#define EXPORT_SYMBOL(sym)            _EXPORT_SYMBOL(sym, "")
#define EXPORT_SYMBOL_GPL(sym)        _EXPORT_SYMBOL(sym, "GPL")
```

各エクスポートは、単一の統合された`__ksymtab`セクションに`struct kernel_symbol`を作成する（`kernel/module/internal.h`）:

```c
struct kernel_symbol {
    int value_offset;    /* この構造体からシンボルへのオフセット */
    int name_offset;     /* この構造体から名前文字列へのオフセット */
    int namespace_offset;
};
```

ライセンス文字列自体は、別のシンボルテーブルではなく、モジュールの`flagstab`配列（上の`struct module`を参照）に置かれる — `__ksymtab_gpl`は存在しない。モジュールのロード中、再配置ステップは`__ksymtab`（およびロード済みモジュールの`__ksymtab`）を検索して未定義シンボルを解決する。解決したシンボルのライセンスがGPL限定で、ロードしようとしているモジュールの`MODULE_LICENSE`がGPL互換でない場合、ロードは失敗する:

```
ERROR: "some_gpl_function" [drivers/mydriver/mydriver.ko] undefined!
```

ユーザー空間から:
```bash
# エクスポートされたすべてのカーネルシンボルを一覧表示:
cat /proc/kallsyms | grep ' T '    # T = グローバルなテキストシンボル

# 特定のモジュールがエクスポートしたシンボルを一覧表示:
cat /proc/kallsyms | grep '\[mydriver\]'

# .ko がエクスポートするシンボルを一覧表示:
nm mydriver.ko | grep '__ksymtab'

# .ko がインポートする（他のモジュールから必要とする）シンボルを一覧表示:
modinfo mydriver.ko
# または
nm mydriver.ko | grep ' U '  # U = 未定義（インポート）
```

---

## ロード時のモジュールパラメータ {#module-parameters-during-load}

```c
/* モジュールのソース内で: */
static int timeout_ms = 100;
module_param(timeout_ms, int, 0644);
MODULE_PARM_DESC(timeout_ms, "Timeout in milliseconds (default 100)");

static char *device_name = "default";
module_param(device_name, charp, 0444);
MODULE_PARM_DESC(device_name, "Target device name");
```

```bash
# ロード時にパラメータを渡す:
modprobe mydriver timeout_ms=500 device_name=eth0
insmod mydriver.ko timeout_ms=500

# 実行時にパラメータを読み書きする（パーミッションが許せば）:
cat /sys/module/mydriver/parameters/timeout_ms
echo 200 > /sys/module/mydriver/parameters/timeout_ms
```

`module_param()`マクロは`.data..param`セクションに`struct kernel_param`を登録する。`load_module()`は`init()`を呼び出す前に、`finit_module()`に渡されたパラメータ文字列に対して`parse_args()`を呼び出すので、パラメータはドライバが初期化される前に設定される。

---

## モジュールの依存関係 {#module-dependencies}

依存関係の解決は`modprobe`（`kmod`パッケージに含まれる）が行う。依存関係データベースは`depmod`によって構築される:

```bash
# 新しいモジュールをインストールした後、モジュール依存関係データベースを再構築する:
depmod -a

# 結果:
cat /lib/modules/$(uname -r)/modules.dep
# drivers/net/ethernet/intel/e1000e/e1000e.ko.xz: \
#     kernel/drivers/pci/pci.ko.xz
```

`depmod`は各`.ko`ファイルの未定義シンボルを読み取り、他のモジュールのエクスポートシンボルと突き合わせて、有向の依存関係グラフを生成する。`modprobe`はこのグラフをたどり、前提となるモジュールが先にロードされるようにする。

```bash
# モジュールの依存関係を表示する:
modinfo e1000e | grep depends

# モジュールとそのすべての依存先をロードする:
modprobe e1000e

# モジュールと、それに依存するモジュールを削除する（使用中でなければ）:
modprobe -r e1000e
```

---

## /sys/module/と/proc/modules {#sysmodule-and-procmodules}

```bash
# ロード済みの全モジュールをサイズと参照カウント付きで一覧表示:
cat /proc/modules
# e1000e 290816 0 - Live 0xffffffffc0234000
# 形式: name size refcount dependencies state memaddr

# または lsmod で（より見やすい）:
lsmod

# モジュールの詳細情報:
ls /sys/module/e1000e/
# coresize  initsize  initstate  notes  parameters  refcnt  sections  srcversion  taint

cat /sys/module/e1000e/initstate
# live

# モジュールパラメータ:
ls /sys/module/e1000e/parameters/
cat /sys/module/e1000e/parameters/debug

# セクションアドレス（デバッグに有用）:
cat /sys/module/e1000e/sections/.text
```

---

## モジュール署名 {#module-signing}

`CONFIG_MODULE_SIG`は、モジュールのロード時に暗号学的な署名検証を有効にする。カーネルは公開鍵を埋め込んでおり、モジュールはビルド時に対応する秘密鍵で署名される。

```bash
# モジュールに手動で署名する（鍵はカーネルに組み込まれた公開鍵と一致している必要がある）:
scripts/sign-file sha256 signing_key.pem signing_key.x509 mydriver.ko

# 署名が存在することを確認する（検証はしない）:
modinfo mydriver.ko | grep sig
tail -c 28 mydriver.ko   # "~Module signature appended~" と表示されるはず
```

`CONFIG_MODULE_SIG_FORCE`を有効にすると、署名のないモジュールや署名が正しくないモジュールはすべて拒否される。`CONFIG_MODULE_SIG`のみで`_FORCE`がない場合、署名のないモジュールはtaintフラグ付きで受け入れられる:

```
mydriver: module verification failed: signature and/or required key missing
Tainted: G           E
```

署名は、ELFデータの後ろに付加される小さなカーネル独自の`struct module_signature`トレーラであり、
PKCS#7（やその他の業界標準）のコンテナではない。カーネル自身のドキュメントによれば、「the
signatures are not themselves encoded in any industrial standard type」（署名自体はいかなる業界標準の形式でもエンコードされていない）。このトレーラの`id_type`フィールドは、
包んでいる署名バイト列のペイロード形式を示しはする — 現在定義されているのは`PKCS7`のみ — が、それをオフラインで検証する
汎用の`openssl`風コマンドは存在しない。検証はロード時にカーネル内部でのみ、カーネルに組み込まれた公開鍵に対して
行われる。

---

## initcallのデバッグ {#initcall-debugging}

```bash
# initcall_debug 付きでブートし、すべての initcall とその所要時間を確認する:
# カーネルコマンドラインに追加:
initcall_debug

# dmesg 内:
# calling  e1000e_init_module+0x0/0x3c [e1000e] @ 3
# initcall e1000e_init_module+0x0/0x3c [e1000e] returned 0 after 1243 usecs

# "@ N" は CPU 番号であり、並列性の問題を特定するのに役立つ。

# 関数がどの initcall レベルにあるかを調べる:
grep '__initcall_' /proc/kallsyms | grep 'mydriver'

# モジュールのロードパスをトレースする:
bpftrace -e '
kprobe:do_init_module {
    printf("loading module: %s\n",
           str(((struct module *)arg0)->name));
}'
```

---

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/module.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/module.h) — `module_init()` / `module_exit()`マクロと`struct module`の定義
- [include/linux/init.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/init.h) — initcallレベルのマクロ（`early_initcall()`から`late_initcall()`まで）と`__define_initcall()`
- [kernel/module/main.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/main.c) — `load_module()`と、それが順に呼び出すステップ: `module_sig_check()`、`elf_validity_cache_copy()`、`layout_and_allocate()`、`add_unformed_module()`、`apply_relocations()`、`post_relocation()`、`complete_formation()`、`do_init_module()`。アンロードは`SYSCALL_DEFINE2(delete_module, ...)` → `try_stop_module()` → `free_module()`（`mod_sysfs_teardown()`、`synchronize_rcu()`、`free_mod_mem()`）
- [include/uapi/linux/module_signature.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/module_signature.h) — `struct module_signature`。署名済み`.ko`に付加される独自のトレーラ
- [include/linux/export.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/export.h) — `EXPORT_SYMBOL()`と`EXPORT_SYMBOL_GPL()`
- [kernel/module/internal.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/internal.h) — 現在の`struct kernel_symbol`の定義と`__ksymtab`の境界シンボル

### manページ {#man-pages}

- [`init_module(2)`](https://man7.org/linux/man-pages/man2/init_module.2.html) — ユーザー空間のバッファからモジュールをロードする
- [`finit_module(2)`](https://man7.org/linux/man-pages/man2/finit_module.2.html) — ファイルディスクリプタからモジュールをロードする。Linux 3.8で追加
- [`delete_module(2)`](https://man7.org/linux/man-pages/man2/delete_module.2.html) — `rmmod`の背後にあるシステムコール
- [`modprobe(8)`](https://man7.org/linux/man-pages/man8/modprobe.8.html) — 依存関係を考慮したモジュールのロード/削除
- [`depmod(8)`](https://man7.org/linux/man-pages/man8/depmod.8.html) — 各モジュールの未定義シンボルから`modules.dep`を構築する

### 関連ページ {#related-pages}

- [初期ブートとstart_kernel()](early-boot.md) — initcallレベルと`do_initcalls()`
- [カーネルモジュール](../modules/README.md) — ツリー外モジュールの作成とビルド
- [モジュール署名](../modules/module-signing.md) — 鍵の生成と署名のワークフロー
- [パラメータ、シンボル、Kconfig](../modules/module-params.md) — `module_param()`の詳細

### LWN記事 {#lwn-articles}

- [LWN: Enforcement (or not) for module-specific exported symbols](https://lwn.net/Articles/1029492/) — エクスポートを名前で指定したツリー内モジュールの集合に制限することをめぐる議論。議論されたマクロは`EXPORT_SYMBOL_GPL_FOR_MODULES()`として提案され、`GPL_`を外した`EXPORT_SYMBOL_FOR_MODULES()`としてマージされた（上のカーネルソースの項を参照）
- [LWN: The proper use of EXPORT_SYMBOL_GPL()](https://lwn.net/Articles/769471/) — エクスポートシンボルをGPL限定にすべきときをメンテナがどう判断するか

### 外部資料 {#external}

- [Kernel module signing facility](https://docs.kernel.org/admin-guide/module-signing.html) — `CONFIG_MODULE_SIG`、鍵の生成、`scripts/sign-file`
- [Building External Modules](https://docs.kernel.org/kbuild/modules.html) — ツリー外モジュールのためのkbuildの仕組み
- [Symbol Namespaces](https://docs.kernel.org/core-api/symbol-namespaces.html) — 名前空間付きエクスポートのための`EXPORT_SYMBOL_NS()`
