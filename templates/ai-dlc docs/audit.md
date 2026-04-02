# AI-DLC Audit Log

> プロジェクトの全対話・意思決定の記録

---

## Phase 1: Inception（構想）

### Step 0: Codebase Analysis（コードベース分析）※Brownfield のみ

**開始**: YYYY-MM-DD HH:MM

#### 分析結果サマリー

- プロジェクト種別: Greenfield / Brownfield
- （Brownfield の場合）技術スタック・主要構成・既存機能の概要

#### 対話記録

> **人間**:（原文のまま記録。要約・言い換え禁止）
>
> **AI**:（要約でよい）

#### 生成した成果物

- `codebase-snapshot.json` - 作成/更新

**完了**: YYYY-MM-DD HH:MM

---

### Step 1: Intent（意図の明確化）

**開始**: YYYY-MM-DD HH:MM

#### 対話記録

> **人間**:（原文のまま記録。要約・言い換え禁止）
>
> **AI**:（要約でよい）
>
> ...

#### 決定事項

- 決定1: ...
- 決定2: ...

#### 生成した成果物

- `inception/intent.md` - 作成

**完了**: YYYY-MM-DD HH:MM

---

### Step 2: Exploration（機能探索）

**開始**: YYYY-MM-DD HH:MM

#### 対話記録

> **人間**:（原文のまま記録。要約・言い換え禁止）
>
> **AI**:（要約でよい）
>
> ...

#### 決定事項

- ...

#### 生成した成果物

- `inception/intent.md` - 追記

**完了**: YYYY-MM-DD HH:MM

---

### Step 3: Requirements（要件の具体化）

**開始**: YYYY-MM-DD HH:MM

#### 対話記録

> **人間**:（原文のまま記録。要約・言い換え禁止）
>
> **AI**:（要約でよい）
>
> ...

#### 決定事項

- ...

#### 生成した成果物

- `inception/requirements.md` - 作成
- `inception/user-stories.md` - 作成

**完了**: YYYY-MM-DD HH:MM

---

### Step 4: Visualize（ビジュアライズ）

**開始**: YYYY-MM-DD HH:MM

#### 対話記録

> **人間**:（原文のまま記録。要約・言い換え禁止）
>
> **AI**:（要約でよい）
>
> ...

#### 決定事項

- ...

#### 生成した成果物

- デザインファイル（.pen） - 作成

**完了**: YYYY-MM-DD HH:MM

---

### Step 5: QAプラン（QA Plan）

**開始**: YYYY-MM-DD HH:MM

#### 対話記録

> **人間**:（原文のまま記録。要約・言い換え禁止）
>
> **AI**:（要約でよい）
>
> ...

**完了**: YYYY-MM-DD HH:MM

---

### Step 6: Approval Gate（承認）

**開始**: YYYY-MM-DD HH:MM

#### 対話記録

> **人間**:（原文のまま記録。要約・言い換え禁止）
>
> **AI**:（要約でよい）
>
> ...

#### 承認結果

- [ ] 承認 / フィードバックあり
- フィードバック時: 戻り先と理由を記録

**完了**: YYYY-MM-DD HH:MM

---

### Step 7: Unit分割（Unit Decomposition）

**開始**: YYYY-MM-DD HH:MM

#### 対話記録

> **人間**:（原文のまま記録。要約・言い換え禁止）
>
> **AI**:（要約でよい）
>
> ...

#### 決定事項

- ...

#### 生成した成果物

- `inception/units.md` - 作成

**完了**: YYYY-MM-DD HH:MM

---

## Phase 2: Blueprint（設計）

### Step 0: GitHub Projects v2 セットアップ

**開始**: YYYY-MM-DD HH:MM

#### 対話記録

> **人間**:（原文のまま記録。要約・言い換え禁止）
>
> **AI**:（要約でよい）

#### 決定事項

- ...

#### 生成した成果物

- `config/github-projects.json` - 作成/更新

**完了**: YYYY-MM-DD HH:MM

---

<!-- Unit ごとに以下のセクション（Step 1〜3）を繰り返す。inception/units.md から Unit 一覧を読み込み、Unit 数分コピーして展開する。 -->

### Unit: {unit-name}

#### Step 1: Functional Design（機能設計）

**開始**: YYYY-MM-DD HH:MM

##### 対話記録

> **人間**:（原文のまま記録。要約・言い換え禁止）
>
> **AI**:（要約でよい）

##### 決定事項

- ...

##### 生成した成果物

- `blueprint/units/{unit-name}/functional-design.md` - 作成
- `blueprint/functional-design.md` - 作成/更新

**完了**: YYYY-MM-DD HH:MM

---

#### Step 2: Architecture（アーキテクチャ設計）

**開始**: YYYY-MM-DD HH:MM

##### 対話記録

> **人間**:（原文のまま記録。要約・言い換え禁止）
>
> **AI**:（要約でよい）

##### 決定事項

- ...

##### 生成した成果物

- `blueprint/units/{unit-name}/architecture.md` - 作成
- `blueprint/architecture.md` - 作成/更新

**完了**: YYYY-MM-DD HH:MM

---

#### Step 3: Bolt Plan（Bolt 分割計画）— Unit 承認

**開始**: YYYY-MM-DD HH:MM

##### 対話記録

> **人間**:（原文のまま記録。要約・言い換え禁止）
>
> **AI**:（要約でよい）

##### 承認結果

- [ ] 承認 / フィードバックあり

##### 生成した成果物

- `blueprint/units/{unit-name}/bolt-plan.md` - 作成

**完了**: YYYY-MM-DD HH:MM

---

### Step 4: Blueprint Approval Gate（全体承認）

**開始**: YYYY-MM-DD HH:MM

#### 対話記録

> **人間**:（原文のまま記録。要約・言い換え禁止）
>
> **AI**:（要約でよい）

#### 承認結果

- [ ] 承認 / フィードバックあり
- フィードバック時: 戻り先と理由を記録

#### 生成した成果物

- `config/project-adapter.md` - 作成/更新

**完了**: YYYY-MM-DD HH:MM

---

## Phase 3: Construction（実装）

### Step 1: Execution（実行）

<!-- 自動実行ステップ。エスカレーション等の人間とのやり取りがあった場合に記録する。 -->

#### エスカレーション記録

- ...

**完了**: YYYY-MM-DD HH:MM

---

### Step 2: Approval Gate（承認）

**開始**: YYYY-MM-DD HH:MM

#### 対話記録

> **人間**:（原文のまま記録。要約・言い換え禁止）
>
> **AI**:（要約でよい）

#### 承認結果

- [ ] 承認 / フィードバックあり
- フィードバック時: 戻り先と理由を記録

**完了**: YYYY-MM-DD HH:MM

---

## Phase 4: Quality Gate（品質関門）

### Step 1: Quality Check & Auto-Fix

<!-- 自動実行ステップ。記録は quality-gate/report.md に出力される。 -->

**完了**: YYYY-MM-DD HH:MM

---

### Step 2: Approval Gate（Go/No-Go）

**開始**: YYYY-MM-DD HH:MM

#### 対話記録

> **人間**:（原文のまま記録。要約・言い換え禁止）
>
> **AI**:（要約でよい）

#### 判断結果

- [ ] Go / No-Go
- No-Go 時: 理由と戻り先を記録

**完了**: YYYY-MM-DD HH:MM
