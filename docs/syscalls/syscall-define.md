# SYSCALL_DEFINEとディスパッチ {#syscall_define-and-dispatch}

> syscallがCでどのように定義され、ディスパッチテーブルに組み込まれるか

## SYSCALL_DEFINEマクロ {#syscall_define-macro}

カーネル内のすべてのsyscallは、次のことを行うマクロを使って定義される。
1. 正しいシグネチャ（`asmlinkage`）を持つ関数を作成する
2. `struct pt_regs` からの引数の取り出しを処理する
3. 計装用のフック（tracepoint、audit）を提供する

```c
/* include/linux/syscalls.h */

/* SYSCALL_DEFINE<N>。N は引数の数 */
SYSCALL_DEFINE0(getpid)
SYSCALL_DEFINE1(close, unsigned int, fd)
SYSCALL_DEFINE3(write, unsigned int, fd,
                const char __user *, buf,
                size_t, count)
SYSCALL_DEFINE6(mmap, unsigned long, addr,
                unsigned long, len,
                unsigned long, prot,
                unsigned long, flags,
                unsigned long, fd,
                unsigned long, off)
```

`__user` アノテーションは、（カーネルメモリではなく）ユーザー空間を指すポインタであることを示す。sparseツールはこれを使って `copy_from_user` 呼び出しの欠落を検出する。

### SYSCALL_DEFINEは何に展開されるか {#what-syscall_define-expands-to}

```c
/* SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf, size_t, count)
   はおおよそ次のように展開される: */

static long __do_sys_write(unsigned int fd, const char __user *buf, size_t count);

/* 外側のラッパー: pt_regs から引数を取り出す */
__visible long __x64_sys_write(const struct pt_regs *regs)
{
    return __do_sys_write(
        (unsigned int)regs->di,    /* rdi = 第1引数 */
        (const char __user *)regs->si,  /* rsi */
        (size_t)regs->dx           /* rdx */
    );
}

/* 32ビット互換用の __ia32_sys_write も作成される */

/* そして実際の実装: */
static long __do_sys_write(unsigned int fd, const char __user *buf, size_t count)
{
    /* ここに実装 */
}

/* SYSCALL_METADATA() は、ftrace のsyscall tracepointが使う
 * struct syscall_metadata とトレースイベントを生成する — これはトレーシング専用であり、
 * auditではない。auditのフックは別途組み込まれる（syscallの監査を参照） */
```

### 完全な例: sys_write {#full-example-sys_write}

```c
/* fs/read_write.c */
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
                size_t, count)
{
    return ksys_write(fd, buf, count);
}

ssize_t ksys_write(unsigned int fd, const char __user *buf, size_t count)
{
    CLASS(fd_pos, f)(fd);  /* クリーンアップガード: 戻るときに自動で fdput_pos() */
    ssize_t ret = -EBADF;

    if (!fd_empty(f)) {
        loff_t pos, *ppos = file_ppos(fd_file(f));
        if (ppos) {
            pos = *ppos;
            ppos = &pos;
        }
        ret = vfs_write(fd_file(f), buf, count, ppos);
        if (ret >= 0 && ppos)
            fd_file(f)->f_pos = pos;
    }

    return ret;
}
```

`ksys_write()` ラッパーに注目してほしい。これにより、syscallのエントリパスを経由せずに、カーネル内部（例: `init` のコード）からsyscallのロジックを呼び出せる。このパターンは、カーネル自身が内部的に呼び出す必要のあるsyscallに使われる。

`CLASS(fd_pos, f)(fd)` は、古い `struct fd f = fdget_pos(fd); ... fdput_pos(f);` パターンを置き換えたものである。これはスコープベースのクリーンアップガード（`DEFINE_CLASS` で生成される）であり、`f` がスコープを抜けると自動的に `fdput_pos()` を呼ぶので、どの復帰パスでも明示的な呼び出しなしに参照が解放される。`fd_empty(f)`/`fd_file(f)` は、ガードされたfdのアクセサであり、`f.file` への直接アクセスを置き換える。

## syscallテーブル {#the-syscall-table}

### 64ビット: arch/x86/entry/syscalls/syscall_64.tbl {#64-bit-archx86entrysyscallssyscall_64tbl}

```
# <number>  <abi>   <name>         <entry point>
0           common  read           sys_read
1           common  write          sys_write
2           common  open           sys_open
3           common  close          sys_close
...
```

ABIは `64`（64ビットのみ）、`x32`（x32 ABI）、`common`（両方）のいずれかである。ビルドシステムは `arch/x86/include/generated/asm/syscalls_64.h` を生成する。

```c
/* 自動生成 */
__SYSCALL(0,  sys_read)
__SYSCALL(1,  sys_write)
__SYSCALL(2,  sys_open)
...
```

これがsyscallテーブルの配列にインクルードされる。

```c
/* arch/x86/entry/syscall_64.c */
const sys_call_ptr_t sys_call_table[] = {
#include <asm/syscalls_64.h>
};
```

`sys_call_table[]` は、もはや実際のディスパッチの仕組みではない — 実際のディスパッチ（同じファイルの `do_syscall_64()`）は、同じ `<asm/syscalls_64.h>` のインクルード（`case nr: return __x64_##sym(regs);`）から構築される `switch` 文である `x64_sys_call()` を通る。これは、関数ポインタ配列を通じたインデックス呼び出しを置き換えたもので、その目的は特に*間接*呼び出しを避けることだった。[この変更を行ったコミット](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1e3ad78334a69b36e107232e337f9d693dcc9df2)が説明しているように、`switch` ならコンパイラは代わりに条件分岐を生成でき、そこでの予測ミスは*誤ったsyscall*を投機的に実行するだけで済む — 限定的で無害な結果である — 一方、間接呼び出しの予測ミスは攻撃者の影響下にあるコードへ飛びうる。配列自体が残されているのは、`kernel/trace/trace_syscalls.c` がまだ各syscallのアドレスを必要としているからにすぎない。

### アーキテクチャ非依存: include/uapi/asm-generic/unistd.h {#architecture-independent-includeuapiasm-genericunistdh}

独自のsyscallテーブルを持たないアーキテクチャ向け:

```c
/* include/uapi/asm-generic/unistd.h */
#define __NR_io_setup 0
__SC_COMP(__NR_io_setup, sys_io_setup, compat_sys_io_setup)
#define __NR_io_destroy 1
__SYSCALL(__NR_io_destroy, sys_io_destroy)
...
```

## syscall ABIの安定性 {#syscall-abi-stability}

syscallは一度マージされると、その番号と引数のセマンティクスが**変更されることは決してない**。これはカーネルの厳格なルールである。ユーザー空間を壊してはならない。

具体的な保証:

- syscall番号は変わらない
- 既存の引数が別の意味に解釈し直されることはない
- ポインタで渡される構造体は大きくなるだけである（新しいフィールドは末尾に追加され、ゼロは「未設定」を意味する）

新機能のために、カーネルは古いsyscallを互換性のない形で拡張するのではなく、新しいsyscallを追加する。

- `clone()` → `clone3()`（構造体ベース、拡張可能）
- `open()` → `openat()` → `openat2()`（`how` 構造体を追加）
- `read()` → `pread64()`、`readv()`、`preadv()`、`preadv2()`

## 構造体ベースのsyscall（現代的なパターン） {#struct-based-syscalls-modern-pattern}

現代的なアプローチは引数を構造体で渡し、新しいsyscallなしで将来の拡張を可能にする。

```c
/* 構造体ベースのsyscall: clone3（kernel/fork.c） */
SYSCALL_DEFINE2(clone3, struct clone_args __user *, uargs, size_t, size)
{
    int err;
    struct kernel_clone_args kargs;   /* struct clone_args ではない — 後述 */
    pid_t set_tid[MAX_PID_NS_LEVEL];

    kargs.set_tid = set_tid;

    err = copy_clone_args_from_user(&kargs, uargs, size);
    if (err)
        return err;

    if (!clone3_args_valid(&kargs))
        return -EINVAL;

    return kernel_clone(&kargs);
}
```

ここには2つの異なる構造体が関わっており、どちらもここに現れるので混同しやすい。

- `struct clone_args`（`include/uapi/linux/sched.h`）は、`uargs` が指す、固定でバージョン付きのユーザー空間向けレイアウトである — ABIの契約であり、フィールドはすべて `__aligned_u64` である。
- `struct kernel_clone_args`（`include/linux/sched/task.h`）は、カーネル内部の表現 `kargs` である — `clone3()` だけでなく、cloneが起こるあらゆる場所（`fork()`、`vfork()`、カーネルスレッド、`io_uring` ワーカー）で使われるスーパーセットであり、`fn`/`fn_arg` のようにユーザー空間に対応物のないカーネル専用のフィールドを持つ。

両者を橋渡しするのが `copy_clone_args_from_user()`（これも `kernel/fork.c` にある）である。これはローカルの `struct clone_args` に対して `copy_struct_from_user(&args, sizeof(args), uargs, usize)` を呼び、バージョン管理を処理する —

- `usize < sizeof(args)` の場合: あるものをコピーし、残りをゼロにする（古いユーザー空間、新しいカーネル）
- `usize > sizeof(args)` の場合: 拡張部分のバイトがゼロであることをチェックする（新しいユーザー空間、古いカーネル）

— そのうえで、検証済みのフィールドを、clone処理の残りの部分が使う `struct kernel_clone_args` にコピーする。

## 32ビット互換syscall {#32-bit-compat-syscalls}

64ビットカーネルは32ビットのユーザー空間バイナリをサポートする。ABI間で引数のレイアウトやサイズが実際に異なる場合、`COMPAT_SYSCALL_DEFINEx()` は別のcompatエントリポイントを定義する。しかしそれが必要なのは本当に必要な場合だけである — `mmap2` は有用な反例である。その引数はすべてのABIですでに同じ幅と意味を持つため、compat版はまったく必要ない。

```c
/* mm/mmap.c — これはネイティブのsyscallであり、COMPAT_SYSCALL_DEFINE ではない: */
SYSCALL_DEFINE6(mmap_pgoff, unsigned long, addr, unsigned long, len,
                unsigned long, prot, unsigned long, flags,
                unsigned long, fd, unsigned long, pgoff)
{
    return ksys_mmap_pgoff(addr, len, prot, flags, fd, pgoff);
}
```

`mmap2`（i386のsyscall 192）は、`arch/x86/entry/syscalls/syscall_32.tbl` で `sys_mmap_pgoff` に直接結び付けられており、別のcompatエントリポイントはまったくない — 一方、レガシーな `mmap`（i386のsyscall 90）にはcompatエントリポイント（`compat_sys_ia32_mmap`）がある。`mmap2` はオフセットをすでにページ単位（バイトではなく `pgoff`）で受け取るので、独自のABI固有の変換を必要としない。64ビットの `mmap` syscall（*別の*エントリポイントで、`arch/x86/kernel/sys_x86_64.c` の `SYSCALL_DEFINE6(mmap, ...)`）は、自身のABIが要求するバイトからページ単位への変換 — バイトオフセットを `PAGE_SHIFT` だけ右シフトする — を行ってから、`sys_mmap_pgoff` も呼ぶ同じ共通の内部ヘルパー `ksys_mmap_pgoff()` を呼ぶ。

独立した `ia32_sys_call_table` シンボルは存在しない — `arch/x86/entry/syscall_32.c` は `syscall_64.c` のパターンをそのまま踏襲している。名残の `sys_call_table[]` 配列（純粋な32ビットカーネルでのみビルドされ、トレーシングのために残されている）と並んで、`<asm/syscalls_32.h>` から構築されるもう1つの `switch` 文である、実際のディスパッチャ `ia32_sys_call()` がある。

## syscallのトレーシングとaudit {#syscall-tracing-and-audit}

SYSCALL_DEFINEマクロは自動的にtracepointを生成する。

```bash
# 利用可能なsyscall tracepoint
ls /sys/kernel/tracing/events/syscalls/
# sys_enter_read  sys_exit_read  sys_enter_write  sys_exit_write ...

# すべての write() 呼び出しをトレースする
echo 1 > /sys/kernel/tracing/events/syscalls/sys_enter_write/enable
cat /sys/kernel/tracing/trace_pipe
# bash-1234 [000] sys_enter_write: fd=1, buf=0x7fff..., count=6

# strace は ptrace を使い、別のレベルでインターセプトする
strace -e write ls
```

`audit` サブシステムも、セキュリティのためのログ記録を目的としてsyscallのエントリ/イグジットにフックする。

```bash
# uid 1000 によるすべての open を監査する
auditctl -a always,exit -F arch=b64 -S open,openat -F uid=1000
ausearch -k access
```

## 低速パスと高速パス {#slow-path-vs-fast-path}

一部のsyscallは、シグナル、スケジューリング、再開を処理する「低速パス」を通る。

```c
/* kernel/entry/common.c（簡略化 — 実際のループは外側の
 * exit_to_user_mode_loop() とこのインラインヘルパーに分かれており、
 * ここに示したより多くの作業項目を扱う: uprobes、livepatch、アーキテクチャ固有の作業） */
static unsigned long __exit_to_user_mode_loop(struct pt_regs *regs,
                                               unsigned long ti_work)
{
    while (ti_work & EXIT_TO_USER_MODE_WORK_LOOP) {
        local_irq_enable();

        if (ti_work & (_TIF_NEED_RESCHED | _TIF_NEED_RESCHED_LAZY))
            schedule();                        /* プリエンプションポイント */

        if (ti_work & (_TIF_SIGPENDING | _TIF_NOTIFY_SIGNAL))
            arch_do_signal_or_restart(regs);    /* シグナルを配送する */

        if (ti_work & _TIF_NOTIFY_RESUME)
            resume_user_mode_work(regs);        /* seccomp、ptrace の通知 */

        local_irq_disable();
        ti_work = read_thread_flags();          /* 新しい作業がないか再チェック */
    }

    return ti_work;
}
```

これが、syscallがプリエンプションとシグナル配送のポイントとなる理由である — カーネルは、syscallから戻るたびにthread-infoのワークフラグをチェックする。実際のフラグセットには `_TIF_UPROBE`（保留中のuprobeコールバック）と `_TIF_PATCH_PENDING`（livepatchの遷移）も含まれ、それぞれループ内に独自のハンドラを持つ。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/syscalls.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/syscalls.h) — `SYSCALL_DEFINEx()`/`__SYSCALL_DEFINEx()` マクロの定義と、`__MAP()` による引数マーシャリングの仕組み
- [arch/x86/include/asm/syscall_wrapper.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/syscall_wrapper.h) — x86による `__SYSCALL_DEFINEx()` のオーバーライド: `pt_regs` をデコードして `__x64_sys_*()`/`__ia32_sys_*()` スタブにする
- [fs/read_write.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/read_write.c) — `SYSCALL_DEFINE3(write, ...)` と `ksys_write()`。このページの例題
- [kernel/fork.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/fork.c) — `SYSCALL_DEFINE2(clone3, ...)` と `copy_clone_args_from_user()`: 構造体ベースの拡張可能なsyscallパターン
- [include/uapi/linux/sched.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/sched.h) — `struct clone_args`。`clone3()` のコピー元となる、バージョン付きのユーザー空間ABI構造体
- [include/linux/sched/task.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/sched/task.h) — `struct kernel_clone_args`。`clone3()` だけでなく、すべてのclone処理（`fork()`、`vfork()`、カーネルスレッド）が共有する内部構造体
- [mm/mmap.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mmap.c) — `SYSCALL_DEFINE6(mmap_pgoff, ...)`。`mmap2` が直接結び付けられている、ネイティブの（compatでない）syscall
- [arch/x86/entry/syscalls/syscall_32.tbl](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/entry/syscalls/syscall_32.tbl) — i386のsyscallテーブル。compatエントリポイントのない `mmap2` → `sys_mmap_pgoff` と、レガシーな `mmap` → `sys_old_mmap`/`compat_sys_ia32_mmap` の対比がわかる
- [arch/x86/entry/syscalls/syscall_64.tbl](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/entry/syscalls/syscall_64.tbl) — `sys_call_table` の生成元となる、syscall番号 → エントリポイントのテーブル
- [arch/x86/entry/syscall_64.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/entry/syscall_64.c) — `do_syscall_64()` と `x64_sys_call()`: `sys_call_table[]` を直接インデックスする方式を置き換えた、実際の `switch` ベースのディスパッチ
- [arch/x86/entry/syscall_32.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/entry/syscall_32.c) — `syscall_64.c` の32ビット/IA32互換版: `ia32_sys_call()`。同じ `switch` ベースのディスパッチパターン
- [commit 1e3ad78334a6](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1e3ad78334a69b36e107232e337f9d693dcc9df2) — "x86/syscall: Don't force use of indirect calls for system calls"。この変更と、そこで述べられているSpectre緩和策としての根拠
- [kernel/entry/common.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/entry/common.c) — `exit_to_user_mode_loop()`/`__exit_to_user_mode_loop()`: syscallからの復帰時に低速パスで行われるシグナル/再スケジュール/uprobe/livepatchのチェック

### manページ {#man-pages}

- [`syscalls(2)`](https://man7.org/linux/man-pages/man2/syscalls.2.html) — Linuxシステムコールの概要と、ラッパー関数の命名規則
- [`strace(1)`](https://man7.org/linux/man-pages/man1/strace.1.html) — このページのトレーシング例で使った、`ptrace` によるユーザー空間からのsyscallトレーシング

### 関連ページ {#related-pages}

- [syscallのエントリパス](syscall-entry.md) — ハードウェアの仕組みとエントリのアセンブリ
- [新しいsyscallの追加](adding-syscall.md) — ステップバイステップのガイド
- [32ビット互換syscall](compat.md) — `COMPAT_SYSCALL_DEFINEx()` とcompatテーブルが実際にどう動作するか
- [ptraceとsyscallのインターセプト](ptrace-interception.md) — strace、デバッガ、seccomp-notifyがsyscallをインターセプトする仕組み
- [syscallの監査](audit.md) — トレーシング/auditの節で触れたLinux Auditサブシステム
- [syscallの再開メカニズム](restart-block.md) — 低速パスのシグナル処理がどのように `ERESTARTSYS`/`restart_syscall()` につながるか

### LWN記事 {#lwn-articles}

- [Anatomy of a system call, part 1](https://lwn.net/Articles/604287/) — David Drysdale、LWN.net（2014）。`read()` の `fs/read_write.c` での定義を例に、`SYSCALL_DEFINEn()` を解説する

### 外部資料 {#external}

- [Adding a New System Call](https://docs.kernel.org/process/adding-syscalls.html) — カーネルの公式プロセスガイド（`Documentation/process/adding-syscalls.rst`）
