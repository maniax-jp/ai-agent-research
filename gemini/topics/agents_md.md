---
description: 2026年以降のAIエージェントの技術的動向：AGENTS.md
---

# AGENTS.md によるマシン向けコンテキストの標準化

## 1. 概要と概念

**AGENTS.md**（または `AGENTS.md`）は、AIエージェントがソフトウェアプロジェクトを正しく理解し、操作するための「AI向けREADME」として機能するオープン標準のマークダウンファイルです。

人間の開発者がリポジトリに参画する際に `README.md` や `CONTRIBUTING.md` を読むように、AIコーディングアシスタント（Cursor, Copilot, 独立型エージェントなど）はリポジトリのルートにある `AGENTS.md` を最初に読み込みます。

### なぜAGENTS.mdが必要なのか？

*   **コンテキストの共通化**: すべての開発メンバー（および彼らが使うAIツール）が同じコーディング規約や設計思想を共有するため。
*   **「実行契約」としての機能**: 単なる自然言語のメモではなく、AIに「何をしてよいか（許可コマンド）」「何をしてはいけないか（禁止操作など）」を明確化する契約書（Contract）として働きます。
*   **ハルシネーションの抑制**: AIに「このプロジェクトではReactではなくVueを使っている」「DBのパスワードは環境変数から読む」といった絶対的なルールを事前に与えることで、誤ったコードの生成を防ぎます。
*   **環境依存アクセスの定義**: AIがテストを実行したり、ビルドしたりする際の具体的なコマンド（例: `npm run test:e2e`）を定義します。

### 読み込み優先順位（階層構造）

AIエージェントは通常、複数のスコープで定義された `AGENTS.md`（または `CLAUDE.md` など各ツールの標準ファイル）をマージして解釈します。

1.  グローバル設定（例：`~/.claude/CLAUDE.md`）
2.  プロジェクト・ルート設定（例：`./AGENTS.md`）
3.  サブディレクトリ設定（例：`./src/payment/AGENTS.md`）

より深いディレクトリの設定が、グローバルな設定を上書きまたは拡張します。これにより、「決済モジュール周りだけは特に厳密なセキュリティ確認を指示する」といった制御が可能です。

## 2. AGENTS.md の基本構造

AGENTS.md は通常、以下のようなセクションで構成されます。特に決まった厳格な文法はありませんが、LLMがパースしやすいように見出しとリスト構造を多用します。

1.  **プロジェクトの目的**: 何を作るシステムなのかの全体像
2.  **技術スタック (Tech Stack)**: 使用している言語、フレームワーク、ライブラリとそのバージョン
3.  **ディレクトリ構造**: どこに何が配置されているかのマッピング
4.  **コーディング規約 (Rules / Conventions)**: 命名規則、避けるべきアンチパターン
5.  **コマンドライン定義**: ビルド、テスト、リンターなどの実行コマンド

## 3. 各種AIツールにおけるマシングルールファイルのサポート状況（2026年時点）

2026年現在、各主要AIコーディングアシスタントは `AGENTS.md` またはそれに類するプロジェクトルールの読み込みをネイティブにサポートしています。

| AI ツール (2026年) | サポートするデフォルトファイル名 | 特徴・対応状況 |
| :--- | :--- | :--- |
| **Claude Code** | `CLAUDE.md` | グローバル (`~/.claude/CLAUDE.md`) とプロジェクト直下の階層マージに完全対応。コマンドの定義やファイルパスの自動補完機能と強力に連携。 |
| **Codex** | `AGENTS.md` | 本稿で解説した「実行契約（Execution Contract）」の概念を最も強く押し出しており、許容/禁止操作の制限に厳格。 |
| **Antigravity** (Gemini) | `AGENTS.md` (またはカスタムルール) | リポジトリのコンテキストとして柔軟にルールを読み込み。ユーザー固有のプレファレンスや動的コンテキスト検索と組み合わせて動作。 |
| **GitHub Copilot** | `.github/copilot-instructions.md` | Copilot Chat用にリポジトリレベルでのカスタムインストラクションを記述。VS Code等のエディタ設定（`.vscode/settings.json` の `github.copilot.chat.codeGeneration.instructions`）とも連携可能。 |

## 4. ハンズオン：AGENTS.md を作成してAIの挙動を制御する

実際にサンプルプロジェクトのルートディレクトリに配置する `AGENTS.md` を作成してみましょう。

### ステップ 1: AGENTS.md の作成

プロジェクトのルート（例: `/my-web-app/`）に、以下の内容で `AGENTS.md` というファイルを作成します。

```markdown
# AGENTS.md for MyWebApp

## Role & Purpose
あなたはTypeScriptとNext.jsのシニアエンジニアとして振る舞います。
このプロジェクトは、社内のタスク管理を行うためのシンプルなWebダッシュボードです。

## Tech Stack
- Frontend: Next.js 14 (App Router)
- Language: TypeScript (厳格モード)
- Styling: Tailwind CSS
- State Management: Zustand

## Code Architecture
- `src/app/`: Next.jsのルーティング定義のみを配置。ビジネスロジックは書かない。
- `src/components/`: 再利用可能なUIコンポーネント。状態(State)を持たないこと。
- `src/lib/`: API呼び出しやヘルパー関数。
- `src/stores/`: Zustandのストア定義。

## Coding Rules (CRITICAL)
1. **絶対に** `any` 型を使用しないでください。必ず適切なインターフェースや型を定義すること。
2. データの取得はクライアントサイドフェッチではなく、Server Componentsを利用してください。
3. すべてのコンポーネントは `export default` ではなく Named Export (`export const`) を使用してください。

## Allowed Commands (許可された操作)
- `npm run dev`, `npm run lint`, `npm run test`
- `git status`, `git diff`

## Disallowed Commands (禁止された操作)
- `git reset --hard`
- `rm -rf`
- `npm publish`

## Edit Scope (編集可能範囲)
- `src/components/**`
- `src/app/**`
- `src/stores/**`

## Review Gate (レビュー合格基準)
- 破壊的変更がないか
- 新規ロジックに対するテスト（90% カバレッジ）が追加されていること
```

### ステップ 2: AIツール（Cursor等）での効果検証（シミュレーション）

この `AGENTS.md` が存在する場合としない場合で、AIに「新しいボタンコンポーネントを作って」と指示した際の挙動の違いを見てみましょう。

**AGENTS.md が無い場合（一般的な回答）:**
```tsx
// AIは一般的なReactの書き方をしがち
import React from 'react';

export default function Button({ onClick, children }: any) { // any型が使われるリスク
  return (
    <button onClick={onClick} style={{ padding: '10px' }}> {/* インラインスタイル */}
      {children}
    </button>
  );
}
```

**AGENTS.md が有る場合（ルールに沿った回答）:**
```tsx
// Rule: Named Exportを使用, Tailwindを使用, any型の禁止 を遵守
import { ReactNode } from 'react';

interface ButtonProps {
  onClick: () => void;
  children: ReactNode;
  className?: string; // 型の明示
}

export const Button = ({ onClick, children, className = '' }: ButtonProps) => {
  return (
    <button 
      onClick={onClick} 
      className={`px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600 ${className}`}
    >
      {children}
    </button>
  );
};
```

## 4. `AGENTS.md` の CI による「契約強制（Contract Enforcement）」

2026年のベストプラクティスでは、`AGENTS.md` がただのドキュメントではなく「エージェントの振る舞いを拘束する契約」として機能するため、CI/CD パイプラインでその記述内容を検証します。

### CI 検証スクリプトの例（PowerShell例）

例えば、プロジェクトに必須の「許可／禁止コマンド」セクションが欠落していないかをチェックするタスクを走らせます。

```powershell
# validate-agents.ps1 の例
if (!(Test-Path AGENTS.md)) { throw "AGENTS.md が存在しません" }
$txt = Get-Content AGENTS.md -Raw
if ($txt -notmatch "## Allowed Commands") { throw "Allowed Commands セクションが存在しません！エージェントが暴走する危険があります" }
if ($txt -notmatch "## Disallowed Commands") { throw "Disallowed Commands セクションが存在しません！" }
Write-Output "✅ AGENTS.md の形式は健全です"
```

これにより、人間がPRを出した際に「AIエージェント向けの安全装置（AGENTS.md）」が正しく更新されているかを自動で担保でき、エージェントの予測不能な行動（逸脱）をプロジェクト単位で強力に制御できます。

## 5. 運用上のベストプラクティス

*   **肥大化させない**: `AGENTS.md` が長すぎると、AIが重要なルールを見落とします。本当に重要なアーキテクチャや「やってはいけないこと（アンチパターン）」に絞って記述します。
*   **AGENTS.md自体をレビューする**: チームメンバーがPRを出す際、AGENTS.mdの変更もレビュー対象にします。
*   **複数ファイルへの分割と参照**: 内容が巨大になる場合は、`AGENTS.md` 内で `@docs/architecture.md` のように別ファイルを参照（インポート）させる形にしてコンテキスト消費を最適化します。
