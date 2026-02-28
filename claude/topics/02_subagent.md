# Subagent — コンテキスト分離・並列実行エージェント

**カテゴリ:** Claude Code / Agent SDK
**導入時期:** 2025年（Claude Code Agent SDK）
**ステータス:** 本番利用可能（2026年現在）

---

## 1. 概要

**Subagent** は、メインエージェントが専門的なサブタスクを委譲するための独立したエージェントインスタンスです。

### 主な用途

1. **コンテキスト分離** — メインの会話履歴を汚染せずに大量のファイルを処理
2. **並列実行** — 複数の独立したタスクを同時に実行して高速化
3. **権限制限** — サブタスクに必要な最小限のツールだけを付与
4. **専門化** — 特定ドメインに特化したシステムプロンプトを持つエージェント

### Claude Code に組み込まれているビルトイン Subagent

| Subagent | 用途 | 特徴 |
|----------|------|------|
| **Explore** | コードベース探索 | 読み取り専用、高速、Read/Grep/Glob のみ |
| **Plan** | 計画立案時のコード調査 | プランモード中に自動委譲 |
| **general-purpose** | 汎用タスク | フルツールアクセス |

### サブエージェントのネスト制限（孫エージェント）

```
オーケストレーター（メイン）
    ├─ Subagent A  ← OK（正式サポート）
    └─ Subagent B  ← OK
           └─ Subagent C  ← ⚠️ 原則として非サポート（Swarms実験機能で一部可能）
```

正式仕様ではサブエージェントがさらにサブエージェントを起動することはできません。ただし **2026年1月に `Swarms` という実験的機能**が発見され、feature flag で有効化することで階層的な協調が可能になります（詳細は後述）。

---

## 2. Planner → Worker → Reviewer パターン（2026年の標準テンプレート）

Codex が定式化した **3層構造** は、2026年における Subagent 設計の実装テンプレートとして定着しています。

```
┌─────────────────────────────────────────────────────┐
│           Planner                                    │
│  入力要件 → タスク分解 → Worker へのチェックリスト出力  │
└────────────────────────┬────────────────────────────┘
                         │ handoff.json
┌────────────────────────▼────────────────────────────┐
│           Worker                                     │
│  チェックリストに従い実装 → 変更ファイルと理由を記録     │
└────────────────────────┬────────────────────────────┘
                         │ result.json
┌────────────────────────▼────────────────────────────┐
│           Reviewer                                   │
│  仕様一致・回帰・テスト不足を検査                       │
│  Fail → Worker へ差し戻し / Pass → 完了              │
└─────────────────────────────────────────────────────┘
```

### 受け渡し契約（handoff.json）

サブエージェント間の受け渡しは **構造化JSONで記録**します。これにより失敗時にどの層で問題が起きたかが特定しやすくなります。

```json
{
  "task_id": "T-001",
  "from": "Planner",
  "to": "Worker",
  "inputs": ["spec.md"],
  "checklist": [
    "src/auth.ts に JWT発行ロジックを追加する",
    "refresh token の有効期限を7日に設定する",
    "src/auth.test.ts にテストを追加する"
  ],
  "acceptance_criteria": ["all_tests_pass", "no_security_regression"],
  "timestamp": "2026-02-28T10:00:00Z"
}
```

### 各エージェントの定義ファイル（YAML形式）

```yaml
# .claude/agents/planner.yaml
name: planner
description: >
  タスク要件を受け取り、Workerへのチェックリストを生成する。
  実装はしない。タスク分解と受け渡し契約の作成のみを担当する。
system_prompt: |
  あなたはタスクを分解するプランナーです。
  与えられた要件を具体的な実装タスクのチェックリストに変換し、
  acceptance_criteriaを明確化してください。
  出力は必ず handoff.json 形式にしてください。
allowed_tools:
  - Read
  - Glob
```

```yaml
# .claude/agents/worker.yaml
name: worker
description: >
  Plannerのチェックリストに従い実装を行う。
  変更したファイルと変更理由を記録する。
system_prompt: |
  あなたは実装担当のエンジニアです。
  チェックリストのタスクを1つずつ確実に実装してください。
  各変更について理由を result.json に記録してください。
allowed_tools:
  - Read
  - Write
  - Edit
  - Bash
```

```yaml
# .claude/agents/reviewer.yaml
name: reviewer
description: >
  Workerの実装結果をレビューする。仕様との一致・回帰・テスト不足を確認する。
  問題があれば差し戻し理由を添えてWorkerに返却する。
system_prompt: |
  あなたはコードレビュアーです。
  acceptance_criteriaを全て満たしているかを確認してください。
  Critical/Highの問題があればWorkerに差し戻します。
  問題なければ "APPROVED" を返してください。
allowed_tools:
  - Read
  - Grep
  - Glob
  - Bash
```

### 失敗時の再実行単位

```
タスク失敗時の差し戻しフロー:

Reviewer → "NEEDS_REVISION: src/auth.ts の refresh token 実装が仕様と不一致"
    │
    ▼
Worker（差し戻し理由を含む再実行）
    ├─ 前回の handoff.json を参照
    ├─ 差し戻し理由をコンテキストに追加
    └─ 修正版を実装して再度 Reviewer へ
```

再実行コストを最小化するため、**Planner は変更せず Worker→Reviewer のループだけ**を繰り返します。

---

## 3. Claude Agent SDK でのカスタム Subagent 定義

### Subagent の設定フィールド

```typescript
// Claude Agent SDK（TypeScript）
const subagentConfig = {
  name: "doc-reviewer",
  description: "ドキュメントファイルの品質を確認する専用エージェント",

  // システムプロンプト：このエージェント固有の指示
  systemPrompt: `あなたはテクニカルライターです。
ドキュメントの正確性・完全性・読みやすさを評価してください。`,

  // 許可するツールの制限（省略時は全ツール）
  allowedTools: ["Read", "Grep", "Glob"],

  // 永続メモリディレクトリ（会話をまたいで保持）
  memory: "~/.claude/memory/doc-reviewer/",

  // コンテキスト制限（トークン数）
  maxContextTokens: 50000,
};
```

### YAML 形式での定義（Claude Code の場合）

```yaml
# ~/.claude/agents/doc-reviewer.yaml
name: doc-reviewer
description: >
  Markdownドキュメント・README・API仕様書の品質レビューを行う。
  ドキュメントの正確性、完全性、読みやすさ、コードサンプルの動作確認を担当。

system_prompt: |
  あなたはシニアテクニカルライターです。
  以下の観点でドキュメントをレビューしてください：
  1. 技術的正確性（コードサンプルが実際に動作するか）
  2. 完全性（必要な情報が全て含まれているか）
  3. 読みやすさ（構造・文体・例示）
  4. リンクの有効性

allowed_tools:
  - Read
  - Grep
  - Glob

memory: ~/.claude/memory/doc-reviewer/
```

---

## 3. ハンズオン：コードベース解析 Subagent を構築する

### Step 1: プロジェクトを準備する

```bash
mkdir -p ~/handson-subagent
cd ~/handson-subagent
git init
npm init -y

# サンプルソースを作成
cat > src/auth.ts << 'EOF'
// 意図的に問題を含んだサンプル
export function login(username: string, password: string) {
  const query = `SELECT * FROM users WHERE username='${username}' AND password='${password}'`;
  return db.query(query); // SQLインジェクション脆弱性
}

export function generateToken(userId: number) {
  return Math.random().toString(36); // 弱いトークン生成
}
EOF
```

### Step 2: カスタム Subagent を定義する

```bash
mkdir -p .claude/agents

cat > .claude/agents/security-auditor.yaml << 'EOF'
name: security-auditor
description: >
  TypeScript/JavaScriptコードのセキュリティ脆弱性を検出する。
  SQLインジェクション、XSS、認証バイパス、機密情報漏洩などを特定してレポートを返す。

system_prompt: |
  あなたはアプリケーションセキュリティの専門家です。
  OWASP Top 10を中心にコードを分析し、脆弱性を検出してください。

  ## レポート形式
  ### 重大度: CRITICAL / HIGH / MEDIUM / LOW
  - **脆弱性タイプ**: （例: SQL Injection）
  - **場所**: ファイル名:行番号
  - **説明**: 何が問題か
  - **修正案**: 具体的なコード例

allowed_tools:
  - Read
  - Grep
  - Glob
EOF
```

### Step 3: メインエージェントから Subagent を呼び出す（Claude Code）

Claude Code のセッションで：

```
src/ 配下のTypeScriptファイルをsecurity-auditorエージェントでセキュリティ監査してください
```

Claude は `security-auditor` サブエージェントを起動し、独立したコンテキストで分析を実行。結果のみをメインコンテキストに返します。

### Step 4: SDK から Subagent を制御する（Python）

```python
# Python SDK での Subagent 呼び出し
import anthropic
from anthropic.agent_sdk import AgentSession, SubagentConfig

client = anthropic.Anthropic()

# メインセッションを作成
session = AgentSession(
    client=client,
    model="claude-opus-4-6",
    system="あなたはソフトウェアエンジニアリングチームのリードです。"
)

# セキュリティ監査サブエージェントを定義
security_auditor = SubagentConfig(
    name="security-auditor",
    description="コードのセキュリティ脆弱性を検出する",
    system_prompt="""OWASP Top 10の観点でコードを分析し、
脆弱性を検出してレポートを返してください。""",
    allowed_tools=["Read", "Grep", "Glob"],
)

# パフォーマンス分析サブエージェントを定義
perf_analyzer = SubagentConfig(
    name="performance-analyzer",
    description="コードのパフォーマンス問題を検出する",
    system_prompt="N+1クエリ、メモリリーク、不要な計算などを特定してください。",
    allowed_tools=["Read", "Grep", "Glob"],
)

# 並列実行：セキュリティ監査とパフォーマンス分析を同時に実施
results = session.run_subagents_parallel([
    {
        "agent": security_auditor,
        "task": "src/ ディレクトリ配下の全TSファイルをセキュリティ監査してください"
    },
    {
        "agent": perf_analyzer,
        "task": "src/ ディレクトリのパフォーマンス問題を分析してください"
    }
])

# 結果を統合
for result in results:
    print(f"=== {result.agent_name} の結果 ===")
    print(result.output)
```

### Step 5: 実行結果の確認

並列実行により、逐次実行の約1/2〜1/3の時間で完了します。

```
=== security-auditor の結果 ===
### 重大度: CRITICAL
- **脆弱性タイプ**: SQL Injection
- **場所**: src/auth.ts:3
- **説明**: ユーザー入力を直接SQLクエリに連結しています
- **修正案**:
  ```typescript
  // プリペアドステートメントを使用
  const query = 'SELECT * FROM users WHERE username=? AND password=?';
  return db.query(query, [username, password]);
  ```

### 重大度: HIGH
- **脆弱性タイプ**: Weak Cryptography
- **場所**: src/auth.ts:8
- **説明**: Math.random()は暗号学的に安全ではありません
...

=== performance-analyzer の結果 ===
パフォーマンス上の問題は検出されませんでした。
```

---

## 4. Subagent のメモリ機能（永続化）

```python
# メモリを持つサブエージェント
research_agent = SubagentConfig(
    name="codebase-researcher",
    description="コードベースの構造を学習・記憶する",
    system_prompt="コードベースのアーキテクチャを分析し、パターンを記録してください。",
    allowed_tools=["Read", "Grep", "Glob", "Write"],

    # セッションをまたいで保持されるメモリディレクトリ
    memory_dir="~/.claude/memory/codebase-researcher/",
)

# 初回実行：コードベースを学習
session.run_subagent(
    agent=research_agent,
    task="このリポジトリのアーキテクチャを分析してmemory/MEMORY.mdに記録してください"
)

# 2回目以降：前回の知識を活用
session.run_subagent(
    agent=research_agent,
    task="認証モジュールに新機能を追加する最適な場所を提案してください"
    # 前回学習した構造情報を活用して回答
)
```

---

## 5. ツール別 Subagent サポート状況比較（2026年2月時点）

### 概要比較表

| 機能 | Claude Code | OpenAI Codex | Antigravity (Google) | GitHub Copilot |
|------|:-----------:|:------------:|:--------------------:|:--------------:|
| **Subagent 基本機能** | ✅ 正式 | ✅ 正式 | ✅ 正式 | ✅ 正式 |
| **ビルトインエージェント** | Explore / Plan / general-purpose | — | browser_subagent 他 | Explore / Task / Code Review / Plan |
| **カスタムエージェント定義** | ✅ YAML | ✅ ロール定義 | ✅ Agent Builder | ⚠️ 限定的 |
| **並列実行** | ✅ | ✅ (spawn_agents_on_csv等) | ✅ 最大5並列 | ✅ |
| **コンテキスト分離** | ✅ 独立ウィンドウ | ✅ | ✅ | ✅ (runSubagent) |
| **ツール権限制限** | ✅ allowed_tools | ✅ ロール別 | ✅ | ⚠️ 一部 |
| **永続メモリ** | ✅ memory_dir | — | — | — |
| **孫エージェント (Nested)** | ⚠️ Swarms実験的 | ⚠️ max_depth設定可 | ❓ 不明 | ❌ 未サポート |
| **AGENTS.md / CLAUDE.md** | ✅ CLAUDE.md | ✅ AGENTS.md | ✅ AGENTS.md | ✅ AGENTS.md |
| **MCP 連携** | ✅ | ✅ | ✅ | ✅ |

---

### Claude Code の Swarms（実験的機能）

2026年1月24日、開発者 Mike Kelly が Claude Code に隠されていた **Swarms** 機能を発見しました。これは正式リリース前の実験的機能で、通常の「孫エージェント非サポート」という制限を一部回避します。

#### Swarms の有効化

```json
// ~/.claude/settings.json
{
  "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
}
```

または環境変数で：

```bash
CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 claude
```

#### Swarms のアーキテクチャ

```
Team Lead エージェント（Opus 4.6）
    │
    ├─ 共有タスクボード（Kanban形式）
    │       ├─ task-001: [specialist-A に割り当て]
    │       ├─ task-002: [specialist-B に割り当て]
    │       └─ task-003: [specialist-C に割り当て]
    │
    ├─ Specialist A  ←→ メッセージパッシングで協調
    ├─ Specialist B
    └─ Specialist C
```

- Team Lead がタスクを分解して共有ボードに積む
- 各 Specialist がボードからタスクを取得して並列実行
- Specialist 間のメッセージパッシングで依存関係を解決
- **孫エージェント（孫以下のネスト）はSwarms内でも不可**

#### Swarms の実装例（9エージェント構成）

```python
# swarms_example.py
# （CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 が必要）
import anthropic
from anthropic.agent_sdk import AgentSession, SwarmConfig

client = anthropic.Anthropic()

swarm = SwarmConfig(
    team_lead_model="claude-opus-4-6",
    specialist_model="claude-sonnet-4-6",
    specialists=[
        {"name": "researcher",  "description": "情報調査・ドキュメント収集"},
        {"name": "architect",   "description": "設計・アーキテクチャ決定"},
        {"name": "frontend",    "description": "React/TypeScript実装"},
        {"name": "backend",     "description": "Node.js/API実装"},
        {"name": "database",    "description": "DBスキーマ・クエリ最適化"},
        {"name": "tester",      "description": "テスト作成・実行"},
        {"name": "reviewer",    "description": "コードレビュー"},
        {"name": "security",    "description": "セキュリティ監査"},
        {"name": "docs",        "description": "ドキュメント作成"},
    ],
    shared_task_board=True,
    message_passing=True,
)

session = AgentSession(client=client, swarm_config=swarm)
session.run("ECサイトのチェックアウト機能をフルスタックで実装してください")
```

---

### OpenAI Codex の Subagent（`agents.max_depth`）

Codex の `agents.max_depth` パラメータでネスト深度を制御できます。

```json
// codex の設定
{
  "agents": {
    "max_depth": 1,      // デフォルト: 1（直接の子エージェントのみ）
    "max_agents": 10,    // 最大同時エージェント数
    "roles": [
      {
        "name": "researcher",
        "description": "調査専門エージェント",
        "config_file": ".codex/researcher.md"
      }
    ]
  }
}
```

- `max_depth: 1` → 子エージェントのみ（孫なし）
- `max_depth: 2` → 孫エージェントまで許可（実験的）
- 2026年2月現在、`max_depth > 1` は不安定な場合がありFeature Request として issue [#2604](https://github.com/openai/codex/issues/2604) が存在

**2026年2月追加の multi-agent 機能:**

```python
# CSV から並列エージェントを展開（spawn_agents_on_csv）
# tasks.csv の各行に対して独立エージェントが並列起動される
# ETA表示・進捗追跡・エラーリトライが組み込み
```

---

### Antigravity (Google) の Subagent

Google Antigravity（2025年11月18日発表 / 2026年1月 GA）は Gemini 3.1 Pro + 3 Flash を基盤とするエージェントファーストIDEです。

#### 専用サブエージェント

| サブエージェント | 説明 |
|----------------|------|
| `browser_subagent` | Chromium を自動操作。WebP動画で操作を録画 |
| `code_subagent` | コード実行・ビルド・テスト担当 |
| `search_subagent` | Web検索・ドキュメント調査 |

#### 並列エージェント起動

```python
# Antigravity の並列エージェント（最大5並列）
# 例: 5つのバグを5エージェントが同時修正
agents = antigravity.spawn_agents([
    {"task": "bug-001", "branch": "fix/bug-001"},
    {"task": "bug-002", "branch": "fix/bug-002"},
    {"task": "bug-003", "branch": "fix/bug-003"},
    {"task": "bug-004", "branch": "fix/bug-004"},
    {"task": "bug-005", "branch": "fix/bug-005"},
])
# Agent Manager UIで各エージェントの状態・成果物を可視化
```

#### 孫エージェントのサポート状況

2026年2月時点で Antigravity における孫エージェント（エージェントがさらにエージェントをスポーン）の公式サポートは**未確認**です。Google AI Developers Forum でも明確な回答は出ていません（[該当スレッド](https://discuss.ai.google.dev/t/antigravity-sub-agents/114381)）。

---

### GitHub Copilot の Subagent

GitHub Copilot は Agent Mode（2025年末 GA）でエージェント機能を強化しました。

#### ビルトインエージェント（自動委譲）

| エージェント | 説明 |
|------------|------|
| **Explore** | コードベース高速解析（読み取り専用） |
| **Task** | ビルド・テスト実行 |
| **Code Review** | 変更差分の品質レビュー |
| **Plan** | 実装計画立案 |

#### `runSubagent` ツール（VS Code）

VS Code 1.107（2025年11月）で追加された `runSubagent` ツールにより、サブエージェントがメインチャットとは独立した専用コンテキストで実行されます。

```typescript
// VS Code の runSubagent（概念実装）
const result = await vscode.agent.runSubagent({
  prompt: "src/ 配下のTypeScriptの型エラーを修正してください",
  context: {
    workspaceFiles: selectedFiles,
    // メインチャットのコンテキストとは独立
  },
  tools: ["Edit", "Read", "Bash"],
});
```

#### 孫エージェントのサポート状況

2026年2月現在、孫エージェントは**未サポート**です。copilot-sdk の issue [#301「Support for subagents and orchestration」](https://github.com/github/copilot-sdk/issues/301) として機能追加が要望されています。

---

### 孫エージェント（Nested Subagent）サポート状況まとめ

| ツール | 孫エージェント | 詳細 |
|--------|:-----------:|------|
| **Claude Code** | ⚠️ 実験的 | Swarms（`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`）で一部可能。2026年Swarming強化予定 |
| **OpenAI Codex** | ⚠️ 設定依存 | `agents.max_depth` を 2以上に設定すると孫エージェントが動作（不安定） |
| **Antigravity** | ❓ 不明 | 公式ドキュメントに明記なし。フォーラムで議論中 |
| **GitHub Copilot** | ❌ 未サポート | issue #301 として要望中 |

> **推奨**: 本番環境では孫エージェントへの依存を避け、オーケストレーター+フラットなサブエージェント群（深さ1）で設計すること。孫エージェントが必要な複雑なワークフローは Worktree + AgentTeam パターンで代替できます。

---

## 6. セキュリティのベストプラクティス（2026年）

```yaml
# 最小権限の原則：サブエージェントには必要最小限のツールのみ許可
security-auditor:
  allowed_tools:
    - Read    # 読み取りのみ
    - Grep
    - Glob
  # Write や Bash は付与しない → 誤ってコードを変更するリスクを排除

# 危険なコマンドのブロック
hooks:
  pre_tool:
    - command: "deny-dangerous-commands.sh"
      # rm -rf、git push --force などをブロック
```

---

## 参考リンク

**Claude Code:**
- [Create custom subagents - Claude Code Docs](https://code.claude.com/docs/en/sub-agents)
- [Subagents in the SDK - Claude API Docs](https://platform.claude.com/docs/en/agent-sdk/subagents)
- [Claude Code multiple agent systems: Complete 2026 guide](https://www.eesel.ai/blog/claude-code-multiple-agent-systems-complete-2026-guide)
- [Claude Code's new hidden feature: Swarms (Hacker News)](https://news.ycombinator.com/item?id=46743908)
- [Claude Code Swarms (Addy Osmani)](https://addyosmani.com/blog/claude-code-agent-teams/)
- [Context Management with Subagents in Claude Code](https://www.richsnapp.com/article/2025/10-05-context-management-with-subagents-in-claude-code)
- [awesome-claude-code-subagents](https://github.com/VoltAgent/awesome-claude-code-subagents)

**OpenAI Codex:**
- [Multi-agents - Codex Docs](https://developers.openai.com/codex/multi-agent)
- [Subagent Support · Issue #2604](https://github.com/openai/codex/issues/2604)
- [Feature Request: Sub-Agent Collaboration · Issue #9846](https://github.com/openai/codex/issues/9846)

**Antigravity (Google):**
- [Google Antigravity Developer Blog](https://developers.googleblog.com/build-with-google-antigravity-our-new-agentic-development-platform/)
- [Antigravity sub agents - Google AI Developers Forum](https://discuss.ai.google.dev/t/antigravity-sub-agents/114381)
- [Parallel agents in Antigravity (Medium)](https://medium.com/google-cloud/parallel-agents-in-antigravity-64237120161d)

**GitHub Copilot:**
- [GitHub Copilot coding agent - What's new](https://github.blog/ai-and-ml/github-copilot/whats-new-with-github-copilot-coding-agent/)
- [Support for subagents and orchestration · Issue #301](https://github.com/github/copilot-sdk/issues/301)
- [VS Code Unified Agent Experience](https://code.visualstudio.com/blogs/2025/11/03/unified-agent-experience)
