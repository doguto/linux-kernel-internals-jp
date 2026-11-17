# KFENCE（Kernel Electric Fence） {#kfence-kernel-electric-fence}

> 本番カーネル向けの、低オーバーヘッドなサンプリングベースのヒープバグ検出

## KFENCEとは何か {#what-is-kfence}

KFENCEは、カーネル内のヒープの範囲外アクセス、use-after-freeバグ、不正な解放のエラーを捕捉する軽量なメモリエラー検出器である。すべてのメモリアクセスを計装する本格的なサニタイザとは異なり、KFENCEは**サンプリングベース**のアプローチを使う。どの時点でも割り当てのごく一部しか保護せず、数時間から数日の稼働時間をかけてバグを捕捉することを統計的な確率に頼っている。

鍵となる着想は、本番環境で頻繁に起きるバグは、十分な時間があればいずれKFENCEで保護された割り当てに当たるということである。保護される割り当ての割合は、サンプリング間隔に対するシステムの割り当て頻度に依存する。これにより、KASANではコストが高すぎて使えないような常時稼働の展開でも、KFENCEは実用的になる。

## なぜKFENCEが存在するのか {#why-kfence-exists}

### テストと本番の隔たり {#the-gap-between-testing-and-production}

KFENCE以前、カーネル開発者はジレンマに直面していた。

| ツール | オーバーヘッド | 実行される場所 | バグの検出範囲 |
|------|----------|---------------|--------------|
| KASAN（generic） | 高（[KASANのドキュメント](https://docs.kernel.org/dev-tools/kasan.html)参照） | CI、テスト | すべての割り当て |
| KASAN（SW/HWタグ） | 中〜低 | テスト、一部のハードウェア | すべての割り当て |
| なし | 0% | 本番 | なし |

多くのヒープバグは、実際の本番ワークロードの下でしか表面化しない。特定のタイミング、メモリ圧迫、アクセスパターンなど、テスト環境では再現が難しいものである。KFENCEはこの隔たりを埋める。ほぼゼロのオーバーヘッドで本番環境で動作し、そうでなければデータ破壊やクラッシュを起こすまで検出されないままだったはずのバグを捕捉する。

### 現実の動機 {#real-world-motivation}

[KFENCEの設計ドキュメント](https://docs.kernel.org/dev-tools/kfence.html)に記されているとおり、動機は次のとおりだった。

1. 多くのバグは、実際のトラフィックの下にある本番マシンでしか現れなかった
2. 完全なサニタイザのオーバーヘッドは、本番のフリートでは受け入れられなかった
3. 低いサンプリング率でも、高頻度のバグはすぐに捕捉できる

## KASANとの違い {#how-it-differs-from-kasan}

```
KASAN (always-on):
  Every allocation → instrumented → checked on every access
  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
  │shadow│ │shadow│ │shadow│ │shadow│ │shadow│  ← shadow memory for ALL
  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘

KFENCE (sampling):
  1 in ~N allocations → placed in guarded pool → hardware trap on error
  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
  │normal│ │normal│ │KFENCE│ │normal│ │normal│  ← only some protected
  └──────┘ └──────┘ │guard │ └──────┘ └──────┘
                     │pages │
                     └──────┘
```

| 観点 | KASAN | KFENCE |
|--------|-------|--------|
| 検出 | 決定的（すべてのアクセス） | 確率的（サンプリング） |
| オーバーヘッド | CPU 2〜3倍、メモリ1.5〜3倍 | CPU 1%未満、メモリは固定で約2MB |
| 展開先 | テスト、CI | 本番、常時稼働 |
| 仕組み | シャドウメモリ + コンパイラによる計装 | ガードページ + ハードウェアのページフォールト |
| 検出範囲 | すべての割り当て | 入れ替わる小さな部分集合 |
| 設定 | コンパイル時のみ | sysfs経由で実行時に調整可能 |

## 仕組み {#how-it-works}

### オブジェクトプール {#the-object-pool}

KFENCEはブート時に固定サイズのメモリプールを事前に割り当てる。デフォルトでは、このプールは255個のオブジェクト分の領域を持つ。プールは、すべてのオブジェクトスロットの間に**ガードページ**を挟んで配置される。

```
KFENCE pool layout:

  ┌───────┐ ┌────────┐ ┌───────┐ ┌────────┐ ┌───────┐ ┌────────┐ ┌───────┐
  │ guard │ │ object │ │ guard │ │ object │ │ guard │ │ object │ │ guard │
  │ page  │ │ page   │ │ page  │ │ page   │ │ page  │ │ page   │ │ page  │
  │ (no   │ │        │ │ (no   │ │        │ │ (no   │ │        │ │ (no   │
  │access)│ │        │ │access)│ │        │ │access)│ │        │ │access)│
  └───────┘ └────────┘ └───────┘ └────────┘ └───────┘ └────────┘ └───────┘
  no-access   mapped     no-access   mapped     no-access   mapped     no-access
```

各オブジェクトスロットは1ページ丸ごと（x86では4KB）だが、実際のオブジェクトはページの**左端**か**右端**のどちらかに配置される。

- **右寄せ**: オブジェクトをページの末尾に配置するので、右側への範囲外アクセスはすぐにガードページに当たる
- **左寄せ**: オブジェクトをページの先頭に配置するので、左側への範囲外アクセスはガードページに当たる

KFENCEは配置を交互に切り替えて、左右両方のオーバーフローを捕捉する。

### サンプリングの仕組み {#the-sampling-mechanism}

KFENCEは、すべての`kmalloc()`や`kmem_cache_alloc()`の呼び出しに介入するわけではない。代わりに、**タイマーベースのサンプリング間隔**を使う。

1. 静的分岐（通常は分岐しない）がタイマーで切り替えられる
2. 有効に切り替えられると、**次の**割り当てがKFENCEプールにリダイレクトされる
3. 割り当てを1つ捕捉すると、静的分岐は無効に戻される
4. 設定された間隔（デフォルト: 100ms）の後にタイマーが再び発火する

```
Time →
  ─────────────────────────────────────────────────────────────
  │         │ sample │         │ sample │         │ sample │
  │  normal │  one   │  normal │  one   │  normal │  one   │
  │  allocs │ alloc  │  allocs │ alloc  │  allocs │ alloc  │
  ─────────────────────────────────────────────────────────────
             ↑ 100ms  ↑         ↑ 100ms  ↑
            timer    captured   timer    captured
```

この方式により、通常の動作中のオーバーヘッドはほぼゼロに抑えられる。静的分岐（`static_branch_unlikely`）は高速パスではNOPにコンパイルされるので、サンプリングされない99.99%超の割り当てにとってコストはゼロである。

### バグの検出 {#detecting-bugs}

KFENCEで保護された割り当てが誤用されると、ハードウェアのMMUが検出を行う。

**範囲外アクセス**: アクセスがアクセス不可のガードページ（[`kfence_protect_page()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/kfence.h)によってpresentビットがクリアされたページ）に当たり、ページフォールトが発生する。KFENCEのフォールトハンドラはフォールトしたアドレスを特定し、最も近いKFENCEオブジェクトを見つけて、オーバーフローまたはアンダーフローを報告する。

**use-after-free**: オブジェクトが解放されると、そのページはアクセス不可になる（保護が`PROT_NONE`に設定される）。その後のアクセスはすべてページフォールトを引き起こす。KFENCEは解放されたオブジェクトについてのメタデータ（解放時のスタックトレースを含む）を保持しているので、何がいつ解放されたかを報告できる。

**不正な解放**: KFENCEのアドレスに対して`kfree()`が呼ばれると、KFENCEはポインタを検証する。二重解放や不正なポインタは捕捉されて報告される。

## KFENCEレポートの例 {#what-a-kfence-report-looks-like}

KFENCEがバグを検出すると、カーネルログに詳細なレポートを出力する。以下は範囲外書き込みの注釈付きの例である。

```
==================================================================
BUG: KFENCE: out-of-bounds write in driver_probe+0x84/0x120      ← Bug type and location

Out-of-bounds write at 0xffff8881045ab000 (1 byte(s) to the right ← Faulting address and
 of kfence-#72):                                                     which KFENCE slot
 driver_probe+0x84/0x120                                          ← Stack trace of the
 really_probe+0x170/0x3e0                                            bad access
 __driver_probe_device+0x78/0x160
 driver_probe_device+0x1e/0x90
 bus_for_each_drv+0x84/0xe0

kfence-#72: 0xffff8881045aafe0-0xffff8881045aafff, size=32,       ← Object details:
 cache=kmalloc-32                                                    address range, size,
                                                                     slab cache name

allocated by task 1234 on cpu 3 at 42.010s:                       ← Allocation stack trace
 kmalloc+0x2e/0x40                                                   with task, CPU, and
 driver_init+0x30/0x80                                               timestamp
 do_init_module+0x50/0x200
 load_module+0x2a4c/0x2e70
 __do_sys_finit_module+0xac/0x120
 do_syscall_64+0x5c/0xf0
==================================================================
```

use-after-freeバグの場合、レポートにはオブジェクトがどこで解放されたかを示す**解放時のスタックトレース**も含まれる。

```
==================================================================
BUG: KFENCE: use-after-free read in corrupt_data+0x28/0x40

Use-after-free read at 0xffff8881045ab000 (4 byte(s)):
 corrupt_data+0x28/0x40
 test_function+0x60/0x90

kfence-#51: 0xffff8881045a6fe0-0xffff8881045a6fff, size=32,
 cache=kmalloc-32

allocated by task 789 on cpu 1 at 18.500s:
 kmalloc+0x2e/0x40
 setup_object+0x24/0x60

freed by task 789 on cpu 1 at 19.200s:                            ← Free stack trace
 kfree+0x4e/0x60                                                     helps you find the
 cleanup_object+0x1c/0x40                                            premature free
 work_handler+0x80/0xa0
==================================================================
```

これらのレポートは根本原因を診断するのに十分な情報を提供する。どのオブジェクトが関係していたか、その大きさ、どのslabキャッシュから来たか、そして割り当て、解放、問題のアクセスそれぞれの完全なコールスタックを正確に確認できる。

## 設定 {#configuration}

### ビルド時オプション {#build-time-options}

カーネル設定でKFENCEを有効化する。

```
CONFIG_KFENCE=y
```

必須のオプションはこれだけである。KFENCEはオーバーヘッドが低いので、ディストリビューションのカーネルにも適している（[KFENCEのドキュメント](https://docs.kernel.org/dev-tools/kfence.html)を参照）。

任意のビルド時設定:

| 設定オプション | デフォルト | 説明 |
|---------------|---------|-------------|
| `CONFIG_KFENCE` | n | KFENCEを有効化する |
| `CONFIG_KFENCE_SAMPLE_INTERVAL` | 100 | デフォルトのサンプリング間隔（ミリ秒） |
| `CONFIG_KFENCE_NUM_OBJECTS` | 255 | KFENCEプール内のオブジェクト数 |
| `CONFIG_KFENCE_STRESS_TEST_FAULTS` | 0 | KFENCE自体をテストするためにフォールトを注入する |

### 実行時のチューニング {#runtime-tuning}

サンプリング間隔は、再起動せずに実行時に調整できる。

```bash
# 現在のサンプリング間隔（ミリ秒）を確認する
cat /sys/module/kfence/parameters/sample_interval

# 間隔を長くする（オーバーヘッドは減り、捕捉も減る）
echo 500 > /sys/module/kfence/parameters/sample_interval

# 間隔を短くする（オーバーヘッドは増え、捕捉も増える）
echo 50 > /sys/module/kfence/parameters/sample_interval

# 実行時にKFENCEを無効化する
echo 0 > /sys/module/kfence/parameters/sample_interval
```

### ブートパラメータ {#boot-parameters}

間隔はブート時にも設定できる。

```
kfence.sample_interval=200
```

## 性能特性 {#performance-characteristics}

### なぜ本番環境で安全なのか {#why-it-is-safe-for-production}

KFENCEは、最初から本番での利用を前提に設計された。

**高速パスのオーバーヘッドがほぼゼロ**: サンプリングのチェックは、NOP命令にコンパイルされる静的分岐を使う。x86では、KFENCEがサンプリングしていないとき（時間の99.99%超）、割り当ての高速パスに追加される命令はゼロである。

**固定のメモリコスト**: KFENCEプールはブート時に割り当てられ、サイズは固定である。デフォルトの255オブジェクトでは、オブジェクトスロットとガードページを交互に並べて、プールは`(255 + 1) * 2 = 512`ページ（約2MB）になる。これは増えない。

**コンパイラによる計装が不要**: KASANとは異なり、KFENCEはコンパイラのサポートもコードの変更も必要としない。どのコンパイラでも動作し、コードサイズも増やさない。

**シャドウメモリが不要**: KASANはシステムの総メモリに比例したシャドウメモリ（典型的にはRAMの1/8）を必要とする。KFENCEはシステムのメモリサイズに関係なく、固定のプールだけを使う。

| 指標 | KASAN（generic） | KFENCE |
|--------|-----------------|--------|
| CPUオーバーヘッド | 高（[KASANのドキュメント](https://docs.kernel.org/dev-tools/kasan.html)参照） | ほぼゼロ（[KFENCEのドキュメント](https://docs.kernel.org/dev-tools/kfence.html)参照） |
| メモリオーバーヘッド | RAMの約12.5% | 固定で約2MB |
| コードサイズの増加 | 大きい | なし |
| 本番環境での安全性 | 安全でない | 安全 |

### トレードオフ {#the-trade-off}

低オーバーヘッドの代償は**確率的な検出**である。KFENCEはすべてのバグをすぐに捕捉するわけではない。バグが検出されるには、KFENCEで保護された割り当てに当たる必要がある。検出までの時間はバグの頻度とサンプリング間隔に依存する。高頻度のバグはより早く捕捉され、まれなバグはより長い時間を要する。

## 試してみる {#try-it-yourself}

### KFENCEが有効か確認する {#check-if-kfence-is-enabled}

```bash
# カーネル設定を確認する
zcat /proc/config.gz 2>/dev/null | grep KFENCE || grep KFENCE /boot/config-$(uname -r)

# KFENCEが動作しているか確認する（間隔が非ゼロなら動作中）
cat /sys/module/kfence/parameters/sample_interval
```

### KFENCEの動作を監視する {#monitor-kfence-activity}

```bash
# KFENCEの統計はdebugfsで参照できる
cat /sys/kernel/debug/kfence/stats

# 出力例:
#   enabled: 1
#   objects allocated: 189
#   objects freed: 42
#   zombie allocations: 0
#   skip allocs (pool full): 312
#   bugs found: 0
```

### テスト用のバグを発生させる（VM内で） {#trigger-a-test-bug-in-a-vm}

!!! warning
    これはテスト用のVMまたはQEMUインスタンスでのみ行い、本番システムでは決して行わないこと。

カーネルにはKFENCEのテストモジュールが含まれている。

```bash
# CONFIG_KFENCE_KUNIT_TEST=y でビルドする
# KFENCEのKUnitテストを実行する
modprobe kfence_test

# またはKUnitフレームワーク経由で実行する
./tools/testing/kunit/kunit.py run kfence
```

### ワークロードに合わせたチューニング {#tuning-for-your-workload}

```bash
# 控えめに始める（デフォルトは100ms）
cat /sys/module/kfence/parameters/sample_interval

# より早く検出したく、最小限のオーバーヘッドを許容できる場合:
echo 20 > /sys/module/kfence/parameters/sample_interval

# 検出されたバグがないか監視する
dmesg | grep -i kfence
```

## 歴史 {#history}

### v5.12: 導入（2021年） {#v512-introduction-2021}

**コミット**:

- [0ce20dd84089](https://git.kernel.org/linus/0ce20dd84089)（"mm: add Kernel Electric Fence infrastructure"）
- [bc8fbc5f305a](https://git.kernel.org/linus/bc8fbc5f305a)（"mm, kfence: insert KFENCE hooks for SLAB"）
- [d3fb45f370d9](https://git.kernel.org/linus/d3fb45f370d9)（"mm, kfence: insert KFENCE hooks for SLUB"）

**作者**: Marco ElverとAlexander Potapenko（Google）

**カーネル**: v5.12

KFENCEはGoogleで開発された。[v1 RFC](https://lore.kernel.org/lkml/20200907134055.2878499-1-elver@google.com/)（2020年9月）には、本番品質のサンプリングベース検出器をアップストリームに取り込むための、元々の設計の根拠と動機が記されている。

パッチシリーズはLKMLで広範なレビューを経た: [KFENCE v7 patch series](https://lore.kernel.org/lkml/20201103175841.3495947-1-elver@google.com/)（2020年11月、2021年初頭にv5.12向けにマージ）。

**LWNの記事**: [KFENCE: A low-overhead sampling-based memory safety error detector](https://lwn.net/Articles/830877/)は、設計の根拠をわかりやすく概説している。

### v5.17: レポートと検出範囲の改善 {#v517-improved-reporting-and-coverage}

**コミット**: [58f116052668](https://git.kernel.org/linus/58f116052668)（"kfence: track memcache for allocations not covered by a slab cache"）

### v5.18: 遅延可能タイマーのサポート {#v518-deferrable-timer-support}

**コミット**: [737b6a10ac19](https://git.kernel.org/linus/737b6a10ac19)（"kfence: allow use of a deferrable timer"）

**作者**: Marco Elver（Google）

消費電力が重要なシステムでのオーバーヘッドを減らすため、遅延可能（deferrable）タイマーのオプションが追加された。遅延可能タイマーはアイドル状態のCPUを起こさないので、KFENCEはモバイルや組み込みシステムにとってより扱いやすくなった。

## 主要なソースファイル {#key-source-files}

| ファイル | 説明 |
|------|-------------|
| [`mm/kfence/core.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kfence/core.c) | KFENCEの主要な実装: プールの初期化、割り当て、サンプリング |
| [`mm/kfence/report.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kfence/report.c) | バグレポートの生成と整形 |
| [`mm/kfence/kfence_test.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kfence/kfence_test.c) | KUnitテストスイート |
| [`include/linux/kfence.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/kfence.h) | 公開APIとインラインフック |
| [`mm/kfence/kfence.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kfence/kfence.h) | プールのメタデータ構造体を含む内部ヘッダ |

## 参考資料 {#references}

### カーネルドキュメント {#kernel-documentation}

- [KFENCE公式ドキュメント](https://docs.kernel.org/dev-tools/kfence.html) -- 設定と設計に関する包括的なリファレンス

### LWNの記事 {#lwn-articles}

- [KFENCE: A low-overhead sampling-based memory safety error detector](https://lwn.net/Articles/830877/) -- 最初の投稿時の概説記事

### LKMLでの議論 {#lkml-discussions}

- [KFENCE v7 patch series](https://lore.kernel.org/lkml/20201103175841.3495947-1-elver@google.com/) -- マージ前の最終レビューのシリーズ
- [KFENCE v1 RFC](https://lore.kernel.org/lkml/20200907134055.2878499-1-elver@google.com/) -- 設計についての議論を含む最初のRFC

### 関連項目 {#related}

- [slab](slab.md) -- KFENCEはslabアロケータ（SLUB/SLAB）にフックして割り当てに介入する
- [vmalloc](vmalloc.md) -- KFENCEプールは、vmallocのガードページと同様のページ単位の保護を使う
- [oom](oom.md) -- KFENCEのプールはサイズが固定なので、OOMの圧迫には寄与しない

## 参考文献 {#further-reading}

### カーネルドキュメント {#kernel-documentation_1}

- `Documentation/dev-tools/kfence.rst` — 設計の根拠、設定オプション、サンプリング間隔とプールサイズのチューニング指針
- [mm/kfence/](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kfence) — KFENCEの完全な実装: プールの割り当て、ガードページの管理、サンプリングタイマー、レポートの生成

### LWNの記事 {#lwn-articles_1}

- [KFENCE: A low-overhead sampling-based memory safety error detector](https://lwn.net/Articles/830877/) — 設計の根拠と本番環境への展開モデルについてのMarco Elverによる概説（2021年）

### 関連ドキュメント {#related-docs}

- [kasan.md](kasan.md) — KASANはすべての割り当てを計装する。KFENCEは、KASANを実行できない本番ワークロードをカバーすることでKASANを補完する
- [kmemleak.md](kmemleak.md) — メモリリークを検出する。KFENCEと組み合わせることで、アクセスのバグとリークした割り当ての両方をカバーできる
- [fault-injection.md](fault-injection.md) — エラーパスをテストするために意図的に割り当てを失敗させる。長時間のストレステストでKFENCEとよく組み合わせられる
