# カーネルコアの内部構造 {#kernel-core-internals}

> すべてのサブシステムをつなぎ合わせる接着剤: ブート、ロギング、パラメータ、パニック、モジュール初期化

## 「カーネルコア」とは何か {#what-is-kernel-core}

カーネルコアとは、特定のサブシステムには属さないが、すべてのサブシステムが依存しているインフラストラクチャの集まりである。主に`init/`、`kernel/`、`arch/x86/kernel/`（および他アーキテクチャの対応するディレクトリ）に存在し、次の問いに答えるものである: *圧縮されたバイナリが、どのようにしてタスクをスケジュールし、割り込みに応答し、ドライバをロードできる稼働中のオペレーティングシステムになるのか？*

ここで扱うトピックは次のとおり:

- Cのエントリポイント（`start_kernel()`）と、それが駆動するサブシステム初期化呼び出しの順序付けられたシーケンス
- カーネルがブートローダからブートパラメータを受け取り、パースする仕組み
- モジュールと組み込みサブシステムが、initcall機構を通じて初期化関数を登録する仕組み
- 何かが致命的に失敗したときに起こること — oops、パニック、クラッシュダンプの収集
- カーネル内のあらゆる診断メッセージを支える`printk`ロギング基盤

これはメモリ管理でもスケジューラでもネットワークでもない — それらすべてをセットアップしてから制御を引き渡す機構である。

---

## このセクションのページ {#pages-in-this-section}

| ページ | 扱う内容 |
|------|---------------|
| [printkとカーネルロギング](printk.md) | ログレベル、リングバッファ、レート制限、dynamic debug、dmesg |
| [初期ブートとstart_kernel()](early-boot.md) | Cエントリポイント、サブシステム初期化順序、initcall、`__init`セクション |
| [カーネルブートパラメータ](boot-params.md) | コマンドラインのパース、`__setup`、`early_param`、モジュールパラメータ |
| [カーネルパニックとOops](panic-oops.md) | Oops処理、`panic()`、スタックアンワインド、kdump、KASAN/KFENCE |
| [モジュール初期化とinitcall](initcalls-modules.md) | `module_init`/`module_exit`、struct module、ロードの流れ、署名 |
| [カーネルコアの事件簿](war-stories.md) | 初期化順序のバグ、`__init`の誤用、パニックループ |

---

## 推奨する読む順序 {#suggested-reading-order}

カーネル内部構造に初めて触れる場合は、次の順序で読むとよい:

1. **[初期ブートとstart_kernel()](early-boot.md)** — 何よりも先に初期化シーケンスを理解する
2. **[カーネルブートパラメータ](boot-params.md)** — コードが実行される前にカーネルがどのように設定されるか
3. **[モジュール初期化とinitcall](initcalls-modules.md)** — サブシステムとドライバがどのようにブートシーケンスにフックするか
4. **[printkとカーネルロギング](printk.md)** — 他のすべてが利用する可観測性の基盤
5. **[カーネルパニックとOops](panic-oops.md)** — 何かが失敗したときに起こること
6. **[カーネルコアの事件簿](war-stories.md)** — 概念を結び付ける実例

---

## カーネルのブートフロー {#kernel-boot-flow}

```
Power on / bootloader (GRUB/EFI)
        |
        | loads compressed kernel image (bzImage / Image)
        v
arch entry point (arch/x86/kernel/head_64.S)
        |
        | sets up early page tables, GDT, stack
        v
startup_64 / __startup_64
        |
        | decompresses kernel, relocates if KASLR
        v
x86_64_start_kernel()   (arch/x86/kernel/head64.c)
        |
        | early CR0/CR4/EFER setup, early IDT
        v
start_kernel()          (init/main.c)          <--- C entry point
        |
        |-- set_task_stack_end_magic(&init_task)
        |-- smp_setup_processor_id()
        |-- boot_cpu_init()
        |-- setup_arch()         (memory map, page tables, ACPI)
        |-- setup_per_cpu_areas()
        |-- sched_init()
        |-- rcu_init()
        |-- init_IRQ()
        |-- timekeeping_init()
        |-- kmem_cache_init()
        |-- console_init()       (printk now goes to screen)
        |
        v
rest_init()
        |
        |-- kernel_thread(kernel_init, ...)  --> PID 1
        |-- kernel_thread(kthreadd, ...)     --> PID 2
        |
        v
cpu_startup_entry(CPUHP_ONLINE)   <-- CPU 0 becomes idle thread
        |
        v (PID 1 runs)
kernel_init()
        |
        |-- do_initcalls()        (runs all initcall levels)
        |-- prepare_namespace()   (mount root filesystem)
        |
        v
exec /sbin/init  (or init= boot parameter)
        |
        v
userspace init system (systemd / sysvinit)
```

---

## クイックリファレンス: 主要なカーネルグローバル変数 {#quick-reference-key-kernel-globals}

| シンボル | ヘッダ | 説明 |
|--------|--------|-------------|
| `init_task` | `<linux/init_task.h>` | PID 0（アイドルスレッド）の静的なtask_struct。すべてのプロセスの系譜の根 |
| `jiffies` | `<linux/jiffies.h>` | 単調増加するティックカウンタ。32ビットでは約497日でラップアラウンドする |
| `HZ` | `<asm/param.h>` | タイマー割り込みの周波数（1秒あたりのティック数）。通常は100、250、1000のいずれか |
| `nr_cpu_ids` | `<linux/threads.h>` | possibleなCPUの数（初期ブート中に`setup_nr_cpu_ids()`によって設定される） |
| `NUMA_NO_NODE` | `<linux/numa.h>` | 「特定のNUMAノードなし」を意味する番兵値（-1）。アロケータAPIで使われる |
| `boot_command_line[]` | `<linux/init.h>` | ブートローダから受け取ったコマンドラインのコピー |
| `system_state` | `<linux/kernel.h>` | 現在のブートフェーズ: `SYSTEM_BOOTING`、`SYSTEM_RUNNING`、`SYSTEM_HALT`など |

---

## 参考文献 {#further-reading}

- [カーネルモジュール](../modules/README.md) — ツリー外モジュールの作成とロード
- [printk](printk.md) — ロギングの詳細
- [カーネルデバッグ](../debugging/README.md) — KGDB、kdump、oops解析
- `init/main.c` — `start_kernel()`と`rest_init()`の正典となるソース
- `Documentation/admin-guide/kernel-parameters.txt` — ブートパラメータの網羅的な一覧
