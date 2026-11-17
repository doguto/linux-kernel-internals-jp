# BPF/eBPF {#bpfebpf}

> Linuxカーネルへの、安全でプログラム可能なフック

## eBPFとは何か {#what-is-ebpf}

eBPF（extended Berkeley Packet Filter）を使うと、カーネルモジュールを書かずに、サンドボックス化されたプログラムをカーネル内で実行できる。プログラムは次のような性質を持つ。

- ロード前に**検証される** — カーネルは、プログラムが必ず終了し、不正なメモリに決してアクセスしないことを証明する
- **JITコンパイルされる** — 初回ロード後はネイティブマシンコードの性能で動く
- **イベント駆動である** — フック（システムコール、ネットワーク、tracepoint、kprobe）にアタッチされる
- **観測可能である** — mapを介してユーザー空間とデータを共有する

```
Userspace                          Kernel
─────────────────────────────────────────────────────────
 C source                         Event hook fires
    │                                  │
    ▼                                  ▼
 clang -target bpf                BPF program runs
    │                                  │
    ▼                              reads/writes
 .o (BPF bytecode)               BPF maps ◄──────── userspace reads
    │
    ▼
 libbpf / bpf() syscall
    │
    ▼
 Verifier (safety check)
    │
    ▼
 JIT compiler → native code
    │
    ▼
 Attach to hook
```

## このセクションのページ {#pages-in-this-section}

| ページ | 扱う内容 |
|------|----------------|
| [アーキテクチャとプログラムタイプ](bpf-overview.md) | bpf()システムコール、BPF_PROG_TYPE_*、ロードの流れ |
| [BPF Maps](bpf-maps.md) | ハッシュ、配列、ringbuf、per-CPU map、ピン留め |
| [BPF Verifier](bpf-verifier.md) | 安全性の証明、レジスタ追跡、ポインタの制限 |
| [BPFセキュリティモデル](bpf-security-model.md) | ケーパビリティ、非特権BPFのsysctl、BPF LSM |
| [BTFとCO-RE](btf-core.md) | 型情報、カーネルバージョン間で移植可能なプログラム |
| [ヘルパー、kfunc、JIT](bpf-helpers-kfuncs-jit.md) | `BPF_CALL`の呼び出し規約、kfuncの仕組み、プログラムがネイティブコードにコンパイルされる方法 |
| [libbpfとスケルトン](libbpf.md) | ユーザー空間API、BPFスケルトンのワークフロー |
| [BPFリングバッファ](bpf-ringbuf.md) | `BPF_MAP_TYPE_RINGBUF`、予約モデル、epoll通知 |
| [BPFネットワーキング](bpf-networking.md) | TC、cgroupフック、sockmap |
| [事件簿](war-stories.md) | VerifierのCVE、BPFにおけるSpectre、そして非特権BPFがデフォルトで無効化された理由 |

## 簡単な見取り図 {#quick-orientation}

**BPFがフックする場所:**

```
Network receive    → XDP (before skb allocation)
                   → TC (after skb, before routing)
                   → socket filter
Syscall entry/exit → tracepoint sys_enter_*/sys_exit_*
                   → seccomp-bpf
Kernel functions   → kprobe/kretprobe
                   → fentry/fexit (BTF-based, lower overhead)
User functions     → uprobe/uretprobe
Perf events        → perf_event (PMU, hardware counters)
Cgroups            → cgroup/skb, cgroup/sock, cgroup/connect
Scheduler          → sched_ext (BPF-defined schedulers, 6.12+)
```

**主要なカーネルファイル:**

- `kernel/bpf/verifier.c` — 安全性チェッカー
- `kernel/bpf/core.c` — インタプリタとJITのグルーコード
- `kernel/bpf/syscall.c` — bpf()システムコールの実装
- `kernel/bpf/helpers.c` — bpf_*ヘルパー関数
- `arch/x86/net/bpf_jit_comp.c` — x86-64 JITコンパイラ
