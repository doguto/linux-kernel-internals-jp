# BPF Maps {#bpf-maps}

> BPFプログラム同士、そしてBPFプログラムとユーザー空間をつなぐデータ構造

## BPF mapとは何か {#what-bpf-maps-are}

BPF mapは、BPFプログラム（ヘルパー経由）とユーザー空間（bpf()システムコール経由）の両方からアクセスできるキーバリューストアである。次の用途のための主要な仕組みである。

- BPFプログラムからユーザー空間へのデータ受け渡し（統計、イベント）
- ユーザー空間からBPFプログラムへの設定の受け渡し
- 複数のBPFプログラム間での状態の共有
- 同じBPFプログラムの複数回の呼び出しにまたがる状態の保持

```
BPF program                       Userspace
───────────────────────────────────────────────
bpf_map_lookup_elem(&map, &key)  bpf(BPF_MAP_LOOKUP_ELEM, ...)
bpf_map_update_elem(&map, &key, &val, BPF_ANY)
bpf_map_delete_elem(&map, &key)  bpf(BPF_MAP_DELETE_ELEM, ...)
```

## mapの作成 {#creating-a-map}

```c
/* BPFプログラムから（libbpfのマクロを使用） */
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 10000);
    __type(key, u32);         /* キーの型（BTF対応ツール向け） */
    __type(value, u64);       /* 値の型 */
} counts SEC(".maps");

/* ユーザー空間から直接 */
union bpf_attr attr = {
    .map_type    = BPF_MAP_TYPE_HASH,
    .key_size    = sizeof(u32),
    .value_size  = sizeof(u64),
    .max_entries = 10000,
};
int map_fd = bpf(BPF_MAP_CREATE, &attr, sizeof(attr));
```

## mapタイプ {#map-types}

### BPF_MAP_TYPE_HASH {#bpfmaptype_hash}

任意のキー/値サイズを持つ汎用ハッシュmap。

```c
/* BPF側 */
u32 pid = bpf_get_current_pid_tgid() >> 32;
u64 *count = bpf_map_lookup_elem(&counts, &pid);
if (count) {
    (*count)++;
} else {
    u64 init = 1;
    bpf_map_update_elem(&counts, &pid, &init, BPF_NOEXIST);
}
```

更新フラグ:

- `BPF_ANY` — 作成または更新
- `BPF_NOEXIST` — 作成のみ（キーが存在すれば失敗）
- `BPF_EXIST` — 更新のみ（キーが存在しなければ失敗）

内部構造: ロックベースのハッシュテーブルを使い、エントリごとに`htab_elem`を持つ。複数CPUからの並行アクセスをサポートする。

### BPF_MAP_TYPE_ARRAY {#bpfmaptype_array}

u32でインデックス付けされる固定サイズの配列。事前に割り当てられ、実行時に動的メモリを使わない。値はゼロクリアする必要がある（削除はできない）。

```c
/* 小さな整数でインデックス付けされるイベントごとのカウンタに向く */
struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __uint(max_entries, 256);    /* インデックス0..255 */
    __type(key, u32);
    __type(value, u64);
} hist SEC(".maps");

u32 slot = /* ヒストグラムのバケットを計算する */;
u64 *val = bpf_map_lookup_elem(&hist, &slot);
if (val)
    __sync_fetch_and_add(val, 1);  /* アトミックなインクリメント */
```

### BPF_MAP_TYPE_PERCPU_HASH / BPF_MAP_TYPE_PERCPU_ARRAY {#bpfmaptypepercpuhash-bpfmaptypepercpuarray}

CPUごとに1つの値を持つ — CPUごとのカウンタでアトミック操作が不要になる。

```c
struct {
    __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
    __uint(max_entries, 1);
    __type(key, u32);
    __type(value, u64);
} bytes_total SEC(".maps");

u32 key = 0;
u64 *val = bpf_map_lookup_elem(&bytes_total, &key);
if (val)
    *val += skb->len;  /* アトミック不要 — このCPUだけが触る */
```

ユーザー空間は全CPUの値を読んで合計する。
```c
/* ユーザー空間: nr_cpus個の値を読む */
u64 values[nr_cpus];
bpf_map_lookup_elem(map_fd, &key, values);
u64 total = 0;
for (int i = 0; i < nr_cpus; i++)
    total += values[i];
```

### BPF_MAP_TYPE_RINGBUF {#bpfmaptype_ringbuf}

BPFからユーザー空間へ可変長イベントを送るための推奨される仕組み（ほとんどのユースケースでperf_event_outputを置き換える）。

```c
struct event {
    u32 pid;
    u32 uid;
    char comm[16];
    u32 syscall_nr;
};

struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 1 << 24);  /* 16MBのリングバッファ */
} events SEC(".maps");

/* BPF側: 予約 → 書き込み → submit */
SEC("tracepoint/syscalls/sys_enter_execve")
int trace_execve(struct trace_event_raw_sys_enter *ctx)
{
    struct event *e = bpf_ringbuf_reserve(&events, sizeof(*e), 0);
    if (!e)
        return 0;  /* リングが満杯、破棄 */

    e->pid = bpf_get_current_pid_tgid() >> 32;
    e->uid = bpf_get_current_uid_gid();
    bpf_get_current_comm(e->comm, sizeof(e->comm));
    e->syscall_nr = ctx->id;

    bpf_ringbuf_submit(e, 0);  /* 0 = BPF_RB_*フラグなし */
    return 0;
}
```

```c
/* ユーザー空間: epollでポーリングする */
struct ring_buffer *rb = ring_buffer__new(map_fd, handle_event, NULL, NULL);

while (true) {
    ring_buffer__poll(rb, 100 /* タイムアウト(ms) */);
}

static int handle_event(void *ctx, void *data, size_t size)
{
    struct event *e = data;
    printf("pid=%u comm=%s syscall=%u\n", e->pid, e->comm, e->syscall_nr);
    return 0;
}
```

perf_event出力よりringbufを選ぶ理由:

- 全CPUで共有される単一のリング（CPUごとの無駄がない）
- リング内でイベントが順序付けられる
- コピーなし — ユーザー空間はリングのメモリから直接読む
- 可変長エントリ

### BPF_MAP_TYPE_LRU_HASH {#bpfmaptypelruhash}

満杯になるとLRUで追い出しを行うハッシュmap。明示的なクリーンアップなしでコネクションを追跡するのに便利である。

```c
struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __uint(max_entries, 100000);
    __type(key, struct flow_key);
    __type(value, struct flow_stats);
} flows SEC(".maps");
/* 最も長く使われていないエントリを自動的に追い出す */
```

### BPF_MAP_TYPE_PROG_ARRAY {#bpfmaptypeprogarray}

テールコール用のBPFプログラムのファイルディスクリプタの配列。スタックを伸ばさずにプログラムを連鎖させられる。

```c
struct {
    __uint(type, BPF_MAP_TYPE_PROG_ARRAY);
    __uint(max_entries, 10);
    __uint(key_size, sizeof(u32));
    __uint(value_size, sizeof(u32));  /* プログラムのfd */
} jump_table SEC(".maps");

/* テールコール: prog[index]へジャンプし、決して戻らない */
u32 index = 3;
bpf_tail_call(ctx, &jump_table, index);
/* テールコールが成功すれば、以下のコードは決して実行されない */
/* prog[3]が空なら、実行はここから続く */
```

テールコールはコールスタックの深さを増やさない。現在のプログラムを置き換えるのである。連鎖の最大長は33である。

### BPF_MAP_TYPE_HASH_OF_MAPS / BPF_MAP_TYPE_ARRAY_OF_MAPS {#bpfmaptypehashofmaps-bpfmaptypearrayofmaps}

map-in-map: 値が別のmapのファイルディスクリプタである。CPUごと、あるいはnamespaceごとのmap分離に便利である。

```c
/* 外側のmap: key=netnsのinode、value=内側のmapのfd */
struct {
    __uint(type, BPF_MAP_TYPE_HASH_OF_MAPS);
    __uint(max_entries, 1000);
    __type(key, u32);
    __uint(value_size, sizeof(u32));  /* 内側のmapのfd */
    __array(values, struct inner_map_type);
} per_netns_stats SEC(".maps");
```

### BPF_MAP_TYPE_SOCKMAP / BPF_MAP_TYPE_SOCKHASH {#bpfmaptypesockmap-bpfmaptypesockhash}

ソケットへの参照を格納する。sk_msgおよびsk_skbプログラムタイプでのソケットリダイレクトに使われる。

```c
/* カーネル内コピーなしでソケット間をリダイレクトする */
bpf_msg_redirect_map(msg, &sock_map, key, 0);
```

## map操作のまとめ {#map-operations-summary}

| 操作 | BPFヘルパー | ユーザー空間のシステムコール |
|-----------|-----------|-------------------|
| 検索 | `bpf_map_lookup_elem` | `BPF_MAP_LOOKUP_ELEM` |
| 更新 | `bpf_map_update_elem` | `BPF_MAP_UPDATE_ELEM` |
| 削除 | `bpf_map_delete_elem` | `BPF_MAP_DELETE_ELEM` |
| 反復 | —（BPFからは不可） | `BPF_MAP_GET_NEXT_KEY` |
| バッチ操作 | — | `BPF_MAP_LOOKUP_BATCH` |

## mapのピン留め（永続化） {#map-pinning-persistence}

デフォルトでは、mapはそのファイルディスクリプタが開いている間だけ存在する。プロセス終了後も残すには、`/sys/fs/bpf/`にピン留めする。

```bash
# bpftoolでピン留めする
bpftool map pin id 42 /sys/fs/bpf/my_map

# 別のプロセスからアクセスする
bpftool map dump pinned /sys/fs/bpf/my_map
```

```c
/* ユーザー空間のコードからピン留めする */
bpf(BPF_OBJ_PIN, &(union bpf_attr){
    .pathname = (uint64_t)"/sys/fs/bpf/my_map",
    .bpf_fd   = map_fd,
}, sizeof(union bpf_attr));

/* 後で取り出す */
int map_fd = bpf(BPF_OBJ_GET, &(union bpf_attr){
    .pathname = (uint64_t)"/sys/fs/bpf/my_map",
}, sizeof(union bpf_attr));
```

## mapの制限とメモリ {#map-limits-and-memory}

```bash
# mapのメモリ使用量を確認する
bpftool map list
# 3: hash  name counts  flags 0x0
#    key 4B  value 8B  max_entries 10000  memlock 819200B

# BPFのメモリ合計
cat /proc/meminfo | grep Bpf

# プロセスごとのBPFメモリ制限（ulimit）
ulimit -l  # RLIMIT_MEMLOCK
# rootは無制限、非特権ユーザーは制限される
```

メモリアカウンティング: mapは作成したプロセスの`memlock` rlimitに課金される（`BPF_F_MMAPABLE`や新しいアカウンティング方式の場合を除く）。

## map内でのアトミック操作 {#atomic-operations-in-maps}

hash/array mapでper-CPUを使わずにアトミックに更新するには:

```c
/* アトミック加算（BPF組み込み） */
u64 *count = bpf_map_lookup_elem(&map, &key);
if (count)
    __sync_fetch_and_add(count, 1);

/* BPFのアトミック命令（カーネル5.12以降） */
u64 *val = bpf_map_lookup_elem(&map, &key);
if (val)
    __atomic_add_fetch(val, 1, __ATOMIC_RELAXED);
```

高頻度のカウンタには、アトミック操作を完全に避けられる`BPF_MAP_TYPE_PERCPU_ARRAY`を優先すること。

## 参考文献 {#further-reading}

- [BPF Verifier](bpf-verifier.md) — mapアクセスの安全性がどのようにチェックされるか
- [libbpfとスケルトン](libbpf.md) — ユーザー空間からの便利なmapアクセス
- `kernel/bpf/hashtab.c` — ハッシュmapの実装
- `kernel/bpf/ringbuf.c` — リングバッファの実装
