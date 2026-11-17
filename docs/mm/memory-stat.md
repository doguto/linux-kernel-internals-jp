# memory.statフィールドリファレンス {#memorystat-field-reference}

> /sys/fs/cgroup/*/memory.stat（cgroup v2）の全フィールドを1つずつ詳しく解説する

## memory.statとは何か? {#what-is-memorystat}

`/sys/fs/cgroup/<cgroup>/memory.stat`は、カーネルがcgroupについて公開しているメモリアカウンティングのファイルの中で最も詳細なものである。これは[`mm/memcontrol.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c)の`memory_stat_show()`によって生成される。この関数はcgroupのper-CPU統計カウンタを読み取り、名前/値のペアとして整形する。

```bash
cat /sys/fs/cgroup/mycontainer/memory.stat
```

バイト値はすべて正確なバイト数である（page数でもキロバイトでもない）。イベントカウンタはすべて、cgroupが作成されてから単調増加する。

フィールド定義の正式な情報源は[`Documentation/admin-guide/cgroup-v2.rst`](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory-interface-files)である。

---

## 階層的アカウンティングとフラットなアカウンティングに関する注記 {#a-note-on-hierarchical-vs-flat-accounting}

`memory.stat`のすべてのフィールドは**階層的な合計**、つまりcgroup自身とそのすべての子孫cgroupの合計を報告する。このファイルを通じて、フラットな（そのcgroupローカルのみの）値は公開されていない。

つまり、次のような構成の場合:

```
/myapp (memory.stat: anon = 600 MiB)
├── /myapp/worker-1 (local anon = 400 MiB)
└── /myapp/worker-2 (local anon = 200 MiB)
```

`/myapp/memory.stat`を読むと`anon 629145600`（合計）が得られる。各子cgroupの`memory.stat`は、それ自身の小計のみを報告する。

`memory.current`も同様に階層的である。あるフィールドに対するcgroupごとのローカルな寄与は直接公開されていないが、次のように計算できる。

```
local = parent_stat_field - sum(child_stat_fields)
```

---

## フィールドのグループ {#field-groups}

### 無名メモリ {#anonymous-memory}

無名pageはファイルに裏付けられていない。ヒープの割り当て、スタックの伸長、`mmap(MAP_ANONYMOUS)`の領域を裏付ける。

---

#### `anon` {#anon}

**何を数えるか**: 現在RAMに常駐している無名メモリのバイト数。ヒープ、スタック、無名の`mmap()`領域、およびダーティになった`MAP_PRIVATE`のcopy-on-write pageが含まれる。

**カーネルのカウンタ**: cgroupの`vmstats`内の`NR_ANON_MAPPED`。無名pageがマップまたはアンマップされるたびに`mod_lruvec_state()`で更新される。`include/linux/mmzone.h`で定義されている。

**階層的**: はい。すべての子孫cgroupを含む。

**診断での使い方**: アプリケーションのヒープ/スタックのサイズを示す主要な指標である。`anon`が際限なく増えている場合はメモリリークを疑う。`anon_thp`と比較すれば、どれだけの割合がhuge pageにあるかが分かる。

```bash
# cgroup内の全プロセスのヒープとスタックのメモリ
grep '^anon ' /sys/fs/cgroup/mycontainer/memory.stat
```

---

#### `anon_thp` {#anon_thp}

**何を数えるか**: Transparent Huge Pages（THP）で裏付けられた無名メモリのバイト数。`anon`の部分集合である。

**カーネルのカウンタ**: cgroupのvmstats内の`NR_ANON_THPS`。

**階層的**: はい。

**診断での使い方**: 無名メモリのフットプリントのうち、どれだけが2 MiBのhuge pageにあるかを示す。`anon_thp / anon`の比率が高ければ、このcgroupではTHPがうまく機能している（TLBのカバー範囲が広い）ことを意味する。比率が予想外に低い場合は、断片化によってTHPの割り当てが妨げられている可能性がある。[Transparent Huge Pages](thp.md)を参照。

---

#### `shmem` {#shmem}

**何を数えるか**: 現在RAMに常駐している共有メモリ（`shmget`、`shm_open`、tmpfs）と、無名ファイルの`mmap(MAP_SHARED)` pageのバイト数。`/dev/shm`の使用量も含む。

**カーネルのカウンタ**: cgroupのvmstats内の`NR_SHMEM`。

**階層的**: はい。

**診断での使い方**: `shmem`が高く`anon`が低い場合、多くは共有メモリを介したプロセス間通信を示している（例: PostgreSQLの共有バッファプール、RedisのAOF共有メモリ）。共有pageは、最初にそれをフォールトインしたcgroupに課金される。

!!! note "shmemと課金の帰属"
    2つのcgroupが同じ共有メモリ領域をマップしている場合、物理pageの課金は一方のcgroupにのみ行われる。pageは、最初にページフォールトを起こしたcgroupに課金される。そのため、共有メモリをあるコンテナで作成し、別のコンテナでマップすると、意外なアカウンティング結果になることがある。

---

### ファイルバックのメモリ（ページキャッシュ） {#file-backed-memory-page-cache}

ファイルバックのpageはディスク上のファイルに裏付けられている。スワップへの書き込みなしに回収でき、（クリーンなら）破棄されるか、（ダーティなら）ディスクへフラッシュされる。

---

#### `file` {#file}

**何を数えるか**: 現在ページキャッシュにあり、このcgroupに帰属するファイルバックのメモリのバイト数。通常ファイルのクリーンページとダーティページ、`mmap(MAP_SHARED)`によるファイルマッピング、先読みされたpageが含まれる。

**カーネルのカウンタ**: cgroupのvmstats内の`NR_FILE_PAGES`。

**階層的**: はい。

**診断での使い方**: cgroup内で最も回収しやすいメモリである。`file`が大きく`anon`が小さい場合、そのcgroupは大量のファイルI/Oを行って結果をキャッシュしている。これは正常かつ健全である。圧迫時には、無名pageがスワップされる前に`file`のpageが追い出される。ただし、`memory.high`によるスロットリングが起きていて`file`が大きい場合は、キャッシュを収められるよう`memory.high`を引き上げることを検討する。

```bash
# メモリ使用量の大半はキャッシュか、それとも本物のワーキングセットか?
awk '/^anon / {anon=$2} /^file / {file=$2} END {
    total = anon + file
    printf "anon: %.0f MiB (%.0f%%)\nfile: %.0f MiB (%.0f%%)\n",
           anon/1048576, anon*100/total, file/1048576, file*100/total
}' /sys/fs/cgroup/mycontainer/memory.stat
```

---

#### `file_mapped` {#file_mapped}

**何を数えるか**: 少なくとも1つのプロセスの仮想アドレス空間に現在マップされている（つまりVMAに含まれている）ファイルバックのpageのバイト数。`file`の部分集合である。

**カーネルのカウンタ**: cgroupのvmstats内の`NR_FILE_MAPPED`。

**階層的**: はい。

**診断での使い方**: ファイルキャッシュのうち、`mmap()`を介して能動的に使われている部分である。マップされていないファイルキャッシュは、実行中のプロセスにコストをかけずに直ちに回収できる。マップされたファイルキャッシュは、まずアンマップが必要であり、再びアクセスされるとメジャーフォールトを引き起こすことがある。`file_mapped / file`の比率が高い場合、ファイルキャッシュの大部分が単なる先読みキャッシュではなくメモリマップされていることを意味する。

---

#### `file_dirty` {#file_dirty}

**何を数えるか**: 変更されたがまだディスクに書き込まれていないファイルバックのpageのバイト数。

**カーネルのカウンタ**: cgroupのvmstats内の`NR_FILE_DIRTY`。

**階層的**: はい。

**診断での使い方**: ダーティページは、まずディスクに書き込む（ライトバックする）ことなしには回収できない。メモリ圧迫下では、カーネルは回収の前に`flush`スレッドを起こしてそれらを書き出す。圧迫下で`file_dirty`が高いと、I/Oが増えて回収が遅くなる。`pglazyfreed`イベントと比較すること。このカウンタが増えている場合、pageはライトバックなしに回収されている（クリーンなpageにのみ当てはまる）。

---

#### `file_writeback` {#file_writeback}

**何を数えるか**: `flush`カーネルスレッドによって現在ディスクへライトバック中のファイルバックのpageのバイト数。

**カーネルのカウンタ**: cgroupのvmstats内の`NR_WRITEBACK`。

**階層的**: はい。

**診断での使い方**: メモリ圧迫下で`file_writeback`が非ゼロなのは正常で想定どおりである。カーネルはダーティページを回収できるようにクリーンにしているのである。値が持続的に高い場合、ストレージのI/Oパスが回収のボトルネックになっていることを意味する。確認には、I/OのPSI（`$CGROUP/io.pressure`）と突き合わせる。

---

#### `shmem_thp` {#shmem_thp}

**何を数えるか**: Transparent Huge Pagesで裏付けられた共有メモリ（`shmem` / tmpfs）のバイト数。`shmem`の部分集合である。

**カーネルのカウンタ**: cgroupのvmstats内の`NR_SHMEM_THPS`。

**階層的**: はい。

**診断での使い方**: 大きな無名共有メモリ領域を使うワークロード（データベース、科学技術計算）に関係する。shmemにTHPが使われているかどうかを示す。THPはTLBの圧迫を減らす。

---

#### `file_thp` {#file_thp}

**何を数えるか**: Transparent Huge Pagesで裏付けられたファイルバックのpageのバイト数（無名でないマッピング向けに導入された、読み取り専用のファイルTHP）。`file`の部分集合である。

**カーネルのカウンタ**: cgroupのvmstats内の`NR_FILE_THPS`。

**階層的**: はい。

**診断での使い方**: 読み取り専用のファイルマッピング（例: 共有ライブラリ、実行ファイル）がhuge pageを使っていることを示す。これにより、テキスト（コード）の比重が大きいワークロードでTLBの圧迫が減る。

---

### カーネルメモリ {#kernel-memory}

カーネルメモリは、cgroupのプロセスに代わってカーネルコードが割り当てるメモリである。slabオブジェクト、ページテーブル、カーネルスタック、ソケットバッファなどがある。cgroup v2では、カーネルメモリはデフォルトでcgroupに課金される（[背景: 新しいslabメモリコントローラ](https://lwn.net/Articles/798605/)）。

---

#### `kernel` {#kernel}

**何を数えるか**: このcgroupに帰属するカーネルメモリの総バイト数。`kernel_stack`、`pagetables`、`percpu`、`sec_pagetables`、およびslabメモリ（`slab_reclaimable` + `slab_unreclaimable`）の合計である。注意: ソケットバッファ（`sock`）は別に追跡されており、この合計には**含まれない**。

**階層的**: はい。

**診断での使い方**: `kernel`を使うと、このcgroup内のプロセスのためのカーネルデータ構造のオーバーヘッドを把握できる。短命なプロセスを多数実行するコンテナでは、`kernel`が無視できない大きさになることがある。これは`memory.current`に計上される。

!!! note "カーネルメモリはmemory.currentに計上される"
    cgroup v2では`memory.current = anon + file + kernel + ...`である。これはよく驚かれる点である。コンテナが上限に達する原因がアプリケーションのヒープではなく、カーネルのslabの増加（例: ファイルシステムの走査による巨大なdentryキャッシュ）である場合がある。

---

#### `kernel_stack` {#kernel_stack}

**何を数えるか**: このcgroup内のタスクのカーネルスタックのバイト数。各タスクはカーネルスタックを1つ持つ（x86-64では通常8 KiB、`KASAN`が有効な場合は16 KiB）。

**カーネルのカウンタ**: cgroupのvmstats内の`NR_KERNEL_STACK_KB`（KiB単位）。

**階層的**: はい。

**診断での使い方**: cgroup内の生存スレッド数に比例する。スパイクはスレッド生成のバースト（フォーク爆弾、接続の殺到）を示す。10,000スレッドを持つコンテナは、約80 MiBのカーネルスタックメモリを使う。

---

#### `pagetables` {#pagetables}

**何を数えるか**: このcgroup内のプロセスのページテーブルメモリのバイト数。ページテーブルは仮想アドレスを物理アドレスに変換するもので、プロセスのVMAが拡大するにつれてカーネルが割り当てる。

**カーネルのカウンタ**: cgroupのvmstats内の`NR_PAGETABLE`。

**階層的**: はい。

**診断での使い方**: 小さな無名マッピングが多数あって`pagetables`が大きい場合、仮想アドレス空間が高度に断片化したプロセス（`munmap()`なしに多数の小さな`mmap()`呼び出しを行う）が示唆される。THPを使うと、1つのhuge pageエントリが512個の個別の4 KiBエントリの代わりに2 MiBをカバーするため、ページテーブルは小さくなる。

---

#### `sec_pagetables` {#sec_pagetables}

**何を数えるか**: このcgroupに帰属するセカンダリページテーブル（例: KVMゲスト用のシャドウページテーブル、デバイスI/O仮想化のためのIOMMUページテーブル）のバイト数。

**カーネルのカウンタ**: cgroupのvmstats内の`NR_SECONDARY_PAGETABLE`。

**階層的**: はい。

**診断での使い方**: 主にKVM仮想マシンを実行するcgroupに関係する。IOMMUを使ったデバイスパススルーを行うコンテナでは非ゼロになる。

---

#### `percpu` {#percpu}

**何を数えるか**: このcgroup内のプロセスに代わって実行されるカーネルコードが割り当てたper-CPUカーネルメモリのバイト数。

**階層的**: はい。

**診断での使い方**: ほとんどのワークロードでは、実際上大きな値になることはまれである。主にアカウンティングの完全性のために現れる。

---

#### `sock` {#sock}

**何を数えるか**: このcgroup内のプロセスが所有するソケットのために割り当てられたネットワークソケットバッファ（送信バッファと受信バッファ、sk_buff構造体）のメモリのバイト数。

**カーネルのカウンタ**: cgroupのvmstats内の`MEMCG_SOCK`。

**階層的**: はい。

**診断での使い方**: ネットワーク負荷の高いコンテナ（プロキシ、ロードバランサ、gRPCサーバー）で`sock`が高い場合、バッファの圧迫を示す。ソケットバッファが際限なく増えている場合、プロセスが受信バッファを消費しきれていない（遅いコンシューマ）可能性がある。どれだけの割合がネットワークのオーバーヘッドかを把握するには、`memory.current`と突き合わせる。

---

#### `slab_reclaimable` {#slab_reclaimable}

**何を数えるか**: このcgroupに課金された回収可能なカーネルslabメモリのバイト数。主に`dentry`（ディレクトリエントリキャッシュ）と`inode_cache`のオブジェクトで、カーネルはメモリ圧迫下で`shrink_slab()`によってこれらを解放できる。

**カーネルのカウンタ**: cgroupのvmstats内の`NR_SLAB_RECLAIMABLE_B`。

**階層的**: はい。

**診断での使い方**: ファイルシステムを大量に走査するコンテナ（例: `find`、再帰的なビルド、ログスキャナ）では、`slab_reclaimable`が大きく溜まる。これは健全であり、圧迫下で解放される。ただし`memory.high`がきつく設定されていると、このメモリはいつでも解放できるにもかかわらず、この回収可能なslabがスロットリングを引き起こすことがある。

---

#### `slab_unreclaimable` {#slab_unreclaimable}

**何を数えるか**: このcgroupに課金された回収不能なカーネルslabメモリのバイト数。プロセスが明示的に解放するまで解放できないカーネルオブジェクトである（例: `task_struct`、`mm_struct`、オープン中のファイルディスクリプタ、ネットワーク接続、`struct socket`）。

**カーネルのカウンタ**: cgroupのvmstats内の`NR_SLAB_UNRECLAIMABLE_B`。

**階層的**: はい。

**診断での使い方**: 上限のある増加は想定内である。増加し続ける場合は、カーネルレベルのリソースリーク、つまりファイルディスクリプタ、ネットワーク接続、ソケットを開いたまま閉じないプロセスが示唆される。ユーザー空間のリークとは異なり、`slab_unreclaimable`はプロセスのRSSに現れないため、`top`や`ps`での診断が難しい。

---

### スワップ {#swap}

スワップのフィールドは、cgroupがスワップを利用できる場合（`memory.swap.max`がゼロでなく、ホストにスワップが設定されている）にのみ現れる。

---

#### `swapcached` {#swapcached}

**何を数えるか**: スワップとRAMの両方に同時に存在する無名メモリのバイト数（つまり、スワップアウトされた後にフォールトで読み戻されたが、まだスワップデバイスから削除されていないもの）。これらのpageは`anon`と`swapcached`の両方に計上される。

**カーネルのカウンタ**: cgroupのvmstats内の`NR_SWAPCACHE`。

**階層的**: はい。

**診断での使い方**: `swapcached`が非ゼロの場合、cgroupがスワップを能動的に使っていることを意味する。pageがスワップアウトされ、少なくともその一部が読み戻されている。これはメモリ圧迫の兆候であり、cgroupが最近スワップを必要としたことを示す。`swapcached`が大きいまま減らない場合、pageはフォールトで読み戻されているが、cgroupは依然として圧迫下にあることを意味する。

---

### 回収イベント {#reclaim-events}

これらは単調増加するイベントカウンタである。現在の状態を測定するものではなく、cgroupの作成以降に起きた事象を記録する。

---

#### `pgfault` {#pgfault}

**何を数えるか**: このcgroupにおけるマイナーページフォールトの総数。ディスクI/Oなしに解決されたフォールトである（pageはRAMにあったが、マップされていなかっただけ。例: copy-on-write、ページキャッシュにあったpageへのアクセス）。

**カーネルのカウンタ**: `PGFAULT`（`memcg_vm_event_stat[]`内）。

**階層的**: はい。

**診断での使い方**: メモリアクセスパターンの基礎的な指標である。多くのpageにアクセスするワークロードでは、`pgfault`の頻度が非常に高いのは正常である。コード変更の前後で`pgfault`の頻度を比較すれば、リグレッションを検出できる。

---

#### `pgmajfault` {#pgmajfault}

**何を数えるか**: メジャーページフォールトの総数。解決にディスクI/Oを必要としたフォールトである（pageがRAMになく、ディスクまたはスワップから読み込まなければならなかった）。

**カーネルのカウンタ**: `MEMCG_PGMAJFAULT`。

**階層的**: はい。

**診断での使い方**: スワップの圧迫や、ワーキングセットの追い出しによって引き起こされたファイルI/Oの最も直接的な指標である。`pgmajfault`の頻度が持続的に高いコンテナはスラッシングしている。以前に追い出されたpageをディスクから読み続けているのである。`memory.pressure`の`full`値と相関を取ること。

```bash
# pgmajfaultの頻度を計算する（2回サンプリングし、間隔で割る）
S1=$(grep pgmajfault /sys/fs/cgroup/mycontainer/memory.stat | awk '{print $2}')
sleep 10
S2=$(grep pgmajfault /sys/fs/cgroup/mycontainer/memory.stat | awk '{print $2}')
echo "pgmajfault rate: $(( (S2 - S1) / 10 )) faults/sec"
```

---

#### `pgrefill` {#pgrefill}

**何を数えるか**: LRUの補充（refill）中にスキャンされたpage数。回収中にactive LRUリストからinactiveリストへ移されたpageである。移されたpage 1つにつき1イベントと数える。

**カーネルのカウンタ**: `MEMCG_PGREFILL`。

**階層的**: はい。

**診断での使い方**: `pgrefill`が増えているのは、能動的な回収の圧力を示す。カーネルがpageをactiveからinactiveのLRUへ降格させているのである。回収パイプライン全体を把握するには、`pgscan`や`pgsteal`と突き合わせる。

---

#### `pgscan` {#pgscan}

**何を数えるか**: ページ回収のコードが解放するpageを探す間にスキャンしたpage数。直接回収（プロセスの割り当てが引き起こす）とkswapd（バックグラウンド回収デーモン）の両方を含む。

**カーネルのカウンタ**: `MEMCG_PGSCAN`。

**階層的**: はい。

**診断での使い方**: 回収の「作業量」を表すメトリクスである。`pgscan`の頻度が高い場合、回収が激しく動いていることを意味する。`pgscan`と`pgsteal`を比較すると、回収効率の比率が得られる。

---

#### `pgsteal` {#pgsteal}

**何を数えるか**: ページ回収のコードによって実際に回収された（解放またはスワップアウトされた）page数。常に`pgscan`以下である。スキャンされたpageの一部は直ちには回収できない（ダーティ、ロック中、参照されている）からである。

**カーネルのカウンタ**: `MEMCG_PGSTEAL`。

**階層的**: はい。

**診断での使い方**: 回収の実際の成果である。`pgscan`が高いのに`pgsteal`が低い場合、回収は多くのpageをスキャンしているのにわずかしか解放できていない。これはスラッシングの特徴である。`pgsteal / pgscan`の比率は（ヒューリスティックとして）回収効率を示す。圧迫下でこの比率が約50%を下回る場合、ワーキングセットが利用可能なメモリより大きいことが示唆される。

---

#### `pgactivate` {#pgactivate}

**何を数えるか**: 追い出される前に再びアクセスされたために、inactive LRUリストからactive LRUリストへ昇格したpage数。

**カーネルのカウンタ**: `MEMCG_PGACTIVATE`。

**階層的**: はい。

**診断での使い方**: ワーキングセットが保たれていること、つまり頻繁にアクセスされるpageが保護されていることを示す。激しい回収の最中に`pgactivate`が非常に低い場合、ワーキングセットが完全に追い出されてしまったことを示している可能性がある。

---

#### `pgdeactivate` {#pgdeactivate}

**何を数えるか**: active LRUからinactive LRUへ移され、回収の対象となったpage数。`pgactivate`の逆である。

**カーネルのカウンタ**: `MEMCG_PGDEACTIVATE`。

**階層的**: はい。

**診断での使い方**: メモリ圧迫によって、カーネルが以前はactiveだったpageの回収を検討せざるを得なくなっていることを示す。同じpageに対して`pgdeactivate`が持続的に高い頻度で起き、その後`pgsteal`が続く場合、スラッシングを意味する。

---

#### `pglazyfreed` {#pglazyfreed}

**何を数えるか**: 「lazy free」パスで回収された無名page数。終了したプロセスに関連付けられていたため、ライトバックなしに解放されたpageである（pageはスワップにあり、メモリ上のコピーは破棄された）。

**カーネルのカウンタ**: `MEMCG_PGLAZYFREED`。

**階層的**: はい。

**診断での使い方**: 回収における「タダ飯」、つまりI/Oなしに破棄できたpageを数える。単独で重要になることはまれだが、回収の内訳全体を把握するのに役立つ。

---

#### `thp_fault_alloc` {#thp_fault_alloc}

**何を数えるか**: ページフォールト時に成功したTransparent Huge Pageの2 MiB割り当ての回数（つまり、プロセスが新しい領域に触れたときに、無名マッピングに対してhuge pageが割り当てられた）。

**階層的**: はい。

**診断での使い方**: THPが能動的に使われていることを示す。THPの恩恵を受けるはずのワークロードでこれがゼロの場合は、`/sys/kernel/mm/transparent_hugepage/enabled`と断片化を確認する。

---

#### `thp_collapse_alloc` {#thp_collapse_alloc}

**何を数えるか**: このcgroupについて、`khugepaged`が複数の4 KiB pageを1つの2 MiB huge pageへまとめる（collapse）ことに成功した回数。

**階層的**: はい。

**診断での使い方**: バックグラウンドのTHP最適化が機能していることを示す。ゼロの場合は、断片化によってcollapseが妨げられているか、`khugepaged`が無効化されている可能性がある。

---

### メモリ保護 {#memory-protection}

これらのフィールドは、保護の設定と、それが効果を発揮しているかどうかを反映する。

---

#### `workingset_refault_anon` {#workingset_refault_anon}

**何を数えるか**: RAMから追い出された後に、フォールトで再び読み込まれた（refaultした）無名page数。以前にスワップアウトされたpageが再びアクセスされたときに加算される。

**カーネルのカウンタ**: `WORKINGSET_REFAULT_ANON`。

**階層的**: はい。

**診断での使い方**: 無名メモリにおけるスラッシングの決定的な指標である。`workingset_refault_anon`が増えている場合、無名pageが追い出されては再びフォールトしている。つまり、ワーキングセットが利用可能なメモリに収まっていない。`pgmajfault`の高さと組み合わされば、より多くのRAMかより大きなスワップが必要であることが確定する。

---

#### `workingset_refault_file` {#workingset_refault_file}

**何を数えるか**: ページキャッシュから追い出された後に、フォールトで再び読み込まれたファイルバックのpage数。

**カーネルのカウンタ**: `WORKINGSET_REFAULT_FILE`。

**階層的**: はい。

**診断での使い方**: ファイルバックのメモリにおけるスラッシングの指標である。値が高い場合、I/Oのワーキングセット（ディスクから読むデータベース、静的ファイルを読むWebサーバー）がページキャッシュに収まっていないことを意味する。`memory.high`やコンテナのメモリ上限を引き上げてファイルキャッシュにより多くの余地を与えれば、これは減る。

---

#### `workingset_activate_anon` {#workingset_activate_anon}

**何を数えるか**: refaultした無名pageのうち、ワーキングセットの一部と判断されたために、refault後すぐにactive LRUへ昇格したpage数。

**カーネルのカウンタ**: `WORKINGSET_ACTIVATE_ANON`。

**階層的**: はい。

**診断での使い方**: カーネルのワーキングセット検出が無名pageに対して動作していることを示す。すぐにrefaultするpageはホットなワーキングセットにある可能性が高く、`workingset_activate_anon`はそうしたpageを数える。

---

#### `workingset_activate_file` {#workingset_activate_file}

**何を数えるか**: refaultしたファイルバックのpageのうち、refault後にactive LRUへ昇格したpage数。

**カーネルのカウンタ**: `WORKINGSET_ACTIVATE_FILE`。

**階層的**: はい。

**診断での使い方**: ファイルバックのpageについて、上と同じ。

---

#### `workingset_restore_anon` {#workingset_restore_anon}

**何を数えるか**: 追い出しから「復元（restore）」された無名page数。refaultのシャドウエントリが失効する前にactiveリストへ昇格したpageである（つまり、activeと認識されるほど早くフォールトで読み戻された）。

**カーネルのカウンタ**: `WORKINGSET_RESTORE_ANON`。

**階層的**: はい。

---

#### `workingset_restore_file` {#workingset_restore_file}

**何を数えるか**: 復元された（シャドウの失効前にactiveへ昇格した）ファイルバックのpage数。

**カーネルのカウンタ**: `WORKINGSET_RESTORE_FILE`。

**階層的**: はい。

---

#### `workingset_nodereclaim` {#workingset_nodereclaim}

**何を数えるか**: ワーキングセットがactiveと認識されたために、ノードの回収が回避された回数。このカウンタは、refault距離の検出機構が、pageのシャドウからそのpageがactiveなワーキングセットにあると判断したときを記録する。

**カーネルのカウンタ**: `WORKINGSET_NODERECLAIM`。

**階層的**: はい。

**診断での使い方**: 上級者向けの診断である。`workingset_refault_*`に比べて値が低い場合、ワーキングセット検出があまり役立っていないことが示唆される。ワークロードが、カーネルのLRU近似ではうまく扱えないパターン（例: 大規模なシーケンシャルスキャン）でpageにアクセスしている可能性がある。

---

## 計算例: アカウンティングの恒等式 {#worked-example-accounting-identity}

`memory.current`は、常駐しているメモリ種別の合計に等しくなるはずである。[`Documentation/admin-guide/cgroup-v2.rst`](https://docs.kernel.org/admin-guide/cgroup-v2.html)に記載されている正確な関係は次のとおり。

```
memory.current ≈ anon + file + kernel + swapcached + sock + (children's contributions)
```

実際には、per-CPUカウンタのバッチ処理とタイミングのため、ある瞬間の`memory.current`は`memory.stat`のフィールドの合計とわずかに異なることがある。これは想定内である。カーネルは、ホットパスでのキャッシュラインの競合を避けるため、`mod_lruvec_state()`でカウンタの更新をバッチ処理している。

### スナップショットの例 {#example-snapshot}

```bash
$ cat /sys/fs/cgroup/myapp/memory.stat
anon 549453824          # 約524 MiBのヒープ/スタック
file 157286400          # 約150 MiBのページキャッシュ
shmem 0
kernel_stack 2097152    # 2 MiB（256スレッド × 8 KiB）
pagetables 4194304      # 4 MiB
slab_reclaimable 20971520   # 約20 MiB（dentry、inodeキャッシュ）
slab_unreclaimable 8388608  # 約8 MiB（task構造体、file構造体）
sock 10485760           # 約10 MiB（ソケットバッファ）
swapcached 0            # スワップの活動なし
...
workingset_refault_anon 0
workingset_refault_file 123456

$ cat /sys/fs/cgroup/myapp/memory.current
753207296               # 約718 MiB
```

アカウンティング:

| 構成要素 | バイト | MiB |
|-----------|-------|-----|
| `anon` | 549,453,824 | 524 |
| `file` | 157,286,400 | 150 |
| `kernel_stack` | 2,097,152 | 2 |
| `pagetables` | 4,194,304 | 4 |
| `slab_reclaimable` | 20,971,520 | 20 |
| `slab_unreclaimable` | 8,388,608 | 8 |
| `sock` | 10,485,760 | 10 |
| **合計** | **752,877,568** | **718** |
| `memory.current` | 753,207,296 | ~718 |

約330 KiBの差は、`percpu`カウンタとバッチ処理の遅延によるもので、十分に正常の範囲内である。`workingset_refault_file`が123,456イベントであることは、このコンテナがファイルキャッシュの追い出しとrefaultを経験していることを示しており、`memory.high`がI/Oのワーキングセットに対してきつすぎる可能性を示唆している。

### memory.statに含まれないもの {#what-memorystat-does-not-include}

| 項目 | 含まれない理由 |
|------|-----------------|
| スワップ上の（RAMにない）page | `memory.swap.current`が別に追跡する |
| hugetlbのpage | `memory`ではなく`hugetlb`コントローラが管理する |
| メモリマップされたデバイスI/O | memcgでは追跡されない（page構造体による裏付けがない） |
| slab以外のカーネルメモリ | 例: `struct page`による裏付けのない、ioremapされた割り当てや物理的に連続した割り当て |

---

## クイックリファレンス表 {#quick-reference-table}

| フィールド | 種別 | 回収可能か? | カーネルのカウンタ |
|-------|------|-------------|----------------|
| `anon` | 無名 | スワップ経由でのみ | `NR_ANON_MAPPED` |
| `anon_thp` | 無名THP | スワップ経由でのみ | `NR_ANON_THPS` |
| `shmem` | 共有メモリ | はい（スワップへ） | `NR_SHMEM` |
| `shmem_thp` | 共有メモリのTHP | はい（スワップへ） | `NR_SHMEM_THPS` |
| `file` | ページキャッシュ | はい（クリーン: 破棄、ダーティ: ライトバック） | `NR_FILE_PAGES` |
| `file_mapped` | マップされたファイルpage | はい（アンマップ後） | `NR_FILE_MAPPED` |
| `file_dirty` | ダーティなページキャッシュ | ライトバック後 | `NR_FILE_DIRTY` |
| `file_writeback` | ライトバック中 | ライトバック完了後 | `NR_WRITEBACK` |
| `file_thp` | ファイルTHP | はい | `NR_FILE_THPS` |
| `kernel` | カーネルオーバーヘッドの合計 | 部分的に | （導出された合計） |
| `kernel_stack` | スレッドのカーネルスタック | いいえ（終了時に解放） | `NR_KERNEL_STACK_KB` |
| `pagetables` | ページテーブルのpage | いいえ（終了時に解放） | `NR_PAGETABLE` |
| `sec_pagetables` | セカンダリページテーブル | いいえ | `NR_SECONDARY_PAGETABLE` |
| `percpu` | per-CPU割り当て | いいえ | （memcg固有のカウンタ） |
| `sock` | ソケットバッファ | いいえ（close時に解放） | `MEMCG_SOCK` |
| `slab_reclaimable` | 回収可能なslab | はい | `NR_SLAB_RECLAIMABLE_B` |
| `slab_unreclaimable` | 回収不能なslab | いいえ | `NR_SLAB_UNRECLAIMABLE_B` |
| `swapcached` | スワップキャッシュ | はい（スワップ側のコピーを破棄） | `NR_SWAPCACHE` |
| `pgfault` | イベントカウンタ | — | `PGFAULT` |
| `pgmajfault` | イベントカウンタ | — | `PGMAJFAULT` |
| `pgrefill` | イベントカウンタ | — | `PGREFILL` |
| `pgscan` | イベントカウンタ | — | `PGSCAN_KSWAPD` + `PGSCAN_DIRECT` + `PGSCAN_PROACTIVE` + `PGSCAN_KHUGEPAGED`の合計 |
| `pgsteal` | イベントカウンタ | — | `PGSTEAL_KSWAPD` + `PGSTEAL_DIRECT` + ...の合計 |
| `pgactivate` | イベントカウンタ | — | `PGACTIVATE` |
| `pgdeactivate` | イベントカウンタ | — | `PGDEACTIVATE` |
| `pglazyfreed` | イベントカウンタ | — | `PGLAZYFREED` |
| `thp_fault_alloc` | イベントカウンタ | — | （memcgのTHPカウンタ） |
| `thp_collapse_alloc` | イベントカウンタ | — | （memcgのTHPカウンタ） |
| `workingset_refault_anon` | イベントカウンタ | — | `WORKINGSET_REFAULT_ANON` |
| `workingset_refault_file` | イベントカウンタ | — | `WORKINGSET_REFAULT_FILE` |
| `workingset_activate_anon` | イベントカウンタ | — | `WORKINGSET_ACTIVATE_ANON` |
| `workingset_activate_file` | イベントカウンタ | — | `WORKINGSET_ACTIVATE_FILE` |
| `workingset_restore_anon` | イベントカウンタ | — | `WORKINGSET_RESTORE_ANON` |
| `workingset_restore_file` | イベントカウンタ | — | `WORKINGSET_RESTORE_FILE` |
| `workingset_nodereclaim` | イベントカウンタ | — | `WORKINGSET_NODERECLAIM` |

---

## 診断レシピ {#diagnostic-recipes}

### コンテナのメモリ上限はファイルキャッシュに対してきつすぎないか? {#is-the-container-memory-limit-set-too-tight-for-its-file-cache}

```bash
STAT=/sys/fs/cgroup/mycontainer/memory.stat
HIGH=$(cat /sys/fs/cgroup/mycontainer/memory.high)
FILE=$(grep '^file ' $STAT | awk '{print $2}')
REFAULT=$(grep '^workingset_refault_file' $STAT | awk '{print $2}')

echo "File cache: $(( FILE / 1048576 )) MiB"
echo "File refaults (lifetime): $REFAULT"
echo "memory.high: $(( HIGH / 1048576 )) MiB"
# REFAULTが高くFILEがHIGHに近い場合は、memory.highを引き上げてキャッシュに余地を与える
```

### slabの増加がカーネルのリソースリークを隠していないか? {#is-slab-growth-hiding-a-kernel-resource-leak}

```bash
STAT=/sys/fs/cgroup/mycontainer/memory.stat
ANON=$(grep '^anon ' $STAT | awk '{print $2}')
FILE=$(grep '^file ' $STAT | awk '{print $2}')
SLAB_U=$(grep '^slab_unreclaimable' $STAT | awk '{print $2}')
CURRENT=$(cat /sys/fs/cgroup/mycontainer/memory.current)

echo "anon: $(( ANON / 1048576 )) MiB"
echo "file: $(( FILE / 1048576 )) MiB"
echo "slab_unreclaimable: $(( SLAB_U / 1048576 )) MiB"
echo "memory.current: $(( CURRENT / 1048576 )) MiB"
echo "unexplained: $(( (CURRENT - ANON - FILE - SLAB_U) / 1048576 )) MiB"
# 説明のつかない差が大きい、またはslab_unreclaimableが増え続けている = カーネルリークの可能性
```

### コンテナは現在スラッシングしているか? {#is-the-container-actively-thrashing}

```bash
STAT=/sys/fs/cgroup/mycontainer/memory.stat
SCAN=$(grep '^pgscan ' $STAT | awk '{print $2}')
STEAL=$(grep '^pgsteal ' $STAT | awk '{print $2}')
REFAULT_A=$(grep '^workingset_refault_anon' $STAT | awk '{print $2}')
REFAULT_F=$(grep '^workingset_refault_file' $STAT | awk '{print $2}')
MAJFAULT=$(grep '^pgmajfault' $STAT | awk '{print $2}')

echo "pgscan/pgsteal ratio (lifetime): $SCAN / $STEAL"
echo "workingset_refault_anon: $REFAULT_A"
echo "workingset_refault_file: $REFAULT_F"
echo "pgmajfault: $MAJFAULT"
# refaultが多い + pgmajfaultが高い + pgscan >> pgsteal = スラッシング
```

---

## 主要なソースファイル {#key-source-files}

| ファイル | 関連 |
|------|-----------|
| [`mm/memcontrol.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c) | `memory_stat_show()`が`memory.stat`を生成する。`mod_lruvec_state()`がカウンタを更新する |
| [`include/linux/memcontrol.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/memcontrol.h) | `mem_cgroup`構造体、`MEMCG_*`統計のenum、`mem_cgroup_stat_names[]` |
| [`include/linux/mm.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mm.h) | グローバルおよびlruvecごとの統計のための`NR_*` enumの値 |
| [`mm/vmscan.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) | `pgscan`、`pgsteal`、`pgrefill`、`pgactivate`、`pgdeactivate`を更新する |
| [`mm/workingset.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/workingset.c) | `workingset_refault_*`、`workingset_activate_*`を更新する |
| [`Documentation/admin-guide/cgroup-v2.rst`](https://docs.kernel.org/admin-guide/cgroup-v2.html) | フィールドの正式な説明 |

---

## 参考文献 {#further-reading}

- [メモリcgroup](memcg.md) — 何がどのようにcgroupに課金されるか
- [コンテナのチューニング](tuning-containers.md) — memory.statのデータを使ってmemory.highとmemory.maxのサイズを決める
- [/proc/meminfo](proc-meminfo.md) — memory.statのシステム全体版
- [コンテナのメモリ上限](container-memory-limits.md) — memory.statの数値が上限に近づくと何が起こるか
