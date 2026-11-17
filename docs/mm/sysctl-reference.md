# メモリ関連sysctlチューニングリファレンス {#memory-sysctl-tuning-reference}

`/proc/sys/vm/` 以下にあるメモリ関連のすべてのsysctlの簡潔なリファレンス。各エントリは[カーネルのsysctlドキュメント](https://docs.kernel.org/admin-guide/sysctl/vm.html)にリンクしている。

どの値も実行時に読み書きできる。

```bash
# 読む
sysctl vm.swappiness
# 書く（永続化されない）
sysctl -w vm.swappiness=10
# 永続化（再起動後も残る）
echo "vm.swappiness = 10" >> /etc/sysctl.d/99-tuning.conf
sysctl --system
```

---

## 回収 {#reclaim}

カーネルがどれだけ積極的にメモリを回収するか、また空きページのウォーターマークをどう管理するかを制御する。

| sysctl | デフォルト | 説明 | 変更すべきとき |
|--------|---------|-------------|----------------|
| [`vm.swappiness`](https://docs.kernel.org/admin-guide/sysctl/vm.html#swappiness) | 60 | anonymous pageの回収とページキャッシュの回収の重み付け（0-200） | スワップよりキャッシュの破棄を好むレイテンシ重視のワークロードでは下げる。メモリをオーバーコミットしているホストでは上げる |
| [`vm.min_free_kbytes`](https://docs.kernel.org/admin-guide/sysctl/vm.html#min-free-kbytes) | 環境による（RAMに応じて変わる） | カーネルが予約しておく空きメモリの最小KB数 | 直接回収によるストールが発生するシステムや、バースト的な割り当てを必要とするシステム（例: 高速ネットワーク）では上げる |
| [`vm.watermark_scale_factor`](https://docs.kernel.org/admin-guide/sysctl/vm.html#watermark-scale-factor) | 10 | min/low/highウォーターマーク間の間隔。ゾーンサイズに対する割合（単位は0.01%） | 上げるとkswapdが早く起動し、直接回収が減る。バースト的な割り当てパターンに有用 |
| [`vm.watermark_boost_factor`](https://docs.kernel.org/admin-guide/sysctl/vm.html#watermark-boost-factor) | 15000 | メモリ断片化イベント後の一時的なウォーターマーク引き上げ（単位は0.01%） | 引き上げられた回収が不要なスワップを引き起こすシステムでは、下げるか0にする |
| [`vm.vfs_cache_pressure`](https://docs.kernel.org/admin-guide/sysctl/vm.html#vfs-cache-pressure) | 100 | ページキャッシュに対してdentry/inodeキャッシュを回収する傾向（0 = 回収しない、100 = 公平、>100 = 積極的） | 小さなファイルが多いファイルサーバーのワークロードでは下げる。dentry/inodeキャッシュがメモリを消費しすぎる場合は上げる |

---

## dirtyページ {#dirty-pages}

dirty（変更済み）なページキャッシュを、いつ、どのくらいの頻度でディスクにライトバックするかを制御する。

| sysctl | デフォルト | 説明 | 変更すべきとき |
|--------|---------|-------------|----------------|
| [`vm.dirty_ratio`](https://docs.kernel.org/admin-guide/sysctl/vm.html#dirty-ratio) | 20 | 書き込み中のプロセスが強制的にライトバックを開始させられる、総メモリに対する割合 | レイテンシ重視のI/Oでは下げる。高速ストレージでスループットを得たい場合は上げる |
| [`vm.dirty_background_ratio`](https://docs.kernel.org/admin-guide/sysctl/vm.html#dirty-background-ratio) | 10 | バックグラウンドのライトバック（デバイスごとのライトバックスレッド）が始まる、総メモリに対する割合 | 下げるとメモリ上のdirtyデータが減る（電源喪失時により安全）。高速ストレージでは上げて書き込みをより多くバッチ化する |
| [`vm.dirty_expire_centisecs`](https://docs.kernel.org/admin-guide/sysctl/vm.html#dirty-expire-centisecs) | 3000 | dirtyデータがライトバックの対象になる経過時間（1/100秒単位） | データの安全性を高めるには下げる。書き込みの結合を増やすには上げる |
| [`vm.dirty_writeback_centisecs`](https://docs.kernel.org/admin-guide/sysctl/vm.html#dirty-writeback-centisecs) | 500 | ライトバックスレッドが起床する間隔（1/100秒単位） | より頻繁にフラッシュするには下げる。バッテリー駆動のデバイスでは上げる（または0にして無効化する）ことでディスクの起動を減らす |

!!! tip
    RAMに応じて変化しない絶対的な上限が必要な場合は、ratio系の代わりに `dirty_bytes` / `dirty_background_bytes` を使う。

---

## オーバーコミット {#overcommit}

カーネルのメモリオーバーコミットのポリシーを制御する。[メモリのオーバーコミット](overcommit.md)も参照。

| sysctl | デフォルト | 説明 | 変更すべきとき |
|--------|---------|-------------|----------------|
| [`vm.overcommit_memory`](https://docs.kernel.org/admin-guide/sysctl/vm.html#overcommit-memory) | 0 | ポリシー: 0 = ヒューリスティック、1 = 常に許可、2 = 厳格な制限 | OOM killが許容できないシステム（データベース、組み込み）では2にする。特定のHPCやコンテナ構成では1にする |
| [`vm.overcommit_ratio`](https://docs.kernel.org/admin-guide/sysctl/vm.html#overcommit-ratio) | 50 | `overcommit_memory=2` の場合: コミット上限 = swap + RAM * ratio / 100 | アプリケーションが正当により多くの仮想メモリを必要とする場合に上げる。モード2でのみ有効 |
| [`vm.overcommit_kbytes`](https://docs.kernel.org/admin-guide/sysctl/vm.html#overcommit-kbytes) | 0 | `overcommit_memory=2` の場合: KB単位の絶対的なオーバーコミット上限（非ゼロならratioより優先される） | バイト単位で正確なコミット上限が必要な場合にratioの代わりに使う |

---

## ヒュージページ {#huge-pages}

静的な（hugetlbfsの）ヒュージページの割り当てを制御する。Transparent Huge Pageについては[THP](thp.md)を参照。

| sysctl | デフォルト | 説明 | 変更すべきとき |
|--------|---------|-------------|----------------|
| [`vm.nr_hugepages`](https://docs.kernel.org/admin-guide/sysctl/vm.html#nr-hugepages) | 0 | 事前に割り当てる永続的なヒュージページの数 | アプリケーション（例: DPDK、大規模データベース）が必要とするヒュージページの数に設定する |
| [`vm.nr_overcommit_hugepages`](https://docs.kernel.org/admin-guide/sysctl/vm.html#nr-overcommit-hugepages) | 0 | `nr_hugepages` を超えてオンデマンドで割り当てられる追加のヒュージページ | アプリケーションがバースト的にヒュージページを必要とする可能性があるが、すべてのメモリを最初から固定したくない場合に設定する |
| [`vm.hugetlb_shm_group`](https://docs.kernel.org/admin-guide/sysctl/vm.html#hugetlb-shm-group) | 0 | ヒュージページを裏付けとするSysV共有メモリセグメントの作成を許可されるGID | ヒュージページの共有メモリを必要とする非特権ユーザー（例: データベースのサービスアカウント）のグループIDに設定する |

---

## NUMA {#numa}

Non-Uniform Memory Accessシステム向けのチューニングノブ。[NUMA](numa.md)も参照。

| sysctl | デフォルト | 説明 | 変更すべきとき |
|--------|---------|-------------|----------------|
| [`vm.zone_reclaim_mode`](https://docs.kernel.org/admin-guide/sysctl/vm.html#zone-reclaim-mode) | 0 | リモートから割り当てる前にカーネルがローカルノードのメモリを回収するかどうかを制御するビットマスク（1 = ゾーン回収を有効化、2 = ライトバック、4 = スワップ） | キャッシュの保持よりローカルメモリのレイテンシが重要な大規模NUMAシステムで有効にする。ほとんどのワークロードでは0のままにする |
| [`vm.numa_stat`](https://docs.kernel.org/admin-guide/sysctl/vm.html#numa-stat) | 1 | `/proc/vmstat` におけるノードごとのNUMA hit/miss/foreign統計を有効にする | これらのカウンタの更新ごとのオーバーヘッドが無視できない大規模NUMAマシンでは無効にする（0にする） |

---

## OOM {#oom}

Out-Of-Memory killerの振る舞いを制御する。[メモリ不足](oom.md)も参照。

| sysctl | デフォルト | 説明 | 変更すべきとき |
|--------|---------|-------------|----------------|
| [`vm.oom_kill_allocating_task`](https://docs.kernel.org/admin-guide/sysctl/vm.html#oom-kill-allocating-task) | 0 | 1にすると、最悪の原因プロセスを探す代わりに、OOMを引き起こしたタスクをkillする | 走査が遅すぎるシステムや、決定的なOOMの振る舞いが欲しい場合に有効にする |
| [`vm.oom_dump_tasks`](https://docs.kernel.org/admin-guide/sysctl/vm.html#oom-dump-tasks) | 1 | OOM時にタスクごとのメモリ情報をカーネルログにダンプする | タスクが数千あり、ダンプ自体が問題を引き起こすシステムでは無効にする |
| [`vm.panic_on_oom`](https://docs.kernel.org/admin-guide/sysctl/vm.html#panic-on-oom) | 0 | OOM killerを起動する代わりにカーネルパニックを起こす（0 = オフ、1 = パニック、2 = cgroupのOOMでもパニック） | 性能が劣化したノードより停止したノードのほうが望ましいクラスタで有効にする（ウォッチドッグやクラスタマネージャがマシンを再起動できる） |

---

## その他 {#miscellaneous}

| sysctl | デフォルト | 説明 | 変更すべきとき |
|--------|---------|-------------|----------------|
| [`vm.max_map_count`](https://docs.kernel.org/admin-guide/sysctl/vm.html#max-map-count) | 65530 | プロセスが持てるVMAの最大数 | 多数のマッピングを持つアプリケーション（例: JVM、Elasticsearch、mmapを多用するデータベース）では上げる |
| [`vm.mmap_min_addr`](https://docs.kernel.org/admin-guide/sysctl/vm.html#mmap-min-addr) | 0（カーネルのデフォルト。ほとんどのディストリビューションは65536に設定） | ユーザー空間プロセスがmmapできる最小の仮想アドレス | レガシーなソフトウェアがアドレス0へのマップを必要とする場合にのみ下げる。NULLポインタ参照に対する保護のため高く保つ |
| [`vm.compact_memory`](https://docs.kernel.org/admin-guide/sysctl/vm.html#compact-memory) | 書き込み専用 | 1を書き込むと、すべてのゾーンで同期的なメモリコンパクションが起動される | 断片化したシステムでヒュージページを割り当てる前に使う。本番環境のホットパスでは避ける |
| [`vm.drop_caches`](https://docs.kernel.org/admin-guide/sysctl/vm.html#drop-caches) | 書き込み専用 | 1を書き込む = ページキャッシュを解放、2 = dentry/inodeを解放、3 = 両方 | ベンチマークやデバッグでのみ使う。カーネルは自力でキャッシュを効果的に管理するため、本番環境では推奨されない |

---

## ちょっとしたヒント {#quick-tips}

- **現在の値を確認する**: `sysctl -a | grep vm.` で、すべての `vm.*` パラメータとその値が一覧できる。
- **効果を監視する**: sysctlを変更した後は、`/proc/vmstat`、`/proc/meminfo`、`/proc/zoneinfo` のゾーンごとの統計を観察する。
- **変更を永続化する**: `/etc/sysctl.conf` を直接編集するのではなく、`/etc/sysctl.d/*.conf` ファイルにチューニングを置く。
- **コンテナを意識する**: コンテナ内では、多くの `vm.*` sysctlはホスト全体に効くものであり、特権アクセスを必要とする。メモリcgroupのノブ（[memcg](memcg.md)を参照）がコンテナごとの同等物である。
