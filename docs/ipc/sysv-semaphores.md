# SysV IPC: セマフォとメッセージキュー {#sysv-ipc-semaphores-and-message-queues}

> 元祖UnixのIPCプリミティブ: セマフォ、メッセージキュー、そしてそのカーネル内部

SysV IPCはLinuxで最も古いIPC層であり、System V Unixから受け継がれたものである。POSIX IPCより前から存在し、APIの使いにくさで悪名高いが、多くの商用アプリケーションやデータベース（PostgreSQL、Oracle）がいまだに依存しているため、今も広く使われている。その内部構造と障害モードを理解することは、本番システムをデバッグするすべての人にとって不可欠である。

## SysVセマフォ {#sysv-semaphores}

### 作成とオープン: semget() {#creating-and-opening-semget}

```c
#include <sys/sem.h>

/* IPC_PRIVATE: このプロセスツリーに限定（キーは無視される） */
int semid = semget(IPC_PRIVATE, 1, IPC_CREAT | 0600);

/* ファイル + プロジェクトIDから導出した共有キー */
key_t key = ftok("/var/run/myapp.pid", 'A');
int semid = semget(key, 3, IPC_CREAT | IPC_EXCL | 0600);
/* nsems=3: 3つのセマフォからなるセットを作成 */
```

`semget()`はセマフォセット識別子（`semid`）を返す。セットは`nsems`個の個別のセマフォを保持し、それぞれ独立に指定できる。セット内のすべてのセマフォは、1回の`semop()`呼び出しによってアトミックに操作される。

### 操作: semop() {#operations-semop}

```c
#include <sys/sem.h>

struct sembuf {
    unsigned short sem_num;  /* セット内のセマフォのインデックス（0..nsems-1） */
    short          sem_op;   /* 操作: >0 インクリメント、<0 デクリメント、0 ゼロ待ち */
    short          sem_flg;  /* IPC_NOWAIT | SEM_UNDO */
};

/* P（down/wait）: 1減らす。値が負になるならブロック */
struct sembuf lock_op   = { .sem_num = 0, .sem_op = -1, .sem_flg = SEM_UNDO };
/* V（up/signal）: 1増やし、待機者を起こす */
struct sembuf unlock_op = { .sem_num = 0, .sem_op = +1, .sem_flg = SEM_UNDO };
/* ゼロ待ち: セマフォが0になるまでブロック */
struct sembuf zero_op   = { .sem_num = 0, .sem_op =  0, .sem_flg = 0 };

semop(semid, &lock_op, 1);
/* ... クリティカルセクション ... */
semop(semid, &unlock_op, 1);
```

`semop()`は`nsops`個の操作すべてを**アトミックに**適用する。すべて成功するか、どれも適用されないかのどちらかである。これがPOSIXセマフォに対する重要な利点である。複数のセマフォをアトミックにデクリメントできるため、デッドロックのないリソース割り当て（Dijkstraの「銀行家のアルゴリズム」方式）が可能になる。

### SEM_UNDO: 終了時の自動クリーンアップ {#sem_undo-automatic-cleanup-on-exit}

`SEM_UNDO`は堅牢性にとって極めて重要である。プロセスが（正常に、またはシグナルによって）終了すると、カーネルはそのプロセスが行ったすべての`SEM_UNDO`操作を自動的に取り消す:

```c
/* ロック時には常に SEM_UNDO を使う */
struct sembuf op = { 0, -1, SEM_UNDO };
semop(semid, &op, 1);
/* このプロセスがクラッシュすると、セマフォは自動的にインクリメントして戻される */
```

カーネルはプロセスごとのundoリストを`struct sem_undo`（`ipc/sem.c`で定義）で追跡する。`do_exit()`時に`exit_sem()`がundoリストを走査し、保留中の各調整を取り消す。

**`SEM_UNDO`なしの場合**: セマフォを保持したまま死んだプロセスは、それを永続的にデクリメントされたままにする。以後のすべての待機者は永遠にブロックする。

### semctl(): 制御と検査 {#semctl-control-and-inspection}

```c
#include <sys/sem.h>

union semun {
    int              val;    /* SETVAL */
    struct semid_ds *buf;    /* IPC_STAT / IPC_SET */
    unsigned short  *array;  /* GETALL / SETALL */
};

/* セマフォ0を値1に初期化（mutex） */
union semun arg = { .val = 1 };
semctl(semid, 0, SETVAL, arg);

/* 現在値を読む */
int val = semctl(semid, 0, GETVAL);

/* セット内のすべての値を読む */
unsigned short vals[3];
arg.array = vals;
semctl(semid, 0, GETALL, arg);

/* キューの統計を取得 */
struct semid_ds ds;
arg.buf = &ds;
semctl(semid, 0, IPC_STAT, arg);
printf("last semop: %ld, nsems: %lu\n",
       (long)ds.sem_otime, (unsigned long)ds.sem_nsems);

/* セマフォセットを削除 */
semctl(semid, 0, IPC_RMID);
```

### カーネル内部 {#kernel-internals}

```c
/* ipc/sem.c（カーネル内部の構造体。ヘッダーにはエクスポートされない） */
struct sem_array {
    struct kern_ipc_perm  sem_perm;    /* パーミッション + IPC ID */
    time64_t              sem_ctime;   /* 最後の semctl() の時刻 */
    struct list_head      pending_alter;  /* セットを変更する保留中の sops */
    struct list_head      pending_const;  /* 変更しない保留中の sops（ゼロ待ち） */
    struct list_head      list_id;     /* この配列の sem_undo リストの先頭 */
    int                   sem_nsems;   /* セット内のセマフォ数 */
    int                   complex_count; /* 保留中の複数セマフォ操作の数 */
    unsigned int          use_global_lock; /* グローバルな sem_lock を使うかセマフォごとのロックを使うか */
    struct sem             sems[];     /* セマフォのフレキシブル配列 */
};

struct sem {
    int          semval;        /* 現在値 */
    struct pid  *sempid;        /* 最後の semop() の pid（名前空間を考慮） */
    spinlock_t   lock;          /* セマフォごとのロック（単純な操作用） */
    struct list_head pending_alter;  /* 保留中の単純な変更操作 */
    struct list_head pending_const;  /* 保留中の単純な非変更操作 */
    time64_t     sem_otime;     /* 最後の semop() の時刻 -- sem_array ではなく
                                  * セマフォごとに持つ。並行操作時の
                                  * キャッシュラインのスラッシングを避けるため複製されている */
};
```

待機中の操作は`struct sem_queue`（`ipc/sem.c`で定義された内部構造体で、ヘッダーにはエクスポートされない）で追跡される。`semop()`がすぐに完了できない場合、`sem_queue`エントリをキューに入れてスリープする。`ipc/sem.c`の`do_semtimedop()`（`semop()`と`semtimedop()`の両システムコールの背後にある共通パス）は、保留キューを走査し、セマフォのインクリメントによって操作が満たせるようになったスリープ中のタスクを起こす。

## SysVメッセージキュー {#sysv-message-queues}

### 作成とオープン: msgget() {#creating-and-opening-msgget}

```c
#include <sys/msg.h>

key_t key = ftok("/var/run/myapp", 'M');
int msqid = msgget(key, IPC_CREAT | 0600);
```

### 送信: msgsnd() {#sending-msgsnd}

```c
struct msgbuf {
    long mtype;    /* メッセージタイプ: 0より大きくなければならない */
    char mtext[256]; /* メッセージデータ */
};

struct msgbuf msg = {
    .mtype = 1,
};
snprintf(msg.mtext, sizeof(msg.mtext), "hello");

/* msgsz = データサイズのみ（mtype は含まない） */
msgsnd(msqid, &msg, strlen(msg.mtext) + 1, 0);
/* IPC_NOWAIT フラグ: キューが満杯ならブロックせずに EAGAIN を返す */
msgsnd(msqid, &msg, sizeof(msg.mtext), IPC_NOWAIT);
```

### 受信: msgrcv() {#receiving-msgrcv}

`msgtyp`引数は、どのメッセージを取り出すかを制御する:

```c
long msgtyp;
struct msgbuf recv_buf;

/* msgtyp == 0: タイプに関係なく最も古いメッセージを受信 */
msgtyp = 0;
msgrcv(msqid, &recv_buf, sizeof(recv_buf.mtext), msgtyp, 0);

/* msgtyp > 0: ちょうどそのタイプの最も古いメッセージを受信 */
msgtyp = 2;
msgrcv(msqid, &recv_buf, sizeof(recv_buf.mtext), msgtyp, 0);

/* msgtyp < 0: タイプが |msgtyp| 以下の最も古いメッセージを受信 */
msgtyp = -5;  /* 5以下で最小のタイプ */
msgrcv(msqid, &recv_buf, sizeof(recv_buf.mtext), msgtyp, 0);
```

このタイプによるフィルタリングにより、1つのキュー上で複数のメッセージクラスを扱え、優先度を模擬できる。

### msgctl(): 制御と検査 {#msgctl-control-and-inspection}

```c
#include <sys/msg.h>

struct msqid_ds ds;
msgctl(msqid, IPC_STAT, &ds);
printf("messages in queue: %lu\n",   (unsigned long)ds.msg_qnum);
printf("bytes in queue:    %lu\n",   (unsigned long)ds.msg_cbytes);
printf("queue byte limit:  %lu\n",   (unsigned long)ds.msg_qbytes);
printf("last send:  %ld\n",          (long)ds.msg_stime);
printf("last recv:  %ld\n",          (long)ds.msg_rtime);

/* バイト数の上限を増やす（CAP_SYS_RESOURCE または所有者であることが必要） */
ds.msg_qbytes = 1048576;
msgctl(msqid, IPC_SET, &ds);

/* キューを削除 */
msgctl(msqid, IPC_RMID, NULL);
```

### カーネル内部 {#kernel-internals_1}

```c
/* ipc/msg.c */
struct msg_queue {
    struct kern_ipc_perm q_perm;   /* パーミッション + IPC ID */
    time64_t             q_stime;  /* 最後の msgsnd() の時刻 */
    time64_t             q_rtime;  /* 最後の msgrcv() の時刻 */
    time64_t             q_ctime;  /* 最後の msgctl() の時刻 */
    unsigned long        q_cbytes; /* キュー内の現在のバイト数 */
    unsigned long        q_qnum;   /* 現在のメッセージ数 */
    unsigned long        q_qbytes; /* キュー内の最大バイト数 */
    struct pid          *q_lspid;  /* 最後の msgsnd() の pid */
    struct pid          *q_lrpid;  /* 最後の msgrcv() の pid */
    struct list_head     q_messages; /* struct msg_msg のリスト */
    struct list_head     q_receivers; /* ブロック中の受信者 */
    struct list_head     q_senders;   /* ブロック中の送信者 */
};
```

メッセージは`q_messages`上の`struct msg_msg`エントリの連結リストとして格納される。キューが満杯（`q_cbytes >= q_qbytes`）のとき、送信プロセスは`q_senders`上でスリープする。空のとき、受信者は`q_receivers`上でスリープする。送信または受信が成功すると、反対側の待ちキューが起こされる。

## SysV IPCの観察 {#observing-sysv-ipc}

### /proc/sysvipc {#procsysvipc}

```bash
# すべてのセマフォセットを一覧表示
cat /proc/sysvipc/sem
# key      semid perms      nsems   uid   gid  cuid  cgid      otime      ctime

# すべてのメッセージキューを一覧表示
cat /proc/sysvipc/msg
# key      msqid perms      cbytes  qnum    lspid   lrpid   uid   gid  cuid  cgid      stime      rtime      ctime
```

### ipcs / ipcrm {#ipcs-ipcrm}

```bash
# すべての SysV IPC リソースを表示
ipcs

# セマフォのみを詳細表示
ipcs -s -v

# メッセージキューのみを表示
ipcs -q

# 特定のセマフォセットを削除
ipcrm -s <semid>

# 特定のメッセージキューを削除
ipcrm -q <msqid>

# 現在のユーザーが所有するすべてのセマフォセットを削除（最終手段）
ipcs -s | awk 'NR>2 && $1 != "" {print $2}' | xargs -I{} ipcrm -s {}
```

### システムの制限（sysctl） {#system-limits-sysctls}

```bash
# システム全体の最大セマフォセット数
sysctl kernel.sem
# kernel.sem = 32000 1024000000 500 32000
#              SEMMSL SEMMNS      SEMOPM SEMMNI
#   SEMMSL: セットあたりの最大セマフォ数
#   SEMMNS: システム全体の最大セマフォ数
#   SEMOPM: semop() 1回あたりの最大操作数
#   SEMMNI: 最大セマフォセット数（これに達すると ENOSPC になる）

# メッセージキューの制限
sysctl kernel.msgmni    # 最大メッセージキュー数
# kernel.msgmni = 32000
sysctl kernel.msgmax    # 最大メッセージサイズ（バイト）
# kernel.msgmax = 8192
sysctl kernel.msgmnb    # キューあたりの最大バイト数（msg_qbytes のデフォルト）
# kernel.msgmnb = 16384
```

## 比較: セマフォ機構 {#comparison-semaphore-mechanisms}

| 機構 | API | ブロッキング | 複数セマフォのアトミック操作 | 自動クリーンアップ | epoll対応 |
|-----------|-----|---------|------------------|--------------|-----------------|
| SysVセマフォ | `semop()` | あり | **あり** | `SEM_UNDO`のみ | 不可 |
| POSIX名前付きセマフォ | `sem_wait()` | あり | なし | `sem_unlink()` | 不可 |
| POSIX無名セマフォ | `sem_wait()` | あり | なし | 破棄時 | 不可 |
| `futex` | `futex()` | あり | なし | プロセス終了時（robust） | 不可 |
| `eventfd` | `read()`/`write()` | あり | なし | fdクローズ時 | **可** |

## SysV IPCのリーク問題 {#the-sysv-ipc-leak-problem}

SysV IPCオブジェクト（セマフォセットとメッセージキュー）は、**明示的に削除される（`IPC_RMID`）か、システムが再起動されるまで存続する**。ファイルディスクリプタとは異なり、どのプロセスの寿命にも結び付いていない。後始末をせずにクラッシュしたプロセスは、孤立したオブジェクトを残していく。

これは2種類の障害を引き起こす:

1. **リソース枯渇**: `SEMMNI`（デフォルトは32000、`kernel.sem`で調整可能）がセマフォセット数を制限する。上限に達すると、`semget()`は`ENOSPC`を返す。新しいサービスインスタンスが起動できなくなる。

2. **古い状態**: 値が0でない孤立したセマフォによって、次のサービスインスタンスが`semop()`で無期限にブロックすることがある。

### 診断 {#diagnosis}

```bash
# 実行中のどのプロセスにも結び付いていないセマフォセットを探す
ipcs -s -v | grep -v "^--" | awk 'NR>2 {
    cmd = "kill -0 " $5 " 2>/dev/null"; # $5 = cpid (creator pid)
    if (system(cmd) != 0)
        print "ORPHANED semid=" $2 " key=" $1 " creator_pid=" $5
}'

# 所有者ごとに数える
ipcs -s | awk 'NR>2 {print $5}' | sort | uniq -c | sort -rn
```

### 予防 {#prevention}

- シグナルハンドラを含むすべての終了パスで、必ず`semctl(semid, 0, IPC_RMID)`を呼ぶ
- あるいは、POSIX名前付きセマフォ（`sem_open()` / `sem_unlink()`）を優先する。名前は`sem_unlink()`で削除され、セマフォ自体は最後の`sem_t`が閉じられた時点で破棄される。永続的なカーネルオブジェクトは残らない
- 純粋なプロセス内の同期には、`pthread_mutex`や`futex`を直接使う

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [ipc/sem.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/ipc/sem.c) — `struct sem_array`、`struct sem`、`struct sem_undo`。`do_semtimedop()`（`semop()`を支える操作処理パス）と`exit_sem()`
- [ipc/msg.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/ipc/msg.c) — `struct msg_queue`と、`msgget()`/`msgsnd()`/`msgrcv()`/`msgctl()`の実装
- [include/uapi/linux/sem.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/sem.h) — `struct sembuf`、`union semun`、`SEMMSL`/`SEMMNS`/`SEMOPM`/`SEMMNI`のデフォルト値
- [include/uapi/linux/msg.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/msg.h) — `struct msgbuf`、`struct msqid_ds`、`MSGMNI`/`MSGMAX`/`MSGMNB`のデフォルト値

### manページ {#man-pages}

- [`semget(2)`](https://man7.org/linux/man-pages/man2/semget.2.html) — セマフォセットの作成/オープン、`IPC_PRIVATE`、`nsems`の制限
- [`semop(2)`](https://man7.org/linux/man-pages/man2/semop.2.html) — `semop()`/`semtimedop()`、`SEM_UNDO`、複数操作のアトミックな意味論
- [`semctl(2)`](https://man7.org/linux/man-pages/man2/semctl.2.html) — `union semun`、`GETALL`/`SETALL`/`SETVAL`/`IPC_RMID`
- [`msgget(2)`](https://man7.org/linux/man-pages/man2/msgget.2.html) — メッセージキューの作成/オープン
- [`msgop(2)`](https://man7.org/linux/man-pages/man2/msgop.2.html) — `msgtyp`によるフィルタリング規則を含む`msgsnd()`/`msgrcv()`
- [`msgctl(2)`](https://man7.org/linux/man-pages/man2/msgctl.2.html) — メッセージキューに対する`IPC_STAT`/`IPC_SET`/`IPC_RMID`
- [`ftok(3)`](https://man7.org/linux/man-pages/man3/ftok.3.html) — パスとプロジェクトIDからIPCキーを導出する
- [`ipcs(1)`](https://man7.org/linux/man-pages/man1/ipcs.1.html) — 有効なSysV IPCオブジェクトを調べる
- [`ipcrm(1)`](https://man7.org/linux/man-pages/man1/ipcrm.1.html) — キーまたは識別子でSysV IPCオブジェクトを削除する
- [`proc_sysvipc(5)`](https://man7.org/linux/man-pages/man5/proc_sysvipc.5.html) — `/proc/sysvipc/{msg,sem,shm}`ファイル

### 関連ページ {#related-pages}

- [共有メモリ、セマフォ、eventfd](shared-memory.md) — SysVの`shmget`/`shmat`とPOSIXセマフォ。ここで扱うSysVセマフォとの対比
- [POSIXメッセージキュー](mqueue.md) — 本ページのメッセージキューの節に対応するPOSIX版。優先度順の比較を含む
- [eventfdとsignalfd](eventfd-signalfd.md) — `semop()`でブロックする代わりとなる、poll可能なfdベースの手段
- [IPC事件簿](war-stories.md) — 上述の`SEMMNI`枯渇という障害モードをたどる、実際の孤立セマフォセット事例

### 外部資料 {#external}

- [Kernel docs: sysctl/kernel.rst — msgmax, msgmnb, and msgmni](https://docs.kernel.org/admin-guide/sysctl/kernel.html#msgmax-msgmnb-and-msgmni) — 上で示したメッセージキュー制限の現在のデフォルト値に関する公式ドキュメント
