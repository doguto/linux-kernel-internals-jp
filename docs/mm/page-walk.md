# プログラムからページテーブルを走査する {#walking-page-tables-programmatically}

> pagewalk API、apply_to_page_range、ptdump — カーネルコードからページテーブルを読む

## なぜページテーブルを走査するのか {#why-walk-page-tables}

カーネルは、さまざまな場面でページテーブルエントリをプログラムから調べたり変更したりする必要がある。

- **KVM**: ゲストのメモリを走査してdirtyなpageを見つける
- **マイグレーション**: pageを移動する前に、そのpageをマップしているすべてのPTEを見つける
- **デバッグ**: `ptdump` はページテーブルを `/sys/kernel/debug/kernel_page_tables` にダンプする
- **mprotect**: VA範囲の保護ビットを変更する
- **userfaultfd**: 範囲に書き込み保護を設定する
- **NUMA balancing**: PTEを走査して、NUMAノードをまたいだホット/コールドなpageを検出する

## pagewalk API {#the-pagewalk-api}

```c
#include <linux/pagewalk.h>

/* 走査処理がページテーブル階層をたどる際に呼ばれるコールバック */
struct mm_walk_ops {
    /* 各PGDエントリ（最上位）に対して呼ばれる */
    int (*pgd_entry)(pgd_t *pgd, unsigned long addr, unsigned long next,
                     struct mm_walk *walk);

    /* 各P4D（5段ページング）に対して呼ばれる。4段の場合はまったく呼ばれない */
    int (*p4d_entry)(p4d_t *p4d, unsigned long addr, unsigned long next,
                     struct mm_walk *walk);

    /* 各PUDエントリに対して呼ばれる */
    int (*pud_entry)(pud_t *pud, unsigned long addr, unsigned long next,
                     struct mm_walk *walk);

    /* 各PMDエントリに対して呼ばれる */
    int (*pmd_entry)(pmd_t *pmd, unsigned long addr, unsigned long next,
                     struct mm_walk *walk);

    /* 各PTE（リーフ。最もよく使われる）に対して呼ばれる */
    int (*pte_entry)(pte_t *pte, unsigned long addr, unsigned long next,
                     struct mm_walk *walk);

    /* ホール（VMAが見つからない）に対して呼ばれる: */
    int (*pte_hole)(unsigned long addr, unsigned long next,
                    int depth, struct mm_walk *walk);

    /* ヒュージページのPMD/PUDに入る前後に呼ばれる */
    int (*hugetlb_entry)(pte_t *pte, unsigned long hmask,
                         unsigned long addr, unsigned long next,
                         struct mm_walk *walk);

    /* 各VMAについて、その範囲を走査する前に呼ばれる */
    int (*pre_vma)(unsigned long start, unsigned long end,
                   struct mm_walk *walk);

    /* 各VMAについて、その範囲を走査した後に呼ばれる */
    void (*post_vma)(struct mm_walk *walk);

    /* このVMAを走査するかどうかを判定する（0を返す=スキップ、1=走査） */
    int (*test_walk)(unsigned long start, unsigned long end,
                     struct mm_walk *walk);
};

/* mm_walkはコールバック間でコンテキストを運ぶ */
struct mm_walk {
    const struct mm_walk_ops *ops;
    struct mm_struct         *mm;
    pgd_t                    *pgd;   /* 現在走査中のPGD */
    struct vm_area_struct    *vma;   /* 現在のVMA */
    void                     *private; /* 呼び出し側のプライベートデータ */
    bool                      no_vma;  /* VMAを参照せずに走査する */
};
```

## ユーザーのアドレス範囲を走査する {#walk-a-user-address-range}

```c
static int count_present_pte(pte_t *pte, unsigned long addr,
                               unsigned long next, struct mm_walk *walk)
{
    long *count = walk->private;
    if (pte_present(*pte))
        (*count)++;
    return 0;
}

static const struct mm_walk_ops count_ops = {
    .pte_entry = count_present_pte,
};

long count_resident_pages(struct mm_struct *mm, unsigned long start,
                           unsigned long end)
{
    long count = 0;
    mmap_read_lock(mm);
    walk_page_range(mm, start, end, &count_ops, &count);
    mmap_read_unlock(mm);
    return count;
}
```

## カーネルのアドレス空間を走査する {#walk-the-kernel-address-space}

```c
/* カーネルのページテーブルを走査する（mmもVMAもなし）: */
static int dump_pte_entry(pte_t *pte, unsigned long addr,
                           unsigned long next, struct mm_walk *walk)
{
    if (pte_present(*pte)) {
        pr_info("  %016lx: PTE=%016llx pfn=%lx %s%s%s\n",
                addr, (u64)pte_val(*pte),
                pte_pfn(*pte),
                pte_write(*pte) ? "W" : "r",
                pte_exec(*pte) ? "X" : "-",
                pte_dirty(*pte) ? "D" : "-");
    }
    return 0;
}

static const struct mm_walk_ops kernel_walk_ops = {
    .pte_entry = dump_pte_entry,
};

/* カーネルのvmalloc範囲を走査する: */
walk_page_range_novma(&init_mm, VMALLOC_START, VMALLOC_END,
                       &kernel_walk_ops, NULL, NULL);
```

## 走査中にPTEを変更する {#modifying-ptes-during-the-walk}

```c
/* 範囲を書き込み保護する（userfaultfd、KVMのdirty追跡で使われる）: */
static int wp_pte(pte_t *pte, unsigned long addr,
                  unsigned long next, struct mm_walk *walk)
{
    struct vm_area_struct *vma = walk->vma;
    spinlock_t *ptl = pte_lockptr(walk->mm, pte);

    spin_lock(ptl);

    if (pte_present(*pte) && pte_write(*pte)) {
        pte_t old_pte = *pte;
        pte_t new_pte = pte_wrprotect(old_pte);

        /* PTEをアトミックに更新する */
        set_pte_at(walk->mm, addr, pte, new_pte);

        /* このアドレスのTLBをフラッシュする */
        flush_tlb_page(vma, addr);
    }

    spin_unlock(ptl);
    return 0;
}
```

## apply_to_page_range: マッピングの設定 {#applytopage_range-set-up-mappings}

`apply_to_page_range` は必要に応じてページテーブル用のpageを割り当て、各PTEに対して関数を呼び出す。

```c
/* シグネチャ: */
int apply_to_page_range(struct mm_struct *mm, unsigned long addr,
                         unsigned long size, pte_fn_t fn, void *data);

/* 例: 物理ページの範囲をカーネルのvmalloc空間にマップする */
static int set_pte_fn(pte_t *pte, unsigned long addr,
                       unsigned long next, void *data)
{
    phys_addr_t phys = *(phys_addr_t *)data + (addr - start_addr);
    pte_t new_pte = pfn_pte(phys >> PAGE_SHIFT,
                             PAGE_KERNEL);  /* RW, NX */
    set_pte_at(&init_mm, addr, pte, new_pte);
    return 0;
}

apply_to_page_range(&init_mm, vaddr, size, set_pte_fn, &phys_base);
```

## ptdump: ページテーブルのデバッグ {#ptdump-debugging-page-tables}

```bash
# カーネルのページテーブルのダンプ（CONFIG_PTDUMP_DEBUGFS=y が必要）:
cat /sys/kernel/debug/kernel_page_tables | head -50
# 0xffff888000000000-0xffffc87fffffffff   131072G PGD  RW NX  SZ=1G
# 0xffffc90000000000-0xffffe8ffffffffff     32768G PGD  RW NX  SZ=2M
# 0xffffea0000000000-0xffffeb7fffffffff      3072G PGD  RW NX  SZ=2M  dirty
# 0xffffffff80000000-0xffffffff9fffffff       512M PGD  ro  X  SZ=2M  (kernel text)
# 0xffffffff9fffffff-0xffffffffa0000000         0 PGD (hole)
# 0xffffffffa0000000-0xffffffffb0000fffff      ~1G PGD  RW NX  (modules)

# ユーザープロセスのページテーブル:
cat /sys/kernel/debug/page_tables/pid_<PID>

# ページテーブルエントリを手動でデコードする:
python3 -c "
pte = 0x8000000004021025  # from ptdump
pfn = (pte >> 12) & 0xFFFFFFFFF
present = pte & 1
rw = (pte >> 1) & 1
user = (pte >> 2) & 1
nx = pte >> 63
print(f'PFN: {pfn:#x}  present={present} rw={rw} user={user} NX={nx}')
"
```

## PTEのビットを操作する関数 {#pte-bit-manipulation-functions}

```c
/* PTEの属性を読む: */
pte_present(pte)      /* ページがRAM上にある（Pビットが立っている） */
pte_write(pte)        /* 書き込み可能 */
pte_exec(pte)         /* 実行可能（x86ではNXビットがクリア） */
pte_dirty(pte)        /* ページが書き込まれた（ハードウェアが設定する） */
pte_young(pte)        /* ページがアクセスされた（ハードウェアが設定する） */
pte_soft_dirty(pte)   /* ソフトウェアによるdirty追跡ビット */
pte_special(pte)      /* 特殊なマッピング（struct pageなし） */
pte_devmap(pte)       /* デバイスメモリ（ZONE_DEVICE） */
pte_pfn(pte)          /* 物理フレーム番号 */

/* PTEの属性を変更する: */
pte_mkwrite(pte)          /* 書き込み可能にする */
pte_wrprotect(pte)        /* 書き込み可能をクリアする */
pte_mkdirty(pte)          /* dirtyにする */
pte_mkclean(pte)          /* dirtyをクリアする */
pte_mkyoung(pte)          /* accessedにする */
pte_mkold(pte)            /* accessedをクリアする */
pte_mkspecial(pte)        /* specialにする */

/* PFNと保護フラグからPTEを作る: */
pte_t pte = pfn_pte(pfn, PAGE_KERNEL);      /* カーネル RW NX */
pte_t pte = pfn_pte(pfn, PAGE_KERNEL_EXEC); /* カーネル RX */
pte_t pte = pfn_pte(pfn, PAGE_SHARED);      /* ユーザー RW */

/* PTEからstruct pageを得る: */
struct page *page = pte_page(pte);           /* pfn_to_page()経由 */
```

## ヒュージページのPTE {#huge-page-ptes}

```c
/* PMDレベルのヒュージページ（2MB）: */
pmd_t *pmd = pmd_offset(pud, addr);
if (pmd_large(*pmd)) {
    /* これはPTへのポインタではなく、2MBのリーフPTEである */
    phys_addr_t phys = (phys_addr_t)pmd_pfn(*pmd) << PAGE_SHIFT;
    phys += addr & ~PMD_MASK;  /* 2MBページ内のオフセット */
}

/* PUDレベルのヒュージページ（1GB）: */
pud_t *pud = pud_offset(p4d, addr);
if (pud_large(*pud)) {
    /* 1GBのリーフ */
    phys_addr_t phys = (phys_addr_t)pud_pfn(*pud) << PAGE_SHIFT;
    phys += addr & ~PUD_MASK;
}
```

## NUMA balancingのページテーブル走査 {#numa-balancing-page-table-scan}

NUMAバランサは、マイグレーションすべきpageを見つけるためにページテーブルの走査を使う。

```c
/* kernel/sched/fair.c（簡略化） */
static void task_numa_work(struct callback_head *work)
{
    struct task_struct *p = current;
    struct mm_struct *mm = p->mm;
    unsigned long start = p->mm->numa_scan_offset;

    /* プロセスのアドレス空間の一部分を走査する: */
    walk_page_range(mm, start, start + NUMA_SCAN_SIZE,
                    &numa_walk_ops, p);

    /* numa_walk_ops.pte_entryは次のことを行う:
     *   - pte_mkold()でAccessedビットをクリアする
     *   - 次のアクセスでCPUがAccessedビットを立てる → NUMAフォールト
     *   - task_numa_fault()がアクセスのNUMAノードを記録する
     *   - スケジューラはこれを使い、タスクをそのホットなページの
     *     近くで実行することを優先する
     */
}
```

## 参考文献 {#further-reading}

- [ページテーブル](page-tables.md) — ページテーブルの構造と形式
- [ページフォールトハンドラ](page-fault.md) — フォールトがどう解決されるか
- [NUMA](numa.md) — ページ走査を使うNUMA balancing
- [KVMのメモリ](../virtualization/kvm-memory.md) — dirtyページの追跡
- [userfaultfd](userfaultfd.md) — ページ走査による書き込み保護
- `mm/pagewalk.c` — pagewalkの実装
- `arch/x86/mm/dump_pagetables.c` — ptdumpのdebugfs
