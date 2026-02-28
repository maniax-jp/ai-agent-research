# 03. AGENTS.mdの実行契約化

## 技術動向
`AGENTS.md` は、自然言語メモではなく、エージェントが従う`実行契約`として扱われます。特に「何をしてよいか」「何をしてはいけないか」を明確化する用途が中心です。

## 設計ポイント
- 許可コマンドと禁止操作を分離記述する
- 編集可能範囲（ディレクトリ）を明示する
- レビュー時の合格基準を定義する
- サブディレクトリ単位で追加ルールを分離する

## ハンズオン（AGENTS.mdを契約化する）
1. ルート `AGENTS.md` を作成
```powershell
@'
# AGENTS.md

## Allowed
- rg
- git status
- npm test

## Disallowed
- git reset --hard
- rm -rf

## Edit Scope
- src/**
- tests/**

## Review Gate
- 破壊的変更なし
- 新規ロジックにテスト追加
'@ | Set-Content AGENTS.md -Encoding UTF8
```

2. サブディレクトリ用ルールを作成（例: 決済）
```powershell
New-Item -ItemType Directory -Force src/payment | Out-Null
@'
# AGENTS.md

## Extra Rules
- 決済モジュールでは外部APIのモックテスト必須
- 金額計算は decimal を使用する
'@ | Set-Content src/payment/AGENTS.md -Encoding UTF8
```

3. CIチェック追加（最小例）
```powershell
@'
if (!(Test-Path AGENTS.md)) { throw "AGENTS.md missing" }
$txt = Get-Content AGENTS.md -Raw
if ($txt -notmatch "## Allowed") { throw "Allowed section missing" }
if ($txt -notmatch "## Disallowed") { throw "Disallowed section missing" }
if (Test-Path "src/payment/AGENTS.md") {
  $child = Get-Content "src/payment/AGENTS.md" -Raw
  if ($child -notmatch "## Extra Rules") { throw "payment AGENTS.md missing Extra Rules" }
}
'@ | Set-Content scripts/validate-agents.ps1 -Encoding UTF8
```

4. 手動検証
```powershell
powershell -File scripts/validate-agents.ps1
```

## 期待結果
- エージェント実行時の逸脱を早期検知できる
- チーム間で「安全な自動化範囲」を共有できる

## 失敗しやすい点
- ルート規約が抽象的すぎて運用解釈が分かれる
- サブディレクトリ規約を置かず、重要領域でガードが弱い
- ルール更新後にCI検証が追従せず形骸化する

## 次の強化
- PR時に `validate-agents.ps1` を必須化
- セクションをYAML化し機械可読性を高める
