# メモリホットプラグ {#memory-hotplug}

> システムを動かしたまま物理メモリを追加・削除する仕組み。クラウドVMが再起動なしで拡大・縮小できるようにする機構

## メモリホットプラグとは何か {#what-is-memory-hotplug}

メモリホットプラグとは、実行時に物理RAMを追加・削除するカーネルの能力である。この機能には、互いに異なる方向へ引っ張り合う2つの大きなユースケースがある:

**物理的なホットプラグ** — ホットスワップ可能なメモリベイを備えたエンタープライズサーバーは、システムの稼働中に新しいDIMMを受け入れられる。データベースサーバーは、メンテナンス時間なしに256 GBから512 GBへ拡張できる。同様に、故障したDIMMを電源を落とさずに隔離して取り外せる。

**仮想マシンのメモリサイズ変更** — クラウドのハイパーバイザは、アイドル状態のテナントから未使用のページを回収して忙しいテナントに与えるために、ゲストVMにメモリを追加・削除する。現在ではこれが主要なユースケースである。オーケストレーションシステムは、ワークロードが急増したときにVMのメモリを増やしたり、リソースを回収するために減らしたりできる。`virtio-mem` のようなツールは、その下の仕組みとしてホットプラグを使う。

カーネルは対称的な2つの操作を提供する:

- **ホットアド**（オンライン化）: 新しい物理メモリ範囲を追加し、アロケータが使えるようにする。
- **ホットリムーブ**（オフライン化）: メモリ範囲からすべてのページを退避させてから、物理的に取り外す。ホットリムーブはホットアドよりはるかに難しい。

## メモリセクション {#memory-sections}

カーネルのスパースメモリモデルは、物理メモリを固定サイズの**セクション**単位で追跡する。各セクションは、独立してオンライン化・オフライン化できる最小単位である。

セクションサイズはアーキテクチャごとに `SECTION_SIZE_BITS` で定義される。x86-64では:

```c
/* arch/x86/include/asm/sparsemem.h */
# define SECTION_SIZE_BITS  27  /* matt - 128 is convenient right now */
```

つまりx86-64ではセクションは `1 << 27` = **128 MB** である。arm64も4Kページではデフォルトで同じ128 MBのセクションを使う。`CONFIG_ARM64_64K_PAGES` では512 MBのセクション（`SECTION_SIZE_BITS = 29`）を使う。

各セクションは `include/linux/mmzone.h` の `struct mem_section` で表される:

```c
struct mem_section {
    /*
     * 論理的には、これはstruct pageの配列へのポインタである。
     * ただし、他の仕掛けと一緒に格納されている。
     */
    unsigned long section_mem_map;

    struct mem_section_usage *usage;
#ifdef CONFIG_PAGE_EXTENSION
    /* ... */
#endif
};
```

`section_mem_map` フィールドは、このセクションの `struct page` 配列へのポインタと、下位ビットに詰め込まれた一連の状態フラグの両方をエンコードしている:

- `SECTION_MARKED_PRESENT` — ここに物理メモリが存在する
- `SECTION_HAS_MEM_MAP` — `struct page` 配列が有効である
- `SECTION_IS_ONLINE` — セクションがアロケータに対してオンライン化されている

ユーザーから見えるレベルでは、カーネルは複数のセクションを**メモリブロック**にまとめる。メモリブロックはsysfsで公開される単位である。`MIN_MEMORY_BLOCK_SIZE` は `1 << SECTION_SIZE_BITS` に設定されているので、x86-64では最小ブロックサイズは128 MBである。実際にはブロックはもっと大きいことが多い。

## sysfsインターフェイス {#the-sysfs-interface}

各メモリブロックは `/sys/devices/system/memory/` の下にsysfsディレクトリを持つ:

```
/sys/devices/system/memory/
├── auto_online_blocks          # 自動オンライン化のポリシー
├── block_size_bytes            # メモリブロックサイズ（16進のバイト数）
├── memory0/                    # ブロック0（ファームウェア予約であることが多い）
│   ├── node0                   # NUMAノードへのシンボリックリンク
│   ├── phys_index              # 物理ブロックインデックス
│   ├── removable               # ブロックを取り外せるかどうか
│   ├── state                   # offline / online / going-offline
│   └── valid_zones             # このブロックをオンライン化できるゾーン
├── memory1/
│   └── state
...
├── memory128/
│   └── state
```

`state` ファイルが主要な制御インターフェイスである。次の値を受け付ける:

| 書き込む値 | 効果 |
|-------------|--------|
| `online` | 自動ポリシーでオンライン化（`MMOP_ONLINE`） |
| `online_kernel` | `ZONE_NORMAL` にオンライン化（`MMOP_ONLINE_KERNEL`） |
| `online_movable` | `ZONE_MOVABLE` にオンライン化（`MMOP_ONLINE_MOVABLE`） |
| `offline` | オフライン化を試みる（失敗することがある） |

```bash
# メモリブロックの状態を確認する
cat /sys/devices/system/memory/memory128/state

# ZONE_MOVABLEにオンライン化する
echo online_movable > /sys/devices/system/memory/memory128/state

# オフライン化する（移動不能なページがあると失敗することがある）
echo offline > /sys/devices/system/memory/memory128/state
```

これらの書き込みを解析するカーネルコードは `drivers/base/memory.c` にある:

```c
/* drivers/base/memory.c */
static const char *const online_type_to_str[] = {
    [MMOP_OFFLINE]         = "offline",
    [MMOP_ONLINE]          = "online",
    [MMOP_ONLINE_KERNEL]   = "online_kernel",
    [MMOP_ONLINE_MOVABLE]  = "online_movable",
};
```

## ホットアドのパス {#hot-add-path}

新しいメモリが現れると（ファームウェア/ACPIの通知や `virtio-mem` デバイスによって）、カーネルは次のパスをたどる:

```
ファームウェア/ACPIまたはvirtio-memが新しいメモリ範囲を通知
        │
        ▼
add_memory(nid, start, size, mhp_flags)    [mm/memory_hotplug.c]
        │
        ▼
add_memory_resource()
        │
        ├── arch_add_memory()              [アーキテクチャ固有のページテーブル設定]
        │         新しい範囲をカーネルのリニアマップにマップする
        │
        ├── create_memory_block_devices()  [drivers/base/memory.c]
        │         /sys/devices/system/memory/memoryN のエントリを作成する
        │
        └── (auto_online_blocksポリシー)
              │
              ▼
        online_pages(pfn, nr_pages, zone, group)  [mm/memory_hotplug.c]
              │
              ├── move_pfn_range_to_zone()     ページをゾーンに関連付ける
              ├── memory_notify(MEM_GOING_ONLINE, ...)
              ├── online_pages_range()          ページをバディアロケータに追加する
              ├── undo_isolate_page_range()     範囲を割り当てに開放する
              └── memory_notify(MEM_ONLINE, ...)
```

`add_memory()` は、NUMAノードID、物理開始アドレス、バイト単位のサイズ、フラグのビットフィールド（`mhp_t`）を受け取る。主なフラグには次のものがある:

- `MHP_MERGE_RESOURCE` — 新しいリソースエントリを `/proc/iomem` 内の隣接するものとマージする
- `MHP_MEMMAP_ON_MEMORY` — `struct page` 配列（vmemmap）をホットアドされたメモリ自体の中に格納し、既存のメモリを節約する

`arch_add_memory()` はアーキテクチャ固有の作業を担当する。カーネルのダイレクトマッピングが新しい物理アドレスをカバーするよう、ページテーブルエントリを構築する。

`online_pages()` はメモリモデルの作業を行う。PFN範囲を要求されたゾーンに関連付け、ドライバやサブシステムが反応できるようメモリ通知を発行し、`online_pages_range()` でページをバディアロケータに追加し、それから `undo_isolate_page_range()` を呼んで範囲を通常の割り当てに開放する。

### 新しいメモリにはZONE_MOVABLEかZONE_NORMALか {#zone_movable-vs-zone_normal-for-new-memory}

オンライン化の際には、ゾーンを選ばなければならない:

**`ZONE_NORMAL`（online_kernel）**: 新しいページは通常のカーネルゾーンに加わる。カーネルのデータ構造がここに配置されうる。これは一方通行の扉である。カーネルが移動不能なデータを置いてしまうと、後でそのセクションをオフライン化できない。

**`ZONE_MOVABLE`（online_movable）**: 新しいページはmovableゾーンに加わる。カーネルは移動不能な割り当てをここに決して置かないことを保証する。これは出口である。このメモリをオフライン化する必要が生じても、すべてのページを他所へ移行できるので可能である。

!!! tip "VMでホットプラグしたメモリには常に `online_movable` を使う"
    VMのメモリを後でオフライン化（縮小）する必要が生じる可能性が少しでもあるなら、新しいセクションは `ZONE_MOVABLE` としてオンライン化すること。`ZONE_NORMAL` としてオンライン化すると、カーネルのslab、ページテーブル、その他の移動不能な割り当てが即座にそこに置かれうるので、そのメモリは実質的に恒久的なものになる。

## ZONE_MOVABLE {#zone_movable}

`ZONE_MOVABLE` は、まさにホットリムーブを可能にするために存在する。その約束は厳格である。**移動不能なカーネル割り当ては決してここに置かれない**。別の場所に移行できるページだけが `ZONE_MOVABLE` に置かれる。無名メモリ、ファイルで裏付けられたページ、その他同様の移動可能な内容である。

このゾーンは通常のシステムではデフォルトで作成されない。ブートパラメータで有効にする:

```bash
# 8GBを除くすべてをZONE_MOVABLEに確保する（8GBはカーネルが使えるゾーンに残る）
kernelcore=8G

# あるいは、ZONE_MOVABLEにする量を指定する
movablecore=80%
```

`kernelcore=N` は、カーネルが使えるゾーンに残さなければならないメモリ量を設定する。残りは `ZONE_MOVABLE` になる。`movablecore=N` はその逆で、movableにする量を指定する。

現在のゾーン配置を調べるには:

```bash
# Movableを含むすべてのゾーンを表示する
grep -A5 "Node.*Movable" /proc/zoneinfo

# /proc/meminfoからの簡潔な要約
grep -E "MemTotal|MemFree" /proc/meminfo
```

!!! note "ZONE_MOVABLEとCMA"
    カーネルゾーン内のCMAページ（[CMA](cma.md)を参照）は `ZONE_MOVABLE` と同様に振る舞い、移行できる。しかし、メモリセクション全体を確実に回収できるようにするための主要な仕組みは `ZONE_MOVABLE` である。

## ホットリムーブのパス {#hot-remove-path}

ホットリムーブはホットアドよりはるかに難しい。根本的な課題は、カーネルが物理メモリ範囲を返却する前に、その範囲内のすべての `struct page` が空いていなければならないことである。つまり、生きているページをすべて他所へ移行する必要がある。移行できないページはオフライン化を妨げる。

オフライン化のパスは次のとおりである:

```
ユーザーがsysfsのstateファイルに "offline" を書き込む
        │
        ▼
memory_block_offline()              [drivers/base/memory.c]
        │
        ▼
offline_pages(pfn, nr_pages, zone, group)  [mm/memory_hotplug.c]
        │
        ├── zone_pcp_disable()          CPUごとのページリストを無効化
        ├── lru_cache_disable()         LRUキャッシュを無効化
        │
        ├── start_isolate_page_range()  [mm/page_isolation.c]
        │       すべてのpageblockをMIGRATE_ISOLATEにマークする
        │       範囲への新たな割り当てを防ぐ
        │       隔離できないpageblockがあれば-EBUSYを返す
        │
        ├── memory_notify(MEM_GOING_OFFLINE, ...)
        │
        ├── [ループ]
        │       scan_movable_pages()    まだ存在するページを探す
        │       do_migrate_range()      migrate_pages()で外へ移行する
        │       [移動可能なページがなくなるか失敗するまで繰り返す]
        │
        ├── dissolve_free_hugetlb_folios()   ヒュージページを処理する
        │
        ├── test_pages_isolated()       範囲が完全に空いたことを確認する
        │
        ├── __offline_isolated_pages()  バディアロケータからページを取り除く
        │       セクションを SECTION_IS_ONLINE = 0 にマークする
        │
        ├── undo_isolate_page_range()   成功時: 不要
        │   (失敗時: MIGRATE_MOVABLEに戻して取り消す)
        │
        └── memory_notify(MEM_OFFLINE, ...)
              失敗時は memory_notify(MEM_CANCEL_OFFLINE, ...)
```

### ページの隔離 {#page-isolation}

`mm/page_isolation.c` の `start_isolate_page_range()` は、対象範囲内のすべてのpageblockを `MIGRATE_ISOLATE` としてマークする。隔離されると、ページアロケータはこの範囲のページを新たな割り当てに渡さなくなる。これにより、範囲からページが解放されたのに新たな割り当てがすぐにまたそこを埋めてしまう、という競合を防ぐ。

隔離できないpageblockがあると（たとえばゾーン境界をまたいでいる、あるいは構造上の制約がある場合）、`start_isolate_page_range()` は `-EBUSY` を返し、オフライン化は即座に失敗する。

### ページの移行 {#page-migration}

`do_migrate_range()` は、移行理由を `MR_MEMORY_HOTPLUG` として `migrate_pages()` を呼ぶ。移行はシステム内の別の場所に代替ページを割り当て、すべての内容とページテーブルのマッピングを移す。ページは `page_is_unmovable()` で分類される:

```c
/* mm/page_isolation.c */
bool page_is_unmovable(struct zone *zone, struct page *page,
        enum pb_isolate_mode mode, unsigned long *step)
{
    /* ブート時の割り当てとメモリホールはPG_reservedで移動不能 */
    if (PageReserved(page))
        return true;

    /* ZONE_MOVABLE内のページはすべて移動可能とみなされる */
    if (zone_idx(zone) == ZONE_MOVABLE)
        return false;
    ...
}
```

### 失敗のケース {#failure-cases}

オフライン化はいくつかの理由で失敗しうる。カーネルはそれをdmesgに報告する:

| 失敗理由 | 原因 |
|----------------|-------|
| `failure to isolate range` | pageblockをMIGRATE_ISOLATEにマークできなかった |
| `unmovable page` | 範囲内のページに移行方法がない |
| `notifier failure` | カーネルの通知先（ドライバ、サブシステム）がオフライン化を拒否した |
| `signal backoff` | 移行ループ中にシグナルが届いた |
| `failure to dissolve huge pages` | hugetlbfsのページを解放できなかった |
| `memory holes` | 範囲にRAMでない隙間が含まれている |

!!! warning "移動不能なページが最もよくある障害要因"
    メモリが `ZONE_NORMAL` としてオンライン化されていた場合、カーネルのslabキャッシュ、ページテーブル、その他の移動不能な割り当てがそこに置かれている可能性がある。これらは移行できない。オフライン化は移行ループを空回りし、最終的に `unmovable page` で失敗する。唯一の対処法は、今後ホットプラグするメモリを `ZONE_MOVABLE` としてオンライン化することである。

オフライン化に失敗した後は、`undo_isolate_page_range()` と `memory_notify(MEM_CANCEL_OFFLINE, ...)` がセクションを通常の使用状態に戻す。

### remove_memory {#remove_memory}

物理範囲をカバーするすべてのブロックがオフライン化されると、`remove_memory()` を呼んでハードウェアの取り外しを完了できる:

```c
/* include/linux/memory_hotplug.h (CONFIG_MEMORY_HOTREMOVE) */
extern int remove_memory(u64 start, u64 size);
extern void __remove_memory(u64 start, u64 size);
extern int offline_and_remove_memory(u64 start, u64 size);
```

`remove_memory()` はすべてのブロックがオフラインであることを確認してから、`arch_remove_memory()` を呼んでその範囲のページテーブルマッピングを解体し、続いて `sparse_remove_section()` で `struct page` 配列を解放する。

## メモリのオンライン化ポリシー {#memory-onlining-policies}

### auto_online_blocks {#auto_online_blocks}

`/sys/devices/system/memory/auto_online_blocks` にあるsysfs属性 `auto_online_blocks` は、新しいメモリブロックが現れたときに何が起きるか（自動的にオンライン化されるか、どのゾーンにか）を制御する:

```bash
# 現在のポリシーを確認する
cat /sys/devices/system/memory/auto_online_blocks

# 取りうる値: offline, online, online_kernel, online_movable

# 新しいブロックを自動的にZONE_MOVABLEにオンライン化する
echo online_movable > /sys/devices/system/memory/auto_online_blocks

# 新しいブロックを手動処理のためにオフラインのままにする
echo offline > /sys/devices/system/memory/auto_online_blocks
```

この属性は、カーネルの `mhp_get_default_online_type()` / `mhp_set_default_online_type()` の組に直接対応する。

### online_policyモジュールパラメータ {#the-online_policy-module-parameter}

`memory_hotplug.online_policy`（カーネルコマンドラインまたは `/sys/module/memory_hotplug/parameters/online_policy` で設定可能）は、単なる `online`（自動）モードを使うときのゾーン選択を制御する:

| ポリシー | 挙動 |
|--------|----------|
| `contig-zones`（デフォルト） | ゾーンを連続に保つ。隣接するメモリブロックが使っているゾーンにオンライン化する |
| `auto-movable` | movable:kernelの比率が許す場合に `ZONE_MOVABLE` にオンライン化する |

`auto-movable` ポリシーは、`ZONE_MOVABLE : kernel zones` の比率の上限として `auto_movable_ratio`（デフォルト301%）を使う。これにより、システムがMOVABLEに偏りすぎてカーネルの割り当てが枯渇するのを防ぐ。

### クラウドVM向けの推奨設定 {#recommended-settings-for-cloud-vms}

ハイパーバイザによってメモリが追加・削除されるゲストVMでは:

```bash
# ブート時に（/etc/rc.localまたはcloud-initで）:
echo online_movable > /sys/devices/system/memory/auto_online_blocks
```

これにより、ホットプラグされたすべてのセクションがすぐにアプリケーションから使え、後で妨げなく取り外せるようになる。

## NUMAとホットプラグ {#numa-and-hotplug}

### 既存のノードへのメモリ追加 {#adding-memory-to-an-existing-node}

よくあるケースは、新しいDIMMやVMのメモリ増分がすでにオンラインのNUMAノードに属する場合である。これは比較的単純である。`add_memory(nid, ...)` が既存ノードのIDで呼ばれ、ノードはすでにオンラインなので `try_online_node()` は何もしない。

### 新しいNUMAノードの追加 {#adding-a-new-numa-node}

まったく新しいNUMAノードにメモリを追加するのはより複雑である。まずノードの `pg_data_t` 構造体を割り当て、NUMAトポロジ（距離テーブル、フォールバックリスト）を再構築しなければならない。`try_online_node()` が最初のメモリの場合を処理する:

```c
/* mm/memory_hotplug.c */
int try_online_node(int nid)
{
    int ret;
    mem_hotplug_begin();
    ret = __try_online_node(nid, true);
    mem_hotplug_done();
    return ret;
}
```

新しいノードにCPUが関連付けられている場合（CPUホットプラグ）、さらに調整が必要になる。CPUは、そのノードがメモリを持った後にのみオンライン化しなければならない。

### ノードから最後のメモリを削除する {#removing-the-last-memory-from-a-node}

`offline_pages()` がNUMAノードから最後の存在ページを削除しようとするとき、`node_notify(NODE_REMOVING_LAST_MEMORY, ...)` でシステムに通知する。成功すると、`node_clear_state(node, N_MEMORY)` がそのノードをメモリなしとしてマークする。そのノードの `kswapd` と `kcompactd` は停止される。ゾーンリストは空のノードを除いて再構築される。

## ホットプラグイベントの観察 {#observing-hotplug-events}

### dmesg {#dmesg}

カーネルはメモリのオンライン化・オフライン化のイベントをログに記録する:

```
kernel: [  123.456789] memory: online [mem 0x0000001000000000-0x000000103fffffff]
kernel: [  456.789012] Offlined Pages 32768
```

オフライン化に失敗した場合:

```
kernel: memory offlining [mem 0x0000001000000000-0x000000101fffffff] failed due to unmovable page
```

### udevイベント {#udev-events}

sysfsにメモリブロックが作成されると、`udev` はサブシステム `memory` のデバイスについて `add` イベントを受け取る。`udev` ルールでカスタムスクリプトを起動できる:

```
ACTION=="add", SUBSYSTEM=="memory", ATTR{state}=="offline", \
  RUN+="/usr/local/bin/memory-online.sh $env{DEVPATH}"
```

`systemd` やディストリビューションのスクリプトは、カーネル組み込みの `auto_online_blocks` の代替として、この方法で自動オンライン化を実装している。

### /sys/devices/system/memory/ {#sysdevicessystemmemory}

```bash
# メモリブロックの総数を数える
ls /sys/devices/system/memory/ | grep -c memory

# オフラインのブロックをすべて列挙する
for d in /sys/devices/system/memory/memory*/; do
  state=$(cat "$d/state")
  [ "$state" = "offline" ] && echo "$d: $state"
done

# 各ブロックをどのゾーンにオンライン化できるかを表示する
for d in /sys/devices/system/memory/memory*/; do
  echo "$d: $(cat $d/valid_zones 2>/dev/null)"
done
```

## クラウドとVMのユースケース {#the-cloud-and-vm-use-case}

### virtio-mem {#virtio-mem}

`virtio-mem`（Linux 5.8で導入）は、KVM/QEMU向けの現代的なメモリバルーンデバイスである。ページをピン留めしてゲストから取り上げることで膨張する従来の `virtio-balloon` と異なり、`virtio-mem` はメモリホットプラグを使う。ゲストから見える実際の物理アドレス範囲を追加・削除するのである。

ドライバ（`drivers/virtio/virtio_mem.c`）は、アラインメントに基づいて自動的に選ばれる2つのモードで動作する:

- **Sub Block Mode（SBM）**: 1つのLinuxメモリブロックが、より小さい複数のサブブロックにまたがる。ブロック内の個々のサブブロックをplug/unplugできる。メモリはLinuxメモリブロック単位で追加・削除される。
- **Big Block Mode（BBM）**: 1つのBig Blockが1つ以上のLinuxメモリブロックにまたがる。デバイスのブロックサイズがLinuxメモリブロックサイズを超える場合に使われる。

どちらのモードでも、`virtio-mem` はメモリをplugするために `add_memory()` を、unplugするために `offline_and_remove_memory()` を呼ぶ。ドライバはどのサブブロックがplugされているかをビットマップで追跡する。

ゾーン選択への含意: `virtio-mem` は、追加したブロックのオンライン化をユーザー空間またはカーネルの `auto_online_blocks` に頼っている。`ZONE_MOVABLE` のブロックはオフライン化できるので、`virtio-mem` がメモリをunplugできるようにするには `auto_online_blocks=online_movable` の設定が不可欠である。

### なぜVMはゲストRAM全体にZONE_MOVABLEを好むか {#why-vms-prefer-zone_movable-for-all-guest-ram}

一部のVM構成（特に `movable_node` ブートオプションを使うもの）は、ゲストメモリ全体を `ZONE_MOVABLE` に置く。その理由は:

1. ハイパーバイザは、ホットプラグされたページだけでなく、任意のゲストページを回収したい場合がある。
2. ゲストカーネルは、自分のメモリが恒久的に割り当てられていると想定すべきではない。
3. ライブマイグレーションはゲストメモリ全体を移動させるので、それが移行可能である必要がある。

`movable_node` ブートパラメータは、NUMAノード全体を `ZONE_MOVABLE` にすることを強制する。

### Kubernetesとメモリホットプラグ {#kubernetes-and-memory-hotplug}

一部のKubernetesランタイム（特に `kata-containers` や `cloud-hypervisor` を使うもの）は、Podのリソース制限に合わせてVMのメモリをリサイズする。Podのメモリ制限が増えると、ランタイムはVMにメモリをホットアドする。制限が減ると（メモリバルーン方式を使う場合）、メモリセクションがオフライン化されて取り除かれる。このため、`auto_online_blocks=online_movable` の設定はクラウドVMイメージにおける一般的なチューニングの推奨事項となっている。

## 主要なソースファイル {#key-source-files}

| ファイル | 説明 |
|------|-------------|
| [`mm/memory_hotplug.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memory_hotplug.c) | ホットプラグの中核ロジック: `add_memory()`、`remove_memory()`、`online_pages()`、`offline_pages()`、`do_migrate_range()` |
| [`include/linux/memory_hotplug.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/memory_hotplug.h) | 公開API: `add_memory()`、`online_pages()`、`offline_pages()`、`mhp_t` フラグ、`struct mhp_params` |
| [`drivers/base/memory.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/base/memory.c) | sysfsインターフェイス: `state` 属性、`auto_online_blocks`、`memory_block_online()`、`memory_block_offline()` |
| [`mm/page_isolation.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_isolation.c) | ページの隔離: `start_isolate_page_range()`、`undo_isolate_page_range()`、`page_is_unmovable()` |
| [`include/linux/mmzone.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mmzone.h) | `struct mem_section`、`SECTION_SIZE_BITS` の参照、`struct mem_section_usage` |
| [`include/linux/memory.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/memory.h) | `struct memory_block`、`struct memory_group`、`enum memory_block_state`、`register_memory_notifier()` |
| [`arch/x86/include/asm/sparsemem.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/sparsemem.h) | x86-64の `SECTION_SIZE_BITS = 27`（128 MBのセクション） |
| [`drivers/virtio/virtio_mem.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/virtio/virtio_mem.c) | `virtio-mem` ドライバ — `add_memory()` / `offline_and_remove_memory()` を使う |
| [`mm/mm_init.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mm_init.c) | `kernelcore=` と `movablecore=` ブートパラメータの解析 |

## 試してみる {#try-it-yourself}

```bash
# すべてのメモリブロックとその状態を列挙する
ls /sys/devices/system/memory/ | grep memory

# 現在のオンライン化ポリシーを確認する
cat /sys/devices/system/memory/auto_online_blocks

# メモリブロックサイズを確認する
printf "%d MB\n" $(($(cat /sys/devices/system/memory/block_size_bytes) / 1024 / 1024))

# オフラインのメモリブロックを探す
for d in /sys/devices/system/memory/memory*/; do
  state=$(cat "$d/state" 2>/dev/null)
  [ "$state" = "offline" ] && echo "OFFLINE: $d"
done

# メモリブロックをどのゾーンにオンライン化できるかを確認する
cat /sys/devices/system/memory/memory0/valid_zones

# /proc/zoneinfoでZONE_MOVABLEのサイズを確認する
grep -A5 "Node.*Movable" /proc/zoneinfo

# dmesgでホットプラグイベントをリアルタイムに監視する
dmesg -w | grep -E "memory|hotplug|Offlined|online"

# virtio-memを使うVMで、ドライバがロードされているか確認する
lsmod | grep virtio_mem
ls /sys/bus/virtio/drivers/virtio_mem/ 2>/dev/null
```

## よくある問題 {#common-issues}

### オフライン化が "unmovable page" で失敗する {#offline-fails-with-unmovable-page}

セクションに移動不能なカーネル割り当て（slabオブジェクト、ページテーブル、ピン留めされたページ）が含まれている。

**診断**:
```bash
dmesg | grep "offlining.*failed"
# 具体的な失敗理由を探す

# ブロックがどのゾーンにあるかを確認する
cat /sys/devices/system/memory/memoryN/valid_zones
# "Normal" しか表示されない場合、normalゾーンのすべての割り当てが妨げになっている
```

**対処法**:

- 可能なら、代わりにブロックを `ZONE_MOVABLE` としてオンライン化する（再オンライン化が必要）
- 次のホットプラグイベントの前に `auto_online_blocks=online_movable` を設定する
- オフライン化を繰り返し試す — 試行の間にメモリコンパクションが走り、範囲が空く可能性がある

### ホットプラグ後にメモリブロックが現れない {#memory-block-not-appearing-after-hotplug}

物理デバイスは追加されたが、sysfsのエントリが現れない。

- `dmesg` でACPIやファームウェアのエラーを確認する
- 実行中のカーネルで `CONFIG_MEMORY_HOTPLUG=y` になっているか確認する（`zcat /proc/config.gz | grep MEMORY_HOTPLUG`）
- VMでは、ハイパーバイザからのシグナル（ACPI通知や `virtio-mem` イベント）が届いたか確認する

### ZONE_MOVABLEが空である {#zone_movable-is-empty}

`grep -A5 "Node.*Movable" /proc/zoneinfo` がゼロページを示す。

- `ZONE_MOVABLE` にページが入るのは、ブート時に `kernelcore=`、`movablecore=`、`movable_node` のいずれかが指定された場合か、メモリが `online_movable` でオンライン化された場合だけである
- `online_movable` としてオンライン化されたホットプラグメモリはここに現れる

## 歴史 {#history}

### SPARSEMEMと基盤（v2.6.13、2005年） {#sparsemem-and-the-foundations-v2613-2005}

メモリホットプラグには新しい[メモリモデル](memory-model.md)が必要だった。古典的な `FLATMEM` モデルは、すべての物理メモリにまたがる連続した `mem_map` 配列を前提としており、実行時にメモリが現れたり消えたりすると破綻する。SPARSEMEMはセクションごとの `struct mem_section` 配列を導入し、`struct page` 配列を不連続かつ独立に割り当て可能にした。

### 初期のホットプラグサポート（v2.6.15、2006年） {#initial-hotplug-support-v2615-2006}

x86-64とia64向けに基本的なメモリホットアドのサポートがマージされ、`add_memory()` と初期のsysfsインターフェイスが提供された。ホットリムーブはまだサポートされていなかった。

### メモリのホットリムーブ（v2.6.24、2008年） {#memory-hot-remove-v2624-2008}

`offline_pages()` とホットリムーブのためのページ移行機構が追加され、`ZONE_MOVABLE` を取り外し可能なメモリの待機領域として使えるようになった。

### virtio-mem（v5.8、2020年） {#virtio-mem-v58-2020}

**コミット**: [35ee9a3e812b](https://git.kernel.org/linus/35ee9a3e812b)

**作者**: David Hildenbrand（Red Hat）

`virtio-mem` ドライバは、ハイパーバイザがゲストメモリを細粒度のブロックで追加・削除するための整ったインターフェイスを提供し、クラウドのユースケースにおいて、より粗いバルーンドライバ方式を置き換えた。

**LWNの記事**: [virtio-mem: paravirtualized memory hot(un)plug](https://lwn.net/Articles/820428/)

### memmap_on_memory（v5.14、2021年） {#memmap_on_memory-v514-2021}

`MHP_MEMMAP_ON_MEMORY` フラグと `memory_hotplug.memmap_on_memory` パラメータにより、ホットプラグされたメモリの `struct page` メタデータを、既存のRAMを消費するのではなく、ホットプラグされたメモリ自体の中に格納できるようになった。

## 参考資料 {#references}

- [LWN: Memory hotplug](https://lwn.net/Articles/197855/) — ホットリムーブ設計の初期の概要
- [LWN: virtio-mem](https://lwn.net/Articles/820428/) — virtio-memの設計とユースケース
- [numa](numa.md) — NUMAトポロジと、ホットプラグがノードのオンライン化・オフライン化とどう関わるか
- [cma](cma.md) — CMAと `MIGRATE_CMA`。関連する移動可能メモリの機構
- [compaction](compaction.md) — オフライン化中に `do_migrate_range()` が使うページ移行
- [page-allocator](page-allocator.md) — マイグレートタイプとゾーン配置

## 参考文献 {#further-reading}

- [Kernel docs: Memory hotplug](https://docs.kernel.org/admin-guide/mm/memory-hotplug.html) — sysfsインターフェイス、ゾーン選択、オンライン/オフラインの状態機械を扱う公式カーネルドキュメント
- [Kernel docs: Memory hotplug notifier](https://docs.kernel.org/core-api/memory-hotplug.html) — `memory_notify()`、`MEM_GOING_ONLINE`/`MEM_CANCEL_OFFLINE`、`struct memory_notify`
- [`Documentation/admin-guide/mm/memory-hotplug.rst`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/admin-guide/mm/memory-hotplug.rst) — メモリホットプラグ管理ガイドのカーネルソース
- [`mm/memory_hotplug.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memory_hotplug.c) — 中核の実装: `add_memory()`、`remove_memory()`、`online_pages()`、`offline_pages()`
- [`mm/page_isolation.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_isolation.c) — オフライン化中に使われる `start_isolate_page_range()` と `page_is_unmovable()`
- [`drivers/virtio/virtio_mem.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/virtio/virtio_mem.c) — VM向けにSub Block ModeとBig Block Modeのホットプラグを実装するvirtio-memドライバ
- [LWN: Memory hotplug](https://lwn.net/Articles/197855/) — ホットリムーブのパスとZONE_MOVABLEの根拠に関する初期の設計概要（2007年）
- [LWN: virtio-mem: paravirtualized memory hot(un)plug](https://lwn.net/Articles/820428/) — virtio-memの設計、サブブロックの粒度、クラウドVMのユースケース（2020年）
- [LWN: Memory offlining and migration](https://lwn.net/Articles/650996/) — `offline_pages()` の背後にあるページの隔離と移行の機構（2015年）
- [numa.md](numa.md) — ホットプラグでノードが追加・削除されるときのNUMAトポロジの変化
- [cma.md](cma.md) — `MIGRATE_CMA` ページと、`ZONE_MOVABLE` の移動可能ページ保証との関係
- [compaction.md](compaction.md) — メモリのオフライン化中に `do_migrate_range()` が再利用するページ移行機構
