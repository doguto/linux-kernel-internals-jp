# クレデンシャルとユーザーnamespace {#credentials-and-user-namespaces}

> struct cred、uidマッピング、namespace境界をまたぐケーパビリティの継承

## struct cred: タスクのクレデンシャル {#struct-cred-task-credentials}

すべてのタスクは、自身のセキュリティ上の識別情報を保持する`struct cred`を持つ:

```c
/* include/linux/cred.h */
struct cred {
    atomic_long_t usage;        /* 参照カウント */

    kuid_t        uid;          /* 実UID */
    kgid_t        gid;          /* 実GID */
    kuid_t        suid;         /* 保存UID */
    kgid_t        sgid;         /* 保存GID */
    kuid_t        euid;         /* 実効UID（パーミッションチェックに使われる） */
    kgid_t        egid;         /* 実効GID */
    kuid_t        fsuid;        /* ファイルシステムアクセス用のUID */
    kgid_t        fsgid;        /* ファイルシステムアクセス用のGID */

    kernel_cap_t  cap_inheritable; /* 子がexecveで得られるケーパビリティ */
    kernel_cap_t  cap_permitted;   /* 許可されるケーパビリティの上限 */
    kernel_cap_t  cap_effective;   /* 現在有効なケーパビリティ */
    kernel_cap_t  cap_bset;        /* ケーパビリティのバウンディングセット */
    kernel_cap_t  cap_ambient;     /* ambientケーパビリティ（execをまたぐ） */

    struct user_struct *user;
    struct user_namespace *user_ns;  /* このUIDが属するnamespace */

    /* LSMのblob */
    void *security;   /* selinux_credなど */
};
```

### クレデンシャルの不変性 {#credential-immutability}

クレデンシャルは**copy-on-write**である。クレデンシャルを変更するには、新しいコピーを用意し、それを変更してからコミットする:

```c
/* kernel/cred.c */
/* 例: setuid()の実装 */
int do_setuid(uid_t uid)
{
    struct cred *new;

    /* 変更可能なコピーを用意する */
    new = prepare_creds();
    if (!new)
        return -ENOMEM;

    /* コピーを変更する */
    new->uid  = make_kuid(current_user_ns(), uid);
    new->suid = new->uid;
    new->euid = new->uid;

    /* アトミックにコミットする: current->credを置き換える */
    return commit_creds(new);
    /* 古いcredは参照がなくなった時点で解放される */
}

/* クレデンシャルの読み取り（常に安全）: */
uid_t uid = current_uid().val;
uid_t euid = current_euid().val;

/* あるいは任意のタスクから: */
uid_t uid = task_uid(task).val;
```

### execveにおけるクレデンシャルの遷移 {#the-credential-transition-on-execve}

```c
/* security/commoncap.c */
int cap_bprm_creds_from_file(struct linux_binprm *bprm, const struct file *file)
{
    const struct cred *old = current_cred();
    struct cred *new = bprm->cred;

    /* set-uidバイナリの場合: */
    if (uid_eq(new->euid, root_uid) ||
        capable_wrt_inode_uidgid(&init_user_ns, file_inode(bprm->file),
                                   CAP_SETUID)) {
        new->euid = inode->i_uid;  /* set-uid: ファイル所有者のuidを得る */
    }

    /* ケーパビリティ:
     * pP' = (pI & fI) | (fP & bounding_set)
     * pE' = pP' & fE
     * pI' = pI
     *
     * ここで:
     *   p = プロセス（変更前）
     *   f = ファイルのケーパビリティビット
     *   P = permitted, E = effective, I = inheritable
     */
    new->cap_permitted =
        cap_union(cap_intersect(old->cap_inheritable, fcaps->inheritable),
                  cap_intersect(fcaps->permitted, old->cap_bset));
}
```

## ユーザーnamespace {#user-namespaces}

**ユーザーnamespace**は、namespace内のUID/GIDとホストシステムとの間に専用のマッピングを作る。これにより次のことが可能になる:

- 非特権でのコンテナ作成
- ホストのrootなしでのコンテナ内の「root」
- ケーパビリティの隔離: namespace内のroot ≠ 外部のroot

```bash
# ユーザーnamespaceを作成する（非特権）:
unshare --user --map-root-user bash
# 内部では: whoami → root
# しかし: /proc/self/uid_mapを見ると自分の実UIDにマッピングされている

cat /proc/self/uid_map
# 0     1000    1   （ns内のUID 0 = ホストのUID 1000。1つのUIDをマッピング）
```

### uid_mapとgid_map {#uid_map-and-gid_map}

```bash
# /proc/<pid>/uid_mapの形式:
# <ns-uid-start> <host-uid-start> <count>

# 典型的な単一ユーザーのマッピング（コンテナ用）:
# 0     100000    65536
# namespace内のUID 0〜65535がホストの100000〜165535にマッピングされる

# マッピングを書き込む（親namespaceから、または特権で）:
echo "0 1000 1" > /proc/<pid>/uid_map
echo "0 1000 1" > /proc/<pid>/gid_map
# 注: プロセス自身であること（unshare）か、ホストでのCAP_SETUIDのどちらかが必要
```

### newuidmap / newgidmap {#newuidmap-newgidmap}

```bash
# 範囲マッピングを使うコンテナの場合:
# /etc/subuidと/etc/subgidが許可される範囲を定義する:
cat /etc/subuid
# alice:100000:65536
# （aliceはコンテナのマッピングにUID 100000〜165535を使える）

# コンテナのUIDをマッピングする:
newuidmap <pid> 0 1000 1 1 100000 65536
# コンテナ内のUID 0 = ホストの1000（1つのUID）
# コンテナ内のUID 1〜65536 = ホストの100000〜165535

newgidmap <pid> 0 1000 1 1 100000 65536
```

## カーネル: UIDの変換 {#kernel-uid-translation}

カーネルはパーミッションチェックのためにUIDを解決する際、常に初期ユーザーnamespaceにマッピングされる**k**uid（カーネルUID）を扱う:

```c
/* include/linux/uidgid.h */
typedef struct {
    uid_t val;
} kuid_t;  /* 常に初期ユーザーnamespaceにおける値 */

/* namespaceからカーネルuidへ変換する: */
kuid_t kuid = make_kuid(current_user_ns(), ns_uid);
/* 逆変換する: */
uid_t  ns_uid = from_kuid(current_user_ns(), kuid);

/* kuidがnamespace内で有効か確認する: */
if (!uid_valid(kuid))
    return -EINVAL;

/* 所有者を確認する: */
if (!uid_eq(kuid, file_inode->i_uid))
    return -EACCES;
```

### struct user_namespace {#struct-user_namespace}

```c
/* include/linux/user_namespace.h */
struct user_namespace {
    struct uid_gid_map uid_map;    /* uidの変換 */
    struct uid_gid_map gid_map;
    struct uid_gid_map projid_map;

    struct user_namespace *parent; /* 外側のnamespace */
    int level;                     /* ネストの深さ（最大32） */
    kuid_t owner;                  /* 親ns内での作成者のkuid */
    kgid_t group;                  /* 親ns内での作成者のkgid */

    struct ns_common ns;
    unsigned long flags;

    /* /proc/<pid>/uid_map + gid_map */
    struct list_head  keyring_name_list;
    struct key        *user_keyring_register;

    /* namespaceごとのプロセス制限 */
    ucount_t         ucounts;
};
```

## namespaceをまたぐケーパビリティ {#capabilities-across-namespaces}

ケーパビリティは**namespaceスコープ**である。あるネットワークnamespaceで`CAP_NET_ADMIN`を持っていても、制御できるのはそのnamespaceのネットワークスタックだけである。

```bash
# namespace内のケーパビリティを確認する:
capsh --print
# Current: =
# cap_net_admin+eip  （net_adminのみがeffective/inheritable/permitted）

# 特定のケーパビリティでプロセスを実行する:
capsh --caps='cap_net_admin+eip cap_sys_admin+eip' -- -c 'ip link add ...'

# net_adminのみを持つコンテナ（Docker経由）:
docker run --cap-drop=ALL --cap-add=NET_ADMIN myimage
```

```c
/* ケーパビリティチェック: プロセスは許可されているか */
bool capable(int cap)
{
    return ns_capable(&init_user_ns, cap);
}

bool ns_capable(struct user_namespace *ns, int cap)
{
    /* 指定されたnamespaceでの実効ケーパビリティを確認する */
    return security_capable(current_cred(), ns, cap, CAP_OPT_NONE) == 0;
}

/* ファイルシステム操作の場合: ファイルのnamespaceに対して確認する */
bool capable_wrt_inode_uidgid(struct user_namespace *ns,
                                const struct inode *inode, int cap)
{
    /* inodeのuidがマッピングされているnsでcapを持っていれば許可 */
    struct user_namespace *inode_ns = inode->i_sb->s_user_ns;
    return ns_capable(inode_ns, cap);
}
```

## loginuid: auditにおける識別情報 {#the-loginuid-audit-identity}

`/proc/self/loginuid`は、`su`/`sudo`の後でも、ログインした元のユーザーを記録する:

```bash
# aliceとしてsshでログインした後:
cat /proc/self/loginuid    # 1000（aliceのUID）

# sudo suの後:
id                         # uid=0(root) ...
cat /proc/self/loginuid    # 1000（まだalice!）

# これがauditがauid（audit uid）に使うものである:
ausearch -ua 1000          # rootとしての操作も含め、aliceのすべての操作を探す
```

```c
/* ログイン時にPAMによって設定される: */
/* /proc/<pid>/loginuidの実体はtask->loginuid */
kuid_t task_loginuid(const struct task_struct *t)
{
    return t->loginuid;
}

/* 一度設定されると変更できない（CAP_AUDIT_CONTROLなしでは） */
```

## クレデンシャルの変更を観測する {#observing-credential-changes}

```bash
# setuid/setgidの呼び出しをトレースする
bpftrace -e '
tracepoint:syscalls:sys_enter_setuid {
    printf("%s(%d) setuid(%d)\n", comm, pid, args->uid);
}'

# ケーパビリティの使用を監視する
bpftrace -e '
kprobe:cap_capable {
    printf("%s: checking cap %d in ns %p\n", comm, arg2, arg1);
}'

# クレデンシャルの変更（execve、setuidなど）を追跡する
auditctl -a always,exit -F arch=b64 \
    -S execve -S setuid -S setresuid -S setresgid \
    -k cred_change
ausearch -k cred_change -i | tail -20
```

## 参考文献 {#further-reading}

- [ケーパビリティ](capabilities.md) — ケーパビリティビットのリファレンス
- [LSMフレームワーク](lsm.md) — クレデンシャル遷移時のLSMフック
- [namespace](../cgroups/namespaces.md) — ユーザーnamespaceと他のnamespace
- [コンテナの隔離](../cgroups/container-isolation.md) — 実践的なコンテナセキュリティ
- [Linuxのaudit](audit.md) — auditはauidにloginuidを使う
- `kernel/cred.c` — クレデンシャルのライフサイクル
- `kernel/user_namespace.c` — ユーザーnamespaceの実装
