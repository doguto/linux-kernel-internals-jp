# cgroupとnamespace {#cgroups-namespaces}

> コンテナを支えるリソース制御と隔離のプリミティブ

## cgroupとnamespaceが行うこと {#what-cgroups-and-namespaces-do}

**cgroup（control group）** は、プロセスのグループが *どれだけ* リソースを使えるかを制限する:

- CPU時間（cpu.weight, cpu.max）
- メモリ（memory.max, memory.swap.max）
- I/O帯域（io.weight, io.max）
- プロセス数（pids.max）

**namespace** は、プロセスから *何が* 見えるかを制御する:

- どのプロセスが見えるか（PID namespace）
- どのようなファイルシステムのビューを持つか（mount namespace）
- どのネットワークインターフェースとアドレスを持つか（network namespace）
- ホスト名とドメイン名（UTS namespace）
- ユーザー/グループID（user namespace）

この2つを組み合わせてコンテナの隔離が実現される。Docker、KubernetesのPod、systemdのサービスはこれらのプリミティブの上に構築されている。

```
┌──────────────────────────────────────────────────────────────┐
│  Container / Pod                                             │
│  ┌─────────────────────────┐  ┌────────────────────────────┐ │
│  │ Namespaces (isolation)  │  │ Cgroup (resource limits)   │ │
│  │  - PID: pid 1 inside    │  │  - cpu.max 200000/1000000  │ │
│  │  - mnt: own filesystem  │  │  - memory.max 512M         │ │
│  │  - net: own veth pair   │  │  - pids.max 100            │ │
│  │  - user: uid 0 → 1000   │  │                            │ │
│  └─────────────────────────┘  └────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

## このセクションのページ {#pages-in-this-section}

| ページ | 扱う内容 |
|------|----------------|
| [cgroup v2のアーキテクチャ](cgroup-v2.md) | 階層、コントローラ、struct cgroup |
| [リソースコントローラ](cgroup-controllers.md) | cpu、memory、io、pidsの各コントローラの詳細 |
| [namespace](namespaces.md) | 全8種類のnamespace、clone/unshare/setns |
| [コンテナの隔離](container-isolation.md) | namespaceとcgroupを組み合わせてコンテナを実現する方法 |
| [ioコントローラ](io-cgroup.md) | blkio、io.weight、io.max、io.latency、BFQ |
| [cgroup BPFプログラム](cgroup-bpf.md) | eBPFのアタッチタイプ、デバイスポリシー、ネットワークフィルタリング |
| [systemdとcgroupの統合](systemd-cgroups.md) | slice、scope、service、委譲 |
| [事件簿](war-stories.md) | OOMループ、CFSスロットリング、v1→v2移行の苦労 |

## クイックリファレンス {#quick-reference}

```bash
# cgroup v2 ファイルシステム
mount | grep cgroup2
# cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime)

# 現在のプロセスの cgroup
cat /proc/self/cgroup
# 0::/user.slice/user-1000.slice/session-1.scope

# systemd は cgroup を自動的に作成する
systemctl status nginx
# CGroup: /system.slice/nginx.service

# 現在のプロセスの namespace
ls -la /proc/self/ns/
# lrwxrwxrwx ... cgroup -> 'cgroup:[4026531835]'
# lrwxrwxrwx ... mnt -> 'mnt:[4026531840]'
# lrwxrwxrwx ... net -> 'net:[4026531992]'
# lrwxrwxrwx ... pid -> 'pid:[4026531836]'
# lrwxrwxrwx ... user -> 'user:[4026531837]'
# lrwxrwxrwx ... uts -> 'uts:[4026531838]'
# lrwxrwxrwx ... ipc -> 'ipc:[4026531839]'
# lrwxrwxrwx ... time -> 'time:[4026531834]'
```
