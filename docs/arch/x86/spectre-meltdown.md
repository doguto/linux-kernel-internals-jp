# SpectreとMeltdown {#spectre-and-meltdown}

> ハードウェアの脆弱性、カーネルの緩和策、そして性能への影響

2018年1月、MeltdownとSpectreの協調開示は、ソフトウェアとハードウェアの関係を根本から変えた。これらの脆弱性は、CPUの性能向上のための機能 — 投機的実行と分岐予測 — を悪用し、特権やプロセスの境界を越えて情報を漏洩させる。Linuxカーネルの緩和策は、システムのほぼすべてのホットパスに手を入れている。

---

## 背景: 投機的実行とキャッシュサイドチャネル {#background-speculative-execution-and-cache-side-channels}

現代のCPUは命令を投機的に実行する — 分岐を予測し、確定したプログラムの流れより先に実行する。予測が誤っていれば、投機的に実行された結果は破棄される。問題は、**キャッシュへの副作用は破棄されない**ことである。

キャッシュサイドチャネルは次のように機能する。
1. プローブ配列をキャッシュからフラッシュする（`CLFLUSH`を使う）
2. 被害者にメモリへの投機的アクセスを行わせ、データをキャッシュにロードさせる
3. プローブ配列へのアクセス時間を測定し、どのキャッシュラインがロードされたかを特定する
4. タイミングの差から秘密のバイトを推測する

これは**Flush+Reload**攻撃や**Prime+Probe**攻撃と呼ばれる。主要な投機的実行の脆弱性は、いずれもこの手法の変種を使う。

---

## Meltdown（CVE-2017-5754） {#meltdown-cve-2017-5754}

### 脆弱性 {#the-vulnerability}

Meltdownは、投機的実行と権限チェックの間の競合を悪用する。脆弱なIntel CPU（および一部のARM）では、CPUはユーザーモードであってもカーネルアドレスからのメモリ読み取りを投機的に実行する — #PF（ページフォールト、ベクタ14）フォルトを発生させるはずの権限チェックが行われる**前に**。

```
Attack sequence:
1. User code executes: mov rax, [kernel_address]
2. CPU speculatively reads the kernel byte into rax
   (permission check not yet complete)
3. Attacker uses the value in rax as an index: mov rbx, [probe_array + rax*4096]
4. CPU raises a #PF (Page Fault, vector 14 — permission check completes) — speculatively loaded value is discarded
5. But the cache line for probe_array[secret_byte * 4096] is now warm
6. Attacker measures access times to probe_array — finds which byte was accessed
7. Secret kernel byte recovered
```

**影響を受けるハードウェア**: 2010年頃から2018年頃（Coffee Lake Refresh以前）までのほとんどのIntel CPU。AMDのCPUは一般に影響を受けない。ARM Cortex-A75は影響を受ける。

### 緩和策: KPTI {#mitigation-kpti}

KPTI（Kernel Page Table Isolation）は、CPUがユーザーコードを実行している間、カーネルアドレスがページテーブルにそもそも**マップされていない**ようにすることで攻撃を防ぐ。カーネルアドレスがページテーブルになければ、投機的な読み取りは（キャッシュされた値ではなく）TLBミスを生むだけで、サイドチャネルは成立しない。

KPTIの詳しい説明は[page-tables.md](page-tables.md)を参照。

```bash
cat /sys/devices/system/cpu/vulnerabilities/meltdown
# "Mitigation: PTI"         — KPTIが有効
# "Not affected"            — CPUは脆弱でない
```

---

## Spectre v1（CVE-2017-5753）: 境界チェックのバイパス {#spectre-v1-cve-2017-5753-bounds-check-bypass}

### 脆弱性 {#the-vulnerability_1}

Spectre v1は、境界チェックを越えて進む投機的実行を悪用する。攻撃者は分岐予測器を訓練して分岐が成立すると予測させ、その後チェックを失敗させる — しかし投機的実行はすでに範囲外のデータを読み出している。

```c
/* 脆弱なパターン */
if (index < array_size) {
    /* index >= array_sizeであっても、CPUはこれを投機的に実行する */
    value = array[index];           /* 投機的な範囲外読み取り */
    sink  = probe_array[value * 4096]; /* 秘密のバイトをキャッシュにロードする */
}
/* 分岐予測器が誤りに気づいてロールバックする — しかしキャッシュは温まっている */
```

攻撃者は、有効なインデックスでその関数を何度も呼び出して分岐予測器を訓練し、その後悪意のあるインデックスで一度だけ呼び出す。CPUは「分岐成立」と予測し、範囲外の値を投機的に読み取る。

**影響を受けるハードウェア**: 分岐予測を持つすべてのCPU — 事実上すべての現代的なCPU。

### 緩和策 {#mitigations}

**array_index_nospec()**: カーネル内の主要な緩和策である。境界チェックが失敗する場合、投機的実行中であっても、インデックスをゼロにマスクする。

```c
/* include/linux/nospec.h */
/*
 * array_index_nospec() - 境界チェックの後に配列インデックスを無害化する。
 *
 * 次のようなコード列に対して:
 *   if (index < size) {
 *       index = array_index_nospec(index, size);
 *       val = array[index];
 *   }
 *
 * 無害化の後は、予測を誤った投機的実行パスの中であっても、
 * 'index'は範囲内に収まる（またはゼロになる）。
 */
#define array_index_nospec(index, size)                          \
({                                                               \
    typeof(index) _i = (index);                                  \
    typeof(size)  _s = (size);                                   \
    unsigned long _mask = array_index_mask_nospec(_i, _s);       \
    BUILD_BUG_ON(sizeof(_i) > sizeof(long));                     \
    (typeof(_i)) (_i & _mask);                                   \
})
```

`array_index_mask_nospec()`は、CPUが投機によって回避できない算術演算を使い、`index < size`のときは`~0UL`、それ以外のときは`0`となるマスクを計算する。

```c
static inline unsigned long array_index_mask_nospec(unsigned long index,
                                                      unsigned long size)
{
    /*
     * sizeからindexを引き、size <= indexのときにボロー（ビット63）を生じさせる。
     * マスクは算術右シフトの結果 — すべて1かすべて0になる。
     */
    unsigned long mask;
    asm volatile ("cmp %1,%2; sbb %0,%0;"
                  : "=r" (mask) : "g"(size), "r" (index) : "cc");
    return mask;
}
```

**コンパイラによる緩和策**: カーネルは`-mindirect-branch=thunk-extern`（GCC）またはそれに相当するオプションでコンパイルされ、これは（後述の）Spectre v2にも役立つ。v1については、境界チェックの後にシリアライズを行う`LFENCE`フェンスを挿入できる。カーネルは重要なパスでこれを選択的に使っている。

システムコールのディスパッチパスは、`sys_call_table[]`へのアクセスを保護するために`array_index_nospec()`を使う。

```c
/* arch/x86/entry/common.c */
if (likely(unr < NR_syscalls)) {
    unr = array_index_nospec(unr, NR_syscalls);
    regs->ax = sys_call_table[unr](regs);
}
```

---

## Spectre v2（CVE-2017-5715）: 分岐ターゲットインジェクション {#spectre-v2-cve-2017-5715-branch-target-injection}

### 脆弱性 {#the-vulnerability_2}

Spectre v2は**間接分岐予測器**（BTB: Branch Target Buffer）を標的にする。攻撃者はBTBを汚染して、被害者の間接呼び出しや間接ジャンプが、攻撃者の制御するコードを投機的に実行するよう仕向けられる。

攻撃の流れ:
1. 攻撃者はBTBを訓練し、被害者の間接呼び出しのアドレスを攻撃者が選んだターゲットに関連付けさせる
2. 被害者が間接呼び出しを実行すると、CPUは攻撃者のターゲットへ投機的にジャンプする
3. 攻撃者のターゲットは、秘密を読み取ってキャッシュにエンコードするガジェットである

これはプロセス間（同じCPUコア上）で機能し、構成によっては特権レベルをまたいで（ユーザー → カーネル）も機能する。

**影響を受けるハードウェア**: 間接分岐予測を持つすべての現代的なCPU。

### 緩和策1: Retpoline {#mitigation-1-retpoline}

**Retpoline**（return trampoline）は、間接呼び出しと間接ジャンプを、ターゲットに対する分岐予測を無効化する命令列に置き換えるソフトウェア技法である。

```asm
/* *raxを介した間接呼び出しのためのRetpoline命令列 */

call    __x86_indirect_thunk_rax

.global __x86_indirect_thunk_rax
__x86_indirect_thunk_rax:
    /* ステップ1: 本当のターゲットをスタックにプッシュする */
    call    1f
    /* ステップ2: 投機的実行はここでループに陥る */
    .align 16
2:  pause
    lfence
    jmp     2b          /* 投機的な無限ループ */
    /* ステップ3: 実際のcallはRIPをこの地点に設定する */
1:  mov     %rax, (%rsp) /* 戻りアドレスをターゲットで置き換える */
    ret                  /* 投機的にはループ。実際にはターゲットへジャンプ */
```

CPUの戻りアドレス予測器（RSB: Return Stack Buffer）は、`ret`が`call 1f`の次の命令 — すなわち`pause; lfence; jmp`のループ — へジャンプすると予測する。投機的実行はそのループの中で無害に空回りする。`ret`の実際の（投機的でない）実行は正しいスタックの値（`*rax`に上書きされたもの）を使い、本当のターゲットへジャンプする。

カーネルは`-mindirect-branch=thunk-extern`（GCC 7.3以降）でコンパイルされ、すべての間接呼び出し/ジャンプがretpolineのサンクに置き換えられる。サンクは`arch/x86/lib/retpoline.S`で定義されている。

### 緩和策2: eIBRS（Enhanced IBRS） {#mitigation-2-eibrs-enhanced-ibrs}

新しいIntel CPU（Ice Lake以降）は、ハードウェアによる緩和策である**eIBRS**（Enhanced Indirect Branch Restricted Speculation）を実装している。ブート時に一度`IA32_SPEC_CTRL.IBRS = 1`を設定すれば、CPUは特権をまたぐBTBの汚染を継続的に防ぐ — retpolineは不要になる。

eIBRSを持つCPUでは、カーネルはretpolineを無効にし、代わりにハードウェアの緩和策を使う。

```bash
dmesg | grep -i "retpoline\|eibrs\|ibrs"
# "Spectre v2 : Mitigation: Retpoline"
# または
# "Spectre v2 : Mitigation: Enhanced / Automatic IBRS"
```

### 緩和策3: IBRS（素の） {#mitigation-3-ibrs-plain}

素の**IBRS**（Indirect Branch Restricted Speculation）は特権をまたぐBTBの汚染を制限するが、カーネルに入るたびに再度有効にしなければならない（性能への影響を避けるため、出るときにクリアされる）。eIBRSを持たないCPUでは、retpolineより遅い。

```bash
cat /sys/devices/system/cpu/vulnerabilities/spectre_v2
# 次のいずれか:
# "Mitigation: Retpoline"
# "Mitigation: Enhanced / Automatic IBRS"
# "Mitigation: IBRS"
# "Vulnerable"
```

---

## IBRS、IBPB、STIBP: MSRベースの3つの制御 {#ibrs-ibpb-and-stibp-the-three-msr-based-controls}

これら3つの制御は、`IA32_SPEC_CTRL` MSR（`0x48`）と`IA32_PRED_CMD` MSR（`0x49`）を介して動作する。

### IBRS（Indirect Branch Restricted Speculation） {#ibrs-indirect-branch-restricted-speculation}

- `IA32_SPEC_CTRL`のビット0
- セットされているとき: 低い特権レベル（ユーザー）の間接分岐が、高い特権レベル（カーネル）の投機的実行のターゲットに影響を与えられなくなる
- 性能コスト: 古い実装では計測可能なほどある
- eIBRS: 常時有効なハードウェア版。現代のIntelでは性能コストはない

### IBPB（Indirect Branch Predictor Barrier） {#ibpb-indirect-branch-predictor-barrier}

- `IA32_PRED_CMD`のビット0に1を書き込むと、間接分岐予測器の状態がフラッシュされる
- プロセス間のコンテキストスイッチで使われる: プロセスAのBTBの状態がプロセスBに影響を与えるのを防ぐ
- 一方向のバリア — コンテキストスイッチ時にカーネルが発行する

```c
/* arch/x86/include/asm/nospec-branch.h */
static inline void indirect_branch_prediction_barrier(void)
{
    u64 val = PRED_CMD_IBPB;
    alternative_msr_write(MSR_IA32_PRED_CMD, val, X86_FEATURE_USE_IBPB);
}
```

### STIBP（Single Thread Indirect Branch Predictors） {#stibp-single-thread-indirect-branch-predictors}

- `IA32_SPEC_CTRL`のビット1
- 兄弟ハイパースレッドが互いの間接分岐予測に影響を与えるのを防ぐ
- 2つのハードウェアスレッドが1つの物理コアを共有するSMT（ハイパースレッディング）CPUで意味を持つ
- カーネルはSTIBPを常に有効にすることも、タスク単位で（プロセスが要求した場合に）有効にすることも、まったく有効にしないこともできる

```bash
# STIBPの緩和策の状態
cat /sys/devices/system/cpu/vulnerabilities/spectre_v2
# "Mitigation: Retpoline, IBPB: conditional, STIBP: conditional, RSB filling, PBRSB-eIBRS: Not affected"
```

---

## Spectre v4（CVE-2018-3639）: Speculative Store Bypass {#spectre-v4-cve-2018-3639-speculative-store-bypass}

### 脆弱性 {#the-vulnerability_3}

CPUは、ロードのアドレスが直前のストアと衝突しないと投機し、ストアの値をバイパスすることがある — つまり、書き込まれたばかりの値ではなく、キャッシュやメモリから古いデータを読み取る。攻撃者はこれを悪用して、上書きされているはずの値を読み取れる。

### 緩和策: SSBD {#mitigation-ssbd}

**SSBD**（Speculative Store Bypass Disable）は、`IA32_SPEC_CTRL`のビット2をセットして、投機的なストアバイパスをグローバルに無効化する。

```c
/* arch/x86/kernel/cpu/bugs.c */
static void ssb_select_mitigation(void)
{
    switch (ssb_mode) {
    case SPEC_STORE_BYPASS_DISABLE:
        /* SSBDをグローバルに有効にする */
        x86_spec_ctrl_base |= SPEC_CTRL_SSBD;
        wrmsrl(MSR_IA32_SPEC_CTRL, x86_spec_ctrl_base);
        break;
    case SPEC_STORE_BYPASS_PRCTL:
        /* prctl(PR_SET_SPECULATION_CTRL)によりスレッド単位で有効にする */
        break;
    /* ... */
    }
}
```

ユーザープロセスは次のようにしてSSBDを要求できる。

```c
prctl(PR_SET_SPECULATION_CTRL, PR_SPEC_STORE_BYPASS, PR_SPEC_DISABLE, 0, 0);
```

カーネルは、そのようなスレッドをスケジュールインするときにSSBDビットをセットし、コンテキストスイッチでスケジュールアウトするときにクリアする。

```bash
cat /sys/devices/system/cpu/vulnerabilities/spec_store_bypass
# "Mitigation: Speculative Store Bypass disabled via prctl"
# または "Vulnerable"
```

---

## MDS: Microarchitectural Data Sampling（CVE-2018-12130など） {#mds-microarchitectural-data-sampling-cve-2018-12130-etc}

MDSは、CPU内部のバッファ（ラインフィルバッファ、ストアバッファ、ロードポート）から、同じ物理CPUコア上の攻撃者へデータを漏洩させうる脆弱性のファミリーである。

変種:

- **MFBDS**（CVE-2018-12130）: Microarchitectural Fill Buffer Data Sampling（ZombieLoad）
- **MLPDS**（CVE-2018-12127）: Microarchitectural Load Port Data Sampling
- **MSBDS**（CVE-2018-12126）: Microarchitectural Store Buffer Data Sampling（Fallout）
- **MDSUM**（CVE-2019-11091）: Microarchitectural Data Sampling Uncacheable Memory

### 緩和策: VERW {#mitigation-verw}

`VERW`命令（本来はセグメントの検証用）は、影響を受けるIntel CPUにおいて、カーネルからユーザーモードへ遷移する直前やゲストVMに入る直前に実行されたとき、副作用として関連するマイクロアーキテクチャのバッファをフラッシュするよう転用された。

```asm
/* arch/x86/entry/entry_64.S — SYSRET/IRETの前 */
/* MDSの影響を受けるCPUでは: */
sub     $8, %rsp
mov     %ds, (%rsp)
verw    (%rsp)       /* CPUバッファをフラッシュする */
add     $8, %rsp
```

これはalternative機構を介して適用される。

```c
/* arch/x86/include/asm/nospec-branch.h */
static inline void mds_clear_cpu_buffers(void)
{
    static const u16 ds = __KERNEL_DS;
    asm volatile("verw %[ds]" : : [ds] "m" (ds) : "cc");
}
```

`VERW`は安価（数サイクル）だが、カーネル→ユーザーの遷移のたびに実行しなければならないため、小さな一定のオーバーヘッドが加わる。

```bash
cat /sys/devices/system/cpu/vulnerabilities/mds
# "Mitigation: Clear CPU buffers; SMT vulnerable"
# "Not affected"
```

---

## 性能への影響 {#performance-impact}

緩和策は計測可能なオーバーヘッドを課し、それはシステムコールの多いワークロードやコンテキストスイッチの多いワークロードに集中する。

| 脆弱性 | 緩和策 | 典型的なオーバーヘッド |
|---------------|------------|-----------------|
| Meltdown | KPTI（PTI） | システムコールの多いワークロードで5–30%。PCIDありで約5% |
| Spectre v2 | Retpoline | 間接呼び出しの頻度に応じて2–15% |
| Spectre v2 | eIBRS（ハードウェア） | 約1%（現代のIntel Ice Lake以降） |
| Spectre v1 | `array_index_nospec` | 無視できる程度（チェックごとに数命令） |
| IBPB | コンテキストスイッチごと | コンテキストスイッチの多いワークロードで1–10% |
| STIBP | 継続的 | SMTシステムで5–15%（常時有効の場合） |
| MDS | 出口ごとのVERW | 約1-2%（カーネルの出口ごとに数サイクル） |
| Spectre v4 | SSBD | スレッド単位で有効にした場合2–8%。オフなら無視できる程度 |

ワークロード別の影響:

- **データベースサーバー（PostgreSQL、MySQL）**: システムコールの頻度が高い → KPTIとIBPBの影響が最も大きい
- **Webサーバー（nginx、Apache）**: システムコールの頻度は中程度。retpolineのオーバーヘッドが見える
- **HPC / バッチ計算**: ほとんどがユーザー空間での計算。緩和策の影響はほぼ見えない
- **コンテナオーケストレーション**: コンテキストスイッチの頻度が高い。IBPBのオーバーヘッドが効いてくる

```bash
# システムコールのオーバーヘッドを前後で測定する（緩和策オフとオンを比較する）
# ブート時に指定: mitigations=off   （すべて無効にする。隔離されたベンチマーク専用！）
# 通常:           mitigations=auto  （デフォルト）
# 最大:           mitigations=auto,nosmt

# 簡易的なシステムコールのベンチマーク
perf stat -e instructions,cycles,cache-misses -r 5 \
    bash -c 'for i in $(seq 100000); do :; done'

# CR3切り替えのコストを測定する（KPTI）
perf stat -e dTLB-load-misses,iTLB-load-misses ls

# retpolineの状況を見る
grep . /sys/devices/system/cpu/vulnerabilities/*
dmesg | grep -E "Spectre|Meltdown|MDS|retpoline|IBRS|IBPB|STIBP|VERW"
```

---

## 緩和策の制御 {#mitigation-control}

カーネルは緩和策を制御するいくつかの方法を提供している。

### ブートパラメータ {#boot-parameters}

```bash
# すべての緩和策を無効にする（危険！ 隔離された環境でのベンチマーク専用）
mitigations=off

# デフォルトの自動選択
mitigations=auto

# 最も強い隔離のためにSMT（ハイパースレッディング）も無効にする
mitigations=auto,nosmt

# 個別の制御
nopti                    # KPTIを無効にする
nospectre_v1             # Spectre v1の緩和策を無効にする
nospectre_v2             # Spectre v2の緩和策を無効にする
spec_store_bypass_disable=off  # SSBDを無効にする
mds=off                  # MDSの緩和策を無効にする
```

### 実行時の制御（プロセス単位） {#runtime-control-per-process}

```c
/* Spectre v4 / SSBD — ユーザーは自分のプロセスに対して緩和策を要求できる */
prctl(PR_SET_SPECULATION_CTRL, PR_SPEC_STORE_BYPASS, PR_SPEC_DISABLE, 0, 0);
prctl(PR_SET_SPECULATION_CTRL, PR_SPEC_INDIRECT_BRANCH, PR_SPEC_DISABLE, 0, 0);
```

### すべての緩和策を一度に観察する {#observing-all-mitigations-at-once}

```bash
grep . /sys/devices/system/cpu/vulnerabilities/*
# /sys/devices/system/cpu/vulnerabilities/itlb_multihit:KVM: Mitigation: VMX disabled
# /sys/devices/system/cpu/vulnerabilities/l1tf:Not affected
# /sys/devices/system/cpu/vulnerabilities/mds:Not affected
# /sys/devices/system/cpu/vulnerabilities/meltdown:Not affected
# /sys/devices/system/cpu/vulnerabilities/mmio_stale_data:Not affected
# /sys/devices/system/cpu/vulnerabilities/retbleed:Not affected
# /sys/devices/system/cpu/vulnerabilities/spec_rstack_overflow:Not affected
# /sys/devices/system/cpu/vulnerabilities/spec_store_bypass:Mitigation: Speculative Store Bypass disabled via prctl
# /sys/devices/system/cpu/vulnerabilities/spectre_v1:Mitigation: usercopy/swapgs barriers and __user pointer sanitization
# /sys/devices/system/cpu/vulnerabilities/spectre_v2:Mitigation: Retpoline; IBPB: conditional; IBRS_FW; STIBP: conditional; RSB filling; PBRSB-eIBRS: Not affected; BHI: Not affected
# /sys/devices/system/cpu/vulnerabilities/srbds:Not affected
# /sys/devices/system/cpu/vulnerabilities/tsx_async_abort:Not affected
```

---

## 年表 {#timeline}

| 日付 | 出来事 |
|------|-------|
| 2018年1月3日 | MeltdownとSpectre v1/v2の協調的な一般公開 |
| 2018年1月2日 | KPTI（Meltdownの修正）を含むLinux 4.15がリリースされる |
| 2018年1月 | GoogleがRetpoline技法を公開。カーネル4.15にretpolineが入る |
| 2018年5月 | Spectre v4（SSBD）が公開される。カーネル4.17がSSBDのprctlを追加 |
| 2018年8月14日 | L1TF（Foreshadow）が公開される。カーネル4.18が緩和策を追加 |
| 2019年5月 | MDS（Zombieload）が公開される。カーネル5.1がVERWによるフラッシュを追加 |
| 2019年11月 | TAA（TSX Async Abort）。カーネル5.4が緩和策を追加 |
| 2020年以降 | マイクロアーキテクチャの脆弱性が次々と見つかり続け、リリースごとに緩和策が追加されている |

---

## 参考文献 {#further-reading}

### カーネルソースとドキュメント {#kernel-source-documentation}

- [Spectre Side Channels](https://docs.kernel.org/admin-guide/hw-vuln/spectre.html) — 変種と緩和策の制御についての、カーネル自身による管理者向けドキュメント
- [arch/x86/kernel/cpu/bugs.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/cpu/bugs.c) — すべての緩和策が選択・報告される場所
- [meltdownattack.com](https://meltdownattack.com/) — MeltdownとSpectreの原論文

### 関連ページ {#related-pages}

- [ARM64におけるSpectreとMeltdown](../arm64/spectre-meltdown.md) — 同じ脆弱性をarm64の視点から見る
- [x86-64のページテーブル](page-tables.md) — KPTIの仕組み: 2つのPGD、PCID
- [カーネルのハードニング](../../security/kernel-hardening.md) — より広い緩和策の全体像

### LWNの記事 {#lwn-articles}

- [Notes from the Intelpocalypse](https://lwn.net/Articles/742702/) — 開示の週を冷静に解説したもの
- [KAISER: hiding the kernel from user space](https://lwn.net/Articles/738975/) — 結果的にMeltdownを阻止したKASLR防御
