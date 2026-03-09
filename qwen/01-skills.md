# Skills システム - AI エージェントに新しい機能を与える仕組み

**作成者:** Qwen
**日付:** 2026 年 3 月 9 日

---

## 目次

1. [概要](#概要)
2. [Skills と MCP の違い](#skills と-mcp の違い)
3. [Skills のアーキテクチャ](#skills のアーキテクチャ)
4. [ハンズオン：カスタムスキルの作成](#ハンズオンカスタムスキルの作成)
5. [進階：スキルシステムの拡張](#進階スキルシステムの拡張)
6. [ベストプラクティス](#ベストプラクティス)

---

## 概要

**Skills**は、AI エージェントに新しい機能やドメイン知識を追加するための仕組みです。2026 年以降、`Skills` は「プロンプト集」ではなく、`実行可能な業務ユニット`として扱う設計が主流です。

### 最小構成

再利用可能なスキルには以下の構成が推奨されます：

```
skill-name/
├── SKILL.md      # スキル定義（必須）
├── scripts/      # 実行可能スクリプト（オプション）
└── templates/    # 出力テンプレート（オプション）
```

### Skills の特徴

- **軽量:** サーバーインフラが不要
- **ポータブル:** プロジェクトに同梱可能
- **バージョン管理可能:** Git で管理可能
- **オフライン対応:** ネットワーク接続不要
- **入出力契約:** 明確な入出力形式で再利用可能

---

## Skills と MCP の違い

| 特徴 | Skills | MCP |
|------|--------|-----|
| **データタイプ** | 静的（ドキュメント、パターン） | 動的（リアルタイムデータ） |
| **接続** | ファイルベース | ランタイム接続 |
| **セットアップ** | 0 設定 | サーバー起動が必要 |
| **使用ケース** | コーディングパターン、SDK ドキュメント | データベース、API、外部サービス |
| **パフォーマンス** | 高速（ファイル読み込みのみ） | 遅延（ネットワーク呼び出し） |

### 選択ガイド

**Skills を選ぶ場合:**
- 一貫したコーディングパターンが必要
- チーム全員がゼロセットアップでコンテキストを得たい
- オフラインアクセスが重要

**MCP を選ぶ場合:**
- データが頻繁に更新される
- リアルタイムのアクションが必要
- 動的なクエリや操作が必要

---

## Skills のアーキテクチャ

### スキルファイル構造

```
my-project/
├── AGENTS.md              # メインのプロジェクト指示
├── .claude/
│   └── skills/
│       ├── database-operations/
│       │   └── SKILL.md   # データベース操作スキル
│       └── api-integration/
│           └── SKILL.md   # API 統合スキル
└── src/
    └── ...
```

### SKILL.md の標準フォーマット

```markdown
---
name: skill-name
description: スキルの説明
version: 1.0.0
author: your-name
tags: [tag1, tag2]
---

# Skill Name

## 概要

このスキルの説明。

## 使用法

### 前提条件

必要な前提条件。

### インストール

インストール手順。

### 入力データ形式

期待される入力形式。

### 出力データ形式

生成される出力形式。

### 例

```bash
# 使用例
```

## エラーハンドリング

エラー処理の説明。

## セキュリティ考慮事項

セキュリティに関する注意。
```

---

## ハンズオン：カスタムスキルの作成

### ステップ 1: 環境準備

```bash
# プロジェクトディレクトリを作成
mkdir ai-skills-demo
cd ai-skills-demo

# .claude/skills ディレクトリを作成
mkdir -p .claude/skills
```

### ステップ 2: 単純なスキル作成（Git 操作スキル）

```bash
# スキルディレクトリを作成
mkdir -p .claude/skills/git-operations
```

`.claude/skills/git-operations/SKILL.md` を作成:

```markdown
---
name: git-operations
description: Git 操作の標準パターンとベストプラクティス
version: 1.0.0
author: Qwen
tags: [git, version-control, workflow]
---

# Git Operations Skill

## 概要

このスキルは、Git 操作に関する標準パターンとベストプラクティスを提供します。

## コミットメッセージ規約

### 形式

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Type の一覧

- `feat`: 新機能
- `fix`: バグ修正
- `docs`: ドキュメント変更
- `style`: コードスタイルの変更（機能に影響なし）
- `refactor`: リファクタリング
- `test`: テストの追加/修正
- `chore`: ビルドタスクや Auxiliaries の変更

### 例

```
feat(auth): ユーザー認証機能を追加

- JWT トークンベースの認証を実装
- パスワードハッシングに bcrypt を使用
- リフレッシュトークン機能を追加

Closes #123
```

## ブランチ戦略

### ブランチネーミング

```
feat/feature-name
fix/bug-description
hotfix/critical-fix
release/version-number
```

### Git Flow

```bash
# 1. 開発ブランチを作成
git checkout -b feat/feature-name

# 2. 変更をコミット
git add .
git commit -m "feat: 機能を実装"

# 3. メインブランチに統合
git checkout main
git pull origin main
git merge feat/feature-name
```

## 使用例

### 新機能の実装

```bash
# 1. 新ブランチを作成
git checkout -b feat/user-profile

# 2. 変更を加えてコミット
git add src/user/
git commit -m "feat(user): プロフィール機能を追加"

# 3. リモートにプッシュ
git push -u origin feat/user-profile
```

### バグ修正

```bash
# 1. 修正ブランチを作成
git checkout -b fix/login-bug

# 2. 修正をコミット
git commit -m "fix(auth): ログイン時のエッジケースを修正"

# 3. メインにマージ
git checkout main
git merge fix/login-bug
```

## よくあるパターン

### 部分的なコミット

```bash
# 特定のファイルのみステージング
git add src/specific-file.js

# ステージングから除外
git reset HEAD src/exclude-file.js

# インタラクティブに選択
git add -p
```

### コミットの修正

```bash
# 直前のコミットを修正
git commit --amend

# ファイルを誤ってコミットした場合
git reset HEAD src/wrong-file.js
```

## セキュリティ考慮事項

- センシティブな情報をコミットしない
- .gitignore を適切に設定
- Git ヒストリから機密情報を削除する必要がある場合は git-filter-repo を使用
```

### ステップ 3: AGENTS.md の作成

`AGENTS.md` を作成してスキルを参照:

```markdown
# プロジェクトガイド

## 概要

このプロジェクトは、AI エージェントのスキルシステムを実証するデモです。

## スキル

以下のスキルが利用可能です：

@.claude/skills/git-operations/SKILL.md

## 開発環境

### 前提条件

- Node.js 18+
- Git 2.30+

### セットアップ

```bash
npm install
npm run dev
```

## コーディング規約

- TypeScript を使用
- ESLint と Prettier を使用
- 機能ごとにテストを書く

## ワークフロー

1. 機能ブランチを作成
2. 実装とテスト
3. プルリクエストを作成
4. レビュー後にマージ
```

### ステップ 4: スキルの使用

Claude Code や他の AI エージェントを使用する場合：

```bash
# Claude Code を起動
claude

# プロンプトでスキルを参照
@AGENTS.md で新機能を実装してください
```

エージェントは自動的にスキルファイルからコンテキストを読み込み、適切な Git 操作を行います。

---

## 進階：スキルシステムの拡張

### ステップ 5: スクリプトベースのスキル作成

より複雑なスキルでは、スクリプトを統合できます：

```bash
# スクリプトディレクトリを作成
mkdir -p .claude/skills/deployment/scripts
```

`.claude/skills/deployment/SKILL.md` を作成:

```markdown
---
name: deployment
description: デプロイメントワークフローとスクリプト
version: 1.0.0
---

# Deployment Skill

## 概要

このスキルは、デプロイメントワークフローを自動化します。

## 利用可能なスクリプト

### 本番環境へのデプロイ

```bash
uv run scripts/deploy-prod.py
```

### ステージング環境へのデプロイ

```bash
uv run scripts/deploy-staging.py
```

### ロールバック

```bash
uv run scripts/rollback.py <version>
```

## 使用例

```bash
# 本番環境へのデプロイ
uv run scripts/deploy-prod.py --confirm

# 特定のバージョンへのロールバック
uv run scripts/rollback.py v1.2.3
```
```

`scripts/deploy-prod.py` を作成:

```python
#!/usr/bin/env python3
"""本番環境へのデプロイスクリプト"""

import subprocess
import sys
from pathlib import Path

def deploy():
    """本番環境にデプロイする"""
    print("本番環境へのデプロイを開始します...")

    # ビルド
    print("ビルド中...")
    subprocess.run(["npm", "run", "build"], check=True)

    # テスト実行
    print("テスト実行中...")
    subprocess.run(["npm", "test"], check=True)

    # デプロイ
    print("デプロイ中...")
    subprocess.run(["npm", "run", "deploy:prod"], check=True)

    print("デプロイが完了しました！")

if __name__ == "__main__":
    deploy()
```

### ステップ 6: MCP サーバーとの統合

Skills と MCP を組み合わせて使用：

```markdown
---
name: database-operations
description: データベース操作とクエリ
---

# Database Operations Skill

## 概要

このスキルは、データベース操作のベストプラクティスを提供します。

## MCP サーバー

リアルタイムのデータベース操作には、以下の MCP サーバーを使用：

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"]
    }
  }
}
```

## クエリパターン

### 安全なクエリ構築

```sql
-- 推奨：パラメータ化されたクエリ
SELECT * FROM users WHERE id = $1;

-- 避ける：文字列連結
-- SELECT * FROM users WHERE id = ' || user_input || ';
```
```

---

## ベストプラクティス

### 1. スキルの分割

1 つのスキルで全てをカバーしようとせず、小さく分割：

```
.claude/skills/
├── authentication/
├── database/
├── api/
└── deployment/
```

### 2. 進段的開示

すべての情報を一度に提示せず、必要に応じて開示：

```markdown
# スキル名

## 概要

簡潔な概要。

## 詳細

<!-- 詳細は必要に応じて開示 -->

### 高度な使用法

<!-- 高度な情報は折りたたむ -->
```

### 3. 例の提供

具体的な例を提供：

```markdown
## 使用例

### シンプルな例

```bash
command --option value
```

### 複雑な例

```bash
command --option1 value1 --option2 value2 --flag
```
```

### 4. エラーハンドリング

一般的なエラーとその解決策：

```markdown
## よくあるエラー

### エラー：Connection Refused

**原因:** サーバーが起動していません

**解決:**
```bash
# サーバーを起動
npm run server
```
```

### 5. セキュリティ

- センシティブな情報をスキルファイルに含めない
- 環境変数を使用
- 権限を最小限に

---

## 設計ポイント（codex 基準）

### 入出力契約の明文化

スキルは明確な入出力契約を持つことで再利用可能になります：

```markdown
## Inputs
- target_paths: レビュー対象のファイル/ディレクトリ
- severity_threshold: 重大度の閾値（critical, high, medium, low）

## Outputs
- findings.md: 重大度順の指摘一覧
- summary.json: 統計情報（JSON 形式）
```

### 副作用の限定

スキルが操作できる範囲を明確に限定：

```markdown
## Scope
- Read: repo 全体
- Write: findings.md のみ
- Execute: scripts/list_changed.ps1 のみ
```

### テンプレートの使用

出力形式をテンプレートで固定することで、後段の処理を安定化：

```markdown
# Findings Template

## Critical
- [file:line] 内容
- 根拠
- 修正案

## High
- [file:line] 内容
- 根拠
- 修正案
```

### 失敗しやすい点

- `SKILL.md` が抽象的で毎回出力が揺れる
- 出力形式が固定されず、AgentTeam の後段でパース不能になる
- 書き込み範囲を制限しておらず副作用が広がる

### 次の強化

- `version` を `SKILL.md` に追加
- スキル登録時に lint/検証を CI で自動実行
- スキル単体テスト（最低限：実行可否）を回せる

---

## まとめ

Skills システムは、AI エージェントにドメイン知識と機能を提供する軽量で効果的な方法です。静的コンテキストファイルと動的 MCP サーバーを組み合わせることで、柔軟で強力なエージェントワークフローを構築できます。

### 次のステップ

1. [Subagent](./02-subagent.md) - タスクの委任と並列処理
2. [AGENTS.md](./03-agents.md.md) - プロジェクト仕様書の作成
3. [OpenClaw](./04-openclaw.md) - 自律エージェントの構築

---

*関連リソース:*
- [Anthropic Skills Documentation](https://docs.anthropic.com/)
- [MCP Servers](https://modelcontextprotocol.io/)
- [Skills Repository](https://skills.md/)