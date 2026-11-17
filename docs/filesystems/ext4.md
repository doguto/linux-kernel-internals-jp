# ext4 {#ext4}

> Linuxのデフォルトファイルシステム。ジャーナリング、エクステントベースで、実戦で鍛えられている。Theodore Ts'o、Andreas Dilger、Alex Tomasらによって開発され、Linux 2.6.28で安定版としてマージされた。

## オンディスクレイアウト {#on-disk-layout}

ext4ファイルシステムは**ブロックグループ**に分割される。各ブロックグループは次のものを含む。

```
Block Group 0:
┌─────────────────────────────────────────────────────────────────┐
│ Super-   │ Group     │ Block    │ Inode    │ Inode │ Data       │
│ block    │ Descriptor│ Bitmap   │ Bitmap   │ Table │ Blocks     │
│ (1 block)│ Table     │ (1 block)│ (1 block)│       │            │
└─────────────────────────────────────────────────────────────────┘

Superblock: copies in groups 0, 1, 3^n, 5^n, 7^n (backup)
```

```bash
# ファイルシステムのレイアウトを調べる
dumpe2fs /dev/sda1 | head -80
# Inode count:              2621440
# Block count:              10485760
# Block size:               4096
# Blocks per group:         32768
# Inodes per group:         8192
# Inode size:               256
# Journal inode:            8
```

## ext4のスーパーブロック {#the-ext4-superblock}

```c
/* fs/ext4/ext4.h */
struct ext4_super_block {
    __le32  s_inodes_count;         /* inode の総数 */
    __le32  s_blocks_count_lo;      /* ブロックの総数（下位 32 ビット） */
    __le32  s_r_blocks_count_lo;    /* 予約ブロック */
    __le32  s_free_blocks_count_lo;
    __le32  s_free_inodes_count;
    __le32  s_first_data_block;     /* 最初のデータブロック（4K ブロックでは 0） */
    __le32  s_log_block_size;       /* ブロックサイズ = 1024 << s_log_block_size */
    __le32  s_blocks_per_group;
    __le32  s_inodes_per_group;
    __le32  s_mtime;                /* 最終マウント時刻 */
    __le32  s_wtime;                /* 最終書き込み時刻 */
    __le16  s_magic;                /* 0xEF53 */
    __le16  s_state;                /* ファイルシステムの状態: clean/errors */
    __le16  s_errors;               /* エラー時の動作: continue/remount-ro/panic */
    /* ... */
    __le32  s_feature_compat;       /* 互換機能フラグ */
    __le32  s_feature_incompat;     /* 非互換機能フラグ */
    __le32  s_feature_ro_compat;    /* 読み取り専用互換機能 */
    __u8    s_uuid[16];             /* ファイルシステムの UUID */
    char    s_volume_name[16];
    /* ... ジャーナル情報、暗号化など */
};
```

## ext4のinode {#the-ext4-inode}

```c
struct ext4_inode {
    __le16  i_mode;             /* ファイルモード（S_IFREG、S_IFDIR など） */
    __le16  i_uid;              /* UID の下位 16 ビット */
    __le32  i_size_lo;          /* ファイルサイズ（下位 32 ビット） */
    __le32  i_atime;            /* アクセス時刻 */
    __le32  i_ctime;            /* inode 変更時刻 */
    __le32  i_mtime;            /* 更新時刻 */
    __le32  i_dtime;            /* 削除時刻 */
    __le16  i_gid;
    __le16  i_links_count;      /* ハードリンク数 */
    __le32  i_blocks_lo;        /* 使用している 512 バイトブロック数 */
    __le32  i_flags;            /* EXT4_EXTENTS_FL、EXT4_INLINE_DATA_FL、... */
    __le32  i_block[EXT4_N_BLOCKS]; /* 15 ワード: エクステントツリーのルート、またはブロックマップ */
    __le32  i_generation;
    __le32  i_file_acl_lo;      /* 拡張属性ブロック */
    __le32  i_size_high;        /* ファイルサイズの上位 32 ビット */
    /* ... 128 バイトを超える inode では、オフセット 128 以降に追加フィールド ... */
    __le16  i_extra_isize;      /* 追加 inode サイズ（新しいフィールドを可能にする） */
    /* ナノ秒精度のタイムスタンプ: */
    __le32  i_ctime_extra;
    __le32  i_mtime_extra;
    __le32  i_atime_extra;
    __le32  i_crtime;           /* 作成時刻 */
    __le32  i_crtime_extra;
    /* ... バージョン、チェックサム ... */
};
```

## エクステントツリー: 大きなファイルの効率的な表現 {#extent-tree-efficient-large-file-representation}

ext4は古い間接ブロックマップの代わりにエクステント（連続したブロックの並び）を使う。エクステントはAlex Tomasによって導入された [(commit)](https://git.kernel.org/linus/a86c61812637c7dd0c57e29880cffd477b62f2e7)。

```
i_block[0..3] holds the extent tree root:
┌───────────────────────────────────────────────────────┐
│ ext4_extent_header                                    │
│   eh_magic=0xF30A, eh_entries=N, eh_depth=0 (leaf)    │
├───────────────────────────────────────────────────────┤
│ ext4_extent[0]: ee_block=0,  ee_len=128, ee_start=1024│
│ ext4_extent[1]: ee_block=128,ee_len=64,  ee_start=2048│
└───────────────────────────────────────────────────────┘

For deep files, eh_depth>0: i_block contains internal nodes
with ext4_extent_idx entries pointing to child blocks
```

```c
struct ext4_extent {
    __le32  ee_block;       /* カバーする最初の論理ブロック */
    __le16  ee_len;         /* ブロック数（unwritten の場合は 0x8000 | len） */
    __le16  ee_start_hi;    /* 物理ブロック（上位 16 ビット） */
    __le32  ee_start_lo;    /* 物理ブロック（下位 32 ビット） */
};

struct ext4_extent_header {
    __le16  eh_magic;       /* 0xF30A */
    __le16  eh_entries;     /* エントリ数 */
    __le16  eh_max;         /* このノードの容量 */
    __le16  eh_depth;       /* 深さ（0=リーフ、>0=内部ノード） */
    __le32  eh_generation;
};
```

古い間接マップと比べた利点は次のとおりである。

- 1つのエクステントで連続したブロックをカバーできる（例: 1エントリで128MB）
- シーケンシャルな読み出しがディスク上で連続する → 高速
- 大きなファイルでinodeツリーの深さが減る

## jbd2によるジャーナリング {#journaling-with-jbd2}

ext4はクラッシュ一貫性のためにJournal Block Device（jbd2）を使う。

### 書き込みモード {#write-modes}

| モード | ジャーナリング対象 | データの安全性 | 性能 |
|------|-----------|-------------|-------------|
| `data=writeback` | メタデータのみ | クラッシュ後、データが事前割り当てされたゴミになっている可能性がある | 最速 |
| `data=ordered`（デフォルト） | メタデータのみだが、データはメタデータより先に書き込まれる | メタデータがコミットされた時点でデータもコミット済み | バランスが良い |
| `data=journal` | メタデータとデータの両方 | 最も強い | 最も遅い |

```bash
# 現在のモードを確認する
tune2fs -l /dev/sda1 | grep "Default mount options"
# Default mount options: user_xattr acl

# 特定のジャーナルモードでマウントする
mount -o data=journal /dev/sda1 /mnt
```

### ジャーナルの構造（jbd2） {#journal-structure-jbd2}

```c
/* fs/jbd2/journal.c */
struct journal_s {              /* journal_t */
    unsigned long   j_flags;   /* JBD2_UNMOUNT、JBD2_ABORT、... */
    int             j_errno;

    struct buffer_head *j_sb_buffer; /* ジャーナルのスーパーブロック */
    journal_superblock_t *j_superblock;

    tid_t           j_head;     /* 最新のトランザクション通し番号 */
    tid_t           j_tail;     /* ディスク上にまだ残っている最も古いコミット済みトランザクション */
    unsigned long   j_free;     /* 残りの空き領域 */

    struct transaction_s *j_running_transaction;  /* 現在のトランザクション */
    struct transaction_s *j_committing_transaction;
    struct list_head j_checkpoint_transactions;

    wait_queue_head_t j_wait_transaction_locked;
    wait_queue_head_t j_wait_done_commit;
    wait_queue_head_t j_wait_commit;
    wait_queue_head_t j_wait_updates;

    struct task_struct *j_task;  /* kjournald スレッド */
    int j_max_transaction_buffers;
    unsigned long j_commit_interval;  /* デフォルトは 5 秒 */
};
```

### トランザクションの流れ {#transaction-flow}

```
1. jbd2_journal_start(): begin transaction
      → get handle_t for this transaction
      → reserve journal space (journal credits)

2. jbd2_journal_get_write_access(bh): mark buffer as modified
      → copy original buffer to journal (for undo)

3. Modify the buffer (update data)

4. jbd2_journal_dirty_metadata(bh): mark buffer dirty in journal

5. jbd2_journal_stop(): end transaction
      → if journal full or commit_interval elapsed:
          kjournald2 commits:
            → write descriptor + data blocks to journal
            → write commit record to journal (seals the transaction)
            → checkpoint: write data blocks to final location, mark journal free

6. On crash: journal replay restores consistency
```

## 遅延割り当て（delalloc） {#delayed-allocation-delalloc}

ext4はブロックの割り当てを実際のライトバックまで遅らせる。

```c
/* page が書き込まれたとき: */
/* 1. page をダーティにする。ブロックはまだ割り当てない */
/* 2. ライトバック時（ダーティの期限切れまたは sync）: */
ext4_writepages()
  → ext4_da_convert_inline_data()
  → mpage_map_and_submit_extent()
      → ext4_map_blocks(CREATE)  ← ここで初めてブロックを割り当てる
          → ext4_ext_map_blocks() → find/create extent
          → jbd2_journal_get_write_access()
```

利点は次のとおりである。

- すぐに削除される小さな書き込みは、ブロック割り当てをまったく必要としない
- 大きなシーケンシャル書き込みは連続したブロックを得られる（エクステントの効率が良くなる）
- ジャーナルへの負荷が減る（割り当てがまとめられる）

## オンライン機能 {#online-features}

```bash
# オンラインリサイズ（ファイルシステムを拡大する）
resize2fs /dev/sda1  # パーティション全体まで拡大する

# オンラインデフラグ（断片化したファイル向け）
e4defrag /data/myfile
e4defrag /data/  # ディレクトリをデフラグする

# ファイルシステムの健全性を確認する
e2fsck -n /dev/sda1   # ドライラン。修正はしない

# ジャーナルのコミット間隔を調整する（デフォルトは 5 秒）
tune2fs -o journal_data_ordered /dev/sda1
mount -o commit=60 /dev/sda1 /mnt  # 60 秒ごとにコミットする（安全性は下がるが高速）

# インラインデータを有効にする（小さなファイルを inode に直接格納する）
tune2fs -O inline_data /dev/sda1
```

## ext4の主要な統計 {#key-ext4-statistics}

```bash
# debugfs による ext4 の統計
debugfs /dev/sda1
> stats
> icheck <block_number>  # ブロックを所有する inode を探す
> ncheck <inode_number>  # inode の名前を探す
> blocks <inode_number>  # inode が使っているブロックを表示する

# ext4 の tracepoint
ls /sys/kernel/tracing/events/ext4/
echo 1 > /sys/kernel/tracing/events/ext4/ext4_da_write_begin/enable
cat /sys/kernel/tracing/trace_pipe

# ブロック割り当ての統計
cat /proc/fs/ext4/sda1/mb_groups  # バディアロケータのグループごとの状態
```

## 参考文献 {#further-reading}

- [tmpfsとramfs](tmpfs.md) — メモリをバックエンドとする代替
- [btrfs](btrfs.md) — Copy-on-Writeの代替
- [VFS: write()の一生](../vfs/life-of-write.md) — 書き込みがどのようにext4に届くか
- [Kernel docs: ext4 data structures and algorithms](https://docs.kernel.org/filesystems/ext4/index.html) — 詳細なオンディスクフォーマット
- [Kernel docs: ext4 high-level design](https://docs.kernel.org/filesystems/ext4/overview.html) — ブロック、エクステント、inode
