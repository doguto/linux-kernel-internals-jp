# IPC事件簿 {#ipc-war-stories}

> リークしたセマフォ、fd枯渇、シグナルストーム、パイプ容量の意外な落とし穴

実際のIPCバグには共通のテーマがある。正常系は完璧に動くが、エラーパス、遅い消費者、あるいは何日も稼働して初めて表面化するエッジケースが、カーネルがIPCリソースをどう管理しているかについての根本的な誤解を露わにするのである。以下は、事後分析（ポストモーテム）に登場するたぐいの、実際に起きた5つのカテゴリの事例である。

---

## 1. SysVセマフォのリーク {#1-the-sysv-semaphore-leak}

### 何が起きたか {#what-happened}

あるC++アプリケーションは、プロセス間のロックにSysVセマフォを使っていた。各ワーカープロセスは起動時に`semget(IPC_CREAT, ...)`を呼び、正常なシャットダウン時に`semctl(IPC_RMID)`を呼んでいた。バックエンドのデータベースに到達できないときに通るエラーパスでは、コードが`IPC_RMID`を呼ぶ前に早期リターンしていた。

断続的な接続失敗を伴いながら48時間稼働した後、`ipcs -s`は128個のセマフォセットを表示した。このデプロイメントでは、防御的なリソース上限として`kernel.sem`の`SEMMNI`設定をカーネルデフォルトの32000から128に下げており、その上限に達していたのである。以後のワーカー起動では`semget()`が`ENOSPC`を返した。サービスは停止した。

```bash
# 診断: セットの数を数える
ipcs -s | wc -l          # 131（128 セット + ヘッダー行 3 行）

# 各セットの所有 UID と最終操作時刻
ipcs -s -c   # 作成者/所有者の UID と GID
ipcs -s -t   # 最後の semop() の時刻と最後のパーミッション変更の時刻
```

これ以上踏み込んで、孤立したセットをそれを作成した死んだプロセスに結び付ける方法はない。
`struct semid_ds`（`ipc/sem.c`、`include/uapi/linux/sem.h`）には、いかなる種類のPIDフィールドもない。
あるのは作成者/所有者のUIDとGID、それに上記の2つのタイムスタンプだけである。`ipcs -p`（「作成者と
最後の操作者のPIDを表示」）は、`cpid`/`lpid`を追跡している共有メモリとメッセージキューには
存在するが、セマフォセットに対しては何もしないことが文書化されており、何も出力しない。
実際には、どのセットが本当に放棄されたものかを特定するには、所有UIDと古い
`otime`/`ctime`を、自分のサービスについてすでに分かっていることと突き合わせるしかない。一度も記録されていないPIDを
カーネルに問い合わせることはできない。

### 根本原因 {#root-cause}

SysVセマフォセットは、`IPC_RMID`が呼ばれるかマシンが再起動されるまでカーネルメモリ内に存続する。どのファイルディスクリプタやプロセスの寿命とも関連付けられていない。カーネルはそれらを数値IDをキーとする`ipc_ids`構造体（`ipc/util.c`）で追跡しており、自動的なガベージコレクションは存在しない。

### 対処 {#fix}

互いに補完する2つの変更:

1. **即時対応**: リークした各semidについて`ipcrm -s`で手動で片付け、サービスを再起動する。

2. **恒久対応**: すべての`semop()`呼び出しで`SEM_UNDO`を使い、プロセスが終了した場合にカーネルが自動的に操作を取り消すようにする。そして`SIGTERM`/`SIGINT`ハンドラ*と*通常のエラーパスの両方で`IPC_RMID`を呼ぶ。

さらに良いのは、POSIX名前付きセマフォに移行することである。`sem_unlink()`はファイルシステム名前空間から名前を削除し、セマフォオブジェクト自体は最後の`sem_t`ハンドルが閉じられた時点で破棄される。永続的なカーネルオブジェクトはなく、リークもない。

```c
/* POSIX 名前付きセマフォ: 自動的なクリーンアップ */
sem_t *sem = sem_open("/myapp-lock", O_CREAT | O_EXCL, 0600, 1);
/* ... sem_wait() / sem_post() を使う ... */
sem_close(sem);
sem_unlink("/myapp-lock");  /* 名前は消え、最後の利用者が閉じた時点でオブジェクトが破棄される */
```

---

## 2. SCM_RIGHTSによるUnixソケットのfdリーク {#2-the-unix-socket-fd-leak-via-scm_rights}

### 何が起きたか {#what-happened_1}

ある特権分離デーモンは、特権を持つ親プロセスと特権を持たないワーカーに分かれていた。親は機密ファイルを開き、そのファイルディスクリプタをUnixドメインソケット（`SOCK_SEQPACKET`）越しに`SCM_RIGHTS`でワーカーに渡していた。ワーカーはリクエストを処理してfdを使い、使用後に`close()`することになっていた。

下流のサービスが予期しないステータスコードを返したときに通るエラー処理分岐のバグにより、ワーカーは`close()`呼び出しをスキップしていた。失敗したリクエストごとにfdが1つ開いたまま残った。数千件のリクエストの後、ワーカープロセスは`EMFILE`（errno 24、"Too many open files"）に達した。カーネルが受信したfdをワーカーのテーブルにインストールできなくなったため、新しいリクエストは`recvmsg()`の段階で拒否された。

```bash
# 診断
lsof -p <worker_pid> | wc -l        # 数千
lsof -p <worker_pid> | grep passwd  # /etc/passwd のコピーが大量に開かれている
```

### 根本原因 {#root-cause_1}

`SCM_RIGHTS`がファイルディスクリプタを配送するとき、カーネルは`receive_fd()`（`fs/file.c`内）を呼ぶ。これは受信者の`files_struct`に新しいファイルディスクリプタを予約してインストールする（`get_unused_fd_flags()`/`fd_install()`を介し、それらがさらに下位のスロットアロケータ`alloc_fd()`に到達する）。これはカーネルレベルでの完全な`dup()`であり、受信者のfdテーブルは、送信者とはまったく独立に、受信したfd1つにつき1つ増える。送信者が自分のコピーを閉じても、受信者のコピーには影響しない。

送信中（in-flight）の`SCM_RIGHTS`のfdは、`struct unix_sock`自体ではなく`user_struct`単位（`user->unix_inflight`）で追跡される。配送された後は、そのfdは受信者だけの責任となる。

### 対処 {#fix_1}

エラーパスを含むすべてのコードパスで、受け取ったfdを閉じる。パターン:

```c
int received_fd = -1;

/* ... recvmsg して received_fd を取り出す ... */

int ret = process_request(received_fd);
/* 成功時だけでなく「常に」閉じる */
if (received_fd >= 0) {
    close(received_fd);
    received_fd = -1;
}
if (ret < 0)
    return ret;
```

多層防御として: 受け取ったfdには`fcntl(received_fd, F_SETFD, FD_CLOEXEC)`で直ちに`O_CLOEXEC`を適用し、`exec()`で起動される子プロセスにリークしないようにする。また、`RLIMIT_NOFILE`を引き上げて早めに警告できる余裕を持たせ、`/proc/<pid>/fd`の数をアラートで監視する。

---

## 3. パイプ容量の意外な落とし穴 {#3-the-pipe-capacity-surprise}

### 何が起きたか {#what-happened_2}

あるログ集約パイプラインでは、生産者（構造化されたログ行を生成する）と消費者（圧縮してリモートのエンドポイントに転送する）が無名パイプでつながれていた。通常の負荷では消費者は追いついていた。リモート接続でのTLS再ネゴシエーションによって50msの処理スパイクが発生した間、消費者は停滞した。

その50msの間も、生産者は書き込みを続けた。パイプはデフォルトの65536バイトの上限まで埋まった（`/proc/sys/fs/pipe-max-size`のデフォルトは1 MiBだが、*パイプごとの*デフォルト容量は64 KiBであり、`pipe_max_size` sysctlと個々のパイプの現在の容量によって制御される）。生産者の`write()`呼び出しはブロックした。生産者スレッドは入ってくるログイベントの取り込みも担当していたため、取り込みパス全体が詰まった。消費者が回復した時点で、8秒分のログが失われていた。

```bash
# 現在のパイプ容量を確認（Linux 2.6.35 以降）
fcntl(fd, F_GETPIPE_SZ)  # 現在の容量をバイト単位で返す
cat /proc/sys/fs/pipe-max-size  # システム全体の上限
```

### 根本原因 {#root-cause_2}

パイプは固定容量の循環バッファである。`struct pipe_inode_info`（`include/linux/pipe_fs_i.h`で定義）は、`struct pipe_buffer`ページのリングを追跡する。すべてのページが埋まると、`fs/pipe.c`の`anon_pipe_write()`はブロックする（デフォルト）か、`EAGAIN`を返す（`O_NONBLOCK`の場合）。カーネルのパイプ容量を超える自動的なバッファリングは存在しない。

### 対処 {#fix_2}

侵襲性の低い順に3つの選択肢がある:

1. **パイプ容量を増やす**（応急処置）:
   ```c
   /* 1 MiB に増やす（/proc/sys/fs/pipe-max-size が上限） */
   fcntl(pipefd[1], F_SETPIPE_SZ, 1048576);
   ```
   これは時間を稼ぐが、ブロッキングをなくすわけではない。遅らせるだけである。

2. **ユーザー空間バッファを伴う非ブロッキング書き込み**: 書き込み端に`O_NONBLOCK`を設定し、`EAGAIN`をユーザー空間のリングバッファに溜めることで処理し、次の反復で再試行する。メモリを犠牲にして、生産者と消費者を切り離せる。

3. **`SO_SNDBUF`を備えたUnixソケットに切り替える**: 大きな`SO_SNDBUF`を持つ`SOCK_STREAM`のUnixソケットは、より調整しやすいバッファリングを提供する。さらに`O_NONBLOCK`と組み合わせれば、`EAGAIN`によるバックプレッシャーが明示的になり、生産者スレッドをブロックせずに回復できる。

---

## 4. SIGCHLDによるシグナルストーム {#4-the-signal-storm-from-sigchld}

### 何が起きたか {#what-happened_3}

あるジョブスケジューラは、短命なワーカープロセスを数千個生成していた。親は`SIGCHLD`ハンドラをインストールしていたが、そのハンドラは子を1つ回収するために`waitpid(-1, &status, WNOHANG)`を1回だけ呼んで戻るものだった。低負荷時にはこれで動いていた。子が1つ終了し、シグナルが1つ届き、1回の`waitpid()`呼び出しでそれを回収する。

高負荷時には、短い時間内に複数の子が終了した。Linuxの標準シグナル（`SIGCHLD`はシグナル17で、標準シグナルである）はキューイング*されない*。シグナルがすでに保留中であれば、追加の配送は破棄される。親は、実際には200個の子の終了に対して1つの`SIGCHLD`しか受け取らなかった。1回の`waitpid()`呼び出しはゾンビを1つ回収した。残りの199個はゾンビとしてプロセステーブルに残った。

数時間後、`ps aux | grep Z`は数千の`<defunct>`エントリを表示した。PID名前空間が枯渇した。新たな`fork()`呼び出しは`EAGAIN`を返した。

```bash
# ゾンビの子を数える
ps -o pid,stat,ppid | awk '$2 ~ /Z/ && $3 == <parent_pid> {count++} END {print count}'
```

### 根本原因 {#root-cause_3}

標準シグナル（1〜31）は、スレッドごとに1ビットの保留マスク（`struct task_struct`内の`pending`、型は`struct sigpending`）を使う。マスクはシグナル番号ごとに1ビットを持つ。別の子が終了したときに`SIGCHLD`がすでに保留中であれば、新しい配送は黙って破棄される。マスクのビットはすでに立っているからである。何個の子が終了したかにかかわらず、シグナルハンドラが見るのは1回の配送だけである。

リアルタイムシグナル（`SIGRTMIN`から`SIGRTMAX`まで。カーネル上の範囲は32〜64、glibcが予約する2つのシグナルを除くとユーザーから見える範囲は34〜64）はキュー（`struct sigqueue`）を使い、このような統合の対象にはならない。しかし`SIGCHLD`は標準シグナルである。

### 対処 {#fix_3}

`waitpid()`が0（待機可能な子がもうない）または-1/`ECHILD`（子がまったくいない）を返すまで、必ずループする:

```c
static void sigchld_handler(int sig)
{
    int saved_errno = errno;
    int status;
    pid_t pid;

    /* ループ: 1つのシグナルが複数の子の終了を表していることがある */
    while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
        /* 必要なら pid/status を記録 */
    }

    errno = saved_errno;  /* waitpid は errno を設定しうる。中断されたシステムコールのために復元する */
}
```

この`while`ループは省略できない。これを欠いた`SIGCHLD`のシグナルハンドラは、複数の子がいるどんな状況でも微妙に壊れている。あるいは、スレッドレベルで`SIGCHLD`をブロックしたうえで`signalfd`を使い、signalfdが読み出し可能になるたびに`waitpid()`をループさせてもよい。

---

## 5. eventfdのオーバーフロー {#5-the-eventfd-overflow}

### 何が起きたか {#what-happened_4}

ある高スループットのイベントパイプラインは、保留中の作業項目を消費者に通知するために`EFD_SEMAPHORE | EFD_NONBLOCK`付きの`eventfd`を使っていた。生産者はタイトなループで動き、作業項目ごとに`write(efd, &one, 8)`を呼んでいた。消費者はeventfdから読み出し、項目を1つずつ処理していた。

トラフィックのスパイク時、生産者は消費者が処理できるよりも速く項目を生成した。eventfdのカウンタは`UINT64_MAX - 1`に向かって増えていった。カウンタがその閾値に達すると、以後の`write()`呼び出しは`EAGAIN`を返した。それ以上インクリメントすると`uint64_t`のカウンタがオーバーフローするため、カーネルが拒否したのである（`fs/eventfd.c`の`eventfd_write()`を参照）。

生産者のコードは`write()`の戻り値をきちんとチェックしていなかった:

```c
/* バグ: EAGAIN が黙って捨てられる */
write(efd, &(uint64_t){1}, 8);
```

作業項目は生産者の内部キューに入れられたが、消費者にはその存在が一切通知されなかった。スパイクが収まって消費者が追いつくと、eventfdのカウンタは下がった。しかし取り残された項目は、次の生産者の書き込みが新たな通知を引き起こすまで、処理されないままキューに残った。それらの項目には際限のないレイテンシが生じた。

### 根本原因 {#root-cause_4}

`fs/eventfd.c`の`eventfd_write()`は次のようにチェックする:

```c
if (ULLONG_MAX - ctx->count <= ucnt) {
    /* UINT64_MAX - 1 を超えてしまう */
    if (file->f_flags & O_NONBLOCK)
        return -EAGAIN;
    /* それ以外: カウンタが下がるまでブロック */
}
```

カウンタの上限は`UINT64_MAX - 1`である（`UINT64_MAX`ではない。`UINT64_MAX`は内部で番兵値として使われるため）。カウンタはその上限で飽和するのではなく、ブロックするか失敗する。`EFD_NONBLOCK`の場合、書き込みはブロックせずに黙って`EAGAIN`を返す。呼び出し元が`EAGAIN`をバックプレッシャーとして扱わなければ、そのイベントは失われる。

### 対処 {#fix_4}

非ブロッキングのeventfd書き込みのたびに、`EAGAIN`を明示的に処理する:

```c
uint64_t one = 1;
ssize_t n = write(efd, &one, sizeof(one));
if (n < 0) {
    if (errno == EAGAIN) {
        /* カウンタが上限に達している: 消費者が遅れている。 */
        /* バックプレッシャーをかける: 速度を落とす、ログに残す、または別の方法で消費者を起こす。 */
        apply_backpressure();
    } else {
        perror("eventfd write");
    }
}
```

あるいは、バックプレッシャーが特別なエラーコードではなくプロトコルの一部になっているIPC機構に切り替える:

- **POSIXメッセージキュー**（`mq_open`）: キューが満杯（`mq_maxmsg`個）のとき、`mq_send()`はブロックする（または`EAGAIN`を返す）。キューの深さは明示的で、設定可能である。
- **Unixソケット**（`AF_UNIX SOCK_DGRAM`）: 送信バッファが満杯のとき、非ブロッキングの`sendmsg()`は`EAGAIN`/`EWOULDBLOCK`を返す（ブロッキングの場合は代わりに待つ）。メッセージごとに明示的なバックプレッシャーがかかる。（`send(2)`によれば、`ENOBUFS`は*ネットワークインターフェース*の出力キューについて文書化されたものであり、ここには当てはまらない。`net/unix/af_unix.c`のデータグラム送信パスは、他のすべてのソケットタイプと同じ汎用のソケットバッファ待機、つまり`net/core/sock.c`の`sock_alloc_send_pskb()`を通り、バッファが満杯なら`-EAGAIN`を返す。）
- **ブロッキングのeventfd**（`EFD_NONBLOCK`なし）: カウンタが最大値のとき生産者はブロックし、スレッドがブロックされる代償と引き換えに自然なフロー制御が得られる。

一般原則: `EFD_NONBLOCK`が適切なのは、`EAGAIN`を稀なエラーとしてではなく、第一級の状態として明示的に扱う場合だけである。

---

## 5つすべてに共通するパターン {#patterns-across-all-five}

| 事例 | 根本的な仕組み | 検出 | 予防 |
|----------|---------------|-----------|------------|
| SysVセマフォのリーク | 永続的なカーネルオブジェクト | `ipcs -s`、`ENOSPC` | すべてのパスで`IPC_RMID`。POSIXセマフォを優先 |
| SCM_RIGHTSのfdリーク | fdは受信者が所有する | `lsof`、`EMFILE` | すべてのパスで閉じる。`O_CLOEXEC` |
| パイプ容量 | 固定長のカーネルバッファ | 生産者がブロックする | `F_SETPIPE_SZ`。非ブロッキング + ユーザー空間バッファ |
| SIGCHLDの統合 | 標準シグナルのビットマスク | ゾンビの蓄積 | ハンドラ内で`waitpid()`をループ |
| eventfdのオーバーフロー | `UINT64_MAX - 1`の上限 | 黙って返る`EAGAIN` | 戻り値をチェックし、`EAGAIN`をバックプレッシャーとして扱う |

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [ipc/util.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/ipc/util.c) — `ipc_addid()`、`ipc_rmid()`、およびSysVオブジェクトが明示的に削除されるまで存在し続ける名前空間ごとの`ipc_ids`テーブル（事例1の背景）
- [ipc/sem.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/ipc/sem.c) — `semget()`、`semctl()`、および終了時に調整を行う`SEM_UNDO`の処理（事例1の対処の背景）
- [include/uapi/linux/sem.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/sem.h) — PIDフィールドをまったく持たない`struct semid_ds`（事例1の診断の限界の背景）
- [fs/file.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/file.c) — 受信した`SCM_RIGHTS`ディスクリプタを受信者自身の`files_struct`に予約・インストールする`receive_fd()`（事例2の背景）
- [net/core/scm.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/core/scm.c) — `scm_detach_fds()`と`scm_recv_one_fd()`: 各`SCM_RIGHTS`ディスクリプタに対して`receive_fd()`を呼ぶ制御メッセージのパス（事例2の背景）
- [net/unix/garbage.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/unix/garbage.c) — ユーザーごとの`unix_inflight`カウンタと、送信中の`SCM_RIGHTS` fdに対する循環検出ガベージコレクタ（事例2の背景）
- [include/linux/pipe_fs_i.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/pipe_fs_i.h) — `struct pipe_inode_info`と`struct pipe_buffer`: 事例3の背景にある固定サイズのリング
- [fs/pipe.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/pipe.c) — `anon_pipe_write()`、`pipe_max_size` sysctl、`F_SETPIPE_SZ`の処理（事例3の背景）
- [kernel/signal.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/signal.c) — 対象ですでに保留中の標準シグナルを破棄する`legacy_queue()`と`__send_signal_locked()`（事例4の背景）
- [include/linux/signal_types.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/signal_types.h) — `struct sigpending`（ビットマスク）と`struct sigqueue`（キューイングされるリストエントリ）: 事例4における標準/リアルタイムの違いの背景にあるデータ構造
- [fs/eventfd.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/eventfd.c) — `eventfd_write()`と、ブロック/EAGAINとなる`ULLONG_MAX - 1`の上限（事例5の背景）
- [net/core/sock.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/core/sock.c) — `sock_alloc_send_pskb()`: すべてのソケットタイプ（`AF_UNIX`を含む）が通る、送信バッファ満杯時の汎用の待機/`-EAGAIN`パス（事例5のUnixソケットによる代替手段の背景）

### manページ {#man-pages}

- [`semop(2)`](https://man7.org/linux/man-pages/man2/semop.2.html) — `SEM_UNDO`について記述している。プロセス終了時に調整が自動的に取り消される（事例1で使用）
- [`sem_unlink(3)`](https://man7.org/linux/man-pages/man3/sem_unlink.3.html) — 名前付きPOSIXセマフォは、それを開いているすべてのプロセスが閉じた時点で初めて破棄されることを確認できる。事例1の「POSIXセマフォへの移行」という対処の背景にある仕組み
- [`ipcs(1)`](https://man7.org/linux/man-pages/man1/ipcs.1.html) — 事例1で使った診断ツール
- [`unix(7)`](https://man7.org/linux/man-pages/man7/unix.7.html) — オープン済みファイルディスクリプタをプロセス間で渡すための`SCM_RIGHTS`補助データについて記述している（事例2で使用）
- [`pipe(7)`](https://man7.org/linux/man-pages/man7/pipe.7.html) — パイプの容量（Linux 2.6.11以降はデフォルトで16ページ / 65,536バイト）とブロッキング書き込みの挙動について記述している（事例3で使用）
- [`fcntl(2)`](https://man7.org/linux/man-pages/man2/fcntl.2.html) — パイプのサイズ変更のための`F_SETPIPE_SZ`/`F_GETPIPE_SZ`について記述している（事例3の対処で使用）
- [`signal(7)`](https://man7.org/linux/man-pages/man7/signal.7.html) — 標準シグナルはキューイングされず（保留中の複数のインスタンスは1つにまとめられる）、リアルタイムシグナルはキューイングされることを記述している。事例4の背景にある仕組み
- [`wait(2)`](https://man7.org/linux/man-pages/man2/waitpid.2.html) — `waitpid()`と`WNOHANG`について記述している（事例4の対処で使用）
- [`eventfd(2)`](https://man7.org/linux/man-pages/man2/eventfd.2.html) — `EFD_SEMAPHORE`、`EFD_NONBLOCK`、およびブロック/EAGAINとなるカウンタの`UINT64_MAX - 1`の上限について記述している（事例5で使用）
- [`mq_send(3)`](https://man7.org/linux/man-pages/man3/mq_send.3.html) — `mq_maxmsg`に達したときのブロック/`EAGAIN`の挙動について記述している。事例5の対処で触れた代替機構

### 関連ページ {#related-pages}

- [シグナル](signals.md) — `SIGCHLD`、`sigaction`、`sigprocmask`
- [パイプとFIFO](pipes.md) — `struct pipe_inode_info`、`PIPE_BUF`、`F_SETPIPE_SZ`
- [Unixドメインソケット](unix-sockets.md) — `SCM_RIGHTS`、`SCM_CREDENTIALS`、fd受け渡しの内部構造
- [SysV IPC: セマフォとメッセージキュー](sysv-semaphores.md) — `SEM_UNDO`、`IPC_RMID`、`SEMMNI`
- [eventfdとsignalfd](eventfd-signalfd.md) — `EFD_SEMAPHORE`、`EFD_NONBLOCK`、カウンタの意味論
- [POSIXメッセージキュー](mqueue.md) — `mq_send`、`mq_maxmsg`、ブロッキング/バックプレッシャーの意味論。事例5で論じた代替手段

### LWN記事 {#lwn-articles}

- [Circular pipes](https://lwn.net/Articles/118750/)（2005年1月11日）— 2.6.11における複数ページの循環パイプバッファの導入。事例3の背景にある`struct pipe_inode_info`リングの祖先
- [The evolution of pipe buffers](https://lwn.net/Articles/119682/)（2005年1月18日）— パイプバッファの再設計の続報。事例3のデフォルトのパイプサイズ64 KiBの背景にある16ページという容量上限を含む
- [io_uring, SCM_RIGHTS, and reference-count cycles](https://lwn.net/Articles/779472/)（2019年2月13日）— カーネルが送信中の`SCM_RIGHTS`ファイルディスクリプタをどのように追跡し、ガベージコレクションするか。事例2の背景にある仕組み
