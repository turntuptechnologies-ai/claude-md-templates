# TaskFlow

チーム向けタスク管理 Web アプリケーション。
プロジェクトごとにタスクを管理し、メンバー間のアサイン・進捗追跡・通知をリアルタイムで行う。

## プロジェクトフェーズ

現在: **PROTOTYPE**

| フェーズ | 意味 | 状態 |
|---|---|---|
| **PROTOTYPE** | 試作期 | 試作・検証中。コア機能の開発とプロトタイピング |
| **ALPHA / BETA** | 検証期 | 主要機能が揃い、フィードバック収集・改善中 |
| **PREVIEW** | 公開準備期 | 本番環境に近い状態で最終調整 |
| **STABLE** | 安定稼働期 | 正式リリース。安定稼働中 |

## 言語・ツール

- 言語: TypeScript
- フロントエンド: React + Vite
- バックエンド: Express.js (Node.js)
- DB: PostgreSQL + Prisma (ORM)
- リアルタイム通信: WebSocket (Socket.IO)
- パッケージマネージャ: pnpm (workspaces)
- ビルド: `pnpm build`
- テスト: `pnpm test`
- 統合テスト: `pnpm test:integration`
- リント: `pnpm lint`
- フォーマット: `pnpm format`

## アーキテクチャ概要

### データフロー

1. **リクエスト受信** — React フロントエンドから REST API / WebSocket でリクエスト
2. **処理** — Express.js でビジネスロジックを実行、Prisma 経由で PostgreSQL にアクセス
3. **リアルタイム通知** — タスク更新時に Socket.IO で関連ユーザーに即座に通知

### 主要コンポーネント

- **REST API**: タスク・プロジェクト・ユーザーの CRUD エンドポイント
- **WebSocket サーバー**: リアルタイム通知（タスク更新・コメント追加等）
- **認証ミドルウェア**: JWT ベースの認証・認可
- **フロントエンド SPA**: React + React Router によるシングルページアプリケーション

## ディレクトリ構成

```
packages/
  frontend/
    src/
      components/        # React コンポーネント
      hooks/             # カスタム hooks
      pages/             # ページコンポーネント
      services/          # API クライアント
      types/             # フロントエンド固有の型定義
  backend/
    src/
      routes/            # Express ルートハンドラ
      middleware/         # 認証・バリデーション等
      services/          # ビジネスロジック
      prisma/            # Prisma スキーマ・マイグレーション
  shared/
    types/               # フロントエンド・バックエンド共有の型定義
```

## コーディング規約

- ESLint + Prettier でコードスタイルを統一する
- 命名規則:
  - コンポーネント・型・インターフェース: `PascalCase`
  - 関数・変数: `camelCase`
  - 定数: `SCREAMING_SNAKE_CASE`
  - ファイル名（コンポーネント）: `PascalCase.tsx`
  - ファイル名（その他）: `kebab-case.ts`
- `any` 型の使用は禁止。やむを得ない場合は `// eslint-disable-next-line` とコメントで理由を明記する
- React コンポーネントは関数コンポーネント + hooks で統一する
- API レスポンスの型は `shared/types/` に定義し、フロントエンド・バックエンドで共有する
- エラーハンドリング:
  - バックエンド: カスタムエラークラスを定義し、エラーハンドリングミドルウェアで一元処理する
  - フロントエンド: Error Boundary でクラッシュを防止する

## テスト戦略

### テストの種類と実行方法

| 種類 | 対象 | 実行コマンド |
|------|------|-------------|
| 単体テスト | コンポーネント・ユーティリティ・サービス層 | `pnpm test` |
| 統合テスト | API エンドポイント（DB 含む） | `pnpm test:integration` |

### テストファイルの配置

- 単体テスト: 対象ファイルと同じディレクトリに `.test.ts` / `.test.tsx` サフィックス（例: `TaskList.tsx` → `TaskList.test.tsx`）
- 統合テスト: `tests/integration/` ディレクトリに配置

## コミットメッセージ規約

[Conventional Commits](https://www.conventionalcommits.org/) に従う。

```
<type>: <description> (#<issue-number>)
```

### type 一覧

| type | 用途 |
|------|------|
| `feat` | 新機能の追加 |
| `fix` | バグ修正 |
| `docs` | ドキュメントのみの変更 |
| `test` | テストの追加・修正 |
| `refactor` | リファクタリング（機能変更なし） |
| `chore` | ビルド・CI・依存関係等の雑務 |

### ルール

- description は日本語で記述する
- Issue 番号がある場合は末尾に `(#番号)` を付ける
- squash merge 時の PR タイトルもこの規約に従う

## 開発フロー

1. **Issue 作成** — コード・ドキュメント等の変更には必ず Issue を作成し、調査内容・判断の経緯をコメントに記録する
   - `backlog` ラベル: 優先度が低く、通常の開発サイクルでは取り組まない Issue に付与する。「オープンの Issue に取り組んで」等の指示では対象外とする
2. **ブランチ作成** — Issue に紐づくブランチで作業する。命名規則: `issue-<番号>/<簡単な説明>`（例: `issue-15/add-task-filter`）
3. **実装** — 作業中は適宜 commit・push を行う。**main への直接 push は禁止**
4. **PR 作成** — Issue を参照する（`Closes #15` 等）
5. **コードレビュー** — マージ前にレビューチームが PR をレビューする。セキュリティ観点のチェックを含める（認証・認可の抜け漏れ、入力バリデーション、インジェクション対策等）
6. **マージ** — squash merge を使用し、マージ後にブランチを削除する

## チーム体制

| チーム | メンバー | 役割 |
|--------|----------|------|
| 設計 | architect | 要件整理・アーキテクチャ設計・Issue 作成 |
| レビュー | reviewer | 設計の妥当性チェック・PR のコードレビュー |
| 実装 | developer, developer2 | コーディング・commit・PR 作成 |
| テスト | tester | テストコード作成・品質検証 |
| ドキュメント | documenter | ドキュメント作成・整備 |

## 作業方式

開発作業は必ず Agent Teams（`TeamCreate`）を使い、tmux 画面分割でエージェントが並行作業する形式で進めること。

### 基本ルール

- **Issue ごとにチームを作成する** — 関連する Issue をまとめて 1 チームで対応してもよい
- **チーム構成は以下の順で進行する**:
  1. 設計（architect） — 要件整理・アーキテクチャ設計
  2. レビュー（reviewer） — 設計の妥当性チェック
  3. 実装（developer / developer2） — コーディング・commit・PR 作成
  4. テスト（tester） — テストコード作成・品質検証
  5. ドキュメント（documenter） — ドキュメント作成・整備
- **タスクの依存関係を設定する** — 設計完了後にレビュー、レビュー完了後に実装、のように順序を守る
- **設計判断・調査内容は Issue コメントに記録する**
- **tmux 画面分割で並行作業の様子を確認できるようにする**

### チームを使わずに作業してはならない場面

- コード変更を伴う Issue の対応
- 新機能の追加・既存機能の変更
- バグ修正

> **例外**: typo 修正や 1 行の設定変更など、明らかに軽微な変更はチームなしで対応してよい。

## チーム解散前の必須チェック

チームを解散する前に、必ず以下の 3 つを検証すること:

1. **設計との乖離** — ドキュメント（CLAUDE.md, README.md 等）と実装コードにズレがないか
2. **テストの不足** — 新規・変更コードに対してテストが十分か、カバレッジに穴がないか
3. **ドキュメントの不足** — 新機能・変更がドキュメントに反映されているか

- 検証は**実装に関与していないメンバー**（reviewer / tester、または検証用に新規起動したエージェント）が行う。実装した本人の自己チェックで代替しない
- 発見した問題は反証を試みて誤検出を除いてから Issue 化し、対応してから解散する

## デプロイ

- 当面: VPS（Docker Compose）
- 将来: クラウド PaaS への移行を視野に入れる

## サプライチェーンセキュリティ（pnpm）

pnpm のセキュリティ機能を活用し、依存パッケージ経由の攻撃リスクを低減する。

### ロックファイル

- `pnpm-lock.yaml` は必ずリポジトリにコミットする
- CI では `pnpm install --frozen-lockfile` を使い、ロックファイルと一致しない場合はエラーにする

### postinstall スクリプトの制御

- pnpm v10 ではデフォルトで依存パッケージの postinstall スクリプトが無効化されている
- ネイティブバイナリのビルド等が必要なパッケージのみ `package.json` の `pnpm.allowBuilds` でホワイトリストに追加する
- `pnpm.dangerouslyAllowAllBuilds` は使用禁止

### 非標準ソースの依存の遮断

- `package.json` に `pnpm.blockExoticSubdeps: true` を設定し、間接依存が Git リポジトリや tarball URL から引かれることを防止する

### 新規公開パッケージの遅延インストール

- `package.json` に `pnpm.minimumReleaseAge: "4320"` を設定し、公開から 3 日未満のバージョンをインストールしない
- 悪意あるパッケージは公開後すぐに検出されることが多いため、遅延によりリスクを回避する
- 緊急のセキュリティパッチ等で即時更新が必要な場合は一時的に設定を解除して対応する

### 信頼ポリシー

- `package.json` に `pnpm.trustPolicy: "no-downgrade"` を設定し、provenance（出所証明）の信頼レベルが下がったバージョンのインストールを防止する

## ライセンスルール

- GPL 系ライセンスの依存パッケージは使用禁止（商用転用の可能性があるため）
- 許可: MIT, Apache-2.0, BSD-2-Clause, BSD-3-Clause, ISC 等の permissive ライセンス
- 依存追加時はライセンスを必ず確認する

## 環境ルール

- sudo は使用しない
- ツールやランタイムのインストールには mise を使用する
- サービスやミドルウェアが必要な場合は Docker を使用し、`docker-compose.yaml` で管理する

## 言語ルール

- Issue、コメント、PR の説明、コミットメッセージなど自然言語を書く箇所はすべて日本語で記述する
