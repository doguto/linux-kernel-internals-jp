# syzkaller: カーネルファジング {#syzkaller-kernel-fuzzing}

> カーネルのバグを見つけるための自動システムコールファジング

## syzkallerとは {#what-is-syzkaller}

syzkallerはGoogleが開発した自動カーネルファザーである。ランダムな**システムコールの列**を生成してVM内で実行し、クラッシュ、ハング、（KASAN/KCSANによる）メモリエラー、その他の異常を監視する。

2015年以来、syzkallerは[syzbot](https://syzkaller.appspot.com)で報告された数千件のカーネルバグを発見してきた。

## アーキテクチャ {#architecture}

```
syz-manager (host)
├── Manages VM pool (QEMU/GCE/VMware)
├── Generates corpus of interesting syscall sequences
├── Mutates programs: add/remove/modify syscall arguments
├── Tracks coverage via kcov (kernel coverage instrumentation)
└── Reports crashes via email/dashboard

syz-fuzzer (in VM)
├── Executes programs (syz-executor subprocess)
├── Measures kcov coverage
└── Reports new coverage / crashes to manager

syz-executor (in VM)
├── Runs as root
├── Executes actual syscalls (one program at a time)
└── Monitors for crashes (KASAN, UBSAN, KCSAN, general panics)
```

## kcov: カーネルカバレッジ計装 {#kcov-kernel-coverage-instrumentation}

syzkallerは**kcov**を使って、ファジングを新しいコードパスへと誘導する:

```c
/* CONFIG_KCOV=y: すべての基本ブロックを計装する */
/* 各カーネル関数は次のように計装される: */
void my_function(void)
{
    __sanitizer_cov_trace_pc();   /* kcovが挿入したもの */
    /* ... カーネルのコード ... */
    if (condition) {
        __sanitizer_cov_trace_pc();   /* 別のエッジ */
        /* ... */
    }
}

/* ユーザー空間（syz-executor）は次のようにしてカバレッジを読む: */
int kcov_fd = open("/sys/kernel/debug/kcov", O_RDWR);
ioctl(kcov_fd, KCOV_INIT_TRACE, COVER_SIZE);
void *cover = mmap(NULL, COVER_SIZE * sizeof(unsigned long), ...);
ioctl(kcov_fd, KCOV_ENABLE, KCOV_TRACE_PC);

/* システムコールを実行する */
syscall(...);

ioctl(kcov_fd, KCOV_DISABLE, 0);
unsigned long n_pcs = cover[0];
/* cover[1..n_pcs] = 実行されたカーネルコードのPCアドレス */
```

## システムコール記述（syzlang） {#syscall-descriptions-syzlang}

syzkallerは記述言語（syzlang）を使ってシステムコールの引数をモデル化する:

```
# sys/linux/socket.txt（抜粋）:
socket(domain flags[socket_domain], type flags[socket_type], proto int32) fd[sock]

connect(fd fd[sock], addr ptr[in, sockaddr], addrlen bytesize[addr])

# フラグ:
socket_domain = AF_UNIX, AF_INET, AF_INET6, AF_NETLINK, ...
socket_type = SOCK_STREAM, SOCK_DGRAM, SOCK_RAW, SOCK_SEQPACKET

# 構造体:
sockaddr_in {
    family  const[AF_INET, int16]
    port    proc[20000, 4, int16be]  # [20000, 20004]の範囲のポート
    addr    ipv4_addr                # 特殊なIPv4アドレス型
    pad     array[const[0, int8], 8]
}
```

これにより、syzkallerは完全にランダムなバイト列ではなく、それなりに妥当なシステムコール列を生成できる。

## syzkallerをローカルで実行する {#running-syzkaller-locally}

```bash
# 必要なもの: Go、QEMU、kcov付きLinuxカーネル

# 1. ファジング用カーネルをビルドする:
cat >> .config << 'EOF'
CONFIG_KCOV=y
CONFIG_DEBUG_INFO=y
CONFIG_KASAN=y
CONFIG_KASAN_INLINE=y
CONFIG_UBSAN=y
CONFIG_FAULT_INJECTION=y
CONFIG_FAULT_INJECTION_DEBUG_FS=y
CONFIG_FAILSLAB=y
CONFIG_FAIL_PAGE_ALLOC=y
CONFIG_FAIL_MAKE_REQUEST=y
CONFIG_LOCKDEP=y
CONFIG_PROVE_LOCKING=y
CONFIG_DEBUG_ATOMIC_SLEEP=y
CONFIG_FORTIFY_SOURCE=y
CONFIG_HARDENED_USERCOPY=y
CONFIG_RANDOMIZE_BASE=y
EOF
make -j$(nproc)

# 2. syzkallerをビルドする:
git clone https://github.com/google/syzkaller
cd syzkaller
make

# 3. 設定ファイル（syzkaller.cfg）を作成する:
cat > syzkaller.cfg << 'EOF'
{
    "target": "linux/amd64",
    "http": "127.0.0.1:56741",
    "workdir": "./workdir",
    "kernel_obj": "/path/to/kernel",
    "image": "/path/to/vm.img",
    "sshkey": "/path/to/id_rsa",
    "syzkaller": ".",
    "procs": 8,
    "type": "qemu",
    "vm": {
        "count": 4,
        "kernel": "/path/to/bzImage",
        "cpu": 2,
        "mem": 2048
    }
}
EOF

# 4. 実行する:
./bin/syz-manager -config syzkaller.cfg

# 5. ブラウザで開く: http://127.0.0.1:56741
# 表示内容: コーパスサイズ、発見したクラッシュ、カバレッジマップ
```

## クラッシュの再現 {#reproducing-crashes}

syzkallerはクラッシュを見つけると、**再現プログラム（reproducer）**を生成する:

```c
/* Cの再現プログラム（syz-reproツールが生成）: */
#define _GNU_SOURCE
#include <unistd.h>
#include <sys/syscall.h>
#include <sys/socket.h>
/* ... */

int main(void)
{
    /* バグを引き起こすシステムコールの列: */
    int sock = socket(AF_INET, SOCK_RAW, IPPROTO_ICMP);
    /* ... さらにシステムコール ... */
    return 0;
}
```

```bash
# 再現プログラムをコンパイルして実行する:
gcc -o repro repro.c && sudo ./repro
# 同じカーネルクラッシュが発生するはず

# あるいはsyzlangプログラムを直接使う:
./bin/syz-execprog -executor ./bin/syz-executor -repeat=10 crash.log
```

## syzbot: 継続的ファジング {#syzbot-continuous-fuzzing}

[syzbot](https://syzkaller.appspot.com)はアップストリームカーネル上でsyzkallerを24時間365日実行している:

```bash
# 自分のコードに影響するバグを検索する:
# https://syzkaller.appspot.com/upstream

# 各バグレポートに含まれるもの:
# - タイトル: 例 "KASAN: use-after-free in sk_destruct"
# - カーネルバージョンとコミット
# - Cの再現プログラム
# - syzlangの再現プログラム
# - スタックトレース

# バグを修正したことを報告するには:
# syzbotのメールに次の内容で返信する: #syz fix: <commit subject line>
# syzbotが検証してバグをクローズする
```

## トリアージと解析 {#triage-and-analysis}

```bash
# クラッシュアドレスをデコードする:
./scripts/faddr2line vmlinux sk_destruct+0x123/0x456
# → net/core/sock.c:1857

# 原因コミットを見つける（bisect）:
git bisect start
git bisect bad HEAD
git bisect good v5.15
git bisect run ./scripts/faddr2line vmlinux <addr>

# crashでコアダンプを解析する:
crash vmlinux core.dump
crash> bt      # クラッシュしたプロセスのバックトレース
crash> dis -l sk_destruct  # ソース行付きで逆アセンブル
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/kcov.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/kcov.c) — `CONFIG_KCOV`のもとでコンパイラが挿入するエッジごとのフックである`__sanitizer_cov_trace_pc()`と、`kcov_ioctl()`による`KCOV_INIT_TRACE`/`KCOV_ENABLE`/`KCOV_DISABLE`の処理
- [scripts/faddr2line](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/faddr2line) — 「トリアージと解析」の節で使った、クラッシュアドレスをソース行に変換するデコーダ

### manページ {#man-pages}

- [`ioctl(2)`](https://man7.org/linux/man-pages/man2/ioctl.2.html) — kcovが`KCOV_INIT_TRACE`、`KCOV_ENABLE`、`KCOV_DISABLE`に使う`ioctl()`インターフェース

### 関連ページ {#related-pages}

- [KASAN](../mm/kasan.md) — syzkallerと併用されるメモリエラーサニタイザ
- [KCSAN](kcsan.md) — データ競合検出器
- [Oops解析](oops-analysis.md) — クラッシュレポートの解析
- [kdumpとcrash](kdump.md) — クラッシュダンプの取得
- [フォールトインジェクション](../mm/fault-injection.md) — エラーパスを強制的に通す

### LWN記事 {#lwn-articles}

- [Coverage-guided kernel fuzzing with syzkaller](https://lwn.net/Articles/677764/)（2016） — syzkallerを最初に紹介したLWN記事: 動機、再現の手順、KCOV/KASANの利用
- [Scrutinizing bugs found by syzbot](https://lwn.net/Articles/872649/)（2021） — syzbotが最初の4年間で発見した約4,000件のバグの分析
- [Troubles with triaging syzbot reports](https://lwn.net/Articles/917762/)（2022） — syzbotの自動バグレポートがメンテナ側に課すコスト

### 外部資料 {#external}

- [syzkaller](https://github.com/google/syzkaller) — Googleの公式リポジトリ
- [syzbot documentation](https://github.com/google/syzkaller/blob/master/docs/syzbot.md) — syzbotレポートをメールでトリアージするための`#syz fix:`、`#syz dup:`などのコマンド
- [Reproducing crashes](https://github.com/google/syzkaller/blob/master/docs/reproducing_crashes.md) — syzkallerがSyzおよびCの再現プログラムを生成する方法と`syz-repro`ツール
- [Syscall description syntax](https://github.com/google/syzkaller/blob/master/docs/syscall_descriptions_syntax.md) — `sys/linux/*.txt`で使われるsyzlangの文法
- [Setting up syzkaller for Linux](https://github.com/google/syzkaller/blob/master/docs/linux/setup.md) — `CONFIG_KCOV`と`syz-manager`のセットアップ。`CONFIG_KASAN`を含む推奨設定一式は[kernel_configs.md](https://github.com/google/syzkaller/blob/master/docs/linux/kernel_configs.md)へのリンク先にある
- [syzbot](https://syzkaller.appspot.com) — 継続的ファジングのダッシュボード
- [KCOV: code coverage for fuzzing](https://docs.kernel.org/dev-tools/kcov.html) — KCOVインターフェースに関するカーネル自身のドキュメント。ioctl/mmapの手順や、本ページの例で使った`cover[0]`を件数、`cover[1..n]`をPCアドレスとするバッファレイアウトを含む
