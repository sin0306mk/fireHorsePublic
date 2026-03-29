# 運用手順書

川崎市消防局火災通知システム（FireHorse）の運用・保守手順をまとめたドキュメント。

---

## デプロイ前後チェックリスト

### デプロイ前

- [ ] `make ci`（lint + test + security）がすべてパスしていること
- [ ] `make plan` で差分が意図通りであることを確認
- [ ] Secrets Manager の値（LINE トークン・ユーザー ID）が最新であること
- [ ] SES 送信元メールアドレスが AWS で検証済みであること

### デプロイ後

- [ ] AWS コンソールで Lambda の最終実行結果が `Succeeded` であること
- [ ] CloudWatch Logs（`/aws/lambda/firehorse`）に `{"event": "timestamp_unchanged"}` または `{"event": "timestamp_saved"}` が出力されていること
- [ ] CloudWatch アラーム（`firehorse-lambda-errors`, `firehorse-dlq-depth`）が `OK` 状態であること
- [ ] `alarm_email` を設定した場合、SNS 購読確認メールを承認済みであること

---

## ロールバック手順

### Lambda コードのロールバック

Lambda は zip デプロイのため、前バージョンの zip を再デプロイする。

```bash
# 直前のコミットに戻す場合
git revert HEAD
git push origin main
# → GitHub Actions CD が自動で前バージョンをデプロイ
```

Terraform 管理外で緊急ロールバックが必要な場合：

```bash
# AWS CLI で直前バージョンに切り戻し（Lambda バージョン発行が前提）
aws lambda update-function-code \
  --function-name firehorse \
  --s3-bucket <bucket> --s3-key <previous-key>
```

### Terraform インフラのロールバック

```bash
# plan で差分を確認してから apply
git checkout <previous-commit> -- deploy/terraform/
make plan
make deploy
```

---

## 緊急時対応フロー

### Lambda エラーアラームが発火した場合

1. CloudWatch Logs で直近のエラーログを確認

```
# Log Insights クエリ例
fields @timestamp, @message
| filter @message like /"event": "error"/
| sort @timestamp desc
| limit 20
```

2. エラー種別に応じて対処：

| エラー種別 | 原因候補 | 対処 |
|---|---|---|
| `URLError` | 川崎市サイト障害 / ネットワーク瞬断 | tenacity が3回リトライするため一時障害は自動回復。継続する場合は Lambda を手動実行して確認 |
| `ValueError: HTML 構造が想定と異なります` | 川崎市サイトの HTML 構造変更 | `scraper.py` の `find_all("font")` ロジック修正が必要 |
| `KeyError` | `config.yaml` キー名の誤り / 欠落 | config.yaml の内容を確認 |
| `ClientError` (Secrets Manager) | IAM ポリシー不整合 / シークレット削除 | `terraform plan` で IAM 差分を確認、シークレットの存在確認 |

3. DLQ にメッセージが溜まっている場合：

```bash
# DLQ の中身を確認（最大10件）
aws sqs receive-message \
  --queue-url $(terraform -chdir=deploy/terraform output -raw dlq_url) \
  --max-number-of-messages 10

# 調査完了後にメッセージを削除
aws sqs purge-queue \
  --queue-url $(terraform -chdir=deploy/terraform output -raw dlq_url)
```

### LINE 通知が届かない場合

1. CloudWatch Logs で `{"event": "notification_sent", "notify_line": true}` が出力されているか確認
2. 出力されている場合 → LINE SDK / API の問題。LINE Developer Console でトークンの有効期限・権限を確認
3. 出力されていない場合 → フィルター条件を確認（`config.yaml` の `exclude_areas`, `target_ward`）

### メール通知が届かない場合

1. SES コンソールで送信元メールアドレスの検証状態を確認
2. SES サンドボックスモードの場合、宛先アドレスも検証済みである必要がある
3. SES バウンス・苦情レートが閾値超過でアカウントが停止されていないか確認

---

## 定期メンテナンス項目

### 月次

- [ ] `pip-audit` でライブラリの CVE を確認（CI で自動実行されるが目視確認推奨）
- [ ] LINE チャネルアクセストークンの有効期限を確認（発行から 30 日）
- [ ] `make line` で今月の LINE 送信数・残数を確認（無料プラン上限 200件/月）

### 四半期

- [ ] `src/tests/fixtures/sample.html` を川崎市サイトから最新版に差し替え
  ```bash
  curl -o src/tests/fixtures/sample.html \
    --compressed "https://sc.city.kawasaki.jp/saigai/index.htm"
  ```
- [ ] CloudWatch Logs のコスト確認（1分ごとの実行は月約43,200回）

### 年次

- [ ] AWS OIDC ロール・IAM ポリシーの棚卸し（不要な権限がないか確認）
- [ ] Terraform Provider バージョンのアップデート（`deploy/terraform/main.tf` の `required_providers`）

---

## 設定変更手順

### 監視エリアの変更（`config.yaml`）

```yaml
# 除外エリア・除外区・対象区を変更する場合
exclude_areas:
  - 生田
  - 枡形
  # ... 追加・削除
```

変更後は通常の PR → マージ → CD デプロイで反映される（Lambda 再起動不要）。

### LINE トークンの更新（Secrets Manager）

```bash
aws secretsmanager update-secret \
  --secret-id firehorse/line-channel-access-token \
  --secret-string "<new-token>"
```

Lambda は次回実行時に新しい値を取得するため再起動不要。

### 監視先 URL の変更（`config.yaml`）

```yaml
scrape_url: "https://新しいURL"
```

PR → マージ → CD デプロイで反映。

---

## コスト目安（ap-northeast-1、2026年3月時点）

| リソース | 実行頻度 | 月間推計 |
|---|---|---|
| Lambda 実行 | 約 43,200 回/月 | 無料枠内（100万回/月まで無料） |
| DynamoDB | 読み書き数に依存 | < $1 |
| Secrets Manager | 2シークレット × $0.40/月 | $0.80 |
| CloudWatch Logs | ログ量に依存 | < $1 |
| EventBridge Scheduler | 43,200 回/月 | 無料枠内（100万回/月まで無料） |
| **合計** | | **< $5/月** |
