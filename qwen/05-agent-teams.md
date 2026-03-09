# Agent Teams - Claude のマルチエージェントオーケストレーション

**作成者:** Qwen
**日付:** 2026 年 3 月 9 日

---

## 目次

1. [概要](#概要)
2. [Agent Teams のアーキテクチャ](#agent-teams のアーキテクチャ)
3. [ハンズオン：Agent Teams の使用](#ハンズオンagent-teams の使用)
4. [進階：カスタムオーケストレーション](#進階カスタムオーケストレーション)
5. [ベストプラクティス](#ベストプラクティス)

---

## 概要

**Agent Teams**は、Claude Opus 4.6 に搭載されたマルチエージェントオーケストレーションシステムです。単一の汎用 AI ではなく、複数の専門エージェントを並列で実行し、協調して複雑なタスクを完了します。

### 主要な特徴

- **専門化されたエージェント:** 各エージェントが特定の役割に特化
- **並列実行:** 複数のエージェントが同時に作業
- **独立した観測:** 各エージェントの作業を個別に監視可能
- **協調出力:** 結果を統合して最終出力を生成

### サブエージェントとの違い

| 特徴 | サブエージェント | Agent Teams |
|------|-----------------|-------------|
| **可視性** | 内部処理は非表示 | 各エージェントを個別に観測可能 |
| **介入** | 最終結果のみ | 中間段階で介入可能 |
| **通信** | 親エージェント経由 | エージェント間直接通信 |
| **使用ケース** | シンプルな委任 | 複雑な協調タスク |

---

## Agent Teams のアーキテクチャ

### 基本構造

```
スーパーバイザーエージェント（オーケストレーター）
    │
    ├─→ リサーチエージェント [独立パネル]
    ├─→ ストラテジストエージェント [独立パネル]
    ├─→ コピーライターエージェント [独立パネル]
    ├─→ レビュアーエージェント [独立パネル]
    └─→ 統合エージェント [結果統合]
```

### コミュニケーションフロー

1. **タスク受信:** スーパーバイザーがタスクを受信
2. **タスク分解:** タスクをサブタスクに分割
3. **エージェント割り当て:** 各サブタスクを適切なエージェントに割り当て
4. **並列実行:** エージェントが同時に作業
5. **中間共有:** エージェント間で中間結果を共有
6. **結果統合:** 統合エージェントが最終出力を生成

---

## ハンズオン：Agent Teams の使用

### ステップ 1: 環境準備

```bash
# Claude Code を起動
claude

# または、Claude Opus 4.6 にアクセス
```

### ステップ 2: シンプルな Agent Teams の起動

```
# プロンプト例
Agent Teams を使用して、以下のタスクを実行してください：
- リサーチ：AI エージェントの最新動向を調査
- 分析：調査結果を分析
- 執筆：調査レポートを作成
- レビュー：レポートをレビュー
```

Claude は自動的に Agent Teams を起動し、以下のパネルが表示されます：

```
┌─────────────────────────────────────────────────────────┐
│ Superivisor Agent                                       │
│ Task: AI エージェント動向調査                            │
│ Status: In Progress                                     │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ Research Agent                                          │
│ Role: 情報収集                                           │
│ Status: Searching sources...                            │
│ Progress: 45%                                            │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ Analyst Agent                                           │
│ Role: 結果分析                                           │
│ Status: Waiting for research data...                    │
│ Progress: 0%                                             │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ Writer Agent                                            │
│ Role: レポート執筆                                       │
│ Status: Waiting for analysis...                         │
│ Progress: 0%                                             │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ Reviewer Agent                                          │
│ Role: 品質レビュー                                       │
│ Status: Waiting for draft...                            │
│ Progress: 0%                                             │
└─────────────────────────────────────────────────────────┘
```

### ステップ 3: 明示的な Agent Teams の定義

より制御が必要な場合、明示的にエージェントを定義：

```
# プロンプト例
以下の Agent Teams を作成して実行してください：

チーム構成：
1. リサーチエージェント
   - 役割：技術調査
   - 出力：調査結果ドキュメント

2. アーキテクトエージェント
   - 役割：システム設計
   - 出力：アーキテクチャ図と仕様

3. 開発エージェント
   - 役割：コード実装
   - 出力：ソースコード

4. テスターエージェント
   - 役割：テスト作成
   - 出力：テストスイート

タスク：
「TypeScript で REST API を作成」
```

### ステップ 4: エージェント間の協調

Agent Teams は自動的にエージェント間通信を管理：

```
# 通信フローの例

Research Agent:
> 調査完了：Next.js 14 の最新機能を確認

Architect Agent:
> Next.js 14 の App Router を採用
> API ルートは /api/* で実装

Developer Agent:
> API ルートを作成中...
> src/app/api/users/route.ts

Tester Agent:
> テスト作成中...
> src/app/api/users/route.test.ts
```

### ステップ 5: 中間結果の確認

各エージェントの中間結果を確認：

```
# リサーチエージェントの結果
> 調査結果：
> - Next.js 14: App Router, Server Components
> - TypeScript 5.4: Decorators, Better inference
> - Express.js: 安定版 4.18

# アーキテクトエージェントの結果
> アーキテクチャ：
> - フロントエンド：Next.js 14 + React 18
> - バックエンド：Next.js API Routes
> - データベース：PostgreSQL + Prisma
```

### ステップ 6: 個別エージェントへの介入

必要に応じて個別のエージェントに指示：

```
# リサーチエージェントに追加調査を依頼
@research-agent より詳細なベンチマークデータを調査してください

# 開発エージェントに修正を依頼
@developer-agent エラーハンドリングを追加してください
```

---

## 進階：カスタムオーケストレーション

### ステップ 7: カスタムエージェント定義

`.claude/agent-teams/` ディレクトリを作成：

```bash
mkdir -p .claude/agent-teams
```

### ステップ 8: エージェントテンプレートの作成

`.claude/agent-teams/content-team.yaml` を作成：

```yaml
# コンテンツ作成チーム
name: content-team
description: コンテンツ作成のための Agent Teams

supervisor:
  name: supervisor
  model: claude-opus-4-6
  instructions: |
    あなたはコンテンツ作成チームのスーパーバイザーです。
    タスクを適切に分割し、各エージェントに委任します。

agents:
  - name: researcher
    role: リサーチャー
    model: claude-sonnet-4-6
    instructions: |
      あなたは調査の専門家です。
      与えられたトピックについて包括的な調査を行います。
    outputs:
      - research-report.md

  - name: strategist
    role: ストラテジスト
    model: claude-sonnet-4-6
    instructions: |
      あなたはコンテンツ戦略の専門家です。
      調査結果に基づいてコンテンツ戦略を策定します。
    inputs:
      - research-report.md
    outputs:
      - content-strategy.md

  - name: writer
    role: ライター
    model: claude-sonnet-4-6
    instructions: |
      あなたはプロのライターです。
      戦略に基づいて高品質なコンテンツを作成します。
    inputs:
      - content-strategy.md
    outputs:
      - content-draft.md

  - name: editor
    role: エディター
    model: claude-sonnet-4-6
    instructions: |
      あなたは編集者の専門家です。
      コンテンツを校正し、品質を向上させます。
    inputs:
      - content-draft.md
    outputs:
      - content-final.md

workflow:
  - stage: research
    agents: [researcher]
    parallel: false

  - stage: strategy
    agents: [strategist]
    parallel: false
    depends_on: [research]

  - stage: writing
    agents: [writer]
    parallel: false
    depends_on: [strategy]

  - stage: editing
    agents: [editor]
    parallel: false
    depends_on: [writing]
```

### ステップ 9: カスタムチームの起動

```
# プロンプト例
@.claude/agent-teams/content-team.yaml を使用して、
「AI エージェントの未来」について記事を作成してください
```

### ステップ 10: 並列ワークフローの定義

`.claude/agent-teams/parallel-dev.yaml` を作成：

```yaml
# 並列開発チーム
name: parallel-dev
description: 並列開発のための Agent Teams

supervisor:
  name: supervisor
  model: claude-opus-4-6

agents:
  - name: frontend-dev
    role: フロントエンド開発者
    model: claude-sonnet-4-6
    worktree: feature-frontend

  - name: backend-dev
    role: バックエンド開発者
    model: claude-sonnet-4-6
    worktree: feature-backend

  - name: db-designer
    role: データベース設計者
    model: claude-sonnet-4-6
    worktree: feature-database

  - name: tester
    role: テスター
    model: claude-sonnet-4-6
    worktree: feature-tests

workflow:
  - stage: design
    agents: [db-designer]
    parallel: true

  - stage: development
    agents: [frontend-dev, backend-dev]
    parallel: true
    depends_on: [design]

  - stage: testing
    agents: [tester]
    parallel: true
    depends_on: [development]

  - stage: integration
    agents: [supervisor]
    parallel: false
    depends_on: [testing]
```

---

## ベストプラクティス

### 1. 適切なエージェント数の選択

**推奨:** 3-5 エージェント

- **少なさすぎ:** 各エージェントの負担过大
- **多すぎ:** コーディネーションコストの増大

```
# 良い例（4 エージェント）
- リサーチャー
- アーキテクト
- 開発者
- テスター

# 悪い例（8 エージェント）
- リサーチャー
- バックエンドリサーチャー
- フロントエンドリサーチャー
- DB リサーチャー
- アーキテクト
- バックエンド開発者
- フロントエンド開発者
- テスター
```

### 2. 明確な役割定義

各エージェントの役割を明確に定義：

```yaml
agents:
  - name: researcher
    role: リサーチャー
    responsibilities:
      - 情報収集
      - ソースの検証
      - 調査結果のドキュメント化
    not_responsible:
      - コード実装
      - 意思決定
```

### 3. 入出力の明示

エージェント間の入出力を明示：

```yaml
agents:
  - name: researcher
    outputs:
      - research-findings.md
      - source-list.md

  - name: architect
    inputs:
      - research-findings.md
    outputs:
      - architecture-design.md
```

### 4. 並列化の活用

可能な限り並列化：

```yaml
workflow:
  # 悪い例： sequential
  - stage: research
    agents: [researcher]
  - stage: strategy
    agents: [strategist]
  - stage: design
    agents: [designer]

  # 良い例： parallel
  - stage: research
    agents: [researcher, competitor-analyst]
    parallel: true
  - stage: planning
    agents: [strategist, designer]
    parallel: true
    depends_on: [research]
```

### 5. エラーハンドリング

エラー処理を定義：

```yaml
error_handling:
  retry:
    max_attempts: 3
    backoff: exponential

  fallback:
    on_failure: supervisor
    escalate_after: 2
```

---

## トラブルシューティング

### 問題 1: エージェントが応答しない

**解決:**

```
# エージェントの状態を確認
@supervisor-agent エージェントの状態を確認してください

# 再起動
@supervisor-agent エージェントを再起動してください
```

### 問題 2: 結果が整合性がない

**解決:**

```
# 統合エージェントに修正を依頼
@integrator-agent 結果の整合性を確認して修正してください

# スーパーバイザーに再調整を依頼
@supervisor-agent エージェント間の調整を再実行してください
```

### 問題 3: 実行時間が長い

**解決:**

```
# 並列化を増やす
workflow:
  - stage: development
    agents: [frontend, backend, database]
    parallel: true  # 追加

# モデルを高速なモデルに切り替え
agents:
  - name: researcher
    model: claude-haiku-4-5  # より高速
```

---

## まとめ

Agent Teams は、複雑なタスクを複数の専門エージェントで効率的に処理するための強力なシステムです。適切なオーケストレーションにより、生産性と品質を大幅に向上できます。

### 次のステップ

1. [Context Fork](./06-context-fork.md) - コンテキストの分岐パターン
2. [Git Worktree](./07-git-worktree.md) - 並列実行の詳細
3. [MCP](./08-mcp.md) - 標準接続プロトコル

---

*関連リソース:*
- [Claude Agent Teams Documentation](https://docs.anthropic.com/)
- [Multi-Agent Orchestration Patterns](https://www.anthropic.com/research)
- [Agent Teams Examples](https://github.com/anthropic/agent-teams-examples)