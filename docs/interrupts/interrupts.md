# 割り込み処理の概要 {#interrupt-handling-overview}

> ハードウェアの信号からカーネルのハンドラまで: その全経路

## ハードウェアが割り込みを送ると何が起きるか {#what-happens-when-hardware-sends-an-interrupt}

x86での割り込みの流れは次のとおりである。

```
Hardware device asserts IRQ line
    ↓
Interrupt Controller (APIC/PIC)
  Identifies interrupt source, routes to CPU
    ↓
CPU finishes current instruction
  Saves: RIP, RFLAGS, RSP, CS, SS to kernel stack
  Clears IF (disables hardware interrupts)
  Looks up handler in IDT (Interrupt Descriptor Table)
    ↓
common_interrupt() (arch/x86/kernel/irq.c)
  irq_enter_rcu()    ← marks hardirq context
  handle_irq()       ← calls the high-level handler
  irq_exit_rcu()     ← exits hardirq context, runs pending softirqs
    ↓
irq_exit_rcu() checks if softirqs are pending
  If yes: invoke_softirq() → __do_softirq()
```

## IDTと割り込みベクタ {#the-idt-and-interrupt-vectors}

x86の割り込み記述子テーブル（IDT）は256エントリを持ち、それぞれがハンドラを指している。

```
Vectors 0-31:   CPU exceptions (divide by zero, page fault, etc.)
Vectors 32-127: Hardware IRQs (mapped by APIC)
Vector 128:     System call (int 0x80, legacy)
Vectors 129-255: Other IPIs, local APIC, MSI
```

各IDTエントリは、特権レベル、セグメント、ハンドラのアドレスを指定する。割り込みが発火すると、CPUはIDTを索引してハンドラを見つける。

## 割り込みの入口: irq_enterとirq_exit {#interrupt-entry-irqenter-and-irqexit}

カーネルは、自分が割り込みコンテキストにいるかどうかを`preempt_count`内のper-CPUカウンタで追跡する。

```c
/* include/linux/preempt.h */
#define SOFTIRQ_SHIFT   (PREEMPT_SHIFT + PREEMPT_BITS)
#define HARDIRQ_SHIFT   (SOFTIRQ_SHIFT + SOFTIRQ_BITS)

/* preempt_countのレイアウト:
   ビット0-7:  プリエンプションカウント
   ビット8-15: softirqカウント
   ビット16-19: hardirqカウント
   ビット20以降: NMIカウント
*/

#define in_hardirq()  (hardirq_count())  /* hardirqハンドラ内 */
#define in_softirq()  (softirq_count())  /* softirqハンドラ内 */
#define in_interrupt() (irq_count())     /* いずれかの割り込みコンテキスト内 */
```

`irq_enter()`はhardirqカウントをインクリメントする。`irq_exit()`はそれをデクリメントし、プロセスコンテキストに戻る場合には保留中のsoftirqを実行する。

```c
/* kernel/softirq.c（簡略化） */
void irq_exit_rcu(void)
{
    /* hardirqのネストカウントをデクリメントする */
    preempt_count_sub(HARDIRQ_OFFSET);

    /* プロセスコンテキストに戻っていればsoftirqを実行する */
    if (!in_interrupt() && local_softirq_pending())
        invoke_softirq();

    tick_irq_exit();
    /* 注意: rcu_irq_exit()とtrace_hardirq_exit()はここでは呼ばれない。
     * _rcu接尾辞は、呼び出し元がRCUのアイドル状態遷移を管理することを意味する */
}
```

## irq_descの検索 {#the-irq_desc-lookup}

カーネルは割り込みベクタから`irq_desc`（割り込みディスクリプタ）を見つける。

```c
/* arch/x86/kernel/irq.c（簡略化） */
__visible void __irq_entry common_interrupt(struct pt_regs *regs, u64 vector)
{
    struct irq_desc *desc;

    irq_enter_rcu();

    /* CPUベクタ → Linux IRQ番号 → irq_descに変換する */
    desc = __this_cpu_read(vector_irq[vector]);
    if (likely(!IS_ERR_OR_NULL(desc)))
        handle_irq(desc, regs);

    irq_exit_rcu();
}
```

`handle_irq(desc, regs)`は1つの汎用ハンドラを実行するのではない。この特定の割り込みについて`desc->handle_irq`に格納されている*フローハンドラ*（`handle_level_irq`、`handle_edge_irq`、`handle_fasteoi_irq`、その他いくつか）を呼び出す。この間接化が存在するのは、スケールしなかった設計があったからである。元のLinuxの割り込みコアは、単一の`__do_IRQ()`という「スーパーハンドラ」を使い、レベルトリガ、エッジトリガ、per-CPU、fast-EOIといったあらゆるコントローラの癖を1つの関数の中で処理しようとし、内部では`hw_interrupt_type`のコールバック構造体を介してディスパッチしていた。アーキテクチャをまたいで割り込みコントローラの種類が増えるにつれ、これは実際の重複を生んだ。x86だけでも、低レベルの詳細のほとんどを共有しながらフロー制御だけが異なる`ioapic_level_irq`と`ioapic_edge_irq`の2つの変種が必要だった。アーキテクチャ横断で分析すると、レベル、エッジ、シンプル、fast-EOIという同じ少数のフローの分類がどこにでも繰り返し現れることがわかった。これこそが、各アーキテクチャが独自のものを再発明する代わりに、共有された1組のフローハンドラをすべてのアーキテクチャで使えるようにしたものである（[Documentation/core-api/genericirq.rst](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/core-api/genericirq.rst)）。

Russell Kingは、2.5/2.6開発系列向けにARMの割り込み処理を構築する中で、実際の分類（レベル、エッジ、シンプル、そして（後に）fast-EOI）をすでに特定していた。Thomas Gleixnerは2005〜2006年に、その分割を全アーキテクチャにまたがる現在の`irq_desc`/フローハンドラ/`irq_chip`のアーキテクチャへと一般化した（`kernel/irq/handle.c`自身のヘッダは、この一般化について"Copyright (C) 2005-2006, Thomas Gleixner, Russell King"とクレジットしており、ファイルのgenirq以前の元の歴史については古い"Copyright (C) 1992, 1998-2006 Linus Torvalds, Ingo Molnar"の行が並んでいる。結果として生まれた構造体の詳細は[IRQディスクリプタとirq_chip](irq-desc.md)を参照）。「この割り込みのライフサイクルがどう流れるか」（フローハンドラ）と「この特定のハードウェアとどう通信するか」（`irq_chip`の操作）を分離したことの見返りこそが、上の`common_interrupt()`をこれほど短くしている。この関数はIOAPICやGIC、MSIについて何も知る必要がない。その知識はすべて`desc->handle_irq`と`desc->irq_data.chip`の背後にあり、これによって1つの`common_interrupt()`が、そのアーキテクチャがサポートするあらゆる種類の割り込みコントローラに対応できるのである。

## 割り込みコンテキストの制約 {#interrupt-context-restrictions}

hardirqコンテキストで動作するコードは、厳格なルールに従わなければならない。

```c
/* してはいけない: 割り込みコンテキストでスリープする */
irqreturn_t my_handler(int irq, void *dev)
{
    msleep(1);           /* バグ: スリープしうる */
    kmalloc(sz, GFP_KERNEL);  /* バグ: スリープしうる */
    mutex_lock(&m);      /* バグ: スリープしうる */
    return IRQ_HANDLED;
}

/* すべきこと: スリープしない代替手段を使う */
irqreturn_t my_handler(int irq, void *dev)
{
    kmalloc(sz, GFP_ATOMIC);  /* 決してスリープしない */
    spin_lock(&s);             /* スリープしない */
    /* ... */
    spin_unlock(&s);
    return IRQ_HANDLED;
}
```

hardirqハンドラの実行中は、現在のCPUでハードウェア割り込みが無効化されている（CPUが入口で`IF`をクリアしている）。他のCPU上の割り込みは影響を受けない。

## /proc/interrupts {#procinterrupts}

```bash
cat /proc/interrupts
#            CPU0       CPU1       CPU2       CPU3
#   0:         21          0          0          0  IR-IO-APIC    2-edge      timer
#   1:          2          0          0          1  IR-IO-APIC    1-edge      i8042
#  16:          0          0          0          0  IR-IO-APIC   16-level     uhci_hcd:usb3
#  24:          0          0          0          0  IR-PCI-MSI 524288-edge      xhci_hcd
# LOC:     198432     197654     196543     198123   Local timer interrupts
# SPU:          0          0          0          0   Spurious interrupts
# PMI:          0          0          0          0   Performance monitoring interrupts
# RES:       1234       1567       1234       1345   Rescheduling interrupts
# CAL:        234        234        234        234   Function call interrupts
```

- 1列目: IRQ番号（特殊な割り込みの場合は名前）
- CPU列: ブート以降のCPUごとの割り込み回数
- コントローラ: 割り込みコントローラとルーティング
- 名前: `request_irq()`で指定されたドライバ名

```bash
# 割り込みレートを観察する
watch -d -n 1 'cat /proc/interrupts'

# 1秒あたりのIRQ
awk 'NR>1 {for(i=2;i<=NF-2;i++) if($i+0>0) {print $0; next}}' /proc/interrupts
```

## IRQアフィニティ {#irq-affinity}

各IRQは特定のCPUに振り向けることができる。

```bash
# IRQアフィニティを表示する（ビットマスク）
cat /proc/irq/24/smp_affinity      # 16進ビットマスク
cat /proc/irq/24/smp_affinity_list # 人間が読める形式のCPUリスト

# IRQ 24をCPU 0-3に固定する
echo "f" > /proc/irq/24/smp_affinity   # 16進のf = CPU 0-3
echo "0-3" > /proc/irq/24/smp_affinity_list

# irqbalanceデーモンはIRQアフィニティを自動的にバランスさせる
systemctl status irqbalance
```

## 参考文献 {#further-reading}

- [IRQディスクリプタとirq_chip](irq-desc.md) — irq_descのデータ構造
- [request_irqとfree_irq](request-irq.md) — ハンドラの登録
- [Softirq](softirq.md) — 保留中のときにirq_exitが実行するもの
- `Documentation/core-api/genericirq.rst` — 汎用IRQサブシステムのドキュメント
