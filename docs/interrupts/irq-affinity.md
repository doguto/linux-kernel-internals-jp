# IRQアフィニティとCPU分離 {#irq-affinity-and-cpu-isolation}

> 性能とレイテンシのために、どのCPUが割り込みを処理するかを制御する

## なぜIRQアフィニティが重要か {#why-irq-affinity-matters}

デフォルトでは、カーネル（`irqbalance`デーモン）が割り込みを自動的にCPU間に分散する。レイテンシに敏感なワークロードや高スループットのワークロードでは、アフィニティを手動で制御することで次のことが可能になる。

- 特定のネットワークキューにCPUを専有させる（CPUごとに1キュー）
- リアルタイムタスクを割り込みストームから隔離する
- IRQ処理を、メモリを所有するNUMAノードと同じ場所に配置する

## IRQの割り当ての確認 {#viewing-irq-assignments}

```bash
# すべてのIRQをCPUアフィニティと回数付きで一覧表示する
cat /proc/interrupts
# CPU0     CPU1     CPU2     CPU3
#   0:   ...   IO-APIC   2-edge      timer
#  16:   ...   PCI-MSI   xhci_hcd
# eth0-q0: (MSI-X queue 0)
# eth0-q1: (MSI-X queue 1)

# 特定のIRQのアフィニティを表示する
cat /proc/irq/42/smp_affinity      # ビットマスク（16進）
cat /proc/irq/42/smp_affinity_list # 人間が読める形式（CPUリスト）

# 例: IRQ 42がCPU 0とCPU 1で動作する
# /proc/irq/42/smp_affinity:      00000003  (CPUs 0,1)
# /proc/irq/42/smp_affinity_list: 0-1
```

## IRQアフィニティの設定 {#setting-irq-affinity}

```bash
# IRQ 42をCPU 3のみに固定する
echo 8 > /proc/irq/42/smp_affinity       # 0x8 = ビット3 = CPU 3
# または:
echo 3 > /proc/irq/42/smp_affinity_list  # CPU 3

# 複数のCPU（0と3）に固定する:
echo 9 > /proc/irq/42/smp_affinity       # 0x9 = ビット0,3
echo 0,3 > /proc/irq/42/smp_affinity_list

# すべてのCPU:
echo ff > /proc/irq/42/smp_affinity      # 0xff = CPU 0-7
```

### 高スループットNIC: CPUごとに1キュー {#high-throughput-nic-one-queue-per-cpu}

```bash
# NICの各キューのIRQ番号を調べる
grep 'eth0' /proc/interrupts
# 100:  0  0  0  0  PCI-MSI  eth0-q0
# 101:  0  0  0  0  PCI-MSI  eth0-q1
# 102:  0  0  0  0  PCI-MSI  eth0-q2
# 103:  0  0  0  0  PCI-MSI  eth0-q3

# 各キューを1つのCPUに結びつける
echo 1  > /proc/irq/100/smp_affinity  # q0 → CPU 0
echo 2  > /proc/irq/101/smp_affinity  # q1 → CPU 1
echo 4  > /proc/irq/102/smp_affinity  # q2 → CPU 2
echo 8  > /proc/irq/103/smp_affinity  # q3 → CPU 3

# TX用にXPS（Transmit Packet Steering）も設定する:
echo 1 > /sys/class/net/eth0/queues/tx-0/xps_cpus  # tx-0 → CPU 0
echo 2 > /sys/class/net/eth0/queues/tx-1/xps_cpus  # tx-1 → CPU 1
```

## irqbalance: 自動バランシング {#irqbalance-automatic-balancing}

`irqbalance`は、IRQをCPU間で定期的に再配分するデーモンである。

```bash
# irqbalanceの開始/停止
systemctl start irqbalance
systemctl stop irqbalance  # 手動で制御するために停止する

# irqbalanceのポリシーヒント
# IRQを「performance」としてマークする（移動させない）:
IRQBALANCE_BANNED_IRQS="42 43"  # /etc/sysconfig/irqbalance内で

# 1回だけ: バランシングして終了する
irqbalance --oneshot

# irqbalanceの判断を観察する
irqbalance --debug --foreground 2>&1 | head -50
```

レイテンシに敏感なワークロードでは、irqbalanceを停止してアフィニティを手動で設定する。

## カーネルAPI: irq_set_affinity_hint {#kernel-api-irqsetaffinity_hint}

ドライバは、自分のIRQに対して望ましいCPUを提案できる。

```c
#include <linux/interrupt.h>
#include <linux/cpumask.h>

/* このIRQにCPU 3を提案する（irqbalanceはこのヒントを尊重する） */
irq_set_affinity_hint(irq, cpumask_of(3));

/* CPUの範囲を提案する */
cpumask_t mask;
cpumask_clear(&mask);
cpumask_set_cpu(0, &mask);
cpumask_set_cpu(1, &mask);
irq_set_affinity_hint(irq, &mask);

/* ヒントをクリアする */
irq_set_affinity_hint(irq, NULL);

/* アフィニティを強制する（irqbalanceを迂回する） */
irq_set_affinity(irq, &mask);
```

mlx5のようなNICドライバは、キューのセットアップ時に`irq_set_affinity_hint`を使ってNUMAローカルなCPUを提案する。

```c
/* net/ethernet/mellanox/mlx5/core/eq.c */
static int mlx5_irq_set_affinity_hint(struct mlx5_irq *irq, int i)
{
    struct cpumask *mask = mlx5_irq_get_affinity_mask(irq->pool, i);
    return irq_set_affinity_hint(irq->map.irq, mask);
}
```

## リアルタイムワークロードのためのCPU分離 {#cpu-isolation-for-real-time-workloads}

割り込まれてはならないリアルタイムタスクのために、CPUをカーネルのスケジューラと割り込みバランサから分離する。

### isolcpus: CPUをスケジューラから外す {#isolcpus-remove-cpus-from-the-scheduler}

```bash
# ブートパラメータ: CPU 2,3をスケジューラの負荷分散から外す
GRUB_CMDLINE_LINUX="isolcpus=2,3"

# ブート後: CPU 2,3ではカーネルスレッドや通常のプロセスが動作しない
# 明示的に配置されたタスクだけがそこで動作する（taskset/cgroup cpuset）

# 分離されたCPUにタスクを配置する:
taskset -c 2 ./realtime_task

# あるいはcgroup cpuset経由で:
echo 2 > /sys/fs/cgroup/realtime/cpuset.cpus
echo 0 > /sys/fs/cgroup/realtime/cpuset.mems
```

### nohz_full: tickレス動作 {#nohz_full-tickless-operation}

NO_HZ（tickレスアイドル）機能はIngo MolnárとThomas GleixnerによってLinux 2.6.21で導入された[(LWN)](https://lwn.net/Articles/223185/)。完全なdynticksのための`nohz_full`は後のLinux 3.10で登場した。

```bash
# CPUをスケジューリングtickから外す（タイマー割り込みをほぼ0に減らす）
GRUB_CMDLINE_LINUX="isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3"

# nohz_full: タスクが1つだけ動作しているときに周期的なスケジューリングtickを無効にする
# rcu_nocbs: これらのCPUのRCUコールバックを「RCU kthread」にオフロードする

# 確認: ブート後、分離されたCPUの割り込みはほぼゼロになっているはず
watch -n1 'cat /proc/interrupts | head -5'
```

### irq_default_affinity: 分離されたCPUを除外する {#irqdefaultaffinity-exclude-isolated-cpus}

```bash
# システム全体のデフォルトアフィニティをCPU 2,3を除外するように設定する
echo 3 > /proc/irq/default_smp_affinity  # 0x3 = CPU 0,1のみ
# 新しいIRQはデフォルトでCPU 0,1に配置される
```

### 完全な分離の例（レイテンシが重要なCPU） {#full-isolation-example-latency-critical-cpu}

```bash
#!/bin/bash
# リアルタイムタスクのためにCPU 3を分離する

CPU=3
MASK=$((1 << CPU))
HEX_MASK=$(printf '%x' $MASK)  # CPU 3の場合は"8"

# 1. 移動可能なすべてのIRQをCPU 3から移す
for irq in /proc/irq/*/; do
    irq_num=$(basename $irq)
    [ "$irq_num" = "0" ] && continue  # タイマーはスキップ
    current=$(cat /proc/irq/$irq_num/smp_affinity 2>/dev/null)
    # 現在のマスクがCPU 3を含んでいれば取り除く
    new_mask=$(( 0x$current & ~MASK ))
    [ $new_mask -eq 0 ] && new_mask=1  # 空のマスクにはできない
    echo $new_mask > /proc/irq/$irq_num/smp_affinity 2>/dev/null
done

# 2. irqbalanceがIRQを戻さないように停止する
systemctl stop irqbalance

# 3. カーネルスレッドをCPU 3から移す（任意、tunaが必要）
tuna --cpus=3 --isolate

# 4. リアルタイムタスクを固定する
taskset -c $CPU ./realtime_task &
```

## 割り込み分散の監視 {#monitoring-interrupt-distribution}

```bash
# CPUごとの割り込み回数をリアルタイムで表示する
watch -n0.5 'cat /proc/interrupts'

# IRQごとの割り込みレートを表示する（sar）
sar -I ALL 1 5

# CPUごとの割り込み総数:
cat /proc/stat | grep intr

# IRQアフィニティの要約スクリプト:
for irq in /proc/irq/*/; do
    name=$(cat $irq/irq_name 2>/dev/null || basename $irq)
    aff=$(cat $irq/smp_affinity_list 2>/dev/null)
    printf "IRQ %-4s affinity=%-10s name=%s\n" $(basename $irq) "$aff" "$name"
done
```

## Receive Packet Steering（RPS/RFS） {#receive-packet-steering-rpsrfs}

ハードウェアキューの数がCPU数より少ないNICでは、RPSがソフトウェアでパケットを特定のCPUに振り分ける。

```bash
# RPSを有効にする: 受信パケットをすべてのCPUに振り分ける
echo ff > /sys/class/net/eth0/queues/rx-0/rps_cpus  # 8つのCPUすべて

# RFS: アプリケーションを最後に実行したCPU（キャッシュがホット）に振り分ける
echo 4096 > /sys/class/net/eth0/queues/rx-0/rps_flow_cnt
echo 4096 > /proc/sys/net/core/rps_sock_flow_entries
```

RPSはsoftirqとして実装されており、IPI（プロセッサ間割り込み）を介してパケット処理を対象のCPUに移す。

## 参考文献 {#further-reading}

- [IRQディスクリプタとirq_chip](irq-desc.md) — struct irq_descの内部
- [request_irqとfree_irq](request-irq.md) — IRQハンドラの登録
- [スレッド化IRQ](threaded-irq.md) — レイテンシのためのスレッド化IRQハンドラ
- [PCIドライバ](../drivers/pci-driver.md) — NICドライバにおけるMSI-Xのアフィニティ
- [CPUアフィニティ](../sched/cpu-affinity.md) — プロセスのCPU固定
- [スケジューリングドメイン](../sched/sched-domains.md) — スケジューラのトポロジ
- `kernel/irq/manage.c` — irq_set_affinityの実装
