# FireHorse

川崎市消防局の災害情報ページを定期監視し、多摩区管轄エリアの火災情報をメール・LINEで通知するシステムです。

## 概要

- **監視対象**: 川崎市 災害情報を1分ごとにチェック
- **通知対象**: 多摩区管轄エリア（宿河原・堰・登戸・長尾・和泉・菅・寺尾台・中野島・布田）の火災情報
- **通知手段**:
  - 出場中（`火災の通報`）: LINE Messaging API + AWS SES（メール）
  - 鎮火（`鎮火` / `対応は終了しました`）: AWS SES（メール）のみ
- **実行環境**: AWS Lambda + EventBridge Scheduler（1分 cron）

## 通知フロー

```
監視 (1分ごと)
 └─ ページ更新タイムスタンプが変化した？
       └─ 多摩区管轄エリアの火災・鎮火情報あり？
             └─ 既通知済み？（DynamoDB で重複排除）
                   ├─ 火災（出場中）→ LINE Messaging API + SES メール送信
                   └─ 鎮火        → SES メール送信のみ
```

## ディレクトリ構成

```
FireHorseCtnr/
│
├── src/                        # ソースコード・テスト資産
│   ├── app/                    # アプリケーションコード（Lambda 本体）
│   │   ├── main.py             # Lambda ハンドラー・起動エントリポイント
│   │   ├── scraper.py          # サイト取得・Shift-JIS デコード・HTML 解析
│   │   ├── filter.py           # エリアフィルター・火災情報抽出
│   │   ├── dedup.py            # DynamoDB による重複チェック・保存
│   │   ├── notifier.py         # SES・LINE Messaging API への通知送信
│   │   └── config.yaml         # 対象エリア・通知設定
│   └── tests/                  # テストコード
│       ├── unit/               # ユニットテスト
│       ├── integration/        # 統合テスト（moto / モック使用）
│       ├── e2e/                # E2E テスト（Lambda 全体フロー）
│       └── fixtures/           # テスト用 HTML サンプル等
│
├── deploy/                     # デプロイ資産
│   ├── docker/                 # コンテナ定義
│   │   ├── Dockerfile.test     # テスト実行用イメージ
│   │   ├── Dockerfile.deploy   # デプロイ用イメージ（Terraform・AWS CLI 含む、linux/amd64 固定）
│   │   └── Dockerfile.status   # 運用確認用イメージ（AWS CLI のみ、ホスト CPU アーキテクチャ自動判定）
│   ├── terraform/              # AWS インフラ定義（Terraform）
│   ├── requirements.txt        # 本番依存ライブラリ
│   ├── requirements-dev.txt    # 開発・テスト用依存ライブラリ
│   ├── .env.example            # 環境変数テンプレート
│   └── .env                    # デプロイ設定（gitignore・要作成）
│
├── docs/                       # 仕様書・設計ドキュメント
└── Makefile                    # ローカル開発・デプロイコマンド
```

## AWS リソース

| リソース | 用途 |
|---|---|
| Lambda | 監視・通知処理の実行環境（Python 3.12） |
| EventBridge Scheduler | 1分ごとの Lambda 起動トリガー |
| DynamoDB | 重複排除（`firehorse-seen-incidents`）・ページ状態キャッシュ（`firehorse-page-state`） |
| SES | メール送信 |
| Secrets Manager | LINE チャネルアクセストークン・LINE ユーザー ID の管理 |
| S3 | Terraform ステート保存（`firehorse-terraform-state`） |

## ローカル開発

### 前提条件

- Docker（テスト実行環境）
- AWS CLI + 認証情報 CSV（デプロイ時のみ）

### テスト・Lint コマンド

```bash
# テスト用 Docker イメージをビルド
make build

# テスト実行（カバレッジ付き）
make test

# テストを絞り込む場合
make test ARGS='src/tests/unit/test_filter.py -k test_filter'

# lint（ruff format / ruff check / mypy）
make lint

# セキュリティ検査（bandit / pip-audit）
make security

# CI と同じ順序で全実行
make ci

# Docker イメージを削除
make clean
```

## デプロイ

### 前提条件

AWS コンソールの「セキュリティ認証情報」から IAM ユーザーのアクセスキーを CSV 形式でダウンロードし、**`FireHorseCtnr/` と同じディレクトリ**（一つ上の階層）に配置してください。

```
親ディレクトリ/
├── credentials_xxxxxxxx.csv   ← ここに配置
└── FireHorseCtnr/             ← リポジトリ
```

CSV のフォーマット（AWS コンソール出力形式）:

```
User name,Access key ID,Secret access key
firehorse-deploy,AKIAIOSFODNN7EXAMPLE,wJalrXUtnFEMI/K7MDENG/...
```

### デプロイコマンド

すべての deploy 操作は `deploy/docker/Dockerfile.deploy` で構築したコンテナ内で実行されます（Terraform・AWS CLI・Python が含まれる隔離環境）。

デプロイ前に `deploy/.env.example` を参考に `deploy/.env` を作成してください。

```bash
# デプロイ用 Docker イメージをビルド（初回・Dockerfile.deploy 変更時）
make build-deploy

# Lambda zip・Layer zip をビルド（コンテナ内）
make package

# Terraform 初期化（初回のみ・S3 バックエンドに接続、コンテナ内）
make tf-init

# デプロイ内容の確認（terraform plan、コンテナ内）
make plan

# AWS へデプロイ（terraform apply、コンテナ内）
make deploy

# インフラを全削除（要確認フラグ、コンテナ内）
make destroy CONFIRM=yes
```

### デプロイフロー

```
make deploy
  └─ make plan
       └─ make package   （コンテナ内: Lambda zip + Layer zip ビルド）
       └─ make tf-init   （コンテナ内: terraform init）
        ↓
        ホストで CSV 読み取り → コンテナに AWS 認証情報を注入
        コンテナ内で Secrets Manager から LINE トークン取得
        terraform plan -out=tfplan
  ↓
  terraform apply -auto-approve tfplan
```

> **注意**: `make destroy CONFIRM=yes` はすべての AWS リソースを削除します。誤実行防止のため `CONFIRM=yes` が必須です。

## 運用確認コマンド

AWS 認証情報 CSV（`../` に配置）と `deploy/.env` があれば、Docker 経由で以下を実行できます。

```bash
# DynamoDB 両テーブルの件数・最新レコードを表示
make ddb

# Lambda 直近ログを表示（デフォルト 20 件）
make logs
make logs LOGS_LINES=50        # 件数変更
make logs FILTER=ERROR         # エラーのみ

# LINE 今月の送信数・上限・友達数を表示
make line

# CloudWatch Alarms 状態 + 上記すべてを一括確認
make status
```

## ドキュメント

詳細は [docs/README.md](docs/README.md) を参照。

| ディレクトリ | 主なファイル | 内容 |
|---|---|---|
| `docs/spec/` | architecture.md / design.md / config-spec.md / error-handling.md | システム仕様書（設計・ロジック・インフラ） |
| `docs/dev/` | testing.md / cicd.md | テスト方針・CI/CD |
| `docs/ops/` | system-overview.md / operations.md / business-spec.md | 運用・管理者向けガイド・利用者向け説明書 |
