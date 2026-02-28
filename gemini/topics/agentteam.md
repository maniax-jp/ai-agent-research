---
description: 2026年以降のAIエージェントの技術的動向：AgentTeam
---

# AgentTeam によるマルチエージェント・オーケストレーション

## 1. 概要と概念

**AgentTeam** とは、複数の特化型AIエージェントを透過的に連携させ、複雑なビジネスワークフローや開発プロセスを自動化するためのオーケストレーション（指揮・統合）フレームワーク、またはその概念です。

単一の高度なLLMにすべてを任せるのではなく、「リサーチャー」「コーダー」「テスター」「レビューアー」といった役割（ロール）を各エージェントに割り当て、チームとして機能させます。

### なぜチーム化（マルチエージェント）が必要なのか？

*   **専門性の分離 (Separation of Concerns)**: 「コードを書くプロンプト」と「テストを書くプロンプト」は相反することがあります。役割を分けることで、各エージェントが最大のパフォーマンスを発揮できます。
*   **相互監視と自己修復**: コーダーエージェントが書いたコードを、独立したレビューアーエージェントが評価することで、品質が担保され、ハルシネーションを早期に発見・修正（自己修復）できます。
*   **明確な「Done」の定義（実行契約）**: ロールごとに「受入基準が明文化されたか（PM）」「テストが通ったか（Coder）」「重大な脆弱性がないか（Reviewer）」といった完了条件（Done）を技術的に固定し、手戻りの原因を局所化できます。

## 2. ワークフローの代表的なパターン

AgentTeamにおけるプロセス連携には、アーキテクチャに応じていくつかのパターンが存在します。

1.  **シーケンシャル（パイプライン型）**: 
    PM(要件定義) -> Coder(実装) -> Tester(テスト検証) -> Reviewer(コードレビュー)。出力が次のエージェントの入力になります。
2.  **階層型（Manager-Worker）**: 
    Managerが全体を統括し、複数のWorkerエージェント（Frontend担当、Backend担当など）に独立したタスクを並列で割り当て、最後にManagerが結果を統合します。
3.  **議論型（Debate / Consensus）**: 
    複数の専門家エージェントが、ある設計案についてディスカッションを行い、合意形成を図ります。
4.  **Swarm（群れ型）**: 
    中央集権的なオーケストレーターを持たず、エージェント同士が必要に応じて直接通信とタスク委譲を行い合う分散型アプローチです。

## 3. ハンズオン：AgentTeamの概念実装の実証

ここでは、Python（擬似コード）を用いて、`Manager`、`Coder`、`Tester` の3つのエージェントからなるシンプルな「ソフトウェア開発AgentTeam」を構築する例を示します。

### ステップ 1: エージェントの定義

役割を持った各エージェントクラスを定義します。

```python
# agents.py
class BaseAgent:
    def __init__(self, role, instructions):
        self.role = role
        self.instructions = instructions

    def prompt(self, task, context=""):
        # LLMへのAPI呼び出しをシミュレート
        print(f"\n[{self.role}] タスクを受信: {task}")
        # print(f"前提知識: {context}")
        return f"{self.role}による処理結果: {task} を完了しました。"

class CoderAgent(BaseAgent):
    def generate_code(self, spec):
        result = self.prompt(f"仕様に基づいてPythonコードを生成して: {spec}")
        # 擬似的にバグを含むコードを生成したと仮定
        return "def add(a, b):\n    return a - b  # Bug!"

class TesterAgent(BaseAgent):
    def run_tests(self, code):
        result = self.prompt(f"次のコードのユニットテストを書き、実行して: {code}")
        # バグを検出したと仮定
        return "テスト失敗: add(1, 2) が -1 を返しました。"
```

### ステップ 2: チームの管理とオーケストレーション（Manager）

複雑なロジックツリーを管理するManagerエージェントを作成します。

```python
# team_manager.py
from agents import CoderAgent, TesterAgent

class TeamManager:
    def __init__(self):
        self.role = "Manager"
        self.coder = CoderAgent("Senior Python Coder", "PEP8に準拠した高品質なコードを書く")
        self.tester = TesterAgent("QA Engineer", "エッジケースを含む厳格なテストを行う")

    def run_workflow(self, feature_request):
        print(f"[{self.role}] 開発フロー開始: {feature_request}")
        
        # 1. Coderに実装を依頼
        code = self.coder.generate_code(feature_request)
        print(f"[{self.role}] Coderからコードを受け取りました。:\n{code}")

        max_retries = 3
        for attempt in range(max_retries):
            # 2. Testerにテストを依頼
            test_result = self.tester.run_tests(code)
            print(f"[{self.role}] Testerからの報告: {test_result}")

            # 3. テスト合否判定と自己修復(Self-healing)ループ
            if "テスト失敗" in test_result:
                print(f"[{self.role}] バグを検出。Coderに修正を依頼します (試行 {attempt + 1}/{max_retries})")
                # エラーコンテキストを付与して再生成
                code = self.coder.generate_code(f"元の仕様: {feature_request}\nエラー内容: {test_result}\nこれを修正して。")
                # 擬似的に修正成功したと仮定
                if attempt == 0:
                     code = "def add(a, b):\n    return a + b  # Fixed!"
            else:
                print(f"\n[{self.role}] テスト合格。開発フローを完了します。")
                return code
                
        print(f"[{self.role}] 最大再試行回数に達しました。フローを中止します。")
        return code

# 実行
if __name__ == "__main__":
    manager = TeamManager()
    final_code = manager.run_workflow("2つの数値を足し合わせる関数")
```

### コードの解説とAgentTeamの利点

このサンプルでは以下のプロセスが自動化されています。

1.  ManagerがタスクをCoderに渡す（**生成**）。
2.  Coderの成果物をTesterに渡す（**検証**）。
3.  検証に失敗した場合、そのエラー出力をコンテキストとして再びCoderに渡し、自己修復ループを回す（**修正**）。

2026年においては、これらをハードコードするのではなく、エージェント間の「受け渡し形式（JSONなど）」と「完了条件」を明確な契約（Contract）として定義し、YAMLやGUIノードエディタ（n8n, Dify, LangFlow等）を使って視覚的に**エージェントの有向非巡回グラフ(DAG)**を構築するスタイルが主流となっています。

#### 受け渡し契約（Handoff Contract）の例
エージェント間でタスクを渡す際、何を必須フィールドとするかを厳格に定めます。
```json
{
  "handoff": {
    "from": "PM Agent",
    "to": "Coder Agent",
    "required_fields": ["task_id", "requirements", "acceptance_criteria"]
  }
}
```
