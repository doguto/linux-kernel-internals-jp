# ページポイズニングとメモリのゼロ化 {#page-poisoning-and-memory-zeroing}

> カーネルのポイズニング機構で use-after-free バグを捕捉し、メモリの衛生を保つ

Linux カーネルは、使用前または使用後にメモリを既知のパターンで埋めるための、互いに重複する複数の機構を提供している。純粋にデバッグ用（バグの捕捉）のものもあれば、セキュリティ用（データ漏洩の防止）のものもある。どのツールを使うべきかを判断するには、それらの違いを理解する必要がある。

## 概要 {#overview}

| 機構 | レイヤ | パターン | 目的 | 設定 |
|-----------|-------|---------|------|--------|
| `CONFIG_PAGE_POISONING` | ページアロケータ | `0xAA`（解放時）、`0x00`（割り当て時） | デバッグ: UAF と初期化前使用の捕捉 | `CONFIG_PAGE_POISONING` |
| `init_on_alloc` / `init_on_free` | ページ + slab | `0x00`（ゼロ） | セキュリティ: データ漏洩の防止 | ブートパラメータ（v5.3+） |
| `slub_debug=P` | SLUB の slab キャッシュ | `0x6b`（解放時）、`0xa5`（レッドゾーン） | デバッグ: slab の UAF とオーバーフローの捕捉 | `CONFIG_SLUB_DEBUG` + ブートパラメータ |

これらは独立した機構である。組み合わせることもできるが、一部の組み合わせは冗長になる（例: `CONFIG_PAGE_POISONING` と `init_on_free=1` を両方有効にすると、同じページを2回ゼロ化する）。

## CONFIG_PAGE_POISONING: デバッグ用ポイズニング {#config_page_poisoning-debug-poisoning}

### 何をするのか {#what-it-does}

`CONFIG_PAGE_POISONING` は、ページアロケータに、ページの解放時にそのページを既知のバイトパターンで埋めさせ、次にそのページが割り当てられるときにパターンが乱されていないことを検証させる。

実装は [`mm/page_poison.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_poison.c) にある。

使われるパターン:

- **解放時**: ページは `0xAA` で埋められる（[`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) の `__free_pages_prepare()` から呼ばれる `kernel_poison_pages()` による）
- **割り当て時**: カーネルはページを要求元に渡す前に、すべてのバイトがまだ `0xAA` であることを検証し、その後 `0x00` で埋める

```
CONFIG_PAGE_POISONING でのページのライフサイクル:

  ページを割り当てる
  ┌────────────────────────────┐
  │  verify all bytes == 0xAA  │  ← 一致しなければ: BUG（解放後に誰かが書き込んだ）
  │  fill with 0x00            │  ← 呼び出し元はゼロ化されたページを得る
  └────────────────────────────┘
          │
          │  （呼び出し元がページを使う）
          ▼
  ページを解放する
  ┌────────────────────────────┐
  │  fill with 0xAA            │  ← ポイズンマーカー
  └────────────────────────────┘
          │
          │  （ページはフリーリスト上にとどまる）
          ▼
  このページからの次の割り当て:
  ┌────────────────────────────┐
  │  verify all bytes == 0xAA  │  ← 解放中に起きた書き込みを捕捉する
  └────────────────────────────┘
```

### use-after-free をどう捕捉するか {#how-it-catches-use-after-free}

コードが解放済みのページに書き込むと、いくつかの `0xAA` バイトが上書きされる。次にそのページが割り当てられるとき、検証スキャンが `0xAA` と一致しないバイトを見つけ、破損を報告する。

問題はタイミングである。書き込みは解放から次の割り当てまでの間のいつでも起こりうる。ページがすぐに再利用されれば、その窓は小さい。ページがしばらくフリーリスト上にとどまっていれば（例: 割り当て圧力が低い場合）、より後に行われた書き込みも検出される。

### 有効化する {#enabling-it}

```
# カーネルの設定
CONFIG_PAGE_POISONING=y
```

ポイズニングはブート時に切り替えることもできる（コンパイル時に組み込まれていても、デフォルトでは **オフ** である）:

```bash
# ブート時に有効化（必須 — CONFIG_PAGE_POISONING=y だけでは有効にならない）
page_poison=on

# 明示的に無効化（デフォルト）
page_poison=off
```

### 性能コスト {#performance-cost}

ページポイズニングは、解放のたびにページ全体の書き込みを、割り当てのたびにページ全体の読み出し（検証）と書き込み（ゼロ化）を追加する。order-0 ページ（x86 では4KB）の場合、割り当てサイクルごとに2回の余分なメモリ操作となる。高次ページ（例: 2MB の huge page）では、コストはページサイズに比例して増える。

実際には、オーバーヘッドは測定可能だがワークロードに依存する。割り当ての多いワークロード（例: ネットワーク集約型、小さな割り当てが頻繁なもの）は、起動時に一度だけ割り当てるワークロードよりも大きなオーバーヘッドを受ける。このツールは開発用カーネルや CI に適しており、本番環境向けではない。

### ページポイズニングのレポートの読み方 {#reading-a-page-poisoning-report}

検証で破損したバイトが検出されると、カーネルは次のようなメッセージを出力する:

```
BUG: Bad page state in process kworker/0:1  pfn:0x12ab3
page:ffffea00004aac00 refcount:0 mapcount:0 mapping:0000000000000000 index:0x0 pfn:0x12ab3
flags: 0x200000000000000(zone=2)
raw: 0200000000000000 ffffea00004aac08 ffffea00004aac08 0000000000000000
raw: 0000000000000000 0000000000000000 00000000ffffffff 0000000000000000
page dumped because: PAGE_FLAGS_CHECK_AT_FREE flag(s) set
Poison overwritten
First byte 0x0 instead of 0xaa
Call Trace:
 dump_stack+0x48/0x5e
 bad_page+0xb0/0x100
 check_new_page_bad+0x48/0x70
 get_page_from_freelist+0x590/0x13e0
 __alloc_pages_nodemask+0x12b/0x280
 ...
```

**レポートの読み方**:

1. `Poison overwritten` — ポイズニングのチェックが失敗した。このページがフリーリスト上にある間に、何らかのコードが書き込んだ
2. `First byte 0x0 instead of 0xaa` — 見つかった具体的なバイト値（ここではゼロで、書き込みがゼロ化操作か bzero 風の memset だったことを意味する）
3. コールトレースは、チェックを引き起こした **ページを割り当てた側** を示す — 書き込んだ側ではない（それには KASAN かウォッチポイントが必要）
4. 書き込んだ側を見つけるには、KASAN と突き合わせるか、ポイズンされたアドレスにハードウェアウォッチポイント（`perf mem`）を設定する

!!! tip "KASAN との併用"
    `CONFIG_PAGE_POISONING` と KASAN は相補的な役割を果たす。KASAN（[KASAN](kasan.md) を参照）は書き込みが起きたその瞬間に（正確なスタックトレース付きで）捕捉する。ページポイズニングは次の割り当て時に捕捉するので、有効化は簡単だが、犯人についての情報は少ない。最大のカバレッジを得るには、デバッグビルドで両方を一緒に実行するとよい。

### いつ使うか {#when-to-use-it}

`CONFIG_PAGE_POISONING` を使うのは次のような場合である:

- ページ単位の割り当てが関わる use-after-free が疑われる（slab オブジェクトではない — slab には `slub_debug=P` を使う）
- 単純でコンパイラに依存しないチェックが欲しい（KASAN と違い、特別なコンパイラフラグは不要）
- KASAN のオーバーヘッドが大きすぎる VM や低性能なシステムで実行している

## init_on_alloc と init_on_free: セキュリティのためのゼロ化 {#init_on_alloc-and-init_on_free-security-zeroing}

### 何をするのか {#what-they-do}

Linux v5.3 で導入された `init_on_alloc` と `init_on_free` は、デバッグツールではなく **セキュリティ機能** である。その目的は、機密性のあるカーネルデータが新しい割り当てに漏洩するのを防ぐことである。

**コミット**: [6471384af2a6](https://git.kernel.org/linus/6471384af2a6)（"mm: security: introduce init_on_alloc=1 and init_on_free=1 boot options"）
**作者**: Alexander Potapenko

実装のフックは [`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c)（ページ用）と [`mm/slab_common.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/slab_common.c)（slab 用）にある。

- **`init_on_alloc=1`**: すべてのページと slab の割り当ては、呼び出し元に渡される前にゼロ化される。呼び出し元にはすべてのバイトが `0x00` に見える。
- **`init_on_free=1`**: すべてのページと slab オブジェクトは、解放時にゼロ化される。解放されたメモリは、アロケータに戻る前にクリアされる。

```
init_on_free=1（セキュリティ重視）:
  呼び出し元がメモリを解放する
  └─→ カーネルが即座にゼロ化する             ← 機密データはこの時点で消去される
      └─→ メモリは（ゼロ化された状態で）フリーリスト上にとどまる
          └─→ 次の呼び出し元はゼロ化されたメモリを得る（おまけ）

init_on_alloc=1（より単純で、保証がやや異なる）:
  呼び出し元がメモリを要求する
  └─→ カーネルが返す直前にゼロ化する         ← 前の所有者の機密データが
      └─→ 呼び出し元はゼロ化されたメモリを得る     クリアされる
```

`init_on_free` のほうが強いセキュリティ保証を与える。メモリが再割り当てされない場合でも（例: 解放されたまま放置されるカーネルバッファ）、解放した瞬間にデータが消去されることを保証するからである。

### デバッグ用ポイズニングとの違い {#how-it-differs-from-debug-poisoning}

| 観点 | `CONFIG_PAGE_POISONING` | `init_on_alloc/free` |
|--------|------------------------|---------------------|
| パターン | 解放時に `0xAA`、割り当て時に検証 | 常に `0x00`（ゼロ） |
| 目的 | デバッグ: 解放済みメモリへの書き込みを検出 | セキュリティ: 機密データを消去 |
| UAF の検出 | する（非ゼロのパターン） | しない（割り当て時にゼロは検証されない） |
| オーバーヘッド | ページサイクルあたり読み出し+書き込みが2倍 | 割り当てまたは解放ごとに書き込み1回 |
| コンパイラ対応の要否 | 不要 | 不要 |
| 対象範囲 | ページのみ | ページと slab オブジェクト |

重要な違いは次のとおりである。`0xAA` パターンはセキュリティには役立たない（攻撃者は `0xAA` バイトを探すだけで、そのメモリが機密でないとわかる）が、デバッグには役立つ（それ以外の値の書き込みは検出できる）。ゼロ化はセキュリティには役立つが、デバッグからは見えない（解放済みページへのゼロの書き込みは、期待されるゼロパターンと同じに見える）。

### 有効化する {#enabling-it_1}

```bash
# ブートパラメータ（再コンパイル不要）
init_on_alloc=1
init_on_free=1

# あるいは両方を設定
init_on_alloc=1 init_on_free=1
```

これらはコンパイル時のデフォルトとして設定することもできる:

```
CONFIG_INIT_ON_ALLOC_DEFAULT_ON=y
CONFIG_INIT_ON_FREE_DEFAULT_ON=y
```

### 性能コスト {#performance-cost_1}

v5.3 のコミットメッセージでは、カーネルのコンパイル（CPU と割り当てを多用するワークロード）でオーバーヘッドが測定されている:

- `init_on_alloc=1`: オーバーヘッド1%未満（割り当て時のゼロ化はキャッシュのウォーミングと大部分が重なる）
- `init_on_free=1`: カーネルビルドでウォールタイムのオーバーヘッドが約8%。`security/Kconfig.hardening` ではこれを「ほとんどの場合3–5%」と記載している

`init_on_free` は `init_on_alloc` よりかなり高コストである。解放されたメモリはキャッシュ上でコールドであり、それをゼロ化するとキャッシュミスが強制されるからである。メモリ割り当ての多いワークロードでは相対的なオーバーヘッドが大きくなる。大きな連続バッファを一度だけ割り当てて再利用するワークロードでは、オーバーヘッドはほぼゼロである。

!!! note
    `CONFIG_PAGE_POISONING` も有効になっている場合、解放時に `init_on_free` とページポイズニングの両方が実行される。カーネルはこれらを自動的に重複排除しないため、ページを2回ゼロ化する可能性がある。本番に近い性能テストを行う場合は、一方を有効にするなら他方を無効にすること。

### いつ使うか {#when-to-use-it_1}

- **セキュリティが重要なカーネル**: セキュリティ強化ガイド（例: KSPP の推奨事項）に準拠するために `init_on_free=1` を有効にする
- **コンテナやマルチテナントのワークロード**: テナント A が解放したメモリが、再割り当て後にテナント B から読めてしまうのを防ぐ
- **デバッグツールではない**: use-after-free を捕捉しようとしているなら、代わりに KASAN か `CONFIG_PAGE_POISONING` を使う

## slub_debug ポイズニング: slab レベルのデバッグ {#slub_debug-poisoning-slab-level-debugging}

### 何をするのか {#what-it-does_1}

`CONFIG_PAGE_POISONING` はページアロケータのレベルで動作し、生のページ割り当てが関わるバグを捕捉する。しかしカーネルのほとんどの割り当ては slab アロケータ（`kmalloc`、`kmem_cache_alloc`）を経由し、slab アロケータはページをより小さなオブジェクトに切り分ける。それらには slab レベルのポイズニングが必要である。

SLUB（v2.6.23 以降のデフォルトの slab アロケータ）には、`slub_debug` ブートパラメータで制御される組み込みのデバッグ機能がある。`P` フラグでポイズニングが有効になる。

実装は [`mm/slub.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/slub.c) にある。

### ポイズンパターン {#the-poison-patterns}

SLUB のポイズニングは、2つの異なるバイト値を使う（[`include/linux/poison.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/poison.h) で定義）:

| パターン | 値 | 意味 |
|---------|-------|---------|
| `POISON_FREE` | `0x6b`（ASCII の `k`） | オブジェクトは解放済み |
| `POISON_END` | `0xa5` | オブジェクト終端のマーカー（レッドゾーン境界） |
| `POISON_INUSE` | `0x5a` | オブジェクトは使用中（一部のアロケータ） |

解放時には、オブジェクト本体が `0x6b` で埋められ、レッドゾーンのバイトが `0xa5` に設定される。次の割り当て時に、SLUB はオブジェクトを返す前にこれらのパターンを検証する。

```
ポイズニング（P フラグ）とレッドゾーン（Z フラグ）ありの SLUB オブジェクトレイアウト:

  ┌──────────┬─────────────────────────────┬──────────┐
  │ redzone  │    object body (0x6b when   │ redzone  │
  │ (0xa5)   │    free, data when in use)  │ (0xa5)   │
  └──────────┴─────────────────────────────┴──────────┘
```

`0x6b` という値（ASCII の `k`）はメモリダンプ中で見分けやすい目印であり、`crash(8)` のようなクラッシュ解析ツールで解放済みの slab メモリを見つけやすくしている。

### SLUB ポイズニングの有効化 {#enabling-slub-poisoning}

```bash
# すべての slab キャッシュで有効化
slub_debug=P

# 他のフラグと組み合わせる（徹底的なデバッグに推奨）
slub_debug=FZPU
# F = 整合性チェック
# Z = レッドゾーン（隣接オブジェクトへのオーバーフローを捕捉）
# P = ポイズニング（use-after-free を捕捉）
# U = 割り当て/解放のスタックを追跡

# 特定のキャッシュ（例: ext4_inode_cache）だけで有効化
slub_debug=P,ext4_inode_cache

# kmalloc-64 だけで有効化
slub_debug=P,kmalloc-64
```

あるいは、実行時に sysfs 経由で特定のキャッシュを対象にする:

```bash
# どのキャッシュが存在するか確認
ls /sys/kernel/slab/

# 特定キャッシュのポイズン状態を確認（読み取り専用 — コンパイル時の SLAB_POISON フラグを反映）
cat /sys/kernel/slab/kmalloc-64/poison
```

カーネル設定で `CONFIG_SLUB_DEBUG=y` が必要である（デバッグカーネルでは通常自動的に有効になっている）。

### slab の use-after-free をどう捕捉するか {#how-it-catches-slab-use-after-free}

コードが解放済みの slab オブジェクトに書き込むと、いくつかの `0x6b` バイトが上書きされる。次にその slab からオブジェクトが取り出されるとき、SLUB の `check_object()` 関数が本体をスキャンして `0x6b` でないバイトを探す。見つかった場合、ポイズン破損を報告する:

```
=============================================================================
BUG kmalloc-64 (Tainted: G    B   ): Poison overwritten
-----------------------------------------------------------------------------

INFO: 0xffff888100a51640-0xffff888100a5167f @offset=1600. First byte 0x0 instead of 0x6b
INFO: Allocated in bad_driver_init+0x38/0x60 age=142 cpu=0 pid=1234
INFO: Freed in bad_driver_exit+0x20/0x40 age=12 cpu=0 pid=1234
INFO: The buggy address belongs to the object at ffff888100a51640
 which belongs to the cache kmalloc-64 of size 64

Call Trace:
 dump_stack+0x48/0x5e
 print_trailer+0xf0/0x150
 object_err+0x2e/0x40
 check_object+0x218/0x270          ← SLUB のポイズン検証
 alloc_debug_processing+0x6e/0x170
 ___slab_alloc+0x42e/0x560
 kmalloc_trace+0x26/0x50
 ...
```

**レポートの読み方**:

1. `BUG kmalloc-64: Poison overwritten` — `kmalloc-64` キャッシュ内の解放済みオブジェクトに、解放中に書き込みがあった
2. `First byte 0x0 instead of 0x6b` — オブジェクトのオフセット0のバイトが、期待される `0x6b` ポイズンではなく `0x0` だった
3. `Allocated in` / `Freed in` — SLUB は最後の割り当て箇所と解放箇所を追跡している（`slub_debug` の `U` フラグが必要）
4. コールトレースはチェックを引き起こした側（次の割り当て元）を示し、書き込んだ側は示さない

### いつ使うか {#when-to-use-it_2}

`slub_debug=P` を使うのは次のような場合である:

- slab で割り当てられたオブジェクトでの use-after-free が疑われる（`kmalloc` と `kmem_cache_alloc` のオブジェクトのほとんどがこれに該当する）
- どの slab キャッシュが関わっているかを特定したい（キャッシュ指定の形式 `slub_debug=P,mycache` を使う）
- 完全な KASAN よりオーバーヘッドを抑えつつ、ポイズンレベルのカバレッジは必要

!!! warning "性能への影響"
    `slub_debug=FZPU` をすべてのキャッシュに適用すると、割り当ての多いワークロードではシステムが5–10倍遅くなることがある。性能が重要な場合は、`slub_debug=P,cache_name` で疑わしいキャッシュだけを対象にすること。

## 適切なツールの選び方 {#choosing-the-right-tool}

```
use-after-free の疑い
├── ページ単位の割り当て（vmalloc、alloc_pages）が関わる?
│   └── CONFIG_PAGE_POISONING
├── slab オブジェクト（kmalloc、kmem_cache_alloc）が関わる?
│   ├── スタックトレース付きで正確に「誰がやったか」を知りたい → KASAN（kasan.md を参照）
│   ├── より低いオーバーヘッドでパターンベースの検出をしたい → slub_debug=P
│   └── 本番環境で実行している → KFENCE（kfence.md を参照）
└── セキュリティ上の懸念（テナント間のデータ漏洩を防ぎたい）?
    └── init_on_free=1
```

## 主要なソースファイル {#key-source-files}

| ファイル | 説明 |
|------|-------------|
| [`mm/page_poison.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_poison.c) | `CONFIG_PAGE_POISONING` の実装: `kernel_poison_pages()`、`kernel_unpoison_pages()` |
| [`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) | ポイズニングと `init_on_alloc/free` を呼び出すページアロケータのフック |
| [`mm/slub.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/slub.c) | SLUB アロケータ: `set_freepointer()`、`check_object()`、ポイズンの書き込みと検証 |
| [`mm/slab_common.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/slab_common.c) | slab レイヤ向けの `init_on_alloc` / `init_on_free` フック |
| [`include/linux/poison.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/poison.h) | ポイズンバイトの定数: `POISON_FREE`（`0x6b`）、`POISON_END`（`0xa5`） |

## 歴史 {#history}

### CONFIG_PAGE_POISONING {#config_page_poisoning}

ページポイズニングは、git による体系的な履歴管理よりも前から存在する。`PAGE_POISONING` 設定オプションと `0xAA` パターンは、デバッグ支援として長年カーネルに存在してきた。現在の `mm/page_poison.c` ファイルは、以前は `mm/page_alloc.c` に散らばっていたロジックをまとめたものである。

### init_on_alloc / init_on_free（v5.3、2019年） {#init_on_alloc-init_on_free-v53-2019}

**コミット**: [6471384af2a6](https://git.kernel.org/linus/6471384af2a6)（"mm: security: introduce init_on_alloc=1 and init_on_free=1 boot options"）
**作者**: Alexander Potapenko
**LWN の記事**: [Initializing memory on allocation and release](https://lwn.net/Articles/791380/)（2019年）

ある文脈のカーネルメモリが、クリアされないまま別の文脈で再利用されるという、情報漏洩の脆弱性のクラスが動機となった。このパッチは、カーネルのメモリのライフサイクルをデフォルトでより安全にしようという、Kernel Self Protection Project（KSPP）に沿ったより広範な取り組みの一部であった。

### slub_debug ポイズニング {#slub_debug-poisoning}

SLUB は SLAB の置き換えとして v2.6.23（2007年）で導入された。`P`（ポイズニング）フラグと `0x6b`/`0xa5` パターンは、当初から SLUB のデバッグ基盤の一部であり、旧来の SLAB アロケータの慣例を引き継いだものである。`/sys/kernel/slab/<cache>/poison` の sysfs インターフェースにより、再起動せずにキャッシュごとの制御が可能になっている。

## 参考文献 {#further-reading}

- [mm/page_poison.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_poison.c) — `CONFIG_PAGE_POISONING` の実装: `kernel_poison_pages()` が解放されたページを `0xAA` で埋め、`kernel_unpoison_pages()` が割り当て時にパターンを検証する
- [mm/page_alloc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) — ページポイズニングと、`init_on_alloc`/`init_on_free` によるセキュリティ用ゼロ化パスの両方を呼び出すページアロケータのフック
- [mm/slub.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/slub.c) — SLUB アロケータ: `set_freepointer()` と `check_object()` が `0x6b`/`0xa5` の slab ポイズンパターンを実装している
- [include/linux/poison.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/poison.h) — `POISON_FREE`（`0x6b`）と `POISON_END`（`0xa5`）の正式な定義
- `Documentation/admin-guide/kernel-parameters.txt` — ブートパラメータ `page_poison`、`init_on_alloc`、`init_on_free`、`slub_debug` のリファレンス項目
- [Initializing memory on allocation and release](https://lwn.net/Articles/791380/)（LWN、2019年） — `init_on_alloc=1` と `init_on_free=1` の設計根拠と、セキュリティ対デバッグのトレードオフ
- [kasan](kasan.md) — コンパイラによる計装を用いたメモリエラー検出。書き込みの瞬間に正確なスタックトレース付きで use-after-free を捕捉し、ページポイズニングによる遅延検出を補完する
- [kfence](kfence.md) — サンプリングベースのメモリエラー検出。より重いページポイズニングや KASAN と異なり、本番環境で有効にしても安全である
- [slab](slab.md) — SLUB アロケータの内部: slab キャッシュのレイアウト、オブジェクトのライフサイクル、そして `slub_debug=P` の土台となるデバッグフラグの基盤
