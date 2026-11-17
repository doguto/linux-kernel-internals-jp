# io_uring {#io_uring}

> ファイル、ソケット、パイプなどあらゆるものに対する真の非同期I/O。操作ごとのシステムコールは不要

## はじめに {#getting-started}

このドキュメントでは、io_uringがどのように動作するかを解説する。APIだけでなく、リングレイアウトの背後にある設計判断、それを形作ったトレードオフ、そしてデータベース、ストレージエンジン、ネットワークサーバーで使われている実世界のパターンまで扱う。

### 前提知識 {#prerequisites}

このドキュメントは以下の知識を前提とする。

- **Cプログラミング**: io_uringのAPIもカーネル実装もCで書かれている
- **ファイルディスクリプタとシステムコール**: `read()`、`write()`、`accept()`がOSレベルでどう動作するか
- **基本的な非同期の概念**: イベントループ、ノンブロッキングI/O、コールバック

`epoll`や`libaio`を使ったことがあれば、十分な基礎がある。そうでなくても、ブロッキングI/OとノンブロッキングI/Oの違いを理解していれば始めるには十分である。

### カーネルソースの入手 {#getting-the-kernel-source}

```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
```

io_uringの実装は`io_uring/`にある（約25,000行）。公開ABIは`include/uapi/linux/io_uring.h`にある。

### 推奨する読む順序 {#suggested-reading-order}

**まずここから:**

1. **[アーキテクチャとリング](io-uring-arch.md)**: リングレイアウト、SQE/CQE構造体、サブミッションとコンプリーションの流れ
2. **[リクエストの一生](life-of-request.md)**: エンドツーエンドのトレース。ユーザー空間のSQEからカーネルでの完了、CQEまで

**操作:**

3. **[操作リファレンス](io-uring-ops.md)**: 全オペコード表、フラグ、操作ごとのセマンティクス

**発展的なトピック:**

4. **[固定バッファと固定ファイル](fixed-buffers.md)**: 事前登録したバッファとファイルディスクリプタ、ゼロコピーI/O
5. **[マルチショット操作](multishot-ops.md)**: 1つのSQEから多数のCQE。マルチショットのaccept、recv、poll
6. **[ネットワーキング](networking.md)**: acceptループ、ゼロコピー送信、TCPゼロコピー受信

**比較とセキュリティ:**

7. **[io_uring vs epoll](io-uring-vs-epoll.md)**: それぞれをいつ使うか、epollにできないこと、移行パターン
8. **[セキュリティ](security.md)**: 必要な特権、攻撃対象領域、seccompによるフィルタリング
9. **[事件簿](war-stories.md)**: 実際のバグ、CVE、本番環境からの教訓

### 学べること {#what-youll-learn}

| 教科書的な概念 | Linuxの実際 |
|------------------|---------------|
| 「非同期I/Oにはスレッドが必要」 | io_uringの[SQPOLL](io-uring-arch.md)モードでは、アプリケーションスレッドなしでI/Oを駆動できる。カーネルスレッドがSQリングをポーリングする |
| 「I/O 1回につきシステムコール1回」 | io_uringはサブミッションをバッチ化する。SQPOLLを使えば`io_uring_enter()`を完全になくせる |
| 「epollはすべてのI/Oを扱える」 | [epollはファイルI/Oを扱えない](io-uring-vs-epoll.md)。通常ファイルは常にreadyだからである。io_uringはファイル、ソケット、パイプを統一的に扱う |
| 「io_uringはサーバー専用」 | データベース（RocksDB、ScyllaDB）、ストレージエンジン、ゲームエンジンはいずれもファイルI/Oにio_uringを使っている |
| 「ゼロコピーならCPUを使わない」 | [固定バッファ](fixed-buffers.md)はユーザー空間とカーネル間のデータコピーをなくすが、DMAのセットアップや完了割り込みは依然としてCPUを消費する |

### このドキュメントの特徴 {#what-makes-this-different}

- **実際のコミット**: すべての主張は実際のカーネルコミットにリンクしている
- **実際のバグ**: 何が壊れ、なぜ壊れたかから学ぶ
- **実際の議論**: 設計判断が議論されたLKMLへのリンク
- **実際のトレードオフ**: 「何を」だけでなく「なぜあれではなくこれなのか」
- **ハンズオン**: 実行するコマンドと読むべきコードを示した「自分で試す」セクション

---

## io_uringとは何か {#what-io_uring-is}

io_uringはLinux 5.1（2019年5月）でJens Axboeによって導入された。共有メモリ上の2つのロックフリーなリングを中心に構築された、非同期I/Oのためのカーネルインターフェースである。

- **サブミッションキュー（SQ）**: ユーザー空間がI/O操作を記述したSQEを書き込む
- **コンプリーションキュー（CQ）**: カーネルが結果を格納したCQEを投稿する

両方のリングがユーザー空間のメモリにマップされているため、一般的なケースではI/Oの投入と回収にシステムコールが不要である。SQPOLLを有効にした場合は特にそうである。

```
Userspace                          Kernel
────────────────────────────────────────────────────────────────
                              ┌───────────────────────┐
  SQ ring (mmap'd)  ────────► │  io_uring instance    │
  SQE array (mmap'd) ────────►│  (struct io_ring_ctx) │
                              └───────────────────────┘
  CQ ring (mmap'd)  ◄────────         │
                              io_uring_enter() or SQPOLL
                                       │
                               direct dispatch / io-wq
                                       │
                               complete → post CQE
```

### io_uringはepollやPOSIX AIOとどう違うか {#how-io_uring-differs-from-epoll-and-posix-aio}

**epoll**はレディネス通知の仕組みである。ファイルディスクリプタがブロックせずに読み込み*可能*になったときにそれを知らせる。実際にデータを転送するには2回目のシステムコールが依然として必要である。ファイルは常に「ready」であるため、通常ファイルのI/Oには使えない。

**POSIX AIO**（`aio_read`、`aio_write`）はglibc内でスレッドプールとして実装されており、真のカーネルインターフェースではない。性能が低く、シグナルベースの複雑な通知モデルを持ち、対応する操作も限られている。

**Linux libaio**（`io_submit`、`io_getevents`）は本物のカーネルインターフェースだが、O_DIRECTのブロックI/Oに限定され、ソケットをサポートせず、操作ごとにシステムコールが必要である。

**io_uring**はこれらすべてに対処する。

| 機能 | epoll | libaio | io_uring |
|---------|-------|--------|----------|
| 通常ファイルのI/O | 不可 | O_DIRECTのみ | 可 |
| ソケットI/O | 可 | 不可 | 可 |
| システムコールなしのパス | 不可 | 不可 | 可（SQPOLL） |
| バッチ化されたサブミッション | 不可 | 限定的 | 可 |
| リンクされた操作 | 不可 | 不可 | 可 |
| 固定バッファ | 不可 | 不可 | 可 |
| マルチショット操作 | 不可 | 不可 | 可 |

---

## ドキュメント {#documentation}

### 中核となる読み物 {#core-reading}

| ドキュメント | 学べること |
|----------|-------------------|
| [io-uring-arch](io-uring-arch.md) | リングレイアウト、SQE/CQE構造体、`io_uring_setup`、サブミッションとコンプリーションの流れ |
| [life-of-request](life-of-request.md) | 単一のreadのエンドツーエンドのトレース。SQEの投入、カーネルのディスパッチ、CQEの投稿 |
| [io-uring-ops](io-uring-ops.md) | オペコード表、操作ごとのフラグとセマンティクス、キャンセル |

### 発展的なトピック {#advanced-topics}

| ドキュメント | 学べること |
|----------|-------------------|
| [fixed-buffers](fixed-buffers.md) | 事前登録したバッファとファイルディスクリプタ。I/Oごとのセットアップコストをなくす |
| [multishot-ops](multishot-ops.md) | 1つのSQEが多数のCQEを生成する。マルチショットのaccept、recv、poll |
| [networking](networking.md) | 高スループットなacceptループ、ゼロコピーTCP送信、バッファ選択付きのrecv |

### 比較とセキュリティ {#comparison-and-security}

| ドキュメント | 学べること |
|----------|-------------------|
| [io-uring-vs-epoll](io-uring-vs-epoll.md) | それぞれのモデルが適する場面、epollにできないこと、移行パターン |
| [security](security.md) | 攻撃対象領域、必要な特権、seccompとio_uring、CVEの歴史 |
| [war-stories](war-stories.md) | 実際のバグ、本番環境での障害、そしてそれらが実装について教えてくれること |

---

## 主要なソースファイル {#key-source-files}

実際のカーネルコードを読みたい場合:

| ファイル | 役割 |
|------|--------------|
| [`io_uring/io_uring.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/io_uring/io_uring.c) | リング管理の中核、`io_uring_setup`、`io_uring_enter`、コンテキストのライフサイクル |
| [`io_uring/rw.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/io_uring/rw.c) | 読み書き操作。`IORING_OP_READ`、`IORING_OP_WRITE`、ベクタ版 |
| [`io_uring/net.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/io_uring/net.c) | ネットワーク操作。`IORING_OP_ACCEPT`、`IORING_OP_SEND`、`IORING_OP_RECV` |
| [`io_uring/sqpoll.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/io_uring/sqpoll.c) | SQPOLLカーネルスレッド。ユーザー空間のシステムコールなしでSQリングをポーリングする |
| [`io_uring/io-wq.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/io_uring/io-wq.c) | インラインでディスパッチできない操作のための非同期ワーカースレッドプール |
| [`io_uring/rsrc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/io_uring/rsrc.c) | 固定バッファと固定ファイルの登録、リソーステーブルの管理 |
| [`include/uapi/linux/io_uring.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/io_uring.h) | ユーザー空間ABI。`io_uring_sqe`、`io_uring_cqe`、オペコード、フラグ、機能ビット |

---

## クイックリファレンス {#quick-reference}

liburingを使った最小限のio_uring読み込みループ:

```c
#include <liburing.h>
#include <fcntl.h>
#include <stdio.h>

int main(void) {
    struct io_uring ring;
    struct io_uring_sqe *sqe;
    struct io_uring_cqe *cqe;
    char buf[4096];
    int fd, ret;

    /* 32個のSQEスロットでリングを初期化 */
    io_uring_queue_init(32, &ring, 0);

    fd = open("/etc/hostname", O_RDONLY);

    /* 空きSQEスロットを取得し、readを記述する */
    sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, fd, buf, sizeof(buf), 0);
    sqe->user_data = 42;   /* CQEにそのままコピーされるタグ */

    /* 投入: 保留中のSQEがあることをカーネルに伝える */
    io_uring_submit(&ring);

    /* 完了を1つ待つ */
    ret = io_uring_wait_cqe(&ring, &cqe);
    if (ret == 0) {
        printf("read %d bytes (tag=%llu)\n", cqe->res, cqe->user_data);
        io_uring_cqe_seen(&ring, cqe);   /* CQのheadを進める */
    }

    io_uring_queue_exit(&ring);
    return 0;
}
```

コンパイル方法:

```bash
gcc -o example example.c -luring
```

### 主要なパターン {#key-patterns}

**SQPOLL（システムコールなしのI/O）:**

```c
struct io_uring_params params = {
    .flags = IORING_SETUP_SQPOLL,
    .sq_thread_idle = 2000,  /* スレッドがスリープするまでのミリ秒 */
};
int ring_fd = io_uring_setup(128, &params);
/* これ以降、io_uring_submit()はio_uring_enter()を必要としない */
```

**リンクされたリクエスト（順序付きチェーン）:**

```c
/* writeの後にfsync。順序が保証される */
sqe = io_uring_get_sqe(&ring);
io_uring_prep_write(sqe, fd, buf, len, 0);
sqe->flags |= IOSQE_IO_LINK;   /* 次のSQEにリンクする */

sqe = io_uring_get_sqe(&ring);
io_uring_prep_fsync(sqe, fd, 0);
/* ここにはIOSQE_IO_LINKを付けない。チェーンを終端する */

io_uring_submit(&ring);
```

---

## 参考文献 {#further-reading}

### Jens Axboeによる論文と記事 {#papers-and-posts-by-jens-axboe}

- Axboe, [*Efficient IO with io_uring*](https://kernel.dk/io_uring.pdf) (2019): 最初の設計論文。リングレイアウト、SQPOLL、固定バッファを解説している
- Axboe, [*Lord of the io_uring*](https://unixism.net/loti/): 実例付きの実践的なガイド（Shuveb Hussainがホスト）

### LWNの記事 {#lwn-articles}

- [*The rapid growth of io_uring*](https://lwn.net/Articles/810414/) (2020): 機能と設計の根拠に関する初期の概観
- [*Ringing in a new asynchronous I/O API*](https://lwn.net/Articles/776703/) (2019): 既存の非同期インターフェースとの比較
- [*Operations restrictions for io_uring*](https://lwn.net/Articles/826053/) (2020): セキュリティモデルとseccompとの相互作用

### ユーザー空間ライブラリ {#userspace-library}

- [GitHub上のliburing](https://github.com/axboe/liburing): 標準のユーザー空間ライブラリ。すべてのオペコードの例とラッパーを含む

### カーネルドキュメント {#kernel-documentation}

- [`include/uapi/linux/io_uring.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/io_uring.h): ABIの決定版リファレンス

### 教科書と背景知識 {#textbooks-and-background}

- Love, [*Linux Kernel Development*](https://www.oreilly.com/library/view/linux-kernel-development/9780768696974/): 第13章（Block I/O）。io_uringが土台とするI/Oスタックの背景知識
- Stevens & Rago, [*Advanced Programming in the UNIX Environment*](https://www.oreilly.com/library/view/advanced-programming-in/9780321638014/): 第14章（Advanced I/O）。io_uring以前の状況を理解するために

---

## 付録 {#appendix}

- **[用語集](../mm/glossary.md)**: 用語リファレンス（SQE、CQE、SQPOLL、io-wq、固定バッファ、マルチショット）
