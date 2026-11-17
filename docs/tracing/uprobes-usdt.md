# uprobeとUSDT: ユーザー空間の動的トレーシング {#uprobes-and-usdt-userspace-dynamic-tracing}

> ユーザー空間の関数や静的tracepointにBPFプログラムをアタッチする

## uprobe: ユーザー空間の動的計装 {#uprobes-dynamic-userspace-instrumentation}

**uprobe**はkprobeのユーザー空間版であり、Linux 3.5でSrikar Dronamraju（IBM）によって導入された[（コミット）](https://git.kernel.org/linus/2b144498350860b6ee9dc57ff27a93ad488de5dc)。ユーザー空間バイナリ内の任意の命令アドレスを、次の手順で計装する。
1. バイナリ内のアドレスを見つける
2. その命令をブレークポイント（x86ではINT3）に置き換える
3. ブレークポイントでBPF/ftraceハンドラを実行する

```
libc の malloc() への uprobe:
  プロセス:    [通常のコード]→ malloc() → [int3 トラップ] → uprobe ハンドラ → malloc 続行
  CPU コスト:  1 ヒットあたり約 100-500ns（ユーザー/カーネル遷移のため kprobe よりはるかに高い）
```

## bpftraceでuprobeをアタッチする {#attaching-uprobes-with-bpftrace}

```bash
# 任意のプロセスの malloc() 呼び出しをトレースする:
bpftrace -e 'uprobe:/lib/x86_64-linux-gnu/libc.so.6:malloc
{ printf("malloc(%d) by %s\n", arg0, comm); }'

# 特定のバイナリをトレースする:
bpftrace -e 'uprobe:/usr/bin/nginx:ngx_http_process_request
{ @[comm] = count(); }'

# 関数の引数:
bpftrace -e 'uprobe:/lib/libc.so.6:fopen
{ printf("fopen(\"%s\", \"%s\")\n", str(arg0), str(arg1)); }'

# 戻り値（uretprobe）:
bpftrace -e '
uprobe:/lib/libc.so.6:malloc { @start[tid] = arg0; }
uretprobe:/lib/libc.so.6:malloc /@start[tid]/
{
    printf("malloc(%d) = %p\n", @start[tid], retval);
    delete(@start[tid]);
}'

# PID を指定してトレースする:
bpftrace -p 1234 -e 'uprobe:/usr/bin/python3:PyObject_MALLOC
{ @alloc_size = hist(arg0); }'

# シンボルオフセットを使う:
bpftrace -e 'uprobe:/usr/bin/nginx:0x12345  # specific offset
{ printf("hit offset 0x12345\n"); }'
```

## libbpfでのuprobe {#uprobes-with-libbpf}

```c
/* uprobe_example.bpf.c */
#include <vmlinux.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

/* malloc() の入口にアタッチする */
SEC("uprobe//lib/x86_64-linux-gnu/libc.so.6:malloc")
int handle_malloc_entry(struct pt_regs *ctx)
{
    size_t size = PT_REGS_PARM1(ctx);  /* 第 1 引数 */
    bpf_printk("malloc(%zu) called\n", size);
    return 0;
}

/* malloc() の戻りにアタッチする */
SEC("uretprobe//lib/x86_64-linux-gnu/libc.so.6:malloc")
int handle_malloc_return(struct pt_regs *ctx)
{
    void *ptr = (void *)PT_REGS_RC(ctx);  /* 戻り値 */
    bpf_printk("malloc returned %p\n", ptr);
    return 0;
}
```

```c
/* uprobe_example.c - ユーザー空間のローダー */
#include "uprobe_example.skel.h"

int main(void)
{
    struct uprobe_example_bpf *skel = uprobe_example_bpf__open_and_load();

    /* libbpf は SEC() アノテーションに基づいて自動的にアタッチする */
    uprobe_example_bpf__attach(skel);

    /* あるいは特定のオフセットに手動でアタッチする: */
    skel->links.handle_malloc_entry =
        bpf_program__attach_uprobe(skel->progs.handle_malloc_entry,
                                    false,    /* 戻りではなく入口 */
                                    -1,       /* すべての PID */
                                    "/lib/x86_64-linux-gnu/libc.so.6",
                                    0x12345); /* ファイルオフセット */

    /* 割り込まれるまで実行する */
    pause();
    uprobe_example_bpf__destroy(skel);
    return 0;
}
```

## USDT: ユーザー静的定義トレースポイント {#usdt-user-statically-defined-tracepoints}

USDTトレースポイントは、アプリケーション開発者が追加する**静的な**計装ポイントである。（任意の命令にアタッチできる）uprobeとは異なり、USDTは意味のある位置に意図的に配置される。

```c
/* アプリケーションのコード（Python、PostgreSQL、Node.js、MySQL など） */
#include <sys/sdt.h>   /* または probe.h */

void process_request(struct request *req)
{
    /* USDT トレースポイント: プローブがアタッチされているときに発火し、それ以外は NOP */
    DTRACE_PROBE1(myapp, request-start, req->id);

    /* ... 処理 ... */

    DTRACE_PROBE2(myapp, request-done, req->id, req->status);
}
```

```c
/* DTRACE_PROBE* のコンパイル結果: */
/* 通常の実行時: NOP 命令（x86 では 0x90、オーバーヘッド約 0ns） */
/* アタッチ時: NOP が INT3 に置き換えられ、uprobe ハンドラが発火する */
/* ツールがアタッチされていなければオーバーヘッドは完全にゼロ */
```

### USDTプローブを見つける {#finding-usdt-probes}

```bash
# バイナリ内のすべての USDT プローブを一覧表示する:
readelf -n /usr/bin/python3 | grep stapsdt | head -20
# NT_STAPSDT (System Tap Static Probe)
#   Provider: python
#   Name: function__entry
#   Location: 0x000000000041a3c0
#   Base: 0x0000000000000000
#   Semaphore: 0x0000000000000000
#   Arguments: -4@%rdi -8@%rsi

# bpftrace を使う:
bpftrace -l 'usdt:/usr/bin/python3:*'
# usdt:/usr/bin/python3:python:function__entry
# usdt:/usr/bin/python3:python:function__return
# usdt:/usr/bin/python3:python:import__find__load__start
# usdt:/usr/bin/python3:python:gc__start
# usdt:/usr/bin/python3:python:gc__done

# tplist（BCC）を使う:
tplist -l /usr/bin/python3
```

## bpftraceでのUSDT: Python関数のトレース {#usdt-with-bpftrace-python-function-tracing}

```bash
# Python の関数呼び出しをトレースする:
bpftrace -e '
usdt:/usr/bin/python3:python:function__entry
{
    printf("%s:%d %s\n",
           str(arg1),   /* filename */
           arg2,        /* line number */
           str(arg0));  /* function name */
}'

# Python の GC による停止を数える:
bpftrace -e '
usdt:/usr/bin/python3:python:gc__start
{ @[arg0] = count(); }   /* arg0 = generation */'

# Python の関数呼び出しの時間を計測する:
bpftrace -e '
usdt:/usr/bin/python3:python:function__entry
{ @start[tid] = nsecs; }

usdt:/usr/bin/python3:python:function__return
/@start[tid]/
{
    @lat_us[str(arg0)] = hist((nsecs - @start[tid]) / 1000);
    delete(@start[tid]);
}'
```

## PostgreSQLのUSDTプローブ {#postgresql-usdt-probes}

```bash
# PostgreSQL は豊富な USDT をサポートしている:
bpftrace -l 'usdt:/usr/lib/postgresql/14/bin/postgres:postgresql:*'
# usdt:...:postgresql:query__start
# usdt:...:postgresql:query__done
# usdt:...:postgresql:lock__wait__start
# usdt:...:postgresql:lock__wait__done
# usdt:...:postgresql:transaction__start
# usdt:...:postgresql:transaction__commit
# usdt:...:postgresql:transaction__abort

# 遅いクエリ（100ms 超）をトレースする:
bpftrace -p $(pgrep postgres) -e '
usdt:/usr/lib/postgresql/14/bin/postgres:postgresql:query__start
{ @start[tid] = nsecs; @query[tid] = str(arg0); }

usdt:/usr/lib/postgresql/14/bin/postgres:postgresql:query__done
/@start[tid] && (nsecs - @start[tid]) > 100000000/  /* 100ms */
{
    printf("SLOW QUERY (%dms): %s\n",
           (nsecs - @start[tid]) / 1000000,
           @query[tid]);
    delete(@start[tid]);
    delete(@query[tid]);
}'
```

## Node.jsのUSDTプローブ {#nodejs-usdt-probes}

```bash
# --with-dtrace 付きでビルドされた Node.js:
bpftrace -l 'usdt:/usr/bin/node:*'
# usdt:/usr/bin/node:node:http__server__request
# usdt:/usr/bin/node:node:net__server__connection
# usdt:/usr/bin/node:node:gc__start

# HTTP リクエストをトレースする:
bpftrace -e '
usdt:/usr/bin/node:node:http__server__request
{
    printf("HTTP %s %s from %s\n",
           str(arg2),  /* method */
           str(arg3),  /* url */
           str(arg5)); /* remote address */
}'
```

## uprobeのオーバーヘッドとtracepointの比較 {#uprobe-overhead-vs-tracepoints}

```
計装オーバーヘッドの比較:
  kprobe:         1 ヒットあたり約 100ns（コンテキストはカーネル内のまま）
  tracepoint:     1 ヒットあたり約 10ns（静的に配置、オーバーヘッド最小）
  uprobe:         1 ヒットあたり約 200-500ns（ユーザー→カーネル→ユーザーの遷移）
  USDT (attached): 約 200-500ns（アタッチ時は uprobe と同じ）
  USDT (idle):     約 0ns（NOP 命令、オーバーヘッドなし）
  perf_event:     ハードウェア PMU カウンタで約 50ns

本番環境での考慮事項:
  - uprobe: ホットパスでは目立つオーバーヘッドを生じうる
  - USDT: プローブされていなければコストゼロ、プローブ時も最小限
  - 本番環境のトレーシングでは uprobe より USDT を優先する
  - uprobe のアタッチは実行時間の 1% 未満の経路に限定する
```

## uprobeのカーネル内部 {#uprobe-kernel-internals}

```c
/* kernel/events/uprobes.c */

/* uprobe 構造体: */
struct uprobe {
    struct rb_node   rb_node;    /* uprobe_tree 内、(inode, offset) をキーとする */
    refcount_t       ref;
    struct rw_semaphore register_rwsem;
    struct inode    *inode;      /* バイナリの inode */
    loff_t           offset;     /* プローブ対象命令のファイルオフセット */
    loff_t           ref_ctr_offset; /* USDT セマフォのオフセット */
    unsigned long    flags;
    struct arch_uprobe arch;     /* 保存された元の命令 */
};

/* トレース対象プロセスが uprobe の INT3 を実行したとき: */
/* arch/x86/kernel/traps.c: do_int3 → uprobe_pre_sstep_notifier */
void uprobe_notify_resume(struct pt_regs *regs)
{
    /* 1. どの uprobe が発火したかを特定する */
    /* 2. 登録済みのすべてのハンドラ（BPF プログラム）を呼び出す */
    /* 3. 元の命令をシングルステップ実行する */
    /* 4. 通常の実行に戻る */
}
```

## 参考文献 {#further-reading}

- [kprobeとtracepoint](kprobes-tracepoints.md) — カーネル側の動的プローブ
- [BPFマップ](../bpf/bpf-maps.md) — BPFプログラム間でのデータ共有
- [libbpfとスケルトン](../bpf/libbpf.md) — libbpfによるuprobeのアタッチ
- [perfプロファイリング](perf-profiling.md) — USDTを使ったperf record
- `kernel/events/uprobes.c` — uprobeの実装
- `Documentation/trace/uprobetracer.rst`
- BCCツール: `trace`、`tplist`、`argdist`、`funccount`
