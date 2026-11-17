# syscallの監査 {#syscall-auditing}

> Linux Auditサブシステム: 誰が何をいつ呼んだかを記録する

## auditサブシステムが行うこと {#what-the-audit-subsystem-does}

Linuxのauditサブシステムは、セキュリティに関係するイベントを記録し、それらをユーザー空間のデーモン（`auditd`）が
ログ記録、フィルタリング、分析に使えるようにする。
主な用途はコンプライアンスである。Common Criteria評価、PCI-DSS、HIPAAなどの
フレームワークはいずれも、特定のシステムイベントがログに記録されたこと、そしてそのログが
監査対象のプロセス自身によって抑止されえないことの証明を求める。

監査対象の各syscallについて、カーネルは次のものを記録する。

- syscall番号、引数、戻り値
- PID、UID、EUID、そしてauditログインUID（`auid`）— `setuid()` による遷移の後であっても、
  最初に認証したユーザーのUID
- 実行ファイルのパス（`exe`）とカレントワーキングディレクトリ
- アクセスされたファイルパス、使われたソケットアドレス、触れられたIPCオブジェクト

レコードは、syscallがユーザー空間に戻る前にカーネルによって書き込まれる。
監査対象のプロセスがそれを抑止したり改変したりすることはできない。

## アーキテクチャ {#architecture}

```
   kernel                          userspace
   ──────                          ─────────
   audit_syscall_entry()
   audit_syscall_exit()
        │
        │  audit_buffer records
        ▼
   kernel netlink socket
   (NETLINK_AUDIT, family 9)
        │
        │  AF_NETLINK messages
        ▼
      auditd
        │
        ├── /var/log/audit/audit.log
        └── audisp plugins (e.g., syslog, remote logging)
```

`NETLINK_AUDIT` を開くのは `auditd` だけである。auditルールは
`auditctl` を介して設定され、`auditctl` は `AUDIT_ADD_RULE` や
`AUDIT_DEL_RULE` 型のnetlinkメッセージをカーネルに送る。カーネルは `AUDIT_SYSCALL`、
`AUDIT_PATH`、`AUDIT_SOCKADDR` などの型のレコードを、同じソケットを通じて `auditd` に
送り返す。

## カーネル内のフック: audit_syscall_entry() とaudit_syscall_exit() {#in-kernel-hooks-audit_syscall_entry-and-audit_syscall_exit}

auditサブシステムへの2つの主要なエントリポイントは、汎用の
syscall境界コードから呼ばれる。

```c
/* include/linux/audit.h */
static inline void audit_syscall_entry(int major, unsigned long a0,
                                        unsigned long a1, unsigned long a2,
                                        unsigned long a3);
static inline void audit_syscall_exit(void *pt_regs);
```

どちらも `static inline` のラッパーであり、何か処理をする前に `audit_context()` をチェックし、
その後 `kernel/auditsc.c` で定義された実際の実装 — `__audit_syscall_entry()` と
`__audit_syscall_exit()` — に転送する。

これらは、現在のタスクの `syscall_work` に `SYSCALL_WORK_SYSCALL_AUDIT` フラグが設定されているとき、
`include/linux/entry-common.h` の `syscall_enter_from_user_mode()` と
`syscall_exit_to_user_mode()` から呼ばれる。

`SYSCALL_WORK_SYSCALL_AUDIT` は、タスクにコンテキストが割り当てられるたびに `audit_alloc()` で設定される —
そしてauditがシステム全体で有効になれば、それが一般的なケースになる
（次節を参照）。タスクレベルのルールが明示的に無効化したタスクか、
auditが一度も有効にされたことのないシステムだけが、これを完全にスキップする。

## struct audit_context {#struct-audit_context}

監査対象の各タスクは、syscallの間 `audit_context` を保持する。
このコンテキストには、syscallと、それが触れたカーネルオブジェクトに関するすべての情報が蓄積される。

```c
/* kernel/audit.h（内部） */
struct audit_context {
    int                dummy;       /* ダミーのコンテキストなら1 */
    enum {
        AUDIT_CTX_UNUSED,
        AUDIT_CTX_SYSCALL,
        AUDIT_CTX_URING,
    }                  context;
    enum audit_state   state;         /* audit_filter_task() による基準値。audit_alloc() で一度だけ設定される */
    enum audit_state   current_state; /* syscallごと。各syscallで 'state' にリセットされ、audit_filter_syscall() で絞り込まれる */

    int                major;        /* syscall番号 */
    unsigned long      argv[4];      /* syscallの最初の4引数 */
    long               return_code;  /* syscallの戻り値 */
    int                return_valid; /* return_code が設定されたら1 */

    int                name_count;
    struct list_head   names_list;   /* struct audit_names のリスト */

    struct audit_stamp stamp;        /* serial と ctime を含む */

    /* ... IPC、ネットワーク、LSM固有のデータのためのさらなるフィールド ... */
};
```

注: `serial` と `ctime` は、ネストした `struct audit_stamp` を介して `ctx->stamp.serial` と
`ctx->stamp.ctime` としてアクセスされる。`loginuid` と
`sessionid` は `struct audit_context` のフィールドではない — これらは
`struct task_struct` に直接置かれている（`tsk->loginuid`、`tsk->sessionid`）。

コンテキストは `current->audit_context` に付加され、syscallごとではなく*タスク*ごとの
割り当てである — `audit_alloc()`（次節を参照）で一度だけ割り当てられ、
タスクが行うすべてのsyscallで再利用される。
`audit_syscall_entry()` は何も割り当てない。各syscallで、開始するsyscallのために
既存のコンテキストのフィールド（`major`、`argv`、`arch`、
`context = AUDIT_CTX_SYSCALL`、新しいタイムスタンプ）を埋め直す。`audit_syscall_exit()` も
それを解放しない — フィルタリングと、必要に応じたログ記録の後、`audit_reset_context()` を呼ぶ。
これは*蓄積された*syscallごとのデータ（names、補助レコード）を解放し、`current_state`
を基準値の `state` に戻して、タスクの次のsyscallに備えてコンテキストを準備する。
コンテキストの構造体自体が解放されるのはタスクの終了時だけであり、
`audit_free()`/`audit_free_context()` を介して行われる。

## 2つのフィルタ段階: audit_filter_task() とaudit_filter_syscall() {#two-filter-stages-audit_filter_task-and-audit_filter_syscall}

ルールのマッチングは1段階ではなく、2つの別々の段階で行われる。

**1. `audit_filter_task()`**（`kernel/auditsc.c`）は、タスクの作成時/exec時に
`audit_alloc()` から、タスクごとに一度だけ実行される — そのタスクのコンテキストが存在する*前に*である。

```c
/* kernel/auditsc.c */
static enum audit_state audit_filter_task(struct task_struct *tsk, char **key);
```

タスクレベルのルールがマッチすれば、その状態（`AUDIT_STATE_RECORD` または
`AUDIT_STATE_DISABLED`）がそのまま返される。何もマッチしなければ、
`audit_filter_task()` はデフォルトで `AUDIT_STATE_RECORD` の弱い親戚である
`AUDIT_STATE_BUILD` を返す — `AUDIT_STATE_DISABLED` **ではない**。`audit_alloc()` がコンテキストの割り当てを
完全にスキップするのは、状態が `AUDIT_STATE_DISABLED` で返ってきたときだけであり、
`AUDIT_STATE_BUILD` でもコンテキストは割り当てられる。実際には、システム全体で*何らかの*auditルールが
一度でもロードされれば、ほぼすべてのタスクが `BUILD` 状態の
コンテキストを得る — 本当の節約は、割り当てをスキップすることではなく、次の段階が高コストな作業を
先送りすることから生まれる。

**2. `audit_filter_syscall()`**（`kernel/auditsc.c`）は、syscallの*イグジット*時に、
`audit_filter_task()` がすでに割り当てたコンテキストに対して実行される。

```c
/* kernel/auditsc.c */
static void audit_filter_syscall(struct task_struct *tsk,
                                  struct audit_context *ctx);
```

ここでのルールは、次の組み合わせでマッチングされる。

- syscall番号（`-S openat`、`-S execve`）
- アーキテクチャ（`-F arch=b64`）
- UID / EUID / AUID（`-F uid=1000`、`-F auid!=4294967295`）
- PID（`-F pid=1234`）
- 実行ファイルのパス（`-F exe=/usr/bin/ssh`）
- ファイルパス（ウォッチルール、`-w /etc/passwd`）
- カスタムキー（`-k mykey`）

`audit_filter_syscall()` は `static void` である — 値を返すことも、
自分で `ctx->current_state` を設定することもない。共通のヘルパーである
`__audit_filter_op()` に処理を委ね、それがルールリストをたどって、マッチした場合は
戻る前に `ctx->current_state = state` を設定する。何もマッチしなければ、
`current_state`（段階1ですでに `BUILD`）はそのまま残され、このsyscallについてはレコードが
出力されない。アクション `always` のルールがマッチすれば、
`current_state` は `AUDIT_STATE_RECORD` になり、蓄積されたコンテキストが
ログに記録される。

これが高速パスである。`BUILD` 状態のコンテキストでは、`audit_filter_syscall()` がマッチを見つけなければ、
すべての記録作業は無駄に終わる — しかしその
記録作業は、完全なレコードを組み立てて出力するよりはるかに安価であり、段階2が実際に関門となっているのは
後者である。

## カーネル内のauditログ記録API {#the-in-kernel-audit-logging-api}

他のカーネルサブシステム — LSM、ファイルシステムのフック、デバイスドライバ — も、
同じ3関数のAPIを使ってauditレコードを出力する。

```c
/* kernel/audit.c */

/* 型 'type' のレコード用に新しいauditバッファを割り当てる */
struct audit_buffer *audit_log_start(struct audit_context *ctx,
                                      gfp_t gfp_mask, int type);

/* 書式付きテキストをバッファに追加する（printf形式） */
void audit_log_format(struct audit_buffer *ab, const char *fmt, ...);

/* バッファを確定し、auditd への配送のためにキューに入れる */
void audit_log_end(struct audit_buffer *ab);
```

`kernel/auditsc.c` からの例（簡略化）— `AUDIT_SYSCALL`
レコードそのものを構築する:

```c
ab = audit_log_start(context, GFP_KERNEL, AUDIT_SYSCALL);
if (ab) {
    audit_log_format(ab, "arch=%x syscall=%d", context->arch, context->major);
    if (context->return_valid != AUDITSC_INVALID)
        audit_log_format(ab, " success=%s exit=%ld",
                         str_yes_no(context->return_valid == AUDITSC_SUCCESS),
                         context->return_code);
    audit_log_format(ab, " a0=%lx a1=%lx a2=%lx a3=%lx items=%d",
                     context->argv[0], context->argv[1],
                     context->argv[2], context->argv[3],
                     context->name_count);
    audit_log_task_info(ab);   /* ppid/pid/auid/uid/gid/.../ses/comm/exe を追加する */
    audit_log_key(ab, context->filterkey);
    audit_log_end(ab);
}
```

`pid=`/`uid=`/`auid=`/`ses=`/`comm=`/`exe=` のフィールドは、
手書きのフォーマット呼び出しから来るのではない — `audit_log_task_info()`（`kernel/audit.c`）は
プロセスの身元情報ブロック全体を追加する共通ヘルパーであり、`AUDIT_SYSCALL` だけでなく、
「誰がこれを行ったか」を必要とするすべてのレコード型で再利用される。

## 補助レコード {#auxiliary-records}

1つのsyscallが複数のauditレコードを生成することはよくある。まず `AUDIT_SYSCALL`
レコードが出力され、その後1つ以上の補助レコードが、
同じイベントグループ（共通のシリアル番号で識別される）に追加される。

| レコード型 | 出力されるとき | 取得方法 |
|-------------|-------------|--------|
| `AUDIT_PATH` | syscall中に解決された各ファイルパスについて | `fs/namei.c` の呼び出し元での `audit_inode()` |
| `AUDIT_CWD` | カレントワーキングディレクトリ | （`kernel/auditsc.c` で直接構築される） |
| `AUDIT_SOCKADDR` | ネットワークsyscallが使ったソケットアドレス | `net/socket.c` の `audit_sockaddr()` |
| `AUDIT_IPC` | アクセスされたSysV IPCオブジェクト | IPC syscallのパス。例: `ipc/shm.c`/`ipc/sem.c`/`ipc/msg.c` |
| `AUDIT_MQ_*` | POSIXメッセージキューの操作 | `ipc/mqueue.c` の `audit_mq_*()` |

これらはいずれも、上の `audit_inode()` と同じパターンの2段階の受け渡しである。
syscallのパスは、（`include/linux/audit.h` で宣言された）薄い取得用ラッパーを呼んで、
関連するデータを現在の `audit_context` に格納しておく。しかし、レコードを構築して出力する
実際の `audit_log_start(..., AUDIT_SOCKADDR)` のような呼び出しは、後で集中的に、
syscallのイグジット時にコンテキストが確定されるときに `kernel/auditsc.c` で行われる —
取得箇所そのものではない。

1つのsyscallに関するすべてのレコードは、同じ `msg=audit(timestamp:serial)` フィールドを共有する。
`ausearch` と `aureport` は、出力を表示するときにシリアルでそれらをグループ化する。

## ユーザー空間のインターフェース {#userspace-interface}

### auditctlによるルールの追加 {#adding-rules-with-auditctl}

```bash
# uid 1000 によるすべての openat 呼び出しを監視する
auditctl -a always,exit -F arch=b64 -S openat -F uid=1000 -k file_access

# 機密ファイルへの書き込みと属性変更を監視する
auditctl -w /etc/passwd -p wa -k passwd_changes

# 特権関連のsyscallをシステム全体で監査する
auditctl -a always,exit -F arch=b64 \
    -S setuid -S setgid -S setresuid -S setresgid \
    -k priv_change

# 現在のルールを一覧表示する
auditctl -l

# 次の再起動までルールを変更不可にする（audit設定をロックする）
auditctl -e 2
```

### ausearchによるログの検索 {#searching-logs-with-ausearch}

```bash
# キーでタグ付けされたイベントを探す
ausearch -k passwd_changes

# 今日の0時以降のすべての openat 呼び出し
ausearch -sc openat --start today

# 特定の実行ファイルからのイベント
ausearch -x /usr/bin/sudo -i

# あるファイルに影響したイベント
ausearch -f /etc/shadow -i
```

### aureportによる集計レポート {#summary-reports-with-aureport}

```bash
# 全体の集計
aureport --summary

# ファイルアクセスの集計
aureport --file --summary

# 認証イベント
aureport -au

# 失敗したイベントのみ
aureport --failed
```

### 永続的なルール {#persistent-rules}

`/etc/audit/rules.d/*.rules` に書かれたルールは、ブート時に `augenrules` によって
読み込まれ、`auditctl -R` に渡される。

```
# /etc/audit/rules.d/50-syscall.rules
-a always,exit -F arch=b64 -S execve -k exec_tracking
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
```

`/etc/audit/auditd.conf` はデーモンを制御する。ログファイルのパス、ローテーションポリシー、
ディスク容量の制限、そしてディスクが満杯になったときの動作（`suspend`、`halt`、
`keep_logs`）である。

## 性能: 高速パスとaudit_dummy_context() {#performance-the-fast-path-and-audit_dummy_context}

auditは、監査対象でないタスクやsyscallについてはコストがほぼゼロになるよう
設計されている。

`audit_dummy_context()` は、現在のタスクのauditコンテキストが
NULLの場合、または `ctx->dummy` が非ゼロの場合にtrueを返す。`dummy` は `__audit_syscall_entry()` で
`context->dummy = !audit_n_rules` として設定される。`audit_n_rules` は、現在システム全体でロードされている
auditルールの数を表すグローバルなカウンタである。これは、上で扱ったタスクごと/syscallごとの
`current_state` によるフィルタリングより粗く、安価である。「*この*タスクやsyscallにマッチするルールがあるか」
ではなく、「今*どこかに*ルールが*1つでも*存在するか」だけを問う。「ダミー」のコンテキストとは、
（auditが有効になった後、syscallのたびにコンテキストの存在をチェックするオーバーヘッドを避けるために）
割り当てられてはいるものの、フィルタリングの対象となるものが何もロードされていないため、
そのタスクにとってより精密なフィルタリングから得るものがないコンテキストである。
「コンテキストが割り当てられていない」という意味では**ない**。本来なら高コストなパスや
inodeのレコードを構築するコードは、まず `audit_dummy_context()` をチェックし、その作業をスキップする。

```c
/* include/linux/audit.h */
static inline void audit_inode(struct filename *name,
                                const struct dentry *dentry,
                                unsigned int aflags)
{
    if (unlikely(!audit_dummy_context()))
        __audit_inode(name, dentry, aflags);
}
```

`audit_inode()` 自体はヘッダ内の薄い `static inline` の関門である — それが守っている
「高コストなレコードの組み立て」は `__audit_inode()`
（`kernel/auditsc.c`）にあり、パスを追加すべき実際のコンテキストがある場合にのみ
到達する。

`SYSCALL_WORK_SYSCALL_AUDIT` は、`audit_syscall_entry()` と
`audit_syscall_exit()` がそもそも何かを行うかどうかを決める関門である — しかし上で述べたように、auditが
システム全体で有効になると、このフラグはルールが特に選んだタスクだけでなく、デフォルトで
ほぼすべてのタスクに設定される（`AUDIT_STATE_BUILD`）。本当の
高速パスは `audit_dummy_context()` である。これは、`audit_inode()`/`audit_log_start()` のような
呼び出し箇所のすべてでチェックできるほど安価なので、`RECORD` に昇格することのない `BUILD` 状態の
コンテキストのコストは、チェックごとにポインタの読み取りと分岐1つで済み、
完全なレコードの組み立ては発生しない。

`AUDIT_BACKLOG_LIMIT`（`auditctl -b` で設定可能）は、`auditd` の消費が遅れている間に
カーネル内にバッファリングされるauditレコードの数を制限する。
バックログが満杯になると、`backlog_wait_time` の設定に応じて、新しいレコードが破棄される（
`audit: backlog limit exceeded` というカーネルメッセージとともに）か、カーネルが
待機する。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/audit.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/audit.h) — 公開されている `audit_syscall_entry()`/`audit_syscall_exit()`/`audit_inode()` のインラインラッパー（コンテキストが存在する場合にのみ実際の `__audit_syscall_entry()`/`__audit_syscall_exit()`/`__audit_inode()` を呼ぶ）と、`audit_dummy_context()`
- [kernel/auditsc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/auditsc.c) — `__audit_syscall_entry()`、`__audit_syscall_exit()`、`__audit_inode()`（インラインラッパーの背後にある実際の実装）、および `audit_alloc()`、`audit_filter_task()`、`audit_filter_syscall()`、`__audit_filter_op()`: 2段階のルールマッチングのパス
- [kernel/audit.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/audit.h) — 内部の `struct audit_context` と `struct audit_stamp` の定義
- [kernel/audit.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/audit.c) — `NETLINK_AUDIT` ソケット、`audit_log_start()`/`audit_log_format()`/`audit_log_end()`、`audit_backlog_limit` の処理
- [include/linux/entry-common.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/entry-common.h) — `syscall_enter_from_user_mode()` と `syscall_exit_to_user_mode()`。`SYSCALL_WORK_SYSCALL_AUDIT` フラグを関門として、`audit_syscall_entry()`/`audit_syscall_exit()` が実際に呼び出される場所

### manページ {#man-pages}

- [`auditctl(8)`](https://man7.org/linux/man-pages/man8/auditctl.8.html) — カーネルのauditルールをロード、一覧表示、削除するユーティリティ
- [`ausearch(8)`](https://man7.org/linux/man-pages/man8/ausearch.8.html) — キー、syscall、実行ファイル、ファイルによるauditログの検索
- [`aureport(8)`](https://man7.org/linux/man-pages/man8/aureport.8.html) — auditログの集計レポート
- [`auditd.conf(5)`](https://man7.org/linux/man-pages/man5/auditd.conf.5.html) — auditデーモンの設定: ログのパス、ローテーション、ディスク満杯時の動作

### 関連ページ {#related-pages}

- [syscallのエントリパス](syscall-entry.md) — auditのフックが呼ばれる `syscall_enter_from_user_mode()` と `syscall_exit_to_user_mode()`
- [Linux Auditサブシステム](../security/audit.md) — IMA、PAMとの統合、ファイルウォッチルールを含む、より広範なauditの解説
- [LSMフレームワーク](../security/lsm.md) — LSMフックのフレームワーク。そのSELinuxの小節では、`ausearch -m AVC` を使ったSELinux自身のAVC拒否auditレコードの読み方を扱う
- [seccomp BPF](../security/seccomp.md) — seccompによる拒否はauditログに現れる

### 外部資料 {#external}

- [docs.kernel.org: Core kernel API — audit functions](https://docs.kernel.org/core-api/kernel-api.html#c.audit_log_start) — `audit_log_start()`、`audit_log_format()`、`audit_log_end()`、`__audit_syscall_entry()`、`__audit_syscall_exit()` のkernel-docリファレンス
