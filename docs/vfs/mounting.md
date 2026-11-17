# ファイルシステムの登録とマウント {#filesystem-registration-and-mounting}

> ファイルシステムがVFSにどう組み込まれ、mount()がどう動作するか

## ファイルシステムタイプの登録 {#registering-a-filesystem-type}

すべてのファイルシステムは`file_system_type`を定義し、それをVFSに登録しなければならない:

```c
/* include/linux/fs.h */
struct file_system_type {
    const char *name;       /* "ext4", "tmpfs", "proc" など */
    int fs_flags;           /* FS_REQUIRES_DEV, FS_USERNS_MOUNT など */
    int (*init_fs_context)(struct fs_context *); /* マウントコンテキストを埋める */
    void (*kill_sb)(struct super_block *);       /* アンマウント時の後始末 */
    struct module *owner;
    struct file_system_type *next;     /* 全ファイルシステムの連結リスト */
    struct hlist_head fs_supers;       /* このタイプの全superblock */
};

/* モジュールのロード時に登録する */
static struct file_system_type my_fs_type = {
    .name           = "myfs",
    .init_fs_context = my_init_fs_context,
    .kill_sb        = kill_litter_super,  /* 単純なインメモリfs用 */
    .owner          = THIS_MODULE,
};

static int __init myfs_init(void)
{
    return register_filesystem(&my_fs_type);
}
module_init(myfs_init);

static void __exit myfs_exit(void)
{
    unregister_filesystem(&my_fs_type);
}
module_exit(myfs_exit);
```

登録後、ファイルシステムは`/proc/filesystems`に現れる:

```bash
cat /proc/filesystems
# nodev  sysfs
# nodev  tmpfs
# nodev  bdev
# nodev  proc
#        ext4
# nodev  btrfs
#        vfat
```

`nodev`は、そのファイルシステムがブロックデバイスを必要としないことを意味する。

## mountシステムコール {#the-mount-syscall}

`mount(2)`システムコールは、ファイルシステムをディレクトリツリーに接続する[(man page)](https://man7.org/linux/man-pages/man2/mount.2.html)。

```c
/* ユーザーの呼び出し: mount("/dev/sda1", "/mnt/data", "ext4", 0, "") */
SYSCALL_DEFINE5(mount, ...)
    → path_mount()
        → do_new_mount()
            1. get_fs_type("ext4")    ← 登録済みのext4 fs_typeを探す
            2. fs_context_for_mount() ← fs_contextを割り当てる
            3. vfs_get_tree()
                 → fs_type->init_fs_context(fc)  ← ファイルシステムがコンテキストを埋める
                 → fc->ops->get_tree(fc)          ← superblockを作成/検索する
                    → ext4_get_tree() → mount_bdev()
                        a. blkdev_get_by_path("/dev/sda1") ← ブロックデバイスをオープンする
                        b. sget_dev(): このデバイスのsuper_blockを検索または作成する
                        c. ext4_fill_super(): ディスクからsuperblockを読み、
                           s_op、s_rootなどを設定する
            4. do_new_mount_fc(): マウントをツリーに接続する
                 → graft_tree(): マウントポイントに新しいvfsmountを挿入する

/* 結果: namespaceのマウントツリーに新しいstruct mountが接続される */
```

## マウントnamespaceとvfsmount {#mount-namespace-and-vfsmount}

各プロセスは、`vfsmount`オブジェクトのツリーを含む**マウントnamespace**（`struct mnt_namespace`）を持つ。マウントnamespaceはLinux 2.4.19で導入された[(man page)](https://man7.org/linux/man-pages/man7/mount_namespaces.7.html)。それ以前は、すべてのプロセスが単一のグローバルなマウントテーブルを共有していた。各`vfsmount`は1つのマウントポイントを表す:

```c
struct vfsmount {
    struct dentry *mnt_root;     /* このマウントされたfsのルートdentry */
    struct super_block *mnt_sb;  /* superblock */
    int mnt_flags;               /* MNT_READONLY, MNT_NOSUID など */
};

struct mount {
    struct vfsmount mnt;
    struct mount *mnt_parent;   /* これがマウントされている先のマウント */
    struct dentry *mnt_mountpoint; /* 親の中でマウントされている位置のdentry */
    struct list_head mnt_child;    /* mnt_parentの子 */
};
```

パス解決がマウントポイントのdentryに達すると、`follow_mount()`がマウントされたファイルシステムのルートdentryへ切り替え、新しい`vfsmount`へ横断する。

## /proc/mountsと/proc/self/mountinfo {#procmounts-and-procselfmountinfo}

```bash
# 現在のすべてのマウント
cat /proc/mounts
# sysfs /sys sysfs rw,nosuid,nodev,noexec,relatime 0 0
# proc /proc proc rw,nosuid,nodev,noexec,relatime 0 0
# /dev/sda1 / ext4 rw,relatime,errors=remount-ro 0 0
# tmpfs /tmp tmpfs rw,nosuid,nodev 0 0

# マウントIDを含む詳細な形式（systemdが使用）
cat /proc/self/mountinfo
# 36 35 98:0 /mnt1 /mnt2 rw,noatime master:1 - ext4 /dev/sdb rw,errors=remount-ro
# ^  ^  ^    ^     ^                ^           ^    ^
# mount_id parent_id maj:min root  mount_point opts peer_group fstype source
```

## 単純なインメモリファイルシステムの実装 {#implementing-a-simple-in-memory-filesystem}

新しいファイルシステムの最小限のスケルトンを示す:

```c
/* ディレクトリ用のinode操作 */
static struct inode_operations myfs_dir_iops = {
    .lookup = simple_lookup,
    .create = myfs_create,
    .mkdir  = myfs_mkdir,
    .unlink = simple_unlink,
    .rmdir  = simple_rmdir,
};

/* 通常ファイル用のファイル操作 */
static struct file_operations myfs_file_fops = {
    .read_iter  = generic_file_read_iter,
    .write_iter = generic_file_write_iter,
    .llseek     = generic_file_llseek,
    .mmap       = generic_file_mmap,
    .fsync      = noop_fsync,
};

/* アドレス空間操作（ページキャッシュ） */
static struct address_space_operations myfs_aops = {
    .dirty_folio    = filemap_dirty_folio,
    .writepage      = simple_writepage,
};

/* superblock操作 */
static struct super_operations myfs_super_ops = {
    .statfs         = simple_statfs,
    .drop_inode     = generic_delete_inode,
};

/* superblockを埋める（マウント中に呼ばれる） */
static int myfs_fill_super(struct super_block *sb, struct fs_context *fc)
{
    struct inode *root_inode;

    sb->s_maxbytes = MAX_LFS_FILESIZE;
    sb->s_blocksize = PAGE_SIZE;
    sb->s_blocksize_bits = PAGE_SHIFT;
    sb->s_magic = 0x4D594653;  /* 'MYFS' */
    sb->s_op = &myfs_super_ops;

    /* ルートinodeを作成する */
    root_inode = new_inode(sb);
    root_inode->i_ino = 1;
    root_inode->i_mode = S_IFDIR | 0755;
    root_inode->i_op = &myfs_dir_iops;
    root_inode->i_fop = &simple_dir_operations;

    /* ルートdentryを作成する */
    sb->s_root = d_make_root(root_inode);

    return 0;
}

static int myfs_get_tree(struct fs_context *fc)
{
    return get_tree_nodev(fc, myfs_fill_super);
}

static const struct fs_context_operations myfs_context_ops = {
    .get_tree = myfs_get_tree,
};

static int myfs_init_fs_context(struct fs_context *fc)
{
    fc->ops = &myfs_context_ops;
    return 0;
}

static struct file_system_type myfs_type = {
    .name            = "myfs",
    .init_fs_context = myfs_init_fs_context,
    .kill_sb         = kill_litter_super,
    .owner           = THIS_MODULE,
};
```

## bind mountとmove mount {#bind-mount-and-move-mount}

```bash
# bind mount: ディレクトリを別の場所にマウントする
mount --bind /original/path /new/path
# 同じdentry/inodeツリーを指す新しいvfsmountを作成する

# move mount: マウント位置をアトミックに変更する
mount --move /old/mountpoint /new/mountpoint
```

bind mountは、ホストのディレクトリをコンテナのnamespaceに共有するために、コンテナで広く使われている[(man page)](https://man7.org/linux/man-pages/man8/mount.8.html)。

## 参考文献 {#further-reading}

- [VFSオブジェクト](vfs-objects.md) — superblock、inode、dentry、fileとは何か
- [パス解決](namei.md) — 検索中にマウントポイントがどう横断されるか
- [write()システムコールの一生](life-of-write.md) — マウント後に何が起きるか
