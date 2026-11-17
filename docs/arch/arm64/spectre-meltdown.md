# ARM64におけるSpectreとMeltdown {#spectre-and-meltdown-on-arm64}

> 投機実行の脆弱性とその緩和策

ARM64はx86とは異なる脆弱性プロファイルを持つ。ほとんどのARM64 CPUはMeltdownの影響を**受けず**、Spectre系で主に問題となるのはSpectre v1、Spectre v2、Spectre-BHB、そしてSpeculative Store Bypassである。緩和策は、x86のMSRベースの仕組みではなく、ARM64固有のアセンブリ、システムレジスタ、アーキテクチャ上の機能ビットによって実装されている。

---

## 概要表 {#summary-table}

| 脆弱性 | CVE | 影響を受けるARM64ハードウェア | 主な緩和策 | KPTIは必要か |
|---------------|-----|-------------------------|--------------------|--------------|
| Meltdown | CVE-2017-5754 | Cortex-A75のみ | KPTI | はい（A75のみ） |
| Spectre v1 | CVE-2017-5753 | すべて | `array_index_nospec()` | いいえ |
| Spectre v2 | CVE-2017-5715 | ほとんど | CSV2 / IBPB / ファームウェア | いいえ |
| Spectre-BHB | CVE-2022-23960 | 多数（A72、A77、A78、…） | CLRBHB / ループ / CSV2_3 | いいえ |
| Spectre v4 / SSB | CVE-2018-3639 | CPUに依存 | SSBSレジスタビット | いいえ |

---

## 背景: 投機実行とキャッシュサイドチャネル {#background-speculative-execution-and-cache-side-channels}

現代のCPUは命令を投機的に実行する。どの分岐が取られるかを予測し、確定したプログラムの流れより先に実行するのである。予測が誤っていれば、投機的に実行された結果は破棄される。問題は、**キャッシュ上の副作用は破棄されない**ことである。

キャッシュサイドチャネルは次のように機能する。

1. プローブ配列をキャッシュからフラッシュする
2. 被害者に投機的にメモリへアクセスさせ、秘密データをキャッシュにロードさせる
3. プローブ配列へのアクセス時間を計測し、どのキャッシュラインがロードされたかを特定する
4. タイミングの差から秘密のバイトを推測する

この**Flush+Reload**手法が、ここで説明するすべての脆弱性の根底にある。

ARM64の**弱い順序付け**のメモリモデル（[メモリモデル](memory-model.md)を参照）は、投機的ロードがx86のTotal Store Orderモデルとは異なる振る舞いをすることを意味する。これが、ほとんどのARM64実装が古典的なMeltdownに対して脆弱でない主な理由である。

---

## Meltdown（CVE-2017-5754） {#meltdown-cve-2017-5754}

### なぜほとんどのARM64は影響を受けないか {#why-most-arm64-is-not-affected}

x86でMeltdownが成立するのは、CPUがユーザーモードからカーネルアドレスを投機的に読み、パーミッションフォールトが発生する前にその値が一時的にレジスタ内で可視になるからである。これは、投機的ロードが後続の命令へ伝播する値を生成しうるという、TSO的な振る舞いに依存している。

ARM64の弱いメモリモデルは、一般にこれを許さない。ほとんどのARM64マイクロアーキテクチャでは、ロードでのパーミッションフォールトにより、ロードされた値が依存する命令へフォワードされることが防がれる。投機ウィンドウが開かないか、開いても使える値を生まない。

**例外: Cortex-A75。** ARMは、Cortex-A75がMeltdown相当の攻撃の影響を受けやすいことを確認している。この場合、投機的にロードされたカーネルのバイトが、パーミッションアボートが発生する前にキャッシュ状態に影響を与えうる。

### 緩和策: KPTI（Cortex-A75のみ） {#mitigation-kpti-cortex-a75-only}

KPTI（Kernel Page Table Isolation）は、ユーザーモードで使うページテーブルとカーネルモードで使うページテーブルを分離し、EL0のコードが実行されている間はカーネルアドレスがマップされないようにする。マップされていないアドレスの投機的読み込みはキャッシュされた値ではなくTLBミスを生むので、サイドチャネルが断たれる。

ARM64では、KPTIは`arch/arm64/mm/mmu.c`の`arm64_kernel_use_ng_mappings()`によって制御される。KPTIが必要な場合、カーネルは自身のマッピングを非グローバル（`nG`ビットをセット）としてマークし、ユーザーモードのTLBがカーネルの変換をキャッシュしないようにする。

```bash
cat /sys/devices/system/cpu/vulnerabilities/meltdown
# Cortex-A75:   "Mitigation: PTI"
# それ以外すべて: "Not affected"
```

ARM64ハードウェアの大多数（AWS Graviton、Apple Silicon、Qualcomm Snapdragon、ほとんどのCortex-Aシリーズ）では、KPTIは有効化されず、それに伴う性能低下もない。

---

## Spectre v1（CVE-2017-5753）: 境界チェックのバイパス {#spectre-v1-cve-2017-5753-bounds-check-bypass}

### 脆弱性 {#the-vulnerability}

Spectre v1は、境界チェックを越えた投機実行を悪用する。攻撃者は、条件付きの境界チェックに対して「分岐成立」と予測するよう分岐予測器を訓練し、その後に範囲外のインデックスを与える。CPUは不正なインデックスを使ったロードを投機的に実行し、その結果をキャッシュにエンコードする。

```c
/* 脆弱なパターン — ユーザーから与えられたインデックスを扱うカーネルコードに見られる */
if (index < array_size) {
    /*
     * index >= array_sizeであっても、CPUはここを投機的に実行する。
     * 分岐予測器が「成立」と予測するよう訓練されているからである。
     */
    value = kernel_array[index];           /* 投機的な範囲外読み込み */
    sink  = probe_array[value * 4096];     /* 秘密をキャッシュにエンコード */
}
/* パーミッションチェックが解決し、ロールバック — だがprobe_array[secret*4096]はキャッシュに残っている */
```

Spectre v1は、すべてのARM64実装を含め、分岐予測を持つ**すべての**CPUに影響する。

### 緩和策: array_index_nospec() {#mitigation-array_index_nospec}

カーネル内の主な緩和策は、`include/linux/nospec.h`の`array_index_nospec()`を使う。このマクロは、CPUが投機的に回避できる条件分岐を使わずに、インデックスが範囲内ならすべて1、そうでなければすべて0となるマスクを計算する。

```c
/* include/linux/nospec.h */
#define array_index_nospec(index, size)                          \
({                                                               \
    typeof(index) _i = (index);                                  \
    typeof(size)  _s = (size);                                   \
    unsigned long _mask = array_index_mask_nospec(_i, _s);       \
    BUILD_BUG_ON(sizeof(_i) > sizeof(long));                     \
    (typeof(_i)) (_i & _mask);                                   \
})
```

ARM64における`array_index_mask_nospec()`の実装は、条件付き選択（CSEL）を使い、投機的な分岐なしにインデックスをゼロに切り詰める。

```c
/* arch/arm64/include/asm/barrier.h */
static inline unsigned long array_index_mask_nospec(unsigned long idx,
                                                     unsigned long sz)
{
    unsigned long mask;
    /*
     * idx >= szなら0、そうでなければすべて1のマスクを作る。
     * CPUが投機的に通り抜けられないCSELベースの演算を使う。
     */
    asm volatile(
    "   cmp     %1, %2\n"
    "   cset    %0, lo\n"          /* lo: idx < sz（符号なし） */
    "   neg     %0, %0\n"          /* 0 → 0、1 → 0xFFFF...FFFF */
    : "=r" (mask)
    : "r" (idx), "r" (sz)
    : "cc");
    return mask;
}
```

サニタイズ後のパターンは次のようになる。

```c
if (index < array_size) {
    index = array_index_nospec(index, array_size);  /* 投機的にも切り詰める */
    val = array[index];   /* 安全: 投機的であってもindexは範囲内 */
}
```

### 投機バリア {#speculation-barrier}

`array_index_nospec()`が適用できないパスでは、`__nospeculation_barrier()`（ARM64の`CSDB` — Consumption of Speculative Data Barrier、古いコアでは`ISB`に対応付けられる）を挿入して、CPUが投機的にロードした値を後続の命令へフォワードしないようにできる。

```bash
cat /sys/devices/system/cpu/vulnerabilities/spectre_v1
# "Mitigation: __user pointer sanitization"
```

---

## Spectre v2（CVE-2017-5715）: 分岐ターゲットインジェクション {#spectre-v2-cve-2017-5715-branch-target-injection}

### 脆弱性 {#the-vulnerability_1}

Spectre v2は**間接分岐予測器**（Branch Target Buffer、BTB）を標的とする。EL0で動作する攻撃者はBTBのエントリを汚染でき、カーネルが間接分岐（関数ポインタ呼び出し、仮想ディスパッチ）を実行したときに、CPUが攻撃者の選んだガジェットへ投機的にジャンプし、データをキャッシュにエンコードするようにできる。

```
Attack sequence:
1. Attacker (EL0) repeatedly calls an indirect branch at address A → target T_attacker
   (trains the BTB: when kernel calls A, predict target T_attacker)
2. Kernel (EL1) executes:  blr x9  (indirect call via register)
   CPU speculatively jumps to T_attacker (attacker's gadget)
3. T_attacker reads a kernel secret and encodes it via cache timing
4. Rollback — but cache is warm
```

### 緩和策1: CSV2（アーキテクチャ上の保証） {#mitigation-1-csv2-architectural-guarantee}

**CSV2**（Cache Speculation Variant 2）は`ID_AA64PFR0_EL1`にあるアーキテクチャ上の機能ビットである。CSV2=1のとき、CPU実装は分岐予測状態が**例外レベルやセキュリティドメインをまたいで共有されない**ことを保証する。ユーザーモードの攻撃者は、EL1が使うBTBエントリを汚染できない。

CSV2はマイクロアーキテクチャ上の性質であり、ソフトウェアによる緩和シーケンスを一切必要としない。CSV2=1を公開しているCPUには、Apple Mシリーズ、AWS Graviton3、そして最近の多くのCortex-Aコアが含まれる。

```bash
cat /sys/devices/system/cpu/vulnerabilities/spectre_v2
# "Mitigation: CSV2"               — ハードウェアによる保証、SWのオーバーヘッドなし
# "Mitigation: Branch predictor hardening"   — ファームウェア/SWによる緩和策が有効
# "Vulnerable"                     — 利用可能な緩和策なし
```

### 緩和策2: SMCCCファームウェア呼び出しによるIBPB {#mitigation-2-ibpb-via-smccc-firmware-calls}

一部のARM64実装は、SMCCC（ARM SMC Calling Convention）インターフェイスを使い、ファームウェアを通じてIBPB相当（Indirect Branch Predictor Barrier）の機能を公開している。カーネルは特権遷移の際にEL3のファームウェアを呼び出し、分岐予測器の状態をフラッシュする。

カーネルはSMCCCの`ARCH_WORKAROUND_1`および`ARCH_WORKAROUND_2`関数IDを介してこの機能を検出し、利用可能であればEL0→EL1の遷移にファームウェア呼び出しを挿入する。

### 緩和策3: リターントランポリン（ARM64版のretpoline相当） {#mitigation-3-return-trampolines-arm64-retpoline-equivalent}

ARM64にはx86のRSB（Return Stack Buffer）を使ったretpolineの技法はないが、カーネルはソフトウェアによる緩和策が必要なコアにおいて、間接分岐に**投機安全な呼び出しシーケンス**を使う。これらは、リターンに類するシーケンスの後に`SB`（Speculation Barrier、ARMv8.0-SB）または`ISB`を挿入し、CPUが分岐の解決点を越えて投機実行するのを防ぐことに依拠している。

エントリパスにあるカーネルの`call_via_r*`トランポリンはこれらのバリアを使い、攻撃者が制御する投機パスがカーネル内のガジェットを実行するのを防ぐ。

---

## Spectre-BHB（CVE-2022-23960）: Branch History Bufferインジェクション {#spectre-bhb-cve-2022-23960-branch-history-buffer-injection}

### 脆弱性 {#the-vulnerability_2}

Spectre-BHB（Branch History Buffer）は、2022年3月に公開された、Spectre v2をより深刻にした改良版である。**Branch History Buffer**（BHB）は間接分岐の予測に影響する。BTBのエントリだけでなく、BTBへのセレクタにも影響するのである。BHBを汚染した攻撃者は、CSV2を実装したコアであっても、被害者の間接分岐予測をリダイレクトできる。

これは幅広いARMコアに影響する。

- Cortex-A72（Raspberry Pi 4や多くのサーバーSoCで使用）
- Cortex-A73、Cortex-A75
- Cortex-A76、Cortex-A77、Cortex-A78
- Neoverse N1（AWS Graviton2で使用）、Neoverse V1（AWS Graviton3で使用）、Neoverse N2（AWS Graviton4で使用）
- その他さまざまな実装（ARMの公式エラッタ一覧を参照）

### 緩和策1: CSV2_3（アーキテクチャ上の分離） {#mitigation-1-csv2_3-architectural-isolation}

**CSV2_3**はCSV2の拡張（`ID_AA64PFR0_EL1`にエンコードされる）であり、さらにセキュリティドメイン間のBHBの分離を保証する。CSV2_3=1のCPUでは、ソフトウェアによるBHBの緩和策は不要である。

### 緩和策2: CLRBHB命令（ARMv8.9 / Cortex-X3以降） {#mitigation-2-clrbhb-instruction-armv89-cortex-x3-and-later}

**CLRBHB**はARMv8.9で導入された新しい命令で、Branch History Bufferを明示的にクリアする。Linuxカーネルは、EL0→EL1遷移の例外エントリパスに`CLRBHB`を挿入し、ユーザーモードのBHB状態がカーネルの間接分岐予測に影響しないようにする。

```asm
/* arch/arm64/kernel/entry.S — カーネルエントリでのBHBクリア（CLRBHB版） */
    clrbhb
    isb
```

`CLRBHB`の後の`ISB`は、後続の間接分岐が予測される前にBHBのクリアが確実に有効になるために必要である。

### 緩和策3: BHBクリアループ（ファームウェアが指定） {#mitigation-3-bhb-clearing-loop-firmware-specified}

`CLRBHB`も`ECBHB`も持たないコアでは、カーネルは**分岐の多いループ**を実行し、BHBの履歴を無害なエントリで上書きする。ループの反復回数はそのマイクロアーキテクチャのBHBの深さに依存するため、CPUモデルごとにファームウェアによって（SMCCC呼び出し、またはカーネル内のCPU固有データを介して）指定される。

```asm
/* arch/arm64/kernel/entry.S — BHBクリアループ（簡略化） */
/*
 * 'k'回の分岐を実行してBranch History Bufferをフラッシュする。
 * ループ回数はCPU固有である（例: Cortex-A72では32回）。
 */
    mov     x0, #<loop_count>
1:  b       2f
2:  subs    x0, x0, #1
    b.ne    1b
    dsb     nsh
    isb
```

このシーケンスは影響を受けるコアのすべてのEL0→EL1の例外エントリ（システムコール、IRQ、フォールト）に挿入される。つまり、これらのCPUではカーネルに入るたびに小さいながらも測定可能なコストがかかる。

### 緩和策4: ECBHB（ハードウェアによる自動クリア） {#mitigation-4-ecbhb-hardware-automatic-clearing}

一部の新しいARMコアは`ID_AA64MMFR1_EL1`の**ECBHB**ビットをセットしている。ECBHB=1のとき、CPUハードウェアは下位ELからの例外エントリ時に自動的にBHBをクリアする。ソフトウェアによるBHBクリアのシーケンスは不要である。カーネルはこのビットをチェックし、セットされていればループや`CLRBHB`のシーケンスをスキップする。

### BHB緩和策の確認 {#checking-for-bhb-mitigation}

```bash
cat /sys/devices/system/cpu/vulnerabilities/spec_rstack_overflow
# （BHBではない — これはAMD固有）

cat /sys/devices/system/cpu/vulnerabilities/spectre_v2
# "Mitigation: CSV2, BHB"          — ハードウェアのCSV2_3またはECBHB
# "Mitigation: Spectre-BHB loop, Force BHB"
# "Mitigation: CLRBHB"
```

---

## Spectre v4 / Speculative Store Bypass（CVE-2018-3639） {#spectre-v4-speculative-store-bypass-cve-2018-3639}

### 脆弱性 {#the-vulnerability_3}

CPUは、ロードが直近の未完了のストアとエイリアスしないと投機することがある（アドレスの曖昧性解消はコストが高いため）。その投機が誤っていると、ロードは書き込まれたばかりの値ではなく、キャッシュやメモリから**古い値**を返す。

攻撃者は次のようにしてこれを悪用できる。
1. 安全なポインタ値をメモリに書き込む
2. 書き込みが完了する前に、CPUに同じアドレスから別の（攻撃者が制御する）値を投機的にロードさせる
3. その古い値を配列インデックスとして使い、キャッシュにエンコードする

### 緩和策: SSBSレジスタビット {#mitigation-ssbs-register-bit}

緩和策をサポートするARM64 CPUは、`PSTATE`に**SSBS**（Speculative Store Bypass Safe）ビットを公開している。`SSBS=0`にすると、CPUに投機的なストアバイパスを無効化するよう指示し、攻撃を防ぐ。このシステムレジスタには`MSR SSBS, Xn` / `MRS Xn, SSBS`でアクセスする（レジスタ名は`SSBS`であり、`SSBS_EL1`ではない）。

サポートの有無は**`ID_AA64PFR1_EL1`のSSBSフィールド**で示される。

カーネルは`prctl`を介してSSBSの制御をユーザープロセスに公開している。

```c
/* ユーザープロセスが緩和策を選択する */
prctl(PR_SET_SPECULATION_CTRL,
      PR_SPEC_STORE_BYPASS,
      PR_SPEC_DISABLE,
      0, 0);
```

スレッドが緩和策を要求すると、カーネルはそのスレッドをスケジュールインするときに`SSBS=0`をセットし、コンテキストスイッチでスケジュールアウトするときに元に戻す。オーバーヘッドは明示的に選択したスレッドに限られる。

```bash
cat /sys/devices/system/cpu/vulnerabilities/spec_store_bypass
# "Mitigation: Speculative Store Bypass disabled via prctl"
# "Not affected"
# "Vulnerable"
```

---

## MTEと投機 {#mte-and-speculation}

ARM64のMemory Tagging Extension（MTE、ARMv8.5-A）は、メモリの16バイトのグラニュールごとに4ビットのタグを関連付ける。タグが割り当てタグと一致しないポインタを使ったロードやストアはフォールトを発生させる。

MTEは主要なSpectre緩和策では**ない**が、副次的な効果がある。タグチェックにより、どの投機的ロードが使える値を生みうるかが制約される。タグの合わないアドレスへの投機的ロードは最終的にフォールトするため、一部の投機ガジェットの有用性が制限される。

MTEは主としてメモリ安全性とバグ検出のための機能である。詳細はカーネルのドキュメント`arm64/memory-tagging-extension.rst`を参照のこと。

---

## 脆弱性の状態を確認する {#checking-vulnerability-status}

```bash
# すべての脆弱性の状態を一度に表示
for f in /sys/devices/system/cpu/vulnerabilities/*; do
    echo "$(basename $f): $(cat $f)"
done

# CSV2対応のGraviton3システムでの出力例:
# meltdown: Not affected
# spectre_v1: Mitigation: __user pointer sanitization
# spectre_v2: Mitigation: CSV2, BHB
# spec_store_bypass: Mitigation: Speculative Store Bypass disabled via prctl
# srbds: Not affected
```

最近のほとんどのARM64ハードウェアでは、出力は心強いものである。Meltdownは関係なく、Spectre v2はハードウェア（CSV2）でカバーされ、Spectre-BHBはハードウェアでクリアされる（ECBHB）か、短いエントリパスのシーケンスで緩和される。

---

## カーネルのコードの場所 {#kernel-code-locations}

ARM64のSpectre緩和策は複数のファイルに分散している。

| ファイル | 含まれるもの |
|------|-----------------|
| `arch/arm64/kernel/entry.S` | EL0→EL1エントリでのBHBクリアループと`CLRBHB`。カーネルエントリのトランポリン |
| `arch/arm64/kernel/proton-pack.c` | 実行時検出、CPUごとの緩和状態、SMCCCファームウェア呼び出し |
| `arch/arm64/include/asm/spectre.h` | 緩和策の種類のenum、CPUごとのspectre状態の構造体 |
| `arch/arm64/include/asm/barrier.h` | `array_index_mask_nospec()`のARM64実装。`__nospeculation_barrier()` |
| `include/linux/nospec.h` | アーキテクチャ非依存の`array_index_nospec()`マクロ |
| `arch/arm64/mm/mmu.c` | `arm64_kernel_use_ng_mappings()` — Cortex-A75向けのKPTIの判断 |

### entry.SでのBHBクリア {#bhb-clearing-in-entrys}

`entry.S`の例外エントリマクロは、ブート時にセットされるCPUごとのケーパビリティフラグに基づいて、BHBの緩和策を条件付きで適用する。alternativesによるパッチ適用の仕組み（`alternative_if` / `alternative_else`）が、影響を受けないハードウェアでは共通パスに分岐を追加することなく、実行時に適切なシーケンスを選択する。

```asm
/* arch/arm64/kernel/entry.Sにおける概念的な構造 */
/* EL0からの例外エントリ時: */

alternative_if ARM64_WORKAROUND_SPECTRE_BHB_LOOP
    /* 分岐によるクリアループを挿入（回数はCPUモデル固有） */
    ...
alternative_else_nop_endif

alternative_if ARM64_WORKAROUND_SPECTRE_BHB_INSN
    clrbhb
    isb
alternative_else_nop_endif
```

このパターンにより、ECBHBやCSV2_3を持つCPUは元の`nop`命令を実行し（オーバーヘッドなし）、影響を受けるCPUは緩和シーケンスを実行する。

---

## x86との比較 {#comparison-with-x86}

| 観点 | x86 | ARM64 |
|--------|-----|-------|
| Meltdown | 2019年以前のほとんどのIntel CPUに影響 | Cortex-A75のみ |
| KPTI | ほぼすべてのIntelで必要 | Cortex-A75でのみ必要 |
| Spectre v2の緩和策 | Retpoline / eIBRS / IBRS MSR | CSV2（ハードウェア）、SMCCCファームウェア、エントリのバリア |
| Spectre v1 | `array_index_nospec()` + `LFENCE` | `array_index_nospec()` + `CSDB`/`ISB` |
| Spectre v4 | `IA32_SPEC_CTRL.SSBD`ビット | SSBS `PSTATE`ビット |
| BHB | BHI_DIS_S / eIBRS（最近のIntel） | CLRBHB / ループ / ECBHB / CSV2_3 |
| 緩和策の制御 | MSRへの書き込み（ring 0） | システムレジスタのビット。EL3へのSMCCC呼び出し |

アーキテクチャ上の重要な違いとして、ARM64ではほとんどのハードウェア緩和策が、実行時に切り替えるMSRベースの制御ではなく、ブート時にチェックされる**システムレジスタのビット**（`ID_AA64PFR0_EL1`、`ID_AA64PFR1_EL1`、`ID_AA64MMFR1_EL1`）として表現される。このため、ソフトウェアスタックはより単純であることが多い。ハードウェアが分離を保証する（CSV2、ECBHB）か、エントリパスの短いシーケンスがそれを処理するかのどちらかである。

---

## 緩和策のブートパラメータ {#mitigation-boot-parameters}

```bash
# すべての緩和策を無効化（危険 — 隔離されたベンチマーク専用）
mitigations=off

# デフォルトの自動選択
mitigations=auto

# 特定のARM64緩和策を無効化
nospectre_v1            # Spectre v1の緩和策を無効化
nospectre_v2            # Spectre v2 / BHBの緩和策を無効化
ssbd=off                # Speculative Store Bypassの緩和策を無効化
nopti                   # KPTIを無効化（いずれにせよA75以外では効果なし）
```

---

## 性能面の考慮事項 {#performance-considerations}

ほとんどのARM64システムはハードウェアレベルの分離を持つか、短いエントリパスのシーケンスしか必要としないため、ARM64のSpectre緩和策の性能への影響は一般にx86より**小さい**。

| 緩和策 | オーバーヘッド |
|------------|----------|
| KPTI（Cortex-A75のみ） | システムコールが多い場合5–30%。ASIDベースの最適化で約5% |
| `array_index_nospec()` | 無視できる（境界チェックごとに数命令） |
| BHBクリアループ | カーネルエントリごとに小さな一定コスト（影響を受けるコアで約20–40サイクル） |
| `CLRBHB` | ごく小さい（約1–2サイクル + ISB） |
| ECBHB / CSV2_3 | ゼロ（ハードウェア、ソフトウェアのオーバーヘッドなし） |
| SSBS | スレッド単位なら無視できる。無効時はゼロ |
| SMCCC IBPBファームウェア呼び出し | 影響を受けるコアでは特権遷移時に中程度 |

最近のハードウェア（Graviton3、Apple Mシリーズ、Cortex-X3以降）では、CSV2_3またはECBHBによってカバーされるため、これらのパスのほとんどは`nop`のalternativesを実行し、測定可能なオーバーヘッドは生じない。

---

## 参考文献 {#further-reading}

- [x86におけるSpectreとMeltdown](../x86/spectre-meltdown.md) — x86の視点: KPTI、retpoline、eIBRS、IBRS/IBPB/STIBP、MDS/VERW
- [ARM64の例外モデル](exception-model.md) — EL0-EL3の遷移、VBAR_EL1、BHBクリアが挿入される例外エントリパス
- [ARM64のCPU機能](cpu-features.md) — カーネルがブート時にCPU機能ビットをどう検出し使うか（x86のCPUIDに対するシステムレジスタ方式）
- ARM Architecture Reference Manual（ARM DDI 0487） — `ID_AA64PFR0_EL1`のCSV2/CSV2_3フィールド、`ID_AA64PFR1_EL1`のSSBSフィールド、`ID_AA64MMFR1_EL1`のECBHBフィールド
- ARM Security Advisory: Spectre-BHB（2022） — コアごとのBHBの深さの表と推奨ループ回数
- [Spectre Side Channels](https://docs.kernel.org/admin-guide/hw-vuln/spectre.html) — すべてのSpectre変種に関する正統なカーネルドキュメント
- [LWN: Notes from the Intelpocalypse](https://lwn.net/Articles/742702/) — 公開の経緯とカーネルの対応
