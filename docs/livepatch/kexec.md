# kexec {#kexec}

> ハードウェアリセットなしに、稼働中のカーネルの中から新しいカーネルを起動する

## kexecが行うこと {#what-kexec-does}

`kexec`は新しいカーネルをメモリにロードし、BIOS/UEFIのPOSTとブートローダを迂回して直接そこへジャンプする。

```
Normal boot:      BIOS/UEFI → GRUB → kernel
kexec boot:       running kernel → kexec_exec → new kernel
                  (BIOS/POST skipped → faster reboot, savings vary
                   widely by firmware and hardware)
```

主な用途:
1. **高速リブート**: 数分ではなく数秒でカーネルを更新する
2. **kdump**: クラッシュしたカーネルがキャプチャカーネルを起動し、vmcoreを保存する
3. **A/Bカーネル更新**: 新しいカーネルへアトミックに切り替える

## kexec_loadシステムコール {#kexec_load-syscall}

```c
/* kernel/kexec.c */
/*
 * kexec_load(entry, nr_segments, segments, flags)
 *   entry:      新しいカーネルのエントリポイント
 *   nr_segments: ロードするメモリセグメントの数
 *   segments:   kexec_segment構造体の配列
 *   flags:      KEXEC_ON_CRASH=kdump用、KEXEC_ARCH=アーキテクチャ、など
 */

/* include/linux/kexec.h */
struct kexec_segment {
    /* kexec_load()では->buf（ユーザー空間ポインタ）、kexec_file_load()では->kbuf（カーネルポインタ） */
    union {
        void __user *buf;
        void        *kbuf;
    };
    size_t        bufsz;  /* バッファのサイズ */
    unsigned long mem;    /* 配置先の物理アドレス */
    size_t        memsz;  /* 配置先でのサイズ */
};
```

### kexec_file_load: 署名検証付きのロード {#kexec_file_load-signature-verified-loading}

`kexec_file_load`は生のメモリセグメントの代わりにファイルディスクリプタを受け取り、カーネルイメージの署名検証を可能にする。

```c
/* kexec_file_load(kernel_fd, initrd_fd, cmdline_len, cmdline, flags) */
int kexec_file_load(int kernel_fd,      /* open("/boot/vmlinuz", O_RDONLY) */
                     int initrd_fd,      /* initrdのファイルディスクリプタ */
                     unsigned long cmdline_len,
                     const char __user *cmdline,
                     unsigned long flags);
```

`CONFIG_KEXEC_SIG=y`の場合、カーネルイメージは信頼された鍵で署名されていなければならない。これはSecure Bootとの互換性のために有用である。

## machine kexec: ジャンプ {#machine-kexec-the-jump}

`kexec -e`を実行すると、`machine_kexec()`が実際の引き渡しを行う。

```c
/* arch/x86/kernel/machine_kexec_64.c */
void machine_kexec(struct kimage *image)
{
    unsigned long reloc_start = (unsigned long)__relocate_kernel_start;
    relocate_kernel_fn *relocate_kernel_ptr;
    unsigned int relocate_kernel_flags;
    void *control_page;

    /* この時点で、kernel_kexec()はmachine_shutdown()のstop_other_cpus()
       （またはCONFIG_KEXEC_JUMPのpreserve-context経路では
       suspend_disable_secondary_cpus()）によって他のすべてのCPUを
       既に停止させている。この関数自身はCPUを停止させない。 */

    /* リブート中に割り込みは許されない */
    local_irq_disable();
    hw_breakpoint_disable();
    cet_disable();

    control_page = page_address(image->control_code_page);

    /* relocate_kernel_ptrは、control_pageにコピーされた
       再配置トランポリンを指す */
    relocate_kernel_ptr = control_page + ((unsigned long)relocate_kernel - reloc_start);

    relocate_kernel_flags = 0;
    if (image->preserve_context)
        relocate_kernel_flags |= RELOC_KERNEL_PRESERVE_CONTEXT;

    load_segments();

    /* 再配置トランポリンへジャンプする: 新しいカーネルのセグメントを
       最終的な位置へコピーし、その後新しいカーネルのエントリへジャンプする */
    image->start = relocate_kernel_ptr((unsigned long)image->head,
                                        virt_to_phys(control_page),
                                        image->start,
                                        relocate_kernel_flags);
}
```

## kimage: ロードされたカーネル {#kimage-the-loaded-kernel}

```c
/* include/linux/kexec.h */
struct kimage {
    kimage_entry_t  head;            /* pageリストの先頭 */
    kimage_entry_t *entry;           /* pageリスト内の現在位置 */
    kimage_entry_t *last_entry;      /* pageリストの末尾 */

    unsigned long    start;          /* 新しいカーネルのエントリポイント */

    struct page     *control_code_page; /* 再配置コード用のpage */
    struct page     *swap_page;         /* コピー用の一時page */

    unsigned long    nr_segments;    /* ロードされたセグメントの数 */
    struct kexec_segment segment[KEXEC_SEGMENT_MAX]; /* 最大16セグメント */

    struct list_head control_pages;  /* kexec自身が使うpage */
    struct list_head dest_pages;     /* 新しいカーネル用のpage */
    struct list_head unusable_pages; /* 使用できないpage */

    /* 特別な処理を示すフラグ */
    unsigned int type : 1;        /* KEXEC_TYPE_DEFAULTまたはKEXEC_TYPE_CRASH */
    unsigned int preserve_context : 1;
    unsigned int file_mode : 1;   /* kexec_file_load()でロードされた場合にセット */
    unsigned int no_cma : 1;

    /* ... */
};
```

## kdumpとの統合 {#kdump-integration}

kdumpは、プライマリカーネルがクラッシュしたときにキャプチャカーネルを起動するためにkexecを使う。

```
Primary kernel running
    │ panic() / oops
    ▼
machine_crash_shutdown()
    │ stop all CPUs
    │ save registers
    ▼
machine_kexec(kexec_crash_image)
    │
    ▼
Capture kernel boots
    │ reads /proc/vmcore (primary kernel's memory)
    ▼
makedumpfile saves vmcore → reboot
```

キャプチャカーネルは、通常の起動時に予約された物理メモリ領域（`crashkernel=256M`）にロードされる。プライマリカーネルはこの領域を決して使わないため、クラッシュ後もその内容は無傷である。

```bash
# 起動時にkdumpカーネルをロードする
kexec -p /boot/vmlinuz-kdump \
    --initrd=/boot/initrd-kdump.img \
    --reuse-cmdline \
    --append="irqpoll nr_cpus=1 reset_devices"

# ロードされたカーネルを確認する
cat /sys/kernel/kexec_crash_loaded
# 1 = クラッシュカーネルがロード済み

cat /sys/kernel/kexec_loaded
# 0 = 通常のkexecカーネルはロードされていない
```

## kexecのユーザー空間ツール {#kexec-userspace-tool}

```bash
# kexecリブート用にカーネルをロードする
kexec -l /boot/vmlinuz \
    --initrd=/boot/initrd.img \
    --reuse-cmdline       # 現在のブートコマンドラインを使う

# コマンドラインを明示してロードする
kexec -l /boot/vmlinuz \
    --initrd=/boot/initrd.img \
    --append="root=/dev/sda1 quiet"

# 実行: 直ちに新しいカーネルへジャンプする
kexec -e

# あるいは: 次回のリブートで実行する（systemctl経由）
systemctl kexec
# シャットダウン中にkexec -eを実行する

# クラッシュカーネルをロードする
kexec -p /boot/vmlinuz-crash \
    --initrd=/boot/initrd-crash.img \
    --append="1 irqpoll"

# クラッシュカーネルをアンロードする
kexec -p -u
```

## カーネル内のkexec {#kexec-in-the-kernel}

### シャットダウンシーケンス {#shutdown-sequence}

```c
/* kernel/kexec_core.c */
int kernel_kexec(void)
{
    int error = 0;

    if (!kexec_trylock())
        return -EBUSY;
    if (!kexec_image) {
        error = -EINVAL;
        goto Unlock;
    }

#ifdef CONFIG_KEXEC_JUMP
    if (kexec_image->preserve_context) {
        /* preserve-context（"kexec jump"）経路: ハイバネーションと
           同じデバイスサスペンドのコールバックを再利用する */
        pm_prepare_console();
        freeze_processes();
        console_suspend_all();
        dpm_suspend_start(PMSG_FREEZE);
        dpm_suspend_end(PMSG_FREEZE);
        suspend_disable_secondary_cpus();
        local_irq_disable();
        syscore_suspend();
    } else
#endif
    {
        kexec_in_progress = true;
        /* kernel_restart_prepare()はリブート通知チェーンを実行し、
           system_stateを設定し、usermodehelperを無効化し、
           デバイスをシャットダウンする */
        kernel_restart_prepare("kexec reboot");
        migrate_to_reboot_cpu();
        syscore_shutdown();
        cpu_hotplug_enable();
        machine_shutdown();
    }

    kmsg_dump(KMSG_DUMP_SHUTDOWN);
    /* 新しいカーネルへジャンプする。成功時は戻らない */
    machine_kexec(kexec_image);

#ifdef CONFIG_KEXEC_JUMP
    /* preserve_contextで、"jump"先のカーネルが後で制御を返した場合にのみ
       到達する（上のブロックを鏡写しにした再開処理） */
#endif

 Unlock:
    kexec_unlock();
    return error;
}
```

### EFIランタイムサービスの保持 {#preserving-efi-runtime-services}

```c
/* EFIシステムの場合: EFIランタイムメモリを保持する */
if (efi_enabled(EFI_RUNTIME_SERVICES)) {
    /* EFIランタイム領域を保持対象としてマークする */
    /* 新しいカーネルはEFIランタイムサービスを使用できる */
}
```

## kexecの観察 {#observing-kexec}

```bash
# カーネルのkexecサポートを確認する
grep KEXEC /boot/config-$(uname -r)
# CONFIG_KEXEC=y
# CONFIG_KEXEC_FILE=y
# CONFIG_KEXEC_SIG=y   （任意）

# kdump用に予約されたメモリ
cat /proc/iomem | grep -i crash
# 100000000-10fffffff : Crash kernel

# kexec_loadedは、イメージが現在ロードされていて実行可能な状態に
# あるかどうかを示す。稼働中のカーネル自身がkexec経由で
# 起動されたかどうかを示すものではない
cat /sys/kernel/kexec_loaded
dmesg | grep -i kexec

# 所要時間: kexec対コールドブート。kexecはBIOS/UEFIのPOSTと
# ブートローダの段階を飛ばすため、通常は完全なリブートより速い。
# ただし実際の短縮幅（およそ1秒から数十秒まで幅がある）は
# ファームウェアのPOST時間とCPU数に大きく依存するので、
# 固定の数値を仮定せず自分のハードウェアで測定すること
time systemctl kexec
# 対
time reboot
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/kexec.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/kexec.c) — `SYSCALL_DEFINE4(kexec_load, ...)`: セグメントリストに基づくロード用システムコール
- [kernel/kexec_file.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/kexec_file.c) — `SYSCALL_DEFINE5(kexec_file_load, ...)`と`CONFIG_KEXEC_SIG`の署名検証経路
- [kernel/kexec_core.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/kexec_core.c) — `kernel_kexec()`のシャットダウンシーケンスと、`/sys/kernel/kexec_loaded`・`/sys/kernel/kexec_crash_loaded`として公開される`loaded`/`crash_loaded`のsysfs属性
- [include/linux/kexec.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/kexec.h) — `struct kimage`と`struct kexec_segment`の定義
- [arch/x86/kernel/machine_kexec_64.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/machine_kexec_64.c) — `machine_kexec()`: x86-64における新しいカーネルへの実際の引き渡し

### manページ {#man-pages}

- [kexec_load(2)](https://man7.org/linux/man-pages/man2/kexec_load.2.html) — `kexec_load()`と`kexec_file_load()`の両システムコールを扱う
- [kexec(8)](https://man7.org/linux/man-pages/man8/kexec.8.html) — ユーザー空間の`kexec-tools`コマンド: `-l`/`-p`/`-e`/`-u`オプション

### 関連ページ {#related-pages}

- [kdumpとcrash](../debugging/kdump.md) — kexecを使ったクラッシュダンプの収集
- [カーネルライブパッチ](klp.md) — リブートそのものを回避する
- [電源管理: システムサスペンド](../power/suspend.md) — `CONFIG_KEXEC_JUMP`のpreserve-context経路がハイバネーションのコード経路から再利用している、`freeze_processes()`/`dpm_suspend_start()`によるデバイスサスペンドのシーケンス

### LWNの記事 {#lwn-articles}

- [kexec: A new system call to allow in kernel loading](https://lwn.net/Articles/574400/) — kernel-fd/initrd-fdの設計を持つ`kexec_file_load()`を導入した最初のRFC
- [Reworking kexec for signatures](https://lwn.net/Articles/603116/) — `kexec_file_load()`が存在する理由: kexecを署名済みカーネルに限定すること、およびSecure Bootとの関係

### 外部資料 {#external}

- [Kdump](https://docs.kernel.org/admin-guide/kdump/kdump.html) — kdumpの公式セットアップガイド。`crashkernel=`のサイズ設定と、キャプチャカーネルのロード時に使う`irqpoll`/`nr_cpus=1`/`reset_devices`ブートオプションを扱う
