# Execution Policy: Clean Regeneration

## Core Principle

失敗したら修正しない。コンテキストにエラー情報を追加して再生成する。

## Regeneration Flow

```
Gate Fail (Step 3)
  ↓
1. Capture error logs (full stdout/stderr of failed checks)
2. Clean workspace (preserve .bolt-context):
   cd {worktree} && git checkout -- . && git clean -fd -e .bolt-context
3. Pass error logs to Prepare as PREVIOUS_ERRORS
4. Re-execute Step 1 (Prepare) — regenerate implementer prompts with error context
5. Re-execute Step 2 (Execute) — implementers start fresh with error context
6. Re-execute Step 3 (Gate)
  → Pass → Step 4 (Ship)
  → Fail → Escalate to human, stop execution
```

## Max Attempts

| Phase                      | Max Attempts                 |
| -------------------------- | ---------------------------- |
| Execute + Gate             | 2 (initial + 1 regeneration) |
| CI fix after PR (Step 4.3) | 2                            |

## Escalation Triggers

Any of these → stop execution and inform the user:

- 2x Gate failure (same Bolt)
- Design-level contradictions (requirements vs architecture mismatch)
- Authentication / permission errors
- Dependency Bolt not actually merged (pre-check should catch this)

### Escalation Cleanup

エスカレーション時は以下を実行してから停止する:

1. BoltStatus を `blocked` に戻す
2. **Lock を `unlocked` に戻す**（再実行を可能にする）
3. Worktree が存在する場合は削除する

```bash
ITEM_ID=$(gh project item-list $PROJECT_NUMBER --owner $PROJECT_ORG --format json --limit 500 \
  | jq -r --arg url "https://github.com/$REPO/issues/{issue}" '.items[] | select(.content.url == $url) | .id')
gh project item-edit --project-id $PROJECT_ID --id "$ITEM_ID" \
  --field-id $FIELD_STATUS --single-select-option-id $OPT_BLOCKED
gh project item-edit --project-id $PROJECT_ID --id "$ITEM_ID" \
  --field-id $FIELD_LOCK --single-select-option-id $OPT_UNLOCKED
```

## Logging

実行ログは `.bolt-context/logs/{INTENT}-{UNIT}-{BOLT}.json` に記録する。スキーマ v3 の詳細は `aidlc-workflows/references/harness-eval.md` > 実行ログスキーマ v3 を参照。

**When to write:**

- Initialize at Step 0 (Pre-check) start
- Append attempt entries as each Execute+Gate cycle completes
- Finalize at Step 4 completion (set outcome)

**Do not write raw stdout/stderr or sensitive values into this committed log.**
