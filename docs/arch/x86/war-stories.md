# x86アーキテクチャの事件簿 {#x86-architecture-war-stories}

> 実際のインシデント: KPTIのリグレッション、TSCのドリフト、SYSRETのバグなど

x86アーキテクチャの複雑さ（数十年にわたる後方互換性、マイクロアーキテクチャ上の癖、ハードウェアとソフトウェアのインターフェースの微妙な点）は、現実のインシデントを絶え間なく生み出している。ここで取り上げるのは、公開されているカーネルのコミットログ、CVEの開示情報、LWNの記事、ベンダーのアドバイザリから集めた5つのインシデントである。いずれも本番システムに測定可能な影響を与えた。

---

## 事例1: データベースワークロードにおけるKPTIの性能リグレッション {#case-1-kpti-performance-regression-on-database-workloads}

### 以前の状態 {#before-state}

2018年1月にKPTI（Kernel Page Table Isolation）がLinux 4.15で出荷されたとき、それは急いで投入しなければならない緊急のMeltdown修正だった。緩和策は正しく効果的だったが、その性能への影響はリリース前に十分に把握されていなかった。

PostgreSQL、MySQLなどのデータベースサーバーは、非常に高い頻度でシステムコールを発行する。ホットパス上では`read()`、`write()`、`lseek()`、`fsync()`、`futex()`が呼ばれる。OLTPトランザクションを処理するPostgreSQLインスタンスは、コアあたり毎秒数十万回のsyscallを実行することもある。

### リグレッション {#the-regression}

カーネルのアップグレード直後にLinux 4.15上でPostgreSQL 10を動かしていたチームは、TPC-B相当のベンチマークで**15〜30%**のスループット低下を報告した。リグレッションはPostgreSQLのコードにはなく、ワークロードもまったく同じだった。カーネルのアップグレードだけが原因だった。

### 診断 {#diagnosis}

`perf stat`が事情を物語っていた。

```bash
perf stat -e dTLB-load-misses,iTLB-load-misses,cycles \
          -p $(pgrep postgres | head -1) sleep 10

# KPTI前: dTLB-load-misses はロードの約0.1%
# KPTI後: dTLB-load-misses はロードの約8-15%（TLBが荒らされている）
```

スタックトレース付きの`perf record`では、時間が`native_write_cr3()`とエントリ/イグジットのトランポリンコード、つまりCR3切り替えのオーバーヘッドに集中していることが示された。各syscallでは2回のCR3ロード（エントリ時にユーザーPGD → カーネルPGD、イグジット時にカーネルPGD → ユーザーPGD）が必要で、PCIDを持たないCPUではそのたびにTLBがフラッシュされていた。

根本的な問題は次のとおりである。PCIDがなければ、エントリ/イグジット時のCR3ロードのたびに、そのCPUのTLBエントリがすべて破棄される。毎秒200,000回のsyscallを行うPostgreSQLワーカーは、コアあたり毎秒400,000回TLBを埋め直していたことになる。

### 根本原因 {#root-cause}

KPTIはPCID最適化を念頭に置いて設計されていたが、その最適化は`INVPCID`もサポートするCPUでしか使えなかった。初期のKPTI実装は、PCIDを持つが`INVPCID`を欠くCPUでは低速な（TLB全体をフラッシュする）パスにフォールバックしていた。そこには本番で使われていた一部のHaswellサーバーCPUも含まれていた。

PCID+INVPCIDを完全にサポートするCPUであっても、KPTIのPCID最適化（カーネルPCID = ユーザーASID | 0x80、つまりbit 7を立てるデュアルASID方式）は、最初期のKPTIパッチではデフォルトで有効になっておらず、明示的に有効化し、その後自動検出されるようにする必要があった。

### 修正 {#fix}

PCID最適化は改良され、各世代のCPUで検証された。PCID+INVPCIDをサポートするCPUでは、カーネルは次のように動作する。

1. 各プロセスに2つのPCIDを割り当てる。1つはユーザーPGD用（例: PCID 5）、もう1つはカーネルPGD用（例: PCID 5 | 0x80 = 133）
2. 同じプロセスの2つのPGD間で切り替えるときは、`bit 63 = 1`（TLBフラッシュなし）でCR3をロードする。古いPCIDでタグ付けされたTLBエントリは有効なままだからである
3. 必要なとき（例: `munmap`の後）には`INVPCID`を使ってエントリを選択的に無効化する

PCID最適化により、syscallの多いワークロードでのKPTIのオーバーヘッドは約5%まで低下した。

### 現在の状態を確認する {#checking-the-current-state}

```bash
# KPTIは有効か?
cat /sys/devices/system/cpu/vulnerabilities/meltdown
# "Mitigation: PTI"  = KPTI有効
# "Not affected"     = CPUは影響を受けず、KPTI不要

# PCIDは使われているか?（CPUフラグにPCIDがあるか確認）
grep pcid /proc/cpuinfo

# KPTI + PCID最適化 — dmesgを確認
dmesg | grep -i "pcid\|pti\|kpti"

# syscallのオーバーヘッドを直接測定
time (for i in $(seq 100000); do true; done)
```

### 教訓 {#what-it-taught-us}

正しさを優先したセキュリティ修正は、特定のワークロードで深刻な性能低下を招くことがある。PCID最適化は事前に知られていたが、2018年1月のエンバーゴ解除までに、本番負荷のもとで全世代のCPUにわたって検証することはできなかった。その後のカーネルリリース（4.15.x、4.16）でPCIDパスが改良された。このインシデントをきっかけに、セキュリティ緩和策のベンチマーク手法が改善され、CPUモデル固有のフォールバックにより注意が払われるようになった。

---

## 事例2: マルチソケットシステムにおけるTSCのドリフト {#case-2-tsc-drift-on-multi-socket-systems}

### 以前の状態 {#before-state_1}

**TSC**（Time Stamp Counter）は、一定のレートでインクリメントされるx86のレジスタである（最近のCPUでは、CPUID leaf 0x80000007で示される「invariant TSC」）。タイムスタンプを取得する最も安価な方法であり、カーネル呼び出しのオーバーヘッドなしに`RDTSC`命令1つで済む。

Linuxカーネルは、TSCが信頼できると判断されたシステムでは、TSCを主要なclocksource（`clocksource=tsc`）として使う。vDSOの`clock_gettime(CLOCK_REALTIME)`は、seqlockで保護されたTSCベースのvvarデータを読み取る。すべてユーザー空間で完結し、syscallは発生しない。

### 問題 {#the-problem}

デュアルソケット（およびそれ以上）のNUMAサーバーで、運用者は`gettimeofday()`が**逆行する時刻**を返すことに気づいた。タイムスタンプを順にログ出力するアプリケーションが、後のイベントを前のイベントより早いタイムスタンプで記録することがあった。順序付けにウォールクロック時刻を使うデータベースのレプリケーションシステムは、順序保証を守れなくなった。

```
T1 (thread on socket 0): timestamp = 1234567890.100
Thread migrates to socket 1
T2 (thread on socket 1): timestamp = 1234567889.998  ← time went backwards!
```

これにより、ログアナライザでのアサーション失敗、レプリケーションの順序バグ、謎の「time skew」アラートが発生した。

### 診断 {#diagnosis_1}

異なるソケット上のTSCカウンタは、電源投入時に同期していることが保証されていない。BIOSファームウェアは同期を試みるが完全ではなく、プラットフォームによっては起動時のずれが数百マイクロ秒以上になることもある。さらに古いCPUでは、ソケットごとに異なるクロックドメインがわずかに異なるレートで動くため、TSCの値が時間とともに乖離していくことがある（これはNehalem以降の「invariant TSC」機能で緩和されているが、起動時の同期の問題は残る）。

カーネルは、`tsc_reliable`フラグ（CPUIDとキャリブレーションに基づいて設定される）と、`arch/x86/kernel/tsc.c`における`tsc_clocksource_reliable`のチェック結果によって、TSCの信頼性を判定する。CPU間のTSC同期は、起動時に`check_tsc_sync_source()`と`check_tsc_sync_target()`でテストされる。

```c
/* arch/x86/kernel/tsc_sync.c */
/*
 * 各セカンダリCPUから呼ばれ、自身のTSCをブートCPUのものと比較する。
 * 差がしきい値を超えると、TSCは信頼できないとマークされる。
 */
void check_tsc_sync_target(void)
{
    struct tsc_adjust *ref, *cur;
    cycles_t cur_max_warp;

    /* TSCを読み、ブートCPUのTSCと比較する */
    /* 差 > しきい値 なら: TSCを信頼できないとマークする */
    if (nr_warps) {
        mark_tsc_unstable("TSC ADJUST differs");
    }
}
```

カーネルはTSCのドリフトやスキューを検出すると、clocksourceとしてHPET（High Precision Event Timer）またはACPI PMタイマーにフォールバックする。

### 根本原因 {#root-cause_1}

別々の問題が2つあった。

1. **起動時の同期ずれ**: 各ソケットのTSCカウンタは、電源投入時に異なる値から始まる。たとえ1マイクロ秒の差でも、ソケット0からソケット1へマイグレーションしたプロセス（あるいは共有の`vvar`ページを介して他方のソケットのCPUから読み取るプロセス）には、時刻が逆行したように見える。

2. **TSC調整なしのタスクマイグレーション**: タスクが異なるソケット上のCPU間でマイグレーションすると、新しいCPUのTSCは進んでいるか遅れている可能性がある。カーネルの時刻管理がソケットを意識していなければ、vDSOは誤ったTSCベースを読んでしまう。

### 修正と緩和策 {#fix-and-mitigations}

**カーネル側の検出とフォールバック**: `check_tsc_sync_source()`がCPU間で大きなTSCスキューを検出すると、`mark_tsc_unstable()`を呼び出し、clocksourceを`tsc`から`hpet`または`acpi_pm`に切り替える。これは`dmesg`で確認できる。

```
Detected 1 TSC warp(s) in TSC synchronization.
Marking TSC unstable due to: TSC ADJUST differs.
Switched to clocksource hpet
```

**TSC_ADJUST MSR**: 新しいIntel CPU（Skylake以降）には`TSC_ADJUST` MSR（0x3B）があり、ファームウェアとOSがCPUごとのオフセットを適用してTSC値をそろえられる。カーネルは`tsc_sync.c`で`TSC_ADJUST`の値を読み取り、検証する。

**`rdtsc_ordered()`**: カーネル自身のRDTSCの使用では、メモリフェンスを追加する`rdtsc_ordered()`を使う。

```c
/* arch/x86/include/asm/msr.h */
static __always_inline u64 rdtsc_ordered(void)
{
    /*
     * RDTSC命令はメモリアクセスに対して順序付けされない。
     * 順序を保証するためにシリアライズ命令（LFENCE）を追加する。
     */
    WARN_ON_ONCE(preemptible());
    return rdtsc();  /* LFENCE_RDTSC機能を持つx86-64の場合 */
}
```

**clocksourceの選択**: マルチソケットサーバーでTSCの信頼性に懸念がある運用者は、安定したclocksourceを強制できる。

```bash
# 現在のclocksourceを確認
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
# tsc

# 利用可能なclocksourceを確認
cat /sys/devices/system/clocksource/clocksource0/available_clocksource
# tsc hpet acpi_pm

# HPETを強制（acpi_pmよりオーバーヘッドが小さく、影響を受けるHWではtscより信頼性が高い）
echo hpet > /sys/devices/system/clocksource/clocksource0/current_clocksource

# あるいはブートパラメータで設定（恒久的）
# clocksource=hpet
```

### 教訓 {#what-it-taught-us_1}

マルチソケットシステムでTSCが信頼できないことはよく知られているが、その障害モード（時刻の逆行）は、ウォールクロック時刻が単調増加すると思い込んでいる開発者を驚かせる。LinuxのTSC同期の仕組み（`tsc_sync.c`）とVDSOのseqlockで保護された時刻データはこれに対処するよう設計されているが、HPETへのフォールバックには実際の性能上の影響がある。HPETはリング遷移を必要とするため、`clock_gettime()`はもはやvDSOの速さでは動かない。教訓は、`dmesg`と現在のclocksourceを確認せずに、TSCがソケット間で同期していると決して仮定しないことである。

---

## 事例3: IntelのSYSRET正規アドレスバグ（CVE-2012-0217） {#case-3-intel-sysret-canonical-address-bug-cve-2012-0217}

### 以前の状態 {#before-state_2}

`SYSRET`（具体的には64ビットの`SYSRETQ`で、`SYSCALL`に対応する高速な復帰命令）は、`RCX`から`RIP`を、`R11`から`RFLAGS`を復元し、コードセグメントをリング3に設定することでユーザー空間に戻る。カーネルが`SYSRET`を実行する時点では、ユーザーの`RSP`は*すでに*復元され、`swapgs`でユーザーの`GS.base`も復元されている。つまりCPUはユーザー空間まであと命令1つという状態にありながら、形式上はまだCPL 0にいる。

重要な微妙さはここにある。`RCX`（戻り先の`RIP`）が**非正規アドレス**、つまりビット63:48がビット47の符号拡張になっていないアドレスだったらどうなるか。プロセッサは`#GP`を発生させなければならないが、それを*いつ*、*どの状態で*行うかはベンダーによって異なり、その違いこそがこのバグのすべてである。

### 脆弱性 {#the-vulnerability}

**Intel**のプロセッサでは、`SYSRET`は新しい`RIP`が正規かどうかをチェックし、不正なら**リング0（CPL 0）のまま**`#GP`を発生させる。しかしそれは、`RSP`がユーザーのスタックを指し、`GS.base`がユーザーのものに入れ替えられた*後*である。そのため`#GP`ハンドラは**ユーザーが制御するスタック上でカーネル権限で**実行を始め、`GS.base`がユーザーのものになっているので、最初の`gs:`相対アクセスでフォルトする。非正規アドレスへの復帰を仕組み、ユーザースタックとGSを整えた攻撃者は、フォルトハンドラを自分のコードへ誘導できる。非特権からリング0への、きれいな権限昇格である。

**AMD**のプロセッサでは、CPUがその悪用可能な状態に達する前に`#GP`が配送されるため、AMDのハードウェアは**影響を受けない**。この非対称性こそが、バグが何年も隠れていた理由である。`SYSRET`はAMD（AMD64仕様の策定者）が生み出したもので、カーネルはAMDのセマンティクスに基づいて書かれており、Intelの実装はひっそりと異なっていた。

### 発見と影響 {#discovery-and-impact}

これを良い事件簿にしているひねりはここにある。**Linuxはこのまったく同じバグを2006年にすでに修正していた。** Linuxは初期のIntel EM64T CPUで非正規`SYSRET`の穴に遭遇し、**2.6.16.5**（CVE-2006-0744）でそれを塞いでいた。ユーザースタック上で誤ったGSのままフォルトするという問題で、ほぼ同じ言葉で説明されている。6年後、Rafal Wojtczukは、Linuxの修正を取り入れて*いなかった*すべてのOSがまだ脆弱であることを発見し、CVE-2012-0217（2012年6月）として開示した。`SYSRET`は微妙すぎて各カーネルが独立に間違えていたため、幅広いシステムが影響を受けた。いずれも**Intel CPU上で動作する**ものである。

- Intelホスト上のXen PVゲスト（Xenの`SYSRET`パス）— ゲストからハイパーバイザへの脱出
- FreeBSD、NetBSD、Oracle Solaris / illumos、Microsoft Windows（7 / Server 2008 R2）
- **Linuxは2012年には影響を*受けなかった*** — 2006年から修正済みだった
- AMDのハードウェアは一貫して影響を受けなかった

脆弱なシステム上の非特権ユーザープロセス（またはPVゲスト）は、カーネル（リング0）での完全な実行権を得ることができた。

### 根本原因 {#root-cause_2}

素朴な`SYSRET`の復帰パスは、`RCX`内の戻りアドレスが正規かどうかを先にチェックせずに命令を発行し、正規でなければCPUが安全にフォルトしてくれると信頼している。これはAMDでは正しいがIntelではそうではない。修正は、`SYSRET`の*前に*`RCX`が正規かどうかをチェックし、非正規の場合は（権限の変更を安全に扱う）`IRET`にフォールバックすることである。

### 修正 {#fix_1}

Linuxの修正（最終的に他の全員が取り入れたもの）は、復帰前に`RCX`をチェックする。現代の`arch/x86/entry/entry_64.S`での形を以下に示す（2006年のオリジナルは、シンボル名が異なるだけで考え方は同じである）。

```c
/* arch/x86/entry/entry_64.S（修正後、簡略化） */
/*
 * 戻りアドレス（RCX）が非正規なら、SYSRETは使えない。
 * 非正規なRCXでSYSRETすると、ユーザーRSPのままリング0で#GPになる。
 * 代わりに、権限レベルの変更を安全に扱うIRETを使う。
 */
testq   $0xffff000000000000, %rcx
jnz     swapgs_restore_regs_and_return_to_usermode  /* IRETパスを使う */

/* 通常のケース: 正規アドレスなので安全にSYSRETできる */
USERGS_SYSRET64
```

IRETのフォールバックパス（`swapgs_restore_regs_and_return_to_usermode`）はすべてのレジスタを復元して`IRET`命令を使う。`IRET`は非正規の戻りアドレスに対し、正しい権限レベル（リング3、正しいスタック）で`#GP`を発生させる。

Xenも自身の`SYSRET`エミュレーションパスに同じ修正を実装した。

### 修正を観察する {#observing-the-fix}

```bash
# 修正はSYSRETパスにあり、次の場所で確認できる:
grep -n "SYSRET\|non_canonical\|swapgs_restore" \
    arch/x86/entry/entry_64.S

# 影響を受けるシステムでは、非特権ユーザーは次のようにしてバグを再現できた:
# 1. mmapでsyscallを呼び、非正規アドレスにマッピングを作成する
# 2. その非正規アドレスをスタック上の戻り先として使う（sigreturnフレームの
#    操作、またはスレッドローカルなRIPの直接操作による）
# 修正済みのカーネルはSYSRETの前にRCXをチェックする。
```

### 教訓 {#what-it-taught-us_2}

`SYSCALL`/`SYSRET`のペアにはベンダー固有のエッジケースがある。非正規の`RIP`に対する`SYSRET`の挙動は、AMDでは安全だがIntelでは悪用可能である。カーネルはAMDのセマンティクスに基づいて書かれており（AMD64を定義したのはAMDである）、Intelの実装は異なっていたからだ。注目すべきは、これが**一度学ばれ、そして忘れられた**ことである。Linuxは2006年にこの穴を塞いだ（CVE-2006-0744）にもかかわらず、同じ罠が6年後にXen、BSD系、Solaris、WindowsをCVE-2012-0217として捕らえた。権限遷移命令は*各*ベンダーのマニュアルに照らして検証しなければならず、あるカーネルでの修正は他のカーネルには伝播しない。誰もが鋭い角を独立に学び直すことになる。修正そのものは`SYSRET`の前の正規アドレスチェック1つにすぎない。高くついたのは、それが必要だと気づくことだった。

---

## 事例4: Spectre v2のretpolineがBPF JITの間接呼び出しを破った件 {#case-4-spectre-v2-retpoline-breaking-bpf-jit-indirect-calls}

### 以前の状態 {#before-state_3}

BPF JITコンパイラ（`arch/x86/net/bpf_jit_comp.c`）は、BPFバイトコードを実行時にネイティブのx86-64機械語に変換する。BPFヘルパー関数の呼び出しは間接呼び出しとしてコンパイルされる。つまりJITは関数ポインタを介して呼び出すコードを出力する。

Spectre v2の緩和策以前は、BPFヘルパー呼び出しはおおよそ次のようにコンパイルされていた。

```asm
mov     rax, [helper_ptr]
call    *rax              /* 直接の間接呼び出し */
```

### 問題 {#the-problem_1}

retpolineを有効にするためにカーネルを`-mindirect-branch=thunk-extern`付きでコンパイルすると、コンパイラはCコード中のすべての間接`call *rax`命令をretpolineサンク呼び出しに置き換える。しかし**BPF JITは機械語を直接出力する**ため、コンパイラを経由しない。JITは、緩和策によってretpolineが必要になった後も、生の`call *rax`命令を出力し続けていた。

これは、カーネルコンテキストで実行されるBPFプログラムが、**retpolineで保護されていない**JITコンパイル済みの間接呼び出しを行っていたことを意味し、潜在的なSpectre v2ガジェットとして残っていた。

さらに、一部のBPFプログラムは間接ジャンプであるテールコール（`BPF_TAIL_CALL`）を使う。これらにもretpolineによる対処が必要だった。

### 診断 {#diagnosis_2}

この問題はSpectre後のコード監査の中で発見された。JITの出力（`/proc/sys/net/core/bpf_jit_dump`やBPF逆アセンブラから取得できる）をレビューしていたセキュリティ研究者が、カーネルがretpoline付きでコンパイルされているにもかかわらず、JIT出力に`call *rax`が現れることに気づいた。

### 修正 {#fix_2}

BPF JITは、すべての間接呼び出しとテールコールに対してretpolineのシーケンスを直接出力するよう更新された。JITは生のバイトを出力するため、retpolineのバイト列を手作業で出力する必要があった。

```c
/* arch/x86/net/bpf_jit_comp.c（修正後） */

/* 'reg'を介したretpoline安全な間接呼び出しを出力する */
static void emit_indirect_call(u8 **pprog, int reg)
{
    u8 *prog = *pprog;

    if (cpu_feature_enabled(X86_FEATURE_RETPOLINE_LFENCE)) {
        /* AMDのretpoline変種: LFENCE; JMP *reg */
        EMIT3(0x0F, 0xAE, 0xE8);   /* LFENCE（3バイト: 0F AE E8） */
        EMIT2(0xFF, 0xE0 + reg); /* JMP *reg（オペコード FF /4、例: FF E0 = JMP rax） */
    } else {
        /* Intelのretpoline: サンクを呼び出す */
        /* 出力: call __x86_indirect_thunk_rax（または該当するレジスタ） */
        emit_call(&prog, __x86_indirect_thunk_array[reg], prog);
    }
    *pprog = prog;
}
```

テールコール（間接ジャンプ）についても、同様のretpolineジャンプシーケンスが出力された。

この修正は、カーネルのヘルパー関数を呼び出す`BPF_CALL`命令に対するJITと、BPFプログラム配列を介した間接ジャンプを使うテールコールの仕組みにも影響した。

### 性能とセキュリティのトレードオフ {#performance-vs-security-tradeoff}

retpolineはすべてのBPFヘルパー呼び出しにオーバーヘッドを加える。通常は数サイクル程度の追加である。ヘルパー呼び出しを多数行うBPFプログラム（例: XDPプログラムでのマップ検索）では、これは測定可能な差になる。このトレードオフが受け入れられたのは、BPFプログラムはカーネルコンテキストで実行され、JIT出力中のSpectre v2ガジェットはカーネルのCコード中のものとまったく同じくらい危険だからである。

eIBRS（ハードウェアによるSpectre v2緩和策、Ice Lake以降で利用可能）を持つCPUでは、eIBRSがハードウェアレベルの保護を常時有効にするため、カーネルはより高速な呼び出し規約を使える。

```bash
# eIBRSが使われているか確認（BPF JITの挙動に影響する）
cat /sys/devices/system/cpu/vulnerabilities/spectre_v2
# "Mitigation: Enhanced / Automatic IBRS" = eIBRS、BPF呼び出しが高速
# "Mitigation: Retpoline" = ソフトウェアretpoline、すべての間接呼び出しがサンクになる
```

### 教訓 {#what-it-taught-us_3}

JITコンパイラは緩和策の話の中で特別なケースである。コンパイラベースの緩和策（`-mindirect-branch=thunk-extern`）はGCC/ClangでコンパイルされたCコードにしか効かず、JITコンパイラやインタプリタが実行時に出力する機械語や、手書きのアセンブリには何の効果もない。カーネル内のすべてのJIT（BPF、KVMの命令エミュレータなど）を個別に監査し、更新しなければならなかった。これが、JIT出力を監査するためのより良いツールや、JITコードからretpolineシーケンスを出力するためのより良い抽象化の開発を促した。

---

## 事例5: 初期のHaswell CPUでINVPCIDが使えなかった件 {#case-5-invpcid-not-available-on-early-haswell-cpus}

### 以前の状態 {#before-state_4}

PCID（Process-Context Identifiers）はIntel Sandy Bridge（2011年）で導入され、TLBがプロセス識別子でタグ付けされたエントリをキャッシュできるようにして、コンテキストスイッチ時のTLB全体のフラッシュを回避する。KPTIのPCID最適化（Linux 4.15）は、Meltdown緩和に必要なデュアルPGDのCR3切り替えのオーバーヘッドを減らすためにPCIDを使う。

`INVPCID`命令（Invalidate Process-Context Identifier）は、PCID単位での選択的なTLB無効化を可能にする。KPTI+PCID最適化を完全に行うにはこれが必要である。理由は次のとおり。

- INVPCIDがなければ、特定のPCIDをフラッシュするにはno-flushビットをクリアしてCR3をロードする必要があり、これはより広範な操作になる
- INVPCIDがあれば、必要なエントリだけを正確に無効化できる

### 問題 {#the-problem_2}

初期のHaswell CPU（2013年リリース）はPCIDをサポートする（`/proc/cpuinfo`に`pcid`ビットが現れる）が、**INVPCIDをサポートしない**（`invpcid`がない）。これは既知のエラッタ/ステッピングの問題で、INVPCIDは後のHaswellステッピングでCPUIDの機能ビットに追加された。

KPTIパッチがLinux 4.15で出荷されたとき、初期のPCID最適化パスは`X86_FEATURE_PCID`だけをチェックしてINVPCIDを使おうとした。初期のHaswellシステムでは、カーネルが最初に`INVPCID`を実行しようとした時点で**未定義命令（#UD）フォルト**が発生し、起動時にクラッシュした。

さらに、一部のVMハイパーバイザ（特に古いKVMやVMwareのバージョン）はCPUIDでPCIDを公開しながらINVPCIDを正しく仮想化しておらず、VM内でも同じ起動時クラッシュが起きた。

### 診断 {#diagnosis_3}

起動時のクラッシュでは次のようなメッセージが出力された。

```
kernel BUG at arch/x86/mm/tlb.c:!
invalid opcode: 0000 [#1] SMP PTI
CPU: 0 PID: 0 Comm: swapper/0 Not tainted 4.15.0
RIP: 0010:invpcid_flush_one
```

フォルトした命令は`INVPCID`命令そのもの（オペコード`66 0F 38 82 /r`）だった。

### 根本原因 {#root-cause_3}

KPTIのPCIDコードは`X86_FEATURE_PCID`をチェックしていたが、`X86_FEATURE_INVPCID`は別途チェックしていなかった。修正には、両方の機能をチェックし、INVPCIDが使えない場合のフォールバックパスを実装することが必要だった。

### 修正 {#fix_3}

カーネルは、両方のCPUIDビットを独立にチェックするよう更新された。

```c
/* arch/x86/include/asm/tlbflush.h（修正後） */

/*
 * PCIDとINVPCIDの両方がサポートされている場合にのみPCIDを使う。
 * PCIDがあってもINVPCIDがなければ、KPTIにとってPCIDは利点をもたらさない
 * （INVPCIDなら選択的に処理できるTLB無効化のために、CR3の完全な
 * 再ロードが必要になってしまう）。
 */
static inline bool cpu_has_pcid(void)
{
    return boot_cpu_has(X86_FEATURE_PCID) &&
           boot_cpu_has(X86_FEATURE_INVPCID);
}
```

PCIDはあるがINVPCIDがない場合、カーネルはCR3の再ロード（PCIDをクリアし、必要に応じてTLB全体をフラッシュする）にフォールバックする。これはPCID+INVPCIDの高速パスより遅いが、正しい。

これとは別のフォールバックが、ハイパーバイザがこの命令を公開しない仮想化環境における初期のINVPCIDエミュレーションを扱う。

```bash
# PCIDとINVPCIDを別々に確認
grep -E '\bpcid\b|\binvpcid\b' /proc/cpuinfo

# pcidがあってinvpcidがない場合: PCID最適化は無効
# 両方ある場合: KPTIのPCID最適化が有効

# dmesgで確認
dmesg | grep -i "pcid\|invpcid"

# 影響を受けるシステム（invpcidなし）では、KPTIのオーバーヘッドが大きい:
# 各syscallがCR3切り替え時にTLB全体をフラッシュする
perf stat -e dTLB-load-misses,iTLB-load-misses -p $$ sleep 5
```

### 教訓 {#what-it-taught-us_4}

CPUIDの機能ビットが完全で、一貫しており、互いに独立していると仮定することはできない。CPUはINVPCIDを欠いたままPCIDのサポートを報告することがある。この構成は単独では有用だが（PCIDは通常のコンテキストスイッチでは機能する）、カーネルのKPTIパスは両方がそろわなければ正しく使えなかった。修正では、PCIDとINVPCIDを独立した機能として扱い、すべての組み合わせに対して正しいフォールバックを実装する必要があった。

この事例は、カーネルの機能検出とalternativeパッチングの仕組みがなぜ存在するのかを示す好例でもある。`X86_FEATURE_*`フラグと`cpu_has()`系の関数は、機能チェックのための唯一の権威ある情報源を提供する。これにより、ある機能が必要とされるすべての箇所を監査し、欠けているチェックを一貫して追加できるようになる。

---

## 参考文献 {#further-reading}

### 出典 {#sources}

- [CVE-2012-0217 (NVD)](https://nvd.nist.gov/vuln/detail/CVE-2012-0217) — SYSRETの非正規アドレスによる権限昇格（脆弱な挙動はIntelのSYSRETのもので、AMD CPUは影響を受けない）
- [CVE-2006-0744 (NVD)](https://nvd.nist.gov/vuln/detail/CVE-2006-0744) — Linuxにおける同じバグ。6年前にカーネル2.6.16.5で修正された
- [fail0verflow: Intel's SYSRET kernel privilege escalation](https://fail0verflow.com/blog/2012/cve-2012-0217-intel-sysret-freebsd/) — リング0/ユーザースタックでのフォルトとエクスプロイトについての定番の技術解説
- [arch/x86/kernel/tsc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/tsc.c) — TSCのキャリブレーションとウォッチドッグ。事例2の舞台
- [Spectre Side Channels](https://docs.kernel.org/admin-guide/hw-vuln/spectre.html) — retpolineと間接呼び出しとの相互作用（事例4）

### 関連ページ {#related-pages}

- [x86-64のシステムコールエントリ](syscall-entry.md) — 事例3で悪用されたSYSRETパス
- [SpectreとMeltdown](spectre-meltdown.md) — KPTIとretpolineの事例の背景
- [x86のCPU機能](cpu-features.md) — 機能検出、INVPCIDのフォールバック（事例5）
