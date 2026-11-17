# PCI BARのメモリマッピング {#pci-bar-memory-mapping}

> カーネルがPCIデバイスのメモリ領域をどのように発見・確保し、カーネル仮想アドレス空間へマップするか

## PCI BARとは何か {#what-are-pci-bars}

すべてのPCIおよびPCIeデバイスは、そのコンフィギュレーション空間に最大6つの**ベースアドレスレジスタ**（Base Address Register、BAR）を公開しており、BAR0からBAR5まで番号が振られている。これらのレジスタは、デバイスのメモリマップドI/O（MMIO）領域が物理アドレス空間のどこにあるかをオペレーティングシステムに伝える。

BARは標準PCIコンフィギュレーションヘッダのオフセット`0x10`から`0x24`にあり、[`include/uapi/linux/pci_regs.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/pci_regs.h)で定義されている。

```c
#define PCI_BASE_ADDRESS_0  0x10  /* 32 bits */
#define PCI_BASE_ADDRESS_1  0x14  /* 32 bits */
#define PCI_BASE_ADDRESS_2  0x18  /* 32 bits */
#define PCI_BASE_ADDRESS_3  0x1c  /* 32 bits */
#define PCI_BASE_ADDRESS_4  0x20  /* 32 bits */
#define PCI_BASE_ADDRESS_5  0x24  /* 32 bits */
```

標準BARの最大数は、[`include/uapi/linux/pci_regs.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/pci_regs.h)で`PCI_STD_NUM_BARS = 6`として定義されている。

### BARのサイズはどのように決まるか {#how-bar-size-is-determined}

ファームウェアやOSは、ハードウェアのプロトコルを使ってBAR領域のサイズを発見する。BARレジスタにすべて1（`0xFFFFFFFF`）を書き込み、値を読み戻し、タイプビットをマスクして反転する。結果はサイズから1を引いたものになる。これは**BARサイジング**と呼ばれ、[`drivers/pci/probe.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pci/probe.c)の`pci_read_bases()`によって実行される。この関数はバス列挙の際に`pci_setup_device()`から呼ばれる。

```
Write: 0xFFFFFFFF → BAR register
Read back: 0xFFFFF000  (device decodes 12 address bits → 4 KB region)
Mask type bits, invert: size = 4096 bytes
```

デバイスは自分が応答するビットだけをデコードし、残りはゼロとして読み戻される。下位ビットはタイプ情報を保持しており、反転の前にマスクされる。

## BARの種類 {#bar-types}

各BARレジスタの下位ビットは、その領域の種類をエンコードしている。

```c
#define PCI_BASE_ADDRESS_SPACE         0x01  /* 0 = メモリ, 1 = I/O */
#define PCI_BASE_ADDRESS_SPACE_IO      0x01
#define PCI_BASE_ADDRESS_SPACE_MEMORY  0x00
#define PCI_BASE_ADDRESS_MEM_TYPE_MASK 0x06
#define PCI_BASE_ADDRESS_MEM_TYPE_32   0x00  /* 32ビットアドレス */
#define PCI_BASE_ADDRESS_MEM_TYPE_1M   0x02  /* 1M未満（廃止） */
#define PCI_BASE_ADDRESS_MEM_TYPE_64   0x04  /* 64ビットアドレス */
#define PCI_BASE_ADDRESS_MEM_PREFETCH  0x08  /* プリフェッチ可能か */
```

### メモリBAR（`PCI_BASE_ADDRESS_SPACE_MEMORY`） {#memory-bars-pcibaseaddressspacememory}

一般的なケースである。BARは、デバイスのレジスタやデバイスローカルメモリに直接対応する物理アドレス空間のウィンドウを記述する。このウィンドウへのCPUアクセスはPCIeバスに出て、デバイスに届く。カーネルはこれらの領域を`ioremap()`またはその変種の1つを使ってマップする。

### I/O BAR（`PCI_BASE_ADDRESS_SPACE_IO`） {#io-bars-pcibaseaddressspaceio}

x86固有のレガシーな種類である。デバイスはx86のI/Oポートアドレス空間の範囲を占有し、通常のメモリのロード／ストアではなく`inb()`/`outb()`命令でアクセスされる。現代のデバイスがI/O BARを使うことはまれである。PCIeのファームウェアはしばしばI/O BARを割り当てるが、デバイスは実際には必要としていないことがある。`pci_iomap()`は両方の種類を透過的に扱う。

### プリフェッチ可能なBAR（`PCI_BASE_ADDRESS_MEM_PREFETCH`） {#prefetchable-bars-pcibaseaddressmemprefetch}

読み出しに副作用がなく、CPUやPCIeファブリックが安全にデータをプリフェッチし、書き込みを結合できることが保証されたメモリBARである。これは、シーケンシャルな書き込みがライトコンバイニングの恩恵を受けるGPUのフレームバッファのような領域にのみ設定される。プリフェッチ不可のBARを投機的に読み出してはならない -- 副作用（読み出し時にステータスレジスタがクリアされるなど）が誤って引き起こされてしまう。

### 64ビットBAR（`PCI_BASE_ADDRESS_MEM_TYPE_64`） {#64-bit-bars-pcibaseaddressmemtype_64}

64ビットのメモリBARは、64ビットのベースアドレス全体を格納するために**連続する2つのBARスロット**を使う。下位32ビットはBAR `n`に、上位32ビットはBAR `n+1`に格納される。つまり、デバイスは最大3つの64ビットBARを持てる（6つのスロットすべてをペアで使う）。また`pci_resource_start(dev, n)`でリソースにアクセスすると、完全な64ビットの物理アドレスが正しく得られる -- カーネルは`pci_read_bases()`の中で2つのレジスタの組み立てを透過的に処理する。

```
BAR Layout for a device with one 64-bit BAR at BAR0:

  BAR0  [0x10]: lower 32 bits of address + type bits
  BAR1  [0x14]: upper 32 bits of address
  BAR2  [0x18]: next independent BAR (e.g., another memory region)
```

## LinuxにおけるBARの列挙 {#linux-bar-enumeration}

PCIバスの列挙の間、カーネルは見つかったすべてのデバイスについて`pci_setup_device()` → `pci_read_bases()`を呼ぶ。この関数はすべてのBARを読み取ってサイジングし、結果を`dev->resource[]`に格納する。これは`struct resource`の配列で、（標準BAR用に）0から5までのインデックスを持ち、エントリ6は拡張ROM用に予約されている。

列挙の後は、[`include/linux/pci.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/pci.h)の3つのマクロが、ドライバに必要なものをすべて提供する。

```c
pci_resource_start(dev, bar)  /* BAR の物理ベースアドレス */
pci_resource_len(dev, bar)    /* BAR 領域のバイト単位のサイズ */
pci_resource_flags(dev, bar)  /* 種類と属性のフラグ */
```

フラグは[`include/linux/ioport.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/ioport.h)の`IORESOURCE_*`定数を使う。

| フラグ | 値 | 意味 |
|------|-------|---------|
| `IORESOURCE_IO` | `0x00000100` | I/OポートBAR |
| `IORESOURCE_MEM` | `0x00000200` | メモリBAR |
| `IORESOURCE_PREFETCH` | `0x00002000` | プリフェッチ可能なメモリ |
| `IORESOURCE_MEM_64` | `0x00100000` | 64ビットアドレスのBAR |

## BARの要求とマッピング {#requesting-and-mapping-a-bar}

ドライバは3つのステップを順に踏まなければならない。デバイスを有効化し、カーネルのリソースツリーからBAR領域を確保し、BARをカーネル仮想アドレス空間へマップする。3つの関数はすべて[`drivers/pci/pci.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pci/pci.c)と[`drivers/pci/iomap.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pci/iomap.c)にある。

### 標準的なドライバのパターン {#the-standard-driver-pattern}

```c
static int my_driver_probe(struct pci_dev *pdev,
                           const struct pci_device_id *id)
{
    void __iomem *base;
    int err;

    /* 1. デバイスを有効化する（電源投入、バスマスタリングの有効化） */
    err = pci_enable_device(pdev);
    if (err)
        return err;

    /* 2. BAR 0 を確保する — 他のドライバが触れないようにする。
     *    pci_request_regions() はすべての BAR を一度に確保する。 */
    err = pci_request_region(pdev, 0, "my_driver");
    if (err)
        goto err_disable;

    /* 3. BAR 0 をカーネル仮想アドレス空間へマップする。
     *    BAR 全体をマップするには maxlen に 0 を渡す。 */
    base = pci_iomap(pdev, 0, 0);
    if (!base) {
        err = -ENOMEM;
        goto err_release;
    }

    /* 4. ioread*／iowrite* を使ってデバイスにアクセスする — ポインタを
     *    決して直接デリファレンスしないこと。 */
    u32 version = ioread32(base + REG_VERSION);
    iowrite32(CTRL_ENABLE, base + REG_CONTROL);

    pci_set_drvdata(pdev, base);
    return 0;

err_release:
    pci_release_region(pdev, 0);
err_disable:
    pci_disable_device(pdev);
    return err;
}

static void my_driver_remove(struct pci_dev *pdev)
{
    void __iomem *base = pci_get_drvdata(pdev);

    pci_iounmap(pdev, base);
    pci_release_region(pdev, 0);
    pci_disable_device(pdev);
}
```

### 関数リファレンス {#function-reference}

**`pci_request_region(pdev, bar, name)`** / **`pci_request_regions(pdev, name)`**

カーネルの`iomem`または`ioport`リソースツリーにエントリを挿入することで、1つのBAR（またはすべてのBAR）を確保する。別のドライバがすでにその領域を所有していれば`-EBUSY`を返す。これは、2つのドライバが同じハードウェアレジスタを同時にマップするのを防ぐソフトウェア上のロックである。[`drivers/pci/pci.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pci/pci.c)で定義されている。

**`pci_iomap(pdev, bar, maxlen)`**

BARをカーネル仮想アドレス空間にマップし、`void __iomem *`のクッキーを返す。内部では、メモリBARには`ioremap()`を、I/O BARには`__pci_ioport_map()`を呼ぶ。`maxlen`が0なら、BAR全体がマップされる。[`drivers/pci/iomap.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pci/iomap.c)で定義されている。

**`ioread32(addr)` / `iowrite32(val, addr)`**

MMIOにアクセスする正しい方法である。これらの関数は必要なメモリバリアを処理し、下位のマッピングが本物のMMIOかI/Oポート範囲かにかかわらず正しく動作する。`__iomem`ポインタを直接デリファレンス**しない**こと -- コンパイラは許してしまうが、バリアを迂回するため、一部のアーキテクチャでは壊れる。

**`pci_iounmap(pdev, addr)` / `pci_release_regions(pdev)`**

マッピングと確保の後始末を行うペアである。ドライバの`remove()`で必ず呼ぶこと。

### マネージド版: `pcim_iomap()` {#managed-variant-pcim_iomap}

ドライバのデタッチ時に自動的な後始末を望むドライバ向けに、devresで管理される変種`pcim_iomap()`が用意されている。これには`pci_enable_device()`の代わりに`pcim_enable_device()`が必要である。マッピングはドライバがアンバインドされると自動的に解放される。[`drivers/pci/devres.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pci/devres.c)で定義されている。

```c
err = pcim_enable_device(pdev);
if (err)
    return err;

base = pcim_iomap(pdev, 0, 0);
if (!base)
    return -ENOMEM;
/* remove() での明示的な後始末は不要 */
```

## BARマッピングのメモリタイプ {#memory-types-for-bar-mappings}

CPUがMMIOにどうアクセスするかは、仮想マッピングの作成時に適用される**ページ属性**に依存する。この選択は性能に大きく影響する。x86では、これは**Page Attribute Table（PAT）**によって制御され、[`arch/x86/mm/pat/memtype.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/pat/memtype.c)に実装されている。

### キャッシュなし（UC） — プリフェッチ不可のBAR {#uncached-uc-non-prefetchable-bars}

`ioremap()`（`pci_iomap()`から内部的に呼ばれる）は、領域を**UC-minus**（キャッシュ不可、ライトポステッド）としてマップする。すべての読み書きはデバイスに直接行き、どのCPUキャッシュにもデータは保持されない。これは、各読み出しが副作用を持ちうるうえ、書き込みが即座にデバイスから見える必要がある制御レジスタには必須である。

```c
/* ioremap() がデフォルト -- UC-minus キャッシング */
void __iomem *ioremap(resource_size_t phys_addr, unsigned long size);
```

### ライトコンバイニング（WC） — プリフェッチ可能なBAR {#write-combining-wc-prefetchable-bars}

`ioremap_wc()`は、**ライトコンバイニング**を有効にして領域をマップする。CPUは複数の書き込みをバッファし、1回のバーストでデバイスに送ることができるため、シーケンシャルな書き込みのスループットが劇的に向上する。読み出しは依然としてデバイスに行くが、読み出しのプリフェッチはない。これはフレームバッファ、PCIeのピアツーピアメモリ、そしてあらゆる大規模なストリーミング書き込みのワークロードにとって正しい選択である。

```c
/* ioremap_wc() -- プリフェッチ可能な BAR 向けのライトコンバイニング */
void __iomem *ioremap_wc(resource_size_t phys_addr, unsigned long size);
```

PCIサブシステムは、ライトコンバイニングの場合のためのラッパーを提供しており、[`drivers/pci/iomap.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pci/iomap.c)で定義されている。

```c
/* BAR をライトコンバイニングでマップする（プリフェッチ可能な BAR のみ） */
void __iomem *pci_iomap_wc(struct pci_dev *dev, int bar,
                            unsigned long maxlen);
```

`pci_iomap_wc()`は、I/O BARに対しては`NULL`を返し（ポートI/Oにライトコンバイニングは意味をなさない）、メモリBARに対しては`ioremap_wc()`を呼ぶ。

### 適切なマッピングの選択 {#choosing-the-right-mapping}

```c
unsigned long flags = pci_resource_flags(pdev, bar);

if (flags & IORESOURCE_PREFETCH) {
    /* ライトコンバイニングを安全に使える */
    base = pci_iomap_wc(pdev, bar, 0);
} else {
    /* プリフェッチ不可: キャッシュなしを使う */
    base = pci_iomap(pdev, bar, 0);
}
```

!!! warning "性能への影響"
    256 MBのフレームバッファをWCではなくUCとしてマップすると、シーケンシャルな書き込みのスループットが1桁低下しうる。GPUドライバは、同じデバイス上であっても、VRAMのBARには`pci_iomap_wc()`を、レジスタのBARには`pci_iomap()`を慎重に使い分けている。

## IOMMUとBARアクセス {#iommu-and-bar-access}

IOMMUは、デバイスが発行するDMAアドレス（デバイス → RAMの方向）を変換する。BARアクセスは**逆方向**（CPU → デバイス）に進み、IOMMUを完全に迂回する。CPUはPCIeバス上で物理メモリトランザクションを発行し、デバイスが直接応答する。

```
DMA (device → RAM):
  Device --[DMA addr]--> IOMMU --[phys addr]--> RAM

MMIO (CPU → device):
  CPU --[phys addr]--> PCIe --[reaches device, not RAM]
```

IOMMUはMMIOの読み書きには関与しない。ただし、BARの割り当て時に、ファームウェア（および任意でカーネル）は、BARの物理アドレスが搭載RAMと重ならないことを保証しなければならない。x86では、これは`E820`のメモリマップによって強制され、PCIサブシステムはBARの割り当てを受け入れる前に`iomem_resource`を確認する。

## 64ビットBARとアドレス空間の衝突 {#64-bit-bars-and-address-space-conflicts}

4 GBを超えるRAMを持つシステムでは、32ビットBAR（4 GB未満のアドレスしか記述できない）が搭載メモリと衝突しうる。現代のほとんどのx86システムでは、ファームウェアが4 GB未満の領域（**PCIホール**や**TOLUD** -- Top Of Low Usable DRAM -- と呼ばれることがある）を予約し、そこにBARを衝突なく配置することでこれに対処する。しかし、これは実質的にアドレス可能な低位メモリを縮小させる。

ドライバのコードからBARが64ビットアドレッシングを使っているかを確認するには、次のようにする。

```c
if (pci_resource_flags(pdev, bar) & IORESOURCE_MEM_64) {
    /* BAR は64ビットアドレスを使う。bar+1 は上位ビットに消費される */
    pr_info("64-bit BAR at %pa, size %pa\n",
            &pci_resource_start(pdev, bar),
            &pci_resource_len(pdev, bar));
}
```

現代のNVMeドライブ、CXLメモリデバイス、ハイエンドGPUは、4 GBを超える64ビットBAR（例: 16 GBのVRAM BAR）を4 GB境界より上に配置して日常的に使っている。ファームウェアがBARを未割り当てのまま残したり、最適でない形で割り当てたりした場合、カーネルは[`drivers/pci/setup-res.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pci/setup-res.c)の`pci_reassign_resource()`を使ってこれらを配置する。

!!! note "ファームウェアとカーネルによるBARの割り当て"
    ほとんどのx86システムでは、カーネルが起動する前にファームウェアがすべてのBARを割り当てる。カーネルはその割り当てを読み取り、信頼する。組み込みシステムや、カーネルコマンドラインに`pci=realloc`が渡された場合には、カーネル自身がBARを再割り当てすることがある。

## BARマッピングのデバッグ {#debugging-bar-mappings}

### `lspci -v` {#lspci-v}

デバイスに割り当てられたすべてのBARを、物理アドレス、サイズ、種類を含めて表示する。

```
$ lspci -v -s 03:00.0
03:00.0 VGA compatible controller: NVIDIA Corporation ...
        Memory at f6000000 (32-bit, non-prefetchable) [size=16M]
        Memory at e0000000 (64-bit, prefetchable) [size=256M]
        Memory at f0000000 (64-bit, prefetchable) [size=32M]
        I/O ports at e000 [size=128]
```

デバイスの説明の後の行がBARである。`prefetchable`は`IORESOURCE_PREFETCH`が設定されていることを示す。

### `/proc/iomem` {#prociomem}

BARが、RAMや他のリソースと並んで物理アドレス空間をどのように占有しているかを示す。

```
$ cat /proc/iomem
00000000-00000fff : Reserved
...
e0000000-efffffff : PCI Bus 0000:03
  e0000000-efffffff : 0000:03:00.0
    e0000000-efffffff : nvidia
f6000000-f6ffffff : 0000:03:00.0
  f6000000-f6ffffff : nvidia
```

エントリはリソースの階層を示すためにインデントされている。バスウィンドウ → デバイスのBAR → ドライバによる確保、である。

### `/sys/bus/pci/devices/BBDF/resource` {#sysbuspcidevicesbbdfresource}

各PCIデバイスには`resource`ファイルがあり、6つのBARすべてと拡張ROMを、1行に1つずつ、開始・終了・フラグの3つの16進フィールドとして示す。

```
$ cat /sys/bus/pci/devices/0000:03:00.0/resource
0x00000000f6000000 0x00000000f6ffffff 0x0000000000040200
0x00000000e0000000 0x00000000efffffff 0x000000000014220c
0x00000000f0000000 0x00000000f1ffffff 0x000000000014220c
0x000000000000e000 0x000000000000e07f 0x0000000000040101
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
```

3列目は`IORESOURCE_*`フラグである。`0x200`は`IORESOURCE_MEM`、`0x2000`は`IORESOURCE_PREFETCH`、`0x100000`は`IORESOURCE_MEM_64`である。

### `/sys/bus/pci/devices/BBDF/resource0`から`resource5` {#sysbuspcidevicesbbdfresource0-through-resource5}

これらのファイルは（BARが排他としてマークされていなければ）ユーザー空間から`mmap()`できる。ユーザー空間のツールは、カーネルドライバなしにBARを直接マップできる。

```c
int fd = open("/sys/bus/pci/devices/0000:03:00.0/resource0", O_RDWR);
size_t len = /* resource ファイルまたは lspci から */ 16 * 1024 * 1024;
void *mmio = mmap(NULL, len, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
uint32_t val = *(volatile uint32_t *)mmio;  /* ユーザー空間からの直接アクセス */
```

これは、`nvtop`やGPU計算用デバッガのようなツールが、カーネルドライバなしにデバイスメモリにアクセスする方法である。なお、`pci_request_regions_exclusive()`は`IORESOURCE_EXCLUSIVE`を設定してsysfsのmmap経路をブロックすることで、これを防ぐ。

### 起動時のBAR割り当ての確認 {#checking-bar-assignments-at-boot}

```bash
# バス列挙中の PCI 関連のすべてのメッセージ
dmesg | grep -i pci | grep -i BAR

# sysfs から特定のデバイスのリソースを確認する
cat /sys/bus/pci/devices/0000:03:00.0/resource

# iomem マップ全体を表示する
cat /proc/iomem | grep -A2 "PCI Bus"
```

## 主要なソースファイル {#key-source-files}

| ファイル | 説明 |
|------|-------------|
| [`drivers/pci/iomap.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pci/iomap.c) | `pci_iomap()`、`pci_iomap_wc()`、`pci_iomap_range()`、`pci_iounmap()` |
| [`drivers/pci/pci.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pci/pci.c) | `pci_request_region()`、`pci_request_regions()`、`pci_release_regions()` |
| [`drivers/pci/probe.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pci/probe.c) | `pci_setup_device()`、`pci_read_bases()` — BARのサイジングと列挙 |
| [`drivers/pci/devres.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pci/devres.c) | `pcim_iomap()`、`pcim_enable_device()` — マネージド（devres）版 |
| [`drivers/pci/setup-res.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pci/setup-res.c) | `pci_reassign_resource()` — BARの再配置 |
| [`include/linux/pci.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/pci.h) | `pci_resource_start()`、`pci_resource_len()`、`pci_resource_flags()` |
| [`include/uapi/linux/pci_regs.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/pci_regs.h) | `PCI_BASE_ADDRESS_*`定数、`PCI_STD_NUM_BARS` |
| [`include/linux/ioport.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/ioport.h) | `IORESOURCE_MEM`、`IORESOURCE_PREFETCH`、`IORESOURCE_MEM_64` |
| [`arch/x86/mm/ioremap.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/ioremap.c) | x86での`ioremap()`、`ioremap_wc()`、`ioremap_uc()` |
| [`arch/x86/mm/pat/memtype.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/pat/memtype.c) | PAT — キャッシュモードを制御するページ属性テーブル |

---

## 参考資料 {#references}

### 参考文献 {#further-reading}

- [Kernel documentation: Device I/O](https://docs.kernel.org/driver-api/device-io.html) — MMIOアクセス関数に関する信頼できるガイド
- [PCI Local Bus Specification, rev 3.0](https://pcisig.com/) — BARのレイアウト、サイジングのプロトコル、タイプビットを定義している
- [LWN: PCI resource management](https://lwn.net/Articles/51834/) — LinuxがPCIリソースをどのように管理するかの初期の概観
- [LWN: Write-combining mappings](https://lwn.net/Articles/252029/) — x86でのPATとライトコンバイニングの性能

### 関連ページ {#related-pages}

- [DMAメモリ割り当て](dma.md) — デバイスがRAMとの間でどのようにデータを転送するか（MMIOとは別物）
- [vmalloc](vmalloc.md) — `ioremap()`が切り出すカーネル仮想アドレス空間
- [ページテーブル](page-tables.md) — `ioremap()`のマッピングがカーネルのページテーブルにどう現れるか
- [CXLメモリ階層化](cxl-memory-tiering.md) — CXLデバイスはメモリ拡張のために大きな64ビットBARを公開する

## 参考文献 {#further-reading_1}

- [Kernel docs: Device I/O](https://docs.kernel.org/driver-api/device-io.html) — `ioread*`/`iowrite*`、`ioremap`の変種、MMIOアクセスの規則に関する信頼できるガイド
- [`Documentation/PCI/`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/PCI) — BARの割り当て、エラー処理、ホストブリッジの列挙を扱うカーネルのPCIドキュメントのディレクトリ
- [`Documentation/PCI/pci.rst`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/PCI/pci.rst) — 有効化／要求／マップのライフサイクルを含む、PCIドライバの書き方のステップバイステップガイド
- [`drivers/pci/iomap.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pci/iomap.c) — `pci_iomap()`と`pci_iomap_wc()`のソース
- [`drivers/pci/probe.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pci/probe.c) — `pci_read_bases()`によるBARサイジングのプロトコルの実装
- [LWN: PCI resource management](https://lwn.net/Articles/51834/) — LinuxがPCIリソースをどのように確保・追跡するかの初期の概観（2003年）
- [LWN: Write-combining and the PAT](https://lwn.net/Articles/252029/) — x86でのPATメモリタイプとライトコンバイニングの性能（2008年）
- [../drivers/pci-driver.md](../drivers/pci-driver.md) — PCIドライバモデル: probe/removeのライフサイクル、デバイスID、電源管理
- [デバイスメモリのコヒーレンシ](device-coherency.md) — MMIO領域に対するキャッシュ属性の選択（`ioremap`か`ioremap_wc`か）
- [DMAメモリ割り当て](dma.md) — BARベースのMMIOアクセスを補完する、デバイスからRAMへのデータ経路
