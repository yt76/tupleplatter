# Tuple Platter 開発マニュアル（コーディングエージェント向けドラフト）

> **Note**: 本ドキュメントは [README.md](./README.md) の構想に基づくドラフトです。HTTP API のエンドポイントや認証方式などの詳細はサーバー実装に合わせて調整してください。

このマニュアルは、Tuple Platter（Tuple Space サーバー）をデータストアとして利用するアプリケーションを開発する**コーディングエージェント**向けの仕様書兼ガイドラインです。
アプリケーションを実装する際は、以下のプロトコルおよびルールに従ってください。

---

## 1. 基本概念

- **スペース（Space）**: データを保存・共有する独立した空間です。1つのスペースは `https://<hostname>/<space_path>` の URL で識別されます。
  - デフォルトスペース: `https://<hostname>/default`
  - 例 (`foo.db`): `https://<hostname>/foo`
  - 例 (`bar/baz.db`): `https://<hostname>/bar/baz`
  - 予約語: API パス（`/<space_path>/api/...`）との衝突を防ぐため、`api` はパスセグメントとして使用できません（例: `api`、`bar/api`、`api/baz` は不可）。
- **タプル（Tuple）**: スペースに保存される JSON オブジェクト（Key-Value 形式の名前付きタプル）です。
  - 例: `{"type": "todo", "status": "open", "assignee": "alice", "payload": {"title": "牛乳を買う"}}`
- **パターン（Pattern）**: タプルを検索・取得するための条件となる JSON オブジェクトです。指定したすべてのキーと値が一致するタプルにマッチします（部分一致）。

---

## 2. データ形式とパターンマッチングのルール

1. **タプルは JSON オブジェクト（Key-Value）**として表現します。
   - 検索・絞り込みに使いたい属性（`type`、`status`、`assignee` など）は、トップレベルのキーとしてスカラー値（文字列、数値、真偽値、`null`）で配置してください。
   - 検索に使わない詳細なペイロード（複数フィールドを持つデータやリスト）は、トップレベルのキー（例: `payload` や `data`）の値として JSON オブジェクト（`dict`）にまとめてください。
2. **`value` が `dict`（ネストした JSON オブジェクト）または `array`（配列）の場合はパターンマッチング対象外**です。
   - オブジェクト内部のキーや配列要素による検索はできません。検索に使いたい属性は必ずトップレベルのスカラー値として置いてください。
   - 良い例: `{"type": "todo", "status": "open", "assignee": "alice", "payload": {"title": "買い物", "due": "2026-12-01"}}`
   - 悪い例: `{"type": "todo", "payload": {"status": "open", "assignee": "alice", "title": "買い物"}}`（`status` や `assignee` で絞り込めない）
3. **部分一致によるマッチング**:
   - パターンには絞り込みたいキーだけを指定します。指定しなかったキーは任意の値（ワイルドカード）として扱われます。
   - 例: パターン `{"type": "todo", "status": "open"}` は、`type` が `"todo"` かつ `status` が `"open"` であるすべてのタプルにマッチします（`assignee` や `payload` の内容は問いません）。

---

## 3. HTTP API プロトコル（暫定仕様）

すべてのリクエストは対象スペースのパス `<space_path>`（例: `foo`、`bar/baz`、`default`）を含む URL（`https://<hostname>/<space_path>/api/v1/...`）に対して行います。

### 認証（アクセスコントロール）

アクセスコントロールはスペース単位で行われます。スペースが認証を要求する場合、以下のヘッダを付与してください（環境変数等から読み込み、コードにハードコードしないこと）。

```http
Authorization: Bearer <SPACE_TOKEN>
Content-Type: application/json
```

### 基本操作

| 操作 | 意味 | エンドポイント | 挙動 |
| :--- | :--- | :--- | :--- |
| **out** | 書き込む | `POST /<space_path>/api/v1/tuples` | タプルを1件スペースに追加する（重複可） |
| **rd** | 読む（残す） | `POST /<space_path>/api/v1/tuples/read` | パターンに一致するタプルをスペースから消さずに取得する |
| **in** | 取り出す（消す） | `POST /<space_path>/api/v1/tuples/take` | パターンに一致するタプルを1件取得し、スペースから**削除**する（アトミック） |

#### リクエスト / レスポンス例（スペースが `foo` の場合）

**1. タプルの書き込み (`out`)**
```http
POST /foo/api/v1/tuples
{"tuple": {"type": "todo", "status": "open", "assignee": "alice", "payload": {"title": "買い物"}}}
```
レスポンス (`201 Created`):
```json
{
  "id": "tup_01J...",
  "tuple": {"type": "todo", "status": "open", "assignee": "alice", "payload": {"title": "買い物"}}
}
```

**2. タプルの読み取り (`rd`)**
```http
POST /foo/api/v1/tuples/read
{"pattern": {"type": "todo", "status": "open"}, "limit": 50}
```
レスポンス (`200 OK`):
```json
{
  "tuples": [
    {
      "id": "tup_01J...",
      "tuple": {"type": "todo", "status": "open", "assignee": "alice", "payload": {"title": "買い物"}}
    }
  ]
}
```

**3. タプルの取り出し (`in`)**
```http
POST /foo/api/v1/tuples/take
{"pattern": {"type": "todo", "status": "open"}, "timeout_ms": 0}
```
- 一致するタプルがあれば `200 OK` で1件返し、そのタプルをスペースから削除します。
- 一致するタプルがない場合、`timeout_ms` が `0` なら即座に `404 Not Found`（または空結果）を返します。

### スキーマドキュメント操作

同じスペースを共有する他アプリや AI エージェントがデータ構造を理解できるよう、アプリごとの `TUPLE_SCHEMA.md` をスペース内に保存・参照できます。`<name>` にはアプリ名などの識別子（例: `todo-app`、英数字・ハイフン・アンダースコア推奨）を指定します。

| 操作 | エンドポイント | 挙動 |
| :--- | :--- | :--- |
| **登録・更新** | `PUT /<space_path>/api/v1/schemas/<name>` | 生の Markdown 本文を登録・上書きする（`Content-Type: text/markdown; charset=utf-8`） |
| **一覧取得** | `GET /<space_path>/api/v1/schemas` | 登録済みスキーマの一覧（名前・更新日時）を JSON で返す。`Accept: text/markdown` 指定時は全スキーマを結合した Markdown を返す |
| **個別取得** | `GET /<space_path>/api/v1/schemas/<name>` | 指定したスキーマの Markdown 本文を `text/markdown` で返す |
| **削除** | `DELETE /<space_path>/api/v1/schemas/<name>` | 指定したスキーマを削除する |

#### リクエスト / レスポンス例（スペースが `foo`、アプリ名が `todo-app` の場合）

**1. スキーマの登録・更新 (`PUT`)**
```http
PUT /foo/api/v1/schemas/todo-app
Authorization: Bearer <SPACE_TOKEN>
Content-Type: text/markdown; charset=utf-8

# Todo App Schema
...
```
レスポンス (`200 OK` または `201 Created`):
```json
{
  "name": "todo-app",
  "updated_at": "2026-11-27T13:00:00Z"
}
```

**2. スキーマ一覧の取得 (`GET`)**
```http
GET /foo/api/v1/schemas
Authorization: Bearer <SPACE_TOKEN>
```
レスポンス (`200 OK`):
```json
{
  "schemas": [
    {
      "name": "todo-app",
      "updated_at": "2026-11-27T13:00:00Z"
    }
  ]
}
```

---

## 4. アプリ実装時のルール

1. **接続先とトークンは設定可能にする**
   - スペースの URL（例: `TUPLE_PLATTER_URL`）と認証トークン（例: `TUPLE_PLATTER_TOKEN`）は環境変数または設定ファイルから読み込むように実装してください。
2. **データ更新の考え方**
   - タプルは原則イミュータブルとして扱います。状態を変更したい場合は `in` で古いタプルを取り出し、新しい状態のタプルを `out` で書き込んでください。
3. **競合への配慮**
   - 複数アプリが同じタプルを処理する可能性がある場合（タスクキューなど）は、必ず `rd` ではなく `in` を使って排他的に取り出してください。
4. **種別キーの推奨**
   - 1つのスペースを複数アプリで共有する場合に備え、タプルには用途を示すキー（例: `"type": "todo"` や `"kind": "log"`）を含めることを推奨します。

---

## 5. 【必須】スキーマドキュメントの出力・アップロードルール

開発したアプリケーションが同じ Tuple Space を利用する**他のアプリや開発者、AI エージェントとデータを共有できるようにするため**、実装完了時に以下を行ってください。

1. プロジェクトルートへ `TUPLE_SCHEMA.md` を出力・更新する。
2. 接続先スペースとトークンが設定されている場合は、`PUT /<space_path>/api/v1/schemas/<app_name>` へ `TUPLE_SCHEMA.md` をアップロードする（またはアプリ起動時・デプロイ時にアップロードする仕組みを用意する）。

```bash
curl -X PUT "$TUPLE_PLATTER_URL/api/v1/schemas/todo-app" \
  -H "Authorization: Bearer $TUPLE_PLATTER_TOKEN" \
  -H "Content-Type: text/markdown; charset=utf-8" \
  --data-binary @TUPLE_SCHEMA.md
```

スキーマドキュメントには、そのアプリが読み書きする**すべてのタプル型**について、以下の項目を漏れなく記載してください。

1. **利用するスペース名 / 用途**
2. **トップレベルキーの定義**（キー名、型、値の意味、マッチング対象かどうか）
3. **ペイロード（`dict`）内のフィールド定義**（キー名、型、必須/任意、説明）
4. **操作種別**（このアプリが `out` / `rd` / `in` のどれを行うか）
5. **具体的な JSON サンプル**

### スキーマドキュメント記載例

```markdown
### タプル型: `todo`

- **用途**: ユーザーのタスク管理
- **このアプリの操作**: `out`, `rd`, `in`
- **トップレベルキー**:
  - `type` (`string`, 必須, マッチング対象): 固定値 `"todo"`
  - `status` (`string`, 必須, マッチング対象): `"open"` | `"done"`
  - `assignee` (`string`, 必須, マッチング対象): 担当者名
  - `payload` (`object`, 必須, マッチング対象外):
    - `title` (`string`, 必須): タスクのタイトル
    - `due` (`string`, 任意): 期限 (ISO 8601)
- **サンプル**:
  ```json
  {
    "type": "todo",
    "status": "open",
    "assignee": "alice",
    "payload": {
      "title": "買い物",
      "due": "2026-12-01"
    }
  }
  ```
```
