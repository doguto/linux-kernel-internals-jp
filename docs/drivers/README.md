# デバイスドライバ {#device-drivers}

> ハードウェアがどのようにしてユーザー空間から使えるようになるか

## 概要 {#overview}

デバイスドライバは、ハードウェアとカーネルの他の部分とを橋渡しする。Linuxのドライバモデルは、これを次の要素で構成する。

- **デバイス**（`struct device`）— ハードウェアまたは論理デバイスを表す
- **バス**（`struct bus_type`）— 通信チャネル（PCI、USB、I2C、platform）
- **ドライバ**（`struct device_driver`）— 特定のデバイスを扱うコード
- **クラス**（`struct class`）— 似た振る舞いをするデバイスのグループ（例: "input"、"net"）

```
sysfs (/sys/bus/pci/devices/0000:00:1f.2)
         │
         ▼
struct device ←────── struct pci_dev (bus-specific extension)
         │
    bus_type.match()
         │
         ▼
struct device_driver ←── struct pci_driver (driver-specific extension)
         │
    driver.probe()
         │
         ▼
    Hardware
```

## このセクションのページ {#pages-in-this-section}

| ページ | 扱う内容 |
|------|----------------|
| [Linuxデバイスモデル](device-model.md) | struct device、バス、ドライバ、kobject、sysfs |
| [プラットフォームドライバ](platform-driver.md) | platformバス、probe/remove、devm_、Device Tree |
| [キャラクタデバイスとmiscデバイス](chardev.md) | cdev、file_operations、ioctl、ドライバ側から見たmmap |
| [PCIドライバ](pci-driver.md) | 列挙、コンフィギュレーション空間、BAR、MSIの設定 |
| [I2CとSPI](i2c-spi.md) | 低速バスのドライバ、regmap、デバイスのアドレッシング |
| [事件簿](war-stories.md) | ドライバコアで実際に起きたバグとリグレッション: probe順序の競合、sysfsのデッドロック、fw_devlinkのデフォルト化 |
| [Device Tree](device-tree.md) | 自動検出できないバスのためのハードウェア記述 |

## クイックリファレンス {#quick-reference}

```bash
# すべてのデバイスとそのドライバを一覧表示
ls /sys/bus/pci/devices/
ls /sys/bus/platform/devices/

# デバイスのドライバを確認
cat /sys/bus/pci/devices/0000:00:1f.2/driver/module/name

# カーネルモジュールをロード
modprobe e1000e
lsmod | grep e1000

# ドライバを手動でbind/unbind
echo "0000:00:1f.2" > /sys/bus/pci/drivers/ahci/unbind
echo "0000:00:1f.2" > /sys/bus/pci/drivers/ahci/bind

# キャラクタデバイスのmajor/minor
ls -la /dev/null /dev/random
# crw-rw-rw- 1 root root 1, 3 ... /dev/null  ← major=1, minor=3
```
