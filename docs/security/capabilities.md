# Linuxのケーパビリティ {#linux-capabilities}

> きめ細かな権限の分解

## rootの問題点 {#the-problem-with-root}

伝統的に、Unixにおける権限は二値である。プロセスはUID 0（root、全能）を持つか持たないかのどちらかだ。これは粗すぎる — Webサーバーはポート80にbindする必要があるが、rawネットワークソケットへのアクセス、カーネルモジュールのロード、システムの再起動はできるべきではない。

Linuxのケーパビリティは、rootの権力を個別に付与できる約40個の権限に分割する。撤回されたPOSIX.1eドラフト標準に基づき、Linux 2.2（1999年1月）で導入された。ファイルケーパビリティのサポート（setuidなしでバイナリにケーパビリティを持たせる機能）は、Serge HallynによってLinux 2.6.24で追加された — [`b53767719b6c`](https://git.kernel.org/linus/b53767719b6cd8789392ea3e7e2eb7b8906898f0)。[`man 7 capabilities`](https://man7.org/linux/man-pages/man7/capabilities.7.html)を参照。

## 40個のケーパビリティ {#the-40-capabilities}

```c
/* include/uapi/linux/capability.h */

/* ネットワーク */
CAP_NET_BIND_SERVICE  /* 1024未満のポートにbindする */
CAP_NET_RAW           /* rawソケット、ICMPを使う */
CAP_NET_ADMIN         /* ネットワークインターフェース、ルーティングを設定する */
CAP_NET_BROADCAST     /* ソケットでブロードキャストする */

/* プロセス制御 */
CAP_KILL             /* 任意のプロセスにシグナルを送る */
CAP_SETUID           /* 任意のUIDを設定する */
CAP_SETGID           /* 任意のGIDを設定する */
CAP_SETPCAP          /* ケーパビリティを設定/放棄する */

/* システム管理 */
CAP_SYS_ADMIN        /* 多数の特権操作（範囲が広すぎる） */
CAP_SYS_BOOT         /* reboot() syscall */
CAP_SYS_MODULE       /* カーネルモジュールをロード/アンロードする */
CAP_SYS_TIME         /* システムクロックを設定する */
CAP_SYS_PTRACE       /* 任意のプロセスをptraceする */
CAP_SYS_RAWIO        /* iopl、直接のポートI/O */
CAP_SYS_CHROOT       /* chroot() */
CAP_SYS_NICE         /* 負のnice値、スケジューリングクラスを設定する */
CAP_SYS_RESOURCE     /* リソース制限を設定し、ハードリミットを上書きする */

/* ファイル操作 */
CAP_CHOWN            /* ファイルの所有者を変更する */
CAP_DAC_OVERRIDE     /* ファイルパーミッションのチェックを迂回する */
CAP_DAC_READ_SEARCH  /* 読み取り/検索のチェックを迂回する */
CAP_FOWNER           /* ファイル所有者のチェックを迂回する */
CAP_FSETID           /* ファイルにsetuid/setgidを設定する */
CAP_LINUX_IMMUTABLE  /* immutable/append-onlyのファイルフラグを設定する */
CAP_MKNOD            /* スペシャルファイルを作成する */

/* IPC */
CAP_IPC_LOCK         /* メモリをロックする（mlock） */
CAP_IPC_OWNER        /* IPCの所有者チェックを上書きする */

/* セキュリティ */
CAP_MAC_ADMIN        /* MACポリシーを管理する（SELinux/AppArmor） */
CAP_MAC_OVERRIDE     /* MACを上書きする（SELinux） */
CAP_AUDIT_CONTROL    /* auditを設定する */
CAP_AUDIT_READ       /* auditログを読む */
CAP_AUDIT_WRITE      /* auditログに書く */

/* BPF */
CAP_BPF              /* BPFプログラムをロードし実行する（5.8以降） */
CAP_PERFMON          /* トレーシングのためのperf_event_open（5.8以降） */

/* チェックポイント/リストア */
CAP_CHECKPOINT_RESTORE  /* CRIUのリストア操作 */
```

## ケーパビリティセット {#capability-sets}

各プロセスは**5つのケーパビリティセット**を持つ:

```c
/* スレッドごとのケーパビリティの状態 */
struct thread_info {
    /* ... */
};

/* struct cred内: */
struct cred {
    /* ... */
    kernel_cap_t    cap_inheritable; /* execをまたいで継承できる */
    kernel_cap_t    cap_permitted;   /* effectiveの上位集合 */
    kernel_cap_t    cap_effective;   /* 実際にチェックに使われる */
    kernel_cap_t    cap_bset;        /* バウンディングセット: cap_permittedの上限 */
    kernel_cap_t    cap_ambient;     /* ambient: exec時に自動的にpermittedに追加される（4.3以降） */
};
```

規則:

- **Effective**: アクセスチェック（`capable()`）で使われる
- **Permitted**: effectiveの上位集合。放棄したeffectiveのケーパビリティを再追加できる
- **Inheritable**: ファイル側も持っていれば、execをまたいで引き渡せる
- **Bounding**: 上限 — bsetにあるケーパビリティはpermittedに入れられる。一度放棄すると取り戻せない
- **Ambient**: 子プロセスとexecをまたいで自動的に継承される（Linux 4.3以降 — Andy Lutomirskiによる[`58319057b784`](https://git.kernel.org/linus/58319057b7849f6272afebf1ffeed1da70e5f68a)）

## カーネル内でのケーパビリティのチェック {#checking-capabilities-in-the-kernel}

```c
/* カーネル内のどこでも: */
if (!capable(CAP_SYS_ADMIN))
    return -EPERM;

/* namespaceを考慮したチェック: */
if (!ns_capable(net->user_ns, CAP_NET_ADMIN))
    return -EPERM;

/* タスクを明示して: */
if (!has_capability(task, CAP_KILL))
    return -EPERM;
```

`capable()`は`current->cred->cap_effective`をチェックする。

## ケーパビリティの表示と変更 {#viewing-and-modifying-capabilities}

```bash
# 現在のプロセスのケーパビリティを表示する
cat /proc/self/status | grep -i cap
# CapInh: 0000000000000000
# CapPrm: 0000003fffffffff
# CapEff: 0000003fffffffff
# CapBnd: 0000003fffffffff
# CapAmb: 0000000000000000

# ケーパビリティのビットマスクをデコードする
capsh --decode=0000003fffffffff
# 0x0000003fffffffff=cap_chown,cap_dac_override,cap_dac_read_search,...

# 実行中のプロセスのケーパビリティを表示する
cat /proc/1234/status | grep Cap

# 特定のケーパビリティでコマンドを実行する
capsh --caps="cap_net_bind_service+eip" -- -c "/usr/sbin/nginx"

# すべてのケーパビリティを放棄する（セキュリティのハードニング）
capsh --drop=all -- -c "myprogram"
```

## ファイルケーパビリティ（setcap） {#file-capabilities-setcap}

実行ファイルは拡張属性にケーパビリティを持つことができ、setuidなしで昇格した権限で実行できる:

```bash
# 実行ファイルにケーパビリティを付与する
setcap cap_net_bind_service=+eip /usr/bin/node

# ファイルのケーパビリティを表示する
getcap /usr/bin/node
# /usr/bin/node cap_net_bind_service=eip

# すべてのケーパビリティを削除する
setcap -r /usr/bin/node

# よくある使用例:
setcap cap_net_bind_service=+eip /usr/bin/node  # ポート80でNode.jsを動かす
setcap cap_net_raw=+eip /usr/bin/ping           # rootなしでraw ICMP
setcap cap_dac_read_search=+eip /usr/sbin/rsync  # rootなしでバックアップ
```

`e`、`i`、`p`フラグ:

- `e` = effective
- `i` = inheritable
- `p` = permitted

## ケーパビリティに基づく権限の放棄 {#capability-based-privilege-dropping}

適切に書かれたデーモンはrootで起動し、必要なケーパビリティを取得してから、残りを放棄する:

```c
/* デーモンの起動処理: */
static void drop_privileges(void)
{
    /* 必要なものだけを残す: CAP_NET_BIND_SERVICE */
    cap_t caps = cap_get_proc();

    cap_value_t keep[] = { CAP_NET_BIND_SERVICE };

    /* すべてのケーパビリティをクリアし、必要なものだけを設定する */
    cap_clear(caps);
    cap_set_flag(caps, CAP_EFFECTIVE,   1, keep, CAP_SET);
    cap_set_flag(caps, CAP_PERMITTED,   1, keep, CAP_SET);
    cap_set_flag(caps, CAP_INHERITABLE, 0, NULL, CAP_CLEAR);

    if (cap_set_proc(caps) < 0)
        err(1, "cap_set_proc");
    cap_free(caps);

    /* 再取得を防ぐため、バウンディングセットからも放棄する */
    if (prctl(PR_CAPBSET_DROP, CAP_SYS_ADMIN, 0, 0, 0) < 0)
        err(1, "PR_CAPBSET_DROP");

    /* 非rootのuid/gidに変更する */
    if (setgid(www_gid) < 0 || setuid(www_uid) < 0)
        err(1, "setuid/setgid");
}
```

## No-New-Privileges {#no-new-privileges}

`PR_SET_NO_NEW_PRIVS`は、プロセスがexecを通じて新たな権限を得ることを防ぐ（setuidバイナリやファイルケーパビリティ経由であっても）:

```c
prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
/* これ以降: execve()はケーパビリティを得ることも、setuidでuidを変えることもできない */
/* これはseccompフィルタのインストールによっても設定される */
```

```bash
# no new privilegesで実行する（プログラム自身の中でprctlにより設定する）
# prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
unshare --user bash
```

## CAP_SYS_ADMIN: 問題のケーパビリティ {#cap_sys_admin-the-problem-capability}

`CAP_SYS_ADMIN`は、無関係な操作をあまりに多く含むため、「新しいroot」と呼ばれることがある:

- ファイルシステムのマウント/アンマウント
- ネットワークnamespaceの設定
- `ptrace`（制限付き）
- カーネルのキーリングへのアクセス
- ディスククォータの変更
- リソース制限の上書き
- ホスト名の設定（UTS namespace）
- ……ほか40以上

可能な限り、より限定的なケーパビリティを使うべきである。ライブラリが「CAP_SYS_ADMINが必要」と言っている場合は、より狭いケーパビリティで動くかどうかを調べること。

## 参考文献 {#further-reading}

- [LSMフレームワーク](lsm.md) — ケーパビリティチェックの上に載るMAC
- [seccomp BPF](seccomp.md) — syscallレベルのフィルタリング
- [cgroupとnamespace: ユーザーnamespace](../cgroups/namespaces.md) — ユーザーnamespaceにおけるケーパビリティ
- `man 7 capabilities` — ケーパビリティの完全なリファレンス
