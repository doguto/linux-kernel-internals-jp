# Linux Kernel Internals 日本語版

[kernel-internals.org](https://kernel-internals.org/)（[laveeshb/linux-kernel-internals](https://github.com/laveeshb/linux-kernel-internals)）の非公式日本語訳。

Linuxカーネルの各サブシステムについて、*何を*しているかだけでなく、*なぜ*そう設計されているのか（トレードオフ、採用されなかった代替案、その背後にあるLKMLでの議論）を解説するドキュメント群である。

**[日本語版サイトを見る](https://doguto.github.io/linux-kernel-internals-jp/)**

## 翻訳について

- 原文: [laveeshb/linux-kernel-internals@6ee74a0](https://github.com/laveeshb/linux-kernel-internals/commit/6ee74a0b2e5f49c75d107ac4935369f73a004546) 時点の内容を翻訳している。
- 訳語・表記の方針は [TRANSLATION_GUIDE.md](TRANSLATION_GUIDE.md) を参照。
- 見出しのアンカー（`{#slug}`）は原文の英語のまま固定しており、原文と同じ `#fragment` でリンクできる。
- 誤訳・改善案は Issue / Pull Request で歓迎する。内容そのものの誤りは原文リポジトリへ報告してほしい。

## ローカルでのビルド

```bash
pip install -r requirements.txt
mkdocs serve
```

## ライセンス

原文と同じく以下のライセンスで提供する。

- ドキュメント: [CC BY-SA 4.0](LICENSE-CC-BY-SA-4.0)（原著作者: laveeshb および linux-kernel-internals のコントリビュータ。本リポジトリは翻訳による改変を含む）
- コード片: [GPL-2.0](LICENSE-GPL-2.0)
- Tux ロゴ: Larry Ewing（原画）、Simon Budig（SVG）
