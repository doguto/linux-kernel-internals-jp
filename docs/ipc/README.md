# IPC: プロセス間通信 {#ipc-inter-process-communication}

> プロセス同士がデータを交換し、協調動作するための仕組み

## IPC機構の概要 {#ipc-mechanisms-overview}

| 機構 | 通信形態 | 方向 | データサイズ | 永続性 |
|-----------|--------------|-----------|-----------|-------------|
| シグナル | イベント通知 | 一方向 | シグナル番号のみ | 配送されるまで |
| パイプ（無名） | バイトストリーム | 単方向 | バッファあり（~64KB） | 両端が閉じられるまで |
| FIFO（名前付きパイプ） | バイトストリーム | 単方向 | バッファあり | ファイルが存在する間 |
| Unixソケット | ストリームまたはデータグラム | 双方向 | 柔軟 | fdが開いている間 |
| POSIX共有メモリ | 生のメモリ | 双方向 | 任意 | shm_unlinkまで |
| POSIXメッセージキュー | 型付きメッセージ | 双方向 | max_msg_size | mq_unlinkまで |
| eventfd | カウンタ + 通知 | 双方向 | 8バイトのカウンタ | すべてのfdが閉じられるまで |
| signalfd | fdとしてのシグナル | 一方向 | siginfo_t | fdが開いている間 |

## このセクションのページ {#pages-in-this-section}

| ページ | 扱う内容 |
|------|----------------|
| [シグナル](signals.md) | シグナル配送、sigaction、マスク、リアルタイムシグナル |
| [パイプとFIFO](pipes.md) | パイプの内部構造、splice、vmsplice |
| [共有メモリとセマフォ](shared-memory.md) | POSIX shm、SysV shm、セマフォ、eventfd |
| [Unixドメインソケット](unix-sockets.md) | SOCK_SEQPACKET、抽象名前空間、SCM_RIGHTS、SO_PEERCRED |
| [SysV IPC: セマフォとメッセージキュー](sysv-semaphores.md) | semget/semop、msgget/msgsnd、SEM_UNDO、リーク防止 |
| [IPC事件簿](war-stories.md) | リークしたセマフォ、fd枯渇、シグナルストーム、パイプの意外な挙動 |
