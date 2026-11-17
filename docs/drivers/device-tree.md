# Device Tree: ARMのハードウェア記述 {#device-tree-arm-hardware-description}

> DTSの構文、of_ API、プラットフォームドライバのバインディング、DTオーバーレイ

## Device Treeとは何か {#what-is-a-device-tree}

x86では、カーネルは実行時にハードウェアを検出する（ACPI、PCIの列挙）。ARMやその他の組み込みプラットフォームでは、ハードウェアは自己記述的ではない。**Device Tree**（DT）は、ブートローダからカーネルに渡される静的な記述である。Device Treeの形式はSunのOpenFirmwareに由来し、Linux/PowerPC向けに採用されたのち、Grant LikelyがARMへの採用を推進した[（LWN）](https://lwn.net/Articles/414016/)。ARMのDevice TreeサポートはLinux 3.0（2011年7月）で入った[（commit）](https://git.kernel.org/linus/9eb8f6743b076b67f00776cda4330c802e157b41)。

```
ブートローダ（U-Boot）
  → カーネルImage + DTB（device tree blob）をRAMにロードする
  → x0 = DTBの物理アドレス、x1 = 0（予約）、x2 = 0、x3 = 0 を設定する
  → カーネルのエントリポイントにジャンプする

カーネル:
  → setup_machine_fdt(dtb)がblobをマップしてパースする
  → 各ハードウェアブロックについてstruct deviceのノードを作成する
  → プラットフォームドライバがcompatible文字列を介してノードにbindする
```

## DTSの構文 {#dts-syntax}

Device Treeのソース（`.dts`）は、`dtc`でバイナリblob（`.dtb`）にコンパイルされる。

```dts
/* arch/arm64/boot/dts/example.dts */
/dts-v1/;

/ {
    model = "My Board v1.0";
    compatible = "myvendor,myboard";
    #address-cells = <1>;    /* アドレスにu32を1個 */
    #size-cells = <1>;       /* サイズにu32を1個 */

    /* CPU */
    cpus {
        #address-cells = <1>;
        #size-cells = <0>;
        cpu@0 {
            compatible = "arm,cortex-a53";
            device_type = "cpu";
            reg = <0>;
            enable-method = "psci";
        };
    };

    /* メモリ */
    memory@80000000 {
        device_type = "memory";
        reg = <0x80000000 0x40000000>;  /* 0x80000000から1GB */
    };

    /* SoCの周辺機器 */
    soc {
        compatible = "simple-bus";
        #address-cells = <1>;
        #size-cells = <1>;
        ranges;                          /* 恒等マッピング */

        uart0: serial@11000000 {
            compatible = "ns16550a";
            reg = <0x11000000 0x1000>;   /* MMIOのベース + サイズ */
            interrupts = <0 32 4>;       /* タイプSPI、irq 32、アクティブLowのレベル */
            clocks = <&uart_clk>;
            clock-names = "baudclk";
            current-speed = <115200>;
            status = "okay";             /* 除外するには"disabled" */
        };

        i2c0: i2c@12000000 {
            compatible = "myvendor,i2c-v1";
            reg = <0x12000000 0x100>;
            interrupts = <0 40 4>;
            #address-cells = <1>;
            #size-cells = <0>;
            clock-frequency = <400000>;  /* 400kHzのファストモード */
            status = "okay";

            /* アドレス0x48のI2Cデバイス */
            temp_sensor: temperature@48 {
                compatible = "ti,tmp102";
                reg = <0x48>;
            };
        };

        gpio: gpio@13000000 {
            compatible = "myvendor,gpio";
            reg = <0x13000000 0x1000>;
            #gpio-cells = <2>;           /* ピン + フラグ */
            gpio-controller;
            interrupt-controller;
            #interrupt-cells = <2>;
        };
    };
};
```

```bash
# DTSをDTBにコンパイルする:
dtc -I dts -O dtb -o board.dtb board.dts

# DTBをDTSに逆コンパイルする:
dtc -I dtb -O dts -o board.dts board.dtb

# 実行中のカーネルから現在のDTBをダンプする:
dtc -I dtb -O dts /proc/device-tree > running.dts
# または:
ls /proc/device-tree/
```

## DTSIのインクルード {#dtsi-includes}

```dts
/* SoCレベルのDTSI（共有されるハードウェア記述）: */
/* arch/arm64/boot/dts/myvendor/myvendor-soc.dtsi */
/ {
    soc {
        uart0: serial@11000000 {
            compatible = "ns16550a";
            reg = <0x11000000 0x1000>;
            status = "disabled";         /* デフォルトでオフ */
        };
    };
};

/* ボードレベルのDTS（ボード固有の設定）: */
/* &uart0のようなラベル参照は、/ { }の外側のトップレベルに置かなければならない */
#include "myvendor-soc.dtsi"
/ {
    model = "My Board v1.0";
};

&uart0 {
    status = "okay";             /* このボードで有効化する */
    current-speed = <115200>;
};
```

## プラットフォームドライバのバインディング {#platform-driver-binding}

### compatible文字列 {#the-compatible-string}

カーネルは`compatible`プロパティを介してDTのノードとドライバをマッチさせる。

```c
/* drivers/tty/serial/ns16550.c */

/* デバイステーブル: DTS内の"compatible"文字列とマッチする */
static const struct of_device_id ns16550_of_match[] = {
    { .compatible = "ns16550a",   .data = &ns16550_config },
    { .compatible = "ns16550",    .data = &ns16550_config },
    { .compatible = "snps,dw-apb-uart", .data = &dw_apb_config },
    { /* 番兵 */ }
};
MODULE_DEVICE_TABLE(of, ns16550_of_match);

static struct platform_driver ns16550_driver = {
    .probe   = ns16550_probe,
    .remove  = ns16550_remove,
    .driver  = {
        .name           = "ns16550",
        .of_match_table = ns16550_of_match,   /* DTバインディング */
        .pm             = &ns16550_pm_ops,
    },
};
module_platform_driver(ns16550_driver);
```

### DTのプロパティを読むprobe() {#probe-reading-dt-properties}

```c
static int ns16550_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct device_node *np = dev->of_node;   /* DTノード */
    struct ns16550_port *port;
    struct resource *res;
    u32 clk_freq;
    int irq, ret;

    port = devm_kzalloc(dev, sizeof(*port), GFP_KERNEL);
    if (!port)
        return -ENOMEM;

    /* DTの"reg"プロパティからMMIOリソースを取得する */
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    port->base = devm_ioremap_resource(dev, res);
    if (IS_ERR(port->base))
        return PTR_ERR(port->base);

    /* DTの"interrupts"プロパティからIRQを取得する */
    irq = platform_get_irq(pdev, 0);
    if (irq < 0)
        return irq;

    /* u32プロパティを読む */
    ret = of_property_read_u32(np, "current-speed", &port->baud);
    if (ret)
        port->baud = 115200;     /* デフォルト */

    /* 文字列プロパティを読む */
    const char *str;
    ret = of_property_read_string(np, "clock-names", &str);

    /* u32の配列を読む */
    u32 clk_vals[2];
    ret = of_property_read_u32_array(np, "clock-frequency", clk_vals, 2);

    /* 真偽値プロパティを確認する */
    if (of_property_read_bool(np, "fifo-enable"))
        port->has_fifo = true;

    /* GPIOを取得する */
    port->reset_gpio = devm_gpiod_get_optional(dev, "reset", GPIOD_OUT_LOW);

    /* クロックを取得する */
    port->clk = devm_clk_get(dev, "baudclk");
    if (!IS_ERR(port->clk)) {
        clk_prepare_enable(port->clk);
        port->uartclk = clk_get_rate(port->clk);
    }

    /* レギュレータを取得する */
    port->vdd = devm_regulator_get_optional(dev, "vdd");

    ret = devm_request_irq(dev, irq, ns16550_isr, 0, dev_name(dev), port);
    if (ret)
        return ret;

    return uart_add_one_port(&ns16550_uart_driver, &port->uart);
}
```

## of_ APIリファレンス {#of_-api-reference}

```c
/* ノードの検索 */
struct device_node *of_find_compatible_node(struct device_node *from,
                                             const char *type,
                                             const char *compatible);
struct device_node *of_find_node_by_phandle(phandle handle);
struct device_node *of_get_parent(const struct device_node *node);
struct device_node *of_get_next_child(const struct device_node *node,
                                       struct device_node *prev);

/* プロパティの読み出し */
int of_property_read_u32(const struct device_node *np,
                          const char *propname, u32 *out_value);
int of_property_read_u64(const struct device_node *np,
                          const char *propname, u64 *out_value);
int of_property_read_string(const struct device_node *np,
                             const char *propname, const char **out_string);
int of_property_read_u32_array(const struct device_node *np,
                                const char *propname,
                                u32 *out_values, size_t sz);
bool of_property_read_bool(const struct device_node *np,
                            const char *propname);

/* アドレス変換 */
int of_address_to_resource(struct device_node *dev, int index,
                             struct resource *r);
void __iomem *of_iomap(struct device_node *node, int index);

/* IRQのマッピング */
int of_irq_get(struct device_node *dev, int index);
int of_irq_count(struct device_node *dev);
```

## phandleと相互参照 {#phandles-and-cross-references}

```dts
/* phandleを使うと、ノードが他のノードを参照できる */
clocks {
    uart_clk: uart_clk {
        compatible = "fixed-clock";
        #clock-cells = <0>;
        clock-frequency = <24000000>;   /* 24MHz */
    };

    spi_clk: spi_clk {
        compatible = "fixed-clock";
        #clock-cells = <0>;
        clock-frequency = <48000000>;
    };
};

uart0: serial@11000000 {
    clocks = <&uart_clk>;            /* phandle参照 */
    clock-names = "baudclk";
};

/* 引数付きのGPIO参照 */
leds {
    led-red {
        gpios = <&gpio 5 GPIO_ACTIVE_HIGH>;  /* gpioノード、ピン5、アクティブHigh */
    };
};
```

```c
/* ドライバ内でphandle参照を解決する: */
struct device_node *clk_node = of_parse_phandle(np, "clocks", 0);

/* あるいはクロックフレームワークの抽象化を使う: */
struct clk *clk = of_clk_get_by_name(np, "baudclk");
```

## Device Treeオーバーレイ（DTO） {#device-tree-overlays-dto}

オーバーレイを使うと、稼働中のDTを動的に変更できる（例: Raspberry PiのHAT）。

```dts
/* overlay.dts - i2c0にセンサーを追加する */
/dts-v1/;
/plugin/;

/ {
    compatible = "raspberrypi,3-model-b";

    fragment@0 {
        target = <&i2c0>;
        __overlay__ {
            #address-cells = <1>;
            #size-cells = <0>;

            bmp280@76 {
                compatible = "bosch,bmp280";
                reg = <0x76>;
            };
        };
    };
};
```

```bash
# オーバーレイをコンパイルする:
dtc -I dts -O dtb -o my-overlay.dtbo my-overlay.dts

# 実行時にオーバーレイを適用する（Raspberry Pi / configfs）:
mkdir /sys/kernel/config/device-tree/overlays/my-overlay
cat my-overlay.dtbo > /sys/kernel/config/device-tree/overlays/my-overlay/dtbo

# オーバーレイを削除する:
rmdir /sys/kernel/config/device-tree/overlays/my-overlay

# 適用済みのオーバーレイを確認する:
ls /sys/kernel/config/device-tree/overlays/
```

## Device Treeのデバッグ {#debugging-device-trees}

```bash
# ファイルシステムとしての稼働中のDT:
ls /proc/device-tree/
cat /proc/device-tree/model
# My Board v1.0

# sysfs: DTから作成されたプラットフォームデバイス:
ls /sys/bus/platform/devices/
# 11000000.serial  12000000.i2c  ...

# ドライバのbindを確認する:
ls -la /sys/bus/platform/devices/11000000.serial/driver
# → /sys/bus/platform/drivers/ns16550

# あるcompatibleにマッチするすべてのDTノードをダンプする:
grep -r "compatible" /proc/device-tree/ 2>/dev/null | head -20

# 稼働中のカーネルに対してdtcを使う:
apt install device-tree-compiler
dtc -I fs /proc/device-tree 2>/dev/null | grep -A5 "serial@"

# DTのprobe失敗を確認する:
dmesg | grep -E "of_platform|platform|DT" | head -20

# Device Treeの統計:
cat /sys/kernel/debug/of_platform_bus/stats 2>/dev/null

# ノードに何がマッチしたかを調べる:
cat /sys/bus/platform/devices/11000000.serial/of_node/compatible
# ns16550a

# fdt_probe_driver: ドライバがprobeしなかった理由を確認する
dmesg | grep "11000000.serial"
# [    1.234] 11000000.serial: ttyS0 at MMIO 0x11000000 (irq = 32, ...) is a NS16550A
```

## プラットフォームデバイスのライフサイクル {#platform-device-lifecycle}

```
DT blob → of_platform_populate() → creates struct platform_device for each node
                                                      ↓
                             platform_device.name = "11000000.serial"
                             platform_device.dev.of_node = &dt_node
                             platform_device.resource[] = [MMIO, IRQ]
                                                      ↓
                        platform_driver registered → of_match_table checked
                                                      ↓
                                    probe() called with struct platform_device *
```

```c
/* ブート中: arch/arm64/kernel/setup.c */
of_platform_populate(NULL, of_default_bus_match_table, NULL, NULL);
/* of_default_bus_match_table（"simple-bus"などを含む）にマッチする
   compatibleを持つすべてのノードについてplatform_deviceを作成する */

/* probe()に渡されるstruct platform_device: */
struct platform_device {
    const char      *name;          /* "11000000.serial" */
    int             id;             /* DT由来なら-1 */
    struct device   dev;            /* of_nodeへのポインタを含む */
    u32             num_resources;
    struct resource *resource;      /* MMIOのベース+サイズ、IRQ */
};
```

## DTとACPIの比較 {#dt-vs-acpi}

| 機能 | Device Tree | ACPI |
|---------|-------------|------|
| プラットフォーム | ARM、RISC-V、PowerPC | x86、ARM64サーバー |
| 記述 | DTBにコンパイルされる静的なDTSファイル | BIOS内のAMLバイトコード |
| 検出 | ドライバ内のof_match_table | ACPIテーブル内のHID/CID |
| 更新 | DTオーバーレイ、再起動 | DSDTパッチ |
| デバッグ | /proc/device-tree、dtc | acpidump、acpiexec |
| クロック | DTのクロックフレームワーク | ACPI _CRS |
| 電源 | DTのレギュレータ、PSCI | ACPI _PSxメソッド |

```bash
# システムがACPIとDTのどちらを使っているか確認する:
ls /proc/device-tree  2>/dev/null && echo "Device Tree" || echo "ACPI/other"
ls /sys/firmware/acpi 2>/dev/null && echo "ACPI present"
```

## 参考文献 {#further-reading}

- [PCIドライバ](pci-driver.md) — PCIデバイスのバインディング（DTのplatformバインディングとの対比）
- [IRQアフィニティ](../interrupts/irq-affinity.md) — DTの割り込みコントローラのバインディング
- [プリエンプションモデル](../sched/preemption.md) — 組み込み/ARMでのRT
- [リアルタイムチューニング](../sched/rt-tuning.md) — ARM向けのPREEMPT_RT
- `Documentation/devicetree/bindings/` — DTバインディングの仕様
- `drivers/of/` — of_ APIの実装
- `scripts/dtc/` — Device Tree Compiler
- `arch/arm64/boot/dts/` — アップストリームのDTSファイル
