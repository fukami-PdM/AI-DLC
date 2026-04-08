# Phase 3: Construction プレイブック

> Blueprint で策定した設計・計画に従い、AI agent が完全自律でプロダクションコードを構築する。人間はコードを書かない。

## 前提条件

- Phase 2 (Blueprint) が完了していること（aidlc-docs/aidlc-state.md で確認）
- Bolt Plan が承認され、GitHub Issue として登録されていること
- `aidlc-docs/blueprint/` 配下の全設計成果物が存在すること

## スキルのセットアップ（自動）

Phase 3 開始時に、Bolt パイプラインに必要なスキルを自動配置する。テンプレートは `aidlc-workflows/templates/skills/` に格納されている。

```bash
mkdir -p .claude/skills
cp -r aidlc-workflows/templates/skills/. .claude/skills/
```

> **AI への指示**: Phase 3 に入ったとき、テンプレートとデプロイ先のスキルファイルを比較し、差分がある場合は上記コマンドで再デプロイすること。`.claude/skills/run-bolt/SKILL.md` が存在しない場合も同様に実行する。
>
> ```bash
> # 差分チェック（差分があれば再デプロイ）
> diff -rq aidlc-workflows/templates/skills/ .claude/skills/ 2>/dev/null || cp -r aidlc-workflows/templates/skills/. .claude/skills/
> ```

## GitHub Projects v2 設定

GitHub Projects v2 を操作する際は、`aidlc-docs/config/github-projects.json` を読み込み、変数として使用する。変数の対応は `phase2-blueprint.md` > GitHub Projects v2 設定 を参照。

## フェーズ開始時の成果物読み込み

Phase 3 の各ステップに入る前に、以下の成果物を一括で読み込む。これらはオーケストレーター（Phase 3 全体を制御する AI）が全体像を把握するために使用する。個々の Bolt 実行時には、Prepare エージェントが Context Budget に従って必要な部分のみを抽出し、Implementer プロンプトに反映する。

- `aidlc-docs/inception/intent.md`
- `aidlc-docs/inception/requirements.md`
- `aidlc-docs/inception/user-stories.md`
- `aidlc-docs/inception/units.md`
- `aidlc-docs/inception/` 配下のデザインファイル（.pen）（存在する場合）
- `aidlc-docs/blueprint/functional-design.md`
- `aidlc-docs/blueprint/architecture.md`
- `aidlc-docs/blueprint/units/{unit-name}/functional-design.md`
- `aidlc-docs/blueprint/units/{unit-name}/architecture.md`
- `aidlc-docs/blueprint/units/{unit-name}/bolt-plan.md`

---

## Step 1: Execution（実行）

> Bolt Plan に従い、Bolt ごとに Bolt パイプラインを実行してプロダクションコードを構築する。

### 設計思想

> パイプラインの設計思想・詳細手順は `.claude/skills/run-bolt/SKILL.md` を参照。

### 前提: Project Adapter

Bolt パイプラインはプロジェクト固有の設定を `aidlc-docs/config/project-adapter.md` から読み込む。以下が定義されていること:

- **Workstreams**: 実装エージェント（Agent 定義、Trigger paths、Owned paths、Skills）
- **Gate Commands**: CI 相当のチェックコマンド
- **Review Configuration**: AI レビューの設定（オプション）

テンプレート: `aidlc-workflows/templates/aidlc-docs/config/project-adapter.md`

### AIの行動

> **1セッション1 Bolt**: 各 Bolt の実行は `/run-bolt` スキルを通じて1セッション1 Bolt で行う。複数の ready Bolt が同時に存在する場合は、人間が別の Claude Code セッションを起動して並列実行する。AI は1つの Bolt を実行し、他の ready Bolt の存在を人間に通知する。

1. **Bolt Plan を読み込み、実行ループを開始する**
   - `blueprint/units/{unit-name}/bolt-plan.md` を読み込む
   - `aidlc-docs/config/project-adapter.md` を読み込む
   - `aidlc-docs/eval/` ディレクトリが存在しない場合は作成する: `mkdir -p aidlc-docs/eval`
   - **以下のループを `ready` Bolt がなくなるまで繰り返す:**

2. **ループ: ready Bolt の検出と実行**

   ```
   WHILE true:
     a. GitHub Projects v2 API を実行して ready 状態の Bolt を全件取得する
        ⚠️ 必ず gh project item-list を再実行すること。前回イテレーションの取得結果や
        ループ開始時の情報を使い回してはならない。他セッションの Ship cleanup により
        下流 Bolt が新たに ready 化されている可能性がある。
     b. ready Bolt が 0 件 → ループ終了、下記の手順 4「全 Bolt 完了後の確認」へ
     c. ready Bolt が 1 件 → その Bolt の Lock を取得し、Bolt パイプラインを実行する
     d. ready Bolt が 2 件以上 → 1件目の Lock を取得し Bolt パイプラインを実行する。
        残りの ready Bolt を人間に一覧表示し、「並列実行する場合は別の Claude Code
        セッションで `/run-bolt` を実行してください」と案内する
     e. Bolt パイプライン完了後（Ship の cleanup で下流 Bolt が ready 化される）、
        ループ先頭に戻る（ステップ a で最新状態を再取得する）
   ```

   - **ready Bolt の検出**:

     ```bash
     # フィールド名は gh project item-list の出力形式に合わせること（github-projects.json の定義名とは大文字小文字が異なる場合がある）
     gh project item-list $PROJECT_NUMBER --owner $PROJECT_ORG --format json --limit 500 \
       | jq --arg intent "I{intent}" '[.items[] | select(.boltStatus == "ready" and .intent == $intent and .content.type == "Issue")] | [.[] | {number: .content.number, title: .content.title, url: .content.url, intent: .intent, bolt: .bolt}]'
     ```

   - **並列実行の方法**: 並列実行は人間が複数の Claude Code セッションを起動して行う。各セッションが `/run-bolt` で1 Bolt を実行する。各 Bolt は独自の worktree・ブランチで動作するため競合しない。Lock 機構により同一 Bolt が重複実行されることはない。

   - **Bolt パイプライン**: `/run-bolt` スキルで実行する。手順の詳細は `.claude/skills/run-bolt/SKILL.md` を参照
   - 失敗時: コンテキストにエラーログを追加して再生成（max 1回）。2回失敗 → エスカレーション

   - **ハーネス評価**: Bolt 単位の軽量チェック（Ship 直後）と Intent 全量分析（全 Bolt 完了後）を実行する。詳細は `aidlc-workflows/references/harness-eval.md` を参照

3. **エスカレーション Bolt の扱い**
   - 2回失敗でエスカレーションされた Bolt は `blocked` に戻す
   - その下流 Bolt も `blocked` のままとなる
   - エスカレーション一覧をループ終了時にまとめて報告する

4. **全 Bolt 完了後の確認**（ループ終了後）
   - GitHub Projects v2 で全 Bolt の Status を確認する
   - エスカレーションされた Bolt がある場合、一覧と影響を受ける下流 Bolt を人間に提示する
   - ビルドを実行し、成功することを確認する
   - 全テストを実行し、パスすることを確認する
   - アプリケーションが起動・動作することを確認する

### 完了条件

- 全 Bolt が完了またはエスカレーション済みである（GitHub Projects v2 で全 Bolt の Status が `done` または `blocked`（エスカレーション））
- エスカレーション Bolt がある場合:
  - エスカレーション一覧と影響範囲を人間に提示し、以下のいずれかの判断を得ている:
    - **続行**: エスカレーション Bolt は Phase 3 Approval Gate で扱う（残存課題として報告）
    - **再試行**: 人間の指示に基づき該当 Bolt を修正して再実行する
    - **スコープ除外**: 該当 Bolt を今回のスコープから除外する
- ビルドが成功している
- 全テストがパスしている
- アプリケーションが動作する状態になった
- ハーネス評価レポートが出力されている
- `aidlc-docs/aidlc-state.md` の Step 1 チェックボックスを `[x]` にする

---

## Step 2: Approval Gate（承認）

> **承認ゲート**: 実装結果を人間がレビューし、承認する。

### AIの行動

1. **実装結果を提示する**
   - アプリケーションの起動方法を伝える
   - 実装サマリーを提示する:
     - 実装した機能の概要
     - user-stories.md の受入基準の充足状況
     - テスト結果（パス率・カバレッジ）
   - 「実装結果をレビューしてください。フィードバックをお聞かせください」と伝える

2. **フィードバックに基づき次のアクションを決定する**

   フィードバックルートの詳細は `overview.md` > Phase 3 を参照。戻る場合は共通のフィードバックループ手順に従う。

### 完了条件

- 人間が実装結果を承認した
- `aidlc-docs/aidlc-state.md` の Step 2 チェックボックスを `[x]` にする
- `aidlc-docs/aidlc-state.md` の `Current Phase` を `Phase 4 - Quality Gate` に更新する
