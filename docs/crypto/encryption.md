# dm-cryptとfscrypt {#dm-crypt-and-fscrypt}

> Linuxにおけるブロックレベル暗号化とファイルレベル暗号化

## 2つの暗号化レイヤー {#two-encryption-layers}

Linuxは2つのレベルで暗号化を提供する:

| レイヤー | サブシステム | 暗号化される対象 | 鍵の粒度 |
|-------|-----------|-----------------|-----------------|
| ブロックデバイス | dm-crypt | パーティション/ディスク全体 | ボリュームごとに1つの鍵 |
| ファイルシステム | fscrypt | 個々のファイル/ディレクトリ | ファイルごとまたはディレクトリごとの鍵 |

dm-cryptはファイルシステムから透過的である。fscryptはアプリケーションから透過的だが、ファイル単位で動作する。

## dm-crypt: ブロックレベル暗号化 {#dm-crypt-block-level-encryption}

dm-cryptはdevice mapperのターゲットであり、カーネル暗号APIを使って読み書きを透過的に暗号化する。

```
Application → read/write /dev/mapper/myvolume
                              │
                         dm-crypt layer
                         (AES-XTS decrypt/encrypt)
                              │
                         /dev/sda2 (ciphertext on disk)
```

### LUKS: Linux Unified Key Setup {#luks-linux-unified-key-setup}

LUKSはdm-cryptボリュームの標準フォーマットである。暗号化された鍵素材をヘッダーに格納し、複数のパスフレーズで同じボリュームをアンロックできるようにする:

```bash
# LUKSボリュームを作成
cryptsetup luksFormat /dev/sda2
# パスフレーズを入力

# ボリュームを開く（復号する）
cryptsetup luksOpen /dev/sda2 myvolume
# これで /dev/mapper/myvolume が存在する

# 通常のブロックデバイスと同様に使う
mkfs.ext4 /dev/mapper/myvolume
mount /dev/mapper/myvolume /mnt/secure

# 閉じる（ボリューム鍵を再暗号化し、デバイスを削除）
umount /mnt/secure
cryptsetup luksClose myvolume
```

### dm-cryptの内部 {#dm-crypt-internals}

```c
/* drivers/md/dm-crypt.c */
struct crypt_config {
    struct dm_dev       *dev;           /* 下位のブロックデバイス */
    sector_t             start;

    /* CPUごとのI/Oワークキュー */
    struct workqueue_struct *io_queue;
    struct workqueue_struct *crypt_queue;

    /* 暗号変換 */
    union {
        struct crypto_skcipher **tfms;  /* 並列化のためCPUごとに1つ */
        struct crypto_aead    **tfms_aead;
    } cipher_tfm;
    unsigned int          tfms_count;

    /* IVジェネレータ */
    const struct crypt_iv_operations *iv_gen_ops;  /* ESSIV, plain64, benbi, ... */
    union {
        struct iv_benbi_private    benbi;
        struct iv_lmk_private      lmk;
        struct iv_tcw_private      tcw;
        struct iv_elephant_private elephant;
    } iv_gen_private;                   /* IVモードごとのプライベート状態 */

    u64                    iv_offset;   /* IV用のセクタ番号オフセット */
    unsigned int           iv_size;

    char                  *cipher_string;  /* 埋め込みではなく割り当てられる */
};

/* 各I/Oはcrypt_ioの中で処理される */
struct dm_crypt_io {
    struct crypt_config    *cc;
    struct bio             *base_bio;
    u8                     *integrity_metadata;

    struct work_struct      work;   /* crypt_queueに投入される */

    struct convert_context  ctx;    /* 現在の変換状態 */
    atomic_t                io_pending;
    blk_status_t            error;
    sector_t                sector;
};
```

### AES-XTS: なぜCBCではないのか {#aes-xts-why-not-cbc}

dm-cryptはディスク暗号化にAES-CBCではなくAES-XTS（XEX-based tweaked-codebook mode with ciphertext stealing）を使う:

**ディスク暗号化におけるCBCの弱点**: CBCでは、ある鍵に対してセクタ1は常に同じ暗号文に暗号化される（IVがセクタ番号から導出されるため）。攻撃者はどのセクタが同一の平文を持つかを検出できる。AES-XTSは物理セクタ番号から導出したtweakを使い、各セクタの暗号化を一意にする。

```
AES-XTS(key, sector_num, plaintext) → ciphertext

Tweak = AES_encrypt(key2, sector_num)
For each 16-byte block i in the sector:
  ciphertext[i] = AES_encrypt(key1, plaintext[i] XOR tweak_i) XOR tweak_i
  tweak_{i+1}   = GF(2^128) multiplication of tweak_i
```

### IVの生成 {#iv-generation}

dm-cryptはいくつかのIVモードをサポートする:

```c
/* IVモード（暗号文字列で選択する）: */

/* plain/plain64: IV = セクタ番号 */
/* "aes-cbc-plain64" */

/* essiv: IV = ボリューム鍵のSHA256でencrypt(sector_num) */
/* CBCに対するウォーターマーキング攻撃を防ぐ */
/* "aes-cbc-essiv:sha256" */

/* 新規ボリュームの標準: */
/* "aes-xts-plain64" — セクタ番号を64ビットLE整数として使う */
```

## dm-integrity: 認証付きブロックストレージ {#dm-integrity-authenticated-block-storage}

dm-integrityはデータ破損を検出するためにセクタごとのチェックサムを追加する:

```bash
# integrityデバイスを作成（512バイトセクタごとに4バイトのCRCを追加）
cryptsetup open --type integrity /dev/sda2 myintegrity
# あるいはdm-cryptと組み合わせて認証付き暗号化にする:
cryptsetup luksFormat --integrity hmac-sha256 /dev/sda2
```

## fscrypt: ファイルシステムレベル暗号化 {#fscrypt-filesystem-level-encryption}

fscryptはext4、f2fs、ubifsに組み込まれている。ファイルの内容とファイル名を暗号化し、鍵はカーネルkeyringに格納される。

```
Directory with encryption policy:
  /encrypted/
  ├── Afvd3kLm (encrypted filename → real: "secret.txt")
  └── Bxyz9pQr (encrypted filename → real: "notes.md")

Reading a file:
  open("/encrypted/Afvd3kLm") → page cache miss
  → read block from disk (ciphertext)
  → fscrypt_decrypt_pagecache_blocks()
  → AES-256-XTS with per-file derived key
  → return plaintext
```

### fscryptのセットアップ {#setting-up-fscrypt}

```bash
# CONFIG_FS_ENCRYPTION=y とfscrypt対応ファイルシステムが必要

# マスター鍵を生成（256ビット）
keyctl new_session
fscryptctl generate-key > master.key

# マスター鍵をファイルシステムに追加
fscryptctl add-key /mnt/myfs < master.key
# Key identifier: 0123456789abcdef0123456789abcdef

# ディレクトリに暗号化ポリシーを設定
fscryptctl set-policy 0123456789abcdef0123456789abcdef /mnt/myfs/private/
# これで /mnt/myfs/private/ 以下に作られるファイルはすべて暗号化される

# 鍵なしでアンマウントして再マウントした後:
# ファイル名は暗号文として表示され、内容は読めない
```

### 鍵の階層 {#key-hierarchy}

fscryptはHKDF-SHA512を使ってマスター鍵からファイルごとの鍵を導出する:

```
Master key (256 bit)
      │
      │ HKDF-SHA512(master_key, "fscrypt\0" || context || nonce)
      ▼
Per-file key (256 bit, unique per inode)
      │
      │ AES-256-XTS
      ▼
Encrypted content
```

inodeに格納されるnonceはファイルごとに一意であり、内容が同一のファイルでも異なる鍵になることを保証する。

### fscryptポリシー {#fscrypt-policies}

```c
/* include/uapi/linux/fscrypt.h */
struct fscrypt_policy_v2 {
    __u8 version;                       /* FSCRYPT_POLICY_V2 = 2 */
    __u8 contents_encryption_mode;      /* FSCRYPT_MODE_AES_256_XTS */
    __u8 filenames_encryption_mode;     /* FSCRYPT_MODE_AES_256_CTS */
    __u8 flags;                         /* FSCRYPT_POLICY_FLAGS_PAD_* */
    __u8 log2_data_unit_size;           /* 0 = ファイルシステムのブロックサイズ */
    __u8 __reserved[3];
    __u8 master_key_identifier[FSCRYPT_KEY_IDENTIFIER_SIZE]; /* 16バイト */
};

/* 暗号化モード: */
FSCRYPT_MODE_AES_256_XTS   /* ファイル内容用（推奨） */
FSCRYPT_MODE_AES_256_CTS   /* ファイル名用 */
FSCRYPT_MODE_AES_128_CBC   /* レガシー */
FSCRYPT_MODE_ADIANTUM      /* 低消費電力デバイス用（AES-NIなし） */
```

### カーネル内のfscrypt {#fscrypt-in-the-kernel}

```c
/* fs/crypto/crypto.c */
int fscrypt_decrypt_pagecache_blocks(struct folio *folio, size_t len,
                                      size_t offs)
{
    const struct inode *inode = folio->mapping->host;
    const struct fscrypt_inode_info *ci = fscrypt_get_inode_info_raw(inode);
    const unsigned int du_bits = ci->ci_data_unit_bits;
    const unsigned int du_size = 1U << du_bits;
    u64 index = ((u64)folio->index << (PAGE_SHIFT - du_bits)) + (offs >> du_bits);
    size_t i;
    int err;

    for (i = offs; i < offs + len; i += du_size, index++) {
        struct page *page = folio_page(folio, i >> PAGE_SHIFT);

        /* ファイルごとの鍵で1データユニットを復号する。indexは
         * IVの入力（論理データユニット番号）も兼ねる */
        err = fscrypt_crypt_data_unit(ci, FS_DECRYPT, index, page,
                                      page, du_size, i & ~PAGE_MASK);
        if (err)
            return err;
    }
    return 0;
}
```

## カーネルkeyringとの統合 {#kernel-keyring-integration}

dm-crypt（cryptsetup経由）とfscryptはどちらも鍵素材をカーネルkeyringに格納する:

```bash
# 現在のkeyringを表示
keyctl show @s         # セッションkeyring
keyctl show @u         # ユーザーkeyring
keyctl show @us        # ユーザーセッションkeyring

# dm-cryptのボリューム鍵をkeyringに追加
keyctl add logon cryptsetup:0123456789abcdef <key_data> @s

# fscryptのマスター鍵を追加
keyctl add logon fscrypt:0123456789abcdef <key_data> @s

# 使われる鍵タイプ:
# "user": 任意のユーザーデータ
# "logon": カーネル専用（ユーザー空間から読み戻せない）
# "keyring": keyringそのもの（他の鍵を保持できる）
# "encrypted": マスター鍵で暗号化される
```

```c
/* カーネル: 鍵を検索する */
struct key *key = request_key(&key_type_logon, "fscrypt:...", NULL);
const struct user_key_payload *ukp = user_key_payload_locked(key);
memcpy(master_key, ukp->data, ukp->datalen);
key_put(key);
```

## 暗号化の観測 {#observing-encryption}

```bash
# dm-cryptの状態
cryptsetup status myvolume
# /dev/mapper/myvolume is active and is in use.
#   type:    LUKS2
#   cipher:  aes-xts-plain64
#   keysize: 512 bits     (256 data + 256 tweak for XTS)
#   key location: dm-crypt
#   device:  /dev/sda2

# LUKSヘッダー情報
cryptsetup luksDump /dev/sda2

# fscryptの状態
fscryptctl get-policy /mnt/myfs/private/
# Encryption policy for /mnt/myfs/private/:
#   Policy version: 2
#   Master key identifier: 0123456789abcdef0123456789abcdef
#   Contents encryption mode: AES-256-XTS
#   Filenames encryption mode: AES-256-CTS

# 性能
cryptsetup benchmark
# 一般的な暗号/モードの組み合わせをテストする

# ブロック層の統計（dm-cryptを通るI/O）
cat /sys/block/dm-0/stat
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [drivers/md/dm-crypt.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/md/dm-crypt.c) — `struct crypt_config`、`crypt_map()`、CPUごとの暗号/IVのセットアップ
- [fs/crypto/crypto.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/crypto/crypto.c) — `fscrypt_decrypt_pagecache_blocks()`
- [include/uapi/linux/fscrypt.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/fscrypt.h) — `struct fscrypt_policy_v2` と `FSCRYPT_MODE_*` 定数

### manページ {#man-pages}

- [`cryptsetup(8)`](https://man.archlinux.org/man/cryptsetup.8.en) — LUKS/dm-cryptのボリューム管理
- [`keyctl(1)`](https://man7.org/linux/man-pages/man1/keyctl.1.html) — keyringのコマンドラインツール（`add`、`show`、`new_session`、...）
- [`keyrings(7)`](https://man7.org/linux/man-pages/man7/keyrings.7.html) — カーネル内の鍵管理機構

### 関連ページ {#related-pages}

- [dm-crypt: ブロックレベル暗号化](../security/dm-crypt.md) — LUKS2のオンディスクフォーマット、鍵導出、カーネル側の `crypt_map()` I/Oパスをさらに掘り下げる
- [fscrypt: ファイルごとの暗号化](../security/fscrypt.md) — 鍵の階層、`FS_IOC_*` ioctl API、インライン暗号化ハードウェアをさらに掘り下げる
- [カーネルkeyring](keyring.md) — dm-cryptとfscryptの両方が鍵素材の格納に使う `logon` 鍵タイプ
- [カーネル暗号API](crypto-api.md) — dm-cryptとfscryptが呼び出す `skcipher` およびAEAD変換（`xts(aes)`、`gcm(aes)`）
- [crypto_engine: ハードウェアオフロードフレームワーク](crypto-engine.md) — dm-cryptのような呼び出し元の下にハードウェア暗号アクセラレータがどう組み込まれるか
- [乱数生成](rng.md) — 鍵素材（`/dev/urandom`、`get_random_bytes()`）がどこから来るか

### LWN記事 {#lwn-articles}

- [Ext4 encryption](https://lwn.net/Articles/639427/) — Jonathan Corbet、2015年4月8日 — 後にfscryptとなった、ファイルごとの暗号化の当初の設計
- [Adiantum: encryption for the low end](https://lwn.net/Articles/776721/) — Jake Edge、2019年1月16日 — AES-NIを持たないデバイス向けの `FSCRYPT_MODE_ADIANTUM` 暗号

### 外部資料 {#external}

- [Documentation/filesystems/fscrypt.rst](https://docs.kernel.org/filesystems/fscrypt.html) — カーネル自身によるfscryptの設計と脅威モデルのドキュメント
- [Documentation/admin-guide/device-mapper/dm-crypt.rst](https://docs.kernel.org/admin-guide/device-mapper/dm-crypt.html) — カーネル自身によるdm-cryptターゲットのドキュメント
