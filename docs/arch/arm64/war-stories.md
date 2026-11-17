# ARM64事件簿 {#arm64-war-stories}

> キャッシュコヒーレンシのバグ、TLB shootdownの順序、BTIの強制、SVEコンテキストの破損

各事例は、ARM64アーキテクチャ固有の事情 — 弱いメモリ順序付け、ハードウェアのコヒーレンシ要件、CPU機能の強制、エラッタ対応 — に根ざした実在のバグの類型を追う。

!!! note "これらの事例の書き方について"
    これらは、1つのコミットや CVE に結びついた特定の名前付きインシデントの記録ではない。ARM64 システムで実際に起こる障害パターンから組み立てた、現実的なシナリオである。名前や製品は説明のための架空のものだが、障害の様態は本物である。ページ末尾の参考文献では、各事例が依拠する ARM64 の基盤メカニズム（遅延/コンテキストスイッチされる FPSIMD 状態、TLB/バリアの順序付け、MIDR ベースのエラッタ照合、BTI の強制）を、現行のカーネルソースとアーキテクチャ文書に照らして検証している。サイトの通常の形式 — 日付を特定できる具体的なインシデントで、そのほとんど（すべてではない — [wake_wide()](../../sched/war-stories/wake-wide-heuristic.md) を参照）が修正コミットや CVE に結びついているもの — については、[BPF](../../bpf/war-stories.md)、[スケジューラ](../../sched/war-stories.md)、[割り込み](../../interrupts/war-stories.md)の事件簿ページを参照のこと。

---

## 1. SVEコンテキストスイッチのバグ {#1-the-sve-context-switch-bug}

### 状況 {#setup}

あるストレージドライバは、I/O完了時にデータブロックのCRC32チェックサム計算をSIMDで高速化するよう最適化されていた。開発者はチェックサムのパスで `kernel_neon_begin()` と `kernel_neon_end()` を正しく呼んでいた — 少なくとも本人はそう思っていた。リファクタリングの過程で、チェックサムロジックの一部がworkqueueから呼ばれるヘルパー関数に移された。そのヘルパーには `kernel_neon_begin()` はあったが、早期リターンするエラーパスが `kernel_neon_end()` を飛ばしていた。

カーネルはSVEをサポートするARMv8.2プラットフォーム上で動いていた。科学技術計算のワークロードで、複数のユーザー空間スレッドがBLASルーチンにSVEを使っており、高負荷時にはストレージドライバのチェックサム処理が同じCPU上で実行されていた。

### 何が起きたか {#what-happened}

高負荷のシステムで、チェックサム処理を行うkworkerスレッドが、数回後の呼び出しで一見無関係なローカル状態を時折破壊した — ローカル変数が化けたり、最悪の場合はリターンアドレスが破壊されて、意味をなさないバックトレース付きでクラッシュしたりした。破損は発生時点では無症状で、後になってまったく別のコードで初めて表面化した。

突き止められたメカニズムは次のとおりである。

1. プロセスコンテキストまたはsoftirqコンテキスト（このworkqueueハンドラのような）から `kernel_neon_begin()` を呼ぶカーネル関数は、呼び出し元が所有するバッファ — `struct user_fpsimd_state`、通常はスタック上に宣言される — を渡さなければならない。これらのコンテキストでのカーネルモードNEONの使用は **プリエンプト可能** であり、その期間中ずっとプリエンプションを禁止して実行されるわけではないからである。`kernel_neon_begin(&state)` はそのバッファを `current->thread.kernel_fpsimd_state` に記録し、`TIF_KERNEL_FPSTATE` スレッドフラグをセットする。これにより、このタスクがNEONレジスタをまだ「所有」している間にスケジュールアウトされても、スケジューラはスイッチをまたいでその状態を保存・復元する方法を知ることができる。
2. `kernel_neon_end(&state)` はその契約のもう半分である。`TIF_KERNEL_FPSTATE` をクリアし、`current->thread.kernel_fpsimd_state` を `NULL` にリセットして、このタスクにはもう保存すべき生きたカーネルモードNEON状態がないことをスケジューラに伝える。
3. 早期リターンのパスは `kernel_neon_end()` を飛ばしていた。関数は `state` を保持していたスタックフレームを巻き戻してリターンしたが、`TIF_KERNEL_FPSTATE` はセットされたままで、`current->thread.kernel_fpsimd_state` は今や無効になったスタックアドレスを指したままだった。
4. その状態がクリアされる前にこのkworkerスレッドがプリエンプトされると、（`__switch_to()` から呼ばれる）`fpsimd_thread_switch()` が `TIF_KERNEL_FPSTATE` のセットを検出して `fpsimd_save_kernel_state(current)` を呼び、生きているNEONレジスタの内容を `*current->thread.kernel_fpsimd_state` — つまり古いスタックアドレス — に書き込む。その時点では、そのスタック領域はkworkerスレッドが次に呼んだ何かによって再利用されている可能性が非常に高く、この書き込みはまったく別のコードに属するローカル変数、保存済みレジスタ、あるいはリターンアドレスを黙って上書きしてしまった。

関係する主要なレジスタと関数（`arch/arm64/kernel/fpsimd.c`）:

- `TIF_KERNEL_FPSTATE` — このタスクのカーネルモードFPSIMD/NEON状態をコンテキストスイッチをまたいで保存・復元しなければならないことを示すスレッドフラグ
- `current->thread.kernel_fpsimd_state` — その状態を格納する、呼び出し元が提供したバッファへのポインタ。`kernel_neon_begin()` でセットされ、`kernel_neon_end()` でクリアされる
- `get_cpu_fpsimd_context()` / `put_cpu_fpsimd_context()` — 短時間の `local_bh_disable()`/`local_bh_enable()` のペア（`CONFIG_PREEMPT_RT` では `preempt_disable()`/`preempt_enable()`）。`kernel_neon_begin()`/`kernel_neon_end()` がこの管理情報を更新する間だけ保持され、NEONのクリティカルセクション全体では保持されない
- `fpsimd_thread_switch()` — `__switch_to()` から呼ばれる。切り替え元のタスクに `TIF_KERNEL_FPSTATE` がセットされていれば `fpsimd_save_kernel_state()` を呼び、`task->thread.kernel_fpsimd_state` へ保存する

```c
/* 誤り: 早期リターンが kernel_neon_end() を飛ばし、このスタックフレームが
 * 消えた後も kernel_fpsimd_state が &state を指したままになる */
static int checksum_block(struct request *rq)
{
    struct user_fpsimd_state state;
    int ret;

    kernel_neon_begin(&state);

    ret = validate_header(rq);
    if (ret < 0)
        return ret;   /* バグ: kernel_neon_end() が呼ばれない */

    do_neon_checksum(rq);
    kernel_neon_end(&state);
    return 0;
}

/* 正しい: スタック上に確保した state バッファがスコープを外れる前に、
 * すべてのリターンパスで kernel_neon_end() を呼ぶ */
static int checksum_block(struct request *rq)
{
    struct user_fpsimd_state state;
    int ret;

    kernel_neon_begin(&state);

    ret = validate_header(rq);
    if (ret < 0)
        goto out;

    do_neon_checksum(rq);
out:
    kernel_neon_end(&state);
    return ret;
}
```

### 診断 {#diagnosis}

このバグは無症状かつ非決定的で、破損は原因から遠く離れた場所で表面化した。診断の手順は次のとおりである。

1. **ストレステストで再現する**: チェックサムのワークロードを高負荷で実行し、このkworkerスレッドが、スタックに生きた（しかし放棄された）`kernel_fpsimd_state` ポインタを抱えたまま頻繁にプリエンプトされるようにする。
2. **アサーションを追加する**: `kernel_neon_begin()` で保護された関数から抜けるすべてのリターンパスに対応する `kernel_neon_end()` があるかを監査する。`TIF_KERNEL_FPSTATE` がセットされたままユーザー空間に戻る、長時間ブロックする、または終了するタスクは、ペアリングを漏らしている。
3. **トレーシング**: `kernel_neon_begin()`/`kernel_neon_end()` の前後に `ftrace` フックを追加し、呼び出し箇所とその `state` ポインタを記録する。同じタスク上で、`state` アドレスが対応する `end` に一度も現れない `begin` が決定的な証拠となる。
4. **KASAN/stack-protectorのシグナル**: 破損したメモリは再利用されたスタックスロットなので、同じkworker上のまったく無関係な関数でのstack-protectorのカナリア違反やKASANのstack-out-of-boundsレポートは、このバグと矛盾しない — クラッシュした場所がバグの場所だと思い込まないこと。

### 修正 {#fix}

`kernel_neon_begin()` を呼ぶすべてのカーネルコードパスで、すべての出口パスに対応する `kernel_neon_end()` があり、両方の呼び出しで同じ `state` バッファを使うようにする。よくあるパターンは、上に示した `goto out` イディオムである。NEONの使用が途中で失敗しうる場合は、`kernel_neon_end()` を単一のクリーンアップラベルにまとめておく。

このように保護されたカーネルのNEON使用は、非 `PREEMPT_RT` カーネルではプリエンプト可能である — そもそもプロセス/softirqコンテキストで呼び出し元提供のバッファが必要なのはそのためである。`kernel_neon_begin()` と `kernel_neon_end()` の間の領域全体でプリエンプションが禁止されていると思い込まないこと。短時間だけ禁止されるのは、その2つの呼び出し自体の内部での管理情報の更新だけである。

### 教訓 {#lesson}

ARM64のコンテキストスイッチ可能なカーネルモードNEON状態により、カーネルでのNEON使用は効率的かつプリエンプト可能になっているが、正しさの契約は厳格である。`kernel_neon_begin()` と `kernel_neon_end()` は、呼び出し元が所有する同じバッファを使って、完全にペアにしなければならない。`kernel_neon_end()` を欠いてもすぐには失敗しない — `current->thread.kernel_fpsimd_state` にダングリングポインタを残し、それが次のコンテキストスイッチで初めて被害をもたらす。しかもその被害は、バグの箇所ではなく、たまたまスタックが再利用された場所に及ぶ。コードレビューではこれらのペアをmutexのlock/unlockと同じくらい慎重に — いや、それ以上に — 扱わなければならない。ここでの障害の様態はデッドロックではなく、無秩序な書き込み（wild write）だからである。

---

## 2. TLBIの前のDSBの欠落 {#2-the-missing-dsb-before-tlbi}

!!! note "この事例は今では過去のもの"
    Linux 6.16 以降、`set_pte()` 自体が、有効な非ユーザー PTE に対して自動的に `queue_pte_barriers()`/`emit_pte_barriers()` — `dsb(ishst); isb();` — を呼ぶ（タグ間の差分で確認: `arch/arm64/include/asm/pgtable.h` で v6.15 には存在せず、v6.16 には存在する）。以下の具体的なバグは、ドライバが使う通常の `set_pte()` ヘルパー経由で発生するものなので、6.16 以降のカーネルでは起こりえない。それでも掲載しているのは、根底にある順序付けのルールは今も有効で、`set_pte()` を迂回するあらゆるコード — たとえば独自の PTE レイアウトを持つ IOMMU やアクセラレータ向けに、ドライバが管理するページテーブル形式 — を今も拘束するからであり、また以下の「なぜ 16 コアのサーバーでだけ現れたのか」という推論が今も変わらないからである。

### 状況 {#setup_1}

独自のDMAリマッピングエンジン用のドライバが、デバイスから見えるバッファを制限されたVA空間にマッピングするため、独自のページテーブルエントリ群を管理していた。これはLinux 6.16の自動的なPTEバリアのバッチ処理より前のものである。バッファが差し替えられると、ドライバはPTEを更新し、次に `flush_tlb_range()` を呼んで全CPUの古いTLBエントリをshootdownしていた。このコードはx86の経験を持つエンジニアが書いたもので、16コアのARM64サーバーにデプロイされる前に、シングルコアの開発ボードでテストされていた。

### 何が起きたか {#what-happened_1}

16コアのシステムでは、数時間に1回程度、あるCPUが、リマップされているはずのアドレスでフォールトを起こした。そのフォールトは新しいVAでの変換フォールトだった — MMUのページテーブルウォーカーが、すでに解放された古い物理ページを指す古いPTEを取得していたのである。

根本原因は、PTEの書き込みとTLBI命令の間に `dsb(ishst)` が欠けていたことだった。

ARM64は **弱いメモリモデル** を持つ。PTEへのストアは通常のメモリ書き込みである。TLB無効化（`TLBI` 命令）は、他のCPUのTLBにエントリをフラッシュするよう通知するブロードキャスト操作である。しかしARMアーキテクチャが保証するのは、TLBIがTLBエントリに関して効力を持つことだけである — 先に `DSB`（Data Synchronization Barrier）を発行しない限り、先行するPTEストアが、そのCPUでTLBIが完了する前に他のCPUのMMUウォーカーから見えることは保証 **されない**。

ARM64でのPTE更新に必要な手順は次のとおりである。

```
1. 新しい PTE をメモリに書き込む                (str  x1, [x0])
2. DSB ISHST — Inner Shareable ストアバリア     (dsb  ishst)
   TLBI の前に、PTE ストアがすべての MMU
   ページテーブルウォーカーから見えることを保証する
3. 対象の VA 範囲に対して TLBI を発行する        (TLBI ASIDE1IS / VAAE1IS / etc.)
4. DSB ISH — Inner Shareable バリア             (dsb  ish)
   TLBI が全 CPU で完了したことを保証する
5. ISB — 命令同期バリア                          (isb)
   後続の命令が更新後のマッピングから
   フェッチすることを保証する
```

ドライバのコードをCレベルで表すと次のようになる。

```c
/* 誤り: PTE 書き込みと TLB フラッシュの間に DSB がない */
static void remap_buffer(struct my_dev *dev, unsigned long va,
                         phys_addr_t new_pa)
{
    pte_t *ptep = lookup_pte(dev, va);
    set_pte(ptep, pfn_pte(new_pa >> PAGE_SHIFT, PAGE_KERNEL));
    /* 欠落: ここに dsb(ishst) が必要 */
    flush_tlb_range(&dev->vma, va, va + PAGE_SIZE);
}

/* 正しい: バリアにより TLBI の前に PTE ストアが見えることが保証される */
static void remap_buffer(struct my_dev *dev, unsigned long va,
                         phys_addr_t new_pa)
{
    pte_t *ptep = lookup_pte(dev, va);
    set_pte(ptep, pfn_pte(new_pa >> PAGE_SHIFT, PAGE_KERNEL));
    dsb(ishst);   /* ARM64: TLBI ブロードキャストの前のストアバリア */
    flush_tlb_range(&dev->vma, va, va + PAGE_SIZE);
    /* ARM64 では flush_tlb_range が内部で dsb(ish) + isb を発行する */
}
```

なぜx86では動いたのか。x86は **Total Store Order（TSO）** を採用している。すべてのストア操作は、後続のシリアライズ命令（TLB shootdownに使われる `INVLPG` やIPIなど）より前に、プログラム順で他のプロセッサから見えるようになる。TSOはDSB相当のものを無償で提供してくれる。ARM64の緩いモデルはそうではない。

なぜシングルコアでは動いたのか。CPUが1つなら、競合する他のMMUウォーカーは存在しない。ローカルのTLB無効化は、ローカルのストアバッファから更新後のPTEを見る。

### 診断 {#diagnosis_1}

1. 有効であるはずのアドレスで **断続的な変換フォールト** が発生する。`dmesg` にはカーネルアドレスに対する `do_page_fault` が表示される。
2. フォールトしたVAは最近リマップされた領域と一致し、古いPTE内の物理アドレスは古いマッピングと一致する。
3. 診断用に `dsb(ishst)` を追加し、フォールト率がゼロに下がることを観察して絞り込む。
4. ドライバのすべてのPTE更新パスを、ARM ARM（Architecture Reference Manual）が「break-before-make」とPTE更新について要求する手順と照らし合わせてレビューする。

### 修正 {#fix_1}

PTEの書き込みと、あらゆるTLBIまたは `flush_tlb_*` の呼び出しの間に `dsb(ishst)` を挿入する。ほとんどの場合、ドライバはPTEを直接操作すべきではない — カーネルの `remap_pfn_range()`、`vm_insert_page()`、`io_remap_pfn_range()` を使えばバリアは正しく処理され、Linux 6.16以降は `set_pte()` を直接呼んでもこの特定の順序付けは自動的に処理される。ドライバが汎用の `set_pte()`/`pte_t` の仕組みの完全に外側で独自のページテーブル形式を管理している場合 — 独自のテーブルレイアウトを持つIOMMUやアクセラレータなど — こうした自動処理は一切適用されず、ARM64の完全な手順を明示的に守らなければならない。

### 教訓 {#lesson_1}

ARM64のメモリモデルでは、TLB無効化の前に明示的なストアバリアが必要である。x86から移植されたコードや、ユニプロセッサシステムでしかテストされていないコードは、動いているように見えて、マルチコアのARM64で負荷がかかると失敗する。TLBIの前の `dsb(ishst)` はパフォーマンス上のヒントではない — 汎用カーネルのPTEヘルパーがまだカバーしていないあらゆるページテーブル形式にとって、正しさのためにアーキテクチャ上必須のものである。

---

## 3. キャッシュコヒーレンシの罠: 非コヒーレントデバイスからのDMA {#3-the-cache-coherency-trap-dma-from-a-non-coherent-device}

### 状況 {#setup_2}

ある組み込みSoC（産業制御用のカスタムARM64ボード）は、独自のDMAエンジンを持つPCIeエンドポイントを備えていた。汎用のPCIeカードでは、ルートコンプレックスがカードに代わってCPUキャッシュをスヌープするため、その読み書きは通常コヒーレントに保たれる（デバイスはPCIeの「No Snoop」TLP属性によってトランザクション単位でこれを無効にできる）。しかしこのDMAエンジンはそれと異なり、SoCのバスファブリックを通じてDRAMに直接アクセスしており、そのスヌープ経路を完全に迂回していた。SoCのデータシートにはこのことが記載されていたが、ドライバの作者は「PCIe = キャッシュコヒーレント」と思い込み、それを前提にドライバを書いた。

### 何が起きたか {#what-happened_2}

ドライバは `kmalloc()` で送信バッファを割り当て、パケットデータで埋めてから、DMAエンジンにそのバッファを読み出してカスタムファブリックリンク経由で送信するようプログラムした。デバイスは時折、意図したパケット内容の代わりにゼロや古いデータを送信した。

その流れは次のとおりである。

1. `kmalloc()` は、物理ページに裏付けられたカーネル仮想アドレスへのポインタを返す。このバッファは **キャッシュ可能** であり、この領域に対してCPUのL1/L2キャッシュが有効になっている。
2. ドライバはバッファにパケットデータを書き込む。これらの書き込みはCPUのL1（場合によってはL2）キャッシュに入る。まだDRAMにライトバック（フラッシュ）されていない可能性がある。
3. ドライバはバッファの物理アドレスをDMAエンジンのディスクリプタレジスタに書き込む。
4. DMAエンジンはCPUキャッシュを迂回してDRAMからデータを取得し、CPUの書き込みがまだキャッシュに留まっているため、ゼロや古い値を読み出す。

DMA APIが提供する修正方法:

```c
/* 誤り: 非コヒーレント DMA で kmalloc バッファを直接使っている */
static int bad_driver_tx(struct my_dev *dev, void *data, size_t len)
{
    void *buf = kmalloc(len, GFP_KERNEL);
    memcpy(buf, data, len);
    /* CPU の書き込みはキャッシュにある。デバイスは古い DRAM を読む */
    program_dma(dev, virt_to_phys(buf), len, DMA_TO_DEVICE);
    return 0;
}

/* 正しい: 非コヒーレントなアーキテクチャでキャッシュをフラッシュする dma_map_single() を使う */
static int good_driver_tx(struct my_dev *dev, void *data, size_t len)
{
    void *buf = kmalloc(len, GFP_KERNEL);
    dma_addr_t dma_addr;

    memcpy(buf, data, len);

    dma_addr = dma_map_single(dev->dev, buf, len, DMA_TO_DEVICE);
    if (dma_mapping_error(dev->dev, dma_addr)) {
        kfree(buf);
        return -EIO;
    }
    /* DMA_TO_DEVICE での dma_map_single() は、非コヒーレントなプラットフォームで
     * 該当するキャッシュラインをクリーンする（無効化はしない）。
     * ARM64 ではバッファ内の各キャッシュラインに対して
     * dc cvac（Data Cache Clean by VA to PoC）を発行する — クリーンのみで、
     * クリーン+無効化ではない。 */
    program_dma(dev, dma_addr, len, DMA_TO_DEVICE);
    return 0;
}
```

**デバイスが書き込み**、**CPUが読み出す** バッファ（`DMA_FROM_DEVICE`）では逆になる。DMAの完了後、`dma_unmap_single()`（または `dma_sync_single_for_cpu()`）でキャッシュラインを無効化し、デバイスがDRAMに書いた内容の代わりにCPUがキャッシュ上の古いデータを読まないようにしなければならない。

頻繁に使われる共有バッファには、`dma_alloc_coherent()` が適切な手段である — キャッシュ不可（またはキャッシュを迂回する属性）としてマッピングされたメモリを割り当てるので、どちらの側も明示的なフラッシュ/無効化の呼び出しを必要としない。

```c
/* リングやディスクリプタに最適: 最初からコヒーレントとして割り当てる */
buf = dma_alloc_coherent(dev->dev, size, &dma_addr, GFP_KERNEL);
/* CPU とデバイスの両方が一貫したビューを見る。フラッシュは不要 */
```

デバイスに明示的なキャッシュ管理が必要かどうかは、`dev->dma_coherent`（プラットフォームのコヒーレンシファブリックを記述するDTまたはACPIから設定される）、またはスヌープを提供するIOMMUの有無によって決まる。完全にコヒーレントなインターコネクトを持つプラットフォームでは、`dma_map_single()` はキャッシュ管理に関しては何もしない。非コヒーレントなプラットフォームでは、必要な `dc cvac` の手順を実行する。

### 診断 {#diagnosis_2}

1. デバイスが誤ったデータ — ゼロ、古い値、またはゴミ — を送信する。ランダムな破損ではない。
2. このバグは **低速時には一貫している**（競合の前にキャッシュラインが常にフラッシュされている）が、**負荷時に悪化する**（キャッシュラインが自然に追い出されている可能性が低い）。
3. 一時的な回避策: `pgprot_noncached()` や `pgprot_writecombine()` を使ってバッファのメモリをキャッシュ不可にする — バグが消えれば、キャッシュコヒーレンシが原因だと確定する。
4. SoCのTRM（Technical Reference Manual）とLinuxのDTSで、デバイスノードに `dma-coherent` プロパティがあるかを確認する。
5. ドライバ内のすべての `virt_to_phys()` 呼び出しを監査する — 非コヒーレントなプラットフォームでは、`dma_map_*` を迂回して物理アドレスを直接使っている箇所はすべて危険信号である。

### 修正 {#fix_2}

`virt_to_phys()` を直接使ったレジスタのプログラミングをすべてDMA API（`dma_map_single()` / `dma_unmap_single()` または `dma_alloc_coherent()`）に置き換える。ハードウェアが実際にサポートしていない限り、DTSのデバイスノードに `dma-coherent` を **付けない** ようにする。ディスクリプタリングと制御構造体には `dma_alloc_coherent()` を使う。

### 教訓 {#lesson_2}

洗練されたSoCであっても、DMA可能なすべてのデバイスでキャッシュコヒーレンシが保証されるわけではない。LinuxのDMA APIは、まさにコヒーレンシの要件を抽象化するために存在する。コヒーレントなプラットフォームでは何もせず、非コヒーレントなプラットフォームでは正しいキャッシュ操作を行う。ポータブルなドライバで `dma_map_single()` の代わりに `virt_to_phys()` を直接使うのは常に誤りである — 非コヒーレントなハードウェアでは黙って失敗する。

---

## 4. JITコンパイラにおけるBTI強制によるクラッシュ {#4-the-bti-enforcement-crash-in-a-jit-compiler}

### 状況 {#setup_3}

コンテナオーケストレーションエージェントに組み込まれた言語ランタイムが、ポリシー式を評価するために実行時にネイティブのARM64コードを生成していた。このJITは、生成したコードを意図的に `mmap(PROT_READ | PROT_EXEC | PROT_BTI)` でマッピングしていた — JITが出力したコードへの任意の間接分岐の着地点をブロックするというセキュリティ上の利点のために、わざとそのページをBTI（Branch Target Identification）の強制対象にしていたのである。システムは、BTIサポート付きでビルドされたディストリビューションカーネルを動かす最新のARM64サーバーだった。

### 何が起きたか {#what-happened_3}

ランタイムはシグナルを受けてクラッシュした。`dmesg` には次のように表示された。

```
agent[4321]: unhandled exception: BTI, ESR 0x0000000034000002, undefined instruction in agent[7f3a0000+1000]
```

クラッシュは、ランタイムがJITコンパイル済みの関数ポインタを呼び出したまさにその瞬間に発生した。BTI（ARMv8.5-A）の保護がJITコードへの間接分岐を拒否していたのである。JIT自身がそのページを強制対象にしていたためである。

**BTIの仕組み:**

BTIは、間接分岐（関数ポインタ経由の呼び出し、`blr`、`br`）が指定されたランディングパッド命令にしか着地できないことを強制する新しいメカニズムを追加する。これは次のものによって制御される。

- `SCTLR_EL1.BT0` — EL0（ユーザー空間）に対するBTI強制を有効にする
- `SCTLR_EL1.BT1` — EL1（カーネル）に対するBTI強制を有効にする
- `PSTATE` の `BTYPE` フィールド — 直前に行われた分岐の種類（なし / call / jump）を追跡する

間接分岐が実行されると、CPUは分岐の種類を記録するために `BTYPE` をセットする。着地先アドレスの命令は、次のいずれかでなければならない。

- `BTI c` — 呼び出し（`blr`、間接 `bl`）用のランディングパッド
- `BTI j` — ジャンプ（`br`、間接 `b`）用のランディングパッド
- `BTI jc` — 両方用のランディングパッド
- `NOP` — 一部の構成でのみ有効

着地先アドレスに正しい `BTI` 命令がない場合、CPUはBranch Target Exceptionを発生させる。ESR_EL1はこれをEC=`0x0D`（`ESR_ELx_EC_BTI`、`arch/arm64/include/asm/esr.h` で定義）としてエンコードする。これはデータアボート（EC=`0x24`）とは別物であり、BTI例外は独自の例外クラスとISSエンコーディングを持つ。

カーネルは、ELFバイナリの `.note.gnu.property` セクションに `GNU_PROPERTY_AARCH64_FEATURE_1_BTI` ビットがセットされている場合に、そのプロセスでBTIを有効にする。これは次のように確認できる。

```bash
readelf -n /usr/bin/agent | grep -A2 "AArch64 Instruction Set Architecture"
# 出力には次のように表示される: BTI enabled
```

言語ランタイムのメインバイナリではBTIが宣言され有効になっていた。しかし、そのJITエミッタは `BTI` のプリアンブルなしに生のARM64命令を生成していた。

```c
/* 誤り: JIT のプロローグに BTI ランディングパッドがない */
static void emit_function_prologue(struct jit_ctx *ctx)
{
    /* 最初に出力される命令が実際の関数本体になっている */
    emit(ctx, STP_X29_X30_SP_M16);  /* stp x29, x30, [sp, #-16]! */
    emit(ctx, MOV_X29_SP);          /* mov x29, sp                 */
    /* ... 関数の残り */
}

/* 正しい: 最初の命令として BTI c を出力する */
static void emit_function_prologue(struct jit_ctx *ctx)
{
    emit(ctx, BTI_C);               /* bti c  — 呼び出し用ランディングパッド */
    emit(ctx, STP_X29_X30_SP_M16); /* stp x29, x30, [sp, #-16]!   */
    emit(ctx, MOV_X29_SP);          /* mov x29, sp                  */
    /* ... 関数の残り */
}
```

`BTI c` のエンコーディングは `0xd503245f`（hint空間の命令）である。`BTI j` は `0xd503249f`、`BTI jc` は `0xd50324df` である。

BTIの強制はプロセス全体ではなくページ単位の性質である。明示的に「Guarded Page」（`PTE_GP`）としてマッピングされたページにのみ適用される。カーネルがこれをセットするのは、マッピングが `PROT_BTI` 付きで作成または変更されたとき — このJITのように `mmap()`/`mprotect()` で直接指定した場合か、ELFの `GNU_PROPERTY_AARCH64_FEATURE_1_BTI` ノートが存在しバイナリ全体がBTI対応でビルドされている場合にメイン実行ファイルとその共有ライブラリに対して自動的に — である。生成コードを素の `PROT_READ | PROT_EXEC`（`PROT_BTI` なし）でマッピングするJITは、呼び出し元のバイナリ自体がBTI有効かどうかにかかわらず、そのメモリに対してBTIの強制をまったく受けない — 強制は呼び出し元ではなく、分岐先のページに従う。このJITは意図的にその強制を求めており、まさにそれが欠けたランディングパッドを露呈させたのである。

### 診断 {#diagnosis_3}

1. クラッシュのバックトレースは、`xN` がJITコードのアドレスを保持している `blr xN` の直後の命令を指している。
2. シグナル情報または `dmesg` からESR_EL1をデコードする: `EC=0x0D`（`ESR_ELx_EC_BTI`）で、カーネルの `esr_class_str[]` テーブルでは `"BTI"` と報告される。これは `si_code` `ILL_ILLOPC` 付きの `SIGILL` を発生させる（`do_el0_bti()`、`arch/arm64/kernel/traps.c`）— トリガーは分岐先のチェックであるにもかかわらず、セグメンテーションフォールトではない。
3. JITバッファの先頭4バイトを調べる: `5f 24 03 d5`（`bti c`）であるべきところが、関数本体の最初の命令になっている。
4. JITのページがGuardedであることを確認する: `grep -E '^VmFlags.*\bbt\b' /proc/$(pidof agent)/smaps`（`VM_ARM64_BTI` に対応するカーネルのsmapsの `VmFlags` トークンは2文字の `bt` であり、文字列 "BTI" ではない）。また、CPUのサポートを確認するには `/proc/cpuinfo` のFeatures行に `bti` があるかを確認する。
5. バイナリのBTIノートを確認する: `readelf -n /usr/bin/agent`。

### 修正 {#fix_3}

JITエミッタは、間接呼び出し（`blr`）で到達しうるすべての関数エントリポイントの最初の命令として、`BTI c` を出力しなければならない。ジャンプテーブルや間接 `br` の分岐先には、代わりに `BTI j` を出力する。この変更は、関数プロローグのエミッタに1ワードを追加するだけである。

JITが混在環境（一部の呼び出し元がBTI非対応でありうる）を対象とする場合、すべてのエントリポイントで `BTI jc` を使えば最大限の互換性が得られる。

逆に、最初はコードを素の `PROT_READ | PROT_EXEC` でマッピングするJITは、後から `mprotect(buf, size, PROT_READ | PROT_EXEC | PROT_BTI)`（Linux 5.8以降）で同じ強制を有効にできる — セキュリティ上の利点のために意図的に行う価値はあるが、それはすべてのコード生成パスが必要なランディングパッドを実際に出力するようになってからにすべきである。

### 教訓 {#lesson_3}

BTIはARMv8.5に組み込まれた前方エッジのCFI（Control Flow Integrity）メカニズムであり、プロセス単位ではなくページ単位で強制される。明示的に `PROT_BTI` でマッピングされたメモリ（または、よくあるケースとして、BTI対応の実行ファイル自身のコードとその共有ライブラリ）だけが、そこへの間接分岐に対してランディングパッドのチェックを受ける。生成コードをその強制対象にするJITは本物のセキュリティ上の利点を得るが、その場合はすべてのコード生成パス — 実行時生成コード、eBPFのバックエンド、FFIスタブを含む — が `BTI c` / `BTI j` のプリアンブルを出力しなければならない。これは関数エントリごとに1命令で済む修正だが、強制を有効にする前にJITがBTIのABIを実際に理解している必要がある。

---

## 5. MIDR範囲を誤ったエラッタ回避策 {#5-the-erratum-workaround-with-the-wrong-midr-range}

!!! note "意図的に汎用化したハードウェア"
    この事例では、実在の ARM 設計ではなくプレースホルダーのコア名とパーツ番号を使っている。以下のエラッタ、MIDR 値、リビジョン範囲が、実在の文書化されたシリコンエラッタに結び付けられないようにするためである — メカニズム（`arm64_errata[]` 内の MIDR リビジョン範囲）は完全に汎用的であり、どの実在のコアを保護しているかには依存しない。

### 状況 {#setup_4}

あるSoCベンダーがARM Cortex系のコア（具体的な実在のコアはバグには関係ないので、ここでは「MyCore」と呼ぶ）をライセンスしていた。このコアのリビジョンr0p2にはエラッタがあった。特定の条件下で、メモリパイプライン内の偽のイベントが、実際にはアクセスされていないアドレスに対するフォールトを発生させ、カーネルoopsを引き起こしうるというものである。アップストリームのカーネルの修正は、エラッタが検出されたときに実装定義レジスタを介して回避策を適用していた。

下流のベンダーカーネルチームがこの修正をバックポートした。エラッタ回避策はMIDR_EL1の値に応じて適用されるもので、影響を受けるコアのr0p0、r0p1、r0p2に適用されるべきだった。ところが、バックポートには転記ミスがあり、エラッタテーブルのリビジョン範囲はr0p0とr0p1しかカバーしていなかった。

### 何が起きたか {#what-happened_4}

r0p2のシリコンを搭載して出荷された製品デバイスが、メモリ逼迫時に偽のカーネルフォールトに見舞われた。フォールトアドレスは常にマッピングされておらず、ユーザー空間からは再現できず、確実にマッピングされているカーネルアドレスに対して `do_page_fault` の奥深くで発生した。

**MIDR_EL1レジスタの形式:**

```
Bits [31:24] — Implementer  （例: 0x41 = ARM、設計のライセンサー）
Bits [23:20] — Variant      （メジャーリビジョン、例: 0 = r0）
Bits [19:16] — Architecture （0xf = ARMv8）
Bits [15:4]  — Part number  （コア設計ごとに実装定義）
Bits [3:0]   — Revision     （マイナーリビジョン、例: 2 = p2）
```

`MIDR_CPU_MODEL()` マクロは、実装者（implementer）とパーツ番号で照合する。`arm64_errata[]` テーブルのエラッタエントリは、`midr_range` 構造体（バリアントとリビジョンをエンコードした最初と最後のMIDR値）でリビジョン範囲を指定する。誤ったバックポートは次のとおりである。

```c
/* 誤り: 範囲が r0p1（revision = 1）で終わっており、r0p2 が漏れている */
static const struct midr_range affected_range[] = {
    MIDR_RANGE(MIDR_MYCORE, 0, 0, 0, 1),
    /* variant_min=0, revision_min=0, variant_max=0, revision_max=1 */
};

/* 正しい: 範囲に r0p2（revision = 2）を含めなければならない */
static const struct midr_range affected_range[] = {
    MIDR_RANGE(MIDR_MYCORE, 0, 0, 0, 2),
    /* variant_min=0, revision_min=0, variant_max=0, revision_max=2 */
};
```

起動時、カーネルは `arm64_errata[]` を走査し、各エントリの `matches` 関数を呼ぶ。現在のCPUのMIDRが宣言された範囲内にあれば、エラッタが適用され、カーネルの `arm64_errata[]` エントリが実際に使う形式で起動メッセージが出力される — このコードの `pr_fmt` は `"CPU features: "` であり、各エントリの `.desc` 文字列は単に `"ARM erratum NNNNNN"` となっている（たとえば `arch/arm64/kernel/cpu_errata.c` にある、この件とは無関係の実在のエントリは `"ARM erratum 832075"`、`"ARM erratum 834220"` などとなっている — コア名も「Workaround for」という接頭辞もない）。

```
CPU features: detected: ARM erratum NNNNNN
```

影響を受けたr0p2デバイスでは、このようなメッセージは現れなかった — 回避策が有効になっていないことを示す明確な兆候である。

稼働中のシステムのMIDRを読むには次のようにする。

```bash
cat /sys/devices/system/cpu/cpu0/regs/identification/midr_el1
# 例: 0x00000000410FFFF2
```

下位32ビット `410FFFF2` を上のビットレイアウトに照らして読むと、実装者 `0x41`（ARM）、バリアント `0x0`（r0）、アーキテクチャ `0xf`（ARMv8）、パーツ番号 `0xfff`（このコアのプレースホルダーのパーツ番号）、リビジョン `0x2`（p2）となる。リビジョンフィールド（`bits[3:0]`）は `2`（p2）と読めたが、エラッタテーブルの範囲は `1`（p1）までしかなかった — 1つずれのエラーである。

### 診断 {#diagnosis_4}

1. マッピングされているアドレスでの **偽のカーネルフォールト**。フォールトは非決定的で、負荷に依存する。
2. エラッタ回避策の起動メッセージが出ているかを確認する: `dmesg | grep -i erratum`。（これらのエントリに対するカーネル自身の `.desc` 文字列は「ARM erratum ...」であり、「workaround」ではない — 「workaround」だけでgrepしても見つからない。）
3. sysfsから `midr_el1` を読む: `cat /sys/devices/system/cpu/cpu0/regs/identification/midr_el1`。上で示したようにリビジョンフィールドをデコードする。
4. シリコンのリビジョンをカーネルソース内のエラッタテーブルと突き合わせる。MIDR範囲については `arch/arm64/kernel/cpu_errata.c`（またはベンダーの同等物）を確認する。
5. 範囲にr0p2を含めるようパッチを当てて再起動し、回避策のメッセージが現れて偽のフォールトが止まることを確認する。

### 修正 {#fix_4}

影響を受けるすべてのシリコンリビジョンを含むよう、MIDRのリビジョン範囲を広げる。修正後、製品のr0p2デバイスで起動メッセージが現れることを確認する。将来のバックポートを監査しやすくするため、エラッタIDと影響を受けるリビジョン範囲を明示的に参照するコメントを追加する。

```c
/* MyCore erratum NNNNNN の回避策。
 * 影響を受けるもの: r0p0, r0p1, r0p2。
 * ARM erratum document ID NNNNNN, revision C を参照。 */
static const struct midr_range affected_range[] = {
    MIDR_RANGE(MIDR_MYCORE, 0, 0, 0, 2),
};
```

### 教訓 {#lesson_4}

エラッタ回避策は安全上きわめて重要である。MIDRリビジョン範囲の1つずれは、デバイスを黙って無防備のまま放置する。エラッタ修正のバックポートはすべて、対象ハードウェアに実際に搭載されているシリコンリビジョンに照らして検証しなければならない。sysfsから `midr_el1` を読むのは1行で済むチェックであり、あらゆるブリングアップのチェックリストに含めるべきである。エラッタの有効化を確認する起動時メッセージは貴重な診断シグナルであり、それがないことは警告である。

---

## まとめ {#summary}

| # | バグ | 根本原因 | 検出方法 | ARM64固有か |
|---|-----|-----------|-----------------|----------------|
| 1 | プリエンプション時のスタック破損 | `kernel_neon_begin/end` の不一致により、無効になったスタックフレームを指すダングリングな `kernel_fpsimd_state` ポインタが残る | バグのある関数がリターンした後の無関係なコードでの破損。begin/endの呼び出し箇所に対するftrace | はい — ARM64のコンテキストスイッチ可能なカーネルモードNEON状態はARM64固有 |
| 2 | TLBI後の古いPTE | TLB無効化の前の `dsb(ishst)` の欠落 | 最近リマップされたVAでの断続的な変換フォールト | はい — ARM64の弱い順序付け。x86のTSOはこれを隠す |
| 3 | デバイスが古いDMAデータを読む | キャッシュフラッシュなしの非コヒーレントDMA | デバイスがゼロを送信する。`pgprot_noncached` で確認 | 部分的に — 非コヒーレントDMAは他のアーキテクチャにも存在するが、組み込みARM64でよく見られる |
| 4 | BTI強制によるクラッシュ | JITコードに `BTI c` ランディングパッド命令がない | ESR EC=0x0D（ESR_ELx_EC_BTI）を伴うSIGILL（ILL_ILLOPC） | はい — ARMv8.5のBTIはARM64固有 |
| 5 | エラッタ回避策が適用されない | エラッタテーブル内のMIDRリビジョン範囲の1つずれ | 偽のフォールト。エラッタの起動メッセージがない。`midr_el1` のsysfs | はい — ARM64のMIDRベースのエラッタ基盤 |

---

## 参考文献 {#further-reading}

- [例外モデル](exception-model.md) — EL0-EL3、ESR_EL1のシンドロームのデコード、SCTLR_EL1の制御ビット
- [メモリモデル](memory-model.md) — 弱い順序付け、DSB/ISB/DMBバリア、load-acquire/store-release
- [ページテーブル](page-tables.md) — PTEの形式、TLB管理、`flush_tlb_range()` の内部
- [CPU機能](cpu-features.md) — 機能検出、MIDR_EL1の形式、`arm64_errata[]` 基盤、SVE/BTI/MTEの機能プロービング
- [SpectreとMeltdown](spectre-meltdown.md) — 投機的実行、エラッタに基づく緩和策、SMCCCファームウェアインターフェース
- [Documentation/arch/arm64/silicon-errata.rst](https://docs.kernel.org/arch/arm64/silicon-errata.html) — 回避策が実装されたすべてのエラッタのカーネルによる一覧（事例5の世界）
- [arch/arm64/kernel/cpu_errata.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/kernel/cpu_errata.c) — エラッタ回避策のためのMIDR照合範囲
