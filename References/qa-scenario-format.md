# QA シナリオファイル仕様

> QAシナリオをYAML形式で宣言的に定義するための仕様。特定のブラウザ自動化ツールに依存しない。

## ファイル配置

```
qa/scenarios/web/{unit-name}/{scenario-name}.yml
```

## トップレベル構造

```yaml
version: "1.0"
name: "Login flow test"
description: "ログインフローのE2Eテスト"
app: "web"
tags: ["auth", "critical"]

preconditions:
  auth: false
  auth_state: ""
  viewport: "1920x1080"
  routes: []

evidence:
  screenshots: true
  har: false
  video: false
  network: true

steps:
  - id: "step-01"
    action: "goto"
    target: "/login"
    description: "ログインページに遷移する"
    assertions:
      - type: "snapshot-contains"
        value: "Sign In"
    evidence:
      screenshot: true
```

## フィールド仕様

### メタデータ

| フィールド    | 必須 | 型       | 説明                                   |
| ------------- | ---- | -------- | -------------------------------------- |
| `version`     | Yes  | string   | シナリオ形式バージョン（現在 `"1.0"`） |
| `name`        | Yes  | string   | シナリオ名（英語）                     |
| `description` | Yes  | string   | シナリオの説明                         |
| `app`         | Yes  | string   | 対象アプリ名（`web`）                  |
| `tags`        | No   | string[] | 分類タグ                               |

### preconditions

| フィールド   | 必須 | 型       | デフォルト  | 説明                      |
| ------------ | ---- | -------- | ----------- | ------------------------- |
| `auth`       | No   | boolean  | false       | 認証済み状態が必要か      |
| `auth_state` | No   | string   | ""          | 認証状態ファイルパス      |
| `viewport`   | No   | string   | "1920x1080" | ビューポートサイズ（WxH） |
| `routes`     | No   | object[] | []          | APIモック定義             |

#### routes の定義

```yaml
routes:
  - pattern: "**/stan.v1.UserService/GetUser"
    response:
      status: 200
      body: '{"user": {"id": "1", "name": "Test User"}}'
      content_type: "application/json"
  - pattern: "**/stan.v1.EventService/ListEvents"
    response:
      status: 500
```

### evidence（グローバル設定）

| フィールド    | 必須 | 型      | デフォルト | 説明                         |
| ------------- | ---- | ------- | ---------- | ---------------------------- |
| `screenshots` | No   | boolean | true       | 各 assertion で screenshot   |
| `har`         | No   | boolean | false      | ネットワーク詳細録画         |
| `video`       | No   | boolean | false      | 操作の動画録画               |
| `network`     | No   | boolean | true       | ネットワークリクエストの監視 |

### steps

| フィールド        | 必須  | 型       | 説明                               |
| ----------------- | ----- | -------- | ---------------------------------- |
| `id`              | Yes   | string   | ステップID（`step-01` 形式）       |
| `action`          | Yes   | string   | アクション型（下記参照）           |
| `target`          | Yes\* | string   | アクション対象（\*action による）  |
| `value`           | No    | string   | 入力値（fill, type, select 用）    |
| `description`     | Yes   | string   | ステップの説明                     |
| `target-selector` | No    | string   | セマンティックセレクタ（下記参照） |
| `assertions`      | No    | object[] | アクション後の検証                 |
| `evidence`        | No    | object   | ステップ固有のエビデンス設定       |

## アクション型

| アクション   | target      | value        | 説明                                   |
| ------------ | ----------- | ------------ | -------------------------------------- |
| `goto`       | URLパス     | -            | ページ遷移（相対パスはbase_urlに結合） |
| `click`      | 要素ref     | -            | 要素クリック                           |
| `dblclick`   | 要素ref     | -            | 要素ダブルクリック                     |
| `fill`       | 要素ref     | 入力値       | 入力フィールドに値を設定               |
| `type`       | -           | 入力テキスト | テキストをタイプ                       |
| `press`      | -           | キー名       | キーを押下（Enter, ArrowDown等）       |
| `select`     | 要素ref     | オプション値 | セレクトボックスの選択                 |
| `hover`      | 要素ref     | -            | 要素ホバー                             |
| `check`      | 要素ref     | -            | チェックボックスON                     |
| `uncheck`    | 要素ref     | -            | チェックボックスOFF                    |
| `upload`     | 要素ref     | ファイルパス | ファイルアップロード                   |
| `wait`       | 条件        | -            | 条件待機（テキスト出現等）             |
| `eval`       | JS式        | -            | JavaScript実行                         |
| `snapshot`   | -           | -            | スナップショット取得（要素ref確認用）  |
| `screenshot` | -           | ファイル名   | スクリーンショット取得                 |
| `assert`     | -           | -            | アクションなし（assertionsのみ実行）   |
| `api-check`  | URLパターン | -            | ネットワークリクエスト検証             |
| `go-back`    | -           | -            | ブラウザバック                         |
| `go-forward` | -           | -            | ブラウザフォワード                     |
| `reload`     | -           | -            | ページリロード                         |

## アサーション型

| 型                      | パラメータ      | 説明                                 |
| ----------------------- | --------------- | ------------------------------------ |
| `snapshot-contains`     | `value`         | snapshotテキストに値が含まれる       |
| `snapshot-not-contains` | `value`         | snapshotテキストに値が含まれない     |
| `url-matches`           | `value`         | 現在URLがパターンに一致する          |
| `url-not-matches`       | `value`         | 現在URLがパターンに一致しない        |
| `console-no-errors`     | -               | consoleにエラーがない                |
| `console-contains`      | `value`         | consoleに指定メッセージがある        |
| `network-status`        | `url`, `status` | リクエストが期待ステータスを返す     |
| `network-body-contains` | `url`, `value`  | レスポンスボディに値が含まれる       |
| `element-visible`       | `value`         | snapshot内に指定テキストの要素がある |
| `element-not-visible`   | `value`         | snapshot内に指定テキストの要素がない |
| `cookie-exists`         | `value`         | 指定名のcookieが存在する             |
| `localstorage-value`    | `key`, `value`  | localStorageのキーが期待値を持つ     |

## セマンティックセレクタ（target-selector）

`role:name` 形式でアクセシビリティロールと名前で要素を特定する。ツール固有の要素ref（@e1 等）はセッションごとに変わるため、シナリオでは target-selector で宣言的に指定し、実行時にツールが解決する。

```yaml
steps:
  - id: "step-02"
    action: "fill"
    target-selector: "textbox:Email" # role=textbox, name=Email
    value: "test@example.com"
    description: "メールアドレスを入力する"

  - id: "step-03"
    action: "click"
    target-selector: "button:Sign In" # role=button, name=Sign In
    description: "ログインボタンをクリックする"
```

**実行時の解決手順:**

1. ブラウザのアクセシビリティスナップショットを取得する
2. `target-selector` の role と name でスナップショット内の要素を検索する
3. 一致する要素の ref を使ってアクションを実行する
4. 一致する要素が見つからない場合はステップをFAILとする

### サポートするセレクタ形式

```
textbox:Email              → role=textbox, name=Email
button:Sign In             → role=button, name=Sign In
link:Dashboard             → role=link, name=Dashboard
heading:Welcome            → role=heading, name=Welcome
checkbox:Remember me       → role=checkbox, name=Remember me
combobox:Country           → role=combobox, name=Country
```

## 完全なシナリオ例

```yaml
version: "1.0"
name: "Login flow E2E test"
description: "メールとパスワードによるログインフローの正常系テスト"
app: "web"
tags: ["auth", "critical", "e2e"]

preconditions:
  auth: false
  viewport: "1920x1080"

evidence:
  screenshots: true
  har: true
  network: true

steps:
  - id: "step-01"
    action: "goto"
    target: "/login"
    description: "ログインページに遷移する"
    assertions:
      - type: "url-matches"
        value: "/login"
      - type: "element-visible"
        value: "Sign In"
    evidence:
      screenshot: true

  - id: "step-02"
    action: "fill"
    target-selector: "textbox:Email"
    value: "test@example.com"
    description: "メールアドレスを入力する"

  - id: "step-03"
    action: "fill"
    target-selector: "textbox:Password"
    value: "password123"
    description: "パスワードを入力する"

  - id: "step-04"
    action: "click"
    target-selector: "button:Sign In"
    description: "ログインボタンをクリックする"
    assertions:
      - type: "url-matches"
        value: "/dashboard"
      - type: "element-visible"
        value: "Welcome"
      - type: "console-no-errors"
      - type: "network-status"
        url: "**/stan.v1.UserService/**"
        status: 200
    evidence:
      screenshot: true
```
