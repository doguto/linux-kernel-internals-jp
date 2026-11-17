# ユーザーnamespaceとクレデンシャルのマッピング {#user-namespaces-and-credential-mapping}

> uid/gidマッピング、ケーパビリティセット、コンテナのための権限モデル

## ユーザーnamespaceが提供するもの {#what-user-namespaces-provide}

ユーザーnamespaceは、プロセスから見たユーザーIDとグループIDをホストシステムとの間でマッピングする。namespace内ではプロセスはuid 0（root）を持ちながら、ホスト上では非特権のuidにマッピングされうる。

これが**rootlessコンテナ**、つまりホストの権限を一切持たずにコンテナを実行する仕組みの基盤である。

```
Host system:           Container (user namespace):
  uid=1000 (alice)  ←→  uid=0 (root inside container)
  gid=1000           ←→  gid=0
  uid=1001-65535     ←→  uid=1-64535
```

## ユーザーnamespaceの作成 {#creating-a-user-namespace}

```c
#include <sched.h>
#include <unistd.h>

/* 新しいユーザーnamespaceでcloneする */
pid_t pid = clone(child_func, stack_top,
                  CLONE_NEWUSER | SIGCHLD, NULL);

/* あるいはシェルで: */
/* unshare -U /bin/bash */
/* id → マッピングされるまではuid=65534(nobody) */
```

`clone(CLONE_NEWUSER)`の後、子プロセスは新しいユーザーnamespaceを持つが、**uid/gidのマッピングはまだない**。親がマッピングを書き込むまで、権限を必要とする操作はすべて失敗する。

## uid_mapとgid_map {#uid_map-and-gid_map}

```bash
# 親から（fork後に）:
# 子のuid 0 → ホストのuid 1000、範囲1
echo "0 1000 1" > /proc/<child_pid>/uid_map
echo "0 1000 1" > /proc/<child_pid>/gid_map

# 形式: <ns-uid> <host-uid> <count>
# ns-uidの範囲[0, count)をhost-uidの範囲[host-uid, host-uid+count)にマッピングする

# 複数範囲のマッピング（多数のユーザーを持つコンテナ用）:
echo "0 1000 1" > /proc/<pid>/uid_map    # rootはホストの1000にマッピングされる
echo "1 100000 65534" >> ...             # uid 1〜65534はホストの100000〜165533にマッピングされる
```

### newuidmap / newgidmap {#newuidmap-newgidmap}

非特権ユーザーが単一エントリの制限を超えるマッピングを書き込むには、setuidヘルパーの`newuidmap`を使う:

```bash
# /etc/subuidの形式: user:start:count
# alice:100000:65536
# aliceはユーザーnamespaceでホストのuid 100000〜165535を使えるという意味

# newuidmapは非特権プロセスのためにuid_mapを書き込む:
newuidmap <pid> <ns-uid> <host-uid> <count> [...]
newuidmap 1234 0 1000 1 1 100000 65535

# Podman/rootless Dockerはこれを自動的に使う:
podman run --rm alpine id
# uid=0(root) gid=0(root) groups=0(root)
# （ホスト上では、プロセスはuid 1000のaliceとして実行される）
```

### /etc/subuidと/etc/subgid {#etcsubuid-and-etcsubgid}

```bash
# ユーザーに割り当てられた従属IDの範囲を表示する:
cat /etc/subuid
# alice:100000:65536
# bob:165536:65536

# 新しいユーザーに範囲を割り当てる:
usermod --add-subuids 200000-265535 charlie
usermod --add-subgids 200000-265535 charlie
```

## ユーザーnamespaceにおけるケーパビリティ {#capabilities-in-user-namespaces}

新しいユーザーnamespace内では、プロセスは**ケーパビリティをすべて**持った状態で開始する — ただしそのnamespace内でのみ有効である:

```c
/* ユーザーnamespaceにおけるケーパビリティはスコープが限定される: */

/* namespace内で許可されるもの: */
CAP_NET_ADMIN    /* そのnsが所有するネットワークnamespaceのネットワーク設定を変更する */
CAP_SYS_ADMIN    /* namespace内でのさまざまな管理操作 */
CAP_SETUID       /* uidをマッピング内の任意のuidに変更する */
CAP_CHOWN        /* マッピング内の任意のuidにchownする */
/* ... */

/* namespace内でも許可されないもの: */
/* CAP_SYS_RAWIO: ハードウェアへの直接アクセス */
/* CAP_SYS_MODULE: カーネルモジュールのロード */
/* システム全体へのアクセスを必要とするもの全般 */
```

### struct cred {#struct-cred}

```c
/* include/linux/cred.h */
struct cred {
    atomic_long_t   usage;

    kuid_t          uid;    /* 実UID */
    kgid_t          gid;    /* 実GID */
    kuid_t          suid;   /* 保存UID */
    kgid_t          sgid;   /* 保存GID */
    kuid_t          euid;   /* 実効UID */
    kgid_t          egid;   /* 実効GID */
    kuid_t          fsuid;  /* ファイルシステムUID */
    kgid_t          fsgid;  /* ファイルシステムGID */
    unsigned        securebits;  /* SUID関連のセキュリティビット */

    kernel_cap_t    cap_inheritable; /* 新しいプロセスが継承できるケーパビリティ */
    kernel_cap_t    cap_permitted;   /* このプロセスが使用してよいケーパビリティ */
    kernel_cap_t    cap_effective;   /* 現在使用中のケーパビリティ */
    kernel_cap_t    cap_bset;        /* ケーパビリティのバウンディングセット */
    kernel_cap_t    cap_ambient;     /* ambientケーパビリティセット */

    struct user_namespace   *user_ns;  /* 所有者のユーザーnamespace */
    struct user_struct      *user;
    struct group_info       *group_info;

    /* RCUで保護される: */
    struct rcu_head         rcu;
} __randomize_layout;
```

### ケーパビリティセット {#capability-sets}

```
cap_permitted:   maximum capabilities the process can have
cap_effective:   capabilities currently active (checked by kernel)
cap_inheritable: capabilities passed across execve
cap_bset:        bounding set: caps can never exceed this
cap_ambient:     ambient: automatically inherited without execve magic
```

### ケーパビリティのチェック {#checking-capabilities}

```c
/* kernel/capability.c */
bool ns_capable(struct user_namespace *ns, int cap)
{
    return ns_capable_common(ns, cap, CAP_OPT_NONE);
}

bool capable(int cap)
{
    return ns_capable(&init_user_ns, cap);
}
```

プロセスが`capable(CAP_NET_ADMIN)`を呼ぶと、カーネルはそのケーパビリティを`init_user_ns` — システム全体のルートnamespace — に対してチェックする。つまり`capable()`は、ホストの本物のrootケーパビリティを持つプロセスに対してのみtrueを返す。namespaceスコープのチェック（たとえば、このプロセスは特定のネットワークデバイスを所有するnamespaceでCAP_NET_ADMINを持っているか）には、カーネルは代わりに`ns_capable(ns, cap)`を使う。

## カーネルにおけるkuid_tとuidの再マッピング {#kuid_t-and-uid-remapping-in-the-kernel}

カーネルは、namespace内のUIDとホストのUIDとの混同を避けるため、全体を通して`kuid_t`（カーネルUID）を使う:

```c
/* include/linux/uidgid.h */
typedef struct {
    uid_t val;
} kuid_t;

/* namespaceのUIDとカーネルUIDの間で変換する: */
kuid_t make_kuid(struct user_namespace *from, uid_t uid)
{
    /* fromのuid_mapを通してuidをマッピングし、ホストのuidを得る */
    return KUIDT_INIT(map_id_up(&from->uid_map, uid));
}

uid_t from_kuid(struct user_namespace *to, kuid_t uid)
{
    /* toのuid_mapを通してホストのuidをマッピングし、nsのuidを得る */
    return map_id_down(&to->uid_map, __kuid_val(uid));
}

/* 例: */
/* ユーザーがnamespace内でuid=0が所有するファイルにuid=0として書き込む */
/* カーネルは変換する: kuid = make_kuid(task->user_ns, 0) = ホストのuid 1000 */
/* ファイルのst_uid = from_kuid(task->user_ns, file->kuid) = 0（ns内では0と表示される） */
```

## ユーザーnamespaceのセキュリティ境界 {#user-namespace-security-boundaries}

### ユーザーnamespaceにできないこと {#what-a-user-namespace-cannot-do}

ユーザーnamespace内でrootであっても、プロセスは次のことができない:

```
✗ カーネルモジュールのロード（CAP_SYS_MODULEはinit_user_nsを必要とする）
✗ rawブロックデバイスへのアクセス（CAP_SYS_RAWIO）
✗ グローバルなネットワークインターフェースの変更（ns所有のものだけ可能）
✗ システムクロックの変更（CAP_SYS_TIME）
✗ namespace外のプロセスに対するptrace
✗ マッピングされていないuidが所有するファイルへのアクセス（root所有の/proc、/sys）
```

### ネストしたユーザーnamespace {#nested-user-namespaces}

ユーザーnamespaceはネストできる（最大32階層）:

```
init_user_ns (uid 0 = system root)
└── container_user_ns (uid 0 → host uid 1000)
    └── nested_user_ns (uid 0 → host uid 1000)
        （ケーパビリティは各階層にスコープされる）
```

## 実践: rootlessコンテナ {#practical-rootless-containers}

```bash
# Podmanでrootlessコンテナを実行する（sudo不要）:
podman run --rm -it fedora bash
# 内部では: whoami → root
# ホストでは: psはプロセスをuid 1000（自分のユーザー）として表示する

# 仕組み:
# 1. Podmanがユーザーnamespaceを作成する
# 2. uid_mapを書き込む: 0 → 1000（自分）、1-65535 → 100000-165534（subuidの範囲）
# 3. ユーザーnamespaceが所有する他のnamespace（pid、mnt、net）を作成する
# 4. overlayfsをマウントする（ユーザーnamespaceがmount namespaceを所有するため許可される）

# マッピングを調べる:
cat /proc/$(pgrep -f "podman.*fedora")/uid_map
# 0       1000          1
# 1     100000      65535

# コンテナプロセスのケーパビリティを確認する:
cat /proc/$(pgrep -f "podman.*fedora")/status | grep Cap
# CapPrm: 000001ffffffffff  ← ユーザーns内ではすべて
# CapEff: 000001ffffffffff
# CapBnd: 000001ffffffffff
```

## セキュリティ上の考慮事項 {#security-considerations}

```bash
# CVE-2022-0185: fs contextのヒープオーバーフロー。ユーザーns経由で悪用可能
# CVE-2021-3493: OverlayFS + ユーザーnsによる権限昇格
# 注: CVE-2022-0847（dirty pipe）はユーザーnamespaceを必要としなかった —
# 任意の非特権ユーザーが直接悪用できた
# 緩和策: 非特権ユーザーnamespaceを制限する

# 非特権ユーザーnamespaceが許可されているか確認する:
cat /proc/sys/kernel/unprivileged_userns_clone  # Debian/Ubuntu
cat /proc/sys/user/max_user_namespaces         # 標準のカーネル

# 制限する（一部のディストリビューション）:
echo 0 > /proc/sys/kernel/unprivileged_userns_clone  # 無効化
echo 1 > /proc/sys/user/max_user_namespaces          # 最大1つまで許可

# AppArmor: 非特権でのユーザーns作成を拒否する
# Ubuntu 23.10以降: 既知のアプリケーション（Chrome、Podmanなど）を除き、
# 非rootによるユーザーnamespaceの作成をデフォルトでブロックする
echo 1 > /proc/sys/kernel/apparmor_restrict_unprivileged_userns
```

## ユーザーnamespaceの活動を観測する {#observing-user-namespace-activity}

```bash
# ユーザーnamespaceの階層を表示する:
lsns -t user
# NS         TYPE  NPROCS   PID USER  COMMAND
# 4026531837 user     223     1 root  /sbin/init
# 4026532536 user       2  5434 alice /usr/bin/podman

# UID/GIDのマッピングを表示する:
cat /proc/<pid>/uid_map
cat /proc/<pid>/gid_map

# ユーザーnamespaceの作成をトレースする:
bpftrace -e '
kprobe:create_user_ns {
    printf("user ns created by pid %d (%s)\n", pid, comm);
}'

# どのnamespaceがケーパビリティチェックを所有しているか確認する:
bpftrace -e '
kprobe:ns_capable {
    printf("ns_capable: pid=%d cap=%d\n", pid, arg1);
}'

# straceでnamespace関連のsyscallを表示する:
strace -e clone,unshare,setns podman run ...
# clone(... CLONE_NEWUSER|CLONE_NEWPID|CLONE_NEWNS|CLONE_NEWNET ...)
```

## 参考文献 {#further-reading}

- [クレデンシャル](credentials.md) — struct cred、権限モデル
- [ケーパビリティ](capabilities.md) — ケーパビリティセットとチェック
- [コンテナの隔離](../cgroups/container-isolation.md) — namespaceの組み合わせ
- [ネットワークnamespace](../net/net-namespaces.md) — ネットワークの隔離
- [SELinux](selinux.md) — DACの上に載るMAC
- `kernel/user_namespace.c` — uid_mapの解析とnamespaceの作成
- `include/linux/cred.h` — struct credの定義
- `man 7 user_namespaces` — 包括的なmanページ
- `man 1 newuidmap` — 従属uidのマッピングヘルパー
