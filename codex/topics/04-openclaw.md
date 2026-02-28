# 04. OpenClaw系ツールチェーンの活用

## 技術動向
OpenClaw系の価値は、モデル比較ではなく`実タスク再現性`の評価です。CLI実行、ファイル編集、テスト実行まで含むE2E評価を継続できる点が重要です。

## 設計ポイント
- 評価対象を「プロンプト品質」ではなく「タスク成功」に置く
- 同一タスクを複数回走らせ、安定性を測る
- 実行ログを保存して失敗原因を追跡可能にする
- コマンド実行は `allowlist` と `timeout` で制限する

## ハンズオン（E2E評価の最小パイプライン）
前提: `tasks/` と `logs/` ディレクトリを使用

1. ディレクトリ作成
```powershell
New-Item -ItemType Directory -Force tasks,logs | Out-Null
```

2. タスク定義を作る（例）
```json
{
  "id": "fix-001",
  "goal": "failing testを1件修正",
  "success_criteria": ["all_tests_pass", "no_new_lint_error"]
}
```

3. 評価ランナーを作る（簡易例）
```powershell
@'
param([string]$TaskFile="tasks/fix-001.json")
$ts = Get-Date -Format "yyyyMMdd-HHmmss"
$log = "logs/run-$ts.log"
"Task: $TaskFile" | Tee-Object -FilePath $log
npm test 2>&1 | Tee-Object -FilePath $log -Append
'@ | Set-Content scripts/eval-run.ps1 -Encoding UTF8
```

4. コマンドガードを追加（簡易）
```powershell
@'
param([string]$Command)
$allow = @("npm test","npm run lint","git status")
if ($allow -notcontains $Command) { throw "blocked command: $Command" }
Invoke-Expression $Command
'@ | Set-Content scripts/guarded-exec.ps1 -Encoding UTF8
```

5. 実行
```powershell
powershell -File scripts/eval-run.ps1
```

6. 結果確認
```powershell
Get-ChildItem logs
Get-Content (Get-ChildItem logs | Sort-Object LastWriteTime -Desc | Select-Object -First 1).FullName
```

## 期待結果
- 各試行の成功/失敗ログが時系列で残る
- 同一タスクでの成功率を算出できる

## 失敗しやすい点
- 失敗ログに `task_id` がなく、後で分析できない
- 無制限コマンド実行で誤操作リスクが高まる
- 成功率だけ見て、平均再試行回数を見落とす

## 次の強化
- タスクごとの再試行回数と平均処理時間を記録
- 失敗パターンを分類し、プロンプトではなく工程で改善
