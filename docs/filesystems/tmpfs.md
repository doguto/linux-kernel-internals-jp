# tmpfsとramfs {#tmpfs-and-ramfs}

> 永続ストレージを持たない、メモリをバックエンドとするファイルシステム

## tmpfsとは何か {#what-tmpfs-is}

tmpfsは完全にRAM（および任意でスワップ）上に存在するファイルシステムである。ページキャッシュをバックエンドとしており、各ファイルのデータは無名メモリで、メモリ逼迫時にはスワップへ追い出すことができる[(kernel docs)](https://www.kernel.org/doc/html/latest/filesystems/tmpfs.html)。現在の実装は2001年ごろにChristoph Rohlandによって作られ、古いshmファイルシステムを置き換えた。

```bash
# tmpfs をマウントする
mount -t tmpfs -o size=512m tmpfs /mnt/mytmpfs

# 一般的な用途:
# /tmp — 一時ファイル
# /run — 実行時データ（PID ファイル、ソケット）
# /dev/shm — POSIX 共有メモリ（shm_open）
# コンテナのオーバーレイ — Docker コンテナの上位レイヤ
# /var/cache — ビルドキャッシュ
```

## tmpfsとramfsの比較 {#tmpfs-vs-ramfs}

| | tmpfs | ramfs |
|---|-------|-------|
| サイズ制限 | あり（設定可能） | なし（OOMになりうる） |
| スワップ | あり（pageはスワップへ追い出される） | なし（常にRAM上） |
| VFS操作 | 完全（truncateなど） | 完全 |
| 永続性 | なし（umountで消える） | なし（umountで消える） |
| shmemバックエンド | あり | なし |
| 用途 | /tmp、/run、shm | スワップのない組み込み |

## tmpfsの実装 {#tmpfs-implementation}

tmpfsは`shmem`（共有メモリ）サブシステムの上に構築されている。

```c
/* mm/shmem.c */
static const struct file_system_type shmem_fs_type = {
    .name           = "tmpfs",
    .init_fs_context = shmem_init_fs_context,
    .kill_sb        = kill_litter_super,
    .fs_flags       = FS_USERNS_MOUNT,
};
```

### shmemのinode {#shmem-inode}

```c
struct shmem_inode_info {
    spinlock_t          lock;
    unsigned int        seals;          /* memfd 用の F_SEAL_* */
    unsigned long       flags;
    unsigned long       alloced;        /* 割り当て済みの page 数 */
    unsigned long       swapped;        /* スワップアウトされた page 数 */
    pgoff_t             fallocend;      /* fallocate 範囲の終端 */
    struct list_head    shrinklist;     /* スワップを持つ inode のリスト */
    struct list_head    swaplist;       /* グローバルなスワップリストへのリンク */
    struct shared_policy policy;        /* NUMA ポリシー */
    struct simple_xattrs xattrs;
    atomic_long_t       i_direct_seals;
    struct inode        vfs_inode;      /* 必ず最後に置くこと */
};
```

### tmpfsのページフォールトパス {#page-fault-path-for-tmpfs}

プロセスがtmpfsのファイルから読み出し、そのpageがメモリ上にない場合は次のようになる。

```c
/* mm/shmem.c: shmem_fault() → shmem_get_folio() */
shmem_get_folio(inode, index, ...)
  → filemap_get_folio()   /* ページキャッシュを確認 */
  → if not found:
      shmem_alloc_and_add_folio()
          → alloc_folio(GFP_HIGHUSER)
          → /* page がスワップされていた場合: swap_read_folio() */
          → /* それ以外: page をゼロ埋めする */
          → add_to_page_cache()
```

スワップアウトされていたpageについては、`shmem_fault`が戻る前にスワップから読み戻す。これはアプリケーションからは透過的である。

### サイズ制限 {#size-limits}

```bash
# デフォルト: RAM の半分
mount -t tmpfs tmpfs /tmp

# 明示的なサイズ指定
mount -t tmpfs -o size=1g tmpfs /tmp       # 1GB のハードリミット
mount -t tmpfs -o size=50% tmpfs /tmp      # RAM の 50%

# nr_inodes の制限
mount -t tmpfs -o size=512m,nr_inodes=100k tmpfs /tmp

# 現在の使用量を確認する
df -h /tmp
cat /proc/mounts | grep tmpfs

# tmpfs の統計
cat /proc/meminfo | grep -E "Shmem|Swap"
# Shmem:           524288 kB  ← tmpfs の合計使用量
```

### NUMAポリシー {#numa-policy}

```bash
# 優先する NUMA ノードを指定してマウントする
mount -t tmpfs -o mpol=prefer:0 tmpfs /tmp  # ノード 0 を優先

# ノード間でインターリーブする
mount -t tmpfs -o mpol=interleave tmpfs /tmp

# 特定のノードにバインドする
mount -t tmpfs -o mpol=bind:0-1 tmpfs /tmp
```

## devtmpfs: /devファイルシステム {#devtmpfs-dev-filesystem}

devtmpfsは、udevが動き出す前にカーネルがデバイスノードを配置する特別なtmpfsである。

```bash
mount | grep devtmpfs
# devtmpfs on /dev type devtmpfs (rw,nosuid,size=...)

# デバイスが検出されるにつれてカーネルがノードを作成する:
ls -la /dev/sda /dev/null /dev/tty
# brw-rw---- root disk  8, 0  /dev/sda   ← ブロックデバイス（major=8, minor=0）
# crw-rw-rw- root root  1, 3  /dev/null  ← キャラクタデバイス
# crw-rw-rw- root tty   5, 0  /dev/tty
```

## tmpfsとLinuxの/proc、/sys {#tmpfs-and-linuxs-proc-sys}

`/proc`と`/sys`はいずれも、同じ基盤の上に構築された特別な疑似ファイルシステムである。

```c
/* proc_fs_type */
static struct file_system_type proc_fs_type = {
    .name           = "proc",
    .init_fs_context = proc_init_fs_context,
    .kill_sb        = proc_kill_sb,
    .fs_flags       = FS_USERNS_MOUNT,
};

/* kernfs の上に構築された sysfs */
static struct file_system_type sysfs_fs_type = {
    .name           = "sysfs",
    .init_fs_context = sysfs_init_fs_context,
    .kill_sb        = sysfs_kill_sb,
    .fs_flags       = FS_USERNS_MOUNT,
};
```

これらのファイルシステムはバッキングストレージを使わない — 読み出しはカーネルのデータ構造からデータを合成し、書き込みはカーネルの操作を引き起こす。

## 揮発性マウント: コンテナでのtmpfs {#volatile-mounts-tmpfs-in-container-use}

コンテナは書き込み可能なレイヤや一時的なストレージにtmpfsを使う。

```bash
# Docker: コンテナ内の /tmp に tmpfs をマウントする
docker run --tmpfs /tmp:size=512m,mode=1777 ubuntu bash

# Kubernetes: medium: Memory を指定した emptyDir
# Pod の一時ボリューム用に tmpfs マウントを作成する
```

## 参考文献 {#further-reading}

- [ext4](ext4.md) — 永続的な代替
- [btrfs](btrfs.md) — CoWの永続的な代替
- [メモリ管理: mmap](../mm/mmap.md) — tmpfsのpageがどのようにフォールトインされるか
- [IPC: 共有メモリ](../ipc/shared-memory.md) — tmpfsをバックエンドとするPOSIX shm
- [Kernel docs: tmpfs](https://docs.kernel.org/filesystems/tmpfs.html) — マウントオプション（`size`、`nr_inodes`、`huge`）とセマンティクス
