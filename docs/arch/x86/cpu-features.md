# x86のCPU機能とalternative {#x86-cpu-features-and-alternatives}

> CPUID、機能フラグ、alternativeパッチング、そしてハードウェアの能力

カーネルは、数十年にわたるx86の進化にまたがるCPU — 初期のPentium 4から最新のAMD ZenやIntel Coreの世代まで — の上で動作しなければならない。CPU機能の検出とalternativeパッチングの基盤により、カーネルは一度コンパイルするだけで、ブート時に現在のハードウェアに合わせて自動的に自身を調整できる。

---

## CPUID: CPUの能力を問い合わせる {#cpuid-querying-cpu-capabilities}

`CPUID`は、CPUの能力を列挙するためのx86の命令である。呼び出し側は`CPUID`を実行する前に、**リーフ番号**を`EAX`に（場合によってはサブリーフを`ECX`に）ロードする。CPUは能力ビットを`EAX`、`EBX`、`ECX`、`EDX`に返す。

```
CPUID usage:
  Input:  EAX = leaf, ECX = subleaf (for leaves that use it)
  Output: EAX, EBX, ECX, EDX = feature bits (leaf-defined)

Key leaves:
  EAX=0:           Maximum supported leaf; CPU vendor string in EBX/ECX/EDX
  EAX=1:           Basic feature bits
                    EDX: FPU, VME, PSE, TSC, MSR, PAE, MCE, CX8, APIC, SEP,
                         MTRR, PGE, MCA, CMOV, PAT, PSE-36, PSN, CLFSH, DS,
                         ACPI, MMX, FXSR, SSE, SSE2, SS, HTT, TM, IA64, PBE
                    ECX: SSE3, PCLMULQDQ, DTES64, MONITOR, DS-CPL, VMX, SMX,
                         EIST, TM2, SSSE3, CNXT-ID, SDBG, FMA, CX16, xTPR,
                         PDCM, PCID, DCA, SSE4.1, SSE4.2, x2APIC, MOVBE,
                         POPCNT, TSC-Deadline, AES, XSAVE, OSXSAVE, AVX,
                         F16C, RDRAND, Hypervisor
  EAX=7, ECX=0:    Extended features
                    EBX: FSGSBASE, TSC_ADJUST, SGX, AVX2, FDP_EXCPTN_ONLY,
                         SMEP, BMI2, ERMS, INVPCID, RTM, MPX, AVX512F,
                         AVX512DQ, RDSEED, ADX, SMAP, AVX512IFMA, CLFLUSHOPT,
                         CLWB, INTEL_PT, AVX512PF, AVX512ER, AVX512CD,
                         SHA_NI, AVX512BW, AVX512VL
                    ECX: PKU, OSPKE, WAITPKG, AVX512_VBMI2, SHSTK, GFNI,
                         VAES, VPCLMULQDQ, AVX512_VNNI, AVX512_BITALG,
                         AVX512_VPOPCNTDQ, LA57, RDPID, BUS_LOCK_DETECT,
                         CLDEMOTE, MOVDIRI, MOVDIR64B, ENQCMD, SGX_LC, PKS
  EAX=0x80000001:  AMD extended features (also used by Intel for some bits)
                    EDX: SYSCALL, NX, MMXEXT, FXSR_OPT, PDPE1GB, RDTSCP,
                         LM (long mode = 64-bit), 3DNOWEXT, 3DNOW
                    ECX: LAHF_LM, CMP_LEGACY, SVM, EXTAPIC, CR8_LEGACY,
                         ABM (LZCNT), SSE4A, MISALIGNSSE, 3DNOWPREFETCH,
                         OSVW, IBS, XOP, SKINIT, WDT, LWP, FMA4, TCE, NODEID,
                         TBM, TOPOEXT, PERFCTR_CORE, PERFCTR_NB
```

カーネルはブートの初期段階で、`arch/x86/kernel/cpu/common.c`内から`cpuid()`および`cpuid_count()`ヘルパーを介してCPUIDを呼ぶ。

```c
/* arch/x86/include/asm/processor.h */
static inline void cpuid(unsigned int op,
                          unsigned int *eax, unsigned int *ebx,
                          unsigned int *ecx, unsigned int *edx)
{
    *eax = op;
    *ecx = 0;
    __cpuid(eax, ebx, ecx, edx);
}

static inline void cpuid_count(unsigned int op, int count,
                                unsigned int *eax, unsigned int *ebx,
                                unsigned int *ecx, unsigned int *edx)
{
    *eax = op;
    *ecx = count;
    __cpuid(eax, ebx, ecx, edx);
}
```

---

## x86_capability[]: カーネルの機能ビットマスク {#x86_capability-the-kernels-feature-bitmask}

すべてのCPUIDリーフを解析した後、カーネルはCPU機能を`struct cpuinfo_x86`内の`x86_capability`配列に格納する。

```c
/* arch/x86/include/asm/processor.h */
struct cpuinfo_x86 {
    /* ... */
    __u32   x86_capability[NCAPINTS + NBUGINTS];
    /* 機能用にNCAPINTSワード、バグフラグ用にNBUGINTSワード */
    /* ... */
};

/* ブートCPUのcpuinfo — ほとんどの機能チェックで使われる */
extern struct cpuinfo_x86 boot_cpu_data;
```

`NCAPINTS`は、機能ビットマスク配列内の32ビットワードの数である。各ビットは1つの`X86_FEATURE_*`定数に対応する。各ワードは、定められた順序でCPUIDの出力レジスタに対応付けられる。

```c
/* arch/x86/include/asm/cpufeatures.h */
/*
 * ワード 0: CPUIDリーフ1のEDX
 * ワード 1: CPUIDリーフ1のECX
 * ワード 2: CPUIDリーフ0x80000001のEDX
 * ワード 3: Linuxが定義する合成的な機能（単一のCPUIDビットを持たない）
 * ワード 4: CPUIDリーフ0x80000001のECX
 * ワード 5: CPUIDリーフ1のEBX（キャッシュ情報）
 * ワード 6: CPUIDリーフ7、サブリーフ0のEBX
 * ワード 7: CPUIDリーフ7、サブリーフ0のEDX
 * ワード 8: CPUIDリーフ0x80000007のEBX（APM）
 * ワード 9: CPUIDリーフ0x80000008のEBX
 * ワード10: CPUIDリーフ7、サブリーフ0のECX
 * ワード11: CPUIDリーフ7、サブリーフ1のEDX
 * ...
 */

#define X86_FEATURE_FPU         ( 0*32+ 0)  /* Onboard FPU */
#define X86_FEATURE_VME         ( 0*32+ 1)  /* Virtual Mode Extensions */
#define X86_FEATURE_DE          ( 0*32+ 2)  /* Debugging Extensions */
#define X86_FEATURE_PSE         ( 0*32+ 3)  /* Page Size Extensions */
#define X86_FEATURE_TSC         ( 0*32+ 4)  /* Time Stamp Counter */
#define X86_FEATURE_MSR         ( 0*32+ 5)  /* Model-Specific Registers */
#define X86_FEATURE_PAE         ( 0*32+ 6)  /* Physical Address Extensions */
/* ... ほかにも多数 ... */
#define X86_FEATURE_FSGSBASE    ( 6*32+ 0)  /* RDFSBASE, WRFSBASE, RDGSBASE, WRGSBASE */
#define X86_FEATURE_PCID        ( 1*32+17)  /* Process Context IDs */
#define X86_FEATURE_INVPCID     ( 6*32+10)  /* INVPCID instruction */
#define X86_FEATURE_PKU         (16*32+ 3)  /* Protection Keys for Userspace */
#define X86_FEATURE_UMIP        (16*32+ 2)  /* User-Mode Instruction Prevention */
#define X86_FEATURE_AVX512F     ( 9*32+16)  /* AVX-512 Foundation */
```

---

## CPU機能をチェックする3つの方法 {#three-ways-to-check-cpu-features}

カーネルは機能をチェックするために、性能特性の異なる3つのマクロを提供している。

### 1. cpu_has(): 実行時チェック {#1-cpu_has-runtime-check}

```c
/* arch/x86/include/asm/cpufeature.h */
#define cpu_has(c, bit) test_cpu_cap(c, bit)

/* 例: 特定のCPU構造体がPCIDを持つかチェックする */
if (cpu_has(&boot_cpu_data, X86_FEATURE_PCID)) {
    /* ... */
}
```

`cpu_has()`は実行時チェックである — `x86_capability`配列から読み取る。（ブートCPUだけでなく）特定のCPUをチェックする必要がある場合や、alternativeがパッチされる前のブート初期段階では、これを使う。

### 2. boot_cpu_has(): ブート時チェック {#2-bootcpuhas-boot-time-check}

```c
#define boot_cpu_has(bit) cpu_has(&boot_cpu_data, bit)

/* 例: サポートされていればPCIDを有効にする */
if (boot_cpu_has(X86_FEATURE_PCID))
    cr4_set_bits(X86_CR4_PCIDE);
```

`boot_cpu_has()`はブートCPUの機能フラグをチェックする。ブート中の一度きりのセットアップ判断に使われる。これらの呼び出しはalternative機構によってパッチされない。

### 3. static_cpu_has(): コンパイル時に用意されパッチされる {#3-staticcpuhas-compile-time-patched}

```c
#define static_cpu_has(bit) __static_cpu_has(bit)

/* 例: FSGSBASEの高速パス */
if (static_cpu_has(X86_FEATURE_FSGSBASE)) {
    /* FSGSBASEを持たないCPUではこの分岐は完全に除去されることがあり、
     * 常に持つCPUでは無条件ジャンプに変わることがある */
    wrfsbase(value);
} else {
    wrmsrl(MSR_FS_BASE, value);
}
```

`static_cpu_has()`は**パッチ可能な**命令列を出力する。ブート時に`apply_alternatives()`がそのコードをその場でパッチする。

- 機能が**存在する**場合: テストは`jmp over_else`に置き換えられる（常にif側の分岐を取る）
- 機能が**存在しない**場合: テストは`jmp over_if`に置き換えられる（常にelse側の分岐を取る）

パッチ後、このチェックは単一の無条件分岐になる — メモリ読み取りと比べてオーバーヘッドはゼロである。

---

## alternativeパッチング {#alternative-patching}

**alternative**機構により、カーネルは安全で保守的なコードを同梱しておき、実際のCPUの能力に基づいて、ブート時に最適化された命令列をパッチで適用できる。

### ALTERNATIVEマクロ {#the-alternative-macro}

```c
/* arch/x86/include/asm/alternative.h */

/*
 * ALTERNATIVE(orig_insn, alt_insn, feature):
 *   boot_cpu_dataに'feature'が存在する場合: alt_insnを使う
 *   それ以外: orig_insnを使う
 */
#define ALTERNATIVE(oldinstr, newinstr, feature)            \
    OLDINSTR(oldinstr)                                       \
    ".pushsection .altinstructions,\"a\"\n"                  \
    ALTINSTR_ENTRY(feature, 0)                               \
    ".popsection\n"                                          \
    ".pushsection .altinstr_replacement,\"ax\"\n"            \
    ALTINSTR_REPLACEMENT(newinstr, 0)                        \
    ".popsection\n"
```

デフォルトの命令（`orig_insn`）は`.text`セクションにインラインで出力される。置き換え用の命令（`alt_insn`）は`.altinstr_replacement`に置かれる。パッチ適用を記述するディスクリプタ（`struct alt_instr`）は`.altinstructions`に置かれる。

### struct alt_instr {#struct-alt_instr}

```c
/* arch/x86/include/asm/alternative.h */
struct alt_instr {
    s32  instr_offset;    /* 元の命令のオフセット */
    s32  repl_offset;     /* 置き換え命令のオフセット */
    u16  cpuid;           /* CPUIDの機能ビット */
    u8   instrlen;        /* 元の命令の長さ */
    u8   replacementlen;  /* 置き換え命令の長さ */
};
```

### ブート時のapply_alternatives() {#apply_alternatives-at-boot}

```c
/* arch/x86/kernel/alternative.c */
void __init_or_module apply_alternatives(struct alt_instr *start,
                                          struct alt_instr *end)
{
    struct alt_instr *a;

    for (a = start; a < end; a++) {
        u8 *instr  = (u8 *)&a->instr_offset  + a->instr_offset;
        u8 *repl   = (u8 *)&a->repl_offset   + a->repl_offset;

        if (!boot_cpu_has(a->cpuid))
            continue;   /* 機能が存在しないので元のままにする */

        /* パッチ: 置き換え用のバイト列を元の命令の上にコピーする。
         * 置き換えの方が短ければNOPで埋める。 */
        text_poke_early(instr, repl, a->replacementlen);
        /* ... */
    }
}
```

`apply_alternatives()`は`start_kernel()`の中で`alternative_instructions()`を介して呼ばれる。これは`.altinstructions`セクションを走査し、`text_poke_early()`（一時的に`.text`を書き込み可能にする）を使って各命令をその場でパッチする。

### alternativeの実際の使用例 {#real-examples-of-alternative-use}

```c
/* LOCKプレフィックス: SMPでは存在し、ユニプロセッサ（UP）カーネルではNOPにパッチされる */
ALTERNATIVE("lock; ", "nop", X86_FEATURE_UP)

/* 高速な文字列操作（ERMS: Enhanced REP MOVSB/STOSB） */
ALTERNATIVE("rep movsq", "rep movsb", X86_FEATURE_ERMS)

/* メモリバリア: LFENCEかMFENCEか何もしないか（Spectre対策） */
ALTERNATIVE("", "lfence", X86_FEATURE_LFENCE_RDTSC)
```

---

## CPUのバグフラグ: x86_bugs[] {#cpu-bug-flags-x86_bugs}

機能フラグに加えて、カーネルは`x86_capability[]`の上位`NBUGINTS`ワードに一連の**バグフラグ**を保持している。これらは、CPUが緩和策を必要とすることを示すために、初期のCPU検出中に設定される。

```c
/* arch/x86/include/asm/cpufeatures.h */
/*
 * バグフラグ — x86_capability[]の上位NBUGINTSワードに格納される。
 * マクロX86_BUG(x)は、バグフラグをすべての機能ワードの上に配置する:
 *   #define X86_BUG(x)  (NCAPINTS*32 + (x))
 * したがってX86_BUG_CPU_MELTDOWN = X86_BUG(14) = NCAPINTS*32 + 14。
 */
#define X86_BUG_F00F            X86_BUG( 0)  /* IntelのF0 0Fバグ */
#define X86_BUG_FDIV            X86_BUG( 1)  /* FPUのFDIV */
#define X86_BUG_AMD_APM_ERRATUM X86_BUG( 2)  /* AMDのAPMバグ */
#define X86_BUG_CPU_MELTDOWN    X86_BUG(14)  /* CPUがMeltdownに対して脆弱 */
#define X86_BUG_SPECTRE_V1      X86_BUG(15)  /* CPUがSpectre v1に対して脆弱 */
#define X86_BUG_SPECTRE_V2      X86_BUG(16)  /* CPUがSpectre v2に対して脆弱 */
#define X86_BUG_SPEC_STORE_BYPASS X86_BUG(17) /* Speculative store bypass */
#define X86_BUG_L1TF            X86_BUG(18)  /* L1 Terminal Fault */
#define X86_BUG_MDS             X86_BUG(19)  /* Microarchitectural Data Sampling */
#define X86_BUG_MSBDS_ONLY      X86_BUG(20)
#define X86_BUG_SWAPGS          X86_BUG(21)
#define X86_BUG_TAA             X86_BUG(22)  /* TSX Async Abort */
#define X86_BUG_ITLB_MULTIHIT   X86_BUG(23)
#define X86_BUG_SRBDS           X86_BUG(24)
#define X86_BUG_MMIO_STALE_DATA X86_BUG(25)
/* ... ハードウェアの脆弱性が新たに見つかるたびに追加される ... */
```

バグフラグは`arch/x86/kernel/cpu/bugs.c`の`check_bugs()`の中で設定される。

```c
/* arch/x86/kernel/cpu/bugs.c */
void __init check_bugs(void)
{
    identify_boot_cpu();

    /*
     * identify_boot_cpu()がx86_capability[]を設定した。
     * 次に、CPUのfamily/model/steppingに基づいてバグフラグを設定する。
     */
    if (cpu_matches(cpu_vuln_whitelist, NO_MELTDOWN))
        ; /* 脆弱ではない */
    else
        setup_force_cpu_bug(X86_BUG_CPU_MELTDOWN);

    /* ... 各脆弱性についてチェックする ... */
    spectre_v2_select_mitigation();
    spectre_v1_select_mitigation();
    mds_select_mitigation();
    /* ... */
}
```

---

## カーネルにおける主要なCPU機能 {#key-cpu-features-in-the-kernel}

### X86_FEATURE_FSGSBASE {#x86featurefsgsbase}

`RDFSBASE`、`WRFSBASE`、`RDGSBASE`、`WRGSBASE`命令を追加する — MSRインターフェースを経由せずに、FSおよびGSセグメントのベースレジスタを高速に読み書きできる。

- FSGSBASEがない場合: FS.baseの設定には`wrmsr(MSR_FS_BASE, value)`が必要 — これはシリアライズ命令である
- FSGSBASEがある場合: `wrfsbase(value)` — はるかに高速。ユーザー空間のTLSや、glibcのスレッドローカルストレージの実装に使われる

カーネルは、CPUがサポートしていればFSGSBASEを有効にする（`CR4.FSGSBASE`をセットする）。このサポートは**Linux 5.9**で追加された。

### X86_FEATURE_INVPCID {#x86featureinvpcid}

`INVPCID`（Invalidate Process-Context Identifier）命令: PCIDと仮想アドレスを指定してTLBエントリを選択的に無効化する、または特定のPCIDのグローバルでないエントリをすべてフラッシュする。

INVPCIDがない場合、PCIDのフラッシュにはCR3への書き込みが必要になる（そのPCIDのTLBエントリがすべてフラッシュされる）。INVPCIDの方が精密で高速である。

初期のHaswell CPUの一部はPCIDをサポートするがINVPCIDはサポートしない — カーネルはこれを検出し、CR3ベースのフラッシュにフォールバックする。

### X86_FEATURE_PKU {#x86featurepku}

**Protection Keys for Userspace（PKU）**: ユーザー空間のコードがメモリ領域に4ビットの「保護キー」を付けられるようにする。`PKRU`レジスタは、システムコールやページテーブルの変更を必要とせずに、16通りあるキーそれぞれのアクセス権（読み取り/書き込みの禁止）を制御する。

プロセス内の機密データ（例: OpenSSLの秘密鍵）を隔離するのに役立つ。カーネルのサポートは**Linux 4.6**で追加された。

### X86_FEATURE_UMIP {#x86featureumip}

**User-Mode Instruction Prevention**: `CR4.UMIP = 1`のとき、ring 3で`SGDT`、`SIDT`、`SLDT`、`SMSW`、`STR`を実行すると、ハードウェアのディスクリプタテーブルのアドレスを返す代わりに#GPが発生する。

ユーザープロセスがIDT、GDT、LDTのアドレスを知ることを防ぐ — ASLRの回避を阻止するのに役立つ。Linuxは利用可能であればUMIPを自動的に有効にする。

### X86_FEATURE_RDRAND {#x86featurerdrand}

`RDRAND`命令: 暗号学的に安全な乱数値を返すハードウェア乱数生成器である。カーネルはこれを`/dev/random`などのエントロピー源として使う。注意: カーネルでは`RDRAND`単独では唯一のエントロピー源として信頼されない（一部のIvy Bridgeシステムで、Intelの`RDRAND`実装に定数値を返すバグが見つかった）。

---

## CPU機能を観察する {#observing-cpu-features}

```bash
# カーネルが検出した機能フラグ（CPUIDの出力と同じものをフィルタしたもの）
cat /proc/cpuinfo | grep flags | head -1 | tr ' ' '\n' | sort | grep -E "pcid|invpcid|fsgsbase|pku|umip"

# すべての脆弱性緩和策
grep . /sys/devices/system/cpu/vulnerabilities/*

# CPUIDの生の出力（ユーザー空間のツール）
# インストール: apt-get install cpuid / dnf install cpuid
cpuid -1        # CPU 0の全リーフ
cpuid -l 7      # 拡張機能のリーフ7

# alternativeがパッチされたことを確認する
# （CONFIG_X86_PTDUMPか、/proc/kcoreの慎重な利用が必要）
# /proc/cpuinfoのflagsに機能が現れていれば、その機能フラグが設定されていることが確認できる

# alternativeパッチングのサイズ統計（CONFIG_X86_ALT_PATCHING_STATSならdmesgに出る）
dmesg | grep -i alternat

# 特定のMSRを確認する
modprobe msr
rdmsr 0xC0000082    # LSTAR
rdmsr 0x48b         # IA32_CORE_CAPABILITIES

# どのCPUがERMS（Enhanced REP MOVSB/STOSB）を持つか確認する
grep erms /proc/cpuinfo

# PCIDのサポートと使用状況を確認する
dmesg | grep -i pcid
```

---

## バージョンに関する注記 {#version-notes}

| 機能 | Linuxバージョン | 備考 |
|---------|--------------|-------|
| `static_cpu_has()` | 2.6.39 | コンパイル時に用意されるパッチ可能な機能チェック |
| `ALTERNATIVE`マクロ | 初期から。3.x頃に整備 | アセンブリレベルのパッチング |
| PKUのサポート | 4.6 | `CONFIG_X86_INTEL_MEMORY_PROTECTION_KEYS` |
| UMIPの強制 | 4.19 | サポートされていれば自動的に有効化 |
| カーネルでのFSGSBASE命令 | 5.9 | `CR4.FSGSBASE`を有効化 |
| 投機的実行向けの`X86_BUG_*`フラグ | 4.15 | Meltdown/Spectreの時代 |

---

## 参考文献 {#further-reading}

### カーネルソースとドキュメント {#kernel-source-documentation}

- [arch/x86/include/asm/cpufeatures.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/cpufeatures.h) — 機能ビットの全語彙（`X86_FEATURE_*`）
- [arch/x86/kernel/cpu/common.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/cpu/common.c) — `get_cpu_cap()`とブート時の検出
- [arch/x86/kernel/alternative.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/alternative.c) — `ALTERNATIVE`の背後にある実行時パッチングエンジン
- [Intel SDM](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) — Vol. 2AにすべてのCPUIDリーフが記載されている

### 関連ページ {#related-pages}

- [ARM64のCPU機能](../arm64/cpu-features.md) — CPUIDに相当するシステムレジスタと、arm64のalternativeエンジン
- [カーネルのハードニング](../../security/kernel-hardening.md) — これらの機能/バグビットによって選択される緩和策
