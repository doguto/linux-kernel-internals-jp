# ARM64のCPU機能 {#arm64-cpu-features}

> 機能検出、alternativesによるパッチ適用、hwcaps、エラッタの回避策

## 概要 {#overview}

ARM64（AArch64）は、実行時にCPUのケーパビリティを検出し、それを活用するためにカーネルイメージを透過的にパッチする、充実した仕組みを実装している。関わる層は3つある。

1. **機能検出** — ブート時にIDレジスタを読み、全CPUにわたる安全な共通部分としてケーパビリティを算出する。
2. **alternativesによるパッチ適用** — ある機能の存在が確認されると、カーネルtext内のホットパスがその場で書き換えられ、より高速またはより正しい命令を使うようになる。
3. **ユーザー空間への公開** — 検出された機能は、`getauxval(AT_HWCAP)`と`/proc/cpuinfo`の`Features:`行を介して`HWCAP_*`ビットとして公開される。

カーネルのコードは主に次の場所にある。

| パス | 目的 |
|------|---------|
| `arch/arm64/kernel/cpufeature.c` | 機能検出、ケーパビリティの算出 |
| `arch/arm64/include/asm/cpufeature.h` | `struct arm64_cpu_capabilities`、マクロ |
| `arch/arm64/kernel/alternative.c` | alternativesのパッチ適用エンジン |
| `arch/arm64/include/asm/alternative.h` | `alternative()`、`alternative_insn()`マクロ |
| `arch/arm64/kernel/cpu_errata.c` | エラッタ回避策のテーブル |
| `arch/arm64/include/uapi/asm/hwcap.h` | ユーザー空間に公開される`HWCAP_*`定数 |

---

## ブート時の機能検出 {#feature-detection-at-boot}

### IDレジスタ {#id-registers}

ARM64は、EL1で読み取り可能な一群のシステムレジスタを通じてCPUのケーパビリティを公開する。カーネルはこれらを`setup_cpu_features()`（`setup_arch()`から呼ばれる）の中で読む。

| レジスタ | 記述する内容 |
|----------|------------------|
| `ID_AA64ISAR0_EL1` | 命令セット: AES、SHA、CRC32、アトミック、RDM、… |
| `ID_AA64ISAR1_EL1` | DPB、JSCVT、FCMA、LRCPC、GPA、GPI、FRINTTS、… |
| `ID_AA64ISAR2_EL1` | WFXT、RPRES、GPA3、APA3、MOPS、BC、… |
| `ID_AA64MMFR0_EL1` | PAサイズ、ASIDのビット数、TGranサイズ |
| `ID_AA64MMFR1_EL1` | VH、HPDS、LO、PAN、SpecSEI、XNX |
| `ID_AA64MMFR2_EL1` | CNP、UAO、LSM、IESB、VARange、CCIDX |
| `ID_AA64PFR0_EL1` | EL0/1/2/3の幅、FP、AdvSIMD、GIC、RAS、SVE |
| `ID_AA64PFR1_EL1` | BT（BTI）、SSBS、MTE、RAS_frac、MPAM_frac |
| `MIDR_EL1` | 実装者、アーキテクチャ、バリアント、パーツ、リビジョン |

各レジスタは複数の4ビットフィールドをエンコードしている。ヘルパー`cpuid_feature_extract_field()`（`arch/arm64/include/asm/cpufeature.h`で定義）が単一のフィールドを取り出す。

```c
/* arch/arm64/include/asm/cpufeature.h */
static inline int cpuid_feature_extract_field(u64 features, int field, bool sign)
{
    return (sign)
        ? cpuid_feature_extract_signed_field(features, field)
        : (int)((features >> field) & 0xf);
}
```

`MIDR_EL1`は、エラッタの照合に使われるCPUの識別情報をエンコードしている。

```
MIDR_EL1 layout
 [31:24] Implementer  (0x41 = ARM Ltd, 0x51 = Qualcomm, 0x53 = Samsung, …)
 [23:20] Variant      (major revision)
 [19:16] Architecture (always 0xF for ARMv8+)
 [15:4]  PartNum      (0xD03 = Cortex-A53, 0xD0B = Cortex-A76, …)
 [3:0]   Revision     (minor revision)
```

### CPU間でのケーパビリティの集約 {#capability-accumulation-across-cpus}

SMPシステムでは、機能の共通部分を全CPUにわたって計算しなければならない。各セカンダリCPUは立ち上げ時に`check_local_cpu_capabilities()`を呼ぶ。プライマリCPUは`update_cpu_capabilities()`を介して最終的な安全な集合を決定する。

```
Boot CPU reads ID registers
       │
       ▼
update_cpu_capabilities(arm64_features)
       │   calls matches() on every arm64_cpu_capabilities entry
       ▼
set_cpu_cap() sets bit in cpu_hwcaps[] bitmap
       │
Secondary CPUs boot, call check_local_cpu_capabilities()
       │   verifies they do not lack any already-set cap
       ▼
apply_alternatives_all()  ← patches kernel text
       │
cpu_enable() hooks run per-CPU (e.g., enable SSBS, PAN)
```

グローバルな`cpu_hwcaps`は、`arch/arm64/include/asm/cpucaps.h`で定義された`ARM64_NCAPS`ビットの`DECLARE_BITMAP`である。

---

## struct arm64_cpu_capabilities {#struct-arm64_cpu_capabilities}

ARM64のすべての機能またはエラッタは、`struct arm64_cpu_capabilities`（`arch/arm64/include/asm/cpufeature.h`）の1エントリで記述される。

```c
struct arm64_cpu_capabilities {
    const char          *desc;
    u16                  capability;   /* ARM64_*のcapインデックス。例: ARM64_HAS_LSE_ATOMICS */
    u16                  type;         /* ARM64_CPUCAP_*フラグのビットマスク */
    bool                (*matches)(const struct arm64_cpu_capabilities *cap, int scope);
    void                (*cpu_enable)(const struct arm64_cpu_capabilities *cap);
    union {
        /* レジスタベースの機能用: */
        struct {
            u32         sys_reg;       /* SYS_ID_AA64ISAR0_EL1など */
            u8          field_pos;     /* フィールドのビット位置 */
            u8          field_width;   /* フィールド幅（通常4） */
            u8          min_field_value;
            u8          hwcap_type;
            unsigned long hwcap;       /* HWCAP_*またはHWCAP2_*のビット */
        };
        /* MIDRベースのエラッタ用: */
        const struct midr_range *midr_range_list;
        struct midr_range        midr_range;
    };
};
```

主な`type`フラグの組み合わせ:

| フラグ | 意味 |
|------|---------|
| `ARM64_CPUCAP_SYSTEM_FEATURE` | 全CPUが持つ場合にのみ安全に使える |
| `ARM64_CPUCAP_BOOT_CPU_FEATURE` | ブートCPUのみから検出する |
| `ARM64_CPUCAP_WEAK_LOCAL_CPU_FEATURE` | CPUごと。一部のCPUで欠けていても許容される |
| `ARM64_CPUCAP_SCOPE_LOCAL_CPU` | ホットプラグ時にCPUごとにチェックされる |

主な2つのグローバルテーブル:

```c
/* arch/arm64/kernel/cpufeature.c */
static const struct arm64_cpu_capabilities arm64_features[];
static const struct arm64_cpu_capabilities arm64_errata[];
```

`arm64_features[]`は`HWCAP_*`の公開とalternativesによるパッチ適用を駆動する。`arm64_errata[]`は回避策の適用を駆動する。

---

## 実行時のケーパビリティ確認 {#checking-capabilities-at-runtime}

### システム全体: cpus_have_cap() {#system-wide-cpus_have_cap}

```c
/* arch/arm64/include/asm/cpufeature.h */
static inline bool cpus_have_cap(unsigned int num)
{
    return test_bit(num, cpu_hwcaps);
}
```

これはグローバルな`cpu_hwcaps`ビットマップから1ビットを読む。`setup_cpu_features()`の完了後であれば、どのコンテキストから呼んでも安全である。典型的な使い方:

```c
if (cpus_have_cap(ARM64_HAS_LSE_ATOMICS))
    use_lse_path();
```

### CPUごと: this_cpu_has_cap() {#per-cpu-this_cpu_has_cap}

```c
static inline bool this_cpu_has_cap(unsigned int cap)
{
    if (!WARN_ON(cap >= ARM64_NCAPS))
        return !!test_bit(cap, (unsigned long *)this_cpu_ptr(&cpu_hwcaps));
    return false;
}
```

機能が全CPUで一様でない可能性がある場合（例: SSBS、FPMR）に使われる。

### static key: cpus_have_const_cap() {#static-keys-cpus_have_const_cap}

極めてホットなパス向けに、ケーパビリティは`static_key_false`エントリでも裏付けられており、チェックはブート時に分岐へパッチされる単一のNOPにコンパイルされる。

```c
/* 高速パス — ビットマップの読み込みなし、static branchのみ */
if (cpus_have_const_cap(ARM64_HAS_LSE_ATOMICS)) { … }
```

---

## alternativesによるパッチ適用 {#alternatives-patching}

### 仕組み {#how-it-works}

alternativesの仕組みにより、カーネルはすべてのARMv8 CPUで正しく動作する1つのバイナリを出荷し、ブート時に実際のハードウェア向けに最適化できる。その方式は次のとおりである。

1. コンパイラが`.altinstructions` ELFセクションに**パッチサイト**を出力する: `{orig_offset, alt_offset, cpucap, orig_len, alt_len}`。
2. 元の（保守的な）命令シーケンスが実行されるtextに置かれる。
3. ブート時に`apply_alternatives_all()`が`.altinstructions`を走査し、`cpucap`ビットが`cpu_hwcaps`でセットされている各エントリについて、`aarch64_insn_patch_text()`を使って実行中のtextを代替シーケンスで上書きする。
4. キャッシュメンテナンス（`__flush_icache_range()`）によって、新しい命令が命令ストリームから見えるようになる。

### アセンブリマクロ {#assembly-macros}

```asm
/* arch/arm64/include/asm/alternative.h */

/*
 * 'cap'が存在する場合、命令シーケンス'oldinstr'を'newinstr'で置き換える。
 * 両シーケンスのバイト長は同じでなければならない。
 */
.macro alternative_insn oldinstr, newinstr, cap, enable = 1
    .if \enable
661:    \oldinstr
662:    .pushsection .altinstructions, "a"
        altinstruction_entry 661b, 663f, \cap, 662b-661b, 664f-663f
    .popsection
    .pushsection .altinstr_replacement, "ax"
663:    \newinstr
664:    .popsection
    .endif
.endm
```

Cコードでは、`alternative()`マクロがインラインアセンブリをラップする。

```c
/* インラインasm形式 */
asm volatile(ALTERNATIVE("nop", "isb", ARM64_WORKAROUND_1542419));
```

### 例: LSEアトミック（LDXR/STXR → CAS） {#example-lse-atomics-ldxrstxr-cas}

パッチ適用前、ARM64の`atomic_add()`はLL/SCループを含んでいる。

```asm
/* 元のコード（常に正しい） */
661:
    ldxr    w0, [x1]
    add     w0, w0, w2
    stxr    w3, w0, [x1]
    cbnz    w3, 661b
```

`ARM64_HAS_LSE_ATOMICS`がセットされていると、alternativesはこれを次のように置き換える。

```asm
/* パッチ後（LSE） */
    casa    w2, w0, [x1]
```

ケーパビリティのインデックス`ARM64_HAS_LSE_ATOMICS`は`arch/arm64/include/asm/cpucaps.h`で定義されており、`arm64_features[]`にあるその検出エントリは`ID_AA64ISAR0_EL1.Atomic` >= 2をチェックする。

### apply_alternatives_all() {#apply_alternatives_all}

```c
/* arch/arm64/kernel/alternative.c */
void __init apply_alternatives_all(void)
{
    struct alt_region region = {
        .begin  = (struct alt_instr *)__alt_instructions,
        .end    = (struct alt_instr *)__alt_instructions_end,
    };
    /*
     * cpucapsは確定済み。すべてのaltinstructionエントリを走査し、
     * 対応するcapビットがセットされていればパッチする。
     */
    BUG_ON(!system_capabilities_finalized());
    __apply_alternatives(&region, false, &cpu_hwcaps_ptrs);
}
```

セカンダリCPUも`apply_alternatives_this_cpu()`を呼び、自身のコンテキストで`ARM64_CPUCAP_LOCAL_CPU_FEATURE`のパッチを適用する。

---

## HWCAP: ユーザー空間への機能の公開 {#hwcap-userspace-feature-exposure}

検出された機能は、2つの仕組みを通じてユーザー空間に通知される。

- **`getauxval(AT_HWCAP)`**と**`getauxval(AT_HWCAP2)`** — `execve()`時にカーネルがELF補助ベクタに置くビットマスク値。
- **`/proc/cpuinfo`** — `Features:`行が同じケーパビリティを人間が読める文字列で列挙する。

### HWCAP定数 {#hwcap-constants}

`arch/arm64/include/uapi/asm/hwcap.h`で定義されている。

| 定数 | ビット | 機能 |
|----------|-----|---------|
| `HWCAP_FP` | 0 | 浮動小数点（ARMv8では必須） |
| `HWCAP_ASIMD` | 1 | Advanced SIMD / Neon（必須） |
| `HWCAP_EVTSTRM` | 2 | イベントストリーム（汎用タイマー） |
| `HWCAP_AES` | 3 | AES命令 |
| `HWCAP_PMULL` | 4 | 多項式乗算（PMULL/PMULL2） |
| `HWCAP_SHA1` | 5 | SHA-1命令 |
| `HWCAP_SHA2` | 6 | SHA-256命令 |
| `HWCAP_CRC32` | 7 | CRC32命令 |
| `HWCAP_ATOMICS` | 8 | Large System Extensions（LSEアトミック） |
| `HWCAP_FPHP` | 9 | 半精度FP |
| `HWCAP_ASIMDHP` | 10 | Advanced SIMDの半精度 |
| `HWCAP_CPUID` | 11 | EL0からのIDレジスタアクセス |
| `HWCAP_ASIMDRDM` | 12 | 丸め付き倍精度乗算累算 |
| `HWCAP_JSCVT` | 13 | JavaScript用FJCVTZS命令 |
| `HWCAP_FCMA` | 14 | 浮動小数点の複素数乗算 |
| `HWCAP_LRCPC` | 15 | Load-acquire RCpc |
| `HWCAP_DCPOP` | 16 | DC CVAP命令 |
| `HWCAP_SHA3` | 17 | SHA-3命令 |
| `HWCAP_SM3` | 18 | SM3命令 |
| `HWCAP_SM4` | 19 | SM4命令 |
| `HWCAP_ASIMDDP` | 20 | SIMDのドット積 |
| `HWCAP_SHA512` | 21 | SHA-512命令 |
| `HWCAP_SVE` | 22 | Scalable Vector Extension |
| `HWCAP_ASIMDFHM` | 23 | SIMDのFP16乗算累算 |
| `HWCAP_DIT` | 24 | Data Independent Timing |
| `HWCAP_USCAT` | 25 | アラインされていないシングルコピーアトミックアクセス |
| `HWCAP_ILRCPC` | 26 | LRCPC2（即値オフセット） |
| `HWCAP_FLAGM` | 27 | フラグ操作命令 |
| `HWCAP_SSBS` | 28 | Speculative Store Bypass Safe |
| `HWCAP_SB` | 29 | Speculation Barrier命令 |
| `HWCAP_PACA` | 30 | ポインタ認証（アドレス） |
| `HWCAP_PACG` | 31 | ポインタ認証（汎用） |

主な`AT_HWCAP2`定数（`hwcap.h`内で`HWCAP2_*`として定義）:

| 定数 | 機能 |
|----------|---------|
| `HWCAP2_DCPODP` | DC CVADP命令 |
| `HWCAP2_SVE2` | SVE2 |
| `HWCAP2_SVEAES` | SVE2 + AES |
| `HWCAP2_BTI` | Branch Target Identification |
| `HWCAP2_MTE` | Memory Tagging Extension |
| `HWCAP2_ECV` | Enhanced Counter Virtualization |
| `HWCAP2_AFP` | Alternate Floating-Point Behavior |
| `HWCAP2_RPRES` | 12ビットの逆数推定 |
| `HWCAP2_MTE3` | MTEの非対称フォールト報告 |
| `HWCAP2_SME` | Scalable Matrix Extension |

### ユーザー空間からHWCAPを読む {#reading-hwcaps-from-userspace}

```c
#include <sys/auxv.h>
#include <asm/hwcap.h>

unsigned long hwcap  = getauxval(AT_HWCAP);
unsigned long hwcap2 = getauxval(AT_HWCAP2);

if (hwcap & HWCAP_ATOMICS)
    /* LSEアトミックを安全に使える */;
if (hwcap & HWCAP_SVE)
    /* SVEが利用可能 */;
if (hwcap2 & HWCAP2_BTI)
    /* BTIが強制されている */;
```

---

## SVE — Scalable Vector Extension {#sve-scalable-vector-extension}

SVE（ARMv8.2以降）は、可変幅のベクタレジスタを導入する。幅は128ビット刻みで128から2048ビットである。NEONとは異なり、ベクタ長はISAの設計時に固定されておらず、実装定義であり、タスクごとに設定できる。

### ベクタ長の管理 {#vector-length-management}

```c
/* arch/arm64/kernel/fpsimd.c */

/* 現在のタスクのSVEベクタ長を設定する */
int sve_set_vector_length(struct task_struct *task, unsigned long vl,
                          unsigned long flags);
```

ユーザー空間は次のようにしてベクタ長を選択する。

```c
prctl(PR_SVE_SET_VL, vl);   /* 希望するVLをバイト単位で設定 */
prctl(PR_SVE_GET_VL);       /* 現在のVLを取得 */
```

カーネルは`vl`を、要求を超えない範囲でサポートされる最大の値に切り下げる。有効な値は16から256バイト（128から2048ビット）までの16の倍数である。システム全体のデフォルトVLは`/proc/sys/abi/sve_default_vector_length`から読み取れる（これは新しいスレッドに適用されるデフォルトであって上限ではない。真の最大値は、VLを任意の大きな値に設定した後に`prctl(PR_SVE_GET_VL)`で報告される）。

### 遅延状態保存 {#lazy-state-save}

SVEのレジスタ状態は遅延的に保存・復元される。

1. タスクが初めてSVE命令を実行すると、`trap_sve`例外が発生する（ほとんどのコンテキストで`CPACR_EL1.ZEN` = 0であるため）。
2. トラップハンドラはスレッドフラグに`TIF_SVE`をセットし、タスクごとのSVE保存領域（`task->thread.sve_state`）を割り当て、`CPACR_EL1.ZEN = 0b11`を設定してSVEを再度有効にする（EL0とEL1がトラップなしでSVEにアクセスできるようにする）。
3. コンテキストスイッチ時、`fpsimd_thread_switch()`は`TIF_SVE`がセットされている場合にのみSVE状態を保存し、SVEを使わないタスクのオーバーヘッドを避ける。

```c
/* thread_infoのフラグ（arch/arm64/include/asm/thread_info.h） */
#define TIF_SVE         23  /* EL0でSVEが有効 */
#define TIF_SVE_VL_INHERIT 24 /* exec後もSVEのVLを継承 */
```

### カーネルでのSVEの使用 {#kernel-sve-use}

SVEを使うカーネルコード（例: 高速化された暗号処理）は、FP/SVEアクセストラップを明示的に有効にしなければならない。

```c
kernel_neon_begin();   /* プリエンプションを無効化し、FPSIMD/SVEを有効化 */
/* NEONまたはSVE命令を使う */
kernel_neon_end();     /* 状態を復元し、プリエンプションを再度有効化 */
```

---

## BTI — Branch Target Identification {#bti-branch-target-identification}

BTIはARMv8.5の制御フロー完全性（CFI）機能である。ユーザー空間（`SCTLR_EL1.BT0 = 1`）またはカーネル（`SCTLR_EL1.BT1 = 1`）で有効にされると、あらゆる間接分岐（`BR`、`BLR`、`RET`）は`BTI`命令（または呼び出し先を意味する対の`BL`/`BLRAAZ`の飛び先）に着地しなければならない。それ以外の場所に着地するとBranch Target例外が発生する。カーネルはユーザープロセスにBTIを強制するために`BT0`をセットし、`BT1`はカーネルコードでの強制を制御する。

### ELFのマーキング {#elf-marking}

バイナリはGNU propertyノートによってオプトインする。

```
GNU_PROPERTY_AARCH64_FEATURE_1_BTI   (bit 0 of GNU_PROPERTY_AARCH64_FEATURE_1_AND)
```

動的リンカ（`ld.so`）はこのノートを読む。カーネルは`execve()`時に`arch/arm64/kernel/process.c`の`arch_parse_elf_property()`を介してこれをチェックし、プロパティが存在すればそのプロセスに対して`SCTLR_EL1.BT0`をセットして、ユーザー空間のコードに対するBTIの強制を有効にする。

### JITおよびシグナルハンドラとの相互作用 {#interaction-with-jit-and-signal-handlers}

JITコンパイラは`BTI c`（呼び出し先）または`BTI j`（ジャンプ先）のランディングパッドを出力しなければならない。カーネルのシグナルリターン用トランポリン（`arch/arm64/kernel/vdso/sigreturn.S`）は、ユーザーのトランポリンから`BLR`で入れるようBTIアノテーションが付けられている。

---

## MTE — Memory Tagging Extension {#mte-memory-tagging-extension}

MTE（ARMv8.5以降）は、タグ付きメモリの16バイトの**グラニュール**ごとに4ビットの**割り当てタグ**を関連付け、対応する**論理タグ**をポインタのビット[59:56]（top-byte ignore領域）にエンコードすることで、ハードウェア支援によるメモリ安全性を実現する。

ロードやストアのたびに、ハードウェアはポインタの論理タグとメモリグラニュールの割り当てタグを比較する。不一致は、選択されたTCR_EL1のモードに応じて、同期フォールトを発生させるか、非同期に記録される。

### カーネルAPI {#kernel-api}

```c
/* 呼び出し元プロセスでMTEを有効化 */
prctl(PR_SET_TAGGED_ADDR_CTRL,
      PR_TAGGED_ADDR_ENABLE | PR_MTE_TCF_SYNC | (0xffff << PR_MTE_TAG_SHIFT),
      0, 0, 0);

/* PROT_MTE付きでmmapし、タグ付け可能なマッピングを得る */
void *p = mmap(NULL, size, PROT_READ | PROT_WRITE | PROT_MTE,
               MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
```

フラグ`PROT_MTE`は`arch/arm64/include/uapi/asm/mman.h`にある。

### モード {#modes}

| モード | SCTLR_EL1.TCF0 | 挙動 |
|------|---------------|---------|
| `PR_MTE_TCF_NONE` | `0b00` | MTE無効 |
| `PR_MTE_TCF_SYNC` | `0b01` | タグ不一致で同期フォールト |
| `PR_MTE_TCF_ASYNC` | `0b10` | 非同期フォールト。SIGBUSで報告される |

COWページについては、ページが実際に書き込まれるまでMTEタグは`fork()`をまたいで保持されない。カーネルは`copy_user_highpage()`でタグの継承を処理する。

---

## エラッタの回避策 {#errata-workarounds}

CPUのエラッタ（ハードウェアのバグ）は同じ`arm64_cpu_capabilities`基盤を通じて扱われるが、`arm64_errata[]`（`arch/arm64/kernel/cpu_errata.c`）に格納される。

### MIDRベースの照合 {#midr-based-matching}

各エラッタのエントリは、`MIDR_CPU_VAR_REV()`または`MIDR_ALL_VERSIONS()`を使って影響を受けるCPUのリビジョンを照合する。

```c
/* arch/arm64/kernel/cpu_errata.c */
{
    .desc       = "Cortex-A53: 843419: A load or store might access "
                  "an incorrect address",
    .capability = ARM64_WORKAROUND_843419,
    .type       = ARM64_CPUCAP_LOCAL_CPU_ERRATUM,
    ERRATA_MIDR_REV_RANGE(MIDR_CORTEX_A53, 0, 0, 4),
},
```

`MIDR_CORTEX_A53`は`0x410FD030`である。範囲`(variant=0, rev_min=0, rev_max=4)`はr0p0からr0p4までに一致する。

### 回避策の仕組み {#workaround-mechanisms}

| 仕組み | 使われる場面 |
|-----------|----------|
| alternativesによるパッチ | 実行時: 命令シーケンスを置き換える |
| `cpu_enable`フック | 実行時: システムレジスタのフラグをセットする |
| Kconfigオプション | コンパイル時: バリアを無条件に挿入する |
| リンカフラグ | ビルド時: 例: `--fix-cortex-a53-843419` |

### 主なエラッタ {#selected-errata}

| エラッタ | CPU | 説明 | 回避策 |
|---------|-----|-------------|-----------|
| 843419 | Cortex-A53 r0p0–r0p4 | ADRPシーケンスで誤ったアドレスが使われる | リンカの`--fix-cortex-a53-843419` |
| 835769 | Cortex-A53 r0p0–r0p4 | ロード後のMUL/MADDが誤った結果を返す | コンパイラの`-mfix-cortex-a53-835769` |
| 1530923 | Cortex-A55 | 投機的なAT命令がフォールトを引き起こすことがある | 投機バリアのalternatives |
| 2457168 | Cortex-A510 | PMULL2が誤った結果を生むことがある | alternativesによるパッチ |
| 1418040 | Cortex-A55 | 命令キャッシュの無効化が不完全になることがある | `ic iallu`のalternatives |

エラッタの回避策は`CONFIG_ARM64_ERRATUM_*` Kconfigシンボルによって条件付きでコンパイルされ、影響を受けないハードウェアではオーバーヘッドを追加しない。

---

## 可観測性 {#observability}

### /proc/cpuinfo {#proccpuinfo}

```bash
$ cat /proc/cpuinfo | grep -E "CPU implementer|CPU architecture|CPU variant|CPU part|CPU revision|Features"
processor       : 0
CPU implementer : 0x41          # ARM Ltd
CPU architecture: 8
CPU variant     : 0x3
CPU part        : 0xd0b         # Cortex-A76
CPU revision    : 1

Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp \
                  asimdhp cpuid asimdrdm lrcpc dcpop asimddp ssbs
```

### sysfsのIDレジスタ {#sysfs-id-registers}

Linux 5.11以降は、生のIDレジスタ値をsysfs以下に公開している（`CONFIG_ARM64_CPUIDLE`ビルド設定と`ID_AA64DFR0_EL1.PMSVer`ケーパビリティのチェックによって制限される）。

```bash
$ grep . /sys/devices/system/cpu/cpu0/regs/identification/*
/sys/devices/system/cpu/cpu0/regs/identification/midr_el1:0x413FD0B1
/sys/devices/system/cpu/cpu0/regs/identification/revidr_el1:0x00000000
```

### 有効なhwcapsをプログラムから確認する {#checking-active-hwcaps-programmatically}

```c
#include <sys/auxv.h>
#include <asm/hwcap.h>
#include <stdio.h>

int main(void) {
    unsigned long hwcap  = getauxval(AT_HWCAP);
    unsigned long hwcap2 = getauxval(AT_HWCAP2);

    printf("LSE atomics : %s\n", (hwcap  & HWCAP_ATOMICS) ? "yes" : "no");
    printf("SVE         : %s\n", (hwcap  & HWCAP_SVE)     ? "yes" : "no");
    printf("BTI         : %s\n", (hwcap2 & HWCAP2_BTI)    ? "yes" : "no");
    printf("MTE         : %s\n", (hwcap2 & HWCAP2_MTE)    ? "yes" : "no");
    return 0;
}
```

### カーネルのケーパビリティビットマップ（デバッグ） {#kernel-capability-bitmap-debug}

ケーパビリティビットマップ全体は直接エクスポートされていないが、`Features:`行と`/sys/devices/system/cpu/cpu*/regs/identification/midr_el1`の読み取りから有効な集合を推測できる。

カーネル開発者にとっては、`cpus_have_cap(ARM64_HAS_LSE_ATOMICS)`がカーネル内での正式なチェックである。ブートの初期段階（セカンダリCPUが立ち上がる前）では、代わりに`boot_cpu_has(ARM64_HAS_LSE_ATOMICS)`が使われる。

---

## ブート時の流れのまとめ {#boot-time-flow-summary}

```
start_kernel()
    └─ setup_arch()
          ├─ setup_machine_fdt()
          ├─ init_mem_init()
          └─ setup_cpu_features()
                ├─ init_cpu_features()        ← read ID regs on boot CPU
                ├─ setup_system_capabilities()
                │     └─ update_cpu_capabilities(arm64_features)
                │           └─ for each cap: cap->matches() → set_cpu_cap()
                ├─ apply_alternatives_all()   ← patch kernel text
                └─ (later) smp_cpus_done()
                      └─ check_local_cpu_capabilities()  per secondary CPU
```

---

## 参考文献 {#further-reading}

- `arch/arm64/include/asm/cpufeature.h` — `struct arm64_cpu_capabilities`の完全な定義と、すべての`ARM64_CPUCAP_*`型フラグ
- `arch/arm64/include/asm/cpucaps.h` — すべての`ARM64_*`ケーパビリティインデックスと`ARM64_NCAPS`の列挙
- [arch/arm64/kernel/cpufeature.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/kernel/cpufeature.c) — `arm64_features[]`テーブル、検出ロジック、`update_cpu_capabilities()`、`apply_alternatives_all()`の呼び出し箇所
- `arch/arm64/kernel/cpu_errata.c` — MIDRの完全な照合範囲と回避策の説明を含む`arm64_errata[]`テーブル
- `arch/arm64/kernel/alternative.c` — パッチ適用エンジン、`__apply_alternatives()`
- `arch/arm64/include/asm/alternative.h` — `alternative_insn`、`alternative`、`ALTERNATIVE`マクロの定義
- `arch/arm64/include/uapi/asm/hwcap.h` — ビット位置付きのすべての`HWCAP_*`および`HWCAP2_*`定数
- `arch/arm64/kernel/fpsimd.c` — SVEの状態管理、`kernel_neon_begin/end`、`TIF_SVE`の処理
- `arch/arm64/kernel/process.c` — `execve()`時のBTIの強制、新しいプロセスに対するMTEの初期化
- [ARM Architecture Reference Manual, ARMv8-A](https://developer.arm.com/documentation/ddi0487/latest) — すべてのシステムレジスタと機能エンコーディングの決定版リファレンス
- [ARM CPU Feature Registers](https://www.kernel.org/doc/html/latest/arch/arm64/cpu-feature-registers.html) — sysfsでのレジスタ公開と`HWCAP_CPUID`に関するカーネルドキュメント
- [ARM64 ELF ABI](https://github.com/ARM-software/abi-aa/blob/main/sysvabi64/sysvabi64.rst) — `AT_HWCAP`の割り当て、BTIのELFノート、SVE ABI
