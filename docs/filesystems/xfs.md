# XFS: 高性能ジャーナリングファイルシステム {#xfs-high-performance-journaling-filesystem}

> アロケーショングループ、B-treeインデックス構造、遅延割り当て、オンライン修復

## なぜXFSか {#why-xfs}

XFSはもともと1993年にSGIでIRIX向けに開発され、その後Steve LordとSGIチームによってLinuxへ移植された。メインラインカーネルにはLinux 2.5.36（2002年）でマージされた。XFSが優れているのは次の点である。

- **大きなファイルとファイルシステム**: 最大8 EiBのボリューム、8 EiBのファイル
- **高い並行性**: アロケーショングループごとのロックにより、グローバルな競合が最小限
- **メタデータ性能**: エクステント、inode、空き領域に対するB-treeインデックス
- **オンライン操作**: マウントしたままでのデフラグ、拡張、修復

| 機能 | ext4 | XFS | Btrfs |
|---------|------|-----|-------|
| 最大ファイルシステムサイズ | 1 EiB | 8 EiB | 16 EiB |
| 最大ファイルサイズ | 16 TiB | 8 EiB | 16 EiB |
| 遅延割り当て | あり | あり | あり |
| ジャーナル | JBD2 | 内蔵（xlog） | なし（CoW） |
| チェックサム | メタデータのみ | メタデータ + ディレクトリ | 全ブロック |
| オンライン修復 | なし | あり（5.19以降） | あり（btrfs scrub） |
| Reflink | なし | あり | あり |
| クォータ | あり | あり | あり |
| リアルタイムセクション | なし | あり | なし |

## アロケーショングループ {#allocation-groups}

XFSはファイルシステムを**アロケーショングループ（AG）**に分割する。各AGは自分のinode、空き領域、B-treeを独立して管理する。

```
XFS filesystem:
┌──────────────────────────────────────────────────────────┐
│ Superblock │ AG 0 │ AG 1 │ AG 2 │ AG 3 │ ... │ AG N-1    │
└──────────────────────────────────────────────────────────┘
              Each AG:
              ┌────────────────────────────────────────┐
              │ AG superblock (agf + agi + agfl)       │
              │ Free space B-tree 1 (by block number)  │
              │ Free space B-tree 2 (by block count)   │
              │ Inode B-tree                           │
              │ Free inode B-tree                      │
              │ Reverse mapping B-tree (rmapbt)        │
              │ Reference count B-tree (refcntbt)      │
              │ [Data blocks and inodes]               │
              └────────────────────────────────────────┘
```

AGのサイズはデフォルトでおよそ1 GiB（ファイルシステムサイズ / 4以下の2のべき乗）である。AGの数が並列度を決める。

```bash
# AG の数とサイズを確認する:
xfs_info /dev/sda1
# meta-data=/dev/sda1    isize=512  agcount=16, agsize=6553600 blks
#          =             sectsz=512   attr=2, projid32bit=1
#          =             crc=1        finobt=1, sparse=1, rmapbt=1
# data     =             bsize=4096   blocks=104857600, imaxpct=25
# naming   =version 2    bsize=4096   ascii-ci=0, ftype=1
# log      =internal     bsize=4096   blocks=51200, version=2
#          =             sectsz=512   sunit=0 blks, lazy-count=1
# realtime =none         extsz=4096   blocks=0, rtextents=0
```

## B-tree構造 {#b-tree-structures}

### 空き領域B-tree（bnobtとcntbt） {#free-space-b-trees-bnobt-and-cntbt}

2つのB-treeが空き領域を追跡し、O(log n)での割り当てを可能にする。

```c
/* 空きエクステントのレコード: */
struct xfs_alloc_rec {
    __be32  ar_startblock;  /* 開始ブロック番号 */
    __be32  ar_blockcount;  /* 空きブロック数 */
};
/* bnobt: startblock をキーとする（ヒント付近での連続割り当て用） */
/* cntbt: blockcount をキーとする（サイズによるベストフィット割り当て用） */
```

```bash
# 空き領域 B-tree をダンプする:
xfs_db -c 'ag 0' -c 'agf' -c 'print' /dev/sda1
# magicnum = 0x58414746   (XAGF)
# versionnum = 1
# seqno = 0
# length = 6553600
# bnoroot = 2
# cntroot = 3
# bnolevel = 2
# cntlevel = 2
# freeblks = 5234567
```

### inode B-tree（inobt）と空きinode B-tree（finobt） {#inode-b-tree-inobt-and-free-inode-b-tree-finobt}

```c
/* inode チャンクのレコード: チャンクあたり 64 inode */
struct xfs_inobt_rec {
    __be32  ir_startino;   /* チャンク内の最初の inode 番号 */
    __be16  ir_holemask;   /* inode チャンク内のホール（スパース inode） */
    __u8    ir_count;      /* このレコード内の inode 数 */
    __u8    ir_freecount;  /* 空き inode 数 */
    __be64  ir_free;       /* 空き inode のビットマスク */
};
/* finobt: 同じフォーマットだが、ir_freecount > 0 のレコードのみ */
/* finobt により、inobt を走査する代わりに O(log n) で空き inode を検索できる */
```

### 逆マッピングB-tree（rmapbt） {#reverse-mapping-b-tree-rmapbt}

Linux 4.8で導入された。割り当てられたすべてのブロックが、その所有者への後方参照を持つ。

```c
struct xfs_rmap_rec {
    __be32  rm_startblock; /* エクステントの開始ブロック */
    __be32  rm_blockcount; /* エクステントの長さ */
    __be64  rm_owner;      /* 所有者（inode 番号、またはメタデータの場合は XFS_RMAP_OWN_*） */
    __be64  rm_offset;     /* ファイルオフセット（データエクステントの場合） */
    /* フラグ: unwritten、attr fork、bmbt ブロック */
};
```

rmapbtにより、オンライン修復（破損したブロックをどのinodeが所有しているかを見つける）、オンラインscrub、reflinkのCOW追跡が可能になる。

## inodeのフォーマット {#inode-format}

XFSのinodeは512バイトである（256または2048に設定可能）。

```c
struct xfs_dinode {
    __be16  di_magic;      /* 0x494e "IN" */
    __be16  di_mode;       /* ファイルタイプ + パーミッション */
    __u8    di_version;    /* inode のバージョン 1/2/3 */
    __u8    di_format;     /* XFS_DINODE_FMT_* */
    __be64  di_size;
    __be64  di_nblocks;    /* データ + 属性ブロックの数 */
    __be32  di_extsize;    /* エクステントサイズのヒント */
    __be32  di_nextents;   /* データエクステントの数 */
    __be16  di_anextents;  /* 属性エクステントの数 */
    __be64  di_atime;
    __be64  di_mtime;
    __be64  di_ctime;
    __be64  di_crtime;     /* v3: 作成時刻 */
    __be64  di_ino;        /* v3: inode 番号（自己識別） */
    uuid_t  di_uuid;       /* v3: ファイルシステムの UUID */
    /* ... チェックサム、フラグ ... */
    /* この後に続くもの: データフォーク（エクステント、B-tree のルート、またはインラインデータ） */
};

/* di_format の値: */
/* XFS_DINODE_FMT_DEV:    特殊ファイル。デバイス番号をインラインで持つ */
/* XFS_DINODE_FMT_LOCAL:  データをインラインで持つ（小さなファイル／ディレクトリ） */
/* XFS_DINODE_FMT_EXTENTS: エクステントリストが inode に収まる */
/* XFS_DINODE_FMT_BTREE:  inode をルートとする B-tree（エクステントが多い場合） */
```

### エクステントのフォーマット {#extent-format}

XFSはエクステントをコンパクトな128ビットのフォーマットで格納する。

```c
/* xfs_bmbt_rec: 128 ビットにパックされたエクステントレコード */
/* ビット:
   [127]      unwritten フラグ
   [126:73]   ブロック単位のファイルオフセット（54 ビット）
   [72:21]    開始ブロック（52 ビット）
   [20:0]     ブロック数（21 ビット。エクステントあたり最大 2M ブロック）
*/
```

最大約20個のエクステントがinode内にインラインで収まる。それを超えると、inodeをルートとするB-tree（bmbt）が使われる。

## 遅延割り当て {#delayed-allocation}

XFSは書き込み性能を向上させるために**遅延割り当て**を使う。書き込み時にすぐディスクブロックを割り当てる代わりに、XFSはメモリ上で領域を予約し、実際の割り当てをライトバックまで遅らせる。

```
write() → page cache → dirty pages
                           ↓ (delayed: no block allocation yet)
                    speculative reservation (in-memory)
                           ↓ (at writeback time)
                    actual allocation from cntbt
                    extent map updated
                    blocks written to disk
```

利点は次のとおりである。

- 連続した書き込みを大きなエクステントにまとめられる
- より多くの文脈が得られた時点（ファイルサイズ全体が分かった時点）で割り当てが行われる
- エクステントがより少なく、より大きくなる → 断片化が減る

```bash
# 割り当ての統計を監視する:
xfs_io -c 'statfs' /data
# geometry.rtextsize = 0
# geometry.rtblocks = 0

# 断片化を確認する:
xfs_db -c 'freesp -s' /dev/sda1
# from      to extents  blocks    pct
#    1       1  123456  123456   5.40
#    2       3   23456   46912   2.06
# ...

# デフラグする:
xfs_fsr /dev/sda1       # ファイルシステムの再編成ツール
```

## XFSのジャーナル（xlog） {#xfs-journal-xlog}

XFSはJBD2とは別の、独自のジャーナリング（xlog）を使う。

```
Journal write sequence:
  1. Begin transaction (xfs_trans_alloc)
  2. Lock and log modified buffers (xfs_buf_item_log)
  3. Commit transaction → write log records to circular journal
  4. Async: CIL (Committed Item List) batches commits
  5. Checkpoint: write all modified metadata to its real location
  6. Log tail advances: journal space is freed

On crash recovery:
  xlog_recover() replays committed transactions since last checkpoint
```

### Committed Item List（CIL） {#committed-item-list-cil}

CILはJBD2の実行中トランザクションに相当するXFSの仕組みだが、フラッシュが遅延される。遅延ロギング（ログインテントアイテム）はDave Chinnerによって設計・実装された。

```c
/* 複数のトランザクションが 1 回の CIL プッシュにまとめられる: */
/* - xfs_log_commit_cil() がメモリ上の CIL にアイテムを追加する */
/* - CIL がしきい値（ログの 1/4）を超えると、ディスクへプッシュする */
/* - 多数のトランザクションに対して 1 回の xlog_write() → I/O が減る */
```

```bash
# ジャーナルの統計:
xfs_logprint -t /dev/sda1 | head -30

# ジャーナルのサイズを確認する:
xfs_info /dev/sda1 | grep log
# log  =internal   bsize=4096  blocks=51200, version=2
# ジャーナル: 51200 * 4096 = 200MB の内部ログ
```

## オンライン修復（XFS 5.19以降） {#online-repair-xfs-519}

XFS独自の機能として、マウントしたままファイルシステムを修復できる。

```bash
# オンライン scrub: 不整合を検査する（読み取り専用、安全）:
xfs_scrub /data
# Phase 1: Check internal metadata.
# Phase 2: Scan directory tree.
# Phase 3: Check directory connectivity.
# Phase 4: Verify inode disk data.
# Phase 5: Check summary counters.

# 積極的: 見つかった問題も修復する:
xfs_scrub -n /data   # ドライラン: 報告するだけ
xfs_scrub /data      # インプレースで修復する

# 従来のオフライン修復（アンマウント状態）:
xfs_repair /dev/sda1
```

scrubはB-tree同士を相互参照することで動作する。rmapbtは割り当てられたすべてのブロックを知っているので、scrubberはエクステントB-treeが参照するすべてのブロックがrmapbtにも存在することを検証できる。

## クォータ {#quotas}

```bash
# マウント時にクォータを有効にする:
mount -o uquota /dev/sda1 /data     # ユーザークォータ
mount -o gquota /dev/sda1 /data     # グループクォータ
mount -o pquota /dev/sda1 /data     # プロジェクトクォータ（ディレクトリ用）

# 制限を設定する:
xfs_quota -x -c 'limit bsoft=1g bhard=2g user1' /data
xfs_quota -x -c 'limit -p bsoft=10g bhard=20g 100' /data  # プロジェクト 100

# レポート:
xfs_quota -c 'quota -h user1' /data
xfs_quota -c 'df -h' /data

# プロジェクトクォータ: ディレクトリツリーを追跡する
echo "100:/data/project1" >> /etc/projects
echo "100:Project One" >> /etc/projid
xfs_quota -x -c 'project -s 100' /data  # ディレクトリをプロジェクトに割り当てる
```

## 性能チューニング {#performance-tuning}

```bash
# 特定のワークロードに最適な mkfs:
mkfs.xfs \
    -b size=4096 \          # ブロックサイズ（デフォルトは 4K）
    -l size=256m \          # ジャーナルサイズ（大きいほどバーストに強い）
    -d agcount=32 \         # 32 AG = 32 並列
    -i size=512 \           # inode サイズ（512B。小さなファイルなら 256B）
    /dev/sda1

# マウントオプション:
mount -o noatime,largeio,inode64 /dev/sda1 /data
# noatime:  atime の更新を省く（書き込みが減る）
# largeio:  ストライプ構成のストレージ向けに 512KB の I/O を使う
# inode64:  inode をファイルシステム全体に割り当てる（先頭 1TB だけではなく）

# アロケータへのヒント: RAID のジオメトリを XFS に伝える
mkfs.xfs -d su=64k,sw=4 /dev/md0
# su=ストライプユニット（64K）、sw=ストライプ幅（4 ディスク）
```

## 可観測性 {#observability}

```bash
# XFS の統計:
cat /proc/fs/xfs/stat
# extent_alloc 12345 678 90123 456
# abt 0 0 0 0       ← 割り当て B-tree の操作
# blk_map ...
# bmbt ...          ← エクステント B-tree の操作
# dir ...           ← ディレクトリ操作
# trans ...         ← トランザクションの統計

# I/O の統計:
xfs_io -c 'fiemap' /data/bigfile  # ファイルのエクステントマップを表示する

# XFS の B-tree 操作をトレースする:
bpftrace -e 'kprobe:xfs_alloc_ag_vextent { @[probe] = count(); }'

# inode 数を確認する:
df -i /data

# 詳細な領域使用状況:
xfs_db -c 'freesp -d' /dev/sda1
```

## 参考文献 {#further-reading}

- [ext4ジャーナリング（JBD2）](ext4-journal.md) — 別のジャーナリング手法
- [Btrfs](btrfs.md) — CoWファイルシステムとの比較
- [blk-mq](../block/blk-mq.md) — XFSがI/Oを発行する際に経由するブロックレイヤ
- [ページキャッシュ](../mm/page-cache.md) — XFSはバッファードI/Oにページキャッシュを使う
- [Kernel docs: XFS](https://docs.kernel.org/filesystems/xfs/index.html) — XFSの開発者向けドキュメント
- [Kernel docs: XFS delayed logging design](https://docs.kernel.org/filesystems/xfs/xfs-delayed-logging-design.html) — ログのアーキテクチャの詳細
- [Kernel docs: XFS self-describing metadata](https://docs.kernel.org/filesystems/xfs/xfs-self-describing-metadata.html) — ブロックごとのCRCを持つv5オンディスクフォーマット
- `fs/xfs/` — XFSの実装
- `xfsprogs` — mkfs.xfs、xfs_info、xfs_repair、xfs_scrub、xfs_io
