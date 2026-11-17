# 仮想メモリ・物理メモリ・常駐メモリの違い {#virtual-vs-physical-vs-resident-memory}

> `VSZ`、`RSS`を理解し、なぜメモリの数値の合計が合わないのかを知る

## よくある混乱 {#the-confusion}

`ps aux`を実行すると、あるプロセスが500MBの仮想メモリ（`VSZ`）を使っているように見える。しかし`free`を見ると、利用可能なメモリは十分にある。あるいは、すべての`RSS`の値を合計すると、搭載RAMの総量を超えてしまう。何が起きているのだろうか。

答えは、メモリを測る3つの異なる方法を理解することにある。

## 3種類のメモリ {#the-three-memory-types}

### 仮想メモリ（VSZ/VIRT） {#virtual-memory-vszvirt}

**何か:** プロセスがマップしているアドレス空間の総量。

**含まれるもの:**

- コード（テキストセグメント）
- データ（ヒープ、スタック、グローバル変数）
- 共有ライブラリ
- メモリマップトファイル
- まだ触れられていない割り当て

仮想メモリは単なる*アドレス空間*である。物理RAMが使われていることを意味するわけではない。

**メモリ使用量を把握するうえで、`VSZ`はほぼ常に無意味である。** 「このプロセスはどれだけメモリを使っているか?」という問いに答えるのに使ってはならない。代わりに`RSS`、`PSS`、`USS`を使う。`VSZ`が役立つのは、アドレス空間のレイアウトに関する問題をデバッグするときだけである。

```bash
# 仮想500MBのプロセスが500MBのRAMを使っているわけではない
$ ps -o pid,vsz,rss,comm -p $$
  PID    VSZ   RSS COMMAND
 1234 502400 12340 bash
```

この例では、`bash`は500MBの仮想アドレス空間を持つが、RAMに常駐しているのは約12MBだけである。キャパシティプランニングにおいて、`VSZ`の数値は本質的に無意味である。

### 物理メモリ {#physical-memory}

**何か:** システムに実際に搭載されているRAMチップ。

**使っているもの:**

- カーネルのコードとデータ構造
- 常駐しているプロセスのページ（`RSS`）
- ページキャッシュ（メモリにキャッシュされたファイルデータ。ブロックバッファを含む）
- slabキャッシュ（カーネルオブジェクトのキャッシュ）

物理メモリは共有され、キャッシュされ、再利用される。カーネルはこれを希少な資源として管理する。

```bash
$ free -h
              total        used        free      shared  buff/cache   available
Mem:           15Gi       4.2Gi       1.1Gi       890Mi       10Gi        10Gi
```

ここでは10GBがキャッシュに「使われている」が、必要になれば利用可能である。

### 常駐メモリ（RSS/RES） {#resident-memory-rssres}

**何か:** 仮想メモリのうち、現在物理RAMに載っている部分。

**含まれるもの:**

- 実際にRAMにあるプライベートページ
- 共有ページ（ライブラリ、共有メモリ）

**含まれないもの:**

- スワップアウトされたページ
- 一度も触れられていないページ（デマンドページング）
- 現在ロードされていないメモリマップトファイル

共有メモリについては、`RSS`は誤解を招くことがある。その理由を次に説明する。

## なぜ`RSS`の合計が合わないのか {#why-rss-doesnt-add-up}

同じ共有ライブラリを使う2つのプロセスを考える。

```
Process A: RSS = 100MB (includes 40MB libc)
Process B: RSS = 80MB (includes 40MB libc)
Total RSS = 180MB
```

しかし`libc`は物理RAM上に1つしか存在しない。実際に使われているメモリは140MBに近い。

これが、`RSS`の合計がしばしば物理RAMを超える理由である。共有ページが複数回数えられているのだ。

### `PSS`: Proportional Set Size {#pss-proportional-set-size}

この二重計上の問題を解決するため、Linuxは`PSS`を提供している。

```
Shared memory contribution = (shared size) / (number of sharers)
```

上の例では次のようになる。
```
Process A: PSS = 60MB + (40MB / 2) = 80MB
Process B: PSS = 40MB + (40MB / 2) = 60MB
Total PSS = 140MB (accurate!)
```

```bash
# 2.6.25 (2008年) 以降のどのカーネルでも動く
$ awk '/^Pss:/ {sum += $2} END {print sum " kB"}' /proc/<pid>/smaps

# カーネル4.14+ではより高速 (パース不要)
$ grep ^Pss: /proc/<pid>/smaps_rollup
Pss:               98765 kB
```

注: `PSS`はカーネル2.6.25以降、`/proc/<pid>/smaps`経由で利用できる。事前集計された`/proc/<pid>/smaps_rollup`は、パースのオーバーヘッドを避けるために[カーネル4.14で追加された](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=493b0e9d945f)（2017年）。

## 仮想メモリは巨大になりうる {#virtual-memory-can-be-huge}

プロセスは、物理RAMの量よりはるかに多くの仮想メモリをマップできる。

```c
// 16GBのシステムでもこれは成功する
void *p = mmap(NULL, 1UL << 40,  // 1 TB
               PROT_READ | PROT_WRITE,
               MAP_PRIVATE | MAP_ANONYMOUS | MAP_NORESERVE,
               -1, 0);
// MAP_NORESERVE: このマッピング用のスワップ領域を予約しない。
// これがないと、厳格なオーバーコミット (モード2) では割り当てが拒否されうる。
```

なぜか。理由は次のとおりである。
1. **デマンドページング**: ページは触れられるまで割り当てられない
2. **オーバーコミット**: Linuxは物理RAMを超える割り当てを許す
3. **疎なマッピング**: 空間の大部分は一度も使われないかもしれない

## メモリのライフサイクル {#the-memory-lifecycle}

```
┌─────────────────────────────────────────────────────────┐
│                    Virtual Address Space                │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│  │  Mapped  │ │  Mapped  │ │  Mapped  │ │  Mapped  │    │
│  │ (unused) │ │ (in RAM) │ │ (swapped)│ │ (file)   │    │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘    │
└─────────────────────────────────────────────────────────┘
        │              │             │            │
        │              ▼             │            │
        │       ┌──────────┐         │            │
        │       │ Physical │         │            │
        │       │   RAM    │◄────────┘            │
        │       └──────────┘   (swap in)          │
        │              │                          │
        │              ▼                          ▼
        │       ┌──────────┐              ┌──────────┐
        │       │   Swap   │              │   Disk   │
        │       │  Device  │              │  (file)  │
        │       └──────────┘              └──────────┘
        │
        ▼
   Page fault on
   first access
   (demand paging)
```

## 実例 {#practical-examples}

### 例1: JVMのヒープ {#example-1-jvm-heap}

```bash
$ java -Xmx4g -jar myapp.jar &
$ ps -o vsz,rss,comm -p $!
   VSZ    RSS COMMAND
5242880 512000 java
```

仮想: 約5GB（最大ヒープの予約を含む）
常駐: 約500MB（実際のヒープ使用量）

JVMは最大ヒープ分の仮想空間を予約するが、物理RAMは必要な分だけしか使わない。

### 例2: メモリマップトファイル {#example-2-memory-mapped-file}

```bash
$ ls -l bigfile.dat
-rw-r--r-- 1 user user 10737418240 Jan 1 00:00 bigfile.dat  # 10GBのファイル

# プロセスがファイル全体をマップする
$ cat /proc/<pid>/maps | grep bigfile
7f0000000000-7f0280000000 r--s ... bigfile.dat

$ ps -o vsz,rss -p <pid>
    VSZ    RSS
10500000  50000   # 仮想10GB、常駐50MB
```

ファイルはマップされているが、RAMにあるのはアクセスされたページだけである。

### 例3: 共有ライブラリ {#example-3-shared-libraries}

```bash
$ pmap -x <pid> | grep libc
00007f1234000000   2048K r-x-- libc.so.6    # コード (共有)
00007f1234200000     64K r---- libc.so.6    # 読み取り専用データ
00007f1234210000     16K rw--- libc.so.6    # プライベートデータ (COW)
```

`libc`の大部分は、それを使うすべてのプロセスの間で共有される。

### `USS`: Unique Set Size {#uss-unique-set-size}

**`USS`**（Unique Set Size）は、プロセスが使うプライベートメモリだけを測る。これは、そのプロセスだけをkillした場合に解放されるメモリである。共有メモリはすべて除外される。

```bash
# USS = Private_Clean + Private_Dirty (どちらも"Private"にマッチ)
$ grep ^Private /proc/<pid>/smaps | awk '{sum += $2} END {print sum " kB"}'
```

`USS`は、プロセスごとの真のメモリコストを把握するのに役立つ。特に、コンテナの集積度やkillの影響を分析する際に有用である。

## 歴史的背景 {#historical-context}

システムが複雑になり、「このプロセスはどれだけメモリを使っているか?」という問いに答えるのが難しくなるにつれて、メモリの指標は発展してきた。

### 初期のUnix: 単純な指標 {#early-unix-simple-metrics}

初期のUnixシステムでは、メモリの計上は単純だった。

- **仮想サイズ**: どれだけのアドレス空間がマップされているか
- **常駐サイズ**: どれだけが物理RAMにあるか

プロセスがメモリを共有することがまれで、システムのメモリがギガバイトではなくメガバイト単位だった時代には、これで十分だった。

### 共有ライブラリ問題（1990年代〜2000年代） {#the-shared-library-problem-1990s-2000s}

共有ライブラリが当たり前になると、`RSS`は誤解を招くものになった。100個のプロセスが`libc`を共有していると、それぞれが`RSS`に`libc`のサイズを含めて報告するため、合計すると大幅に水増しされた値になる。

`/proc`ファイルシステム（Linux開発の初期、1.0よりかなり前から存在）は基本的なメモリ統計を提供していたが、マッピングごとの詳細な情報が得られるようになったのは後のことである。

### Linuxが詳細な計上を追加 {#linux-adds-detailed-accounting}

| バージョン | 年 | 追加内容 |
|---------|------|----------|
| **2.6.14** | 2005年10月 | `/proc/pid/smaps` - Mauricio Linによる[commit e070ad49](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e070ad49f31155d872d8e96cab2142840993e3c0) |
| **2.6.25** | 2008年4月 | **`PSS`の追加** - Fengguang Wuによる[LKMLパッチ](https://lkml.iu.edu/hypermail/linux/kernel/0710.3/1565.html)、構想はMatt Mackall |
| **4.14** | 2017年11月 | `/proc/pid/smaps_rollup` - 集計統計を高速化する[commit 493b0e9d](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=493b0e9d945f) |

### `PSS`という革新（2007〜2008年） {#the-pss-innovation-2007-2008}

Matt Mackallは[Embedded Linux Conference](https://lwn.net/Articles/230975/)で、*「アプリケーションは実際にどれだけのメモリを使っているのか?」*という問いに答えるために`PSS`を提案した。

洞察は単純だった。4つのプロセスが40MBのライブラリを共有しているなら、それぞれに40MBではなく10MB（40/4）を割り当てる。こうすると、`PSS`の値の合計が実際の物理メモリ使用量に一致する。

[最初のLKMLパッチ](https://lkml.iu.edu/hypermail/linux/kernel/0708.1/3420.html)（2007年8月）は、Matt Mackallの構想を実装したものとしてFengguang Wuによって投稿された。これはカーネル2.6.25で[「maps4」シリーズ](https://lwn.net/Articles/256333/)の一部としてマージされた（[commit ec4dd3eb3575](https://git.kernel.org/linus/ec4dd3eb3575)）。

### より細かいPSSの内訳 {#finer-grained-pss-breakdown}

現代のカーネル（4.14+）は、`smaps_rollup`で種類別に分けたPSSを提供する。

```bash
$ grep Pss /proc/<pid>/smaps_rollup
Pss:               98765 kB
Pss_Anon:          41000 kB   # 匿名 (ヒープ、スタック)
Pss_File:          54000 kB   # ファイルバックのマッピング
Pss_Shmem:          3765 kB   # 共有メモリ (41000 + 54000 + 3765 = 98765)
```

smapsをパースせずにRSSを手早く確認するには、`/proc/<pid>/status`を使う。

```bash
$ grep -E '^(VmRSS|RssAnon|RssFile|RssShmem)' /proc/<pid>/status
VmRSS:      123456 kB
RssAnon:     45000 kB
RssFile:     75000 kB
RssShmem:     3456 kB
```

### 現代の課題 {#modern-challenges}

今日、複雑さはさらに増し続けている。

- **`cgroups`**: プロセスグループ全体にわたるメモリ制限
- **`KSM`**: プロセス間で同一のページを統合する
- **Transparent Huge Pages**: 2MB/1GBのページが計上を複雑にする
- **メモリ階層化**: 高速なメモリと低速なメモリ（DRAMとPMEM）

基本的な`VSZ`/`RSS`/`PSS`のモデルは今でも通用するが、`cgroup`のメモリ統計や`/proc/meminfo`のようなツールが、より完全な全体像を提供する。

## まとめ {#summary}

| 指標 | 測るもの | 共有メモリの扱い | 用途 |
|--------|------------------|---------------|---------|
| `VSZ`/`VIRT` | アドレス空間 | すべて数える | アドレスレイアウトの確認 |
| `RSS`/`RES` | RAM上のページ | プロセスごとに数える | 手早いメモリ確認 |
| `PSS` | 按分したRAM | 公平に分割する | 正確な計上 |
| `USS` | プライベートのみ | 除外する | プロセスごとの影響 |

## 試してみる {#try-it-yourself}

```bash
# VSZとRSSを比較
ps aux --sort=-%mem | head -10

# メモリの詳細な内訳を見る
cat /proc/self/smaps_rollup

# メモリの変化を観察
watch -n 1 'cat /proc/meminfo | head -10'

# 共有とプライベートを見る
pmap -x <pid>
```

## 参考文献 {#further-reading}

- [proc(5) man page](https://man7.org/linux/man-pages/man5/proc.5.html) - `/proc`ファイルシステムのドキュメント
- [ELC: How much memory are applications really using?](https://lwn.net/Articles/230975/) - Matt Mackallによる`PSS`の紹介
- [smaps_rollupのLKMLでの議論](https://lore.kernel.org/lkml/20170806225154.18820-1-dancol@google.com/) - 最初のパッチシリーズ
