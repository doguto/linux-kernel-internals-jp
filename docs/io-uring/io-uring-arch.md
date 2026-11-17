# io_uringのアーキテクチャとリング {#io_uring-architecture-and-rings}

> 共有メモリのリング、SQE、CQE

## io_uringが解決する問題 {#the-problem-io_uring-solves}

従来のI/Oインターフェース:

- `read()`/`write()`: 操作ごとにシステムコール1回、ブロッキング
- `aio_read()`（POSIX AIO）: スレッドプールベース、APIが複雑、性能が低い
- `epoll` + ノンブロッキング: 操作ごとにシステムコール2回（確認に1回、実行に1回）、バッチ化なし
- Linux `libaio`: 低レベル、対応する操作が不完全

io_uringの解決策は、**共有メモリ上の2つのロックフリーなリング**である。ユーザー空間はサブミッションキューに書き込み、カーネルはコンプリーションキューに結果を投稿する。操作ごとのシステムコールは不要である（特にSQPOLLを使う場合）。

## io_uring_setup {#iouringsetup}

```c
#include <linux/io_uring.h>

struct io_uring_params params = {
    .flags = IORING_SETUP_SQPOLL,  /* 任意: カーネルのポーリングスレッド */
    /* .sq_thread_cpu, .sq_thread_idle など */
};

int ring_fd = io_uring_setup(32,  /* sq_entries: 2の累乗 */
                              &params);

/* paramsはカーネルによって埋められる: */
/* params.sq_off: sqリングのmmap内のオフセット */
/* params.cq_off: cqリングのmmap内のオフセット */
/* params.features: IORING_FEAT_SINGLE_MMAP、IORING_FEAT_NODROPなど */
```

`io_uring_setup`の後、ユーザー空間は2つ（`IORING_FEAT_SINGLE_MMAP`の場合は1つ）のメモリ領域をマップする。

```c
/* SQリングとCQリングをマップする */
void *sq_ring = mmap(NULL, params.sq_off.array + params.sq_entries * sizeof(u32),
                     PROT_READ|PROT_WRITE, MAP_SHARED|MAP_POPULATE,
                     ring_fd, IORING_OFF_SQ_RING);

void *cq_ring = mmap(NULL, params.cq_off.cqes +
                           params.cq_entries * sizeof(struct io_uring_cqe),
                     PROT_READ|PROT_WRITE, MAP_SHARED|MAP_POPULATE,
                     ring_fd, IORING_OFF_CQ_RING);

/* SQE配列は別途マップする */
struct io_uring_sqe *sqes = mmap(NULL,
    params.sq_entries * sizeof(struct io_uring_sqe),
    PROT_READ|PROT_WRITE, MAP_SHARED|MAP_POPULATE,
    ring_fd, IORING_OFF_SQES);
```

## リングレイアウト {#ring-layout}

```
SQ ring (at IORING_OFF_SQ_RING):
  ┌─────────────────────────────────────────┐
  │ head     (u32) ← kernel reads from here │
  │ tail     (u32) ← userspace writes here  │
  │ mask     (u32) = sq_entries - 1         │
  │ entries  (u32) = sq_entries             │
  │ flags    (u32) = IORING_SQ_NEED_WAKEUP  │
  │ dropped  (u32) = dropped SQEs           │
  │ array[]  (u32) = indices into SQE array │
  └─────────────────────────────────────────┘

SQE array (at IORING_OFF_SQES):
  ┌───────────────────────────────────────────┐
  │ sqe[0] │ sqe[1] │ ... │ sqe[sq_entries-1] │
  └───────────────────────────────────────────┘

CQ ring (at IORING_OFF_CQ_RING):
  ┌─────────────────────────────────────────┐
  │ head     (u32) ← userspace reads here   │
  │ tail     (u32) ← kernel writes here     │
  │ mask     (u32) = cq_entries - 1         │
  │ entries  (u32) = cq_entries             │
  │ overflow (u32) = CQEs dropped (if full) │
  │ cqes[]   = CQE ring array               │
  └─────────────────────────────────────────┘
```

## struct io_uring_sqe {#struct-iouringsqe}

サブミッションキューエントリ（SQE）は1つのI/O操作を記述する。

```c
/* include/uapi/linux/io_uring.h */
struct io_uring_sqe {
    __u8    opcode;         /* IORING_OP_READ, IORING_OP_WRITE, ... */
    __u8    flags;          /* IOSQE_FIXED_FILE, IOSQE_IO_LINK, ... */
    __u16   ioprio;         /* I/O優先度（ioniceのクラス/レベル） */
    __s32   fd;             /* ファイルディスクリプタ（または固定ファイルのインデックス） */
    union {
        __u64   off;        /* read/writeのオフセット */
        __u64   addr2;      /* 一部の操作で使用 */
        struct {
            __u32 cmd_op;
            __u32 __pad1;
        };
    };
    union {
        __u64   addr;       /* バッファへのポインタ（または固定バッファのインデックス） */
        __u64   splice_off_in;
    };
    __u32   len;            /* バッファ長、またはLINKの場合はSQE数 */
    union {
        __kernel_rwf_t  rw_flags;   /* read/writeのフラグ */
        __u32           fsync_flags;
        __u16           poll_events;
        __u32           poll32_events;
        __u32           sync_range_flags;
        __u32           msg_flags;
        __u32           timeout_flags;
        __u32           accept_flags;
        __u32           cancel_flags;
        __u32           open_flags;
        __u32           statx_flags;
        __u32           fadvise_advice;
        __u32           splice_flags;
        __u32           rename_flags;
        __u32           unlink_flags;
        __u32           hardlink_flags;
        __u32           xattr_flags;
        __u32           msg_ring_flags;
        __u32           uring_cmd_flags;
    };
    __u64   user_data;      /* CQEにコピーされるタグ。完了の照合用 */
    union {
        __u16   buf_index;  /* 固定バッファのインデックス（IORING_OP_*用） */
        __u16   buf_group;  /* バッファ選択用のバッファグループ */
    };
    __u16   personality;    /* この操作で使うクレデンシャル */
    union {
        __s32   splice_fd_in;
        __u32   file_index;   /* 固定ファイル用 */
        __u32   optlen;
        struct {
            __u16 addr_len;
            __u16 __pad3[1];
        };
    };
    union {
        struct {
            __u64 addr3;
            __u64 __pad2[1];
        };
        __u8 cmd[0];  /* IORING_OP_URING_CMDのペイロード */
    };
};
```

## struct io_uring_cqe {#struct-iouringcqe}

コンプリーションキューエントリ（CQE）は結果を運ぶ。

```c
struct io_uring_cqe {
    __u64   user_data;  /* SQEからコピーされる（照合用） */
    __s32   res;        /* 結果コード: 転送バイト数、または-errno */
    __u32   flags;      /* IORING_CQE_F_MORE, IORING_CQE_F_SOCK_NONEMPTY */
    __u64   big_cqe[];  /* IORING_SETUP_CQE32の場合: 追加の16バイト */
};
```

`res`のセマンティクスは、対応するブロッキングシステムコールと同じである。

- `IORING_OP_READ`: 読み込んだバイト数、または-errno
- `IORING_OP_WRITE`: 書き込んだバイト数、または-errno
- `IORING_OP_ACCEPT`: 新しいfd、または-errno
- `IORING_OP_CONNECT`: 成功時は0、または-errno

## サブミッションの流れ {#submission-flow}

```
1. SQEスロットを取得する:
   tail = sq_ring->tail & sq_ring->mask
   sqe = &sqes[tail]

2. SQEを埋める:
   sqe->opcode = IORING_OP_READ
   sqe->fd = fd
   sqe->addr = (uintptr_t)buf
   sqe->len = len
   sqe->off = offset
   sqe->user_data = my_tag

3. tailを進める（カーネルに公開する）:
   sq_ring->array[tail] = tail  /* 間接インデックス */
   smp_store_release(&sq_ring->tail, tail + 1)

4. 必要に応じて: io_uring_enter(ring_fd, to_submit, min_complete, flags)
   /* SQPOLLの場合: これは省略する。カーネルが自動的にポーリングする */
```

## 完了の回収 {#completion-harvesting}

```
1. 新しいCQEがあるか確認する:
   if (cq_ring->head == READ_ONCE(cq_ring->tail)) { /* 完了なし */ }

2. CQEを処理する:
   head = cq_ring->head & cq_ring->mask
   cqe = &cq_ring->cqes[head]
   result = cqe->res
   tag = cqe->user_data

3. headを進める（消費する）:
   smp_store_release(&cq_ring->head, cq_ring->head + 1)
```

## io_uring_enter: システムコール {#iouringenter-the-syscall}

```c
int io_uring_enter(unsigned int fd,    /* リングのfd */
                   unsigned int to_submit,  /* 投入するSQE数 */
                   unsigned int min_complete, /* N個のCQEを待つ */
                   unsigned int flags,   /* IORING_ENTER_GETEVENTS, ... */
                   sigset_t *sig,        /* 待機中のシグナルマスク（任意） */
                   size_t sigsz);
```

バッチ化により、1回の`io_uring_enter`で多数の操作を投入し、多数の完了を待つことができる。これにより、システムコールのオーバーヘッドが並行するすべてのI/Oに分散（償却）される。

## カーネル側: struct io_ring_ctx {#kernel-side-struct-ioringctx}

```c
/* io_uring/io_uring.h */
struct io_ring_ctx {
    struct {
        unsigned        flags;
        unsigned        compat;
        unsigned        drain_next;
        unsigned        eventfd_async;
        /* ... */
    } ____cacheline_aligned_in_smp;

    struct io_rings     *rings;          /* 共有SQ/CQリング */
    struct io_uring_sqe *sq_sqes;        /* SQE配列 */

    /* 完了の状態 */
    spinlock_t          completion_lock;
    struct list_head    locked_free_list;
    unsigned int        locked_free_nr;

    /* サブミッションの状態 */
    struct io_sq_data   *sq_data;        /* SQPOLLスレッド */

    /* ブロッキング操作用のワーカー */
    struct io_wq        *io_wq;

    /* ... ほか多数のフィールド ... */
};
```

## 参考文献 {#further-reading}

- [操作と高度な機能](io-uring-ops.md): サポートされる操作、SQPOLL、固定バッファ
- [BPF](../bpf/README.md): io_uring + BPFプログラム
- カーネルツリーの`io_uring/`: 完全な実装（約25,000行）
- `liburing`: 便利なAPIを備えたユーザー空間ライブラリ
