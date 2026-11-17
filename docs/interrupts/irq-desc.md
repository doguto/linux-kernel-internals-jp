# IRQディスクリプタとirq_chip {#irq-descriptor-and-irq_chip}

> 割り込みハードウェアに対するカーネルの抽象化

## 3層の割り込み抽象化 {#the-three-layer-interrupt-abstraction}

Linuxの割り込みサブシステム、すなわちThomas GleixnerとIngo MolnárがLinux 2.6.16で導入したgenirqフレームワーク[(LWN)](https://lwn.net/Articles/184750/)は、大きく異なる割り込みコントローラ（APIC、GIC、IOAPIC、MSI、GPIOコントローラなど）を抽象化するために3つの構造体を使う。

```
Linux IRQ number
    ↓
struct irq_desc        ← per-IRQ kernel state (handlers, counters, flags)
    │
    ├── struct irq_data      ← chip-level data (controller, chip-specific)
    │       │
    │       └── struct irq_chip   ← controller operations (mask, unmask, ack, eoi)
    │
    └── struct irqaction     ← linked list of registered handlers
```

## struct irq_desc {#struct-irq_desc}

`irq_desc`はLinuxのIRQ番号ごとに1つ存在する。これはハードウェア抽象化とソフトウェアのハンドラを結びつける中心的なオブジェクトである。

```c
/* include/linux/irqdesc.h */
struct irq_desc {
    struct irq_common_data irq_common_data;  /* irq/chip間で共有するデータ */
    struct irq_data        irq_data;         /* チップレベルのデータ */
    struct irqstat __percpu *kstat_irqs;     /* CPUごとの割り込み回数 */
    irq_flow_handler_t     handle_irq;       /* 高レベルハンドラ */
    struct irqaction       *action;          /* ハンドラチェーン */
    unsigned int           depth;            /* ネストした無効化の回数 */
    unsigned int           irq_count;        /* ストール検出用 */
    unsigned int           irqs_unhandled;   /* スプリアス割り込みの回数 */
    raw_spinlock_t         lock;             /* この構造体を保護する */
    struct cpumask         *percpu_enabled;  /* CPUごとの有効状態 */
    const char             *name;
};
```

主なフィールド:

- `handle_irq`: **フローハンドラ**。アクションチェーンをどのように実行するか（エッジトリガ、レベルトリガ、per-CPUなど）を決める
- `action`: `irqaction`構造体の連結リストで、登録されたハンドラごとに1つ
- `depth`: `disable_irq()`でインクリメントされ、`enable_irq()`でデクリメントされる。0のときだけ実際に有効である

## struct irqaction {#struct-irqaction}

`irqaction`は`request_irq()`の呼び出しごとに1つ作られる。複数のドライバが1本のIRQラインを共有でき、それぞれが自分の`irqaction`を持つ。

```c
/* include/linux/interrupt.h */
struct irqaction {
    irq_handler_t    handler;     /* 割り込みハンドラ関数 */
    void             *dev_id;     /* ハンドラに渡され、デバイスを識別する */
    struct irqaction *next;       /* このIRQを共有する次のハンドラ */
    irq_handler_t    thread_fn;   /* スレッド化ハンドラ（IRQF_ONESHOTの場合） */
    struct task_struct *thread;   /* スレッド化ハンドラ用のカーネルスレッド */
    unsigned int     irq;         /* IRQ番号 */
    unsigned int     flags;       /* IRQF_*フラグ */
    const char       *name;       /* /proc/interruptsに表示される */
};
```

共有IRQが発火すると、カーネルは`irqaction`チェーンをたどって各ハンドラを順に呼び出す。各ハンドラは、割り込みを処理した場合は`IRQ_HANDLED`を、自分のデバイスからの割り込みでなかった場合は`IRQ_NONE`を返す。

## struct irq_chip {#struct-irq_chip}

`irq_chip`は、割り込みコントローラのハードウェアを制御するための操作を提供する。各コントローラ（APIC、ARM GICなど）がこのインターフェースを実装する。

```c
/* include/linux/irq.h */
struct irq_chip {
    const char *name;           /* /proc/interruptsに表示される */

    /* 基本的な制御 */
    void (*irq_mask)(struct irq_data *data);    /* コントローラで無効化 */
    void (*irq_unmask)(struct irq_data *data);  /* コントローラで有効化 */
    void (*irq_enable)(struct irq_data *data);  /* 有効化（unmaskと同じ場合もある） */
    void (*irq_disable)(struct irq_data *data); /* 無効化 */
    void (*irq_ack)(struct irq_data *data);     /* 受領通知（エッジ） */
    void (*irq_eoi)(struct irq_data *data);     /* 割り込み終了（レベル） */

    /* アフィニティ */
    int  (*irq_set_affinity)(struct irq_data *data,
                              const struct cpumask *dest, bool force);

    /* トリガ種別 */
    int  (*irq_set_type)(struct irq_data *data, unsigned int flow_type);

    /* ウェイクアップ */
    int  (*irq_set_wake)(struct irq_data *data, unsigned int on);
};
```

ドライバは通常`irq_chip`の操作を直接呼ばない。汎用IRQレイヤーが適切なタイミングでそれらを呼び出す。

## IRQドメイン {#irq-domains}

現代のシステムは階層的な割り込みコントローラ（例: GPIOエクスパンダ → GIC → CPU）を持つ。**IRQドメイン**は、ハードウェアの割り込み番号をLinuxのIRQ番号に対応付ける。

```
Hardware interrupt 47 (on GPIO expander)
    ↓ irq_domain (gpio chip)
Linux IRQ 234
    ↓ irq_domain (GIC)
Hardware interrupt 58 (at GIC)
    ↓
CPU interrupt vector
```

```c
/* IRQドメインの作成（ドライバのコード） */
struct irq_domain *domain = irq_domain_add_linear(
    node,           /* デバイスツリーのノード */
    num_irqs,       /* ハードウェアIRQの数 */
    &my_domain_ops, /* hw → linux IRQの変換 */
    host_data       /* ドライバのプライベートデータ */
);

/* ドメインの操作内: HW IRQをLinux IRQに対応付ける */
static int my_irq_domain_map(struct irq_domain *d,
                              unsigned int virq, irq_hw_number_t hwirq)
{
    irq_set_chip_and_handler(virq, &my_chip, handle_level_irq);
    irq_set_chip_data(virq, d->host_data);
    return 0;
}
```

## フローハンドラ {#flow-handlers}

フローハンドラ（`irq_desc`内の`handle_irq`）は、割り込みの配送がどのように行われるかを決める。

| ハンドラ | 用途 |
|---------|-------|
| `handle_edge_irq` | エッジトリガ: アクションチェーンを呼ぶ前にackする |
| `handle_level_irq` | レベルトリガ: マスクし、アクションを実行し、アンマスクする |
| `handle_fasteoi_irq` | 現代的なレベル（EOI付き）: ARM/x86 MSIの大半 |
| `handle_percpu_irq` | per-CPU割り込み（ローカルタイマー、IPI） |
| `handle_simple_irq` | mask/ackなしで、アクションを実行するだけ |

エッジとレベルの違いは重要である。

- **エッジ**: 信号が遷移したときに割り込みが1回発火する。取りこぼすと再び発火することはない。
- **レベル**: 信号がアサートされている間、割り込みが発火し続ける。処理中はマスクしなければならない。

## IRQ状態の観察 {#observing-irq-state}

```bash
# コントローラ情報付きの完全なIRQテーブル
cat /proc/interrupts

# IRQごとの情報
ls /proc/irq/24/
# affinity_hint  node  smp_affinity  smp_affinity_list  spread_affinity

# 現在のアフィニティ
cat /proc/irq/24/smp_affinity_list  # 例: "0-3"

# IRQチップ名とハンドラ
cat /sys/kernel/debug/irq/irqs/24  # （debugfsが必要）
```

## 参考文献 {#further-reading}

- [割り込み処理の概要](interrupts.md) — ハードウェア割り込みの流れ
- [request_irqとfree_irq](request-irq.md) — irqactionの登録
- `Documentation/core-api/genericirq.rst` — 汎用IRQサブシステムの設計
