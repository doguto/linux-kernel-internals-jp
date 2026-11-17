# メモリ管理のバグ（カーネル） {#memory-management-bugs-kernel}

> Linuxカーネルのメモリ管理における注目すべきバグ、脆弱性、エッジケースのカタログ

この索引は、Linuxカーネルの mm/ サブシステムにおけるバグ（セキュリティ脆弱性、データ破損、性能リグレッション、安定性の問題）を扱う。いずれも、教育目的で分析する実際のカーネルバグである。

## クイックリファレンス {#quick-reference}

### 深刻度別 {#by-severity}

| 深刻度 | 件数 | 例 |
|----------|-------|----------|
| **致命的（CVE、エクスプロイト）** | 15+ | Dirty COW、Meltdown、SLUBのエクスプロイト |
| **データ破損** | 5+ | large folioのデータ消失、スワップのABA |
| **デッドロック/ハング** | 5+ | mmap_semのデッドロック、OOMのライブロック |
| **性能** | 5+ | スラッシング、readaheadの罠 |

### 年別 {#by-year}

| 年 | 注目すべきバグ |
|------|--------------|
| 2024 | スワップスロットのABA（CVE-2024-26759） |
| 2023 | StackRot（CVE-2023-3269）、large folioのデータ消失、glibcのヒープオーバーフロー（CVE-2023-6246） |
| 2022 | io_uringのUAF（CVE-2022-29582）、systemdによるcgroup大量殺戮 |
| 2021 | Netfilterのヒープ境界外アクセス（CVE-2021-22555） |
| 2020 | THPのCOW競合（CVE-2020-29368） |
| 2018 | mremapのTLB競合（CVE-2018-18281）、Spectreの亜種 |
| 2017 | Meltdown（CVE-2017-5754）、Spectre（CVE-2017-5753/5715）、Stack Clash |
| 2016 | Dirty COW（CVE-2016-5195） |

---

## バグの詳細ドキュメント {#detailed-bug-documentation}

### サブシステムごとのバグのセクション {#subsystem-bug-sections}

特定のサブシステムにおけるバグの詳細な分析（各トピックのドキュメント内にある）:

| ドキュメント | サブシステム | 主なバグ |
|----------|-----------|----------|
| [SLUBアロケータのバグ](../slab.md#notorious-bugs-and-edge-cases) | mm/slub.c | CVE-2021-22555、CVE-2022-29582、ヒープのエクスプロイト |
| [THPのバグ](../thp.md#notorious-bugs-and-edge-cases) | mm/huge_memory.c | CVE-2020-29368、khugepagedの競合、collapseのバグ |
| [ページテーブルのバグ](../page-tables.md#notorious-bugs-and-edge-cases) | arch/*/mm/ | Meltdown、Spectre、TLBフラッシュの競合 |

### ライフサイクル系ドキュメントのバグのセクション {#lifecycle-document-bug-sections}

ライフサイクル系ドキュメントには、端から端までのウォークスルーの一部として、物語形式でのバグの解説が含まれている。

| ドキュメント | セクション | 主なバグ |
|----------|---------|----------|
| [fork.md](../fork.md#notorious-bugs-and-edge-cases) | COWとforkのバグ | Dirty COW（CVE-2016-5195）、StackRot（CVE-2023-3269）、mremap（CVE-2004-0077） |
| [oom.md](../oom.md#notorious-bugs-and-edge-cases) | OOM killerのバグ | mmap_semのデッドロック、スラッシングによるライブロック、CVE-2012-4398 |
| [life-of-malloc.md](../life-of-malloc.md#notorious-bugs-and-edge-cases) | ヒープと割り当てのバグ | glibc CVE-2023-6246、Stack Clash、mmap_min_addrのバイパス |
| [life-of-page.md](../life-of-page.md#notorious-bugs-and-edge-cases) | ページのライフサイクルのバグ | 参照カウントのオーバーフロー、large folioのデータ消失、LRUの破損 |
| [life-of-read.md](../life-of-read.md#notorious-bugs-and-edge-cases) | ページキャッシュのバグ | filemap_faultの競合、9pの破損、readaheadの問題 |
| [swapping.md](../swapping.md#notorious-bugs-and-edge-cases) | スワップのバグ | スワップスロットのABA（CVE-2024-26759）、zswapの競合 |

---

## バグの分類 {#bug-categories}

### セキュリティ脆弱性（CVE） {#security-vulnerabilities-cves}

CVEが割り当てられたバグ。典型的には権限昇格や情報漏洩に悪用可能なもの。

| CVE | 名称 | サブシステム | 種類 | 詳細 |
|-----|------|-----------|------|---------|
| CVE-2024-26759 | スワップスロットのABA | swap | 競合/破損 | [swapping.md](../swapping.md#case-1-the-swap-slot-aba-problem-cve-2024-26759) |
| CVE-2023-3269 | StackRot | maple tree | UAF | [fork.md](../fork.md#case-2-stackrot-cve-2023-3269) |
| CVE-2023-6246 | glibc syslog | glibcのヒープ | オーバーフロー | [life-of-malloc.md](../life-of-malloc.md#case-1-glibc-heap-overflow-cve-2023-6246) |
| CVE-2022-29582 | io_uringのUAF | io_uring/slub | UAF | [slab.md](../slab.md#case-2-io_uring-use-after-free-cve-2022-29582) |
| CVE-2021-22555 | Netfilterのヒープ境界外アクセス | netfilter/slub | ヒープ境界外アクセス | [slab.md](../slab.md#case-1-netfilter-heap-out-of-bounds-cve-2021-22555) |
| CVE-2020-29368 | THPのCOW競合 | thp | 競合 | [thp.md](../thp.md#case-1-thp-cow-race-cve-2020-29368) |
| CVE-2018-18281 | mremapのTLB競合 | mremap | TLB競合 | [fork.md](../fork.md#case-4-tlb-flush-races-in-mremap-cve-2018-18281) |
| CVE-2017-5754 | Meltdown | CPU/ページテーブル | サイドチャネル | [page-tables.md](../page-tables.md#case-1-meltdown-cve-2017-5754) |
| CVE-2017-5753 | Spectre v1 | CPU | サイドチャネル | [page-tables.md](../page-tables.md#case-2-spectre-cve-2017-5753-cve-2017-5715) |
| CVE-2017-5715 | Spectre v2 | CPU | サイドチャネル | [page-tables.md](../page-tables.md#case-2-spectre-cve-2017-5753-cve-2017-5715) |
| CVE-2016-5195 | Dirty COW | COW | 競合 | [fork.md](../fork.md#case-1-dirty-cow-cve-2016-5195) |
| CVE-2012-4398 | OOMのデッドロック | OOM | デッドロック | [oom.md](../oom.md#case-5-cve-2012-4398-oom-deadlock-denial-of-service) |
| CVE-2009-2695 | mmap_min_addrのバイパス | mmap | ロジック | [life-of-malloc.md](../life-of-malloc.md#case-3-mmap_min_addr-bypass-cve-2009-2695) |
| CVE-2004-0077 | mremapの大惨事 | mremap | ロジック | [fork.md](../fork.md#case-3-the-mremap-disaster-cve-2004-0077) |

### データ破損バグ {#data-corruption-bugs}

セキュリティ上の影響はないが、気づかれないままデータの消失や破損を引き起こすバグ。

| バグ | 年 | サブシステム | 詳細 |
|-----|------|-----------|---------|
| large folioのデータ消失 | 2023 | writeback/folio | [life-of-page.md](../life-of-page.md#case-2-large-folio-data-loss-2023) |
| スワップスロットのABA | 2024 | swap | [swapping.md](../swapping.md#case-1-the-swap-slot-aba-problem-cve-2024-26759) |
| 9pの読み込み破損 | 2025 | 9p/netfs | [life-of-read.md](../life-of-read.md#case-2-9p-read-corruption-2025) |
| zswapの競合 | 継続中 | zswap | [swapping.md](../swapping.md#case-2-zswap-race-conditions) |

### デッドロックとハングのバグ {#deadlock-hang-bugs}

システムのハングや回復不能な状態を引き起こすバグ。

| バグ | 年 | サブシステム | 詳細 |
|-----|------|-----------|---------|
| mmap_semのデッドロック | 2010-2016 | OOM | [oom.md](../oom.md#case-2-the-mmap_sem-deadlock-2010-2016) |
| NFS上のスワップのデッドロック | 過去 | swap/NFS | [swapping.md](../swapping.md#case-3-swap-over-nfs-instability) |

### 性能バグ {#performance-bugs}

深刻な性能低下を引き起こすバグ。

| バグ | 年 | サブシステム | 詳細 |
|-----|------|-----------|---------|
| スラッシングによるライブロック | 2010+ | reclaim | [oom.md](../oom.md#case-3-the-thrashing-livelock-2010-mitigated-2018) |
| readaheadの罠 | 継続中 | readahead | [swapping.md](../swapping.md#case-5-the-swap-readahead-trap) |
| khugepagedのCPU消費 | 継続中 | thp | [thp.md](../thp.md#case-3-khugepaged-cpu-storms) |

---

## エクスプロイト手法 {#exploitation-techniques}

mmのバグを悪用する際によく使われる手法:

### ヒープのエクスプロイト（SLUB） {#heap-exploitation-slub}

| 手法 | 説明 | 使用例 |
|-----------|-------------|---------|
| **ヒープスプレー** | 制御下のオブジェクトでヒープを埋め尽くす | CVE-2021-22555 |
| **クロスキャッシュ攻撃** | 異なるキャッシュをまたいでオブジェクトを悪用する | CVE-2022-29582 |
| **フリーリストの破壊** | SLUBのフリーリストのポインタを上書きする | 多数のヒープオーバーフロー |
| **msg_msgの悪用** | msgsnd() を使ってヒープのレイアウトを制御する | CVE-2021-22555 |

### 競合状態 {#race-conditions}

| 手法 | 説明 | 使用例 |
|-----------|-------------|---------|
| **userfaultfd** | カーネルを狙ったタイミングで一時停止させる | Dirty COWの亜種 |
| **FUSE** | ファイルシステム経由でページフォールトのタイミングを制御する | さまざまなエクスプロイト |
| **CPUピン留め** | エクスプロイトのコードを実行するCPUを制御する | 競合の窓の拡大 |

### サイドチャネル {#side-channels}

| 手法 | 説明 | 使用例 |
|-----------|-------------|---------|
| **Flush+Reload** | キャッシュのタイミングを計測する | Meltdown、Spectre |
| **Prime+Probe** | キャッシュを埋め、追い出しを計測する | Spectreの亜種 |

---

## 堅牢化と緩和策 {#hardening-mitigations}

### カーネルのコンフィグオプション {#kernel-config-options}

| オプション | 防御対象 | 性能への影響 |
|--------|------------------|-------------------|
| `CONFIG_SLAB_FREELIST_RANDOM` | ヒープレイアウトの予測 | 最小限 |
| `CONFIG_SLAB_FREELIST_HARDENED` | フリーリストのポインタの破壊 | 最小限 |
| `CONFIG_INIT_ON_ALLOC_DEFAULT_ON` | 未初期化メモリ経由の情報漏洩 | 低 |
| `CONFIG_INIT_ON_FREE_DEFAULT_ON` | UAFによる情報漏洩 | 低 |
| `CONFIG_PAGE_TABLE_ISOLATION` | Meltdown | 1-5% |
| `CONFIG_RETPOLINE` | Spectre v2 | 状況による |

### 実行時のチューニング可能なパラメータ {#runtime-tunables}

```bash
# userfaultfd を制限する（競合を突くエクスプロイトの攻撃対象を減らす）
echo 0 > /proc/sys/vm/unprivileged_userfaultfd

# カーネルポインタの露出を制限する
echo 2 > /proc/sys/kernel/kptr_restrict

# KFENCE のサンプリングを有効にする
echo 100 > /sys/module/kfence/parameters/sample_interval
```

---

## 外部リソース {#external-resources}

### 研究と解説記事 {#research-write-ups}

- [Google Project Zero](https://googleprojectzero.blogspot.com/) - カーネル脆弱性の研究
- [xairy/linux-kernel-exploitation](https://github.com/xairy/linux-kernel-exploitation) - 網羅的なリンク集
- [how2heap](https://github.com/shellphish/how2heap) - ヒープのエクスプロイト手法

### 公開リスト {#disclosure-lists}

- [oss-security](https://www.openwall.com/lists/oss-security/) - 脆弱性の公開情報
- [LKML security](https://lore.kernel.org/linux-security-module/) - カーネルのセキュリティに関する議論

---

## 貢献 {#contributing}

ここに載っていないバグを見つけたら、ぜひ貢献してほしい。

1. `bugs/` 内の適切なサブシステムのファイルに追加する
2. CVE（割り当てられていれば）、修正コミット、根本原因の分析を含める
3. この索引からリンクする
