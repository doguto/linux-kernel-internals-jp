# コンテナ隔離の内部 {#container-isolation-internals}

> Docker、Kubernetes、コンテナランタイムが namespace と cgroup をどう使うか

## カーネルから見たコンテナとは何か {#what-a-container-is-kernel-side}

コンテナとは、次の条件を満たすプロセス（またはプロセスツリー）である:

1. 一連の **namespace**（システムリソースの隔離されたビュー）の中に存在する
2. **cgroup**（リソース制限）によって管理される
3. **ファイルシステムのルート**を持つ（mount namespace + chroot/pivot_root による）
4. 必要に応じて **seccomp**（システムコールのフィルタリング）と **LSM**（AppArmor/SELinux）によって制限される

「コンテナ」というカーネルオブジェクトは存在しない。既存のプリミティブの組み合わせである。

```
                    Container
    ┌─────────────────────────────────────────┐
    │  PID namespace: PIDs 1,2,3,...          │
    │  Mount namespace: / → container rootfs  │
    │  Net namespace: eth0=veth pair          │
    │  User namespace: uid 0=container root   │
    │  UTS namespace: hostname=container-1    │
    │  IPC namespace: isolated SysV/POSIX     │
    │  Cgroup namespace: /sys/fs/cgroup view  │
    ├─────────────────────────────────────────┤
    │  Cgroup: /sys/fs/cgroup/containers/c1/  │
    │    cpu.max = 500000/1000000 (50%)       │
    │    memory.max = 512M                    │
    │    pids.max = 100                       │
    ├─────────────────────────────────────────┤
    │  seccomp: syscall whitelist/blacklist   │
    │  AppArmor/SELinux: MAC policy           │
    └─────────────────────────────────────────┘
```

## コンテナ作成の手順 {#container-creation-sequence}

`docker run` や `runc` がカーネルレベルで行っていることは次のとおりである:

```c
/* 簡略化した runc のコンテナ作成 */

/* 1. clone() で新しい namespace を作成する */
pid_t child = clone(container_init,
    stack + STACK_SIZE,
    CLONE_NEWPID    /* 新しい PID namespace */
    | CLONE_NEWNS   /* 新しい mount namespace */
    | CLONE_NEWNET  /* 新しい network namespace */
    | CLONE_NEWIPC  /* 新しい IPC namespace */
    | CLONE_NEWUTS  /* 新しい UTS namespace */
    | CLONE_NEWCGROUP  /* 新しい cgroup namespace */
    | SIGCHLD,
    &config);

/* 2. cgroup を設定する（親の namespace から） */
/* cgroup.procs に pid を書き込む */
write_to_file("/sys/fs/cgroup/containers/c1/cgroup.procs", pid_str);

/* 3. ネットワークを設定する（veth ペア、親の namespace から） */
/* runc は netlink を呼んで veth0/veth1 を作成し、veth1 を子の netns に移す */

/* 4. 子で: コンテナのファイルシステムに pivot_root する */
/* コンテナの rootfs をマウントし、pivot_root または chroot を呼ぶ */

/* 5. 子で: setuid/setgid し、ケーパビリティを落とす */

/* 6. 子で: seccomp フィルタをインストールする */
prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog);

/* 7. コンテナのエントリポイントを exec する */
execve("/entrypoint", argv, envp);
```

## pivot_root: ファイルシステムのルートの切り替え {#pivot_root-switching-filesystem-root}

コンテナはファイルシステムを完全に隔離するために、（`chroot` だけでなく）`pivot_root` を使う:

```c
/* コンテナの init: */

/* 1. コンテナの rootfs を一時的な場所にマウントする */
mount(rootfs_path, "/new_root", "none", MS_BIND, NULL);

/* 2. mount namespace をプライベートにする（ホストへの伝播を防ぐ） */
mount("", "/", "none", MS_PRIVATE | MS_REC, NULL);

/* 3. 古いルートを新しいルート配下のどこかに置く */
mkdir("/new_root/.old_root", 0755);

/* 4. pivot_root: new_root が / になり、古いルートは /.old_root に移る */
syscall(SYS_pivot_root, "/new_root", "/new_root/.old_root");

/* 5. 古いルートをアンマウントして隔離を完了する */
umount2("/.old_root", MNT_DETACH);
rmdir("/.old_root");

/* ここで /proc、/sys、/dev を新たにマウントする必要がある */
mount("proc", "/proc", "proc", 0, NULL);
mount("sysfs", "/sys", "sysfs", 0, NULL);
mount("tmpfs", "/dev", "tmpfs", 0, NULL);
```

## ネットワークの設定: veth ペア {#network-setup-veth-pairs}

コンテナのネットワークは仮想イーサネットのペアを使う:

```
Host namespace                    Container namespace
─────────────────                 ──────────────────
veth0 (10.0.0.1)  ←──────────→  eth0 (172.17.0.2)
    │                  kernel
    │                  bridge
docker0 bridge
(172.17.0.1)
    │
    └── routes, NAT (iptables MASQUERADE)
```

```bash
# runc がコンテナのネットワークのために行うこと（簡略化）:
ip link add veth0 type veth peer name eth0
ip link set eth0 netns <container_pid>

# ホストの namespace で:
ip link set veth0 master docker0
ip link set veth0 up

# コンテナの namespace で:
ip netns exec <container_pid> ip addr add 172.17.0.2/16 dev eth0
ip netns exec <container_pid> ip route add default via 172.17.0.1
```

## cgroup namespace: コンテナから見た cgroup {#cgroup-namespace-containers-view-of-cgroups}

cgroup namespace がなければ、コンテナからホストの cgroup 階層全体が見えてしまう。`CLONE_NEWCGROUP` を使うと:

```bash
# ホストからの見え方:
cat /proc/1/cgroup
# 0::/

# コンテナ（自身の cgroup namespace 内）からの見え方:
cat /proc/1/cgroup  # コンテナ内から
# 0::/              ← コンテナ自身の cgroup がルートのように見える

# コンテナ内の /sys/fs/cgroup/ には自身のサブツリーだけが見える
```

cgroup namespace はコンテナの cgroup をルートのように見せるので、コンテナ内の `systemd` からはホストの階層が見えない。

## user namespace: rootless コンテナ {#user-namespace-rootless-containers}

rootless コンテナ（Docker rootless、Podman）は、完全に非特権ユーザーとして動作する:

```
Host:     uid=1000 (user)
          uid=100000-165535 (subuid range)

Container: uid=0 (root inside)
           = uid=100000 outside  (via uid_map: 0 → 100000, range 65536)
```

```bash
# /etc/subuid はユーザーを従属 UID の範囲にマップする
cat /etc/subuid
# user:100000:65536

# rootless コンテナを起動する
podman run --rm -it ubuntu bash
# 内部: id → uid=0(root)
# 外部: ps ではプロセスが uid=100000 として動いているように見える

# カーネルは次を強制する: この「root」はホスト上の特権を持たない
# /etc/shadow へのアクセスやモジュールのロードなどはできない
```

user namespace は、他のすべての種類の namespace を非特権で作成することも可能にする。user namespace は Linux 3.9 で非特権ユーザーからも完全に機能するようになった [(LWN)](https://lwn.net/Articles/532593/)。

## seccomp: システムコールのフィルタリング {#seccomp-syscall-filtering}

seccomp は、コンテナが発行できるシステムコールを制限する:

```c
/* seccomp フィルタをインストールする（システムコール引数に対する BPF プログラム） */
struct sock_filter filter[] = {
    /* システムコール番号をロード */
    BPF_STMT(BPF_LD | BPF_W | BPF_ABS,
             offsetof(struct seccomp_data, nr)),

    /* read、write、exit、sigreturn を許可 */
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_read, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_write, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    /* デフォルト: kill */
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL),
};

struct sock_fprog prog = {
    .len = ARRAY_SIZE(filter),
    .filter = filter,
};
prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog);
```

Docker のデフォルトの seccomp プロファイルは、約40個のシステムコール（`keyctl`、`kexec_load`、`perf_event_open`、`ptrace` など）をブロックする。

## OCI ランタイム仕様 {#oci-runtime-spec}

Open Container Initiative（OCI）のランタイム仕様は、コンテナランタイム（runc、crun、kata）が行うべきことを定義する。仕様は JSON ファイル（`config.json`）である:

```json
{
  "ociVersion": "1.0.0",
  "process": {
    "user": {"uid": 0, "gid": 0},
    "args": ["/bin/sh"],
    "capabilities": {
      "bounding": ["CAP_NET_BIND_SERVICE", "CAP_KILL", ...]
    },
    "rlimits": [{"type": "RLIMIT_NOFILE", "hard": 1024, "soft": 1024}],
    "seccompProfile": "default.json"
  },
  "root": {"path": "rootfs", "readonly": true},
  "mounts": [
    {"destination": "/proc", "type": "proc", "source": "proc"},
    {"destination": "/dev", "type": "tmpfs", "source": "tmpfs"}
  ],
  "linux": {
    "namespaces": [
      {"type": "pid"}, {"type": "network"}, {"type": "ipc"},
      {"type": "uts"}, {"type": "mount"}, {"type": "cgroup"}
    ],
    "cgroupsPath": "/containers/mycontainer",
    "resources": {
      "memory": {"limit": 536870912},
      "cpu": {"quota": 200000, "period": 1000000}
    }
  }
}
```

## コンテナ隔離のデバッグ {#debugging-container-isolation}

```bash
# コンテナのプロセスがどの namespace に属しているか確認する
PID=$(docker inspect --format '{{.State.Pid}}' mycontainer)
ls -la /proc/$PID/ns/

# コンテナの namespace に入る
nsenter --target $PID --mount --pid --net -- bash

# コンテナのプロセスの cgroup を確認する
cat /proc/$PID/cgroup

# コンテナの cgroup の制限を見る
cat /sys/fs/cgroup/$(cat /proc/$PID/cgroup | cut -d: -f3)/memory.max

# ホストからコンテナのプロセスを strace する
strace -p $PID

# インストールされた seccomp フィルタを確認する
cat /proc/$PID/status | grep Seccomp
# Seccomp: 2    ← 0=なし、1=strict、2=filter

# ケーパビリティを確認する
cat /proc/$PID/status | grep Cap
# CapPrm: 00000000a80425fb
capsh --decode=00000000a80425fb
```

## 参考文献 {#further-reading}

- [cgroup v2 のアーキテクチャ](cgroup-v2.md) — リソース制御の内部
- [リソースコントローラ](cgroup-controllers.md) — cpu、memory、io の制限
- [namespace](namespaces.md) — 全8種類の namespace の詳細
- [BPF/eBPF](../bpf/README.md) — seccomp-bpf と cgroup BPF プログラム
- `runc` のソースコード — OCI コンテナランタイムのリファレンス実装
