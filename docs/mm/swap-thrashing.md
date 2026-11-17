# スワップスラッシングの検出と防止 {#detecting-and-preventing-swap-thrashing}

> システムが有用な仕事よりもスワップに多くの時間を費やすとき

## スワップスラッシングとは何か {#what-is-swap-thrashing}

スワップスラッシングは、システムのワーキングセット（実際に使用中のpage）が利用可能なRAMを超え、カーネルがpageのスワップインとスワップアウトを絶え間なく強いられるときに起きる。ページフォールトのたびにpageを戻すためのディスクI/Oが発生するが、それを読み込むと、すぐに必要になる別のpageが追い出される。システムはアプリケーションのコードを実行する代わりに、ほとんどの時間をRAMとディスクの間でpageを移動させることに費やすため、応答しなくなる。

```
Normal swapping:
  Page out → [idle page] → swap     (page not needed for a while)
  Page in  ← [needed]    ← swap    (occasional, fast)

Thrashing:
  Page A out → swap    (to make room for B)
  Page B out → swap    (to make room for C)
  Page A in  ← swap    (needed again immediately!)
  Page C out → swap    (to make room for A)
  Page B in  ← swap    (needed again immediately!)
  ... cycle continues, system stalls
```

重要な違いは次のとおりである。通常の[スワップ処理](swapping.md)はコールドなpageをまれにディスクへ移す。スラッシングはホットなpageを絶え間なく行き来させる。

## 検出方法 {#how-to-detect-it}

### vmstat: スワップイン/アウトの頻度 {#vmstat-swap-inout-rates}

```bash
$ vmstat 1
procs -----------memory---------- ---swap-- -----io----
 r  b   swpd   free   buff  cache   si   so    bi    bo
 5 12 4194304  12340   1024  56000  8500  9200  8600  9300
 4 14 4194304  11200   1024  55800  9100  8800  9200  9000
```

- **si**（スワップイン）と **so**（スワップアウト）: 持続的に高い値（毎秒数千KB）はスラッシングを示す
- **b** 列: I/O待ちでブロックされた多数のプロセス
- **r** 列: 実行可能なプロセスが積み上がっている

### /proc/vmstatのカウンタ {#procvmstat-counters}

```bash
$ grep -E 'pswpin|pswpout|pgmajfault' /proc/vmstat
pswpin 892451
pswpout 1045823
pgmajfault 934102
```

これらを時間を追って観察する。`pswpin` と `pswpout` がそろって急速に増加していれば、pageが両方向にスワップされていることを意味する。`pgmajfault`（メジャーページフォールト）が多ければ、pageがディスクからフォールトで読み込まれていることが確認できる。

### PSI: Pressure Stall Information（v4.20+） {#psi-pressure-stall-information-v420}

```bash
$ cat /proc/pressure/memory
some avg10=45.00 avg60=38.12 avg300=21.45 total=89234567
full avg10=32.00 avg60=25.88 avg300=14.22 total=62345678
```

- **some**: 少なくとも1つのタスクがメモリ待ちでストールしている時間の割合
- **full**: すべてのタスクがメモリ待ちでストールしている時間の割合

!!! warning
    `full avg10` が上昇していることは、すべてのタスクが同時にメモリ待ちでストールしていることを示す。生産的な作業は何も行われていない。持続的な `full` プレッシャーがあれば調査すべきである。[PSIのドキュメント](psi.md)では、PSIトリガーを使ってこれらのシグナルにプログラムから対処する方法を説明している。

### cgroupごとのメモリプレッシャー {#per-cgroup-memory-pressure}

```bash
$ cat /sys/fs/cgroup/myapp/memory.pressure
some avg10=78.00 avg60=65.20 avg300=42.10 total=45678901
full avg10=55.00 avg60=41.30 avg300=28.50 total=31234567
```

これはプレッシャーを特定のワークロードに切り分けるものであり、`systemd-oomd` はこれを使ってcgroupレベルでスラッシングを検出している。cgroupのメモリ制御について詳しくは[メモリcgroup](memcg.md)を参照。

## なぜ起きるのか {#why-it-happens}

| 原因 | 何が起きているか |
|-------|----------------|
| ワーキングセット > RAM | 最もよくある原因。アプリケーションが物理的に利用可能な量より多くのメモリを必要としている。 |
| `vm.swappiness` が高すぎる | ファイルキャッシュを代わりに回収できる場合でも、カーネルがanonymous pageを積極的にスワップアウトする。[ページ回収](reclaim.md)を参照。 |
| スワップが設定されていない | 逆説的だが、より悪い。[スワップ](swap.md)がないと、システムは段階的に劣化することができず、いきなりOOM killerに進む。 |
| メモリリーク | あるプロセスがゆっくりとすべてのRAMを消費し、他のすべてをスワップへ押し出す。 |
| ソフトリミット付きのcgroupが多すぎる | 複数のcgroupが同じメモリを奪い合い、それぞれが他から回収する。 |

## 対処方法 {#how-to-fix-it}

### 1. RAMを追加するか、ワーキングセットを減らす {#1-add-ram-or-reduce-the-working-set}

ワーキングセットがRAMを超えている場合の本当の解決策は、メモリを追加するか、実行するワークロードを減らす／小さくするかのどちらかしかない。

### 2. vm.swappinessを調整する {#2-tune-vmswappiness}

```bash
# 現在の値を確認する（デフォルトは 60）
$ sysctl vm.swappiness
vm.swappiness = 60

# 下げて、anonymous page のスワップよりファイルキャッシュの回収を優先させる
$ sysctl -w vm.swappiness=10
```

値を下げると、カーネルはanonymous pageのスワップよりもファイルバックドpage（ページキャッシュ）の破棄を優先するようになる。これは、カーネルがスワップに積極的すぎることがスラッシングの原因である場合に役立つ。swappinessが回収の判断にどう影響するかについては[スワップ](swap.md)を参照。

### 3. zswapまたはzramを使う {#3-use-zswap-or-zram}

圧縮スワップはpageをディスクに書き込む代わりに（圧縮して）RAMに保持するため、I/Oを劇的に減らす。

```bash
# zswap（圧縮スワップキャッシュ）を有効にする
$ echo 1 > /sys/module/zswap/parameters/enabled
$ echo lz4 > /sys/module/zswap/parameters/compressor
$ echo 20 > /sys/module/zswap/parameters/max_pool_percent

# または zram（圧縮ブロックデバイス）をスワップとして使う
$ modprobe zram
$ echo lz4 > /sys/block/zram0/comp_algorithm
$ echo 4G > /sys/block/zram0/disksize
$ mkswap /dev/zram0
$ swapon -p 100 /dev/zram0
```

zswap/zramは2〜3倍の圧縮率を達成でき、ディスクI/Oなしに使用可能なメモリを実質的に拡張する。

### 4. cgroupのメモリ制限を設定する {#4-set-cgroup-memory-limits}

ハードリミットは、単一のワークロードがシステム全体のスラッシングを引き起こすのを防ぐ。

```bash
# cgroup v2: ワークロードを 4GB に制限する
$ echo 4G > /sys/fs/cgroup/myapp/memory.max

# 早期の回収を起動させる上限ウォーターマークを設定する
$ echo 3G > /sys/fs/cgroup/myapp/memory.high
```

`memory.high` はハードリミットに達する前にスロットリングと回収を起動し、アプリケーションに背圧をかける。[メモリcgroup](memcg.md)を参照。

## スラッシングによるライブロックの問題 {#the-thrashing-livelock-problem}

カーネルv4.20より前は、システムには大規模なスラッシングを検出する良い仕組みがなかった。カーネルは次のことを行っていた。

1. pageの回収を試みる
2. すぐにまた必要になるpageをスワップアウトする
3. それらをスワップインし、必要な別のpageを追い出す
4. これを際限なく繰り返す

システムは生きてはいるが前に進まない状態、すなわち **ライブロック** に陥る。技術的にはメモリ割り当ては（スワップを介して非常にゆっくりとではあるが）成功しているため、OOM killerは発動しなかった。ユーザーからは、完全に応答しないのに自力では回復しないシステムに見えた。

核心的な問題は、[回収](reclaim.md)の経路に、自分の作業が有用かどうかを測る手段がなかったことである。pageを回収することはできても、まさにそのpageがすぐにフォールトで戻されていることを知ることができなかった。

## 現代の解決策 {#modern-solutions}

### PSIベースのOOM（systemd-oomd） {#psi-based-oom-systemd-oomd}

カーネルv4.20は[Pressure Stall Information（PSI）](https://docs.kernel.org/accounting/psi.html)を導入した。これはリソース競合が実行中のタスクに与える実際の影響を測定する。`systemd-oomd` はPSIを使ってスラッシングを検出し、ワークロードを先回りしてkillする。

```
PSI memory.pressure > threshold for duration
        │
        v
systemd-oomd identifies the cgroup causing pressure
        │
        v
Kills processes in that cgroup before the system locks up
```

これによりライブロックの問題が解決する。割り当てが失敗するのを待つ（スラッシング中には決して起きないかもしれない）のではなく、システムは観測されたストール時間に基づいて行動する。

```bash
# systemd-oomd が動作しているか確認する
$ systemctl status systemd-oomd

# その設定を表示する
$ cat /etc/systemd/oomd.conf
```

### MGLRU: より良いpageのエージング（v6.1+） {#mglru-better-page-aging-v61}

[Multi-Gen LRU（MGLRU）](https://docs.kernel.org/mm/multigen_lru.html)は、従来のactive/inactive LRUを複数の世代で置き換え、より正確にpageの経過時間を追跡する。より良い追い出しの判断を下すことでスラッシングを減らす。カーネルが、すぐに必要になるpageを追い出す可能性が低くなる。

```bash
# MGLRU が有効か確認する
$ cat /sys/kernel/mm/lru_gen/enabled
```

MGLRUは、ワーキングセットが大きくメモリにぎりぎり収まるようなワークロードで特に効果的である。このような場合、従来のLRUはスラッシングを起こすが、MGLRUは最もコールドなpageを正しく識別する。

## 簡易診断チェックリスト {#quick-diagnostic-checklist}

```bash
# 1. システムは今スラッシングしているか？
vmstat 1 5                              # si/so 列を見る

# 2. メモリプレッシャーはどれくらい深刻か？
cat /proc/pressure/memory               # full avg10 > 20% = 問題あり

# 3. 何がメモリを使い切っているか？
ps aux --sort=-%mem | head -10          # メモリ消費の上位
smem -tk                                # プロセスごとの按分メモリ

# 4. スワップはどれくらい使われているか？
swapon --show                           # スワップデバイスと使用量
free -h                                 # スワップの合計/使用/空き

# 5. メジャーフォールトの推移を確認する
grep pgmajfault /proc/vmstat            # 2 回実行して値を比較する

# 6. どの cgroup がプレッシャー下にあるか？（cgroup v2）
find /sys/fs/cgroup -name memory.pressure \
  -exec sh -c 'echo "=== $1 ==="; cat "$1"' _ {} \;
```

## 関連項目 {#see-also}

- [スワップ](swap.md) -- スワップのアーキテクチャと設定
- [スワップ処理の間に何が起きるか](swapping.md) -- pageレベルのスワップイン/アウトのライフサイクル
- [ページ回収](reclaim.md) -- カーネルが何を追い出すかをどう決めるか
- [メモリ不足](oom.md) -- 回収とスワップの両方が失敗したときに何が起きるか
- [メモリcgroup](memcg.md) -- ワークロードのメモリの分離と制限

## 参考文献 {#further-reading}

- [swap](swap.md) — スワップのアーキテクチャ、swappinessの調整、zswapやzramを含むスワップの種類
- [reclaim](reclaim.md) — カーネルが追い出すpageをどう選ぶか、また回収の判断を左右するLRUリスト
- [psi](psi.md) — Pressure Stall Information: メモリのストール時間を測定し、それに対処するためのカーネルインターフェース
- [mglru](mglru.md) — Multi-Gen LRU。従来のactive/inactive LRUをv6.1で置き換え、大きなワーキングセットでのスラッシングを減らす
- [Toward less swapping](https://lwn.net/Articles/401088/)（LWN, 2010） — スワップの先読みと、従来のLRUにおけるスラッシングのコストに関する初期の分析
- [Making MGLRU a better LRU](https://lwn.net/Articles/909280/)（LWN, 2022） — 従来の2リスト方式と比べて、Multi-Gen LRUがpageのエージングをどう改善し、スラッシングを減らすか
- `Documentation/admin-guide/mm/concepts.rst` — ページ回収のモデルと、スワップとメモリプレッシャーの関係に関するカーネルのドキュメント
