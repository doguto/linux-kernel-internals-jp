# DMA メモリの割り当て {#dma-memory-allocation}

> RAM に直接アクセスするデバイスのために、カーネルはどのようにメモリを割り当て、管理するのか

## DMA とは何か {#what-is-dma}

ダイレクトメモリアクセス（DMA）は、転送するバイトごとに CPU を介さずに、ハードウェアデバイスがシステムメモリを読み書きできるようにする仕組みである。パケットを受信するネットワークカード、ブロックを取得するディスクコントローラ、テクスチャを読み込む GPU — いずれも CPU を迂回して RAM と直接やり取りする。

これは性能にとって不可欠である。DMA がなければ、CPU は有用な仕事をする代わりに、デバイスのレジスタとメモリの間でバイトをコピーすることに時間を費やすことになる。しかし DMA は根本的な問題をもたらす。**デバイスは仮想アドレスではなく物理アドレスで動作し**、しかも物理メモリのすべてに到達できるとは限らないのである。

## なぜ DMA には特別なメモリが必要なのか {#why-dma-needs-special-memory}

3つの制約により、DMA 用のメモリ割り当ては通常の `kmalloc()` とは異なるものになる:

1. **物理アドレスの制限**: 多くのデバイスは物理メモリの一部しかアドレス指定できない。古い ISA サウンドカードは最初の16MB にしか到達できない。32ビットの PCI カードは最初の4GB にしか到達できない。カーネルがデバイスに物理アドレス8GB のバッファを渡しても、デバイスは単純にそれにアクセスできない。

2. **キャッシュコヒーレンシ**: CPU は自身と RAM の間にキャッシュ（L1、L2、L3）を使う。デバイスが DMA で RAM に書き込んだとき、CPU キャッシュにはそのアドレスの古いデータがまだ残っている可能性がある。逆に、CPU がキャッシュに書き込んだデータがまだ RAM に到達していなければ、デバイスは古いメモリを読むことになる。カーネルはこれを明示的に管理しなければならない。

3. **連続性の要件**: デバイスが見るのは仮想メモリではなく物理メモリである。カーネルの仮想アドレス空間で連続して見えるバッファも、物理ページ上では散らばっていることがある。ハードウェアの支援（IOMMU）がなければ、デバイスは物理的に連続したメモリを必要とする。

## DMA ゾーン {#dma-zones}

カーネルは、デバイスのアドレス指定の制限に対処するために、物理メモリをゾーンに分割している。[`include/linux/mmzone.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mmzone.h) より:

```c
enum zone_type {
    ZONE_DMA,       /* 最初の16MB - ISA デバイス */
    ZONE_DMA32,     /* 最初の4GB - 32ビット PCI デバイス */
    ZONE_NORMAL,    /* アドレス指定可能なすべてのメモリ */
    /* ... */
};
```

### ZONE_DMA（最初の16MB） {#zone_dma-first-16mb}

ISA バスのデバイス向けのレガシーなゾーンである。ISA バスは24ビットのアドレスを使うため、DMA は RAM の最初の 2^24 = 16MB に制限される。最近のシステムでこれが必要になることはまれだが、一部のアーキテクチャがまだ定義しているため、このゾーンは残っている。

`GFP_DMA` で割り当てる。

### ZONE_DMA32（最初の4GB） {#zone_dma32-first-4gb}

32ビットの DMA アドレスを使う32ビット PCI デバイス向けである。64GB の RAM を持つシステムでも、これらのデバイスは下位4GB にしかアクセスできない。これは今でも重要である — 多くの組み込みデバイスや古い PCIe カードは32ビットの DMA マスクを持っている。

`GFP_DMA32` で割り当てる。

### ZONE_NORMAL {#zone_normal}

カーネルと、完全なアドレス指定能力（64ビット DMA）を持つデバイスがアクセスできるメモリである。特別な制約はない。

```
物理メモリのレイアウト（64GB の RAM を持つ64ビットシステム）:

0                16MB              4GB                              64GB
|--- ZONE_DMA ---|--- ZONE_DMA32 ---|------------ ZONE_NORMAL -----------|
     ISA only        32-bit PCI              Full addressing
    (GFP_DMA)       (GFP_DMA32)             (GFP_KERNEL)
```

ゾーンの境界はアーキテクチャ固有である。ARM64 には当初 `ZONE_DMA` がまったくなかったが、コミット [1a8e1cef7603](https://git.kernel.org/linus/1a8e1cef7603)（"arm64: use both ZONE_DMA and ZONE_DMA32"）が、物理メモリの最下位1GB しかアドレス指定できないデバイスに対処するためにこれを追加した。これは、アーキテクチャ仕様が何を許すかにかかわらず、一部の ARM SoC や PCIe デバイスに存在する制約である。

## DMA API {#the-dma-api}

カーネルは、アーキテクチャや IOMMU の違いを抽象化する統一的な DMA API を提供している。API は [`include/linux/dma-mapping.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/dma-mapping.h) にあり、中核の実装は [`kernel/dma/`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma) にある。

### dma_alloc_coherent() {#dma_alloc_coherent}

CPU とデバイスの両方から同時にアクセスでき、ハードウェアによってコヒーレンシが保証される（手動でのキャッシュ管理が不要な）バッファを割り当てる。

```c
void *dma_alloc_coherent(struct device *dev, size_t size,
                         dma_addr_t *dma_handle, gfp_t gfp);
void dma_free_coherent(struct device *dev, size_t size,
                       void *cpu_addr, dma_addr_t dma_handle);
```

2つのものを返す。CPU の仮想アドレス（戻り値）と、デバイスから見える DMA アドレス（`dma_handle` 経由）である。ドライバは CPU アドレスを使い、デバイスには DMA アドレスを設定する。

**使うべきとき**: CPU とデバイスの間で共有される長寿命のバッファ — ディスクリプタリング、コマンドキュー、ファームウェアとの通信領域。コヒーレンシが保証されるため、どちらの側もキャッシュのフラッシュや無効化を行う必要がない。

**トレードオフ**: ハードウェアによるキャッシュコヒーレンシを持たないアーキテクチャ（例: 一部の ARM SoC）では、コヒーレントメモリはキャッシュ不可であることが多い。このため CPU からのアクセスが遅くなる。CPU が集中的に読み書きする大きなデータバッファには使わないこと。

### dma_map_single() {#dma_map_single}

既存のカーネルバッファを、デバイスが DMA でアクセスできるようにマップする。これは「ストリーミング」マッピングであり、方向とライフサイクルを持つ。

```c
dma_addr_t dma_map_single(struct device *dev, void *cpu_addr,
                          size_t size, enum dma_data_direction dir);
void dma_unmap_single(struct device *dev, dma_addr_t dma_addr,
                      size_t size, enum dma_data_direction dir);
```

方向は、どのキャッシュ操作が必要かをカーネルに伝える:

| 方向 | 意味 | キャッシュ操作 |
|-----------|---------|-----------------|
| `DMA_TO_DEVICE` | CPU が書き、デバイスが読む | DMA の前に CPU キャッシュをフラッシュする |
| `DMA_FROM_DEVICE` | デバイスが書き、CPU が読む | DMA の後に CPU キャッシュを無効化する |
| `DMA_BIDIRECTIONAL` | 双方向 | 前にフラッシュし、後に無効化する |

**使うべきとき**: 一時的な転送 — ネットワークパケットの送信、ディスク I/O 要求の発行。転送の前にマップし、完了したらアンマップする。

### dma_map_sg() {#dma_map_sg}

スキャッタギャザーリスト — 連続していない複数のメモリ領域 — を、単一の DMA 操作のためにマップする。カーネルは、複数のページにまたがる I/O をこの方法で処理する。

```c
int dma_map_sg(struct device *dev, struct scatterlist *sg,
               int nents, enum dma_data_direction dir);
void dma_unmap_sg(struct device *dev, struct scatterlist *sg,
                  int nents, enum dma_data_direction dir);
```

`dma_map_sg()` は DMA マップされたエントリの数を返すが、これは入力のエントリ数より *少ない* ことがある。IOMMU があれば、カーネルは物理的に不連続なページを、デバイスから見て単一の連続した DMA 領域にまとめることができる。

**使うべきとき**: ブロック I/O、大きなネットワーク転送など、データが複数のページにまたがるあらゆる場合。実際のデバイスドライバのほとんどは、スキャッタギャザーを多用している。

## コヒーレントマッピングとストリーミングマッピング {#coherent-vs-streaming-dma-mappings}

これは DMA API における中心的な設計判断である:

| 特性 | コヒーレント | ストリーミング |
|----------|----------|-----------|
| **API** | `dma_alloc_coherent()` | `dma_map_single()`、`dma_map_sg()` |
| **寿命** | 長寿命（ドライバのロードからアンロードまで） | 転送ごと（マップ、DMA、アンマップ） |
| **キャッシュの扱い** | 自動（ハードウェアまたはキャッシュ不可） | 手動（マップ/アンマップ時にフラッシュ/無効化） |
| **CPU アクセスの速度** | 遅い可能性がある（一部のアーキテクチャではキャッシュ不可） | 通常（キャッシュ可能） |
| **ユースケース** | ディスクリプタリング、コマンドキュー | データバッファ、パケット、ディスクブロック |

重要な点は次のとおりである: **コヒーレントマッピングは CPU 性能と引き換えに利便性を得ており、ストリーミングマッピングは利便性と引き換えに CPU 性能を得ている。** ほとんどのドライバは、小さな制御構造にはコヒーレントマッピングを、データにはストリーミングマッピングを使う。

### sync 操作 {#sync-operations}

マップとアンマップの間に CPU がストリーミングバッファに触れる必要がある場合は、sync 操作を使わなければならない:

```c
/* デバイスが書き込んだバッファを CPU が読みたい */
dma_sync_single_for_cpu(dev, dma_handle, size, DMA_FROM_DEVICE);
/* ... CPU がデータを読む ... */

/* バッファをデバイスに返す */
dma_sync_single_for_device(dev, dma_handle, size, DMA_FROM_DEVICE);
```

これらは sync を呼び出した時点でのキャッシュコヒーレンシを保証する。これらがなければ、CPU はキャッシュされた古いデータを読む可能性がある。

## IOMMU {#iommu}

入出力メモリ管理ユニット（IOMMU）はデバイスと物理メモリの間に位置し、デバイスから見える「I/O 仮想アドレス」（IOVA）を物理アドレスに変換する。本質的には、デバイスのためのページテーブルである。

```
IOMMU なし:
  Device --[physical addr]--> RAM
  （デバイスは実際の物理アドレスを使わなければならない）

IOMMU あり:
  Device --[IOVA]--> IOMMU --[physical addr]--> RAM
  （デバイスは仮想アドレスを使い、ハードウェアが変換する）
```

### IOMMU が可能にすること {#what-iommu-enables}

1. **スキャッタギャザーの結合**: 物理的に散らばったページを、デバイスから見て連続しているようにマップする。散らばった16ページにまたがる64KB のバッファが、デバイスには1つの連続した64KB の領域に見える。

2. **デバイスの隔離**: バグのある、あるいは悪意のあるデバイスは、その IOMMU ページテーブルに明示的にマップされたメモリ領域にしかアクセスできない。これは、仮想マシンへの安全なデバイスパススルー（VFIO）に不可欠である。

3. **アドレス指定の制限の解消**: IOMMU が高位の物理ページをデバイスの32ビットのアドレスウィンドウにマップするため、32ビットのデバイスでも4GB を超えるメモリにアクセスできる。

4. **DMA リマッピング**: Intel は VT-d、AMD は AMD-Vi と呼んでいる。どちらも PCIe デバイス向けのハードウェア IOMMU を提供する。

Linux の IOMMU ドライバは [`drivers/iommu/`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu) 以下にある。DMA API は IOMMU が存在すれば自動的にそれを使うため、ドライバがそれを知ったり気にしたりする必要はない。

### LWN の記事 {#lwn-coverage}

DMA のアドレス指定、キャッシュコヒーレンシ、デバイスの隔離に関する LWN の記事:

- [The trouble with 64-bit DMA](https://lwn.net/Articles/904210/)（2022年） -- DMA アドレスマスク、IOVA、IOMMU がデバイスアドレスをどうマップするか
- [Noncoherent DMA mappings](https://lwn.net/Articles/855328/)（2021年） -- コヒーレントマッピングとストリーミングマッピング、キャッシュコヒーレンシの管理
- [Restricted DMA](https://lwn.net/Articles/841916/)（2021年） -- IOMMU のないシステムで、SWIOTLB のバウンスバッファを使って信頼できないデバイスを隔離する

## SWIOTLB バウンスバッファ {#swiotlb-bounce-buffers}

IOMMU がなく、デバイスが物理メモリのすべてをアドレス指定できない場合、カーネルは SWIOTLB（Software I/O Translation Lookaside Buffer）— バウンスバッファの仕組み — にフォールバックする。

### 仕組み {#how-it-works}

1. カーネルはブート時に、低位メモリのバウンスバッファのプールを割り当てる（通常は4GB 未満のメモリから64MB）
2. デバイスが到達できないバッファに対して `dma_map_single()` が呼ばれると、カーネルは次のことを行う:
   - SWIOTLB プールからバウンスバッファを割り当てる
   - `DMA_TO_DEVICE` の場合: 元のバッファからバウンスバッファにデータをコピーする
   - デバイスにバウンスバッファの物理アドレスを渡す
3. `dma_unmap_single()` が呼ばれると:
   - `DMA_FROM_DEVICE` の場合: バウンスバッファから元のバッファにデータをコピーして戻す
   - バウンスバッファを解放する

```
高位メモリ（>4GB）:
+------------------+
| Original Buffer  |  <-- CPU はこれを使う
+------------------+
        |  copy
        v
低位メモリ（<4GB）:
+------------------+
| Bounce Buffer    |  <-- デバイスはここに DMA する
| (SWIOTLB pool)  |
+------------------+
```

### コスト {#the-cost}

バウンスバッファリングは **メモリコピーが2倍になる** ことを意味し、性能上の大きなペナルティとなる。これが、最近のシステムで IOMMU が強く好まれる理由である。

SWIOTLB プールのサイズは、ブート時に `swiotlb=N`（2KB スロットの数）で設定できる。実装は [`kernel/dma/swiotlb.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma/swiotlb.c) にある。

### コンフィデンシャルコンピューティングと SWIOTLB {#confidential-computing-and-swiotlb}

SWIOTLB は、コンフィデンシャルコンピューティング（AMD SEV、Intel TDX）によって再び重要性を増した。これらの環境では、デバイスの DMA は暗号化されたゲストメモリに直接アクセスできない。ゲストは、すべての DMA 操作に復号されたバウンスバッファを使わなければならない。コミット [0b84e4f8b793](https://git.kernel.org/linus/0b84e4f8b793)（"swiotlb: Add restricted DMA pool initialization"）は、こうしたシナリオに対処するために、制限付き DMA プールのサポートを追加した。

## DMA メモリプール {#dma-memory-pools}

小さな固定サイズの DMA コヒーレントバッファ（USB の転送ディスクリプタやハードウェアのコマンドエントリなど）を頻繁に割り当て・解放するドライバにとって、毎回 `dma_alloc_coherent()` を呼ぶオーバーヘッドは過大である。DMA プール API がこれを解決する。

[`include/linux/dmapool.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/dmapool.h) より:

```c
struct dma_pool *dma_pool_create(const char *name, struct device *dev,
                                  size_t size, size_t align, size_t boundary);
void dma_pool_destroy(struct dma_pool *pool);

void *dma_pool_alloc(struct dma_pool *pool, gfp_t gfp,
                     dma_addr_t *dma_handle);
void dma_pool_free(struct dma_pool *pool, void *vaddr,
                   dma_addr_t dma_handle);
```

プールは大きなコヒーレント領域を事前に割り当て、それを固定サイズのチャンクに切り分ける — slab アロケータと同じ考え方だが、DMA コヒーレントメモリ向けである。`boundary` パラメータは、割り当てが指定された2のべき乗の境界をまたがないようにする。これを必要とするハードウェアもある。

実装は [`mm/dmapool.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/dmapool.c) にある。

## よくある間違い {#common-mistakes}

### アンマップし忘れる {#forgetting-to-unmap}

すべての `dma_map_single()` には対応する `dma_unmap_single()` が必要である。すべての `dma_map_sg()` には対応する `dma_unmap_sg()` が必要である。リークしたマッピングは IOMMU のアドレス空間や SWIOTLB プールを枯渇させ、最終的に割り当ての失敗を引き起こす。

これを捕捉するには `CONFIG_DMA_API_DEBUG` を有効にする。これはすべてのマッピングを追跡し、ドライバのアンロード時に残っているものがあれば警告する:

```bash
# DMA API のデバッグ付きでブート
echo 1 > /sys/kernel/debug/dma-api/enable
```

### 誤った GFP フラグを使う {#using-wrong-gfp-flags}

ZONE_DMA（最初の16MB）のメモリが本当に必要でない限り、`GFP_DMA` を使ってはならない。32ビットの DMA 制限を持つ最近のデバイスのほとんどは、代わりに `GFP_DMA32` を必要とする。しかし理想的には、DMA API を使ってゾーンの選択を任せるべきである:

```c
/* 誤り: DMA ゾーンから手動で割り当てている */
buf = kmalloc(4096, GFP_KERNEL | GFP_DMA);
dma_handle = virt_to_phys(buf); /* 壊れている: IOMMU を迂回する! */

/* 正しい: DMA API に任せる */
buf = dma_alloc_coherent(dev, 4096, &dma_handle, GFP_KERNEL);
```

DMA API は、デバイスの DMA マスクを参照して正しいゾーンを自動的に決定する。

### キャッシュコヒーレンシのバグ {#cache-coherency-bugs}

ハードウェアがコヒーレンシを管理しないアーキテクチャでは、適切な sync 操作なしにストリーミング DMA バッファにアクセスすると、データ破損が起きる。これは多くの場合、再現が極めて難しい断続的な障害として現れる:

```c
/* バグ: sync せずにデバイスのデータを読んでいる */
dma_handle = dma_map_single(dev, buf, len, DMA_FROM_DEVICE);
start_device_dma(dev, dma_handle, len);
wait_for_dma_complete(dev);
memcpy(dest, buf, len);  /* キャッシュされた古いデータを読むかもしれない! */
dma_unmap_single(dev, dma_handle, len, DMA_FROM_DEVICE);

/* 正しい: 読む前にアンマップする（アンマップが sync を行う） */
dma_handle = dma_map_single(dev, buf, len, DMA_FROM_DEVICE);
start_device_dma(dev, dma_handle, len);
wait_for_dma_complete(dev);
dma_unmap_single(dev, dma_handle, len, DMA_FROM_DEVICE);
memcpy(dest, buf, len);  /* 安全: アンマップがキャッシュを無効化した */
```

### DMA に virt_to_phys() を使う {#using-virt_to_phys-for-dma}

仮想アドレスを物理アドレスに変換してデバイスに渡してはならない。これは IOMMU を迂回し、バウンスバッファリングをスキップし、デバイスの DMA マスクを無視する。常に DMA API を使うこと。

### スタックやモジュールのメモリへの DMA {#dma-to-stack-or-module-memory}

スタックのメモリやモジュールのテキストは vmalloc 空間に存在することがあり、物理的に連続していることも DMA でアドレス指定可能であることも保証されない。DMA バッファには常に `kmalloc()` か `dma_alloc_coherent()` を使うこと。

## 試してみる {#try-it-yourself}

### DMA ゾーンのサイズを確認する {#check-dma-zone-sizes}

```bash
# ゾーン情報を表示
cat /proc/zoneinfo | grep -E "^Node|pages free|managed|zone"

# ゾーンの簡潔な表示
cat /proc/buddyinfo
# 出力例:
# Node 0, zone      DMA      1    1    0    0    2    1    1    0    1    1    3
# Node 0, zone    DMA32   2907 2312  975  574  256   94   40   16    6    5  252
# Node 0, zone   Normal  19374 7834 5327 2581  784  168   30    5    0    0    0
```

### IOMMU の状態を調べる {#inspect-iommu-state}

```bash
# IOMMU が有効かどうかを確認
dmesg | grep -i iommu

# IOMMU グループを表示（各グループがアドレス空間の隔離を共有する）
ls /sys/kernel/iommu_groups/
# 各グループのディレクトリには、PCI デバイスを列挙する devices/ サブディレクトリがある

# 特定のデバイスの DMA マスクを確認
# （カーネルモジュール内から、または debugfs 経由で）
cat /sys/bus/pci/devices/0000:03:00.0/dma_mask_bits
```

### SWIOTLB の使用状況を監視する {#monitor-swiotlb-usage}

```bash
# SWIOTLB プールのサイズと使用量を確認
cat /sys/kernel/debug/swiotlb/io_tlb_nslabs  2>/dev/null
cat /sys/kernel/debug/swiotlb/io_tlb_used    2>/dev/null

# あるいはブート時の dmesg から
dmesg | grep -i swiotlb
# 例: "software IO TLB: mapped [mem 0x00000000b3c00000-0x00000000b7c00000] (64MB)"
```

### DMA API のデバッグチェックを有効にする {#enable-dma-api-debug-checking}

```bash
# CONFIG_DMA_API_DEBUG=y でカーネルをビルド
# その後、ブート時または実行時に:
echo 1 > /sys/kernel/debug/dma-api/enable

# DMA API のデバッグ統計を表示
cat /sys/kernel/debug/dma-api/num_errors
cat /sys/kernel/debug/dma-api/num_free_entries
cat /sys/kernel/debug/dma-api/min_free_entries
```

### DMA プールの統計を表示する {#view-dma-pool-statistics}

```bash
# デバイスごとの DMA プール情報
cat /sys/kernel/debug/dma_pools 2>/dev/null

# あるいは sysfs から（カーネル設定による）
find /sys/kernel/debug/ -name "*dma*" 2>/dev/null
```

## 主要なソースファイル {#key-source-files}

| ファイル | 説明 |
|------|-------------|
| [`kernel/dma/mapping.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma/mapping.c) | DMA マッピングの中核実装 |
| [`kernel/dma/direct.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma/direct.c) | ダイレクト DMA（IOMMU なし）の実装 |
| [`kernel/dma/swiotlb.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma/swiotlb.c) | SWIOTLB バウンスバッファの実装 |
| [`include/linux/dma-mapping.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/dma-mapping.h) | DMA API のヘッダ |
| [`include/linux/dmapool.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/dmapool.h) | DMA プール API のヘッダ |
| [`mm/dmapool.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/dmapool.c) | DMA プールの実装 |
| [`drivers/iommu/`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu) | IOMMU ドライバの実装 |
| [`include/linux/mmzone.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mmzone.h) | ゾーン種別の定義 |

---

## 参考資料 {#references}

### 参考文献 {#further-reading}

- [DMA API Guide](https://docs.kernel.org/core-api/dma-api.html) -- DMA API に関する正式なカーネルドキュメント
- [DMA API HOWTO](https://docs.kernel.org/core-api/dma-api-howto.html) -- ドライバ開発者向けの実践的なガイド
- [Dynamic DMA mapping Guide](https://docs.kernel.org/core-api/dma-api-howto.html) -- 各マッピング種別をいつ、どう使うか
- [LWN: The trouble with 64-bit DMA](https://lwn.net/Articles/904210/)（2022年） -- DMA アドレスマスク、IOVA、IOMMU
- [LWN: Noncoherent DMA mappings](https://lwn.net/Articles/855328/)（2021年） -- コヒーレントマッピングとストリーミングマッピング、キャッシュコヒーレンシ
- [LWN: Restricted DMA](https://lwn.net/Articles/841916/)（2021年） -- SWIOTLB のバウンスバッファで信頼できないデバイスを隔離する

### 関連項目 {#related}

- [Page Allocator](page-allocator.md) -- DMA メモリの最終的な供給元
- [vmalloc](vmalloc.md) -- 仮想メモリの割り当て（DMA には適さない）
- [NUMA](numa.md) -- NUMA を考慮した割り当てと、DMA ゾーンとの相互作用
- [Overview](overview.md) -- メモリ管理の階層の中で DMA の割り当てがどこに位置するか

## 参考文献 {#further-reading_1}

- [Kernel docs: DMA API](https://docs.kernel.org/core-api/dma-api.html) — DMA API のすべての関数に関する正式なリファレンス
- [Kernel docs: DMA API HOWTO](https://docs.kernel.org/core-api/dma-api-howto.html) — 各マッピング種別をいつ、どう使うかを扱う実践的なガイド
- [`Documentation/core-api/dma-api.rst`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/core-api/dma-api.rst) — DMA API ドキュメントのカーネルソース
- [`kernel/dma/`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma) — DMA サブシステム全体のソースディレクトリ: mapping、direct、swiotlb、pool
- [LWN: The trouble with 64-bit DMA](https://lwn.net/Articles/904210/) — DMA アドレスマスク、IOVA、IOMMU がデバイスアドレスをどうマップするか（2022年）
- [LWN: Noncoherent DMA mappings](https://lwn.net/Articles/855328/) — コヒーレント DMA マッピングとストリーミング DMA マッピング、キャッシュコヒーレンシの管理（2021年）
- [LWN: Restricted DMA](https://lwn.net/Articles/841916/) — IOMMU なしで、SWIOTLB のバウンスバッファを使って信頼できないデバイスを隔離する（2021年）
- [Device Memory Coherency](device-coherency.md) — すべての DMA マッピングを支配するキャッシュ保守のルール
- [CMA](cma.md) — `dma_alloc_coherent()` が Contiguous Memory Allocator を介して物理的に連続したメモリをどう確保するか
