# 事件簿: mmのバグ事件の記録 {#war-stories-mm-bug-incident-narratives}

> Linuxメモリマネージャの深い設計上の問題を明らかにした3つの本番環境のバグ。運用者が何かおかしいと気づいた瞬間から、調査、修正、そしてそこから得られた設計上の教訓までを語る

これらはいずれも実際に起きたことである。以下のコミットハッシュは実際の修正を指しており、LKMLのスレッドや本番環境での報告にもリンクしているので、元の議論を自分で読むことができる。

3つのバグは、サブシステムの異なる部分と異なる種類の障害を網羅するように選んだ:

| バグ | 領域 | 障害の種類 | 時期 |
|-----|------|---------------|-----|
| zone_reclaim_mode | NUMA回収ポリシー | 静かな性能破壊 | 2006–2014 |
| THPのcompound pageロック | ページアロケータ / AIOパス | 巻き添えのオーバーヘッド | 2011–2013 |
| RSS percpu_counterの不正確さ | メモリアカウンティング / OOM | 誤った犠牲プロセスの選択 | 2022–2026 |

---

## バグ1: zone_reclaim_mode — デフォルトで性能を破壊したNUMA回収ポリシー {#bug-1-zone_reclaim_mode-the-numa-reclaim-policy-that-destroyed-performance-by-default}

*v2.6.16（2006年）頃に導入。v3.16（2014年）でデフォルト無効化。コミット: [4f9b16a64753](https://git.kernel.org/linus/4f9b16a64753)。*

### 症状 {#the-symptom}

NUMAハードウェア（典型的には2ソケットまたは4ソケットのx86サーバー）上でデータベース、Webサーバー、インメモリキャッシュを動かしていた運用者たちは、アプリケーションが想定よりかなり遅く動作していることに気づいた。症状は一貫しなかった。問題のないサーバーもあれば、同一ハードウェアでも問題のあるサーバーもあった。影響を受けたマシンでは、明白な原因なしに性能が50%以上低下することもあった。

この速度低下は、CPU飽和、ネットワークレイテンシ、ディスクI/Oのいずれとも相関しなかった。メモリには十分な空きがあるように見えた。カーネルはスワップしていなかった。それでもアプリケーションは明らかに待ち時間を費やしていた。

注意深い運用者が `/proc/vmstat` を調べると、`zone_reclaim_failed` のカウントが高くなっていることに気づいただろう。これは、カーネルがローカルNUMAノードからメモリを回収しようとして失敗するたびに増加するカウンタである。回収が失敗していたのは、ローカルに回収できるものが何もなかったからだ。ワークロードのアクティブなページは両方のNUMAノードに分散していたが、`zone_reclaim_mode` はアロケータに対し、もう一方のノードにあふれさせる前に、単一ノードから割り当てを満たそうと繰り返し試みる（そして失敗する）ことを強いていた。

### 調査 {#investigation}

調査は困難だった。アプリケーション側から見ると、速度低下が明らかにメモリ関連だとは分からなかったからである。リモートNUMAメモリにアクセスするプロセスは、すべてのメモリアクセスでより高いレイテンシを被る。これは全体的なスループットの低下（クエリの低速化、レイテンシの増加、IPCの低下）として現れ、トップレベルのツールでは単一のボトルネックが見えない。

多くの運用者にとって最初の手がかりは `numastat` ツールで、メモリアクセスの大部分が非ローカルのNUMAノードに向かっていることを明らかにした:

```bash
numastat -p <pid>
# プロセスのNUMAノードごとの割り当て統計
# ローカル割り当てとリモート割り当ての間に大きな差がある場合、
# zone_reclaim_modeがノード間割り当てを妨げていることを示唆する

cat /proc/vmstat | grep zone_reclaim
# zone_reclaim_failed: <large number>
# 1増えるごとに、カーネルがローカルで回収を試みたが失敗したことを意味する
```

確認すべき場所を知っている者には、設定が見えていた:

```bash
cat /proc/sys/vm/zone_reclaim_mode
# 1   ← 有効。これが犯人
```

これを0に設定すると性能が即座に回復し、診断が確定した。修正はsysctl1つで済んだが、それはその存在を知っている者に限られた。

### 根本原因 {#root-cause}

`zone_reclaim_mode` が導入されたのは、NUMAシステムが主にHPCクラスタで使われていた時代である。それらのマシン上のワークロードは慎重に分割されていた。各NUMAノードは分散計算の別々の一部を実行し、ノード間のメモリアクセスは本当に高コストで避けるべきものだった。あるノードのメモリが逼迫したときは、リモートノードにあふれさせるのではなくローカルのページを回収するのが正しい挙動だった。リモートアクセスのペナルティの方が回収コストより悪かったからである。

有効にすると（`zone_reclaim_mode=1`）、カーネルのページアロケータはリモートNUMAノードにフォールバックする前に、ローカルノードのLRUリストからページを回収しようと試みた。この回収は割り当てパスの中でインラインで実行された。カーネルがローカルノードのページをスキャンして追い出す間、割り当てを行うプロセスは停止していた。

問題は、この前提（ノード間アクセスは常にローカル回収より悪い）が、NUMAハードウェアがHPC以外でも一般的になるにつれて成り立たなくなったことだった。現代の2ソケットのデータベースサーバーでは:

1. ワークロードのホットなページは自然に両方のノードに分散していた。
2. アプリケーションのワーキングセットは1つのノードに収まらなかった。
3. `zone_reclaim_mode` は、リモートノードから割り当てる前に、ローカルノードからアクティブなワーキングセットのページを回収することをアロケータに強いた。
4. 追い出されたページはすぐにフォールトで読み戻され、回収・フォールト・追い出しのループが生じた。
5. 結局リモートノードからの割り当ては（回収の試みの後で）行われたので、回収は無駄な作業であり、しかもワーキングセットを損なった。

最終的にデフォルトを無効化したMel Gormanのコミットメッセージは、これを率直に述べている（[4f9b16a64753](https://git.kernel.org/linus/4f9b16a64753)）:

> *"zone_reclaim_mode causes processes to prefer reclaiming memory from local node instead of spilling over to other nodes. This made sense initially when NUMA machines were almost exclusively HPC and the workload was partitioned into nodes. The NUMA penalties were sufficiently high to justify reclaiming the memory. On current machines and workloads it is often the case that zone_reclaim_mode destroys performance but not all users know how to detect this."*

### 修正 {#the-fix}

即時の修正は実行時に行えた。`vm.zone_reclaim_mode=0` を設定するとローカル優先の回収ポリシーが無効になり、アロケータはリモートノードへ自由にあふれさせられるようになった。影響を受けたシステムでは性能が即座に回復した。

```bash
# zone_reclaim_modeを無効化する（ノード間割り当てが自由に進められるようになる）
sysctl vm.zone_reclaim_mode=0
echo 0 > /proc/sys/vm/zone_reclaim_mode
```

構造的な修正はMel Gormanによるコミット [4f9b16a64753](https://git.kernel.org/linus/4f9b16a64753)（v3.16、2014年6月）で、カーネルのデフォルトを `1` から `0` に変更した。このコミットはJohannes Weiner、Michal Hocko、Christoph Lameterというmmの中核チームからackを受けており、古いデフォルトが大多数のシステムにとって誤りだという幅広い合意があったことを示している。

関連コミット [6b187d0260b6](https://git.kernel.org/linus/6b187d0260b6) は、そもそもzone_reclaimを検討すべき場合のヒューリスティックを改良した。現在では、NUMAノード間距離がしきい値（`RECLAIM_DISTANCE`、通常30）を超える場合にのみ自動的に有効化される。つまり、本当に遠いノード（単に別ソケットにあるだけでなく、相応のインターコネクトを介するもの）では、引き続きこのポリシーの恩恵を受ける。

### 設計上の教訓 {#design-lesson}

**デフォルトの挙動は、歴史的なワークロードではなく大多数のワークロードに合わせなければならない。** zone_reclaim_modeは導入当時には理にかなっていた。それが動くマシンはほぼすべて、ワークロードを分割したHPCクラスタだったからである。NUMAサーバーがどこにでもある時代になると、このデフォルトはほとんどのユーザーにとって誤りになったが、無効化する必要のある人々が無名のsysctlについて知っていなければならなかったため、何年も有効なままだった。

**静かな性能破壊はエラーより悪い。** zone_reclaim_modeは、このオプションに詳しくない運用者がその兆候だと認識できるようなエラー、警告、異常なメトリクスを一切生じなかった。障害の様態は、明白な原因のない数パーセントポイントのスループット低下だった。根本原因を指す明確なシグナルがなく「何かが遅い」とだけ現れる性能バグは、本番環境で何年も残り続けることがある。

!!! note "その後の設計変更"
    この経験の後、NUMAポリシーのデフォルトとドキュメントに改めて注意が向けられた。`zone_reclaim_mode` のカーネルドキュメントは拡充され、ノード間アクセスが本当に高コストになるほどNUMAノード距離が大きい場合を除き、通常は有害であることが記された。`numastat` や `numactl` のツールも改善され、ディストリビューションはデータベースワークロード向けのNUMAチューニングの指針をドキュメントに含めるようになった。

    NUMAハードウェアを運用していて説明のつかない性能劣化が起きているなら、他の何かを調べる前にこの設定を確認すること:
    ```bash
    cat /proc/sys/vm/zone_reclaim_mode
    # ほとんどのワークロードでは0であるべき。0以外なら0に設定してみる
    ```

---

## バグ2: THPのcompound pageロックのオーバーヘッドがhugetlbfsのAIOを壊した {#bug-2-thps-compound-page-locking-overhead-broke-hugetlbfs-aio}

*v2.6.38（2011年）でTHPとともに混入。v3.12（2013年）で修正。コミット: [7cb2ef56e6a8](https://git.kernel.org/linus/7cb2ef56e6a8)。*

### 症状 {#the-symptom_1}

Linux上でI/O性能のベンチマークを取っていたOracleデータベースの管理者たちは、カーネルをアップグレードした後に急激なリグレッションに気づいた。Oracleの `orion` I/Oベンチマークツール（データベースのI/Oパターンでストレージ性能を特性評価するための標準ツール）を使い、フラッシュストレージからhugetlbfsで裏付けられたバッファへの読み込みスループットを測定していた。コミットメッセージにある数値は正確である:

| I/Oブロックサイズ | THP以前 | THP以後（v2.6.39） | THP以後（v3.11-rc5） |
|----------------|---------|---------------------|----------------------|
| 1MB読み込み | 8,384 MB/s | 5,629 MB/s | 6,501 MB/s |
| 64KB読み込み | 7,867 MB/s | 4,576 MB/s | 4,251 MB/s |

1MBのワークロードではスループットが33%低下した。64KBのワークロードでは41%低下した。どちらも後続のリリースで回復しておらず、v3.11-rc5の数値は、THP導入から2年経っても問題が続いていることを示している。

これはTHPのワークロードではなかった。ベンチマークは `SHM_HUGETLB` 付きの `shmget()` を使って従来のhugetlbfsページを割り当てていた。THPより何年も前からある仕組みである。アプリケーションは変わっていなかった。ストレージも変わっていなかった。変わったのはカーネルだけだった。

### 調査 {#investigation_1}

調査はOracleのKhalid Azizが行った。鍵となったツールは `perf top` で、根本原因を即座に明らかにした:

```
# perf top output on the regressed kernel, during AIO benchmark
  40.3%  [kernel]  __get_page_tail
  18.7%  [kernel]  put_compound_page
   8.4%  [kernel]  aio_complete
   ...
```

CPUサイクルの40%超が、2つのページ参照カウント関数 `__get_page_tail()` と `put_compound_page()` で消費されていた。これらの関数は、compound pageに対するすべての `get_page()` と `put_page()` で呼ばれる。AIO層がhugetlbfsページにI/Oを発行するたびに参照を取るため `get_page()` を呼び、I/Oが完了するたびに解放するため `put_page()` を呼んでいた。毎秒数十万回の操作を行う64KBブロックのI/Oベンチマークでは、これらの呼び出しがクリティカルパス上にあった。

問題は、なぜこれらの関数が高コストなのかということだった。THP以前はそうではなかった。

答えは、THPがcompound pageの処理パスに追加したものにあった。THPが導入されたとき、新たなリスクが生まれた。別のスレッドが `get_page()` や `put_page()` の途中にいる間に、透過的ヒュージページが分割されうるのである。これを安全に扱うため、カーネルはcompound pageの参照カウントパスにロックを追加した。具体的には、`get_page_tail()` を呼ぶたびにcompound pageのロックを取得し、並行する分割が参照カウントと競合しないようにした。

このロックは、並行アクセス下で分割されうる透過的ヒュージページには必要だった。しかしそれは**すべての**compound pageに適用された。分割できない従来のhugetlbfsページも含めてである。hugetlbfsページは事前に確保されたプールから割り当てられ、その生涯を通じて固定サイズを持つ。分割操作は存在しない。このロックは純粋なオーバーヘッドだった。

### 根本原因 {#root-cause_1}

THP以前のコードパス: compound pageのtailに対する `get_page()` は、headページの参照カウントをアトミックにインクリメントしていた。高速で、ロックなし。

THP以後のコードパス: `__get_page_tail()` は `PageTail()` をチェックし、次にcompound pageのロックを取得し、ページがまだtailであることを再チェックし（チェックとロックの間に分割が起きた場合に備えて）、それから参照カウントをインクリメントし、ロックを解放した。THPにとっては正しい。hugetlbfsにとってはまったく不要である。

同じパターンは `put_compound_page()` にも当てはまった。compound pageのtailへの参照の解放は、分割が進行中かもしれない場合を扱うために設計されたロックパスを通るようになった。

毎秒数十万リクエストでI/Oを行い、各I/Oがページ参照の取得と解放を行うベンチマークでは、このオーバーヘッドが積み重なった。compound pageのロックがホットスポットになった。`perf top` はそれをはっきり示しており、そこからの診断は単純だった。

### 修正 {#the-fix_1}

Khalid Azizによるコミット [7cb2ef56e6a8](https://git.kernel.org/linus/7cb2ef56e6a8) はv3.12（2013年11月）でマージされ、hugetlbfsページについては分割保護のロックを迂回する高速パスを追加した。hugetlbfsページは `PageHuge()` で識別される。これは割り当て時に設定され、決してクリアされない。compound pageに `PageHuge()` が設定されていれば、それは透過的ヒュージページではありえず、したがって分割されえない。ロックは不要である。

```c
/* 修正ロジックの簡略版 */
static void __get_page_tail(struct page *page)
{
    if (likely(!PageTail(page)))
        return;

    /* hugetlbfsページは分割できない — THPのロックをスキップする */
    if (PageHuge(compound_head(page))) {
        atomic_inc(&compound_head(page)->_count);
        return;
    }

    /* THPページは並行分割から守るためロックが必要 */
    /* ... THP固有のロックパス ... */
}
```

効果は即座に現れた。1MB読み込みのスループットは8,371 MB/sに回復した（THP以前のベースライン8,384 MB/sと比べ、実質的に完全回復）。64KBのスループットは4,251 MB/sから6,510 MB/sへと53%改善したが、依然としてTHP以前のベースラインを下回っており、64KBのパスの別の箇所に追加のオーバーヘッドが残っていることを示していた。

### 設計上の教訓 {#design-lesson_1}

**新機能のための最適化が、無関係なコードパスにオーバーヘッドを加えてはならない。** THPのcompound page分割保護は、透過的ヒュージページにとって必要かつ正しかった。それをすべてのcompound pageに適用したのは、自然ではあるが誤った一般化だった。ロックパスは「十分に速い」と想定されていたが、毎秒数十万回の操作では、操作ごとのわずかなオーバーヘッドでも積み重なって大きなスループットのリグレッションになる。

**説明のつかないCPUバウンドのリグレッションには、`perf top` が最初に使うべきツールである。** このリグレッションは、レイテンシやエラー率の増加としてではなく、CPU消費の増加（同じCPU使用率でのスループット低下）として現れた。`perf top` が `__get_page_tail` をサイクルの40%と示した瞬間、調査は実質的に終わっていた。

**HPCやデータベースのAIOパターンは参照カウントに負荷をかける。** ページを一度割り当てて保持し続けるワークロードと違い、ダイレクトI/Oのパターンはページ参照の取得と解放を絶えず行う（I/O操作ごとに1回）。このパス上のコードはアルゴリズムだけでなく命令レベルで最適化しなければならない。操作ごとに余分なロック取得が1つあるだけで壊滅的になりうる。

!!! note "その後の設計変更"
    このバグを契機に、compound page処理コードが何に適用されるかがより慎重にレビューされるようになった。`PageHuge()`（分割できず、THPのロック不要）とTHPページ（分割でき、ロックが必要）の区別は、compound pageのコードで明示的に考慮される事項になった。その後のfolio抽象化の取り組みは、高速パスを安全に通れるようにする型の区別をさらに形式化した。

    この種のリグレッションを検出するには、hugetlbfsや大きなページで裏付けられたメモリ上でのI/O集約的なワークロード中に、参照カウント関数（`get_page`、`put_page`、`__get_page_tail`、`put_compound_page`、および新しいカーネルではそれらのfolio版）に予想外に多くの時間が費やされていないか、`perf top` で監視すること。

---

## バグ3: RSSのpercpuカウンタの不正確さと誤ったOOM犠牲プロセス {#bug-3-rss-percpu-counter-inaccuracy-and-wrong-oom-victims}

*RSSの変換はv6.2（2023年2月）で導入、コミット [f1a7941243c1](https://git.kernel.org/linus/f1a7941243c1)。不正確さは2026年に修正、コミット [5898aa8f9a0b](https://git.kernel.org/linus/5898aa8f9a0b)。*

### 症状 {#the-symptom_2}

Linux v6.2以降のカーネルにアップグレードした後、多コアのマシンで本番サービスを運用するエンジニアリングチームは、2種類の異常な挙動を目にするようになった。

1つ目: メモリの使いすぎを検出するためにRSSを監視していたモニタリングシステムが、誤って発報するようになった。約100MBのメモリを使うと想定されたサービス（`/proc/<pid>/status` を読むウォッチドッグプロセスが監視していた）が、想定より数十メガバイト多いRSSを報告していた。250 CPUコアのマシンでは、5スレッドのサービスが実際より100MB多くメモリを使っているように見えることがあった。監視上の上限に近いサービスでは、これが不要なアラートや再起動を引き起こした。

2つ目は、より深刻だった。システムが実際にOOM状態に達したとき、カーネルが誤ったプロセスをkillしたのである。（アカウンティングの不正確さのせいで）見かけのRSSがたまたま大きかった低メモリのサービスが、よりメモリを使うサービスが真の圧迫の原因であるにもかかわらず、OOMの犠牲プロセスとして選ばれた。実際のメモリ使用量が最も多いサービスは生き残り、代わりに軽いサービスがkillされた。これにより、OOM状況からの回復も診断も難しくなった。

どちらの症状もv6.2カーネルへのアップグレード後に現れ、v5.xに戻すと消えた。アップグレードが唯一の変更だった。

### 調査 {#investigation_2}

調査はモニタリングの異常から始まった。ウォッチドッグは `/proc/<pid>/status` を読んでいた:

```bash
cat /proc/<pid>/status | grep VmRSS
# VmRSS:    156432 kB   ← 報告された値
# 期待値: ~102400 kB (100MB)
# 差異: ~54MB
```

差異は読み取りごとに変動し、CPUの多いマシンほど大きかった。これは決定的なものではなく統計的なアカウンティング誤差を示唆していた。自然な仮説は、RSSの数え方で何かが変わったというものだった。

v6.2カーネルの変更履歴を調べると、コミット [f1a7941243c1](https://git.kernel.org/linus/f1a7941243c1)（"mm: convert mm's rss stats into percpu_counter"）に直接行き着いた。このコミットはスレッドごとのRSSバッチ処理モデルを変更した。

**v6.2以前**: 各スレッドは私的なRSSカウンタを持ち、そのフラッシュしきい値は `TASK_RSS_EVENTS_THRESH`（64ページ = 256KB）だった。スレッドのカウンタがしきい値を超えると、`mm_struct` ごとのアトミックカウンタにフラッシュされた。任意の時点での最大誤差は `nr_threads × 64 pages` だった。5スレッドのプロセスなら 5 × 64 × 4KB = 1.25MB。小さく、上限がある。

**v6.2以後**: このコミットは、RSSをカーネル標準のCPUごとの近似カウンタ基盤である `percpu_counter` を使うように変換した。`percpu_counter` はCPUごとのバッチを保持し、バッチがあふれるとグローバルな合計にフラッシュする。`percpu_counter` のバッチサイズはCPU数に応じて大きくなる。250 CPUのマシンでは、最大誤差はおよそ `nr_cpus × batch_size × nr_counters` になる。コミットメッセージはこのスケーリングを認めている:

> *"convert the error margin from (nr_threads * 64) to approximately (nr_cpus ^ 2)"*

意図は、高度にマルチスレッドなワークロードにおける `mm_struct` ごとのカウンタでのロック競合を減らすことであり、それは実在の問題だった。しかしそのトレードオフ（誤差がおよそCPU数の2乗で増える）は、RSSの正確さが最も重要な文脈、すなわちOOMの犠牲プロセス選択とメモリ監視において、十分に検討されていなかった。

250 CPUのマシンでは、誤差の幅は上限のあるキロバイト単位から、潜在的にはギガバイト単位にまで拡大した。単一のプロセスのRSSが、実際の常駐セットサイズと数百メガバイト異なって見えることがあった。コミットは、OOM killerやメモリ回収のような読み手は「性能上クリティカルではなく、低速な読み取りでも問題ないはずだ（are not performance critical and should be ok with slow read）」と記していたが、正確な読み取りパスは実装しておらず、それらは高速な（不正確な）カウンタを使ったままになっていた。

### 根本原因 {#root-cause_2}

`percpu_counter` 基盤は、近似値で十分な統計のために設計されている。ページ数、ネットワークパケットのカウンタ、イベントレートなどである。`percpu_counter_read()` 関数は近似値を即座に返す。`percpu_counter_sum()` は正確な値を返すが、すべてのCPUごとのスロットを合計しなければならないので低速である。

v6.2以後、OOM killer全体と `/proc` のRSS報告で使われる `get_mm_counter()` は `percpu_counter_read()` を呼んでいた。これは小さなマシンでの短い報告には十分正確だったが、大きなマシンでは劣化した。スレッドが生涯の間にCPU間を移動し、まだフラッシュされていない古い値をCPUごとのスロットに残していくからである。

`oom_badness()` のOOM killerパスは、犠牲プロセスを選ぶ際に `get_mm_counter()` を使って各プロセスのRSSを読んでいた。250 CPUのサーバーでは:

- プロセスA（真のメモリ大食い）: 実際のRSS 2GB、報告値 1.7GB
- プロセスB（小さなサービス）: 実際のRSS 100MB、報告値 650MB

OOM killerはプロセスBをkillする。プロセスAは動き続け、すぐに再びOOMを引き起こす。システムは無実のサービスをkillし続けるOOMループに入る。

修正コミットで引用されているSweet Tea Dorminyからのバグ報告は、まさにこのシナリオが本番環境で起きたことを記述していた。

### 修正 {#the-fix_2}

コミット [5898aa8f9a0b](https://git.kernel.org/linus/5898aa8f9a0b)（2026年1月）は `get_mm_counter_sum()` を導入した。これは正確さが重要な文脈のために、`percpu_counter_read()` の代わりに `percpu_counter_sum()` を呼ぶ。OOM killerの犠牲プロセス選択、そのコンソールダンプ出力、OOM reaperの進捗報告は、すべて正確な版を使うように変換された。

トレードオフは明示的である。`percpu_counter_sum()` はすべてのCPUごとのスロットを読まなければならず、CPU数に比例した時間がかかる。OOM killerにとってこれは許容できる。OOMイベントはまれであり、正確なRSS値のために数マイクロ秒余分に待つ方が、誤ったプロセスをkillするよりはるかに望ましい。

```c
/* 修正前 — 高速だが、大きなマシンではギガバイト単位でずれる可能性がある */
points += get_mm_counter(p->mm, MM_ANONPAGES) +
          get_mm_counter(p->mm, MM_FILEPAGES) +
          get_mm_counter(p->mm, MM_SHMEMPAGES);

/* 修正後 — 低速だが、肝心なときに正確 */
points += get_mm_counter_sum(p->mm, MM_ANONPAGES) +
          get_mm_counter_sum(p->mm, MM_FILEPAGES) +
          get_mm_counter_sum(p->mm, MM_SHMEMPAGES);
```

この修正は、OOM killが起きたときにカーネルログに出力されるRSS値の同じ不正確さにも対処した。以前は、大きなマシンではログに記録されるRSSが大きく外れることがあり、事後分析を誤らせていた。

### 設計上の教訓 {#design-lesson_2}

**意思決定のパスにおいて、近似カウンタは正確なカウンタと交換可能ではない。** `percpu_counter` 基盤は意図的に近似的であり、それがその目的である。RSSカウンタをそれを使うように変換するのは、正確な値が不要なホットパスにとっては適切な最適化である。誤りは、正確な読み取りパスを提供し、正確さを必要とするすべての呼び出し元がそれを使うようにしなかったことだった。元のコミットのコメント（"OOM killer and memory reclaim are not performance critical and should be ok with slow read"）は正しい懸念を特定していたが、コードには反映されなかった。

**カウンタの誤差の上限は、測定される値との相対で理解しなければならない。** 1GBのRSS測定における1MBの誤差幅は0.1%で、無視できる。同じ誤差の式でも、250 CPUのマシンでは数百メガバイトの誤差を生む。誤差がプロセス間のRSSの差を超えると、OOM killerが使う順位付けは無意味になる。誤差の分析は、開発用ワークステーションではなく、現実のマシンサイズの文脈で行わなければならない。

**誤ったOOM killは連鎖的な障害を引き起こす。** OOM killerが誤ったプロセスをkillすると、真の圧迫の原因からはメモリが解放されない。すぐに別のOOM状態が発生する。新たな犠牲プロセスもまた不正確に選ばれる。システムはOOMループに入り、真の犯人が最終的に選ばれるまでに多くの正常なプロセスをkillすることがある。あるいは、いつまでも選ばれないこともある。

!!! note "その後の設計変更"
    このバグは、RSSカウンタの呼び出し元は近似的な読み取りと正確な読み取りのどちらかを明示的に選ばなければならないという設計ルールを強化した。`get_mm_counter()` / `get_mm_counter_sum()` の分割がこれを明示的にしている。`get_mm_counter()` の新たな呼び出し元は、多コアのシステムでは大きくずれうることを意識的に受け入れなければならない。

    大規模サーバー（100以上のCPUコア）を運用していて、v6.2からおよそv6.24までのカーネルを動かしているなら、OOM kill時にログに記録された `anon-rss` を、killされたプロセスの `/proc/<pid>/smaps_rollup` と比較して、OOM killのログが正確なRSS値を示しているか確認すること。ログに記録された値が期待値と大きく異なる場合、このバグがOOMの犠牲プロセス選択に影響している可能性がある。

    ```bash
    # 修正が入っているかカーネルバージョンを確認する
    uname -r
    # 修正は2026年1月以降にマージされたコミットに含まれる

    # OOM時、カーネルは次のようにログを出す:
    # "Killed process <pid> (<name>) total-vm:X anon-rss:Y file-rss:Z"
    # 100以上のCPUを持つv6.2-v6.24カーネルでは、YとZが大きく誤っている可能性がある
    ```

---

## まとめ: これらのバグに共通すること {#summary-what-these-bugs-have-in-common}

3つのバグはすべて、mmの開発で繰り返し現れる構造的なパターンを共有している:

| パターン | zone_reclaim_mode | THPのcompoundロック | RSS percpuの不正確さ |
|---------|-------------------|----------------------|----------------------|
| **破綻した前提** | NUMAワークロードは分割されている | すべてのcompound pageは分割されうる | 近似カウントはどこでも十分 |
| **影響を受けた対象** | マルチソケットNUMAサーバー | hugetlbfs + AIO（データベース） | 多コアサーバー（100以上のCPU） |
| **検出ツール** | numastat、vmstat | perf top | /proc/status、OOMログ |
| **デフォルト修正までの期間** | 約8年 | 約2年 | 約3年 |
| **修正の種類** | デフォルトを変更 | 高速パスを追加 | 正確な読み取りパスを追加 |

繰り返し現れるテーマは、**ある文脈で正しい最適化が、別の文脈では有害になる**ことであり、その害は何を探すべきか知らなければ見えにくい。3つのバグがいずれも本番環境で何年も生き残ったのは、その兆候がクラッシュやデータ損失ではなく、「説明のつかない性能劣化」や「ときどき起きる誤った挙動」だったからである。

!!! note "mmバグの調査ツールキット"
    これら3つのバグは、標準的なLinuxツールで調査できる:

    ```bash
    # zone_reclaim_mode: NUMA回収の活動を確認する
    cat /proc/sys/vm/zone_reclaim_mode
    cat /proc/vmstat | grep zone_reclaim

    # compound pageのロック: 参照カウントのオーバーヘッドをプロファイルする
    perf top -e cycles:k
    # get_page、put_page、__folio_putの各種が高い割合になっていないか探す

    # RSSの正確さ: 近似RSSと正確なRSSを比較する
    cat /proc/<pid>/status | grep VmRSS       # 近似カウンタを使う
    cat /proc/<pid>/smaps_rollup | grep Rss   # 正確なアカウンティングを強制する
    # 多コアシステムで大きな差異があれば、percpuの不正確さを示す
    ```

## 関連項目 {#see-also}

- [thp.md](thp.md) — THPの設定、defragモード、khugepagedのチューニング
- [oom.md](oom.md) — OOM killerの仕組み、デッドロック、OOM reaper
- [war-stories-regressions.md](war-stories-regressions.md) — mmの変更による性能リグレッション（THP defragのストール、NUMAバランシングのオーバーヘッド、khugepagedのCPUストーム）
- [war-stories-cves.md](war-stories-cves.md) — セキュリティ脆弱性の記録（Dirty COW、StackRot、THP COWの競合）
- [numa.md](numa.md) — NUMAメモリポリシーとzone_reclaimの設計
- [reclaim.md](reclaim.md) — kswapdと直接回収の仕組み

## 参考文献 {#further-reading}

- [OOMログの読み方](reading-oom-log.md) — 実際のOOMイベントのdmesg出力を解析する方法。バグ3で論じたタスク表とスコアのフィールドを含む
- [/proc/vmstatリファレンス](proc-vmstat.md) — `zone_reclaim_failed` など、バグ1の診断に関係するカウンタ
- [データベース向けのメモリチューニング](tuning-databases.md) — バグ1と2の根本原因に対処する、実践的な `vm.zone_reclaim_mode`、ヒュージページ、NUMAインターリーブの設定
- [LWN: NUMA in a hurry](https://lwn.net/Articles/473440/) — NUMAメモリ管理の背景と、ローカル優先の回収がもともと導入された理由
- [LWN: Transparent huge pages](https://lwn.net/Articles/423584/) — バグ2につながったcompound pageモデルを扱う、THPの元の設計
- [`mm/oom_kill.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/oom_kill.c) — `oom_badness()` のスコアリング、犠牲プロセスの選択、バグ3で修正された `get_mm_counter` の呼び出し
- [`mm/vmscan.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) — バグ1のzone_reclaim_modeポリシーを実装する `node_reclaim()`
- [Commit 4f9b16a64753](https://git.kernel.org/linus/4f9b16a64753) — `zone_reclaim_mode` をデフォルトで無効化したMel Gormanのコミット。詳細な根拠付き
- [Commit 7cb2ef56e6a8](https://git.kernel.org/linus/7cb2ef56e6a8) — バグ2を解決した `PageHuge()` 高速パスを追加したKhalid Azizの修正
