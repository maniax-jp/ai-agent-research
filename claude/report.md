# AI-Agent 技術動向調査レポート（2026年以降）

**調査担当:** Claude (Anthropic)
**調査日:** 2026年2月28日
**フォーカス:** 技術的動向（Claude Code / Agent SDK エコシステム中心）

> 本レポートはAnthropicのClaude視点から、AI-Agentの **技術的動向** に絞って調査・執筆しています。
> 社会的・経済的動向は対象外です。

---

## ディレクトリ構造

```
claude/
├── report.md          ← このファイル（インデックス）
└── topics/
    ├── 01_skills.md          Skills（再利用可能プロンプト + I/O契約）
    ├── 02_subagent.md        Subagent（Planner→Worker→Reviewer テンプレート）
    ├── 03_claude_md.md       CLAUDE.md / Agent.md（知識 + 実行契約）
    ├── 04_mcp.md             MCP – Model Context Protocol
    ├── 05_agent_team.md      AgentTeam（Self-healing ループ + DAG）
    ├── 06_context_fork.md    Context Fork（会話分岐 + Decision ログ）
    ├── 07_worktree.md        Git Worktree（並列作業 + 失敗破棄パターン）
    └── 08_openclaw.md        OpenClaw（フルアクセス自律エージェント + E2E評価）
```

---

## 技術トピック一覧

| # | トピック | 概要 | 詳細 |
|---|---------|------|------|
| 1 | **Skills** | 再利用可能な業務ユニット。入出力契約・バージョン管理・CI自動検証に対応 | [topics/01_skills.md](topics/01_skills.md) |
| 2 | **Subagent** | Planner→Worker→Reviewer の3層テンプレート。受け渡しJSON・失敗時局所リトライ | [topics/02_subagent.md](topics/02_subagent.md) |
| 3 | **CLAUDE.md / Agent.md** | 「プロジェクト知識」+「実行契約」の2側面を持つ指示ファイル。あり/なし比較つき | [topics/03_claude_md.md](topics/03_claude_md.md) |
| 4 | **MCP** | Model Context Protocol。エージェントと外部ツールを繋ぐ標準プロトコル（2025年標準化） | [topics/04_mcp.md](topics/04_mcp.md) |
| 5 | **AgentTeam** | Self-healing ループ・DAG・LangFlow。4パターンのトポロジーを解説 | [topics/05_agent_team.md](topics/05_agent_team.md) |
| 6 | **Context Fork** | 会話コンテキストの分岐・並列探索・Decision ログによる意思決定の記録 | [topics/06_context_fork.md](topics/06_context_fork.md) |
| 7 | **Worktree** | 並列作業環境。成功時マージ / 失敗時破棄（パターンB）の2パターンを解説 | [topics/07_worktree.md](topics/07_worktree.md) |
| 8 | **OpenClaw** | フルシステムアクセス型自律エージェント。E2E評価パイプライン・安全運用 | [topics/08_openclaw.md](topics/08_openclaw.md) |

---

## 技術スタックの全体像（2026年時点）

```
┌─────────────────────────────────────────────────────────┐
│                    User / Orchestrator                   │
│                   (CLAUDE.md で定義)                      │
└──────────────────────┬──────────────────────────────────┘
                       │ Skills で呼び出し
         ┌─────────────┼──────────────┐
         ▼             ▼              ▼
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │Subagent A│  │Subagent B│  │Subagent C│  ← AgentTeam
   │(Explore) │  │(Plan)    │  │(Custom)  │
   └────┬─────┘  └────┬─────┘  └────┬─────┘
        │              │              │
   ┌────▼─────────────▼──────────────▼─────┐
   │           MCP サーバー層               │
   │  (DB / API / FileSystem / Browser...) │
   └───────────────────────────────────────┘
        │              │              │
   ┌────▼─────┐  ┌─────▼────┐  ┌────▼──────┐
   │ Worktree │  │ Worktree │  │ Worktree  │  ← 並列Git環境
   │(branch A)│  │(branch B)│  │(branch C) │
   └──────────┘  └──────────┘  └───────────┘
                       ↕
              Context Fork で分岐・比較
```

---

## 各トピックの関連性

```
CLAUDE.md（知識 + 実行契約）
  └─ Skills でタスクを構造化（入出力契約・バージョン管理）
       └─ Subagent でコンテキスト分離（Planner→Worker→Reviewer）
            └─ AgentTeam で並列協調（Self-healing・DAG）
                 ├─ Context Fork で探索分岐（Decision ログ記録）
                 ├─ Worktree で並列ファイル作業（失敗時完全破棄）
                 ├─ MCP で外部ツール連携
                 └─ OpenClaw でフルアクセス自律実行（E2E評価）
```

---

*本ディレクトリは Claude (Anthropic) が担当。*
*他担当者: `/codex/` (OpenAI Codex), `/gemini/` (Google Gemini)*
