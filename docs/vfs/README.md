# 仮想ファイルシステム（VFS） {#virtual-filesystem-vfs}

> カーネルのファイルシステム抽象化レイヤ

## VFSとは何か {#what-is-vfs}

Virtual Filesystem Switch（VFS）は、カーネルが多種多様なファイルシステム（ext4、btrfs、tmpfs、procfs、NFS...）を共通のインターフェースを通じてサポートできるようにする抽象化レイヤである。プログラムが`read()`を呼ぶと、その呼び出しはVFSを経由し、VFSが適切なファイルシステム実装へディスパッチする。

VFSによって次のことが可能になる:

- 単一の`open()`、`read()`、`write()` APIが、ext4、NFS、/proc、/sys、パイプ、ソケットのいずれでも動作する
- マウントされたファイルシステムをまたいで、ファイルに透過的にアクセスできる
- ファイルシステムをカーネルモジュールとして開発できる

## 4つのVFSオブジェクト {#the-four-vfs-objects}

VFSは4つの中核オブジェクトを定義している:

| オブジェクト | 表すもの | カーネル構造体 |
|--------|-----------|---------------|
| Superblock | マウントされたファイルシステムのインスタンス | `struct super_block` |
| Inode | ファイルまたはディレクトリ | `struct inode` |
| Dentry | パスの構成要素（名前 → inodeの対応） | `struct dentry` |
| File | オープンされたファイル（open()が1つ作成する） | `struct file` |

それぞれに、ファイルシステムが実装しなければならない操作テーブル（vtable）が関連付けられている:

| オブジェクト | 操作構造体 |
|--------|------------------|
| Superblock | `struct super_operations` |
| Inode | `struct inode_operations` |
| Dentry | `struct dentry_operations` |
| File | `struct file_operations` |

## 目次 {#contents}

### 基礎 {#fundamentals}
- [VFSオブジェクト: superblock、inode、dentry、file](vfs-objects.md) — 4つの中核オブジェクトの詳細
- [パス解決](namei.md) — `/usr/bin/bash`がどのようにinodeになるか
- [ファイル操作](file-ops.md) — file_operations vtableとopen/read/write
- [ファイルシステムの登録とマウント](mounting.md) — ファイルシステムがVFSにどう組み込まれるか

### ライフサイクル {#lifecycle}
- [write()システムコールの一生](life-of-write.md) — write()からディスクまで
- [dentryキャッシュとinodeキャッシュ](dcache-icache.md) — 性能のためのキャッシング

### 事件簿 {#war-stories}
- [事件簿](war-stories.md) — dentryキャッシュ、`seq_file`、OverlayFSのcopy-upパスに由来する4件のCVE
