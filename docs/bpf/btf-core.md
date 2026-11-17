# BTFとCO-RE {#btf-and-co-re}

> 異なるカーネルバージョンをまたいで動くBPFプログラムを書く

## BTFが解決する問題 {#the-problem-btf-solves}

カーネルのデータ構造はバージョン間で変化する。カーネル5.15向けにコンパイルされたBPFプログラムは、構造体が再編成されていた場合、カーネル6.1上では`struct task_struct`に誤ったオフセットでアクセスしてしまうかもしれない。

BTF/CO-RE以前は、BPFプログラムは特定バージョンのカーネルヘッダを使ってコンパイルされ、そのバージョン上でしか動かなかった（あるいはカーネルごとに再コンパイルが必要だった）。

Andrii Nakryikoが開発した**CO-RE**（Compile Once, Run Everywhere）を使うと、BPFプログラムを一度コンパイルするだけで、BTFを持つ5.8以上の任意のカーネルバージョンで動作させられる。

## BTF: BPF Type Format {#btf-bpf-type-format}

BTFはコンパクトな型情報フォーマットであり、特別なELFセクション（`.BTF`）に格納され、BPFプログラムとともにカーネルにロードされる。Martin KaFai LauによってLinux 4.18で導入された。[(commit)](https://git.kernel.org/linus/69b693f0aefa0ed521e8bd02260523b5ae446ad7)

```bash
# カーネルがBTFを持っているか確認する
ls /sys/kernel/btf/vmlinux
# 存在すれば、カーネルは自身の完全な型情報を公開している

# vmlinux.hを生成する（全カーネル型をCヘッダとして出力）
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

### BTFの型エンコーディング {#btf-type-encoding}

```c
/* BTFは型を型記述子の並びとして表現する */
struct btf_type {
    __u32 name_off;    /* 文字列テーブル内のオフセット */
    /* infoのエンコーディング:
     * bits 0-15:  vlen（構造体メンバ数など）
     * bits 16-23: 未使用
     * bits 24-28: kind（BTF_KIND_INT, STRUCT, ARRAY, FUNC, ...）
     * bit 31:     kind_flag
     */
    __u32 info;
    union {
        __u32 size;   /* INT, STRUCT, UNIONの場合: バイト単位のサイズ */
        __u32 type;   /* PTR, TYPEDEF, VOLATILEの場合: 参照先の型ID */
    };
};

/* STRUCT/UNIONではbtf_typeの後に構造体メンバが続く */
struct btf_member {
    __u32 name_off;  /* 文字列テーブル内のオフセット */
    __u32 type;      /* このメンバの型ID */
    __u32 offset;    /* ビットオフセット（またはビットフィールド指定） */
};
```

カーネルは`BTF_LOAD` bpf()コマンドでBTFをロードし、次の用途に使う。

- Verifier向け: ポインタが指す構造体のレイアウトを理解するため
- CO-REリロケーション向け: プログラムのロード時にフィールドオフセットをパッチするため
- bpftool/libbpf向け: mapの値やプログラムのコンテキストを整形表示するため
- BPFイテレータとkfunc向け: 関数シグネチャ

## vmlinux.h: 全カーネル型を1つのヘッダに {#vmlinuxh-all-kernel-types-in-one-header}

個々のカーネルヘッダをインクルードする代わりに、CO-REプログラムでは通常、生成された`vmlinux.h`を使う。

```bash
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

この単一のヘッダには、実行中のカーネルのすべての構造体、共用体、列挙型、typedefが含まれる — BTFから抽出されたものである。

```c
/* BPFプログラムでは、次のように書く代わりに: */
#include <linux/sched.h>
#include <linux/fs.h>
#include <linux/mm_types.h>

/* こう書く: */
#include "vmlinux.h"         /* BTF由来の全カーネル型 */
#include <bpf/bpf_helpers.h> /* BPFヘルパーの宣言 */
#include <bpf/bpf_core_read.h> /* CO-REマクロ */
```

## CO-REによるフィールドアクセス {#co-re-field-access}

CO-REの中核となる仕組みは`BPF_CORE_READ`であり、libbpfがロード時にパッチするリロケーションを記録する。

```c
#include "vmlinux.h"
#include <bpf/bpf_core_read.h>

SEC("kprobe/do_sys_open")
int trace_open(struct pt_regs *ctx)
{
    struct task_struct *task = (struct task_struct *)bpf_get_current_task();

    /* CO-REなし: 固定オフセット。カーネルバージョンが変わると壊れる */
    pid_t pid = *(pid_t *)((char *)task + offsetof(struct task_struct, pid));

    /* CO-REあり: オフセットはロード時にBTFから解決される */
    pid_t pid = BPF_CORE_READ(task, pid);

    /* ネストしたフィールドへのアクセス */
    u32 ns_inum = BPF_CORE_READ(task, nsproxy, pid_ns_for_children, ns.inum);

    bpf_printk("open by pid=%d ns=%u\n", pid, ns_inum);
    return 0;
}
```

`BPF_CORE_READ(ptr, field)`は、リロケーションのメタデータを特別な命令としてBPFバイトコードに埋め込んだ安全な読み出しに展開される。libbpfはプログラムをロードする際に次を行う。
1. `/sys/kernel/btf/vmlinux`からターゲットカーネルのBTFを読む
2. そのカーネルにおける`task_struct.pid`の実際のオフセットを調べる
3. 命令を正しいオフセットでパッチする

### CO-RE読み出しのバリエーション {#co-re-read-variants}

```c
/* 基本的な読み出し */
BPF_CORE_READ(ptr, field)

/* 変数に読み込む */
BPF_CORE_READ_INTO(&dest, ptr, field)

/* ビットフィールドの読み出し（ビットフィールドのパッキングの違いを吸収する） */
BPF_CORE_READ_BITFIELD(ptr, field)
BPF_CORE_READ_BITFIELD_PROBED(ptr, field)

/* 文字列の読み出し */
BPF_CORE_READ_STR_INTO(dest, ptr, field)

/* ターゲットカーネルにフィールドが存在するか確認する */
bpf_core_field_exists(ptr->field)   /* 0または1を返す */
bpf_core_type_exists(struct foo)

/* フィールドのサイズを確認する（型の変更に対応する） */
bpf_core_field_size(ptr->field)
```

### 列挙値に対するCO-RE {#co-re-for-enum-values}

列挙値もカーネル間で変わりうる。CO-REはこれにも対応する。

```c
/* カーネルが名前の変わった列挙値を使っているか確認する */
u32 state;
if (bpf_core_enum_value_exists(enum task_state, TASK_DEAD))
    state = BPF_CORE_READ(task, state);
else
    state = BPF_CORE_READ(task, __state);  /* 5.14で名前が変わった */
```

## ロード時のカーネルバージョンチェック {#kernel-version-checks-at-load-time}

追加または削除された機能の場合:

```c
/* カーネルバージョンに基づく条件付きコンパイル */
extern int LINUX_KERNEL_VERSION __kconfig;

SEC("kprobe/...")
int my_prog(struct pt_regs *ctx)
{
    if (LINUX_KERNEL_VERSION >= KERNEL_VERSION(5, 16, 0)) {
        /* 新しいフィールドやAPIを使う */
    } else {
        /* 古いカーネル向けのフォールバック */
    }
}
```

`extern ... __kconfig`構文は`/proc/config.gz`や`Kconfig`シンボルから値を読み出すものであり、これもlibbpfがロード時に解決する。

## BTFベースのトレーシング: fentry/fexit {#btf-based-tracing-fentryfexit}

BTFは、kprobeよりも効率的な代替手段である`fentry`および`fexit`プログラムを可能にする。

```c
/* fentry: 関数引数の書き換えは不要。
   BTFが型情報を直接提供する */
SEC("fentry/tcp_sendmsg")
int BPF_PROG(trace_tcp_sendmsg,
             struct sock *sk,      /* BTFによれば: 第1引数はsock* */
             struct msghdr *msg,   /* 第2引数はmsghdr* */
             size_t size)          /* 第3引数はsize_t */
{
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    u64 bytes = BPF_CORE_READ(sk, sk_sndbuf);
    bpf_printk("tcp_sendmsg: pid=%u sndbuf=%llu size=%zu\n",
               pid, bytes, size);
    return 0;
}

/* fexit: 戻り値も受け取れる */
SEC("fexit/tcp_sendmsg")
int BPF_PROG(trace_tcp_sendmsg_ret,
             struct sock *sk, struct msghdr *msg, size_t size,
             int ret)   /* 戻り値は最後の引数として追加される */
{
    if (ret < 0)
        bpf_printk("tcp_sendmsg failed: %d\n", ret);
    return 0;
}
```

`fentry`/`fexit`はトランポリン — BPF JITコードへの直接呼び出し — を使い、オーバーヘッドはkprobeの10〜100分の1である。

## mapにおけるBTF: 型付きのmap値 {#btf-in-maps-typed-map-values}

BTFを使うと、mapの値を自己記述的にできる。

```c
struct event {
    u32 pid;
    u32 uid;
    char comm[16];
} __attribute__((preserve_access_index));  /* この構造体でCO-REを有効にする */

struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 1 << 24);
} events SEC(".maps");
```

mapにBTFがアタッチされている場合:
```bash
# bpftoolはBTFを使ってmapの値を整形表示できる
bpftool map dump id 42
# [{
#     "pid": 1234,
#     "uid": 0,
#     "comm": "bash"
# }]
```

## BTFを観察する {#observing-btf}

```bash
# カーネルにロードされているすべてのBTFオブジェクトを一覧表示する
bpftool btf list
# 1: name [vmlinux]  size 5.5MB
# 42: name [bpf_prog] size 1.2KB

# 型情報をダンプする
bpftool btf dump id 1 format raw | grep -A5 "task_struct"

# BTFから構造体のレイアウトを表示する
bpftool btf dump file /sys/kernel/btf/vmlinux format c | \
    grep -A 30 "^struct task_struct {"

# BPFオブジェクトのBTFを検証する
bpftool prog list
bpftool btf dump id <btf_id>
```

## 移植性のあるBPFプログラムを書くためのチェックリスト {#writing-portable-bpf-programs-checklist}

1. カーネルヘッダの代わりに`vmlinux.h`を使う
2. カーネル構造体のフィールドへのアクセスにはすべて`BPF_CORE_READ()`を使う
3. 5.2以降に追加されたフィールドには`bpf_core_field_exists()`を使う
4. 振る舞いの違いには`LINUX_KERNEL_VERSION`によるガードを使う
5. `clang -target bpf -g`でコンパイルする（`-g`は.oにBTFを埋め込む）
6. ロードにはlibbpfを使う（CO-REリロケーションを処理してくれる）

```bash
# BTFを埋め込んでコンパイルする
clang -target bpf -O2 -g \
    -I/usr/include/bpf \
    -c my_prog.bpf.c -o my_prog.bpf.o

# オブジェクトにBTFが存在することを確認する
llvm-readelf --sections my_prog.bpf.o | grep BTF
# .BTF         PROGBITS  ...
# .BTF.ext     PROGBITS  ...  (line number info for verifier)
```

## 参考文献 {#further-reading}

- [libbpfとスケルトン](libbpf.md) — libbpfのロードワークフローでCO-REを使う
- [BPF Verifier](bpf-verifier.md) — BTFがVerifierの型チェックをどう改善するか
- カーネルツリーの`tools/lib/bpf/` — libbpfのソース
- `Documentation/bpf/btf.rst` — BTFフォーマットの仕様
