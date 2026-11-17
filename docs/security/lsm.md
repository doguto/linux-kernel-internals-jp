# Linux Security Module（LSM）フレームワーク {#linux-security-module-lsm-framework}

> カーネルのフックベースの強制アクセス制御基盤

## LSMとは {#what-lsm-is}

Linux Security Moduleフレームワーク（Linux 2.6.0、2003年12月に導入）は、カーネル全体にわたってフックの集合を提供し、セキュリティモジュールが標準のDACを超える追加のアクセス制御を適用できるようにする。Crispin Cowan（WireX）がNSA、SGIなどと共同で設計し、USENIX Security 2002で発表された: ["Linux Security Modules: General Security Support for the Linux Kernel"](https://www.usenix.org/conference/11th-usenix-security-symposium/linux-security-modules-general-security-support-linux)。カーネルドキュメント: [security/lsm.html](https://www.kernel.org/doc/html/latest/security/lsm.html)。

LSMフックはセキュリティ上重要な箇所で呼び出される:

- ファイルのopen/read/write/execute
- ネットワークソケットの作成と接続
- プロセスの作成とクレデンシャルの変更
- IPC操作
- ケーパビリティのチェック

複数のLSMを同時に有効にできる（スタッキング。Linux 4.2以降 — Casey Schauflerによる[`3c4ed7bdf599`](https://git.kernel.org/linus/3c4ed7bdf5997d8020cbb8d4abbef2fcfb9f1284)）。カーネルはそのすべてを評価し、*すべての*LSMが許可した場合にのみアクセスが許可される。

## LSMフックのアーキテクチャ {#lsm-hook-architecture}

```c
/* security/security.c: フックのディスパッチ */

/* すべてのLSMフックリスト（フックごとに1リスト。ポインタではない） */
static struct security_hook_heads security_hook_heads;

/* 例: file_openフック */
int security_file_open(struct file *file)
{
    return call_int_hook(file_open, 0, file);
    /* 登録された各LSMのfile_openハンドラを呼ぶ */
    /* 最初の非ゼロの結果（拒否）を返す */
}
```

```c
/* セキュリティモジュールは自分のフックを登録する: */
static struct security_hook_list mymod_hooks[] __lsm_ro_after_init = {
    LSM_HOOK_INIT(file_open, mymod_file_open),
    LSM_HOOK_INIT(socket_connect, mymod_socket_connect),
    LSM_HOOK_INIT(inode_permission, mymod_inode_permission),
};

static int __init mymod_init(void)
{
    security_add_hooks(mymod_hooks, ARRAY_SIZE(mymod_hooks), "mymod");
    return 0;
}
DEFINE_LSM(mymod) = {
    .name    = "mymod",
    .init    = mymod_init,
    .enabled = &mymod_enabled,
};
```

## LSMフックのカテゴリ一覧 {#complete-list-of-lsm-hook-categories}

```c
/* include/linux/lsm_hooks.h — 200を超えるフック: */

/* ファイルとinodeの操作 */
file_open, file_permission, file_ioctl, file_mmap
inode_permission, inode_create, inode_link, inode_unlink
inode_mkdir, inode_rmdir, inode_rename, inode_readlink
inode_setattr, inode_getattr, inode_setxattr, inode_getxattr

/* プロセスのクレデンシャル */
bprm_check_security, bprm_creds_for_exec
task_alloc, task_free, task_fix_setuid
cred_alloc_blank, cred_prepare, cred_transfer

/* ネットワーク */
socket_create, socket_bind, socket_connect
socket_listen, socket_accept, socket_recvmsg, socket_sendmsg
socket_sock_rcv_skb, socket_getpeersec_stream

/* IPC */
msg_queue_associate, msg_queue_msgctl
sem_associate, sem_semctl
shm_associate, shm_shmctl, shm_shmat

/* ケーパビリティ */
capable, settime

/* システム */
syslog, quotactl, sysctl, getprocattr, setprocattr

/* BPF */
bpf, bpf_map, bpf_prog
```

## SELinux {#selinux}

SELinux（Security-Enhanced Linux）はNSAが開発し、2000年12月にオープンソース化され、Linux 2.6.0-test3（2003年8月）にマージされた。**ラベル**と**ポリシー**に基づく強制アクセス制御を実装している。

### ラベル {#labels}

すべてのオブジェクト（ファイル、プロセス、ソケット）はセキュリティコンテキストを持つ:
```
user:role:type:level
system_u:system_r:httpd_t:s0
```

```bash
# ファイルのコンテキストを見る
ls -Z /etc/passwd
# system_u:object_r:passwd_file_t:s0 /etc/passwd

# プロセスのコンテキストを見る
ps -eZ | grep httpd
# system_u:system_r:httpd_t:s0 ... httpd

# 自分のコンテキストを見る
id -Z
# unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

### ポリシー {#policy}

ポリシーは、どのタイプがどの他のタイプに対してどの操作を行えるかを定義する:

```
# httpdに証明書ファイルの読み取りを許可する
allow httpd_t cert_t:file { read getattr open };

# httpdにhttpポートへのbindを許可する
allow httpd_t http_port_t:tcp_socket name_bind;
```

```bash
# SELinuxの状態を確認する
getenforce
# Enforcing | Permissive | Disabled

sestatus
# SELinux status:                 enabled
# SELinuxfs mount:                /sys/fs/selinux
# SELinux mount point:            /sys/fs/selinux
# SELinuxfs mount:                /sys/fs/selinux

# permissiveに設定する（ログは取るが拒否しない）
setenforce 0

# 拒否についてauditログを確認する
ausearch -m AVC | tail -20
# type=AVC msg=audit(1234.567:890): avc:  denied  { read } for ...
# pid=1234 comm="nginx" name="secret.key" scontext=httpd_t tcontext=sysadm_home_t

# 修正: 正しいラベルを設定する
chcon -t httpd_sys_content_t /var/www/html/myfile
restorecon /var/www/html/  # ポリシーのデフォルトに戻す
```

### SELinuxの内部 {#selinux-internals}

```c
/* security/selinux/hooks.c: inode_permissionフック */
static int selinux_inode_permission(struct inode *inode, int mask)
{
    const struct cred *cred = current_cred();
    struct inode_security_struct *isec;
    struct task_security_struct *tsec;
    u32 sid, newsid, perms;

    /* セキュリティラベルを取得する */
    tsec = selinux_cred(cred);
    isec = selinux_inode(inode);

    /* アクセスマスクをSELinuxのパーミッションビットに変換する */
    perms = file_mask_to_av(inode->i_mode, mask);

    /* SELinuxのポリシーエンジンを呼ぶ */
    rc = avc_has_perm(&selinux_state,
                      tsec->sid, isec->sid,
                      isec->sclass, perms, &ad);
    return rc;
}
```

## AppArmor {#apparmor}

AppArmor（元はImmunixによる「SubDomain」、1998年。2009年以降はCanonicalがメンテナンス）は、John JohansenによってLinux 2.6.36（2010年10月）にマージされた — [`b5e95b48685e`](https://git.kernel.org/linus/b5e95b48685e3481139a5634d14d630d12c7d5ce)。**パスベース**のMACを使い、SELinuxのラベルモデルより単純である。プロファイルはプレーンテキストで、プログラムに何ができるかを指定する:

```
# /etc/apparmor.d/usr.bin.firefox
/usr/bin/firefox {
    # これらのパスの読み取りを許可する
    /etc/firefox/** r,
    /usr/share/firefox/** r,
    @{HOME}/.mozilla/** rw,

    # それ以外はデフォルトですべて拒否する
    deny /etc/passwd r,

    # ネットワーク
    network inet stream,
    network inet6 stream,
}
```

```bash
# プロファイルをロードする
apparmor_parser -r /etc/apparmor.d/usr.bin.firefox

# 状態を確認する
aa-status
# profiles are loaded:
#   /usr/bin/firefox (enforce)
#   /usr/bin/evince (complain)

# プロファイルをcomplainモードにする（ログは取るが拒否しない）
aa-complain /usr/bin/firefox

# ログを確認する
journalctl -g "apparmor" | tail -20
# apparmor="DENIED" operation="open" profile="/usr/bin/firefox"
# name="/etc/shadow" pid=1234 comm="firefox"
```

### AppArmorの内部 {#apparmor-internals}

AppArmorは効率的なパスマッチングのため、プロファイルを有限オートマトン（DFA/HFA）として保持する:

```c
/* security/apparmor/lsm.c: file_openフック */
static int apparmor_file_open(struct file *file)
{
    struct aa_label *label = aa_current_raw_label();
    struct aa_profile *profile;

    /* パスをプロファイルのDFAと照合する */
    aa_str_perms(profile->file.dfa,
                 profile->file.start,
                 path_name,
                 &perms);

    if (!(perms.allow & requested_perm))
        return -EACCES;
    return 0;
}
```

## 参考文献 {#further-reading}

- [ケーパビリティ](capabilities.md) — 完全なrootを使わない権限分割
- [seccomp BPF](seccomp.md) — syscallレベルのフィルタリング
- [cgroupとnamespace: コンテナの隔離](../cgroups/container-isolation.md) — コンテナにおけるLSM
- `Documentation/security/lsm.rst` — LSM開発者ガイド
