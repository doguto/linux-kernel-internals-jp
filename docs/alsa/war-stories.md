# 事件簿: ALSAのバグとリグレッション {#war-stories-alsa-bugs-and-regressions}

> ALSAサウンドサブシステムで起きた6件の事件。6件すべてがCVEであり、USBオーディオのディスクリプタ解析、PCMバッファのロック、rawmidiのリサイズ競合、そして21年近く誰にも踏まれずに眠っていたuse-after-freeにまたがる

[ALSA概要](README.md)、[PCMデータパス](pcm.md)、[ASoC](asoc.md)は、現在動いているサブシステムの姿を説明している。このページは、そのアーキテクチャの一部の背後にある事件の記録である。実際のドライバが出荷してしまったライフタイムとロックのバグ、そしてそれらがどのように発見され修正されたかを扱う。

[DRMの事件簿](../drm/war-stories.md)は全件が信頼性のバグ（GPUのハングであって、セキュリティ境界の問題ではない）でCVEは1件もないが、ここに挙げる事件はすべてCVEである。6件すべてだ。これは2つのサブシステムが何を外部に晒しているかという構造的な違いによる。`sound/usb/`は接続されたUSBデバイスが何であれ、そのデバイスが提供するデータを解析する。PCM/rawmidiは`/dev/snd/*`を開ける任意のプロセスにioctlを直接公開している。どちらも、GPUスケジューラの内部ジョブキューにはほぼ当てはまらない意味で、攻撃者の影響を受ける入力面である。

## 詳細解説 {#deep-dives}

### [21年間潜伏したUSBミキサー解体時のuse-after-free](war-stories/mixer-teardown-use-after-free.md) {#the-21-year-latent-usb-mixer-teardown-use-after-free}
**2026年1月 · CVE-2026-23089**
2005年のコミットにより、登録されたすべてのALSAコントロールがUSBミキサーオブジェクトへの逆ポインタを持つようになった。probeが失敗すると、同じprobe内で先に作成されたコントロールがまだ指している状態のまま、そのオブジェクト全体が解放されうる。このコードパスでは、修正が入るまでほぼ21年間このバグが起こりうる状態だった。

### [USBオーディオのクロックディスクリプタにおける境界外読み出し](war-stories/usb-audio-clock-descriptor-oob.md) {#the-usb-audio-clock-descriptor-out-of-bounds-reads}
**2024年11月 · CVE-2024-53150 · CISA Known Exploited Vulnerabilitiesカタログ掲載**
3つの関数が、USBオーディオデバイスのクロックトポロジーディスクリプタを辿る際に、どのディスクリプタについても読み出すフィールドを保持できるだけの長さがあるかを検証していなかった。この3つすべてを対象とするCVEは、CISAのKnown Exploited Vulnerabilitiesカタログに掲載されている。

## 短い事例 {#quick-cases}

### 事例1: PCMの`hw_params`/`hw_free`競合 — CVE-2022-1048 {#case-1-the-pcm-hw_paramshw_free-race-cve-2022-1048}

ALSA PCMには、`hw_params`/`hw_free`のioctl呼び出し全体を覆うクリティカルセクションがなかった。既存の`snd_pcm_stream_lock`では呼び出し全体をカバーできなかった。両ioctlともスリープする必要があるからである（特にメモリ割り当てのため）。同じsubstreamに対して2つのスレッドが`hw_params`と`hw_free`を並行して発行すると（あるいは同じioctlを2つのスレッドから競合させても）、一方のスレッドの`hw_free`がバッファの裏付けメモリを解放している間に、もう一方のスレッドの`hw_params`がまだそれを使っている、という状況が起こりえた。これは境界チェックの欠落ではなく、ロックの欠落によるuse-after-freeである。

Takashi Iwaiによる修正（[`92ee3c60ec9f`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=92ee3c60ec9fe64404dc035e7c41277d74aa26cb)）は、`include/sound/pcm.h`に新しい`runtime->buffer_mutex`を追加し、`sound/core/pcm.c`でruntimeと一緒に初期化・破棄し、`sound/core/pcm_native.c`で`snd_pcm_hw_params()`と`snd_pcm_hw_free()`の両方をそれで囲む。mutexを使うのはクリティカルセクションがスリープする必要があるからであり、既存のストリームロック（spinlock）ではそれができなかった。[NVD: CVE-2022-1048](https://nvd.nist.gov/vuln/detail/CVE-2022-1048)、CVSS 3.1 7.0 HIGH。

### 事例2: rawmidiバッファのリサイズ競合 — CVE-2020-27786 {#case-2-the-rawmidi-buffer-resize-race-cve-2020-27786}

`sound/core/rawmidi.c`にあるrawmidiの読み出し/書き込みパスは、ユーザー空間との間でコピーする間、runtimeのspinlockをアンロックしなければならない。コピーはフォールトしてスリープしうるので、ユーザー空間とのコピーをまたいでspinlockを保持することはできない。まさにそのアンロックの隙間に、並行する`SNDRV_RAWMIDI_IOCTL_PARAMS`呼び出しが、読み出しや書き込みがまだ使っているruntimeバッファをリサイズ（解放して再割り当て）しえた。

Takashi Iwaiによる修正（[`c1f6e3c818dd`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c1f6e3c818dd734c30f6a7eeebf232ba2cf3181d)）は、spinlockで保護された素朴な参照カウンタ（`buffer_ref`）を追加する。`refcount_t`ではないのは、カウンタが置かれているロックによってすべてのアクセスがすでに直列化されているからである。そして、読み出しや書き込みが参照を保持している間はリサイズが`-EBUSY`を返すようにする。受信/送信の割り込みコールバックには同じチェックは必要ない。それらはすでにロックで完全にカバーされている。[NVD: CVE-2020-27786](https://nvd.nist.gov/vuln/detail/CVE-2020-27786)、CVSS 3.1 7.8 HIGH。

### 事例3: クロックディスクリプタの5年前に起きたmixer-unitディスクリプタの境界外アクセス — CVE-2019-15117 {#case-3-the-mixer-unit-descriptor-oob-five-years-before-the-clock-descriptors-cve-2019-15117}

`sound/usb/mixer.c`の`parse_audio_mixer_unit()`は`uac_mixer_unit_descriptor`を読み取り、それを`uac_mixer_unit_get_channels()`に渡す。この関数はデバイスが提供する`bNrInPins`の数を使って`baSourceID[]`配列を辿るが、ディスクリプタがそれだけの数のエントリを実際に含めるだけの長さを持っているかをチェックしていなかった。報告者はUSBファジングの研究者であるHui PengとMathias Payerである。

修正は、`uac_mixer_unit_get_channels()`内の配列走査の前に、境界チェックを1つ追加するものである: `if (desc->bLength < sizeof(*desc) + desc->bNrInPins) return -EINVAL;`（[`daac07156b33`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=daac07156b330b18eb5071aec4b3ddca1c377f2c)）。これは、5年後に別の`sound/usb/`パーサで[クロックディスクリプタの詳細解説](war-stories/usb-audio-clock-descriptor-oob.md)が再び踏んだのと同じ、長さ検証欠落のパターンである。[NVD: CVE-2019-15117](https://nvd.nist.gov/vuln/detail/CVE-2019-15117)、CVSS 7.8 HIGH。

### 事例4: caiaqのprobeエラー処理におけるuse-after-free — CVE-2026-46004 {#case-4-the-caiaq-probe-error-handling-use-after-free-cve-2026-46004}

Native Instrumentsのcaiaq USBオーディオドライバのprobeルーチン`setup_card()`は、`snd_card_register()`が失敗するとカードを解体するために`snd_card_free()`を呼び、そのあとも実行を続けて、解放されたばかりの構造体に対して`snd_usb_caiaq_control_init()`を呼び出していた。オーディオ、MIDI、入力の初期化呼び出しは`setup_card()`内でもっと前、`snd_card_register()`に到達する前に行われるため、失敗が起きた時点ではすでに正常に完了していた。解放済みメモリに触れたのは、`snd_card_register()`の*後*に走るコントロール設定の呼び出しだけである。`setup_card()`は`void`だったので、エラー時に早期に止まる手段はなく、続行するしかなかった。

Takashi Iwaiによる修正（[`28abd224db4a`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=28abd224db4a49560b452115bca3672a20e45b2f)）は、`setup_card()`が`int`を返すように変更し、すべてのエラーパスに早期`return`を追加し、不要になった`snd_card_free()`呼び出しを削除し（エラーが伝播すれば呼び出し元が後始末をする）、`init_card()`がその戻り値をチェックして伝播するようにする。[NVD: CVE-2026-46004](https://nvd.nist.gov/vuln/detail/CVE-2026-46004)、CVSS 3.1 7.8 HIGH。

## 共通点 {#common-threads}

| パターン | Mixer UAF | Clock OOB | hw_params競合 | Rawmidi競合 | Mixer-unit OOB | caiaq UAF |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| USBディスクリプタの解析（入力を攻撃者が制御する） | — | Yes | — | — | Yes | — |
| 長さ/境界検証の欠落 | — | Yes | — | — | Yes | — |
| ロックがクリティカルセクション全体をカバーしていない | — | — | Yes | Yes | — | — |
| probe/initのエラーパスが古い参照を残す | Yes | — | — | — | — | Yes |
| 修正が入るまで数年/数十年を要した | Yes (~21年) | — | — | — | — | Yes (~18.5年) |

**6件中2件はUSBディスクリプタ解析のバグであり、5年の隔たりをおいて、2つの異なるパーサで同じチェックが欠けていた。** mixer-unitの境界外アクセス（2019年）とクロックディスクリプタの境界外アクセス（2024年）はどちらも、デバイスが提供するカウントフィールドを読み取り、バッファが実際にその長さを持っているかを再検証せずにバッファのさらに先へインデックスすることから生じている。`sound/usb/`には同じ未チェックのパターンの上に作られたディスクリプタ走査処理が複数あり、当時1つを修正しても、ほかの箇所の監査にはつながらなかったようである。

**6件中2件は根本的に同じ形をしている。操作の一部がスリープしなければならないために、ロックが操作全体をカバーできないというものだ。** PCMの`hw_params`/`hw_free`と、rawmidiの読み出し/書き込み対リサイズの競合はどちらも、spinlockで保護された高速パスの*周囲に*mutexや参照カウントを重ねる必要があった。spinlockを保持したままではユーザー空間とのコピーやメモリ割り当てができないからである。どちらの場合も修正は「既存のロックをもっと頑張らせる」ことではなく、「スリープ可能な第2の保護層を追加する」ことだった。

**このページの2つのバグは20年近く潜伏していたが、その理由は異なる。** 上に挙げたUSBディスクリプタとロックのバグは、影響を受けるほぼすべてのカーネルで、適切なデバイスや適切な競合ウィンドウが現れた瞬間に到達可能である。だからこそ、ファジングやCVE探索によって比較的早く見つかった。ミキサー解体のバグ（約21年）は、特定の自明でない順序（いくつかのコントロールがすでに登録され、その*後*のユニットの解析が失敗する）を必要とし、正しく作られたデバイスはそれを実質的に決して生み出さない。これは、ミキサーオブジェクトとALSA自身のコントロールレジストリの間で`id_elems`を共有する方法に組み込まれた設計上の欠陥であり、起こりにくい部分失敗の順序によってのみ露呈した。caiaqのprobeバグ（約18.5年、`setup_card()`は2007年のコミットにさかのぼる）は、より単純なエラー処理の抜けである。早期に止まる手段を持たない`void`の解体関数であり、潜伏していたのは、引き金となる順序が入り組んでいたからではなく、probe時の`snd_card_register()`失敗そのものがまれな出来事だったからにすぎない。どちらも、それを露呈させる失敗パスが一度でも走るまでに、普通に動作するハードウェアでの長い運用期間を要した。

## 関連項目 {#see-also}

- [ALSA概要](README.md) — `struct snd_card`、コントロールの登録、そしてこれらの事件が起きたUSBオーディオのデバイスモデル
- [PCMデータパス](pcm.md) — 事例1の競合が存在する`hw_params`/`hw_free`の仕組み
- [DRMの事件簿](../drm/war-stories.md) — CVEではなく信頼性のバグ（ハング、デッドロック）が大半を占めるサブシステム。バグの種類が信頼性のみの問題ではなく悪用可能なものである場合に「事件簿」が何を意味するかを対比するために
- [ロック](../locking/README.md) — ロックの範囲とクリティカルセクションのバグに関する一般的な背景。事例1と2の背後にあるパターン
