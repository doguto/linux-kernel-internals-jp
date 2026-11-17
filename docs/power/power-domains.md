# 電源ドメインとgenpd {#power-domains-and-genpd}

> 共有電源レールの下にデバイスをまとめる: 汎用電源ドメインフレームワーク

## 問題: 共有電源レール {#the-problem-shared-power-rails}

デスクトップやサーバーのCPUでは、各コアが個別に電源をゲーティングできる。組み込みSoCでは、そう単純であることはまれである。典型的なモバイルSoCでは、GPUブロック、ディスプレイコントローラ、そのディスプレイ用のメモリインターフェースのすべてに、単一のPMICレールから電源を供給していることがある。電力を節約するためにそのレールを切るには、まずそのレール上のすべてのデバイスが静止している必要がある。1つのデバイスでもまだDMAを行っていれば、レールの電源を切るとデータが破損する。

```
SoC power topology (simplified)

  VDD_GPU rail
  ├── GPU core
  ├── GPU MMU
  └── GPU L2 cache

  VDD_DISPLAY rail
  ├── Display controller (DPU)
  ├── DSI host
  └── MIPI PHY

Each rail: single switch, all devices must be idle before it can be cut.
```

genpd以前は、各SoCベンダーが、デバイスのアイドル状態を追跡してレールの電源断を順序立てて行うロジックを重複して実装していた。Linux 3.1（2011年）で導入された**Generic Power Domain（genpd）**フレームワークは、その基盤を一度だけ、決定版として提供する。

## struct generic_pm_domain {#struct-generic_pm_domain}

中心となる構造体（`include/linux/pm_domain.h`）:

```c
/* include/linux/pm_domain.h — 抜粋。実際の構造体には約30個のフィールドがある */
struct generic_pm_domain {
    struct device          dev;          /* ドメイン自身のstruct device */
    struct dev_pm_domain   domain;       /* 埋め込み。dev_pm_opsを公開する */
    struct list_head       gpd_list_node;/* グローバルなgpd_list内のリンク */

    const char            *name;

    enum gpd_status        status;       /* GENPD_STATE_ON または GENPD_STATE_OFF */

    unsigned int           device_count;
    unsigned int           device_id;        /* 一意なデバイスID */
    unsigned int           suspended_count;  /* システムサスペンド用のデバイスカウンタ。runtime PM用ではない */
    unsigned int           prepared_count;
    unsigned int           performance_state; /* 集約された最大パフォーマンス状態 */

    struct dev_power_governor *gov;      /* ガバナが登録されていなければNULL */

    /* コールバック */
    int (*power_off)(struct generic_pm_domain *domain);
    int (*power_on) (struct generic_pm_domain *domain);

    struct list_head       dev_list;      /* デバイスごとのstruct generic_pm_domain_data */
    struct list_head       parent_links;  /* このドメインが親であるリンク（つまりそのサブドメイン） */
    struct list_head       child_links;   /* このドメインが子であるリンク（つまりその親ドメイン） */

    /* ガバナのタイミングデータはgenpd->gd（struct genpd_governor_data）にある */
};
```

`status`は、レールに現在電源が入っているかどうかを反映する。

| 値 | 意味 |
|-------|---------|
| `GENPD_STATE_ON` | レールはオン。このドメインに直接接続されたアクティブなデバイス、または電源の入った子ドメインが必要としている |
| `GENPD_STATE_OFF` | レールはオフ。接続されたデバイスはどれもアクティブでなく、電源を必要とする子ドメインもない |

## 電源ドメインの登録 {#registering-a-power-domain}

```c
/* プラットフォームコードまたはSoCドライバ */
#include <linux/pm_domain.h>

static struct generic_pm_domain gpu_domain = {
    .name      = "gpu-domain",
    .power_off = gpu_domain_power_off,  /* VDD_GPUレールを切る */
    .power_on  = gpu_domain_power_on,   /* VDD_GPUレールを入れる */
};

static int soc_pm_init(void)
{
    /* genpdコアに登録する。第3引数: ドメインは初期状態でオフか?
     * ガバナ引数にNULLを渡すと「デフォルトのガバナを使う」のではなく、
     * ガバナがまったくないことを意味する — その場合genpd_power_off()は
     * power_down_ok()を完全に飛ばし、常に最も浅い電源状態にフォールバックする。
     * QoSを考慮するガバナを得るには&simple_qos_governorを明示的に渡す。 */
    pm_genpd_init(&gpu_domain, &simple_qos_governor, false);
    return 0;
}
```

`pm_genpd_init()`は、ドメインをグローバルな`gpd_list`にリンクし、`genpd->domain.ops`（`struct dev_pm_ops`）にgenpd自身のruntime PMおよびシステムスリープ用のコールバック（`genpd_runtime_suspend()`、`genpd_runtime_resume()`、`genpd_suspend_noirq()`など）を設定する。後でこのドメインに追加されるすべてのデバイスは、`dev_pm_domain_set()`によって`dev->pm_domain`が`&genpd->domain`を指すようになるため、デバイスのバスレベルのコールバックの代わりにこれらのコールバックが実行される。

## ドメインへのデバイスの追加 {#adding-devices-to-a-domain}

```c
/* デバイスのprobe中に呼ばれる。通常はOFのグルーコードまたはプラットフォームコードから */
int ret = pm_genpd_add_device(&gpu_domain, &gpu_dev->dev);
if (ret)
    dev_err(&gpu_dev->dev, "failed to add to power domain: %d\n", ret);
```

この呼び出しの後、`gpu_dev`のRuntime PMのライフサイクルはドメインによって管理される。`gpu_dev`が`pm_runtime_put()`を呼ぶと、genpd層が`runtime_suspend`を横取りし、これが最後のアクティブなデバイスだったかどうかを確認する。そうであれば、`gpu_domain.power_off()`を呼び出す。

Device Treeを使うARM SoCでは、DTSノード内の`power-domains = <&pd_gpu>`というバインディングによって、ドライバコアがprobe中に`dev_pm_domain_attach()`を呼び、それがさらに`genpd_dev_pm_attach()`を呼んでphandleを解析し、自動的に接続を行う。ドライバ側で明示的に呼び出す必要はない。（`of_genpd_add_device()`は、既に`of_phandle_args`を持っていて手動でデバイスを追加したいプラットフォームコード向けの、別の低レベルAPIである。）`&pd_gpu`というphandleは、`of_genpd_add_provider_simple()`または`of_genpd_add_provider_onecell()`で登録された電源ドメインプロバイダを参照する。

## ドメインの依存関係 {#domain-dependencies}

一部のレールには依存関係がある。サブレールは、その親レールもオンでなければオンにできない。genpdはこれをサブドメインのリンクでモデル化する。

```c
/*
 * 宣言: "display_dsi_domain"は、"display_core_domain"に
 * 既に電源が入っていない限り、電源を入れられない。
 */
ret = pm_genpd_add_subdomain(&display_core_domain, &display_dsi_domain);
```

内部的には、各ドメインの`parent_links`リストは自身のサブドメイン（自分が親であるドメイン）へのリンクを保持し、`child_links`は自分がサブドメインとなっているドメインへのリンクを保持する。電源投入時には親が先に電源を入れられ、電源断時には子が先に電源を切られる。

```
Dependency graph:

  display_core_domain (parent)
  └── display_dsi_domain (child)
         └── mipi_phy_domain (child of child)

Power-on order:  display_core → display_dsi → mipi_phy
Power-off order: mipi_phy → display_dsi → display_core
```

## Runtime PMとの統合 {#runtime-pm-integration}

genpdはRuntime PMと透過的に統合されている。`suspended_count`は
**システムサスペンド**用のカウンタであり（上記の`struct
generic_pm_domain`内のコメントを参照）、このruntime PMの判断には一切関与しないことに注意されたい。
実際のチェックは`genpd_power_off()`の中で行われ、`dev_list`を走査して、Runtime
PM自身のデバイスごとの状態を問い合わせる。

```
pm_runtime_put(dev)                [device driver]
        │
        ▼
genpd_runtime_suspend()            [genpd framework, registered as the
        │                           device's dev_pm_ops.runtime_suspend]
        ▼
genpd_power_off(domain)
   for each dev in domain->dev_list:
        if (!pm_runtime_suspended(dev))
                not_suspended++
   if (not_suspended > 1 || (not_suspended == 1 && !one_dev_on))
        return                      /* still devices active — stay on */
   if (genpd->gov && genpd->gov->power_down_ok)
        if (!power_down_ok())  return   /* governor vetoes the power-off */
   if (!genpd->gov)
        genpd->state_idx = 0       /* no governor: hardcode shallowest state */
   domain->power_off()             [platform callback → cuts rail]
```

レジューム時には向きが逆になる。

```
pm_runtime_get_sync(dev)      [device driver]
        │
        ▼
genpd runtime_resume hook
   if (domain->status == GENPD_STATE_OFF)
        domain->power_on()    [platform callback → asserts rail]
   wait for power_on to settle (gpd_timing_data latency)
   dev->runtime_resume()      [device driver callback]
```

## struct gpd_timing_data {#struct-gpd_timing_data}

各デバイスは、予測されるアイドル期間に対してドメインの電源を切る価値があるかどうかをgenpdガバナが判断するのに使う、タイミングのメタデータを持つことができる。

```c
/* include/linux/pm_domain.h */
struct gpd_timing_data {
    s64 suspend_latency_ns;    /* 最後のデバイスのサスペンドからレールオフまでの時間 */
    s64 resume_latency_ns;     /* power_on()呼び出しからデバイスの準備完了までの時間 */
    s64 effective_constraint_ns;
    ktime_t next_wakeup;
    bool constraint_changed;
    bool cached_suspend_ok;
};
```

`simple_qos_governor`の実際の比較（`drivers/pmdomain/governor.c`の`__default_power_down_ok()`内）は、このデバイスごとのタイミングの合計を予測アイドル時間と比べるものではない。代わりに、検討中のドメインの*電源状態*について、`off_on_time_ns = state.power_off_latency_ns + state.power_on_latency_ns`（ドメイン自身のタイミング表である`struct genpd_power_state`から）を計算し、それを各デバイスの`effective_constraint_ns`（QoSから導かれる、そのデバイスが利用不能のまま置かれてよい最大時間）と比較する。いずれかのデバイスの制約が`off_on_time_ns`以下であれば、電源断は拒否される（往復にかかる時間がそのデバイスのレイテンシ予算を破るため）。そうでなければ、残った制約のうち最小のものが、オフのままでいられるドメインの予算`max_off_time_ns`となる。

ドライバは、`domain-idle-state` Device Treeノードの`entry-latency-us`および`exit-latency-us`プロパティ（マイクロ秒単位。カーネルが内部でナノ秒に換算する）によって、状態ごとのタイミングを与えることができる。

## genpdガバナ {#genpd-governors}

ツリー内には3つのガバナが含まれている（`drivers/pmdomain/governor.c`）。

**`simple_qos_governor`**: ほとんどのgenpdバックエンドが明示的に登録する汎用ガバナである。すべてのデバイスがサスペンドされており、かつそれらのデバイス（およびサブドメイン）全体でQoSによって制約される最短のオフ時間が、ドメインの電源断 + 電源投入のレイテンシを依然として上回る場合に、ドメインの電源を切る。デバイスに付与されたresume-latency QoS制約から計算される`effective_constraint_ns`を使う。これは自動的には選択*されない*。`NULL`ガバナで登録されたドメインには、ガバナがまったくない。

**`pm_domain_always_on_gov`**: ドメインの電源を決して切らない。オンのままでなければならないドメイン（例: 常時オンの電源レール）に使われる。

**`pm_domain_cpu_gov`**（`CONFIG_CPU_IDLE`のときのみビルドされる）: CPU/クラスタの電源ドメイン（`GENPD_FLAG_CPU_DOMAIN`）に使われる。滞在時間とレイテンシが予測アイドル期間とCPU PM QoS制約に収まる、最も深いアイドル状態を選ぶ。

ガバナは`pm_genpd_init()`の第2引数として選択され、この関数は単にそれを`genpd->gov`に格納するだけである。デフォルトへのフォールバックはない。`NULL`を渡すと、そのドメインにはガバナが**まったくない**ことになる。その場合`genpd_power_off()`は`power_down_ok()`を完全に飛ばし、電源断を決めるたびに`genpd->state_idx = 0`（最も浅い状態）を固定で設定する。ドメインが複数の電源状態を持つのにガバナが与えられていなければ、`pm_genpd_init()`は`pr_warn("%s: no governor for states\n", ...)`というログまで出力する。上で説明したQoSを考慮する振る舞いを得るには、`&simple_qos_governor`を明示的に渡すこと。

## debugfsによるデバッグ {#debugging-with-debugfs}

genpdコアは、`/sys/kernel/debug/pm_genpd/`以下にドメインごとに1つのディレクトリを作成する。

```bash
# 登録されているすべての電源ドメインを列挙する
ls /sys/kernel/debug/pm_genpd/
# pm_genpd_summary  gpu-domain  display_core  display_dsi  ...

# すべてのドメインとそのデバイスのグローバルな概要:
cat /sys/kernel/debug/pm_genpd/pm_genpd_summary
# domain                          status          children        performance
#     /device                         runtime status                  managed by
# ------------------------------------------------------------------------------
# gpu-domain                      on                              0
#     /devices/platform/soc/gpu       active                      0           SW
# display_core                    off-0                           0
#     /devices/platform/soc/display   suspended                   0           SW

# ドメインごとのディレクトリには次のものが含まれる: current_state、sub_domains、
# idle_states、active_time、total_idle_time、devices
cat /sys/kernel/debug/pm_genpd/gpu-domain/current_state
```

グローバルな`pm_genpd_summary`ファイルは、`drivers/pmdomain/core.c`の`summary_show()`によって合成される。この関数はグローバルな`gpd_list`を走査し、ドメインごとに`genpd_summary_one()`を呼ぶ。各ドメインについて、ドメイン名、`on`/`off-<state_idx>`の状態、集約された`performance_state`を出力し、続いて接続された各デバイスについて1行ずつ、`rtpm_status_str()`によるそのデバイスのruntime PM状態（`active`/`suspending`/`suspended`/`resuming`/`unsupported`/`error`）、デバイス自身のパフォーマンス状態、そして`mode_status_str()`による`SW`/`HW`の表示（そのデバイスの電源状態が現在ソフトウェアによって管理されているのか、ハードウェアの自律制御に任されているのか）を出力する。

## 実際の実装 {#real-world-implementations}

いくつかの実運用のgenpdバックエンドが、このフレームワークをよく示している。

**Qualcomm GDSC（Globally Distributed Switch Controller。`drivers/clk/qcom/gdsc.h`の構造体コメントによる）** — `drivers/clk/qcom/gdsc.c`。各GDSCが1つの電源ドメインを制御する。`gdsc_enable()` / `gdsc_disable()`関数は、`generic_pm_domain`の`power_on`/`power_off`コールバックとして直接登録される。これらは（`gdsc_toggle_logic()`を介して）GDSCのcollapseビットを切り替え、戻る前にドメインが安定するまでハードウェアをポーリングする。

**Samsung Exynosの電源ドメイン** — `drivers/pmdomain/samsung/exynos-pm-domains.c`。`LOCAL_PWR_CFG`レジスタに書き込むことで電源をゲーティングし、PMUからの応答を求めてオフセット`+0x4`の状態レジスタをポーリングする。

**Rockchipの電源ドメイン** — `drivers/pmdomain/rockchip/pm-domains.c`。`regmap`経由でアクセスする電源ドメインコントローラ周辺機器を使う。サブドメインの依存関係はDevice Treeで宣言され、ドライバはDTのトポロジーに基づいて`pm_genpd_add_subdomain()`を呼ぶ。

3つとも同じパターンに従っている。`struct generic_pm_domain`を割り当て、`power_on`/`power_off`コールバックを埋め、`pm_genpd_init()`を呼び、DTプロバイダとして登録する。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/pm_domain.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/pm_domain.h) — `struct generic_pm_domain`、`struct gpd_timing_data`、および`pm_genpd_*`/`of_genpd_*` API
- [drivers/pmdomain/core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pmdomain/core.c) — genpdのコア: `pm_genpd_init()`、`genpd_add_subdomain()`、`genpd_dev_pm_attach()`、debugfsの概要
- [drivers/pmdomain/governor.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/pmdomain/governor.c) — `simple_qos_governor`と`pm_domain_always_on_gov`
- [drivers/clk/qcom/gdsc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/clk/qcom/gdsc.c) — Qualcomm GDSCのgenpdバックエンド: `gdsc_enable()`/`gdsc_disable()`
- [Documentation/devicetree/bindings/power/power-domain.yaml](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/devicetree/bindings/power/power-domain.yaml) — `power-domains`と`#power-domain-cells`のDTバインディング仕様

### 関連ページ {#related-pages}

- [Runtime PM](runtime-pm.md) — 使用カウント、autosuspend、dev_pm_ops。genpdのruntime_suspend/resumeフックはこれの上に構築されている
- [システムサスペンド](suspend.md) — システム全体のスリープ。genpdはfreeze/suspendの手順に参加する
- [デバイスツリー](../drivers/device-tree.md) — phandleの解析。`power-domains` DTバインディングが依存する仕組み

### LWNの記事 {#lwn-articles}

- [PM / Domains: Support for generic I/O PM domains](https://lwn.net/Articles/449302/) — Rafael J. Wysockiによる最初のgenpdパッチセット（2011年）。このページで説明している共有電源リソースと親子ドメインのモデルを解説している

### 外部資料 {#external}

- [Documentation/driver-api/pm/devices.rst — Device Power Management Domains](https://docs.kernel.org/driver-api/pm/devices.html#device-power-management-domains) — `dev->pm_domain`、`struct dev_pm_domain`、そして電源ドメインのコールバックがサブシステムのコールバックより優先される理由についてのアップストリームの説明
