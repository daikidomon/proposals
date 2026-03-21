# 提案資料スライド生成ワークフロー

Markdownで作成した提案書をpandocでスライド形式に変換し、顧客への提案資料として利用する。

## 前提

- pandocがインストール済みであること
- 提案書がMarkdownで作成されていること

```bash
# インストール（Ubuntu）
sudo apt install pandoc

# バージョン確認
pandoc --version
```

## 変換コマンド

### PowerPoint（.pptx）

```bash
pandoc proposals/example/proposal.md -o output/proposal.pptx
```

### Google Slides

Google SlidesはMarkdownからの直接変換に対応していないため、一度PowerPointに変換してからGoogle Driveにアップロードする。

```bash
# 1. PowerPointに変換
pandoc proposals/example/proposal.md -o output/proposal.pptx

# 2. Google Driveにアップロードし「Google スライドとして開く」を選択
```

## オプション

```bash
# テンプレート適用（会社のスライドテンプレートを使用）
pandoc proposal.md -o proposal.pptx --reference-doc=templates/template.pptx

# 複数ファイルを結合して1つのスライドに変換
pandoc 01-intro.md 02-analysis.md 03-plan.md -o proposal.pptx
```

## Markdown記述ルール

pandocでスライドに変換する際、見出しレベルがスライドの区切りになる。

| 要素 | 記法 | スライド上の扱い |
|------|------|------------------|
| `#` 見出し1 | セクション区切り | セクションタイトルスライド |
| `##` 見出し2 | スライド区切り | 各スライドのタイトル |
| `---` | 水平線 | タイトルなしのスライド区切り |
| 本文・箇条書き | 通常のMarkdown | スライド本文 |
| 表 | Markdown表記 | スライド内の表 |
| 画像 | `![alt](path)` | スライド内の画像 |

### 記述例

```markdown
# セクション: 現状分析

## 現在の課題

- コストが高い（月額$916）
- セキュリティリスクがある
- キャッシュ機構がない

## コスト比較

| 項目 | 現状 | 移行後 |
|------|------|--------|
| 月額 | $916 | $11 |

---

![アーキテクチャ図](images/architecture.png)
```

## テンプレートの作成

会社共通のスライドテンプレートを用意しておくと、ブランドの統一感を保てる。

```bash
# 1. pandocのデフォルトテンプレートを出力
pandoc -o templates/template.pptx --print-default-data-file reference.pptx

# 2. PowerPointでtemplate.pptxのスライドマスターを編集（ロゴ・配色・フォント等）

# 3. テンプレートを指定して変換
pandoc proposal.md -o proposal.pptx --reference-doc=templates/template.pptx
```

## 基本的なワークフロー

1. Markdownで提案書を作成（`proposals/` 配下）
2. スライド用にMarkdownの見出し構造を調整（必要に応じて）
3. `pandoc` でPowerPointに変換
4. テンプレートを適用して体裁を整える
5. 必要に応じてGoogle Driveにアップロードし、Google Slidesとして共有
