# エラーハンドリング仕様

Lambda ハンドラー（`main.py`）のエラー処理フローと各コンポーネントのエラー伝播を定義する。

---

## Lambda ハンドラーのエラーフロー

```
lambda_handler()
    │
    ├─ 正常処理
    │     └─ {"statusCode": 200, "body": "ok"} を返す
    │
    └─ 例外発生
          │
          ├─【1】構造化エラーログを CloudWatch Logs に出力
          │     {"event": "error", "error_type": "...", "message": "...", "traceback": "..."}
          │
          ├─【2】EMAIL_ERROR_RECIPIENTS 環境変数が空でない場合
          │     └─ SES でエラー通知メールを送信
          │           件名: "[FireHorse ERROR] {ExceptionType}: {message}"
          │           本文: traceback 全文
          │
          │     エラーメール送信自体が失敗した場合:
          │     └─ {"event": "error_email_failed", "message": "..."} をログ出力
          │           ※ re-raise しない（元の例外のみ伝播させる）
          │
          └─【3】元の例外を re-raise
                → Lambda が失敗として記録される
                → EventBridge がリトライせず DLQ にイベントを送る（非同期呼び出しの場合）
```

---

## コンポーネント別エラー種別

### `scraper.py`

| 例外 | 発生条件 | ハンドラーの処理 |
|---|---|---|
| `ValueError` | URLスキームが http/https 以外 | エラーログ → エラーメール → re-raise |
| `ValueError` | `<font>` タグ数が不足（HTML構造変更） | エラーログ → エラーメール → re-raise |
| `urllib.error.URLError` | ネットワークエラー | tenacity が最大3回リトライ後、上記に同じ |

`ValueError` のメッセージには取得できたタグの先頭3件の内容が含まれるため、CloudWatch Logs だけで HTML 構造変化の概要を確認できる。

### `dedup.py`

| 例外 | 発生条件 | ハンドラーの処理 |
|---|---|---|
| `ClientError` | DynamoDB 接続失敗 / テーブル不在 | エラーログ → エラーメール → re-raise |
| `ConditionalCheckFailedException` | 重複通知（正常系として吸収） | `is_new_and_mark()` が `False` を返す。例外はキャッチ済み |

### `notifier.py`

| 例外 | 発生条件 | ハンドラーの処理 |
|---|---|---|
| `ApiException` | LINE API エラー（トークン失効等） | エラーログ → エラーメール → re-raise |
| `ClientError` | SES 送信エラー（アドレス未検証等） | エラーログ → エラーメール → re-raise |

### `config_model.py`

| 例外 | 発生条件 | ハンドラーの処理 |
|---|---|---|
| `ValidationError` | `config.yaml` の型・値が不正 | エラーログ → エラーメール → re-raise |
| `FileNotFoundError` | `config.yaml` が見つからない | エラーログ → エラーメール → re-raise |

---

## エラーメール仕様

環境変数 `EMAIL_ERROR_RECIPIENTS` が設定されている場合のみ送信する。

| 項目 | 内容 |
|---|---|
| 送信元 | `SES_SENDER_EMAIL` 環境変数 |
| 宛先 | `EMAIL_ERROR_RECIPIENTS` 環境変数（カンマ区切り） |
| 件名 | `[FireHorse ERROR] {ExceptionType}: {message}` |
| 本文 | Python `traceback.format_exc()` の全文 |

エラーメール送信が失敗しても Lambda の失敗判定には影響しない（元の例外が re-raise されるため）。

---

## DLQ との連携

Lambda が例外を re-raise して失敗した場合の挙動（EventBridge Scheduler 非同期呼び出し時）：

1. EventBridge が設定に応じてリトライ（デフォルト: リトライなし）
2. リトライ上限後、DLQ（`firehorse-dlq`）にイベントを送信
3. DLQ の深度が 1 以上になると CloudWatch アラーム（`firehorse-dlq-depth`）が発火
4. アラームは SNS 経由で `alarm_email` に通知

DLQ のメッセージ確認・削除手順は `docs/ops/operations.md` を参照。

---

## ログ出力一覧

| `event` 値 | 出力タイミング | レベル |
|---|---|---|
| `timestamp_unchanged` | ページ変化なしでスキップ | INFO |
| `notification_skipped` | 重複排除でスキップ | INFO |
| `notification_sent` | 通知送信完了 | INFO |
| `timestamp_saved` | タイムスタンプ保存完了 | INFO |
| `fetch_retry` | HTTP リトライ発生 | INFO |
| `error` | ハンドラーで例外をキャッチ | ERROR |
| `error_email_failed` | エラーメール送信失敗 | ERROR |
