# TC（トラフィック制御）とqdisc {#tc-traffic-control-and-qdisc}

> カーネルは送信トラフィックをどうシェーピングし、スケジューリングし、ポリシングするか

## TCが行うこと {#what-tc-does}

トラフィック制御（TC）は、パケットの送信を制御するためのカーネルのフレームワークである。`dev_queue_xmit()`とNICドライバの間に位置し、次のことを扱う:

- **シェーピング**: 送信トラフィックのレートを制限する（TBF、HTB）
- **スケジューリング**: 次にどのパケットを送るかを制御する（PRIO、FQ）
- **ポリシング**: レートを超えたパケットをドロップする（ポリサー）
- **分類**: パケットを処理クラスに対応付ける

TCはデフォルトでは**egress**（送信）しか制御しない。ingressのシェーピングには、リダイレクトの工夫（IFBデバイス）が必要である。

## qdiscの階層 {#the-qdisc-hierarchy}

すべてのネットワークデバイスはルート**qdisc**（queuing discipline、キューイングディシプリン）を持つ。パケットが送信されるとき、ドライバに届く前にqdiscを通過する:

```
dev_queue_xmit(skb)
    → q->enqueue(skb, q)   // qdiscへのエンキュー
    → q->dequeue(q)         // qdiscが次のパケットを選ぶ
    → dev_hard_start_xmit() // ドライバのDMA
```

qdiscは階層化できる:

```
root qdisc (HTB)
├── class 1:10 (rate 100Mbit)
│   └── leaf qdisc (fq_codel)
├── class 1:20 (rate 50Mbit)
│   └── leaf qdisc (pfifo)
└── class 1:30 (rate 10Mbit)  ← default
    └── leaf qdisc (sfq)
```

## 現在のqdiscを確認する {#viewing-current-qdisc}

```bash
# インターフェースのqdiscを表示
tc qdisc show dev eth0

# デフォルト（ほとんどのインターフェース）: pfifo_fast
# → TOSビットに基づく3バンドのFIFO
# ハンドル"0:"がルート

# 階層全体を表示
tc qdisc show dev eth0
tc class show dev eth0
tc filter show dev eth0
```

## よく使われるqdisc {#common-qdiscs}

### pfifo_fast（デフォルト） {#pfifo_fast-default}

パケットのDSCP/TOSフィールドに基づく3つの優先度バンド（0=高、1=中、2=低）。各バンド内は単純なFIFO。レート制限はない。

```bash
# ほとんどのインターフェースでのデフォルト
tc qdisc show dev eth0
# qdisc pfifo_fast 0: root refcnt 2 bands 3 priomap 1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
```

### fq_codel（Linuxデスクトップでの現代的なデフォルト） {#fq_codel-modern-default-on-linux-desktops}

**Fair Queueing + Controlled Delay（CoDel）**。フローごとの公平なスケジューリングとアクティブキュー管理（AQM）を組み合わせ、バッファブロートを軽減する:

```bash
tc qdisc add dev eth0 root fq_codel
tc qdisc show dev eth0
# qdisc fq_codel 8001: root refcnt 2 limit 10240p flows 1024
#   quantum 1514 target 5ms interval 100ms memory_limit 32Mb ecn
```

パラメータ:

- `target`: 許容される最小の滞留キュー遅延（デフォルト5ms）
- `interval`: CoDelの制御ループの間隔（デフォルト100ms）
- `flows`: 公平キューイングのためのハッシュバケット数（デフォルト1024）
- `ecn`: 可能な場合はドロップする代わりにパケットにマークを付ける

### fq（Fair Queue — BBRのデフォルト） {#fq-fair-queue-default-for-bbr}

ペーシング付きのフローごとの公平キューイング。各フローが望む送信レートを指定できる。TCP BBR輻輳制御の正確なペーシングに必要:

```bash
tc qdisc replace dev eth0 root fq

# BBRを有効にした後:
echo bbr > /proc/sys/net/ipv4/tcp_congestion_control
tc qdisc replace dev eth0 root fq  # fqによって正確なレートペーシングが可能になる
```

### TBF（Token Bucket Filter） {#tbf-token-bucket-filter}

バーストを許容しつつ、トラフィックを特定の帯域幅にレート制限する:

```bash
# 10KBのバースト付きで1Mbit/sに制限
tc qdisc add dev eth0 root tbf rate 1mbit burst 10k latency 50ms

# パラメータ:
#   rate:    持続レート
#   burst:   最大バースト（rateを超えるトラフィックは、ここに蓄積されたトークンを消費する）
#   latency: 最大キューイングレイテンシ（キューサイズの計算に使われる）
```

### HTB（Hierarchical Token Bucket） {#htb-hierarchical-token-bucket}

複数クラスのレート制限における標準的な選択肢。保証レートと上限レートを持つクラスの階層をサポートする:

```bash
# ルートのHTB qdisc
tc qdisc add dev eth0 root handle 1: htb default 30

# クラス: 保証レート + 上限
tc class add dev eth0 parent 1:  classid 1:1 htb rate 100mbit
tc class add dev eth0 parent 1:1 classid 1:10 htb rate 80mbit ceil 100mbit
tc class add dev eth0 parent 1:1 classid 1:20 htb rate 15mbit ceil 100mbit
tc class add dev eth0 parent 1:1 classid 1:30 htb rate  5mbit ceil 100mbit

# 各クラスのリーフqdisc
tc qdisc add dev eth0 parent 1:10 handle 10: fq_codel
tc qdisc add dev eth0 parent 1:20 handle 20: fq_codel
tc qdisc add dev eth0 parent 1:30 handle 30: fq_codel
```

クラス1:10の使用量が80Mbit未満のとき、その未使用の帯域幅は1:20と1:30が、それぞれの上限（100Mbit）まで借用できる。

## TCフィルタ: パケットの分類 {#tc-filters-classifying-packets}

フィルタは、パケットがどのクラスに入るかを決定する:

```bash
# 宛先ポートでマッチ → クラス1:10
tc filter add dev eth0 parent 1: protocol ip prio 1 \
    u32 match ip dport 80 0xffff flowid 1:10

# 送信元IPでマッチ
tc filter add dev eth0 parent 1: protocol ip prio 2 \
    u32 match ip src 10.0.0.0/8 flowid 1:20

# fwmark（iptablesのMARKによるもの）でマッチ
tc filter add dev eth0 parent 1: protocol ip prio 3 \
    handle 0x1 fw flowid 1:10

# eBPF分類器（現代的なアプローチ）
tc filter add dev eth0 parent 1: bpf obj tc_prog.o sec tc flowid 1:10
```

## TC eBPF（TC-BPF） {#tc-ebpf-tc-bpf}

eBPFプログラムは、分類器またはアクションとしてTCにアタッチできる。XDPと異なり、TC-BPFは**`sk_buff`の割り当て後**に実行されるため、パケットを完全に書き換えられる:

```c
// TC egressプログラム: VLANタグを付加する
SEC("tc")
int tc_add_vlan(struct __sk_buff *skb)
{
    bpf_skb_vlan_push(skb, htons(ETH_P_8021Q), 100);  // VLAN 100
    return TC_ACT_OK;
}
```

```bash
# TC-BPFプログラムをegressにアタッチ
tc qdisc add dev eth0 clsact
tc filter add dev eth0 egress bpf obj tc_prog.o sec tc direct-action

# ingressにもアタッチできる（受信トラフィックのシェーピング用）
tc filter add dev eth0 ingress bpf obj tc_ingress.o sec tc direct-action
```

TC-BPFの判定結果:

- `TC_ACT_OK`（= 0）: 次の段階へ渡す
- `TC_ACT_SHOT`（= 2）: ドロップする
- `TC_ACT_REDIRECT`（= 7）: 別のインターフェースへリダイレクトする

## IFBによるingressシェーピング {#ingress-shaping-with-ifb}

TCがネイティブにシェーピングできるのはegressのみである。ingressをシェーピングするには:

```bash
# IFB（Intermediate Functional Block）デバイスを作成
modprobe ifb
ip link set dev ifb0 up

# すべてのingressトラフィックをIFBへリダイレクト
tc qdisc add dev eth0 ingress handle ffff:
tc filter add dev eth0 parent ffff: protocol all u32 \
    match u32 0 0 action mirred egress redirect dev ifb0

# ここでifb0のegress（= eth0のingress）をシェーピングする
tc qdisc add dev ifb0 root tbf rate 10mbit burst 20k latency 100ms
```

## netem: ネットワークエミュレーション {#netem-network-emulation}

`netem`は人為的な遅延、ジッタ、パケットロス、データ破損を加える。テストに有用である:

```bash
# 10msのジッタ付きで100msの遅延を加える
tc qdisc add dev eth0 root netem delay 100ms 10ms

# 5%のパケットロスを加える
tc qdisc add dev eth0 root netem loss 5%

# 組み合わせ: 50msの遅延 + 1%のロス + 0.1%の破損
tc qdisc add dev eth0 root netem delay 50ms loss 1% corrupt 0.1%

# 削除
tc qdisc del dev eth0 root
```

## 統計 {#statistics}

```bash
# qdiscの統計を表示
tc -s qdisc show dev eth0
# qdisc fq_codel 8001: root ...
#  Sent 12345 bytes 100 pkt (dropped 0, overlimits 0 requeues 0)
#  backlog 0b 0p requeues 0
#  maxpacket 1514 drop_overlimit 0 new_flow_count 50

# クラスの統計を表示（HTB）
tc -s class show dev eth0

# リアルタイムで監視
watch -n 1 'tc -s qdisc show dev eth0'
```

## 参考文献 {#further-reading}

- [XDP](xdp.md) — スタック前でのパケット処理（ドロップについてはTCより高性能）
- [AF_XDPソケット](af-xdp.md) — 完全なカーネルバイパスなしのユーザー空間パケットI/O
- [パケットの一生（送信）](life-of-packet-tx.md) — 送信パスの中でTCがどこに位置するか
- [CPU帯域制御](../sched/cpu-bandwidth.md) — CPU時間に対する類似の制御
