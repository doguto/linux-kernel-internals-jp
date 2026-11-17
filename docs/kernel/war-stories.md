# カーネルコアの事件簿 {#kernel-core-war-stories}

> 初期化順序、ブートパラメータのバグ、パニックループ、その他

カーネルコアの基盤 — initcall、ブートパラメータ、`__init`セクション、panic/kdump、モジュールのロード — は成熟しており、十分にテストされている。しかし、これらの機構と、現実のドライバ、設定、デプロイ上の前提との相互作用は、本当に診断の難しいバグを生み出す。その失敗は、無症状であったり、遅れて現れたり、自らを隠蔽したりする傾向がある。

!!! note "これらの事例の書き方について"
    これらは、1つのコミットやCVEに結び付いた特定の名前付きインシデントの記録ではない — 長年にわたり、さまざまなドライバやディストリビューションで繰り返し現れてきた、実在の反復的なバグパターンを例示するものである。ページ末尾の参考資料は、根底にあるカーネルの機構を現在のソースとドキュメントに照らして検証している。このサイトの通常の形式 — 特定の、日付を特定できるインシデントで、そのほとんど（すべてではない — [wake_wide()](../sched/war-stories/wake-wide-heuristic.md)を参照）が修正コミットやCVEに結び付いているもの — については、[BPF](../bpf/war-stories.md)、[スケジューラ](../sched/war-stories.md)、[割り込み](../interrupts/war-stories.md)の事件簿ページを参照。

以下の各事例は、カーネルコアの動作に関する基本的な性質と、コードがそれについて誤った前提を置いたときに何が起こるかを示している。

---

## 事例1: ネットワークスタックを壊すinitcallの順序バグ {#case-1-initcall-ordering-bug-breaking-the-network-stack}

### 問題 {#the-problem}

あるチームが、新しいEthernetドライバを組み込み製品に統合していた。そのドライバは初期化関数の中でnetdevを登録し、すぐにネットワークコアを呼び出して統計情報の基盤をセットアップしていた。開発用カーネルではドライバは問題なく動作した。しかし、わずかに異なるKconfigでビルドされた本番用カーネルでは、`net_device_ops`でのNULLポインタ参照によりシステムがブート時にハングした。

oopsは`calling ... returned ...`というinitcallのデバッグ出力の途中、`core_initcall`レベル（レベル1）の途中で、`device_initcall`のドライバがまだ1つも実行されていない時点で発生していた。

### 診断 {#diagnosis}

本番用カーネルで`initcall_debug`を有効にすると、oopsのトレースは次のようになった:

```
calling  mydriver_core_init+0x0/0x68 @ 1
BUG: unable to handle kernel NULL pointer dereference at 0000000000000018
RIP: mydriver_net_stats_init+0x34/0x90
Call Trace:
 mydriver_core_init+0x58/0x68
 do_one_initcall+0x58/0x190
```

ドライバのinit関数は`mydriver_net_stats_init()`を呼び出しており、それがネットワークサブシステムを呼び出していた。しかしoopsは、ドライバが`core_initcall`レベル（レベル1）で実行されていたことを示していた。ネットワークコアは`net/core/dev.c`の`net_dev_init()`によって`subsys_initcall`レベル（レベル4）で初期化される。

レベル1はレベル4より前に実行される。ネットワークサブシステムの内部状態 — 特にper-CPUのsoftnet_data構造体 — はまだ割り当てられていなかった。ドライバは未初期化のメモリに手を伸ばしていたのである。

### 根本原因 {#root-cause}

ドライバの登録マクロは次のようになっていた:

```c
/* 誤り: ネットワークの準備ができる前に実行される */
core_initcall(mydriver_core_init);
```

`core_initcall`と書かれていたのは、開発者が「core」を「これはinitcallレベル1で、ネットワークのようなサブシステムより前に実行される」という意味ではなく、「これは我々のシステムの中核部分である」という意味に取り違えたためだった。

### 修正 {#fix}

```c
/* 正しい: すべてのサブシステム初期化の後、レベル 6 で実行される */
module_init(mydriver_core_init);
/* または同等に: device_initcall(mydriver_core_init); */
```

`device_initcall`（または同等の`module_init`）に変更すると、ドライバはinitcallレベル6に配置され、`net_dev_init()`が実行される`subsys_initcall`レベル4より後になる。初期化はただちに成功した。

### 教訓 {#lesson}

initcallレベルの名前は、*あなたの*コードが何であるかを記述するものではない — 他のものと比べてあなたのコードがいつ実行されるかを記述するものである。ほとんどのドライバは`device_initcall`（`module_init`）に属する。特定のサブシステムより前に実行する明示的な理由がある場合にのみ、より早いレベルを使い、呼び出すすべてのサブシステムがより早いレベルで初期化されることを確認すること。`initcall_debug`ブートパラメータを使えば、これを実行時に可視化できる。

---

## 事例2: `__init`セクションの誤用による「Freeing init memory」クラッシュ {#case-2-freeing-init-memory-crash-due-to-__init-section-misuse}

### 問題 {#the-problem_1}

あるストレージドライバは、いくつものカーネルバージョンにわたって正しく動作していた。リファクタリングを行った後、フィールドエンジニアから、そのドライバをロードしたシステムがブートから約30分後にoopsするという報告があった — 具体的には、メモリ逼迫時にドライバが特定の種類のI/Oリクエストを処理したときである。oopsは次のとおりだった:

```
BUG: Bad page state in process kworker/u8:2 pfn:12a4b0
page:ffffea0004a92c00 refcount:0 mapcount:0 mapping:0000000000000000
flags: 0x200000000(reserved)
...
Call Trace:
 free_pages_prepare+0x1a4/0x2b0
 __free_pages_ok+0x29/0xd0
 my_driver_handle_io+0x7c/0x1a0
```

`flags: reserved`を伴う`Bad page state`は、`free_initmem()`によって解放されたメモリにアクセスしたことを示す特徴的なシグネチャである。

### 診断 {#diagnosis_1}

`reserved`フラグは、initセクションに属するページに設定される。`free_initmem()`が実行されると、これらのページはアロケータに返却され、`reserved`フラグはクリアされる。reservedフラグ付きページでの「Bad page state」oopsは、コードがinitメモリへのポインタを保持しており、そのページが解放・再割り当てされた後に使おうとしたことを意味する。

初期化中に保存されて後で使われるものをドライバ内で探したところ、問題のパターンが浮かび上がった:

```c
/* ドライバのヘッダ内: */
struct my_driver_ops {
    int (*handle_compressed)(struct request *rq, const u8 *buf, size_t len);
    /* ... */
};

/* ドライバ内: */
static struct my_driver_ops global_ops;

static int __init decompress_init_helper(struct request *rq,
                                          const u8 *buf, size_t len)
{
    /* 圧縮されたファームウェアを処理するために初期化中のみ使われる */
    return decompress_and_store(rq, buf, len);
}

static int __init mydriver_init(void)
{
    global_ops.handle_compressed = decompress_init_helper; /* バグ */
    /* ... */
    return 0;
}

/* 後で、実行時に: */
static void my_driver_handle_io(struct request *rq)
{
    if (rq->flags & MY_COMPRESSED)
        global_ops.handle_compressed(rq, rq->data, rq->len); /* oops */
}
```

`decompress_init_helper`には`__init`が付けられており、`.init.text`に配置されていた。関数ポインタは、（解放されない）`.data`にある`global_ops`に格納されていた。`free_initmem()`が実行された後、initテキストのページはアロケータに返却された。やがてそのうちの1つが別の用途に再割り当てされた。メモリ逼迫時に（それまで実行されたことのなかったコードパスを通って）`handle_compressed`が呼ばれたとき、関数ポインタは、その時点でslabオブジェクトとして使われていたページを指していた。それを関数として呼び出した結果、bad page stateのoopsが発生した。

このバグは稼働時間中ずっと潜伏しており、特定のリクエストが到着したときにのみ発現した — そのため、リファクタリングとは無関係に見えた。

### 根本原因 {#root-cause_1}

リファクタリングで`decompress_init_helper`は新しいファイルに移され、ある開発者が「初期化中にしか使われない」と考えて`__init`を付け加えた。プローブ時に圧縮ファームウェアを処理するケースについてはそれは正しいが、関数ポインタは後で使われる可能性に備えて保存されていた。

### 修正 {#fix_1}

ポインタが保存されてブート後に呼ばれる可能性のある関数からは、`__init`を外す:

```c
/* 正しい: __init アノテーションなし。恒久的に .text に留まる */
static int decompress_init_helper(struct request *rq,
                                   const u8 *buf, size_t len)
{
    return decompress_and_store(rq, buf, len);
}
```

あるいは、その関数が本当に初期化中にしか必要ないのであれば、初期化フェーズを越えて残る場所にそのポインタを保存してはならない。ポインタはすぐに使い、その後NULLに設定する — ただしこれは壊れやすい。安全なルールは次のとおり: 関数ポインタが`__data`のライフタイムを持つデータ構造に格納されるなら、指し先の関数は`__init`であってはならない。

### 教訓 {#lesson_1}

`__init`アノテーションは、見かけはだまされるほど単純である。これはコンパイラへの単なるヒントではない — 関数を含む物理ページそのものを変え、そのページはブート後に解放される。`__init`コードへの関数ポインタを保存すれば、それは時限爆弾になる。ページがslabメモリ以外のものに再割り当てされた場合、KASANはこれを捕捉しない。正しい検出戦略は次のとおり: generic KASAN（`CONFIG_KASAN_GENERIC`）を有効にすること — 解放されたinitページがslabメモリとして再利用された場合に不正アクセスを報告する — あるいは、`init_section_contains()`を使ったコンパイル時/ロード時の監査で、永続的なデータに格納された関数ポインタが`.init.text`の範囲を指していないことを検証することである。（`CONFIG_KASAN_VMALLOC`はここでは該当しない — `__init`テキストはvmalloc空間ではなく、ダイレクトマップされたカーネルイメージ内にある。）

---

## 事例3: パニックループ — kdumpがクラッシュを捕捉しない {#case-3-panic-loop-kdump-not-capturing-the-crash}

### 問題 {#the-problem_2}

ある本番サーバーが再起動ループを起こし始めた。システムは起動し、しばらく動作し、パニックし（無関係な理由 — 開発中のバグのあるNICドライバ — による）、解析用のvmcoreを残さずに再起動した。運用チームはまさにこの目的のためにkdumpをセットアップしていた: クラッシュカーネルは設定済みで、`kexec -p`はブート時にsystemdユニットによって実行され、クラッシュカーネル環境には`/proc/vmcore`が現れるはずだった。ところが、システムはただきれいに再起動するだけだった。

vmcoreがなければ、根底にあるNICドライバのバグを診断することはほぼ不可能だった。

### 診断 {#diagnosis_2}

最初の手がかりは、シリアルコンソール経由で取得したクラッシュカーネルの`dmesg`にあった。シリアルコンソールにはクラッシュカーネルのブートシーケンスが記録されていたが、それは唐突に終わっていた:

```
[    0.342819] Booting crash kernel...
[    0.344102] Linux version 5.15.0 (gcc version 11.3.0) #1 SMP
[    0.512433] Command line: root=/dev/sda1 irqpoll maxcpus=1 reset_devices
...
[    2.214891] Out of memory: Kill process 1 (makedumpfile) score 1000
[    2.214903] Killed process 1 (makedumpfile) total-vm:1048576kB, anon-rss:524288kB
[    2.215011] Kernel panic - not syncing: System is deadlocked on memory
```

クラッシュカーネル自体がOOMを起こしていた。`makedumpfile`（`/proc/vmcore`をディスクにコピーするツール）は、元のカーネルのvmcoreを読み取ってフィルタリングする必要があり、それには相当なメモリが必要だったためにkillされていた。しかし、クラッシュカーネルには`crashkernel=128M`で128MBしか割り当てられていなかった。

元のカーネルは256GBのRAMを持っていた。256GBのシステムのvmcoreは — 空きページを除外した後でさえ — 数十ギガバイトになる。`makedumpfile`には、そのワーキングセットとカーネルのページテーブル解析用の構造体をバッファリングできるだけのメモリが必要だった。128MBではまったく足りなかった。

クラッシュカーネルはパニックし、`emergency_restart()`が起動され、システムは何事もなかったかのようにきれいに再起動した。

### 根本原因 {#root-cause_2}

`crashkernel=128M`の予約は、システムのRAMが16GBだったときに設定され、システムが256GBにアップグレードされたときに更新されていなかった。`crashkernel`のサイズの目安はシステムのRAM量に応じてスケールする。その理由は次のとおり:

1. クラッシュカーネルはブートするのに十分なメモリ（カーネル + initramfs + 基本的なインフラ）を必要とする: それだけで典型的には64〜128MB
2. `makedumpfile`は、解析・フィルタリングすべきページ数に比例したワーキングメモリを必要とする（おおよそページあたり1バイトのメタデータ = 4KBページで256GBなら64MB）
3. I/Oとvmcoreのフィルタリング処理のためのバッファ領域

合計: 256GBのシステムでは、`crashkernel=512M`以上が適切である。

### 修正 {#fix_2}

```bash
# /etc/default/grub 内:
GRUB_CMDLINE_LINUX="... crashkernel=512M"

# grub の設定を再生成して再起動する:
update-grub
reboot
```

本番環境で頼りにする前に、kdumpの設定をテストすること:

```bash
# テスト用のパニックを発生させる（kdump をテストする非破壊的な方法）:
echo c > /proc/sysrq-trigger
# （システムはパニックし、vmcore の取得を試みる）
```

`crashkernel=auto`は、メインラインカーネル自身のコマンドラインパーサが理解する値ではない — `kernel/crash_reserve.c`には文字列「auto」に対する特別扱いはなく、具体的なサイズを期待している。一部のディストリビューションが提供する自動サイズ設定は、それぞれのユーザー空間ツール（たとえばdracut/kdumpのインストーラスクリプト）によって行われ、搭載RAMからサイズを計算して具体的な`crashkernel=N`の値をブートローダの設定に書き込む。あるディストリビューションのツールがどのサイズを選ぶにせよ、それは最後にツールが実行された時点での固定的な予約にすぎない — 後でRAMが追加されても自動的には追従しない。これこそがこの事例の失敗モードである。

### 教訓 {#lesson_2}

kdumpは、予約されたメモリから起動される第2のカーネルである。その予約は、クラッシュカーネルが行う作業に見合ったサイズでなければならず、それは固定値ではなく本番カーネルのRAMに応じてスケールする。kdumpの設定はテストしなければならない — 動かないことに気付くのに最悪のタイミングは、実際の本番インシデントの最中である。メンテナンスウィンドウ中に定期的に`echo c > /proc/sysrq-trigger`のテストを実行し、vmcoreファイルが生成されて解析可能であることを確認すること。

---

## 事例4: ブートパラメータのタイプミスによる無症状のSMP設定ミス {#case-4-boot-parameter-typo-causing-silent-smp-misconfiguration}

### 問題 {#the-problem_3}

QEMU仮想マシン群を管理するシステム管理者が、各VMを4 CPUに制限するつもりでカーネルコマンドラインパラメータを追加した。パラメータはVMのGRUB設定に追加された:

```
GRUB_CMDLINE_LINUX="... mxcpus=4"
```

VMはエラーなくブートした。`dmesg`には一見したところ警告はなかった。アプリケーションの性能監視では、VMが想定以上にCPUを使っていることが示された — 4 CPUで動くはずのワークロードが、明らかに利用可能な8 CPUすべてを使っていた。調べてみると、`nproc`は8を返し、`/proc/cpuinfo`には8 CPUすべてが表示されていた。

### 診断 {#diagnosis_3}

正しいパラメータは`mxcpus=4`ではなく`maxcpus=4`である。カーネルは`mxcpus`を認識しないので、`parse_args()`はフォールバックハンドラに到達する。このパラメータはどの`__setup()`ハンドラにも一致せず、（モジュールパラメータを示す）`.`も含まないので、ログに記録されたうえで環境変数としてinitプロセスに渡される:

```bash
# カーネルのブートログ内（数百行に埋もれて見落としやすい）:
dmesg | grep -i 'unknown\|mxcpus'
# Unknown kernel command line parameters "mxcpus=4", will be passed to userspace
```

このメッセージは`KERN_WARNING`ではなく`pr_notice()`（`KERN_NOTICE`、レベル5）でログに記録される — しかし、多くのディストリビューションが静かなブートのために設定するコンソールログレベル4では（アップストリームのKconfigデフォルト`CONFIG_CONSOLE_LOGLEVEL_DEFAULT`は実際には7である）、やはり画面には表示されない: コンソールログレベルより小さいメッセージだけが出力されるからである。メッセージはリングバッファにしか入らない。SMP初期化コードはこのパラメータを一度も目にしなかったので、8 CPUすべてがオンラインになった。

### 根本原因 {#root-cause_3}

未知のパラメータに対するカーネルの方針は、失敗させることではなく、環境変数としてinitプロセスに渡すことである。これは意図的なものである: 一部のパラメータは本当にinitシステム向けのもの（たとえば`systemd.unit=emergency.target`）であり、カーネルは「ユーザーのタイプミス」と「正当なinitパラメータ」を区別できない。カーネルの視点から見れば、この失敗は無症状である。

### 検出 {#detection}

```bash
# ユーザー空間に渡された未知のパラメータを確認する:
dmesg | grep "Unknown kernel command line"

# 再起動前にコマンドラインを検証する:
# （ドキュメント化されたパラメータ一覧と照合する）
grep -w 'mxcpus' /proc/cmdline  # 存在しなければ何も返さない

# 正式なパラメータ一覧:
grep 'maxcpus' Documentation/admin-guide/kernel-parameters.txt
```

一部のディストリビューションは、コマンドラインパラメータを既知の正しい一覧に照らして検証する`bootparamchk`などのツールをinitramfsに含めている。自動化されたデプロイでは、ブート後のヘルスチェックで`nproc`を期待されるCPU数と比較すれば、この種のエラーをただちに捕捉できる。

### 教訓 {#lesson_3}

カーネルは未知のパラメータを黙って受け入れる — 設計上そうなっている。重要なブートパラメータのタイプミスはエラーを生まず、dmesg出力の数百行に埋もれたログメッセージを1つ残すだけである。運用プロセスでは、パラメータが効いたと仮定するのではなく、期待した効果が生じたこと（CPU数、メモリ制限などを確認する）を検証すべきである。フリート管理においては、ブート後の期待される状態を構成管理ツールで定義し、それをアサートすること。

---

## 事例5: 誤ったエラーコードを返すモジュール初期化 {#case-5-module-init-returning-wrong-error-code}

### 問題 {#the-problem_4}

カスタムPCIeデバイス用のドライバがコードレビューを通過し、社内ツリーにマージされた。統合テスト中、ドライバは正常にロードされたように見えた — `modprobe`は終了コード0を返し、`lsmod`にはドライバがロード済みと表示され、デバイスは初期コマンドに応答した。しかしその後の数日間、カーネルログに散発的なメモリリークの報告が現れた:

```
unreferenced object 0xffff9d4b81234500 (size 512):
  comm "modprobe", pid 1234, jiffies 4294987654
  hex dump (first 32 bytes):
  ...
  backtrace:
    kmalloc+0x...
    mydriver_probe+0x60/0x180
```

（デバッグ用カーネルで有効にしていた）`kmemleak`が、解放されることのない割り当てを追跡していた。

### 診断 {#diagnosis_4}

ドライバの`init()`関数は、内部状態を割り当て、キャラクタデバイスを登録し、PCIドライバを登録していた。PCIドライバの登録が失敗した場合（一部のテストマシンにはデバイスが存在しなかったため）、関数はクリーンアップしてエラーを返すはずだった。ところが実際は次のとおりだった:

```c
static int __init mydriver_init(void)
{
    int ret;

    mydriver_state = kmalloc(sizeof(*mydriver_state), GFP_KERNEL);
    if (!mydriver_state)
        return -ENOMEM;

    ret = cdev_add(&mydriver_cdev, mydriver_devno, 1);
    if (ret) {
        kfree(mydriver_state);
        return ret;
    }

    ret = pci_register_driver(&mydriver_pci_driver);
    if (ret) {
        cdev_del(&mydriver_cdev);
        kfree(mydriver_state);
        return 1;  /* バグ: 負の errno であるべき */
    }

    return 0;
}
```

PCI登録失敗時の`return 1`は、負のerrnoではなく*正の*値を返していた。ロード可能モジュールの場合、`init()`は`do_one_initcall(mod->init)`経由で実行されるが、これは単に関数を呼び出してその戻り値をそのまま返すだけであり、そこでは成功/失敗の解釈は行われない。解釈が行われるのは呼び出し元の`do_init_module()`に戻ってからである:

```c
/* kernel/module/main.c */
if (mod->init != NULL)
    ret = do_one_initcall(mod->init);
if (ret < 0) {
    /* ... モジュールのロードは失敗し、巻き戻される ... */
    goto fail_free_freeinit;
}
if (ret > 0)
    pr_warn("%s: init suspiciously returned %d, it should follow 0/-E convention\n",
            mod->name, ret);

/* Now it's a first class citizen! */
mod->state = MODULE_STATE_LIVE;
```

戻り値が`> 0`の場合は警告が出るだけで、ロードを失敗させることは*ない*。モジュールはロード済みとして報告された。しかし`pci_register_driver()`は実際には失敗していたので、`cdev_del()`と`kfree()`のクリーンアップ分岐が実行されていた — キャラクタデバイスの登録と割り当てた状態を解放していたのである。モジュールは、`init()`が「成功」を返したにもかかわらずすべてをクリーンアップしてしまった状態、つまり中途半端に初期化された抜け殻になっていた。

その後のキャラクタデバイス経由（またはドライバフレームワークのコールバック経由）のアクセスは、解放済みのメモリや存在しない登録に到達した。kmemleakの報告は少し異なる経路から来ていた: PCIデバイスは存在するが一時的なエラーを返したマシンでは、`cdev_add`がデバイスを登録したが`pci_register_driver`が失敗した。`cdev_del`は実行されたが、競合するプロセスがすでにキャラクタデバイスをオープンしていたため、`cdev_del`は完全には完了しなかった。状態の割り当て（`mydriver_state`）は、それへの参照がまだ存在している間に解放された。

### 根本原因 {#root-cause_4}

`return 1`は、モジュールinit関数の戻り値として有効なものではない。カーネルの規約は、0 = 成功、負のerrno = 失敗である。正の戻り値は成功として扱われるが、意味を持たない — 有効なerrnoでもなく、成功コードの変種としてドキュメント化されてもいない。この誤りは、実行された中途半端なクリーンアップによってさらに悪化した: 関数はリソースを解放しながら成功を報告し、モジュールをありえない状態に置き去りにした。

### 修正 {#fix_3}

```c
    ret = pci_register_driver(&mydriver_pci_driver);
    if (ret) {
        cdev_del(&mydriver_cdev);
        kfree(mydriver_state);
        return ret;  /* pci_register_driver は失敗時に負の errno を返す */
    }
```

`pci_register_driver()`はすでに失敗時に負のerrnoを返すので、それをそのまま返すのが正しい。独自のエラーコードを生成する場合は負の値を使う: `return -ENODEV;`とし、決して`return 1;`としてはならない。

クリーンアップの順序も割り当ての逆順（LIFO）であるべきで、単一のエラーパスにまとめるべきである:

```c
static int __init mydriver_init(void)
{
    int ret;

    mydriver_state = kmalloc(sizeof(*mydriver_state), GFP_KERNEL);
    if (!mydriver_state)
        return -ENOMEM;

    ret = cdev_add(&mydriver_cdev, mydriver_devno, 1);
    if (ret)
        goto err_free_state;

    ret = pci_register_driver(&mydriver_pci_driver);
    if (ret)
        goto err_del_cdev;

    return 0;

err_del_cdev:
    cdev_del(&mydriver_cdev);
err_free_state:
    kfree(mydriver_state);
    return ret;
}
```

### 教訓 {#lesson_4}

モジュールinit関数は、成功時には0を、失敗時には負のerrnoを返さなければならない。正の戻り値は成功として扱われ、内部状態が不整合であってもモジュールはロード済みとみなされる。カーネルAPI関数からの正の値（デバイスのマイナー番号や個数など）を`init()`からそのまま返してはならない。`goto`ベースのエラーパスのパターン（「エラー処理のためのgoto」と呼ばれることもある）は、クリーンアップの順序を明示的にし、間違えにくくする。`kmemleak`やKASANのようなツールは、開発やCIで結果として生じる不整合を捕捉するうえで非常に貴重である。

---

## 参考文献 {#further-reading}

- [初期ブートとstart_kernel()](early-boot.md) — initcallレベルと順序
- [モジュール初期化とinitcall](initcalls-modules.md) — `load_module()`、エラーコードの扱い
- [カーネルパニックとOops](panic-oops.md) — クラッシュ解析とkdumpのセットアップ
- [ブートパラメータ](boot-params.md) — `__setup()`、`early_param()`、未知のパラメータの扱い
- [Kmemleak](../mm/kmemleak.md) — 事例5のようなメモリリークの検出
- [KASAN](../mm/kasan.md) — 事例2の背後にある検出戦略である`CONFIG_KASAN_GENERIC`、`CONFIG_KASAN_SW_TAGS`、`CONFIG_KASAN_HW_TAGS`（`CONFIG_KASAN_VMALLOC`の区別については、特に下の外部参考資料を参照）
- [include/linux/init.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/init.h) — initcallレベルのマクロ: `core_initcall()` = レベル1、`subsys_initcall()` = レベル4、`device_initcall()` = レベル6。事例1の背後にある順序を裏付ける
- [init/main.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/init/main.c) — `do_one_initcall()`とレベルごとのinitcallディスパッチ（`do_initcall_level()`）、および事例4の背後にある`unknown_bootoption()`/`print_unknown_bootoptions()`
- [kernel/module/main.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/main.c) — `do_init_module()`: `if (ret > 0) pr_warn("%s: init suspiciously returned %d, it should follow 0/-E convention\n", ...)`。事例5のバグをロード時に捕捉できたはずの実際のカーネルコード

## 外部参考資料 {#external-references}

- [Kernel documentation: Kdump Support](https://docs.kernel.org/admin-guide/kdump/kdump.html) — `crashkernel=`のサイズ指定構文とガイダンス（例: `crashkernel=512M-2G:64M,2G-:128M`）。事例3の背後にある、RAMに応じてスケールするというルール
- [Kernel documentation: Kernel Parameters](https://docs.kernel.org/admin-guide/kernel-parameters.html) — 「if it doesn't recognize a parameter and it doesn't contain a '.', the parameter gets passed to init: parameters with '=' go into init's environment」（パラメータを認識できず、'.'も含まない場合、そのパラメータはinitに渡される: '='を含むパラメータはinitの環境に入る）— 事例4の無症状な`mxcpus=4`タイプミスの背後にあるまさにその機構
- [Kernel documentation: Linux kernel coding style, §16 "Function return values and names"](https://docs.kernel.org/process/coding-style.html) — 事例5の`return 1;`が違反した、0が成功・負のerrnoが失敗という規約
- [Kernel documentation: KASAN](https://docs.kernel.org/dev-tools/kasan.html) — デフォルトでは、実際のシャドウメモリを持つのはリニアマッピングだけであり、vmalloc空間には実際の検出能力を持たない読み取り専用のプレースホルダのシャドウページが割り当てられる。vmallocの割り当てを個別にシャドウで追跡できるようにするのが`CONFIG_KASAN_VMALLOC`である — これは、ダイレクトマップされた`__init`テキストページにはそもそも`CONFIG_KASAN_VMALLOC`が当てはまらないという事例2の主張を裏付ける。そのページは最初からvmalloc空間にはなかったからである
- [Kernel documentation: Kmemleak](https://docs.kernel.org/dev-tools/kmemleak.html) — 「detecting possible kernel memory leaks ... the orphan objects are not freed but only reported」（カーネルのメモリリークの可能性を検出する……孤立したオブジェクトは解放されず、報告されるだけである）。事例5のバグを表面化させるツール

このページの冒頭で述べたとおり、上の本文には特定のコミットハッシュ、CVE番号、カーネルバージョンに結び付いた機能の主張は登場しない — 上の参考資料は代わりに、根底にあるカーネルの機構（initcallの順序、`__init`/KASAN、kdump/crashkernelのサイズ設定、未知のブートパラメータの扱い、モジュールinitの戻り値）を、現在のメインラインのソースとドキュメントに照らして検証している。
