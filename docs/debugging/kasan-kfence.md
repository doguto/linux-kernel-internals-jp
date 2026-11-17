# KASANとKFENCE {#kasan-and-kfence}

> メモリエラー検出: 境界外アクセス、use-after-free、ヒープ破壊

## 概要 {#overview}

カーネルには、実行時にメモリエラーを捕まえるための相補的な2つのツールがある:

- **KASAN**（Kernel Address Sanitizer） — 網羅的かつ決定的だがオーバーヘッドが大きい。開発やCIに最適
- **KFENCE**（Kernel Electric Fence） — サンプリングベースでオーバーヘッドはほぼゼロ。本番システムに適する

両者は同時に有効にできる。KFENCEはサンプリングしたものをオーバーヘッドなしで捕まえ、KASANは大きなオーバーヘッドと引き換えにすべてを捕まえる。

---

## KASAN: Kernel Address Sanitizer {#kasan-kernel-address-sanitizer}

`CONFIG_KASAN`はコンパイル時の計装を有効にし、すべてのメモリアクセスをシャドウメモリ領域と照合してチェックする。

### シャドウメモリ {#shadow-memory}

Generic KASANは、カーネルメモリの8バイトごとに、専用のシャドウ領域に格納された1つの**シャドウバイト**を対応付ける。4レベルページングのx86-64では、シャドウ領域は`0xdffffc0000000000`（`KASAN_SHADOW_OFFSET`）から始まる。

```
shadow_address = (address >> 3) + 0xdffffc0000000000;
```

シャドウバイトのエンコーディング:

| シャドウ値 | 意味 |
|---|---|
| `0` | 8バイトすべてにアクセス可能 |
| `1`–`7` | 先頭Nバイトにアクセス可能。N+1〜8バイト目はレッドゾーン |
| `0xFB`（`KASAN_SLAB_FREE`） | オブジェクトは解放済み（use-after-free） |
| `0xFC`（`KASAN_SLAB_REDZONE`） | kmallocの右側レッドゾーン |
| `0xf9`（`KASAN_GLOBAL_REDZONE`） | グローバル変数のレッドゾーン |
| `0xF1`（`KASAN_STACK_LEFT`） | スタックの左レッドゾーン |
| `0xF2`（`KASAN_STACK_MID`） | スタックの中間レッドゾーン |
| `0xF3`（`KASAN_STACK_RIGHT`） | スタックの右レッドゾーン |
| `0xF4`（`KASAN_STACK_PARTIAL`） | スタックフレームの部分レッドゾーン |

これらの定数は`mm/kasan/kasan.h`で定義されている。（`include/linux/kasan-tags.h`はこれとは別のポインタタグ定数 — `KASAN_TAG_KERNEL`、`KASAN_TAG_INVALID`など — を定義しており、これらは後述のSW-tag/HW-tagモードで使われるもので、これらのポイズン/レッドゾーン値ではない。）

### コンパイラによる計装 {#compiler-instrumentation}

`CONFIG_KASAN_GENERIC`では、GCC/Clangが`-fsanitize=kernel-address`付きで起動される。コンパイラはすべてのメモリアクセスの前に呼び出しを挿入する:

```c
/* ptrからの4バイトロードの前: */
__asan_load4(ptr);

/* ptrへの4バイトストアの前: */
__asan_store4(ptr);
```

これらの関数は`mm/kasan/generic.c`に実装されている（共通のエントリポイントは`kasan_check_range()`）。シャドウアドレスを計算し、シャドウバイトを読み、アクセスが不正なら報告する。

```c
/* 簡略化したシャドウチェック（mm/kasan/generic.c）: */
static __always_inline bool memory_is_poisoned_1(unsigned long addr)
{
    s8 shadow_value = *(s8 *)kasan_mem_to_shadow((void *)addr);
    if (unlikely(shadow_value)) {
        s8 last_accessible_byte = addr & KASAN_GRANULE_MASK;
        return unlikely(last_accessible_byte >= shadow_value);
    }
    return false;
}
```

### インライン計装とアウトライン計装 {#inline-vs-outline-instrumentation}

`CONFIG_KASAN_INLINE`（デフォルト）: シャドウチェックが各アクセス箇所にインライン展開される。高速だが、カーネルバイナリが大きくなる。

`CONFIG_KASAN_OUTLINE`: コンパイラは`__asan_load*`/`__asan_store*`への関数呼び出しを生成する。バイナリは小さいが、呼び出しのオーバーヘッドが大きい。

### slabアロケータとの統合 {#slab-allocator-integration}

KASANは`mm/kasan/common.c`で定義された関数を介してslabアロケータにフックする:

- `kasan_unpoison_pages(page, order, init)` — `__alloc_pages()`から呼ばれ、新たに割り当てられたページのポイズンを解除する
- `kasan_poison_pages(page, order, init)` — ページの解放時に呼ばれ、その領域をポイズンする
- `kasan_slab_alloc(cache, object, flags, init)` — 呼び出し元に返すslabオブジェクトのポイズンを解除する
- `kasan_slab_free(cache, object, ...)` — 解放されたslabオブジェクトを`KASAN_SLAB_FREE (0xFB)`でポイズンする

use-after-freeの検出のため、KASANは割り当て時と解放時のスタックトレースも記録する:

- `kasan_save_stack(flags)` — 現在のコールスタックを取得し、スタックレコードのリングバッファに格納する
- `kasan_set_track(track, stack)` — 取得したスタックをオブジェクトの`kasan_track`メタデータに格納する

割り当て側のメタデータは、オブジェクトのレッドゾーン内に置かれた`struct kasan_alloc_meta`に格納される。解放側のメタデータは別の`struct kasan_free_meta`にあり、解放されたオブジェクト自身のメモリ内に格納される:

```c
/* mm/kasan/kasan.h */
struct kasan_alloc_meta {
    struct kasan_track alloc_track;
    /* 解放トラックはkasan_free_metaに格納される。 */
    depot_stack_handle_t aux_stack[2];
};

struct kasan_free_meta {
    struct qlist_node quarantine_link;
    struct kasan_track free_track;
};
```

### SW-tag KASAN（arm64） {#sw-tag-kasan-arm64}

`CONFIG_KASAN_SW_TAGS`を有効にしたarm64では、KASANは**top-byte ignore**（TBI）機能を使う。ポインタのビット[63:56]はMMUに無視されるので、KASANはそこにランダムな8ビットのタグを格納する。

- 各`kmalloc`割り当てにはランダムなタグが与えられ、呼び出し元に返されるポインタのビット[63:56]にそのタグが入る。
- シャドウバイトには、その粒度単位（granule）に期待されるタグが格納される。
- アクセスのたびに、コンパイラが挿入したチェックがポインタのタグとシャドウのタグを比較し、不一致なら報告する。

これにより、粒度の粗さに起因する見逃しが減り、generic KASANが見逃すオブジェクト間の境界外アクセスの一部も捕まえられるようになる。

### HW-tag KASAN（arm64 MTE） {#hw-tag-kasan-arm64-mte}

`CONFIG_KASAN_HW_TAGS`はARMの**Memory Tagging Extension**（MTE）ハードウェアを使う:

- CPUは16バイトの粒度単位ごとに、4ビットのタグを物理メモリに格納する。
- ポインタはビット[59:56]に対応する4ビットのタグを持つ。
- 不一致があるとCPUがタグチェックフォールトを発生させる。シャドウメモリは不要で、アイドル時のオーバーヘッドはほぼゼロである。
- 統合部分は`mm/kasan/hw_tags.c`にあり、アロケータはMTE命令（`irg`、`stg`、`ldg`）のラッパーである`set_tag()`/`get_tag()`を呼ぶ。

### KASANレポートの形式 {#kasan-report-format}

```
==================================================================
BUG: KASAN: use-after-free in skb_put+0x68/0xa4
Write of size 4 at addr ffff888101234560 by task eth0/1234

CPU: 2 PID: 1234 Comm: eth0 Not tainted 6.8.0 #1
Call Trace:
 kasan_report+0xbd/0x100
 kasan_check_range+0x115/0x1a0
 skb_put+0x68/0xa4
 ath9k_rx_tasklet+0x234/0x580 [ath9k]
 tasklet_action_common+0x9a/0x160

Allocated by task 1234:
 kasan_save_stack+0x26/0x50
 __kasan_kmalloc+0x81/0xa0
 skb_alloc+0x44/0x80
 ath9k_rx_alloc+0x12/0x30 [ath9k]

Freed by task 1234:
 kasan_save_stack+0x26/0x50
 __kasan_slab_free+0x120/0x160
 kmem_cache_free+0x68/0x2e0
 dev_kfree_skb+0x18/0x20
 ath9k_rx_complete+0x88/0xc0 [ath9k]

The buggy address belongs to the object at ffff888101234500
 which belongs to the cache skbuff_head_cache of size 232
Shadow bytes around the buggy address:
 ffff888101234400: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
 ffff888101234480: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
>ffff888101234500: fb fb fb fb fb fb fb fb fb fb fb fb fb fb fb fb
                                                              ^
==================================================================
```

主なセクション:
1. **バグの種類と場所** — アクセスの種類、サイズ、アドレス、問題を起こした関数
2. **割り当てスタック** — オブジェクトが最初に割り当てられた場所
3. **解放スタック** — オブジェクトが解放された場所
4. **シャドウダンプ** — 周辺のシャドウバイト。`fb` = `KASAN_SLAB_FREE`（解放済みslabオブジェクト）

### オーバーヘッド {#overhead}

| リソース | コスト |
|---|---|
| メモリ | 約12.5%のオーバーヘッド（カーネルの8バイトにつき1シャドウバイト） |
| CPU | 約2〜3倍の速度低下（すべてのアクセスに計装） |
| インライン対アウトライン | インライン: 約20%高速、`.text`が約40%大きい |

### KASANを有効にする {#enabling-kasan}

```bash
# 最小限のKASAN設定:
CONFIG_KASAN=y
CONFIG_KASAN_GENERIC=y       # arm64ではKASAN_SW_TAGS / KASAN_HW_TAGSでもよい
CONFIG_KASAN_INLINE=y        # シャドウチェックをインライン化（高速）
CONFIG_DEBUG_INFO=y          # スタックトレースのシンボルに必要
CONFIG_SLUB_DEBUG=y          # KASANが必要とするSLUBメタデータを有効化
CONFIG_STACKTRACE=y          # 割り当て/解放のスタックを取得

# 任意だが推奨:
CONFIG_KASAN_STACK=y         # スタックの境界外アクセスを検出（多少のオーバーヘッドあり）
CONFIG_KASAN_VMALLOC=y       # vmallocの境界外アクセスを検出
CONFIG_KASAN_MODULE_TEST=y   # 自己チェック用のテストモジュール
```

---

## KFENCE: Kernel Electric Fence {#kfence-kernel-electric-fence}

`CONFIG_KFENCE`は、平均オーバーヘッドがほぼゼロの**サンプリング型**メモリエラー検出器を提供し、本番カーネルに適している。

### 仕組み {#how-it-works}

KFENCEはシャドウメモリを使わない。代わりに、オブジェクトの小さな**KFENCEプール**を管理し、各オブジェクトを**ガードページ**（マップされていないページ）で囲む。ガードページへの境界外書き込みは即座にページフォールトを引き起こし、KFENCEがそれを横取りして報告する。

```
KFENCE pool layout (one slot):
┌──────────────────┐  ← unmapped guard page (4 KB)
│  GUARD PAGE      │    OOB access here → page fault → KFENCE report
├──────────────────┤
│  object (N bytes)│   ← the actual allocation (left- or right-aligned, chosen at random)
├──────────────────┤
│  GUARD PAGE      │  ← unmapped guard page (4 KB)
│                  │    OOB access here → page fault → KFENCE report
└──────────────────┘
```

各オブジェクトは、**割り当てごとにランダムに選ばれる左右どちらかのページ境界**に寄せて配置される（`mm/kfence/core.c`の`get_random_u32_below(2)`）。これにより、その方向に1バイトでもはみ出すと、すぐに隣接するガードページに当たる。

**use-after-free**の検出: KFENCEオブジェクトが解放されると、そのページはアクセス不能とマークされる（`mprotect(PROT_NONE)`と同様）。以後のアクセスはすべてフォールトし、KFENCEレポートを発生させる。

### サンプリング {#sampling}

KFENCEはすべての割り当てで呼ばれるわけではない。代わりに、slabアロケータが設定可能な頻度で`__kfence_alloc()`を呼ぶ:

```c
/* mm/kfence/core.c — 簡略化。slabアロケータのホットパスから呼ばれる */
void *__kfence_alloc(struct kmem_cache *s, size_t size, gfp_t flags)
{
    /* 各呼び出しはゲートを1目盛り進める。0から1にした呼び出しだけが
     * この割り当てをKFENCEプールにサンプリングする。 */
    int allocation_gate = atomic_inc_return(&kfence_allocation_gate);
    if (allocation_gate > 1)
        return NULL;         /* この割り当ての番ではない */

    return kfence_guarded_alloc(s, size, flags, ...);
}
```

周期的なワークアイテム（`toggle_allocation_gate()`）が`kfence_sample_interval`ミリ秒ごとに発火し、`kfence_allocation_gate`を`-kfence_burst`にリセットする。これにより、インターバルごとに1回だけでなく最大`kfence_burst + 1`回のサンプルをまとめて取れる（`kfence_burst`のデフォルトは0、つまりインターバルごとに1サンプル）。`__kfence_alloc()`の各呼び出しは`atomic_inc_return()`でゲートをアトミックにインクリメントし、結果が`≤ 1`になった呼び出しだけがKFENCEプールにリダイレクトされる。`CONFIG_KFENCE_STATIC_KEYS`が有効な場合は、static branch（`kfence_allocation_key`）も高速パスをゲートするため、サンプリングされない割り当てはこのチェックをほぼ完全にスキップする。

### 設定 {#configuration}

```bash
# Kconfig:
CONFIG_KFENCE=y
CONFIG_KFENCE_NUM_OBJECTS=255   # KFENCEスロット数（デフォルト255）
                                 # スロットが多いほど検出確率が上がる
                                 # 各スロットは2ページ（ガード + オブジェクト）を使う

# ブートパラメータ:
kfence.sample_interval=100      # 約100msごとに1つの割り当てをサンプリング（デフォルト）
                                 # 0 = KFENCEを無効化
                                 # 値を小さくするほどカバレッジが増え、メモリ使用量も増える

# sysctl（実行時の調整）:
echo 50 > /sys/module/kfence/parameters/sample_interval
```

### KFENCEプールのサイズ {#kfence-pool-size}

`CONFIG_KFENCE_NUM_OBJECTS=255`でスロットあたり2ページ（8KB）の場合、プールは約2MBの物理メモリを使う。各スロットは固定サイズの領域であり、ページより小さいオブジェクトは、スロットのデータページ内でランダムに選ばれた左右どちらかの境界に寄せて配置される。

プールは起動時に`kfence_init()`（`mm/kfence/core.c`）で割り当てられ、回収されないようページアロケータに登録される。

### KFENCEレポートの形式 {#kfence-report-format}

```
==================================================================
BUG: KFENCE: out-of-bounds write in kstrdup+0x48/0x70

Out-of-bounds write at 0xffff888080a3c007 (1B left of kfence-#42):
 kstrdup+0x48/0x70
 driver_attach+0x18/0x40
 bus_add_driver+0x1a0/0x2a0

kfence-#42 [0xffff888080a3c000-0xffff888080a3c006, size=7, cache=kmalloc-8] allocated by task 1:
 kfence_alloc+0x7c/0xb0
 __kmalloc+0x3e/0x80
 kstrdup+0x20/0x70
 driver_register+0x30/0x60

CPU: 0 PID: 1 Comm: swapper Not tainted 6.8.0 #1
==================================================================
```

主なフィールド:

- **kfence-#42** — KFENCEのスロット番号
- **0xffff888080a3c000-0xffff888080a3c006** — オブジェクトのアドレス範囲（7バイト）
- **1B left of kfence-#42** — はみ出した距離

### KASANと比べた制約 {#limitations-vs-kasan}

KFENCEはサンプリングされた割り当てのエラーしか捕まえない。サンプリングされていない割り当てでの境界外アクセスやUAFはKFENCEからは見えない。これは意図的なもので、このトレードオフによって本番環境でも安全なオーバーヘッドが実現されている。

---

## KASANとKFENCEの比較 {#kasan-vs-kfence-comparison}

| 特性 | KASAN（generic） | KFENCE |
|---|---|---|
| 検出範囲 | すべての割り当て | サンプリング（インターバルごとに約1つ） |
| オーバーヘッド（CPU） | 約2〜3倍 | 約0%（償却） |
| オーバーヘッド（メモリ） | シャドウで約12.5% | 約2MBの固定プール |
| 見逃し | ごく少ない（8バイト粒度） | 多い（サンプリングされたオブジェクトのみ） |
| 誤検知 | なし | なし |
| 決定的か | はい | いいえ（サンプリング） |
| 本番利用 | 不可 | 可 |
| 境界外アクセスの検出 | シャドウバイトのチェック | ガードページのフォールト |
| UAFの検出 | シャドウバイト（`KASAN_SLAB_FREE`、0xFB） | ガードページ（mprotect） |
| 割り当てスタック | あり | あり |
| 解放スタック | あり | あり |
| カーネルバージョン | 4.0以降 | 5.12以降 |
| arm64のハードウェア版 | MTE（HW-tag KASAN） | なし |

### 両方を併用する {#using-both-together}

開発用カーネルでは両方を有効にする。KASANはすべてを捕まえ、KFENCEは独立した検出器を追加する。KFENCEのレポートは、KASANのオーバーヘッドが大きすぎて常時実行できない本番相当のワークロードでも得られる。

```bash
CONFIG_KASAN=y
CONFIG_KFENCE=y
# サンプリングされた割り当てではKFENCEが優先され、残りはKASANがカバーする
```

---

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [mm/kasan/report.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kasan/report.c) — `kasan_report()`と`print_report()`: 上で示したKASANレポート形式を生成するコード
- [mm/kasan/kasan.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kasan/kasan.h) — 実際のシャドウバイト定数（`KASAN_SLAB_FREE`、`KASAN_SLAB_REDZONE`など）と`kasan_alloc_meta`/`kasan_free_meta`のレイアウトを持つ内部ヘッダ
- [mm/kfence/core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kfence/core.c) — 上で参照した`__kfence_alloc()`と`kfence_allocation_gate`サンプリングカウンタ
- [mm/kfence/report.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kfence/report.c) — `kfence_report_error()`: 上で示したKFENCEレポート形式を生成するコード

### 関連ページ {#related-pages}

- [KCSAN](kcsan.md) — データ競合検出（KASANを補完する）
- [syzkaller](syzkaller.md) — KASANを使ってバグを自動的に見つけるファザー
- [Oops解析](oops-analysis.md) — KASANレポートのスタックトレースのデコード
- [mm/のKASAN](../mm/kasan.md) — シャドウメモリのレイアウト、3つのタグ付けモード、コンパイラ計装の詳細
- [mm/のKFENCE](../mm/kfence.md) — プールのレイアウト、ガードページの仕組み、サンプリング設計の詳細

### 外部資料 {#external}

- [KASAN official documentation](https://docs.kernel.org/dev-tools/kasan.html) — レポート形式のリファレンス、ブートパラメータ（`kasan.mode`、`kasan.fault`、`kasan.stacktrace`）、KUnitテストスイートの実行方法
- [KFENCE official documentation](https://docs.kernel.org/dev-tools/kfence.html) — レポート形式のリファレンス、`/sys/kernel/debug/kfence/{stats,objects}`インターフェース、ブート/sysfsのチューニング用パラメータ
