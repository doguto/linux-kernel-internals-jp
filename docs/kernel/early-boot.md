# 初期ブートとstart_kernel() {#early-boot-and-start_kernel}

> サブシステムの初期化順序、initcall、`__init`セクション

## 概要 {#overview}

ブートローダがカーネルを展開し、アーキテクチャ固有のアセンブリのエントリポイント（x86-64では`startup_64`、arm64では`primary_entry`）が必要最小限のもの — 初期ページテーブル、有効なスタック、GDT — をセットアップした後、制御は`init/main.c`の`start_kernel()`に移る。これがCのエントリポイントであり、カーネル内のすべてのサブシステムは、その初期化をたどっていくと、この関数から直接または間接に行われる呼び出しに行き着く。

`start_kernel()`は、長く順序付けられた関数呼び出しのシーケンスである。この順序は意図的なものである: あるサブシステムが初期化できるようになるには、他のサブシステムがすでに動作している必要がある場合がある。メモリ割り当ては、メモリを割り当てるあらゆるものより前に来なければならない。割り込みは、割り込みコントローラが設定されるまで無効にしておかなければならない。診断メッセージが役に立つためには、その前にコンソールがセットアップされていなければならない。

---

## start_kernel(): 初期化シーケンス {#start_kernel-the-initialization-sequence}

以下は、`init/main.c`（Linux 6.x）に現れる初期化の順序である。すべての呼び出しを挙げてはいない — システムを理解するうえで重要なものだけである。

```
start_kernel()
    set_task_stack_end_magic(&init_task)    [1]
    smp_setup_processor_id()               [2]
    debug_objects_early_init()
    init_vmlinux_build_id()
    cgroup_init_early()
    local_irq_disable()                    <- interrupts off
    boot_cpu_init()                        [3]
    page_address_init()                    [4]
    pr_notice(linux_banner)                [5]
    setup_arch(&command_line)              [6]
    mm_core_init_early()
    jump_label_init()
    static_call_init()
    early_security_init()                  [7]
    setup_boot_config()
    setup_command_line(command_line)       [8]
    setup_nr_cpu_ids()                     [9]
    setup_per_cpu_areas()                  [10]
    smp_prepare_boot_cpu()                 [11]
    parse_early_param()                    <- early_param() handlers run here
    ...
    trap_init()                            [12]
    mm_core_init()                         [13]
    poking_init()                          [14]
    ftrace_init()                          [15]
    early_trace_init()                     [16]
    sched_init()                           [17]
    radix_tree_init()                      [18]
    workqueue_init_early()                 [19]
    rcu_init()                             [20]
    init_IRQ()                             [21]
    tick_init()                            [22]
    timekeeping_init()                     [23]
    time_init()                            [24]
    kmem_cache_init_late()                 [25]
    ...
    console_init()                         [26]
    ...
    rest_init()                            [27]
```

`setup_arch()`は実際に`early_security_init()`より前に実行される — このページの以前の版から順序を入れ替えた。両者の間で、`start_kernel()`は`mm_core_init_early()`（後述のステップ[13]で説明する、初期メモリ初期化の分割部分）、`jump_label_init()`、`static_call_init()`も呼び出す。`setup_boot_config()`は`early_security_init()`と`setup_command_line()`の間で実行される。これらはいずれも、上の他の番号なしの呼び出しと同様に、独自のステップ番号を持たない。

`build_all_zonelists(NULL)`と（改名された）`page_alloc_init_cpuhp()`は、もはや`start_kernel()`から直接呼ばれることは一切ない — どちらも`kmem_cache_init()`とともに`mm_core_init()`（ステップ[13]）の内部に移された。

### 主要なステップの解説 {#key-steps-explained}

**[1] `set_task_stack_end_magic(&init_task)`**

`init_task`のカーネルスタックの底にカナリア値（`STACK_END_MAGIC = 0x57AC6E9F`）を置く。スタックがオーバーフローしてこの値を上書きすると、カーネルはそれを検出し、黙ってメモリを破壊する代わりにパニックできる。`init_task`は静的に割り当てられた`struct task_struct`であり、アイドルスレッド（PID 0）であり、すべてのプロセスの祖先である。

**[2] `smp_setup_processor_id()`**

ブートCPUのハードウェアIDを記録する。x86ではAPIC IDを、arm64ではMPIDRを読み取る。これは`smp_processor_id()`を使うあらゆるコードより前に行わなければならない。

**[3] `boot_cpu_init()`**（`kernel/cpu.c`）

CPUビットマスク（`cpu_present_mask`、`cpu_possible_mask`など）において、ブートCPUをpresent、possible、active、onlineとしてマークする。セカンダリCPUは後のSMP立ち上げ中にオンラインになるが、ブートCPUは最初に登録されていなければならない。

**[4] `page_address_init()`**

highmemページについて`struct page *`を仮想アドレスに変換するために使うページアドレスのハッシュテーブルを初期化する。ダイレクトマッピングを持つ64ビットカーネルではほとんど何もしないが、あらゆるページアドレスの検索より前に呼ばれなければならない。

**[5] `pr_notice(linux_banner)`**

人間が読める最初のログ行: `Linux version 6.x.y (compiler) #N SMP ...`。これが`dmesg`の先頭に見えるバージョン文字列である。この時点では`printk`はメモリ内のバッファに書き込むだけで、コンソールにはまだ何も表示されない。

**[6] `setup_arch(&command_line)`**（`arch/x86/kernel/setup.c`または`arch/arm64/kernel/setup.c`）

`start_kernel()`の中で最も大きく、最もアーキテクチャ固有の呼び出しである。x86-64では次のものが含まれる:

- ブートローダからのe820メモリマップのパース
- 初期カーネルページテーブルのセットアップ
- memblock（ブート時のメモリアロケータ）の初期化
- CPUとNUMAトポロジのためのACPIテーブル（MADT、SRAT、SLIT）のパース
- 初期コマンドラインポインタのセットアップ
- カーネルの仮想アドレスレイアウトの設定

`setup_arch()`から戻った後、カーネルは物理メモリがどれだけあり、どこにあるかを把握している。

**[7] `early_security_init()`**（`security/lsm_init.c`）

`setup_arch()`の前ではなく、後に実行される。「early」とフラグ付けされたLSMを走査し（`lsm_early_for_each_raw()`）、それぞれを`lsm_init_single()`で初期化する — これにより、完全性計測や同様の早期フックを行うLSMが、カーネルの大部分が立ち上がる前に登録できる。残りの（early以外の）LSMを初期化する`security_init()`は、ずっと後、`console_init()`よりかなり後に実行される。

**[8] `setup_command_line(command_line)`**

コマンドラインを2つの静的バッファにコピーする: `boot_command_line[]`（参照用で、決して変更されない）と`saved_command_line[]`（これも保持される）。`command_line`内の作業用コピーは、後続のパーサによって変更されることがある。

**[9] `setup_nr_cpu_ids()`**

CPUのpossibleビットマスクから`nr_cpu_ids`を計算する。これにより、カーネルの残りの部分は、今後目にすることになる最大のCPUインデックスを知り、per-CPU配列のサイズを正しく決められる。

**[10] `setup_per_cpu_areas()`**

per-CPUデータ用のメモリを割り当てる。リンカはper-CPU変数を`.data..percpu`セクションに配置する。実行時に`setup_per_cpu_areas()`は、このセクションのコピーをpossibleなCPUごとに1つずつ、偽共有を防ぐのに十分なパディングを挟んで作成する。この呼び出しの後、`this_cpu_*()`マクロが正しく動作する。

**[11] `smp_prepare_boot_cpu()`**

セカンダリCPUが起動する前に、ブートCPU上で必要なSMP固有の初期化を行う。x86では、per-CPUアクセスのためにブートCPUのGSセグメントをセットアップする。

**[12] `trap_init()`**（`arch/x86/kernel/traps.c`）

x86では、すべてのCPU例外 — 除算エラー（#DE）、ページフォールト（#PF）、一般保護例外（#GP）、ダブルフォルト（#DF）など — に対するハンドラを持つ割り込み記述子テーブル（IDT）をセットアップする。この呼び出しの後、CPUは例外を正しく処理できる。これより前は、あらゆるフォルトが致命的である。

**[13] `mm_core_init()`**（`mm/mm_init.c`）

コアのメモリ管理インフラを初期化する: `mem_init()`（memblockからバディアロケータへの移行）、`kmem_cache_init()`（slabアロケータのブートストラップ）、ページテーブルキャッシュなど。`build_all_zonelists(NULL)`（`mm/page_alloc.c` — `mm/numa.c`のNUMA距離順に、NUMAゾーンのフォールバックリストを構築する）と`page_alloc_init_cpuhp()`（各セカンダリCPUのper-CPUページフレーム/pcplistをセットアップするCPUホットプラグのコールバックを登録する — 以前の`page_alloc_init()`から改名された）が呼ばれるのもここであり、どちらももはや独立した`start_kernel()`のステップではない。`mm_core_init()`の後、汎用のページアロケータ（`alloc_pages()`）と`kmalloc()`の両方が動作する。注意: `kmem_cache_init_late()`（後述のステップ[25]）は、`start_kernel()`から行われる別個の、より後の、単純な直接呼び出しである — `mm_core_init()`の内部から呼ばれるのではなく、`core_initcall()`やinitcallレベルの機構を通じて登録されるものでもない。

**[14] `poking_init()`**

kprobe、ftrace、ライブパッチが実行時にカーネルテキストを安全に書き換えるために使う、テキストポーキング基盤を初期化する。x86では、読み取り専用のカーネルページに書き込むための一時的なマッピング機構をセットアップする。

**[15] `ftrace_init()`**

関数トレーサの内部データ構造（`ftrace_ops`リスト、`__mcount_loc`に記録されたmcount呼び出し箇所）を準備する。トレーサはまだ有効ではないが、基盤は準備が整う。

**[16] `early_trace_init()`**

ほとんどのサブシステムが立ち上がる前にトレーシングのリングバッファとトレースクロックを初期化し、初期ブート中のトレースイベントを取得できるようにする。

**[17] `sched_init()`**（`kernel/sched/core.c`）

スケジューラを初期化する: per-CPUのランキュー（`struct rq`）を割り当て、CFS、RT、deadlineのスケジューリングクラスをセットアップし、`init_task`を実行可能にする。`sched_init()`の後、コンテキストスイッチが可能になる。

**[18] `radix_tree_init()`**

radix treeノード用のkmem_cacheを初期化する。radix tree（VMAについては現在maple treeになっているが、radix treeは他の場所でまだ使われている）は、ページキャッシュ、inodeキャッシュ、その他多くのサブシステムで使われる。

**[19] `workqueue_init_early()`**（`kernel/workqueue.c`）

初期のworkqueue基盤を作成する。workqueueの完全な初期化（`workqueue_init()`）は後でcore_initcall経由で行われる。この早期の呼び出しはデータ構造をセットアップし、ワーカースレッドが存在する前でも`schedule_work()`でキューに入れられるようにする。

**[20] `rcu_init()`**（`kernel/rcu/tree.c`）

RCU（Read-Copy-Update）を初期化する。RCUツリー（rcu_node階層）、per-CPUの状態、rcu_tasks機構をセットアップする。この後、`rcu_read_lock()` / `rcu_read_unlock()`と`call_rcu()`を安全に使えるようになる。

**[21] `init_IRQ()`**（`arch/x86/kernel/irqinit.c`）

割り込みコントローラを初期化する。x86では、レガシーの8259 PIC（存在すればAPIC）をセットアップし、IRQ記述子を割り当て、割り込み処理フレームワークを準備する。この後、`request_irq()`が可能になる。

**[22] `tick_init()`**（`kernel/time/tick-common.c`）

ティックフレームワーク（NO_HZ、HRTICK、ブロードキャストクロックイベント）を初期化する。時刻管理より前に行わなければならない。

**[23] `timekeeping_init()`**（`kernel/time/timekeeping.c`）

ハードウェアのクロックソースを読み取り、`timekeeper`を初期化する。この後、`ktime_get()`と`ktime_get_real_ts64()`が動作する。これはカーネルの内部クロックであり、壁時計ではない。

**[24] `time_init()`**（`arch/x86/kernel/time.c`）

アーキテクチャ固有の時間の初期化。x86では、PITまたはHPETを基準にTSCを較正し、タイマー割り込み用のクロックイベントデバイスをセットアップする。

**[25] `kmem_cache_init_late()`**（`mm/slub.c` — SLABは削除されており、残っているアロケータはSLUBだけである）

これはslabのブートストラップではない — それは`kmem_cache_init()`であり、すでに`mm_core_init()`（ステップ[13]）の内部で済んでおり、`kmalloc()`はその時点から動作している。`kmem_cache_init_late()`はずっと後、`console_init()`の直前に実行される: SLUBがper-CPUのpartial slabキャッシュをフラッシュするために使う`slub_flushwq` workqueueを割り当て、（`CONFIG_SLAB_FREELIST_RANDOM`が有効な場合）`prandom_init_once()`によってブートごとのフリーリストランダム化の状態をシードする。

**[26] `console_init()`**（`kernel/printk/printk.c`）

コンソールサブシステムを初期化し、登録されたコンソールドライバを呼び出す。この呼び出しの後、`printk`の出力はシリアルコンソールやVGAに出る — ログリングにバッファリングされていたブートメッセージが実際に画面に現れるのはこのときである。

**[27] `rest_init()`**（`init/main.c`）

`start_kernel()`の最後の呼び出し。カーネルスレッドを作成し、アイドルループに入る（後述）。

---

## rest_init(): PID 1とPID 2の生成 {#rest_init-spawning-pid-1-and-pid-2}

```c
/* init/main.c */
static noinline void __ref __noreturn rest_init(void)
{
    struct kernel_clone_args init_args = {
        .flags      = (CLONE_VM | CLONE_UNTRACED),
        .fn         = kernel_init,
        .fn_arg     = NULL,
    };
    struct task_struct *tsk;
    int pid;

    rcu_scheduler_starting();
    /*
     * We need to spawn init first so that it obtains pid 1, however
     * the init task will end up wanting to create kthreads, which, if
     * we schedule it before we create kthreadd, will OOPS.
     */
    pid = kernel_clone(&init_args);
    /*
     * Pin init on the boot CPU. Task migration is not properly working
     * until sched_init_smp() has been run. It will set the allowed
     * CPUs for init to the non isolated CPUs.
     */
    rcu_read_lock();
    tsk = find_task_by_pid_ns(pid, &init_pid_ns);
    tsk->flags |= PF_NO_SETAFFINITY;
    set_cpus_allowed_ptr(tsk, cpumask_of(smp_processor_id()));
    rcu_read_unlock();

    numa_default_policy();
    pid = kernel_thread(kthreadd, NULL, NULL, CLONE_FS | CLONE_FILES);
    rcu_read_lock();
    kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
    rcu_read_unlock();

    /*
     * Enable might_sleep() and smp_processor_id() checks.
     * They cannot be enabled earlier because with CONFIG_PREEMPTION=y
     * kernel_thread() would trigger might_sleep() splats. With
     * CONFIG_PREEMPT_VOLUNTARY=y the init task might have scheduled
     * already, but it's stuck on the kthreadd_done completion.
     */
    system_state = SYSTEM_SCHEDULING;

    complete(&kthreadd_done);

    /*
     * The boot idle thread must execute schedule()
     * at least once to get things moving:
     */
    schedule_preempt_disabled();
    /* Call into cpu_idle with preempt disabled */
    cpu_startup_entry(CPUHP_ONLINE);
}
```

- **PID 1（`kernel_init`）**: `CLONE_VM | CLONE_UNTRACED`を要求する`struct kernel_clone_args`を指定して`kernel_clone()`で生成される（汎用の`kernel_thread()`ではない — PID 1には本物のユーザー空間への復帰パスが必要である。このコードの古い版では同じ目的のために専用の`user_mode_thread()`ヘルパを呼んでいたが、現在の`rest_init()`は`kernel_clone_args`を直接組み立てる）。`do_initcalls()`を実行して登録されたすべてのinitcallを実行し、その後ルートファイルシステムをマウントして`/sbin/init`（またはコマンドラインの`init=`で指定されたもの）をexecする。
- **PID 2（`kthreadd`）**: カーネルスレッドデーモンで、`kernel_thread()`で生成される。以降のすべてのカーネルスレッドは、`kernel_thread()`呼び出しに応じて`kthreadd`によって作成される。
- **CPU 0のアイドル**: `rest_init()`が終わると、CPU 0は`cpu_startup_entry()`に入ってアイドルスレッドになり、`do_idle()`をループで実行する。`rest_init()`自体が`start_kernel()`に戻ることはない。

---

## initcallレベル {#initcall-levels}

組み込みのカーネルサブシステムは、initcallマクロを使って初期化関数を登録する。これらの関数は、`start_kernel()`が完了した後、`kernel_init()`内の`do_initcalls()`によって呼び出される。レベルが順序を定める:

| マクロ | レベル | 典型的な用途 |
|-------|-------|-------------|
| `early_initcall(fn)` | early | 非常に早い段階。pureより前 |
| `pure_initcall(fn)` | 0 | 純粋なインフラ。ハードウェアには触れない |
| `core_initcall(fn)` | 1 | コアカーネルのインフラ（workqueueなど） |
| `core_initcall_sync(fn)` | 1s | coreの後の同期ポイント |
| `postcore_initcall(fn)` | 2 | コアサブシステムの後 |
| `arch_initcall(fn)` | 3 | アーキテクチャ固有の初期化 |
| `subsys_initcall(fn)` | 4 | サブシステムの初期化（PCI、USBコア、ネットワーク） |
| `subsys_initcall_sync(fn)` | 4s | |
| `fs_initcall(fn)` | 5 | ファイルシステムの登録 |
| `rootfs_initcall(fn)` | rootfs | ルートファイルシステム（initramfsの展開） |
| `device_initcall(fn)` | 6 | ほとんどのドライバ。`module_init()`はこのエイリアス |
| `device_initcall_sync(fn)` | 6s | |
| `late_initcall(fn)` | 7 | すべてのデバイスが初期化された後 |

### 仕組み {#how-it-works}

各マクロ（たとえば`core_initcall(fn)`）は、次の連鎖を通じて展開される: `__define_initcall(fn, id)` → `___define_initcall(fn, id, .initcall##id)` → `__unique_initcall(fn, id, __sec, __iid)` → `____define_initcall(...)`。最後のものが、実際にセクションエントリを出力する層である:

```c
/* include/linux/init.h */
#ifdef CONFIG_HAVE_ARCH_PREL32_RELOCATIONS
#define ____define_initcall(fn, __stub, __name, __sec)		\
	__define_initcall_stub(__stub, fn)			\
	asm(".section	\"" __sec "\", \"a\"		\n"	\
	    __stringify(__name) ":			\n"	\
	    ".long	" __stringify(__stub) " - .	\n"	\
	    ".previous					\n");	\
	static_assert(__same_type(initcall_t, &fn));
#else
#define ____define_initcall(fn, __unused, __name, __sec)	\
	static initcall_t __name __used 			\
		__attribute__((__section__(__sec))) = fn;
#endif

#define __unique_initcall(fn, id, __sec, __iid)			\
	____define_initcall(fn,					\
		__initcall_stub(fn, __iid, id),			\
		__initcall_name(initcall, __iid, id),		\
		__initcall_section(__sec, __iid))

#define ___define_initcall(fn, id, __sec)			\
	__unique_initcall(fn, id, __sec, __initcall_id(fn))

#define __define_initcall(fn, id) ___define_initcall(fn, id, .initcall##id)
```

現在、ほとんどの64ビットアーキテクチャ（x86-64とarm64を含む）は`CONFIG_HAVE_ARCH_PREL32_RELOCATIONS=y`でビルドされる。このパスでは、各initcallエントリは単純な関数ポインタではない — `__define_initcall_stub()`が`fn()`を呼ぶだけの小さなラッパー関数（`__stub`）を出力し、`asm()`ブロックがそのスタブへの**PC相対の32ビットオフセット**を格納する。これは読み出されるときにだけ実アドレスに解決される（下で使われる`initcall_from_entry()`）。64ビットでは4バイトの相対オフセットは完全なポインタの半分のサイズなので、わずかなデコードのオーバーヘッドと引き換えに`initcall*.init`セクションを小さくできる。`CONFIG_HAVE_ARCH_PREL32_RELOCATIONS`がない場合、`____define_initcall()`は、このセクションでかつて唯一示していた単純な静的関数ポインタ配列にフォールバックする。

リンカスクリプト（`arch/x86/kernel/vmlinux.lds.S`。`include/asm-generic/vmlinux.lds.h`経由）は、すべての`.initcall*.init`セクションをレベル順にまとめる。`do_initcalls()`はこの配列を走査し、各関数ポインタを呼び出す:

```c
/* init/main.c */
static void __init do_initcall_level(int level, char *command_line)
{
    initcall_entry_t *fn;

    parse_args(initcall_level_names[level],
               command_line, __start___param,
               __stop___param - __start___param,
               level, level,
               NULL, ignore_unknown_bootoption);

    do_trace_initcall_level(initcall_level_names[level]);
    for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)
        do_one_initcall(initcall_from_entry(fn));
}

static void __init do_initcalls(void)
{
    int level;
    size_t len = saved_command_line_len + 1;
    char *command_line;

    command_line = kzalloc(len, GFP_KERNEL);
    if (!command_line)
        panic("%s: Failed to allocate %zu bytes\n", __func__, len);

    for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++) {
        /* パーサが command_line を変更するので、毎回元に戻す */
        strcpy(command_line, saved_command_line);
        do_initcall_level(level, command_line);
    }

    kfree(command_line);
}
```

レベルごとのディスパッチという明白な点以外に、注目すべき点が2つある: `do_initcall_level()`は各レベル固有のモジュールパラメータの区画に対して`parse_args()`を再実行する（そのため`module.param=value`形式のブート引数は、`parse_early_param()`の時点で一度だけグローバルに照合されるのではなく、正しいレベルに対して照合される）。そして`do_initcalls()`は、各レベルの前に`saved_command_line`を作業用バッファにコピーし直す。`parse_args()`は与えられた文字列を走査しながら書き換えるからである — この復元がなければ、完全で未変更のコマンドラインを見られるのは最初のレベルだけになってしまう。

initcallの戻り値は重要である: 非ゼロの戻り値は`pr_warn`メッセージを出すが、ブート処理を止めることはない（`initcall_debug`がより厳格なモードに設定されている場合を除く）。エラーを返したinitcallは、単にログに記録されてスキップされる。

---

## `__init`と`__exit`セクション {#__init-and-__exit-sections}

### `__init` {#__init}

`__init`アノテーションを付けた関数は、`.init.text` ELFセクションに配置される。`do_initcalls()`が完了してinitプロセスが始まると、カーネルは`.init.text`と`.init.data`セクション全体を解放する:

```c
/* init/main.c */
static int __ref kernel_init(void *unused)
{
    ...
    free_initmem();
    ...
}
```

`kernel_init()`には`__init`ではなく`__ref`が付けられている — それ自体が`__init`であることはできない。なぜなら、`free_initmem()`を呼んで`.init.text`/`.init.data`セクションを解放するのがこの関数だからである。`.init.text`に置かれた関数が、そのセクションが解放されるときにまだ実行中であれば、自分自身が載っているページを足元から回収させることになる。`__ref`はmodpostに対して「この関数は意図的に`__init`のコード/データを参照しているので、警告しないでほしい」と伝える。

`free_initmem()`は`free_reserved_area()`を呼び出して、これらのページをバディアロケータに返却する。ブートログに次のようなメッセージが含まれるのはこのためである:

```
Freeing unused kernel image (initmem) memory: 2548K
```

実際上の帰結: **ブート後に`__init`関数を決して呼んではならない**。それを含むページは解放されており、再利用されている可能性がある。`__init`関数への関数ポインタを保存することは時限爆弾である。

```c
/* 誤り: __init 関数へのポインタを保存している */
static int (*saved_fn)(void);

static int __init my_init(void)
{
    saved_fn = some_init_helper;  /* some_init_helper は __init */
    return 0;
}

/* ブート後に saved_fn を呼ぶ → use-after-free、oops */
```

### `__exit` {#__exit}

`__exit`アノテーションを付けた関数は`.exit.text`に配置される。組み込みコード（ロード可能モジュールではないもの）の場合、`.exit.text`セクションはリンク時に破棄される — 組み込みコードは決してアンロードできないので、リンカはそれを最終イメージに含めない。ロード可能モジュールの場合、`.exit.text`は保持され、モジュールが削除されるときに呼び出される。

### データセクション {#data-sections}

| アノテーション | セクション | ブート後に解放されるか |
|-----------|---------|------------------|
| `__init` | `.init.text` | はい |
| `__initdata` | `.init.data` | はい |
| `__initconst` | `.init.rodata` | はい |
| `__exit` | `.exit.text` | 破棄（組み込み） / 保持（モジュール） |
| `__exitdata` | `.exit.data` | 同上 |

### サイズの節約 {#size-savings}

`__init`機構は、無視できない量のメモリを取り戻す。典型的なx86-64カーネルでは、initセクションは合計1〜3MBになる。メモリの限られた組み込みシステムでは、これは非常に重要である。

---

## ブート順序を観測する {#observing-boot-order}

### タイムスタンプ付きのdmesg {#dmesg-with-timestamps}

```bash
dmesg -T          # 人間が読めるタイムスタンプ
dmesg --color     # レベルごとに色付けした出力
dmesg | head -50  # 最初の 50 行（アーキテクチャ初期化、バージョン、メモリマップ）
```

### initcall_debugブートパラメータ {#initcall_debug-boot-parameter}

カーネルコマンドラインに`initcall_debug`を追加する（GRUBでは`linux ... initcall_debug`）。カーネルはすべてのinitcallをその実行時間とともに出力する:

```
calling  e1000e_init_module+0x0/0x3c [e1000e] @ 2
initcall e1000e_init_module+0x0/0x3c [e1000e] returned 0 after 1243 usecs
calling  ahci_init+0x0/0x28 @ 2
initcall ahci_init+0x0/0x28 returned 0 after 87 usecs
```

`@`の後の数字は、そのinitcallが実行されたCPUである。

### /proc/kallsyms {#prockallsyms}

```bash
# 登録された initcall シンボルをセクション順に見る:
grep '__initcall_' /proc/kallsyms | head -20
```

これらのシンボルは、`.initcall*.init`セクションに配置された関数ポインタのエントリに対応する。

### ftraceによるトレーシング {#tracing-with-ftrace}

```bash
# ブート中に行われたすべての呼び出しをトレースする（早期の ftrace セットアップが必要）
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | grep 'do_one_initcall'
```

---

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [init/main.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/init/main.c) — `start_kernel()`、`rest_init()`、`kernel_init()`、`do_initcalls()`
- [include/linux/init.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/init.h) — `__init`/`__exit`セクションのマクロと、`early_initcall()` … `late_initcall()`のレベルマクロ
- [arch/x86/kernel/head_64.S](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/head_64.S) — `startup_64`。`start_kernel()`より前に実行されるx86-64のアセンブリのエントリポイント
- [arch/arm64/kernel/head.S](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/kernel/head.S) — `primary_entry`。arm64での相当物
- [arch/x86/kernel/setup.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/setup.c) — x86-64の`setup_arch()`: e820のパース、memblockとページテーブルのセットアップ、ACPIテーブルのパース
- [kernel/cpu.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/cpu.c) — `boot_cpu_init()`
- [mm/mm_init.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mm_init.c) — `mm_core_init()`
- [mm/page_alloc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) — `build_all_zonelists()`
- [mm/slub.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/slub.c) — SLUBのブートストラップである`kmem_cache_init()`と、後の`slub_flushwq`/フリーリストランダム化のセットアップを行う`kmem_cache_init_late()`
- [security/lsm_init.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/security/lsm_init.c) — `early_security_init()`: 「early」とフラグ付けされたLSMを初期化する
- [kernel/sched/core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/core.c) — `sched_init()`
- [kernel/rcu/tree.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/rcu/tree.c) — `rcu_init()`
- [kernel/workqueue.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/workqueue.c) — `workqueue_init_early()`
- [kernel/printk/printk.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/printk/printk.c) — `console_init()`
- [include/asm-generic/vmlinux.lds.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/asm-generic/vmlinux.lds.h) — `INIT_CALLS`/`INIT_CALLS_LEVEL`。`.initcall*.init`セクションをレベル順に配置するリンカマクロ

### 関連ページ {#related-pages}

- [カーネルブートパラメータ](boot-params.md) — `early_param`と`__setup`がどのようにパースされるか
- [モジュール初期化とinitcall](initcalls-modules.md) — `module_init()`のモジュール側
- [printkとカーネルロギング](printk.md) — printkが`console_init()`の後では動作し、前では動作しない理由
- [memblock: ブート時のメモリアロケータ](../mm/memblock.md) — `setup_arch()`が初期化し、バディアロケータが存在する前に使われるアロケータ
- [SLUBアロケータの内部構造](../mm/slab-internals.md) — `struct kmem_cache`、フリーリストの仕組み、per-CPUキャッシュ: `kmem_cache_init()`がブートストラップする実行時の機構
- [ランキューとタスク選択](../sched/runqueues.md) — `sched_init()`が割り当てるper-CPUの`struct rq`
- [RCU（Read-Copy-Update）](../locking/rcu.md) — `rcu_init()`の後に安全に使えるようになるもの

### 外部資料 {#external}

- [The Linux/x86 Boot Protocol](https://docs.kernel.org/arch/x86/boot.html) — `startup_64`に先立つ、ブートローダ/カーネル間の引き渡し
- [Kernel Parameters: `initcall_debug`](https://docs.kernel.org/admin-guide/kernel-parameters.html) — 「ブート順序を観測する」で扱ったブートパラメータの公式説明
