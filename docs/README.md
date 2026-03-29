# ドキュメント一覧

FireHorse（川崎市消防局 火災通知システム）の仕様書・設計書・運用書の目次。

---

## ディレクトリ構成

```
docs/
├── spec/          システム仕様書（設計・ロジック・インフラ）
├── dev/           開発・テスト・CI/CD
└── ops/           運用・保守・利用者向け
```

---

## spec/ — システム仕様書

基本設計が変わらない限り更新しない。変更時は PR レビュー必須。

| ファイル | 内容 | 主な参照タイミング |
|---|---|---|
| [architecture.md](spec/architecture.md) | システム全体図・AWSリソース構成・コンポーネント間インターフェース仕様 | 構成変更・新機能設計時 |
| [design.md](spec/design.md) | 監視対象HTMLの構造・フィルター条件・通知振り分けルール・重複排除設計・メッセージフォーマット | フィルター・スクレイパー・通知ロジック変更時 |
| [config-spec.md](spec/config-spec.md) | `config.yaml` の全フィールド定義・バリデーションルール・エリア追加手順・`place_mappings` テンプレート変数仕様 | エリア追加・設定変更時 |
| [error-handling.md](spec/error-handling.md) | Lambdaエラーフロー・コンポーネント別例外種別・エラーメール仕様・DLQ連携・ログ出力一覧 | エラー調査・エラー処理修正時 |

---

## dev/ — 開発・テスト・CI/CD

| ファイル | 内容 | 主な参照タイミング |
|---|---|---|
| [testing.md](dev/testing.md) | テスト方針・TDDサイクル・コンポーネント別テスト要件（F-XX / S-XX / D-XX / N-XX / E2E-XX）・モック方針 | テスト追加・修正時 |
| [cicd.md](dev/cicd.md) | GitHub Actions CI/CDワークフロー・ブランチ戦略・AWS OIDC認証・Dependabot設定 | CI/CD変更・初期セットアップ時 |

---

## ops/ — 運用・保守・利用者向け

| ファイル | 内容 | 主な参照タイミング |
|---|---|---|
| [system-overview.md](ops/system-overview.md) | システム概要書（AWS構成・検知〜通知フロー・コスト目安を非技術者向けに解説） | システムの説明・管理者への引き継ぎ時 |
| [operations.md](ops/operations.md) | デプロイ前後チェックリスト・ロールバック手順・障害対応フロー・定期メンテナンス項目・コスト目安 | デプロイ・障害対応・運用時 |
| [business-spec.md](ops/business-spec.md) | 利用者向け説明書（自治会・消防団向け・非技術者向け） | 利用者への説明・配布時 |

---

## ドキュメント間の関係

```
spec/architecture.md      ← システム全体の「地図」
    │
    ├── spec/design.md         ← 処理ロジックの詳細（フィルター・通知・重複排除）
    │       └── spec/config-spec.md  ← config.yaml の設定値仕様
    │
    ├── spec/error-handling.md ← エラー時の挙動・ログ・DLQ連携
    │
    ├── dev/testing.md         ← 各コンポーネントのテスト要件
    ├── dev/cicd.md            ← テスト・デプロイの自動化
    └── ops/operations.md      ← 本番運用・障害対応
```

---

## よくある参照パターン

| やりたいこと | 参照先 |
|---|---|
| 新しい通知エリアを追加したい | [config-spec.md § エリア追加手順](spec/config-spec.md#新しいエリアを追加する手順) |
| フィルター条件を変更したい | [design.md § フィルター条件](spec/design.md#フィルター条件) + [testing.md § filter.py](dev/testing.md#5-1-filterpy最重点) |
| CloudWatch でエラーを調査したい | [error-handling.md § ログ出力一覧](spec/error-handling.md#ログ出力一覧) + [operations.md § 緊急時対応](ops/operations.md#緊急時対応フロー) |
| HTML 構造が変わった可能性がある | [design.md § 監視対象サイトの構造](spec/design.md#監視対象サイトの構造) + `scrape-qa` エージェント |
| LINE トークンを更新したい | [operations.md § LINE トークンの更新](ops/operations.md#line-トークンの更新secrets-manager) |
| 初めてデプロイする | [cicd.md § セットアップ手順](dev/cicd.md#7-github-リポジトリのセットアップ手順) + [operations.md § デプロイ前後チェックリスト](ops/operations.md#デプロイ前後チェックリスト) |
