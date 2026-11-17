# kprobeとtracepoint {#kprobes-and-tracepoints}

> カーネル内の動的・静的な計装ポイント

## なぜ3つの仕組みがあるのか {#why-three-mechanisms}

kprobe、tracepoint、BPFは冗長なのではない。10年以上かけて発展してきた問題の、それぞれ異なる部分を解決している。

### kprobe: 動的だが壊れやすい（2000年代） {#kprobes-dynamic-but-fragile-2000s}

kprobeは、再コンパイルなしに本番カーネルを計装する手段としてLinux 2.6.9で登場した。`tcp_sendmsg`へのkprobeは、その関数が呼ばれるたびに発火し、レジスタの状態と、任意のコードを実行する能力を与えてくれる。

問題は、kprobeが本質的に不安定であることだ。`tcp_sendmsg`の名前が変わったり、コンパイラによってインライン化されたり、呼び出し規約が変わったりすると、プローブは黙って何もしなくなるか、誤った場所にアタッチされる。あるカーネルバージョン向けに書かれたツールは、次のバージョンで壊れることが多い。このためkprobeは、場当たり的なデバッグには有用だが、本番環境の監視には信頼できないものとなった。

### tracepoint: 安定したABI（Linux 2.6.28、2008年） {#tracepoints-stable-abi-linux-2628-2008}

Mathieu Desnoyersは、トレーシングツールに安定したインターフェースを与えるため、`TRACE_EVENT`基盤を導入した[（LWN）](https://lwn.net/Articles/379903/)。`sched_switch`や`block_rq_issue`のようなtracepointは、カーネル開発者が意味的に重要な位置に意図的に配置した、名前付きのアノテーションである。

その契約は、tracepointの*名前*と*フィールド*がカーネルバージョンをまたいで安定している（ユーザー空間から見えるABIのように扱われる）ことである。`sched_switch`にアタッチして`prev_pid`/`next_pid`を読むツールは、周辺のコードがどう変わろうと、リリースを重ねても動作し続ける。

オーバーヘッドゼロの設計（コンシューマが登録されていないときはチェック全体をパッチで取り除くjump label）のおかげで、サブシステムのメンテナはtracepointを惜しみなく追加する気になった — 使われていないときのコストはゼロだからである。

制約は、tracepointが開発者の配置した場所にしか存在しないことである。興味のあるすべての関数にtracepointがあるわけではなく、追加するにはカーネルパッチが必要になる。

### BPF: 両方の層でのプログラマブルなロジック（Linux 3.18以降） {#bpf-programmable-logic-at-both-layers-linux-318}

BPFプログラムはtracepointにも*kprobeにも*アタッチできるが、決定的な能力を付け加える。すべてのイベントを単にログに記録する代わりに、BPFプログラムはカーネル内で集計、フィルタ、計算を行い、要約だけをユーザー空間に送れる。

これによりtracepointは「発火して記録する」ものから「発火して計算する」ものへと変わった。`block_rq_complete`上のBPFプログラムは、デバイスごとのレイテンシヒストグラムを完全にカーネルメモリ内で維持し、BPFマップとして公開できる。BPFがなければ、そのヒストグラムを収集するには生のイベントをすべてユーザー空間にコピーしてそこで集計する必要があり、高いI/Oレートではそれは法外なコストだった。

`fentry`/`fexit`（Linux 5.5、2020年）[（コミット）](https://git.kernel.org/linus/fec56f5890d93fc2ed74166c397dc186b1c25951)は4つ目の層を追加した。BTFの型情報を使って型付きの引数にアクセスしつつ、任意のカーネル関数の入口／出口にアタッチするBPFプログラムである。kprobeのINT3トラップのオーバーヘッドもなく、明示的な`TRACE_EVENT`アノテーションも必要としない。

```
kprobe       → どこにでもアタッチ可能、不安定な ABI、INT3 のオーバーヘッド
tracepoint   → 安定した ABI、オーバーヘッドゼロ、カバー範囲は限定的
BPF (kprobe) → プログラマブルな kprobe、カーネル内集計
BPF (tp)     → プログラマブルな tracepoint、構造化された引数
fentry/fexit → 型付きの関数トレース、最小限のオーバーヘッド
```

結果として階層化されたシステムができあがった。安定した本番環境の計装にはtracepointを、深掘りする一回限りの調査にはkprobe/fentryを使い、生のイベントでユーザー空間を溢れさせないためにBPFプログラムを使う。

## kprobe: 任意の命令への動的プローブ {#kprobes-dynamic-probes-on-any-instruction}

kprobeは（関数の入口だけでなく）カーネル内の任意の命令にアタッチできる。ヒットすると、自分のコード内のハンドラが呼ばれる。

### kprobeの仕組み {#how-kprobes-work}

```c
/* kernel/kprobes.c: 簡略化したもの */

/* プローブ登録時: */
register_kprobe(kp)
  → kp->addr にある命令を見つける
  → 元の命令をコピーする（シングルステップ実行のため）
  → 先頭バイトを INT3（0xCC）ブレークポイントに置き換える
  → /* x86 の場合 */

/* 実行時、INT3 が発火したとき: */
do_int3()
  → kprobe_int3_handler()
      → kp->pre_handler(kp, regs) を呼ぶ
      → シングルステップを準備: 元の命令を復元し、TF フラグを立てる
      → /* 戻って元の命令をシングルステップ実行する */
  → シングルステップ後:
      → post_kprobe_handler()
          → kp->post_handler(kp, regs, 0) を呼ぶ
          → INT3 ブレークポイントを復元する
```

x86-64で`CONFIG_KPROBES_ON_FTRACE`が有効な場合、関数の入口へのkprobeはINT3の代わりに既存のftraceフックを使い、ブレークポイントのオーバーヘッドをなくす。

### カーネルモジュールからkprobeを登録する {#registering-kprobes-from-a-kernel-module}

```c
/* drivers/mymodule/mymodule.c */
#include <linux/kprobes.h>

static int pre_handler(struct kprobe *p, struct pt_regs *regs)
{
    /* プローブされた命令が実行される「前に」呼ばれる */
    /* regs にはプローブ地点でのレジスタ状態が入っている */
    printk(KERN_INFO "kprobe: hit tcp_sendmsg, rdi=0x%lx\n", regs->di);
    return 0;
}

static void post_handler(struct kprobe *p, struct pt_regs *regs,
                          unsigned long flags)
{
    /* プローブされた命令が（シングルステップで）実行された「後に」呼ばれる */
}

static struct kprobe kp = {
    .symbol_name = "tcp_sendmsg",  /* または .addr = (kprobe_opcode_t *)addr */
    .pre_handler = pre_handler,
    .post_handler = post_handler,
};

static int __init mymodule_init(void)
{
    int ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("register_kprobe failed: %d\n", ret);
        return ret;
    }
    pr_info("kprobe registered at %p\n", kp.addr);
    return 0;
}

static void __exit mymodule_exit(void)
{
    unregister_kprobe(&kp);
}
```

### kretprobe: 関数の戻りをフックする {#kretprobe-hook-on-function-return}

```c
static int entry_handler(struct kretprobe_instance *ri, struct pt_regs *regs)
{
    /* 関数の入口で呼ばれる。ri->data にデータを保存できる */
    *(ktime_t *)ri->data = ktime_get();
    return 0;
}

static int return_handler(struct kretprobe_instance *ri, struct pt_regs *regs)
{
    /* 関数の戻り時に呼ばれる */
    ktime_t start = *(ktime_t *)ri->data;
    long retval = regs_return_value(regs);
    ktime_t elapsed = ktime_sub(ktime_get(), start);

    pr_info("tcp_sendmsg returned %ld, took %lld ns\n",
            retval, ktime_to_ns(elapsed));
    return 0;
}

static struct kretprobe my_kretprobe = {
    .handler     = return_handler,
    .entry_handler = entry_handler,
    .data_size   = sizeof(ktime_t),  /* インスタンスごとのデータ */
    .maxactive   = 20,               /* 同時に存在できるインスタンスの最大数 */
    .kp.symbol_name = "tcp_sendmsg",
};
```

## 静的tracepoint: TRACE_EVENT {#static-tracepoints-trace_event}

静的tracepointは、カーネル内の特定の場所にコンパイルされて組み込まれるアノテーションポイントである。次の特徴を持つ。

- 無効時のオーバーヘッドはゼロ（何もしないテストが1つだけ）
- 豊富な構造化データ（単なるレジスタダンプではない）
- 安定したABI（関数名に依存するkprobeとは異なる）

### TRACE_EVENTマクロ {#trace_event-macro}

```c
/* include/trace/events/sched.h */
TRACE_EVENT(sched_switch,

    /* tracepoint 呼び出しの引数: */
    TP_PROTO(bool preempt,
             struct task_struct *prev,
             struct task_struct *next,
             unsigned int prev_state),

    /* 実際の呼び出し引数: */
    TP_ARGS(preempt, prev, next, prev_state),

    /* リングバッファに格納されるフィールド: */
    TP_STRUCT__entry(
        __array(        char,   prev_comm,  TASK_COMM_LEN   )
        __field(        pid_t,  prev_pid                    )
        __field(        int,    prev_prio                   )
        __field(        long,   prev_state                  )
        __array(        char,   next_comm,  TASK_COMM_LEN   )
        __field(        pid_t,  next_pid                    )
        __field(        int,    next_prio                   )
    ),

    /* リングバッファのエントリにデータをコピーするコード: */
    TP_fast_assign(
        memcpy(__entry->next_comm, next->comm, TASK_COMM_LEN);
        __entry->prev_pid   = prev->pid;
        __entry->prev_prio  = prev->prio;
        __entry->prev_state = __trace_sched_switch_state(preempt, prev_state, prev);
        memcpy(__entry->prev_comm, prev->comm, TASK_COMM_LEN);
        __entry->next_pid   = next->pid;
        __entry->next_prio  = next->prio;
    ),

    /* 人間が読める出力のためのフォーマット文字列: */
    TP_printk("prev_comm=%s prev_pid=%d prev_prio=%d prev_state=%s%s ==> "
              "next_comm=%s next_pid=%d next_prio=%d",
              __entry->prev_comm, __entry->prev_pid, __entry->prev_prio,
              ...)
);
```

### 自分のコードにtracepointを追加する {#adding-a-tracepoint-to-your-code}

```c
/* 1. tracepoint を定義する（ヘッダ内で一度だけ）: */
/* include/trace/events/mysubsystem.h */
#undef TRACE_SYSTEM
#define TRACE_SYSTEM mysubsystem

#if !defined(_TRACE_MYSUBSYSTEM_H) || defined(TRACE_HEADER_MULTI_READ)
#define _TRACE_MYSUBSYSTEM_H

#include <linux/tracepoint.h>

TRACE_EVENT(my_event,
    TP_PROTO(int value, const char *name),
    TP_ARGS(value, name),
    TP_STRUCT__entry(
        __field(int, value)
        __string(name, name)  /* 可変長の文字列 */
    ),
    TP_fast_assign(
        __entry->value = value;
        __assign_str(name, name);
    ),
    TP_printk("value=%d name=%s", __entry->value, __get_str(name))
);

#endif /* _TRACE_MYSUBSYSTEM_H */
#include <trace/define_trace.h>

/* 2. ソースファイルで使う: */
/* kernel/mysubsystem.c */
#define CREATE_TRACE_POINTS
#include <trace/events/mysubsystem.h>

void my_function(int val, const char *name)
{
    /* tracepoint: 無効時はオーバーヘッドゼロ */
    trace_my_event(val, name);
    /* ... */
}
```

### tracepointがオーバーヘッドゼロを実現する仕組み {#how-tracepoints-achieve-zero-overhead}

```c
/* TRACE_EVENT によって生成されるコード: */
static inline void trace_my_event(int value, const char *name)
{
    /* 分岐 1 つだけ: 呼び出し先が登録されているかを確認する */
    if (static_key_false(&__tracepoint_my_event.key)) {
        /* 有効なときだけオーバーヘッドが生じる */
        __tracepoint_my_event.funcs(value, name);
    }
}
```

`static_key_false`は（jump labelによって）パッチされるnop/jmp命令を使う。誰も聞いていないときは、`if`ブロック全体が実行時にパッチで取り除かれる。

## uprobe: ユーザー空間の動的プローブ {#uprobes-userspace-dynamic-probes}

uprobeはkprobeと同様に動作するが、ユーザー空間のバイナリが対象であり、Linux 3.5で導入された[（コミット）](https://git.kernel.org/linus/2b144498350860b6ee9dc57ff27a93ad488de5dc)。カーネルはマップされたページにブレークポイントを挿入する。

```bash
# 任意のプロセスにおける malloc へのすべての呼び出しをトレースする
echo 'p:malloc /lib/x86_64-linux-gnu/libc.so.6:0x9d430' \
    > /sys/kernel/tracing/uprobe_events
echo 1 > /sys/kernel/tracing/events/uprobes/malloc/enable

# シンボル名でトレースする（デバッグシンボルが必要）
echo 'p:my_func /usr/bin/myapp:my_function' \
    > /sys/kernel/tracing/uprobe_events
```

```c
/* カーネルの struct uprobe */
struct uprobe {
    struct rb_node  rb_node;   /* inode の uprobe ツリー */
    refcount_t      ref;
    struct rw_semaphore register_rwsem;
    struct rw_semaphore consumer_rwsem;
    struct list_head   pending_list;
    struct uprobe_consumer *consumers;
    struct inode       *inode;   /* 対象バイナリの inode */
    loff_t              offset;  /* ファイル内のオフセット */
    loff_t              ref_ctr_offset;
    unsigned long       flags;
    struct arch_uprobe  arch;    /* アーキテクチャ固有の、保存された命令 */
};
```

## USDT: ユーザー空間の静的定義トレースポイント {#usdt-userspace-statically-defined-tracepoints}

カーネルのTRACE_EVENTと同様に、USDTプローブはユーザー空間コード内の静的マーカーである。uprobe機構を介したUSDTプローブへのBPFのアタッチは、Linux 4.8で利用可能になった。

```c
/* C コード内で（systemtap-sdt-dev または dtrace のプローブが必要）: */
#include <sys/sdt.h>
DTRACE_PROBE2(myapp, request_start, req_id, req_type);
/* ... */
DTRACE_PROBE1(myapp, request_end, req_id);
```

```bash
# バイナリ内の USDT プローブを一覧表示する
readelf --notes /usr/bin/python3 | grep stapsdt

# bpftrace でトレースする
bpftrace -e 'usdt:/usr/bin/python3:function__entry { printf("%s\n", str(arg1)); }'

# perf でトレースする
perf probe -x /usr/bin/python3 sdt_pythonX_Y:function__entry
```

## プローブの動作を観察する {#observing-probe-activity}

```bash
# kprobe のヒット数を数える
cat /sys/kernel/tracing/kprobe_profile
# tcp_sendmsg        42         0  ← ヒット数、ミス数

# イベントのヒット数
cat /sys/kernel/tracing/events/syscalls/sys_enter_read/hist

# bpftrace で集計する
bpftrace -e 'kprobe:tcp_sendmsg { @[comm] = count(); }'
# Attaching 1 probe...
# ^C
# @[nginx]: 1234
# @[postgres]: 567

# kprobe を使った動的な function graph
echo function_graph > /sys/kernel/tracing/current_tracer
echo 'tcp_sendmsg' > /sys/kernel/tracing/set_graph_function
echo 1 > /sys/kernel/tracing/tracing_on
```

## 参考文献 {#further-reading}

- [ftrace](ftrace.md) — リングバッファとtracefsインターフェース
- [perfイベント](perf-events.md) — ハードウェアカウンタとサンプリング
- [BPF: アーキテクチャ](../bpf/bpf-overview.md) — kprobe/tracepointのBPFプログラム
- `Documentation/trace/kprobes.rst` — カーネルのkprobeドキュメント
- `Documentation/trace/tracepoints.rst` — 静的tracepointのドキュメント
