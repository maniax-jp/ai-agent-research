# OpenClaw — フルシステムアクセス型自律エージェントとE2E評価

**カテゴリ:** 自律エージェント / E2E評価 / セキュリティ
**参照:** codex担当「OpenClaw系ツールチェーンの活用」/ gemini担当「OpenClawとフルシステムアクセス環境」
**ステータス:** 実験的〜実用段階移行期（2026年現在）

---

## 1. 概要

**OpenClaw**（旧称: Clawdbot / Moltbot）は、ローカルマシンやVPSで**フルシステム権限（シェル実行・ファイル操作・ブラウザ自動化）**を持って自律動作するオープンソースの自律型エージェントです。

Claude Code との関係では：

| 比較軸 | Claude Code | OpenClaw |
|--------|-------------|----------|
| 権限モデル | 段階的承認（許可ベース） | フルアクセス（設定次第） |
| 実行環境 | クラウド推奨 / ローカル両対応 | ローカルファースト |
| データ保存 | Anthropicクラウド | ローカルMarkdown / SQLite |
| LLM | Claude専用 | Claude / ローカルLLM 切替可 |
| 用途 | ソフトウェア開発支援 | 汎用システム自動化 |

> Claude の視点：OpenClaw は「安全性をトレードオフに強力な自律性を得る」選択肢です。
> Claude Code は**段階的な承認モデル**で安全性を維持しつつ、MCP等で同等の統合を実現します。

---

## 2. OpenClaw のアーキテクチャ

```
┌─────────────────────────────────────────────────────────┐
│              OpenClaw エージェント                        │
│                                                         │
│  ┌─────────────┐    ┌──────────────────────────────┐   │
│  │ LLM Engine  │───▶│  自律思考ループ                │   │
│  │(Claude/Local│◀───│  ① タスク理解                 │   │
│  │ LLM)        │    │  ② コマンド生成               │   │
│  └─────────────┘    │  ③ 実行・結果読み取り          │   │
│                     │  ④ エラー → 再思考 → 再実行   │   │
│                     └──────────────────────────────┘   │
└─────────────────────────┬───────────────────────────────┘
                          │ システムアクセス
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼
    シェル実行          ファイル操作       ブラウザ自動化
    (bash/cmd)       (読み/書き/削除)    (Playwright等)
```

### IronClaw アーキテクチャ（セキュア版）

```
エージェント
    │
    ▼
Rust製セキュリティレイヤー（カーネルレベル制限）
    ├─ 許可: git, cargo build, npm test
    ├─ 拒否: rm -rf /, 外部への不正POST
    └─ TEE（信頼できる実行環境）でAPIキー保護
    │
    ▼
ホストOS
```

---

## 3. E2E評価パイプライン（Claudeエコシステムでの実践）

OpenClaw の最大の価値は「モデル品質の比較」ではなく、**実タスクの再現性（成功率・安定性）の継続評価**です。

Claude Code + Claude Agent SDK でも同等のE2E評価を構築できます。

### タスク定義スキーマ

```json
{
  "id": "fix-001",
  "goal": "failing test を1件修正する",
  "context": {
    "repo": "https://github.com/example/myapp",
    "branch": "main",
    "target_file": "src/auth.test.ts"
  },
  "success_criteria": [
    "all_tests_pass",
    "no_new_lint_error",
    "no_regression"
  ],
  "max_retries": 3,
  "timeout_seconds": 120
}
```

---

## 4. ハンズオン：自律的トラブルシューティング・ループを実装する

### Step 1: コマンド実行ラッパーを構築する

```python
# executor.py
import subprocess
import shlex

class SystemExecutor:
    # 安全のため許可コマンドを明示（AGENTS.md/CLAUDE.md のAllowedリストに対応）
    ALLOWED_PREFIXES = ("npm", "node", "python", "git log", "git status", "git diff")
    DENIED_PATTERNS = ("rm -rf", "git reset --hard", "git push --force", ":()")

    def _is_safe(self, command: str) -> bool:
        for denied in self.DENIED_PATTERNS:
            if denied in command:
                return False
        return True

    def execute(self, command: str, working_dir: str = ".") -> dict:
        if not self._is_safe(command):
            return {"status": "blocked", "output": f"安全でないコマンドをブロック: {command}"}

        print(f"[Executor] 実行: {command}")
        try:
            result = subprocess.run(
                shlex.split(command),
                cwd=working_dir,
                capture_output=True,
                text=True,
                timeout=30,
            )
            if result.returncode == 0:
                return {"status": "success", "output": result.stdout}
            else:
                return {"status": "error", "output": result.stderr}
        except subprocess.TimeoutExpired:
            return {"status": "timeout", "output": "タイムアウト（30秒）"}
        except FileNotFoundError as e:
            return {"status": "error", "output": f"コマンドが見つかりません: {e}"}
```

### Step 2: 自律的トラブルシューティング・ループを実装する

```python
# autonomous_agent.py
import anthropic
from executor import SystemExecutor

class AutonomousAgent:
    def __init__(self):
        self.client = anthropic.Anthropic()
        self.executor = SystemExecutor()
        self.history = []  # 実行ログ

    def _think(self, context: str) -> str:
        """LLMに状況を渡し、次のコマンドを決定させる"""
        response = self.client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=256,
            system="""あなたはターミナルを操作するエージェントです。
与えられた状況を分析し、次に実行すべき1つのシェルコマンドのみを返してください。
コマンドだけを返してください（説明不要）。""",
            messages=[{"role": "user", "content": context}],
        )
        return response.content[0].text.strip()

    def run_task_until_success(self, goal: str, working_dir: str = ".", max_steps: int = 5):
        print(f"\n[Agent] タスク開始: {goal}")
        current_error = None

        for step in range(1, max_steps + 1):
            print(f"\n--- Step {step}/{max_steps} ---")

            # 現在の状況をLLMに伝える
            context = f"目標: {goal}\n"
            if current_error:
                context += f"前回のエラー: {current_error}\n"
            context += "次のコマンドを1行で返してください:"

            # LLMがコマンドを決定
            command = self._think(context)

            # コマンドを実行
            result = self.executor.execute(command, working_dir)
            self.history.append({"step": step, "command": command, "result": result})

            if result["status"] == "success":
                print(f"[Agent] 成功:\n{result['output']}")
                print("[Agent] タスク完了！")
                return {"success": True, "steps": step, "history": self.history}
            elif result["status"] == "blocked":
                print(f"[Agent] 危険なコマンドをブロックしました: {command}")
                current_error = f"コマンド '{command}' はブロックされました。別のアプローチを試してください。"
            else:
                print(f"[Agent] エラー発生。再思考します...\n{result['output']}")
                current_error = result["output"]

        print("[Agent] 最大ステップ数に到達。タスク失敗。")
        return {"success": False, "steps": max_steps, "history": self.history}
```

### Step 3: E2E評価ランナーを実装する

```python
# eval_runner.py
import json
import time
from pathlib import Path
from datetime import datetime
from autonomous_agent import AutonomousAgent

class EvalRunner:
    def __init__(self, tasks_dir: str = "tasks", logs_dir: str = "logs"):
        self.tasks_dir = Path(tasks_dir)
        self.logs_dir = Path(logs_dir)
        self.logs_dir.mkdir(exist_ok=True)

    def run_task(self, task_file: Path) -> dict:
        with open(task_file) as f:
            task = json.load(f)

        print(f"\n{'='*50}")
        print(f"タスク: {task['id']} - {task['goal']}")
        print(f"{'='*50}")

        agent = AutonomousAgent()
        start_time = time.time()

        result = agent.run_task_until_success(
            goal=task["goal"],
            max_steps=task.get("max_retries", 5),
        )

        elapsed = time.time() - start_time
        log_entry = {
            "task_id": task["id"],
            "timestamp": datetime.now().isoformat(),
            "success": result["success"],
            "steps_taken": result["steps"],
            "elapsed_seconds": round(elapsed, 2),
            "history": result["history"],
        }

        # ログを保存
        log_path = self.logs_dir / f"{task['id']}-{datetime.now().strftime('%Y%m%d-%H%M%S')}.json"
        with open(log_path, "w") as f:
            json.dump(log_entry, f, ensure_ascii=False, indent=2)

        return log_entry

    def run_all(self) -> list:
        """tasks/ 配下の全タスクを実行して成功率を集計"""
        results = []
        for task_file in sorted(self.tasks_dir.glob("*.json")):
            result = self.run_task(task_file)
            results.append(result)

        # 集計レポート
        total = len(results)
        success = sum(1 for r in results if r["success"])
        avg_steps = sum(r["steps_taken"] for r in results) / total if total > 0 else 0

        print(f"\n{'='*50}")
        print(f"評価完了: {success}/{total} 成功 ({success/total*100:.1f}%)")
        print(f"平均ステップ数: {avg_steps:.1f}")
        print(f"{'='*50}")
        return results


# 使用例
if __name__ == "__main__":
    # タスク定義を作成
    Path("tasks").mkdir(exist_ok=True)
    task = {
        "id": "test-001",
        "goal": "npm test を実行してテスト結果を確認する",
        "max_retries": 3,
    }
    with open("tasks/test-001.json", "w") as f:
        json.dump(task, f, indent=2)

    runner = EvalRunner()
    runner.run_all()
```

### Step 4: 実行してログを確認する

```bash
pip install anthropic
python eval_runner.py

# ログを確認
ls logs/
cat logs/test-001-*.json
```

### 実行結果の例

```json
{
  "task_id": "test-001",
  "timestamp": "2026-02-28T10:30:00",
  "success": true,
  "steps_taken": 2,
  "elapsed_seconds": 8.42,
  "history": [
    {
      "step": 1,
      "command": "npm test",
      "result": {"status": "error", "output": "Error: Cannot find module './auth'"}
    },
    {
      "step": 2,
      "command": "npm install",
      "result": {"status": "success", "output": "added 42 packages"}
    }
  ]
}
```

---

## 5. Human-in-the-loop（安全運用）

完全自律は危険です。実運用でのガードレールを設定します。

```python
# 危険なコマンドは人間に確認を求める
REQUIRE_HUMAN_APPROVAL = [
    "git push",
    "git merge",
    "npm publish",
    "docker rm",
    "DROP TABLE",
]

def execute_with_approval(command: str) -> dict:
    if any(pattern in command for pattern in REQUIRE_HUMAN_APPROVAL):
        print(f"\n⚠️  重要な操作を検出しました: {command}")
        answer = input("実行しますか？ (y/N): ").strip().lower()
        if answer != "y":
            return {"status": "skipped", "output": "ユーザーによりスキップされました"}
    return executor.execute(command)
```

### Dockerコンテナによる完全サンドボックス

```bash
# エージェントを使い捨てコンテナで実行（メインOSを守る）
docker run --rm \
  -v $(pwd)/tasks:/tasks:ro \
  -v $(pwd)/logs:/logs \
  --network none \                # ネットワーク遮断
  --memory 512m \                 # メモリ制限
  -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  my-agent-image \
  python eval_runner.py
```

---

## 6. Claude Code との連携：OpenClaw 的な使い方

Claude Code 自体も Hooks + MCP を使えば OpenClaw 的な自律実行が可能です。

```json
// .claude/settings.json
{
  "hooks": {
    "pre_tool": [
      {
        "event": "pre_tool",
        "command": "bash scripts/safety-check.sh ${TOOL_NAME} ${TOOL_INPUT}"
      }
    ],
    "post_tool": [
      {
        "event": "post_tool",
        "command": "bash scripts/log-tool-use.sh ${TOOL_NAME} ${TOOL_RESULT}"
      }
    ]
  }
}
```

```bash
# scripts/safety-check.sh
#!/bin/bash
TOOL_NAME=$1
TOOL_INPUT=$2

# 危険なパターンをチェック
if echo "$TOOL_INPUT" | grep -qE "rm -rf|git reset --hard|git push --force"; then
  echo "BLOCKED: 危険なコマンドを検出" >&2
  exit 1  # exit 1 でツール実行をブロック
fi

exit 0  # OK
```

---

## 参考リンク

- [OpenClaw系ツールチェーンの活用 (codex担当)](../../../codex/topics/04-openclaw.md)
- [OpenClawとフルシステムアクセス環境 (gemini担当)](../../../gemini/openclaw.md)
- [Building agents with the Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)
