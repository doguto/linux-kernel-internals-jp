# モジュールの事件簿 {#module-war-stories}

> ABIの破壊、署名の失敗、taintの連鎖、バージョニングの意外な落とし穴

## 1. CONFIG_MODVERSIONSのCRC不一致 {#1-the-config_modversions-crc-mismatch}

**症状:** ベンダーカーネル上で、新しくコンパイルしたドライバのロードが即座に失敗した:

```
insmod: ERROR: could not insert module mydriver.ko: Invalid module format
dmesg | tail -2
# mydriver: disagrees about version of symbol module_layout
```

**何が起きたか:** エンジニアは、名目上は同じバージョンのバニラなアップストリームのソースツリーに対してモジュールをコンパイルしていた。本番サーバーで動いていたのはベンダーカーネルで、後のリリースサイクルからいくつかのパッチがバックポートされており、そのうちの1つが`module_layout`の依存する内部構造体のレイアウトを変更していた。型シグネチャの走査がその変更された構造体に到達するため、ベンダーの`genksyms`はアップストリームの`genksyms`とは異なる`module_layout`のCRCを計算した。

カーネルの視点からすると、ABIが変わっていたことになる — そして`module_layout`は、ローダーが真っ先にチェックするものである。`load_module()`は`early_mod_check()`を呼び出し、それがモジュールイメージ内の他の何かを使うより前に`check_modstruct_version()`を呼び出す。これは`module_layout`アンカーシンボルに対するカーネル側のCRCを検索し、モジュールの`__versions`テーブルと比較して、`disagrees about version of symbol module_layout`を出力し、0を返す — `check_version()`は述語であって、errnoの発生源ではない。失敗を変換するのは呼び出し側である。`early_mod_check()`が`-ENOEXEC`を返し、これが`insmod`の報告した`Invalid module format`である。

どちらのパスが発火したのかを正確に把握しておく価値がある。パスは2つあり、それぞれ挙動が異なるからだ。もし代わりに*通常の*エクスポートシンボルが不一致だったなら、モジュール構造体のチェックは通過し、失敗はずっと後のシンボル解決の段階で表面化していただろう。`resolve_symbol()`は`check_version()`の失敗を`ERR_PTR(-EINVAL)`に変換し、続いて`simplify_symbols()`が`mydriver: Unknown symbol <name> (err -22)`を出力する — `-22`は`EINVAL`である。`module_layout`についてはそのような行は決して現れない。`module_layout`はCRCテーブルのエントリとしてしか存在せず、モジュールにはそれへの未定義ELF参照が含まれないので、シンボル解決はそれをまったく見ないからである。

**診断が難しかった理由:** ビルドでは警告が一切出なかった。アップストリームのヘッダはベンダーのものと同じプロトタイプを宣言していたので、ドライバ自身のソースには何もおかしく見える箇所がなく、実際に変更された構造体はそこでは一度も名前が出てこなかった — 不一致は、`genksyms`は追跡するが人間はめったに調べない推移的な型依存を経由していた。さらに悪いことに、モジュールの`__versions`セクションに入る数値は、ドライバのソースから再計算されるわけではまったくない。`modpost`がビルドツリーの`Module.symvers`からそれをコピーする。誤ったCRCがビルド時に、誤ったツリーから焼き込まれ、ビルド時にはそれを指摘するものが何もない。

**修正:** ツリー外モジュールは、常にターゲットカーネルの正確な`Module.symvers`に対してビルドすること。稼働中のカーネルのビルドディレクトリから`Module.symvers`をコピーする:

```bash
# アップストリームのものではなく、ベンダーカーネルの Module.symvers を使う
cp /lib/modules/$(uname -r)/build/Module.symvers .

# 他のツリー外モジュールに依存するモジュールの場合は、
# それらの Module.symvers ファイルを連結する:
export KBUILD_EXTRA_SYMBOLS="/path/to/dep1/Module.symvers /path/to/dep2/Module.symvers"
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules
```

**教訓:** CRCの不一致はカーネルバージョンだけの問題ではない。ベンダーは、カーネルのバージョン文字列を変えずに、エクスポートされたABIを変更するパッチを日常的にバックポートする。真実の拠り所は`Module.symvers`であり、カーネルヘッダではない。

---

## 2. モジュール署名のループ {#2-the-module-signing-loop}

**症状:** あるディストリビューションが`CONFIG_MODULE_SIG_FORCE=y`に切り替えた後、署名されていないサードパーティモジュールはすべてロードを拒否された。管理者はカスタム署名鍵を登録し、モジュールに署名した。すべて正常に動作した — 次のカーネル更新までは。更新後、モジュールは再びロードを拒否された:

```
# After kernel update:
modprobe mydriver
# modprobe: ERROR: could not insert 'mydriver': Key was rejected by service
```

**何が起きたか:** 管理者は、最初の試行でうまくいった唯一の方法で公開署名証明書を登録していた。ディストリビューションのカーネルソースパッケージを、証明書を`CONFIG_SYSTEM_TRUSTED_KEYS`に追加した状態で再ビルドし、その1つのカーネルイメージにコンパイルして組み込んだのである。それはうまくいき、そのイメージがデフォルトのブートエントリである限りは動作し続けた。

その後、ディストリビューションがカーネルのエラッタ更新を出荷した。パッケージマネージャは*標準の*ディストリビューションカーネルをインストールし、それが新しいデフォルトのブートエントリになった。そしてそのイメージには、ディストリビューション自身の署名鍵しか組み込まれていなかった。カスタム鍵は消えていた — それは最初から、ローカルでビルドされた1つのvmlinuzの中以外のどこにも存在していなかったのである。

モジュールは依然として管理者の秘密鍵で署名されていた。署名は有効だった。しかし稼働中のカーネルは対応する公開鍵の記録を持っておらず、モジュールの検証を拒否した。

**診断が難しかった理由:** モジュールの署名は変わっていなかった。`modinfo mydriver.ko | grep sig`は依然として署名を表示していた。`Key was rejected by service`というエラーは、署名そのものではなくキーリングを指していた。管理者は当初、モジュールの再署名が必要だと考えた。

**修正:** 公開証明書をUEFI MOK（Machine Owner Key）データベースに登録する。これはUEFI NVRAMに永続化され、カーネル更新後も残る — ただし登録は仕事の半分にすぎない。MOK証明書が*どの*カーネルキーリングに入るかによって、モジュール署名がそれを使えるかどうかがそもそも決まるからである。

`kernel/module/signing.c`の`mod_verify_sig()`は、`VERIFY_USE_SECONDARY_KEYRING`でモジュール署名を検証する。これは`.secondary_trusted_keys`に解決され、そこには`.builtin_trusted_keys`がリンクされている — そして`CONFIG_INTEGRITY_MACHINE_KEYRING=y`であれば`.machine`もリンクされる（`certs/system_keyring.c`の`set_machine_trusted_keys()`がブート時にその`key_link()`を行う）。`.platform`キーリングは*別の*信頼ストアである。そのKconfigのヘルプテキストは、これをプラットフォーム/ファームウェアの鍵を保持するもので、「for verifying the kexec'ed kerne[l] image and, possibly, the initramfs signature」（kexecされたカーネルイメージ、および場合によってはinitramfsの署名の検証用）と説明している（アップストリームのヘルプテキスト自体は「kerned」となっている）。モジュール署名の検証はこれを決して参照しない。`CONFIG_INTEGRITY_MACHINE_KEYRING`自身のヘルプテキストも同じ線引きを明示している: 「Unlike keys in the platform keyring, keys contained in the .machine keyring will be trusted within the kernel.」（platformキーリングの鍵とは異なり、.machineキーリングに含まれる鍵はカーネル内で信頼される。）

したがって、`CONFIG_INTEGRITY_MACHINE_KEYRING`なしでビルドされたカーネル（このオプションはv5.18でアップストリームに入った。コミット`d19967764ba8`）では、MOKへの登録はモジュールのロードに対して何の効果もない。鍵は`.platform`にロードされ、使われないままそこに置かれる。MOKの経路がそもそも使えると想定する前に、稼働中のカーネルのconfigを確認すること:

```bash
# このカーネルは、モジュール署名が到達できるキーリングに MOK 鍵を受け入れるか?
grep CONFIG_INTEGRITY_MACHINE_KEYRING /boot/config-$(uname -r)
# CONFIG_INTEGRITY_MACHINE_KEYRING=y

# 公開証明書を MOK にインポートする
sudo mokutil --import signing_key.der
# (次回再起動時に、システムが一回限りの登録パスワードを求める)

# UEFI では、MokListTrustedRT 変数が存在する場合にのみカーネルは MOK 鍵を信頼する
# -- machine_keyring.c の uefi_check_trust_mok_keys() がこれを探す
sudo mokutil --trust-mok

# 再起動して MOK Manager の UI で両方のプロンプトを確認した後:
keyctl list %:.machine | grep "your key CN"

# ...そして、署名の照合先となるキーリングから到達可能であることを確認する
# -- .machine はこのキーリングにリンクされている
keyctl list %:.secondary_trusted_keys

# モジュールに署名する
/usr/src/linux-headers-$(uname -r)/scripts/sign-file \
    sha256 signing_key.pem signing_key.der mydriver.ko
```

**教訓:** カーネルイメージに埋め込まれた鍵はカーネルごとであり、MOKデータベースの鍵はマシンごとである。したがって、カーネル更新後も残る必要がある鍵には、MOKが正しい登録経路である。しかし「登録済み」は「モジュールに対して信頼されている」と同じではない。MOK証明書がモジュール署名の検証に到達するのは`.machine` → `.secondary_trusted_keys`を経由する場合だけであり、それには稼働中のカーネルで`CONFIG_INTEGRITY_MACHINE_KEYRING`が有効であること、さらにUEFIでは明示的な`mokutil --trust-mok`が必要である。`.platform`に入った場合、`mokutil --list-enrolled`では成功したように見えるが、モジュールがロードされるかどうかについては何も変わらない。

---

## 3. taintの連鎖 {#3-the-taint-cascade}

**症状:** 本番のデータベースサーバーが、`blk_mq_complete_request()`でのNULLポインタ参照によりクラッシュした。カーネルのoopsはアップストリームにバグ報告として提出された。カーネル開発者は数時間以内に、taintされていないカーネルでの再現を求めてその報告をクローズした。

```
BUG: kernel NULL pointer dereference, address: 0000000000000008
...
Call Trace:
  blk_mq_complete_request+0x42/0x90
  nvme_complete_rq+0x31/0x70
  ...
Tainted: P           OE
# P = proprietary module loaded
# O = out-of-tree module loaded
# E = unsigned module loaded
```

**何が起きたか:** メトリクスの高速化にCUDAを使う監視ツールのために、プロプライエタリなGPUドライバがロードされていた。GPUドライバはブート時にロードされ、初期化中のおよそ30秒間使われた後、監視ツールはCPUのみのモードに移行した — しかしモジュールはロードされたままだった。

3日後、ブロック層のまったく無関係なバグがクラッシュを引き起こした。そのバグは本物で、再現可能で、GPUドライバが原因ではなかった。しかしtaintフラグ`P`が立っており、それはクリアされなかった。

**なぜ問題になるか:** グローバルなtaintフラグは、`kernel/panic.c`にあるファイルスコープの`static unsigned long tainted_mask`ただ1つに格納されており、`set_bit()`を行う`add_taint(flag, lockdep_ok)`と、ワード全体を返す`get_taint()`を通してしかアクセスできない。モジュールごとのtaintは`struct module::taints`で別途追跡される。カーネル内で`tainted_mask`のビットをクリアするものは何もない — `proc_taint()`は`/proc/sys/kernel/tainted`への書き込みを受け付けるが、そのコメント自体が、taint値は増やすことしかできないと述べており、新しいビットをORで加えるだけである。問題のモジュールをアンロードしても役に立たない。アップストリームの開発者の視点からすると、taintされたカーネルはクリーンなバグ報告の確認には使えない。プロプライエタリなモジュールが30秒間の実行中にカーネルのデータ構造を破壊した可能性を排除できないからである。

```bash
# 現在の taint フラグを確認する
cat /proc/sys/kernel/tainted
# 12289  (P + O + E: プロプライエタリ、ツリー外、署名なしのモジュールがロードされた --
#         bit 0 + bit 12 + bit 13 = 1 + 4096 + 8192)
# ビットの定義は Documentation/admin-guide/tainted-kernels.rst を参照

# taint ビット 0 (値 1): P — プロプライエタリなモジュールがロードされた
# taint ビット 1 (値 2): モジュールの強制ロード
# taint ビット 12 (値 4096): O — ツリー外モジュール
# taint ビット 13 (値 8192): E — 署名なしモジュール
```

**修正:** 実行時の修正方法はない — taintビットは再起動しなければクリアできない。アップストリームへのバグ報告のためには:

1. 現在のブートでプロプライエタリなモジュールやツリー外モジュールを一度もロードしていないカーネルでバグを再現する。カーネルコマンドラインでモジュールをブラックリストに入れ（`module_blacklist=nvidia`。`kernel/module/main.c`の`blacklisted()`がチェックするカンマ区切りのリスト）、再起動するのが、そこに到達する唯一の確実な方法である。事後の`rmmod`は何の効果もない。モジュールをアンロードしても、`add_taint()`がすでに立てたビットはクリアできないからである。
2. ワークロードにプロプライエタリなモジュールが必要な場合は、ハードウェアベンダーにバグを報告する。
3. 報告用のカーネルを`CONFIG_MODULE_UNLOAD_TAINT_TRACKING=y`でビルドする。これはtaintをクリアしない — クリアできるものは何もない — が、taintの原因を特定可能にする。`kernel/module/tracking.c`は`unloaded_tainted_modules`リストを保持し、`print_unloaded_tainted_modules()`がoopsに`Unloaded tainted modules:`行を追加して、カーネルをtaintした後にアンロードされた各モジュールの名前を示す。これにより「3日前に何かがこのカーネルをtaintした」が具体的なモジュール名になり、メンテナがその報告をまだ調べる価値があるかどうかを判断するには、多くの場合それで十分である。

**教訓:** taintはセッションレベルのフラグである。プロプライエタリなモジュールをほんの短時間ロードしただけでも、カーネルは稼働時間の残り全体にわたってtaintされる。アップストリームへのバグ報告に使う本番カーネルでは、プロプライエタリなモジュールを決してロードしないというポリシーを持つべきである。

---

## 4. initセクションのuse-after-free {#4-the-init-section-use-after-free}

**症状:** ある組み込みシステムは何日も安定して動作した後、マップされていないメモリを指すコールトレースを出してクラッシュした。クラッシュアドレスは発生のたびに異なり、中程度のI/O負荷の下でしか現れなかった。本番ではKASANは有効になっていなかった。

```
BUG: unable to handle kernel paging request at ffffffffc03a8120
...
Call Trace:
  [<ffffffffc03a8120>] ? 0xffffffffc03a8120
  [<ffffffffc0401f30>] my_device_event_handler+0x28/0x50 [mydriver]
```

**何が起きたか:** `probe()`の間に、ドライバは永続的なコールバック構造体に関数ポインタを格納していた:

```c
static void __init mydriver_init_error_handler(struct my_device *dev)
{
    /* 初期化中にしか発生し得ないエラーを処理する */
}

static int __init mydriver_probe(struct platform_device *pdev)
{
    struct my_device *dev = /* ... */;

    /* バグ: __init 関数のポインタを永続的な構造体に格納している */
    dev->error_handler = mydriver_init_error_handler;

    platform_set_drvdata(pdev, dev);
    return 0;
}

static struct platform_driver mydriver_driver = {
    .remove = mydriver_remove,
    .driver = { .name = "mydriver" },
};

/* 注: .probe フィールドはない。platform_driver_probe() は probe 関数を
 * 別の引数として受け取る。まさにそれを __init に置けるようにするためである。 */
module_platform_driver_probe(mydriver_driver, mydriver_probe);
```

`mydriver_init_error_handler`は`__init`とマークされていたので、`.init.text`セクションに置かれていた。モジュールの場合、initセクションは**その特定のモジュールの**`mod->init()`が正常に戻った直後に`do_free_init()`（ワークアイテム）を介して解放される — すべてのinitcallが完了した後ではない。`dev->error_handler`の関数ポインタは、もはやモジュールが所有していないメモリを指すことになった。

デバイスイベントが`dev->error_handler`を起動したとき、CPUはもはやこのドライバのコードではないアドレスへジャンプした。

**なぜビルドは何も言わなかったか:** `scripts/mod/modpost.c`は、毎回のビルドですべての再配置に対して`check_section_mismatch()`を実行し、その最初のルール — `fromsec = { TEXT_SECTIONS, DATA_SECTIONS }`、`bad_tosec = { ALL_INIT_SECTIONS, ... }` — はまさにこれを捕捉するために存在する。それが発火しなかったのには2つの理由があり、ここではたまたまその両方が成り立っていた。第一に、問題の格納は`mydriver_probe()`の中にあり、それ自体が`__init`である。再配置は`.init.text` → `.init.text`であり、initコードがinitコードを参照することを禁じるルールはない。第二に、`__init`なprobeが*実際に*捕捉される通常の経路 — `.data`内の`struct platform_driver`の`.probe`が`.init.text`を指しているという、`DATA_SECTIONS` → `ALL_INIT_SECTIONS`の不一致であり、`secref_whitelist()`の`*_ops`/`*_console`の除外にも該当しないもの — は、`platform_driver_probe()`を通して登録することで回避されていた。その関数のヘッダコメントには、これは「probe() and its support may live in `__init` sections, conserving runtime memory.」（probe()とその補助コードを`__init`セクションに置き、実行時のメモリを節約できるようにする）ために存在すると書かれている。ドライバは`__init`なprobeのための公式に認められたパターンを使い、そのうえでそのパターンが依存する唯一の不変条件 — probeの後に、その中を指すポインタを保持したまま生き残るものは何もない — に違反したのである。

**なぜ断続的だったか:** `do_free_init()`は`execmem_free()`を呼び出し、それが何をするかはカーネルによって異なる。`CONFIG_ARCH_HAS_EXECMEM_ROX`がなければ、`execmem_cache_free()`は処理を辞退し、`execmem_free()`は`vfree()`にフォールスルーする — その範囲はアンマップされ、そこへのジャンプは即座に、決定論的にページフォールトを起こす。上のトレースがそれである。`CONFIG_ARCH_HAS_EXECMEM_ROX=y`のx86-64では、その範囲は代わりにexecmemのROXキャッシュに戻される。`__execmem_cache_free()`はその範囲をトラップ命令（x86では`INT3`）で埋め、**マップされたまま**にして、次に実行可能メモリを必要とするモジュールに渡せるようにしておく。これにより、現代の失敗モードは予測しやすくなるどころか、厳密に予測しにくくなる。まだ誰もその範囲を再利用していなければ、呼び出しは`INT3`に着地してトラップする。後の`insmod`がすでにその範囲を受け取っていれば、呼び出しは無関係なモジュールのコードの途中に着地し、そのバイト列が行うことを何でも行う。つまりクラッシュのシグネチャは、その後マシンが他に何をロードしたかに依存する — これこそが、このバグが何日も隠れ、二度と同じ形で再現しなかった理由である。

**修正:** `__init`関数へのポインタを、init段階より長く生きる構造体に決して格納しないこと。init後に呼ばれる可能性のある関数から`__init`アノテーションを取り除くか、永続的なコールバックが非init関数を指すようにコードを再構成する:

```c
/* 選択肢1: __init を取り除く — 関数は init 後も保持される */
static void mydriver_error_handler(struct my_device *dev) { /* ... */ }

/* 選択肢2: 状態をチェックする非 __init のラッパーを使う */
static void mydriver_error_handler(struct my_device *dev)
{
    if (dev->init_complete)
        return;  /* init 後のエラーは黙って無視する */
    /* init 段階のエラーを処理する */
}
```

この種のバグに対するビルド時のセーフティネットは`modpost`であり、sparseではない — sparseには`__init`やELFセクションという概念がないので、`make C=1`はこれについて一言も発しない。`check_section_mismatch()`は追加のフラグなしに毎回のビルドのMODPOST段階で実行され、通常のテキストやデータから`.init.*`への再配置をすべて捕捉する。捕捉できないのは、今回のように、initコンテキストから行われるinitポインタの*実行時の*格納である。modpostが見る再配置はinitからinitへのものであり、これは正当だからだ。そうしたものには実行時の解析が必要である — KASAN、あるいは「このポインタはprobeより長く生きるか?」を毎回問う価値のある問いとして扱うコードレビューである。

**教訓:** `__init`は強い約束である — そのメモリは*必ず*解放される。永続的なデータ構造に漏れ出した`__init`関数へのポインタはすべて、時間差のuse-after-freeである。

---

## 5. 負のリングサイズ {#5-the-negative-ring-size}

**症状:** あるNICドライバが`insmod`の時点で即座にNULLポインタ参照によりクラッシュした。ただしそれは、自動化スクリプトがデフォルト以外のRXリングサイズを渡したサーバーでのみ起きた。影響を受けたすべてのサーバーが、ブートの同じ時点で、最初の試行で、まったく同じようにクラッシュした。

```
insmod mydriver.ko rx_ring_size=-1
dmesg | tail -4
# mydriver: allocating RX ring
# ------------[ cut here ]------------
# WARNING: ... at mm/page_alloc.c:... __alloc_frozen_pages_noprof+...
# BUG: kernel NULL pointer dereference, address: 0000000000000018
```

**何が起きたか:** モジュールパラメータは符号付きの`int`として宣言され、上限のみで検証されていた:

```c
static int rx_ring_size = 256;
module_param(rx_ring_size, int, 0644);
MODULE_PARM_DESC(rx_ring_size, "Number of RX descriptors (max 4096)");

static int __init mydriver_init(void)
{
    if (rx_ring_size > 4096) {
        pr_err("rx_ring_size too large\n");
        return -EINVAL;
    }
    /* バグ: 下限のチェックがない */
    pr_info("allocating RX ring\n");
    ring = dma_alloc_coherent(dev, rx_ring_size * sizeof(struct rx_desc),
                               &dma_handle, GFP_KERNEL);
    ring[0].status = 0;   /* バグ: NULL チェックもない */
    ...
}
```

ドライバをデプロイする自動化スクリプトにはテンプレートのバグがあり、ある種類のサーバーで`rx_ring_size=-1`を渡していた。`kstrtoint()`（`param_set_int()`が使うパーサ）は、この特定の`int`が論理的には個数であるという概念を持たない — 符号付き32ビット整数に収まる任意の値を受け付け、`-1`は収まる。`if (rx_ring_size > 4096)`のチェックもこれを受け付ける。符号付き比較では`-1 > 4096`は偽なので、ドライバが持っていた唯一の検証はそれをそのまま通してしまった。

危険になったのは割り当てサイズの段階である。`rx_ring_size * sizeof(struct rx_desc)`は、**符号付き**の`int`と**符号なし**の`size_t`を乗算する。Cの通常の算術変換により、乗算の前に`int`オペランドは`size_t`に変換される — 32ビットintの`-1`は64ビットの`-1`に符号拡張され、その後符号なしとして再解釈されて`0xFFFFFFFFFFFFFFFF`（64ビットビルドでの`SIZE_MAX`）になる。それに`sizeof(struct rx_desc)`を掛けても、小さな値に**ラップアラウンドはしない**。積は正確に`SIZE_MAX + 1 - |rx_ring_size| * sizeof(struct rx_desc)`である — このインシデントの`-1`と16バイトのディスクリプタでは`SIZE_MAX`より16バイト小さいだけであり、`INT_MIN`のような桁違いに大きな負の値であっても、`SIZE_MAX`から数十GB以内に収まる。構造体のサイズやコンパイラの違いによってこれが小さすぎる割り当てになることはない — 常に、ほどほどではなく天文学的に大きすぎるのである。

その先で実際に何が起きるかはNICがどのDMAパスを使うかによるが、どちらの場合も結末は同じである。これらのサーバーでは`dma_alloc_coherent()`はダイレクトDMAパスを使っていた。`SIZE_MAX`にこれほど近い値に対する`PAGE_ALIGN()`はオーバーフローして`0`にラップし、その結果のページオーダーは`BITS_PER_LONG - PAGE_SHIFT`として計算される — `MAX_PAGE_ORDER`を優に超える値である。ページアロケータ自身のサニティチェックがそれを捕捉して割り当てを即座に拒否し、`WARNING`をログに出して`NULL`を返す。（NICがIOMMUの変換ドメインの背後にあるサーバーでは代わりに`iommu_dma_alloc()`を通り、同じラップしたサイズからページ数0を計算して別の経路で失敗する — `mm/page_alloc.c`の警告は出ないが、結果は同じ`NULL`である。）いずれにせよこれは決定論的である。このドライバを`rx_ring_size=-1`でロードするすべてのサーバーが、最初の呼び出しで、毎回`NULL`を受け取る。

クラッシュは、ドライバがその次に行うこと、つまり何もしないことから生じる。`ring[0].status = 0`は`dma_alloc_coherent()`が返したばかりの`NULL`を参照し、即座にフォールトする。

**なぜもっと早く捕捉されなかったか:** ドライバは4096までのすべての正の値で正しく動作していた。下限の欠如は、誰も手で負のRXリングサイズを試さなかったのでコードレビューでは見えなかった。また、割り当ての NULL チェックの欠如は、ドライバのテストに使われた小さく正当なサイズでは`dma_alloc_coherent()`が事実上決して失敗しないので見えなかった。互いに無関係な2つの独立した見落とし — 下限がないこと、失敗チェックがないこと — が組み合わさって初めて、どちらかが問題になったのである。

**修正:** 正しいパラメータ型を使い、両方の境界を検証し、割り当てをチェックする:

```c
static unsigned int rx_ring_size = 256;

/* 論理的に個数である値には、int ではなく uint を使う */
module_param(rx_ring_size, uint, 0644);
MODULE_PARM_DESC(rx_ring_size, "Number of RX descriptors (64-4096)");

static int __init mydriver_init(void)
{
    if (rx_ring_size < 64 || rx_ring_size > 4096) {
        pr_err("invalid rx_ring_size %u, must be 64-4096\n", rx_ring_size);
        return -EINVAL;
    }
    ring = dma_alloc_coherent(dev, rx_ring_size * sizeof(struct rx_desc),
                               &dma_handle, GFP_KERNEL);
    if (!ring)
        return -ENOMEM;
    ...
}
```

`unsigned int`への切り替えだけでも、この特定のミスは捕捉できていただろう。`kstrtouint()`は先頭の`-`を即座に拒否するので、`rx_ring_size=-1`は`insmod`の時点で`-EINVAL`としてパースに失敗し、ドライバが実行されるよりずっと前に止まっていたはずである。ただし、それだけでは完全な修正ではない — `kstrtouint()`は依然として`4294967295`を受け付ける。これも同様に無意味なリングサイズであり、型の変更だけでは拒否できない — ので、明示的な範囲チェックはやはり重要である。sysfs経由の実行時の書き込みでも検証が必要なパラメータについては、ロード時だけでなくすべての書き込みで同じ範囲を強制するカスタムの`set`関数を持つ`module_param_cb()`を使うこと。

**教訓:** 上限チェックしかない符号付きパラメータは、あらゆる負の値を通してしまう。そしてサイズ計算における符号付き/符号なしの混在算術は、負の個数を小さな値ではなく`SIZE_MAX`近くの値に変える — アロケータはそれを黙って受け入れるのではなく、きれいに拒否するように作られているので、実際のリスクはラップアラウンドによるオーバーフローではなく`NULL`の返却である。実際にハードウェアまで届くバグは、ほとんど常に2つ目の、無関係なミス、つまり割り当ての失敗をチェックしないことである。個数のようなパラメータは両方の境界を検証し、第二の防御線として符号なし型を優先し、攻撃者や誤った設定がサイズに影響し得る割り当てでは決してNULLチェックを省略しないこと。

---

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [kernel/module/version.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/version.c) — `check_version()`とその`"disagrees about version of symbol"`警告、およびモジュール内の他の何かに触れる前に`module_layout`アンカーシンボルのCRCを検索する`check_modstruct_version()`。事例1の背景
- [scripts/genksyms/genksyms.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/genksyms/genksyms.c) — エクスポートシンボルの完全な型定義を、ネストした構造体も含めて展開し、CRCにするパーサ。事例1のベンダービルドとアップストリームビルドで食い違ったCRCである
- [scripts/mod/modpost.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/mod/modpost.c) — `Module.symvers`からモジュールの`__versions` CRCテーブルを構築する`sym_add_unresolved("module_layout", ...)`と`add_versions()`（事例1）、および非initから`.init.*`への参照を指摘するビルド時チェックである、`ALL_INIT_SECTIONS`ルールを持つ`check_section_mismatch()`（事例4）
- [kernel/module/main.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/main.c) — `check_version()`が不一致を示したシンボルを失敗させる`resolve_symbol()`と、*その*モジュールのinit関数が戻った直後にモジュールの`.init.text`を解放する`do_init_module()`/`do_free_init()`。事例1と事例4の背景
- [include/linux/init.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/init.h) — `__section(".init.text")`に展開される`__init`。事例4のドライバが、そのメモリへの生きたポインタを保持し続けたアノテーションである
- [mm/execmem.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/execmem.c) — `do_free_init()`がモジュールのinitテキストを返却するのに使う`execmem_free()`と、その範囲がアンマップされるか再利用されるかを決めるROXキャッシュ。事例4の背景
- [kernel/module/signing.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/signing.c) — `VERIFY_USE_SECONDARY_KEYRING`でモジュール署名を検証する`mod_verify_sig()`と、`CONFIG_MODULE_SIG_FORCE`の背後にある`sig_enforce`パラメータ。事例2
- [certs/system_keyring.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/certs/system_keyring.c) — `.builtin_trusted_keys`、`.secondary_trusted_keys`、`.machine`、`.platform`の各キーリング。モジュール署名はセカンダリキーリングのチェーンに対して検証され、事例2で登録した鍵が到達しなければならないのはそこである
- [security/integrity/Kconfig](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/security/integrity/Kconfig) — Machine Owner Keyを`.machine`キーリングに受け入れるオプション`CONFIG_INTEGRITY_MACHINE_KEYRING`。そのヘルプテキストは、platformキーリングの鍵とは異なり`.machine`の鍵はカーネル内で信頼されると明記している — 事例2の修正が依拠する区別である
- [scripts/sign-file.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/sign-file.c) — 事例2の修正で呼び出される`sign-file`ツール
- [kernel/panic.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/panic.c) — `add_taint()`/`get_taint()`を伴う`tainted_mask`ビットマスクと、`proc_taint()`。その「Taint values can only be increased」というコメントが、事例3のtaintが決してクリアされない理由である
- [include/linux/panic.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/panic.h) — 事例3で引用した`TAINT_PROPRIETARY_MODULE`（0）、`TAINT_FORCED_MODULE`（1）、`TAINT_OOT_MODULE`（12）、`TAINT_UNSIGNED_MODULE`（13）のビット番号
- [include/linux/module.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/module.h) — 「same bits as kernel:taint_flags」とコメントされた`struct module`の`taints`フィールド。事例3におけるモジュールごとのtaintの記録
- [kernel/module/tracking.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/module/tracking.c) — すでにアンロードされたtaint元モジュールの識別情報を保存する`unloaded_tainted_modules`リスト。事例3に対する救済策に最も近いもの
- [kernel/params.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/params.c) — `STANDARD_PARAM_DEF(int, int, "%i", kstrtoint)`。`param_set_int()`の実体はこれだけであり、値が符号付き`int`に収まることしか検証せず、特定の`int`パラメータが論理的に非負であるという概念を持たない。事例5の背景
- [lib/kstrtox.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/kstrtox.c) — 負の値を含む範囲内の任意の符号付き値を受け付ける`kstrtoint()`。事例5で`rx_ring_size=-1`が問題なくパースされた理由である
- [kernel/dma/direct.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/dma/direct.c) — `dma_direct_alloc()`の`size = PAGE_ALIGN(size)`。事例5の過大な割り当ての背後にあるオーバーフロー箇所
- [include/asm-generic/getorder.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/asm-generic/getorder.h) — 事例5でサイズが0にラップした値に対して`BITS_PER_LONG - PAGE_SHIFT`を返す`get_order()`
- [mm/page_alloc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) — 事例5の割り当て要求が実際に引っかかる`WARN_ON_ONCE_GFP(order > MAX_PAGE_ORDER, gfp)`ガードと、それが返す`NULL`

### manページ {#man-pages}

- [`init_module(2)`](https://man7.org/linux/man-pages/man2/init_module.2.html) — これらの事例が生むロード時のエラー。カーネルが拒否したモジュールイメージに対する`ENOEXEC`（事例1）。manページは鍵が見つからない場合について`ENOKEY`を記載しているが、v5.4（コミット`49fcf732bdae`、「lockdown: Enforce module signatures if the kernel is locked down」）以降、強制モードのカーネルは実際には`EKEYREJECTED` — 「Key was rejected by service」 — を返す。事例2の背景
- [`delete_module(2)`](https://man7.org/linux/man-pages/man2/delete_module.2.html) — 事例3の`rmmod`に関する助言の背後にあるアンロードパスと、その`O_TRUNC`強制アンロードフラグ
- [`modprobe(8)`](https://man7.org/linux/man-pages/man8/modprobe.8.html) — 事例2と事例5で使われたローダー。末尾の`name=value`形式のモジュールパラメータがどのようにカーネルに渡されるかも含む
- [`modinfo(8)`](https://man7.org/linux/man-pages/man8/modinfo.8.html) — `.ko`からモジュール属性を`-F field`で抽出する機能。事例1と事例2の調査手順である（manページ自体のフィールド一覧には`sig*`/`vermagic`は記載されていないが、実際には`modinfo`はそれらを表示する — `Documentation/admin-guide/module-signing.rst`を参照）
- [`keyctl(1)`](https://man7.org/linux/man-pages/man1/keyctl.1.html) — 事例2の確認手順で使った`keyctl list`と、キーリングを名前で指定する`%:<name>`記法

### 関連ページ {#related-pages}

- [モジュールロードの内部](module-loading-internals.md) — `load_module()`、`__versions`のCRCチェック、`Module.symvers`。事例1の背後にある仕組み
- [モジュール署名](module-signing.md) — `CONFIG_MODULE_SIG_FORCE`、署名フォーマット、鍵の登録。事例2の背景
- [モジュールパラメータ、シンボル、Kconfig](module-params.md) — `module_param()`の型、`module_param_cb()`、および同じ`has no CRC!`/バージョン不一致の症状。事例1と事例5の背景
- [Kbuild: カーネルビルドシステム](kbuild.md) — ツリー外の`M=`ビルド。事例1の修正のビルド側
- [カーネルモジュールの作成とロード](module-basics.md) — 事例4の背後にある`__init`/`__exit`アノテーションとモジュールのライフサイクル
- [カーネルキーリング](../crypto/keyring.md) — 事例2の`keyctl`チェックの背後にある鍵の種類とキーリングの検索セマンティクス
- [カーネルoopsの解析](../debugging/oops-analysis.md) — 事例3と事例4で引用したtaintされたコールトレースの読み解き
- [プラットフォームドライバ](../drivers/platform-driver.md) — 事例4のドライバが基づいている`probe()`/`platform_set_drvdata()`のパターン

### LWNの記事 {#lwn-articles}

- [The end of modversions?](https://lwn.net/Articles/707520/)（2016年11月30日） — vermagicとmodversionsの比較、そして変更されたCRCが1つあるだけで、それ以外は互換性のあるカーネルへのモジュールのロードが止まる仕組み。事例1の背後にあるメカニズム
- [A new version of modversions](https://lwn.net/Articles/986892/)（2024年8月26日） — `genksyms`が展開された型情報からシンボルのチェックサムをどのように導出するか。推移的な構造体の変更が事例1のモジュールを壊す理由である
- [The module signing endgame](https://lwn.net/Articles/525592/)（2012年11月21日） — モジュール署名と組み込み公開鍵が、カーネルのビルドおよびインストール手順にどのように組み込まれたか。事例2が引き継いだ構成である
- [Tracing unsigned modules](https://lwn.net/Articles/588799/)（2014年3月5日） — `CONFIG_MODULE_SIG_FORCE`、`module.sig_enforce`ブートパラメータ、署名なしモジュールに対して立てられるtaintフラグ。事例2と事例3にまたがる
- [module: Introduce module unload taint tracking](https://lwn.net/Articles/893584/)（2022年5月2日） — 後でアンロードされたtaint元モジュールの記録を追加するAaron Tomlinのパッチシリーズ。事例3の「どのモジュールがこのカーネルをtaintしたのか?」という問題を正面から狙ったもの
- [Yet another memory allocator for executable code](https://lwn.net/Articles/933867/)（2023年6月8日） — `execmem`となった提案。事例4でモジュールのinitテキストを割り当て・解放するアロケータである

### 外部資料 {#external}

- [Tainted kernels](https://docs.kernel.org/admin-guide/tainted-kernels.html) — 権威あるtaintビットの表（4096 = 外部ビルドのモジュール、8192 = 署名なしモジュール）と、問題のモジュールがアンロードされた後もカーネルはtaintされたままであるという記述。事例3の背景
- [Reporting issues](https://docs.kernel.org/admin-guide/reporting-issues.html) — アップストリームの「taintフラグを確認する」手順と、正常でtaintされていないカーネルでの再現を求めるその主張。事例3のバグ報告が突き当たったルールである
- [Building External Modules](https://docs.kernel.org/kbuild/modules.html) — `Module.symvers`、`KBUILD_EXTRA_SYMBOLS`、`__versions`セクション。事例1の修正を文書化したもの
- [Kernel module signing facility](https://docs.kernel.org/admin-guide/module-signing.html) — `CONFIG_MODULE_SIG_FORCE`、`CONFIG_SYSTEM_TRUSTED_KEYS`、`scripts/sign-file`。事例2の出発点となった構成そのもの
