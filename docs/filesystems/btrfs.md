# Btrfs: B-treeファイルシステム {#btrfs-b-tree-filesystem}

> copy-on-writeのB-treeレイアウト、スナップショット、サブボリューム、RAID。OracleのChris Masonが作成し、Linux 2.6.29でメインラインカーネルにマージされた。設計の経緯については[A short history of btrfs](https://lwn.net/Articles/342892/)（LWN）を参照。

## なぜBtrfsか {#why-btrfs}

Btrfs（B-tree filesystem）は、ext4の限界に対処するために設計された。

| 機能 | ext4 | Btrfs |
|---------|------|-------|
| 最大ボリューム | 1 EiB | 16 EiB |
| チェックサム | なし | あり（CRC32C/xxhash/SHA256） |
| スナップショット | なし | あり（即時、書き込み可能） |
| サブボリューム | なし | あり（独立した名前空間） |
| RAID | mdraidのみ | RAID 0/1/5/6/10を内蔵 |
| 圧縮 | なし | zlib/lzo/zstdによるインライン圧縮 |
| 重複排除 | なし | オフライン（duperemove） |
| Send/receive | なし | あり（増分バックアップ） |
| Copy-on-write | なし（ジャーナリングを使う） | データ + メタデータ |

## B-treeの構造 {#b-tree-structure}

BtrfsではすべてがB-treeに格納される（それゆえ「B-tree filesystem」と呼ばれる）。この設計はReiserFSとZFSの概念に着想を得ている。独立したinodeテーブルやブロックビットマップは存在せず、すべてのメタデータはこれらのツリーの中にある。

```
Btrfs superblock
└── tree of tree roots
    ├── root tree     ← 他のすべてのツリーに関するメタデータ
    ├── extent tree   ← 空き領域とブロックの参照カウント
    ├── chunk tree    ← 論理 → 物理アドレスのマッピング
    ├── device tree   ← 物理デバイス
    ├── checksum tree ← データブロックのチェックサム
    └── [subvolume trees]
        ├── subvol 256 (default root)  ← ファイルとディレクトリ
        ├── subvol 257 (first snapshot)
        └── ...
```

### B-treeノードのフォーマット {#b-tree-node-format}

```c
/* include/uapi/linux/btrfs_tree.h */

/* すべてのノードはこのヘッダで始まる: */
struct btrfs_header {
    __u8    csum[BTRFS_CSUM_SIZE]; /* このブロックのチェックサム */
    __u8    fsid[BTRFS_FSID_SIZE]; /* ファイルシステムの UUID */
    __le64  bytenr;                /* このブロックの bytenr（論理アドレス） */
    __le64  flags;
    __u8    chunk_tree_uuid[BTRFS_UUID_SIZE];
    __le64  generation;            /* 書き込まれたときのトランザクション ID */
    __le64  owner;                 /* このノードが属するツリーの ID */
    __le32  nritems;               /* アイテム数 */
    __u8    level;                 /* 0 = リーフ、>0 = 内部ノード */
};

/* 内部ノード: (key, blockptr) ペアの配列 */
struct btrfs_key_ptr {
    struct btrfs_disk_key key;
    __le64 blockptr;           /* 子ノードのアドレス */
    __le64 generation;
};

/* リーフノード: アイテムとデータを両端から詰める */
struct btrfs_item {
    struct btrfs_disk_key key;  /* (objectid, type, offset) の 3 つ組 */
    __le32 offset;              /* リーフ末尾からのデータオフセット */
    __le32 size;                /* データサイズ */
};
/* アイテムのデータはリーフの末尾に、アイテム配列は先頭に置かれ、互いに中央へ向かって伸びる */
```

### キー {#keys}

Btrfsのすべてのツリーのすべてのアイテムは、3つの部分からなるキーを持つ。

```
(objectid, type, offset)
   ↑            ↑      ↑
 inode#    item type  varies

Examples:
(256, INODE_ITEM,    0)           → inode metadata
(256, INODE_REF,     parent_id)   → directory reference
(256, EXTENT_DATA,   file_offset) → file data mapping
(256, DIR_ITEM,      name_hash)   → directory entry
(256, XATTR_ITEM,    name_hash)   → extended attribute
```

## Copy-on-writeのセマンティクス {#copy-on-write-semantics}

Btrfsはデータをインプレースで上書きすることがない。すべての書き込みは新しいブロックを作る。

```
Write to file:
  1. Allocate new block(s) from extent tree
  2. Write new data to new block
  3. Update file's EXTENT_DATA item → new block address
  4. COW the leaf containing EXTENT_DATA
  5. COW all parent nodes up to root
  6. Update root pointer in superblock

Old blocks become unreferenced → freed in next transaction

Benefits:
  • Crash safety: superblock update is atomic, old tree still valid
  • Snapshots: share unchanged blocks between snapshots
  • No journal needed for metadata consistency
```

```
Before write:                After write:

Root ──→ [A]                 Root ──→ [A']  (new)
         │                            │
        [B]                          [B']  (new)
         │                            │
        [leaf: extent→blk1]          [leaf': extent→blk2]  (new)

blk1 still valid (could be shared by snapshot)
```

## サブボリュームとスナップショット {#subvolumes-and-snapshots}

**サブボリューム**は、独立してルートとなりうるB-tree（独自のルートinodeを持つ）である。

```bash
# サブボリュームを作成する:
btrfs subvolume create /data/myapp

# サブボリュームを一覧表示する:
btrfs subvolume list /data
# ID 256 gen 100 top level 5 path myapp
# ID 257 gen 102 top level 5 path myapp-snap-2024

# スナップショットを作成する（即時、CoW）:
btrfs subvolume snapshot /data/myapp /data/myapp-snap-2024
# 読み取り専用スナップショット:
btrfs subvolume snapshot -r /data/myapp /data/myapp-backup

# スナップショットを削除する:
btrfs subvolume delete /data/myapp-snap-2024

# デフォルトのサブボリューム（/ にマウントされるもの）を設定する:
btrfs subvolume set-default 256 /data
```

スナップショットの作成はO(1)である — B-treeのルートポインタをコピーするだけだからだ。共有されたブロックはエクステントツリーで参照カウントされる。

## エクステントツリー: 領域の管理 {#extent-tree-space-accounting}

```c
/* 割り当てられた各エクステントは後方参照を持つ: */
struct btrfs_extent_item {
    __le64  refs;          /* 参照カウント */
    __le64  generation;
    __le64  flags;         /* BTRFS_EXTENT_FLAG_DATA または _TREE_BLOCK */
};

/* インライン後方参照（所有者が 1 つの場合）: */
struct btrfs_extent_inline_ref {
    __u8    type;           /* BTRFS_EXTENT_DATA_REF_KEY など */
    __le64  offset;         /* type に依存する */
};
```

```bash
# エクステントの使用状況を表示する:
btrfs filesystem df /data
# Data, single: total=2.00GiB, used=1.23GiB
# Metadata, DUP: total=256.00MiB, used=12.50MiB
# System, DUP: total=8.00MiB, used=16.00KiB

# 詳細な領域情報:
btrfs filesystem usage /data

# スナップショット間で共有されているエクステントを表示する:
btrfs inspect-internal dump-tree -t extent /dev/sda1 | head -50
```

## チェックサム {#checksums}

Btrfsはすべてのデータブロックとすべてのメタデータブロックにチェックサムを付ける。

```bash
# デフォルトのチェックサム: crc32c（高速、ハードウェアアクセラレーション対応）
# 利用可能なもの: xxhash（より高速）、sha256、blake2b（衝突耐性あり）

# mkfs 時にチェックサムを指定する:
mkfs.btrfs --checksum xxhash /dev/sda1

# ファイルシステムを検査する:
btrfs scrub start /data          # すべてのチェックサムを検証する
btrfs scrub status /data         # 進捗を確認する
# scrub device /dev/sda1 (id 1) done
# total bytes scrubbed: 100.23GiB with 0 errors

# scrub は ext4 なら見逃すサイレントなビット腐敗を見つける!
```

```c
/* カーネルは読み出しのたびにチェックサムを検証する（fs/btrfs/disk-io.c）:
 * btrfs_validate_extent_buffer() はエクステントバッファのデータに対して
 * チェックサムを計算し、バッファヘッダの先頭バイトに格納された
 * チェックサムと比較する。
 * fs_info->csum_type を使って正しいハッシュ（crc32c、xxhash64、
 * sha256、blake2b）にディスパッチする。 */
```

## インライン圧縮 {#inline-compression}

```bash
# 圧縮を有効にしてマウントする:
mount -o compress=zstd /dev/sda1 /data      # zstd（圧縮率と速度のバランスが最良）
mount -o compress=lzo /dev/sda1 /data       # lzo（最速）
mount -o compress=zlib /dev/sda1 /data      # zlib（圧縮率が最良）
mount -o compress-force=zstd /dev/sda1 /data # 圧縮できないデータでも強制する

# ファイルごとの圧縮:
btrfs property set /data/bigfile compression zstd

# 圧縮率を確認する:
compsize /data/bigfile
# Type       Perc     Disk Usage   Uncompressed Referenced
# TOTAL       42%      420M         1.00G        1.00G
# zstd        42%      420M         1.00G        1.00G
```

## Send/receive（増分バックアップ） {#sendreceive-incremental-backups}

```bash
# フルバックアップ:
btrfs subvolume snapshot -r /data/myapp /data/myapp-snap-1
btrfs send /data/myapp-snap-1 | btrfs receive /backup/

# 増分: snap-1 以降に変更されたブロックのみ
btrfs subvolume snapshot -r /data/myapp /data/myapp-snap-2
btrfs send -p /data/myapp-snap-1 /data/myapp-snap-2 | btrfs receive /backup/
# snap-1 と snap-2 の差分だけを送る

# SSH 経由:
btrfs send /data/myapp-snap-2 | ssh backuphost "btrfs receive /backup/"
```

## BtrfsのRAID {#raid-in-btrfs}

```bash
# RAID 1（メタデータとデータをミラーリング）を作成する:
mkfs.btrfs -d raid1 -m raid1 /dev/sda /dev/sdb

# 既存のファイルシステムにデバイスを追加する:
btrfs device add /dev/sdc /data
btrfs balance start -dconvert=raid1 -mconvert=raid1 /data

# RAID の状態を表示する:
btrfs filesystem show /data

# 故障したデバイスを交換する:
btrfs replace start /dev/sdb /dev/sdd /data
btrfs replace status /data
```

## トランザクションモデル {#transaction-model}

Btrfsはジャーナルを持たない、トランザクションベースのモデルを使う。

```c
/* fs/btrfs/transaction.c */
struct btrfs_trans_handle {
    u64             transid;         /* トランザクション ID */
    u64             bytes_reserved;  /* 予約された領域 */
    struct btrfs_transaction *transaction;
};

/* トランザクションを開始する（領域を予約する）: */
trans = btrfs_start_transaction(root, 10); /* 10 ブロック分を予約する */

/* B-tree のアイテムを変更する（例: inode ref を挿入する）: */
btrfs_insert_inode_ref(trans, root, name, namelen, objectid, dir_objectid, index);

/* コミットする（ダーティなノードをすべて COW し、スーパーブロックを更新する）: */
btrfs_commit_transaction(trans);
```

スーパーブロックは最後に書き込まれ、tree-of-rootsのルートを含む。スーパーブロックが更新されるまで古いツリーは無傷のままであり、どの時点でクラッシュしても直前のトランザクションの状態が有効なまま残る。

## デフラグ {#defragmentation}

CoWによる断片化は時間とともに蓄積する。

```bash
# ファイルをデフラグする（エクステントをシーケンシャルに書き直す）:
btrfs filesystem defragment /data/bigfile

# 圧縮しながらデフラグする:
btrfs filesystem defragment -czstd /data/bigfile

# サブボリューム全体をデフラグする（警告: スナップショットとの CoW 共有が壊れる!）:
btrfs filesystem defragment -r /data/myapp

# マウント時の自動デフラグ:
mount -o autodefrag /dev/sda1 /data
```

## 可観測性 {#observability}

```bash
# ファイルシステムの統計:
btrfs filesystem show
btrfs filesystem df /data
btrfs filesystem usage /data

# ツリーの調査（低レベル）:
btrfs inspect-internal dump-tree /dev/sda1       # すべてのツリー
btrfs inspect-internal dump-tree -t fs /dev/sda1 # サブボリュームツリー
btrfs inspect-internal dump-super /dev/sda1       # スーパーブロック

# 論理アドレスを所有するファイルを探す:
btrfs inspect-internal logical-resolve 12345678 /data

# エラーを検査する（オフライン、非破壊）:
btrfs check --readonly /dev/sda1

# 統計: デバイスごとの読み書きエラー
btrfs device stats /data

# 性能のトレーシング:
bpftrace -e 'kprobe:btrfs_file_write_iter { @[comm] = count(); }'
```

## 参考文献 {#further-reading}

- [ext4ジャーナリング（JBD2）](ext4-journal.md) — ジャーナルベースとの対比
- [ページキャッシュ](../mm/page-cache.md) — BtrfsのCoW書き込みが経由するページキャッシュ
- [Copy-on-Write](../mm/cow.md) — プロセスのCoWとファイルシステムのCoW
- [xattr](../vfs/xattr.md) — Btrfsはxattrをサブボリュームツリーに格納する
- [Kernel docs: btrfs](https://docs.kernel.org/filesystems/btrfs.html) — 機能の概要と、btrfsドキュメントへの入口
- [btrfs feature status](https://btrfs.readthedocs.io/en/latest/Status.html) — RAID5/6に関する注意点を含む、機能ごとの安定性
- [A short history of btrfs (LWN)](https://lwn.net/Articles/342892/) — Valerie Auroraによる設計の根拠
- `fs/btrfs/` — Btrfsの実装
