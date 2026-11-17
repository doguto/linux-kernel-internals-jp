# fscrypt: ファイル単位の暗号化 {#fscrypt-per-file-encryption}

> 鍵階層、HKDFによる導出、ext4/F2FS/Btrfsにおける透過的暗号化

## 概要 {#overview}

fscryptはVFS層で**ディレクトリ単位**の透過的暗号化を提供する。dm-crypt（ディスク全体）とは異なり、fscryptは個々のファイルとディレクトリを暗号化するため、次のことが可能になる:

- 複数のユーザーが、異なる鍵を使う別々の暗号化ディレクトリを持つ
- 選択的な暗号化（すべてのファイルを暗号化する必要はない）
- 高速な鍵の失効（鍵を削除 → ファイルにアクセスできなくなる）

対応ファイルシステム: **ext4、F2FS、UBIFS、Btrfs**（5.14以降）

```
Application read()/write()
          ↓
    Page cache (plaintext while key is present)
          ↓ (encrypt on writeback, decrypt on readahead)
    fscrypt layer
          ↓
    Block device (ciphertext)
```

## 鍵階層 {#key-hierarchy}

fscryptは2段階の鍵階層を使う:

```
User passphrase / TPM / hardware key
         ↓ PBKDF2 or Argon2 (userspace)
Master key (256-bit, type: ext4/logon)
         ↓ HKDF-SHA512 (kernel)
         ├── File encryption key (per-file, derived from master + file nonce)
         ├── Filename encryption key (for directory entries)
         └── IV (per-block, derived from file nonce + block number)
```

暗号化された各ディレクトリは、次の内容を記録する**ポリシー**を持つ:

- どのマスター鍵を使うか（鍵記述子または鍵識別子で指定）
- どの暗号化モードを使うか（内容にはAES-256-XTS、ファイル名にはAES-256-CTS）
- 鍵導出のバージョン（v1は信頼されたキーリング、v2はHKDF）

## 暗号化のセットアップ（API v2） {#setting-up-encryption-api-v2}

```bash
# ステップ1: ファイルシステムで暗号化を有効にする（ext4）:
tune2fs -O encrypt /dev/sda1
# またはmkfs時に:
mkfs.ext4 -O encrypt /dev/sda1

# ステップ2: マスター鍵をカーネルのキーリングに追加する:
# 512ビットの鍵を生成する:
dd if=/dev/urandom bs=64 count=1 of=/tmp/master_key 2>/dev/null

# セッションキーリングに追加する:
keyctl add logon fscrypt:$(xxd -p /tmp/master_key | head -c 16) \
    /tmp/master_key @s

# あるいはfscryptツールを使う:
apt install fscrypt
fscrypt setup /dev/sda1
fscrypt encrypt /home/user/private/ --user=user
```

### カーネルのioctl API {#kernel-ioctl-api}

```c
#include <linux/fscrypt.h>

/* ステップ1: マスター鍵をファイルシステムのキーリングに追加する */
struct fscrypt_add_key_arg {
    struct fscrypt_key_specifier key_spec;  /* 鍵を識別する */
    __u32 raw_size;     /* AESでは16〜64バイト、HKDFでは64バイト */
    __u32 key_id;       /* 出力: カーネルが割り当てたID */
    __u32 __reserved[8];
    __u8  raw[];        /* 鍵素材 */
};

struct fscrypt_key_specifier key_spec = {
    .type = FSCRYPT_KEY_SPEC_TYPE_IDENTIFIER,  /* v2: HKDFベース */
};
int key_fd = open("/", O_RDONLY);
ioctl(key_fd, FS_IOC_ADD_ENCRYPTION_KEY, &add_key_arg);
/* key_spec.u.identifier（鍵の16バイトのハッシュ）を返す */

/* ステップ2: ディレクトリに暗号化ポリシーを設定する */
struct fscrypt_policy_v2 policy = {
    .version           = FSCRYPT_POLICY_V2,
    .contents_encryption_mode = FSCRYPT_MODE_AES_256_XTS,
    .filenames_encryption_mode = FSCRYPT_MODE_AES_256_CTS,
    .flags             = FSCRYPT_POLICY_FLAGS_PAD_32,
    .master_key_identifier = /* key_spec.u.identifier */,
};
int dir_fd = open("/home/user/private", O_RDONLY);
ioctl(dir_fd, FS_IOC_SET_ENCRYPTION_POLICY, &policy);
/* 以降、このディレクトリ内の新しいファイルはすべて暗号化される */
```

## 鍵導出の内部 {#key-derivation-internals}

```c
/* fs/crypto/hkdf.c */

/* ファイルごとの鍵はすべてHKDF-SHA512で導出される: */

/* ファイル内容の鍵: */
/* 入力鍵素材（IKM） = master_key */
/* Info = "fscrypt\0" + mode + filesystem UUID + file nonce */
fscrypt_hkdf_expand(&ci->ci_master_key->mk_secret.hkdf,
                     HKDF_CONTEXT_PER_FILE_ENC_KEY,
                     info, infolen,
                     derived_key, mode->keysize);

/* ファイルnonce: ディスク上のinodeに格納されるランダムな16バイト */
/* 各ファイルは固有のnonceを持つ → 同じマスター鍵でも鍵は固有になる */

/* ファイルごとの鍵の格納先: */
struct fscrypt_inode_info {
    struct fscrypt_master_key *ci_master_key;
    union fscrypt_iv    ci_iv;          /* ブロックごとのIVの基点 */
    struct crypto_skcipher *ci_enc_key; /* 実際の暗号 */
    /* ... */
};
```

### IVの生成（ブロック単位） {#iv-generation-per-block}

```c
/* AES-256-XTS（内容の暗号化、デフォルトのファイルごとの鍵）の場合: */
/* IV = block_number（ファイル内の論理ブロックインデックス、0始まり） */
/* ファイルごとの鍵はHKDFでマスター鍵 + ファイルnonceから導出される */

/* ファイル名の暗号化（AES-256-CTS-CBC）の場合: */
/* IV = 0（ECBではなくCTS-CBC — ファイル名全体にわたる拡散を提供する） */
/* → 同じディレクトリ内の同じファイル名は常に同じように暗号化される */
/* → （鍵があれば）ディレクトリ一覧の取得が可能。順序は漏れる */

/* ポリシーフラグFSCRYPT_POLICY_FLAG_IV_INO_LBLK_64の場合: */
/* IV = inode_number（上位ビット） | block_number（下位ビット）、64ビットに詰める */
/* ハードウェアのインライン暗号化を可能にする（ファイルシステムごとに1つの鍵） */
/* 注: FSCRYPT_POLICY_FLAG_IV_INO_LBLK_32は32ビットIVにSipHashを使う */
```

## インライン暗号化ハードウェア {#inline-encryption-hardware}

```bash
# インライン暗号化のサポートを確認する:
cat /sys/block/nvme0n1/queue/crypto_capabilities 2>/dev/null
# AES-256-XTS, key size 64, data unit size 512/1024/2048/4096

# インライン暗号化を使ってマウントする:
mount -o inlinecrypt /dev/nvme0n1p1 /data
# カーネルがNVMeのキースロットを設定し、ブロック層がハードウェアで暗号化する
# CPUは鍵を設定するだけ — ブロックごとの暗号処理のオーバーヘッドはない

# inlinecryptが有効か確認する:
cat /proc/mounts | grep inlinecrypt
```

## 鍵の失効とロック {#key-revocation-and-locking}

```c
/* ファイルシステムからマスター鍵を削除する（暗号化ディレクトリをロックする）: */
ioctl(fs_fd, FS_IOC_REMOVE_ENCRYPTION_KEY, &key_spec);
/* 暗号化ファイルのすべてのページがページキャッシュから追い出される */
/* inodeのci_enc_keyが解放される → ファイルは読めなくなる */
/* 次のアクセス試行時: -ENOKEY（鍵が利用できない） */
```

```bash
# fscryptツールを使う:
fscrypt lock /home/user/private/   # ロック（カーネルから鍵を削除）
fscrypt unlock /home/user/private/ # アンロック（鍵を再追加）

# 暗号化の状態を確認する:
fscrypt status /home/user/private/
# "/home/user/private": encrypted
#   Policy version:      2
#   Encryption mode:     AES-256-XTS (contents)
#                        AES-256-CTS (filenames)
#   Key:                 unlocked (key identifier: abc123...)
```

## fscryptとdm-cryptの比較 {#fscrypt-vs-dm-crypt}

| | fscrypt | dm-crypt |
|---|---------|----------|
| 粒度 | ディレクトリ単位 | ブロックデバイス全体 |
| 複数の鍵 | 可 | 不可（デバイスごとに1つの鍵） |
| 鍵の失効 | 稼働中に失効可能（ページは追い出される） | アンマウントが必要 |
| メタデータの暗号化 | ファイル名のみ | すべてのメタデータ |
| 性能 | CPU負荷がやや高い | AES-NIでほぼコストなし |
| 用途 | マルチユーザー（AndroidのFBE、共有ラップトップ） | シングルユーザーのラップトップ、サーバー |
| ファイルシステム | ext4/F2FS/Btrfs | 上に載せる任意のFS |

## 可観測性 {#observability}

```bash
# ファイルが暗号化されているか確認する:
fscryptctl get_policy /home/user/private/secret.txt
# Encryption policy for /home/user/private/secret.txt:
#   Policy version: 2
#   Master key identifier: abc1234567890def
#   Contents encryption mode: AES-256-XTS
#   Filenames encryption mode: AES-256-CTS
#   Flags: PAD_32

# ファイルシステム内のすべてのマスター鍵を一覧表示する:
ioctl(fs_fd, FS_IOC_GET_ENCRYPTION_KEY_STATUS, &status)
# （あるいはfscrypt statusを使う）

# 暗号関連のエラーをdmesgで確認する:
dmesg | grep -E "fscrypt|crypto"

# BPFで鍵の検索をトレースする:
bpftrace -e 'kprobe:fscrypt_get_encryption_info { @[comm] = count(); }'
```

## 参考文献 {#further-reading}

- [dm-crypt](dm-crypt.md) — 代替手段としてのディスク全体の暗号化
- [xattr](../vfs/xattr.md) — fscryptはポリシーを`trusted.fscrypt` xattrに格納する
- [Linuxのaudit](audit.md) — fscryptの鍵操作を監査する
- [クレデンシャル](credentials.md) — マスター鍵のためのユーザーごとのキーリング
- `fs/crypto/` — fscryptの実装
- `Documentation/filesystems/fscrypt.rst`
