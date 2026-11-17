# 32ビット互換syscall {#32-bit-compat-syscalls}

> 64ビットカーネル上での32ビットユーザー空間のサポート

## なぜcompatが存在するのか {#why-compat-exists}

64ビットカーネルは、32ビットのELFバイナリを直接実行できる（x86-64では `CONFIG_IA32_EMULATION` による）。問題は、
32ビットと64ビットのABIが、レジスタ幅を超えたところで異なっていることである。

- **ポインタのサイズ**: 32ビットのユーザー空間は4バイトのポインタを使い、カーネルは8バイトのポインタを使う。
- **`time_t` と `off_t`**: 32ビットABIではこれらは `int`（32ビット）として定義され、64ビットABIでは `long`（64ビット）が使われる。
- **構造体のレイアウト**: `long`、`size_t`、ポインタのメンバーを含む構造体は、パディングとサイズが異なる。
- **引数の受け渡し**: 32ビットx86は `int 0x80` のパスか `sysenter` を使い、引数は32ビットレジスタで渡される。

32ビットプロセスが `read()` を呼ぶと、`buf` には32ビットのポインタが渡される。カーネルはそれを64ビットの
ポインタとして直接扱うことはできない — 値をゼロ拡張し、32ビットのアドレス空間内で検証しなければならない。`read()` のような
単純なsyscallでは、違いはわずかである。`time_t`、`off_t`、あるいは埋め込みポインタを含む構造体を渡すsyscall
（`sendmsg()`、`stat()`、`select()` など）では、32ビットのレイアウトをカーネルの内部表現に変換するために、
カーネルはまったく別のハンドラを必要とする。

## COMPAT_SYSCALL_DEFINEマクロ {#the-compat_syscall_define-macro}

compat syscallのハンドラは、`include/linux/compat.h` にある `COMPAT_SYSCALL_DEFINE` で定義される。

```c
/* include/linux/compat.h */
COMPAT_SYSCALL_DEFINE3(read, unsigned int, fd,
                       char __user *, buf,
                       compat_size_t, count)
```

`COMPAT_SYSCALL_DEFINE` は、`compat_sys_xxx` という名前のC関数（例: `compat_sys_read`）を生成する。x86では、
`arch/x86/include/asm/syscall_wrapper.h` にあるアーキテクチャ固有の `__IA32_COMPAT_SYS_STUBx` マクロが、さらに
32ビットレジスタをデコードして `compat_sys_xxx` を呼ぶ、別の `__ia32_compat_sys_xxx` エントリポイントスタブを
生成する。マクロ自体が生成するのは `compat_sys_xxx` であり、ia32スタブではない。マクロはまた、
compat syscallがネイティブのsyscallと同じように `ftrace` や `perf` の出力に現れるよう、
必要なtracepointのメタデータも生成する。

引数の型がどの幅でも同一であるsyscall（例: `close(int fd)`）では、`SYSCALL_DEFINE` の
ハンドラがそのまま再利用される。compatテーブルは同じ `__x64_sys_close` を指す。

## compat syscallテーブル {#the-compat-syscall-tables}

x86-64では、32ビットプロセスは別のテーブルを通じてディスパッチされる。

```
# arch/x86/entry/syscalls/syscall_32.tbl (partial)
# <number>  <abi>  <name>   <entry point>         <compat entry point>
3           i386   read     sys_read               # reuses 64-bit handler
...
11          i386   execve   sys_execve             compat_sys_execve
...
102         i386   socketcall  sys_socketcall       compat_sys_socketcall
...
```

ビルドシステムは `arch/x86/include/generated/asm/syscalls_32.h` を生成し、それが次の箇所にインクルードされる。

```c
/* arch/x86/entry/syscall_32.c */
#define __SYSCALL(nr, sym) case nr: return __ia32_##sym(regs);
static noinline long ia32_sys_call(const struct pt_regs *regs, unsigned int nr)
{
	switch (nr) {
	#include <asm/syscalls_32.h>
	default: return __ia32_sys_ni_syscall(regs);
	}
}
```

32ビットプロセスが `int 0x80` または `sysenter` でカーネルに入ると、エントリコードは32ビットのCS
セグメントを検出して `ia32_sys_call()` を呼ぶ。これはsyscall番号でswitchし、対応する `__ia32_*` スタブに振り分ける —
ネイティブの64ビットパスが使うのと同じswitchベースのディスパッチパターンである（`x64_sys_call()`。
[syscallのエントリパス](syscall-entry.md)を参照）。`CONFIG_X86_32` では `sys_call_table[]` 配列もまだ構築されるが、
それは `kernel/trace/trace_syscalls.c` がトレーシングのためにsyscall番号からアドレスへのテーブルを必要とするからにすぎない —
ディスパッチには使われない。

## 主要なcompat型 {#key-compat-types}

```c
/* include/asm-generic/compat.h（各アーキテクチャの asm/compat.h から取り込まれる） */
typedef u32                 compat_uptr_t;   /* 32ビットのユーザー空間ポインタ */
typedef u32                 compat_size_t;   /* 32ビットの size_t */
typedef s32                 compat_ssize_t;
typedef s32                 compat_long_t;
typedef u32                 compat_ulong_t;
typedef s32                 compat_int_t;
typedef s64 __attribute__((aligned(4))) compat_s64;
typedef u64 __attribute__((aligned(4))) compat_u64;
```

```c
/* include/vdso/time32.h */
struct old_timespec32 {
    old_time32_t tv_sec;   /* s32 */
    s32          tv_nsec;
};
```

`old_timespec32` は8バイトではなく4バイトでアラインされ、32ビットコンパイラが生成するレイアウトと一致する。
これが、32ビットの `clock_gettime()` が64ビットのハンドラを共有できない根本的な理由である。

## compat_ptr() とptr_to_compat() {#compat_ptr-and-ptr_to_compat}

`compat_ptr()` は、`compat_uptr_t`（u32）をゼロ拡張してカーネルの `void __user *` に変換する。

```c
/* include/linux/compat.h */
static inline void __user *compat_ptr(compat_uptr_t uptr)
{
    return (void __user *)(unsigned long)uptr;
}
```

32ビットプロセスは4 GBを超える有効なアドレスを持てないので、これは安全である。ゼロ拡張は単に、
32ビットアドレスの正しい64ビット表現を生成するだけである。

逆方向には `ptr_to_compat()` を使う。

```c
static inline compat_uptr_t ptr_to_compat(void __user *uptr)
{
    return (u32)(unsigned long)uptr;
}
```

## in_compat_syscall() {#in_compat_syscall}

32ビットの呼び出し元に対して異なる振る舞いをする必要があるコード（例: 正しい構造体サイズを選ぶため）は、
`in_compat_syscall()` を使う。x86では、これは単一のチェックではなく、syscallごとのステータス
ビットに至る3段階の連鎖である — スレッドフラグではない。他のいくつかのアーキテクチャ（ARM64、MIPS）は代わりに `TIF_32BIT` を使う。汎用の
宣言は `include/linux/compat.h` にあり、x86は `arch/x86/include/asm/compat.h` でそれをオーバーライドする。

```c
/* include/linux/compat.h — 汎用のフォールバック */
static inline bool in_compat_syscall(void)
{
    return is_compat_task();  /* アーキテクチャ固有 */
}

/* arch/x86/include/asm/compat.h — x86 によるオーバーライド */
static inline bool in_32bit_syscall(void)
{
    return in_ia32_syscall() || in_x32_syscall();
}

static inline bool in_compat_syscall(void)
{
    return in_32bit_syscall();
}
```

`in_ia32_syscall()` は関数ではなくマクロである — 2段階下の
`arch/x86/include/asm/thread_info.h` で定義されている。

```c
/* arch/x86/include/asm/thread_info.h */
#define in_ia32_syscall() (IS_ENABLED(CONFIG_IA32_EMULATION) && \
                           current_thread_info()->status & TS_COMPAT)
```

`TS_COMPAT` は、タスクが32ビット互換モードで実行しているときに設定される。これはVFSやネットワークスタックの
さまざまな箇所でチェックされ、わずかな違いしかない場合には、単一のカーネル内コードパスで
32ビットと64ビットの両方の呼び出し元に対応できるようにしている。

## 構造体レイアウトの違い: compat_iovecとcompat_stat {#struct-layout-differences-compat_iovec-and-compat_stat}

### compat_iovec {#compat_iovec}

```c
/* include/linux/compat.h */
struct compat_iovec {
    compat_uptr_t   iov_base;   /* u32 — 32ビットポインタ */
    compat_size_t   iov_len;    /* u32 */
};

/* ネイティブ版との比較: */
struct iovec {
    void __user    *iov_base;   /* x86-64では8バイト */
    __kernel_size_t iov_len;    /* x86-64では8バイト */
};
```

32ビットプロセスが渡す `iovec` 配列は要素あたり8バイトであり、64ビットのレイアウトは要素あたり16バイトである。
`readv()` と `writev()` はcompatのケースを検出し、`compat_iovec` を使って反復しなければならない。

### compat_stat {#compat_stat}

```c
/* arch/x86/include/asm/compat.h */
struct compat_stat {
    u32             st_dev;
    compat_ino_t    st_ino;     /* u32 */
    compat_mode_t   st_mode;    /* x86ではu16、汎用定義ではu32 */
    compat_nlink_t  st_nlink;   /* x86ではu16、汎用定義ではu32 */
    __compat_uid_t  st_uid;     /* x86ではu16、汎用定義ではu32 */
    __compat_gid_t  st_gid;     /* x86ではu16、汎用定義ではu32 */
    u32             st_rdev;
    u32             st_size;    /* わずか32ビット！ */
    u32             st_blksize;
    u32             st_blocks;
    u32             st_atime;   /* 32ビットの time_t */
    u32             st_atime_nsec;
    u32             st_mtime;
    u32             st_mtime_nsec;
    u32             st_ctime;
    u32             st_ctime_nsec;
    u32             __unused4;
    u32             __unused5;
};
```

x86-64のネイティブの `struct stat` は、`st_size` に64ビットの `off_t` を、そして64ビットの `time_t` を使う。カーネルは、
どちらのハンドラが呼ばれたかに応じて、いずれかの形式を埋める。

## compatが不要な場合 {#when-compat-is-not-needed}

移植性を念頭に置いて設計された現代的な構造体ベースのsyscallは、UAPI構造体で `long` やポインタ型ではなく、
幅が明示された型（`__u64`、`__u32`、`__s64`）を使う。構造体のレイアウトが
32ビットと64ビットで同一なので、compatハンドラは不要である。

```c
/* include/uapi/linux/openat2.h — openat2() が使う */
struct open_how {
    __u64   flags;
    __u64   mode;
    __u64   resolve;
};
```

32ビットプロセスが渡す `struct open_how` は、64ビットプロセスと同じバイナリレイアウトを持つ。単一の
`sys_openat2` ハンドラが両方で動作する。これは、あらゆる新しいsyscallに推奨されるパターンである。

## compat_ioctl: ドライバレベルのcompat {#compat_ioctl-driver-level-compat}

`ioctl()` は特別である。ioctlコマンドは構造体のサイズをエンコードしており、多くのコマンドは
アーキテクチャ依存のレイアウトを持つ構造体を渡すからである。ドライバは両方を実装する。

```c
/* include/linux/fs.h — file_operations */
struct file_operations {
    /* ... */
    long (*unlocked_ioctl)(struct file *, unsigned int, unsigned long);
    long (*compat_ioctl)(struct file *, unsigned int, unsigned long);
    /* ... */
};
```

32ビットプロセスが `ioctl()` を呼ぶと、compatのioctl syscallはまず、VFS自身が（ドライバとは独立に）汎用的に理解するコマンドについて
`do_vfs_ioctl()` を試す。それが `-ENOIOCTLCMD`（それらのコマンドではない）を返すと、
設定されていればドライバの `compat_ioctl` に処理が渡る。任意のドライバ固有コマンドに対する、汎用的で自動的な
変換テーブルによるフォールバックは存在しない — `compat_ioctl` がNULLなら、32ビットの
呼び出し元は、ドライバが明示的に処理していないものすべてに対して `-ENOTTY` を受け取る。唯一の引数が
存在しないか、幅の間ですでに互換性がある（ポインタであり、ポインタを埋め込んだ生の `unsigned long` 値ではない）コマンドについては、
ドライバは独自に書く代わりに、汎用の `compat_ptr_ioctl()` ヘルパー（`fs/ioctl.c`）を選択的に使える
— これは単に `compat_ptr()` を通じて `arg` をゼロ拡張し、`unlocked_ioctl` を呼ぶだけである。

```c
/* 例: compatを明示的に処理するドライバ */
static long my_compat_ioctl(struct file *file, unsigned int cmd,
                             unsigned long arg)
{
    switch (cmd) {
    case MY_IOCTL_GET_INFO32: {
        struct my_info32 __user *uinfo = compat_ptr(arg);
        struct my_info32 kinfo;
        /* カーネルの状態から kinfo を埋める */
        if (copy_to_user(uinfo, &kinfo, sizeof(kinfo)))
            return -EFAULT;
        return 0;
    }
    default:
        return my_ioctl(file, cmd, arg);
    }
}
```

## 実例: compat_sys_readとsys_read {#practical-example-compat_sys_read-vs-sys_read}

`read()` が渡すのは、32ビットのポインタと32ビットのカウントだけである。x86-64では、カーネルはこれを透過的に処理する。
`ia32_sys_call()` の `read` のケースは、ネイティブの `sys_read` ハンドラに直接ディスパッチする。32ビットのバッファアドレスに対する
`compat_ptr()` が正しい64ビットのユーザーポインタを生成し、`compat_size_t`（u32）は `size_t` に収まるからである。
別のcompatハンドラは必要ない。

## 実例: compat_sys_sendmsgとsys_sendmsg {#practical-example-compat_sys_sendmsg-vs-sys_sendmsg}

`sendmsg()` は `struct msghdr __user *` を渡す。これには埋め込みの `struct iovec __user *` と、
制御メッセージ用の `void __user *` が含まれる。どちらもポインタなので、32ビットと64ビットでサイズが
異なる。

```c
/* net/compat.c */
static inline long __compat_sys_sendmsg(int fd, struct compat_msghdr __user *msg,
                                        unsigned int flags)
{
    return __sys_sendmsg(fd, (struct user_msghdr __user *)msg,
                         flags | MSG_CMSG_COMPAT, false);
}

COMPAT_SYSCALL_DEFINE3(sendmsg, int, fd, struct compat_msghdr __user *, msg,
                       unsigned int, flags)
{
    return __compat_sys_sendmsg(fd, msg, flags);
}
```

これがcompat呼び出しであることを示すのは、別個のブール値ではなく `MSG_CMSG_COMPAT` フラグである — ここでORされている。
`__sys_sendmsg()` の最後の引数は `forbid_cmsg_compat` である。`true` のときは、
`MSG_CMSG_COMPAT` が設定された呼び出し元を拒否する（*ネイティブ*の `sys_sendmsg` パスで、そのフラグを偽装する
ネイティブの呼び出し元を拒否するために使われる）。compatパスは正当にフラグを自分で設定するので、`false` を渡す。下流では、
実際に `MSG_CMSG_COMPAT` で分岐するのは `sendmsg_copy_msghdr()`（`net/socket.c`）である。compatパスでは
`get_compat_msghdr()` を呼び、これが（`compat_uptr_t` フィールドを含む）`struct compat_msghdr` を読み取って、
処理を進める前にカーネル空間でネイティブの `struct msghdr` を再構築する。ネイティブパスでは、代わりに
`copy_msghdr_from_user()` を呼ぶ。

## 32ビットプロセスの検出 {#detecting-32-bit-processes}

```bash
# ELFクラスが32ビットであることを示す
file /proc/$(pidof myapp)/exe
# ELF 32-bit LSB executable ...

# あるいはタスクの pt_regs 内のCSセグメントレジスタを確認する
# ia32 では CS == __USER32_CS (0x23)、x86-64 では CS == __USER_CS (0x33)
```

カーネル内では、トレースされているタスクの実行モードを判定するために、`task_pt_regs(task)->cs` が `__USER32_CS` と
比較される。

## 古いcompat_alloc_user_space()パターン — Linux 5.15で削除 {#the-old-compat_alloc_user_space-pattern-removed-in-linux-515}

古いcompatハンドラは、上の共通ヘルパーパターンとは異なる戦略を使っていた。ユーザー自身のスタックポインタの下に
切り出したスクラッチ領域にネイティブレイアウトの構造体を構築し、ユーザー空間自身がその構造体を渡したかのように
*ネイティブ*のsyscallハンドラを呼ぶ、というものである。`compat_alloc_user_space()` は、そのスクラッチ領域を
切り出す関数だった — 結果を16バイトにアラインし、それが呼び出し元のアドレス範囲内にあることをチェックし、
`access_ok()` で検証してから返していた。

その関数と、それが可能にしていたパターンは、もう存在しない。`set_fs()` ベースのアドレス空間オーバーライドを
廃止するArnd Bergmannのパッチシリーズが最後の呼び出し元を取り除き、`compat_alloc_user_space()` 自体は
[commit a7a08b275a8b](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a7a08b275a8bbade798c4bdaad07ade68fe7003c)
（"arch: remove compat_alloc_user_space"）で削除された — **Linux 5.15**（2021年10月）以降は存在しない。

置き換えは新しい関数ではない。このページですでに `sendmsg()` について示した、カーネル空間の共通ヘルパーパターンである。
すでにコピーして取り込んだカーネルデータを受け取る共通の `__sys_*()` 関数を、ネイティブとcompatの両方の
エントリポイントから直接呼び出し、compat固有の変換（ポインタ幅、`MSG_CMSG_COMPAT` など）はその途中で
カーネル空間で行う — ユーザー自身のスタック上に構造体を偽造し、ユーザー空間がそれを構築したかのように
再入することは決してない。`compat_sys_socketcall()`（`net/compat.c`）— まさにこの節がかつて `compat_alloc_user_space()`
の利用者として挙げていた関数 — は、*以前*と*以後*を1か所で示す良い例である。これは依然として本当に独立した
compatハンドラである（32ビットプロセスはsocketcallの引数配列をネイティブの `unsigned long` ではなく `u32` として詰めるので、
ネイティブのパーサをそのまま再利用することはできない）。しかし現在は、その配列を `copy_from_user()` でカーネル空間に取り込み、
ネイティブパスが使うのと同じ内部ヘルパー（`__sys_bind()`、`__sys_connect()` など）に直接ディスパッチし、
その途中でポインタサイズの引数を `compat_ptr()` で変換する — ユーザースタック上のスクラッチ領域は一切関与しない。

## まとめ: compatハンドラを書くべきとき {#summary-when-to-write-a-compat-handler}

| 状況 | 対応 |
|-----------|--------|
| syscallの引数が `int`、`unsigned int`、固定サイズの型 | ネイティブのハンドラを共有する |
| 引数に `long`、`size_t`、`off_t`、ポインタが含まれる | `COMPAT_SYSCALL_DEFINE` のハンドラを書く |
| ポインタで渡される構造体が明示的な `__u64`/`__u32` を使う | ネイティブのハンドラを共有する（compat不要） |
| 構造体に埋め込みポインタや `time_t` がある | compat構造体とcompatハンドラを書く |
| ポインタ引数を持つドライバの `ioctl` | `file_operations` に `compat_ioctl` を実装する |

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/compat.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/compat.h) — `COMPAT_SYSCALL_DEFINEx()` マクロ、`compat_ptr()`/`ptr_to_compat()`、`struct compat_iovec`
- [include/asm-generic/compat.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/asm-generic/compat.h) — 基本となる `compat_uptr_t`、`compat_size_t`、`compat_long_t`、`compat_s64`/`compat_u64` のtypedef（各アーキテクチャの `asm/compat.h` から取り込まれる）
- [arch/x86/include/asm/compat.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/compat.h) — `struct compat_stat` と、x86による `in_compat_syscall()`/`in_32bit_syscall()` のオーバーライド
- [arch/x86/include/asm/thread_info.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/thread_info.h) — `in_ia32_syscall()` マクロと、それがチェックする `TS_COMPAT` フラグ
- [include/vdso/time32.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/vdso/time32.h) — `struct old_timespec32`。実際の32ビットレイアウトのtimespec
- [arch/x86/include/asm/syscall_wrapper.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/syscall_wrapper.h) — `__IA32_COMPAT_SYS_STUBx()`: `__ia32_compat_sys_xxx` エントリスタブを生成する
- [arch/x86/entry/syscalls/syscall_32.tbl](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/entry/syscalls/syscall_32.tbl) — ia32のsyscallテーブル。compatエントリポイントの列を含む
- [include/uapi/linux/openat2.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/openat2.h) — `struct open_how`: compatハンドラを必要としない、幅が明示されたUAPI構造体
- [net/compat.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/compat.c) — `compat_sys_sendmsg`/`compat_sys_recvmsg`、`compat_sys_socketcall()`、`get_compat_msghdr()`
- [include/linux/fs.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/fs.h) — `compat_ioctl` メンバーを含む `struct file_operations`
- [fs/ioctl.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/ioctl.c) — `compat_ptr_ioctl()` と、compatの `ioctl()` syscallのディスパッチロジック
- [commit a7a08b275a8b](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a7a08b275a8bbade798c4bdaad07ade68fe7003c) — "arch: remove compat_alloc_user_space"、Arnd Bergmann、Linux 5.15で初めて取り込まれた

### 関連ページ {#related-pages}

- [SYSCALL_DEFINEとディスパッチ](syscall-define.md) — ネイティブのsyscallがどのように定義され、ディスパッチされるか
- [syscallのエントリパス](syscall-entry.md) — `do_syscall_64()` のディスパッチと、`do_syscall_x32()` のx32パス。ia32 compatは、これらのいずれでもなく、独自の別のエントリポイント（`ia32_sys_call()`）を通じてディスパッチされる
- [新しいsyscallの追加](adding-syscall.md) — 新しいsyscallにcompatハンドラが必要かどうかを判断するチェックリストを含む

### LWN記事 {#lwn-articles}

- [LWN: System calls and 64-bit architectures](https://lwn.net/Articles/311630/) — Jake Edge、2008年12月。`preadv()`/`pwritev()` の設計をめぐる議論であり、`compat_sys_*` ハンドラを必要とするsyscallとそうでないsyscallがある理由を示している

### 外部資料 {#external}

- [Adding System Calls — The Linux Kernel documentation](https://docs.kernel.org/process/adding-syscalls.html) — `Documentation/process/adding-syscalls.rst` のレンダリング版。compatハンドラが必要となる場合については "Compatibility System Calls (Generic)" を参照
