# AI Agent技術動向レポート（Codex担当）

- 作成者: codex
- 作成日: 2026-02-28
- 対象: 2026年以降のAI Agent技術トレンド（実装視点）

## このページの役割
この `report.md` は目次です。各技術動向の詳細説明と、ハンズオン可能なステップは別ページに分割しています。

## 技術動向ページ一覧
1. [Skillsのモジュール化と配布可能化](./topics/01-skills.md)
2. [subagent前提の階層型オーケストレーション](./topics/02-subagent.md)
3. [AGENTS.mdの実行契約化](./topics/03-agents-md.md)
4. [OpenClaw系ツールチェーンの活用](./topics/04-openclaw.md)
5. [AgentTeam設計の定式化](./topics/05-agentteam.md)
6. [context forkによる並列探索](./topics/06-context-fork.md)
7. [worktreeベース開発の標準化](./topics/07-worktree.md)
8. [MCP（Model Context Protocol）によるツール連携標準化](./topics/08-mcp.md)

## 推奨読了順
1. `03-agents-md.md` で全体ルールを定義
2. `01-skills.md` で再利用可能な実行単位を整備
3. `02-subagent.md` と `05-agentteam.md` で分業設計
4. `06-context-fork.md` で代替案探索を並列化
5. `07-worktree.md` で安全な並列実装
6. `08-mcp.md` で外部ツール連携を標準化
7. `04-openclaw.md` でE2E評価を継続運用

## 期待アウトカム
- 単体Agent最適化から、TeamとしてのE2E成功率最適化へ移行できる
- 仕様変更や並列開発時の衝突コストを低減できる
- 再利用性（Skills）と統制（AGENTS.md）を両立できる

## 今回のアップデート方針
- 他担当ドキュメントを参照し、欠落していた `MCP` を追加
- 各ページに「手順の検証方法」と「失敗時の対処」を追記
- 実運用で使えるよう、出力物（ログ/契約ファイル/判定基準）を明示
