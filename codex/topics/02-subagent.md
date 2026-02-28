# 02. subagent前提の階層型オーケストレーション

## 技術動向
`subagent` は、単一Agentに全責務を持たせる方式の限界を解消するための標準パターンです。2026年以降は、`Planner -> Worker -> Reviewer` の3層構造が実装テンプレート化されています。

## 設計ポイント
- 各subagentの責務を1つに限定する
- コンテキスト共有は最小限（必要情報のみ）
- 失敗時の再実行単位をsubagent単位で定義する
- 各subagentの成果物をファイルに残す（監査可能性）

## ハンズオン（3層構成を作る）
前提: `AGENTS.md` があるリポジトリ

1. subagent定義ディレクトリを作る
```powershell
New-Item -ItemType Directory -Force .agents | Out-Null
```

2. `planner.md` を作る
```markdown
# Planner
- 入力要件をタスクへ分解
- Workerに渡すチェックリストを出力
```

3. `worker.md` を作る
```markdown
# Worker
- Plannerのチェックリストに従い実装
- 変更ファイルと理由を記録
```

4. `reviewer.md` を作る
```markdown
# Reviewer
- 仕様一致、回帰、テスト不足を検査
- Fail時はWorkerへ差し戻し理由を返却
```

5. 受け渡しフォーマットを決める（例: JSON）
```json
{
  "task_id": "T-001",
  "inputs": ["spec.md"],
  "acceptance_criteria": ["tests_pass", "no_security_regression"]
}
```

6. 成果物フォルダを用意
```powershell
New-Item -ItemType Directory -Force .runs/T-001 | Out-Null
```

7. 1タスクを通しで実行し、成果物を保存
- Planner: `.runs/T-001/plan.md`
- Worker: `.runs/T-001/changes.md`
- Reviewer: `.runs/T-001/review.md`

8. 判定ゲート
```powershell
if (Select-String -Path .runs/T-001/review.md -Pattern "CRITICAL") {
  Write-Error "Release blocked"
}
```

## 期待結果
- 不具合時にどの層で失敗したか特定できる
- 局所リトライで再実行コストを下げられる

## 失敗しやすい点
- Plannerが実装詳細まで書きすぎてWorker責務と衝突
- Reviewer基準が曖昧で、同じ変更の合否が毎回変わる
- 成果物が会話ログにしか残らず、再現不能になる

## 次の強化
- subagentごとに利用可能ツールを明示
- 受け渡しJSONをスキーマ検証
