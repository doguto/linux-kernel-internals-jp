# POSIXメッセージキュー {#posix-message-queues}

> プロセス間での、優先度順でカーネル管理のメッセージ受け渡し

## 概要 {#overview}

POSIXメッセージキュー（`mq_open`、`mq_send`、`mq_receive`）は、次の特徴を持つメッセージ受け渡し型のIPC機構を提供する:

- **優先度順**: メッセージは優先度の高いものから順に取り出される
- **ブロッキングと非ブロッキング**: 読み手はメッセージが届くまでブロックする
- **非同期通知**: `mq_notify`は、メッセージ到着時にシグナルを配送するかスレッドを起動する
- **カーネル永続性**: キューは明示的に削除されるかシステムが再起動されるまで存続する
- **VFS統合**: キューは`/dev/mqueue`配下のファイルとして見える

```bash
# mqueue ファイルシステムをマウント（最近のシステムでは通常自動）
mount -t mqueue none /dev/mqueue
ls /dev/mqueue/   # 開かれているすべてのキューを一覧表示
```

## 基本的な使い方 {#basic-usage}

```c
#include <mqueue.h>
/* -lrt とリンクする */

/* キューを作成または開く */
struct mq_attr attr = {
    .mq_maxmsg  = 10,     /* キュー内の最大メッセージ数 */
    .mq_msgsize = 256,    /* 最大メッセージサイズ（バイト） */
};

/* 生産者: 書き込み用に開く */
mqd_t mq = mq_open("/myqueue", O_WRONLY | O_CREAT, 0644, &attr);

/* 消費者: 読み出し用に開く */
mqd_t mq = mq_open("/myqueue", O_RDONLY);

/* メッセージを送信（優先度5） */
const char *msg = "hello";
mq_send(mq, msg, strlen(msg), 5);  /* prio=5（大きいほど先に取り出される） */

/* 最も優先度の高いメッセージを受信 */
char buf[256];
unsigned int prio;
ssize_t n = mq_receive(mq, buf, sizeof(buf), &prio);
printf("received %zd bytes, priority=%u: %.*s\n", n, prio, (int)n, buf);

/* キューを削除 */
mq_close(mq);
mq_unlink("/myqueue");
```

### キューの属性 {#queue-attributes}

```c
/* キューの属性を問い合わせる */
struct mq_attr attr;
mq_getattr(mq, &attr);
printf("maxmsg=%ld msgsize=%ld curmsgs=%ld\n",
       attr.mq_maxmsg, attr.mq_msgsize, attr.mq_curmsgs);

/* 非ブロッキングモードを設定 */
attr.mq_flags = O_NONBLOCK;
mq_setattr(mq, &attr, NULL);

/* 非ブロッキング受信: 空なら EAGAIN を返す */
ssize_t n = mq_receive(mq, buf, sizeof(buf), &prio);
if (n == -1 && errno == EAGAIN)
    printf("queue empty\n");
```

## 優先度キューの挙動 {#priority-queue-behavior}

メッセージは優先度順（高いものが先）に並ぶ。同じ優先度のメッセージはFIFOである:

```c
mq_send(mq, "low",    3, 1);  /* 優先度 1 */
mq_send(mq, "high",   4, 9);  /* 優先度 9 */
mq_send(mq, "medium", 6, 5);  /* 優先度 5 */
mq_send(mq, "high2",  5, 9);  /* 優先度 9 */

/* 取り出し順: high (p9)、high2 (p9)、medium (p5)、low (p1) */
```

優先度の範囲: 0から`sysconf(_SC_MQ_PRIO_MAX) - 1`まで（最低32、通常は32768）。

## タイムアウト付き操作 {#timed-operations}

```c
#include <time.h>

/* タイムアウト付き送信（絶対時刻） */
struct timespec deadline;
clock_gettime(CLOCK_REALTIME, &deadline);
deadline.tv_sec += 5;  /* 今から5秒後 */

int ret = mq_timedsend(mq, msg, len, prio, &deadline);
if (ret == -1 && errno == ETIMEDOUT)
    printf("queue full, timed out\n");

/* タイムアウト付き受信 */
ssize_t n = mq_timedreceive(mq, buf, size, &prio, &deadline);
if (n == -1 && errno == ETIMEDOUT)
    printf("queue empty, timed out\n");
```

## 非同期通知: mq_notify {#async-notification-mq_notify}

`mq_notify`は、空のキューにメッセージが到着したときに通知を配送する:

```c
/* シグナルによる通知 */
struct sigevent sev = {
    .sigev_notify = SIGEV_SIGNAL,
    .sigev_signo  = SIGUSR1,
};
mq_notify(mq, &sev);

/* 通知に登録できるのは一度に1プロセスだけ */
/* 登録はワンショット: 通知のたびに再登録が必要 */

/* スレッドによる通知: メッセージ到着時にスレッドを起動する */
struct sigevent sev = {
    .sigev_notify            = SIGEV_THREAD,
    .sigev_notify_function   = my_handler,
    .sigev_notify_attributes = NULL,  /* デフォルトのスレッド属性 */
};
mq_notify(mq, &sev);

void my_handler(union sigval sv) {
    /* メッセージ到着時に新しいスレッドで呼ばれる */
    mqd_t mq = sv.sival_int;  /* 必要なら sigval 経由で mq を渡す */
    /* メッセージを読み、再登録する: */
    mq_notify(mq, &sev);
}
```

### mq_notify + eventfdによるepoll {#epoll-with-mq_notify-eventfd}

イベントループと統合するには:

```c
int efd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);

/* eventfd への書き込みで通知する */
struct sigevent sev = {
    .sigev_notify            = SIGEV_THREAD,
    .sigev_notify_function   = notify_handler,
    .sigev_value.sival_int   = efd,
};
mq_notify(mq, &sev);

void notify_handler(union sigval sv) {
    int efd = sv.sival_int;
    write(efd, &(uint64_t){1}, 8);
    mq_notify(mq, &sev);  /* 再登録 */
}

/* メッセージが到着すると efd に対する epoll_wait が発火する */
```

## システムの制限 {#system-limits}

```bash
# ユーザーあたりのメッセージキューの最大数
cat /proc/sys/fs/mqueue/queues_max    # デフォルト: 256

# キューあたりのデフォルトの最大メッセージ数
cat /proc/sys/fs/mqueue/msg_max       # デフォルト: 10

# デフォルトの最大メッセージサイズ
cat /proc/sys/fs/mqueue/msgsize_max   # デフォルト: 8192

# 属性を明示せずに作成されたキューに適用されるデフォルトの mq_maxmsg
# （優先度とは無関係。メッセージサイズについては msgsize_default が対応する）
cat /proc/sys/fs/mqueue/msg_default

# 開かれているすべてのキューを属性付きで表示:
ls -la /dev/mqueue/
cat /dev/mqueue/myqueue  # 表示内容: QSIZE:0 NOTIFY:0 SIGNO:0 NOTIFY_PID:0
```

## カーネルでの実装 {#kernel-implementation}

### struct mqueue_inode_info {#struct-mqueueinodeinfo}

```c
/* ipc/mqueue.c */
struct mqueue_inode_info {
    spinlock_t         lock;
    struct inode       vfs_inode;
    wait_queue_head_t  wait_q;

    /* 優先度をキーとする1本のrbツリー -- 優先度ごとのリストの配列ではない。
     * 同じ優先度のメッセージはノードごとの msg_list によって FIFO になる。 */
    struct rb_root     msg_tree;
    struct rb_node    *msg_tree_rightmost;
    struct posix_msg_tree_node *node_cache;  /* 予備ノード。GFP_ATOMIC での割り当てを避ける */
    struct mq_attr     attr;

    /* 通知 */
    struct sigevent    notify;
    struct pid        *notify_owner;
    u32                notify_self_exec_id;
    struct user_namespace *notify_user_ns;
    struct ucounts    *ucounts;       /* 作成したユーザー（アカウンティング用） */
    struct sock       *notify_sock;
    struct sk_buff    *notify_cookie;

    /* それぞれ空き領域の待機者とメッセージの待機者 */
    struct ext_wait_queue e_wait_q[2];

    unsigned long      qsize;         /* メモリ上のキューの合計サイズ */
};

/* rbツリーの各ノードは1つの異なる優先度を担当し、その優先度の
 * メッセージは FIFO 順に msg_list にぶら下がる */
struct posix_msg_tree_node {
    struct rb_node    rb_node;
    struct list_head  msg_list;
    int               priority;
};
```

### メッセージ構造体 {#message-structure}

```c
/* include/linux/msg.h */
struct msg_msg {
    struct list_head  m_list;   /* 優先度リスト上 */
    long              m_type;   /* 優先度（type として格納） */
    size_t            m_ts;     /* メッセージテキストのサイズ */
    struct msg_msgseg *next;    /* PAGE_SIZE を超えるメッセージ用 */
    void             *security; /* LSM のセキュリティラベル */
    /* メッセージデータはメモリ上でこの直後に続く */
};
```

### mq_sendのカーネルパス {#mq_send-kernel-path}

```c
/* ipc/mqueue.c -- 実際の do_mq_timedsend() を簡略化したもの。実際の
 * 関数は、投機的な node_cache の割り当てや、すでに待機している受信者への
 * パイプライン化された直接の受け渡し（存在すればツリーを完全にスキップする）
 * も扱う */
static int do_mq_timedsend(mqd_t mqdes, const char __user *u_msg_ptr,
                           size_t msg_len, unsigned int msg_prio,
                           struct timespec64 *ts)
{
    struct mqueue_inode_info *info;
    struct msg_msg *msg_ptr;

    /* メッセージデータを割り当て、ユーザー空間からコピー */
    msg_ptr = load_msg(u_msg_ptr, msg_len);
    msg_ptr->m_ts = msg_len;
    msg_ptr->m_type = msg_prio;

    spin_lock(&info->lock);

    if (info->attr.mq_curmsgs == info->attr.mq_maxmsg) {
        /* キューが満杯: ブロック（wq_sleep() により SEND 待ちリスト上で）または EAGAIN */
        if (filp->f_flags & O_NONBLOCK) {
            spin_unlock(&info->lock);
            return -EAGAIN;
        }
        return wq_sleep(info, SEND, timeout, &wait);  /* ロックは自身で解放する */
    }

    /* 待機中の受信者なし: 優先度ツリーに挿入して通知 */
    msg_insert(msg_ptr, info);
    __do_notify(info);

    spin_unlock(&info->lock);
    return 0;
}
```

### 優先度ツリーへの挿入 {#priority-tree-insertion}

```c
/* ipc/mqueue.c */
static int msg_insert(struct msg_msg *msg, struct mqueue_inode_info *info)
{
    struct rb_node **p, *parent = NULL;
    struct posix_msg_tree_node *leaf;
    bool rightmost = true;

    /* この優先度の rbツリーノードを探す（なければ作る） */
    p = &info->msg_tree.rb_node;
    while (*p) {
        parent = *p;
        leaf = rb_entry(parent, struct posix_msg_tree_node, rb_node);

        if (likely(leaf->priority == msg->m_type))
            goto insert_msg;
        else if (msg->m_type < leaf->priority) {
            p = &(*p)->rb_left;
            rightmost = false;
        } else
            p = &(*p)->rb_right;
    }
    /* この優先度の既存ノードがない: 予備の node_cache があればそれを使い、
     * なければ割り当てる（ロック下なので GFP_ATOMIC） */
    if (info->node_cache) {
        leaf = info->node_cache;
        info->node_cache = NULL;
    } else {
        leaf = kmalloc_obj(*leaf, GFP_ATOMIC);
        if (!leaf)
            return -ENOMEM;
        INIT_LIST_HEAD(&leaf->msg_list);
    }
    leaf->priority = msg->m_type;
    if (rightmost)
        info->msg_tree_rightmost = &leaf->rb_node;
    rb_link_node(&leaf->rb_node, parent, p);
    rb_insert_color(&leaf->rb_node, &info->msg_tree);

insert_msg:
    /* 同じ優先度のメッセージは、このノードの msg_list 内で FIFO */
    info->attr.mq_curmsgs++;
    info->qsize += msg->m_ts;
    list_add_tail(&msg->m_list, &leaf->msg_list);
    return 0;
}
```

## POSIX mqueueとSysVメッセージキューの比較 {#posix-mqueue-vs-sysv-message-queues}

| 機能 | POSIX mqueue | SysV msgsnd/msgrcv |
|---------|-------------|-------------------|
| API | `mq_open`/`mq_send` | `msgget`/`msgsnd` |
| 名前 | mqueue FS内の`/name` | 整数キー（IPC_PRIVATE） |
| 優先度 | あり（メッセージごとの優先度） | typeによるフィルタリング |
| 通知 | `mq_notify`（シグナル/スレッド） | なし（ポーリングのみ） |
| epoll | 可（Linuxでは本物のfd） | 不可 |
| VFS | `/dev/mqueue/` | `/proc/sysvipc/msg` |
| 永続性 | `mq_unlink`まで | 明示的に削除するまで |
| 移植性 | POSIX標準 | 歴史的（XSI） |

注: Linuxでは、POSIX mqueueのファイルディスクリプタは本物のfdであり、`select()`、`poll()`、`epoll()`で直接扱える（2.6.19以降）。

## SysVメッセージキュー（簡単に） {#sysv-message-queues-brief}

```c
#include <sys/msg.h>

/* SysV メッセージキューを作成/取得 */
int msqid = msgget(IPC_PRIVATE, IPC_CREAT | 0666);
/* またはキーで: key_t key = ftok("/some/file", 'A'); */

/* 送信 */
struct msgbuf {
    long mtype;   /* 0より大きくなければならない。フィルタリングに使われる */
    char mtext[256];
} msg = { .mtype = 1, .mtext = "hello" };
msgsnd(msqid, &msg, strlen(msg.mtext), 0);

/* 受信: mtype=0 → 先頭のメッセージ、mtype>0 → その type の最初のメッセージ */
struct msgbuf recv;
msgrcv(msqid, &recv, sizeof(recv.mtext), 0, 0);

/* 削除 */
msgctl(msqid, IPC_RMID, NULL);
```

```bash
# SysV キューを一覧表示
ipcs -q

# キューの詳細を表示
ipcs -q -i <msqid>

# すべての SysV キューを削除
ipcrm --all=msg
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [ipc/mqueue.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/ipc/mqueue.c) — POSIX mqueueファイルシステム、システムコール（`do_mq_timedsend()`、`do_mq_timedreceive()`）、`struct mqueue_inode_info`
- [ipc/msg.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/ipc/msg.c) — SysVメッセージキューの実装（`msgget`/`msgsnd`/`msgrcv`）
- [include/uapi/linux/mqueue.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/mqueue.h) — `struct mq_attr`と`MQ_PRIO_MAX`（32768）
- [include/linux/msg.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/msg.h) — `struct msg_msg`: POSIXとSysVのキューで共有されるメッセージ表現

### manページ {#man-pages}

- [`mq_overview(7)`](https://man7.org/linux/man-pages/man7/mq_overview.7.html) — POSIXメッセージキューAPI、`/dev/mqueue`、`/proc/sys/fs/mqueue`の制限の概要
- [`mq_open(3)`](https://man7.org/linux/man-pages/man3/mq_open.3.html) — キューの作成/オープン、`O_CREAT`/`O_EXCL`の意味論、`struct mq_attr`
- [`mq_send(3)`](https://man7.org/linux/man-pages/man3/mq_send.3.html) — `mq_send()`/`mq_timedsend()`
- [`mq_receive(3)`](https://man7.org/linux/man-pages/man3/mq_receive.3.html) — `mq_receive()`/`mq_timedreceive()`
- [`mq_notify(3)`](https://man7.org/linux/man-pages/man3/mq_notify.3.html) — `SIGEV_SIGNAL`/`SIGEV_THREAD`/`SIGEV_NONE`の通知モード
- [`mq_getattr(3)`](https://man7.org/linux/man-pages/man3/mq_getattr.3.html) — `mq_getattr()`/`mq_setattr()`

### 関連ページ {#related-pages}

- [SysV IPC: セマフォとメッセージキュー](sysv-semaphores.md) — 上で簡単に触れた`msgget`/`msgsnd`/`msgrcv`とカーネル内部の詳しい解説
- [eventfdとsignalfd](eventfd-signalfd.md) — 上のmq_notifyの例で使った`epoll`統合パターン
- [シグナル](signals.md) — `mq_notify`の`SIGEV_SIGNAL`モードで使われるリアルタイムシグナルの配送
- [パイプとFIFO](pipes.md) — mqueueのメッセージ指向モデルと対比されるバイトストリームIPC
- [共有メモリ、セマフォ、eventfd](shared-memory.md) — ゼロコピーの大量データIPCと、mqueueの小さく上限のあるメッセージとの対比
- [POSIXタイマーとtimerfd](../time/posix-timers.md) — `mq_notify`と同じ`sigevent`ベースの通知モデル

### LWN記事 {#lwn-articles}

- [IPC medley: message-queue peeking, io_uring, and bus1](https://lwn.net/Articles/1065490/) — Jonathan Corbet、2026年4月。非破壊的でインデックス指定のメッセージ読み出しを行う`mq_timedreceive2()`/`MQ_PEEK`拡張の提案（本ページが対象とするカーネルの時点ではまだマージされていない）を扱っている
