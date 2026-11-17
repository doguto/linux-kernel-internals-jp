# write()システムコールの一生 {#life-of-a-write-syscall}

> アプリケーションの呼び出しからページキャッシュ内のダーティページまで（そして最終的にはディスクまで）

## 概要 {#overview}

```
Application: write(fd, buf, 4096)
    ↓
Syscall entry → vfs_write()
    ↓
Permission check
    ↓
generic_file_write_iter()
    ↓
Page cache: find or create pages
    ↓
copy_from_user() → dirty page
    ↓
Mark inode dirty, update i_size
    ↓
Return to application
    ↓ (later)
Writeback daemon (kworker/flusher)
    ↓
Filesystem's ->writepages() (ext4, etc.)
    ↓
Block layer → disk
```

## フェーズ1: システムコールへのエントリ {#phase-1-syscall-entry}

`write(2)`システムコールはLinuxのmanページに記載されている[(man page)](https://man7.org/linux/man-pages/man2/write.2.html)。VFSレイヤは、ファイルシステム固有のコードに到達する前に、汎用ヘルパーの連鎖を通じてこれをディスパッチする。

```c
/* arch/x86/entry/syscalls/syscall_64.tbl → sys_write */
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf, size_t, count)
{
    /* 1. fdからstruct fileを取得する */
    struct fd f = fdget_pos(fd);

    /* 2. 現在位置を取得する */
    loff_t pos = file_pos_read(f.file);

    /* 3. VFSへディスパッチする */
    ret = vfs_write(f.file, buf, count, &pos);

    /* 4. ファイル位置を更新する */
    file_pos_write(f.file, pos);
    fdput_pos(f);
    return ret;
}
```

## フェーズ2: VFSのチェック {#phase-2-vfs-checks}

```c
/* fs/read_write.c */
ssize_t vfs_write(struct file *file, const char __user *buf,
                  size_t count, loff_t *pos)
{
    /* ファイルが書き込み用にオープンされているか確認する */
    if (!(file->f_mode & FMODE_WRITE))
        return -EBADF;

    /* セキュリティチェック（SELinux、AppArmorなど） */
    ret = security_file_permission(file, MAY_WRITE);

    /* ファイルシステムのwrite_iterへディスパッチする */
    if (file->f_op->write_iter)
        ret = new_sync_write(file, buf, count, pos);
    else if (file->f_op->write)
        ret = file->f_op->write(file, buf, count, pos);
}
```

## フェーズ3: generic_file_write_iter {#phase-3-generic_file_write_iter}

通常ファイルの場合、ファイルシステムの`write_iter`（例: `ext4_file_write_iter`）は最終的に`generic_file_write_iter()`を呼び出す:

```c
/* mm/filemap.c */
ssize_t generic_file_write_iter(struct kiocb *iocb, struct iov_iter *from)
{
    struct file *file = iocb->ki_filp;
    struct inode *inode = file->f_mapping->host;

    /* inodeを書き込み用にロックする（並行書き込みを防ぐ） */
    inode_lock(inode);

    /* ファイルサイズの上限を確認する */
    ret = generic_write_checks(iocb, from);

    /* ページキャッシュへ書き込む */
    ret = generic_perform_write(iocb, from);

    inode_unlock(inode);
    return ret;
}
```

## フェーズ4: ページキャッシュへの書き込み {#phase-4-writing-to-the-page-cache}

`generic_perform_write()`は必要な各ページについてループする:

```c
ssize_t generic_perform_write(struct kiocb *iocb, struct iov_iter *i)
{
    struct address_space *mapping = iocb->ki_filp->f_mapping;
    loff_t pos = iocb->ki_pos;

    do {
        unsigned long offset = pos & (PAGE_SIZE - 1); /* ページ内のオフセット */
        size_t bytes = min(PAGE_SIZE - offset, iov_iter_count(i));

        /* 1. ページキャッシュ内のページを検索または割り当てる */
        page = a_ops->write_begin(file, mapping, pos, bytes, &page, &fsdata);
        /* 部分的な書き込みの場合、write_beginはディスクからページを読むことがある */

        /* 2. ユーザー空間からページへデータをコピーする */
        copied = copy_page_from_iter_atomic(page, offset, bytes, i);

        /* 3. ページをダーティにして状態を更新する */
        a_ops->write_end(file, mapping, pos, bytes, copied, page, fsdata);
        /* write_endはset_page_dirty()を呼ぶ → ページをライトバック対象にする */

        pos += copied;
    } while (iov_iter_count(i));

    return pos - iocb->ki_pos;
}
```

## フェーズ5: ダーティの追跡 {#phase-5-dirty-tracking}

`set_page_dirty()`の後、複数の構造がダーティとしてマークされる:

```c
/* ページはダーティなfolioとしてaddress_space->i_pagesに追加される */
set_page_dirty(page)
    → folio_mark_dirty(folio)
        → mapping->a_ops->dirty_folio()
        → __set_page_dirty()
            → radix_tree_tag_set(mapping, page_index, PAGECACHE_TAG_DIRTY)

/* inodeはsuperblockのダーティinodeリストに追加される */
__mark_inode_dirty(inode, I_DIRTY_PAGES)
    → list_move(&inode->i_io_list, &wb->b_dirty)
    /* wb = ライトバック制御（bdi = ブロックデバイスごとに1つ） */

/* まだであればライトバックタイマーをセットする */
wb_wakeup_delayed(wb)
```

## フェーズ6: ライトバック {#phase-6-writeback}

ライトバックデーモン（`kworker`）は、定期的にダーティページをディスクへフラッシュする:

```c
/* fs/fs-writeback.c のkworkerスレッドから呼ばれる */
void wb_workfn(struct bdi_writeback *wb)
{
    /* すべてのダーティinodeを探す */
    while (!list_empty(&wb->b_io)) {
        inode = list_first_entry(&wb->b_io, ...);

        /* このinodeのすべてのダーティページを書き出す */
        writeback_single_inode(inode, wbc);
            → mapping->a_ops->writepages(mapping, wbc)
                → ext4_writepages()
                    → mpage_writepages()
                        → for each dirty page:
                            submit_bio(bio)
                            → block layer → disk
    }
}
```

ライトバックは次の契機で起動される:

- **時間**: `dirty_expire_centisecs`（デフォルト3000 = 30秒）の経過後
- **メモリ逼迫**: ダーティページが`dirty_ratio`（デフォルトはRAMの20%）を超えたとき
- **fsync()**: アプリケーションによる明示的なフラッシュ

## ダーティページの状態を確認する {#viewing-dirty-page-state}

```bash
# 現在のダーティメモリ
cat /proc/meminfo | grep -i dirty
# Dirty:          102400 kB   ← 書き込まれたがまだディスクにないデータ
# Writeback:        4096 kB   ← まさに今ディスクへ書き込み中のデータ

# ライトバックのチューニング可能なパラメータ
cat /proc/sys/vm/dirty_ratio          # スロットリング前のダーティの上限（RAMに対する%）
cat /proc/sys/vm/dirty_background_ratio  # バックグラウンドライトバックを開始する%
cat /proc/sys/vm/dirty_expire_centisecs  # ダーティデータが書き出されるまでの経過時間
cat /proc/sys/vm/dirty_writeback_centisecs  # ライトバックを起こす頻度

# デバイスごとのライトバック統計
ls /sys/class/bdi/*/
cat /sys/class/bdi/8:0/read_ahead_kb
```

## fsync()、fdatasync()、sync()の違い {#fsync-vs-fdatasync-vs-sync}

`fsync()`[(man page)](https://man7.org/linux/man-pages/man2/fsync.2.html)は、ファイルのすべてのデータとメタデータが安定したストレージ上にあることを保証する。`fdatasync()`はより軽量な変種で、データと、そのデータを読み戻すのに必要なメタデータ（例: ファイルサイズ）をフラッシュするが、アクセス時刻のような必須でないメタデータは省略する。`sync()`[(man page)](https://man7.org/linux/man-pages/man2/sync.2.html)はシステム全体のすべてをフラッシュする。

```c
/* fsync: このファイルのデータ + メタデータをフラッシュする */
fsync(fd)
    → vfs_fsync() → file->f_op->fsync()
    → ext4_sync_file()
        → filemap_write_and_wait_range()  ← ダーティページをフラッシュ
        → ext4_flush_completed_IO()       ← ジャーナルのコミット

/* fdatasync: データ + サイズのメタデータのみフラッシュする（atime/mtimeなし） */
fdatasync(fd)  /* データだけが重要な場合はfsyncより高速 */

/* sync: システム全体のすべてのダーティデータとメタデータをフラッシュする */
sync()         /* 控えめに使うこと — 完了するまでブロックする */
```

## 参考文献 {#further-reading}

- [ページキャッシュ](../mm/page-cache.md) — ダーティページがメモリ内にどう保持されるか
- [ファイル操作](file-ops.md) — file_operations vtable
- [VFSオブジェクト](vfs-objects.md) — struct file、inode、dentry、superblock
- [ファイル読み込みの一生](../mm/life-of-read.md) — 読み込みの方向
