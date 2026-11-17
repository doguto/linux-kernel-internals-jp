# カーネルkeyring {#kernel-keyring}

> 鍵の格納、鍵タイプ、keyring、TPMで保護された鍵

## 概要 {#overview}

Linuxカーネルのkeyringサブシステムは、暗号素材（鍵、トークン、証明書）をアクセス制御を強制したうえでカーネルメモリに格納する、汎用の仕組みを提供する。鍵はカーネル空間に置かれる — 鍵タイプが禁じていれば、プロセスは鍵を参照できても、ユーザー空間でその内容を読めることは一切ない。

呼び出し元は `keyctl(2)` システムコール（あるいは `add_key(2)` や `request_key(2)` のようなラッパー）を介してkeyringとやり取りする。fscrypt、NFS、IPsec、IMAなどのサブシステムは、ユーザー空間がインストールした鍵を格納するために内部でkeyringを使う。

## struct key {#struct-key}

すべての鍵 — keyringを含む — は `include/linux/key.h` の `struct key` で表される（ここでは2つの条件付きフィールド、すなわち `CONFIG_KEY_NOTIFICATIONS` 下の `struct watch_list *watchers` と、`KEY_DEBUGGING` 下のデバッグ用 `magic` フィールドを省いて示す）:

```c
struct key {
    refcount_t              usage;
    key_serial_t            serial;         /* 一意な鍵ID（keyctl showが表示するもの） */
    union {
        struct list_head    graveyard_link;
        struct rb_node      serial_node;
    };
    struct rw_semaphore     sem;            /* 鍵の内容を保護する */
    struct key_user         *user;          /* 所有者のアカウンティング */
    void                    *security;      /* LSMのセキュリティblob */
    union {
        time64_t            expiry;         /* 0 = 期限なし */
        time64_t            revoked_at;
    };
    time64_t                last_used_at;

    kuid_t                  uid;
    kgid_t                  gid;
    key_perm_t              perm;           /* パーミッションビット（下記参照） */

    unsigned short          quotalen;       /* クォータに課金されるバイト数 */
    unsigned short          datalen;        /* ペイロード長 */
    short                   state;          /* 鍵の状態(+)または拒否エラー(-);
                                              * KEY_IS_UNINSTANTIATED / KEY_IS_POSITIVE */

    unsigned long           flags;
    /* KEY_FLAG_DEAD, KEY_FLAG_REVOKED, KEY_FLAG_IN_QUOTA, KEY_FLAG_INVALIDATED, ...
     * （インスタンス化はフラグビットではなく、上の`state`で追跡される） */

    /* タイプ + 説明: アクセス方法に応じて2通りに重ねられている */
    union {
        struct keyring_index_key index_key;
        struct {
            unsigned long    hash;
            unsigned long    len_desc;
            struct key_type  *type;         /* "user", "logon", "keyring", ... */
            struct key_tag   *domain_tag;   /* 操作のドメイン */
            char             *description;  /* 人が読める名前 */
        };
    };

    union {
        union key_payload   payload;        /* 実際の鍵素材 */
        struct {
            /* keyringの場合: */
            struct list_head name_link;
            struct assoc_array      keys;
        };
    };

    struct key_restriction  *restrict_link; /* keyringのみ: 何をリンクできるかを制限する */
};
```

`key_serial_t` は32ビット整数（`keyctl show` が表示する数値ID）である。鍵は参照カウントされており、`key_get()` / `key_put()` が寿命を管理する。

## 鍵タイプ {#key-types}

`struct key_type` は、鍵がどのようにインスタンス化、検証、読み出し、破棄されるかを定義する。組み込みの鍵タイプ:

### "user" — ユーザー空間から読めるバイト列 {#user-userspace-readable-bytes}

```c
/* security/keys/user_defined.c */
```

任意のバイトblobである。説明とペイロードの両方が、許可されたプロセスから見える。ユーザー空間が値を取得する必要がある汎用の鍵格納に使われる:

```bash
keyctl add user mykey "hello world" @u
keyctl pipe <serial>   # ペイロードのバイト列を出力
```

### "logon" — カーネル専用、ユーザー空間からは読めない {#logon-kernel-only-unreadable-by-userspace}

ペイロード形式は "user" と同じだが、鍵タイプの `.read` コールバックが `-EPERM` を返す。決してユーザー空間に再び公開してはならない鍵素材を格納する必要がある、カーネル内の利用者（fscrypt、dm-crypt）向けに設計されている:

```bash
keyctl add logon fscrypt:abcdef0123456789 <binary_key_data> @s
# 読み戻せない: keyctl pipeは"permission denied"を返す
```

fscryptは `fscrypt:<key_identifier>` という形式のlogon鍵を使う。dm-crypt（cryptsetup経由）は `cryptsetup:<uuid>` を使う。

### "keyring" — keyringそれ自体が鍵である {#keyring-a-keyring-is-itself-a-key}

keyringは他の鍵への参照を保持する。keyringを検索すると、カーネルはチェーンを再帰的にたどる。ペイロードは `struct key *` の `assoc_array` である。

```bash
keyctl newring mykeyring @u   # ユーザーkeyringにリンクされた名前付きkeyringを作成
keyctl link <key_serial> <ring_serial>   # 鍵をkeyringにリンク
```

### "encrypted" — マスター鍵で暗号化される {#encrypted-encrypted-under-a-master-key}

encrypted鍵は「マスター鍵」（trusted鍵またはuser鍵）で暗号化されて格納される。平文がカーネル空間の外に出ることはない:

```c
/* security/keys/encrypted-keys/encrypted.c */
struct encrypted_key_payload {
    struct rcu_head     rcu;
    char               *format;         /* datablob: フォーマット */
    char               *master_desc;    /* 暗号化に使う鍵の説明 */
    char               *datalen;
    u8                 *iv;
    u8                 *encrypted_data;
    unsigned short      datablob_len;         /* datablobの長さ */
    unsigned short      decrypted_datalen;
    unsigned short      payload_datalen;
    unsigned short      encrypted_key_format; /* encrypted鍵のフォーマット */
    u8                 *decrypted_data;         /* カーネル内でAES-CBC復号されたもの */
    u8                  payload_data[];
};
```

encrypted鍵の作成:

```bash
# まずuserマスター鍵を作成
keyctl add user kmk "$(dd if=/dev/urandom bs=32 count=1 2>/dev/null)" @u

# kmkで暗号化された新しい32バイトのencrypted鍵を作成
keyctl add encrypted myenckey "new user:kmk 32" @u

# 保存するためにエクスポート（暗号化されたまま）
keyctl pipe <serial> > myenckey.blob

# 後で: blobから再ロード
keyctl add encrypted myenckey "load user:kmk $(cat myenckey.blob)" @u
```

2つのサブタイプがある: `"new"` は新しいランダムな鍵を作成し、`"load"` は既存の暗号化されたblobをロードする。

### "trusted" — TPMに封印される {#trusted-sealed-to-a-tpm}

trusted鍵はTPM（Trusted Platform Module）によって生成され、封印（seal）される。平文の鍵素材がユーザー空間に公開されることはない — TPMの内部か、封印解除されている間のカーネルメモリにしか存在しない:

```c
/* include/keys/trusted-type.h（security/keys/trusted-keys/trusted_tpm1.c と trusted_tpm2.c が使う） */
struct trusted_key_payload {
    struct rcu_head     rcu;
    unsigned int        key_len;            /* 鍵サイズ（バイト） */
    unsigned int        blob_len;           /* TPMで封印されたblobのサイズ */
    unsigned char       migratable;         /* 鍵を移行できるか */
    unsigned char       old_format;
    unsigned char       key[MAX_KEY_SIZE + 1];   /* 平文、カーネル内のみ */
    unsigned char       blob[MAX_BLOB_SIZE];     /* TPMで封印された暗号文 */
};
```

TPMは鍵をStorage Root Key（SRK）で封印し、必要に応じてPCR値に結び付ける（そのため、システムが特定の計測済みブート状態にあるときにしか鍵は封印解除されない）:

```bash
# CONFIG_TRUSTED_KEYS=y とTPMが必要

# 新しい32バイトのtrusted鍵を作成し、現在のPCR状態に封印する
keyctl add trusted mytrustedkey "new 32" @u

# 封印されたblob（TPMで暗号化された形式 — 保存しても安全）をエクスポート
keyctl pipe <serial> > mytrustedkey.blob

# 再起動後: blobを再ロード（PCRが一致すればTPMが封印を解除する）
keyctl add trusted mytrustedkey "load $(xxd -p mytrustedkey.blob | tr -d '\n')" @u

# trusted鍵をencrypted鍵のマスター鍵として使う:
keyctl add encrypted myenckey "new trusted:mytrustedkey 64" @u
```

### "asymmetric" — 署名検証用の公開鍵 {#asymmetric-public-key-for-signature-verification}

RSAまたはECの公開鍵（X.509証明書、PKCS#7）を格納する。IMA（Integrity Measurement Architecture）がファイル署名の検証に使い、カーネルモジュール署名サブシステムも使う:

```bash
# IMA: 署名用証明書をIMA keyringに追加
# （通常はpivot_rootの前にinitramfs内で行う）
evmctl import /etc/keys/ima.crt $(keyctl show @u | awk '/\.ima/ {print $1}')
```

カーネルは、システムkeyring（`.builtin_trusted_keys` または `.secondary_trusted_keys`）内の鍵まで証明書チェーンを検証する。

## keyringの階層 {#keyring-hierarchy}

keyringは階層状に配置されている。鍵を検索すると、カーネルはチェーンをたどっていく:

```
@t  thread keyring        ← searched first (per-thread, rarely used)
@p  process keyring       ← per-process (all threads share it)
@s  session keyring       ← per-login-session (inherited across fork/exec)
@u  user keyring          ← per-UID, persists while user is logged in
@us user-session keyring  ← per-UID session (union of @u and @s for most purposes)
@g  group keyring         ← per-GID (unimplemented — see note below)

System keyrings (kernel-internal, not per-process):
.builtin_trusted_keys     ← built-in X.509 certificates (module signing, IMA)
.secondary_trusted_keys   ← runtime-added trusted certs
.ima                      ← IMA policy and measurement list keys
.blacklist                ← revoked certificate hashes
```

`@g` は予約された `KEY_SPEC_GROUP_KEYRING` 定数だが、その背後にグループkeyringの実装は存在しない: `lookup_user_key()`（`security/keys/process_keys.c`）はこれに対して無条件に `-EINVAL` を返し、「group keyrings are not yet supported」というコメントが付いている。この定数が追加されて以来ずっとこの状態であり、変わる兆しもない。

```bash
keyctl show         # 現在のセッションkeyringのツリー
keyctl show @u      # ユーザーkeyring
keyctl show @s      # セッションkeyring

# 出力例:
# Keyring
#  xxxxxxxx --alswrv    500  500  keyring: _ses
#  yyyyyyyy --alswrv    500  500   \_ keyring: _uid.500
#  zzzzzzzz --alswrv    500  500       \_ user: mykey
```

`fork()` では、子は親のセッションkeyringとユーザーkeyringを継承する。`exec()` では、スレッドkeyringがクリアされる。`CLONE_NEWUSER` では、新しいuser namespace用に新しいユーザーkeyringが作成される。

## 鍵のパーミッション: key_perm_t {#key-permissions-key_perm_t}

パーミッションは、4つの主体 × 6つのパーミッションビットをエンコードした32ビット値である:

```
Bits 31..24: possessor permissions  (the process that "possesses" the key)
Bits 23..16: user permissions       (process UID == key UID)
Bits 15.. 8: group permissions      (process GID == key GID)
Bits  7.. 0: other permissions

Per-subject permission bits:
  0x01  view      see the key's description and metadata
  0x02  read      read the key's payload
  0x04  write     update the payload or add links to a keyring
  0x08  search    find the key during a search
  0x10  link      link the key into a keyring
  0x20  setattr   change ownership, permissions, expiry
```

```bash
# パーミッションを8進数風の形式で表示:
# --alswrv は、possessorについて{link,search,write,read,view}のすべてを意味する
keyctl show @u
# xxxxxxxx --alswrv    500  500  keyring: _uid.500
#           ^^^^^^^^
#           permフィールド: possessor=alswrv, user=alswrv, ...

# パーミッションを明示的に設定（16進数）:
keyctl setperm <serial> 0x3f3f0000
#   0x3f = view|read|write|search|link|setattr
#   possessor=0x3f, user=0x3f, group=0, other=0
```

## keyctl(2)システムコール {#keyctl2-syscall}

```c
#include <sys/types.h>
#include <keyutils.h>   /* keyctl(2)をラップするユーザー空間ライブラリ */

/* 鍵を追加する */
key_serial_t add_key(const char *type, const char *description,
                     const void *payload, size_t plen,
                     key_serial_t keyring);

/* 鍵を要求（検索）し、見つからなければ必要に応じてユーザー空間ヘルパーを呼ぶ */
key_serial_t request_key(const char *type, const char *description,
                          const char *callout_info, key_serial_t dest_keyring);

/* keyctl操作 */
long keyctl(int operation, ...);
```

主な `keyctl` 操作:

| 操作 | 説明 |
|---|---|
| `KEYCTL_GET_KEYRING_ID` | 特殊keyringの定数（`KEY_SPEC_*`）をシリアル番号に解決する |
| `KEYCTL_DESCRIBE` | 説明、タイプ、UID、GID、パーミッションを取得する |
| `KEYCTL_READ` | ペイロードを読む（"logon" タイプでは `-EPERM` を返す） |
| `KEYCTL_UPDATE` | 鍵のペイロードを更新する |
| `KEYCTL_REVOKE` | 鍵を失効させる（deadとマークし、検索でスキップされる） |
| `KEYCTL_UNLINK` | keyringから鍵を取り除く |
| `KEYCTL_LINK` | keyringに鍵を追加する |
| `KEYCTL_SEARCH` | タイプ+説明でkeyringのツリーから鍵を検索する |
| `KEYCTL_SET_TIMEOUT` | 鍵の有効期限を設定またはクリアする |
| `KEYCTL_SETPERM` | 鍵のパーミッションを変更する |
| `KEYCTL_INSTANTIATE` | ユーザー空間の鍵ヘルパーから鍵をインスタンス化する |
| `KEYCTL_INVALIDATE` | 鍵を直ちに無効化する |

特殊keyring用の `KEY_SPEC_*` 定数:

```c
KEY_SPEC_THREAD_KEYRING    /* @t */
KEY_SPEC_PROCESS_KEYRING   /* @p */
KEY_SPEC_SESSION_KEYRING   /* @s */
KEY_SPEC_USER_KEYRING      /* @u */
KEY_SPEC_USER_SESSION_KEYRING  /* @us */
KEY_SPEC_GROUP_KEYRING     /* @g */
KEY_SPEC_REQKEY_AUTH_KEY   /* request_keyの認証用 */
```

## シェルでの使い方 {#shell-usage}

```bash
# セッションkeyringに"user"タイプの鍵を追加
keyctl add user mypasswd "s3cr3t" @s

# keyringのツリーを表示
keyctl show @s

# 鍵のペイロードを読む
keyctl pipe <serial>
keyctl print <serial>     # 同じだが、テキスト出力になるようにする

# タイプと説明で鍵を検索
keyctl search @s user mypasswd

# 鍵の情報を表示（タイプ、uid、gid、perm、説明）
keyctl describe <serial>

# タイムアウトを設定（現在からの秒数）
keyctl timeout <serial> 300     # 鍵は5分後に期限切れになる

# 鍵を直ちに失効させる
keyctl revoke <serial>

# keyringからリンクを外す
keyctl unlink <serial> @s

# 新しい空のkeyringを作成
keyctl newring myring @s

# 鍵を追加のkeyringにリンクする
keyctl link <key_serial> <ring_serial>
```

## 鍵のガベージコレクション {#key-garbage-collection}

鍵は次の場合に自動的に回収される:

1. **期限切れ**: `expiry` が設定されていて、その `time64_t` の時刻を過ぎた場合。鍵は「期限切れ」状態になり、以降の検索や読み出しは `-EKEYEXPIRED` を返す。

2. **失効**: `keyctl revoke` またはカーネルによる `key_revoke()` の呼び出し。`-EKEYREVOKED` を返す。

3. **参照カウントがゼロになる**: 鍵を保持していたすべてのプロセスが終了するかリンクを外し、どのkeyringからも参照されなくなった場合。

4. **ユーザークォータ**: 各UIDには鍵の数と合計バイト数のクォータがある。クォータを超える鍵を作成すると `-EDQUOT` が返る。`/proc/sys/kernel/keys/maxkeys` と `/proc/sys/kernel/keys/maxbytes` で調整できる。

```bash
# 鍵のクォータを確認
cat /proc/key-users
# 0:     4 4/1000000 4/1000000 0/1000000   (uid: keys/max keys/max bytes/max)

cat /proc/keys   # 呼び出し元プロセスから見えるすべての鍵
```

## カーネル側での鍵の検索 {#kernel-side-key-lookup}

カーネルのサブシステムは内部で `request_key()` を使う:

```c
/* 既存の鍵を検索する（ユーザー空間へのアップコールなし） */
struct key *key = request_key(&key_type_logon, "fscrypt:identifier", NULL);
if (IS_ERR(key)) {
    return PTR_ERR(key);  /* 見つからなければ-ENOKEY */
}

/* 鍵のRCUロックの下でペイロードを読む */
const struct user_key_payload *ukp;
rcu_read_lock();
ukp = user_key_payload_rcu(key);
if (!ukp) {
    rcu_read_unlock();
    key_put(key);
    return -EKEYREVOKED;
}
memcpy(out_key_material, ukp->data, ukp->datalen);
rcu_read_unlock();

key_put(key);  /* 参照カウントを減らす */
```

## TPMを使うtrusted鍵: さらに詳しく {#trusted-keys-with-tpm-deeper-look}

trusted鍵はTPMの鍵階層を使う。TPM 2.0での流れ:

```
Boot:
  UEFI/firmware → measures boot components → extends TPM PCRs
  kernel loads  → PCR[4] = kernel hash, PCR[7] = SecureBoot state, etc.

keyctl add trusted mytrustedkey "new 32 keyhandle=0x81000001 pcr:sha256:0,1,7" @u
                                          │               │
                                          │               └─ seal to PCR values 0, 1, 7
                                          └─ use this persistent SRK handle

# TPM seals the key:
#   1. Generate random 32 bytes
#   2. TPM2_Create() under the SRK, binding the current PCR[0,1,7] values
#   3. Return TPM2B_PRIVATE + TPM2B_PUBLIC (the sealed blob)

# Later (after reboot):
keyctl add trusted mytrustedkey "load <hex_blob>" @u
#   TPM2_Load() + TPM2_Unseal() — only succeeds if PCRs match
```

PCRが一致しない場合（例: 別のカーネルでブートした）、`TPM2_Unseal` はエラーを返し、鍵はインスタンス化できない。これは、システムが既知の正常な状態にあるときにだけ自動的にアンロックされるフルディスク暗号化（TPMのみモードのWindows BitLockerに類似）の背後にある仕組みである。

## encrypted鍵: "new"と"load" {#encrypted-keys-new-vs-load}

encrypted鍵は、作成時にマスター鍵がkeyring内に存在している必要がある:

```bash
# "new": 新しいランダムな鍵を生成し、マスター鍵で暗号化して、暗号化されたblobをカーネルに格納する
keyctl add encrypted myenckey "new user:mymaster 32" @u

# 暗号化されたblobをエクスポート（これは暗号文なので、ディスクに保存しても安全）
keyctl pipe <serial> | xxd

# "load": keyring内のマスター鍵を使って既存のblobを復号する
keyctl add encrypted myenckey "load user:mymaster <hex_blob>" @u
```

ディスク上のペイロード形式は `<key_type> <master_desc> <datalen> <iv> <ciphertext> <hmac>` で、すべて16進数である。暗号化にはHMAC-SHA256による完全性チェック付きのAES-**256**-CBCが使われる — `get_derived_key()`（`security/keys/encrypted-keys/encrypted.c`）は `sha256()` を使ってマスター鍵から `HASH_SIZE`（32バイト、`SHA256_DIGEST_SIZE`）の鍵を導出し、その32バイトの鍵全体が `cbc(aes)` に渡される — 128ビット鍵が使われたことは一度もない。

## fscryptとIMAとの統合 {#fscrypt-and-ima-integration}

**fscrypt** はlogon鍵を使う。ユーザー空間が `FS_IOC_ADD_ENCRYPTION_KEY` を呼ぶと、カーネルは次のことを行う:

1. 「ファイルシステムレベル」の鍵識別子を導出する（生の鍵素材のHKDF-SHA512 — 正確には、単純なSHA-512ハッシュではなく、HKDF-SHA512展開の最初の16バイト）
2. マスター鍵をファイルシステムのカーネル内鍵構造体に格納する
3. 必要に応じて、レガシーなv1 API経由で "logon" タイプの鍵も受け付ける

**IMA** はファイル署名の検証にasymmetric鍵を使う:

```bash
# IMAの署名検証の流れ:
# 1. ブート時、IMAはIMA keyring（.ima）から証明書をロードする
# 2. ファイルアクセス時、IMAはsecurity.ima xattr（PKCS#7署名）を読む
# 3. integrity_digsig_verify(INTEGRITY_KEYRING_IMA, ...)が.ima内で署名鍵を検索する
# 4. 見つからない場合や署名が無効な場合: ポリシーに従った動作（警告/拒否）

# IMA証明書を手動で追加（例: initramfs内で）:
keyctl padd asymmetric "" %:.ima < /etc/ima/signing_key.der
```

## コンテナの分離 {#container-isolation}

独立した `struct key_namespace` は存在しない。代わりに、ユーザーkeyringとユーザーセッションkeyringは、コミット [`0f44e4d976f9`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0f44e4d976f9)（"keys: Move the user and user-session keyrings to the user_namespace"）によって、既存の `struct user_namespace` 上に直接移された（`user_keyring_register` と `keyring_sem` フィールド、`include/linux/user_namespace.h`）。このコミットは **Linux 5.3** で入った（これらのフィールドはv5.2タグにはなく、v5.3以降に存在する）。`CLONE_NEWUSER` で作成された各user namespaceは独自の `user_keyring_register` を得るため、コンテナの `@u`/`@us` はホストのものから分離される — カーネルのどこにも専用の「鍵namespace」型は存在しないにもかかわらず、である。具体的には:

- `CLONE_NEWUSER` は新しいuser namespaceを作成し、そのnamespace内のUIDには新しいユーザーkeyring（`@u`）とセッションkeyring（`@s`）が与えられる。
- コンテナの `@u` はホストの `@u` とは別物なので、コンテナ内での `keyctl add user key "data" @u` はホストに影響しない。
- カーネル内部のシステムkeyring（`.builtin_trusted_keys`、`.ima`）はグローバルであり、すべてのnamespaceで共有される。

コンテナランタイムは、コンテナごとにセッションkeyringが新たに確立されることに依存している。

## keyringの観測 {#observing-the-keyring}

```bash
# 現在のプロセスから見える鍵
cat /proc/keys
# Serial  Flags  Usage  Expiry  Perm      UID   GID  Type  Description
# 0c7ec5e3 I--Q--   1 perm 1f3f0000     0     0 keyring _uid_ses.0

# システム内のすべての鍵（他者の鍵にはCAP_SYS_ADMINが必要）
cat /proc/key-users

# シェルからkeyringのツリーを表示
keyctl show @s

# IMA keyring上の鍵を一覧表示
keyctl show %:.ima

# 特定の鍵が存在するか確認
keyctl search @u logon fscrypt:0123456789abcdef
```

## 関連ソース {#relevant-source}

- `security/keys/` — keyringの中核実装
- `security/keys/trusted-keys/` — trusted鍵タイプ（TPM 1.2と2.0）
- `security/keys/encrypted-keys/` — encrypted鍵タイプ
- `include/linux/key.h` — `struct key`、`struct key_type`
- `include/uapi/linux/keyctl.h` — `KEYCTL_*` 定数
- `crypto/asymmetric_keys/` — asymmetric鍵タイプ
- `Documentation/security/keys/core.rst` — カーネルのドキュメント

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [include/linux/key.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/key.h) — `struct key`、`struct key_type`、`KEY_POS_*`/`KEY_USR_*` パーミッションビットの定義
- [security/keys/](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/security/keys) — keyringの中核実装: `keyctl()` システムコールのディスパッチ（`keyctl.c`）、`request_key()`（`request_key.c`）、パーミッションチェック（`permission.c`）
- [include/keys/trusted-type.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/keys/trusted-type.h) — `struct trusted_key_payload`、TPMで封印された鍵のペイロード
- [include/keys/encrypted-type.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/keys/encrypted-type.h) — `struct encrypted_key_payload`、AES-CBC + HMAC-SHA256のdatablob形式
- [include/uapi/linux/keyctl.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/keyctl.h) — `KEYCTL_*` 操作コードと `KEY_SPEC_*` 特殊keyring定数
- [Documentation/security/keys/core.rst](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/security/keys/core.rst) — カーネル自身によるKey Retention Serviceのドキュメント
- [include/linux/user_namespace.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/user_namespace.h) — `user_keyring_register`/`keyring_sem`、コンテナの分離を支えるuser namespaceごとのkeyringフィールド
- [security/keys/process_keys.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/security/keys/process_keys.c) — `lookup_user_key()`（未実装の `KEY_SPEC_GROUP_KEYRING` のケースを含む）

### manページ {#man-pages}

- [`add_key(2)`](https://man7.org/linux/man-pages/man2/add_key.2.html) — カーネルの鍵管理機構に鍵を追加する
- [`request_key(2)`](https://man7.org/linux/man-pages/man2/request_key.2.html) — 鍵を検索し、見つからなければユーザー空間へのアップコールを呼び出す
- [`keyctl(2)`](https://man7.org/linux/man-pages/man2/keyctl.2.html) — すべての `KEYCTL_*` 操作の背後にあるシステムコール
- [`keyrings(7)`](https://man7.org/linux/man-pages/man7/keyrings.7.html) — カーネル内の鍵管理・保持機構の概要
- [`keyctl(1)`](https://man7.org/linux/man-pages/man1/keyctl.1.html) — このページ全体で使っているkeyutilsのコマンドラインツール

### 関連ページ {#related-pages}

- [dm-cryptとfscrypt](encryption.md) — logon鍵がfscryptやdm-cryptとどう統合されるか
- [カーネル暗号API](crypto-api.md) — encrypted鍵とtrusted鍵が土台とするAES-CBCとHMAC-SHA256のプリミティブ
- [fscrypt: ファイルごとの暗号化](../security/fscrypt.md) — `FS_IOC_ADD_ENCRYPTION_KEY` と、fscryptがマスター鍵をlogon鍵としてどう格納するかの詳細な解説
- [user namespaceと資格情報のマッピング](../security/user-namespaces.md) — 上で説明したコンテナ分離の動作の背後にあるuid/gidマッピングのモデル

### LWN記事 {#lwn-articles}

- [Trusted and encrypted keys](https://lwn.net/Articles/408439/) — Jake Edge、2010年10月: EVM向けの "trusted" および "encrypted" 鍵タイプの導入。TPMによる封印とAES + HMACのdatablob形式を含む
