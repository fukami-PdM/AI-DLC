# Project Adapter

<!-- AI-DLC の Bolt パイプラインをプロジェクト固有の技術スタックに接続する設定ファイル。
     テンプレートからコピーし、プロジェクトに合わせて埋めてください。 -->

> このファイルは `aidlc-docs/config/project-adapter.md` に配置し、Bolt パイプラインの実行時に自動的に読み込まれる。

## Workstreams

<!-- 各 Workstream は並列実行可能な実装エージェントに対応する。
     プロジェクトの技術スタックに合わせて Workstream を定義する。
     最低1つの Workstream が必要。 -->

### <workstream-name>

- **Agent**: `<agent-subagent-type>`
- **Trigger paths**: `<glob patterns that indicate this workstream is needed>`
- **Owned paths**: `<glob patterns this agent is allowed to edit>`
- **Agent definition**: `.claude/agents/<agent-name>.md`
- **Skills** (execute in order, skip N/A):
  <!-- プロジェクト固有の Claude Code Skills を実行順に列挙する。
       Skills はプロジェクトの .claude/skills/ に配置する。 -->
  1. `<skill-name>` (<when to use>)

<!-- 必要に応じて Workstream を追加する。例:
### backend
### frontend
### mobile
### infrastructure
-->

## Integrator

<!-- 2つ以上の Workstream が同時に必要な Bolt で、統合エージェントを使用する場合に設定する。
     不要な場合はセクションごと削除してよい。 -->

- **Agent**: `integrator`
- **Trigger**: 2+ workstreams needed for a single Bolt
- **Agent definition**: `.claude/agents/integrator.md`

## Workspace Setup

<!-- Worktree 作成後に実行する依存パッケージインストール等のコマンド。
     WORKTREE 変数にワークツリーの絶対パスが設定される。
     不要な場合はセクションごと削除してよい。 -->

```bash
# Example:
# (cd "${WORKTREE}/apps/server" && go mod download)
# (cd "${WORKTREE}" && npm install)
```

## Gate Commands

<!-- Bolt の実装後に実行する CI 相当のチェック。
     すべてパスすることが Ship の前提条件。
     プロジェクトのビルドツール・テストフレームワークに合わせて記述する。 -->

```bash
# Example:
# cd apps/server && make lint && make test && make build
# npm run typecheck
# npm run test
```

## Review Configuration

<!-- Gate 通過後、Ship 前に実行する AI レビュー。
     不要な場合はセクションごと削除してよい。 -->

- **Method**: `<skill name or "none">`
- **Reviewers**: <comma-separated reviewer types>
- **Max rounds**: <number>
