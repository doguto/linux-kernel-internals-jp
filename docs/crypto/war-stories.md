# 暗号の事件簿 {#crypto-war-stories}

> 実際のインシデント: タイミング攻撃、IVの再利用、鍵のリーク、壊れたRNG

ここで扱うのは、カーネルの暗号処理の挙動に根ざした5つの実際の問題である。いずれも同じ構成をとる: 問題、診断の方法、カーネル内部における根本原因、そして修正である。

---

## 1. plain IVモードのdm-cryptにおけるIVの再利用 {#1-iv-reuse-in-dm-crypt-with-plain-iv-mode}

### 問題 {#problem}

あるシステム管理者が、レガシーな `aes-cbc-plain` 暗号指定を使ってdm-cryptボリュームを構成した。それをサーバー群に展開した後、セキュリティレビューによって、生のブロックデバイスへの読み取りアクセスを持つ攻撃者が、2つのセクタが同一の平文を含むときにそれを判別できることがわかった。

### 診断 {#diagnosis}

暗号指定は次のとおりだった:

```bash
cryptsetup create myvolume /dev/sda2 --cipher aes-cbc-plain --key-size 256
```

"plain" IVモードは、各セクタのIVが単に32ビットのセクタ番号であることを意味する:

```c
/* drivers/md/dm-crypt.c */
static int crypt_iv_plain_gen(struct crypt_config *cc, u8 *iv,
                               struct dm_crypt_request *dmreq)
{
    memset(iv, 0, cc->iv_size);
    *(__le32 *)iv = cpu_to_le32(dmreq->iv_sector & 0xffffffff);
    return 0;
}
```

AES-CBCと、32ビットのセクタ番号だけから導出される静的なIVの組み合わせでは、同じセクタ番号を持つ2つのセクタの平文の先頭ブロックがたまたま同じであれば、暗号文の先頭ブロックも同一になる。これがCBCのIV再利用の主要な失敗モードである: 2つのメッセージが同じIVと鍵を共有していると、攻撃者は先頭の暗号文ブロックを比較することで、**平文の先頭ブロックが同一である** ことを検出できる。

```
AES_CBC(key, IV=42, plaintext_v1)[block_0] == AES_CBC(key, IV=42, plaintext_v2)[block_0]
    iff plaintext_v1[block_0] == plaintext_v2[block_0]
```

これは **ウォーターマーキング/トラフィック解析** 攻撃である: 生のデバイスへの読み取りアクセスを持つ攻撃者は、同じセクタの2つのバージョンが同じ16バイトのブロックで始まることを検出できる。これは「暗号文のXORが平文のXORに等しい」という形の漏洩とは別物であり、そちらはCBCではなくストリーム暗号モード（CTR、OFB）に当てはまる。CBCでは、IVの再利用はブロックの同一性を明らかにするのであって、平文の内容を直接明らかにするわけではない。

```bash
# 暗号モードを確認
cryptsetup status myvolume
# cipher: aes-cbc-plain    ← 脆弱
```

### 根本原因 {#root-cause}

`aes-cbc-plain` は、古いcryptsetupのバージョンにおける歴史的なデフォルトだった。これは `aes-cbc-essiv:sha256`（IVを `AES_encrypt(SHA256(volume_key), sector_num)` として導出し、ウォーターマーキングを防ぐ）に取って代わられ、さらに `aes-xts-plain64`（CBCのIV再利用の脆弱性を持たない、数学的に健全なtweakableブロック暗号モードを使う）に取って代わられた。

"plain" という接尾辞は、IVがセクタ番号そのものであり、導出を行わないことを意味する。"plain64" という接尾辞は64ビットのセクタ番号を使い、元のplainの32ビットオーバーフローの問題を修正している。

### 修正 {#fix}

設計上IV再利用の問題を排除しているAES-XTSに移行する:

```bash
# 現在の構成を確認
cryptsetup benchmark
# Testing 128 bit cipher AES-XTS...   1623.5 MiB/s (encrypt)  1607.2 MiB/s (decrypt)
# Testing 128 bit cipher AES-CBC...   1784.2 MiB/s (encrypt)   532.1 MiB/s (decrypt)
# 注: AES-XTSの復号はAES-CBCの復号より約3倍速い

# 新規ボリューム: デフォルトの暗号（aes-xts-plain64）でLUKS2を使う
cryptsetup luksFormat --type luks2 /dev/sda2
# cryptsetup 2.x以降のデフォルト暗号: aes-xts-plain64

# 既存ボリューム: 再暗号化が必要（LUKS2ではインプレースの再暗号化が可能）
cryptsetup reencrypt --cipher aes-xts-plain64 --key-size 512 /dev/sda2
```

AES-XTSは2つの鍵（そのためAES-256-XTSでは512ビット: 256ビットのデータ鍵 + 256ビットのtweak鍵）と、GF(2^128)上でセクタ番号から導出したtweakを使い、平文が同一のセクタであっても各セクタの暗号化を数学的に独立させる。

---

## 2. MAC検証におけるタイミングサイドチャネル: crypto_memneq {#2-timing-side-channel-in-mac-verification-crypto_memneq}

### 問題 {#problem_1}

独自の認証プロトコルを実装したあるカーネルモジュールが、認証タグを `memcmp()` で比較していた。巧妙に細工されたネットワーク入力のもとで、ローカルネットワーク上の攻撃者は応答レイテンシをマイクロ秒精度で計測することにより、有効な認証タグと無効なものを区別できた。

### 診断 {#diagnosis_1}

脆弱なコード:

```c
/* 誤り: 実行時間が変動する比較 */
if (memcmp(received_tag, expected_tag, TAG_LEN) != 0) {
    return -EBADMSG;
}
```

`memcmp()` は異なるバイトを見つけた時点で戻る。攻撃者が（最初のバイトではなく）最後のバイトで異なるMACタグを送ると、`memcmp()` の実行時間が長くなる。十分なサンプル（数百万のリクエスト）があれば、バイトごとの比較は統計的に区別可能になり、タイミングオラクル攻撃によって期待されるタグを1バイトずつ復元できる。

これは2013年のLucky13 TLS攻撃や、さまざまなHMAC検証のバグと同じ種類の脆弱性である。

### 根本原因 {#root-cause_1}

`memcmp()` は最初の相違点の後で戻るように定義されている — これは一般的な用途では正しいが、秘密の比較には誤りである。また、コンパイラは比較コードを、実行時間のばらつきを生むような形で自由に最適化できる。

Linuxは、まさにこれに対処するために、カーネル3.13で `crypto_memneq()` を追加した（コミット `6bf37e5aa90f`、"crypto: crypto_memneq - add equality testing of memory regions w/o timing leaks"、James Yonan）:

```c
/* include/crypto/utils.h */
static inline int crypto_memneq(const void *a, const void *b, size_t size)
{
    return __crypto_memneq(a, b, size) != 0UL ? 1 : 0;
}

/* lib/crypto/memneq.c — 簡略化した汎用パス。実際の実装には
 * ループのない16バイト用の高速パスと、非アラインアクセス版もある。
 * このヘルパーはstatic inlineであり、noinlineが付いているのはエクスポートされた
 * ディスパッチャ__crypto_memneq()のほうである。これはサイズで分岐してこれか高速パスを呼ぶ。 */
static inline unsigned long __crypto_memneq_generic(const void *a, const void *b, size_t size)
{
    unsigned long neq = 0;

    /* 全バイトのXORを累積する。a == bのときに限り結果は0。
     * 常に全バイトを読み、途中で抜けることはない。 */
    while (size >= sizeof(unsigned long)) {
        neq |= *(unsigned long *)a ^ *(unsigned long *)b;
        OPTIMIZER_HIDE_VAR(neq);
        a   += sizeof(unsigned long);
        b   += sizeof(unsigned long);
        size -= sizeof(unsigned long);
    }
    while (size > 0) {
        neq |= *(unsigned char *)a ^ *(unsigned char *)b;
        OPTIMIZER_HIDE_VAR(neq);
        a++;
        b++;
        size--;
    }
    return neq;
}
```

各累積ステップの後の `OPTIMIZER_HIDE_VAR(neq)`（`-Os` コンパイラフラグではない — このファイルにそのような上書きは存在しない）は、累積途中の値をオプティマイザから隠し、ループが途中で抜けられるとオプティマイザが証明するのを防ぐ。エクスポートされたディスパッチャ `__crypto_memneq()` が `noinline` であること（呼び出し元の最適化コンテキストが比較に影響するのを防ぐ）と合わせて、この関数は定数時間の性質を維持する。

### 修正 {#fix_1}

認証パスにあるすべての `memcmp()` 呼び出しを `crypto_memneq()` に置き換える:

```c
/* 正しい: 定数時間の比較 */
#include <crypto/utils.h>

if (crypto_memneq(received_tag, expected_tag, TAG_LEN)) {
    return -EBADMSG;
}
```

AEAD暗号API（`crypto_aead_decrypt()`）は、認証タグの検証に内部ですでに定数時間の比較を使っている。独自のコードでは `crypto_memneq()` が適切な道具である。カーネルは `crypto_authenc_extractkeys()` や、これを正しく扱うより高水準の構成要素も提供している。

---

## 3. ブート時のgetrandom()のブロック: サービスがエントロピーを待ってハングする {#3-getrandom-blocking-at-boot-services-hang-waiting-for-entropy}

### 問題 {#problem_2}

独自のLinuxベースのアプライアンスを実行するKVM仮想マシン群が、ブート中に再現性のあるハングを示した: `sshd` の起動に60〜90秒かかり、systemdのジャーナルにはサービスのタイムアウトが記録され、ブートは最終的には継続するものの、ネットワークスタックの起動が大幅に遅れた。

### 診断 {#diagnosis_2}

症状は `getrandom()` のブロックにたどり着いた:

```bash
# ブートの遅いVMで、ブロックされているプロセスを確認
cat /proc/*/wchan | sort | uniq -c | sort -rn | head
# 12 random_read_iter     ← 12個のプロセスがrandom_read_iterでブロックされている
```

ブロックされたコールスタック（カーネルoopsまたはsysrq-Tから）:

```
[<0>] __schedule+0x3c4/0xa80
[<0>] schedule+0x4a/0xb0
[<0>] wait_for_random_bytes+0x...     ← waiting for crng_ready()
[<0>] sys_getrandom+0x...
```

`getrandom()` は、256ビットのエントロピーがCRNGに集まるまでブロックする。このVMでは:

```bash
# virtio-rngが存在するか確認
lsmod | grep virtio_rng
# （空 — ロードされていない）

# virtio-rngなし、RDRANDのパススルーなし、保存済みシードなし:
dmesg | grep -E "random:|crng"
# [  127.441821] random: crng init done  ← 127秒!
```

（中間の「fast init」のログ行は存在しない — `drivers/char/random.c` は早期エントロピー状態を、自前の `pr_notice()`/`pr_warn()` を一切出さずに黙って通過する。`crng init done` は、それが起動時に出す唯一のメッセージである。）

このVMは次のような状態だった:

- virtio-rngデバイスを構成せずにKVM上で実行されていた
- CPU機能のマスキングにより、RDRANDがゲストから見えなくなっていた
- `systemd-random-seed.service` がブートシーケンスの遅すぎる段階でシードファイルをロードしていた（`network.target` の後であり、その `network.target` 自体が `sshd` を必要とし、`sshd` がgetrandomを必要としていた…）

### 根本原因 {#root-cause_2}

ブートにおける循環依存:
```
network.target
    needs sshd
        needs getrandom() (blocks for entropy)
            needs: interrupt jitter, virtio-rng, or saved seed
                saved seed loaded by systemd-random-seed.service
                    After basic.target (late)
```

仮想マシンでは、割り込みジッタが小さい（ハイパーバイザは予測可能なタイマー割り込みを配送する）。virtio-rngがなければ、カーネルがエントロピーを集める速度は非常に遅い。

### 修正 {#fix_2}

多層防御のために、3つの独立した修正を併用した:

**1. VMの定義にvirtio-rngを追加する**（最善の修正）:

```xml
<!-- QEMU/libvirt: virtio-rngデバイスを追加する -->
<rng model='virtio'>
  <backend model='random'>/dev/urandom</backend>
</rng>
```

ゲストカーネルのドライバ（`drivers/char/hw_random/virtio-rng.c`）が実装しているのは `hwrng.read` コールバック（`virtio_read()`）だけである。ドライバの `read()` を呼び出し、返ってきたバイトを `add_hwgenerator_randomness()` でゲストのエントロピープールに供給するのは、汎用hwrngコアのポーリング用kthread（`drivers/char/hw_random/core.c`）である — この場合、QEMU側のバックエンドには渡せるホストのエントロピーが十分にあるので、供給は即座に行われる。

**2. systemd-random-seedが早期にロードされるようにする**:

```bash
# サービスが有効で、遅いターゲットに入っていないことを確認
systemctl cat systemd-random-seed.service | grep -A 5 '\[Unit\]'
# Before= にはsysinit.targetまたは同様の早期ターゲットが含まれているべき
```

**3. TPMまたはIntel CPUを持つベアメタルの場合**:

```bash
# ハードウェアから収集するためにrng-toolsを有効化
apt install rng-tools
systemctl enable --now rngd
# rngdは/dev/hwrng（RDRAND/TPMを使う）を読み、/dev/randomに供給する
```

virtio-rngを追加した後:

```bash
dmesg | grep "crng init done"
# [    1.234567] random: crng init done  ← 1.2秒、正常
```

---

## 4. /proc経由のカーネルkeyringのリーク {#4-kernel-keyring-leak-via-proc}

### 問題 {#problem_3}

セキュリティ監査により、非特権プロセスが `/proc/keys` を読むことで、他のユーザーのkeyringにある鍵の説明を列挙できることがわかった。懸念は、これによって鍵の存在や目的に関する機密情報が漏れるのではないか、ということだった。

### 診断 {#diagnosis_3}

```bash
# どのプロセスも/proc/keysを読める
cat /proc/keys
# 0c7ec5e3 I--Q--   1 perm 1f3f0000     0     0 keyring _uid_ses.0
# 2effa75e I--Q--   1 perm 1f3f0000  1000  1000 user    myapp:db_password
#                                    ^^^^
#                                    この鍵の説明から、"myapp"が
#                                    "db_password"という鍵を持つことが漏れている
```

鍵の _説明_（名前）は、ペイロードを読めなくても、`/proc/keys` を読めるすべてのプロセスから見える。これは設計どおりである — `/proc/keys` はシステム内のすべての鍵を表示するが、呼び出し元プロセスの `view` パーミッションでフィルタされる。

### 根本原因 {#root-cause_3}

`/proc/keys` は `security/keys/proc.c` で実装されている。カーネルは各鍵について、`KEY_NEED_VIEW` パーミッションビットを指定して `key_task_permission()` を呼ぶ:

```c
/* security/keys/proc.c（簡略化 — vは実際にはrb_nodeであり、struct keyを得るには
 * rb_entry()が必要。実際のコードは、下で渡すkey_refを決める前に、
 * keyring_search_contextを介して呼び出し元の資格情報が実際にその鍵を
 * 所有（possess）しているかどうかも確認する） */
static int proc_keys_show(struct seq_file *m, void *v)
{
    struct key *key = rb_entry((struct rb_node *)v, struct key, serial_node);
    ...

    /* このプロセスが閲覧できない鍵はスキップする。注: ここで使われる資格情報は
     * current_cred()ではなく、/proc/keysをopenした時点で取得された
     * m->file->f_credなので、readで見えるものはopen時に決まる。 */
    rc = key_task_permission(make_key_ref(key, 0), m->file->f_cred,
                              KEY_NEED_VIEW);
    if (rc < 0)
        return 0;   /* 黙ってスキップ */

    /* 説明、タイプ、パーミッション、uid、gid、有効期限を表示 */
    /* ペイロードは表示しない */
    seq_printf(m, "%08x %s%s%s%s%s%s %5d %3d %s %s\n",
               key->serial, ...
               key->type->name,
               key->description);
    return 0;
}
```

ユーザーが作成した鍵のデフォルトのパーミッションは、全員に `view` パーミッションを与える（`other` ビットに `0x01 = view` が含まれる）。`perm = 0x1f3f0000` の鍵は次のパーミッションを持つ:

- possessor: `0x1f` = view|read|write|search|link（setattrビット0x20は設定されて **いない**）
- user: `0x3f` = すべてのパーミッション（view|read|write|search|link|setattr）
- group: `0x00` = パーミッションなし
- other: `0x00` = パーミッションなし

**しかし**: このケースでは、鍵は緩い `other` パーミッションで作成されていた:

```bash
keyctl setperm <serial> 0x3f3f3f01  # otherが"view"できる — バグ!
```

あるいは、アプリケーションが機密情報を露出する鍵の _説明_ を使っていた（例: 説明にホスト名やデータベース名が含まれていた）。

### 根本原因（続き）: logon鍵はこれを防ぐ {#root-cause-continued-logon-keys-prevent-this}

`"logon"` タイプの鍵は、所有プロセスでさえ読むことができない — `.read` コールバックが `-EPERM` を返す。fscryptとcryptsetupが実際の鍵素材にlogon鍵を使うのはこのためである: 説明に対する `view` パーミッションがあっても、ペイロードには到達できない:

```bash
# logon鍵: /proc/keysには見えるが、ペイロードは読めない
keyctl add logon fscrypt:abc123 "$(dd if=/dev/urandom bs=32 count=1 2>/dev/null)" @s

# 説明は見える:
# 1234abcd I--Q--   1 perm 1f1f0000  1000  1000 logon   fscrypt:abc123

# ペイロードにはアクセスできない:
keyctl print 1234abcd
# keyctl_read_alloc: Permission denied
```

### 修正 {#fix_3}

1. **機密素材にはlogon鍵を使う** — keyringに秘密を格納するアプリケーションは、ペイロードがユーザー空間から決して読めないように `"logon"` タイプを使うべきである。

2. **不透明な説明を使う** — `"user"` タイプの鍵を使う場合、説明に機密情報をエンコードすべきではない（例: `myapp:db_password` ではなくUUIDを使う）。

3. **制限的なパーミッションを設定する** — `other=0x00` で鍵を作成する:

```bash
keyctl add user myapp:token "..." @s
keyctl setperm <serial> 0x3f3f0000  # possessorとuserのみ。groupとotherには何も与えない
```

4. **`/proc/key-users` を使う**（`/proc/keys` ではなく） — 説明を露出せずにクォータを監視できる。これはUIDごとのカウントしか表示しない。

---

## 5. 誤った結果を返すハードウェアアクセラレータ: 暗号文の静かな破損 {#5-hardware-accelerator-returning-wrong-results-silent-ciphertext-corruption}

### 問題 {#problem_4}

TLSプロキシでのAES-GCM暗号化にIntel QuickAssist Technology（QAT）アクセラレータを使っていたサーバーが、リクエストの約0.01%で認証失敗を起こすようになった。アプリケーションログには、カーネルのAEAD復号パスからの断続的な `-EBADMSG` が記録されていた。失敗のタイミングはランダムで、特定の入力では再現できなかった。

### 診断 {#diagnosis_4}

プロキシは、高優先度のAES-GCM実装を登録する `qat` カーネルドライバを使っていた:

```bash
cat /proc/crypto | grep -A 15 "name.*gcm(aes)"
# name         : gcm(aes)
# driver       : qat_aes_gcm
# module       : intel_qat
# priority     : 4001        ← AES-NI（800）より高い
# type         : aead
# async        : yes
```

ハードウェアは優先度によって選択されていた。QATモジュールを無効にすると、AES-NIへのフォールバックが強制された:

```bash
# テスト: ドライバをアンロードしてQATを迂回する
rmmod intel_qat
# 失敗は止まった。QATハードウェアが破損した出力を生成していた。
```

根本原因は、特定のQATリビジョンにおけるファームウェアのバグで、4GBの物理アドレス境界をまたぐscatter-gatherリストを正しく扱えないというものだった。

### 根本原因: CRYPTO_ALG_TESTEDとセルフテストフレームワーク {#root-cause-cryptoalgtested-and-the-self-test-framework}

カーネルの暗号サブシステムには必須のセルフテストフレームワーク（`crypto/testmgr.c`）があり、登録されたすべてのアルゴリズムに対して、使用可能とマークする前に既知解テスト（known-answer test）を実行する。セルフテストに失敗したアルゴリズムには `CRYPTO_ALG_TESTED` フラグが付与されず、使用できない:

```c
/* include/linux/crypto.h */
#define CRYPTO_ALG_TESTED       0x00000400

/* このビットが設定されている場合（またはテストが無効な場合）にのみアルゴリズムは使用可能 */
```

AES-GCMのセルフテストは、NIST CAVPテストスイートのハードコードされたテストベクタを使う:

```c
/* crypto/testmgr.h — 抜粋（例示。実際のファイルには数百のベクタがある） */
static const struct aead_testvec aes_gcm_tv_template[] = {
    {
        .key   = "\x00\x00\x00\x00\x00\x00\x00\x00"
                 "\x00\x00\x00\x00\x00\x00\x00\x00",
        .klen  = 16,
        .iv    = "\x00\x00\x00\x00\x00\x00\x00\x00"
                 "\x00\x00\x00\x00",
        .ptext = "",
        .plen  = 0,
        .aad   = "",
        .alen  = 0,
        .ctext = "\x58\xe2\xfc\xce\xfa\x7e\x30\x61"
                 "\x36\x7f\x1d\x57\xa4\xe7\x45\x5a", /* 認証タグのみ */
        .clen  = 16,
    },
    /* ... さらに多くのベクタ ... */
};
```

QATドライバの実装はセルフテスト（小さくアラインされたテストベクタを使う）には合格したが、アドレス境界をまたぐ大きなscatter-gatherリストを伴う実際のワークロードでは失敗した。セルフテストフレームワークはアルゴリズムレベルのバグは捕捉したが、これは特定のDMAレイアウトでのみ顕在化するハードウェアのバグだった。

### 修正 {#fix_4}

**即時の対応**: 優先度を下げるかドライバをアンロードして、AES-GCMでのQATの使用を無効にする。

**ドライバ側**: scatter-gatherのセットアップにDMA境界のチェックを追加する:

```c
/* どのDMAセグメントも4GB境界をまたがないことを確認する */
static int qat_check_sg_alignment(struct scatterlist *sg, int nents)
{
    struct scatterlist *s;
    int i;

    for_each_sg(sg, s, nents, i) {
        dma_addr_t start = sg_dma_address(s);
        dma_addr_t end   = start + sg_dma_len(s) - 1;
        if ((start >> 32) != (end >> 32))
            return -EINVAL;   /* 4GB境界をまたぐので、フォールバックを使う */
    }
    return 0;
}
```

境界チェックに失敗すると、ドライバは [crypto_engine](crypto-engine.md) で説明したのと同じフォールバックパターンを使って、ソフトウェアのAES-GCM実装にフォールバックする。

**長期的な対応**: カーネルの `tcrypt` セルフテストモードを使えば、より大きく多様な入力でストレステストを行える:

```bash
# AEADテストスイートを実行（CONFIG_CRYPTO_TESTが必要）
# 注: tcryptのモード番号はカーネルバージョン間で安定しておらず、このモードも
# 以前に変わったことがある -- 現在のメインラインのcrypto/tcrypt.cでは、素のgcm(aes)テストは
# mode 35である（151/152は関連するrfc4106/rfc4543のAEADラッパーであり、素の
# gcm(aes)ではない）。使う前に自分のカーネルのcrypto/tcrypt.cで確認すること。
modprobe tcrypt mode=35   # gcm(aes)のaeadテスト — 自分のカーネルのcrypto/tcrypt.cで確認すること
# testing gcm(aes)...
# test 0 (512 byte blocks): passed.
# ...
```

`CRYPTO_ALG_TESTED` の仕組みは、セルフテストに失敗するドライバが呼び出し元に公開されないことを保証する。しかし、セルフテストには合格するが実際のDMA条件下で失敗する潜在的なハードウェアのバグからは守れない。ドライバはハードウェア固有のエッジケースについて独自の正当性チェックを実装すべきであり、本番環境では、新しいアクセラレータに頼る前に、実際のワークロードに即したテストパターンでハードウェアを検証すべきである。

---

## まとめ: 現場からの教訓 {#summary-lessons-from-the-field}

| インシデント | 中心的な誤り | 正しいアプローチ |
|---|---|---|
| IVの再利用（dm-crypt） | レガシーな `aes-cbc-plain` 暗号文字列 | `aes-xts-plain64`（LUKS2のデフォルト）を使う |
| タイミング攻撃（memcmp） | 秘密の比較に `memcmp()` を使う | `<crypto/utils.h>` の `crypto_memneq()` |
| ブート時のエントロピー枯渇 | VMにハードウェアRNGがない | virtio-rngデバイス + 保存済みシード |
| keyringの説明のリーク | 鍵の説明に機密データ | 不透明な説明 + logon鍵タイプ |
| ハードウェアによる静かな破損 | セルフテストを十分な検証とみなした | DMA境界チェック + フォールバック |

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [drivers/md/dm-crypt.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/md/dm-crypt.c) — `crypt_iv_plain_gen()` と `crypt_iv_essiv_gen()`。事例1のplain IVによるウォーターマーキング問題の背後にあるIVジェネレータ
- [include/crypto/utils.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/crypto/utils.h) と [lib/crypto/memneq.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/crypto/memneq.c) — `crypto_memneq()`/`__crypto_memneq()`。事例2の背後にある定数時間比較
- [commit 6bf37e5aa90f](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6bf37e5aa90f18baf5acf4874bca505dd667c37f) — "crypto: crypto_memneq - add equality testing of memory regions w/o timing leaks"。`crypto_memneq()` を導入したLinux 3.13のコミット
- [drivers/char/random.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/char/random.c) — `add_hwgenerator_randomness()` と、事例3の背後にある `pr_notice("crng init done\n")` のログ行
- [drivers/char/hw_random/virtio-rng.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/char/hw_random/virtio-rng.c) — virtio-rngゲストドライバの `hwrng.read` コールバック。事例3の修正は、ホストのエントロピーをゲストのプールに供給するためにこれに依存している
- [security/keys/proc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/security/keys/proc.c) — `proc_keys_show()` とその `key_task_permission(..., KEY_NEED_VIEW)` チェック。事例4の背後にある `/proc/keys` のviewパーミッションによるフィルタリング
- [include/linux/crypto.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/crypto.h) — `CRYPTO_ALG_TESTED`（`0x00000400`）。事例5の背後にあるセルフテストの関門
- [crypto/testmgr.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/crypto/testmgr.c) と [crypto/tcrypt.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/crypto/tcrypt.c) — 事例5の修正で言及した、必須のセルフテストフレームワークと単体のストレステスト用モジュール

### manページ {#man-pages}

- [`cryptsetup(8)`](https://man7.org/linux/man-pages/man8/cryptsetup.8.html) — 事例1全体で使ったLUKS/dm-cryptのコマンドラインツール
- [`getrandom(2)`](https://man7.org/linux/man-pages/man2/getrandom.2.html) — 事例3の背後にある、シードされるまでブロックする動作を記述している
- [`random(4)`](https://man7.org/linux/man-pages/man4/random.4.html) — `/dev/random` と `/dev/urandom` の違い、エントロピープールの初期化、ブート初期に `getrandom(2)` が推奨される理由
- [`keyctl(1)`](https://man7.org/linux/man-pages/man1/keyctl.1.html) — 事例4で使った `setperm`、`print`、およびview/read/write/search/link/setattrのパーミッションビット

### 関連ページ {#related-pages}

- [カーネル暗号API](crypto-api.md) — AEAD、SKCIPHER、アルゴリズムの登録モデル
- [dm-cryptとfscrypt](encryption.md) — IVモードと暗号の選択
- [カーネルkeyring](keyring.md) — 鍵タイプとパーミッションモデル
- [crypto_engine](crypto-engine.md) — ハードウェアドライバのためのフォールバックパターン
- [乱数生成](rng.md) — ブート時のエントロピー

### LWN記事 {#lwn-articles}

- [Constant-time instructions and processor optimizations](https://lwn.net/Articles/921511/)（2023年2月） — タイミングサイドチャネルと、定数時間コードとCPUレベルの最適化との間の緊張関係。事例2の直接の背景
- [FIPS-compliant random numbers for the kernel](https://lwn.net/Articles/877607/)（2021年12月） — ブート時のエントロピー収集を高速化することをめぐる議論。事例3のより広い背景

### 外部資料 {#external}

- [Kernel documentation: dm-crypt](https://docs.kernel.org/admin-guide/device-mapper/dm-crypt.html) — `aes-cbc-essiv:sha256` や `aes-xts-plain64` を含む `cipher-chainmode-ivmode[:ivopts]` 指定形式。事例1の背景
- [Kernel documentation: hw_random](https://docs.kernel.org/admin-guide/hw_random.html) — ハードウェアRNGドライバ（virtio-rngを含む）が供給する `/dev/hwrng` フレームワーク。事例3の修正の背景
- [Kernel documentation: Key Service (core)](https://docs.kernel.org/security/keys/core.html) — `/proc/keys` のセマンティクス、view/read/write/search/link/setattrのパーミッションビット、`logon` 鍵タイプ。事例4の背景
- [Kernel documentation: Kernel Crypto API Architecture](https://docs.kernel.org/crypto/architecture.html) — `/proc/crypto` に表示される `priority` と `selftest` フィールド、および優先度に基づく実装の選択。事例5の背景
