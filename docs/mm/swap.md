# スワップ {#swap}

> メモリをディスクへと拡張する

## スワップとは何か {#what-is-swap}

スワップにより、カーネルはあまり使われないpageをRAMからディスクへ移し、活発に使うためのメモリを空けることができる。それらのpageが再び必要になると、スワップから読み戻される。

```
RAM Full, need more memory:
┌────────────────────────────────────────────┐
│ RAM: [Active] [Active] [Inactive] [Active] │
└────────────────────────────────────────────┘
                    │
                    ▼ (swap out)
┌────────────────────────────────────────────┐
│ RAM: [Active] [Active] [FREE] [Active]     │
└────────────────────────────────────────────┘
┌────────────────────────────────────────────┐
│ Swap: [Inactive page]                      │
└────────────────────────────────────────────┘

Later, page accessed:
                    │
                    ▼ (swap in)
┌────────────────────────────────────────────┐
│ RAM: [Active] [Page back] [X] [Active]     │
└────────────────────────────────────────────┘
```

## スワップの種類 {#swap-types}

### スワップパーティション {#swap-partition}

スワップ専用のディスクパーティション:

```bash
# スワップパーティションを作成する（インストール時、または fdisk で）
mkswap /dev/sda2
swapon /dev/sda2

# 有効なスワップを表示する
swapon --show
# NAME      TYPE      SIZE USED PRIO
# /dev/sda2 partition 8G   1.2G -2
```

### スワップファイル {#swap-file}

スワップとして使う通常のファイル:

```bash
# 4GB のスワップファイルを作成する（ext4 や同様のファイルシステムで動作）
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# /etc/fstab で永続化する:
# /swapfile none swap sw 0 0
```

**注:** ext4では `fallocate` で確実にスワップファイルを作成できる。XFSや一部の他のファイルシステムでは、カーネルがスワップに使えないエクステントが作られることがある。どこでも安全な代替手段としては `dd if=/dev/zero of=/swapfile bs=1M count=4096` を使う。

**btrfs（COWファイルシステム）についての注:** `+C`（no-copy-on-write）属性は[ファイルにデータが入る前に設定しなければならない](https://btrfs.readthedocs.io/en/latest/Swapfile.html)（[`chattr(1)`](https://man7.org/linux/man-pages/man1/chattr.1.html) を参照）。

```bash
# 方法 1: 空のファイルを作成し、+C を設定してから領域を確保する
touch /swapfile
chattr +C /swapfile
fallocate -l 4G /swapfile
chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile

# 方法 2: 親ディレクトリに +C を設定する（新しいファイルはこれを継承する）
mkdir /swap && chattr +C /swap
fallocate -l 4G /swap/swapfile
# ... あとは通常どおり mkswap、swapon
```

### zswap（圧縮スワップキャッシュ） {#zswap-compressed-swap-cache}

ディスクに書き込む前にpageを圧縮し、多くの場合ディスクI/Oを完全に回避する。

```
Page to swap out
       │
       v
Compress in RAM (zswap pool)
       │
       ├── Fits in pool? ──► Store compressed (no disk I/O)
       │
       └── Pool full? ──► Write to backing swap device
```

```bash
# zswap を有効にする
echo 1 > /sys/module/zswap/parameters/enabled

# 圧縮器とプールを設定する
echo lz4 > /sys/module/zswap/parameters/compressor
echo 20 > /sys/module/zswap/parameters/max_pool_percent

# 統計を表示する
grep -r . /sys/kernel/debug/zswap/ 2>/dev/null
```

### zram（圧縮RAMディスク） {#zram-compressed-ram-disk}

圧縮機能付きのRAMベースのブロックデバイス。ディスクなしのスワップである。

```bash
# モジュールをロードする
modprobe zram

# サイズを設定する（圧縮により実効サイズはこれより大きくなる）
echo 4G > /sys/block/zram0/disksize

# スワップとして使う
mkswap /dev/zram0
swapon /dev/zram0 -p 100  # ディスクのスワップより高い優先度
```

## スワップのアーキテクチャ {#swap-architecture}

### スワップ領域 {#swap-areas}

Linuxは優先度付きの複数のスワップ領域をサポートする。

```bash
# スワップ領域を表示する
cat /proc/swaps
# Filename        Type        Size      Used    Priority
# /dev/sda2       partition   8388604   1234560 -2
# /dev/zram0      partition   4194300   567890  100
```

優先度の高いスワップから先に使われる。優先度が等しい場合はストライプされる（ラウンドロビン）。

### スワップキャッシュ {#swap-cache}

最近スワップインされたpageは、しばらくスワップキャッシュに残る。

```
Swap cache:
┌─────────────────────────────────────────┐
│ Page in RAM + still on swap             │
│ (if process forks, child can share)     │
└─────────────────────────────────────────┘
```

スワップインされたpageが変更されずに再びスワップアウトされる場合、ディスクへの書き込みは不要である。

### スワップスロット {#swap-slots}

スワップ領域はスロット（pageごとに1つ）に分割される。

```c
/* 各スロットは 1 つのスワップされた page を追跡する */
swap_entry_t entry = swp_entry(type, offset);
/* type: どのスワップ領域か */
/* offset: その領域内のスロット */
```

## スワップの仕組み {#swapping-mechanics}

### スワップアウト（pageからディスクへ） {#swap-out-page-to-disk}

```
Memory pressure
       │
       v
Select victim page (from inactive list)
       │
       v
Allocate swap slot
       │
       v
Write page to swap
       │
       v
Update PTE: present=0, swap_entry=slot
       │
       v
Free page frame
```

### スワップイン（ディスクからpageへ） {#swap-in-disk-to-page}

```
Process accesses swapped page
       │
       v
Page fault (not present)
       │
       v
Read swap entry from PTE
       │
       v
Allocate page frame
       │
       v
Read from swap into page
       │
       v
Update PTE: present=1, page_frame=new
       │
       v
Resume process
```

## 設定 {#configuration}

### swappiness {#swappiness}

匿名pageをスワップするか、ファイルキャッシュを破棄するかの優先度を制御する（[vm sysctlのドキュメント](https://docs.kernel.org/admin-guide/sysctl/vm.html)を参照）。

```bash
cat /proc/sys/vm/swappiness
# 60（デフォルト）

# 低い = キャッシュの破棄を優先し、スワップを避ける
echo 10 > /proc/sys/vm/swappiness

# 高い = よりスワップしやすくなる
echo 80 > /proc/sys/vm/swappiness

# 0 = OOM を避けるためにのみスワップする（決してスワップしないわけではない）
```

### スワップの優先度 {#swap-priority}

```bash
# 有効化するときに優先度を設定する
swapon -p 100 /dev/zram0    # 高い優先度
swapon -p -2 /dev/sda2      # 低い優先度（デフォルト）

# /etc/fstab では:
# /dev/zram0 none swap sw,pri=100 0 0
# /dev/sda2  none swap sw,pri=-2  0 0
```

### オーバーコミット {#overcommit}

詳細は[vm sysctlのドキュメント](https://docs.kernel.org/admin-guide/sysctl/vm.html)を参照。

```bash
# メモリのオーバーコミットポリシー
cat /proc/sys/vm/overcommit_memory
# 0 = ヒューリスティック（デフォルト）- 妥当なオーバーコミットを許可
# 1 = 常に許可 - malloc は決して失敗しない
# 2 = 厳格 - スワップ + ratio*RAM に制限

# モード 2 の場合の比率:
cat /proc/sys/vm/overcommit_ratio
# 50（デフォルト）= スワップ + RAM の 50%
```

## 監視 {#monitoring}

### スワップ使用量 {#swap-usage}

```bash
# 簡易表示
free -h
#                total   used   free  shared  buff/cache  available
# Swap:           8.0G   1.2G   6.8G

# 詳細
cat /proc/meminfo | grep -i swap
# SwapCached:    123456 kB  (スワップと RAM の両方にある page)
# SwapTotal:    8388604 kB
# SwapFree:     7000000 kB
```

### スワップの活動 {#swap-activity}

```bash
# スワップイン/アウトされた page
cat /proc/vmstat | grep -E "pswpin|pswpout"
# pswpin  - スワップから読み込まれた page
# pswpout - スワップに書き込まれた page

# リアルタイム監視
vmstat 1
# si = スワップイン（KB/s）
# so = スワップアウト（KB/s）
```

### プロセスごとのスワップ {#per-process-swap}

```bash
# プロセスごとのスワップ使用量
cat /proc/<pid>/status | grep -i swap
# VmSwap:     1234 kB

# システム全体の合計
awk '/VmSwap/{sum+=$2} END {print sum" kB"}' /proc/*/status 2>/dev/null

# スワップ消費の多いプロセス
grep VmSwap /proc/*/status 2>/dev/null | sort -k2 -n | tail
```

### zswapの統計 {#zswap-statistics}

```bash
cat /sys/kernel/debug/zswap/pool_total_size     # 圧縮後のサイズ
cat /sys/kernel/debug/zswap/stored_pages        # zswap 内の page
cat /sys/kernel/debug/zswap/written_back_pages  # ディスクに追い出された page
```

## 変遷 {#evolution}

### 最初のスワップ（v0.12、1992年1月） {#original-swap-v012-january-1992}

スワップ（仮想メモリ）はLinux 0.12（1992年初頭）で追加された。[RELNOTES-0.12](https://www.kernel.org/pub/linux/kernel/Historic/old-versions/RELNOTES-0.12)には、新しい `mkswap` プログラムとスワップの有効化について記載されている。Linux 0.01にはスワップのサポートがなかった。

### 複数のスワップ領域（v1.3） {#multiple-swap-areas-v13}

優先度付きの複数のスワップパーティションのサポート。

### スワップファイル（v2.6） {#swap-files-v26}

スワップファイルはパーティションと同じくらい効率的になった。カーネルはswapon時にファイルのブロックをディスクのセクタにマップし、ファイルシステムを完全にバイパスして直接ブロックI/Oを行う（詳細は[スワッピング: スワップ領域の構成](swapping.md#swap-space-organization)を参照）。

### zswap（v3.11、2013年） {#zswap-v311-2013}

**コミット**: [2b2811178e85](https://git.kernel.org/linus/2b2811178e85) ("zswap: add to mm/") | [LKML](https://lore.kernel.org/lkml/1361397888-14863-1-git-send-email-sjenning@linux.vnet.ibm.com/)

ディスクI/Oを減らすための圧縮スワップキャッシュ。

### zramスワップ（v3.14、2014年） {#zram-swap-v314-2014}

**コミット**: [cd67e10ac699](https://git.kernel.org/linus/cd67e10ac699) ("zram: promote zram from staging")

zramがstagingからメインラインに移り、ディスクレスのスワップが可能になった。

### THPスワップ（v4.13、2017年） {#thp-swap-v413-2017}

**コミット**: [38d8b4e6bdc8](https://git.kernel.org/linus/38d8b4e6bdc8) ("mm, THP, swap: delay splitting THP during swap out") | [LKML](https://lore.kernel.org/lkml/20170724051840.2309-1-ying.huang@intel.com/)

**作者**: Huang Ying

Transparent Huge Pagesを、先に分割することなくスワップできるようになった。

### Swap-over-NFS（実験的） {#swap-over-nfs-experimental}

ディスクレスシステムのために、ネットワークストレージへのスワップを可能にする作業が進行中である。

## スワップあり対スワップなし {#swap-vs-no-swap}

### スワップを使う論拠 {#arguments-for-swap}

| 利点 | 説明 |
|---------|-------------|
| OOMの防止 | スワップはOOMキラーが動く前の緩衝となる |
| ハイバネーション | suspend-to-diskにはスワップが必要 |
| アイドルなpageの追い出し | 使われていないpageを外に移せる |
| オーバーコミットの安全性 | メモリ使用量の急増に対する余裕が増える |

### スワップを使わない論拠 {#arguments-against-swap}

| 懸念 | 説明 |
|---------|-------------|
| レイテンシ | スワップは遅く、ハングを引き起こしうる |
| SSDの摩耗 | 頻繁なスワップはフラッシュを摩耗させる |
| スラッシング | 激しいスワップ = システムが使い物にならない |
| メモリ問題の隠蔽 | メモリリークを覆い隠す |

### 推奨 {#recommendation}

ほとんどのシステムはいくらかのスワップの恩恵を受ける。カーネル自体にはサイズの推奨はなく、これはディストリビューションレベルの判断である。[Red HatのRHEL 9ガイド](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_storage_devices/getting-started-with-swap_managing-storage-devices)はよく参照されるガイドラインである。

| RAM | スワップ（ハイバネーションなし） | スワップ（ハイバネーションあり） |
|-----|----------------------|------------------------|
| ≤ 2 GB | RAMの2倍 | RAMの3倍 |
| 2-8 GB | RAMと同じ | RAMの2倍 |
| 8-64 GB | 少なくとも4 GB | RAMの1.5倍 |
| > 64 GB | 少なくとも4 GB | 推奨されない |

その上にzswapを加えると、ディスクI/Oが大幅に減る。組み込みシステムは、ストレージが限られワークロードが予測可能なため、スワップを使わないことが多い。

## よくある問題 {#common-issues}

### スワップスラッシング {#swap-thrashing}

絶え間ないスワップでシステムが使い物にならなくなる。

**症状**: vmstatの `si`/`so` が高く、システムが応答しない

**解決策**:

- RAMを増やす
- ワークロードを減らす
- swappinessを下げる
- メモリを大量に消費するプロセスを終了させる

### スワップが満杯 {#swap-full}

使えるスワップ領域がない。

**症状**: 「空き」メモリがあるのにOOMキルが起きる

**解決策**:

- スワップを増やす
- zswap/zramを有効にする
- メモリ使用状況を調査する

### SSDの摩耗 {#ssd-wear}

過剰なスワップ書き込みがSSDを摩耗させる。

**解決策**:

- zswapを使う（書き込みを2〜5分の1に減らす）
- swappinessを下げる
- RAMを増やす

## リファレンス {#references}

### 主要なコード {#key-code}

| ファイル | 説明 |
|------|-------------|
| [`mm/swapfile.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/swapfile.c) | スワップ領域の管理 |
| [`mm/swap_state.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/swap_state.c) | スワップキャッシュ |
| [`mm/zswap.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/zswap.c) | 圧縮スワップキャッシュ |
| [`drivers/block/zram/`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/block/zram) | zramの実装 |

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/admin-guide/mm/zswap.rst`](https://docs.kernel.org/admin-guide/mm/zswap.html)

### 関連 {#related}

- [reclaim](reclaim.md) - スワップがいつ起動されるか
- [page-cache](page-cache.md) - ファイルpageと匿名page
- [mmap](mmap.md) - スワップされる匿名メモリ

## 参考文献 {#further-reading}

- [mm/swap_state.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/swap_state.c) — スワップキャッシュの実装: スワップキャッシュへのpageの追加、検索、無効化
- [mm/swapfile.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/swapfile.c) — スワップ領域の管理: `swapon`/`swapoff`、スロットの割り当て、優先度の処理、スワップマップ
- `Documentation/admin-guide/mm/concepts.rst` — 匿名メモリ、スワップ、回収の概念についてのカーネルドキュメントの概要
- [Toward a better swapping policy](https://lwn.net/Articles/645539/)（LWN、2015年） — swappiness、回収のコストモデル、カーネルがファイルキャッシュと匿名メモリのスワップのバランスをどう取るかについての議論
- [zswap](zswap.md) — pageがディスクのスワップデバイスに到達する前に横取りする圧縮スワップキャッシュ
- [swap-thrashing](swap-thrashing.md) — ワーキングセットがRAMを超えたときに起きるライブロックの検出と回復
- [swapping](swapping.md) — スワップインとスワップアウトのpageレベルの仕組み。スワップキャッシュのライフサイクルを含む
- [reclaim](reclaim.md) — カーネルが追い出す犠牲pageをどう選び、メモリ逼迫時にスワップアウトをどう起動するか
