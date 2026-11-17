# ロックと同期 {#locking-and-synchronization}

> Linuxカーネルが共有データへの並行アクセスをどのように直列化するか

## なぜロックが重要か {#why-locking-matters}

Linuxは数百個のCPUを持つシステム上で動作し、それらのCPUはすべてカーネルのデータ構造を共有している。同期がなければ、2つのCPUが同時にリンクリストを変更し、ポインタを破壊したり、エントリを失ったり、無限ループに陥ったりしうる。ロックはこれを防ぐ。

カーネルは豊富なプリミティブを提供しており、それぞれ性能特性と使用コンテキストが異なる。

| プリミティブ | スリープするか | IRQセーフか | 使う場面 |
|-----------|---------|-----------|----------|
| `raw_spinlock_t` | しない | はい | スリープしてはならない、IRQコンテキスト |
| `spinlock_t` | しない（RTではする） | いいえ | プロセスコンテキスト、短いクリティカルセクション |
| `mutex_t` | する | いいえ | プロセスコンテキスト、長めのクリティカルセクション |
| `rw_semaphore` | する | いいえ | 読み手が多く書き手が少ない |
| `rwlock_t` | しない | はい | 読み手が多い、IRQコンテキスト |
| RCU | しない（読み手） | はい（読み手） | 読み込み中心、ポインタベースのデータ |
| `seqlock_t` | しない | はい | 小さなデータ、書き手を飢餓状態にしてはならない |

## 目次 {#contents}

### 基礎 {#fundamentals}
- [spinlockとraw_spinlock](spinlock.md) — アトミックな基本構成要素
- [mutexとrt_mutex](mutex.md) — プロセスコンテキスト用のスリープするロック
- [rwlockとrwsem](rwlock-rwsem.md) — reader-writerロック
- [RCU（Read-Copy-Update）](rcu.md) — グレースピリオドによるロックフリーな読み込み
- [seqlock](seqlock.md) — 頻繁に読まれる小さなデータのためのシーケンスカウンタ

### 応用 {#advanced}
- [アトミック操作とメモリバリア](atomics.md) — プリミティブの下にあるプリミティブ
- [per-CPU変数](percpu.md) — 共有状態そのものを排除する
- [lockdep](lockdep.md) — カーネルのロック検証器
- [ロック競合のデバッグ](lock-debugging.md) — ロックのホットスポットを診断する
- [futexの内部](futex.md) — ユーザー空間のロックがどう動くか

### 事件簿 {#war-stories}
- [事件簿](war-stories.md) — rt_mutexとPI-futexにまつわる3つのCVEと1つの起源の物語

## 階層: 何の内側に何をネストできるか {#the-hierarchy-what-can-nest-inside-what}

```
Hardware interrupts (NMI)
    └── can only use: raw_spinlock (irqsave), atomic ops
Hardware interrupts (IRQ)
    └── can use: raw_spinlock (irqsave), rwlock (irqsave), seqlock
Softirqs / tasklets
    └── can use: spinlock (bh), rwlock (bh), RCU read side
Process context (preemptible)
    └── can use: everything, including mutex, semaphore, RCU
```

割り込みコンテキストからスリープするロックを呼んではならない。カーネルは警告を出すか、デッドロックする。
