# 翻訳ガイド（kernel-internals.org 日本語版）

原文: https://kernel-internals.org/ （laveeshb/linux-kernel-internals, CC BY-SA 4.0）

## 基本方針

- 文体は **である調**（常体）。「〜です／〜ます」は使わない。
- 直訳より、日本語として自然に読めることを優先する。ただし技術的な意味は絶対に変えない。情報の追加・省略をしない。
- 英語の強調（`*foo*` / `**foo**`）は日本語側でも同じ箇所に残す。
- 全角英数字は使わない。英単語・数字・記号は半角。英単語と日本語の間にスペースは入れない（例: 「CPU を」ではなく「CPUを」でもよいが、ファイル内で統一する。迷ったら **スペースなし**）。
- 句読点は「、」「。」。括弧は全角「（）」を基本とするが、コード・識別子を囲む場合は半角でもよい。

## 絶対に変えないもの（Markdown 構造）

- **コードブロック（``` で囲まれた部分）の中身はコードそのものを変更しない。** コード中の英語コメント（`//` `/* */` `#`）は日本語に訳してよい（訳すのを推奨）。ただし文字列リテラル、コマンド、出力例、識別子は変えない。
- mermaid 図のブロックは、ノード内のラベル文字列（`"..."` やラベル部分）のみ訳してよい。ノードID・矢印・構文は変えない。`<br/>` は残す。ラベルに `(` `)` 等を入れる場合は必ず `"..."` で囲む。
- インラインコード `` `foo` ``、関数名・構造体名・ファイルパス・sysctl 名・コマンド名・Kconfig 名は訳さない。
- **見出し末尾の `{#slug}` は一字一句そのまま残す。** 見出しテキスト部分のみ訳す。
  - 例: `## What bandwidth control does {#what-bandwidth-control-does}` → `## 帯域制御が行うこと {#what-bandwidth-control-does}`
- リンクの URL 部分 `(...)` は変更しない。リンクテキスト `[...]` は訳してよい（コミット名・論文名・記事タイトルなど固有の題名は原文のまま）。
- 表（table）の列数・区切り行を崩さない。
- admonition（`!!! note "Title"` など）のキーワード（note/warning/tip 等）は変えず、タイトル文字列と本文を訳す。本文のインデント（4スペース）を維持する。
- HTML タグ、`{ .class }` などの attr_list、脚注記号は残す。
- 行の追加・削除で構造を壊さない（段落は段落、リストはリストのまま）。

- 強調 `**...**` の内側を句読点や括弧で終わらせない（Markdown が強調として認識しなくなる）。括弧は強調の外に出す。

## 訳さないもの

- 人名、メーリングリスト名（LKML, lore）、記事・論文・パッチのタイトル、コミットハッシュ、カーネルバージョン。
- 引用されたメール・コミットメッセージの英文本文（`>` の引用ブロックで原文をそのまま引用しているもの）。必要なら直後に「（訳: …）」を付けてもよいが、原文は残す。
  - ただし各ページ冒頭の `> 一行サマリー` はページの説明文なので **訳す**。

## 用語集（統一すること）

原語のまま（カタカナ化しない）: page, folio, slab, cgroup, namespace, RCU, softirq, tasklet, workqueue, kthread, inode, dentry, superblock, bio, BPF/eBPF, XDP, io_uring, THP, MGLRU, DAMON, NUMA, TLB, MMU, IOMMU, DMA, KVM, syscall（文中では「システムコール」も可）, futex, mutex, spinlock, seqlock, LSM, SELinux, ftrace, kprobe, tracepoint, perf

| 英語 | 訳語 |
|---|---|
| kernel | カーネル |
| subsystem | サブシステム |
| memory management | メモリ管理 |
| page table | ページテーブル |
| page fault | ページフォールト |
| page cache | ページキャッシュ |
| reclaim | 回収（メモリ回収） |
| writeback | ライトバック |
| allocator | アロケータ |
| allocation / allocate | 割り当て / 割り当てる |
| free (v.) | 解放する |
| scheduler | スケジューラ |
| run queue (runqueue) | ランキュー |
| preemption / preempt | プリエンプション / プリエンプトする |
| context switch | コンテキストスイッチ |
| throttle / throttling | スロットリングする / スロットリング |
| interrupt | 割り込み |
| interrupt handler | 割り込みハンドラ |
| lock / locking | ロック |
| lock contention | ロック競合 |
| race condition | 競合状態（レースコンディション） |
| critical section | クリティカルセクション |
| memory barrier | メモリバリア |
| file system / filesystem | ファイルシステム |
| block device | ブロックデバイス |
| device driver | デバイスドライバ |
| network stack | ネットワークスタック |
| socket | ソケット |
| packet | パケット |
| user space / userspace | ユーザー空間 |
| kernel space | カーネル空間 |
| address space | アドレス空間 |
| virtual memory | 仮想メモリ |
| physical memory | 物理メモリ |
| process / task / thread | プロセス / タスク / スレッド |
| hot path / fast path / slow path | ホットパス / 高速パス / 低速パス |
| overhead | オーバーヘッド |
| latency / throughput | レイテンシ / スループット |
| trade-off | トレードオフ |
| design decision | 設計判断 |
| upstream / mainline | アップストリーム / メインライン |
| patch / patch series | パッチ / パッチシリーズ |
| maintainer | メンテナ |
| merged (into mainline) | （メインラインに）マージされた |
| regression | リグレッション |
| tunable | チューニング可能なパラメータ |
| vulnerability | 脆弱性 |
| mitigation | 緩和策 |
| hypervisor / guest / host | ハイパーバイザ / ゲスト / ホスト |
| tracing | トレーシング |
| War Stories | 事件簿 |
| Getting Started | はじめに |
| CVE case study | CVE事例研究 |
| use-after-free | use-after-free（原語のまま） |
| What's New | 更新履歴 |
| time / timekeeping | 時間 / 時刻管理 |
| timer | タイマー |
| Observability | 可観測性 |
| verifier (BPF) | 検証器（初出は「検証器（verifier）」） |
| capability | ケーパビリティ |
| skeleton / trampoline / tail call | スケルトン / トランポリン / テールコール |
| wait queue / waitqueue | 待ちキュー |
| Further reading | 参考文献 |
| See also | 関連項目 |
| Key takeaways | 要点 |
| Life of a ... | 〜の一生 |
| Why ... | なぜ〜か |

### 事件簿（War Stories）ページの定型見出し

| 英語 | 訳語 |
|---|---|
| Disclosed | 公開日 |
| Reported by / Found by | 報告者 / 発見者 |
| Bug present since | バグの存在期間 |
| Exploitable since | 悪用可能になった時期 |
| Fixed in | 修正 |
| Actively exploited | 実際の悪用 |
| Before state | 事前の状態 |
| The trigger | きっかけ |
| Resolution | 解決 |
| What it taught us | 教訓 |
| External references | 外部参考資料 |
| Pattern to watch for | 注意すべきパターン |
| Observed behavior | 観測された挙動 |
| Why it happened | なぜ起きたか |
| Introduced in | 混入 |
| （冒頭の帰属表記） | `*[事件簿: …](../war-stories.md)の一部。*` |
| erratum / errata | エラッタ |
| grace period | グレースピリオド |
| extent | エクステント |

上の表にない用語は、Linux カーネル関連の日本語文献で一般的な訳を使う。迷ったら原語のまま残す。長音記号は「ユーザー」「サーバー」「メモリ」「コンピュータ」「パラメータ」「ハンドラ」「アロケータ」「スケジューラ」とする（語尾の長音は、上の語以外は JIS 慣例に従い省略してよい）。
