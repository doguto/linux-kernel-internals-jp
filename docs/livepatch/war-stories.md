# ライブパッチの事件簿 {#live-patching-war-stories}

> 実際のインシデント、厄介なエッジケース、そして得られた教訓

## 1. 停滞した遷移 {#1-the-stuck-transition}

### シナリオ {#scenario}

kthreadが使うネットワーク処理関数のバグを修正するために、本番システムに
ライブパッチが適用された。パッチは正常にロードされ、
`enabled`は`1`を返したが、`transition`は何時間も`1`のままだった。

```bash
cat /sys/kernel/livepatch/net-fix/transition
# 1   （4時間経ってもまだ停滞）
```

### 根本原因 {#root-cause}

問題のkthreadはタイトなポーリングループを実行していた。

```c
/* 簡略化: バグのあるkthread */
static int net_kthread(void *arg)
{
    while (!kthread_should_stop()) {
        process_pending_work();   /* 古い（バグのある）関数を呼ぶ */
        /* スリープもcond_resched()もない — 決してスケジュールアウトしない */
    }
    return 0;
}
```

このkthreadは自発的にCPUを明け渡すことも、スリープに入ることもなかったため、
`klp_sched_try_switch()`がそれを試せるコンテキストスイッチに一度も到達しなかった。
このスケジューラ経路のチェックは、切り替えで出ていくタスクがfreezableなスリープ状態に
入ろうとしているときにのみ実行されるが、それこそこのkthreadが決してしなかったことである。`klp_send_signals()`は
kthreadに対して`wake_up_state(task, TASK_INTERRUPTIBLE)`を呼ぶが、これは割り込み可能な
スリープ中のタスクしか起こせず、タイトなループで実際に実行中のタスクには効かない。
`cond_resched()`のないkthreadのタイトなループが、スケジューラ経路のチェックと
`TASK_INTERRUPTIBLE`の起床の両方を無効にしていた。

kthreadのスタックがそれを裏付けていた。

```bash
cat /proc/<kthread_pid>/stack
# [<0>] process_pending_work+0x3a/0x120  ← 古い関数がまだスタック上にある
# [<0>] net_kthread+0x18/0x50
# [<0>] kthread+0xd6/0x100
# [<0>] ret_from_fork+0x22/0x30
```

### 解決 {#resolution}

選択肢は2つあった。

1. **遷移を強制する**（応急処置として選択）: この特定のケースでは、実行途中であっても
   新しい関数を呼んで安全であることを確認したうえで、forceファイルに
   書き込んだ。

   ```bash
   echo 1 > /sys/kernel/livepatch/net-fix/force
   ```

   カーネルはモジュールをロードした時点で既に`TAINT_LIVEPATCH`でtaintされていた。
   強制すること自体が追加のtaintを付けることはない。強制そのものの
   危険性は応急処置として受け入れられ、システムは24時間以内の
   計画的なリブートの対象としてマークされた。

2. **kthreadをスリープさせる**（恒久的な修正）: 後続のパッチで、ループが空回りする
   代わりに反復の合間にスリープするようにした。これにより遷移には2つの
   入口ができる。定期的なworkqueueの走査が実行されるときにkthreadはCPUから外れている
   （そのため`klp_check_and_switch_task()`はもう`-EBUSY`を返さない）し、待っている間は
   スタックにパッチ対象の関数がもう載っていない。

   現在のカーネルでは、そのスリープは本物のスリープでなければならない。単なる
   `cond_resched()`では不十分である。コミット
   [`676e8cf70cb0`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=676e8cf70cb0)
   （"sched,livepatch: Untangle cond_resched() and live-patching"、2025-05-09）以降、
   livepatchのフックは`cond_resched()`ではなく`__schedule()`内の`TASK_FREEZABLE`スリープに
   ぶら下がっているので、譲歩するだけのループは依然として
   `__klp_sched_try_switch()`に到達しない。

### 教訓 {#lesson}

一度もスリープせずにループするkthreadは、ライブパッチにとって最も手強い対象である。
ユーザー空間へ戻ることはなく、スケジューラ経路のチェックが鍵とするfreezableなスリープに
入ることもなく、workqueueの走査が試すときには常にCPU上にいる。
そのような関数をカバーするパッチを書く前に:

- kthreadのソースを管理しているなら（別の準備用パッチを通じて）、本物のスリープを与えること。
  短い`msleep()`、より良いのはfreezableな待機である。`cond_resched()`で
  譲歩するだけでは、もはや遷移の助けにならない。

  注: `struct klp_func`の`nop`フィールドは、累積的な置き換えの
  仕組み（`klp_add_nops()`）が、古いパッチがカバーしているが新しい累積パッチが
  明示的にパッチしていない関数について、元の関数へそのまま呼び出しを通す
  プレースホルダーのエントリを作るために使うものである。このシナリオでスタックを
  空にするための道具ではない。

## 2. shadow変数のライフサイクルのバグ {#2-shadow-variable-lifecycle-bug}

### シナリオ {#scenario_1}

あるライブパッチは、新しいセキュリティ属性を追跡するためにソケットごとの状態を追加した。
パッチは`connect()`の際にshadow変数を割り当て、`send()`の際にそれを読んだ。
パッチはデプロイされて正しく動作した。しかしその後の1週間で、影響を受けたホストの
メモリ使用量はじわじわと増え続け、決して安定しなかった。

```bash
# 回収不能なslabが増え続け、決して減らない
grep -E '^(Slab|SReclaimable|SUnreclaim):' /proc/meminfo
# Slab:            1842160 kB
# SReclaimable:     732876 kB
# SUnreclaim:      1109284 kB   ← 1日約40 MB、接続の入れ替わりに比例
```

### 根本原因 {#root-cause_1}

shadow変数は、（オブジェクトポインタ, ID）をキーとするグローバルなハッシュテーブル
`klp_shadow_hash`に格納される。パッチは新しいソケットごとにshadowを割り当てたが、
ソケットがクローズされたときにそれを解放していなかった。

```c
/* パッチのコード — バグ */
static int patched_tcp_connect(struct sock *sk, struct sockaddr *uaddr,
                                int addr_len)
{
    struct my_shadow *s;

    s = klp_shadow_get_or_alloc(sk, KLP_MY_SHADOW_ID,
                                 sizeof(*s), GFP_KERNEL,
                                 NULL, NULL);
    s->attr = compute_security_attr(uaddr);

    /* 元のロジックを呼ぶ ... */
}

/* 欠落: ソケット解放経路でのklp_shadow_free() */
```

クローズされたソケットのそれぞれが、ハッシュテーブル内に孤立した`struct klp_shadow`を残した。
ハッシュテーブル自体は`kernel/livepatch/shadow.c`内のファイルスコープの
`DEFINE_HASHTABLE(klp_shadow_hash, 12)`であり、それを縮めるものは何もない。出口は2つだけで、
`klp_shadow_free()`（1つの`<obj, id>`の組）と`klp_shadow_free_all()`
（指定した`id`を持つすべてのshadow）である。どちらもパッチが呼ばなければならない。

### 検出 {#detection}

ここが厄介な部分である。grepで探せる`klp_shadow`のslabキャッシュは存在しない。
`kernel/livepatch/shadow.c`には`kmem_cache_create()`が一切含まれておらず、すべての
shadow変数は単なる`kmalloc`から割り当てられる。

```c
/* kernel/livepatch/shadow.c — __klp_shadow_get_or_alloc() */
new_shadow = kzalloc(size + sizeof(*new_shadow), gfp_flags);
```

そのため割り当ては、同じサイズの他のすべてのものと共有される汎用の`kmalloc-*`バケットに
入り、`/proc/slabinfo`にはlivepatch固有の行がない。`struct klp_shadow`は64ビットでは
48バイトのヘッダ（`hlist_node`、`rcu_head`、`obj`、`id`）に、パッチ自身のペイロード
（ここでは32ビットの属性1つ）を加えた合計52バイトなので、これらのshadowは`kmalloc-64`から
割り当てられていた。そのバケットを監視することは裏付けの証拠にはなるが、決して証明にはならない。

```bash
# 汎用の共有バケット — ヒントであって、帰属の特定ではない
watch -n 60 "grep -E '^kmalloc-64 ' /proc/slabinfo"
```

kmemleakも役に立たない。孤立したオブジェクトはまだ
`klp_shadow_hash`にリンクされており、`struct klp_shadow`の`hlist_node`は最初のメンバなので、
すべての割り当ての先頭を指すポインタが生きたカーネルデータ構造の中に存在している。
kmemleakはそのようなポインタが見つからないオブジェクトしか報告しないので、
これらに印を付けることは決してない。これはライフサイクルのリークであり、
参照されていないメモリのリークではない。

実際に事件を解決したのは2段階のプロセスだった。まず相関を取る。
`/proc/meminfo`の`SUnreclaim:`をホストのソケットのオープン/クローズ率に対してプロットし、
その傾きがパッチのロード後にのみ、そしてパッチを持つホストでのみ現れたことを確認した。
次に、ハッシュテーブルを直接たどって確認した。
`klp_shadow_hash`は`static`で、procfs、sysfs、debugfsのインターフェースもないので、
これにはデバッグ情報付きのカーネルと、カーネルのデータ構造を読めるツールが必要である。
稼働中のカーネルに対しては`drgn`、ダンプに対しては`crash`を使う。

```python
# drgn、デバッグ情報付きでビルドされたカーネル上で
from collections import Counter
from drgn.helpers.linux.list import hlist_for_each_entry

counts = Counter()
for bucket in prog["klp_shadow_hash"]:
    for s in hlist_for_each_entry("struct klp_shadow",
                                  bucket.address_of_(), "node"):
        counts[hex(int(s.id))] += 1
print(counts)
# Counter({'0x1': 41823})   ← ホスト上の生きたソケットは約1200個だった
```

### 解決 {#resolution_1}

後続のパッチでソケット解放経路をフックし、shadowを解放した。

```c
/* 修正パッチ — ソケット解放に追加 */
static void patched_sock_release(struct socket *sock)
{
    /* patched_tcp_connectで付加したshadow変数を解放する */
    klp_shadow_free(sock->sk, KLP_MY_SHADOW_ID, NULL);

    /*
     * ... net/socket.cからそのままコピーしたsock_release()の本体の残り。
     * ここでは省略する。KLPにはklp_call_orig()はない（klp.mdを参照）
     * ので、パッチモジュールは本体全体を持つ。
     */
}
```

### 教訓 {#lesson_1}

すべての`klp_shadow_get_or_alloc()`には、対応する
`klp_shadow_free()`がなければならない。パッチを書く前に、カーネルオブジェクトの
ライフサイクル全体を洗い出すこと。

- **割り当て**: `connect()` — `klp_shadow_get_or_alloc()`を呼ぶ必要がある
- **使用**: `send()`、`recv()` — `klp_shadow_get()`を呼ぶ必要がある
- **解放**: `sock_release()`、`tcp_close()` — `klp_shadow_free()`を呼ぶ必要がある

オブジェクトのデストラクタがshadowを作成する場所とは別のモジュールにある場合、
作成側とデストラクタ側の両方の経路を一緒にパッチしなければならない。
shadow変数のAPIについては[カーネルライブパッチ](klp.md)を参照。

## 3. compatシステムコールの見落とし {#3-the-compat-syscall-miss}

### シナリオ {#scenario_2}

レガシーなXFS互換の領域予約ioctl、すなわち
`FS_IOC_RESVSP`、`FS_IOC_UNRESVSP`、`FS_IOC_ZERO_RANGE`にCVEが発見された。ライブパッチが
2時間以内にすべての本番ホストへデプロイされ、`fs/ioctl.c`の`ioctl_preallocate()`を
修正した。脆弱性は修復されたとみなされた。3日後、
セキュリティ監査で、x86_64ホスト上の32ビットプロセスがまったく同じioctlコマンドを通じて
依然として悪用可能であることが判明した。

### 根本原因 {#root-cause_2}

パッチはネイティブのioctl経路だけを対象にしていた。`CONFIG_X86_64`では、32ビットの
compatエントリポイントである`compat_sys_ioctl()`は、同等の`_32`接尾辞付きのコマンドコード
（`FS_IOC_RESVSP_32`、`FS_IOC_UNRESVSP_32`、
`FS_IOC_ZERO_RANGE_32`）を、別の関数である`compat_ioctl_preallocate()`へ直接
ディスパッチしていた。パッチはこの関数に一切触れていなかった。

```c
/* ネイティブ経路 — パッチ適用済み */
static int ioctl_preallocate(struct file *filp, int mode, void __user *argp)
{
    struct space_resv sr;

    if (copy_from_user(&sr, argp, sizeof(sr)))
        return -EFAULT;
    ...
    return vfs_fallocate(filp, mode | FALLOC_FL_KEEP_SIZE,
                          sr.l_start, sr.l_len);   /* ← パッチ適用済み */
}

/* x86_64上の32ビットcompat経路 — パッチ未適用 */
#if defined CONFIG_COMPAT && defined(CONFIG_X86_64)
/* ia32ではl_startは32ビット境界にある。アライメントの違いを
 * 考慮するだけでよい */
static int compat_ioctl_preallocate(struct file *file, int mode,
                                    struct space_resv_32 __user *argp)
{
    struct space_resv_32 sr;   /* 別個の、packedな32ビットレイアウト */

    if (copy_from_user(&sr, argp, sizeof(sr)))
        return -EFAULT;
    ...
    return vfs_fallocate(file, mode | FALLOC_FL_KEEP_SIZE,
                          sr.l_start, sr.l_len);   /* ← 同じ呼び出しだが... */
}
#endif
```

`compat_ioctl_preallocate()`が存在するのは、`struct space_resv`の64ビットの
`l_start`/`l_len`フィールドが8バイト境界にアラインされているのに対し、ia32では4バイト境界にしか
アラインされていないからである。そのためcompat経路は独自のpackedな`struct space_resv_32`と
独自のコピーインのルーチンを定義している（`include/linux/falloc.h`）。
`COMPAT_SYSCALL_DEFINE3(ioctl...)`の中で、`_32`のコマンドコードは
`compat_ioctl_preallocate()`によって直接処理され、パッチがフックした関数である
`do_vfs_ioctl()` → `file_ioctl()` → `ioctl_preallocate()`には決して到達しない。

### 検出 {#detection_1}

```bash
# /proc/kallsymsでcompat版の対応物を確認する
grep -i preallocate /proc/kallsyms
# ffffffff812a1050 t ioctl_preallocate
# ffffffff812a1120 t compat_ioctl_preallocate   ← 別のシンボル、パッチ未適用

# ソースでcompatのディスパッチを確認する
grep -n "compat_ioctl_preallocate\|FS_IOC_RESVSP_32" fs/ioctl.c
```

### 解決 {#resolution_2}

`compat_ioctl_preallocate()`もフックする2つ目のパッチがデプロイされた。
以後、チームはioctlハンドラのパッチごとにチェックリスト項目を追加した。

1. そのioctlコマンドに`_32`のcompat版はあるか。そしてそれ用に、`space_resv_32`のような
   `CONFIG_X86_64`でガードされた構造体は存在するか。
2. compat経路（ドライバの別個の`f_op->compat_ioctl`、
   あるいはこの例のように`fs/ioctl.c`の`compat_sys_ioctl()`内の分岐）は、
   ネイティブのハンドラへ転送するのではなく、本当に別の関数を
   呼んでいるか。
3. 脆弱性はネイティブとcompatの両方の解析ルーチンに
   存在するか。

### 教訓 {#lesson_2}

`do_vfs_ioctl()`または`f_op->unlocked_ioctl`を通じて到達する関数を対象とするパッチでは、
同じコマンド番号に、異なるコードを実行するcompat版の対応物があるかどうかを確認すること。
すべての`compat_ioctl`が分岐するわけではない。
`compat_ptr_ioctl()`のような汎用的なものは、ポインタ変換の後にネイティブのハンドラへ
転送するだけである。しかし、この例の`space_resv`と`space_resv_32`のように、
本当に異なるcompat引数レイアウトを持つハンドラには、それ専用の
livepatchの対象が必要である。関数名の系列について`grep -i <name> /proc/kallsyms`するのは
素早い最初のチェックであり、決定的な答えはソースにある。

## 4. 重複シンボルの間違った出現 {#4-the-wrong-occurrence-of-a-duplicated-symbol}

### シナリオ {#scenario_3}

あるバグが、組み込みネットワークドライバ内の小さな`static`ヘルパーである`rx_ring_refill()`に
たどり着いた。それに対するライブパッチが書かれた。最初の`insmod`は
あっさり失敗した。

```bash
insmod rx-fix.ko
# insmod: ERROR: could not insert module rx-fix.ko: Invalid parameters

dmesg | tail -1
# livepatch: unresolvable ambiguity for symbol 'rx_ring_refill' in object '(null)'
```

このメッセージは、`klp_find_object_symbol()`が推測を拒否したものである。`old_sympos`は
デフォルトの`0`のままで、これは「このシンボルは一意でなければならない」を意味する。そして
2つ目の組み込みドライバが独自の`static rx_ring_refill()`を定義していた。（オブジェクトが
`(null)`と表示されるのは、`vmlinux`を対象とする場合、`klp_find_object_symbol()`には
NULLの`objname`が渡されるからである。）

エンジニアは曖昧さを解消するため`.old_sympos = 1`を設定し、リビルドして再ロードした。今度は
すべてが問題なく立ち上がった。

```bash
insmod rx-fix.ko
# エラーなし

cat /sys/kernel/livepatch/rx-fix/enabled      # 1
cat /sys/kernel/livepatch/rx-fix/transition   # 0 — 完了
cat /sys/kernel/livepatch/rx-fix/vmlinux/patched
# 1  ← フックは確かにインストールされている
```

それでも元のバグは再現し続けた。さらに悪いことに、同じホスト上の無関係な無線
インターフェースが負荷時にフレームを落とし始めた。

### 根本原因 {#root-cause_3}

パッチは有効で、遷移は完了しており、ftraceフックはインストールされていた。
ただし間違った関数に。`old_sympos`はもう一方のドライバの
`rx_ring_refill()`を選択していた。

出現の番号付けは恣意的なものではなく、`nm`が表示する順序でもない。
`scripts/kallsyms.c`は名前のインデックスを`compare_names()`でソートし、これは
同じ名前同士のタイを**アドレスの昇順**で決める。

```c
/* scripts/kallsyms.c — compare_names() */
ret = strcmp(sym_name(sa), sym_name(sb));
if (!ret) {
    if (sa->addr > sb->addr)
        return 1;
    else if (sa->addr < sb->addr)
        return -1;
    ...
}
```

`vmlinux`を対象とする場合、`klp_find_object_symbol()`は
`kallsyms_on_each_match_symbol()`経由でそのインデックスをたどり、累計が
`sympos`に達したところで止まる。したがって*sympos Nはアドレスの昇順でN番目の出現*である。（モジュールを
対象とする場合はもう一方の分岐である`module_kallsyms_on_each_symbol()`を通る。これは
そのモジュールのELFシンボルテーブルをインデックス順にたどり、`SHN_UNDEF`の
エントリを飛ばす。これは別の順序であり、仮定せずに検証すべきものである。）
エンジニアは位置を`nm vmlinux | grep rx_ring_refill`から導いていたが、
`nm`は`-n`を指定しない限り名前のアルファベット順にソートする。同じ名前の2つの
シンボルについては`strcmp()`が0を返し、タイの順序は`qsort()`が残した
ままになり、ここではそれがアドレス順と一致しなかった。位置1は無線
ドライバのコピーだった。

カーネルはこれに何の異議も唱えない。範囲内の`sympos`は*何らかの*実在する関数に
解決され、パッチ適用は普通に進む。拒否されるのは範囲外の値だけであり、
その場合は別のメッセージが出る。

```c
/* kernel/livepatch/core.c — klp_find_object_symbol() */
} else if (sympos != args.count && sympos > 0) {
        pr_err("symbol position %lu for symbol '%s' in object '%s' not found\n",
               sympos, name, objname ? objname : "vmlinux");
}
```

つまり、大きな音を立てて失敗する曖昧さは安全な方である。間違っているが有効な
`old_sympos`こそが、音もなく失敗する方である。

### 検出 {#detection_2}

`dmesg`はきれいだった。それ自体が手がかりだった。これは解決の失敗ではないので、
問題は*何に*解決されたかだった。`old_func`は意図的にsysfsでは公開されていないが、
実際に使われた位置は関数ごとのディレクトリ名に現れる。ディレクトリは
`old_sympos ? old_sympos : 1`を使って`"%s,%lu"`という名前が付けられるので、
`.old_sympos`が`0`の場合も`,1`と表示される
（構造体のフィールド自体は`0`のままで、ディレクトリ名だけが実際の位置に置き換えられる）。

```bash
ls /sys/kernel/livepatch/rx-fix/vmlinux/
# patched  rx_ring_refill,1     ← "<function>,<sympos>"
```

そこから、対応関係をオフラインで再構築した。`/proc/kallsyms`はアドレス順に出力されるので、
その一覧が*そのまま*カーネルの番号付けである。

```bash
grep ' rx_ring_refill$' /proc/kallsyms
# ffffffff81a4c210 t rx_ring_refill    ← sympos 1
# ffffffff81b0e8c0 t rx_ring_refill    ← sympos 2

# 各アドレスをコンパイル単位に対応づける
addr2line -f -e /usr/lib/debug/lib/modules/$(uname -r)/vmlinux \
    0xffffffff81a4c210 0xffffffff81b0e8c0
# rx_ring_refill
# drivers/net/wireless/.../rx.c:412        ← パッチが当たったもの
# rx_ring_refill
# drivers/net/ethernet/.../rx.c:88         ← 意図していたもの
```

### 解決 {#resolution_3}

パッチは`.old_sympos = 2`でリビルドされ、再デプロイされた。挙動を待つ前に、
sysfsのディレクトリ名が新しい対象を確認した。

```bash
ls /sys/kernel/livepatch/rx-fix/vmlinux/
# patched  rx_ring_refill,2
```

その後チームはビルド時のチェックを追加した。各`klp_func`について、
`old_name`をアドレス順にソートしたシンボル一覧に対して解決し、設定された`old_sympos`の
出現が`addr2line`経由でパッチを書いた対象のソースファイルに対応しない限り、
ビルドを失敗させるというものである。

### 教訓 {#lesson_3}

`old_sympos`はオブジェクトごとに数える1始まりのインデックスであり（`vmlinux`では
アドレスの昇順）、これを間違えることは、カーネルが代わりに捕まえてくれない
数少ないlivepatchのミスの1つである。

- `vmlinux`を対象とする場合は、**アドレス順**の一覧から導くこと。
  `/proc/kallsyms`は既にその順序であり、`nm -n vmlinux`もその順序にできる。
  単なる`nm`は名前のアルファベット順にソートし、位置については何も教えてくれない。
- パッチが有効になったことだけでなく、実際に何に解決されたかを常に確認すること。
  `<function>,<sympos>`のディレクトリ名と、対応する`/proc/kallsyms`のエントリの
  `addr2line`がチェックのすべてである。
- `old_sympos`を`0`のままにすることが安全なデフォルトであるのは、まさに名前が一意でないときに
  ロードを失敗させるからである。そのエラーを黙らせるために位置を設定してはならない。
  どの出現が欲しいのかを確認したから設定するのである。

これらすべての前に、その関数がそもそもシンボルとして存在することを確認すること。

```bash
grep " <function_name>$" /proc/kallsyms
```

存在しない場合（ヘッダからインライン展開された、マクロである、あるいは稼働中の設定で
コンパイルから除外されている）、`klp_find_object_symbol()`は`symbol '%s' not found in
symbol table`を出力して`-EINVAL`を返し、これは
`klp_init_object_loaded()`と`klp_init_patch()`を通じて`klp_enable_patch()`まで伝播するので、
`insmod`は失敗し、モジュールはロードされない。後で発見されるような部分的に適用された
状態は存在しない。単に呼び出し元の方にパッチを当てるしかない。

## 5. `.replace`を忘れた累積パッチ {#5-the-cumulative-patch-that-forgot-replace}

### シナリオ {#scenario_4}

あるフリートには3つのライブパッチが蓄積していた。`tcp_sendmsg`を修正するP1、
`ip_output`を修正するP2、`nf_conntrack_in`を修正するP3である。P2の修正は後に
スループットのリグレッションの原因と特定され、取り下げる予定になった。

計画は、P1とP3の修正に加えて`udp_sendmsg`の新しいセキュリティ修正を持ち、
P2の変更は*持たない*単一の累積パッチP4を作ることだった。そうすれば
P4の適用後、
`ip_output`は再び元のアップストリームのコードを実行するはずだった。P4はビルドされ、展開され、
すべてが正常と報告された。

```bash
insmod p4-cumulative.ko
cat /sys/kernel/livepatch/p4-cumulative/enabled      # 1
cat /sys/kernel/livepatch/p4-cumulative/transition   # 0 — 完了
```

1週間分のスループットのグラフを見た後も、パッチを当てたすべてのホストでリグレッションは
残っていた。

### 根本原因 {#root-cause_4}

P4の`struct klp_patch`で、`.replace`はデフォルトの`false`のままになっていた。
それを設定する行はパッチモジュールのソーステンプレートのリファクタリングで落ちており、
ビルド時にもロード時にも何も文句を言わなかった。非累積の
パッチは完全に正当なパッチだからである。

`.replace`がなければ、置き換えの仕組みは一切動かない。
`klp_init_patch()`では、nopの生成がガードされている。

```c
/* kernel/livepatch/core.c — klp_init_patch() */
if (patch->replace) {
        ret = klp_add_nops(patch);
        if (ret)
                return ret;
}
```

そして`klp_complete_transition()`では、置き換えられたパッチの後始末が
同じようにガードされている。

```c
/* kernel/livepatch/transition.c — klp_complete_transition() */
if (klp_transition_patch->replace && klp_target_state == KLP_TRANSITION_PATCHED) {
        klp_unpatch_replaced_patches(klp_transition_patch);
        klp_discard_nops(klp_transition_patch);
}
```

そのためP4は、通常のパッチがすることだけを行った。名指しした各関数について、
自身の`klp_func`をその関数の`ops->func_stack`の先頭にプッシュし
（`klp_patch_func()`内の`list_add_rcu()`）、そこで止まった。P1、P2、P3は
有効なままだった。`tcp_sendmsg`と`nf_conntrack_in`は2段のスタックになり、
その先頭はたまたまP4の同等の実装だったので、何もおかしく見えなかった。しかし
`ip_output`はP4によってまったく名指しされていなかった（P4は意図的にそれを外していた）ので、
それについては何もプッシュされず、その`func_stack`の先頭はP2の`klp_func`のままだった。
リグレッションが元に戻ることはありえなかった。誰もそれを求めていなかったからである。

古いパッチが触れた関数を元に戻すことは、まさに`.replace`がもたらすものであり、
`.replace`だけがもたらすものである。アトミックリプレースが可能にすることについての
アップストリーム自身の一覧も、それを筆頭に挙げている。「Atomically revert some functions in a previous patch
while upgrading other functions.」（他の関数を更新しつつ、以前のパッチの一部の関数をアトミックに元に戻す）

### 検出 {#detection_3}

決定的な兆候は、置き換えられたはずのパッチがすべてまだ存在していたことである。本当の
アトミックリプレースの後であれば、それらは存在しないはずである。遷移が落ち着くと、
`klp_try_complete_transition()`は`.replace`パッチについて`klp_free_replaced_patches_async()`を
呼び、これが新しいパッチより前にあるすべてのパッチを
`klp_free_patch_start()`（`klp_patches`からの`list_del()`）と
`klp_free_patch_finish()`（`kobject_put()`）に通すので、それらのsysfsディレクトリは
完全に消える。

```bash
# 4つのディレクトリ、4つの有効なパッチ — 置き換えは起きていない
grep . /sys/kernel/livepatch/*/enabled
# /sys/kernel/livepatch/p1/enabled:1
# /sys/kernel/livepatch/p2/enabled:1
# /sys/kernel/livepatch/p3/enabled:1
# /sys/kernel/livepatch/p4-cumulative/enabled:1

# フラグはパッチごとに読み取り専用で公開されている
cat /sys/kernel/livepatch/p4-cumulative/replace
# 0   ← 一度も累積パッチではなかった

# stack_orderはklp_patches内でのパッチの1始まりの位置。アトミックリプレースが
# 成功すれば、新しいパッチだけが1の位置に残る
cat /sys/kernel/livepatch/p4-cumulative/stack_order
# 4   ← 3つのパッチを置き換えるのではなく、その上に積み重なっている

# そしてip_outputはまだp2のものである
ls /sys/kernel/livepatch/p2/vmlinux/
# ip_output,1  patched
ls /sys/kernel/livepatch/p4-cumulative/vmlinux/
# nf_conntrack_in,1  patched  tcp_sendmsg,1  udp_sendmsg,1
```

### 解決 {#resolution_4}

P4は`.replace = true`以外は変更せずにリビルドされ、P5として再ロードされた。
今度は`klp_add_nops()`が実行された。P1からP4がカバーするすべての関数をたどり、
`ip_output`には新しいパッチ内に対応物がないことを見つけ、それ用に動的な
`nop`の`klp_func`を割り当てた。ftraceハンドラが意図的に何もしないエントリであり、
そのため元の`ip_output`が実行される。遷移が終わると、
4つの古いパッチはパッチを外され、`klp_patches`から取り除かれ、それらの
モジュールは削除可能になった。

```bash
insmod p5-cumulative.ko
cat /sys/kernel/livepatch/p5-cumulative/transition   # 0になるのを待つ

cat /sys/kernel/livepatch/p5-cumulative/replace      # 1
cat /sys/kernel/livepatch/p5-cumulative/stack_order  # 1
ls /sys/kernel/livepatch/
# p5-cumulative        ← 古いディレクトリは消えている
```

### 教訓 {#lesson_4}

パッチが累積パッチなのは、あなたがそう宣言したからにすぎない。パッチの内容からそれを
推論するものは何もなく、`.replace`が欠けていてもどの段階でも警告は出ない。

- `Documentation/livepatch/cumulative-patches.rst`のアップストリームの助言に従うこと。
  「A good practice is to set .replace flag in any released livepatch.」（リリースするすべての
  livepatchで.replaceフラグを設定するのが良い習慣である）。すべてを置き換えるパッチは、
  置き換えるものが何もない場合でも正しい。
- デプロイの合否は、`enabled`と`transition`ではなく、観察可能な結果で判断すること。
  遷移が落ち着いた後、`replace`は`1`、`stack_order`は`1`を返すべきであり、
  `/sys/kernel/livepatch/`にはちょうど1つのディレクトリだけがあるべきである。
- 「replace」がしないことを覚えておくこと。実行されるのは新しい累積パッチの
  (un)patchingコールバックだけであり、置き換えられたパッチのコールバックは
  飛ばされる。置き換え対象のパッチがコールバックで行っていたことは何であれ、
  累積パッチが自分で行わなければならない。

`func_stack`、`nop` func、アトミックリプレースの流れについての完全な説明は
[累積パッチとアトミックリプレース](klp-cumulative.md)を参照。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/livepatch/transition.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/livepatch/transition.c) — 割り込み可能なスリープ中のkthreadしか起こさない`wake_up_state(task, TASK_INTERRUPTIBLE)`を使う`klp_send_signals()`と、`klp_try_complete_transition()`。事例1の背景
- [kernel/livepatch/shadow.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/livepatch/shadow.c) — `klp_shadow_hash`ハッシュテーブルと、事例2のパッチが間違えた`klp_shadow_get_or_alloc()`/`klp_shadow_free()`の対応関係
- [kernel/livepatch/core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/livepatch/core.c) — `old_sympos`の数え方と"unresolvable ambiguity"/"symbol position ... not found"エラーが事例4を引き起こす`klp_find_object_symbol()`、および事例5の背景にある`klp_add_nops()`周りの`if (patch->replace)`ガード
- [kernel/livepatch/patch.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/livepatch/patch.c) — `ops->func_stack`への`klp_patch_func()`の`list_add_rcu()`（事例5のパッチが置き換えの代わりに得た単純な積み重ね）と、`klp_ftrace_handler()`の`if (func->nop)`による早期脱出
- [scripts/kallsyms.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/kallsyms.c) — 同名シンボル間のタイをアドレスの昇順で決め、それによって事例4で`old_sympos`が何を数えるかを定義する`compare_names()`
- [include/linux/livepatch.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/livepatch.h) — `struct klp_func`の`nop`と`old_sympos`フィールド。`old_sympos`が0なら一意なシンボルが必要であることを述べるコメントを含む。事例1と4の背景
- [Documentation/livepatch/cumulative-patches.rst](https://docs.kernel.org/livepatch/cumulative-patches.html) — アップストリームのアトミックリプレースのガイド: 事例5の鍵となる「atomically revert some functions」機能と「set .replace flag in any released livepatch」という助言
- [fs/ioctl.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/ioctl.c) — `ioctl_preallocate()`と、x86_64の別個のエントリポイントである`compat_ioctl_preallocate()`。事例3の背景にある2つの関数
- [include/linux/falloc.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/falloc.h) — `struct space_resv`/`struct space_resv_32`。事例3の背景にあるネイティブとcompatのレイアウトの不一致

### 関連ページ {#related-pages}

- [カーネルライブパッチ](klp.md) — これらのインシデントの土台となるftraceのリダイレクト機構とshadow変数API
- [KLP一貫性モデル](klp-consistency.md) — 事例1の背景にあるタスクごとの遷移状態とスタック検査
- [累積パッチとアトミックリプレース](klp-cumulative.md) — 事例5の背景にある`.replace=true`、`nop` func、`func_stack`の積み重ね
- [KLP State: 独自の一貫性チェック](klp-state.md) — 累積パッチ間で状態を引き継ぐためのAPI。事例5のコールバックに関する注意点と対になる問題
- [kexec](kexec.md) — ファームウェアのPOSTを完全には経ずにリブートする。修正にダウンタイムが必要な場合のライブパッチの代替手段

### LWNの記事 {#lwn-articles}

- [livepatch: consistency model](https://lwn.net/Articles/632582/)（2015年2月9日） — 事例1の背景にある、タスクごとの一貫性モデルの最初のRFCとタスクごとの遷移の設計
- [livepatch: introduce shadow variable API](https://lwn.net/Articles/731585/)（2017年8月21日） — `klp_shadow_alloc()`/`klp_shadow_get()`/`klp_shadow_free()`と、事例2の背景にある`(obj, id)`をキーとするハッシュテーブルを導入したJoe Lawrenceのパッチ
- [livepatch: introduce atomic replace](https://lwn.net/Articles/734997/)（2017年9月27日） — 事例5で論じたアトミックリプレース/累積パッチの設計

### 外部資料 {#external}

- [Shadow variables](https://docs.kernel.org/livepatch/shadow-vars.html) — shadow変数のライフサイクルを親オブジェクトのライフサイクルに合わせることについてのアップストリームのドキュメント。事例2のパッチが破った規則
