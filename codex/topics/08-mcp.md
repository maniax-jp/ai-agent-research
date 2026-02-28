# 08. MCP（Model Context Protocol）によるツール連携標準化

## 技術動向
`MCP` は、Agentと外部ツール（DB/API/ファイル操作）を接続するための標準プロトコルです。2026年以降は、ベンダー固有の個別統合よりも、`MCPサーバーを共通接続層にする設計`が主流です。

## 設計ポイント
- Agent側は「接続先固有実装」を持たず、MCPクライアント経由で利用する
- ツール定義（引数スキーマ・説明）を明示し、誤呼び出しを減らす
- 認可情報はMCPサーバー側で管理し、Agentに秘密情報を渡しすぎない

## ハンズオン（最小MCPサーバーを試す）
前提: Node.js 18+ が利用可能

1. 作業ディレクトリ作成
```powershell
New-Item -ItemType Directory -Force mcp-demo/src | Out-Null
Set-Location mcp-demo
npm init -y
npm install @modelcontextprotocol/sdk zod
```

2. サーバー実装（`src/server.mjs`）
```javascript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({ name: "demo-tools", version: "1.0.0" });

server.tool(
  "sum",
  "2つの数値を足し算する",
  { a: z.number(), b: z.number() },
  async ({ a, b }) => ({ content: [{ type: "text", text: String(a + b) }] })
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

3. 起動確認
```powershell
node src/server.mjs
```

4. Agent側設定に登録（利用中ツールの設定ファイルに追加）
- `command`: `node`
- `args`: `["<absolute-path>/mcp-demo/src/server.mjs"]`

5. 呼び出し検証
- Agentに「MCPの `sum` で `3+5` を計算して」と指示
- 返り値 `8` を確認

## 期待結果
- Agent実装と外部接続実装を分離できる
- スキーマ付きツール呼び出しで入力ミスを減らせる

## 失敗しやすい点
- ツール説明が曖昧で誤った引数が渡される
- サーバー側例外を握りつぶし、原因追跡できない
- 認証情報をAgent側プロンプトに埋め込み漏えいリスクを作る

## 次の強化
- `stdio` だけでなくHTTPトランスポート構成も追加
- MCPサーバーごとに権限境界（read-only / write）を分離
