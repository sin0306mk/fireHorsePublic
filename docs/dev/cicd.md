[toc]

# CI/CD 仕様書

## 1. 概要

GitHub Actions を使用してコード品質の維持と AWS Lambda への自動デプロイを実現する。

| パイプライン | トリガー | 目的 |
|---|---|---|
| CI | `src/**` への push（全ブランチ）・main への PR | lint・テスト・セキュリティ検査・Terraform検証 |
| CD | `src/**` が main へ push されたとき | Lambda zip ビルド・AWS デプロイ |

**`docs/**` のみの変更では CI・CD ともに実行しない。**

---

## 2. ブランチ戦略

```
main
 └── feature/* または fix/*  （作業ブランチ）
       └── PR → main でマージ時に CD 実行
```

- `main` が本番環境に対応する唯一のブランチ
- 作業は必ずブランチを切り、PR 経由でマージする
- CI が全て通過しないと PR をマージできないよう Branch Protection Rule を設定する

---

## 3. ワークフロー構成

```
.github/
└── workflows/
    ├── ci.yml   # src/** の push（全ブランチ）・main への PR で実行
    └── cd.yml   # src/** が main に push されたとき実行
```

---

## 4. CI ワークフロー（`ci.yml`）

### トリガー

```yaml
on:
  push:
    paths:
      - 'src/**'          # src/ 配下の変更がある push のみ実行
  pull_request:
    branches: [main]
    paths:
      - 'src/**'          # src/ 配下の変更がある PR のみ実行
```

`docs/`・`deploy/`・`.github/` のみの変更では CI は実行されない。

### ジョブ構成

```
ci
├── lint          # コードスタイル・静的解析
├── test          # 自動テスト・カバレッジ
├── security      # セキュリティ検査
└── terraform     # インフラ定義の検証（src/ 変更時も常に実行）
```

### lint ジョブ

| ステップ | ツール | 内容 |
|---|---|---|
| フォーマットチェック | `ruff format --check` | コードフォーマットの確認 |
| リントチェック | `ruff check` | コーディング規約・バグパターンの検出 |
| 型チェック | `mypy src/app/` | 型アノテーションの整合性確認 |

### test ジョブ

| ステップ | ツール | 内容 |
|---|---|---|
| テスト実行 | `pytest src/tests/` | 全テストケースの実行 |
| カバレッジ計測 | `pytest --cov=src/app --cov-report=xml` | カバレッジレポート生成 |
| カバレッジ表示 | GitHub Actions Summary | PR・push のサマリーにカバレッジを表示 |

テスト時の外部依存はすべてモックに置き換える。

| モック対象 | ライブラリ |
|---|---|
| DynamoDB | `moto` |
| AWS SES | `moto` |
| LINE Messaging API | `unittest.mock` |
| HTTP fetch（川崎市サイト） | `unittest.mock` / `responses` |

### security ジョブ

| ステップ | ツール | 内容 |
|---|---|---|
| Pythonコードの脆弱性検査 | `bandit -r src/app/` | インジェクション・ハードコード秘密情報などの検出 |
| 依存ライブラリの脆弱性検査 | `pip-audit` | CVE 情報との照合 |

### terraform ジョブ

| ステップ | ツール | 内容 |
|---|---|---|
| フォーマットチェック | `terraform fmt -check -recursive` | HCL フォーマットの確認 |
| 構文検証 | `terraform validate` | Terraform 定義の構文チェック |
| リントチェック | `tflint` | ベストプラクティス・非推奨記法の検出 |
| セキュリティ検査 | `checkov -d deploy/terraform/` | IAM 過剰権限・暗号化設定漏れなどの検出 |

> terraform ジョブは AWS 認証不要（`terraform validate` はローカル検証のみ）

---

## 5. CD ワークフロー（`cd.yml`）

### トリガー

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'src/**'          # src/ 配下の変更が main に push されたときのみ実行
```

### ジョブ構成

```
cd
├── build    # Lambda zip パッケージ・Layer のビルド
└── deploy   # AWS へのデプロイ（Terraform apply）
```

### build ジョブ

| ステップ | 内容 |
|---|---|
| アプリコードの zip 化 | `src/app/` を `lambda_function.zip` としてパッケージ |
| Lambda Layer の zip 化 | `beautifulsoup4`・`lxml`・`line-bot-sdk` を `lambda_layer.zip` としてパッケージ |
| アーティファクトの保存 | `upload-artifact` で deploy ジョブへ引き渡し |

### deploy ジョブ

| ステップ | 内容 |
|---|---|
| AWS 認証 | OIDC（`aws-actions/configure-aws-credentials`） |
| Terraform init | S3 バックエンドの初期化 |
| Terraform plan | 変更内容の確認・ログ出力 |
| Terraform apply | インフラ・Lambda の自動更新 |

---

## 6. AWS 認証（OIDC）

長期的な IAM アクセスキーを GitHub Secrets に保存しない。GitHub Actions の OIDC を使用して一時的な認証情報を取得する。

```
GitHub Actions
    │ OIDC トークン発行
    ▼
AWS IAM Identity Provider（github.com）
    │ AssumeRoleWithWebIdentity
    ▼
IAM ロール: firehorse-github-actions-role
    │
    ├── Lambda 更新権限
    ├── S3（Terraform state）読み書き権限
    ├── DynamoDB 管理権限
    └── Secrets Manager 管理権限
```

LINE トークンやメールアドレス等のアプリケーション秘密情報は GitHub Secrets には置かず、**AWS Secrets Manager で管理**する。

---

## 7. GitHub リポジトリのセットアップ手順

CI/CD を動作させるために GitHub 上で以下の設定を行う。

### 7-1. Branch Protection Rules（`main` ブランチ）

`Settings` → `Branches` → `Add branch protection rule` で以下を設定する。

| 設定項目 | 値 |
|---|---|
| Branch name pattern | `main` |
| Require a pull request before merging | ✅ ON |
| Require status checks to pass before merging | ✅ ON |
| 必須ステータスチェック | `lint`・`test`・`security`・`terraform` |
| Require branches to be up to date before merging | ✅ ON |
| Do not allow bypassing the above settings | ✅ ON |

### 7-2. GitHub Secrets の登録

`Settings` → `Secrets and variables` → `Actions` → `New repository secret` で以下を登録する。

| Secret 名 | 内容 | 取得元 |
|---|---|---|
| `AWS_ROLE_ARN` | OIDC で引き受ける IAM ロールの ARN | Terraform apply 後に `outputs.tf` から取得 |
| `AWS_REGION` | デプロイ先リージョン（例: `ap-northeast-1`） | 任意で設定 |

### 7-3. GitHub Actions の権限設定

`Settings` → `Actions` → `General` → `Workflow permissions` で以下を設定する。

| 設定項目 | 値 |
|---|---|
| Workflow permissions | `Read and write permissions` |
| Allow GitHub Actions to create and approve pull requests | ✅ ON（カバレッジコメント投稿に使用） |

### 7-4. AWS 側の OIDC プロバイダー設定（初回のみ）

Terraform apply 前に AWS コンソールまたは CLI で GitHub Actions 用の OIDC プロバイダーを登録する。
（`deploy/terraform/iam_oidc.tf` で Terraform 管理するため、初回は手動 apply または CLI で作成する）

```
IAM → Identity providers → Add provider
  Provider type : OpenID Connect
  Provider URL  : https://token.actions.githubusercontent.com
  Audience      : sts.amazonaws.com
```

### 7-5. Dependabot の設定更新

`.github/dependabot.yml` に Python パッケージと GitHub Actions の自動更新を追加する。

```yaml
version: 2
updates:
  - package-ecosystem: "devcontainers"   # 既存
    directory: "/"
    schedule:
      interval: "weekly"

  - package-ecosystem: "pip"             # 追加
    directory: "/"
    schedule:
      interval: "weekly"

  - package-ecosystem: "github-actions"  # 追加
    directory: "/"
    schedule:
      interval: "weekly"
```

---

## 8. ファイル・ツールの追加

### 追加するファイル

```
.github/
└── workflows/
    ├── ci.yml
    └── cd.yml
deploy/
└── terraform/
    └── iam_oidc.tf     # GitHub Actions 用 OIDC プロバイダー・IAM ロール定義
```

### `requirements-dev.txt`（開発・CI用依存）

```
ruff
mypy
pytest
pytest-cov
moto[dynamodb,ses]
responses
bandit
pip-audit
```

### `pyproject.toml`（ツール設定）

ruff・mypy・pytest の設定を一元管理する。

---

## 9. 全体資産構成への追加分

```
FireHorseCtnr/
├── .github/
│   ├── dependabot.yml              # pip・github-actions を追加
│   └── workflows/
│       ├── ci.yml                  # lint・test・security・terraform 検証
│       └── cd.yml                  # build・deploy
├── src/
│   └── ...（既存）
├── deploy/
│   └── terraform/
│       ├── iam_oidc.tf             # GitHub Actions OIDC 設定（追加）
│       └── ...（既存）
├── pyproject.toml                  # ruff・mypy・pytest 設定（追加）
├── requirements.txt                # Lambda 本番依存
└── requirements-dev.txt            # CI・開発用依存（追加）
```
