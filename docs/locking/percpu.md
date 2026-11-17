# per-CPU変数 {#per-cpu-variables}

> 各CPUに専用のコピーを与えることで共有状態を排除する

## 基本的な考え方 {#the-core-idea}

多くのカーネルサブシステムは、CPUごとのカウンタや状態を保持している。コンテキストスイッチの回数、キャッシュの統計、ランキューへのポインタなどである。素直な方法はロック付きのグローバル変数だが、per-CPUのアプローチのほうが優れている。各CPUに専用のコピーを与えるのである。

利点:

- **CPUローカルなアクセスに同期が不要**: 自分のCPUのコピーの読み書きにはロックが要らない
- **キャッシュラインのバウンスがない**: 各CPUは自分のデータにしか触れないので、false sharingが起きない
- **自然にスケールする**: CPU数が増えても性能が劣化しない

トレードオフ: システム全体の合計を読むには、全CPUのコピーを合計する必要がある。

## 仕組み {#how-it-works}

カーネルは起動時に、CPUごとに連続したper-CPU領域を割り当てる。per-CPU変数はこの領域内のオフセットである。CPU Nのコピーにアクセスするには、`base[N] + offset`を計算する。

```
Per-CPU memory layout:

CPU 0 area:  [var_a] [var_b] [counter] [...]
CPU 1 area:  [var_a] [var_b] [counter] [...]
CPU 2 area:  [var_a] [var_b] [counter] [...]
                      ↑
                      same offset, different base address per CPU
```

## per-CPU変数の宣言と使用 {#declaring-and-using-per-cpu-variables}

```c
#include <linux/percpu.h>
#include <linux/percpu-defs.h>

/* per-CPU変数を宣言する（CPUごとに1つのコピー） */
DEFINE_PER_CPU(int, my_counter);
DEFINE_PER_CPU(struct my_stats, cpu_stats);

/* ヘッダでの宣言: */
DECLARE_PER_CPU(int, my_counter);

/* 自分のCPUのコピーにアクセスする */
/* get_cpu_var: プリエンプションを無効化し、参照を返す */
int val = get_cpu_var(my_counter);
get_cpu_var(my_counter)++;
put_cpu_var(my_counter);  /* プリエンプションを再有効化する */

/* this_cpu_*: 低オーバーヘッド版。マイグレーションしないことを呼び出し側が保証する必要がある */
this_cpu_inc(my_counter);
this_cpu_add(my_counter, 5);
int v = this_cpu_read(my_counter);
this_cpu_write(my_counter, 0);
```

`get_cpu_var`と`this_cpu_*`の違い:

- `get_cpu_var`はプリエンプションを無効化する。コードがプリエンプトされうる場合でも安全である
- `this_cpu_*`はプリエンプションを無効化*しない*。呼び出し側がマイグレーションしないことを保証しなければならない（例: すでにプリエンプション無効区間にいる、またはper-CPUコンテキストにいる）

## 他のCPUのデータへのアクセス {#accessing-another-cpus-data}

```c
/* CPU Nのコピーを読む（観測のみ。読み込みにロックは不要） */
int val = per_cpu(my_counter, cpu_id);

/* CPU Nのコピーへのポインタを得る */
int *ptr = per_cpu_ptr(&my_counter, cpu_id);
```

他のCPUの変数に書き込むには、通常、ロックを使うか、IPI（プロセッサ間割り込み）を使ってそのCPU上でコードを実行する必要がある。

## 全CPUの合計 {#summing-all-cpus}

```c
/* per-CPUの整数を全CPUにわたって合計する */
long total = 0;
int cpu;
for_each_possible_cpu(cpu)
    total += per_cpu(my_counter, cpu);

/* あるいはpercpu_counter用の便利なヘルパを使う: */
s64 sum = percpu_counter_sum(&my_percpu_counter);
```

## percpu_counter: 既製の解決策 {#percpu_counter-the-ready-made-solution}

単純なカウンタには、`percpu_counter`が、更新をまとめて中央のカウンタに反映する完全な実装を提供している。

```c
#include <linux/percpu_counter.h>

struct percpu_counter my_counter;
percpu_counter_init(&my_counter, 0, GFP_KERNEL);

/* 高速: このCPUのローカルカウントを更新する */
percpu_counter_add(&my_counter, 1);
percpu_counter_inc(&my_counter);
percpu_counter_dec(&my_counter);

/* 近似: 高速な合計（多少ずれることがある） */
s64 approx = percpu_counter_read(&my_counter);

/* 正確: 全CPUを合計する（遅い） */
s64 exact = percpu_counter_sum(&my_counter);

percpu_counter_destroy(&my_counter);
```

`percpu_counter`は更新をバッチ化する。各CPUは最大`batch`（デフォルト32）個のローカルな変更を蓄積してから、グローバルカウンタにフラッシュする。これにより、インクリメントは高速パスでアトミック操作なしのO(1)になる。

## per-CPU変数を使うべきでない場合 {#when-not-to-use-per-cpu-variables}

per-CPUが常に答えになるわけではない。

```
per-CPUを使うべき場合:
  ✓ データが主に所有CPUによって更新される
  ✓ グローバルな集計値が近似でよい（統計、カウンタ）
  ✓ CPUごとの状態が意味的に妥当である（ランキュー、CPUローカルなキャッシュ）

per-CPUを使うべきでない場合:
  ✗ どのCPUも同じ頻度でデータを更新しうる
  ✗ グローバルな合計が常に正確で一貫していなければならない
  ✗ データが本質的にグローバルである（単一のフラグ、単一のポインタ）
```

## 実際の使用例: VM統計 {#real-world-usage-vm-stats}

mmサブシステムは、ページカウンタにper-CPU変数を使っている。

```c
/* mm/vmstat.c */
DEFINE_PER_CPU(struct vm_event_state, vm_event_states);

/* VMイベントカウンタ（例: ページフォールトの回数）をインクリメントする */
static inline void count_vm_event(enum vm_event_item item)
{
    this_cpu_inc(vm_event_states.event[item]);
}

/* グローバルな合計を読む */
unsigned long global_node_page_state(enum node_stat_item item)
{
    long x = atomic_long_read(&vm_node_stat[item]);
    return x;
}
```

そしてスケジューラはper-CPUのランキューを使っている。

```c
/* kernel/sched/core.c */
DEFINE_PER_CPU_SHARED_ALIGNED(struct rq, runqueues);

#define cpu_rq(cpu) (&per_cpu(runqueues, (cpu)))
#define this_rq()   this_cpu_ptr(&runqueues)
```

## 参考文献 {#further-reading}

- [アトミック操作](atomics.md) — 同期が本当に必要なとき
- [spinlock](spinlock.md) — 真に共有されたデータを保護する
- カーネルツリー内の`Documentation/core-api/per-cpu.rst`
