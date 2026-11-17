# request_irqとfree_irq {#request_irq-and-free_irq}

> 割り込みハンドラをカーネルに登録する

## 基本的なAPI {#the-basic-api}

```c
#include <linux/interrupt.h>

/* 割り込みハンドラを登録する */
int request_irq(unsigned int irq,
                irq_handler_t handler,
                unsigned long flags,
                const char *name,
                void *dev_id);

/* 成功時は0、失敗時は負のerrnoを返す */

/* 登録解除: dev_idはrequest_irqに渡したものと一致しなければならない */
void free_irq(unsigned int irq, void *dev_id);

/* 管理版: デバイスが取り外されると自動的に解放される */
int devm_request_irq(struct device *dev, unsigned int irq,
                     irq_handler_t handler, unsigned long flags,
                     const char *name, void *dev_id);
```

## なぜその下のすべてが変わったのに、このAPIはほとんど変わっていないのか {#why-this-api-has-barely-changed-while-everything-under-it-has}

`request_irq()`の形は、2.6系列の初期からほとんど手付かずのままである。その下にあるハードウェアは、IOAPIC、MSI、MSI-X、per-CPUベクタ、そして（他のアーキテクチャでは）ARMのGICのようなまったく異なるコントローラファミリーへと変遷してきたにもかかわらず、である。この安定性は偶然でも、APIが琥珀に閉じ込められたように凍結された結果でもない。これは[割り込み処理の概要](interrupts.md#the-irq_desc-lookup)のページで扱った`irq_desc`/フローハンドラ/`irq_chip`の分離が直接もたらした見返りである。ドライバのハンドラはLinuxのIRQ番号とだけやりとりし、コントローラ固有のものはすべてドライバが決して触れない`irq_chip`の操作の背後にあるため、既存のどのドライバの`request_irq()`呼び出しも変更することなく、まったく新しい割り込みコントローラをカーネルに組み込めるのである。

ハンドラのシグネチャそのものに対する唯一の本当の変更は、その第3引数である。Linux 2.6.18までは、ハンドラは`irqreturn_t (*)(int, void *, struct pt_regs *)`であり、割り込み発生時点の生のレジスタ状態が「念のため」すべてのハンドラに渡されていた。（`irq_handler_t`というtypedef自体はこの直前に登場した。同じ2006年10月のシリーズの、すぐ前にある別のコミット"IRQ: Typedef the IRQ handler function type"が、その時点でのハンドラのシグネチャが何であれ、それに名前を付けたのである。）その後、David Howellsの[*"IRQ: Maintain regs pointer globally rather than passing to IRQ handlers"*](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7d12e780e003f93433d49ce78cfedf4b4c52adc5)（2006年10月）が2.6.19で`pt_regs`引数を削除し（タグ間の差分で確認済み: v2.6.18には存在し、v2.6.19ではなくなっている）、今日の`irqreturn_t (*)(int, void *)`が残った。実際に`regs`を使っているものはほとんどなかった（コミットメッセージ自身が本当の利用者を挙げている。プロファイリングのための`profile_tick()`と、ユーザー対カーネルのCPU時間計上のための`update_process_times()`である。これは`/proc/interrupts`ではなく`/proc/stat`やプロセスごとの`/proc/<pid>/stat`に現れる種類のものであり、どちらも代わりに`get_irq_regs()`を通じて現在のレジスタ状態を得ることができた）。そして生きたポインタをすべてのハンドラに渡すことは、本来の診断目的以外のことに悪用することを常に誘っているようなものだった。必要としていた少数の呼び出し元に同等に有能な代替手段が用意された時点で、誰も必要としない引数を削除するというのは、正当化するのは容易だが元に戻すのは難しい種類のAPI整理である。おそらくそれが、20年経った今でも、これがこのシグネチャにとって唯一の互換性を壊す変更である理由だろう。

## ハンドラの戻り値 {#handler-return-values}

```c
typedef irqreturn_t (*irq_handler_t)(int irq, void *dev_id);

#define IRQ_NONE      (0)            /* 自分の割り込みではない */
#define IRQ_HANDLED   (1)            /* 割り込みを処理した */
#define IRQ_WAKE_THREAD (2)          /* スレッド化ハンドラを起床させる */
```

- 割り込みが自分のデバイスからでなければ`IRQ_NONE`を返す（共有IRQでは重要）
- 処理したら`IRQ_HANDLED`を返す
- スレッド化ハンドラを起床させるには`IRQ_WAKE_THREAD`を返す（[スレッド化IRQ](threaded-irq.md)を参照）

## IRQFフラグ {#irqf-flags}

```c
/* トリガ種別（通常はファームウェア/デバイスツリーで設定済み） */
IRQF_TRIGGER_RISING   /* エッジ: low → high */
IRQF_TRIGGER_FALLING  /* エッジ: high → low */
IRQF_TRIGGER_HIGH     /* レベル: アクティブhigh */
IRQF_TRIGGER_LOW      /* レベル: アクティブlow */

/* 共有 */
IRQF_SHARED           /* 割り込みラインを他のデバイスと共有する */

/* スレッド化 */
IRQF_ONESHOT          /* スレッドハンドラが完了するまでIRQラインを無効にする */
IRQF_NO_THREAD        /* 要求されてもスレッド化しないよう強制する */

/* 電源管理 */
IRQF_NO_SUSPEND       /* サスペンド中も無効にしない */
IRQF_FORCE_RESUME     /* 無効化されていてもレジューム時に強制的に有効にする */

/* その他 */
IRQF_NOBALANCING      /* IRQバランシングの対象から外す */
IRQF_PERCPU           /* per-CPU割り込み */
```

## 最小限のドライバの例 {#a-minimal-driver-example}

```c
#include <linux/interrupt.h>
#include <linux/module.h>

struct my_device {
    int irq;
    void __iomem *regs;
    /* ... */
};

static irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;
    u32 status;

    /* 割り込みステータスレジスタを読む */
    status = readl(dev->regs + STATUS_REG);

    /* 自分の割り込みかどうかを確認する */
    if (!(status & MY_IRQ_BIT))
        return IRQ_NONE;

    /* 割り込みを受領通知（クリア）する */
    writel(MY_IRQ_BIT, dev->regs + STATUS_CLEAR_REG);

    /* ここでは最小限の処理だけを行い、遅延処理をキューに積む */
    schedule_work(&dev->work);

    return IRQ_HANDLED;
}

static int my_driver_probe(struct platform_device *pdev)
{
    struct my_device *dev;
    int ret;

    dev = devm_kzalloc(&pdev->dev, sizeof(*dev), GFP_KERNEL);

    dev->irq = platform_get_irq(pdev, 0);
    dev->regs = devm_ioremap_resource(&pdev->dev, res);

    INIT_WORK(&dev->work, my_work_handler);

    /* devm_request_irq: デバイスが取り外されると自動的に解放される */
    ret = devm_request_irq(&pdev->dev, dev->irq, my_irq_handler,
                           0, dev_name(&pdev->dev), dev);
    if (ret) {
        dev_err(&pdev->dev, "Failed to request IRQ %d\n", dev->irq);
        return ret;
    }

    return 0;
}
```

## 共有IRQ {#shared-irqs}

レガシーなシステム（例: PCI INTx）では、複数のデバイスが1本のIRQラインを共有する。それぞれが`IRQF_SHARED`付きで登録する。

```c
/* 両方のデバイスが同じIRQに登録する */
request_irq(irq, handler_a, IRQF_SHARED, "device-a", dev_a);
request_irq(irq, handler_b, IRQF_SHARED, "device-b", dev_b);

/* IRQが発火すると、両方のハンドラが順に呼ばれる:
   カーネルはhandler_a(irq, dev_a)を呼び、次にhandler_b(irq, dev_b)を呼ぶ
   各ハンドラは自分のハードウェアのステータスレジスタを確認し、
   自分の割り込みでなければIRQ_NONEを返さなければならない */
```

共有IRQの要件:
1. 共有するすべてのドライバが`IRQF_SHARED`を設定しなければならない
2. `dev_id`は一意でなければならない（通常はデバイス構造体へのポインタ）
3. 各ハンドラは、それが自分のデバイスからのものかを判定できなければならない

## disable_irq / enable_irq {#disable_irq-enable_irq}

```c
/* IRQを無効にし、実行中のハンドラの完了を待つ */
disable_irq(irq);          /* ハンドラが終わるまでスリープする */
disable_irq_nosync(irq);   /* すぐに戻る */

/* 再度有効にする */
enable_irq(irq);

/* 対になる: ネストした呼び出し（depthカウンタ） */
disable_irq(irq);  /* depth=1 */
disable_irq(irq);  /* depth=2 */
enable_irq(irq);   /* depth=1、まだ無効 */
enable_irq(irq);   /* depth=0、ここで有効になる */
```

`disable_irq()`は`irq_desc->depth`をインクリメントする。IRQが物理的に無効化されるのは`depth`が0から1になったとき（最初のdisable呼び出し）であり、0に戻ったとき（最後のenable呼び出し）に再度有効化される。

## synchronize_irq {#synchronize_irq}

`free_irq()`は戻る前に内部で`synchronize_irq()`をすでに呼んでいるため、`free_irq()`が戻った時点で、実行中だったハンドラはすべて完了していることが保証される。

```c
free_irq(dev->irq, dev);
/* devを解放しても安全。free_irq()が実行中のハンドラを待った */
```

`devm_request_irq()`はデバイスの取り外し時にこれを自動的に処理するため、手動の`request_irq()` + `free_irq()`よりもこちらを優先すべきである。

## IRQ番号の取得 {#getting-the-irq-number}

```c
/* プラットフォームリソースから */
irq = platform_get_irq(pdev, 0);

/* デバイスツリーから */
irq = of_irq_get(np, 0);

/* PCIから */
irq = pci_irq_vector(pdev, 0);  /* MSIまたはレガシー */

/* GPIOから */
irq = gpiod_to_irq(gpio);
```

## 参考文献 {#further-reading}

- [IRQディスクリプタとirq_chip](irq-desc.md) — 内部で何が登録されているか
- [スレッド化IRQ](threaded-irq.md) — 処理をhardirqコンテキストの外へ移す
- [Workqueue](workqueues.md) — 割り込みハンドラからの遅延処理向け
