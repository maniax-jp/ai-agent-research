# OpenClaw - オープンソース自律 AI エージェント

**作成者:** Qwen
**日付:** 2026 年 3 月 9 日

---

## 目次

1. [概要](#概要)
2. [OpenClaw のアーキテクチャ](#openclaw のアーキテクチャ)
3. [ハンズオン：OpenClaw のインストールと設定](#ハンズオンopenclaw のインストールと設定)
4. [進階：カスタムスキルの作成](#進階カスタムスキルの作成)
5. [高度な使用法](#高度な使用法)

---

## 概要

**OpenClaw**（元 Clawdbot、Moltbot）は、ローカルで動作し、自律的にタスクを実行するオープンソース AI エージェントです。2025 年 11 月にオーストリアの開発者 Peter Steinberger によってリリースされ、短期間で 175,000 以上の GitHub スターを獲得しました。

### 主要な特徴

- **完全ローカル:** データがローカルに保存
- **自律実行:** メール管理、カレンダー、コードデプロイなど
- **24/7 動作:** 常時稼働可能
- **マルチプラットフォーム:** Telegram、WhatsApp、Discord など 12+ のメッセージングプラットフォーム対応
- **スキルシステム:** ClawHub によるコミュニティスキル

---

## OpenClaw のアーキテクチャ

### 基本構造

```
OpenClaw
├── Gateway (ローカルゲートウェイ)
│   ├── API 接続 (Claude, OpenAI, Ollama)
│   └── メッセージング統合
├── Agent Loop (エージェントループ)
│   ├── 入力処理
│   ├── 推論
│   ├── ツール実行
│   └── 出力生成
├── Memory (メモリシステム)
│   ├── SOUL.md (人格・設定)
│   ├── MEMORY.md (長期記憶)
│   ├── AGENTS.md (エージェント設定)
│   └── HEARTBEAT.md (プロアクティブタスク)
└── Skills (スキルシステム)
    └── ClawHub (コミュニティスキルリポジトリ)
```

### ファイル構造

```
~/.openclaw/
├── workspace/
│   ├── AGENTS.md      # エージェント設定
│   ├── SOUL.md        # 人格・設定・トーン
│   ├── MEMORY.md      # 長期記憶
│   ├── HEARTBEAT.md   # プロアクティブタスク
│   └── memory/
│       └── 2026-03-01.md  # 日々のエフェメラログ
├── config.yaml        # 設定ファイル
├── logs/              # ログファイル
└── skills/            # インストールされたスキル
```

---

## ハンズオン：OpenClaw のインストールと設定

### ステップ 1: 前提条件の確認

```bash
# OS の確認
# macOS、Linux、または Windows (WSL2)

# Node.js のインストール確認
node --version  # 18+ が必要

# Git のインストール確認
git --version
```

### ステップ 2: OpenClaw のインストール

```bash
# OpenClaw をインストール
npm install -g openclaw

# または、GitHub からクローン
git clone https://github.com/PeterSteinberger/openclaw.git
cd openclaw
npm install
```

### ステップ 3: オンボーディングウィザードの実行

```bash
# オンボーディングウィザードを起動
openclaw onboard
```

ウィザードは以下の設定をガイド：

1. **ゲートウェイ設定**
   - API プロバイダーの選択（Claude、OpenAI、Ollama など）
   - API キーの入力

2. **ワークスペース設定**
   - ワークスペースディレクトリの選択
   - 初期設定

3. **メッセージングチャンネル**
   - Telegram、WhatsApp、Discord の選択
   - 各プラットフォームの設定

4. **初期スキルのインストール**
   - 基本スキルの選択
   - コミュニティスキルのインストール

### ステップ 4: API キーの設定

`.openclaw/config.yaml` を編集：

```yaml
# API 設定
api:
  provider: anthropic  # または openai, ollama
  api_key: "your-api-key-here"

# Claude の場合
anthropic:
  api_key: "sk-ant-..."
  model: "claude-3-5-sonnet"

# OpenAI の場合
openai:
  api_key: "sk-..."
  model: "gpt-4o"

# Ollama（ローカル）の場合
ollama:
  base_url: "http://localhost:11434"
  model: "llama3"
```

### ステップ 5: SOUL.md の作成

`~/.openclaw/workspace/SOUL.md` を作成：

```markdown
# My OpenClaw Soul

## 人格

私は助け役で、効率的で、詳細に注意を払う AI アシスタントです。

## トーン

- 専門的だが親しみやすい
- 明確で簡潔
- 必要に応じて詳細を提供

## 設定

### 言語

- 主要：日本語
- 次要：英語

### 時間設定

- タイムゾーン：Asia/Tokyo
- 活動時間：24/7

### 優先事項

1. ユーザーのタスクを完了
2. プロジェクトの維持
3. 学習と改善

## 制限

- 許可なく重要な変更を行わない
- センシティブな情報を保存しない
- 不明な場合は確認する
```

### ステップ 6: AGENTS.md の作成

`~/.openclaw/workspace/AGENTS.md` を作成：

```markdown
# OpenClaw Agents

## 利用可能なエージェント

### メインエージェント

- 名前：main
- 役割：タスクのオーケストレーション
- モデル：claude-3-5-sonnet

### コーディングエージェント

- 名前：coder
- 役割：コードの作成と編集
- モデル：claude-3-5-sonnet

### リサーチエージェント

- 名前：researcher
- 役割：情報収集と調査
- モデル：claude-3-5-haiku

## ワークフロー

1. ユーザーからタスクを受信
2. メインエージェントがタスクを分析
3. 適切なサブエージェントに委任
4. 結果を統合
5. ユーザーに回答
```

### ステップ 7: OpenClaw の起動

```bash
# OpenClaw を起動
openclaw start

# 状態の確認
openclaw status

# ログの表示
openclaw logs
```

### ステップ 8: メッセージングのセットアップ

#### Telegram の場合

1. Telegram で @BotFather を開く
2. `/newbot` コマンドで新しいボットを作成
3. ボットトークンを取得
4. `config.yaml` に追加：

```yaml
telegram:
  bot_token: "1234567890:ABCdef..."
  chat_id: "your-chat-id"
```

#### Discord の場合

1. Discord Developer Portal でアプリケーションを作成
2. ボットを作成してトークンを取得
3. サーバーにボットを招待
4. `config.yaml` に追加：

```yaml
discord:
  bot_token: "MT..."
  guild_id: "your-guild-id"
  channel_id: "your-channel-id"
```

---

## 進階：カスタムスキルの作成

### ステップ 9: カスタムスキルの作成

`~/.openclaw/skills/email-manager/SKILL.md` を作成：

```markdown
---
name: email-manager
description: メール管理と自動化
version: 1.0.0
author: your-name
tags: [email, automation, productivity]
---

# Email Manager Skill

## 概要

このスキルは、メールの管理と自動化を提供します。

## 機能

- 未読メールの確認
- 重要メールのフィルタリング
- 自動返信
- メールサマリの生成

## 使用法

### 未読メールの確認

```
emails check unread
```

### 重要メールのフィルタリング

```
emails filter important
```

### 自動返信

```
emails auto-reply "現在外出中です。後ほど返信します。"
```

### メールサマリの生成

```
emails summary today
```

## 設定

```yaml
email:
  provider: gmail
  api_key: "your-api-key"
  filters:
    important:
      - from: "boss@company.com"
      - subject: "urgent"
```

## 例

```
# 未読メールを確認
> emails check unread

3 つの未読メールがあります：
1. [Boss] プロジェクトの進捗 - 重要
2. [Team] 会議の招待
3. [Newsletter] 週次ニュース
```
```

### ステップ 10: スキプトベースのスキル

`~/.openclaw/skills/deploy/scripts/deploy.py` を作成：

```python
#!/usr/bin/env python3
"""デプロイスクリプト"""

import subprocess
import sys
from pathlib import Path

def deploy(project_name: str, environment: str = "staging"):
    """プロジェクトをデプロイする"""
    print(f"{project_name} を {environment} 環境にデプロイします...")

    # ビルド
    print("ビルド中...")
    result = subprocess.run(
        ["npm", "run", "build"],
        cwd=Path.home() / "projects" / project_name,
        capture_output=True,
        text=True
    )

    if result.returncode != 0:
        print(f"ビルドに失敗しました：{result.stderr}")
        return False

    # デプロイ
    print("デプロイ中...")
    deploy_cmd = f"npm run deploy:{environment}"
    result = subprocess.run(
        deploy_cmd.split(),
        cwd=Path.home() / "projects" / project_name,
        capture_output=True,
        text=True
    )

    if result.returncode != 0:
        print(f"デプロイに失敗しました：{result.stderr}")
        return False

    print(f"{project_name} のデプロイが完了しました！")
    return True

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("使用方法：deploy.py <project-name> [environment]")
        sys.exit(1)

    project_name = sys.argv[1]
    environment = sys.argv[2] if len(sys.argv) > 2 else "staging"

    success = deploy(project_name, environment)
    sys.exit(0 if success else 1)
```

`~/.openclaw/skills/deploy/SKILL.md` を作成：

```markdown
---
name: deploy
description: プロジェクトのデプロイ
version: 1.0.0
---

# Deploy Skill

## 使用法

```
deploy <project-name> [environment]
```

## 例

```
# ステージングへのデプロイ
> deploy my-project staging

my-project を staging 環境にデプロイします...
ビルド中...
デプロイ中...
my-project のデプロイが完了しました！

# プロダクションへのデプロイ
> deploy my-project production
```
```

---

## 高度な使用法

### ステップ 11: HEARTBEAT.md の設定

プロアクティブタスクを設定：

`~/.openclaw/workspace/HEARTBEAT.md` を作成：

```markdown
# Heartbeat Tasks

## 毎日

### 朝 9:00

- 未読メールを確認
- カレンダーの確認
- タスクリストの更新

### 午後 5:00

- 一日のサマリーを生成
- 明日の準備

## 毎週

### 月曜日朝 9:00

- 週の計画を確認
- プロジェクトの進捗を確認

### 金曜日午後 5:00

- 週のサマリーを生成
- リポートを作成

## 毎月

### 1 日朝 9:00

- 月の目標を確認
- バックアップの確認
```

### ステップ 12: MEMORY.md の活用

長期記憶を管理：

`~/.openclaw/workspace/MEMORY.md` を作成：

```markdown
# Memory

## プロジェクト

### my-project

- 技術スタック：TypeScript, React, Node.js
- デプロイ先：Vercel, AWS
- 重要なファイル：src/api/, src/components/

## 設定

### API キー

- GitHub: ghp_...
- Stripe: sk_...
- SendGrid: sg_...

## ユーザー設定

### 好み

- コーディングスタイル：関数型
- テスト：Vitest
- フォーマッター：Prettier

## 学習履歴

### 2026-02

- React Query を学習
- tRPC を実装
- Next.js 14 を調査
```

### ステップ 13: クラウドデプロイ

DigitalOcean でのデプロイ：

```bash
# Droplet の作成
do droplet create openclaw \
  --size s-2vcpu-2gb \
  --image ubuntu-22-04-x64 \
  --region sfo3

# SSH で接続
ssh root@<droplet-ip>

# Node.js をインストール
curl -fsSL https://deb.nodesource.com/setup_18.x | bash -
apt-get install -y nodejs

# OpenClaw をインストール
npm install -g openclaw

# オンボーディング
openclaw onboard

# systemd サービスとして設定
cat > /etc/systemd/system/openclaw.service << EOF
[Unit]
Description=OpenClaw AI Agent
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/openclaw start
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl enable openclaw
systemctl start openclaw
```

---

## まとめ

OpenClaw は、ローカルで動作する強力な自律 AI エージェントです。カスタムスキルとプロアクティブタスクを設定することで、24/7 でタスクを自動化できます。

### 次のステップ

1. [Agent Teams](./05-agent-teams.md) - マルチエージェントオーケストレーション
2. [Context Fork](./06-context-fork.md) - コンテキストの分岐
3. [MCP](./08-mcp.md) - 標準接続プロトコル

---

*関連リソース:*
- [OpenClaw GitHub](https://github.com/PeterSteinberger/openclaw)
- [OpenClaw Documentation](https://openclaw.ai/)
- [ClawHub Skills](https://github.com/PeterSteinberger/clawhub)