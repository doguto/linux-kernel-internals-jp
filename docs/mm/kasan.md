# KASAN（Kernel Address Sanitizer） {#kasan-kernel-address-sanitizer}

> 実行時にカーネル内のメモリバグを検出する

## KASANとは何か {#what-is-kasan}

KASANはLinuxカーネル向けの動的メモリエラー検出器である（[docs.kernel.org](https://docs.kernel.org/dev-tools/kasan.html)）。次のものを検出する。

- **範囲外アクセス**: 割り当てられたオブジェクトの末尾を越えて読み書きする
- **use-after-free**: 解放済みのメモリにアクセスする
- **二重解放**: 同じオブジェクトを2回解放する
- **不正な解放**: 一度も割り当てられていないポインタを解放する

これらのバグはデータ破壊、クラッシュ、セキュリティ脆弱性の原因になり得る。Dirty COW（[CVE-2016-5195](https://git.kernel.org/linus/19be0eaffa3a)）は、発見されるまで9年間残り続けたメモリバグの一例である。

GenericモードとSW_TAGSモードでは、コンパイラがすべてのメモリアクセスにチェックを挿入し、アクセス先のメモリが有効であることを検証する。HW_TAGSモードではハードウェアがこのチェックを行う。違反が検出されると、カーネルはバグの種類、問題のアクセス、関係するオブジェクトの割り当て／解放履歴を示す詳細なレポートを出力する。

## KASANの仕組み {#how-kasan-works}

### 3つのモード {#the-three-modes}

KASANには3つのモードがあり、それぞれ検出範囲、性能、ハードウェア要件の間で異なるトレードオフを持つ。

| モード | 設定オプション | 仕組み | オーバーヘッド | 用途 |
|------|--------------|-----------|----------|----------|
| Generic KASAN | `CONFIG_KASAN_GENERIC` | シャドウメモリ | 高（[ドキュメント](https://docs.kernel.org/dev-tools/kasan.html)参照） | 開発およびCIテスト |
| SW_TAGS KASAN | `CONFIG_KASAN_SW_TAGS` | ソフトウェアによるメモリタグ付け | 中 | より広範なテスト（arm64のみ） |
| HW_TAGS KASAN | `CONFIG_KASAN_HW_TAGS` | ARM MTEのハードウェアタグ付け | 低 | 本番環境でも使えるテスト |

### Generic KASAN: シャドウメモリ {#generic-kasan-shadow-memory}

Generic KASANは最も広く使われているモードである。カーネルメモリのすべてのバイトの有効性を追跡する**シャドウメモリ**領域を維持することで動作する。

鍵となる着想は、1バイトのシャドウで実メモリ8バイトの状態を表現できることである。

```
Real memory (8-byte granule):
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │  ← 8 bytes of kernel memory
└───┴───┴───┴───┴───┴───┴───┴───┘
                │
                │  maps to
                ▼
         ┌───────────┐
         │ shadow[N] │  ← 1 byte of shadow memory
         └───────────┘
```

**シャドウバイトのエンコーディング**:

シャドウバイトの値（[`mm/kasan/kasan.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kasan/kasan.h)で定義）:

| シャドウ値 | カーネルの定義名 | 意味 |
|-------------|--------------|---------|
| `0x00` | — | 8バイトすべてが有効（アクセス可能） |
| `0x01`-`0x07` | — | 先頭Nバイトのみ有効（部分グラニュール） |
| `0xFF` | `KASAN_SHADOW_INIT` | アクセス不可／未初期化のシャドウ |
| `0xFE` | `KASAN_PAGE_REDZONE` | kmalloc_large割り当てのレッドゾーン |
| `0xFC` | `KASAN_SLAB_REDZONE` | slabオブジェクトのレッドゾーン |
| `0xFB` | `KASAN_SLAB_FREE` | 解放済みslabオブジェクト |
| `0xFA` | `KASAN_KMALLOC_FREETRACK` | 解放追跡メタデータ付きの解放済みkmallocオブジェクト |
| `0xF9` | `KASAN_GLOBAL_REDZONE` | グローバル変数のレッドゾーン |
| `0xF8` | `KASAN_VMALLOC_INVALID` | vmap領域内のアクセス不可領域 |
| `0xF1` | `KASAN_STACK_LEFT` | スタックの左レッドゾーン |
| `0xF2` | `KASAN_STACK_MID` | スタックの中間レッドゾーン |
| `0xF3` | `KASAN_STACK_RIGHT` | スタックの右レッドゾーン |

すべてのメモリアクセスに対して、コンパイラは次のチェックを挿入する。

```
1. Compute shadow address:  shadow_addr = (addr >> 3) + KASAN_SHADOW_OFFSET
2. Load shadow byte:        shadow_val  = *shadow_addr
3. If shadow_val != 0:
     a. If accessing all 8 bytes → BUG (invalid access)
     b. If partial access: check if (addr & 7) + size > shadow_val → BUG
4. Otherwise: access is valid, proceed
```

各メモリアクセスには、シャドウチェックのための追加命令が発生する。シャドウメモリ自体はカーネルアドレス空間全体の1/8を消費する。

```
Kernel address space layout with KASAN:

┌──────────────────────────┐  High addresses
│    Kernel code/data      │
├──────────────────────────┤
│    Shadow memory         │  ← 1/8 of kernel address space
│    (tracks validity of   │
│     all kernel memory)   │
├──────────────────────────┤
│    Normal kernel memory  │
└──────────────────────────┘  Low addresses
```

### SW_TAGS KASAN: ソフトウェアによるメモリタグ付け {#sw_tags-kasan-software-memory-tagging}

Generic KASANのオーバーヘッドのため、専用のCIシステム以外で広くテストに使うのは現実的ではなかった。SW_TAGS KASANは、メモリ割り当てとポインタにランダムな**タグ**を付与することで、決定的な検出と引き換えにオーバーヘッドを下げる。アクセスのたびに、ポインタのタグがメモリのタグと一致するかをチェックする。

```
Tagged pointer (arm64 Top Byte Ignore):
┌────────┬──────────────────────────────────────────────┐
│ Tag(8) │              Virtual Address (56)            │
└────────┴──────────────────────────────────────────────┘

Shadow memory stores the memory tag (1 byte per 16 bytes of real memory).

Access check:
  pointer_tag = ptr >> 56
  memory_tag  = shadow[addr >> 4]
  if pointer_tag != memory_tag → BUG
```

これはarm64のTop Byte Ignore（TBI）機能に依存している。TBIはアドレス変換時にポインタの最上位バイトを無視する機能であり、カーネルはその最上位バイトにランダムなタグを格納する。

メモリが解放されると、シャドウには新しいランダムなタグが割り当てられる。古いポインタは古いタグを持ったままなので、use-after-freeはおそらく検出される（タグは8ビットなので、1回のアクセスあたりの検出確率は1 - 1/256）。

### HW_TAGS KASAN: ハードウェアによるメモリタグ付け（ARM MTE） {#hw_tags-kasan-hardware-memory-tagging-arm-mte}

HW_TAGS KASANは、ARMv8.5以降のプロセッサで利用できるARM Memory Tagging Extension（MTE）を使う。MTEは同じタグチェックの概念を完全にハードウェアで実装している。

- CPUは16バイトのメモリグラニュールごとに4ビットのタグを格納する（メモリコントローラ内の専用タグ記憶領域を使う）
- ポインタはビット[59:56]に4ビットのタグを持つ
- CPUは**追加命令ゼロ**ですべてのメモリアクセスのタグをチェックする

```
ARM MTE (tag in bits [59:56], defined in arch/arm64/include/asm/mte-def.h):
┌──────┬──────────┬──────────────────────────────────────┐
│ (4)  │ Tag(4)   │          Virtual Address (56)        │
│63..60│ 59..56   │              55..0                   │
└──────┴──────────┴──────────────────────────────────────┘
        │                        │
        │   hardware checks      │
        ▼                        ▼
   pointer tag  ==?  memory tag (stored in DRAM tag bits)
        │
        └── mismatch → hardware exception
```

タグチェックはCPUが行うため、メモリアクセスのパスに追加命令は入らない。トレードオフとして、タグ空間がより粗く（4ビット = 16通りのタグなので、1回のアクセスあたり1/16の確率でバグを見逃す）、MTE対応ハードウェア（ARMv8.5以降）が必要になる。

## KASANレポートの例 {#what-a-kasan-report-looks-like}

KASANがバグを検出すると、カーネルログにレポートを出力する。以下はslab-out-of-boundsアクセスの現実的な例である。

```
==================================================================
BUG: KASAN: slab-out-of-bounds in example_function+0x58/0x90
Write of size 4 at addr ffff888012345678 by task insmod/1234

CPU: 2 PID: 1234 Comm: insmod Not tainted 6.8.0-kasan #1
Hardware name: QEMU Standard PC (Q35 + ICH9, 2009)
Call trace:
 dump_backtrace+0x0/0x30
 show_stack+0x18/0x28
 dump_stack_lvl+0x48/0x60
 print_report+0xd0/0x520
 kasan_report+0xa8/0xe0
 example_function+0x58/0x90           ← the buggy access
 test_module_init+0x30/0x48
 do_one_initcall+0x80/0x2a0

Allocated by task 1234:
 kmalloc_trace+0x28/0x40
 example_function+0x20/0x90           ← where the object was allocated
 test_module_init+0x30/0x48
 do_one_initcall+0x80/0x2a0

The buggy address belongs to the object at ffff888012345660
 which belongs to the cache kmalloc-32 of size 32
The buggy address is located 24 bytes to the right of
 allocated 32-byte region [ffff888012345660, ffff888012345680)

Memory state around the buggy address:
 ffff888012345500: fb fb fb fb fc fc fc fc 00 00 00 00 fc fc fc fc
 ffff888012345580: fb fb fb fb fc fc fc fc 00 00 00 00 fc fc fc fc
>ffff888012345600: 00 00 00 00 fc fc fc fc 00 00 00 00 fc fc fc fc
                                                       ^
 ffff888012345680: fc fc fc fc fb fb fb fb fc fc fc fc 00 00 00 00
 ffff888012345700: fc fc fc fc fb fb fb fb fc fc fc fc 00 00 00 00
==================================================================
```

**レポートの読み方**:

1. **バグの種類**: `slab-out-of-bounds` -- アクセスがslabオブジェクトの末尾を越えた
2. **アクセスの詳細**: アドレス`ffff888012345678`への4バイトの書き込み
3. **コールトレース**: 不正アクセスが起きた正確な場所（`example_function+0x58`）を示す
4. **割り当てトレース**: オブジェクトが最初に割り当てられた場所を示す
5. **オブジェクトの詳細**: オブジェクトは`kmalloc-32`（32バイトのslabキャッシュ）にあり、アクセスは32バイトの割り当ての末尾から24バイト先だった
6. **シャドウメモリのダンプ**: `^`マーカーはバグのあるアドレスに対応するシャドウバイトを指す。`fc`はKASANのレッドゾーンを意味し、コードがslabオブジェクト間のパディングに書き込んだことを示す

## KASANの有効化 {#enabling-kasan}

### カーネル設定 {#kernel-configuration}

```
# KASANを有効化する（モードを1つ選ぶ）
CONFIG_KASAN=y

# Genericモード（コンパイラが対応していれば全アーキテクチャで動作）
CONFIG_KASAN_GENERIC=y

# または: ソフトウェアタグベースモード（arm64のみ）
CONFIG_KASAN_SW_TAGS=y

# または: ハードウェアタグベースモード（MTE対応のarm64のみ）
CONFIG_KASAN_HW_TAGS=y

# 併用推奨のオプション
CONFIG_KASAN_INLINE=y        # インラインチェック（outlineより高速）
CONFIG_STACKTRACE=y          # レポート中のスタックトレースを改善
CONFIG_SLUB_DEBUG=y          # slabデバッグ情報を強化
```

### ビルド要件 {#build-requirements}

コンパイラの要件（[`lib/Kconfig.kasan`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/Kconfig.kasan)より）:

| モード | GCC | Clang |
|------|-----|-------|
| Generic | 8.3.0以降 | カーネルがサポートする任意のバージョン |
| SW_TAGS | 11以降 | カーネルがサポートする任意のバージョン |
| HW_TAGS | 10以降 | 12以降 |

コンパイラは`-fsanitize=kernel-address`（Generic）または`-fsanitize=kernel-hwaddress`（SW_TAGS/HW_TAGS）をサポートしている必要がある。

```bash
# KASANを有効にしたカーネルのビルド
make defconfig
./scripts/config -e KASAN -e KASAN_GENERIC -e KASAN_INLINE
make -j$(nproc)
```

### ブート時オプション {#boot-time-options}

```bash
# HW_TAGSモード用: ブート時に有効化／無効化する
kasan.mode=on          # 有効化（デフォルト）
kasan.mode=off         # 無効化
kasan.stacktrace=on    # 割り当て／解放のスタックトレースを収集
kasan.fault=report     # レポートするがpanicしない（デフォルト）
kasan.fault=panic      # 最初のKASANエラーでpanicする
```

## 試してみる {#try-it-yourself}

### QEMUでKASANレポートを発生させる {#trigger-a-kasan-report-in-qemu}

カーネルにはKASAN用の組み込みテストモジュールが含まれている。

```bash
# KASANを有効にしたカーネルをQEMUで起動してから:

# KASANテストモジュールをロードする
modprobe kasan_test

# またはKASANのKUnitテストスイートを実行する
# （CONFIG_KUNIT=y と CONFIG_KASAN_KUNIT_TEST=y が必要）
./tools/testing/kunit/kunit.py run kasan
```

### KASANが有効か確認する {#check-if-kasan-is-active}

```bash
# カーネル設定を確認する
zcat /proc/config.gz | grep KASAN

# ブートメッセージを確認する
dmesg | grep -i kasan
```

### KASANレポートを読む {#read-kasan-reports}

```bash
# KASANレポートをリアルタイムで監視する
dmesg -w | grep -A 30 "BUG: KASAN"
```

## 性能オーバーヘッド {#performance-overhead}

| モード | CPUオーバーヘッド | メモリオーバーヘッド | 検出確率 |
|------|-------------|-----------------|----------------------|
| Generic | 高 | カーネルメモリの1/8をシャドウに使用 | 100% — 決定的 |
| SW_TAGS | 中 | カーネルメモリの1/16をシャドウに使用 | アクセスあたり1 - 1/256（約99.6%） |
| HW_TAGS | 低 | ハードウェアのタグ記憶領域（実装依存） | アクセスあたり1 - 1/16（約93.75%） |

（オーバーヘッドはワークロードによって異なる。詳細は[KASANのドキュメント](https://docs.kernel.org/dev-tools/kasan.html)を参照。）

Genericモードはオーバーヘッドが最も大きいが、すべてを検出する。カーネルのCIシステムや開発用ビルドでは標準的な選択肢である。HW_TAGSモードは本番ハードウェアに展開できるほどオーバーヘッドが小さい。

## 歴史 {#history}

### v4.0: 導入（2015年） {#v40-introduction-2015}

**コミット**: [0b24becc8105](https://git.kernel.org/linus/0b24becc8105)（"kasan: add kernel address sanitizer infrastructure"）

**作者**: Andrey Ryabinin（Samsung/Google）

KASANは、GoogleのKonstantin Serebryanyが作ったユーザー空間向けのAddressSanitizer（ASan）に着想を得たものである。Andrey Ryabininは、コンパイラによる計装とシャドウメモリを用いて、この概念をLinuxカーネルに適用した。初期実装はx86_64のみをサポートしていた。

**LWNの記事**: [The kernel address sanitizer](https://lwn.net/Articles/612153/)（2014年）

### v5.0: ソフトウェアタグベースモード（2019年） {#v50-software-tag-based-mode-2019}

**コミット**: [2bd926b439b4](https://git.kernel.org/linus/2bd926b439b4)（"kasan: add CONFIG_KASAN_GENERIC and CONFIG_KASAN_SW_TAGS"）

**作者**: Andrey Konovalov（Google）

Generic KASANのオーバーヘッドのため、専用のCI環境以外での利用は現実的ではなかった。SW_TAGS KASANは、arm64のTop Byte Ignore（TBI）機能を活用して、ソフトウェアによるメモリタグ付けを導入した。このモードは決定的な検出と引き換えにオーバーヘッドを下げ、より広範なテストシナリオを対象とする。

**LWNの記事**: [Tag-based KASAN](https://lwn.net/Articles/766768/)（2018年）

### v5.11: ハードウェアタグベースモード（2021年） {#v511-hardware-tag-based-mode-2021}

**コミット**: [2e903b914797](https://git.kernel.org/linus/2e903b914797)（"kasan, arm64: implement HW_TAGS runtime"）

**作者**: Andrey Konovalov（Google）

HW_TAGSモードはARM Memory Tagging Extension（MTE）を活用してタグチェックをハードウェアで行い、オーバーヘッドをさらに削減する。これによりKASANを本番ハードウェアに展開できるようになった。

**LWNの記事**: [Arm Memory Tagging Extension and KASAN](https://lwn.net/Articles/834289/)（2020年）

### v5.12: KASANのブートパラメータ {#v512-kasan-boot-parameters}

**コミット**: [3c8bc5e09bb2](https://git.kernel.org/linus/3c8bc5e09bb2)（"kasan: add boot parameters for hw tags mode"）

HW_TAGSモードにブート時パラメータ（`kasan.mode`、`kasan.stacktrace`、`kasan.fault`）が導入され、再コンパイルなしにKASANを有効化／無効化できるようになった。これは本番での利用に不可欠である。

### 対象範囲の拡大 {#expanding-coverage}

KASANは段階的に拡張され、より多くの種類のカーネルメモリをカバーするようになった。スタック変数とグローバル変数は、Genericモードではコンパイラによって計装される（v4.0の最初の実装に含まれていた）。vmalloc割り当てはv5.4で追加された（[commit 3c5c3cfb9ef4](https://git.kernel.org/linus/3c5c3cfb9ef4)、Daniel Axtens）。

## 主要なソースファイル {#key-source-files}

| ファイル | 説明 |
|------|-------------|
| [`mm/kasan/`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kasan) | KASAN実装のディレクトリ |
| [`mm/kasan/common.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kasan/common.c) | 全モード共通のコード（slabアロケータへのフック、ポイズニング） |
| [`mm/kasan/generic.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kasan/generic.c) | Generic（シャドウメモリ）モードの実装 |
| [`mm/kasan/sw_tags.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kasan/sw_tags.c) | ソフトウェアタグベースモード |
| [`mm/kasan/hw_tags.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kasan/hw_tags.c) | ハードウェアタグベースモード（ARM MTE） |
| [`mm/kasan/report.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kasan/report.c) | バグレポートの整形と出力 |
| [`mm/kasan/quarantine.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kasan/quarantine.c) | 解放済みオブジェクトの隔離（再利用を遅らせてuse-after-freeを検出する） |
| [`mm/kasan/shadow.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kasan/shadow.c) | シャドウメモリの管理 |
| [`include/linux/kasan.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/kasan.h) | KASANの公開APIとインラインフック |
| [`mm/kasan/kasan.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kasan/kasan.h) | KASAN内部の定義とシャドウ値の定数 |

## 参考文献 {#further-reading}

### カーネルドキュメント {#kernel-documentation}

- [KASAN公式ドキュメント](https://docs.kernel.org/dev-tools/kasan.html) -- 3つのモードすべて、設定、レポートの解釈を網羅した包括的なガイド

### LWNの記事 {#lwn-articles}

- [The kernel address sanitizer](https://lwn.net/Articles/612153/) -- 最初の紹介記事（2014年）
- [Tag-based KASAN](https://lwn.net/Articles/766768/) -- SW_TAGSモードの設計（2018年）
- [Arm Memory Tagging Extension and KASAN](https://lwn.net/Articles/834289/) -- HW_TAGSモード（2020年）

### 関連項目 {#related}

- [slab](slab.md) -- KASANはslabアロケータのフックを計装して割り当てと解放を追跡する
- [vmalloc](vmalloc.md) -- KASANはvmallocのメモリ領域もカバーする
- [vrealloc](vrealloc.md) -- vreallocの開発中に見つかったKASANのポイズニングのバグ
