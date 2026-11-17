# ページフォールトハンドラ {#page-fault-handler}

> ハードウェア例外からページのマッピングまで：フォールト処理パスの全体像

## ハードウェアページフォールト {#hardware-page-fault}

CPUがページテーブルにマッピングされていない仮想アドレスに遭遇すると、**ページフォールト例外**（x86では#PF）を発生させる。

```
CPU accesses virtual address 0x7fff1234
  → page table walk: PTE not present
  → hardware raises exception #14 (Page Fault)
  → saves RIP, RFLAGS, error code on stack
  → jumps to IDT entry 14 → kernel's fault handler
```

## x86のページフォールトエラーコード {#x86-page-fault-error-codes}

CPUがスタックに積むエラーコードがフォールトの内容を表す。

```
Bit 0 (P):   0 = not present (PTE absent)
             1 = protection violation (present but wrong permissions)
Bit 1 (W):   0 = read access
             1 = write access
Bit 2 (U):   0 = kernel mode access
             1 = user mode access
Bit 3 (R):   0 = non-reserved bit fault
             1 = reserved bit set in PTE
Bit 4 (I):   0 = data access
             1 = instruction fetch (NX violation)
Bit 5 (PK):  1 = protection key violation
Bit 15 (SS): 1 = shadow stack access
```

```c
/* arch/x86/mm/fault.c */
#define X86_PF_PROT   (1 << 0)   /* PTEは存在: 保護違反 */
#define X86_PF_WRITE  (1 << 1)   /* 書き込みアクセス */
#define X86_PF_USER   (1 << 2)   /* ユーザー空間から */
#define X86_PF_RSVD   (1 << 3)   /* PTEの予約ビット */
#define X86_PF_INSTR  (1 << 4)   /* 命令フェッチ */
#define X86_PF_PK     (1 << 5)   /* 保護キー */
```

`CR2`レジスタには、フォールト発生時の**フォールトを起こした仮想アドレス**が格納される。

## カーネルのエントリポイント {#kernel-entry-point}

```c
/* arch/x86/mm/fault.c */
DEFINE_IDTENTRY_RAW_ERRORCODE(exc_page_fault)
{
    unsigned long address = read_cr2();  /* フォールトを起こした仮想アドレス */

    irqentry_state_t state = irqentry_enter(regs);
    handle_page_fault(regs, error_code, address);
    irqentry_exit(regs, state);
}

static __always_inline void
handle_page_fault(struct pt_regs *regs, unsigned long error_code,
                   unsigned long address)
{
    if (unlikely(kmmio_fault(regs, address)))
        return;  /* MMIOトレース: 処理済み */

    /* フォールトはカーネル空間で起きたか? */
    if (unlikely(fault_in_kernel_space(address)))
        do_kern_addr_fault(regs, error_code, address);
    else
        do_user_addr_fault(regs, error_code, address);
}
```

## ユーザー空間のフォールトパス {#user-space-fault-path}

```c
/* arch/x86/mm/fault.c */
static void
do_user_addr_fault(struct pt_regs *regs, unsigned long error_code,
                    unsigned long address)
{
    struct vm_area_struct *vma;
    struct task_struct *tsk = current;
    struct mm_struct *mm = tsk->mm;
    vm_fault_t fault;

    /* 1. 明らかに非カノニカルなアドレスを確認 */
    if (unlikely(error_code & X86_PF_RSVD))
        goto bad_area;

    /* 2. mmap_lockを書き込みで保持している間はフォールトを処理しない */
    if (unlikely(faulthandler_disabled() || !mm))
        goto bad_area_nosemaphore;

    /* 3. 'address'を含むはずのVMAを探す */
    vma = lock_vma_under_rcu(mm, address);
    if (!vma) {
        /* VMAなし: mmap_lockのパスを試す */
        mmap_read_lock(mm);
        vma = find_vma(mm, address);
    }

    /* 4. VMAがフォールトアドレスをカバーしているか確認 */
    if (unlikely(!vma || address < vma->vm_start))
        goto bad_area;

    /* 5. VMAフラグに対してアクセス権を確認 */
    if (unlikely(access_error(error_code, vma)))
        goto bad_area;

    /* 6. フォールトを処理 */
    fault = handle_mm_fault(vma, address, flags, regs);

    /* 7. フォールトの完了状態を確認 */
    if (fault_signal_pending(fault, regs))
        return;  /* フォールト処理中にシグナルが到着 */

    if (unlikely(fault & VM_FAULT_ERROR)) {
        if (fault & VM_FAULT_OOM)
            goto out_of_memory;
        if (fault & VM_FAULT_SIGBUS)
            goto do_sigbus;
        if (fault & VM_FAULT_SIGSEGV)
            goto bad_area;
    }
    return;

bad_area:
    /* VMAが見つからない、またはアクセス拒否 */
    force_sig_fault(SIGSEGV, SEGV_MAPERR, (void __user *)address);
    return;

do_sigbus:
    /* ハードウェアエラーまたはVM_FAULT_HWPOISON */
    force_sig_fault(SIGBUS, BUS_ADRERR, (void __user *)address);
}
```

## フォールトの種類とその処理 {#fault-types-and-their-handling}

```c
/* mm/memory.c */
vm_fault_t handle_mm_fault(struct vm_area_struct *vma,
                             unsigned long address,
                             unsigned int flags,
                             struct pt_regs *regs)
{
    /* まずhuge pageのフォールトを処理 */
    if (unlikely(is_vm_hugetlb_page(vma)))
        return hugetlb_fault(vma->vm_mm, vma, address, flags);

    /* THP: transparent huge pageのフォールト */
    if (pmd_none(*vmf.pmd) && hugepage_vma_check(vma, vm_flags))
        return create_huge_pmd(vmf);

    return handle_pte_fault(vmf);
}

static vm_fault_t handle_pte_fault(struct vm_fault *vmf)
{
    pte_t entry;

    if (unlikely(pmd_none(*vmf->pmd))) {
        vmf->pte = NULL;
    } else {
        vmf->pte = pte_offset_map_lock(vmf->vma->vm_mm, vmf->pmd,
                                         vmf->address, &vmf->ptl);
        entry = *vmf->pte;
    }

    if (!vmf->pte || pte_none(entry)) {
        /* PTEなし: 匿名またはファイルバックのフォールト */
        if (vma_is_anonymous(vmf->vma))
            return do_anonymous_page(vmf);  /* デマンドゼロページ */
        else
            return do_fault(vmf);           /* ファイルバック: filemap_fault */
    }

    if (!pte_present(entry)) {
        /* PTEはあるがページが存在しない: スワップアウト済み */
        return do_swap_page(vmf);           /* ページをスワップイン */
    }

    /* PTEが存在: 読み取り専用PTEへの書き込みフォールトのはず */
    if (vmf->flags & FAULT_FLAG_WRITE) {
        if (!pte_write(entry))
            return do_wp_page(vmf);         /* コピーオンライト */
    }

    /* スプリアスフォールトまたはsoft dirty追跡 */
    return 0;
}
```

## フォールトの種類のまとめ {#fault-types-summary}

| PTEの状態 | アクセス種別 | ハンドラ | 説明 |
|-----------|------------|---------|-------------|
| PTEなし、匿名VMA | 読み取り/書き込み | `do_anonymous_page` | デマンドゼロページ |
| PTEなし、ファイルVMA | 読み取り | `do_read_fault` → `filemap_fault` | ファイルからのデマンドページング |
| PTEなし、ファイルVMA | 書き込み | `do_cow_fault` | ファイルからのコピーオンライト |
| スワップPTE | 任意 | `do_swap_page` | ディスクからスワップイン |
| 存在、読み取り専用 | 書き込み | `do_wp_page` | COWまたは共有dirty化 |
| VMAなし | 任意 | SIGSEGV | セグメンテーション違反 |
| VMA: 書き込み権限なし | 書き込み | SIGSEGV | アクセス拒否 |
| VMA: 実行権限なし | フェッチ | SIGSEGV | NX違反 |

## do_anonymous_page: デマンドゼロページ {#do_anonymous_page-demand-zero-page}

```c
/* mm/memory.c */
static vm_fault_t do_anonymous_page(struct vm_fault *vmf)
{
    struct vm_area_struct *vma = vmf->vma;
    struct page *page;
    pte_t entry;

    if (read_fault) {
        /* 未書き込みの匿名ページの読み取り: ゼロページをマップ */
        /* （共有の物理ゼロページ — 後で書き込まれたらコピーオンライト） */
        entry = pte_mkspecial(pfn_pte(my_zero_pfn(vmf->address),
                                       vmf->vma->vm_page_prot));
        set_pte_at(vmf->vma->vm_mm, vmf->address, vmf->pte, entry);
        return 0;
    }

    /* 書き込み: ゼロクリアした実ページを割り当てる */
    page = alloc_zeroed_user_highpage_movable(vma, vmf->address);
    if (!page)
        return VM_FAULT_OOM;

    /* PTEを設定 */
    entry = mk_pte(page, vma->vm_page_prot);
    entry = pte_sw_mkyoung(entry);
    if (vma->vm_flags & VM_WRITE)
        entry = pte_mkwrite(pte_mkdirty(entry));

    set_pte_at(vmf->vma->vm_mm, vmf->address, vmf->pte, entry);
    return 0;
}
```

## do_wp_page: コピーオンライト {#do_wp_page-copy-on-write}

```c
/* mm/memory.c */
static vm_fault_t do_wp_page(struct vm_fault *vmf)
{
    struct vm_area_struct *vma = vmf->vma;
    struct page *old_page = vm_normal_page(vma, vmf->address, vmf->orig_pte);

    if (PageAnon(old_page)) {
        /* 匿名ページ: 再利用できるか? */
        if (page_count(old_page) == 1) {
            /* 所有者は自分だけ: その場で書き込み可能にする */
            wp_page_reuse(vmf);
            return VM_FAULT_WRITE;
        }
        /* 共有されている: 新しいコピーを割り当てる */
        return wp_page_copy(vmf);
    }

    if (vma->vm_flags & VM_SHARED) {
        /* 共有マッピング: その場でページをdirtyにする */
        return wp_page_shared(vmf);
    }

    /* プライベートなファイルマッピング: COW */
    return wp_page_copy(vmf);
}

static vm_fault_t wp_page_copy(struct vm_fault *vmf)
{
    /* 新しいページを割り当てる */
    new_page = alloc_page_vma(GFP_HIGHUSER_MOVABLE, vmf->vma, vmf->address);

    /* 古いページの内容をコピー */
    cow_user_page(new_page, old_page, vmf);

    /* 書き込み可能な新しいPTEを設定 */
    new_entry = mk_pte(new_page, vma->vm_page_prot);
    new_entry = pte_mkwrite(pte_mkdirty(new_entry));
    set_pte_at_notify(vmf->vma->vm_mm, vmf->address, vmf->pte, new_entry);

    /* 古いページへの参照を減らす */
    put_page(old_page);
    return VM_FAULT_WRITE;
}
```

## カーネルのページフォールト {#kernel-page-faults}

```c
/* カーネル空間のフォールト: 通常はバグ */
static noinline void
do_kern_addr_fault(struct pt_regs *regs, unsigned long error_code,
                    unsigned long address)
{
    /* vmallocフォールト: init_mmからマッピングが同期されていない */
    if (!(error_code & X86_PF_USER) &&
        !(error_code & X86_PF_PROT) &&
        vmalloc_fault(address) >= 0)
        return;

    /* fixup_exception: copy_to_userなどで想定されたフォールト */
    if (!user_mode(regs) && fixup_exception(regs, X86_TRAP_PF,
                                              error_code, address))
        return;

    /* 回復不能: カーネルのバグ */
    oops_begin();
    show_fault_oops(regs, error_code, address);
    oops_end(flags, regs, SIGKILL);
}
```

よくあるカーネルフォールトは、無効なユーザーポインタを渡された`copy_to_user` / `copy_from_user`である。`fixup_exception`の仕組みは`{fault_address → recovery_address}`のペアの表を使う。フォールトアドレスが表にあれば、実行は回復コードへジャンプする（`-EFAULT`を返す）。

## ページフォールトの観測 {#observing-page-faults}

```bash
# プロセスごとのフォールト統計
cat /proc/<pid>/stat | awk '{print "minor:", $10, "major:", $12}'
# minor: 12345  (ページはキャッシュにあり、マッピングだけが必要だった)
# major: 42     (ページがメモリになく、I/Oが必要だった)

# システム全体のフォールト率 (perf)
perf stat -e page-faults,major-faults -- my_program

# ftraceで特定のフォールトをトレース
echo 1 > /sys/kernel/debug/tracing/events/exceptions/page_fault_user/enable
cat /sys/kernel/debug/tracing/trace

# perf recordでフォールトの発生箇所を探す
perf record -e page-faults -- my_program
perf report

# USDT/bpftrace:
bpftrace -e 'tracepoint:exceptions:page_fault_user {
    @addrs[uaddr] = count();
}'
```

## 参考文献 {#further-reading}

- [ファイルバックmmapとページフォールト](file-mmap.md) — do_fault/filemap_faultのパス
- [コピーオンライト](cow.md) — do_wp_pageの仕組み
- [プロセスのアドレス空間](mmap.md) — VMAの構造
- [スワップ](swap.md) — do_swap_pageのパス
- [KASAN](kasan.md) — 不正なカーネルメモリアクセスの検出
- `arch/x86/mm/fault.c` — x86のページフォールトのエントリ
- `mm/memory.c` — handle_mm_fault、do_anonymous_page、do_wp_page
