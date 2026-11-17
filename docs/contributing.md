# Linux Kernel Internalsへのコントリビュート {#contributing-to-linux-kernel-internals}

> **これをWebサイトで読んでいるなら？** コントリビュート、issueの作成、ディスカッションへの参加は[GitHubリポジトリ](https://github.com/laveeshb/linux-kernel-internals)で行ってほしい。

コントリビュートに関心を持ってくれてありがとう！ このプロジェクトは、Linuxカーネルの内部構造を、わかりやすく興味を引く形で文書化することを目指している。

## プロジェクトの理念 {#project-philosophy}

これは、今日のLinuxカーネルがどうなっているかを文書化するもう1つのリファレンス**ではない**。公式の[カーネルドキュメント](https://docs.kernel.org/)がすでにそれをうまくやっている。

その代わりに、我々は次のことに焦点を当てる:

- 物事が現在の設計になっている**理由**
- サブシステムが時間とともに**どのように**進化してきたか
- **どのような問題**が現在の解決策につながったか
- 検討された**トレードオフ**

執筆するときは、コードが*何*をするかを説明するだけにしないこと — 設計判断の背後にある理由付け、歴史的な文脈、そしてカーネルバージョンを経て物事がどう変わってきたかを読者が理解できるよう手助けしてほしい。こうした判断を記録しているコミット、メーリングリストでの議論、カンファレンスでの講演へリンクすること。

## コントリビュートの方法 {#how-to-contribute}

### 取り組む作業を見つける {#finding-work}

[オープンなissue](https://github.com/laveeshb/linux-kernel-internals/issues)を閲覧し、[ラベル](https://github.com/laveeshb/linux-kernel-internals/labels)で絞り込む:

| ラベルの接頭辞 | 目的 |
|--------------|---------|
| `priority/*` | 重要度（high、medium、low） |
| `area/*` | カーネルサブシステム（mm、schedulerなど） |
| `type/*` | ドキュメントのスタイル（narrative、reference、guide、explainer） |
| `epic/*` | まとめられた作業項目 |

プロジェクトに初めて参加するなら`good first issue`を探してほしい。

### ワークフロー {#workflow}

1. リポジトリを**フォーク**する
2. `main`から**ブランチを作成**する（例: `add-kasan-docs`）
3. 下記のスタイルガイドに従って**内容を書く**
4. `mkdocs serve`で**ローカルでテスト**する
5. 明確な説明を付けて**PRを提出**する
6. レビュアーからの**フィードバックに対応**する

### プルリクエストの要件 {#pull-request-requirements}

- PRはマージ前に**1件の承認**が必要
- PRは単一のトピックに絞る
- 関連するissueへリンクする（例: "Closes #42"）

## 執筆スタイルガイド {#writing-style-guide}

### 語り口とトーン {#voice-and-tone}

- **わかりやすく、かつ正確に** - 正確さを犠牲にせずに、複雑なトピックを明快に説明する
- **文書化するだけでなく、教える** - *何*だけでなく*なぜ*を読者が理解できるよう助ける
- **能動態を使う** - "Pages are allocated by the kernel"ではなく"The kernel allocates pages"
- **簡潔に** - つなぎの言葉や不要な専門用語を避ける

### ドキュメントの構成 {#document-structure}

すべてのドキュメントは、おおむね次の構成に従うべきである:

```markdown
# Title

> One-line description of what this covers

## What is [Topic]?

Brief introduction explaining what this is and why it matters.

## Core Concepts

Main technical content with examples.

## How It Works

Step-by-step explanations, diagrams, code examples.

## Try It Yourself

Commands readers can run to explore the concept.

## History / Evolution (if relevant)

When features were added, why decisions were made.

## Further Reading

Links to kernel docs, LWN articles, relevant commits.
```

### 書式の規約 {#formatting-conventions}

**見出し**

- センテンスケースを使う: "Page Allocator"ではなく"Page allocator"
- H1（`#`）はページタイトルのみに使う
- H2（`##`）は主要なセクションに使う
- H3（`###`）はサブセクションに使う

**コードブロック**
```c
// シンタックスハイライトのために言語ヒントを使う
struct page *page = alloc_pages(GFP_KERNEL, 0);
```

**シェルコマンド**
```bash
# コマンドが何をするかをコメントで示す
cat /proc/meminfo | grep -i huge
```

比較には**表**を使う:
| 機能 | オプションA | オプションB |
|---------|----------|----------|
| 速度   | 速い     | 遅い     |
| メモリ  | 少ない      | 多い     |

ASCIIまたはMermaidによる**図**:
```
+-----------+      +-----------+
|  Process  |----->|  Kernel   |
+-----------+      +-----------+
```

```mermaid
graph LR
    A["ユーザー空間"] --> B["システムコール"]
    B --> C["カーネル"]
```

### 内容のガイドライン {#content-guidelines}

**含めるべきもの:**

- 実際のカーネルの関数名や構造体
- 変更について論じる際の、実際のカーネルコミットへのリンク
- 機能が導入された具体的なカーネルバージョン
- コマンド付きの「自分で試してみよう」セクション
- よくある誤解や落とし穴
- 設計判断の歴史的な文脈

**含めるべきでないもの:**

- 憶測や未検証の情報
- 古いことを明記していない古い情報
- 他の情報源からコピー＆ペーストした内容
- マーケティング的な言葉や誇大な宣伝
- 商業的な情報源（ベンダーのブログ、製品ドキュメント、スポンサー付きコンテンツ）

### 主張の出典 {#sourcing-claims}

あらゆる主張、意見、技術的な記述は、検証可能な出典に裏付けられているべきである:

**受け入れられる出典:**

- カーネルのコミット（`git.kernel.org`のみ — ミラーやフォークは不可）
- LKMLのスレッド（`lore.kernel.org`）
- 公式カーネルドキュメント（`docs.kernel.org`）
- LWN.netの記事
- カンファレンスでの講演（Linux Plumbers、Kernel Recipesなど）
- 学術論文
- メンテナのブログ記事

**受け入れられないもの:**

- 商業ベンダーのドキュメント
- 製品のマーケティング資料
- 出典のないStack Overflowの回答
- 検証されていないAI生成コンテンツ

主張をするときは、その出典へリンクする:
```markdown
The buddy allocator coalesces free blocks to reduce fragmentation
([Knuth, TAOCP Vol 1](https://example.com), first implemented in
[commit abc123](https://git.kernel.org/...)).
```

### カーネルソースへのリンク {#linking-to-kernel-source}

カーネルのコードを参照するとき:

- 閲覧可能なソースとして[git.kernel.org](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/)へリンクする
- 歴史的な変更を論じる場合を除き、stable/最新のカーネルバージョンを使う
- APIを紹介するときは関数シグネチャを含める

例:
```markdown
The [`alloc_pages()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/gfp.h)
function is the primary interface...
```

### コミットの参照 {#referencing-commits}

機能が導入または変更されたときは、そのコミットへリンクする:

```markdown
This was introduced in kernel 5.18
([commit abc1234](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=abc1234):
"mm: add feature X").
```

## 技術的な正確さ {#technical-accuracy}

### 検証 {#verification}

- コマンドは載せる前にテストする
- カーネルバージョンの情報を検証する
- 公式カーネルドキュメントと照合する
- 確信が持てない場合は、その旨を記す: "As of kernel 6.x..."や"This may vary by configuration"

### レビューの流れ {#review-process}

レビュアーは次の点を確認する:

- 技術的な正確さ
- 明快さと読みやすさ
- スタイルガイドへの準拠
- 動作するコード例
- 適切なリンクと参照

## ローカルでの開発 {#local-development}

### セットアップ {#setup}

```bash
# 自分のフォークをクローンする
git clone https://github.com/YOUR_USERNAME/linux-kernel-internals.git
cd linux-kernel-internals

# 仮想環境を作成する
python -m venv .venv
source .venv/bin/activate

# 依存関係をインストールする
pip install mkdocs mkdocs-material

# ローカルサーバーを起動する
mkdocs serve
```

### プレビュー {#preview}

`http://127.0.0.1:8000`を開いて変更をプレビューする。サーバーはファイルの変更時に自動でリロードする。

### ビルド {#build}

```bash
mkdocs build
```

ビルド出力に警告がないか確認する。

## コミットメッセージ {#commit-messages}

明確で説明的なコミットメッセージを書く:

```
Add KASAN documentation

- Explain what KASAN detects
- Document configuration options
- Include example report analysis
```

**すべきこと:**

- 命令形を使う（"Added"ではなく"Add"）
- 1行目は50文字未満に収める
- 本文で何を、なぜ変更したかを説明する

**すべきでないこと:**

- AI/アシスタントによる帰属表記を含める
- "Update docs"のような曖昧なメッセージを使う
- 無関係な変更を1つのコミットに含める

## 質問は？ {#questions}

- 質問には[Discussion](https://github.com/laveeshb/linux-kernel-internals/discussions)を開く
- バグや提案には[Issue](https://github.com/laveeshb/linux-kernel-internals/issues)を登録する

## ライセンス {#license}

コントリビュートすることで、あなたの貢献がプロジェクトと同じデュアルライセンスの下でライセンスされることに同意したものとする:

- ドキュメント: [CC BY-SA 4.0](https://github.com/laveeshb/linux-kernel-internals/blob/main/LICENSE-CC-BY-SA-4.0)
- コード例: [GPL-2.0](https://github.com/laveeshb/linux-kernel-internals/blob/main/LICENSE-GPL-2.0)
