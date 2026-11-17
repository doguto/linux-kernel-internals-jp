# seccomp BPF {#seccomp-bpf}

> 権限削減と攻撃対象領域の最小化のためのsyscallフィルタリング

## seccompが行うこと {#what-seccomp-does}

seccomp（secure computing mode）は、すべてのsyscallエントリで実行されるBPFプログラムをインストールする。フィルタは次のことができる:

- syscallの続行を許可する
- 呼び出し元にエラーを返す（カーネルでの処理は行わない）
- プロセスをkillする
- イベントを記録する
- ユーザー空間のプロセスに通知する（SECCOMP_RET_USER_NOTIF）

```
Process → syscall instruction
             │
             ▼
       seccomp filter runs   (Classic BPF on struct seccomp_data)
             │
     ┌───────┴───────────┐
     │                   │
  ALLOW              KILL_PROCESS / ERRNO / TRAP / USER_NOTIF
     │
     ▼
 kernel handles syscall
```

## strictモードからseccomp-BPFへ {#from-strict-mode-to-seccomp-bpf}

### 最初のseccomp（Linux 2.6.12、2005年） {#the-original-seccomp-linux-2612-2005}

Andrea Arcangeliは、特定の脅威に対処するためにseccompを導入した — [`d949d0ec9c60`](https://git.kernel.org/linus/d949d0ec9c601f2b148bed3cdb5f87c052968554)。その脅威とは、計算用サンドボックス（たとえば商用の「安全なブラウジング」やグリッドコンピューティングのサービス）内で動く信頼できないコードである。設計は最大限に制限的だった。プロセスが`prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT)`を呼ぶと、以後は`read`、`write`、`exit`、`sigreturn`の4つのsyscallしか使えなくなる。それ以外のsyscallを呼ぶとSIGKILLでkillされた。

これは安全ではあったが、実際のアプリケーションにはまったく使えなかった。ファイルを開いたり、メモリを割り当てたり、ネットワークとやり取りしたりする必要のあるアプリケーションは、strictモードをまったく使えなかった。

### なぜカーネルは（BPFなしでは）それ以上のことができなかったか {#why-the-kernel-couldnt-do-more-without-bpf}

制約はポリシーの表現にあった。「`read`と`write`だけを許可する」は単純である。「サンドボックスに入る前に開かれていたファイルディスクリプタに対する`read`と`write`は許可するが、`open`は許可しない」は状態を持ち複雑である。「`PROT_READ`での`mmap`は許可するが`PROT_WRITE|PROT_EXEC`は許可しない」は引数の検査を必要とする。考えうるすべてのポリシーをカーネル内にエンコードするには、ドメイン固有言語が必要になったはずだ — それこそがまさにBPFである。

### seccomp-BPF（Linux 3.5、2012年） {#seccomp-bpf-linux-35-2012}

Will Drewry（Google Chromeチーム）は、Chromeのサンドボックスの要件を動機としてseccomp-BPF拡張を書いた — [`e2cfabdfd075`](https://git.kernel.org/linus/e2cfabdfd075648216f99c2c03821cf3f47c1727)、[LWN](https://lwn.net/Articles/475043/)。Chromeはレンダラプロセスをカーネルから隔離する必要があった — レンダラが正当に必要とするsyscallだけを許可し、許可したsyscallでさえ悪用されないよう引数をフィルタリングするのである。

鍵となる洞察は次のとおりである。Classic BPF（既存のソケットフィルタVM）は、すでにカーネルのデータを検査するための汎用ポリシー言語だった。`struct seccomp_data` — syscall番号と6つの引数 — は64バイトの構造体にすぎない。cBPFプログラムはその構造体の任意のフィールドを検査し、ALLOWかDENYを返せる。

seccomp-BPFは新しいポリシー言語を作る代わりに、cBPFをそのまま再利用した。カーネルはすべてのsyscallエントリでBPFプログラムを実行し、プログラムはアクションコードを返す。これには新しいカーネルの仕組みはほとんど必要なかった — BPFのインタプリタ/JITはすでに存在していたからである。

ChromeはChrome 23（2012年）でseccomp-BPFを搭載し、この方法で自らをサンドボックス化した最初の主要アプリケーションとなった。systemd、Docker、Firefox、Androidはいずれもseccomp-BPFフィルタを採用した。`SECCOMP_RET_USER_NOTIF`拡張（Linux 5.0、2019年 — Tycho Andersenによる[`6a21cc50f0c7`](https://git.kernel.org/linus/6a21cc50f0c7f87dae5259f6cfefe024412313f6)）により、スーパーバイザがサンドボックス化されたプロセスに代わってsyscallを横取りして処理できるようになり、rootなしでのユーザーnamespaceコンテナのエミュレーションが可能になった。

## クラシック（strict）モード {#classic-strict-mode}

BPFフィルタ以前、seccompは1つのモードしか提供していなかった:

```c
/* read、write、exit、sigreturnのみを許可する */
prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT);
/* それ以外のsyscall → SIGKILL */
```

## BPFフィルタモード {#bpf-filter-mode}

完全なseccompは、`struct seccomp_data`を対象に動作するClassic BPFプログラムを使う:

```c
/* include/uapi/linux/seccomp.h */
struct seccomp_data {
    int     nr;                     /* syscall番号 */
    __u32   arch;                   /* AUDIT_ARCH_* */
    __u64   instruction_pointer;    /* syscall時のrip */
    __u64   args[6];                /* syscallの引数 */
};
```

### フィルタの戻り値 {#filter-return-values}

```c
SECCOMP_RET_KILL_PROCESS  /* プロセス全体をSIGSYSでkillする（最も厳しい） */
SECCOMP_RET_KILL_THREAD   /* 呼び出したスレッドだけをkillする */
SECCOMP_RET_TRAP          /* SIGSYSを配送する（シグナルハンドラで捕捉可能） */
SECCOMP_RET_ERRNO         /* エラーを返す: SECCOMP_RET_DATAにerrnoが入る */
SECCOMP_RET_USER_NOTIF    /* スーパーバイザプロセスに通知する（5.0以降） */
SECCOMP_RET_TRACE         /* ptraceデバッガに通知する */
SECCOMP_RET_LOG           /* 許可するがsyscallを記録する */
SECCOMP_RET_ALLOW         /* 許可する（最も寛容） */
```

戻り値の優先順位（最も低い値が勝つ）: KILL > TRAP > ERRNO > USER_NOTIF > TRACE > LOG > ALLOW。

### 生のBPFフィルタの例 {#raw-bpf-filter-example}

```c
#include <sys/prctl.h>
#include <linux/seccomp.h>
#include <linux/filter.h>
#include <linux/audit.h>
#include <sys/syscall.h>

static int install_filter(void)
{
    struct sock_filter filter[] = {
        /* アーキテクチャをロードする */
        BPF_STMT(BPF_LD | BPF_W | BPF_ABS,
                 (offsetof(struct seccomp_data, arch))),
        /* x86-64であることを検証する */
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K,
                 AUDIT_ARCH_X86_64, 1, 0),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL_PROCESS),

        /* syscall番号をロードする */
        BPF_STMT(BPF_LD | BPF_W | BPF_ABS,
                 (offsetof(struct seccomp_data, nr))),

        /* readを許可する */
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_read, 0, 1),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),

        /* writeを許可する */
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_write, 0, 1),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),

        /* exit_groupを許可する */
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_exit_group, 0, 1),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),

        /* デフォルト: kill */
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL_PROCESS),
    };

    struct sock_fprog prog = {
        .len    = ARRAY_SIZE(filter),
        .filter = filter,
    };

    /* フィルタをインストールする前にPR_SET_NO_NEW_PRIVSを設定しなければならない
       （rootで実行している場合を除く） */
    if (prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0) < 0)
        return -1;

    if (prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog) < 0)
        return -1;

    return 0;
}
```

### libseccomp: 高水準API {#libseccomp-high-level-api}

生のBPFは間違いやすい。代わりに`libseccomp`を使うこと:

```c
#include <seccomp.h>

/* 許可リスト方式: すべてをブロックした状態から始める */
scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL_PROCESS);

/* 許可を追加する */
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(mmap), 0);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(munmap), 0);

/* 条件付きルール: 読み取り専用フラグのopenatのみを許可する */
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(openat), 1,
    SCMP_A2(SCMP_CMP_MASKED_EQ, O_WRONLY | O_RDWR, 0));
/*          ^ 引数2（flags）をO_WRONLY|O_RDWRでマスクした値が0でなければならない */

/* フィルタをロードする */
seccomp_load(ctx);
seccomp_release(ctx);
```

```bash
# libseccompとリンクする
gcc -o myapp myapp.c -lseccomp
```

## コンテナでの利用: Dockerのデフォルトプロファイル {#container-use-dockers-default-profile}

Dockerは約40個の危険なsyscallをブロックするseccompプロファイルをインストールする:

```bash
# Dockerのデフォルトのseccompプロファイルを表示する
docker inspect --format '{{.HostConfig.SecurityOpt}}' container_name

# seccompなしで実行する（デバッグ用）
docker run --security-opt seccomp=unconfined ubuntu bash

# 独自のプロファイルで実行する
docker run --security-opt seccomp=/path/to/profile.json ubuntu bash
```

Dockerのデフォルトプロファイルは（特に）次のものをブロックする:

- `acct` — プロセスアカウンティング
- `add_key` — カーネルのキーリング
- `bpf` — BPFプログラム
- `kexec_load` — カーネルの置き換え
- `mount` — ファイルシステムのマウント
- `perf_event_open` — 性能監視
- `ptrace` — プロセスのトレース
- `swapon/swapoff` — スワップの管理
- `syslog` — カーネルのリングバッファの読み取り
- `uselib` — 古いライブラリのロード

## SECCOMP_RET_USER_NOTIF: スーパーバイザパターン {#seccomp_ret_user_notif-supervisor-pattern}

Linux 5.0以降、フィルタは判断をスーパーバイザプロセスに委ねられる。これにより次のことが可能になる:

- ユーザー空間でのポリシーの適用
- コンテナエスケープの防止（コンテナマネージャがマウントを承認する）
- デバッグ（スーパーバイザが許可したsyscallを記録する）

```c
/* スーパーバイザプロセス: 通知を受け取る */
int notif_fd = seccomp_notify_fd(ctx);  /* seccomp_loadの後 */

struct seccomp_notif *notif = /* 割り当てる */;
struct seccomp_notif_resp *resp = /* 割り当てる */;

while (1) {
    ioctl(notif_fd, SECCOMP_IOCTL_NOTIF_RECV, notif);

    /* 検査する: notif->pid、notif->data.nr（syscall）、notif->data.args */
    if (should_allow(notif))
        resp->error  = 0;        /* 許可 */
        resp->flags  = SECCOMP_USER_NOTIF_FLAG_CONTINUE;
    else
        resp->error  = -EPERM;   /* 拒否 */

    resp->id = notif->id;
    ioctl(notif_fd, SECCOMP_IOCTL_NOTIF_SEND, resp);
}
```

rootlessコンテナランタイム（Podman、nerdctl）は、このようにしてコンテナに`mount`のような特権操作を行わせている — スーパーバイザがそれを横取りし、安全に処理するのである。

## カーネルのseccomp実装 {#kernel-seccomp-implementation}

```c
/* kernel/seccomp.c: syscallエントリのパス内 */
static int __seccomp_filter(u32 this_syscall, const struct seccomp_data *sd,
                             const bool recheck_after_trace)
{
    u32 filter_ret, data;
    struct seccomp_filter *match = NULL;
    int action;

    /* インストール済みのすべてのフィルタを実行する（チェーンされている） */
    filter_ret = run_filters(sd, &match);

    data    = filter_ret & SECCOMP_RET_DATA;
    action  = filter_ret & SECCOMP_RET_ACTION_FULL;

    switch (action) {
    case SECCOMP_RET_ALLOW:
        return 0;
    case SECCOMP_RET_KILL_PROCESS:
        do_group_exit(SIGSYS);  /* スレッドグループ全体をkillする */
    case SECCOMP_RET_KILL_THREAD:
        do_exit(SIGSYS);        /* 呼び出したスレッドだけをkillする */
    case SECCOMP_RET_ERRNO:
        syscall_set_return_value(current, current_pt_regs(),
                                  -data, 0);
        return -1;
    /* ... */
    }
}
```

## seccompのアクションを監査する {#auditing-seccomp-actions}

```bash
# プロセスの現在のseccompモードを確認する
cat /proc/self/status | grep Seccomp
# Seccomp: 2   （0=off、1=strict、2=filter）

# auditログでseccompの違反を確認する
ausearch -m SECCOMP | head -20
# type=SECCOMP msg=audit(1234.567:890): avc:  seccomp
# sig=31 arch=c000003e syscall=62 compat=0 ip=0x7f... code=0x80000000

# seccompと一緒にstraceを使う
strace -e seccomp ./myprogram
```

## 参考文献 {#further-reading}

- [LSMフレームワーク](lsm.md) — seccompの後に（個々のsyscall実装内で）実行されるMACポリシー
- [ケーパビリティ](capabilities.md) — seccompの前に行う権限の削減
- [BPF: アーキテクチャ](../bpf/bpf-overview.md) — seccompフィルタとしてのBPFプログラム
- [cgroup: コンテナの隔離](../cgroups/container-isolation.md) — コンテナにおけるseccomp
- `man 2 seccomp` — syscallのドキュメント
