# iomap: エクステントベースのI/Oフレームワーク {#iomap-the-extent-based-io-framework}

> ファイルオフセットをデバイスブロックへマッピングし、その上でバッファードI/O、ダイレクトI/O、DAX I/Oを駆動する、ファイルシステム非依存の現代的なレイヤ — それが何であり、何を置き換え、なぜ生まれたのか

Linuxの歴史の大半において、`read()`や`write()`をブロックI/Oに変換する汎用コードは**`buffer_head`**の上に構築されていた。これはファイルシステムブロック1つにつき1つの小さな記述子である。iomapはそれを置き換えたフレームワークであり、現在ではXFS、gfs2、zonefs、そしてext4とbtrfsのダイレクトI/Oパス（およびext4のDAXパス）の下で共有されるI/Oエンジンとなっている。iomapを理解することは、現代のLinuxファイルシステムが実際にどのようにデータを動かしているかを理解することである。

## `buffer_head`の問題点 {#the-problem-with-buffer_head}

`buffer_head`は**単一の**ファイルシステムブロック（通常4 KiB）のマッピングと状態を記述する。この粒度こそが問題のすべてである。

- **ブロックごとに1つの構造体。** 1 MiBのバッファードライトは約256個の`buffer_head`を割り当ててつなぎ合わせ、それぞれを個別にマッピングし、追跡し、破棄する。論理的には1つの連続した操作にすぎないものに対して、現実のCPUとメモリのオーバーヘッドが生じる。
- **エクステントという概念がない。** 現代のファイルシステム（XFS、ext4、btrfs）はファイルを**エクステント**として格納する — 「この40,000ブロックはディスク上で連続している」というものだ。`buffer_head`モデルはそれを表現できず、マッピングを1ブロックずつ導出し直す。
- **絡み合った状態。** `buffer_head`はブロックの*マッピング*と、pageのダーティ／ライトバック状態やロックとを混在させており、汎用パスを理解しにくく、並行化しにくいものにしていた。

## 中核となる考え方: ブロックではなくマッピングを記述する {#the-core-idea-describe-a-mapping-not-a-block}

iomapの中心となる構造体`struct iomap`は、ファイル範囲からデバイス範囲への任意の大きさの**連続したマッピング**を単一のオブジェクトで記述する — オフセット、長さ、デバイスブロックアドレス、バッキングデバイス、そして**タイプ**である。

| タイプ | 意味 |
|---|---|
| `IOMAP_MAPPED` | 既知のアドレスにある、ディスク上の実際に割り当て済みのブロック |
| `IOMAP_HOLE` | スパースなホール — 読み出しはゼロを返し、ブロックは割り当てられていない |
| `IOMAP_UNWRITTEN` | ブロックは割り当て済みだがまだ初期化されていない（事前割り当て）。読み出しはゼロを返す |
| `IOMAP_DELALLOC` | 遅延割り当て — メモリ上で予約済みだが、物理ブロックはまだ選ばれていない |
| `IOMAP_INLINE` | データがinode自体の中にインラインで格納されている |

1つの`struct iomap`がメガバイト単位をカバーできるため、汎用コードはファイルシステムにマッピングを*一度*尋ねるだけで、範囲全体を処理できる。

## 仕組み: `iomap_begin`と反復ループ {#how-it-works-iomap_begin-and-the-iteration-loop}

要点は役割分担にある。ファイルシステムが`struct iomap_ops`で提供するコールバックはたった2つである。

- **`->iomap_begin(inode, pos, length, ...)`** — 「このファイル範囲について`struct iomap`を埋めよ」。これが*唯一の*ファイルシステム固有のロジックである。エクステントツリーを参照し、必要なら割り当てを行い、マッピングを返す。
- **`->iomap_end()`** — 範囲の処理後に行う、任意のクリーンアップ／コミット処理。

それ以外はすべて汎用の`fs/iomap/`コードにある。操作は**`iomap_iter`**ループを実行する。現在位置について`->iomap_begin`を呼び出し、マッピングを受け取り、そのマッピング全体にわたって実際の作業（ページキャッシュとのコピー、`bio`の構築と発行）を行い、位置を進め、リクエスト全体がカバーされるまで繰り返す。ファイルシステムはページキャッシュや[ブロックレイヤ](../block/bio-request.md)に直接触れることはない — 「このファイルオフセットはどこにあるのか」に答えるだけである。

このイテレータの上に、iomapは高水準操作一式を提供する。そのためファイルシステムは`iomap_begin`を一度実装するだけで、それらすべてを手に入れられる。

- **バッファードI/O** — `iomap_file_buffered_write()`とバッファードリードパス。folioネイティブである。
- **ダイレクトI/O** — `iomap_dio_rw()`。共有のO_DIRECTエンジンである。
- **ライトバック** — ダーティなfolioから、そのマッピングに基づいて`bio`を構築する。
- **DAX** — ページキャッシュをバイパスする、永続メモリへのダイレクトアクセスI/O。
- **`FIEMAP`、`SEEK_HOLE`/`SEEK_DATA`、スワップファイルの有効化** — いずれも同じ「fsにマッピングを尋ねる」モデルから自然に導かれる。

## 歴史と採用 {#history-and-adoption}

iomapは2016年にChristoph Hellwigによって導入された（[`ae259a9c8593`](https://git.kernel.org/linus/ae259a9c8593) "fs: introduce iomap infrastructure"、Linux 4.8）。当初の目的は、XFSにクリーンな`SEEK_HOLE`/`FIEMAP`およびダイレクトI/Oパスを与えることだった。十分に汎用的であることが判明したため、コードは後に独自のサブシステムディレクトリに分離され（[`1c230208f53d`](https://git.kernel.org/linus/1c230208f53d) "iomap: start moving code to fs/iomap/"）、各ファイルシステムが自らのパスをその上へ移行していった。

- **XFS** — 最初の利用者。すべてのI/Oパスがiomap上で動作する。
- **gfs2、zonefs、erofs** — ブロックマッピングとI/Oにiomapを採用した。
- **ext4** — DAXとダイレクトI/Oにiomapを使用する。
- **btrfs** — ダイレクトI/Oパスをiomap上へ移行した。

iomapは**早期にfolioへ変換された**。そして決定的に重要なのは、その汎用パスがファイルシステムごとの`buffer_head`コードの背後に重複して存在するのではなく、すでに`fs/iomap/`に集約されていたことである。カーネルのfolio変換がiomapベースのファイルシステムから先に進んだのはこのためである。変換すべき場所はファイルシステムごとではなく、1か所だけだった。

## 参考文献 {#further-reading}

- [Kernel docs: iomap design](https://docs.kernel.org/filesystems/iomap/design.html) — データモデルと動作原理
- [Kernel docs: iomap operations](https://docs.kernel.org/filesystems/iomap/operations.html) — バッファード、ダイレクト、DAX、ライトバックの各API
- [bioとrequest構造体](../block/bio-request.md) — iomapが構築して発行するブロックレイヤのオブジェクト
- [ext4](ext4.md) · [XFS](xfs.md) · [btrfs](btrfs.md) — I/Oパスがiomap上で動作するファイルシステム
