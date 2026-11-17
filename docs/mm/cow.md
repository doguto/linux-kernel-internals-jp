# コピーオンライトはいつ破れるのか? {#when-does-copy-on-write-break}

> COWは理論上はエレガントだが、エッジケースには驚かされる

## COWの約束 {#the-cow-promise}

コピーオンライト（COW）はカーネルの最も重要な最適化の1つである。`fork()`したとき、子プロセスは親のメモリの物理的なコピーを受け取らない。両プロセスは同じpageを共有し、それらは[ページテーブル](page-tables.md)上で読み取り専用とマークされる。どちらかのプロセスが書き込んだときにだけ、カーネルはそのpageをコピーする。

```
After fork():
┌─────────────┐     ┌─────────────┐
│   Parent    │     │    Child    │
│  Page Table │     │  Page Table │
└──────┬──────┘     └──────┬──────┘
       │                   │
       │    (read-only)    │
       └────────┬──────────┘
                ▼
         ┌─────────────┐
         │   Shared    │
         │   Physical  │
         │    Page     │
         └─────────────┘
```

これが`fork()`が速い理由である。コピーは一切なく、ページテーブルを操作するだけだ。では、この共有は実際にはいつ破れるのだろうか?

## COWがコピーを引き起こすとき {#when-cow-triggers-a-copy}

### 1. 共有pageへのあらゆる書き込み {#1-any-write-to-a-shared-page}

明白なケースである。どちらかのプロセスが書き込むと次のようになる。

```c
pid_t pid = fork();
if (pid == 0) {
    // 子プロセスが共有pageに書き込む
    buffer[0] = 'x';  // ページフォールト → COWコピー
}
```

書き込みはページフォールトを引き起こし、カーネルはそのpageが共有されていることを確認すると、[ページアロケータ](page-allocator.md)から新しいpageを割り当て、内容をコピーし、フォールトしたプロセスの[ページテーブル](page-tables.md)を更新する。

### 2. プライベートなファイルマッピングへの書き込み {#2-writing-to-a-private-file-mapping}

`MAP_PRIVATE`のファイルマッピングもCOWを使う。

```c
void *p = mmap(NULL, size, PROT_READ | PROT_WRITE,
               MAP_PRIVATE, fd, 0);  // プライベートマッピング

// 最初はページキャッシュとpageを共有する
p[0] = 'x';  // COW: プライベートなコピーを得る
```

最初のマッピングは[ページキャッシュ](page-cache.md)とpageを共有する。書き込みがCOWを引き起こし、ファイルとは別の道を歩むプライベートなコピーが得られる。

### 3. KSMによる重複排除（とその後の書き込み） {#3-ksm-de-duplication-and-then-writing}

[KSM](ksm.md)（Kernel Same-page Merging）は同一のpageを探してCOWでマージする。

```
Before KSM:
Process A: [Page with "hello"]
Process B: [Page with "hello"]  (identical content)

After KSM merging:
Process A ──┐
            ├──► [Single shared page: "hello"]
Process B ──┘

After Process A writes:
Process A: [Private copy: "hello!"]  ← COW triggered
Process B: [Still sharing original]
```

同一のゲストOSのpageを持つVMがメモリを共有できるのはこのためである。ただしゲストが書き込むまでは。

### 4. fork後の`MADV_WIPEONFORK`領域 {#4-madv_wipeonfork-regions-after-fork}

`MADV_WIPEONFORK`（[カーネル4.14で追加](https://git.kernel.org/linus/d2cd9ede6e19) | [LKML](https://lore.kernel.org/lkml/20170811212829.29186-3-riel@redhat.com/)）でマークされたメモリは、子プロセスではゼロ埋めされて見える。

```c
madvise(addr, len, MADV_WIPEONFORK);
fork();
// 子: ゼロ埋めされたメモリが見える（アクセス時）
// 親: pageは変化しない
```

子プロセスはこの領域について親のページマッピングを継承しない。子がそのメモリにアクセスすると、新しいゼロpageがオンデマンドでフォールトインされる。fork時に即座に割り当てられるわけではないが、内容は実質的に消去される。

## COWが期待どおりに動かないとき {#when-cow-doesnt-work-as-expected}

### 問題1: fork()直後に書き込むパターン {#problem-1-fork-immediate-write-pattern}

古典的なアンチパターンである。

```c
char *big_buffer = malloc(1GB);
memset(big_buffer, 0, 1GB);  // これでpageが割り当てられた

pid_t pid = fork();
if (pid == 0) {
    // 子プロセスがただちに全体に書き込む
    memset(big_buffer, 1, 1GB);  // COWが1GB全体をコピーする
    exit(0);
}
```

子プロセスの`memset`の後、使用中の物理メモリは1GBではなく2GBになる。forkは「タダ」だったが、書き込みはそうではなかった。[メモリのオーバーコミット](overcommit.md)が機能するのはこのためである。カーネルは、ほとんどのCOW pageが実際にはコピーされないと想定している。

**解決策**: 子プロセスがすぐに`exec()`するなら心配は要らない。`exec()`はCOWが起きる前にアドレス空間を破棄する。子プロセスがデータを処理する必要があるなら、[`mmap(MAP_SHARED)`](mmap.md)でメモリを明示的に共有することを検討するか、fork+execのパターンには`posix_spawn()`を使う。

*注: `vfork()`はページテーブルのコピーを回避するが危険である。子プロセスは親のスタックとアドレス空間を丸ごと共有する。子での書き込みはすべて親を破壊する。ただちに`exec()`または`_exit()`する場合にだけ使うこと。*

### 問題2: 読むだけなら安全とは限らない {#problem-2-reading-doesnt-mean-safe}

共有pageの読み取りは安全である……通常は。しかし、「読み取り」の中には実は書き込みであるものがある。

```c
// 読み取りに見えるが……
if (buffer[i] != 0) { ... }  // 安全な読み取り

// これは単なる読み取りではない:
buffer[i]++;  // 読み取り-変更-書き込み = 書き込み = COW

// これも同様（アトミックな読み取り-変更-書き込み）:
__sync_fetch_and_add(&buffer[i], 1);  // やはりCOWを引き起こす
```

### 問題3: ページテーブルのオーバーヘッド {#problem-3-page-table-overhead}

COWはpageをコピーしないが、fork時に**[ページテーブル](page-tables.md)**はコピーする。

```
Parent with large virtual address space:
├── Page tables: tens to hundreds of MB (depends on mapping density)
└── Physical pages: Whatever is actually in use

After fork():
├── Child gets copy of page tables (can be significant)
└── Physical pages: Still shared (COW)
```

大きく疎なアドレス空間を持つプロセスでは、fork中のページテーブルのコピーが無視できない量になりうる。`vfork()`が存在するのはこのためである。`vfork()`はページテーブルを一切コピーしない。

### 問題4: COWとヒュージページ {#problem-4-cow-and-huge-pages}

[ヒュージページ](thp.md)（2MBまたは1GB）は興味深い形でCOWと相互作用する。無名THPの場合、1バイトの書き込みのために2MBをコピーするのを避けるため、カーネルはCOWの前にヒュージページを4KBのpageに分割する。

```
Original 2MB THP (after fork, shared):
┌─────────────────────────────────────────────────┐
│                    2MB page                     │
└─────────────────────────────────────────────────┘
                        │
                        ▼ Write triggers split first
┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ 4K │ 4K │ 4K │... │ 4K │ 4K │ 4K │ 4K │ 4K │ 4K │
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
                        │
                        ▼ Then COW only the written 4K page
```

この分割の挙動は、カーネル4.5頃から無名THPのデフォルトである。しかし**hugetlbfs**のpage（`mmap(..., MAP_HUGETLB)`による明示的なヒュージページ）は分割*されない*。共有された2MBのhugetlbfs pageに1バイト書き込むと、2MB全体がコピーされる。

### 問題5: GUP（get_user_pages）とCOW {#problem-5-gup-get_user_pages-and-cow}

カーネルが（`get_user_pages()`を通じて）DMAのためにユーザーpageをピン留めすると、COWは複雑になる。[Dirty COW脆弱性](https://dirtycow.ninja/)（CVE-2016-5195）はこの領域の競合状態を突いたものだった。

カーネルは現在これをより慎重に扱っているが、ピン留めされたpageとCOWの相互作用は依然として複雑である。仕組みの全体像（getとpin、FOLL_LONGTERM、なぜピン留めがCOWと衝突するのか）については[ユーザーページの取得（GUP）](gup.md)を、生々しい詳細については[`mm/gup.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/gup.c)を参照してほしい。

## 性能への影響 {#performance-implications}

### メモリ帯域幅 {#memory-bandwidth}

COWのコピーはメモリ帯域幅を消費する。多数のプロセスがforkして書き込むと次のようになる。

```
Time:     t0          t1          t2          t3
          │           │           │           │
Parent    [====]      [====]      [====]      [====]
          fork↓
Child1    [====]      [====]──write──[copy]   [====]
                      fork↓
Child2              [====]──write──[copy]     [====]
                              fork↓
Child3                      [====]──write──[copy]
                                    │
                                    ▼
                          Memory bandwidth spike
```

### CPUオーバーヘッド {#cpu-overhead}

COWのページフォールトパス（[`mm/memory.c:do_wp_page()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memory.c)）はよく最適化されているが、タダではない。

1. 新しいfolioを割り当てる（ロックを保持せずに）
2. `__wp_page_copy_user()`でpageの内容をコピーする
3. ページテーブルロックを取得する
4. PTEが変化していないか再確認する（別のスレッドが処理したかもしれない）
5. キャッシュとTLBをフラッシュする（`ptep_clear_flush()`）
6. rmapを更新する（`folio_add_new_anon_rmap()`）
7. 新しいPTEを設定する（`set_pte_at()`）
8. ロックを解放する

コピーはロックを取得する*前に*行われる。これにより、異なるpageに対するフォールトを並行して処理できる。それでも、COWフォールトの多いホットパスでは、オーバーヘッドが積み重なる。

### TLBシュートダウン {#tlb-shootdowns}

COWのコピーが起きると、古い読み取り専用のPTEを、新しいpageを指す書き込み可能なPTEに置き換えなければならない。そのためには、フォールトしたCPU上の古くなったTLBエントリを無効化する必要がある。

`fork()`の後、親と子は*別々の*アドレス空間（`mm_struct`）を持つことに注意してほしい。子がpageをCOWしても、変わるのは子のページテーブルだけで、元のpageへの親のマッピングは有効なままである。ここではプロセスをまたぐTLBシュートダウンは関係しない。

TLBシュートダウンが高くつくのは、複数のCPUが同じ`mm`を共有する*マルチスレッド*プロセスである。あるスレッドが、他のCPUがTLBにキャッシュしているpageをCOWすると、それらのエントリはプロセッサ間割り込み（IPI）によって無効化しなければならない。

## COWの監視 {#monitoring-cow}

```bash
# COWフォールトを監視する
perf stat -e page-faults,minor-faults,major-faults ./program

# COWイベントに絞ってトレースする
perf trace -e 'exceptions:page_fault_*' ./program

# プロセスのCOW関連メモリを確認する（詳細はvirtual-physical-resident.mdを参照）
grep -E "Private|Shared" /proc/<pid>/smaps | head -20

# Private_Clean:  このプロセス専有のpageで、マップ以降書き込まれていない
# Private_Dirty:  このプロセス専有のpageで、書き込まれた（COWされた可能性あり）
# Shared_Clean:   他と共有されたpageで、書き込まれていない
# Shared_Dirty:   他と共有されたpageで、書き込まれた
# smapsの詳しい説明は[Virtual vs Physical vs Resident](virtual-physical-resident.md)を参照
```

## 進化 {#evolution}

### 起源（Linux以前） {#origins-pre-linux}

TENEXの論文（[Bobrow et al., CACM 1972](https://dl.acm.org/doi/10.1145/361268.361271)）はPDP-10上のCOWを説明しており、その最初の開発をSDS 940向けのBBN-LISPシステム（1960年代後半）に帰している。この手法は1980年代にUnixシステムで広く普及した。Linuxは、Linus Torvaldsが1991年に最初のバージョンを書いたときに、Unixの伝統からCOWを受け継いだ。

*注: 元々のLinuxのCOW実装はgitより前のものである。`mm/memory.c`の中核ロジックはこれまで何度も大幅に書き直されてきた。*

### Dirty COW脆弱性（2016年） {#the-dirty-cow-vulnerability-2016}

**CVE-2016-5195**: [commit 19be0eaffa3a](https://git.kernel.org/linus/19be0eaffa3a)（"mm: remove gup_flags FOLL_WRITE games from __get_user_pages()"）で修正 | [LKML（stableへのバックポート）](https://lore.kernel.org/lkml/20161019182853.224077572@linuxfoundation.org/)

COWと`get_user_pages()`の間の競合状態により、特権のないユーザーが読み取り専用ファイルに書き込むことができ、権限昇格が可能になった。このバグは11年前からのものだった。Linus Torvaldsは2005年に修正を試みていた（[commit 4ceb5db9757a](https://git.kernel.org/linus/4ceb5db9757a)）が、その修正はs390での問題のために取り消された。

*注: 2005年の修正は、lore.kernel.orgでの現代的なLKMLアーカイブより前のものである。*

修正コミットのメッセージより:
> *"This is an ancient bug that was actually attempted to be fixed once (badly) by me eleven years ago... In the meantime, the VM has become more scalable, and what used to be a purely theoretical race back then has become easier to trigger."*

（訳: これは古くからあるバグで、実は11年前に私が一度（まずいやり方で）修正しようとしたものだ……その間にVMはよりスケーラブルになり、当時は純粋に理論上の競合だったものが、より簡単に引き起こせるようになった。）

### FOLL_PINとpin_user_pages()（v5.6、2020年） {#foll_pin-and-pin_user_pages-v56-2020}

**コミット**: [eddb1c228f79](https://git.kernel.org/linus/eddb1c228f79)（"mm/gup: introduce pin_user_pages*() and FOLL_PIN"） | [LKML](https://lore.kernel.org/lkml/20200107224558.2362728-12-jhubbard@nvidia.com/)

**作者**: John Hubbard（NVIDIA）

Dirty COWの後、カーネルには次の2つを明確に区別する必要があった。

- **get_user_pages()**: 短期の参照（すぐに解放される）
- **pin_user_pages()**: DMAのための長期のピン（新しいFOLL_PINフラグ）

これにより、COWの仕組みは、pageがDMAのためにピン留めされているときにそれを知り、より慎重に扱えるようになった。

### PG_anon_exclusive（v5.19、2022年） {#pg_anon_exclusive-v519-2022}

**コミット**: [6c287605fd56](https://git.kernel.org/linus/6c287605fd56)（"mm: remember exclusively mapped anonymous pages with PG_anon_exclusive"） | [LKML](https://lore.kernel.org/lkml/20220428083441.37290-13-david@redhat.com/)

**作者**: David Hildenbrand（Red Hat）

GUP/COWの競合に対する決定的な修正である。無名pageが排他的に所有されているかどうかを追跡するpageフラグを追加した。コミットからの重要な洞察は次のとおり。

> *"Let's mark exclusively mapped anonymous pages with PG_anon_exclusive as exclusive, and use that information to make GUP pins reliable and stay consistent with the page mapped into the page table even if the page table entry gets write-protected."*

（訳: 排他的にマップされた無名pageをPG_anon_exclusiveで排他的とマークし、その情報を使ってGUPのピンを信頼できるものにし、ページテーブルエントリが書き込み保護されても、ページテーブルにマップされたpageとの一貫性を保てるようにしよう。）

これにより「refcount > 1」というヒューリスティックが明示的な排他性の追跡に置き換えられ、ピン留めされたpageがあってもCOWの挙動が予測可能になった。

### VMAロック下でのCOW（v6.4以降、2023年） {#cow-under-vma-lock-v64-2023}

**コミット**: [164b06f238b9](https://git.kernel.org/linus/164b06f238b9)（"mm: call wp_page_copy() under the VMA lock"） | [LKML](https://lkml.kernel.org/r/20231006195318.4087158-3-willy@infradead.org)

ページフォールトのスケーラビリティを改善するための、進行中のVMA単位ロックの作業の一部である。多くの場合、COWフォールトを`mmap_lock`を取得せずに処理できるようになった。

### THPのCOW改善（v6.15、2025年） {#thp-cow-improvements-v615-2025}

**コミット**: [1da190f4d0a6](https://git.kernel.org/linus/1da190f4d0a6)（"mm: Copy-on-Write (COW) reuse support for PTE-mapped THP"） | [LKML](https://lkml.kernel.org/r/20250303163014.1128035-14-david@redhat.com)

THPがPTEでマップされている状態でCOWフォールトが起きた場合、カーネルは常にコピーするのではなく、排他的に所有されているサブページを再利用できるようになった。これによりTHPのCOWオーバーヘッドが大幅に減る。

## 関連する歴史 {#related-history}

### MADV_WIPEONFORK（v4.14、2017年） {#madv_wipeonfork-v414-2017}

**コミット**: [d2cd9ede6e19](https://git.kernel.org/linus/d2cd9ede6e19)（"mm,fork: introduce MADV_WIPEONFORK"） | [LKML](https://lore.kernel.org/lkml/20170811212829.29186-3-riel@redhat.com/)

**作者**: Rik van Riel（Red Hat）

fork後に秘密情報（PRNGの状態、PKCS#11のセッションなど）を再生成する必要がある、セキュリティ上重要なアプリケーションのために追加された。アドレス空間のレイアウトを保つ、`MADV_DONTFORK`の代替である。

## ベストプラクティス {#best-practices}

### アプリケーション開発者向け {#for-application-developers}

1. **fork後はすぐにexecする**: サブプロセスを起動するだけなら、メモリに触れる前に`exec()`する
2. **サブプロセスの作成には`posix_spawn()`を使う**: `fork()+exec()`より安全で、多くの場合高速である
3. **本当に共有するデータにはMAP_SHAREDを使う**: 共有したいデータをCOWに頼らないこと。[mmap](mmap.md)を参照
4. **自分が何をしているか正確にわかっているのでなければ`vfork()`は避ける**: 速いが危険である（子が親のスタックを共有する）

### カーネル/ドライバ開発者向け {#for-kerneldriver-developers}

1. **GUPの使用を最小限にする**: pageのピン留めはCOWを複雑にする
2. **長期のピンにはFOLL_PINを使う**: 適切な計上がCOWの仕組みを助ける
3. **fork()のパスをテストする**: COWのバグはfork後にしか現れないことが多い

## 自分で試す {#try-it-yourself}

```bash
# COWの動作を観察する
# ターミナル1: 大きなメモリを持つプロセスを作成する
python3 -c "
import os
data = bytearray(100 * 1024 * 1024)  # 100MB
print(f'Parent PID: {os.getpid()}')
pid = os.fork()
if pid == 0:
    print(f'Child PID: {os.getpid()}')
    input('Press Enter to write (trigger COW)...')
    data[0] = 1  # COWを引き起こす
    input('COW done. Press Enter to exit...')
else:
    os.wait()
"

# ターミナル2: メモリ使用量を監視する
watch -n 1 'ps -o pid,rss,vsz,comm -p <parent_pid>,<child_pid>'
# 子が書き込んだ後にRSSが増える（COWコピー）

# fork+exec中のページフォールトをトレースする
strace -f -e trace=clone,execve -o /tmp/trace.log /bin/ls
```

## 参考文献 {#further-reading}

### 関連ドキュメント {#related-docs}

- [メモリのオーバーコミット](overcommit.md): COWがオーバーコミットを可能にする
- [KSM](ksm.md): COWに基づくpageの重複排除
- [THP](thp.md): ヒュージページでのCOWの挙動
- [プロセスのアドレス空間](mmap.md): VMAとページフォールト処理

### LWNの記事 {#lwn-articles}

- [Dirty COW and clean commit messages](https://lwn.net/Articles/704231/)（2016年）: Jonathan CorbetによるDirty COWの競合とその修正の解説
- [Patching until the COWs come home (part 1)](https://lwn.net/Articles/849638/)（2021年）: Vlastimil BabkaによるGUPのピン対COWのデータリーク問題の解説
- [get_user_pages() and COW, 2022 edition](https://lwn.net/Articles/895439/)（2022年）: David HildenbrandのPG_anon_exclusiveによる解決策

### 主要なLKMLスレッド {#key-lkml-threads}

- [FOLL_PIN patch series](https://lore.kernel.org/lkml/20200107224558.2362728-12-jhubbard@nvidia.com/): John Hubbardのv5.6向けシリーズ
- [PG_anon_exclusive series](https://lore.kernel.org/lkml/20220428083441.37290-13-david@redhat.com/): David Hildenbrandのv5.19向けシリーズ
- [COW under VMA lock](https://lore.kernel.org/linux-mm/20230109205336.3665937-1-surenb@google.com/): Suren BaghdasaryanのVMA単位ロックの作業
