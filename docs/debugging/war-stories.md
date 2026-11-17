# デバッグ事件簿 {#debugging-war-stories}

> KASAN、lockdep、syzkaller、その他のツールが捕まえたバグ

ここに挙げるのは、カーネルのデバッグツールが現実に捕まえる種類のバグについて、技術的な裏付けのある記述である。各ストーリーは、バグの分類、それを表面化させたツール、出力の読み方、そして修正を示す。

!!! note "これらの記述について"
    これらは、1つのコミットやCVEに結び付いた特定の事件の報告ではない。代わりに、実際に繰り返し現れるバグとツールのパターンを示している（[カーネルコアの事件簿](../kernel/war-stories.md)と同じアプローチに従う）。ページ末尾の参考文献は、根底にあるカーネルの仕組みを現在のメインラインのソースとドキュメントに照らして検証している。サイトの通常の形式、つまり日付を特定できる具体的な事件で、その大半（すべてではない — [wake_wide()](../sched/war-stories/wake-wide-heuristic.md)を参照）が修正コミットやCVEに結び付いているものについては、[BPF](../bpf/war-stories.md)、[スケジューラ](../sched/war-stories.md)、[割り込み](../interrupts/war-stories.md)の事件簿ページを参照のこと。

---

## 1. KASANがネットワークドライバのRXパスでuse-after-freeを発見する {#1-kasan-finds-a-use-after-free-in-a-network-driver-rx-path}

**ツール**: KASAN（generic、`CONFIG_KASAN_GENERIC`）
**バグの分類**: use-after-free
**サブシステム**: ネットワーク（`sk_buff`、NAPIのRXハンドラ）

### 何が起きたか {#what-happened}

あるネットワークドライバのRX完了ハンドラは、受信したフレームをループで処理していた。ループ内で、ドライバは`netif_receive_skb()`で`sk_buff`をスタックの上位に渡した後、`dev_kfree_skb()`を呼んでそれを解放していた。しかしその後、ループは統計処理のコードで`skb->len`にアクセスしていた。skbが解放された後に、である。

このアクセスは、開発者が最適化を進めていた高速パス内にあった。以前のリファクタリングで統計処理のコードが`dev_kfree_skb()`呼び出しの下に移動したが、その時点で`skb`がダングリングポインタになっていることに気づかなかったのである。

### KASANレポート {#the-kasan-report}

```
==================================================================
BUG: KASAN: use-after-free in netif_receive_skb_internal+0x234/0x580
Read of size 4 at addr ffff8881043a1068 by task ksoftirqd/0/12

CPU: 0 PID: 12 Comm: ksoftirqd/0 Not tainted 6.7.0 #1
Call Trace:
 kasan_report+0xbd/0x100
 kasan_check_range+0x115/0x1a0
 netif_receive_skb_internal+0x234/0x580    ← アクセス箇所
 mydrv_rx_completion+0x1a8/0x3f0 [mydrv]   ← 問題のドライバ

Allocated by task 0:
 __kasan_kmalloc+0x81/0xa0
 kmem_cache_alloc+0x68/0x2e0
 __alloc_skb+0x44/0x200                    ← ここで割り当て
 mydrv_rx_alloc+0x20/0x60 [mydrv]

Freed by task 12:
 __kasan_slab_free+0x120/0x160
 kmem_cache_free+0x68/0x2e0
 kfree_skb_reason+0x58/0xa0
 dev_kfree_skb+0x18/0x20
 mydrv_rx_completion+0x198/0x3f0 [mydrv]  ← ここで解放（上のアクセスより前）
==================================================================
```

レポートの読み方:

- **Read of size 4**は`skb->len`（`struct sk_buff`のオフセット`0x68`にある`__u32`）へのアクセスである
- 解放スタックは`mydrv_rx_completion+0x198`を、アクセススタックは`mydrv_rx_completion+0x1a8`を示している。アクセスは同じ関数内で16バイト後にあり、同じ呼び出しフレーム内でのuse-after-freeであることが確認できる

### 修正 {#the-fix}

```c
/* 修正前（バグあり）: */
void mydrv_rx_completion(struct mydrv *priv, struct rx_desc *desc)
{
    struct sk_buff *skb = desc->skb;

    skb_put(skb, desc->len);
    netif_receive_skb(skb);
    dev_kfree_skb(skb);          /* ここでskbが解放される */

    priv->stats.rx_bytes += skb->len;   /* バグ: skbはダングリングポインタ */
}

/* 修正後: 解放前にlenを保存する */
void mydrv_rx_completion(struct mydrv *priv, struct rx_desc *desc)
{
    struct sk_buff *skb = desc->skb;
    unsigned int len = desc->len;   /* 解放前に保存 */

    skb_put(skb, len);
    netif_receive_skb(skb);
    dev_kfree_skb(skb);

    priv->stats.rx_bytes += len;    /* 保存したコピーを使う */
}
```

**教訓**: KASANはアクセススタックと並べて、割り当てと解放の正確なスタックを報告する。そのため、最適化されたコードであってもUAFの原因がすぐにわかる。

---

## 2. lockdepが本番投入前にABBAデッドロックを捕まえる {#2-lockdep-catches-an-abba-deadlock-before-production}

**ツール**: lockdep（`CONFIG_PROVE_LOCKING`）
**バグの分類**: 循環ロック依存（デッドロックの可能性）
**サブシステム**: ファイルシステム（ext4風のinode統計）

### 何が起きたか {#what-happened_1}

ある開発者が、集計I/O統計を保護するために、スーパーブロックごとの`stats_lock` spinlockを追加した。そして、新しいロックを獲得する2つのコードパスを導入した:

- **パスA**（`ext4_write_end`）: `inode->i_rwsem`（書き込み）を獲得し、次に`sb->stats_lock`を獲得する
- **パスB**（`stats_flush_worker`）: `sb->stats_lock`を獲得し、次にダーティなinodeを走査して`inode_lock()`（`inode->i_rwsem`を書き込みで獲得する）を呼ぶkworker

これは典型的なABBAパターンである:
```
Path A: inode→i_rwsem → sb→stats_lock
Path B: sb→stats_lock → inode→i_rwsem
```

### lockdepのsplat {#the-lockdep-splat}

```
======================================================
WARNING: possible circular locking dependency detected
------------------------------------------------------
kworker/u4:1/234 is trying to acquire lock:
ffff888100cd1a80 (&inode->i_rwsem){++++}-{3:3}, at: inode_lock+0x1d/0x30

but task is already holding:
ffffffff82a45678 (&sb->stats_lock){....}-{2:2}, at: stats_flush_worker+0x44/0x120

which lock already depends on the new lock.

the existing dependency chain (in reverse order) is:
-> #1 (&sb->stats_lock){....}:
       _raw_spin_lock+0x52/0x70
       ext4_write_end+0x2c0/0x5a0    ← パスA: i_rwsemを保持し、stats_lockを獲得

-> #0 (&inode->i_rwsem){++++}:
       down_write+0x52/0x120
       inode_lock+0x1d/0x30
       stats_flush_worker+0x78/0x120  ← パスB: stats_lockを保持し、i_rwsemを獲得

 *** DEADLOCK ***
======================================================
```

lockdepは、本番システムがこのコードを実行するより前に、`stats_flush_worker`の**最初の実行時**に発火した。実際のデッドロックはまだ起きていなかったが、忙しいファイルシステムでは両方のパスが別々のCPUで同時に実行されうる状況だった。

### 修正 {#the-fix_1}

開発者は`stats_flush_worker`での獲得順序を逆にした。`stats_lock`の下でダーティinodeのリストをローカル配列に集め、`stats_lock`を解放してから、各inodeのロックを個別に獲得するようにしたのである。

```c
/* 修正前（バグあり）: */
void stats_flush_worker(struct work_struct *work)
{
    struct super_block *sb = container_of(work, ...);

    spin_lock(&sb->stats_lock);           /* 先にstats_lockを獲得 */
    list_for_each_entry(inode, &sb->s_inodes, i_sb_list) {
        inode_lock(inode);                /* 次にi_rwsem — ABBA! */
        flush_inode_stats(inode, sb);
        inode_unlock(inode);
    }
    spin_unlock(&sb->stats_lock);
}

/* 修正後: ロック下でスナップショットを取り、stats_lockなしで走査する */
void stats_flush_worker(struct work_struct *work)
{
    struct super_block *sb = container_of(work, ...);
    u64 total_bytes = 0;

    spin_lock(&sb->stats_lock);
    total_bytes = sb->pending_bytes;
    sb->pending_bytes = 0;
    spin_unlock(&sb->stats_lock);       /* inodeのロックの前にstats_lockを解放 */

    /* これでstats_lockを保持せずに安全にinodeのロックを獲得できる */
    update_sb_stats(sb, total_bytes);
}
```

**教訓**: lockdepは、再現が不可能に近いデッドロックを決定的に見つける。実際のデッドロックには、2つのCPUがまさに悪いタイミングでインターリーブする必要があり、負荷の高いシステムでも週に1回しか起きないかもしれない。lockdepは最初のテスト実行でそれを見つけた。

---

## 3. syzkallerが細工したio_uring SQEでNULL参照外しを発見する {#3-syzkaller-finds-a-null-deref-via-crafted-io_uring-sqe}

**ツール**: syzkaller + syzbot
**バグの分類**: NULLポインタ参照外し（境界チェックの欠落）
**サブシステム**: io_uring（`io_write`、固定ファイルテーブル）

### 何が起きたか {#what-happened_2}

新しいio_uringの機能が`linux-next`に入った。2時間以内に、syzbotは`io_write()`でのNULLポインタ参照外しを報告した。これは、`IOSQE_FIXED_FILE`を設定した`IORING_OP_WRITE`のSQEで、`ctx->file_table.files`がNULL（固定ファイルが登録されていない）の場合に発生した。

カーネルは、`ctx->file_table.files`が初期化されているかを確認せずに`ctx->file_table.files[req->fixed_file]`を読んでいた。再現プログラムは20行のCプログラムだった。

### syzbotのレポート {#the-syzbot-report}

```
BUG: kernel NULL pointer dereference, address: 0000000000000000
#PF: supervisor read access in kernel mode
#PF: error_code(0x0000) - not-present page
RIP: 0010:io_file_get_fixed+0x2c/0x90

Call Trace:
 io_write+0xa4/0x5a0
 io_issue_sqe+0x3a0/0x8c0
 io_submit_sqes+0x1f0/0x4d0
 __sys_io_uring_enter+0x2c0/0x560
 __x64_sys_io_uring_enter+0x1c/0x30
 do_syscall_64+0x5b/0x1d0

Reproduced with:
 r0 = io_uring_setup(0x8, &(0x7f0000000000)={0x1f4})  /* setup with no fixed files */
 io_uring_enter(r0, 0x1, 0x0, 0x0,
   {op=IORING_OP_WRITE, flags=IOSQE_FIXED_FILE, fd=0x0,
    buf=0x7f00, len=0x10, off=0x0})
```

### 根本原因 {#the-root-cause}

```c
/* 修正前（バグあり） — io_uring/io_uring.c: */
static struct file *io_file_get_fixed(struct io_ring_ctx *ctx,
                                      struct io_kiocb *req,
                                      unsigned int issue_flags)
{
    unsigned int fd = req->fixed_file;   /* SQEからの固定ファイルインデックス */

    /* 欠落: file_tableが初期化されているかのチェック */
    if (fd >= ctx->file_table.nr)    /* nrが0 → 次の行でNULLを返す */
        return NULL;

    return io_fixed_file_slot(&ctx->file_table, fd)->file_ptr & ~3;
    /* file_table.filesがNULL → NULL + オフセットの参照外し */
}
```

### 修正 {#the-fix_2}

```c
/* 修正後: */
static struct file *io_file_get_fixed(struct io_ring_ctx *ctx,
                                      struct io_kiocb *req,
                                      unsigned int issue_flags)
{
    unsigned int fd = req->fixed_file;   /* SQEからの固定ファイルインデックス */

    if (unlikely(!ctx->file_table.files))   /* ガード: 固定ファイルが登録されていない */
        return NULL;
    if (unlikely(fd >= ctx->file_table.nr))
        return NULL;

    return io_fixed_file_slot(&ctx->file_table, fd)->file_ptr & ~3;
}
```

あるいは、`io_write()`から早期に`-EBADF`を返すチェックにすることもできる:
```c
if (req->flags & REQ_F_FIXED_FILE) {
    if (unlikely(!ctx->file_table.files))
        return -EBADF;
}
```

**教訓**: syzkallerは、人間のテスターなら決して試さないようなフラグとレジスタのあらゆる組み合わせを体系的に試す。細工されたSQEは微妙な初期化の競合を突いた。この機能は固定ファイルが登録されていれば問題なく動作したが、登録数がゼロの場合は動作しなかった。syzbotのC再現プログラムはバグレポートとともにメーリングリストに送られ、パッチは24時間以内にマージされた。

---

## 4. KFENCEが`kstrdup`のoff-by-oneを捕まえる {#4-kfence-catches-an-off-by-one-in-kstrdup}

**ツール**: KFENCE（`CONFIG_KFENCE`）
**バグの分類**: ヒープの境界外書き込み（off-by-one）
**サブシステム**: 文字列ヘルパー / ドライバコア

### 何が起きたか {#what-happened_3}

最近のカーネルを実行していた本番サーバーで、カーネルログにKFENCEレポートが出始めた。システムは`CONFIG_KFENCE=y`で、サンプリング間隔はデフォルトの100msだった。レポートは起動後1時間以内に現れた。

### KFENCEレポート {#the-kfence-report}

```
==================================================================
BUG: KFENCE: out-of-bounds write in kstrdup+0x48/0x70

Out-of-bounds write at 0xffff888080a3c007 (1B left of kfence-#17):
 kstrdup+0x48/0x70           ← 7バイトの割り当ての8バイト目に書き込む
 some_helper+0x24/0x60 [mydriver]
 driver_probe_device+0x1a8/0x3c0
 bus_probe_device+0x88/0xe0

kfence-#17 [0xffff888080a3c000-0xffff888080a3c006, size=7, cache=kmalloc-8]
allocated by task 1:
 kfence_alloc+0x7c/0xb0
 __kmalloc+0x3e/0x80
 kstrdup+0x20/0x70
 some_helper+0x10/0x60 [mydriver]

CPU: 1 PID: 1 Comm: swapper/0 Not tainted 6.8.0 #1
==================================================================
```

### 根本原因 {#root-cause}

`mydriver`のヘルパーは、アップストリームのユーティリティ関数を介して間接的に`kstrdup()`を呼んでいた。そのユーティリティ関数にはバグがあり、`strlen(s) + 1`ではなく`strlen(s)`バイトを割り当てていたため、NUL終端の分の領域が抜けていた。そして`kstrdup`の実装はバイトインデックス`len`、つまり割り当ての末尾を1つ越えた位置にNULを書き込んでいた。

```c
/* バグのあるアップストリームのヘルパー: */
char *make_name(const char *prefix, const char *suffix)
{
    size_t len = strlen(prefix) + strlen(suffix);
    char *buf = kmalloc(len, GFP_KERNEL);   /* バグ: NUL用の+1がない */
    if (!buf)
        return NULL;
    sprintf(buf, "%s%s", prefix, suffix);   /* buf[len]にNULを書き込む */
    return buf;
}
```

このバグは複数のカーネルリリースにわたって存在していたが、KASANを有効にしたカーネルでこのコードパスがまさに実行されたことは一度もなかった。`make_name()`に到達するのは、KASANを有効にしたCI環境にも開発者のローカルのテスト環境にも存在しないハードウェアに結び付いたドライバのプローブパスからだけだったのである。これはカバレッジの穴であり、KASANの検出粒度の穴ではない。generic KASANのシャドウバイトは各8バイトの粒度単位内でアクセス可能なバイト数を正確にエンコードしている（シャドウバイトの表は[KASANとKFENCE](kasan-kfence.md)を参照）ので、7バイトの`kmalloc-8`オブジェクトの8バイト目へのNUL書き込みは、KASANのもとで実行されていれば即座に検出されていたはずである。このバグが生き延びたのは、KASANの約2〜3倍のCPUオーバーヘッドのために、KASANは通常、開発とCIで、エンジニアがたまたまそこで実行するハードウェアとコードパス上でしか動かされないからであって、このちょうど境界でのoff-by-oneがKASANから何らかの理由で見えないからではない。

対照的にKFENCEは、本番環境でほぼゼロのオーバーヘッドで割り当てをサンプリングするので、この割り当てがたまたま4KBのデータページ内で右寄せになったKFENCEスロットに入ったとき、最終的にそれを捕まえた（KFENCEは割り当てごとに左右の配置をランダム化する）。ガードページは7バイト目の直後にあった。8バイト目へのNUL書き込みがガードページに当たり、フォールトしたのである。

### 修正 {#the-fix_3}

```c
/* 修正版: */
char *make_name(const char *prefix, const char *suffix)
{
    size_t len = strlen(prefix) + strlen(suffix) + 1;   /* NUL用の+1 */
    char *buf = kmalloc(len, GFP_KERNEL);
    if (!buf)
        return NULL;
    sprintf(buf, "%s%s", prefix, suffix);
    return buf;
}
```

あるいは、より慣用的に`kasprintf()`を使う:

```c
char *make_name(const char *prefix, const char *suffix)
{
    return kasprintf(GFP_KERNEL, "%s%s", prefix, suffix);
}
```

**教訓**: KASANとKFENCEはどちらも正確な検出器だが、どちらも実行されないコードパス上のバグは捕まえられない。KASANはオーバーヘッドのために開発とCIに限定され、そこではエンジニアがテストするハードウェアとワークロードしかカバーされない。KFENCEはオーバーヘッドがほぼゼロなので、本番のトラフィックを継続的にサンプリングできる。本番環境でKFENCEを動かしたことで、コードレビューとKASANを有効にしたすべてのテスト実行をすり抜けていたバグが捕まった。それらのどれもが、このコードパスに到達したことがなかったというだけの理由ですり抜けていたのである。

---

## 5. 静かなメモリ破壊の後でkdumpが窮地を救う {#5-kdump-saves-the-day-after-silent-memory-corruption}

**ツール**: kdump + `crash` + KASAN（追跡調査）
**バグの分類**: `kmem_cache`のフリーリストを破壊するuse-after-free
**サブシステム**: ブロック層（`bio`オブジェクト）、デバイスのライフサイクル

### 何が起きたか {#what-happened_4}

あるストレージサーバーで、周期的に静かなハングが発生していた。システムが応答しなくなり、（ハードウェアウォッチドッグによって）再起動するが、oopsもパニックもなく、カーネルログにもメッセージがない。クラッシュは高いI/O負荷の下で数日おきに発生した。

kdumpを有効にした（`CONFIG_KEXEC=y`、ブートコマンドラインに`crashkernel=256M`）後、チームは次のクラッシュでvmcoreを取得した。

### crashによる解析 {#crash-analysis}

```bash
crash vmlinux /var/crash/vmcore

crash> kmem -s bio
CACHE         OBJSIZE  ALLOCATED     TOTAL  SLABS  SLAB SIZE  NAME
ffff888200041800     256       1234      1280      5     8192  bio

crash> kmem -S bio
...
FREE / [ALLOCATED]
 ffff888100ab1200  (0)   ← 空きオブジェクト
[ffff888100ab1300] (1)   ← 割り当て済みオブジェクト（正常）
 ffff888100ab1400  (0)   ← 空きオブジェクト — だがそのフリーリストポインタを見ると:

crash> rd ffff888100ab1400 4
ffff888100ab1400:  deadbeef41414141 0000000000000000   ←  破壊されたフリーリストポインタ!
```

`bio`オブジェクトの`kmem_cache`フリーリストが破壊されていた。解放済みの`bio`が上書きされており、その先頭8バイト（`SLUB`がフリーリストポインタとして使う部分）には`0xdeadbeef41414141`が入っていた。これは明らかにカーネルコードが書き込んだもので、有効なカーネルポインタではない。

この破壊されたフリーリストポインタにより、いずれ`kmem_cache_alloc()`が偽のポインタを返すことになる。その時点でカーネルは任意のカーネルアドレスに書き込み、静かなハングに至る。

### 書き込み元を突き止める {#tracking-down-the-writer}

`crash`セッションにより、破壊された`bio`を最後に割り当てたのが特定のブロックドライバであることが、`SLUB`のデバッグメタデータに格納された`bio`のバックトレースからわかった（`CONFIG_SLUB_DEBUG=y`が有効だったので見えた）。

その情報をもとに、チームは同じドライバとワークロードを使うテストシステムで`CONFIG_KASAN=y`を有効にした。

### KASANレポート {#the-kasan-report_1}

テスト用のI/Oワークロードを開始して数分以内に:

```
==================================================================
BUG: KASAN: use-after-free in blk_complete_request+0x8c/0x120
Write of size 8 at addr ffff888100ab1400 by task blkd/56

Freed by task 56:
 __kasan_slab_free+0x120/0x160
 bio_put+0x44/0x80
 bio_endio+0x9c/0x100        ← bio_endio経由でここでbioが解放される

Allocated by task 56:
 __kasan_kmalloc+0x81/0xa0
 bio_alloc_bioset+0x68/0x1a0
 myblk_submit_io+0x34/0x80 [myblk]
==================================================================
```

### 根本原因 {#root-cause_1}

ドライバは`bio_endio()`を呼んだ後に`bio->bi_iter`にアクセスしていた。`bio_endio()`はbioの完了コールバックを呼び、それが`bio_put()`を呼ぶことがある。これにより最後の参照が落とされ、bioはslabに解放される。ドライバの参照カウントが誤っていれば、`bio_endio()`がbioを解放し、その後ドライバが解放済みメモリに書き込むことになる。

実際のバグ: ドライバは対象デバイスに対して先に`get_device()`を呼ばずに`bio_endio()`を呼んでいた。メモリ逼迫時にデバイスの`release()`コールバックが実行され、bioプールを含む共有構造体を解放した。その後`bio_endio()`が実行されると、すでに破棄されたプールにbioを解放し、フリーリストを破壊したのである。

### 修正 {#the-fix_4}

```c
/* 修正前（バグあり）: ドライバはデバイスの参照を保持せずにI/Oを発行する */
static int myblk_submit_io(struct myblk_dev *dev, struct bio *bio)
{
    bio->bi_end_io = myblk_end_io;
    bio->bi_private = dev;
    submit_bio(bio);
    return 0;
}

/* 修正後: 各I/Oの生存期間中はデバイスの参照を保持する */
static int myblk_submit_io(struct myblk_dev *dev, struct bio *bio)
{
    if (!get_device(&dev->dev))   /* 発行前に参照を増やす */
        return -ENODEV;

    bio->bi_end_io = myblk_end_io;
    bio->bi_private = dev;
    submit_bio(bio);
    return 0;
}

static void myblk_end_io(struct bio *bio)
{
    struct myblk_dev *dev = bio->bi_private;

    /* ... 完了を処理する ... */
    bio_put(bio);
    put_device(&dev->dev);        /* I/O完了後に参照を落とす */
}
```

**教訓**: 静かなメモリ破壊は最もデバッグが難しい種類のバグである。破壊の時点ではoopsが出ず、クラッシュは後になって破壊されたデータが使われたときに起きるからだ。ここでのワークフローは次のとおりである: (1) kdumpでvmcoreを取得して破壊された状態を見る、(2) `crash`を使って何が上書きされたかを特定し、それをサブシステムまで追跡する、(3) テストシステムでKASANを有効にして、実際の書き込みをそれが起きた瞬間に捕まえる。どちらのツールも単独では十分ではなかっただろう。

---

## ツールのまとめ {#tool-summary}

| ストーリー | バグ | 発見したツール | 発見までの時間 |
|---|---|---|---|
| ネットワークドライバのUAF | `dev_kfree_skb()`後の`skb->len` | KASAN | 最初のテスト実行 |
| ファイルシステムのABBAデッドロック | `stats_lock` → `i_rwsem`の逆転 | lockdep | 最初のテスト実行 |
| io_uringのNULL参照外し | 固定ファイルテーブルの境界チェック欠落 | syzkaller | マージから2時間 |
| `kstrdup`のoff-by-one | `strlen(s)+1`ではなく`strlen(s)` | KFENCE（本番） | 稼働1時間 |
| `bio`フリーリストの破壊 | `get_device()`の欠落 | kdump → crash → KASAN | 取得まで数日、診断まで数時間 |

---

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/locking/lockdep.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/locking/lockdep.c) — `check_noncircular()`と`check_deadlock()`。ケース2のsplatの元になる実行時の循環検出コード
- [mm/kfence/core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kfence/core.c)と[lib/Kconfig.kfence](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/Kconfig.kfence) — サンプリングゲートの実装と`CONFIG_KFENCE_SAMPLE_INTERVAL`（デフォルト100、単位はミリ秒）。ケース4の「サンプリング間隔はデフォルトの100ms」という記述の裏付け
- [mm/slub.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/slub.c) — `get_freepointer()`/`set_freepointer()`。ケース5で`bio`のフリーリストが破壊された背景にあるフリーリストポインタの仕組み
- [mm/kasan/report.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/kasan/report.c) — `kasan_report()`。ケース1とケース5の追跡調査でのKASANレポートの元になる報告経路
- [io_uring/io_uring.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/io_uring/io_uring.c) — `io_file_get_fixed()`。ケース3の説明用バグのモデルとなった実際の固定ファイル解決関数（現在のソースは、ここで示した簡略化された直接のインデックスチェックではなく、`io_rsrc_node_lookup()`を通じて検証している）

### 関連ページ {#related-pages}

- [KASANとKFENCE](kasan-kfence.md) — メモリエラー検出器のリファレンス
- [lockdepの実践](dynamic-debug-lockdep.md) — splatの読み方、誤検知へのアノテーション
- [syzkaller](syzkaller.md) — カーネルファザーのセットアップと使い方
- [kdumpとcrash](kdump.md) — カーネルクラッシュダンプの取得と解析
- [Oops解析](oops-analysis.md) — スタックトレースのデコード
- [KCSAN](kcsan.md) — データ競合検出

### 外部資料 {#external}

- [Kernel documentation: KASAN](https://docs.kernel.org/dev-tools/kasan.html) — シャドウメモリとuse-after-free/境界外アクセスの検出。ケース1のレポートとケース5の追跡調査でのKASAN実行の背景にある仕組み
- [Kernel documentation: KFENCE](https://docs.kernel.org/dev-tools/kfence.html) — サンプリング型のガードページ検出器、本番環境でも安全なオーバーヘッド、スロット内での左右の割り当て配置。ケース4の背景
- [Kernel documentation: Lockdep Design](https://docs.kernel.org/locking/lockdep-design.html) — ケース2の「possible circular locking dependency detected」splatを生成する依存循環の証明エンジン
- [Kernel documentation: kdump](https://docs.kernel.org/admin-guide/kdump/kdump.html) — kexecベースのクラッシュカーネルによる取得と、ケース5でvmcoreの解析に使った`crash`ユーティリティ
- [Kernel documentation: SLUB debugging](https://docs.kernel.org/admin-guide/mm/slab.html) — `CONFIG_SLUB_DEBUG`、レッドゾーン、オブジェクトのポイズニング。ケース5で`crash`が破壊された`bio`の割り当てバックトレースを追跡できた仕組み

ページ冒頭で述べたとおり、上の本文には特定のコミットハッシュ、CVE番号、カーネルバージョンに結び付いた機能の主張は登場しない。代わりに上の参考文献が、根底にあるカーネルの仕組み — KASAN/KFENCEの検出の内部、lockdepの循環検出、kdump/crashによる取得、SLUBのデバッグメタデータ、io_uringの固定ファイル解決 — を現在のメインラインのソースとドキュメントに照らして検証している。
