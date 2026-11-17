# メモリ管理におけるRCU {#rcu-in-memory-management}

> Read-Copy-Updateによって、ページフォールトがロックを取らずにVMAを見つけられる仕組み

## 主要なソースファイル {#key-source-files}

| ファイル | 目的 |
|------|---------|
| `mm/mmap_lock.c` | `lock_vma_under_rcu()`、`vma_start_read()` |
| `include/linux/mmap_lock.h` | `vma_end_write_all()` |
| `mm/vma_init.c` | `vm_area_alloc()`、`vm_area_free()`、`SLAB_TYPESAFE_BY_RCU`のslab設定 |
| `lib/maple_tree.c` | `ma_free_rcu()`、`mt_free_walk()` — RCUで遅延させたノードの解放 |
| `include/linux/mm_types.h` | `struct vm_area_struct` — `vm_freeptr`、`vm_lock_seq`、`vm_refcnt` |
| `include/linux/rcupdate.h` | `rcu_read_lock()`、`rcu_dereference()`、`call_rcu()`、`kfree_rcu()`、`synchronize_rcu()` |
| `mm/mmu_notifier.c` | SRCUで保護されたnotifierリスト; `srcu_read_lock()` / `srcu_read_unlock()` |
| `mm/filemap.c` | `filemap_get_folios_contig()`、`filemap_get_folios_tag()` — RCUによるページキャッシュの走査 |
| `arch/x86/mm/fault.c` | ページフォールトの入口: `lock_mm_and_find_vma()`の前の`lock_vma_under_rcu()`による高速パス |

---

## なぜメモリ管理にRCUを使うか {#why-rcu-in-memory-management}

すべてのプロセスの仮想アドレス空間は、`mm->mm_mt`のmaple treeに格納されたVMA（Virtual Memory Area）の集合である。すべてのページフォールトは、ページテーブルエントリを設置する前に、フォールトしたアドレスに対応する正しいVMAを見つける必要がある。Linux 6.4までは、この検索には常に`mmap_lock`を読み込み用に取得する必要があった。

`mmap_lock`は`mm_struct`ごとの読み書きセマフォである。読み込みロックは共有されるので、同じプロセス内の並列なフォールトは一緒に進行できる。しかしそれは、いずれかの書き込み側（任意のスレッドでの`mmap()`、`munmap()`、`mprotect()`、`mremap()`呼び出し）が書き込みロックを取って**並行するすべてのフォールトをブロックする**までの話である。多数のスレッドが同時にフォールトするシステムでは、書き込み側のわずかな時間枠でさえ、フォールト処理の大きなバーストを直列化してしまう。

問題は構造的でもある。`mmap_lock`は粒度が粗く、VMAツリー全体を保護している。まったく別々のメモリ領域にフォールトする2つのスレッドでさえ、このロックで競合する。

RCU（Read-Copy-Update）はこのボトルネックを打破する。maple treeはRCUセーフであり、そのノードは即座にではなく`call_rcu()`を介して解放される。つまり、`rcu_read_lock()`を保持するリーダーは、すでに観測したノードが足元で解放されることはないと分かっているので、ツリーをたどってノードに安全にアクセスできる。VMAごとの参照カウントと組み合わせることで、よくあるケース、すなわち今まさに変更されていないVMAを見つけるケースは、リーダーの視点から見て完全にロックフリーになる。

---

## RCUの基本 {#rcu-basics}

RCUは読み込みが多いワークロード向けに最適化された同期メカニズムである。中核となる約束事は次のとおり:

- **リーダー**は、RCUで保護されたデータにアクセスする前に`rcu_read_lock()`を呼び、終わったら`rcu_read_unlock()`を呼ぶ。リーダーのセクションではスリープできない（非PREEMPTカーネルではプリエンプションを無効化し、PREEMPT_RCUでは特別なトークンを保持する）。
- **リーダーはポインタを`rcu_dereference()`で参照外しする**。これは適切なメモリバリアを挿入し、ポインタ自体がロードされる前に、コンパイラやCPUが依存データのロードを投機的に行うことを防ぐ。
- **ライター**は`rcu_assign_pointer()`を使って新しいポインタを公開することでデータを更新し、その後、現在のすべてのリーダーがRCU読み込みセクションを終えるのを待ってから古いバージョンを解放する。この待ち期間を**グレースピリオド**と呼ぶ。

グレースピリオドを待つ方法は2つある:

```c
/* 同期的: すべてのリーダーが完了するまで呼び出し元をブロックする */
synchronize_rcu();

/* 非同期: グレースピリオド後に実行されるコールバックをスケジュールする */
call_rcu(&obj->rcu_head, my_free_function);

/* 便利なラッパー: グレースピリオド後にオブジェクトを解放する */
kfree_rcu(ptr, rcu_head_field);
```

重要なポイントは、`synchronize_rcu()`には数ミリ秒かかりうるということである。すべてのCPU上のリーダーがそれぞれ現在のRCU読み込みセクションを終えなければならないからである。`call_rcu()`と`kfree_rcu()`は直ちに戻り、解放をRCUコールバックスレッドに委ねる。メモリ管理のコードは、待つ余裕があるかどうかに基づいてこれらを使い分ける。

---

## RCU下でのVMA検索 {#vma-lookup-under-rcu}

### maple treeはRCUセーフ {#the-maple-tree-is-rcu-safe}

maple tree（`mm->mm_mt`）はアドレス範囲をキーとしてVMAを格納する。VMAが削除されたり、ツリーのノードが置き換えられたりしても、古いノードは即座には解放されない。代わりに、`lib/maple_tree.c`の`ma_free_rcu()`が`kfree_rcu(node, rcu)`を呼び、解放をグレースピリオド後まで遅延させる:

```c
/* lib/maple_tree.c */
static void ma_free_rcu(struct maple_node *node)
{
    WARN_ON(node->parent != ma_parent_ptr(node));
    kfree_rcu(node, rcu);
}
```

したがって、`rcu_read_lock()`を保持するリーダーはツリーのノードを安全にたどることができる。リーダーが見たノードはどれも、読み込みセクションが終わるまでは割り当てられたまま（ツリー内にあるとは限らないが）であることが保証される。

### `lock_vma_under_rcu()` {#lock_vma_under_rcu}

`mm/mmap_lock.c`の`lock_vma_under_rcu()`は、v6.4以降ページフォールトで使われている高速パスのVMA検索である。maple treeの走査のためのRCU読み込みセクションと、見つかったVMAを安定させるためのVMAごとの参照カウントを組み合わせる:

```c
/* mm/mmap_lock.c */
struct vm_area_struct *lock_vma_under_rcu(struct mm_struct *mm,
                                          unsigned long address)
{
    MA_STATE(mas, &mm->mm_mt, address, address);
    struct vm_area_struct *vma;

retry:
    rcu_read_lock();
    vma = mas_walk(&mas);           /* RCU で保護された maple tree の走査 */
    if (!vma) {
        rcu_read_unlock();
        goto inval;
    }

    vma = vma_start_read(mm, vma);  /* VMA ごとの読み込みロックを試みる */
    if (IS_ERR_OR_NULL(vma)) {
        if (PTR_ERR(vma) == -EAGAIN)
            goto retry;             /* VMA が切り離された、リトライ */
        goto inval;                 /* rcu_read_unlock() は呼び出し済み */
    }

    rcu_read_unlock();

    /* ロックした VMA が実際にそのアドレスをカバーしているか検証 */
    if (unlikely(address < vma->vm_start || address >= vma->vm_end)) {
        vma_end_read(vma);
        goto inval;
    }
    return vma;   /* 安定した、VMA ごとにロックされた VMA。mmap_lock は保持していない */

inval:
    count_vm_vma_lock_event(VMA_LOCK_ABORT);
    return NULL;
}
```

!!! note "重要: `vma_start_read()`は失敗時にRCUロックを解放することがある"
    ソース中のコメントは明確である。`vma_start_read()`がエラーを返したとき、すでに`rcu_read_unlock()`を呼んでいる。呼び出し元は、`vma_start_read()`が失敗した後のエラーパスで再び`rcu_read_unlock()`を呼んではならない。この非対称性は、コードを読む際によくある混乱の原因である。

### `vma_start_read()` {#vma_start_read}

`vma_start_read()`（同じく`mm/mmap_lock.c`内）は、VMAごとの読み込みロックの取得である。`vm_lock_seq`と`mm_lock_seq`を使って、VMAが現在書き込み中かどうかを検出する:

```c
/* mm/mmap_lock.c */
static inline struct vm_area_struct *vma_start_read(struct mm_struct *mm,
                                                    struct vm_area_struct *vma)
{
    /* 早期脱出: ロックシーケンスが一致すれば、VMA は書き込みロックされている */
    if (READ_ONCE(vma->vm_lock_seq) == READ_ONCE(mm->mm_lock_seq.sequence)) {
        vma = NULL;
        goto err;   /* rcu_read_unlock() も呼ぶ */
    }

    /* acquire フェンス付きで vm_refcnt をインクリメント */
    if (unlikely(!__refcount_inc_not_zero_limited_acquire(&vma->vm_refcnt,
                                                          &oldcnt,
                                                          VM_REFCNT_LIMIT))) {
        vma = oldcnt ? NULL : ERR_PTR(-EAGAIN);
        goto err;
    }

    /* 参照カウントを保持した状態でロックシーケンスを再確認 */
    if (unlikely(vma->vm_lock_seq == raw_read_seqcount(&mm->mm_lock_seq))) {
        vma_refcount_put(vma);
        vma = NULL;
        goto err;
    }
    return vma;
    ...
}
```

ロジックは次のとおり:

1. `vma->vm_lock_seq == mm->mm_lock_seq.sequence`であれば、VMAは現在書き込みロックされているので、脱出する。
2. acquireセマンティクスで`vm_refcnt`をインクリメントし、VMAのフィールドのロードがカウントのインクリメントより前に並べ替えられることを防ぐ。
3. ステップ1とステップ2の間でシーケンス番号が変わった可能性があるので、参照カウントを保持した状態で`vm_lock_seq`を再確認する。

VMAごとのロックの完全な状態機械と書き込み側（`vma_end_write_all()`）については、[VMA単位のロック](per-vma-locks.md)を参照。

### ページフォールトの高速パス（x86） {#the-page-fault-fast-path-x86}

`arch/x86/mm/fault.c`のアーキテクチャのフォールトハンドラは、まずRCUパスを試みる:

```c
/* arch/x86/mm/fault.c */
vma = lock_vma_under_rcu(mm, address);
if (!vma)
    goto lock_mmap;   /* mmap_lock にフォールバック */

if (unlikely(access_error(error_code, vma))) {
    bad_area_access_error(...);
    count_vm_vma_lock_event(VMA_LOCK_SUCCESS);
    return;
}
fault = handle_mm_fault(vma, address, flags | FAULT_FLAG_VMA_LOCK, regs);
if (!(fault & (VM_FAULT_RETRY | VM_FAULT_COMPLETED)))
    vma_end_read(vma);
...
lock_mmap:
    vma = lock_mm_and_find_vma(mm, address, regs);  /* 低速パス */
```

`FAULT_FLAG_VMA_LOCK`フラグは、どのロックが保持されているかを`handle_mm_fault()`に伝える。ARM64、ARM、PowerPC、s390、LoongArch、RISC-Vはすべて同じパターンを実装している。

---

## `struct vm_area_struct`とRCU {#struct-vm_area_struct-and-rcu}

### 割り当て: `SLAB_TYPESAFE_BY_RCU` {#allocation-slab_typesafe_by_rcu}

`vm_area_struct`オブジェクトは、`mm/vma_init.c`で作成される専用のslabキャッシュ（`vm_area_cachep`）から割り当てられる:

```c
/* mm/vma_init.c */
vm_area_cachep = kmem_cache_create("vm_area_struct",
        sizeof(struct vm_area_struct), &args,
        SLAB_HWCACHE_ALIGN | SLAB_PANIC | SLAB_TYPESAFE_BY_RCU |
        SLAB_ACCOUNT);
```

`SLAB_TYPESAFE_BY_RCU`が鍵となるフラグである。これは次のことを意味する:

- オブジェクトは解放されても**即座には**システムに**返されず**、グレースピリオド後まで保持される。
- 決定的に重要なのは、グレースピリオドが経過すると、slabアロケータは解放されたオブジェクトを新しい割り当てのために**再利用**しうるが、そのメモリをOSや別のslabに返すことはないという点である。
- `rcu_read_lock()`を保持するリーダーは、maple treeから得た`vm_area_struct`ポインタから安全に読み込むことができる。構造体のメモリは、読み込みセクションが終わるまで割り当てられたまま（再利用されている可能性はあるが）である。

これは解放コールバック付きの`call_rcu()`とは異なる。`SLAB_TYPESAFE_BY_RCU`では、オブジェクトの型（`struct vm_area_struct`）がグレースピリオドをまたいで保たれる。リーダーは依然として、見つけたVMAが期待したものであることを（`vma_start_read()`によって）検証しなければならない。

`mm/vma_init.c`の`vm_area_free()`は、公開されたVMA破棄関数である:

```c
/* mm/vma_init.c */
void vm_area_free(struct vm_area_struct *vma)
{
    vma_assert_detached(vma);
    vma_numab_state_free(vma);
    free_anon_vma_name(vma);
    vma_pfnmap_track_ctx_release(vma);
    kmem_cache_free(vm_area_cachep, vma);  /* SLAB_TYPESAFE_BY_RCU の slab に戻す */
}
```

グレースピリオドを扱うのは呼び出し元ではなくslabである。

### RCU下で（`vma_start_read()`の前に）アクセスできるフィールド {#fields-accessible-under-rcu-before-vma_start_read}

`include/linux/mm_types.h`の`struct vm_area_struct`定義にあるコメントは次のように述べている:

> Only explicitly marked struct members may be accessed by RCU readers before getting a stable reference.

（訳: 安定した参照を得る前にRCUリーダーがアクセスしてよいのは、明示的にマークされた構造体メンバーだけである。）

不安定なRCUリーダー（`vma_start_read()`が成功する前）がアクセスしてよいフィールド:

| フィールド | 安定している理由 |
|-------|-----------|
| `vm_start`、`vm_end` | `vm_freeptr`とunionになっている。VMAがmaple treeに入った後は安定 |
| `vm_mm` | 不安定なRCUリーダーがアクセス可能であると明示的に文書化されている |
| `vm_lock_seq` | 楽観的チェックのために`vma_start_read()`内で`READ_ONCE()`で読まれる |
| `vm_refcnt` | ロックプロトコル全体を通じてアトミックにアクセスされる |

`vma_start_read()`（VMAごとの読み込みロック）の成功なしでは安全**でない**フィールド:

- `vm_flags` — `mprotect()`によって変更中かもしれない
- `vm_file`、`vm_pgoff`、`anon_vma` — アンマップやリマップで変わる可能性がある
- `vm_ops`、`vm_private_data` — 背後のファイル/デバイスに結びついている

!!! warning "`vm_freeptr`によるフィールドの再利用"
    `vm_start`と`vm_end`は、SLABのフリーリストポインタである`vm_freeptr`とunionを共有している。VMAがslabに解放されると、アロケータが`vm_freeptr`に書き込み、`vm_start`/`vm_end`を上書きすることがある。これこそが、`vma_start_read()`がVMAごとのロックを取得した*後*にアドレス範囲を検証する理由である。ロックが失敗した場合、構造体はすでに未定義の状態にあるかもしれない。

---

## RCU下でのページテーブルの走査 {#page-table-walks-under-rcu}

### `follow_page_mask()`と`vma_pgtable_walk_begin()` {#follow_page_mask-and-vma_pgtable_walk_begin}

`mm/gup.c`の`follow_page_mask()`は、ユーザー仮想アドレスに対応する物理pageを検索する（`get_user_pages()`などで使われる）。この関数はページテーブルの走査を次のように囲む:

```c
/* mm/gup.c */
static struct page *follow_page_mask(struct vm_area_struct *vma,
                                     unsigned long address, unsigned int flags,
                                     unsigned long *page_mask)
{
    vma_pgtable_walk_begin(vma);
    ...
    page = follow_p4d_mask(vma, address, pgd, flags, page_mask);
    vma_pgtable_walk_end(vma);
    return page;
}
```

`mm/memory.c`の`vma_pgtable_walk_begin()`は、VMAがHugeTLBマッピングの場合、hugetlbのVMA読み込みロックを取得する:

```c
/* mm/memory.c */
void vma_pgtable_walk_begin(struct vm_area_struct *vma)
{
    if (is_vm_hugetlb_page(vma))
        hugetlb_vma_lock_read(vma);
}

void vma_pgtable_walk_end(struct vm_area_struct *vma)
{
    if (is_vm_hugetlb_page(vma))
        hugetlb_vma_unlock_read(vma);
}
```

通常の（hugetlbでない）VMAではこれらは何もしない。保護はページテーブルロック（`pte_offset_map_lock()`）と、ページテーブル用pageが`mm_inc_nr_ptes()` / `mm_dec_nr_ptes()`での`mm->pgtables_bytes`の追跡によって参照カウントされていることから得られる。

### ページテーブル用pageの寿命 {#page-table-page-lifetime}

ページテーブル用page（PTEを*保持する*物理page）は、次によって追跡される:

- 割り当てと解放時の`mm_inc_nr_ptes()` / `mm_dec_nr_ptes()`。`mm->pgtables_bytes`を更新する。
- 一部のグローバルなページテーブル操作のための、`mm_struct`内の`page_table_lock` spinlock。
- PTEレベルの操作のために`pte_offset_map_lock()`で取得される、ページテーブルごとのspinlock。

`follow_page_pte()`がPTEレベルまでたどると、`pte_offset_map_lock()`を使ってページテーブルごとのspinlockを取得し、PTEの読み込み中にページテーブル用pageが（`mm_dec_nr_ptes()`を介して）解放されることを防ぐ。

RCUだけではページテーブル用pageの解放を防げない。階層化された設計は次のとおりである。RCUは*VMA*を保護し（フォールトハンドラが`mmap_lock`なしで正しいVMAを見つけられるように）、次にVMAごとのロックがVMAを安定させ、そして通常のページテーブルのロック（`pte_offset_map_lock()`）がPTEの内容を保護する。

---

## MMU notifierとSRCU {#mmu-notifiers-and-srcu}

MMU notifierにより、KVMやRDMAのようなサブシステムは、カーネルがページテーブルエントリを無効化しようとしていることを知ることができる。KVMはこれを使い、ホストが領域を破棄またはリマップするたびに、ゲスト物理 → ホスト仮想のマッピングをフラッシュする。

### notifierリストは素のRCUではなくSRCUを使う {#the-notifier-list-uses-srcu-not-plain-rcu}

`mm/mmu_notifier.c`のmmu notifierリストは、従来のRCUではなく**SRCU**（Sleepable RCU）を使う。重要な違いは、SRCUでは読み込み側のクリティカルセクション内でスリープが許されることであり、notifierのコールバック（KVMのTLBフラッシュパスなど）はそれを必要とすることがある。

```c
/* mm/mmu_notifier.c */
DEFINE_STATIC_SRCU(srcu);   /* すべての MM に対して1つのグローバルな SRCU ドメイン */
```

カーネルが購読者に通知する必要があるとき（たとえば`unmap_vmas()`でのページテーブル破棄中）:

```c
/* mm/mmu_notifier.c — 簡略化 */
int __mmu_notifier_invalidate_range_start(struct mmu_notifier_range *range)
{
    int id = srcu_read_lock(&srcu);
    hlist_for_each_entry_rcu(subscription, &subscriptions->list, hlist,
                             srcu_read_lock_held(&srcu)) {
        subscription->ops->invalidate_range_start(subscription, range);
    }
    srcu_read_unlock(&srcu, id);
    ...
}
```

走査には`hlist_for_each_entry_rcu()`が使われる。notifierの登録と登録解除は`mmap_write_lock()` + SRCUの同期の下でリストを変更するので、これは安全である。リーダーは`srcu_read_lock()`だけを必要とする。

### `mmu_notifier_get()`と`mmu_notifier_put()` {#mmu_notifier_get-and-mmu_notifier_put}

```c
/* include/linux/mmu_notifier.h */
static inline struct mmu_notifier *
mmu_notifier_get(const struct mmu_notifier_ops *ops, struct mm_struct *mm)
{
    struct mmu_notifier *ret;
    mmap_write_lock(mm);
    ret = mmu_notifier_get_locked(ops, mm);   /* __mmu_notifier_register() を呼ぶ */
    mmap_write_unlock(mm);
    return ret;
}

void mmu_notifier_put(struct mmu_notifier *subscription);
```

`mmu_notifier_get()`には`mmap_write_lock`が必要である。登録はnotifierリストを変更するので、直列化されなければならない。`mmu_notifier_put()`はSRCUを使い、進行中のすべての`invalidate_range_start`コールバックが完了するまで最終的な後始末を遅延させる。

### KVMにとってなぜ重要か {#why-this-matters-for-kvm}

SRCUがなければ、KVMの無効化コールバックはスリープせずに完了しなければならない。これは、vCPUスレッドと同期する必要があるハイパーバイザ横断のTLBフラッシュやIPIベースのシュートダウンにとって現実的ではない。SRCUにより、KVMは並行するリストの変更から保護されたまま、`invalidate_range_start()`の中でスリープできる。

---

## folioとページキャッシュのRCU {#folio-and-page-cache-rcu}

ページキャッシュは、XArray（`mapping->i_pages`）を使ってファイルオフセットをfolio（連続したpageのまとまり）に対応付ける。XArrayは内部的にRCUセーフであり、その`xa_head`ポインタは`__rcu`アノテーション付きで、ノードのスロットはRCUで保護されている。

### `filemap.c`におけるRCU検索 {#rcu-lookups-in-filemapc}

いくつかの関数は`rcu_read_lock()`の下でページキャッシュを走査し、RCUセクションが終わる前に`folio_try_get()`を使ってアトミックに参照を取得する:

```c
/* mm/filemap.c — filemap_get_folios_contig() */
unsigned filemap_get_folios_contig(struct address_space *mapping,
        pgoff_t *start, pgoff_t end, struct folio_batch *fbatch)
{
    XA_STATE(xas, &mapping->i_pages, *start);
    rcu_read_lock();

    for (folio = xas_load(&xas); folio && xas.xa_index <= end;
            folio = xas_next(&xas)) {
        if (xas_retry(&xas, folio))
            continue;
        if (xa_is_value(folio))   /* スワップ/シャドウエントリ */
            goto update_start;
        if (!folio_try_get(folio))
            goto retry;
        if (unlikely(folio != xas_reload(&xas)))
            goto put_folio;       /* 取得している間に folio が置き換えられた */
        folio_batch_add(fbatch, folio);
        ...
    }
    rcu_read_unlock();
    ...
}
```

パターンは次のとおり:

1. `rcu_read_lock()` — XArrayのノードの解放を防ぐ。
2. `xas_load()` / `xas_next()` — RCUの保護下で走査する。
3. `folio_try_get()` — folioの参照カウントをアトミックにインクリメントする。カウントがすでにゼロだった場合（folioが解放中）はfalseを返す。
4. `xas_reload()` — XArrayのスロットがまだ同じfolioを指していることを確認する（`xas_next()`と`folio_try_get()`の間に新しいfolioに置き換えられた可能性がある）。
5. `rcu_read_unlock()` — folioは参照カウントによって保持されているので、RCUセクションの外で安全に使える。

`filemap_get_folio()`（単一folioの検索）は、最終的に`filemap_get_entry()`を呼び、これも同じ`rcu_read_lock()` + `folio_try_get()` + `xas_reload()`の手順を使う。

### なぜ単なるspinlockではないのか? {#why-not-just-a-spinlock}

ページキャッシュは変更されるよりもはるかに頻繁に読まれる。負荷がかかった状態では、検索のたびにspinlockを取るとキャッシュラインがCPU間を行き来してしまう。`folio_try_get()`を伴うRCUにより、すべてのCPUがXArrayを並行して読めるようになる。その代償は、folioが同時に追い出されているまれな場合に、リトライのロジックがやや複雑になることである。

---

## グレースピリオドのコスト {#grace-period-costs}

### `synchronize_rcu()`は遅い {#synchronize_rcu-is-slow}

`synchronize_rcu()`は、すべてのCPUが静止状態（そのCPU上でRCU読み込みセクションがアクティブでない時点）を通過するまでブロックする。大規模なNUMAシステムでは、これに数ミリ秒かかることがある。

メモリ管理のコードは、性能が重要なパスでは同期的なグレースピリオドを避ける:

| 操作 | メカニズム | 理由 |
|-----------|-----------|--------|
| maple treeのノード削除 | `ma_free_rcu()`経由の`kfree_rcu(node, rcu)` | VMAのアンマップは高速でなければならない |
| VMAのslab解放 | `SLAB_TYPESAFE_BY_RCU`（slab内の暗黙のグレースピリオド） | VMAごとの`call_rcu`のオーバーヘッドを避ける |
| MMU notifierの登録解除 | `synchronize_srcu(&srcu)` | notifierを解放する前に、実行中のすべてのコールバックが完了していることを保証しなければならない |
| `mmap_write_unlock()` | `vma_end_write_all()` → `mm_lock_seqcount_end()`を呼ぶ | `mm_lock_seq`をインクリメントしてVMAごとのすべての読み込みロックを無効化する。グレースピリオドは不要 |

### `mmap_write_lock()`に隠れたコスト {#the-hidden-cost-in-mmap_write_lock}

`mmap_write_lock()`が`mmap_lock`の書き込み側を取得するとき、それ自体はRCUのグレースピリオドを*待たない*。しかし書き込みロックは、現在のすべての`mmap_read_lock()`保持者が解放するまでブロックし、それらの読み込み保持者自身がフォールトパスで`rcu_read_lock()`セクション内にいる可能性がある。実質的に、書き込みロックは進行中のフォールト処理の完了を待つことになる。

`mmap_write_unlock()`が呼ばれると、`vma_end_write_all()`を呼び、これが`mm_lock_seq`をインクリメントして、未解放のVMAごとの読み込みロックをすべてアトミックに無効化する。その後に`vma_start_read()`を呼ぶスレッドは、更新されたシーケンス番号を見て、次のフォールトでは`mmap_lock`の低速パスにフォールバックする。

---

## mmにおけるRCUの問題のデバッグ {#debugging-rcu-issues-in-mm}

### `CONFIG_PROVE_RCU` {#config_prove_rcu}

`CONFIG_PROVE_RCU`（`CONFIG_PROVE_LOCKING`によって暗黙に有効になる）を有効にすると、RCUの使用に対してlockdepによるチェックが得られる。このオプションでは、RCUアノテーション付きのポインタが有効なRCU読み込みセクションの外でアクセスされると、`RCU_LOCKDEP_WARN()`の呼び出しが警告を出す。

mmのコードは`RCU_LOCKDEP_WARN`を多用している:

```c
/* mm/mmap_lock.c — vma_start_read() */
RCU_LOCKDEP_WARN(!rcu_read_lock_held(), "no rcu lock held");

/* mm/mmap_lock.c — lock_next_vma() */
RCU_LOCKDEP_WARN(!rcu_read_lock_held(), "no rcu read lock held");
```

### `lockdep_assert_in_rcu_read_lock()` {#lockdep_assert_in_rcu_read_lock}

`include/linux/rcupdate.h`は`lockdep_assert_in_rcu_read_lock()`（`CONFIG_PROVE_RCU`が必要）を定義しており、これはRCU読み込みセクションの外で呼ばれると警告を出す。RCUによる保護を前提とする新しいコードではこれを使うとよい。

### sparseと`__rcu`アノテーション {#sparse-and-__rcu-annotations}

RCUで保護されるポインタフィールドは、宣言で`__rcu`とアノテーションされる（例: `struct xarray`の`void __rcu *xa_head`）。`sparse`でビルドする（`make C=1`）と、そのようなポインタが`rcu_dereference()`なしで参照外しされた場合に警告が出る。

### mmのRCUコードでよくあるバグ {#common-bugs-in-mm-rcu-code}

**`rcu_read_lock()`の中でスリープする**。
従来のRCUではスリープは許されない。ページフォールトパスでは、`lock_vma_under_rcu()`はmaple treeの走査の間だけ`rcu_read_lock()`を保持し、`handle_mm_fault()`を呼ぶ前に解放する。コードパスが`rcu_read_lock()`を取得した後にスリープしうるもの（メモリ割り当て、ファイルI/O、mutex）を呼ぶと、`CONFIG_PROVE_RCU`が発動する。

```
BUG: sleeping function called from invalid context
in_atomic(): 1, irqs_disabled(): 0, ...
```

**安定した参照なしにVMAのフィールドにアクセスする**。
`rcu_read_lock()`だけの下で（`vma_start_read()`の成功なしに）`vma->vm_flags`や`vma->vm_file`を読むのは誤りである。それらのフィールドは並行して変更されている可能性がある。`SLAB_TYPESAFE_BY_RCU`の保証は、*メモリ*が解放されないことを意味するだけで、*フィールド*が安定していることは意味しない。

**`SLAB_TYPESAFE_BY_RCU`による再利用の混乱**。
slabオブジェクトはグレースピリオド後に再利用されうるので、maple treeから`vm_area_struct *`を得た後に（RCUセクションの外で）スリープしたリーダーは、目覚めたときに同じメモリが別の`mm`のまったく別のVMAを表していることに気づくかもしれない。常に`vma_start_read()`で検証すること。この関数は`vma->vm_mm == mm`とロックシーケンス番号を確認する。

**すべてのパスで`vma_end_read()`を忘れる**。
`vma_start_read()`は`vm_refcnt`をインクリメントする。`lock_vma_under_rcu()`からの成功した戻りはすべて、ちょうど1回の`vma_end_read()`で釣り合わなければならない。これを忘れると参照がリークし、最終的にはすべてのリーダーを排出する必要のあるライター（例: `munmap()`）が停滞する。

---

## まとめ: 通常のVMAでの読み込みフォールト {#putting-it-together-a-read-fault-on-a-normal-vma}

```
User thread takes a page fault
│
├─ arch fault handler (e.g. arch/x86/mm/fault.c)
│   ├─ lock_vma_under_rcu(mm, address)
│   │   ├─ rcu_read_lock()
│   │   ├─ mas_walk(&mas)          → finds VMA in maple tree under RCU
│   │   ├─ vma_start_read(mm, vma) → increments vm_refcnt, validates lock seq
│   │   └─ rcu_read_unlock()       → RCU section ends; VMA stable via vm_refcnt
│   │
│   └─ handle_mm_fault(vma, address, FAULT_FLAG_VMA_LOCK, regs)
│       └─ __handle_mm_fault()
│           └─ do_fault() / do_anonymous_page() / ...
│               └─ pte_offset_map_lock()   → page table spinlock
│
└─ vma_end_read(vma)               → decrements vm_refcnt
```

よくあるケースでは`mmap_lock`はまったく取得されない。VMAツリーの走査、VMAの安定性、PTEの更新は、それぞれ適切な狭い範囲のロックまたはRCUセクションによって保護される。

---

## 関連項目 {#see-also}

- [VMA単位のロック](per-vma-locks.md) — `vm_lock_seq`、`vm_refcnt`、書き込み側プロトコルの詳しい解説
- [Maple Tree](maple-tree.md) — VMAを格納するRCUセーフなBツリー
- [ページテーブル](page-tables.md) — PTEのロック、`pte_offset_map_lock()`、ページテーブル用pageの寿命
- [ページキャッシュ](page-cache.md) — XArrayベースのfolio格納と、より広いファイルバックのフォールトパス

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [mm/mmap.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mmap.c) — `mmap_write_lock()` / `mmap_write_unlock()`と、`mm_lock_seq`を増やす`vma_end_write_all()`
- [mm/mmap_lock.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mmap_lock.c) — `lock_vma_under_rcu()`と`vma_start_read()`の実装
- [include/linux/rcupdate.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/rcupdate.h) — `rcu_read_lock()`、`rcu_dereference()`、`call_rcu()`、`kfree_rcu()`、`synchronize_rcu()`

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/RCU/whatisRCU.rst`](https://docs.kernel.org/RCU/whatisRCU.html) — RCUのセマンティクスとread-copy-updateの約束事についての公式な入門
- [`Documentation/RCU/rcu_dereference.rst`](https://docs.kernel.org/RCU/rcu_dereference.html) — RCUで保護されたポインタを安全に参照外しするためのルール
- [`Documentation/RCU/Design/Requirements/Requirements.rst` — Sleepable RCU](https://docs.kernel.org/RCU/Design/Requirements/Requirements.html#sleepable-rcu) — sleepable RCU（MMU notifierで使われるもの）をいつ、なぜ使うか

### 関連ページ {#related-pages}

- [../locking/rcu.md](../locking/rcu.md) — RCUの一般的な概念、静止状態、グレースピリオドの仕組み
- [per-vma-locks.md](per-vma-locks.md) — RCUによるVMA検索の上に位置する、VMA単位のロック層

### LWNの記事 {#lwn-articles}

- [LWN: What is RCU, fundamentally?](https://lwn.net/Articles/262464/) — Paul McKenneyによるRCU設計についての決定版シリーズ
- [LWN: RCU usage in the Linux kernel](https://lwn.net/Articles/573424/) — メモリ管理のパターンを含む、各サブシステムがRCUをどう使っているかの概観
- [LWN: `SLAB_TYPESAFE_BY_RCU`](https://lwn.net/Articles/878104/) — 微妙な保証（型安全であって値が安定しているわけではない）とよくある落とし穴の解説
