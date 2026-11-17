# perfイベント {#perf-events}

> ハードウェア性能カウンタ、サンプリング、プロファイリング

## perfイベントとは {#what-perf-events-are}

perfイベントサブシステムはIngo MolnárとPeter Zijlstraによって作られ、Linux 2.6.31で導入された[（LWN）](https://lwn.net/Articles/337493/)。`perf_event_open`は、性能監視ハードウェアにアクセスするためのLinuxのシステムコールである。

- **ハードウェアカウンタ**（PMU）: CPUサイクル、命令数、キャッシュミス、分岐予測ミス
- **ソフトウェアカウンタ**: ページフォールト、コンテキストスイッチ、CPUマイグレーション
- **tracepoint**: カーネルの静的イベント
- **動的イベント**: kprobe、uprobe、BPF
- **サンプリング**: 統計的プロファイリングのため、NイベントごとにPC／スタックを記録する

## perf_event_openシステムコール {#perf_event_open-syscall}

```c
#include <linux/perf_event.h>

struct perf_event_attr attr = {
    .type           = PERF_TYPE_HARDWARE,
    .config         = PERF_COUNT_HW_CPU_CYCLES,  /* CPU サイクルを数える */
    .size           = sizeof(attr),
    .disabled       = 1,    /* 無効状態で開始する */
    .exclude_kernel = 0,    /* カーネルのイベントも数える */
    .exclude_hv     = 1,    /* ハイパーバイザを除外する */
};

/* 任意の CPU 上で現在のプロセスを監視する */
int fd = syscall(SYS_perf_event_open, &attr,
                 0,    /* pid: 0=現在のプロセス */
                 -1,   /* cpu: -1=任意 */
                 -1,   /* group_fd: -1=グループなし */
                 0);   /* フラグ */

/* カウントを開始する */
ioctl(fd, PERF_EVENT_IOC_ENABLE, 0);

/* ... 計測対象のコード ... */

/* カウンタを読む */
long long count;
read(fd, &count, sizeof(count));
printf("CPU cycles: %lld\n", count);

/* 停止する */
ioctl(fd, PERF_EVENT_IOC_DISABLE, 0);
close(fd);
```

## イベントの種類 {#event-types}

```c
/* ハードウェアイベント（CPU PMU） */
.type = PERF_TYPE_HARDWARE
.config:
    PERF_COUNT_HW_CPU_CYCLES         /* CPU サイクル */
    PERF_COUNT_HW_INSTRUCTIONS       /* リタイアした命令 */
    PERF_COUNT_HW_CACHE_REFERENCES   /* キャッシュアクセス */
    PERF_COUNT_HW_CACHE_MISSES       /* キャッシュミス */
    PERF_COUNT_HW_BRANCH_INSTRUCTIONS /* 分岐命令 */
    PERF_COUNT_HW_BRANCH_MISSES      /* 分岐予測ミス */
    PERF_COUNT_HW_BUS_CYCLES
    PERF_COUNT_HW_STALLED_CYCLES_FRONTEND
    PERF_COUNT_HW_STALLED_CYCLES_BACKEND

/* ソフトウェアイベント（カーネルのカウンタ） */
.type = PERF_TYPE_SOFTWARE
.config:
    PERF_COUNT_SW_CPU_CLOCK          /* CPU クロック（ns） */
    PERF_COUNT_SW_TASK_CLOCK         /* タスクローカルなクロック */
    PERF_COUNT_SW_PAGE_FAULTS        /* ページフォールト */
    PERF_COUNT_SW_CONTEXT_SWITCHES   /* 自発的 + 非自発的 */
    PERF_COUNT_SW_CPU_MIGRATIONS     /* タスクが CPU 間を移動した */
    PERF_COUNT_SW_PAGE_FAULTS_MIN    /* マイナーフォールト（ディスク I/O なし） */
    PERF_COUNT_SW_PAGE_FAULTS_MAJ    /* メジャーフォールト（ディスク I/O あり） */
    PERF_COUNT_SW_ALIGNMENT_FAULTS
    PERF_COUNT_SW_EMULATION_FAULTS
    PERF_COUNT_SW_DUMMY

/* ハードウェアキャッシュイベント */
.type = PERF_TYPE_HW_CACHE
/* .config のエンコード: cache_id | (cache_op_id << 8) | (result_id << 16) */
/* 例: L1D の読み出しミス: */
.config = PERF_COUNT_HW_CACHE_L1D | (PERF_COUNT_HW_CACHE_OP_READ << 8) |
          (PERF_COUNT_HW_CACHE_RESULT_MISS << 16)

/* tracepoint イベント */
.type = PERF_TYPE_TRACEPOINT
.config = <tracepoint_id>  /* /sys/kernel/tracing/events/.../id から得る */

/* 生の CPU PMU イベント */
.type = PERF_TYPE_RAW
.config = <cpu-specific PMU event code>  /* 例: Intel の perfmon イベント */
```

## サンプリングモード {#sampling-mode}

サンプリングは、Nイベントごとに命令ポインタ（およびオプションでコールスタック）を記録する。

```c
struct perf_event_attr attr = {
    .type           = PERF_TYPE_HARDWARE,
    .config         = PERF_COUNT_HW_CPU_CYCLES,
    .sample_type    = PERF_SAMPLE_IP | PERF_SAMPLE_TID |
                      PERF_SAMPLE_TIME | PERF_SAMPLE_CALLCHAIN,
    .sample_period  = 1000000,  /* 100 万サイクルごと */
    /* または: */
    .sample_freq    = 99,       /* 約 99 Hz（カーネルが周期を調整する） */
    .freq           = 1,        /* 周波数モードを有効にする */
    .wakeup_events  = 1,        /* N サンプルごとにユーザー空間を起こす */
    /* コールチェーンの深さ: */
    .exclude_callchain_kernel = 0,
    .exclude_callchain_user   = 0,
};
```

サンプルは`mmap`されたリングバッファ（perf mmapバッファ）に書き込まれる。

```c
/* リングバッファをマップする */
void *mmap_buf = mmap(NULL, (1 + pages) * PAGE_SIZE,
                      PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

struct perf_event_mmap_page *header = mmap_buf;
/* header->data_head: 書き込み位置（カーネルが更新する） */
/* header->data_tail: 読み出し位置（ユーザー空間が更新する） */

void *data = mmap_buf + PAGE_SIZE;  /* リングはメタデータページの後から始まる */

/* 読み出しループ: */
while (header->data_tail != header->data_head) {
    struct perf_event_header *event = data + (header->data_tail % data_size);
    /* イベントを処理する */
    header->data_tail += event->size;
}
```

## perfツール: ユーザー空間のフロントエンド {#perf-tool-the-userspace-frontend}

### カウント {#counting}

```bash
# コマンドのイベントを数える
perf stat ls
# Performance counter stats for 'ls':
#      0.98 msec task-clock                #    0.783 CPUs utilized
#         1      context-switches          #    1.022 K/sec
#         0      cpu-migrations            #    0.000 /sec
#       163      page-faults               #  166.389 K/sec
#   2,345,678    cycles                    #    2.394 GHz
#   1,234,567    instructions              #    0.53  insn per cycle
#     456,789    branches                  #  466.520 M/sec
#      12,345    branch-misses             #    2.70% of all branches

# 特定のイベント
perf stat -e cycles,instructions,cache-misses,cache-references -- ./myapp

# システム全体
perf stat -a sleep 5

# CPI（命令あたりのサイクル数）の分析
perf stat -e cycles,instructions -r 3 ./myapp  # 3 回繰り返す
```

### perf recordによるCPUプロファイリング {#cpu-profiling-with-perf-record}

```bash
# コールグラフ付きで記録する（フレームポインタ）
perf record -g -F 99 -- ./myapp
# あるいは: すべての CPU をサンプリングする
perf record -g -F 99 -a sleep 30

# レポート
perf report
# Overhead  Command   Symbol
#  25.43%   myapp     [k] __schedule
#  18.21%   myapp     [.] malloc
#  12.33%   myapp     [.] my_function

# コールグラフを表示する
perf report --call-graph=graph

# ソースで注釈を付ける
perf report --stdio
```

### フレームグラフ {#flame-graphs}

```bash
# 記録する
perf record -g -F 99 -p <pid> sleep 30

# フレームグラフに変換する（Brendan Gregg のスクリプトを使う）
perf script | stackcollapse-perf.pl | flamegraph.pl > flamegraph.svg

# あるいは hotspot（GUI）を使う
hotspot perf.data
```

### カーネル内部: struct perf_event {#kernel-internals-struct-perf_event}

```c
/* include/linux/perf_event.h */
struct perf_event {
    struct list_head        event_entry;
    struct list_head        sibling_list; /* グループの兄弟 */
    struct list_head        active_list;

    struct perf_event_attr  attr;         /* ユーザー空間からの設定 */
    struct hw_perf_event    hw;           /* PMU ハードウェアの状態 */

    struct perf_event_context *ctx;       /* コンテキスト（タスクまたは CPU） */
    atomic_long_t           refcount;

    /* オーバーフロー／サンプル発生時のコールバック: */
    perf_overflow_handler_t overflow_handler;
    void                   *overflow_handler_context;

    /* サンプル用のリングバッファ: */
    struct perf_buffer __rcu *rb;

    /* tracepoint: */
    struct trace_event_call *tp_event;

    /* BPF: */
    struct bpf_prog         *prog;

    u64                     id;           /* 一意なイベント ID */
    /* ... */
};

struct hw_perf_event {
    /* PMU ハードウェアレジスタ: */
    union {
        struct { /* ハードウェア */
            u64 config;          /* PMU にプログラムされる値 */
            u64 last_tag;
            unsigned long config_base;
            unsigned long event_base;
            int event_base_rdpmc;
            int idx;             /* カウンタのインデックス */
            int last_cpu;
            int flags;
        };
        /* ... */
    };
    u64                     prev_count;  /* 最後に読んだ値 */
    u64                     sample_period;
    u64                     last_period;
    local64_t               period_left;
    /* ... */
};
```

### PMU: ハードウェア性能監視ユニット {#pmu-hardware-performance-monitoring-unit}

各CPUアーキテクチャには、プログラム可能なカウンタを限られた数だけ持つPMUがある。Intelの場合:

```
Intel PMU（Skylake）:
  - 汎用カウンタ 4 個（PERFCTR0..3）  ← 任意のイベントをプログラム可能
  - 固定機能カウンタ 3 個:
      FIXED_CTR0 = リタイアした命令
      FIXED_CTR1 = CPU クロックサイクル
      FIXED_CTR2 = リファレンスサイクル
  - 各カウンタはオーバーフロー時に NMI を発生させる

関連する MSR:
  IA32_PERFEVTSELx: イベントセレクタ（event、umask、usr、os、edge など）
  IA32_PMCx:        カウンタ値
  IA32_FIXED_CTR_CTRL: 固定カウンタを有効にする
  IA32_PERF_GLOBAL_CTRL: すべてのカウンタを有効化／無効化する
```

perfサブシステムは、`context_switch`と`rotation`を用いて、限られたハードウェアカウンタ上にソフトウェアイベントを多重化する。

## perfの内部を観察する {#observing-perf-internals}

```bash
# 利用可能なイベントを一覧表示する
perf list

# PMU が動作しているか確認する
perf stat -e cycles echo hello

# 生イベント用のハードウェアイベントコード
# Intel: https://perfmon-events.intel.com/
perf stat -e r4c14  # 生の Intel イベントコード（16 進）

# NMI ハンドラを確認する
cat /proc/interrupts | grep NMI

# perf_event_paranoid: アクセスを制御する
cat /proc/sys/kernel/perf_event_paranoid
# -1: 制限なし, 0: カーネルのプロファイリングを許可, 1: ユーザーのみ, 2: カーネルアドレスなし（デフォルト）
echo 0 > /proc/sys/kernel/perf_event_paranoid
```

## 参考文献 {#further-reading}

- [ftrace](ftrace.md) — 低レベルな関数トレース
- [kprobeとtracepoint](kprobes-tracepoints.md) — perfで使われる動的・静的イベント
- [BPF: アーキテクチャ](../bpf/bpf-overview.md) — BPF_PROG_TYPE_PERF_EVENTプログラム
- カーネルツリー内の`tools/perf/` — perfツールのソース
- `Documentation/admin-guide/perf-security.rst`
