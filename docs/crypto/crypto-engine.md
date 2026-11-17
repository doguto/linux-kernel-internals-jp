# crypto_engine: ハードウェアオフロードフレームワーク {#crypto_engine-hardware-offload-framework}

> ハードウェア暗号アクセラレータがカーネルにどう組み込まれるか

## crypto_engineが解決する問題 {#the-problem-crypto_engine-solves}

ハードウェア暗号アクセラレータ — STM32 CRYP、Allwinner CE、Marvell CESA、Intel QAT — はいずれも同じ基本的なI/Oモデルを共有している: DMAディスクリプタをプログラムし、ハードウェアがそれを非同期に処理し、終わると割り込みを受け取る。ハードウェアのキューイング、フォールバック、エラー回復を管理しながら、カーネル暗号APIの `do_one_request()` の契約を正しく満たすドライバを書くには、大量の定型コードが必要になる。

`crypto_engine`（カーネル3.19で導入）は、暗号APIとハードウェアドライバの間に位置する汎用のワークキュー層を提供する。これは次のことを行う:

- 一度に1つしか処理できないハードウェアへのリクエストを直列化する
- ハードウェアがビジーのときのリトライを扱う（`engine->retry_support = true` で有効化）
- 適切なタイミングでドライバのコールバックを呼ぶ（prepare → do → finalize）
- 非同期リクエストの完了パスと統合する

crypto_engineがなければ、各ドライバが独自のキュー、独自のリトライロジック、独自のロックを実装することになり、間違えやすいコードが何百行も重複する。

```
Caller (IPsec, dm-crypt, TLS)
        │
        │ crypto_skcipher_encrypt(req)  ← async request
        ▼
  Kernel Crypto API (crypto/skcipher.c)
        │
        │ alg->encrypt(req)
        ▼
  crypto_engine work queue
  (crypto/engine.c)
        │  ← serialized, one request at a time
        │  1. driver->do_one_request(engine, req)
        │     ↓ programs DMA, returns -EINPROGRESS
        │
  Hardware DMA + interrupt
        │
        │ IRQ handler calls:
        │  crypto_finalize_skcipher_request(engine, req, err)
        ▼
  Caller's completion callback
```

## struct crypto_engine {#struct-crypto_engine}

ドライバ内部向けヘッダー `include/crypto/internal/engine.h`（公開ヘッダーの `include/crypto/engine.h` ではない）で定義されている:

```c
/* include/crypto/internal/engine.h */
struct crypto_engine {
    char                    name[ENGINE_NAME_LEN];
    bool                    busy;         /* リクエストポンプがビジー */
    bool                    running;

    bool                    retry_support;
    bool                    rt;           /* ポンプをリアルタイムタスクとして実行する */

    struct list_head        list;
    spinlock_t              queue_lock;
    struct crypto_queue     queue;              /* 保留中のリクエスト */
    struct device           *dev;

    struct kthread_worker   *kworker;
    struct kthread_work     pump_requests;      /* キューをポンプするワークアイテム */

    void                    *priv_data;
    struct crypto_async_request *cur_req;
};
```

専用のkthreadワーカー（`kworker`）が `pump_requests` を実行し、リクエストをデキューしてハードウェアにディスパッチする — 汎用のworkqueueではない。ほとんどのドライバはハードウェアチャネルごとに1つのエンジンを割り当てるか、ハードウェアがシングルチャネルならデバイスごとに1つ割り当てる。この構造体にはバッチ処理用のコールバックはない。後述の「バッチ処理」を参照。

## ドライバのコールバック: crypto_engine_op {#driver-callbacks-crypto_engine_op}

crypto_engineに登録される各アルゴリズムは `struct crypto_engine_op` を提供する:

```c
/* include/crypto/engine.h (kernel 5.13+) */
struct crypto_engine_op {
    int (*do_one_request)(struct crypto_engine *engine,
                          void *areq);
};
```

| コールバック | 呼ばれるタイミング | 典型的な処理 |
|---|---|---|
| `do_one_request` | ハードウェアがアイドルで、リクエストがキューの先頭にあるとき | DMAを開始し、`-EINPROGRESS` を返す |

`do_one_request` は必須であり、現在のカーネルでは唯一のコールバックである。

> **注**: カーネル6.5までは、`struct crypto_engine_op` は（リクエストタイプ別に分かれていない）単一の汎用構造体で、`prepare_request`、`unprepare_request`、`do_one_request` の3つのコールバックを持っていた。現在のモデル — `do_one_request` のみで、`skcipher_engine_alg` のようなタイプ別の構造体にラップされる — へのリファクタリングは6.5と6.6の間に入った。

## アルゴリズムの登録: skcipher_engine_alg {#algorithm-registration-skcipher_engine_alg}

エンジンのopは、変換ごとのコンテキスト構造体ではなく、アルゴリズム記述子の構造体に埋め込まれる。たとえば `struct skcipher_engine_alg` は、`struct skcipher_alg` に `struct crypto_engine_op op` を付け加えてラップしたものである:

```c
/* include/crypto/engine.h */
struct skcipher_engine_alg {
    struct skcipher_alg     base;
    struct crypto_engine_op op;
};
```

同じパターンがAEAD（`struct aead_engine_alg`）、ahash（`struct ahash_engine_alg`）、akcipher（`struct akcipher_engine_alg`）にも当てはまる。ドライバはエンジンのopをアルゴリズム記述子の中に埋め込み、通常の暗号APIのヘルパーではなく、エンジン対応のヘルパー（例: `crypto_engine_register_skcipher()`）で登録する。

## ドライバがcrypto_engineを使う方法 {#how-a-driver-uses-crypto_engine}

### ステップ1: probeでエンジンを割り当てて開始する {#step-1-allocate-and-start-the-engine-in-probe}

```c
/* drivers/crypto/mydrv.c */
struct mydrv_dev {
    struct device           *dev;
    void __iomem            *base;
    struct crypto_engine    *engine;
    struct clk              *clk;
    /* ... */
};

static int mydrv_probe(struct platform_device *pdev)
{
    struct mydrv_dev *dd;

    dd = devm_kzalloc(&pdev->dev, sizeof(*dd), GFP_KERNEL);
    dd->dev = &pdev->dev;

    /* エンジンを割り当てて初期化する */
    dd->engine = crypto_engine_alloc_init(&pdev->dev, true);
    if (!dd->engine)
        return -ENOMEM;

    /* 任意: リトライサポートを有効化（ハードウェアがビジーならエンジンが再キューイングする） */
    dd->engine->retry_support = true;

    /* エンジンのワークキューを開始する */
    ret = crypto_engine_start(dd->engine);
    if (ret)
        goto err_engine;

    /* アルゴリズムを登録 ... */
    return 0;

err_engine:
    crypto_engine_exit(dd->engine);
    return ret;
}

static int mydrv_remove(struct platform_device *pdev)
{
    struct mydrv_dev *dd = platform_get_drvdata(pdev);
    crypto_engine_stop(dd->engine);
    crypto_engine_exit(dd->engine);
    return 0;
}
```

### ステップ2: crypto_engineのヘルパーでアルゴリズムを登録する {#step-2-register-algorithms-via-crypto_engine-helpers}

`crypto_register_skcipher()` の代わりに、エンジン対応のラッパーを使う:

```c
static struct skcipher_engine_alg mydrv_aes_algs[] = {
    {
        .base = {
            .base = {
                .cra_name        = "cbc(aes)",
                .cra_driver_name = "mydrv-cbc-aes",
                .cra_priority    = 300,
                .cra_flags       = CRYPTO_ALG_ASYNC | CRYPTO_ALG_KERN_DRIVER_ONLY,
                .cra_blocksize   = AES_BLOCK_SIZE,
                .cra_ctxsize     = sizeof(struct mydrv_ctx),
                .cra_module      = THIS_MODULE,
            },
            .min_keysize = AES_MIN_KEY_SIZE,
            .max_keysize = AES_MAX_KEY_SIZE,
            .ivsize      = AES_BLOCK_SIZE,
            .setkey      = mydrv_aes_setkey,
            .encrypt     = mydrv_aes_encrypt,   /* crypto_transfer_skcipher_request_to_engine()でエンキューする */
            .decrypt     = mydrv_aes_decrypt,
            .init        = mydrv_aes_init,
            .exit        = mydrv_aes_exit,
        },
        .op = {
            .do_one_request    = mydrv_do_one_request,
        },
    },
};

/* probe内で、エンジン開始後に（各アルゴリズムを個別に登録する）: */
ret = crypto_engine_register_skcipher(&mydrv_aes_algs[0]);
```

### ステップ3: encrypt/decryptのエントリポイントがリクエストをエンキューする {#step-3-the-encryptdecrypt-entry-points-enqueue-the-request}

```c
/* 呼び出し元がcrypto_skcipher_encrypt()を行うと暗号APIから呼ばれる */
static int mydrv_aes_encrypt(struct skcipher_request *req)
{
    struct mydrv_ctx *ctx = crypto_skcipher_ctx(
                                crypto_skcipher_reqtfm(req));
    /*
     * リクエストをエンジンのキューに引き渡す。
     * キューに入れられた場合は直ちに-EINPROGRESSを返す。
     * 同期的に完了した場合は0を返す（ハードウェアでは珍しい）。
     */
    return crypto_transfer_skcipher_request_to_engine(ctx->dd->engine, req);
}

static int mydrv_aes_decrypt(struct skcipher_request *req)
{
    struct mydrv_ctx *ctx = crypto_skcipher_ctx(
                                crypto_skcipher_reqtfm(req));
    return crypto_transfer_skcipher_request_to_engine(ctx->dd->engine, req);
}
```

### ステップ4: do_one_requestがハードウェアをプログラムする {#step-4-do_one_request-programs-the-hardware}

```c
static int mydrv_do_one_request(struct crypto_engine *engine, void *areq)
{
    struct skcipher_request *req = skcipher_request_cast(areq);
    struct mydrv_ctx *ctx = crypto_skcipher_ctx(
                                crypto_skcipher_reqtfm(req));
    struct mydrv_dev *dd = ctx->dd;

    /* DMAを設定: ハードウェアFIFOとの間のscatter-gather */
    ret = dma_map_sg(dd->dev, req->src, sg_nents(req->src), DMA_TO_DEVICE);
    ret = dma_map_sg(dd->dev, req->dst, sg_nents(req->dst), DMA_FROM_DEVICE);

    /* ハードウェアレジスタをプログラムする */
    /* 注: req->ivは仮想ポインタなので、dma_map_single()でDMAアドレスを得る */
    dma_addr_t iv_dma = dma_map_single(dd->dev, req->iv, crypto_skcipher_ivsize(...),
                                        DMA_TO_DEVICE);
    writel(MYDRV_CTRL_START | MYDRV_CTRL_CBC, dd->base + MYDRV_CTRL);
    writel(ctx->key_phys, dd->base + MYDRV_KEY_ADDR);
    writel(iv_dma,        dd->base + MYDRV_IV_ADDR);

    /* DMAを開始 */
    mydrv_start_dma(dd, req->src, req->dst, req->cryptlen);

    /* これでハードウェアは非同期に動作している。
     * -EINPROGRESSを返し、呼び出し元の完了コールバックが
     * 後で割り込みコンテキストから呼ばれることを示す。 */
    return -EINPROGRESS;
}
```

### ステップ5: 割り込みハンドラから完了させる {#step-5-complete-from-the-interrupt-handler}

```c
static irqreturn_t mydrv_irq(int irq, void *dev_id)
{
    struct mydrv_dev *dd = dev_id;
    u32 status = readl(dd->base + MYDRV_STATUS);

    if (!(status & MYDRV_STATUS_DONE))
        return IRQ_NONE;

    /* 割り込みを確認応答する */
    writel(MYDRV_STATUS_DONE, dd->base + MYDRV_STATUS);

    /* DMAのマッピングを解除 */
    dma_unmap_sg(dd->dev, dd->cur_req->src, ...);
    dma_unmap_sg(dd->dev, dd->cur_req->dst, ...);

    /* このリクエストが完了したことをエンジンに伝える。
     * 成功ならerr = 0、ハードウェアエラーなら負のerrno。
     * これにより元の要求者の完了コールバックが呼ばれ、
     * 続いてキューから次のリクエストがポンプされる。 */
    crypto_finalize_skcipher_request(dd->engine, dd->cur_req,
                                      (status & MYDRV_STATUS_ERR) ? -EIO : 0);

    return IRQ_HANDLED;
}
```

`crypto_finalize_skcipher_request()`（およびAEAD、ahash、akcipher用の同等の関数）は、リクエストの完了コールバックを呼び出し、その後エンジンを起動してキュー内の次のリクエストをポンプさせる。

## フォールバックパターン {#the-fallback-pattern}

ハードウェアアクセラレータには制約があることが多い: すべての鍵サイズやすべてのモードをサポートしていないことがあり、利用できないこともある（例: サスペンド中）。標準的なパターンは、ソフトウェアの `fallback` 変換を保持しておき、ハードウェアがリクエストを処理できないときにそれを使うことである。

```c
struct mydrv_ctx {
    struct mydrv_dev        *dd;
    struct crypto_skcipher  *fallback;  /* ソフトウェアのAES-CBC */
    u8                       key[AES_MAX_KEY_SIZE];
    unsigned int             keylen;
};

static int mydrv_aes_init(struct crypto_skcipher *tfm)
{
    struct mydrv_ctx *ctx = crypto_skcipher_ctx(tfm);

    /* サポートされないリクエスト用にソフトウェアのフォールバックを割り当てる */
    ctx->fallback = crypto_alloc_skcipher("cbc(aes)", 0,
                                           CRYPTO_ALG_NEED_FALLBACK);
    if (IS_ERR(ctx->fallback))
        return PTR_ERR(ctx->fallback);

    /* リクエストサイズにフォールバックのリクエストサイズが含まれるようにする */
    crypto_skcipher_set_reqsize(tfm, sizeof(struct mydrv_req) +
                                 crypto_skcipher_reqsize(ctx->fallback));
    return 0;
}

static void mydrv_aes_exit(struct crypto_skcipher *tfm)
{
    struct mydrv_ctx *ctx = crypto_skcipher_ctx(tfm);
    crypto_free_skcipher(ctx->fallback);
}

static int mydrv_aes_setkey(struct crypto_skcipher *tfm,
                             const u8 *key, unsigned int keylen)
{
    struct mydrv_ctx *ctx = crypto_skcipher_ctx(tfm);
    int ret;

    /* 128ビット鍵しかサポートしないハードウェアもある */
    if (keylen != AES_KEYSIZE_128)
        ctx->use_fallback = true;
    else
        ctx->use_fallback = false;

    memcpy(ctx->key, key, keylen);
    ctx->keylen = keylen;

    /* フォールバックにも常に鍵を設定する */
    crypto_skcipher_clear_flags(ctx->fallback, CRYPTO_TFM_REQ_MASK);
    crypto_skcipher_set_flags(ctx->fallback,
        crypto_skcipher_get_flags(tfm) & CRYPTO_TFM_REQ_MASK);
    return crypto_skcipher_setkey(ctx->fallback, key, keylen);
}

static int mydrv_aes_encrypt(struct skcipher_request *req)
{
    struct mydrv_ctx *ctx = crypto_skcipher_ctx(
                                crypto_skcipher_reqtfm(req));

    if (ctx->use_fallback) {
        /* 自分のreqデータの後ろに格納されたフォールバックのサブリクエストを使う */
        struct skcipher_request *subreq = skcipher_request_ctx(req);
        skcipher_request_set_tfm(subreq, ctx->fallback);
        skcipher_request_set_callback(subreq, req->base.flags,
                                       req->base.complete, req->base.data);
        skcipher_request_set_crypt(subreq, req->src, req->dst,
                                    req->cryptlen, req->iv);
        return crypto_skcipher_encrypt(subreq);
    }

    return crypto_transfer_skcipher_request_to_engine(ctx->dd->engine, req);
}
```

同じパターンがAEAD（`struct aead_engine_alg`、`crypto_finalize_aead_request()`）、ahash、akcipherにも当てはまる。

## リトライサポート {#retry-support}

`engine->retry_support = true` のとき、`crypto/crypto_engine.c` の `crypto_pump_requests()` は、`do_one_request()` の失敗をエラーに応じて異なる方法で扱う:

- **`-ENOSPC`**: 失敗した直後のその1つのリクエストだけが、順序を保つためにエンジンキューの *先頭* に再キューイングされ（`crypto_enqueue_request_head()`）、次のサイクルでリトライするようにポンプが再スケジュールされる。これは、持続的な負荷のもとでハードウェアのコマンドFIFOが満杯になりうるドライバ（例: Marvell CESA、Allwinner CE）に対応するためのものである。
- **その他のエラー**: リトライサポートの有無にかかわらず、リクエストは直ちに失敗する（`crypto_request_complete()`）。`CRYPTO_TFM_REQ_MAY_BACKLOG` フラグはこのリトライパスとは無関係である — このフラグが制御するのは、すでに満杯のキューにリクエストが最初に投入されたときのバックログ動作であり、`do_one_request()` が失敗した後に何が起こるかではない。

リトライサポートがない場合（あるいは `-ENOSPC` 以外のエラーの場合）、ハードウェアの失敗によってリクエストは直ちに失敗し、呼び出し元にエラーが返る。

## バッチ処理 {#batching}

`crypto_engine` にはバッチ処理用のコールバックはない — `struct crypto_engine` には `do_batch_requests` のようなフックはなく、`crypto_pump_requests()` は常に `do_one_request()` 1回の呼び出しにつき厳密に1つのリクエストをディスパッチする。1回のDMAパスで複数のリクエストを投入できるハードウェア（例: QAT）は、`crypto_engine` の外で独自のバッチ処理を実装するか、そうでなければそれを通して1つずつリクエストを処理する。

## crypto_engineと暗号APIへの直接実装の比較 {#crypto_engine-vs-writing-directly-to-the-crypto-api}

| シナリオ | crypto_engineを使う | 直接書く |
|---|---|---|
| ハードウェアがDMAベースで、IRQ経由で非同期に完了する | はい | — |
| ハードウェアが一度に1つの操作しかできない | はい | — |
| ハードウェアが深いコマンドキュー（例: 64スロット）を持つ | 任意 | 代わりにドライバが独自のキューを管理できる |
| 純粋なソフトウェアアルゴリズム（DMAなし） | いいえ | はい（`crypto_register_skcipher`） |
| SIMDで高速化（AES-NI、ARM CE） | いいえ | はい（kernel_fpu_beginを使う） |
| 独自のスケジューラを持つPCIeオフロードカード | 任意 | そのほうが良い場合もある |

## 実際のドライバの例 {#real-driver-examples}

| ドライバ | ソース | 注目すべきパターン |
|---|---|---|
| stm32-cryp | `drivers/crypto/stm32/stm32-cryp.c` | シングルチャネル、完全なフォールバック、CBCでのIVのローテーション |
| sun8i-ce | `drivers/crypto/allwinner/sun8i-ce/` | 複数アルゴリズムでcrypto_engineを使う、scatter-gather |
| marvell/cesa | `drivers/crypto/marvell/cesa/` | `crypto_engine` を *使わない* — TDMAチェーンに基づく独自のキューイング |
| bcm2835 | `drivers/crypto/bcm/cipher.c` | `crypto_engine` を *使わない* — mailbox（mbox）フレームワーク経由でSPUコプロセッサに投入する |

## crypto_engineの観測 {#observing-crypto_engine}

```bash
# どのアルゴリズムがASYNC（ハードウェア支援）かを確認
cat /proc/crypto | grep -E "^(name|type|async)"
# async        : yes   ← ハードウェアで高速化されている

# 暗号操作のカーネルtracepoint
perf trace -e crypto:*

# 特定のドライバについてはdebugfsを確認
ls /sys/kernel/debug/
# ここにカウンタ（例: リクエスト数、フォールバック回数）をエクスポートするドライバもある

# ハードウェアアルゴリズムに対して暗号テストスイートを実行
# 注: mode=1はMD5のみをテストする。全アルゴリズムのテストを実行するにはmode=0を使う。
modprobe tcrypt mode=0
# これはハードウェアのものも含め、登録済みの全アルゴリズムを実行する
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [crypto/crypto_engine.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/crypto/crypto_engine.c) — エンジンの実装: `crypto_pump_requests()`、リトライ/バックログのロジック、`crypto_engine_register_*()` ヘルパー
- [include/crypto/engine.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/crypto/engine.h) — 公開API: `struct crypto_engine_op`、`*_engine_alg` ラッパー構造体、`crypto_transfer_*_request_to_engine()`、`crypto_finalize_*_request()`
- [include/crypto/internal/engine.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/crypto/internal/engine.h) — 実際の `struct crypto_engine` の定義（ドライバ内部向けヘッダー）
- [Documentation/crypto/crypto_engine.rst](https://docs.kernel.org/crypto/crypto_engine.html) — カーネルのドキュメント。依然として古い `enginectx`/`prepare_cipher_request` の設計を記述しており、現在のコードの `crypto_engine_op`/`do_one_request` モデルに合わせて更新されていない点に注意

### 関連ページ {#related-pages}

- [カーネル暗号API](crypto-api.md) — `crypto_engine` がその下に位置するSKCIPHER、AEAD、ahashのインターフェース
- [dm-cryptとfscrypt](encryption.md) — ハードウェア暗号の利用者
- [暗号の事件簿](war-stories.md) — 事件5では、暗号文を黙って破損させたハードウェアアクセラレータと、それを回避するために使われたフォールバックパターンを扱う
- [メモリ管理: DMA](../mm/dma.md) — `do_one_request()` の実装が使うscatter-gatherとDMAマッピング

### LWN記事 {#lwn-articles}

- [LWN: Asynchronous crypto](https://lwn.net/Articles/109475/)（2004年11月3日） — 非同期の完了コールバックを用いて暗号リクエストをハードウェアアクセラレータにキューイングすることについての当初の設計議論。`crypto_engine` が実装する `-EINPROGRESS`/完了コールバックモデルの概念上の祖先である
