# Adapter Validation Reference

> `project-adapter.md` の整合性を実行前に検証し、設定ミスによる実行時失敗を防ぐ。

## 実行タイミング

- **Pre-check (Step 0)** の Lock Acquisition 前に実行する
- バリデーション失敗 → `STATUS: blocked`, `BLOCKED_REASON: adapter validation failed` で停止

## バリデーション項目

### 1. 構造チェック（必須セクションの存在）

| セクション           | 必須 | 検証内容                                                                   |
| -------------------- | ---- | -------------------------------------------------------------------------- |
| Workstreams          | Yes  | 最低 1 つの Workstream が定義されている                                    |
| 各 Workstream        | Yes  | Agent, Trigger paths, Owned paths, Agent definition, Skills が全て存在する |
| Gate Commands        | Yes  | 最低 1 つのコマンドが定義されている                                        |
| Integrator           | No   | 存在する場合、Agent と Agent definition が定義されている                   |
| Workspace Setup      | No   | 存在する場合、コードブロック内にコマンドがある                             |
| Review Configuration | No   | 存在する場合、Method, Reviewers, Max rounds が定義されている               |

### 2. パス整合性チェック

#### Owned paths の重複検出

複数の Workstream が同一パスを所有していないことを検証する。

```
CHECK: 全 Workstream の Owned paths を展開し、重複する glob パターンがないか確認する。
```

**例（違反）:**

```
backend Owned: apps/server/**, packages/**
web Owned:     apps/web/**, packages/**
→ ERROR: "packages/**" is owned by both backend and web
```

**例（許容）:**

```
backend Owned: apps/server/**, proto/**
native Owned:  apps/native/**, packages/**
→ OK: no overlap
```

> **例外**: `package.json`, `bunfig.toml` のようなルート設定ファイルは複数 Workstream で共有可能。Integrator が設定されている場合のみ許容する。

#### Trigger paths の整合性チェック

Trigger paths には2種類ある:

1. **Owned trigger**: Workstream が編集するファイル（Owned paths 内）
2. **Read-only trigger**: Workstream の実行をトリガーするが、編集対象ではないファイル（例: proto 定義の変更がバックエンドをトリガー）

バリデーションでは以下を検証する:

```
CHECK 1: 各 Trigger path が、自身の Owned paths または他 Workstream の Owned paths のいずれかに含まれるか確認する。
         どの Workstream にも属さないパスは設定ミスの可能性がある。
CHECK 2: 複数の Workstream で同一の Trigger path が設定されている場合、警告を出す（意図的な場合もあるため ERROR ではなく WARN）。
```

**例（OK — read-only trigger）:**

```
Trigger: proto/**, apps/server/**
Owned:   apps/server/**
→ OK: "proto/**" is a read-only trigger (not owned, but valid as trigger source)
```

**例（WARN — orphan trigger）:**

```
Trigger: scripts/migrate/**, apps/server/**
Owned:   apps/server/**
→ WARN: "scripts/migrate/**" is not owned by any Workstream — verify this is intentional
```

### 3. Agent 定義の存在チェック

各 Workstream と Integrator の Agent definition パスにファイルが存在することを検証する。

```bash
# 各 Workstream について
test -f "{agent_definition_path}" || echo "ERROR: Agent definition not found: {path}"
```

### 4. Skill の存在チェック

各 Workstream の Skills リストに記載された Skill が `.claude/skills/` 配下に存在することを検証する。

```bash
# 各 Skill について
test -d ".claude/skills/{skill-name}" || echo "ERROR: Skill not found: {skill-name}"
```

### 5. Gate Commands の実行可能性チェック

Gate Commands で参照されるスクリプトやツールが存在することを検証する。

```bash
# スクリプト参照を検出して存在確認
# 例: ./scripts/check_architecture_boundaries.sh
test -f "{script_path}" || echo "ERROR: Gate script not found: {path}"
```

> コマンド自体（`make`, `bun` 等）のインストール確認は Workspace Setup の責務とし、ここでは検証しない。

### 6. Review Configuration の整合性チェック

Review Configuration が存在する場合:

- Method で参照される Skill が存在することを確認する
- Max rounds が 1 以上の整数であることを確認する

## 出力フォーマット

```
ADAPTER VALIDATION: PASS | FAIL

{FAIL の場合:}
ERRORS:
- [structure] Workstream "backend" missing Owned paths
- [overlap] "packages/**" owned by both native and web (no Integrator)
- [agent] Agent definition not found: .claude/agents/foo.md
- [skill] Skill not found: nonexistent-skill
- [gate] Gate script not found: ./scripts/missing.sh
```

## バリデーション結果のキャッシュ

Adapter ファイルの内容が変更されていない場合（ファイルハッシュで判定）、前回のバリデーション結果を再利用してよい。

```bash
md5sum aidlc-docs/config/project-adapter.md
# キャッシュ: .claude/.adapter-validation-cache
```
