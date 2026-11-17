# x86-64のブートシーケンス {#x86-64-boot-sequence}

> BIOS/UEFIのリセットベクタからstart_kernel()まで

x86-64のブートプロセスは、3つのCPUモードを通り抜ける旅である: リアルモード（16ビット）、プロテクトモード（32ビット）、ロングモード（64ビット）。カーネルは、64ビットコードを1行でも実行できるようになる前に、ブートのたびにこの遷移を通り抜けなければならない。UEFIはこの経路を短縮するが、なくすわけではない。

---

## 2つのブート経路 {#two-boot-paths}

現代のx86-64システムは、2つのファームウェアインターフェースのいずれかを介してブートする。

| | BIOS | UEFI |
|--|------|------|
| 位置付け | レガシー（2010年以前のシステム、一部のVM） | すべての現代的なシステム |
| リセットベクタ | リアルモードで0xFFFFFFF0 | ファームウェアがEFIスタブを直接呼ぶ |
| 制御を渡す時点のCPUモード | 16ビットのリアルモード | 32ビットのプロテクトモード（または64ビット） |
| メモリマップの受け渡し | BIOS INT 0x15 / e820 | `GetMemoryMap()`によるEFIメモリマップ |
| ブートローダーの要否 | 必要（GRUB2、syslinux） | 任意（EFIスタブで直接ブートできる） |

---

## BIOSのブート経路 {#bios-boot-path}

### リセットベクタとPOST {#reset-vector-and-post}

電源投入時、x86-64のCPUはリアルモードにリセットされ、**リセットベクタ**、すなわち物理アドレス`0xFFFFFFF0`（ROMにマップされている）で最初の命令を実行する。これは4GBのアドレス空間の最上位16バイトである。

BIOSファームウェアは**POST**（Power-On Self Test）を実行する。

- RAMを初期化してテストする
- PCIデバイスを列挙する
- メモリマップを検出する（INT 0x15/E820経由でOSに渡される）
- ブートデバイスのMBR（先頭512バイト）から第1段階のブートローダーをロードする

### GRUB2: カーネルのロード {#grub2-loading-the-kernel}

GRUB2（Grand Unified Bootloader version 2）は、BIOSシステムで最も一般的なLinuxのブートローダーである。BIOSによってMBRからロードされた後、次のことを行う。

1. GRUB2の第2段階がカーネルイメージ（`/boot/vmlinuz-*`）をメモリにロードする
2. `struct boot_params`（`arch/x86/include/uapi/asm/bootparam.h`で定義）を読み取るか構築する
3. `boot_params.e820_table`にe820メモリマップのエントリを書き込む
4. 32ビットブートプロトコルを使い、圧縮スタブ内の`startup_32`へ直接ジャンプして、リアルモードのセットアップコードを丸ごと飛ばす（512バイトのブートセクタはオフセット0–511を占める。リアルモードのセットアップコードはオフセット512/0x200から始まるが、現代のGRUB2は32ビットのエントリポイントを使ってそれを飛ばす）

```c
/* arch/x86/include/uapi/asm/bootparam.h */
struct boot_params {
    struct screen_info       screen_info;        /* 0x000 */
    struct apm_bios_info     apm_bios_info;      /* 0x040 */
    /* ... */
    __u8                     e820_entries;        /* 0x1e8 */
    struct setup_header      hdr;                 /* 0x1f1 */
    /* ... */
    struct boot_e820_entry   e820_table[E820_MAX_ENTRIES_ZEROPAGE]; /* 0x2d0 */
};

struct setup_header {
    __u8    setup_sects;      /* セットアップセクタの数 */
    __u16   root_flags;
    __u32   syssize;          /* プロテクトモードコードのサイズ（16バイト単位） */
    /* ... */
    __u32   kernel_alignment; /* カーネルに要求されるアライメント */
    __u8    relocatable_kernel;
    /* ... */
    __u64   pref_address;     /* 推奨ロードアドレス */
    __u32   init_size;        /* 初期化に必要なサイズ */
};
```

### bzImageのレイアウト {#bzimage-layout}

カーネルは`bzImage`（big zImage） — 自己展開型のアーカイブ — として配布される。

```
bzImage on disk:
┌────────────────────────────────────────────┐
│  Boot sector (512 bytes, MBR-compatible)   │
├────────────────────────────────────────────┤
│  Real-mode setup code (arch/x86/boot/)     │
│  • setup.elf compiled to 16-bit code       │
│  • Detects memory, video mode, CPUID       │
│  • Switches to protected mode              │
├────────────────────────────────────────────┤
│  Protected-mode stub (startup_32)          │
│  arch/x86/boot/compressed/head_32.S        │
├────────────────────────────────────────────┤
│  Compressed vmlinux (gzip/lz4/zstd/xz)     │
│  • The actual kernel ELF, compressed       │
│  • Decompressed into place at runtime      │
└────────────────────────────────────────────┘
```

---

## リアルモードのセットアップコード（`arch/x86/boot/`） {#real-mode-setup-code-archx86boot}

リアルモードのセットアップコードは16ビットモードで動作し、プロテクトモードに切り替える前に初期のハードウェア検出を行う。主要なファイル: `arch/x86/boot/main.c`、`arch/x86/boot/memory.c`、`arch/x86/boot/video.c`。

### メモリ検出: e820 {#memory-detection-e820}

セットアップコードはINT 0x15/E820を使って、BIOSに物理メモリマップを問い合わせる。

```c
/* arch/x86/boot/memory.c */
static int detect_memory_e820(void)
{
    int count = 0;
    struct biosregs ireg, oreg;
    struct boot_e820_entry *desc = boot_params.e820_table;
    static struct boot_e820_entry buf; /* staticなのでゼロ初期化される */

    initregs(&ireg);
    ireg.ax  = 0xe820;
    ireg.cx  = sizeof(buf);
    ireg.edx = SMAP;    /* 'SMAP'シグネチャ */
    ireg.di  = (size_t)&buf;

    do {
        intcall(0x15, &ireg, &oreg);
        ireg.ebx = oreg.ebx; /* 継続値 */
        /* ... bufをboot_params.e820_table[count++]にコピーする ... */
    } while (ireg.ebx && count < ARRAY_SIZE(boot_params.e820_table));

    return boot_params.e820_entries = count;
}
```

得られたe820テーブルは、どの物理メモリ範囲が使用可能なRAMか、予約済み（ACPI、MMIO）か、使用不可かを記述する。カーネルのメモリ管理サブシステム（`memblock`）は、ブートの後半でこのテーブルを使う。

### ビデオモードの設定とその他の検出 {#video-mode-setup-and-other-detection}

セットアップコードは次のことも行う。

- 利用可能なビデオモードを検出し、コンソールの解像度を設定する
- APM BIOS、EDD（Enhanced Disk Drive）の情報を問い合わせる
- CPUIDを読んでCPUの機能を検出する
- セットアップコード自身が使うヒープを用意する

### プロテクトモードへの切り替え: CR0.PEを有効にする {#switching-to-protected-mode-enabling-cr0pe}

リアルモードのセットアップコードの最後の仕事は、32ビットのプロテクトモードへの切り替えである。

```c
/* arch/x86/boot/pm.c */
void go_to_protected_mode(void)
{
    /* プロテクトモードに入る前のフック */
    realmode_switch_hook();

    /* A20ラインを有効にする（1MBを超えるアクセスに必要） */
    if (enable_a20()) {
        puts("A20 gate not responding, unable to boot...\n");
        die();
    }

    /* コプロセッサをリセットする */
    reset_coprocessor();

    /* すべての割り込みを無効にする — これからモードを切り替える */
    mask_all_interrupts();

    /* 最小限のGDTとIDTを用意する */
    setup_idt();
    setup_gdt();

    /* 切り替え: CR0.PEビットをセットし、32ビットコードへファージャンプする */
    protected_mode_jump(boot_params.hdr.code32_start,
                        (u32)&boot_params + (ds() << 4));
}
```

`protected_mode_jump`（`arch/x86/boot/pmjump.S`内）は`CR0.PE = 1`を設定し、`startup_32` — 最初の32ビットコード — へファージャンプを実行する。

---

## プロテクトモード → ロングモード {#protected-mode-long-mode}

### startup_32（`arch/x86/boot/compressed/head_32.S`） {#startup_32-archx86bootcompressedhead_32s}

`startup_32`は32ビットのプロテクトモードで動作する。その仕事は次のとおりである。

1. フラットな32ビットディスクリプタでセグメントレジスタを設定する
2. BSSをクリアする
3. スタックを用意する
4. 自身が動作している物理アドレスを計算する（位置独立）
5. 遷移中に必要となるアイデンティティマッピング用の初期ページテーブルを用意する
6. PAE（Physical Address Extension）を有効にする: `CR4.PAE = 1`を設定する
7. 初期ページテーブルを`CR3`にロードする
8. WRMSR（MSR 0xC0000080）で`EFER.LME = 1`を設定する
9. ページングを有効にする: `CR0.PG = 1`を設定する — これによりロングモードが不可分に有効になる（EFER.LMAが1になる）
10. 64ビットコードセグメントへファージャンプを行う → CPUは64ビットのロングモードになる

```asm
/* arch/x86/boot/compressed/head_32.S（簡略化） */

/* PAEを有効にする */
movl    %cr4, %eax
orl     $X86_CR4_PAE, %eax
movl    %eax, %cr4

/* ページテーブルをロードする */
leal    pgtable(%ebx), %eax
movl    %eax, %cr3

/* ロングモードを有効にする（EFER.LME） */
movl    $MSR_EFER, %ecx
rdmsr
btsl    $_EFER_LME, %eax
wrmsr

/* ページング + 保護を有効にする。これでロングモードが有効になる */
movl    $(X86_CR0_PG | X86_CR0_PE), %eax
movl    %eax, %cr0

/* 64ビットコードセグメントへファージャンプする */
lret    /* スタックからcs:eipをポップする。csは64ビットディスクリプタを選択する */
```

### startup_64（`arch/x86/boot/compressed/head_64.S`） {#startup_64-archx86bootcompressedhead_64s}

`startup_64`は最初の64ビットコードである。この時点では次のとおりである。

- CPUは64ビットのロングモードにある
- 最小限のアイデンティティマップされたページテーブルしか存在しない
- 圧縮されたカーネルのblobはまだメモリ上にあり、展開されていない

`startup_64`は、カーネルを展開するのに十分な基盤を用意する。
1. 必要なら自身を再配置する（KASLR — Kernel Address Space Layout Randomization — のため）
2. 適切なスタックを用意する
3. `extract_kernel()`を呼び、それが`decompress_kernel()`を呼ぶ

### 展開 {#decompression}

```c
/* arch/x86/boot/compressed/misc.c */
asmlinkage __visible void *extract_kernel(void *rmode, memptr heap,
                                           unsigned char *input_data,
                                           unsigned long input_len,
                                           unsigned char *output,
                                           unsigned long output_len)
{
    /* ... */
    decompress_kernel(output, input_data, input_len, error);
    /* ELFヘッダを解析し、再配置を処理する */
    parse_elf(output);
    /* ... */
    return output;  /* 展開されたカーネルのエントリポイントを返す */
}
```

展開器は自己完結している — ビルド設定に応じて、展開ライブラリ（gzip、lz4、zstd、xz、またはlzma）の独自のコピーを含んでいる。展開後、制御は`arch/x86/kernel/head_64.S`にある、展開済みカーネルの`startup_64`へ移る。

---

## UEFIのブート経路 {#uefi-boot-path}

### EFIからの引き渡し {#efi-handoff}

UEFIファームウェアはカーネルイメージをPE/COFF実行ファイルとしてロードし、そのエントリポイントを直接呼ぶ。カーネルの**EFIスタブ**（`arch/x86/boot/compressed/efi_stub_64.S`、`drivers/firmware/efi/libstub/`）がこれを処理する。

1. EFIスタブはUEFI環境内で動作する（ファームウェアに応じてプロテクトモードまたはロングモード）
2. UEFIのBoot Servicesを呼び出して次のことを行う:
   - カーネル用のメモリを割り当てる
   - UEFIメモリマップ（e820に相当）を取得する
   - カーネルが期待する`struct boot_params`を用意する
3. `ExitBootServices()`を呼んで、UEFIファームウェアの制御を手放す
4. カーネルのエントリポイントへジャンプする

利点: **リアルモードが不要**である。ファームウェアがハードウェアの初期化（POSTに相当）を行い、BIOSよりはるかにクリーンな状態で制御を渡す。

```c
/* drivers/firmware/efi/libstub/x86-stub.c（簡略化） */
efi_status_t __efiapi efi_pe_entry(efi_handle_t handle,
                                    efi_system_table_t *sys_table_arg)
{
    /* ... メモリを割り当て、boot_paramsを構築する ... */
    status = efi_stub_common(handle, image, sys_table_arg, cmdline_ptr);
    /* ... */
    return status;
}
```

UEFIでブートしたシステムには`/sys/firmware/efi`ディレクトリが存在する。

```bash
# UEFIでブートしたかを確認する
ls /sys/firmware/efi          # UEFIブートなら存在し、BIOSなら存在しない
cat /sys/firmware/efi/fw_platform_size  # 64 = 64ビットUEFI

# EFI変数
efibootmgr -v               # ブート順序とエントリを表示する
```

---

## カーネル自身のstartup_64（`arch/x86/kernel/head_64.S`） {#the-kernels-own-startup_64-archx86kernelhead_64s}

展開後、展開済みカーネルの`startup_64`が実行される。これは圧縮スタブの`startup_64`とは別物である。これはカーネルの恒久的なページテーブルと中核の基盤を用意する。

```asm
/* arch/x86/kernel/head_64.S（構造を簡略化したもの） */

SYM_CODE_START_NOALIGN(startup_64)
    /* ロングモードにいることを確認する */
    /* %ss, %ds, %es, %fs, %gsをカーネルデータセグメントに設定する */

    /* 初期のカーネルページテーブル（4レベルまたは5レベル）を用意する */
    /* early_top_pgtが初期のPGDである */

    /* CR3がearly_top_pgtを指すように設定する */
    leaq    early_top_pgt(%rip), %rax
    movq    %rax, %cr3

    /* GSベースが初期のper-CPUデータを指すように設定する */
    /* （カーネルではGSはper-CPU変数へのアクセスに使われる） */
    movl    $MSR_GS_BASE, %ecx
    /* ... initial_gsをMSRにロードする ... */
    wrmsr

    /* BSSをクリアする */
    xorl    %eax, %eax
    leaq    __bss_start(%rip), %rdi
    leaq    __bss_stop(%rip), %rcx
    subq    %rdi, %rcx
    shrq    $3, %rcx
    rep stosq

    /* Cに入る: x86_64_start_kernel() */
    leaq    x86_64_start_kernel(%rip), %rax
    callq   *%rax
SYM_CODE_END(startup_64)
```

---

## x86_64_start_kernel() → start_kernel() {#x86_64_start_kernel-start_kernel}

```c
/* arch/x86/kernel/head64.c */
asmlinkage __visible void __init x86_64_start_kernel(char *real_mode_data)
{
    /*
     * まだinitスタック上で動作しており、割り込みはオフである。
     * カーネルはまだ、あらゆる場所で最終的な仮想アドレスにマップされているわけではない。
     */

    /* 最終的な初期ページテーブルを構築する */
    /* BSSはstartup_64のアセンブリですでにクリアされている */

    /*
     * boot_paramsをリアルモードのデータ領域から
     * 最終的な場所にコピーする（ブートローダーからのreal_mode_dataポインタ）
     */
    copy_bootdata(__va(real_mode_data));

    /*
     * GDTとIDTを最終的な値でロードする。
     * これによりTSSと例外処理の基盤が用意される。
     */
    load_ucode_bsp();

    /* cr4のシャドウを初期化する（cr4_set_bitsなどで使われる） */
    cr4_init_shadow();

    /* 最終的なページテーブルの設定。利用可能ならNXを有効にする */
    x86_64_start_reservations(real_mode_data);
}

void __init x86_64_start_reservations(char *real_mode_data)
{
    /* ... */
    start_kernel();  /* アーキテクチャ非依存のエントリポイント */
}
```

`start_kernel()`（`init/main.c`内）は最初のアーキテクチャ非依存のコードである: スケジューラ、メモリ管理、割り込みサブシステムを初期化し、最終的に`init`プロセスを開始する。

### start_kernel()が初期化するもの {#what-start_kernel-initializes}

順に（主なものを抜粋）:

```
start_kernel()
  setup_arch()          — x86 arch init: e820 processing, APIC, CPU detection
  mm_init()             — memory management: memblock → buddy allocator
  sched_init()          — scheduler: runqueues, idle task
  trap_init()           — IDT: exception handlers
  init_IRQ()            — interrupt handling infrastructure
  softirq_init()        — softirq / tasklet infrastructure
  time_init()           — clocksource (TSC, HPET, etc.)
  rest_init()           — creates kernel_init thread, becomes idle
    kernel_init()
      do_initcalls()    — module __initcall() functions
      run_init_process() — exec /sbin/init (PID 1)
```

---

## 主要なデータ構造 {#key-data-structures}

| 構造体 | 場所 | 用途 |
|-----------|----------|---------|
| `struct boot_params` | `arch/x86/include/uapi/asm/bootparam.h` | ブートローダー → カーネルへのパラメータ受け渡し |
| `struct setup_header` | 同じファイル。`boot_params`に埋め込まれている | カーネルイメージのメタデータ（ロードアドレス、フラグ） |
| `struct boot_e820_entry` | `arch/x86/include/uapi/asm/e820.h` | 物理メモリマップの1エントリ |
| `struct e820_table` | `arch/x86/include/asm/e820/types.h` | メモリマップ全体（最大128エントリ） |

---

## ブートプロセスを観察する {#observing-the-boot-process}

```bash
# ブートログ全体（GRUBからstart_kernelまで）
dmesg | head -200

# カーネルから見たメモリマップ
cat /proc/iomem

# BIOS/UEFIからのe820エントリ
dmesg | grep -i e820

# UEFI固有
ls /sys/firmware/efi/
cat /sys/firmware/efi/fw_platform_size   # 32または64

# ブートローダーから渡されたカーネルコマンドライン
cat /proc/cmdline

# ブート時に検出されたCPUの詳細
dmesg | grep -E "CPU|microcode|BIOS"

# KASLR: カーネルのロードアドレス（CONFIG_RANDOMIZE_BASE=yならランダム化される）
grep _text /proc/kallsyms | head -1

# ブート時間の内訳（systemdが必要）
systemd-analyze
systemd-analyze blame
```

---

## バージョンに関する注記 {#version-notes}

- **KASLR**（Kernel Address Space Layout Randomization）: x86-64向けにLinux 3.14で導入された。ブートのたびにカーネルの物理および仮想ロードアドレスをランダム化する。`arch/x86/boot/compressed/kaslr.c`で実装されている。
- **ブート時の5レベルページング**: CPUがLA57をサポートし、カーネルが`CONFIG_X86_5LEVEL=y`で構成されていれば、`startup_64`は5レベルのページテーブルを用意する。Linux 4.14で導入された。
- **UEFIスタブ**: カーネル内蔵のEFIスタブ（`CONFIG_EFI_STUB`）はLinux 3.3で導入され、ブートローダーなしでUEFIから直接カーネルをブートできるようになった。
- **圧縮カーネルの形式**: gzipが最も古い。lz4、lzma、xz、zstdのサポートは段階的に追加された。zstd（展開が最も速い）はLinux 5.9で追加された。

---

## 参考文献 {#further-reading}

### カーネルソースとドキュメント {#kernel-source-documentation}

- [The Linux/x86 Boot Protocol](https://docs.kernel.org/arch/x86/boot.html) — セットアップヘッダ、エントリ規約、プロトコルの歴史
- [arch/x86/boot/](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/boot/) — 上で順にたどったリアルモードのセットアップコード
- [arch/x86/kernel/head_64.S](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/head_64.S) — `startup_64`と初期のページング設定

### 関連ページ {#related-pages}

- [x86_64のブート時ページテーブル設定](../../mm/boot-page-tables.md) — 初期ページテーブル、KASLR、5レベルの有効化の詳細
- [memblock](../../mm/memblock.md) — このシーケンスが引き継ぐブート時のメモリアロケータ
- [ARM64のブートシーケンス](../arm64/boot.md) — もう一方の主要アーキテクチャにおける同じ旅路
