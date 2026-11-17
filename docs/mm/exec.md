# exec()の間に何が起きるか {#what-happens-during-exec}

> execve()システムコールから、新しいプロセスが実行されるまで

## 概要 {#overview}

`execve()` は現在のプロセスイメージを新しいものに置き換える。PIDは変わらないが、アドレス空間、ファイルディスクリプタ（`O_CLOEXEC` でないもの）、レジスタは置き換えられる。

```
execve("./myprogram", argv, envp)
   │
   ├─ syscall entry → do_execveat_common()
   ├─ bprm allocation (struct linux_binprm)
   ├─ bprm_init: open + read file header
   ├─ search_binary_handler: find ELF/script handler
   ├─ load_elf_binary:
   │     ├─ parse ELF headers, validate
   │     ├─ flush old address space (exec_mmap)
   │     ├─ map PT_LOAD segments → VMAs
   │     ├─ map interpreter (ld.so) → VMAs
   │     ├─ setup stack (argv, envp, auxv)
   │     └─ set entry point: interpreter start or ELF e_entry
   └─ start_thread: set IP = entry, SP = stack top
```

## linux_binprm構造体 {#the-linux_binprm-structure}

```c
/* include/linux/binfmts.h */
struct linux_binprm {
    struct vm_area_struct *vma;
    unsigned long         vma_pages;
    struct mm_struct *mm;

    /* argv[0]のパス。/proc/PID/exeに使われる */
    const char *filename;
    const char *interp;

    /* ファイルの先頭BINPRM_BUF_SIZEバイト */
    char buf[BINPRM_BUF_SIZE];  /* 256バイト */

    struct file *file;
    struct cred *cred;          /* 提案された新しいクレデンシャル */

    int argc, envc;

    /* 新しいmmにおけるスタックの先頭アドレス */
    unsigned long p;

    /* argv[]とenvp[]はpより上のスタックに格納される */
    unsigned long argmin;

    /* ELF固有: 実行形式の種別フラグ */
    unsigned int per_flags;
    unsigned int unsafe;
};
```

## ステップ1: システムコールのエントリ {#step-1-syscall-entry}

```c
/* fs/exec.c */
SYSCALL_DEFINE3(execve,
    const char __user *, filename,
    const char __user *const __user *, argv,
    const char __user *const __user *, envp)
{
    return do_execve(getname(filename), argv, envp);
}

static int do_execveat_common(int fd, struct filename *filename,
                               struct user_arg_ptr argv,
                               struct user_arg_ptr envp,
                               int flags)
{
    struct linux_binprm *bprm;

    bprm = alloc_bprm(fd, filename, flags);

    /* argv/envpの文字列をbprmにコピーする（長さの上限をチェック） */
    retval = copy_strings_kernel(1, &bprm->filename, bprm);
    retval = copy_strings(bprm->envc, envp, bprm);
    retval = copy_strings(bprm->argc, argv, bprm);

    /* ファイルの先頭256バイトを読む（マジックナンバーのチェック） */
    retval = bprm_execve(bprm, fd, filename, flags);
}
```

## ステップ2: バイナリ形式の検出 {#step-2-binary-format-detection}

```c
/* fs/exec.c */
static int search_binary_handler(struct linux_binprm *bprm)
{
    /* 登録済みバイナリハンドラのリストを走査する */
    list_for_each_entry(fmt, &formats, lh) {
        retval = fmt->load_binary(bprm);
        if (retval != -ENOEXEC)
            break;
    }
}

/* 登録済みハンドラ（順番どおり）: */
/* 1. ELF:       fs/binfmt_elf.c    — "\x7fELF" にマッチ */
/* 2. スクリプト: fs/binfmt_script.c — "#!" にマッチ       */
/* 3. Misc:      fs/binfmt_misc.c   — ユーザー定義のルール */
/* 4. Flat:      fs/binfmt_flat.c   — uClinuxのflatバイナリ */
```

`#!/bin/sh` スクリプトの場合、`binfmt_script` は `/bin/sh` を新しい実行ファイルとして `search_binary_handler` を再度呼び出す。

## ステップ3: ELFのロード {#step-3-elf-loading}

```c
/* fs/binfmt_elf.c */
static int load_elf_binary(struct linux_binprm *bprm)
{
    struct elf_phdr *elf_phdata;
    struct elf_ehdr *elf_ex = (struct elf_ehdr *)bprm->buf;

    /* 1. ELFマジックとアーキテクチャを検証する */
    if (memcmp(elf_ex->e_ident, ELFMAG, SELFMAG) != 0)
        return -ENOEXEC;

    /* 2. すべてのプログラムヘッダを読む */
    elf_phdata = load_elf_phdrs(elf_ex, bprm->file);

    /* 3. インタプリタを探す（PT_INTERP = 動的リンカのパス） */
    for (i = 0; i < elf_ex->e_phnum; i++) {
        if (elf_phdata[i].p_type == PT_INTERP) {
            /* "/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2" を読む */
            elf_interpreter = kmalloc(elf_phdata[i].p_filesz, GFP_KERNEL);
            kernel_read(bprm->file, elf_interpreter, ...);
        }
    }

    /* 4. 古いアドレス空間を破棄し、新しいmmを作る */
    retval = begin_new_exec(bprm);
    /* → exec_mmap: 現在のmmを新しいmm_structに置き換える */
    /* → O_CLOEXECのfdを閉じる */
    /* → シグナルハンドラをリセットする */

    /* 5. PT_LOADセグメントをマップする */
    for each PT_LOAD segment {
        elf_map(bprm->file, load_addr + vaddr,
                eppnt, elf_prot, elf_flags, total_size);
        /* セグメントごとにVMAを作る:
             .text: PROT_READ|EXEC, MAP_PRIVATE|FIXED
             .data: PROT_READ|WRITE, MAP_PRIVATE|FIXED  */
    }

    /* 6. インタプリタ（ld.so）をランダムなアドレスにロードする */
    if (elf_interpreter) {
        interp_load_addr = load_elf_interp(&interp_elf_ex, interpreter);
        elf_entry = interp_load_addr + interp_elf_ex.e_entry;
        /* プロセスはバイナリのe_entryではなくld.soから開始する */
    } else {
        elf_entry = elf_ex->e_entry;  /* 静的リンクの場合 */
    }

    /* 7. スタックをセットアップする */
    retval = setup_arg_pages(bprm, randomize_stack_top(STACK_TOP),
                              executable_stack);
    /* スタックのVMAを作る: PROT_READ|WRITE、下方向に伸びる */

    /* 8. [vvar]と[vdso]のマッピングを作る */
    arch_setup_additional_pages(bprm, !!elf_interpreter);

    /* 9. 補助ベクタをスタックに積む */
    /* auxvのエントリ: AT_PHDR, AT_PHNUM, AT_ENTRY, AT_BASE,
                     AT_PAGESZ, AT_RANDOM, AT_PLATFORM, ... */

    /* 10. 新しいバイナリを開始する */
    start_thread(regs, elf_entry, bprm->p);
    /* IP=elf_entry、SP=bprm->p（argv/envp/auxvを含むスタックの先頭）に設定する */
}
```

## 結果として得られるアドレス空間 {#the-resulting-address-space}

execが完了した後、`/proc/<pid>/maps` は次のようになる。

```
Address           Perms  Name
00400000-00401000 r--p   /usr/bin/myprogram    [ELF .text]
00401000-00402000 r-xp   /usr/bin/myprogram    [ELF .text]
00402000-00403000 r--p   /usr/bin/myprogram    [ELF .rodata]
00403000-00404000 r--p   /usr/bin/myprogram    [.data start]
00404000-00405000 rw-p   /usr/bin/myprogram    [ELF .data/.bss]
7f1234560000-...  r--p   /lib/.../ld-linux.so  [ld.so segments]
7f1234590000-...  r-xp   /lib/.../ld-linux.so
7ffcc1234000-...  rw-p   [stack]
7ffcc1400000-...  r--p   [vvar]
7ffcc1401000-...  r-xp   [vdso]
```

## 動的リンカの起動 {#dynamic-linker-startup}

カーネルはメインバイナリではなく `ld.so` のエントリポイントにジャンプする。`ld.so` は次のことを行う。
1. 自分自身をマップする（位置独立である）
2. バイナリの `PT_DYNAMIC` セグメントを読み、共有ライブラリの依存関係を見つける
3. 各 `.so` を `mmap()` でロードする（新しいVMAを作る）
4. PLT/GOTの再配置（シンボルアドレス）を解決する
5. `DT_INIT` のコンストラクタ（例: `__attribute__((constructor))` 関数）を呼び出す
6. メインバイナリの `e_entry`（`_start` シンボル）にジャンプする

```c
/* ld.soはライブラリをプロセスのアドレス空間にロードする */
/* ld.soの処理が終わった後: */
00400000-...  r-xp   /usr/bin/myprogram
7f0000000000-...  r-xp   /lib/x86_64-linux-gnu/libc.so.6
7f0001000000-...  r-xp   /lib/x86_64-linux-gnu/libm.so.6
/* 各共有ライブラリはそれぞれ独自のVMA群を持つ */
```

## begin_new_exec: 引き返せない地点 {#beginnewexec-the-point-of-no-return}

```c
/* fs/exec.c */
int begin_new_exec(struct linux_binprm *bprm)
{
    struct task_struct *me = current;

    /* 新しいmm_structを作る（空のアドレス空間） */
    retval = exec_mmap(bprm->mm);
    /* これはcurrent->mmを置き換える — 古いマッピングは消える！ */

    /* O_CLOEXECのファイルディスクリプタを閉じる */
    do_close_on_exec(current->files);

    /* シグナルハンドラをSIG_DFLにリセットする */
    flush_signal_handlers(me, 0);

    /* set-uidによって獲得したケーパビリティを落とす */
    /* set-uid/set-gidバイナリならクレデンシャルを更新する */
    retval = install_exec_creds(bprm);

    /* プロセス名を変更する（/proc/PID/comm） */
    __set_task_comm(me, kbasename(bprm->filename), true);

    return 0;
}
```

## execとスレッド {#exec-and-threads}

呼び出したプロセスに複数のスレッドがある場合、`execve` はアドレス空間を置き換える前に他のすべてのスレッドを終了させる。

```c
/* fs/exec.c */
static int de_thread(struct task_struct *tsk)
{
    struct signal_struct *sig = tsk->signal;
    struct sighand_struct *oldsighand = tsk->sighand;

    /* スレッドグループの共有を解除する */
    /* 他のすべてのスレッドに終了をシグナルで通知する */
    zap_other_threads(tsk);

    /* 他のすべてのスレッドが終了するのを待つ */
    while (atomic_read(&sig->count) > 1)
        schedule_timeout(1);
}
```

## execの観測 {#observing-exec}

```bash
# exec システムコールをトレースする
strace -e execve /bin/ls

# システム全体のすべての exec をトレースする
bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%s exec %s\n", comm, str(args->filename)); }'

# ELF セグメントを表示する
readelf -l /usr/bin/ls | grep -A1 LOAD

# 動的リンカのロード処理を表示する
LD_DEBUG=all /usr/bin/ls 2>&1 | head -50

# exec 後のアドレス空間全体を表示する
cat /proc/$(pgrep myprogram)/maps
```

## 参考文献 {#further-reading}

- [fork()したときに何が起きるか](fork.md) — forkが出発点となるmmを作る
- [プロセスのアドレス空間](mmap.md) — VMAのレイアウト
- [mallocの一生](life-of-malloc.md) — exec後のヒープ
- [Copy-on-Write](cow.md) — ELFのデータセグメントはCOWを使う
- `fs/exec.c` — execveの実装
- `fs/binfmt_elf.c` — ELFローダ
