# バッファードI/OとダイレクトI/O {#buffered-io-vs-direct-io}

> ページキャッシュをバイパスすることが役立つのはどんなときで、害になるのはどんなときか

## 選択肢 {#the-choice}

Linux上のすべての読み書きは、ストレージへの2つの経路のどちらかを通る:

**バッファードI/O**（デフォルト）: データはページキャッシュを通過する。読み出しはまずキャッシュを確認する。書き込みはキャッシュに入り、ライトバックスレッドによって非同期にディスクへフラッシュされる。

**ダイレクトI/O**（`O_DIRECT`）: データはページキャッシュを完全にバイパスする。読み出しはストレージからユーザー空間のバッファへ直接届き、書き込みはユーザー空間のバッファからストレージへ直接送られる。

```
Buffered I/O path:
application → write() → page cache → [background] → storage
application → read() → page cache → (cache miss) → storage → page cache → application

Direct I/O path:
application → write() → DMA → storage
application → read() → DMA → storage → application buffer
```

両者のどちらを選ぶかは、システムプログラミングにおいて最も影響の大きいI/Oの判断の1つである。ワークロードに対して誤った選択をすると、RAMを浪費したり、オーバーヘッドが増えたり、正しさの問題を引き起こしたりする。

---

## ページキャッシュが行うこと {#what-the-page-cache-does}

ページキャッシュは、最近アクセスされたファイルデータのカーネル内キャッシュである。次の4つの目的を果たす:

1. **読み出しのキャッシュ**: キャッシュにヒットしたファイル読み出しは、ストレージアクセスを完全に回避する（ミリ秒ではなくマイクロ秒）。
2. **書き込みのバッファリング**: 書き込みはアプリケーションに即座に戻り、ストレージへのライトバックは非同期に行われる。
3. **先読み**: シーケンシャルなアクセスパターンに対して、カーネルは現在の読み出し位置より先のpageを投機的にプリフェッチする。
4. **共有**: 同じファイルを読む複数のプロセスは、同じキャッシュ済みpageを共有する（プロセスごとの重複コピーがない）。

```
Page cache as a shared read cache:

Process A        Process B        Process C
read("foo.dat")  read("foo.dat")  read("foo.dat")
     |                |                |
     └────────────────┴────────────────┘
                      |
              [single copy in page cache]
                      |
                  storage
```

これら4つの性質により、ページキャッシュは多くのワークロードにとって価値がある — そしてほかのワークロードにとっては無関係か、有害である。

---

## バッファードI/Oが有利な場合 {#when-buffered-io-wins}

### 同じデータを繰り返し読む {#repeated-reads-of-the-same-data}

ページキャッシュは、ストレージアクセスのコストを多数の読み出しで償却する。ファイルが複数のプロセスに読まれる場合や、同じファイル領域が繰り返し読まれる場合、キャッシュはストレージに触れずにデータを届ける。

```
Cache hit: application → page cache → return in ~1µs
Cache miss: application → page cache → storage → page cache → return in ~100µs
                                                                (NVMe) or ~10ms (SSD)

10× repeated reads: cache saves 9 storage accesses
100× repeated reads: cache saves 99 storage accesses
```

### デバイスのスループットを超える書き込みバースト {#write-bursts-that-exceed-device-throughput}

バッファード書き込みはアプリケーションに即座に戻る。ストレージデバイスがバーストのレートに追いつけなくても、ページキャッシュが書き込みバーストを吸収する。ライトバックサブシステムは、バーストをならしてデバイスの能力に応じた持続的な書き込みにする。

```
Application write rate: 2 GB/s (in-memory speed)
Device write rate: 500 MB/s

Buffered I/O: application writes at 2GB/s to page cache → writeback at 500MB/s
Direct I/O: application blocks until device absorbs each write at 500MB/s
```

データをバースト的に生成するワークロード（ログライター、イベントストリーム、チェックポイント）では、バッファード書き込みによってwrite()のレイテンシが下がり、アプリケーションのスループットが向上する。

### 先読みを伴うシーケンシャル読み出し {#sequential-reads-with-readahead}

カーネルの先読みアルゴリズムは、シーケンシャルなアクセスパターンを検出して現在の位置より先をプリフェッチする。大きなファイルをシーケンシャルに読むプロセスに対して、先読みはデバイスを忙しく保ち、要求される前にデータを準備しておくことで、ストレージのレイテンシを隠す。

```bash
# 先読みの効果を確認
grep -E '(pgpgin|pgpgout)' /proc/vmstat  # ページインのレート
cat /sys/block/sda/queue/read_ahead_kb   # 現在の先読みウィンドウ
```

バッファードなシーケンシャル読み出しは、アプリケーションが明示的に並列性を管理しなくても、デバイスを飽和させるスループットを達成できる。先読みがそれを担うからである。

### 小さなファイル、多数のファイル {#small-files-many-files}

多数の小さなファイルを開いて読むのは、バッファードI/Oの方が効率的である。小さなファイルは最初のアクセスの後、丸ごとページキャッシュに収まることがある。以降のアクセス（たとえばリクエストごとに設定ファイルを読み直す）はメモリから応答される。

O_DIRECTでは、キャッシュの状態にかかわらず、すべての読み出しがストレージへ行く。

---

## ダイレクトI/Oが有利な場合 {#when-direct-io-wins}

### アプリケーションが独自のキャッシュを持つ {#application-has-its-own-cache}

データベース（PostgreSQLのshared buffers、MySQL InnoDBのバッファプール、OracleのSGA）は、独自のアプリケーションレベルのキャッシュを持っており、それぞれのアクセスパターンに対してはページキャッシュより賢い。どのpageがホットかを知っており、追い出しを明示的に管理し、起動時にキャッシュを事前にウォームアップできる。

こうしたアプリケーションにとって、ページキャッシュは同じデータの2つ目のコピーである:

```
Without O_DIRECT (wasteful):
┌─────────────┐    ┌─────────────┐    ┌──────────┐
│  App cache  │ ←→ │ Page cache  │ ←→ │ Storage  │
│  (64 GB)    │    │  (64 GB)    │    │          │
└─────────────┘    └─────────────┘    └──────────┘
  Two copies of the same 64GB of database pages

With O_DIRECT:
┌─────────────┐                       ┌──────────┐
│  App cache  │ ←────────────────────→│ Storage  │
│  (64 GB)    │                        │          │
└─────────────┘                        └──────────┘
  One copy; the other 64GB of RAM is available for other uses
```

O_DIRECTを使えば、アプリケーションはアクセスパターンを完全に把握したうえでキャッシュを管理でき、同じデータのためのカーネルキャッシュにRAMを浪費しない。

### ストリーミング的な一度きりの読み出し {#streaming-single-pass-reads}

データを一度読むだけで二度と見返さないバックアッププロセス、動画トランスコーダ、ETLパイプラインは、キャッシュから何の恩恵も受けない — それどころか、ほかのプロセスのワーキングセットをページキャッシュから追い出すという害をもたらす。

```bash
# O_DIRECT なし: 100GB のバックアップがページキャッシュを埋め、ワーキングセットを追い出す
# バックアップ前のページキャッシュ: [database pages] [web server pages] [config files]
# バックアップ中のページキャッシュ: [backup data] [backup data] [backup data]
# バックアップ後のページキャッシュ: [backup data] [backup data] [backup data]  ← ほかはすべて追い出された!

# O_DIRECT あり: バックアップの読み出しはディスクへ直接行き、ページキャッシュは変わらない
# バックアップ後のページキャッシュ: [database pages] [web server pages] [config files]  ← 保持される
```

`POSIX_FADV_DONTNEED`は中間的な選択肢を提供する。バッファードI/Oを使い（先読みの恩恵を受け）つつ、処理済みのpageを破棄するようカーネルに助言する。

### 予測可能な書き込みレイテンシが求められる場合 {#predictable-write-latency-requirements}

バッファード書き込みは即座に戻るが、実際のストレージへの書き込みは将来の不明な時点で行われる。アプリケーションがデータがいつストレージに載ったかを知る必要がある場合（クラッシュ安全性や永続性の保証のため）、`fsync()`を呼ばなければならない — これはすべてのダーティpageがフラッシュされるまでブロックする。

`O_SYNC`付きのO_DIRECT（または各書き込み後に明示的に`fdatasync()`を呼ぶO_DIRECT）は同期書き込みを提供する。データがストレージに載るまでwrite()呼び出しは戻らない。これにより書き込みレイテンシが決定的になる。

```c
/* 永続性の保証が分かっている同期書き込み */
int fd = open(path, O_WRONLY | O_DIRECT | O_SYNC);
ssize_t n = write(fd, buf, len);
/* この時点で、データはストレージ上に永続化されている */

/* 永続性が不明なバッファード書き込み */
int fd = open(path, O_WRONLY);
write(fd, buf, len);
/* データはページキャッシュにある。ライトバック次第で 0ms〜30s 後にストレージに載るかもしれない。 */
fdatasync(fd);  /* これでストレージに載った。ただしこれは 2 回目の呼び出しである。 */
```

---

## ダイレクトI/Oの制約 {#constraints-of-direct-io}

O_DIRECTには、バッファードI/Oにはない要件がある:

**アライメント**: バッファアドレス、ファイルオフセット、転送長はすべて、デバイスの論理ブロックサイズ（通常512または4096バイト）にアラインされていなければならない。

```c
/* O_DIRECT のアライメント要件 */
void *buf;
posix_memalign(&buf, 4096, IO_SIZE);  /* 4096 バイトアラインのバッファ */

/* ファイルオフセットもアラインされていなければならない */
lseek(fd, 4096 * page_number, SEEK_SET);  /* アラインされたオフセット */

/* 転送サイズもアラインされていなければならない */
read(fd, buf, 4096 * n_pages);  /* アラインされたサイズ */
```

これらの要件のいずれかに違反すると、`read()`や`write()`は`EINVAL`を返す。

**先読みなし**: O_DIRECTの読み出しは同期的であり、先読みを引き起こさない。各読み出しは要求されたバイトだけをストレージから取得する。

**書き込みバッファリングなし**: O_DIRECTの書き込みはデフォルトで同期的である。書き込みスループットはRAMではなくデバイスのスループットで制限される。

**混在モードでの一貫性**: 同じファイルに対してO_DIRECTとバッファードI/Oを混在させると、古いデータを読むことがある。事件簿: データ損失を参照。

---

## 定量的な比較 {#quantitative-comparison}

| シナリオ | バッファードI/O | ダイレクトI/O |
|---|---|---|
| 1MBファイルのコールドリード | 約100µs（NVMe） | 約100µs（NVMe） |
| 1MBファイルのウォームリード（キャッシュ済み） | 約20µs | 約100µs（常にデバイスへ行く） |
| 10GBのシーケンシャル読み出し | ウォームアップ後はデバイスの速度 | デバイスの速度（先読みの恩恵なし） |
| 1MBの書き込み、戻りのレイテンシ | 約1µs（キャッシュへの書き込み） | 約100µs（デバイスを待つ） |
| 1MBの書き込み、永続化までのレイテンシ | 約100µs（fsync後） | 約100µs（同期） |
| 64GBのデータベース、64GBのRAM | キャッシュによりメモリ使用量が倍になる | 64GB全体をDBバッファに使える |
| 100GBの一度きりのバックアップ | 100GB分のページキャッシュを汚染する | キャッシュへの影響なし |

---

## 判断ガイド {#decision-guide}

```
Does the application manage its own cache? (database, key-value store)
    YES → O_DIRECT
    NO  → continue

Is the data accessed more than once?
    NO  → O_DIRECT or POSIX_FADV_DONTNEED
    YES → Buffered (cache amortizes storage cost)

Is the access pattern sequential?
    YES → Buffered (readahead helps significantly)
    NO  → Both are similar; O_DIRECT avoids cache pollution

Is write latency predictable important?
    YES → O_DIRECT | O_SYNC, or O_DIRECT + explicit fdatasync
    NO  → Buffered (lower write latency for the application)

Is there a risk of mixing O_DIRECT and buffered I/O on the same file?
    YES → Commit to one mode; do not mix
```

---

## 中間的な選択肢としての`posix_fadvise` {#using-posix_fadvise-as-a-middle-ground}

ページキャッシュの一部の機能（先読み）からは恩恵を受けるが、ほかの機能（長期的なキャッシュ）は必要としないワークロードでは、`posix_fadvise`を使うと`O_DIRECT`のアライメント要件なしにきめ細かく制御できる:

```c
/* シーケンシャルスキャン: 積極的な先読みを有効化 */
posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL);

/* ランダムアクセス: このファイルの先読みを無効化 */
posix_fadvise(fd, 0, 0, POSIX_FADV_RANDOM);

/* 必要になる前に page をキャッシュへ読み込んでおく */
posix_fadvise(fd, offset, length, POSIX_FADV_WILLNEED);

/* 使用後にキャッシュを解放（ストリーミング的な一度きりの読み出し） */
posix_fadvise(fd, processed_offset, processed_length, POSIX_FADV_DONTNEED);
```

`POSIX_FADV_DONTNEED`はストリーミングワークロードで特に有用である。バッファードI/Oを使って先読みの恩恵を受けつつ、処理済みのpageを解放してキャッシュ汚染を防ぐ。

---

## 関連ページ {#related-pages}

- [バッファードI/Oとページキャッシュ](buffered-io.md) — ページキャッシュの内部
- [Direct I/O](direct-io.md) — O_DIRECTのカーネルパス
- [先読み](readahead.md) — 先読みアルゴリズム
- [ストリーミングワークロード向けのI/Oチューニング](tuning-streaming.md) — 一度きりの読み出しの戦略
