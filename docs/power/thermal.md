# 熱管理: サーマルゾーンと冷却 {#thermal-management-thermal-zones-and-cooling}

> Linuxの熱フレームワーク、トリップポイント、冷却デバイス、サーマルガバナ

## 概要 {#overview}

現代のSoCやCPUは熱を発生させる。熱管理がなければ、予期せずスロットリングしたりシャットダウンしたりしてしまう。Linuxの熱フレームワークは、次のことを行うための構造化された方法を提供する。
1. サーマルゾーンを介して温度を**監視する**
2. トリップポイントとガバナを介して熱ポリシーを**定義する**
3. 冷却デバイス（CPU周波数スケーリング、ファン制御）を介して冷却を**実施する**

```
Hardware sensor → thermal zone → governor → cooling device
     (NTC, PCH, CPU, GPU)          |           (cpufreq, fan, GPU)
                                   ↓
                          trip points: warning/critical/throttle
```

## サーマルゾーン {#thermal-zones}

**サーマルゾーン**は、センサーとそれに関連付けられたポリシーを表す。

```bash
# すべてのサーマルゾーンを列挙する:
ls /sys/class/thermal/
# thermal_zone0  thermal_zone1  thermal_zone2  ...  cooling_device0  ...

# ゾーンを確認する:
cat /sys/class/thermal/thermal_zone0/type          # "x86_pkg_temp"
cat /sys/class/thermal/thermal_zone0/temp          # 現在の温度（ミリ度単位）
# 45000 = 45°C
cat /sys/class/thermal/thermal_zone0/mode          # "enabled" または "disabled"
cat /sys/class/thermal/thermal_zone0/policy        # "step_wise" など

# トリップポイントを列挙する:
for trip in /sys/class/thermal/thermal_zone0/trip_point_*_temp; do
    type=$(echo $trip | sed 's/_temp/_type/')
    echo "$(cat $type): $(cat $trip)m°C"
done
# passive: 95000m°C   ← 95°Cでスロットリング
# critical: 105000m°C ← 105°Cで緊急シャットダウン
```

### カーネルの構造体 {#kernel-structures}

```c
/* drivers/thermal/thermal_core.h — struct thermal_zone_device は熱サブシステムに
 * 非公開であり、include/linux/thermal.h では公開されていない。ドライバが
 * 目にするのは、登録関数が返す不透明なポインタだけである。 */

struct thermal_zone_device {
    int                 id;
    char                type[THERMAL_NAME_LENGTH];
    struct device       device;
    enum thermal_device_mode mode;
    void                *devdata;
    int                 num_trips;

    int                 temperature;        /* 現在の温度（ミリ度単位） */
    int                 last_temperature;
    int                 emul_temperature;   /* テスト用 */
    int                 passive;            /* パッシブ冷却が有効か? */

    struct thermal_zone_device_ops  ops;    /* .get_temp, .set_trips */
    struct thermal_zone_params      *tzp;   /* ガバナのパラメータ */
    struct thermal_governor         *governor;

    struct list_head    trips_high;         /* 現在温度より上のトリップ */
    struct list_head    trips_reached;      /* 現在越えているトリップ */
    struct mutex        lock;
    struct thermal_trip_desc trips[];       /* フレキシブル配列。トリップごとに1つ */
    /* ... */
};

/* トリップポイントの種類: include/uapi/linux/thermal.h */
enum thermal_trip_type {
    THERMAL_TRIP_ACTIVE = 0,    /* 冷却デバイス（ファン）を作動させる */
    THERMAL_TRIP_PASSIVE,       /* 性能を下げる（cpufreq） */
    THERMAL_TRIP_HOT,           /* ドライバ定義の動作 */
    THERMAL_TRIP_CRITICAL,      /* 緊急シャットダウン */
};
```

## サーマルゾーンの登録（ドライバ） {#registering-a-thermal-zone-driver}

```c
/* ドライバ: 温度センサーをサーマルゾーンとして登録する */
static int my_get_temp(struct thermal_zone_device *tzd, int *temp)
{
    /* struct thermal_zone_device は熱コアに非公開なので（上記参照）、
     * ドライバはtzd->devdataを直接参照できない —
     * 代わりにinclude/linux/thermal.hがエクスポートするアクセサを使う。 */
    struct my_sensor *sensor = thermal_zone_device_priv(tzd);
    *temp = read_sensor_millidegrees(sensor);
    return 0;
}

static struct thermal_zone_device_ops my_tz_ops = {
    .get_temp = my_get_temp,
};

/* probe時: */
struct thermal_trip trips[] = {
    { .temperature = 80000, .type = THERMAL_TRIP_PASSIVE },    /* 80°C */
    { .temperature = 100000, .type = THERMAL_TRIP_CRITICAL },  /* 100°C */
};

struct thermal_zone_device *tzd = thermal_zone_device_register_with_trips(
    "my_sensor",           /* typeの文字列 */
    trips,                 /* トリップポイントの配列 */
    ARRAY_SIZE(trips),     /* トリップポイントの数 */
    sensor_data,           /* ドライバのプライベートデータ */
    &my_tz_ops,
    NULL,                  /* サーマルゾーンのパラメータ */
    1000,                  /* passive_delay（ms） */
    5000                   /* polling_delay（ms） */
);
thermal_zone_device_enable(tzd);
```

## 冷却デバイス {#cooling-devices}

```c
/* 冷却デバイスは電力を減らすことができる: 例えばCPU周波数 */
struct thermal_cooling_device {
    int         id;
    const char  *type;
    struct      device device;
    struct      thermal_cooling_device_ops *ops;
    /* ... */
};

struct thermal_cooling_device_ops {
    int (*get_max_state)(struct thermal_cooling_device *, unsigned long *);
    int (*get_cur_state)(struct thermal_cooling_device *, unsigned long *);
    int (*set_cur_state)(struct thermal_cooling_device *, unsigned long);
};

/* CPU周波数スケーリングによる冷却デバイス（cpufreq_cooling）: */
/* state 0 = 最大周波数、state N = 最小周波数 */
struct thermal_cooling_device *cdev =
    cpufreq_cooling_register(cpu_policy);
/* これがcpufreqによる熱スロットリングの仕組みである */
```

```bash
# 冷却デバイスを見る:
ls /sys/class/thermal/cooling_device*/
cat /sys/class/thermal/cooling_device0/type    # "Processor" または "cpufreq-cpu0"
cat /sys/class/thermal/cooling_device0/max_state  # 例: 3（4段階）
cat /sys/class/thermal/cooling_device0/cur_state  # 現在のレベル（0=オフ）
```

## サーマルガバナ {#thermal-governors}

ガバナは、冷却を**いつ**作動させ、**どの程度**行うかを決定する。

### step_wise（多くのプラットフォームでのデフォルト） {#step_wise-default-on-many-platforms}

温度がトリップポイントを超えると冷却レベルを1上げ、下回ると1下げる。

```
temp rising:  trip reached → cooling_state++
temp falling: temp < (trip - hysteresis) → cooling_state--
```

### power_allocator（IPA — Intelligent Power Allocation） {#power_allocator-ipa-intelligent-power-allocation}

電力予算を冷却デバイス間に配分するPIDコントローラである。

```c
/* drivers/thermal/gov_power_allocator.c */
/* パラメータ（/sys/class/thermal/thermal_zone*/
/*            /sustainable_power などで調整可能）:   */
/* sustainable_power: 持続可能な最大電力（mW）        */
/* k_po, k_pu, k_i, k_d: PIDコントローラのゲイン — 比例 */
/* （オーバーシュート/アンダーシュート）、積分、微分項。 */
/* 下記のsysfsファイルは10進整数を表示・受け付けるが、   */
/* 値は単純な整数ではなく固定小数点（ドライバ内の        */
/* mul_frac()/int_to_frac()）として格納される。          */
/* 見積もり: (current_temp - control_temp) → power_budget */
/* → 各冷却デバイスに比例配分する                       */
```

```bash
# power_allocatorガバナを調整する:
echo power_allocator > /sys/class/thermal/thermal_zone0/policy
echo 3000 > /sys/class/thermal/thermal_zone0/sustainable_power  # 3Wの予算
echo 100  > /sys/class/thermal/thermal_zone0/k_po  # 比例ゲイン（オーバーシュート）
echo 100  > /sys/class/thermal/thermal_zone0/k_pu  # 比例ゲイン（アンダーシュート）
echo 10   > /sys/class/thermal/thermal_zone0/k_i   # 積分ゲイン
echo 0    > /sys/class/thermal/thermal_zone0/k_d   # 微分ゲイン
```

### user_space {#user_space}

判断をユーザー空間のデーモン（例: thermald）に委ねる。

```bash
echo user_space > /sys/class/thermal/thermal_zone0/policy
# thermaldなどの熱管理デーモンが温度を読み取り、次のように設定する:
echo 1 > /sys/class/thermal/cooling_device0/cur_state
```

## ACPIサーマルゾーン {#acpi-thermal-zones}

x86では、ACPIがDSDT/SSDTの中でサーマルゾーンを定義する。

```bash
# ACPIサーマルゾーン（当初はLinuxの熱フレームワークとは別）:
cat /sys/class/thermal/thermal_zone*/type | grep -i acpi
# acpitz

# ACPIのトリップポイントはACPIの_PSV（パッシブ）、_CRT（クリティカル）、_HOTから来る。
# ThermalZoneはASLのキーワードであり、生のAMLバイナリには現れないので、
# 生のダンプには出てこない — まず逆アセンブルする:
acpidump -n DSDT -o dsdt.dat
iasl -d dsdt.dat            # dsdt.dsl（ASLソース）を生成する
grep -A 20 "ThermalZone" dsdt.dsl
```

## ファン制御 {#fan-control}

```bash
# ファンの冷却デバイス:
cat /sys/class/thermal/cooling_device*/type | grep -i fan
# Fan

# 手動のファン制御（熱フレームワークをバイパスする）:
# （プラットフォームによって異なる。よくある方法:）

# Dellのノートパソコン（i8k）: /proc/i8kは読み取り専用の状態出力である（drivers/hwmon/dell-smm-hwmon.c
# は.proc_ioctlハンドラを登録するが、.proc_writeは登録しない）。ファンレベルの設定には
# I8K_SET_FAN ioctlが必要であり（例: `i8kutils`/`i8kfan`ユーザー空間ツール経由）、
# 単純なsysfs風の書き込みではできない。

# hwmon経由のACPIファン:
cat /sys/class/hwmon/hwmon*/name
cat /sys/class/hwmon/hwmon0/fan1_input  # RPM
echo 200 > /sys/class/hwmon/hwmon0/pwm1  # 0-255の速度

# ファンが熱制御されているか確認する:
cat /sys/class/hwmon/hwmon0/pwm1_enable
# 0 = 全速、1 = 手動、2 = 自動（熱制御）
echo 2 > /sys/class/hwmon/hwmon0/pwm1_enable  # 自動モード
```

## 可観測性とデバッグ {#observability-and-debugging}

```bash
# サーマルゾーンをリアルタイムで監視する:
watch -n 1 'for z in /sys/class/thermal/thermal_zone*/; do
    echo -n "$(cat $z/type): $(cat $z/temp)m°C  "; done; echo'

# カーネルの熱関連tracepoint（drivers/thermal/thermal_trace.h）:
echo 1 > /sys/kernel/debug/tracing/events/thermal/enable
cat /sys/kernel/debug/tracing/trace_pipe
# thermal_temperature: thermal_zone=x86_pkg_temp id=0 temp_prev=44000 temp=45000
# thermal_zone_trip: thermal_zone=x86_pkg_temp id=0 trip=0 trip_type=passive
# （thermal_zone_tripが持つのはトリップのインデックスと種類であり、温度ではない —
#  現在の温度は対になるthermal_temperatureイベントか/sysから読む）

# BPFで熱スロットリングをトレースする:
bpftrace -e '
tracepoint:thermal:thermal_zone_trip
{
    printf("TRIP: zone=%s, id=%d, trip=%d, type=%d\n",
           str(args->thermal_zone), args->id, args->trip,
           args->trip_type);
}'

# CPUが熱スロットリング（PROCHOT）されているか確認する: 情報はthermal_throttle/以下の
# 数値カウンタファイルの*名前*にあり、中身にはないので、文字列をgrepしても
# 一致しない — 代わりにカウンタをcatする:
cat /sys/devices/system/cpu/cpu0/thermal_throttle/core_throttle_count
cat /sys/devices/system/cpu/cpu0/thermal_throttle/core_power_limit_count
cat /sys/devices/system/cpu/cpu0/thermal_throttle/package_throttle_count

# IntelでのMSRベースのスロットリング（IA32_THERM_STATUSのビット、SDM Vol 3B §15.7による）:
rdmsr -p0 0x19c   # IA32_THERM_STATUS（コアごと）
# bit 0: Thermal Status — 現在、温度によりスロットリングされている
# bit 1: Thermal Status Log — 「前回のクリア以降に発生したか」を示すスティッキーフラグ
# bit 2: PROCHOT#/FORCEPR Event — 現在、外部からのPROCHOT#アサートによりスロットリングされている
# bit 3: PROCHOT#/FORCEPR Log — bit 2のスティッキー版
# （bit 0 = 現在の熱状態によるスロットリング、bit 2 = 現在のPROCHOT#によるスロットリング — 別々のイベント）

# パッケージ全体の相当物:
rdmsr -p0 0x1b1   # IA32_PACKAGE_THERM_STATUS: 同じビット配置、パッケージ全体
```

## ARMの熱管理: SCMIとTF-A {#arm-thermal-scmi-and-tf-a}

ARM SoCでは、温度管理にファームウェアが関与することが多い。

```bash
# SCMIによる熱管理（System Control and Management Interface）:
# DTで記述されたゾーンは、sysfsの"type"をthermal-zonesノード名そのものから
# 得る（drivers/thermal/thermal_of.c内のthermal_zone_device_register_with_trips(np->name, ...)）。
# SCMIを使うゾーンのセンサーは、固定のtype文字列ではなく、
# "thermal-sensors = <&scmi_sensors N>"という参照から得られる。
cat /sys/class/thermal/thermal_zone*/type

# TF-A（Trusted Firmware-A）がクリティカルシャットダウンを処理する。
# LinuxはセキュアワールドへのSCMIプロトコル経由で温度を取得する
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/thermal.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/thermal.h) — `struct thermal_trip`、`struct thermal_zone_device_ops`、そして実際の登録APIである`thermal_zone_device_register_with_trips()`（古い`thermal_zone_device_register()`はもう存在しない）
- [drivers/thermal/thermal_core.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/thermal/thermal_core.h) — サブシステムに非公開の、実際の`struct thermal_zone_device`のレイアウト（`include/linux/thermal.h`にはない。ドライバが持つのは不透明なポインタだけである）
- [drivers/thermal/gov_step_wise.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/thermal/gov_step_wise.c) — `get_target_state()`。`step_wise`の背後にある`cur_state ± 1`のスロットリングロジック
- [drivers/thermal/gov_power_allocator.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/thermal/gov_power_allocator.c) — `power_allocator`の背後にあるPIDコントローラ（`k_po`、`k_pu`、`k_d`、`sustainable_power`）
- [include/linux/cpu_cooling.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/cpu_cooling.h) — cpufreq冷却デバイスのコンストラクタである`cpufreq_cooling_register()`
- [drivers/thermal/thermal_trace.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/thermal/thermal_trace.h) — `thermal_temperature`および`thermal_zone_trip` tracepointのフィールド定義
- [arch/x86/include/asm/msr-index.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/msr-index.h) — `MSR_IA32_THERM_STATUS`（0x19c、コアごと）と`MSR_IA32_PACKAGE_THERM_STATUS`（0x1b1）。どちらもPROCHOTはビット0

### 関連ページ {#related-pages}

- [cpufreqとP-state](cpufreq.md) — CPU周波数スケーリング（熱の冷却デバイス）
- [cpuidle](cpuidle.md) — C-stateは電力を減らして熱の問題を防ぐ
- [Runtime PM](runtime-pm.md) — デバイスの電源管理
- [デバイスツリー](../drivers/device-tree.md) — ARMではDTSでサーマルゾーンが定義される

### LWNの記事 {#lwn-articles}

- [The power allocator thermal governor](https://lwn.net/Articles/602517/)（2014年6月17日） — `power_allocator`/IPAの背後にあるPIDコントローラの設計と、複数アクターにわたる電力予算配分

### 外部資料 {#external}

- [Generic Thermal Sysfs driver How To](https://docs.kernel.org/driver-api/thermal/sysfs-api.html) — サーマルゾーンと冷却デバイスの登録API、およびsysfs ABIの全体（トリップポイント、`policy`、`mode`）
- [Power allocator governor tunables](https://docs.kernel.org/driver-api/thermal/power_allocator.html) — `k_po`、`k_pu`、`k_d`、`sustainable_power`が実際に何を制御するか
