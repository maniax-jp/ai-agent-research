# 05. AgentTeam設計の定式化

## 技術動向
`AgentTeam` は複数Agentの同時起動ではなく、ロール間契約を持つ`協調システム`として設計されます。鍵は「責務境界」「受け渡し形式」「完了定義」です。

## 設計ポイント
- ロールは4つ程度に固定（PM/Coder/Reviewer/Release）
- 受け渡しを構造化（JSON or Markdownテンプレート）
- 完了条件をロールごとに定義
- トポロジーを先に決める（階層型 / パイプライン型）

## ハンズオン（4ロールの連携設計）
1. チーム仕様ディレクトリ作成
```powershell
New-Item -ItemType Directory -Force agent-team/contracts | Out-Null
```

2. ロール定義作成
```markdown
# PM
- 要件分解
- 優先度付け

# Coder
- 実装
- 変更理由記録

# Reviewer
- 回帰/セキュリティ/テスト確認

# Release
- マージ判定
- リリースノート生成
```

3. 受け渡し契約（例）
```json
{
  "handoff": {
    "from": "PM",
    "to": "Coder",
    "fields": ["task_id", "requirements", "acceptance_criteria"]
  }
}
```

4. 完了条件定義
```markdown
- PM Done: 受入基準が明文化されている
- Coder Done: テストが追加/更新されている
- Reviewer Done: Critical/Highが0件
- Release Done: mainへ安全に統合済み
```

5. 実行ログの器を作る
```powershell
New-Item -ItemType Directory -Force agent-team/runs/T-001 | Out-Null
```

6. 1サイクル実行し、各ロールの出力を保存
- `agent-team/runs/T-001/pm.md`
- `agent-team/runs/T-001/coder.md`
- `agent-team/runs/T-001/reviewer.md`
- `agent-team/runs/T-001/release.md`

7. リリース判定を自動化（簡易）
```powershell
if (Select-String -Path agent-team/runs/T-001/reviewer.md -Pattern "Critical|High") {
  throw "Release blocked by review gate"
}
```

## 期待結果
- 役割重複を減らし、手戻り原因を局所化できる
- 合意可能な「Done」を技術的に固定できる

## 失敗しやすい点
- ロール定義が曖昧で実質単一Agent運用になる
- 受け渡し形式が自由記述で、後段が読み取れない
- リリース判定が人依存で再現性がない

## 次の強化
- 契約ファイルをスキーマ検証
- ロール別KPI（差戻し率、レビュー時間）を可視化
