# kdumpとcrash {#kdump-and-crash}

> カーネルクラッシュダンプの取得と解析

## kdumpの仕組み {#how-kdump-works}

kdumpは`kexec`を使い、2つ目の「キャプチャカーネル」をメモリの予約領域にあらかじめロードしておく。プライマリカーネルがクラッシュ（パニック）すると、kexec経由でキャプチャカーネルに制御を渡す。キャプチャカーネルは起動し、1つ目のカーネルのメモリ（「vmcore」）をディスクに保存して再起動する。

```
Normal operation:
  Primary kernel runs
  Capture kernel pre-loaded at reserved address (e.g., 0x100000000)
  ────────────────────────────────────────────────
Crash:
  Primary kernel panics
  kexec jumps to capture kernel entry point
  ────────────────────────────────────────────────
Capture kernel:
  Boots with crashkernel= memory
  Mounts root filesystem
  /proc/vmcore: primary kernel's memory accessible here
  makedumpfile saves compressed vmcore to /var/crash/
  System reboots
```

## セットアップ {#setup}

### ステップ1: キャプチャカーネル用のメモリを予約する {#step-1-reserve-memory-for-capture-kernel}

カーネルのブートパラメータに追加する:

```bash
# /etc/default/grub（または同等のファイル）
GRUB_CMDLINE_LINUX="crashkernel=256M"
# キャプチャカーネル用に256MBを高位アドレスに予約する

# 4GBを超えるRAMを持つシステムの場合（一般的）:
crashkernel=256M,high   # 高位メモリを優先
crashkernel=64M,low     # 低位メモリにも64MBを予約（DMA用）

# grubの設定を再生成する
sudo update-grub
sudo reboot
```

```bash
# 起動後に予約を確認する
cat /proc/cmdline | grep crashkernel
dmesg | grep "crashkernel"
# crashkernel: memory value: 0x10000000 at 0x100000000
```

### ステップ2: kdumpをインストールして設定する {#step-2-install-and-configure-kdump}

```bash
# Debian/Ubuntu
sudo apt install kdump-tools

# RHEL/Fedora
sudo dnf install kexec-tools

# kdumpサービスを開始する（kexec経由でキャプチャカーネルをロードする）
sudo systemctl enable --now kdump

# kdumpの準備ができていることを確認する
sudo kdump-config show
# DUMP_MODE:        kdump
# USE_KDUMP:        1
# KDUMP_SYSCTL:     kernel.panic_on_oops=1
# current state:    ready to kdump
```

### ステップ3: kdumpをテストする {#step-3-test-kdump}

```bash
# テストのために強制的にクラッシュさせる（クラッシュダンプを書き出してから再起動する）
echo 1 > /proc/sys/kernel/sysrq
echo c > /proc/sysrq-trigger

# 再起動後: /var/crash/を確認する
ls /var/crash/
# 202601301234/  vmcore  vmcore-dmesg.txt
```

## crashによるクラッシュダンプの解析 {#analyzing-a-crash-dump-with-crash}

`crash`ツールはvmcoreファイルを解析する標準的な方法である:

```bash
# crashをインストールする
sudo apt install crash   # Debian/Ubuntu
sudo dnf install crash   # RHEL

# クラッシュダンプを開く
crash /usr/lib/debug/boot/vmlinux-$(uname -r) /var/crash/*/vmcore
# あるいはカーネルのdebuginfoパッケージを使う場合:
crash /path/to/vmlinux vmcore
```

### crashコマンドリファレンス {#crash-command-reference}

crashの中に入ったら:

```
crash> help          # 全コマンドを一覧表示

# システム情報
crash> sys           # カーネルバージョン、パニック文字列、稼働時間
# KERNEL: /usr/lib/debug/.../vmlinux
# DUMPFILE: vmcore  [PARTIAL DUMP]
# CPUS: 8
# DATE: Mon Jan 30 12:34:56 2026
# UPTIME: 2 days, 03:21:45
# PANIC: "general protection fault, maybe for address..."

# パニックメッセージとタスク
crash> bt            # クラッシュしたタスクのバックトレース
crash> bt -a         # 全タスクのバックトレース
crash> bt <pid>      # 特定タスクのバックトレース

# プロセス情報
crash> ps            # プロセス一覧（ps auxと同様）
crash> ps -t         # スレッド一覧
crash> task <pid>    # プロセスのtask_struct全体
crash> files <pid>   # プロセスが開いているファイル

# メモリの調査
crash> rd address 16          # addressから16ワードを読む
crash> wr address value       # 書き込み（vmcoreを変更する — 危険）
crash> kmem -i                # メモリ情報
crash> kmem -z                # ゾーン情報
crash> kmem -s                # slab情報

# カーネル構造体
crash> struct task_struct <address>      # 構造体を整形表示
crash> p variable_name                   # 変数を表示
crash> p *((struct inode *)0xffff..)    # ポインタを参照外し

# モジュール
crash> mod                    # モジュール一覧
crash> mod -s mymodule /path/to/mymodule.ko  # モジュールのシンボルをロード

# 仮想メモリ
crash> vm <pid>               # プロセスのVMA
crash> pte <address>          # ページテーブルエントリ
crash> vtop <virtual>         # 仮想→物理アドレス変換

# ログ
crash> log                    # vmcore中のカーネルリングバッファ
crash> log -m                 # タイムスタンプのプレフィックスを含める
```

### バックトレースの読み方 {#reading-the-backtrace}

```
crash> bt
PID: 1234   TASK: ffff888012345678  CPU: 3   COMMAND: "myapp"
 #0 [ffff8880abcd0000] machine_kexec at ffffffff81064180
 #1 [ffff8880abcd0058] __crash_kexec at ffffffff8112e943
 #2 [ffff8880abcd0120] crash_kexec at ffffffff8112ea8c
 #3 [ffff8880abcd0138] oops_end at ffffffff810512e1
 #4 [ffff8880abcd0160] no_context at ffffffff81065e28
 #5 [ffff8880abcd01b0] __bad_area_nosemaphore at ffffffff81066037
 #6 [ffff8880abcd0200] bad_area_nosemaphore at ffffffff81066167
 #7 [ffff8880abcd0210] do_user_addr_fault at ffffffff81066b8e
 #8 [ffff8880abcd0280] exc_page_fault at ffffffff8106765f
 #9 [ffff8880abcd02a0] asm_exc_page_fault at ffffffff81c00b62
    RIP: 0033:0x7f1234abcd00   RSP: 0018:ffff8880abcd0300   EFLAGS: 00010246
    RAX: 0000000000000000  RBX: ffff888001234567  RCX: 0000000000000000
```

各フレームには、フレーム番号、カーネルスタックのアドレス、関数名、ソースアドレスが表示される。

### クラッシュしたコードの調査 {#inspecting-the-crashing-code}

```
# フォールトした命令を見つける
crash> dis -l ffffffff81066b8e     # ソース行付きで逆アセンブル
# do_user_addr_fault():
# /build/linux/arch/x86/mm/fault.c: 1402
# 0xffffffff81066b80: mov    (%rbx),%rax    ← ここでNULL参照外し
# 0xffffffff81066b83: test   %rax,%rax

# フォールトした変数を表示する
crash> p my_struct->field

# 関数がどのモジュールに属するかを調べる
crash> sym ffffffffc0a01234
# ffffffffc0a01234 (t) mymodule_function [mymodule]
```

## makedumpfile {#makedumpfile}

生のvmcoreファイルは非常に大きくなりうる（物理RAMと同じサイズ）。`makedumpfile`はそれを圧縮し、フィルタリングする:

```bash
# ゼロページと空きページを除外した圧縮ダンプを作成する
makedumpfile -c -d 31 /proc/vmcore /var/crash/dump

# 除外レベル: -d N の各ビットの意味:
# bit 0 (1): ゼロページ
# bit 1 (2): キャッシュページ
# bit 2 (4): キャッシュ（プライベート）
# bit 3 (8): ユーザー空間ページ
# bit 4 (16): 空きページ
# 31 = 上記すべて

# 別のマシンで後から解析するために:
makedumpfile --dump-dmesg /proc/vmcore dmesg.txt
```

## pstore: 再起動をまたぐ永続ストレージ {#pstore-persistent-storage-across-reboots}

組み込みシステムや、完全なkdumpが現実的でない場合には、pstoreがカーネルのコンソールログとoopsのバックトレースを永続ストレージ（ACPI APEI、NVRAM、MTDフラッシュ、ramoops）に保存する:

```bash
# ramoops: kexecをまたいで残るRAM領域を予約する
# （ブートパラメータに追加する）
ramoops.mem_address=0x8f000000 ramoops.mem_size=0x100000

# クラッシュ/再起動後、保存されたログを読む:
ls /sys/fs/pstore/
# dmesg-ramoops-0   (console output before crash)
# console-ramoops-0

cat /sys/fs/pstore/dmesg-ramoops-0

# 読んだ後に消去する（pstoreのファイルは削除することで消える）
rm /sys/fs/pstore/dmesg-ramoops-0
```

## パニック時の挙動の確認 {#observing-panic-behavior}

```bash
# パニック時の動作を制御する
cat /proc/sys/kernel/panic
# 0 = ハングする（デフォルト）
# N > 0 = N秒後に再起動
# N < 0 = 即座に再起動
echo 10 > /proc/sys/kernel/panic

# あらゆるoopsでパニックする（致命的なものだけでなく）
echo 1 > /proc/sys/kernel/panic_on_oops

# ソフトロックアップでパニックする（ハングタスクのウォッチドッグ）
echo 1 > /proc/sys/kernel/softlockup_panic

# ハングタスクのタイムアウト（タスクがN秒停止したらパニック）
echo 120 > /proc/sys/kernel/hung_task_timeout_secs
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/panic.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/panic.c) — `panic_timeout`と`panic_on_oops`: `kern_panic_table[]`にある`kernel.panic`および`kernel.panic_on_oops` sysctlエントリの実体となる変数
- [kernel/watchdog.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/watchdog.c) — `softlockup_panic`: `kernel.softlockup_panic`の実体となる変数で、ソフトロックアップのウォッチドッグが発火したときに確認される
- [kernel/hung_task.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/hung_task.c) — `sysctl_hung_task_timeout_secs`: `kernel.hung_task_timeout_secs`の実体
- [fs/pstore/ram.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/pstore/ram.c) — `mem_address`/`mem_size`モジュールパラメータ: ramoops領域の予約に使う`ramoops.mem_address=`/`ramoops.mem_size=`ブートパラメータの実体

### manページ {#man-pages}

- [`kexec_load(2)`](https://man7.org/linux/man-pages/man2/kexec_load.2.html) — `KEXEC_ON_CRASH`を説明している。これはカーネルイメージを、`crashkernel=`で予約されたメモリにロードされるクラッシュカーネルとして登録するフラグである

### 関連ページ {#related-pages}

- [Oops解析](oops-analysis.md) — crashツールを使わずにoopsを読む
- [KGDB](kgdb.md) — 稼働中カーネルのデバッグ
- [カーネルパニックとOops](../kernel/panic-oops.md) — 実際にkdumpを起動する`die()`/`panic()`のコードパス（`crash_kexec()`/`__crash_kexec()`）
- [メモリ管理: KASAN](../mm/kasan.md) — クラッシュする前にuse-after-freeを捕まえる
- [メモリ管理: KFENCE](../mm/kfence.md) — 軽量なメモリエラー検出

### LWN記事 {#lwn-articles}

- [Crash dumps with kexec](https://lwn.net/Articles/108595/) — kexecベースのクラッシュダンプの当初の設計に関するJonathan Corbetの記事（2004）
- [Persistent storage for a kernel's "dying breath"](https://lwn.net/Articles/434821/) — Jake Edgeによるpstoreの紹介（2011）。当時は別の仕組みだったramoopsは後にpstoreのバックエンドとしてリファクタリングされ、現在はその形で動作している

### 外部資料 {#external}

- [Kdump](https://docs.kernel.org/admin-guide/kdump/kdump.html) — 公式のkdumpセットアップガイド: `crashkernel=`の構文、`/proc/vmcore`、`makedumpfile`の使い方
- [Ramoops oops/panic logger](https://docs.kernel.org/admin-guide/ramoops.html) — ramoops pstoreバックエンド。`mem_address`/`mem_size`パラメータを含む
