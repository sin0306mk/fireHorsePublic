[toc]

# テスト方針

## 1. 基本方針

### テスト設計の起点：仕様書

**テストケースの入力は「仕様書・ビジネスルール」であり、「既存の実装コード」ではない。**

既存コードを読んでテストを生成すると、「コードが実際にやっていること」を確認するだけの自作自演テストになる。仕様書から設計すれば、「コードがやるべきこと」を検証するテストになる。この違いが、テストが防護網として機能するかどうかの分かれ目になる。

AIとの協働でも同じ原則が成り立つ。Claude Code にテストを依頼する際の入力は実装コードではなく仕様書にする。仕様書を起点にすれば、自作自演テストは生まれにくく、モック境界の判断も「仕様上の関心（外部依存の境界はどこか）」から自然に導ける。

### コアビジネスロジックへの集中

すべてのコードを均一にカバーしようとしない。守るべきロジックを見極めてそこに集中する。

| 種別 | 扱い |
|---|---|
| **「間違ってはいけない」ビジネスロジック** | 仕様書の条件分岐を網羅的にテストケース化する（本プロジェクトでは `filter.py`） |
| **単純なCRUD・インフラ操作** | 重要度に応じてテストの粒度を調整する（`dedup.py` は moto 統合テストで十分） |
| **外部サービス連携** | パラメータの正確性のみ検証し、ロジックの網羅はしない（`notifier.py`） |

### カバレッジ方針

カバレッジには2つの罠がある。「カバレッジを把握していないこと」と「カバレッジがすべてだと考えること」。数値目標を定めてテストを量産するのは後者の罠にはまっている状態であり、仕様に基づいた300件のテストは実装から生成した1300件のテストよりメンテナンスコストが低く信頼性が高い。

テストコードも立派なコードベースであり、増やせば増やすほどメンテナンスコストが発生する。

- 数値達成のためだけのテストは書かない
- カバレッジが低い箇所があっても、業務上重要でなければパーセントを上げるためだけにテストを追加しない
- 未テストの分岐があれば、仕様書に対応する要件があるかを確認してからテストを追加する

### 品質原則

| 原則 | 内容 | プロジェクトへの適用 |
|---|---|---|
| **古典派テスト** | 実装詳細ではなく入出力・状態変化で検証する | `filter.py` のフィルター・パース処理を中心に、引数と戻り値で検証 |
| **仕様駆動テスト** | 業務要件からテストケースを直接導出する | 仕様書の条件（多摩区・火災判定・通知振り分け）をテストケースに1対1で対応させる |
| **統合テスト優先（CRUD処理）** | ビジネスロジックが薄いAWS操作はモックより実エミュレーターで検証する | `dedup.py` のDynamoDB読み書きは `moto` でテーブル丸ごと検証 |

---

## 2. テスト実行コマンド

```bash
# 全テスト実行（カバレッジ付き）
make test

# テストを絞り込む場合
make test ARGS='src/tests/unit/test_filter.py -k test_filter'

# カバレッジ詳細確認
make test ARGS='--cov-report=html'
```

---

## 3. テストの分類

### ディレクトリ構成

```
src/tests/
├── conftest.py             # pytest設定・共通フィクスチャ
├── fixtures/
│   └── sample.html         # テスト用HTMLサンプル（実サイトから取得）
├── unit/                   # 純粋関数・変換ロジック（外部依存なし）
│   ├── test_filter.py      # フィルター条件・火災通知パース・通知振り分け
│   └── test_scraper.py     # HTMLパース（フィクスチャHTMLを使用）
├── integration/            # AWSサービス呼び出しを含む処理
│   ├── test_dedup.py       # moto で DynamoDB をエミュレート
│   └── test_notifier.py    # SES・LINE SDK のモック
└── e2e/                    # Lambda全体フロー
    └── test_handler.py     # インライン HTML → フィルター → 重複排除 → 通知送信の結合確認
```

### コンポーネント別テスト種別と重点

| コンポーネント | テスト種別 | 重点 |
|---|---|---|
| `filter.py` | **ユニット（最重点）** | フィルター条件の網羅・境界値・通知振り分けの正確性 |
| `scraper.py` | ユニット | `fixtures/sample.html` でパース結果を検証。HTTP通信はモック |
| `dedup.py` | **統合（moto）** | DynamoDB のput/get/TTL設定を実動作で検証 |
| `notifier.py` | 統合（SDK mock） | SES・LINE APIへの送信パラメータを検証。実API呼び出しなし |
| `main.py` | **E2E** | Lambda 全体フロー（フィルター→重複排除→通知→エラー処理）の結合確認 |

---

## 4. テスト記述ルール

### 4-1. テスト構造パターン

テストの種類に応じてパターンを使い分ける。

| テスト種別 | パターン | コメント |
|---|---|---|
| **UIテスト** | Given-When-Then | ユーザーの操作シナリオを自然言語に近い形で記述する |
| **非UIテスト**（ユニット・統合） | AAA（Arrange-Act-Assert） | 準備・実行・検証の3フェーズを明示する |

**AAA パターン（非UIテスト）**

```python
def test_tama_ward_fire_notification_goes_to_line_and_email() -> None:
    # Arrange（準備）
    raw_text = "３月１５日　１２時２４分　頃　多摩区　宿河原１丁目　付近で発生した　火災の通報"

    # Act（実行）
    result = parse_and_filter(raw_text)

    # Assert（検証）
    assert result is not None
    assert result.notify_line is True
```

**Given-When-Then パターン（UIテスト・将来追加）**

```python
def test_user_receives_line_notification_on_fire(page: Page) -> None:
    # Given（前提条件）
    page.goto("/settings")
    page.check("#line-notification-enabled")

    # When（操作）
    trigger_fire_event(ward="多摩区", place="宿河原１丁目")

    # Then（結果）
    expect(page.locator(".notification-badge")).to_be_visible()
```

### 4-2. テスト命名規則

**関数名**（非パラメトリックテスト）: ビジネス上の意味が伝わる名前をつける。メソッド名・戻り値の型をテスト名に含めない。

```python
# 良い例（ビジネス上の意味が明確）
def test_unseen_notification_is_new() -> None: ...
def test_duplicate_notification_is_rejected() -> None: ...
def test_first_run_has_no_saved_timestamp() -> None: ...

# 悪い例（メソッド名・戻り値をそのまま含む）
def test_should_return_true_when_notification_is_new() -> None: ...
def test_should_return_false_when_notification_is_duplicate() -> None: ...
def test_should_return_none_when_no_timestamp_saved() -> None: ...
```

**パラメトリックテストの description 文字列**: `[条件]の場合、[振る舞い]こと`

```python
@pytest.mark.parametrize("description, raw_text, expected", [
    (
        "多摩区で火災の通報がある場合、LINE・メール両方を通知すること",
        "３月１５日　１２時２４分　頃　多摩区　宿河原１丁目　...",
        FireNotification(notify_line=True, ...),
    ),
    (
        "高津区の場合、通知しないこと",
        "３月１５日　１２時２４分　頃　高津区　...",
        None,
    ),
])
def test_filter(description: str, raw_text: str, expected: FireNotification | None) -> None:
    ...
```

### 4-3. テストを書くタイミング

テストは実装の前後どちらでもよい。ただし **仕様書を起点にすること** を必須とする。実装コードを読んでテストを後から書く場合でも、テストケースの入力は「仕様書の要件」から導出し、実装の動作をそのままなぞるだけのテストにしないこと。

### 4-4. 禁止事項

- テストを修正して実装に合わせること（実装を修正せよ）
- `pytest.mark.skip` を残したままにすること
- テストファイルで `Any` 型を使うこと
- 1つのテストで複数の異なる振る舞いを検証すること
- 実装の内部詳細（プライベートメソッド名・内部変数）に依存するテストを書くこと
- テスト関数名にメソッド名・戻り値の型を含めること（例: `test_should_return_true_when_...`）

### 4-5. 二重保証を避ける

上位コンポーネントで保証済みの観点を下位コンポーネントで繰り返してはいけない。

```python
# 悪い例: filter.py で多摩区フィルターを検証済みなのに notifier.py でも同じ条件を検証する
# → filter.py が多摩区フィルターを保証している。notifier.py は「受け取った通知を正しく送信するか」だけを検証する。
```

統合テストが保証済みの細部をユニットテストで繰り返すのも二重保証になる。テスト対象のコンポーネントが責任を持つ振る舞いのみを検証する。

### 4-6. 古典派テストを優先（ロンドン派を避ける）

モックの呼び出し回数・引数を中心に検証するロンドン派テストは避け、状態変化・出力値を検証する古典派テストを優先する。

```python
# 避けるべき（ロンドン派: 内部実装の呼び出しを検証）
mock.some_internal_method.assert_called_once_with(expected_arg)

# 推奨（古典派: 出力・状態変化を検証）
assert result == expected_value
assert table.get_item(Key={"id": id})["Item"]["ttl"] > 0
```

ただし、外部 API（LINE・SES）は実呼び出しが不可能なため、例外的にモック呼び出し検証を許容する。

### 4-7. モック方針

- **外部 API のみモック可**（内部モジュールは実際のコードを使う）
- **Dependency Injection パターンを優先**（`Notifier`・`DedupChecker` はコンストラクタ注入）
- **モック対象はフィクスチャに理由コメントを記述すること**

```python
@pytest.fixture
def ses_mock():
    # boto3（SES）をモック: 実 AWS への送信・課金を防ぐため
    with patch("src.app.notifier.boto3") as mock_boto3:
        yield mock_boto3.client.return_value

@pytest.fixture
def line_mock():
    # LINE Messaging API SDK をモック: 実ユーザーへの誤送信を防ぐため
    with (
        patch("src.app.notifier.ApiClient") as mock_api_client_cls,
        patch("src.app.notifier.MessagingApi") as mock_messaging_api_cls,
    ):
        ...
```

---

## 5. コンポーネント別テスト方針

### 5-1. `filter.py`（最重点）

業務要件を直接反映する最重要モジュール。フィルター条件を網羅的にテストする。

**テスト対象の要件**

| # | 要件 | テストケース例 |
|---|---|---|
| F-01 | 多摩区を含むこと | 他区（高津区・麻生区）は除外 |
| F-02 | 火災\|鎮火を含むこと | ガス漏れ・災害の通報は除外 |
| F-03 | 生田管轄（生田\|枡形\|三田\|栗谷\|長沢）を除外 | 地名が場所フィールドに含まれる場合 |
| F-04 | `火災の通報` → LINE＋メール | `notify_line=True, notify_email=True` |
| F-05 | `鎮火` → メールのみ | `notify_line=False, notify_email=True` |
| F-06 | `対応は終了しました` → メールのみ | `notify_line=False, notify_email=True` |
| F-07 | 火災通知テキストの全角スペース分割 | `df[0]`〜`df[6]` への正確なマッピング |
| F-08 | 空スロット（全角スペースのみ）はスキップ | `None` を返す |

### 5-2. `scraper.py`

`fixtures/sample.html` を使いHTMLパースのみをテストする。HTTP通信は `unittest.mock` でモックし外部アクセスしない。

**テスト対象の要件**

| # | 要件 |
|---|---|
| S-01 | `req[1]` にタイムスタンプが取得できること |
| S-02 | `req[2]`〜`req[7]` に火災通知テキストが取得できること（6スロット固定） |
| S-03 | Shift-JISが正しくデコードされること |
| S-04 | `http`/`https` 以外のスキームを渡した場合、`ValueError` が送出されること |
| S-05 | 一時的なネットワークエラーが発生した場合、リトライして成功すること |
| S-06 | リトライ上限を超えた場合、例外が伝播すること |
| S-07 | `<font>` タグ数が不足している場合、`ValueError` が送出されること |

### 5-3. `dedup.py`

`moto` ライブラリで DynamoDB をエミュレートし、実際のテーブル操作として検証する。モックに置き換えない理由は、TTL設定ミスや条件式のバグをモックでは検出できないため。

**テスト対象の要件**

| # | 要件 |
|---|---|
| D-01 | 未登録の `incident_id` は新規と判定されること |
| D-02 | 登録済みの `incident_id` は重複と判定されること |
| D-03 | 新規火災通知保存時に TTL（検出時刻 + 7日）が設定されること |
| D-04 | `incident_id` が `SHA256(raw_text.strip())` で生成されること |
| D-05 | 初回実行時（未保存状態）はページタイムスタンプが `None` を返すこと |
| D-06 | 保存したページタイムスタンプが取得できること |
| D-07 | ページタイムスタンプを再保存した場合、最新の値に上書きされること |

### 5-4. `notifier.py`

LINE SDK・boto3（SES）は `unittest.mock.patch` でモックし、実API呼び出しを行わない。検証対象は「正しいパラメータで呼び出されているか」。

**テスト対象の要件**

| # | 要件 |
|---|---|
| N-01 | `火災の通報` 時に LINE に通知されること |
| N-02 | `鎮火` 時に LINE に通知されないこと |
| N-03 | メール件名フォーマットが仕様通りであること（`{プレフィックス}{area}:{タイムスタンプ}{time}`） |
| N-04 | `place` からエリア名・Mapion URLが正しく解決されること |
| N-05 | `email_recipients` に設定された全宛先に送信されること |
| N-06 | `ReplyToAddresses` に `ses_reply` が設定されること |

### 5-5. `main.py`（E2E）

DynamoDB は moto 実エミュレーション、SES・LINE SDK・HTTP fetch は unittest.mock でモックし、Lambda ハンドラー全体フローを結合確認する。

**テスト対象の要件**

| # | 要件 |
|---|---|
| E2E-01 | 新規火災通報スロットがある場合、LINE + メールが送信されること |
| E2E-02 | ページタイムスタンプが前回と同じ場合、処理をスキップして `"no change"` を返すこと |
| E2E-03 | ページ更新後も同一通知スロットを検出した場合、重複排除により通知されないこと |
| E2E-04 | 鎮火スロットがある場合、メールのみ送信され LINE には通知されないこと |
| E2E-05 | 通知対象外スロットのみの場合、通知が一切行われないこと |
| E2E-06 | 例外発生時に構造化エラーログを出力し、例外を再送出すること |
| E2E-07 | 例外発生時に `email_error_recipients` が設定されている場合、エラーメールを送信すること |
| E2E-08 | エラーメール送信自体が失敗した場合、`error_email_failed` ログを出力すること |

---

## 6. 設計上の決定事項

テスト種別・対象を決める際の判断理由を記録する。「なぜモックではなく moto か」「なぜ実 API を呼ばないか」を明示することで、後から変更する際の根拠になる。

### DynamoDB は `moto` で統合テストにする

`dedup.py` のテストで DynamoDB をモックオブジェクトに置き換えない理由：

- TTL 属性の設定漏れ
- `put_item` の条件式ミス
- テーブル名・キー名の設定ミス

これらはモックでは検出できず、`moto` による実エミュレーションで初めて発見できる。

### 外部 API（LINE・SES）は必ずモックにする

LINE Messaging API と SES は実呼び出しで課金・通知の副作用が発生するため、テスト時は必ずモックにする。検証対象は「送信パラメータの正確性」に限定する（4-5節の例外ルール）。

### HTTP 通信は必ずモックにする

`scraper.py` のテストで実サイトへの HTTP アクセスは行わない。`fixtures/sample.html` をローカルフィクスチャとして使用する。外部サイトの HTML 構造変更でテストが壊れることを防ぐため。
