# メモリのオーバーコミット: なぜそれが唯一まともなデフォルトなのか {#memory-overcommit-why-its-the-only-sane-default}

> 本当の問いは「なぜオーバーコミットするのか」ではない。「なぜオーバーコミット*しない*理由があるのか」である

## 本当の問い {#the-real-question}

RAMが8GBあるとする。プログラムが`malloc(16GB)`を呼び出す。カーネルは拒否すべきだろうか。

ほとんどのLinuxシステムでは（デフォルト設定のままなら）、この割り当ては許可される。しかし問うべきは「なぜLinuxはこんな無茶なことをするのか?」ではない。問うべきは**代わりにどうするのか?**である。

## オーバーコミットがなければfork()が破綻する {#without-overcommit-fork-breaks}

答えは`fork()`と[コピーオンライト](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memory.c)（COW）から始まる。

`fork()`すると、子は親のアドレス空間のコピーを得る。しかし、どちらかのプロセスが書き込むまで実際にはページは1つもコピーされない（`copy_present_pte()`を参照）。これがCOWであり、Unixにとって不可欠なものである。

だがここに問題がある。オーバーコミットがなければ、カーネルはfork時に、それら潜在的なコピーすべてのための物理メモリを**予約**しなければならない。4GBのシステム上の3GBのプロセスは、たとえ子がすぐに`exec()`を呼んですべてを破棄するとしても、forkできなくなる。

それは正気の沙汰ではない。だからオーバーコミットが存在する。

fork()のためにオーバーコミットを受け入れてしまえば、それを一般の割り当てに広げるのは小さな一歩である。
1. 疎な配列が機能する（1GBをマップし、10MBに触れる）
2. メモリマップトファイルは、読まれていないページのためにRAMを必要としない
3. 予約してから使うプログラム（JVMなど）がメモリを無駄にしない

## fork()の例 {#the-fork-example}

`fork()`を考えてみよう。

```c
// 4GBのRAMを使っている親プロセス
pid_t pid = fork();
// 4GBのアドレス空間のコピーを持つ子が作られる
if (pid == 0) {
    char *args[] = {"/bin/ls", NULL};
    execve("/bin/ls", args, NULL);  // ただちにアドレス空間を置き換える
}
```

オーバーコミットがなければ、子がすぐに`exec()`を呼んでコピーを破棄するにもかかわらず、`fork()`には合計8GB（親4GB + 子4GB）が必要になる。

コピーオンライトとオーバーコミットがあれば、次のようになる。

- 子は親のページを共有する（まだコピーしない）
- コピーが起きる前に`exec()`がアドレス空間を置き換える
- 実際に必要なメモリ: 約4GB（8GBではない）

## オーバーコミットの仕組み {#how-overcommit-works}

プロセスが`mmap()`や`brk()`（実際のシステムコール。`malloc()`はこれらを呼び出すユーザー空間のラッパーである）でメモリを要求すると、カーネルは次のことを行う。

1. **VMAを作成する** - 領域を記述する`vm_area_struct`（定義は[include/linux/mm_types.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mm_types.h)、操作は[mm/vma.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vma.c)）
2. **物理RAMは割り当てない** - まだページは割り当てられない
3. **成功を返す** - プロセスはそのアドレスを使える

注: ページテーブルはmmap時には作られ**ない**。ページに初めてアクセスしたとき（ページフォールト）にオンデマンドで作られる。

物理RAMは後で、最初のアクセス時に割り当てられる（デマンドページング）。

```
mmap(1GB)            First write to page
     │                      │
     ▼                      ▼
┌─────────┐           ┌─────────┐
│ VMA     │           │ Page    │
│ created │    ───►   │ fault   │
│ (no RAM)│           │ handler │
└─────────┘           └─────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │ Allocate    │
                    │ physical    │
                    │ page now    │
                    └─────────────┘
```

## オーバーコミットのモード {#overcommit-modes}

Linuxは`/proc/sys/vm/overcommit_memory`で3つのモードを提供する。

### モード0: ヒューリスティック（デフォルト） {#mode-0-heuristic-default}

```bash
$ cat /proc/sys/vm/overcommit_memory
0
```

**挙動:** Linuxはヒューリスティックを使って、割り当てが「妥当」かどうかを判断する。

- 明らかに過大な割り当ては拒否する
- 適度なオーバーコミットは許可する
- 利用可能なRAM、スワップ、現在の使用量を考慮する

このヒューリスティックは`__vm_enough_memory()`（[mm/util.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/util.c)）に実装されている。RAMの総量、スワップ、hugetlbの予約、管理者/ユーザー向けの予備、既存のコミットを考慮する。単純化したルールは、明らかに過大なら拒否し、満たせそうなら許可する、というものである。

### モード1: 常にオーバーコミット {#mode-1-always-overcommit}

```bash
$ echo 1 > /proc/sys/vm/overcommit_memory
```

**挙動:** メモリの空き状況を理由に割り当てを拒否することは決してない。

- 16GBのシステムで1TBの`mmap()`が成功する
- 疎な配列やメモリマップトファイルに有用
- プログラムが割り当てた分に実際に触れる場合は危険

### モード2: 厳格な計上 {#mode-2-strict-accounting}

```bash
$ echo 2 > /proc/sys/vm/overcommit_memory
```

**挙動:** コミット量を追跡し、上限を超えたら拒否する。

コミット上限は[mm/util.c:vm_commit_limit()](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/util.c)で計算される。

```
If overcommit_kbytes is set:
    Commit limit = overcommit_kbytes + Swap

Otherwise:
    Commit limit = ((RAM - HugePages) * overcommit_ratio/100) + Swap
```

```bash
$ cat /proc/sys/vm/overcommit_ratio
50  # デフォルト: 50%

# あるいは絶対値の上限を設定する ([added in kernel 3.14](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=49f0ce5f9232)):
$ echo 8388608 > /proc/sys/vm/overcommit_kbytes  # 8GB

# デフォルトの比率で、RAM 8GB + スワップ2GBのシステムの場合:
# Commit limit = (8GB * 0.5) + 2GB = 6GB
```

現在の状態を確認する。
```bash
$ cat /proc/meminfo | grep Commit
CommitLimit:    6291456 kB
Committed_AS:   2145678 kB
```

## オーバーコミットが破綻したとき: OOM killer {#when-overcommit-fails-the-oom-killer}

プログラムが利用可能な量を超えるメモリを実際に使うと、誰かが死ななければならない。

```
Physical RAM exhausted
        │
        ▼
┌───────────────────┐
│    OOM Killer     │
│                   │
│ Scores processes: │
│ - RSS + swap      │
│ - Page tables     │
│ - oom_score_adj   │
│                   │
│ Kills highest     │
│ scoring process   │
└───────────────────┘
```

OOM killerは次の要素に基づいて犠牲者を選ぶ（[mm/oom_kill.c:oom_badness()](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/oom_kill.c)）。

- **`RSS`** - 常駐メモリの使用量
- **スワップエントリ** - スワップアウトされたメモリ
- **ページテーブル** - ページテーブルに使われているメモリ
- **oom_score_adj** - ユーザーが調整可能な補正値（-1000〜1000）

以前のカーネルはプロセスの経過時間や子プロセスも考慮していたが、これらのヒューリスティックは信頼性が低いとして[2019年に削除された](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=bbbe48029720)。その根拠は[LKMLでの議論](https://lore.kernel.org/lkml/20190225065854.30027-1-shakeelb@google.com/)を参照。

```bash
# プロセスのOOMスコアを見る
$ cat /proc/<pid>/oom_score
150

# 重要なプロセスを保護する
$ echo -500 > /proc/<pid>/oom_score_adj
```

## オーバーコミットを単に無効にすればよいのでは? {#why-not-just-disable-overcommit}

厳格モード（モード2）のほうが安全に聞こえる。なぜデフォルトではないのか。

**問題1: `fork()`が高くつくようになる**

オーバーコミットがなければ、すぐ後に`exec()`が続く場合でも、すべての`fork()`が潜在的なCOWコピーのためにメモリを予約しなければならない。

**問題2: 疎なデータ構造が失敗する**

```c
// 疎な配列: 1GBのアドレス空間を予約し、10MBを使う
void *sparse = mmap(NULL, 1UL << 30,  // 1GB
                    PROT_READ | PROT_WRITE,
                    MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
// 実際に書き込まれたページだけが物理RAMを消費する
// オーバーコミットなし: 割り当てが失敗するか、1GBを予約する
// オーバーコミットあり: 問題なく動き、RSSは実際の使用量を反映する
```

**問題3: メモリマップトファイル**

```c
// 10GBのファイルをマップし、100バイトを読む
void *p = mmap(NULL, 10UL*1024*1024*1024,
               PROT_READ, MAP_PRIVATE, fd, 0);
// オーバーコミットなし: 失敗するかもしれない
// オーバーコミットあり: アクセスされたページだけがロードされる
```

**問題4: JVMや同様のランタイム**

```bash
# JVMは起動時に最大ヒープを予約する
java -Xmx8g -jar app.jar
# 実際の使用量は500MBかもしれない
```

## トレードオフ {#the-trade-off}

| アプローチ | 長所 | 短所 |
|----------|------|------|
| オーバーコミット（デフォルト） | より多くのプログラムが動く、`fork()`が機能する、疎な構造が使える | OOM killerが発動しうる |
| 厳格（モード2） | 失敗が予測可能 | メモリを無駄にする、`fork()`が重い、疎な使い方が破綻する |

## 歴史的背景 {#historical-context}

### Linux以前: 伝統的なUnix {#before-linux-traditional-unix}

初期のUnixシステム（1970〜80年代）は、一般に**厳格なメモリ計上**を使っていた。`malloc()`を呼ぶと、システムは物理メモリ（またはスワップ）が利用可能かどうかを確認した。これは単純だったが無駄が多かった。あるプロセスが予約したが使っていないメモリを、他のプロセスが使えなかったからである。

### Linuxの積極的なアプローチ（1990年代） {#linuxs-aggressive-approach-1990s}

Linuxは当初から別の道を選んだ。Linus Torvaldsは**楽観的な割り当て**を好んだ。プログラムには自由に割り当てさせ、不足は実際に起きたときに対処する。これはプログラムの実際の振る舞い、つまり使う以上に割り当てることが多いという性質に合っていた。

これが理にかなっていた理由:
1. **メモリは高価だった** - 希少なRAMの利用率を最大化する
2. **`fork()`/`exec()`パターン** - Unixで支配的なパターンであり、COW + オーバーコミットから莫大な恩恵を受ける
3. **疎な使い方は一般的** - 実際のプログラムは割り当てたメモリのすべてに触れるわけではない

### OOM killer（1998年） {#the-oom-killer-1998}

楽観的な割り当ての避けられない帰結として、賭けが外れることもある。Rik van Rielは、これに対処するために[**1998年8月**](https://lwn.net/1998/0903/a/oom.html)にOOM killerを作った（Linux 2.1.x開発シリーズ。[`mm/oom_kill.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/oom_kill.c)を参照）。

> *"Copyright (C) 1998,2000 Rik van Riel"*
> *"Thanks go out to Claus Fischer for some serious inspiration and for goading me into coding this file."*

OOM killerは最初から議論の的だった。しかし、代替案はもっと悪い。

- **(a)** `ENOMEM`を返し、どのみちプログラムをクラッシュさせる
- **(b)** 決してオーバーコミットせず、膨大な量のメモリを無駄にする
- **(c)** 何かをkillしてシステムを回復させる

選択肢(c)が最も悪くない。それがエンジニアリングの現実である。

### オーバーコミット制御の変遷 {#evolution-of-overcommit-controls}

- **Linux 2.4**: `vm.overcommit_memory` sysctl（モード0、1、2）が正式化された
- **Linux 2.6**: 厳格モードの調整用に`vm.overcommit_ratio`が追加された
- **Linux 3.14**: ratioの代替として`vm.overcommit_kbytes`が追加された（[commit 49f0ce5f](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=49f0ce5f9232)）
- **2010年**: OOM killerが[David Rientjes（Google）によって書き直された](https://lwn.net/Articles/391222/)
- **Linux 4.6（2016年）**: OOM reaperが導入された（[commit aac4536355](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=aac453635549)） - OOMの犠牲者からより速くメモリを回収する
- **Linux 4.19（2018年）**: `cgroup`対応のOOM killer（[commit 3d8b38eb81](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3d8b38eb81ca)） - `cgroup`全体を1単位としてkillできる

## 推奨事項 {#recommendations}

**ほとんどのワークロード:** デフォルト（モード0）のままにする

**データベース/重要なサービス:**
```bash
# OOM killerから保護する
echo -1000 > /proc/<pid>/oom_score_adj
```

**コンテナ:**

- メモリcgroupを使って制限・分離する
- OOMの処理はコンテナランタイムに任せる

**厳格な環境:**
```bash
# 慎重に選んだ比率でモード2を使う
echo 2 > /proc/sys/vm/overcommit_memory
echo 80 > /proc/sys/vm/overcommit_ratio
```

## 試してみる {#try-it-yourself}

```bash
# 現在のモードを確認
cat /proc/sys/vm/overcommit_memory

# コミット上限と使用量を見る
grep -i commit /proc/meminfo

# OOMスコアを見る
for pid in /proc/[0-9]*; do
    echo "$(cat $pid/oom_score 2>/dev/null) $(cat $pid/comm 2>/dev/null)"
done | sort -rn | head -10

# OOMイベントを監視
dmesg -w | grep -i oom
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [mm/mmap.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/mmap.c) — `mmap()`と`brk()`の際に`__vm_enough_memory()`を呼び出し、オーバーコミットのポリシーを強制する
- [mm/util.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/util.c) — `__vm_enough_memory()`のヒューリスティック（モード0）と`vm_commit_limit()`の計算（モード2）
- [mm/oom_kill.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/oom_kill.c) — `oom_badness()`：犠牲者がどのようにスコア付けされ選ばれるか

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/mm/overcommit-accounting.rst`](https://docs.kernel.org/mm/overcommit-accounting.html) — 3つのモードとコミット上限の式についての公式の説明
- [`Documentation/admin-guide/sysctl/vm.rst`](https://docs.kernel.org/admin-guide/sysctl/vm.html#overcommit-memory) — sysctlのノブ：`overcommit_memory`、`overcommit_ratio`、`overcommit_kbytes`

### 関連ページ {#related-pages}

- [oom.md](oom.md) — OOM killerのスコア付け、`oom_score_adj`、cgroup対応のOOM
- [mmap.md](mmap.md) — VMAの作成とデマンドページング：なぜアドレス空間は安価なのか

### LWNの記事 {#lwn-articles}

- [LWN: Toward better OOM handling](https://lwn.net/Articles/391222/) — 2010年のGoogleでのDavid RientjesによるOOM killerの書き直し
- [LWN: The OOM killer](https://lwn.net/Articles/317814/) — 犠牲者選択のヒューリスティックの詳細な解説
- [LWN: Smarter out-of-memory handling](https://lwn.net/Articles/761118/) — cgroup対応のOOMとOOM reaper
