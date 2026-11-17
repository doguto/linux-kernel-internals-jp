# VMA単位のロック {#per-vma-locks}

> ページフォールト処理におけるmmap_lockのボトルネックを解消する

## mmap_lockのボトルネック {#the-mmap_lock-bottleneck}

すべてのプロセスは、`mm_struct` ごとに1つの読み書きセマフォ `mmap_lock` を持つ。歴史的に、これはプロセスの仮想メモリレイアウトに触れるほぼすべての操作の関門だった。

- ページフォールトはこれを **読み込み用** に獲得する（他のフォールトと共有アクセス）
- `mmap()`、`munmap()`、`mprotect()`、`mremap()`、`brk()` はこれを **書き込み用** に獲得する（排他）

読み込み側の競合の問題は微妙である。rwsemの読み込みロックは共有されるため、同時にフォールトする複数のスレッドは並列に進めるはずであり、実際に進める。ただしそれはライターが現れるまでの話である。どのスレッドであれ、1回の `mmap()` や `munmap()` の呼び出しが書き込みロックを獲得すると、ライターが終わるまでプロセス内の **並行するすべてのページフォールトがブロックされる**。

頻繁なフォールトとたまのVMA変更が混在するマルチスレッドのワークロード（例: 多数のワーカースレッドが新しいヒープのpageをフォールトで読み込む一方、アロケータが時折 `mmap()` を呼ぶ）では、`mmap_lock` が直列化ポイントとなり、フォールトのスループットを頭打ちにする。

`mmap_lock` は粒度も粗い。VMAツリー全体を保護している。スレッドAが領域Xでフォールトし、スレッドBが領域Yでフォールトしている場合、両者はまったく異なるVMAに触れているにもかかわらず、同じロックで競合する。

## VMA単位のロックの設計 {#per-vma-lock-design}

VMA単位のロック（Linux 6.4でマージ）は、各 `vm_area_struct` に独自の軽量なロックを持たせる。ページフォールトの経路はまずこのVMA単位のロックを試み、VMAが並行して変更されている場合にのみ `mmap_lock` にフォールバックする。

### VMA内のロック状態 {#lock-state-in-the-vma}

`vm_area_struct` の2つのフィールドがVMA単位のロックを実装している（`CONFIG_PER_VMA_LOCK` で有効化される）。

```c
/* vm_area_struct内（include/linux/mm_types.h） */

unsigned int vm_lock_seq;      /* シーケンス番号。書き込みロックのたびに更新される */
refcount_t vm_refcnt;          /* 参照カウント + リーダー/ライターの状態 */
```

`mm_struct` 側の対応するフィールドは次のとおりである。

```c
/* mm_struct内 */
int mm_lock_seq;               /* mmap_lockが書き込みロックされるたびにインクリメントされる */
```

VMAは `vma->vm_lock_seq != mm->mm_lock_seq` のとき **ロックされていない** とみなされる。mmが（たとえば `mmap()` のために）書き込みロックされるたびに `mm_lock_seq` がインクリメントされ、これにより既存のVMA単位の読み込みロックがすべて一斉に無効化される。個々のVMAに触れる必要はない。

`vm_refcnt` フィールドは、リーダー数に加えて、書き込みロック時とVMAのデタッチ時に使われる特別な `VM_REFCNT_EXCLUDE_READERS_FLAG` ビットをエンコードしている。詳細な状態遷移は `include/linux/mm_types.h` の `vm_area_struct` の定義に記述されている。

!!! note "シーケンスカウンタはオーバーフローしてもよい"
    `vm_lock_seq` に付けられたカーネルのコメントは、オーバーフローが許容されることを明記している。オーバーフローが引き起こしうるのは、時折の不要な `mmap_lock` へのフォールバックだけであり、正しさが損なわれることはない。

### VMAの探索: maple tree + RCU {#the-vma-lookup-maple-tree-rcu}

高速なフォールト経路でのVMA探索は、RCUの保護下でmaple tree（`mm->mm_mt`、型は `struct maple_tree`）を使う。maple treeはRCUセーフなB木であり、アドレス範囲をキーとしてVMAを格納する。

`mm/memory.c` の `lock_vma_under_rcu()` がエントリポイントである。

```c
struct vm_area_struct *lock_vma_under_rcu(struct mm_struct *mm,
                                          unsigned long address)
{
    MA_STATE(mas, &mm->mm_mt, address, address);
    struct vm_area_struct *vma;

retry:
    rcu_read_lock();
    vma = mas_walk(&mas);           /* RCUで保護されたmaple treeの走査 */
    if (!vma) { ... goto inval; }

    if (!vma_start_read(vma))       /* VMA単位の読み込みロックを試みる */
        goto inval;                 /* ロック失敗（VMAが変化した）、フォールバック */

    rcu_read_unlock();

    /* アドレス範囲を検証する */
    if (address < vma->vm_start || address >= vma->vm_end) {
        vma_end_read(vma);
        goto inval;
    }
    return vma;  /* 安定したロック済みVMA。mmap_lockは保持していない */
    ...
}
```

`vma_start_read()` は `vma->vm_lock_seq == mm->mm_lock_seq` かどうかを確認する。一致すればVMAは現在書き込みロックされているので、関数はNULLを返す（mmap_lockにフォールバックする）。一致しなければ、acquireフェンスを使って `vm_refcnt` をインクリメントし、再確認したうえで読み込みロックを確立する。

## フォールトの経路 {#the-fault-path}

`mm/memory.c` の `handle_mm_fault()` は、アーキテクチャのフォールトハンドラから、VMAロックか `mmap_lock` のどちらかを保持した状態で呼ばれる。どちらであるかは `FAULT_FLAG_VMA_LOCK` フラグで示される。

```c
/* mm/memory.c */
vm_fault_t handle_mm_fault(struct vm_area_struct *vma, unsigned long address,
                            unsigned int flags, struct pt_regs *regs)
{
    /*
     * ここに来た時点で、VMAロックかmmap_lockのどちらかをすでに保持している
     * （どちらであるかはFAULT_FLAG_VMA_LOCKでわかる）。
     */
    if (unlikely(is_vm_hugetlb_page(vma)))
        ret = hugetlb_fault(vma->vm_mm, vma, address, flags);
    else
        ret = __handle_mm_fault(vma, address, flags);
    ...
}
```

アーキテクチャ固有のフォールトハンドラ（例: x86の `do_user_addr_fault()`）は、まず `lock_vma_under_rcu()` を試みる。成功すれば `FAULT_FLAG_VMA_LOCK` を設定して `handle_mm_fault()` を呼ぶ。RCUの高速パスが失敗した場合（VMAの変更が進行中、VMAが見つからない、あるいはアドレス範囲の不一致）は、通常の方法で `mmap_lock` を読み込み用に獲得する方式にフォールバックする。

`__handle_mm_fault()` 内のフォールトハンドラが、`mmap_lock` なしでは先に進めないと判断した場合（たとえばブロッキングI/Oの後にリトライする必要があるとき）は、`vma_end_read()` でVMAロックを解放し、`VM_FAULT_RETRY` を返す。呼び出し側はその後 `mmap_lock` を獲得し直す。

## VMAの書き込みロック {#write-locking-a-vma}

VMAを変更しなければならないとき（例: `mprotect()` の間、`mmap()` 中の分割/結合、`munmap()` 中のマップ解除）、カーネルは `vma_start_write()` を呼ぶ。

```c
/* mm/mmap.c と mm/memory.c — 呼び出し側 */
vma_start_write(vma);
```

`vma_start_write()`（`mm/mmap_lock.c` 内）は、`mmap_lock` がすでに書き込み用に保持されていることを要求する。これは `vma->vm_lock_seq = mm->mm_lock_seq` を設定し、書き込みロックが解放されて `mm_lock_seq` が再びインクリメントされるまで、このVMAに対する `vma_start_read()` を失敗させる。

このプロトコルが正しさを保証する。VMAを変更するあらゆる操作は、まず `mmap_lock` を書き込み用に保持し、それから `vma_start_write()` を呼ばなければならない。これにより、VMA単位の読み込みロックを保持しているフォールトハンドラが、途中までの変更を目にすることはなくなる。

## 依然としてmmap_lockを必要とする操作 {#operations-that-still-require-mmap_lock}

VMA単位のロックが高速化するのは、フォールトの **読み込み** 側だけである。次の操作は依然として書き込み用の `mmap_lock` を必要とする。

| 操作 | 理由 |
|-----------|--------|
| `mmap()` / `munmap()` | VMAツリーへの挿入/削除 |
| `mprotect()` | VMAフラグの変更 |
| `mremap()` | VMAのアドレス範囲の変更 |
| `brk()` | ヒープ境界の変更 |
| フォールト中のVMAの分割/結合 | 隣接するVMAを変更する |
| `execve()` | アドレス空間全体を破棄する |

読み込み用の `mmap_lock` は、VMA単位のロックの経路が失敗したあらゆるフォールトのフォールバックとして、依然として使われている。

## 可観測性 {#observability}

VMA単位のロックにsysfsのノブはない。`CONFIG_PER_VMA_LOCK=y`（サポートされているアーキテクチャでは、カーネル6.4以降のデフォルト）であれば、この機能は常に有効である。

4つのvmstat形式のカウンタがVMA単位のロックの活動を追跡する（`include/linux/vm_event_item.h` で定義されている）。

| カウンタ | 意味 |
|---------|---------|
| `VMA_LOCK_SUCCESS` | 高速パスが成功し、フォールトがVMA単位のロックの下で処理された |
| `VMA_LOCK_ABORT` | 高速パスが失敗し、`mmap_lock` にフォールバックした |
| `VMA_LOCK_RETRY` | 高速パスがリトライされた（例: 一時的な失敗の後） |
| `VMA_LOCK_MISS` | 探索とロックの間にVMAがデタッチされたため、リトライした |

これらは標準の `count_vm_vma_lock_event()` 基盤を通じて読める。`CONFIG_PER_VMA_LOCK` が有効なカーネルでは `/proc/vmstat` に現れる。

!!! tip "競合の診断"
    `VMA_LOCK_SUCCESS` に比べて `VMA_LOCK_ABORT` が多い場合、そのワークロードはVMAの変更中に頻繁にフォールトしている。アプリケーションが、重いフォールトを起こすスレッドから高頻度で `mmap()`/`munmap()` を呼んでいるなら、これは想定どおりである。

## 制限と特殊なケース {#limitations-and-special-cases}

**hugetlbのフォールト** はVMA単位のロックを使わない。`handle_mm_fault()` は `VM_HUGETLB` のpageを `hugetlb_fault()` に回し、これは独自の `hugetlb_vma_lock_read()` / `hugetlb_vma_lock_write()` 基盤を使う。hugetlbのVMAのための別個のロック方式である。

**FAULT_FLAG_RETRY_NOWAIT** は `FAULT_FLAG_VMA_LOCK` と組み合わせられない。カーネルは `sanitize_fault_flags()` でこれをアサートしている。

```c
/* mm/memory.c */
if (WARN_ON_ONCE((*flags &
        (FAULT_FLAG_VMA_LOCK | FAULT_FLAG_RETRY_NOWAIT)) ==
        (FAULT_FLAG_VMA_LOCK | FAULT_FLAG_RETRY_NOWAIT)))
    return VM_FAULT_SIGSEGV;
```

理由: `FAULT_FLAG_RETRY_NOWAIT` は `VM_FAULT_RETRY` の際にロックが手放されないことを前提としているが、VMA単位のロックのプロトコルではロックを手放す必要がある。

**userfaultfd** には独自の相互作用がある。userfaultfdのコード（`mm/userfaultfd.c`）は一部の経路で `lock_vma_under_rcu()` を直接使い、`vma_start_read_locked()` と `vma_end_read()` を明示的に呼ぶ。

**madvise** は一部の挙動（例: `MADV_DONTNEED` の範囲）について、最近のカーネルで追加された範囲単位の経路で `lock_vma_under_rcu()` を使う。

!!! warning "VMA単位のロックにはCONFIG_PER_VMA_LOCKが必要"
    `CONFIG_PER_VMA_LOCK` が設定されていない場合、この機能全体がコンパイルから除外される。そのようなカーネルでは、すべてのフォールトが `mmap_lock` を経由する。ほとんどの本番向けディストリビューションのカーネルは6.4以降でこれを有効にしている。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/mmap_lock.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mmap_lock.h) — `vma_start_read()`、`vma_end_read()`、`vma_start_write()`、およびシーケンスカウンタのヘルパー
- [mm/mmap_lock.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mmap_lock.c) — `lock_vma_under_rcu()`: RCUとVMA単位のロックを組み合わせた高速パスのVMA探索
- [include/linux/mm_types.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mm_types.h) — `vm_lock_seq`、`vm_refcnt`、`mm_lock_seq` のフィールド定義と、それぞれのロックの規約

### カーネルコミット {#kernel-commits}

- [5e31275cc997](https://git.kernel.org/linus/5e31275cc997) — Linux 6.4: "mm: add per-VMA lock and helper functions to control it"（Suren Baghdasaryan、Google）
- [d4af56c5c7c6](https://git.kernel.org/linus/d4af56c5c7c6) — Linux 6.1: maple treeによるVMAの格納。スケーラブルなVMA単位のロックの前提条件

### 関連ページ {#related-pages}

- [rcu-mm.md](rcu-mm.md) — `SLAB_TYPESAFE_BY_RCU` とmaple treeが、VMA単位のロックの土台となるロックフリーなVMA探索をどう実現しているか
- [mmap.md](mmap.md) — 依然として粗い粒度の `mmap_lock` の書き込み側を必要とする操作

### LWNの記事 {#lwn-articles}

- [LWN: Per-VMA locks](https://lwn.net/Articles/906852/) — VMA単位のロックの設計と動機を扱った2022年の記事
- [LWN: Scalable page-fault handling](https://lwn.net/Articles/932298/) — Linux 6.4でのマージとベンチマークを扱った続報
