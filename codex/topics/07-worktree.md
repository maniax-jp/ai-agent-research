# 07. worktreeベース開発の標準化

## 技術動向
`git worktree` は、1つのリポジトリで複数ブランチ作業を物理分離する実践手段です。subagentやAgentTeamの並列実装で衝突を減らすため、2026年以降の運用基盤になっています。

## 設計ポイント
- 役割ごとにworktreeを分ける（例: `wt-coder`, `wt-review`)
- 各worktreeで独立テストを実行
- 統合前に差分と契約準拠を確認
- 失敗時に `worktree remove --force` で即時破棄できる設計にする

## ハンズオン（2並列worktree）
前提: Gitリポジトリ直下

1. ブランチ作成
```powershell
git branch feat/a
git branch feat/b
```

2. worktree追加
```powershell
git worktree add ..\\wt-a feat/a
git worktree add ..\\wt-b feat/b
git worktree list
```

3. 並列作業
```powershell
Set-Location ..\\wt-a
git status

Set-Location ..\\wt-b
git status
```

4. それぞれで実装・テスト
```powershell
npm test
```

5. メインリポジトリへ戻って統合
```powershell
Set-Location ..\\ai-agent-research
git checkout main
git merge feat/a
git merge feat/b
```

6. 後片付け（成功時）
```powershell
git worktree remove ..\\wt-a
git worktree remove ..\\wt-b
```

7. 後片付け（失敗時）
```powershell
git worktree remove ..\\wt-a --force
git branch -D feat/a
git worktree prune
```

## 期待結果
- 作業ディレクトリ汚染を防ぎつつ並列実装できる
- マージ時の衝突を早期発見できる

## 失敗しやすい点
- worktreeを同一階層に乱立させ、どれが生存中か不明になる
- 未コミットのまま強制削除して作業を失う
- mainに直接変更して並列開発の利点を失う

## 次の強化
- worktreeごとにCI jobを紐付ける
- `context fork` と連動してA/B案を自動比較
