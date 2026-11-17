# メモリ仮想化 {#memory-virtualization}

> KVMゲストのためのEPT、シャドウページング、バルーンドライバ、huge page

## 2段階のアドレス変換 {#two-layers-of-address-translation}

ゲストOSは自分が物理メモリを制御していると考えている。実際にはそうではない。ゲストOSが制御しているのは*ゲスト物理アドレス*（GPA）であり、KVMはそれを2段目のページテーブル層を使って*ホスト物理アドレス*（HPA）に変換する。

```
Guest virtual → Guest physical → Host physical
      (guest CR3)       (EPT / shadow PT)

Without hardware:  two separate walks, merged by software (shadow paging)
With EPT/NPT:      hardware does a 2D walk automatically
```

## Extended Page Tables（EPT / NPT） {#extended-page-tables-ept-npt}

Intel EPT（Extended Page Tables）とAMD NPT（Nested Page Tables）は、GPA→HPA変換のためのハードウェア機構である。これらにより、最近のハードウェアではシャドウページテーブルが不要になる。

### EPTのページウォーク {#ept-page-walk}

CPUは、GCR3（ゲストのCR3）とEPTポインタ（EPTP）の両方を同時にたどる。

```
Guest CR3 → PML4 → PDPT → PD → PT → GPA
                                        │
                           EPTP → EPT PML4 → EPT PDPT → EPT PD → EPT PT → HPA
```

4段のゲストが4段のEPTをたどる場合、最悪のケースでハードウェアは最大24個のページテーブルエントリにアクセスすることがある（ゲストの4回 + EPTの4×4回のウォーク）。

### struct kvm_mmu {#struct-kvm_mmu}

```c
/* 簡略化したもの — arch/x86/include/asm/kvm_host.h。いくつかのコールバックと
 * シャドウルートのフィールド（get_guest_pgd、get_pdptr、inject_page_fault、
 * gva_to_gpa、sync_spte、mirror_root_hpa、cpu_role、pkru_mask、
 * permissions[]、pml4_root、pml5_root、シャドウ/ゲストの予約ビットの
 * バリデータ）は省略している */
struct kvm_mmu {
    /* ページフォールトハンドラ: GPAのフォールト時に呼ばれる */
    int (*page_fault)(struct kvm_vcpu *vcpu,
                      struct kvm_page_fault *fault);

    /* ルート（最上位のEPT/シャドウPTの物理アドレス） */
    struct kvm_mmu_root_info root;   /* root.hpa, root.pgd */
    union kvm_mmu_page_role root_role;

    struct kvm_mmu_root_info prev_roots[KVM_MMU_NUM_PREV_ROOTS];

    /* PAEのルートページテーブル — 64ビットのホストMMUで32ビット/PAEの
     * ゲストのページングモードをシャドウする場合（例: NPT下）に必要 */
    u64                  *pae_root;
};
```

### EPT violationの処理 {#ept-violation-handling}

ゲストがEPTエントリのないメモリにアクセスすると、CPUはEPT violationのVM exitを発生させる。

```c
/* arch/x86/kvm/vmx/vmx.c — handle_ept_violation()、簡略化したもの */
static int handle_ept_violation(struct kvm_vcpu *vcpu)
{
    gpa_t gpa    = vmcs_read64(GUEST_PHYSICAL_ADDRESS);
    u64   exit_qual = vmcs_readl(EXIT_QUALIFICATION);
    u64   error_code = 0;

    /*
     * exit_qualのビット:
     *   ビット0: 読み込みフォールト
     *   ビット1: 書き込みフォールト
     *   ビット2: フェッチフォールト
     *   ビット7: GVAのアドレス変換中のGPA（直接アクセスではない）
     */
    if (exit_qual & EPT_VIOLATION_ACC_WRITE)
        error_code |= PFERR_WRITE_MASK;
    if (exit_qual & EPT_VIOLATION_ACC_INSTR)
        error_code |= PFERR_FETCH_MASK;
    /* （present/MBEC/GVA変換/TDXプライベートのビットはここでは省略） */

    /* HPAのマッピングを検索または作成する */
    return kvm_mmu_page_fault(vcpu, gpa, error_code, NULL, 0);
}

/* arch/x86/kvm/mmu/mmu.c — kvm_mmu_page_fault()、簡略化したもの（実際のシグネチャは
 * 可変長引数ではなく具体的な5つの引数をとる。staticではない。ソフトウェア保護VMの
 * 属性や書き込み保護フォールトも処理するが、ここでは省略） */
int noinline kvm_mmu_page_fault(struct kvm_vcpu *vcpu, gpa_t cr2_or_gpa,
                                 u64 error_code, void *insn, int insn_len)
{
    bool direct = vcpu->arch.mmu->root_role.direct;
    int r, emulation_type = EMULTYPE_PF;

    /* error_codeに予約ビットがセットされているのは、KVM自身のMMIOのヒント */
    if (error_code & PFERR_RSVD_MASK) {
        r = handle_mmio_page_fault(vcpu, cr2_or_gpa, direct);
        if (r == RET_PF_EMULATE)
            goto emulate;
    } else {
        /* EPTエントリを構築してフォールトの解決を試みる */
        r = kvm_mmu_do_page_fault(vcpu, cr2_or_gpa, error_code, false,
                                  &emulation_type, NULL);
    }

    if (r != RET_PF_EMULATE)
        return r;

emulate:
    /* アクセスをエミュレートする（例: デバイス領域へのMMIO） */
    return x86_emulate_instruction(vcpu, cr2_or_gpa, emulation_type,
                                   insn, insn_len);
}
```

### struct kvm_mmu_page（シャドウページ） {#struct-kvm_mmu_page-shadow-pages}

KVMは、EPT/シャドウページを`struct kvm_mmu_page`で追跡する。ページテーブルのページ1つにつき1つの構造体がある。

```c
struct kvm_mmu_page {
    struct list_head    link;          /* kvm->arch.active_mmu_pages内 */
    struct hlist_node   hash_link;     /* gfnによるハッシュテーブル */

    bool                unsync;        /* sptesが古くなっている可能性がある */
    u8                  mmu_valid_gen; /* 世代カウンタ */
    bool                nx_huge_page_disallowed; /* NX huge pageの緩和策の
                                                     ため、ここではhuge pageを
                                                     使えない */

    gfn_t               gfn;          /* このページがマップするゲストのフレーム番号 */
    union kvm_mmu_page_role role;      /* レベル、cr4_pae、アクセスビット、... */

    u64                *spt;          /* 実際のページテーブルのページ（4096バイト） */
    u64                *shadowed_translation; /* sptごと: シャドウされたGPA（上位
                                                  ビット）+ アクセス権（下位） */
    struct kvm_rmap_head parent_ptes;  /* 逆マップ: どこからここを指しているか */

    atomic_t            write_flooding_count;

    /* 上のactive_mmu_pagesとは独立した別のリスト（kvm->arch.possible_nx_huge_pages）:
     * iTLB multihitのエラッタを緩和するためだけにKVMが4Kに分割したページ
     * （そうしないと、実行可能なhuge PTEがハードウェアによって分割され、
     * 特定のCPUをハングさせ得る）。回復スレッド（kvm_recover_nx_huge_pages()）が
     * 周期的に再びzapし、KVMがhugeマッピングを再試行して緩和策による
     * TLBのコストを取り戻せるようにする。 */
    struct list_head    possible_nx_huge_page_link;
};
```

## シャドウページング（EPTなし） {#shadow-paging-without-ept}

EPTのない古いハードウェアでは、KVMはGVA→HPAを直接マップする**シャドウページテーブル**を維持する。ゲストのCR3は、ゲスト自身のページテーブルではなくシャドウページテーブルを指す。

```
Guest CR3 ──────────► Shadow PT (GVA → HPA, maintained by KVM)
                              ▲
Guest's own PT ──────────────┘
(GVA → GPA, never loaded)
```

### 書き込み保護によるトラップ {#the-write-protection-trap}

シャドウページングでは、KVMはゲストによるページテーブルへのすべての書き込みをインターセプトする必要がある。

1. KVMは、シャドウPT内のゲストのページテーブルページをすべて書き込み保護する（書き込み権限をクリアする）
2. ゲストが自身のページテーブルを変更しようとすると → 書き込みフォールト → VM exit
3. KVMは書き込みをエミュレートし、ゲストのPTとシャドウPTの両方を更新する
4. ゲストを再開する

これはページテーブル操作の多いワークロードではコストが高い。EPTはこれを完全に回避する。

## メモリスロット {#memory-slots}

KVMは、ゲストの物理メモリを*スロット*単位でマップする。スロットとは、ホストのユーザー空間メモリを裏付けとする連続したGPA範囲である。

```c
/* include/linux/kvm_host.h */
struct kvm_memory_slot {
    struct hlist_node   id_node[2];
    struct interval_tree_node hva_node[2];
    struct rb_node      gfn_node[2];

    gfn_t               base_gfn;   /* 開始GFN */
    unsigned long       npages;     /* ゲストページ数 */
    unsigned long      *dirty_bitmap;
    struct kvm_arch_memory_slot arch;

    unsigned long       userspace_addr; /* 裏付けとなるメモリのHVA */
    u32                 flags;          /* KVM_MEM_LOG_DIRTY_PAGESなど */
    short               id;
    u16                 as_id;
};
```

スロットは次のいずれかである。

- **通常のRAM**: VMM（例: QEMU）内の`mmap(MAP_ANONYMOUS)`メモリを裏付けとする
- **ROM**: 読み取り専用（BIOS、オプションROM）
- **MMIO**: 裏付けなし — アクセス時に`KVM_EXIT_MMIO`を発生させる

## ダーティページ追跡 {#dirty-page-tracking}

KVMは、どのゲストページが変更されたかを追跡できる（ライブマイグレーションで使われる）。

```c
/* スロットでダーティ追跡を有効にする */
struct kvm_userspace_memory_region region = {
    .slot  = 0,
    .flags = KVM_MEM_LOG_DIRTY_PAGES,  /* ダーティビットマップを有効にする */
    /* ... */
};
ioctl(vm_fd, KVM_SET_USER_MEMORY_REGION, &region);

/* ダーティビットマップを取得してクリアする（収集中はゲストの書き込みを一時停止する） */
struct kvm_dirty_log log = {
    .slot       = 0,
    .dirty_bitmap = bitmap,
};
ioctl(vm_fd, KVM_GET_DIRTY_LOG, &log);
```

### ダーティリング（5.11以降） {#dirty-ring-511}

ダーティリングは、ビットマップの走査を避ける、より高速な代替手段である。

```c
/* カーネルはダーティなGFNをvCPUごとのリングにエクスポートする */
struct kvm_dirty_ring {
    u32    dirty_index;   /* カーネルが書き込む */
    u32    reset_index;   /* VMMが読み込み/リセットする */
    u32    size;
    u32    soft_limit;
    struct kvm_dirty_gfn *dirty_gfns; /* mmapされたリング */
};

struct kvm_dirty_gfn {
    __u32 flags;
    __u32 slot;
    __u64 offset;  /* スロット内のページオフセット */
};
```

VMMはリングをポーリングし、ダーティなGFNを処理してリセットする。ビットマップの走査は不要であり、書き込み頻度の高いゲストでははるかに高速である。

## バルーンドライバ {#balloon-driver}

バルーンドライバは、ゲストにページテーブルの操作を気付かせることなく、ホストがゲストからメモリを回収するための準仮想化の仕組みである。

```
Host (KVM)              Guest (virtio-balloon driver)
    │                           │
    │  inflate request ─────────►
    │  (need N pages back)      │
    │                           │ allocate N pages from guest allocator
    │                           │ add their GFNs to balloon page list
    │                           │
    │  guest tells host GPAs ◄──┘
    │
    │  host removes EPT entries for those GPAs
    │  host can now use the physical memory for other guests
    │
    │  deflate request ──────────►
    │  (guest can have pages back) │
    │                              │ release pages back to guest allocator
```

### バルーンの統計 {#balloon-stats}

ゲストはメモリの統計をホストに報告することもできる。

```c
/* drivers/virtio/virtio_balloon.c、簡略化したもの。
 * update_stat()はタグだけでなく実行中のインデックスを受け取る — 実際の呼び出し側は
 * update_stat(vb, idx++, TAG, val)とする。実際のコードはこれを2つに分けている:
 * まずupdate_balloon_vm_stats()がvmイベント由来の統計（スワップの
 * イン/アウト、メジャー/マイナーフォールト、OOM kill、回収/スキャンの回数、hugetlbの
 * 統計 — ここでは省略）を埋めて個数を返し、下のupdate_balloon_stats()が
 * そこから続けてsysinfo由来の統計を追加する。どちらも
 * voidではなく、埋めたエントリの総数を返す。 */
static unsigned int update_balloon_stats(struct virtio_balloon *vb)
{
    struct sysinfo i;
    unsigned int idx;
    unsigned long caches;

    idx = update_balloon_vm_stats(vb);  /* SWAP_IN/OUT、MAJFLT、MINFLT、... */

    si_meminfo(&i);
    caches = global_node_page_state(NR_FILE_PAGES);

    update_stat(vb, idx++, VIRTIO_BALLOON_S_MEMFREE,
               pages_to_bytes(i.freeram));
    update_stat(vb, idx++, VIRTIO_BALLOON_S_MEMTOT,
               pages_to_bytes(i.totalram));
    update_stat(vb, idx++, VIRTIO_BALLOON_S_AVAIL,
               pages_to_bytes(si_mem_available()));
    update_stat(vb, idx++, VIRTIO_BALLOON_S_CACHES,
               pages_to_bytes(caches));

    return idx;
}
```

ホストはこれらの統計を使って、膨張/収縮の判断を賢く行う（例: ゲストに空きメモリがなければ膨張させない）。

## ゲストのためのhuge page {#huge-pages-for-guests}

EPTでhuge pageを使うと、TLBの効率が向上する。KVMは、2MBまたは1GBのGPA範囲を単一のEPTエントリでマップできる。

### ゲストメモリのためのTransparent huge page（THP） {#transparent-huge-pages-thp-for-guest-memory}

QEMUは通常、`mmap(MAP_ANONYMOUS)`でゲストのRAMを割り当てる。THPが有効であれば、ホストカーネルはゲストのページを自動的にhuge pageに昇格させることがあり、KVMは2MBのEPTエントリを使う。

```bash
# ホスト上で: 無名メモリにTHPを許可する
echo always > /sys/kernel/mm/transparent_hugepage/enabled

# ゲストのページがhugepageで裏付けられているか確認する
grep -i huge /proc/$(pgrep qemu)/smaps | head
# AnonHugePages: 1234567 kB
```

### hugetlbfsを裏付けとするゲストメモリ {#hugetlbfs-backed-guest-memory}

THPの負荷なしにhuge pageを保証するには、QEMUはhugetlbfsを使える。

```bash
# ホスト上で2MBのhugepageを512個（合計1GB）割り当てる
echo 512 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

# hugetlbfsのメモリでQEMUを起動する
qemu-system-x86_64 \
    -m 1G \
    -mem-path /dev/hugepages \
    -mem-prealloc \
    ...
```

EPTがこれらをマップすると、ゲスト物理メモリ2MBごとに、512個ではなく1個のEPTリーフエントリを使う。TLBミスが減る → メモリ集約的なワークロードでゲストの性能が向上する。

### KVMのラージページの扱い {#kvm-large-page-handling}

```c
/* arch/x86/kvm/mmu/mmu.c — kvm_mmu_hugepage_adjust()、簡略化したもの。
 * staticではなく、intではなくvoidを返す。実際のアラインメント/ホストのページサイズの
 * チェックは別のヘルパーkvm_mmu_max_mapping_level()で行われる（各memslotの
 * gfnごとのlpage_infoと実際のホストのページサイズをたどる）— ここでは再現していない。 */
void kvm_mmu_hugepage_adjust(struct kvm_vcpu *vcpu, struct kvm_page_fault *fault)
{
    struct kvm_memory_slot *slot = fault->slot;
    kvm_pfn_t mask;

    fault->huge_page_disallowed = fault->exec &&
                                  fault->nx_huge_page_workaround_enabled;

    if (fault->max_level == PG_LEVEL_4K)
        return;
    if (is_error_noslot_pfn(fault->pfn))
        return;
    if (kvm_slot_dirty_track_enabled(slot))
        return;   /* KVMは4KiB粒度でダーティログを取るので、スロットがダーティ追跡
                     されている場合、huge pageは最初の書き込みで4Kに分割される */

    /* GPAのアラインメントとホストのページの両方がサポートする最大のレベル: */
    fault->req_level = kvm_mmu_max_mapping_level(vcpu->kvm, fault,
                                                 fault->slot, fault->gfn);
    if (fault->req_level == PG_LEVEL_4K || fault->huge_page_disallowed)
        return;

    fault->goal_level = fault->req_level;
    mask = KVM_PAGES_PER_HPAGE(fault->goal_level) - 1;
    fault->pfn &= ~mask;
}
```

## ゲストメモリの観測 {#observing-guest-memory}

```bash
# 専用の"ept_violations"ファイルはない（実際の完全な一覧はx86.cの
# kvm_vcpu_stats_desc[]を参照）。pf_takenが利用できる最も近い代替指標である —
# kvm_mmu_page_fault()が処理するすべてのフォールトを数えるので、EPT violationを
# 含むが、それだけに限定されてはいない:
cat /sys/kernel/debug/kvm/*/vcpu0/pf_taken

# ゲストのTLBフラッシュ要求（これもvCPUごと、同じディレクトリ階層）
cat /sys/kernel/debug/kvm/*/vcpu0/tlb_flush

# NX huge pageの緩和策のためにKVMがhugeにできなかったシャドウページ
# （これはVMごとの統計で、VMのディレクトリ直下にある）
cat /sys/kernel/debug/kvm/*/nx_lpage_splits

# ホストからはゲストのRSSはQEMUプロセスとして見える
cat /proc/$(pgrep qemu)/status | grep VmRSS

# EPTが有効か確認する
cat /sys/module/kvm_intel/parameters/ept
# Y   (or N if disabled)

# テストのためにEPTを強制的に無効にする（シャドウページングへのフォールバックが必要）
modprobe kvm_intel ept=0
```

## ライブマイグレーションのためのKVMのダーティロギング {#kvm-dirty-logging-for-live-migration}

ライブマイグレーションでは、ゲストのメモリを移行先に転送しなければならない。KVMのダーティロギングは、前回の反復以降にどのページが書き込まれたかを追跡する。

```c
/* メモリスロットでダーティロギングを有効にする: */
struct kvm_dirty_log {
    __u32 slot;
    __u32 padding1;
    union {
        void __user *dirty_bitmap; /* ダーティビット用のユーザー空間バッファ */
        __u64 padding2;
    };
};

/* フェーズ1: ダーティロギングを有効にする */
struct kvm_userspace_memory_region region = {
    .slot  = 0,
    .flags = KVM_MEM_LOG_DIRTY_PAGES,  /* ダーティ追跡を有効にする */
    .guest_phys_addr = 0,
    .memory_size = 4ULL << 30,         /* 4GB */
    .userspace_addr = (uint64_t)vm_mem,
};
ioctl(vm_fd, KVM_SET_USER_MEMORY_REGION, &region);

/* フェーズ2: 反復的なコピー — ダーティページを転送し、ビットマップをクリアし、繰り返す */
void *dirty_bitmap = calloc(1, bitmap_size);
struct kvm_dirty_log dirty = {
    .slot = 0,
    .dirty_bitmap = dirty_bitmap,
};
ioctl(vm_fd, KVM_GET_DIRTY_LOG, &dirty);
/* これでdirty_bitmapには、前回の呼び出し以降に書き込まれた4Kページごとにビットがセットされている */
/* それらのページを移行先に転送し、繰り返す */
```

EPTハードウェアのダーティビット（EPT PTEのビット9）は、KVMがビットマップをリセットするときにクリアされる。次の書き込みアクセスでEPT violationが発生し → KVMはページをダーティとしてマークし、ダーティビットを再び有効にする。

### KVM_CAP_MANUAL_DIRTY_LOG_PROTECT2 {#kvm_cap_manual_dirty_log_protect2}

競合を避ける、非常に大きなVM向けの2段階のプロトコル:

```bash
# フェーズ1: ダーティページを取得する（カーネルは新たなダーティ化を捉えるため、それらを書き込み保護もする）
ioctl(vm_fd, KVM_GET_DIRTY_LOG, &dirty);
# それらのダーティページを移行先に転送する...
# フェーズ2: 転送したばかりのページのダーティビットをクリアする
ioctl(vm_fd, KVM_CLEAR_DIRTY_LOG, &clear);
```

## メモリのオーバーコミットとKSM {#memory-overcommit-and-ksm}

```bash
# KSM（Kernel Samepage Merging）: VM間で同一のページをマージする
echo 1 > /sys/kernel/mm/ksm/run         # 有効にする
echo 200 > /sys/kernel/mm/ksm/pages_to_scan  # ページ数/間隔
echo 100 > /sys/kernel/mm/ksm/sleep_millisecs

# 統計:
cat /sys/kernel/mm/ksm/pages_shared     # マージされたページ
cat /sys/kernel/mm/ksm/pages_sharing    # それらの共有ページを使っているもの
cat /sys/kernel/mm/ksm/pages_unshared   # マージ不可
# 節約量 = pages_sharing * 4KB

# QEMU: ゲストメモリでKSMを有効にする
# （自動: QEMUはゲストのRAMに対してmadvise(MADV_MERGEABLE)を呼ぶ）

# メモリバルーン: アイドルなゲストからメモリを回収する
# ゲスト内のvirtio-balloonドライバがホストにページの回収を伝える
# libvirt/QEMUがVM間でRAMをオーバープロビジョニングするために使う
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [arch/x86/include/asm/kvm_host.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/kvm_host.h) — `struct kvm_mmu`: vCPUごとのMMUコンテキスト（`root` — ルートのHPAを保持する`struct kvm_mmu_root_info` — 、`page_fault`ハンドラ、`root_role`）
- [arch/x86/kvm/mmu/mmu_internal.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kvm/mmu/mmu_internal.h) — `struct kvm_mmu_page`: シャドウ/EPTのページテーブルページごとに1つの構造体。逆マップ`parent_ptes`を含む
- [arch/x86/kvm/mmu/mmu.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kvm/mmu/mmu.c) — `kvm_mmu_page_fault()`と`kvm_mmu_hugepage_adjust()`: GPAのフォールト解決とhuge pageのレベル選択
- [arch/x86/kvm/vmx/vmx.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kvm/vmx/vmx.c) — `handle_ept_violation()`: VMXのEPT violationによるVM exitのハンドラ
- [include/linux/kvm_host.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/kvm_host.h) — `struct kvm_memory_slot`: ゲスト物理メモリのスロットのレイアウト
- [include/uapi/linux/kvm.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/kvm.h) — `struct kvm_userspace_memory_region`、`struct kvm_dirty_gfn`、および`KVM_SET_USER_MEMORY_REGION` / `KVM_MEM_LOG_DIRTY_PAGES`のioctl ABI
- [include/linux/kvm_dirty_ring.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/kvm_dirty_ring.h) — `struct kvm_dirty_ring`: vCPUごとのダーティGFNのリングバッファ（5.11以降）
- [drivers/virtio/virtio_balloon.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/virtio/virtio_balloon.c) — `update_balloon_stats()`: ホストに報告されるゲストのメモリ統計

### 関連ページ {#related-pages}

- [KVMアーキテクチャ](kvm-arch.md) — /dev/kvm API、vCPUの実行ループ
- [virtio](virtio.md) — 準仮想化I/O: virtqueue、virtio-net、virtio-blk
- [VFIO](vfio.md) — VMへのデバイスの直接パススルー
- [KVMライブマイグレーション](live-migration.md) — ダーティビットマップとダーティリングによる追跡が、プレコピーのマイグレーションループをどう駆動するか
- [メモリ管理: ページテーブル](../mm/page-tables.md) — ホスト側のページテーブル
- [メモリ管理: THP](../mm/thp.md) — KVMが利用するhuge pageへの昇格

### LWNの記事 {#lwn-articles}

- [KVM: Dirty ring interface](https://lwn.net/Articles/833784/) — ダーティビットマップに代わるvCPUごとのダーティリングを導入したPeter Xuのパッチ投稿

### 外部資料 {#external}

- [The Definitive KVM API Documentation](https://docs.kernel.org/virt/kvm/api.html) — メモリスロットのioctl（`KVM_SET_USER_MEMORY_REGION`）、ダーティロギング（`KVM_GET_DIRTY_LOG`、`KVM_CLEAR_DIRTY_LOG`）、ダーティリングのケーパビリティ
- [Kernel Samepage Merging](https://docs.kernel.org/admin-guide/mm/ksm.html) — `/sys/kernel/mm/ksm/`のsysfsの設定項目と、KSMが同一のゲストページを重複排除する仕組み
- [Transparent Hugepage Support](https://docs.kernel.org/admin-guide/mm/transhuge.html) — 上のゲストのhuge pageのセクションで参照したTHPの設定
