# 乱数生成 {#random-number-generation}

> /dev/random、getrandom()、DRBG、ハードウェアエントロピー

## 概要 {#overview}

Linuxカーネルの乱数生成器は、2つの異なる利用者に対応している:

- **ユーザー空間**: `/dev/random`、`/dev/urandom`、`getrandom(2)` システムコール
- **カーネル内部**: `get_random_bytes()`、`get_random_u32()`、DRBG暗号API

どちらも最終的には同じエントロピープールを源としており、そのプールはハードウェアイベントと（対応ハードウェアでは）ハードウェアRNG命令からシードされる。

## エントロピープール {#the-entropy-pool}

### 5.6以前: 複数のプール {#pre-56-multiple-pools}

初期のカーネルバージョンでは、カーネルは3つのプールを保持していた:

- `input_pool`: 主となるエントロピー蓄積器（4096ビット、ハッシュベースの混合）
- `blocking_pool`: `/dev/random` に供給していた。推定エントロピーが低いとブロックした
- `nonblocking_pool`: `/dev/urandom` に供給していた。ブロックすることはなかった

`nonblocking_pool` はそれ以前にCRNG出力段に統合され、`blocking_pool` は **Linux 5.6** で削除された。5.6の時点では `input_pool` だけが残り、ChaCha20ベースのCRNG出力段を持つようになった。`/dev/random` がノンブロッキングになったのもこのときである。エントロピーの _推定器_ には異論が多かった。「エントロピーのビット数」を追跡すると称していたが、その実態はほぼヒューリスティックだった。実際には過大推定や過小推定がしばしば起こり、不必要なブロックや誤った安心感を生んでいた。

### 5.17以降: 単一のBLAKE2sプール {#517-a-single-blake2s-pool}

Linux 5.17（2022年） — 主にJason Donenfeldによる改修 — は、複数プール設計を256ビットの単一のBLAKE2sベースのプールに置き換えた。鍵となる洞察は、本物のエントロピーが256ビットあれば、それ以上蓄積しても安全性は上がらないということである。総当たりできない秘密がすでに手元にあるからだ。

新しいプール:

```c
/* drivers/char/random.c (5.17+) */

/* 主プールはBLAKE2sで混合される256ビット（入力プールであり、CRNGの出力ではない） */
static struct {
    struct blake2s_ctx      hash;
    spinlock_t              lock;
    unsigned int            init_bits;  /* これまでに蓄積された量 */
} input_pool = {
    .hash.h = { /* BLAKE2sのIV、出力長で事前に混合済み */ ... },
    .hash.outlen = BLAKE2S_HASH_SIZE,
    .lock = __SPIN_LOCK_UNLOCKED(input_pool.lock),
};
```

`crng_init` カウンタは3つの状態を持つ: 0（未シード）、1（早期シード済み、約128ビットのエントロピー、使用可能だが完全には初期化されていない）、2（完全にシード済み、256ビット）。遷移は0 → 1 → 2である。`crng_init` が2に達した後も、プールは新しいエントロピーから定期的に再シードされるが、CRNGの出力はすでに暗号学的に乱数と区別できない。

CRNG（Cryptographically Secure Random Number Generator）はストリーム暗号としてChaCha20を使う。CPUごとのCRNG（5.18で追加され、それ以前のNUMAノードごとの設計を置き換えた）により、高速かつロックフリーな生成が可能になる:

```c
/* struct crng — 内部型で、ChaCha20の状態をラップする。公開カーネルAPIではない */
/* 各CPUはロックフリーな生成のために独自のCRNGインスタンスを持つ */
```

## エントロピー源 {#entropy-sources}

カーネルは複数の源からプールにエントロピーを供給する:

| 源 | 方法 | 備考 |
|---|---|---|
| 割り込みのタイミング | `add_interrupt_randomness()` — 割り込みハンドラから呼ばれる | 割り込み間のジッタがエントロピーを持つ |
| ハードウェアRNG | `add_hwgenerator_randomness()` — `/dev/hwrng` から | RDRAND、TPM、virtio-rng |
| ディスクI/O | `add_disk_randomness()` によるブロック層のタイミング | 現在も存在し、エクスポートされている（`EXPORT_SYMBOL_GPL`） |
| ブート時 | rng-tools/systemd-random-seedによる前回ブートのシードファイル | VMでは極めて重要 |
| CPUのRDRAND | `arch_get_random_long()` — CRNGの初期化中に使われる | 下記のRDRANDに関する注を参照 |

### RDRAND {#rdrand}

IntelのRDRAND命令（Sandy Bridge、2012年以降）は、チップ上のハードウェアエントロピー源から乱数を生成する。カーネルはRDRANDを _実際に_ 使うが、唯一の源としてではなく、多数の入力のうちの1つとしてのみ使う:

```c
/* arch/x86/kernel/cpu/rdrand.c */
void x86_init_rdrand(struct cpuinfo_x86 *c)
{
    /* テスト: 8個の値を生成し、いずれかが失敗するか十分に変化しなければRDRANDの使用を無効にする */
    ...
}
```

RDRANDだけを信頼しない理由は次のとおりである: Intel（あるいは別の誰か）がRDRANDの実装にバックドアを仕込んでいた場合、それだけに頼るとすべての暗号操作が危殆化する。RDRANDを割り込みジッタや他の源と混合することで、危殆化したRDRANDでも出力を偏らせることはできない。これはTheodore Ts'oによるコミットのカーネル変更履歴で議論された、明示的な設計判断である。

## getrandom(2) {#getrandom2}

```c
#include <sys/random.h>

ssize_t getrandom(void *buf, size_t buflen, unsigned int flags);
```

`getrandom(2)` はLinux 3.17で追加された。`/dev/random` や `/dev/urandom` よりも推奨されるインターフェースである。

動作:

- **フラグなし（0）**: CRNGが完全に初期化される（256ビットのエントロピー）までブロックし、その後、暗号学的に強い乱数バイトを返す。初期化後はブロックしない。
- **`GRND_NONBLOCK`**: デフォルトと同様だが、CRNGがまだ初期化されていない場合はブロックせずに `-EAGAIN` を返す。
- **`GRND_RANDOM`**: かつては「ブロッキングプール」（エントロピー推定器から読む）を選択していた。5.6以降、このフラグは **効果がない** — デフォルトとまったく同じに扱われる。新しいコードでは `GRND_RANDOM` を使わないこと。デフォルトより高い安全性を提供するわけではない。
- **`GRND_INSECURE`**（5.6で追加）: CRNGが初期化されていなくても乱数バイトを返す。暗号以外の用途（例: 完全な初期化前のカーネルASLR）専用である。

```c
/* ユーザー空間での正しい使い方 */
uint8_t key[32];
ssize_t n = getrandom(key, sizeof(key), 0);
if (n < 0) {
    perror("getrandom");
    exit(1);
}
/* これでkeyには暗号学的に強い32バイトの乱数が入っている */
```

`getrandom()` はシグナルによって中断されない（`/dev/random` からの読み出しとは異なる）。256バイト以下の要求では、バッファ全体を返すか失敗するかのどちらかであることが保証される — 短い読み出し（short read）は起こらない。

## /dev/randomと/dev/urandom {#devrandom-and-devurandom}

```
                      ┌───────────────────────────────┐
Entropy sources ────► │  Input pool (BLAKE2s, 256 bits│
                      │  accumulating to crng_init=2) │
                      └──────────────────┬────────────┘
                                         │
                               ChaCha20 CRNG
                              /         |          \
                    /dev/random   /dev/urandom  getrandom()
```

**`/dev/random`**: Linux 5.6以降、`/dev/random` は **ノンブロッキング** である。CRNGが初期化された後に「十分なエントロピー」を待ってブロックすることはもうない。`/dev/urandom` と同じく、CRNGのバイトを出力する。5.6より前は、エントロピーの推定値がしきい値を下回るたびにブロックし、スクリプトが苛立たしく固まる原因となっていた。このブロッキング動作は、誤った安全性の保証を与えるものだと広く認められていた。

**`/dev/urandom`**: 常にノンブロッキングである。ブート時の初期化後は、すべての暗号用途に安全である。CRNGが初期化される前（ブートのごく初期）に `/dev/urandom` を読むのは _技術的には_ 安全ではない（出力が予測可能かもしれない）が、実際にはカーネルは早期に初期化される。初期化前にurandomが読まれると、カーネルは警告をログに出す。

**どれを使うべきか?** 新しいコードでは `getrandom(2)` を使う。ファイルディスクリプタを使わなければならないなら、`/dev/urandom` で問題ない。5.6以降、`/dev/random` は `/dev/urandom` に比べて追加の安全性を何も提供しないし、5.6より前のブロッキング動作は有害だった。

## DRBG: 決定論的乱数ビット生成器 {#drbg-deterministic-random-bit-generator}

カーネルは、NIST SP 800-90Aの決定論的乱数ビット生成器（Deterministic Random Bit Generator）を暗号API経由で実装している（`crypto/drbg.c`）。これらは、NIST準拠のRNG動作を必要とするカーネルサブシステム（IPsec、特定のFIPSモード操作）が使う:

```c
#include <crypto/rng.h>

/* カーネルに登録されたDRBGを割り当てる（汎用名"stdrng"で公開されている） */
struct crypto_rng *rng = crypto_alloc_rng("stdrng", 0, 0);
if (IS_ERR(rng))
    return PTR_ERR(rng);

/* カーネルのエントロピープールからシードする */
u8 seed[32];
get_random_bytes(seed, sizeof(seed));
ret = crypto_rng_reset(rng, seed, sizeof(seed));

/* バイトを生成 */
u8 output[16];
ret = crypto_rng_get_bytes(rng, output, sizeof(output));

crypto_free_rng(rng);
```

このカーネル時点では、`crypto/drbg.c` は単一のアルゴリズムを登録する: `cra_name = "stdrng"`、`cra_driver_name = "drbg_nopr_hmac_sha512"` — SHA-512上に構築された、予測耐性（prediction resistance）なしのHMAC-DRBGである。以前の設計では `drbg_{pr,nopr}_{hmac,hash,ctr}_{hash_alg}` というパターンに従う名前のファミリーを登録しており（呼び出し元が特定のモード/ハッシュを選び、予測耐性の有無を選択できた）、そのファミリーと予測耐性のサポート自体は後の簡素化で削除され、`crypto_alloc_rng("stdrng", 0, 0)` がこれに到達する手段として残った。

## カーネル内部インターフェース {#kernel-internal-interfaces}

乱数データを必要とするカーネルコード向け:

```c
/* 汎用: CRNGが初期化されるまでブロックし、その後バイトを返す */
void get_random_bytes(void *buf, size_t len);

/* 高速なCPUごとの版（ロックフリー、5.18以降） */
u32 get_random_u32(void);
u64 get_random_u64(void);
u32 get_random_u32_below(u32 ceil);  /* [0, ceil)の一様分布 — 6.2で追加 */
```

`get_random_bytes_arch()` と引数なしの `prandom_u32()` — どちらも古いドキュメントで時折参照される古いAPI — は現在のツリーにはもう存在しない。どちらの用途でも `get_random_u32()` が現在の置き換えである。

```c
/* 例: ネットワークプロトコル用のランダムなnonceを生成する */
u8 nonce[16];
get_random_bytes(nonce, sizeof(nonce));

/* 例: リトライループでのランダムなジッタ */
usleep_range(1000 + get_random_u32_below(1000), 3000);
```

## 初期化のタイムライン {#initialization-timeline}

```
Power on
    │
    ├─ rand_initialize() / early kernel init ← CRNG allocated, NOT yet initialized
    │  (internal routines; not part of the stable API)
    │  crng_init = 0
    │
    ├─ Interrupt jitter accumulates  ← add_interrupt_randomness()
    │  RDRAND mixes in               ← arch_get_random_long()
    │
    ├─ systemd-random-seed.service   ← loads saved seed from /var/lib/systemd/random-seed
    │  (or rngd from rng-tools)
    │
    ├─ 256 bits accumulated          ← crng_init = 2
    │  crng_init_done()              ← wake_up_all() for getrandom() waiters
    │  pr_notice("random: crng init done")
    │
    └─ Periodic reseed               ← every CRNG_RESEED_INTERVAL (60 seconds) from fresh entropy
```

ベアメタルでの典型的なブートでは、`crng_init_done` は最初の数秒以内に起こる。virtio-rngも保存済みシードもないVMでは、数十秒以上かかることがある。

初期化状態は次のように確認できる:

```bash
# CRNGが初期化された時刻はカーネルログでわかる
dmesg | grep "crng init done"
# [    2.341801] random: crng init done

# 現在のエントロピー推定値（5.17以降は参考情報にすぎない）
cat /proc/sys/kernel/random/entropy_avail

# プールサイズ（ビット）
cat /proc/sys/kernel/random/poolsize
# 256   （5.17以降は常に256）
```

## VMのエントロピー問題 {#vm-entropy-problem}

仮想マシンは、タイミングを制御するハイパーバイザを共有している。割り込みジッタ — ベアメタルでの主要なエントロピー源 — は、VMでははるかにランダム性が低い:

1. ハイパーバイザは固定間隔で割り込みを配送することがある
2. 同じスナップショットから起動された（あるいはスナップショットを取ってクローンされた）2つのVMは、スナップショット時点で同一のRNG状態を共有する

**フォーク問題**: VMのスナップショットが取られ、2つのインスタンスが同じ状態から再開すると、RNG状態が分岐するまで両者は同一の乱数を生成する。これは現実の暗号上の危険である（同一のTLSセッション鍵、重複したnonce）。

### 解決策 {#solutions}

**virtio-rng**: ホストに乱数バイトを要求する準仮想化デバイス:

```bash
# VM内でvirtio-rngを確認
lsmod | grep virtio_rng
# または
ls /sys/bus/virtio/drivers/virtio_rng/

# ホスト（例: QEMU）は自身の /dev/random からエントロピーを供給する
# ゲストのカーネルはadd_hwgenerator_randomness()でこれを自身のエントロピープールに混合する
```

**ゲスト内のRDRAND / RDSEED**: ハイパーバイザがCPUのRNG命令をパススルーしていれば、VMは本物のハードウェアエントロピーを使う。

**VMフォークの検出**: Linux 5.17以降には、仮想マシン世代ID（VMGENID）ACPIデバイスを介してVMのフォークを検出する仕組みが含まれている。ハイパーバイザが（スナップショットからの再開時に）世代IDを変更すると、カーネルは `add_vmfork_randomness()` を呼んでCRNGを再シードする:

```c
/* drivers/virt/vmgenid.c */
/* ACPI/DTデバイスが128ビットの世代カウンタを公開する */
struct vmgenid_state {
    u8 *next_id;              /* デバイスメモリ内の現在のカウンタへのポインタ */
    u8 this_id[VMGENID_SIZE]; /* 最後に見た値のコピー */
};

static void vmgenid_notify(struct device *device)
{
    struct vmgenid_state *state = device->driver_data;
    u8 old_id[VMGENID_SIZE];

    memcpy(old_id, state->this_id, sizeof(old_id));
    memcpy(state->this_id, state->next_id, sizeof(state->this_id));
    if (!memcmp(old_id, state->this_id, sizeof(old_id)))
        return;  /* 変化なし: フォーク/再開イベントではない */

    add_vmfork_randomness(state->this_id, sizeof(state->this_id));
}
```

ACPI通知ハンドラは、プラットフォームが世代IDの変更を通知したときに `vmgenid_notify(dev)` を呼ぶだけの薄いラッパーである。実際に変更を検出して再シードするのは、上記の比較と再シードのロジックである。

## コンテナ内のgetrandom() {#getrandom-in-containers}

`getrandom()` は **namespace化されていない** — コンテナのnamespaceにかかわらず、ホストカーネルのエントロピープールを使う。コンテナ内のプロセスが `getrandom()` を呼ぶと、ホストと同じプールから乱数バイトを得る。これは正しい: エントロピープールはグローバルな資源であり、共有しても弱まることはない。利用者が増えればエントロピーの混合は増えこそすれ、減ることはない。

コンテナランタイムは `getrandom()` を横取りしない（し、すべきでもない）。

## RNGサブシステムの観測 {#observing-the-rng-subsystem}

```bash
# 現在のエントロピー推定値
cat /proc/sys/kernel/random/entropy_avail
# 256   （5.17以降はプールサイズが上限）

# プールのパラメータ
cat /proc/sys/kernel/random/poolsize      # 256
# 注: read_wakeup_thresholdはLinux 5.17で削除された。write_wakeup_thresholdは
# 現在もsysctlノードとして存在するが、今では名残にすぎない（読み出すと値が返り、書き込みは
# 黙って無視される）。代わりにentropy_avail、poolsize、urandom_min_reseed_secsを使うこと。

# /dev/urandomから生成されたUUID（手軽なテスト）
cat /proc/sys/kernel/random/uuid

# 出力に対するFIPS 140-2統計テスト（rng-toolsが必要）
rngtest -c 1000 < /dev/urandom
# rngtest: bits tested:                 20000
# rngtest: FIPS 140-2 successes:        1000
# rngtest: FIPS 140-2 failures:         0

# 性能テスト
dd if=/dev/urandom of=/dev/null bs=1M count=100 2>&1
# 最新のハードウェアでは100 MB/s以上（ChaCha20 CRNGは高速）

# tracepointでエントロピーイベントを監視（カーネル5.x以降）
perf trace -e random:*
# または（これらのtracepointが有効なカーネルでは）:
trace-cmd record -e random:* sleep 10
trace-cmd report

# ハードウェアRNGが存在し使われているか確認
cat /sys/class/misc/hw_random/rng_current
# virtio_rng.0   （virtio-rngのあるVM内）
# tpm-rng-0      （RNG源としてのTPM）
# intel_rng      （Intelのハードウェア）

# rngdにハードウェアRNGからエントロピーを追加させる
rngd -f -r /dev/hwrng
```

## 関連ソース {#relevant-source}

- `drivers/char/random.c` — エントロピープール、CRNG、getrandom()の実装のすべて
- `crypto/drbg.c` — NIST SP 800-90A DRBGの実装
- `drivers/char/hw_random/` — ハードウェアRNGドライバ（virtio-rng、intel-rng、tpm-rngなど）
- `drivers/virt/vmgenid.c` — VMGENIDによるVMフォークの検出
- `arch/x86/kernel/cpu/rdrand.c` — RDRANDのサポート
- `include/linux/random.h` — カーネル内部のRNG API

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [drivers/char/random.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/char/random.c) — BLAKE2s入力プール、ChaCha20 CRNG、`getrandom()`/`/dev/random`/`/dev/urandom` の実装
- [crypto/drbg.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/crypto/drbg.c) — `crypto_alloc_rng()` を通じて公開されるNIST SP 800-90A DRBG
- [include/linux/random.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/random.h) — `get_random_bytes()`、`get_random_u32()`/`get_random_u64()`、`get_random_u32_below()`、およびエントロピーを供給する `add_*_randomness()` API
- [drivers/virt/vmgenid.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/virt/vmgenid.c) — VMGENID ACPI/DTデバイスによるVMフォークの検出
- [arch/x86/kernel/cpu/rdrand.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/cpu/rdrand.c) — `x86_init_rdrand()`、ブート時のRDRANDセルフテスト

### manページ {#man-pages}

- [`getrandom(2)`](https://man7.org/linux/man-pages/man2/getrandom.2.html) — システムコールのフラグ（`GRND_NONBLOCK`、`GRND_RANDOM`）、シグナル時の動作、256バイトでの短い読み出しなしの保証
- [`random(4)`](https://man7.org/linux/man-pages/man4/random.4.html) — `/dev/random` と `/dev/urandom` デバイスのセマンティクス（Linux 5.6でのノンブロッキング化を含む）

### 関連ページ {#related-pages}

- [カーネル暗号API](crypto-api.md) — `crypto_alloc_rng()` と、このページのエントロピープールの上に構築されたDRBGインターフェース
- [dm-cryptとfscrypt](encryption.md) — `get_random_bytes()`/`/dev/urandom` の出力がボリューム鍵やIVのために消費される場所
- [カーネルkeyring](keyring.md) — 生成された鍵素材が作成後に格納される場所
- [暗号の事件簿](war-stories.md) — 事件3「ブート時の `getrandom()` のブロック」。このページの内容に基づく、実際のエントロピー枯渇の事例

### LWN記事 {#lwn-articles}

- [The search for truly random numbers in the kernel](https://lwn.net/Articles/567055/) — Jonathan Corbet、2013年9月18日 — カーネルがRDRANDだけを信頼せずにプールへ混合する理由
- [A system call for random numbers: getrandom()](https://lwn.net/Articles/606141/) — Jake Edge、2014年7月23日 — このページで扱ったシステムコールの当初の設計議論
- [Uniting the Linux random-number devices](https://lwn.net/Articles/884875/) — Jake Edge、2022年2月16日 — プール書き換えの時期に、`/dev/random` と `/dev/urandom` を同一の動作にしようとしたJason Donenfeldの取り組み

### 外部資料 {#external}

- [NIST SP 800-90A Rev. 1](https://csrc.nist.gov/pubs/sp/800/90/a/r1/final) — "Recommendation for Random Number Generation Using Deterministic Random Bit Generators"、`crypto/drbg.c` が実装する標準
