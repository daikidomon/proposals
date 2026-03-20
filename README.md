# proposals

提案書・調査記録を会社（プロジェクト）単位で管理するリポジトリです。

## ディレクトリ構成

```
proposals/
├── docs/                          # 技術ドキュメント・調査資料
│   └── <会社名>/                  # 会社単位で分類
├── proposals/                     # 提案書
│   └── <会社名>/                  # 会社単位で分類
├── <会社名>/                      # 会社ごとのワークスペース
│   ├── proposals/                 # その会社向けの提案書
│   └── <リポジトリ名>/            # サブモジュール（対象リポジトリ）
└── README.md
```

### 現在の構成例

```
proposals/
├── docs/
│   └── rasukichi-system/
├── proposals/
│   └── rasukichi-system/
├── rasukichi/
│   ├── proposals/
│   │   ├── issue-46-translation-migration-investigation.md
│   │   └── proposal-translation-backend-migration.md
│   └── rasukichi-system/          # ← サブモジュール (rasukichisama/rasukichi-system)
└── README.md
```

## 使い方

### リポジトリのクローン

サブモジュールを含めてクローンしてください。

```bash
git clone --recurse-submodules <このリポジトリのURL>
```

既にクローン済みの場合:

```bash
git submodule update --init --recursive
```

### 新しい会社・プロジェクトを追加する

1. 会社名のディレクトリを作成する

```bash
mkdir -p <会社名>/proposals
mkdir -p docs/<会社名>
mkdir -p proposals/<会社名>
```

2. 対象リポジトリをサブモジュールとして追加する（必要な場合）

```bash
git submodule add <リポジトリURL> <会社名>/<リポジトリ名>
```

### 提案書を追加する

`<会社名>/proposals/` に Markdown ファイルとして作成します。

命名規則の例:
- `proposal-<機能名>.md` — 提案書
- `issue-<番号>-<概要>.md` — Issue に紐づく調査記録

## サブモジュールについて

このリポジトリは対象プロジェクトのソースコードをサブモジュールとして参照しています。

- サブモジュールは特定のコミットを指すため、最新に追従するには明示的な更新が必要です
- サブモジュールの更新: `git submodule update --remote`
