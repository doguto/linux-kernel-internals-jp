# Landlock: 非特権サンドボックス {#landlock-unprivileged-sandboxing}

> rootなしでプロセスのファイルシステムとネットワークへのアクセスを制限する — Linuxの新しいサンドボックスLSM

## Landlockとは {#what-is-landlock}

Landlock（Mickaël SalaünによってLinux 5.13でマージされた — [`17ae69aba89d`](https://git.kernel.org/linus/17ae69aba89dbfa2139b7f8024b757ab3cc42f59)、[LWN](https://lwn.net/Articles/859908/)）は、**非特権プロセス**が自分自身のファイルシステム（5.13以降）とTCPネットワークポート（Linux 6.7、ABI v4以降）へのアクセスを制限できるようにするLinux Security Moduleである。syscallをフィルタするseccompとは異なり、Landlockはリソースへのアクセスを意味的なレベルで制御する。

主な特性:

- **rootは不要**: どのプロセスも自分用のLandlockサンドボックスを作成できる
- **不可逆**: 一度ルールが適用されると、緩めることはできない
- **継承される**: 子プロセスは制限を継承する
- **積み重ね**: ルールはさらに制限することしかできず、拡張することはできない

## ユースケース {#use-cases}

- ブラウザによるレンダラプロセスのサンドボックス化
- 言語ランタイムによるダウンロードしたコードの制限
- セキュリティ上重要なデーモンによる攻撃対象領域の縮小
- 多層防御の一層としてのコンテナランタイム

## APIの概要 {#api-overview}

```c
#include <linux/landlock.h>
#include <sys/syscall.h>

/* 3つのステップ: ルールセットの作成 → ルールの追加 → 適用 */

/* ステップ1: ルールセットを作成する */
struct landlock_ruleset_attr ruleset_attr = {
    .handled_access_fs =
        LANDLOCK_ACCESS_FS_EXECUTE |
        LANDLOCK_ACCESS_FS_WRITE_FILE |
        LANDLOCK_ACCESS_FS_READ_FILE |
        LANDLOCK_ACCESS_FS_READ_DIR |
        LANDLOCK_ACCESS_FS_REMOVE_DIR |
        LANDLOCK_ACCESS_FS_REMOVE_FILE |
        LANDLOCK_ACCESS_FS_MAKE_CHAR |
        LANDLOCK_ACCESS_FS_MAKE_DIR |
        LANDLOCK_ACCESS_FS_MAKE_REG |
        LANDLOCK_ACCESS_FS_MAKE_SYM |
        LANDLOCK_ACCESS_FS_MAKE_SOCK |
        LANDLOCK_ACCESS_FS_MAKE_FIFO |
        LANDLOCK_ACCESS_FS_MAKE_BLOCK |
        LANDLOCK_ACCESS_FS_REFER |      /* ディレクトリをまたぐrename（v2） */
        LANDLOCK_ACCESS_FS_TRUNCATE,    /* truncate（v3） */
    .handled_access_net =
        LANDLOCK_ACCESS_NET_BIND_TCP |
        LANDLOCK_ACCESS_NET_CONNECT_TCP,
};

int ruleset_fd = syscall(SYS_landlock_create_ruleset,
                          &ruleset_attr, sizeof(ruleset_attr), 0);
```

### ステップ2: ルールを追加する {#step-2-add-rules}

```c
/* /usrへの読み取りアクセスを許可する: */
int path_fd = open("/usr", O_PATH | O_CLOEXEC);

struct landlock_path_beneath_attr path_attr = {
    .allowed_access =
        LANDLOCK_ACCESS_FS_EXECUTE |
        LANDLOCK_ACCESS_FS_READ_FILE |
        LANDLOCK_ACCESS_FS_READ_DIR,
    .parent_fd = path_fd,
};

syscall(SYS_landlock_add_rule,
        ruleset_fd,
        LANDLOCK_RULE_PATH_BENEATH,
        &path_attr, 0);
close(path_fd);

/* /tmpへの読み書きを許可する: */
path_fd = open("/tmp", O_PATH | O_CLOEXEC);
path_attr.allowed_access =
    LANDLOCK_ACCESS_FS_READ_FILE |
    LANDLOCK_ACCESS_FS_WRITE_FILE |
    LANDLOCK_ACCESS_FS_READ_DIR |
    LANDLOCK_ACCESS_FS_MAKE_REG |
    LANDLOCK_ACCESS_FS_REMOVE_FILE;
path_attr.parent_fd = path_fd;

syscall(SYS_landlock_add_rule,
        ruleset_fd, LANDLOCK_RULE_PATH_BENEATH, &path_attr, 0);
close(path_fd);

/* ポート443へのTCP connectを許可する: */
struct landlock_net_port_attr net_attr = {
    .allowed_access = LANDLOCK_ACCESS_NET_CONNECT_TCP,
    .port = 443,
};
syscall(SYS_landlock_add_rule,
        ruleset_fd, LANDLOCK_RULE_NET_PORT, &net_attr, 0);
```

### ステップ3: 適用する {#step-3-enforce}

```c
/* ambientケーパビリティを放棄する（適用前に必要）: */
prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);

/* ルールセットをこのプロセス（とそのすべての子）に適用する: */
syscall(SYS_landlock_restrict_self, ruleset_fd, 0);
close(ruleset_fd);

/* これ以降: ファイルシステムとネットワークへのアクセスは制限される */
/* ルールでカバーされないアクセスはすべて → EACCES */
```

## サンドボックスの完全な例 {#complete-sandbox-example}

```c
#include <fcntl.h>
#include <linux/landlock.h>
#include <sys/prctl.h>
#include <sys/syscall.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>
#include <stdio.h>

static inline int landlock_create_ruleset(
    const struct landlock_ruleset_attr *attr, size_t size, __u32 flags)
{
    return syscall(__NR_landlock_create_ruleset, attr, size, flags);
}
static inline int landlock_add_rule(int ruleset_fd, enum landlock_rule_type type,
                                     const void *attr, __u32 flags)
{
    return syscall(__NR_landlock_add_rule, ruleset_fd, type, attr, flags);
}
static inline int landlock_restrict_self(int ruleset_fd, __u32 flags)
{
    return syscall(__NR_landlock_restrict_self, ruleset_fd, flags);
}

int sandbox_init(void)
{
    /* ABIバージョンを確認する: */
    int abi = syscall(__NR_landlock_create_ruleset, NULL, 0,
                       LANDLOCK_CREATE_RULESET_VERSION);
    if (abi < 1) {
        /* カーネルがLandlockをサポートしていない — 穏便にフォールバックする */
        fprintf(stderr, "Landlock not supported (abi=%d)\n", abi);
        return 0;
    }

    /* カーネルのABIバージョンで利用可能な機能だけを使う: */
    __u64 fs_access =
        LANDLOCK_ACCESS_FS_EXECUTE |
        LANDLOCK_ACCESS_FS_READ_FILE |
        LANDLOCK_ACCESS_FS_READ_DIR |
        LANDLOCK_ACCESS_FS_WRITE_FILE |
        LANDLOCK_ACCESS_FS_REMOVE_FILE |
        LANDLOCK_ACCESS_FS_MAKE_REG;
    if (abi >= 2)
        fs_access |= LANDLOCK_ACCESS_FS_REFER;
    if (abi >= 3)
        fs_access |= LANDLOCK_ACCESS_FS_TRUNCATE;

    struct landlock_ruleset_attr ruleset_attr = {
        .handled_access_fs = fs_access,
    };
    if (abi >= 4) {
        ruleset_attr.handled_access_net =
            LANDLOCK_ACCESS_NET_BIND_TCP |
            LANDLOCK_ACCESS_NET_CONNECT_TCP;
    }

    int ruleset_fd = landlock_create_ruleset(&ruleset_attr,
                                              sizeof(ruleset_attr), 0);
    if (ruleset_fd < 0)
        return -errno;

    /* /usrと/libへの読み取り専用アクセスを許可する: */
    const char *ro_paths[] = { "/usr", "/lib", "/lib64", NULL };
    for (int i = 0; ro_paths[i]; i++) {
        int fd = open(ro_paths[i], O_PATH | O_CLOEXEC);
        if (fd < 0) continue;
        struct landlock_path_beneath_attr pa = {
            .allowed_access = LANDLOCK_ACCESS_FS_EXECUTE |
                              LANDLOCK_ACCESS_FS_READ_FILE |
                              LANDLOCK_ACCESS_FS_READ_DIR,
            .parent_fd = fd,
        };
        landlock_add_rule(ruleset_fd, LANDLOCK_RULE_PATH_BENEATH, &pa, 0);
        close(fd);
    }

    /* /tmpへの読み書きを許可する: */
    int tmp_fd = open("/tmp", O_PATH | O_CLOEXEC);
    if (tmp_fd >= 0) {
        struct landlock_path_beneath_attr pa = {
            .allowed_access = LANDLOCK_ACCESS_FS_READ_FILE |
                              LANDLOCK_ACCESS_FS_WRITE_FILE |
                              LANDLOCK_ACCESS_FS_READ_DIR |
                              LANDLOCK_ACCESS_FS_MAKE_REG |
                              LANDLOCK_ACCESS_FS_REMOVE_FILE,
            .parent_fd = tmp_fd,
        };
        landlock_add_rule(ruleset_fd, LANDLOCK_RULE_PATH_BENEATH, &pa, 0);
        close(tmp_fd);
    }

    prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
    int ret = landlock_restrict_self(ruleset_fd, 0);
    close(ruleset_fd);
    return ret;
}
```

## ファイルシステムのアクセス権 {#filesystem-access-rights}

```c
/* ファイルのアクセス権: */
LANDLOCK_ACCESS_FS_EXECUTE      /* ファイルを実行する */
LANDLOCK_ACCESS_FS_WRITE_FILE   /* ファイルに書き込む */
LANDLOCK_ACCESS_FS_READ_FILE    /* ファイルをopen/readする */
LANDLOCK_ACCESS_FS_READ_DIR     /* ディレクトリを一覧する（opendir） */
LANDLOCK_ACCESS_FS_REMOVE_DIR   /* rmdir */
LANDLOCK_ACCESS_FS_REMOVE_FILE  /* unlink */
LANDLOCK_ACCESS_FS_MAKE_CHAR    /* キャラクタデバイスをmknodする */
LANDLOCK_ACCESS_FS_MAKE_DIR     /* mkdir */
LANDLOCK_ACCESS_FS_MAKE_REG     /* 通常ファイルを作成する（creat/open O_CREAT） */
LANDLOCK_ACCESS_FS_MAKE_SOCK    /* Unixソケットを作成する */
LANDLOCK_ACCESS_FS_MAKE_FIFO    /* mkfifo */
LANDLOCK_ACCESS_FS_MAKE_BLOCK   /* ブロックデバイスをmknodする */
LANDLOCK_ACCESS_FS_MAKE_SYM     /* symlink */
LANDLOCK_ACCESS_FS_REFER        /* ディレクトリをまたぐrename/link（v2以降） */
LANDLOCK_ACCESS_FS_TRUNCATE     /* truncate（v3以降） */

/* ネットワークのアクセス権（v4以降）: */
LANDLOCK_ACCESS_NET_BIND_TCP    /* TCPソケットをポートにbind()する */
LANDLOCK_ACCESS_NET_CONNECT_TCP /* TCPソケットでポートにconnect()する */
```

## カーネルの実装 {#kernel-implementation}

### LSMフック {#lsm-hook}

```c
/* security/landlock/fs.c */
static int hook_path_link(struct dentry *old_dentry,
                           const struct path *new_dir,
                           struct dentry *new_dentry)
{
    const struct landlock_ruleset *dom = landlock_get_current_domain();
    if (!dom)
        return 0;  /* サンドボックス化されていない */

    /* プロセスがnew_dirにリンクを作成することを許可されているか確認する: */
    return check_access_path(dom, new_dir,
                             LANDLOCK_ACCESS_FS_MAKE_REG);
}
```

### ルールの格納 {#rule-storage}

```c
/* ルールはinode番号をキーとする赤黒木に格納される: */
struct landlock_ruleset {
    struct rb_root_cached  root_inode;  /* FSのルール */
    struct rb_root_cached  root_net;    /* ネットワークのルール */
    /* ... */
};

/* LANDLOCK_RULE_PATH_BENEATHでのlandlock_add_rule時: */
/* - path_fdを開く → inode番号を得る */
/* - allowed_accessビットマスクを持つノードをroot_inodeに挿入する */
/* - そのinode配下のすべてのパスがそのアクセス権を得る */
```

## Landlock、seccomp、AppArmorの比較 {#landlock-vs-seccomp-vs-apparmor}

| 機能 | seccomp | Landlock | AppArmor |
|---------|---------|----------|----------|
| rootが必要 | 不要 | 不要 | 必要（ポリシーのロード） |
| 粒度 | syscall | リソースの意味 | パス/ケーパビリティ |
| スタック可能 | 可（AND） | 可（最も厳しいものが優先） | 不可 |
| 実行時の性能 | 高速（BPF） | 高速（赤黒木） | 高速 |
| ポリシー言語 | C/BPF | C API | テキストのプロファイル |
| ファイルシステム | syscall経由 | FSの意味を直接扱う | 可 |
| ネットワーク | syscall経由 | TCPポート（v4以降） | 可 |

## デバッグ {#debugging}

```bash
# Landlockのサポートを確認する:
grep LANDLOCK /boot/config-$(uname -r)
# CONFIG_SECURITY_LANDLOCK=y

# Landlockの拒否をトレースする（LSMのauditでEACCESとして表示される）:
ausearch -m AVC | grep landlock
# または:
dmesg | grep "landlock"

# サンドボックスをテストする:
# （landlock-abi >= 1が必要）
unshare --user -- bash -c '
  python3 -c "
import ctypes, os
# ... Pythonによる簡略化したlandlock ...
"'

# straceは拒否された操作に対してEACCESを表示する:
strace -e open,openat,write ./sandboxed_program 2>&1 | grep EACCES
```

## 参考文献 {#further-reading}

- [seccomp](seccomp.md) — syscallレベルのフィルタリング
- [LSMフレームワーク](lsm.md) — LandlockがLSMスタックにどのようにフックするか
- [ケーパビリティ](capabilities.md) — 権限の削減
- [ユーザーnamespace](user-namespaces.md) — 非特権での隔離
- `security/landlock/` — カーネルの実装
- `Documentation/userspace-api/landlock.rst` — カーネルの公式ドキュメント
- `samples/landlock/sandboxer.c` — カーネルツリー内の例
