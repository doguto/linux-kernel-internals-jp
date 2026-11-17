# KCSAN: Kernel Concurrency Sanitizer {#kcsan-kernel-concurrency-sanitizer}

> ウォッチポイントとLinux Kernel Memory Modelを用いたデータ競合検出

## データ競合とは {#what-is-a-data-race}

**データ競合（data race）**は、2つのスレッドが同じメモリ位置に並行してアクセスし、少なくとも一方が書き込みであり、両者の間に同期がない場合に発生する:

```c
/* スレッド1: */         /* スレッド2: */
x++;                    y = x;     /* データ競合! */
/* xを読み、インクリメントし、xに書く — 3つの非アトミック操作 */
/* スレッド2は中途半端な状態を見る可能性がある */
```

データ競合はC11では未定義動作であり、微妙なバグの原因になる。コンパイラは競合するコードを予期しない形で誤ってコンパイルしうるし、ハードウェアのCPUはアクセスを並べ替えたり結合したりすることがある。

## KCSANの仕組み {#how-kcsan-works}

KCSANは**ソフトウェアウォッチポイント**を使って競合するアクセスを検出する:

```
Thread 1 write to &x:
  1. KCSAN randomly decides to set a watchpoint on &x
  2. Records &x in a per-CPU software watchpoint slot (no hardware debug registers)
  3. Delays briefly (~microseconds) to give another thread time to access &x

Thread 2 read from &x (during the delay):
  4. Watchpoint fires: both threads' call stacks recorded
  5. KCSAN reports the data race
```

これは確率的なアプローチであり、すべての競合が検出されるわけではないが、（シャドウメモリ方式と比べて）誤検知なしに実際の競合を現実的に見つけられる。

## 設定 {#configuration}

```bash
# Kconfigオプション（CONFIG_KCSAN=yにしたときの実際のデフォルト値。1つを除く）:
CONFIG_KCSAN=y
CONFIG_KCSAN_REPORT_ONCE_IN_MS=3000     # レポートのレート制限（デフォルト）
CONFIG_KCSAN_DELAY_RANDOMIZE=y          # ウォッチポイントの遅延をランダム化（デフォルト）
CONFIG_KCSAN_NUM_WATCHPOINTS=64         # ソフトウェアウォッチポイントのスロット数。全CPUで共有（デフォルト）
CONFIG_KCSAN_INTERRUPT_WATCHER=y        # 割り込みコンテキストも監視 -- デフォルトではない（KCSAN_STRICT=y
                                         # でない限りnがデフォルト）。ここではオプトインの例として示す
CONFIG_KCSAN_REPORT_RACE_UNKNOWN_ORIGIN=y  # 2つ目のスタックがなくても報告する（デフォルト）

# ビルド: .configでCONFIG_KCSAN=yを設定し、通常どおりビルドする
```

## KCSANレポートの読み方 {#reading-kcsan-reports}

```
==================================================================
BUG: KCSAN: data-race in my_function+0x1234/0x5678 [my_module]
            my_other_function+0xabcd/0xef01 [my_module]

read to 0xffffffff12345678 of 4 bytes by task 1234 on cpu 0:
 my_function+0x1234/0x5678
 some_caller+0x0abc/0x0def
 kernel_thread+0x0001/0x0012

write to 0xffffffff12345678 of 4 bytes by task 5678 on cpu 1:
 my_other_function+0xabcd/0xef01
 another_caller+0x0123/0x0456
 kernel_thread+0x0001/0x0012

value changed: 0x00000000 -> 0x00000001
Reported by Kernel Concurrency Sanitizer on:
CPU: 1 PID: 5678 Comm: worker Tainted: G    B W
==================================================================
```

主なフィールド:

- **read/write**: ウォッチポイントを発火させた操作
- **address**: 競合しているメモリ位置
- **value changed**: 競合を引き起こした書き込みを示す
- **2つのコールスタック**: 競合する各アクセスにつき1つ

## アノテーション: 意図的で無害な競合 {#annotation-intentional-benign-races}

カーネルコードの一部は、ヒントや最適化として、同期なしで意図的に変数を読む。KCSANを黙らせるには、これらにアノテーションを付けなければならない:

```c
/* READ_ONCE / WRITE_ONCE: このアクセスが意図的に非同期であることを */
/* コンパイラとKCSANの両方に伝える */
int val = READ_ONCE(shared_var);   /* 読み出しを最適化で消させない */
WRITE_ONCE(shared_var, new_val);   /* 書き込みを最適化で消させない */

/* 意図的なデータ競合（不正確でよいカウンタ、ヒューリスティクス）の場合: */
/* data_race()はアクセスを意図的に競合するものとしてマークする: */
int approximate_count = data_race(shared_counter);

/* __no_kcsan: 関数全体の計装を抑制する */
void __no_kcsan my_racy_function(void)
{
    /* KCSANはこの関数を計装しない */
}
```

## アトミック操作とKCSAN {#atomic-operations-and-kcsan}

アトミック操作は競合ではない:

```c
/* これらは問題ない（アトミックであり、競合ではない）: */
atomic_inc(&counter);              /* アトミックなread-modify-write */
atomic_cmpxchg(&ptr, old, new);    /* compare-and-swap */

/* smp_load_acquire / smp_store_release: acquire/releaseバリア */
/* これらもKCSANには同期されたアクセスとして認識される: */
old = smp_load_acquire(&ptr);
smp_store_release(&ptr, new);
```

## Linux Kernel Memory Model（LKMM） {#linux-kernel-memory-model-lkmm}

KCSANは、並行カーネルコードの形式的なセマンティクスを定義する**Linux Kernel Memory Model**を強制するよう設計されている:

```c
/* LKMMの規則（簡略化）: */

/* 1. プレーンアクセス（x = y）は、他のCPUが同期なしで並行してxに
      書き込まない場合にのみ許される */

/* 2. ACQUIRE/RELEASEバリアは順序付けを提供する:
      lock()/unlock()の対により:
        lock()より前のすべてのアクセスはunlock()より後のすべてのアクセスにhappen-beforeする */

/* 3. smp_mb() / smp_rmb() / smp_wmb()はフルバリアを提供する */

/* 4. READ_ONCE/WRITE_ONCEはコンパイラによる並べ替えを防ぐ */

/* 正しいコードの例: */
static DEFINE_SPINLOCK(lock);
static int shared_var;

void writer(int val)
{
    spin_lock(&lock);
    WRITE_ONCE(shared_var, val);  /* ロック下なら単にshared_var = valでもよい */
    spin_unlock(&lock);
}

int reader(void)
{
    int val;
    spin_lock(&lock);
    val = READ_ONCE(shared_var);
    spin_unlock(&lock);
    return val;
}
```

## KCSANと他のサニタイザの比較 {#kcsan-vs-other-sanitizers}

| | KCSAN | KASAN | ThreadSanitizer |
|---|-------|-------|-----------------|
| 検出対象 | データ競合 | メモリエラー（UAF、OOB） | データ競合 |
| 方式 | ウォッチポイント（確率的） | シャドウメモリ（決定的） | シャドウメモリ |
| オーバーヘッド | 約5%（ウォッチポイント） | メモリ約2倍、CPU約2倍 | CPU約5〜15倍 |
| 誤検知 | ごく少ない | なし | 一部あり |
| 対象 | カーネルのみ | カーネルのみ | ユーザー空間 |

## syzkallerとの統合 {#syzkaller-integration}

syzkaller（カーネルファザー）とKCSANを組み合わせると、新しいシステムコールパスの競合を見つけられる:

```bash
# syzkaller設定でのKCSAN（syzkaller/tools/syz-env）:
"kernel_config": "CONFIG_KCSAN=y,CONFIG_KCSAN_INTERRUPT_WATCHER=y,..."

# syzkallerはシステムコールを並列に実行する複数のVMを起動する
# KCSANのウォッチポイントが並行するシステムコール呼び出し間の競合を検出する
```

## 報告された競合のデバッグ {#debugging-a-reported-race}

```bash
# 1. 競合を取得する:
dmesg | grep -A 30 "KCSAN"

# 2. addr2lineでソースを特定する:
addr2line -e vmlinux -i 0xffffffff12345678
# → mm/slub.c:3456

# 3. git logで最近の変更を確認する:
git log --oneline mm/slub.c | head -10

# 4. より確実に再現する（可能なら）:
# debugfsの遅延ノブは存在しない -- 遅延の調整は書き込み可能な2つの
# モジュールパラメータ kcsan.udelay_task / kcsan.udelay_interrupt（kernel/kcsan/core.c）で行い、
# /sys/module/kcsan/parameters/ 以下に公開されている:
echo 100 > /sys/module/kcsan/parameters/udelay_task  # 最大遅延を100usに引き上げる
# CONFIG_KCSAN_DELAY_RANDOMIZE=y（デフォルト）はアクセスごとにこの値から
# ランダムな量を差し引く。=nにしてリビルドすると遅延は毎回ちょうど
# udelay_taskになり、完全に決定的な再現ができる。

# 5. 修正の選択肢:
# a. 適切なロックを追加する
# b. アトミック操作を使う
# c. 意図的かつ安全ならdata_race()でアノテーションする
# d. 無害な競合にはREAD_ONCE/WRITE_ONCEを使う

# 6. 修正を検証する:
# KCSANを有効にしてsyzkallerまたはストレステストを実行する
# 同じ箇所について新たな競合レポートが出ないこと
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/kcsan/core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/kcsan/core.c) — ウォッチポイントベースの競合検出エンジン
- [kernel/kcsan/report.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/kcsan/report.c) — レポートの整形: 競合する2つのコールスタックと「value changed」行
- [kernel/kcsan/debugfs.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/kcsan/debugfs.c) — `/sys/kernel/debug/kcsan`制御ファイル（オン/オフ、ブラックリスト/ホワイトリスト）
- [lib/Kconfig.kcsan](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/Kconfig.kcsan) — Kconfigオプションとデフォルト値（`KCSAN_NUM_WATCHPOINTS`のデフォルトは64、`KCSAN_REPORT_ONCE_IN_MS`のデフォルトは3000、`KCSAN_DELAY_RANDOMIZE`のデフォルトはy）
- [include/linux/compiler.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/compiler.h) — `data_race()`マクロの定義
- [include/linux/compiler_types.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/compiler_types.h) — `__no_kcsan`関数属性の定義
- [Documentation/dev-tools/kcsan.rst](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/dev-tools/kcsan.rst) — 公式のKCSANドキュメント
- [tools/memory-model/](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/memory-model/) — Linux Kernel Memory Model（LKMM）の形式仕様

### 関連ページ {#related-pages}

- [KASAN](../mm/kasan.md) — メモリエラー検出（use-after-free、OOB）
- [KFENCE](../mm/kfence.md) — 軽量なサンプリング型メモリエラー検出器
- [アトミック操作とメモリバリア](../locking/atomics.md) — 正しい並行アクセス
- [RCU](../locking/rcu.md) — RCUは競合のない、読み取り中心のパターンである
- [spinlock](../locking/spinlock.md) — spinlockは競合を防ぐ

### LWN記事 {#lwn-articles}

- [Finding race conditions with KCSAN](https://lwn.net/Articles/802128/) — 発表直後のJonathan CorbetによるKCSANの紹介（2019年10月。最終的に5.8でマージされる前）
- [Concurrency bugs should fear the big bad data-race detector](https://lwn.net/Articles/816850/) — KCSANとLinux Kernel Memory Modelの開発者が共同で執筆した、2つのプロジェクトがどう噛み合うかの解説（2020年4月）

### 外部資料 {#external}

- [docs.kernel.org: Kernel Concurrency Sanitizer (KCSAN)](https://docs.kernel.org/dev-tools/kcsan.html) — レンダリングされた公式ドキュメント。debugfsインターフェースやチューニング用パラメータを含む
