# カーネルデバッグ {#kernel-debugging}

> Linuxカーネルをデバッグするためのツールと手法

## デバッグのアプローチ {#debugging-approaches}

```
Problem type                  Best tool
────────────────────────────────────────────────────────────
Kernel crash (already happened) kdump + crash
Live kernel, single machine     kgdb over serial/netconsole
Memory corruption               KASAN, KFENCE, SLUB debug
Race conditions                 lockdep, KCSAN
Crashes with no debugger        Oops analysis, addr2line
Performance                     perf, ftrace, BPF
Logic bugs in drivers           dynamic debug, pr_debug
Boot failures                   early_printk, earlyprintk=ttyS0
```

## このセクションのページ {#pages-in-this-section}

| ページ | 扱う内容 |
|------|----------------|
| [KGDB](kgdb.md) | シリアルまたはネットワーク経由のGDBによる稼働中カーネルのデバッグ |
| [kdumpとcrash](kdump.md) | カーネルクラッシュダンプの取得と解析 |
| [Oops解析](oops-analysis.md) | oops出力の読み方、addr2line、decode_stacktrace |
| [KCSAN](kcsan.md) | ウォッチポイントとLinux Kernel Memory Modelを用いたデータ競合検出 |
| [syzkaller](syzkaller.md) | 自動システムコールファジング、syzbot、再現プログラムのワークフロー |
| [KASANとKFENCE](kasan-kfence.md) | メモリエラー検出: 境界外アクセス、use-after-free、シャドウメモリ |
| [lockdepの実践](dynamic-debug-lockdep.md) | splatの読み方、誤検知へのアノテーション、競合統計 |
| [事件簿](war-stories.md) | KASAN、lockdep、syzkaller、kdump、KFENCEが捕まえたバグ |

## クイックリファレンス {#quick-reference}

```bash
# 必要に応じてoopsを発生させる（テスト用）
echo c > /proc/sysrq-trigger

# SysRqを有効化する
echo 1 > /proc/sys/kernel/sysrq

# よく使うSysRqキー:
# Alt+SysRq+t: 全タスクをダンプ
# Alt+SysRq+m: メモリ情報
# Alt+SysRq+l: 全CPUのバックトレース
# Alt+SysRq+b: 再起動
# Alt+SysRq+c: クラッシュ（kdumpのテスト用）

# oops時にパニックさせる（kdumpのテストに便利）
echo 1 > /proc/sys/kernel/panic_on_oops
echo 5 > /proc/sys/kernel/panic          # パニックの5秒後に再起動

# kernel.orgのデコード
./scripts/decode_stacktrace.sh vmlinux < oops.txt
```
