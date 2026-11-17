# io_uringのセキュリティと堅牢化 {#io_uring-security-and-hardening}

> 攻撃対象領域、seccompのバイパス、CVE、本番環境でのロックダウン

## 攻撃対象領域 {#the-attack-surface}

io_uringは、**io-wq**と呼ばれるカーネルのワーカースレッドプールから非同期操作を実行する。操作がインラインで完了できない場合（ブロッキングするread、ファイルのopenなど）、カーネルはそれをio-wqスレッドにキューイングする。このスレッドは投入したタスクのクレデンシャル（同じUID、GID、ケーパビリティ、オープンファイルテーブル）で動作するが、**通常のシステムコールパスの外**で動作する。

これが重要なのは、システムコールの境界を前提に設計されたあらゆるカーネルのセキュリティフックが、io-wqでの実行には無関係になってしまうからである。

- **seccompフィルタ**はシステムコールの入口で評価される。io-wqスレッドは、ユーザー空間からシステムコールを発行することなく同等の操作を実行する
- **ptrace**ベースのサンドボックスはシステムコールを観測する。io-wqの処理はそれらからは見えない
- システムコール番号をキーにした**auditルール**は、非同期パスでは発火しない場合がある
- SQEの**personality**フィールドを使うと、*別の登録済みクレデンシャル*で操作を投入できる。これは正当な機能だが、攻撃対象領域を大きく広げる

`io_uring/io-wq.c`を通るio-wqのパスと、`io_uring/io_uring.c`のサブミッション機構の中核は、大きく複雑なステートマシンを形成している。このステートマシンにおける微妙なライフタイムのバグや型の取り違え（type confusion）は、権限昇格のCVEの原因に何度もなってきた。

## seccompとio_uring {#seccomp-and-io_uring}

プロセスは、ほぼすべてのシステムコールを許可しない厳格なseccompフィルタをインストールした上で、io_uringを使って同じ操作を非同期に実行できる。io-wqスレッドが処理を実行するとき、カーネルは投入元プロセスのseccompポリシーを再評価しない。

```
Userspace (seccomp: deny openat) ──► io_uring_enter ──► SQE: IORING_OP_OPENAT
                                                              │
                                          ┌───────────────────┘
                                          │  io-wq thread
                                          │  (no seccomp check)
                                          ▼
                                      do_filp_open()   ← opens the file
```

`IORING_SETUP_SQPOLL`はこれをさらに悪化させる。専用のカーネルポーリングスレッドがSQリングを継続的に処理するため、リングのセットアップ後は、プロセスは`io_uring_enter`（seccompでブロック*できる*システムコール）を呼ぶ必要すらない。

`IORING_SETUP_NO_SQTHREAD`（デフォルト。`IORING_SETUP_SQPOLL`を省略した状態）ではポーリングスレッドはなくなるが、インラインで完了できない非同期処理については、io-wqスレッドが依然としてseccompをバイパスする。

`IORING_SETUP_SINGLE_ISSUER`（Linux 6.2）は、投入をリングを作成した1つのスレッドに制限するが、seccompバイパスの性質は変わらない。

実際上の帰結は、**io_uringのファイルディスクリプタにアクセスできるプロセスにとって、seccompだけでは十分なサンドボックスにならない**ということである。Android、Chrome OS、gVisorはいずれも、seccompポリシーで`io_uring_setup`を完全に無効化することでこれに対応している（後述）。

## 時とともに追加された制限 {#restrictions-added-over-time}

カーネルはio_uringの特権境界を段階的に厳しくしてきた。

| バージョン | 変更 |
|---------|--------|
| 5.10 | `io_uring_enter()`の制限。リングを作成したタスクだけがこれを呼べる |
| 6.2 | `IORING_SETUP_SINGLE_ISSUER`。1つのスレッドだけがリングにSQEを投入できる |
| 6.1 | `IORING_SETUP_DEFER_TASKRUN`。完了処理は投入したスレッド上で実行される。SQPOLLカーネルスレッドを一切生成しない |
| すべて | `IORING_REGISTER_RESTRICTIONS`。リングが使用を許されるオペコードと登録操作をホワイトリストで指定する |

`IORING_SETUP_DEFER_TASKRUN`はサンドボックス化されたコンテキストで特に有用である。遅延された完了処理のためにio-wqスレッドが生成されず、すべての実行が既知でフィルタ可能なユーザー空間スレッド上にとどまる。

## `IORING_REGISTER_RESTRICTIONS` {#ioringregisterrestrictions}

リングをセットアップした後、特権を持つコンポーネントは、リングのfdを信頼できないコンポーネントに渡す前に**リングをロックダウン**できる。`IORING_REGISTER_RESTRICTIONS`の登録呼び出し（`io_uring_register(2)`）は、`struct io_uring_restriction`エントリの配列を受け取る。一度適用された制限はリングのライフタイムの間ずっと有効であり、削除できない。

制限の種類:

| 種類 | 効果 |
|------|--------|
| `IORING_RESTRICTION_REGISTER_OP` | 指定した`IORING_REGISTER_*`操作のみを許可する |
| `IORING_RESTRICTION_SQE_OP` | SQEで指定した`IORING_OP_*`オペコードのみを許可する |
| `IORING_RESTRICTION_SQE_FLAGS_ALLOWED` | ユーザー空間が設定してよいSQEフラグのマスク |
| `IORING_RESTRICTION_SQE_FLAGS_REQUIRED` | すべてのSQEに設定しなければならないSQEフラグのマスク |

実装は`io_uring/rsrc.c`（リソースの登録）と`io_uring/io_uring.c`（`io_uring_register`のディスパッチと制限の強制）にある。

### 例: 制限付きリングを信頼できないコンポーネントに渡す {#example-handing-a-restricted-ring-to-an-untrusted-component}

```c
#include <linux/io_uring.h>
#include <sys/syscall.h>
#include <unistd.h>
#include <string.h>
#include <stdint.h>

/* ラッパー。liburingはio_uring_register()を提供しているが、ここでは生の呼び出しを示す */
static int uring_register(int ring_fd, unsigned opcode, void *arg, unsigned nr_args)
{
    return (int)syscall(__NR_io_uring_register, ring_fd, opcode, arg, nr_args);
}

int setup_restricted_ring(unsigned sq_entries)
{
    struct io_uring_params params;
    memset(&params, 0, sizeof(params));

    /*
     * IORING_SETUP_SINGLE_ISSUER: 作成したスレッドだけが投入できる。
     * IORING_SETUP_DEFER_TASKRUN: SQPOLLスレッドなし。完了処理は
     *   投入スレッド上で実行され、実行がio-wqに移らない。
     * 両フラグを組み合わせて使うにはLinux 6.2が必要。
     */
    params.flags = IORING_SETUP_SINGLE_ISSUER |
                   IORING_SETUP_DEFER_TASKRUN;

    int ring_fd = (int)syscall(__NR_io_uring_setup, sq_entries, &params);
    if (ring_fd < 0)
        return ring_fd;

    /* 制限リストを構築する */
    struct io_uring_restriction restrictions[] = {
        /* IORING_OP_READとIORING_OP_WRITEのみを許可する */
        {
            .opcode     = IORING_RESTRICTION_SQE_OP,
            .sqe_op     = IORING_OP_READ,
        },
        {
            .opcode     = IORING_RESTRICTION_SQE_OP,
            .sqe_op     = IORING_OP_WRITE,
        },
        /* SQEフラグでIOSQE_FIXED_FILEとIOSQE_IO_DRAINを禁止する */
        {
            .opcode             = IORING_RESTRICTION_SQE_FLAGS_ALLOWED,
            .sqe_flags          = IOSQE_IO_LINK | IOSQE_ASYNC,
        },
    };

    int ret = uring_register(ring_fd, IORING_REGISTER_RESTRICTIONS,
                              restrictions,
                              sizeof(restrictions) / sizeof(restrictions[0]));
    if (ret < 0) {
        close(ring_fd);
        return ret;
    }

    /*
     * これでring_fdはロックされた。READかWRITE以外のオペコード、または
     * 禁止されたフラグを持つSQEは-EACCESで拒否される。
     * ring_fdを信頼できないコンポーネントに渡す。
     */
    return ring_fd;
}
```

`IORING_REGISTER_RESTRICTIONS`が適用された後、カーネルはサブミッションのホットパスで、各SQEを`io_ring_ctx->restrictions`の制限ビットマップと照合する。禁止されたオペコードやフラグを使おうとすると`-EACCES`が返る。

## CVEとカーネルの堅牢化 {#cves-and-kernel-hardening}

io_uringのステートマシンの複雑さ（キャンセル、タイムアウト、リンクされたリクエスト、固定バッファ、登録済みクレデンシャル）は、微妙なメモリ安全性のバグが入り込む大きな余地を生む。

**CVE-2022-29582**: io_uringのファイルテーブル処理（`io_uring/fdinfo.c`、`io_uring/rsrc.c`）におけるuse-after-free。ファイルテーブルの更新と非同期操作の後始末との間の競合状態により、ローカルの攻撃者が権限を昇格できた。Linux 5.17.3 / 5.15.34で修正。CVSS 7.8。

**CVE-2023-2598**: 固定バッファ登録における整数オーバーフロー。`IORING_REGISTER_BUFFERS`で固定バッファを登録する際、バッファ数の検証が不十分だったため、オーバーフローが起きてカーネルメモリが破壊された。Linux 6.3で修正。脆弱な計算は`io_uring/rsrc.c:io_sqe_buffers_register()`にあった。

**一般的なパターン**: io_uringのCVEの多くは同じ形をしている。非同期の後始末パスで参照カウント、ライフタイム、またはサイズ計算がおかしくなり、解放済みオブジェクトが悪用可能なプリミティブに変わる。複数のio-wqスレッド、ユーザーが制御するライフタイム、共有されるカーネルオブジェクトの組み合わせは、本質的に推論が難しい。

最近のカーネルを使い続けることが最も効果的な緩和策である。io_uringサブシステムはstableツリーにバックポートされたセキュリティ修正を受け取るが、それは限られた期間だけである。

## Android、Chrome OS、gVisorでの制限 {#android-chrome-os-and-gvisor-restrictions}

これらのプラットフォームは、自らの脅威モデルにとってseccompバイパスのリスクが性能上の利点を上回ると結論づけた。

**Android**（Android 12以降）: アプリのプロセスに適用されるデフォルトのseccompポリシーで`io_uring_setup`がブロックされる（`SCMP_ACT_ERRNO`）。特権を持つシステムプロセスは使用できるが、信頼できないアプリのコードはリングを作成できない。これはBionicのseccompフィルタテーブルで強制されている。

**Chrome OS**: Chromeのサンドボックスは、レンダラーとGPUプロセスのサンドボックスで`io_uring_setup`を拒否するseccomp-bpfポリシーを使う。この判断は、seccompフィルタがio-wqの操作をカバーしないことを理由に明示的に下された。

**gVisor**: gVisorのカーネル（`runsc`）はsentryの境界でシステムコールをインターセプトする。io_uringの非同期実行モデルはgVisorのシステムコール介在アーキテクチャと根本的に相容れないため、gVisorのコンテナではデフォルトで`io_uring_setup`が`ENOSYS`を返す。これをサポートするには、sentryがio-wqの実行モデル全体をエミュレートする必要がある。

**Linux Security Modules**: SELinuxとAppArmorは、`io_uring`オブジェクトクラス（Linux 5.12で追加）を通じてio_uringの操作を制限できる。ポリシーは、一般的なio_uringへのアクセスとは独立に、`sqpoll`（SQPOLLカーネルスレッドの生成）と`override_creds`（`personality`フィールドを使って別のクレデンシャルで操作を実行すること）を拒否できる。

```
# SELinux: confined_tに対してSQPOLLスレッドの作成を拒否する
deny confined_t self:io_uring sqpoll;

# AppArmor: クレデンシャルのオーバーライドを拒否する
deny io_uring override_creds,
```

## 本番環境でのベストプラクティス {#best-practices-for-production}

**フラグを最小限にする**

- デフォルト: ポーリングのレイテンシが決定的に重要で、かつプロセスが信頼できる場合を除き、`IORING_SETUP_SQPOLL`は省略する。SQPOLLは、プロセスのクレデンシャルで継続的に動作するカーネルスレッドを生成する。
- 単一の投入スレッドからアクセスされるリングには、`IORING_SETUP_SINGLE_ISSUER`（Linux 6.2以降）を使う。
- 完了処理のためにio-wqスレッドが生成されるのを避けるため、`IORING_SETUP_SINGLE_ISSUER`と併せて`IORING_SETUP_DEFER_TASKRUN`（Linux 6.1以降）を使う。

**サンドボックス化されたコンテキストではリングをロックダウンする**

`io_uring_setup`の直後、リングのfdを信頼できないコードに渡す前に`IORING_REGISTER_RESTRICTIONS`を適用する。コンポーネントが必要とする最小限のオペコードに制限する。これにより、カーネルに未修正の脆弱性があったとしても、侵害されたコンポーネントの被害範囲を限定できる。

**seccompポリシー**

プロセスがio_uringをまったく必要としないなら、seccompフィルタで`io_uring_setup`をブロックする。プロセスが初期化中にリングを作成してから特権を落とすなら、リング作成後に`io_uring_setup`をブロックする。これにより、侵害されたプロセスが制限のない追加のリングを作成することを防げる。

**必要でない限り`personality`（登録済みクレデンシャル）を避ける**

SQEの`personality`フィールドを使うと、事前に登録したクレデンシャルセット（`IORING_REGISTER_PERSONALITY`）で操作を実行できる。これは強力な機能だが、誤用しやすい。`IORING_RESTRICTION_REGISTER_OP`でその使用を制限し、リングのセットアップ後に新しいpersonalityが登録されるのを防ぐ。

**auditで監視する**

カーネルのauditサブシステムは、`io_uring_setup`と`io_uring_register`に対してレコードを出力する。auditdやauditログを取り込むSIEMがある環境では、これらのイベント種別に対するルールにより、予期しないリングの作成やケーパビリティの登録を検出できる。

## 参考文献 {#further-reading}

- [アーキテクチャとリング](io-uring-arch.md): SQ/CQのレイアウト、SQE/CQE構造体
- [操作と高度な機能](io-uring-ops.md): サポートされる操作、SQPOLL、固定バッファ
- `io_uring/io_uring.c`: サブミッションパスの中核と制限の強制
- `io_uring/rsrc.c`: 固定バッファとクレデンシャルの登録。CVE-2023-2598の発生箇所
- `io_uring/io-wq.c`: ワーカースレッドプールの実装
- `include/uapi/linux/io_uring.h`: `IORING_RESTRICTION_*`定数と`struct io_uring_restriction`
- [Auditing io_uring](https://lwn.net/Articles/858023/): io_uringへのLSMとauditフックの追加に関するLWNの記事
- [CVE-2022-29582 writeup (Qualys)](https://www.qualys.com/2022/05/02/cve-2022-29582/lpe-io-uring.txt): 詳細な悪用の分析
