# 共有メモリ、セマフォ、eventfd {#shared-memory-semaphores-and-eventfd}

> ゼロコピーIPCと軽量な同期プリミティブ

## POSIX共有メモリ {#posix-shared-memory}

POSIX共有メモリは、tmpfsを裏付けとする名前付きオブジェクトを作成し、複数のプロセスがそれを自身のアドレス空間に`mmap`できるようにする:

```c
#include <sys/mman.h>
#include <fcntl.h>

/* プロセスA: 作成して書き込む */
int fd = shm_open("/myshm",
                  O_CREAT | O_RDWR,
                  0600);
ftruncate(fd, 4096);  /* サイズを設定 */

void *ptr = mmap(NULL, 4096,
                 PROT_READ | PROT_WRITE,
                 MAP_SHARED, fd, 0);
close(fd);  /* mmap後はfdは不要 */

*(int *)ptr = 42;
munmap(ptr, 4096);

/* 名前を削除する（最後のmmapがなくなった時点でページが解放される） */
shm_unlink("/myshm");

/* プロセスB: 開いて読み出す */
int fd = shm_open("/myshm", O_RDONLY, 0);
void *ptr = mmap(NULL, 4096, PROT_READ, MAP_SHARED, fd, 0);
close(fd);
int val = *(int *)ptr;   /* val == 42 */
munmap(ptr, 4096);
```

`shm_open`は`/dev/shm/`（tmpfsマウント）に対する`open`として実装されている。共有ページはページキャッシュ内にあり、両方のプロセスのページテーブルにマップされる。

```bash
ls /dev/shm/   # 現在の共有メモリオブジェクトを確認
ipcs -m        # System V共有メモリセグメントを確認
```

## System V共有メモリ {#system-v-shared-memory}

より古いが、より広く利用可能なSHM API:

```c
#include <sys/shm.h>
#include <sys/ipc.h>

/* セグメントを作成/取得 */
key_t key = ftok("/tmp/myfile", 'A');  /* パス+IDからキーを生成 */
int shmid = shmget(key,
                   4096,               /* サイズ */
                   IPC_CREAT | 0600);  /* フラグ */

/* アタッチ（アドレス空間にマップ） */
void *ptr = shmat(shmid, NULL, 0);     /* NULL = カーネルがアドレスを選ぶ */

/* 使用する */
*(int *)ptr = 99;

/* デタッチ */
shmdt(ptr);

/* 削除（全プロセスがデタッチした時点でカーネルが解放） */
shmctl(shmid, IPC_RMID, NULL);
```

```bash
# System V共有メモリを表示
ipcs -m
# ------ Shared Memory Segments --------
# key        shmid    owner   perms  bytes  nattch  status
# 0x00000000 0        root    600    4096   0

# セグメントを削除
ipcrm -m <shmid>
```

## 同期: POSIXセマフォ {#synchronization-posix-semaphores}

共有メモリには外部の同期が必要である。POSIXセマフォは、アトミックにインクリメント/デクリメントできるカウンタを提供する:

### 名前付きセマフォ（無関係なプロセス間） {#named-semaphores-between-unrelated-processes}

```c
#include <semaphore.h>

/* プロセスA */
sem_t *sem = sem_open("/mysem",
                      O_CREAT, 0600, 1);  /* 初期値 = 1 */

sem_wait(sem);       /* P(): デクリメント、0ならブロック */
/* クリティカルセクション */
sem_post(sem);       /* V(): インクリメント、待機者を1つ起こす */

sem_close(sem);      /* fdを閉じる */
sem_unlink("/mysem"); /* 名前を削除 */

/* プロセスB */
sem_t *sem = sem_open("/mysem", 0);  /* 既存のものを開く */
sem_wait(sem);
/* ... */
sem_post(sem);
sem_close(sem);
```

### 無名セマフォ（共有メモリまたはスレッド） {#unnamed-semaphores-shared-memory-or-threads}

```c
/* 共有メモリ内に置き、複数プロセスからアクセス可能にする */
struct shared {
    sem_t sem;
    int   data;
};

struct shared *sh = mmap(NULL, sizeof(*sh),
                         PROT_READ|PROT_WRITE,
                         MAP_SHARED|MAP_ANONYMOUS, -1, 0);

sem_init(&sh->sem, 1, 1);  /* pshared=1: プロセス間で共有 */

/* プロセスA/B: */
sem_wait(&sh->sem);
sh->data++;
sem_post(&sh->sem);

/* 後始末 */
sem_destroy(&sh->sem);
munmap(sh, sizeof(*sh));
```

### カーネルでの実装 {#kernel-implementation}

POSIXセマフォは内部でfutexを使う:

```c
/* glibcのsem_wait: */
sem_wait(sem) {
    if (__atomic_sub_fetch(&sem->value, 1, __ATOMIC_ACQ_REL) >= 0)
        return;  /* 高速パス: カウンタは > 0 だった */
    /* 低速パス: 待機 */
    futex_wait(&sem->value, ...);
}

sem_post(sem) {
    if (__atomic_add_fetch(&sem->value, 1, __ATOMIC_RELEASE) <= 0)
        futex_wake(&sem->value, 1);  /* 待機者を1つ起こす */
}
```

## eventfd: 軽量なイベント通知 {#eventfd-lightweight-event-notification}

`eventfd`は64ビットカウンタを裏付けとするファイルディスクリプタを作成する。これは最も軽量な
通知機構であり、別のスレッドやコンテキストから`epoll`駆動のイベントループを起こすための
標準的な方法である:

```c
#include <sys/eventfd.h>

int efd = eventfd(0, EFD_CLOEXEC | EFD_NONBLOCK);  /* 初期値 0 */

uint64_t val = 1;
write(efd, &val, sizeof(val));      /* 通知: counter += 1 */

uint64_t count;
read(efd, &count, sizeof(count));   /* 消費: カウンタを返し、0にリセット */
```

QEMU/KVMのvirtio通知、io_uringの完了通知、
libuv/libeventのイベントループバックエンド、cgroupのイベント通知で広く使われている。
詳しくは[eventfdとsignalfd](eventfd-signalfd.md)を参照のこと。`EFD_SEMAPHORE`モード、
epoll統合パターン、実際のカーネル実装（`struct eventfd_ctx`、
`eventfd_write()`/`eventfd_read()`）を扱っている。

## memfd: 匿名のファイル裏付け共有メモリ {#memfd-anonymous-file-backed-shared-memory}

`memfd_create`はメモリ上に匿名ファイル（ファイルシステムパスなし）を作成する。名前の衝突なしに共有するのに便利である:

```c
#include <sys/mman.h>

/* 匿名ファイルを作成 */
int fd = memfd_create("mydata", MFD_CLOEXEC | MFD_ALLOW_SEALING);
ftruncate(fd, size);

void *ptr = mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);

/* シール: 以後の変更を防ぐ（読み取り専用の共有に有用） */
fcntl(fd, F_ADD_SEALS, F_SEAL_WRITE | F_SEAL_GROW | F_SEAL_SHRINK);

/* Unixソケット経由でfdを別プロセスに渡す（SCM_RIGHTS） */
send_fd_over_socket(socket_fd, fd);
/* 相手のプロセスはそのfdをmmapできる — 名前は不要 */
```

`memfd_create`の利用例:

- グラフィックス/Wayland: クライアントとコンポジタ間でのフレームバッファの共有
- dbus-broker: サイズの大きすぎるログエントリがsystemd journalソケットへの1つのデータグラムに
  収まらない場合、それをmemfdに封入（シール）し、代わりにデータグラムのペイロードとして送る。
  これは大きなフィールドに対するjournal自身の規約であり、D-Busのワイヤプロトコル自体の一部ではない
  （dbus-brokerの`src/util/misc.c`にある`misc_memfd()`で、`src/util/log.c`から使われる）

## 共有メモリ手法の比較 {#comparing-shared-memory-approaches}

| 手法 | セットアップ | fs上の名前 | fd | シーリング | 最適な用途 |
|----------|-------|-----------|-----|---------|---------|
| POSIX shm (`shm_open`) | `/dev/shm/`内のパス | あり | open後 | なし | 名前付き、永続的 |
| System V (`shmget`) | IPCキー | `ipcs`経由 | なし | なし | レガシー互換性 |
| `mmap(MAP_SHARED | MAP_ANONYMOUS)` | 匿名 | なし | なし | 親子間のみ |
| `memfd_create` | 匿名ファイル | なし | あり | あり | 動的、fd受け渡し |

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [ipc/shm.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/ipc/shm.c) — `shmget()`、`shmat()`/`shmdt()`、`shmctl()`システムコール。`newseg()`が`shmem_kernel_file_setup()`（`SHM_HUGETLB`の場合は`hugetlb_file_setup()`）を通じてセグメントの隠れた裏付けファイルを作成する
- [mm/shmem.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/shmem.c) — `shmem_file_setup()`/`shmem_kernel_file_setup()`: POSIXの`shm_open()`オブジェクトとSysVの`shmget()`セグメントの両方が共有するtmpfsの裏付け
- [mm/memfd.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memfd.c) — `memfd_create()`システムコールと`F_SEAL_*`シーリングのチェック
- [fs/eventfd.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/eventfd.c) — `eventfd_write()`/`eventfd_read()`と、`struct eventfd_ctx`のカウンタ/待ちキューの実装

### manページ {#man-pages}

- [`shmget(2)`](https://man7.org/linux/man-pages/man2/shmget.2.html) — System V共有メモリセグメントを割り当てる
- [`shmat(2)`/`shmdt(2)`](https://man7.org/linux/man-pages/man2/shmat.2.html) — System Vセグメントをアタッチ/デタッチする
- [`shmctl(2)`](https://man7.org/linux/man-pages/man2/shmctl.2.html) — `IPC_RMID`を含む制御操作
- [`shm_open(3)`](https://man7.org/linux/man-pages/man3/shm_open.3.html) — POSIX共有メモリオブジェクト。Linuxにおける`/dev/shm` tmpfs実装について記述している
- [`shm_overview(7)`](https://man7.org/linux/man-pages/man7/shm_overview.7.html) — POSIX共有メモリAPIの概要
- [`sem_overview(7)`](https://man7.org/linux/man-pages/man7/sem_overview.7.html) — POSIXセマフォ: 名前付きと無名の違い、名前付きセマフォの`/dev/shm`での永続性
- [`memfd_create(2)`](https://man7.org/linux/man-pages/man2/memfd_create.2.html) — シール可能な匿名メモリファイル
- [`eventfd(2)`](https://man7.org/linux/man-pages/man2/eventfd.2.html) — `EFD_SEMAPHORE`を含む、カウンタ裏付けの通知fd

### 関連ページ {#related-pages}

- [プロセスのアドレス空間](../mm/mmap.md) — VMAと、POSIX shmと匿名共有マッピングの両方が依拠する`MAP_SHARED`フラグ
- [ファイル裏付けmmapとページフォールト](../mm/file-mmap.md) — `MAP_PRIVATE`と`MAP_SHARED`のフォールト処理の詳細
- [futexの内部構造](../locking/futex.md) — 競合のない高速パスのアトミック操作だけでは足りないとき、`sem_wait()`/`sem_post()`がどのようにカーネルにフォールバックするか
- [SysV IPC: セマフォとメッセージキュー](sysv-semaphores.md) — `semget`/`semop`、`SEM_UNDO`、SysV IPCのリーク問題
- [eventfdとsignalfd](eventfd-signalfd.md) — `struct eventfd_ctx`、epoll統合、signalfdを詳しく扱う
- [Unixドメインソケット](unix-sockets.md) — `SCM_RIGHTS`: 任意のオープン済みファイルディスクリプタ（`memfd_create()`のfdを含む）をプロセス間で渡すための汎用機構

### LWN記事 {#lwn-articles}

- [Sealed files](https://lwn.net/Articles/593918/) (2014) — 本ページのmemfdの節で説明した`memfd_create()`と`F_SEAL_*`シーリング機構に関するJonathan Corbetの記事

### 外部資料 {#external}

- [tmpfs — The Linux Kernel documentation](https://docs.kernel.org/filesystems/tmpfs.html) — POSIXの`shm_open()`/`/dev/shm`とSysVの`shmget()`セグメントの両方がtmpfsを裏付けとすることを確認できる（SysVは内部の不可視マウント経由）
