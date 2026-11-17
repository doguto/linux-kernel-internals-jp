# 拡張属性（xattr） {#extended-attributes-xattr}

> ファイルやディレクトリに任意のメタデータを付与する

## 拡張属性とは何か {#what-are-extended-attributes}

拡張属性（xattr）を使うと、ファイルやディレクトリに名前と値の組を付与できる。これは従来のPOSIXメタデータ（所有者、パーミッション、タイムスタンプ）を任意のキーと値のデータで拡張するものである。xattrのサポートはLinux 2.4で導入された[(man page)](https://www.man7.org/linux/man-pages/man2/getxattr.2.html)。

```c
#include <sys/xattr.h>

/* 拡張属性を設定する */
setxattr("/path/to/file", "user.comment", "reviewed", 8, 0);

/* 拡張属性を取得する */
char value[256];
ssize_t len = getxattr("/path/to/file", "user.comment", value, sizeof(value));

/* すべての属性を列挙する */
char list[4096];
ssize_t list_len = listxattr("/path/to/file", list, sizeof(list));
/* listはNUL区切り: "user.comment\0user.author\0" */

/* 属性を削除する */
removexattr("/path/to/file", "user.comment");
```

シンボリックリンクをたどらない`l`系の変種（`lgetxattr`、`lsetxattr`）や、オープンされたファイルディスクリプタに対して動作する`f`系の変種（`fgetxattr`、`fsetxattr`）もある。

## 名前空間プレフィックス {#namespace-prefixes}

すべてのxattr名は、認識される名前空間プレフィックスで始まらなければならない:

| 名前空間 | プレフィックス | 設定/取得できる者 | 目的 |
|-----------|--------|----------------|---------|
| `user` | `user.` | 任意のプロセス（ファイルへの書き込み権限があれば） | アプリケーションの一般的な用途 |
| `system` | `system.` | カーネルと特権プロセス | NFSv4 ACL、POSIX ACL |
| `security` | `security.` | カーネルのセキュリティモジュール（LSM） | SELinuxラベル、AppArmor |
| `trusted` | `trusted.` | CAP_SYS_ADMINのみ | コンテナランタイム、OverlayFS |

### `user`名前空間 {#user-namespace}

ファイルへの書き込み権限を持つ任意のプロセスが`user.*`属性を設定できる:

```bash
# アプリケーション用途でファイルにタグを付ける
setfattr -n user.reviewed -v "2024-01-15" report.pdf
getfattr -n user.reviewed report.pdf
# file: report.pdf
# user.reviewed="2024-01-15"

# すべてのuser属性を列挙する
getfattr -d report.pdf
```

制限:

- シンボリックリンクやデバイスファイルではサポートされない（rootが所有するデバイスを介した権限昇格を防ぐため）
- 有効であればファイルシステムのクォータの対象になる

### `security`名前空間 {#security-namespace}

Linux Security Modulesが使用する。これらを直接設定することはまれである:

```bash
# SELinuxのファイルコンテキストラベル
getfattr -n security.selinux /etc/passwd
# security.selinux="system_u:object_r:passwd_file_t:s0"

# AppArmor（security.apparmorを使う）
getfattr -n security.apparmor /usr/bin/python3
```

`security.*`の設定には、`CAP_SYS_ADMIN`またはLSM固有の権限が必要である。

### `trusted`名前空間 {#trusted-namespace}

`CAP_SYS_ADMIN`が必要である。次のものが使用する:

- **OverlayFS**: ディレクトリ/ファイルを隠すための`trusted.overlay.opaque`、`trusted.overlay.whiteout`
- **コンテナランタイム**: イメージレイヤ上にコンテナのメタデータを格納する
- **クォータシステム**: 一部の実装はクォータデータを`trusted.*`に格納する

```bash
# OverlayFSのopaqueディレクトリマーカー
getfattr -n trusted.overlay.opaque /some/overlayfs/dir
# trusted.overlay.opaque="y"
```

### `system`名前空間 {#system-namespace}

POSIX ACL（NFSv4および従来のもの）に使用される:

```bash
# POSIX ACLはsystem.posix_acl_accessとして格納される
setfacl -m u:alice:rw /shared/file
getfattr -n system.posix_acl_access /shared/file
# バイナリ形式のACLエンコーディングが返る
```

## ファイルシステムのサポート {#filesystem-support}

```bash
# ファイルシステムがxattrをサポートしているか確認する
mount | grep " / " | grep -o 'user_xattr\|xattr'
# または
tune2fs -l /dev/sda1 | grep "Default mount options"

# ext4: デフォルトですべての名前空間をサポート
# tmpfs: user.*とtrusted.*をサポート（デフォルトではsecurityなし）
# btrfs: 完全なxattrサポート
# XFS: 完全なxattrサポート（属性が多い場合はinodeごとにB-treeを使う）
# NFS: サーバーに依存。NFSv4.2はネイティブのxattrサポートを持つ
# OverlayFS: 上位/下位のファイルシステムに委譲する
```

## カーネルの実装 {#kernel-implementation}

### xattrのディスパッチ: struct xattr_handler {#xattr-dispatch-struct-xattr_handler}

各ファイルシステムは、名前空間ごとにハンドラを登録する:

```c
/* include/linux/xattr.h */
struct xattr_handler {
    const char *name;    /* 名前の完全一致（例: "system.posix_acl_access"） */
    const char *prefix;  /* プレフィックス一致（例: "user."）  */
    int flags;

    bool (*list)(struct dentry *dentry);
    int  (*get)(const struct xattr_handler *,
                struct dentry *, struct inode *,
                const char *name, void *buffer, size_t size);
    int  (*set)(const struct xattr_handler *,
                struct mnt_idmap *, struct dentry *, struct inode *,
                const char *name, const void *value, size_t size, int flags);
};
```

ext4のxattrハンドラテーブル:

```c
/* fs/ext4/xattr.c */
static const struct xattr_handler * const ext4_xattr_handler_map[] = {
    [EXT4_XATTR_INDEX_USER]              = &ext4_xattr_user_handler,
    [EXT4_XATTR_INDEX_POSIX_ACL_ACCESS]  = &posix_acl_access_xattr_handler,
    [EXT4_XATTR_INDEX_POSIX_ACL_DEFAULT] = &posix_acl_default_xattr_handler,
    [EXT4_XATTR_INDEX_TRUSTED]           = &ext4_xattr_trusted_handler,
    [EXT4_XATTR_INDEX_SECURITY]          = &ext4_xattr_security_handler,
    NULL
};

const struct xattr_handler * const *ext4_xattr_handlers = ext4_xattr_handler_map;
```

### VFSのxattrシステムコール経路 {#vfs-xattr-syscall-path}

```c
/* fs/xattr.c */
SYSCALL_DEFINE5(setxattr, const char __user *, pathname,
                const char __user *, name,
                const void __user *, value, size_t, size, int, flags)
{
    struct path path;
    user_path_at(AT_FDCWD, pathname, LOOKUP_FOLLOW, &path);
    error = setxattr(mnt_idmap(path.mnt), path.dentry, name, kvalue, size, flags);
}

int setxattr(struct mnt_idmap *idmap, struct dentry *dentry,
             const char __user *name, const void __user *value,
             size_t size, int flags)
{
    /* 名前空間プレフィックスを解析する: "user."、"security." など */
    handler = xattr_resolve_name(dentry->d_inode, &kname);

    /* 名前空間固有の権限を確認する */
    error = handler->set(handler, idmap, dentry, inode, kname,
                          kvalue, size, flags);
}
```

### ext4での格納 {#storage-in-ext4}

ext4はxattrを2つの場所に格納する:

1. **inode本体**（EA inodeインライン）: 小さな属性はinodeの未使用領域に直接格納される
2. **専用のxattrブロック**: 複数の属性をキーと値の形式で保持する4KBのブロック

```
ext4 inode
├── i_extra_isize (points to end of fixed inode)
└── [xattr inline area]
    ├── entry: name_index=1("user.") name_len=3 value_len=5 → "foo" = "hello"
    └── ...

[if overflow] → separate xattr block (4096 bytes)
    ├── header: magic, refcount, hash
    ├── entries: name/value pairs sorted by name
    └── values: stored at end of block (grow backwards)
```

```bash
# ext4のxattr格納状況を調べる
debugfs -R "stat <inode_number>" /dev/sda1
# i_extra_isize: 28
# Extended attributes stored in inode body / block
```

## セキュリティモジュールとの統合 {#security-module-integration}

SELinuxやその他のLSMは、ファイルにラベルを付けるために`security.*` xattrを使う:

```c
/* security/selinux/hooks.c */
static int selinux_inode_init_security(struct inode *inode,
                                        struct inode *dir,
                                        const struct qstr *qstr,
                                        const char **name,
                                        void **value,
                                        size_t *len)
{
    /* 親のコンテキストに基づいて新しいファイルにsecurity.selinuxを設定する */
    xattrs[0].name  = XATTR_SELINUX_SUFFIX;  /* "selinux" */
    xattrs[0].value = context;
    xattrs[0].value_len = clen;
    return 0;
}

/* vfs_create → security_inode_init_security から呼ばれる */
```

```bash
# ファイルのラベルを付け直す（ポリシーからSELinuxコンテキストを復元する）:
restorecon /path/to/file

# コンテキストを確認する:
ls -Z /etc/passwd
# system_u:object_r:passwd_file_t:s0 /etc/passwd

# 復元後に再帰的にラベルを付け直す:
restorecon -R /var/www/html/
```

## fscryptとxattr {#fscrypt-and-xattr}

fscryptは、ファイルごとの暗号化ポリシーをxattrとして格納する:

```c
/* fs/crypto/policy.c */
/* fscryptはファイルシステムのset_context操作を介して暗号化コンテキストを格納する: */
static int fscrypt_set_context(struct inode *inode, void *fs_data)
{
    /* ファイルシステム内部のxattrインデックス（EXT4_XATTR_INDEX_ENCRYPTION）を使って格納され、
     * 名前付きのuser/trusted xattrとして直接アクセスすることはできない */
    return inode->i_sb->s_cop->set_context(inode, &ci->ci_policy,
                                            fscrypt_context_size(&ci->ci_policy), fs_data);
}

/* ext4が使うもの: EXT4_XATTR_INDEX_ENCRYPTION → 内部xattr。trusted.fscryptとしては公開されない */
```

```bash
# ファイルが暗号化されているか確認する（getfattrではなくfscryptのツールを使う）:
fscryptctl get_policy /encrypted/file
# または:
lsattr /encrypted/file | grep -o 'E'
```

## OverlayFSのwhiteoutとopaqueマーカー {#overlayfs-whiteout-and-opaque-markers}

OverlayFSは、レイヤ構造を実装するために`trusted.*` xattrを使う:

```bash
# opaqueディレクトリ: overlayfsに下位レイヤの探索をやめるよう伝える
getfattr -n trusted.overlay.opaque /overlay/upper/somedir
# trusted.overlay.opaque="y"

# origin: これがどの下位inodeからコピーされたかを追跡する
getfattr -n trusted.overlay.origin /overlay/upper/file
```

```c
/* fs/overlayfs/xattr.c */
int ovl_check_setxattr(struct ovl_fs *ofs, struct dentry *upperdentry,
                        enum ovl_xattr ox, const void *value, size_t size, int xerr)
{
    /* ox == OVL_XATTR_OPAQUE: trusted.overlay.opaque */
    return ovl_do_setxattr(ofs, upperdentry, ox, value, size);
}
```

## 参考文献 {#further-reading}

- [VFSオブジェクト](vfs-objects.md) — xattrを格納するinode
- [OverlayFS](../filesystems/overlayfs.md) — trusted.overlay.* xattrを使う
- [dm-cryptとfscrypt](../crypto/encryption.md) — fscryptの暗号化ポリシーxattr
- [LSMフレームワーク](../security/lsm.md) — SELinux/AppArmorはsecurity.* xattrを使う
- `fs/xattr.c` — VFSのxattrディスパッチ
- `fs/ext4/xattr.c` — ext4のxattr格納の実装
