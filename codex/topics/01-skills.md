# 01. Skillsのモジュール化と配布可能化

## 技術動向
2026年以降、`Skills` は「プロンプト集」ではなく、`実行可能な業務ユニット`として扱う設計が主流です。最小構成は `SKILL.md` に加えて、再利用コード（`scripts/`）と雛形（`templates/`）を同梱する形です。

## SkillsとMCPの違い
- `Skills`: タスクの進め方を定義する（手順・判断・出力形式）
- `MCP`: 外部ツールへの接続方法を標準化する（DB/API/ファイル）
- 実務では `SkillがMCPツールを呼ぶ` 形で組み合わせる

## 設計ポイント
- スキル入出力を明文化する（例: `input.md`, `output.json`）
- 副作用を限定する（編集対象パス、実行コマンドの許可範囲）
- 依存を局所化する（1スキルが他スキルに過度依存しない）

## ハンズオン（レビューSkillを作る）
前提: Git管理された任意リポジトリ直下で作業

1. フォルダ作成
```powershell
New-Item -ItemType Directory -Force .codex/skills/code-review/scripts | Out-Null
New-Item -ItemType Directory -Force .codex/skills/code-review/templates | Out-Null
```

2. `SKILL.md` 作成
```markdown
# code-review

## Purpose
差分に対してバグ・リスク・テスト欠落を指摘する。

## Scope
- Read: repo全体
- Write: findings.md のみ

## Inputs
- target_paths: レビュー対象のファイル/ディレクトリ

## Outputs
- findings.md: 重大度順の指摘一覧

## Workflow
1. 変更点を取得
2. 振る舞い回帰の有無を評価
3. テスト不足を列挙
```

3. スクリプト実装（例）
```powershell
@'
param([string]$Path=".")
git -C $Path diff --name-only HEAD~1..HEAD
'@ | Set-Content .codex/skills/code-review/scripts/list_changed.ps1 -Encoding UTF8
```

4. テンプレート作成（判定を固定）
```markdown
# Findings

## Critical
- [file:line] 内容
- 根拠
- 修正案

## High
- [file:line] 内容
- 根拠
- 修正案

## Medium
- [file:line] 内容
- 根拠
```

5. 実行確認
```powershell
powershell -File .codex/skills/code-review/scripts/list_changed.ps1
```

6. 出力ファイルを作る
```powershell
Copy-Item .codex/skills/code-review/templates/findings.md findings.md -Force
```

## 期待結果
- 入出力契約を持つスキルを再利用可能な形で保管できる
- スキル単体テスト（最低限: 実行可否）を回せる

## 失敗しやすい点
- `SKILL.md` が抽象的で毎回出力が揺れる
- 出力形式が固定されず、AgentTeamの後段でパース不能になる
- 書き込み範囲を制限しておらず副作用が広がる

## 次の強化
- `version` を `SKILL.md` に追加
- スキル登録時にlint/検証をCIで自動実行
