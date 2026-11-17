# カーネルTLS（kTLS） {#kernel-tls-ktls}

> ゼロコピーHTTPSのためのカーネル内TLSレコード層

## kTLSが解決する問題 {#the-problem-ktls-solves}

従来、TLSは完全にユーザー空間（OpenSSL）で動作する:

```
Application → plaintext → OpenSSL encrypt → ciphertext → write() syscall → kernel → NIC
```

これは次のことを意味する:

- TLSレコードごとにコピーが発生する: ユーザーバッファ → カーネルのソケットバッファ
- `sendfile()`が使えない: 暗号化のためにデータがユーザー空間を通過しなければならない
- NICへのTLSオフロードには、いずれにせよカーネルの関与が必要になる

kTLSはTLSレコード層をカーネル内に移す:

```
Application → plaintext → write()/sendfile() → kernel TLS encrypt → NIC
```

利点:

- TLSで`sendfile()`が使える: ファイル → TLS → NICのゼロコピー
- NICによるTLSオフロード: カーネルが暗号処理をハードウェアに任せられる
- TLS中心のワークロード（nginx、envoy）でのCPU使用率の低減

## ULP: 上位層プロトコル {#ulp-upper-layer-protocol}

kTLSはULP（Upper Layer Protocol）として実装されている。ULPは、アプリケーションとTCPの間で送受信を横取りするソケット層である:

```
Application
    │ write()/sendmsg()
    ▼
[ULP layer: TLS record framing + encryption]
    │
    ▼
TCP layer
    │
    ▼
NIC
```

## kTLSのセットアップ {#setting-up-ktls}

kTLSのセットアップは、（例えばOpenSSLによる）TLSハンドシェイクの完了後に行う:

```c
#include <linux/tls.h>

/* SSL_accept()/SSL_connect()の完了後: */

/* ステップ1: ソケット上でkTLS ULPを有効化する */
setsockopt(sockfd, SOL_TCP, TCP_ULP, "tls", sizeof("tls"));

/* ステップ2: ハンドシェイクでネゴシエートされた共通鍵を取得する
   （OpenSSLはこれをSSL_get_current_cipher、SSL_CTX_set_keylog_callback、
    またはktls OpenSSLエンジン経由で公開している） */

/* ステップ3: TXの暗号情報を設定する */
struct tls12_crypto_info_aes_gcm_128 crypto_info;
crypto_info.info.version     = TLS_1_3_VERSION;  /* またはTLS_1_2_VERSION */
crypto_info.info.cipher_type = TLS_CIPHER_AES_GCM_128;
memcpy(crypto_info.iv,  tx_iv,  sizeof(crypto_info.iv));
memcpy(crypto_info.key, tx_key, sizeof(crypto_info.key));
memcpy(crypto_info.salt, tx_salt, sizeof(crypto_info.salt));
memcpy(crypto_info.rec_seq, tx_seq, sizeof(crypto_info.rec_seq));

setsockopt(sockfd, SOL_TLS, TLS_TX, &crypto_info, sizeof(crypto_info));

/* ステップ4: RXの暗号情報を設定する（オプション — カーネル内で復号する場合） */
/* ... rx_key/iv/seqで同様に ... */
setsockopt(sockfd, SOL_TLS, TLS_RX, &crypto_info, sizeof(crypto_info));

/* これでwrite()/sendfile()は自動的にTLSレコードとして暗号化される */
sendfile(sockfd, file_fd, NULL, file_size);
/* ↑ ゼロコピー: ファイルのページ → TLS暗号化してNICへ、ユーザー空間へのコピーなし */
```

## サポートされている暗号スイート {#supported-cipher-suites}

```c
/* include/uapi/linux/tls.h */
TLS_CIPHER_AES_GCM_128
TLS_CIPHER_AES_GCM_256
TLS_CIPHER_AES_CCM_128
TLS_CIPHER_CHACHA20_POLY1305

TLS_1_2_VERSION
TLS_1_3_VERSION
```

## カーネル内のkTLS {#ktls-in-the-kernel}

```c
/* net/tls/tls_sw.c: TLSのソフトウェア実装 */

/* TXパス: sendmsgを横取りする */
static int tls_sw_sendmsg(struct sock *sk, struct msghdr *msg, size_t size)
{
    struct tls_context *tls_ctx = tls_get_ctx(sk);
    struct tls_sw_context_tx *ctx = tls_sw_ctx_tx(tls_ctx);

    /* 平文をTLSレコード（最大16KB）にまとめる */
    while (msg_data_left(msg)) {
        /* 平文をレコードバッファにコピーする */
        copied = sk_msg_memcopy_from_iter(sk, &msg->msg_iter,
                                           msg, try_to_copy);

        /* レコードが満杯になったかMSG_EOMなら: 暗号化して送信する */
        if (full_record || eor) {
            rc = tls_push_record(sk, msg->msg_flags, record_type);
        }
    }
    return copied;
}

static int tls_push_record(struct sock *sk, int flags, unsigned char record_type)
{
    struct tls_context *ctx = tls_get_ctx(sk);

    /* TLSレコードヘッダを構築する */
    /* AEADで暗号化する: 平文 → 暗号文 + 認証タグ */
    rc = tls_do_encryption(sk, ctx, ctx->sw_tx, aead_req, data_len, i);

    /* 暗号化したレコードをTCPに渡す */
    return tls_push_sg(sk, ctx, &sg_array[0], 0, flags);
}
```

## NICによるTLSオフロード {#nic-tls-offload}

TLSオフロードをサポートするハードウェアでは、kTLSは暗号処理をNICに任せられる:

```
Without NIC offload:    CPU encrypts → NIC sends ciphertext
With NIC offload:       CPU sends plaintext → NIC encrypts and sends
```

```c
/* net/tls/tls_device.c */
static int tls_device_push_pending_record(struct sock *sk, int flags)
{
    struct tls_context *ctx = tls_get_ctx(sk);
    struct tls_offload_context_tx *offload_ctx = tls_offload_ctx_tx(ctx);

    /* NICドライバはtls_dev_add/del操作を登録する */
    /* TCPセグメントはTLSメタデータ（seqno、レコードのシーケンス番号）付きで送信される */
    /* NICがハードウェアで暗号化する */
    return tls_push_sg(sk, ctx, NULL, 0, flags);
}
```

NICによるTLSオフロードのサポート: Mellanox ConnectX-6 Dx（ConnectX-5はTLSオフロードをサポートしない）、Intel E810、一部のChelsio NIC。

```bash
# NICのTLSオフロードを有効化（ドライバのサポートが必要）
ethtool -K eth0 tls-hw-tx-offload on
ethtool -K eth0 tls-hw-rx-offload on

# オフロードの状態を確認
ethtool -k eth0 | grep tls
# tls-hw-tx-offload: on
# tls-hw-rx-offload: on
```

## HTTPSのためのゼロコピーsendfile {#zero-copy-sendfile-for-https}

kTLSの主なユースケースは、ゼロコピーでのファイル配信である:

```c
/* kTLSで静的ファイルを配信するHTTPSサーバー */
int setup_ktls_server(int sockfd, SSL *ssl)
{
    /* ハンドシェイク後: kTLSをインストールする */
    setsockopt(sockfd, SOL_TCP, TCP_ULP, "tls", sizeof("tls"));
    /* ... SSLからTXの暗号情報をインストールする ... */
}

void serve_file(int sockfd, const char *path)
{
    int fd = open(path, O_RDONLY);
    struct stat st;
    fstat(fd, &st);

    /* ゼロコピー: ページキャッシュ → TLS暗号化 → NIC */
    /* ユーザー空間バッファへのコピーは不要 */
    sendfile(sockfd, fd, NULL, st.st_size);
    close(fd);
}
```

従来のHTTPSの流れ:
```
disk → page cache → read() → user buffer → SSL_write() → OpenSSL encrypt → write() → kernel → NIC
        (4 copies)
```

kTLS + sendfileの流れ:
```
disk → page cache → sendfile() → kTLS encrypt → NIC
        (1 copy, or 0 with DMA)
```

## RXパス: カーネル内での復号 {#rx-path-in-kernel-decrypt}

`TLS_RX`が設定されていると、読み取りはカーネル内で復号される:

```c
/* アプリケーション: */
char buf[4096];
int n = read(sockfd, buf, sizeof(buf));
/* bufには復号された平文が入る — TLSレコード層はカーネルが処理する */

/* オプション: レコードタイプのための制御メッセージ */
struct msghdr msg;
char cmsg_buf[CMSG_SPACE(sizeof(unsigned char))];
/* ... */
recvmsg(sockfd, &msg, 0);
/* SOL_TLS、TLS_GET_RECORD_TYPEのCMSG: アラートやハンドシェイクメッセージを確認する */
```

## kTLSの観察 {#observing-ktls}

```bash
# ソケット上でkTLSが有効か確認
ss -tinHp | grep -i tls

# TLSの統計
cat /proc/net/tls_stat
# TlsCurrTxSw        5       # SW TXを使っているソケット
# TlsCurrRxSw        3       # SW RXを使っているソケット
# TlsCurrTxDevice    2       # NIC TXオフロードを使っているソケット
# TlsTxSoftware      12345   # SW TXレコードの総数
# TlsDecryptError    0       # 認証の失敗

# ソケットごとのTLS情報（iproute2 >= 5.9のssが必要）
ss -tinH --tcp state established

# nginxのkTLS設定
# nginx.conf: ssl_sendfile on;  （kTLSでsendfileを使う）
```

## 参考文献 {#further-reading}

- [TCPの実装](tcp.md) — kTLSが統合されるソケット層
- [ネットワークデバイスとNAPI](napi.md) — NICのTLSオフロードでの受信
- [Crypto: カーネル暗号API](../crypto/crypto-api.md) — TLSレコードに使われるAEAD
- [io_uring: 操作](../io-uring/io-uring-ops.md) — kTLSと組み合わせたio_uringのsend_zc
- カーネルツリーの`net/tls/` — kTLSの実装
- カーネルツリーの`Documentation/networking/tls.rst`
