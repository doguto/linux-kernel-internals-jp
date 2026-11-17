# ブロックcgroup: I/Oの制御 {#block-cgroup-controlling-io}

> カーネルがデバイスの帯域をcgroup間でどう分配するか — ハードリミット、比例ウェイト、あるいはレイテンシ目標によって

複数のワークロードが1つのディスクを共有すると、デフォルトでは早い者勝ちになる。重い書き込みを行うバッチジョブが、同じデバイス上の対話的なサービスを飢餓状態に追い込むこともある。ブロックI/Oコントローラ（`blkcg`）は、cgroup v2が秩序をもたらす仕組みである。そして、「公平性」の意味は人によって異なるため、本質的に異なる3つのポリシーを提供している。

## 3つのつまみ、3つのポリシー {#three-knobs-three-policies}

cgroup v2の`io`コントローラはいくつかのインターフェースファイルを公開しており、それぞれが異なるカーネル内ポリシーに支えられている。

### `io.max` — ハードリミット（[blk-throttle](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/blk-throttle.c)） {#iomax-hard-limits-blk-throttle}

デバイスごとの絶対的な上限: `rbps`、`wbps`、`riops`、`wiops`。上限を超えたcgroupは、予算内に戻るまでI/Oが遅延させられる。

```
# デバイス259:0に10 MB/sの書き込み上限
echo "259:0 wbps=10485760" > io.max
```

単純で予測可能だが、**容量を無駄にする**。上限を課されたcgroupは、デバイスがほかに何もしていないときでさえ、遊んでいる帯域を借りることができない。厳格なマルチテナント分離や課金には向くが、利用率の面では劣る。

### `io.weight` — 比例配分（[blk-iocost](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/blk-iocost.c) / BFQ） {#ioweight-proportional-sharing-blk-iocost-bfq}

1から10000までのウェイト（デフォルトは100）。帯域は、実際に競合しているcgroupの間でウェイトに*比例して*分配される — ウェイト200のcgroupは、ウェイト100のcgroupのおよそ2倍の取り分を得る — が、遊んでいる容量はそれでも使われる。実装は2つある。

- **BFQ**は[I/Oスケジューラ](io-schedulers.md)であり、比例配分アルゴリズムを通じて`io.weight`を直接尊重する。正確だが、リクエストごとの管理処理が最速のデバイスにはコストが高すぎる。
- **blk-iocost**（[`7caa47151ab2`](https://git.kernel.org/linus/7caa47151ab2)、"blkcg: implement blk-iocost"で導入）は、`none`を含む*任意の*スケジューラと組み合わせて動作する。スケジューリングを行う代わりに、デバイスの**コストモデル**を構築し — 各I/Oがデバイス容量をどれだけ消費するかを見積もり — 各cgroupを公平な取り分にペース配分する。これにより、BFQのオーバーヘッドが支配的になってしまう高速なNVMe上でも、比例制御が実用的になる。

### `io.latency` — レイテンシ保護（[blk-iolatency](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/blk-iolatency.c)） {#iolatency-latency-protection-blk-iolatency}

cgroupに目標レイテンシを設定する。*観測された*レイテンシが目標を上回ると、コントローラは保護対象が回復するまで、優先度の低い**他の**cgroupをスロットリングする。これは「帯域を分配する」ものではなく、「他のすべてを犠牲にしてでもこの1つのワークロードを速く保つ」ものであり、共有デバイス上のレイテンシに敏感なサービスがまさに望むものである。

## `io.stat` {#iostat}

cgroupごとのカウンタ — デバイスごとの読み書きのバイト数とI/O数（iocostが有効な場合はコスト計上も加わる）。ポリシーが想定どおりに動いているかを確認するときに見る場所である。

## ポリシーの選び方 {#choosing-a-policy}

| 望むもの | 使うもの | 実装 |
|---|---|---|
| ハードな上限 / テナント分離 | `io.max` | blk-throttle |
| 高速デバイスでの比例的な公平性 | `io.weight` | blk-iocost |
| 1つのワークロードのレイテンシを安定させる | `io.latency` | blk-iolatency |

## 難所: ライトバックを正しいcgroupに課金する {#the-hard-part-charging-writeback-to-the-right-cgroup}

ブロックcgroup制御を本質的に難しくしている微妙な点は**ライトバック**である。ダーティページは後で、カーネルのフラッシャースレッドによってディスクに書き込まれる — ダーティにしたプロセスによってではない。ブロックコントローラがそのI/Oをフラッシャーに課金すると、cgroupの計上は無意味になってしまう。そのため、ブロックコントローラは[メモリコントローラ](../mm/memcg.md)と連携している。ページをダーティにしたcgroupが記録され、最終的にフラッシュが行われるときに、そのライトバックI/Oはそのcgroupに課金し戻される。このため、ブロックI/O制御とメモリ制御は切り離せない — [cgroup I/O](../cgroups/io-cgroup.md)を参照。

## 参考文献 {#further-reading}

- [Kernel docs: cgroup v2 — IO controller](https://docs.kernel.org/admin-guide/cgroup-v2.html#io)
- [`7caa47151ab2`](https://git.kernel.org/linus/7caa47151ab2) — blk-iocostの導入。`io.weight`の背後にあるデバイスコストモデル
- [`block/blk-throttle.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/blk-throttle.c)、[`blk-iolatency.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/blk-iolatency.c)、[`blk-iocost.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/block/blk-iocost.c) — 3つのポリシーの実装
