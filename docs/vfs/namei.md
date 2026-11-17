# パス解決（namei） {#path-resolution-namei}

> カーネルが"/usr/bin/bash"のような文字列をどのようにinodeへ変換するか

## 課題 {#the-challenge}

パス解決は単純に聞こえる。パスを`/`で分割し、各構成要素を親ディレクトリの中で検索すればよい。しかし、次のものを扱わなければならない:

- シンボリックリンク（それ自体が解決すべきパスを含む）
- マウントポイント（別のファイルシステムへの横断）
- 並行する変更（走査中に別のプロセスがディレクトリをrenameする）
- `.`（カレントディレクトリ）と`..`（親ディレクトリ）
- 各階層でのパーミッションチェック

カーネルのパス解決は`fs/namei.c`にあり、**namei**サブシステムと呼ばれる（歴史的なUnixの関数名に由来する）。

## エントリポイント: filename_lookup {#entry-point-filename_lookup}

ほとんどのVFS操作はパスから始まる。カーネルは次の経路でそれを解決する:

```c
/* open("/usr/bin/bash", O_RDONLY) の簡略化した呼び出しチェーン */
do_sys_open()
  → do_filp_open()
    → path_openat()
      → link_path_walk()   ← メインの解決ループ
        → walk_component()  ← パスの構成要素を1つ解決する
          → lookup_fast()   ← dcache検索（RCU、ロックフリー）
          → lookup_slow()   ← ファイルシステムでの検索（i_rwsemを取る）
      → do_last()          ← 最後の構成要素を処理する（ファイルをオープン）
```

## nameidata構造体 {#the-nameidata-structure}

`struct nameidata`は解決処理全体を通じて状態を保持する:

```c
/* fs/namei.c */
struct nameidata {
    struct path path;         /* 現在位置: (dentry, vfsmount) */
    struct qstr last;         /* 最後の構成要素の名前 */
    struct path root;         /* この検索のルート */
    struct inode *inode;      /* カレントディレクトリのinode */
    unsigned int flags;       /* LOOKUP_* フラグ */
    unsigned int state;       /* ND_* 状態ビット */
    unsigned int depth;       /* シンボリックリンクの再帰深度 */
    int last_type;            /* LAST_NORM, LAST_ROOT, LAST_DOT, LAST_DOTDOT */
    /* ... */
};
```

## link_path_walk: メインループ {#link_path_walk-the-main-loop}

`link_path_walk()`は`/`で区切られたパスの構成要素を順に処理する:

```c
/* fs/namei.c（簡略化） */
static int link_path_walk(const char *name, struct nameidata *nd)
{
    /* 先頭のスラッシュを読み飛ばす */
    while (*name == '/')
        name++;

    for (;;) {
        struct qstr this;
        int type;

        /* 次の構成要素を取得する（次の '/' まで） */
        type = hash_name(nd->path.dentry, name, &this);

        /* 特殊ケース: "." と ".." */
        if (type != LAST_NORM) {
            /* . と .. を処理する */
        }

        /* 構成要素を検索する */
        err = walk_component(nd, WALK_MORE);

        /* "/" の先へ進む */
        name += this.len;
        while (*name == '/')
            name++;
        if (!*name)
            break;  /* パスの終端 */
    }
    return 0;
}
```

## walk_component: RCU-walkとref-walk {#walk_component-rcu-walk-and-ref-walk}

各構成要素は2段階で検索される:

### 段階1: RCU walk（ロックフリー、楽観的） {#stage-1-rcu-walk-lock-free-optimistic}

Linux 2.6.38でNick PigginによるRCU-walkが導入された[(LWN)](https://lwn.net/Articles/419811/)。これにより、ほとんどのパス検索は、ロックを一切取らず、参照カウントも一切増やさずに完了できる。

```c
static struct dentry *lookup_fast(struct nameidata *nd)
{
    struct dentry *parent = nd->path.dentry;

    /* RCUの下でdcacheから探す（ロックなし、非常に高速） */
    dentry = __d_lookup_rcu(parent, &nd->last, &nd->next_seq);
    if (!dentry)
        return NULL;  /* ミス: ref-walkにフォールスルー */

    /* 検証: 検索中にdentryが削除されなかったか？ */
    if (!d_try_get_rcu(dentry, nd->next_seq))
        return NULL;  /* 無効化された: 再試行 */

    return dentry;
}
```

RCU walkは`d_seq`（各dentryにあるseqlock）を使い、ロックを一切取らずに並行する変更を検出する。これが高速パスである。ほとんどの検索はdcacheにヒットし、ロックを取らずに戻る。

### 段階2: Ref walk（i_rwsemを取る） {#stage-2-ref-walk-takes-i_rwsem}

RCU walkが失敗した場合（キャッシュミスまたは再試行）:

```c
static struct dentry *lookup_slow(const struct qstr *name,
                                   struct dentry *dir, unsigned int flags)
{
    struct inode *inode = dir->d_inode;
    struct dentry *dentry;

    /* ディレクトリを読み取り用にロックする */
    inode_lock_shared(inode);  /* down_read(&inode->i_rwsem) */

    /* dcacheを再確認する（競合: その間に追加されたかもしれない） */
    dentry = __d_lookup(dir, name);
    if (!dentry) {
        /* キャッシュにない: ファイルシステムを呼び出す */
        dentry = d_alloc(dir, name);
        inode->i_op->lookup(inode, dentry, flags);
        /* dentryはこれでpositive（inodeあり）かnegativeになる */
    }

    inode_unlock_shared(inode);
    return dentry;
}
```

`i_op->lookup()`はファイルシステム（例: ext4）を呼び出し、ファイルシステムはディスクからディレクトリを読み、エントリを見つけ、そのinodeをdentryに関連付ける。

## マウントポイントの横断 {#mount-point-crossing}

解決処理がマウントポイントに達すると、マウントされたファイルシステムへ暗黙のうちに横断する:

```c
/* マウントポイントにて: 新しいファイルシステムのルートへ切り替える */
if (d_mountpoint(dentry)) {
    /* マウントをたどる: nd->path = mnt->mnt_root */
    follow_mount(&nd->path);
    /* 新しいファイルシステム内で解決を続ける */
}
```

`/proc`が`/`とは別のファイルシステムであるにもかかわらず`cd /proc/net`が透過的に動作するのは、このためである。

## シンボリックリンクの解決 {#symlink-resolution}

構成要素がシンボリックリンクの場合、`link_path_walk`はシンボリックリンクのターゲットを引数に再帰的に呼び出される:

```c
/* シンボリックリンク: リンクのターゲットを取得して解決する */
const char *link = nd->inode->i_op->get_link(nd->path.dentry,
                                               nd->inode, &delayed);
/* linkを新しいパスとしてlink_path_walkへ再帰する */
nd->depth++;
if (nd->depth > MAXSYMLINKS)  /* デフォルトは40 */
    return -ELOOP;
err = link_path_walk(link, nd);
nd->depth--;
```

## LOOKUPフラグ {#lookup-flags}

```c
LOOKUP_FOLLOW    /* 最後のシンボリックリンクをたどる */
LOOKUP_DIRECTORY /* 最後の構成要素はディレクトリでなければならない */
LOOKUP_OPEN      /* open()から呼ばれた: ファイル作成の準備をする */
LOOKUP_CREATE    /* 存在しなければ作成する */
LOOKUP_EXCL      /* 存在すれば失敗する（O_EXCL用） */
LOOKUP_REVAL     /* dentryの再検証を強制する（NFS） */
LOOKUP_RCU       /* 可能ならRCU walkモードのままでいる */
```

## /proc/self/fdと/proc/self/fdinfo {#procselffd-and-procselffdinfo}

```bash
# オープンされた各ファイルは /proc/self/fd/ にシンボリックリンクを持つ
ls -la /proc/self/fd
# lrwxrwxrwx 1 root 0 -> /dev/null
# lrwxrwxrwx 1 root 1 -> /dev/pts/0
# lrwxrwxrwx 1 root 2 -> /dev/pts/0

# リンクを読むとファイルのパスに解決される
readlink /proc/self/fd/0
# /dev/null
```

`/proc/self/fd/N`のシンボリックリンクは、ファイルシステムを経由するのではなく、プロセスのファイルディスクリプタテーブルを経由する特別な解決経路を使う。

## 参考文献 {#further-reading}

- [VFSオブジェクト](vfs-objects.md) — superblock、inode、dentry、fileとは何か
- [dentryキャッシュとinodeキャッシュ](dcache-icache.md) — パス検索がどのようにキャッシュされるか
- [ファイル操作](file-ops.md) — パスが解決された後に何が起きるか
