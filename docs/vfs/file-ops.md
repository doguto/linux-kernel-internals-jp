# ファイル操作とfile構造体 {#file-operations-and-the-file-struct}

> open()、read()、write()、close()がVFSでどう動作するか

## file_operations vtable {#the-file_operations-vtable}

オープンされたすべてのファイルは`struct file`を持ち、その`f_op`ポインタはファイルシステムの`file_operations`を指している。VFSはすべてのI/O操作でこのvtableを経由して呼び出す[(kernel docs)](https://www.kernel.org/doc/html/latest/filesystems/vfs.html):

```c
/* include/linux/fs.h */
struct file_operations {
    loff_t  (*llseek)(struct file *, loff_t, int);
    ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
    ssize_t (*read_iter)(struct kiocb *, struct iov_iter *);   /* 推奨 */
    ssize_t (*write_iter)(struct kiocb *, struct iov_iter *);  /* 推奨 */
    int     (*mmap)(struct file *, struct vm_area_struct *);
    int     (*open)(struct inode *, struct file *);
    int     (*release)(struct inode *, struct file *);  /* 最後のclose時 */
    int     (*fsync)(struct file *, loff_t, loff_t, int datasync);
    long    (*unlocked_ioctl)(struct file *, unsigned int, unsigned long);
    int     (*iterate_shared)(struct file *, struct dir_context *);
    __poll_t (*poll)(struct file *, struct poll_table_struct *);
};
```

ファイルシステムは通常、その一部だけを実装する。NULLのままのフィールドについては、VFSは汎用のフォールバック（例: `generic_file_read_iter`、`noop_llseek`）を使う。

## open(): システムコールからstruct fileまで {#open-from-syscall-to-struct-file}

`open(2)`システムコールはLinuxのmanページに記載されている[(man page)](https://man7.org/linux/man-pages/man2/open.2.html)。ページキャッシュを迂回する`O_DIRECT`は、Linux 2.4.10で追加された[(man page)](https://man7.org/linux/man-pages/man2/open.2.html)。

```c
/* 簡略化: open("/path/to/file", O_RDWR) */
SYSCALL_DEFINE3(open, ...) → do_sys_open()
    1. 新しいファイルディスクリプタ番号を取得する
    2. do_filp_open()
         a. path_openat(): パス名を解決する → path + flags
         b. do_last(): 最後の構成要素を処理する
              - パーミッションチェック: inode_permission()
              - O_CREATでファイルが存在しない場合: i_op->create()を呼ぶ
              - alloc_file(): struct fileを割り当てる
                   f_path = 解決されたパス
                   f_op = inode->i_fop
                   f_mode = FMODE_READ | FMODE_WRITE
              - f_op->open(inode, file)を呼ぶ
                   （例えばext4_file_open）
    3. プロセスのファイルテーブルに fd → struct file を登録する
    4. fdを返す
```

`f_op->open()`の呼び出しは、ファイルシステムが`file->private_data`を設定する機会である（例: 内容を動的に生成する/procファイルの場合）。

## read(): システムコールの経路 {#read-the-syscall-path}

```c
/* read(fd, buf, count) */
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
{
    struct fd f = fdget_pos(fd);  /* struct fileを取得し、f_posロックを取る */
    loff_t pos = file_pos_read(f.file);

    ret = vfs_read(f.file, buf, count, &pos);  /* f_op->readまたはread_iterを呼ぶ */

    file_pos_write(f.file, pos);  /* f_posを更新する */
    fdput_pos(f);
    return ret;
}

/* vfs_read → ファイルシステムのread_iterを呼ぶ */
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
{
    if (file->f_op->read)
        ret = file->f_op->read(file, buf, count, pos);
    else if (file->f_op->read_iter)
        ret = new_sync_read(file, buf, count, pos);
    /* new_sync_readはread_iterをiov_iterでラップする */
}
```

通常ファイルの場合、`read_iter`は`generic_file_read_iter()`を呼び出し、これは次のことを行う:
1. 必要なページがページキャッシュにあるか確認する
2. キャッシュにある場合: ページをユーザー空間へコピーする（`copy_to_user`）
3. キャッシュにない場合: `readahead`を起動 → `mapping->a_ops->read_folio()` → ディスクI/O → 待機 → コピー

## write(): ダーティページとライトバック {#write-dirty-pages-and-writeback}

```c
/* write(fd, buf, count) */
vfs_write() → file->f_op->write_iter()
    → generic_file_write_iter()
        → generic_perform_write()
            書き込みが必要な各ページについて:
                1. ページキャッシュ内のページを検索または割り当てる
                2. copy_from_user() → page
                3. ページをダーティにする（set_page_dirty()）
                4. 必要ならinodeのサイズを更新する

/* ページはすぐにはディスクへ書き込まれない！
   ページキャッシュ内でダーティとしてマークされるだけである。
   ライトバック（BDIごとのフラッシャーkworker）が後で書き出す。 */
```

データがディスク上にあることが保証されるのは、次の後に限られる:

- `fsync(fd)`または`fdatasync(fd)` — 明示的なフラッシュ[(man page)](https://man7.org/linux/man-pages/man2/fsync.2.html)
- ライトバックデーモンによるフラッシュ（通常は30秒後、またはメモリ逼迫が高いとき。`dirty_expire_centisecs`で制御される）
- ファイルシステムの`sync_fs()`が呼ばれたとき

## llseek(): ファイル位置の更新 {#llseek-updating-the-file-position}

```c
loff_t vfs_llseek(struct file *file, loff_t offset, int whence)
{
    if (file->f_op->llseek)
        return file->f_op->llseek(file, offset, whence);
    return generic_file_llseek(file, offset, whence);
}

/* 汎用実装 */
loff_t generic_file_llseek(struct file *file, loff_t offset, int whence)
{
    switch (whence) {
    case SEEK_SET: offset = offset; break;
    case SEEK_CUR: offset = file->f_pos + offset; break;
    case SEEK_END: offset = inode->i_size + offset; break;
    }
    /* 検証してf_posを更新する */
    file->f_pos = offset;
    return offset;
}
```

## mmap(): ファイルをアドレス空間にマッピングする {#mmap-mapping-the-file-into-address-space}

```c
/* mmap(NULL, len, PROT_READ, MAP_SHARED, fd, offset) */
vm_mmap() → do_mmap()
    → mmap_region()
        → vma = vm_area_alloc(mm)
        → call_mmap(file, vma)    /* file->f_op->mmap() */

/* 通常ファイルの場合: */
int generic_file_mmap(struct file *file, struct vm_area_struct *vma)
{
    vma->vm_ops = &generic_file_vm_ops;
    /* vm_ops->fault()がオンデマンドでページフォールトを処理する */
    return 0;
}
```

実際のページは、アクセスされるまでフォールトインされない。各フォールトは`filemap_fault()`を呼び出し、これがページキャッシュ（またはディスク）から読み込む。

## release(): 最後のclose {#release-last-close}

`f_op->release()`は、ある`struct file`を指す最後のファイルディスクリプタがクローズされたときに呼ばれる。注意: `close()`はすぐに`release()`を呼ぶわけではない。参照カウントを減らすだけで、`release()`はそれが0になったときにのみ呼ばれる。

```c
/* close(fd) */
__close_fd() → filp_close()
    → f_op->flush()      /* closeのたびに呼ばれる */
    → fput(file)         /* 参照カウントを減らす */
        → if (--f_ref == 0): __fput(file)
            → f_op->release(inode, file)
            → dput(dentry)   /* dentryの参照を解放する */
```

`flush()`と`release()`の違い:

- `flush()`: `close()`のたびに呼ばれる。これをエラーチェックに使うファイルシステムもある（NFSのダーティデータのフラッシュ）
- `release()`: `struct file`への最後の参照が落とされたときにのみ呼ばれる（例: `dup()`したfdがすべてクローズされた後）

## fsync(): データをディスクへ強制的に書き出す {#fsync-forcing-data-to-disk}

```c
int vfs_fsync(struct file *file, int datasync)
{
    return vfs_fsync_range(file, 0, LLONG_MAX, datasync);
}

/* ファイルシステムはこれを実装してフラッシュする:
   datasync=0: データ + メタデータをフラッシュ（完全なfsync）
   datasync=1: データ + サイズのメタデータのみフラッシュ（fdatasync） */
```

## 参考文献 {#further-reading}

- [VFSオブジェクト](vfs-objects.md) — 4つのVFS構造体
- [write()システムコールの一生](life-of-write.md) — エンドツーエンドの書き込み経路
- [ページキャッシュ](../mm/page-cache.md) — ファイルデータがバッファリングされる場所
- [ファイル読み込みの一生](../mm/life-of-read.md) — 読み込みの全経路
