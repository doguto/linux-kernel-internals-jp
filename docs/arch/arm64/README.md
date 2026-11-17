# ARM64アーキテクチャ {#arm64-architecture}

> 例外レベル、メモリモデル、そしてARM固有のカーネル内部構造

## なぜARM64か {#why-arm64}

ARM64（AArch64）は次の分野で支配的なアーキテクチャである。

- モバイル（Android、iOS）
- 組み込みおよびIoT（Raspberry Pi、NXP、TI）
- サーバー（AWS Graviton、Ampere Altra）
- ノートPC（Apple Silicon、Qualcomm Snapdragon X）

これらのプラットフォームでカーネルを扱うには、ARM64固有の挙動を理解することが不可欠である。

## このセクションのページ {#pages-in-this-section}

| ページ | 扱う内容 |
|------|----------------|
| [ブートシーケンス](boot.md) | Head.S、アイデンティティマッピング、デコンプレッサ、MMU有効化、SMPの立ち上げ |
| [例外モデル](exception-model.md) | EL0-EL3、VBAR_EL1、シンドロームレジスタ、GIC |
| [メモリモデル](memory-model.md) | 弱い順序付け、バリア、load-acquire/store-release、LDAR/STLR |
| [ページテーブル](page-tables.md) | TTBR0/TTBR1、グラニュールサイズ、PTE形式、ASID、MAIR_EL1、stage 2 |
| [システムコールのエントリ](syscall-entry.md) | SVC命令、el0_svc、システムコールテーブル、vDSO、compat |
| [CPU機能とalternatives](cpu-features.md) | 機能検出、alternativesによるパッチ適用、hwcaps、エラッタ、SVE/BTI/MTE |
| [SpectreとMeltdown](spectre-meltdown.md) | Spectre v1/v2/BHB/v4、SSBS、CSV2、BHBクリア、KPTI |
| [事件簿](war-stories.md) | SVEのコンテキストスイッチ、TLBの順序付け、DMAコヒーレンシ、BTIクラッシュ、エラッタ範囲のバグ |

## ARM64とx86の主な違い {#arm64-vs-x86-key-differences}

| 機能 | x86-64 | ARM64 |
|---------|--------|-------|
| メモリ順序付け | TSO（total store order） | 弱い（RVWMO relaxed） |
| 特権レベル | ring 0-3 | EL0-EL3 |
| 割り込みコントローラ | APIC | GIC（Generic Interrupt Controller） |
| システムレジスタ | MSR/RDMSR | MRS/MSR命令 |
| SIMD | SSE/AVX | NEON / SVE / SME |
| ページサイズ | 4K/2M/1G | 4K/16K/64Kベース + 2M/1G huge |
| TLBシュートダウン | 他のCPUへのIPI | DSBを介したTLBIブロードキャスト |

## クイックリファレンス {#quick-reference}

```bash
# ARM64のCPU機能
cat /proc/cpuinfo | grep Features
# Features: fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics ...

# ARM64固有のシステムレジスタ（perfまたはMSR相当の手段で）
# CNTVCT_EL0を読む（仮想カウンタ — RDTSCのARM版）
# mrs x0, CNTVCT_EL0

# 実行中のコードの例外レベル
# EL0 = ユーザー空間、EL1 = カーネル、EL2 = ハイパーバイザ、EL3 = セキュアモニタ
```
