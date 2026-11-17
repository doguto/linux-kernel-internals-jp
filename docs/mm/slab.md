# slabアロケータ（SLUB） {#slab-allocator-slub}

> 小さなカーネルオブジェクトの効率的な割り当て

## slabアロケータとは何か {#what-is-the-slab-allocator}

slabアロケータは、ページアロケータとカーネルコードの間に位置する。ページを固定サイズのオブジェクトキャッシュに切り分けることで、内部断片化を減らし、よく使われる構造体の割り当てを高速化する。

```c
void *kmalloc(size_t size, gfp_t flags);
void kfree(const void *ptr);

/* あるいは専用のキャッシュを作成する */
struct kmem_cache *kmem_cache_create(const char *name, size_t size, ...);
void *kmem_cache_alloc(struct kmem_cache *cache, gfp_t flags);
void kmem_cache_free(struct kmem_cache *cache, void *ptr);
```

## なぜページアロケータをそのまま使わないのか {#why-not-just-use-the-page-allocator}

ページアロケータは `4KB` 単位で動作する。しかしカーネルオブジェクトは、それよりずっと小さいことが多い:

| オブジェクト | 典型的なサイズ（x86-64） |
|--------|------------------------|
| `struct inode` | 約600バイト |
| `struct dentry` | 約200バイト |
| `struct file` | 約256バイト |
| `struct sk_buff` | 約256バイト |

200バイトのオブジェクトのためにページ全体を割り当てると、メモリの95%が無駄になる。

## slabという考え方 {#the-slab-concept}

もともとはSunOSに由来する考え方で（[Bonwick, USENIX Summer 1994](https://www.usenix.org/conference/usenix-summer-1994-technical-conference/slab-allocator-object-caching-kernel)）、その概要は次のとおりである:

1. 特定のオブジェクト型のためにページを事前に割り当てる
2. 各ページを固定サイズのスロットに切り分ける
3. 高速な割り当て/解放のためにフリーリストを維持する
4. ページアロケータに返さずにメモリを再利用する

```
256バイトのオブジェクト用の slab（1ページ）:
+-------+-------+-------+-------+-------+-------+...+-------+
| obj 0 | obj 1 | obj 2 | obj 3 | obj 4 | obj 5 |   |obj 15 |
+-------+-------+-------+-------+-------+-------+...+-------+
   ^                 ^                       ^
   |                 |                       |
 使用中             空き                    空き
```

## SLUB vs SLAB vs SLOB {#slub-vs-slab-vs-slob}

Linuxにはこれまで3つのslab実装があった:

| アロケータ | 状態 | 特徴 |
|-----------|--------|-----------------|
| SLAB | [削除済み（v6.8）](https://git.kernel.org/linus/16a1d968358a) | オリジナル、複雑、per-CPUキュー |
| SLUB | **デフォルト**（[v2.6.23以降](https://git.kernel.org/linus/a0acd820)） | より単純、オーバーヘッドが小さい、キューなし |
| SLOB | [削除済み（v6.4）](https://git.kernel.org/linus/c9929f0e344a) | 最小限、組み込みシステム向け |

### なぜSLUBがSLABを置き換えたか {#why-slub-replaced-slab}

オリジナルのSLABアロケータ（Jeff Bonwickによる設計で、2.0でChristoph LameterがLinuxに移植）は、キャッシュごと・CPUごとに *3つ* のslabリストを維持していた。per-CPUキュー、共有キュー、そしてNUMAノードごとのリストである。この3階層構造はCPU間のオブジェクト移動を最小化した（解放されたオブジェクトはまずper-CPUリストに入り、競合を避ける）が、代償もあった:

- 大規模環境での **メモリオーバーヘッド**: 数千のキャッシュ × NUMAノード × 3つのリスト = 大きなメタデータのフットプリント。多数の `kmem_cache` インスタンスを持つ256 CPUのNUMAマシンでは、SLABのメタデータが数百メガバイトを消費した。
- **複雑さ**: 3つの階層間でキューを排出するロジックには微妙なバグがあり、メモリ圧迫下での挙動を推論するのが難しかった。
- **共有キューのロック競合**: per-CPUリストが尽きると、複数のCPUがノードごとの共有キューのロックで競合する可能性があった。

Christoph Lameterは、キューを完全に取り除くことでSLUBを設計した（Linux 2.6.23で2007年にマージ）。3つのリスト間でオブジェクトを移動させる代わりに、SLUBはオブジェクトを元のページ（slab）上に置いたままにし、per-CPUのフリーリストポインタに対して `cmpxchg` のアトミック操作を使うことで、高速パスをロックレスにしている。部分slabリスト（一部が空きで一部が使用中のスロットを持つslab）はCPUごとではなくノードごとであり、より単純な2階層モデルになっている。

その結果、SLUBはメタデータが少なく、ロックも少なく、性能は同等かそれ以上となった。SLUBは2.6.23でデフォルトになった。SLABは互換性のために `Kconfig` オプションとして残され、最終的にLinux 6.8（2024年）で削除された。SLOB（MMUを持たない組み込みシステム向けの小さなfirst-fitアロケータ）は、組み込みプラットフォームがSLUBを使えるだけのRAMを持つようになったため、Linux 6.4（2023年）で削除された。

## SLUBの仕組み {#how-slub-works}

### per-CPU slab {#per-cpu-slabs}

各CPUは、高速な割り当てのための「カレント」slabを持つ:

```
CPU 0                    CPU 1
  |                        |
  v                        v
+--------+              +--------+
| slab   |              | slab   |
| page   |              | page   |
|--------|              |--------|
| free   |              | free   |
| list   |              | list   |
+--------+              +--------+
```

高速パスは、（ハードウェアがサポートしていれば）ロックの代わりにアトミックな `cmpxchg` を使う。

### 割り当ての高速パス {#allocation-fast-path}

`mm/slub.c` より:

> "The fast path allocation (slab_alloc_node()) and freeing (do_slab_free()) are fully lockless when satisfied from the percpu slab (and when cmpxchg_double is possible to use, otherwise slab_lock is taken). They rely on the transaction id (tid) field to detect being preempted or moved to another cpu."

（訳: 高速パスの割り当て（slab_alloc_node()）と解放（do_slab_free()）は、percpu slabから満たされる場合（かつcmpxchg_doubleが使える場合。使えなければslab_lockを取る）には完全にロックレスである。プリエンプトされたことや別のCPUに移動したことの検出は、トランザクションID（tid）フィールドに依存している。）

高速パスは次のとおり:
1. per-CPU構造体から `freelist` と `tid` を読む
2. アトミックなcompare-and-swap（`this_cpu_cmpxchg`）でフリーリストの更新を試みる
3. cmpxchgが失敗した場合（別のCPUやプリエンプションによってtidが変わった場合）は再試行する
4. フリーリストが空なら、ロックを使う低速パスにフォールバックする

これにより、プリエンプションやマイグレーションの下でも正しさを保ちながら、一般的なケースで従来のロックを回避できる。

### フリーリストのエンコーディング {#freelist-encoding}

SLUBはフリーリストのポインタを空きオブジェクト自体の中に格納する:

```
空きオブジェクトのレイアウト:
+------------------+------------------+
| next free ptr    | (rest of object) |
+------------------+------------------+
^
|-- freelist はここを指す

割り当てられると、ポインタは実際のデータで上書きされる
```

### 部分slab {#partial-slabs}

CPUのslabが満杯になると、SLUBはノードの部分リストから一部が埋まったslabを取得する:

```
Per-CPU          Per-Node Partial List
+--------+       +--------+  +--------+  +--------+
| full   | ----> |partial |->|partial |->|partial |
+--------+       +--------+  +--------+  +--------+
```

per-CPU部分リストの挙動は `CONFIG_SLUB_CPU_PARTIAL` で制御される。有効な場合（デフォルト）、各CPUは自身の部分リストを維持し、ノードの部分リストでの競合を減らす。

## kmallocのサイズクラス {#kmalloc-size-classes}

`kmalloc()` は、2のべき乗サイズ用に事前作成されたキャッシュを使う:

```
kmalloc-8      (8 bytes)
kmalloc-16     (16 bytes)
kmalloc-32     (32 bytes)
kmalloc-64     (64 bytes)
kmalloc-96     (96 bytes)    <- 2のべき乗ではない
kmalloc-128    (128 bytes)
...
kmalloc-8k     (8192 bytes)
```

100バイトの要求 -> `kmalloc-128`（28バイトの内部断片化）

### kvmalloc {#kvmalloc}

大きくなる可能性のある割り当てには、`kmalloc()` の代わりに `kvmalloc()` を使う:

```c
void *kvmalloc(size_t size, gfp_t flags);
void kvfree(const void *addr);
```

`kvmalloc()` はまず `kmalloc()` を試し、より大きな割り当てでは `vmalloc()` にフォールバックする。これにより、小さな割り当てを高速に保ちつつ、高次の割り当て失敗を回避できる。

## キャッシュのマージ {#cache-merging}

メモリオーバーヘッドを減らすため、SLUBは似たキャッシュをマージする:

```
struct foo { int a, b, c; };      /* 12バイト */
struct bar { int x, y, z; };      /* 12バイト */

/* どちらも同じ下位キャッシュを使う! */
```

デバッグ時には、ブートオプション `slub_nomerge` で無効にできる。

## SLUBのデバッグ {#slub-debugging}

SLUBには豊富なデバッグ機能がある:

```bash
# ブートオプション
slub_debug=P     # ポイズニング - パターンで埋める
slub_debug=F     # 解放時の整合性チェック
slub_debug=Z     # レッドゾーン - ガードバイト
slub_debug=U     # ユーザー追跡 - 割り当て/解放の呼び出し元を記録
slub_debug=T     # トレース - 割り当て/解放を出力

# あるいは特定のキャッシュに対して
slub_debug=PFZ,kmalloc-256

# sysfs で確認
cat /sys/kernel/slab/kmalloc-256/sanity_checks
```

### ポイズン値 {#poison-values}

| パターン | 意味 |
|---------|---------|
| `0x5a` | 未初期化（`POISON_INUSE`） - 未初期化アクセスを検出 |
| `0x6b` | オブジェクト解放済み（`POISON_FREE`） - use-after-freeを検出 |
| `0xa5` | 終端マーカー（`POISON_END`） - ポイズン領域の終わりを示す |
| `0xbb` | 非アクティブなレッドゾーン（`SLUB_RED_INACTIVE`） - 空きオブジェクトでのバッファオーバーフローを検出 |
| `0xcc` | アクティブなレッドゾーン（`SLUB_RED_ACTIVE`） - 使用中オブジェクトでのバッファオーバーフローを検出 |

## セキュリティ強化 {#security-hardening}

SLUBにはいくつかのセキュリティ機能がある:

| 設定オプション | 目的 |
|---------------|---------|
| `CONFIG_SLAB_FREELIST_RANDOM` | フリーリストの順序をランダム化し、ヒープエクスプロイトを困難にする |
| `CONFIG_SLAB_FREELIST_HARDENED` | フリーリストのポインタをランダム値とXORし、破損を検出する |
| `CONFIG_INIT_ON_ALLOC_DEFAULT_ON` | 割り当て時にメモリをゼロ化する |
| `CONFIG_INIT_ON_FREE_DEFAULT_ON` | 解放時にメモリをゼロ化する |

### KFENCE {#kfence}

KFENCE（Kernel Electric Fence）は、サンプリングベースのメモリ安全性エラー検出器である。[カーネルのドキュメント](https://docs.kernel.org/dev-tools/kfence.html)より:

> "KFENCE is designed to be enabled in production kernels, and has near zero performance overhead."

（訳: KFENCEは本番カーネルで有効化されるよう設計されており、性能オーバーヘッドはほぼゼロである。）

`slub_debug` と違い、KFENCEは割り当てのごく一部しかサンプリングしないため、本番環境で実行できる。

```bash
# 有効かどうかを確認
cat /sys/module/kfence/parameters/sample_interval
```

## 試してみる {#try-it-yourself}

### すべてのslabキャッシュを表示する {#view-all-slab-caches}

```bash
# 全キャッシュの概要
cat /proc/slabinfo

# あるいは slabtop でライブ監視
slabtop -s c    # キャッシュサイズでソート
```

### 特定のキャッシュを調べる {#inspect-specific-cache}

```bash
# sysfs 経由（SLUB）
ls /sys/kernel/slab/kmalloc-256/

# 主要なファイル:
# - object_size: 実際のオブジェクトサイズ
# - slab_size: メタデータを含むサイズ
# - objs_per_slab: slab ページあたりのオブジェクト数
# - partial: 部分 slab の数
# - cpu_slabs: per-CPU slab
```

### キャッシュごとのメモリ使用量 {#memory-usage-by-cache}

```bash
# メモリ消費の多い上位
cat /proc/slabinfo | awk 'NR>2 {print $1, $3*$4}' | sort -k2 -nr | head
```

### 割り当てをトレースする {#trace-allocations}

```bash
# kmalloc のトレーシングを有効化
echo 1 > /sys/kernel/debug/tracing/events/kmem/kmalloc/enable
cat /sys/kernel/debug/tracing/trace_pipe
```

## 主要なデータ構造 {#key-data-structures}

### struct kmem_cache {#struct-kmem_cache}

```c
struct kmem_cache {
    struct kmem_cache_cpu __percpu *cpu_slab;  /* per-CPU データ */
    unsigned int size;          /* メタデータを含むオブジェクトサイズ */
    unsigned int object_size;   /* 実際のオブジェクトサイズ */
    struct kmem_cache_node *node[MAX_NUMNODES];  /* ノードごとのデータ */
    const char *name;
    /* ... */
};
```

### struct kmem_cache_cpu {#struct-kmem_cache_cpu}

```c
/* mm/slub.c より */
struct kmem_cache_cpu {
    union {
        struct {
            void *freelist;        /* 次に利用可能なオブジェクトへのポインタ */
            unsigned long tid;     /* グローバルに一意なトランザクション ID */
        };
        freelist_full_t freelist_tid;
    };
    struct slab *slab;             /* 割り当て元の slab */
    struct slab *partial;          /* 部分的に割り当て済みの slab */
    local_trylock_t lock;          /* 上記のフィールドを保護する */
};
```

## 歴史 {#history}

### SLAB（1996年） {#slab-1996}

SunOSから移植されたもので、[Bonwickの1994年の論文](https://people.eecs.berkeley.edu/~kubitron/courses/cs194-24-S14/hand-outs/bonwick_slab.pdf)に基づいている。

*注: LKMLのアーカイブより前のものである。*

### SLUBの導入（v2.6.22、2007年） {#slub-introduction-v2622-2007}

**コミット**: [81819f0fc828](https://git.kernel.org/linus/81819f0fc828)

**作者**: Christoph Lameter

*注: コミットメッセージに設計の根拠が記載されている。2008年以前のLKMLアーカイブはまばらである。*

### SLABの非推奨化（v6.5、2023年） {#slab-deprecation-v65-2023}

**コミット**: [eb07c4f39c3e](https://git.kernel.org/linus/eb07c4f39c3e)（"mm/slab: rename CONFIG_SLAB to CONFIG_SLAB_DEPRECATED"） | [LKML](https://lore.kernel.org/linux-arm-kernel/20230523091139.21449-1-vbabka@suse.cz/)

**作者**: Vlastimil Babka

SLABが非推奨となり、SLUBが唯一残るアロケータとなった。

### SLABの削除（v6.8、2024年） {#slab-removal-v68-2024}

**コミット**: [16a1d968358a](https://git.kernel.org/linus/16a1d968358a)（"mm/slab: remove mm/slab.c and slab_def.h"） | [LKML](https://lore.kernel.org/lkml/20231113191340.17482-22-vbabka@suse.cz/)

**作者**: Vlastimil Babka

SLABのコードが削除された。現在はSLUBが唯一のslabアロケータである。

## よくある問題 {#common-issues}

### slabの断片化 {#slab-fragmentation}

オブジェクトが割り当てられたまま解放されず、slabページを返却できなくなる。

**デバッグ**: `/sys/kernel/slab/*/partial` を確認する

### メモリリーク {#memory-leaks}

割り当てに対応する解放がない。

**デバッグ**: `slub_debug=U` を使って割り当て箇所を追跡するか、kmemleakを使う。

### キャッシュラインのバウンシング {#cache-line-bouncing}

あるCPUで割り当てて別のCPUで解放するワークロードでのper-CPU slabの競合。

## 悪名高いバグとエッジケース {#notorious-bugs-and-edge-cases}

ヒープの悪用はカーネルエクスプロイトの基本中の基本である。SLUBアロケータの設計 — フリーリストのポインタをオブジェクトと同じ場所に格納すること、per-CPUキャッシュを使うこと、似たキャッシュをマージすること — は、豊富な攻撃対象領域を生み出している。

### ケース1: Netfilterのヒープ範囲外アクセス（CVE-2021-22555） {#case-1-netfilter-heap-out-of-bounds-cve-2021-22555}

#### 何が起きたか {#what-happened}

2021年7月、Andy Nguyen（Google）は、Linux Netfilterに15年間存在していたヒープの範囲外書き込みを実証した。これは最新の緩和策をすべて回避してカーネルコードの実行を達成するものだった。このエクスプロイトはGoogleのkCTFプログラムで1万ドルを獲得した。

#### バグ {#the-bug}

[Project Zeroの解説記事](https://google.github.io/security-research/pocs/linux/cve-2021-22555/writeup.html)より:

脆弱性は `net/netfilter/x_tables.c` に存在する。互換モードで `IPT_SO_SET_REPLACE` が呼ばれると、構造体を32ビット形式から64ビット形式に変換する必要がある。変換関数 `xt_compat_target_from_user()` は、範囲外に4バイトのゼロを書き込む可能性があった。

#### 悪用手法 {#exploitation-technique}

このエクスプロイトは、いくつかの高度な手法を使う:

1. **msg_msgによるヒープスプレー**: `msgsnd()` を使い、制御された `msg_msg` 構造体でヒープをスプレーする
2. **msg_msg->m_list.nextの破壊**: 4バイトのゼロ書き込みがフリーリストのポインタを部分的に上書きする
3. **UAFプリミティブ**: 破壊されたポインタがuse-after-freeの状態を作り出す
4. **カーネルアドレスのリーク**: 解放済みのmsg_msgを読み出してKASLRを無効化する
5. **偽オブジェクトの注入**: 偽オブジェクトをスプレーして実行を制御する

#### 修正 {#the-fix}

**コミット**: [b29c457a6511](https://git.kernel.org/linus/b29c457a6511)（"netfilter: x_tables: fix compat match/target pad out-of-bound write"）

**作者**: Florian Westphal

---

### ケース2: io_uringのuse-after-free（CVE-2022-29582） {#case-2-io_uring-use-after-free-cve-2022-29582}

#### 何が起きたか {#what-happened_1}

2022年、セキュリティ研究者がio_uringのタイムアウト処理における競合状態を発見し、それがuse-after-freeにつながることを示した。このエクスプロイトは、Googleの強化されたkCTF環境でroot権限を獲得した。

#### バグ {#the-bug_1}

[技術解説記事](https://ruia-ruia.github.io/2022/08/05/CVE-2022-29582-io-uring/)より:

io_uringでは、タイムアウトのフラッシュと削除の間に競合が存在する。同じタイムアウト要求が両方のパスで同時に処理されると、2回解放され、use-after-freeが生じる。

#### 悪用手法 {#exploitation-technique_1}

このエクスプロイトは **クロスキャッシュ攻撃** を使う:

1. **UAFを引き起こす**: タイムアウトのパスを競合させて `io_kiocb` を2回解放する
2. **別のオブジェクトで再取得する**: slabページを解放し、別のキャッシュで再割り当てする
3. **型の混同**: 解放された `io_kiocb` のスロットに、今度は別の型のオブジェクトが入る
4. **任意の読み書き**: 混同されたオブジェクトを操作してプリミティブを得る

#### 修正 {#the-fix_1}

**コミット**: [e677edbcabee](https://git.kernel.org/linus/e677edbcabee)（"io_uring: fix race between timeout flush and removal"）

**作者**: Jens Axboe

---

### ケース3: SLUBフリーリスト強化の回避 {#case-3-slub-freelist-hardening-bypasses}

#### 強化策 {#the-hardening}

Linuxは、フリーリストの破壊から保護するために `CONFIG_SLAB_FREELIST_HARDENED` を追加した。フリーリストのポインタは、キャッシュごとのランダム値とポインタ自身のアドレスとでXORされる。

#### 強化策の回避 {#bypassing-the-hardening}

強化策があるにもかかわらず、研究者たちは回避方法を見つけている:

1. **まず情報リーク**: XORキーをリークできれば、強化策は何の保護にもならない
2. **部分的な上書き**: フリーリストのポインタの下位バイトだけを上書きすることで、有効なポインタを作れることがある
3. **二重解放のタイミング**: 解放のタイミングを制御して検出を回避する

| 設定 | 保護 | 回避の難しさ |
|--------|-----------|-------------------|
| `SLAB_FREELIST_HARDENED` | フリーリストのポインタのXOR | 中（リークが必要） |
| `SLAB_FREELIST_RANDOM` | フリーリストの順序のランダム化 | 低（確率的） |
| `RANDOM_KMALLOC_CACHES` | サイズごとに複数のキャッシュ | 中（より多くのスプレーが必要） |

---

### ケース4: キャッシュのマージによるセキュリティ問題 {#case-4-cache-merging-security-issues}

#### 問題 {#the-problem}

SLUBは、メモリオーバーヘッドを減らすために、オブジェクトサイズとフラグが似たキャッシュをマージする。キャッシュがマージされると、異なる型のオブジェクトが同じslabページを共有し、ある型のオブジェクトでのUAFが別の型に影響しうる。

#### 修正 {#the-fix_2}

セキュリティ上重要なキャッシュには、`SLAB_ACCOUNT` または `SLAB_NO_MERGE` を使う。ブートオプション `slub_nomerge` はすべてのマージを無効にする（性能コストあり）。

---

### ケース5: slab外へのアクセスパターン {#case-5-out-of-slab-access-patterns}

#### バグのクラス {#the-bug-class}

多くの脆弱性は、slabオブジェクトのすぐ外側のメモリへのアクセス（off-by-one、線形オーバーフロー）を伴う。標準のSLUBでは、これにより隣接オブジェクトやフリーリストのポインタが破壊される。

#### 検出 {#detection}

```bash
# レッドゾーンを有効にしてオーバーフローを検出
slub_debug=Z

# 本番環境では KFENCE
CONFIG_KFENCE=y
```

---

### まとめ: 得られた教訓 {#summary-lessons-learned}

| バグ | 年 | 根本原因 | 悪用方法 |
|-----|------|------------|--------------|
| CVE-2021-22555 | 2021 | 境界チェックの誤り | msg_msgスプレー |
| CVE-2022-29582 | 2022 | 競合状態 | クロスキャッシュ攻撃 |
| フリーリスト強化の回避 | 継続中 | 強化策の限界 | 情報リーク + 細工 |
| キャッシュのマージ | 設計 | メモリ最適化 | 型の混同 |

**パターン**: SLUBのバグがSLUB自体にあることはまれで、SLUBを *使う* コードの中にある。しかし、悪用と防御の両方において、SLUBの内部を理解することは不可欠である。

## 参考資料 {#references}

### 主要なコード {#key-code}

| ファイル | 説明 |
|------|-------------|
| [`mm/slub.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/slub.c) | SLUBの実装 |
| [`include/linux/slab.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/slab.h) | slabのAPI |
| [`mm/slab_common.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/slab_common.c) | slabの共通コード |

### 主要なコミット {#key-commits}

| コミット | カーネル | 説明 |
|--------|--------|-------------|
| [81819f0fc828](https://git.kernel.org/linus/81819f0fc828) | v2.6.22 | SLUBの導入 |
| [eb07c4f39c3e](https://git.kernel.org/linus/eb07c4f39c3e) | v6.5 | SLABの非推奨化 |
| [16a1d968358a](https://git.kernel.org/linus/16a1d968358a) | v6.8 | SLABの削除 |

### 参考文献 {#further-reading}

- [Bonwick, "The Slab Allocator" (1994)](https://people.eecs.berkeley.edu/~kubitron/courses/cs194-24-S14/hand-outs/bonwick_slab.pdf) - オリジナルの設計論文

### 関連項目 {#related}

- [page-allocator](page-allocator.md) - SLUBがページを取得する先
- [overview](overview.md) - アロケータの階層の中でslabがどこに位置するか
- [バグ索引](bugs/README.md) - mmカーネルバグの全索引

## 参考文献 {#further-reading_1}

- [page-allocator.md](page-allocator.md) — SLUBがページを取得するbuddyシステム
- [vmalloc.md](vmalloc.md) — `kvmalloc()` は、kmallocで満たせない大きな割り当てではvmallocにフォールバックする
- [kasan.md](kasan.md) — Kernel Address Sanitizer。SLUBのデバッグフックを使ってヒープ破損を検出する
- [kfence.md](kfence.md) — 本番環境で動かすことを想定した、低オーバーヘッドなサンプリングベースのメモリ安全性検出器
- [slab-internals.md](slab-internals.md) — SLUBの内部データ構造と高速パスの仕組みの詳細
- [mm/slub.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/slub.c) — SLUBの実装。`slab_alloc_node()` が割り当ての高速パスである
- [mm/slab_common.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/slab_common.c) — アロケータ間で共有される、キャッシュの作成、マージ、kmallocのサイズクラスの設定
- [include/linux/slab.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/slab.h) — 公開slab API: `kmalloc`、`kfree`、`kmem_cache_create`
- [LWN: The SLUB allocator](https://lwn.net/Articles/229984/) — Christoph LameterによるSLUBの最初の紹介と、SLUBがSLABに取って代わる理由
- [LWN: Toward a better object allocator](https://lwn.net/Articles/546021/) — SLUBの改良と、SLAB削除に至る道筋についての議論
- `Documentation/mm/slub.rst` — SLUBの内部とデバッグオプションに関するカーネルドキュメント
