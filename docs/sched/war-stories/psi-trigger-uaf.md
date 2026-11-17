# PSIポーリング: 所有者のcgroupより長生きした待ちキュー {#psi-polling-a-waitqueue-that-outlived-the-cgroup-that-owned-it}

> CVE-2023-52707 — プレッシャー監視用の待ちキューはcgroupが削除された瞬間に解放されていたが、プレッシャーファイルをまだポーリングしているスレッドがそれへの参照を保持しており、終了時に解放済みメモリを参照外しする可能性があった

修正
:   コミット `c2dbe32d5db5`（"sched/psi: Fix use-after-free in ep_remove_wait_queue()"）、メインラインLinux 6.2（2023年2月）

発見・修正
:   Munehisa Kamata（Amazon）

バグの存在期間
:   Linux 5.2以降、コミット `0e94682b73bf` による

*[事件簿: スケジューラのバグ](../war-stories.md)の一部。*

## 事前の状態 {#before-state}

Pressure Stall Information（PSI）では、プロセスがcgroupの `cpu.pressure`、`memory.pressure`、`io.pressure` ファイルにトリガーを登録し、プレッシャーがしきい値を超えたときの通知を受けるためにそれをpoll（または `epoll`）できる。各トリガーは待ちキューを所有しており、`epoll` の内部管理構造（`struct eventpoll`）は開いたファイルを介してそこにアタッチする。これは通常の `epoll` のライフサイクルに従う。つまり待ちキューは、いずれかの `epoll` インスタンスがそのファイルを参照している限り有効であり続けることが期待され、そのファイルが最終的にクローズされる過程で切り離され解放される。

## きっかけ {#the-trigger}

スレッドがまだプレッシャーファイルの1つをポーリングしている間に非ルートcgroupを削除すると、ファイルを通常どおりクローズする場合とは異なる破棄パスを通る。

```
do_rmdir
  cgroup_rmdir
    kernfs_drain_open_files
      cgroup_file_release
        cgroup_pressure_release
          psi_trigger_destroy
```

`psi_trigger_destroy()` は、cgroupの破棄の一環としてトリガーの待ちキューを解放する。どこかのプロセスがまだプレッシャーファイルを開いて `epoll` に登録しているかどうかは関係ない。ポーリングしているスレッドのファイルディスクリプタはまだ有効であり、その `epoll` 登録もまだ生きている。ただ、今や解放済みメモリを指しているだけである。

## 観測された挙動 {#observed-behavior}

後でポーリングスレッドがファイルをクローズするか、終了してファイルディスクリプタが後始末されると、`epoll` 自身の解放パスが実行される。

```
fput
  ep_eventpoll_release
    ep_free
      ep_remove_wait_queue
        remove_wait_queue
```

`ep_remove_wait_queue()` は、cgroupの削除時に `cgroup_rmdir()` がすでに解放した待ちキューを参照外しする。これは単純なuse-after-freeであり、非特権プロセスから到達可能である。必要なのは、後で誰か（あるいは何か）によって削除されるcgroupのプレッシャーファイルをポーリングする権限だけである。

## なぜ起きたか {#why-it-happened}

修正コミットが直接述べているとおり、根本的な問題は、`cgroup_file_release()`（ひいてはプレッシャートリガーの待ちキューの寿命）が、背後にあるファイルの実際の寿命に結び付けられていなかったことである。`epoll` の契約は、開いているファイルディスクリプタに対して登録された待ちキューは、そのディスクリプタ自身の `struct file` が解放されるまで有効であり続けると仮定している。PSIのプレッシャートリガーの待ちキューは代わりにcgroupの削除ライフサイクルに従っていた。これはまったく別の、より早いイベントであり、`epoll` にはそれを知る方法も待つ方法もなかった。

## 解決 {#resolution}

正しい修正、つまり `cgroup_file_release()` のタイミングをファイルの本当の寿命に真に一致させることは、cgroupまたはkernfsレイヤーで自明でないリファクタリングを必要としただろう。代わりに修正は `wake_up_pollfree()` を使う。これはまさにこの種の問題のために追加されたヘルパーである（導入コミット `42288cb44c4b` による）。待ちキューを解放する前に `psi_trigger_destroy()` からこれを呼ぶことで、まだ登録されている `epoll` のウォッチャーはcgroup削除時に起こされて自らをきれいに切り離す。後でクローズ時に発見されるダングリング参照を保持したまま放置されることはなくなる。

## 教訓 {#what-it-taught-us}

**別のサブシステムのライフサイクル契約に参加するサブシステムは、自身の自然な破棄パスが異なっていても、その契約を正確に守らなければならない。** PSIトリガーはcgroupスコープであり、cgroupがなくなると破棄される。`epoll` の待ちキューはファイルスコープであり、ファイル自体が解放されるまで生き残ることを期待している。両者を正しく橋渡しするには、「cgroupがなくなった」と「ファイルがクローズされた」は本当に異なるイベントであり、どちらの順序でも起こりうると認識する必要がある。先に起こったほうが後者も意味するかのように扱ってはならない。

**専用の脱出口（`wake_up_pollfree()`）は、「きれいな」修正でなくても正しい修正になりうる。** コミットメッセージは、`cgroup_file_release()` を `fput()` に正しく結び付けるほうがより正しい修正であるが、はるかに大きな修正でもあると明言している。これほど到達しやすい問題に対しては、「この待ちキューの寿命はファイルの寿命に結び付いていない」場合の既存パターンを使うほうが、より安全で小さな変更だと判断された。

!!! warning "注意すべきパターン"
    あるカーネルオブジェクト（ここではcgroupが所有するトリガー）が、*別の*オブジェクトの寿命（`struct file`）を前提に作られた汎用の通知機構（ここでは `epoll`）から参照されている場合は常に、通知機構がまだ参照は生きていると信じている間に、所有オブジェクトが帯域外で先に破棄されたら何が起こるかを確認すること。

## 関連項目 {#see-also}

- [スケジューラの概要](../README.md) — プレッシャーストールの計上とPSI
- [cgroup v2](../../cgroups/cgroup-v2.md) — cgroupの削除とファイルのライフサイクル

## 外部参考資料 {#external-references}

- [git.kernel.org: c2dbe32d5db5](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c2dbe32d5db5c4ead121cf86dabd5ab691fb47fe) — "sched/psi: Fix use-after-free in ep_remove_wait_queue()"、メインラインの修正
- [git.kernel.org: security/vulns — CVE-2023-52707](https://git.kernel.org/pub/scm/linux/security/vulns.git/plain/cve/published/2023/CVE-2023-52707.mbox) — カーネルCVEチームの公式アナウンス。各stableブランチにおける影響を受けるバージョンと修正済みバージョンを含む
