# 新しいシステムコールの追加 {#adding-a-new-system-call}

> カーネルコントリビュータ向けの完全な解説

## 新しいsyscallを追加すべきとき {#when-to-add-a-new-syscall}

新しいsyscallが適切なアプローチとなるのは、次のような場合である。

- 新しいカーネルの機能をユーザー空間に公開する
- その操作に、ユーザー空間では得られない特権やアトミックなセマンティクスが必要である
- カーネル内部の状態へのアクセスが必要である

次の手段で実現できることのためにsyscallを追加するのは避ける。

- 既存のデバイス/ファイルに対する `ioctl`（デバイス固有の操作の場合）
- `sysfs`/`procfs`（設定や観測の場合）
- Netlink（ネットワーク関連の設定の場合）
- 既存のsyscallの拡張（例: 新しい `cmd` を持つ `fcntl`）

## ステップ1: syscallを定義する {#step-1-define-the-syscall}

新しいsyscallには、構造体ベースの現代的なパターンを使う。

```c
/* 例: kernel/hello.c にある架空の "hello" syscall */
#include <linux/syscalls.h>
#include <linux/uaccess.h>
#include <uapi/linux/hello.h>

/**
 * sys_hello - write a greeting to a buffer
 * @uargs: pointer to struct hello_args in userspace
 * @size: size of the struct passed by userspace
 *
 * Returns 0 on success, -errno on failure.
 */
SYSCALL_DEFINE2(hello, struct hello_args __user *, uargs, size_t, size)
{
    struct hello_args args;
    char kernel_name[64];
    int ret;

    /* 構造体を検証してコピーする */
    if (size < HELLO_ARGS_SIZE_VER0)
        return -EINVAL;
    if (copy_struct_from_user(&args, sizeof(args), uargs, size))
        return -EFAULT;

    /* フラグを検証する: 既知のフラグのみ許可 */
    if (args.flags & ~HELLO_FLAG_LOUD)
        return -EINVAL;

    /* ユーザー空間から名前文字列をコピーする */
    if (strncpy_from_user(kernel_name, u64_to_user_ptr(args.name),
                          sizeof(kernel_name)) < 0)
        return -EFAULT;
    kernel_name[sizeof(kernel_name) - 1] = '\0';

    /* 実際の処理を行う */
    pr_info("Hello, %s!\n", kernel_name);
    if (args.flags & HELLO_FLAG_LOUD)
        pr_info("HELLO, %s!!!\n", kernel_name);

    return 0;
}
```

## ステップ2: UAPI構造体を定義する {#step-2-define-the-uapi-struct}

ユーザー空間から見える型は `include/uapi/linux/hello.h` に置く。

```c
/* include/uapi/linux/hello.h */
#ifndef _UAPI_LINUX_HELLO_H
#define _UAPI_LINUX_HELLO_H

#include <linux/types.h>

/* hello() 用のフラグ */
#define HELLO_FLAG_LOUD  (1 << 0)

/* バージョン0の構造体サイズ */
#define HELLO_ARGS_SIZE_VER0  (sizeof(struct hello_args))

struct hello_args {
    __u32   flags;      /* HELLO_FLAG_* */
    __u32   pad;        /* 明示的なパディング（暗黙の隙間を決して残さない） */
    __u64   name;       /* 名前文字列へのポインタ（64/32ビット互換のため __u64 を使う） */
};

#endif /* _UAPI_LINUX_HELLO_H */
```

UAPI構造体の重要なルール:

- `__u8`、`__u16`、`__u32`、`__u64` を使う（`int`、`long` などは使わない）
- アラインメントのために明示的なパディング（`__u32 pad`）を追加する — 構造体の暗黙の穴に決して頼らない
- ポインタサイズのフィールドには `__u64` を使う（32ビットユーザー空間との互換性が得られる）
- 構造体はABIである — 一度マージされたら、互換性のない形で変更することはできない

## ステップ3: syscallテーブルに追加する {#step-3-add-to-the-syscall-table}

### x86-64の場合: `arch/x86/entry/syscalls/syscall_64.tbl` {#for-x86-64-archx86entrysyscallssyscall_64tbl}

```
# Append at the end (after the highest existing number)
451     common  hello           sys_hello
```

### 汎用アーキテクチャの場合: `include/uapi/asm-generic/unistd.h` {#for-generic-architectures-includeuapiasm-genericunistdh}

```c
#define __NR_hello 451
__SYSCALL(__NR_hello, sys_hello)
#undef __NR_syscalls
#define __NR_syscalls 452
```

### ARM64の場合: `arch/arm64/include/asm/unistd.h` {#for-arm64-archarm64includeasmunistdh}

ARM64は汎用テーブルを使うので、`include/uapi/asm-generic/unistd.h` でカバーされる。

### 32ビット互換の場合 {#for-32-bit-compat}

syscallがポインタを（`__u64` として格納した）構造体で受け取るなら、32ビットプロセスでもすでに正しく動作する — compat版は不要である。アーキテクチャ固有の型を使う場合は、compat版を追加する。

```c
/* 必要な場合のみ（現代的なsyscallのほとんどはこれを必要としない） */
#ifdef CONFIG_COMPAT
COMPAT_SYSCALL_DEFINE2(hello, compat_uptr_t, uargs, compat_size_t, size)
{
    /* 32ビットの呼び出し元を処理する */
}
#endif
```

## ステップ4: カーネルヘッダを追加する {#step-4-add-the-kernel-header}

```c
/* include/linux/hello.h */
#ifndef _LINUX_HELLO_H
#define _LINUX_HELLO_H

#include <uapi/linux/hello.h>

/* カーネル内部の宣言（ユーザー空間にはエクスポートされない） */
long ksys_hello(struct hello_args __user *uargs, size_t size);

#endif
```

## ステップ5: ビルドに組み込む {#step-5-wire-it-up-in-the-build}

適切な `Makefile` に追加する。

```makefile
# kernel/Makefile
obj-y += hello.o
```

小さければ、既存のファイルに追加してもよい。

## ステップ6: manページの雛形を追加する {#step-6-add-a-man-page-stub}

```
man2/hello.2  ← syscall man pages live in section 2
```

syscallはman-pagesプロジェクト（カーネルツリー外）が文書化している。そちらにもパッチを送ること。

## ステップ7: selftestを書く {#step-7-write-a-selftest}

```c
/* tools/testing/selftests/hello/hello_test.c */
#include <sys/syscall.h>
#include <unistd.h>
#include <stdio.h>
#include <assert.h>

#ifndef __NR_hello
#define __NR_hello 451
#endif

#include <linux/hello.h>

int main(void)
{
    struct hello_args args = {
        .flags = 0,
        .pad   = 0,
        .name  = (unsigned long)"world",
    };

    long ret = syscall(__NR_hello, &args, sizeof(args));
    assert(ret == 0);

    /* テスト: 未知のフラグは失敗するはず */
    args.flags = 0xdeadbeef;
    ret = syscall(__NR_hello, &args, sizeof(args));
    assert(ret == -1 && errno == EINVAL);

    /* テスト: NULLポインタは失敗するはず */
    args.flags = 0;
    args.name = 0;
    ret = syscall(__NR_hello, &args, sizeof(args));
    assert(ret == -1 && errno == EFAULT);

    printf("All tests passed\n");
    return 0;
}
```

```makefile
# tools/testing/selftests/hello/Makefile
TEST_GEN_PROGS := hello_test
include ../lib.mk
```

## よくある間違い {#common-mistakes}

### 構造体サイズの検証を忘れる {#forgetting-to-validate-struct-size}

```c
/* 誤り: 構造体がちょうどこのサイズであると仮定している */
if (copy_from_user(&args, uargs, sizeof(args)))

/* 正しい: バージョン付き構造体には copy_struct_from_user を使う */
if (copy_struct_from_user(&args, sizeof(args), uargs, size))
```

### 構造体に暗黙のパディングを残す {#leaving-implicit-padding-in-structs}

```c
/* 誤り: コンパイラが flags の後に4バイトのパディングを追加する */
struct bad_args {
    __u32 flags;
    __u64 ptr;   /* 8バイトアラインメントのため、この前に4バイトのパディングが必要 */
};

/* 正しい: 明示的なパディングで、驚きがない */
struct good_args {
    __u32 flags;
    __u32 pad;   /* 明示的なパディング */
    __u64 ptr;
};
```

### UAPIでカーネルの型を使う {#using-kernel-types-in-uapi}

```c
/* 誤り: size_t はアーキテクチャによって4バイトにも8バイトにもなる */
struct bad {
    size_t size;
};

/* 正しい: サイズが明示されている */
struct good {
    __u64 size;
};
```

### ユーザー空間のポインタを直接参照外しする {#directly-dereferencing-userspace-pointers}

```c
/* 誤り: 未定義動作 + セキュリティ脆弱性 */
int result = *(int *)uargs->ptr;

/* 正しい: copy_from_user または get_user を使う */
int value;
if (get_user(value, (int __user *)uargs->ptr))
    return -EFAULT;
```

### ケーパビリティチェックの欠落 {#missing-capability-check}

```c
/* syscallが特権を必要とする場合: */
if (!capable(CAP_SYS_ADMIN))
    return -EPERM;

/* namespaceスコープの特権の場合: */
if (!ns_capable(current_user_ns(), CAP_SYS_ADMIN))
    return -EPERM;
```

## マージせずにテストする {#testing-without-merging}

```bash
# 変更を加えたカーネルをビルドする
make -j$(nproc) LOCALVERSION="-test"

# VMにインストールする
make INSTALL_MOD_PATH=/tmp/modules modules_install
make INSTALL_PATH=/boot install

# selftestでテストする
cd tools/testing/selftests/hello
make
sudo ./run_kselftest.sh
```

## レビュープロセス {#the-review-process}

レビュアーからは次のような質問が来ると考えておくこと。
1. **なぜ新しいsyscallなのか？** ioctl/sysfsなどでは実現できないのか？
2. **ABIレビュー**: 構造体は将来にわたって通用するか？ 明示的なパディングがあるか？
3. **セキュリティ**: すべてのポインタが検証されているか？ 悪意のある値で害を及ぼせないか？
4. **エラー処理**: すべてのパスが意味のあるerrnoを返すか？
5. **互換性**: 64ビットカーネル上の32ビットプロセスで動作するか？
6. **ドキュメント**: manページは？ kernel-docコメントは？
7. **テスト**: selftestは含まれているか？

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/syscalls.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/syscalls.h) — `SYSCALL_DEFINE0`...`SYSCALL_DEFINE6` マクロと、それらが展開される `SYSCALL_DEFINEx`/`__SYSCALL_DEFINEx` の連鎖
- [arch/x86/entry/syscalls/syscall_64.tbl](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/entry/syscalls/syscall_64.tbl) — x86-64のsyscall番号テーブル（`<number> <abi> <name> <entry point>`）
- [include/uapi/asm-generic/unistd.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/asm-generic/unistd.h) — ARM64などのアーキテクチャが共有する汎用syscallテーブル。`__NR_syscalls` と `__SYSCALL()` マクロ
- [include/linux/uaccess.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/uaccess.h) — `copy_struct_from_user()`。拡張可能なsyscall ABIのための、バージョン付き構造体のコピーヘルパー

### manページ {#man-pages}

- [`syscall(2)`](https://man7.org/linux/man-pages/man2/syscall.2.html) — Cからの低レベルなsyscall呼び出しと、`SYS_*` 番号マクロ
- [`syscalls(2)`](https://man7.org/linux/man-pages/man2/syscalls.2.html) — Linuxシステムコールの全一覧。カーネルバージョンとアーキテクチャごとの利用可否付き

### 関連ページ {#related-pages}

- [syscallのエントリパス](syscall-entry.md) — 新しいsyscallがどのように呼び出されるか
- [SYSCALL_DEFINEとディスパッチ](syscall-define.md) — ディスパッチの仕組み
- [32ビット互換syscall](compat.md) — ステップ3で触れたcompat層

### 外部資料 {#external}

- [Adding a New System Call](https://docs.kernel.org/process/adding-syscalls.html) — 権威あるカーネルのガイド（`Documentation/process/adding-syscalls.rst`）。syscallの代替手段、ABIの拡張性、テーブルへの登録、compatエントリポイントを扱う
