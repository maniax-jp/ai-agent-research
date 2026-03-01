# MCP（Model Context Protocol）- AI エージェントの標準接続プロトコル

**作成者:** Qwen
**日付:** 2026 年 3 月 1 日

---

## 目次

1. [概要](#概要)
2. [MCP と Skills の違い](#mcp と-skills の違い)
3. [MCP のアーキテクチャ](#mcp のアーキテクチャ)
4. [ハンズオン：MCP サーバーの作成](#ハンズオンmcp サーバーの作成)
5. [進階：Streamable HTTP と MCP Apps](#進階streamable-http と-mcp-apps)
6. [既存 MCP サーバーの活用](#既存-mcp サーバーの活用)
7. [ベストプラクティス](#ベストプラクティス)

---

## 概要

**Model Context Protocol (MCP)** は、AI エージェントと外部ツール・データソースを接続するための標準化されたプロトコルです。Anthropic によって 2024 年 11 月に提案され、2025 年 3 月に仕様 v2025-03-26 が確定しました。

「AI エージェントのための USB 規格」と表現されることもあり、どの AI クライアントからでも対応サーバーを利用できます。

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
任意の AI ──→ MCP ──→ MCP Server (Slack)
# 1 つのサーバー実装で全クライアントが使える
```

### 対応クライアント（2026 年 2 月時点）

- Claude Code / Claude.ai
- ChatGPT（OpenAI）
- GitHub Copilot
- VS Code（Goose 統合）
- Cursor
- 200 以上のコミュニティサーバー

---

## MCP と Skills の違い

| 特徴 | Skills | MCP |
|------|--------|-----|
| **データタイプ** | 静的（ドキュメント、パターン） | 動的（リアルタイムデータ） |
| **接続** | ファイルベース | ランタイム接続 |
| **セットアップ** | 0 設定 | サーバー起動が必要 |
| **使用ケース** | コーディングパターン、SDK ドキュメント | データベース、API、外部サービス |
| **パフォーマンス** | 高速（ファイル読み込みのみ） | 遅延（ネットワーク呼び出し） |

### 実務での組み合わせ

実務では「Skill が MCP ツールを呼ぶ」形で組み合わせます：

```
Skill（手順定義）
    │
    ├─→ MCP ツール呼び出し（データ取得）
    ├─→ 処理ロジック
    └─→ 出力生成
```

---

## MCP のアーキテクチャ

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
| **stdio** | ローカルプロセス | 最もシンプル。CLI ツール向け |
| **Streamable HTTP** | リモートサーバー | v2025-03-26 の新標準 |
| **SSE**（旧） | リモートサーバー | 後方互換のため継続サポート |

---

## ハンズオン：MCP サーバーの作成

### ステップ 1: MCP SDK をインストール

```bash
mkdir my-mcp-server
cd my-mcp-server
npm init -y
npm install @modelcontextprotocol/sdk zod
```

### ステップ 2: シンプルな MCP サーバーを実装

`src/server.ts` を作成：

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";
import * as fs from "fs/promises";

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
    extension: z.string().optional().describe("フィルタする拡張子（例：.ts）"),
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
          text: `ファイルを書き込みました：${file_path}`,
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

### ステップ 3: Claude Code に MCP サーバーを登録

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

### ステップ 4: プロジェクト限定の MCP 設定

`.claude/settings.json`（プロジェクトルート）：

```json
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

### ステップ 5: Claude から MCP ツールを使う

Claude Code のセッションで：

```
src/ の TypeScript ファイルをすべて一覧表示してください
```

→ Claude が `file-manager` の `list_files` ツールを自動的に選択して呼び出します。

```
auth/login.ts の内容を読んで、セキュリティ問題を報告してください
```

→ `read_file` ツールを呼び出してからセキュリティ分析を実行します。

---

## 進階：Streamable HTTP と MCP Apps

### Streamable HTTP トランスポート

リモートサーバーとして MCP を提供する場合は Streamable HTTP を使用：

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

### MCP Apps（2026 年新機能）

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

## 既存 MCP サーバーの活用

### 主要 MCP サーバー一覧（2026 年 2 月時点）

```bash
# データベース
npx -y @modelcontextprotocol/server-postgres    # PostgreSQL
npx -y @modelcontextprotocol/server-sqlite      # SQLite
npx -y mcp-server-mysql                          # MySQL

# 開発ツール
npx -y @modelcontextprotocol/server-github      # GitHub
npx -y @modelcontextprotocol/server-gitlab      # GitLab
npx -y @modelcontextprotocol/server-git         # ローカル Git

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

### 使用例：PostgreSQL MCP サーバー

```bash
# 設定ファイルに追加
cat >> ~/.claude/settings.json << 'EOF'
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_URL": "postgresql://user:pass@localhost/mydb"
      }
    }
  }
}
EOF
```

Claude Code で：

```
ユーザーテーブルから最新の 10 件を取得して、平均年齢を計算してください
```

→ PostgreSQL MCP サーバーが自動的に SQL を生成して実行します。

---

## ベストプラクティス

### 1. ツール説明の明確化

```typescript
// 悪い例
server.tool(
  "query",
  "クエリを実行",  // 曖昧
  { q: z.string() },
  // ...
);

// 良い例
server.tool(
  "query_database",
  "SQL クエリを実行して結果を返す。SELECT 文のみ許可。",
  {
    sql: z.string().describe("実行する SQL クエリ（SELECT のみ）"),
    limit: z.number().optional().describe("返す行数の制限（デフォルト：100）"),
  },
  // ...
);
```

### 2. エラーハンドリング

```typescript
server.tool(
  "read_file",
  "ファイルの内容を読み取る",
  { file_path: z.string() },
  async ({ file_path }) => {
    try {
      const content = await fs.readFile(file_path, "utf-8");
      return {
        content: [{ type: "text", text: content }],
      };
    } catch (err) {
      return {
        content: [{ type: "text", text: `エラー：${err.message}` }],
        isError: true,  // isError フラグをセット
      };
    }
  }
);
```

### 3. 認証情報の管理

```typescript
// 悪い例：Agent に認証情報を渡す
server.tool(
  "api_call",
  "API を呼び出す",
  { token: z.string() },  // Agent にトークンを渡させる
  // ...
);

// 良い例：サーバー側で認証情報を管理
const API_TOKEN = process.env.API_TOKEN;  // 環境変数から読み取り

server.tool(
  "api_call",
  "API を呼び出す",
  {},  // 認証情報は隠蔽
  async () => {
    const response = await fetch(url, {
      headers: { Authorization: `Bearer ${API_TOKEN}` }
    });
    // ...
  }
);
```

### 4. スキーマの厳格化

```typescript
// 厳格なスキーマ定義
server.tool(
  "create_user",
  "新規ユーザーを作成",
  {
    email: z.string().email(),
    name: z.string().min(1).max(100),
    role: z.enum(["admin", "user", "guest"]),
  },
  // ...
);
```

---

## まとめ

MCP は AI エージェントと外部ツールを接続するための標準プロトコルとして、2026 年現在業界標準となっています。一度 MCP サーバーを作成すれば、複数の AI クライアントから利用可能になるため、開発効率が大幅に向上します。

### 次のステップ

1. [Skills](./01-skills.md) - Skills と MCP の組み合わせ
2. [AGENTS.md](./03-agents.md.md) - MCP 設定の統合
3. [OpenClaw](./04-openclaw.md) - OpenClaw と MCP の統合

---

*関連リソース:*
- [MCP: Model Context Protocol](https://modelcontextprotocol.io/)
- [Connect Claude Code to tools via MCP](https://code.claude.com/docs/en/mcp)
- [MCP SDK GitHub](https://github.com/modelcontextprotocol/typescript-sdk)
- [MCP Servers List](https://modelcontextprotocol.io/servers)