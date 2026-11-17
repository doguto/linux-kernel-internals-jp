# スケジューリングと並行性 {#scheduling-concurrency}

カーネルが競合するタスク間でCPUをどのように共有し、並行コードの正しさをどのように保つか。

- [スケジューラ（sched/）](../sched/README.md) — 次にどのタスクをいつ実行するかをカーネルが決める仕組み
- [ロック（locking/）](../locking/README.md) — 並行コードの正しさを保つプリミティブ: spinlock、mutex、RCU
- [割り込み（interrupts/）](../interrupts/README.md) — ハードウェアイベントの処理と、作業の安全な遅延実行
- [IPC（ipc/）](../ipc/README.md) — プロセス間の通信方法: パイプ、シグナル、共有メモリ、メッセージキュー
