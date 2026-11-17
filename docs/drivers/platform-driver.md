# プラットフォームドライバ {#platform-drivers}

> 組み込みデバイスやSoCデバイスで最も一般的なドライバのパターン

## プラットフォームデバイスとは {#what-platform-devices-are}

プラットフォームデバイスとは、（PCIやUSBとは異なり）ハードウェアによって検出できないデバイスである。`platform_device` / `platform_driver`という抽象化は、Linux 2.6.0とともに出荷されたLinuxドライバモデルの一部である。よい概説として、Jonathan Corbetによる2011年のLWN記事がある[（LWN）](https://lwn.net/Articles/448499/)。デバイスは次のいずれかによってインスタンス化される。

- **Device Tree**（ARM、RISC-V、組み込み）
- **ACPI**（x86、最近のARMサーバー）
- **ボードファイル**（古いARM、レガシー）

例: UART、I2Cコントローラ、GPIOコントローラ、クロック — SoC上に配線されていて、動的にprobeできないハードウェアである。

## struct platform_driver {#struct-platform_driver}

```c
/* include/linux/platform_device.h */
struct platform_driver {
    int (*probe)(struct platform_device *);   /* デバイスを検出、ドライバをbind */
    void (*remove)(struct platform_device *);  /* デバイスを削除、unbind */
    void (*shutdown)(struct platform_device *);
    int (*suspend)(struct platform_device *, pm_message_t state);
    int (*resume)(struct platform_device *);
    struct device_driver driver;              /* 埋め込まれた汎用ドライバ */
    const struct platform_device_id *id_table; /* レガシーなIDマッチング */
    bool driver_managed_dma;
};

struct platform_device {
    const char      *name;           /* レガシーなマッチング用 */
    int              id;             /* 名前の後ろに付加される: "serial.0" */
    struct device    dev;            /* 埋め込まれた汎用デバイス */
    u32              num_resources;
    struct resource *resource;       /* I/Oメモリ、IRQなど */
    const struct platform_device_id *id_entry;
    const char      *driver_override; /* 特定のドライバを強制する */
    struct mfd_cell *mfd_cell;
    struct pdev_archdata archdata;
};
```

## 最小限のプラットフォームドライバ {#a-minimal-platform-driver}

```c
/* drivers/mydriver/mydriver.c */
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/of.h>
#include <linux/io.h>
#include <linux/interrupt.h>
#include <linux/clk.h>

struct mydev {
    void __iomem    *base;    /* マップされたレジスタのベース */
    struct clk      *clk;
    int              irq;
};

/* IRQハンドラ */
static irqreturn_t mydev_irq(int irq, void *data)
{
    struct mydev *dev = data;
    u32 status = readl(dev->base + STATUS_REG);

    if (!(status & MY_IRQ_BIT))
        return IRQ_NONE;

    /* 割り込みを処理する */
    writel(MY_IRQ_BIT, dev->base + STATUS_CLR_REG);
    return IRQ_HANDLED;
}

static int mydev_probe(struct platform_device *pdev)
{
    struct mydev *dev;
    struct resource *res;
    int ret;

    /* ドライバのプライベートデータを割り当てる（devm: remove時に自動的に解放される） */
    dev = devm_kzalloc(&pdev->dev, sizeof(*dev), GFP_KERNEL);
    if (!dev)
        return -ENOMEM;

    /* リソース#0のI/Oメモリをマップする */
    dev->base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(dev->base))
        return PTR_ERR(dev->base);

    /* クロックを取得して有効化する */
    dev->clk = devm_clk_get(&pdev->dev, "mydev_clk");
    if (IS_ERR(dev->clk))
        return PTR_ERR(dev->clk);

    ret = clk_prepare_enable(dev->clk);
    if (ret)
        return ret;

    /* IRQを要求する */
    dev->irq = platform_get_irq(pdev, 0);
    if (dev->irq < 0)
        return dev->irq;

    ret = devm_request_irq(&pdev->dev, dev->irq, mydev_irq,
                           IRQF_SHARED, "mydev", dev);
    if (ret)
        return ret;

    /* ドライバデータを保存する */
    platform_set_drvdata(pdev, dev);

    dev_info(&pdev->dev, "mydev initialized at %p, irq %d\n",
             dev->base, dev->irq);
    return 0;
}

static void mydev_remove(struct platform_device *pdev)
{
    struct mydev *dev = platform_get_drvdata(pdev);

    /* devmリソースは自動的に解放される。クロックの無効化だけが必要 */
    clk_disable_unprepare(dev->clk);
}

/* Device Treeのマッチテーブル */
static const struct of_device_id mydev_of_match[] = {
    { .compatible = "vendor,mydev-v1" },
    { .compatible = "vendor,mydev-v2", .data = &mydev_v2_config },
    { }  /* 番兵 */
};
MODULE_DEVICE_TABLE(of, mydev_of_match);

/* レガシーなplatform IDマッチ（DT以外） */
static const struct platform_device_id mydev_id_table[] = {
    { "mydev", 0 },
    { }
};
MODULE_DEVICE_TABLE(platform, mydev_id_table);

static struct platform_driver mydev_driver = {
    .probe  = mydev_probe,
    .remove = mydev_remove,
    .id_table = mydev_id_table,
    .driver = {
        .name           = "mydev",
        .of_match_table = mydev_of_match,
        .pm             = &mydev_pm_ops,  /* 任意 */
    },
};

module_platform_driver(mydev_driver);
/* platform_driver_register/unregisterを呼ぶ
   module_init/module_exitに展開される */

MODULE_DESCRIPTION("Example platform driver");
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Example Author <author@example.com>");
```

## devm_: 自動リソース管理 {#devm_-automatic-resource-management}

`devm_`（device-managed）関数は、リソースの寿命を`struct device`に結びつける。デバイスがunbind（削除）されると、すべての`devm_`リソースが登録と逆の順序で解放される。

```c
/* メモリ */
devm_kzalloc(&dev, size, GFP_KERNEL)    /* デバイス削除時に解放される */
devm_kmalloc(&dev, size, GFP_KERNEL)

/* I/Oメモリ */
devm_ioremap(&dev, phys_addr, size)
devm_ioremap_resource(&dev, resource)
devm_platform_ioremap_resource(pdev, index)  /* インデックスでリソースをマップ */

/* IRQ */
devm_request_irq(&dev, irq, handler, flags, name, data)
devm_request_threaded_irq(...)

/* クロック */
devm_clk_get(&dev, "clock-name")
devm_clk_get_enabled(&dev, "clock-name")  /* 取得と有効化を1回の呼び出しで */

/* GPIO */
devm_gpiod_get(&dev, "reset", GPIOD_OUT_LOW)

/* レギュレータ */
devm_regulator_get(&dev, "vdd")

/* DMA */
devm_dma_request_chan(&dev, "tx")

/* 独自のクリーンアップを登録する */
devm_add_action(&dev, my_cleanup_fn, my_data);
devm_add_action_or_reset(&dev, my_cleanup_fn, my_data);
```

### なぜdevm_が重要か {#why-devm_-matters}

```c
/* devmなし: エラーパスは悪夢である */
static int probe(struct platform_device *pdev)
{
    dev->base = ioremap(res->start, resource_size(res));
    if (!dev->base) { return -ENOMEM; }

    ret = request_irq(irq, handler, 0, "dev", dev);
    if (ret) {
        iounmap(dev->base);  /* ioremapを取り消さなければならない */
        return ret;
    }

    dev->clk = clk_get(&pdev->dev, "myclk");
    if (IS_ERR(dev->clk)) {
        free_irq(irq, dev);   /* request_irqを取り消さなければならない */
        iounmap(dev->base);   /* ioremapを取り消さなければならない */
        return PTR_ERR(dev->clk);
    }
    /* ... */
}

/* devmあり: エラー時は単にreturnし、devmがクリーンアップを行う */
static int probe(struct platform_device *pdev)
{
    dev->base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(dev->base)) return PTR_ERR(dev->base);

    ret = devm_request_irq(&pdev->dev, irq, handler, 0, "dev", dev);
    if (ret) return ret;

    dev->clk = devm_clk_get(&pdev->dev, "myclk");
    if (IS_ERR(dev->clk)) return PTR_ERR(dev->clk);
    /* ... */
}
```

## Device Treeのバインディング {#device-tree-binding}

Device Treeはハードウェアのトポロジーを記述する。`compatible`プロパティを持つ各ノードは、ドライバとマッチさせることができる。

```dts
/* Device Treeノードの例 */
/ {
    soc {
        mydev: mydev@10000000 {
            compatible = "vendor,mydev-v1";
            reg = <0x10000000 0x1000>;  /* I/Oメモリ: ベース、サイズ */
            interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>;
            clocks = <&ccu CLK_MYDEV>;
            clock-names = "mydev_clk";
            reset-gpios = <&gpio 10 GPIO_ACTIVE_LOW>;
            status = "okay";
        };
    };
};
```

platformバスは、カーネルのブート時にこのノードから自動的に`platform_device`を作成する。ドライバの`of_match_table`が`compatible`と比較される。

### Device Treeのプロパティを読む {#reading-device-tree-properties}

```c
static int mydev_probe(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;
    u32 frequency;
    bool active_high;

    /* 単純な整数プロパティを読む */
    if (of_property_read_u32(np, "clock-frequency", &frequency))
        frequency = 100000;  /* デフォルト */

    /* 真偽値プロパティを読む */
    active_high = of_property_read_bool(np, "active-high");

    /* 名前付きのGPIOを取得する */
    struct gpio_desc *reset_gpio = devm_gpiod_get(&pdev->dev, "reset",
                                                   GPIOD_OUT_LOW);

    /* compatible文字列に対応するデータを取得する */
    const struct mydev_config *cfg =
        of_device_get_match_data(&pdev->dev);
}
```

## probeの延期 {#deferring-probe}

リソースの準備がまだ整っていない場合（例: クロックドライバがまだprobeしていない）:

```c
static int mydev_probe(struct platform_device *pdev)
{
    dev->clk = devm_clk_get(&pdev->dev, "myclk");
    if (IS_ERR(dev->clk)) {
        if (PTR_ERR(dev->clk) == -EPROBE_DEFER)
            return -EPROBE_DEFER;  /* clkが利用可能になったら後でリトライする */
        return PTR_ERR(dev->clk);
    }
}
```

`-EPROBE_DEFER`は、他のいずれかのドライバがprobeを完了したときにこのドライバをリトライするよう、probeの基盤に伝える。

## プラットフォームデバイスの観察 {#observing-platform-devices}

```bash
# プラットフォームデバイスを一覧表示
ls /sys/bus/platform/devices/

# プラットフォームデバイスにbindされたドライバを確認
cat /sys/bus/platform/devices/mydev.0/driver/module/name

# リソース（I/Oメモリ、IRQ）を確認
cat /sys/bus/platform/devices/mydev.0/resource
# 0x10000000 0x10000fff 0x200  ← 開始 終了 フラグ

# 強制的に再bind
echo mydev.0 > /sys/bus/platform/drivers/mydev/unbind
echo mydev.0 > /sys/bus/platform/drivers/mydev/bind

# probeのメッセージをカーネルログで確認
dmesg | grep mydev
```

## 参考文献 {#further-reading}

- [Linuxデバイスモデル](device-model.md) — struct device、バス、ドライバの内部
- [キャラクタデバイスとmiscデバイス](chardev.md) — /dev経由でデバイスを公開する
- `Documentation/driver-api/driver-model/platform.rst`
- `Documentation/devicetree/bindings/` — DTバインディングのドキュメント
