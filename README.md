# claude-md-templates

Claude Code で使う `CLAUDE.md` のテンプレート集です。

Agent Teams 対応のチーム体制・開発フローを含む、実践的なテンプレートを提供します。

## 使い方

### Claude Code で自動生成する（推奨）

1. このリポジトリをクローンする

```bash
git clone https://github.com/turntuptechnologies-ai/claude-md-templates.git
cd claude-md-templates
```

2. Claude Code を起動して、対象プロジェクト用の CLAUDE.md を生成してもらう

```bash
claude
```

```
> /home/noto/workspace/my-project 用の CLAUDE.md を作って。
> TypeScript + React + Express のWebアプリで、PostgreSQL を使います。
```

Claude Code がテンプレートを読み、プロジェクトに合った CLAUDE.md を自動生成します。
不足する情報があれば質問してくれます。

### 手動でテンプレートをコピーする

1. `templates/base.md` を対象プロジェクトのルートに `CLAUDE.md` としてコピー
2. Agent Teams を使う場合は `templates/agent-teams.md` の内容を追記
3. `{{ ... }}` の記入欄をプロジェクトに合わせて記入

## ファイル構成

```
claude-md-templates/
├── CLAUDE.md                    # Claude Code への指示書（自動生成の仕組み）
├── README.md                    # このファイル
├── templates/
│   ├── base.md                  # ベーステンプレート（全プロジェクト共通）
│   └── agent-teams.md           # Agent Teams 用テンプレート
└── examples/
    ├── typescript-web-app.md    # 記入済みの例（TypeScript + React + Express）
    └── rust-cli-app.md          # 記入済みの例（Rust CLI）
```

## テンプレートの内容

### base.md

| セクション | 内容 |
|---|---|
| プロジェクト概要 | 名前と説明 |
| プロジェクトフェーズ | INCUBATION → HATCHING → EMERGING → CRUISING の4段階 |
| 言語・ツール | 技術スタック・ビルドコマンド |
| コーディング規約 | 言語固有のルール |
| 開発フロー | Issue 駆動・worktree・squash merge |
| チーム解散前チェック | 設計乖離・テスト不足・ドキュメント不足 |
| Dependabot 対応 | 緊急度別の対応方針（オプション） |
| ライセンスルール | 依存ライブラリのライセンス方針 |
| 環境ルール | 実行環境の制約 |
| 言語ルール | 自然言語の記述言語 |

### agent-teams.md

| セクション | 内容 |
|---|---|
| チーム体制 | 設計・レビュー・実装・テスト・ドキュメントの5チーム |
| 作業方式 | TeamCreate + tmux 並行作業 |
| 基本ルール | 進行順序・依存関係・記録方針 |
