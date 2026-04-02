# AI-DLC State

> AI-Driven Development Lifecycle の進捗状況

## Current Phase: Phase 1 - Inception

<!-- 有効な値: Phase 1 - Inception | Phase 2 - Blueprint | Phase 3 - Construction | Phase 4 - Quality Gate | Complete -->

---

## 横断ルール

### 過信防止（Overconfidence Prevention）

- AIは「たぶんこうだろう」で進めず、不確実な点は人間に質問すること
- 特に以下の場面では確認を優先する:
  - 要件の解釈が複数あり得るとき
  - 技術選定でトレードオフがあるとき
  - スコープの境界が曖昧なとき

### 監査ログ（Audit Log）

- `aidlc-docs/audit.md` にはユーザーの発言を原文のまま記録すること（要約・言い換え禁止）
- 追記のみ。過去の記録を上書き・削除しない
- 各エントリにはフェーズ・ステップのコンテキストを含める

---

## Phase 1: Inception（構想）

- [ ] Step 0: コードベース分析（Codebase Analysis）※Brownfield のみ
  - 既存コードがある場合、コードベースを分析しスナップショットを生成する
  - Greenfield の場合はスキップ
  - 成果物: `codebase-snapshot.json`
- [ ] Step 1: 意図の明確化（Intent）
  - 人間がビジョンを語り、AIが対話で深掘りする
  - プロダクト動機タイプ（課題駆動型 / 意志駆動型）を判別する
  - 【課題駆動型】課題と解決策を分離すること（XY問題の検出）。具体的なシーン・数値・頻度に達するまで深掘りすること
  - 【意志駆動型】プロダクトの存在意義と提供価値を掘り下げ、コア体験が具体的なシーン・ユーザーの行動・感情を含むレベルに達するまで深掘りすること
  - 成果物: `inception/intent.md`
- [ ] Step 2: 機能探索（Exploration）
  - AIと人間が一緒に「どんな機能があるとよいか」をアイデア発散・探索する
  - 1件ずつ方向性を深掘りし、人間が「もうアイデアはない」と確認するまで続ける
  - 成果物: `inception/intent.md`（備考・未整理メモに追記）
- [ ] Step 3: 要件の具体化（Requirements）
  - AIが要件・ユーザーストーリー・MVPスコープを生成する
  - 要件間の矛盾を検出し、解消されるまで次に進まないこと
  - 成果物: `inception/requirements.md`, `inception/user-stories.md`
- [ ] Step 4: ビジュアライズ（Visualize）
  - 要件をもとにデザインファイル（.pen）を作成する
  - 画面一覧・遷移フロー・レイアウトをデザインで表現する
  - コード生成は行わない
  - 成果物: デザインファイル（.pen）
- [ ] Step 5: QAプラン（QA Plan）
  - デザインとユーザーストーリーから QA シナリオと QA Fixture を設計する
  - シナリオ YAML を `qa/scenarios/web/` に生成する
  - 成果物: `inception/qa-plan.md`, `qa/scenarios/web/*.yml`
- [ ] Step 6: 承認（Approval Gate）
  - 人間が全成果物（要件 + デザイン + QAプラン）を確認し承認する
  - 軽微な修正 → Gate 内で修正
  - QA シナリオの修正 → Step 5 に戻る
  - デザインの大幅変更 → Step 4 に戻る
  - 機能追加・再設計 → Step 1 に戻る
  - 承認 → Step 7 へ
- [ ] Step 7: Unit分割（Unit Decomposition）
  - 承認後、並列実行可能な単位に分割する
  - 依存関係を考慮して実行順序を決定する
  - 成果物: `inception/units.md`

## Phase 2: Blueprint（設計）

- [ ] Step 0: GitHub Projects v2 セットアップ
  - `config/github-projects.json` がプレースホルダーの場合、GitHub CLI で情報を取得し設定する
  - セットアップ済みの場合はスキップ
  - 成果物: `config/github-projects.json`（設定済み）

<!-- Unit ごとに Step 1〜3 を繰り返す。最初の Unit 着手時に inception/units.md から Unit 一覧を読み込み、
     以下のテンプレートを Unit 数分コピーして展開する（{unit-name} を実際の Unit 名に置換する）。

### Unit: voting-basic
- [ ] Step 1: Functional Design
- [ ] Step 2: Architecture
- [ ] Step 3: Bolt Plan

### Unit: voting-ranking
- [ ] Step 1: Functional Design
- [ ] Step 2: Architecture
- [ ] Step 3: Bolt Plan
-->

<!-- ↓ 最初の Unit 着手時に、このプレースホルダーを実際の Unit 名で置換し、Unit 数分コピーする ↓ -->

### Unit: {unit-name}

- [ ] Step 1: Functional Design（機能設計）
  - 技術に依存しない、純粋なビジネスロジックの設計を行う
  - ドメインモデル・ビジネスロジック・ビジネスルールを定義する
  - 成果物: `blueprint/functional-design.md`（全体設計）および `blueprint/units/{unit-name}/functional-design.md`（Unit設計）
- [ ] Step 2: Architecture（アーキテクチャ設計）
  - 技術スタック・構成図・API設計・データモデル（物理）・インフラ・ディレクトリ構成を設計する
  - NFR関連の意思決定（パフォーマンス、スケーラビリティ、セキュリティ等）を含む
  - 成果物: `blueprint/architecture.md`（全体設計）および `blueprint/units/{unit-name}/architecture.md`（Unit設計）
- [ ] Step 3: Bolt Plan（Bolt 分割計画）
  - Unit の設計成果物を実装可能な Bolt に分割し、依存順序と受入基準を定義する
  - Unit 単位で人間の承認を得てから次の Unit または Step 4 に進む
  - 成果物: `blueprint/units/{unit-name}/bolt-plan.md`

- [ ] Step 4: Blueprint Approval Gate（全体承認）
  - 全 Unit の設計成果物と Bolt Plan を横断的にレビューする
  - `config/project-adapter.md` を作成・確認する（Phase 3 の前提）
  - Unit 間の整合性を確認し、人間の承認を得てから Phase 3 に進む
  - 成果物: `config/project-adapter.md`

## Phase 3: Construction（実装）

- [ ] Step 1: Execution（実行）
  - Bolt ごとに `/run-bolt` を実行してプロダクションコードを構築する（1セッション1 Bolt）
  - 複数の ready Bolt がある場合、人間が別の Claude Code セッションを起動して並列実行する
  - ビルド成功 + 全テスト通過 + 動作確認を行う
  - 成果物: プロダクションコード + テストコード + `eval/i{intent}-eval-report.md`

- [ ] Step 2: Approval Gate（承認）
  - 人間が実装結果をレビューし承認する
  - → 方向性が違う → Phase 1 に戻る
  - → 設計変更 → Phase 2 に戻る
  - → 実装の修正 → Step 1 に戻り該当 Bolt を再実行
  - → OK → Phase 4 へ進む

## Phase 4: Quality Gate（品質関門）

- [ ] Step 1: Quality Check & Auto-Fix（品質チェック・自動修正）
  - AI agent がコード品質（重複検出含む）・セキュリティ・パフォーマンスを横断的にチェックする
  - テスト実行: 全テストが通ることを確認する
  - 自動修正可能な問題は修正し、品質レポートを生成する
  - 成果物: `quality-gate/report.md`（テスト結果を含む）
- [ ] Step 2: Approval Gate（Go/No-Go）
  - 人間が品質レポートをレビューし、Go/No-Go 判断する
