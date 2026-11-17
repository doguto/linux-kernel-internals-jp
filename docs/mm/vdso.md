# vDSO: Virtual Dynamic Shared Object {#vdso-virtual-dynamic-shared-object}

> カーネルに入らないシステムコール: ユーザー空間の速度で動くgettimeofday

## 問題 {#the-problem}

システムコールは毎回モード切り替え（ユーザー → カーネル → ユーザー）を必要とする。`gettimeofday()` の場合、これは次を意味する。

- レジスタを退避し、特権レベルを変更する（CPL 0）
- カーネル内で時刻を参照する
- レジスタを復元し、ユーザー空間へ戻る

高頻度で時刻を問い合わせる用途（トレーディングシステム、プロファイラ、ゲームループ）では、この往復に約100〜300nsかかる。vDSOはこれをなくす。

## vDSOとは何か {#what-is-the-vdso}

vDSO（virtual dynamic shared object）は、カーネルが **すべてのプロセスのアドレス空間に** 自動的にマップする小さな共有ライブラリである。いくつかの時刻関連関数を含んでおり、これらは純粋なユーザー空間コードとして実装され、カーネルが管理する共有メモリ領域から値を読む。システムコールは不要である。

```bash
# すべてのプロセスで vDSO のマッピングを確認する
cat /proc/self/maps | grep vdso
# 7fff12345000-7fff12346000 r-xp  [vdso]

# vDSO を取り出して逆アセンブルする
objcopy --dump-section .text=/tmp/vdso.so [vdso] /dev/null 2>/dev/null
# または:
dd if=/proc/self/mem bs=4096 count=1 skip=$((0x7fff12345000/4096)) > /tmp/vdso.so
```

## vDSOに含まれる関数 {#functions-in-the-vdso}

```c
/* vDSOは次のシンボルをエクスポートする（x86-64）: */
__vdso_clock_gettime()    /* clock_gettime(CLOCK_REALTIME, ...)  — 高速 */
__vdso_clock_getres()     /* clock_getres()                      — 高速 */
__vdso_gettimeofday()     /* gettimeofday()                      — 高速 */
__vdso_time()             /* time()                              — 高速 */
__vdso_getcpu()           /* getcpu() — 現在のCPU/NUMAノード      — 高速 */

/* これらはカーネルの代わりにvDSOを呼び出す: */
/* glibcはこれらすべてをラップし、自動的にvDSOを使う */
```

## vdso_data共有ページ {#the-vdso_data-shared-page}

カーネルは **vvar**（vDSO variable）ページを管理している。このページは次の性質を持つ。

- すべてのプロセスに読み取り専用でマップされる
- カーネルによって書き込まれる（現在時刻を保持する）
- vDSOの関数によって読まれる（システムコールなし）

```c
/* arch/x86/include/asm/vvar.h */
/* arch/x86/entry/vdso/vma.c */
struct vdso_data {
    u32 seq;              /* seqlockのシーケンスカウンタ */

    s32 clock_mode;       /* VDSO_CLOCKMODE_* */
    u64 cycle_last;       /* 最終更新時のTSC値 */
    u64 mask;             /* TSCのビットマスク */
    u32 mult;             /* TSC → ns の乗数 */
    u32 shift;            /* TSC → ns のシフト量 */

    /* clockidごとのデータ: */
    struct {
        u64 nsec;         /* 1秒未満のナノ秒 */
        u64 sec;          /* エポックからの秒数 */
        u64 snsec;        /* CLOCK_MONOTONICのオフセット */
    } basetime[VDSO_BASES];

    s32 tz_minuteswest;   /* タイムゾーン */
    s32 tz_dsttime;
    u32 hrtimer_res;
    u32 __unused;
};

/* NMI安全性のための2つのコピー: */
extern struct vdso_data _vdso_data[CS_BASES];  /* vmlinux内 */
/* 各プロセスの[vvar]ページにマップされる */
```

## seqlockパターン {#the-seqlock-pattern}

カーネルが `vdso_data` を更新する一方で、ユーザーコードはそれを並行して読む。**seqlock** が一貫した読み取りを保証する。

```c
/* カーネル（timekeeping.c）: 書き込み側 */
void update_vsyscall(struct timekeeper *tk)
{
    struct vdso_data *vdata = __arch_get_k_vdso_data();

    /* seqを奇数にインクリメント: 「更新中」とマークする */
    vdso_write_begin(vdata);

    /* すべてのフィールドを更新する */
    vdata->cycle_last = tk->tkr_mono.cycle_last;
    vdata->mult       = tk->tkr_mono.mult;
    vdata->shift      = tk->tkr_mono.shift;
    /* ... */

    /* seqを偶数にインクリメント: 「更新完了」 */
    vdso_write_end(vdata);
}
```

```c
/* vDSOのユーザー空間側: 読み取り側（[vdso]マッピング内） */
notrace static __always_inline int
do_hres(const struct vdso_data *vd, clockid_t clk, struct __kernel_timespec *ts)
{
    const struct vdso_timestamp *vdso_ts = &vd->basetime[clk];
    u64 cycles, ns;
    u32 seq;

    do {
        /* seqを読む: 偶数（更新中でない）でなければならない */
        seq = vdso_read_begin(vd);
        if (unlikely(vd->clock_mode == VDSO_CLOCKMODE_NONE))
            return -1;  /* システムコールへフォールバック */

        /* TSCを読む */
        cycles = __arch_get_hw_counter(vd->clock_mode, vd);

        /* ナノ秒を計算: (cycles - cycle_last) * mult >> shift */
        ns = vdso_ts->nsec;
        ns += (cycles - vd->cycle_last) * vd->mult;
        ns >>= vd->shift;

        ts->tv_sec  = vdso_ts->sec;
        ts->tv_nsec = ns;

    } while (unlikely(vdso_read_retry(vd, seq)));
    /* 読み取り途中でseqが変わったら（カーネルが更新中だった）リトライする */

    return 0;
}
```

鍵となるのはリトライループである。読み取り中にカーネルが `vdso_data` を更新した（seqが変わった）場合はリトライする。通常リトライは0回である。

## AT_SYSINFO_EHDR: vDSOの位置を知る {#atsysinfoehdr-finding-the-vdso}

カーネルはvDSOのアドレスをELFの補助ベクタ（auxiliary vector）経由で渡す。

```c
/* プログラム起動時の補助ベクタのエントリ */
AT_SYSINFO_EHDR = 33   /* vDSOのELFヘッダのアドレス */

/* glibcは起動時にこれを読む: */
void __dl_discover_osversion(void)
{
    ElfW(Phdr) *phdr = (void *)getauxval(AT_SYSINFO_EHDR);
    /* dlopenに似た方法でvDSOをロードする */
}
```

```c
/* /proc/self/auxv から補助ベクタを読む: */
#include <sys/auxv.h>
unsigned long vdso_addr = getauxval(AT_SYSINFO_EHDR);
printf("vDSO at %#lx\n", vdso_addr);

/* または /proc/self/auxv を手動で解析する: */
#include <elf.h>
Elf64_auxv_t auxv;
int fd = open("/proc/self/auxv", O_RDONLY);
while (read(fd, &auxv, sizeof(auxv)) == sizeof(auxv)) {
    if (auxv.a_type == AT_SYSINFO_EHDR) {
        printf("vDSO ELF header at %#lx\n", auxv.a_un.a_val);
        break;
    }
}
```

## vDSOからシステムコールへのフォールバック {#vdso-fallback-to-syscall}

vDSOが答えを出せない場合（たとえばクロックがTSCベースでない、あるいは非単調クロックである場合）、本物のシステムコールにフォールバックする。

```c
/* arch/x86/entry/vdso/vclock_gettime.c */
notrace int __vdso_clock_gettime(clockid_t clock, struct __kernel_timespec *ts)
{
    const struct vdso_data *vd = __arch_get_vdso_data();

    switch (clock) {
    case CLOCK_REALTIME:
    case CLOCK_MONOTONIC:
    case CLOCK_BOOTTIME:
    case CLOCK_TAI:
        /* TSCによる高速パス */
        if (likely(do_hres(vd + clock, clock, ts) == 0))
            return 0;
        break;

    case CLOCK_REALTIME_COARSE:
    case CLOCK_MONOTONIC_COARSE:
        /* 粗い精度: TSCを使わずvdso_dataから読むだけ */
        do_coarse(vd + clock, clock, ts);
        return 0;

    default:
        break;
    }

    /* フォールバック: 本物のシステムコール */
    return clock_gettime_fallback(clock, ts);
}
```

## 性能比較 {#performance-comparison}

```c
/* ベンチマーク: システムコール vs vDSO vs rdtsc */

/* システムコールのgettimeofday: 約150〜300ns */
struct timeval tv;
gettimeofday(&tv, NULL);  /* glibcでは自動的にvDSOを使う */

/* vDSOの直接呼び出し（glibcのgettimeofdayと同じ）: 約5〜15ns */

/* 生のRDTSC（最速だが単位がない）: 約2〜5ns */
uint64_t t = __rdtsc();
```

```bash
# gettimeofday のレイテンシを測定する
perf stat -e syscalls:sys_enter_gettimeofday -- \
    python3 -c "import time; [time.time() for _ in range(1000000)]"
# syscalls:sys_enter_gettimeofday: 0  (システムコールはゼロ — すべて vDSO!)

# システムコール経路を強制する（vDSO を迂回する）:
LD_PRELOAD=/lib/x86_64-linux-gnu/libpthread.so.0  # もう機能しない
# seccomp で横取りするか、strace を使う:
strace -c -e gettimeofday myprogram  # vDSO が機能していれば 0 と表示される
```

## 他のアーキテクチャにおけるvDSO {#vdso-on-other-architectures}

vDSOはアーキテクチャ固有である。

| アーキテクチャ | vDSOの関数 |
|-------------|---------------|
| x86-64 | clock_gettime, gettimeofday, time, getcpu |
| ARM64 | clock_gettime, gettimeofday, clock_getres |
| RISC-V | clock_gettime, gettimeofday |
| Power | clock_gettime, gettimeofday, getcpu |
| s390 | clock_gettime, gettimeofday |

```bash
# ARM64 の vDSO 関数
nm /proc/self/mem 2>/dev/null | grep vdso || \
    objdump -d /proc/self/mem 2>/dev/null | head
# または:
cat /proc/self/maps | grep vdso
# 取り出して調べる:
dd if=/proc/self/mem of=/tmp/vdso.so bs=4096 count=1 \
   skip=$(($(cat /proc/self/maps | grep vdso | cut -d- -f1 | head -1 | xargs printf '%d')/4096)) 2>/dev/null
nm /tmp/vdso.so
```

## 参考文献 {#further-reading}

- [時刻管理とクロックソース](../time/timekeeping.md) — カーネルが時刻をどう管理するか
- [プロセスのアドレス空間](mmap.md) — プロセスのマップにおけるvDSOのVMA
- [exec()の間に何が起きるか](exec.md) — 補助ベクタ中のAT_SYSINFO_EHDR
- [システムコールのエントリパス](../syscalls/syscall-entry.md) — vDSOがシステムコールにフォールバックするとき
- `arch/x86/entry/vdso/` — x86-64のvDSO実装
- `include/vdso/datapage.h` — struct vdso_data
