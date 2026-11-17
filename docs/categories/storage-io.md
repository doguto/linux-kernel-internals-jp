# ストレージとI/O {#storage-io}

アプリケーションのreadやwriteから物理デバイスに至るまでデータがたどる経路 — そしてそれを汎用的かつ高速にする各層。

- [VFS（vfs/）](../vfs/README.md) — 1つのAPIですべてのファイルシステムを駆動できるようにする抽象化
- [ファイルシステム（filesystems/）](../filesystems/README.md) — ディスク上のフォーマットとページキャッシュがバイト列をファイルに変える仕組み
- [ブロック層（block/）](../block/README.md) — ファイルシステムの要求から物理デバイスまでの経路
- [I/Oパターン（io/）](../io/README.md) — バッファードI/OとダイレクトI/O、同期と非同期、そしてそれらの間のトレードオフ
- [io_uring（io-uring/）](../io-uring/README.md) — 高性能な非同期I/Oのためのリングベースのインターフェース
