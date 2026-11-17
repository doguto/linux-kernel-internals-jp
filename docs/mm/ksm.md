# Kernel Same-page Merging（KSM） {#kernel-same-page-merging-ksm}

> 同一内容のメモリページの重複排除

## KSMとは何か {#what-is-ksm}

KSMは、メモリをスキャンして内容が同一のページを探し、それらを単一のコピーオンライト（COW）ページに統合する。複数のプロセスがメモリ上に同じデータを持っている場合、KSMはコピーを1つだけ保持することでRAMを節約する。

```
Before KSM:
Process A          Process B          Process C
┌─────────┐       ┌─────────┐        ┌─────────┐
│ Page X  │       │ Page X  │        │ Page X  │
│ (same   │       │ (same   │        │ (same   │
│ content)│       │ content)│        │ content)│
└─────────┘       └─────────┘        └─────────┘
   3 pages = 12KB

After KSM:
Process A          Process B          Process C
    │                  │                  │
    └──────────────────┼──────────────────┘
                       │
                       ▼
                 ┌─────────┐
                 │ Page X  │  (shared, COW)
                 │         │
                 └─────────┘
   1 page = 4KB (saved 8KB)
```

## なぜKSMか {#why-ksm}

### 仮想化のユースケース {#the-virtualization-use-case}

KSMは仮想マシンのために設計された。複数のVMは、同一のオペレーティングシステムを動かしていることが多い。

| シナリオ | VM数 | 共有できる可能性があるもの |
|----------|-----|-------------------|
| 同一のLinux VMが10台 | 10 | カーネル、ライブラリ、共通データ |
| 開発クラスタ | 50 | ベースOS、ツールチェーン |
| VDI（仮想デスクトップ） | 100+ | Windows、Office、ブラウザ |

KSMなし: 10 VM × 2GB = 20GBのRAM
KSMあり: 共有ページにより約8GBに削減（ワークロードによって異なる）

### VM以外 {#beyond-vms}

KSMは次の場合にも役立つ。

- ベースイメージを共有するコンテナ
- 同じアプリケーションの複数インスタンス
- forkを多用するワークロード

## KSMの仕組み {#how-ksm-works}

### ksmdデーモン {#the-ksmd-daemon}

カーネルスレッド（`ksmd`）が継続的にメモリをスキャンする。

```
ksmd loop:
    │
    ├── Scan pages marked for merging
    │
    ├── Hash page content
    │
    ├── Look for matching hash in stable/unstable trees
    │
    ├── If match: verify byte-by-byte, merge pages
    │
    └── Sleep, repeat
```

### 2本の木の構造 {#two-tree-structure}

KSMは2本の赤黒木を使う。

**stable tree**: 統合済み（共有）のページを保持する

- すでに重複排除されたページ
- 一致を探す際に最初に検索される

**unstable tree**: 候補ページを保持する

- 一度見られて、一致を待っているページ
- スキャンサイクルごとに再構築される

```
New page arrives
      │
      v
Search stable tree ──── Match? ──── Yes ──► Merge with stable page
      │                                           │
      No                                          │
      │                                           │
      v                                           │
Search unstable tree ── Match? ──── Yes ──► Merge both, add to stable
      │                                           │
      No                                          │
      │                                           │
      v                                           │
Add to unstable tree                              │
      │                                           │
      └───────────────────────────────────────────┘
```

### コピーオンライト {#copy-on-write}

統合されたページはCOWとしてマークされる。いずれかのプロセスがそのページに書き込むと、次のようになる。

1. ページフォールトが発生する
2. カーネルが書き込み側のためにページをコピーする
3. 書き込み側はプライベートなコピーを得る
4. 他のプロセスは共有ページを使い続ける

## KSMの有効化 {#enabling-ksm}

### アプリケーション向け {#for-applications}

アプリケーションは、KSMのスキャン対象とするメモリを明示的にマークしなければならない。

```c
#include <sys/mman.h>

void *ptr = mmap(NULL, size, PROT_READ|PROT_WRITE,
                 MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);

/* この領域のKSMスキャンを有効にする */
madvise(ptr, size, MADV_MERGEABLE);

/* KSMスキャンを無効にする */
madvise(ptr, size, MADV_UNMERGEABLE);
```

### QEMU/KVM向け {#for-qemukvm}

QEMUはデフォルトでゲストメモリに対してKSMを有効にする（`-machine mem-merge=off`で無効にできる）。

### システム全体の制御 {#system-wide-control}

```bash
# ksmdの有効化/無効化
echo 1 > /sys/kernel/mm/ksm/run   # スキャン開始
echo 0 > /sys/kernel/mm/ksm/run   # スキャン停止
echo 2 > /sys/kernel/mm/ksm/run   # 停止してすべての統合を解除

# 動作中か確認
cat /sys/kernel/mm/ksm/run
```

## 設定 {#configuration}

### スキャンレート {#scan-rate}

デフォルト値は[KSM管理者ガイド](https://docs.kernel.org/admin-guide/mm/ksm.html)に記載されている。

```bash
# サイクルごとにスキャンするページ数
cat /sys/kernel/mm/ksm/pages_to_scan
echo 100 > /sys/kernel/mm/ksm/pages_to_scan  # デフォルト: 100

# サイクル間のスリープ (ミリ秒)
cat /sys/kernel/mm/ksm/sleep_millisecs
echo 20 > /sys/kernel/mm/ksm/sleep_millisecs  # デフォルト: 20
```

`pages_to_scan`を大きくし`sleep_millisecs`を小さくすると、統合は速くなるがCPU使用量が増える。

### 統合の挙動 {#merge-behavior}

```bash
# NUMAノードをまたいでページを統合するか? (0=しない、1=する)
cat /sys/kernel/mm/ksm/merge_across_nodes
# デフォルト: 1 (ノードをまたいで統合する)
# NUMAを意識するワークロードでは0に設定する

# 統合ページの最大値 (0=無制限)
cat /sys/kernel/mm/ksm/max_page_sharing
# 1つのページを共有できるプロセス数を制限する
```

## 監視 {#monitoring}

### KSMの統計 {#ksm-statistics}

```bash
# 現在共有されている (重複排除された) ページ数
cat /sys/kernel/mm/ksm/pages_shared

# 共有ページへの参照の総数
cat /sys/kernel/mm/ksm/pages_sharing

# 節約されたメモリ = (pages_sharing - pages_shared) × 4KB

# unstable treeのサイズ
cat /sys/kernel/mm/ksm/pages_unshared

# 統合できないページ (変化が激しい)
cat /sys/kernel/mm/ksm/pages_volatile

# 完了したフルスキャンの回数
cat /sys/kernel/mm/ksm/full_scans
```

### 節約量の計算 {#calculating-savings}

```bash
#!/bin/bash
shared=$(cat /sys/kernel/mm/ksm/pages_shared)
sharing=$(cat /sys/kernel/mm/ksm/pages_sharing)
saved=$(( (sharing - shared) * 4 ))  # 節約されたKB
echo "KSM saving: ${saved} KB"
```

### プロセスごとのKSM使用状況 {#per-process-ksm-usage}

```bash
# プロセスで統合されたページ数
cat /proc/<pid>/ksm_merging_pages

# このプロセスで節約されたメモリ (バイト、v6.1+)
cat /proc/<pid>/ksm_stat
```

## 変遷 {#evolution}

### 導入（v2.6.32、2009年） {#introduction-v2632-2009}

**コミット**: [f8af4da3b4c1](https://git.kernel.org/linus/f8af4da3b4c1) ("ksm: the mm interface to ksm")

**作者**: Izik Eidus, Andrea Arcangeli, Chris Wright (Red Hat, VMware)

*注: lore.kernel.orgにある2010年以前のLKMLアーカイブはまばらである。設計はコミットメッセージに記されている。*

もともとは、仮想マシンのメモリフットプリントを減らすためにKVM向けに開発された。

### NUMA対応（v3.9、2013年） {#numa-awareness-v39-2013}

**コミット**: [90bd6fd31c80](https://git.kernel.org/linus/90bd6fd31c80) ("ksm: allow trees per NUMA node") | [LKML](https://lore.kernel.org/all/alpine.LNX.2.00.1302111410290.1174@eggly.anvils/)

**作者**: Petr Holasek（投稿はHugh Dickins）

NUMAの局所性を損なうノード間の共有を防ぐため、`merge_across_nodes`オプションが追加された。

### プロセスごとの統計（v6.1、2022年） {#per-process-statistics-v61-2022}

**コミット**: [cb4df4cae4f2](https://git.kernel.org/linus/cb4df4cae4f2) ("ksm: count allocated ksm rmap_items for each process") | [LKML](https://lore.kernel.org/linux-mm/20220822053653.204150-1-xu.xin16@zte.com.cn/)

**作者**: xu xin

プロセスごとのメモリ節約量を追跡するため、`/proc/<pid>/ksm_stat`が追加された。

## トレードオフ {#trade-offs}

### 利点 {#benefits}

| 利点 | 効果 |
|---------|--------|
| メモリの節約 | 大きく異なる：[Metaの多様なワークロード](https://lwn.net/Articles/953141/)では約9%、同一のVMが多数ある場合はより高い（[Owens et al., IEEE ISPA 2011](https://ieeexplore.ieee.org/document/5951913/)を参照） |
| VM集積度の向上 | ホストあたりのVM数が増える |
| オーバーコミット | 物理RAMを超える量を安全に動かせる |

### コスト {#costs}

| コスト | 影響 |
|------|--------|
| CPUオーバーヘッド | ksmdのスキャンがCPUサイクルを使う |
| COWフォールト | 共有ページへの書き込み = ページフォールト + コピー |
| レイテンシ | COWが最初の書き込みにレイテンシを加える |
| セキュリティ | サイドチャネル攻撃（タイミング）が可能 |

### セキュリティ上の考慮事項 {#security-considerations}

KSMは、タイミングサイドチャネルを通じて情報を漏らす可能性がある（[CVE-2021-3714](https://www.ndss-symposium.org/wp-content/uploads/2022-81-paper.pdf)）。

- 攻撃者が推測した内容のページを割り当てる
- すぐに統合されれば、その内容が他の場所に存在する
- タイミング攻撃によってメモリの内容が明らかになる
- [ASLRの回避](https://svs.informatik.uni-hamburg.de/publications/2018/2018-04-10-Lindemann-Memory-Deduplication-Side-Channel.pdf)や、同じホストに同居するVMで動いているアプリケーションの検出に使われうる

[Red HatはKSMを無効にすることを推奨している](https://docs.redhat.com/en/documentation/red_hat_openstack_platform/17.1/html/hardening_red_hat_openstack_platform/assembly_hardening-infrastructure-and-virtualization_security_and_hardening)。対象はマルチテナント環境（パブリッククラウド）である。カーネルのデフォルトではKSMはオフ（`/sys/kernel/mm/ksm/run` = 0）であり、アプリケーションは`MADV_MERGEABLE`で明示的にオプトインしなければならない。

## よくある問題 {#common-issues}

### 高いCPU使用率 {#high-cpu-usage}

ksmdがCPUを使いすぎている。

**症状**: `ksmd`プロセスのCPU使用率が高い

**対策**:

- `pages_to_scan`を減らす
- `sleep_millisecs`を増やす
- 節約量がわずかなら無効にする

### 統合率が低い {#low-merge-rate}

統合されるページが少ない。

**デバッグ**: `pages_shared`と`pages_unshared`を比較する

**原因**:

- データが一意なワークロード
- スキャンレートが遅すぎる
- メモリが`MADV_MERGEABLE`でマークされていない

### NUMAの性能問題 {#numa-performance-issues}

ノード間の統合が性能を損なっている。

**症状**: KSMを有効にした後、メモリレイテンシが高くなる

**対策**:
```bash
echo 0 > /sys/kernel/mm/ksm/merge_across_nodes
```

## KSMと他の重複排除の比較 {#ksm-vs-other-deduplication}

| 方式 | 対象範囲 | オーバーヘッド | 用途 |
|--------|-------|----------|----------|
| KSM | RAM、実行時 | スキャンのためのCPU | VM、コンテナ |
| zswap | スワップ、圧縮 | 圧縮のためのCPU | メモリの拡張 |
| ファイルシステムの重複排除 | ディスク | スキャンのためのI/O | ストレージ効率 |
| COWファイルシステム | ディスク、スナップショット | メタデータのオーバーヘッド | バックアップ、クローン |

## リファレンス {#references}

### 主要なコード {#key-code}

| ファイル | 説明 |
|------|-------------|
| [`mm/ksm.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/ksm.c) | KSMの実装 |

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/admin-guide/mm/ksm.rst`](https://docs.kernel.org/admin-guide/mm/ksm.html)

### メーリングリストでの議論 {#mailing-list-discussions}

- [最初のKSMパッチシリーズ](https://lore.kernel.org/lkml/1247851850-4298-8-git-send-email-ieidus@redhat.com/) - 2009年7月のIzik Eidusによる投稿
- [KSMのNUMAツリーとページマイグレーション](https://lore.kernel.org/all/alpine.LNX.2.00.1302111410290.1174@eggly.anvils/T/) - 2013年のHugh DickinsによるNUMA対応パッチ

### 関連 {#related}

- [reclaim](reclaim.md) - メモリ逼迫との相互作用
- [numa](numa.md) - ノード間の統合に関する考慮事項
- [thp](thp.md) - KSMはhuge pageを統合しない
- [cow](cow.md) - KSMで統合されたページに書き込まれたときのCOWの挙動

## 参考文献 {#further-reading}

- `Documentation/admin-guide/mm/ksm.rst` — `ksmd`の設定、sysfsのノブ、`MADV_MERGEABLE` / `MADV_UNMERGEABLE`、stable/unstable treeのモデル、監視用カウンタを扱うカーネルの管理者ガイド。[docs.kernel.org](https://docs.kernel.org/admin-guide/mm/ksm.html)で閲覧できる
- [mm/ksm.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/ksm.c) — KSMの実装全体：2本の赤黒木、`ksmd`のスキャンループ、ページのハッシュ化、COWページの置き換え、NUMA対応の木の管理
- [LWN: KSM: kernel samepage merging](https://lwn.net/Articles/330589/) — Izik EidusとAndrea Arcangeliによる最初のKSMの設計を紹介した2009年の記事。KVMのVM集積度というユースケースに焦点を当てている
- [LWN: KSM comes to anonymous memory](https://lwn.net/Articles/953141/) — MetaによるKSMの本番導入と、多様なサーバーワークロード全体での約9%のメモリ節約を扱った記事
- コミット[f8af4da3b4c1](https://git.kernel.org/linus/f8af4da3b4c1) — v2.6.32でのKSMの導入。コミットメッセージにstable/unstable treeの設計と`MADV_MERGEABLE`によるオプトインのインターフェースが説明されている
- コミット[90bd6fd31c80](https://git.kernel.org/linus/90bd6fd31c80) — v3.9での`merge_across_nodes=0`のサポート。メモリの局所性を低下させるNUMAノード間の統合を防ぐ
- コミット[cb4df4cae4f2](https://git.kernel.org/linus/cb4df4cae4f2) — v6.1での`/proc/<pid>/ksm_stat`によるプロセスごとのKSM統計
