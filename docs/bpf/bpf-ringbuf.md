# BPFリングバッファ {#bpf-ring-buffer}

> BPFプログラムからの、ロックフリーでメモリ効率の良いイベントストリーミング

## なぜperfイベント出力ではないのか {#why-not-perf-event-output}

`BPF_MAP_TYPE_RINGBUF`（5.8）以前は、BPFプログラムは`bpf_perf_event_output`を介してユーザー空間にイベントをストリーミングしていた。

```c
/* 旧来の方式: CPUごとのperfリングバッファ */
struct bpf_map_def SEC("maps") events = {
    .type        = BPF_MAP_TYPE_PERF_EVENT_ARRAY,
    .key_size    = sizeof(int),
    .value_size  = sizeof(u32),
    .max_entries = MAX_CPUS,
};

SEC("tracepoint/syscalls/sys_enter_write")
int trace_write(struct trace_event_raw_sys_enter *ctx) {
    struct event e = { .pid = bpf_get_current_pid_tgid() >> 32 };
    bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, &e, sizeof(e));
    return 0;
}
```

`PERF_EVENT_ARRAY`の問題点:

- **CPUごと**: CPUごとに別々のバッファがあり、ユーザー空間ではCPUごとのポーリングが必要になる
- **メモリの無駄**: ほとんどアイドル状態であっても、各CPUがバッファを丸ごと1つ持つ
- **可変長レコード不可**: イベントのサイズを事前に宣言しなければならない
- **ウェイクアップのオーバーヘッド**: `epoll_wait`にはCPUごとのfdが必要になる

## BPFリングバッファの設計 {#bpf-ring-buffer-design}

`BPF_MAP_TYPE_RINGBUF`は、全CPUで共有される**単一のリングバッファ**を、ロックフリーな設計で使う。

```
Ring buffer memory layout:

  consumer_pos (4 bytes) ← read by userspace, written by userspace
  [padding]
  producer_pos (4 bytes) ← read by userspace, written by BPF kernel
  [padding]
  data area (size bytes) ← records written here

  Record format:
  ┌────────────────────────────────────────┐
  │ len (28 bits) | flags (4 bits)         │  ← 4-byte header
  ├────────────────────────────────────────┤
  │ data (len bytes)                       │
  │ [padding to 8-byte alignment]          │
  └────────────────────────────────────────┘

  flags: BPF_RINGBUF_BUSY_BIT   = 0x1 (record being written)
         BPF_RINGBUF_DISCARD_BIT = 0x2 (record discarded)
```

主な性質:

- **単一バッファ**: CPUごとの無駄がない。全CPUが1つのリングを共有する
- **spinlock不要**: producer位置の予約にcompare-and-swapを使う
- **2フェーズコミット**: 領域を予約し、そこを埋め、それからsubmitまたはdiscardする
- **メモリマップ**: ユーザー空間はシステムコールなしで直接読み出す

## BPFプログラム側 {#bpf-program-side}

```c
#include <vmlinux.h>
#include <bpf/bpf_helpers.h>

struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 256 * 1024);  /* 256KBのリング */
} rb SEC(".maps");

struct event {
    u32 pid;
    u32 tid;
    u64 ts;
    char comm[16];
    int fd;
    ssize_t ret;
};

SEC("tracepoint/syscalls/sys_exit_write")
int trace_write_exit(struct trace_event_raw_sys_exit *ctx)
{
    struct event *e;

    /* リングバッファ内の領域を予約する */
    e = bpf_ringbuf_reserve(&rb, sizeof(*e), 0);
    if (!e)
        return 0;  /* リングが満杯: イベントを破棄 */

    /* レコードを埋める（安全: submitまではこちらが所有している） */
    e->pid = bpf_get_current_pid_tgid() >> 32;
    e->tid = (u32)bpf_get_current_pid_tgid();
    e->ts  = bpf_ktime_get_ns();
    bpf_get_current_comm(e->comm, sizeof(e->comm));
    e->fd  = (int)ctx->args[0];
    e->ret = (ssize_t)ctx->ret;

    /* コミット: ユーザー空間から見えるようにする */
    bpf_ringbuf_submit(e, 0);
    return 0;
}
```

### bpf_ringbuf_output: 1回で済ませる代替手段 {#bpf_ringbuf_output-one-shot-alternative}

reserve+submitが大げさな、小さな固定長イベントの場合:

```c
/* 1回の呼び出し: 予約、コピー、submitをアトミックに行う */
bpf_ringbuf_output(&rb, &event, sizeof(event), 0);
/* 大きなイベントではreserve+submitより効率が悪い */
/* （スタックからコピーするため、その場で埋められない） */
```

### エラー時のdiscard {#discard-on-error}

```c
e = bpf_ringbuf_reserve(&rb, sizeof(*e), 0);
if (!e)
    return 0;

/* ... eを埋める ... */

if (should_drop) {
    bpf_ringbuf_discard(e, 0);  /* 予約した領域を解放する。イベントは出ない */
    return 0;
}

bpf_ringbuf_submit(e, 0);
```

## libbpfによるユーザー空間での消費 {#userspace-consumption-with-libbpf}

```c
#include <bpf/libbpf.h>

struct ring_buffer *rb;

static int handle_event(void *ctx, void *data, size_t size)
{
    struct event *e = data;
    printf("[%llu] pid=%u comm=%s fd=%d ret=%zd\n",
           e->ts, e->pid, e->comm, e->fd, e->ret);
    return 0;
}

int main(void)
{
    struct my_bpf *skel = my_bpf__open_and_load();
    my_bpf__attach(skel);

    /* リングバッファのコンシューマを作成する */
    rb = ring_buffer__new(bpf_map__fd(skel->maps.rb),
                           handle_event, NULL, NULL);

    /* ポーリング: 利用可能な各レコードに対してhandle_eventを呼ぶ */
    while (1) {
        ring_buffer__poll(rb, 100 /* ミリ秒のタイムアウト */);
    }

    ring_buffer__free(rb);
    my_bpf__destroy(skel);
}
```

### 手動のmmapポーリング {#manual-mmap-polling}

libbpfを使わない場合、リングバッファには`mmap`経由でアクセスする。

```c
int map_fd = bpf_map__fd(skel->maps.rb);
int page_size = getpagesize();
int map_size = 256 * 1024;

/* マップ: consumerページ + producerページ + データ領域 */
void *mmap_base = mmap(NULL, page_size + page_size + map_size,
                        PROT_READ | PROT_WRITE, MAP_SHARED,
                        map_fd, 0);

unsigned long *consumer_pos = mmap_base;
unsigned long *producer_pos = mmap_base + page_size;
void *data = mmap_base + 2 * page_size;

/* ポーリングループ: */
while (1) {
    unsigned long prod = __atomic_load_n(producer_pos, __ATOMIC_ACQUIRE);
    unsigned long cons = *consumer_pos;

    while (cons < prod) {
        unsigned long offset = cons % map_size;
        u32 *hdr = data + offset;
        u32  len = *hdr & ~0xf;  /* フラグビットをマスクする */
        u32  flags = *hdr & 0xf;

        if (flags & BPF_RINGBUF_BUSY_BIT)
            break;  /* producerがまだコミットしていない */

        if (!(flags & BPF_RINGBUF_DISCARD_BIT)) {
            /* 有効なレコード: データはヘッダの後から始まる */
            process_event(data + offset + sizeof(u32), len);
        }

        cons += sizeof(u32) + ALIGN(len, 8);
        *consumer_pos = cons;  /* consumerを進める */
    }

    /* スリープするか、リングバッファのfdでepollを使う */
    usleep(1000);
}
```

## BPF_MAP_TYPE_RINGBUFとPERF_EVENT_ARRAYの比較 {#bpf_map_type_ringbuf-vs-perf_event_array}

| 機能 | PERF_EVENT_ARRAY | RINGBUF |
|---------|-----------------|---------|
| メモリ | CPUごとのバッファ | 単一の共有バッファ |
| 可変長レコード | 不可 | 可 |
| ユーザー空間でのポーリング | CPUごとのepoll fd | 単一のfd |
| その場での書き込み | 不可（スタックからコピー） | 可（reserve+submit） |
| discardのサポート | なし | あり |
| mmapアクセス | 可 | 可 |
| カーネルバージョン | 4.4 | 5.8 |
| オーバーヘッド | 高い | 低い |

## ウェイクアップの制御 {#wakeup-control}

```c
/* ユーザー空間を起こさずにsubmitする（複数イベントをバッチ化） */
bpf_ringbuf_submit(e, BPF_RB_NO_WAKEUP);

/* 即座にウェイクアップを強制する（レイテンシが重要なイベント向け） */
bpf_ringbuf_submit(e, BPF_RB_FORCE_WAKEUP);

/* デフォルト: ユーザー空間が待っていれば起こし、そうでなければ起こさない */
bpf_ringbuf_submit(e, 0);
```

libbpfの`ring_buffer__poll`と組み合わせると、`BPF_RB_NO_WAKEUP`と最後の`BPF_RB_FORCE_WAKEUP`によって多数のイベントを1回のウェイクアップにまとめることができ、オーバーヘッドを大幅に減らせる。

## 空き容量の問い合わせ {#query-available-space}

```c
/* 割り当てる前にリングに空きがあるか確認する */
u64 avail = bpf_ringbuf_query(&rb, BPF_RB_AVAIL_DATA);
u64 ring_size = bpf_ringbuf_query(&rb, BPF_RB_RING_SIZE);
u64 cons_pos  = bpf_ringbuf_query(&rb, BPF_RB_CONS_POS);
u64 prod_pos  = bpf_ringbuf_query(&rb, BPF_RB_PROD_POS);
```

## 参考文献 {#further-reading}

- [BPF Maps](bpf-maps.md) — その他のBPF mapタイプ
- [BPFアーキテクチャ](bpf-overview.md) — プログラムタイプの概要
- [libbpfとスケルトン](libbpf.md) — `ring_buffer__new` / `ring_buffer__poll`
- [kprobeとtracepoint](../tracing/kprobes-tracepoints.md) — tracepointへのアタッチ
- `kernel/bpf/ringbuf.c` — リングバッファの実装
- `tools/lib/bpf/ringbuf.c` — libbpfのコンシューマ
