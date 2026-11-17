# dentryキャッシュとinodeキャッシュ {#dentry-and-inode-caches}

> VFSが性能のためにファイルシステムの検索をどうキャッシュするか

## なぜ3つの別々のキャッシュなのか {#why-three-separate-caches}

Linuxはファイルシステムのデータについて3つの別個のキャッシュを保持している。**dentryキャッシュ**（パス検索）、**inodeキャッシュ**（ファイルのメタデータ）、そして**ページキャッシュ**（ファイルの内容）である。これらが1つの統合されたキャッシュではなく別々になっている理由を理解すると、VFSの多くの設計判断が説明できる。

### 異なるデータ、異なるアクセスパターン {#different-data-different-access-patterns}

| キャッシュ | 格納するもの | 典型的なサイズ | アクセスパターン |
|-------|---------------|-------------|----------------|
| dcache | `(parent_dir, name) → inode number` | エントリあたり約200バイト | すべての`open()`、`stat()`、`exec()` — 極めて高頻度 |
| icache | inodeのメタデータ（サイズ、パーミッション、タイムスタンプ、ブロック位置） | エントリあたり約600バイト | ファイルがオープンされているとき、またはそのメタデータが必要なときにアクセスされる |
| ページキャッシュ | ファイルの内容（4KBのページ） | 可変（ホットなファイルでは最大数GB） | `read()`/`write()`/`mmap()`でアクセスされる |

`stat()`呼び出しはdentryとinodeを必要とするが、ページキャッシュのページは必要としない。`read()`は3つすべてを必要とする。大きなディレクトリでの`ls -l`は多数のdentryとinodeを必要とするが、ページキャッシュのページはほとんど、あるいはまったく必要としない。キャッシュを統合すると、サイズが大きく異なるオブジェクトを1つの構造に格納しなければならず、slab割り当ての効率性が損なわれる。

### なぜ「ファイルが見つからない」をキャッシュするのか {#why-cache-file-not-found}

dcacheは**negative dentry**、つまり検索がENOENTを返したエントリを格納する。これは無駄に思える。なぜ失敗をキャッシュするのか？

コンパイラは絶えずnegativeな検索を発生させる。`<stdio.h>`を探すCコンパイラは、インクルードパス内の各ディレクトリを試す: `/usr/local/include/stdio.h`、`/usr/include/stdio.h`など。negativeキャッシュがなければ、失敗した検索のたびにファイルシステムへアクセスすることになる。negativeキャッシュがあれば、カーネルはディスクを読まずにdcacheからENOENTを返す。何千ものファイルをコンパイルするビルドサーバーでは、これは大きな効果がある。

negative dentryは、positiveなものと同様にメモリ逼迫時に追い出され、ファイルが作成されると即座に無効化される（`d_instantiate()`がnegative dentryを置き換える）。

### なぜキャッシュが不可欠なのか {#why-caches-are-essential}

キャッシュがなければ、パスの構成要素を検索するたびに、ディレクトリエントリを見つけるためのディスク読み込みが必要になる。毎秒何千回も`open()`を呼ぶシステムでは、それは壊滅的に遅くなる。VFSはパス検索を**dentryキャッシュ（dcache）**にキャッシュし、オープンされたinodeを**inodeキャッシュ（icache）**に格納する。

## dentryキャッシュ（dcache） {#the-dentry-cache-dcache}

dcacheは`(parent_dentry, name) → dentry`を対応付けるハッシュテーブルである。パスの構成要素が一度検索されると、そのdentryがキャッシュされ、以降の検索ではファイルシステムを経由しなくて済む:

```
First lookup of "/usr/bin/bash":
  "/" → dentry for /   (in cache)
  "usr" → miss → ext4_lookup("/", "usr") → read disk → cache
  "bin" → miss → ext4_lookup("/usr", "bin") → read disk → cache
  "bash" → miss → ext4_lookup("/usr/bin", "bash") → read disk → cache

Second lookup of "/usr/bin/bash":
  "/" → hit
  "usr" → hit
  "bin" → hit
  "bash" → hit (entire path resolved from cache, no disk I/O)
```

### dentryの状態 {#dentry-states}

```
In-use (d_lockref.count > 0):
  Referenced by a struct file, nameidata, or other kernel object.
  Not eligible for eviction.

Unused (d_lockref.count == 0):
  Not currently referenced. On the LRU list.
  May be evicted under memory pressure.

Negative (d_inode == NULL):
  Cached "file not found" result.
  Avoids repeated filesystem lookups for non-existent files.
```

### dcacheの操作 {#dcache-operations}

Linux 2.6.38で、Nick Pigginの貢献により、dcache検索のためのRCUベースのロックフリーな高速パスが追加された[(LWN)](https://lwn.net/Articles/419811/)。現在では、ほとんどのパス検索はロックを一切取得せず、参照カウントも一切変更せずに完了する。各dentryのseqlockが並行する変更を検出し、必要な場合にのみ、より低速な参照カウント付きの走査へのフォールバックを引き起こす。

```c
/* dcacheを検索する（RCU、ロックフリーの高速パス。Linux 2.6.38でNick Pigginが導入） */
struct dentry *__d_lookup_rcu(const struct dentry *parent,
                               const struct qstr *name, unsigned *seqp);

/* 参照を取る完全な検索 */
struct dentry *d_lookup(const struct dentry *parent, const struct qstr *name);

/* 新しいdentryをキャッシュに追加する */
void d_add(struct dentry *entry, struct inode *inode);

/* dentryを未使用にする（参照を解放する） */
dput(dentry);

/* 無効化してキャッシュから削除する */
d_invalidate(dentry);

/* dentryがマウントポイントか確認する */
d_mountpoint(dentry);
```

### dcacheのサイズと統計 {#dcache-size-and-statistics}

```bash
# 現在のdcacheのサイズ
cat /proc/sys/fs/dentry-state
# 234567 198765 45 0 0 0
# nr_dentry nr_unused age_limit want_pages nr_negative nr_unused

# dcacheが使用しているメモリ
cat /proc/slabinfo | grep dentry
# dentry  198765  198765   192  21  1 : tunables  0  0  0 : slabdata  9465  9465  0

# dcacheのチューニング: より多くのエントリを保持する（デフォルト: 自動サイズ調整）
echo 1000000 > /proc/sys/fs/dentry-state  # 直接は書き込めない。vfs_cache_pressureで調整する
```

### vfs_cache_pressure {#vfs_cache_pressure}

メモリ逼迫時に、カーネルがdcacheとicacheをどれだけ積極的に回収するかを制御する:

```bash
# デフォルト: 100（バランス）
cat /proc/sys/vm/vfs_cache_pressure

# 高くする: より積極的に回収する（メモリ使用量は減り、キャッシュミスは増える）
echo 200 > /proc/sys/vm/vfs_cache_pressure

# 低くする: より多くのエントリを保持する（検索は速くなり、メモリ使用量は増える）
echo 50 > /proc/sys/vm/vfs_cache_pressure
```

## inodeキャッシュ（icache） {#the-inode-cache-icache}

dentryのinodeがディスクからロードされる（または作成される）と、inodeキャッシュに格納される。inodeは、参照されている限り（dentryまたはオープンされたファイルによって）キャッシュに残る。

### inodeの状態 {#inode-states}

```
In-use: referenced by at least one dentry or open file
  I_NEW: being initialized, not yet visible
  I_DIRTY_*: has dirty metadata (needs writeback)
  I_WRITEBACK: currently being written to disk
  I_FREEING: being evicted from cache

Unreferenced: no dentries, no open files
  On the inode LRU list, eligible for eviction
```

### inodeのライフサイクル {#inode-lifecycle}

```c
/* 新しいinodeを割り当てる（ファイルシステムから呼ばれる） */
struct inode *new_inode(struct super_block *sb)
{
    struct inode *inode = alloc_inode(sb);
    /* 定義されていればsb->s_op->alloc_inode()が呼ばれる */
    inode->i_sb = sb;
    inode_init_always(sb, inode);
    return inode;
}

/* 既存のinodeを取得する（キャッシュになければ作成する） */
struct inode *iget_locked(struct super_block *sb, unsigned long ino)
{
    /* (sb, ino)によるハッシュ検索 */
    inode = find_inode(sb, ino);
    if (inode)
        return inode;  /* キャッシュヒット */

    /* キャッシュミス: 割り当ててI_NEWをマークする */
    inode = alloc_inode(sb);
    insert_inode_hash(inode);
    /* 呼び出し元はinodeのフィールドを埋めてunlock_new_inode()を呼ばなければならない */
    return inode;
}

/* inodeの参照を解放する */
iput(inode)
    → if (--i_count == 0) {
        /* 参照がなくなった: evict_inodeを呼ぶ */
        if (inode->i_nlink == 0)
            sb->s_op->evict_inode(inode);  /* ファイルシステムから削除する */
        /* nlink > 0の場合: 回収候補としてLRUへ移動する */
    }
```

### 追い出し {#eviction}

メモリが少なくなると、カーネルは`prune_icache()`を呼び出して未使用のinodeを解放する:

```c
/* shrinkerから呼ばれる */
static long super_cache_scan(struct shrinker *shrink,
                              struct shrink_control *sc)
{
    struct super_block *sb = shrink->private_data;
    /* 未使用のdentryとinodeを追い出す */
    prune_dcache_sb(sb, sc);
    prune_icache_sb(sb, sc);
}
```

inodeが解放される前に、`evict_inode()`が呼ばれる。ext4の場合、これはダーティデータを書き戻し、ジャーナルのリソースを解放する。

## キャッシュの統計 {#cache-statistics}

```bash
# slabアロケータの統計（dentryキャッシュとinodeキャッシュを含む）
cat /proc/slabinfo | grep -E "^dentry|^inode_cache|ext4_inode"

# slabメモリ全体の内訳
cat /proc/meminfo | grep -E "Slab|SReclaimable|SUnreclaim"
# Slab:           2345678 kB
# SReclaimable:   1234567 kB  ← 逼迫時に回収可能（dcache、icache）
# SUnreclaim:     1111111 kB  ← 回収不可能

# dcache/icacheが実際にどれだけ使っているか:
vmstat -m | grep -E "dentry|inode"
```

## キャッシュの強制破棄（テスト用） {#forcing-cache-drop-for-testing}

```bash
# すべてのキャッシュを破棄する（pagecache + dcache + icache） — 本番環境では使わないこと
echo 3 > /proc/sys/vm/drop_caches

# dentryとinodeだけを破棄する
echo 2 > /proc/sys/vm/drop_caches

# pagecacheだけを破棄する
echo 1 > /proc/sys/vm/drop_caches
```

これは、コールドキャッシュの性能を確実に測定するためのベンチマークに役立つ。ダーティページは一切解放しない（いずれにせよ先にフラッシュされる）。

## 参考文献 {#further-reading}

- [VFSオブジェクト](vfs-objects.md) — dentryとinodeとは何か
- [パス解決](namei.md) — 検索中にdcacheがどう使われるか
- [ページキャッシュ](../mm/page-cache.md) — ファイルデータのための類似のキャッシュ
- [Shrinker API](../mm/shrinker.md) — キャッシュがメモリ回収システムにどう登録されるか
