# I2CとSPIのバスドライバ {#i2c-and-spi-bus-drivers}

> LinuxカーネルでI2CとSPIの周辺デバイス向けドライバを書く

## I2Cの概要 {#i2c-overview}

I2C（Inter-Integrated Circuit）は、低速の周辺機器（センサー、EEPROM、タッチコントローラ）をマイクロプロセッサに接続するための2線式シリアルプロトコルである。LinuxのI2Cサブシステムは、Greg Kroah-Hartmanのドライバモデルの作業を通じてメインラインカーネルに取り込まれ、その後Jean Delvare（元lm-sensorsプロジェクト）が共同でメンテナンスし、2005年9月に彼が単独のメンテナとなった[（LWN: I2C patches for 2.6.12-rc1）](https://lwn.net/Articles/130158/)。

```
物理的な配線:
  SDA (data)  ────────────────────────
  SCL (clock) ────────────────────────
              │       │       │
           Master    Dev1    Dev2
        (host CPU) (0x48) (0x50)
                     ↑       ↑
                  7ビットアドレス
```

### I2Cトランザクションの形式 {#i2c-transaction-format}

```
Start: SCLがHighの間にSDAが立ち下がる
  │
  └─ [7-bit address] [R/W bit] [ACK]
     │
     └─ [data byte 0] [ACK]
        [data byte 1] [ACK]
        ...
Stop: SCLがHighの間にSDAが立ち上がる
```

## I2Cクライアントドライバ {#i2c-client-driver}

### struct i2c_driver {#struct-i2c_driver}

```c
#include <linux/i2c.h>
#include <linux/module.h>

/* デバイス固有のプライベートデータ */
struct my_sensor {
    struct i2c_client *client;
    struct mutex       lock;
    int                irq;
};

/* Device Tree / ACPI用のマッチテーブル */
static const struct of_device_id my_sensor_of_match[] = {
    { .compatible = "vendor,my-sensor" },
    { /* 番兵 */ }
};
MODULE_DEVICE_TABLE(of, my_sensor_of_match);

static const struct i2c_device_id my_sensor_id[] = {
    { "my-sensor", 0 },
    { /* 番兵 */ }
};
MODULE_DEVICE_TABLE(i2c, my_sensor_id);

/* デバイスが見つかったときに呼ばれる */
static int my_sensor_probe(struct i2c_client *client)
{
    struct my_sensor *sensor;
    int ret;

    /* 必要な操作をデバイスがサポートしているか確認する */
    if (!i2c_check_functionality(client->adapter,
                                  I2C_FUNC_SMBUS_BYTE_DATA)) {
        dev_err(&client->dev, "SMBus byte operations required\n");
        return -EOPNOTSUPP;
    }

    sensor = devm_kzalloc(&client->dev, sizeof(*sensor), GFP_KERNEL);
    if (!sensor)
        return -ENOMEM;

    sensor->client = client;
    mutex_init(&sensor->lock);
    i2c_set_clientdata(client, sensor);

    /* デバイスIDレジスタを読んで検証する */
    ret = i2c_smbus_read_byte_data(client, REG_DEVICE_ID);
    if (ret < 0) {
        dev_err(&client->dev, "failed to read device ID: %d\n", ret);
        return ret;
    }
    if (ret != EXPECTED_DEVICE_ID) {
        dev_err(&client->dev, "unexpected device ID: 0x%02x\n", ret);
        return -ENODEV;
    }

    /* IIOまたはhwmonサブシステムに登録する */
    return my_sensor_register_iio(sensor);
}

static void my_sensor_remove(struct i2c_client *client)
{
    struct my_sensor *sensor = i2c_get_clientdata(client);
    my_sensor_unregister_iio(sensor);
}

static struct i2c_driver my_sensor_driver = {
    .driver = {
        .name         = "my-sensor",
        .of_match_table = my_sensor_of_match,
    },
    .probe    = my_sensor_probe,
    .remove   = my_sensor_remove,
    .id_table = my_sensor_id,
};
module_i2c_driver(my_sensor_driver);
/* 展開結果: i2c_add/del_driverを使うmodule_init + module_exit */
```

## I2Cの転送関数 {#i2c-transfer-functions}

### SMBus（単純なデバイスで一般的） {#smbus-common-for-simple-devices}

```c
/* レジスタアドレスの1バイトを読み書きする: */
s32 i2c_smbus_read_byte_data(struct i2c_client *client, u8 reg);
s32 i2c_smbus_write_byte_data(struct i2c_client *client, u8 reg, u8 value);

/* 16ビットのワードを読み書きする: */
s32 i2c_smbus_read_word_data(struct i2c_client *client, u8 reg);
s32 i2c_smbus_write_word_data(struct i2c_client *client, u8 reg, u16 value);

/* データのブロックを読む: */
s32 i2c_smbus_read_i2c_block_data(struct i2c_client *client, u8 reg,
                                   u8 length, u8 *values);
/* valuesには、レジスタ`reg`から始まる`length`バイトが格納される */

/* 例: レジスタ0x28から加速度センサーのデータを6バイト読む: */
u8 data[6];
ret = i2c_smbus_read_i2c_block_data(client, 0x28, 6, data);
```

### 生のI2Cメッセージ（複雑なプロトコル向け） {#raw-i2c-messages-for-complex-protocols}

```c
/* SMBusの慣習に従わないデバイス向け: */
int i2c_transfer(struct i2c_adapter *adap, struct i2c_msg *msgs, int num);

/* 例: レジスタアドレスを書き込んでから複数バイトを読む
   （一般的なレジスタアクセスのパターン）: */
static int my_read_regs(struct i2c_client *client, u8 reg,
                         u8 *buf, size_t len)
{
    struct i2c_msg msgs[2] = {
        {
            /* 書き込みフェーズ: レジスタアドレスを送る */
            .addr  = client->addr,
            .flags = 0,              /* 書き込み */
            .len   = 1,
            .buf   = &reg,
        },
        {
            /* 読み出しフェーズ: `len`バイトを読む */
            .addr  = client->addr,
            .flags = I2C_M_RD,      /* 読み出し */
            .len   = len,
            .buf   = buf,
        },
    };
    return i2c_transfer(client->adapter, msgs, 2);
}
```

## regmap: レジスタアクセスの抽象化 {#regmap-register-access-abstraction}

`regmap`は、I2C、SPI、MMIOなどのバスをまたいで、レジスタアクセスのための統一されたAPIを提供する。

```c
#include <linux/regmap.h>

/* レジスタマップの設定を定義する: */
static const struct regmap_config my_regmap_config = {
    .reg_bits   = 8,    /* レジスタアドレスの幅 */
    .val_bits   = 8,    /* レジスタ値の幅 */
    .max_register = 0xFF,

    /* 任意: 読み出し専用または低速なレジスタのキャッシュ */
    .cache_type = REGCACHE_RBTREE,

    /* 読み出し可能/書き込み可能なレジスタのマスク: */
    .readable_reg = my_readable_reg,
    .writeable_reg = my_writeable_reg,
    .volatile_reg  = my_volatile_reg,  /* キャッシュされない */
};

static int my_sensor_probe(struct i2c_client *client)
{
    struct my_sensor *sensor;
    struct regmap *regmap;

    /* I2Cを裏付けとするregmapを作成する: */
    regmap = devm_regmap_init_i2c(client, &my_regmap_config);
    if (IS_ERR(regmap))
        return PTR_ERR(regmap);

    sensor->regmap = regmap;
    /* ... */
}

/* レジスタアクセスにregmapを使う: */
unsigned int val;
regmap_read(sensor->regmap, REG_STATUS, &val);
regmap_write(sensor->regmap, REG_CONFIG, 0x01);

/* 読み出し-変更-書き込み（アトミック）: */
regmap_update_bits(sensor->regmap, REG_CONFIG,
                   BIT(3) | BIT(2),   /* マスク */
                   BIT(3));           /* 値: ビット3をセット、ビット2をクリア */

/* 連続する複数のレジスタを一括で読む: */
u8 buf[6];
regmap_bulk_read(sensor->regmap, REG_DATA_START, buf, 6);
```

## SPIの概要 {#spi-overview}

SPI（Serial Peripheral Interface）は同期式の4線プロトコルである。LinuxのSPIサブシステムはDavid Brownellによって書かれ、Linux 2.6.16でマージされた[（commit 8ae12a0d）](https://git.kernel.org/linus/8ae12a0d85987dc138f8c944cb78a92bf466cea0)。

```
SCLK  ──────────────────── クロック（マスターが駆動）
MOSI  ──────────────────── master-out slave-in
MISO  ──────────────────── master-in slave-out
CS/SS ──────────────────── チップセレクト（デバイスごとに1本、アクティブLow）

I2Cとの違い:
  - 全二重（TXとRXを同時に行う）
  - アドレッシングなし: CS線でデバイスを選択する
  - ACKなし: スレーブが受信したかどうかをマスターは知ることがない
  - より高速（I2Cの3.4 MHzに対して100 MHz以上まで）
```

## SPIデバイスドライバ {#spi-device-driver}

```c
#include <linux/spi/spi.h>

struct my_spi_dev {
    struct spi_device *spi;
    struct mutex       lock;
};

static const struct of_device_id my_spi_of_match[] = {
    { .compatible = "vendor,my-spi-sensor" },
    { }
};

static int my_spi_probe(struct spi_device *spi)
{
    struct my_spi_dev *dev;

    /* SPIのパラメータを設定する: */
    spi->bits_per_word = 8;
    spi->mode          = SPI_MODE_0;    /* CPOL=0, CPHA=0 */
    spi->max_speed_hz  = 10000000;      /* 10 MHz */
    spi_setup(spi);                      /* 設定を適用する */

    dev = devm_kzalloc(&spi->dev, sizeof(*dev), GFP_KERNEL);
    dev->spi = spi;
    spi_set_drvdata(spi, dev);

    return my_spi_dev_init(dev);
}

static struct spi_driver my_spi_driver = {
    .driver = {
        .name         = "my-spi-sensor",
        .of_match_table = my_spi_of_match,
    },
    .probe = my_spi_probe,
};
module_spi_driver(my_spi_driver);
```

## SPIの転送関数 {#spi-transfer-functions}

```c
/* 単純な同期読み書き: */
u8 tx_buf[4], rx_buf[4];

/* 全二重転送（TXとRXを同時に）: */
struct spi_transfer t = {
    .tx_buf = tx_buf,
    .rx_buf = rx_buf,
    .len    = 4,
    .speed_hz = 5000000,  /* デフォルトの速度を上書きする */
};
struct spi_message m;
spi_message_init(&m);
spi_message_add_tail(&t, &m);
ret = spi_sync(spi, &m);

/* 便利なラッパー: */
spi_write(spi, tx_buf, 4);           /* TXのみ */
spi_read(spi, rx_buf, 4);            /* RXのみ（0x00を送信する） */
spi_write_then_read(spi, tx_buf, 2, rx_buf, 4);  /* 間はCSをLowに保持する */

/* SPI用のregmap（I2Cと同じAPI!）: */
regmap = devm_regmap_init_spi(spi, &my_regmap_config);
regmap_read(regmap, REG_ID, &val);
```

## Device Treeによるインスタンス化 {#device-tree-instantiation}

```dts
/* ボードのDTS: */
&i2c1 {
    status = "okay";
    clock-frequency = <400000>;  /* 400 kHzのファストモード */

    my_sensor: sensor@48 {
        compatible = "vendor,my-sensor";
        reg = <0x48>;            /* I2Cアドレス */
        vdd-supply = <&ldo3>;    /* 電源 */
        interrupt-parent = <&gpio0>;
        interrupts = <15 IRQ_TYPE_EDGE_FALLING>;
    };
};

&spi0 {
    status = "okay";
    cs-gpios = <&gpio1 5 GPIO_ACTIVE_LOW>;

    my_spi_sensor: sensor@0 {
        compatible = "vendor,my-spi-sensor";
        reg = <0>;              /* チップセレクト0 */
        spi-max-frequency = <10000000>;
        spi-cpol;               /* CPOL=1 */
        spi-cpha;               /* CPHA=1 → 合わせてSPI_MODE_3 */
    };
};
```

## I2C/SPIのデバッグ {#debugging-i2cspi}

```bash
# I2Cバスを一覧表示:
ls /dev/i2c-*
i2cdetect -l

# I2Cバス1をスキャンしてデバイスを探す:
i2cdetect -y 1
#      0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
# 40: -- -- -- -- -- -- -- -- 48 -- -- -- -- -- -- --
# （アドレス0x48でデバイスが見つかった）

# アドレス0x48のデバイスからレジスタ0x00を読む:
i2cget -y 1 0x48 0x00

# レジスタ0x10に0x01を書き込む:
i2cset -y 1 0x48 0x10 0x01

# I2Cのトラフィックをデコードする（ハードウェアが対応していれば）:
# /sys/kernel/debug/i2c-0/を使う
cat /sys/kernel/debug/i2c-0/timing

# SPIのトレース:
echo 1 > /sys/kernel/debug/dynamic_debug/control
# またはspidev_testを使う:
spidev_test -D /dev/spidev0.0 -s 1000000 -p '\x01\x02\x03'

# regmapのdebugfs:
ls /sys/kernel/debug/regmap/
cat /sys/kernel/debug/regmap/my-sensor-0-0048/registers
# 00: 01
# 01: 0f
# ...

# I2Cトランザクションをトレースする:
bpftrace -e '
kprobe:i2c_transfer {
    printf("i2c_transfer: adapter=%s num_msgs=%d\n",
           ((struct i2c_adapter *)arg0)->name, (int)arg2);
}'
```

## 参考文献 {#further-reading}

- [プラットフォームドライバ](platform-driver.md) — MMIOでレジスタがマップされたデバイス
- [Device Tree](device-tree.md) — I2C/SPIデバイスのDTSバインディング
- [デバイスモデル](device-model.md) — バス/デバイス/ドライバの枠組み
- [DMA API](../iommu/dma-api.md) — SPIコントローラのためのDMA
- `drivers/i2c/` — I2Cサブシステム
- `drivers/spi/` — SPIサブシステム
- `include/linux/regmap.h` — regmap API
- `Documentation/driver-api/i2c.rst` — I2Cドライバのガイド
