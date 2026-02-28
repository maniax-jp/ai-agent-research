# CLAUDE.md / Agent.md — エージェントへの永続的指示ファイル

**カテゴリ:** Claude Code / Agent SDK
**導入時期:** 2025年（Claude Code 1.x〜）
**ステータス:** 本番利用可能（2026年現在 / 標準的な手法として定着）

---

## 1. 概要

**CLAUDE.md** は Claude エージェントに「このプロジェクトについて何を知っておくべきか」を伝える永続的な指示ファイルです。

- 会話のたびに同じ説明をしなくてよい
- チーム全員が同じコンテキストをClaudeに与えられる
- 階層構造（グローバル → プロジェクト → サブディレクトリ）でスコープを制御

### CLAUDE.md の2つの役割

Codex の AGENTS.md の設計思想を参照すると、CLAUDE.md には2つの側面があります：

| 側面 | 説明 | 主なセクション |
|------|------|--------------|
| **プロジェクト知識** | 「何を作っているか」をエージェントに理解させる | 概要・技術スタック・ディレクトリ構造 |
| **実行契約** | 「何をしてよいか・何をしてはいけないか」を定義する | Allowed/Disallowed・Review Gate |

**実行契約としての CLAUDE.md** は、エージェントの逸脱を防ぐための安全装置として機能します。

### Agent.md との関係

| ファイル名 | スコープ | 用途 |
|-----------|---------|------|
| `CLAUDE.md` | プロジェクト全体 | アーキテクチャ・規約・常に知っておくべき情報 |
| `Agent.md` | 特定エージェント向け | 特定サブエージェントへの専門指示 |
| `.claude/settings.json` | Claude Code 設定 | ツール・パーミッション設定 |

> 2026年時点では `CLAUDE.md` が主流。`Agent.md` は特定エージェントへの専門的な指示に使われます。

---

## 2. CLAUDE.md の読み込み優先順位

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

---

## 3. CLAUDE.md の基本構造

```markdown
# プロジェクト名

## プロジェクト概要
このリポジトリは○○を行う△△サービスです。

## 技術スタック
- 言語: TypeScript 5.x
- フレームワーク: Next.js 15
- データベース: PostgreSQL 16 + Prisma ORM
- テスト: Vitest + Playwright

## ディレクトリ構造
src/
├── app/        # Next.js App Router
├── lib/        # 共有ユーティリティ
├── components/ # UIコンポーネント
└── server/     # サーバーサイドロジック

## コーディング規約
- コミットメッセージは Conventional Commits 形式
- 関数名はキャメルケース、ファイル名はケバブケース
- テストファイルは *.test.ts または *.spec.ts

## 禁止事項
- console.log のコミット禁止（logger を使うこと）
- any 型の使用禁止
- 直接 fetch を使わず apiClient を使うこと

## よく使うコマンド
\`\`\`bash
npm run dev      # 開発サーバー起動
npm test         # テスト実行
npm run typecheck # 型チェック
\`\`\`
```

---

## 4. ハンズオン：実践的な CLAUDE.md を設定する

### Step 1: グローバル CLAUDE.md（全プロジェクト共通設定）

```bash
cat > ~/.claude/CLAUDE.md << 'EOF'
# グローバル設定

## 基本方針
- 日本語で回答すること（コードコメントは英語でも可）
- コードを変更する前に必ず現在のコードを読んでから変更すること
- 不必要なリファクタリングは行わないこと
- セキュリティリスクを発見した場合は必ず報告すること

## Git の扱い
- コミットは明示的に指示された場合のみ行う
- force push は絶対にしない
- .env ファイルは絶対にコミットしない

## 環境情報
- OS: Windows 11 / WSL2 Ubuntu 22.04
- Node.js: 22.x (bun を優先)
- Python: 3.12
EOF
```

### Step 2: プロジェクト固有の CLAUDE.md

```bash
cd ~/my-project

cat > CLAUDE.md << 'EOF'
# my-project — ECサイトバックエンド

## 概要
Node.js + Fastify + PostgreSQL で構築したECサイトのバックエンドAPI。
月間100万リクエストを処理する本番サービス。

## 重要な設計原則
- **ドメイン駆動設計 (DDD)**: src/domain/ 配下にドメインロジック
- **CQRS**: 読み取りと書き込みを分離（src/query/ / src/command/）
- **イベント駆動**: 主要な状態変化はイベントで通知（src/events/）

## データベース
- 本番: PostgreSQL 16（接続は DATABASE_URL 環境変数）
- テスト: Docker で起動する分離DB（npm run test:db）
- マイグレーション: Prisma Migrate を使用（手動SQL禁止）

## API 設計
- RESTful API（OpenAPI 3.1仕様 → docs/openapi.yaml）
- 認証: JWT（access token 15分、refresh token 7日）
- レート制限: 100req/min per IP

## テスト
- ユニットテスト: Vitest（src/**/*.test.ts）
- E2Eテスト: Playwright（e2e/**/*.spec.ts）
- テストカバレッジ 80% 以上を維持すること

## 注意事項
- src/legacy/ は触らないこと（移行中）
- Payment 関連のコードは必ずセキュリティレビューを通すこと
- ログに個人情報を含めないこと（GDPR対応）
EOF
```

### Step 3: サブディレクトリ固有の設定

```bash
cat > src/payment/CLAUDE.md << 'EOF'
# Payment モジュール — 追加指示

## セキュリティ要件（最優先）
このディレクトリのコードを変更する場合：
1. 変更前後のセキュリティ影響を必ず説明すること
2. PCI DSS の要件に準拠すること
3. カード番号・CVVは絶対にログに出力しないこと

## 使用ライブラリ
- 決済処理: Stripe SDK v15 のみ使用（他のSDK追加禁止）
- 暗号化: Node.js の crypto モジュール（bcrypt は禁止）

## テスト
- Stripe のテストモードを使用（本番キーを使ったテスト禁止）
- 全ての決済フローに対してハッピーパスとエラーケースのテストが必須
EOF
```

### Step 4: CLAUDE.md の効果を確認する

```bash
# Claude Code を起動
claude

# CLAUDE.md の内容が反映されているか確認
> このプロジェクトの認証方式は？
# → JWT（CLAUDE.mdに記載した内容）を答えてくれる

> payment モジュールに新しい決済メソッドを追加したい
# → payment/CLAUDE.md の制約（Stripeのみ、PCI DSS等）を守った提案をしてくれる
```

---

## 5. CLAUDE.md あり/なし比較：エージェントの応答品質の変化

Gemini の AGENTS.md 検証手法を Claude に応用した比較です。「CLAUDE.md がある場合とない場合」でエージェントが返すコードがどう変わるかを確認できます。

### 検証プロジェクト設定

```bash
mkdir -p ~/claude-comparison-test
cd ~/claude-comparison-test
npm init -y
```

### CLAUDE.md なし → 汎用的な出力

```
# ユーザー指示（CLAUDE.md なし）
"新しいButtonコンポーネントを作ってください"
```

```tsx
// Claude の出力（CLAUDE.md なし）
// ← 汎用的なReactの書き方（プロジェクト規約を知らない）
import React from 'react';

export default function Button({ onClick, children }: any) {  // ← any型
  return (
    <button onClick={onClick} style={{ padding: '10px' }}>    // ← インラインスタイル
      {children}
    </button>
  );
}
```

### CLAUDE.md あり → 規約に準拠した出力

```bash
cat > CLAUDE.md << 'EOF'
# MyWebApp

## Tech Stack
- Next.js 14 (App Router)
- TypeScript（厳格モード）
- Tailwind CSS

## Coding Rules (CRITICAL)
1. **絶対に** `any` 型を使用しない。必ず適切な型を定義すること。
2. コンポーネントは `export default` ではなく Named Export を使用すること。
3. スタイルは Tailwind CSS のみ使用すること（インラインスタイル禁止）。
4. すべてのコンポーネントに Props のインターフェース定義を書くこと。
EOF
```

```
# ユーザー指示（CLAUDE.md あり）
"新しいButtonコンポーネントを作ってください"
```

```tsx
// Claude の出力（CLAUDE.md あり）
// ← CLAUDE.md のルールをすべて遵守
import { ReactNode } from 'react';

interface ButtonProps {                    // ← 型定義（any禁止のため）
  onClick: () => void;
  children: ReactNode;
  variant?: 'primary' | 'secondary';
  className?: string;
}

export const Button = ({                  // ← Named Export（default禁止のため）
  onClick,
  children,
  variant = 'primary',
  className = ''
}: ButtonProps) => {
  const base = 'px-4 py-2 rounded font-medium transition-colors';
  const variants = {
    primary: 'bg-blue-500 text-white hover:bg-blue-600',
    secondary: 'bg-gray-200 text-gray-800 hover:bg-gray-300',
  };
  return (
    <button                               // ← Tailwindのみ（インラインスタイル禁止のため）
      onClick={onClick}
      className={`${base} ${variants[variant]} ${className}`}
    >
      {children}
    </button>
  );
};
```

この差が、**CLAUDE.md が「実行契約」として機能している**証拠です。

### 実行契約としての CLAUDE.md テンプレート

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
- docs/**（READMEのみ）

## Review Gate（完了条件）
- 破壊的変更なし
- 新規ロジックにはテストが追加されている
- lint エラーがゼロ
```

---

## 6. Agent.md — 特定エージェント向けの指示

特定のサブエージェントに専用の指示を与える場合に使います。

```bash
mkdir -p .claude/agents

cat > .claude/agents/refactor-agent.md << 'EOF'
# Refactoring Agent への指示

あなたはコードリファクタリング専門のエージェントです。

## リファクタリング原則
1. **外部動作を変えない** — テストが通ることを確認してから次に進む
2. **小さなステップ** — 一度に1つのリファクタリングのみ適用
3. **コミット単位** — 各リファクタリングを独立したコミットにする

## 適用するリファクタリング技法（優先順）
1. Extract Function（関数が長すぎる場合）
2. Rename Variable（意味不明な変数名）
3. Remove Duplication（DRY原則）
4. Replace Conditional with Polymorphism

## 禁止事項
- アーキテクチャの大幅変更
- ライブラリの変更・追加
- テストコードの削除
EOF
```

---

## 6. CLAUDE.md のベストプラクティス（2026年）

### 書くべき内容 vs 書かない内容

```markdown
# ✅ 書くべき内容
- 静的な知識（変わりにくい規約・アーキテクチャ）
- 「なぜ」の説明（レガシーコードを触るな → 理由）
- よく間違えやすい点
- プロジェクト固有の用語定義

# ❌ 書かない内容
- 動的な情報（現在のタスク、一時的な状態）
- 一般的なプログラミングの原則（知っている）
- 超長文の仕様書（必要な部分だけ）
- 個人情報・シークレット
```

### サイズ管理

```
CLAUDE.md は 200行以内を推奨。
長くなる場合は以下を参照形式にする：

## 詳細仕様
詳細なAPI仕様は docs/api-spec.md を参照。
データベーススキーマは prisma/schema.prisma を参照。
```

### `@import` でファイルを参照（Claude Code 2.x+）

```markdown
<!-- CLAUDE.md -->
# プロジェクト設定

@docs/architecture.md
@docs/coding-standards.md
@.claude/team-preferences.md
```

`@` プレフィックスで他のファイルをインポートできます。必要なときだけ読み込まれるため、コンテキスト効率が上がります。

---

## 参考リンク

- [Understanding Claude Code's Full Stack](https://alexop.dev/posts/understanding-claude-code-full-stack/)
- [A Guide to Claude Code 2.0](https://sankalp.bearblog.dev/my-experience-with-claude-code-20-and-how-to-get-better-at-using-coding-agents/)
