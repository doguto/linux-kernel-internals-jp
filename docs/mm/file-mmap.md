# ファイルバックドmmapとページフォールト {#file-backed-mmap-and-page-faults}

> ファイルのmmap()はどう機能するか: マッピングから最初の読み込みまで

## 通常ファイルのmmap {#mmap-of-a-regular-file}

```c
int fd = open("data.bin", O_RDONLY);
void *addr = mmap(NULL, file_size, PROT_READ, MAP_PRIVATE | MAP_POPULATE, fd, 0);
```

`mmap()` は **VMA**（virtual memory area）を作るが、ファイルを即座に読むことは **しない**。pageは遅延してフォールトで読み込まれる。各pageへの最初のアクセスが **ページフォールト** を引き起こす。

```
mmap() call:
  1. Allocate struct vm_area_struct
  2. Link to mm_struct's VMA tree (maple tree)
  3. Set vma->vm_file = file
     vma->vm_ops = &generic_file_vm_ops (or ext4_file_vm_ops)
  4. Return virtual address — no pages allocated yet

First read from addr:
  1. CPU: address not in page table → hardware page fault
  2. Kernel: do_page_fault() → handle_mm_fault()
  3. VMA found → filemap_fault() or ext4_filemap_fault()
  4. Page cache lookup: is the page already in cache?
     Hit: map the existing page into the process's page table
     Miss: allocate a page, read from disk, map into page table
```

## ページフォールトの経路 {#the-page-fault-path}

```c
/* mm/memory.c */
vm_fault_t handle_mm_fault(struct vm_area_struct *vma,
                             unsigned long address, unsigned int flags,
                             struct pt_regs *regs)
{
    pgd_t *pgd = pgd_offset(mm, address);
    p4d_t *p4d = p4d_alloc(mm, pgd, address);
    pud_t *pud = pud_alloc(mm, p4d, address);
    pmd_t *pmd = pmd_alloc(mm, pud, address);

    return handle_pte_fault(vmf);
}

static vm_fault_t handle_pte_fault(struct vm_fault *vmf)
{
    pte_t entry = *vmf->pte;

    if (!pte_present(entry)) {
        if (pte_none(entry)) {
            if (vma->vm_ops) {
                /* ファイルバックド: VMAのフォールトハンドラを呼ぶ */
                return do_fault(vmf);
            }
            /* 匿名: ゼロページを割り当てる */
            return do_anonymous_page(vmf);
        }
        /* スワップエントリ: ページはスワップ上にある */
        return do_swap_page(vmf);
    }

    /* 存在するが書き込み保護されている: copy-on-write */
    if (vmf->flags & FAULT_FLAG_WRITE && !pte_write(entry))
        return do_wp_page(vmf);

    return VM_FAULT_PFNMAP;
}
```

### do_fault: ファイルバックドのフォールト {#do_fault-file-backed-fault}

```c
/* mm/memory.c */
static vm_fault_t do_fault(struct vm_fault *vmf)
{
    struct vm_area_struct *vma = vmf->vma;

    if (!vma->vm_ops->fault)
        return VM_FAULT_SIGBUS;

    if (!(vmf->flags & FAULT_FLAG_WRITE)) {
        /* 読み込みフォールト: ページを共有する（MAP_PRIVATEでもMAP_SHAREDでも） */
        return do_read_fault(vmf);
    }

    if (!(vma->vm_flags & VM_SHARED)) {
        /* MAP_PRIVATEでの書き込みフォールト: copy-on-write */
        return do_cow_fault(vmf);
    }

    /* MAP_SHAREDでの書き込みフォールト: ページをdirtyにする */
    return do_shared_fault(vmf);
}

static vm_fault_t do_read_fault(struct vm_fault *vmf)
{
    /* VMAのfault()操作を呼ぶ */
    ret = __do_fault(vmf);
    /* ext4の場合: → ext4_filemap_fault → filemap_fault */

    map_pages(vmf, vmf->pgoff, vmf->pgoff);
    /* PTEを設定する: 仮想アドレス → 物理ページ */
}
```

### filemap_fault: ページキャッシュの参照 {#filemap_fault-page-cache-lookup}

```c
/* mm/filemap.c */
vm_fault_t filemap_fault(struct vm_fault *vmf)
{
    struct file *file = vmf->vma->vm_file;
    struct address_space *mapping = file->f_mapping;
    pgoff_t offset = vmf->pgoff;

    /* 1. ページキャッシュからページを探す */
    folio = filemap_get_folio(mapping, offset);

    if (!folio) {
        /* ページキャッシュミス: ディスクから読む必要がある */

        /* 2. 周辺ページの先読みを起動する */
        page_cache_sync_readahead(mapping, ra, file, offset,
                                   ra->ra_pages);

        /* 3. 新しいfolioを割り当てる */
        folio = filemap_alloc_folio(vmf->gfp_mask, 0);

        /* 4. ページキャッシュに追加する */
        filemap_add_folio(mapping, folio, offset, vmf->gfp_mask);

        /* 5. address_space_operations経由でディスクから読む */
        mapping->a_ops->readahead(rac);
        /* → ext4_readahead → ext4_mpage_readpages → bioの発行 */

        wait_on_page_locked(folio);
    } else {
        /* キャッシュヒット: 将来のページのために非同期先読みを起動する */
        page_cache_async_readahead(mapping, ra, file, folio,
                                    offset, ra->ra_pages);
    }

    vmf->page = folio_file_page(folio, offset);
    return VM_FAULT_LOCKED;
    /* 呼び出し側がPTEを設定する */
}
```

## 先読み {#readahead}

カーネルは、シーケンシャルアクセスに先立ってpageをプリフェッチする。

```c
/* mm/readahead.c */
void page_cache_sync_readahead(struct address_space *mapping,
                                struct file_ra_state *ra,
                                struct file *filp,
                                pgoff_t index, unsigned long req_count)
{
    /* 適応的: 小さく始め、ra_maxの上限に達するまで倍増させる */
    /* 典型的には: 初期値128KB、最大512KB以上 */
    ondemand_readahead(mapping, ra, filp, false, index, req_count);
}
```

ファイルごとの先読み状態（`struct file_ra_state`）:

```c
struct file_ra_state {
    pgoff_t start;          /* 先読みの開始位置 */
    unsigned int size;      /* 先読みするページ数 */
    unsigned int async_size;/* 非同期先読みを起動するページ数 */
    unsigned int ra_pages;  /* 先読みウィンドウの最大値 */
    unsigned int mmap_miss; /* mmapの先読み用 */
    loff_t prev_pos;        /* 前回のファイル位置 */
};
```

## MAP_PRIVATEとMAP_SHARED {#mapprivate-vs-mapshared}

### MAP_PRIVATE（copy-on-write） {#map_private-copy-on-write}

```
Initial state:
  Process PTE → page cache page (read-only)

On first write:
  COW fault:
  1. Allocate a new page
  2. Copy page cache page → new page
  3. Install new PTE → new page (writable)
  4. Page cache page unchanged
```

```c
/* MAP_PRIVATEの書き込みフォールト */
static vm_fault_t do_cow_fault(struct vm_fault *vmf)
{
    /* 新しいページを割り当てる */
    vmf->cow_page = alloc_page_vma(GFP_HIGHUSER_MOVABLE, vma, vmf->address);

    /* 元のファイルページを読む */
    ret = __do_fault(vmf);

    /* 元のページを新しいCOWページにコピーする */
    copy_user_highpage(vmf->cow_page, vmf->page, vmf->address, vma);

    /* 書き込み可能な新しいコピーを指すPTEを設定する */
    finish_fault(vmf);
}
```

### MAP_SHARED（共有されたdirty） {#map_shared-shared-dirty}

```
Write to MAP_SHARED:
  1. PTE was read-only (even though VMA is writable)
  2. do_shared_fault: dirty the page in-place
  3. Mark page dirty in page cache
  4. Install writable PTE

Later:
  Writeback daemon flushes dirty pages to disk
```

```c
static vm_fault_t do_shared_fault(struct vm_fault *vmf)
{
    /* ファイルからページを得る（ディスクから読むこともある） */
    ret = __do_fault(vmf);

    /* ファイルシステムが必要とするなら: dirtyページについて通知する */
    if (vma->vm_ops->page_mkwrite) {
        vma->vm_ops->page_mkwrite(vmf);
        /* ext4: ジャーナルトランザクションを開始し、ページをdirtyにマークする */
    }

    /* 書き込み可能なPTEを設定する */
    finish_fault(vmf);
    set_page_dirty(vmf->page);
}
```

## madviseによるヒント {#madvise-hints}

`madvise()` は、アプリケーションがpageにどのようにアクセスするかをカーネルに伝える。

```c
madvise(addr, length, advice);
```

| アドバイス | 効果 |
|--------|--------|
| `MADV_SEQUENTIAL` | 先読みを積極的に増やす。使用後のpageを解放する |
| `MADV_RANDOM` | 先読みを無効にする。ランダムアクセスパターン |
| `MADV_WILLNEED` | 今すぐpageをプリフェッチする（非同期先読み） |
| `MADV_DONTNEED` | pageを破棄し、物理メモリを解放する |
| `MADV_FREE` | pageを回収候補としてマークする（遅延解放） |
| `MADV_HUGEPAGE` | この範囲にTHPを要求する |
| `MADV_POPULATE_READ` | 読み込みを模擬してpageを事前に読み込む（COWなし） |
| `MADV_POPULATE_WRITE` | 書き込みを模擬してpageを事前に読み込む（COWを引き起こす） |

```c
/* 処理する前にファイルの一部をプリフェッチする */
madvise(file_addr, CHUNK_SIZE, MADV_WILLNEED);
/* ... 他の作業をする ... */
/* ここでfile_addrにアクセスする — ページはおそらくすでにキャッシュにある */

/* 使い終わった大きなマッピングを解放する */
madvise(mmap_addr, file_size, MADV_DONTNEED);
/* 物理ページは即座に解放される。次のアクセスではファイルから再度読む */
```

### カーネル内のMADV_SEQUENTIAL {#madv_sequential-in-the-kernel}

```c
/* mm/madvise.c */
static long madvise_behavior(struct vm_area_struct *vma, ...)
{
    switch (behavior) {
    case MADV_SEQUENTIAL:
        new_flags |= VM_SEQ_READ;
        new_flags &= ~VM_RAND_READ;
        break;
    case MADV_RANDOM:
        new_flags |= VM_RAND_READ;
        new_flags &= ~VM_SEQ_READ;
        break;
    }
}

/* filemap_fault内: vm_flagsを確認する */
if (vma->vm_flags & VM_SEQ_READ)
    ra->ra_pages = max(ra->ra_pages, bdi->ra_pages);
if (vma->vm_flags & VM_RAND_READ)
    ra->ra_pages = 0;  /* 先読みを無効にする */
```

## ファイルマッピングのためのヒュージページ（THP） {#huge-pages-for-file-mappings-thp}

ファイルシステムが対応していれば（ext4、tmpfs、5.18以降のxfs）、ファイルバックドのマッピングでもTransparent Huge Pageを使える。

```c
/* ヒュージページのフォールト: 4KBではなく2MBをマップする */
vm_fault_t do_huge_pmd_anonymous_page(struct vm_fault *vmf)
{
    /* ファイルバックドの場合: filemap_fault_huge → 連続する512ページを読む */
}
```

```bash
# ファイルマッピングに対して THP を有効にする
echo always > /sys/kernel/mm/transparent_hugepage/enabled

# マッピング単位で
madvise(addr, length, MADV_HUGEPAGE);

# THP の使用状況を確認する
cat /proc/<pid>/smaps | grep -A5 "file_name"
# AnonHugePages:    2048 kB
```

## 参考文献 {#further-reading}

- [ページキャッシュ](page-cache.md) — filemap_faultが読み出すキャッシュ
- [Copy-on-Write](cow.md) — MAP_PRIVATEのCOWの仕組み
- [プロセスのアドレス空間](mmap.md) — VMAのレイアウト
- [exec()の間に何が起きるか](exec.md) — ELFローダが作るファイルmmap
- [ページ回収](reclaim.md) — ファイルバックドpageの回収
- `mm/filemap.c` — filemap_fault、先読み
- `mm/madvise.c` — madviseの実装
