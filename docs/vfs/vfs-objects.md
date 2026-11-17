# VFSオブジェクト: superblock、inode、dentry、file {#vfs-objects-superblock-inode-dentry-file}

> Linuxにおけるあらゆるファイルシステムオブジェクトを表す4つのデータ構造

## 概要 {#overview}

VFSは、組み合わせてあらゆるファイルシステム資源を表す4つのオブジェクトを定義している[(kernel docs)](https://www.kernel.org/doc/html/latest/filesystems/vfs.html):

```
Mounted filesystem
    │
    └── struct super_block
           (one per mounted filesystem)
               │
               └── struct inode  ←────────────────────────────────┐
                      (one per file/dir/symlink)                   │
                          │                                        │
                          └── struct dentry ──────── struct dentry │
                                 /                                 bin
                                 usr                               bash
                          (path components, cached)
                          │
                          └── struct file
                                 (one per open() call)
```

## struct super_block: マウントされたファイルシステム {#struct-super_block-the-mounted-filesystem}

`super_block`は、マウントされたファイルシステムのインスタンスごとに1つ存在する。マウントされたファイルシステムのグローバルな状態を保持する:

```c
/* include/linux/fs.h */
struct super_block {
    dev_t                    s_dev;       /* デバイス識別子 */
    unsigned long            s_blocksize; /* バイト単位のブロックサイズ */
    loff_t                   s_maxbytes;  /* 最大ファイルサイズ */
    struct file_system_type *s_type;      /* ファイルシステムタイプ */
    const struct super_operations *s_op; /* superblock操作のvtable */
    unsigned long            s_flags;    /* SB_RDONLYなど（4.17以降はSB_*フラグ） */
    unsigned long            s_magic;    /* ファイルシステムのマジックナンバー */
    struct dentry           *s_root;     /* ルートディレクトリのdentry */
    struct rw_semaphore      s_umount;   /* アンマウント用セマフォ */
    void                    *s_fs_info;  /* ファイルシステム固有のデータ */
};

struct super_operations {
    struct inode *(*alloc_inode)(struct super_block *sb);
    void         (*destroy_inode)(struct inode *inode);
    void         (*dirty_inode)(struct inode *inode, int flags);
    int          (*write_inode)(struct inode *inode,
                                struct writeback_control *wbc);
    void         (*evict_inode)(struct inode *inode);
    void         (*put_super)(struct super_block *sb);
    int          (*sync_fs)(struct super_block *sb, int wait);
    int          (*statfs)(struct dentry *dentry, struct kstatfs *kstatfs);
};
```

`s_fs_info`は「抜け道」であり、ファイルシステムはここに独自のプライベートデータを格納する。ext4の場合、これはジャーナルハンドルやブロックグループディスクリプタなどを持つ`struct ext4_sb_info`である。

## struct inode: ファイルシステムオブジェクト {#struct-inode-a-file-system-object}

`inode`は、ファイル、ディレクトリ、シンボリックリンク、デバイスノード、ソケットなど、ファイルシステム内の名前付きオブジェクトを表す。1つのinodeは複数の名前（ハードリンク）を持てるが、メタデータは1組だけである。

```c
/* include/linux/fs.h */
struct inode {
    umode_t              i_mode;     /* ファイル種別 + パーミッション */
    unsigned int         i_flags;    /* S_IMMUTABLE, S_APPEND など */
    kuid_t               i_uid;      /* 所有者のUID */
    kgid_t               i_gid;      /* 所有者のGID */
    const struct inode_operations *i_op;  /* inode操作のvtable */
    struct super_block  *i_sb;       /* これが属するファイルシステム */
    struct address_space *i_mapping; /* このファイルのページキャッシュ */
    unsigned long        i_ino;      /* inode番号 */
    unsigned int         i_nlink;    /* ハードリンクの数 */
    loff_t               i_size;     /* バイト単位のファイルサイズ */
    blkcnt_t             i_blocks;   /* 512バイトブロックの数 */
    spinlock_t           i_lock;     /* i_blocks、i_bytes、i_sizeを保護する */
    struct timespec64    __i_atime;  /* アクセス時刻 */
    struct timespec64    __i_mtime;  /* 変更時刻 */
    struct timespec64    __i_ctime;  /* 状態変更時刻 */
};

struct inode_operations {
    struct dentry *(*lookup)(struct inode *, struct dentry *, unsigned int);
    int (*create)(struct mnt_idmap *, struct inode *, struct dentry *,
                  umode_t, bool);
    int (*link)(struct dentry *, struct inode *, struct dentry *);
    int (*unlink)(struct inode *, struct dentry *);
    int (*mkdir)(struct mnt_idmap *, struct inode *, struct dentry *, umode_t);
    int (*rmdir)(struct inode *, struct dentry *);
    int (*rename)(struct mnt_idmap *, struct inode *, struct dentry *,
                  struct inode *, struct dentry *, unsigned int);
    int (*setattr)(struct mnt_idmap *, struct dentry *, struct iattr *);
    int (*getattr)(struct mnt_idmap *, const struct path *,
                   struct kstat *, u32, unsigned int);
    int (*permission)(struct mnt_idmap *, struct inode *, int);
    const char *(*get_link)(struct dentry *, struct inode *,
                             struct delayed_call *);  /* シンボリックリンク用 */
};
```

`i_mapping`フィールドは、このinodeとページキャッシュ内のページとを結び付ける`address_space`構造体を指す。ファイルを`read()`すると、データはこの`address_space`によってインデックス付けされたページに入る。

## struct dentry: パスの構成要素 {#struct-dentry-a-path-component}

`dentry`（ディレクトリエントリ）は、名前からinodeへの対応をキャッシュする。**dentryキャッシュ**（dcache）はVFSの主要な検索キャッシュであり、カーネルはディスクへ行く前に、パスの構成要素がすでにdcacheにあるかを確認する。Linux 2.6.38以降、ほとんどのdcache検索は、Nick Pigginが導入したRCUベースのロックフリーな高速パスを使う[(LWN)](https://lwn.net/Articles/419811/)。`d_seq` seqlockは、一般的な（キャッシュヒットの）経路ではspinlockを一切取らずに、並行する変更を検出する。

```c
/* include/linux/dcache.h */
struct dentry {
    unsigned int              d_flags;    /* 状態フラグ */
    seqcount_spinlock_t       d_seq;      /* RCU walk用のseqlock（Linux 2.6.38で追加） */
    struct hlist_bl_node      d_hash;     /* 検索用ハッシュテーブル */
    struct dentry            *d_parent;   /* 親ディレクトリのdentry */
    struct qstr               d_name;     /* 名前（ハッシュ付きのquick string） */
    struct inode             *d_inode;    /* この名前が指すinode
                                            （NULL = negative dentry） */
    const struct dentry_operations *d_op;
    struct super_block       *d_sb;       /* ファイルシステムのsuperblock */
    struct lockref            d_lockref;  /* 参照カウント + spinlock */
    struct list_head          d_lru;      /* LRU追い出しリスト */
    struct hlist_head         d_children; /* 子dentry */
};
```

重要な区別:

- **positive dentry**: `d_inode != NULL` — その名前はファイルシステム内に存在する
- **negative dentry**: `d_inode == NULL` — その名前は検索されたが存在しない（失敗する検索の繰り返しを避けるためにキャッシュされる）

複数のdentryが同じinodeを指すことができる（ハードリンク）。`link("a", "b")`を実行すると、`a`と同じinodeを指す新しいdentry `b`が作成される。

## struct file: オープンファイル記述 {#struct-file-an-open-file-description}

`file`は`open()`が呼ばれるたびに作成される。複数の`file`構造体が同じinodeを参照することがある（異なるプロセスから、または同じプロセス内での複数回のオープンによって）:

```c
/* include/linux/fs.h */
struct file {
    spinlock_t                    f_lock;
    fmode_t                       f_mode;     /* FMODE_READ, FMODE_WRITE */
    const struct file_operations *f_op;       /* ファイル操作のvtable */
    struct address_space         *f_mapping;  /* ページキャッシュ */
    struct inode                 *f_inode;    /* inode */
    unsigned int                  f_flags;    /* O_RDONLY, O_NONBLOCK など */
    const struct cred            *f_cred;     /* オープン時のクレデンシャル */
    struct path                   f_path;     /* dentry + vfsmount */
    loff_t                        f_pos;      /* 現在のファイル位置 */
    file_ref_t                    f_ref;      /* 参照カウント */
    void                         *private_data; /* オープンごとのファイルシステム固有の状態 */
};

struct file_operations {
    loff_t  (*llseek)(struct file *, loff_t, int);
    ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
    ssize_t (*read_iter)(struct kiocb *, struct iov_iter *);
    ssize_t (*write_iter)(struct kiocb *, struct iov_iter *);
    int     (*mmap)(struct file *, struct vm_area_struct *);
    int     (*open)(struct inode *, struct file *);
    int     (*release)(struct inode *, struct file *);  /* 最後のclose時に呼ばれる */
    int     (*fsync)(struct file *, loff_t, loff_t, int datasync);
    long    (*unlocked_ioctl)(struct file *, unsigned int, unsigned long);
    int     (*iterate_shared)(struct file *, struct dir_context *);  /* readdir */
};
```

`f_pos`はオープンされたファイルごとに存在する。そのため、同じファイルをそれぞれ`open()`した2つのプロセスは、異なる位置から読むことができる。ただし`fork()`の後は、どちらかが再び`open()`を呼ぶまで、親と子は同じ`struct file`（したがって同じ`f_pos`）を共有する。

## 4つのオブジェクトとユーザー操作の関係 {#how-the-four-objects-relate-to-user-operations}

```
open("/usr/bin/bash", O_RDONLY)
  ↓
namei resolution:
  dentry "/" → inode of /
  dentry "usr" → inode of /usr
  dentry "bin" → inode of /usr/bin
  dentry "bash" → inode of /usr/bin/bash  ← target inode found
  ↓
alloc_file():
  new struct file created
  f_inode = bash's inode
  f_op = inode->i_fop  (ext4_file_operations)
  f_op->open() called
  ↓
returns fd (index into process's file descriptor table)

read(fd, buf, 4096)
  ↓
file = task->files->fdt->fd[fd]
file->f_op->read_iter()
  → ext4_file_read_iter()
  → generic_file_read_iter()
  → page cache lookup → disk read if miss
```

## 参考文献 {#further-reading}

- [パス解決](namei.md) — VFSが"/"からターゲットのdentryまでどう辿るか
- [ファイル操作](file-ops.md) — file_operations vtableの詳細
- [dentryキャッシュとinodeキャッシュ](dcache-icache.md) — VFSが検索をどうキャッシュするか
- [ページキャッシュ](../mm/page-cache.md) — ファイルデータがメモリ内のどこに置かれるか
