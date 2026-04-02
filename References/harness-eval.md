# Harness Eval Reference

> Bolt 実行結果からハーネス（Skills, Context, Gate）の品質を定量評価し、改善サイクルを回す。

## 目的

ハーネスエンジニアリングの本質は「ハーネス自体の継続的改善」にある。Bolt の成功/失敗はコードの品質だけでなく、ハーネスの品質を反映する。メトリクスを蓄積・分析し、Skills や Prepare のプロンプト品質を改善する。

## メトリクス収集

### 収集タイミング

各 Bolt の Step 4 (Ship) cleanup 時に、実行ログ (`.bolt-context/logs/i{intent}-{unit}-b{N}.json`) を確定させる。命名規則は `.claude/skills/run-bolt/SKILL.md` を参照。

### 実行ログスキーマ v3

```json
{
  "version": "3.0",
  "bolt": "I{intent}-B{N}",
  "issue": 123,
  "branch": "feat/i{intent}-b{N}-{desc}",
  "started_at": "<ISO 8601 UTC>",
  "completed_at": "<ISO 8601 UTC>",
  "outcome": "merged | escalated | failed",
  "workstreams": ["backend-implementer", "web-implementer"],
  "skills_used": {
    "backend-implementer": ["backend-domain-layer-generator", "backend-repository-layer-generator"],
    "web-implementer": ["building-native-ui"]
  },
  "timing": {
    "pre_check_sec": 15,
    "prepare_sec": 40,
    "execute_sec": 200,
    "gate_sec": 55,
    "review_sec": 90,
    "ship_sec": 30,
    "total_sec": 430
  },
  "context": {
    "design_docs_total_lines": 450,
    "context_budget_applied": true,
    "summarized_files": ["functional-design.md"],
    "prompt_lines_per_workstream": {
      "backend-implementer": 78,
      "web-implementer": 65
    }
  },
  "attempts": [
    {
      "attempt": 1,
      "gate_results": {
        "lint": "pass",
        "typecheck": "pass",
        "test": "fail",
        "build": "pass"
      },
      "gate_overall": "fail",
      "failure_category": "test-logic",
      "errors_summary": "TestCreateCharacter: expected 201, got 400 — validation rule missing"
    },
    {
      "attempt": 2,
      "gate_results": {
        "lint": "pass",
        "typecheck": "pass",
        "test": "pass",
        "build": "pass"
      },
      "gate_overall": "pass",
      "failure_category": null,
      "errors_summary": null
    }
  ],
  "review": {
    "rounds": 1,
    "findings_total": 3,
    "findings_resolved": 3,
    "outcome": "approved"
  },
  "ci": {
    "attempts": 1,
    "outcome": "pass"
  }
}
```

### timing フィールド

各ステップの所要時間（秒）を記録する。計測方法:

```bash
# 各ステップの開始時に記録
STEP_START=$(date -u +%s)
# ... ステップ実行 ...
# 終了時に差分を算出
STEP_END=$(date -u +%s)
STEP_SEC=$((STEP_END - STEP_START))
```

| フィールド      | 計測区間                                               |
| --------------- | ------------------------------------------------------ |
| `pre_check_sec` | Step 0 開始 〜 Pre-check 完了                          |
| `prepare_sec`   | Step 1 開始 〜 Prepare 完了                            |
| `execute_sec`   | Step 2 開始 〜 全 Implementer + Integrator 完了        |
| `review_sec`    | Step 3 開始 〜 Review 完了（スキップ時は 0）           |
| `gate_sec`      | Step 4 開始 〜 Gate 完了（再生成時は最後の Gate のみ） |
| `ship_sec`      | Step 5 開始 〜 merge + cleanup 完了                    |
| `total_sec`     | `started_at` 〜 `completed_at` の差分                  |

### context フィールド

Prepare が処理した設計文書のサイズとプロンプト生成結果を記録する。

| フィールド                    | 内容                                        |
| ----------------------------- | ------------------------------------------- |
| `design_docs_total_lines`     | 読み込んだ設計文書の合計行数                |
| `context_budget_applied`      | Context Budget の要約が適用されたか         |
| `summarized_files`            | 要約が適用されたファイル名の一覧            |
| `prompt_lines_per_workstream` | 各 Workstream の Implementer プロンプト行数 |

### failure_category 分類

| Category          | 説明                               | 改善対象                               |
| ----------------- | ---------------------------------- | -------------------------------------- |
| `test-logic`      | テストロジックの実装ミス           | Skill（テストパターン）                |
| `test-missing`    | テストが不足・未実装               | Prepare（受入基準の抽出精度）          |
| `lint`            | lint/format 違反                   | Skill（コードスタイル制約）            |
| `typecheck`       | 型エラー                           | Skill（型定義パターン）                |
| `build`           | ビルドエラー（import, dependency） | Skill（モジュール構成）                |
| `architecture`    | レイヤー境界違反                   | Skill（レイヤー制約）                  |
| `integration`     | Workstream 間の不整合              | Integrator / Prepare（ownership 分割） |
| `design-mismatch` | 設計文書との乖離                   | Prepare（コンテキスト品質）            |
| `external`        | 環境・権限・ネットワーク           | Adapter（Workspace Setup）             |

## 分析プロセス

### 2段階の評価タイミング

Eval は **Bolt 単位の軽量チェック** と **Intent 単位の全量分析** の2段階で実行する。

#### Bolt 単位: リアルタイム異常検出

**タイミング**: 各 Bolt の Ship 完了直後（実行ログ確定後）

**目的**: 同じ問題で後続 Bolt が連続失敗するのを防ぐ早期アラート

**チェック項目** (直近の完了済み Bolt ログを対象):

1. **連続失敗パターン**: 直近 3 件の Bolt で同じ `failure_category` が発生 → アラート
2. **再生成率の急増**: 直近 3 件中 2 件以上が再生成 → アラート
3. **ボトルネック異常**: 直近の Bolt で特定ステップが `total_sec` の 60% 超 → 警告
4. **プロンプト上限接近**: `prompt_lines_per_workstream` のいずれかが 90 行超 → 警告

**アクション**:

| 検出結果     | アクション                                                                                     |
| ------------ | ---------------------------------------------------------------------------------------------- |
| アラートなし | 次の Bolt に進む                                                                               |
| 警告のみ     | ログに記録し、次の Bolt に進む                                                                 |
| アラートあり | **ループを一時停止**し、人間に報告。判断を仰ぐ: (a) 続行 (b) Skill を修正してから続行 (c) 中止 |

**実装**: オーケストレーターが Ship 完了後に `.bolt-context/logs/` の直近ログを読んで判定する。サブエージェント不要。

#### Intent 単位: 全量分析

**タイミング**:

- Phase 3 Step 1 の全 Bolt 完了後、Step 2 (Approval Gate) の前
- オンデマンド（ユーザーが明示的に要求した場合）

**目的**: 傾向分析、ハーネス改善提案の生成

### 分析ステップ (Intent 単位)

#### 1. メトリクス集計

`.bolt-context/logs/` 配下の全ログを読み込み、以下を算出する:

```
## Harness Eval Report — Intent I{intent}

### Summary
- Total Bolts: N
- First-pass success rate: X/N (Y%)
- Regeneration rate: X/N (Y%)
- Escalation rate: X/N (Y%)
- CI fix rate: X/N (Y%)

### Gate Results by Check
| Check | Pass (1st) | Pass (2nd) | Fail |
|-------|-----------|-----------|------|
| lint  | N         | N         | N    |
| test  | N         | N         | N    |
| build | N         | N         | N    |

### Failure Category Distribution
| Category | Count | % |
|----------|-------|---|
| test-logic | N | X% |
| lint | N | X% |
| ...  | N | X% |

### Skill Effectiveness
| Skill | Bolts Used | 1st-pass Rate | Notes |
|-------|-----------|---------------|-------|
| ... | N | X% | ... |

### Performance (Observability)
| Step | Avg (sec) | P90 (sec) | Max (sec) | Bottleneck? |
|------|-----------|-----------|-----------|-------------|
| Pre-check | N | N | N | |
| Prepare | N | N | N | |
| Execute | N | N | N | ★ |
| Gate | N | N | N | |
| Review | N | N | N | |
| Ship | N | N | N | |
| **Total** | N | N | N | |

Bottleneck: P90 が Total P90 の 40% 以上を占めるステップに ★ を付与する。

### Context Budget Usage
- Budget 適用率: X/N Bolts (Y%)
- 要約対象ファイル: {file} (N回), {file} (N回)
- プロンプト平均行数: N 行 (上限 100)
- プロンプト最大行数: N 行
```

#### 2. パターン検出

集計結果から以下のパターンを検出する:

- **繰り返し失敗**: 同じ `failure_category` が 3 回以上 → Skill 改善候補
- **Skill 相関**: 特定 Skill を使った Bolt の 1st-pass rate が低い → Skill の品質問題
- **Workstream 偏り**: 特定 Workstream の失敗率が高い → Agent 定義 or Skill の問題
- **再生成効果**: 再生成で解決した割合が低い → エラーコンテキストの伝達方法の問題
- **ボトルネック**: 特定ステップの P90 が Total の 40% 以上 → そのステップの最適化が必要
- **プロンプト肥大化**: `prompt_lines_per_workstream` の平均が 80 行以上 → Context Budget の閾値見直し
- **要約頻度**: `context_budget_applied` が 50% 以上の Bolt で true → 設計文書の分割粒度の見直し

#### 3. 改善提案の生成

検出したパターンに基づき、具体的な改善アクションを提案する:

```
## Improvement Recommendations

### HIGH: Skill "{skill-name}" needs test pattern update
- Evidence: 4/6 Bolts using this skill failed with `test-logic`
- Action: Add {specific pattern} to the skill's test guidance section

### MEDIUM: Prepare prompt missing {X} context
- Evidence: 2 Bolts failed with `design-mismatch` on API response format
- Action: Ensure Prepare extracts API contract details from architecture.md
```

#### 4. 改善の実施判断

| 改善タイプ                    | 自動実施 | 人間承認         |
| ----------------------------- | -------- | ---------------- |
| Skill へのパターン追記        | No       | Yes — 提案を提示 |
| Prepare プロンプトの調整      | No       | Yes — 提案を提示 |
| Adapter の Gate Commands 追加 | No       | Yes — 提案を提示 |
| failure_category の新規追加   | No       | Yes — 提案を提示 |

> **設計判断**: 初期フェーズではすべて人間承認とする。自動改善は十分なデータと信頼性が確立されてから導入する。

## レポート出力先

```
aidlc-docs/eval/
├── i{intent}-eval-report.md    # Intent ごとの評価レポート
└── harness-changelog.md        # 改善履歴（何を、なぜ変えたか）
```

## harness-changelog.md のフォーマット

```markdown
# Harness Changelog

## [YYYY-MM-DD] I{intent} 評価後

### Changed

- **Skill: {name}**: {変更内容} — {根拠: failure_category X が N 回}

### Rationale

{分析レポートへのリンクと判断理由}
```
