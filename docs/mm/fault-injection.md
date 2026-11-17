# メモリテストのためのフォールトインジェクション {#fault-injection-for-memory-testing}

> カーネルの割り当てを意図的に失敗させてエラー処理パスを動かす

## フォールトインジェクションとは何か {#what-is-fault-injection}

フォールトインジェクションとは、ドライバやサブシステムのエラー処理パスが正しく動作するかをテストするために、カーネルの操作を意図的に失敗させる手法である。Linuxカーネルにはこれを体系的に行うための組み込みのフォールトインジェクションフレームワークがある。失敗の確率、範囲、対象を設定すると、カーネルは特定の操作をランダムに失敗させ始める。

最も一般的な用途は、メモリ割り当ての失敗の注入である。つまり`kmalloc()`、`alloc_pages()`および関連する呼び出しが`NULL`や`-ENOMEM`を返すようにする。よく書かれたドライバは、こうした失敗を穏便に処理しなければならない（部分的な割り当てをすべて解放し、エラーを返し、panicしない）。出来の悪いドライバはoopsを起こしたり、メモリをリークさせたり、黙って状態を破壊したりする。フォールトインジェクションは、自分のコードがどちらに当てはまるかを明らかにする。

!!! tip "なぜこれが重要か"
    エラーパスは十分にテストされていないことで悪名高い。ドライバには、実際には一度も実行されたことのない`goto err_free`のパスがあるかもしれない。フォールトインジェクションは、本番環境で本物の割り当て失敗がそれを露呈させる前に、管理された条件下で（理想的にはVM内で）それらのパスを動かす。

## フォールトインジェクションフレームワーク {#the-fault-injection-framework}

### カーネル設定 {#kernel-configuration}

```
# フォールトインジェクションの中核インフラ
CONFIG_FAULT_INJECTION=y
CONFIG_FAULT_INJECTION_DEBUG_FS=y     # debugfsインターフェース（実行時の制御に必要）
CONFIG_FAULT_INJECTION_STACKTRACE_FILTER=y  # 呼び出し元でフィルタする（CONFIG_STACKTRACE=yが必要）

# 個別のフォールト種別
CONFIG_FAILSLAB=y          # kmalloc / kmem_cache_alloc（slab割り当て）を失敗させる
CONFIG_FAIL_PAGE_ALLOC=y   # alloc_pages / get_free_pages（ページアロケータ）を失敗させる
CONFIG_FAIL_FUTEX=y        # futex操作を失敗させる（futexのエラーパスのテスト用）
CONFIG_FAIL_MAKE_REQUEST=y # ブロックI/O要求を失敗させる（bio層）
CONFIG_FAIL_IO_TIMEOUT=y   # I/Oタイムアウトを注入する
CONFIG_FAIL_SUNRPC=y       # SunRPC呼び出しを失敗させる（NFSのテスト用）
```

フレームワークの中核は[`lib/fault-inject.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/fault-inject.c)にある。個々のサブシステムのフックは次の場所にある。

- [`mm/failslab.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/failslab.c) — slab割り当ての失敗
- [`mm/fail_page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/fail_page_alloc.c) — ページアロケータの失敗

### フレームワークが失敗させるかどうかを決める方法 {#how-the-framework-decides-whether-to-fail}

フレームワークは、`should_fail()`が呼ばれるたびに一連の設定可能な属性を評価する。

```
should_fail() decision logic (lib/fault-inject.c):

  1. Is fault injection enabled? (probability > 0)
  2. Are we past the initial no-fail period? (interval counter)
  3. Does the random draw pass probability check?
  4. Is the call site in the stack trace filter? (if configured)
  5. Is this process/task marked make-it-fail? (if using per-task control)
  6. → YES to all: return true (fail this call)
```

これらはそれぞれ、debugfsまたはタスクごとのファイルで調整できる。

## debugfsインターフェース {#the-debugfs-interface}

`CONFIG_FAULT_INJECTION_DEBUG_FS=y`が有効な場合、各フォールト種別は`/sys/kernel/debug/`の下にディレクトリを公開する。

```
/sys/kernel/debug/
├── fail_page_alloc/
│   ├── probability        ← integer 0-100 (percent chance of failure per call)
│   ├── interval           ← fail at most 1 in N calls (0 = no limit)
│   ├── times              ← how many times to fail (-1 = unlimited)
│   ├── space              ← call-budget counter; decremented by alloc size; injection suppressed until it reaches zero
│   ├── verbose            ← 0=quiet, 1=log failures, 2=log + stack trace
│   ├── task-filter        ← if 1: only fail tasks with make-it-fail=1
│   └── stacktrace-depth   ← how many stack frames to inspect for filtering
├── failslab/
│   ├── probability
│   ├── interval
│   ├── times
│   ├── space
│   ├── verbose
│   ├── cache-filter       ← if 1: use per-cache sysfs enable/disable
│   └── task-filter
└── fail_make_request/
    ├── probability
    ├── interval
    ├── times
    └── verbose
```

### 属性リファレンス {#attribute-reference}

| 属性 | 型 | 意味 |
|-----------|------|---------|
| `probability` | 0–100 | 対象となる各呼び出しが失敗する確率（パーセント） |
| `interval` | 1以上の整数 | 注入の最小間隔: 対象となる呼び出しN回につき最大1回失敗させる。`1` = 対象となる呼び出しを毎回失敗させる |
| `times` | 整数 | 失敗を注入する回数。`-1` = 無制限 |
| `space` | 整数 | 呼び出し予算のカウンタ: `should_fail()`が呼ばれるたびに割り当ての`size`だけ減算される。`space`がゼロに達するまで注入は抑制される |
| `verbose` | 0、1、2 | `0` = 出力なし、`1` = 失敗ごとにログ出力、`2` = ログ出力に加えてスタックトレースをダンプ |
| `task-filter` | 0または1 | `1`の場合、`/proc/<pid>/make-it-fail`を設定したタスクによる割り当てのみを失敗させる |

### クイックスタート: すべてのページ割り当てを10%で失敗させる {#quick-start-fail-all-page-allocations-at-10}

```bash
# まだマウントされていなければdebugfsをマウントする
mount -t debugfs none /sys/kernel/debug

# fail_page_allocを有効化: 確率10%、回数無制限
echo 10  > /sys/kernel/debug/fail_page_alloc/probability
echo -1  > /sys/kernel/debug/fail_page_alloc/times
echo  0  > /sys/kernel/debug/fail_page_alloc/interval
echo  1  > /sys/kernel/debug/fail_page_alloc/verbose

# 設定を確認する
cat /sys/kernel/debug/fail_page_alloc/probability
```

!!! warning
    `task-filter`なしでシステム全体に高い確率（20%超）を設定すると、急速にシステムが不安定になる。安全に対象を絞って注入するには`task-filter=1`と`/proc/self/make-it-fail`を使うこと。グローバルな注入は専用のテストVMにとどめること。

## 特定のプロセスを対象にする {#targeting-specific-processes}

最も粗い注入モード（タスクフィルタなし）は、あらゆるカーネルコンテキストからの割り当てをランダムに失敗させるため、すぐにシステムが不安定になる。ドライバのテストでは、ほぼ常に注入を特定のプロセスに限定したいはずである。

### /proc/self/make-it-fail {#procselfmake-it-fail}

すべてのタスクは`/proc`に`make-it-fail`ファイルを持つ。

```bash
# このシェルと、そこから起動されるすべてのプロセスで注入を有効化する
echo 1 > /proc/self/make-it-fail

# ドライバをロードする（このプロセスとその子だけが影響を受ける）
insmod my_driver.ko

# このシェルで注入を無効化する
echo 0 > /proc/self/make-it-fail
```

これを機能させるには、フォールトインジェクタ側でも`task-filter`を設定しておく必要がある。

```bash
echo 1 > /sys/kernel/debug/failslab/task-filter
echo 1 > /proc/self/make-it-fail
```

`make-it-fail`フラグは子プロセスに継承される（fork/exec）ので、対象のバイナリを起動する前にこれを設定するテストスクリプトは、サブプロセスのツリー全体に失敗を注入することになる。

### 特定のPIDを対象にする {#targeting-a-specific-pid}

```bash
# すでに実行中のプロセスを対象にする
echo 1 > /proc/<pid>/make-it-fail
```

## 呼び出し元によるフィルタリング {#filtering-by-call-site}

`CONFIG_FAULT_INJECTION_STACKTRACE_FILTER=y`を使うと、コールスタック上で特定の関数を通過する割り当てだけに注入を限定できる。これは他のドライバを不安定にせずに1つのドライバをテストするのに強力である。

フィルタは、フォールトインジェクタのディレクトリ下にあるdebugfsファイルで設定する。

```bash
# failslabのスタックトレースフィルタリングを有効化する
# コールスタックに'my_driver_probe'が現れる場合のみ注入する
echo 1 > /sys/kernel/debug/failslab/stacktrace-depth
# （stacktrace-depthは調べるフレーム数を制御する）
```

スタックトレースフィルタは、`should_fail()`が呼ばれた時点のコールスタックを読み取り、いずれかのフレームが設定された**アドレス範囲**内にあるかどうかをチェックすることで動作する（フレームを必須とするには`require-start`/`require-end`、除外するには`reject-start`/`reject-end`を使う）。アドレスは対応するdebugfsファイルに16進数で書き込む。実装については[`lib/fault-inject.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/fault-inject.c)、特に`fail_stacktrace()`を参照。

実際には、スタックフィルタはカーネルツリーで提供されるヘルパースクリプト経由で使われることが最も多い。

## 実践例: ドライバでのkmalloc失敗をテストする {#practical-example-testing-kmalloc-failures-in-a-driver}

以下の手順では、仮想的なドライバがprobe中に`kmalloc`が`NULL`を返した場合をどう処理するかをテストする。

### セットアップ {#setup}

```bash
# 次の設定でテスト用VMを起動する:
# CONFIG_FAILSLAB=y
# CONFIG_FAULT_INJECTION_DEBUG_FS=y
# CONFIG_FAULT_INJECTION_STACKTRACE_FILTER=y

# ステップ1: task-filter付きでfailslabを有効化する（安全 — マークされたタスクにのみ影響）
echo  1 > /sys/kernel/debug/failslab/task-filter
echo 10 > /sys/kernel/debug/failslab/probability
echo -1 > /sys/kernel/debug/failslab/times
echo  2 > /sys/kernel/debug/failslab/verbose   # 失敗のたびにスタックを出力

# ステップ2: このシェルをフォールトインジェクションの対象としてマークする
echo 1 > /proc/self/make-it-fail

# ステップ3: ドライバをロードする（このプロセスからのkmalloc呼び出しは約10%の確率で失敗する）
insmod my_driver.ko

# ステップ4: カーネルログで失敗とエラーを監視する
dmesg -w
```

### dmesgで何を見るべきか {#what-to-look-for-in-dmesg}

正しく書かれたドライバは、エラーをログに出して`-ENOMEM`を返す。

```
my_driver: probe of 0000:01:00.0 failed with error -12
```

バグのあるドライバは次のような挙動を示し得る。

- oopsする（返されたポインタをチェックせずに使ったことによるNULLポインタ参照）
- ハングする（獲得されなかったリソースを待ち続ける）
- 黙って成功するがメモリをリークする（エラーパスが`kfree`を飛ばしている場合）

### 反復 {#iterating}

より多くのパスを動かすために、確率を徐々に上げていく。

```bash
# 控えめに始める
echo 5 > /sys/kernel/debug/failslab/probability

# oopsがないことを確認したら上げる
echo 20 > /sys/kernel/debug/failslab/probability
echo 50 > /sys/kernel/debug/failslab/probability
```

1回のテスト実行中の失敗の総数を制限するには`times`を使う。

```bash
# ちょうど1回だけ失敗させて止める
echo 1 > /sys/kernel/debug/failslab/times
```

これは原因の絞り込みに役立つ。つまり、リグレッションを引き起こす割り当て失敗を正確に特定できる。

## カーネルのヘルパースクリプトを使う {#using-the-kernels-helper-script}

カーネルには、スタックトレースフィルタリング付きの`FAIL_POINT`機構を使って、特定のコマンドに対するフォールトインジェクションをセットアップするヘルパースクリプトが同梱されている。

```bash
# カーネルツリーから:
tools/testing/fault-injection/failcmd.sh

# 例: 確率20%のslabフォールトインジェクション付きでmodprobeを実行する
tools/testing/fault-injection/failcmd.sh --probability=20 -- modprobe my_driver
```

このスクリプトはdebugfsのパラメータをセットアップし、プロセスに`make-it-fail`をマークし、コマンドを実行して、後片付けを行う。debugfsファイルを手作業で管理せずに対象を絞った注入を行うための推奨手段である。カーネルソースの[`tools/testing/fault-injection/`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/testing/fault-injection)を参照。

## fail_make_request: ブロックI/Oの失敗注入 {#failmakerequest-block-io-failure-injection}

`CONFIG_FAIL_MAKE_REQUEST`は、ブロックI/O層（`submit_bio()`のパス）で失敗を注入する。これはファイルシステムやブロックドライバがI/Oエラーをどう処理するかをテストするもので、メモリ割り当ての失敗とは別物である。

```bash
# ブロックI/Oの失敗注入を有効化する
echo 10 > /sys/kernel/debug/fail_make_request/probability
echo -1 > /sys/kernel/debug/fail_make_request/times
echo  1 > /sys/kernel/debug/fail_make_request/verbose

# 特定のデバイスに限定する（major:minorで指定）
echo <major>:<minor> > /sys/block/<device>/make-it-fail
# 例:
echo 1 > /sys/block/sdb/make-it-fail
```

デバイスごとの`make-it-fail`ファイル（`/sys/block/<device>/`内）は注入を特定のブロックデバイスに限定するもので、システム全体に注入するよりもはるかに安全である。ファイルシステムのエラーパスのテストにはこの方法が推奨される。

典型的な用途:

- 書き込みが失敗したときのファイルシステムのジャーナル回復のテスト
- デバイスがエラーを報告したときのRAID再構築ロジックの検証
- データベースのwrite-aheadログがI/O失敗を破損なく処理できることのテスト

## Syzkallerによるフォールトインジェクション {#fault-injection-with-syzkaller}

[Syzkaller](https://github.com/google/syzkaller)は、カーネルの主要なカバレッジ誘導型ファジングツールである。フォールトインジェクションフレームワークと統合されており、自身のシステムコール列が引き起こすエラーパスを自動的に動かす。

Syzkallerは2つの仕組みでフォールトインジェクションを使う。

**1. procfs経由の自動注入**

Syzkallerは、`failslab`と`fail_page_alloc`をグローバルに設定したうえで、自身が起動するプロセスに`/proc/<pid>/make-it-fail`を設定できる。これにより、システムコール列がランダムな箇所で割り当て失敗に遭遇し、処理されていないエラーパスが露呈する。

**2. `KCOV` + フォールトポイントのトレーシングによる対象を絞った注入**

Syzkallerがカバレッジ（`CONFIG_KCOV=y`）によって注目すべきコードパスを特定すると、カバレッジデータに現れる特定の呼び出し元にフォールトインジェクションを向けることができ、コードパス内の各割り当てを順に失敗させてすべてのエラー分岐をチェックする。

syzkallerのVMをフォールトインジェクション用に設定するには次のようにする。

```
# syzkallerの設定（syzkaller.cfg）内:
{
  "enable_syscalls": ["..."],
  "fault_injection": true
}
```

これにより`failslab/task-filter=1`が設定され、フレームワークが自動的に有効化される。

特にメモリ管理をファジングする場合、`CONFIG_KASAN=y` + `CONFIG_FAILSLAB=y` + `CONFIG_FAIL_PAGE_ALLOC=y`の組み合わせにより、通常の割り当てパターンで引き起こされるバグ（KASAN）と、エラーパスを通じてのみ到達可能なバグ（フォールトインジェクション）の両方を捕捉できる。

syzkallerが通常フォールトインジェクションと組み合わせるサニタイザについては[KASAN](kasan.md)と[KFENCE](kfence.md)を参照。

## フォールトインジェクションを使ったストレステスト {#stress-testing-with-fault-injection}

（一度きりのデバッグではなく）リグレッションテストや堅牢性の検証では、アイドル状態で注入するよりも、ストレスワークロードと並行してフォールトインジェクションを実行するほうが効果的である。

### ストレスと注入の組み合わせ {#combined-stress-injection}

```bash
# ターミナル1: slabのフォールトインジェクションを有効化する（task-filterはオフ — すべてに影響）
echo 5  > /sys/kernel/debug/failslab/probability
echo -1 > /sys/kernel/debug/failslab/times
echo  0 > /sys/kernel/debug/failslab/task-filter
echo  1 > /sys/kernel/debug/failslab/verbose

# ターミナル2: 並行してメモリにストレスをかける
stress-ng --vm 4 --vm-bytes 75% --vm-keep --timeout 60s &

# ターミナル3: ドライバのロード／アンロードを繰り返す
for i in $(seq 1 50); do
  insmod my_driver.ko 2>/dev/null
  rmmod my_driver 2>/dev/null
done

# ターミナル4: oopsを監視する
dmesg -w | grep -E "BUG|Oops|WARN|NULL pointer|general protection"
```

### slabキャッシュを対象にした注入 {#targeted-slab-cache-injection}

ドライバが特定の名前付きキャッシュから割り当てる場合は、巻き添えの被害を避けるために注入を限定する。

```bash
# cache-filterモードを有効化する
echo 1 > /sys/kernel/debug/failslab/cache-filter

# 対象のキャッシュに対してのみ注入を有効化する
# （sysfsエントリが存在する、つまりキャッシュがすでに有効である必要がある）
echo 1 > /sys/kernel/slab/my_driver_cache/failslab
```

## カーネルドキュメント {#kernel-documentation}

フォールトインジェクションフレームワークの権威あるリファレンスは次のとおりである。

- [`Documentation/fault-injection/fault-injection.rst`](https://docs.kernel.org/fault-injection/fault-injection.html) — すべてのdebugfsパラメータ、`make-it-fail`インターフェース、スタックトレースフィルタリング、`failcmd.sh`ヘルパーを網羅している

## 主要なソースファイル {#key-source-files}

| ファイル | 説明 |
|------|-------------|
| [`lib/fault-inject.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/fault-inject.c) | 中核のフレームワーク: `should_fail()`、probability/interval/timesのロジック、debugfsのセットアップ |
| [`mm/failslab.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/failslab.c) | `CONFIG_FAILSLAB`: slabアロケータへのフック |
| [`mm/fail_page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/fail_page_alloc.c) | `CONFIG_FAIL_PAGE_ALLOC`: ページアロケータへのフック |
| [`include/linux/fault-inject.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/fault-inject.h) | `fault_attr`構造体と`should_fail()`の宣言 |
| [`tools/testing/fault-injection/failcmd.sh`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/testing/fault-injection/failcmd.sh) | コマンド単位の注入のためのヘルパースクリプト |

## クイックリファレンス {#quick-reference}

```bash
# slabへの注入、タスクフィルタ付き（安全）
echo 1  > /sys/kernel/debug/failslab/task-filter
echo 20 > /sys/kernel/debug/failslab/probability
echo -1 > /sys/kernel/debug/failslab/times
echo 1  > /proc/self/make-it-fail
<run test>
echo 0  > /proc/self/make-it-fail

# ページアロケータへの注入、タスクフィルタ付き
echo 1  > /sys/kernel/debug/fail_page_alloc/task-filter
echo 10 > /sys/kernel/debug/fail_page_alloc/probability
echo -1 > /sys/kernel/debug/fail_page_alloc/times
echo 1  > /proc/self/make-it-fail
<run test>
echo 0  > /proc/self/make-it-fail

# ブロックデバイスI/Oへの注入（特定のデバイス）
echo 1  > /sys/block/sdb/make-it-fail
echo 5  > /sys/kernel/debug/fail_make_request/probability
echo -1 > /sys/kernel/debug/fail_make_request/times
<run filesystem test on sdb>
echo 0  > /sys/block/sdb/make-it-fail

# すべての注入を無効化する
echo 0 > /sys/kernel/debug/failslab/probability
echo 0 > /sys/kernel/debug/fail_page_alloc/probability
echo 0 > /sys/kernel/debug/fail_make_request/probability
```

## 参考文献 {#further-reading}

### カーネルドキュメント {#kernel-documentation_1}

- [Fault injection capabilities infrastructure](https://docs.kernel.org/fault-injection/fault-injection.html) — すべてのパラメータとインターフェースの完全なリファレンス

### 関連項目 {#related}

- [KASAN](kasan.md) — メモリエラーの検出。フォールトインジェクションと組み合わせることで、通常のバグとエラーパスのバグの両方を1回のテスト実行で捕捉できる
- [KFENCE](kfence.md) — 低オーバーヘッドで本番環境向けのメモリ安全性検査。長時間のストレステストでフォールトインジェクションと並行して実行できる
- [OOMのデバッグ](oom-debugging.md) — out-of-memoryによるkillの診断と防止。フォールトインジェクションでOOMに近い状況を意図的に引き起こせる
- [page-poisoning](page-poisoning.md) — use-after-freeの検出。解放済みメモリが不正に再利用されていないことを検証することで、フォールトインジェクションを補完する
