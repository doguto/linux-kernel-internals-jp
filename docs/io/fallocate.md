# fallocateとファイル空間の管理 {#fallocate-and-file-space-management}

> データを書かずにディスクブロックを予約する: ENOSPC、断片化、スパースファイルの思わぬ挙動を避ける

## `write()`ベースの事前割り当ての問題 {#the-problem-with-write-based-preallocation}

ファイルを埋める前にその領域を保証する伝統的な方法は、ゼロを書き込むことである。

```c
/* 古いパターン: ゼロを書き込んで 1 GiB を事前割り当てする */
char zeros[4096] = {0};
for (size_t i = 0; i < (1UL << 30) / sizeof(zeros); i++) {
    write(fd, zeros, sizeof(zeros));
}
```

これには3つの深刻な問題がある。

**遅い。** `write()`でダーティになったすべてのpageは、ページキャッシュを通じてフラッシュされ、最終的にディスクに書き込まれなければならない。データに意味がないにもかかわらず、1 GiBのゼロを書き込むには実際のI/O時間がかかる。

**書き込み途中のENOSPCを防げない。** 領域は、書き込みが届くたびに1 pageずつしか予約されない。ファイルシステムを埋めている並行プロセスがあると、ループの途中で`ENOSPC`が発生しうる。その結果、部分的に書かれたファイルが残り、呼び出し側はどう回復するかを決めなければならなくなる。

**断片化を引き起こす。** ファイルシステムは書き込みが届くたびにブロックを割り当て、それらを他の並行する割り当てと交互に配置する。大きなシーケンシャルファイルが、ディスク上に散らばった数百の小さなエクステントになってしまうことがある。

`fallocate(2)`はこの3つすべてを解決する。カーネルはファイルシステムに働きかけ、1回の操作で連続したエクステントを予約し、それらを割り当て済みだが未書き込みとしてマークする。I/Oは発生しない。ファイルサイズは予約を反映する（モードによっては反映しない）。後の`write()`はすでに割り当て済みのブロックを埋めるので、その範囲について`ENOSPC`で失敗することはもうない。

```
write()-based preallocation:
  write(zeros) → page cache → writeback → allocate block → ...  (N writes, N allocations)

fallocate(mode=0):
  fallocate() → filesystem allocates extents → done  (1 syscall, contiguous extent)
  write(data) → fills already-allocated blocks (no allocation at write time)
```

## `fallocate()`システムコール {#fallocate-system-call}

```c
#include <fcntl.h>

int fallocate(int fd, int mode, off_t offset, off_t len);
```

- `fd` — オープン済みのファイルディスクリプタ（書き込み可能でなければならない）
- `mode` — デフォルトの割り当てならゼロ、または`FALLOC_FL_*`フラグのビットマスク
- `offset` — 操作を開始するバイトオフセット
- `len` — 操作対象の領域のバイト長

成功時は0、エラー時は-1を返す。エラーには次のものがある。

| エラー | 意味 |
|-------|---------|
| `ENOSPC` | 要求を満たすのに十分な領域がない |
| `EOPNOTSUPP` | ファイルシステムまたはモードがサポートされていない |
| `EINVAL` | 不正なoffset/len、または互換性のないフラグ |
| `EBADF` | `fd`が書き込み用に開かれていない |
| `ESPIPE` | `fd`がパイプまたはFIFO |

フラグ定数は`include/uapi/linux/falloc.h`にあり、VFSのディスパッチは`fs/fallocate.c`にある。

## モードの概要 {#modes-overview}

| モード | 定数 | 効果 |
|------|----------|--------|
| デフォルト | `0` | ブロックを割り当てる。必要ならファイルサイズを拡張する。ゼロで埋める |
| サイズ維持 | `FALLOC_FL_KEEP_SIZE` | ブロックを割り当てる。ファイルサイズは拡張 **しない** |
| ホールパンチ | `FALLOC_FL_PUNCH_HOLE \| FALLOC_FL_KEEP_SIZE` | ブロックの割り当てを解除する。読み出しはゼロを返す |
| 範囲の削除 | `FALLOC_FL_COLLAPSE_RANGE` | ファイルの途中からバイトを取り除く |
| 範囲のゼロ化 | `FALLOC_FL_ZERO_RANGE` | 割り当てを取り除かずに範囲をゼロにする |
| 範囲の挿入 | `FALLOC_FL_INSERT_RANGE` | 空の領域を挿入し、後続のデータをずらす |
| 共有の解除 | `FALLOC_FL_UNSHARE_RANGE` | reflinkされたエクステントのコピーオンライト共有を解消する |

`FALLOC_FL_PUNCH_HOLE`は常に`FALLOC_FL_KEEP_SIZE`と組み合わせなければならない。カーネルは`FALLOC_FL_PUNCH_HOLE`単独の指定を`EINVAL`で拒否する。

## デフォルトの割り当て（mode = 0） {#default-allocation-mode-0}

```c
/* 512 MiB のファイルを事前割り当てする */
int fd = open("data.bin", O_RDWR | O_CREAT, 0644);
if (fallocate(fd, 0, 0, 512UL << 20) == -1)
    perror("fallocate");
```

この呼び出しの後は次のようになる。

- `stat(2).st_size`は512 MiBに等しい — ファイルサイズが拡張される。
- `stat(2).st_blocks`は512 MiBに比例する — 物理ブロックが割り当てられる。
- 未書き込みの領域を読むとゼロが返るが、I/Oは必要ない。ファイルシステムはエクステントを「割り当て済みだが未書き込み」としてマークしている。
- その範囲への後続の`write()`呼び出しは新しいブロックを割り当てる必要がなく、その範囲について`ENOSPC`で失敗することはない。

### `st_size`と`st_blocks`への影響 {#effect-on-stsize-vs-stblocks}

```
Before fallocate:
  st_size   = 0
  st_blocks = 0

After fallocate(fd, 0, 0, 512MiB):
  st_size   = 536870912   (512 * 1024 * 1024)
  st_blocks = 1048576     (in 512-byte units: 512MiB / 512)

After fallocate(fd, FALLOC_FL_KEEP_SIZE, 0, 512MiB):
  st_size   = 0           (unchanged!)
  st_blocks = 1048576     (blocks still allocated)
```

### スパースファイルの見分け方: `du`と`ls` {#sparse-file-detection-du-vs-ls}

`ls -l`は`st_size`（論理サイズ）を報告する。`du`は`st_blocks`（物理的な割り当て）を報告する。スパースファイルは`ls -l`では大きなサイズを示すが、`du`の結果は小さい。

```bash
# 1 GiB のスパースファイルを作成する（ftruncate、ブロックなし）
truncate -s 1G sparse.bin
ls -lh sparse.bin          # 1.0G
du -sh sparse.bin          # 4.0K  （inode のみ）

# 1 GiB を事前割り当てする（fallocate、実際のブロック）
fallocate -l 1G preallocated.bin
ls -lh preallocated.bin    # 1.0G
du -sh preallocated.bin    # 1.1G  （実際のディスク使用量）
```

### ファイルシステム固有の挙動 {#filesystem-specific-behavior}

**ext4**: エクステントの事前割り当て（`fs/ext4/extents.c`の`ext4_fallocate`）を使う。未書き込み（unwritten）エクステントを作成し、エクステントツリーはその範囲を`EXT_UNWRITTEN`としてマークする。未書き込みエクステントからの読み出しは、ディスクI/Oなしにゼロを返す。ブロックへの最初の書き込みで、エクステントは書き込み済みに変換される。

**XFS**: `xfs_alloc_file_space`を呼び、B-tree内に未書き込みエクステントを作成する。XFSは、`XFS_IOC_ALLOCSP`を通じて`fallocate`方式の事前割り当てを実装した最初のLinuxファイルシステムである。

**btrfs**: `fallocate`をサポートするが、コピーオンライトのセマンティクスのため、時間とともにより多くの断片化を生じることがある。事前割り当てされたブロックを上書きすると、その場で更新するのではなく新しいエクステントが作られる（ファイルに`nodatacow`が設定されていない限り）。

**tmpfs**: メモリ上のファイルに対して`fallocate`をサポートする。ブロックはDRAMのpageである。ホールパンチもサポートされる。

## `FALLOC_FL_KEEP_SIZE`: 拡張せずに事前割り当てする {#fallocflkeep_size-preallocate-without-extending}

このモードは物理ブロックを割り当てるが、`st_size`は変更しない。これは先行書き込みログ（WAL）の事前割り当てにおける定番のパターンである。

```c
/*
 * PostgreSQL 方式の WAL セグメントの事前割り当て。
 * ファイルは空に見える（st_size=0）が、ブロックはすでに予約されている。
 * WAL の書き込み側が write() を呼ぶと、事前割り当てされたブロックが埋まる。
 */
int preallocate_wal_segment(const char *path, size_t segment_size)
{
    int fd = open(path, O_RDWR | O_CREAT, 0600);
    if (fd < 0)
        return -1;

    /* ブロックを予約する。ファイルサイズは 0 のまま */
    if (fallocate(fd, FALLOC_FL_KEEP_SIZE, 0, segment_size) == -1) {
        /* fallocate が未サポートならゼロの書き込みにフォールバックする */
        if (errno == EOPNOTSUPP)
            return write_zeros(fd, segment_size);
        close(fd);
        return -1;
    }
    close(fd);
    return 0;
}
```

なぜサイズをゼロのままにするのか。消費側のプロセス（WALを適用する側）は、WALの書き込み側がそう告げるまで、ファイルにデータがあると見てはならないからである。`st_size`は「ここにどれだけのデータがあるか」を示す正式なシグナルであり、`st_blocks`はストレージ層の詳細にすぎない。

## `FALLOC_FL_PUNCH_HOLE`: ファイル内の領域を回収する {#fallocflpunch_hole-reclaiming-space-within-a-file}

ホールパンチは、`st_size`を変えずにファイルの途中のブロックの割り当てを解除する。パンチされた領域はスパースなホールになる。読み出しはゼロを返すが、ディスクブロックは消費されない。

```c
#include <linux/falloc.h>

/*
 * バイト 'offset' から 'len' バイトのホールをパンチする。
 * FALLOC_FL_PUNCH_HOLE には FALLOC_FL_KEEP_SIZE が必須である。
 */
int punch_hole(int fd, off_t offset, off_t len)
{
    return fallocate(fd, FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE,
                     offset, len);
}
```

### 用途 {#use-cases}

**ログローテーション**: ログファイルをunlinkして作り直す（他のプロセスが保持しているオープン中のファイルディスクリプタを壊してしまう）代わりに、消費済みの領域をパンチする。

```c
/* プロセスはバイト [0, consumed_offset) を読み終えた。その領域を回収する。 */
punch_hole(log_fd, 0, consumed_offset);
/* st_size は変わらない。読み出し側には元のサイズのままのファイルが見える */
```

**データベースの空きリスト管理**: 独自の空きリストを管理するデータベース（PostgreSQLの、deadとマークされたヒープpageなど）は、解放されたpageの範囲にホールをパンチし、ファイルを切り詰めることなくブロックをファイルシステムへ返すことができる。

**大きなメディアファイル**: 動画編集ツールは、削除されたフレームの範囲にホールをパンチし、ファイル構造をそのまま保ちながら領域を回収できる。

### `SEEK_HOLE` / `SEEK_DATA`によるホールの走査 {#iterating-holes-with-seekhole-seekdata}

ホールをパンチした後は、`SEEK_HOLE`と`SEEK_DATA`を指定した`lseek(2)`を使って、データ領域とスパース領域の境界を見つける。

```c
#include <unistd.h>
#include <stdio.h>

/*
 * ファイル内のすべてのデータとホールのエクステントを表示する。
 * SEEK_DATA: offset 以降の次のデータ領域へシークする。
 * SEEK_HOLE: offset 以降の次のホールへシークする。
 * ファイルは常に st_size の位置にある仮想的なホールで終わる。
 */
void print_extents(int fd, off_t file_size)
{
    off_t pos = 0;

    while (pos < file_size) {
        off_t data_start = lseek(fd, pos, SEEK_DATA);
        if (data_start == -1)
            break;  /* もうデータはない: 残りはホール */

        off_t hole_start = lseek(fd, data_start, SEEK_HOLE);
        if (hole_start == -1)
            hole_start = file_size;  /* データは末尾まで続く */

        printf("  data: [%lld, %lld)  (%lld bytes)\n",
               (long long)data_start,
               (long long)hole_start,
               (long long)(hole_start - data_start));

        /* ホールがどこで終わるかを見つける */
        off_t next_data = lseek(fd, hole_start, SEEK_DATA);
        if (next_data == -1)
            next_data = file_size;

        if (hole_start < next_data)
            printf("  hole: [%lld, %lld)  (%lld bytes)\n",
                   (long long)hole_start,
                   (long long)next_data,
                   (long long)(next_data - hole_start));

        pos = next_data;
    }
}
```

`SEEK_HOLE`と`SEEK_DATA`はPOSIX 2013で標準化され、ext4、XFS、btrfs、tmpfsでサポートされている。これらをサポートしないファイルシステムでは、`SEEK_HOLE`は`st_size`を返す（ファイル全体がデータであるように見える）。

## `FALLOC_FL_ZERO_RANGE`: パンチせずにゼロ化する {#fallocflzero_range-zeroing-without-punching}

`FALLOC_FL_ZERO_RANGE`は、ファイルの領域を効率的にゼロにする。ホールパンチと異なり、ブロックは割り当てられたままである。`write()`でゼロを書き込む場合と異なり、領域がブロック境界にアラインされていれば、ファイルシステムはこれをメタデータのみの操作として実装できる。

```c
/* ブロックの割り当てを解除せずにバイト [offset, offset+len) をゼロにする */
fallocate(fd, FALLOC_FL_ZERO_RANGE, offset, len);

/* 必要ならファイルサイズの拡張を避ける（KEEP_SIZE と同じ挙動） */
fallocate(fd, FALLOC_FL_ZERO_RANGE | FALLOC_FL_KEEP_SIZE, offset, len);
```

ext4とXFSでは、アラインされたゼロ範囲の要求は、エクステントツリー内の書き込み済みエクステントを未書き込みエクステントに変換する。これは新しく事前割り当てされたブロックと同じ表現である。実際のゼロバイトがディスクに書かれることはなく、ファイルシステムの読み出し経路がそれを合成する。アラインされていない端の部分については、部分ブロックが読み込まれ、ゼロにされ、書き戻される。

## `FALLOC_FL_COLLAPSE_RANGE`: ファイルの一部を取り除く {#fallocflcollapse_range-removing-a-file-section}

`FALLOC_FL_COLLAPSE_RANGE`は、ファイルの`offset`から始まる`len`バイトを取り除き、後続のデータをすべて前方へずらす。ホールは残らず、ファイルは`len`バイト縮む。`offset`と`len`はともにファイルシステムのブロックサイズの倍数でなければならない。

```c
/* オフセット 4 MiB から 1 MiB を取り除き、残りを前へずらす */
fallocate(fd, FALLOC_FL_COLLAPSE_RANGE, 4UL << 20, 1UL << 20);
```

これはインプレースでのログの圧縮に役立つ。残りのデータを新しいファイルにコピーする代わりに、消費済みの先頭部分を取り除けばよい。ext4（3.15以降）とXFS（3.15以降）でサポートされている。btrfsはサポートしていない。

## `FALLOC_FL_INSERT_RANGE`: 領域を挿入する {#fallocflinsert_range-inserting-space}

collapseの逆で、`FALLOC_FL_INSERT_RANGE`は`offset`に`len`バイトのゼロを挿入し、後続のデータを後方へずらす。ファイルは`len`だけ大きくなる。どちらの値もブロックサイズにアラインされていなければならない。

```c
/* オフセット 2 MiB に 512 KiB を挿入する */
fallocate(fd, FALLOC_FL_INSERT_RANGE, 2UL << 20, 512UL << 10);
```

ext4（4.1以降）とXFS（4.1以降）でサポートされている。実際に使われることはまれである。

## `FALLOC_FL_UNSHARE_RANGE`: reflinkの共有を解消する {#fallocflunshare_range-breaking-reflink-sharing}

reflinkをサポートするコピーオンライト型ファイルシステム（btrfs、XFS）では、`cp --reflink`は元のファイルとエクステントツリーのエントリを共有するファイルを作成する。どちらかのファイルへの書き込みは、新しいエクステントへのコピーオンライトを引き起こす。

`FALLOC_FL_UNSHARE_RANGE`は、カーネルに共有データを物理的にコピーさせ、共有関係を解消する。

```c
struct stat st;
fstat(fd, &st);

/* すべての共有エクステントを強制的にプライベートなコピーにする */
fallocate(fd, FALLOC_FL_UNSHARE_RANGE, 0, st.st_size);
```

これは、reflinkされたファイルに激しいランダム書き込みを行う前に役立つ。共有を解除しなければ、すべての書き込みがCOWによる割り当てを引き起こし、エクステントが散らばる可能性がある。共有を解除した後は、書き込みはインプレースかつ連続的になる（ファイルがすでに断片化していなければ）。

!!! note "ファイルシステムによる実装の違い"
    XFSは、エクステントのメタデータを直接操作する、ネイティブで効率的なカーネルレベルのunshare実装を持っている。btrfsはこのフラグに対するネイティブなカーネルレベルのunshare操作を持たず、読み出し・再書き込みの経路にフォールバックする。これは効率が悪く、ファイルサイズに比例したI/Oを生む。

より広い文脈については、後述の[コピーオンライト型ファイルシステム](#copy-on-write-filesystems-btrfs-and-xfs)を参照のこと。

## `posix_fallocate()`: POSIXのフォールバック {#posix_fallocate-the-posix-fallback}

```c
#include <fcntl.h>

int posix_fallocate(int fd, off_t offset, off_t len);
```

`posix_fallocate()`は、領域の予約を保証するlibcの関数である（直接のシステムコールではない）。カーネルが`fallocate(fd, 0, offset, len)`をサポートしていれば、glibcはそれを直接呼ぶ。サポートしていなければ（NFS、CIFS、古いカーネルなど）、glibcはゼロの書き込みにフォールバックする。

```c
/* glibc sysdeps/unix/sysv/linux/posix_fallocate.c（簡略化） */
int posix_fallocate(int fd, off_t offset, off_t len)
{
    int ret = fallocate(fd, 0, offset, len);
    if (ret == 0)
        return 0;
    if (errno != EOPNOTSUPP)
        return errno;

    /* フォールバック: 低速な方法でゼロを書き込む */
    return write_zeros_range(fd, offset, len);
}
```

このフォールバックは正しいが遅く、書き込み途中のENOSPCの可能性も依然として残る。性能を重視する場合は、`fallocate(2)`を直接呼び、`EOPNOTSUPP`を自分で処理するほうがよい。

## `ftruncate()`と`fallocate()` {#ftruncate-vs-fallocate}

`ftruncate(fd, new_size)`と`fallocate(fd, 0, 0, new_size)`はどちらもファイルを`new_size`まで拡張するが、行うことは根本的に異なる。

| 観点 | `ftruncate(fd, size)` | `fallocate(fd, 0, 0, size)` |
|--------|----------------------|----------------------------|
| ブロックの割り当て | なし（スパース） | あり（実ブロック） |
| `st_size` | `size`に設定 | `size`に設定 |
| `st_blocks` | 変化なし（ほぼ0） | `size`に比例 |
| 後の書き込みでENOSPCになりうるか | なりうる | ならない（ブロックは予約済み） |
| 断片化のリスク | 拡張時にはなし | なし（エクステントは連続） |
| ディスク使用量 | ほぼゼロ | `size`全体 |
| 速度 | 即時（メタデータのみ） | `size`に比例 |

```c
/* ftruncate: 1 GiB のスパースファイルを作成する */
ftruncate(fd, 1UL << 30);
/* st_size=1GiB、st_blocks≈8（inode のみ）、ディスク使用量: 約 4KiB */

/* fallocate: 1 GiB の事前割り当て済みファイルを作成する */
fallocate(fd, 0, 0, 1UL << 30);
/* st_size=1GiB、st_blocks=2097152、ディスク使用量: 約 1GiB */
```

縮小する場合、`ftruncate(fd, smaller_size)`と`fallocate(fd, 0, 0, smaller_size)`はどちらもファイルを切り詰める。切り詰めには`ftruncate`を使うこと。`fallocate`は現在のサイズより小さくする用途については文書化されておらず、挙動はファイルシステムに依存する。

## コード付きの実践的な用途 {#practical-use-cases-with-code}

### 1.ダウンロードファイルの領域予約 {#1-download-file-reservation}

ダウンロードの開始前にファイルサイズ全体を事前割り当てし、領域を保証して断片化を避ける。

```c
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>

int create_download_file(const char *path, off_t expected_size)
{
    int fd = open(path, O_RDWR | O_CREAT | O_TRUNC, 0644);
    if (fd < 0)
        return -1;

    if (fallocate(fd, 0, 0, expected_size) == -1) {
        if (errno == EOPNOTSUPP) {
            /*
             * ファイルシステムが fallocate をサポートしていない。
             * ftruncate にフォールバックする: スパースファイルが作られる。
             * 実際のダウンロード中に ENOSPC が起こりうる。
             */
            if (ftruncate(fd, expected_size) == -1) {
                close(fd);
                return -1;
            }
        } else {
            /* ENOSPC またはその他の本当のエラー */
            close(fd);
            return -1;
        }
    }

    return fd;  /* 呼び出し側は pwrite() でダウンロードデータを書き込む */
}
```

### 2. WAL/ジャーナルの事前割り当て（PostgreSQLのパターン） {#2-waljournal-preallocation-postgresql-pattern}

```c
#include <fcntl.h>
#include <linux/falloc.h>

/*
 * WAL セグメントファイルを事前割り当てする。
 * KEEP_SIZE を使うので、WAL の読み出し側にはファイルが空に見える。
 * 書き込み側はシーケンシャルな write() 呼び出しでそれを埋める。
 */
int preallocate_wal(const char *path, size_t wal_segment_size)
{
    int fd = open(path, O_RDWR | O_CREAT | O_EXCL, 0600);
    if (fd < 0)
        return -1;

    int ret = fallocate(fd, FALLOC_FL_KEEP_SIZE, 0, wal_segment_size);
    if (ret == -1 && errno != EOPNOTSUPP) {
        close(fd);
        return -1;
    }

    close(fd);
    return 0;
}
```

### 3.ホールパンチによるログファイルのローテーション {#3-log-file-rotation-with-hole-punching}

```c
#include <fcntl.h>
#include <linux/falloc.h>
#include <sys/stat.h>

/*
 * 消費側がログのバイト [0, consumed_up_to) を処理した後、
 * 消費済みの領域をパンチしてディスク領域を返却する。
 * ファイルディスクリプタは開いたままで、稼働中の書き込み側は影響を受けない。
 */
int reclaim_consumed_log(int fd, off_t consumed_up_to)
{
    struct stat st;
    if (fstat(fd, &st) == -1)
        return -1;

    /* ファイルシステムのブロックサイズにアラインする */
    long blksz = st.st_blksize;
    off_t aligned_end = (consumed_up_to / blksz) * blksz;

    if (aligned_end <= 0)
        return 0;   /* まだパンチするものはない */

    return fallocate(fd, FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE,
                     0, aligned_end);
}
```

### 4.スパースファイルの作成とホールの調査 {#4-sparse-file-creation-and-hole-inspection}

```c
#include <fcntl.h>
#include <linux/falloc.h>
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    const char *path = "sparse_demo.bin";
    const off_t size = 10UL << 20;  /* 10 MiB のファイル */

    /* 10 MiB の事前割り当て済みファイルを作成する */
    int fd = open(path, O_RDWR | O_CREAT | O_TRUNC, 0644);
    fallocate(fd, 0, 0, size);

    /* 1 MiB のブロック 2 つにデータを書く */
    char buf[1 << 20];
    memset(buf, 'A', sizeof(buf));
    pwrite(fd, buf, sizeof(buf), 0);           /* バイト [0, 1MiB) */
    pwrite(fd, buf, sizeof(buf), 4 << 20);     /* バイト [4MiB, 5MiB) */

    /* 不要な領域にホールをパンチする */
    fallocate(fd, FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE,
              1 << 20, 3 << 20);               /* [1MiB, 4MiB) をパンチ */
    fallocate(fd, FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE,
              5 << 20, 5 << 20);               /* [5MiB, 10MiB) をパンチ */

    /* SEEK_DATA / SEEK_HOLE でエクステントを走査する */
    printf("Extents in %s:\n", path);
    off_t pos = 0;
    while (pos < size) {
        off_t data = lseek(fd, pos, SEEK_DATA);
        if (data == -1) break;

        off_t hole = lseek(fd, data, SEEK_HOLE);
        if (hole == -1) hole = size;

        printf("  data [%lld, %lld)\n", (long long)data, (long long)hole);
        pos = lseek(fd, hole, SEEK_DATA);
        if (pos == -1) break;
    }

    close(fd);
    return 0;
}
/*
 * 出力:
 *   data [0, 1048576)
 *   data [4194304, 5242880)
 */
```

## ファイルシステムのサポート表 {#filesystem-support-matrix}

| モード | ext4 | XFS | btrfs | tmpfs | NFS | CIFS |
|------|------|-----|-------|-------|-----|------|
| `mode=0`（割り当て） | Yes | Yes | Yes | Yes | No* | No* |
| `FALLOC_FL_KEEP_SIZE` | Yes | Yes | Yes | Yes | No* | No* |
| `FALLOC_FL_PUNCH_HOLE` | Yes (3.0+) | Yes | Yes | Yes | 部分的† | 部分的 |
| `FALLOC_FL_COLLAPSE_RANGE` | Yes (3.15+) | Yes (3.15+) | No | No | No | No |
| `FALLOC_FL_ZERO_RANGE` | Yes (3.15+) | Yes | Yes | No | No | No |
| `FALLOC_FL_INSERT_RANGE` | Yes (4.1+) | Yes (4.1+) | No | No | No | No |
| `FALLOC_FL_UNSHARE_RANGE` | No | Yes | Yes | No | No | No |

\* NFS（`FALLOC_FL_PUNCH_HOLE`以外のモード）とCIFSは、サーバー側のファイルシステムによっては一部のモードをサポートすることがある。NFS越しの`fallocate`は通常クライアントから`EOPNOTSUPP`を返し、`posix_fallocate`はゼロの書き込みにフォールバックする。

† NFSv4.2クライアントは、サーバー側のファイルシステムもサポートしている場合、NFSの`DEALLOCATE`操作を介してホールパンチをサポートする。NFSv3、NFSv4.0、NFSv4.1は`FALLOC_FL_PUNCH_HOLE`をサポートせず、`EOPNOTSUPP`を返す。

常に`EOPNOTSUPP`を確認し、フォールバックを用意しておくこと。

```c
ret = fallocate(fd, FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE,
                offset, len);
if (ret == -1 && errno == EOPNOTSUPP) {
    /* ファイルシステムがホールパンチをサポートしていない。
     * 選択肢: ゼロを書く（領域は使われたまま）か、無視する。 */
}
```

## コピーオンライト型ファイルシステム: btrfsとXFS {#copy-on-write-filesystems-btrfs-and-xfs}

### reflinkと`cp --reflink` {#reflinks-and-cp-reflink}

btrfsとXFSは *reflink* をサポートする。これは元のファイルとエクステントツリーのエントリを共有するファイルのコピーである。`cp --reflink`はほぼ瞬時に完了し、どちらかのファイルが変更されるまで追加のディスク領域を使わない。

```bash
# 1 GiB のファイルを作成し、reflink コピーする（瞬時、追加領域なし）
fallocate -l 1G original.bin
cp --reflink=always original.bin copy.bin
du -sh original.bin copy.bin   # どちらも約 1GiB と表示されるが、ブロックは共有されている
```

reflinkされたブロックに書き込まれると、ファイルシステムはコピーオンライトを行う。書き込んだプロセスは新しいプライベートなエクステントを得て、もう一方のファイルのエクステントは変わらない。

### `copy_file_range`とカーネル側のreflink {#copyfilerange-and-kernel-side-reflinks}

`copy_file_range(2)`は、ファイル間で範囲をコピーするようカーネルに依頼する。reflinkに対応したファイルシステムでは、カーネルはバイトをコピーする代わりに、メタデータのみのクローン操作としてこれを満たすことがある。

```c
#include <sys/sendfile.h>
/* 実際には: */
#include <unistd.h>

ssize_t copy_file_range(int fd_in, loff_t *off_in,
                         int fd_out, loff_t *off_out,
                         size_t len, unsigned int flags);
```

ブロックサイズが一致するXFSやbtrfs上でブロックにアラインされた範囲の場合、これはreflinkクローンになる。コストはO(エクステント数)であり、単一エクステントのファイルでは実質的に一定だが、ひどく断片化したファイルではエクステント数に応じて増える。それ以外の場合は読み書きのループにフォールバックする。

### COWによる時間経過での断片化 {#fragmentation-over-time-with-cow}

COWファイルシステムでは、頻繁に上書きされるファイルに断片化が蓄積する。上書きのたびに新しいエクステントが作られ、エクステントツリーが大きくなり、シーケンシャルな読み出しがランダムI/Oになる。`FALLOC_FL_UNSHARE_RANGE`はこれに対抗する手段の1つで、共有エクステントを単一の連続したプライベートな割り当てに統合する。

btrfs上のデータベースについては、データベースファイルのデータCOWを無効にするのが標準的な推奨である。

```bash
# ディレクトリで COW を無効にする（新しいファイルはフラグを継承する）
chattr +C /var/lib/postgresql/

# または既存のファイルに対して（空でなければならない）
chattr +C database.db
```

`+C`（nodatacow）を指定すると、`fallocate`と`write`はext4上と同様に振る舞うようになる。すなわち、ブロックごとのコピーオンライトなしのインプレース更新である。

### エクステント共有の検出 {#detecting-extent-sharing}

XFSとbtrfsでは、`FIEMAP` ioctlの`FIEMAP_EXTENT_SHARED`フラグが共有エクステントを示す。

```c
#include <linux/fiemap.h>
#include <linux/fs.h>
#include <sys/ioctl.h>

/* ファイル内に共有された（reflink された）エクステントがあるか確認する */
int has_shared_extents(int fd)
{
    struct {
        struct fiemap fm;
        struct fiemap_extent fe[64];
    } buf;

    memset(&buf, 0, sizeof(buf));
    buf.fm.fm_length = FIEMAP_MAX_OFFSET;
    buf.fm.fm_extent_count = 64;
    buf.fm.fm_flags = FIEMAP_FLAG_SYNC;

    if (ioctl(fd, FS_IOC_FIEMAP, &buf.fm) == -1)
        return -1;

    for (uint32_t i = 0; i < buf.fm.fm_mapped_extents; i++) {
        if (buf.fm.fm_extents[i].fe_flags & FIEMAP_EXTENT_SHARED)
            return 1;
    }
    return 0;
}
```

## カーネル内部 {#kernel-internals}

`fallocate`のVFSエントリポイントは、`fs/fallocate.c`の`vfs_fallocate()`である。

```c
/* fs/fallocate.c */
int vfs_fallocate(struct file *file, int mode, loff_t offset, loff_t len)
{
    struct inode *inode = file_inode(file);

    /* 基本的な妥当性チェック */
    if (offset < 0 || len <= 0)
        return -EINVAL;
    if (mode & ~(FALLOC_FL_KEEP_SIZE | FALLOC_FL_PUNCH_HOLE |
                 FALLOC_FL_COLLAPSE_RANGE | FALLOC_FL_ZERO_RANGE |
                 FALLOC_FL_INSERT_RANGE | FALLOC_FL_UNSHARE_RANGE))
        return -EOPNOTSUPP;

    /* ファイルシステムが fallocate を実装していなければならない */
    if (!file->f_op->fallocate)
        return -EOPNOTSUPP;

    /* セキュリティとクォータのチェック */
    ret = security_file_permission(file, MAY_WRITE);
    if (ret)
        return ret;

    return file->f_op->fallocate(file, mode, offset, len);
}
```

各ファイルシステムは独自の`fallocate`操作を提供する。ext4の場合は次のとおりである。

```c
/* fs/ext4/extents.c（簡略化） */
long ext4_fallocate(struct file *file, int mode, loff_t offset, loff_t len)
{
    struct inode *inode = file_inode(file);

    if (mode & FALLOC_FL_PUNCH_HOLE)
        return ext4_punch_hole(file, offset, len);
    if (mode & FALLOC_FL_COLLAPSE_RANGE)
        return ext4_collapse_range(inode, offset, len);
    if (mode & FALLOC_FL_ZERO_RANGE)
        return ext4_zero_range(file, offset, len, mode);
    if (mode & FALLOC_FL_INSERT_RANGE)
        return ext4_insert_range(inode, offset, len);

    /* デフォルト: 未書き込みエクステントを割り当てる */
    return ext4_alloc_file_blocks(file, lblk, max_blocks,
                                   new_size, flags);
}
```

フラグ定数の定義は`include/uapi/linux/falloc.h`にある。

```c
/* include/uapi/linux/falloc.h */
#define FALLOC_FL_KEEP_SIZE      0x01
#define FALLOC_FL_PUNCH_HOLE     0x02
#define FALLOC_FL_NO_HIDE_STALE  0x04  /* 内部用、ユーザー空間向けではない */
#define FALLOC_FL_COLLAPSE_RANGE 0x08
#define FALLOC_FL_ZERO_RANGE     0x10
#define FALLOC_FL_INSERT_RANGE   0x20
#define FALLOC_FL_UNSHARE_RANGE  0x40
```

## 参考文献 {#further-reading}

- [バッファードI/Oとページキャッシュ](buffered-io.md) — 事前割り当てされたエクステントとページキャッシュの相互作用
- [ダイレクトI/O](direct-io.md) — 事前割り当てされた範囲へのO_DIRECT書き込みでページキャッシュをバイパスする
- [writeの一生](life-of-a-write.md) — `write()`が事前割り当てされた（未書き込みの）エクステントとどう相互作用するか
- [VFS: ファイル操作](../vfs/file-ops.md) — `f_op->fallocate`のディスパッチ
- [splice、sendfile、copy_file_range](../vfs/splice-sendfile.md) — カーネル側のreflinkコピー
- [btrfs](../filesystems/btrfs.md) — COWのセマンティクスとエクステントの共有
- [XFS](../filesystems/xfs.md) — エクステントB-treeと事前割り当ての歴史
- `fs/fallocate.c` — VFSの`vfs_fallocate()`の実装
- `include/uapi/linux/falloc.h` — `FALLOC_FL_*`フラグの定義
- `fs/ext4/extents.c` — `ext4_fallocate()`と未書き込みエクステントの処理
