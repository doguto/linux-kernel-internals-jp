# vDSOと仮想システムコール {#vdso-and-virtual-system-calls}

> リング遷移なしで、カーネル支援による高速な時刻読み取りなどの高速パスsyscallを実現する

## 問題: 時刻取得のためのsyscallのオーバーヘッド {#the-problem-syscall-overhead-for-time}

`clock_gettime(CLOCK_MONOTONIC, &ts)` は、典型的なサーバープロセスで最も頻繁に呼ばれる関数の1つである。
高頻度のタイマー、レイテンシのヒストグラム、O(1)のスケジューリング判断は、いずれもタイトなループの中でこれを呼ぶ。
本物の `syscall` 命令は、リング遷移、KPTIによるCR3の切り替え、パイプラインのフラッシュ — さらにマイクロアーキテクチャ上の
Spectre緩和策 — のために、最新のx86-64 CPUで1回あたりおよそ100〜200 nsかかる。

`clock_gettime()` を毎秒100万回呼ぶプロセスでは、そのオーバーヘッドだけで、純粋に特権遷移のために毎秒100〜200 msの
CPU時間が消費される。カーネルはこれを**vDSO**で解決する。

## vDSOとは何か {#what-the-vdso-is}

vDSO（virtual Dynamic Shared Object、仮想動的共有オブジェクト）は、カーネルがカーネルイメージに組み込んでビルドし、
`execve()` 時にすべてのプロセスのアドレス空間にマップする小さなELF共有ライブラリである。`/proc/self/maps` で見ることができる。

```bash
cat /proc/self/maps | grep vdso
# 7ffee3a2d000-7ffee3a2f000 r-xp 00000000 00:00 0  [vdso]
```

アドレスは、プロセスが起動するたびにASLRによってランダム化される。vDSOは、時間的に重要なsyscallを
完全にユーザー空間で実装する少数の関数（`nm` や `objdump` で見える）をエクスポートする。

```bash
nm /proc/self/maps   # 直接は動かない。まずvDSOを取り出す:
vdso=$(cat /proc/self/maps | awk '/\[vdso\]/{print $1}' | head -1)
# 実行中のシステムでは、glibc は、exec 時にカーネルが設定する AT_SYSINFO_EHDR
# 補助ベクタのエントリを介して、vDSOを自動的にリンクする。
objdump -T /lib/x86_64-linux-gnu/vdso.so.1 2>/dev/null
# __vdso_clock_gettime
# __vdso_gettimeofday
# __vdso_clock_getres
# __vdso_getcpu
# __vdso_time
```

glibcは起動時に `AT_SYSINFO_EHDR` 補助ベクタのエントリからこれらのシンボルを解決し、
本物のsyscallラッパーの代わりにそれらを使う。

## vvarページ: 共有されるカーネルデータ {#the-vvar-page-shared-kernel-data}

vDSOのマッピングに隣接して**vvar**ページがある — すべてのプロセスにマップされる読み取り専用のページであり、
カーネルが現在の時刻管理の状態で更新する。

```bash
cat /proc/self/maps | grep vvar
# 7ffee3a2b000-7ffee3a2d000 r--p 00000000 00:00 0  [vvar]
```

vvarページは、ユーザー空間では読み取り専用、カーネルでは読み書き可能としてマップされる。vDSOのコードは
`[vvar]` から読み、カーネルのタイマー割り込みがそこに書き込む。マップされたページの読み取りは単なるメモリアクセスなので、
リング遷移は必要ない。

## struct vdso_time_dataとstruct vdso_clock {#struct-vdso_time_data-and-struct-vdso_clock}

現在のカーネルには、単一の統合された `struct vdso_data` は存在しない。vvarページは `struct vdso_time_data` であり、
それが配列として保持するクロックソースごとのseqlock状態が `struct vdso_clock` である — どちらも
`include/vdso/datapage.h` で定義されている。

```c
/* include/vdso/datapage.h */
struct vdso_clock {
    u32             seq;            /* seqlockのシーケンスカウンタ */
    s32             clock_mode;     /* VDSO_CLOCKMODE_TSC, _PVCLOCK, _HVCLOCK, _NONE */
    u64             cycle_last;     /* 最後の更新時のTSC値 */
#ifdef CONFIG_GENERIC_VDSO_OVERFLOW_PROTECT
    u64             max_cycles;     /* 乗算がオーバーフローしうる手前の、安全な最大差分 */
#endif
    u64             mask;           /* TSCマスク（32ビットカウンタ用） */
    u32             mult;           /* TSCからnsへの乗数 */
    u32             shift;          /* mult 用の右シフト量 */
    union {
        struct vdso_timestamp   basetime[VDSO_BASES]; /* クロックごとの基準値 */
        struct timens_offset    offset[VDSO_BASES];   /* 時間namespaceのオフセット */
    };
};

struct vdso_time_data {
    struct arch_vdso_time_data  arch_data;               /* アーキテクチャ固有 */
    struct vdso_clock           clock_data[CS_BASES];     /* クロックソースごとに1つ */
    struct vdso_clock           aux_clock_data[MAX_AUX_CLOCKS]; /* 補助クロックソース */
    s32                         tz_minuteswest; /* タイムゾーン */
    s32                         tz_dsttime;
    u32                         hrtimer_res;    /* hrtimerの分解能（ns） */
    u32                         __unused;
};

struct vdso_timestamp {
    u64     sec;    /* 基準の秒 */
    u64     nsec;   /* 基準のナノ秒（スケール済み） */
};
```

`VDSO_BASES` はサポートされるクロックIDの数であり、各 `vdso_clock` の `basetime[]` はクロックIDごとに
1エントリを保持する。`CS_BASES` は最上位のページが追跡するクロックソースの「ベース」の数（高分解能と、それに対応する
coarse版）である — `clock_data[CS_BASES]` は、vdso_clock/vdso_time_dataの分割によって
「独自のseqlockを持つクロックソースごとの状態」と「一度だけ更新されるページ全体のデータ」が分離される前は、
1つの大きな構造体に埋め込まれた単一のseqlockだったものである。

## seqlockのパターン {#the-seqlock-pattern}

カーネルはvvarページをタイマー割り込みのコンテキストから（そして場合によっては複数のCPUから）更新するので、
vDSOは並行した読み取りに対処しなければならない。vDSOは、生の `seqcount_t`/`seqcount_latch_t` ではなく、
各 `vdso_clock` を範囲とする**seqlock** — ロックレスな読み取り側プロトコル — を使う。`include/vdso/helpers.h` は、
汎用のseqcount-latchヘルパーを呼び出すのではなく、`READ_ONCE()`/`cpu_relax()`/`smp_rmb()` を使って
リトライループを手書きで実装している。

```c
/* include/vdso/helpers.h */
static __always_inline u32 vdso_read_begin(const struct vdso_clock *vc)
{
    u32 seq;

    while (unlikely((seq = READ_ONCE(vc->seq)) & 1))
        cpu_relax();   /* 奇数のシーケンス番号: 書き込み側が更新中なのでスピンする */

    smp_rmb();
    return seq;
}

static __always_inline u32 vdso_read_retry(const struct vdso_clock *vc, u32 start)
{
    u32 seq;

    smp_rmb();
    seq = READ_ONCE(vc->seq);
    return unlikely(seq != start);
}
```

典型的なクロックの読み取りは次のようになる — `do_hres()`（`lib/vdso/gettimeofday.c`）がseqlockの
ループを回し、TSCからナノ秒への実際の計算は同じファイルの `vdso_get_timestamp()` に委ねる。

```c
/* lib/vdso/gettimeofday.c */
bool do_hres(const struct vdso_time_data *vd, const struct vdso_clock *vc,
             clockid_t clk, struct __kernel_timespec *ts)
{
    u64 sec, ns;
    u32 seq;

    do {
        seq = vdso_read_begin(vc);
        if (!vdso_get_timestamp(vd, vc, clk, &sec, &ns))
            return false;   /* clock_mode == VDSO_CLOCKMODE_NONE: 呼び出し元は本物のsyscallにフォールバックする */
    } while (vdso_read_retry(vc, seq));

    vdso_set_timespec(ts, sec, ns);
    return true;
}

static __always_inline bool
vdso_get_timestamp(const struct vdso_time_data *vd, const struct vdso_clock *vc,
                    unsigned int clkidx, u64 *sec, u64 *ns)
{
    const struct vdso_timestamp *vdso_ts = &vc->basetime[clkidx];
    u64 cycles;

    if (unlikely(!vdso_clocksource_ok(vc)))
        return false;

    cycles = __arch_get_hw_counter(vc->clock_mode, vd);
    if (unlikely(!vdso_cycles_ok(cycles)))
        return false;

    *ns  = vdso_calc_ns(vc, cycles, vdso_ts->nsec);  /* TSC差分 → ns、乗算とシフト */
    *sec = vdso_ts->sec;
    return true;
}
```

（実際の `do_hres()` には、上のループの前に `vdso_read_begin_timens()` でチェックされる時間namespace用の分岐もあるが、
このスケッチでは省いている。）`vdso_read_begin()` はシーケンスカウンタを読み、それが奇数（書き込み側がロックを保持している）の間は
スピンする。`vdso_read_retry()` はカウンタを再度読み、それが変化していればtrueを返す。これは読み取りの途中で
カーネルの更新が起き、値に一貫性がないことを意味する。ループは、クリーンな読み取りが完了するまでリトライする
— 実際には、リトライは0回か1回である。

## TSCから実時間への変換 {#tsc-to-wall-clock-conversion}

`vdso_get_timestamp()` は `vdso_calc_ns()` を呼んでナノ秒への変換を行う — 除算を避ける
乗算とシフトによる式である。

```
ns = ((cycles - cycle_last) * mult) >> shift
```

- `mult` と `shift` は、乗算が64ビット整数にオーバーフローせずに収まるよう、カーネル内で
  `clocksource_cyc2ns()` によって事前計算される。
- `cycle_last` は、カーネルが最後に更新したときのTSC値である。
- `ns` を `basetime[clk].nsec` に加えると、その秒の中での現在のナノ秒オフセットが得られる。

カーネルは、NTPによる調整のたびに、そしてタイマー割り込みのたびに（`kernel/time/vsyscall.c` の
`update_vsyscall()` を介して）、`cycle_last`、`mult`、`shift`、`basetime` を更新する。更新中はseqlockの書き込み側を
保持する。

## どのクロックがvDSOで高速化されるか {#which-clocks-are-vdso-accelerated}

| クロックID | vDSOによる高速化 | 備考 |
|----------|-----------------|-------|
| `CLOCK_REALTIME` | あり | 実時間。NTPによる調整で更新される |
| `CLOCK_MONOTONIC` | あり | 決して後退しない |
| `CLOCK_BOOTTIME` | あり | サスペンド時間を含む |
| `CLOCK_REALTIME_COARSE` | あり | 低分解能。`basetime` を直接読み、TSCの計算をしない |
| `CLOCK_MONOTONIC_COARSE` | あり | 同上の単調増加版 |
| `CLOCK_TAI` | あり | 国際原子時（REALTIME + うるう秒オフセット） |
| `CLOCK_PROCESS_CPUTIME_ID` | なし | カーネル内でのタスクごとのアカウンティングが必要 |
| `CLOCK_THREAD_CPUTIME_ID` | なし | 同上 |

coarseクロックはTSCの読み取りを完全にスキップし、事前計算された `basetime` をそのまま返す（分解能は
1 jiffy、約1〜4 ms）。このため、TSCベースのクロックよりさらに安価である。

## VDSO_CLOCKMODE_NONE: 本物のsyscallへのフォールバック {#vdso_clockmode_none-fallback-to-real-syscall}

TSCが信頼できない場合 — たとえば、VMのライブマイグレーションの後（TSCがジャンプしうる）、CPUホットプラグ中、
あるいはカーネルがTSCの不安定さを検出したとき — カーネルは `clock_mode` を `VDSO_CLOCKMODE_NONE` に設定する。
vDSOはこれを検出して `clock_gettime_fallback()` を呼ぶ。これはアーキテクチャ固有であり — x86-64では、共通の
`lib/vdso/gettimeofday.c` ではなく `arch/x86/include/asm/vdso/gettimeofday.h` にある。vDSOはlibcなしで動作し、
libcの `syscall()` を呼べないためである。代わりに `VDSO_SYSCALL2()` マクロを通じて、本物のsyscall命令を
直接発行する。

```c
/* arch/x86/include/asm/vdso/gettimeofday.h */
static __always_inline
long clock_gettime_fallback(clockid_t _clkid, struct __kernel_timespec *_ts)
{
    return VDSO_SYSCALL2(clock_gettime, 64, _clkid, _ts);
}
```

このフォールバックは呼び出し元からは透過的である — 関数のシグネチャも戻り値も同一である。
`VDSO_CLOCKMODE_NONE` は、`/sys/devices/system/clocksource/clocksource0/current_clocksource` に書き込んで
TSCから切り替えることで、手動で引き起こすこともできる。

## vDSOにおけるgetcpu() {#getcpu-in-the-vdso}

`getcpu(cpu, node, NULL)` は、syscallなしで現在のCPUとNUMAノードを返す。x86-64では、
`__vdso_getcpu()` は `vdso_read_cpunode()`（`arch/x86/include/asm/segment.h`）に処理を委ねる。これはデフォルトで
`LSL` 命令を使い — per-CPUのGDTエントリから、CPU/ノードをエンコードしたディスクリプタのリミットを読み取る —
CPUがサポートしていれば `alternative_io` を介して代わりに `RDPID` を使う。`RDTSCP` はこの用途にはまったく使われない。LSLが
選ばれたのは、まさにRDTSCPより速く、すべてのCPUで動作するからである。per-CPUのセットアップ関数
`setup_getcpu()`（`arch/x86/kernel/cpu/common.c`）は、そのGDTエントリを設定する*とともに*、CPUが `RDTSCP` か `RDPID` を
持つ場合は `MSR_TSC_AUX`（RDPIDはこのMSRを直接読む）に書き込む。GS相対のメモリアクセスは
関与しない — GSはカーネルモードのレジスタであり、ユーザー空間はカーネルのGSベースにアクセスできない。

```c
/* arch/x86/entry/vdso/common/vgetcpu.c */
notrace long __vdso_getcpu(unsigned *cpu, unsigned *node, void *unused)
{
    vdso_read_cpunode(cpu, node);
    return 0;
}

/* arch/x86/include/asm/segment.h */
static inline void vdso_read_cpunode(unsigned *cpu, unsigned *node)
{
    unsigned long p;

    /* LSL は RDTSCP より速く、すべてのCPUで動作する。利用可能なら代わりに RDPID を使う */
    alternative_io("lsl %[seg],%k[p]", "rdpid %[p]", X86_FEATURE_RDPID,
                    [p] "=r" (p), [seg] "r" (__CPUNODE_SEG));

    if (cpu)
        *cpu = p & VDSO_CPUNODE_MASK;   /* 0xfff */
    if (node)
        *node = p >> VDSO_CPUNODE_BITS; /* 12 */
}
```

## vDSOの問題のデバッグ {#debugging-vdso-issues}

**straceはclock_gettimeを常に本物のsyscallとして表示する？**: straceはptraceを差し込み、トレーシーに
`SYSCALL_WORK_SYSCALL_TRACE` を設定する。これによりカーネルは `syscall` 命令のパスをインターセプトするが、
vDSOは `syscall` を完全に*迂回する* — vDSO関数は一度もカーネルに入ることなくユーザー空間で実行される。
したがって、vDSOがフォールバックしない限り、straceがvDSOで高速化された呼び出しを目にすることはない。

vDSOが*実際に*フォールバックした場合（例: `VDSO_CLOCKMODE_NONE`）、straceはそのsyscallを通常どおり表示する。

**テストのためにvDSOを無効化する**:

```bash
# ブートパラメータ: vDSOを完全に無効化する
# （カーネルコマンドラインに追加する）
vdso=0

# あるいは実行時に、LD_PRELOAD を介して単一のプロセスに対して:
# __vdso_clock_gettime を、本物のsyscallを呼ぶ版でオーバーライドする
```

**vDSOが有効であることを確認する**:

```c
/* AT_SYSINFO_EHDR 補助ベクタはvDSOのELFヘッダを指す。
   これがなければ、vDSOはマップされていない。 */
#include <sys/auxv.h>
unsigned long vdso_addr = getauxval(AT_SYSINFO_EHDR);
printf("vDSO ELF at: 0x%lx\n", vdso_addr);
```

**vvarのデータを調べる**:

```bash
# vvarページはユーザー空間のツールから直接は読めないが、
# カーネルデバッガなら vdso_time_data 構造体を調べられる:
# （crash、または vmlinux を使った gdb）
# p *((struct vdso_time_data *)vdso_data_ptr)
```

## vsyscall: レガシーな固定アドレスのインターフェース（x86-64のみ） {#vsyscall-legacy-fixed-address-interface-x86-64-only}

vDSO以前、x86-64には**vsyscall**ページがあった。これは固定アドレス `0xffffffffff600000` にマップされたコードで、
`gettimeofday`、`time`、`getcpu` を実装していた。アドレスが固定されていたため、リターン指向プログラミング（ROP）の
ガジェットとして容易に悪用できた。

`vsyscall=` ブートパラメータで制御される3つのモードがある — `native` モードはもう存在しない。
カーネル自身のデフォルトである `CONFIG_LEGACY_VSYSCALL_XONLY` は `vsyscall=xonly` である。このページは実行をトラップして
エミュレートするが読み取りは拒否し、vsyscallページの固定アドレスをデータ読み取りのガジェットとして使う道を塞ぐ。
`vsyscall=emulate`（実行はトラップしてエミュレートし、読み取りも許可する）はまだ存在するが非推奨であり、
ビルド時のデフォルトとしては選べず、コマンドラインからのみ選択できる。`vsyscall=none` はマッピングを
完全に取り除く — それを必要とする古いバイナリは `SIGSEGV` を受け取る。

```bash
# vsyscallモードを確認する（マッピングが存在する限り /proc/self/maps に表示される）
cat /proc/self/maps | grep vsyscall
# ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0  [vsyscall]
```

vsyscallページを使うのは、非常に古いバイナリ（2012年以前のglibc、または古いlibcに静的リンクされたもの）だけである。
現代のソフトウェアはすべてvDSOだけを使う。

## arm64のvDSO {#arm64-vdso}

arm64は `arch/arm64/kernel/vdso/` に独自のvDSOを持つ。仕組みは同一である — 同じ
`struct vdso_time_data`/`struct vdso_clock` のレイアウトを持つvvarページ、seqlock、そしてARMの
汎用タイマー（`CNTVCT_EL0` カウンタレジスタ）を使ったTSC相当のもの。主な違いは次のとおり。

- arm64にはvsyscallのレガシーがない。
- フォールバックは `syscall` ではなく `svc #0` 命令（arm64のシステムコール）を使う。
- arm64は、常時動作する仮想カウンタを使った `VDSO_CLOCKMODE_ARCHTIMER` をサポートする。

`lib/vdso/gettimeofday.c` にある共通のvDSOライブラリコードは、x86-64とarm64の両方向けにコンパイルされる。
アーキテクチャ固有なのは、ハードウェアカウンタの読み取り（`__arch_get_hw_counter`）だけである。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/vdso/datapage.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/vdso/datapage.h) — `struct vdso_clock` と `struct vdso_time_data`: 古い単一の統合 `struct vdso_data` を置き換えた、現在のvvarレイアウト
- [include/vdso/helpers.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/vdso/helpers.h) — `vdso_read_begin()` / `vdso_read_retry()`: seqlockの読み取り側ヘルパー
- [lib/vdso/gettimeofday.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/vdso/gettimeofday.c) — `do_hres()` と `vdso_get_timestamp()`: すべてのアーキテクチャが使う、seqlockで保護されたTSCからtimespecへの共通の読み取りパス
- [arch/x86/entry/vdso/common/vclock_gettime.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/entry/vdso/common/vclock_gettime.c) — `__vdso_clock_gettime()` / `__vdso_gettimeofday()`: x86-64のエントリポイント
- [arch/x86/entry/vdso/common/vgetcpu.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/entry/vdso/common/vgetcpu.c) — `__vdso_getcpu()`
- [arch/x86/include/asm/vdso/gettimeofday.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/vdso/gettimeofday.h) — `clock_gettime_fallback()` と `__arch_get_hw_counter()`: 本物のsyscallへのフォールバックとTSCの読み取り
- [arch/x86/include/asm/segment.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/segment.h) — `vdso_read_cpunode()`: `__vdso_getcpu()` が使う、LSL/RDPIDによる実際のCPU+ノードの取得
- [kernel/time/vsyscall.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/time/vsyscall.c) — `update_vsyscall()`: タイマーティックごとのカーネル側でのvvar更新
- [arch/arm64/kernel/vdso](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/kernel/vdso) — arm64のvDSO実装

### manページ {#man-pages}

- [`vdso(7)`](https://man7.org/linux/man-pages/man7/vdso.7.html) — vDSOの仕組みと `AT_SYSINFO_EHDR` による発見の概要
- [`clock_gettime(2)`](https://man7.org/linux/man-pages/man2/clock_gettime.2.html) — vDSOが高速化するsyscall
- [`gettimeofday(2)`](https://man7.org/linux/man-pages/man2/gettimeofday.2.html) — vDSOが高速化するsyscall
- [`getcpu(2)`](https://man7.org/linux/man-pages/man2/getcpu.2.html) — NOTESの節に、vDSOに支えられた高速パスが記載されている
- [`getauxval(3)`](https://man7.org/linux/man-pages/man3/getauxval.3.html) — プロセス開始時にvDSOの位置を特定するために使われる `AT_SYSINFO_EHDR`

### 関連ページ {#related-pages}

- [syscallのエントリパス](syscall-entry.md) — KPTIとリング遷移のコスト
- [vDSO: 仮想動的共有オブジェクト](../mm/vdso.md) — vDSOのマッピングとアーキテクチャごとの関数テーブルについての、mm側の姉妹ページ
- [時刻管理とクロックソース](../time/timekeeping.md) — vvar更新の背景にあるTSC、クロックソース、NTP
- [時間namespace](../time/time-namespaces.md) — vvarレイアウトで触れた `timens_offset` の仕組み

### LWN記事 {#lwn-articles}

- [On vsyscalls and the vDSO](https://lwn.net/Articles/446528/) — 固定アドレスのvsyscallページを廃止してvDSOに移行すべきセキュリティ上の理由
- [Implementing virtual system calls](https://lwn.net/Articles/615809/) — `__vdso_gettimeofday()` と `__vdso_clock_gettime()` の仕組みと、新しいvDSO関数の追加方法

### 外部資料 {#external}

- [カーネルのブートパラメータ: `vdso=` と `vsyscall=`](https://docs.kernel.org/admin-guide/kernel-parameters.html) — `vdso=0` と `vsyscall={emulate,xonly,none}` ブートオプションの公式ドキュメント
