---
name: run-bolt
description: >
  Execute a single Bolt through the pipeline: Pre-check (workspace setup) → Prepare
  (context generation) → Execute (parallel implementation with Skills) →
  Review (optional AI review) → Gate (CI-equivalent checks) → Ship (PR +
  auto-merge). Uses Skills as inline harness for quality, CI as the only
  hard gate, and clean regeneration on failure.
  Use when the user says "run-bolt", "run bolt", "next bolt", "next", or
  wants to start implementation of a Bolt Issue.
---

# Run Bolt

Execute one Bolt per session. To run multiple Bolts in parallel, launch a separate Claude Code session for each Bolt.

User input: $ARGUMENTS

## Configuration Loading

実行開始時に以下の設定ファイルを読み込む:

1. **`aidlc-docs/config/github-projects.json`** — GitHub Projects v2 の変数:
   `$PROJECT_ORG`, `$PROJECT_NUMBER`, `$PROJECT_ID`, `$REPO`, `$FIELD_STATUS`, `$FIELD_INTENT`, `$FIELD_BOLT`, `$FIELD_LOCK`, `$OPT_READY`, `$OPT_BLOCKED`, `$OPT_IN_PROGRESS`, `$OPT_DONE`, `$OPT_LOCKED`, `$OPT_UNLOCKED`

2. **`aidlc-docs/config/project-adapter.md`** — プロジェクト固有の実行設定:
   - **Workstreams**: 実装エージェント定義（Agent subagent_type, Trigger paths, Owned paths, Skills）
   - **Integrator**: 統合エージェント設定
   - **Gate Commands**: CI 相当のチェックコマンド
   - **Review Configuration**: AI レビュー設定

### Project Adapter のステップ別参照

| ステップ  | Adapter から読む情報                                        |
| --------- | ----------------------------------------------------------- |
| Pre-check | Workspace Setup（依存パッケージインストール）               |
| Prepare   | Workstreams（どの実装エージェントを起動するか、Skill 順序） |
| Execute   | Workstreams の Agent 定義、Owned paths                      |
| Review    | Review Configuration                                        |
| Gate      | Gate Commands                                               |
| Ship      | （Adapter 不要 — 共通手順）                                 |

## Goal Resolution

1. **Explicit goal**: User provided a specific goal — use that.
2. **Backlog-driven**: Input is empty, a path, or "next"/"continue":
   - List ready Bolts from GitHub Projects v2:
     ```bash
     gh project item-list $PROJECT_NUMBER --owner $PROJECT_ORG --format json --limit 500 \
       | jq '[.items[] | select(.boltStatus == "ready" and .lock != "locked" and .content.type == "Issue")] | [.[] | {number: .content.number, title: .content.title, url: .content.url, intent: .intent, bolt: .bolt}]'
     ```
   - No ready Bolts: inform user and exit.
   - **Single Bolt only**: pick the first ready Bolt and execute it. If other Bolts are also ready, list them and instruct the user to launch a **separate Claude Code session** for each additional Bolt. Do NOT execute multiple Bolts in a single session.
   - **Immediate Lock**: Bolt を選択したら、pre-check 起動前に即座に Lock を取得する（race condition 防止）:
     ```bash
     ITEM_ID=$(gh project item-list $PROJECT_NUMBER --owner $PROJECT_ORG --format json --limit 500 \
       | jq -r --arg url "https://github.com/$REPO/issues/{issue}" '.items[] | select(.content.url == $url) | .id')
     gh project item-edit --project-id $PROJECT_ID --id "$ITEM_ID" \
       --field-id $FIELD_LOCK --single-select-option-id $OPT_LOCKED
     ```

## Naming Convention

| Item         | Pattern                                       | Example (I001-B2a)                               |
| ------------ | --------------------------------------------- | ------------------------------------------------ |
| Issue title  | `[I{intent}-B{N}] {unit}: <title>`            | `[I001-B2a] auth-profile: Character master CRUD` |
| Branch       | `feat/i{intent}-b{N}-{desc}`                  | `feat/i001-b2a-character-crud`                   |
| Worktree     | `.claude/worktrees/i{intent}-{unit}-b{N}`     | `.claude/worktrees/i001-auth-profile-b2a`        |
| Context JSON | `.bolt-context/i{intent}-b{N}[.<agent>].json` | `.bolt-context/i001-b2a.json`                    |

## Execution Pipeline

> プロジェクト固有の設定は `aidlc-docs/config/project-adapter.md` から読み込む。

```
Step 0: Pre-check  → Agent(pre-check-bolt) — subagent handles recovery, lock, worktree
Step 1: Prepare    → Agent(prepare-bolt) — subagent reads design docs + adapter, returns implementer prompts
Step 2: Execute    → Agent(workstream-1) || Agent(workstream-2) || ...  ← parallel per adapter
                     Agent(integrator) if 2+ workstreams (per adapter)
Step 3: Review     → Per adapter Review Configuration — parallel reviewer agents, auto-fix, iterate
Step 4: Gate       → Shell script per adapter Gate Commands — NOT an agent
Step 5: Ship       → commit, push, PR create, background CI wait, auto-merge, cleanup
```

### Failure Policy — see [references/execution-policy.md](references/execution-policy.md)

- Gate fail → clean workspace, add errors to context, **regenerate** (not patch)
- 2nd Gate fail → escalate to human
- Max 2 attempts total (initial + 1 regeneration)
- Review auto-fix による Gate 失敗の場合も同じポリシーを適用する

---

## Step 0: Pre-check (Subagent)

**Delegate to a subagent** to keep recovery logic, shell outputs, and lock management out of the orchestrator's context. Instructions are in [references/pre-check.md](references/pre-check.md).

```
Agent(subagent_type="general-purpose",
  name="pre-check-bolt",
  description="Pre-check I{intent}-B{N}",
  prompt="You are the pre-check agent. Read and follow .claude/skills/run-bolt/references/pre-check.md exactly.

    ISSUE: {issue_number}
    INTENT: i{intent}
    BOLT: b{N}
    UNIT: {unit-name}
    BOLT_DESC: {slug from issue title}

    Execute all steps and return ONLY the structured output defined in the Output Contract.")
```

**Parse the result**:

- `STATUS: blocked` → inform user and exit.
- `STATUS: ready` → proceed to Step 1.

Extract `WORKTREE` and `BRANCH` from the result for subsequent steps.

## Step 1: Prepare (Subagent)

**Delegate to a subagent** to keep design document content out of the orchestrator's context. The subagent reads all artifacts, generates context JSON, and returns only the implementer prompts. Instructions are in [references/prepare.md](references/prepare.md).

Determine `{unit}` from the Bolt Issue title or bolt-plan.md path convention.

```
Agent(subagent_type="general-purpose",
  name="prepare-bolt",
  description="Prepare I{intent}-B{N}",
  prompt="You are the prepare agent. Read and follow .claude/skills/run-bolt/references/prepare.md exactly.

    WORKTREE: {abs_worktree_path}
    ISSUE: {issue_number}
    INTENT: i{intent}
    BOLT: b{N}
    UNIT: {unit-name}

    {if regeneration attempt:}
    PREVIOUS_ERRORS:
    {error log from previous Gate failure}

    Execute all steps and return ONLY the structured output defined in the Output Contract.")
```

**Parse the result**:

- `IMPLEMENTERS` → list of agents to spawn in Step 2
- `NEEDS_INTEGRATOR` → whether to spawn integrator after
- Each `--- {AGENT}-IMPLEMENTER PROMPT ---` section → use as-is for the corresponding Agent prompt in Step 2

## Step 2: Execute

Spawn implementers **in parallel** (single message, multiple Agent calls), using the prompts returned by the prepare-bolt subagent.

The prepare-bolt subagent determines which workstreams are needed based on `project-adapter.md`. For each workstream listed in IMPLEMENTERS, spawn the corresponding agent:

```
# For each workstream in IMPLEMENTERS (all in SAME message for parallel execution):
Agent(subagent_type="{workstream agent from adapter}",
  description="{Workstream} I{intent}-B{N}",
  prompt="{paste {WORKSTREAM}-IMPLEMENTER PROMPT section from prepare-bolt result}")
```

**If NEEDS_INTEGRATOR is true** (per adapter Integrator config): spawn integrator after all implementers complete:

```
Agent(subagent_type="{integrator agent from adapter}",
  description="Integrate I{intent}-B{N}",
  prompt="WORKTREE: {abs_path}. Resolve cross-stream conflicts.")
```

## Step 3: Review (per adapter)

**Read Review Configuration from `aidlc-docs/config/project-adapter.md`.**

- If Method is `none` → skip to Step 4.
- Otherwise, invoke the configured review skill with the specified reviewers and max rounds.

**Invocation** (example for `/agent-review-loop`):

```
Skill("agent-review-loop",
  args="WORKTREE={abs_worktree_path} INTENT={intent} UNIT={unit} BOLT={bolt} BASE_BRANCH=main MAX_ROUNDS={max_rounds} --reviewers {comma-separated reviewer ids}")
```

Concrete example with adapter defaults:

```
Skill("agent-review-loop",
  args="WORKTREE=/Users/me/repo/.claude/worktrees/i001-auth-profile-b2a INTENT=i1 UNIT=auth-profile BOLT=b2a BASE_BRANCH=main MAX_ROUNDS=3 --reviewers code-quality,security,performance,test-coverage,silent-failure")
```

**Decision**:

- **All reviewers APPROVE** → proceed to Step 4
- **Max rounds exceeded with unresolved findings** → report warnings to human, proceed to Step 4 with review summary attached

## Step 4: Gate

**Run as shell commands — NOT as an agent.**

Execute the Gate Commands defined in `aidlc-docs/config/project-adapter.md`:

```bash
cd {worktree}
# Run each command from adapter's Gate Commands section.
# Only run checks relevant to the files changed by this Bolt.
```

**Decision**:

- **All pass** → proceed to Step 5
- **Any fail** → execute regeneration policy (see [references/execution-policy.md](references/execution-policy.md))

## Step 5: Ship

### 5.0 Execution Log Finalization（コミット前）

**コミット前に**実行ログを `.bolt-context/logs/{INTENT}-{UNIT}-{BOLT}.json` に書き出す。これにより実行ログが PR に含まれる。

スキーマ v3 に従い以下を記録する（詳細は `aidlc-workflows/references/harness-eval.md`）:

スキーマ v3 の全フィールド定義は `aidlc-workflows/references/harness-eval.md` > 実行ログスキーマ v3 を参照。

### 5.1 Commit, rebase, and push

```bash
cd {worktree}
git add -A
git commit -m "$(cat <<'EOF'
feat: I{intent}-B{N} - {title}

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"

# Rebase onto latest main before push
git fetch origin main
git rebase origin/main
```

- **コンフリクトなし** → push へ進む
- **コンフリクトあり** → 「Merge Conflict Resolution」手順（後述）に従い解決する

```bash
git push -u origin feat/i{intent}-b{N}-{desc}
```

### 5.2 Create PR

```bash
gh pr create --title "[I{intent}-B{N}] {title}" --body "$(cat <<'EOF'
## Summary
- {2-3 bullet points}

Closes #{issue}

## Acceptance Criteria
- [x] {linked to US-XXX}

## Gate Results
- lint: PASS/FAIL
- test: PASS/FAIL
- build: PASS/FAIL
- architecture boundaries: PASS/FAIL
- test coverage: PASS/FAIL

## Attempts
- Attempt 1: {pass/fail}
- Attempt 2: {pass/fail, if applicable}

## Risk and Rollback
- {risk level / rollback plan}

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

### 5.3 Merge Loop

**NEVER merge without all CI checks passing.**

merge 成功まで以下をループする（最大3回）:

```
MERGE_ATTEMPTS = 0
WHILE MERGE_ATTEMPTS < 3:
  1. CI パスを待つ（background で監視）
  2. squash merge を試行する
  3. merge 成功 → ループ終了、Cleanup へ
  4. コンフリクトで merge 失敗 → rebase + conflict resolution + Gate 再実行 + force push → 1 に戻る
  MERGE_ATTEMPTS += 1

3回失敗 → 人間にエスカレーション
```

**CI 監視（background）:**

```bash
# Run in background (use Bash tool with run_in_background=true):
gh pr checks {pr_number} --watch --fail-fast
```

**CI パス後の merge 試行:**

```bash
gh pr merge {pr_number} --squash
# Confirm:
gh pr view {pr_number} --json state --jq '.state'  # Must return "MERGED"
```

**merge 失敗時（コンフリクト）:**

```bash
cd {worktree}
git fetch origin main
git rebase origin/main
# → コンフリクトがあれば「Merge Conflict Resolution」手順で解決
# → Gate 再実行（Step 4 の Gate Commands）
git push --force-with-lease origin feat/i{intent}-b{N}-{desc}
# → CI 監視に戻る
```

- CI failure（コンフリクト以外）: `gh run view {run_id} --log-failed`, fix, push, wait. Max 2 CI fix iterations.
- Human intervention needed: inform user and stop.

### Merge Conflict Resolution

rebase 時にコンフリクトが発生した場合の解決手順:

1. コンフリクトファイルを特定する: `git diff --name-only --diff-filter=U`
2. 各コンフリクトファイルについて、両方の変更意図を把握する:
   - **ours（現在の Bolt）**: Bolt のコンテキスト JSON とスコープから意図を参照する
   - **theirs（main に merge 済みの変更）**: `git log origin/main --oneline -5` で直近の変更を確認する
3. 両方の変更を保持する方向で解決する（追記の競合は両方の追記を統合する）
4. `git rebase --continue` で rebase を完了する
5. **Gate を再実行する**（Step 4 の Gate Commands を再度実行し、解決結果の安全性を検証する）
6. Gate パス → force push して続行。Gate 失敗 → 人間にエスカレーション

### 5.4 Cleanup

1. Set current Bolt status to `done` and release Lock:
   ```bash
   ITEM_ID=$(gh project item-list $PROJECT_NUMBER --owner $PROJECT_ORG --format json --limit 500 \
     | jq -r --arg url "https://github.com/$REPO/issues/{issue}" '.items[] | select(.content.url == $url) | .id')
   gh project item-edit --project-id $PROJECT_ID --id "$ITEM_ID" \
     --field-id $FIELD_STATUS --single-select-option-id $OPT_DONE
   # Release Lock
   gh project item-edit --project-id $PROJECT_ID --id "$ITEM_ID" \
     --field-id $FIELD_LOCK --single-select-option-id $OPT_UNLOCKED
   ```
2. **Update downstream Bolts (MANDATORY)**: scan open Bolt Issues, check deps, set to `ready` if **all** deps are satisfied:

   ```bash
   # 1. 全 Bolt Issue を取得する
   ALL_ITEMS=$(gh project item-list $PROJECT_NUMBER --owner $PROJECT_ORG --format json --limit 500)

   # 2. blocked 状態の Bolt を列挙する
   BLOCKED_ISSUES=$(echo "$ALL_ITEMS" | jq '[.items[] | select(.boltStatus == "blocked" and .content.type == "Issue")] | [.[] | {number: .content.number, url: .content.url, id: .id}]')

   # 3. 各 blocked Bolt の Issue body から Depends On を取得し、全依存先が done かチェックする
   for row in $(echo "$BLOCKED_ISSUES" | jq -r '.[] | @base64'); do
     ISSUE_NUM=$(echo "$row" | base64 --decode | jq -r '.number')
     ITEM_ID=$(echo "$row" | base64 --decode | jq -r '.id')
     # Issue body の "Depends On" セクションから依存 Issue 番号を抽出
     DEP_ISSUES=$(gh issue view "$ISSUE_NUM" --json body --jq '.body' \
       | grep -oP '#\d+' | grep -oP '\d+' || true)
     ALL_DONE=true
     for DEP in $DEP_ISSUES; do
       DEP_STATE=$(gh issue view "$DEP" --json state --jq '.state')
       if [ "$DEP_STATE" != "CLOSED" ]; then
         ALL_DONE=false
         break
       fi
     done
     if [ "$ALL_DONE" = true ] && [ -n "$DEP_ISSUES" ]; then
       gh project item-edit --project-id $PROJECT_ID --id "$ITEM_ID" \
         --field-id $FIELD_STATUS --single-select-option-id $OPT_READY
       gh issue comment "$ISSUE_NUM" --body "UNBLOCKED: All deps satisfied. (Triggered by I{intent}-B{N})"
     fi
   done
   ```

3. Run `before_remove` hook if exists.
4. `git worktree remove .claude/worktrees/i{intent}-{unit}-b{N}`
5. Report newly-readied Bolts to user.

## Post-Ship: Bolt 単位ハーネスチェック

Ship 完了・実行ログ確定後、**サブエージェントを使わず**オーケストレーターが直接実行する軽量チェック。

> 詳細は `aidlc-workflows/references/harness-eval.md` > Bolt 単位: リアルタイム異常検出 を参照。

1. main ブランチの最新を取得し、ログを読む:
   ```bash
   git fetch origin main
   git show origin/main:.bolt-context/logs/{INTENT}-{UNIT}-{BOLT}.json
   ```
   直近 3 件の完了済みログは `git log origin/main -- .bolt-context/logs/` でファイル一覧を取得し、各ファイルを `git show` で読む（worktree は削除済みのため）
2. 以下をチェックする:
   - 同じ `failure_category` が 3 連続 → **ALERT**
   - 直近 3 件中 2 件以上が再生成（`attempts` 配列の長さ ≥ 2）→ **ALERT**
   - 直近 Bolt で特定ステップが `total_sec` の 60% 超 → **WARN**
   - `prompt_lines_per_workstream` のいずれかが 90 行超 → **WARN**
3. **ALERT がある場合**: Required Outputs に `HARNESS_ALERT` セクションを追加し、Phase 3 ループの一時停止を促す
4. **WARN のみ**: Required Outputs に `HARNESS_WARN` セクションを追加（ループは続行）

## Required Outputs

- Intent + Bolt ID, Issue number + URL, Branch name
- Implementers spawned and what each did
- Review report (`.bolt-context/reviews/{INTENT}-{UNIT}-{BOLT}.md` — findings count, rounds, verdict)
- Gate results (pass/fail per check, attempt count)
- PR URL
- Downstream Bolts unblocked
- Remaining risks
- HARNESS_ALERT / HARNESS_WARN（該当する場合のみ）

## Timing — ステップ計測

オーケストレーターは各ステップの開始・終了時刻を記録し、実行ログの `timing` フィールドに書き込む。

```bash
# 各ステップ開始時:
STEP_START=$(date -u +%s)

# 各ステップ終了時:
STEP_END=$(date -u +%s)
STEP_SEC=$((STEP_END - STEP_START))
```

記録タイミング:

- **Step 0 完了後**: `pre_check_sec` を記録
- **Step 1 完了後**: `prepare_sec` を記録
- **Step 2 完了後**: `execute_sec` を記録
- **Step 3 完了後**: `review_sec` を記録（スキップ時は 0）
- **Step 4 完了後**: `gate_sec` を記録（再生成時は最後の Gate）
- **Step 5 完了後**: `ship_sec` を記録、`total_sec` を算出
