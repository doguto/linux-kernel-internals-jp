# カーネルモジュール署名 {#kernel-module-signing}

> ロード前のモジュールの暗号学的検証

## なぜモジュール署名か {#why-module-signing}

モジュール署名がなければ、root権限を持つ者は誰でも `insmod` によって任意のカーネルコードを挿入できる。モジュール署名は、セキュリティが重視される環境で、許可されていないモジュールのロードを防ぐ。

- **Secure Boot**: UEFIファームウェアがブートチェーンを検証する。モジュールも検証されなければならない
- **ロックダウンされたカーネル**: `lockdown=integrity` モードは未署名のモジュールを阻止する
- **コンプライアンス**: PCI-DSS、FIPS 140-3の要件

## 設定 {#configuration}

```bash
# Kconfigオプション:
CONFIG_MODULE_SIG=y           # モジュール署名のサポートを有効にする
CONFIG_MODULE_SIG_FORCE=y     # 署名を必須にする（未署名は拒否する）
CONFIG_MODULE_SIG_ALL=y       # 'make modules_install' の際にすべてのモジュールに自動的に署名する
CONFIG_MODULE_SIG_SHA512=y    # 署名ハッシュにSHA-512を使う。kernel/module/Kconfig
                              # の選択肢はSHA-256/384/512と
                              # SHA3-256/384/512（SHA-1とSHA-224は廃止された）。
                              # ちょうど1つが選択され、デフォルトはSHA-512
CONFIG_MODULE_SIG_KEY="certs/signing_key.pem"  # 署名鍵のパス

# 現在の設定を確認する:
zcat /proc/config.gz | grep CONFIG_MODULE_SIG
```

## 署名鍵の生成 {#generating-signing-keys}

```bash
# 新しい鍵ペアを生成する（カーネルのビルド中）:
# カーネルのビルドシステムが鍵を自動生成するのは、CONFIG_MODULE_SIG_KEYが
# デフォルトの "certs/signing_key.pem" のままで、そのファイルが存在しない場合だけである。
# 他のパスを指定した場合は、そのファイルがすでに存在していなければならない --
# ビルドにはそれを作成するルールがない。

# 手動での生成:
openssl req -new -nodes -utf8 -sha512 -days 36500 \
    -batch -x509 \
    -config x509.genkey \
    -outform PEM \
    -out signing_key.pem \
    -keyout signing_key.pem

# x509.genkeyの内容:
cat > x509.genkey << 'EOF'
[ req ]
default_bits = 4096
distinguished_name = req_distinguished_name
prompt = no
string_mask = utf8only
x509_extensions = myexts

[ req_distinguished_name ]
O = My Organization
CN = Module Signing Key
emailAddress = root@localhost

[ myexts ]
basicConstraints=critical,CA:FALSE
keyUsage=digitalSignature
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid
EOF
```

## モジュールへの署名 {#signing-a-module}

```bash
# カーネルのsign-fileユーティリティを使ってモジュールに署名する:
/usr/src/linux-<version>/scripts/sign-file sha512 \
    signing_key.pem \
    signing_key.pem \
    mymodule.ko
# 1つ目のpem: 秘密鍵、2つ目のpem: X.509証明書

# sign-fileには検証モードが「ない」-- 署名はできるが、検証は決してできない。
# 自身の使用法テキストがインターフェースのすべてである:
#   sign-file [-dp] <hash algo> <key> <x509> <module> [<dest>]
#   sign-file -s <raw sig> <hash algo> <x509> <module> [<dest>]
# そしてgetopt(argc, argv, "sdpk")が受け付けるフラグはちょうど4つである:
#   -p  PKCS#7のblobを<module>.p7sにも書き出す
#   -d  署名のみ: <module>.p7sを書き出して終了し、追記しない（-pを含意する）
#   -s  署名する代わりに、作成済みの生の署名をファイルから取り込む
#   -k  署名者をissuer+serialではなくkeyidで識別する
# ロングオプションはまったくないので、--dry-runも--verifyもない。
# 署名を検証するのは、ロード時のカーネルだけである。

# 署名済みモジュールの署名を調べる:
hexdump -C mymodule.ko | tail -5
# EOFから逆にたどると: 最後の28バイトが
# "~Module signature appended~\n" のマーカーで、その前に12バイトの
# struct module_signature、さらにその前にPKCS#7のblobがある。
```

## モジュール署名の形式 {#module-signature-format}

署名は `.ko` ファイルの末尾に追加される。

```c
/* include/uapi/linux/module_signature.h */
struct module_signature {
    __u8  algo;        /* 公開鍵暗号アルゴリズム [0] */
    __u8  hash;        /* ダイジェストアルゴリズム [0] */
    __u8  id_type;     /* 鍵識別子の種類 [enum module_signature_type] */
    __u8  signer_len;  /* 署名者名の長さ [0] */
    __u8  key_id_len;  /* 鍵識別子の長さ [0] */
    __u8  __pad[3];
    __be32 sig_len;    /* 署名データの長さ */
};
```

これらのコメント中の `[0]` は飾りではない: `id_type` と `sig_len` 以外はすべて、
0でなければならないレガシーなフィールドである。これらはPKCS#7以前の形式に由来し、
当時は署名者名、鍵ID、ダイジェストアルゴリズムが平文で運ばれていた。現在はPKCS#7が
それらすべてを署名blob自体の中に持つので、これらのフィールドはレイアウトの互換性のためだけに
残された無用の長物である。`kernel/module_signature.c` の `mod_check_sig()` がこれを強制する —
`algo`、`hash`、`signer_len`、`key_id_len`、`__pad` のいずれかのバイトが非0であればモジュールは
`-EBADMSG` で失敗し、`id_type` が `MODULE_SIGNATURE_TYPE_PKCS7`（= 2）でなければ
`-ENOPKG` で失敗する。この定数はかつて `PKEY_ID_PKCS7` と呼ばれていた。これはenumが
汎用の暗号コードにあった頃の名残であり、構造体を `include/linux/module_signature.h` から
uapiへ移したのと同じv7.1のシリーズで（値2を保ったまま）
名前が変更された。

したがって、生きているフィールドは2つだけである: 常に2である `id_type` と、ビッグエンディアンの
`sig_len` で、後者はPKCS#7のblobが構造体からどれだけ前から始まるかをカーネルに
伝える。

署名済み `.ko` の末尾のレイアウト:
```
[ELF sections]
[PKCS#7 signed data (DER encoded)]
[struct module_signature]
"~Module signature appended~\n"
```

```bash
# 署名情報を読む:
modinfo mymodule.ko | grep sig
# sig_id: PKCS#7
# signer: My Organization: Module Signing Key
# sig_key: 01:23:45:67:89:ab:cd:ef:...
# sig_hashalgo: sha512
```

## ロードと検証 {#loading-and-verification}

```c
/* kernel/module/signing.c */
int module_sig_check(struct load_info *info, int flags)
{
    int err = -ENODATA;
    const unsigned long markerlen = sizeof(MODULE_SIGNATURE_MARKER) - 1;
    const char *reason;
    const void *mod = info->hdr;
    bool mangled_module = flags & (MODULE_INIT_IGNORE_MODVERSIONS |
                                   MODULE_INIT_IGNORE_VERMAGIC);
    /*
     * 改変されたモジュールは許可しない。バージョン情報が取り除かれた
     * モジュールは、もはや署名されたモジュールではないからである。
     */
    if (!mangled_module &&
        info->len > markerlen &&
        memcmp(mod + info->len - markerlen, MODULE_SIGNATURE_MARKER, markerlen) == 0) {
        /* モジュールを切り詰めて署名を取り除く */
        info->len -= markerlen;
        err = mod_verify_sig(mod, info);
        if (!err) {
            info->sig_ok = true;
            return 0;
        }
    }

    /*
     * 有効な署名のないモジュールを信頼されたカーネルにロードすることは
     * 許可しないが、強制していない場合、一部のエラーは致命的ではない。
     */
    switch (err) {
    case -ENODATA:
        reason = "unsigned module";
        break;
    case -ENOPKG:
        reason = "module with unsupported crypto";
        break;
    case -ENOKEY:
        reason = "module with unavailable key";
        break;

    default:
        /*
         * その他のエラーはすべて致命的である。メモリ不足、
         * 解析できない署名、署名チェックの失敗などを含む --
         * 署名が必須でない場合でもそうである。
         */
        return err;
    }

    if (is_module_sig_enforced()) {
        pr_notice("Loading of %s is rejected\n", reason);
        return -EKEYREJECTED;
    }

    return security_locked_down(LOCKDOWN_MODULE_SIGNATURE);
}
```

この関数には、誤解しやすい点が4つある。

**`mangled_module` のガードが最初に来る。** 呼び出し側が `finit_module(2)` に
`MODULE_INIT_IGNORE_MODVERSIONS` または `MODULE_INIT_IGNORE_VERMAGIC` を渡した場合、
マーカーは探されることすらない。バージョン情報が取り除かれたモジュールは
署名されたモジュールではないので、カーネルはそうでないふりをすることを拒み、
そのまま `-ENODATA`（"unsigned module"）の経路に落ちる。

**`err` の初期値は `-ENOKEY` ではなく `-ENODATA` である。** マーカーがまったくない場合に
switchが見るのはこの初期値であり、「署名がまったくない」場合と
「署名はあるが検証に使う鍵がない」場合とで異なるメッセージが出るのはこのためである。
許容される3つのエラーは、`-ENODATA`（未署名）、`-ENOPKG`
（このカーネルが持たない暗号で署名されている）、`-ENOKEY`（信頼されていない鍵で署名されている）
である。それ以外 — 壊れた署名、`mod_check_sig()` からの `-EBADMSG`、
`-ENOMEM` — は、強制が有効かどうかにかかわらず致命的である。

**強制時は、元のエラーではなく `-EKEYREJECTED` を返す。** そのためユーザー空間からは、
3つの理由のどれに該当したかにかかわらず、単一の「key was rejected by service」のerrnoが見える。
理由の文字列は `dmesg` にだけ現れる。

**強制していない経路は `0` ではなくロックダウンの判定を返す。** これが、このページの
残りの部分で述べるモジュール署名とカーネルロックダウンの実際の継ぎ目である:
ロックダウンがオフなら `security_locked_down()` は0を返してロードが進む。
`lockdown=integrity` 以上では `-EPERM` を返し、`sig_enforce` が設定されていなくても
未署名のモジュールは拒否される。

`module_sig_check()` 自体はカーネルにtaintを付けない点に注意。taintは後で、
`kernel/module/main.c` の `module_augment_kernel_taints()` で付けられる。
この関数は `info->sig_ok == false` を見て次のことを行う。

```c
/* kernel/module/main.c */
pr_notice_once("%s: module verification failed: signature "
               "and/or required key missing - tainting "
               "kernel\n", mod->name);
add_taint_module(mod, TAINT_UNSIGNED_MODULE, LOCKDEP_STILL_OK);
```

これは `TAINT_UNSIGNED_MODULE`、つまり `E` フラグである — `TAINT_OOT_MODULE`（`O`）ではない。
後者は、`intree` のmodinfoタグを持たないモジュールに対する、同じ関数内の別のチェックである。
自分でビルドして署名し忘れたツリー内のモジュールには `E` が付き、`O` は付かない。
適切に署名されたツリー外のモジュールには `O` が付き、`E` は付かない。

## 登録された鍵: システムキーリング {#enrolled-keys-the-system-keyring}

カーネルは複数のキーリングを管理しており、証明書がどのキーリングに入るかは
非常に重要である — `mod_verify_sig()` は `verify_pkcs7_signature()` に
`VERIFY_USE_SECONDARY_KEYRING` を渡すので、モジュール署名の検証は
`.secondary_trusted_keys` と、リンクを通じて `.builtin_trusted_keys` および
`.machine` を参照する。`.platform` は参照**しない**。

```bash
# カーネルの組み込みキーリングを表示する:
keyctl show %:.builtin_trusted_keys
# Keyring
#  1234567 ---lswrv      0     0  keyring: .builtin_trusted_keys
#  7654321 ---lswrv      0     0   \_ asymmetric: My Org: Module Signing Key: abc123

# セカンダリキーリングを表示する: 実行時に追加された鍵に加えて、
# .builtin_trusted_keysへのリンクと（設定されていれば）.machineへのリンクがある:
keyctl show %:.secondary_trusted_keys
```

モジュール署名に関係する4つのキーリングと、それぞれに何が入りうるか（カーネルには
他にも `.blacklist`、`.ima`、`.evm` などがあるが、ここでは関係しない）:

| キーリング | 投入元 | モジュール署名の検証で参照されるか |
| --- | --- | --- |
| `.builtin_trusted_keys` | コンパイル時に組み込み: `CONFIG_MODULE_SIG_KEY`、`CONFIG_SYSTEM_TRUSTED_KEYS` | はい |
| `.secondary_trusted_keys` | 実行時に追加された鍵。`.builtin_trusted_keys` と `.machine` へのリンクも持つ | はい（`CONFIG_SECONDARY_TRUSTED_KEYRING`） |
| `.machine` | MOKの証明書（`CONFIG_INTEGRITY_MACHINE_KEYRING` の場合） | はい — `.secondary_trusted_keys` にリンクされている |
| `.platform` | UEFI Secure Bootの `db`、およびそれ以外の場合のMOK証明書 | **いいえ** — kexecイメージやdm-verityなどに使われ、モジュールには決して使われない |

実行時に鍵を追加できるかどうかは権限の問題ではない — そして現在のメインラインでは、
追加先は `.builtin_trusted_keys` ではない。
`Documentation/admin-guide/module-signing.rst` は今でもそのキーリングに対する `keyctl padd` の
手順を説明しているが、その手順はアップストリームではすでに機能しない。
`system_trusted_keyring_init()`（`certs/system_keyring.c`）では、
`.builtin_trusted_keys` はリンク制限の引数に `NULL` を渡して割り当てられる —
リンク制限はまったくない — そしてそのパーミッションマスクは
`KEY_USR_VIEW | KEY_USR_READ | KEY_USR_SEARCH` で、`KEY_USR_WRITE` を含まない。
また、どのプロセスキーリングにもリンクされないので、ユーザー空間の誰もそれを*所有*せず、
`KEY_POS_*` のビットも関係しない。見ることはできるが、
書き込むことはできない。このキーリングは、ビルド時に組み込まれた証明書から1回だけ
満たされる。

実際に実行時の追加を受け付けるのは `.secondary_trusted_keys` であり、
下の例が対象とするのもこれである。これは `KEY_USR_WRITE` 付きで、
`get_builtin_and_secondary_restriction()` が返す制限とともに割り当てられる。
この関数は `CONFIG_INTEGRITY_MACHINE_KEYRING=y` の場合は
`restrict_link_by_builtin_secondary_and_machine` を、それ以外の場合は
`restrict_link_by_builtin_and_secondary_trusted` を設定する。証明書に対する
規則はどちらでも同じである: 新しい鍵のX.509ラッパーは、builtinまたはsecondaryのキーリング
（machineキーリングが有効ならmachineキーリングも）にすでに存在する鍵によって正しく
署名されていなければならない。machine対応版の違いは、`.machine` キーリング自体のリンクも
許可する点だけである — それ以外のケースはすべて
`restrict_link_by_builtin_and_secondary_trusted()` にそのまま委ねる。rootであることも、
`CAP_SYS_ADMIN` を持つことも、`sig_enforce` をオフにすることも役に立たない。保証のない鍵を
信頼チェーンに注入できるケーパビリティは存在せず、それこそが
この仕組みの要点である。

`restrict_link_by_builtin_trusted()` は同じファイルに実在する関数で、
この話題の解説記事で通常名前が挙がるのはこれである — しかし、これは
`.builtin_trusted_keys` を保護してはいない。利用者はIMAのブラックリストキーリング
（`.ima_blacklist`、`security/integrity/ima/ima_mok.c`）と、`keyctl restrict_keyring` が
非対称鍵のキーリングに適用できる `"builtin_trusted"` メソッド
（`crypto/asymmetric_keys/asymmetric_type.c`）である。

```bash
# 常駐している鍵によって正しく署名されている鍵を追加する:
keyctl padd asymmetric "" %:.secondary_trusted_keys < my_key.x509
# 保証されていない鍵は、誰が実行してもここで拒否される。
```

## 独自の鍵の登録（UEFI + Secure Boot） {#enrolling-a-custom-key-uefi-secure-boot}

```bash
# 1. 証明書をエクスポートする
openssl x509 -in signing_key.pem -outform DER -out signing_key.der

# 2. UEFIのMachine Owner Key（MOK）データベースに登録する:
mokutil --import signing_key.der
# パスワードを求められる。UEFIで確認するために再起動が必要

# 3. MOKリストをカーネル*内部*で信頼してよいことをファームウェアに伝える:
mokutil --trust-mok
# --importと同様に、これも再起動時に確認する。これによりファームウェアが
# MokListTrustedRTを公開するようになる。これがないと、MOKの証明書は
# モジュール署名が参照しない.platformに入ってしまう。

# 4. 登録を確認する:
mokutil --list-enrolled | grep "Subject:"

# 5. この鍵で署名されたモジュールをロードする:
insmod mymodule.ko
```

手順3は多くの解説記事が省略している手順であり、これがないと、モジュールのロードに関しては
作業全体が何の効果もないまま終わる。
`security/integrity/platform_certs/keyring_handler.c` の `get_handler_for_mok()` は、
`CONFIG_INTEGRITY_MACHINE_KEYRING` が有効で、*かつ*ファームウェアが `MokListTrustedRT` を
公開している場合にのみ、MOKの証明書を `add_to_machine_keyring()` に回す。そうでなければ
`add_to_platform_keyring()` にフォールバックし、`.platform` の鍵は
`mod_verify_sig()` からは見えない。したがって「再起動後に鍵が信頼され、その鍵で署名された
モジュールがロードできる」が成り立つのは、`CONFIG_INTEGRITY_MACHINE_KEYRING`
（v5.18以降、コミット `d19967764ba8`）付きでビルドされ、MOKリストが信頼済みとしてマークされた
カーネルだけである。それがないカーネルでは、`mokutil --import` からモジュール署名の
検証に至るアップストリームの経路はまったく存在しない — 代わりに、証明書を
`CONFIG_SYSTEM_TRUSTED_KEYS` に入れてリビルドしなければならない。

## カーネルのロックダウンモード {#kernel-lockdown-mode}

Linuxのロックダウン（5.4以降）は、rootにできることを制限する。

```bash
# ロックダウンモードを確認する:
cat /sys/kernel/security/lockdown
# [none] integrity confidentiality

# モード:
# none:           制限なし
# integrity:      モジュール署名と完全性チェックの回避を防ぐ
# confidentiality: さらにカーネルメモリの読み取りも防ぐ

# ロックダウンを設定する。lockdown_write()はケーパビリティのチェックを一切行わない --
# アクセスはsecurityfsのファイルのモード（0644、root所有）だけで制御される。
# 唯一の制限は方向である: 要求されたレベルが現在のレベルより強くなければ
# lock_kernel_down()は-EPERMを返すので、引き上げることはできるが、
# 引き下げることは決してできない。
echo integrity > /sys/kernel/security/lockdown

# 他の2つの方法も、同じく一方通行である:
#   カーネルコマンドラインのlockdown=integrity（early_param）
#   ビルド時のCONFIG_LOCK_DOWN_KERNEL_FORCE_INTEGRITY
```

!!! warning "アップストリームではSecure Bootはロックダウンを有効にしない"

    UEFI Secure Bootを有効にするとカーネルが自動的に `lockdown=integrity` になる、
    という主張が広く繰り返されている。これは**ディストリビューションのカーネルの
    挙動**であり — Fedora、RHEL、Ubuntuはそうするパッチを持っている — メインラインの
    挙動ではない。メインラインでは、`lock_kernel_down()` は `security/lockdown/lockdown.c` 内の
    `static` 関数で、呼び出し元はちょうど3つ: `lockdown=` ブートパラメータ、
    `CONFIG_LOCK_DOWN_KERNEL_FORCE_*` のビルド時の選択、そして上のsecurityfsへの書き込みである。
    EFIやSecure Bootのコード経路から呼ばれることはない。両者の連動は繰り返し提案され、
    アップストリームで却下されてきた — 下の参考文献の "Kernel lockdown locked out — for now" を参照。
    Secure Bootがオンのバニラカーネルでも、
    `cat /sys/kernel/security/lockdown` は `[none]` を報告する。

    メインラインはSecure Bootを1つのものと連動させては*いる*が、それはロックダウンではない:
    `CONFIG_IMA_ARCH_POLICY=y` の場合、Secure Bootを検出するとモジュール署名の強制が、
    ロックダウンをまったく経由せずに直接有効になる。このページの末尾の
    [強制の経路](#modprobe-force-does-not-defeat-enforcement)を
    参照。

理由コードは `enum lockdown_reason` の中で順序付けられており、
`LOCKDOWN_MODULE_SIGNATURE` は `LOCKDOWN_NONE` の直後の最初のものである — そのため
どのロックダウンレベルでも未署名のモジュールは阻止される。
`LOCKDOWN_INTEGRITY_MAX` までのものはすべてintegrityの理由であり、それより後のものは
`confidentiality` でのみ阻止される。

`lockdown=integrity` の下では（`security/security.c` の `lockdown_reasons[]`）:

- 未署名のモジュールが拒否される — `MODULE_SIG_FORCE` にかかわらず、
  `module_sig_check()` の末尾の `security_locked_down()` 呼び出しによる
- `/dev/mem`、`/dev/kmem`、`/dev/port` へのアクセスが拒否される
- 未署名イメージのkexec、ハイバネーション、直接のPCIアクセス、生のioportとMSRへの
  アクセス、ACPIテーブルとデバイスツリーの変更
- debugfsへのアクセス、安全でないモジュールパラメータ、mmiotrace
- BPFによる*ユーザー*RAMへの書き込み、kgdb/kdbによるカーネルRAMへの書き込み

`confidentiality` はその上に読み取り側の制限を加える: `/proc/kcore`、
kprobe、tracefs、perf、そしてBPFやkgdbによるカーネルRAMの読み取りである。

## モジュール署名の問題のデバッグ {#debugging-module-signature-issues}

```bash
# モジュールのロードはなぜ失敗したのか?
dmesg | grep -E "module|sig|sign|Lockdown" | tail -20
# 強制がオンの場合、module_sig_check()から:
#   Loading of unsigned module is rejected
# 強制はオフだがロードはされた場合、module_augment_kernel_taints()から:
#   mymodule: module verification failed: signature and/or required key
#   missing - tainting kernel
# ロックダウンが拒否した場合、lockdown_is_locked_down()から:
#   Lockdown: insmod: unsigned module loading is restricted; see man
#   kernel_lockdown.7

# モジュールが署名されているか確認する:
modinfo mymodule.ko | grep sig_id
# （未署名なら空）

# カーネルの署名強制の状態は?
# sig_enforceは組み込みの "module" モジュールのモジュールパラメータであり、
# sysctlではない -- /proc/sys/kernel/sig_enforceは存在しない:
cat /sys/module/module/parameters/sig_enforce
# YまたはN。bool_enable_onlyとして宣言されているので、実行時にN -> Yへは
# 切り替えられるが、Y -> Nへは決して切り替えられない:
echo 1 > /sys/module/module/parameters/sig_enforce   # 動作する
echo 0 > /sys/module/module/parameters/sig_enforce   # 一度設定されると-EROFS

# taintフラグを確認する:
cat /proc/sys/kernel/tainted
# ビット13（値8192）はTAINT_UNSIGNED_MODULE、つまり 'E' フラグ。
# ビット12（値4096）はTAINT_OOT_MODULE、つまり 'O' フラグ -- 別物である。
# ビットの意味はDocumentation/admin-guide/tainted-kernels.rstを参照
```

### `modprobe --force` は強制を無効化しない {#modprobe-force-does-not-defeat-enforcement}

`modprobe --force` で `sig_enforce` を回避できる、という根強い俗説がある。
実際には役に立つどころか逆効果である。`--force` は `--force-vermagic` と
`--force-modversion` を合わせたもので、modprobeは
`finit_module(2)` に `MODULE_INIT_IGNORE_VERMAGIC | MODULE_INIT_IGNORE_MODVERSIONS` を
渡すことになる — そしてこれらはまさに、`module_sig_check()` で
`mangled_module` を設定する2つのフラグである。すると署名はまったく検証されず、
モジュールは完全に有効な署名を持っていても未署名として扱われる。

```bash
# CONFIG_MODULE_FORCE_LOAD=yでビルドされ、強制がオフのカーネルの場合:
# これはロードされ、'F'（強制）と 'E'（未署名）の両方のtaintが付く
modprobe --force mymodule

# はるかに一般的なCONFIG_MODULE_FORCE_LOAD=n（デフォルト）の場合:
# IGNORE_VERMAGICフラグはcheck_modinfo()内のtry_to_force_load()に届き、
# CONFIG_MODULE_FORCE_LOADが組み込まれていなければこれは即座に-ENOEXECを返す
# -- 署名のチェックに到達する前にロードが失敗する
modprobe --force mymodule
# modprobe: ERROR: could not insert 'mymodule': Invalid module format

# 強制がオンのカーネルの場合（CONFIG_MODULE_FORCE_LOADの設定にかかわらず）:
# 署名済みモジュールであっても、別の理由で失敗する -- 強制ロードにより
# 改変済みとしてマークされ、未署名となった経路が強制によって拒否される
modprobe --force mymodule
# modprobe: ERROR: could not insert 'mymodule': Key was rejected by service
```

`Key was rejected by service` は `EKEYREJECTED` であり、強制時に
`module_sig_check()` が返すerrnoである。強制を行っているカーネルに未署名のモジュールを
ロードさせるフラグは存在しない。抜け道は、カーネルがすでに信頼している鍵で署名するか、
強制を行っていないカーネルを動かすかのどちらかしかない。

「強制を行っていない」は見た目より狭い条件である。`sig_enforce` は一度設定されると
クリアできず、メインラインにはそれを設定する方法が3つあるからである。

1. **ビルド時の `CONFIG_MODULE_SIG_FORCE=y`。** これは文字どおり初期化子である:
   `kernel/module/signing.c` の `static bool sig_enforce = IS_ENABLED(CONFIG_MODULE_SIG_FORCE)`。
2. **コマンドラインの `module.sig_enforce=1`**、あるいは後から
   `/sys/module/module/parameters/sig_enforce` に書き込まれた `1` — 同じ `bool_enable_only`
   パラメータであり、書き込みが一方通行なのはこのためである。
3. **`CONFIG_IMA_ARCH_POLICY=y` のカーネルでのSecure Boot。** 最初の2つのどちらも
   存在しないので、多くの人が不意を突かれるのはこれである。
   `security/integrity/ima/ima_efi.c` の `arch_get_ima_policy()` は、
   `arch_get_secureboot()` がSecure Bootのオンを報告するとすぐに、`set_module_sig_enforced()` を
   呼ぶ — アーキテクチャのルールセットを返す前に、無条件に。
   `arch/powerpc/kernel/ima_arch.c` には `is_ppc_secureboot_enabled()` に対する同等の処理がある。
   そのためx86、arm64、powerpc（基盤となる `CONFIG_IMA_SECURE_AND_OR_TRUSTED_BOOT` を
   selectする3つのアーキテクチャ）では、IMAのarchポリシー付きでビルドされ
   Secure Bootで起動されたカーネルは、`CONFIG_MODULE_SIG_FORCE` が設定されておらず、
   コマンドラインに `sig_enforce` について何も書かれておらず、さらに — 上の警告のとおり —
   ロックダウンが `[none]` のままであっても、モジュール署名を強制する。
   確認すべきは設定やコマンドラインではなく、`cat /sys/module/module/parameters/sig_enforce`
   である。

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/module/signing.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/signing.c) — `module_sig_check()` と `mod_verify_sig()`: マーカーのチェック、署名ブロックの切り詰め、そしてセカンダリキーリングに対する実際の検証を行う `verify_pkcs7_signature()` の呼び出し
- [include/uapi/linux/module_signature.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/module_signature.h) — `MODULE_SIGNATURE_MARKER`（`"~Module signature appended~\n"`）と `struct module_signature`。署名者/鍵ID/署名/情報ブロックのレイアウトを説明するフィールド順序のコメント付き
- [kernel/module_signature.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module_signature.c) — `mod_check_sig()`: `id_type` が `MODULE_SIGNATURE_TYPE_PKCS7` でないもの、あるいは `algo`/`hash`/`signer_len`/`key_id_len` のフィールドが非0のものを拒否する健全性チェック
- [scripts/sign-file.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/sign-file.c) — 署名ツール: 引数の順序、OpenSSLのCMS/PKCS#7の呼び出し、4つのフラグ（`-s`、`-p`、`-d`、`-k`）
- [certs/system_keyring.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/certs/system_keyring.c) — `.builtin_trusted_keys` と `.secondary_trusted_keys` の割り当て（このファイル内の `keyring_alloc()` 呼び出しはこの2つだけ）、`get_builtin_and_secondary_restriction()` による後者のリンク制限、そして `security/integrity/digsig.c` で割り当てられる `.machine` キーリングをセカンダリキーリングにリンクする `set_machine_trusted_keys()`
- [certs/Kconfig](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/certs/Kconfig) — `CONFIG_MODULE_SIG_KEY`、RSA/ECDSA/ML-DSAの鍵種別の選択、`CONFIG_SYSTEM_TRUSTED_KEYS`、`CONFIG_SECONDARY_TRUSTED_KEYRING`
- [kernel/module/Kconfig](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/Kconfig) — `CONFIG_MODULE_SIG`、`CONFIG_MODULE_SIG_FORCE`、`CONFIG_MODULE_SIG_ALL`、そして `CONFIG_MODULE_SIG_SHA*` のハッシュの選択肢一式
- [security/lockdown/lockdown.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/security/lockdown/lockdown.c) — ロックダウンLSM: `lockdown=` ブートパラメータ、`/sys/kernel/security/lockdown` のsecurityfsファイル、そして `lock_kernel_down()` がより弱いレベルへの移行を拒否する処理
- [commit 000d388ed3bb](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=000d388ed3bbed745f366ce71b2bb7c2ee70f449) — "security: Add a static lockdown policy LSM"（2019年8月、5.4でマージ）。上で使ったintegrity/confidentialityのレベルを導入したコミット
- [commit d19967764ba8](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d19967764ba876f5c82dabaa28f983b21eb642a2) — "integrity: Introduce a Linux keyring called machine"（2022年1月、5.18でマージ）。MOKに登録された鍵が、モジュール署名が参照する信頼チェーンに届くようにするアップストリームの仕組み

### manページ {#man-pages}

- [`finit_module(2)`](https://man7.org/linux/man-pages/man2/finit_module.2.html) — ロード経路そのもの。`EBADMSG`（不正な形式の署名）と `ENOKEY`（"returned only if the kernel was configured with `CONFIG_MODULE_SIG_FORCE`"）、そして `MODULE_INIT_IGNORE_MODVERSIONS`/`MODULE_INIT_IGNORE_VERMAGIC` フラグを説明している。`ENOKEY` の記述は古く、上で説明したカーネルの挙動と矛盾している点に注意: 5.4以降、強制を行っているカーネルが返すのは `ENOKEY` ではなく `EKEYREJECTED` であり、`ENOKEY` はむしろ `module_sig_check()` 内で*許容される*3つのエラーの1つである
- [`kernel_lockdown(7)`](https://man7.org/linux/man-pages/man7/kernel_lockdown.7.html) — ロックダウンが実際に阻止するもの: 未署名のモジュール、`/dev/mem`、`/dev/kmem`、`/dev/kcore`、`/dev/ioports`、未署名のkexecイメージ、BPF、kprobe
- [`keyctl(1)`](https://man7.org/linux/man-pages/man1/keyctl.1.html) — `keyctl show` と `keyctl padd` のサブコマンドと、上で `.builtin_trusted_keys` を指定するのに使った `%:<name>` 構文
- [`modprobe(8)`](https://man7.org/linux/man-pages/man8/modprobe.8.html) — `--force` の実体（`--force-vermagic` と `--force-modversion`）。改変されたモジュールはもはや署名されたモジュールではないので、これが重要になる
- [`modinfo(8)`](https://man7.org/linux/man-pages/man8/modinfo.8.html) — `.ko` の属性フィールドの出力と、1つだけを選択する `-F`

### 関連ページ {#related-pages}

- [カーネルモジュールの作成とロード](module-basics.md) — この検証が前段に置かれるモジュールのライフサイクル
- [モジュールロードの内部](module-loading-internals.md) — `load_module()`、ELFの解析、そして再配置やシンボル解決に対して署名チェックがどの時点で行われるか
- [カーネルキーリング](../crypto/keyring.md) — 鍵の種別、キーリング、そして信頼された証明書を保持する鍵保持サービス
- [カーネル暗号API](../crypto/crypto-api.md) — 署名検証の下にあるハッシュの仕組み（`crypto_shash`）。非対称暗号の側については、上のキーリングのページの "asymmetric" 鍵種別を参照
- [LSMフレームワーク](../security/lsm.md) — フックテーブル、`security_add_hooks()`、モジュールが自身を登録する方法。ロックダウンLSMが組み込まれる仕組みである
- [Linuxケーパビリティ](../security/capabilities.md) — `CAP_SYS_MODULE`。署名チェックに到達する以前に必要な特権

### LWNの記事 {#lwn-articles}

- [Loading signed kernel modules](https://lwn.net/Articles/470906/) — Jake Edge、2011年12月。David Howellsによる初期の署名付きモジュールのパッチについて。署名をELFコンテナの外に追加するのではなく、`.module_sig` というELFセクションに格納していた
- [The module signing endgame](https://lwn.net/Articles/525592/) — Jake Edge、2012年11月。ビルド時署名かインストール時署名かの議論について。これを受けてTorvaldsが署名を `make modules_install` に移した
- [Kernel lockdown locked out — for now](https://lwn.net/Articles/751061/) — Jonathan Corbet、2018年4月。UEFI Secure Bootが検出されたときにロックダウンを自動的に有効にすることへのTorvaldsの反対について。あるカーネルが両者を連動させていると思い込む前に読む価値がある
- [Lockdown as a security module](https://lwn.net/Articles/791863/) — Jonathan Corbet、2019年6月。実際にマージされたLSMとしての作り直しについて。integrityの理由がconfidentialityの理由より下位に並ぶ理由や、`LOCKDOWN_MODULE_SIGNATURE` の位置付けを含む

### 外部資料 {#external}

- [Documentation/admin-guide/module-signing.rst](https://docs.kernel.org/admin-guide/module-signing.html) — 権威あるガイド: すべての `CONFIG_MODULE_SIG_*` オプション、`certs/x509.genkey` による鍵の生成、`sign-file` の4つの引数、そして署名済みモジュールを後からstripしてはならないという警告
- [Documentation/security/keys/core.rst](https://docs.kernel.org/security/keys/core.html) — 信頼されたキーリングの背後にある鍵保持サービス。`/proc/keys` と `keyctl` インターフェースを含む
- [Documentation/admin-guide/tainted-kernels.rst](https://docs.kernel.org/admin-guide/tainted-kernels.html) — `/proc/sys/kernel/tainted` の解読。署名に対応したカーネルで未署名のモジュールがロードされたときに設定される `E` フラグを含む
