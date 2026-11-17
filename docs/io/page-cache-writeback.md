# ページキャッシュのライトバック {#page-cache-writeback}

> カーネルがダーティpageをストレージへどうフラッシュするか、そしてそのタイミングを制御するものは何か

## ライトバックとは何か {#what-writeback-is}

プロセスが`write()`を呼ぶと、カーネルはデータを**ページキャッシュ**のpageにコピーし、それを**ダーティ** — メモリ上では変更されているが、まだディスクには載っていない — としてマークする。その後のある時点で、カーネルはこれらのダーティpageをストレージへフラッシュしなければならない。この処理を**ライトバック**と呼ぶ。

```
write() path:
  process → copy_from_user() → page cache (dirty) → return to caller
                                       ↓
                              [later, asynchronously]
                                       ↓
                             writeback → block layer → disk
```

ライトバックは、相反する2つの圧力の交点に位置している:

- **書き込み性能**: 書き込みをDRAMにバッファリングして大きなバッチでフラッシュする方が、すべての`write()`呼び出しを同期的にディスクへ書くよりはるかに速い。
- **メモリ圧迫**: ダーティpageはライトバックされるまで回収できない。ダーティpageが多すぎると、新たな割り当てに使える空きメモリが減る。

カーネルは、ある時点でダーティにできるRAMの量を制限し、バックグラウンドで先回りしてダーティpageをフラッシュすることで、これらのバランスを取っている。

## ライトバックスレッド: `bdi_writeback` {#writeback-threads-bdi_writeback}

各ブロックデバイスには**backing device info**（`struct bdi_writeback`）が関連付けられており、そのデバイス上のすべてのファイルシステムのライトバック状態を保持している。BDIごとのカーネルワーカースレッド（`wb_workfn`）が実際のフラッシュ処理を実行する。

```c
/* include/linux/backing-dev-defs.h */
struct bdi_writeback {
    struct backing_dev_info *bdi;   /* 親 BDI */
    unsigned long state;            /* WB_writeback_running など */
    unsigned long last_old_flush;   /* 古いデータ（期限切れ）のライトバックを最後に実行した時刻 */

    struct list_head b_dirty;       /* ダーティな inode */
    struct list_head b_io;          /* ライトバックの準備ができた inode */
    struct list_head b_more_io;     /* さらなるライトバックを待っている inode */
    struct list_head b_dirty_time;  /* 期限切れのダーティな inode */

    spinlock_t list_lock;           /* inode リストを保護する */

    struct percpu_counter stat[NR_WB_STAT_ITEMS]; /* 書き込んだバイト数など */

    struct delayed_work dwork;      /* wb_workfn の work アイテム */
    struct delayed_work bw_dwork;   /* 帯域幅推定の work */
};
```

work関数`wb_workfn()`（`fs/fs-writeback.c`内）は、`dwork`の遅延workアイテムを通じてスケジュールされる。この関数は`writeback_inodes_wb()`を呼び出してダーティinodeのリストをたどり、pageをストレージへ送り出す:

```c
/* fs/fs-writeback.c（簡略化） */
void wb_workfn(struct work_struct *work)
{
    struct bdi_writeback *wb =
        container_of(to_delayed_work(work), struct bdi_writeback, dwork);
    struct wb_writeback_work *work_item;

    /* まず明示的な work アイテム（sync、fsync）を処理する */
    while ((work_item = get_next_work_item(wb)) != NULL) {
        wb_do_writeback(wb);
        finish_writeback_work(wb, work_item);
    }

    /* 次に定期的/バックグラウンドのライトバックを行う */
    wb_do_writeback(wb);

    /* まだダーティなデータがあれば再スケジュールする */
    if (wb_has_dirty_io(wb))
        wb_wakeup_delayed(wb);
}

static long writeback_inodes_wb(struct bdi_writeback *wb,
                                 struct wb_writeback_work *work)
{
    /* inode を b_dirty → b_io へ移す */
    queue_io(wb, work, dirtied_before);

    /* b_io リストを順にたどり、各 inode のダーティ page を書き込む */
    while (!list_empty(&wb->b_io)) {
        struct inode *inode = wb_inode(wb->b_io.prev);
        writeback_sb_inodes(inode->i_sb, wb, work);
    }
}
```

実行中のシステムにどのBDIが存在するかを確認するには:

```bash
ls /sys/class/bdi/
# 例: 8:0  259:0  0:12  ...

# BDI ごとのライトバック統計:
cat /sys/class/bdi/8:0/stats
```

## ダーティスロットリング {#dirty-throttling}

カーネルは、蓄積できるダーティデータの量を制御するため、利用可能なメモリに対する割合で表される2つのしきい値を設けている。

### `dirty_background_ratio`（デフォルト: 10%） {#dirty_background_ratio-default-10}

**ソフトリミット**である。ダーティpageがメモリのこの割合を超えると、バックグラウンドのフラッシャー（`wb_workfn`）が起こされて書き込みを開始する。新しいデータを書き込んでいるプロセスは停止させられない — ライトバックは並行して実行される。

### `dirty_ratio`（デフォルト: 20%） {#dirty_ratio-default-20}

**ハードリミット**である。ダーティpageがメモリのこの割合に達すると、カーネルは書き込み中のプロセスを直接スロットリングする。`balance_dirty_pages()`は、十分な量のダーティpageがフラッシュされるまで、プロセスを（`io_schedule_timeout()`によって`TASK_UNINTERRUPTIBLE`のD状態にして）停止させる。これは**直接回収スロットリング**（direct reclaim throttling）または**ダーティスロットリング**と呼ばれることがある。スロットリングされた書き込み側はD状態（I/Oを待つ割り込み不可能なスリープ）として現れるため、`vmstat`はそれらを`b`（blocked）として報告する。

```
          0%        10%         20%
          |---------|-----------|
          clean   background   direct
                   flusher     reclaim
                   wakes up    (process stalls)
```

### `/proc/sys/vm/`のパラメータ {#procsysvm-parameters}

| パラメータ | デフォルト | 説明 |
|-----------|---------|-------------|
| `dirty_ratio` | 20 | ハードリミット: 書き込み中のプロセスがスロットリングされるまでにダーティにできるメモリの割合（%） |
| `dirty_background_ratio` | 10 | ソフトリミット: バックグラウンドのライトバックを起動するメモリの割合（%） |
| `dirty_bytes` | 0 | バイト単位のハードリミット（0以外のとき`dirty_ratio`より優先される） |
| `dirty_background_bytes` | 0 | バイト単位のソフトリミット（0以外のとき`dirty_background_ratio`より優先される） |
| `dirty_writeback_centisecs` | 500 | フラッシャーが起きる間隔。センチ秒単位（500 = 5秒） |
| `dirty_expire_centisecs` | 3000 | ダーティpageがライトバックの対象となるまでの経過時間（3000 = 30秒） |

```bash
# 現在の値を読む:
sysctl vm.dirty_ratio vm.dirty_background_ratio \
       vm.dirty_writeback_centisecs vm.dirty_expire_centisecs

# 書き込みの多いワークロード向けにチューニング（バッファリングを増やす）:
sysctl -w vm.dirty_ratio=40
sysctl -w vm.dirty_background_ratio=20

# 永続性重視のチューニング（より積極的にフラッシュする）:
sysctl -w vm.dirty_writeback_centisecs=100   # 1 秒ごとに起きる
sysctl -w vm.dirty_expire_centisecs=500      # 5 秒より古い page をフラッシュ
```

ハードリミットを強制するロジックは`mm/page-writeback.c`にある:

```c
/* mm/page-writeback.c（簡略化） */
static void balance_dirty_pages(struct bdi_writeback *wb,
                                 unsigned long pages_dirtied)
{
    for (;;) {
        unsigned long nr_dirty = global_node_page_state(NR_FILE_DIRTY);
        unsigned long dirty_thresh = wb_calc_thresh(wb, ...);

        if (nr_dirty <= dirty_thresh)
            break;

        /* バックグラウンドのライトバックを起動 */
        wb_start_background_writeback(wb);

        /* 書き込み中のプロセスをスロットリング（タスクは D 状態 / TASK_UNINTERRUPTIBLE に入る） */
        nr_reclaimable = nr_dirty - dirty_thresh;
        pause = msecs_to_jiffies(
            balance_dirty_pages_ratelimited_interval(wb, nr_reclaimable));
        io_schedule_timeout(pause);
    }
}
```

## ライトバックが起動するとき {#when-writeback-triggers}

ライトバックは4つの異なる発生源から起動される:

### 1. 定期的（タイマーベース） {#1-periodic-timer-based}

`dirty_writeback_centisecs`センチ秒ごとに、BDIごとの遅延workアイテムが`wb_workfn`を起動する。これは`dirty_expire_centisecs`より古いpageをフラッシュする。ほとんどのワークロードにとって、これが通常の定常状態の経路である。

### 2. ソフトリミットの超過 {#2-soft-limit-crossed}

ダーティpageが`dirty_background_ratio`を超えると、`balance_dirty_pages()`から`wb_start_background_writeback()`が呼ばれ、書き込み側を停止させることなくフラッシャーを即座に起こす。

### 3. メモリ圧迫（kswapd） {#3-memory-pressure-kswapd}

`kswapd`がpageを回収しようとするとき、ダーティpageはまず書き出さなければ回収できない。`kswapd`は`wakeup_flusher_threads()`を呼び、ダーティpageをフラッシュして回収可能にするようライトバックに依頼する。バックグラウンドのライトバックが遅すぎる場合は、`kswapd`自身がpageを直接書き出すこともある。

### 4. 明示的なsync呼び出し {#4-explicit-sync-calls}

`sync()`、`fsync()`、`fdatasync()`は、`wb_writeback_work`アイテムをBDIのworkキューに直接投入し、定期タイマーを完全にバイパスする。

```
                ┌──────────────────────────────────────────┐
                │            writeback triggers            │
                ├───────────┬───────────┬──────────────────┤
                │ periodic  │ mem       │ explicit         │
                │ (5s timer)│ pressure  │ sync/fsync       │
                └─────┬─────┴─────┬─────┴────┬─────────────┘
                      └───────────┴──────────┘
                                  │
                           wb_workfn()
                                  │
                    writeback_inodes_wb()
```

## ライトバックの経路 {#the-writeback-path}

`wb_workfn`が動き出すと、呼び出しチェーンはVFSを下ってファイルシステムへと進む:

```
wb_workfn()
  └── wb_do_writeback()
        └── writeback_inodes_wb()          fs/fs-writeback.c
              └── writeback_sb_inodes()    per-superblock loop
                    └── writeback_single_inode()
                          └── do_writepages()     mm/page-writeback.c
                                └── a_ops->writepages()   filesystem op
                                      └── (e.g.) ext4_writepages()
                                            └── mpage_writepages() / iomap_writepages()
                                                  └── submit_bio()
                                                        └── block layer → driver → disk
```

重要な境界は`a_ops->writepages` — `address_space_operations`ベクタ — である。各ファイルシステムは独自の実装を持つが、最終的にはいずれも`submit_bio()`を呼び出す。

```c
/* mm/page-writeback.c */
int do_writepages(struct address_space *mapping, struct writeback_control *wbc)
{
    int ret;

    if (mapping->a_ops->writepages)
        ret = mapping->a_ops->writepages(mapping, wbc);
    else
        ret = generic_writepages(mapping, wbc);

    return ret;
}
```

`writeback_control`構造体は、ライトバック処理のパラメータを運ぶ。何page書くか、I/Oの完了を待つかどうか、ファイルのどの範囲をフラッシュするかである。

```c
/* include/linux/writeback.h */
struct writeback_control {
    long nr_to_write;           /* 書き込む page 数（減算されていく） */
    long pages_skipped;         /* スキップされた page */
    loff_t range_start;         /* 書き込むファイル範囲の開始 */
    loff_t range_end;           /* ファイル範囲の終了 */
    enum writeback_sync_modes sync_mode; /* WB_SYNC_NONE または WB_SYNC_ALL */
    unsigned for_kupdate:1;     /* 定期的なフラッシャーによるライトバック */
    unsigned for_background:1;  /* バックグラウンド（ソフトリミット）のライトバック */
    unsigned tagged_writepages:1; /* タグベースのライトバック */
};
```

## `fsync()`と`fdatasync()`と`sync()` {#fsync-vs-fdatasync-vs-sync}

この3つの呼び出しはよく混同される。範囲と、どのメタデータを保証するかが異なる:

| 呼び出し | 範囲 | データ | ファイルのメタデータ（サイズ、mtime） | その他のメタデータ（ディレクトリエントリ） |
|------|-------|------|-----------------------------|---------------------------|
| `fdatasync(fd)` | 単一ファイル | はい | データの完全性に必要な場合のみ | いいえ |
| `fsync(fd)` | 単一ファイル | はい | はい | いいえ |
| `sync()` | すべてのファイルシステム | はい | はい | はい（遅延され、即時は保証されない） |

```c
/* 新しく書き込んだファイルを永続化し、クラッシュ後も見つけられるようにする */

int fd = open("data.bin", O_WRONLY | O_CREAT, 0644);
write(fd, buf, len);

/* ステップ 1: ファイルのデータとメタデータをディスクへフラッシュ */
fsync(fd);

/* ステップ 2: ディレクトリエントリをフラッシュ（ファイル名がクラッシュ後も残るように） */
int dir_fd = open(dirname(path), O_RDONLY);
fsync(dir_fd);    /* ディレクトリのデータをフラッシュし、エントリを永続化する */
close(dir_fd);

close(fd);
```

`fdatasync()`は、重要でないメタデータ（mtime、atime、ctime）と、それに伴うinodeのジャーナルコミットをすべてスキップする — ただしファイルサイズが変わった場合は例外で、正しくデータを読み出すために必要なのでサイズの更新は引き続きフラッシュされる。ext4では、サイズの変更がなければ`fdatasync`ごとにジャーナルトランザクションを丸ごと1回節約できる。

`fsync()`のカーネルパス:

```c
/* fs/sync.c */
SYSCALL_DEFINE1(fsync, unsigned int, fd)
{
    return do_fsync(fd, 0);  /* 0 = データ + メタデータを sync */
}

SYSCALL_DEFINE1(fdatasync, unsigned int, fd)
{
    return do_fsync(fd, 1);  /* 1 = datasync（メタデータは必要な場合のみ） */
}

static int do_fsync(unsigned int fd, int datasync)
{
    struct fd f = fdget(fd);
    ret = vfs_fsync(f.file, datasync);
    /* vfs_fsync → file->f_op->fsync → ファイルシステムの fsync 実装 */
}
```

## ライトバリアと永続化 {#write-barriers-and-persistence}

`fsync()`が戻った後でも、永続性はストレージデバイスが書き込み順序を守るかどうかに依存する。カーネルは**ライトバリア**を使って、順序の要件をブロック層に伝える。

### `O_SYNC`と`O_DSYNC` {#o_sync-and-o_dsync}

`O_SYNC`でファイルを開くと、すべての`write()`が`write()` + `fsync()`のように振る舞う — データとメタデータが永続ストレージにフラッシュされるまで呼び出しは戻らない。

`O_DSYNC`は`fdatasync()`の書き込みごと版である。各`write()`はデータ（およびファイルが拡張された場合はサイズのメタデータ）をフラッシュするが、タイムスタンプはフラッシュしない。

```c
/* O_SYNC: すべての書き込みが即座に永続化される */
int fd = open("journal.log", O_WRONLY | O_CREAT | O_SYNC, 0644);
write(fd, record, len);  /* ディスクに載るまで戻らない */

/* O_DSYNC: データは永続化されるが、mtime は遅れることがある */
int fd = open("data.bin", O_WRONLY | O_CREAT | O_DSYNC, 0644);
write(fd, buf, len);  /* データは永続化される。inode の mtime はそうとは限らない */
```

これらのフラグはカーネル内の`kiocb`に`IOCB_SYNC` / `IOCB_DSYNC`を設定する。ファイルシステムは書き込みの完了後にこれを確認し、`vfs_fsync_range()`をインラインで呼ぶかどうかを決める。

### キャッシュフラッシュコマンド {#flush-cache-commands}

ブロック層は、ライトバリアの要件をストレージのコマンドに変換する:

| 要件 | SATA/SCSIのコマンド | NVMeのコマンド |
|-------------|-------------------|--------------|
| ライトキャッシュのフラッシュ | `FLUSH CACHE` | `Flush` |
| Force Unit Access（書き込みごと） | 書き込みの`FUA`ビット | 書き込みの`FUA`ビット |

FUA（Force Unit Access）は、特定の書き込みについてライトキャッシュをバイパスするようドライブに指示し、別途フラッシュコマンドを送る往復のコストを回避する。ブロック層は、FUAをサポートするドライブでは`O_DSYNC`の書き込みにFUAを使う:

```bash
# ドライブが FUA をサポートしているか確認:
cat /sys/block/sda/queue/fua       # 1 = サポート
cat /sys/block/nvme0n1/queue/fua   # 1 = サポート

# ライトキャッシュの状態を確認:
hdparm -W /dev/sda                 # 0 = 無効、1 = 有効
```

### 順序保証のまとめ {#ordering-guarantees-summary}

```
O_SYNC write:
  write() → page cache → writeback → FLUSH CACHE → return
                                          (or FUA on supported drives)

fsync():
  write() → page cache → return
  ...
  fsync() → writeback of all dirty pages → FLUSH CACHE → return

fdatasync():
  same as fsync() but skips all non-critical metadata (mtime, atime, ctime)
  and the associated journal commit for the inode — unless the file size
  changed, in which case the size update is still flushed because it is
  required for correct data reads. On ext4 this can save one full journal
  transaction per fdatasync when no size change occurred.
```

ドライブが揮発性のライトキャッシュを持ち、ライトキャッシュが有効になっている場合、`O_SYNC`/`fsync()`なしで書き込まれたデータは、`write()`が戻った後でも電源障害で失われうる — まだドライブのDRAMに残っている可能性があるからである。

## ライトバックの観測 {#observing-writeback}

```bash
# ダーティ page の統計をリアルタイムで:
watch -n1 'grep -E "Dirty|Writeback" /proc/meminfo'

# BDI ごとのライトバック統計:
cat /sys/class/bdi/*/stats

# ftrace でライトバックのイベントをトレース:
echo 1 > /sys/kernel/debug/tracing/events/writeback/enable
cat /sys/kernel/debug/tracing/trace_pipe

# vmstat: ライトバックの列を見る（bo = 1 秒あたりの出力ブロック数）:
vmstat 1

# iostat: デバイスごとの書き込みスループットを見る:
iostat -x 1
```

## 参考文献 {#further-reading}

- [バッファードI/O](buffered-io.md) — ページキャッシュの仕組みと、ダーティpageがどう作られるか
- [Direct I/O](direct-io.md) — ページキャッシュとライトバックの完全なバイパス
- [非同期I/O](async-io.md) — ノンブロッキングなI/O発行のためのio_uringとLinux AIO
- `mm/page-writeback.c` — ダーティスロットリング、`balance_dirty_pages()`、`do_writepages()`
- `fs/fs-writeback.c` — `wb_workfn()`、`writeback_inodes_wb()`、`writeback_sb_inodes()`
- `include/linux/backing-dev-defs.h` — `struct bdi_writeback`の定義
- `include/linux/writeback.h` — `struct writeback_control`
