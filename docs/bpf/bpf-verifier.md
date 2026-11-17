# BPF Verifier {#bpf-verifier}

> カーネルは、BPFプログラムを実行する前にそれが安全であることをどのように証明するのか

## Verifierが行うこと {#what-the-verifier-does}

すべてのBPFプログラムは、実行可能になる前にVerifierを通過しなければならない。Verifierはバイトコードを静的に解析し、次のことを証明する。

1. **停止性** — 無限ループがない（カーネル5.3以降は有界ループが可能）
2. **メモリ安全性** — すべてのポインタ参照外しが範囲内である
3. **型安全性** — ヘルパーが正しい型の引数で呼び出される
4. **コンテキストの正しさ** — そのプログラムタイプで許可されたフィールドにしかアクセスしない
5. **スタック安全性** — スタックオーバーフローがなく、未初期化領域の読み出しもない

Verifierはこの解析を`kernel/bpf/verifier.c`（約30,000行）で行う。Alexei StarovoitovによってLinux 3.18でeBPFとともに導入された。[(commit)](https://git.kernel.org/linus/17a5267067f3c372fec9ffb798d6eaba6b5e6a4c)

## 検証の2つのフェーズ {#two-phases-of-verification}

### フェーズ1: CFG解析（デッドコードの排除） {#phase-1-cfg-analysis-dead-code-elimination}

```
bpf_check()
  → check_subprogs()        /* 関数の境界を見つける */
  → check_cfg()             /* 制御フローグラフを構築する */
      → mark reachable instructions
      → detect back edges (loops)
      → verify loop bounds (for bounded loops)
```

CFGチェックは到達可能なすべての命令に印を付ける。デッドコード（到達不能なコード）は拒否される。

### フェーズ2: 抽象解釈（型と範囲の追跡） {#phase-2-abstract-interpretation-type-and-bounds-tracking}

```
bpf_check()
  → do_check_main()         /* すべてのパスの実行をシミュレートする */
      → for each instruction:
          check_alu_op()    /* 算術演算の安全性 */
          check_mem_access() /* ポインタ参照外しの安全性 */
          check_helper_call() /* ヘルパー引数の型 */
      → explore all branches (must verify both sides of conditional)
```

## レジスタ状態の追跡 {#register-state-tracking}

Verifierは、プログラムのあらゆる地点で、すべてのレジスタの型と範囲を追跡する。

```c
/* kernel/bpf/verifier.c */
struct bpf_reg_state {
    enum bpf_reg_type type;  /* これがどの種類のポインタか */
    /* スカラー値の場合: */
    s64  smin_value;   /* 符号付き最小値 */
    s64  smax_value;   /* 符号付き最大値 */
    u64  umin_value;   /* 符号なし最小値 */
    u64  umax_value;   /* 符号なし最大値 */
    /* ポインタの場合: */
    s32  off;          /* 加算された定数オフセット */
    u32  range;        /* このポインタからアクセス可能なバイト数 */
    struct bpf_map *map_ptr; /* PTR_TO_MAP_VALUEの場合 */
    /* ... */
};

enum bpf_reg_type {
    NOT_INIT,           /* レジスタが初期化されていない */
    SCALAR_VALUE,       /* 整数（ポインタの意味を持たない） */
    PTR_TO_CTX,         /* プログラムコンテキスト（xdp_mdなど）へのポインタ */
    PTR_TO_MAP_KEY,     /* スタック上のmapキー領域へのポインタ */
    PTR_TO_MAP_VALUE,   /* map値へのポインタ（lookupの結果） */
    PTR_TO_MAP_VALUE_OR_NULL, /* 使用前にNULLチェックが必須 */
    PTR_TO_STACK,       /* BPFスタック内へのポインタ */
    PTR_TO_PACKET,      /* skbデータ内へのポインタ */
    PTR_TO_PACKET_END,  /* skbデータの終端へのポインタ */
    PTR_TO_FUNC,        /* BPFサブ関数へのポインタ */
    /* ... */
};
```

## ポインタの安全性: NULLチェックの要求 {#pointer-safety-the-null-check-requirement}

Verifierに拒否される最も一般的な原因は、NULLチェックをせずにポインタを使うことである。

```c
/* これは検証に失敗する */
SEC("kprobe/tcp_sendmsg")
int bad_prog(struct pt_regs *ctx)
{
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    u64 *count = bpf_map_lookup_elem(&counts, &pid);
    (*count)++;  /* エラー: countはNULLかもしれない */
    return 0;
}

/* Verifierのエラー:
 * 5: (85) call unknown#1
 * 6: (07) r0 += 1
 * R0 invalid mem access 'map_value_or_null'
 */

/* これは通る */
SEC("kprobe/tcp_sendmsg")
int good_prog(struct pt_regs *ctx)
{
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    u64 *count = bpf_map_lookup_elem(&counts, &pid);
    if (!count)
        return 0;  /* NULLチェックが必須 */
    (*count)++;    /* これでVerifierはcount != NULLだと分かる */
    return 0;
}
```

NULLチェックの後、`count`は`PTR_TO_MAP_VALUE_OR_NULL`から`PTR_TO_MAP_VALUE`に遷移する — Verifierは、2つの分岐にまたがるこの状態の分割を追跡している。

## パケットアクセスの範囲チェック {#bounds-checking-for-packet-access}

XDPとTCのプログラムは、すべてのパケットアクセスが範囲内であることを証明しなければならない。

```c
SEC("xdp")
int parse_http(struct xdp_md *ctx)
{
    void *data     = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    struct ethhdr *eth = data;

    /* 必須: ethのフィールドにアクセスする前に範囲チェックを行う */
    if ((void *)(eth + 1) > data_end)
        return XDP_PASS;

    /* これでVerifierはeth[0..sizeof(ethhdr)-1]が有効だと分かる */
    if (eth->h_proto != htons(ETH_P_IP))
        return XDP_PASS;

    struct iphdr *ip = data + sizeof(struct ethhdr);
    if ((void *)(ip + 1) > data_end)
        return XDP_PASS;

    /* ip->...へのアクセスは、これで安全と検証された */
    return XDP_PASS;
}
```

Verifierは`PTR_TO_PACKET`のオフセットを追跡し、算術演算の結果が`[data, data_end)`の範囲内に収まることを検証する。

## スタックの使用 {#stack-usage}

BPFプログラムのスタックは512バイト固定である。Verifierはどのスタックスロットが初期化済みかを追跡する。

```c
/* スタックに割り当てられた変数はスロット（8バイト）単位で追跡される */
SEC("kprobe/do_sys_open")
int trace_open(struct pt_regs *ctx)
{
    char path[256];  /* 512バイトのスタックのうち256バイトを使う */

    /* ヘルパーに渡す前に初期化しなければならない */
    /* bpf_probe_read_user_strがpath[]を初期化する */
    bpf_probe_read_user_str(path, sizeof(path),
                            (void *)PT_REGS_PARM2(ctx));
    bpf_printk("open: %s\n", path);
    return 0;
}
```

未初期化のスタックスロットを、それを読み出すヘルパーに渡すとVerifierエラーになる。

## 有界ループ（5.3以降） {#bounded-loops-since-53}

5.3より前は、ループはすべて禁止されていた。現在のVerifierは、上限を証明できればループを許可する。

```c
/* 動く: Verifierはi < 10を追跡できる */
int sum = 0;
for (int i = 0; i < 10; i++)
    sum += array[i];

/* 動く: mapのmax_entriesで上限が決まる */
bpf_for_each_map_elem(&map, callback, &ctx, 0);

/* 失敗するかもしれない: 動的な上限の場合、Verifierは常に停止性を
   証明できるとは限らない — #pragma unrollか手動の上限を加える */
u32 n = bpf_map_lookup_elem(&config, &zero);
for (u32 i = 0; i < n; i++) {   /* nは実行時の値 */
    /* 反復回数の上限を決められなければVerifierは拒否するかもしれない */
}

/* 修正: 上限を明示的に抑える */
if (n > MAX_ITERATIONS)
    n = MAX_ITERATIONS;
for (u32 i = 0; i < n; i++) { /* これで有界になる */ }
```

Verifierは、解析中の状態数の指数的な爆発を防ぐためにループの反復回数を制限する。

## 複雑度の制限 {#complexity-limit}

Verifierはプログラムの状態（各命令におけるレジスタとスタックのスナップショット）を追跡する。複雑なプログラムによるDoSを防ぐため、次の制限がある。

```
BPF_COMPLEXITY_LIMIT_INSNS = 1,000,000  /* 検証される命令数 */
/* Verifierは状態の爆発を防ぐため、命令あたりの状態数も制限する */
```

プログラムが複雑すぎる場合:
```
BPF program is too large. Processed 1000001 insn
```

複雑度を下げるための戦略:

- `__always_inline`を使ってヘルパー関数をインライン化する（呼び出し箇所を減らす）
- テールコールに分割する（各プログラムがそれぞれの上限を持つ）
- 展開されたループの代わりに`bpf_loop()`を使う

## Verifierの出力を読む {#reading-verifier-output}

ロードに失敗したときはVerifierログを要求する。

```c
char log_buf[1 << 16];
union bpf_attr attr = {
    .log_level = 2,           /* 1=エラー、2=詳細、3以上=さらに詳細 */
    .log_buf   = (uint64_t)log_buf,
    .log_size  = sizeof(log_buf),
    /* ... */
};
```

型エラーに対するVerifier出力の例:

```
0: (bf) r6 = r1
1: (85) call bpf_get_current_pid_tgid#14
2: (77) r0 >>= 32
3: (63) *(u32 *)(r10 -4) = r0     ; pidをスタックに格納
4: (bf) r2 = r10
5: (07) r2 += -4                   ; r2 = &pid（スタックポインタ）
6: (18) r1 = 0xffff...             ; r1 = &counts map
8: (85) call bpf_map_lookup_elem#1
9: (15) if r0 == 0x0 goto pc+2    ; NULLチェック
10: (07) r0 += 1                   ; カウンタをインクリメント
11: (62) *(u32 *)(r0 +0) = ...    ; 誤り: ポインタ+0に書き込んでいる
; R0はPTR_TO_MAP_VALUEだが、u64をオフセット0に書き込むには範囲チェックが必要
```

## 特権BPFと非特権BPF {#privileged-vs-unprivileged-bpf}

非特権BPF（`CAP_BPF`が設定されていない）は厳しく制限される。

- `BPF_PROG_TYPE_SOCKET_FILTER`と`BPF_PROG_TYPE_CGROUP_SKB`のみ
- map値へのアクセスを超えるポインタ演算は不可
- JITハードニング: 定数ブラインディング（JITスプレー攻撃を防ぐ）
- Verifierがより厳しいルールを適用する

特権BPFには次が必要である。

- `CAP_BPF`（5.8以降、`CAP_SYS_ADMIN`から分離）
- トレーシングプログラムには`CAP_PERFMON`
- XDPとTCのプログラムには`CAP_NET_ADMIN`

```bash
# カーネルのBPF設定を確認する
sysctl kernel.unprivileged_bpf_disabled
# 0 = 非特権BPFを許可、1 = 無効（元に戻せない）、2 = 無効（管理者が再有効化できる）
# （値にかかわらず、このsysctlへの書き込み自体にCAP_SYS_ADMINが必要）
```

3つの状態の詳しい内訳と、その背後にあるケーパビリティモデルについては[BPFセキュリティモデル](bpf-security-model.md)を参照のこと。

## 参考文献 {#further-reading}

- [BPF Maps](bpf-maps.md) — Verifierがどのようなmapアクセスパターンを許可するか
- [BTFとCO-RE](btf-core.md) — BTFが型情報でVerifierをどう助けるか
- `kernel/bpf/verifier.c` — Verifierの完全な実装
- `Documentation/bpf/verifier.rst` — カーネル公式のVerifierドキュメント
