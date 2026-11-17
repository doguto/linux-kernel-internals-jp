# Device Mapper: dm-verityと整合性検査 {#device-mapper-dm-verity-and-integrity-checking}

> AndroidとChromeOSのための、Merkleツリーによるブロックデバイスデータの検証

## dm-verityとは？ {#what-is-dm-verity}

dm-verityは、Merkle（ハッシュ）ツリーを使ってブロックデバイスの**透過的な整合性検査**を提供する。すべての読み込みは、事前に計算されたハッシュに対して検証される。データが改ざんされていれば、カーネルはエラーを返すかパニックする。

使用例:

- **Android**: 読み取り専用のシステムパーティションの検証
- **ChromeOS**: rootfsのverified boot
- **Linuxディストリビューション**: 整合性を備えたイミュータブルなルートファイルシステム

```
Without dm-verity: read block → use data (no check)
With dm-verity:    read block → hash block → compare with Merkle tree → use data
```

## Merkleツリーの構造 {#merkle-tree-structure}

```
Data blocks:        [B0][B1][B2][B3][B4][B5][B6][B7]
                      ↓   ↓   ↓   ↓   ↓   ↓   ↓   ↓
Level 1 hashes:     [H0][H1][H2][H3][H4][H5][H6][H7]  ← leaf level (level N-1)
                     └─────────┘   └─────────────────┘
Level 0 hashes:         [H01]              [H23-47]    ← level 0 = closest to root
                          └──────────────────┘
Root hash:                         [ROOT]

On read of B3:
  1. Compute H3 = hash(B3)
  2. Verify H3 matches stored hash at level 0
  3. Verify level 0 hash block matches level 1 hash
  4. ... up to ROOT
  5. Verify ROOT matches the stored root hash (from trusted source)
```

ルートハッシュはブロックデバイスの外部（カーネルコマンドライン、TPM、または署名付きパーティションテーブル）に格納される。

## dm-verityのディスク上のレイアウト {#dm-verity-on-disk-layout}

```
dm-verity device:
┌───────────────────────────────────┐
│ Data blocks (read-only)           │  ← filesystem or raw data
│ [block 0][block 1]...[block N-1]  │
├───────────────────────────────────┤
│ Hash tree                         │
│ Level L-1 (leaf hashes)           │
│ Level L-2                         │
│ ...                               │
│ Level 0 (root)                    │
└───────────────────────────────────┘
```

ハッシュツリーは同じデバイス上（データの後ろ）に置くことも、別のデバイスに置くこともできる。

## dm-verityのセットアップ {#setting-up-dm-verity}

```bash
# ステップ1: ブロックデバイスをdm-verity用にフォーマットする:
veritysetup format /dev/sdb1 /dev/sdc1   # データデバイス、ハッシュデバイス
# VERITY header information for /dev/sdb1
# UUID:                   abc123-...
# Hash type:              1
# Data blocks:            262144
# Data block size:        4096
# Hash block size:        4096
# Hash algorithm:         sha256
# Salt:                   deadbeef...
# Root hash:              1234567890abcdef...  ← これを保存すること！

# ステップ2: dm-verityをオープンする:
veritysetup open /dev/sdb1 verified-root /dev/sdc1 \
    1234567890abcdef...   # ステップ1のルートハッシュ

# ステップ3: 検証済みデバイスをマウントする:
mount -o ro /dev/mapper/verified-root /mnt

# 改ざんされたブロックがあれば → 読み込みは-EIOを返す（設定によってはパニック）

# ステップ4: 手動で検証する:
veritysetup verify /dev/sdb1 /dev/sdc1 1234567890abcdef...
# Verification successful.
```

## カーネルのdm-verity実装 {#kernel-dm-verity-implementation}

```c
/* drivers/md/dm-verity-target.c */

struct dm_verity {
    struct dm_dev    *data_dev;     /* データブロックデバイス */
    struct dm_dev    *hash_dev;     /* ハッシュツリーデバイス */
    sector_t          data_start;   /* データの開始セクタ */
    sector_t          hash_start;   /* ハッシュツリーの開始セクタ */
    sector_t          data_blocks;  /* データブロック数 */
    unsigned int      data_dev_block_bits;
    unsigned int      hash_dev_block_bits;
    unsigned int      digest_size;  /* ハッシュ出力のサイズ */
    const char       *alg_name;    /* "sha256" */
    u8               *root_digest; /* 信頼されたルートハッシュ */
    u8               *salt;        /* 原像攻撃を防ぐためのソルト */
    unsigned int      salt_size;
    struct crypto_ahash *tfm;      /* ハッシュ変換 */
    /* ... */
};

/* 読み込みのたびに: */
static int verity_verify_io(struct dm_verity_io *io)
{
    struct dm_verity *v = io->v;
    struct bio_vec bv;

    /* bio内の各データブロックについて: */
    while (more_blocks) {
        /* 1. データブロックをハッシュする */
        verity_hash_init(v, io);
        /* io->req = そのブロック用のcrypto_ahash_request */

        /* 2. Merkleツリーのレベルを上にたどる */
        for (level = v->levels - 1; level >= 0; level--) {
            /* hash_devからハッシュブロックを読み込む */
            /* このレベルのハッシュを親に対して検証する */
        }

        /* 3. ルートハッシュを検証する */
        if (memcmp(root_hash, v->root_digest, v->digest_size)) {
            if (v->mode == DM_VERITY_MODE_EIO)
                return -EIO;       /* サイレントな失敗 */
            else if (v->mode == DM_VERITY_MODE_PANIC)
                panic("dm-verity: data corrupted!");
        }
    }
    return 0;
}
```

### エラーモード {#error-modes}

```c
enum verity_mode {
    DM_VERITY_MODE_EIO,          /* 破損時に-EIOを返す（デフォルト） */
    DM_VERITY_MODE_LOGGING,      /* ログに記録して続行 */
    DM_VERITY_MODE_RESTART,      /* 再起動 */
    DM_VERITY_MODE_PANIC,        /* カーネルパニック */
};
```

## Androidにおけるdm-verity {#dm-verity-in-android}

AndroidはAndroid 4.4（KitKat）以降、`/system`パーティションにdm-verityを使っている。

```
Android boot process:
  1. Bootloader verifies kernel + dm-verity metadata signature (AVB)
  2. Kernel boots with dm-verity= parameter:
     dm-verity=1,/dev/sda1,/dev/sda2,4096,4096,N_BLOCKS,HASH_START,sha256,ROOT_HASH,SALT
  3. Init mounts /dev/mapper/verity as /system
  4. Any modified system file → IO error → boot failure

AVB (Android Verified Boot) 2.0:
  - Uses vbmeta partition to store root hashes and signature
  - RSA-PSS or ECDSA signature verified against OEM public key embedded in vbmeta
```

```bash
# Android: verityの状態を確認する
adb shell getprop ro.boot.veritymode
# enforcing

# dm-verityデバイスを確認する:
adb shell dmctl table system
# 0 ...: verity 1 /dev/block/sda1 /dev/block/sda2 4096 4096 ...
```

## fec（前方誤り訂正）付きのdm-verity {#dm-verity-with-fec-forward-error-correction}

```bash
# FECを有効にする: 失敗せずに最大Nバイトの破損を訂正できる:
veritysetup format --fec-device=/dev/sdc2 --fec-roots=2 \
    /dev/sdb1 /dev/sdc1
# FEC roots=2 → データの2*255/255 = 約0.8%を訂正できる

# FEC付きでオープンする:
veritysetup open --fec-device=/dev/sdc2 /dev/sdb1 verified /dev/sdc1 \
    <root_hash>
```

## 可観測性 {#observability}

```bash
# dm-verityの破損統計を確認する:
dmsetup status verified
# 0 524288000 verity V
# → V = エラーなし、C = 破損を検出

# 破損時のカーネルエラーログ:
dmesg | grep -E "dm-verity|verity"
# dm-verity: device 252:0 is corrupted
# dm-verity: hashed block 1234 is corrupted

# verityが強制されているか確認する:
cat /sys/module/dm_verity/parameters/error_behavior
# EIO
```

## dm-thin: シンプロビジョニング {#dm-thin-thin-provisioning}

もう1つの重要なdevice mapperターゲットである — シンプロビジョニングは、物理ストレージより大きな仮想ブロックデバイスの作成を可能にする。

```bash
# シンプールを作成する（メタデータ + データデバイス）:
dmsetup create pool --table \
    "0 20971520 thin-pool /dev/sdb1 /dev/sdb2 128 0"
#   セクタ数   ターゲット メタデータ   データ     チャンクサイズ 最小空き容量

# シンボリュームを作成する（仮想デバイス、物理的な割り当てなし）:
dmsetup message pool 0 "create_thin 0"       # シンID 0
dmsetup create thin0 --table "0 2097152 thin /dev/mapper/pool 0"

# 実際のブロックは最初の書き込み時に割り当てられる（プロビジョニングを伴うcopy-on-write）

# スナップショットを作成する:
dmsetup message pool 0 "create_snap 1 0"   # スナップショットID 1、元のID 0
dmsetup create snap0 --table "0 2097152 thin /dev/mapper/pool 1"
```

## 参考文献 {#further-reading}

- [dm-crypt](../security/dm-crypt.md) — dm-cryptはdm-verityとうまくスタックできる
- [NVMeドライバ](nvme.md) — dm-verityが保護するブロックデバイス
- [BPFネットワーキング](../bpf/bpf-networking.md) — 類似したパイプラインでの介入
- `drivers/md/dm-verity*.c` — dm-verityの実装
- `Documentation/admin-guide/device-mapper/verity.rst`
- AndroidのAVBドキュメント
