# ページキャッシュ {#page-cache}

> ファイルデータをメモリにキャッシュする

## ページキャッシュとは何か {#what-is-the-page-cache}

ページキャッシュは、最近アクセスされたファイルデータをRAMに保持する。ファイルを読むとデータはメモリに残り、以降の読み取りはディスクではなくRAMから行われる。

```
First read:
Application ──► Kernel ──► Storage (slow: ~10ms HDD, ~100μs NVMe)
                  │
                  ▼
             Page Cache (stores copy)

Second read:
Application ──► Kernel ──► Page Cache (fast, ~100ns)
                           (storage not touched)
```

## なぜページキャッシュか {#why-page-cache}

### 速度の差 {#the-speed-gap}

| ストレージ | レイテンシ | 帯域幅 |
|---------|---------|-----------|
| RAM | ~100ns | ~100 GB/s |
| NVMe SSD | ~100µs | ~7 GB/s |
| SATA SSD | ~100µs | ~600 MB/s |
| HDD | ~10ms | ~200 MB/s |

（桁感を示す数値である。[Latency Numbers Every Programmer Should Know](https://colin-scott.github.io/personal_website/research/interactive_latency.html)を参照。）

RAMはストレージより1000〜100,000倍速い。キャッシュは時間的局所性、すなわち最近アクセスされたデータは再びアクセスされやすいという性質を利用する。

### 利点 {#benefits}

- **読み取り性能**: ホットなデータはRAMから提供される
- **書き込みのバッファリング**: 書き込みは同期的ではなくまとめて行われる
- **データの共有**: 複数のプロセスがキャッシュされたページを共有する
- **メモリ効率**: 使われていないRAMがキャッシュになる

## 仕組み {#how-it-works}

### ファイルの読み取り {#reading-files}

```
read(fd, buf, 4096)
        │
        v
Find page in cache? ─── Yes ──► Copy to user buffer
        │
        No
        │
        v
Allocate page
        │
        v
Read from disk into page
        │
        v
Add to page cache
        │
        v
Copy to user buffer
```

### ファイルへの書き込み {#writing-files}

```
write(fd, buf, 4096)
        │
        v
Find/allocate page in cache
        │
        v
Copy from user buffer to page
        │
        v
Mark page dirty
        │
        v
Return to application (write "complete")
        │
        ▼ (later, asynchronously)
Writeback thread flushes to disk
```

書き込みは即座に戻る。データはキャッシュ内にあり、dirtyとマークされている。実際のディスクI/Oはバックグラウンドのライトバックが処理する。

### address_space構造体 {#the-address_space-structure}

各ファイル（inode）は、キャッシュされたページを管理する`address_space`を持つ。

```c
struct address_space {
    struct inode *host;           /* 所有するinode */
    struct xarray i_pages;        /* キャッシュされたページ */
    unsigned long nrpages;        /* キャッシュされたページ数 */
    const struct address_space_operations *a_ops;  /* 操作 */
    /* ... */
};
```

ページはXArray内でファイルオフセットによってインデックス付けされ、高速な検索が可能になっている。

## dirtyページとライトバック {#dirty-pages-and-writeback}

### dirtyページのライフサイクル {#dirty-page-lifecycle}

```
Clean page ──► write() ──► Dirty page ──► writeback ──► Clean page
                               │
                               ▼
                     Tracked in dirty lists
```

### ライトバックのトリガ {#writeback-triggers}

| トリガ | タイミング |
|---------|------|
| 定期 | `dirty_writeback_centisecs`ごと（デフォルト: 5秒） |
| メモリ逼迫 | 回収がページを解放する必要があるとき |
| sync/fsync | 明示的なフラッシュ要求 |
| dirtyしきい値 | システム全体でdirtyページが多すぎるとき |
| dirtyの経過時間 | ページが`dirty_expire_centisecs`より長くdirtyのままのとき |

### dirtyページの上限 {#dirty-page-limits}

デフォルト値は[`mm/page-writeback.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page-writeback.c)で定義され、チューニング可能なパラメータは[vm sysctlのドキュメント](https://docs.kernel.org/admin-guide/sysctl/vm.html)に記載されている。

```bash
# dirtyにできるメモリの割合
cat /proc/sys/vm/dirty_ratio
# 20 (デフォルト) - 20%で書き込み側のブロックを開始

# バックグラウンドライトバックのしきい値
cat /proc/sys/vm/dirty_background_ratio
# 10 (デフォルト) - 10%でバックグラウンドライトバックを開始

# 時間ベースの設定 (センチ秒)
cat /proc/sys/vm/dirty_expire_centisecs
# 3000 (30秒) - これより古いページは書き出される

cat /proc/sys/vm/dirty_writeback_centisecs
# 500 (5秒) - ライトバックスレッドが起床する間隔
```

### ライトバックスレッド {#writeback-threads}

```bash
# デバイスごとのライトバックスレッド
ps aux | grep writeback
# kworker/u8:0+flush-8:0   (デバイス8:0用)
```

## 読み取りの戦略 {#reading-strategies}

### 先読み（readahead） {#readahead}

カーネルはシーケンシャルなアクセスを予測して先読みする。

```
Application reads page 0
        │
        v
Kernel detects sequential pattern
        │
        v
Prefetch pages 1, 2, 3, 4... into cache
        │
        v
When application reads page 1, it's already cached
```

```bash
# デフォルトの先読み量 (KB)
cat /sys/block/sda/queue/read_ahead_kb
# 128 (デフォルト)

# ワークロードに合わせて調整
echo 256 > /sys/block/sda/queue/read_ahead_kb
```

### mmapとread()の比較 {#mmap-vs-read}

| 方式 | 仕組み | 適した用途 |
|--------|-----------|----------|
| `read()` | キャッシュからユーザーバッファへコピー | シーケンシャル、単純な用途 |
| `mmap()` | キャッシュのページをプロセスのアドレス空間にマップ | ランダムアクセス、大きなファイル |

`mmap()`では、ページキャッシュのページをプロセスのアドレス空間に直接マップできる。

```c
void *ptr = mmap(NULL, size, PROT_READ, MAP_SHARED, fd, 0);
/* ptrはページキャッシュのページを直接参照する - 真のゼロコピー */

/* MAP_PRIVATEでは、書き込みでCOW (コピーオンライト) が起きる */
void *ptr = mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_PRIVATE, fd, 0);
/* 読み取りはゼロコピーだが、書き込みではプライベートなコピーが作られる */
```

## メモリ逼迫 {#memory-pressure}

### 「空き」メモリとしてのページキャッシュ {#page-cache-as-free-memory}

Linuxは「空いている」RAMをページキャッシュに使う。アプリケーションがメモリを必要とすると、次のようになる。

1. cleanなキャッシュページは即座に破棄される
2. dirtyなキャッシュページは書き出されてから破棄される
3. アプリケーションがメモリを得る

```bash
# メモリの内訳を見る
cat /proc/meminfo | grep -E "MemFree|Cached|Buffers|Available"
# MemFree:     1234567 kB   (本当に空いている)
# Cached:      8765432 kB   (ページキャッシュ)
# MemAvailable: 9876543 kB  (空き + 回収可能なキャッシュ)
```

### キャッシュの破棄（テスト専用） {#drop-caches-testing-only}

```bash
# cleanなキャッシュを破棄 (ベンチマーク用)
sync  # 先にdirtyページを書き出す
echo 3 > /proc/sys/vm/drop_caches
# 1 = ページキャッシュ
# 2 = dentry/inode
# 3 = 両方
```

## 監視 {#monitoring}

### システム全体のキャッシュ統計 {#system-wide-cache-statistics}

```bash
# ディスクI/Oカウンタ (直接のキャッシュヒット/ミスではない - 先読みやダイレクトI/Oを含む)
cat /proc/vmstat | grep -E "pgpgin|pgpgout"
# pgpgin   - ディスクから読み込んだページ (KB)
# pgpgout  - ディスクへ書き出したページ (KB)

# ページキャッシュのサイズと状態
cat /proc/meminfo | grep -E "Cached|Buffers|Dirty|Writeback"
# Cached:      ページキャッシュのサイズ (メモリ上のファイルバックページ)
# Buffers:     ブロックデバイスのメタデータキャッシュ
# Dirty:       変更され、書き出しを待っているページ
# Writeback:   現在ディスクへ書き出し中のページ
```

注: Linuxは直接的な「キャッシュヒット率」のカウンタを公開していない。pgpginが多いのにCachedがあまり増えない場合は、キャッシュが逼迫していることを示唆する。

### ファイルごとのキャッシュ状態 {#per-file-cache-status}

```bash
# ファイルがキャッシュされているか確認 (vmtouchまたはfincoreを使用)
vmtouch -v /path/to/file
# Files: 1
# Pages: 1000 (cached: 800)

# あるいはfincore (coreutils) で
fincore /path/to/file
```

### トレーシング {#tracing}

```bash
# ページキャッシュのイベントをトレース
echo 1 > /sys/kernel/debug/tracing/events/filemap/mm_filemap_add_to_page_cache/enable
echo 1 > /sys/kernel/debug/tracing/events/writeback/writeback_dirty_page/enable
cat /sys/kernel/debug/tracing/trace_pipe
```

## 変遷 {#evolution}

### 起源（1990年代） {#origins-1990s}

ページキャッシュは初期のLinuxから存在する。当初は（ブロックデバイス用の）「バッファキャッシュ」とは別物だった。

### 統合ページキャッシュ（v2.4、2001年） {#unified-page-cache-v24-2001}

バッファキャッシュがページキャッシュに統合された。すべてのファイルI/Oが1つのキャッシュを通るようになった。

### Radix Tree（v2.6） {#radix-tree-v26}

ファイルオフセットで高速に検索できるよう、ページがradix treeでインデックス付けされるようになった。

### XArray（v4.20、2018年） {#xarray-v420-2018}

**コミット**: [a28334862993](https://git.kernel.org/linus/a28334862993) ("page cache: Finish XArray conversion") | [LKML](https://lore.kernel.org/lkml/20180611140639.17215-1-willy@infradead.org/)

**作者**: Matthew Wilcox

radix treeがXArrayに置き換えられた。APIはより整理され、性能は同等である。

### folio（v5.16、2022年） {#folios-v516-2022}

**コミット**: [7b230db3b8d3](https://git.kernel.org/linus/7b230db3b8d3) ("mm: Introduce struct folio") | [LKML](https://lore.kernel.org/linux-mm/20210430180740.2707166-1-willy@infradead.org/)

**作者**: Matthew Wilcox

folioは1つ以上の連続したページを単一の単位として表す。この抽象化により、ページキャッシュ内のhuge pageのオーバーヘッドが減り、関数がヘッドページを扱うのか任意のページを扱うのかという混乱がなくなる。

## ダイレクトI/O（キャッシュのバイパス） {#direct-io-bypassing-cache}

アプリケーションによってはページキャッシュを完全にバイパスする。

```c
int fd = open("file", O_RDWR | O_DIRECT);
/* 読み書きはページキャッシュをバイパスする */
/* アラインされたバッファとサイズが必要 */
```

**用途**:

- 独自のキャッシュを持つデータベース
- 二重キャッシュの回避
- 予測可能なレイテンシ

**トレードオフ**:

- 先読みの恩恵がない
- 書き込みのバッファリングがない
- アプリケーションがキャッシュを扱う必要がある

## よくある問題 {#common-issues}

### キャッシュのスラッシング {#cache-thrashing}

ワーキングセットが利用可能なRAMより大きい。

**症状**: `pgpgin`/`pgpgout`が多い、I/Oが遅い

**対策**:

- RAMを増やす
- ワーキングセットを減らす
- 一部のワークロードでO_DIRECTを使う

### dirtyページの蓄積 {#dirty-page-buildup}

dirtyページが多すぎて書き込みのストールが起きる。

**症状**: プロセスが`balance_dirty_pages`でブロックされる

**対策**:

- `dirty_ratio`を下げる
- より高速なストレージ
- 書き込みレートを下げる

### 先読みのミスマッチ {#readahead-mismatch}

シーケンシャルな先読みがランダムアクセスのワークロードに悪影響を与える。

**対策**:

- `posix_fadvise(POSIX_FADV_RANDOM)`を使う
- `read_ahead_kb`を減らす
- mmapには`madvise(MADV_RANDOM)`を使う

## リファレンス {#references}

### 主要なコード {#key-code}

| ファイル | 説明 |
|------|-------------|
| [`mm/filemap.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/filemap.c) | ページキャッシュのコア |
| [`mm/readahead.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/readahead.c) | 先読みのロジック |
| [`mm/page-writeback.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page-writeback.c) | dirtyページの処理 |

### 関連 {#related}

- [reclaim](reclaim.md) - キャッシュページがどのように回収されるか
- [mmap](mmap.md) - メモリマップトファイルI/O
- [swap](swap.md) - 匿名ページがディスクへ行くとき

## 参考文献 {#further-reading}

- [mm/filemap.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/filemap.c) — ページキャッシュのコア：`filemap_read()`、`filemap_fault()`、`add_to_page_cache_lru()`、およびaddress_spaceの操作
- [mm/readahead.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/readahead.c) — 先読みのロジック：シーケンシャル検出、ウィンドウサイズの決定、`page_cache_async_ra()`
- [mm/page-writeback.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page-writeback.c) — dirtyページの計上とライトバックのスロットリング：`balance_dirty_pages()`、dirty ratioの強制
- `Documentation/admin-guide/mm/concepts.rst` — ページキャッシュ、dirtyページ、およびメモリのライフサイクルにおけるその役割を概説するカーネルドキュメント
- [reclaim](reclaim.md) — メモリ逼迫時にカーネルがcleanおよびdirtyなページキャッシュのページをどのように回収するか
- [mmap](mmap.md) — ファイルバックのmmap領域がページキャッシュのページをプロセスのアドレス空間に直接マップする仕組み
- [folio](folio.md) — 大きな連続したファイルエクステントをよりよくサポートするため、ページキャッシュで生のページに取って代わったfolio抽象化
- [The page cache](https://lwn.net/Articles/712467/) (LWN, 2016) — ページキャッシュの内部、address_space構造体、ライトバックの仕組みの詳しい解説
- [Folios](https://lwn.net/Articles/849538/) (LWN, 2021) — 現代のページキャッシュAPIの基盤となるfolio抽象化の動機と設計
