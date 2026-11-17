# 暗号サブシステム {#crypto-subsystem}

> 暗号アルゴリズム、カーネルの鍵管理、ストレージ暗号化

## 暗号スタック {#the-crypto-stack}

```
User space:    openssl / libgcrypt / kernel keyctl
                         │
      ┌──────────────────┼───────────────────────┐
      │                  │                       │
  AF_ALG socket     kernel keyring           dm-crypt / fscrypt
  (access kernel    (key storage)            (disk/file encryption)
   crypto from       struct key
   userspace)            │
      │                  │
      └──────────┬───────┘
                 │
         Kernel Crypto API
         (crypto/*)
         struct crypto_alg
         ┌───────────────────────────────┐
         │ SKCIPHER  AEAD  HASH  AKCIPHER│
         │ (AES-XTS) (AES-GCM) (SHA-256) │
         └───────────────────────────────┘
                 │
         Hardware acceleration
         Intel AES-NI / QAT / ARM CE
```

## このセクションのページ {#pages-in-this-section}

| ページ | 扱う内容 |
|------|----------------|
| [カーネル暗号API](crypto-api.md) | struct crypto_alg、SKCIPHER、AEAD、ハッシュ、ハードウェアオフロード |
| [dm-cryptとfscrypt](encryption.md) | ブロックレベル暗号化とファイルレベル暗号化、keyringとの統合 |
| [crypto_engine: ハードウェアオフロードフレームワーク](crypto-engine.md) | DMAベースのハードウェアアクセラレータ、ドライバコールバック、フォールバックパターン |
| [カーネルkeyring](keyring.md) | 鍵タイプ（user、logon、trusted、encrypted）、TPMで保護された鍵、keyctl |
| [乱数生成](rng.md) | /dev/random、getrandom()、DRBG、ハードウェアエントロピー、VMのエントロピー |
| [暗号の事件簿](war-stories.md) | IVの再利用、タイミング攻撃、ブート時のエントロピー、keyringのリーク、ハードウェアのバグ |

## 読む順序 {#reading-order}

| ステップ | ページ | 理由 |
|------|------|-----|
| 1 | [はじめに](README.md) | スタック図とクイックリファレンス |
| 2 | [カーネル暗号API](crypto-api.md) | 中核となる抽象化: SKCIPHER、AEAD、ハッシュ、AF_ALG |
| 3 | [dm-cryptとfscrypt](encryption.md) | ストレージ暗号化で暗号APIがどう使われるか |
| 4 | [crypto_engine](crypto-engine.md) | ハードウェアアクセラレータの組み込み方と非同期ドライバモデル |
| 5 | [カーネルkeyring](keyring.md) | 鍵の置き場所、鍵タイプ、パーミッション、TPMとの統合 |
| 6 | [乱数生成](rng.md) | 暗号用の乱数がどこから来るか |
| 7 | [暗号の事件簿](war-stories.md) | 実際のバグと、上記の内部構造がそれをどう招いたか（あるいは防いだか） |

## クイックリファレンス {#quick-reference}

```bash
# 利用可能な暗号アルゴリズムを一覧表示
cat /proc/crypto

# AES-NIが利用可能か確認
grep aes /proc/cpuinfo

# 暗号の性能をテスト
cryptsetup benchmark
# Testing 128 bit cipher AES-XTS... 1234.5 MiB/s

# 鍵管理
keyctl show           # プロセスのkeyringを表示
keyctl add user mykey "mysecret" @u
keyctl search @u user mykey
```
