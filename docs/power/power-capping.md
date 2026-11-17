# パワーキャッピングとRAPL {#power-capping-and-rapl}

> Intel RAPL、powercapフレームワーク、そしてCPU消費電力の制限

## RAPLとは何か {#what-is-rapl}

**Running Average Power Limit**（RAPL）は、Sandy Bridge（2011年）で導入されたIntelのハードウェア機能であり、次の2つを行う。

1. ダイ上のカウンタを用いて、主要なハードウェアドメインのエネルギー消費を**計測する**
2. ドメインの移動平均が設定された上限を超えたときにクロック周波数をスロットリングすることで、持続的な消費電力を**制限する**

RAPLはソフトウェアによる電力計ではない。カウンタはパッケージの電源管理ユニットに組み込まれており、1ティックあたりおよそ15.3 µJ（デフォルトのエネルギー単位である2⁻¹⁶ J。`MSR_RAPL_POWER_UNIT`から得られる）の精度を持つ。これは`perf stat -e power/energy-pkg/`、powertopのパッケージ電力表示、そして多くのデータセンター向け電源管理ツールの基盤となっている。

## RAPLドメイン {#rapl-domains}

各電力ドメインは、それぞれ異なる物理的範囲をカバーする。

| ドメイン | 範囲 | 利用可能になった世代 |
|--------|-------|----------------|
| `package` | CPUソケット全体（コア + uncore + LLC） | Sandy Bridge |
| `core` | CPUコア + L1/L2キャッシュのみ | Sandy Bridge |
| `uncore` | GPU（統合グラフィックス / リングバス） | Sandy Bridge |
| `dram` | DRAMの消費電力 | Sandy Bridge EP |
| `psys` | プラットフォーム全体（SoC + VRの損失） | Skylake |

デュアルソケットのサーバーでは、2つのパッケージに対応する`intel-rapl:0`と`intel-rapl:1`が見え、それぞれがサブドメイン用の子ゾーンを持つ。

## MSRインターフェース {#the-msr-interface}

ハードウェアレベルでは、RAPLはモデル固有レジスタ（Model-Specific Register）を通じて公開されている。

```
MSR_PKG_ENERGY_STATUS   (0x611)  — rolling energy counter, ~15.3 µJ units
MSR_PKG_POWER_LIMIT     (0x610)  — configures limit 1 (sustained) and limit 2 (burst)
MSR_PKG_POWER_INFO      (0x614)  — TDP, min power, max power (read-only)

MSR_PP0_ENERGY_STATUS   (0x639)  — core domain energy
MSR_PP1_ENERGY_STATUS   (0x641)  — uncore domain energy
MSR_DRAM_ENERGY_STATUS  (0x619)  — DRAM domain energy
MSR_PLATFORM_ENERGY_STATUS (0x64d) — psys domain energy (Skylake+)
```

エネルギー単位（LSBあたりのジュール）は`MSR_RAPL_POWER_UNIT (0x606)`のビット[12:8]から読み取る。電力単位（LSBあたりのワット）は同じMSRのビット[3:0]から得られ、通常は1/8 Wである。

MSRへの直接アクセスには`CAP_SYS_RAWIO`と`msr`カーネルモジュールが必要である。ユーザー空間からはpowercapのsysfsインターフェースを使うのが推奨される経路である。

## powercapフレームワーク {#the-powercap-framework}

`drivers/powercap/`は、パワーキャッピング用ハードウェアのための汎用的な抽象化を提供する。このフレームワークは次のことを行う。

- 各ハードウェアドメインを**powercapゾーン**（`struct powercap_zone`）として表現する
- ゾーンを`/sys/class/powercap/`以下のディレクトリとして公開する
- 異なるハードウェア（Intel RAPL、ARM SCMI）向けのドライバが、sysfsのコードを重複させることなくゾーンを登録できるようにする

```c
/* include/linux/powercap.h */
struct powercap_zone {
    int                             id;
    char                           *name;
    void                           *control_type_inst;
    const struct powercap_zone_ops *ops;
    struct device                   dev;
    int                             const_id_cnt;
    struct idr                      idr;
    struct idr                     *parent_idr;
    /* ... */
};

struct powercap_zone_ops {
    int (*get_max_energy_range_uj)(struct powercap_zone *, u64 *);
    int (*get_energy_uj)(struct powercap_zone *, u64 *);
    int (*reset_energy_uj)(struct powercap_zone *);
    int (*get_max_power_range_uw)(struct powercap_zone *, u64 *);
    int (*get_power_uw)(struct powercap_zone *, u64 *);
    int (*set_enable)(struct powercap_zone *, bool);
    int (*get_enable)(struct powercap_zone *, bool *);
    int (*release)(struct powercap_zone *);
};
```

まずコントロールタイプ（例: `intel-rapl`）が`powercap_register_control_type()`で登録され、続いて各RAPLドメインが`powercap_register_zone()`でゾーンとして登録される。子ゾーン（core、dram、uncore）は、パッケージゾーンを親として登録される。

## intel_raplドライバ {#the-intel_rapl-driver}

`drivers/powercap/intel_rapl_common.c`は、RAPLのMSRとpowercapフレームワークを橋渡しするドライバである。このドライバは次のことを行う。

1. CPUパッケージごとにMSRをプローブしてRAPLドメインを列挙する
2. `MSR_RAPL_POWER_UNIT`からエネルギー単位と電力単位を読み取る
3. 各ドメインを`powercap_zone`として登録する
4. 適切なエネルギーMSRを読み取ってスケーリングすることで`get_energy_uj`を実装する

この共通コードの上に、2つのインターフェースドライバがロードされる。

- `intel_rapl_msr.c` — MSR経由で直接RAPLにアクセスする（ベアメタル。Sandy Bridge以来の従来の経路）
- `intel_rapl_tpmi.c` — TPMI（Topology Aware Register and PM Capsule Interface）経由でRAPLにアクセスする。TPMIはPCIe VSECで公開されるMMIO領域であり、MSRアクセスで必要となるCPUごとのスケジューリングを回避できる。PMT/TPMIハードウェアを備えた新しいXeonプラットフォームで使われる

ドライバがロードされたシステムでは、`lsmod | grep rapl`を実行すると、`intel_rapl_common`に加えて、インターフェースに応じて`intel_rapl_msr`または`intel_rapl_tpmi`のいずれか、そして（一部のプラットフォームでは）perfイベントのバックエンドである`rapl`が表示されるはずである。

## sysfsインターフェース {#sysfs-interface}

```bash
# すべてのpowercapゾーンを列挙する
ls /sys/class/powercap/
# intel-rapl  intel-rapl:0  intel-rapl:0:0  intel-rapl:0:1  intel-rapl:0:2

# ゾーン名
cat /sys/class/powercap/intel-rapl:0/name          # "package-0"
cat /sys/class/powercap/intel-rapl:0:0/name        # "core"
cat /sys/class/powercap/intel-rapl:0:1/name        # "uncore" or "dram"

# 現在の累積エネルギーを読む（µJ。ラップアラウンドするカウンタ）
cat /sys/class/powercap/intel-rapl:0/energy_uj
# 4882348230

# 電力を計算する: energy_ujを1秒間隔で2回読む
E1=$(cat /sys/class/powercap/intel-rapl:0/energy_uj)
sleep 1
E2=$(cat /sys/class/powercap/intel-rapl:0/energy_uj)
echo "Package power: $(( (E2 - E1) / 1000000 )) W"

# 現在の長期電力制限を読む（µW）
cat /sys/class/powercap/intel-rapl:0/constraint_0_power_limit_uw
# 45000000  (= 45 W)

# パッケージ0に35 Wの長期電力制限を設定する
echo 35000000 > /sys/class/powercap/intel-rapl:0/constraint_0_power_limit_uw

# ゾーンを有効にする（既に有効な場合もある）-- enabledはゾーン単位の
# 属性であり、制約ごとではない。constraint_0_enabledというファイルは存在しない
echo 1 > /sys/class/powercap/intel-rapl:0/enabled

# 制限の平均化に使われる時間窓を読む
cat /sys/class/powercap/intel-rapl:0/constraint_0_time_window_us
# 999424  (~1 second)
```

## Constraint 0とConstraint 1 {#constraint-0-vs-constraint-1}

RAPLはゾーンごとに2つの独立した電力制約をサポートする。

| 制約 | 名前 | 典型的な時間窓 | 振る舞い |
|-----------|------|---------------------|----------|
| `constraint_0` | 長期（PL1） | 1 s – 28 s | 持続的な上限。移動平均がこれを超えるとハードウェアがスロットリングする |
| `constraint_1` | 短期（PL2） | 1 ms – 7 ms | バースト用の上限。応答性のためにPL1を短時間超えることを許す |

PL2は常にPL1以上である。典型的なOEMの設定はPL1 = 15 W（熱設計上の上限）、PL2 = 25 W（上記の`constraint_1`の時間窓に従い数ミリ秒間のバースト）である。バースト窓が過ぎると、プロセッサはPL1に制限される。

```bash
# 短期（バースト）制限
cat /sys/class/powercap/intel-rapl:0/constraint_1_name
# "short_term"
cat /sys/class/powercap/intel-rapl:0/constraint_1_power_limit_uw
# 64000000  (64 W burst)
cat /sys/class/powercap/intel-rapl:0/constraint_1_time_window_us
# 2440       (~2.4 ms)
```

## perfによるエネルギー計測 {#measuring-energy-with-perf}

`perf`ツールは`power` PMUを通じてRAPLのエネルギーカウンタを読み取れる。これは内部的には同じMSRを使うが、perfイベント基盤と統合されている。

```bash
# ワークロードのパッケージエネルギーを計測する
perf stat -e power/energy-pkg/ -- ./my_workload
#  Performance counter stats for './my_workload':
#        24.35 Joules power/energy-pkg/
#       3.152370284 seconds time elapsed

# 利用可能なすべてのRAPLイベント
perf list | grep power
#  power/energy-cores/     [Kernel PMU event]
#  power/energy-gpu/       [Kernel PMU event]
#  power/energy-pkg/       [Kernel PMU event]
#  power/energy-ram/       [Kernel PMU event]
```

perfのバックエンドは`arch/x86/events/rapl.c`にあり、`power`という名前のPMUを登録して、`energy-pkg`を`MSR_PKG_ENERGY_STATUS`に、`energy-cores`を`MSR_PP0_ENERGY_STATUS`に、といった具合に対応付ける。

## ARMでの相当機能: SCMIパワーキャッピング {#arm-equivalent-scmi-power-capping}

ARMプラットフォームでは、SCMI（System Control and Management Interface）がプラットフォームファームウェアを介して同等の機能を提供する。ドライバ`drivers/powercap/arm_scmi_powercap.c`は、専用のSCMI **Powercap**プロトコル（`SCMI_PROTOCOL_POWERCAP`、プロトコルID `0x18`）のゾーンをpowercapゾーンとして登録するため、ファームウェアがこれを実装しているARMプラットフォームであれば、同じ`/sys/class/powercap/`インターフェースが使える。これは**Power domain management**（`SCMI_PROTOCOL_POWER`、プロトコルID `0x11`）とは別のSCMIプロトコルであり、後者はpowercapではなくLinuxの汎用電源ドメイン（genpd）フレームワークを支えるものである。各ゾーンの名前は、ファームウェアが報告するSCMIパワーキャッピングドメインのディスクリプタ（ドライバ内の`spz->info->name`）からそのまま取られる。つまりプラットフォームが定義するものであり、カーネル内の固定文字列ではない。

```bash
# SCMI powercapをサポートするARMプラットフォーム上で
ls /sys/class/powercap/
# arm-scmi  arm-scmi:0  ...
cat /sys/class/powercap/arm-scmi:0/name
# whatever name the platform firmware assigned this domain
```

SCMIプロトコル（Arm DEN0056で定義）は、LinuxとSCP（System Control Processor）の間で共有メモリまたはメールボックスを介して動作する。SCMIのメッセージタイプ`POWERCAP_CAP_GET` / `POWERCAP_CAP_SET`は、powercapゾーンのopsに直接対応する。

## 熱管理との相互作用 {#interaction-with-thermal-management}

RAPLの電力制限と熱フレームワークは補完関係にあり、重複するものではない。

```
Temperature-based:  Thermal zone → trip point → cpufreq cooling → frequency drop
Power-based:        RAPL PL1/PL2 → hardware throttles clocks directly

RAPL acts faster (hardware decision, microsecond granularity).
Thermal acts on sustained temperature (software, second granularity).
PROCHOT (thermal limit from temperature sensor) overrides both.
```

RAPLの制限をTDPより低く設定すると、温度センサーに頼ることなく定常状態の電力を制御できる。これは、熱的な余裕が制約にならない電力上限付きの環境（ラックごとに電力制限があるデータセンターなど）で有用である。

## 可観測性 {#observability}

```bash
# turbostatはパッケージごとのRAPLデータを継続的に表示する
sudo turbostat --show PkgWatt,CorWatt,RAMWatt --interval 1

# エネルギーカウンタを直接監視する
watch -n 1 'for z in /sys/class/powercap/intel-rapl:*/; do
    echo -n "$(cat $z/name): $(cat $z/energy_uj) µJ  "; done; echo'

# 制約が現在有効になっている（スロットリングしている）かを確認する
# （直接のsysfsノードはない。CPU周波数の低下やperfカウンタから推測する）
perf stat -e cpu-cycles,ref-cycles -- sleep 1
# ref-cycles >> cpu-cyclesなら、CPUはスロットリングされている
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/powercap.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/powercap.h) — `struct powercap_zone`、`struct powercap_zone_ops`、およびコントロールタイプ/ゾーンの登録API
- [drivers/powercap/intel_rapl_common.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/powercap/intel_rapl_common.c) — 両方のインターフェースドライバが使う、共通のRAPLドメイン/制約ロジック
- [drivers/powercap/intel_rapl_msr.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/powercap/intel_rapl_msr.c) — MSRベースのRAPLインターフェースドライバ
- [arch/x86/include/asm/msr-index.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/msr-index.h) — `MSR_RAPL_POWER_UNIT`、`MSR_PKG_ENERGY_STATUS`、およびこのページで使われるその他のMSRオフセット
- [drivers/powercap/arm_scmi_powercap.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/powercap/arm_scmi_powercap.c) — ARM SCMI powercapドライバ。SCMIのパワーキャッピングドメインをpowercapゾーンとして登録する
- [arch/x86/events/rapl.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/events/rapl.c) — RAPLカウンタを`perf stat -e power/energy-*/`に公開する`power` PMUのバックエンド

### manページ {#man-pages}

- [perf-stat(1)](https://man7.org/linux/man-pages/man1/perf-stat.1.html) — `power` PMUを通じてRAPLのエネルギーカウンタを読むのに使う`-e`のイベント選択構文

### 関連ページ {#related-pages}

- [cpufreqとP-state](cpufreq.md) — 周波数スケーリング。RAPL自身のハードウェアによるスロットリングとは別の制御経路
- [熱管理](thermal.md) — 温度ベースのスロットリング。RAPLの電力ベースの制限を補完する
- [電源管理の事件簿](war-stories.md) — RAPLのDRAMドメイン電力制限に関するインシデント

### LWNの記事 {#lwn-articles}

- [Power Capping Framework](https://lwn.net/Articles/562015/) — Srinivas Pandruvadaによる最初のRFCパッチシリーズ（2013年）。RAPLが登録先とする汎用の`drivers/powercap/` sysfsフレームワークを導入した

### 外部資料 {#external}

- [Documentation/power/powercap/powercap.rst](https://docs.kernel.org/power/powercap/powercap.html) — アップストリームのpowercapフレームワークのドキュメント: 用語、sysfsツリーの構成、コントロールタイプとゾーン
- Intel 64 and IA-32 Architectures Software Developer's Manual, Volume 3B, Chapter 15 (Power and Thermal Management) — RAPLのMSR定義
