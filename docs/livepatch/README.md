# ライブパッチとkexec {#live-patching-and-kexec}

> 稼働中のカーネルを再起動せずに更新する

## このセクションのページ {#pages-in-this-section}

| ページ | 扱う内容 |
|------|----------------|
| [カーネルライブパッチ](klp.md) | KLP、struct klp_patch、ftraceによるリダイレクト、一貫性モデル |
| [KLP一貫性モデル](klp-consistency.md) | タスクごとのパッチ状態、スタック検査、遷移workqueue、強制遷移 |
| [累積パッチとアトミックリプレース](klp-cumulative.md) | パッチの積み重ね、.replace=true、struct klp_ops、パッチの無効化と削除 |
| [KLP State: 独自の一貫性チェック](klp-state.md) | klp_state API、遷移コールバック、pre/postパッチフック、累積パッチにおける状態の継承 |
| [kGraftとkpatchの歴史](kgraft-vs-kpatch-history.md) | SUSEのkGraftとRed Hatのkpatchがどのようにアップストリームのlivepatchに収束したか、そして一貫性モデルのマージになぜ何年もかかったか |
| [kexec](kexec.md) | kexec_load、machine_kexec、kdumpとの統合、高速リブート |
| [事件簿](war-stories.md) | 遷移の停滞、shadow変数のリーク、compatシステムコールの見落とし、`old_sympos`の曖昧さ、`.replace`の付け忘れ |

## クイックリファレンス {#quick-reference}

```bash
# ライブパッチがサポートされているか確認する
grep CONFIG_LIVEPATCH /boot/config-$(uname -r)
# CONFIG_LIVEPATCH=y

# 有効なライブパッチを一覧表示する
cat /sys/kernel/livepatch/*/enabled

# ライブパッチを適用する（カーネルモジュール）
insmod mypatch.ko
cat /sys/kernel/livepatch/mypatch/enabled
# 1 = 有効（完全に一貫した状態かは transition=0 で確認する）

# ライブパッチを無効化する
echo 0 > /sys/kernel/livepatch/mypatch/enabled

# kexec: 新しいカーネルをロードする
kexec -l /boot/vmlinuz --initrd=/boot/initrd.img --reuse-cmdline

# ロードしたカーネルを実行する（即時、POSTなし）
kexec -e
```
