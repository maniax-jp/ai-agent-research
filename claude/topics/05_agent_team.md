# AgentTeam — マルチエージェント協調パターン

**カテゴリ:** アーキテクチャパターン / Claude Agent SDK
**ステータス:** 実用段階（2026年現在。Swarming 機能強化が予定）

---

## 1. 概要

**AgentTeam** は複数のAIエージェントが連携して複雑なタスクを分担・協調して解決するアーキテクチャパターンです。

単一エージェントでは対処しにくい問題（コンテキスト長の限界、専門性の必要性、並列処理の要件）を解決します。

### 基本モデル：オーケストレーター＋ワーカー

```
┌──────────────────────────────────────────────────────┐
│              オーケストレーター（指揮者）                │
│  タスク分解・優先度付け・結果統合・エラーハンドリング      │
└────────────────────┬─────────────────────────────────┘
                     │ サブタスクを委譲
        ┌────────────┼────────────┐
        ▼            ▼            ▼
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │Worker A  │ │Worker B  │ │Worker C  │
  │(Frontend)│ │(Backend) │ │(Testing) │
  └──────────┘ └──────────┘ └──────────┘
        │            │            │
        └────────────┴────────────┘
                     │ 結果を統合
                     ▼
             最終成果物・レポート
```

---

## 2. AgentTeam のトポロジーパターン

### パターン 1: Hierarchical（階層型）

```
オーケストレーター
    ├── サブオーケストレーターA（フロントエンドチーム）
    │       ├── UIエージェント
    │       └── スタイルエージェント
    └── サブオーケストレーターB（バックエンドチーム）
            ├── APIエージェント
            └── DBエージェント
```

大規模プロジェクト向け。役割が明確に分離される。

### パターン 2: Pipeline（パイプライン型）

```
エージェントA → エージェントB → エージェントC → 最終出力
（コード生成） （コードレビュー） （テスト作成）
```

出力が次のエージェントの入力になるシーケンシャルな処理。

### パターン 3: Swarm（群れ型）※2026年注目

```
エージェント1 ←→ エージェント2
      ↕                ↕
エージェント3 ←→ エージェント4
```

中央集権的なオーケストレーターを持たず、エージェント同士が直接通信。Anthropicが2026年に強化予定の「Swarming」機能。

### パターン 4: Self-healing Loop（自己修復ループ）

Gemini の AgentTeam 実装で示されているパターンです。コーダーエージェントが書いたコードをテスターが検証し、失敗した場合にエラー情報をコンテキストとして渡してコーダーが自動修正するループです。

```
Coder ──→ コード生成
  ↑              ↓
  │         Tester 検証
  │              ↓
  └── ❌失敗 ← テスト結果
              ↓
           ✅成功 → Manager → リリース
```

**ポイント**: 差し戻し時にエラーの内容を**次のプロンプトに含める**ことで、エージェントが「何が間違っていたか」を理解した上で修正できます。

### パターン 5: DAG（有向非巡回グラフ）型

条件分岐・ループを含む複雑なワークフローを表現します。2026年ではビジュアルオーケストレーションツールで定義するのが主流です。

```
要件取得
   ↓
設計判定 ──→ [シンプル] → 直接実装
   │
   └──→ [複雑] → アーキテクチャ設計 → レビュー → 実装
                                           ↓
                                    [差し戻し] → 再設計
```

**2026年の主要ビジュアルオーケストレーションツール**:
- **n8n** — ノードエディタでワークフローを定義、Claude MCP と統合可
- **LangFlow** — LangChain ベースのビジュアルDAGビルダー
- **Dify** — エージェントアプリを GUIで構築。Claude API に対応
- **Claude Agent SDK** — コードベースのDAG定義（より細かい制御が可能）

---

## 3. Claude Agent SDK での実装

### Step 1: オーケストレーターエージェントを定義する

```python
# orchestrator.py
import anthropic
from anthropic.agent_sdk import (
    AgentSession,
    SubagentConfig,
    TaskResult,
)

client = anthropic.Anthropic()

# ワーカーエージェントの定義
WORKER_AGENTS = {
    "architect": SubagentConfig(
        name="architect",
        description="システム設計・アーキテクチャの決定を担当する",
        system_prompt="""あなたはシステムアーキテクトです。
要件を分析し、最適なアーキテクチャを設計してください。
出力は必ず設計書（Markdown形式）にしてください。""",
        allowed_tools=["Read", "Grep", "Glob"],
    ),
    "implementer": SubagentConfig(
        name="implementer",
        description="コード実装を担当する",
        system_prompt="""あなたはシニアエンジニアです。
設計書に従って高品質なコードを実装してください。
TypeScriptを使用し、型安全性を維持してください。""",
        allowed_tools=["Read", "Write", "Edit", "Bash"],
    ),
    "tester": SubagentConfig(
        name="tester",
        description="テストコードの作成とテスト実行を担当する",
        system_prompt="""あなたはQAエンジニアです。
実装されたコードに対して包括的なテストを作成し、実行してください。
カバレッジ80%以上を目指してください。""",
        allowed_tools=["Read", "Write", "Edit", "Bash"],
    ),
    "reviewer": SubagentConfig(
        name="reviewer",
        description="コードレビューを担当する",
        system_prompt="""あなたはコードレビュアーです。
実装とテストを確認し、問題点と改善案を報告してください。""",
        allowed_tools=["Read", "Grep", "Glob"],
    ),
}


class SoftwareDevelopmentOrchestrator:
    def __init__(self):
        self.session = AgentSession(
            client=client,
            model="claude-opus-4-6",
            system="""あなたはソフトウェア開発チームのテックリードです。
タスクを適切に分解し、各専門エージェントに委譲してください。""",
        )

    def develop_feature(self, feature_description: str) -> dict:
        """新機能を設計→実装→テスト→レビューの流れで開発する"""

        print("=== Phase 1: アーキテクチャ設計 ===")
        design = self.session.run_subagent(
            agent=WORKER_AGENTS["architect"],
            task=f"以下の機能を設計してください：\n{feature_description}",
        )
        print(f"設計完了: {design.summary}")

        print("=== Phase 2: 実装 ===")
        implementation = self.session.run_subagent(
            agent=WORKER_AGENTS["implementer"],
            task=f"以下の設計に従って実装してください：\n{design.output}",
        )
        print(f"実装完了: {implementation.summary}")

        print("=== Phase 3: テスト作成（並列実行）===")
        # テスト作成とコードレビューを並列実行
        parallel_results = self.session.run_subagents_parallel([
            {
                "agent": WORKER_AGENTS["tester"],
                "task": f"以下のコードのテストを作成・実行してください：\n{implementation.output}",
            },
            {
                "agent": WORKER_AGENTS["reviewer"],
                "task": f"以下の実装をレビューしてください：\n{implementation.output}",
            },
        ])

        test_result = parallel_results[0]
        review_result = parallel_results[1]
        print(f"テスト: {test_result.summary}")
        print(f"レビュー: {review_result.summary}")

        return {
            "design": design.output,
            "implementation": implementation.output,
            "tests": test_result.output,
            "review": review_result.output,
        }


# 使用例
if __name__ == "__main__":
    orchestrator = SoftwareDevelopmentOrchestrator()
    result = orchestrator.develop_feature(
        "ユーザー認証機能：メールアドレスとパスワードでログイン、JWT発行"
    )
    print("\n=== 開発完了 ===")
    print(result["review"])
```

### Step 1.5: Self-healing ループの実装

Gemini の AgentTeam 実装から学んだ **自己修復パターン** を Claude Agent SDK で実装します。

```python
# self_healing_team.py
import anthropic

client = anthropic.Anthropic()

class CoderAgent:
    def generate_code(self, spec: str, previous_error: str = None) -> str:
        prompt = f"仕様に基づいてPythonコードを生成してください:\n{spec}"
        if previous_error:
            prompt += f"\n\n前回のエラー（これを修正してください）:\n{previous_error}"

        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            system="あなたはPythonエンジニアです。正しく動作するコードのみを返してください。",
            messages=[{"role": "user", "content": prompt}],
        )
        return response.content[0].text

class TesterAgent:
    def run_tests(self, code: str) -> dict:
        response = client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=512,
            system="""コードを受け取り、テスト結果を以下のJSON形式で返してください:
{"passed": true/false, "error": "エラーメッセージ（失敗時のみ）"}""",
            messages=[{"role": "user", "content": f"以下のコードをテストしてください:\n{code}"}],
        )
        import json
        try:
            return json.loads(response.content[0].text)
        except Exception:
            return {"passed": False, "error": response.content[0].text}

class SelfHealingManager:
    def __init__(self, max_retries: int = 3):
        self.coder = CoderAgent()
        self.tester = TesterAgent()
        self.max_retries = max_retries

    def develop(self, spec: str) -> dict:
        print(f"[Manager] 開発開始: {spec}")
        error = None

        for attempt in range(1, self.max_retries + 1):
            print(f"\n[Manager] 試行 {attempt}/{self.max_retries}")

            # Coder がコードを生成（エラーがある場合はコンテキストとして渡す）
            code = self.coder.generate_code(spec, previous_error=error)
            print(f"[Coder] コード生成完了")

            # Tester が検証
            result = self.tester.run_tests(code)

            if result["passed"]:
                print(f"[Tester] ✅ テスト合格！")
                print(f"[Manager] 開発完了")
                return {"success": True, "code": code, "attempts": attempt}
            else:
                error = result["error"]
                print(f"[Tester] ❌ テスト失敗: {error}")
                print(f"[Manager] エラー情報をCoderに渡して再試行します")

        return {"success": False, "code": code, "attempts": self.max_retries}


# 実行
if __name__ == "__main__":
    manager = SelfHealingManager(max_retries=3)
    result = manager.develop("2つの数値を加算する関数 add(a, b) を実装する")
    print(f"\n最終結果: {'成功' if result['success'] else '失敗'} ({result['attempts']}回試行)")
```

### Step 2: Claude Code での AgentTeam 設定

```yaml
# .claude/agent-team.yaml
name: development-team
description: フルスタック開発チームのエージェント設定

orchestrator:
  model: claude-opus-4-6
  system_prompt: |
    あなたはテックリードとして各専門エージェントを指揮します。
    タスクを分解し、適切なエージェントに委譲してください。

workers:
  - name: frontend-dev
    description: React/TypeScriptのフロントエンド実装
    allowed_tools: [Read, Write, Edit, Bash]
    model: claude-sonnet-4-6  # コスト最適化

  - name: backend-dev
    description: Node.js/PostgreSQLのバックエンド実装
    allowed_tools: [Read, Write, Edit, Bash]
    model: claude-sonnet-4-6

  - name: qa-engineer
    description: テスト作成と品質検証
    allowed_tools: [Read, Write, Edit, Bash]
    model: claude-haiku-4-5  # テスト生成は軽量モデルで十分

  - name: security-auditor
    description: セキュリティ脆弱性の検出
    allowed_tools: [Read, Grep, Glob]
    model: claude-opus-4-6  # セキュリティは最高品質で
```

---

## 4. ハンズオン：リポジトリ全体の健全性チェックを AgentTeam で実行する

### ユースケース

大規模リポジトリを複数エージェントで並列スキャンして健全性レポートを作成します。

### Step 1: スキャン対象を定義する

```bash
mkdir -p ~/repo-health-check
cd ~/repo-health-check

cat > health_check.py << 'EOF'
import asyncio
import anthropic
from anthropic.agent_sdk import AgentSession, SubagentConfig

client = anthropic.Anthropic()

# 各専門スキャナーエージェント
SCANNERS = [
    SubagentConfig(
        name="dependency-scanner",
        description="依存関係の脆弱性・古いバージョンをチェック",
        system_prompt="package.jsonやrequirements.txtを解析し、脆弱な依存関係を報告してください。",
        allowed_tools=["Read", "Glob", "Bash"],
    ),
    SubagentConfig(
        name="code-quality-scanner",
        description="コード品質（複雑度・重複・型安全性）をチェック",
        system_prompt="コードの複雑度・重複・型エラーを検出してください。",
        allowed_tools=["Read", "Glob", "Grep"],
    ),
    SubagentConfig(
        name="test-coverage-scanner",
        description="テストカバレッジと品質をチェック",
        system_prompt="テストファイルを解析し、カバレッジ不足の領域を報告してください。",
        allowed_tools=["Read", "Glob", "Bash"],
    ),
    SubagentConfig(
        name="doc-scanner",
        description="ドキュメントの充実度をチェック",
        system_prompt="README・JSDoc・型定義コメントを確認し、不足箇所を報告してください。",
        allowed_tools=["Read", "Glob", "Grep"],
    ),
]

async def run_health_check(repo_path: str):
    session = AgentSession(client=client, model="claude-opus-4-6")

    print(f"🔍 {repo_path} の健全性チェックを開始します...")

    # 全スキャナーを並列実行
    tasks = [
        {
            "agent": scanner,
            "task": f"{repo_path} を分析して問題点を報告してください"
        }
        for scanner in SCANNERS
    ]

    results = await session.run_subagents_parallel_async(tasks)

    # 結果を統合してレポートを作成
    report_data = {
        scanner.name: result.output
        for scanner, result in zip(SCANNERS, results)
    }

    # オーケストレーターが最終レポートを作成
    final_report = session.run(
        f"""以下の各エージェントのスキャン結果を統合して、
優先度付きの改善提案レポートを作成してください：

{report_data}

フォーマット：
## エグゼクティブサマリー
## 重大度別の問題一覧
## 推奨アクション（優先順）
"""
    )

    return final_report.output

if __name__ == "__main__":
    import sys
    repo_path = sys.argv[1] if len(sys.argv) > 1 else "."
    report = asyncio.run(run_health_check(repo_path))
    print(report)
EOF
```

### Step 2: 実行する

```bash
python health_check.py /path/to/your/repo
```

### Step 3: 期待される出力

```markdown
## エグゼクティブサマリー
4エージェントによる並列スキャンを完了しました。
重大な問題 2件、警告 8件、提案 15件が検出されました。

## 重大度別の問題一覧

### 🔴 CRITICAL
1. lodash 4.17.19 に既知の脆弱性 (CVE-2021-23337)
2. src/api/users.ts:45 SQLインジェクションリスク

### 🟡 WARNING
1. 全体のテストカバレッジが 42% (目標: 80%)
2. src/legacy/ に重複コードが 3箇所

## 推奨アクション
1. lodash を 4.17.21 以上にアップグレード（即時）
2. SQLクエリをプリペアドステートメントに変更（即時）
3. 認証モジュールのテストを追加（今週中）
```

---

## 5. コスト最適化戦略

AgentTeam では各エージェントに適切なモデルを割り当てることでコストを削減できます。

```python
MODEL_STRATEGY = {
    # 高精度が必要なタスク → 最高品質モデル
    "security-audit": "claude-opus-4-6",
    "architecture-design": "claude-opus-4-6",

    # 一般的な開発タスク → バランスモデル
    "implementation": "claude-sonnet-4-6",
    "code-review": "claude-sonnet-4-6",

    # 単純・大量処理タスク → 軽量モデル
    "test-generation": "claude-haiku-4-5-20251001",
    "doc-generation": "claude-haiku-4-5-20251001",
    "formatting": "claude-haiku-4-5-20251001",
}
```

---

## 参考リンク

- [Building agents with the Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)
- [Claude Agent SDK Best Practices](https://skywork.ai/blog/claude-agent-sdk-best-practices-ai-agents-2025/)
- [Claude Code Agent SDK: The Complete Beginner's Guide (2026)](https://www.contextstudios.ai/blog/claude-code-agent-sdk-the-complete-beginners-guide-to-building-ai-agents-2026)
