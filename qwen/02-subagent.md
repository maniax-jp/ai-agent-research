# Subagent（サブエージェント）- タスクの委任と並列処理

**作成者:** Qwen
**日付:** 2026 年 3 月 1 日

---

## 目次

1. [概要](#概要)
2. [サブエージェントのアーキテクチャ](#サブエージェントのアーキテクチャ)
3. [ハンズオン：Claude Code のサブエージェント](#ハンズオンclaude-code のサブエージェント)
4. [進階：カスタムサブエージェントの作成](#進階カスタムサブエージェントの作成)
5. [パターンとベストプラクティス](#パターンとベストプラクティス)

---

## 概要

**Subagent（サブエージェント）**は、メインエージェントが複雑なタスクを専門化されたサブエージェントに委任するパターンです。2026 年以降、`Planner → Worker → Reviewer` の 3 層構造が実装テンプレート化されています。

### 主要なメリット

- **並列処理:** 複数のタスクを同時に実行
- **専門化:** 各エージェントが特定の仕事に特化
- **コンテキスト分離:** 各エージェントが独立したコンテキストを保有
- **スケーラビリティ:** 複雑なタスクを分割して処理
- **監査可能性:** 各 subagent の成果物をファイルに残す

### 設計ポイント

- 各 subagent の責務を 1 つに限定する
- コンテキスト共有は最小限（必要情報のみ）
- 失敗時の再実行単位を subagent 単位で定義する
- 各 subagent の成果物をファイルに残す（監査可能性）

---

## サブエージェントのアーキテクチャ

### 基本構造

```
メインエージェント（オーケストレーター）
    │
    ├─→ サブエージェント 1（リサーチ）
    ├─→ サブエージェント 2（コード作成）
    ├─→ サブエージェント 3（テスト）
    └─→ サブエージェント 4（レビュー）
```

### コミュニケーションフロー

1. **タスク分解:** メインエージェントがタスクをサブタスクに分割
2. **委任:** 各サブタスクを適切なサブエージェントに委任
3. **並列実行:** サブエージェントが独立して作業
4. **結果統合:** メインエージェントが結果を統合

---

## ハンズオン：Claude Code のサブエージェント

### ステップ 1: 環境準備

```bash
# プロジェクトディレクトリを作成
mkdir subagent-demo
cd subagent-demo

# Claude Code を起動
claude
```

### ステップ 2: シンプルなサブエージェント使用

Claude Code でサブエージェントを起動:

```
# プロンプト例
このプロジェクトのコードベースを調査するサブエージェントを起動してください
```

Claude は自動的にサブエージェントを起動し、以下の作業を行います：

1. プロジェクト構造の調査
2. 主要ファイルの分析
3. 依存関係のマッピング
4. レポートの生成

### ステップ 3: 明示的なサブエージェント委任

複数のサブエージェントを明示的に起動：

```
# プロンプト例
以下のタスクを実行するサブエージェントを起動してください：
1. バックエンド：Express.js ルートを作成
2. フロントエンド：React コンポーネントを作成
3. テスト：統合テストを書く
4. ドキュメント：API ドキュメントを作成
```

### ステップ 4: 出力の確認

各サブエージェントの出力を確認：

```bash
# 各サブエージェントの結果
- backend-agent: src/routes/*.js を作成
- frontend-agent: src/components/*.tsx を作成
- test-agent: tests/*.test.js を作成
- docs-agent: docs/api.md を作成
```

---

## 進階：3 層構成の実装（Planner → Worker → Reviewer）

codex のドキュメントを参照した、標準的な 3 層構成の実装例です。

### ステップ 4.5: subagent 定義ディレクトリを作成

```bash
mkdir -p .agents
```

### ステップ 4.6: planner.md を作成

`.agents/planner.md` を作成：

```markdown
# Planner

## 役割

- 入力要件をタスクへ分解
- Worker に渡すチェックリストを出力

## 出力フォーマット

```json
{
  "task_id": "T-001",
  "inputs": ["spec.md"],
  "acceptance_criteria": ["tests_pass", "no_security_regression"],
  "tasks": [
    {"id": "T-001-1", "description": "ユーザー認証機能を実装"},
    {"id": "T-001-2", "description": "セッション管理を実装"}
  ]
}
```
```

### ステップ 4.7: worker.md を作成

`.agents/worker.md` を作成：

```markdown
# Worker

## 役割

- Planner のチェックリストに従い実装
- 変更ファイルと理由を記録

## 出力フォーマット

```json
{
  "task_id": "T-001-1",
  "changes": [
    {"file": "src/auth.ts", "reason": "認証ロジック追加"},
    {"file": "src/auth.test.ts", "reason": "テスト追加"}
  ],
  "status": "completed"
}
```
```

### ステップ 4.8: reviewer.md を作成

`.agents/reviewer.md` を作成：

```markdown
# Reviewer

## 役割

- 仕様一致、回帰、テスト不足を検査
- Fail 時は Worker へ差し戻し理由を返却

## 出力フォーマット

```json
{
  "task_id": "T-001-1",
  "status": "pass" | "fail",
  "findings": [
    {"severity": "critical" | "high" | "medium", "message": "..."}
  ],
  "recommendation": "approve" | "reject" | "request_changes"
}
```
```

### ステップ 4.9: 成果物フォルダを用意

```bash
mkdir -p .runs/T-001
```

各 subagent の成果物を保存：
- Planner: `.runs/T-001/plan.md`
- Worker: `.runs/T-001/changes.md`
- Reviewer: `.runs/T-001/review.md`

### ステップ 4.10: 判定ゲート

```bash
# PowerShell 例
if (Select-String -Path .runs/T-001/review.md -Pattern "CRITICAL") {
  Write-Error "Release blocked"
}
```

## 期待結果

- 不具合時にどの層で失敗したか特定できる
- 局所リトライで再実行コストを下げられる

## 失敗しやすい点

- Planner が実装詳細まで書きすぎて Worker 責務と衝突
- Reviewer 基準が曖昧で、同じ変更の合否が毎回変わる
- 成果物が会話ログにしか残らず、再現不能になる

## 次の強化

- subagent ごとに利用可能ツールを明示
- 受け渡し JSON をスキーマ検証

---

## 進階：カスタムサブエージェントの作成

### ステップ 5: カスタムサブエージェントの定義

`.claude/subagents/` ディレクトリを作成：

```bash
mkdir -p .claude/subagents
```

### ステップ 6: リサーチエージェントの作成

`.claude/subagents/researcher.md` を作成：

```markdown
---
name: researcher
description: 技術調査と情報収集を行う専門エージェント
model: claude-opus-4-6
max_turns: 20
isolation: worktree
---

# リサーチエージェント

## 役割

あなたは技術調査の専門家です。与えられたトピックについて包括的な調査を行い、実用的な情報を提供します。

## ワークフロー

1. **トピックの理解**
   - 調査対象を明確に定義
   - 関連するドメインを特定

2. **情報収集**
   - ドキュメントの調査
   - ソースコードの分析
   - ベストプラクティスの特定

3. **分析**
   - 収集した情報の整理
   - パターンの特定
   - 推奨事項の導出

4. **レポート作成**
   - 調査結果のドキュメント化
   - 具体例の提供
   - 参考文献の記載

## 出力形式

調査結果は以下の形式で提供：

```markdown
# [トピック] 調査レポート

## 概要

[簡潔な概要]

## 主要発見

- [発見 1]
- [発見 2]
- [発見 3]

## 推奨事項

1. [推奨 1]
2. [推奨 2]

## 具体例

```
[コード例]
```

## 参考文献

- [リソース 1]
- [リソース 2]
```

## 制約

- 最新の情報に重点を置く
- 実用的な情報に焦点を当てる
- ソースを明記する
```

### ステップ 7: コーディングエージェントの作成

`.claude/subagents/coder.md` を作成：

```markdown
---
name: coder
description: コード実装を行う専門エージェント
model: claude-sonnet-4-6
max_turns: 30
isolation: worktree
---

# コーディングエージェント

## 役割

あなたはコード実装の専門家です。要件に基づいて高品質なコードを実装します。

## ワークフロー

1. **要件の理解**
   - 機能要件の明確化
   - 非機能要件の特定

2. **設計**
   - アーキテクチャの計画
   - 関数シグネチャの定義

3. **実装**
   - コードの作成
   - エラーハンドリング
   - エッジケースの対応

4. **テスト**
   - ユニットテストの作成
   - 統合テストの作成

## コーディング規約

- TypeScript を使用
- 関数型プログラミングの原則を適用
- 明確なエラーメッセージ
- 包括的なコメント

## 出力

- 実装コード
- テストコード
- 使用ドキュメント
```

### ステップ 8: メインオーケストレーターの使用

メインの Claude セッションでサブエージェントを起動：

```bash
claude
```

```
# プロンプト例
@.claude/subagents/researcher.md を使用して、
「2026 年の React ベストプラクティス」について調査してください

その後、@.claude/subagents/coder.md を使用して、
調査結果に基づいて React コンポーネントを実装してください
```

---

## パターンとベストプラクティス

### パターン 1: 探索 - 計画 - 実装

```
探索エージェント → 計画エージェント → 実装エージェント
```

**使用例:**

```markdown
# 探索フェーズ
@.claude/subagents/explorer.md
プロジェクトのコードベースを探索し、関連ファイルを特定

# 計画フェーズ
@.claude/subagents/planner.md
探索結果に基づいて実装計画を作成

# 実装フェーズ
@.claude/subagents/coder.md
計画に基づいてコードを実装
```

### パターン 2: 並列専門化

```
メインエージェント
    ├─→ フロントエンドエージェント
    ├─→ バックエンドエージェント
    ├─→ データベースエージェント
    └─→ テストエージェント
```

**使用例:**

```markdown
# 並列実行
以下のエージェントを同時に起動：
- @.claude/subagents/frontend.md - UI コンポーネント
- @.claude/subagents/backend.md - API エンドポイント
- @.claude/subagents/database.md - スキーマ設計
- @.claude/subagents/tester.md - テストスイート
```

### パターン 3: ワイター - レビュアー

```
ライターエージェント → レビュアーエージェント → 修正ループ
```

**使用例:**

```markdown
# ライター
@.claude/subagents/writer.md
コードを実装

# レビュアー
@.claude/subagents/reviewer.md
コードをレビューし、改善点を指摘

# 修正
@.claude/subagents/writer.md
レビューに基づいて修正
```

### パターン 4: Worktree 隔離

各サブエージェントに独立した Git worktree を提供：

```markdown
---
name: isolated-coder
description: 隔離された環境でコードを実装
isolation: worktree
---

# 隔離コーディングエージェント

## 特徴

- 独立した Git worktree
- メインブランチへの影響なし
- 完了後自動的にクリーンアップ
```

---

## Worktree を使った並列サブエージェント

### ステップ 9: Worktree 設定

`.claude/subagents/parallel-dev.md` を作成：

```markdown
---
name: parallel-dev
description: Worktree を使用した並列開発
model: claude-opus-4-6
max_turns: 50
isolation: worktree
---

# 並列開発エージェント

## 役割

あなたは複数の機能を並列で開発する専門家です。

## Worktree ワークフロー

1. **Worktree 作成**
   ```bash
   git worktree add ../.claude/worktrees/feature-x feat/feature-x
   ```

2. **並列開発**
   - 各 Worktree で独立して開発
   - 干渉なし

3. **結果マージ**
   ```bash
   git checkout main
   git merge feat/feature-x
   ```

## 使用例

```bash
# 複数の機能を並列で開発
claude --worktree feature-auth "認証機能を実装"
claude --worktree feature-profile "プロフィール機能を実装"
claude --worktree feature-settings "設定機能を実装"
```
```

### ステップ 10: 実行

```bash
# 並列サブエージェントを起動
claude @.claude/subagents/parallel-dev.md

# プロンプト
以下の機能を並列で実装してください：
1. ユーザー認証
2. プロフィール管理
3. 設定画面
```

---

## トラブルシューティング

### 問題 1: サブエージェントが応答しない

**解決:**

```bash
# サブエージェントファイルを確認
cat .claude/subagents/researcher.md

# フロントマターを確認
# ---
# name: researcher
# description: ...
# ---
```

### 問題 2: コンテキストが共有されない

**解決:**

```markdown
# メインエージェントで明示的に共有
@.claude/subagents/researcher.md の結果を
@.claude/subagents/coder.md に渡す
```

### 問題 3: Worktree のクリーンアップ

**解決:**

```bash
# 使用済みの Worktree を削除
git worktree remove ../.claude/worktrees/feature-x

# 自動クリーンアップを有効化
# .claude/subagents/xxx.md に isolation: worktree を設定
```

---

## まとめ

サブエージェントパターンは、複雑なタスクを効率的に処理するための強力な手法です。適切な委任と並列処理により、AI エージェントの生産性を大幅に向上できます。

### 次のステップ

1. [AGENTS.md](./03-agents.md.md) - プロジェクト仕様書
2. [Agent Teams](./05-agent-teams.md) - マルチエージェントオーケストレーション
3. [Git Worktree](./07-git-worktree.md) - 並列実行の詳細

---

*関連リソース:*
- [Claude Code Subagents](https://code.claude.com/docs)
- [Anthropic Agent Teams](https://docs.anthropic.com/)
- [Multi-Agent Patterns](https://www.anthropic.com/research)