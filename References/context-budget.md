# Context Budget Reference

> Prepare が生成する Implementer プロンプトのサイズを管理し、Context Window 溢れによる品質劣化を防ぐ。

## 問題

Claude Code のコンテキストウィンドウには上限がある。Implementer プロンプトに設計文書を詰め込みすぎると：

- 重要な指示が埋もれる（Lost in the Middle 問題）
- Skill の制約が無視される
- テストや受入基準の漏れが発生する

## Budget 定義

### Implementer プロンプトの構成と目安

| セクション                    | 目安行数                     | 優先度   |
| ----------------------------- | ---------------------------- | -------- |
| WORKTREE, CONTEXT, BOLT ISSUE | 5 行                         | 必須     |
| SCOPE                         | 2-5 bullet points（5-15 行） | 必須     |
| ACCEPTANCE CRITERIA           | 10-30 行                     | 必須     |
| SKILL SEQUENCE                | 5-15 行                      | 必須     |
| PREVIOUS ERRORS               | 0-50 行                      | 条件付き |
| **合計目標**                  | **100 行以下**               | —        |

> Implementer プロンプト自体は小さく保つ。詳細なコンテキストは Context JSON とリポジトリ内の Skill ファイルが提供する。

### Context JSON のサイズ管理

Context JSON (`.bolt-context/i{intent}-b{N}.json`) は Implementer が参照するが、プロンプトに展開しない。サイズ制限は不要。命名規則の詳細は `.claude/skills/run-bolt/SKILL.md` を参照。

## 設計文書が大きい場合の要約戦略

### 判定基準

Prepare が設計文書を読み込んだ後、以下の場合に要約が必要：

- 単一ファイルが 300 行超
- 全設計文書の合計が 1000 行超

### 要約ルール

#### functional-design.md（機能設計）

- **抽出対象**: 対象 Bolt に関連するセクションのみ
- **除外**: 他の Bolt / Unit の機能記述、背景説明の詳細
- **保持**: ビジネスロジック（ユースケースの入力/出力/処理フロー）、バリデーションルール、状態遷移ルール

#### architecture.md（アーキテクチャ）

- **抽出対象**: 対象 Bolt が触るレイヤー（domain / repository / usecase / handler）の設計のみ
- **除外**: 他レイヤーの詳細、インフラ構成の全体図
- **保持**: レイヤー境界のルール、依存方向の制約

#### user-stories.md（ユーザーストーリー）

- **抽出対象**: 対象 Bolt にマッピングされた受入基準のみ
- **除外**: 他 Bolt の受入基準、ストーリーの背景詳細
- **保持**: 受入基準のテスト可能な記述（Given/When/Then）

#### bolt-plan.md（Bolt 計画）

- **抽出対象**: 対象 Bolt のセクション全体 + 依存 Bolt の成果物リスト
- **除外**: 他 Bolt の詳細実装計画
- **保持**: 成果物パス、依存関係

#### デザインファイル (.pen)

- **抽出対象**: 対象 Bolt が実装する画面のデザインのみ
- **除外**: 他画面のデザイン
- **保持**: コンポーネント構造、スタイル仕様、画面遷移

### 要約の適用場所

要約はすべて **Prepare (Step 1) の内部処理** として行う。要約結果は：

1. **SCOPE セクション** に 2-5 bullet points として反映
2. **ACCEPTANCE CRITERIA セクション** にテスト可能な基準として反映
3. **Context JSON** に構造化データとして格納（Implementer が必要時に参照）

Implementer プロンプトに設計文書の生テキストを含めない。

## Prepare の出力チェック

Prepare は Output Contract を返す前に以下を確認する：

1. 各 Implementer プロンプトが 100 行以下であること
2. SCOPE が 2-5 bullet points であること
3. ACCEPTANCE CRITERIA が具体的でテスト可能な記述であること（「〜できること」ではなく「〜した場合、〜が返ること」）
4. 設計文書の生テキストが含まれていないこと

> **100 行超の場合**: SCOPE か ACCEPTANCE CRITERIA を圧縮する。圧縮で失われる詳細は Context JSON に格納し、プロンプト内に `See context JSON for details: {path}` と記載する。
