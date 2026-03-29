# config.yaml 仕様書

`src/app/config.yaml` の全フィールド定義・バリデーションルール・設定手順を記載する。
設定値は Lambda デプロイ時にコードと一緒にパッケージされ、起動時に pydantic で検証される。

---

## フィールド一覧

### トップレベル

| フィールド | 型 | デフォルト | バリデーション | 説明 |
|---|---|---|---|---|
| `check_interval_minutes` | int | `1` | ≥ 1 | EventBridge Scheduler の実行間隔（分）。コード側での参照のみで、Scheduler の設定変更は Terraform で行う |
| `scrape_url` | str | 必須 | — | 監視対象 URL |
| `exclude_areas` | list[str] | 必須 | — | 除外するエリア名リスト（生田管轄）。テキスト全体に含まれるかで判定 |
| `exclude_wards` | list[str] | 必須 | — | 除外する区名リスト。テキスト全体に含まれるかで判定 |
| `target_ward` | str | 必須 | — | 通知対象の区名（例: `多摩区`）。テキスト全体に含まれるかで判定 |
| `incident_ttl_days` | int | `7` | ≥ 1 | DynamoDB の重複排除 TTL（日数） |
| `line_target_areas` | list[str] | `[]` | — | LINE通知するエリア名の前方一致リスト。空の場合は多摩区全体を対象とする |
| `place_mappings` | object | 必須 | — | エリア名・Mapion URL の解決設定（後述） |

---

### `place_mappings`

場所テキスト（`df[4]`）からエリア名と Mapion 地図 URL を解決するための設定。

| フィールド | 型 | 説明 |
|---|---|---|
| `base_url` | str | Mapion URL のベース（例: `http://www.mapion.co.jp/address/14135/`） |
| `fallback_area` | str | いずれのパターンにもマッチしない場合のエリア名（例: `多摩区`） |
| `chome_patterns` | list | 丁目番号が変化するエリアの正規表現パターン（後述） |
| `fixed_areas` | list | 固定 URL のエリア定義（後述） |

#### 解決アルゴリズム（優先順位）

1. `chome_patterns` を上から順に評価し、最初に正規表現がマッチしたエントリを使用する
2. マッチしない場合、`fixed_areas` を上から順に評価し、`place` が `key` で**前方一致**するエントリを使用する
3. どちらにもマッチしない場合、`fallback_area` と `base_url` を使用する

**重要:** `fixed_areas` の順序は前方一致の精度に影響する。より具体的なキー（`登戸新町`）は汎用的なキー（`登戸`）より**前に**定義すること。

---

### `place_mappings.chome_patterns`

丁目番号が数字で変化するエリア（例: 宿河原1丁目〜7丁目）のパターン定義。

| フィールド | 型 | 説明 |
|---|---|---|
| `pattern` | str | Python 正規表現。キャプチャグループ `([１-７])` 等で全角数字を捕捉する。起動時にコンパイル検証される |
| `name_template` | str | エリア名テンプレート。`{fw}` が全角数字に置換される |
| `url_path` | str | `base_url` に連結する URL パス。`{hw}` が半角数字に置換される |

**テンプレート変数:**

| 変数 | 意味 | 例 |
|---|---|---|
| `{fw}` | full-width（全角数字）: 正規表現のキャプチャグループそのまま | `１` |
| `{hw}` | half-width（半角数字）: `{fw}` を半角に変換したもの | `1` |

**設定例:**

```yaml
- pattern: "宿河原([１-７])丁目"
  name_template: "宿河原{fw}丁目"    # → "宿河原３丁目"
  url_path: "4:{hw}/"               # → "4:3/"
```

`place = "宿河原３丁目"` の場合:
- `pattern` にマッチ、キャプチャ = `３`（全角）
- `{fw}` → `３`、`{hw}` → `3`
- エリア名 = `宿河原３丁目`、URL = `http://www.mapion.co.jp/address/14135/4:3/`

---

### `place_mappings.fixed_areas`

丁目番号が変化しない固定エリアの定義。

| フィールド | 型 | 説明 |
|---|---|---|
| `key` | str | `place` フィールドとの前方一致キー |
| `area_name` | str | エリア名（区名プレフィックスを含む場合はそのまま、含まない場合は通知時に `ward` を付加） |
| `url_path` | str | `base_url` に連結する固定 URL パス |

**区名プレフィックスの付加ルール:**
- `area_name` が `ward`（例: `多摩区`）で始まっている場合 → そのまま使用
- 始まっていない場合 → `"{ward} {area_name}"` として表示

---

## 新しいエリアを追加する手順

### 丁目番号が変化するエリア（chome_patterns）

```yaml
chome_patterns:
  - pattern: "新エリア([１-９])丁目"
    name_template: "新エリア{fw}丁目"
    url_path: "<Mapion の丁目コード>:{hw}/"
```

Mapion の丁目コードは `http://www.mapion.co.jp/address/14135/` を参照して確認する。

### 固定エリア（fixed_areas）

```yaml
fixed_areas:
  - key: "新エリア名"
    area_name: "多摩区新エリア名"    # 区名を付ける場合
    url_path: "<Mapion のエリアコード>/"
```

**注意:** より具体的なキーを汎用的なキーより前に定義すること（前方一致の順序依存）。

---

## バリデーションエラー時の挙動

`config.yaml` の読み込みは Lambda 起動時（`_load_config()`）に行われる。
バリデーションエラーが発生した場合は `ValidationError` が送出され、Lambda は失敗として終了する。
エラーは CloudWatch Logs に構造化 JSON で記録され、DLQ に入る（`docs/spec/error-handling.md` 参照）。
