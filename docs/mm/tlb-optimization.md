# TLB最適化の技法 {#tlb-optimization-techniques}

> Linuxカーネルが、CPU間でTranslation Lookaside Bufferの一貫性を保つコストをどのように最小化しているか

## 主要なソースファイル {#key-source-files}

| ファイル | 役割 |
|------|---------|
| `arch/x86/include/asm/tlbflush.h` | `flush_tlb_mm()`、`flush_tlb_range()`、`flush_tlb_page()`、`struct tlb_state`、`struct flush_tlb_info` |
| `arch/x86/mm/tlb.c` | TLBフラッシュのバッチ化、lazy TLBモード、PCID/ASID管理、`flush_tlb_mm_range()`、`enter_lazy_tlb()`、`switch_mm_irqs_off()` |
| `include/asm-generic/tlb.h` | `struct mmu_gather`、`tlb_gather_mmu()`、`tlb_finish_mmu()`、`tlb_flush_mmu()` |
| `mm/mmu_gather.c` | mmu_gatherの実装: `tlb_gather_mmu()`、`tlb_gather_mmu_fullmm()`、`tlb_finish_mmu()`、`tlb_flush_mmu()` |

---

## なぜTLBが性能にとって重要か {#why-the-tlb-matters-for-performance}

Translation Lookaside Buffer（TLB）は、最近の仮想アドレスから物理アドレスへの変換結果を保持する小さなハードウェアキャッシュである。TLBがなければ、すべてのメモリアクセスで完全なページテーブルウォークが必要になる。これはページングの段数に応じて、通常3〜5回の追加メモリ読み出しを伴う。

### コストの差 {#the-cost-difference}

```
TLB hit:   ~1  cycle    (~0.3 ns)
TLB miss:  ~50 cycles   (~17 ns) + page table walk
Page walk: up to 4 serialized cache misses on a cold hierarchy
```

最近のx86では、L1 dTLBは4KBページ用に64〜96エントリ、2MBページ用に32エントリを持つ。第2レベルTLB（STLB）がさらに1024〜3072エントリを提供する。これらの数は、現実的なワーキングセットと比べるとはるかに小さい。

| ワーキングセット | 必要な4KBページ数 | 必要なTLBエントリ数 | TLBに収まるか |
|-------------|-----------------|---------------------|--------------|
| 4MB | 1,024 | 1,024 | いいえ（STLBの4倍） |
| 1GB | 262,144 | 262,144 | いいえ（STLBの256倍） |
| 100GB | 26,214,400 | 26,214,400 | いいえ |

大きな仮想アドレス空間、多数のVMA、あるいは頻繁な`mmap()`/`munmap()`を行うプロセスは、大きなTLB圧迫を生む。無効化しなければならない古いエントリと、その後に生じるミスの1つ1つがサイクル数として支払われ、マルチCPUシステムではさらにIPIレイテンシとしても支払われる。

---

## TLBフラッシュの基本 {#tlb-flush-basics}

カーネルは、CPUが誤ったデータを観測しうるような形で下位のマッピングが変更されたときには、必ずTLBエントリをフラッシュしなければならない。

- **ページのアンマップ**: 物理ページが解放された後に、有効な変換が使われてはならない。
- **権限の格下げ**: 例えばページを書き込み保護する場合、キャッシュされた古い読み書き可能エントリを追い出さなければならない。
- **PTEの変更**: present状態のPTEに対する、アクセス権を狭める、または物理アドレスを変えるあらゆる変更。

規則は *ページを解放する前にフラッシュし、決して後にはしない* である。

### 3つの中核フラッシュ関数（x86） {#the-three-core-flush-functions-x86}

これらは`arch/x86/include/asm/tlbflush.h`で定義されている。x86ではいずれも、下位の`flush_tlb_mm_range()`を包む薄いラッパーとして実装されている。

```c
/* 全CPU上でmmのアドレス空間全体をフラッシュする */
#define flush_tlb_mm(mm) \
    flush_tlb_mm_range(mm, 0UL, TLB_FLUSH_ALL, 0UL, true)

/* vma->vm_mm内のページ範囲をフラッシュする（huge pageのストライドを考慮） */
#define flush_tlb_range(vma, start, end) \
    flush_tlb_mm_range((vma)->vm_mm, start, end, \
                       ((vma)->vm_flags & VM_HUGETLB) \
                           ? huge_page_shift(hstate_vma(vma)) \
                           : PAGE_SHIFT, true)

/* 単一ページをフラッシュする */
static inline void flush_tlb_page(struct vm_area_struct *vma,
                                   unsigned long a)
{
    flush_tlb_mm_range(vma->vm_mm, a, a + PAGE_SIZE, PAGE_SHIFT, false);
}
```

カーネルマッピングについては、`flush_tlb_kernel_range(start, end)`がすべてのCPUにフラッシュを送る。カーネル空間にはプロセス単位のスコープが存在しない。

### リモートCPUへのIPI {#ipis-to-remote-cpus}

マッピングが変わると、変換をキャッシュしている可能性のあるすべてのCPUに通知しなければならない。カーネルはこのためにプロセッサ間割り込み（IPI）を使う。`flush_tlb_multi()`は、対象の`mm_cpumask(mm)`に含まれる全CPUへ`flush_tlb_func()`ハンドラを送る。各リモートCPUは`INVLPG`（単一ページ）またはCR3の再ロード（完全フラッシュ）を実行して古いエントリを追い出す。

IPIの配送には無視できないコストがある。負荷のかかったシステムでは、往復レイテンシが数マイクロ秒に達することがある。多数のソケットを持つNUMAシステムでは、このコストが倍増する。

---

## mmu_gather — バッチ化されたTLBフラッシュ {#mmu_gather-batched-tlb-flushing}

素朴に実装すると、N個のページをアンマップするにはN回の個別のTLBフラッシュが必要になる。カーネルは`mmu_gather`によってこれを回避する。これは操作ごとのアキュムレータであり、フラッシュすべきアドレス範囲をすべて集め、最後に1回のバッチ化されたフラッシュを発行する。

### 構造体 {#the-struct}

`include/asm-generic/tlb.h`で定義されている。

```c
struct mmu_gather {
    struct mm_struct    *mm;

    unsigned long        start;   /* アンマップされた最小アドレス */
    unsigned long        end;     /* アンマップされた最大アドレス */

    unsigned int         fullmm        : 1; /* mm全体を破棄中か */
    unsigned int         need_flush_all: 1; /* アーキテクチャが完全フラッシュを要求 */
    unsigned int         freed_tables  : 1; /* ページテーブルのページを解放した */
    unsigned int         delayed_rmap  : 1; /* 保留中のrmap削除 */

    /* どのページテーブル階層がクリアされたか */
    unsigned int         cleared_ptes : 1;
    unsigned int         cleared_pmds : 1;
    unsigned int         cleared_puds : 1;
    unsigned int         cleared_p4ds : 1;

    /* tlb_start_vma()時に取得したVMAフラグ */
    unsigned int         vma_exec : 1;
    unsigned int         vma_huge : 1;
    unsigned int         vma_pfn  : 1;

    unsigned int         batch_count;

    struct mmu_gather_batch *active;  /* 現在のページ解放バッチ */
    struct mmu_gather_batch  local;   /* スタック上の初期バッチ */
    struct page         *__pages[MMU_GATHER_BUNDLE]; /* 小さな初期配列 */
};
```

### ライフサイクル {#lifecycle}

gatherのライフサイクルは3つのステップからなり、`mm/mmu_gather.c`で実装されている。

```
tlb_gather_mmu(tlb, mm)          ← begin gathering; resets range and page list
    |
    | ... page-table walk, PTE clears, tlb_remove_page() calls ...
    | Each cleared PTE updates tlb->start / tlb->end via __tlb_adjust_range()
    |
tlb_finish_mmu(tlb)              ← issue TLB flush, free queued pages
```

`tlb_gather_mmu()`は構造体を初期化し（範囲を`[TASK_SIZE, 0]`にリセットし、ページバッチを空にする）、ネストしたTLBフラッシュが必要になりうることを示すために`mm->tlb_flush_pending`をインクリメントする。

`tlb_finish_mmu()`は`tlb_flush_mmu()`を呼び出し、これが次の処理を行う。
1. `tlb_flush_mmu_tlbonly()`を呼ぶ — 累積した範囲全体をカバーする`flush_tlb_range()`または`flush_tlb_mm()`を発行する。
2. バッチ化されたすべてのページを解放する（`tlb_batch_pages_flush()`）。

最後に`dec_tlb_flush_pending()`がカウンタを元に戻す。

mm全体を対象とする場合（`exec`、`exit`）、`tlb_gather_mmu_fullmm()`は`fullmm = 1`を設定する。これによりVMAごとの範囲追跡が省略される。どうせアドレス空間全体をカバーするフラッシュを狭めても意味がないからである。ASIDを持つアーキテクチャでは、`fullmm`によってIPIを送る代わりに新しいASIDへ切り替えることも可能になる。

### なぜバッチ化が重要か {#why-batching-matters}

1GBの匿名マッピング（262,144ページ）を`munmap()`する場合を考える。バッチ化がなければ次のようになる。

- 262,144回の`flush_tlb_page()` = SMPシステムで262,144回のIPI往復。
- IPI 1回あたり約1〜5 µs。合計で、IPIオーバーヘッドだけで1秒を超える可能性がある。

`mmu_gather`を使えば、カーネルはすべてのPTEをたどって範囲`[start, start+1GB)`を累積し、その後`flush_tlb_range()`を **1回** だけ発行する。この1回のIPIで、各リモートCPU上の範囲全体が一度にフラッシュされる。

!!!tip "性能に関するヒント"
    ワークロードが小さな`munmap()`呼び出しをタイトなループで多数行っている場合は、それらを1回の大きなアンマップにまとめるか、`madvise(MADV_FREE)`で置き換えることを検討するとよい。後者は実際のページ解放を遅延させ、即時のTLBシュートダウンを回避する。

---

## lazy TLBモード {#lazy-tlb-mode}

CPUがユーザー空間プロセスからカーネルスレッド（あるいはmmが異なる別のプロセス）へ切り替わるとき、直前のmmのためにすぐTLBをフラッシュする必要はない。カーネルスレッドから保護すべきユーザーマッピングは存在しないからである。早期にフラッシュするのはCPUサイクルの無駄である。

カーネルはこれを **lazy TLBモード** として活用している。これは`struct tlb_state_shared`（per-CPU、`cpu_tlbstate_shared`）の`is_lazy`フラグで追跡される。

```c
struct tlb_state_shared {
    bool is_lazy;
};
DECLARE_PER_CPU_SHARED_ALIGNED(struct tlb_state_shared, cpu_tlbstate_shared);
```

### lazyモードへの移行 {#entering-lazy-mode}

`enter_lazy_tlb()`は、カーネルスレッドへ切り替える際にスケジューラから呼ばれる。x86では次のとおり。

```c
void enter_lazy_tlb(struct mm_struct *mm, struct task_struct *tsk)
{
    if (this_cpu_read(cpu_tlbstate.loaded_mm) == &init_mm)
        return;

    this_cpu_write(cpu_tlbstate_shared.is_lazy, true);
}
```

`mm_cpumask(mm)`内のそのCPUのビットは **セットされたまま** である。CPUはcpumaskに残っているため、いずれフラッシュIPIを受け取ることになる。しかし`is_lazy`フラグは、ページテーブルが解放される場合を除いてIPIの送信を省略してよいことを送信側に伝える。

### lazyなCPUへのIPIの省略 {#skipping-ipis-to-lazy-cpus}

`native_flush_tlb_multi()`（`arch/x86/mm/tlb.c`）では、IPIはそのmmを実際に実行しているCPUにだけ送られる。

```c
if (info->freed_tables || mm_in_asid_transition(info->mm))
    on_each_cpu_mask(cpumask, flush_tlb_func, (void *)info, true);
else
    on_each_cpu_cond_mask(should_flush_tlb, flush_tlb_func,
                          (void *)info, 1, cpumask);
```

`should_flush_tlb()`は`is_lazy == true`のCPUに対して`false`を返す（ただしページテーブルが解放される場合は除く。その場合は、解放済みメモリの投機的なウォークを防ぐため、すべてのCPUをフラッシュしなければならない）。

lazyなCPUが次にユーザーコードを実行するとき、`switch_mm_irqs_off()`が`is_lazy`を`false`に戻し、`tlb_gen`を確認する。CPUがlazyであった間に世代カウンタが進んでいれば、その時点で完全フラッシュが発行される。

!!!tip "実際の意味"
    大規模マシン上のアイドルCPUやカーネルスレッドは、通常のページテーブル変更においてIPIコストをまったく生じない。この節約は、フラッシュ時点でカーネルスレッドを実行中またはアイドル状態のCPU数に比例して線形に増える。

---

## huge pageとTLBカバレッジ {#huge-pages-and-tlb-coverage}

2MBのhuge pageは、512個の個別の4KB PTEエントリではなく、PMDレベルの単一のTLBエントリでマップされる。エントリあたりのTLBカバレッジは512倍になる。

| ワーキングセット | 4KBページのTLBエントリ数 | 2MBページのTLBエントリ数 | 削減率 |
|-------------|---------------------|---------------------|-----------|
| 1GB | 262,144 | 512 | 512× |
| 4GB | 1,048,576 | 2,048 | 512× |

これがTransparent Huge Pages（THP）の主要な性能上の利点である。つまり、メモリ使用量の削減ではなく、TLB圧迫の軽減である。

### カーネルはhuge pageエントリをどう追跡するか {#how-the-kernel-tracks-huge-page-entries}

`struct mmu_gather`は、アンマップ操作中にどのページテーブル階層がクリアされたかを追跡する。

```c
unsigned int  cleared_ptes : 1;  /* 4KBページ */
unsigned int  cleared_pmds : 1;  /* 2MBページ */
unsigned int  cleared_puds : 1;  /* 1GBページ */
unsigned int  cleared_p4ds : 1;
```

`tlb_get_unmap_shift()`はこれらのビットを使って、アンマップされた最小のTLBエントリサイズを返す。これにより`tlb_flush()`は正しい`stride_shift`を`flush_tlb_range()`へ渡せる。huge pageのVMAでは、`flush_tlb_range()`はストライドとして`PMD_SHIFT`（21）を受け取り、4KBではなく2MB単位でフラッシュするようハードウェアに指示する。

`struct mmu_gather`の`vma_huge`ビットは、`VM_HUGETLB`のVMAを処理するときに`tlb_update_vma_flags()`によってセットされる。これにより、`tlb_finish_mmu()`の前に途中で`tlb_flush_mmu()`が発動した場合でも、フラッシュに正しいストライドを渡せる。

!!!tip "TLBを意識した割り当て"
    数ギガバイトのワーキングセットを持つワークロードでは、THPを有効にする（`/sys/kernel/mm/transparent_hugepage/enabled = always`）か、ホットな領域に`madvise(MADV_HUGEPAGE)`を使うことで、TLBミス率が劇的に下がる。大きく長寿命な割り当てでは、TLB圧迫の軽減が、メモリ断片化のコストを通常は上回る。

---

## PCID — Process Context Identifier {#pcid-process-context-identifiers}

PCIDがなければ、コンテキストスイッチのたびに（CR3に新しいページテーブルのルートを再ロードすることで）TLB全体がフラッシュされる。PCIDがあると、TLBにタグが付く。各エントリはそれを作ったプロセスのPCIDを持ち、CPUは複数のプロセスのエントリを同時に保持できる。コンテキストスイッチでフラッシュはもはや不要になり、切り替わってきたプロセスは単に自分のタグ付きエントリを使い始める。

### ハードウェアにおけるPCID {#pcid-in-the-hardware}

x86は最大4096個のPCID（CR3内の12ビットフィールド）をサポートする。既存のTLBエントリをフラッシュ *せずに* プロセスへ切り替えるため、カーネルはCR3への書き込み時に`CR3_NOFLUSH`ビット（ビット63）をセットする。

### LinuxのASIDモデル {#linuxs-asid-model}

Linuxは各プロセスにグローバルに一意なPCIDを与えるわけではない。代わりに、**動的ASID**（`TLB_NR_DYN_ASIDS = 6`）の小さなper-CPU配列を保持し、各CPU上で直近に実行されたいくつかのプロセスをキャッシュする。

```c
struct tlb_context {
    u64 ctx_id;   /* このスロットを所有するmm */
    u64 tlb_gen;  /* 最後にフラッシュしたときの世代 */
};

struct tlb_state {
    struct mm_struct *loaded_mm;
    u16               loaded_mm_asid;
    u16               next_asid;
    bool              invalidate_other;  /* 次のスイッチでロードされていないctxをフラッシュ */
    unsigned short    user_pcid_flush_mask;
    struct tlb_context ctxs[TLB_NR_DYN_ASIDS];
    /* ... */
};
DECLARE_PER_CPU_ALIGNED(struct tlb_state, cpu_tlbstate);
```

`choose_new_asid()`は`ctxs[]`を走査し、`ctx_id`が`next->context.ctx_id`と一致する既存スロットを探す。見つかれば`tlb_gen`が最新かどうかを確認し、最新であれば`need_flush = false`となり、CR3のロードには`CR3_NOFLUSH`が使われる。

```c
static inline unsigned long build_cr3_noflush(pgd_t *pgd, u16 asid,
                                               unsigned long lam)
{
    VM_WARN_ON_ONCE(!boot_cpu_has(X86_FEATURE_PCID));
    return build_cr3(pgd, asid, lam) | CR3_NOFLUSH;
}
```

世代が進んでいる場合（このCPUが別のプロセスを実行している間にTLBフラッシュが起きた場合）は`need_flush = true`となり、（`CR3_NOFLUSH`なしの）完全なCR3再ロードによって、そのASIDの古いエントリが追い出される。

### KPTIと2倍のPCID {#kpti-and-doubled-pcids}

#### なぜKPTIが存在するか {#why-kpti-exists}

2018年1月3日、Meltdown脆弱性（CVE-2017-5754）が公開された。根本原因はこうである。Linuxは性能のため、カーネルの仮想アドレス空間を常にすべてのプロセスのページテーブルにマップしていた。ユーザーモードからカーネルモードへの切り替え（システムコール、割り込み）ではCR3の再ロードが不要で、カーネルの仮想アドレスにはすでに到達可能だった。これにより、カーネルへの出入りのたびにTLBフラッシュを節約していた。

Meltdownは、ユーザープロセスがサイドチャネル（キャッシュタイミング）を用い、この共有マッピングを通じてハードウェアの権限ビットを迂回し、カーネルメモリを投機的に読み出せることを示した。修正策である **Kernel Page Table Isolation（KPTI）** は、概念的には単純だった。各プロセスに2組の別々のページテーブルを与えるのである。「ユーザー」ページテーブルにはカーネルマッピングがほとんどない（モード切り替えを処理するのに必要な小さなトランポリンだけ）。「カーネル」ページテーブルはカーネルモードにいる間だけ有効になる。

コストは深刻だった。すべてのシステムコールと割り込みで、CR3への書き込みが2回必要になった（入口でカーネルページテーブルへ切り替えるのに1回、戻るときに切り替え直すのに1回）。PCID以前のプロセッサでは、CR3への書き込みのたびにTLB全体がフラッシュされる。ベンチマークでは、システムコールの多いワークロードで5〜30%の性能リグレッションが示された。

#### PCIDがKPTIを許容可能にした {#pcid-made-kpti-bearable}

PCID（Process Context Identifiers）はLinux 4.7で導入されていたが、ほとんど使われていなかった。すべてのプロセスのページテーブルがすでにメモリ上にある状況では、コンテキストスイッチでのTLBフラッシュを避ける理由がほとんどなかったからである。Meltdownはこの計算を完全に変えた。

PCIDが有効だと、システムコールごとの2回のCR3書き込みはもはやTLBをフラッシュしない。カーネルとユーザーのページテーブルはそれぞれ異なるPCIDタグを持ち、CPUは両方を同時にTLBに保持する。カーネルへの入口ではカーネルのPCIDに切り替わり、ユーザー空間への復帰ではユーザーのPCIDに戻る。温まったTLBエントリはこの遷移を越えて生き残る。

Linux 4.14でKPTIとPCIDの両方が一緒に有効化された。PCID対応ハードウェア（Westmere以降）では、システムコールの多いワークロードでKPTIのオーバーヘッドが約30%から約5〜10%に下がった。PCIDを持たない古いハードウェア（依然としてCR3書き込みのたびにフラッシュが必要）ではKPTIは高コストのままであり、信頼できるワークロードでは管理者が無効化することもあった。

Kernel Page Table Isolation（KPTI）が有効な場合、各mmには2つのPCIDが必要になる。1つはユーザー空間のページテーブル用、もう1つはカーネル空間用である。これにより、利用可能なPCID空間は実質2048個のASIDに半減する（`PTI_CONSUMED_PCID_BITS = 1`）。

### グローバルASID（INVLPGB） {#global-asids-invlpgb}

`INVLPGB`（ブロードキャストTLB無効化）をサポートするAMDプロセッサでは、カーネルは4つ以上のCPU上で同時にアクティブなプロセスに **グローバルASID** を割り当てられる。グローバルASIDはすべてのCPUで同一であり、IPIなしでハードウェア支援によるリモートTLB無効化を可能にする。

```c
/* arch/x86/mm/tlb.cより */
static void consider_global_asid(struct mm_struct *mm)
{
    if (!cpu_feature_enabled(X86_FEATURE_INVLPGB))
        return;

    if (mm_active_cpus_exceeds(mm, 3))
        use_global_asid(mm);
}
```

その後`broadcast_tlb_flush()`は、`invlpgb_flush_user_nr_nosync()`に続けて`__tlbsync()`を使い、ハードウェアで全CPUにわたってTLBをフラッシュする。これにより、高度にマルチスレッド化されたプロセスでIPIオーバーヘッドが完全に排除される。

!!!tip "PCIDの利点の概要"
    同じ少数のプロセス間で頻繁にコンテキストスイッチが起きるワークロード（例えば、固定のワーカープールを持つWebサーバー）では、PCIDは、プロセスのASIDスロットがまだ有効かつ最新であるCPU上で、すべてのTLBフラッシュを回避する。節約量は、コンテキストスイッチごとのTLBウォームアップ時間に比例する。

---

## NUMAシステムにおけるTLBシュートダウンのコスト {#tlb-shootdown-cost-on-numa-systems}

マルチソケットのNUMAマシンでは、TLBシュートダウンのIPIはリモートCPUに届くためにソケット間インターコネクトを越えなければならない。こうしたソケット間IPIは、ソケット内のものよりかなり高コストである。

### コストの内訳 {#cost-anatomy}

64 CPU・4ソケットのマシンで、1回の`munmap()`に対するTLBシュートダウンは次のようになる。

1. **IPI送信**: 最大63個のCPUへ、NUMAインターコネクトを越えてブロードキャストする。
2. **IPI受信のオーバーヘッド**: 各CPUはIPIハンドラに入り、INVLPGまたはCR3再ロードを実行し、戻らなければならない。
3. **同期**: 開始側のCPUは、`flush_tlb_multi()`から戻る前に全員の応答を待つ。

`mmu_gather`のバッチ化はこのコストを直接償却する。IPIの数はアンマップされたページ数ではなく、異なるフラッシュイベントの数で上限が決まる。1回の`munmap()`でアンマップされるのが1ページでも1,000,000ページでも、必要なフラッシュIPIの往復は高々1回である。

### `tlb_single_page_flush_ceiling` {#tlbsinglepageflushceiling}

小さな範囲では、個別の`INVLPG`命令（1ページに1つ）のほうが完全フラッシュより安い。大きな範囲では完全フラッシュが有利になる。カーネルは`tlb_single_page_flush_ceiling`ページ（デフォルト: 33）を境に戦略を切り替える。

```c
/* arch/x86/mm/tlb.c */
unsigned long tlb_single_page_flush_ceiling __read_mostly = 33;
```

`get_flush_tlb_info()`の内部で、`(end - start) >> stride_shift > tlb_single_page_flush_ceiling`であれば、その範囲は`TLB_FLUSH_ALL`に格上げされる。

この閾値は`/sys/kernel/debug/x86/tlb_single_page_flush_ceiling`で調整できる。

!!!tip "シュートダウンコストの削減"
    - ホットパスでは **頻繁な小さいアンマップを避ける**。`munmap()`呼び出しをより大きな範囲にまとめる。
    - 可能な場合は`munmap()`の代わりに **`madvise(MADV_FREE)`を使う**。カーネルはメモリ圧迫が起きるまで実際のアンマップを遅延させ、即時のTLB無効化を回避する。
    - **`madvise(MADV_DONTNEED)`を使う** と、VMAをアンマップせずにページの内容を捨てられ、これもTLBシュートダウンの作業を遅延させる。
    - スレッド数の多いワークロードでは、カーネルがグローバルASID（AMD INVLPGB）を使ってIPIをハードウェアのブロードキャストフラッシュに置き換えられるかどうかの評価を検討するとよい。

---

## カーネルのTLB圧迫: vmallocとグローバルエントリ {#kernel-tlb-pressure-vmalloc-and-global-entries}

`vmalloc()`で割り当てられたカーネル仮想メモリは、グローバルなページテーブル領域を使う。カーネルマッピングはプロセスごとにタグ付けされず、共有されたカーネルPGDエントリを通じてすべてのアドレス空間に現れる。カーネルのTLBエントリをフラッシュするには、どのプロセスが実行中かにかかわらず **すべてのCPU** で無効化する必要がある。

### `flush_tlb_kernel_range()` {#flushtlbkernel_range}

```c
void flush_tlb_kernel_range(unsigned long start, unsigned long end)
```

この関数はシステム上のすべてのCPUにフラッシュを送る。64 CPUのマシンでは、カーネルTLBフラッシュを引き起こす`vfree()`のたびに63回のIPIがかかる。カーネルPTEの`_PAGE_GLOBAL`フラグは、これらのエントリが通常のユーザー空間のCR3再ロードでは追い出されないことを意味するため、IPIを回避する機会はない。

`INVLPGB`を持つプロセッサでは、`flush_tlb_kernel_range()`はIPIの代わりに`invlpgb_flush_addr_nosync()` + `__tlbsync()`を使い、CPUごとの割り込みオーバーヘッドを排除する。

### なぜvmallocは高コストか {#why-vmalloc-is-expensive}

```
vmalloc() call path:
  1. Allocate physical pages (GFP_KERNEL)
  2. Find a free range in the vmalloc virtual address space
  3. Map pages into kernel page tables
  4. flush_tlb_kernel_range() → IPI to ALL CPUs

vfree() call path:
  1. Unmap pages from kernel page tables
  2. flush_tlb_kernel_range() → IPI to ALL CPUs
  3. Free physical pages
```

`vmalloc()`/`vfree()`の組のたびに発生するグローバルフラッシュのため、vmallocはホットパスでの高頻度な割り当てには向かない。

!!!tip "vmallocのガイドライン"
    - **ホットパスではvmallocを避ける**。 約128KB未満のオブジェクトには`kmalloc()`（slab）を使う。これはダイレクトマップ領域内で動作し、グローバルなTLB無効化を必要としない。
    - **一度割り当てて再利用する大きなカーネルバッファ** であれば、vmallocでも許容できる。TLBコストは割り当て時と解放時に一度ずつ支払うだけである。
    - **`vmalloc_huge()`** はvmallocマッピングに2MBのhuge pageを使う。これにより必要なカーネルTLBエントリ数が減り、大きく永続的なvmalloc領域の性能が向上しうる。

---

## まとめ: TLB最適化戦略 {#summary-tlb-optimization-strategies}

| 技法 | 仕組み | カーネルでの実装 |
|-----------|-----------|----------------------|
| フラッシュのバッチ化 | 範囲を累積し、アンマップ操作1回につきフラッシュ1回 | `struct mmu_gather`、`tlb_gather_mmu()` / `tlb_finish_mmu()` |
| lazy TLBモード | カーネルスレッドを実行中のCPUへのIPIを省略 | `enter_lazy_tlb()`、`cpu_tlbstate_shared.is_lazy`、`should_flush_tlb()` |
| PCID / ASIDキャッシュ | フラッシュせずにコンテキストスイッチを越えて生き残る | `choose_new_asid()`、`build_cr3_noflush()`、`cpu_tlbstate.ctxs[]` |
| huge page | エントリあたり512倍のTLBカバレッジ | `tlb_update_vma_flags()`、`vma_huge`、PMDレベルのフラッシュストライド |
| グローバルASID（INVLPGB） | ハードウェアブロードキャスト。IPIなし | `use_global_asid()`、`broadcast_tlb_flush()` |
| 範囲か完全かのヒューリスティック | 33ページを超えたら完全フラッシュに切り替え | `tlb_single_page_flush_ceiling`、`get_flush_tlb_info()` |
| カーネルでのvmallocの回避 | slab割り当てではグローバルTLBフラッシュなし | ホットパスでは`vmalloc()`の代わりに`kmalloc()`を使う |

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [arch/x86/mm/tlb.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/tlb.c) — `flush_tlb_mm_range()`、`native_flush_tlb_multi()`、`enter_lazy_tlb()`、`switch_mm_irqs_off()`、PCID/ASID管理
- [arch/x86/include/asm/tlbflush.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/tlbflush.h) — `flush_tlb_mm()`、`flush_tlb_range()`、`flush_tlb_page()`、`struct tlb_state`
- [include/asm-generic/tlb.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/asm-generic/tlb.h) — `struct mmu_gather`、`tlb_gather_mmu()`、`tlb_finish_mmu()`
- [mm/mmu_gather.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mmu_gather.c) — mmu_gatherの実装: バッチ化、範囲の累積、ページの解放

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/arch/x86/tlb.rst`](https://docs.kernel.org/arch/x86/tlb.html) — x86のTLBフラッシュの設計ノート: 特定のシュートダウン経路が選ばれる理由

### 関連ページ {#related-pages}

- [page-tables.md](page-tables.md) — TLBエントリがキャッシュするページテーブル階層
- [thp.md](thp.md) — huge pageが512倍大きなエントリを提供することでTLB圧迫をどう軽減するか
- [boot-page-tables.md](boot-page-tables.md) — PCIDの有効化と、ASIDサポートを初期化する初期のCR3設定

### LWN記事 {#lwn-articles}

- [LWN: Improving page-fault scalability](https://lwn.net/Articles/741937/) — PCIDの利点とコンテキストスイッチ時のTLBコスト削減を扱う
- [LWN: Meltdown and Spectre, part 2](https://lwn.net/Articles/743287/) — KPTIがTLB性能に与える影響と、PCIDを2倍使う回避策
- [LWN: INVLPGB and broadcast TLB flushing](https://lwn.net/Articles/951489/) — AMD INVLPGBのサポート: TLB無効化のためのCPU間IPIの排除
