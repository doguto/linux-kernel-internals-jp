# バッファードI/O {#buffered-io}

> `read()`と`write()`がVFS、ページキャッシュを通って流れ、戻ってくるまで

## バッファードI/Oとは何か {#what-buffered-io-means}

デフォルトでは、通常ファイルに対するすべての`read()`と`write()`は **ページキャッシュ** を経由する。ページキャッシュとはカーネルが管理するDRAM上の領域で、ファイルデータを4 KBのpage（あるいはより大きなfolio。`struct folio`はLinux 5.16で導入されたが、ファイルページキャッシュでのラージfolioにはLinux 6.1以降が必要）としてキャッシュする。カーネルは読み出しをこのキャッシュ済みpageから満たし、書き込みはダーティページに蓄積してから非同期にストレージへフラッシュする。

```
read()  → VFS → page cache hit?  yes → copy to userspace
                                  no  → fetch from disk → cache → copy to userspace

write() → VFS → page cache → mark dirty → return to userspace
                                 ↓  (later, by writeback)
                              storage
```

これが、データがディスクに届く前に`write()`が戻る理由であり、同じファイルを繰り返し読むと速い理由でもある。

## カーネルへのシステムコール経路 {#the-syscall-path-into-the-kernel}

ユーザー空間から始まり、通常ファイルに対する`read()`や`write()`は次の連鎖をたどる。

```
read(fd, buf, count)
  → sys_read()                              [fs/read_write.c]
  → ksys_read()
  → vfs_read()
  → file->f_op->read_iter()                 [set at open time]
      → generic_file_read_iter()            [mm/filemap.c]   (most filesystems)

write(fd, buf, count)
  → sys_write()
  → ksys_write()
  → vfs_write()
  → file->f_op->write_iter()
      → generic_file_write_iter()           [mm/filemap.c]
```

`generic_file_read_iter`と`generic_file_write_iter`は、ext4、xfs、btrfsをはじめとする大半のファイルシステムで使われる汎用実装である。これらは`address_space`抽象を介して、完全にページキャッシュに対して動作する。

`kiocb`構造体は、この連鎖を通じて操作ごとの状態を運ぶ。

```c
/* include/linux/fs.h */
struct kiocb {
    struct file       *ki_filp;    /* 操作対象のファイル */
    loff_t             ki_pos;     /* 現在のファイルオフセット */
    void (*ki_complete)(struct kiocb *, long);  /* 非同期完了コールバック */
    void              *private;
    int                ki_flags;   /* IOCB_* フラグ（O_DIRECT、O_SYNC など） */
    u16                ki_ioprio;
    struct wait_page_queue *ki_waitq;  /* 非同期の page 待ち用 */
};
```

## ページキャッシュの構造 {#page-cache-structure}

ファイルデータを保持するすべてのinodeは、キャッシュ済みpageを所有する`address_space`を埋め込んでいる。

```c
/* include/linux/fs.h */
struct address_space {
    struct inode            *host;          /* 所有者の inode */
    struct xarray            i_pages;       /* キャッシュ済みの全 folio。page インデックスで索引付け */
    struct rw_semaphore      invalidate_lock;
    gfp_t                    gfp_mask;      /* 新しい page 用の割り当てフラグ */
    atomic_t                 i_mmap_writable;
    struct rb_root_cached    i_mmap;        /* このファイルをマップする全 VMA */
    unsigned long            nrpages;       /* キャッシュされたベース page 単位の総数（folio ごとに 1 ではなく folio_nr_pages()） */
    unsigned long            writeback_index; /* 次にライトバックする page */
    const struct address_space_operations *a_ops;  /* read_folio、dirty_folio、writepages など */
    unsigned long            flags;
    errseq_t                 wb_err;        /* 保持され続けるライトバックエラー */
    spinlock_t               private_lock;
    struct list_head         private_list;
    void                    *private_data;
};
```

Linux 5.16より前は、キャッシュは`struct page *`の`xarray`だった。5.16以降、単位は **folio**、すなわち`struct folio`で表される物理的に連続したpageのグループである。folioは単一のベースpage（order 0）の場合もあれば、ラージfolio（order N、2^N個のpageをカバー）の場合もある。ファイルページキャッシュでのラージfolioにはLinux 6.1以降が必要だった。`i_pages`へのインデックスは、folioの先頭pageのpageインデックスである。

```c
/* include/linux/mm_types.h（簡略化） */
struct folio {
    /* flags の下位ビットが folio の状態をエンコードする */
    unsigned long flags;
    union {
        struct {
            /* ファイルバックの folio の場合: */
            struct address_space *mapping;
            pgoff_t               index;     /* ファイル内オフセット >> PAGE_SHIFT */
        };
    };
    atomic_t      _refcount;
    atomic_t      _mapcount;
    /* … この後に実際の page データが続く … */
};
```

`a_ops`ポインタは、汎用のページキャッシュコードとファイルシステム固有のロジックを結びつける重要なディスパッチテーブルである。

```c
/* include/linux/fs.h */
struct address_space_operations {
    int  (*read_folio)(struct file *, struct folio *);  /* 以前の readpage */
    int  (*writepage)(struct page *, struct writeback_control *);  /* writepage はカーネルから削除されつつある。writepages を優先すること */
    bool (*dirty_folio)(struct address_space *, struct folio *);
    int  (*write_begin)(struct file *, struct address_space *, loff_t,
                         unsigned len, struct folio **, void **);
    int  (*write_end)(struct file *, struct address_space *, loff_t,
                       unsigned len, unsigned copied, struct folio *, void *);
    int  (*writepages)(struct address_space *, struct writeback_control *);
    /* … */
};
/* write_begin/write_end のシグネチャは Linux 6.12 以降 struct folio を使う */
```

## 読み出し経路 {#read-path}

`generic_file_read_iter`は、バッファードI/Oについては`filemap_read`に処理を委ねる。

```c
/* mm/filemap.c */
ssize_t filemap_read(struct kiocb *iocb, struct iov_iter *iter,
                      ssize_t already_read)
{
    struct file           *filp  = iocb->ki_filp;
    struct address_space  *mapping = filp->f_mapping;
    struct inode          *inode = mapping->host;
    struct folio_batch     fbatch;
    pgoff_t index = iocb->ki_pos >> PAGE_SHIFT;
    loff_t  end;
    int     error = 0;

    do {
        /* xarray 内で [index, last] をカバーする folio を探す */
        nr = filemap_get_folios(mapping, &index, last_index, &fbatch);

        for (i = 0; i < nr; i++) {
            struct folio *folio = fbatch.folios[i];

            /*
             * folio が最新になるまで待つ。これは filemap_read で実際に
             * 使われている待ちパターンである: PG_locked がクリアされるのを
             * 待ち、その後 PG_uptodate を明示的に確認する。
             */
            folio_wait_locked(folio);
            if (!folio_test_uptodate(folio))
                goto io_error;

            /* folio からユーザーの iov_iter へデータをコピーする */
            copied = copy_folio_to_iter(folio, offset, bytes, iter);
        }
    } while (iov_iter_count(iter) && iocb->ki_pos < isize && !error);

out:
    return already_read ? already_read : error;
}
```

### キャッシュヒット {#cache-hit}

要求されたpageインデックスが`i_pages`に存在し、`PG_uptodate`がセットされていれば、`filemap_read`はfolioからユーザーバッファへ直接コピーする。I/Oは発行されず、呼び出しはスリープせずに戻る。

### キャッシュミス {#cache-miss}

`i_pages`の検索で、あるインデックスについて何も返らなかった場合、カーネルはfolioを埋めなければならない。

```
filemap_read
  → filemap_get_folios() returns 0 for a range
  → filemap_create_folio()
      → __filemap_add_folio()       add locked folio to i_pages
      → filemap_read_folio()
          → a_ops->read_folio()     filesystem submits bio
              e.g. ext4_read_folio → mpage_read_folio → submit_bio
  → folio_wait_locked() + folio_test_uptodate()  sleep until PG_uptodate is set
      (woken by end_page_read / folio_end_read after bio completes)
  → copy_folio_to_iter()            now safe to copy
```

I/Oの実行中、folioはロックされたまま（`PG_locked`）保持される。folioがすでに`i_pages`にあるがまだ最新でないことを見つけた他の読み出し側は、重複したI/Oを発行する代わりに、同じfolioの待ちキュー上で`folio_wait_locked`の中でブロックする。

先読み（read-ahead）は並行して動作する。`page_cache_async_ra`と`ondemand_readahead`は、現在の読み出し位置より先のキャッシュを投機的に埋めてレイテンシを隠す。先読みウィンドウは小さく始まり、`ra->ra_pages`に達するまで幾何級数的に拡大する。

## 書き込み経路 {#write-path}

`generic_file_write_iter`はロックと同期セマンティクスを処理し、その後実際の処理のために`generic_perform_write`を呼び出す。

```c
/* mm/filemap.c */
ssize_t generic_perform_write(struct kiocb *iocb, struct iov_iter *i)
{
    struct file          *file    = iocb->ki_filp;
    struct address_space *mapping = file->f_mapping;
    const struct address_space_operations *a_ops = mapping->a_ops;
    loff_t pos = iocb->ki_pos;
    ssize_t written = 0;

    do {
        struct page *page;
        void        *fsdata;
        unsigned long offset = pos & (PAGE_SIZE - 1);
        unsigned long bytes  = min_t(unsigned long,
                                     PAGE_SIZE - offset,
                                     iov_iter_count(i));

        /*
         * 1. 対象 page の準備をファイルシステムに依頼する。
         *    ブロックマップ型のファイルでは、page を見つけるか割り当て、
         *    必要ならディスクから部分 page のデータを読み込む（read-modify-write）。
         */
        status = a_ops->write_begin(file, mapping, pos, bytes,
                                     &page, &fsdata);
        if (unlikely(status < 0))
            break;

        /*
         * 2. ユーザーバッファから page へコピーする。
         */
        copied = copy_page_from_iter_atomic(page, offset, bytes, i);

        /*
         * 3. コピーが終わったことをファイルシステムに伝える。ファイルシステムは
         *    page をダーティにしてアンロックする。
         */
        status = a_ops->write_end(file, mapping, pos, bytes, copied,
                                   page, fsdata);
        if (unlikely(status != copied)) { /* ショートライトまたはエラー */ }

        pos     += copied;
        written += copied;

        balance_dirty_pages_ratelimited(mapping); /* ダーティが多すぎればスロットリングする */
    } while (iov_iter_count(i));

    return written;
}
```

`write_begin`と`write_end`は、各ファイルシステムがコピーの前後でブロック割り当てとジャーナリングを管理できるようにする2つのフックである。`data=ordered`モードのext4では、`write_begin`がジャーナルクレジットを確保し、`write_end`が`ext4_journal_stop()`を呼んでジャーナルハンドルを解放する。これはコミットを引き起こさない。トランザクションは後で、fsync時またはトランザクションが満杯になったときにコミットされる。データ順序の制約（メタデータがコミットされる前にデータが書かれること）は、`write_end`ではなくライトバック時に強制される。

## ダーティページのライフサイクル {#dirty-page-lifecycle}

`write_end`がpageをダーティにすると、カーネルはライトバックのためにそれを追跡する。folioの状態遷移は次のとおりである。

```
[not in cache]
      │  filemap_create_folio / filemap_fault
      ▼
[clean, uptodate]       ← disk data is authoritative
      │  copy_page_from_iter + SetPageDirty / folio_mark_dirty
      ▼
[dirty]                 ← page has newer data than disk
      │  wakeup_flusher_threads / writeback_inodes_wb
      ▼
[writeback]             ← bio submitted, PG_writeback set
      │  end_page_writeback / folio_end_writeback (bio completion)
      ▼
[clean, uptodate]       ← back to steady state
```

### pageがダーティになる仕組み {#how-a-page-becomes-dirty}

呼び出しの連鎖には2つの異なる層がある。`folio_mark_dirty`は
`a_ops->dirty_folio`を通じてディスパッチする公開APIである。`filemap_dirty_folio`は、大半のファイルシステムが
`a_ops->dirty_folio`に登録するデフォルト実装である。これらはディスパッチ連鎖の中の
別々の2つの関数であり、再帰呼び出しではない。

```c
/*
 * 公開 API（mm/page-writeback.c）:
 * ファイルシステムが a_ops->dirty_folio を登録していればそれを通じてディスパッチし、
 * そうでなければ filemap_dirty_folio にフォールバックする。
 */
bool folio_mark_dirty(struct folio *folio)
{
    struct address_space *mapping = folio->mapping;

    if (mapping && mapping->a_ops->dirty_folio)
        return mapping->a_ops->dirty_folio(mapping, folio);

    return filemap_dirty_folio(mapping, folio);
}

/*
 * a_ops->dirty_folio のデフォルト実装（mm/filemap.c）:
 * 実際の計上を行う。folio をダーティにし、ライトバック統計を更新し、
 * inode 追跡の仕組みに通知する。
 */
bool filemap_dirty_folio(struct address_space *mapping, struct folio *folio)
{
    folio_memcg_lock(folio);
    if (folio_test_set_dirty(folio)) {   /* PG_dirty のアトミックな test-and-set */
        folio_memcg_unlock(folio);
        return false;   /* すでにダーティ */
    }
    folio_memcg_unlock(folio);

    /* ダーティメモリをライトバックデバイスに対して計上する */
    wb_stat_mod(inode_to_bdi(mapping->host)->wb, WB_RECLAIMABLE, 1);

    __mark_inode_dirty(mapping->host, I_DIRTY_PAGES);
    return true;
}
```

カーネルは2つのダーティスロットリング閾値（メモリに対する割合で表される）を強制する。

- `dirty_background_ratio` / `dirty_background_bytes` — バックグラウンドのライトバックが始まる
- `dirty_ratio` / `dirty_bytes` — フォアグラウンドのスロットリング。`balance_dirty_pages_ratelimited`が書き込み中のプロセスをブロックする

`/proc/sys/vm/dirty_writeback_centisecs`は、`wb_workfn`（BDIごとのライトバックワーカー）が起床して`dirty_expire_centisecs`より長くダーティのままのpageをフラッシュする頻度を制御する。

### ライトバック {#writeback}

```c
/* fs/fs-writeback.c */
static long writeback_chunk_size(struct bdi_writeback *wb,
                                  struct wb_writeback_work *work)
{ /* … */ }

static long writeback_sb_inodes(struct super_block *sb,
                                 struct bdi_writeback *wb,
                                 struct wb_writeback_work *work)
{
    while (!list_empty(&wb->b_io)) {
        struct inode *inode = wb_inode(wb->b_io.prev);
        __writeback_single_inode(inode, &wbc);
        /* → do_writepages → a_ops->writepages または generic_writepages
             → write_cache_pages → folio ごとに folio_lock + a_ops->writepage
               → ext4_writepage / iomap_writepage → submit_bio
               → folio_start_writeback(): PG_writeback をセットする */
    }
}
```

bioが完了すると、ブロック層はI/O終了ハンドラを呼び出し、それが`folio_end_writeback`を呼んで`PG_writeback`をクリアし、待機者を起こす。

## バッファヘッド {#buffer-heads}

folioと`iomap`以前、ブロックマップ型のファイルはブロックごとのI/O状態を`struct buffer_head`で追跡していた。

```c
/* include/linux/buffer_head.h */
struct buffer_head {
    unsigned long      b_state;     /* BH_Uptodate、BH_Dirty、BH_Lock など */
    struct buffer_head *b_this_page; /* page の bh の循環リスト */
    union {
        struct page    *b_page;      /* この bh が属する page */
        struct folio   *b_folio;
    };
    sector_t           b_blocknr;   /* ディスク上のブロック番号 */
    size_t             b_size;      /* ブロックサイズ */
    char              *b_data;      /* page 内へのポインタ */
    struct block_device *b_bdev;
    bh_end_io_t       *b_end_io;    /* I/O 完了 */
    void              *b_private;
    /* … */
};
```

ブロックマップ型ファイルの各pageは、ファイルシステムブロックごとに1つずつのバッファヘッドのリングを持っていた。1 KBブロックの4 KB pageであれば、4つの個別セクタを追跡する4つのバッファヘッドになる。これは動作するが高コストである。各バッファヘッドはslabから割り当てられる約60バイトであり、リングにはロックのオーバーヘッドがあり、ブロック単位の粒度はfolioサイズの拡大を複雑にする。

現代の代替は **iomap**（`fs/iomap/`）である。各pageにブロックごとのメタデータを付ける代わりに、iomapはファイルのバイト範囲を物理ブロックの範囲へ1回の呼び出しでマップする。これにより、移行済みのファイルシステム（ext4、xfs、btrfs、f2fs）ではバッファヘッドが不要になる。バッファヘッドは古いコードパス、FAT、一部のブロックデバイスに残っている。

```c
/* include/linux/iomap.h */
struct iomap {
    u64            addr;    /* ディスクオフセット（バイト単位）。ホールなら IOMAP_NULL_ADDR */
    loff_t         offset;  /* ファイルオフセット */
    u64            length;  /* このマッピングの長さ */
    u16            type;    /* IOMAP_HOLE、IOMAP_MAPPED、IOMAP_UNWRITTEN など */
    u16            flags;
    struct block_device *bdev;
    struct dax_device   *dax_dev;
};
```

iomapでは、`a_ops->write_begin`は`iomap_write_begin`によって実装される。これはファイルシステムの`iomap_ops->iomap_begin`を呼んで書き込み範囲をカバーする単一の`struct iomap`を取得し、それを使ってfolioをセットアップする。バッファヘッドは一切関与しない。

## O_SYNCとfsync {#o_sync-and-fsync}

バッファードI/Oでの`write()`は、データがページキャッシュに入った時点で戻る。永続性の保証を得るには次のようにする。

```c
/* O_SYNC: 各 write はデータ+メタデータが安定ストレージに届くまでブロックする */
int fd = open("file", O_WRONLY | O_SYNC);

/* O_DSYNC: 各 write はデータが安定ストレージに届くまでブロックする（メタデータは含まない） */
int fd = open("file", O_WRONLY | O_DSYNC);

/* 手動 fsync: このファイルのダーティページ + メタデータをすべてフラッシュする */
write(fd, buf, len);
fsync(fd);         /* → vfs_fsync → a_ops->writepages + inode ログのコミット */

/* fdatasync: ダーティページのみフラッシュする（サイズが変わらなければメタデータは省略） */
fdatasync(fd);
```

`O_SYNC`がセットされていると、`generic_file_write_iter`は`generic_perform_write`の後に`generic_write_sync`を呼び、それが`vfs_fsync_range`を呼んでライトバックを強制し、ユーザー空間に戻る前にその完了を待つ。

## 主要なソースファイル {#key-source-files}

| ファイル | 役割 |
|------|------|
| `mm/filemap.c` | `filemap_read`、`generic_perform_write`、`filemap_dirty_folio`、ページフォールト時の読み出し |
| `mm/page-writeback.c` | ダーティの計上、`balance_dirty_pages`、スロットリング |
| `fs/fs-writeback.c` | ライトバックワーカー、`writeback_sb_inodes`、`__writeback_single_inode` |
| `include/linux/fs.h` | `struct address_space`、`struct kiocb`、`address_space_operations` |
| `include/linux/pagemap.h` | ページキャッシュ検索ヘルパー、folio状態アクセサ |
| `include/linux/buffer_head.h` | `struct buffer_head`、BH_* 状態ビット |
| `include/linux/iomap.h` | `struct iomap`、iomap操作テーブル |
| `fs/iomap/buffered-io.c` | iomapの`write_begin`/`write_end`、iomap writepage |
| `mm/readahead.c` | `ondemand_readahead`、先読みウィンドウの管理 |

## 参考文献 {#further-reading}

- [非同期I/Oの進化](async-io.md) — POSIX AIOとio_uring。io_uringがバッファードI/Oを非同期に扱う方法を含む
- [ダイレクトI/O](direct-io.md) — O_DIRECTによるページキャッシュのバイパス
- [ページキャッシュのライフサイクル](../mm/page-cache.md) — 追い出し、LRUリスト、メモリ圧迫
- `mm/filemap.c` — バッファード読み書きの中核実装
- `include/linux/fs.h` — `address_space`、`kiocb`、`address_space_operations`の定義
