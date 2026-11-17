# procfsとsysfs {#procfs-and-sysfs}

> /procと/sysがカーネルの状態をユーザー空間へどう公開するか

## /proc: プロセスファイルシステム {#proc-the-process-filesystem}

`/proc`は、カーネルとプロセスの情報を公開する仮想ファイルシステムである。`/proc`内のファイルにはディスク上の表現がなく、内容は読み込まれた時点でその場で生成される。`proc`ファイルシステムはUnixの伝統を受け継いだもので、Linuxの最初期のバージョンから存在している[(man page)](https://www.man7.org/linux/man-pages/man5/proc.5.html)。

```bash
# よく使われる /proc のファイル:
/proc/cpuinfo         # CPUの説明
/proc/meminfo         # メモリ統計
/proc/interrupts      # CPUごとのIRQ回数
/proc/net/dev         # ネットワークインターフェースの統計
/proc/sys/            # sysctlのチューニング可能なパラメータ

# プロセスごと（PIDごとに1ディレクトリ）:
/proc/<pid>/maps      # 仮想メモリ領域
/proc/<pid>/status    # タスクの状態
/proc/<pid>/fd/       # オープンされたファイルディスクリプタ（シンボリックリンク）
/proc/<pid>/cmdline   # コマンドライン引数
/proc/<pid>/mem       # プロセスのメモリ（ptraceが必要）
```

## /procエントリの作成 {#creating-a-proc-entry}

### 単純な単一値のファイル {#simple-single-value-file}

```c
#include <linux/proc_fs.h>
#include <linux/seq_file.h>

/* show関数: 読み込み時に呼ばれる */
static int mymod_show(struct seq_file *m, void *v)
{
    seq_printf(m, "value: %d\n", my_value);
    seq_printf(m, "name: %s\n", my_name);
    return 0;
}

/* write関数: 書き込み時に呼ばれる */
static ssize_t mymod_write(struct file *file, const char __user *buf,
                             size_t count, loff_t *ppos)
{
    char kbuf[32];
    if (copy_from_user(kbuf, buf, min(count, sizeof(kbuf) - 1)))
        return -EFAULT;
    kbuf[count] = '\0';
    kstrtoint(kbuf, 10, &my_value);
    return count;
}

/* ファイル操作: 単純な1ページのファイルにはsingle_openを使う */
static int mymod_open(struct inode *inode, struct file *file)
{
    return single_open(file, mymod_show, NULL);
}

static const struct proc_ops mymod_fops = {
    .proc_open    = mymod_open,
    .proc_read    = seq_read,
    .proc_write   = mymod_write,
    .proc_lseek   = seq_lseek,
    .proc_release = single_release,
};

/* init時に /proc/mymod を作成する */
static int __init mymod_init(void)
{
    proc_create("mymod", 0644, NULL, &mymod_fops);
    /* 親がNULL = /proc/ のルート */
    return 0;
}

static void __exit mymod_exit(void)
{
    remove_proc_entry("mymod", NULL);
}
```

### seq_file: 複数ページのイテレータ {#seq_file-multi-page-iterators}

多数のエントリを持つファイル（`/proc/net/tcp`など）では、seq_fileのイテレータAPIを使う:

```c
/* エントリのリストを順に処理する */
static void *mylist_seq_start(struct seq_file *m, loff_t *pos)
{
    return seq_list_start(&my_list, *pos);
}

static void *mylist_seq_next(struct seq_file *m, void *v, loff_t *pos)
{
    return seq_list_next(v, &my_list, pos);
}

static void mylist_seq_stop(struct seq_file *m, void *v)
{
    /* 必要なら後始末を行う（アンロックなど） */
}

static int mylist_seq_show(struct seq_file *m, void *v)
{
    struct my_entry *e = list_entry(v, struct my_entry, list);
    seq_printf(m, "%s %d %llu\n", e->name, e->count, e->bytes);
    return 0;
}

static const struct seq_operations mylist_seq_ops = {
    .start = mylist_seq_start,
    .next  = mylist_seq_next,
    .stop  = mylist_seq_stop,
    .show  = mylist_seq_show,
};

/* open: seq_fileをセットアップする */
static int mylist_open(struct inode *inode, struct file *file)
{
    return seq_open(file, &mylist_seq_ops);
}
```

`seq_file`はページ分割を自動的に処理する。出力が1ページに収まらない場合、カーネルは次の読み込みのために適切なオフセットで`start`/`next`/`show`を再度呼び出す。

### /procディレクトリ {#proc-directory}

```c
struct proc_dir_entry *mydir;

/* /proc/mymod/ ディレクトリを作成する */
mydir = proc_mkdir("mymod", NULL);

/* /proc/mymod/status を作成する */
proc_create("status", 0444, mydir, &status_fops);

/* /proc/mymod/stats を作成する（読み取り専用） */
proc_create_single("stats", 0444, mydir, stats_show);

/* exit時に削除する */
remove_proc_subtree("mymod", NULL);
```

## /procの内部: struct proc_dir_entry {#proc-internals-struct-proc_dir_entry}

```c
/* fs/proc/internal.h */
struct proc_dir_entry {
    /* inode番号: 連番で割り当てられる */
    unsigned int    low_ino;
    nlink_t         nlink;
    kuid_t          uid;
    kgid_t          gid;
    loff_t          size;

    const struct inode_operations *proc_iops;
    union {
        const struct proc_ops *proc_ops;
        const struct file_operations *proc_dir_ops;
    };
    const struct dentry_operations *proc_dops;
    union {
        const struct seq_operations *seq_ops;
        int (*single_show)(struct seq_file *, void *);
    };
    proc_write_t write;
    void         *data;
    unsigned int  state_size;
    unsigned int  len;
    char          name[];
};
```

`/proc`ファイルシステムは、パスが検索されたときにオンデマンドでinodeを作成する、通常のVFSファイルシステム（`proc_fs_type`）として実装されている。

## sysfs: デバイスとドライバの階層 {#sysfs-the-device-and-driver-hierarchy}

`sysfs`は、カーネルオブジェクト（kobject）とその属性をユーザー空間へエクスポートするためのRAMベースのファイルシステムとして、Patrick MochelによってLinux 2.6.0で導入された[(kernel docs)](https://www.kernel.org/doc/html/latest/filesystems/sysfs.html)。`/sys`にマウントされ、カーネルのデバイスモデルをディレクトリツリーとして公開する:

```
/sys/
├── bus/           (bus types: pci, usb, i2c, ...)
├── class/         (device classes: net, block, input, ...)
├── devices/       (device hierarchy mirroring hardware topology)
│   ├── pci0000:00/
│   │   └── 0000:00:1c.0/   (PCI bridge)
│   │       └── 0000:01:00.0/  (network card)
└── kernel/        (kernel parameters and statistics)
    ├── mm/transparent_hugepage/
    └── security/
```

## kobject: sysfsのオブジェクト {#kobject-the-sysfs-object}

`kobject`は、sysfs/デバイスモデル階層における基本的なオブジェクトである:

```c
/* include/linux/kobject.h */
struct kobject {
    const char          *name;
    struct list_head    entry;
    struct kobject      *parent;         /* 親kobject */
    struct kset         *kset;
    const struct kobj_type *ktype;
    struct kernfs_node  *sd;             /* sysfsのディレクトリノード */
    struct kref         kref;            /* 参照カウント */
    /* ... */
};

/* kobjectを作成する（sysfsディレクトリが作成される）: */
struct kobject *kobj;
kobj = kobject_create_and_add("myobject", kernel_kobj);
/* 作成されるもの: /sys/kernel/myobject/ */

/* 後始末: */
kobject_put(kobj);  /* 参照を減らす。0になると解放される */
```

## sysfs属性: sysfs内のファイル {#sysfs-attributes-files-in-sysfs}

```c
/* sysfs属性 = /sys/... 内の1つのファイル */
struct attribute {
    const char *name;
    umode_t     mode;   /* パーミッション */
};

/* デバイス属性（struct device）用: */
struct device_attribute {
    struct attribute  attr;
    ssize_t (*show)(struct device *dev, struct device_attribute *attr, char *buf);
    ssize_t (*store)(struct device *dev, struct device_attribute *attr,
                      const char *buf, size_t count);
};

/* デバイス属性を定義する: */
static ssize_t speed_show(struct device *dev,
                           struct device_attribute *attr, char *buf)
{
    struct mydev *d = dev_get_drvdata(dev);
    return sysfs_emit(buf, "%u\n", d->speed);
}

static ssize_t speed_store(struct device *dev,
                             struct device_attribute *attr,
                             const char *buf, size_t count)
{
    struct mydev *d = dev_get_drvdata(dev);
    unsigned int val;
    if (kstrtouint(buf, 10, &val))
        return -EINVAL;
    d->speed = val;
    return count;
}

static DEVICE_ATTR_RW(speed);   /* dev_attr_speedを作成する */
/* DEVICE_ATTR_RO(speed): 読み取り専用 */
/* DEVICE_ATTR_WO(speed): 書き込み専用 */
```

### 属性グループ {#attribute-groups}

ドライバは通常、属性グループ（複数の属性をまとめたもの）を登録する:

```c
/* 属性を宣言する */
static DEVICE_ATTR_RO(stats);
static DEVICE_ATTR_RW(enable);
static DEVICE_ATTR_RW(threshold);

/* それらをグループ化する */
static struct attribute *mydev_attrs[] = {
    &dev_attr_stats.attr,
    &dev_attr_enable.attr,
    &dev_attr_threshold.attr,
    NULL,
};

static const struct attribute_group mydev_attr_group = {
    .attrs = mydev_attrs,
    /* 任意: .name = "config" → /sys/.../config/ サブディレクトリを作成する */
};

static const struct attribute_group *mydev_attr_groups[] = {
    &mydev_attr_group,
    NULL,
};

/* デバイスに登録する（probe内で）: */
/* 方法1: 手動 */
device_add_group(&pdev->dev, &mydev_attr_group);

/* 方法2: 登録時にデバイスのdriver_data経由で */
struct device_driver mydev_driver = {
    .dev_groups = mydev_attr_groups,
    /* ... */
};
/* 属性はデバイスとともに自動的に作成/削除される */
```

## kernfs: sysfsのバックエンド {#kernfs-the-sysfs-backend}

sysfsは、汎用の仮想ファイルシステム基盤である`kernfs`の上に構築されている:

```c
/* fs/kernfs/kernfs-internal.h */
struct kernfs_node {
    atomic_t         count;
    atomic_t         active;
    struct kernfs_node *parent;
    const char       *name;
    struct rb_node   rb;

    const void       *ns;       /* namespaceごとのファイル用のnamespace */
    unsigned int     hash;

    union {
        struct kernfs_elem_dir    dir;      /* ディレクトリ */
        struct kernfs_elem_symlink symlink; /* シンボリックリンク */
        struct kernfs_elem_attr   attr;     /* 属性（ファイル） */
    };

    void     *priv;       /* kobject/属性へのポインタ */
    kuid_t    uid;
    kgid_t    gid;
    struct kernfs_iattrs *iattr;
};
```

`/sys`の属性は次の経路を通る: `kernfs_fop_read_iter → kernfs_seq_show → sysfs_kf_seq_show → attribute->show()`

## sysctl: sysfs風のAPIによる/proc/sys {#sysctl-procsys-via-sysfs-like-api}

```c
#include <linux/sysctl.h>

static int my_value = 42;
static int my_min = 0;
static int my_max = 1000;

static struct ctl_table mymod_table[] = {
    {
        .procname   = "my_value",
        .data       = &my_value,
        .maxlen     = sizeof(int),
        .mode       = 0644,
        .proc_handler = proc_dointvec_minmax,
        .extra1     = &my_min,
        .extra2     = &my_max,
    },
    { }  /* 番兵 */
};

static struct ctl_table_header *mymod_sysctl;

static int __init mymod_init(void)
{
    mymod_sysctl = register_sysctl("mymod", mymod_table);
    /* 作成されるもの: /proc/sys/mymod/my_value */
    return 0;
}

static void __exit mymod_exit(void)
{
    unregister_sysctl_table(mymod_sysctl);
}
```

## 参考文献 {#further-reading}

- [VFSオブジェクト](vfs-objects.md) — inode/dentryの基礎
- [Linuxデバイスモデル](../drivers/device-model.md) — kobject階層の詳細
- [sysctlリファレンス](../mm/sysctl-reference.md) — /proc/sys経由のカーネルのチューニング可能なパラメータ
- [Netlink](../net/netlink.md) — /procを超えたカーネルとユーザー空間のメッセージング
- `fs/proc/` — procfsの実装
- `fs/sysfs/`と`fs/kernfs/` — sysfs/kernfsの実装
- `include/linux/kobject.h`、`include/linux/device.h`
