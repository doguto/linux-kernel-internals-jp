# モジュールロードの内部 {#module-loading-internals}

> load_module()が行うこと: ELFの解析、再配置、シンボル解決、バージョニング

## システムコール {#the-syscalls}

カーネルモジュールをロードできるシステムコールは2つある:

```c
/* ユーザー空間のバッファからロードする — 元々のインターフェース */
long init_module(void __user *umod, unsigned long len, const char __user *uargs);

/* ファイルディスクリプタからロードする — Linux 3.8 で追加 */
long finit_module(int fd, const char __user *uargs, int flags);
```

`finit_module()`が追加されたのは、モジュールがどんなバイト列を含むかだけでなく、それが*どこから来たか*をカーネルが判断できるようにするためである。ファイルディスクリプタはファイルシステム上のファイルを指すので、カーネルや任意のLSMは、匿名のユーザー空間バッファを判定するのではなく、ファイルの来歴 — 拡張属性に保持された署名や、dm-verityで保護されたルートファイルシステム — を考慮できる。どちらのシステムコールも、最終的には`kernel/module/main.c`の`load_module()`を呼び出す。

`finit_module()`が受け付けるフラグはちょうど3つであり、`flags`にそれ以外のものがあれば`EINVAL`で拒否される:

- `MODULE_INIT_IGNORE_MODVERSIONS` — バージョンセクションのインデックスをゼロにし、CRCの比較をスキップする
- `MODULE_INIT_IGNORE_VERMAGIC` — モジュールにvermagic文字列がないものとして扱い、バージョン文字列の比較をスキップする
- `MODULE_INIT_COMPRESSED_FILE` — ファイルは、カーネルがビルドされたときの単一の形式（`CONFIG_MODULE_COMPRESS_GZIP`、`_XZ`、`_ZSTD`のいずれか — `kernel/module/decompress.c`は`#if`/`#elif`の連鎖でちょうど1つを選択する）で圧縮された`.ko`である。カーネルが`module_decompress()`を介して自ら展開する

最初の2つのフラグは`try_to_force_load()`を経由する。これは`CONFIG_MODULE_FORCE_LOAD=y`であればカーネルを`TAINT_FORCED_MODULE`（`F`）でtaintし、そうでなければ`ENOEXEC`でロードを失敗させる。

## load_module(): ステップごとの解説 {#load_module-step-by-step}

`load_module()`は生のELFバイト列を受け取り、動作するモジュールを生成する。ステップ1は`load_module()`に入る直前のシステムコールラッパーで行われる。ステップ2から11は`load_module()`本体であり、処理が行われる順に並べてある（ステップ8のバージョンチェックは、独立したパスとしてではなく、ステップ7のシンボル解決の中で実行される）。

### 1. ELFイメージをカーネルに取り込む {#1-get-the-elf-image-into-the-kernel}

どちらのシステムコールも、`load_module()`にユーザー空間のポインタを渡すことはない。それぞれがまず`.ko`全体をカーネルメモリ上に実体化し、`struct load_info`を埋める。これは`info->hdr`（イメージ）と`info->len`（その長さ）を、以降のすべての段階に運ぶ。

`init_module()`は`copy_module_from_user()`を使う:

```c
/* kernel/module/main.c — 抜粋 */
info->hdr = __vmalloc(info->len, GFP_KERNEL | __GFP_NOWARN);
if (!info->hdr)
        return -ENOMEM;

if (copy_chunked_from_user(info->hdr, umod, info->len) != 0) {
        err = -EFAULT;
        goto out;
}
```

`copy_chunked_from_user()`は`copy_from_user()`を囲むループで、1回の反復で最大`16 * PAGE_SIZE`を移動し、チャンクの合間に`cond_resched()`を呼び出す。そのため、大きなモジュールのコピーがCPUを際限なく占有することはない。

`finit_module()`は別の経路をたどる。`idempotent_init_module()` → `init_module_from_file()`であり、後者は次のようにファイルを読み込む

```c
len = kernel_read_file(f, 0, &buf, INT_MAX, NULL,
                       compressed ? READING_MODULE_COMPRESSED :
                                    READING_MODULE);
```

`MODULE_INIT_COMPRESSED_FILE`が渡された場合は`module_decompress()`が`buf`を`info->hdr`に展開し、そうでなければ`buf`が*そのまま*`info->hdr`になる。`idempotent_*`ラッパーは並行するロードの重複を排除する。ファイルのinodeを小さなハッシュテーブルにハッシュし、別のタスクがすでに同じinodeをロード中であれば、2番目の呼び出し元は処理を二重に行う代わりに、単にcompletionを待って最初の呼び出し元の結果を返す。

いずれの場合も、得られるバッファは完全なELF再配置可能オブジェクトである。

### 2. モジュール署名をチェックする {#2-check-the-module-signature}

`module_sig_check()`は、どのセクションが調べられるよりも先に最初に実行される。そのため、真正性が確立される前に、壊れたイメージがローダーを誘導することはできない。また、`info->len`を調整してイメージ末尾に付加された署名を切り落とすので、以降のすべての境界チェックは実際のELFの長さに対して行われる。詳細は[カーネルモジュールの署名](module-signing.md)を参照。

### 3. ELFヘッダを検証し、セクションテーブルをキャッシュする {#3-validate-the-elf-headers-and-cache-the-section-table}

`elf_validity_cache_copy()`は構造上のチェックを行い、ローダーが後で必要とするセクションのインデックスを記録する。特に次の点を検証する:

- `e_ident`がELFのマジックバイト（`\x7fELF`）で始まる
- `e_type == ET_REL` — モジュールは再配置可能オブジェクトであり、実行ファイルや共有ライブラリではない
- `elf_check_arch()`が`e_machine`を受け入れる（x86-64では`EM_X86_64`）
- `e_shentsize`が稼働中のカーネルの`sizeof(Elf_Shdr)`に等しく、セクションヘッダ配列が`info->len`の範囲内にある
- セクション0が、サイズとアドレスがゼロの正しい`SHT_NULL`エントリである
- すべてのセクションの内容が範囲内にある

ここでの失敗は`ENOEXEC`を返す。成功すると、`info->mod`は一時イメージ内にあるモジュールのディスク上の`struct module`を指し、`info->index.*`はキャッシュされたセクションインデックス（`.modinfo`、シンボルテーブル、文字列テーブル、`__versions`、拡張modversionsのペア）を保持する。

### 4. 早期チェック: ブラックリスト、module_layoutのCRC、vermagic {#4-early-checks-blacklist-module_layout-crc-vermagic}

`early_mod_check()`は、何かを割り当てる前に実行できるチェックを行う:

- `blacklisted()`はモジュール名をカンマ区切りの`module_blacklist=`ブートパラメータと比較し、そこに含まれていれば`EPERM`を返す
- `rewrite_section_headers()`はすべての`sh_addr`を一時イメージに向け、`.modinfo`とバージョンセクションの`SHF_ALLOC`をクリアする。これにより、それらはロード中には使えるが、モジュールの実行時メモリには決してコピーされない
- `check_modstruct_version()`は擬似シンボル`module_layout`に対してmodversionsのチェックを行い、互換性のない`struct module`に対してビルドされたモジュールを捕捉する
- `check_modinfo()`は`.modinfo`にあるモジュールの`vermagic`文字列を稼働中のカーネルのものと比較し、不一致なら`ENOEXEC`を返す
- `module_patient_check_exists()`は名前の重複を拒否する — その名前の既存モジュールがまだ`COMING`か`UNFORMED`であればまず待機し、それが`LIVE`になった場合は`EEXIST`を、そうでなければ`EBUSY`を返す

vermagicの比較は、おなじみの「version magic」エラーの発生源である:

```c
} else if (!same_magic(modmagic, vermagic, info->index.vers)) {
        pr_err("%s: version magic '%s' should be '%s'\n",
               info->name, modmagic, vermagic);
        return -ENOEXEC;
}
```

`same_magic()`は、モジュールがCRCを持つ場合、文字列の先頭にあるカーネルリリースの語をスキップする。そのため、modversionsが有効なモジュールは、リリースの完全一致ではなく、残りの構成フィールド（SMP、プリエンプションモデル、モジュールのアンロードのサポートなど）で判定される。

### 5. モジュールメモリをレイアウトし、割り当てる {#5-lay-out-and-allocate-module-memory}

`layout_and_allocate()`は、すべての`SHF_ALLOC`セクションに7つのメモリクラスのいずれかの中でのオフセットを割り当て、続いて`move_module()`が各クラスを割り当ててセクションをコピーする。

古い2領域モデル — どちらも`struct module_layout`である`mod->core_layout`と`mod->init_layout` — はなくなった。コミット`ac3b43283923`（v6.4）以降、モジュールは独立に割り当てられる領域の配列を所有する:

```c
/* include/linux/module.h */
enum mod_mem_type {
        MOD_TEXT = 0,
        MOD_DATA,
        MOD_RODATA,
        MOD_RO_AFTER_INIT,
        MOD_INIT_TEXT,
        MOD_INIT_DATA,
        MOD_INIT_RODATA,

        MOD_MEM_NUM_TYPES,
        MOD_INVALID = -1,
};

struct module {
        ...
        struct module_memory mem[MOD_MEM_NUM_TYPES] __module_memory_align;
};
```

| クラス | 保持するもの | 寿命 |
|-------|-------|----------|
| `MOD_TEXT` | 実行可能セクション（`SHF_EXECINSTR \| SHF_ALLOC`） | アンロードまで |
| `MOD_DATA` | 書き込み可能なデータ、およびアーキテクチャのsmall-dataセクション | アンロードまで |
| `MOD_RODATA` | 割り当てられた書き込み不可のデータ | アンロードまで |
| `MOD_RO_AFTER_INIT` | `SHF_RO_AFTER_INIT`セクション — init中は書き込み可能で、その後封印される | アンロードまで |
| `MOD_INIT_TEXT` | `.init.text` | init後に解放 |
| `MOD_INIT_DATA` | `.init.data` | init後に解放 |
| `MOD_INIT_RODATA` | `.init.rodata` | init後に解放 |

このようにテキストとデータを分割しているからこそ、`complete_formation()`は、区別のない1つの塊に対してではなく、クラスごとに異なるページ権限を適用できる。

各クラスは`module_memory_alloc()`によって割り当てられる。これはもはや独自のアロケータを持たない — `module_alloc()`は汎用の実行可能メモリアロケータに置き換えられた:

```c
/* kernel/module/main.c — 抜粋 */
static int module_memory_alloc(struct module *mod, enum mod_mem_type type)
{
        unsigned int size = PAGE_ALIGN(mod->mem[type].size);
        enum execmem_type execmem_type;
        void *ptr;

        if (mod_mem_type_is_data(type))
                execmem_type = EXECMEM_MODULE_DATA;
        else
                execmem_type = EXECMEM_MODULE_TEXT;

        ptr = execmem_alloc_rw(execmem_type, size);
        ...
}
```

`EXECMEM_MODULE_TEXT`はアーキテクチャのモジュール領域に対応する。x86-64では、その領域は次の位置から始まる

```c
/* arch/x86/include/asm/pgtable_64_types.h */
#define MODULES_VADDR   (__START_KERNEL_map + KERNEL_IMAGE_SIZE)

#ifndef CONFIG_DEBUG_KMAP_LOCAL_FORCE_MAP
# define MODULES_END    _AC(0xffffffffff000000, UL)
#else
# define MODULES_END    _AC(0xfffffffffe000000, UL)
#endif
```

`__START_KERNEL_map`が`0xffffffff80000000`で、`KERNEL_IMAGE_SIZE`が1 GiB — `CONFIG_RANDOMIZE_BASE`（KASLR）が有効な場合に使われる値であり、事実上すべてのディストリビューションカーネルで有効になっている — であれば、モジュールは`0xffffffffc0000000`から始まる。ロードされたモジュールが`/proc/modules`で`0xffffffffc0400000`のようなアドレスを示すのはこのためである。KASLRがない場合、イメージの上限は512 MiBに縮み、モジュール領域は代わりに`0xffffffffa0000000`から始まって1.5 GiBに広がる。

カーネルイメージに近いことは偶然ではなく必須である。コンパイラは32ビットのPC相対再配置（`R_X86_64_PC32`、`R_X86_64_PLT32`）を生成し、その符号付き32ビットオフセットは命令から±2 GBの範囲にしか届かない。モジュールをそれより遠くに置けば、これらがオーバーフローしてしまう。

### 6. オプションのセクションを見つける {#6-find-the-optional-sections}

モジュールが最終的なアドレスに置かれた状態で、`find_module_sections()`は、カーネルが実行時に参照するセクションへのポインタを記録する。ステップ3でキャッシュされたインデックスと合わせて、ロードに関係するのは次のセクションである:

| セクション | 目的 |
|---------|---------|
| `.gnu.linkonce.this_module` | モジュールの`struct module` |
| `.modinfo` | ヌル区切りの`key=value`文字列（ライセンス、作者、説明、vermagic） |
| `.init.text` | 初期化コード — `mod->init()`が戻った後に解放される |
| `.exit.text` | クリーンアップコード — `CONFIG_MODULE_UNLOAD=y`ではアンロードまで保持される。そうでなければinitメモリとしてレイアウトされ、解放される |
| `__versions` | CRCチェック用の`struct modversion_info`の配列 |
| `__version_ext_crcs` / `__version_ext_names` | 同じデータの拡張modversions形式。長いシンボル名向け |
| `__ksymtab` | このモジュールが提供するエクスポートシンボル |
| `__kcrctab` | このモジュールのエクスポートシンボルのCRC |
| `__kflagstab` | `__ksymtab`のエントリごとに1バイトのフラグ — `KSYM_FLAG_GPL_ONLY`はGPL専用のエクスポートを示す |
| `__param` | モジュールの`module_param()`エントリ |

独立した`__ksymtab_gpl`はもはや存在しない。コミット`55fcb926b6d8`（v7.1）以降、GPL専用かどうかは並列する`__kflagstab`バイト配列の1ビットであり、ローダーは古いセクションを見つけると警告を出す:

```c
/* — 抜粋。それより前の mod->kp = section_objs(info, "__param", ...) は省略 — */
mod->syms = section_objs(info, "__ksymtab", sizeof(*mod->syms), &mod->num_syms);
mod->crcs = section_addr(info, "__kcrctab");
mod->flagstab = section_addr(info, "__kflagstab");

if (section_addr(info, "__ksymtab_gpl"))
        pr_warn("%s: ignoring obsolete section __ksymtab_gpl\n", mod->name);
if (section_addr(info, "__kcrctab_gpl"))
        pr_warn("%s: ignoring obsolete section __kcrctab_gpl\n", mod->name);
```

続いて`check_export_symbol_sections()`が、シンボルをエクスポートしているのに`__kflagstab`を持たないモジュールを`ENOEXEC`で拒否する。

### 7. シンボルを解決する {#7-resolve-symbols}

シンボル解決は再配置の**前**に行われ、そうでなければならない。未定義シンボルに対する再配置にはそのシンボルのアドレスが必要であり、それを供給するのがシンボル解決だからである。`load_module()`はまさにその順序でこれらを呼び出す:

```c
/* シンボルを修正し、st_value が位置へのポインタになるようにする。 */
err = simplify_symbols(mod, info);
if (err < 0)
        goto free_modinfo;

err = apply_relocations(mod, info);
if (err < 0)
        goto free_modinfo;
```

`simplify_symbols()`はモジュールのシンボルテーブルを走査し、各エントリの`st_shndx`で分岐する。定義済みシンボルは、`st_value`にそのセクションのベースアドレスが加算される。`SHN_ABS`シンボルはそのままにされる。percpuシンボルはモジュールのpercpu割り当てへと振り向けられる。未定義シンボルは検索パスに回される:

```c
case SHN_UNDEF:
        ksym = resolve_symbol_wait(mod, info, name);
        /* 解決できれば OK。 */
        if (ksym && !IS_ERR(ksym)) {
                sym[i].st_value = kernel_symbol_value(ksym);
                break;
        }
        ...
        ret = PTR_ERR(ksym) ?: -ENOENT;
        pr_warn("%s: Unknown symbol %s (err %d)\n", mod->name, name, ret);
```

`resolve_symbol_wait()`は`resolve_symbol()`を、30秒の猶予を持つ`wait_event_interruptible_timeout()`で包む。この待機は依存関係の順序付けのために存在する。シンボルをエクスポートするモジュール自体がロード途中でまだ`MODULE_STATE_COMING`にある場合、`strong_try_module_get()`は`-EBUSY`を返し、カーネルはロードを失敗させる代わりに、提供側が`LIVE`になるかタイムアウトが切れる（`gave up waiting for init of module %s`）までスリープする。

`resolve_symbol()`は`find_symbol()`を呼び出し、これは次の順に検索する:

1. vmlinux自身のエクスポート。`__start___ksymtab`と`__stop___ksymtab`のリンカシンボルの間にある
2. `MODULE_STATE_UNFORMED`ではない、ロード済みの全モジュールの`__ksymtab`

どちらも`struct kernel_symbol`のソート済み配列であり、`find_exported_symbol_in_section()`はハッシュ検索ではなく`bsearch()`で名前を探す。テーブルは提供側ごとに1つであり、2つではない — GPL専用かどうかは、検索がヒットした後に対応する`__kflagstab`のバイトから読み出される:

```c
sym = bsearch(fsa->name, syms->start, syms->stop - syms->start,
              sizeof(struct kernel_symbol), cmp_name);
if (!sym)
        return false;

sym_flags = *(syms->flagstab + (sym - syms->start));
if (!fsa->gplok && (sym_flags & KSYM_FLAG_GPL_ONLY))
        return false;
```

この`return false`こそが、プロプライエタリなモジュールがGPL専用のエクスポートを使おうとしても、独自の「ライセンス」エラーが出ない理由である。エントリは存在しないかのようにスキップされ、`find_symbol()`は失敗を報告し、`simplify_symbols()`は`-ENOENT`のパスに行き着いて、本当に存在しないシンボルと同様に`Unknown symbol <name>`を出力する。同じ検索処理は、GPL専用のエクスポートが*実際に*正常に使われたときに`mod->using_gplonly_symbols`もセットする。これにより、そのようなモジュールがプロプライエタリなモジュールからもインポートすることが後で阻止される。

### 8. CRCチェックサムを検証する {#8-verify-crc-checksums}

バージョンチェックは独立したパスではない — `resolve_symbol()`は、ステップ7の間に見つけた各シンボルについて`check_version()`を呼び出す。`CONFIG_MODVERSIONS=y`では、インポートされるすべてのシンボルが、モジュールの`__versions`セクションにCRCを持つ:

```c
struct modversion_info {
    unsigned long crc;
    char          name[MODULE_NAME_LEN];
};
```

`MODULE_NAME_LEN`は`64 - sizeof(unsigned long)`であり、64ビットでは56バイトになる — これではすべてのエクスポートシンボル名には足りない。新しい*拡張modversions*形式はレコードを2つに分け、`u32`の配列である`__version_ext_crcs`と、長さに制限のないヌル終端文字列の塊である`__version_ext_names`にする。`check_version()`は、それが存在すればそちらを優先する:

```c
/* 拡張バージョン情報があれば、それに依拠する */
if (info->index.vers_ext_crc) {
        for_each_modversion_info_ext(version_ext, info) {
                ...
        }
}
```

存在しなければ、固定サイズの`struct modversion_info`配列にフォールバックする。

いずれにせよ、モジュールに記録されたCRCは、*提供側*が供給したCRC（提供側の`__kcrctab`から読み出された`fsa.crc`）と比較される。不一致は、モジュールのビルドに使われたカーネルと稼働中のカーネルとで、シンボルの型シグネチャが異なることを意味する。`check_version()`は次のログを出し

```
mymodule: disagrees about version of symbol tcp_sendmsg
```

0を返す。すると`resolve_symbol()`はそのシンボルを`EINVAL`で失敗させる:

```c
if (!check_version(info, name, mod, fsa.crc)) {
        fsa.sym = ERR_PTR(-EINVAL);
        goto getname;
}
```

この`EINVAL`のパスに到達するのは*通常の*インポートシンボルだけである。`module_layout`擬似シンボルがこの経路を通ることはない。`modpost`は、モジュールのCRCテーブルに行を持たせるためだけに`sym_add_unresolved("module_layout", mod, false)`でそれを合成しており、モジュールにはそれへの未定義ELF参照が含まれない — そのため`simplify_symbols()`も`resolve_symbol()`もそれを見ることはない。そのCRCは、ずっと早いステップ4で`check_modstruct_version()`によって一度だけ比較され、`early_mod_check()`がその失敗を`ENOEXEC`（`Invalid module format`）に変換する。

失敗コードを区別しておく価値がある。ユーザー空間からは混同しやすいからだ:

| 条件 | errno | 発生元 |
|-----------|-------|-----------|
| vermagic文字列の不一致 | `ENOEXEC` | `check_modinfo()`、ステップ4 |
| `module_layout`のCRC不一致 | `ENOEXEC` | `early_mod_check()`経由の`check_modstruct_version()`、ステップ4 |
| インポートシンボルのCRC不一致 | `EINVAL` | `check_version()`経由の`resolve_symbol()` |
| シンボルが存在しない、またはGPL専用で到達できない | `ENOENT` | `simplify_symbols()` |

### 9. 再配置を適用する {#9-apply-relocations}

ELFの`.rela.*`セクションには再配置エントリが含まれており、ステップ7がすべての`st_value`を埋めたので、今や解決可能になっている。カーネルは次の関数でそれらを適用する:

```c
/* アーキテクチャ非依存のディスパッチ */
static int apply_relocations(struct module *mod, const struct load_info *info);

/* アーキテクチャ固有の実装（例: arch/x86/kernel/module.c） */
int apply_relocate_add(Elf64_Shdr *sechdrs, const char *strtab,
                       unsigned int symindex, unsigned int relsec,
                       struct module *mod);
```

x86-64では、よく使われる再配置タイプは次のとおりである:

| タイプ | 意味 |
|------|---------|
| `R_X86_64_64` | 64ビット絶対アドレス |
| `R_X86_64_PC32` | 32ビットPC相対（近くのシンボルへの`call`、`jmp`に使われる） |
| `R_X86_64_PLT32` | PLT経由の32ビットPC相対 — サンクを経由する可能性のある呼び出しに使われる |
| `R_X86_64_32S` | 32ビット符号拡張絶対アドレス |

各エントリについて、カーネルは`val = sym->st_value + r_addend`を計算し、PC相対タイプでは再配置自身のアドレスを減算して、その結果を再配置オフセットに書き込む。PC相対タイプ（`R_X86_64_PC32`/`PLT32`）は範囲チェックがまったく行われない — ターゲットがステップ5で述べた±2 GBの到達範囲を超えると、切り詰めは黙って行われる。一方、`R_X86_64_32S`と`R_X86_64_32`は絶対アドレスのタイプであり、`val`が縮小変換に耐えられなければ`ENOEXEC`（「overflow」）でロードを失敗させる。これは`-mcmodel=kernel`の制約（モジュールの絶対シンボル参照は、アドレス空間の最上位2 GBに収まらなければならない）であり、±2 GBのPC相対ウィンドウと同じものではない。

### 10. 形成を完了し、COMING通知を発火する {#10-complete-formation-and-fire-the-coming-notifiers}

`complete_formation()`は、モジュールを他のCPUから観測しても安全な状態にする作業を行う。`module_mutex`の下で、`verify_exported_symbols()`によりエクスポートの重複をチェックし、BUGテーブルとCFIテーブルを確定させ、ステップ5で割り当てたメモリクラスにページ権限を適用し（`module_enable_rodata_ro()`、`module_enable_data_nx()`、`module_enable_text_rox()`）、そうして初めて状態の変更を公開する:

```c
/*
 * 状態を coming にする。strong_try_module_get() はこのモジュールを無視するが、
 * kallsyms などからは見える。
 */
mod->state = MODULE_STATE_COMING;
```

通知はその後、`prepare_coming_module()`から発火する:

```c
err = blocking_notifier_call_chain_robust(&module_notify_list,
                MODULE_STATE_COMING, MODULE_STATE_GOING, mod);
err = notifier_to_errno(err);
```

重要な点は`_robust`版であることだ。チェーン内のいずれかのコールバックが失敗すると、すでに成功したコールバックに対して`MODULE_STATE_GOING`でチェーンを自動的に再実行するので、モジュールにパッチを当てたサブシステムはその作業を元に戻せる。kprobeなどのサブシステムは、新しいモジュールのコードを計装するためにこのチェーンに登録する（`kprobe_register_module_notifier()`）。ftraceとライブパッチは代わりに直接組み込まれている — 同じ理由で、`ftrace_module_enable()`と`klp_module_coming()`がチェーンの直前に実行される。

### 11. mod->init()を呼び出す {#11-call-mod-init}

`load_module()`は`return do_init_module(mod);`で終わり、`do_init_module()`は`do_one_initcall()`を通してモジュールのinit関数を呼び出す — 組み込みのinitcallに使われるのとまったく同じヘルパーである。`initcall_blacklist=`ブートパラメータがモジュールのinit関数にも効くのはこのためである。`initcall_blacklisted()`は`sprint_symbol_no_offset()`で関数ポインタを解決し、比較の前に`[module_name]`の接尾辞を取り除く。`do_one_initcall()`を経由することで、モジュールのinitにはinitcallのtracepointと、プリエンプションや割り込みの状態を不均衡なまま残した場合の`WARN`も適用される。

```c
do_mod_ctors(mod);
/* モジュールを開始する */
if (mod->init != NULL)
        ret = do_one_initcall(mod->init);
```

**成功した場合**（`ret == 0`）、モジュールは`MODULE_STATE_LIVE`になり、`MODULE_STATE_LIVE`の通知が発火し、`KOBJ_ADD` ueventが発行され、初期参照が`module_put()`で落とされ、3つのinitメモリクラスが`mod->mem[]`から切り離される。正の戻り値は受け入れられるが、警告が出る（`init suspiciously returned %d`）。

initメモリはその場では解放*されない*。ベースアドレスがロックレスリストに積まれ、workqueueのアイテムがスケジュールされる:

```c
if (llist_add(&freeinit->node, &init_free_list))
        schedule_work(&init_free_wq);
```

その後`do_free_init()`がそのworkqueueから実行され、RCUのグレースピリオドが過ぎるのを待ち（kallsymsがRCUの読み取り側クリティカルセクション内でinitシンボルを走査している可能性がある）、それから初めて各領域に対して`execmem_free()`を呼び出す。`module_memfree()`は`module_alloc()`とともになくなった。

```c
static void do_free_init(struct work_struct *w)
{
        ...
        synchronize_rcu();

        llist_for_each_safe(pos, n, list) {
                initfree = container_of(pos, struct mod_initfree, node);
                execmem_free(initfree->init_text);
                execmem_free(initfree->init_data);
                execmem_free(initfree->init_rodata);
                kfree(initfree);
        }
}
```

**失敗した場合**（負の戻り値）、モジュールはexit関数を一度も実行されることなく破棄される — `mod->exit()`は`rmmod`専用であり、initが失敗したモジュールは何もセットアップを終えていないとみなされる:

```c
fail:
        /* バグのある参照カウンタから身を守ることを試みる。 */
        mod->state = MODULE_STATE_GOING;
        synchronize_rcu();
        module_put(mod);
        blocking_notifier_call_chain(&module_notify_list,
                                     MODULE_STATE_GOING, mod);
        klp_module_going(mod);
        ftrace_release_mod(mod);
        free_module(mod);
```

戻る途中で1つのerrnoが書き換えられる。`EEXIST`は、この名前のモジュールがすでにロードされていることをユーザー空間に伝えるために`init_module()`/`finit_module()`が予約しているので、自身のinitが`-EEXIST`を返したモジュールはそれを付け替えられる:

```c
if (ret == -EEXIST)
        ret = -EBUSY;
```

## struct module {#struct-module}

`struct module`（`include/linux/module.h`で定義）は、ロードされたモジュールのカーネル内での実行時表現である。主なフィールド:

```c
struct module {
    enum module_state    state;         /* 現在のライフサイクル状態 */
    struct list_head     list;          /* グローバルな modules リストにつながれる */
    char                 name[MODULE_NAME_LEN];

    /* エクスポートシンボル — テーブル1つと、エントリごとに並列するフラグバイト */
    const struct kernel_symbol *syms;
    const u32                  *crcs;
    const u8                   *flagstab;
    unsigned int                num_syms;

    /* このモジュールが GPL 専用のエクスポートのインポートに成功したときにセットされる */
    bool using_gplonly_symbols;

    /* 開始関数 */
    int (*init)(void);

    /* メモリ領域 — クラスごとに1つ、独立に割り当てられる */
    struct module_memory mem[MOD_MEM_NUM_TYPES] __module_memory_align;

#ifdef CONFIG_MODULE_UNLOAD
    /* どのモジュールが自分に依存しているか? */
    struct list_head source_list;
    /* 自分はどのモジュールに依存しているか? */
    struct list_head target_list;

    /* 破棄関数。 */
    void (*exit)(void);

    atomic_t refcnt;
#endif
};
```

2つの依存リストは、`struct module_use`のエッジの視点から命名されている。このエッジは`source`（インポートする側のモジュール）と`target`（エクスポートする側のモジュール）を記録する。`add_module_usage(a, b)` — `a`が`b`からインポートする — は、エッジを`b->source_list`と`a->target_list`につなぐ。したがって:

- **`source_list`**は、`target`がこのモジュールであるエッジを保持する。つまり**自分に依存しているもの**である。これは`lsmod`が「Used by」列に表示するリストである。
- **`target_list`**は、`source`がこのモジュールであるエッジを保持する。つまり**自分が依存しているもの**である。

これらが合わさって`rmmod`が走査する依存グラフを形成する。モジュールを削除できるのは、その`source_list`が空で、`refcnt`が尽きたときだけである。

アンロードの仕組み — 両方のリスト、`exit`、参照カウント — は`CONFIG_MODULE_UNLOAD`の下でのみ存在することに注意。これなしでビルドするとモジュールはロード専用になる。`print_unload_info()`が、参照カウントと依存リストの代わりに2つのダッシュをそのまま出力するスタブを持っているのはこのためである。

参照カウント自体は単純な`atomic_t refcnt`である。それが置き換えたper-CPUの`struct module_ref __percpu *refptr`は、v3.19以降存在しない。

## MODULE_STATE_*のライフサイクル {#module_state_-lifecycle}

```
insmod / modprobe
      │
      ▼
 UNFORMED    add_unformed_module(): modules リスト内に名前が予約されるが、
      │      find_symbol() と /proc/modules はどちらもこれをスキップする
      ▼
  COMING     complete_formation() がページ権限を封印して状態をセットし、
      │      続いて prepare_coming_module() が ftrace/klp のセットアップと
      │      COMING 通知チェーンを実行する
      ▼
 do_init_module() → do_one_initcall(mod->init)
      │
      ├── 0 を返す ───────────────► LIVE
      │                              │   init メモリは workqueue に送られる。
      │                              │   do_free_init() が RCU のグレース
      │                              │   ピリオドを待ってから execmem_free() する
      │                              │
      │                            rmmod   (空の source_list と尽きた refcnt が
      │                              │       必要。mod->exit() を実行する)
      │                              ▼
      └── 0 未満を返す ─────────────► GOING
                                     │   GOING 通知が発火する。このパスでは
                                     │   mod->exit() は呼ばれない — init に失敗した
                                     │   モジュールの exit 関数は呼ばれない
                                     ▼
                                  free_module() → mem[] クラスごとに execmem_free()
```

`UNFORMED`と`COMING`の状態はどちらもグローバルなmodulesリストを通して他のCPUから見え、どちらもそこで特別扱いされる。`find_symbol()`は`UNFORMED`のモジュールを完全にスキップし、`m_show()`はそれらを`/proc/modules`から除外し、`strong_try_module_get()`は`COMING`のモジュールを`-EBUSY`で拒否する。これにより、依存する側のロードは作りかけの提供側に結び付くのではなく待機する。（`GOING`は`find_symbol()`でそのような特別扱いを受けない — モジュールはアンロード中でもシンボルをエクスポートできる — が、`m_show()`は`Unloading`としてそれを表示する。）モジュールが`GOING`に達した時点で、どのスレッドもまだモジュールの*中*を実行していないことを保証するのは、RCUではなく参照カウントである。`try_module_get()`/`module_put()`（および`__symbol_get()`の背後にある`strong_try_module_get()`）がカウントを保持し、`delete_module()`は、呼び出し元が強制しない限り、生きたカウントが残っている状態で`GOING`をセットしない（`try_force_unload()`が成功すると、`try_stop_module()`はカウントに関係なく`GOING`をセットする。これは`CONFIG_MODULE_FORCE_UNLOAD`と`delete_module(2)`への`O_TRUNC`フラグによって制限される）— そうでなければ`try_release_module_ref()`がベース参照を戻し、`try_stop_module()`は`-EWOULDBLOCK`を返して、`rmmod`は即座に失敗する。（カーネルは、v3.13のコミット`3f2b9c9cdf38`「module: remove rmmod --wait option.」以降、ブロックして参照カウントが尽きるのを待つことはしていない。）破棄パス上の`synchronize_rcu()`呼び出しは別の役割を果たす。`free_module()`はモジュールを`_rcu`版を使ってmodulesリスト、modツリー、bugリストからリンク解除し — 「Unlink carefully: kallsyms could be walking list」（慎重にリンク解除せよ: kallsymsがリストを走査している可能性がある） — それからグレースピリオドを待つので、それらのRCU読み取り側は解放中のメモリに触れることができない。

## /proc/modulesの形式 {#procmodules-format}

```bash
cat /proc/modules
# e1000e 262144 0 - Live 0xffffffffc0400000
# ^^^^^^ ^^^^^^ ^ ^ ^^^^ ^^^^^^^^^^^^^^^^^^
# name   size   | | |    load address
#               | | state (Live/Loading/Unloading)
#               | dependents ("Used by" in lsmod)
#               refcount
```

形式の権威は`kernel/module/procfs.c`の`m_show()`である。各フィールド:

- **size**: `module_total_size(mod)`。これはコアのものだけでなく、**7つすべての**メモリクラスにわたって`mod->mem[type].size`を合計する。そのため、モジュールのinitメモリが解放される前に`/proc/modules`を読むと、解放後に読んだときよりわずかに大きな数値が報告される。
- **refcount**: `module_refcount(mod)` — 保持されている参照の数、または`try_stop_module()`が`MODULE_REF_BASE`を落とした後、つまり`Unloading`と表示されるモジュールでは`-1`。（`module_refcount()`自身のkernel-docには「return the refcount or -1 if unloading」とある。このウィンドウは実在する。`delete_module()`は`GOING`をセットした後、モジュールがまだmodulesリストに載っている間に`mod->exit()`を実行し、`m_show()`は`UNFORMED`しかスキップしないからである。）`CONFIG_MODULE_UNLOAD=n`では参照カウントも依存関係の追跡もまったくなく、`print_unload_info()`のスタブは、このフィールド*と*次のフィールドの代わりに2つのダッシュ（`- -`）をそのまま出力する。
- **dependents**: このモジュールが依存しているモジュールではなく、*この*モジュールに依存しているモジュール。`print_unload_info()`は`mod->source_list`を走査し、それぞれについて末尾にカンマを付けて`use->source->name`を出力する — `lsmod`が「Used by」の下に表示するのと同じ集合である。init関数を持つがexit関数を持たないモジュールには、さらに`[permanent],`が付加される。これは、強制削除でない限りそのモジュールのアンロードを拒否する`delete_module()`内の`if (mod->init && !mod->exit)`テストに対応する。何も出力されなかった場合は、`-`が1つ出力される。
- **state**: `Live`、`Loading`（`MODULE_STATE_COMING`）、`Unloading`（`MODULE_STATE_GOING`）のいずれか。`MODULE_STATE_UNFORMED`のモジュールは完全にスキップされる。
- **load address**: `mod->mem[MOD_TEXT].base` — 何らかの結合された「コアレイアウト」のベースではなく、モジュールのテキスト領域のベース。`kallsyms_show_value()`を満たさない読み手には、代わりに`0x0000000000000000`が見える。

`mod->taints`が非ゼロの場合は、括弧付きのフラグ接尾辞が続く。たとえば、ツリー外の署名なしモジュールなら`(OE)`であり、モジュールのロード中は`+`、アンロード中は`-`が付加される。

## シンボルバージョニングの詳細 {#symbol-versioning-detail}

CRCの生成器は2つある。`scripts/genksyms/`がオリジナルであり、カーネルのビルド時に実行され、プリプロセス済みのCを解析し、各エクスポートシンボルの完全な型シグネチャ（構造体と共用体のレイアウトを再帰的に展開したもの）に対してCRCを計算する。そのCパーサはRustを見ることができず、型がデバッグ情報でしか完全に記述されないシンボルをバージョン付けすることもできない。そこで`CONFIG_GENDWARFKSYMS`の下で`scripts/gendwarfksyms/`がそれと並んで追加された。こちらはソースではなくDWARFから同種のCRCを導出する。

いずれにせよ結果は`Module.symvers`に入り、`modpost`が`write_dump()`からそれを書き出す:

```c
buf_printf(&buf, "0x%08x\t%s\t%s\tEXPORT_SYMBOL%s\t%s\n",
           sym->crc, sym->name, mod->name,
           sym->is_gpl_only ? "_GPL" : "",
           sym->namespace);
```

タブ区切りの5つのフィールドは、CRC、シンボル名、提供するモジュール（組み込みなら`vmlinux`）、エクスポートの種類、名前空間である。名前空間なしでエクスポートされたシンボルでは名前空間フィールドは空文字列なので、名前空間のない行は単にタブで終わる:

```bash
grep -w module_layout Module.symvers | cat -A
# 0x1a2b3c4d^Imodule_layout^Ivmlinux^IEXPORT_SYMBOL^I$
```

（`cat -A`はタブを`^I`、行末を`$`として表示する。CRC自体はビルドに固有の値である。）

ツリー外モジュールをビルドする際、`make`は`$(KDIR)`から`Module.symvers`を読み込み、正しいCRCをモジュールの`__versions`（または`__version_ext_crcs`/`__version_ext_names`）セクションに埋め込む。誤った`Module.symvers`に対してビルドすることが、バージョン不一致エラーの最も一般的な原因である。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/module/main.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/main.c) — ローダーの中核: `load_module()`、`simplify_symbols()`、`apply_relocations()`、`do_init_module()`、および`init_module`/`finit_module`の`SYSCALL_DEFINE`
- [kernel/module/internal.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/internal.h) — `struct load_info`。ロードのすべての段階を通して受け渡されるELFセクションのキャッシュ
- [kernel/module/version.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/version.c) — `check_version()`と`same_magic()`: `__versions`のCRC比較とvermagicチェック、および長いシンボル名向けの新しい拡張modversionsのパス
- [kernel/module/kallsyms.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/kallsyms.c) — `layout_symtab()`と`add_kallsyms()`: モジュール自身のシンボルテーブルが、割り当てられたメモリにどのように運ばれ、`/proc/kallsyms`を通して公開されるか
- [kernel/module/procfs.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/procfs.c) — `m_show()`: `/proc/modules`のフィールド順序、`Live`/`Loading`/`Unloading`の文字列、`[permanent]`マーカーに関する権威あるソース
- [include/linux/module.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/module.h) — `struct module`、`enum module_state`、`struct modversion_info`、および割り当てられた各セクションクラスを保持する`struct module_memory mem[]`配列
- [include/linux/export.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/export.h) — `EXPORT_SYMBOL()`/`EXPORT_SYMBOL_GPL()`: modpostが`__ksymtab`に変換する`.export_symbol`エントリ
- [include/uapi/linux/module.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/module.h) — `finit_module()`のフラグの完全な集合: `MODULE_INIT_IGNORE_MODVERSIONS`、`MODULE_INIT_IGNORE_VERMAGIC`、`MODULE_INIT_COMPRESSED_FILE`
- [arch/x86/kernel/module.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/module.c) — `apply_relocate_add()`: 各再配置を計算して書き込む、x86-64の`R_X86_64_*`のswitch
- [scripts/genksyms/](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/genksyms) — `Module.symvers`に記録される値を生成する、`CONFIG_MODVERSIONS`のCRC生成器
- [`ac3b43283923`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ac3b43283923440900b4f36ca5f9f0b1ca43b70e) — 「module: replace module_layout with module_memory」（Song Liu, 2023）: 古いcore/initの2領域レイアウトを、クラスごとの`mod->mem[]`配列に分割したコミット
- [`12af2b83d0b1`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=12af2b83d0b17ec8b379b721dd4a8fbcd5d791f3) — 「mm: introduce execmem_alloc() and execmem_free()」（Mike Rapoport, 2024）: 現在モジュールのテキストとデータが切り出されるアロケータ
- [`55fcb926b6d8`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=55fcb926b6d8b5cfb40873e4840a69961db1bb69) — 「module: use kflagstab instead of *_gpl sections」（Siddharth Nayyar, 2026）: GPL専用が並列する`__kflagstab`セクションの1ビットになり、ローダーは独立した`__ksymtab_gpl`を検索しなくなった

### manページ {#man-pages}

- [`init_module(2)`](https://man7.org/linux/man-pages/man2/init_module.2.html) — 両方のシステムコールを1ページで説明している: プロトタイプ、`finit_module()`の3つのフラグ、そして`finit_module()`がLinux 3.8で登場したこと
- [`delete_module(2)`](https://man7.org/linux/man-pages/man2/delete_module.2.html) — アンロード側。他のロード済みモジュールがこのモジュールで定義されたシンボルを参照している場合に呼び出しが`EWOULDBLOCK`で失敗すること、および`O_NONBLOCK`/`O_TRUNC`が何を変えるかを述べている
- [`lsmod(8)`](https://man7.org/linux/man-pages/man8/lsmod.8.html) — 「a trivial program which nicely formats the contents of the `/proc/modules`」（`/proc/modules`の内容をきれいに整形する単純なプログラム）。上で分解したファイルのユーザー空間からの見え方
- [`depmod(8)`](https://man7.org/linux/man-pages/man8/depmod.8.html) — 「what symbols each module exports and needs」（各モジュールがどのシンボルをエクスポートし、必要とするか）を判定し、結果のグラフを`modules.dep`に書き出す。依存関係順のロードの入力となる
- [`modprobe(8)`](https://man7.org/linux/man-pages/man8/modprobe.8.html) — `modules.dep`を参照するツール。`modules.dep`は「lists what other modules each module needs (if any), and modprobe uses this to add or remove these dependencies automatically」（各モジュールが必要とする他のモジュールを（あれば）列挙し、modprobeはこれを使ってそれらの依存関係を自動的に追加または削除する）。manページは最終的に発行するシステムコールを明記していないが、kmodの実装は各`.ko`を`finit_module()`でロードする

### 関連ページ {#related-pages}

- [モジュールパラメータ、シンボル、Kconfig](module-params.md) — `EXPORT_SYMBOL`/`EXPORT_SYMBOL_GPL`、`has no CRC!`と`disagrees about version`の症状、`/proc/kallsyms`をモジュール作成者の側から
- [カーネルモジュールの署名](module-signing.md) — `load_module()`がELFセクションを1つも見る前に実行する`module_sig_check()`
- [Kbuild: カーネルビルドシステム](kbuild.md) — ツリー外ビルドとクロスコンパイル。このウォークスルーがロードする`.ko`を生成するビルド側
- [カーネルモジュールの作成とロード](module-basics.md) — ソースレベルの視点: `module_init`/`module_exit`、`__init`セクション、そしてこのページが分解する`.ko`
- [モジュールの事件簿](war-stories.md) — 本番環境で見られたCRCの不一致、taintの連鎖、バージョニングの意外な落とし穴

### LWNの記事 {#lwn-articles}

- [Loading modules from file descriptors](https://lwn.net/Articles/519010/) — Michael Kerrisk、2012年10月。`finit_module()`となった提案について。主な動機は、モジュールのバイト列だけでなく、その*ファイルシステム上の出自*（dm-verityで裏付けられたルート、拡張属性から署名を読むLSM）をカーネルが判断できるようにすることだった
- [The return of modversions](https://lwn.net/Articles/21393/) — 2003年1月。上のステップ8が実装している仕組みを説明している。インポートされるシンボルごとに「the symbol name and its checksum」（シンボル名とそのチェックサム）を保持する構造体が1つあり、特別なセクションにリンクされ、ロードが成功すると破棄される
- [A new version of modversions](https://lwn.net/Articles/986892/) — 2024年8月。`genksyms`のCパーサがRustのコードをバージョン付けできない理由と、ソースではなくDWARFからCRCを導出する代替の`gendwarfksyms`
- [Yet another memory allocator for executable code](https://lwn.net/Articles/933867/) — 2023年6月。実行可能メモリの割り当てをモジュールローダーから切り離すべきだというMike Rapoportの主張と、その動機となったダイレクトマップの断片化
- [Two approaches to tightening restrictions on loadable modules](https://lwn.net/Articles/998221/) — 2024年11月。記事の時点での2テーブル（`__ksymtab`/`__ksymtab_gpl`）モデルにおいて、ローダーのGPL専用の強制がどう機能していたか（v7.1で、上で説明した単一テーブルの`__kflagstab`方式に置き換えられた）、およびエクスポートを名前付きのモジュール集合に制限する提案

### 外部資料 {#external}

- [Building External Modules](https://docs.kernel.org/kbuild/modules.html) — ツリー外ビルドに関するアップストリームのガイド。「Symbols From the Kernel (vmlinux + modules)」や、モジュール間の`Module.symvers`のための抜け道である`KBUILD_EXTRA_SYMBOLS`を含む
- [Tainted kernels](https://docs.kernel.org/admin-guide/tainted-kernels.html) — 権威あるtaintフラグの表。`F`（「module was force loaded」、モジュールが強制ロードされた）、`O`（ツリー外モジュール）、`E`（署名なしモジュール）を含む
