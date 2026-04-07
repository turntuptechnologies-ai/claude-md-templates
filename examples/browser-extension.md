# TaskFlow Notify

TaskFlow Web アプリと連携するブラウザ拡張機能。
タスクの期限切れ・アサイン・ステータス変更をブラウザ通知でリアルタイムに受け取り、ポップアップで未読通知の一覧確認・タスクへの直接遷移ができる。

## プロジェクトフェーズ

現在: **PROTOTYPE**

| フェーズ | 意味 | 状態 |
|---|---|---|
| **PROTOTYPE** | 試作期 | 試作・検証中。コア機能の開発とプロトタイピング |
| **ALPHA / BETA** | 検証期 | 主要機能が揃い、フィードバック収集・改善中 |
| **PREVIEW** | 公開準備期 | 本番環境に近い状態で最終調整 |
| **STABLE** | 安定稼働期 | 正式リリース。安定稼働中 |

## 関連プロジェクト

- [TaskFlow](https://github.com/example/taskflow) — 本拡張が連携するタスク管理 Web アプリ。REST API からタスク・通知データを取得する

## 言語・ツール

- 言語: TypeScript
- 拡張フレームワーク: WXT（Manifest V3 対応のクロスブラウザ拡張フレームワーク）
- UI: React + Tailwind CSS v4
- パッケージマネージャ: pnpm
- ビルド: `pnpm build`（Chrome）/ `pnpm build:firefox`（Firefox）
- 開発サーバー: `pnpm dev`（Chrome）/ `pnpm dev:firefox`（Firefox）
- テスト: `pnpm test`
- リント: `pnpm lint`
- フォーマット: `pnpm format`
- パッケージング: `pnpm zip` / `pnpm zip:firefox`

## 技術要件

### 対応ブラウザ

- Chrome（Manifest V3）
- Firefox（Manifest V3）

### 外部 API

- TaskFlow REST API（タスク・通知データの取得、認証トークンベース）

### 拡張機能の権限（manifest）

- `notifications` — ブラウザ通知の表示
- `alarms` — 定期ポーリング用タイマー（Service Worker のスリープ対策）
- `storage` — 認証トークン・通知キャッシュの永続化
- ホスト権限 — TaskFlow API エンドポイントへのアクセス

権限は `wxt.config.ts` の `manifest` セクションで定義する。

## アーキテクチャ概要

### ブラウザ拡張の 3 層アーキテクチャ

ブラウザ拡張は Content Script・Background（Service Worker）・Popup の 3 層で構成される。各層は `browser.runtime.sendMessage()` でメッセージをやり取りする。

```
                                    ┌──────────────────┐
                                    │  TaskFlow API    │
                                    └────────▲─────────┘
                                             │ REST API
┌──────────────────┐    メッセージ     ┌──────┴───────────┐    メッセージ     ┌──────────────────┐
│  Content Script  │ ──────────────→ │   Background     │ ←────────────── │     Popup        │
│  (TaskFlow上で   │                  │  (Service Worker)│                  │  (React アプリ)   │
│   実行)          │                  │                  │                  │                  │
│ - ログイン検出   │                  │ - API ポーリング  │                  │ - 通知一覧表示    │
│ - トークン取得   │                  │ - 通知キャッシュ   │                  │ - 既読管理        │
│                  │                  │ - ブラウザ通知発行 │                  │ - タスク遷移      │
└──────────────────┘                  └──────────────────┘                  └──────────────────┘
```

### データフロー

1. **認証** — Content Script が TaskFlow Web アプリ上でログイン状態を検出し、認証トークンを Background に送信する
2. **ポーリング** — Background が `browser.alarms` で定期的に TaskFlow API をポーリングし、新規通知を取得する
3. **通知** — 新規通知があれば `browser.notifications` でブラウザ通知を発行し、バッジに未読数を表示する
4. **表示** — Popup が Background から通知一覧を取得し、React で描画する。タスクをクリックすると TaskFlow Web アプリの該当タスクに遷移する

### 主要コンポーネント

- **Content Script**: TaskFlow Web アプリ上でのみ動作し、ログイン検出・認証トークンの受け渡しを行う
- **Background Service Worker**: API ポーリング、通知キャッシュ管理、ブラウザ通知の発行、バッジ更新
- **Popup**: React アプリケーション。未読通知の一覧・既読管理・タスクへの直接遷移

### 設計パターン

- **メッセージ型の判別共用体** — Content Script / Background / Popup 間のメッセージは TypeScript の判別共用体（discriminated union）で型安全に定義する。`satisfies` 演算子で型安全にメッセージを送信する
- **通知フィルター** — 通知種別（期限切れ・アサイン・ステータス変更・コメント）ごとのフィルター関数を純粋関数として実装し、テストしやすくする
- **TTL キャッシュ** — `browser.storage.local` を使い、TTL 付きで API レスポンスをキャッシュする。オフライン時もキャッシュから通知を表示できる
- **Alarms によるポーリング** — Manifest V3 の Service Worker はアイドル時にスリープするため、`browser.alarms` で定期的にウェイクアップしてポーリングする

## ディレクトリ構成

```
entrypoints/                 # WXT エントリポイント（自動的にビルド対象になる）
  background.ts              # Service Worker（API ポーリング・通知管理）
  content.ts                 # Content Script（ログイン検出・トークン取得）
  popup/
    index.html               # Popup の HTML シェル
    main.tsx                 # React エントリポイント
    App.tsx                  # メインコンポーネント（状態マシン）
    style.css                # Tailwind CSS エントリ
    components/
      NotificationList.tsx   # 通知一覧
      NotificationItem.tsx   # 通知カード（展開可能）
      FilterBar.tsx          # 通知種別フィルター
      EmptyState.tsx         # 通知なし状態
      LoginPrompt.tsx        # 未ログイン状態
utils/
  types.ts                   # 共有型定義（メッセージ型・通知型等）
  api-client.ts              # TaskFlow API クライアント（認証・エラーハンドリング付き）
  api-client.test.ts         # テスト
  notification-filter.ts     # 通知フィルターロジック
  notification-filter.test.ts # テスト
  badge.ts                   # バッジ更新ユーティリティ
  badge.test.ts              # テスト
public/
  icon/                      # 拡張アイコン（16/32/48/96/128px）
wxt.config.ts                # WXT 設定（manifest 定義・権限・Vite プラグイン）
vitest.config.ts             # テスト設定
```

### WXT のエントリポイント規則

WXT は `entrypoints/` ディレクトリ内のファイルを自動認識してビルドする:
- `background.ts` → Service Worker
- `content.ts` → Content Script（`defineContentScript()` で対象 URL やタイミングを指定）
- `popup/` ディレクトリ → ブラウザアクションのポップアップ

## コーディング規約

- ESLint + Prettier でコードスタイルを統一する
- 命名規則:
  - コンポーネント・型・インターフェース: `PascalCase`
  - 関数・変数: `camelCase`
  - 定数: `SCREAMING_SNAKE_CASE`（例: `API_BASE_URL`, `POLLING_INTERVAL_MS`, `CACHE_TTL_MS`）
  - ファイル名（コンポーネント）: `PascalCase.tsx`
  - ファイル名（その他）: `kebab-case.ts`
- `any` 型の使用は禁止。やむを得ない場合は `unknown` + 型アサーションを使う
- React コンポーネントは関数コンポーネント + hooks で統一する
- メッセージ型は判別共用体で定義し、`satisfies` 演算子で型安全にメッセージを送信する
- パスエイリアス `@/` をプロジェクトルートにマッピングして使用する（例: `@/utils/types`）
- エラーメッセージ・テスト説明は日本語で記述する
- 外部 API のエラーハンドリングは `.catch(() => デフォルト値)` でグレースフルに処理する（拡張全体がクラッシュしないようにする）

## テスト戦略

### テストの種類と実行方法

| 種類 | 対象 | 実行コマンド |
|------|------|-------------|
| 単体テスト | フィルター・API クライアント・ユーティリティ | `pnpm test` |
| ウォッチモード | 開発中のテスト自動実行 | `pnpm test:watch` |

### テストファイルの配置

- 対象ファイルと同じディレクトリに `.test.ts` サフィックス（例: `api-client.ts` → `api-client.test.ts`）
- テスト環境: Vitest + jsdom（DOM 操作を含むテスト用）
- テストヘルパー関数（`createNotification()`, `createApiResponse()` 等）でテストデータのセットアップを簡潔にする

### テスト方針

- フィルター・バッジ更新ロジックは純粋関数として実装し、`browser` API のモックなしでテストできるようにする
- API クライアントのテストでは `vi.stubGlobal()` で `fetch` をモックする
- ポーリング・キャッシュのテストでは `vi.useFakeTimers()` で時間を制御する
- `it.each` でパラメータ化テストを活用する

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

1. **Issue 作成** — コード・ドキュメント等の変更には必ず Issue を作成する
   - Issue のコメントに調査内容・試行錯誤・判断の経緯を記録する
   - 他の Issue を参照するときは番号だけでなく説明を付けた箇条書きにする（例: `- #5 — 期限切れ通知のフィルター`）
   - 適宜ラベルを付与する
   - `backlog` ラベル: 優先度が低く、通常の開発サイクルでは取り組まない Issue に付与する。「オープンの Issue に取り組んで」等の指示では `backlog` ラベルの Issue は対象外とする
2. **設計** — 設計チームが要件整理・アーキテクチャ設計を行い、Issue に設計内容を記載する
3. **設計レビュー** — レビューチームが設計の妥当性をチェックし、設計チームと相談・調整する
4. **ブランチ作成** — git worktree を使い、Issue に紐づくブランチで作業する
   - ブランチ命名規則: `issue-<番号>/<簡単な説明>` (例: `issue-5/add-deadline-notification`)
5. **実装** — 実装チームが設計に基づきコーディングする
   - 不明点は設計チーム・レビューチームに相談する
6. **PR 作成** — Issue を参照する (`Closes #5` 等)
7. **コミット・プッシュ** — 作業中は適宜 commit・push を行う（main ブランチへの直接 push は禁止）
8. **コードレビュー** — マージ前にレビューチームが PR をレビューする
   - セキュリティ観点のチェックを含める（認証トークンの安全な保存、拡張機能の権限スコープ、XSS 等）
9. **マージ** — squash merge を使用する
10. **ブランチ削除** — マージ後にブランチを自動削除する

## チーム体制

| チーム | メンバー | 役割 |
|--------|----------|------|
| 設計 | designer | 要件整理・アーキテクチャ設計・Issue 作成 |
| レビュー | reviewer | 設計の妥当性チェック・PR のコードレビュー |
| 実装 | developer, developer2 | コーディング・commit・PR 作成 |
| テスト | tester | テストコード作成・品質検証 |
| ドキュメント | documenter | ドキュメント作成・整備 |

## 作業方式

開発作業は必ず Agent Teams（`TeamCreate`）を使い、tmux 画面分割でエージェントが並行作業する形式で進めること。

### 基本ルール

- **Issue ごとにチームを作成する** — 関連する Issue をまとめて 1 チームで対応してもよい
- **チーム構成は以下の順で進行する**:
  1. 設計（designer） — 要件整理・アーキテクチャ設計
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

チームを解散する前に、必ず以下の 3 つの検証を実施すること:

1. **設計との乖離チェック** — ドキュメント（CLAUDE.md, README.md 等）と実装コードに乖離がないか
2. **テストの不足チェック** — 新規・変更コードに対してテストが十分か、カバレッジに穴がないか
3. **ドキュメントの不足チェック** — 新機能・変更がドキュメントに反映されているか

発見された問題は Issue 化して対応してから解散する。

## デプロイ

- Chrome Web Store / Firefox Add-ons (AMO) への公開を想定
- `pnpm zip` / `pnpm zip:firefox` で配布用 ZIP を生成する

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

## 言語ルール

- Issue、コメント、PR の説明、コミットメッセージなど自然言語を書く箇所はすべて日本語で記述する
