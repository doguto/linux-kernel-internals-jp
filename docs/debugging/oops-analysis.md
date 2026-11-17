# カーネルOops解析 {#kernel-oops-analysis}

> oops出力を読み、アドレスをデコードし、バグを見つける

## カーネルoopsとは {#what-is-a-kernel-oops}

**カーネルoops**は致命的ではないカーネルエラーである。カーネルが不整合（通常はNULLポインタ参照外し、不正なメモリアクセス、またはBUG()）を検出したが、（問題を起こしたタスクはkillされるものの）動作を継続できる状態を指す。**カーネルパニック**は致命的なoops、またはカーネルが安全に継続できない状況である。

## oopsメッセージの構造 {#anatomy-of-an-oops-message}

```
[  123.456789] BUG: kernel NULL pointer dereference, address: 0000000000000018
[  123.456790] #PF: supervisor read access in kernel mode
[  123.456791] #PF: error_code(0x0000) - not-present page
[  123.456792] PGD 0 P4D 0
[  123.456793] Oops: Oops: 0000 [#1] SMP NOPTI
```

「Oops: Oops:」の重複は実際の出力であり、誤植ではない。`__die_header()`（`arch/x86/kernel/dumpstack.c`）は常に`"Oops: %s: ..."`を出力し、汎用のカーネルモードページフォールトハンドラ（`arch/x86/mm/fault.c`）はその`%s`としてリテラル文字列`"Oops"`を渡すため、両者が重なるのである。`[#1]`の後のフラグは条件付きで、実行中のカーネルの設定から組み立てられる: `SMP`（`CONFIG_SMP`）、`DEBUG_PAGEALLOC`（`debug_pagealloc_enabled()`の場合）、`KASAN`（`CONFIG_KASAN`）、そして`PTI`/`NOPTI`（`CONFIG_MITIGATION_PAGE_TABLE_ISOLATION`。CPUがその緩和策を必要とする場合は`PTI`、そうでなければ`NOPTI`）である。この行に`PREEMPT`フラグは一切ない。

1行目を分解すると:

- `BUG: kernel NULL pointer dereference` — フォールトの種類
- `address: 0x18` — フォールトを引き起こした仮想アドレス（0x18 = 構造体内のオフセット24）

エラーコード`0000`:

- ビット0 = 0: ページが存在しない（保護違反ではない）
- ビット1 = 0: 読み出し（書き込みではない）
- ビット2 = 0: カーネルモード（ユーザーモードではない）

```
[  123.456794] CPU: 3 PID: 1234 Comm: myapp Tainted: G  --------- -  5.19.0 #1
[  123.456795] Hardware name: QEMU Standard PC, BIOS rel-1.16.0
[  123.456796] RIP: 0010:mydriver_write+0x34/0xb0 [mydriver]
```

- `CPU: 3` — 実行していたCPU
- `PID: 1234 Comm: myapp` — oopsを引き起こしたタスク
- `Tainted: G` — カーネルのtaintフラグ（G = ロード済みモジュールがすべてGPL互換で、プロプライエタリなモジュールがない）
- `RIP: 0010:` — 命令ポインタ。`0010` = カーネルのコードセグメント
- `mydriver_write+0x34/0xb0` — 関数 + オフセット / 関数のサイズ

### taintフラグ {#taint-flags}

```
G — ロード済みモジュールがすべてGPL互換（プロプライエタリなモジュールなし）
P — プロプライエタリ（非GPL）モジュールがロードされた
F — モジュールが強制ロードされた（不正な署名またはバージョン不一致）
S — 仕様外のCPU/システムでカーネルが動作している
M — マシンチェック例外
B — 不良ページ
U — ユーザー（ユーザー空間が明示的に設定した）
D — died（OOPSが記録された。以後taint状態となる）
A — ACPIテーブルが上書きされた
W — 警告（WARN()でtaintする）
C — stagingドライバがロードされた
I — プラットフォームファームウェアのバグに対する回避策が適用された
K — ライブパッチが適用された
```

## レジスタダンプ {#register-dump}

```
[  123.456797] RSP: 0018:ffffc900012c7d28 EFLAGS: 00010246
[  123.456798] RAX: 0000000000000000 RBX: ffff888012345678 RCX: 0000000000000040
[  123.456799] RDX: 0000000000000001 RSI: 00007fff12345678 RDI: ffff888087654321
[  123.456800] RBP: ffffc900012c7d60 R08: 0000000000000000 R09: 0000000000000000
[  123.456801] R10: 0000000000000000 R11: 0000000000000246 R12: ffff888087654321
[  123.456802] R13: ffff888012345678 R14: 0000000000000040 R15: 00007fff12345678
[  123.456803] FS:  00007f1234567890(0000) GS:ffff88813fc00000(0000) knlGS:0000000000000000
[  123.456804] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[  123.456805] CR2: 0000000000000018 CR3: 000000010a234000 CR4: 00000000003506e0
```

診断の鍵となるレジスタ:

- `RIP` — クラッシュが起きた場所
- `CR2` — フォールトしたアドレス（ページフォールトの場合）
- `RAX = 0` — 参照外しされたNULLポインタであることが多い
- `RBX`、`R12-15` — 構造体ポインタを保持していることが多く、データ構造の手がかりになる

**オフセットのトリック**: `CR2 = 0x18 = 24`。NULLポインタからオフセット24にある構造体メンバである。構造体のフィールドを数えれば、どのメンバにアクセスしたかを特定できる。

## スタックトレース {#stack-trace}

```
[  123.456806] Call Trace:
[  123.456807]  <TASK>
[  123.456808]  mydriver_write+0x34/0xb0 [mydriver]
[  123.456809]  vfs_write+0xb5/0x2a0
[  123.456810]  ksys_write+0x67/0xe0
[  123.456811]  __x64_sys_write+0x1d/0x30
[  123.456812]  do_syscall_64+0x3b/0x80
[  123.456813]  entry_SYSCALL_64_after_hwframe+0x63/0xcd
[  123.456814]  </TASK>
```

上（最も新しい）から下（最も古い呼び出し）へと読む。クラッシュは`mydriver_write`内のオフセット`0x34`で起きた。

## addr2line: ソース行を見つける {#addr2line-finding-the-source-line}

フォールトしたアドレスから、正確なソース行を見つける:

```bash
# 方法1: vmlinuxに対してaddr2lineを使う
addr2line -e /usr/lib/debug/vmlinux ffffffff81234567
# /build/linux/drivers/mydriver/mydriver.c:245

# 関数名付きで:
addr2line -e vmlinux -f ffffffff81234567
# mydriver_write
# /build/linux/drivers/mydriver/mydriver.c:245

# 方法2: モジュールの場合（stripされていない.koが必要）
addr2line -e mydriver.ko 0x34   # モジュール内のオフセット
# /path/to/mydriver.c:245
```

## decode_stacktrace.sh {#decode_stacktracesh}

カーネルには、スタックトレース中のすべてのアドレスを解決するスクリプトが同梱されている:

```bash
# oopsをdecode_stacktraceにパイプで渡す
./scripts/decode_stacktrace.sh /path/to/vmlinux /path/to/kernel/source < oops.txt

# モジュールも含める場合:
./scripts/decode_stacktrace.sh vmlinux . mydriver.ko < oops.txt
```

出力では各`function+0xNN/0xNN`が`function (file.c:line)`に解決される。

## objdump: クラッシュ周辺の逆アセンブル {#objdump-disassembling-around-the-crash}

```bash
# クラッシュした関数を逆アセンブルする
objdump -d --start-address=0xffffffff81234500 --stop-address=0xffffffff81234600 vmlinux

# モジュールの場合:
objdump -d mydriver.ko | grep -A 30 "<mydriver_write>"
```

クラッシュは`mydriver_write`のオフセット`0x34`で起きた:

```
0000000000000000 <mydriver_write>:
   0: push %rbp
   1: mov %rsp,%rbp
  ...
  34: mov 0x18(%rax),%rdx    ← ここでクラッシュ: raxがNULLで、オフセット0x18にアクセスした
  38: test %rdx,%rdx
```

ここから、`%rax`がNULLで、コードがそこからオフセット0x18のメンバを読もうとしたことがわかる。

## よくあるoopsのパターン {#common-oops-patterns}

### NULLポインタ参照外し {#null-pointer-dereference}

```
BUG: kernel NULL pointer dereference, address: 0000000000000018
RIP: mydriver_write+0x34/0xb0
RAX: 0000000000000000
```

診断: `RAX = 0`（NULLポインタ）。オフセット`0x18`からどのフィールドかがわかる。そのオフセットにある構造体のメンバを確認する。

```c
/* 構造体が次のようなものだとすると: */
struct mydev {
    spinlock_t lock;      /* オフセット0 */
    void *private;        /* オフセット8 */
    struct device *dev;   /* オフセット16 = 0x10 */
    int refcount;         /* オフセット24 = 0x18 ← このフィールド */
};
/* オフセット0x18の読み出し = NULLのmydev*からmydev->refcountを読むこと */
```

### use-after-free {#use-after-free}

```
BUG: KASAN: slab-use-after-free in mydriver_read+0x45/0x100
Read of size 8 at addr ffff888012345678 by task myapp/1234
Freed by task 5678:
  kfree+0x...
  mydriver_cleanup+0x...
Allocated by task 1234:
  kmalloc+0x...
  mydriver_probe+0x...
```

KASANはメモリがどこで割り当てられ、どこで解放されたかを正確に示す。競合状態の調査では非常に貴重である。

### スタックオーバーフロー {#stack-overflow}

`CONFIG_VMAP_STACK`（デフォルト）では、カーネルスタックは両側にマップされていないガードページを置いて割り当てられるため、スタックをはみ出すと、後続のメモリを黙って破壊するのではなく即座にフォールトする。`handle_stack_overflow()`（`arch/x86/kernel/traps.c`）がそのフォールトを捕まえて報告し、続いて`die("stack guard page", regs, 0)`を呼んで通常のoopsを出力する:

```
BUG: IRQ stack guard page was hit at ffffc900012c8000 (stack is ffffc900012c4000..ffffc900012c8000)
Oops: stack guard page: 0000 [#1] SMP NOPTI
```

スタックの種類（「IRQ」、「task」、...）は`stack_type_name()`から得られる。タスクスタックのオーバーフローなら代わりに「task stack guard page was hit」と表示される。カーネルスタックは通常8〜16KBである。深い再帰（例: overlayfsでの深いVFS再帰）はこれをオーバーフローさせうる。

### ソフトロックアップ {#soft-lockup}

```
watchdog: BUG: soft lockup - CPU#3 stuck for 22s! [kworker/3:1:5678]
Modules linked in: ...
CPU: 3 PID: 5678 Comm: kworker/3:1
Call Trace:
  <IRQ>
  do_something_slow+0x123/0x456
```

CPUが20秒以上スケジュールしていない。原因は通常、spinlockを長く保持しすぎているか、`cond_resched()`のないタイトなループである。

### RCUストール {#rcu-stall}

```
rcu: INFO: rcu_sched self-detected stall on CPU
rcu:    2-O.N.: (1 GPs behind) idle=... softirq=100/205 fqs=0
rcu:     (t=21012 jiffies g=4989 q=157 ncpus=4)
```

RCUの読み取り側クリティカルセクションが長く保持されすぎているか、RCUのグレースピリオドが停滞している。プリエンプションを無効にしたまま長時間ロックを保持していることが原因であることが多い。

## BUG()とWARN() {#bug-and-warn}

```c
/* カーネルコード内: 特定の箇所で強制的にoopsさせる */
BUG();                      /* 常に: oops + スタックトレース */
BUG_ON(condition);          /* 条件付き */

WARN();                     /* ログ + スタックトレースを出すが、実行は継続 */
WARN_ON(condition);
WARN_ON_ONCE(condition);    /* 1回だけ出力 */
WARN_ONCE(condition, fmt);
```

`WARN`は、システムをクラッシュさせずに予期しない状態を検出するのに役立つ。システムが正常に見えても、`dmesg`に`WARNING:`行がないか確認するとよい。

## netconsole: ネットワーク経由でoopsを取得する {#netconsole-capture-oops-over-the-network}

シリアルコンソールを見られないヘッドレスサーバー向け:

```bash
# netconsoleをロードする（またはモジュールとして設定する）
modprobe netconsole netconsole=@192.168.1.10/eth0,@192.168.1.1/00:11:22:33:44:55
# 書式: @src_ip/src_dev,@dst_ip/dst_mac

# 受信側のマシンで:
nc -ulp 6665   # netconsoleのデフォルトポートで待ち受ける

# あるいは動的netconsoleで（configfsを使う）:
mkdir /sys/kernel/config/netconsole/target1
echo eth0              > /sys/kernel/config/netconsole/target1/dev_name
echo 192.168.1.10     > /sys/kernel/config/netconsole/target1/local_ip
echo 192.168.1.1      > /sys/kernel/config/netconsole/target1/remote_ip
echo 00:11:22:33:44:55 > /sys/kernel/config/netconsole/target1/remote_mac
echo 1                > /sys/kernel/config/netconsole/target1/enabled
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [arch/x86/include/asm/trap_pf.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/trap_pf.h) — ページフォールトの`error_code`の内訳（ビット0 = 不在/保護違反、ビット1 = 読み出し/書き込み、ビット2 = カーネル/ユーザー）の元になる`X86_PF_*`ビット定義
- [kernel/panic.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/panic.c) — `taint_flags[]`テーブル: `Tainted:`行の文字とフラグの対応の正典（各フラグの人間向けの意味は、下で挙げる[tainted-kernels.rst](https://docs.kernel.org/admin-guide/tainted-kernels.html)を参照）
- [scripts/decode_stacktrace.sh](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/decode_stacktrace.sh) — 取得したoops中の`function+0xNN/0xNN`エントリを`function (file.c:line)`に解決する
- [Documentation/networking/netconsole.rst](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/networking/netconsole.rst) — `netconsole=`モジュールパラメータの書式と、configfsベースの動的再設定インターフェース（`dev_name`、`local_ip`、`remote_ip`、`remote_mac`、`enabled`、...）

### manページ {#man-pages}

- [`addr2line(1)`](https://man7.org/linux/man-pages/man1/addr2line.1.html) — アドレス（または`function+offset`）をファイル名と行番号に変換する
- [`objdump(1)`](https://man7.org/linux/man-pages/man1/objdump.1.html) — オブジェクトファイルを逆アセンブルする。ここではフォールトした命令を調べるために`--start-address`/`--stop-address`とともに使った

### 関連ページ {#related-pages}

- [カーネルパニックとOops](../kernel/panic-oops.md) — カーネル側のコードパス: `die()`、`oops_begin()`/`oops_end()`、ORCアンワインド、`panic()`
- [kdumpとcrash](kdump.md) — 自動的なクラッシュダンプ収集
- [KGDB](kgdb.md) — 稼働中カーネルのデバッグ
- [メモリ管理: KASAN](../mm/kasan.md) — メモリエラー検出
- [メモリ管理: KFENCE](../mm/kfence.md) — 本番環境向けの軽量なメモリチェック
- [トレーシング: kprobe](../tracing/kprobes-tracepoints.md) — クラッシュさせずにプローブする

### 外部資料 {#external}

- [Tainted kernels](https://docs.kernel.org/admin-guide/tainted-kernels.html) — すべての文字について公式の説明を記載した、taintフラグの完全なリファレンス
