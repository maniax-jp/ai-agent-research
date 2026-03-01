# Git Worktree - 並列エージェント実行のための隔離環境

**作成者:** Qwen
**日付:** 2026 年 3 月 1 日

---

## 目次

1. [概要](#概要)
2. [Git Worktree の基礎](#git-worktree の基礎)
3. [ハンズオン：Worktree を使った並列エージェント](#ハンズオンworktree を使った並列エージェント)
4. [進階：高度な Worktree パターン](#進階高度な-worktree パターン)
5. [ベストプラクティス](#ベストプラクティス)

---

## 概要

**Git Worktree**は、Git リポジトリから複数の独立したワークスペースを作成する機能です。2026 年現在、AI エージェントの並列実行における標準的な隔離メカニズムとなっています。

### 主要なメリット

- **完全な隔離:** 各エージェントが独立したワークスペース
- **ゼロ競合:** ファイルの競合が発生しない
- **共通リポジトリ:** 同じ Git リポジトリを共有
- **効率的:** `.git` ディレクトリを共有

---

## Git Worktree の基礎

### 基本構造

```
my-project/                 # メインワークツリー
├── .git/                   # Git メタデータ（共有）
├── src/
└── package.json

my-project-worktrees/       # Worktrees 親ディレクトリ
├── feature-auth/           # Worktree 1
│   ├── src/               # 独立したファイル
│   └── package.json
├── feature-profile/        # Worktree 2
│   ├── src/
│   └── package.json
└── feature-settings/       # Worktree 3
    ├── src/
    └── package.json
```

### 内部構造

```
my-project/.git/worktrees/  # Worktree メタデータ
├── feature-auth/
│   ├── HEAD
│   ├── environment
│   ├── indices
│   └── locks
├── feature-profile/
└── feature-settings/
```

---

## ハンズオン：Worktree を使った並列エージェント

### ステップ 1: 環境準備

```bash
# プロジェクトディレクトリを作成
mkdir worktree-demo
cd worktree-demo

# Git リポジトリを初期化
git init
git commit --allow-empty -m "Initial commit"

# Worktrees ディレクトリを作成
mkdir -p .claude/worktrees
```

### ステップ 2: 基本的な Worktree の作成

```bash
# Worktree の作成
git worktree add .claude/worktrees/feature-auth -b feature-auth

# Worktree の確認
git worktree list

# 出力例:
# /home/user/worktree-demo              main
# /home/user/worktree-demo/.claude/worktrees/feature-auth  feature-auth
```

### ステップ 3: Claude Code で Worktree を使用

```bash
# Worktree で Claude Code を起動
claude --worktree feature-auth

# または、短縮形
claude -w feature-auth
```

Claude は自動的に：
1. 新しい Worktree を作成
2. 新しいブランチを作成
3. その Worktree でセッションを開始

### ステップ 4: 複数のエージェントを並列実行

```bash
# ターミナル 1: 認証機能
claude -w feature-auth "認証機能を実装してください"

# ターミナル 2: プロフィール機能
claude -w feature-profile "プロフィール機能を実装してください"

# ターミナル 3: 設定機能
claude -w feature-settings "設定機能を実装してください"
```

各エージェントは独立して作業：

```
┌─────────────────────────────────────────────────────────┐
│ Terminal 1: feature-auth worktree                       │
│ Claude: 認証機能を実装中...                               │
│ Files: src/auth/login.ts, src/auth/register.ts          │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ Terminal 2: feature-profile worktree                    │
│ Claude: プロフィール機能を実装中...                         │
│ Files: src/profile/view.ts, src/profile/edit.ts         │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ Terminal 3: feature-settings worktree                   │
│ Claude: 設定機能を実装中...                               │
│ Files: src/settings/general.ts, src/settings/privacy.ts │
└─────────────────────────────────────────────────────────┘
```

### ステップ 5: 完了後のマージ

各 Worktree で完了後：

```bash
# メインブランチに戻る
git checkout main

# 各ブランチをマージ
git merge feature-auth
git merge feature-profile
git merge feature-settings

# Worktree を削除
git worktree remove .claude/worktrees/feature-auth
git worktree remove .claude/worktrees/feature-profile
git worktree remove .claude/worktrees/feature-settings
```

---

## 進階：高度な Worktree パターン

### ステップ 6: 自動 Worktree 管理スクリプト

`.claude/worktree-manager.sh` を作成：

```bash
#!/bin/bash

# Worktree Manager
# 複数の Worktree を管理するスクリプト

WORKTREE_DIR=".claude/worktrees"

# Worktree 作成
create_worktree() {
    local name=$1
    local branch="feature/${name}"

    echo "Worktree '${name}' を作成中..."

    # ディレクトリ作成
    mkdir -p "$WORKTREE_DIR"

    # Worktree 作成
    git worktree add "$WORKTREE_DIR/$name" -b "$branch" 2>/dev/null

    if [ $? -eq 0 ]; then
        echo "✓ Worktree 作成完了：$WORKTREE_DIR/$name"
        echo "  ブランチ：$branch"
    else
        echo "✗ Worktree 作成失敗：$name"
        return 1
    fi
}

# Worktree 削除
remove_worktree() {
    local name=$1

    echo "Worktree '${name}' を削除中..."
    git worktree remove "$WORKTREE_DIR/$name" 2>/dev/null

    if [ $? -eq 0 ]; then
        echo "✓ Worktree 削除完了：$name"
    else
        echo "✗ Worktree 削除失敗：$name"
        return 1
    fi
}

# Worktree リスト
list_worktrees() {
    echo "利用中の Worktrees:"
    git worktree list | grep "$WORKTREE_DIR"
}

# 使用法
case "$1" in
    create)
        create_worktree "$2"
        ;;
    remove)
        remove_worktree "$2"
        ;;
    list)
        list_worktrees
        ;;
    *)
        echo "使用法：$0 {create|remove|list} [name]"
        exit 1
        ;;
esac
```

使用例：

```bash
# Worktree 作成
./worktree-manager.sh create auth

# Worktree リスト
./worktree-manager.sh list

# Worktree 削除
./worktree-manager.sh remove auth
```

### ステップ 7: 並列エージェントオーケストレーター

`.claude/parallel-agent.sh` を作成：

```bash
#!/bin/bash

# Parallel Agent Orchestrator
# 複数のエージェントを並列で実行

WORKTREE_DIR=".claude/worktrees"
LOG_DIR=".claude/logs"

# 設定
declare -A TASKS=(
    ["auth"]="認証機能を実装：ログイン、ログアウト、パスワードリセット"
    ["profile"]="プロフィール機能を実装：表示、編集、アバターアップロード"
    ["settings"]="設定機能を実装：一般設定、プライバシー、通知"
)

# Worktree 作成とエージェント起動
start_agent() {
    local name=$1
    local task=$2

    echo "Agent '${name}' を起動中..."

    # Worktree 作成
    git worktree add "$WORKTREE_DIR/$name" -b "feature/$name" 2>/dev/null

    # ログファイル作成
    mkdir -p "$LOG_DIR"
    local log_file="$LOG_DIR/${name}.log"

    # エージェント起動（バックグラウンド）
    cd "$WORKTREE_DIR/$name"
    claude "$task" > "$log_file" 2>&1 &

    echo $! > "$LOG_DIR/${name}.pid"
    echo "✓ Agent '${name}' 起動完了 (PID: $!)"
}

# すべてのエージェント起動
start_all_agents() {
    echo "すべてのエージェントを起動中..."

    for name in "${!TASKS[@]}"; do
        start_agent "$name" "${TASKS[$name]}"
    done

    echo ""
    echo "すべてのエージェントが起動しました"
    echo "ログの確認：tail -f $LOG_DIR/*.log"
}

# エージェント待機
wait_for_agents() {
    echo "エージェントの完了を待機中..."

    for pid_file in "$LOG_DIR"/*.pid; do
        if [ -f "$pid_file" ]; then
            pid=$(cat "$pid_file")
            wait $pid
            echo "Agent (PID: $pid) が完了しました"
        fi
    done
}

# 結果マージ
merge_results() {
    echo "結果をマージ中..."

    git checkout main

    for name in "${!TASKS[@]}"; do
        branch="feature/$name"
        git merge "$branch" --no-edit 2>/dev/null

        if [ $? -eq 0 ]; then
            echo "✓ ブランチ '${branch}' マージ完了"
        else
            echo "✗ ブランチ '${branch}' マージ失敗"
        fi
    done
}

# Worktree クリーンアップ
cleanup_worktrees() {
    echo "Worktree をクリーンアップ中..."

    for name in "${!TASKS[@]}"; do
        git worktree remove "$WORKTREE_DIR/$name" 2>/dev/null
        echo "✓ Worktree '${name}' 削除完了"
    done
}

# 使用法
case "$1" in
    start)
        start_all_agents
        ;;
    wait)
        wait_for_agents
        ;;
    merge)
        merge_results
        ;;
    cleanup)
        cleanup_worktrees
        ;;
    run)
        start_all_agents
        wait_for_agents
        merge_results
        cleanup_worktrees
        ;;
    *)
        echo "使用法：$0 {start|wait|merge|cleanup|run}"
        exit 1
        ;;
esac
```

使用例：

```bash
# すべて実行
./parallel-agent.sh run

# 個別に実行
./parallel-agent.sh start
./parallel-agent.sh wait
./parallel-agent.sh merge
./parallel-agent.sh cleanup
```

### ステップ 8: Claude Code の Worktree 設定

`.claude/settings.json` を作成：

```json
{
  "worktree": {
    "enabled": true,
    "autoCreate": true,
    "autoCleanup": true,
    "directory": ".claude/worktrees",
    "branchPrefix": "feature/"
  },
  "subagents": {
    "isolation": "worktree",
    "autoMerge": false
  }
}
```

### ステップ 9: Parallel Code の使用

[Parallel Code](https://github.com/johannesjo/parallel-code) は、GUI で複数のエージェントを管理：

```bash
# Parallel Code をインストール
npm install -g parallel-code

# 起動
parallel-code
```

GUI で：
1. タスクを作成
2. エージェントを選択（Claude、Codex、Gemini）
3. 並列実行
4. 結果を確認
5. 1 クリックでマージ

---

## 成功時マージ / 失敗時破棄パターン

claude のドキュメントを参照した、Worktree 運用の最大の価値となるパターンです。

### パターン A：テスト成功時 → マージして統合

```bash
# main ブランチに戻る
cd ~/worktree-demo

# 各機能ブランチをマージ
git merge feature/auth --no-ff -m "feat: JWT 認証機能を統合"
git merge feature/payment --no-ff -m "feat: 決済機能を統合"
git merge feature/notifications --no-ff -m "feat: 通知機能を統合"

# Worktree を削除
git worktree remove ../worktree-auth
git worktree remove ../worktree-payment
git worktree remove ../worktree-notifications

# 確認
git log --graph --oneline -10
```

### パターン B：テスト失敗時 → 何もせず破棄

```bash
# テスト失敗・修正不能と判断した Worktree を破棄する
# ※ main には 1 行も変更が加わっていないため完全安全
cd ~/worktree-demo  # メインに戻る

git worktree remove ../worktree-payment --force  # 強制削除
git branch -D feature/payment                    # 失敗ブランチも削除

# 確認：main は全く変わっていない
git status  # → nothing to commit, working tree clean
git log --oneline -3  # → メインのコミット履歴は変化なし

echo "✅ メイン環境は全く汚染されていません"
```

**このパターン B こそが Worktree 運用の最大の価値**です。エージェントが試行錯誤した全ての中間状態が破棄され、メイン環境は保護されます。

```
試行錯誤コスト:
  通常の git checkout → stash/commit が必要 → コンテキスト混濁
  Worktree → 物理的に分離 → 完全破棄可能
```

---

## ベストプラクティス

### 1. Worktree の命名規則

一貫した命名規則を使用：

```bash
# 良い例
git worktree add .claude/worktrees/feat-auth -b feat/auth
git worktree add .claude/worktrees/fix-login-bug -b fix/login-bug
git worktree add .claude/worktrees/refactor-api -b refactor/api

# 悪い例
git worktree add .claude/worktrees/new -b new
git worktree add .claude/worktrees/feature1 -b feature1
git worktree add .claude/worktrees/test -b test
```

### 2. Worktree の整理

Worktree を整理して管理：

```bash
# Worktrees 親ディレクトリ
mkdir -p .claude/worktrees

# カテゴリ別ディレクトリ
mkdir -p .claude/worktrees/features
mkdir -p .claude/worktrees/bugs
mkdir -p .claude/worktrees/experiments

# Worktree 作成
git worktree add .claude/worktrees/features/auth -b feat/auth
git worktree add .claude/worktrees/bugs/login -b fix/login
```

### 3. 定期的なクリーンアップ

使用済み Worktree を削除：

```bash
# 古い Worktree を確認
git worktree list

# 使用済み Worktree を削除
git worktree remove .claude/worktrees/completed-feature

# 自動クリーンアップスクリプト
find .claude/worktrees -type d -mtime +7 -exec git worktree remove {} \;
```

### 4. 依存関係の管理

`node_modules` をシンボリックリンク：

```bash
# メインディレクトリでインストール
npm install

# Worktree でシンボリックリンク
ln -s ../../node_modules .claude/worktrees/auth/node_modules
```

または、`.gitignore` で除外：

```gitignore
# .claude/worktrees/*/node_modules
```

### 5. 衝突の回避

Worktree 間で衝突を回避：

```bash
# 各 Worktree で異なるポートを使用
# feature-auth/.env
PORT=3001

# feature-profile/.env
PORT=3002

# feature-settings/.env
PORT=3003
```

### 6. ログの管理

各 Worktree のログを管理：

```bash
# ログディレクトリ
mkdir -p .claude/logs

# 各 Worktree のログ
claude -w auth > .claude/logs/auth.log 2>&1
claude -w profile > .claude/logs/profile.log 2>&1
```

---

## トラブルシューティング

### 問題 1: Worktree 作成失敗

**エラー:**
```
fatal: 'path' is not a directory or one of its parents
```

**解決:**
```bash
# ディレクトリを作成
mkdir -p .claude/worktrees

# 再度作成
git worktree add .claude/worktrees/feature -b feature
```

### 問題 2: 同じブランチの Worktree

**エラー:**
```
fatal: cannot lock ref 'refs/heads/main': is already checked out
```

**解決:**
```bash
# 各 Worktree は異なるブランチが必要
git worktree add .claude/worktrees/feature -b feature

# 強制（推奨しない）
git worktree add -f .claude/worktrees/another-main -b main
```

### 問題 3: Worktree の削除失敗

**エラー:**
```
fatal: The path '.claude/worktrees/feature' is in a different repository
```

**解決:**
```bash
# Worktree のパスを確認
git worktree list

# 正しいパスで削除
git worktree remove /full/path/to/.claude/worktrees/feature
```

### 問題 4: マージ競合

**解決:**
```bash
# 競合を解決
cd .claude/worktrees/feature
git checkout main
git merge feature
# 競合ファイルを編集
git add .
git commit -m "Resolve merge conflicts"
```

---

## まとめ

Git Worktree は、AI エージェントの並列実行のための強力な隔離メカニズムです。適切な使用により、複数のエージェントを効率的に管理し、生産性を大幅に向上できます。

### 次のステップ

1. [Agent Teams](./05-agent-teams.md) - Worktree との組み合わせ
2. [Context Fork](./06-context-fork.md) - コンテキストフォークとの比較
3. [Subagent](./02-subagent.md) - サブエージェントとの統合

---

*関連リソース:*
- [Git Worktree Documentation](https://git-scm.com/docs/git-worktree)
- [Claude Code Worktree Support](https://code.claude.com/docs)
- [Parallel Code](https://github.com/johannesjo/parallel-code)
- [Git Worktrees for AI Coding](https://dev.to/mashrulhaque/git-worktrees-for-ai-coding)