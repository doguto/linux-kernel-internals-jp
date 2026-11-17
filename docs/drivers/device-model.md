# Linuxデバイスモデル {#linux-device-model}

> カーネルのすべてのデバイスを組織するオブジェクト指向の枠組み

## なぜデバイスモデルが存在するか {#why-the-device-model-exists}

2.6より前は、各バスサブシステム（PCI、USB、SCSI）が、電源管理、ホットプラグ、sysfsをそれぞれ独立に管理していた。デバイスモデル（2.5でPatrick Mochelが導入[（LWN）](https://lwn.net/Articles/31185/)）は、これを単一の枠組みに統一した。

- **電源管理**: suspend/resumeがデバイスツリーを順序どおりに辿る
- **ホットプラグ**: デバイスの追加/削除について一貫したuevent通知
- **sysfs**: すべてのデバイスとドライバが自動的に`/sys/`に現れる（sysfsは2.5サイクルの間にPatrick Mochelが書き、Linux 2.6.0とともに出荷された[（kernel docs）](https://www.kernel.org/doc/html/latest/filesystems/sysfs.html)）
- **参照カウント**: デバイスは必要とされる間だけちょうど存在する

## kobject: 基底オブジェクト {#kobject-the-base-object}

sysfsに現れるすべてのカーネルオブジェクトは`kobject`を持つ。

```c
/* include/linux/kobject.h */
struct kobject {
    const char          *name;        /* sysfs内のディレクトリ名 */
    struct list_head     entry;       /* 兄弟のリスト */
    struct kobject      *parent;      /* sysfs階層内の親 */
    struct kset         *kset;        /* 所属するセット */
    const struct kobj_type *ktype;    /* sysfs操作を持つ型 */
    struct kernfs_node  *sd;          /* sysfsのディレクトリノード */
    struct kref          kref;        /* 参照カウント */
    unsigned int state_initialized:1;
    unsigned int state_in_sysfs:1;
    unsigned int state_add_uevent_sent:1;
    unsigned int state_remove_uevent_sent:1;
    unsigned int uevent_suppress:1;
};
```

参照カウント:
```c
kobject_get(kobj);    /* 参照カウントを増やす */
kobject_put(kobj);    /* 減らす。ゼロになるとktype->release()を呼ぶ */
```

## struct device: 汎用的なデバイス {#struct-device-the-universal-device}

```c
/* include/linux/device.h */
struct device {
    struct kobject          kobj;          /* 基底のkobject — 最初のフィールド */
    struct device           *parent;       /* 親デバイス */
    struct device_private   *p;            /* ドライバコアのプライベートデータ */

    const char              *init_name;    /* 初期のデバイス名 */
    const struct device_type *type;        /* デバイスの型 */

    struct bus_type         *bus;          /* このデバイスが載っているバス */
    struct device_driver    *driver;       /* デバイスに割り当てられたドライバ */
    void                    *driver_data;  /* ドライバのプライベートデータ */

    struct dev_links_info   links;         /* supplier/consumerのリンク */
    struct dev_pm_info      power;         /* 電源管理の状態 */
    struct dev_pm_domain    *pm_domain;

    struct device_node      *of_node;      /* Device Treeのノード */
    struct fwnode_handle    *fwnode;       /* ファームウェアノード（DTまたはACPI） */

    dev_t                   devt;          /* major:minor（該当する場合） */
    struct class            *class;        /* クラス（input、block、net...） */

    const struct attribute_group **groups; /* sysfsの属性グループ */
    void (*release)(struct device *dev);   /* 参照カウントが0になると呼ばれる */

    struct iommu_group      *iommu_group;
    /* ... */
};
```

主な操作:
```c
device_initialize(dev);          /* kobjectを初期化する */
device_add(dev);                 /* sysfsに登録し、ueventを送る */
device_register(dev);            /* = initialize + add */
device_unregister(dev);          /* sysfsから削除する */

get_device(dev);                 /* 参照を取得する */
put_device(dev);                 /* 参照を解放する */

dev_set_drvdata(dev, data);      /* ドライバのプライベートポインタを保存する */
dev_get_drvdata(dev);            /* それを取り出す */
```

## struct bus_type: バスの抽象化 {#struct-bus_type-the-bus-abstraction}

```c
struct bus_type {
    const char          *name;          /* "pci", "usb", "platform", "i2c" */
    const char          *dev_name;      /* 自動命名されるデバイスの書式 */
    struct device       *dev_root;      /* バスのルートデバイス */

    const struct attribute_group **bus_groups;
    const struct attribute_group **dev_groups;
    const struct attribute_group **drv_groups;

    /* ドライバがこのデバイスを扱うかどうかを確認するために呼ばれる */
    int (*match)(struct device *dev, struct device_driver *drv);
    /* ユーザー空間から/sys/.../ueventへの書き込みによって呼ばれる */
    int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
    /* デバイスがドライバにbindされる直前に呼ばれる */
    int (*probe)(struct device *dev);
    /* PM用の順序付けられたデバイス一覧 */
    int (*remove)(struct device *dev);
    void (*shutdown)(struct device *dev);

    int (*online)(struct device *dev);
    int (*offline)(struct device *dev);

    int (*suspend)(struct device *dev, pm_message_t state);
    int (*resume)(struct device *dev);

    int (*num_vf)(struct device *dev);
    int (*dma_configure)(struct device *dev);
    void (*dma_cleanup)(struct device *dev);

    const struct dev_pm_ops *pm;
    const struct iommu_ops  *iommu_ops;
    struct subsys_private   *p;         /* バスコアのプライベートデータ */
    struct lock_class_key   lock_key;
    bool need_parent_lock;
};
```

```c
/* バスの型を登録する */
int bus_register(struct bus_type *bus);
void bus_unregister(struct bus_type *bus);

/* バス上のデバイスを走査する */
bus_for_each_dev(bus, start, data, fn);
bus_for_each_drv(bus, start, data, fn);
```

## struct device_driver {#struct-device_driver}

```c
struct device_driver {
    const char              *name;         /* ドライバ名 */
    struct bus_type         *bus;
    struct module           *owner;        /* THIS_MODULE */
    const char              *mod_name;

    bool                     suppress_bind_attrs;
    enum probe_type          probe_type;   /* 同期、非同期、force_sync */

    const struct of_device_id   *of_match_table; /* Device Treeのマッチ */
    const struct acpi_device_id *acpi_match_table;

    int (*probe)(struct device *dev);      /* ドライバをデバイスにbindする */
    void (*sync_state)(struct device *dev);
    int (*remove)(struct device *dev);     /* unbind */
    void (*shutdown)(struct device *dev);
    int (*suspend)(struct device *dev, pm_message_t state);
    int (*resume)(struct device *dev);
    const struct attribute_group **groups;
    const struct attribute_group **dev_groups;

    const struct dev_pm_ops *pm;
    void (*coredump)(struct device *dev);

    struct driver_private *p;
};
```

## matchとprobeの流れ {#the-match-and-probe-flow}

デバイスが登録される（あるいはドライバが登録される）と、バスはすべてのドライバ（あるいはデバイス）を走査して`bus->match()`を呼ぶ。マッチすると`driver_probe_device()`が呼ばれる。

```c
/* drivers/base/dd.c: 簡略化 */
static int driver_probe_device(struct device_driver *drv, struct device *dev)
{
    int ret;

    /* すでにbindされているか確認する */
    if (dev->driver)
        return 0;

    /* バスレベルの許可チェック */
    if (drv->bus->match && !drv->bus->match(dev, drv))
        return 0;

    /* デバイスとドライバを結びつける */
    dev->driver = drv;

    /* バスのprobeを呼ぶ（通常はそれがdriver->probeを呼ぶ） */
    if (dev->bus->probe)
        ret = dev->bus->probe(dev);
    else if (drv->probe)
        ret = drv->probe(dev);

    if (ret) {
        dev->driver = NULL;
        return ret;
    }

    /* 成功: デバイスはbindされた */
    driver_bound(dev);
    return 0;
}
```

## sysfs属性 {#sysfs-attributes}

ドライバとデバイスは、sysfsに属性（ファイル）を公開する。

```c
/* 単純なデバイス属性 */
static ssize_t temperature_show(struct device *dev,
                                 struct device_attribute *attr,
                                 char *buf)
{
    struct my_device *mydev = dev_get_drvdata(dev);
    return sysfs_emit(buf, "%d\n", read_temperature(mydev));
}

static ssize_t setpoint_store(struct device *dev,
                               struct device_attribute *attr,
                               const char *buf, size_t count)
{
    struct my_device *mydev = dev_get_drvdata(dev);
    int val;

    if (kstrtoint(buf, 10, &val) < 0)
        return -EINVAL;
    set_setpoint(mydev, val);
    return count;
}

/* DEVICE_ATTR_RO: 読み出し専用、名前 = "temperature" */
static DEVICE_ATTR_RO(temperature);

/* DEVICE_ATTR_WO: 書き込み専用 */
/* DEVICE_ATTR_RW: 読み書き可能 */
static DEVICE_ATTR_RW(setpoint);

/* それらをグループにまとめる */
static struct attribute *mydev_attrs[] = {
    &dev_attr_temperature.attr,
    &dev_attr_setpoint.attr,
    NULL,
};
ATTRIBUTE_GROUPS(mydev);  /* mydev_groups[]を作成する */

/* ドライバのprobe内で: */
device_add_groups(dev, mydev_groups);
```

```bash
# sysfsでの結果
cat /sys/bus/platform/devices/mydev.0/temperature
# 45
echo 50 > /sys/bus/platform/devices/mydev.0/setpoint
```

## uevent: ホットプラグ通知 {#uevent-hotplug-notification}

デバイスが追加/削除されると、カーネルはユーザー空間にueventを送る（`udevd`/`systemd-udevd`が受け取る）。

```bash
# ueventをリアルタイムで監視する
udevadm monitor --kernel --udev

# KERNEL: add@/devices/pci0000:00/0000:00:14.0/usb1/1-1
# ACTION=add
# DEVPATH=/devices/pci0000:00/...
# SUBSYSTEM=usb
# ...

# ネットワークインターフェースの名前を変更するルール
# /etc/udev/rules.d/70-persistent-net.rules:
# SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="aa:bb:cc:dd:ee:ff", NAME="eth0"
```

カーネルの`kobject_uevent()`はnetlinkメッセージを送る。udevdはそれを受け取ってルールを実行する。

## 電源管理: dev_pm_ops {#power-management-dev_pm_ops}

```c
static const struct dev_pm_ops my_pm_ops = {
    .suspend  = my_suspend,   /* 状態を保存し、電力を下げる */
    .resume   = my_resume,    /* 状態を復元する */
    .freeze   = my_freeze,    /* ハイバネーションのスナップショット用 */
    .thaw     = my_thaw,
    .restore  = my_restore,
    /* ランタイムPM: */
    .runtime_suspend = my_runtime_suspend,  /* アイドル時に自動サスペンド */
    .runtime_resume  = my_runtime_resume,
};

/* ドライバ内で: */
static int my_suspend(struct device *dev)
{
    struct my_device *mydev = dev_get_drvdata(dev);
    /* レジスタを保存し、クロックを止めるなど */
    return 0;
}
```

```c
/* デバイスのランタイムPMを有効化する */
pm_runtime_enable(dev);
pm_runtime_set_active(dev);
pm_runtime_use_autosuspend(dev);
pm_runtime_set_autosuspend_delay(dev, 1000);  /* 1秒アイドル → サスペンド */

/* デバイス操作の中で: */
pm_runtime_get_sync(dev);   /* 使用カウントを増やし、サスペンド中ならresumeする */
/* ... デバイスを使う ... */
pm_runtime_put_autosuspend(dev);  /* 減らし、自動サスペンドのタイマーが始まる */
```

## 参考文献 {#further-reading}

- [プラットフォームドライバ](platform-driver.md) — 最も一般的な組み込みドライバのパターン
- [キャラクタデバイスとmiscデバイス](chardev.md) — /dev経由でデバイスをユーザー空間に公開する
- `Documentation/driver-api/driver-model/` — カーネルのドライバモデルのドキュメント
- `drivers/base/` — デバイスモデルのコア実装
