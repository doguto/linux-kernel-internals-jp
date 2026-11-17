# OverlayFS {#overlayfs}

> コンテナイメージのレイヤのためのユニオンマウント。SUSEのMiklos Szerediが書き、Linux 3.18でマージされた [(commit)](https://git.kernel.org/linus/e9be9d5e76e34872f0c37d72e25bc27fe9e2c54c) [(LWN)](https://lwn.net/Articles/618140/)。

## OverlayFSが行うこと {#what-overlayfs-does}

OverlayFSは2つ以上のディレクトリツリーをマージしたビューを提示する。

```
overlay mount (what containers see):
  /
  ├── bin/bash          ← lower 由来（ベースイメージ、読み取り専用）
  ├── lib/              ← lower 由来
  ├── etc/hostname      ← upper 由来（コンテナ固有、書き込み可能）
  └── tmp/myfile        ← upper 由来（コンテナが書き込んだもの）

lower (read-only, multiple layers possible):
  Layer 3 (app image):  /app/
  Layer 2 (node):       /usr/local/bin/node
  Layer 1 (debian):     /bin/ /lib/ /etc/ ...

upper (writable layer — per container):
  etc/hostname
  tmp/myfile

work (internal scratch dir, must be on same filesystem as upper):
  (used for atomic copy-up)
```

## OverlayFSのマウント {#mounting-overlayfs}

```bash
# 基本的な 2 レイヤのオーバーレイ
mount -t overlay overlay \
    -o lowerdir=/lower,upperdir=/upper,workdir=/work \
    /merged

# 複数レイヤ（Docker はイメージレイヤにこれを使う）
# レイヤは右から左へ並べる: 一番右が最下層のレイヤ
mount -t overlay overlay \
    -o lowerdir=/layer3:/layer2:/layer1,upperdir=/upper,workdir=/work \
    /merged

# 読み取り専用のオーバーレイ（upper/work なし — 複数イメージのユニオンに便利）
mount -t overlay overlay \
    -o lowerdir=/layer2:/layer1 \
    /merged
```

## Copy-on-write（copy-up） {#copy-on-write-copy-up}

コンテナが下位レイヤ由来のファイルに書き込むと、OverlayFSは次のように動作する。
1. ファイルをlowerからupperへコピーする（copy-up）
2. upperのコピーに書き込みを適用する
3. 以後の読み出しはupperのコピーを見る

```
Container: open("/etc/passwd", O_RDWR)
  → File is in lower layer (read-only)
  → OverlayFS: copy /etc/passwd to upper/etc/passwd
  → Open the upper copy for writing
  → Container sees the modified version
  → Lower layer unchanged (other containers unaffected)
```

### カーネル内のcopy-up {#copy-up-in-the-kernel}

```c
/* fs/overlayfs/copy_up.c */
static int ovl_copy_up_inode(struct ovl_copy_up_ctx *c, struct dentry *temp)
{
    struct ovl_fs *ofs = OVL_FS(c->dentry->d_sb);
    int err;

    /* データをコピーする */
    if (S_ISREG(c->stat.mode)) {
        struct path upperpath, datapath;
        ovl_path_upper(c->dentry, &upperpath);
        ovl_copy_up_data(c, &upperpath);
    }

    /* メタデータ（タイムスタンプ、xattr、所有者）をコピーする */
    err = ovl_copy_up_metadata(c, temp);

    return err;
}

static int ovl_copy_up_data(struct ovl_copy_up_ctx *c,
                              const struct path *temp)
{
    struct file *new_file, *old_file;

    old_file = ovl_path_open(&c->lowerpath, O_LARGEFILE | O_RDONLY);
    new_file = ovl_path_open(temp, O_LARGEFILE | O_WRONLY);

    /* アトミックなコピー: ゼロコピーのための splice/copy_file_range */
    err = do_splice_direct(old_file, &old_pos, new_file, &new_pos,
                            len, SPLICE_F_MOVE);
    return err;
}
```

copy-upはアトミックである。カーネルは`workdir`内の一時ファイルに書き込み、それを`upper`内の最終的な位置へ`rename()`する。途中で中断されても、部分的なファイルが見えることはない。

## ホワイトアウトファイル {#whiteout-files}

コンテナが下位レイヤのファイルを*削除*すると、OverlayFSはupperに**ホワイトアウト**エントリを作成する — ファイルが削除されたことを示す特別なファイルである。

```bash
# コンテナが /etc/shadow を削除する
rm /etc/shadow

# 上位レイヤにホワイトアウトが作成される:
ls -la /upper/etc/
# c---------  0 root root 0, 0 ... shadow    ← キャラクタデバイス 0,0 = ホワイトアウト

# upper:/etc/shadow がホワイトアウトなので、OverlayFS は lower:/etc/shadow を隠す
```

### 不透明ディレクトリ {#opaque-directories}

コンテナがディレクトリを削除して作り直した場合（あるいはディレクトリのcopy-upが起きた場合）、OverlayFSはupperのディレクトリを**opaque**（不透明）としてマークする — 下位ディレクトリの内容は隠される。

```bash
# コンテナ: rm -rf /etc && mkdir /etc && echo "new" > /etc/hosts

# upper/etc は xattr: trusted.overlay.opaque = "y" を持つ
getfattr -n trusted.overlay.opaque /upper/etc/
# trusted.overlay.opaque="y"
# → lower/etc/ の内容は完全に隠される
```

## Metacopy最適化 {#metacopy-optimization}

**Metacopy**（`metacopy=on`）は、データのcopy-upを実際のデータ書き込みまで先送りする。メタデータの変更（chmod、chown、xattr）ではメタデータのみをコピーし、データはlowerに残す。

```
Without metacopy:  chown myfile → full data copy-up
With metacopy:     chown myfile → only metadata copied to upper
                   write to myfile → data copy-up happens now
```

```bash
mount -t overlay overlay \
    -o lowerdir=lower,upperdir=upper,workdir=work,metacopy=on \
    /merged
```

これにより、パーミッション変更時のcopy-upコストが劇的に削減される。

## Dockerのoverlay2ドライバ {#dockers-overlay2-driver}

DockerはOverlayFSをデフォルトのストレージドライバ（`overlay2`）として使っている — ほとんどのLinuxディストリビューションで、AUFSに代わってデフォルトとなった。

```bash
# Docker イメージのレイヤ
ls /var/lib/docker/overlay2/
# <layer-id>/
#   diff/      ← そのレイヤのファイルシステムツリー
#   link       ← このレイヤの短いシンボリックリンク名
#   lower      ← 親レイヤのシンボリックリンクを ':' 区切りで並べたリスト
#   merged/    ← マウントされたオーバーレイ（実行中のコンテナのみ）
#   work/      ← オーバーレイの workdir

# 実行中のコンテナのオーバーレイを調べる
docker inspect <container> | jq '.[].GraphDriver.Data'
# {
#   "LowerDir": "/var/lib/docker/overlay2/abc/diff:/var/lib/docker/overlay2/def/diff",
#   "MergedDir": "/var/lib/docker/overlay2/xyz/merged",
#   "UpperDir": "/var/lib/docker/overlay2/xyz/diff",
#   "WorkDir": "/var/lib/docker/overlay2/xyz/work"
# }
```

### レイヤの共有 {#layer-sharing}

コンテナにとってのOverlayFSの強みは、同じベースイメージを持つ複数のコンテナが下位レイヤを共有できることである。

```
debian:latest (shared, read-only):
  lower layers: /var/lib/docker/overlay2/abc/diff

Container A:                Container B:
  upper: /xyz/diff           upper: /pqr/diff
  lower: /abc/diff           lower: /abc/diff    (same!)
  (Container A changes)      (Container B changes)
```

コンテナAでの変更はコンテナBに影響しない — それぞれが自分の上位レイヤを持っている。

## OverlayFSの内部 {#overlayfs-internals}

### struct ovl_entry {#struct-ovl_entry}

```c
/* fs/overlayfs/ovl_entry.h */
struct ovl_entry {
    union {
        struct {
            unsigned long    flags;
        };
        struct rcu_head     rcu;
    };
    unsigned                 numlower;   /* 下位レイヤの数 */
    struct ovl_path          lowerstack[]; /* 下位パスのフレキシブル配列 */
};

/* inode ごとのオーバーレイ状態 */
struct ovl_inode {
    union {
        struct ovl_dir_cache  *cache;   /* ディレクトリエントリキャッシュ */
        struct inode          *lowerdata; /* 下位のデータ inode（metacopy） */
    };
    const char        *redirect;         /* redirect xattr の値 */
    u64                version;          /* copy-up のバージョン */
    unsigned long      flags;
    struct inode       vfs_inode;        /* 埋め込まれた VFS inode */
    struct dentry     *__upperdentry;    /* 上位の dentry（copy-up 後） */
    struct ovl_entry  *oe;               /* 下位レイヤ */
};
```

### ルックアップ: ディレクトリエントリのマージ {#lookup-merging-directory-entries}

```c
/* fs/overlayfs/dir.c */
static struct dentry *ovl_lookup(struct inode *dir, struct dentry *dentry,
                                  unsigned int flags)
{
    struct ovl_entry *oe;
    const struct cred *old_cred;

    /* まず upper を検索する */
    if (ovl_dentry_upper(dentry->d_parent)) {
        upperdentry = ovl_lookup_upper(dentry, ...);
        if (upperdentry && ovl_is_whiteout(upperdentry)) {
            /* ホワイトアウトが見つかった: ファイルは削除済み */
            return d_splice_alias(ERR_PTR(-ENOENT), dentry);
        }
    }

    /* 各下位レイヤを検索する */
    for (i = 0; i < poe->numlower; i++) {
        lowerdentry = ovl_lookup_lower(dentry, &poe->lowerstack[i], ...);
        if (lowerdentry)
            break;
    }

    /* マージ: upper（存在すれば）が lower より優先される */
    /* 不透明ディレクトリ: lower の検索をやめる */
}
```

## 性能上の考慮事項 {#performance-considerations}

```bash
# copy-up のレイテンシ: 大きなファイルへの最初の書き込みは完全なコピーを引き起こす
# これはコンテナでレイテンシのスパイクを引き起こしうる

# copy-up のコストを計測する
time (docker exec mycontainer sh -c "echo x >> /etc/large_file")

# 頻繁に書き込まれるデータには名前付きボリュームを使って copy-up を避ける
docker run -v /host/data:/container/data myimage

# オーバーレイの統計を確認する
cat /proc/fs/overlayfs/features  # サポートされている機能

# inode の使用量（overlayfs は上位レイヤのためにホストの inode を消費する）
df -i /var/lib/docker
```

## 参考文献 {#further-reading}

- [コンテナネットワーキング](../net/container-networking.md) — コンテナのネットワーク側
- [cgroup: コンテナの分離](../cgroups/container-isolation.md) — コンテナの完全なセットアップ
- [VFS: ファイルシステムの登録とマウント](../vfs/mounting.md) — overlayfsがどのように登録されるか
- [VFS: パス解決](../vfs/namei.md) — overlayfsを通じたdentryルックアップ
- [メモリ管理: Copy-on-Write](../mm/cow.md) — pageレベルのCoWとファイルレベルのCoW
- [Kernel docs: overlay filesystem](https://docs.kernel.org/filesystems/overlayfs.html) — セマンティクス、copy-up、複数の下位レイヤ
- カーネルツリーの`fs/overlayfs/` — OverlayFSの実装
