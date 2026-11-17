# カーネル暗号API {#kernel-crypto-api}

> 共通鍵暗号、AEAD、ハッシュ関数、ハードウェアアクセラレーション

## 概要 {#overview}

Linuxカーネルの暗号APIは、暗号操作への統一インターフェースを提供する。呼び出し元は名前（例: `"gcm(aes)"`、`"sha256"`）で変換（transform）を要求し、サブシステムは利用可能な最良の実装 — ソフトウェアによるフォールバックかハードウェアアクセラレータ — を選択する。

## struct crypto_alg {#struct-crypto_alg}

すべての暗号アルゴリズムは `struct crypto_alg` を登録する:

```c
/* include/linux/crypto.h */
struct crypto_alg {
    struct list_head    cra_list;
    struct list_head    cra_users;

    u32                 cra_flags;      /* CRYPTO_ALG_TYPE_* */
    unsigned int        cra_blocksize;  /* 暗号のブロックサイズ */
    unsigned int        cra_ctxsize;    /* 変換コンテキストのサイズ */
    unsigned int        cra_alignmask; /* 必要なアライメント */
    unsigned int        cra_reqsize;   /* リクエストごとの追加コンテキストサイズ */

    int                 cra_priority;  /* 高いほど優先（例: AES-NI > ソフトウェア） */
    refcount_t          cra_refcnt;

    char                cra_name[CRYPTO_MAX_ALG_NAME];    /* "aes", "sha256", ... */
    char                cra_driver_name[CRYPTO_MAX_ALG_NAME]; /* "aes-aesni", ... */

    const struct crypto_type *cra_type;

    union {
        struct cipher_alg      cipher;
    } cra_u;
    /* 注: ablkcipher_alg と blkcipher_alg はカーネル5.5で削除された。
     * 現在、すべての共通鍵暗号はskcipher_algを使う。圧縮は現在では
     * 別の変換タイプであり、このunionのメンバーではない。 */

    int (*cra_init)(struct crypto_tfm *tfm);
    void (*cra_exit)(struct crypto_tfm *tfm);
    void (*cra_destroy)(struct crypto_alg *alg);

    struct module          *cra_module;
};
```

## SKCIPHER: 共通鍵暗号 {#skcipher-symmetric-ciphers}

SKCIPHER（symmetric key cipher）は、AES-CBC、AES-XTS、ChaCha20などの共通鍵ブロック暗号向けのAPIである:

```c
#include <crypto/skcipher.h>

/* 暗号変換を割り当てる */
struct crypto_skcipher *tfm = crypto_alloc_skcipher("xts(aes)", 0, 0);
if (IS_ERR(tfm)) {
    pr_err("Failed to allocate xts(aes): %ld\n", PTR_ERR(tfm));
    return PTR_ERR(tfm);
}

/* 鍵を設定（XTSは2倍の鍵が必要: データ鍵 + tweak鍵） */
u8 key[64];  /* データ32バイト + tweak 32バイト = AES-256-XTS */
get_random_bytes(key, sizeof(key));
crypto_skcipher_setkey(tfm, key, sizeof(key));

/* リクエストを割り当てる（操作ごとの状態を保持） */
struct skcipher_request *req = skcipher_request_alloc(tfm, GFP_KERNEL);

/* scatter-gather I/Oを設定 */
struct scatterlist sg;
sg_init_one(&sg, plaintext_buf, buflen);

/* XTSのIV: セクタ番号 */
u8 iv[16] = { 0 };  /* セクタ0 */
skcipher_request_set_crypt(req, &sg, &sg, buflen, iv);

/* インプレースで暗号化（同期） */
int ret = crypto_skcipher_encrypt(req);

/* 復号 */
skcipher_request_set_crypt(req, &sg, &sg, buflen, iv);
ret = crypto_skcipher_decrypt(req);

/* 後始末 */
skcipher_request_free(req);
crypto_free_skcipher(tfm);
```

### よく使われる暗号名 {#common-cipher-names}

```
"cbc(aes)"          AES-CBC (128/192/256 bit key)
"xts(aes)"          AES-XTS (disk encryption)
"ctr(aes)"          AES-CTR (stream cipher mode)
"gcm(aes)"          AES-GCM (AEAD — see below)
"chacha20"          ChaCha20 stream cipher
"chacha20poly1305"  ChaCha20-Poly1305 (AEAD)
"ecb(aes)"          AES-ECB (avoid — deterministic)
```

## AEAD: 認証付き暗号 {#aead-authenticated-encryption}

AEAD（Authenticated Encryption with Associated Data）は、機密性と完全性（AND）を1回の操作で提供する。AES-GCMはカーネルで最も一般的なAEADである:

```c
#include <crypto/aead.h>

struct crypto_aead *tfm = crypto_alloc_aead("gcm(aes)", 0, 0);

/* 鍵（AESでは128、192、256ビット） */
u8 key[32];  /* AES-256-GCM */
crypto_aead_setkey(tfm, key, sizeof(key));

/* 認証タグ長: 16バイト = 128ビットのタグ */
crypto_aead_setauthsize(tfm, 16);

struct aead_request *req = aead_request_alloc(tfm, GFP_KERNEL);

/* scatter-gatherでのレイアウト:
   [関連データ (AAD)] [平文/暗号文] [認証タグ（暗号化時）]

   assoclen = AADの長さ
   cryptlen = 平文の長さ（暗号化時）または暗号文+タグの長さ（復号時） */

struct scatterlist src[2], dst[2];
sg_init_one(&src[0], aad, aadlen);        /* 関連データ */
sg_init_one(&src[1], plaintext, ptlen);    /* 平文 */
sg_init_one(&dst[0], aad, aadlen);        /* そのまま通過するAAD */
sg_init_one(&dst[1], ciphertext, ptlen + 16); /* 暗号文 + 16バイトのタグ */

u8 iv[12];   /* GCMでは96ビットのIVを推奨 */
get_random_bytes(iv, sizeof(iv));

aead_request_set_crypt(req, src, dst, ptlen, iv);
aead_request_set_ad(req, aadlen);

/* 暗号化: 暗号文 + 16バイトの認証タグを出力 */
ret = crypto_aead_encrypt(req);

/* 復号: タグを検証し、認証に失敗すると-EBADMSGを返す */
aead_request_set_crypt(req, src, dst, ptlen + 16, iv);
aead_request_set_ad(req, aadlen);
ret = crypto_aead_decrypt(req);
if (ret == -EBADMSG)
    pr_err("Authentication failed — data corrupted or tampered\n");
```

## ハッシュ関数 {#hash-functions}

```c
#include <crypto/hash.h>

/* 単純なSHA-256ハッシュ */
struct crypto_shash *tfm = crypto_alloc_shash("sha256", 0, 0);

SHASH_DESC_ON_STACK(desc, tfm);  /* descをスタック上に割り当てる */
desc->tfm = tfm;

u8 digest[32];
crypto_shash_init(desc);
crypto_shash_update(desc, data, datalen);
crypto_shash_final(desc, digest);

/* または: ワンショット */
crypto_shash_digest(desc, data, datalen, digest);

crypto_free_shash(tfm);
```

### HMAC {#hmac}

```c
struct crypto_shash *tfm = crypto_alloc_shash("hmac(sha256)", 0, 0);
crypto_shash_setkey(tfm, hmac_key, keylen);

SHASH_DESC_ON_STACK(desc, tfm);
desc->tfm = tfm;

u8 mac[32];
crypto_shash_digest(desc, message, msglen, mac);
```

## 非同期（ahash）API {#asynchronous-ahash-api}

大きなデータやハードウェアオフロードには、非同期ハッシュAPIを使う:

```c
struct crypto_ahash *tfm = crypto_alloc_ahash("sha256", 0, 0);
struct ahash_request *req = ahash_request_alloc(tfm, GFP_KERNEL);

struct scatterlist sg;
sg_init_one(&sg, data, datalen);
ahash_request_set_crypt(req, &sg, digest, datalen);

/* 非同期操作の完了コールバック */
ahash_request_set_callback(req, CRYPTO_TFM_REQ_MAY_BACKLOG,
                            my_hash_complete, &completion);

ret = crypto_ahash_digest(req);
if (ret == -EINPROGRESS)
    wait_for_completion(&completion);  /* ハードウェアが処理中 */
```

## ハードウェアアクセラレーション {#hardware-acceleration}

暗号サブシステムは、ハードウェアアクセラレータが利用可能なら自動的にそれを使う。選択は `cra_priority` フィールドで決まる:

| 優先度 | 実装 |
|----------|---------------|
| 4001 | Intel QATハードウェア |
| 400–800  | Intel AES-GCM、命令セット別: 基本AES-NI（400）、AVX（500）、VAES+AVX2（600）、VAES+AVX512（800） — 実行中のCPUで利用可能な最も高いものが選ばれる |
| 300  | 汎用C実装 |

```bash
# どの実装が選択されているか確認
cat /proc/crypto | grep -A 10 "name.*gcm"
# name         : gcm(aes)
# driver       : generic-gcm-aesni
# module       : kernel
# priority     : 400     ← 基本AES-NIが選択されている（より優先度の高いAVX/VAES版には新しいCPUのサポートが必要）

# 暗号テストスイート全体を実行（CONFIG_CRYPTO_BENCHMARKが必要）
modprobe tcrypt mode=0
```

### Intel AES-NI {#intel-aes-ni}

AES-NIはAESのラウンド用のハードウェア命令を追加する。カーネルの実装はSIMD（SSE/AVX）レジスタ操作を通じてそれを使う:

```c
/* arch/x86/crypto/aesni-intel_glue.c — 実際のcbc_encrypt() */
static int cbc_encrypt(struct skcipher_request *req)
{
    struct crypto_skcipher *tfm = crypto_skcipher_reqtfm(req);
    struct crypto_aes_ctx *ctx = aes_ctx(crypto_skcipher_ctx(tfm));
    struct skcipher_walk walk;
    unsigned int nbytes;
    int err;

    /* これらの呼び出しの前後でカーネルはSIMDレジスタを保存/復元しなければならない */
    err = skcipher_walk_virt(&walk, req, false);
    while ((nbytes = walk.nbytes)) {
        kernel_fpu_begin();   /* FPU状態を保存 */
        aesni_cbc_enc(ctx, walk.dst.virt.addr, walk.src.virt.addr,
                      nbytes & AES_BLOCK_MASK, walk.iv);  /* AESENC命令 */
        kernel_fpu_end();     /* FPU状態を復元 */
        nbytes &= AES_BLOCK_SIZE - 1;
        err = skcipher_walk_done(&walk, nbytes);
    }
    return err;
}
```

SIMDが利用できない場合（例: 一部の割り込みコンテキスト）に使われる `crypto_simd_usable()`/フォールバック分岐は、`cbc_encrypt()` のようなブロック単位の各暗号化関数の中にインラインであるのではなく、1つ上の層の鍵セットアップ（`aes_set_key_common()`）にある。

`kernel_fpu_begin()` は比較的高コスト（約512バイトのSIMD状態を保存する）なので、AES-NIが最も効果を発揮するのは大きなバッファの場合である。

## AF_ALG: ユーザー空間からカーネル暗号へのアクセス {#af_alg-userspace-access-to-kernel-crypto}

AF_ALGソケットはカーネルの暗号機能をユーザー空間に公開する。カーネル自身のドキュメント
（[`userspace-if.rst`](https://docs.kernel.org/crypto/userspace-if.html)、下記に引用）はこれについて率直である: AF_ALGは非推奨であり、「most kernel developers now consider it to be a mistake」（ほとんどのカーネル開発者は現在これを誤りだったと考えている）、
本来の目的（ユーザー空間にハードウェアアクセラレータへのアクセスを提供すること）は削除されており、
最適化されたユーザー空間実装より常に遅く、CVEの履歴もある。新しい
コードは代わりにユーザー空間の暗号ライブラリを使うべきである。このセクションはインターフェースを推奨される手法としてではなく、
存在するものとして記述する。

```c
/* ユーザー空間: AF_ALGソケット経由でカーネルのAES-GCMを使う */
int sockfd = socket(AF_ALG, SOCK_SEQPACKET, 0);

struct sockaddr_alg sa = {
    .salg_family = AF_ALG,
    .salg_type   = "aead",
    .salg_name   = "gcm(aes)",
};
bind(sockfd, (struct sockaddr *)&sa, sizeof(sa));

/* 鍵を設定 */
setsockopt(sockfd, SOL_ALG, ALG_SET_KEY, key, sizeof(key));
setsockopt(sockfd, SOL_ALG, ALG_SET_AEAD_AUTHSIZE, NULL, 16);

/* acceptは操作用のfdを返す */
int opfd = accept(sockfd, NULL, 0);

/* sendmsg/recvmsgで暗号化 */
struct cmsghdr cmsg;
/* ... 補助データでIVとAD長を設定 ... */
sendmsg(opfd, &msg, 0);
recvmsg(opfd, &msg_out, 0);
```

## 暗号サブシステムの観測 {#observing-the-crypto-subsystem}

```bash
# 登録済みの全アルゴリズムと優先度
cat /proc/crypto

# 暗号テストスイート（CONFIG_CRYPTO_TESTが必要）
modprobe tcrypt mode=0   # 全アルゴリズムをテスト（mode=1はMD5のみテスト）
modprobe tcrypt mode=200 # AESのベンチマーク
# testing AES-128-ECB encryption
#  128 bit key:  1234.5 MB/s

# ハードウェア使用率（QATの例）
cat /sys/kernel/debug/qat_*/fw_counters

# 暗号処理のCPUサイクル（perf）
perf stat -e instructions,cycles cryptsetup benchmark
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/crypto.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/crypto.h) — `struct crypto_alg`、`CRYPTO_ALG_TYPE_*`、`cra_*` フラグビット
- [include/crypto/skcipher.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/crypto/skcipher.h) — `crypto_alloc_skcipher()`、`skcipher_request_set_crypt()`、およびSKCIPHER APIの全体
- [include/crypto/aead.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/crypto/aead.h) — `crypto_alloc_aead()`、`aead_request_set_ad()`、`crypto_aead_setauthsize()`
- [include/crypto/hash.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/crypto/hash.h) — `crypto_alloc_shash()`/`crypto_alloc_ahash()`、`SHASH_DESC_ON_STACK`、`ahash_request_set_crypt()`
- [arch/x86/crypto/aesni-intel_glue.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/crypto/aesni-intel_glue.c) — 実際のAES-NI SKCIPHER実装（`cbc_encrypt()`、`ecb_encrypt()`、`DEFINE_GCM_ALGS()` によるAEADファミリー）と、それらが登録する `cra_priority` 値
- [Documentation/crypto/userspace-if.rst](https://docs.kernel.org/crypto/userspace-if.html) — AF_ALGのユーザー空間インターフェース。このカーネル時点では、冒頭に明示的な非推奨の告知がある（下記の注を参照）

### 関連ページ {#related-pages}

- [dm-cryptとfscrypt](encryption.md) — ストレージ暗号化におけるSKCIPHER/AEADの主要な利用者
- [crypto_engine: ハードウェアオフロードフレームワーク](crypto-engine.md) — DMAベースのハードウェアアクセラレータが、このページで説明した `cra_priority` による選択にどう組み込まれるか
- [カーネルkeyring](keyring.md) — `crypto_*_setkey()` に渡される鍵素材が通常格納される場所
- [乱数生成](rng.md) — `crypto_alloc_rng()` と、この同じAPIを通じて公開されるDRBGアルゴリズム
- [暗号の事件簿](war-stories.md) — IVの再利用、タイミングサイドチャネル、AES-GCMの出力を黙って破損させたハードウェアアクセラレータ

### LWN記事 {#lwn-articles}

- [A netlink-based user-space crypto API](https://lwn.net/Articles/410763/) — Jake Edge、2010年10月20日: このページで扱った `AF_ALG` アドレスファミリーに関するHerbert Xuの当初の提案
- [WireGuard and the crypto API](https://lwn.net/Articles/802376/) — Jake Edge、2019年10月16日: 性能が重要でコンパイル時に既知の変換に対して、抽象的なAEAD/SKCIPHERインターフェースを用いることへのJason DonenfeldとLinus Torvaldsの批判
- [Progress in modernizing kernel cryptography](https://lwn.net/Articles/1077427/) — Joe Brockmeier、2026年7月8日: このページで説明した `struct crypto_alg`/SKCIPHER/AEADモデルのオーバーヘッドと、よりシンプルな `lib/crypto` ライブラリ関数への移行が進んでいることについてのEric Biggersの話

### 外部資料 {#external}

- [NIST SP 800-38D: Recommendation for Block Cipher Modes of Operation — GCM and GMAC](https://csrc.nist.gov/pubs/sp/800/38/d/final) — `"gcm(aes)"` の背後にある仕様。上記のAEADの例で使った96ビットIVの推奨もこれによる
