# カーネルパニックとOops {#kernel-panic-and-oops}

> Oopsの処理、`panic()`、コールスタックのデコード、kdump

## 何がoopsを引き起こすか {#what-triggers-an-oops}

「oops」とは、現在のコンテキストでは安全に回復できないフォルトについてのカーネルの報告である。x86-64でよくある原因:

- **NULLポインタ参照** — アドレス`0x0`から`0xfff`へのアクセス（最初のページ4KBはマップされていない）
- **use-after-free** — アロケータに返却され、再割り当てされている可能性のあるメモリへのアクセス
- **スタック破壊** — スタックスマッシュによる戻りアドレスの上書き。スタックカナリアやORCアンワインダによって検出される
- **無効オペコード（#UD）** — `ud2`（`BUG()`で使われる）や本当に無効な命令の実行
- **一般保護例外（#GP）** — 特権リソースへの不正なアクセス、セグメントリミット違反
- **ダブルフォルト（#DF）** — フォルト処理中のフォルト。多くの場合スタックオーバーフロー

CPUが例外を発生させ、IDTハンドラが実行され、制御は`do_page_fault()`や`do_general_protection()`などに到達する。フォルトが（ユーザープログラムではなく）カーネルコードで発生した場合、ハンドラは`die()`を呼び出す。

---

## `die()`とoopsのパス {#die-and-the-oops-path}

```c
/* arch/x86/kernel/dumpstack.c */

void die(const char *str, struct pt_regs *regs, long err)
{
    unsigned long flags = oops_begin();
    int sig = SIGSEGV;

    if (__die(str, regs, err))   /* oops メッセージを整形して出力する */
        sig = 0;

    oops_end(flags, regs, sig);  /* タスクを kill するかパニックするかを決める */
}
```

### `oops_begin()` / `oops_end()` {#oopsbegin-oopsend}

`oops_begin()`はraw spinlock（`die_lock`）を取得して、CPU間で同時に発生するoopsを直列化する — *同じ*CPU上でのネストしたoops（`die_owner`/`die_nest_count`で追跡される）はデッドロックさせずに通過させる — そして`console_verbose()`と`bust_spinlocks(1)`を呼び出して、スピンしているコンソールの書き手をフラッシュする。保存した割り込みフラグを返し、`oops_end()`がそれを復元できるようにする。どちらの関数もウォッチドッグには一切触れず、`oops_count`というsysctlも存在しない。

`kexec_should_crash()`/`crash_kexec()`が実際に呼び出されるのは（`die()`自体の中ではなく）`oops_end()`であり、その後`die_lock`を解放し、カーネルをtaint状態にし（`TAINT_DIE`）、そして — `signr`が非ゼロなら — `panic()`するか（割り込みコンテキストでは常に、あるいは`panic_on_oops`が設定されている場合）、そうでなければ問題を起こしたタスクをkillするかを決める。

---

## oopsメッセージの解剖 {#anatomy-of-an-oops-message}

```
BUG: unable to handle kernel NULL pointer dereference at 0000000000000008
PGD 0 P4D 0
Oops: 0002 [#1] SMP NOPTI
CPU: 3 PID: 1234 Comm: mydriver Tainted: G           O      5.15.0 #1
Hardware name: QEMU Standard PC (i440FX + PIIX, 1996)
RIP: 0010:my_driver_read+0x2c/0x80 [mydriver]
Code: 48 89 e5 41 54 53 48 89 fb 48 8b 47 08 48 85 c0 74 19 ...
RSP: 0018:ffffb3c4c0a1bdc8 EFLAGS: 00010246
RAX: 0000000000000000 RBX: ffff9d4b81234500 RCX: 0000000000000000
RDX: 0000000000000000 RSI: 0000000000000001 RDI: ffff9d4b81234500
RBP: ffffb3c4c0a1bdd0 R08: 0000000000000000 R09: 0000000000000000
R10: 0000000000000000 R11: 0000000000000000 R12: ffff9d4b81234500
R13: 0000000000000000 R14: 0000000000000000 R15: 0000000000000000
FS:  00007f8a3c4a4740(0000) GS:ffff9d4ba7c00000(0000) knlGS:0000000000000000
CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
CR2: 0000000000000008 CR3: 000000004a210001 CR4: 00000000003706e0
Call Trace:
 <TASK>
 vfs_read+0xb3/0x190
 ksys_read+0x5a/0xd0
 __x64_sys_read+0x1a/0x20
 do_syscall_64+0x5b/0xc0
 entry_SYSCALL_64_after_hwframe+0x61/0xcb
 </TASK>
```

主要なフィールド:

| フィールド | 意味 |
|-------|---------|
| `BUG: unable to handle kernel NULL pointer dereference at 0000000000000008` | フォルトの種類とフォルトしたアドレス |
| `Oops: 0002 [#1]` | エラーコード（0002 = 書き込みフォルト）。`[#1]` = このブートで最初のoops |
| `Tainted: G ... O` | taintフラグ（後述） |
| `RIP: 0010:my_driver_read+0x2c/0x80` | 命令ポインタ: 関数 + オフセット / 関数サイズ |
| `CR2` | ページフォールトを引き起こしたアドレス（フォルトした仮想アドレス） |
| `Call Trace` | 現在のフレームからシステムコールのエントリまでのスタックアンワインド |

---

## taintフラグ {#taint-flags}

`Tainted:`行は、カーネルの完全性の状態を要約する。「tainted」なカーネルとは、バグ報告の信頼性を下げるかもしれない形で改変されたカーネルである。

| フラグ | 意味 |
|------|---------|
| `G` | ロードされたすべてのモジュールがGPLライセンスである（これによるtaintはない） |
| `P` | プロプライエタリ（非GPL）モジュールがロードされた |
| `O` | ツリー外（メインラインにない）モジュールがロードされた |
| `W` | WARN()が発生した |
| `D` | `die()`が少なくとも1回呼ばれた |
| `E` | 署名のないモジュールがロードされた |
| `C` | stagingドライバがロードされている |
| `X` | 補助的なtaint（アーキテクチャ固有） |

```bash
# ユーザー空間から taint 状態を確認する:
cat /proc/sys/kernel/tainted
# 0 = taint なし
# 各ビットがフラグに対応する。Documentation/admin-guide/tainted-kernels.rst を参照
```

---

## スタックアンワインド: ORC {#stack-unwinding-orc}

Linux 4.14以降、x86-64はフレームポインタベースのアンワインドの代わりに**ORC（Oops Rewind Capability）アンワインダ**を使う。ORCは、コンパイラがフレームポインタを省略する場合（最適化を有効にすると一般的）でも動作するため、より信頼性が高い。

ビルド時に`objtool`が各関数を解析し、すべての命令位置におけるスタックレイアウトを記述するORCエントリを生成する:

```c
/* arch/x86/include/asm/orc_types.h */
struct orc_entry {
    s16     sp_offset;   /* CFA からのスタックポインタのオフセット */
    s16     bp_offset;   /* ベースポインタのオフセット */
    unsigned int sp_reg:4;   /* CFA を保持するレジスタ */
    unsigned int bp_reg:4;
    unsigned int type:3;
    unsigned int signal:1;
};
```

これらのエントリは`.orc_unwind`と`.orc_unwind_ip` ELFセクションに格納される。アンワインダ（`arch/x86/kernel/unwind_orc.c`）は現在のRIPに対応するORCエントリを検索し、それを使って1フレームずつアンワインドして、oopsメッセージ内の`Call Trace`を生成する。

arm64では、カーネルはフレームポインタベースのアンワインダを使う。arm64は`CONFIG_FRAME_POINTER`を必須としており、フレームポインタのチェーンが常に保たれることが保証されるので、アンワインダはアーキテクチャ固有のアンワインドテーブルなしにフレームをたどれる。

---

## `WARN()`、`BUG()`とその変種 {#warn-bug-and-their-variants}

```c
/* include/linux/bug.h, include/asm-generic/bug.h */

/* スタックトレースを出力して続行する。タスクを kill しない。 */
WARN(condition, fmt, ...)
WARN_ON(condition)          /* フォーマット文字列なし */
WARN_ONCE(condition, fmt, ...)  /* 最大 1 回だけ発火する */
WARN_ON_ONCE(condition)

/* oops を引き起こす（x86 では無効オペコード。タスクを kill するかパニックする）。 */
BUG()
BUG_ON(condition)
```

x86の`BUG()`は`ud2`命令（未定義命令オペコード`0x0f 0x0b`）に展開される。これは`#UD`例外を発生させ、カーネルの`invalid_op`ハンドラに到達し、そこから`die()`が呼ばれる。

x86（および`__WARN_FLAGS`を定義する他のアーキテクチャ）では、`WARN()`は直接の関数呼び出しではなく`ud2`トラップ命令に展開される — 結果として生じる無効オペコード例外は`exc_invalid_op()`で捕捉され、それが`ud2`を警告箇所として認識して`report_bug()` → `__warn()`を呼び出す。（`warn_slowpath_fmt()`はトラップベースの警告を持たないアーキテクチャ向けのフォールバックパスであり、x86ではデッドコードである。）いずれの場合も、`__warn()`は`dump_stack()`と`print_oops_end_marker()`を呼び出すが`die()`は引き起こさない — 実行は`WARN()`の後も続く。

---

## `panic()`: 引き返せない地点 {#panic-the-point-of-no-return}

`panic()`自体は薄い可変長引数のラッパーにすぎない。実際のロジックはすべて`vpanic()`にあり、`panic()`は`va_list`を渡してそれを呼び出す:

```c
/* kernel/panic.c（簡略化 — 実際の vpanic() はマルチ CPU での
 * パニックのリダイレクト、kgdb、sysinfo のダンプ、コンソールのフラッシュも扱う） */
void panic(const char *fmt, ...)
{
    va_list args;

    va_start(args, fmt);
    vpanic(fmt, args);
    va_end(args);
}

void vpanic(const char *fmt, va_list args)
{
    /* 1. ローカル IRQ とプリエンプションを無効化する */
    local_irq_disable();
    preempt_disable_notrace();

    /* 2. パニックメッセージを整形して出力する */
    console_verbose();
    bust_spinlocks(1);
    static char buf[PANIC_MSG_BUFSZ];
    vscnprintf(buf, sizeof(buf), fmt, args);
    pr_emerg("Kernel panic - not syncing: %s\n", buf);

    /* 3. notifier チェーンが実行される*前に*クラッシュカーネルへの kexec を
     *    試みる。ただし crash_kexec_post_notifiers ブートパラメータで
     *    ステップ 4 の後まで遅らせるよう指定されている場合を除く */
    if (!crash_kexec_post_notifiers)
        __crash_kexec(NULL);  /* 公開版 crash_kexec() のパニック CPU チェックを迂回する */

    /* 4. 登録されたパニック notifier を起動する */
    atomic_notifier_call_chain(&panic_notifier_list, 0, buf);

    /* 5. kdump が制御を引き継がなかった場合、再起動を試みる — パニック後は
     *    通常のタイマーが動かないので mdelay() を使い、待機中に NMI ウォッチドッグが
     *    発火しないよう繰り返しタッチする */
    if (panic_timeout > 0) {
        pr_emerg("Rebooting in %d seconds..\n", panic_timeout);
        for (long i = 0; i < panic_timeout * 1000; i += PANIC_TIMER_STEP) {
            touch_nmi_watchdog();
            mdelay(PANIC_TIMER_STEP);
        }
    }
    if (panic_timeout != 0)
        emergency_restart();

    /* 6. 停止: 代わりに softlockup ウォッチドッグをタッチしながら無限ループする
     *   （そうしないと NMI ウォッチドッグが発火してマシンを再起動してしまう） */
    for (long i = 0; ; i += PANIC_TIMER_STEP) {
        touch_softlockup_watchdog();
        mdelay(PANIC_TIMER_STEP);
    }
}
```

`panic_notifier_list`により、サブシステムはシステムが停止する前に実行されるコールバックを登録できる。netconsole、pstore、kdumpはいずれもこのチェーンにフックしている。

### `panic_on_oops` sysctl {#paniconoops-sysctl}

```bash
# 現在の値を読む:
cat /proc/sys/kernel/panic_on_oops
# 0 = 回復を試みる（タスクを kill する）
# 1 = oops でパニックする

# 本番環境向けに設定する（多くのサイトがこうしている）:
echo 1 > /proc/sys/kernel/panic_on_oops
```

`panic_on_oops = 1`の場合、`oops_end()`は続行を試みる代わりに`panic()`を呼び出す。これは、部分的に破損したカーネル状態がクリーンな再起動よりも悪い本番環境に適している。

### `panic=N`ブートパラメータ {#panicn-boot-parameter}

```
panic=30    # reboot 30 seconds after panic
panic=-1    # reboot immediately
panic=0     # do not reboot (halt forever, default)
```

---

## kdump: クラッシュ時のメモリの取得 {#kdump-capturing-crash-memory}

kdumpにより、カーネルはパニック時に完全なメモリダンプを保存できる。この機構は`kexec(2)`を使い、予約されたメモリ領域から事前にロードされた「クラッシュカーネル」を、通常のブートを経ずに起動する。

### セットアップ {#setup}

```bash
# ブート時にクラッシュカーネル用のメモリを予約する:
# （/etc/default/grub の GRUB_CMDLINE_LINUX に追加する）
crashkernel=256M

# クラッシュカーネルをロードする（initscript や systemd ユニットでブート時に実行する）:
kexec -p /boot/vmlinuz-$(uname -r) \
    --initrd=/boot/initrd.img-$(uname -r) \
    --append="root=/dev/sda1 irqpoll maxcpus=1 reset_devices"
```

### パニック時に何が起こるか {#what-happens-on-panic}

1. kdumpは、`panic()`から（内部の`__crash_kexec()`経由で）、または`die()`の`oops_end()`から（公開版の`crash_kexec()`経由で）起動される
2. kexecがクラッシュカーネルのページテーブルに切り替え、そのエントリポイントへジャンプする
3. クラッシュカーネルが新しいカーネルとしてブートする（予約されたメモリ領域のみを使う）
4. クラッシュカーネルは、元のカーネルのメモリを`/proc/vmcore`（ELFコアダンプ）として公開する
5. ユーザー空間のツール（典型的には`makedumpfile`）が`/proc/vmcore`をディスクにコピーする

### ダンプの解析 {#analyzing-the-dump}

```bash
# クラッシュしたシステム上で（または同じカーネルを持つシステム上で）:
crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux vmcore

# crash の中で:
crash> bt         # クラッシュしたタスクのバックトレース
crash> log        # カーネルログのリングバッファ
crash> ps         # クラッシュ時点の全プロセス
crash> vm         # 仮想メモリの状態
```

`crash`ユーティリティ（`crash-utility`パッケージ）は、Linuxカーネルのvmcoreを事後解析するための標準ツールである。

---

## oops出力のデコード {#decoding-oops-output}

### `faddr2line` {#faddr2line}

```bash
# oops 中の関数+オフセットをデコードする:
scripts/faddr2line vmlinux my_driver_read+0x2c

# 出力:
# my_driver_read (drivers/mydriver/mydriver.c:142)
```

### `addr2line` {#addr2line}

```bash
# 生のアドレスをデコードする（デバッグシンボルが必要）:
addr2line -e vmlinux -a ffffffff81234567

# または oops の RIP のオフセットを使って:
addr2line -e vmlinux -i -f ffffffff81234567
```

### `decode_stacktrace.sh` {#decode_stacktracesh}

```bash
# dmesg をカーネルのデコードスクリプトにパイプする:
dmesg | ./scripts/decode_stacktrace.sh vmlinux . /path/to/modules

# 変換前:
# my_driver_read+0x2c/0x80 [mydriver]
# 変換後:
# my_driver_read (drivers/mydriver/mydriver.c:142)
```

### 命令レベルの解析のための`objdump` {#objdump-for-instruction-level-analysis}

```bash
# フォルトした命令の周辺を逆アセンブルする:
objdump -d --start-address=0xffffffff81234540 \
           --stop-address=0xffffffff812345a0 vmlinux
```

---

## KASAN: Kernel Address SANitizer {#kasan-kernel-address-sanitizer}

KASAN（`CONFIG_KASAN`）は、use-after-freeと範囲外アクセスを検出するコンパイル時計装ツールである。各バイトが実メモリ8バイト分の有効性を追跡する「シャドウメモリ」領域を維持する。

アクセスが行われると、コンパイラが挿入した計装コードがシャドウバイトをチェックする。不正なアクセスはoops形式のレポートを生成する:

```
BUG: KASAN: use-after-free in my_driver_read+0x2c/0x80
Read of size 8 at addr ffff888012345600 by task mydriver/1234

CPU: 3 PID: 1234 Comm: mydriver Tainted: G    B
Call Trace:
 kasan_report+0xe1/0x180
 my_driver_read+0x2c/0x80

Allocated by task 1234:
 kmalloc+0x...
 my_driver_alloc+0x...

Freed by task 5678:
 kfree+0x...
 my_driver_free+0x...

The buggy address belongs to the object at ffff888012345600
 which belongs to the cache kmalloc-256 of size 256
```

Generic KASANは実メモリ8バイトあたり1シャドウバイトを使い、（シャドウ領域のために）約12.5%のメモリオーバーヘッドに加え、コンパイラが挿入した計装による2〜3倍のCPU/実行オーバーヘッドを伴う。開発やCIで使われ、本番では使われない。2つの変種が存在する:

- **Generic KASAN**（`CONFIG_KASAN_GENERIC`）: 完全なシャドウメモリ。オーバーヘッドが最も大きい
- **SW tag KASAN**（`CONFIG_KASAN_SW_TAGS`）: ARM64のtop-byte ignoreを使い、オーバーヘッドを抑える
- **HW tag KASAN**（`CONFIG_KASAN_HW_TAGS`）: ARM64 MTEハードウェアを使う。オーバーヘッドは非常に小さい

---

## KFENCE: Kernel Electric-Fence {#kfence-kernel-electric-fence}

KFENCE（`CONFIG_KFENCE`、Linux 5.12以降）は、本番環境での利用を想定して設計された、サンプリングベースのメモリ安全性検出器である。すべての割り当てを計装する代わりに、`kmalloc`と`kmem_cache_alloc`の呼び出しのうちごく一部を定期的に横取りし、返すオブジェクトを、ガードページに挟まれた特別な「保護された」ページに配置する。

ガードページへのアクセスはページフォールトを引き起こし、それをKFENCEのフォルトハンドラが捕捉して違反を報告する:

```
BUG: KFENCE: use-after-free in my_driver_read+0x2c/0x80

Use-after-free read at 0x... (in kfence-#42):
 my_driver_read+0x2c/0x80

kfence-#42 [0x...-0x...], size=256, cache=kmalloc-256
 allocated by task 1234:
  my_driver_alloc+0x... at ...

 freed by task 5678:
  my_driver_free+0x... at ...
```

サンプリングレートは`CONFIG_KFENCE_SAMPLE_INTERVAL`（デフォルト100ms）で制御され、実行時に`/sys/module/kfence/parameters/sample_interval`で調整できる。オーバーヘッドは本番システムで使えるほど小さく、典型的にはCPUへの影響は1%未満である。

KFENCEはすべてのバグを捕捉するわけではない（保護されるのはサンプリングされた割り当てだけである）が、本番環境で長期にわたって継続的なカバレッジを提供し、開発時にしか使われないKASANでは見逃されるバグを捕捉することも多い。

---

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/panic.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/panic.c) — `panic()`/`vpanic()`: メッセージの整形、パニックnotifierチェーン、再起動/停止のシーケンス
- [kernel/crash_core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/crash_core.c) — `kexec_should_crash()`と`crash_kexec()`: kdumpのエントリポイント。`oops_end()`は公開版の`crash_kexec()`を呼び出し、`vpanic()`はパニックCPUチェックを迂回するために内部の`__crash_kexec()`を直接呼び出す
- [arch/x86/kernel/dumpstack.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/dumpstack.c) — `die()`、`oops_begin()`、`oops_end()`: 実際のx86のoopsパス
- [arch/x86/kernel/traps.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/traps.c) — `die()`を呼び出す例外エントリポイント（`exc_general_protection()`、`exc_invalid_op()`、`exc_double_fault()`など）
- [arch/x86/kernel/unwind_orc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/unwind_orc.c) — `orc_find()`: `Call Trace`の生成に使われるORCエントリの検索
- [arch/x86/include/asm/orc_types.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/orc_types.h) — `struct orc_entry`のレイアウト
- [include/asm-generic/bug.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/asm-generic/bug.h) — `WARN()`/`BUG()`ファミリと`__warn()`の宣言。x86で実際にそこへ到達するのは`warn_slowpath_fmt()`ではなく、トラップベースのパス（`exc_invalid_op()`から呼ばれる`report_bug()`）である
- [lib/bust_spinlocks.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/bust_spinlocks.c) — `bust_spinlocks()`: oops中にコンソール出力を読める状態に保つために使われる`oops_in_progress`カウンタ

### manページ {#man-pages}

- [`kexec_load(2)`](https://man7.org/linux/man-pages/man2/kexec_load.2.html) — `kexec -p`の背後にあるシステムコール。クラッシュカーネルを登録するフラグ`KEXEC_ON_CRASH`を説明している

### 関連ページ {#related-pages}

- [Oops解析](../debugging/oops-analysis.md) — oopsの読み方のステップバイステップガイド
- [kdumpとcrash](../debugging/kdump.md) — kdumpのセットアップと利用の詳細
- [KASAN](../mm/kasan.md) — KASANの設定と利用
- [KFENCE](../mm/kfence.md) — 本番環境でのKFENCE

### LWN記事 {#lwn-articles}

- [ORC unwinder](https://lwn.net/Articles/728721/) — ORCアンワインダとobjtoolが生成するアンワインドテーブルを導入した、Josh Poimboeufのパッチシリーズと設計の解説（2017年）
- [Crash dumps with kexec](https://lwn.net/Articles/108595/) — 元祖のkexecベースのクラッシュダンプ設計についてのJonathan Corbetの記事（2004年）
- [The kernel address sanitizer](https://lwn.net/Articles/612153/) — Jake EdgeによるKASANの紹介（2014年）
- [KFENCE: A low-overhead sampling-based memory safety error detector](https://lwn.net/Articles/830877/) — Marco ElverによるKFENCEの設計解説（2020年）

### 外部資料 {#external}

- [Tainted kernels](https://docs.kernel.org/admin-guide/tainted-kernels.html) — taintフラグの完全なリファレンス
- [Kdump](https://docs.kernel.org/admin-guide/kdump/kdump.html) — kdumpのセットアップガイド
