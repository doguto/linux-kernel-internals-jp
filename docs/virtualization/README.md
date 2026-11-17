# 仮想化: KVM {#virtualization-kvm}

> Linuxカーネルにおけるハードウェア支援型の仮想マシン

## KVMアーキテクチャの概要 {#kvm-architecture-overview}

KVM（Kernel-based Virtual Machine）は、Linuxカーネルに組み込まれたハイパーバイザである。ハードウェア仮想化拡張（Intel VT-x / AMD-V）を使い、ネイティブに近い速度で仮想マシンを実行する。

```
Host (Linux + KVM)
┌────────────────────────────────────────────────────────────┐
│  QEMU (userspace)                                          │
│  ┌────────────────────────────────────────────────────┐    │
│  │ VM process: manages devices, memory layout, VNC    │    │
│  │  │                                                 │    │
│  │  │ ioctl(/dev/kvm, KVM_RUN)                        │    │
│  └──│─────────────────────────────────────────────────┘    │
│     ▼                                                      │
│  KVM kernel module                                         │
│  ┌────────────────────────────────────────────────────┐    │
│  │ vCPU thread: runs in VMX non-root mode             │    │
│  │  ├── VM Entry: switch to guest (vmlaunch/vmresume) │    │
│  │  ├── Guest runs (near-native speed)                │    │
│  │  └── VM Exit: return to host on sensitive ops      │    │
│  └────────────────────────────────────────────────────┘    │
│                                                            │
│  Hardware: Intel VT-x / AMD-V                              │
└────────────────────────────────────────────────────────────┘
```

## このセクションのページ {#pages-in-this-section}

| ページ | 扱う内容 |
|------|----------------|
| [KVMアーキテクチャ](kvm-arch.md) | /dev/kvm API、vCPU、VM exit、VMCS |
| [メモリ仮想化](kvm-memory.md) | EPT/NPT、シャドウページング、バルーン、huge page |
| [virtio](virtio.md) | 準仮想化I/O、virtqueue、vhostによる高速化 |
| [VFIOデバイスパススルー](vfio.md) | IOMMUグループ、DMAリマッピング、VFIOのコンテナ階層 |
| [KVMのexit処理](kvm-exits.md) | exit理由、I/Oエミュレーション、MMIO、割り込み注入 |
| [KVMライブマイグレーション](live-migration.md) | ダーティログ、プレコピー、CPU状態の転送、ポストコピー |
| [ネストされた仮想化](nested-virt.md) | L0/L1/L2、vmcs12、vmcs02へのマージ、2段階EPT |
| [事件簿](war-stories.md) | VENOM、L1TF、nested VMXのUAF、vhostのオーバーフロー、そしてMDS緩和策の5年越しのタイミングの隙 |

## クイックリファレンス {#quick-reference}

```bash
# KVMサポートを確認
lsmod | grep kvm
# kvm_intel   or  kvm_amd
# kvm

ls /dev/kvm  # 存在している必要がある

# ハードウェア仮想化を確認
grep -m1 -E "vmx|svm" /proc/cpuinfo

# 実行中のVMを一覧表示（QEMU/KVM）
virsh list --all
qemu-img info disk.qcow2

# VMのメモリ使用量
virsh domstats --list-running --balloon
```
