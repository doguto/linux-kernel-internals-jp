# ARM64のメモリモデル {#arm64-memory-model}

> ARM64における弱い順序付け、バリア、アトミック操作

## ARM64は弱い順序付けである {#arm64-is-weakly-ordered}

x86（TSO — Total Store Order を持ち、ほぼ逐次一貫）とは異なり、ARM64は**弱い順序付け**のメモリモデルを持つ。ハードウェアは性能のためにロードとストアを並べ替えることがある。

```
// ARM64: これらの操作はハードウェアによって並べ替えられる可能性がある:
STR x1, [x2]     // アドレスx2へストア
LDR x3, [x4]     // アドレスx4からロード
// CPUはストアより前にロードを実行するかもしれない
```

これは、ロックフリーアルゴリズムや、明示的なアトミック操作を使わずにスレッド間で同期を取るあらゆるコードにとって重要である。

## ARM64が保証すること {#what-arm64-guarantees}

1. **シングルコピーアトミック性**: 自然にアラインされた単一レジスタのロードまたはストアはアトミックである（値は完全に書き込まれるか、まったく書き込まれないかのどちらかである）。

2. **依存するアクセスのプログラム順序**: ロードAがロードBのアドレス計算に使われる場合、BはAの効果を観測する（データ依存の順序付け）。

3. **独立したアクセスには保証がない**: 無関係なアドレスへのストアとロードは、他のCPUから任意の順序で観測されうる。

## メモリバリア命令 {#memory-barrier-instructions}

ARM64は3つのバリア命令を提供する。

### DMB: Data Memory Barrier {#dmb-data-memory-barrier}

バリアより前のメモリアクセスが、バリアより後のアクセスより先に可視になることを保証する。

```asm
DMB SY      /* システム全体、全種類: LDの後のLD、STの後のLD、LDの後のST、STの後のST */
DMB ST      /* STの後のSTのみ */
DMB LD      /* LDの後のLDとST */
DMB ISH     /* inner shareable: 同じ共有可能ドメイン内の全CPUに影響 */
DMB ISHST   /* inner shareable、ストアのみ */
DMB ISHLD   /* inner shareable、ロードのみ */
DMB NSH     /* non-shareable: このPEのみ */
DMB OSH     /* outer shareable */
```

`SY`（システム全体）が最も保守的である。SMPカーネルでは`ISH`が一般的である。

### DSB: Data Synchronization Barrier {#dsb-data-synchronization-barrier}

DMBより強い。メモリアクセスが順序付けられるだけでなく、先行するすべてのメモリアクセスが完了するまで**バリアがパイプラインをストールさせる**。

```asm
DSB SY      /* すべてのメモリアクセスの完了を待つ、システム全体 */
DSB ISH     /* 同上だがinner shareableのみ */
DSB ST      /* すべてのストアの完了を待つ */
```

メモリアクセスが（単に順序付けられたのではなく）*完了した*ことを知る必要があるときにDSBを使う。たとえば、MMUがウォーク中かもしれないページテーブルを変更する前や、キャッシュメンテナンス操作を有効にする前である。

### ISB: Instruction Synchronization Barrier {#isb-instruction-synchronization-barrier}

命令パイプラインをフラッシュする。

```asm
ISB
```

ISBは次の後に使う。

- 命令実行に影響するシステムレジスタ（例: SCTLR_EL1）への書き込み
- 自己書き換えコード
- 命令キャッシュ属性の変更

ISBがないと、CPUはシステムレジスタへの書き込みより後の命令をすでにフェッチ・デコードしているかもしれない。

## Load-acquireとStore-release {#load-acquire-and-store-release}

ARM64は順序付けのセマンティクスを含む特別なアトミックロード/ストアの変種を提供する。

```asm
LDAR x0, [x1]    /* Load-Acquire: これより後のロードやストアは
                    これより前に並べ替えられない */

STLR x0, [x1]    /* Store-Release: これより前のロードやストアは
                    これより後に並べ替えられない */

LDAPR x0, [x1]   /* Load-Acquire RCpc: より弱いacquire — 異なるアドレスへの
                    先行するSTLRとの並べ替えを許す（LDARとは異なる） */
```

これらはC11/C++11の`memory_order_acquire`と`memory_order_release`のセマンティクスを効率的に実装する。明示的なDMBは不要である。

```c
/* LDAR/STLRに相当するC */
/* LDAR x0, [x1] == */  atomic_load_explicit(ptr, memory_order_acquire);
/* STLR x0, [x1] == */  atomic_store_explicit(ptr, val, memory_order_release);
```

## 排他的ロード/ストア（アトミック） {#exclusive-loadstore-atomics}

ARM64のアトミック操作はload-link/store-conditional（LL/SC）機構を使う。

```asm
/* compare-and-swapループ: [addr]のold_val → new_val */
.loop:
    LDXR  w0, [x1]        /* Load eXclusive: addrを排他としてマーク */
    CMP   w0, w3           /* 期待値と比較 */
    B.NE  .fail
    STXR  w4, w2, [x1]    /* Store eXclusive: 新しい値を書き込む */
    CBNZ  w4, .loop        /* ストアが失敗した（排他を失った）らリトライ */
    /* 成功 */
.fail:
    /* w0には実際の値が入っている */
```

ARMv8.1では**Large System Extensions（LSE）**が追加された。これはリトライループを避ける単一命令のアトミック操作である。

```asm
/* ARMv8.1 LSE: アトミックなcompare-and-swap */
CAS  w0, w2, [x1]    /* [x1]==w0ならw2を書き込む。w0 = 古い値 */
CASAL w0, w2, [x1]   /* acquire-releaseセマンティクス付きのCAS */

/* アトミック加算 */
LDADD x0, x1, [x2]  /* [x2] += x0; x1 = 古い[x2] */
STADD x0, [x1]       /* [x1] += x0; 戻り値なし */

/* アトミックスワップ */
SWP x0, x1, [x2]    /* [x2] = x0; x1 = 古い[x2] */
```

多数のコアを持つシステムではLSEアトミックが好まれる。潜在的に多くのリトライループを回す代わりに、単一のバストランザクションを生成するからである。

## ARM64におけるLinuxカーネルのバリアマクロ {#linux-kernel-barrier-macros-on-arm64}

カーネルは、ARM64命令に対応付けられるアーキテクチャ非依存のマクロを提供する。

```c
/* include/asm-generic/barrier.h（asm/barrier.hでarch/arm64に対応付けられる） */

smp_mb()    /* フルメモリバリア:   DMB ISH  */
smp_rmb()   /* 読み込みメモリバリア:   DMB ISHLD */
smp_wmb()   /* 書き込みメモリバリア:  DMB ISHST */

smp_store_release(p, v)   /* STLR: ストア + release */
smp_load_acquire(p)        /* LDAR: ロード + acquire */

/* データ依存バリア（より弱い — データ依存の順序付けに依拠） */
smp_read_barrier_depends()  /* ARM64ではno-op: データ依存の順序付けが保証されている */

/* コンパイラバリア（コンパイラによる並べ替えを防ぐ、CPUバリアなし） */
barrier()   /* asm volatile("" ::: "memory") */
READ_ONCE(x)    /* ロード、コンパイラ最適化を防ぐ */
WRITE_ONCE(x,v) /* ストア、コンパイラ最適化を防ぐ */
```

### どれをいつ使うか {#when-to-use-which}

```c
/* ロック: acquireセマンティクス — ロックより後のものはロックより前に移動できない */
spin_lock() → smp_load_acquire or DMB ISH + LDAR

/* アンロック: releaseセマンティクス — アンロックより前のものはアンロックより後に移動できない */
spin_unlock() → STLR or DMB ISH + STR

/* RCUの読み込み側: ARM64ではデータ依存で十分 */
rcu_dereference(p)  /* READ_ONCE + smp_read_barrier_depends（ARM64ではnoop） */

/* プロデューサ/コンシューマのロックレスキュー */
/* プロデューサ: */
WRITE_ONCE(item->data, value);
smp_wmb();                     /* インデックスより先にデータを可視にする */
WRITE_ONCE(queue->tail, new_tail);

/* コンシューマ: */
tail = smp_load_acquire(&queue->tail);  /* acquire付きでtailをロード */
data = READ_ONCE(item->data);           /* acquireロードの後ではデータが可視 */
```

## ページテーブルのバリア {#page-table-barriers}

ARM64固有のよくある要件として、ページテーブルエントリを変更した後は、TLB無効化命令の前にそのストアがMMUから可視であることを保証しなければならない。

```asm
/* 新しいPTEを書き込む */
STR x0, [x1]          /* 新しいページテーブルエントリをストア */
DSB ISH                /* ストアの完了を待つ */
/* TLB無効化 */
TLBI VAE1IS, x2        /* TLBエントリを無効化、inner shareable */
DSB ISH                /* TLB無効化の完了を待つ */
ISB                    /* 以降の命令が新しいマッピングを使うことを保証 */
```

カーネルでは次のようになる。

```c
/* arch/arm64/include/asm/tlbflush.h */
static inline void flush_tlb_page(struct vm_area_struct *vma,
                                    unsigned long uaddr)
{
    unsigned long addr = __TLBI_VADDR(uaddr, ASID(vma->vm_mm));
    dsb(ishst);             /* DSB ISH ST — ページテーブルへのストアを待つ */
    __tlbi(vale1is, addr);  /* アドレス指定のTLBI、inner shareable */
    dsb(ish);               /* DSB ISH — TLB無効化を待つ */
}
```

## カーネルにおけるAcquire/Release: spinlock {#acquirerelease-in-the-kernel-spinlock}

```asm
/* arch/arm64/include/asm/spinlock.h */
/* チケットspinlockの獲得: */
arch_spin_lock:
    PRFM    PSTL1KEEP, [x0]   /* 書き込み用にプリフェッチ */
.again:
    LDAXR   w2, [x0]          /* load-exclusive + acquire */
    /* 自分のチケット == 処理中番号かを確認 */
    ...
    CBNZ    w3, .wait
    STXR    w3, w1, [x0]      /* store-exclusive */
    CBNZ    w3, .again
    RET

/* spinlockの解放: ストアするだけ、store-releaseの順序付け */
arch_spin_unlock:
    STLR    w1, [x0]          /* store-release: 先行するすべてのアクセスが可視 */
    RET
```

## 弱いメモリモデルのバグ {#weak-memory-model-bugs}

x86（TSO）では動くが**ARM64では失敗する**よくあるパターン:

### プロデューサ/コンシューマでのバリア欠落 {#missing-barrier-in-producerconsumer}

```c
/* ARM64では誤り: */
struct item {
    int ready;    /* オフセット0 */
    int data;     /* オフセット4 */
};

/* スレッドA（プロデューサ）: */
item->data  = 42;
item->ready = 1;    /* ARM64は並べ替えうる: data=42より先にready=1が可視になる */

/* スレッドB（コンシューマ）: */
while (!item->ready);   /* スピン */
use(item->data);        /* バリアがないのでdata=0が見えるかもしれない! */

/* 正しい: */
WRITE_ONCE(item->data, 42);
smp_wmb();              /* DMB ISHST: readyより先にdataを可視にする */
WRITE_ONCE(item->ready, 1);

/* コンシューマ: */
while (!smp_load_acquire(&item->ready)); /* LDAR: acquireの順序付け */
use(READ_ONCE(item->data));
```

### ページテーブル更新後のdsb欠落 {#missing-dsb-after-page-table-update}

```c
/* 誤り: set_pteの後、CPUが古いページテーブルのウォークを始めるかもしれない */
set_pte(ptep, new_pte);
flush_tlb_page(vma, addr);  /* DSBなしのTLBフラッシュ — MMUが新しいPTEを見ないかもしれない */

/* 正しい（カーネルが自動的に行う）: */
set_pte(ptep, new_pte);
dsb(ishst);                 /* PTEのストアを可視にする */
flush_tlb_page(vma, addr);
```

## メモリ順序付けの問題を観察する {#observing-memory-ordering-issues}

```bash
# litmusテストを実行する（メモリモデルの検証）
# ARMは実ハードウェアの挙動をテストするlitmus7ツールを提供している

# LKMM（Linux Kernel Memory Model）による検証
# https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/memory-model

# コンパイル時の競合検出のためにCONFIG_KCSANを有効化
# KCSANはメモリアクセスを計装し、データ競合をチェックする

# lockdep + KCSANで問題を検出
dmesg | grep "data-race"
```

## 参考文献 {#further-reading}

- [例外モデル](exception-model.md) — ARM64の例外レベルとGIC
- [ロック: アトミック操作とメモリバリア](../../locking/atomics.md) — カーネルのアトミックAPI
- [ロック: RCU](../../locking/rcu.md) — RCUがバリアをどう使うか
- [メモリ管理: ページテーブル](../../mm/page-tables.md) — ページテーブルのバリア要件
- ARM Architecture Reference Manual — B2: The AArch64 Application Level Memory Model
- [Documentation/memory-barriers.txt](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/memory-barriers.txt) — カーネルのメモリ順序付けのバイブル
