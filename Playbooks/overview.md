# AI-DLC Workflow Overview

> 全フェーズ・ステップの Input / Output と流れの一覧

## フェーズ全体の流れ

```
Phase 1: Inception → Phase 2: Blueprint → Phase 3: Construction → Phase 4: Quality Gate
```

## 用語

- **確認**: 中間ステップで方向性をチェックすること。修正すればそのまま続行する
- **承認**: Approval Gate でフェーズ全体を判断すること。次フェーズに進む / 戻るを決定する

## 横断的な仕組み

- `aidlc-docs/aidlc-state.md`: 全フェーズ・ステップの進捗チェックボックス。Current Phase を常に更新する
- `aidlc-docs/audit.md`: ユーザーの発言を原文のまま記録する（要約・言い換え禁止、追記のみ）
- `aidlc-docs/config/project-adapter.md`: Bolt パイプラインのプロジェクト固有設定（Workstreams, Gate Commands, Review Configuration）。テンプレート: `aidlc-workflows/templates/aidlc-docs/config/project-adapter.md`

### 過信防止（Overconfidence Prevention）

- AIは「たぶんこうだろう」で進めず、不確実な点は人間に質問すること
- 特に以下の場面では確認を優先する:
  - 要件の解釈が複数あり得るとき
  - 技術選定でトレードオフがあるとき
  - スコープの境界が曖昧なとき

### フィードバックループ手順（共通）

Approval Gate でフィードバックにより前のステップ/フェーズに戻る場合、以下の手順を必ず実行する。

1. **audit.md に記録する**
   - フィードバック内容（人間の発言を原文のまま）
   - 戻り先（Phase X Step Y）
   - 戻る理由の要約

2. **aidlc-state.md を更新する**
   - `Current Phase` を戻り先のフェーズに更新する
   - 戻り先ステップ以降のチェックボックスを `[ ]` に戻す（それより前の完了済みステップはそのまま）
   - 例: Phase 2 Step 2 に戻る場合 → Step 2〜3 を `[ ]` に、Step 1 は `[x]` のまま

3. **成果物を更新する（上書き、新規作成はしない）**
   - 戻り先ステップの成果物を修正する。既存ファイルを編集し、変更箇所が分かるようにする
   - 下流の成果物（戻り先より後のステップで生成したもの）は、再実行時に整合性を確認して必要に応じて更新する
   - 成果物を削除・再作成しない。差分更新を原則とする

4. **再実行する**
   - 戻り先のステップから順に再実行する
   - 各ステップの完了条件を再度満たしたらチェックボックスを `[x]` にする
   - 最終的に元の Approval Gate まで戻り、再度承認を求める

### 全ステップ共通: 記録ルール

各ステップの完了時に `aidlc-docs/audit.md` の該当セクションに以下を記録する:

- ユーザーの発言（原文のまま。要約・言い換え禁止）
- AIの発言（要約でよい）
- 決定事項と生成した成果物

### 全ステップ共通: 自己レビューと完了条件

各ステップには「自己レビューチェックリスト」がある。完了条件は以下の3点に統一する:

1. 自己レビューチェックリストが全て合格している
2. 人間が内容を確認した（承認ステップでは承認）
3. `aidlc-state.md` の該当チェックボックスを `[x]` にする

---

## Phase 1: Inception（構想）

> 人間の漠然としたアイデアを、実行可能な要件に変換し、デザインで可視化し、実装単位に分割する

| Step                                     | Input                                | Output                                                       | 人間の関与 |
| ---------------------------------------- | ------------------------------------ | ------------------------------------------------------------ | ---------- |
| 0: Codebase Analysis（コードベース分析） | 既存コード（Brownfield のみ）        | `codebase-snapshot.json`                                     | 確認       |
| 1: Intent（意図の明確化）                | 人間の対話（+ スナップショット）     | `inception/intent.md`                                        | 確認       |
| 2: Exploration（機能探索）               | `intent.md`                          | `intent.md`（備考・未整理メモに追記）                        | 確認       |
| 3: Requirements（要件の具体化）          | `intent.md`                          | `inception/requirements.md`, `inception/user-stories.md`     | 確認       |
| 4: Visualize（ビジュアライズ）           | `requirements.md`, `user-stories.md` | デザインファイル（.pen）                                     | 確認       |
| 5: QA Plan（QAプラン）                   | `user-stories.md`, デザインファイル  | `inception/qa-plan.md`, `qa/scenarios/web/{unit-name}/*.yml` | 確認       |
| 6: Approval Gate（承認）                 | 全成果物                             | —                                                            | 承認       |
| 7: Unit Decomposition（Unit分割）        | 全成果物                             | `inception/units.md`                                         | 確認       |

**フィードバックルート（Step 6）:**

- 軽微な修正 → Gate 内で修正
- QA シナリオの修正 → Step 5 に戻る
- デザインの大幅変更 → Step 4 に戻る
- 機能追加・再設計 → Step 1 に戻る

---

## Phase 2: Blueprint（設計）

> Phase 1 の成果物をもとに技術設計し、実装計画（Bolt Plan）を策定する

| Step                               | Input                                         | Output                                                                      | 人間の関与        |
| ---------------------------------- | --------------------------------------------- | --------------------------------------------------------------------------- | ----------------- |
| 0: GitHub Projects v2 セットアップ | `config/github-projects.json`（テンプレート） | `config/github-projects.json`（設定済み）                                   | 確認              |
| 1: Functional Design               | Phase 1 成果物                                | `blueprint/functional-design.md` + `units/{unit-name}/functional-design.md` | 確認              |
| 2: Architecture                    | Functional Design + Phase 1 成果物            | `blueprint/architecture.md` + `units/{unit-name}/architecture.md`           | 確認              |
| 3: Bolt Plan                       | Unit 設計成果物 + `user-stories.md`           | `units/{unit-name}/bolt-plan.md`                                            | 承認（Unit 単位） |
| 4: Blueprint Approval Gate         | 全 Unit の設計成果物 + Bolt Plan              | `config/project-adapter.md`                                                 | 承認（全体）      |

**フィードバックルート（Step 4 / Phase 3 Approval Gate から）:**

- 機能設計変更 → Step 1
- アーキテクチャ変更 → Step 2
- Bolt 分割変更 → Step 3

---

## Phase 3: Construction（実装）

> Blueprint に従い、AI agent が完全自律でプロダクションコードを構築する

| Step             | Input                                                      | Output                                                                | 人間の関与         |
| ---------------- | ---------------------------------------------------------- | --------------------------------------------------------------------- | ------------------ |
| 1: Execution     | `bolt-plan.md` + 設計成果物 + デザインファイル             | プロダクションコード + テストコード + `eval/i{intent}-eval-report.md` | `/run-bolt` で実行 |
| 2: Approval Gate | 動作するアプリケーション + 受入基準充足状況 + 評価レポート | —                                                                     | 承認               |

**フィードバックルート（Step 2）:**

- 方向性違い → Phase 1 Step 1
- 画面構成・デザイン変更 → Phase 1 Step 4
- 機能設計変更 → Phase 2 Step 1
- アーキテクチャ変更 → Phase 2 Step 2
- Bolt 分割変更 → Phase 2 Step 3
- 実装修正 → Step 1
- OK → Phase 4

---

## Phase 4: Quality Gate（品質関門）

> コード品質・セキュリティ・パフォーマンスを横断的に検証する

| Step                        | Input                                                                               | Output                   | 人間の関与 |
| --------------------------- | ----------------------------------------------------------------------------------- | ------------------------ | ---------- |
| 1: Quality Check & Auto-Fix | プロダクションコード + `requirements.md`, `architecture.md`, `functional-design.md` | `quality-gate/report.md` | なし       |
| 2: Approval Gate (Go/No-Go) | `report.md`                                                                         | —                        | 承認       |

**判断ルート:**

- Go → 完了
- No-Go（軽微）→ Step 1 再実行
- No-Go（根本的）→ Phase 3 Step 1

---

## 成果物パス一覧

| Phase           | 成果物                                                                                                                                                                                                                                                                   |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1: Inception    | `codebase-snapshot.json`（Brownfield のみ）, `inception/intent.md`, `inception/requirements.md`, `inception/user-stories.md`, `inception/units.md`, デザインファイル（.pen）                                                                                             |
| 2: Blueprint    | `config/github-projects.json`, `config/project-adapter.md`, `blueprint/functional-design.md`, `blueprint/architecture.md`, `blueprint/units/{unit-name}/functional-design.md`, `blueprint/units/{unit-name}/architecture.md`, `blueprint/units/{unit-name}/bolt-plan.md` |
| 3: Construction | プロダクションコード + テスト, `eval/i{intent}-eval-report.md`, `eval/harness-changelog.md`                                                                                                                                                                              |
| 4: Quality Gate | `quality-gate/report.md`                                                                                                                                                                                                                                                 |

> すべての成果物パスは `aidlc-docs/` 配下の相対パス
