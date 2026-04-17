[toc]

# アーキテクチャ仕様書

## 1. システム概要

### 目的

川崎市消防局の災害情報ページを定期監視し、多摩区管轄エリアの火災情報を関係者へリアルタイム通知する。

### 監視対象

| 項目 | 内容 |
|---|---|
| URL | https://sc.city.kawasaki.jp/saigai/index.htm |
| 文字コード | Shift-JIS |
| 更新方式 | ページ全体の静的HTML差し替え |
| 監視間隔 | 1分 |

---

## 2. システム全体図

```
                        インターネット
                             │
                  ┌──────────▼──────────┐
                  │  川崎市 災害情報     │
                  │  (Shift-JIS HTML)   │
                  └──────────┬──────────┘
                             │ HTTP GET (1分ごと)
┌────────────────────────────▼────────────────────────────────┐
│  AWS                                                        │
│                                                             │
│  ┌──────────────────────┐                                   │
│  │ EventBridge Scheduler│                                   │
│  │  (cron: 1分ごと)     │                                   │
│  └──────────┬───────────┘                                   │
│             │ 起動                                          │
│  ┌──────────▼───────────────────────────────────────────┐   │
│  │  Lambda Function                                     │   │
│  │                                                      │   │
│  │  ┌──────────┐  ┌──────────────────┐                  │   │
│  │  │ Scraper  │─▶│ Filter / Parser  │                  │   │
│  │  └──────────┘  └────────┬─────────┘                  │   │
│  │                         │                            │   │
│  │                ┌────────▼─────────┐                  │   │
│  │                │  DedupChecker    │                  │   │
│  │                │ (DynamoDB参照)   │                  │   │
│  │                └────────┬─────────┘                  │   │
│  │                         │ 新規のみ                   │   │
│  │                ┌────────▼─────────┐                  │   │
│  │                │    Notifier      │                  │   │
│  │                └────────┬─────────┘                  │   │
│  └───────────────────────┬─┼──────────────────────────┘    │
│                          │ │                               │
│       ┌──────────────────┘ └──────────────────┐            │
│       │                                       │            │
│  ┌────▼──────┐  ┌──────────────┐  ┌───────────▼──────┐    │
│  │ DynamoDB  │  │  Secrets     │  │   AWS SES        │    │
│  │ (状態管理)│  │  Manager     │  │  (メール送信)    │    │
│  └───────────┘  │ (トークン等) │  └──────────────────┘    │
│                 └──────────────┘                           │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  CloudWatch Logs  (Lambda実行ログ)                   │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                             │ HTTPS (LINE Messaging API)
                  ┌──────────▼──────────┐
                  │  LINE Platform      │
                  │  (broadcast)        │
                  └─────────────────────┘
```

---

## 3. コンポーネント仕様

### 3-1. Lambda Handler（`main.py`）

| 項目 | 内容 |
|---|---|
| 役割 | Lambda エントリポイント・各コンポーネントの呼び出し制御 |
| 関数シグネチャ | `def lambda_handler(event, context)` |
| 起動元 | EventBridge Scheduler（1分 cron） |
| タイムアウト設定 | 30秒（通常の実行は5秒以内） |
| エラーハンドリング | 例外発生時はエラー内容をログ出力し、SES でエラー通知メール送信 |

### 3-2. Scraper（`scraper.py`）

| 項目 | 内容 |
|---|---|
| 役割 | ページ取得・HTML解析・火災通知リスト返却 |
| 入力 | 監視URL |
| 出力 | タイムスタンプ文字列・火災通知テキストのリスト（最大6件） |
| ライブラリ | `urllib.request`、`BeautifulSoup`（lxmlパーサー） |

**HTMLパース仕様**

```
soup.find_all("font") で全 <FONT> タグを取得
  req[0]  : 出場中ヘッダー（ページ状態確認用）
  req[1]  : ページ更新タイムスタンプ
  req[2]〜req[7] : 火災通知スロット（6件固定）
```

**構造バリデーション**

パース後に以下の2つのバリデーションを実施する。いずれも失敗時は `ValueError` を送出する。
`ValueError` は `_fetch_and_parse_inner` 内でリトライ対象（最大3回・指数バックオフ）となる。全リトライ失敗後は Lambda ハンドラーが捕捉し、`WARNING` ログのみ出力して正常終了する（エラーメール・DLQ アラームは発火しない）。

| チェック | 対象 | 条件 | エラーメッセージキーワード |
|---|---|---|---|
| タグ数チェック | `<font>` タグ数 | 8未満 | `HTML 構造が想定と異なります` |
| タイムスタンプ形式 | `req[1]` のテキスト | `YYYY年M月D日 H時M分現在` 形式に合致しない | `タイムスタンプの形式が想定と異なります` |
| スロット要素数 | `req[2]〜req[7]` の各テキスト | 非空スロットを全角スペースで分割した要素数が7未満 | `スロット[N]の要素数が想定と異なります` |

空スロット（`strip()` 結果が空文字）はスロット要素数チェックをスキップする。

**タイムスタンプ先行チェック**

`req[1]` の値を DynamoDB の `firehorse-page-state` と比較し、一致する場合は即終了（後続処理をスキップ）。

### 3-3. Filter / Parser（`filter.py`）

| 項目 | 内容 |
|---|---|
| 役割 | 通知対象の絞り込み・火災通知テキストの構造化 |
| 入力 | 火災通知テキスト（文字列） |
| 出力 | `FireNotification` オブジェクト（通知先フラグ付き）または `None`（対象外） |

**火災通知テキストのパース**

全角スペース `　` で分割し、以下のフィールドに対応させる。

```
３月１５日　１２時２４分　頃　多摩区　宿河原X丁目　付近で発生した　火災の通報
  [0]          [1]       [2]   [3]      [4]             [5]             [6]
```

| インデックス | フィールド名 | 例 |
|---|---|---|
| `[0]` | `date` | `３月１５日` |
| `[1]` | `time` | `１２時２４分` |
| `[3]` | `ward` | `多摩区` |
| `[4]` | `place` | `宿河原X丁目` |
| `[6]` | `status` | `火災の通報` |

**共通フィルター条件**

```
除外条件（いずれか該当で対象外）:
  - 生田|枡形|三田|栗谷|長沢 を含む（生田管轄）
  - 高津区 を含む
  - 麻生区 を含む

通過条件（両方を満たすこと）:
  - 多摩区 を含む
  - 火災|鎮火 を含む
```

**通知先の振り分け（`status` フィールドで判定）**

| status の内容 | LINE | メール | 件名プレフィックス |
|---|---|---|---|
| `火災の通報` | ✅ | ✅ | `【火災】` |
| `鎮火` | — | ✅ | `[鎮火]` |
| `対応は終了しました` / `火災の対応は終了しました` | — | ✅ | `[完了]` |
| 上記以外・ガス漏れ・災害 | — | — | — |

### 3-4. DedupChecker（`dedup.py`）

| 項目 | 内容 |
|---|---|
| 役割 | 重複通知の防止・既出火災通知の記録 |
| 入力 | `FireNotification` |
| 出力 | 新規判定（bool） |
| ストア | DynamoDB `firehorse-seen-incidents` |

**`incident_id` の生成**

```python
incident_id = SHA256(incident.raw_text.strip())
```

### 3-5. Notifier（`notifier.py`）

| 項目 | 内容 |
|---|---|
| 役割 | LINE / メールへの通知送信 |
| 入力 | `FireNotification` リスト・ページ更新タイムスタンプ |

**LINEメッセージ**

火災通知テキスト（`raw_text`）をそのまま送信。

**メール件名フォーマット**

```
{プレフィックス}{area}:{ページ更新タイムスタンプ}{time}
例: 【火災】多摩区 宿河原X丁目:2026年03月15日 13時11分現在 １２時２４分
```

**メール本文フォーマット**

```
【 詳 細 情 報 】
{ページ更新タイムスタンプ}
           発生時刻: {date} {time}
           発生場所: {ward} {place}
           状況    : {status}
{mapion_url}

 詳細な情報は消防署(宿河原出張所)に直接電話して下さい。
0449000119

 【対象地域】
 多摩区の以下地域
 ...（対象地域一覧）
```

**エリア・Mapion URL 解決テーブル**

`place`（df[4]）をキーにエリア名と地図URLを解決する。

| place | エリア名 | Mapion URL |
|---|---|---|
| 宿河原1〜7丁目 | 宿河原X丁目 | `mapion.co.jp/address/14135/4:X/` |
| 堰1〜3丁目 | 堰X丁目 | `mapion.co.jp/address/14135/12:X/` |
| 長尾1〜7丁目 | 長尾X丁目 | `mapion.co.jp/address/14135/15:X/` |
| 登戸新町 | 登戸新町 | `mapion.co.jp/address/14135/19/` |
| 登戸 | 登戸 | `mapion.co.jp/address/14135/18/` |
| 和泉 | 多摩区和泉 | `mapion.co.jp/address/14135/2/` |
| 菅稲田堤 | 多摩区菅稲田堤 | `mapion.co.jp/address/14135/6/` |
| 菅北浦 | 多摩区菅北浦 | `mapion.co.jp/address/14135/7/` |
| 菅城下 | 多摩区菅城下 | `mapion.co.jp/address/14135/8:/` |
| 菅仙谷 | 多摩区菅仙谷 | `mapion.co.jp/address/14135/9/` |
| 菅野戸呂 | 多摩区菅野戸呂 | `mapion.co.jp/address/14135/10/` |
| 菅馬場 | 多摩区菅馬場 | `mapion.co.jp/address/14135/11/` |
| 寺尾台 | 多摩区寺尾台 | `mapion.co.jp/address/14135/13/` |
| 菅（上記以外） | 多摩区菅 | `mapion.co.jp/address/14135/5/` |
| 中野島 | 多摩区中野島 | `mapion.co.jp/address/14135/14/` |
| 布田 | 多摩区布田 | `mapion.co.jp/address/14135/22:/` |
| （該当なし） | 多摩区 | `mapion.co.jp/address/14135/` |

---

## 4. データストア仕様

### 4-1. DynamoDB: `firehorse-seen-incidents`

重複排除用。通知済み火災通知を記録する。

| 属性 | 型 | キー | 説明 |
|---|---|---|---|
| `incident_id` | String | PK | `raw_text` の SHA256ハッシュ |
| `status` | String | — | `fire` / `extinguished` / `completed` |
| `detected_at` | String | — | ISO8601タイムスタンプ（JST） |
| `raw_text` | String | — | 元の火災通知テキスト |
| `ttl` | Number | — | TTL（検出時刻 + 7日のUnixエポック秒） |

### 4-2. DynamoDB: `firehorse-page-state`

ページ更新タイムスタンプのキャッシュ。未変更時の早期終了に使用する。

| 属性 | 型 | キー | 説明 |
|---|---|---|---|
| `key` | String | PK | 固定値 `"last_timestamp"` |
| `timestamp` | String | — | 最後に処理したタイムスタンプ文字列 |

---

## 5. 外部インターフェース仕様

### 5-1. LINE Messaging API

| 項目 | 内容 |
|---|---|
| ライブラリ | `linebot.v3.messaging` |
| メソッド | `MessagingApi.broadcast` |
| 認証 | `LINE_TOKEN_SECRET_NAME`（Secrets Manager シークレット名） |
| 送信先 | チャンネル登録者全員（Broadcast） |
| メッセージ型 | `TextMessage` |

### 5-2. AWS SES

| 項目 | 内容 |
|---|---|
| ライブラリ | `boto3` |
| メソッド | `ses_client.send_email` |
| リージョン | `SES_REGION` 環境変数（未設定時は `AWS_REGION` を使用） |
| 送信元 | `SES_SENDER_EMAIL`（表示名付き可） |
| Reply-To | `SES_REPLY_EMAIL` |
| 送信先 | 環境変数 `EMAIL_RECIPIENTS`（カンマ区切り複数可） |
| エラー通知先 | 環境変数 `EMAIL_ERROR_RECIPIENTS`（カンマ区切り） |

---

## 6. 設定仕様

### 6-1. 環境変数（AWS Secrets Manager 管理）

| 変数名 | 説明 |
|---|---|
| `LINE_TOKEN_SECRET_NAME` | LINE チャネルアクセストークンの Secrets Manager シークレット名 |
| `SES_SENDER_EMAIL` | SES 送信元アドレス |
| `SES_REPLY_EMAIL` | Reply-To アドレス |
| `AWS_REGION` | Lambda デプロイリージョン（DynamoDB はこのリージョンを使用） |
| `SES_REGION` | SES リージョン（未設定時は `AWS_REGION` を使用。Lambda と SES が別リージョンの場合に設定） |
| `EMAIL_RECIPIENTS` | メール通知先アドレス（カンマ区切り。空の場合メール送信しない） |
| `EMAIL_ERROR_RECIPIENTS` | エラー通知先アドレス（カンマ区切り。空の場合エラーメールを送信しない） |
| `DYNAMODB_TABLE` | `firehorse-seen-incidents` テーブル名 |
| `DYNAMODB_STATE_TABLE` | `firehorse-page-state` テーブル名 |

### 6-2. config.yaml

| キー | デフォルト | 説明 |
|---|---|---|
| `scrape_url` | — | 監視対象URL |
| `check_interval_minutes` | `1` | 監視間隔（分）※EventBridge Scheduler の設定値と一致させること |
| `exclude_areas` | `[生田, 枡形, 三田, 栗谷, 長沢]` | 除外エリア（生田管轄） |
| `exclude_wards` | `[高津区, 麻生区]` | 除外区 |
| `target_ward` | `多摩区` | 監視対象区 |
| `incident_ttl_days` | `30` | DynamoDB TTL（日数） |

---

## 7. AWSインフラ構成

### リソース一覧

| リソース | 用途 | 管理 |
|---|---|---|
| Lambda | 監視・通知処理の実行環境 | Terraform |
| EventBridge Scheduler | 1分ごとの Lambda 起動トリガー | Terraform |
| DynamoDB | 火災通知管理・ページ状態管理（2テーブル） | Terraform |
| SES | メール送信 | Terraform（IAMポリシーのみ） |
| Secrets Manager | 機密情報管理 | Terraform |
| SQS | Dead Letter Queue（Lambda 実行失敗時のメッセージ保持） | Terraform |
| SNS | アラート通知トピック（CloudWatch Alarms → メール転送） | Terraform |
| CloudWatch Alarms | Lambda エラー数・DLQ 深度の監視アラート | Terraform |
| CloudWatch Logs | Lambda 実行ログ収集（自動作成） | Lambda設定内 |
| IAM | Lambda 実行ロール（DynamoDB・SES・SQS・Secrets Manager・X-Ray アクセス権）・GitHub Actions OIDC デプロイロール | Terraform |

### Lambda 設定

| 項目 | 値 |
|---|---|
| ランタイム | Python 3.12 |
| アーキテクチャ | x86_64 |
| メモリ | 128 MB |
| タイムアウト | 30秒 |
| デプロイ方式 | zip（Lambda Layer で依存ライブラリを管理） |
| 実行ロール | `firehorse-lambda-role` |
| 同時実行数 | 1（並列実行を抑止し二重通知を防止） |
| トレーシング | X-Ray Active モード |

### Lambda Layer 構成

`boto3` は Lambda ランタイムに含まれるため除外。以下をLayerとしてパッケージする。

| ライブラリ | 用途 |
|---|---|
| `beautifulsoup4` | HTML解析 |
| `lxml` | BeautifulSoup パーサー |
| `line-bot-sdk` (v3系) | LINE Messaging API |
| `tenacity` | HTTP リクエストのリトライ処理（指数バックオフ） |
| `PyYAML` | `config.yaml` 読み込み |
| `pydantic` (v2系) | 設定モデルのバリデーション |

### EventBridge Scheduler 設定

| 項目 | 値 |
|---|---|
| スケジュール式 | `rate(1 minute)` |
| フレキシブルウィンドウ | OFF（時刻厳守） |
| ターゲット | Lambda 関数 |

### Terraform ファイル構成

```
deploy/terraform/
├── main.tf          # プロバイダー設定・バックエンド
├── lambda.tf        # Lambda 関数・Layer・zip パッケージ
├── eventbridge.tf   # EventBridge Scheduler
├── dynamodb.tf      # DynamoDB テーブル（2テーブル）
├── iam.tf           # Lambda 実行ロール・ポリシー
├── iam_oidc.tf      # GitHub Actions OIDC プロバイダー・デプロイロール
├── secrets.tf       # Secrets Manager シークレット定義
├── monitoring.tf    # SQS DLQ・SNS トピック・CloudWatch Alarms
├── variables.tf
└── outputs.tf
```

---

## 8. 依存ライブラリ

| ライブラリ | バージョン | 用途 | 配置 |
|---|---|---|---|
| `beautifulsoup4` | 最新安定版 | HTML解析 | Lambda Layer |
| `lxml` | 最新安定版 | BeautifulSoup パーサー | Lambda Layer |
| `line-bot-sdk` | v3系 | LINE Messaging API | Lambda Layer |
| `tenacity` | 最新安定版 | HTTP リクエストのリトライ処理 | Lambda Layer |
| `PyYAML` | 最新安定版 | `config.yaml` 読み込み | Lambda Layer |
| `pydantic` | v2系 | 設定モデルのバリデーション | Lambda Layer |
| `boto3` | Lambda ランタイム同梱 | DynamoDB・SES | Lambda ランタイム |

---

## 9. 既存コードからの主な変更点

| 項目 | 既存コード | 本システム |
|---|---|---|
| Python バージョン | Python 2 | Python 3.12 |
| 実行基盤 | EC2（crontab） | Lambda + EventBridge Scheduler |
| 状態管理 | ローカルファイル（Old.txt / Date.txt） | DynamoDB（実行環境に依存しない） |
| LINE API | LINE Messaging API (`linebot.v3`) | 同左（継続） |
| HTTP fetch | `urllib2` | `urllib.request`（Python 3標準） |
| 文字コード処理 | `reload(sys)` / `setdefaultencoding` | Python 3 標準処理 |
| メール宛先 | コードにハードコード | `config.yaml` で外部管理 |
| 通知対象 | 火災・鎮火・災害・ガス漏れ | 火災・鎮火のみ |
| バグ | メールコードでスロット④⑤⑥が `req[4]` を重複参照 | `req[5]`〜`req[7]` を正しく参照 |
