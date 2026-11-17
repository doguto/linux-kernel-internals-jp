# BPFのアーキテクチャとプログラムタイプ {#bpf-architecture-and-program-types}

> eBPFプログラムはどのようにロードされ、検証され、カーネルフックにアタッチされるのか

## cBPFからeBPFへ {#from-cbpf-to-ebpf}

### Classic BPF（1993〜2014） {#classic-bpf-19932014}

BPFはもともとLinuxの発明ではない。Lawrence Berkeley LaboratoryのSteven McCanneとVan JacobsonがUSENIX Winter 1993で発表した論文 *"The BSD Packet Filter: A New Architecture for User-level Packet Capture."* に由来する。[(paper)](https://www.tcpdump.org/papers/bpf-usenix93.pdf)

元のBPFは、2つの32ビットレジスタ（`A`と`X`）、小さなスクラッチメモリ配列、そして単純な命令セットを持つ小さな仮想マシンであった。その命令セットはただ1つの目的、すなわちネットワークパケットをユーザー空間にコピーする前にカーネル内でフィルタリングするために設計されていた。Linuxはこれを`SO_ATTACH_FILTER`として採用した — ソケットにフィルタをアタッチし、関心のないパケットを`recvmsg()`に届く前にドロップするのである。

それが目的のすべてであった。実装はほぼ完全に`net/core/filter.c`の中にあった。

### cBPFの問題点 {#the-problem-with-cbpf}

2000年代から2010年代にかけてLinuxカーネルの開発が加速するにつれ、複数のサブシステムが、ユーザー空間に任意のカーネルコード実行を許すことなく、ユーザー空間がカーネル内でポリシーを表現できる手段を必要とするようになった。

- **seccomp**（2005年）: システムコールをフィルタリングしたかった。cBPFをそのまま使ったが、2レジスタのモデルは現実のポリシーには扱いにくかった。
- **tc**（トラフィック制御）: 単なるフィルタリングではなく、プログラム可能なパケット分類と改変が必要だった。
- **perf/トレーシング**: 生のイベントをユーザー空間にコピーするのを避けるため、プローブポイントに独自の集計ロジックをアタッチしたかった。

カーネルモジュールならこれらすべてが可能だが、危険である。バグのあるモジュールはカーネルをパニックさせるし、CAP_SYS_MODULEが必要であり、特定のカーネルバージョンに縛られる。「信頼できない」カーネルコードを安全に実行する方法は存在しなかった。

### eBPF（Linux 3.18、2014年） {#ebpf-linux-318-2014}

Alexei StarovoitovとDaniel BorkmannはBPF仮想マシンをゼロから書き直した。新しい設計 — 後からeBPF（extended BPF）と呼ばれるようになった [(commit)](https://git.kernel.org/linus/99c55f7d47c0dc6fc64729f37bf435abf43f4c60) [(LWN)](https://lwn.net/Articles/603983/) — は、cBPFのソケットフィルタとの後方互換性を保ちつつ（それらは自動的に変換される）、それ以外の点では新しいアーキテクチャであった。

| | cBPF | eBPF |
|--|------|------|
| レジスタ | 2個（32ビットのA、X） | 11個（64ビットのr0〜r10） |
| 命令長 | 可変 | 64ビット固定 |
| 共有状態 | なし | map（ハッシュ、配列、リングバッファ…） |
| アタッチ箇所 | ソケットフィルタのみ | 30以上のプログラムタイプ |
| Verifier | なし（単純なバリデータ） | 完全な安全性証明 |
| JIT | 一部 | 主要アーキテクチャすべて |
| ユーザー空間とカーネルの共有 | 不可 | 可（map、BTF） |

eBPFを安全にしているのは**Verifier**である。Verifierは、BPFプログラム内のあらゆる実行パスをたどり、プログラムが停止すること、不正なポインタを参照外ししないこと、許可された領域の外のメモリにアクセスしないことを証明する。Verifierが拒否したプログラムは決して実行されない。

**map**という抽象化が共有状態の問題を解決した。BPFプログラムはカーネル側のデータ構造に統計情報（IPごとのカウンタ、レイテンシのヒストグラム）を蓄積でき、ユーザー空間は`bpf()`システムコールを介してその構造を読み出せる — カーネルメモリへの生のアクセスは不要である。

「なぜカーネルモジュールではないのか?」への答えは、今や具体的である。Verifierを通過したBPFプログラムはカーネルをクラッシュさせられず、`CAP_SYS_MODULE`なしでロードでき、ネイティブに近い速度にJITコンパイルされる。

## bpf()システムコール {#the-bpf-syscall}

eBPFのすべては単一のシステムコールを通る。

```c
#include <linux/bpf.h>

int bpf(int cmd, union bpf_attr *attr, unsigned int size);
```

主要なコマンド:

```c
BPF_PROG_LOAD      /* BPFプログラムをロードして検証する */
BPF_MAP_CREATE     /* BPF mapを作成する */
BPF_MAP_LOOKUP_ELEM
BPF_MAP_UPDATE_ELEM
BPF_MAP_DELETE_ELEM
BPF_OBJ_PIN        /* map/プログラムを/sys/fs/bpf/にピン留めする */
BPF_OBJ_GET        /* ピン留めされたオブジェクトを取り出す */
BPF_PROG_ATTACH    /* プログラムをcgroup/フックにアタッチする */
BPF_PROG_DETACH
BPF_LINK_CREATE    /* BPFリンク（管理されたアタッチ）を作成する */
BPF_LINK_UPDATE
BTF_LOAD           /* BTF型情報をロードする */
```

## プログラムのロード {#loading-a-program}

```c
/* kernel/bpf/syscall.c: bpf_prog_load() */
union bpf_attr attr = {
    .prog_type    = BPF_PROG_TYPE_SOCKET_FILTER,
    .insn_cnt     = prog_len / sizeof(struct bpf_insn),
    .insns        = (uint64_t)prog_insns,
    .license      = (uint64_t)"GPL",
    .log_level    = 1,          /* Verifierログを要求する */
    .log_buf      = (uint64_t)log_buf,
    .log_size     = LOG_BUF_SIZE,
    .kern_version = 0,          /* kprobeの場合、カーネルと一致しなければならない */
};

int prog_fd = bpf(BPF_PROG_LOAD, &attr, sizeof(attr));
/* ファイルディスクリプタを返す。失敗時は-1を返しerrnoを設定する */
```

BPF_PROG_LOAD後のカーネル内の流れ:

```
bpf_prog_load()
  → bpf_check_uarg_tail_zero()    /* attrのパディングを検証する */
  → find_prog_type()              /* タイプのopsを検索する */
  → bpf_prog_alloc()              /* struct bpf_progを割り当てる */
  → copy_from_bpfptr(insns)       /* ユーザー空間からバイトコードをコピーする */
  → bpf_check()                   /* VERIFIER — 安全性の証明 */
  → bpf_prog_select_runtime()     /* JITかインタプリタかを選択する */
  → bpf_prog_kallsyms_add()       /* /proc/kallsymsに公開する */
  → anon_inode_getfd("bpf-prog")  /* ファイルディスクリプタを返す */
```

## struct bpf_prog {#struct-bpf_prog}

ロードされたBPFプログラムを表す中心的なカーネルオブジェクト:

```c
/* include/linux/filter.h */
struct bpf_prog {
    u16             pages;          /* ページ数 */
    u16             jited:1,        /* JITコンパイル済みなら真 */
                    jit_requested:1,
                    gpl_compatible:1,
                    cb_access:1,    /* skb->cb領域にアクセスする */
                    dst_needed:1,
                    blinding_requested:1,
                    blinded:1,
                    is_func:1,      /* プログラム配列内の関数 */
                    kprobe_override:1,
                    has_callchain_buf:1,
                    enforce_expected_attach_type:1,
                    call_get_stack:1,
                    call_get_func_ip:1,
                    tstamp_type_access:1;
    enum bpf_prog_type      type;   /* BPF_PROG_TYPE_* */
    enum bpf_attach_type    expected_attach_type;
    u32             len;            /* フィルタブロックの数 */
    u32             jited_len;      /* JITコードのサイズ */
    u8              tag[BPF_TAG_SIZE]; /* SHA1ベースの識別子 */
    struct bpf_prog_stats __percpu *stats;
    struct bpf_prog_aux    *aux;    /* Verifierの状態、map、BTF */
    struct sock_fprog_kern *orig_prog; /* 元のClassic BPF */
    unsigned int    (*bpf_func)(const void *ctx,
                                const struct bpf_insn *insn);
    /* この後にJIT/インタプリタのコードが続く */
    union {
        struct sock_filter  insns[0]; /* Classic BPFの命令 */
        struct bpf_insn     insnsi[]; /* eBPFの命令 */
    };
};
```

## BPF命令セット {#bpf-instruction-set}

eBPFは11個の64ビットレジスタ（r0〜r10）と固定サイズの命令フォーマットを持つ。

```c
/* include/linux/filter.h */
struct bpf_insn {
    __u8    code;       /* オペコード */
    __u8    dst_reg:4;  /* デスティネーションレジスタ */
    __u8    src_reg:4;  /* ソースレジスタ */
    __s16   off;        /* 符号付きオフセット */
    __s32   imm;        /* 符号付き即値 */
};
```

レジスタの規約:

- `r0` — ヘルパー呼び出しの戻り値とプログラム終了時の戻り値
- `r1`〜`r5` — 関数の引数（エントリ時はr1 = コンテキストポインタ）
- `r6`〜`r9` — callee-savedレジスタ
- `r10` — フレームポインタ（読み取り専用）

## プログラムタイプ {#program-types}

各プログラムタイプは、固有のコンテキスト構造体と使用を許可されたヘルパーを持つ。

```c
/* include/uapi/linux/bpf.h */
enum bpf_prog_type {
    BPF_PROG_TYPE_SOCKET_FILTER,    /* ソケットフィルタ、ctx=sk_buff */
    BPF_PROG_TYPE_KPROBE,           /* kprobe/uprobe、ctx=pt_regs */
    BPF_PROG_TYPE_SCHED_CLS,        /* TC分類器、ctx=sk_buff */
    BPF_PROG_TYPE_SCHED_ACT,        /* TCアクション */
    BPF_PROG_TYPE_TRACEPOINT,       /* 静的tracepoint */
    BPF_PROG_TYPE_XDP,              /* XDP、ctx=xdp_md */
    BPF_PROG_TYPE_PERF_EVENT,       /* PMU/perfイベント */
    BPF_PROG_TYPE_CGROUP_SKB,       /* cgroupのソケットバッファ */
    BPF_PROG_TYPE_CGROUP_SOCK,      /* cgroupのソケット作成 */
    BPF_PROG_TYPE_LWT_IN,           /* 軽量トンネル */
    BPF_PROG_TYPE_SOCK_OPS,         /* ソケットイベントのコールバック */
    BPF_PROG_TYPE_SK_SKB,           /* ソケットmapによる振り分け */
    BPF_PROG_TYPE_CGROUP_DEVICE,    /* デバイスアクセス制御 */
    BPF_PROG_TYPE_SK_MSG,           /* sockmapのメッセージリダイレクト */
    BPF_PROG_TYPE_RAW_TRACEPOINT,   /* raw tracepoint（引数の書き換えなし） */
    BPF_PROG_TYPE_CGROUP_SOCK_ADDR, /* cgroupのbind/connect/sendmsg */
    BPF_PROG_TYPE_LWT_SEG6LOCAL,
    BPF_PROG_TYPE_LIRC_MODE2,       /* 赤外線デコード */
    BPF_PROG_TYPE_SK_REUSEPORT,     /* SO_REUSEPORTの振り分け */
    BPF_PROG_TYPE_FLOW_DISSECTOR,   /* 独自のフロー解析 */
    BPF_PROG_TYPE_CGROUP_SYSCTL,    /* sysctlのアクセス制御 */
    BPF_PROG_TYPE_RAW_TRACEPOINT_WRITABLE,
    BPF_PROG_TYPE_CGROUP_SOCKOPT,
    BPF_PROG_TYPE_TRACING,          /* fentry/fexit/fmod_ret */
    BPF_PROG_TYPE_STRUCT_OPS,       /* カーネルのstruct opsを実装する */
    BPF_PROG_TYPE_EXT,              /* 関数の置き換え */
    BPF_PROG_TYPE_LSM,              /* Linux Security Moduleのフック */
    BPF_PROG_TYPE_SK_LOOKUP,        /* ソケット検索 */
    BPF_PROG_TYPE_SYSCALL,          /* 特権システムコールプログラム */
    BPF_PROG_TYPE_NETFILTER,        /* netfilterフック */
};
```

### トレーシングプログラムの詳細 {#tracing-programs-in-detail}

```c
/* fentry: 関数の入口にアタッチする。kprobeのオーバーヘッドがない */
SEC("fentry/tcp_sendmsg")
int BPF_PROG(trace_tcp_sendmsg, struct sock *sk,
             struct msghdr *msg, size_t size)
{
    bpf_printk("tcp_sendmsg: size=%zu\n", size);
    return 0;
}

/* fexit: 関数のリターンにアタッチする。戻り値を見られる */
SEC("fexit/tcp_sendmsg")
int BPF_PROG(trace_tcp_sendmsg_exit, struct sock *sk,
             struct msghdr *msg, size_t size, int ret)
{
    bpf_printk("tcp_sendmsg returned %d\n", ret);
    return 0;
}

/* kprobe: インライン化されていない任意の関数にアタッチできる */
SEC("kprobe/tcp_sendmsg")
int kprobe_tcp_sendmsg(struct pt_regs *ctx)
{
    struct sock *sk = (struct sock *)PT_REGS_PARM1(ctx);
    /* 引数にはPT_REGS_PARM1..5でアクセスする */
    return 0;
}
```

### XDPプログラム {#xdp-program}

```c
/* 最小のXDPプログラム */
SEC("xdp")
int xdp_drop_icmp(struct xdp_md *ctx)
{
    void *data_end = (void *)(long)ctx->data_end;
    void *data     = (void *)(long)ctx->data;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return XDP_PASS;

    if (eth->h_proto != htons(ETH_P_IP))
        return XDP_PASS;

    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end)
        return XDP_PASS;

    if (ip->protocol == IPPROTO_ICMP)
        return XDP_DROP;  /* ICMPをドロップする */

    return XDP_PASS;
}
```

XDPの戻り値コード:

- `XDP_ABORTED` — バグ/エラー。パケットをドロップし、tracepointを発火させる
- `XDP_DROP` — 即座にドロップする（skb割り当て前）
- `XDP_PASS` — 通常のネットワークスタックに渡す
- `XDP_TX` — 同じインターフェースから送り返す
- `XDP_REDIRECT` — 別のインターフェースまたはAF_XDPソケットへリダイレクトする

## BPFプログラムのライフサイクル {#bpf-program-lifecycle}

```
Load: bpf(BPF_PROG_LOAD)  → prog_fd (refcount=1)
                              │
Attach: bpf_link_create()  → link_fd
        or explicit attach      │
                              │   prog refcount++
                              │
Running: hook fires        → bpf_prog_run(prog, ctx)
                              │
Detach: close(link_fd)     → link cleanup, prog refcount--
Unload: close(prog_fd)     → prog refcount--
                              │
refcount==0:               → bpf_prog_free()
                              JIT memory freed
```

BPFリンク（`BPF_LINK_CREATE`）は、次の理由から従来のアタッチ方法よりも推奨される。

- link_fdを閉じると自動的にデタッチされる（宙に浮いたプログラムが残らない）
- リンクは、元のローダープロセスが終了しても存続する（ピン留めされている場合）
- `BPF_LINK_UPDATE`によるアトミックなプログラムの置き換え

## ヘルパー関数 {#helper-functions}

BPFプログラムは任意のカーネル関数を呼べない — ホワイトリスト化されたヘルパーの集合を使う。

```c
/* 一般的なヘルパー */
bpf_map_lookup_elem(map, key)       → value pointer or NULL
bpf_map_update_elem(map, key, val, flags)
bpf_map_delete_elem(map, key)

bpf_ktime_get_ns()                  → nanoseconds since boot
bpf_get_current_pid_tgid()         → (tgid << 32) | pid
bpf_get_current_comm(buf, sz)       → process name string
bpf_get_current_task()             → struct task_struct *

bpf_probe_read_kernel(dst, sz, src) → safe kernel memory read
bpf_probe_read_user(dst, sz, src)   → safe userspace memory read
bpf_probe_read_kernel_str(dst, sz, src)

bpf_printk(fmt, ...)               → /sys/kernel/debug/tracing/trace_pipe

/* ネットワーキング */
bpf_skb_load_bytes(skb, off, to, len)
bpf_skb_store_bytes(skb, off, from, len, flags)
bpf_redirect(ifindex, flags)
bpf_fib_lookup(ctx, params, plen, flags)

/* リングバッファ */
bpf_ringbuf_reserve(map, size, flags)
bpf_ringbuf_submit(data, flags)
bpf_ringbuf_output(map, data, size, flags)
```

各ヘルパーには安定した番号が割り当てられる。新しいヘルパーには新しい番号が与えられ、ABIの安定性のために既存の番号が変更されることは決してない。

## BPFプログラムの観察 {#observing-bpf-programs}

```bash
# ロードされているすべてのBPFプログラムを一覧表示する
bpftool prog list
# 42: xdp  name xdp_drop_icmp  tag a04f5eef06a7f555  gpl
#         loaded_at 2024-01-15T10:00:00+0000  uid 0
#         xlated 112B  jited 88B  memlock 4096B  map_ids 3

# 逆アセンブルを表示する（eBPFまたはJIT）
bpftool prog dump xlated id 42
bpftool prog dump jited id 42

# プログラムのmapを表示する
bpftool prog show id 42 --pretty

# BPFの統計
bpftool prog profile id 42 duration 5 cycles instructions

# トレース出力
cat /sys/kernel/debug/tracing/trace_pipe
```

## 参考文献 {#further-reading}

- [BPF Maps](bpf-maps.md) — BPFとユーザー空間の間でのデータ共有
- [BPF Verifier](bpf-verifier.md) — 安全性がどのように強制されるか
- [BTFとCO-RE](btf-core.md) — 移植性のあるBPFプログラムを書く
- `kernel/bpf/syscall.c` — bpf()システムコールの実装
- カーネルツリーの`samples/bpf/` — ツリー内のサンプル
