# KVMライブマイグレーション {#kvm-live-migration}

> 実行中のVMを最小限のダウンタイムでホスト間に移す仕組み

## 目標 {#the-goal}

ライブマイグレーションは、VMの実行を続けたまま、実行中のVMをあるホストから別のホストへ移す。VMは移行元が停止する前に移行先で見えるようになり、この引き継ぎはゲストのワークロードからは見えない（あるいはほぼ見えない）。目標は、ゲストが知覚する一時停止を100 ms未満にすることである。

課題は、ゲストのRAMがコピーできる速度より速く変更されていることだ。ライブマイグレーションは、残りのダーティな集合が目標とするダウンタイムの予算内で転送できるほど小さくなるまで、プレコピーのフェーズを反復的に実行することでこれを解決する。

```
Source host                          Destination host
┌────────────────────────────┐       ┌────────────────────────────┐
│  QEMU + KVM (VM running)   │       │  QEMU + KVM (VM paused)    │
│                            │       │                            │
│  Phase 1: Pre-copy         │──RAM──►  Receiving pages           │
│  (VM still running,        │       │                            │
│   dirty pages tracked)     │       │                            │
│                            │       │                            │
│  Phase 2: Stop-and-copy    │──RAM──►  Remaining dirty pages     │
│  (VM paused)               │──CPU──►  vCPU register state       │
│                            │──DEV──►  Device state              │
│                            │       │                            │
│  Phase 3: Handoff          │       │  VM resumes here           │
│  (VM stopped on source)    │       │                            │
└────────────────────────────┘       └────────────────────────────┘
```

## KVMのダーティページ追跡 {#kvm-dirty-page-tracking}

KVMは、前回のチェックポイント以降にどのゲストページが変更されたかを追跡する2つの仕組みを提供する。

### ダーティログのビットマップ {#dirty-log-bitmap}

`KVM_MEM_LOG_DIRTY_PAGES`フラグで、メモリスロットごとにダーティ追跡を有効にする。

```c
/* ダーティログを有効にしたメモリスロットをセットアップする */
struct kvm_userspace_memory_region region = {
    .slot            = 0,
    .flags           = KVM_MEM_LOG_DIRTY_PAGES,  /* ダーティ追跡を有効にする */
    .guest_phys_addr = 0x0,
    .memory_size     = ram_size,
    .userspace_addr  = (uint64_t)ram_ptr,
};
ioctl(vm_fd, KVM_SET_USER_MEMORY_REGION, &region);
```

ダーティログを有効にすると、KVMはそのスロットのすべてのEPT/NPTのリーフエントリを書き込み保護する。各ページへの最初の書き込みでEPT violationのexitが発生し、KVMはそのページをダーティビットマップに記録して、マッピングを再び書き込み可能にする。これにより、エポックごとに各ページへの最初の書き込みにつき1回のexitが追加される。

ダーティビットマップは`KVM_GET_DIRTY_LOG`で取得する。

```c
struct kvm_dirty_log dirty_log = {
    .slot        = 0,
    .dirty_bitmap = bitmap,  /* 呼び出し側が割り当てる。サイズ = num_pages / 8バイト */
};
ioctl(vm_fd, KVM_GET_DIRTY_LOG, &dirty_log);
/* この呼び出しの後、bitmapにはダーティなページごとに1ビットがセットされている。
   KVMはすべてのダーティビットをクリアし、ページを再び書き込み保護もする。 */
```

`KVM_CLEAR_DIRTY_LOG`（カーネル5.0で追加）は、スロット全体ではなく特定の範囲のページのダーティビットをクリアし、反復的なプレコピー中のEPTの書き込み保護のフラッシュ回数を減らす。

```c
struct kvm_clear_dirty_log clear = {
    .slot        = 0,
    .num_pages   = pages_to_clear,
    .first_page  = page_offset,
    .dirty_bitmap = bitmap,
};
ioctl(vm_fd, KVM_CLEAR_DIRTY_LOG, &clear);
```

### ダーティリング（KVM_CAP_DIRTY_LOG_RING、カーネル5.11） {#dirty-ring-kvm_cap_dirty_log_ring-kernel-511}

リングバッファによる方式は、ビットマップのポーリングを、ダーティなGFNを順番に記録するvCPUごとのリングに置き換える。

```c
/* ダーティリングがサポートされているか確認する */
int ring_size = ioctl(kvm_fd, KVM_CHECK_EXTENSION, KVM_CAP_DIRTY_LOG_RING);
/* ring_size: vCPUリングあたりの最大エントリ数（2のべき乗）。未サポートなら0 */

/* VM作成時に有効にする */
struct kvm_enable_cap cap = {
    .cap   = KVM_CAP_DIRTY_LOG_RING,
    .args  = { ring_size * sizeof(struct kvm_dirty_gfn) },
};
ioctl(vm_fd, KVM_ENABLE_CAP, &cap);

/* struct kvm_dirty_gfn — ダーティになったページごとに1エントリ */
struct kvm_dirty_gfn {
    __u32 flags;  /* KVM_DIRTY_GFN_F_DIRTY: ページがダーティになった */
    __u32 slot;
    __u64 offset; /* スロット内のページオフセット */
};
```

ダーティリングは、vCPUのfdからオフセット`KVM_DIRTY_LOG_PAGE_OFFSET * PAGE_SIZE`でmmapされる（オフセット0にある`struct kvm_run`とは別の領域である）。QEMUはvCPUスレッドがエントリを生成するのに合わせてリングを読むので、ビットマップをポーリングしてすべてのページを再び書き込み保護する必要がない。

## 3つのマイグレーションフェーズの詳細 {#the-three-migration-phases-in-detail}

### フェーズ1: 反復的プレコピー {#phase-1-iterative-pre-copy}

移行元のQEMUはダーティログを有効にし、次の処理を繰り返す。

1. まだ送っていない（または前回のラウンド以降に再びダーティになった）すべてのページを移行先にコピーする。
2. `KVM_GET_DIRTY_LOG`でダーティビットマップを取得する。
3. 新たにダーティになったページを送る。

ラウンドごとに送るページは減っていき、ゲストのワーキングセットは収束する。QEMUは次を追跡する。

- **ダーティレート**: ゲストが1秒あたりにダーティにするページ数。
- **帯域幅**: 移行先へ1秒あたりに転送するページ数。

プレコピーは、推定される残りの転送時間が目標ダウンタイムを下回ったときに終了する。目標ダウンタイムはマイグレーションパラメータ`downtime-limit`（デフォルト300 ms）で設定する。これは`max-bandwidth`とは別の設定項目であり、`max-bandwidth`は目標ダウンタイムではなく転送速度に上限を設ける。

**Auto-converge**: ダーティレートが高いままで収束が起こらない場合、`mig_throttle_guest_down()`（`migration/ram.c`内）が`cpu_throttle_set()`を呼び出してvCPUスレッドを周期的に停止させる。`throttle-trigger-threshold`の比率を超えるたびにスロットリングの割合を`cpu-throttle-increment`ずつ引き上げ、`max-cpu-throttle`まで上げていく。これにより、一時的なゲスト性能の低下と引き換えに、マイグレーションが確実に完了する。

### フェーズ2: Stop-and-copy {#phase-2-stop-and-copy}

移行元のVMを一時停止する。QEMUは最後の`KVM_GET_DIRTY_LOG`のパスを行い、残りのダーティページをすべて転送する。その後、CPU状態を転送する。

### フェーズ3: CPU状態の転送 {#phase-3-cpu-state-transfer}

各vCPUのレジスタ状態は、一連のioctlによって移行元で保存され、移行先で復元される。

```c
/* 汎用レジスタ */
struct kvm_regs regs;
ioctl(vcpu_fd, KVM_GET_REGS, &regs);     /* rax、rbx、rip、rsp、rflags、... */
ioctl(vcpu_fd, KVM_SET_REGS, &regs);

/* セグメントレジスタ、制御レジスタ、ディスクリプタテーブル */
struct kvm_sregs sregs;
ioctl(vcpu_fd, KVM_GET_SREGS, &sregs);   /* cr0、cr3、cr4、cs、ss、gdt、... */
ioctl(vcpu_fd, KVM_SET_SREGS, &sregs);

/* モデル固有レジスタ（MSR） */
struct kvm_msrs *msrs;  /* 可変長: ヘッダ + kvm_msr_entryの配列 */
ioctl(vcpu_fd, KVM_GET_MSRS, msrs);      /* EFER、LSTAR、FS_BASE、GS_BASE、... */
ioctl(vcpu_fd, KVM_SET_MSRS, msrs);

/* 拡張CPU状態: AVX、AVX-512レジスタ */
struct kvm_xsave xsave;
ioctl(vcpu_fd, KVM_GET_XSAVE, &xsave);
ioctl(vcpu_fd, KVM_SET_XSAVE, &xsave);

/* FPU状態（レガシーな経路。最近のカーネルではKVM_GET_XSAVEに置き換えられている） */
struct kvm_fpu fpu;
ioctl(vcpu_fd, KVM_GET_FPU, &fpu);
ioctl(vcpu_fd, KVM_SET_FPU, &fpu);

/* ローカルAPICの状態 */
struct kvm_lapic_state lapic;
ioctl(vcpu_fd, KVM_GET_LAPIC, &lapic);
ioctl(vcpu_fd, KVM_SET_LAPIC, &lapic);

/* ネストされた仮想化の状態（ゲストもハイパーバイザである場合） */
/* KVM_GET_NESTED_STATE / KVM_SET_NESTED_STATE — nested-virt.mdを参照 */
```

## デバイスの状態 {#device-state}

### QEMUがエミュレートするデバイス {#qemu-emulated-devices}

QEMUは、VMStateフレームワークを使ってデバイスの状態をシリアライズする。各デバイスは`VMStateDescription`を登録する。

```c
/* hw/net/e1000.c、簡略化したもの — 実際のリストには個別のmac_reg[REG]
 * エントリが約50個（1つの配列マクロではない）といくつかの.subsectionsがある。
 * RXバッファの内容は一切マイグレーションされず、レジスタ/設定のみ: */
static const VMStateDescription vmstate_e1000 = {
    .name    = "e1000",
    .version_id = 2,
    .minimum_version_id = 1,
    .fields  = (const VMStateField[]) {
        VMSTATE_PCI_DEVICE(parent_obj, E1000State),
        VMSTATE_UINT32(mac_reg[CTRL], E1000State),
        VMSTATE_UINT32(mac_reg[STATUS], E1000State),
        /* ... レジスタごとに1つのVMSTATE_UINT32(mac_reg[REG], ...) ... */
        VMSTATE_UINT16_ARRAY(eeprom_data, E1000State, 64),
        VMSTATE_END_OF_LIST()
    },
};
```

`VMSTATE_*`マクロは、フィールドの型、オフセット、サイズをエンコードする。QEMUのマイグレーション層は、登録されたすべての`VMStateDescription`構造体を走査し、各フィールドをシリアライズして、その結果をマイグレーションのトランスポートを通じて移行先にストリームする。

### vhostのカーネルバックエンド {#vhost-kernel-backends}

カーネル側のvhostデバイス（vhost-net、vhost-blk）は、カーネル内に存在するためQEMUが直接シリアライズできない状態を持つ。重要な状態はvirtqueueの位置である。

```c
/* マイグレーション前にvhostのvringの状態を保存する */
struct vhost_vring_state state = { .index = queue_idx };
ioctl(vhost_fd, VHOST_GET_VRING_BASE, &state);
/* state.numには現在のavail/usedのインデックスが入っている */

/* 移行先で復元する */
ioctl(vhost_fd, VHOST_SET_VRING_BASE, &state);
```

（vhostではなく）QEMUがバックエンドとなるvirtioデバイスの場合、virtqueueの状態はQEMU自身の`struct VirtQueue`（Linuxカーネルの`struct virtio_device`ではない。こちらはまったく別のコードベースにあるドライバ側の型である）から、`hw/virtio/virtio.c`の`vmstate_virtqueue`を通じて取得される。これには`last_avail_idx`と`used_idx`が含まれる。

## ポストコピーマイグレーション {#post-copy-migration}

ポストコピーは方向を逆にする。すべてのページが転送される**前に**、VMが移行先で起動する。まだコピーされていないページは、必要に応じて移行元から取得される。

```
Pre-copy:   copy all RAM → stop VM → transfer CPU → resume on dest
Post-copy:  stop VM → transfer CPU → resume on dest (immediately)
                                       → fault on missing pages → fetch from source
```

KVMのポストコピーは**userfaultfd**に依存している。移行先のQEMUは、ゲストのRAMのマッピング（無名メモリ）を`MISSING`モードで`userfaultfd`に登録する。ゲストがまだコピーされていないページにアクセスすると、カーネルはQEMUのスレッドに`UFFD_EVENT_PAGEFAULT`を配送し、そのスレッドが移行元ホストからページを取得してインストールする。

```c
/* 移行先: RAMをuserfaultfdに登録する */
int uffd = syscall(SYS_userfaultfd, O_CLOEXEC | O_NONBLOCK);

struct uffdio_api api = { .api = UFFD_API, .features = 0 }; /* 無名メモリではMISSINGモードに機能フラグは不要 */
ioctl(uffd, UFFDIO_API, &api);

struct uffdio_register reg = {
    .range  = { .start = (uint64_t)ram, .len = ram_size },
    .mode   = UFFDIO_REGISTER_MODE_MISSING,
};
ioctl(uffd, UFFDIO_REGISTER, &reg);

/* QEMUのフォールトハンドラスレッド: uffdを読み、移行元から取得し、ページをコピーする */
struct uffd_msg msg;
read(uffd, &msg, sizeof(msg));
/* msg.arg.pagefault.address: フォールトしたGPA → 移行元から取得 → UFFDIO_COPY */
```

ポストコピーのトレードオフ:

| | プレコピー | ポストコピー |
|--|---------|----------|
| 停止時間 | 長い（ダーティレートに比例） | 最小限（CPU状態のための1往復） |
| ページ取得のレイテンシ | 再開後はなし | あり — コピーされていないページごとにvCPUが停止する |
| 障害への脆弱性 | 中止して移行元で再開できる | VMが移行先で起動した後は、移行元の障害は致命的 |

## マイグレーションのトランスポート {#migration-transports}

QEMUは複数のトランスポートチャネルをサポートする。

| トランスポート | 用途 |
|-----------|----------|
| TCP | デフォルト。単純で、どこでも動作する |
| RDMA（`rdma:host:port`） | 大容量RAM向けの低レイテンシな一括転送。DMAによりCPUを迂回する |
| multifd | 並列のTCPストリーム（`migrate_set_parameter multifd-channels N`）。高帯域のリンクを飽和させる |
| Unixソケット | 同一ホストでのテスト |

**multifd**（multiple file descriptors）は、N本の並列チャネルを開き、ページをラウンドロビンでチャネルに割り当てる。QEMU 2.11で実験的機能（`x-multifd-channels`）として追加され、QEMU 4.0で安定化した（`x-`接頭辞が外れ、`multifd-channels`に改名された）。25 Gbpsのリンクで8チャネルを使えば、ストリームごとのスループットの限界により単一のTCPストリームでは飽和させられない帯域幅を飽和させられる。

## マイグレーションの観測 {#observing-migration}

```bash
# QEMUモニタ: マイグレーションを開始する
(qemu) migrate tcp:192.168.1.2:4444

# マイグレーションの状態を確認する（実際のhmp_info_migrateの出力形式）
(qemu) info migrate
# Status: 		active
# Time (ms): 		total=4321
# Remaining: 		56 MiB
# RAM info:
#   Throughput (Mbps): 	120.50
#   Sizes: 		pagesize=4 KiB, total=4096 MiB
#   Transfers: 		transferred=1234 MiB, remain=56 MiB
#     Channels: 		precopy=1234 MiB, multifd=0 MiB, postcopy=0 MiB
#     Page Types: 	normal=316000, zero=1500
#   Page Rates (pps): 	transfer=45000, dirty=12300
#   Others: 		dirty_syncs=8

# 設定された目標ダウンタイムは別のクエリで確認する:
(qemu) info migrate_parameters
# downtime-limit: 300 ms

# 移行元のカーネル: ダーティログのtracepoint
echo 1 > /sys/kernel/tracing/events/kvm/kvm_dirty_ring_push/enable
echo 1 > /sys/kernel/tracing/events/kvm/kvm_dirty_ring_reset/enable

# マイグレーション中のvCPUごとのページフォールト率を監視する（ダーティ追跡の
# 書き込み保護フォールトが発生するにつれて急増してから減少し、収束するはず）—
# exitの種類ごとの内訳を1つのdebugfsファイルで示すものは存在しないので、
# 生のカウンタを監視し、EPT_VIOLATIONに特化した表示には下のperfコマンドを使う:
watch -n1 'cat /sys/kernel/debug/kvm/*/vcpu0/pf_taken'

# perf: 書き込み保護フォールトによる余分なEPT violationを監視する
perf kvm stat report --event EPT_VIOLATION
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [virt/kvm/kvm_main.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/virt/kvm/kvm_main.c) — `kvm_vm_ioctl_get_dirty_log()` / `kvm_vm_ioctl_clear_dirty_log()`: `KVM_GET_DIRTY_LOG`と`KVM_CLEAR_DIRTY_LOG`の実装
- [virt/kvm/dirty_ring.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/virt/kvm/dirty_ring.c) — `kvm_dirty_ring_push()` / `kvm_dirty_ring_reset()`: `KVM_CAP_DIRTY_LOG_RING`のリングバッファの実装
- [include/uapi/linux/kvm.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/kvm.h) — `struct kvm_dirty_gfn`、`KVM_DIRTY_LOG_PAGE_OFFSET`、およびダーティログ/ダーティリングのioctlの定義
- [mm/userfaultfd.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/userfaultfd.c) — `handle_userfault()`と、QEMUのポストコピーの移行先が欠落ページのフォールトを処理するために使う`UFFDIO_COPY`の経路
- [drivers/vhost/vhost.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/vhost/vhost.c) — マイグレーションをまたいでvringのインデックスを保存・復元するための`VHOST_GET_VRING_BASE` / `VHOST_SET_VRING_BASE` ioctlハンドラ

### QEMUのソース {#qemu-source}

QEMUのマイグレーションはユーザー空間のコードである。本サイトの方針に従い、これらはGitHubのミラーではなくQEMU自身の正規のGitLabリポジトリにリンクしている。

- [migration/ram.c](https://gitlab.com/qemu-project/qemu/-/blob/master/migration/ram.c) — ダーティビットマップに基づくRAMのマイグレーションと、auto-convergeのスロットリングのロジック
- [migration/postcopy-ram.c](https://gitlab.com/qemu-project/qemu/-/blob/master/migration/postcopy-ram.c) — ポストコピーの移行先における`userfaultfd`の登録とフォールト処理
- [migration/multifd.c](https://gitlab.com/qemu-project/qemu/-/blob/master/migration/multifd.c) — multifdの並列マイグレーションチャネルの実装
- [migration/vmstate.c](https://gitlab.com/qemu-project/qemu/-/blob/master/migration/vmstate.c) — `vmstate_save_state()` / `vmstate_load_state()`: VMStateのシリアライズエンジン
- [hw/net/e1000.c](https://gitlab.com/qemu-project/qemu/-/blob/master/hw/net/e1000.c) — `vmstate_e1000`: 本ページで参照した`VMStateDescription`の例
- [qapi/migration.json](https://gitlab.com/qemu-project/qemu/-/blob/master/qapi/migration.json) — `throttle-trigger-threshold`、`cpu-throttle-increment`、`multifd-channels`パラメータの定義

### manページ {#man-pages}

- [`userfaultfd(2)`](https://man7.org/linux/man-pages/man2/userfaultfd.2.html) — QEMUのポストコピーの移行先が、ゲストのRAMを登録して`UFFD_EVENT_PAGEFAULT`を受け取るために使うシステムコール

### 関連ページ {#related-pages}

- [KVMアーキテクチャ](kvm-arch.md) — KVMのioctl、`struct kvm_run`、vCPUのライフサイクル
- [KVMのexit処理](kvm-exits.md) — EPT violationとexit理由のディスパッチ
- [ネストされた仮想化](nested-virt.md) — L1ハイパーバイザをマイグレーションするための`KVM_GET_NESTED_STATE`
- [メモリ仮想化](kvm-memory.md) — EPT/NPT、ダーティページ追跡、バルーンドライバ

### LWNの記事 {#lwn-articles}

- [Page faults in user space: MADV_USERFAULT, remap_anon_range(), and userfaultfd()](https://lwn.net/Articles/615086/) — 2014年の当初の設計に関する議論。「the primary use case here is the live migration of virtual machines running under KVM」（主なユースケースはKVM上で動作する仮想マシンのライブマイグレーションである）と明言している
- [The next steps for userfaultfd()](https://lwn.net/Articles/718198/) — userfaultfdの進化に関する続報。userfaultfdが「was originally created to help with the live migration of virtual machines between physical hosts」（もともと物理ホスト間での仮想マシンのライブマイグレーションを助けるために作られた）ことを改めて述べている
- [Blocking userfaultfd() kernel-fault handling](https://lwn.net/Articles/819834/) — userfaultfdのイベントによって駆動される、マイグレーション中のネットワーク越しのページのデマンドフォールトについて論じている

### 外部資料 {#external}

- [The Definitive KVM API Documentation](https://docs.kernel.org/virt/kvm/api.html) — `KVM_GET_DIRTY_LOG`、`KVM_CLEAR_DIRTY_LOG`、`KVM_CAP_DIRTY_LOG_RING`を扱う公式ioctlリファレンス
- [Userfaultfd](https://docs.kernel.org/admin-guide/mm/userfaultfd.html) — ポストコピーで使われる`MISSING`モードのフォールト処理に関するカーネルadmin-guideのドキュメント
- [QEMU Migration framework](https://www.qemu.org/docs/master/devel/migration/main.html) — VMStateフレームワークとプレコピーの設計に関するQEMUの公式開発者向けドキュメント
- [QEMU Postcopy](https://www.qemu.org/docs/master/devel/migration/postcopy.html) — ポストコピーの内部に関するQEMUの公式開発者向けドキュメント
