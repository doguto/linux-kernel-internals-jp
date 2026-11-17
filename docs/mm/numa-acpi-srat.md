# NUMAトポロジの検出: ACPI SRATとSLIT {#numa-topology-discovery-acpi-srat-and-slit}

> カーネルがファームウェアのテーブルを読んで、どのメモリがどのノードに属するか、そしてそれらのノードがどれだけ離れているかを知る仕組み

NUMAシステムでは、カーネルは物理メモリのトポロジを自力で検出できない。どのCPUとメモリ範囲が近接ドメインを共有しているか、そしてそれらのドメインが互いにどれだけ離れているかを記述するのはファームウェアの役割であり、カーネルはそれに頼っている。x86とARM64プラットフォームでは、この情報は2つのACPIテーブル、すなわち**SRAT**（System Resource Affinity Table）と**SLIT**（System Locality Information Table）にある。

## 主要なソースファイル {#key-source-files}

| ファイル | 説明 |
|------|-------------|
| [`drivers/acpi/numa/srat.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/acpi/numa/srat.c) | SRAT/SLITパースの中核: `acpi_numa_init()`、`acpi_parse_slit()` |
| [`arch/x86/mm/srat.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/srat.c) | x86のエントリポイント（`x86_acpi_numa_init()`）とCPUアフィニティのコールバック |
| [`arch/x86/mm/numa.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/numa.c) | x86のNUMA初期化の統括、`dummy_numa_init()` |
| [`mm/numa_memblks.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/numa_memblks.c) | `numa_add_memblk()`、`numa_set_distance()`、NUMA距離テーブル |
| [`include/acpi/actbl3.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/acpi/actbl3.h) | SRATとSLITのC構造体定義 |

## SRATとSLITとは何か {#what-are-srat-and-slit}

SRATとSLITはACPIのファームウェアテーブルであり、両者を合わせてNUMAシステムの物理メモリトポロジを記述する。

**SRAT**が答えるのは: *どのCPUとどのメモリ範囲がどの近接ドメインに属するか?*

**SLIT**が答えるのは: *近接ドメイン同士はどれだけ離れているか?*

**近接ドメイン**（proximity domain、PXM）は、NUMAの局所性グループに対するファームウェアの不透明な識別子である。カーネルはブート時に各PXMをLinuxのノード番号に対応付ける。それ以降、PXMの値はACPIパースコードの内部でのみ使われる。

```
Firmware view (SRAT/SLIT)           Linux kernel view
─────────────────────────           ─────────────────
PXM 0  →  CPU 0-7, 0x00000000-0x3FFFFFFFFF    →  node 0
PXM 1  →  CPU 8-15, 0x4000000000-0x7FFFFFFFFF →  node 1

SLIT distances:
  PXM 0 → PXM 0: 10 (local)          node_distance(0,0) = 10
  PXM 0 → PXM 1: 21 (remote)         node_distance(0,1) = 21
  PXM 1 → PXM 0: 21 (remote)         node_distance(1,0) = 21
  PXM 1 → PXM 1: 10 (local)          node_distance(1,1) = 10
```

## SRATテーブルの形式 {#srat-table-format}

SRATは可変長のACPIテーブルである。標準ヘッダで始まり、その後にサブテーブルエントリの列が続く。各エントリは1つのCPUまたは1つのメモリ範囲を記述する。

### テーブルヘッダ {#table-header}

```c
/* include/acpi/actbl3.h */
struct acpi_table_srat {
    struct acpi_table_header header;  /* "SRAT"シグネチャ、長さ、リビジョン */
    u32 table_revision;               /* 1でなければならない */
    u64 reserved;                     /* 予約済み、0でなければならない */
};
```

`header.revision`フィールドは重要である。SRATリビジョン1では、CPUアフィニティエントリの`proximity_domain`の下位8ビットしか使わない。リビジョン2以降は32ビット値全体を使う。

### メモリアフィニティサブテーブル {#memory-affinity-subtable}

各メモリ範囲には`acpi_srat_mem_affinity`エントリが1つずつ割り当てられる:

```c
/* include/acpi/actbl3.h */
struct acpi_srat_mem_affinity {
    struct acpi_subtable_header header;  /* type=1（ACPI_SRAT_TYPE_MEMORY_AFFINITY）、長さ */
    u32 proximity_domain;                /* このメモリが属するPXM */
    u16 reserved;
    u64 base_address;                    /* 物理開始アドレス */
    u64 length;                          /* バイト単位のサイズ */
    u32 reserved1;
    u32 flags;
    u64 reserved2;
};

/* 関連するフラグ */
#define ACPI_SRAT_MEM_ENABLED       (1)      /* エントリは有効であり、使用すべき */
#define ACPI_SRAT_MEM_HOT_PLUGGABLE (1 << 1) /* 範囲はホットプラグ可能 */
#define ACPI_SRAT_MEM_NON_VOLATILE  (1 << 2) /* 範囲は不揮発性（NVDIMM） */
#define ACPI_SRAT_MEM_SPEC_PURPOSE  (1 << 3) /* 特定用途向けに予約 */
```

カーネルは、`ACPI_SRAT_MEM_ENABLED`が設定されていないエントリをすべてスキップする。ホットプラグ可能な範囲は、メモリホットプラグサブシステムが正しく管理できるよう、memblockにホットプラグ領域として登録される。

### CPUアフィニティサブテーブル {#cpu-affinity-subtables}

システムがレガシーなローカルAPICとx2APICのどちらを使うかに応じて、2種類のサブテーブルがCPUを扱う:

```c
/* レガシーAPICのCPU（SRATタイプ0） */
struct acpi_srat_cpu_affinity {
    struct acpi_subtable_header header;
    u8  proximity_domain_lo;    /* PXMの下位8ビット（SRATリビジョン1はこれだけを使う） */
    u8  apic_id;                /* ローカルAPIC ID */
    u32 flags;                  /* ACPI_SRAT_CPU_ENABLED = 1 */
    u8  local_sapic_eid;        /* SAPIC拡張ID（Itanium用、ほとんど使われない） */
    u8  proximity_domain_hi[3]; /* PXMの上位24ビット（SRATリビジョン >= 2） */
    u32 clock_domain;
};

/* x2APICのCPU（SRATタイプ2、ACPI 4.0+） */
struct acpi_srat_x2apic_cpu_affinity {
    struct acpi_subtable_header header;
    u16 reserved;
    u32 proximity_domain;       /* 32ビット全体のPXM */
    u32 apic_id;                /* x2APIC ID */
    u32 flags;                  /* ACPI_SRAT_CPU_ENABLED = 1 */
    u32 clock_domain;
    u32 reserved2;
};

#define ACPI_SRAT_CPU_ENABLED  (1)  /* エントリは有効 */
```

ARM64システムでは、APICベースの構造体の代わりに`acpi_srat_gicc_affinity`（SRATタイプ3）を使う。

## SLITテーブルの形式 {#slit-table-format}

SLITは、すべての近接ドメインの組の間の相対的なメモリアクセスレイテンシを、平坦なバイト行列としてエンコードする:

```c
/* include/acpi/actbl3.h */
struct acpi_table_slit {
    struct acpi_table_header header;  /* "SLIT"シグネチャ */
    u64 locality_count;               /* 近接ドメインの数（N） */
    u8  entry[];                      /* N*Nのバイト行列、行優先 */
};
```

PXM `i`からPXM `j`への距離は次のとおりである:

```
distance = slit->entry[i * locality_count + j]
```

定義済みの基準値（`include/linux/topology.h`より）:

| 値 | 意味 |
|-------|---------|
| `10`（`LOCAL_DISTANCE`） | ローカルノードへのアクセス |
| `20`（`REMOTE_DISTANCE`） | SLITがない場合のデフォルトのリモート距離 |
| > 10 | 大きいほど遠い |

対角成分（`i == j`）は常に`LOCAL_DISTANCE`（10）でなければならない。非対角成分は10より真に大きくなければならない。カーネルはテーブルを使う前に`slit_valid()`でこれを検証する。SLITが検証に失敗すると、何も言わずに破棄され、デフォルトの距離が使われる。

## カーネルのパース経路 {#kernel-parsing-path}

### エントリポイント {#entry-points}

x86では、NUMAの初期化は`arch/x86/mm/numa.c`の`x86_numa_init()`から統括される。これは各初期化方法を、いずれかが成功するまで順に試す:

```
x86_numa_init()
  └─ numa_init(x86_acpi_numa_init)         [CONFIG_ACPI_NUMA]
       └─ x86_acpi_numa_init()              arch/x86/mm/srat.c
            └─ acpi_numa_init()             drivers/acpi/numa/srat.c
                 ├─ parse SRAT table
                 └─ parse SLIT table
  └─ numa_init(amd_numa_init)              [CONFIG_AMD_NUMA, fallback]
  └─ numa_init(of_numa_init)              [Device Tree, acpi_disabled]
  └─ numa_init(dummy_numa_init)           [last resort: single node 0]
```

### acpi_numa_init()でのSRATのパース {#srat-parsing-in-acpi_numa_init}

`drivers/acpi/numa/srat.c`の`acpi_numa_init()`が、SRAT/SLITのパース処理全体を駆動する:

```c
int __init acpi_numa_init(void)
{
    /* SRATテーブルのヘッダをパースし、すべてのサブテーブル種別を走査する */
    if (!acpi_table_parse(ACPI_SIG_SRAT, acpi_parse_srat)) {
        struct acpi_subtable_proc srat_proc[5];

        /* CPUアフィニティサブテーブルが最初に処理される */
        srat_proc[0].id = ACPI_SRAT_TYPE_CPU_AFFINITY;
        srat_proc[0].handler = acpi_parse_processor_affinity;
        srat_proc[1].id = ACPI_SRAT_TYPE_X2APIC_CPU_AFFINITY;
        srat_proc[1].handler = acpi_parse_x2apic_affinity;
        srat_proc[2].id = ACPI_SRAT_TYPE_GICC_AFFINITY;   /* ARM64 */
        srat_proc[2].handler = acpi_parse_gicc_affinity;
        /* ... */
        acpi_table_parse_entries_array(ACPI_SIG_SRAT, ...);

        /* 次にメモリアフィニティサブテーブル */
        cnt = acpi_table_parse_srat(ACPI_SRAT_TYPE_MEMORY_AFFINITY,
                                    acpi_parse_memory_affinity, 0);
    }

    /* SLITをパースする */
    acpi_table_parse(ACPI_SIG_SLIT, acpi_parse_slit);
    /* ... */
}
```

### CPUエントリの処理 {#cpu-entry-processing}

各CPUアフィニティサブテーブルについて、`acpi_numa_processor_affinity_init()`（`arch/x86/mm/srat.c`内）が呼び出される:

1. `ACPI_SRAT_CPU_ENABLED`フラグを確認する。無効なエントリはスキップする。
2. SRATリビジョンを考慮して近接ドメインを取り出す（リビジョン1は`proximity_domain_lo`のみを使う）。
3. `acpi_map_pxm_to_node(pxm)`を呼び出し、このPXMに対応するLinuxのノード番号を取得または割り当てる。
4. APIC ID → ノードの対応を`__apicid_to_node[]`に記録する。

### メモリエントリの処理 {#memory-entry-processing}

各メモリアフィニティサブテーブルについて、`acpi_parse_memory_affinity()`が呼び出される:

1. `ACPI_SRAT_MEM_ENABLED`を確認する。無効なエントリはスキップする。
2. `base_address`と`length`を取り出し、物理範囲`[start, end)`を形成する。
3. `proximity_domain`を取り出す（SRATリビジョン ≤ 1では8ビットにマスクされる）。
4. `acpi_map_pxm_to_node(pxm)`を呼び出してLinuxのノード番号を得る。
5. `numa_add_memblk(node, start, end)`を呼び出して範囲を登録する。

### PXMからノードへの対応付け {#pxm-to-node-mapping}

ファームウェアのPXM値とLinuxのノード番号との対応は、`drivers/acpi/numa/srat.c`内の2つの配列で保持される:

```c
static int pxm_to_node_map[MAX_PXM_DOMAINS];  /* PXM → ノード */
static int node_to_pxm_map[MAX_NUMNODES];      /* ノード → PXM */
```

この対応は2つの公開関数で提供される:

```c
int pxm_to_node(int pxm);   /* PXMが不明ならNUMA_NO_NODEを返す */
int node_to_pxm(int node);  /* 対応がなければPXM_INVALを返す */
```

`acpi_map_pxm_to_node(pxm)`は、あるPXMに初めて遭遇したときに新しいノード番号を割り当てる内部関数である。次に利用可能なスロットを見つけるのに`first_unset_node(nodes_found_map)`を使う。これにより、まばらなPXM値（例: PXM 0、5、100）が、連続したLinuxのノード番号（0、1、2）に詰められる。

### SLITのパース {#slit-parsing}

`acpi_parse_slit()`は、SRATのパースが完了した後、つまりすべてのPXM→ノードの対応がすでに確立された後にSLIT行列を走査する:

```c
for (i = 0; i < slit->locality_count; i++) {
    int from_node = pxm_to_node(i);
    if (from_node == NUMA_NO_NODE)
        continue;  /* SRATにメモリもCPUもないPXM */

    for (j = 0; j < slit->locality_count; j++) {
        int to_node = pxm_to_node(j);
        if (to_node == NUMA_NO_NODE)
            continue;

        numa_set_distance(from_node, to_node,
            slit->entry[slit->locality_count * i + j]);
    }
}
```

SLITには現れるが、対応するSRATエントリ（CPUもメモリもない）を持たないPXMは、何も言わずにスキップされる。

## 近接ドメインからメモリサブシステムへ {#from-proximity-domains-to-the-memory-subsystem}

SRATのパース後、トポロジ情報は2つの構造体に保持され、それらがカーネルの他の部分に供給される。

### numa_memblksとmemblock {#numa_memblks-and-memblock}

`numa_add_memblk()`は、`[start, end)` → `node`の対応を内部の`numa_meminfo`構造体（`mm/numa_memblks.c`内）に記録する。後に`numa_memblks_init()`が`memblock_set_node()`を呼び出し、すべての`memblock.memory`領域にNUMAノード番号のタグを付ける。これによって、バディアロケータのノードごとのフリーリストがブート時に正しく構築される。

`memblock`がこれらの領域をどのように追跡し、バディアロケータに引き渡すかについては、[memblock: ブート時のメモリアロケータ](memblock.md)を参照。

### NUMA距離行列 {#numa-distance-matrix}

`numa_set_distance(from, to, distance)`は、memblockから割り当てられた平坦な`u8`配列に書き込む:

```c
/* mm/numa_memblks.c */
numa_distance[from * numa_distance_cnt + to] = distance;
```

実行時には、`node_distance(from, to)`（およびその下にある`__node_distance()`）がこの配列を読み、トポロジ上の近さに関するスケジューラやページアロケータからの問い合わせに答える。スケジューラは距離を使ってローカルなメモリ割り当てを優先し、ページアロケータはローカルノードが枯渇したときのフォールバックノードを選ぶのに使う。

```
SRAT parsing          SLIT parsing
     │                     │
     ▼                     ▼
numa_add_memblk()    numa_set_distance()
     │                     │
     ▼                     ▼
numa_meminfo{}       numa_distance[]
     │                     │
     ▼                     ▼
memblock_set_node()  __node_distance()
     │                     │
     ▼                     ▼
buddy allocator      scheduler / page alloc
per-node free lists  fallback node selection
```

## よくあるファームウェアのバグ {#common-firmware-bugs}

!!! warning "壊れたSRAT: メモリが何も言わずにノード0に割り当てられる"
    SRATにメモリエントリが欠けていると、カーネルは`numa_fill_memblks()`にフォールスルーし、これが帰属先のない物理範囲をノード0に割り当てる。4ソケットのシステムでは、3つのリモートノードのメモリがすべてノード0として現れることになる。システムは起動するがNUMAは完全に壊れている。どのリモートメモリも正しいノードからアクセスされず、`numactl -H`は単一の巨大なノードを表示する。

    検出方法: `numactl -H`が、バランスの取れた複数のノードではなく、1つの巨大なノードを報告する。ブートログには、すべてのメモリについて`SRAT: Node 0 PXM 0 [mem ...]`と表示される。

!!! warning "壊れたSLIT: すべての距離が10と報告される"
    一部のファームウェアは、非対角成分を含むSLIT行列全体を誤って値10（`LOCAL_DISTANCE`）で埋めてしまう。これは、すべてのノードが互いに等距離であるとファームウェアが主張していることを意味し、事実に反する。

    カーネルはこれを`slit_valid()`で検出する。これは対角成分が`LOCAL_DISTANCE`に等しく、非対角成分がそれより真に大きいことを確認する。このチェックに失敗したSLITは、次のメッセージとともに拒否される:

    ```
    SLIT table looks invalid. Not used.
    ```

    SLITが拒否されると、カーネルはデフォルトの距離、すなわち同一ノードには`LOCAL_DISTANCE`（10）、ノードをまたぐすべての組には`REMOTE_DISTANCE`（20）にフォールバックする。スケジューラとページアロケータは引き続き機能するが、ローカル/リモートという2段階の区別を超えたトポロジを意識した判断はできない。

!!! warning "まばらな、または連続していないPXM値"
    ACPI仕様では、PXM値がまばらであることが許されている。あるシステムがPXM 0、5、100を持ち、その間のPXMを持たないこともありうる。カーネルはこれを`acpi_map_pxm_to_node()`で扱い、PXMの隙間にかかわらず連続したLinuxのノード番号を割り当てる。ただし、そのようなシステムのSLITは`101 × 101`（PXM 0から100まで）のサイズでなければならず、存在しないPXMのエントリも含まれていなければならない。

    SLIT行列のサイズを誤るファームウェア、たとえば`101 × 101`ではなく`3 × 3`を使うファームウェアでは、カーネルが距離を変換する際にテーブルの末尾を越えて読み込んでしまう。カーネルは何も言わずにゴミの距離値を使うか、SLITを無効として拒否するかのいずれかになる。

!!! warning "SRATリビジョンの不一致"
    SRATリビジョン1では、近接ドメインをCPUアフィニティエントリの`proximity_domain_lo`フィールドの下位8ビットだけに詰め込む。リビジョン2からは、32ビット全体の`proximity_domain`が使われる。`header.revision = 1`を設定しているのに32ビットのPXMを書き込むファームウェアでは、上位ビットが何も言わずにマスクされる（`pxm &= 0xff`）。255を超えるPXMを共有するCPUとメモリは、PXM `pxm & 0xff`に属しているように見え、ノードの割り当てが壊れる。

## NUMAトポロジの検証 {#validating-numa-topology}

### 実行時のツール {#runtime-tools}

```bash
# ノードのサイズ、ノードごとのCPU、距離行列を表示する
numactl --hardware

# 2ノードシステムでの出力例:
# available: 2 nodes (0-1)
# node 0 cpus: 0 1 2 3 4 5 6 7
# node 0 size: 65536 MB
# node 0 free: 61234 MB
# node 1 cpus: 8 9 10 11 12 13 14 15
# node 1 size: 65536 MB
# node 1 free: 60987 MB
# node distances:
# node   0   1
#   0:  10  21
#   1:  21  10

# 特定の組の距離を読む
cat /sys/devices/system/node/node0/distance
# 10 21   （ノード0からノード0、次にノード0からノード1への距離）
```

### ブートログ {#boot-log}

ブートの初期段階で、ACPI NUMAのコードは受け入れた各SRATエントリをログに出力する:

```bash
dmesg | grep -i "SRAT"
# ACPI: SRAT: Node 0 PXM 0 [mem 0x00000000-0x3fffffffff]
# ACPI: SRAT: Node 1 PXM 1 [mem 0x4000000000-0x7fffffffff]
# ACPI: SRAT: Node 0 PXM 0 [mem 0x100000000-0x3fffffffff] hotplug
```

ここに期待されるノードが欠けていれば、SRATが壊れている明確な兆候である。

### 生のテーブルのデコード {#decoding-the-raw-tables}

詳しく調べるには、標準のツールチェーンで生のACPIテーブルをデコードする:

```bash
# すべてのACPIテーブルをバイナリファイルにダンプする
acpidump -o acpi.dat
acpixtract acpi.dat

# SRATとSLITを逆アセンブルする
iasl -d SRAT.dat
iasl -d SLIT.dat
# SRAT.dslとSLIT.dslが生成される — 人間が読めるテキスト

# ホットプラグのメモリ領域を確認する
grep -i "hot" SRAT.dsl
```

## ACPI以外のシステム {#non-acpi-systems}

ACPIベースのSRAT/SLITのパースは、x86とARM64で使われる。他のプラットフォームは別の方法でNUMAトポロジを検出する:

**Device Tree（ARM、RISC-V）**: トポロジはDTノードの`numa-node-id`プロパティで記述される。`of_numa_init()`がこれらを読み、SRATパーサと同じインターフェースで`numa_add_memblk()`を呼び出す。Device Treeのメモリ領域がどのようにmemblockに流れ込むかについては、[memblock: ブート時のメモリアロケータ](memblock.md)を参照。

**ACPIのないAMD（`amd_numa_init()`）**: ACPIのNUMAサポートが壊れているか存在しない古いAMDシステムでは、カーネルはメモリトポロジを得るためにAMDのノースブリッジを直接調べる。これは`x86_acpi_numa_init()`が失敗した場合にのみ実行されるフォールバックである。

**NUMAファームウェアなし（`dummy_numa_init()`）**: 他のすべての方法が失敗した場合（またはカーネルコマンドラインで`numa=off`が渡された場合）、`dummy_numa_init()`は物理アドレス空間全体をカバーする単一の偽のノード0を作成する。実際のハードウェアトポロジにかかわらず、システムはUMAマシンであるかのように動作する。

```c
/* arch/x86/mm/numa.c */
static int __init dummy_numa_init(void)
{
    node_set(0, numa_nodes_parsed);
    numa_add_memblk(0, 0, PFN_PHYS(max_pfn));
    return 0;
}
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [drivers/acpi/numa/srat.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/acpi/numa/srat.c) — `acpi_numa_init()`、`acpi_parse_slit()`、PXMからノードへの対応テーブル
- [arch/x86/mm/srat.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/srat.c) — x86のエントリポイント`x86_acpi_numa_init()`とAPICアフィニティのコールバック
- [arch/x86/mm/numa.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/numa.c) — `x86_numa_init()`による統括、`dummy_numa_init()`、NUMA初期化のフォールバックチェーン
- [mm/numa_memblks.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/numa_memblks.c) — `numa_add_memblk()`、`numa_set_distance()`、実行時の`numa_distance[]`行列
- [include/acpi/actbl3.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/acpi/actbl3.h) — `acpi_table_srat`、`acpi_srat_mem_affinity`、`acpi_table_slit`のC構造体定義

### ACPI仕様 {#acpi-specification}

- [ACPI Specification (latest)](https://uefi.org/specifications) — 第5章（ACPI Software Programming Model）で、SRAT（5.2.16節）とSLIT（5.2.17節）のテーブル形式が正式かつ詳細に記述されている

### LWNの記事 {#lwn-articles}

- [NUMA topology initialization](https://lwn.net/Articles/568950/) — ブート時にACPIテーブルがカーネルのNUMAノードモデルにどのように供給されるか
- [Memory hotplug and NUMA](https://lwn.net/Articles/523874/) — SRATからのホットプラグメモリ領域がメモリホットプラグサブシステムとどのように相互作用するか

### 関連ドキュメント {#related-docs}

- [NUMAメモリ管理](numa.md) — NUMAポリシー、自動バランシング、メモリティアリングの概要
- [NUMA距離とソケット間レイテンシ](numa-distance.md) — SLITから構築された`numa_distance[]`行列が、実行時にスケジューラとアロケータでどのように使われるか
- [memblock: ブート時のメモリアロケータ](memblock.md) — `memblock_set_node()`が、SRATのパースで登録されたノード範囲をどのように使うか
