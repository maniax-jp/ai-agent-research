# AGENTS.md - AI エージェント向けのプロジェクト仕様書

**作成者:** Qwen
**日付:** 2026 年 3 月 1 日

---

## 目次

1. [概要](#概要)
2. [AGENTS.md の標準フォーマット](#agentsmd の標準フォーマット)
3. [ハンズオン：AGENTS.md の作成](#ハンズオンagentsmd の作成)
4. [進階：高度な AGENTS.md](#進階高度な-agentsmd)
5. [ベストプラクティス](#ベストプラクティス)

---

## 概要

**AGENTS.md**は、AI コーディングエージェント向けにプロジェクトのコンテキストと指示を提供するオープン標準です。2025 年以降、60,000 以上のオープンソースプロジェクトで採用されています。

### AGENTS.md の役割

- **コンテキストの提供:** プロジェクトの概要、技術スタック、アーキテクチャ
- **指示の明示:** コーディング規約、ワークフロー、テスト方針
- **一貫性の確保:** 異なるエージェント間での一貫した行動
- **オンボーディングの加速:** 新規エージェントの迅速なプロジェクト理解

---

## AGENTS.md の標準フォーマット

### 基本構造

```markdown
# プロジェクト名

## プロジェクト概要

[プロジェクトの説明]

## 技術スタック

- [言語/フレームワーク]
- [データベース]
- [インフラ]

## 開発環境

### 前提条件

[必要な環境]

### セットアップ

```bash
[インストールコマンド]
```

### コマンド

```bash
npm run dev    # 開発サーバー
npm test       # テスト実行
npm run build  # ビルド
```

## アーキテクチャ

[アーキテクチャの説明]

## コーディング規約

[コーディング規約]

## ワークフロー

[開発ワークフロー]

## テスト方針

[テスト方針]

## デプロイメント

[デプロイメント手順]
```

---

## ハンズオン：AGENTS.md の作成

### ステップ 1: プロジェクトの準備

```bash
# プロジェクトディレクトリを作成
mkdir my-ai-project
cd my-ai-project

# 基本ファイルを作成
touch package.json src/index.ts
```

### ステップ 2: 基本的な AGENTS.md の作成

`AGENTS.md` を作成：

```markdown
# My AI Project

## プロジェクト概要

TypeScript を使用した AI エージェント統合プロジェクトです。

## 技術スタック

- TypeScript 5.0+
- Node.js 18+
- Express.js
- PostgreSQL
- Prisma ORM

## 開発環境

### 前提条件

- Node.js 18+
- pnpm 8+
- PostgreSQL 15+

### セットアップ

```bash
# 依存関係のインストール
pnpm install

# 環境変数の設定
cp .env.example .env

# データベースのマイグレーション
pnpm db:generate
pnpm db:migrate

# 開発サーバーの起動
pnpm dev
```

### コマンド

```bash
pnpm dev          # 開発サーバーを起動
pnpm test         # テストを実行
pnpm test:watch   # ウォッチモードでテスト
pnpm build        # プロダクションビルド
pnpm db:generate  # Prisma クライアントを生成
pnpm db:migrate   # データベースマイグレーション
pnpm db:studio    # Prisma Studio を起動
```

## アーキテクチャ

```
src/
├── api/          # API ルート
├── services/     # ビジネスロジック
├── models/       # データモデル
├── utils/        # 共通ユーティリティ
└── config/       # 設定
```

## コーディング規約

### 命名規則

- 変数・関数：camelCase
- クラス：PascalCase
- 定数：UPPER_SNAKE_CASE
- ファイル名：kebab-case

### インポート

```typescript
// 外部ライブラリ
import { something } from 'external-lib';

// 内部モジュール
import { utility } from '@/utils/utility';
import { Service } from '../services/Service';
```

### エラーハンドリング

```typescript
// カスタムエラーを使用
class AppError extends Error {
  constructor(message: string, public code: string) {
    super(message);
  }
}

// エラーの投げ方
throw new AppError('Not found', 'NOT_FOUND');
```

### タイプ定義

```typescript
// 明示的なタイプを使用
interface User {
  id: string;
  email: string;
  name: string;
  createdAt: Date;
}

// 関数のタイプ注釈
function createUser(data: CreateUserInput): Promise<User> {
  // 実装
}
```

## ワークフロー

### 機能開発

1. 機能ブランチを作成
   ```bash
   git checkout -b feat/feature-name
   ```

2. 実装とテスト
   ```bash
   pnpm test
   ```

3. プルリクエストを作成
   ```bash
   git push -u origin feat/feature-name
   ```

4. コードレビュー
5. マージ

### コミットメッセージ

Conventional Commits を使用：

```
type(scope): subject

body

footer
```

**Type:**
- `feat`: 新機能
- `fix`: バグ修正
- `docs`: ドキュメント
- `style`: スタイル
- `refactor`: リファクタリング
- `test`: テスト
- `chore`: チェア

**例:**
```
feat(auth): JWT 認証を実装

- ログイン/ログアウト機能
- トークンリフレッシュ
- パスワードリセット

Closes #123
```

## テスト方針

### ユニットテスト

- Vitest を使用
- 80% 以上のカバレッジ
- 各機能にテスト

```typescript
import { describe, it, expect } from 'vitest';
import { add } from '@/utils/math';

describe('add', () => {
  it('2 つの数を加算する', () => {
    expect(add(1, 2)).toBe(3);
  });
});
```

### 統合テスト

- API エンドポイントのテスト
- データベースとの連携

```typescript
import { describe, it, expect } from 'vitest';
import request from 'supertest';
import app from '@/app';

describe('GET /api/users', () => {
  it('ユーザーリストを返す', async () => {
    const response = await request(app).get('/api/users');
    expect(response.status).toBe(200);
    expect(Array.isArray(response.body)).toBe(true);
  });
});
```

## デプロイメント

### プロダクションビルド

```bash
pnpm build
```

### Docker

```bash
docker build -t my-app .
docker run -p 3000:3000 my-app
```

### 環境変数

```bash
NODE_ENV=production
DATABASE_URL=postgresql://...
JWT_SECRET=...
```

## セキュリティ

- CORS を設定
- リクエストレート制限
- インプットバリデーション
- SQL インジェクション対策
```

### ステップ 3: AGENTS.md の使用

Claude Code や他の AI エージェントでプロジェクトを開く：

```bash
# プロジェクトディレクトリで Claude を起動
cd my-ai-project
claude
```

```
# プロンプト
@AGENTS.md を参照して、新しいユーザー認証機能を実装してください
```

エージェントは AGENTS.md から以下を理解：

- プロジェクトの技術スタック
- コーディング規約
- テスト方針
- ワークフロー

---

## 進階：CLAUDE.md の階層構造と実行契約

### CLAUDE.md の読み込み優先順位

```
~/.claude/CLAUDE.md          ← グローバル（常に読み込まれる）
    +
{project_root}/CLAUDE.md     ← プロジェクトルート
    +
{current_dir}/CLAUDE.md      ← カレントディレクトリ（深いほど上書き）
    +
{subdir}/CLAUDE.md           ← サブディレクトリ（最も優先される）
```

内容は **すべて連結** されてコンテキストに注入されます（上書きではなく追加）。

### 実行契約としての CLAUDE.md

```markdown
# プロジェクト名 — CLAUDE.md（実行契約版）

## Allowed（許可する操作）
- rg (ripgrep)
- git status / git diff / git log
- npm test / npm run lint
- ファイルの読み取り・編集

## Disallowed（禁止する操作）
- git reset --hard
- git push --force
- rm -rf
- .env ファイルのコミット

## Edit Scope（編集可能範囲）
- src/**
- tests/**
- docs/**（README のみ）

## Review Gate（完了条件）
- 破壊的変更なし
- 新規ロジックにはテストが追加されている
- lint エラーがゼロ
```

### CLAUDE.md あり/なし比較

**CLAUDE.md なし → 汎用的な出力:**

```tsx
// Claude の出力（CLAUDE.md なし）
import React from 'react';

export default function Button({ onClick, children }: any) {  // ← any 型
  return (
    <button onClick={onClick} style={{ padding: '10px' }}>    // ← インラインスタイル
      {children}
    </button>
  );
}
```

**CLAUDE.md あり → 規約に準拠した出力:**

```tsx
// Claude の出力（CLAUDE.md あり）
import { ReactNode } from 'react';

interface ButtonProps {                    // ← 型定義（any 禁止のため）
  onClick: () => void;
  children: ReactNode;
  variant?: 'primary' | 'secondary';
}

export const Button = ({                  // ← Named Export（default 禁止のため）
  onClick,
  children,
  variant = 'primary'
}: ButtonProps) => {
  const base = 'px-4 py-2 rounded font-medium';
  return (
    <button className={`${base} ...`}>    // ← Tailwind のみ
      {children}
    </button>
  );
};
```

---

## 進階：高度な AGENTS.md

### ステップ 4: スキル統合

AGENTS.md にスキルを統合：

```markdown
# プロジェクトガイド

## スキル

このプロジェクトでは以下のスキルが利用可能です：

@.claude/skills/database/SKILL.md
@.claude/skills/api/SKILL.md
@.claude/skills/testing/SKILL.md

## スキルの使用法

```bash
# データベース操作スキルを使用
@database スキルを使用して、ユーザーテーブルを作成

# API スキルを使用
@api スキルを使用して、REST エンドポイントを作成
```
```

### ステップ 5: サブエージェント定義

AGENTS.md にサブエージェントを定義：

```markdown
## サブエージェント

このプロジェクトでは以下のサブエージェントが利用可能です：

### リサーチエージェント

```bash
@.claude/subagents/researcher.md
```

### コーディングエージェント

```bash
@.claude/subagents/coder.md
```

### テストエージェント

```bash
@.claude/subagents/tester.md
```

### 使用例

```bash
# 調査と実装
@.claude/subagents/researcher.md "React Query のベストプラクティスを調査"
@.claude/subagents/coder.md "調査結果に基づいて実装"
```
```

### ステップ 6: MCP サーバー設定

AGENTS.md に MCP サーバーを設定：

```markdown
## MCP サーバー

このプロジェクトでは以下の MCP サーバーが使用可能です：

### PostgreSQL

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://localhost/mydb"
      }
    }
  }
}
```

### ファイルシステム

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "./data"]
    }
  }
}
```

### 使用例

```bash
# PostgreSQL でクエリ実行
postgres:users SELECT * FROM users;

# ファイルシステムからファイル読み込み
filesystem:data cat config.json;
```
```

---

## ベストプラクティス

### 1. 明確さと具体性

**悪い例:**
```markdown
## コーディング規約
コードは綺麗に書く
```

**良い例:**
```markdown
## コーディング規約

### 関数の長さ
- 関数は 50 行以内にする
- 複雑なロジックは小さな関数に分割

### エラーハンドリング
- try-catch を使用
- 具体的なエラーメッセージを提供
- ログにエラーを記録
```

### 2. 例の提供

**悪い例:**
```markdown
## API の作成
API エンドポイントを作成する
```

**良い例:**
```markdown
## API の作成

### ルートの定義

```typescript
import { Router } from 'express';

const router = Router();

router.get('/users', async (req, res) => {
  const users = await userService.findAll();
  res.json(users);
});

export default router;
```

### エラーハンドリング

```typescript
router.post('/users', async (req, res, next) => {
  try {
    const user = await userService.create(req.body);
    res.status(201).json(user);
  } catch (error) {
    next(error);
  }
});
```
```

### 3. 更新の維持

AGENTS.md はプロジェクトとともに進化：

```markdown
## 更新履歴

### 2026-03-01
- TypeScript 5.4 にアップグレード
- 新しいテスト方針を追加

### 2026-02-15
- Prisma を追加
- データベーススキーマを更新
```

### 4. 参照の活用

関連リソースを参照：

```markdown
## 関連ドキュメント

- [API ドキュメント](./docs/api.md)
- [デプロイメントガイド](./docs/deployment.md)
- [トラブルシューティング](./docs/troubleshooting.md)
```

### 5. プロジェクト固有の最適化

プロジェクト固有の情報を追加：

```markdown
## プロジェクト固有の注意

### データベース

- PostgreSQL 15 を使用
- Row Level Security を有効化
- パーティショニングを適用

### キャッシュ

- Redis を使用
- TTL: 1 時間
- キー形式：`{resource}:{id}`
```

---

## AGENTS.md ジェネレーター

手動で書く代わりに、ジェネレーターを使用：

```bash
# AGENTS.md ジェネレーターを起動
npx agents-md-generator

# プロンプトに従って回答
# - プロジェクトタイプを選択
# - 技術スタックを選択
# - ワークフローを選択
```

または、オンラインジェネレーターを使用：
- https://devtk.ai/agents-md-generator

---

## まとめ

AGENTS.md は、AI エージェントがプロジェクトを理解し、効果的に作業するための重要なリソースです。明確で包括的な AGENTS.md を作成することで、エージェントの生産性とコードの品質を向上できます。

### 次のステップ

1. [Skills](./01-skills.md) - カスタムスキルの作成
2. [Subagent](./02-subagent.md) - サブエージェントの定義
3. [OpenClaw](./04-openclaw.md) - 自律エージェント

---

*関連リソース:*
- [AGENTS.md Specification](https://www.agents.md/)
- [Agentic AI Foundation](https://aaifoundation.org/)
- [DevTK AGENTS.md Guide](https://devtk.ai/)