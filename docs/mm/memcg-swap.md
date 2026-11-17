# cgroupにおけるスワップの計上 {#swap-accounting-in-cgroups}

> スワップ使用量がcgroupごとにどのように追跡・制限・監視されるか

## なぜcgroupごとのスワップ計上が重要か {#why-per-cgroup-swap-accounting-matters}

スワップの計上がなければ、`memory.max`が1Gのコンテナは2Gの匿名メモリをスワップへページアウトでき、実質的に3Gのメモリ — RAMに1G、ディスクに2G — を使えてしまう。cgroupの制限が迂回されるのである。cgroupごとのスワップ計上は、スワップ使用量を別個に数え、独立した制限を課すことでこの穴を塞ぐ。

cgroup v2は、RAMとスワップを2つの別々のカウンタと2つの別々の制限で追跡する。この分離は意図的なものであり、cgroupが使うRAMの量と消費できるスワップの量を、運用者が独立に制御できるようにしている。

## cgroup v2のスワップインターフェース {#the-cgroup-v2-swap-interface}

### memory.swap.current {#memoryswapcurrent}

`memory.swap.current`は、cgroupの匿名pageが現在スワップ上で占めているバイト数を報告する。**これはスワップのみである** — スワップアウトされ、現在RAMにないpageである。

```bash
cat /sys/fs/cgroup/mycontainer/memory.swap.current
# 209715200   （現在 200MB がスワップ上にある）
```

スワップアウトされた後にスワップインされたpageは、ここではもう数えられない（今はRAMにあり、`memory.current`に数えられる）。スワップにコピーされたがまだRAMに常駐しているpage（スワップキャッシュ）は、`memory.swap.current`ではなく`memory.current`に数えられる。2つのカウンタは互いに重複しない。`memory.current + memory.swap.current`は、RAMとスワップの両方にわたるcgroupの総メモリフットプリントを与える。

```
memory.current        = pages in RAM charged to this cgroup
memory.swap.current   = pages in swap (not currently in RAM) charged to this cgroup
                        ─────────────────────────────────────────────────────────
total footprint       = memory.current + memory.swap.current
```

### memory.swap.max {#memoryswapmax}

`memory.swap.max`はスワップ使用量のハードリミットである。cgroupの`memory.swap.current`がこの値に達すると、そのcgroupのそれ以上のスワップはブロックされる。スワップアウトされるはずだった匿名pageは、代わりに（可能なら）直接回収されるか、OOMを引き起こす。

```bash
# この cgroup のスワップを完全に無効化する
echo 0 > /sys/fs/cgroup/mycontainer/memory.swap.max

# 最大 512MB のスワップを許可する
echo 512M > /sys/fs/cgroup/mycontainer/memory.swap.max

# スワップ無制限（未設定時のデフォルト）
echo max > /sys/fs/cgroup/mycontainer/memory.swap.max
```

`memory.swap.max = 0`のとき、cgroupはスワップが存在しないかのように振る舞う。匿名pageはページアウトできないため、メモリ逼迫はスワップではなく直接OOMにつながる。これは、`--memory-swap`を`--memory`と等しく設定したときに、ほとんどのコンテナオーケストレータが頼りにしている挙動である。

!!! note "memory.swap.maxはスワップpageのみを制限し、memswではない"
    `memory.swap.max = 512M`と設定すると、cgroupは`memory.max`に加えて最大512MBのスワップを使えるという意味になる。合計（RAM + スワップ）が512MBに制限されるという意味ではない。合算の制限が欲しければ、`memory.max`と`memory.swap.max`の両方を設定する。

### memory.swap.high（v5.8+） {#memoryswaphigh-v58}

`memory.swap.high`はスワップ使用量のソフトリミットであり、RAMに対する`memory.high`に相当する。`memory.swap.current`が`memory.swap.high`を超えると、カーネルはcgroup内のプロセスをスロットリングし、スワップの回収を試みる（pageをスワップインしてから解放するか、解放された匿名pageを回収することによって）。

`memory.swap.high`ではプロセスはkillされない。これは`memory.swap.max`に達する前に背圧（バックプレッシャー）を生み出すために存在する。

**導入**: このインターフェースはカーネルv5.8で追加された。動機と設計の議論については[LWN: Cgroup swap high limit](https://lwn.net/Articles/912708/)を参照。

```bash
# スワップが 256MB を超えたらスロットリングし、512MB でハードストップする
echo 256M > /sys/fs/cgroup/mycontainer/memory.swap.high
echo 512M > /sys/fs/cgroup/mycontainer/memory.swap.max
```

カーネルがv6.2より古い場合、`memory.swap.high`は存在しない。次で確認する。

```bash
ls /sys/fs/cgroup/mycontainer/memory.swap.high 2>/dev/null || echo "not available (kernel < 5.8)"
```

### memory.swap.events（v4.18+） {#memoryswapevents-v418}

```bash
cat /sys/fs/cgroup/mycontainer/memory.swap.events
# high 12    <- スワップが memory.swap.high を超えた回数
# max 2      <- スワップが memory.swap.max に達した回数
# fail 0     <- スワップ割り当ての失敗回数
```

このファイルは、他のスワップ計上の改善とともにカーネルv6.5で導入された。古いカーネルではスワップイベントは別個に報告されず、スワップ関連のOOMイベントは`memory.events`に現れる。

## zswap: cgroupごとの圧縮スワップ {#zswap-compressed-swap-per-cgroup}

[zswap](https://docs.kernel.org/admin-guide/mm/zswap.html)は、スワップpageのためのカーネル内圧縮キャッシュである。匿名pageを直接ディスクに書き込む代わりに、zswapはそれらを圧縮してRAMのプールに格納する。zswapに収まらないpageは通常どおりディスクに書き込まれる。これはCPUと引き換えにスワップI/Oを減らすものである。

### memory.zswap.current {#memoryzswapcurrent}

`memory.zswap.current`は、このcgroupがzswapプールで使用している**圧縮後の**バイト数 — 圧縮データが実際に消費しているRAM — を報告する。スワップデータの非圧縮の論理サイズはこれより大きい。`memory.zswap.current`は、プール内の圧縮pageの物理RAMフットプリントを反映する。

```bash
cat /sys/fs/cgroup/mycontainer/memory.zswap.current
# 41943040   （圧縮スワップ page を保持する RAM が 40MB。非圧縮換算では 100MB 以上の可能性）
```

### memory.zswap.max {#memoryzswapmax}

`memory.zswap.max`は、このcgroupがzswapプールをどれだけ使えるかを制限する。cgroupごとのzswap制限がないと、1つのcgroupがzswapプール全体を埋め尽くし、他のcgroupのpageをディスクへ追いやってしまう可能性がある。

**コミット**: [f4840ccfca25](https://git.kernel.org/linus/f4840ccfca25)（"zswap: memcg accounting"）

**作者**: Johannes Weiner（カーネルv6.5）

```bash
# この cgroup の zswap を 200MB に制限する
echo 200M > /sys/fs/cgroup/mycontainer/memory.zswap.max

# この cgroup の zswap を無効化する（ディスクスワップを直接使う）
echo 0 > /sys/fs/cgroup/mycontainer/memory.zswap.max

# zswap 無制限（デフォルト）
echo max > /sys/fs/cgroup/mycontainer/memory.zswap.max
```

!!! note "memory.zswap.maxにはシステム全体でzswapが有効である必要がある"
    zswapが有効でない場合（`cat /sys/module/zswap/parameters/enabled`で確認）、`memory.zswap.current`は常に0であり、`memory.zswap.max`は効果がない。zswapの有効化については[zswapのドキュメント](https://docs.kernel.org/admin-guide/mm/zswap.html)を参照。

## cgroup v1のmemswインターフェース（レガシー） {#the-cgroup-v1-memsw-interface-legacy}

cgroup v1は異なるモデルを使っていた。`memsw`と呼ばれる、**メモリ + スワップ**の合算制限である。関連するファイルは次のとおり。

| ファイル | 意味 |
|------|---------|
| `memory.limit_in_bytes` | RAMのみの制限 |
| `memory.memsw.limit_in_bytes` | RAM + スワップの合算制限 |
| `memory.memsw.usage_in_bytes` | 現在のRAM + スワップの合算使用量 |
| `memory.memsw.max_usage_in_bytes` | 合算使用量のピーク |

### なぜmemswは問題だったか {#why-memsw-was-problematic}

`memsw`モデルには根本的な設計上の問題があった。RAMとは独立にスワップを制限できなかったのである。`memory.limit_in_bytes = 1G`、`memory.memsw.limit_in_bytes = 2G`と設定すると、cgroupは最大1GのRAMと1Gのスワップを使えた。しかし「RAMは最大1G、スワップは最大512MB」と指定することはできなかった — 2つの制限は合算カウンタによって結合されていた。

さらに、合算カウンタには性能上の問題があった。スワップインのたびに`memory.usage_in_bytes`と`memory.memsw.usage_in_bytes`の両方を更新する必要があり、2つの制限の相互作用が微妙な競合を生んでいた。

### なぜcgroup v2はそれらを分離するのか {#why-cgroup-v2-separates-them}

cgroup v2は`memsw`を、別々の`memory.max`と`memory.swap.max`に置き換えた。これにより独立した制御が可能になる。

```bash
# v2: RAM 1GB、スワップ 512MB — 独立した制限
echo 1G   > /sys/fs/cgroup/mycontainer/memory.max
echo 512M > /sys/fs/cgroup/mycontainer/memory.swap.max
```

この分離はセマンティクスもより明確にする。`memory.current`は常にRAMを意味し、`memory.swap.current`は常にディスクスワップを意味する。誤解の元となる合算カウンタはない。

## スワップ制限に達すると何が起きるか {#what-happens-when-the-swap-limit-is-hit}

`memory.swap.current`が`memory.swap.max`に達すると、次のようになる。

1. カーネルはこのcgroupからそれ以上匿名pageをスワップアウトできない。
2. カーネルはcgroup内のファイルバックのpage（ページキャッシュ）の直接回収を試みる — これらはスワップを必要としない。
3. 回収で割り当てを満たすのに十分なメモリを解放できなければ、cgroupのOOM killerが呼び出される。

OOM killのログには通常のcgroup OOM（接頭辞: `Memory cgroup out of memory:`）が表示される。「スワップ制限超過」専用のkillメッセージはない — カーネルから見れば、スワップが使えなかったためにcgroupがメモリ不足になったのである。

```bash
# スワップ制限に達したことを観察する
cat /sys/fs/cgroup/mycontainer/memory.events
# oom 1           <- OOM が発生した（スワップ制限が原因の可能性あり）
# oom_kill 1

# カーネル >= 6.5 なら、スワップイベントを直接確認する
cat /sys/fs/cgroup/mycontainer/memory.swap.events
# max 1           <- swap.max に達した
# fail 0
```

## cgroupごとのスワップ逼迫の監視 {#monitoring-swap-pressure-per-cgroup}

### 基本的な監視 {#basic-monitoring}

```bash
CGROUP=/sys/fs/cgroup/mycontainer

# 現在の RAM とスワップの使用量
echo "RAM:  $(cat $CGROUP/memory.current) bytes"
echo "Swap: $(cat $CGROUP/memory.swap.current) bytes"

# 制限
echo "RAM limit:  $(cat $CGROUP/memory.max)"
echo "Swap limit: $(cat $CGROUP/memory.swap.max)"

# 制限に対する使用率（算術演算が必要）
swap_current=$(cat $CGROUP/memory.swap.current)
swap_max=$(cat $CGROUP/memory.swap.max)
[ "$swap_max" != "max" ] && echo "Swap usage: $((swap_current * 100 / swap_max))%"
```

### memory.statにおけるスワップ活動 {#swap-activity-in-memorystat}

`memory.stat`にはスワップ関連のカウンタが含まれる。

```bash
cat /sys/fs/cgroup/mycontainer/memory.stat | grep -E "^(pswp|workingset)"

# pswpin         <- スワップインされた page
# pswpout        <- スワップアウトされた page
# workingset_refault_anon  <- 回収された後に再びフォールトした匿名 page
```

!!! note "cgroup v2ではpgpgin/pgpgoutは利用できない"
    `pgpgin`と`pgpgout`はcgroup v1のフィールドである。cgroup v2では`mm/memcontrol.c`によって明示的にスキップされる。スワップ活動には`pswpin`/`pswpout`を、より広いI/Oの計上には`iostat`を使う。

`pswpout`が増加しているなら、cgroupは活発にスワップアウトしている。`workingset_refault_anon`が増加しているなら、スワップアウトされたpageがフォールトで戻されている — ワーキングセットがRAMの割り当てに収まっていない、スワップスラッシングの兆候である。

### スワップ逼迫のシグナルとしてのPSI {#psi-as-a-swap-pressure-signal}

メモリPSI（`memory.pressure`）は、RAMとスワップの両方の逼迫によるストール時間を捕捉する。激しくスワップしているcgroupは、プロセスがスワップI/Oを待ってストールするため、PSIの`some`の値が上昇する。

```bash
cat /sys/fs/cgroup/mycontainer/memory.pressure
# some avg10=15.00 avg60=8.00 avg300=3.00 total=234567
# full avg10=5.00  avg60=2.00 avg300=0.80 total=56789
```

`pswpin` + `pswpout`が高く、PSIの`some`が上昇しているなら、cgroupはスワップI/Oにかなりの時間を費やしている。これは通常、ワーキングセットが`memory.max`の割り当てに対して大きすぎることを意味する。

### スワップをリアルタイムで観察する {#watch-for-swap-in-real-time}

```bash
# 1秒ごとにスワップ使用量をポーリングする
while true; do
    swap=$(cat /sys/fs/cgroup/mycontainer/memory.swap.current)
    echo "$(date +%H:%M:%S) swap: $((swap / 1024 / 1024))MB"
    sleep 1
done

# あるいは watch を使う
watch -n 1 "cat /sys/fs/cgroup/mycontainer/memory.swap.current | \
    awk '{printf \"%.1f MB\n\", \$1/1024/1024}'"
```

## DockerとKubernetesにおけるスワップ {#swap-in-docker-and-kubernetes}

### Docker {#docker}

Dockerは2つのフラグでスワップの制御を公開している。

| フラグ | 効果 |
|------|--------|
| `--memory=1g` | `memory.max = 1G`を設定する |
| `--memory-swap=2g` | RAM+スワップの**合算**制限を2Gに設定する（つまりRAM 1G + スワップ 1G） |
| `--memory-swap=-1` | スワップ無制限 |
| `--memory-swap=<value equal to --memory>` | スワップなし（memsw制限 = RAM制限） |

!!! warning "Dockerの--memory-swapのセマンティクスは紛らわしい"
    `--memory-swap`は**合算**制限（RAM + スワップ）であり、スワップのみの制限ではない。`--memory=1g --memory-swap=1g`はスワップなしを意味する。`--memory=1g --memory-swap=2g`は1GBのスワップを意味する。cgroup v2のホスト（Docker 20.10+）では、Dockerはこれを差分を自動計算して`memory.swap.max`（スワップのみの制限）に変換する。フラグの合算セマンティクスは維持されるが、使われるカーネルインターフェースはcgroup v2ネイティブのものである。

```bash
# 1GB のコンテナに 512MB のスワップを許可する
docker run -d --memory=1g --memory-swap=1536m nginx
# memsw 制限 1536MB - メモリ制限 1024MB = スワップ 512MB

# スワップを無効化する
docker run -d --memory=1g --memory-swap=1g nginx
```

### Kubernetes {#kubernetes}

Kubernetesのメモリ制限は`memory.max`（RAM）にのみ対応する。スケジューラはメモリのrequestとlimitがRAMのみを反映すると想定しているため、Kubernetesノードではスワップはデフォルトで無効になっている。

`NodeSwap`フィーチャーゲート（v1.22でアルファ、v1.28でベータ）はKubernetesノードでスワップを有効にし、ノード設定の`swapBehavior`を通じてpodごとのスワップ制御を可能にする。現在の状況については[Kubernetesのスワップに関するドキュメント](https://kubernetes.io/docs/concepts/architecture/nodes/#swap-memory)を参照。

```yaml
# Kubernetes のスワップはオプトインであり、pod 単位ではなくノードレベルで制御される
# pod レベルのスワップ設定は、安定版 Kubernetes ではまだ利用できない
```

## ベストプラクティス {#best-practices}

### スワップを許可するかどうかの判断 {#deciding-whether-to-allow-swap}

| ワークロードの種類 | スワップに関する推奨 |
|---------------|---------------------|
| レイテンシに敏感なもの（データベース、キャッシュ） | 無効化する（`memory.swap.max = 0`）。スワップは予測不能なレイテンシスパイクを引き起こす。 |
| バッチジョブ、データパイプライン | スワップを許可する。一時的なスパイクはOOMを起こす代わりにスワップに吸収される。 |
| Webサーバー、アプリケーションサーバー | 限定的なスワップを許可する（`memory.swap.max` = `memory.max`の25〜50%）。無制限のページングを許さずにバッファを提供する。 |
| メモリリークのあるコンテナ | スワップを無効化する。スワップはリークを覆い隠す — コンテナはkillされる代わりにスワップを使い、問題の検出を難しくする。 |

### スワップ制限に達するのを避けるためにmemory.highを設定する {#set-memoryhigh-to-avoid-reaching-the-swap-limit}

cgroupでスワップが有効なら、cgroupがRAMをすべて消費する前にスロットリングが始まるよう`memory.high`を設定する。これにより、カーネルは匿名pageのスワップに頼る前にページキャッシュを回収する機会を得る。

```bash
echo 800M > /sys/fs/cgroup/mycontainer/memory.high   # ここでスロットリング
echo 1G   > /sys/fs/cgroup/mycontainer/memory.max    # ここで OOM
echo 512M > /sys/fs/cgroup/mycontainer/memory.swap.max  # スワップの予算
```

### zswapを使ってディスクI/Oを減らす {#use-zswap-to-reduce-disk-io}

システムにCPUの余力があり、スワップI/Oがボトルネックになっているなら、zswapを有効にするとディスクに書き込まれるデータ量が減る。

```bash
# zswap をグローバルに有効化する
echo 1 > /sys/module/zswap/parameters/enabled

# cgroup ごとの zswap 使用量を制限する（カーネル >= 6.5）
echo 200M > /sys/fs/cgroup/mycontainer/memory.zswap.max
```

zswapプールのデフォルトサイズはヒューリスティックである（通常は総RAMの20% — `/sys/module/zswap/parameters/max_pool_percent`を確認）。cgroupごとの制限がないと、1つのコンテナがzswapプールを飽和させうる。

## 自分で試す {#try-it-yourself}

### cgroupでのスワップ使用量を観察する {#observe-swap-usage-in-a-cgroup}

```bash
# RAM を制限し、ある程度のスワップを持つ cgroup を作る
sudo mkdir -p /sys/fs/cgroup/swap-test
echo "+memory" | sudo tee /sys/fs/cgroup/cgroup.subtree_control
echo 128M | sudo tee /sys/fs/cgroup/swap-test/memory.max
echo 256M | sudo tee /sys/fs/cgroup/swap-test/memory.swap.max

# 現在のシェルを cgroup に移す
echo $$ | sudo tee /sys/fs/cgroup/swap-test/cgroup.procs

# RAM 制限を超えて割り当てる — OOM ではなくスワップが使われるはず
python3 -c "
import time
x = bytearray(200 * 1024 * 1024)  # 200MB: 128MB は RAM に収まり、残りはスワップへ
print('allocated 200MB')
time.sleep(10)
"

# 別の端末でスワップ使用量を観察する
watch -n 1 cat /sys/fs/cgroup/swap-test/memory.swap.current
```

### スワップ制限にも達したときのOOMを観察する {#observe-oom-when-swap-limit-is-also-hit}

```bash
sudo mkdir -p /sys/fs/cgroup/swap-oom-test
echo 64M  | sudo tee /sys/fs/cgroup/swap-oom-test/memory.max
echo 64M  | sudo tee /sys/fs/cgroup/swap-oom-test/memory.swap.max  # no swap
echo $$ | sudo tee /sys/fs/cgroup/swap-oom-test/cgroup.procs

# スワップが無効なので、これは OOM になる
python3 -c "x = bytearray(100 * 1024 * 1024)"
```

## 主要なソースファイル {#key-source-files}

| ファイル | 関連するコード |
|------|---------------|
| [`mm/memcontrol.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c) | スワップのチャージ／アンチャージ: `__mem_cgroup_try_charge_swap()`、`__mem_cgroup_uncharge_swap()`、`mem_cgroup_swapin_charge_folio()`。スワップ制限の強制。`memory.swap.current`と`memory.swap.max`の読み書き |
| [`include/linux/memcontrol.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/memcontrol.h) | `struct mem_cgroup`: `swap`、`memsw`、スワップカウンタのフィールド |
| [`mm/swap_state.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/swap_state.c) | スワップキャッシュの管理。memcgのチャージと相互作用する |
| [`mm/zswap.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/zswap.c) | memcgごとの計上を含むzswapの実装 |

## 歴史 {#history}

### cgroup v1 memsw（v2.6.34、2010年） {#cgroup-v1-memsw-v2634-2010}

メモリ+スワップの合算（`memsw`）インターフェースはcgroup v1で導入された。cgroupごとにスワップを制限する唯一の仕組みを提供したが、合算制限モデルは混乱を招き、実装には結合した2つのカウンタを維持することによる性能オーバーヘッドがあった。

### cgroup v2の分離されたスワップ計上 {#cgroup-v2-separate-swap-accounting}

cgroup v2は`memory.max`（RAM）を`memory.swap.max`（スワップのみ）から分離し、`memsw`のセマンティクス上の混乱を解消した。この分離は、v4.5で本番利用可能な水準に達したcgroup v2メモリコントローラの、より広範な再設計の一部であった。[LWN: The unified cgroup hierarchy](https://lwn.net/Articles/679786/)を参照。

### memory.swap.high（v5.8、2020年） {#memoryswaphigh-v58-2020}

**LWN**: [Cgroup swap high limit](https://lwn.net/Articles/912708/)

RAMに対する`memory.high`に相当するソフトなスワップ制限を追加し、RAMと同様にスワップについてもOOM前にスロットリングする挙動を可能にした。

### cgroupごとのzswap計上（v5.19、2022年） {#per-cgroup-zswap-accounting-v519-2022}

**コミット**: [f4840ccfca25](https://git.kernel.org/linus/f4840ccfca25)（"zswap: memcg accounting"）

**作者**: Johannes Weiner

`memory.zswap.current`と`memory.zswap.max`を有効にし、単一のcgroupがzswapプールを独占して他のcgroupのpageをディスクに書き出させることを防いだ。

## 参考資料 {#references}

### 主要なコード {#key-code}

| ファイル | 説明 |
|------|-------------|
| [`mm/memcontrol.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c) | スワップのチャージ／アンチャージ、スワップ制限の強制 |
| [`mm/zswap.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/zswap.c) | zswapの実装とmemcgの計上 |
| [`include/linux/memcontrol.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/memcontrol.h) | memcg構造体とスワップカウンタのフィールド |

### カーネルドキュメント {#kernel-documentation}

- [`Documentation/admin-guide/cgroup-v2.rst`](https://docs.kernel.org/admin-guide/cgroup-v2.html) — メモリインターフェースファイル、スワップの計上
- [`Documentation/admin-guide/mm/zswap.rst`](https://docs.kernel.org/admin-guide/mm/zswap.html) — zswapのドキュメント

### LWNの記事 {#lwn-articles}

- [The unified cgroup hierarchy](https://lwn.net/Articles/679786/) — スワップの分離を含むcgroup v2の設計
- [Cgroup swap high limit](https://lwn.net/Articles/912708/) — memory.swap.highの動機と設計

### 関連項目 {#related}

- [メモリcgroup](memcg.md) — memcgの基礎
- [コンテナ向けのメモリチューニング](tuning-containers.md) — 実践的なコンテナのメモリチューニング
- [cgroup OOMコントローラ](memcg-oom.md) — スワップ制限がOOMの一因となったときに何が起きるか
- [階層的なメモリ制限](memcg-hierarchy.md) — 制限がcgroupツリーをどのように伝播するか

## 参考文献 {#further-reading}

- [mm/memcontrol.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c) — スワップのチャージ／アンチャージ経路である`__mem_cgroup_try_charge_swap()`と`__mem_cgroup_uncharge_swap()`。`memory.swap.current`と`memory.swap.max`の読み書きハンドラ
- [mm/zswap.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/zswap.c) — コミット`f4840ccfca25`で導入されたmemcgごとのプール計上を含むzswapの実装
- `Documentation/admin-guide/cgroup-v2.rst` — `memory.swap.*`と`memory.zswap.*`のインターフェースファイルのセマンティクス。重複しないカウンタのモデル（`memory.current + memory.swap.current = total footprint`）を含む
- `Documentation/admin-guide/mm/zswap.rst` — zswapのアーキテクチャ、有効化の手順、プールの設定
- [LWN: The unified cgroup hierarchy in 4.5](https://lwn.net/Articles/679786/) — cgroup v2の設計の背景。`memory.max`と`memory.swap.max`がv1の合算`memsw`モデルから分離された理由を含む
- [LWN: Cgroup swap high limit](https://lwn.net/Articles/912708/) — `memory.swap.max`がOOMを引き起こす前にスロットリングするソフトなスワップ制限、`memory.swap.high`の動機と設計
- [memcg](memcg.md) — memcgの基礎: `memory.max`、`memory.high`、基本的なcgroup操作
- [memcg-oom](memcg-oom.md) — スワップ制限がkillの直接の原因であるときのOOM killログの見え方
- [reclaim](reclaim.md) — メモリ逼迫時に、カーネルが匿名pageをスワップ経由で、ファイルpageをライトバック経由でどのように回収するか
