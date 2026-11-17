# seqlock {#seqlock}

> 頻繁に更新される小さなデータをロックなしで読むためのシーケンスカウンタ

## seqlockのパターン {#the-seqlock-pattern}

seqlock（シーケンスロック）は特定の問題を解決する。読み手が、ロックを一切取らずに小さなデータ（タイムスタンプやカウンタなど）にアクセスしつつ、並行する書き込みが起きたかどうかを検出できるようにするにはどうすればよいか、という問題である。

書き手は、書き込みの前後でシーケンスカウンタをインクリメントする。読み手は、データを読む前後でカウンタを読む。カウンタが変化していた（または奇数で、書き込み中であることを意味する）場合、読み手はリトライする。

```
Writer:
  seq++      (now odd → write in progress)
  write data
  seq++      (now even → write complete)

Reader:
  s1 = seq
  if (s1 & 1) retry    ← write in progress
  read data
  s2 = seq
  if (s1 != s2) retry  ← write happened during read
  data is valid
```

これにより、**並行する書き込みがない場合の読み込みはコストゼロ**になり、そうでない場合も安価なリトライで済む。

## 2つの型: seqcount_tとseqlock_t {#two-types-seqcount_t-and-seqlock_t}

```c
/* seqcount_t: 生のカウンタ。書き手の直列化は組み込まれていない。
   書き手用のロックは自分で用意する。 */
typedef struct seqcount {
    unsigned sequence;
} seqcount_t;

/* seqlock_t: seqcount + 書き手直列化用の組み込みspinlock */
typedef struct {
    struct seqcount_spinlock seqcount;
    spinlock_t lock;
} seqlock_t;
```

ほとんどのユーザーにとっては`seqlock_t`が完全なパッケージである。`seqcount_t`は、書き手の直列化が何か別の仕組み（例: mutex）で扱われる場合に使われる。

## API {#api}

### seqlock_t {#seqlock_t}

```c
#include <linux/seqlock.h>

/* 初期化 */
DEFINE_SEQLOCK(my_seqlock);    /* 静的 */
seqlock_t my_seqlock;
seqlock_init(&my_seqlock);     /* 動的 */

/* 書き手側 */
write_seqlock(&my_seqlock);
/* データを変更する */
write_sequnlock(&my_seqlock);

/* 書き手 + IRQ無効化 */
write_seqlock_irqsave(&my_seqlock, flags);
write_sequnlock_irqrestore(&my_seqlock, flags);

/* 読み手側: リトライループ */
unsigned seq;
int value;
do {
    seq = read_seqbegin(&my_seqlock);
    value = protected_value;
} while (read_seqretry(&my_seqlock, seq));
/* これでvalueは一貫している */
```

### seqcount_t {#seqcount_t}

```c
#include <linux/seqlock.h>

seqcount_t my_seqcount = SEQCNT_ZERO(my_seqcount);
/* または動的に: seqcount_init(&my_seqcount); */

/* 書き手: 直列化は別途行う */
spin_lock(&my_lock);
write_seqcount_begin(&my_seqcount);
/* データを変更する */
write_seqcount_end(&my_seqcount);
spin_unlock(&my_lock);

/* 読み手 */
unsigned seq;
do {
    seq = read_seqcount_begin(&my_seqcount);
    /* データを読む */
} while (read_seqcount_retry(&my_seqcount, seq));
```

## 実際の例: jiffiesと時間 {#real-world-example-jiffies-and-time}

カーネルは、タイマーティックごとに更新されるが多くの場所から読まれる時間関連のデータに、seqlockを広く使っている。

```c
/* kernel/time/timekeeping.c */
/* 現在時刻を読む */
static inline u64 __ktime_get_fast_ns(void)
{
    struct tk_read_base *tkr;
    unsigned int seq;
    u64 now;

    do {
        seq = raw_read_seqcount_latch(&tk_core.seq);
        tkr = tk_core.tkr_mono + (seq & 0x01);
        now = tkr->base + clocksource_delta(tkr->clock, tkr->mask, tkr->mult, tkr->shift);
    } while (raw_read_seqcount_latch_retry(&tk_core.seq, seq));

    return now;
}
```

`jiffies_64`カウンタも、32ビットシステム（64ビットの読み込みがアトミックでない）ではseqlockで保護されている。

```c
/* include/linux/jiffies.h */
extern seqlock_t jiffies_lock;
extern u64 jiffies_64;

static inline u64 get_jiffies_64(void)
{
    unsigned long seq;
    u64 ret;
    do {
        seq = read_seqbegin(&jiffies_lock);
        ret = jiffies_64;
    } while (read_seqretry(&jiffies_lock, seq));
    return ret;
}
```

## 制約 {#constraints}

seqlockには重要な制限がある。

1. **ポインタは不可**: 読み手はクリティカルセクションの間にポインタをたどってはならない。読み手がポインタを参照外ししてから使うまでの間に書き手が古いオブジェクトを解放すると、読み手はダングリングポインタを持つことになる。ポインタベースのデータには代わりにRCUを使う。

2. **小さく、上限のあるデータ**: リトライループは安価で、必ず終了しなければならない。メガバイト単位のデータや、可変長の読み込みを伴うデータを保護してはならない。

3. **書き手の進行**: 書き込みは素早く完了する。書き込みが頻繁または長いと、読み手はリトライループでスピンし続けることになる。

4. **読み手でのブロック不可**: 読み手のクリティカルセクションはスリープしてはならない（スリープ中にデータが変わりうる）。

```
seqlockを使うべき場合:
  ✓ データが小さく固定サイズである（整数、タイムスタンプ）
  ✓ 読み込みが書き込みよりはるかに頻繁である
  ✓ 書き手が素早く、まれである
  ✓ 保護対象のデータにポインタがない

RCUを使うべき場合:
  ✓ データがポインタベースである
  ✓ 読み込みが複雑な構造の一貫したスナップショットを見なければならない
```

## seqlockとrwlockの比較 {#seqlock-vs-rwlock}

どちらも読み手の並行実行を許すが、次の違いがある。

| | rwlock | seqlock |
|---|--------|---------|
| 読み手のオーバーヘッド | アトミック操作（取得） | 非アトミックな読み込み + バリア |
| 書き手が読み手をブロックするか | する | しない（読み手がリトライ） |
| 読み手の飢餓 | なし | ありうる（忙しい書き手がリトライを強いる） |
| ポインタデータ | 可 | 不可 |
| IRQセーフ | はい | はい |

読み込み中心で書き込みがまれな、ポインタを含まないデータのワークロードではseqlockが勝る。

## 参考文献 {#further-reading}

- [rwlockとrwsem](rwlock-rwsem.md) — ポインタベースのデータや複雑な構造向け
- [RCU](rcu.md) — ポインタベースのデータのロックフリーな読み込み
- カーネルツリー内の`Documentation/locking/seqlock.rst`
