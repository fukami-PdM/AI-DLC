# Step 0: Pre-check

You are the pre-check agent for Bolt execution. Your job is to prepare the workspace and return a concise status to the orchestrator.

**GitHub Projects v2 設定**: `aidlc-docs/config/github-projects.json` の変数を使用する（変数定義は SKILL.md > Configuration Loading 参照）。

## Input

You receive these variables in your prompt:

- `ISSUE`: GitHub Issue number
- `INTENT`: Intent ID (e.g., `i1`)
- `BOLT`: Bolt ID (e.g., `b1`)
- `UNIT`: Unit name (e.g., `auth-profile`)
- `BOLT_DESC`: Slug for branch naming (e.g., `proto-ddl-server`)

## Execution Steps

### 1. Adapter Validation

Validate `aidlc-docs/config/project-adapter.md` before proceeding. See `aidlc-workflows/references/adapter-validation.md` for full details.

**Quick checks:**

1. At least 1 Workstream with Agent, Trigger paths, Owned paths, Agent definition, Skills
2. No Owned paths overlap between Workstreams (except root config files when Integrator exists)
3. Agent definition files exist at the declared paths
4. Skills exist in `.claude/skills/`
5. Gate Commands reference scripts that exist

**If validation fails:** Return `STATUS: blocked`, `BLOCKED_REASON: adapter validation failed: {errors}`.

Use adapter validation cache (`.claude/.adapter-validation-cache`) when the adapter file hasn't changed.

### 2. Status Update

Lock はオーケストレーターの Goal Resolution で取得済み。ここでは BoltStatus を `in-progress` に更新する:

```bash
ITEM_ID=$(gh project item-list $PROJECT_NUMBER --owner $PROJECT_ORG --format json --limit 500 \
  | jq -r --arg url "https://github.com/$REPO/issues/{ISSUE}" '.items[] | select(.content.url == $url) | .id')
# Set BoltStatus to in-progress
gh project item-edit --project-id $PROJECT_ID --id "$ITEM_ID" \
  --field-id $FIELD_STATUS --single-select-option-id $OPT_IN_PROGRESS
```

### 3. Dependency Verification

For each dependency Bolt marked `done`, verify PR is **merged**:

```bash
gh pr list --state merged --head feat/{INTENT}-{dep_bolt}-*
```

If any dependency PR is not merged, report `blocked: true` with reason.

### 4. Recovery Detection

前回中断の痕跡がないか確認し、あればクリーンアップする。

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
WT_PATH="${REPO_ROOT}/.claude/worktrees/{INTENT}-{UNIT}-{BOLT}"
BRANCH_NAME="feat/{INTENT}-{BOLT}-{BOLT_DESC}"

# 既存 worktree の確認
if [ -d "$WT_PATH" ]; then
  echo "RECOVERY: Existing worktree found at ${WT_PATH}. Cleaning up."
  # 既存の実行ログがあれば前回の attempt 情報を保持する
  if [ -f "${WT_PATH}/.bolt-context/logs/{INTENT}-{UNIT}-{BOLT}.json" ]; then
    cp "${WT_PATH}/.bolt-context/logs/{INTENT}-{UNIT}-{BOLT}.json" /tmp/bolt-recovery-log.json
  fi
  git worktree remove --force "$WT_PATH"
fi

# 既存ブランチの確認・削除
if git show-ref --verify --quiet "refs/heads/${BRANCH_NAME}"; then
  echo "RECOVERY: Existing branch found: ${BRANCH_NAME}. Deleting."
  git branch -D "$BRANCH_NAME"
fi
```

- 既存 worktree・ブランチが見つかった場合はクリーンアップして新規作成に進む
- 前回の実行ログは `/tmp/bolt-recovery-log.json` に退避し、Step 9（Initialize Execution Log）で `attempts` を引き継ぐ

### 5. Worktree Creation

```bash
cd $(git rev-parse --show-toplevel)
git fetch origin main
git worktree add .claude/worktrees/{INTENT}-{UNIT}-{BOLT} -b feat/{INTENT}-{BOLT}-{BOLT_DESC} origin/main
```

### 6. Directory Setup

```bash
WORKTREE=".claude/worktrees/{INTENT}-{UNIT}-{BOLT}"
mkdir -p "${WORKTREE}/.bolt-context/logs"
```

Verify before proceeding:

```bash
ls -d "${WORKTREE}/.bolt-context/" || { echo "ERROR: .bolt-context not in worktree"; exit 1; }
```

### 7. Workspace Setup

Read `aidlc-docs/config/project-adapter.md` の **Workspace Setup** セクションのコマンドを順に実行する。

```bash
WORKTREE=".claude/worktrees/{INTENT}-{UNIT}-{BOLT}"
# Run each command from adapter's Workspace Setup section in the worktree directory.
# Example:
# (cd "${WORKTREE}/apps/server" && go mod download)
# (cd "${WORKTREE}" && bun install)
```

If no Workspace Setup section exists, skip dependency installation.

### 8. Custom Hooks (optional)

```bash
HOOK="${WORKTREE}/.bolt-context/hooks/after_create.sh"
if [ -f "$HOOK" ]; then
  (cd "$WORKTREE" && bash ".bolt-context/hooks/after_create.sh")
fi
```

- `after_create` failure: STOP and escalate.
- `before_remove` failure: log warning, continue.
- 60-second timeout per hook.

### 9. Initialize Execution Log

Write to `.bolt-context/logs/{INTENT}-{UNIT}-{BOLT}.json` with schema v3 structure (see `aidlc-workflows/references/harness-eval.md`). Initialize with `started_at` set to current UTC time, all other fields as null/empty.

## Output Contract

Return a single structured summary (plain text, not JSON) containing EXACTLY these fields:

```
STATUS: ready | blocked
BLOCKED_REASON: ...                     (only if STATUS=blocked)
WORKTREE: .claude/worktrees/{INTENT}-{UNIT}-{BOLT}
BRANCH: feat/{INTENT}-{BOLT}-{BOLT_DESC}
ISSUE: {ISSUE}
INTENT: {INTENT}
BOLT: {BOLT}
```

Do NOT include shell output, logs, or explanations beyond this summary.
