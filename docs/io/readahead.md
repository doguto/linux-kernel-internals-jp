# 先読み（readahead） {#readahead}

> プロセスが要求する前にpageをページキャッシュへプリフェッチする

## 先読みとは何か {#what-is-readahead}

プロセスがファイルをシーケンシャルに読むとき、ページキャッシュにミスした`read()`はそれぞれブロックI/Oの待ちを伴う。先読みは、どのpageがまもなく必要になるかを予測し、プロセスが要求する**前に**それらのI/Oを発行することで、このレイテンシをなくす — 理想的には、プロセスがそこに到達する時点でpageがすでにページキャッシュに温まっている状態にする。

カーネルは先読みをページキャッシュ層（`mm/readahead.c`）の内部だけで実装している。アプリケーションからは透過的であり、シーケンシャルな`read()`は、データがすでに取得されているために単に速く戻るだけである。

主な性質:

- **バッファードI/O**にのみ適用される。`O_DIRECT`はページキャッシュをバイパスするため、先読みも完全にバイパスする。
- カーネルは、オープンファイル記述ごとに**先読みウィンドウ** — プリフェッチするpageの連続した範囲 — を追跡する。
- ウィンドウはシーケンシャルなアクセスが確認されると拡大し、ランダムアクセスが検出されると縮小する（または無効になる）。

## `struct file_ra_state` {#struct-filerastate}

すべての`struct file`は、そのオープンファイル記述の先読みウィンドウを記録する`file_ra_state`を持っている。

```c
/* include/linux/fs.h */
struct file_ra_state {
    pgoff_t start;          /* 現在のウィンドウの開始位置（page インデックス） */
    unsigned int size;      /* 現在のウィンドウ内の page 数 */
    unsigned int async_size;/* 非同期プリフェッチが発火するまでにウィンドウ内に残る page 数 */
    unsigned int ra_pages;  /* page 単位の最大先読み量（0 = 無効） */
    unsigned int mmap_miss; /* mmap アクセスのキャッシュミス統計 */
    loff_t prev_pos;        /* シーケンシャル検出のための、前回の読み出し位置 */
};
```

| フィールド | 意味 |
|-------|---------|
| `start` | 現在の先読みウィンドウの最初のpageのインデックス |
| `size` | 現在のウィンドウ内のpageの総数（`start` .. `start + size - 1`） |
| `async_size` | プロセスが`start + size - async_size`に到達すると、*次の*ウィンドウの非同期プリフェッチが起動される |
| `ra_pages` | ウィンドウサイズの上限。`0`で先読みが無効になる |
| `mmap_miss` | シーケンシャルなmmapアクセスを検出するため、mmapされた領域でのページフォールトのミスを数える |
| `prev_pos` | 前回の読み出しのバイトオフセット。現在の読み出しがシーケンシャルかどうかの検出に使われる |

`ra_pages`は、バッキングデバイスの`queue/read_ahead_kb`設定（をページサイズで割った値）で初期化される。`POSIX_FADV_SEQUENTIAL`はこれを2倍にし、`POSIX_FADV_RANDOM`はゼロにする。

## 同期先読み {#synchronous-readahead}

同期先読みは`read()`の**クリティカルパス上で**行われる。VFSが`read()`呼び出しを処理するとき、要求されたpageを読む前に（または読みながら）先読みの仕組みを呼び出す。その結果、プロセスが要求したpageを待ってブロックする前に、要求されたpageより先のpageのI/Oが発行される。

### 呼び出し経路 {#call-path}

```
read()
  → vfs_read()
    → generic_file_read_iter()
      → filemap_read()
        → page_cache_sync_ra()   ← first read or after a miss
          → ondemand_readahead()
            → do_page_cache_ra()
              → page_cache_ra_unbounded()
```

### `page_cache_ra_unbounded()` {#pagecachera_unbounded}

`page_cache_ra_unbounded()`は低レベルの主力関数である。この関数は次のことを行う:

1. ウィンドウ`[ractl->_index, ractl->_index + nr_to_read)`内のpageを順に処理する。
2. まだキャッシュにない各pageについて、pageを割り当てて`PG_locked`をマークする。
3. その範囲をカバーする、マージされた1つの`readahead` bioを発行する。
4. ちょうど1つのpage — 先読み領域（lookahead zone）の最初のpageで、インデックスは`start + size - async_size` — に`PG_readahead`をマークする。この1つのpageがトリガーとして働く。`filemap_read`がこのpageをすでにuptodateの状態で見つけると、次のウィンドウの非同期先読みが発火する。

```c
/* mm/readahead.c */
void page_cache_ra_unbounded(struct readahead_control *ractl,
                             unsigned long nr_to_read,
                             unsigned long lookahead_size)
{
    struct address_space *mapping = ractl->mapping;
    unsigned long index = readahead_index(ractl);

    /*
     * page を割り当ててページキャッシュに追加し、I/O を発行する。
     * 末尾の 'lookahead_size' 個の page には PG_readahead が設定され、
     * プロセスがそこに到達すると非同期先読みが発火する。
     */
    for (i = 0; i < nr_to_read; i++) {
        struct folio *folio = xa_load(&mapping->i_pages, index + i);
        if (folio)
            continue;           /* すでにキャッシュ済み */
        folio = filemap_alloc_folio(gfp, 0);
        filemap_add_folio(mapping, folio, index + i, gfp);
        ractl->_nr_pages++;
    }
    read_pages(ractl);          /* bio を発行 */
}
```

### `ondemand_readahead()` {#ondemand_readahead}

`ondemand_readahead()`は`static`な内部関数（公開APIではない）であり、`do_page_cache_ra()`の上位に位置して、**何pageを**読むかを決める。次節で説明するウィンドウ拡大のヒューリスティックとシーケンシャル検出のロジックを実装している。

> **注:** Linux 6.6以降では、大きなfolioの先読みに関するリファクタリングの一環として、`ondemand_readahead()`は`page_cache_ra_order()`に改名された。

## 非同期先読み {#async-readahead}

非同期先読みは、プロセスが現在のウィンドウをまだ消費している間に、**次の**ウィンドウをバックグラウンドでプリフェッチする。そのため、ウィンドウの境界で停滞が起きない。

### 先読みマーカーpage {#the-readahead-marker-page}

`page_cache_ra_unbounded()`がウィンドウを発行するとき、ちょうど1つのpage — 先読み領域の最初のpageで、インデックスは`start + size - async_size` — に`PG_readahead`フラグを設定する。`filemap_read()`がこのpageをすでにuptodateの状態で見つけると、現在の読み出しの完了を待たずに`page_cache_async_ra()`を呼び出す。これは次のウィンドウのために再び`ondemand_readahead()`を呼び出す。

```
Window N already in cache (or being read):
┌────────────────────────────────────────────┐
│ page page page page page [RA] page page    │
└────────────────────────────────────────────┘
                               ↑
               process reaches here (first page of lookahead zone,
               index start+size-async_size) → triggers window N+1 fetch
               (async, does not block the read())
```

つまり、プロセスがウィンドウNの最後のpageを読む時点で、ウィンドウN+1はすでに転送中（または完了済み）である。適切な`async_size`であれば、プロセスが停滞することはない。

```c
/* mm/readahead.c — PG_readahead の page にヒットしたときに呼ばれる */
void page_cache_async_ra(struct readahead_control *ractl,
                         struct folio *folio,
                         unsigned long req_count)
{
    /* folio がまだ uptodate でない場合や、readahead フラグが
     * もう設定されていない場合は非同期先読みを開始しない。 */
    if (!folio_test_readahead(folio))
        return;
    ondemand_readahead(ractl, folio, req_count);
}
```

## 適応的な先読み {#adaptive-readahead}

カーネルは固定のウィンドウサイズを使わない。`ondemand_readahead()`は、観測されたアクセスパターンに基づいて`ra->size`を拡大・縮小する。

### シーケンシャル検出 {#sequential-detection}

次のいずれかを満たす場合、読み出しはシーケンシャルとみなされる:

- 現在の読み出しオフセットが`ra->start + ra->size`に等しい（前回のウィンドウが終わったところからちょうど続いている）、**または**
- `prev_pos`が現在のオフセットから1page以内にある（プロセスが1回の読み出し分だけ前進した）。

どちらの条件も満たさない場合、そのアクセスは**ランダム**として扱われ、ウィンドウはゼロではなく小さな初期サイズ（通常2page）にリセットされる — これにより、カーネルはシーケンシャルアクセスが再開するかどうかを探る。

### ウィンドウの拡大 {#window-growth}

シーケンシャルであることが確認されるたびに、ウィンドウは`ra->ra_pages`に達するまでおおよそ2倍になる:

```c
/* mm/readahead.c の ondemand_readahead() を簡略化したもの */
if (sequential) {
    ra->size = min(ra->size * 2, ra->ra_pages);
} else {
    /* ランダムに見えるアクセス: 初期の探索サイズにリセット */
    ra->size  = get_init_ra_size(ra->ra_pages, ra->ra_pages);
    ra->start = offset;
}
ra->async_size = ra->size / 2;   /* 半分まで進んだところで非同期プリフェッチを発火 */
```

`async_size`はウィンドウのおよそ半分に設定される。これにより非同期プリフェッチが十分早く発火し、プロセスがウィンドウの末尾に到達する前にI/Oが完了する。

### ステートマシンのまとめ {#state-machine-summary}

```
First read on file
  → small initial window (2–4 pages)
  → submit sync readahead

Each subsequent sequential read
  → window doubles (capped at ra_pages)
  → submit async readahead when marker page hit

Random read detected
  → window reset to initial size
  → no async prefetch until sequential pattern re-established

POSIX_FADV_RANDOM or ra_pages == 0
  → readahead fully disabled
```

## `posix_fadvise()`のヒント {#posix_fadvise-hints}

`posix_fadvise()`を使うと、アプリケーションは意図するアクセスパターンをカーネルに伝え、読み出しAPIを変えずに先読みの挙動を調整できる。

```c
#include <fcntl.h>

int posix_fadvise(int fd, off_t offset, off_t len, int advice);
```

| アドバイス | 先読みへの影響 |
|--------|---------------------|
| `POSIX_FADV_NORMAL` | デフォルトの`ra_pages`に戻す |
| `POSIX_FADV_SEQUENTIAL` | BDIのデフォルトに対して`ra_pages`を2倍にする — 2回呼んでも4倍にはならない |
| `POSIX_FADV_RANDOM` | `ra_pages = 0`に設定する — 先読みを完全に無効化する |
| `POSIX_FADV_WILLNEED` | `[offset, offset+len)`に対して即座に同期先読みを起動する |
| `POSIX_FADV_DONTNEED` | 範囲内のpageをページキャッシュから破棄する |
| `POSIX_FADV_NOREUSE` | pageが再利用されないことを示すヒント（助言的なもので、カーネルのサポートは限定的） |

### `POSIX_FADV_SEQUENTIAL` {#posixfadvsequential}

BDIのデフォルトに対して`ra->ra_pages`を2倍にする — 2回呼んでもウィンドウは4倍にはならない。ファイルを最初から最後までストリーミングし、I/Oレイテンシを隠すためにより大きなプリフェッチウィンドウを望むアプリケーションに有用である。

### `POSIX_FADV_RANDOM` {#posixfadvrandom}

`ra->ra_pages = 0`に設定する。カーネルはこのファイルディスクリプタについて先読みのロジックをすべてスキップする。キャッシュにミスしたページフォールトや`read()`は、要求されたpageだけを取得する — それ以上は取得しない。独自のI/Oスケジューリングを行うデータベースや、プリフェッチしたpageが使われる前に追い出されてしまうような真にランダムなアクセスパターンのワークロードに適している。

### `POSIX_FADV_WILLNEED` {#posixfadvwillneed}

現在のアクセスパターンにかかわらず、指定した範囲の先読みを即座にスケジュールする。この呼び出しはI/Oの完了前に戻る（ノンブロッキングである）。次の用途に有用である:

- シーケンシャルスキャンを開始する前にファイルを事前に読み込む
- レイテンシに敏感な操作の前に、アイドル期間中にキャッシュを温めておく

### `POSIX_FADV_DONTNEED` {#posixfadvdontneed}

範囲内のクリーンなpageをページキャッシュから破棄する。ファイルを一度だけ読み、キャッシュの汚染を避けたいストリーミングアプリケーションに有用である（`O_DIRECT`に似ているが、アライメントの制約がない）。

### Cの例 {#c-example}

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    int fd = open("large_file.dat", O_RDONLY);
    if (fd < 0) { perror("open"); return 1; }

    off_t file_size = lseek(fd, 0, SEEK_END);

    /*
     * シーケンシャルに読むことをカーネルに伝える。
     * これにより ra_pages が 2 倍になり、プリフェッチウィンドウがより速く拡大する。
     */
    posix_fadvise(fd, 0, file_size, POSIX_FADV_SEQUENTIAL);

    /*
     * 任意: 処理ループを始める前に、
     * 先頭 64 MiB を今のうちにキャッシュへ温めておく。
     */
    posix_fadvise(fd, 0, 64 * 1024 * 1024, POSIX_FADV_WILLNEED);

    char *buf = malloc(1024 * 1024);
    ssize_t n;
    lseek(fd, 0, SEEK_SET);
    while ((n = read(fd, buf, 1024 * 1024)) > 0) {
        /* buf[0..n-1] を処理する */

        /*
         * より有用なデータを追い出さないよう、
         * 処理済みの page をキャッシュから破棄する。
         */
        off_t pos = lseek(fd, 0, SEEK_CUR);
        posix_fadvise(fd, pos - n, n, POSIX_FADV_DONTNEED);
    }

    free(buf);
    close(fd);
    return 0;
}
```

## mmap向けの`madvise()`による同等機能 {#madvise-equivalents-for-mmap}

メモリマップされたファイルには`posix_fadvise()`は適用されない。代わりに`madvise()`を使う:

```c
#include <sys/mman.h>

void *addr = mmap(NULL, length, PROT_READ, MAP_SHARED, fd, 0);

/* POSIX_FADV_SEQUENTIAL に相当: */
madvise(addr, length, MADV_SEQUENTIAL);

/* POSIX_FADV_RANDOM に相当: */
madvise(addr, length, MADV_RANDOM);

/* POSIX_FADV_WILLNEED に相当: */
madvise(addr, length, MADV_WILLNEED);

/* page を破棄（POSIX_FADV_DONTNEED に相当）: */
madvise(addr, length, MADV_DONTNEED);
```

| `madvise`のフラグ | 効果 |
|---------------|--------|
| `MADV_SEQUENTIAL` | 先読みを2倍にし、フォールト位置より後ろのpageを積極的に追い出す |
| `MADV_RANDOM` | このマッピングの先読みを無効化する（範囲内のフォールトに対して`ra_pages = 0`） |
| `MADV_WILLNEED` | 範囲に対する非同期先読みを起動する — プロセスがアクセスする前にpageがフォールトインされる |
| `MADV_DONTNEED` | pageをキャッシュから破棄する。次のアクセスで再びフォールトする |

`MADV_SEQUENTIAL`はさらに、カーネルのページフォールトハンドラに対して、現在のフォールト位置より**後ろ**のpageをより積極的に破棄するよう指示し、ストリーミングワークロードのメモリ圧迫を軽減する。

## 実践的なチューニング {#practical-tuning}

### `/sys/block/<dev>/queue/read_ahead_kb` {#sysblockqueuereadaheadkb}

デバイスごとの先読みサイズは、そのデバイス上のすべてのファイルのデフォルトの`ra_pages`を決める:

```bash
# 現在の設定を読む（キロバイト単位）:
cat /sys/block/sda/queue/read_ahead_kb
# 128

# シーケンシャルなワークロード（大きな動画ファイル、バックアップなど）では増やす:
echo 2048 > /sys/block/sda/queue/read_ahead_kb

# ランダムアクセスのワークロード（データベースのデータファイルなど）では減らす:
echo 16 > /sys/block/sda/queue/read_ahead_kb
```

`POSIX_FADV_SEQUENTIAL`はこのfdごとの値を2倍にし、`POSIX_FADV_RANDOM`はゼロにする。sysfsのつまみはグローバルな基準値である。

### データベースワークロードへの影響 {#effect-on-database-workloads}

ほとんどのデータベース（PostgreSQL、MySQL InnoDB、RocksDB）は、データファイルを`O_DIRECT`か明示的な`POSIX_FADV_RANDOM`のどちらかで開く:

- **`O_DIRECT`** — ページキャッシュと先読みを完全にバイパスする。データベースは独自のバッファプールを管理する。
- **`POSIX_FADV_RANDOM`** — バッファードI/Oを維持しつつカーネルの先読みを無効にする。データベースは、シーケンシャルスキャンが来ると分かっているときに`POSIX_FADV_WILLNEED`で独自にプリフェッチを発行する。

**fadviseのヒントなしでバッファードI/O**を使うデータベースでは、大きな`read_ahead_kb`はメモリとI/O帯域幅を無駄にする。データベースが順番にはアクセスしないpageをカーネルがプリフェッチしてしまうからである。データベースのブロックデバイスで`read_ahead_kb`を16〜64 KBに減らすのは、本番環境でよく行われるチューニングである。

逆に、**シーケンシャルなワークロード** — ログシッピング、バックアップストリーム、分析系のフルテーブルスキャン — では、大きな先読みウィンドウ（512 KB〜4 MB）がI/Oパイプラインを満たし続け、停滞をなくす。

### NVMeでの考慮事項 {#nvme-considerations}

NVMeではI/Oレイテンシが十分に低い（50〜200 µs）ため、中程度のシーケンシャル読み出しでは先読みの重要性は低い。しかし、NVMeのキューは深い（1024以上のコマンド）ため、大きな先読みウィンドウを早めに発行することは、依然として帯域幅を飽和させるのに役立つ。NVMeデバイスのデフォルトの`read_ahead_kb`は通常128 KBであり、512 KB以上にチューニングすると、大量のシーケンシャルワークロードでスループットが向上することがある。

## 参考文献 {#further-reading}

- [バッファードI/O](buffered-io.md) — ページキャッシュ、`filemap_read()`、ライトバック
- [Direct I/O](direct-io.md) — ページキャッシュと先読みのバイパス
- `mm/readahead.c` — `ondemand_readahead()`、`page_cache_ra_unbounded()`、`page_cache_async_ra()`
