# ext4のジャーナリングとクラッシュリカバリ {#ext4-journaling-and-crash-recovery}

> JBD2のトランザクションのライフサイクル: ext4が電源障害をどう生き延びるか

## なぜジャーナリングか {#why-journaling}

ジャーナリングがなければ、書き込みの途中で電源障害が起きると、ファイルシステムが不整合な状態に残されうる。

```
Without journal:
  1. Write inode (update file size)          ← power fail here
  2. Write data block
  3. Update block bitmap

After crash: file has new size but block bitmap says block is free.
fsck must scan the entire filesystem to find and fix inconsistencies.
On a 4TB filesystem: hours.
```

ジャーナリングでは、変更はまず**ジャーナル**（循環ログ）に書き込まれる。クラッシュ後、カーネルは数秒でジャーナルをリプレイする。

## ext4のジャーナルモード {#ext4-journal-modes}

```bash
# マウント時にジャーナルモードを確認／設定する:
mount -o data=ordered /dev/sda1 /mnt   # デフォルト
mount -o data=writeback /dev/sda1 /mnt  # 最高性能
mount -o data=journal /dev/sda1 /mnt    # 最も永続性が高い
```

| モード | ジャーナリング対象 | 性能 | 永続性 |
|------|-----------------|-------------|-----------|
| `writeback` | メタデータのみ。データはメタデータの後に書き込まれうる | 最速 | クラッシュ後にデータが壊れている可能性がある |
| `ordered` | メタデータのみ。**データはメタデータより先に書き込まれる** | 良好 | データブロックはメタデータより先にコミットされる |
| `journal` | メタデータとデータの両方 | 最も遅い | 最も強い |

デフォルトは`ordered`である。メタデータの変更はアトミックであり、データブロックは、それを参照するメタデータより先に書き込まれることが保証される。

## JBD2: ジャーナリングレイヤ {#jbd2-the-journaling-layer}

ext4はジャーナリングレイヤとしてJBD2（Journaling Block Device 2）を使う。JBD2はext4から独立しており、任意のブロックデバイスをジャーナリングできる。

### ディスク上のジャーナルのレイアウト {#journal-layout-on-disk}

```
Ext4 partition layout:
┌─────────────────────────────────────────────────────┐
│ Superblock │ Block groups │ Journal (special inode) │
│            │ [ext4 data]  │ [JBD2 circular log]     │
└─────────────────────────────────────────────────────┘

Journal structure (circular log):
┌───────────────────────────────────────────────────────────┐
│ Journal superblock │ T1: [commit block] │ T2: [commit] │..│
│ (seq, tail_sequence, s_start)                             │
└───────────────────────────────────────────────────────────┘

T1 (transaction 1):
  ┌────────────────────────────┬──────────────────────┬──────────────┐
  │ descriptor block           │ data/metadata blocks │ commit block │
  │ (lists what blocks follow) │ (original blocks)    │ (seq #)      │
  └────────────────────────────┴──────────────────────┴──────────────┘
```

ジャーナルのスーパーブロックは`s_start`（末尾）と`s_sequence`（現在のトランザクション）を記録する。リカバリは`s_start`から最後の有効なコミットブロックまでのすべてのトランザクションをリプレイする。

### struct journal_t {#struct-journal_t}

```c
/* include/linux/jbd2.h */
typedef struct journal_s {
    /* ジャーナルのスーパーブロックのデータ */
    unsigned long   j_flags;        /* JBD2_UNMOUNT、JBD2_ABORT、... */
    int             j_errno;
    struct buffer_head *j_sb_buffer; /* ジャーナルのスーパーブロックのバッファ */

    /* トランザクションの状態 */
    transaction_t  *j_running_transaction;    /* 現在オープンしているトランザクション */
    transaction_t  *j_committing_transaction; /* コミット中のもの */
    transaction_t  *j_checkpoint_transactions; /* まだ解放されていないもの */

    /* ジャーナルのブロック範囲 */
    unsigned int    j_maxlen;       /* ジャーナルの総ブロック数 */
    unsigned int    j_tail;         /* 既知の最も古いトランザクション */
    unsigned int    j_free;         /* ジャーナルの空きブロック数 */
    unsigned int    j_first;        /* ジャーナルの開始ブロック */
    unsigned int    j_last;         /* ジャーナルの終了ブロック（この値は含まない） */
    unsigned int    j_head;         /* 現在の書き込み位置 */

    /* シーケンス番号 */
    tid_t           j_tail_sequence;
    tid_t           j_transaction_sequence;
    tid_t           j_commit_sequence;
    tid_t           j_commit_request;

    /* ブロックデバイス */
    struct block_device *j_dev;     /* ジャーナルのデバイス */
    struct block_device *j_fs_dev;  /* ファイルシステムのデバイス */
    unsigned int        j_blocksize;

    /* kjournald カーネルスレッド */
    struct task_struct *j_task;

    /* コミットタイマーと workqueue */
    struct timer_list   j_commit_timer;
} journal_t;
```

### struct transaction_t {#struct-transaction_t}

```c
typedef struct transaction_s {
    journal_t      *t_journal;
    tid_t           t_tid;          /* トランザクション ID */
    enum {
        T_RUNNING,          /* 新しいハンドルを受け付けている */
        T_LOCKED,           /* ハンドルが閉じられるのを待っている */
        T_SWITCH,
        T_FLUSH,
        T_COMMIT,
        T_COMMIT_DFLUSH,
        T_COMMIT_JFLUSH,
        T_FINISHED,
    } t_state;

    struct list_head t_buffers;      /* このトランザクションで変更されたバッファ */
    struct list_head t_forget;       /* コミット後にリンクを外すバッファ */
    struct list_head t_checkpoint_list; /* チェックポイントが必要なバッファ */

    atomic_t        t_handle_count; /* オープンしているハンドルの数 */
    unsigned int    t_outstanding_credits; /* 残りのジャーナル領域 */
} transaction_t;
```

## トランザクションのライフサイクル {#transaction-lifecycle}

### フェーズ1: 開始 {#phase-1-begin}

```c
/* トランザクションを開始する */
handle_t *handle = jbd2_journal_start(journal, nblocks);
/* nblocks: このハンドルが変更する最悪ケースのブロック数 */
/* ハンドルを返し、t_handle_count をインクリメントする */
```

### フェーズ2: バッファの変更 {#phase-2-modify-buffers}

```c
/* バッファを変更しようとしていることを JBD2 に伝える（ジャーナルクレジットを得る）: */
jbd2_journal_get_write_access(handle, bh);
/* bh をトランザクションの t_buffers リストに追加する */
/* 元のブロックをジャーナルバッファにコピーする */

/* バッファを変更する: */
ext4_set_bit(block, group->bg_block_bitmap->b_data);

/* 変更が完了したことを JBD2 に伝える: */
jbd2_journal_dirty_metadata(handle, bh);
/* トランザクション内で bh をダーティとしてマークする */
```

### フェーズ3: コミット（kjournald） {#phase-3-commit-kjournald}

```c
/* kjournald スレッド（fs/jbd2/journal.c） */
static int kjournald2(void *arg)
{
    journal_t *journal = arg;
    transaction_t *transaction;

    for (;;) {
        /* コミットが必要になるか、タイマーが切れるのを待つ */
        wait_event_interruptible_timeout(journal->j_wait_commit, ...);

        /* 現在のトランザクションをコミットする */
        jbd2_journal_commit_transaction(journal);
    }
}
```

コミットのフェーズ:
1. **クローズ**: このトランザクションで新しいハンドルの受け付けを停止する（T_LOCKED）
2. **ディスクリプタの書き込み**: 変更されたすべてのバッファを列挙するディスクリプタブロックを書き込む
3. **データの書き込み**: 変更されたすべてのメタデータバッファをジャーナルに書き込む
4. **フラッシュ**: すべてのジャーナル書き込みの完了を待つ
5. **コミットブロックの書き込み**: トランザクションのシーケンス番号を含むコミットブロックを書き込む
6. **チェックポイント**: コミットブロックがディスク上に載ったら、ファイルシステムのブロックをインプレースで書き込める
7. **解放**: チェックポイントの後、ジャーナル領域を解放する

### フェーズ4: チェックポイント {#phase-4-checkpoint}

コミットの後、ジャーナル領域を再利用できるようにする前に、元のファイルシステムのブロックを書き出さ（チェックポイントし）なければならない。

```c
/* トランザクションで変更されたブロックは、いずれ
   ファイルシステム上の実際の位置へ書き込まれる（チェックポイントされる） */
jbd2_log_do_checkpoint(journal);

/* チェックポイントの後: ジャーナル領域は次のトランザクションのために解放される */
```

## クラッシュリカバリ {#crash-recovery}

```bash
# ジャーナルの状態を表示する
debugfs -R "show_super_stats" /dev/sda1 | grep -i journal
# Journal inode: 8
# Journal backup: inode blocks
# Journal features: journal_64bit, journal_fast_commit
# Journal size: 128m
# Journal length: 32768

# ジャーナルを手動でリプレイする（ドライラン）:
e2fsck -n /dev/sda1 2>&1 | head -20
# Pass 1: Checking inodes, blocks, and sizes
# Pass 2: Checking directory structure
# ...
```

### カーネルによるジャーナルリカバリ {#kernel-journal-recovery}

```c
/* fs/jbd2/recovery.c */
int jbd2_journal_recover(journal_t *journal)
{
    struct recovery_info info = {};

    /* ジャーナルをスキャンして、有効なトランザクションをすべて探す */
    err = do_one_pass(journal, &info, PASS_SCAN);
    /* コミット済みのトランザクションをすべて見つける: info.end_transaction */

    /* 有効なトランザクションをリプレイする: ブロックを実際の位置へ書き込む */
    err = do_one_pass(journal, &info, PASS_REVOKE);
    /* revoke レコード（解放済みでリプレイすべきでないブロック）を適用する */

    err = do_one_pass(journal, &info, PASS_REPLAY);
    /* 残りのトランザクションをファイルシステム上の実際の位置へリプレイする */

    jbd2_journal_clear_revoke(journal);
    err = sync_blockdev(journal->j_fs_dev);
    return err;
}
```

リカバリは**O(ジャーナルサイズ)**であり、O(ファイルシステムサイズ)ではない。リプレイされるのはジャーナル（デフォルト128MB）だけなので、2TBのファイルシステムでも数秒で回復する。

## Fast commit（ext4 5.10以降） {#fast-commit-ext4-510}

JBD2の完全なコミットはコストが高い（数百マイクロ秒）。**Fast commit**は、よくある操作のために最適化されたパスである。

```c
/* fs/ext4/fast_commit.c */
/* Fast commit はブロック全体ではなく、変更されたもの（差分）だけを記録する: */
/* - inode の変更（truncate、setattr）           */
/* - ディレクトリエントリの変更（create、unlink） */

/* ジャーナル書き込みがはるかに小さい: コミットあたり約 1KB 対 約 4KB */
```

```bash
# fast commit を有効にする（新しい ext4 ファイルシステムではデフォルト）:
tune2fs -O fast_commit /dev/sda1
mke2fs -O fast_commit /dev/sda1

# 確認する:
tune2fs -l /dev/sda1 | grep "Filesystem features"
# Filesystem features: fast_commit ...
```

## ext4のジャーナリングとfsync {#ext4-journaling-vs-fsync}

```c
/* アプリケーション: データの永続性を保証する */
fd = open("file", O_WRONLY);
write(fd, data, len);
fsync(fd);     /* データとメタデータの両方がディスクに載るまでブロックする */
/* fsync なしでは: データはディスクではなく、ページキャッシュかジャーナルにあるかもしれない */

/* fdatasync: データのみ（ファイルサイズが変わらない限りメタデータは含まない） */
fdatasync(fd);

/* O_SYNC: 毎回の書き込みの後に fsync する */
fd = open("file", O_WRONLY | O_SYNC);
```

ジャーナルが保証するのは、クラッシュ後のメタデータの一貫性だけである。データの永続性には`fsync()`または`O_SYNC`が必要である。

## ジャーナル性能の監視 {#monitoring-journal-performance}

```bash
# ジャーナルの統計
cat /proc/fs/ext4/<dev>/journal_info
# journal_sequence: 12345
# journal_buffers: 0
# journal_outstanding: 0

# ジャーナルのコミットをトレースする
bpftrace -e '
kprobe:jbd2_journal_commit_transaction {
    @commits = count();
    @time_ns = hist(nsecs);
}'

# iostat: ジャーナルの書き込みは同じデバイスへの書き込みとして現れる
iostat -x 1

# blktrace: ジャーナルの書き込みとデータの書き込みを見分ける
blktrace -d /dev/sda1 -o - | blkparse -i - | grep -v "Q  W" | head -50
```

## 参考文献 {#further-reading}

- [ext4](ext4.md) — ext4のアーキテクチャとディスクレイアウト
- [write()システムコールの一生](../vfs/life-of-write.md) — JBD2で終わる書き込みパス
- [ページキャッシュ](../mm/page-cache.md) — JBD2がジャーナリングするダーティなpage
- [IOMMU DMA](../mm/dma.md) — ジャーナルのI/OはDMAを使う
- [Kernel docs: ext4 journal (jbd2)](https://docs.kernel.org/filesystems/ext4/journal.html) — ディスク上のジャーナルフォーマット
- `fs/jbd2/` — JBD2の実装
- `fs/ext4/` — ext4とJBD2の統合
