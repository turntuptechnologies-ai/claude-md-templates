# SiteGuard

閲覧中の Web サイトのセキュリティ状態を自動チェックするブラウザ拡張機能。
HTTPS・セキュリティヘッダー・ドメイン信頼性・CMS 脆弱性などを検出し、ポップアップで結果を表示する。

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

- NVD API 2.0（脆弱性情報の取得、レート制限あり）
- RDAP API（ドメイン登録情報の取得）
- crt.sh（CT ログの検索）
- ssl-checker.io（TLS 証明書情報の取得）

### 拡張機能の権限（manifest）

- `activeTab` — 現在のタブの情報にアクセスする
- `storage` — キャッシュデータの永続化
- `webRequest` — HTTP レスポンスヘッダーのキャプチャ（セキュリティヘッダーチェック用）
- ホスト権限 — 外部 API エンドポイントへのアクセス

権限は `wxt.config.ts` の `manifest` セクションで定義する。

## アーキテクチャ概要

### ブラウザ拡張の 3 層アーキテクチャ

ブラウザ拡張は Content Script・Background（Service Worker）・Popup の 3 層で構成される。各層は `browser.runtime.sendMessage()` でメッセージをやり取りする。

```
┌──────────────────┐    メッセージ     ┌──────────────────┐    メッセージ     ┌──────────────────┐
│  Content Script  │ ──────────────→ │   Background     │ ←────────────── │     Popup        │
│  (ページ上で実行) │                  │  (Service Worker)│                  │  (React アプリ)   │
│                  │                  │                  │                  │                  │
│ - DOM 解析       │                  │ - 外部 API 呼出  │                  │ - 結果表示        │
│ - CMS 検出       │                  │ - キャッシュ管理  │                  │ - ユーザー操作     │
│ - メタ情報取得    │                  │ - タブ状態管理    │                  │                  │
└──────────────────┘                  └──────────────────┘                  └──────────────────┘
```

### データフロー

1. **検出** — Content Script がページの DOM を解析し、CMS・メタ情報等を検出する
2. **通知** — 検出結果をメッセージで Background に送信する
3. **調査** — Background が外部 API（NVD, RDAP 等）に問い合わせ、結果をキャッシュする
4. **表示** — Popup が Background にポーリングで結果を取得し、React で描画する

### 主要コンポーネント

- **Content Script**: ページ上で動作し、DOM を解析して情報を収集する
- **Background Service Worker**: 外部 API の呼び出し、レート制限、キャッシュ管理、タブごとの状態管理
- **Popup**: React アプリケーション。チェック結果をカード形式で表示する
- **Detector / Checker**: 検出・チェックロジックを純粋関数として分離したモジュール群

### 設計パターン

- **Detector パターン** — CMS 検出は `CmsDetector` インターフェースで抽象化し、新しい CMS はインターフェースを実装して登録するだけで追加できる
- **純粋関数チェッカー** — チェックロジック（URL 構造、ヘッダー、ドメイン年齢等）はデータを受け取り `CheckResult` を返す純粋関数として実装する。副作用なしでテストしやすい
- **メッセージ型の判別共用体** — Content Script / Background / Popup 間のメッセージは TypeScript の判別共用体（discriminated union）で型安全に定義する
- **レート制限** — 外部 API のレート制限に合わせたクライアントサイドのスライディングウィンドウ方式のレートリミッター
- **TTL キャッシュ** — `browser.storage.local` を使い、TTL 付きで API レスポンスをキャッシュする

## ディレクトリ構成

```
entrypoints/                 # WXT エントリポイント（自動的にビルド対象になる）
  background.ts              # Service Worker（API 呼出・状態管理）
  content.ts                 # Content Script（DOM 解析・検出）
  popup/
    index.html               # Popup の HTML シェル
    main.tsx                 # React エントリポイント
    App.tsx                  # メインコンポーネント（状態マシン）
    style.css                # Tailwind CSS エントリ
    components/              # Popup 用 React コンポーネント
utils/
  types.ts                   # 共有型定義（メッセージ型・結果型等）
  detectors/                 # CMS 検出モジュール
    index.ts                 # Detector レジストリ
    wordpress.ts             # WordPress 検出
    wordpress.test.ts        # テスト
  checks/                    # セキュリティチェックモジュール
    url.ts                   # URL 構造チェック
    headers.ts               # セキュリティヘッダーチェック
    domain-age.ts            # ドメイン年齢チェック
  nvd-client.ts              # NVD API クライアント（キャッシュ・レート制限付き）
  rdap.ts                    # RDAP API クライアント
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
  - 定数: `SCREAMING_SNAKE_CASE`（例: `NVD_API_URL`, `CACHE_TTL_MS`, `POLLING_INTERVAL_MS`）
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
| 単体テスト | Detector・Checker・ユーティリティ | `pnpm test` |
| ウォッチモード | 開発中のテスト自動実行 | `pnpm test:watch` |

### テストファイルの配置

- 対象ファイルと同じディレクトリに `.test.ts` サフィックス（例: `wordpress.ts` → `wordpress.test.ts`）
- テスト環境: Vitest + jsdom（DOM 操作を含むテスト用）
- テストヘルパー関数（`createDocument()`, `createNvdResponse()` 等）でテストデータのセットアップを簡潔にする

### テスト方針

- Detector / Checker は純粋関数として実装し、`browser` API のモックなしでテストできるようにする
- 外部 API クライアントのテストでは `vi.stubGlobal()` で `fetch` をモックする
- キャッシュ・レート制限のテストでは `vi.useFakeTimers()` で時間を制御する
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
   - 他の Issue を参照するときは番号だけでなく説明を付けた箇条書きにする（例: `- #3 — WordPress プラグイン検出`）
   - 適宜ラベルを付与する
   - `backlog` ラベル: 優先度が低く、通常の開発サイクルでは取り組まない Issue に付与する。「オープンの Issue に取り組んで」等の指示では `backlog` ラベルの Issue は対象外とする
2. **設計** — 設計チームが要件整理・アーキテクチャ設計を行い、Issue に設計内容を記載する
3. **設計レビュー** — レビューチームが設計の妥当性をチェックし、設計チームと相談・調整する
4. **ブランチ作成** — git worktree を使い、Issue に紐づくブランチで作業する
   - ブランチ命名規則: `issue-<番号>/<簡単な説明>` (例: `issue-3/add-wordpress-detector`)
5. **実装** — 実装チームが設計に基づきコーディングする
   - 不明点は設計チーム・レビューチームに相談する
6. **PR 作成** — Issue を参照する (`Closes #3` 等)
7. **コミット・プッシュ** — 作業中は適宜 commit・push を行う（main ブランチへの直接 push は禁止）
8. **コードレビュー** — マージ前にレビューチームが PR をレビューする
   - セキュリティ観点のチェックを含める（拡張機能の権限スコープ、外部 API キーの露出、XSS 等）
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

## ライセンスルール

- GPL 系ライセンスの依存パッケージは使用禁止（商用転用の可能性があるため）
- 許可: MIT, Apache-2.0, BSD-2-Clause, BSD-3-Clause, ISC 等の permissive ライセンス
- 依存追加時はライセンスを必ず確認する

## 環境ルール

- sudo は使用しない
- ツールやランタイムのインストールには mise を使用する

## 言語ルール

- Issue、コメント、PR の説明、コミットメッセージなど自然言語を書く箇所はすべて日本語で記述する
