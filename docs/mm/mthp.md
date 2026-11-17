# マルチサイズTHP（mTHP） {#multi-size-thp-mthp}

> 2MBでは大きすぎるワークロードのための、PMD未満のサイズのtransparent huge page

## 従来の2MB THPの問題 {#the-problem-with-classic-2mb-thp}

従来のTHPはPMDサイズのhuge pageを割り当てる。4KBのベースページを使うx86-64では2MBである（PMD_ORDER = 9なので、2^9 = 512ページ × 4KB）。大規模なサーバーワークロードにとってはこれが理想的で、TLBエントリが少なくて済み、大規模な環境でのページフォールト処理も速くなる。しかしそれ以外の用途では、2MBは粒度として適切でないことが多い。

**内部断片化。** 64KBのワーキングセットのために2MBのhuge pageを割り当てると、1984KBが無駄になる。組み込みシステム、Android、モバイルといったメモリ制約のある環境では、これがすぐに積み重なる。

**コンパクションの圧力。** バディアロケータは、各THPを裏付けるために物理的に連続した512個の空きページを見つけなければならない。断片化したシステムではこれがコンパクションを引き起こし、高くつく。コンパクションはページをスキャンしてマイグレーションし、連続した空き領域を作り出す。負荷のかかったシステムでは、これがレイテンシのスパイクを引き起こす。

**フォールトのレイテンシ。** フォールトハンドラがhuge pageを割り当てるときは、ユーザー空間に戻る前に2MBの領域全体をゼロクリアしなければならない。これは4KBのフォールトの512倍の仕事量である。レイテンシに敏感なアプリケーションでは、これが測定可能な差になりうる。

THPに関するカーネルドキュメントは、このトレードオフを明示的に説明している。

> "Page faults are significantly reduced (by a factor of e.g. 4, 8, 16, etc), but latency spikes are much less prominent because the size of each page isn't as huge as the PMD-sized variant and there is less memory to clear in each page fault."

（訳: 「ページフォールトは大幅に（例えば4倍、8倍、16倍などの割合で）減少する一方、各ページのサイズがPMDサイズ版ほど巨大ではなく、各ページフォールトでクリアするメモリが少ないため、レイテンシのスパイクははるかに目立たなくなる。」）

## mTHPとは何か {#what-mthp-is}

Linux 6.10で導入されたマルチサイズTHP（mTHP）は、order 2からPMD_ORDER−1まで（両端を含む）のhuge pageで匿名メモリを裏付けることをカーネルに可能にする。4KBのベースページを使うx86-64では、これはorder 2〜8、すなわち16KBから512KBに相当する。

!!! note "order 1は除外される"
    order-1（8KB）は、ソース中の明示的なマスクによって`THP_ORDERS_ALL_ANON`から除外されている。THPの実装にはorder-1で構造上の制約があり、使用できないためである。

これを表すマクロは`include/linux/huge_mm.h`にある。

```c
/* order 2..PMD_ORDER。order-0とorder-1は除外 */
#define THP_ORDERS_ALL_ANON  ((BIT(PMD_ORDER + 1) - 1) & ~(BIT(0) | BIT(1)))
```

従来のPMDサイズのTHP（専用のPMDレベルのページテーブルエントリを使う）とは異なり、mTHPのページは**PTEマップ**のままである。範囲内の2^order個の4KB PTEはそれぞれページテーブルに存在し続け、その下にあるfolioが物理的に連続しているだけである。このため、「PTEマップされたhuge page」と呼ばれることもある。

!!! note "mTHPはshmem/tmpfsでも利用できる"
    ファイルバックのmTHPサポートがtmpfsとshmem向けに追加された。ファイルバックのページのorder範囲は`THP_ORDERS_ALL_FILE_DEFAULT`によって決まり、`MAX_PAGECACHE_ORDER`までをカバーする。サイズごとの制御は、sysfsの同じ`hugepages-NkB/`ディレクトリに公開されている。

## ARM64の連続PTEハードウェア {#arm64-contiguous-pte-hardware}

ARM64では、アーキテクチャが**連続PTE**（contiguous PTE）というハードウェア機能をサポートしているため、PTEマップされたhuge pageが真に効率的になる。自然にアラインされた一連のPTEが物理的に連続したページを指している場合、ハードウェアはそれらのエントリを1つのTLBエントリ（または少数のエントリ）にまとめることができ、PMDレベルのマッピングがなくてもTLBの圧迫を減らせる。

連続PTEの数はアーキテクチャの設定で決まる。4KBページの場合、デフォルトの`CONFIG_ARM64_CONT_PTE_SHIFT`は4であり、2^4 = 16個の連続したPTEで64KBをカバーする。mTHPの割り当てがちょうどこのウィンドウを埋めると、ハードウェアは64KB全体を1つのTLBエントリにまとめられる。PMDマップされたhuge pageと同じ結果を、より小さな粒度で得られるわけである。

カーネルは`contpte_try_fold()`と`contpte_try_unfold()`（`arch/arm64/include/asm/pgtable.h`内）を使い、PTEの設定・クリアに合わせて連続ビットを透過的に管理する。

!!! tip "これがmTHPがARM64で特に価値を持つ理由である"
    64KBのcontpteサイズは、order-4のmTHPサイズと一致する。ワーキングセットが中程度のモバイルや組み込みのARM64ワークロードは、2MBページの断片化やフォールトレイテンシのコストを払わずに、PMDに近いTLB効率を得られる。

## sysfsインターフェース {#the-sysfs-interface}

mTHPは、実行時にsysfsを通じて完全に制御される。`CONFIG_TRANSPARENT_HUGEPAGE`が有効であれば、カーネルの再ビルドは不要である。

サイズごとの制御は次の場所にある。

```
/sys/kernel/mm/transparent_hugepage/hugepages-<size>kB/
```

カーネルはサポートされる各orderにつき1つのディレクトリを作成する。例えば、PMD_ORDER=9の4KBページのシステムでは、`hugepages-16kB`、`hugepages-32kB`、`hugepages-64kB`、`hugepages-128kB`、`hugepages-256kB`、`hugepages-512kB`、`hugepages-1024kB`、`hugepages-2048kB`のディレクトリが見える。

ディレクトリ名は、ブート時に`(PAGE_SIZE << order) / 1024`からkB値として計算される。

### `enabled`属性（匿名メモリ） {#the-enabled-attribute-anonymous}

```
/sys/kernel/mm/transparent_hugepage/hugepages-<size>kB/enabled
```

4つの値を受け付ける。

| 値 | 意味 |
|-------|---------|
| `always` | 匿名ページのフォールトのたびに、無条件でこのサイズを割り当てる |
| `inherit` | トップレベルの`/sys/kernel/mm/transparent_hugepage/enabled`の設定に従う |
| `madvise` | `madvise(MADV_HUGEPAGE)`でマークされた領域に対してのみ割り当てる |
| `never` | このサイズを完全に無効にする |

有効な値は角括弧で示される。

```
$ cat /sys/kernel/mm/transparent_hugepage/hugepages-64kB/enabled
always inherit [madvise] never
```

これらのサイズごとのフラグを裏付ける3つのグローバルなビットマスクは、`mm/huge_memory.c`内の`huge_anon_orders_always`、`huge_anon_orders_inherit`、`huge_anon_orders_madvise`である。

**ブート時のデフォルト:** PMDサイズのTHPのデフォルトは`inherit`である。他のサイズのデフォルトはすべて`never`である。mTHPを初めて有効にしたいときは、小さいサイズの`enabled`ファイルに明示的に`always`か`madvise`を書き込む必要がある。

### `enabled`属性（shmem/ファイルバック） {#the-enabled-attribute-shmemfile-backed}

ファイルバックのorderでは、属性は同じサイズごとのディレクトリ内で`enabled`という名前だが、shmemサブシステム（`thpsize_shmem_enabled_attr`）を経由して処理される。shmemは、匿名THPが使う値に加えて追加の値（`within_size`、`advise`）をサポートする。

### ブート時の設定 {#boot-time-configuration}

`thp_anon=`パラメータを使って、カーネルコマンドラインからmTHPのサイズを設定できる。

```
thp_anon=16K-64K:always;128K,512K:inherit;256K:madvise;1M-2M:never
```

`thp_anon=`が1回でも指定されると、言及されていないサイズはすべて`never`がデフォルトになる。

### サイズごとの統計 {#per-size-statistics}

サイズごとの各ディレクトリには`stats/`サブディレクトリもある。

```
/sys/kernel/mm/transparent_hugepage/hugepages-<size>kB/stats/
    anon_fault_alloc          # successful allocations on page fault
    anon_fault_fallback       # fell back to a smaller order
    anon_fault_fallback_charge  # fallback due to charge failure
    split                     # folios of this size that were split
    split_failed              # split attempts that failed
    split_deferred            # splits deferred to shrinker
    nr_anon                   # current count of live anon folios at this order
    nr_anon_partially_mapped  # partially mapped (e.g. after COW split)
    swpout / swpout_fallback
    swpin / swpin_fallback
    zswpout
```

これらのカウンタは`struct mthp_stat`（`include/linux/huge_mm.h`で宣言）でper-CPUに追跡され、読み取り時に合算される。

!!! note "カウンタが追跡するのはorder 2..PMD_ORDERのみ"
    `mod_mthp_stat()`ヘルパーは、`order <= 0`または`order > PMD_ORDER`のイベントを黙って捨てる。mTHPの`stats/`ディレクトリでPMDオーダーのページのサイズごとの統計を見ることはない。PMDオーダーは、`/proc/vmstat`にある従来のTHPカウンタで別途追跡される。

## mTHPをサポートする割り当てパス {#allocation-paths-that-support-mthp}

**匿名メモリ（フォールト）:** 6.10以降で主にサポートされているパスである。プロセスが匿名ページでフォールトすると、`__handle_mm_fault()`がTHPのコードを呼び出し、THPのコードは`thp_vma_allowable_orders()`を使って、フォールトしたアドレスのVMAでどのorderが対象になるかを決める。その後フォールトハンドラは、対象となるorderのうち割り当てに成功する最大のものを試みる。

**匿名メモリ（khugepaged）:** khugepagedは現在、PMDサイズのcollapseの機会だけを探す。PMD未満のサイズはkhugepagedによってcollapseされない。

**スワップ:** mTHP対応のスワップパスが、大きなfolioのスワップアウトとスワップインを処理する。statsディレクトリの`swpout`、`swpin`および関連するカウンタがこの活動を反映する。

**shmem/tmpfs:** ファイルバックのmTHPは、内部のshmemマウント（`memfd_create`や共有メモリで使われる）とtmpfsマウントでサポートされている。通常のtmpfsは細かな制御なしに利用可能なすべてのサイズを使い、内部のshmemマウントにはサイズごとの制御がある。

## 断片化への影響 {#impact-on-fragmentation}

従来の2MB THPは、512個の連続ページを見つけるのに成功するか、完全に4KBページへフォールバックするかのどちらかである。中間はない。これは崖を生む。軽く断片化したシステムではhuge pageの適用範囲が広くなるが、中程度に断片化したシステムではまったく得られない。

mTHPはこの中間を埋める。order-4（64KB）は連続した16ページしか必要としない。これはバディアロケータにとって、コンパクションを起こさずに満たすのがはるかに容易である。2MBの割り当てが失敗するようなシステムでも、64KBの割り当てなら16個や32個は容易に提供できるかもしれない。

フォールバックのロジックは明示的である。希望するorderを満たせなければ`anon_fault_fallback`がインクリメントされ、アロケータは有効になっている次に小さいorderを試し、必要なら最終的にorder-0（単一の4KBページ）にまでフォールバックする。

## 性能: mTHPが役立つとき {#performance-when-mthp-helps}

**中程度のサイズの割り当て**で、PMDを丸ごと埋めないものが主な受益者である。128KBのバッファを2MBのTHPとして割り当てると1.875MBが無駄になる。128KBのmTHPなら無駄はゼロである。

**ARM64のワークロード**、特にモバイル（Android）や組み込み。連続PTEハードウェアがあるため、order-4（64KB）のmTHPは、PMDのアラインメントを一切必要とせずに実際のTLB圧縮をもたらす。

**低レイテンシのアプリケーション**で、フォールト時にときおり起きる2MBのゼロクリアに耐えられないもの。mTHPはフォールトのレイテンシの平均値と最悪値の両方を下げる。

**断片化したシステム**で、2MB THPでは`thp_fault_fallback`の比率が高くなるもの。小さいorderのほうが割り当ての成功率が高い。

## カーネルの設定 {#kernel-configuration}

コンパイル時のゲートは`CONFIG_TRANSPARENT_HUGEPAGE`だけである。mTHPは独自の`Kconfig`シンボルを持たない。基本のTHP実装の一部であり、実行時にサイズごとのsysfsインターフェースを通じて完全に制御される。

sysfsでのPMD未満のTHPの挙動は、`CONFIG_TRANSPARENT_HUGEPAGE`と`CONFIG_SYSFS`の両方が有効な場合にのみ利用できる（sysfs属性の定義は`#if defined(CONFIG_TRANSPARENT_HUGEPAGE) && defined(CONFIG_SYSFS)`で囲まれている）。

!!! warning "すべてのサイズを無効にしてもMADV_COLLAPSEは無効にならない"
    サイズごとのすべての`enabled`ファイルに`never`を書き込んでも、`madvise(MADV_COLLAPSE)`の動作は妨げられない。`MADV_COLLAPSE`はこれらの設定を無視し、無条件にPMDサイズのcollapseを強制する。カーネルドキュメントの注記を参照のこと。
    > "Setting 'never' in all sysfs THP controls does **not** disable Transparent Huge Pages globally."

    （訳: 「すべてのsysfsのTHP制御で'never'を設定しても、Transparent Huge Pagesがグローバルに無効になるわけでは**ない**。」）

## 参考文献 {#further-reading}

- コミット[19eaf44954df](https://git.kernel.org/linus/19eaf44954df) — 匿名マルチサイズTHPを導入したLinux 6.10のマージコミット。コミットメッセージとシリーズは、sysfsインターフェースの設計、`THP_ORDERS_ALL_ANON`のエンコーディング、ARM64 contpteという動機を扱っている
- [LWN: Multi-size THP](https://lwn.net/Articles/937239/) — Ryan RobertsのmTHPパッチシリーズを扱った2023年の記事。断片化のトレードオフとARM64の連続PTEの利点を解説している
- `Documentation/admin-guide/mm/transhuge.rst` — サイズごとの`enabled`の全値、`thp_anon=`ブートパラメータの構文、サイズごとの`stats/`カウンタを扱う管理者ガイド。[docs.kernel.org](https://docs.kernel.org/admin-guide/mm/transhuge.html)で閲覧できる
- [mm/huge_memory.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/huge_memory.c) — mTHPの割り当ての入り口：`thp_vma_allowable_orders()`、`huge_anon_orders_always`、`huge_anon_orders_madvise`、およびサイズごとのsysfs属性の定義
- [arch/arm64/include/asm/pgtable.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/include/asm/pgtable.h) — `contpte_try_fold()`と`contpte_try_unfold()`：自然にアラインされた一連のPTEマップされたmTHPページを、単一のハードウェアTLBエントリに変えるARM64の実装
- [thp.md](thp.md) — 従来のPMDサイズTHPの内部、khugepagedデーモン、そしてhuge page全般の動機となるTLBの計算
- [folio.md](folio.md) — `struct folio`と大きなfolio：匿名のmTHP割り当てと、大きなファイルバックのページキャッシュエントリの両方を表すのに使われるカーネルオブジェクト
