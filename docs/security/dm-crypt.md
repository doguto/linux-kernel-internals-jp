# dm-crypt: ブロックレベルの暗号化 {#dm-crypt-block-level-encryption}

> device mapperの暗号化ターゲット、LUKS2の鍵管理、AES-XTS

## アーキテクチャ {#architecture}

dm-cryptはdevice mapperのスタック内に位置し、ファイルシステムとブロックデバイスの間のすべてのI/Oを透過的に暗号化/復号する:

```
Application
    ↓ read/write
Filesystem (ext4/Btrfs/XFS)
    ↓ block I/O
dm-crypt (device mapper target)
    ↓ encrypt on write, decrypt on read
Block device (/dev/sda1, NVMe, etc.)
    ↓
Physical storage: only ciphertext on disk
```

```bash
# dm-cryptデバイスを表示する:
dmsetup table /dev/mapper/cryptroot
# 0 976771072 crypt aes-xts-plain64 :64:logon:cryptsetup:...
# sectors:    cipher           key          IV mode
```

## LUKS2のオンディスク形式 {#luks2-on-disk-format}

LUKS（**L**inux **U**nified **K**ey **S**etup）は、dm-cryptヘッダの標準形式である。LUKS2（カーネル4.12 / cryptsetup 2.0以降）:

```
LUKS2 on-disk layout:
┌─────────────────────────────────────────────────────────┐
│ LUKS2 header (4KB) + backup header (4KB)                │
│   magic: "LUKS\xba\xbe" + version: 2                    │
│   hdr_size, seqid, label, checksum_alg                  │
│   salt[64], uuid, subsystem                             │
├─────────────────────────────────────────────────────────┤
│ JSON metadata area (variable, default 16KB)             │
│   keyslots: [{type, priority, kdf_params, af_params}]   │
│   tokens: [{type, keyslots: [...]}]                     │
│   segments: [{type: "crypt", offset, size, iv_tweak,    │
│               encryption: "aes-xts-plain64", ...}]      │
│   digests: [{type: "pbkdf2", keyslots, segments,        │
│              salt, digest}]                             │
├─────────────────────────────────────────────────────────┤
│ Keyslot area (default 16MB)                             │
│   Each keyslot: AF-split master key encrypted with      │
│   user passphrase-derived key                           │
├─────────────────────────────────────────────────────────┤
│ Encrypted data (segment 0)                              │
│   All filesystem data encrypted with master key         │
└─────────────────────────────────────────────────────────┘
```

### 鍵導出（PBKDF2/Argon2） {#key-derivation-pbkdf2argon2}

```
User passphrase
      ↓ PBKDF2 or Argon2id (memory-hard, tuned to take ~1 second)
Derived key (same size as master key)
      ↓ XOR with AF-split stripes
Master key (256-bit for AES-256-XTS)
      ↓ used directly for AES-XTS encryption
```

Argon2id（LUKS2のデフォルト）はメモリハードであり、GPUによる総当たり攻撃を無力化する。

## cryptsetup: LUKSの管理 {#cryptsetup-luks-management}

```bash
# LUKS2コンテナを作成する:
cryptsetup luksFormat --type luks2 \
    --cipher aes-xts-plain64 \
    --key-size 512 \           # 512ビット鍵 = AES-256-XTS（256+256に分割）
    --hash sha512 \
    --pbkdf argon2id \
    --iter-time 3000 \         # KDFの調整: 目標3秒
    /dev/sda1

# オープンする（復号して/dev/mapper/mydevを作成する）:
cryptsetup open /dev/sda1 mydev
# パスフレーズを尋ねる → /dev/mapper/mydevを作成する

# 復号されたデバイス上にファイルシステムを作成する:
mkfs.ext4 /dev/mapper/mydev
mount /dev/mapper/mydev /mnt

# クローズする（復号されたデバイスを削除する）:
umount /mnt
cryptsetup close mydev

# LUKSヘッダの情報を表示する:
cryptsetup luksDump /dev/sda1
# Version:        2
# Epoch:          3
# Metadata area:  16384 [bytes]
# Keyslots area:  16744448 [bytes]
# UUID:           ...
# Label:          (no label)
# Subsystem:      (no subsystem)
# Flags:          (no flags)
# Keyslots:
#  0: luks2 (active)
#     Priority:   normal
#     Cipher:     aes-xts-plain64
#     Cipher key: 512 bits
#     PBKDF:      argon2id

# 2つ目のパスフレーズを追加する（キースロット1）:
cryptsetup luksAddKey /dev/sda1

# キーファイルを追加する（自動アンロック用）:
dd if=/dev/urandom of=/root/keyfile bs=512 count=4
cryptsetup luksAddKey /dev/sda1 /root/keyfile

# キーファイルでオープンする:
cryptsetup open /dev/sda1 mydev --key-file /root/keyfile

# パスフレーズを削除する:
cryptsetup luksKillSlot /dev/sda1 1   # スロット1を削除する
```

## dm-cryptのカーネルターゲット {#dm-crypt-kernel-target}

カーネル側の実装は`drivers/md/dm-crypt.c`にある:

```c
/* device mapperのターゲット構造体 */
struct dm_target {
    /* ... */
    sector_t begin;    /* デバイス内でのターゲットの開始位置 */
    sector_t len;      /* ターゲットの長さ */
    struct target_type *type;
    void *private;     /* crypt_configを指す */
};

/* dm-cryptの私的な状態 */
struct crypt_config {
    struct dm_dev   *dev;          /* 下位のデバイス */
    sector_t         start;        /* 開始セクタ */

    /* 暗号 */
    struct crypto_skcipher **tfms; /* CPUごとの暗号ハンドル */
    unsigned int     tfms_count;   /* = num_cpus */
    char            *cipher_string; /* "aes-xts-plain64" */
    unsigned int     key_size;
    u8              *key;          /* カーネルメモリ内のマスター鍵 */

    /* IV */
    iv_generator_fn  iv_gen_ops;   /* plain64、essiv、tcwなど */

    /* I/O */
    mempool_t        req_pool;     /* crypt_io構造体のプール */
    struct workqueue_struct *io_queue;   /* 暗号化/復号のワーカ */
    struct workqueue_struct *crypt_queue;
};
```

### 暗号化のパス（書き込み） {#encryption-path-write}

```c
/* dm-cryptはすべてのbioを横取りする: */
static int crypt_map(struct dm_target *ti, struct bio *bio)
{
    struct crypt_io *io;
    struct crypt_config *cc = ti->private;

    /* リクエストごとの状態を割り当てる */
    io = mempool_alloc(&cc->io_pool, GFP_NOIO);
    io->cc = cc;
    io->base_bio = bio;

    if (bio_data_dir(bio) == WRITE) {
        /* 暗号化: crypt_queue workqueueに投入する */
        kcryptd_queue_crypt(io);
    } else {
        /* 読み取り: 平文のI/Oを発行し、完了時に復号する */
        crypt_submit_bio(cc, io, bio);
    }
    return DM_MAPIO_SUBMITTED;
}

/* 暗号化のワーカ: */
static void kcryptd_crypt_write_io_submit(struct crypt_io *io, int async)
{
    /* bio内の各ページについて: */
    /*   1. セクタ番号からIVを設定する */
    /*   2. skcipher_request_set_crypt() */
    /*   3. crypto_skcipher_encrypt() */
    /*   4. 暗号化したbioを下位デバイスに発行する */
}
```

### IVモード {#iv-modes}

```
plain64:   IV = sector_number (64-bit)
           Simple, fast; vulnerable to watermarking attacks

essiv:     IV = encrypt(sector_number, hash(master_key))
           Sector numbers not visible as IVs

tcw:       Tweak-CBC-Wide; for XTS mode with twist
           Resists CPA (chosen plaintext attacks)

aes-xts-plain64 (recommended):
           XTS = XEX (XOR-Encrypt-XOR) Tweakable Block Cipher
           Two AES keys: one for encryption, one for IV tweak
           Standard for disk encryption (IEEE P1619)
```

## 性能 {#performance}

```bash
# 暗号のオプションをベンチマークする:
cryptsetup benchmark
# # Tests are approximate using memory only (no storage IO).
# PBKDF2-sha1        1234567 iterations per second for 256-bit key
# PBKDF2-sha512       543210 iterations per second for 256-bit key
# argon2i           7 iterations per second for 256-bit key (64 MB memory)
# argon2id          7 iterations per second for 256-bit key (64 MB memory)
# #     Algorithm |       Key |      Encryption |      Decryption
#          aes-cbc        128b      1234.5 MiB/s      3456.7 MiB/s
#          aes-cbc        256b      1012.3 MiB/s      2890.1 MiB/s
#          aes-xts        256b      2100.4 MiB/s      2098.7 MiB/s
#          aes-xts        512b      1890.2 MiB/s      1887.3 MiB/s

# AES-NIによるハードウェア高速化:
grep -m1 aes /proc/cpuinfo
# flags: ... aes ...  ← ハードウェアAES-NIが存在する

# 生デバイスと比べたdm-cryptのオーバーヘッドを測定する:
fio --filename=/dev/mapper/mydev --direct=1 --rw=read --bs=128k \
    --ioengine=libaio --iodepth=32 --name=crypt-read
fio --filename=/dev/sda1 --direct=1 --rw=read --bs=128k \
    --ioengine=libaio --iodepth=32 --name=raw-read
# AES-NIがあれば: オーバーヘッドは通常5%未満
```

## /etc/crypttab: 永続的な設定 {#etccrypttab-persistent-configuration}

```bash
# /etc/crypttabの形式: name device keyfile options
cryptroot   /dev/sda1   none          luks,discard
cryptdata   /dev/sdb1   /root/keyfile luks,nofail
cryptswap   /dev/sdc1   /dev/urandom  swap,cipher=aes-xts-plain64

# フィールド:
# name:    → /dev/mapper/<name>
# device:  UUID=... or /dev/sdX or LABEL=...
# keyfile: "none" = 入力を求める、パス = ファイル、/dev/urandom = ランダム（スワップ用）
# options: luks、discard（trim）、nofail（存在しなければスキップ）など

# デバイス名に依存しないようUUIDを使う:
blkid /dev/sda1 | grep UUID
cryptroot   UUID=abc123-...   none   luks
```

## LUKSとplain dm-cryptの比較 {#luks-vs-plain-dm-crypt}

```bash
# plain dm-crypt（LUKSヘッダなし、鍵管理なし）:
cryptsetup open --type plain \
    --cipher aes-xts-plain64 \
    --key-size 512 \
    --hash sha512 \
    /dev/sda1 mydev
# 鍵はハッシュを介してパスフレーズから直接導出される（KDFの調整なし）
# ヘッダがないのでメタデータの痕跡が残らない（否認可能な暗号化）
# ただし: 複数の鍵を追加できず、完全性のチェックもない
```

## dm-integrity: 認証 {#dm-integrity-authentication}

dm-cryptは機密性を提供するが、完全性は提供しない。dm-integrityは認証タグを追加する:

```bash
# dm-integrity + dm-cryptのスタックを作成する:
# ステップ1: integrity付きでフォーマットする
cryptsetup open --type plain --integrity hmac-sha256 /dev/sda1 mydev-int
# ステップ2: その上にLUKSを載せる
cryptsetup luksFormat /dev/mapper/mydev-int
cryptsetup open /dev/mapper/mydev-int mydev

# あるいはintegrity付きのLUKS2を直接使う:
cryptsetup luksFormat --type luks2 \
    --integrity hmac-sha256 \
    /dev/sda1
# 512バイトのセクタごとにHMACタグを追加する → 改ざんを検出する
```

## 可観測性 {#observability}

```bash
# dm-cryptの統計:
dmsetup status /dev/mapper/cryptroot
# 0 976771072 crypt 0 0 0 0 /dev/sda1 2048

# dm-cryptデバイスのブロックI/O統計:
iostat -x /dev/mapper/cryptroot 1

# 暗号サブシステムの統計:
cat /proc/crypto | grep -A5 "aes"
# name         : xts(aes)
# driver       : xts-aes-aesni
# module       : kernel
# priority     : 401
# type         : skcipher

# ハードウェア暗号とソフトウェア暗号:
# -aesniで終わるdriver = ハードウェア高速化
# -genericで終わるdriver = ソフトウェアによるフォールバック

# BPFで暗号化のレイテンシをトレースする:
bpftrace -e '
kprobe:kcryptd_crypt_write_convert { @start[tid] = nsecs; }
kretprobe:kcryptd_crypt_write_convert /@start[tid]/
{
    @lat_us = hist((nsecs - @start[tid]) / 1000);
    delete(@start[tid]);
}'
```

## 参考文献 {#further-reading}

- [Btrfs](../filesystems/btrfs.md) — dm-cryptの上でよく使われるファイルシステム
- [ext4のジャーナリング](../filesystems/ext4-journal.md) — dm-cryptの上に載るファイルシステム
- [NVMeドライバ](../block/nvme.md) — dm-cryptの下にあるブロックデバイス
- [カーネルの暗号API](../crypto/crypto-api.md) — dm-cryptが使うskcipher/AEAD
- [LSMフレームワーク](lsm.md) — dm-cryptはSELinux/AppArmorと併用できる
- `drivers/md/dm-crypt.c` — dm-cryptターゲットの実装
- `lib/crypto/` — カーネルのAES-XTS実装
- `Documentation/admin-guide/device-mapper/dm-crypt.rst`
