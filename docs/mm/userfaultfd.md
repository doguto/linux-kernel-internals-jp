# userfaultfd: ユーザー空間でのページフォールト処理 {#userfaultfd-userspace-page-fault-handling}

> ユーザー空間にページフォールトを処理させる — QEMUのライブマイグレーションやCRIUを支える仕組み

## userfaultfdはどんな問題を解決するのか {#what-problem-does-userfaultfd-solve}

通常、ページフォールトはカーネルが処理する。ページが存在しなければ、匿名ページの割り当て、スワップイン、ファイルの読み込みが行われる。しかし一部のアプリケーションは、自前のストレージからページを提供したり、ユーザー空間でコピーオンライトを実装したりするために、ページフォールトを**横取り**する必要がある。

用途:

- **QEMUのライブマイグレーション**: ゲストがアクセスしたタイミングで、ゲストRAMのページをその場で受信する
- **CRIU**（Checkpoint/Restore In Userspace）: プロセスのメモリを遅延的に復元する
- **ハイパーバイザ**: ゲストメモリのデマンドページングをユーザー空間で実装する
- **ガベージコレクタ**: mprotectのオーバーヘッドなしに世代別GCのバリアを実装する
- **サンドボックス**: テストのために決定的なメモリ内容を提供する

## APIの概要 {#api-overview}

```c
#include <sys/ioctl.h>
#include <linux/userfaultfd.h>
#include <poll.h>

/* 1. userfaultfdを作成 */
int uf_fd = syscall(SYS_userfaultfd, O_CLOEXEC | O_NONBLOCK);

/* 2. APIバージョンを確認 */
struct uffdio_api api = {
    .api   = UFFD_API,
    .features = UFFD_FEATURE_PAGEFAULT_FLAG_WP |
                UFFD_FEATURE_MINOR_FAULTS,
};
ioctl(uf_fd, UFFDIO_API, &api);
/* api.featuresにはサポートされている機能が入る */
/* api.ioctlsには利用可能なioctlが入る */

/* 3. 横取りするメモリ範囲を登録 */
struct uffdio_register reg = {
    .range.start = (unsigned long)addr,
    .range.len   = size,
    .mode        = UFFDIO_REGISTER_MODE_MISSING,  /* 存在しないページを横取り */
};
ioctl(uf_fd, UFFDIO_REGISTER, &reg);

/* 4. スレッドでフォールトをpollする */
struct pollfd pfd = { .fd = uf_fd, .events = POLLIN };
while (1) {
    poll(&pfd, 1, -1);

    struct uffd_msg msg;
    read(uf_fd, &msg, sizeof(msg));

    if (msg.event == UFFD_EVENT_PAGEFAULT) {
        unsigned long fault_addr = msg.arg.pagefault.address;
        unsigned long flags      = msg.arg.pagefault.flags;
        /* UFFD_PAGEFAULT_FLAG_WRITE: 書き込みフォールト */
        /* UFFD_PAGEFAULT_FLAG_WP:    書き込み保護フォールト */

        /* 解決: ゼロページを提供する */
        struct uffdio_zeropage zp = {
            .range.start = fault_addr & ~(PAGE_SIZE - 1),
            .range.len   = PAGE_SIZE,
        };
        ioctl(uf_fd, UFFDIO_ZEROPAGE, &zp);
    }
}
```

## フォールトの解決方法 {#fault-resolution-methods}

### UFFDIO_COPY — データ入りのページを設定する {#uffdio_copy-install-a-page-with-data}

```c
static char page_data[PAGE_SIZE];
/* page_dataを望む内容で埋める */

struct uffdio_copy copy = {
    .dst  = fault_addr & ~(PAGE_SIZE - 1),
    .src  = (unsigned long)page_data,
    .len  = PAGE_SIZE,
    .mode = 0,  /* 後で起こすならUFFDIO_COPY_MODE_DONTWAKE */
};
ioctl(uf_fd, UFFDIO_COPY, &copy);
/* フォールトしたスレッドを起こす */
```

### UFFDIO_ZEROPAGE — ゼロページを設定する {#uffdio_zeropage-install-a-zero-page}

```c
struct uffdio_zeropage zp = {
    .range.start = fault_addr & ~(PAGE_SIZE - 1),
    .range.len   = PAGE_SIZE,
    .mode = 0,
};
ioctl(uf_fd, UFFDIO_ZEROPAGE, &zp);
```

### UFFDIO_CONTINUE — マイナーフォールトを解決する（共有マッピング用） {#uffdio_continue-resolve-minor-fault-for-shared-mappings}

```c
/* マイナーフォールト: ページはページキャッシュにあるがPTEにない */
struct uffdio_continue cont = {
    .range.start = fault_addr & ~(PAGE_SIZE - 1),
    .range.len   = PAGE_SIZE,
    .mode = 0,
};
ioctl(uf_fd, UFFDIO_CONTINUE, &cont);
```

### UFFDIO_WRITEPROTECT — ページを書き込み保護して書き込みを捕捉する {#uffdio_writeprotect-write-protect-pages-to-catch-writes}

```c
/* 領域を書き込み保護する (書き込みでWPフォールトが発生) */
struct uffdio_writeprotect wp = {
    .range.start = (unsigned long)addr,
    .range.len   = size,
    .mode = UFFDIO_WRITEPROTECT_MODE_WP,     /* WPを有効化 */
};
ioctl(uf_fd, UFFDIO_WRITEPROTECT, &wp);

/* 後でWPフォールトが届いたら (UFFD_PAGEFAULT_FLAG_WPがセット): */
wp.mode = 0;  /* WPを解除 — 以降の書き込みを許可 */
ioctl(uf_fd, UFFDIO_WRITEPROTECT, &wp);
```

## 登録モード {#registration-modes}

```c
/* UFFDIO_REGISTER_MODE_MISSING: 存在しないページのフォールトを横取り */
reg.mode = UFFDIO_REGISTER_MODE_MISSING;

/* UFFDIO_REGISTER_MODE_WP: 書き込み保護フォールトを横取り */
reg.mode = UFFDIO_REGISTER_MODE_WP;

/* 両方 (Linux 5.7+): */
reg.mode = UFFDIO_REGISTER_MODE_MISSING | UFFDIO_REGISTER_MODE_WP;

/* UFFDIO_REGISTER_MODE_MINOR (Linux 5.13+): shmem/hugetlbのマイナーフォールト用 */
reg.mode = UFFDIO_REGISTER_MODE_MINOR;
```

## 完全な例: 遅延ページプロバイダ {#complete-example-lazy-page-provider}

```c
#include <sys/mman.h>
#include <sys/ioctl.h>
#include <sys/syscall.h>
#include <linux/userfaultfd.h>
#include <poll.h>
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define SIZE    (64 * 1024 * 1024)  /* 64MB */

static int uf_fd;
static void *region;

static void *fault_handler_thread(void *arg)
{
    struct pollfd pfd = { .fd = uf_fd, .events = POLLIN };
    static char page[4096];

    while (1) {
        poll(&pfd, 1, -1);

        struct uffd_msg msg;
        if (read(uf_fd, &msg, sizeof(msg)) != sizeof(msg))
            break;

        if (msg.event != UFFD_EVENT_PAGEFAULT)
            continue;

        unsigned long page_addr = msg.arg.pagefault.address & ~0xFFFUL;
        unsigned long page_index = (page_addr - (unsigned long)region) / 4096;

        /* ページの内容をオンデマンドで生成 */
        memset(page, 0, sizeof(page));
        snprintf(page, 64, "Page %lu: generated on demand", page_index);

        struct uffdio_copy copy = {
            .dst = page_addr,
            .src = (unsigned long)page,
            .len = PAGE_SIZE,
        };
        ioctl(uf_fd, UFFDIO_COPY, &copy);
    }
    return NULL;
}

int main(void)
{
    /* 大きな匿名マッピングを作成 (まだ物理ページはない) */
    region = mmap(NULL, SIZE, PROT_READ | PROT_WRITE,
                  MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

    /* userfaultfdを作成 */
    uf_fd = syscall(SYS_userfaultfd, O_CLOEXEC | O_NONBLOCK);

    struct uffdio_api api = { .api = UFFD_API };
    ioctl(uf_fd, UFFDIO_API, &api);

    /* 領域全体を登録 */
    struct uffdio_register reg = {
        .range = { .start = (unsigned long)region, .len = SIZE },
        .mode  = UFFDIO_REGISTER_MODE_MISSING,
    };
    ioctl(uf_fd, UFFDIO_REGISTER, &reg);

    /* フォールトハンドラスレッドを起動 */
    pthread_t thr;
    pthread_create(&thr, NULL, fault_handler_thread, NULL);

    /* メモリにアクセス — 各アクセスがフォールトを起こし、自前のスレッドが処理する */
    for (int i = 0; i < SIZE / 4096; i += 1024) {
        char *p = (char *)region + i * 4096;
        printf("Reading page %d: '%s'\n", i, p);
    }

    return 0;
}
```

## userfaultfdを使ったQEMUのライブマイグレーション {#qemu-live-migration-with-userfaultfd}

QEMUは**ポストコピー方式のライブマイグレーション**にuserfaultfdを使う。これは、すべてのメモリが届く前に移行先のVMを起動する方式である。

```
Source VM                              Destination VM
─────────────────────────────────────────────────────
                                       QEMU registers guest RAM with uffd
Transfer CPU state + a few pages ────►
                                       Start guest execution
                                       ↓
                                       Guest accesses page not yet received
                                       → userfaultfd fault event
                                       ↓
                                       Fault handler requests page from source
                               ◄────── request page N
Transfer page N ──────────────────────►
                                       uffd_copy resolves fault
                                       Guest continues
                                       (background: transfer remaining pages)
```

主な利点は、移行先のVMがすぐに動き始め、オンデマンドで取得する必要があるのはホットなページだけで済むことである。

## カーネルの実装 {#kernel-implementation}

### フォールトの横取り {#fault-interception}

```c
/* mm/userfaultfd.c */

/*
 * ページが存在しないときにhandle_mm_fault()から呼ばれる。
 * userfaultfdが処理する場合はVM_FAULT_RETRYを返す。
 */
vm_fault_t handle_userfault(struct vm_fault *vmf, unsigned long reason)
{
    struct mm_struct *mm = vmf->vma->vm_mm;
    struct userfaultfd_ctx *ctx;
    struct uffd_msg msg = {};

    ctx = vmf->vma->vm_userfaultfd_ctx.ctx;
    if (!ctx)
        return 0;  /* このVMAにはuffdが登録されていない */

    /* フォールトメッセージを組み立てる */
    msg.event = UFFD_EVENT_PAGEFAULT;
    msg.arg.pagefault.flags = 0;
    if (reason & VM_UFFD_WP)
        msg.arg.pagefault.flags |= UFFD_PAGEFAULT_FLAG_WP;
    if (vmf->flags & FAULT_FLAG_WRITE)
        msg.arg.pagefault.flags |= UFFD_PAGEFAULT_FLAG_WRITE;
    msg.arg.pagefault.address = vmf->address;

    /* フォールトメッセージをキューに入れる — uffdファイルが読み取り可能になる */
    userfaultfd_event_wait_completion(ctx, &ewq);

    /*
     * フォールトハンドラスレッドがUFFDIO_COPYまたはUFFDIO_ZEROPAGEで
     * フォールトを解決した。ここでフォールトを再試行する — ページは存在する。
     */
    return VM_FAULT_RETRY;
}
```

### struct userfaultfd_ctx {#struct-userfaultfd_ctx}

```c
struct userfaultfd_ctx {
    wait_queue_head_t   fault_pending_wqh; /* 読み取りを待つフォールト */
    wait_queue_head_t   fault_wqh;         /* 解決を待つフォールト */
    wait_queue_head_t   event_wqh;         /* フォールト以外のイベント (fork, remap) */
    wait_queue_head_t   fd_wqh;            /* fdのpoll用待ちキュー */

    seqcount_spinlock_t refile_seq;
    unsigned long       features;
    bool                released;
    atomic_t            mmap_changing;

    struct mm_struct    *mm;
};
```

## ページフォールト以外のイベント {#events-beyond-page-faults}

userfaultfdは、監視対象の領域が変化したときにフォールト以外のイベントも配送できる。

```c
/* フォールト以外のイベントを有効にする機能: */
api.features = UFFD_FEATURE_EVENT_FORK        |  /* fork()がVMAを複製 */
               UFFD_FEATURE_EVENT_REMAP       |  /* mremap() */
               UFFD_FEATURE_EVENT_REMOVE      |  /* madvise(MADV_REMOVE) */
               UFFD_FEATURE_EVENT_UNMAP;         /* munmap() */

/* ハンドラ内で: */
switch (msg.event) {
case UFFD_EVENT_FORK:
    /* msg.arg.fork.ufd = 子プロセス用の新しいuffd */
    break;
case UFFD_EVENT_REMAP:
    /* msg.arg.remap.from/to/len */
    break;
case UFFD_EVENT_REMOVE:
case UFFD_EVENT_UNMAP:
    /* msg.arg.remove.start/end */
    break;
}
```

## 性能面の考慮事項 {#performance-considerations}

```
userfaultfd overhead:
  - Per fault: ~1-10µs (syscall + thread wakeup + copy + wakeup)
  - UFFDIO_COPY for 1 page: ~5µs
  - vs. normal page fault: ~1µs (anon) to 1ms+ (from disk)

Optimization techniques:
  1. Batch: use UFFDIO_COPY with larger ranges when possible
  2. Async wake: UFFDIO_COPY_MODE_DONTWAKE, then UFFDIO_WAKE
  3. Multiple handler threads: each reads from the same uf_fd
  4. Huge pages: register with huge page alignment, copy 2MB at a time
```

```bash
# userfaultfdの活動を監視
bpftrace -e '
kprobe:handle_userfault {
    @[kstack] = count();
}'

# プロセスがuserfaultfdを使っているか確認
ls -la /proc/<pid>/fd/ | grep userfaultfd

# uffdのioctlをトレース
bpftrace -e '
tracepoint:syscalls:sys_enter_ioctl
/args->fd > 0 && (args->cmd == 0xc018aa03 || args->cmd == 0xc028aa04)/
{ printf("UFFDIO from pid %d\n", pid); }'
```

## 参考文献 {#further-reading}

- [ページフォールトハンドラ](page-fault.md) — userfaultfdの入り口であるhandle_mm_fault
- [ファイルバックmmap](file-mmap.md) — mmapの基礎
- [QEMU](../virtualization/kvm-memory.md) — userfaultfdを使ったライブマイグレーション
- [CRIUのドキュメント](https://criu.org/Userfaultfd) — 遅延ページ転送による復元
- `mm/userfaultfd.c` — カーネルの実装
- `tools/testing/selftests/mm/userfaultfd.c` — カーネルのセルフテストと使用例
