# libbpfとBPFスケルトン {#libbpf-and-bpf-skeletons}

> BPFプログラムのロードと操作のための標準ユーザー空間ライブラリ

## libbpfが行うこと {#what-libbpf-does}

libbpfはBPFの公式ユーザー空間ライブラリ（カーネルツリーの`tools/lib/bpf/`）であり、もともとはWang NanによってLinux 4.3で導入された。[(commit)](https://git.kernel.org/linus/1b76c13e4b36f978ea45af079f38ad423a229630) 次のことを処理する。

- BPF ELFオブジェクト（`.bpf.o`ファイル）のロード
- ロード時のCO-REリロケーションのパッチ適用
- BPFプログラムの宣言からのmap作成
- プログラムのフックへのアタッチ
- ユーザー空間からのmapアクセス
- リングバッファとperfバッファのポーリング

## BPFスケルトンのワークフロー {#the-bpf-skeleton-workflow}

推奨される現代的な手法では**BPFスケルトン**を使う。これは特定のBPFプログラムをラップする生成されたC構造体であり、そのmapやプログラムへの型安全なアクセスを提供する。

```
                 ┌──────────────────────────────────────┐
                 │ Write BPF program (minimal.bpf.c)    │
                 └───────────────┬──────────────────────┘
                                 │ clang -target bpf -g
                                 ▼
                 ┌──────────────────────────────────────┐
                 │ BPF object (minimal.bpf.o)           │
                 └───────────────┬──────────────────────┘
                                 │ bpftool gen skeleton
                                 ▼
                 ┌──────────────────────────────────────┐
                 │ Generated header (minimal.skel.h)    │
                 │   minimal_bpf__open()                │
                 │   minimal_bpf__load()                │
                 │   minimal_bpf__attach()              │
                 │   minimal_bpf__destroy()             │
                 └───────────────┬──────────────────────┘
                                 │ #include in userspace code
                                 ▼
                 ┌──────────────────────────────────────┐
                 │ Userspace program (minimal.c)        │
                 └──────────────────────────────────────┘
```

## 完全な最小例 {#complete-minimal-example}

### BPF側: `minimal.bpf.c` {#bpf-side-minimalbpfc}

```c
// SPDX-License-Identifier: GPL-2.0
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_core_read.h>

char LICENSE[] SEC("license") = "GPL";

/* map: PIDごとにシステムコールを数える */
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 8192);
    __type(key, u32);    /* pid */
    __type(value, u64);  /* 回数 */
} syscall_count SEC(".maps");

/* グローバル変数（.bssに格納され、MAP_TYPE_ARRAYが裏付けとなる） */
volatile u64 total_syscalls = 0;

SEC("tracepoint/raw_syscalls/sys_enter")
int count_syscall(struct trace_event_raw_sys_enter *ctx)
{
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    u64 *count;

    count = bpf_map_lookup_elem(&syscall_count, &pid);
    if (count) {
        (*count)++;
    } else {
        u64 one = 1;
        bpf_map_update_elem(&syscall_count, &pid, &one, BPF_NOEXIST);
    }

    __sync_fetch_and_add(&total_syscalls, 1);
    return 0;
}
```

### スケルトンの生成 {#generate-the-skeleton}

```bash
# BPFプログラムをコンパイルする
clang -target bpf -O2 -g \
    -I/usr/include/bpf \
    -c minimal.bpf.c -o minimal.bpf.o

# スケルトンヘッダを生成する
bpftool gen skeleton minimal.bpf.o > minimal.skel.h
```

生成された`minimal.skel.h`には次が含まれる。

```c
/* bpftoolによる自動生成 */
struct minimal_bpf {
    struct bpf_object_skeleton *skeleton;
    struct bpf_object *obj;

    struct {
        struct bpf_map *syscall_count;    /* 型付きのmap参照 */
        struct bpf_map *bss;
        struct bpf_map *rodata;
    } maps;

    struct {
        struct bpf_program *count_syscall; /* 型付きのプログラム参照 */
    } progs;

    struct {
        struct bpf_link *count_syscall;    /* 型付きのリンク参照 */
    } links;

    struct minimal_bpf__bss {
        volatile u64 total_syscalls;       /* 直接アクセスできる! */
    } *bss;
};

static inline struct minimal_bpf *minimal_bpf__open(void);
static inline int minimal_bpf__load(struct minimal_bpf *obj);
static inline int minimal_bpf__attach(struct minimal_bpf *obj);
static inline void minimal_bpf__destroy(struct minimal_bpf *obj);
```

### ユーザー空間側: `minimal.c` {#userspace-side-minimalc}

```c
#include <stdio.h>
#include <unistd.h>
#include <signal.h>
#include <bpf/libbpf.h>
#include "minimal.skel.h"

static volatile bool exiting = false;

static void sig_handler(int sig)
{
    exiting = true;
}

int main(void)
{
    struct minimal_bpf *skel;
    int err;

    /* 1. Open: ELFを解析し、map/プログラムのオブジェクトを作成する */
    skel = minimal_bpf__open();
    if (!skel) {
        fprintf(stderr, "Failed to open BPF skeleton\n");
        return 1;
    }

    /* 任意: ロード前に設定する */
    /* skel->rodata->my_config = 42; */

    /* 2. Load: mapを作成し、プログラムを検証+JITし、CO-REリロケーションを行う */
    err = minimal_bpf__load(skel);
    if (err) {
        fprintf(stderr, "Failed to load BPF skeleton: %d\n", err);
        goto cleanup;
    }

    /* 3. Attach: プログラムをそれぞれのフックに接続する */
    err = minimal_bpf__attach(skel);
    if (err) {
        fprintf(stderr, "Failed to attach BPF skeleton: %d\n", err);
        goto cleanup;
    }

    signal(SIGINT, sig_handler);
    printf("Counting syscalls... Press Ctrl+C to stop\n");

    while (!exiting) {
        sleep(1);

        /* スケルトンを通じてグローバル変数に直接アクセスする */
        printf("Total syscalls: %llu\n", skel->bss->total_syscalls);

        /* mapの値にアクセスする */
        u32 pid = getpid();
        u64 count;
        int map_fd = bpf_map__fd(skel->maps.syscall_count);
        if (bpf_map_lookup_elem(map_fd, &pid, &count) == 0)
            printf("This process syscalls: %llu\n", count);
    }

cleanup:
    /* 4. Destroy: プログラムをデタッチし、mapを閉じ、メモリを解放する */
    minimal_bpf__destroy(skel);
    return err;
}
```

```bash
# ユーザー空間側をコンパイルする
gcc -o minimal minimal.c -lbpf -lelf -lz
# 実行する
sudo ./minimal
```

## リングバッファのポーリング {#ring-buffer-polling}

`BPF_MAP_TYPE_RINGBUF`を使うイベント駆動型プログラムの場合:

```c
/* ring_buffer APIを使ったユーザー空間でのポーリング */
#include <bpf/libbpf.h>
#include "events.skel.h"

struct event {
    u32 pid;
    char comm[16];
};

static int handle_event(void *ctx, void *data, size_t data_sz)
{
    struct event *e = data;
    printf("pid=%u comm=%s\n", e->pid, e->comm);
    return 0;
}

int main(void)
{
    struct events_bpf *skel = events_bpf__open_and_load();
    events_bpf__attach(skel);

    /* リングバッファマネージャを作成する */
    struct ring_buffer *rb = ring_buffer__new(
        bpf_map__fd(skel->maps.events),
        handle_event,
        NULL,  /* handle_eventに渡されるctx */
        NULL   /* オプション */
    );

    while (true) {
        /* 100msのタイムアウトでポーリングする */
        int err = ring_buffer__poll(rb, 100);
        if (err == -EINTR)
            break;
    }

    ring_buffer__free(rb);
    events_bpf__destroy(skel);
    return 0;
}
```

## libbpfの中核オブジェクト {#libbpf-core-objects}

```c
/* オブジェクト: .bpf.oファイルを表す */
struct bpf_object *obj = bpf_object__open("prog.bpf.o");
bpf_object__load(obj);   /* mapを作成し、プログラムを検証する */
bpf_object__close(obj);

/* プログラム: 1つのBPF関数を表す */
struct bpf_program *prog =
    bpf_object__find_program_by_name(obj, "my_prog");

/* ロード前にプログラムタイプを変更する */
bpf_program__set_type(prog, BPF_PROG_TYPE_KPROBE);

/* 手動でアタッチする */
struct bpf_link *link = bpf_program__attach(prog);
bpf_link__destroy(link);  /* デタッチする */

/* map: BPF mapを表す */
struct bpf_map *map =
    bpf_object__find_map_by_name(obj, "my_map");
int fd = bpf_map__fd(map);

/* ロード前にmapのサイズを変更する */
bpf_map__set_max_entries(map, 65536);
```

## プログラムの手動アタッチ {#manual-program-attachment}

明示的なセットアップが必要なプログラムタイプの場合:

```c
/* 特定のネットワークインターフェースにアタッチする（XDP） */
struct bpf_link *link = bpf_program__attach_xdp(prog, ifindex);

/* 関数名でkprobeをアタッチする */
struct bpf_link *link = bpf_program__attach_kprobe(prog,
    false,          /* false=kprobe、true=kretprobe */
    "tcp_sendmsg"   /* カーネル関数名 */
);

/* tracepointをアタッチする */
struct bpf_link *link = bpf_program__attach_tracepoint(prog,
    "syscalls",         /* tracepointのカテゴリ */
    "sys_enter_execve"  /* tracepoint名 */
);

/* uprobeをアタッチする */
struct bpf_link *link = bpf_program__attach_uprobe(prog,
    false,              /* false=uprobe、true=uretprobe */
    -1,                 /* -1 = 全プロセス */
    "/lib/libc.so.6",  /* バイナリのパス */
    0x12345             /* バイナリ内のオフセット */
);

/* cgroupにアタッチする */
struct bpf_link *link = bpf_program__attach_cgroup(prog, cgroup_fd);
```

## エラー処理とロギング {#error-handling-and-logging}

```c
/* libbpfのロギングを設定する */
static int libbpf_print_fn(enum libbpf_print_level level,
                            const char *format, va_list args)
{
    if (level == LIBBPF_DEBUG)
        return 0;  /* デバッグログを抑制する */
    return vfprintf(stderr, format, args);
}

libbpf_set_print(libbpf_print_fn);

/* Verifierエラーを確認する */
skel = minimal_bpf__open_and_load();
if (!skel) {
    /* libbpfはprint_fn経由ですでにVerifierログを出力している */
    return 1;
}
```

## スケルトンを使ったBPFオブジェクトのライフサイクル {#bpf-object-lifecycle-with-skeleton}

```
open()           parse ELF sections, create bpf_object
   │
   │  [optional customization here]
   │    - set map sizes
   │    - set program types
   │    - set rodata constants
   ▼
load()           kernel: create maps, verify programs, JIT
   │             libbpf: CO-RE relocation, pin maps
   │
   │  [optional manual attachment]
   ▼
attach()         connect programs to their default hooks
   │             (SEC("tracepoint/...") → auto attach)
   │
   │  [program running, handle events]
   ▼
destroy()        detach all links
                 close file descriptors
                 free bpf_object
```

## 便利なbpftoolコマンド {#useful-bpftool-commands}

```bash
# ロードされたプログラムを調べる
bpftool prog list
bpftool prog show id 42 --pretty

# プログラムのバイトコードをダンプする
bpftool prog dump xlated id 42
bpftool prog dump jited id 42 linum  # ソースの行番号付き

# mapを調べる
bpftool map list
bpftool map dump id 5
bpftool map lookup id 5 key 0 0 0 0  # key=0（4バイト）を検索

# ファイルシステムにピン留めする
bpftool prog pin id 42 /sys/fs/bpf/my_prog
bpftool map pin id 5 /sys/fs/bpf/my_map

# スケルトンを生成する
bpftool gen skeleton prog.bpf.o > prog.skel.h

# vmlinux.hを生成する
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

## 参考文献 {#further-reading}

- [BPFアーキテクチャ](bpf-overview.md) — プログラムタイプとbpf()システムコール
- [BPF Maps](bpf-maps.md) — mapタイプと操作
- [BTFとCO-RE](btf-core.md) — CO-REリロケーションの仕組み
- カーネルツリーの`tools/lib/bpf/` — libbpfのソース
- GitHub上の`libbpf-bootstrap` — BPFプログラムのプロジェクトテンプレート
