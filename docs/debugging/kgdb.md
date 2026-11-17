# KGDB: カーネルGDBデバッガ {#kgdb-kernel-gdb-debugger}

> 稼働中のカーネルをGDBでライブデバッグする

## 概要 {#overview}

KGDBは、GDBをフロントエンドとして稼働中のカーネルをデバッグできるようにする。カーネルはブレークポイントで停止し、シリアルポート経由でGDBリモートプロトコルのスタブを公開する。2台目のマシン（またはVMのホスト）でGDBを実行する。

```
Target machine (running kernel to debug)
  │
  │  kgdb serial transport
  │
  ▼
Host machine (running GDB)
  gdb vmlinux
  (gdb) target remote /dev/ttyS0
```

1台のマシンでデバッグする場合は、VMの仮想シリアルポート経由のKGDBが最も実用的な構成である。

## カーネルの設定 {#kernel-configuration}

```bash
# 必要なカーネル設定オプション:
CONFIG_KGDB=y
CONFIG_KGDB_SERIAL_CONSOLE=y   # kgdboc: コンソール経由のKGDB
CONFIG_KGDB_KDB=y              # 任意: 組み込みのKDBシェル
CONFIG_FRAME_POINTER=y         # 信頼できるスタックトレース
CONFIG_DEBUG_INFO=y            # vmlinux内のシンボル情報
CONFIG_GDB_SCRIPTS=y           # カーネルの型を扱うGDB Pythonスクリプト
```

## 接続のセットアップ {#setting-up-the-connection}

### シリアル経由（物理またはVM） {#over-serial-physical-or-vm}

```bash
# ターゲット: kgdbがシリアルポートttyS0を115200ボーで使うよう設定する
# 起動時（カーネルパラメータ）:
console=ttyS0,115200 kgdboc=ttyS0,115200 kgdbwait

# kgdbwaitはデバッガがアタッチするまでブートの初期段階で停止する — これがないと、kgdbocは
# シリアルコンソールをkgdbのI/Oバックエンドとして登録するだけで、カーネルはそのまま
# 起動を続けるので、自分でデバッガへの突入を引き起こす必要がある（SysRq-gまたはブレークポイント）。

# あるいは実行時に（カーネルが対応していれば）:
echo ttyS0,115200 > /sys/module/kgdboc/parameters/kgdboc

# デバッガへの突入を引き起こす:
echo g > /proc/sysrq-trigger
# または: シリアル経由でSysRq-gを送る: Alt+SysRq+g
# または: ブレークポイントに到達させる
```

```bash
# ホスト: GDBを接続する
gdb vmlinux
(gdb) set serial baud 115200
(gdb) target remote /dev/ttyS0
# Remote debugging using /dev/ttyS0
# (gdb) ...
```

### メインラインにはネットワークトランスポートがない {#there-is-no-network-transport-in-mainline}

メインラインのkgdbにはEthernet/UDPのI/Oバックエンドはなく、`kgdboc`（シリアル）が唯一のトランスポートである。これはカーネル自身の`MAINTAINERS`にあるKGDB/KDBエントリの
ファイル一覧全体（`Documentation/process/debugging/kgdb.rst`、
`drivers/misc/kgdbts.c`、`drivers/tty/serial/kgdboc.c`、`include/linux/kdb.h`、`include/linux/kgdb.h`、
`kernel/debug/`、`kernel/module/kdb.c` — ネットワークドライバはない）で確認できる。`kgdboe=`ブートパラメータも、`kgdb_nmi`
モジュールも、ロードできるnetpollベースのkgdbバックエンドも存在しない。

ただし、この名前は架空のものではない。`kgdboc.c`のヘッダコメント自体に、"based on the same
principle as kgdboe using the NETPOLL api"（NETPOLL APIを使うkgdboeと同じ原理に基づく）と書かれている。kgdboeはnetpollベースのkgdb-over-Ethernetとして実在した設計で、
kgdbocの作者にとっての先例だったが、メインラインにマージされることはなかった（それを追加したコミットは存在せず、
`MAINTAINERS`エントリもない）。シリアル接続されたターゲットにリモートからアクセスする必要がある場合の標準的な
方法は、`ser2net`のようなシリアル-ネットワークブリッジを使うことであり、これはカーネルとは完全に別物で、カーネルの
機能ではない。

シリアル接続でのデバッグでは`kgdboc`が唯一のトランスポートであり、本ページの残りではこれを扱う。

## カーネルデバッグのための基本的なGDBコマンド {#basic-gdb-commands-for-kernel-debugging}

接続したら:

```
# カーネルバージョン情報を表示する
(gdb) info threads
(gdb) thread apply all bt    # 全スレッドのバックトレース

# 現在のタスクを調べる
(gdb) p $lx_current()        # 現在のtask_struct（GDBスクリプトが必要）
(gdb) lx-ps                  # 全プロセスを一覧表示（GDBスクリプト）

# スタックトレース
(gdb) bt
(gdb) bt full                # ローカル変数付き

# フレームの移動
(gdb) up
(gdb) down
(gdb) frame 3

# 変数を調べる
(gdb) p variable_name
(gdb) p *ptr
(gdb) p *(struct task_struct*)0xffff888012345678

# メモリの調査
(gdb) x/16x address          # 16個の16進ワード
(gdb) x/s address            # 文字列
(gdb) x/i address            # 逆アセンブル

# 実行の継続
(gdb) c                      # 実行を継続
(gdb) s                      # ステップ実行（関数内に入る）
(gdb) n                      # 次の行（関数をまたぐ）
(gdb) finish                 # 関数から戻るまで実行
```

## ブレークポイント {#breakpoints}

```
# ソフトウェアブレークポイント（x86ではINT3）
(gdb) break function_name
(gdb) break kernel/sched/core.c:1234
(gdb) break *0xffffffff81234567   # アドレス指定

# 条件付きブレークポイント
(gdb) break schedule if current->pid == 1234

# ハードウェアウォッチポイント: メモリが書き込まれたら停止
(gdb) watch *address           # 書き込みで停止
(gdb) rwatch *address          # 読み出しで停止
(gdb) awatch *address          # 読み出しまたは書き込みで停止

# 例: 特定のinodeが変更されたときに捕まえる
(gdb) watch -l inode->i_count

# ブレークポイントの一覧と管理
(gdb) info breakpoints
(gdb) disable 2
(gdb) delete 2
```

## カーネル用GDBスクリプト {#gdb-scripts-for-kernel}

カーネルには、カーネル固有のコマンドを追加するPythonベースのGDBスクリプト（`scripts/gdb/`）が同梱されている:

```
# スクリプトをロードする（CONFIG_GDB_SCRIPTS=yで.gdbinitが設定済みなら自動）
(gdb) source /path/to/kernel/scripts/gdb/vmlinux-gdb.py

# カーネル固有のコマンド:
(gdb) lx-version          # カーネルバージョン文字列
(gdb) lx-ps              # プロセス一覧（psと同様）
(gdb) lx-dmesg           # カーネルリングバッファ
(gdb) lx-lsmod           # ロード済みモジュール
(gdb) lx-mounts          # マウント済みファイルシステム
(gdb) lx-iomem           # /proc/iomem相当
(gdb) lx-symbols         # モジュールのシンボルをロード

# プリティプリンタ: struct task_struct、list_headなど
(gdb) p $lx_per_cpu("runqueues", 0)  # CPU 0のランキュー
(gdb) p $lx_current()               # 現在のタスク
(gdb) $container_of(ptr, "struct task_struct", "mm")
```

## カーネルモジュールのデバッグ {#debugging-a-kernel-module}

モジュールは実行時に未知のアドレスにロードされる。そのシンボルをGDBにロードしなければならない:

```bash
# ターゲット上で: モジュールがロードされた場所を調べる
cat /proc/modules | grep mymodule
# mymodule 16384 0 - Live 0xffffffffc0a00000

# あるいはsysfs経由で
cat /sys/module/mymodule/sections/.text
cat /sys/module/mymodule/sections/.data
```

```
# GDB内で: 報告されたアドレスにシンボルを追加する
(gdb) add-symbol-file mymodule.ko 0xffffffffc0a00000

# あるいはlx-symbolsで全モジュールを自動的にロードする
(gdb) lx-symbols /lib/modules/$(uname -r)/
```

## KDB: 組み込みのカーネルデバッガ {#kdb-the-built-in-kernel-debugger}

KDB（`CONFIG_KGDB_KDB=y`）は、2台目のマシンを必要とせず完全にターゲット上で動作する、よりシンプルなテキストベースのデバッガを提供する:

```bash
# KDBに入る: Alt+SysRq+gを押すかパニックを発生させる
# あるいはブレークポイントで

# KDBプロンプト:
[0]kdb> help
[0]kdb> ps              # プロセス一覧
[0]kdb> bt              # 現在のタスクのバックトレース
[0]kdb> btp <pid>       # 特定タスクのバックトレース
[0]kdb> md 0xffff888012345678 16   # メモリダンプ
[0]kdb> mm address value           # メモリ変更
[0]kdb> bp function_name           # ブレークポイントを設定
[0]kdb> go                         # 実行を継続
[0]kdb> dmesg                      # リングバッファを表示
[0]kdb> lsmod                      # モジュール一覧
```

## よくあるデバッグシナリオ {#common-debugging-scenarios}

### カーネルoopsのデバッグ {#debugging-a-kernel-oops}

```bash
# oopsの後、システムがまだ動作しているかどうかは場合による
# まだ動作していれば（致命的でないoops）、kgdbに入って:
(gdb) bt           # 現在位置を確認
(gdb) frame 5      # フォールトしたフレームに移動
(gdb) p *ptr       # NULL/不正なポインタを調べる
(gdb) list         # クラッシュ地点周辺のソースを表示
```

### 競合状態のデバッグ {#race-condition-debugging}

```bash
# カウンタ付きブレークポイント（100回目の呼び出しで停止）
(gdb) break kmalloc
(gdb) ignore 1 99        # 最初の99回をスキップ
```

### スケジューラの状態を調べる {#inspecting-scheduler-state}

```bash
(gdb) break __schedule
(gdb) c
# ブレークポイントで:
(gdb) p *current         # 現在のタスク
(gdb) p *current->mm     # 現在のmm_struct
(gdb) p $lx_per_cpu("runqueues", 0)  # ランキュー
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [Documentation/process/debugging/kgdb.rst](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/process/debugging/kgdb.rst) — kgdb/kdbの正式なドキュメント: 設定オプション、`kgdboc`の構文、kdbコマンドリファレンス（注: このファイルは古いツリーでは`Documentation/dev-tools/kgdb.rst`にあり、`Documentation/process/debugging/`に移動された）
- [kernel/debug/debug_core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/debug/debug_core.c) — kgdbとkdbの両方が共有するデバッグコア: 例外への突入とブレークポイントの処理
- [kernel/debug/kdb/kdb_main.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/debug/kdb/kdb_main.c) — kdbコマンドテーブルの大部分（`ps`、`bt`、`btp`、`go`、`dmesg`、`lsmod`、`md`、`mm`、`help`、...）
- [kernel/debug/kdb/kdb_bp.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/debug/kdb/kdb_bp.c) — `kdb_main.c`のテーブルとは別に登録される`bp`/`bph`ブレークポイントコマンド
- [drivers/tty/serial/kgdboc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/tty/serial/kgdboc.c) — kgdbをシリアルコンソールに結び付ける`kgdboc` I/Oドライバ
- [scripts/gdb/vmlinux-gdb.py](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/gdb/vmlinux-gdb.py) — `scripts/gdb/linux/`から`lx-*` GDBヘルパーコマンド（`lx-ps`、`lx-dmesg`、`lx-lsmod`、`lx-symbols`、...）と便利関数（`$lx_current()`、`$lx_per_cpu()`、`$container_of()`）をロードするエントリポイント

### 関連ページ {#related-pages}

- [kdumpとcrash](kdump.md) — 事後のクラッシュ解析
- [Oops解析](oops-analysis.md) — デバッガなしでoopsをデコードする
- [カーネルモジュール](../modules/module-basics.md) — モジュールのデバッグシンボルのロード
- [トレーシング: ftrace](../tracing/ftrace.md) — ブレークポイントより軽量な代替手段

### LWN記事 {#lwn-articles}

- [Bringing kgdb into 2.6](https://lwn.net/Articles/70465/)（2004） — ツリー内デバッガに対するLinus Torvaldsの歴史的な抵抗を押して、kgdbをマージさせようとした初期の議論を呼んだ取り組み
- [Merging kdb and kgdb](https://lwn.net/Articles/374633/)（2010） — kdbシェルとkgdbのgdbリモートスタブを共通のデバッグコアのもとに統合し、kdbをx86以外にも拡張したJason Wesselの作業

### 外部資料 {#external}

- [Using kgdb, kdb and the kernel debugger internals](https://docs.kernel.org/process/debugging/kgdb.html) — 正式なドキュメントのdocs.kernel.org版
- [GDB kernel debugging](https://docs.kernel.org/process/debugging/gdb-kernel-debugging.html) — `scripts/gdb/`のPythonヘルパーと便利関数に関するdocs.kernel.orgのページ
