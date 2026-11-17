# DMAメモリの割り当て {#dma-memory-allocation}

> RAMに直接アクセスするデバイスのために、カーネルはどのようにメモリを割り当て、管理するのか

## DMAとは何か {#what-is-dma}

ダイレクトメモリアクセス（DMA）は、転送するバイトごとにCPUを介さずに、ハードウェアデバイスがシステムメモリを読み書きできるようにする仕組みである。パケットを受信するネットワークカード、ブロックを取得するディスクコントローラ、テクスチャを読み込むGPU — いずれもCPUを迂回してRAMと直接やり取りする。

これは性能にとって不可欠である。DMAがなければ、CPUは有用な仕事をする代わりに、デバイスのレジスタとメモリの間でバイトをコピーすることに時間を費やすことになる。しかしDMAは根本的な問題をもたらす。**デバイスは仮想アドレスではなく物理アドレスで動作し**、しかも物理メモリのすべてに到達できるとは限らないのである。

## なぜDMAには特別なメモリが必要なのか {#why-dma-needs-special-memory}

3つの制約により、DMA用のメモリ割り当ては通常の `kmalloc()` とは異なるものになる:

1. **物理アドレスの制限**: 多くのデバイスは物理メモリの一部しかアドレス指定できない。古いISAサウンドカードは最初の16MBにしか到達できない。32ビットのPCIカードは最初の4GBにしか到達できない。カーネルがデバイスに物理アドレス8GBのバッファを渡しても、デバイスは単純にそれにアクセスできない。

2. **キャッシュコヒーレンシ**: CPUは自身とRAMの間にキャッシュ（L1、L2、L3）を使う。デバイスがDMAでRAMに書き込んだとき、CPUキャッシュにはそのアドレスの古いデータがまだ残っている可能性がある。逆に、CPUがキャッシュに書き込んだデータがまだRAMに到達していなければ、デバイスは古いメモリを読むことになる。カーネルはこれを明示的に管理しなければならない。

3. **連続性の要件**: デバイスが見るのは仮想メモリではなく物理メモリである。カーネルの仮想アドレス空間で連続して見えるバッファも、物理ページ上では散らばっていることがある。ハードウェアの支援（IOMMU）がなければ、デバイスは物理的に連続したメモリを必要とする。

## DMAゾーン {#dma-zones}

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

ISAバスのデバイス向けのレガシーなゾーンである。ISAバスは24ビットのアドレスを使うため、DMAはRAMの最初の2^24 = 16MBに制限される。最近のシステムでこれが必要になることはまれだが、一部のアーキテクチャがまだ定義しているため、このゾーンは残っている。

`GFP_DMA` で割り当てる。

### ZONE_DMA32（最初の4GB） {#zone_dma32-first-4gb}

32ビットのDMAアドレスを使う32ビットPCIデバイス向けである。64GBのRAMを持つシステムでも、これらのデバイスは下位4GBにしかアクセスできない。これは今でも重要である — 多くの組み込みデバイスや古いPCIeカードは32ビットのDMAマスクを持っている。

`GFP_DMA32` で割り当てる。

### ZONE_NORMAL {#zone_normal}

カーネルと、完全なアドレス指定能力（64ビットDMA）を持つデバイスがアクセスできるメモリである。特別な制約はない。

```
物理メモリのレイアウト（64GB の RAM を持つ64ビットシステム）:

0                16MB              4GB                              64GB
|--- ZONE_DMA ---|--- ZONE_DMA32 ---|------------ ZONE_NORMAL -----------|
     ISA only        32-bit PCI              Full addressing
    (GFP_DMA)       (GFP_DMA32)             (GFP_KERNEL)
```

ゾーンの境界はアーキテクチャ固有である。ARM64には当初 `ZONE_DMA` がまったくなかったが、コミット [1a8e1cef7603](https://git.kernel.org/linus/1a8e1cef7603)（"arm64: use both ZONE_DMA and ZONE_DMA32"）が、物理メモリの最下位1GBしかアドレス指定できないデバイスに対処するためにこれを追加した。これは、アーキテクチャ仕様が何を許すかにかかわらず、一部のARM SoCやPCIeデバイスに存在する制約である。

## DMA API {#the-dma-api}

カーネルは、アーキテクチャやIOMMUの違いを抽象化する統一的なDMA APIを提供している。APIは [`include/linux/dma-mapping.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/dma-mapping.h) にあり、中核の実装は [`kernel/dma/`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma) にある。

### dma_alloc_coherent() {#dma_alloc_coherent}

CPUとデバイスの両方から同時にアクセスでき、ハードウェアによってコヒーレンシが保証される（手動でのキャッシュ管理が不要な）バッファを割り当てる。

```c
void *dma_alloc_coherent(struct device *dev, size_t size,
                         dma_addr_t *dma_handle, gfp_t gfp);
void dma_free_coherent(struct device *dev, size_t size,
                       void *cpu_addr, dma_addr_t dma_handle);
```

2つのものを返す。CPUの仮想アドレス（戻り値）と、デバイスから見えるDMAアドレス（`dma_handle` 経由）である。ドライバはCPUアドレスを使い、デバイスにはDMAアドレスを設定する。

**使うべきとき**: CPUとデバイスの間で共有される長寿命のバッファ — ディスクリプタリング、コマンドキュー、ファームウェアとの通信領域。コヒーレンシが保証されるため、どちらの側もキャッシュのフラッシュや無効化を行う必要がない。

**トレードオフ**: ハードウェアによるキャッシュコヒーレンシを持たないアーキテクチャ（例: 一部のARM SoC）では、コヒーレントメモリはキャッシュ不可であることが多い。このためCPUからのアクセスが遅くなる。CPUが集中的に読み書きする大きなデータバッファには使わないこと。

### dma_map_single() {#dma_map_single}

既存のカーネルバッファを、デバイスがDMAでアクセスできるようにマップする。これは「ストリーミング」マッピングであり、方向とライフサイクルを持つ。

```c
dma_addr_t dma_map_single(struct device *dev, void *cpu_addr,
                          size_t size, enum dma_data_direction dir);
void dma_unmap_single(struct device *dev, dma_addr_t dma_addr,
                      size_t size, enum dma_data_direction dir);
```

方向は、どのキャッシュ操作が必要かをカーネルに伝える:

| 方向 | 意味 | キャッシュ操作 |
|-----------|---------|-----------------|
| `DMA_TO_DEVICE` | CPUが書き、デバイスが読む | DMAの前にCPUキャッシュをフラッシュする |
| `DMA_FROM_DEVICE` | デバイスが書き、CPUが読む | DMAの後にCPUキャッシュを無効化する |
| `DMA_BIDIRECTIONAL` | 双方向 | 前にフラッシュし、後に無効化する |

**使うべきとき**: 一時的な転送 — ネットワークパケットの送信、ディスクI/O要求の発行。転送の前にマップし、完了したらアンマップする。

### dma_map_sg() {#dma_map_sg}

スキャッタギャザーリスト — 連続していない複数のメモリ領域 — を、単一のDMA操作のためにマップする。カーネルは、複数のページにまたがるI/Oをこの方法で処理する。

```c
int dma_map_sg(struct device *dev, struct scatterlist *sg,
               int nents, enum dma_data_direction dir);
void dma_unmap_sg(struct device *dev, struct scatterlist *sg,
                  int nents, enum dma_data_direction dir);
```

`dma_map_sg()` はDMAマップされたエントリの数を返すが、これは入力のエントリ数より *少ない* ことがある。IOMMUがあれば、カーネルは物理的に不連続なページを、デバイスから見て単一の連続したDMA領域にまとめることができる。

**使うべきとき**: ブロックI/O、大きなネットワーク転送など、データが複数のページにまたがるあらゆる場合。実際のデバイスドライバのほとんどは、スキャッタギャザーを多用している。

## コヒーレントマッピングとストリーミングマッピング {#coherent-vs-streaming-dma-mappings}

これはDMA APIにおける中心的な設計判断である:

| 特性 | コヒーレント | ストリーミング |
|----------|----------|-----------|
| **API** | `dma_alloc_coherent()` | `dma_map_single()`、`dma_map_sg()` |
| **寿命** | 長寿命（ドライバのロードからアンロードまで） | 転送ごと（マップ、DMA、アンマップ） |
| **キャッシュの扱い** | 自動（ハードウェアまたはキャッシュ不可） | 手動（マップ/アンマップ時にフラッシュ/無効化） |
| **CPUアクセスの速度** | 遅い可能性がある（一部のアーキテクチャではキャッシュ不可） | 通常（キャッシュ可能） |
| **ユースケース** | ディスクリプタリング、コマンドキュー | データバッファ、パケット、ディスクブロック |

重要な点は次のとおりである: **コヒーレントマッピングはCPU性能と引き換えに利便性を得ており、ストリーミングマッピングは利便性と引き換えにCPU性能を得ている。** ほとんどのドライバは、小さな制御構造にはコヒーレントマッピングを、データにはストリーミングマッピングを使う。

### sync操作 {#sync-operations}

マップとアンマップの間にCPUがストリーミングバッファに触れる必要がある場合は、sync操作を使わなければならない:

```c
/* デバイスが書き込んだバッファを CPU が読みたい */
dma_sync_single_for_cpu(dev, dma_handle, size, DMA_FROM_DEVICE);
/* ... CPU がデータを読む ... */

/* バッファをデバイスに返す */
dma_sync_single_for_device(dev, dma_handle, size, DMA_FROM_DEVICE);
```

これらはsyncを呼び出した時点でのキャッシュコヒーレンシを保証する。これらがなければ、CPUはキャッシュされた古いデータを読む可能性がある。

## IOMMU {#iommu}

入出力メモリ管理ユニット（IOMMU）はデバイスと物理メモリの間に位置し、デバイスから見える「I/O仮想アドレス」（IOVA）を物理アドレスに変換する。本質的には、デバイスのためのページテーブルである。

```
IOMMU なし:
  Device --[physical addr]--> RAM
  （デバイスは実際の物理アドレスを使わなければならない）

IOMMU あり:
  Device --[IOVA]--> IOMMU --[physical addr]--> RAM
  （デバイスは仮想アドレスを使い、ハードウェアが変換する）
```

### IOMMUが可能にすること {#what-iommu-enables}

1. **スキャッタギャザーの結合**: 物理的に散らばったページを、デバイスから見て連続しているようにマップする。散らばった16ページにまたがる64KBのバッファが、デバイスには1つの連続した64KBの領域に見える。

2. **デバイスの隔離**: バグのある、あるいは悪意のあるデバイスは、そのIOMMUページテーブルに明示的にマップされたメモリ領域にしかアクセスできない。これは、仮想マシンへの安全なデバイスパススルー（VFIO）に不可欠である。

3. **アドレス指定の制限の解消**: IOMMUが高位の物理ページをデバイスの32ビットのアドレスウィンドウにマップするため、32ビットのデバイスでも4GBを超えるメモリにアクセスできる。

4. **DMAリマッピング**: IntelはVT-d、AMDはAMD-Viと呼んでいる。どちらもPCIeデバイス向けのハードウェアIOMMUを提供する。

LinuxのIOMMUドライバは [`drivers/iommu/`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu) 以下にある。DMA APIはIOMMUが存在すれば自動的にそれを使うため、ドライバがそれを知ったり気にしたりする必要はない。

### LWNの記事 {#lwn-coverage}

DMAのアドレス指定、キャッシュコヒーレンシ、デバイスの隔離に関するLWNの記事:

- [The trouble with 64-bit DMA](https://lwn.net/Articles/904210/)（2022年） -- DMAアドレスマスク、IOVA、IOMMUがデバイスアドレスをどうマップするか
- [Noncoherent DMA mappings](https://lwn.net/Articles/855328/)（2021年） -- コヒーレントマッピングとストリーミングマッピング、キャッシュコヒーレンシの管理
- [Restricted DMA](https://lwn.net/Articles/841916/)（2021年） -- IOMMUのないシステムで、SWIOTLBのバウンスバッファを使って信頼できないデバイスを隔離する

## SWIOTLBバウンスバッファ {#swiotlb-bounce-buffers}

IOMMUがなく、デバイスが物理メモリのすべてをアドレス指定できない場合、カーネルはSWIOTLB（Software I/O Translation Lookaside Buffer）— バウンスバッファの仕組み — にフォールバックする。

### 仕組み {#how-it-works}

1. カーネルはブート時に、低位メモリのバウンスバッファのプールを割り当てる（通常は4GB未満のメモリから64MB）
2. デバイスが到達できないバッファに対して `dma_map_single()` が呼ばれると、カーネルは次のことを行う:
   - SWIOTLBプールからバウンスバッファを割り当てる
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

バウンスバッファリングは **メモリコピーが2倍になる** ことを意味し、性能上の大きなペナルティとなる。これが、最近のシステムでIOMMUが強く好まれる理由である。

SWIOTLBプールのサイズは、ブート時に `swiotlb=N`（2KBスロットの数）で設定できる。実装は [`kernel/dma/swiotlb.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma/swiotlb.c) にある。

### コンフィデンシャルコンピューティングとSWIOTLB {#confidential-computing-and-swiotlb}

SWIOTLBは、コンフィデンシャルコンピューティング（AMD SEV、Intel TDX）によって再び重要性を増した。これらの環境では、デバイスのDMAは暗号化されたゲストメモリに直接アクセスできない。ゲストは、すべてのDMA操作に復号されたバウンスバッファを使わなければならない。コミット [0b84e4f8b793](https://git.kernel.org/linus/0b84e4f8b793)（"swiotlb: Add restricted DMA pool initialization"）は、こうしたシナリオに対処するために、制限付きDMAプールのサポートを追加した。

## DMAメモリプール {#dma-memory-pools}

小さな固定サイズのDMAコヒーレントバッファ（USBの転送ディスクリプタやハードウェアのコマンドエントリなど）を頻繁に割り当て・解放するドライバにとって、毎回 `dma_alloc_coherent()` を呼ぶオーバーヘッドは過大である。DMAプールAPIがこれを解決する。

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

プールは大きなコヒーレント領域を事前に割り当て、それを固定サイズのチャンクに切り分ける — slabアロケータと同じ考え方だが、DMAコヒーレントメモリ向けである。`boundary` パラメータは、割り当てが指定された2のべき乗の境界をまたがないようにする。これを必要とするハードウェアもある。

実装は [`mm/dmapool.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/dmapool.c) にある。

## よくある間違い {#common-mistakes}

### アンマップし忘れる {#forgetting-to-unmap}

すべての `dma_map_single()` には対応する `dma_unmap_single()` が必要である。すべての `dma_map_sg()` には対応する `dma_unmap_sg()` が必要である。リークしたマッピングはIOMMUのアドレス空間やSWIOTLBプールを枯渇させ、最終的に割り当ての失敗を引き起こす。

これを捕捉するには `CONFIG_DMA_API_DEBUG` を有効にする。これはすべてのマッピングを追跡し、ドライバのアンロード時に残っているものがあれば警告する:

```bash
# DMA API のデバッグ付きでブート
echo 1 > /sys/kernel/debug/dma-api/enable
```

### 誤ったGFPフラグを使う {#using-wrong-gfp-flags}

ZONE_DMA（最初の16MB）のメモリが本当に必要でない限り、`GFP_DMA` を使ってはならない。32ビットのDMA制限を持つ最近のデバイスのほとんどは、代わりに `GFP_DMA32` を必要とする。しかし理想的には、DMA APIを使ってゾーンの選択を任せるべきである:

```c
/* 誤り: DMA ゾーンから手動で割り当てている */
buf = kmalloc(4096, GFP_KERNEL | GFP_DMA);
dma_handle = virt_to_phys(buf); /* 壊れている: IOMMU を迂回する! */

/* 正しい: DMA API に任せる */
buf = dma_alloc_coherent(dev, 4096, &dma_handle, GFP_KERNEL);
```

DMA APIは、デバイスのDMAマスクを参照して正しいゾーンを自動的に決定する。

### キャッシュコヒーレンシのバグ {#cache-coherency-bugs}

ハードウェアがコヒーレンシを管理しないアーキテクチャでは、適切なsync操作なしにストリーミングDMAバッファにアクセスすると、データ破損が起きる。これは多くの場合、再現が極めて難しい断続的な障害として現れる:

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

### DMAにvirt_to_phys() を使う {#using-virt_to_phys-for-dma}

仮想アドレスを物理アドレスに変換してデバイスに渡してはならない。これはIOMMUを迂回し、バウンスバッファリングをスキップし、デバイスのDMAマスクを無視する。常にDMA APIを使うこと。

### スタックやモジュールのメモリへのDMA {#dma-to-stack-or-module-memory}

スタックのメモリやモジュールのテキストはvmalloc空間に存在することがあり、物理的に連続していることもDMAでアドレス指定可能であることも保証されない。DMAバッファには常に `kmalloc()` か `dma_alloc_coherent()` を使うこと。

## 試してみる {#try-it-yourself}

### DMAゾーンのサイズを確認する {#check-dma-zone-sizes}

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

### IOMMUの状態を調べる {#inspect-iommu-state}

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

### SWIOTLBの使用状況を監視する {#monitor-swiotlb-usage}

```bash
# SWIOTLB プールのサイズと使用量を確認
cat /sys/kernel/debug/swiotlb/io_tlb_nslabs  2>/dev/null
cat /sys/kernel/debug/swiotlb/io_tlb_used    2>/dev/null

# あるいはブート時の dmesg から
dmesg | grep -i swiotlb
# 例: "software IO TLB: mapped [mem 0x00000000b3c00000-0x00000000b7c00000] (64MB)"
```

### DMA APIのデバッグチェックを有効にする {#enable-dma-api-debug-checking}

```bash
# CONFIG_DMA_API_DEBUG=y でカーネルをビルド
# その後、ブート時または実行時に:
echo 1 > /sys/kernel/debug/dma-api/enable

# DMA API のデバッグ統計を表示
cat /sys/kernel/debug/dma-api/num_errors
cat /sys/kernel/debug/dma-api/num_free_entries
cat /sys/kernel/debug/dma-api/min_free_entries
```

### DMAプールの統計を表示する {#view-dma-pool-statistics}

```bash
# デバイスごとの DMA プール情報
cat /sys/kernel/debug/dma_pools 2>/dev/null

# あるいは sysfs から（カーネル設定による）
find /sys/kernel/debug/ -name "*dma*" 2>/dev/null
```

## 主要なソースファイル {#key-source-files}

| ファイル | 説明 |
|------|-------------|
| [`kernel/dma/mapping.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma/mapping.c) | DMAマッピングの中核実装 |
| [`kernel/dma/direct.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma/direct.c) | ダイレクトDMA（IOMMUなし）の実装 |
| [`kernel/dma/swiotlb.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma/swiotlb.c) | SWIOTLBバウンスバッファの実装 |
| [`include/linux/dma-mapping.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/dma-mapping.h) | DMA APIのヘッダ |
| [`include/linux/dmapool.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/dmapool.h) | DMAプールAPIのヘッダ |
| [`mm/dmapool.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/dmapool.c) | DMAプールの実装 |
| [`drivers/iommu/`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu) | IOMMUドライバの実装 |
| [`include/linux/mmzone.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mmzone.h) | ゾーン種別の定義 |

---

## 参考資料 {#references}

### 参考文献 {#further-reading}

- [DMA API Guide](https://docs.kernel.org/core-api/dma-api.html) -- DMA APIに関する正式なカーネルドキュメント
- [DMA API HOWTO](https://docs.kernel.org/core-api/dma-api-howto.html) -- ドライバ開発者向けの実践的なガイド
- [Dynamic DMA mapping Guide](https://docs.kernel.org/core-api/dma-api-howto.html) -- 各マッピング種別をいつ、どう使うか
- [LWN: The trouble with 64-bit DMA](https://lwn.net/Articles/904210/)（2022年） -- DMAアドレスマスク、IOVA、IOMMU
- [LWN: Noncoherent DMA mappings](https://lwn.net/Articles/855328/)（2021年） -- コヒーレントマッピングとストリーミングマッピング、キャッシュコヒーレンシ
- [LWN: Restricted DMA](https://lwn.net/Articles/841916/)（2021年） -- SWIOTLBのバウンスバッファで信頼できないデバイスを隔離する

### 関連項目 {#related}

- [Page Allocator](page-allocator.md) -- DMAメモリの最終的な供給元
- [vmalloc](vmalloc.md) -- 仮想メモリの割り当て（DMAには適さない）
- [NUMA](numa.md) -- NUMAを考慮した割り当てと、DMAゾーンとの相互作用
- [Overview](overview.md) -- メモリ管理の階層の中でDMAの割り当てがどこに位置するか

## 参考文献 {#further-reading_1}

- [Kernel docs: DMA API](https://docs.kernel.org/core-api/dma-api.html) — DMA APIのすべての関数に関する正式なリファレンス
- [Kernel docs: DMA API HOWTO](https://docs.kernel.org/core-api/dma-api-howto.html) — 各マッピング種別をいつ、どう使うかを扱う実践的なガイド
- [`Documentation/core-api/dma-api.rst`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/core-api/dma-api.rst) — DMA APIドキュメントのカーネルソース
- [`kernel/dma/`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma) — DMAサブシステム全体のソースディレクトリ: mapping、direct、swiotlb、pool
- [LWN: The trouble with 64-bit DMA](https://lwn.net/Articles/904210/) — DMAアドレスマスク、IOVA、IOMMUがデバイスアドレスをどうマップするか（2022年）
- [LWN: Noncoherent DMA mappings](https://lwn.net/Articles/855328/) — コヒーレントDMAマッピングとストリーミングDMAマッピング、キャッシュコヒーレンシの管理（2021年）
- [LWN: Restricted DMA](https://lwn.net/Articles/841916/) — IOMMUなしで、SWIOTLBのバウンスバッファを使って信頼できないデバイスを隔離する（2021年）
- [Device Memory Coherency](device-coherency.md) — すべてのDMAマッピングを支配するキャッシュ保守のルール
- [CMA](cma.md) — `dma_alloc_coherent()` がContiguous Memory Allocatorを介して物理的に連続したメモリをどう確保するか
