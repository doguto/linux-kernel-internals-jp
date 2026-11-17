# Linuxのauditサブシステム {#linux-audit-subsystem}

> 強制的なセキュリティ監査: syscall、ファイルアクセス、ユーザー操作の記録

## auditサブシステムが行うこと {#what-the-audit-subsystem-does}

Linuxのauditサブシステムは、セキュリティに関わるイベントの強制的かつ改ざん耐性のあるロギングを提供する:

- syscallの実行（誰が`open()`、`execve()`、`setuid()`を呼んだか）
- ファイルアクセス（誰が`/etc/shadow`を読んだか）
- 認証イベント（PAM、sudo）
- ユーザー/グループの変更
- ネットワーク接続

任意のロギング（`syslog`）とは異なり、auditレコードは監査対象のプロセスが抑止することはできない — syscallが戻る前にカーネルによって書き込まれるからである。

```bash
# auditdを起動する（通常はブート時に起動される）
systemctl start auditd
systemctl enable auditd

# ログファイル
/var/log/audit/audit.log         # auditレコード
/etc/audit/audit.rules           # 永続的なルール
/etc/audit/auditd.conf           # デーモンの設定
```

## auditctlによるauditルール {#audit-rules-with-auditctl}

```bash
# ファイルの読み書きを監視する:
auditctl -w /etc/shadow -p rwa -k shadow_access
# -w: 監視するパス、-p: パーミッション（r/w/x/a = read/write/exec/attr）
# -k: キー（検索用タグ）

# ディレクトリを監視する:
auditctl -w /etc/audit/ -p wa -k audit_config_change

# syscallを監査する:
auditctl -a always,exit -F arch=b64 -S execve -k exec_tracking

# 特定ユーザーのexecveを監査する:
auditctl -a always,exit -F arch=b64 -S execve \
         -F uid=1000 -k user_execve

# 権限昇格を監査する:
auditctl -a always,exit -F arch=b64 \
         -S setuid -S setgid -S setresuid -S setresgid \
         -k privilege_change

# 現在のルールを一覧表示する:
auditctl -l

# すべてのルールを削除する:
auditctl -D

# ルールを変更不可にする（再起動までロック）:
auditctl -e 2
```

### 永続的なルール {#persistent-rules}

```bash
# /etc/audit/rules.d/50-my-rules.rules
-w /etc/passwd -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/shadow -p wa -k identity

-a always,exit -F arch=b64 -S execve -k execve

# ブート時にルールを適用する:
augenrules --load  # /etc/audit/rules.d/*.rules をマージする
```

## auditログを読む {#reading-audit-logs}

```bash
# 生の形式:
grep "key=shadow_access" /var/log/audit/audit.log
# type=SYSCALL msg=audit(1234567890.123:456):
#   arch=c000003e syscall=2 success=yes exit=3
#   a0=7fff123 a1=0 a2=0 a3=0
#   items=1 ppid=1234 pid=5678 auid=1000 uid=0 gid=0 euid=0 suid=0
#   fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0
#   ses=1 comm="cat" exe="/usr/bin/cat" key="shadow_access"
# type=CWD msg=audit(1234567890.123:456): cwd="/root"
# type=PATH msg=audit(1234567890.123:456):
#   item=0 name="/etc/shadow" inode=12345 dev=08:01
#   mode=0100640 ouid=0 ogid=42 rdev=00:00

# ausearchで整形した形式:
ausearch -k shadow_access --interpret
ausearch -x /usr/bin/sudo -i  # sudoのすべての活動を解釈して表示

# aureportによるサマリー:
aureport --summary                # 概要
aureport -x --summary             # execveのサマリー
aureport -l                       # 失敗したログイン
aureport -au                      # 認証イベント
aureport --failed                 # 失敗したすべてのイベント

# 最近のイベント
ausearch --start today --interpret | head -100
ausearch --start recent -m EXECVE -i | tail -20
```

## カーネルの実装 {#kernel-implementation}

### struct audit_context {#struct-audit_context}

各syscallエントリはauditコンテキストを割り当てる（または再利用する）:

```c
/* include/linux/audit.h */
struct audit_context {
    int           in_syscall; /* 現在syscall内なら1 */
    enum audit_state state;   /* AUDIT_DISABLED, AUDIT_BUILD_CONTEXT, AUDIT_RECORD_CONTEXT */
    unsigned int  serial;     /* シーケンス番号 */
    struct timespec64 ctime;  /* syscallエントリの時刻 */

    int           major;      /* syscall番号 */
    unsigned long argv[4];    /* syscallの引数 */
    long          return_code; /* syscallの戻り値 */
    int           return_valid; /* 0: まだ戻っていない */

    int           name_count;
    struct audit_names names[AUDIT_NAMES]; /* アクセスされたファイルパス */
    struct audit_names *names_list;

    struct {
        uid_t         loginuid; /* /proc/self/loginuid: 元のuid */
        unsigned int  sessionid;
    };

    /* LSM固有のデータ */
    void         *lsm_entry;
};
```

### auditレコードの生成 {#audit-record-generation}

```c
/* kernel/audit.c */
void __audit_syscall_entry(int major)
/* syscallの引数は内部でcurrent_pt_regs()から読み取られる */
{
    struct task_struct *t = current;
    struct audit_context *context = audit_context();

    if (!audit_enabled)
        return;

    /* このsyscallを監査すべきか確認する */
    state = audit_filter_syscall(t, context);
    if (state == AUDIT_DISABLED)
        return;

    context->major       = major;
    context->argv[0]     = a1;
    context->argv[1]     = a2;
    context->argv[2]     = a3;
    context->argv[3]     = a4;
    context->in_syscall  = 1;
    context->state       = state;
    ktime_get_coarse_real_ts64(&context->ctime);
}

void __audit_syscall_exit(int success, long return_code)
{
    /* syscallの終了時に呼ばれる */
    audit_log_exit();  /* SYSCALLレコードをauditバッファに書き込む */
    /* Netlink（NETLINK_AUDIT）経由でauditdへ送る */
}
```

### auditのパスレコード {#audit-path-records}

```c
/* fs/namei.c: ファイルパスが解決されたときに呼ばれる */
void audit_inode(struct filename *name, const struct dentry *dentry,
                  unsigned int aflags)
{
    struct audit_context *context = audit_context();
    struct audit_names *n;

    if (!auditing_this_syscall(context))
        return;

    /* 現在のsyscallのauditレコード用にパスを記録する */
    n = audit_alloc_name(context, AUDIT_TYPE_NORMAL);
    n->name = name;
    n->dentry = dget(dentry);
    /* syscall終了時にTYPE=PATHレコードとして書き出される */
}
```

## ユーザー空間からのaudit: audit_log_user_message {#audit-from-userspace-audit_log_user_message}

```c
#include <libaudit.h>

int audit_fd = audit_open();

/* セキュリティに関わる独自のイベントを記録する */
audit_log_user_message(audit_fd, AUDIT_USER_AUTH,
                        "op=login acct=alice res=failed",
                        NULL, NULL, NULL, 0);

/* PAMは認証のロギングにこれを使う: */
/* type=USER_AUTH msg=audit(1234567890.123:789): pid=5678 uid=1000
   auid=1000 ses=1 op=login acct=alice res=failed */

audit_close(audit_fd);
```

## プログラムからauditログを照会する {#querying-audit-logs-programmatically}

```bash
# ausearch: 強力な照会ツール
# 直近1時間の/bin/shのexecveをすべて探す:
ausearch -x /bin/sh --start recent -i

# /etc/sudoersに誰がアクセスしたかを探す:
ausearch -f /etc/sudoers -i

# 特定ユーザーによるsetuid呼び出しを探す:
ausearch -ua 1000 -sc setuid -i

# JSON出力（auditd 3.0以上）:
ausearch --format enriched -i | head -50
```

## Integrity Measurement Architecture (IMA) {#integrity-measurement-architecture-ima}

IMAはauditを土台にしてファイルの完全性を提供する:

```bash
# IMAは実行前にファイルを測定し、auditに記録する:
# type=INTEGRITY_PCR msg=audit(...):
#   pid=1234 uid=0 auid=0 ses=0
#   op=hash_check
#   cause=open_id
#   comm="myapp" filename=/usr/bin/myapp
#   dev="sda1" ino=12345
#   integrity_status=pass

# IMAのロギングを有効にする:
echo "measure func=FILE_MMAP mask=MAY_EXEC" >> /etc/ima/ima-policy
```

## 参考文献 {#further-reading}

- [LSMフレームワーク](lsm.md) — auditに情報を供給するLSMフック
- [ケーパビリティ](capabilities.md) — ケーパビリティの変更はauditイベントを発生させる
- [seccomp BPF](seccomp.md) — syscallフィルタリング。auditはseccompによる拒否を記録する
- [クレデンシャルとユーザーnamespace](credentials.md) — auditのauid（ログインuid）
- `kernel/audit.c` — auditのコア実装
- `kernel/auditsc.c` — syscallのauditレコード
- `man 8 auditctl`, `man 8 ausearch`, `man 8 aureport`
