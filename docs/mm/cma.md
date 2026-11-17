# CMA（Contiguous Memory Allocator） {#cma-contiguous-memory-allocator}

> 二役をこなす予約領域: 普段は通常のページ、必要なときには DMA バッファ

## CMA とは何か {#what-is-cma}

CMA は、ブート時に物理メモリの領域を予約しておき、デバイスが必要とするときにその領域を回収して大きな連続割り当てに使えるようにする仕組みである。重要なのは、これらの予約ページが無駄にならないという点である。デバイスドライバが連続バッファを要求するまでは、予約ページは通常の移動可能な割り当て（ページキャッシュ、匿名メモリ）に使われ、要求が来た時点でページが移動（マイグレーション）され、領域がドライバに渡される。

```
CMA 領域（アイドル時）:
┌────────────────────────────────────────────────────────────────┐
│  Page   Page   Page   Page   Page   Page   Page   Page         │
│  cache  anon   cache  anon   cache  anon   cache  cache        │
│  (movable allocations using the CMA region normally)           │
└────────────────────────────────────────────────────────────────┘

DMA 割り当て要求が到着:
┌────────────────────────────────────────────────────────────────┐
│  Migrate all movable pages out...                              │
│  ← pages moved to other free memory                            │
└────────────────────────────────────────────────────────────────┘

CMA 領域（デバイスに割り当て済み）:
┌────────────────────────────────────────────────────────────────┐
│              Contiguous DMA buffer for device                  │
└────────────────────────────────────────────────────────────────┘
```

## なぜ CMA が存在するのか {#why-cma-exists}

### 問題 {#the-problem}

DMA デバイスは、物理的に連続したメモリを必要とすることが多い。CPU と違い、多くのデバイスはページテーブルを使って散在する物理ページを連続したアドレス範囲にリマップすることができない。フレームをキャプチャするカメラ、フレームバッファをスキャンアウトするディスプレイコントローラ、ジャンボパケットを送信するネットワークカードは、いずれも単一の連続した物理バッファを必要としうる。

起動直後のシステムでは、大きな連続割り当ては簡単に成功する。しかし何日も稼働した後では、物理メモリが断片化する — 空きページが、長期間残る割り当ての間に散らばってしまう。何ギガバイトも空いていても、連続した8MB のバッファの割り当てが失敗することがある。

### 以前の解決策とその欠点 {#previous-solutions-and-their-downsides}

CMA 以前は、選択肢は貧弱だった:

| 手法 | 欠点 |
|----------|----------|
| ブート時の予約（`memblock_reserve`） | メモリが排他的に確保され、デバイスがアイドルのときは無駄になる |
| 実行時の高次 `alloc_pages` | 断片化した状態では失敗する |
| 割り当て前のコンパクション | 低速で、非常に大きなバッファには信頼できない |
| IOMMU によるリマップ | すべてのプラットフォームに IOMMU があるわけではない。複雑さとレイテンシが増す |

ブート時の予約という手法は、RAM が乏しい組み込みデバイス（スマートフォン、セットトップボックス）では特に無駄が多かった。1日に数分しか使わないカメラのために64MB を予約し、そのメモリがシステムの他の部分から使えなくなる、ということが起こりうる。

### CMA の着想 {#cmas-insight}

CMA は、信頼性と無駄の間のトレードオフを解消する。ブート時に領域を予約する（連続した空間が利用可能であることを保証する）一方で、それまでの間は移動可能な割り当てにその空間を使わせる。デバイスがメモリを必要とすると、CMA は移動可能なページを移動させて、連続した領域を引き渡す。

この二重利用の設計は、Samsung の ARM ベースのモバイルデバイスにとって特に重要だった。限られた RAM と、DMA に依存する多数の周辺機器（カメラ、ディスプレイ、コーデック）のせいで、デバイスドライバの要求とアプリケーションのメモリの間に絶えず緊張関係が生じていたからである。

## CMA の仕組み {#how-cma-works}

### 二重利用の設計 {#the-dual-use-design}

CMA 領域は `ZONE_NORMAL`（アーキテクチャによっては `ZONE_DMA`/`ZONE_DMA32`）の中に存在する。buddy アロケータは CMA ページを特別なマイグレートタイプ `MIGRATE_CMA` として扱う。

```
CMA ありのゾーンレイアウト:
┌──────────────────────────────────────────────────────────────────────┐
│                           ZONE_NORMAL                                │
│                                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  ┌───────────┐     │
│  │ MOVABLE  │  │UNMOVABLE │  │   MIGRATE_CMA    │  │RECLAIMABLE│     │
│  │          │  │          │  │  (CMA region)    │  │           │     │
│  └──────────┘  └──────────┘  └──────────────────┘  └───────────┘     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

`MIGRATE_CMA` ページのルール:

1. **移動可能な割り当ては CMA ページを自由に使える。** ページアロケータは `MIGRATE_MOVABLE` の要求を処理する際、CMA の pageblock にフォールバックする。
2. **移動不可能な割り当てと回収可能な割り当ては CMA ページを使えない。** これは極めて重要である — 移動不可能なカーネルの割り当てが CMA 領域に置かれてしまうと、その領域は二度と完全には回収できなくなる。
3. **ドライバが `cma_alloc()` を呼ぶと、要求された範囲内のすべての移動可能なページが移動される。** カーネルはコンパクションと同じページマイグレーションの仕組みを使う。

### 割り当てパス {#allocation-path}

ドライバが連続メモリを必要とするとき（通常は `dma_alloc_coherent()` を通じて）:

```
ドライバが dma_alloc_coherent(dev, size, ...) を呼ぶ
        │
        ▼
DMA サブシステムがこのデバイス用の CMA 領域を選ぶ
        │
        ▼
cma_alloc(cma, count, align, no_warn)
        │
        ▼
CMA 領域内で適切なページ範囲を探す
        │
        ▼
alloc_contig_range(start, end, MIGRATE_CMA, ...)
        │
        ├─── 範囲を隔離する（新たな割り当てを防ぐ）
        │
        ├─── 範囲内のすべての移動可能なページを移動させる
        │         コンパクションと同じマイグレーションのコードを使う
        │
        ├─── per-cpu のページリストを排出する
        │
        └─── 連続したページを呼び出し元に返す
```

マイグレーションが失敗した場合（例: ページが I/O のためにピン留めされている場合）、`cma_alloc()` は CMA 領域内の別の範囲で再試行する。完全に空けられる範囲が見つかるか諦めるまで、ビットマップで追跡されるチャンク単位で領域をスキャンする。

### CMA のビットマップ {#the-cma-bitmap}

各 CMA 領域は、割り当て状態をビットマップで追跡する。各ビットはページのブロックを表す（粒度は `CONFIG_CMA_ALIGNMENT` で設定され、デフォルトは order-8、4KB ページで1MB）:

```c
struct cma {
    unsigned long   base_pfn;       /* CMA 領域の先頭 */
    unsigned long   count;          /* 領域内の総ページ数 */
    unsigned long   *bitmap;        /* 割り当てビットマップ */
    unsigned int    order_per_bit;  /* 1ビットあたりのページ数（2のべき乗） */
    spinlock_t      lock;
    ...
};
```

ビットマップは、どのチャンクが現在デバイスに割り当てられているかを追跡する。ビットマップで割り当て済みとマークされていないページは、システムの他の部分が移動可能な用途に使える。

### ページアロケータとの相互作用 {#interaction-with-the-page-allocator}

CMA は、CMA と同時に導入された `MIGRATE_CMA` マイグレートタイプを通じて buddy アロケータと統合されている。主な相互作用は次のとおり:

- **フォールバックの挙動**: buddy アロケータは、`MIGRATE_MOVABLE` の要求を移動可能なフリーリストから満たせない場合、`MIGRATE_CMA` ページにフォールバックする。これにより、通常の運用中に CMA 領域が移動可能なページで埋まっていく。
- **奪取の防止**: アロケータは、`MIGRATE_UNMOVABLE` や `MIGRATE_RECLAIMABLE` の要求のために CMA の pageblock を奪うことは決してない。この保証は `mm/page_alloc.c` の `__rmqueue_fallback()` で強制されている。
- **コンパクションによる考慮**: コンパクションは実行時に CMA 領域を理解しており、そのマイグレートタイプを尊重する。CMA ページはコンパクションのマイグレーションスキャナの移動先になりうる（ページを CMA 領域 *へ* 移動できる）が、CMA ページが移動不可能なタイプに変更されることはない。

## 設定 {#configuration}

### カーネルコマンドライン {#kernel-command-line}

CMA を設定する最も簡単な方法:

```bash
# デフォルトのグローバル CMA 領域として256MB を予約
cma=256M

# 配置のヒント（ベースアドレス）付き
cma=256M@0x40000000
```

`cma=` パラメータはデフォルトの CMA 領域のサイズを設定する。この領域は、専用の CMA 領域を持たないすべてのデバイスによって使われる。

### Device Tree（ARM/組み込み） {#device-tree-armembedded}

組み込みプラットフォームでは、通常 Device Tree で CMA 領域を定義する:

```dts
reserved-memory {
    #address-cells = <2>;
    #size-cells = <2>;
    ranges;

    /* デフォルトの CMA 領域 */
    linux,cma {
        compatible = "shared-dma-pool";
        reusable;
        size = <0 0x10000000>;  /* 256MB */
        linux,cma-default;
    };

    /* 特定デバイス専用の領域 */
    camera_mem: camera-buffer {
        compatible = "shared-dma-pool";
        reusable;
        reg = <0 0x78000000 0 0x8000000>;  /* 特定のアドレスに128MB */
    };
};

camera@0 {
    memory-region = <&camera_mem>;
};
```

静的な予約ではなく CMA にしているのは `reusable` プロパティである。`reusable` がなければ、その領域は排他的に予約され、移動可能な割り当てには使えなくなる。

### カーネル設定オプション {#kernel-config-options}

```
CONFIG_CMA=y                    # CMA サポートを有効化
CONFIG_CMA_DEBUG=y              # 追加のデバッグチェック（開発時のみ）
CONFIG_CMA_DEBUGFS=y            # 領域ごとの統計を debugfs で公開
CONFIG_CMA_SIZE_MBYTES=16       # デフォルトの CMA サイズ（MB 単位）
CONFIG_CMA_SIZE_PERCENTAGE=0    # デフォルトの CMA サイズ（RAM に対する割合）
CONFIG_CMA_ALIGNMENT=8          # 最小アラインメント（order）。デフォルト8 = 256ページ = 1MB
CONFIG_DMA_CMA=y                # DMA の連続アロケータのバックエンドとして CMA を使う
```

`CONFIG_CMA_SIZE_MBYTES` は、`cma=` ブートパラメータが与えられないときのデフォルトサイズを設定する。`CONFIG_CMA_SIZE_PERCENTAGE` は、CMA を RAM サイズに応じてスケーリングする別の方法を提供する。2つの値のうち大きいほうが採用される。

## ドライバから CMA を使う {#using-cma-from-drivers}

### DMA API（推奨） {#the-dma-api-recommended}

ほとんどのドライバは DMA API を使うべきである。DMA API は CMA からの割り当てを透過的に扱う:

```c
#include <linux/dma-mapping.h>

/* 連続した DMA バッファを割り当てる -- CONFIG_DMA_CMA=y なら CMA を使う */
void *vaddr = dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL);

/* 使い終わったら解放する */
dma_free_coherent(dev, size, vaddr, dma_handle);
```

DMA サブシステムは `dma_alloc_from_contiguous()` を介して CMA を呼び出し、これは内部で `cma_alloc()` に対応付けられる。デバイスの CMA 領域は、Device Tree の `memory-region` プロパティ、またはデフォルトのグローバル CMA 領域に基づいて選ばれる。

### CMA API の直接利用（まれ） {#direct-cma-api-rare}

直接制御が必要なサブシステム向け:

```c
#include <linux/cma.h>

/* 特定の CMA 領域から、1 << align にアラインされた count ページを割り当てる */
struct page *page = cma_alloc(cma, count, align, no_warn);

/* CMA に返却する */
cma_release(cma, page, count);
```

`cma_release()` の後、ページは `MIGRATE_CMA` ページとして buddy アロケータに戻り、再び移動可能な割り当てに使えるようになる。

## 監視 {#monitoring}

### /proc/meminfo {#procmeminfo}

```bash
$ grep Cma /proc/meminfo
CmaTotal:        262144 kB    # CMA の予約総量
CmaFree:         245760 kB    # デバイスに割り当てられていない CMA ページ
```

`CmaFree` は、CMA 領域のうち現在デバイスドライバが保持していない量を示す。これらのページは移動可能な割り当て（ページキャッシュ、匿名メモリ）に使われている可能性が高いが、デバイスが必要とするときには回収できる。

`CmaTotal` と `CmaFree` はそれぞれ `MemTotal` と `MemFree` に含まれていることに注意。システムから見て、CMA メモリが「消えている」わけではない。

### debugfs（CONFIG_CMA_DEBUGFS） {#debugfs-configcmadebugfs}

```bash
$ ls /sys/kernel/debug/cma/
cma-reserved/

$ ls /sys/kernel/debug/cma/cma-reserved/
alloc  base_pfn  bitmap  count  free  maxchunk  order_per_bit  used

$ cat /sys/kernel/debug/cma/cma-reserved/base_pfn
262144
$ cat /sys/kernel/debug/cma/cma-reserved/count
65536        # 領域内の総ページ数
$ cat /sys/kernel/debug/cma/cma-reserved/used
0            # 現在デバイスに割り当てられているページ数
$ cat /sys/kernel/debug/cma/cma-reserved/maxchunk
65536        # 最大の連続空きチャンク（ページ単位）
```

`alloc` ファイルは書き込み可能であり、テスト割り当てを発生させて CMA が動作していることを確認できる:

```bash
# CMA 領域から1024ページ（4MB）をテスト割り当てする
echo 1024 > /sys/kernel/debug/cma/cma-reserved/alloc
```

### vmstat カウンタ {#vmstat-counters}

```bash
$ grep cma /proc/vmstat
cma_alloc_success 42
cma_alloc_fail 0
```

これらのカウンタ（v5.19 で追加）は、すべての CMA 領域にわたる CMA 割り当ての結果を追跡する。

## 試してみる {#try-it-yourself}

```bash
# CMA が有効か、どれだけ予約されているかを確認
grep Cma /proc/meminfo

# CMA のカーネルブートパラメータを表示
cat /proc/cmdline | tr ' ' '\n' | grep cma

# debugfs が使えるなら CMA 領域を調べる
ls /sys/kernel/debug/cma/ 2>/dev/null

# CMA 割り当ての統計を見る（カーネル 5.19+）
grep cma /proc/vmstat

# buddyinfo で order ごとの空きページ数を確認
# 高い order でカウントが少ないのは、CMA が回避に役立つ断片化を示唆する
cat /proc/buddyinfo

# 組み込みデバイスでは、Device Tree の CMA 設定を表示
# （dtc が必要。/proc/device-tree も使える場合がある）
ls /proc/device-tree/reserved-memory/ 2>/dev/null
```

## 歴史 {#history}

### Samsung での開発 {#development-at-samsung}

CMA は Samsung Electronics の Marek Szyprowski によって開発された。動機は ARM モバイルプラットフォーム（Samsung の Exynos SoC など）から来ている。そこでは複数の周辺機器 — カメラ、ディスプレイコントローラ、マルチメディアコーデック、GPU — がいずれも大きな連続バッファを必要としていたが、システムには静的な予約に割ける RAM が限られていた。

この機能は、マージされるまでにパッチシリーズの複数のリビジョンにわたって広範なレビューを受けた。

### v3.5 でマージ（2012年） {#merged-in-v35-2012}

**コミット**: [c64be2bb1c6e](https://git.kernel.org/linus/c64be2bb1c6e)（"drivers: add Contiguous Memory Allocator"）

**作者**: Marek Szyprowski

最初のマージには、CMA アロケータの中核と DMA サブシステムとの統合が含まれていた。

**LWN の記事**: [A deep dive into CMA](https://lwn.net/Articles/486301/) -- 設計の根拠とレビューの過程を論じている。

### MIGRATE_CMA タイプ（v3.5、2012年） {#migrate_cma-type-v35-2012}

**コミット**: [47118af076f6](https://git.kernel.org/linus/47118af076f6)（"mm: mmzone: MIGRATE_CMA migration type added"）

**作者**: Marek Szyprowski

buddy アロケータに `MIGRATE_CMA` マイグレートタイプを追加した。これにより、CMA ページが移動可能な割り当てに使われつつ、移動不可能な割り当てが CMA 領域に置かれるのを防ぐという二重利用の設計が可能になった。

### デバイスごとの CMA 領域（v3.17、2014年） {#per-device-cma-areas-v317-2014}

**コミット**: [c1f733aaf1e1](https://git.kernel.org/linus/c1f733aaf1e1)（"drivers: of: add initialization code for dma-reserved-memory"）

Device Tree の `memory-region` プロパティのサポートで、個々のデバイスがグローバルな領域を共有するのではなく、専用の CMA 領域を持てるようになった。

### CMA の debugfs（v4.1、2015年） {#cma-debugfs-v41-2015}

**コミット**: [28b24c1fc8c4](https://git.kernel.org/linus/28b24c1fc8c4)（"mm: cma: debugfs interface"）

**作者**: Sasha Levin

CMA を監視するための debugfs インターフェースが追加され、運用者が CMA 領域の使用状況を把握できるようになった。

## 主要なソースファイル {#key-source-files}

| ファイル | 説明 |
|------|-------------|
| [`mm/cma.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/cma.c) | CMA アロケータの中核: `cma_alloc()`、`cma_release()`、ビットマップ管理 |
| [`mm/cma.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/cma.h) | CMA の内部ヘッダ（`struct cma` の定義） |
| [`include/linux/cma.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/cma.h) | ドライバ向けの公開 CMA API |
| [`mm/cma_debug.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/cma_debug.c) | CMA 領域の debugfs インターフェース |
| [`kernel/dma/contiguous.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma/contiguous.c) | DMA サブシステムとの統合: `dma_alloc_from_contiguous()` |
| [`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) | `MIGRATE_CMA` のフォールバックロジックを持つ buddy アロケータ |
| [`mm/page_isolation.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_isolation.c) | `cma_alloc()` の間に使われるページ範囲の隔離 |

## よくある問題 {#common-issues}

### CMA 割り当ての失敗 {#cma-allocation-failures}

**症状**: `cma_alloc_fail` のカウントが増える、ドライバの probe が失敗する、dmesg に DMA 割り当てエラーが出る。

**よくある原因**:

- **CMA 領域内のピン留めされたページ**: I/O 中のページや `get_user_pages()` で保持されているページは移動できない。これは CMA 割り当て失敗の最も一般的な原因である。長期間のピン留めと CMA の移動可能性がなぜ根本的に相容れないのかについては、[Getting User Pages (GUP)](gup.md#foll_longterm-the-pin-that-fights-the-whole-memory-manager) を参照。
- **CMA 領域が小さすぎる**: 領域は、単一の最大の連続割り当てに加えて、他のデバイスからの同時割り当ても収容できなければならない。
- **CMA 内の断片化**: さまざまなサイズのデバイス割り当てが確保・解放を繰り返すと、CMA のビットマップ自体が断片化しうる（CMA 領域内の外部断片化）。

**診断**:

```bash
# CMA の失敗を確認
grep cma /proc/vmstat
dmesg | grep -i cma

# 利用可能な CMA 空間を確認
grep Cma /proc/meminfo

# 詳細な領域の状態は debugfs で
cat /sys/kernel/debug/cma/*/used
cat /sys/kernel/debug/cma/*/maxchunk
```

### CMA メモリが現れない {#cma-memory-not-appearing}

`/proc/meminfo` の `CmaTotal` が0の場合:

- カーネル設定で `CONFIG_CMA=y` と `CONFIG_DMA_CMA=y` になっていることを確認する
- `cma=` ブートパラメータがあること、または `CONFIG_CMA_SIZE_MBYTES` が非ゼロであることを確認する
- Device Tree のプラットフォームでは、reserved-memory ノードに `compatible = "shared-dma-pool"` と `reusable` プロパティがあることを確認する

## 参考資料 {#references}

- [LWN: A deep dive into CMA](https://lwn.net/Articles/486301/) - マージウィンドウ時の設計に関する議論
- [LWN: CMA and compaction](https://lwn.net/Articles/684611/) - CMA とコンパクションの相互作用
- [LWN: Contiguous memory allocation for drivers](https://lwn.net/Articles/396657/) - 初期の CMA の提案
- [Kernel docs: DMA API](https://docs.kernel.org/core-api/dma-api.html) - ドライバが DMA メモリをどう割り当てるべきか
- [contiguous-memory](contiguous-memory.md) - CMA が解決する断片化の問題
- [compaction](compaction.md) - CMA が依存するページマイグレーションの仕組み
- [page-allocator](page-allocator.md) - buddy アロケータとマイグレートタイプ

## 参考文献 {#further-reading}

- [LWN: A deep dive into CMA](https://lwn.net/Articles/486301/) — 設計の根拠と、v3.5（2012年）で CMA がマージされるまでの長いレビュー過程
- [LWN: Contiguous memory allocation for drivers](https://lwn.net/Articles/396657/) — 最終的な設計の動機となった初期の CMA の提案（2010年）
- [LWN: CMA and compaction](https://lwn.net/Articles/684611/) — CMA のマイグレーションとメモリコンパクションの相互作用（2016年）
- [`Documentation/admin-guide/mm/dma-api-howto.rst`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/admin-guide/mm/dma-api-howto.rst) — CMA を使うパスを含め、ドライバが DMA メモリをどう割り当てるべきかを扱うカーネルドキュメント
- [`mm/cma.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/cma.c) — CMA アロケータの中核: ビットマップ管理、`cma_alloc()`、`cma_release()`
- [`kernel/dma/contiguous.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma/contiguous.c) — `dma_alloc_coherent()` と `cma_alloc()` の橋渡し
- [DMA Memory Allocation](dma.md) — ドライバが透過的に CMA に到達するために使う DMA API
- [contiguous-memory](contiguous-memory.md) — CMA が解決するために設計された断片化の問題
- [memory-reservation](memory-reservation.md) — ブート時に memblock を介して CMA 領域が物理メモリからどう切り出されるか
