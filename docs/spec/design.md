[toc]

# FireHorseCtnr 設計ドキュメント

> アーキテクチャ全体の仕様は [architecture.md](architecture.md) を参照。本ドキュメントは実装詳細・既存コード解析の補足情報を記載する。

## 概要

川崎市消防局の災害情報ページを定期的に監視し、多摩区管轄エリアの火災情報をメール（AWS SES）およびLINE（LINE Messaging API）で通知するシステム。

- 監視対象URL: https://sc.city.kawasaki.jp/saigai/index.htm
- 実行環境: AWS Lambda + EventBridge Scheduler（1分ごとに実行）

---

## アーキテクチャ

```
EventBridge Scheduler（1分 cron）
    │ 起動
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Lambda Function                                            │
│                                                             │
│  ┌──────────┐   ┌────────────────────┐                      │
│  │ Scraper  │──▶│ Filter / Parser    │                      │
│  │(Shift-JIS│   └─────────┬──────────┘                      │
│  │  decode) │             │                                 │
│  └──────────┘    ┌────────▼──────────┐                      │
│                  │   重複チェック     │                      │
│                  │  (DynamoDB参照)    │                      │
│                  └────────┬──────────┘                      │
│                  新規のみ │                                  │
│                  ┌────────▼──────────┐                      │
│                  │    Notifier        │                      │
│                  ├────────────────────┤                      │
│                  │ ・AWS SES          │                      │
│                  │ ・LINE Messaging   │                      │
│                  │   API              │                      │
│                  └────────────────────┘                      │
└─────────────────────────────────────────────────────────────┘
         │ boto3                          │ boto3
         ▼                               ▼
┌─────────────────┐             ┌─────────────────┐
│    DynamoDB     │             │    AWS SES      │
│ (既出火災通知│             │  (メール送信)   │
│  TTL: 7日間)    │             └─────────────────┘
└─────────────────┘
```

---

## 監視対象サイトの構造

### HTMLの特徴

- 文字コード: Shift-JIS
- BeautifulSoup で `soup.find_all("font")` により全 `<FONT>` タグを取得する

| インデックス | 内容 |
|---|---|
| `req[0]` | 出場中ヘッダー（「只今、川崎市内に消防車が出場しています」） |
| `req[1]` | ページ更新タイムスタンプ（例: `2026年03月15日 13時11分現在`） |
| `req[2]`〜`req[7]` | 火災通知スロット（6件固定） |

### 火災通知テキストの構造

火災通知テキストは**全角スペース**区切りで以下の要素に分解できる。

```
３月１５日　１２時２４分　頃　多摩区　宿河原X丁目　付近で発生した　火災の通報
  df[0]      df[1]     df[2]  df[3]    df[4]         df[5]          df[6]
```

`df[2]`（`頃`）と `df[5]`（`付近で発生した`）は固定文字列であり、通知処理では使用しない（意図的にスキップ）。

### df[6]（状況）のキーワードと意味

火災関連のみ通知対象とする。ガス漏れ・災害の通報は通知しない。

| df[6] の内容 | 状態 | LINE通知 | メール通知 |
|---|---|---|---|
| `火災の通報` | 火災発生 | ✅ | ✅ `【火災】` |
| `鎮火` | 鎮火確認 | — | ✅ `[鎮火]` |
| `対応は終了しました` | 対応完了 | — | ✅ `[完了]` |
| `災害は発生しておりません` | 異常なし | — | — |
| ガス漏れ関連 | — | — | — |
| 災害の通報・完了 | — | — | — |
| その他 | — | — | — |

---

## フィルター条件

LINEとメールでフィルター条件が異なる。

### 共通フィルター（LINE・メール共通）

```
① 生田|枡形|三田|栗谷|長沢 を含む → 除外（生田管轄）
② 高津区 を含む → 除外
③ 麻生区 を含む → 除外
④ 多摩区 を含む
⑤ 火災の通報|鎮火|対応は終了しました のいずれかを含む（df[6] による判定）
→ ①②③に該当せず、④⑤を満たす場合に通知候補とする
```

### 通知先の振り分け（フィルター通過後）

| df[6] の内容 | LINE | メール |
|---|---|---|
| `火災の通報` | ✅ （※LINE通知エリア制限を参照） | ✅ |
| `鎮火` / `対応は終了しました` | — | ✅ |

### LINE通知エリア制限（line_target_areas）

`config.yaml` の `line_target_areas` が設定されている場合、`df[4]`（場所）が前方一致するエリアのみ LINE通知する。

| line_target_areas の設定 | 動作 |
|---|---|
| 未設定 / 空リスト | 多摩区全体を LINE通知対象とする（後方互換） |
| `[宿河原, 堰]` | 宿河原・堰エリアのみ LINE通知する |

メール通知には `line_target_areas` は影響しない。

---

## 処理フロー

```
1分ごとにページ取得
    │
    ├─ ページ更新タイムスタンプ（req[1]）が前回と同じ → スキップ（早期終了）
    │
    └─ タイムスタンプが変化 → 各スロット（req[2]〜req[7]）を順に処理
          │
          ├─ 空スロット（全角スペースのみ） → スキップ
          │
          ├─ 火災通知テキストを全角スペースで分割し df[0]〜df[6] を取得
          │
          ├─ 共通フィルター評価
          │     ├─ 生田|枡形|三田|栗谷|長沢|高津区|麻生区 を含む → スキップ
          │     ├─ 多摩区 を含まない → スキップ
          │     └─ 火災|鎮火 を含まない → スキップ
          │
          ├─ DynamoDB に同一 incident_id が存在する → スキップ（重複排除）
          │
          └─ 存在しない（新規）
                ├─ DynamoDB に保存（TTL: 7日）
                ├─ df[6] を元に件名プレフィックスを決定
                │
                ├─【火災の通報】
                │     ├─ LINE Messaging API 送信
                │     └─ SES メール送信
                │
                └─【鎮火 / 対応は終了しました】
                      └─ SES メール送信のみ
```

---

## 重複排除設計

### incident_id の生成

火災通知テキスト全体を SHA256 ハッシュ化する。

```
incident_id = SHA256(火災通知テキスト.strip())
```

### DynamoDB テーブル設計

**テーブル名:** `firehorse-seen-incidents`

| 属性 | 型 | 説明 |
|---|---|---|
| `incident_id` (PK) | String | 火災通知テキストのSHA256ハッシュ |
| `status` | String | `fire`・`extinguished`・`completed`（`_STATUS_MAP` による変換） |
| `detected_at` | String | JST の ISO8601タイムスタンプ（例: `2026-03-15T13:24:00+09:00`） |
| `raw_text` | String | 元の火災通知テキスト |
| `ttl` | Number | Unixエポック秒（検出から `incident_ttl_days` 日後） |

**原子的書き込みによる競合防止:**

`put_item` に `ConditionExpression="attribute_not_exists(incident_id)"` を付与することで、
判定（新規か否か）と記録（DynamoDB への書き込み）を**原子的**に実行する。
Lambda が並列実行された場合でも、同一 `incident_id` の二重送信が発生しない。
条件を満たさない（既出）場合は `ConditionalCheckFailedException` が送出され、`False` を返す。

**テーブル名:** `firehorse-page-state`

| 属性 | 型 | 説明 |
|---|---|---|
| `key` (PK) | String | 固定値 `"last_timestamp"` |
| `timestamp` | String | 最後に処理したページ更新時刻テキスト |

---

## 通知メッセージフォーマット

### LINE メッセージ

火災通知テキストをそのまま送信する（既存コードの動作に準拠）。

```
３月１５日　１２時２４分　頃　多摩区　宿河原X丁目　付近で発生した　火災の通報
```

### メール件名

```
【火災】多摩区 宿河原X丁目:2026年03月15日 13時11分現在 １２時２４分
[鎮火]多摩区 宿河原X丁目:2026年03月15日 13時11分現在 １２時２４分
```

件名フォーマット: `{プレフィックス}{area}:{UpdateTime}{df[1]}`

### メール本文

```
【 詳 細 情 報 】
2026年03月15日 13時11分現在
           発生時刻: ３月１５日 １２時２４分
           発生場所: 多摩区 宿河原X丁目
           状況    : 火災の通報
http://www.mapion.co.jp/address/14135/4:1/

 詳細な情報は消防署(宿河原出張所)に直接電話して下さい。
0449000119

 【対象地域】
 多摩区の以下地域
 ...（対象地域一覧）
```

### エリア・地図URL対応表（Getplace）

df[4]（場所）をキーにエリア名・Mapion URL を解決する。主な対応:

| df[4] | エリア名 | Mapion URL |
|---|---|---|
| 宿河原1〜7丁目 | 宿河原X丁目 | `mapion.co.jp/address/14135/4:X/` |
| 堰1〜3丁目 | 堰X丁目 | `mapion.co.jp/address/14135/12:X/` |
| 長尾1〜7丁目 | 長尾X丁目 | `mapion.co.jp/address/14135/15:X/` |
| 登戸 | 登戸 | `mapion.co.jp/address/14135/18/` |
| 登戸新町 | 登戸新町 | `mapion.co.jp/address/14135/19/` |
| 中野島 | 多摩区中野島 | `mapion.co.jp/address/14135/14/` |
| 布田 | 多摩区布田 | `mapion.co.jp/address/14135/22:/` |
| 菅（各丁目） | 多摩区菅系 | 丁目別URL |
| 和泉 | 多摩区和泉 | `mapion.co.jp/address/14135/2/` |
| 寺尾台 | 多摩区寺尾台 | `mapion.co.jp/address/14135/13/` |
| （該当なし） | 多摩区 | `mapion.co.jp/address/14135/` |

---

## ディレクトリ構成

```
FireHorseCtnr/
├── src/
│   ├── app/                    # アプリケーションコード
│   │   ├── main.py             # Lambda ハンドラー・起動エントリポイント
│   │   ├── scraper.py          # サイト取得・Shift-JISデコード・HTML解析
│   │   ├── filter.py           # 共通フィルター・火災通知パーサー
│   │   ├── dedup.py            # DynamoDB による重複チェック・保存
│   │   ├── notifier.py         # SES・LINE Messaging API への通知送信
│   │   ├── config_model.py     # pydantic 設定モデル（AppConfig）
│   │   └── config.yaml         # チェック間隔・対象エリア・宛先リストなどの設定値
│   └── tests/                  # テストコード
│       ├── conftest.py         # pytest設定・共通フィクスチャ
│       ├── _types.py           # テスト共通 TypedDict 定義
│       ├── fixtures/
│       │   └── sample.html     # テスト用HTMLサンプル（実サイトから取得）
│       ├── unit/               # 純粋関数・変換ロジック（外部依存なし）
│       │   ├── test_filter.py  # 共通フィルター・パーサーのテスト
│       │   ├── test_scraper.py # HTML取得・パースのテスト
│       │   └── test_config_model.py  # AppConfig バリデーションのテスト
│       ├── integration/        # AWSサービス呼び出しを含む処理
│       │   ├── test_dedup.py   # 重複排除ロジックのテスト（moto）
│       │   └── test_notifier.py# 通知送信のテスト（SES・LINEモック）
│       └── e2e/                # Lambda全体フロー
│           └── test_handler.py # Lambda ハンドラーの結合確認
├── deploy/
│   └── terraform/              # インフラ定義（Terraform）
│       ├── main.tf
│       ├── lambda.tf           # Lambda 関数・Layer・zip パッケージ
│       ├── eventbridge.tf      # EventBridge Scheduler
│       ├── dynamodb.tf         # DynamoDB テーブル
│       ├── iam.tf              # Lambda 実行ロール・ポリシー
│       ├── secrets.tf          # Secrets Manager シークレット定義
│       ├── monitoring.tf       # SQS DLQ・SNS・CloudWatch Alarms
│       ├── variables.tf
│       └── outputs.tf
├── docs/                       # 仕様書
├── .devcontainer/
│   └── devcontainer.json
├── Makefile                    # ローカル開発・デプロイコマンド
├── requirements.txt
└── requirements-dev.txt        # 開発・テスト用依存ライブラリ
```

---

## 環境変数・設定

### 環境変数

センシティブな値（LINE トークン等）は AWS Secrets Manager で管理する。Lambda 環境変数にはシークレット名のみを格納し、実行時に取得する。

| 変数名 | 内容 |
|---|---|
| `LINE_TOKEN_SECRET_NAME` | Secrets Manager のシークレット名（LINE チャネルアクセストークン） |
| `LINE_USER_ID_SECRET_NAME` | Secrets Manager のシークレット名（送信先 LINE ユーザーID） |
| `SES_SENDER_EMAIL` | SES 送信元メールアドレス（表示名付き可） |
| `SES_REPLY_EMAIL` | SES Reply-To アドレス |
| `AWS_REGION` | Lambda デプロイリージョン（DynamoDB はこのリージョンを使用） |
| `SES_REGION` | SES リージョン（未設定時は `AWS_REGION` を使用） |
| `EMAIL_RECIPIENTS` | メール通知先アドレス（カンマ区切り） |
| `EMAIL_ERROR_RECIPIENTS` | エラー通知先アドレス（カンマ区切り） |
| `DYNAMODB_TABLE` | 火災通知管理テーブル名 |
| `DYNAMODB_STATE_TABLE` | ページ状態管理テーブル名 |

### config.yaml の設定項目

`config.yaml` の全フィールド定義・バリデーションルール・エリア追加手順は [config-spec.md](config-spec.md) を参照。

---

## 依存ライブラリ

| ライブラリ | 用途 |
|---|---|
| `urllib` (標準) | HTTP fetch |
| `beautifulsoup4` | HTML解析 |
| `lxml` | BeautifulSoup パーサー |
| `line-bot-sdk` | LINE Messaging API (`linebot.v3.messaging`) |
| `tenacity` | HTTP リトライ（指数バックオフ） |
| `PyYAML` | config.yaml の読み込み |
| `boto3` | DynamoDB・SES・Secrets Manager へのアクセス |

---

## 既存コードからの主な変更点

| 項目 | 既存コード | 新実装 |
|---|---|---|
| Python バージョン | Python 2（`urllib2`・`print`文等） | Python 3 |
| LINE API | LINE Messaging API (`linebot.v3`) | 同左（継続） |
| 状態管理 | ローカルファイル（OLD.txt, Date.txt） | DynamoDB（コンテナ再起動で消えない） |
| HTTP fetch | `urllib2.urlopen` | `urllib.request.urlopen` |
| 文字コード | `reload(sys)` / `sys.setdefaultencoding` | Python 3 標準の文字列処理 |
| メール宛先 | コードにハードコード | `config.yaml` で管理 |
| スケジューリング | crontab | EventBridge Scheduler |
| バグ修正 | メールコード④⑤⑥が req[4] を重複参照 | req[5], req[6], req[7] を正しく参照 |

---

## AWS リソース一覧

| リソース | 用途 |
|---|---|
| Lambda | 関数実行環境（zip デプロイ + Lambda Layer） |
| EventBridge Scheduler | 1分ごとの定期実行トリガー |
| DynamoDB | 既出火災通知の重複排除・ページ状態管理 |
| SES | メール送信 |
| Secrets Manager | LINE チャネルアクセストークン等の機密情報管理 |
| SQS（DLQ） | Lambda 失敗イベントの保持（サイレントドロップ防止） |
| SNS | CloudWatch アラーム通知用トピック |
| CloudWatch Logs | Lambda 実行ログの収集（構造化 JSON） |
| CloudWatch Alarms | Lambda エラー監視・DLQ 深度監視 |
| IAM | Lambda 実行ロール・EventBridge 呼び出しロール・GitHub Actions OIDC ロール |
