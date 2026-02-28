# MCP — Model Context Protocol

**カテゴリ:** プロトコル標準 / インフラ
**策定:** Anthropic（2024年11月提案）→ 2025年3月仕様 v2025-03-26 確定
**ステータス:** 業界標準として普及中（2026年現在）

---

## 1. 概要

**Model Context Protocol (MCP)** は、AIエージェントと外部ツール・データソースを接続するための標準化されたプロトコルです。

「AIのためのUSB規格」と表現されることもあり、どのAIクライアントからでも対応サーバーを利用できます。

### なぜ MCP が必要か

MCP 以前：
```
Claude ──→ 独自実装 ──→ GitHub API
Claude ──→ 独自実装 ──→ PostgreSQL
Claude ──→ 独自実装 ──→ Slack
# エージェントごとに独自の統合コードが必要
```

MCP 以後：
```
Claude ──→ MCP ──→ MCP Server (GitHub)
Copilot ──→ MCP ──→ MCP Server (PostgreSQL)
任意のAI ──→ MCP ──→ MCP Server (Slack)
# 1つのサーバー実装で全クライアントが使える
```

### 対応クライアント（2026年1月時点）
- Claude Code / Claude.ai
- ChatGPT（OpenAI）
- GitHub Copilot
- VS Code（Goose統合）
- Cursor
- 200以上のコミュニティサーバー

---

## 2. MCP のアーキテクチャ

```
┌─────────────────────────────────────────────────────┐
│                  MCP ホスト（Claude Code）            │
│                                                      │
│  ┌─────────┐     ┌──────────────────────────────┐   │
│  │  Claude  │────▶│        MCP クライアント        │   │
│  │  Model  │◀────│  （プロトコル管理・結果変換）    │   │
│  └─────────┘     └──────────┬───────────────────┘   │
└─────────────────────────────┼───────────────────────┘
                              │ MCP Protocol
              ┌───────────────┼────────────────┐
              ▼               ▼                ▼
    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
    │ MCP Server  │  │ MCP Server  │  │ MCP Server  │
    │  (GitHub)   │  │ (Postgres)  │  │  (Slack)    │
    └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
           │                │                │
    ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
    │  GitHub API │  │  PostgreSQL │  │  Slack API  │
    └─────────────┘  └─────────────┘  └─────────────┘
```

### MCP が提供する機能

| 機能 | 説明 | 例 |
|------|------|-----|
| **Tools** | エージェントが呼び出せる関数 | `create_issue`, `query_db` |
| **Resources** | 読み取り可能なデータソース | ファイル、DB レコード |
| **Prompts** | 再利用可能なプロンプトテンプレート | コードレビューテンプレート |
| **Sampling** | サーバーからモデルへの推論依頼 | LLM による分析 |

### トランスポート方式

| 方式 | 用途 | 特徴 |
|------|------|------|
| **stdio** | ローカルプロセス | 最もシンプル。CLIツール向け |
| **Streamable HTTP** | リモートサーバー | v2025-03-26 の新標準。SSEを置き換え |
| **SSE**（旧） | リモートサーバー | 後方互換のため継続サポート |

---

## 3. ハンズオン：MCP サーバーを自作して接続する

### Step 1: MCP SDK をインストールする

```bash
npm install @modelcontextprotocol/sdk
```

### Step 2: シンプルな MCP サーバーを実装する

```typescript
// mcp-server/src/index.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";
import * as fs from "fs/promises";
import * as path from "path";

// MCP サーバーを作成
const server = new McpServer({
  name: "file-manager",
  version: "1.0.0",
});

// Tool 1: ファイル一覧取得
server.tool(
  "list_files",
  "指定ディレクトリのファイル一覧を取得する",
  {
    directory: z.string().describe("対象ディレクトリのパス"),
    extension: z.string().optional().describe("フィルタする拡張子（例: .ts）"),
  },
  async ({ directory, extension }) => {
    const files = await fs.readdir(directory);
    const filtered = extension
      ? files.filter((f) => f.endsWith(extension))
      : files;

    return {
      content: [
        {
          type: "text",
          text: filtered.join("\n"),
        },
      ],
    };
  }
);

// Tool 2: ファイル内容の読み取り
server.tool(
  "read_file",
  "ファイルの内容を読み取る",
  {
    file_path: z.string().describe("読み取るファイルのパス"),
  },
  async ({ file_path }) => {
    const content = await fs.readFile(file_path, "utf-8");

    return {
      content: [
        {
          type: "text",
          text: content,
        },
      ],
    };
  }
);

// Tool 3: ファイル書き込み
server.tool(
  "write_file",
  "ファイルに内容を書き込む",
  {
    file_path: z.string().describe("書き込み先のファイルパス"),
    content: z.string().describe("書き込む内容"),
  },
  async ({ file_path, content }) => {
    await fs.writeFile(file_path, content, "utf-8");

    return {
      content: [
        {
          type: "text",
          text: `ファイルを書き込みました: ${file_path}`,
        },
      ],
    };
  }
);

// Resource: プロジェクト設定ファイルを公開
server.resource(
  "project-config",
  "project://config",
  "プロジェクトの設定ファイル",
  async (uri) => {
    const config = await fs.readFile("./project.json", "utf-8");
    return {
      contents: [
        {
          uri: uri.href,
          mimeType: "application/json",
          text: config,
        },
      ],
    };
  }
);

// stdio トランスポートで起動
const transport = new StdioServerTransport();
await server.connect(transport);
```

### Step 3: Claude Code に MCP サーバーを登録する

```bash
# グローバル登録
claude mcp add file-manager node /path/to/mcp-server/src/index.ts

# または設定ファイルに直接追記
cat >> ~/.claude/settings.json << 'EOF'
{
  "mcpServers": {
    "file-manager": {
      "command": "node",
      "args": ["/path/to/mcp-server/dist/index.js"],
      "env": {
        "NODE_ENV": "production"
      }
    }
  }
}
EOF
```

### Step 4: プロジェクト限定の MCP 設定

```json
// .claude/settings.json（プロジェクトルート）
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_URL": "postgresql://localhost:5432/mydb"
      }
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

### Step 5: Claude から MCP ツールを使う

Claude Code のセッションで：

```
src/ のTypeScriptファイルをすべて一覧表示してください
```

→ Claude が `file-manager` の `list_files` ツールを自動的に選択して呼び出します。

```
auth/login.ts の内容を読んで、セキュリティ問題を報告してください
```

→ `read_file` ツールを呼び出してからセキュリティ分析を実行します。

---

## 4. Streamable HTTP トランスポート（2025年3月標準化）

リモートサーバーとして MCP を提供する場合は Streamable HTTP を使います。

```typescript
// HTTP サーバーとして公開
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express from "express";

const app = express();
const mcpServer = new McpServer({ name: "remote-tools", version: "1.0.0" });

// ... ツールを定義 ...

app.all("/mcp", async (req, res) => {
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: () => crypto.randomUUID(),
  });
  await mcpServer.connect(transport);
  await transport.handleRequest(req, res);
});

app.listen(3000);
```

---

## 5. MCP Apps（2026年1月新機能）

MCP v1.1 で導入された **MCP Apps** は、ツールの結果として **インタラクティブな UI コンポーネント** を返せる拡張です。

```typescript
// UI コンポーネントを返す MCP ツール
server.tool(
  "show_dashboard",
  "データダッシュボードを表示する",
  { metric: z.string() },
  async ({ metric }) => {
    return {
      content: [
        {
          type: "ui_component",  // 新しいコンテンツタイプ
          component: {
            type: "chart",
            data: await fetchMetricData(metric),
            chartType: "line",
            title: `${metric} の推移`,
          },
        },
      ],
    };
  }
);
```

Claude の会話内でインタラクティブなチャートや表が直接レンダリングされます。

---

## 6. 既存の主要 MCP サーバー（2026年2月時点）

```bash
# データベース
npx -y @modelcontextprotocol/server-postgres    # PostgreSQL
npx -y @modelcontextprotocol/server-sqlite      # SQLite
npx -y mcp-server-mysql                          # MySQL

# 開発ツール
npx -y @modelcontextprotocol/server-github      # GitHub
npx -y @modelcontextprotocol/server-gitlab      # GitLab
npx -y @modelcontextprotocol/server-git         # ローカルGit

# 検索・Web
npx -y @modelcontextprotocol/server-brave-search  # Brave Search
npx -y mcp-server-puppeteer                        # ブラウザ操作

# ファイル・OS
npx -y @modelcontextprotocol/server-filesystem  # ファイルシステム
npx -y mcp-server-docker                        # Docker

# コミュニケーション
npx -y @modelcontextprotocol/server-slack       # Slack
npx -y mcp-server-gmail                         # Gmail
```

---

## 参考リンク

- [MCP: Model Context Protocol](https://institute.sfeir.com/en/claude-code/claude-code-mcp-model-context-protocol/)
- [Connect Claude Code to tools via MCP](https://code.claude.com/docs/en/mcp)
- [Claude Skills vs. MCP: A Technical Comparison](https://intuitionlabs.ai/articles/claude-skills-vs-mcp)
- [MCP, Skills, and Agents](https://cra.mr/mcp-skills-and-agents/)
