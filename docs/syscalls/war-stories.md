# syscallの事件簿 {#syscall-war-stories}

> syscallインターフェースにまつわる実際のバグ、ABI破壊、教訓

これらの事例は、実際のLinuxカーネル開発の歴史から取ったものである。いずれも、本番環境で遭遇するまでは
明らかにならない制約や危険を示している。

## time_tのオーバーフロー（Y2038） {#the-time_t-overflow-y2038}

### 背景 {#background}

元の `stat()` syscall（および `time()`、`select()`、`utime()` など多数）は、タイムスタンプを
`time_t` として格納していた。これは32ビットアーキテクチャでは符号付き32ビット整数である。32ビットの `time_t` は
**2038年1月19日 03:14:07 UTC** にオーバーフローする。その瞬間、値は大きな負の数に折り返し、
1901年の時刻を表すことになる。

### カーネルが行ったこと {#what-the-kernel-did}

`stat64()` と `lstat64()` の追加により、32ビットアーキテクチャでも64ビットのファイルサイズを扱えるようになったが、32ビットLinuxの
`struct stat64` の時刻フィールドは依然として32ビットの `time_t` を使っていた。本当の修正には、
全体を通して真に64ビットのタイムスタンプを持つ新しいsyscallが必要だった。

Linux 4.11（2017年）は `statx()`（`fs/stat.c`、コミット [`a528d35e8bfc`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a528d35e8bfcc521d7cb70aaf03e1bd296c8493f)）を追加した。これは
64ビットの `tv_sec` を持つ `struct statx_timestamp` を使う。

```c
/* include/uapi/linux/stat.h */
struct statx_timestamp {
    __s64   tv_sec;    /* 常に64ビット */
    __u32   tv_nsec;
    __s32   __reserved;
};

struct statx {
    /* ... */
    struct statx_timestamp  stx_atime;
    struct statx_timestamp  stx_btime;
    struct statx_timestamp  stx_ctime;
    struct statx_timestamp  stx_mtime;
    /* ... */
};
```

しかし `statx()` だけでは不十分だった。`clock_gettime()`、`select()`、`ppoll()`、`nanosleep()`、`pselect6()` を呼ぶ
32ビットアーキテクチャ（ARM、MIPS、`CONFIG_IA32_EMULATION` 付きのx86）はどれも、依然として32ビットの
`struct timespec` フィールドを渡しており、これは2038年にオーバーフローする。

Linux 5.1は、32ビットアーキテクチャ向けに一連の `_time64` 版を追加した。
`clock_gettime64`、`clock_settime64`、`clock_adjtime64`、`clock_getres_time64`、`pselect6_time64`、
`ppoll_time64`、`io_pgetevents_time64`、`utimensat_time64` などである — 時刻の型を直すためだけに
20を超える新しいsyscallが追加された。

### 教訓 {#the-lesson}

1970年代に下されたABIの判断（当時のハードウェアに合わせて `time_t` のサイズが決められた）は、50年にわたって
引き継がれた。修正には、glibc、musl、カーネル、そして32ビットLinuxをターゲットとするすべてのツールチェーンにまたがる
協調的な取り組みが必要だった。syscall ABIは事実上永続的である — 出荷済みのカーネルに入った構造体フィールドの幅を
変更する仕組みは存在しない。新しいsyscallが唯一の手段である。

今日、32ビットのタイムスタンプや32ビットのオフセットを持つ新しいsyscallを定義すれば、それは次の
Y2038を書いていることになる。

---

## futex ABIのハッシュ衝突 {#the-futex-abi-hash-collision}

### 背景 {#background_1}

`futex()` は高速なユーザー空間mutexとしてLinux 2.5.7で追加された。カーネルはfutexのアドレスを
ハッシュキーとして使い、待機者を見つける。`futex_wait(addr, val)` は現在のタスクを `addr` をキーとする待ちキューに追加する。
`futex_wake(addr, n)` は `addr` の待機者を見つけて起こす。

### バグ {#the-bug}

初期の実装は**仮想**アドレスをキーにしていた。メモリマップ領域を共有する2つのプロセス
（`mmap(MAP_SHARED)` や共有匿名マッピングによる）は、同じ物理メモリ位置に対して、異なる仮想アドレスで
`futex_wait()` を呼ぶことがありえた。ハッシュの検索には仮想アドレスが使われたため、
待機者と起床者は別々の待ちキューに入り — 起床は何の効果も持たず、待機者は永遠に
スリープした。

### 修正 {#the-fix}

修正は、共有futexのキーを仮想アドレスにまったく依存させず、基盤となるオブジェクトの同一性 — inodeのシーケンス番号と
ページオフセットの組み合わせ — にすることだった。カーネルは現在、
`union futex_key` を計算する。

```c
/* include/linux/futex.h */
union futex_key {
    struct {
        u64           i_seq;   /* inodeのシーケンス番号（ファイルバックの場合） */
        unsigned long pgoff;
        unsigned int  offset;
    } shared;
    struct {
        union {
            struct mm_struct *mm;
            u64 __tmp;
        };
        unsigned long address;
        unsigned int  offset;
    } private;
    struct {
        u64           ptr;
        unsigned long word;
        unsigned int  offset;
    } both;
};
```

共有マッピングでは、`get_futex_key()` はバックするinodeのシーケンス番号と、そのファイル内での
ページのオフセットからキーを導出する — これは、どのプロセスの仮想アドレス空間がそのページをマップしていても
同じになる同一性である。プライベートマッピング（匿名、`MAP_PRIVATE`）は、仮想アドレスと
`mm_struct` ポインタをキーにする。これでプロセス内のマッピングが一意に識別される。

### 教訓 {#the-lesson_1}

syscall ABIは正しく見えた。`futex(addr, FUTEX_WAIT, val, timeout)` は妥当なインターフェースである。
バグは実装のセマンティクスにあった — カーネルの「同じfutex」という概念が、ユーザーの
「共有メモリ内の同じ位置」という概念と一致していなかったのである。修正が純粋に内部的なものだったため、
syscall番号を変えずに直すことができた。しかし誤った挙動が実装ではなく仕様にあったなら、
新しいsyscallが必要になっていただろう。

共有メモリIPCのセマンティクス — 特に、プロセス境界をまたいだアドレスの等価性やオブジェクトの同一性に関わるもの —
は、仮想アドレスではなく、物理的な同一性やファイルの同一性によって定義されなければならない。

---

## cloneとforkにおけるCLONE_FILESのfd共有 {#the-clone-vs-fork-clone_files-fd-sharing}

### 背景 {#background_2}

`clone(CLONE_FILES)` は、親のファイルディスクリプタテーブルを共有する新しいタスクを作成する。両方のタスクが
同じ `struct files_struct` を参照するため、一方での `open()` は、もう一方からも直ちに `fd` として見える。これが
`pthread_create()` の動作の仕組みである。スレッドはファイルディスクリプタを共有する。

### 危険 {#the-hazard}

あるプログラムは `clone(CLONE_FILES | CLONE_VM | CLONE_THREAD | SIGCHLD)` でスレッドを作成し、
次のパターンを使っていた。

```c
/* スレッド: ファイルを開き、パイプ経由で親に fd を返す */
int fd = open("/some/file", O_RDONLY);
write(notify_pipe[1], &fd, sizeof(fd));

/* 親: fd を読み取り、使い終わったら閉じる */
int fd;
read(notify_pipe[0], &fd, sizeof(fd));
/* ... fd を使う ... */
close(fd);
```

これは確実に動作していた。その後、`O_CLOEXEC` が広く使われるようになった。スレッドは安全のために
`O_CLOEXEC` 付きでファイルを開くようになった。問題は、`O_CLOEXEC` がファイル記述（`struct file`）単位で設定され、
ディスクリプタ単位ではないことだった。親プロセスが — まったく別の理由で、別の
スレッドで — `execve()` を呼ぶと、カーネルは共有テーブル内の `O_CLOEXEC` ファイルディスクリプタをすべて閉じた。
親が使おうとしていたfdは、知らないうちに閉じられていた。

さらに微妙な問題がある。`CLONE_FILES` を共有するスレッド間での `dup2(oldfd, newfd)` はアトミックではない。
カーネルは操作中に `files->file_lock` を保持しなければならないが、別のスレッドが有効性チェックと
インストールの間に `close(newfd)` を呼ぶと、挙動は未定義になる。`POSIX_SPAWN_FILE_ACTIONS_ADDDUP2`
インターフェースは、より高いレベルで直列化することでこれを回避するよう特別に設計された。

### 教訓 {#the-lesson_2}

`CLONE_FILES` スレッドは、すべてのfd操作を共有メモリ操作として扱わなければならない。どのスレッドでのclose、dup、
`execve` も、直ちにすべてのスレッドに影響する。後で `execve` を行うのに、`CLONE_FILES` の境界をまたいで共有するつもりの
fdに `O_CLOEXEC` を使うのは論理的な誤りである。fdのライフサイクルをスレッド間で管理する場合の正しいパターンは、
fdテーブルを共有しないこと（`CLONE_THREAD` なしの `CLONE_FILES`）である。

---

## EINTRの不意打ち {#the-eintr-surprise}

### 背景 {#background_3}

`poll()`、`select()`、`read()` をはじめ、ほとんどのブロッキングsyscallは、シグナルによって中断されると
`errno == EINTR` で `-1` を返す。これは文書化されており、よく知られているが、それでも本番環境のバグを頻繁に引き起こしている。

### インシデント {#the-incident}

長時間稼働するデーモンが、数分のタイムアウト付きの `poll()` でネットワークイベントを待っていた。デーモンは
グレースフルシャットダウンを開始するために `SIGTERM` ハンドラをインストールしていた。

```c
void sigterm_handler(int sig)
{
    shutdown_requested = 1;
}

int main(void)
{
    struct sigaction sa = { .sa_handler = sigterm_handler };
    sigaction(SIGTERM, &sa, NULL);

    while (!shutdown_requested) {
        int ret = poll(fds, nfds, TIMEOUT_MS);
        if (ret < 0) {
            perror("poll");  /* "Interrupted system call" をログに出して終了する */
            exit(1);
        }
        /* イベントを処理する */
    }
}
```

`poll()` 中に `SIGTERM` が届くと、ハンドラが実行され、`shutdown_requested = 1` を設定して戻った。
その後 `poll()` は `errno == EINTR` で `-1` を返した。デーモンは `poll()` からの負の戻り値をすべて
致命的とみなして `exit(1)` を呼んだ — 入念に実装したグレースフルシャットダウンを迂回してしまったのである。

### 修正と微妙な点 {#the-fix-and-the-subtlety}

明白な修正は、`EINTR` をチェックしてリトライすることである。

```c
do {
    ret = poll(fds, nfds, TIMEOUT_MS);
} while (ret < 0 && errno == EINTR && !shutdown_requested);
```

それほど明白でない修正は `SA_RESTART` である。

```c
struct sigaction sa = {
    .sa_handler = sigterm_handler,
    .sa_flags   = SA_RESTART,
};
```

`SA_RESTART` があると、カーネルは一部の低速なsyscallを、シグナルハンドラが戻った後に `EINTR` を返さずに
自動的に再開する。`poll()` との相互作用は、見かけよりも微妙である。

Linux 2.6.24以降、`poll()` はシグナルで中断されると（`ERESTARTSYS` ではなく）`ERESTART_RESTARTBLOCK`
を返す。`ERESTART_RESTARTBLOCK` の仕組みにより、カーネルは経過時間を反映してタイムアウトを調整し、
`restart_syscall()` を介してsyscallを再開する。これはユーザー空間からは透過的である —
`poll()` はプロセスに `EINTR` を返さず、自動的に再開する。`SA_RESTART` はこのパスを制御しない
（制御するのは `ERESTARTSYS` だけ）が、最終的な効果は同じで、`poll()` は再開される。

本当に常に `EINTR` を返すsyscallは、`ERESTARTNOHAND` を使うもの（`epoll_wait()` など）である
— これらは `SA_RESTART` があっても、シグナルハンドラが実行されれば `EINTR` を返す。上のインシデントでは、
EINTRの問題は現代の `poll()` ではなく、`epoll_wait()` か2.6.24より前のカーネルの `poll()` で生じたはずである。

`SA_RESTART` によって再開**される**syscall: `read()`、`write()`、`wait()`、`ioctl()`（デバイス）、
`flock()`。

`ERESTART_RESTARTBLOCK` を使う（`SA_RESTART` の制御を受けず、`restart_syscall()` を介して再開する）syscall:
`poll()`、`select()`、`nanosleep()`。

`SA_RESTART` があっても再開**されない**syscall: `epoll_wait()` など `ERESTARTNOHAND` を使うもの。
これらはシグナルハンドラが実行されると常に `EINTR` を返す。

デーモンにとって正しい本番向けのパターンは、`signalfd()`、あるいはシグナルマスク付きの `pselect()`/`ppoll()` を使うことである。
`ppoll()` と `pselect6()` はシグナルマスクのインストールと待機をアトミックに行い、シグナルのブロック解除から
待機に入るまでの間の競合ウィンドウを塞ぐ。

---

## 新アーキテクチャでのsyscall番号の衝突 {#the-syscall-number-collision-on-a-new-architecture}

### 背景 {#background_4}

RISC-Vは、バージョン4.15（2018年）で正式にLinuxカーネルにマージされた。RISC-Vは、手作業で保守されるアーキテクチャ固有のテーブルではなく、
`include/uapi/asm-generic/unistd.h` の汎用syscallテーブルを使う。
syscall番号は、新しいsyscallがアップストリームにマージされるたびに順番に割り当てられる。

### インシデント {#the-incident_1}

ある半導体ベンダーが、独自のハードウェアアクセラレータのファームウェアを制御するためのカスタムインターフェースを必要とする
RISC-V SoC向けに、ダウンストリームカーネルを出荷した。`ioctl()` や `netlink` のインターフェースを使う代わりに、
あるエンジニアがダウンストリームのツリーに新しいツリー外syscallを追加した。

```c
/* ベンダー/ダウンストリームカーネル: kernel/vendor_accel.c */
/* 451番に追加 — フォーク時点で次に空いていた番号 */
SYSCALL_DEFINE3(vendor_accel_submit, ...)
```

ダウンストリームのユーザー空間ツールとアクセラレータSDKは `#define __NR_vendor_accel_submit 451` をハードコードしており、
ファームウェアイメージに入れてエンドユーザーに出荷された。

Linux 5.16（2022年1月にアップストリームでリリース）はsyscall番号449に `futex_waitv()` を追加し、その後の
リリースで450と451がそれぞれ `set_mempolicy_home_node`（5.17）と `cachestat`（6.5）に割り当てられた。
ベンダーのユーザー空間ツールをアップストリームやディストリビューションのカーネルで動かそうとした顧客は、
誤った挙動に遭遇した。syscall 451を呼ぶと、まったく別のカーネル関数が呼び出されたのである。

ベンダーのダウンストリームは `futex_waitv` がマージされる前に分岐していたため、その番号付けは
アップストリームからずれていた。この衝突は、顧客がベンダー提供のユーザー空間とアップストリームのカーネルを
混在させ始めるまで検出できなかった。

### 教訓 {#the-lesson_3}

本番環境でのツリー外syscall番号は時限爆弾である。カーネルのsyscall番号空間は、アップストリームの
`MAINTAINERS` プロセスが管理するグローバルな名前空間である。後でマージするつもりでダウンストリームのツリーで番号を割り当てると、
カーネルのリリースサイクルを重ねるごとに大きくなる衝突リスクを生む。

ベンダー固有のインターフェースの正しい道筋:
1. キャラクタデバイスに対する `ioctl()` を使う — ioctlのコマンド番号はデバイスごとであり、グローバルには衝突しない。
2. `netlink`（ネットワークとシステム設定の場合）や `sysfs`（デバイス属性の場合）を使う。
3. 本当に新しいsyscallが必要なら、アップストリームに提出し、番号が割り当てられるのを待ってから、
   その番号をハードコードしたユーザー空間を出荷する。

`seccomp-notify` の仕組み（[ptraceとsyscallのインターセプト](ptrace-interception.md)を参照）も、
カーネルを一切変更せずに、特権を持つスーパーバイザ内でsyscall風のカスタムインターフェースを実装するための
選択肢となる。

---

## 参考文献 {#further-reading}

- [SYSCALL_DEFINEとディスパッチ](syscall-define.md) — ABIの安定性の保証と構造体ベースのsyscall
- [32ビット互換syscall](compat.md) — Y2038の修正が回避しなければならなかった32ビットABIの仕組み（`compat_stat` の32ビット時刻フィールドなど）
- [syscallの再開メカニズム](restart-block.md) — ERESTARTSYS、ERESTART_RESTARTBLOCK、restart_syscall() の全容。上のEINTRの話の背景にある仕組み
- [シグナル](../ipc/signals.md) — シグナル配送、`sigaction`、そして上のEINTRの話の背景にある `SA_RESTART` のセマンティクス
- [futexの内部](../locking/futex.md) — futex_wait()/futex_wake() とfutexキーのハッシュが現在どう動作するか
- [ptraceとsyscallのインターセプト](ptrace-interception.md) — カスタムsyscallの代替としてのseccomp-notify
- [新しいsyscallの追加](adding-syscall.md) — 上の落とし穴を避けてsyscallを追加する正しい方法
- [kernel/futex/core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/futex/core.c) — `get_futex_key()`: プライベートfutexでは `mm_struct` ポインタとアドレスを、共有futexではinodeのシーケンス番号とページオフセットをキーにする — 物理アドレスではない

## 外部参考資料 {#external-references}

- [git.kernel.org: a528d35e8bfc](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a528d35e8bfcc521d7cb70aaf03e1bd296c8493f) — "statx: Add a system call to make enhanced file info available"、David Howells、Linux 4.11向けにマージ（fs/stat.c）
- [git.kernel.org: 3075d9da0b4c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3075d9da0b4ccc88959db30de80ebd11d2dde175) — "Use ERESTART_RESTARTBLOCK if poll() is interrupted by a signal"。EINTRの話の中心にある `poll()` についての主張の根拠となる fs/select.c の変更（Linux 2.6.24向けにマージ）
- [カーネルドキュメント: ktime accessors](https://docs.kernel.org/core-api/timekeeping.html) — "Deprecated time interfaces" の節: "all interfaces returning a 'struct timeval' or 'struct timespec' have been replaced because the tv_sec member overflows in year 2038 on 32-bit architectures"（訳: 'struct timeval' や 'struct timespec' を返すインターフェースはすべて置き換えられた。32ビットアーキテクチャでは tv_sec メンバーが2038年にオーバーフローするためである）
- [カーネルドキュメント: Adding a New System Call](https://docs.kernel.org/process/adding-syscalls.html) — syscallの番号付けについてのアップストリームのガイダンス（"these numbers are liable to be changed if there are conflicts in the relevant merge window"）と、新しいsyscallよりもioctl/sysfsなどのインターフェースを優先すべきことについて
