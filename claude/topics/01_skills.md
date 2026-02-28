# Skills — 再利用可能なプロンプト + アーティファクトのバンドル

**カテゴリ:** Claude Code / Agent SDK
**導入時期:** 2025年10月（Claude Code 2.0）
**ステータス:** 本番利用可能（2026年現在）

---

## 1. 概要

**Skills** は Claude Code において「再利用可能なプロンプト」を `/skill-name` というスラッシュコマンド形式で呼び出せる仕組みです。

- スクリプト・設定ファイルなどの **アーティファクト** を同梱できる
- スキル本体（`SKILL.md`）は **オンデマンド読み込み** → コンテキスト消費を最小化
- `context: fork` オプションで独立したコンテキストウィンドウ内で実行可能
- ユーザー定義・プロジェクト定義・システム定義の3レイヤーで管理

### MCP との違い

| 比較軸 | Skills | MCP |
|--------|--------|-----|
| 目的 | プロンプトの再利用・構造化 | 外部ツールとの接続 |
| 実行環境 | Claude内部 | 外部サーバー |
| コンテキスト消費 | メタデータのみ（本体はオンデマンド） | ツール呼び出しごと |
| 共有 | ファイルベース（SKILL.md） | サーバーベース |

---

## 2. Skill の構造

Skill は以下のファイル構造で定義します。

```
~/.claude/skills/
└── my-skill/
    ├── SKILL.md          ← スキル本体（指示・プロンプト）
    └── artifacts/        ← 同梱ファイル（スクリプト等）
        ├── setup.sh
        └── template.py
```

### SKILL.md の書式

```markdown
---
name: my-skill
description: "このスキルの説明（Claude がいつ使うか判断するためのメタデータ）"
context: fork          # fork / inherit（省略時は inherit）
tools:                 # 利用を許可するツール（省略時は全ツール）
  - Read
  - Grep
  - Bash
---

# スキル本体の指示

ここに Claude への指示を自然言語で記述します。
${ARTIFACTS_DIR} でアーティファクトディレクトリを参照できます。
```

---

## 3. ハンズオン：コードレビュースキルを作成する

### Step 1: ディレクトリを作成する

```bash
mkdir -p ~/.claude/skills/code-review/artifacts
```

### Step 2: SKILL.md を作成する

```bash
cat > ~/.claude/skills/code-review/SKILL.md << 'EOF'
---
name: code-review
description: "コードレビューを実施する。diff や変更ファイルを受け取り、問題点・改善案・セキュリティ懸念を報告する"
context: fork
tools:
  - Read
  - Grep
  - Glob
---

# コードレビュースキル

あなたはシニアエンジニアとしてコードレビューを行います。

## レビュー観点
1. **正確性** — バグ・ロジックエラー・エッジケース漏れ
2. **セキュリティ** — OWASP Top 10、インジェクション、認証不備
3. **パフォーマンス** — 不必要なN+1クエリ、メモリリーク
4. **可読性** — 命名規則、複雑度、コメント
5. **テスト** — カバレッジ不足、テスト不能な設計

## 出力フォーマット

### 🔴 Critical（要修正）
- [ファイル名:行番号] 問題の説明

### 🟡 Warning（推奨修正）
- [ファイル名:行番号] 問題の説明

### 🟢 Suggestion（任意改善）
- [ファイル名:行番号] 提案内容

### ✅ Summary
変更の概要と総合評価を記述する。
EOF
```

### Step 3: スキルを呼び出す

Claude Code のセッション内で：

```
/code-review src/auth/login.ts を確認してください
```

または対象ファイルを指定せずに：

```
/code-review
```

（Claude がコンテキストから対象ファイルを推測します）

### Step 4: `context: fork` の効果を確認する

`context: fork` を指定すると、スキルは **独立したコンテキストウィンドウ** で実行されます。

```
メインコンテキスト
    │
    ├─ /code-review 呼び出し
    │       │
    │       └─ [Fork] 独立したウィンドウで実行
    │               ├─ SKILL.md の指示を受け取る
    │               ├─ 指定ファイルを読み込む
    │               └─ レビュー結果をメインに返す
    │
    └─ レビュー結果を受け取る（Fork内の詳細コンテキストは破棄）
```

メインコンテキストは汚染されず、長時間のセッションでも効率が維持されます。

---

## 4. 応用パターン

### パターン A: Hot-Reload スキル（Claude Code 2.1+）

Claude Code 2.1.0 で導入された **Skill Hot-Reload** により、SKILL.md を編集すると次の呼び出し時に自動反映されます。Claude の再起動は不要です。

```bash
# スキルを編集
vim ~/.claude/skills/code-review/SKILL.md

# 次の /code-review 呼び出しで変更が反映される
```

### パターン B: プロジェクト固有スキル

グローバルではなく、プロジェクトルートに置くことでプロジェクト限定のスキルを定義できます。

```
my-project/
└── .claude/
    └── skills/
        └── deploy-check/
            └── SKILL.md   ← このプロジェクトだけで使えるスキル
```

### パターン C: アーティファクトを活用したスキル

スクリプトをバンドルして複雑なオペレーションを自動化できます。

```
~/.claude/skills/db-migration/
├── SKILL.md
└── artifacts/
    ├── check_schema.py     ← スキーマ検証スクリプト
    ├── rollback.sh         ← ロールバックスクリプト
    └── migration_template.sql
```

```markdown
<!-- SKILL.md -->
---
name: db-migration
description: "データベースマイグレーションを安全に実行する"
context: fork
tools:
  - Bash
  - Read
  - Write
---

# DBマイグレーションスキル

${ARTIFACTS_DIR}/check_schema.py を実行してスキーマを検証してから
マイグレーションを実施します。問題があれば ${ARTIFACTS_DIR}/rollback.sh を実行します。
```

---

## 5. 入出力契約（I/O Contract）設計

Codex の設計思想に倣い、スキルを「プロンプト集」ではなく**再利用可能な業務ユニット**として扱うには、入出力を明文化することが重要です。

```
~/.claude/skills/code-review/
├── SKILL.md          ← 指示本体（フロー定義）
├── input.md          ← 入力仕様（何を受け取るか）
├── output.json       ← 出力スキーマ（何を返すか）
└── artifacts/
    └── list_changed.sh
```

### input.md の例

```markdown
# code-review の入力仕様

## 必須
- target_paths: レビュー対象のファイルパス（複数可）

## 任意
- severity_threshold: 報告する最低重大度（critical/high/medium/low、デフォルト: medium）
- context: PR番号や変更の目的（あると精度が上がる）

## 入力例
target_paths: ["src/auth.ts", "src/payment.ts"]
severity_threshold: high
context: "PR #123: JWT認証の実装"
```

### output.json の例

```json
{
  "$schema": "http://json-schema.org/draft-07/schema",
  "type": "object",
  "properties": {
    "findings": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["severity", "file", "line", "description"],
        "properties": {
          "severity": { "enum": ["critical", "high", "medium", "low"] },
          "file": { "type": "string" },
          "line": { "type": "integer" },
          "description": { "type": "string" },
          "suggestion": { "type": "string" }
        }
      }
    },
    "summary": { "type": "string" }
  }
}
```

### SKILL.md にバージョンを追加する

```markdown
---
name: code-review
version: "1.2.0"           ← バージョン追加（セマンティックバージョニング推奨）
description: "TypeScript/JavaScriptコードの品質・セキュリティ・パフォーマンスをレビューする"
context: fork
tools:
  - Read
  - Grep
  - Glob
input_schema: ./input.md   ← 入力仕様への参照
output_schema: ./output.json  ← 出力スキーマへの参照
---
```

---

## 6. CI でのスキル自動検証

スキルをチームで共有する場合、CI でスキルの整合性を自動チェックします。

```bash
#!/bin/bash
# scripts/validate-skills.sh

SKILLS_DIR=".claude/skills"
ERRORS=0

for skill_dir in "$SKILLS_DIR"/*/; do
  skill_name=$(basename "$skill_dir")
  echo "検証中: $skill_name"

  # SKILL.md の存在確認
  if [ ! -f "$skill_dir/SKILL.md" ]; then
    echo "  ❌ SKILL.md が見つかりません"
    ERRORS=$((ERRORS + 1))
    continue
  fi

  # 必須フィールドの確認
  if ! grep -q "^name:" "$skill_dir/SKILL.md"; then
    echo "  ❌ name フィールドが見つかりません"
    ERRORS=$((ERRORS + 1))
  fi

  if ! grep -q "^description:" "$skill_dir/SKILL.md"; then
    echo "  ❌ description フィールドが見つかりません"
    ERRORS=$((ERRORS + 1))
  fi

  if ! grep -q "^version:" "$skill_dir/SKILL.md"; then
    echo "  ⚠️  version フィールドが推奨されます"
  fi

  echo "  ✅ OK"
done

if [ $ERRORS -gt 0 ]; then
  echo "エラー: $ERRORS 件のスキルに問題があります"
  exit 1
fi

echo "全スキルの検証が完了しました"
```

```yaml
# .github/workflows/validate-skills.yml
name: Validate Skills
on: [push, pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: bash scripts/validate-skills.sh
```

---

## 7. Skills のライフサイクル管理（2026年のベストプラクティス）

```
定義 → 入出力契約 → テスト → バージョン管理 → CI検証 → チーム共有 → 廃止
```

1. **定義**: `SKILL.md` を作成、`description` を具体的に書く
2. **入出力契約**: `input.md` / `output.json` でインターフェースを明確化
3. **バージョン付与**: `version` フィールドを追加（破壊的変更時はメジャーを上げる）
4. **テスト**: `context: fork` で副作用なく動作確認
5. **CI検証**: `validate-skills.sh` をCIで自動実行
6. **チーム共有**: リポジトリに含めてチーム全員がアクセス可能に
7. **廃止**: 使わなくなったスキルはディレクトリごと削除（削除前にCIで依存確認）

### description の書き方（重要）

Claude がスキルを自動選択するとき、`description` フィールドが判断基準になります。


```markdown
# ❌ 悪い例
description: "コードを処理する"

# ✅ 良い例
description: "TypeScript/JavaScriptファイルのコードレビューを行う。PR の差分や特定ファイルの品質・セキュリティ・パフォーマンスを評価して改善提案を返す"
```

---

## 参考リンク（追加情報）

- [codex: Skillsのモジュール化と配布可能化](../../../codex/topics/01-skills.md)

## 参考リンク

- [Claude Skills: A First Principles Deep Dive](https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/)
- [Claude Skills vs. MCP: A Technical Comparison](https://intuitionlabs.ai/articles/claude-skills-vs-mcp)
- [Building Production-Ready AI Agents with Claude Skills and MCP](https://medium.com/@jageenshukla/build-production-ai-agents-with-claude-skills-mcp-882d70ffe9ee)
