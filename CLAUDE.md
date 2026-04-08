# AI-DLC プロジェクト設定

このリポジトリは **AI-DLC（AI-Driven Development Lifecycle）** のワークフローに従って開発を進めます。

## 基本原則

- **AI が動き、人間がハンドルを握る** — 要件定義・設計・実装・テストはAIが担当し、人間はビジョンの承認・方向修正に集中する
- **承認ゲートを必ず守る** — 各フェーズの終わりに人間の承認なしに次フェーズへ進まない
- **過信防止** — 不確実な点は自律判断せず、必ず人間に確認する
- **成果物は差分更新** — 既存ファイルを削除・再作成せず、編集で更新する

## ドキュメント構造

すべての成果物は `aidlc-docs/` 配下に保存する。プロジェクト開始時にテンプレート `templates/ai-dlc docs/` をコピーして使用する。

```
aidlc-docs/
├── aidlc-state.md          # 全フェーズの進捗チェックボックス（常に最新に保つ）
├── audit.md                # ユーザー発言の原文記録（要約・言い換え禁止）
├── codebase-snapshot.json  # コードベース分析結果（Brownfieldのみ）
├── config/
│   ├── github-projects.json  # GitHub Projects v2 設定
│   └── project-adapter.md    # Bolt パイプライン設定
├── inception/
│   ├── intent.md
│   ├── requirements.md
│   ├── user-stories.md
│   └── units.md
├── blueprint/
│   ├── functional-design.md
│   ├── architecture.md
│   └── units/{unit-name}/
├── eval/
└── quality gate/
```

## スラッシュコマンド一覧

| コマンド | フェーズ | 説明 |
|---------|---------|------|
| `/inception` | Phase 1 | アイデア → 要件定義・ユーザーストーリー・Unit分割 |
| `/blueprint` | Phase 2 | 要件 → 機能設計・アーキテクチャ・Bolt Plan |
| `/construction` | Phase 3 | Bolt Plan → プロダクションコード実装 |
| `/quality-gate` | Phase 4 | コード品質・セキュリティ検証・Go/No-Go判断 |
| `/run-bolt` | Phase 3 内 | 単一Boltをパイプライン実行（スキル） |

## セッション開始時の確認事項

1. `aidlc-docs/aidlc-state.md` を読んで現在のフェーズ・ステップを把握する
2. 対応するコマンドを使って作業を開始する
3. 各ステップ完了後に `aidlc-state.md` のチェックボックスを更新する
4. 人間の承認が必要な場面では必ず待つ

## 横断的なルール

- **audit.md** にはユーザーの発言を**原文のまま**記録する（要約・言い換え禁止）
- フィードバックループで前フェーズに戻る場合は `overview.md` の手順に従う
- Brownfieldプロジェクトでは最初に `/inception` を実行し、Step 0 のコードベース分析から開始する
