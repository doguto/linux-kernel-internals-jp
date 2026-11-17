# Dynamic Debug {#dynamic-debug}

> pr_debugとdev_dbg: 再コンパイルなしに実行時にカーネルのデバッグメッセージを有効にする

## pr_debugの問題点 {#the-problem-with-pr_debug}

`pr_debug()` は2通りの方法で制御される。

- `CONFIG_DYNAMIC_DEBUG=y` の場合: 各 `pr_debug()` は動的に制御可能なサイトになる（デフォルトではNOPで、実行時に有効化される）。
- `CONFIG_DYNAMIC_DEBUG` がない場合: `pr_debug()` は、そのコンパイル単位に対して `DEBUG` というCプリプロセッサマクロが定義されていれば（Makefileの `CFLAGS_file.o += -DDEBUG` によって）`printk(KERN_DEBUG ...)` に展開され、`DEBUG` が定義されていなければ `no_printk()`（コンパイル時のno-op）に展開される。

`CONFIG_DYNAMIC_DEBUG` がない場合、これはオールオアナッシングのコンパイル時スイッチであり、あらゆるサブシステムからのメッセージが同時にログにあふれることになる。

`CONFIG_DYNAMIC_DEBUG` は、これを呼び出しサイト単位のオン/オフスイッチに置き換え、debugfsを通じて実行時に制御できるようにする。無効時のコストはNOP命令1つだけである。特定のドライバや関数のデバッグメッセージを有効にするために、再コンパイルや再起動をする必要はない。

関係するKconfigシンボルは2つある。`CONFIG_DYNAMIC_DEBUG=y` はこの機能をカーネル全体で有効にする: `CONFIG_DYNAMIC_DEBUG_CORE` をselectする*とともに*、ビルド内のすべての `pr_debug()`/`dev_dbg()` 呼び出しサイトをカタログ化する（これにはカーネルテキストのおよそ2%のコストがかかる）。`CONFIG_DYNAMIC_DEBUG_CORE=y` 単独では仕組み — 制御ファイルとクエリエンジン — だけがビルドされ、何もカタログ化されない。個々のモジュールは、Makefileに `ccflags-y += -DDYNAMIC_DEBUG_MODULE` を追加することでオプトインする。`include/linux/dynamic_debug.h` のマクロは `CONFIG_DYNAMIC_DEBUG || (CONFIG_DYNAMIC_DEBUG_CORE && DYNAMIC_DEBUG_MODULE)` を条件としている。この分割は、組み込みシステムのようにサイズに敏感なビルドのために存在する。少数のドライバにはdynamic debugが欲しいが、完全なカタログを持つ余裕はない場合である。

## 仕組み {#how-it-works}

カーネル内のすべての `pr_debug()` と `dev_dbg()` の呼び出しサイトについて、コンパイラは次のものを出力する。

1. オブジェクトファイルの `__dyndbg` セクション内の `static struct _ddebug` ディスクリプタ。その呼び出しサイトに関するメタデータを含む。
2. jump label（static key）基盤による、`.text` 内の呼び出しサイトのNOP命令。

dynamic debugはjump label（static key）基盤（`CONFIG_JUMP_LABEL`）を使う。各 `pr_debug()` の呼び出しサイトには、その `_ddebug` ディスクリプタにstatic keyが埋め込まれている — 後述の `key` 共用体である。無効時、そのサイトはNOPである（分岐しない `jump_label` による）。有効化されると、`static_branch_enable()` / `jump_label_update()` がNOPを短いジャンプに書き換える。書き換えはjump labelサブシステムが行う — dynamic debug自身は `dp->key.dd_key_true` に対して `static_branch_enable()` または `static_branch_disable()` を呼ぶだけである。無効時、呼び出しサイトは実行時オーバーヘッドゼロのNOPである。

`key` がそもそも共用体である理由は2つの共用体メンバーにあり、制御経路と呼び出しサイトはこれを異なる方法で読む。`ddebug_change()` は常に `dd_key_true` を切り替える。一方、呼び出しサイトは `DYNAMIC_DEBUG_BRANCH()` を展開し、これは翻訳単位が `DEBUG` を定義してコンパイルされたかどうかでメンバーを選ぶ。定義されていない場合 — 通常のケース — 分岐は `static_branch_unlikely(&descriptor.key.dd_key_false)` で初期状態はオフとなり、コンパイラはデバッグ呼び出しをunlikelyな経路に配置する。`-DDEBUG` の場合は `static_branch_likely(&descriptor.key.dd_key_true)` で初期状態はオンとなり、サイトは出力する状態から始まり、ログを出す側が高速パスになる。両メンバーは同じ下位のキーを別名で参照しているので、制御ファイルはどちらのビルドも同じように切り替える。

## struct _ddebug {#struct-_ddebug}

`include/linux/dynamic_debug.h` で定義されている。

```c
struct _ddebug {
    const char *modname;    /* モジュール名 */
    const char *function;   /* 囲んでいる関数の名前 */
    const char *filename;   /* ソースファイルのパス */
    const char *format;     /* フォーマット文字列リテラル */
    unsigned int lineno:18; /* ソースの行番号 */
    unsigned int class_id:6;
    unsigned int flags:8;   /* _DPRINTK_FLAGS_* のビットマスク */
#ifdef CONFIG_JUMP_LABEL
    union {
        struct static_key_true dd_key_true;
        struct static_key_false dd_key_false;
    } key;
#endif
} __attribute__((aligned(8)));
```

`flags` フィールドは、このサイトでどの出力装飾が有効になっているか（出力、行番号、ファイル、モジュール、スレッド情報など）を記録する。

ディスクリプタは1つのフラットなグローバル配列として検索されるわけではない。ロード可能な各モジュールは自身の `__dyndbg` セクションを持ち、`lib/dynamic_debug.c` はそれを `struct ddebug_table` としてグローバルな `ddebug_tables` 連結リストに登録する。

```c
/* lib/dynamic_debug.c */
struct ddebug_table {
    struct list_head link, maps;
    const char *mod_name;
    unsigned int num_ddebugs;
    struct _ddebug *ddebugs;
};

static DEFINE_MUTEX(ddebug_lock);
static LIST_HEAD(ddebug_tables);
```

組み込みコードは、セクションとテーブルが1対1でなくなるケースである。vmlinuxのリンカスクリプトは、組み込みの呼び出しサイトをすべて*単一*の境界付き `__dyndbg` セクションに集めるので、いくつの組み込みモジュールが寄与していてもイメージ全体でセクションは1つである。その後 `dynamic_debug_init()` がこのセクションを `__start___dyndbg` から `__stop___dyndbg` まで走査し、多数のテーブルに切り分ける。ディスクリプタは翻訳単位ごとにまとまって並んでいるので、`iter->modname` が変わるのを監視し、同じ名前のエントリの連続ごとに `ddebug_add_module()` を呼ぶだけでよい。入力はセクション1つ、出力は異なる `modname` ごとに1つの `ddebug_table` — ロード可能モジュールがnotifierを通じて登録するモジュールごとのテーブルと対応する。

したがって、制御コマンドは2重のループで照合される。`ddebug_change()` は `ddebug_lock` を取得して `ddebug_tables` を走査する。テーブルごとに、呼び出しサイトを見る前に2つの安価な除外判定を行う: テーブルの `mod_name` をクエリの `module` 指定と照合し — これにより1回の比較でモジュール全体のディスクリプタを飛ばせる — さらにクエリに `class` 指定があれば、`ddebug_find_valid_class()` がテーブルの `ddebug_class_map` の `maps` リストを検索してそのクラス*名*を数値の `class_id` に解決し、このモジュールがそのクラスを宣言していなければテーブル全体を飛ばす。その後 `dt->ddebugs[]` 配列を反復し、呼び出しサイトごとに `class_id`、`filename`、`function`、`format`、`lineno` を判定する。

その内側のループではクラスの判定が最初に来て、しかもこれは省略できない: クエリに `class` 指定がない場合、`valid_class` はデフォルトで `_DPRINTK_CLASS_DFLT` となり、`class_id` がそれと異なるサイトはすべて飛ばされる。そのため、通常のクエリはクラス付きの呼び出しサイトには一切触れない — 後述の `module mymodule +p` の形式も含め、これはそのモジュールのクラスなしのサイトだけを有効にする。サブシステムのクラス付きサイト（たとえばDRMのデバッグカテゴリ）に届かせるには、常にクラスを明示的に指定する必要がある。

## 制御インターフェース {#the-control-interface}

制御ファイルは `/sys/kernel/debug/dynamic_debug/control` にある。これにクエリ文字列を書き込むと、一致する呼び出しサイトが有効または無効になる。

v5.7以降、同じ制御ファイルが `/proc/dynamic_debug/control` にも公開されているので、debugfsなしでビルドされたシステムでもdynamic debugを使い続けられる。両者は交換可能である。以下の例ではdebugfsのパスを使うが、アップストリームのドキュメントは現在procfsのほうを先に挙げている。

```bash
# モジュールのクラスなしのpr_debug()呼び出しを有効にする
# （クラス付きのサイトには明示的な "class" 指定が必要 — 上記参照）
echo "module mymodule +p" > /sys/kernel/debug/dynamic_debug/control

# ソースファイルで有効にする（カーネルルートからの相対パス）
echo "file drivers/net/ethernet/intel/e1000/e1000_main.c +p" \
    > /sys/kernel/debug/dynamic_debug/control

# ファイル内の行範囲を有効にする
echo "file drivers/net/ethernet/intel/e1000/e1000_main.c line 100-200 +p" \
    > /sys/kernel/debug/dynamic_debug/control

# 関数名で有効にする
echo "func tcp_recvmsg +p" > /sys/kernel/debug/dynamic_debug/control

# 追加の出力装飾付きで有効にする
# +p  メッセージを出力する
# +f  関数名を前置する
# +s  ソースファイル名を前置する
# +l  行番号を前置する
# +m  モジュール名を前置する
# +t  スレッドID（または<intr>）を前置する
echo "module mymodule +pmfl" > /sys/kernel/debug/dynamic_debug/control

# モジュール内のすべての呼び出しサイトを無効にする
echo "module mymodule -p" > /sys/kernel/debug/dynamic_debug/control

# 現在の状態を表示する — 登録された呼び出しサイトごとに1行
cat /sys/kernel/debug/dynamic_debug/control | grep mymodule

# 現在有効な（出力する）サイトだけを表示する
grep "=p" /sys/kernel/debug/dynamic_debug/control
```

### クエリ構文 {#query-syntax}

制御文字列は次の形式を持つ。

```
[match-spec ...] flags-spec
```

ここで `match-spec` は次のいずれかである。

- `module <name>` — モジュール名で一致させる。`lsmod` に表示される名前（ディレクトリなし、`.ko` なし、`-` は `_` に変換）を使う
- `file <path>` — ソースファイルで一致させる。値はソースルートからの相対パス名*または*ベース名と比較されるので、`file kernel/freezer.c` と `file svcsock.c` のどちらも動作する。2つの末尾形式も受け付ける: `file inode.c:start_*` は `:tail` を `func` 指定として解析し、`file inode.c:1-100` はそれを `line` の範囲として解析する。
- `func <name>` — 関数名で一致させる
- `line <N>` または `line <N>-<M>` — 行番号または範囲で一致させる。どちらの端も省略できる: `line -1605` は1行目から1605行目まで、`line 1600-` は1600行目からファイル末尾までを意味する。範囲には空白を含めてはならない — `1-30` は有効だが、`1 - 30` は無効である。
- `format <string>` — フォーマット文字列がこの部分文字列を含む場合に一致する。先頭の `^` は一致をフォーマットの先頭に固定する。空白はCの8進表記でエスケープする（`format nfsd:\040SETATTR`）か、値全体を引用符で囲む（`format "nfsd: SETATTR"`）。
- `class <name>` — モジュールが（`ddebug_class_map` によって）宣言したクラスに属する呼び出しサイトだけに一致する。クラスを使うと、サブシステムは呼び出しサイトを名前付きのカテゴリにまとめ、一括で切り替えられるようになる — `class DRM_UT_KMS` のようなクエリで一致させるDRMのデバッグカテゴリが主な利用者である。上の `struct _ddebug` の `class_id` ビットフィールドが選択するのがこれである。未知のクラス名は何も言わずに不一致となり、クラス名はワイルドカードを受け付け*ない*。

複数の一致指定はANDで結合され、キーワードがない場合は `*` と同様に振る舞う。一致指定がまったくないクエリも合法なので、フラグが先に解析される — そのため、不正なフラグ文字があると、エラーメッセージ上で不正なキーワードが隠れてしまう。

ワイルドカードは全面的に使えるわけではない。`match_wildcard()` を通るのは `module`、`file`、`func` の指定なので、これらはグロブを受け付ける: `*` は0文字以上、`?` はちょうど1文字に一致する。`module drm*` は `drm` と `drm_kms_helper` の両方に一致し、`file "drivers/usb/*"` はそのディレクトリ以下のすべてに一致する（シェルが先に展開しないよう引用符で囲むこと）。他の3つの指定はグロブを使わない: `format` は常にリテラルの部分文字列検索（先頭の `^` で固定することは可能）なので、その中の `*` と `?` は普通の文字である。`line` は数値と範囲だけを受け付け、ワイルドカードは不正な行番号として拒否される。`class` の名前は完全一致で照合される。

1回の `write()` で、`;` または改行で区切った複数のクエリを渡すことができ、左から右へ適用される。

```bash
# すべてをオフにしてから、mainモジュールのrun*関数だけを有効にする
echo '-p; module main func run* +p' > /sys/kernel/debug/dynamic_debug/control

# あるいはファイルから、1行に1クエリずつ
cat query-batch-file > /sys/kernel/debug/dynamic_debug/control
```

複数クエリの文字列の中で、`#` で始まるクエリはコメントとして扱われ、読み飛ばされる。

`flags-spec` は、`+`（追加）、`-`（削除）、`=`（そのとおりに設定）のいずれかの後に1つ以上のフラグ文字が続くものである。

### 出力フラグ文字 {#output-flag-letters}

| フラグ | 意味 |
|------|---------|
| `p` | 出力を有効にする（出力を見るには必須） |
| `t` | 出力にスレッドIDを前置する。割り込みコンテキストでは `<intr>` |
| `m` | 出力にモジュール名を前置する |
| `f` | 出力に関数名を前置する |
| `s` | 出力にソースファイル名を前置する |
| `l` | 出力に行番号を前置する |
| `d` | コールトレースを含める |
| `_` | フラグなし（すべてをクリアするには `=_` を使う） |

装飾フラグ（`t`、`m`、`f`、`s`、`l`、`d`）は、クエリで入力した順序にかかわらず、
この固定の順序でメッセージの接頭辞に追加される。`d` は実は接頭辞の装飾ではない — メッセージの後に `dump_stack()` の呼び出しを付け加えるもので、`__dynamic_emit_prefix()` とは別に処理される。

## ブート時の有効化 {#boot-time-enablement}

一部のデバッグメッセージは、debugfsがマウントされる前に必要になる。ブート時の仕組みは2つある。

```bash
# カーネルコマンドライン — クエリに一致するすべての呼び出しサイトに適用される
dyndbg="module mymodule +p"

# モジュールごとのパラメータ — ロード時にそのモジュールだけに適用される
# カーネルコマンドライン、/etc/modprobe.d/*.confの
# "options" 行、またはmodprobeの引数として直接渡す
mymodule.dyndbg="+p"
```

2つの形式は処理のされ方が異なり、その違いは重要である。

- 単独の `dyndbg="QUERY"` は、組み込みのddebugテーブルが登録された直後に実行される `early_initcall` から、**1回だけ**処理される。そのため、そのearly initcallより後に実行されるコード — ACPIのセットアップ、PCIの列挙、その他 `subsys_initcall` 以降の処理 — には届くが、二度と再処理されることはないので、後でロードされるモジュールには影響できない。
- `<module>.dyndbg="QUERY"` は、モジュールのロード時に再処理*される*。指定されたモジュールが組み込みであれば、ブート時の処理がそれを適用する。組み込みでなければ、ブート時の処理はそれを見ても何もせず、クエリはモジュールのロード時に改めて — 今度は実際に — 適用される。

ブート時のクエリは1023文字を超えてはならず、ブートローダが独自にそれより低い上限を課すこともある。`<module>.dyndbg=` の形式では、クエリ自体に `module` の一致指定を含めてはならない: モジュール名はパラメータ名から取られ、文字列内の各クエリに適用される。

再起動をまたいで永続化するには、`/etc/modprobe.d/*.conf` ファイルにモジュールオプションとして設定を書けばよい。

```
options mymodule dyndbg=+pt
```

複数の場所が同じモジュールの `dyndbg` を設定している場合、modprobeはそれらを順に適用する — まず `/etc/modprobe.d/*.conf`、次にブートコマンドラインの `<module>.dyndbg`、最後に `modprobe` 自体に渡された引数 — そして最後のものが優先される。

## モジュールとの統合 {#module-integration}

モジュールがロードされると、`kernel/module/main.c` の `find_module_sections()` がモジュールの `__dyndbg`（および `__dyndbg_classes`）セクションを見つけ、`mod->dyndbg_info` に記録する。

```c
/* kernel/module/main.c、find_module_sections() */
mod->dyndbg_info.descs = section_objs(info, "__dyndbg",
                                      sizeof(*mod->dyndbg_info.descs),
                                      &mod->dyndbg_info.num_descs);
```

登録そのものはモジュールのnotifierチェーンを通じて行われる。`lib/dynamic_debug.c` は `ddebug_module_nb` を登録し、そのコールバックは `MODULE_STATE_COMING` で `ddebug_add_module()` を、`MODULE_STATE_GOING` で `ddebug_remove_module()` を呼ぶ。

```c
/* lib/dynamic_debug.c */
static int ddebug_module_notify(struct notifier_block *self, unsigned long val,
                                void *data)
{
    struct module *mod = data;

    switch (val) {
    case MODULE_STATE_COMING:
        ret = ddebug_add_module(&mod->dyndbg_info, mod->name);
        ...
    case MODULE_STATE_GOING:
        ddebug_remove_module(mod->name);
        ...
    }
}
```

notifierブロックは意図的に `.priority = 0` を使っている。これにより、同じチェーンに登録されているjump labelが、dynamic debugの実行前に初期化される。以前のカーネルは、モジュールローダに組み込まれた明示的な `dynamic_debug_setup()` / `dynamic_debug_remove()` の呼び出しを使っていたが、v6.4でこのnotifierに置き換えられた。

ロード時に `dyndbg` モジュールパラメータが渡された場合（例: `insmod mymodule.ko dyndbg=+p`）、それは本物のモジュールパラメータではない — モジュールローダの `unknown_module_param_cb()` がそれを特別に認識し、`ddebug_dyndbg_module_param_cb()` に渡す。これはディスクリプタが登録された後、`mod->init()` が実行される前に行われるので、init関数自体からのデバッグメッセージも捕捉できる。

## pr_debugとdev_dbg {#prdebug-vs-devdbg}

どちらもdynamic debugの基盤を使い、同じインターフェースで制御される。

```c
/* 汎用 — デバイスに結び付かない */
pr_debug("packet count: %d\n", count);

/* デバイスを意識する — 出力の前にデバイス名を付ける */
dev_dbg(&pdev->dev, "DMA transfer complete, status=%#x\n", status);
```

ドライバのコードでは `dev_dbg()` が推奨される。出力に含まれるデバイス名から、どのハードウェアインスタンスがメッセージを出したかがすぐにわかるからである。どちらも無効時はno-op（NOP）である。

## print_hex_dump_debug {#printhexdump_debug}

大量のデータには、`print_hex_dump_debug()` がバッファを16進 + ASCII形式でダンプする。これもdynamic debugで制御される。

```c
print_hex_dump_debug("rx buf: ", DUMP_PREFIX_OFFSET, 16, 1,
                     buf, len, true);
```

これは次のような出力を生成する:
```
rx buf: 00000000: 45 00 00 3c 1c 46 40 00 40 06 ac 10 7f 00 00 01  E..<.F@.@.......
```

16進ダンプの呼び出しサイトは、上のフラグ規則の部分的な例外である: `print_hex_dump_debug()` と `print_hex_dump_bytes()` では、`p` と `d` だけが効果を持つ。接頭辞の装飾（`t`、`m`、`f`、`s`、`l`）はパーサに受け付けられるが、これらのサイトでは何もしない。出力が、`__dynamic_pr_debug()` 用に装飾付き接頭辞を組み立てる関数である `__dynamic_emit_prefix()` ではなく、`print_hex_dump()` によって生成されるからである。ただし `d` は依然として働く: `dump_stack()` は接頭辞の経路からではなく、16進ダンプを含むすべてのdynamic debugサイトを保護する `include/linux/dynamic_debug.h` の `__dynamic_func_call*` ラッパーマクロから呼び出されるからである。

`format` で一致させることはできるが、対象は本物のフォーマット文字列ではなく `prefix_str` 引数である: ディスクリプタは、`prefix_str` が定数文字列であればそれを、実行時に組み立てられるものであればリテラルの `hexdump` を記録する。そのため、上の呼び出しは `format "rx buf: "` で選択される。

## ftraceと組み合わせた軽量トレーシング {#combining-with-ftrace-for-lightweight-tracing}

dynamic debugの出力でスレッド情報（`+t`）を有効にし、タイムスタンプをftraceのfunction graphトレーサと比較すれば、独自のtracepointを書かずに、デバッグメッセージとカーネルの関数呼び出しを関連付けられる。

```bash
# スレッド情報付きでdynamic debugを有効にする
echo "module mymodule +pt" > /sys/kernel/debug/dynamic_debug/control

# 同じモジュールに対してftraceのfunction graphを有効にする。
# set_ftrace_filterはモジュールではなく関数名に一致するので、
# "mod:" フィルタコマンドでモジュールに絞り込む: <function>:mod:<module>
echo '*:mod:mymodule' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function_graph > /sys/kernel/debug/tracing/current_tracer
cat /sys/kernel/debug/tracing/trace_pipe &

# ワークロードを実行する — pr_debugの出力とftraceが同じタイムライン上に現れる
```

単に `echo mymodule > set_ftrace_filter` としても、これは実現*できない* — 関数名のグロブとして解釈されるからである。モジュールと同じ名前の関数はまずないので何も一致せず、`ftrace_process_regex()` は一致ゼロの書き込みを `-EINVAL` にするので、シェルは黙って無視するのではなく `write error: Invalid argument` を報告する。（無関係な関数がたまたま同じ名前を持っていた場合は、書き込みは成功し、代わりにその関数がトレースされる。）`mod:` フィルタコマンドはモジュール名の前に関数のグロブを取るので、`'*:mod:mymodule'` は「`mymodule` 内のすべての関数」を意味し、`'e1000_tx*:mod:e1000'` は一部に絞り込む。別のモジュールの関数を追加するには `>>` で追記し、エントリを削除するには先頭に `!` を付ける。

## 実装: jump labelの書き換え {#implementation-jump-label-patching}

`_ddebug` エントリの `p` フラグが設定されると、カーネルは次のことを行う。

1. `_ddebug` 構造体の `flags` フィールドを更新する。
2. `_ddebug` ディスクリプタの `key` フィールドに埋め込まれたstatic key（`dp->key.dd_key_true`）に対して `static_branch_enable()` を呼ぶ。
3. jump labelサブシステム（`jump_label_update()`）が呼び出しサイトのNOPを短い分岐に書き換え、実行を `__dynamic_pr_debug()`（または `__dynamic_dev_dbg()`）に向ける。

呼び出しサイトのアドレスとすべてのメタデータは、コンパイル時に `_ddebug` ディスクリプタに直接格納される — 実行時にELFのデバッグ情報を検索することはない。無効化されると、`static_branch_disable()` によって `jump_label_update()` が分岐をNOPに書き戻す。書き換えはjump labelサブシステムによって適用されるので、他のCPU上での並行実行に対して安全である。

## 何が有効になっているかの確認 {#checking-what-is-enabled}

```bash
# 登録されたすべての呼び出しサイト — 有効なら=p、無効なら=_ と表示される
cat /sys/kernel/debug/dynamic_debug/control
# ファイル自身のヘッダ行が形式を示している:
# filename:lineno [module]function flags format
#
# 例（e1000モジュールが登録する約109個のnetdev_dbg()サイトのうち2つで、
# 1つは有効。残りの大半はe1000.hのe_dbg()ラッパーに由来する）:
# drivers/net/ethernet/intel/e1000/e1000_main.c:3573 [e1000]e1000_change_mtu =p "changing MTU from %d to %d\n"
# drivers/net/ethernet/intel/e1000/e1000_main.c:4429 [e1000]e1000_clean_rx_irq =_ "Receive packet consumed multiple buffers\n"

# 有効なサイトを数える
grep -c "=p" /sys/kernel/debug/dynamic_debug/control

# すべて無効にする
echo "module mymodule -p" > /sys/kernel/debug/dynamic_debug/control
```

上の行番号はv7.2-rc7のもので、カーネルのリリースごとにずれていく。ファイル、モジュール、関数の列が安定した部分である。また、最初の列はソースルートからの相対パスであり、これはまさに `file` の一致指定が受け付ける形式である点にも注意 — そのため、この出力の1行は、最初の列を `:lineno` の末尾ごと `file` の後に貼り付けるだけでクエリにできる。

```bash
echo "file drivers/net/ethernet/intel/e1000/e1000_main.c:3573 +p" \
    > /sys/kernel/debug/dynamic_debug/control
```

`ddebug_parse_query()` は `:3573` の末尾を切り離し、1行の範囲として解析し直すので、これは出力行の元になったサイトだけを正確に選択する。2番目の列 `[module]function` はクエリ構文ではなく表示上の書式である — クエリで使うには、`module` と `func` の別々の指定として打ち直す必要がある。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/dynamic_debug.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/dynamic_debug.h) — 上で引用した `struct _ddebug` の定義、フラグ文字の背後にある `_DPRINTK_FLAGS_*` のビット定義、`DEFINE_DYNAMIC_DEBUG_METADATA()` / `DYNAMIC_DEBUG_BRANCH()`（`__dyndbg` ディスクリプタとstatic branchを出力するマクロ）、`dynamic_hex_dump()`
- [lib/dynamic_debug.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/dynamic_debug.c) — 実行時処理のすべて: `ddebug_change()` は一致する呼び出しサイトを走査し、`static_branch_enable()`/`static_branch_disable()` で `dp->key.dd_key_true` を切り替える。`__dynamic_pr_debug()` と `__dynamic_dev_dbg()` は装飾付きの接頭辞を組み立てる。`struct ddebug_table` はモジュールごとの登録レコードである。`dynamic_debug_init_control()` はdebugfsとprocfsの両方の制御ファイルを作成する
- [include/linux/printk.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/printk.h) — このページの冒頭で述べた `pr_debug()` の3通りの展開（`CONFIG_DYNAMIC_DEBUG` の下では `dynamic_pr_debug()`、`DEBUG` の下では `printk(KERN_DEBUG …)`、それ以外では `no_printk()`）と、`print_hex_dump_debug()` および `pr_debug_ratelimited()` の同様のパターン
- [kernel/module/main.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/main.c) — `find_module_sections()` は `__dyndbg`/`__dyndbg_classes` を `mod->dyndbg_info` に取り込み、`unknown_module_param_cb()` はロード時の `dyndbg=` 引数を `ddebug_dyndbg_module_param_cb()` に回す
- [include/asm-generic/vmlinux.lds.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/asm-generic/vmlinux.lds.h) — `BOUNDED_SECTION_BY(__dyndbg, ___dyndbg)`: vmlinuxのリンカスクリプトが、組み込みのすべての呼び出しサイトのディスクリプタを1つの境界付き配列に集める方法
- [commit `7deabd674988`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7deabd67498869640c937c9bd83472574b7dea0b) — "dyndbg: use the module notifier callbacks"（Jason Baron、2023年3月、v6.4でマージ）: モジュールローダから古い `dynamic_debug_setup()`/`dynamic_debug_remove()` のフックを削除して `ddebug_module_notify()` に置き換え、notifierの優先度がjump labelの先行初期化を許すものでなければならない理由を説明している
- [commit `239a5791ffd5`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=239a5791ffd5559f51815df442c4dbbe7fc21ade) — "dynamic_debug: allow to work if debugfs is disabled"（Greg Kroah-Hartman、2020年2月、v5.7でマージ）: `/proc/dynamic_debug/control` を追加したコミット

### manページ {#man-pages}

- [`modprobe.d(5)`](https://man7.org/linux/man-pages/man5/modprobe.d.5.html) — モジュールごとの `dyndbg=` 設定をブートをまたいで永続化するために使う `options <modulename> <option>` ディレクティブ
- [`modprobe(8)`](https://man7.org/linux/man-pages/man8/modprobe.8.html) — コマンドライン、エイリアス、設定ファイルからのモジュールパラメータが、`unknown_module_param_cb()` に届く前にどう結合されるか

### 関連ページ {#related-pages}

- [カーネルモジュールの作成とロード](module-basics.md) — モジュール作成者のログ出力ツール群の中で `pr_debug()`/`dev_dbg()` が占める位置
- [モジュールロードの内部](module-loading-internals.md) — `load_module()`、セクションの解析、dynamic debugがフックする `MODULE_STATE_COMING`/`MODULE_STATE_GOING` のnotifierチェーン
- [モジュールパラメータ、シンボル、Kconfig](module-params.md) — ロード時の `dyndbg=` 引数が、そもそもパラメータとして自身を登録しないことで意図的に回避している `module_param()` の仕組み
- [printk: カーネルのログ出力の内部](../kernel/printk.md) — 呼び出しサイトが有効になった後も、デバッグ出力が通過しなければならないログバッファとログレベルによるフィルタリング
- [ftrace: 関数トレーサ](../tracing/ftrace.md) — 上の関連付けのレシピでdynamic debugと組み合わせたトレーサ

### LWNの記事 {#lwn-articles}

- [The dynamic debugging interface](https://lwn.net/Articles/434833/) — Jonathan Corbet、2011年3月22日: LWNによるこの機能の紹介。`pr_debug()`/`dev_dbg()`、debugfsの制御ファイル、クエリ言語を扱う
- [dynamic debug](https://lwn.net/Articles/286191/) — Jason Baronによる2008年6月の最初のパッチ投稿。単一の統一インターフェースの動機となった問題提起（`dprintk`、`pr_debug`、`DEBUGP`、そしてそれらを有効にする互換性のない十数種類の方法）を含む
- [The perils of `pr_info()`](https://lwn.net/Articles/487437/) — Jonathan Corbet、2012年3月21日: 裸の `printk(KERN_DEBUG …)` より `pr_debug()` が望ましい理由。dynamic debugの制御ファイルから届くのは前者だけだからである
- [Jump label](https://lwn.net/Articles/412072/) — Jonathan Corbet、2010年10月27日: 無効化されたdynamic debugの呼び出しサイトをコストゼロにするNOP書き換えの仕組み
- [Jump label reworked](https://lwn.net/Articles/436041/) — Jonathan Corbet、2011年3月30日: `static_branch()` APIの作り直し。これが最終的に、今日 `ddebug_change()` が呼ぶstatic keyのインターフェースになった

### 外部資料 {#external}

- [Dynamic debug — The Linux Kernel documentation](https://docs.kernel.org/admin-guide/dynamic-debug-howto.html) — 制御ファイルの権威あるリファレンス: 一致指定の完全な文法（`func`、`file`、`module`、`format`、`class`、`line`）、ワイルドカードと行範囲の規則、完全なフラグ一覧と装飾が出力される固定の順序、`;` 区切りの複数クエリの形式
- [The kernel's command-line parameters](https://docs.kernel.org/admin-guide/kernel-parameters.html) — 正式な `dyndbg[="val"]` と `<module>.dyndbg[="val"]` のブートパラメータの項目（上のhowtoは、ブート時のクエリは1023文字を超えてはならないこと、単独の `dyndbg=` はブート時にのみ処理される一方 `<module>.dyndbg` はモジュールのロード時に再処理されることを補足している）
- [Static keys — The Linux Kernel documentation](https://docs.kernel.org/staging/static-keys.html) — `static_branch_enable()`/`static_branch_disable()` の意味と、無効化された呼び出しサイトがコンパイルされた結果のNOPを示す逆アセンブル
- [Message logging with printk](https://docs.kernel.org/core-api/printk-basics.html) — ログレベルと `console_loglevel`。これらは有効化された `pr_debug()` の出力がコンソールに届く途中で依然としてフィルタリングを行う
