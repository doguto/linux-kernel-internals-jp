# Kmemleak（カーネルメモリリーク検出器） {#kmemleak-kernel-memory-leak-detector}

> ガベージコレクション風のスキャンで、参照されなくなったカーネルの割り当てを見つける

## Kmemleakとは何か {#what-is-kmemleak}

Kmemleakは、カーネル内の**メモリリーク**、すなわちどのポインタからももはや参照されていないのに一度も解放されていない割り当てを検出するカーネルサブシステムである。これらは*孤立した*（orphaned）割り当てと呼ばれることもある。即座にクラッシュを引き起こすことはないが、決して回収できないメモリを気付かれないうちに消費し、最終的にはメモリの枯渇につながる。

Kmemleakは[`Documentation/dev-tools/kmemleak.rst`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/dev-tools/kmemleak.rst)に文書化されており、[`mm/kmemleak.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kmemleak.c)に実装されている。

## KmemleakとKASAN・KFENCEの違い {#how-kmemleak-differs-from-kasan-and-kfence}

Kmemleak、[KASAN](kasan.md)、[KFENCE](kfence.md)はいずれもカーネルメモリを守るが、対象とする障害の種類は根本的に異なる:

| ツール | 検出するもの | 仕組み | 用途 |
|------|----------------|--------------|------------|
| Kmemleak | メモリ**リーク**（一度も解放されない） | ガベージコレクション風のスキャン、灰色塗り | テスト、デバッグビルド |
| [KASAN](kasan.md) | **use-after-free**、範囲外アクセス、二重解放 | シャドウメモリ + コンパイラによる計装 | テスト、CI |
| [KFENCE](kfence.md) | use-after-free、範囲外アクセス（サンプリング） | ガードページ、ハードウェアのページフォールト | 本番環境、常時有効 |

バッファが割り当てられ、その唯一のポインタが解放されないまま上書きされ、その割り当てが忘れ去られた場合を考える。KASANとKFENCEは何も報告しない（不正なアクセスは起きていない）。これを報告するのはkmemleakだけである。

逆に、kmemleakは解放済みのポインタに再びアクセスされたことを教えてはくれない。それはKASANの領分である。

## Kmemleakの仕組み {#how-kmemleak-works}

### 灰色塗りアルゴリズム {#the-gray-coloring-algorithm}

Kmemleakは、**ガベージコレクタの三色マーキング**に着想を得たアルゴリズムを使う。追跡されている各割り当てには、3つの状態のいずれかが割り当てられる:

```
White  ── not yet reachable from any root; candidate for reporting
Gray   ── reachable (a pointer to it was found during scanning)
Black  ── explicitly told to ignore (kmemleak_ignore only)
```

スキャンの終わりに:

- **灰色のオブジェクト**は生存しているとみなされ、報告されない。
- **白のオブジェクト**（一度も灰色に塗られなかったもの）は*リークしている可能性がある*。スキャンしたメモリのどこにも、それらを指すポインタが見つからなかったということである。

このアルゴリズムは、[`mm/kmemleak.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kmemleak.c)のソースの先頭に記述されている。

### 何がスキャンされるか {#what-gets-scanned}

ポインタを見つけるため、kmemleakは次のものをスキャンする:

1. カーネルイメージの**データセグメントとBSSセグメント** — 割り当てへのポインタを保持している可能性のあるグローバル/静的変数
2. 解放される前の**initセクション**（`.init.data`）
3. **CPUごとのデータ領域** — 割り当てのポインタを保持しうるCPUごとの変数
4. **すべてのスレッドのスタック** — 各タスクのカーネルスタックが、ポインタらしき値を求めてスキャンされる
5. `kmemleak_scan_area()`で明示的に登録された**追加のスキャン領域** — 上記でカバーされないメモリ領域にポインタを格納するサブシステムが使う

これらのいずれかの領域にある値のうち、追跡中の割り当てを指すポインタの**ように見える**ものがあれば、その割り当ては灰色（到達可能）とマークされる。

### 「ポインタのように見える」とはどういう意味か {#what-looks-like-a-pointer-means}

Kmemleakは保守的な手法を使う。**追跡中の割り当てのアドレス範囲内に収まるワードサイズの値はすべて、有効なポインタとして扱われる**。これは誤検出を避けるため、意図的に緩くしてある。

その代償として見逃しが生じる。ポインタがkmemleakの読めない形式（例: XORで難読化された連結リストのポインタ、詰め込まれたビットフィールド、直接のポインタではなくオフセット）で格納されていると、kmemleakはそれを見落とし、その割り当てを誤ってリークとして報告する。これは既知の制限である。後述の[制限事項](#limitations)を参照。

### 割り当ての追跡 {#tracking-allocations}

Kmemleakは、[`include/linux/kmemleak.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/kmemleak.h)で宣言されたコールバックを通じて、カーネルのアロケータにフックする:

| アロケータのイベント | Kmemleakのコールバック |
|----------------|-------------------|
| `kmalloc()` / `kmem_cache_alloc()` | `kmemleak_alloc()` — 新しいオブジェクトを登録する |
| `kfree()` / `kmem_cache_free()` | `kmemleak_free()` — オブジェクトの登録を解除する |
| `vmalloc()` | `kmemleak_vmalloc()` |
| `vfree()` | `kmemleak_free()` |

これらのフックはslabアロケータの奥深く（[`mm/slab_common.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/slab_common.c)、[`mm/slub.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/slub.c)）から呼び出され、`CONFIG_DEBUG_KMEMLEAK`が設定されていない場合はコンパイル時に完全に取り除かれる。

追跡される各オブジェクトは`struct kmemleak_object`（`mm/kmemleak.c`で定義）で表され、これは次のものを保持する:

- 割り当てのポインタとサイズ
- 割り当て時のスタックトレース（`kmemleak_alloc()`の時点で取得）
- 参照カウントと状態フラグ
- リークと判定する前に必要なポインタ参照の最小数（デフォルトは1。`kmemleak_alloc()`の`min_count`パラメータで設定）

### スキャンスレッド {#the-scan-thread}

カーネルスレッド（`kmemleak` kthread）が定期的に`kmemleak_scan()`を実行する。スキャンは次のように行われる:

1. **すべてのオブジェクトを白にマークする**（候補の状態）
2. **すべてのスキャン領域をたどり**（データ、BSS、CPUごとの領域、スタック）、ポインタ値を探す
3. **到達可能なオブジェクトを灰色に塗る** — スキャンしたルートから指されているオブジェクトは、`make_gray_object()`によって到達可能とマークされる
4. **生き残ったものを報告する** — 走査全体を終えてもまだ白のオブジェクトは、リークの可能性があるものとして報告される

スキャンはdebugfsインターフェースを介してオンデマンドでも起動できる（[Kmemleakの使い方](#using-kmemleak)を参照）。

## Kmemleakを有効にする {#enabling-kmemleak}

### カーネルの設定 {#kernel-configuration}

```
CONFIG_DEBUG_KMEMLEAK=y

# Optional: early log buffer for boot-time leaks
CONFIG_DEBUG_KMEMLEAK_DEFAULT_OFF=n   # Enable at boot (default)
# or
CONFIG_DEBUG_KMEMLEAK_DEFAULT_OFF=y   # Boot disabled; enable via boot param

# Auto-scan interval in seconds (default: 600)
CONFIG_DEBUG_KMEMLEAK_AUTO_SCAN=y
```

kmemleakを有効にするには`CONFIG_DEBUG_KMEMLEAK=y`が必要である。関連するオプションは[`lib/Kconfig.debug`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/Kconfig.debug)で定義されている。

!!! warning "本番環境向けではない"
    Kmemleakは測定可能なメモリとCPUのオーバーヘッドを追加する（[性能上のオーバーヘッド](#performance-overhead)を参照）。開発用やデバッグ用のビルドを想定したものであり、本番環境での運用向けではない。

### ブートパラメータ {#boot-parameters}

```bash
# kmemleakを完全に無効にする（組み込まれていても）
kmemleak=off

# CONFIG_DEBUG_KMEMLEAK_DEFAULT_OFF=yであっても有効にする
kmemleak=on
```

### Kmemleakが有効であることを確認する {#verifying-kmemleak-is-active}

```bash
# 設定を確認する
zcat /proc/config.gz | grep KMEMLEAK

# ブートメッセージを確認する
dmesg | grep kmemleak

# debugfsインターフェースが存在することを確認する
ls /sys/kernel/debug/kmemleak
```

## Kmemleakの使い方 {#using-kmemleak}

### debugfsインターフェース {#the-debugfs-interface}

Kmemleakは`/sys/kernel/debug/kmemleak`に単一のファイルを公開しており、これは制御チャネル（書き込み）とレポートインターフェース（読み込み）の両方の役割を果たす。

#### レポートを読む {#reading-reports}

```bash
# スキャンを起動してから結果を読む
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak
```

典型的なリークレポートは次のようになる:

```
unreferenced object 0xffff88801a2b4c00 (size 128):
  comm "insmod", pid 1423, jiffies 4295032145 (age 142.340s)
  hex dump (first 32 bytes):
    74 65 73 74 20 62 75 66 66 65 72 20 63 6f 6e 74  test buffer cont
    65 6e 74 00 00 00 00 00 00 00 00 00 00 00 00 00  ent.............
  backtrace:
    [<00000000deadbeef>] kmalloc_trace+0x2a/0x40
    [<00000000cafebabe>] leaky_function+0x34/0x60 [my_module]
    [<00000000f00df00d>] do_one_initcall+0x80/0x2a0
    [<00000000abcd1234>] do_init_module+0x5e/0x220
```

**レポートの読み方**:

| フィールド | 意味 |
|-------|---------|
| `unreferenced object 0xffff...` | それを指すポインタが見つからなかった割り当てのアドレス |
| `size 128` | 割り当てのバイト単位のサイズ |
| `comm / pid / jiffies` | 割り当て時のタスク名、PID、カーネルのタイムスタンプ |
| `age` | 割り当てからの経過秒数 |
| `hex dump` | 割り当ての先頭バイト（オブジェクトの種類の特定に役立つ） |
| `backtrace` | 割り当て時のコールスタック — **リークがどこで作られたか**を示す |

バックトレースはメモリが割り当てられた場所を指しており、通常はそこから調査を始める。見つけるべきバグは、リークそのもの（ポインタが失われた場所）である。

#### 制御コマンド {#control-commands}

```bash
# 即座にスキャンを起動する
echo scan > /sys/kernel/debug/kmemleak

# 現在のレポートをすべてクリアする（きれいな基準状態を得るのに便利）
echo clear > /sys/kernel/debug/kmemleak

# 実行時にkmemleakを無効にする（新しい割り当ての追跡を止める）
echo off > /sys/kernel/debug/kmemleak

# 特定のアドレスにある追跡中のオブジェクトの情報をダンプする
echo dump=0xffff88801a2b4c00 > /sys/kernel/debug/kmemleak
```

### ワークフロー: カーネルモジュールのリークを見つける {#workflow-finding-a-leak-in-a-kernel-module}

```bash
# 1. モジュールをロードする前に、きれいな基準状態を得る
echo clear > /sys/kernel/debug/kmemleak
echo scan  > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak    # 空であるか、既存のリークが表示されるはず

# 2. モジュールをロードして動作させる
modprobe my_module
# ... my_moduleを動作させるワークロードを実行する ...

# 3. モジュールをアンロードする（リークはこの時点で孤立しているはず）
rmmod my_module

# 4. 新たにスキャンを起動する
echo scan > /sys/kernel/debug/kmemleak

# 5. 結果を読む
cat /sys/kernel/debug/kmemleak
```

!!! tip "複数回のスキャン"
    ロード後の最初の読み込みには、一時的な割り当てが含まれていることがある。各回の間に数秒待ちながら、`echo scan > /sys/kernel/debug/kmemleak`を2回目、3回目と実行すること。複数回のスキャンにわたって残り続けるオブジェクトほど、本当のリークである可能性が高い。

### 自動スキャン {#automatic-scanning}

デフォルトでは、kmemleakはカーネルスレッドを使って600秒（10分）ごとに自動スキャンをスケジュールする。このスレッドは次のように確認できる:

```bash
ps aux | grep kmemleak
# root         23  0.0  0.0      0     0 ?  S    boot  0:00 [kmemleak]
```

自動スキャンは手動スキャンと同じ出力を生成する。いつでも`/sys/kernel/debug/kmemleak`を確認すれば、蓄積された結果を見ることができる。

## 割り当てへのアノテーション {#annotating-allocations}

Kmemleakは、スキャンできない方法でポインタが格納されている割り当て、たとえばXOR連結リスト、エンコードされたポインタ、ハードウェアのディスクリプタリング、あるいはプログラマが意図的なものとわかっている長寿命の割り当てに対して、誤検出を報告することがある。[`include/linux/kmemleak.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/kmemleak.h)のAPIは、こうした報告を抑制するためのアノテーションを提供している。

### kmemleak_not_leak {#kmemleaknotleak}

```c
#include <linux/kmemleak.h>

void *buf = kmalloc(size, GFP_KERNEL);
/* bufはハードウェアレジスタに格納される — kmemleakはそれらをスキャンできない */
writel(virt_to_phys(buf), hw_reg);
kmemleak_not_leak(buf);
```

`kmemleak_not_leak(ptr)`は、この割り当てが**意図的に追跡対象外である**ことをkmemleakに伝える。スキャン中にそれを指すポインタが見つからなくても報告されない。kmemleakからは見えない経路で割り当てに到達可能だとわかっている場合に使う。

### kmemleak_ignore {#kmemleak_ignore}

```c
void *scratch = kmalloc(size, GFP_KERNEL);
kmemleak_ignore(scratch);
```

`kmemleak_ignore(ptr)`は、この特定の割り当てについての報告を抑制し、その内容のスキャンも抑制する。決して解放されない、本当に意図的な一度きりの割り当て（例: カーネルの生存期間中ずっと存在するブート初期のデータ構造）に使う。

### kmemleak_no_scan {#kmemleaknoscan}

```c
kmemleak_no_scan(ptr);
```

`kmemleak_no_scan(ptr)`は、kmemleakがこの割り当ての内容を、他のオブジェクトへのポインタを求めてスキャンするのを防ぐ。`ptr`自体のリーク報告は抑制しない。kmemleakが`ptr`の内容をルートとして使うのを止めるだけである。オブジェクトがポインタのように見えるがポインタではない値（例: ハッシュ値、チェックサム、生のデータバッファ）を含んでいる場合に使う。

### kmemleak_erase {#kmemleak_erase}

```c
kmemleak_erase(&global_ptr);
```

`kmemleak_erase(ptr)`は、スキャン時の検討対象からポインタを明示的に取り除く。割り当てへのポインタはクリアされたが、その割り当ては後で別の仕組みによって解放される場合に使う。これにより、kmemleakがスキャン中に今やnullとなったポインタをたどるのを防ぐ。

### kmemleak_alloc / kmemleak_free（手動登録） {#kmemleakalloc-kmemleakfree-manual-registration}

`kmalloc`/`vmalloc`ではなく独自のアロケータで管理されるメモリについては、オブジェクトを手動で登録・登録解除できる:

```c
/* 外部で管理される割り当てを登録する */
kmemleak_alloc(ptr, size, min_count, gfp);

/* 解放時に登録を解除する */
kmemleak_free(ptr);
```

`min_count`パラメータは、kmemleakがオブジェクトを到達可能とみなすまでに必要なポインタ参照の数を制御する。`min_count=0`を設定すると、参照の有無にかかわらずkmemleakはこのオブジェクトを決して報告しない（`kmemleak_ignore`と同等）。`min_count=2`を設定すると、少なくとも2つのポインタ参照が必要になる。

### アノテーション関数のまとめ {#summary-of-annotation-functions}

| 関数 | 効果 |
|----------|--------|
| `kmemleak_not_leak(ptr)` | このオブジェクトをリークとして報告しない |
| `kmemleak_ignore(ptr)` | 報告しない。内容もスキャンしない |
| `kmemleak_no_scan(ptr)` | 内容をスキャンしない（リークの追跡は続ける） |
| `kmemleak_erase(addrp)` | ポインタをスキャンの検討対象から取り除く |
| `kmemleak_alloc(ptr, size, min_count, gfp)` | オブジェクトを手動で登録する |
| `kmemleak_free(ptr)` | オブジェクトの登録を手動で解除する |

## 制限事項 {#limitations}

### 誤検出（誤って報告されるリーク） {#false-positives-incorrectly-reported-leaks}

Kmemleakは、スキャンで割り当てを指すポインタが見つからなければ、その割り当てをリークとして報告する。そのため、ポインタが存在していても隠れている場合に誤検出が生じる:

- **エンコードされたポインタ**: XOR連結リスト（`list_add_rcu`の変種）、オフセットベースの参照、意図的に難読化されたポインタは、有効な参照として認識されない。
- **ハードウェアに格納されたポインタ**: MMIOレジスタ、DMAディスクリプタリング、IOMMUテーブルに書き込まれた物理アドレスは、スキャン対象のメモリには含まれない。
- **ユーザー空間のメモリに格納されたポインタ**: カーネルのポインタがユーザー空間のバッファにコピーされた場合（珍しく、通常はそれ自体がバグである）、kmemleakはユーザー空間をスキャンしない。
- **2ワードにまたがって分割されたポインタ**: 非アラインアクセスが許されるアーキテクチャでは、ワード境界をまたいで分割されたポインタは認識されないことがある。

`kmemleak_not_leak()`と`kmemleak_no_scan()`のアノテーションは、まさに問題のないことがわかっているコードでこうした誤検出を黙らせるために存在する。

### 見逃し（検出されないリーク） {#false-negatives-missed-leaks}

Kmemleakは次の場合に本物のリークを検出できないことがある:

- ポインタがたまたまスキャン対象のメモリのどこかに存在し（以前の割り当てから残った、クリアされていない古い値）、オブジェクトが到達可能に見えてしまう場合。
- スキャンが実行される前に割り当てが解放された場合（kmemleakは解放されたオブジェクトを正しく追跡対象から外す）。
- 割り当てがkmemleakの追跡する最小サイズを下回っている場合。

### 性能上のオーバーヘッド {#performance-overhead}

Kmemleakのオーバーヘッドには主に3つの発生源がある:

1. **割り当てごとのコスト**: すべての`kmalloc()`と`kfree()`がkmemleakのコールバックを呼び出し、内部のオブジェクトツリー（割り当てアドレスをキーとする赤黒木）を更新する。これはすべての割り当てにレイテンシを追加する。
2. **メタデータのためのメモリ**: 追跡される各`struct kmemleak_object`が、生存している割り当ての数に比例したメモリを消費する。
3. **スキャンのコスト**: 定期的なスキャンは、データセグメント、CPUごとの領域、すべてのスレッドのカーネルスタックを走査する。スレッドの多いシステムや大きなデータセグメントを持つシステムでは、1回のスキャンに数百ミリ秒かかることがある。

スキャンは一時的にマイグレーションを無効にし、すべてのスレッドのスタックを読むため、スケジューリングのジッタを引き起こすことがある。このため、kmemleakはレイテンシに敏感なワークロードには適さない。

### Kmemleakが検出できないもの {#what-kmemleak-cannot-detect}

- ユーザー空間プロセスのリーク（それにはValgrindやAddressSanitizerを使う）
- メモリマップドI/O領域のリーク
- 技術的には到達可能だが、機能的には決して使われないメモリ（論理的なリーク）
- 参照カウントのバグ（参照カウントが決して0にならず、オブジェクトを無期限に生かし続けるもの） — これらは到達可能なので報告されない

解放されないオブジェクトの蓄積によって引き起こされるOOM状態については、[OOMのデバッグ](oom-debugging.md)を参照。

## 歴史 {#history}

### v2.6.28: 導入（2008年） {#v2628-introduction-2008}

KmemleakはCatalin Marinas（Arm Ltd）によってカーネル2.6.28で導入された。最初の投稿は、基盤とdebugfsインターフェースを追加したコミットに記述されている。その設計は、Boehm-Demers-Weiserの保守的ガベージコレクタのアイデアを、カーネルの平坦な仮想アドレス空間に適用したものである。

**LWNの記事**: [Finding kernel memory leaks](https://lwn.net/Articles/187979/)（2006年）は、アップストリームへの投稿に先立つ初期の設計提案について述べている。

### v2.6.30〜v2.6.32: 安定化 {#v2630v2632-stabilization}

導入後間もなく、Catalin Marinasや他の貢献者が誤検出率とスキャンの対象範囲に取り組み、CPUごとの変数のサポートを追加し、灰色塗りのパスを改良した。

### v3.2: 初期ログバッファ {#v32-early-log-buffer}

初期ログバッファは、kmemleakのデータ構造が完全に初期化される前（ブートのごく初期）に発生する割り当てを捕捉するために追加された。これにより、kmemleakの準備が整った時点でブート時のリークを報告できるようになった。

### v4.x: 安定性と統合の改善 {#v4x-stability-and-integration-improvements}

v4.xシリーズを通じた段階的な改善により、他のデバッグ基盤（フォールトインジェクション、KASAN）との相互作用への対処、誤検出率の低減、デフォルトのスキャン間隔の挙動の改善が行われた。

### v5.x: SLUBとの統合の改善 {#v5x-slub-integration-improvements}

SLUBが主流のslabアロケータになるにつれて（[slabアロケータ](slab.md)を参照）、[`mm/slub.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/slub.c)内のkmemleakのフックは、SLUBの内部的なオブジェクトのライフサイクルを正しく追跡するよう改良され、SLUB自身の内部割り当てによる誤検出が減った。

## 主要なソースファイル {#key-source-files}

| ファイル | 説明 |
|------|-------------|
| [`mm/kmemleak.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kmemleak.c) | kmemleakの実装全体: オブジェクトの追跡、スキャンロジック、debugfsインターフェース |
| [`include/linux/kmemleak.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/kmemleak.h) | 公開API: `kmemleak_alloc`、`kmemleak_free`、アノテーション関数 |
| [`Documentation/dev-tools/kmemleak.rst`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/dev-tools/kmemleak.rst) | カーネルの公式ドキュメント |
| [`mm/slab_common.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/slab_common.c) | slabの共通パスから`kmemleak_alloc`と`kmemleak_free`を呼び出す |
| [`mm/slub.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/slub.c) | SLUBアロケータ。割り当て/解放時にkmemleakのフックを呼び出す |
| [`mm/vmalloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmalloc.c) | `kmemleak_vmalloc` / `kmemleak_vfree`を呼び出す |
| [`lib/Kconfig.debug`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/Kconfig.debug) | `CONFIG_DEBUG_KMEMLEAK`と関連するKconfigオプション |

## 参考文献 {#further-reading}

### カーネルドキュメント {#kernel-documentation}

- [Kmemleak official documentation](https://docs.kernel.org/dev-tools/kmemleak.html) — 設定、使い方、スキャンアルゴリズム

### LWNの記事 {#lwn-articles}

- [Finding kernel memory leaks](https://lwn.net/Articles/187979/) — 動機とアルゴリズムを説明した初期の設計提案

### 関連項目 {#related}

- [KASAN](kasan.md) — use-after-freeと範囲外アクセスを検出する。kmemleakと相補的
- [KFENCE](kfence.md) — 本番環境でも安全なサンプリング型の検出器。リークではなくアクセスのバグを検出する
- [OOMのデバッグ](oom-debugging.md) — 蓄積したリークが枯渇につながる場合
- [slabアロケータ](slab.md) — kmemleakはslabの割り当てパスにフックする
