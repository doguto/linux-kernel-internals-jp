# IOMMU事件簿 {#iommu-war-stories}

> 実際のインシデント: DMAフォールト、グループ分離の失敗、性能面の思わぬ落とし穴

ここに挙げるのは、本番システム、カーネルのバグ報告、公開されたポストモーテムの議論に見られる典型的なパターンに沿った、代表的な実例である。カーネルの挙動、ツール、修正内容は正確である。

---

## 1. 異常動作するNICによるDMARフォールトストーム {#1-dmar-fault-storm-from-a-misbehaving-nic}

### 問題 {#problem}

10GbE NICドライバを動かしているサーバーが、毎秒数千行のログを出し始める。

```
DMAR: [DMA Write] Request device [02:00.0] fault addr 7fa800000000
      DMAR:[fault reason 05] Write access is not set
DMAR: [DMA Write] Request device [02:00.0] fault addr 7fa800001000
      DMAR:[fault reason 05] Write access is not set
DMAR: [DMA Write] Request device [02:00.0] fault addr 7fa800002000
...
```

NICは、IOMMUがマッピングを持っていない、あるいは読み取り専用でマップしているアドレスに送信（DMA書き込み）している。ネットワークのスループットはゼロに落ちる。フォールトストームがカーネルログのリングバッファを埋め尽くし、他のメッセージを押し出してしまう。

### 診断 {#diagnosis}

フォールトアドレス`7fa800000000`は64ビットのIOVA空間にある。最初のステップは、そもそもマッピングが存在するかどうかを確認することである。

```bash
# デバイスは IOMMU ドメインを使っているか?
cat /sys/bus/pci/devices/0000:02:00.0/iommu_group
# 15

# IOMMU は有効か（パススルーではないか）?
dmesg | grep "DMAR: IOMMU"
# DMAR: IOMMU enabled

# DMAR フォールトレジスタ（Intel VT-d 固有）
# Fault reason 05 = "PTE Write access is not set"
# (Fault reason 06 = "PTE Read access is not set")
# これは、マッピングは存在するが権限が誤っている（読み取り専用マッピングへの書き込みが試みられた）ことを意味する

# そのドメインの IOVA アロケータの状態を確認する
cat /sys/kernel/debug/iommu/intel/iommu_perf_stats
```

Fault reason 05（"PTE Write access is not set"）が重要な手がかりである。マッピングは存在するが、PTEが書き込みを許可しておらず、デバイスが書き込もうとしている。これは、受信バッファを`DMA_FROM_DEVICE`ではなく`DMA_TO_DEVICE`（デバイスから見て読み取り専用）で誤ってマップしているドライバを示唆する。

```bash
# DMA API のデバッグはマップ時に方向の不一致を検出するが、これはブート時のみの
# Kconfig オプションであり、実行中に切り替えるものではない: まず
# CONFIG_DMA_API_DEBUG=y で再ビルド/ブートすること。/sys/kernel/debug/dma-api/disabled は
# 読み取り専用 -- フレームワークが自ら無効化したか（例: 追跡用プールを
# 使い果たした後）を報告するものであり、実行時のオン/オフスイッチではない。
dmesg | grep "DMA-API"
# DMA-API: device 0000:02:00.0 mapping error: wrong direction
```

### 根本原因 {#root-cause}

NICドライバのホットパスにあるバッファ補充コードのバグである。受信ディスクリプタが`DMA_FROM_DEVICE`ではなく`DMA_TO_DEVICE`でマップされていた。IOMMUがなければ、これは表面化しない。物理アドレスは方向の引数にかかわらず機能するからである。IOMMUが有効だと、カーネルは`DMA_TO_DEVICE`に対して読み取り専用のIOMMUマッピングを作成する。NICは受信したパケットデータをバッファにDMA書き込みしようとし、IOMMUがそれを阻止し、パケットごとにフォールトが発生する。

テスト環境でIOMMUが有効になっていなかったため、このバグは何年も潜在していた。

### 修正 {#fix}

```c
/* ドライバの修正: 受信バッファのマップ方向を変更する */

/* 誤り: */
dma_handle = dma_map_single(dev, buf, len, DMA_TO_DEVICE);

/* 正しい: デバイスは受信したデータをこのバッファに書き込む */
dma_handle = dma_map_single(dev, buf, len, DMA_FROM_DEVICE);
```

**ドライバの修正が出荷されるまでの本番環境での回避策:**

```bash
# iommu=pt: パススルーモード — VFIO 用に IOMMU グループは存在し続けるが、
# アドレス変換は行われない（IOVA == PA）。権限チェックがなくなる。
# カーネルコマンドラインに追加する:
GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt"
```

`iommu=pt`はDMA変換を無効にするが、IOMMUグループはそのまま保つ。信頼できるデバイスに対する正当な性能最適化ではあるが、セキュリティ上の利点はなくなる。一時的な回避策としてのみ使うこと。本当の修正はドライバの訂正と、`iommu=on`（`pt`なし）の再有効化である。

**長期的には:** CI用のカーネルで`CONFIG_DMA_API_DEBUG=y`を有効にし、方向の不一致を本番に届く前に検出する。

---

## 2. 性能ボトルネックとなったIOVAアロケータ {#2-iova-allocator-as-a-perf-bottleneck}

### 問題 {#problem_1}

あるチームがサーバーを10 GbEから25 GbEにアップグレードする。NICドライバは同じコードベースで、リンクが速くなっただけである。スループットがほぼ線形に伸びると期待していたが、実際には高パケットレート（小さなUDPパケット、約20 Mpps）でパケット処理のCPU使用率が急増する。`perf top`セッションは次のように示す。

```
Overhead  Symbol
  15.3%   alloc_iova
   8.1%   free_iova
   6.2%   iommu_map
   5.7%   iommu_unmap_fast
   ...
```

CPUの29%以上がIOVAの割り当てとマッピングに費やされている。NICドライバはパケットごとに`dma_map_single()` / `dma_map_page()`を使い、受信・送信ディスクリプタごとに1回のIOVA割り当てを発行している。

### 診断 {#diagnosis_1}

```bash
# IOMMU が有効である（パススルーでない）ことを確認する
dmesg | grep "DMAR: IOMMU"

# IOVA キャッシュの効果を見る
# 注: このパスは汎用には存在しない — /sys/kernel/debug/iommu/intel/ またはベンダー固有のパスを確認すること
cat /sys/kernel/debug/iommu/intel/iommu_perf_stats
# 出力はドメインごとの割り当て回数とキャッシュヒット率を示す

# perf: alloc_iova の呼び出し元を表示する
perf record -g -p $(pgrep irq/...-eth0) -- sleep 10
perf report --sort=symbol,dso | head -40
```

perfのコールグラフは、`alloc_iova`が`iommu_dma_map_phys` → `dma_map_page` → NICドライバのディスクリプタ補充関数から、パケットごとに1回呼ばれていることを示す。20 Mppsでは毎秒2000万回の`alloc_iova`呼び出しになり、rcacheが枯渇するたびにそれぞれがグローバルなrbtreeのspinlockに触れる。

なぜrcacheが枯渇するのか。rcacheは、一致するサイズの解放済みIOVAを保持する。2 KBの受信バッファでは、アロケータは1ページのサイズクラスを使う。しかしドライバが解放よりも速く割り当てている（処理中のディスクリプタがrcacheのマガジンサイズ127を超える）と、rcacheは空になり、ロックを取るrbtreeの経路に落ちる。

### 修正 {#fix_1}

**ステップ1: スキャッタギャザーによるDMAのバッチ化に切り替える。**

バッファごとに1回の`dma_map_single()`を呼ぶ代わりに、`dma_map_sg()`で複数のバッファをバッチ化する。これによりIOVAの割り当て回数が減る。

```c
/* IOMMU があれば、dma_map_sg() は連続した物理ページを 1 つの IOVA 範囲に
 * まとめられ、アロケータへの負荷が減る。 */
nents = dma_map_sg(dev, sgl, count, DMA_FROM_DEVICE);
```

512ディスクリプタのリングを持つNICの場合、`dma_map_sg()`で16〜32ページずつまとめて事前に補充すると、パケットごとのアロケータ呼び出しが大幅に減る。

**ステップ2: `IOMMU_DOMAIN_DMA_FQ`（フラッシュキュー）モードを使う。**

```bash
# カーネルがフラッシュキューモードをサポートしているか確認する
grep CONFIG_IOMMU_DEFAULT_DMA_LAZY /boot/config-$(uname -r)
# CONFIG_IOMMU_DEFAULT_DMA_LAZY=y
```

フラッシュキューモードを有効にすると、解放されたIOVAはすぐにはプールに返されず、バッチ化されてまとめてフラッシュされるので、IOTLB無効化のコストが償却される。

**ステップ3: rcacheのカバー範囲を確認する。**

rcacheは1、2、4、8、16、32ページの割り当てをカバーする。NICがジャンボフレーム（MTU 9000バイト、3ページ必要）を使う場合、4ページのクラスに入る。rcacheが実際にエントリを再利用できるよう、ドライバが一貫したバッファサイズを使うようにすること。

**結果**: バッチ化した`dma_map_sg()`に切り替え、フラッシュキューモードを有効にした後、IOVAアロケータのオーバーヘッドは29%から4%未満に下がり、NICはほぼラインレートのスループットを達成した。

---

## 3. VFIOパススルーを阻むIOMMUグループ {#3-iommu-group-blocking-vfio-passthrough}

### 問題 {#problem_2}

あるユーザーが、GPU計算ワークロードのためにディスクリートGPU（`0000:01:00.0`）をVMに渡したいと考える。標準的なVFIOのバインド手順に従う。

```bash
echo "10de 1db6" | sudo tee /sys/bus/pci/drivers/vfio-pci/new_id
echo "0000:01:00.0" | sudo tee /sys/bus/pci/devices/0000:01:00.0/driver/unbind
echo "0000:01:00.0" | sudo tee /sys/bus/pci/drivers/vfio-pci/bind
```

するとQEMUで次のエラーが出る。

```
qemu-system-x86_64: -device vfio-pci,host=01:00.0: \
    vfio: error, group 12 is not viable, please ensure all devices
    within the iommu group are bound to their vfio driver:
    /sys/kernel/iommu_groups/12/devices/0000:01:00.1
```

GPUのHDMIオーディオファンクション（`01:00.1`）が同じIOMMUグループにあり、まだ`snd_hda_intel`にバインドされている。

### 診断 {#diagnosis_2}

```bash
# グループ内のすべてのデバイスを見る
ls /sys/kernel/iommu_groups/12/devices/
# 0000:01:00.0   (GPU, vfio-pci にバインド済み)
# 0000:01:00.1   (GPU HDMI audio, snd_hda_intel にバインド済み)

# なぜ同じグループにあるのか確認する
# 上流ポートの ACS ケーパビリティを探す
lspci -vvv -s 0000:00:01.0 | grep "Access Control"
# (何も出ない — ルートポートに ACS ケーパビリティがない)
```

GPUとそのオーディオファンクションは、同じPCIeスロット上のマルチファンクションデバイスである。上流ポートでACSが有効でない（あるいはポートがACSをまったくサポートしていない）と、IOMMUはそれらを分離できない。2つのファンクション間のピアツーピアDMAはIOMMUを迂回する。カーネルはそれらを正しく同じグループにまとめている。

VFIOは、グループ全体をまとめて割り当てることを強制する。GPUをVM1に、オーディオをVM2に渡すことも、GPUをVMに渡しながらオーディオをホストに残すこともできない。

### 選択肢 {#options}

**選択肢1（正しく安全）: 両方のデバイスをVMに渡す。**

```bash
# オーディオも snd_hda_intel からアンバインドする
echo "0000:01:00.1" | sudo tee /sys/bus/pci/devices/0000:01:00.1/driver/unbind
echo "0000:01:00.1" | sudo tee /sys/bus/pci/drivers/vfio-pci/bind

# QEMU: 両方を渡す
-device vfio-pci,host=01:00.0 \
-device vfio-pci,host=01:00.1
```

VMはGPUとそのオーディオの両方を得る。これが正しく安全な方法である。

**選択肢2: ACSオーバーライドパッチ（安全ではない。リスクを理解すること）。**

コミュニティの一部のカーネルパッチ（`pcie_acs_override=downstream,multifunction`ブートパラメータ）は、ハードウェアのACSがなくても各PCIeファンクションを分離されているものとしてカーネルに扱わせる。これにより、デバイスは共有グループから人為的に切り離される。

```bash
# ACS オーバーライドパッチを適用した状態で:
GRUB_CMDLINE_LINUX="intel_iommu=on pcie_acs_override=downstream,multifunction"
```

**リスク:** このオーバーライドは、分離についてIOMMUサブシステムを欺く。GPUの計算エンジンがオーディオファンクションのメモリにDMAでき（一部のハードウェアでは実際にできる）、かつ両者が別々のVMに割り当てられていると、VM1のデバイスがVM2のメモリにアクセスできてしまう。これはマルチテナントシステムにとって現実の脆弱性である。シングルユーザーのワークステーションでは実際のリスクは低いが、カーネルは意図的にこのパッチをメインラインに含めていない。

**選択肢3: ACS対応のルートポートを持つハードウェアを買う。**

サーバー向けのPCIeスイッチやルートコンプレックス（EPYC、Xeon Scalable）は、多くの場合ファンクションごとにACSをサポートしており、各PCIeデバイスが独自のIOMMUグループを持てる。コンシューマー向けチップセットはサポートしていないことが多い。

---

## 4. アクセラレータによるSVAページフォールトストーム {#4-sva-page-fault-storm-from-an-accelerator}

### 問題 {#problem_3}

あるチームが、ユーザー空間のコピーオフロードのために、SVAを有効にしたIntel DSA（Data Streaming Accelerator）を導入する。アプリケーションはプロセスの仮想アドレスを使って数千のメモリコピーディスクリプタを投入する。最初はうまく動く。負荷がかかると、システムは劇的に遅くなり、`dmesg`に次のように表示される。

```
IOMMU: dmar1: PRQ overflow detected
IOMMU: dmar1: PRQ overflow cleared
```

アプリケーションのスループットは80%低下する。割り込み処理のCPU使用率が急上昇する。注: PRI/PRQのオーバーフロー処理は完全にIOMMUドライバ（`drivers/iommu/intel/prq.c`）の中にある。アクセラレータのドライバ自体（`idxd`）はページ要求キューについて何も出力しない。上の`pr_info_ratelimited()`呼び出しは、アクセラレータのPCIアドレスではなく`iommu->name`（例: `dmar1`）でタグ付けされている。

### 診断 {#diagnosis_3}

アクセラレータは操作の途中でページフォールトに遭遇し、IOMMUにPage Request（PRI）を送っている。IOMMUはこれをカーネルのIOMMUページフォールトハンドラに配送し、ハンドラは次のことをしなければならない。

1. フォールトしたPASIDに対応する`mm_struct`を見つける。
2. `handle_mm_fault()`を呼んでページを読み込む。
3. Page Request Response（PRG Response）をデバイスに返す。

投入レートが高いと、デバイスのPRIキュー（固定サイズのハードウェアFIFO）は、カーネルが排出するよりも速く埋まる。キューがあふれると、デバイスは保留中の作業をストールさせ、操作を中断することがある。

```bash
# DSA デバイスの状態と PRI キューの深さを確認する
cat /sys/bus/pci/devices/0000:6a:01.0/idxd/dsa0/state
# enabled

# ページフォールトハンドラの活動（tracepoint）
echo 1 > /sys/kernel/tracing/events/iommu/io_page_fault/enable
cat /sys/kernel/tracing/trace | head -30
# io_page_fault: IOMMU:idxd 0000:6a:01.0 iova=0x00007f3c80000000 flags=0x0001

# THP の分割が起きているか確認する（フォールトの主要な発生源）
grep thp_split /proc/vmstat
```

このケースの根本原因:
1. **遅延割り当て**: アプリケーションは`mmap()`を呼ぶが、投入前にすべてのページに触れていない。DSAは最初のアクセスで存在しないPTEに遭遇する。
2. **THPの分割**: カーネルは最初バッファを2 MBのhuge pageで裏付けていたが、メモリ逼迫によってワークロードの途中でそれらが4 KBのページに分割され、デバイスのATCエントリが無効化されて再フォールトが大量に発生した。

### 緩和策 {#mitigation}

**投入前にページを事前フォールトさせる:**

```c
/* アプリケーション: DSA に投入する前にすべてのページをフォールトインさせる */
madvise(buf, len, MADV_POPULATE_WRITE);
/* または mlock() を使ってフォールトとピン留めの両方を行う */
mlock(buf, len);
```

`MADV_POPULATE_WRITE`（Linux 5.14で追加）は、root権限なしですべてのページをフォールトインさせ、初回アクセス時のフォールトをなくす。

**アクセラレータのバッファではTHPを無効にする:**

```c
/* THP がバッファを裏付けるのを防ぐ（分割による再フォールトを避ける） */
madvise(buf, len, MADV_NOHUGEPAGE);
```

**PRIキューの深さを増やす**（ハードウェアがサポートしていれば。DVSECレジスタを確認すること）。

**ホットパスには事前にピン留めしたバッファを使う:**

最もホットなコピー経路では、バッファを事前にカーネルに登録し、セッションの存続期間中ピン留めしておく。DPDKの`idxd` PMDはこれを行っている。ピン留めした固定のバッファ集合をマップしてその中だけで投入することで、クリティカルパスではSVAのページフォールトを完全に避けつつ、一般的なケースではSVAを使う。

---

## 5. IOMMU下でのswiotlb枯渇 {#5-swiotlb-exhaustion-under-iommu}

### 問題 {#problem_4}

あるサーバーはさまざまなワークロードを混在させて動かしている。ほとんどは64ビットのNVMe SSDだが、32ビットのDMAマスクを持つレガシーなRAIDコントローラもある。システムのRAMは256 GBである。重いI/O負荷の下で、RAIDコントローラがDMAの割り当てに失敗し始める。

```
kernel: ata1: SCSI error: return code = 0x08000002
kernel: end_request: I/O error, dev sda, sector 0
kernel: DMA: Out of SW-IOMMU space for 131072 bytes at device 0000:04:00.0
```

そしてそれより前のdmesgには次のようにある。

```
software IO TLB: mapped [mem 0x...] (64MB)
```

swiotlbプールは64 MBで、それが枯渇している。

### 診断 {#diagnosis_4}

RAIDコントローラ（`0000:04:00.0`）は32ビットのDMAマスクを持つ。システムのRAMはほとんどが4 GBより上にある。IOMMUが有効なので、カーネルはこのデバイスに4 GB未満のIOVAを割り当てる。しかし微妙な点がある。

IOMMUが有効で32ビットデバイスが存在する場合でも、カーネルは特定のケースでswiotlbを使う。

1. **4 GB未満でのIOVA割り当ての失敗**: 低位のIOVA空間が断片化または枯渇している（`dma_32bit_pfn`領域の割り当て要求が過剰になっている）と、IOMMU DMA層はswiotlbにフォールバックする。
2. **IOMMUを通らない経路でのバウンス**: IOMMUが初期化される前の初期ブート中や、まだIOMMUドメインにアタッチされていないデバイスの場合。

```bash
# swiotlb の使用状況を確認する
cat /sys/kernel/debug/swiotlb/io_tlb_nslabs
# 32768  (64MB / スロットあたり 2KB)

cat /sys/kernel/debug/swiotlb/io_tlb_used
# 32768  ← 完全に枯渇

# 4GB 未満の IOVA 空間を確認する
# 注: このパスは汎用には存在しない — /sys/kernel/debug/iommu/intel/ またはベンダー固有のパスを確認すること
# 32 ビット領域の状態を表示する

# どのデバイスが 32 ビット IOVA を奪い合っているか?
for dev in /sys/bus/pci/devices/*/; do
    mask=$(cat $dev/dma_mask_bits 2>/dev/null)
    [ "$mask" = "32" ] && echo $dev
done
```

問題はこうである。複数のデバイス（RAIDコントローラ、レガシーなUSBコントローラ、古いネットワークカード）がすべて32ビットのDMAマスクを持っている。これらは32ビットのIOVA領域を共有する。負荷の下で、NVMeドライバ（64ビット）は32ビットのIOVA空間を消費していないが、RAIDドライバは低位のIOVA空間を断片化させる大きな要求を送り、割り当ての失敗を引き起こす。カーネルは失敗したIOVA割り当てについてswiotlbにフォールバックし、64 MBのswiotlbプール（デフォルト）は128 KBのRAID I/O要求ですぐに埋まってしまう。

### 修正 {#fix_2}

**swiotlbプールのサイズを増やす**（カーネルコマンドライン）:

```bash
# swiotlb を 512MB に増やす（値は MB 単位、または swiotlb=<num_slabs> を使う）
GRUB_CMDLINE_LINUX="swiotlb=262144"
# 262144 スラブ × スラブあたり 2KB = 512MB
# 注: カーネルの swiotlb パーサーは単純な整数（2KB スロットの数）しか受け付けない。
# 512MB にするには swiotlb=262144 を使う。メインラインでは m/M サフィックスはサポートされていない。
```

**64ビットデバイスが32ビットのIOVA空間を消費しないようにする。**

ドライバを監査し、サポートされている場合は`dma_set_mask_and_coherent(dev, DMA_BIT_MASK(64))`を呼ぶようにする。64ビットのマスクを設定し損ねたドライバは、一部のカーネルではデフォルトで32ビットになり、不必要に低位のIOVAを奪い合う。

```c
/* ドライバの probe での良い慣行: */
ret = dma_set_mask_and_coherent(dev, DMA_BIT_MASK(64));
if (ret) {
    dev_warn(dev, "Unable to set 64-bit DMA mask, falling back to 32-bit\n");
    ret = dma_set_mask_and_coherent(dev, DMA_BIT_MASK(32));
    if (ret) {
        dev_err(dev, "No usable DMA configuration\n");
        return ret;
    }
}
```

**レガシーな32ビットハードウェアの置き換えを検討する。**

RAIDコントローラが唯一の32ビットデバイスであれば、最新のHBAに置き換えることで問題は完全になくなる。swiotlbは主要なDMA経路ではなく安全網として存在する。swiotlbが継続的に使われているということは、システムが最適に動いていないということである。

**先回りして監視する:**

```bash
# swiotlb の使用率が 80% を超えたら警告する
USED=$(cat /sys/kernel/debug/swiotlb/io_tlb_used)
TOTAL=$(cat /sys/kernel/debug/swiotlb/io_tlb_nslabs)
echo "$USED / $TOTAL"
# 監視に追加する（Prometheus node_exporter のカスタムコレクタなど）
```

---

## よくあるパターン {#common-patterns}

| 症状 | 最初に使うツール | 考えられる原因 |
|---------|-----------|-------------|
| dmesgに`DMAR: [fault reason NN]`が大量に出る | `dmesg \| grep DMAR` | DMAの方向の誤り、古いマッピング、またはデバイスに対してIOMMUが設定されていない |
| `perf top`で`alloc_iova`が高い | `perf report`、ベンダー固有のdebugfs（例: `/sys/kernel/debug/iommu/intel/`） | IOVA rcacheのミス、パケットごとのマッピング、不適切なバッファサイズ |
| VFIOの「group not viable」エラー | `ls /sys/kernel/iommu_groups/<N>/devices/` | 1つのグループに複数のデバイスがある。すべてをvfio-pciにバインドする必要がある |
| PRIオーバーフローを伴うDSA/アクセラレータのストール | `dmesg \| grep "page request"`、iommuのtracepoint | ページが事前フォールトされていない、THPの分割、PRIキューが小さすぎる |
| `DMA: Out of SW-IOMMU space` | `cat /sys/kernel/debug/swiotlb/io_tlb_used` | swiotlbプールの枯渇。大きな割り当てを行う32ビットデバイス |

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [drivers/iommu/intel/dmar.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu/intel/dmar.c) — ケース1のDMARフォールトコードの背後にある`dmar_fault_do_one()`と`dma_remap_fault_reasons[]`テーブル（reason 5 = "PTE Write access is not set"、reason 6 = "PTE Read access is not set"）
- [kernel/dma/debug.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma/debug.c) — ケース1で言及した`CONFIG_DMA_API_DEBUG`の計装フレームワーク
- [drivers/iommu/iova.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu/iova.c) — ケース2のアロケータのボトルネックの背後にある`alloc_iova()`、`alloc_iova_fast()`、per-CPU IOVA rcache（`IOVA_RANGE_CACHE_MAX_SIZE`、`IOVA_MAG_SIZE`）
- [drivers/iommu/dma-iommu.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu/dma-iommu.c) — ケース2でIOVA割り当てを駆動するストリーミングDMAの入口である`iommu_dma_map_phys()`と`iommu_dma_map_sg()`
- [drivers/iommu/iommu.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu/iommu.c) — ケース2の修正の背後にある`iommu_map()`、`iommu_unmap_fast()`、`IOMMU_DOMAIN_DMA_FQ`フラッシュキュードメインタイプ
- [drivers/pci/pci.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pci/pci.c) — ケース3のIOMMUグループ分離の背後にある、カーネル側のPCIe ACSネゴシエーション`pci_enable_acs()`
- [drivers/iommu/iommu-sva.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu/iommu-sva.c) — ケース4の背後にある`iommu_sva_handle_mm()`。`handle_mm_fault()`を呼んでデバイスのページ要求を解決する
- [drivers/iommu/intel/prq.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/iommu/intel/prq.c) — ケース4のPRIキューのシナリオの基盤となるIntel IOMMUのPage Request Queueオーバーフロー処理。`pr_info_ratelimited("IOMMU: %s: PRQ overflow detected", iommu->name)`メッセージを含む
- [include/trace/events/iommu.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/trace/events/iommu.h) — ケース4の診断で使った`io_page_fault` tracepoint（`TP_printk("IOMMU:%s %s iova=0x%016llx flags=0x%04x", ...)`）
- [mm/madvise.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/madvise.c) — ケース4で使った緩和策である`MADV_POPULATE_WRITE`と`MADV_NOHUGEPAGE`
- [kernel/dma/swiotlb.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma/swiotlb.c) — ケース5の背後にあるswiotlbバウンスバッファプールと、その`io_tlb_used`/`io_tlb_nslabs` debugfsカウンタ

### manページ {#man-pages}

- [`madvise(2)`](https://man7.org/linux/man-pages/man2/madvise.2.html) — ケース4で使った`MADV_POPULATE_WRITE`（Linux 5.14以降）と`MADV_NOHUGEPAGE`を説明している
- [`lspci(8)`](https://man7.org/linux/man-pages/man8/lspci.8.html) — ケース3でACSケーパビリティの確認に使ったPCIトポロジーツール

### 関連ページ {#related-pages}

- [IOMMUアーキテクチャ](iommu-arch.md) — IOMMUドメイン、IOTLB、DMARフォールトレジスタ
- [DMA API](dma-api.md) — `dma_map_single`、スキャッタギャザー、swiotlbの内部
- [IOVAアロケータ](iova-allocator.md) — rcache、フラッシュキュー、32ビットの制限
- [SVA](sva.md) — PASID、PRIによるページフォールト処理
- [VFIOの内部](vfio-internals.md) — IOMMUグループ、セキュリティモデル

### LWNの記事 {#lwn-articles}

- [Kernel Summit 2006: DMA and IOMMU issues](https://lwn.net/Articles/191931/)（2006年7月19日） — マッピングごとのオーバーヘッドを減らすためのスキャッタギャザーのバッチ化（`dma_map_sg()`）に関する初期の議論。ケース2の修正の背後にあるのと同じ原理
- [Driver API: sleeping poll(), exclusive I/O memory, and DMA API debugging](https://lwn.net/Articles/308426/)（2008年11月24日） — ケース1で方向の不一致を検出するのに使った`CONFIG_DMA_API_DEBUG`フレームワークを取り上げている
- [Safe device assignment with VFIO](https://lwn.net/Articles/474088/)（2012年1月3日） — IOMMUグループが存在する理由と、グループ内のすべてのデバイスが同じドライバにバインドされなければならない理由を説明している。ケース3の背後にある仕組み
- [A security fix briefly breaks DMA](https://lwn.net/Articles/889593/)（2022年4月1日） — swiotlbバウンスバッファの仕組みと、セキュリティ上の前提とドライバの挙動の間の緊張関係。ケース5の背景

### 外部資料 {#external}

- [Kernel documentation: DMA API](https://docs.kernel.org/core-api/dma-api.html)と[DMA API HOWTO](https://docs.kernel.org/core-api/dma-api-howto.html) — ケース1の中心にある`dma_map_single()`と`DMA_TO_DEVICE`/`DMA_FROM_DEVICE`の方向フラグ
- [Kernel documentation: VFIO](https://docs.kernel.org/driver-api/vfio.html) — ケース3の背後にある「group is viable」チェック（グループ内のすべてのデバイスがvfioにバインドされていなければならない）
- [Kernel documentation: Shared Virtual Addressing (SVA) with ENQCMD](https://docs.kernel.org/arch/x86/sva.html) — ケース4の背後にあるPASID、ATS、PRIのページフォールトの仕組み
- [Kernel boot parameters](https://docs.kernel.org/admin-guide/kernel-parameters.html) — ケース1、3、5で使った`iommu=pt`、`intel_iommu=`、`swiotlb=`
