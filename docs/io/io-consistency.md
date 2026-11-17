# I/Oの一貫性と順序付け {#io-consistency-and-ordering}

> Linuxは読み書きに対してどのような順序保証を提供するのか、そしてより強い保証を得るにはどうすればよいか

## 「ディスクに書き込まれた」ことの問題 {#the-problem-with-written-to-disk}

`write()`システムコールが戻った時点で、データはページキャッシュにある。ストレージに届くのは数秒後かもしれないし、電源障害の前には届かないかもしれない。「ディスクに書き込まれた」が具体的な意味を持つのは、それが強制された場合だけである — そしてデフォルトの経路は、最終的なライトバック以上のことを何も強制しない。

LinuxのI/Oにおける一貫性と順序付けには4つの異なるレベルがあり、それぞれ高いコストと引き換えにより強い保証を提供する:

```
Weakest                                                    Strongest
────────────────────────────────────────────────────────────────────
buffered    O_SYNC /    fsync /       barrier         power-fail safe
write()     fdatasync  fdatasync +    writes          with O_DIRECT +
            per-write  dir fsync                      fdatasync + dir
                                                      fsync
```

---

## レベル1: バッファード書き込み — 順序保証なし、永続性なし {#level-1-buffered-write-no-ordering-no-durability}

これがデフォルトである。`write()`はデータをページキャッシュにコピーして戻る。カーネルは、ダーティのしきい値とライトバックのタイムアウトに従って、将来のある時点でそれをストレージへフラッシュする。

```c
int fd = open("file", O_WRONLY);
write(fd, data, len);
close(fd);
/* データがストレージに届くまで 0ms〜30s（dirty_expire_centisecs = 3000 = デフォルト 30s） */
/* ライトバック前にクラッシュするとすべての書き込みが失われる */
```

**他プロセスからの可視性**: バッファードI/Oを使う並行した読み出し側は、更新されたデータを（ページキャッシュから）即座に見ることができる。これはバッファードI/Oサブシステム内では一貫している — すべてのバッファードな読み出し側と書き込み側は同じページキャッシュを共有している。

**書き込み間の順序**: 同じプロセスの同じスレッドからのバッファード書き込みは順序付けられている — 書き込みAの後に発行された書き込みBは、バッファードな読み出し側からは書き込みAの後に見える。しかし、ストレージへのライトバックでは書き込みの順序が入れ替わることがある。書き込みAが書き込みBより先にストレージに届くとは限らない。

---

## レベル2: `O_SYNC`と`fdatasync` — 書き込みごとの永続性 {#level-2-o_sync-and-fdatasync-per-write-durability}

`O_SYNC`はすべての`write()`を同期的にする。データ*と*関連するメタデータがストレージに載るまで呼び出しは戻らない。これは書き込みごとの保証としては最も強い。

`fdatasync(fd)`は、データと、そのデータを取り出すのに必要な最小限のメタデータ（変更があった場合のファイルサイズ）をフラッシュする。データの取り出しに必要でない限り、inodeのatime、mtime、ctimeは更新しない。

`fsync(fd)`は、データとすべてのメタデータ（タイムスタンプを含む）をフラッシュする。

```c
/* O_SYNC: 戻る前にすべての書き込みが永続化される */
int fd = open("file", O_WRONLY | O_SYNC);
write(fd, data, len);  /* データとメタデータがストレージに載ってから戻る */

/* fdatasync: 書き込みをまとめてからフラッシュ */
int fd = open("file", O_WRONLY);
for (int i = 0; i < n; i++) {
    write(fd, records[i], record_size);
}
fdatasync(fd);  /* バッファリングされた全書き込み + 最小限のメタデータをフラッシュ */

/* fsync: fdatasync と同じだがタイムスタンプもフラッシュする */
fsync(fd);
```

**コスト**: `fdatasync()`/`fsync()`のたびに、少なくとも1回のストレージ書き込み（デバイスへのフラッシュコマンド）が発生する。NVMeデバイスでは約100µs、SATA SSDでは約1ms、回転ディスクでは約5〜10msかかる。

**`fsync(fd)`で保証されないもの**: 親ディレクトリのエントリ。ファイルが新規作成またはリネームされた場合、`fsync(fd)`が戻った後でもディレクトリエントリはストレージに載っていないかもしれない。[fsync、fdatasync、O_SYNC](fsync-fdatasync.md)と事件簿: データ損失を参照。

---

## レベル3: ディレクトリのsync — 新しいファイルをクラッシュから守る {#level-3-directory-sync-ensuring-new-files-survive-crashes}

ファイルを作成またはリネームして`fsync(fd)`を呼んだ後、ファイルのデータとinodeはストレージに載っている。しかし、ファイルを指すディレクトリエントリは載っていないかもしれない。この時点でクラッシュすると、孤立したinode（データはディスク上にあるが、それを指すディレクトリエントリがない）やファイルの消失が起こりうる。

```c
/* クラッシュセーフなファイル作成 */
int fd = open("newfile", O_WRONLY | O_CREAT | O_TRUNC, 0644);
write(fd, data, len);
fdatasync(fd);      /* データと inode がストレージに載る */
close(fd);

/* まだ安全ではない: ディレクトリエントリがまだストレージに載っていない */
int dir_fd = open(".", O_RDONLY);
fsync(dir_fd);      /* ディレクトリエントリがストレージに載る */
close(dir_fd);
/* これで安全: newfile はクラッシュ後も残る */
```

**クラッシュセーフなリネームのパターン**（アトミックなファイル置換）:

```c
/* 一時ファイルに書き込む */
int tmp = open("file.tmp", O_WRONLY | O_CREAT | O_TRUNC, 0644);
write(tmp, new_data, len);
fdatasync(tmp);
close(tmp);

/* アトミックに置換 */
rename("file.tmp", "file");

/* ディレクトリを sync する — 誰もが忘れるステップ */
int dir = open(".", O_RDONLY);
fsync(dir);
close(dir);
```

`fsync(dir)`がなければ、リネームはメモリ上にしか存在しない。ディレクトリのpageがフラッシュされる前にクラッシュすると、ファイルシステムはリネーム前の状態になる — まるでリネームが起こらなかったかのように。

---

## レベル4: バリア書き込み — 完全なsyncなしの順序付け {#level-4-barrier-writes-ordering-without-full-sync}

ワークロードによっては、各書き込みを即座に永続化する必要はないが、書き込み間の順序付けが必要な場合がある。たとえばデータベースは、対応するログエントリが書き込まれる前にデータpageがストレージに載っていることを保証する必要があるが、各データpageの書き込みが同期的に戻る必要はない。

**`sync_file_range(2)`**はきめ細かな制御を提供する:

```c
/* 待たずに範囲の書き込みをキューに入れる */
sync_file_range(fd, offset, len, SYNC_FILE_RANGE_WRITE);

/* それらの書き込みの完了を待つ */
sync_file_range(fd, offset, len,
    SYNC_FILE_RANGE_WAIT_BEFORE | SYNC_FILE_RANGE_WRITE | SYNC_FILE_RANGE_WAIT_AFTER);
```

これはPostgreSQLで使われている。PostgreSQLは`SYNC_FILE_RANGE_WRITE`を指定して`sync_file_range()`を呼び、データpageのバックグラウンドライトバックを開始させ、後でWALをfsyncする前にそれらの書き込みの完了を待つ。これによりデータのライトバックとWALの書き込みを重ね合わせ、チェックポイントのスループットを向上させる。

**`O_DSYNC`**は、完全なsyncのオーバーヘッドなしに順序付けられた書き込みを提供する:

```c
/* O_DSYNC: write() が戻る前にデータはストレージに届くが、
   メタデータ（atime、mtime）は届かない */
int fd = open("file", O_WRONLY | O_DSYNC);
```

`O_DSYNC`は書き込みのたびに`fdatasync()`を呼ぶのと等価だが、カーネルがフラッシュ経路を最適化できるため、オーバーヘッドが小さくなる可能性がある。

---

## 書き込みの順序付けとブロック層 {#write-ordering-and-the-block-layer}

カーネルのブロック層には、書き込みの順序付けに影響するいくつかの仕組みがある:

**ライトバリア**（現在は「フラッシュコマンド」と呼ばれる）: Linux 2.6.37より前は、ライトバリアは明示的なブロック層のリクエストであり、それより前のすべての書き込みがバリアの前に完了し、後続のすべての書き込みがバリアの後に開始することを保証していた。2.6.37以降は、ブロック層のフラッシュ基盤によって自動的に処理される。

ファイルシステムがバリアを必要とする書き込み（たとえばジャーナルのコミット）を発行するときは、`REQ_PREFLUSH`および/または`REQ_FUA`フラグを設定する:

```
REQ_PREFLUSH: flush the device's write cache before this write
REQ_FUA: force unit access — write directly to persistent media, bypassing cache

Commit write sequence:
[data writes] → REQ_PREFLUSH → [journal commit write with REQ_FUA]
                     ↑                              ↑
              all data writes               this write is durable
              completed and                before returning
              flushed before this
```

**ライトキャッシュとFUA**: ストレージデバイスは通常、揮発性のライトキャッシュを持つ。デバイスのライトキャッシュに届いた書き込みは、デバイスの観点からは「永続化」されている — しかし、キャッシュがフラッシュされる前に電源障害が起きるとデータは失われる。`REQ_FUA`はキャッシュをバイパスして永続メディアに直接書き込むが、その代償としてレイテンシが高くなる。

```bash
# デバイスがライトキャッシュを持つか確認
hdparm -I /dev/sda | grep 'Write cache'

# ライトキャッシュの有効化/無効化
hdparm -W1 /dev/sda  # 有効化（デフォルト）
hdparm -W0 /dev/sda  # 無効化（すべての書き込みがメディアへ — 非常に遅い）
```

---

## ext4のorderedジャーナリングとdataジャーナリング {#ordered-vs-data-journaling-in-ext4}

ext4のジャーナリングモードが、ファイルシステムの提供する順序保証を決める:

**`data=ordered`**（デフォルト）: データブロックは、ジャーナルのメタデータコミットの*前に*最終的な位置へ書き込まれる。これにより、クラッシュとジャーナルのリプレイの後、ファイルのデータは新しいデータか古いデータのどちらかになり、再割り当てされたブロックの未初期化データになることはない。

**`data=journal`**: すべてのデータ書き込みがジャーナルを経由する。保証は最も強いが、性能は最も低い。各書き込みは2回書かれる（ジャーナル + 最終的な位置）。

**`data=writeback`**: メタデータはジャーナリングされるが、データの書き込みは完全に独立している。最も速いが、ジャーナルのコミット後かつデータ書き込み前にクラッシュすると、再利用されたブロックの古いデータが露出する可能性がある（事件簿: データ損失を参照）。

```bash
# ファイルシステムのジャーナリングモードを確認
tune2fs -l /dev/sda1 | grep "Default mount options"
# または
mount | grep ext4
# 確認する項目: data=ordered、data=journal、data=writeback
```

---

## NFSと分散環境での一貫性 {#nfs-and-distributed-consistency}

NFSはもう1つの次元を加える。クライアントのページキャッシュがサーバーの状態と同期していない可能性があり、複数のクライアントが競合するキャッシュ版を持つ可能性がある。

**NFSの一貫性モード:**

- **`close-to-open`**（NFSv3のデフォルト）: `close()`時にデータがサーバーへフラッシュされ、`open()`時にサーバーの版と照合される。openからcloseまでの間、クライアントは古いデータを持っている可能性がある。
- **`noac`**（属性キャッシュなし）: 属性キャッシュが無効になる。すべてのstat、read、writeがサーバーへ送られる。一貫しているが遅い。
- **デリゲーション**（NFSv4）: サーバーがクライアントに排他的アクセスを付与し、クライアントは自分が唯一の書き込み側であると分かったうえで積極的にキャッシュできる。

```bash
# 厳格な一貫性で NFS をマウント（noac は属性とデータのキャッシュを無効化）
mount -o noac,sync nfsserver:/export /mnt

# 現在のキャッシュタイムアウト設定を確認
mount | grep nfs
# 確認する項目: actimeo、acregmin、acregmax、acdirmin、acdirmax
```

---

## まとめ: 仕組みごとの一貫性保証 {#summary-consistency-guarantees-by-mechanism}

| 仕組み | データがストレージに載る | メタデータがストレージに載る | ディレクトリエントリ | 順序付け |
|-----------|----------------|---------------------|-----------------|----------|
| `write()` | いいえ（非同期） | いいえ | いいえ | なし |
| `write()` + `fsync(fd)` | はい | はい | いいえ | 呼び出し後 |
| `write()` + `fdatasync(fd)` | はい | 最小限 | いいえ | 呼び出し後 |
| `write()` + `O_SYNC` | はい（書き込みごと） | はい | いいえ | 書き込みごと |
| `write()` + `O_DSYNC` | はい（書き込みごと） | 最小限 | いいえ | 書き込みごと |
| `rename()` + `fsync(dir)` | はい（事前にfsyncしていれば） | はい | はい | ディレクトリのfsync後 |
| `sync_file_range(WRITE\|WAIT)` | はい（範囲） | いいえ | いいえ | あり（範囲） |
| `O_DIRECT` + `fdatasync` | はい | 最小限 | いいえ | 呼び出し後 |

---

## 関連ページ {#related-pages}

- [fsync、fdatasync、O_SYNC](fsync-fdatasync.md) — 永続性のセマンティクスの詳細
- [バッファードI/Oとページキャッシュ](buffered-io.md) — バッファード書き込みの仕組み
