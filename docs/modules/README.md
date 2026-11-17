# カーネルモジュール {#kernel-modules}

> カーネルを拡張する、動的にロード可能なコード

## カーネルモジュールとは {#what-kernel-modules-are}

カーネルモジュールとは、実行時にカーネルへロードされるELF共有オブジェクトである。モジュールは次の性質を持つ。

- カーネルのアドレス空間を共有し、ring 0（完全な特権）で動作する
- 他のモジュールやコアカーネルのシンボルをエクスポートしたり利用したりできる
- `insmod`/`modprobe` でロードされ、`rmmod` で削除される
- ロード時に呼ばれるinit関数と、アンロード時に呼ばれるexit関数を持つ

ほとんどのデバイスドライバ、ファイルシステム、ネットワークプロトコルはモジュールとして提供される。

## このセクションのページ {#pages-in-this-section}

| ページ | 扱う内容 |
|------|----------------|
| [モジュールの作成とロード](module-basics.md) | module_init/exit、ライフサイクル、/proc/modules、デバッグ |
| [パラメータ、シンボル、Kconfig](module-params.md) | module_param、EXPORT_SYMBOL、Kconfigとの統合 |
| [モジュール署名](module-signing.md) | CONFIG_MODULE_SIG、PKCS#7、鍵の登録、検証フロー |
| [Kbuildビルドシステム](kbuild.md) | Makefileの構造、Kconfig、ツリー外ビルド、クロスコンパイル |
| [モジュールロードの内部](module-loading-internals.md) | load_module()、ELF解析、再配置、シンボル解決、バージョニング |
| [Dynamic Debug](dynamic-debug.md) | pr_debug/dev_dbg、__dyndbgセクション、debugfs経由の実行時制御 |
| [事件簿](war-stories.md) | CRC不一致、署名の失敗、taintの連鎖、initのuse-after-free |

## クイックリファレンス {#quick-reference}

```bash
# モジュールをロードする
insmod ./mymodule.ko
modprobe e1000e      # 依存関係も含めてロード

# アンロード
rmmod mymodule
modprobe -r e1000e   # 使われなくなった依存モジュールも含めて削除

# ロード済みモジュールを一覧表示
lsmod

# モジュール情報
modinfo e1000e

# モジュールのメッセージをカーネルログで確認
dmesg | tail -20
```
