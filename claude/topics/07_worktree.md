# Git Worktree — 並列作業環境

**カテゴリ:** Git / Claude Code 統合
**基盤技術:** Git Worktree（Git 2.5+）+ Claude Code
**ステータス:** 本番利用可能（2026年現在。Claude Code 2.x で深く統合）

---

## 1. 概要

**Git Worktree** は、1つの Git リポジトリから複数の作業ディレクトリを作成する Git の機能です。Claude Code と組み合わせることで、**複数のエージェントが同一リポジトリを並列作業**できます。

### 通常の開発 vs Worktree 開発

```
通常（ブランチ切り替え方式）:
  リポジトリ（1つ）
  └─ 現在のブランチ: feature/auth
       → feature/payment に切り替えるには stash/commit が必要
       → 同時に複数ブランチで作業できない

Worktree 方式:
  リポジトリ（1つのGit履歴）
  ├─ .git/              ← Git の本体
  ├─ worktree-main/     ← main ブランチの作業ディレクトリ
  ├─ worktree-auth/     ← feature/auth ブランチの作業ディレクトリ
  └─ worktree-payment/  ← feature/payment ブランチの作業ディレクトリ
       → 3つのディレクトリで同時に並列作業可能！
```

### Claude Code での Worktree 活用

```
Claude Code セッション（オーケストレーター）
    │
    ├─ worktree-auth/     ← Claude エージェント1: 認証機能実装
    │       └─ 独自のCLAUDE.md / サブエージェント
    │
    ├─ worktree-payment/  ← Claude エージェント2: 決済機能実装
    │       └─ 独自のCLAUDE.md / サブエージェント
    │
    └─ worktree-tests/    ← Claude エージェント3: テスト作成
            └─ 独自のCLAUDE.md / サブエージェント
```

---

## 2. Git Worktree の基本コマンド

```bash
# Worktree の作成（新ブランチ）
git worktree add ../worktree-auth -b feature/auth

# Worktree の作成（既存ブランチ）
git worktree add ../worktree-main main

# Worktree の一覧
git worktree list

# Worktree の削除
git worktree remove ../worktree-auth

# 不要なWorktreeのクリーンアップ
git worktree prune
```

---

## 3. Claude Code での Worktree 統合

### Claude Code の `--worktree` オプション

```bash
# 新しい Worktree を作成して Claude Code を起動
claude --worktree feature/auth

# 既存の Worktree ディレクトリで起動
claude --worktree-path ../worktree-auth
```

### セッション終了時の動作

```
Worktree セッション終了
    ├─ 変更がない場合 → Worktree は自動削除
    └─ 変更がある場合 → 保持するか確認ダイアログ
```

---

## 4. ハンズオン：Worktree で機能を並列開発する

### Step 1: リポジトリを準備する

```bash
mkdir -p ~/worktree-demo
cd ~/worktree-demo
git init
npm init -y

# 基本構造を作成
mkdir -p src
cat > src/index.ts << 'EOF'
export * from './auth';
export * from './payment';
export * from './notifications';
EOF

cat > src/auth.ts << 'EOF'
// TODO: 認証機能を実装する
export {};
EOF

cat > src/payment.ts << 'EOF'
// TODO: 決済機能を実装する
export {};
EOF

cat > src/notifications.ts << 'EOF'
// TODO: 通知機能を実装する
export {};
EOF

git add -A
git commit -m "chore: 初期プロジェクト構造"
```

### Step 2: 各機能用の Worktree を作成する

```bash
# 認証機能用 Worktree
git worktree add ../worktree-auth -b feature/auth

# 決済機能用 Worktree
git worktree add ../worktree-payment -b feature/payment

# 通知機能用 Worktree
git worktree add ../worktree-notifications -b feature/notifications

# 確認
git worktree list
# ~/worktree-demo         abc1234 [main]
# ~/worktree-auth         abc1234 [feature/auth]
# ~/worktree-payment      abc1234 [feature/payment]
# ~/worktree-notifications abc1234 [feature/notifications]
```

### Step 3: 各 Worktree に専用の CLAUDE.md を配置する

```bash
# 認証機能チームへの指示
cat > ../worktree-auth/CLAUDE.md << 'EOF'
# 認証機能開発 Worktree

## 担当スコープ
このWorktreeでは src/auth.ts の実装のみ担当してください。
他のファイルは読み取り専用で参照可能。

## 実装要件
- JWT ベースの認証（access token: 15分、refresh token: 7日）
- bcryptjs でパスワードハッシュ化
- レート制限: 5回/分のログイン失敗で15分ロック

## 完了条件
- src/auth.ts の実装
- src/auth.test.ts のテスト（カバレッジ 90%以上）
- 完了後: git commit -m "feat: JWT認証機能を実装"
EOF

# 決済機能チームへの指示
cat > ../worktree-payment/CLAUDE.md << 'EOF'
# 決済機能開発 Worktree

## 担当スコープ
src/payment.ts の実装のみ担当してください。

## 実装要件
- Stripe SDK v15 を使用
- カード決済・PayPay・銀行振込に対応
- 冪等性キーを使った重複決済防止

## 重要
- 本番の Stripe キーを使わない（テストモードのみ）
- カード番号・CVVをログに出力しない

## 完了条件
- src/payment.ts の実装
- src/payment.test.ts のテスト
- 完了後: git commit -m "feat: 決済機能を実装"
EOF

# 通知機能チームへの指示
cat > ../worktree-notifications/CLAUDE.md << 'EOF'
# 通知機能開発 Worktree

## 担当スコープ
src/notifications.ts の実装のみ担当してください。

## 実装要件
- メール通知（Resend API）
- プッシュ通知（Web Push）
- Webhook による外部連携

## 完了条件
- src/notifications.ts の実装
- src/notifications.test.ts のテスト
- 完了後: git commit -m "feat: 通知機能を実装"
EOF
```

### Step 4: tmux で並列 Claude Code セッションを起動する

```bash
# tmux セッションを作成して3ペインに分割
tmux new-session -d -s dev -x 220 -y 50

# ウィンドウを3分割
tmux split-window -h -t dev
tmux split-window -v -t dev:0.1

# 各ペインで Claude Code を起動
tmux send-keys -t dev:0.0 "cd ~/worktree-auth && claude" C-m
tmux send-keys -t dev:0.1 "cd ~/worktree-payment && claude" C-m
tmux send-keys -t dev:0.2 "cd ~/worktree-notifications && claude" C-m

# tmux セッションに接続
tmux attach -t dev
```

各ペインで Claude Code が独立したコンテキストで並列作業します。

### Step 5: オーケストレーターで進捗を管理する

```python
# orchestrate_worktrees.py
import subprocess
import time
from pathlib import Path

WORKTREES = {
    "auth": Path("../worktree-auth"),
    "payment": Path("../worktree-payment"),
    "notifications": Path("../worktree-notifications"),
}

def check_branch_status(worktree_path: Path, branch: str) -> dict:
    """Worktree の作業状況を確認する"""
    result = subprocess.run(
        ["git", "-C", str(worktree_path), "log", "--oneline", "-5"],
        capture_output=True, text=True
    )
    return {
        "branch": branch,
        "recent_commits": result.stdout.strip().split("\n"),
        "path": str(worktree_path),
    }

def wait_for_all_complete():
    """全 Worktree の作業完了を待つ"""
    while True:
        all_done = True
        for name, path in WORKTREES.items():
            status = check_branch_status(path, f"feature/{name}")

            # コミットメッセージに "feat:" が含まれているか確認
            has_feature_commit = any(
                "feat:" in commit
                for commit in status["recent_commits"]
            )

            if not has_feature_commit:
                print(f"⏳ {name}: 作業中...")
                all_done = False
            else:
                print(f"✅ {name}: 完了")

        if all_done:
            break
        time.sleep(30)  # 30秒ごとにチェック

def merge_all_features():
    """全機能ブランチを main にマージ"""
    for name in WORKTREES.keys():
        branch = f"feature/{name}"
        subprocess.run(["git", "merge", "--no-ff", branch, "-m",
                        f"Merge {branch}: {name}機能を統合"])
        print(f"✅ {name} をマージ完了")

if __name__ == "__main__":
    print("🚀 並列開発の監視を開始します...")
    wait_for_all_complete()
    print("\n全機能の実装が完了しました。main ブランチにマージします...")
    merge_all_features()
    print("🎉 統合完了！")
```

### Step 6: パターンA（成功）とパターンB（失敗・破棄）

Gemini のドキュメントで示された「成功時マージ / 失敗時破棄」の2パターンを実践します。

#### パターンA：テスト成功時 → マージして統合

```bash
# main ブランチに戻る
cd ~/worktree-demo

# 各機能ブランチをマージ
git merge feature/auth --no-ff -m "feat: JWT認証機能を統合"
git merge feature/payment --no-ff -m "feat: 決済機能を統合"
git merge feature/notifications --no-ff -m "feat: 通知機能を統合"

# Worktree を削除
git worktree remove ../worktree-auth
git worktree remove ../worktree-payment
git worktree remove ../worktree-notifications

# 確認
git log --graph --oneline -10
```

#### パターンB：テスト失敗時 → 何もせず破棄

```bash
# テスト失敗・修正不能と判断した Worktree を破棄する
# ※ main には1行も変更が加わっていないため完全安全
cd ~/worktree-demo  # メインに戻る

git worktree remove ../worktree-payment --force  # 強制削除
git branch -D feature/payment                    # 失敗ブランチも削除

# 確認：mainは全く変わっていない
git status  # → nothing to commit, working tree clean
git log --oneline -3  # → メインのコミット履歴は変化なし

echo "✅ メイン環境は全く汚染されていません"
```

**このパターンBこそが Worktree 運用の最大の価値**です。エージェントが試行錯誤した全ての中間状態が破棄され、メイン環境は保護されます。

```
試行錯誤コスト:
  通常の git checkout → stash/commit が必要 → コンテキスト混濁
  Worktree → 物理的に分離 → 完全破棄可能
```

---

## 5. Worktree のベストプラクティス（2026年）

### ネーミング規則

```bash
# 推奨: 機能名を含む一貫したネーミング
git worktree add .claude/worktrees/feature-auth -b feature/auth

# プロジェクトルートの外に置くと混乱しない
~/my-project/           ← メイン
~/my-project-auth/      ← feature/auth
~/my-project-payment/   ← feature/payment
```

### 独立した Node.js / Python 環境

```bash
# Worktree ごとに独立した依存関係
cd ~/worktree-auth
npm install  # worktree-auth/node_modules/ が作られる（メインとは独立）
```

### Worktree 間での共有設定

```bash
# グローバル CLAUDE.md（全Worktreeで共有）
~/.claude/CLAUDE.md  ← 全エージェントに適用される共通設定

# リポジトリルートの CLAUDE.md（全Worktreeに継承される）
~/my-project/CLAUDE.md  ← 各Worktreeの CLAUDE.md に追加される
```

---

## 6. Worktree + Subagent の組み合わせパターン

```
オーケストレーター（メインディレクトリ）
    │
    ├─ feature/auth Worktree
    │       ├─ Implement Subagent（コード実装）
    │       ├─ Test Subagent（テスト作成）
    │       └─ Review Subagent（レビュー）
    │
    └─ feature/payment Worktree
            ├─ Implement Subagent
            ├─ Test Subagent
            └─ Security Audit Subagent（決済には特別に追加）
```

各 Worktree が独立した Subagent チームを持つことで、完全並列開発が実現します。

---

## 参考リンク

- [Claude Code Worktree: Complete Guide](https://supatest.ai/blog/claude-code-worktree-the-complete-developer-guide)
- [24 Claude Code Tips](https://dev.to/oikon/24-claude-code-tips-claudecodeadventcalendar-52b5)
- [Context Management with Subagents in Claude Code](https://www.richsnapp.com/article/2025/10-05-context-management-with-subagents-in-claude-code)
