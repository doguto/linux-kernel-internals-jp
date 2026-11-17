# I/Oパターン {#io-patterns}

> Linuxがユーザー空間、ページキャッシュ、ストレージの間でデータをどう移動させるか — そして選んだインターフェースがレイテンシから正しさまであらゆるものを左右するのはなぜか

## はじめに {#getting-started}

このドキュメントは、LinuxがI/Oをどう扱うかを説明する — システムコールAPIだけでなく、その下にある実装上の判断についてもである。各段階でデータがどこにあるのか、どんなコピーが起きる（あるいは起きない）のか、カーネルが何を先送りするのか、そして各インターフェースがアプリケーション開発者にどんなトレードオフを強いるのか。

### 前提知識 {#prerequisites}

このドキュメントは、次の事項に慣れていることを前提とする:

- **Cプログラミング** — カーネルはCで書かれており、syscallの挙動はCのレベルで理解するのが最もよい
- **OSの基本概念** — プロセス、カーネル空間とユーザー空間、システムコール、ファイルディスクリプタ
- **ファイルディスクリプタ** — それが何であるか、`open()` / `close()` / `dup()`がどう動作するか、そしてそれがカーネル側の`struct file`オブジェクトを参照していること

Cから`read()`や`write()`を使ったことがあり、その内部で何が起きているのか疑問に思ったことがあるなら、ここが適切な場所である。仮想メモリ（page、ページキャッシュ）について多少の知識があると役立つが、必須ではない — 関連する概念は必要なところで紹介する。推奨する教科書については[参考文献](#further-reading)を参照。

### カーネルソースの入手 {#getting-the-kernel-source}

```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
```

I/Oに最も関係するディレクトリは`mm/`（ページキャッシュ、先読み、ライトバック）と`fs/`（syscallのエントリポイント、splice、fallocate）である。ビルドと設定については[Documentation/admin-guide/README.rst](https://docs.kernel.org/admin-guide/README.html)を参照。

### テストの実行 {#running-tests}

マシンを再起動せずにI/Oの挙動を試す最も手早い方法:

```bash
# virtme-ng をインストール（ホストのファイルシステムを使って QEMU 上でカーネルを実行する）
pip install virtme-ng

# テストモジュール付きでビルドして起動
virtme-ng --kdir . --append 'test_module.run_tests=1'
```

現在のカーネルで実行時にI/Oをトレースするには、`blktrace`、`bpftrace`、`ftrace`が主なツールである — 使い方は[observability.md](observability.md)を参照。

### 推奨する読む順序 {#suggested-reading-order}

**基礎 — ここから始める:**

1. **[バッファードI/O](buffered-io.md)** — `read()`と`write()`がページキャッシュとどう関わるか。「バッファード」が実際に何を意味するか
2. **[Direct I/O](direct-io.md)** — `O_DIRECT`、アライメント要件、ページキャッシュをバイパスすべきときと、すべきでないとき
3. **[先読み](readahead.md)** — カーネルが要求される前にpageを投機的に取得する仕組みと、その制御方法
4. **[ページキャッシュとライトバック](page-cache-writeback.md)** — ダーティpage、フラッシャースレッド、`fsync()`、そして`write()`の永続性保証（あるいはその欠如）

**メモリマップドI/O:**

5. **[mmap I/O](mmap-io.md)** — ファイルをアドレス空間にマップする。I/Oの仕組みとしてのページフォールト。mmapが有利な場合とそうでない場合

**ゼロコピーとベクタI/O:**

6. **[spliceとsendfile](splice-sendfile.md)** — ユーザー空間へのコピーなしにファイルディスクリプタ間でデータを移動する。ここでの「ゼロコピー」が実際に何を意味するか
7. **[ベクタI/O](vectored-io.md)** — `readv()` / `writev()`、スキャッタ・ギャザー、`iov_iter`の内部

**エンドツーエンドのウォークスルー:**

8. **[writeの一生](life-of-a-write.md)** — `write()`呼び出しをユーザー空間からページキャッシュを経てブロック層までトレースする
9. **[非同期I/O](async-io.md)** — POSIX AIO（偽物）、Linux AIO（`io_submit`）、io_uring（真の非同期）。なぜその歴史が重要なのか

**可観測性とチューニング:**

10. **[可観測性](observability.md)** — I/Oを計測するためのツール: `iostat`、`blktrace`、`bpftrace`、`perf`、`/proc/diskstats`
11. **[ストレージI/Oのチューニング](tuning-storage.md)** — スケジューラの選択、先読みサイズ、ダーティ比率のつまみ、`fadvise()`、`madvise()`

**事件簿:**

12. **[事件簿](war-stories.md)** — I/Oサブシステムのエッジケースに起因する実際のバグ、驚くべき挙動、本番環境のインシデント

### 学べること {#what-youll-learn}

| 教科書の概念 | Linuxの現実 |
|------------------|---------------|
| 「`write()`はデータをディスクに保存する」 | [いいえ — ページキャッシュに書き込む](page-cache-writeback.md)。カーネルは数秒間ストレージへフラッシュしないことがある。永続性が必要なら`fsync()`を呼ぶこと |
| 「`O_DIRECT`の方が速い」 | [完全にワークロード次第](direct-io.md)。アライメント要件は厳格で、先読みは無効になり、時間的局所性のあるワークロードでは通常ページキャッシュの方が有利である |
| 「`sendfile`はゼロコピーである」 | [ユーザー空間へのコピーはゼロ、それは正しい](splice-sendfile.md)が、CPUは依然としてメタデータとソケットバッファに触れる。真のゼロコピー（NICへのDMA）には追加のハードウェアサポートが必要である |
| 「`mmap`は常に`read()`より速い」 | [シーケンシャルなストリーミングでは違う](mmap-io.md)。ページフォールトのオーバーヘッドとTLBへの負荷により`read()`も十分に競争力がある。mmapが有利なのはランダムアクセスと共有データの場合である |
| 「AIOは非同期である」 | [どのAIOを指すかによる](async-io.md)。POSIX AIO（`aio_read`）はglibcでスレッドを使って実装されている。Linux AIO（`io_submit`）は真のカーネル内非同期だが`O_DIRECT`に限られる。io_uringは初めての真に汎用的な非同期I/Oの仕組みである |
| 「`splice`はコピーを回避する」 | [変換が不要な場合に限る](splice-sendfile.md)。spliceはカーネルの構造体の間でpageへの参照を移動させる — データに触れた瞬間にコピーが発生する |

### このドキュメントの特徴 {#what-makes-this-different}

- **実際のコミット**: すべての主張は実際のカーネルコミットにリンクしている
- **実際のバグ**: 本番環境で何が壊れ、なぜ壊れたのかから学ぶ
- **実際の議論**: 設計判断が議論されたLKMLへのリンク
- **実際のトレードオフ**: 「何を」だけでなく「なぜこのインターフェースで、あのインターフェースではないのか」
- **ハンズオン**: 実行するコマンドと読むべきコードを示す「自分で試してみよう」セクション

---

## ドキュメント {#documentation}

### 基礎 {#fundamentals}

| ドキュメント | 学べること |
|----------|-------------------|
| [buffered-io](buffered-io.md) | `read()`と`write()`がページキャッシュを通じてデータをどう移動させるか。キャッシュのヒット時とミス時に何が起きるか |
| [readahead](readahead.md) | カーネルの投機的プリフェッチのロジック。`POSIX_FADV_SEQUENTIAL`、`posix_fadvise()`、手動の先読み |
| [page-cache-writeback](page-cache-writeback.md) | ダーティpageの集計、フラッシャースレッド（`wb_workfn`）、ダーティスロットリング、そして`fsync()`が実際に何をフラッシュするか |
| [direct-io](direct-io.md) | カーネル内の`O_DIRECT`の経路、アライメント制約、DIOとバッファードI/Oの性能上のトレードオフ |

### メモリマップドI/O {#memory-mapped-io}

| ドキュメント | 学べること |
|----------|-------------------|
| [mmap-io](mmap-io.md) | ファイルバックの`mmap`、ページフォールトの処理、`MAP_SHARED`と`MAP_PRIVATE`、`msync()`、そしてmmapが害になる場合 |

### ゼロコピーとベクタI/O {#zero-copy-and-vectored}

| ドキュメント | 学べること |
|----------|-------------------|
| [splice-sendfile](splice-sendfile.md) | `splice()`、`sendfile()`、`tee()`。カーネルのゼロコピープリミティブとしてのパイプバッファ。制限事項 |
| [vectored-io](vectored-io.md) | `readv()` / `writev()`、スキャッタ・ギャザーDMA、カーネルの統一バッファ抽象としての`iov_iter` |

### 割り当てと領域管理 {#allocation-and-space-management}

| ドキュメント | 学べること |
|----------|-------------------|
| [fallocate](fallocate.md) | データを書かずに領域を事前割り当てする。`FALLOC_FL_PUNCH_HOLE`、`FALLOC_FL_KEEP_SIZE`、スパースファイル |

### エンドツーエンドのウォークスルー {#end-to-end-walkthroughs}

| ドキュメント | 学べること |
|----------|-------------------|
| [life-of-a-write](life-of-a-write.md) | `write()`をsyscallのエントリからVFS、アドレス空間、ブロック層まで追い、戻ってくるまでをトレースする |
| [async-io](async-io.md) | POSIX AIOの実装、Linux AIO（`io_submit`）の制約、io_uringの発行キューと完了リング |

### 可観測性とチューニング {#observability-and-tuning}

| ドキュメント | 学べること |
|----------|-------------------|
| [observability](observability.md) | `/proc/diskstats`、`iostat`の読み方、`blktrace`のイベントの構造、I/Oトレーシングのための`bpftrace`ワンライナー |
| [tuning-storage](tuning-storage.md) | ブロックスケジューラの選択、`nr_requests`、先読みのチューニング、ダーティ比率のつまみ、`fadvise` / `madvise`のヒント |

### 事件簿 {#war-stories}

| ドキュメント | 学べること |
|----------|-------------------|
| [war-stories](war-stories.md) | 本番環境のインシデントと驚くべき挙動: 書き込み順序のバグ、`fsync`の抜け、mmap + truncateの競合 |

---

## 主要なソースファイル {#key-source-files}

実際のカーネルコードを読みたい場合:

| ファイル | 役割 |
|------|--------------|
| [`mm/filemap.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/filemap.c) | ページキャッシュのコア実装 — `generic_file_read_iter()`、`generic_file_write_iter()`、pageの検索と挿入 |
| [`mm/readahead.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/readahead.c) | 先読みのステートマシン、`page_cache_async_readahead()`、`force_page_cache_readahead()` |
| [`mm/page-writeback.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page-writeback.c) | ダーティpageの集計、ダーティスロットリング（`balance_dirty_pages()`）、ライトバックの制御 |
| [`fs/fs-writeback.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/fs-writeback.c) | inodeのライトバック、`wb_workfn()`フラッシャースレッド、`sync_inodes_sb()`、ライトバック帯域幅の推定 |
| [`fs/read_write.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/read_write.c) | `read()`、`write()`、`pread()`、`pwrite()`のsyscallエントリポイント。`vfs_read()` / `vfs_write()` |
| [`fs/splice.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/splice.c) | `splice()`、`sendfile()`、`tee()` — ファイルディスクリプタ間のパイプバッファの移動 |
| [`fs/fallocate.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/fallocate.c) | `fallocate()` syscallのディスパッチ。モードフラグの検証。ファイルシステムへの委譲 |
| [`lib/iov_iter.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/iov_iter.c) | `iovec`、`kvec`、`bio_vec`、パイプバッファに対する`iov_iter`抽象。ユーザーとの間のコピー |
| [`include/linux/fs.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/fs.h) | `struct file`、`struct inode`、`struct address_space`、`struct file_operations` — VFSの中心的な型 |

---

## 参考文献 {#further-reading}

### ブログと記事 {#blogs-and-articles}

- Jens Axboeのブログ — [https://axboe.dk/](https://axboe.dk/) — Axboeはio_uringとブロック層の作者であり、その投稿は設計判断をソースに即して直接説明している
- LWN.netのI/O関連記事:
    - [Clarifying memory management with page folios](https://lwn.net/Articles/849538/) — folioへの変換と、生のpageポインタからの脱却
    - [Ringing in a new asynchronous I/O API](https://lwn.net/Articles/776703/) — 最初のio_uringの紹介
    - [Buffered I/O without page-cache thrashing](https://lwn.net/Articles/806980/) — ページキャッシュを汚染しないバッファードI/Oモードである`RWF_UNCACHED`
    - [No-I/O dirty throttling](https://lwn.net/Articles/456904/) — ダーティスロットリングがどのように再設計されたか

### カーネルドキュメント {#kernel-documentation}

- [Documentation/filesystems/vfs.rst](https://docs.kernel.org/filesystems/vfs.html) — VFS層の内部、`file_operations`、`address_space_operations`
- [Documentation/filesystems/locking.rst](https://docs.kernel.org/filesystems/locking.html) — ファイルシステムコードのロック規則
- [Documentation/block/](https://docs.kernel.org/block/) — ブロック層、I/Oスケジューラ、リクエストキュー
- [Documentation/admin-guide/mm/](https://docs.kernel.org/admin-guide/mm/) — `vm.dirty_ratio`、`vm.dirty_background_ratio`、その他のチューニング可能なパラメータ

### 教科書 {#textbooks}

- Silberschatz et al., [*Operating System Concepts*](https://www.os-book.com/) — 第13〜14章（Storage and I/O Systems）
- Tanenbaum & Bos, [*Modern Operating Systems*](https://www.pearson.com/en-us/subject-catalog/p/modern-operating-systems/P200000003295) — 第5章（Input/Output）
- Love, [*Linux Kernel Development*](https://www.oreilly.com/library/view/linux-kernel-development/9780768696974/) — 第13章（The Virtual Filesystem）、第15章（The Block I/O Layer）
- Kerrisk, [*The Linux Programming Interface*](https://man7.org/tlpi/) — 第13章（File I/O Buffering）、第49章（Memory Mappings）

### 論文 {#papers}

- Axboe, [*Efficient IO with io_uring*](https://kernel.dk/io_uring.pdf) — 作者自身によるio_uringの設計文書
- Pai et al., [*Flash: An Efficient and Portable Web Server*](https://www.usenix.org/legacy/events/usenix99/full_papers/pai/pai.pdf) — USENIX 1999。実際のワークロードにおけるPOSIX AIOの限界を明らかにした論文
