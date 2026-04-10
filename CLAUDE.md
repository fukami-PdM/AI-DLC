# AI-DLC プロジェクト設定

このリポジトリは **AI-DLC（AI-Driven Development Lifecycle）** のワークフローに従って開発を進めます。

---

## 基本原則

1. **AI が動き、人間がハンドルを握る** — 要件定義・設計・実装・テストは AI が担当し、人間はビジョン・承認・方向修正に集中する
2. **承認ゲートを必ず守る** — 各フェーズの終わりに人間の承認なしに次フェーズへ進まない
3. **過信防止** — 不確実な点は自律判断せず、必ず人間に確認する
4. **成果物は差分更新** — 既存ファイルを削除・再作成しない。編集で更新する

---

## セッション開始時の必須手順

**毎セッションの最初に必ずこの手順を実行すること。**

### ステップ 1: aidlc-docs/ の存在確認

```
aidlc-docs/aidlc-state.md が存在するか確認する
```

- **存在する** → ステップ 2 へ
- **存在しない** → 初回セットアップを実行する（下記参照）

### ステップ 2: 現在の進捗を把握する

`aidlc-docs/aidlc-state.md` を読み、以下を確認する:

- `Current Phase` — 現在のフェーズ
- 最後に `[x]` になったチェックボックス — 完了済みのステップ
- 次に実行すべきステップ

### ステップ 3: 対応するコマンドを実行する

現在フェーズに対応するスラッシュコマンドを使う（下記の「スラッシュコマンド一覧」参照）。

---

## 初回セットアップ（aidlc-docs/ が存在しない場合）

**このリポジトリのテンプレートを使って `aidlc-docs/` を初期化する。**

```bash
# テンプレートをプロジェクトにコピーする
cp -r "templates/ai-dlc docs/" aidlc-docs/
```

コピー後に確認すること:

```
aidlc-docs/
├── aidlc-state.md          ← Current Phase が "Phase 1 - Inception" になっているか確認
├── audit.md                ← 空の監査ログテンプレートか確認
├── codebase-snapshot.json  ← テンプレートのままでよい
├── config/
│   ├── github-projects.json
│   └── project-adapter.md
├── inception/
│   ├── intent.md
│   ├── requirements.md
│   ├── user-stories.md
│   └── units.md
├── blueprint/
│   ├── functional-design.md
│   ├── Architecture.md
│   └── units/
├── eval/
└── quality gate/
```

初期化完了後、`/inception` コマンドを実行してフェーズ 1 を開始する。

---

## スラッシュコマンド一覧

| コマンド | フェーズ | 内容 |
|---------|---------|------|
| `/inception` | Phase 1 | アイデア → 要件定義・ユーザーストーリー・Unit 分割 |
| `/blueprint` | Phase 2 | 要件 → 機能設計・アーキテクチャ・Bolt Plan |
| `/construction` | Phase 3 | Bolt Plan → プロダクションコード実装 |
| `/quality-gate` | Phase 4 | コード品質・セキュリティ検証・Go/No-Go 判断 |
| `/run-bolt` | Phase 3 内 | 単一 Bolt をパイプライン実行（スキル） |

**コマンドの詳細な手順は `.claude/commands/` 配下の各ファイルに記述されている。**  
コマンド実行時は必ずそのファイルの指示に従って行動すること。

---

## テンプレートパス一覧

コマンドファイル内でテンプレートを参照する際は、以下のパスを使用すること:

| テンプレート | パス |
|-------------|------|
| aidlc-state.md | `templates/ai-dlc docs/ai-dlc state.md` |
| audit.md | `templates/ai-dlc docs/audit.md` |
| codebase-snapshot.json | `templates/ai-dlc docs/codebase-snapshot.json` |
| intent.md | `templates/ai-dlc docs/inception/intent.md` |
| requirements.md | `templates/ai-dlc docs/inception/requirements.md` |
| user-stories.md | `templates/ai-dlc docs/inception/user-stories.md` |
| units.md | `templates/ai-dlc docs/inception/units.md` |
| functional-design.md | `templates/ai-dlc docs/blueprint/functional-design.md` |
| architecture.md | `templates/ai-dlc docs/blueprint/Architecture.md` |
| unit functional-design.md | `templates/ai-dlc docs/blueprint/units/unit-functional-design.md` |
| unit architecture.md | `templates/ai-dlc docs/blueprint/units/unit-architecture.md` |
| bolt-plan.md | `templates/ai-dlc docs/blueprint/units/Bolt Plan.md` |
| project-adapter.md | `templates/ai-dlc docs/config/project-adapter.md` |
| github-projects.json | `templates/ai-dlc docs/config/github-projects.json` |
| eval-report.md | `templates/ai-dlc docs/eval/i000-eval-report.md` |
| quality-gate/report.md | `templates/ai-dlc docs/quality gate/report.md` |
| run-bolt SKILL.md | `templates/skills/run-bolt/SKILL.md` |

---

## ドキュメント構造

すべての成果物は `aidlc-docs/` 配下に保存する。

```
aidlc-docs/
├── aidlc-state.md          # 全フェーズの進捗チェックボックス（常に最新に保つ）
├── audit.md                # ユーザー発言の原文記録（要約・言い換え禁止、追記のみ）
├── codebase-snapshot.json  # コードベース分析結果（Brownfield のみ）
├── config/
│   ├── github-projects.json   # GitHub Projects v2 設定
│   └── project-adapter.md     # Bolt パイプライン設定
├── inception/
│   ├── intent.md
│   ├── requirements.md
│   ├── user-stories.md
│   └── units.md
├── blueprint/
│   ├── functional-design.md
│   ├── architecture.md
│   └── units/
│       └── {unit-name}/
│           ├── functional-design.md
│           ├── architecture.md
│           └── bolt-plan.md
├── eval/
│   ├── i{intent}-eval-report.md
│   └── harness-changelog.md
└── quality gate/
    └── report.md
```

---

## 横断的なルール

### audit.md への記録

- ユーザーの発言は**原文のまま**記録する（要約・言い換え禁止）
- 追記のみ。過去の記録を上書き・削除しない
- 各エントリにはフェーズ・ステップのコンテキストを含める

### aidlc-state.md の更新

- 各ステップの完了後に、該当チェックボックスを `[ ]` → `[x]` に更新する
- `Current Phase` は常に現在のフェーズを反映させる
- フィードバックで前のステップに戻る場合、戻り先以降のチェックボックスを `[x]` → `[ ]` に戻す

### 成果物の更新ルール

- 既存ファイルを削除・再作成しない
- 変更は差分更新（既存ファイルの編集）で行う
- 下流の成果物は再実行時に整合性を確認して必要に応じて更新する

### 承認ゲートの扱い

- Approval Gate では人間の明示的な承認を待つ
- 「OK」「承認」「進めて」などの明確な承認なしに次フェーズへ進まない
- フィードバックがある場合は `Playbooks/overview.md` のフィードバックループ手順に従う

### 過信防止

以下の場面では自律判断せず、必ず人間に確認する:

- 要件の解釈が複数あり得るとき
- 技術選定でトレードオフがあるとき
- スコープの境界が曖昧なとき

---

## フェーズ別の概要

詳細な手順は各コマンドファイルおよび `Playbooks/` を参照すること。

| フェーズ | コマンド | 入力 | 主な成果物 |
|---------|---------|------|----------|
| Phase 1: Inception | `/inception` | 人間のビジョン | intent.md, requirements.md, user-stories.md, units.md |
| Phase 2: Blueprint | `/blueprint` | Phase 1 成果物 | functional-design.md, architecture.md, bolt-plan.md |
| Phase 3: Construction | `/construction` + `/run-bolt` | bolt-plan.md | プロダクションコード, eval-report.md |
| Phase 4: Quality Gate | `/quality-gate` | プロダクションコード | quality-gate/report.md |

---

## 参照ドキュメント

| ドキュメント | 内容 |
|-------------|------|
| `Playbooks/overview.md` | 全フェーズ・ステップの Input/Output 一覧、フィードバックループ手順 |
| `Playbooks/phase1-inception.md` | Phase 1 の詳細手順 |
| `Playbooks/phase2-blueprint.md` | Phase 2 の詳細手順 |
| `Playbooks/phase3-construction.md` | Phase 3 の詳細手順 |
| `Playbooks/phase4-quality-gate.md` | Phase 4 の詳細手順 |
| `References/context-budget.md` | Implementer プロンプトのコンテキスト予算ルール |
| `References/harness-eval.md` | ハーネス評価の基準 |
| `References/qa-scenario-format.md` | QA シナリオ YAML の書式 |
| `philosophy.md` | AI-DLC の哲学・設計思想 |
