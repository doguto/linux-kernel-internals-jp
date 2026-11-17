# システムコール {#system-calls}

> ユーザー空間とカーネルの間の契約

## システムコールとは何か {#what-system-calls-are}

システムコールは、カーネルへの制御されたエントリポイントである。システムコールは次のことを行う。

1. **CPUの状態を保存する** — レジスタ、フラグ
2. **特権レベルを切り替える** — ユーザーモード（ring 3）→ カーネルモード（ring 0）
3. **引数を検証する** — ポインタをチェックし、ユーザー空間からデータをコピーする
4. **カーネルコードを実行する** — 実際の処理
5. **ユーザー空間に戻る** — レジスタを復元し、ring 3に戻る

x86-64では、`syscall` 命令がこの遷移を引き起こす。カーネルは `rax` の値をsyscall番号として使い、適切なハンドラにディスパッチする。

```
Userspace                    Kernel
─────────────────────────────────────────────────────
glibc: write(fd, buf, len)
  │
  │  mov rax, 1  (SYS_write)
  │  mov rdi, fd
  │  mov rsi, buf
  │  mov rdx, len
  │  syscall
  │              ──────────→  entry_SYSCALL_64
  │                           saves registers
  │                           calls sys_write()
  │                               → vfs_write()
  │              ←──────────  returns result in rax
  │
  │  return rax (bytes written, or -errno)
```

## このセクションのページ {#pages-in-this-section}

| ページ | 内容 |
|------|----------------|
| [syscallのエントリパス](syscall-entry.md) | x86-64のエントリ、特権レベルの切り替え、引数の受け渡し、vDSO |
| [SYSCALL_DEFINEとディスパッチ](syscall-define.md) | syscallの定義方法、ディスパッチテーブル、ABI |
| [新しいsyscallの追加](adding-syscall.md) | カーネルコントリビュータ向けのステップバイステップ解説 |
| [32ビット互換syscall](compat.md) | 64ビットカーネル上での32ビットユーザー空間のサポート。compat型とヘルパー |
| [ptraceとsyscallのインターセプト](ptrace-interception.md) | strace、デバッガ、seccomp-notifyがsyscallをインターセプトする仕組み |
| [vDSOと仮想システムコール](vdso.md) | リング遷移なしでカーネル支援による高速な時刻読み取り |
| [syscallの監査](audit.md) | auditサブシステム、audit_context、NETLINK_AUDIT、auditctl/ausearch/aureport |
| [syscallの再開メカニズム](restart-block.md) | ERESTARTSYS、ERESTART_RESTARTBLOCK、restart_block、restart_syscall() |
| [syscallの事件簿](war-stories.md) | syscallインターフェースにまつわる実際のバグ、ABI破壊、教訓 |

## クイックリファレンス {#quick-reference}

```bash
# 全syscallを番号付きで一覧表示する
ausyscall --dump | head -20

# プロセスのsyscallをトレースする
strace ls

# syscallの回数を数える
strace -c ls 2>&1

# syscallのオーバーヘッドを見る
perf stat -e 'syscalls:sys_enter_*' -- ls

# カーネル内のsyscallテーブルを見る
grep -r "SYSCALL_DEFINE" kernel/sys.c | head -10
```
