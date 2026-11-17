# inotifyとfanotify {#inotify-and-fanotify}

> ファイルシステムのイベント通知: ファイルやディレクトリの変更を監視する

## inotify: ファイル/ディレクトリ監視API {#inotify-filedirectory-watch-api}

`inotify`は、ファイルやディレクトリが変更されたときに通知を提供する。John McCutchanによる以前の成果を基に、Robert LoveによってLinux 2.6.13で導入された[(commit)](https://git.kernel.org/linus/0eeca28300df110bd6ed54b31193c83b87921443) [(LWN)](https://lwn.net/Articles/104343/)。これは、ディレクトリ専用でシグナルベースのインターフェースを持っていた古い**dnotify**の仕組みを置き換えるものである。ファイルマネージャ、ビルドシステム、エディタ、コンテナランタイムで使われている。

```c
#include <sys/inotify.h>

/* 1. inotifyインスタンスを作成する */
int ifd = inotify_init1(IN_NONBLOCK | IN_CLOEXEC);

/* 2. watchを追加する */
int wd = inotify_add_watch(ifd, "/etc/nginx/nginx.conf",
                            IN_MODIFY | IN_CREATE | IN_DELETE);

/* 3. イベントを読む */
char buf[4096] __attribute__((aligned(__alignof__(struct inotify_event))));
ssize_t n = read(ifd, buf, sizeof(buf));

char *ptr = buf;
while (ptr < buf + n) {
    struct inotify_event *ev = (struct inotify_event *)ptr;

    printf("wd=%d mask=%x cookie=%u name=%s\n",
           ev->wd, ev->mask, ev->cookie,
           ev->len ? ev->name : "");

    ptr += sizeof(*ev) + ev->len;
}

/* 4. watchを削除する */
inotify_rm_watch(ifd, wd);
close(ifd);
```

### イベントマスク {#event-masks}

```c
/* 監視イベント（何を監視するか） */
IN_ACCESS        /* ファイルが読まれた */
IN_MODIFY        /* ファイルが変更された */
IN_ATTRIB        /* メタデータが変更された（chmod、chown、タイムスタンプ） */
IN_CLOSE_WRITE   /* 書き込み用にオープンされたファイルがクローズされた */
IN_CLOSE_NOWRITE /* 読み取り専用でオープンされたファイルがクローズされた */
IN_OPEN          /* ファイルがオープンされた */
IN_MOVED_FROM    /* ファイルが監視対象ディレクトリから移動された */
IN_MOVED_TO      /* ファイルが監視対象ディレクトリへ移動された */
IN_CREATE        /* 監視対象ディレクトリ内でファイル/ディレクトリが作成された */
IN_DELETE        /* 監視対象ディレクトリからファイル/ディレクトリが削除された */
IN_DELETE_SELF   /* 監視対象のファイル/ディレクトリ自体が削除された */
IN_MOVE_SELF     /* 監視対象のファイル/ディレクトリ自体が移動された */

/* 便利な組み合わせ */
IN_CLOSE    /* IN_CLOSE_WRITE | IN_CLOSE_NOWRITE */
IN_MOVE     /* IN_MOVED_FROM | IN_MOVED_TO */
IN_ALL_EVENTS /* 上記のすべて */

/* maskで返されるイベントフラグ（add_watchでは使わない） */
IN_UNMOUNT   /* 監視対象を含むファイルシステムがアンマウントされた */
IN_Q_OVERFLOW /* イベントキューがオーバーフローした（wd = -1） */
IN_IGNORED   /* watchが削除された（inotify_rm_watchまたはファイル削除） */
IN_ISDIR     /* このイベントの対象はディレクトリである */

/* watchフラグ（inotify_add_watchのflags引数） */
IN_DONT_FOLLOW  /* シンボリックリンクを参照外ししない */
IN_EXCL_UNLINK  /* unlinkされたファイルについてイベントを生成しない */
IN_ONESHOT      /* 最初のイベントの後にwatchを削除する */
IN_ONLYDIR      /* パスがディレクトリの場合のみ監視する */
```

### cookieによるrenameの検出 {#rename-detection-with-cookie}

同じrenameに対する`IN_MOVED_FROM`と`IN_MOVED_TO`のイベントは、同じ`cookie`値を共有する:

```c
/* renameを検出する: /src/old → /dst/new */
int wd_src = inotify_add_watch(ifd, "/src", IN_MOVE);
int wd_dst = inotify_add_watch(ifd, "/dst", IN_MOVE);

/* イベントを読む */
/* イベント1: wd=wd_src, mask=IN_MOVED_FROM, cookie=42, name="old" */
/* イベント2: wd=wd_dst, mask=IN_MOVED_TO,   cookie=42, name="new" */

/* cookieで突き合わせてrenameを再構成する */
```

片側しか監視していない場合は、対になるイベントのない`IN_MOVED_FROM`または`IN_MOVED_TO`を受け取る。ディレクトリをまたぐrenameはこの方法で検出できる。

### inotifyの上限 {#inotify-limits}

```bash
# ユーザーごとのinotifyインスタンスの最大数
cat /proc/sys/fs/inotify/max_user_instances  # デフォルト: 128

# インスタンスごとのwatchの最大数
cat /proc/sys/fs/inotify/max_user_watches    # デフォルト: 8192

# キューの最大長（IN_Q_OVERFLOWになるまでのイベント数）
cat /proc/sys/fs/inotify/max_queued_events   # デフォルト: 16384

# 大規模なコードベース向けに増やす（例: webpack、VS Code）:
echo 524288 | sudo tee /proc/sys/fs/inotify/max_user_watches
```

### inotifyの制限事項 {#inotify-limitations}

- **再帰的な監視**: inotifyはサブディレクトリを自動的には監視しない。ディレクトリごとにwatchを追加しなければならない
- **監視者自身のイベントがない**: 監視対象のディレクトリが削除されると`IN_DELETE_SELF`を受け取るが、置き換えられたディレクトリを監視することはできない
- **ネットワークファイルシステム**: inotifyはNFSやCIFSでは動作しないことがある（サーバーのサポートに依存する）
- **オーバーフローでイベントが失われる**: `IN_Q_OVERFLOW`はイベントが破棄されたことを伝えるが、どのイベントかは伝えない

## fanotify: ファイルシステム全体のイベント通知 {#fanotify-filesystem-wide-event-notification}

`fanotify`は、Red HatのEric ParisによってLinux 2.6.36で導入され[(commit)](https://git.kernel.org/linus/ff0b16a9850e8a240ad59e10b0a1291a8fcf7cbc)、Linux 2.6.37で完全に有効化された。これはinotifyの上位集合であり、次の追加機能を持つ:

- 個々のファイル/ディレクトリだけでなく、マウントポイント全体を監視する
- 変更されたファイルのファイルディスクリプタ（読み取り用にオープン済み）を受け取る
- **パーミッションイベント**: アクセスを横取りして拒否する（アンチウイルススキャナが使用）
- ファイルシステム全体の変更（マウント上のすべてのファイル）を監視する

```c
#include <sys/fanotify.h>
#include <fcntl.h>

/* fanotifyインスタンスを作成する: 一部のフラグにはCAP_SYS_ADMINが必要 */
int fan = fanotify_init(FAN_CLOEXEC | FAN_CLASS_NOTIF,
                         O_RDONLY | O_LARGEFILE);

/* マウントポイント全体を監視する */
fanotify_mark(fan, FAN_MARK_ADD | FAN_MARK_MOUNT,
              FAN_OPEN | FAN_CLOSE_WRITE | FAN_MODIFY,
              AT_FDCWD, "/home");

/* イベントを読む */
struct fanotify_event_metadata buf[10];
ssize_t n = read(fan, buf, sizeof(buf));

for (struct fanotify_event_metadata *m = buf;
     FAN_EVENT_OK(m, n);
     m = FAN_EVENT_NEXT(m, n)) {

    if (m->fd == FAN_NOFD)
        continue;  /* オーバーフローまたはエラーのイベント */

    /* m->fdは変更されたファイルへの読み取り可能なfd */
    /* パスを得るには /proc/self/fd/N を読む: */
    char path[PATH_MAX];
    char fdpath[64];
    snprintf(fdpath, sizeof(fdpath), "/proc/self/fd/%d", m->fd);
    readlink(fdpath, path, sizeof(path));

    printf("pid=%d fd=%d mask=%llx path=%s\n",
           m->pid, m->fd, m->mask, path);

    close(m->fd);  /* 各イベントのfdは必ずクローズすること */
}
```

### fanotifyのパーミッションイベント {#fanotify-permission-events}

```c
/* パーミッションイベント: アクセスを横取りして許可/拒否する */
int fan = fanotify_init(FAN_CLOEXEC | FAN_CLASS_CONTENT,
                         O_RDONLY | O_LARGEFILE);

fanotify_mark(fan, FAN_MARK_ADD | FAN_MARK_MOUNT,
              FAN_OPEN_PERM | FAN_ACCESS_PERM,
              AT_FDCWD, "/restricted");

/* パーミッションイベントを読む */
struct fanotify_event_metadata ev;
read(fan, &ev, sizeof(ev));

/* ファイルを検査する（ev.fdは読み取り用にオープン済み） */
/* ... アクセスを許可すべきか確認する ... */

/* 応答する: 許可または拒否 */
struct fanotify_response resp = {
    .fd       = ev.fd,
    .response = FAN_ALLOW,  /* またはFAN_DENY */
};
write(fan, &resp, sizeof(resp));
close(ev.fd);
```

`FAN_CLASS_CONTENT`（`FAN_OPEN_PERM`などのため）には`CAP_SYS_ADMIN`が必要である。アンチウイルスソフトウェア（ClamAVの`clamonacc`など）は、この方法でLinux上のオンアクセススキャンを実装している。

### fanotifyのマーク種別（5.1以降） {#fanotify-mark-types-51}

```c
/* 個々のファイルシステムオブジェクトにマークする（5.1以降） */
fanotify_mark(fan, FAN_MARK_ADD | FAN_MARK_FILESYSTEM,
              FAN_CREATE | FAN_DELETE | FAN_RENAME,
              AT_FDCWD, "/");
/* ファイルシステム全体のすべての作成/削除/renameを監視する */

/* FAN_RENAMEイベントは移動元と移動先の両方の情報を含む（5.17以降） */
/* FAN_REPORT_FID: fdの代わりにファイルIDを報告する（fdのリークを避ける） */
int fan = fanotify_init(FAN_CLASS_NOTIF | FAN_REPORT_FID, 0);
```

## カーネルの実装: fsnotify {#kernel-implementation-fsnotify}

`inotify`と`fanotify`はどちらも、汎用のファイルシステム通知フレームワークである`fsnotify`の上に構築されている:

```c
/* include/linux/fsnotify_backend.h */
struct fsnotify_group {
    /* inotify/fanotifyのインスタンスごとに1つ */
    struct list_head        marks_list;
    const struct fsnotify_ops *ops;
    struct mutex            notification_mutex;
    struct list_head        notification_list; /* 保留中のイベント */
    wait_queue_head_t       notification_waitq;
    atomic_t                num_marks;
    u32                     mask;  /* 集約されたイベントマスク */
};

struct fsnotify_mark {
    /* 付与先: inode、マウントポイント、またはファイルシステム */
    __u32 mask;
    union {
        struct inode    *inode;
        struct mount    *mnt;
        struct super_block *sb;
    };
    struct fsnotify_group *group;
};
```

VFS操作（例: `vfs_write`）が発生すると:

```c
/* fs/notify/fsnotify.c */
int fsnotify(struct inode *inode, __u32 mask, const void *data, int data_type,
             const struct qstr *file_name, u32 cookie)
{
    /* 各操作の後にVFSフックから呼ばれる */

    /* このinode上のマークを走査する */
    hlist_for_each_entry(mark, &inode->i_fsnotify_marks, obj_list) {
        if (mark->mask & mask) {
            /* このグループ（inotifyまたはfanotify）にイベントを配送する */
            mark->group->ops->handle_event(mark->group, mark, inode,
                                            mask, data, ...);
        }
    }
}

/* 呼び出し元: */
/* vfs_write → file_update_time → fsnotify(IN_MODIFY) */
/* vfs_unlink → fsnotify_unlink → fsnotify(IN_DELETE) */
/* security_inode_rename → fsnotify_rename */
```

### VFSのフックポイント {#vfs-hook-points}

```c
/* fs/attr.c */
int notify_change(struct dentry *dentry, ...)
{
    /* ... */
    fsnotify_change(dentry, ia_valid);  /* → IN_ATTRIB */
}

/* fs/namei.c */
int vfs_unlink(...)
{
    /* ... */
    fsnotify_unlink(dir, dentry);  /* → IN_DELETE */
}

int vfs_rename(...)
{
    /* ... */
    fsnotify_move(old_dir, new_dir, old_name, new_name, ...);
    /* → 共通のcookieを持つIN_MOVED_FROM + IN_MOVED_TO */
}
```

## 実践的なパターン {#practical-patterns}

### ディレクトリを再帰的に監視する（ユーザー空間での回避策） {#watch-a-directory-recursively-userspace-workaround}

```c
/* inotifyには再帰モードがない — watchを手動で追加しなければならない */
void watch_recursive(int ifd, const char *path) {
    int wd = inotify_add_watch(ifd, path, IN_ALL_EVENTS);

    DIR *dir = opendir(path);
    struct dirent *de;
    while ((de = readdir(dir))) {
        if (de->d_type == DT_DIR && strcmp(de->d_name, ".") != 0
                                 && strcmp(de->d_name, "..") != 0) {
            char subpath[PATH_MAX];
            snprintf(subpath, sizeof(subpath), "%s/%s", path, de->d_name);
            watch_recursive(ifd, subpath);
        }
    }
    closedir(dir);
}

/* IN_CREATE | IN_ISDIR が届いたら: 新しいサブディレクトリに新しいwatchを追加する */
```

### inotifyとepollを併用する {#using-epoll-with-inotify}

```c
/* inotifyのfdはepollと統合できる */
int epfd = epoll_create1(0);
struct epoll_event ev = {
    .events  = EPOLLIN,
    .data.fd = ifd,
};
epoll_ctl(epfd, EPOLL_CTL_ADD, ifd, &ev);

/* イベントループ */
struct epoll_event evs[16];
int n = epoll_wait(epfd, evs, 16, -1);
for (int i = 0; i < n; i++) {
    if (evs[i].data.fd == ifd) {
        /* inotifyのイベントを読む */
    }
}
```

## inotifyとfanotifyの比較 {#inotify-vs-fanotify}

| 機能 | inotify | fanotify |
|---------|---------|---------|
| 監視の粒度 | ファイルまたはディレクトリ | ファイル、マウント、ファイルシステム |
| 再帰 | なし（ユーザー空間での回避策） | あり（FAN_MARK_MOUNT） |
| 変更されたファイルのイベントfd | なし | あり |
| パーミッションイベント | なし | あり（FAN_CLASS_CONTENT） |
| ネットワークファイルシステム | 限定的 | より良いサポート |
| 必要な権限 | なし | 一部はCAP_SYS_ADMIN |
| renameの追跡 | cookieベース | FAN_RENAME（5.17以降） |
| 典型的な用途 | ビルドシステム、エディタ | アンチウイルススキャン、監査 |

## 参考文献 {#further-reading}

- [VFSオブジェクト](vfs-objects.md) — イベントを受け取るinodeとdentry
- [ファイル操作](file-ops.md) — fsnotifyを起動するVFSフック
- [IPC: シグナル](../ipc/signals.md) — fanotifyのための`SIGEV_SIGNAL`配送
- `fs/notify/` — fsnotify、inotify、fanotifyの実装
- `include/linux/fsnotify.h` — fsnotifyフックの宣言
