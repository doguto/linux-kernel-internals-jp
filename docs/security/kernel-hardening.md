# カーネルのハードニング: KASLR、スタックプロテクタ、Spectreの緩和策 {#kernel-hardening-kaslr-stack-protectors-and-spectre-mitigations}

> 多層防御: 脆弱性が見つかった後の悪用を困難にする

## 防御の階層 {#defense-layers}

現代のLinuxは、異なるレベルで複数のハードニング技術を実装している:

```
Hardware level:      SMEP/SMAP/CET (Intel), PAN/PXN (ARM)
Kernel compilation:  KASLR, stack canaries, CFI, FORTIFY_SOURCE
Runtime:             ASLR (userspace), seccomp, namespaces, SELinux
CPU microcode:       Spectre/Meltdown mitigations, IBRS, STIBP
```

## KASLR: カーネルアドレス空間配置のランダム化 {#kaslr-kernel-address-space-layout-randomization}

KASLRはカーネルとそのモジュールがメモリ上のどこにロードされるかをランダム化し、攻撃者がエクスプロイトで既知のカーネルアドレスを使うことを困難にする:

```bash
# ユーザー空間のASLRが有効か確認する（randomize_va_spaceはKASLRではなくユーザー空間のASLR）:
cat /proc/sys/kernel/randomize_va_space
# 2 = 完全なASLR（ユーザー空間 + vDSO）

# KASLR（カーネルアドレスのランダム化）が有効か確認する:
dmesg | grep KASLR
# [    0.000000] KASLR enabled

# カーネルのベースアドレス（rootのみ参照可能）:
cat /proc/kallsyms | grep "_text" | head -1
# ffffffffc1000000 T _text  ← ブートごとにランダムなベース

# コンパイル時のベースからのKASLRオフセット:
# amd64: textとモジュール領域を別々にランダム化する
# 範囲: textは+-512MB、モジュールは+-1.5GB
```

```
Boot time:
  Decompressor generates random offset (from RDRAND or RDSEED or RTC jitter)
  Maps kernel at: 0xFFFFFFFF81000000 + random_offset
  Sets up page tables with new addresses
```

**回避**: KASLRは次のものによって破られる:

- `/proc`、未初期化の読み取り、情報漏洩バグによるカーネルポインタのリーク
- デフォルトでない設定での`CONFIG_RANDOMIZE_BASE=n`

```bash
# ポインタのリークに対してハードニングする:
echo 2 > /proc/sys/kernel/kptr_restrict      # 全員からカーネルポインタを隠す
echo 1 > /proc/sys/kernel/dmesg_restrict     # dmesgにCAP_SYSLOGを要求する
```

## スタックカナリア（stack-protector） {#stack-canaries-stack-protector}

コンパイラが挿入するカナリアは、関数から戻る前にスタックバッファオーバーフローを検出する:

```c
/* CONFIG_STACKPROTECTORを有効にした場合: */
void vulnerable_function(char *input)
{
    /* コンパイラが挿入する: */
    unsigned long __stack_chk_guard_local = __stack_chk_guard; /* カナリア */

    char buf[64];
    strcpy(buf, input);  /* オーバーフローの可能性 */

    /* コンパイラがreturnの前に挿入する: */
    if (__stack_chk_guard_local != __stack_chk_guard)
        __stack_chk_fail();   /* PANIC: スタック破壊を検出 */
}

/* カーネル内の__stack_chk_fail: */
void __stack_chk_fail(void)
{
    panic("stack-protector: Kernel stack is corrupted in: %pB\n",
          __builtin_return_address(0));
}
```

```bash
# スタックプロテクタが有効か確認する:
grep CONFIG_STACKPROTECTOR /boot/config-$(uname -r)
# CONFIG_STACKPROTECTOR=y
# CONFIG_STACKPROTECTOR_STRONG=y  ← より多くの関数を保護する（より遅い）

# 発動することを確認する:
dmesg | grep "stack-protector"
# 破壊が検出された場合: 関数アドレス付きでカーネルパニック
```

## SMEPとSMAP（Intel x86） {#smep-and-smap-intel-x86}

**SMEP**（Supervisor Mode Execution Prevention）: カーネルがユーザー空間のコードページを実行することを防ぐ。
**SMAP**（Supervisor Mode Access Prevention）: カーネルが予期せずユーザー空間のメモリにアクセスすることを防ぐ。

```bash
# ハードウェアのサポートを確認する:
grep -m1 smep /proc/cpuinfo  # flagsにsmepがある
grep -m1 smap /proc/cpuinfo  # flagsにsmapがある

# カーネルがそれらを使っているか確認する:
grep CONFIG_X86_SMAP /boot/config-$(uname -r)
# CONFIG_X86_SMAP=y

# SMEP: CR4.SMEPビットがセットされている必要がある（ブート時に検証）
# SMAP: カーネルはcopy_to_userの前後でstac/clac命令を使う

# ARM64での相当機能:
# PAN (Privileged Access Never) = SMAP
# PXN (Privileged Execute Never) = SMEP
```

## HARDENED_USERCOPY {#hardened_usercopy}

ユーザー空間との間で過大なデータをコピーしうるカーネルバグを防ぐ:

```bash
CONFIG_HARDENED_USERCOPY=y
# copy_to_user/copy_from_userに境界チェックを追加する:
# - カーネルポインタは有効なslabオブジェクトを指しているか
# - コピーサイズはオブジェクトの境界内か
# - コピー対象がスタックのアドレスではないか

# 違反した場合:
# usercopy: Kernel memory overwrite attempt detected to slab ... size=xxx ...
```

## FORTIFY_SOURCE {#fortify_source}

よく使われる文字列/メモリ関数に対する、コンパイラ挿入の境界チェック:

```c
/* CONFIG_FORTIFY_SOURCE=y: */
/* カーネルのmemcpyは次のようになる: */
#define memcpy(dst, src, n) \
    __builtin_memcpy_chk(dst, src, n, __builtin_object_size(dst, 0))
/* __builtin_object_sizeはdstのコンパイル時または実行時のサイズを与える */
/* n > size(dst)なら: __fortify_panic() */

/* 可変サイズのオブジェクトに対しては実行時に: */
static inline void *memcpy(void *dst, const void *src, size_t len)
{
    size_t p_size = __builtin_object_size(p, 0);
    if (len > p_size)
        fortify_panic("memcpy", ...);
    return __underlying_memcpy(dst, src, len);
}
```

## SpectreとMeltdownの緩和策 {#spectre-and-meltdown-mitigations}

### Meltdown (CVE-2017-5754) {#meltdown-cve-2017-5754}

Meltdownは、ユーザー空間が投機的実行を介してカーネルメモリを読むことを可能にする。**KPTI**（Kernel Page-Table Isolation）によって緩和される:

```bash
# KPTIの状態を確認する:
dmesg | grep -i "page table isolation\|PTI"
# [    0.000000] Kernel/User page tables isolation: enabled

# KPTIのコスト:
# - カーネルモードとユーザーモードで別々のページテーブル
# - syscallのたびに: TLBフラッシュ + CR3の切り替え（高コスト）
# - ハードウェア機能PCIDで軽減される

# PCIDのサポートを確認する:
grep pcid /proc/cpuinfo | head -1
# flags: ... pcid ...  ← TLBがアドレス空間でタグ付けされ、フラッシュ不要
```

### Spectre v1 (CVE-2017-5753): Bounds Check Bypass {#spectre-v1-cve-2017-5753-bounds-check-bypass}

```c
/* 脆弱なパターン（Spectre v1）: */
if (index < array_size)          /* 境界チェック - 投機的に迂回されうる */
    return array[index];          /* 範囲外データの投機的な読み取り */

/* 緩和策: array_index_nospec() */
index = array_index_nospec(index, array_size);
/* 条件付き移動（cmov）を使い、分岐なしでindexを無害化する */
/* 境界チェックの投機的な迂回を無効にする */
return array[index];
```

### Spectre v2 (CVE-2017-5715): Branch Target Injection {#spectre-v2-cve-2017-5715-branch-target-injection}

```bash
# 緩和策: Retpoline（return trampoline）
# 間接分岐をreturn + トランポリンで置き換える:
# IBRS (Indirect Branch Restricted Speculation)
# eIBRS（Enhanced IBRS）: 新しいCPUでのハードウェアによる緩和策

# 緩和策の状態を確認する:
cat /sys/devices/system/cpu/vulnerabilities/spectre_v2
# Mitigation: Enhanced / IBRS, IBPB: conditional, RSB filling, PBRSB-eIBRS: SW sequence

# IBPB: Indirect Branch Predictor Barrier
# コンテキストスイッチ時に分岐予測器をフラッシュする（高コスト: 約4000サイクル）
grep ibpb /proc/cpuinfo   # flagsにibpbがある
cat /sys/kernel/debug/x86/ibpb_enabled  # 0/1
```

### MDS/TAAの緩和策 {#mdstaa-mitigations}

```bash
# CPUのすべての脆弱性の状態を確認する:
cat /sys/devices/system/cpu/vulnerabilities/*
# itlb_multihit: Not affected
# l1tf: Mitigation: PTE Inversion
# mds: Mitigation: Clear CPU buffers; SMT disabled
# meltdown: Mitigation: PTI
# spec_store_bypass: Mitigation: Speculative Store Bypass disabled via prctl
# spectre_v1: Mitigation: usercopy/swapgs barriers and __user pointer sanitization
# spectre_v2: Mitigation: Enhanced / IBRS, IBPB: conditional
# srbds: Not affected
# tsx_async_abort: Not affected

# ベンチマークのために緩和策を無効にする（危険 - テスト専用）:
# 次のオプションでブート: mitigations=off
# または: nospectre_v1 nospectre_v2 nopti mds=off
```

## CFI: 制御フロー完全性 {#cfi-control-flow-integrity}

**Clang CFI**（CONFIG_CFI_CLANG）は、間接呼び出しのターゲットを実行時に検証することで制御フローの乗っ取りを防ぐ:

```c
/* CFIなし: 攻撃者は関数ポインタを上書きして任意の場所を指させられる */
/* CFIあり: 間接呼び出しのターゲットは宣言された関数型と一致しなければならない */

/* Forward CFI: 呼び出し先の型シグネチャが一致するか確認する */
void (*fn_ptr)(int, int) = get_function_pointer();
fn_ptr(arg1, arg2);  /* CFIはfn_ptrが(int,int)->voidの関数を指していることを検証する */

/* 違反した場合: カーネルパニック / BUG() */
```

```bash
# CFIが有効か確認する:
grep CONFIG_CFI_CLANG /boot/config-$(uname -r)
# CONFIG_CFI_CLANG=y
# CONFIG_CFI_PERMISSIVE=n  （n = 強制、y = 警告のみ）

# AndroidはすべてのカーネルモジュールにCFIを使う
```

## ハードニング設定のチェックリスト {#hardening-configuration-checklist}

```bash
# 確認すべき主要なセキュリティ設定オプション:
zcat /proc/config.gz | grep -E \
    "CONFIG_RANDOMIZE_BASE|CONFIG_STACKPROTECTOR|CONFIG_SMAP|CONFIG_SMEP|\
CONFIG_HARDENED_USERCOPY|CONFIG_FORTIFY_SOURCE|CONFIG_CFI_CLANG|\
CONFIG_INIT_ON_ALLOC_DEFAULT_ON|CONFIG_INIT_ON_FREE_DEFAULT_ON|\
CONFIG_PAGE_TABLE_ISOLATION|CONFIG_RETPOLINE"

# 期待値: すべて=yであるべき（性能のために=nの場合もあるINIT_*を除く）

# 実行時のチェック:
cat /sys/devices/system/cpu/vulnerabilities/*  # すべて"Mitigation"と表示されるべき
grep kaslr_offset /proc/kallsyms               # rootが必要なはず
```

## 参考文献 {#further-reading}

- [LSMフレームワーク](lsm.md) — SELinux/AppArmorのセキュリティポリシー
- [クレデンシャル](credentials.md) — ハードニングが保護する権限モデル
- [seccomp BPF](seccomp.md) — syscallフィルタリング
- [ページフォールトハンドラ](../mm/page-fault.md) — ページテーブルを介して実装されるSMEP/SMAP
- `Documentation/security/` — カーネルのセキュリティドキュメント
- kernel-hardeningメーリングリスト — アップストリームのハードニングに関する議論
- KSPP（Kernel Self Protection Project）の推奨設定
