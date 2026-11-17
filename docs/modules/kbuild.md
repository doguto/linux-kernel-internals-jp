# Kbuild: カーネルのビルドシステム {#kbuild-the-kernel-build-system}

> Makefile、Kconfig、ツリー外モジュール、クロスコンパイル

## 概要 {#overview}

カーネルのビルドシステム（Kbuild）はGNU Makeの上に構築されている。ソースツリー全体に分散した `Makefile` の断片の階層と、強力な設定言語（Kconfig）を使う。

```
make menuconfig    → .configを生成する（シンボルの定義）
make               → .configを使ってカーネルをコンパイルする
make modules_install → モジュールを/lib/modules/$(uname -r)/にインストールする
```

## .config: カーネルの設定 {#config-kernel-configuration}

```bash
# デフォルト設定から新たに始める:
make defconfig          # アーキテクチャのデフォルト設定（arch/$SRCARCH/configs/）
make allmodconfig       # すべてをモジュールとして有効化
make allyesconfig       # すべてを組み込みで有効化
make localmodconfig     # 現在ロードされているモジュールに合わせた最小構成

# 対話的な設定ツール:
make menuconfig         # テキストベースのメニュー（ncurses-devが必要）
make xconfig            # Qt GUI
make gconfig            # GTK GUI
make nconfig            # より新しいncurses UI

# 既存の設定をベースにする:
cp /boot/config-$(uname -r) .config
make olddefconfig        # 新しいシンボルにはデフォルトを採用する

# ツリーの広範囲をビルドテストするためのランダムな設定:
make randconfig         # ランダムな設定（テスト用）
```

### 設定ファイルの形式 {#config-file-format}

```bash
# .configの形式: 各CONFIG_SYMBOLは次のいずれかを持つ:
CONFIG_SMP=y          # 組み込み（y = yes）
CONFIG_MODULES=y      # ロード可能モジュールに必要
CONFIG_EXT4_FS=m      # モジュールとしてビルド
# CONFIG_NFS_FS is not set   # 無効

# シンボルを問い合わせる:
./scripts/config --state CONFIG_SMP
# y

# シンボルを設定する（実際にプロンプトを持つものを選ぶこと — プロンプトのない
# シンボルについては「コンパイラオプションとビルドフラグ」の注記を参照）:
./scripts/config --enable CONFIG_DEBUG_KERNEL
./scripts/config --disable CONFIG_SWAP
./scripts/config --module CONFIG_EXT4_FS
./scripts/config --set-val CONFIG_LOG_BUF_SHIFT 18
```

## Kconfig言語 {#kconfig-language}

```kconfig
# drivers/net/ethernet/intel/Kconfig

config E1000
    tristate "Intel(R) PRO/1000 Gigabit Ethernet support"
    depends on PCI && HAS_IOPORT
    help
      This driver supports Intel(R) PRO/1000 gigabit ethernet family of
      adapters.  For more information on how to identify your adapter, go
      to the Adapter & Driver ID Guide that can be located at:

      <http://support.intel.com>
      ...

config E1000E
    tristate "Intel(R) PRO/1000 PCI-Express Gigabit Ethernet support"
    depends on PCI && (!SPARC32 || BROKEN)
    depends on PTP_1588_CLOCK_OPTIONAL
    select CRC32
    help
      ...

# 型:
# bool:     yまたはn
# tristate: y、m（モジュール）、またはn
# string:   CONFIG_DEFAULT_HOSTNAME="(none)"
# int:      CONFIG_LOG_BUF_SHIFT=18
# hex:      CONFIG_PAGE_OFFSET=0xC0000000

# 依存関係:
# depends on A && B    → AとBの両方がy/mでなければならない
# depends on A || B    → 少なくとも一方
# depends on !A        → Aが設定されていてはならない
# select A             → Aを強制的に有効化する（dependsを無視する）
# imply A              → Aの有効化を提案する（上書き可能）
```

## Kbuild Makefile {#kbuild-makefiles}

各ディレクトリには、Kbuildに何をコンパイルするかを伝える `Makefile` がある。

```makefile
# drivers/net/ethernet/intel/e1000/Makefile

# obj-$(CONFIG_E1000): 'y' → 組み込み、'm' → モジュール、未設定 → スキップ
obj-$(CONFIG_E1000) += e1000.o

# e1000.oは複数のソースファイルからビルドされる:
e1000-y := e1000_main.o e1000_hw.o e1000_ethtool.o e1000_param.o

# サブディレクトリ:
obj-y += subdirname/
# → subdirname/Makefileへ再帰する
```

### トップレベルのビルドの流れ {#top-level-build-flow}

```
make
├── scripts/Makefile.build: 各サブディレクトリへ再帰する
├── 各Makefile: obj-y、obj-mをビルドに追加する
├── vmlinux: すべてのobj-yをカーネルバイナリにリンクする
└── modules: すべてのobj-mを.koファイルにコンパイルする
```

## ツリー外モジュールのビルド {#building-an-out-of-tree-module}

```makefile
# 外部モジュール（hello_module.c）用のMakefile:
obj-m := hello_module.o

KDIR ?= /lib/modules/$(shell uname -r)/build
PWD  := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean

install:
	$(MAKE) -C $(KDIR) M=$(PWD) modules_install
```

```bash
# モジュールをビルドする:
make

# インストール（/lib/modules/$(uname -r)/updates/にコピーされる。これは外部
# モジュール用のデフォルトのINSTALL_MOD_DIRである）:
sudo make install
sudo depmod -a       # モジュール依存関係データベースを更新する

# ロード:
modprobe hello_module
```

### 複数のソースファイル {#multiple-source-files}

```makefile
obj-m := mydriver.o
mydriver-y := main.o helper.o init.o
# main.o、helper.o、init.oをビルドし、mydriver.koにリンクする
```

## クロスコンパイル {#cross-compilation}

```bash
# x86-64ホスト上でARM64向けにクロスコンパイルする:
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-

make defconfig      # arch/arm64/configs/defconfigを使う
make -j$(nproc)

# CROSS_COMPILEの接頭辞は次のものの前に付けられる:
# ${CROSS_COMPILE}gcc、${CROSS_COMPILE}ldなど

# 32ビットARM（例: Raspberry Pi 2/3、armv7）の場合:
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabihf-
make multi_v7_defconfig   # arch/arm/configs/。CONFIG_ARCH_BCM2835を含む
make -j$(nproc)

# ツリー外モジュールのクロスコンパイル（kbuildを直接呼び出す: -Cは準備済みの
# カーネルビルドツリーを、M=はモジュールのソースディレクトリを指す）:
make -C /path/to/arm64-kernel-build M=$PWD \
     ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- modules
```

## コンパイラオプションとビルドフラグ {#compiler-options-and-build-flags}

```makefile
# コンパイルフラグを追加する:
ccflags-y += -DDEBUG_MODE
ccflags-y += -I$(src)/include   # $(src) = このMakefileのディレクトリ

# ファイルごとのフラグ:
CFLAGS_hello_module.o += -DTEST_BUILD

# 警告を無効にする:
CFLAGS_my_file.o += -w
```

```bash
# CONFIG_*シンボルは.config（include/config/auto.conf経由）から来るもので、
# makeのコマンドラインから来るのではない — そこで設定するのはkbuildの流儀ではない。
# .configを編集してから、新しいシンボルを解決し直す:

# デバッグ情報付きでビルドする（KGDBやクラッシュ解析用）。CONFIG_DEBUG_INFO
# 自体にはプロンプトがない — 「Debug information」のchoiceによってselectされるので、
# 代わりにchoiceの項目を有効にする。そのchoice自体が
# `depends on DEBUG_KERNEL` であり、DEBUG_KERNELにはデフォルトがないので、先に
# DEBUG_KERNELを有効にしないとolddefconfigがDWARFの選択も黙って取り消してしまう:
./scripts/config --enable CONFIG_DEBUG_KERNEL
./scripts/config --enable CONFIG_DEBUG_INFO_DWARF_TOOLCHAIN_DEFAULT
make olddefconfig && make -j$(nproc)

# アドレスサニタイザ付きでビルドする:
./scripts/config --enable CONFIG_KASAN
make olddefconfig && make -j$(nproc)

# コンパイル時の警告を有効にする。レベルは独立しており、累積的ではない:
# 各レベルはKBUILD_EXTRA_WARNに対する個別のfindstringテストで制御されるので、
# W=3はレベル3の警告だけを有効にし、レベル1と2は有効にならない。
make W=1        # あまり頻繁には出ない、関連性のある警告
make W=2        # かなり頻繁に出るが、関連性がありうる警告
make W=3        # より細かな警告。ほとんどの場合は無視してよい
make W=123      # 3つすべてを組み合わせる（W=12、W=13なども同様に動作する）
make C=1        # sparseによる静的解析
make C=2        # すべてのファイルにsparseを適用

# ドキュメント:
make htmldocs   # カーネルのドキュメントをビルドする
```

## depmodとモジュールの依存関係 {#depmod-and-module-dependencies}

```bash
# modules.dep: modprobe用の依存関係データベース:
depmod -a     # 現在のカーネル用に生成する
depmod -a 5.15.0  # 特定のカーネルバージョン用

# モジュールの依存関係を見る:
modinfo -F depends virtio_net
# virtio,net_failover

# modprobeは依存モジュールを自動的にロードする:
modprobe virtio_net
# virtio_netの依存先のうち、それ自体がモジュールであるもの（ここではvirtio、
# net_failover）をvirtio_netより先にロードする。どれがモジュールかは設定に
# 依存する: CONFIG_VIRTIO_NETはNET_FAILOVER、DIMLIB、PAGE_POOLをselectし、
# VIRTIOにdependsする。NET_FAILOVERとDIMLIBはtristateなのでモジュールになりうるが、
# PAGE_POOLは単なるboolで常に組み込まれる。

# depmodの出力を直接見る:
cat /lib/modules/$(uname -r)/modules.dep | grep virtio_net
# kernel/drivers/net/virtio_net.ko: kernel/drivers/virtio/virtio.ko ...
```

## ビルド成果物 {#build-artifacts}

```bash
# make後:
# トップレベルのビルドディレクトリ:
# vmlinux       ← 非圧縮のELFカーネル（デバッグ用）
# System.map    ← シンボルテーブル（アドレス）
ls arch/x86/boot/
# bzImage       ← ブート可能な圧縮カーネル

# モジュール署名。署名はビルドの一部ではない: `make` はビルドツリー内の.koを
# 未署名のまま残す。scripts/Makefile.modinstが `make modules_install`
# （CONFIG_MODULE_SIG_ALL=yの場合）または `make modules_sign` の際にモジュールに署名するので、
# *インストールされた*コピーを調べること:
modinfo /lib/modules/$(uname -r)/kernel/drivers/net/ethernet/intel/e1000/e1000.ko | grep sig
# sig_id: PKCS#7
# sig_hashalgo: sha256

# モジュールが定義するグローバルなテキストシンボル（EXPORT_SYMBOLによるエクスポートとは別物）:
nm --defined-only drivers/net/ethernet/intel/e1000/e1000.ko | grep " T "
# モジュール内で定義されたテキスト（関数）シンボルのみ

# モジュールが提供するものと必要とするものを確認する:
modinfo drivers/net/ethernet/intel/e1000/e1000.ko
# filename: ...
# license: GPL v2
# description: Intel(R) PRO/1000 Network Driver
# vermagic: 7.2.0-rc7 SMP preempt mod_unload modversions

# vermagic: 実行中のカーネルと完全に一致しなければならない — ただし
# CONFIG_MODVERSIONSの場合は先頭のリリース文字列が読み飛ばされ、後続のフラグ
# だけが比較される（kernel/module/version.cのsame_magic()）
```

## 便利なビルドターゲット {#useful-build-targets}

```bash
make -j$(nproc)             # 並列ビルド
make bzImage                # カーネルイメージのみ
make modules                # モジュールのみ
make modules_install         # モジュールを/lib/modules/にインストールする
make install                 # カーネルイメージとSystem.mapをインストールする

# 単一のファイル:
make drivers/net/ethernet/intel/e1000/e1000.o

# 単一のモジュール:
make drivers/net/ethernet/intel/e1000/e1000.ko

# サニタイズ済みのUAPIヘッダをエクスポートする（古い `headers_check` ターゲットは
# v5.17で削除された。代わりにCONFIG_UAPI_HEADER_TESTがそれらをコンパイルテストする）:
make headers_install

# クリーン:
make clean     # ビルド成果物を削除する（.configは残す）
make mrproper  # .configを含めてすべて削除する
make distclean # mrproper + エディタやタグの残骸: *.orig、*.rej、*~、*.bak、
               # #*#、*%、core、tags/TAGS、cscope*、GPATH/GRTAGS/GSYMS/GTAGS
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [Makefile](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Makefile) — トップレベルのビルドドライバ: `W=1`/`W=2`/`W=3` の警告レベルと `C=1`/`C=2` のsparseチェック、`ARCH`/`CROSS_COMPILE` の処理、上に挙げた `clean`/`mrproper`/`distclean` ターゲット
- [scripts/Makefile.build](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/Makefile.build) — ディレクトリごとの再帰エンジン。各 `Makefile` の `obj-y`/`obj-m` リストを読み、`obj-y += subdir/` へ降りていく
- [scripts/Makefile.lib](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/Makefile.lib) — 複合オブジェクトを展開し（`<module>-y` / `<module>-objs` の接尾辞検索）、ファイルごとのフラグ `ccflags-y`、`asflags-y`、`CFLAGS_<file>.o` を組み立てる
- [scripts/Makefile.modinst](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/Makefile.modinst) — `modules_install` のルール。`INSTALL_MOD_DIR ?= updates` に注意。これは [commit b74d7bb7ca24](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b74d7bb7ca2412ab37bab782591573b5f265872b)（"kbuild: Modify default INSTALL_MOD_DIR from extra to updates"、v6.3）以降の外部モジュール用のデフォルトのインストール先サブディレクトリである
- [scripts/kconfig/](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/kconfig) — Kconfigの実装: `lexer.l` と `parser.y` が上に示した言語を解析し、`mconf.c`/`nconf.c`/`qconf.cc`/`gconf.c` が `menuconfig`/`nconfig`/`xconfig`/`gconfig` の背後にあるプログラムで、`streamline_config.pl` が `localmodconfig` を実装している
- [scripts/config](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/config) — 上で使った `--state`/`--enable`/`--disable`/`--module`/`--set-val` による `.config` の編集を実装するシェルスクリプト

### manページ {#man-pages}

- [`modprobe(8)`](https://man7.org/linux/man-pages/man8/modprobe.8.html) — 依存関係を考慮したモジュールのロード、`-r`、コマンドラインでのモジュールパラメータ
- [`depmod(8)`](https://man7.org/linux/man-pages/man8/depmod.8.html) — `modules.dep` とマップファイルを生成する。`-a`、`-b basedir`、省略可能な `version` 引数
- [`modules.dep(5)`](https://man7.org/linux/man-pages/man5/modules.dep.5.html) — `modprobe` が参照する依存関係データベースである `modules.dep`/`modules.dep.bin` の形式
- [`modinfo(8)`](https://man7.org/linux/man-pages/man8/modinfo.8.html) — 文書化された `.modinfo` のフィールド（`depends`、`license`、`alias`、`parm` など）を `-F field` で抽出する。kmodは上で示した `vermagic` や `sig*` のフィールドも出力するが、manページ自身のフィールド一覧には列挙されていない

### 関連ページ {#related-pages}

- [カーネルモジュールの作成とロード](module-basics.md) — モジュールのinit/exit、これらのMakefileが駆動するソースから `.ko` までのライフサイクル
- [モジュールロードの内部](module-loading-internals.md) — その後カーネルが `.ko` をどう扱うか: ELFの解析、再配置、`vermagic`/modversionsのチェック
- [モジュールパラメータ、シンボル、Kconfig](module-params.md) — `module_param()`、シンボルのエクスポート、Kconfigシンボルがモジュールのコードに届く仕組み
- [カーネルモジュール署名](module-signing.md) — `CONFIG_MODULE_SIG_ALL` とツリー外モジュールへの署名
- [KGDB: カーネルGDBデバッガ](../debugging/kgdb.md) — 上で説明したデバッグ情報付きビルドの利用者

### LWNの記事 {#lwn-articles}

- [How many ways are there to configure the Linux kernel?](https://lwn.net/Articles/1034811/) — Daroc Alden、2025年9月10日: 6.16のx86_64における32,468個のKconfigオプションと、`depends on`/`select` の制約が有効な設定の空間に及ぼす影響
- [A kbuild and kconfig maintainer change](https://lwn.net/Articles/1032722/) — 2025年8月6日: Masahiro Yamadaが8年を経て退任。kbuildはNathan ChancellorとNicolas Schierのもとで「odd fixes」の状態に移り、Kconfigは当時は完全にメンテナ不在となった — その後、同じ2人がKconfigも引き受けており、こちらも「odd fixes」の扱いである

### 外部資料 {#external}

- [Kernel Build System](https://docs.kernel.org/kbuild/index.html) — アップストリームの `Documentation/kbuild/` 一式の索引
- [Linux Kernel Makefiles](https://docs.kernel.org/kbuild/makefiles.html) — `obj-y`/`obj-m`、`<module>-y` による複合オブジェクト、`ccflags-y`/`subdir-ccflags-y`、`$(src)`/`$(obj)` の権威あるリファレンス
- [Building External Modules](https://docs.kernel.org/kbuild/modules.html) — `make -C $KDIR M=$PWD modules` のパターン、`KBUILD_EXTRA_SYMBOLS`、`modules_install` が外部モジュールを置く場所
- [Kconfig Language](https://docs.kernel.org/kbuild/kconfig-language.html) — `bool`/`tristate`/`string`/`int`/`hex` の型と、`depends on`、`select`、`imply` の正確な意味
- [Configuration targets and editors](https://docs.kernel.org/kbuild/kconfig.html) — `menuconfig`/`nconfig`/`xconfig`/`gconfig` のフロントエンドと、`KCONFIG_CONFIG`、`KCONFIG_ALLCONFIG`、`KCONFIG_SEED` の環境変数
- [Kbuild](https://docs.kernel.org/kbuild/kbuild.html) — ビルド時の環境変数: `KBUILD_OUTPUT`/`O=`、`KBUILD_EXTRA_WARN`/`W=`、`CROSS_COMPILE`、sparse用の `CF`、`INSTALL_PATH`
