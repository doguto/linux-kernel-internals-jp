# デバイスメモリのコヒーレンシとキャッシュ管理 {#device-memory-coherency-and-cache-management}

> CPUは積極的にキャッシュするが、DMAデバイスはしない。この溝を正しく埋められるかどうかが、正しく動くドライバと、黙ってデータを破壊するドライバの違いである。

## キャッシュコヒーレンシの問題 {#the-cache-coherency-problem}

最近のCPUは、ロードやストアのたびにRAMへアクセスするわけではない。RAMからキャッシュラインを埋め、そのキャッシュされたコピーに対して操作する。書き込みはRAMへフラッシュされるまで、L1やL2キャッシュに数マイクロ秒とどまることがある。これはCPU上だけで動くソフトウェアからは見えない。2つのコアは同じ物理メモリを共有し、それぞれのキャッシュは互いをスヌープするからである。

DMA対応デバイスはこの前提を崩す。DMAを行うデバイスはRAMを直接読み書きし、CPUキャッシュを完全に迂回する。その結果、2つの障害モードが生じる。

**CPUが書き込み、デバイスが古いデータを読む（DMA送信のバグ）**

```
1. CPU writes packet data to buffer[0..N]      → data in CPU cache, NOT yet in RAM
2. Driver programs NIC: "send buffer at phys X"
3. NIC reads physical address X from RAM        → reads STALE data (pre-write content)
4. Packet sent with wrong content
```

**デバイスが書き込み、CPUが古いデータを読む（DMA受信のバグ）**

```
1. Driver programs NIC: "DMA receive to buffer at phys X"
2. NIC writes received packet to RAM at phys X
3. CPU reads buffer[0..N]                       → reads STALE cache line (old content)
4. Driver processes garbage data
```

これらのバグは間欠的で、負荷に依存し、アーキテクチャ固有である。x86では表面化しない。非コヒーレントなARMでは実際にデータ破壊を引き起こす。

## コヒーレントなシステムと非コヒーレントなシステム {#coherent-vs-non-coherent-systems}

ハードウェアがCPUキャッシュとDMAデバイスの間でコヒーレンシを保証するかどうかは、プラットフォームのメモリインターコネクトによって決まる。

### x86: ハードウェアによるコヒーレンシ {#x86-hardware-enforced-coherency}

x86では、PCIeのDMAトランザクションがCPUキャッシュをスヌープする。デバイスが物理アドレスから読むと、CPUのキャッシュコントローラがトランザクションを横取りし、まだRAMへライトバックされていなくても最新のデータを供給する。デバイスが書き込むと、対応するCPUキャッシュラインは自動的に無効化される。

その結果、x86ではDMAの正しさのためにソフトウェアがCPUキャッシュをフラッシュしたり無効化したりする必要は一切ない。x86では、`dma_map_single()`や`dma_sync_*`関数はキャッシュレベルでは何もしない。

### ARM64: オプションのコヒーレンシ {#arm64-optional-coherency}

ARM64のコヒーレンシはSystem-on-Chipの設計に依存する。

- **コヒーレントインターコネクト（CCI、CMN）**: コヒーレントインターコネクト経由で接続されたデバイスはキャッシュスヌーピングに参加する。サーバー級のARM64プラットフォーム（Ampere、Neoverse、ThunderX）の多くと、最近のモバイルSoCはここに分類される。DMA操作はコヒーレントである。
- **非コヒーレントなデバイス**: 非コヒーレントなインターコネクトや単純なバスブリッジの背後に接続されたデバイス。ソフトウェアは、デバイスの所有権に合わせてCPUキャッシュを明示的にクリーンおよび無効化しなければならない。

デバイスがコヒーレントかどうかはデバイスツリーまたはACPIテーブルに記録され、`dev->dma_coherent`として公開される。カーネルは、[`arch/arm64/mm/dma-mapping.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/mm/dma-mapping.c)の`arch_setup_dma_ops()`によって設定されるこのフラグを確認して、キャッシュメンテナンスを行うかどうかを決める。

```
Coherent platform (x86, coherent ARM64):
  CPU cache ←→ Memory controller ←→ RAM
                      ↑ snoop
                   PCIe device

Non-coherent platform (some ARM64 SoCs):
  CPU cache           RAM
       ↑               ↑
       |       (separate paths, no snooping)
  CPU core         DMA device
```

### 主要なソースファイル {#key-source-files}

| ファイル | 役割 |
|------|---------|
| [`include/linux/dma-mapping.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/dma-mapping.h) | 公開DMA API: `dma_alloc_coherent()`、`dma_map_single()`、`dma_sync_*()` |
| [`include/linux/dma-direction.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/dma-direction.h) | `enum dma_data_direction`の値 |
| [`kernel/dma/direct.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma/direct.h) | IOMMUを使わないDMA経路: `arch_sync_dma_for_device()` / `arch_sync_dma_for_cpu()`を呼ぶ |
| [`arch/arm64/mm/dma-mapping.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/mm/dma-mapping.c) | ARM64の`arch_sync_dma_for_device()` → `dcache_clean_poc()` |
| [`arch/arm64/include/asm/cacheflush.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/include/asm/cacheflush.h) | ARM64のキャッシュ操作: `dcache_clean_poc()`、`dcache_inval_poc()`、`dcache_clean_inval_poc()` |
| [`arch/x86/mm/ioremap.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/ioremap.c) | `ioremap()`、`ioremap_wc()`、`ioremap_uc()`の実装 |
| [`arch/x86/mm/pat/memtype.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/pat/memtype.c) | PATメモリタイプの管理: WB、WC、UC-、WT |

## DMAコヒーレント割り当て {#dma-coherent-allocation}

`dma_alloc_coherent()`は、その存続期間全体を通じてCPUとデバイスの間でコヒーレントであることが保証されたバッファを割り当てる。アクセスする際にキャッシュフラッシュが必要になることは一切ない。

```c
void *dma_alloc_coherent(struct device *dev, size_t size,
                         dma_addr_t *dma_handle, gfp_t gfp);

void dma_free_coherent(struct device *dev, size_t size,
                       void *cpu_addr, dma_addr_t dma_handle);
```

`dma_alloc_coherent()`は2つの値を返す。CPUの仮想アドレス（戻り値）と、`*dma_handle`に書き込まれるデバイスのDMAアドレスである。CPUアドレスは通常のメモリアクセスに使い、DMAアドレスはデバイスのディスクリプタリングやDMAレジスタに設定する。

[`include/linux/dma-mapping.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/dma-mapping.h)より:

```c
static inline void *dma_alloc_coherent(struct device *dev, size_t size,
        dma_addr_t *dma_handle, gfp_t gfp)
{
    return dma_alloc_attrs(dev, size, dma_handle, gfp,
            (gfp & __GFP_NOWARN) ? DMA_ATTR_NO_WARN : 0);
}
```

**アーキテクチャごとのコヒーレンシの実現方法:**

- **x86**: 通常のキャッシュ可能なメモリを割り当てる。ハードウェアのコヒーレンシにより、CPUキャッシュのフラッシュは不要である。
- **非コヒーレントなARM64**: キャッシュされないメモリ（`pgprot_dmacoherent`でマップ）を割り当てる。CPUのアクセスはキャッシュを完全に迂回するため、古いデータは決して生じない。その代償として、CPUの読み書きは遅くなる。

**コヒーレント割り当てを使うべき場合:**

- ディスクリプタリング（NVMeのサブミッション/コンプリーションキュー、ネットワークのTX/RXリング）
- ファームウェアと共有されるコマンド領域やステータス領域
- CPUとデバイスの両方から頻繁にアクセスされる、小さく長寿命な制御構造

**使うべきでない場合:**

CPUが頻繁にアクセスする大きなデータバッファ（パケットペイロード、ディスク読み出しバッファ）には`dma_alloc_coherent()`を避ける。非コヒーレントなシステムでは、大きな転送をキャッシュされないメモリでアクセスするのは、明示的なsync操作を伴うキャッシュ付きメモリよりかなり遅い。

## DMAストリーミングマッピング {#dma-streaming-mappings}

ストリーミングマッピングは、通常のカーネルの手段（`kmalloc`、`__get_free_pages`。`vmalloc`はサポート **されない**）で割り当てられ、一時的にデバイスへ渡す必要があるバッファに使う。コヒーレント割り当てと異なり、これらのバッファは通常キャッシュ可能であり、DMA APIは必要なときだけキャッシュメンテナンスを行う。

### 所有権モデル {#ownership-model}

ストリーミングマッピングは厳格な所有権モデルを課す。**バッファはCPUかデバイスのどちらか一方に属し、同時に両方に属することはない**。

```
CPU owns buffer → dma_map_single() → Device owns buffer
                                           ↓
                                    (device performs DMA)
                                           ↓
Device owns buffer → dma_unmap_single() → CPU owns buffer
```

デバイスがバッファを所有している間（マップからアンマップまで）、CPUはそれを読み書きしてはならない。この規則に違反すると、非コヒーレントなプラットフォームでデータ破壊が起きる。

### dma_map_single() / dma_unmap_single() {#dma_map_single-dma_unmap_single}

```c
/* 仮想的に連続したバッファをマップする */
dma_addr_t dma_map_single(struct device *dev, void *ptr,
                          size_t size, enum dma_data_direction dir);

void dma_unmap_single(struct device *dev, dma_addr_t addr,
                      size_t size, enum dma_data_direction dir);
```

これらは属性ゼロの`dma_map_single_attrs()` / `dma_unmap_single_attrs()`に展開されるマクロである。引数`dir`は、どの方向にキャッシュメンテナンスを適用するかを制御する。

- `DMA_TO_DEVICE` — CPUがデータを書き、デバイスが読む（送信）。マップ時: デバイスがデータを見られるようにCPUキャッシュをRAMへフラッシュする。アンマップ時: 何もしない（デバイスは読んだだけ）。
- `DMA_FROM_DEVICE` — デバイスがデータを書き、CPUが読む（受信）。マップ時: CPUが古いデータを読めないようにCPUキャッシュを無効化する。アンマップ時: デバイスが書いた内容を見えるようにするため、再度CPUキャッシュを無効化する。
- `DMA_BIDIRECTIONAL` — 両方向。保守的であり、クリーンと無効化の両方を行う。

```c
/* TX経路: CPUがバッファを埋めたので、デバイスに渡す */
dma_addr_t dma_addr = dma_map_single(dev, skb->data, skb->len,
                                     DMA_TO_DEVICE);
if (dma_mapping_error(dev, dma_addr))
    goto drop;
/* デバイスのディスクリプタにdma_addrを設定する */
desc->addr = dma_addr;
desc->len  = skb->len;
kick_tx_queue(dev);

/* デバイスが完了した後（TX完了割り込み）: */
dma_unmap_single(dev, dma_addr, skb->len, DMA_TO_DEVICE);
dev_kfree_skb(skb);
```

!!! warning "常に`dma_mapping_error()`を確認すること"
    `dma_map_single()`は失敗しうる（IOMMUの枯渇、デバイスのアドレス範囲のオーバーフロー）。戻り値は必ず`dma_mapping_error(dev, addr)`で確認すること。`DMA_MAPPING_ERROR`をデバイスアドレスとして使うと、ハードウェアフォルトや気付かれないデータ破壊が起きる。

### dma_map_page() {#dma_map_page}

```c
dma_addr_t dma_map_page(struct device *dev, struct page *page,
                        size_t offset, size_t size,
                        enum dma_data_direction dir);

void dma_unmap_page(struct device *dev, dma_addr_t addr,
                    size_t size, enum dma_data_direction dir);
```

バッファが仮想アドレスではなく`struct page`で表される場合に使う。ページベースのskbフラグメントに受信するネットワークドライバでよく使われる。

### dma_map_sg() / dma_unmap_sg() {#dma_map_sg-dma_unmap_sg}

```c
unsigned int dma_map_sg(struct device *dev, struct scatterlist *sg,
                        int nents, enum dma_data_direction dir);

void dma_unmap_sg(struct device *dev, struct scatterlist *sg,
                  int nents, enum dma_data_direction dir);
```

物理的に不連続なページのスキャッタギャザーリストを、1つのDMA操作としてマップする。IOMMUはそれらをI/O仮想アドレスの連続した範囲にマップする。IOMMUがない場合、デバイス自身がスキャッタギャザーをネイティブにサポートしていなければならない。

`dma_map_sg()`はDMAセグメントの数を返す。IOMMUが隣接するエントリを結合した場合、これは`nents`より少なくなることがある。

## dma_sync_*: 部分的な所有権の移譲 {#dma_sync_-partial-ownership-transfers}

アクセスのたびに完全にアンマップして再マップするのは高コストである。`dma_sync_*`関数を使うと、CPUはマップされたバッファをアンマップせずに一時的に所有権を取り戻せる。これは高性能なディスクリプタリングにとって正しいパターンである。

```c
/* include/linux/dma-mapping.hで定義 */
void dma_sync_single_for_cpu(struct device *dev, dma_addr_t addr,
                             size_t size, enum dma_data_direction dir);

void dma_sync_single_for_device(struct device *dev, dma_addr_t addr,
                                size_t size, enum dma_data_direction dir);

void dma_sync_sg_for_cpu(struct device *dev, struct scatterlist *sg,
                         int nelems, enum dma_data_direction dir);

void dma_sync_sg_for_device(struct device *dev, struct scatterlist *sg,
                             int nelems, enum dma_data_direction dir);
```

### syncを用いた所有権プロトコル {#ownership-protocol-with-sync}

```
Initial state: CPU owns buffer

dma_map_single(DMA_FROM_DEVICE)   → Device owns buffer
   [device writes to buffer]
dma_sync_single_for_cpu(...)      → CPU temporarily owns buffer
   [CPU reads completed data]
dma_sync_single_for_device(...)   → Device owns buffer again
   [device writes next chunk]
...
dma_unmap_single(...)             → CPU owns buffer, mapping torn down
```

### RXリングのパターン {#rx-ring-pattern}

受信リングはN個の事前マップ済みバッファを保持する。デバイスは現在のヘッドに書き込み、ドライバはテールから完了したエントリを読む。

```c
/* セットアップ: すべてのRXバッファを事前にマップする */
for (int i = 0; i < RING_SIZE; i++) {
    rx_ring[i].buf = kmalloc(BUF_SIZE, GFP_KERNEL);
    rx_ring[i].dma = dma_map_single(dev, rx_ring[i].buf,
                                    BUF_SIZE, DMA_FROM_DEVICE);
    /* バッファアドレスをハードウェアのディスクリプタに設定する */
    hw_ring[i].addr = rx_ring[i].dma;
}

/* 割り込みハンドラ: デバイスが受信データを書き込んだ */
void rx_interrupt(struct device *dev)
{
    int idx = rx_tail;

    /* 所有権をCPUに戻す */
    dma_sync_single_for_cpu(dev, rx_ring[idx].dma,
                            BUF_SIZE, DMA_FROM_DEVICE);

    /* これで受信データを安全に読める */
    process_packet(rx_ring[idx].buf, hw_ring[idx].len);

    /* 補充: バッファをデバイスに返す */
    dma_sync_single_for_device(dev, rx_ring[idx].dma,
                               BUF_SIZE, DMA_FROM_DEVICE);
    hw_ring[idx].addr = rx_ring[idx].dma;  /* ディスクリプタを再設定 */

    rx_tail = (rx_tail + 1) % RING_SIZE;
}
```

### 方向の値 {#direction-values}

[`include/linux/dma-direction.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/dma-direction.h)より:

```c
enum dma_data_direction {
    DMA_BIDIRECTIONAL = 0,
    DMA_TO_DEVICE     = 1,  /* CPU → デバイス: DMA前にキャッシュをフラッシュ */
    DMA_FROM_DEVICE   = 2,  /* デバイス → CPU: DMA後にキャッシュを無効化 */
    DMA_NONE          = 3,  /* エラー用の番兵。実際の方向ではない */
};
```

## キャッシュ属性とMMIO用のioremap {#cache-attributes-and-ioremap-for-mmio}

デバイスレジスタ（MMIO）には、決してキャッシュされるマッピング経由でアクセスしてはならない。制御レジスタへの書き込みをキャッシュしたCPUは、それを無期限に遅延させるかもしれない。キャッシュされた読み出しは、デバイスに問い合わせる代わりに古い値を返すかもしれない。

`ioremap`ファミリは、物理MMIO領域を適切なキャッシュ属性でカーネルの仮想アドレス空間にマップする。x86では、これらのマッピングは **PAT（Page Attribute Table）** を使い、メモリタイプをページテーブルエントリに直接エンコードする。

### ioremapの種類（x86） {#ioremap-variants-x86}

[`arch/x86/mm/ioremap.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/ioremap.c)より:

```c
/* 標準のキャッシュなしマッピング（UC-minus: 既存のMTRRを尊重） */
void __iomem *ioremap(resource_size_t phys_addr, unsigned long size);

/* 強いキャッシュなし（UC: MTRRを無視し、真にキャッシュなし） */
void __iomem *ioremap_uc(resource_size_t phys_addr, unsigned long size);

/* ライトコンバイン（WC: 書き込みはフラッシュ前にライトバッファで結合される） */
void __iomem *ioremap_wc(resource_size_t phys_addr, unsigned long size);

/* ライトスルー（WT: 書き込みはキャッシュとメモリの両方に行く） */
void __iomem *ioremap_wt(resource_size_t phys_addr, unsigned long size);

void iounmap(volatile void __iomem *addr);
```

**どれを使うべきか:**

| 種類 | PATモード | 用途 |
|---------|----------|----------|
| `ioremap()` | UC-minus | デバイスレジスタのデフォルト。MTRRの範囲を尊重する |
| `ioremap_uc()` | UC | MTRRにかかわらず厳密にキャッシュなしのアクセスが必要な場合 |
| `ioremap_wc()` | WC | フレームバッファ、バルクデータ用のPCIe BAR（GPU VRAM、NVMeドアベル） |
| `ioremap_wt()` | WT | まれ。読み出しはキャッシュしたいが、書き込みは即座に見える必要がある場合 |

ライトコンバイン（`ioremap_wc`）は、フレームバッファなどのバルク書き込み先で特に重要である。CPUは書き込みをライトコンバインバッファにまとめてバーストとしてフラッシュするため、個別のキャッシュなし書き込みに比べてPCIeトランザクションが劇的に減る。

### ioremap_np: ノンポステッド書き込み {#ioremap_np-non-posted-writes}

```c
/* ノンポステッド書き込み: CPUは書き込み完了の応答を待つ */
void __iomem *ioremap_np(phys_addr_t offset, size_t size);
```

[`include/asm-generic/io.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/asm-generic/io.h)より: `ioremap_np()`は、より厳格なノンポステッド書き込みのセマンティクスを要求する。デバイスが書き込みを応答するまでCPUはストールする。これにより、レジスタ書き込みとその後の読み出しの間の順序が保証される。すべてのアーキテクチャが実装しているわけではない。移植性のあるドライバは、`ioremap_np()`が`NULL`を返した場合に`ioremap()`へフォールバックすべきである。

### x86のPATメモリタイプ {#pat-memory-types-on-x86}

x86のPATは、MMIOに関係する4つのメモリタイプをサポートする（[`arch/x86/mm/pat/memtype.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/pat/memtype.c)より）。

```
_PAGE_CACHE_MODE_WB       Write-back (normal RAM, cached)
_PAGE_CACHE_MODE_WC       Write-combining (framebuffers, PCIe bulk)
_PAGE_CACHE_MODE_UC_MINUS UC-minus (device registers, honors MTRRs)
_PAGE_CACHE_MODE_WT       Write-through
```

PATはメモリタイプをページテーブルエントリのビット（PWT、PCD、PATビット）にエンコードする。カーネルはこれを`ioremap_change_attr()`で管理する。これは`_set_memory_uc()`、`_set_memory_wc()`などを呼び出してPTEを更新し、TLBをフラッシュする。

## DMAのためのARM64キャッシュ操作 {#arm64-cache-operations-for-dma}

非コヒーレントなARM64プラットフォームでは、DMAマッピング層が明示的なキャッシュメンテナンスを行う。非コヒーレントなデバイスに対する`dma_map_single()`の呼び出しチェーンは、[`kernel/dma/direct.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma/direct.h)と[`arch/arm64/mm/dma-mapping.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/mm/dma-mapping.c)より次のとおりである。

```
dma_map_single()
  └─ dma_map_single_attrs()
       └─ dma_map_page_attrs()
            └─ dma_direct_map_phys()       [kernel/dma/direct.h]
                 └─ if (!dev_is_dma_coherent(dev)):
                      arch_sync_dma_for_device(paddr, size, dir)
                        └─ dcache_clean_poc(start, start + size)
```

受信時のCPU側のsyncは次のとおり。

```
dma_sync_single_for_cpu()
  └─ __dma_sync_single_for_cpu()
       └─ dma_direct_sync_single_for_cpu()
            └─ if (!dev_is_dma_coherent(dev)):
                 arch_sync_dma_for_cpu(paddr, size, dir)
                   └─ if dir != DMA_TO_DEVICE:
                        dcache_inval_poc(start, start + size)
```

ARM64のキャッシュ関数は、範囲`[start, end)`に対して **Point of Coherency（PoC）** まで操作する。PoCとは、DMA対応デバイスを含むすべての観測者が共有するキャッシュ階層のレベルである。

[`arch/arm64/include/asm/cacheflush.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/include/asm/cacheflush.h)より:

```c
/* クリーン（ダーティなラインをライトバックし、有効なまま保持） */
extern void dcache_clean_poc(unsigned long start, unsigned long end);

/* 無効化（キャッシュラインを破棄し、RAMからの再取得を強制） */
extern void dcache_inval_poc(unsigned long start, unsigned long end);

/* クリーン＋無効化（ライトバックしてから破棄） */
extern void dcache_clean_inval_poc(unsigned long start, unsigned long end);
```

**DMAの文脈で各操作が行うこと:**

- `dcache_clean_poc`: `DMA_TO_DEVICE`で使われる。デバイスが最新のデータを読めるように、CPUキャッシュをRAMへフラッシュする。
- `dcache_inval_poc`: `DMA_FROM_DEVICE`で使われる。デバイスが書き込んだ後にCPUがRAMから再取得するように、CPUのキャッシュラインを破棄する。
- `dcache_clean_inval_poc`: `DMA_BIDIRECTIONAL`で使われる。両方を組み合わせたもので、データの流れの方向がわからない場合に使う。

コヒーレントなARM64プラットフォーム（`dev->dma_coherent`がtrue）では、`arch_sync_dma_for_device()`や`arch_sync_dma_for_cpu()`は呼ばれない。ハードウェアが透過的に処理し、x86の挙動と一致する。

## IOMMUとコヒーレンシ {#the-iommu-and-coherency}

IOMMUは、デバイスのDMAアドレス（I/O仮想アドレス）を物理アドレスへ変換する。これによりアドレス空間の隔離が可能になる。デバイスは、カーネルがそのI/Oページテーブルに明示的にマップしたメモリにしかアクセスできない。

IOMMUはCPUキャッシュのコヒーレンシを解決 **しない**。IOMMUはデバイスとRAMの間のメモリバス上で動作するのであって、CPUとRAMの間ではない。非コヒーレントなシステムでは、IOMMUがあっても、ソフトウェアは依然として`dma_sync_*`や`dma_map_*` / `dma_unmap_*`を通じてキャッシュメンテナンスを行わなければならない。

唯一の例外として、ARM SoC上の一部のIOMMUはコヒーレントインターコネクト側に配置されており、スヌーピングに参加できる。その場合、デバイスのコヒーレンシは、インターコネクト階層のどこにデバイスが配置されているかで決まり、それは`dev->dma_coherent`に反映される。

IOMMUのアドレス変換とメモリ保護については、[DMAメモリ割り当て](dma.md)を参照。

## よくあるバグ {#common-bugs}

!!! warning "マップ済みバッファへのCPUからのアクセス"
    `dma_map_single()`を呼ぶと所有権がデバイスに移る。`dma_unmap_single()`（または`dma_sync_single_for_cpu()`）の前にCPUがそのバッファを読み書きするのは未定義動作である。コヒーレントなx86ではこのバグは表面化しない。非コヒーレントなARM64では、CPUが古いキャッシュを読むか、デバイスが古いRAMを読む。

    ```c
    /* 誤り: マップ後にCPUがバッファにアクセスしている */
    dma_addr_t dma = dma_map_single(dev, buf, len, DMA_FROM_DEVICE);
    buf[0] = 0;                    /* 未定義: bufはデバイスが所有 */
    submit_to_device(dev, dma);

    /* 正しい: CPUの書き込みが終わってからマップする */
    buf[0] = 0;
    dma_addr_t dma = dma_map_single(dev, buf, len, DMA_TO_DEVICE);
    submit_to_device(dev, dma);
    ```

!!! warning "RXバッファを読む前のDMA_FROM_DEVICEのsync忘れ"
    デバイスがRXバッファを埋めた後、それを読む前に`DMA_FROM_DEVICE`で`dma_sync_single_for_cpu()`を呼ぶ必要がある。これがないと、非コヒーレントなプラットフォームでは、CPUはデバイスが書いた内容ではなく自分の古いキャッシュを読む。

    ```c
    /* 非コヒーレントなARM64では誤り: */
    /* デバイスがrx_bufにパケットデータを書き込んだ */
    process_packet(rx_buf, len);   /* 古いキャッシュを読む */

    /* 正しい: */
    dma_sync_single_for_cpu(dev, rx_dma, len, DMA_FROM_DEVICE);
    process_packet(rx_buf, len);   /* RAMから新しいデータを読む */
    ```

!!! warning "大きなストリーミングバッファへのdma_alloc_coherent()の使用"
    非コヒーレントなARM64でのコヒーレント割り当ては、キャッシュされないメモリに裏付けられている。キャッシュなしのマッピングを通じて大量のデータを読み書きするのは遅い。CPUはライトバッファを使ったり、複数のキャッシュラインを一度にロードしたりできないからである。バルクデータ（パケットペイロード、DMAスキャッタリスト）には、代わりに明示的なsyncを伴う`dma_map_single()` / `dma_map_sg()`を使うこと。

!!! warning "キャッシュされるマッピング経由でのMMIOアクセス"
    デバイスレジスタを`ioremap()`ではなく`ioremap_cache()`（ライトバック）でマップすると、読み出しがキャッシュされた値を返し、書き込みが無期限に遅延しうる。MMIO領域には常に`ioremap()`または`ioremap_wc()`を使うこと。`phys_to_virt()`で得たポインタ経由でデバイスレジスタにアクセスしてはならない。そのマッピングはライトバックでキャッシュされる。

!!! warning "dma_mapping_error()の無視"
    `dma_map_single()`と`dma_map_sg()`は失敗しうる。失敗時の戻り値は`DMA_MAPPING_ERROR`（64ビットでは~0ULL）である。この値をデバイスのDMAレジスタに設定すると、デバイスフォルトが起きるか、誤ったアドレスの物理メモリにアクセスすることになる。

    ```c
    dma_addr_t addr = dma_map_single(dev, buf, len, DMA_TO_DEVICE);
    if (dma_mapping_error(dev, addr)) {
        /* エラー処理: パケットを破棄する、エラーを返す、など */
        return -ENOMEM;
    }
    ```

## 参考文献 {#further-reading}

- [Kernel docs: DMA API](https://docs.kernel.org/core-api/dma-api.html) — `dma_alloc_coherent()`、`dma_map_*()`、`dma_sync_*()`のセマンティクスに関する標準的なリファレンス
- [`Documentation/core-api/dma-api.rst`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/core-api/dma-api.rst) — 所有権の規則や方向の値を含む、DMA APIドキュメントのカーネルソース
- [`arch/arm64/mm/dma-mapping.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/mm/dma-mapping.c) — ARM64の`arch_sync_dma_for_device()`と`arch_sync_dma_for_cpu()`の実装
- [`arch/x86/mm/pat/memtype.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/pat/memtype.c) — `ioremap_wc()`および`ioremap_uc()`マッピングのためのx86 PATメモリタイプ管理
- [LWN: DMA and cache coherency](https://lwn.net/Articles/501591/) — 非コヒーレントなプラットフォームでなぜ明示的なキャッシュ管理が必要かの詳解（2012年）
- [DMAメモリ割り当て](dma.md) — IOMMUとの統合、SWIOTLBのバウンスバッファ、DMAゾーンのレイアウト
- [PCI BARマッピング](pci-bar-mapping.md) — デバイスレジスタアクセスのためのMMIOキャッシュ属性（`ioremap`、`ioremap_wc`）
