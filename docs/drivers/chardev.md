# キャラクタデバイスとmiscデバイス {#character-and-misc-devices}

> /dev経由でドライバの機能をユーザー空間に公開する

## キャラクタデバイス {#character-devices}

キャラクタデバイス（cdev）は、ドライバへのバイトストリームのインターフェースを提供する`/dev`内の特殊ファイルである。ブロックデバイスとは異なり、キャラクタデバイスはデータをページキャッシュにバッファリングしない。I/Oは直接ドライバに届く。2段階の`alloc_chrdev_region` / `cdev_add`インターフェースは、拡張された`dev_t`型をサポートするために、Linux 2.6.0の直前にAl Viroによって導入された。古い`register_chrdev()`は互換性のための薄いラッパーとして残っている[（LWN）](https://lwn.net/Articles/195805/)。

```
/dev/mydevice (major=240, minor=0)
        │
        ▼
 struct cdev ────────► struct file_operations
        │                   .open
        │                   .read
        │                   .write
        │                   .unlocked_ioctl
        │                   .mmap
        │                   .release
        ▼
   driver code
```

## major/minor番号の割り当て {#allocating-majorminor-numbers}

```c
/* 静的な割り当て（Documentation/admin-guide/devices.txtで予約） */
int ret = register_chrdev_region(MKDEV(240, 0), 1, "mydev");

/* 動的な割り当て（推奨）: カーネルがmajorを割り当てる */
dev_t devt;
int ret = alloc_chrdev_region(&devt, 0, /* 最初のminor */
                               1,        /* 個数 */
                               "mydev");
int major = MAJOR(devt);
int minor = MINOR(devt);

/* 終了時に解放する */
unregister_chrdev_region(devt, 1);
```

## struct cdev {#struct-cdev}

```c
#include <linux/cdev.h>

struct cdev {
    struct kobject          kobj;
    struct module          *owner;
    const struct file_operations *ops;
    struct list_head        list;
    dev_t                   dev;         /* major:minor */
    unsigned int            count;       /* minorの個数 */
};
```

## struct file_operations {#struct-file_operations}

```c
/* include/linux/fs.h */
struct file_operations {
    struct module *owner;
    loff_t (*llseek)(struct file *, loff_t, int);
    ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
    ssize_t (*read_iter)(struct kiocb *, struct iov_iter *);  /* 推奨 */
    ssize_t (*write_iter)(struct kiocb *, struct iov_iter *);
    int (*iopoll)(struct kiocb *, struct io_comp_batch *, unsigned int flags);
    int (*iterate_shared)(struct file *, struct dir_context *);
    __poll_t (*poll)(struct file *, struct poll_table_struct *);
    long (*unlocked_ioctl)(struct file *, unsigned int, unsigned long);
    long (*compat_ioctl)(struct file *, unsigned int, unsigned long);
    int (*mmap)(struct file *, struct vm_area_struct *);
    int (*open)(struct inode *, struct file *);
    int (*flush)(struct file *, fl_owner_t id);
    int (*release)(struct inode *, struct file *);
    int (*fsync)(struct file *, loff_t, loff_t, int datasync);
    int (*fasync)(int, struct file *, int);
    /* ... */
};
```

## 完全な最小限のキャラクタドライバ {#a-complete-minimal-character-driver}

```c
/* drivers/mychardev/mychardev.c */
#include <linux/module.h>
#include <linux/cdev.h>
#include <linux/fs.h>
#include <linux/uaccess.h>
#include <linux/slab.h>

#define DEVICE_NAME "mychardev"
#define BUF_SIZE    4096

struct mychardev {
    struct cdev cdev;
    char    *buf;
    size_t   buf_len;
    struct mutex lock;
};

static struct mychardev mydev;
static dev_t devt;

static int mydev_open(struct inode *inode, struct file *filp)
{
    /* 必要ならファイルごとの状態をfilp->private_dataに保存する */
    struct mychardev *dev = container_of(inode->i_cdev,
                                          struct mychardev, cdev);
    filp->private_data = dev;
    return 0;
}

static ssize_t mydev_read(struct file *filp, char __user *buf,
                           size_t count, loff_t *ppos)
{
    struct mychardev *dev = filp->private_data;
    ssize_t ret;

    mutex_lock(&dev->lock);

    if (*ppos >= dev->buf_len) {
        ret = 0;  /* EOF */
        goto out;
    }

    count = min(count, dev->buf_len - (size_t)*ppos);
    if (copy_to_user(buf, dev->buf + *ppos, count)) {
        ret = -EFAULT;
        goto out;
    }

    *ppos += count;
    ret = count;

out:
    mutex_unlock(&dev->lock);
    return ret;
}

static ssize_t mydev_write(struct file *filp, const char __user *buf,
                            size_t count, loff_t *ppos)
{
    struct mychardev *dev = filp->private_data;
    ssize_t ret;

    if (count > BUF_SIZE)
        return -EINVAL;

    mutex_lock(&dev->lock);

    if (copy_from_user(dev->buf, buf, count)) {
        ret = -EFAULT;
        goto out;
    }

    dev->buf_len = count;
    *ppos = 0;
    ret = count;

out:
    mutex_unlock(&dev->lock);
    return ret;
}

static int mydev_release(struct inode *inode, struct file *filp)
{
    return 0;
}

static const struct file_operations mydev_fops = {
    .owner   = THIS_MODULE,
    .open    = mydev_open,
    .read    = mydev_read,
    .write   = mydev_write,
    .release = mydev_release,
    .llseek  = default_llseek,  /* f_posを使う */
};

static int __init mydev_init(void)
{
    int ret;

    /* デバイス番号を割り当てる */
    ret = alloc_chrdev_region(&devt, 0, 1, DEVICE_NAME);
    if (ret)
        return ret;

    /* cdevを割り当てて初期化する */
    cdev_init(&mydev.cdev, &mydev_fops);
    mydev.cdev.owner = THIS_MODULE;

    /* バッファを割り当てる */
    mydev.buf = kzalloc(BUF_SIZE, GFP_KERNEL);
    if (!mydev.buf) {
        unregister_chrdev_region(devt, 1);
        return -ENOMEM;
    }

    mutex_init(&mydev.lock);

    /* cdevを登録する — これ以降open()が呼ばれうる */
    ret = cdev_add(&mydev.cdev, devt, 1);
    if (ret) {
        kfree(mydev.buf);
        unregister_chrdev_region(devt, 1);
        return ret;
    }

    pr_info("mychardev: major=%d minor=%d\n", MAJOR(devt), MINOR(devt));
    return 0;
}

static void __exit mydev_exit(void)
{
    cdev_del(&mydev.cdev);
    kfree(mydev.buf);
    unregister_chrdev_region(devt, 1);
}

module_init(mydev_init);
module_exit(mydev_exit);
MODULE_LICENSE("GPL");
```

```bash
# モジュールをロードした後、/proc/devicesからmajor番号を読む:
grep mychardev /proc/devices   # → "240 mychardev"（動的なmajor）
mknod /dev/mychardev c 240 0
# あるいはドライバ内でclass_create/device_createを使い、udevに自動作成させる
echo "hello" > /dev/mychardev
cat /dev/mychardev  # → hello
```

## misc_register: 簡単な方法 {#misc_register-the-easy-path}

複数のminorを必要としない単純なデバイスでは、`misc_register`がmajor/minorの割り当てとクラスの作成を自動的に行う。

```c
#include <linux/miscdevice.h>

static struct miscdevice mymisc = {
    .minor = MISC_DYNAMIC_MINOR,   /* カーネルがminorを割り当てる */
    .name  = "mymisc",             /* /dev/mymisc */
    .fops  = &mydev_fops,
};

static int __init mymisc_init(void)
{
    return misc_register(&mymisc);
    /* /dev/mymiscはudev経由で自動的に作成される */
}

static void __exit mymisc_exit(void)
{
    misc_deregister(&mymisc);
}
```

すべてのmiscデバイスはmajor番号10を共有する。これは少なくともLinux 1.3以降、カーネルの公式デバイスリストで予約されている値である。miscデバイスは`/sys/class/misc/`の下に現れる。

## ioctl: 制御操作 {#ioctl-control-operations}

ioctlを使うと、ユーザー空間からドライバに制御コマンドを送ることができる。

```c
/* ドライバ内: */
static long mydev_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    struct mydev_query query;

    switch (cmd) {
    case MYDEV_IOCTL_RESET:
        reset_hardware();
        return 0;

    case MYDEV_IOCTL_QUERY:
        /* ユーザー空間との間で構造体をコピーする */
        if (copy_from_user(&query, (void __user *)arg, sizeof(query)))
            return -EFAULT;
        query.result = do_query(query.param);
        if (copy_to_user((void __user *)arg, &query, sizeof(query)))
            return -EFAULT;
        return 0;

    default:
        return -ENOTTY;  /* 未知のioctl */
    }
}
```

ioctl番号は、衝突を避けるために構造化された形式を使う。

```c
/* include/uapi/linux/mydev.h */
#include <linux/ioctl.h>

/* _IO(type, nr)          — 引数なし */
/* _IOR(type, nr, dtype)  — カーネルから読み出す */
/* _IOW(type, nr, dtype)  — カーネルへ書き込む */
/* _IOWR(type, nr, dtype) — 読み出し+書き込み */

#define MYDEV_MAGIC 'M'
#define MYDEV_IOCTL_RESET  _IO(MYDEV_MAGIC, 0)
#define MYDEV_IOCTL_QUERY  _IOWR(MYDEV_MAGIC, 1, struct mydev_query)
```

```c
/* ユーザー空間: */
int fd = open("/dev/mydev", O_RDWR);
ioctl(fd, MYDEV_IOCTL_RESET);
ioctl(fd, MYDEV_IOCTL_QUERY, &query);
```

## mmap: デバイスメモリをユーザー空間にマップする {#mmap-mapping-device-memory-to-userspace}

ドライバは、デバイスメモリやカーネルバッファをユーザー空間から直接マップできるようにすることができる。

```c
static int mydev_mmap(struct file *filp, struct vm_area_struct *vma)
{
    struct mychardev *dev = filp->private_data;
    unsigned long size = vma->vm_end - vma->vm_start;

    if (size > BUF_SIZE)
        return -EINVAL;

    /* カーネルバッファをユーザー空間にマップする */
    return remap_pfn_range(vma,
        vma->vm_start,
        virt_to_phys(dev->buf) >> PAGE_SHIFT,
        size,
        vma->vm_page_prot);
}

/* デバイスのI/Oメモリ（例: PCI BAR）の場合: */
static int mydev_mmap_io(struct file *filp, struct vm_area_struct *vma)
{
    unsigned long phys = dev->bar_phys + (vma->vm_pgoff << PAGE_SHIFT);
    unsigned long size = vma->vm_end - vma->vm_start;

    /* デバイスレジスタ用にキャッシュ不可としてマークする */
    vma->vm_page_prot = pgprot_noncached(vma->vm_page_prot);

    return remap_pfn_range(vma, vma->vm_start, phys >> PAGE_SHIFT,
                           size, vma->vm_page_prot);
}
```

```c
/* ユーザー空間: */
void *mapped = mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
/* デバイスメモリに直接アクセスする: */
volatile uint32_t *reg = (volatile uint32_t *)mapped;
uint32_t val = *reg;
```

## poll/selectのサポート {#pollselect-support}

```c
static __poll_t mydev_poll(struct file *filp, poll_table *wait)
{
    struct mychardev *dev = filp->private_data;
    __poll_t mask = 0;

    poll_wait(filp, &dev->read_queue, wait);

    if (dev->data_available)
        mask |= EPOLLIN | EPOLLRDNORM;
    if (dev->write_space)
        mask |= EPOLLOUT | EPOLLWRNORM;

    return mask;
}
```

## struct deviceとの統合 {#struct-device-integration}

適切な`/sys/class/`エントリを得て、`/dev/`を自動作成するには:

```c
static struct class *mydev_class;

static int __init mydev_init(void)
{
    /* ... alloc_chrdev_region, cdev_init, cdev_add ... */

    /* /sys/class/mydev/を作成する */
    mydev_class = class_create(THIS_MODULE, "mydev");
    if (IS_ERR(mydev_class))
        goto err_class;

    /* /sys/class/mydev/mydev0/を作成し、udevをトリガーする → /dev/mydev0 */
    device_create(mydev_class, NULL, devt, NULL, "mydev%d", 0);

    return 0;

err_class:
    /* クリーンアップ */
}
```

## 参考文献 {#further-reading}

- [Linuxデバイスモデル](device-model.md) — struct device、class、kobject
- [プラットフォームドライバ](platform-driver.md) — devm_によるリソース管理
- [VFS: ファイル操作](../vfs/file-ops.md) — open/read/writeがどのようにドライバに届くか
- `Documentation/driver-api/miscellaneous_devices.rst`
