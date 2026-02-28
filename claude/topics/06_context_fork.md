# Context Fork — 会話コンテキストの分岐

**カテゴリ:** Claude Code / Agent SDK
**導入時期:** 2025年（Claude Code 2.1.0 で正式機能化）
**ステータス:** 本番利用可能（2026年現在）

---

## 1. 概要

**Context Fork** は、現在の会話コンテキストを「分岐」させて独立した探索を行う機能です。

```
メインコンテキスト（現在まで）
         │
         ├──── Fork A ────→ 実装アプローチ1を試す
         │                   └─ 成功したらメインに採用 / 失敗しても元に戻れる
         │
         └──── Fork B ────→ 実装アプローチ2を試す
                             └─ 成功したらメインに採用
```

### なぜ Context Fork が必要か

- **実験的な試みを安全に行える** — 失敗しても元のコンテキストは保持される
- **複数アプローチの並列比較** — どの実装方法が良いか同時に試せる
- **コンテキストの「汚染」を防ぐ** — 不要な情報がメインの判断に影響しない
- **Subagent の実行環境** — `context: fork` を持つ Skill や Subagent は自動的に Fork で実行

---

## 2. Fork の種類

### 2.1 インタラクティブ Fork（Esc + Esc）

Claude Code のインタラクティブモードで使用。現在の会話をフォークポイントとして分岐します。

```
[会話]
You: ユーザー認証を実装してください
Claude: JWTを使って実装します...（長いコンテキスト）
You: もっとシンプルな方法でも試したい

→ Esc + Esc を押す
  → フォーク前の時点から新しい方向で再開できる
  → 元のコンテキスト（JWT実装）は保持される
```

### 2.2 Skill による自動 Fork

`context: fork` を設定した Skill は自動的に独立したコンテキストで実行されます（[01_skills.md](01_skills.md) 参照）。

### 2.3 SDK による明示的 Fork

Agent SDK からプログラム的にコンテキストをフォークできます。

---

## 3. Claude Agent SDK での Context Fork

### 基本的な Fork API

```python
import anthropic
from anthropic.agent_sdk import AgentSession

client = anthropic.Anthropic()
session = AgentSession(client=client, model="claude-opus-4-6")

# メインコンテキストでの作業
result = session.run("このコードの問題点を分析してください: ...")
analysis = result.output

# コンテキストをフォークして2つのアプローチを並列で試す
fork_a = session.fork()  # 現在のコンテキストをコピー
fork_b = session.fork()  # 同じコンテキストをもう一つコピー

# 並列で異なる解決策を探索
result_a = fork_a.run("アプローチA: リファクタリングで解決してください")
result_b = fork_b.run("アプローチB: 別のライブラリを使って解決してください")

# 結果を比較してメインコンテキストに最良の解決策を採用
comparison = session.run(f"""
以下の2つのアプローチを比較して、最良の方を選んでください：

アプローチA:
{result_a.output}

アプローチB:
{result_b.output}
""")

print(comparison.output)
```

### Fork チェーンパターン

```python
# 段階的に絞り込んでいく Fork チェーン
session = AgentSession(client=client, model="claude-opus-4-6")

# Phase 1: アイデア生成（広く探索）
idea_forks = []
ideas = ["REST API", "GraphQL", "gRPC", "WebSocket"]

for idea in ideas:
    fork = session.fork()
    result = fork.run(f"{idea}でこの機能を実装した場合の設計をスケッチしてください")
    idea_forks.append((idea, fork, result))

# Phase 2: 最良の2つを選んで詳細設計
# （人間またはオーケストレーターが評価）
top_two = evaluate_and_select(idea_forks, n=2)

# Phase 3: 選んだ2つを詳細実装まで進める
detail_forks = []
for name, fork, initial_result in top_two:
    detail_fork = fork.fork()  # フォークのフォーク
    detail_result = detail_fork.run("詳細な実装計画と概算コードを作成してください")
    detail_forks.append((name, detail_result))

# 最終比較
final_result = session.run(f"詳細設計を比較して最終決定してください: {detail_forks}")
```

---

## 4. ハンズオン：複数アプローチを Fork で比較する

### ユースケース：データキャッシュ実装の最適解を探す

既存の重い API 呼び出しをキャッシュしたいが、実装方法が複数ある場合に Fork で比較します。

### Step 1: 問題を準備する

```bash
mkdir -p ~/fork-demo/src
cd ~/fork-demo

cat > src/user-service.ts << 'EOF'
// 遅い外部API呼び出し（キャッシュしたい）
export class UserService {
  async getUser(id: string) {
    // 毎回 DB に問い合わせ → 遅い
    const response = await fetch(`https://api.example.com/users/${id}`);
    return response.json();
  }

  async getUserPosts(userId: string) {
    // 関連データも毎回取得 → N+1問題
    const user = await this.getUser(userId);
    const posts = await fetch(`https://api.example.com/posts?userId=${userId}`);
    return { user, posts: await posts.json() };
  }
}
EOF
```

### Step 2: Fork で複数アプローチを並列探索する

```python
# fork_comparison.py
import anthropic
from anthropic.agent_sdk import AgentSession

client = anthropic.Anthropic()

# 問題コードを読み込む
with open("src/user-service.ts") as f:
    source_code = f.read()

session = AgentSession(
    client=client,
    model="claude-sonnet-4-6",
    system="TypeScriptエンジニアとしてコードを改善してください"
)

# 現在のコードを分析（メインコンテキスト）
session.run(f"以下のコードを理解してください：\n```typescript\n{source_code}\n```")

print("メインコンテキストで分析完了。3つのアプローチを並列で試します...")

# 3つの Fork を作成して並列実行
approaches = {
    "in-memory": "Node.jsのMapを使ったシンプルなインメモリキャッシュを実装してください。TTL（有効期限）付きで。",
    "redis": "Redisを使ったキャッシュ実装を提案してください。接続管理・シリアライズも含めて。",
    "react-query-style": "React Queryのような楽観的更新パターンを使ったキャッシュ戦略を実装してください。",
}

fork_results = {}
for approach_name, instruction in approaches.items():
    fork = session.fork()
    result = fork.run(instruction)
    fork_results[approach_name] = {
        "code": result.output,
        "fork": fork,
    }
    print(f"✅ {approach_name} アプローチ完了")

# メインセッションで比較評価（Fork の内容はメインには影響しない）
comparison_prompt = "\n\n".join([
    f"### アプローチ: {name}\n{data['code']}"
    for name, data in fork_results.items()
])

evaluation = session.run(f"""
以下の3つのキャッシュ実装を比較してください：

{comparison_prompt}

評価基準：
1. 実装の複雑さ
2. パフォーマンス
3. 運用コスト
4. テストのしやすさ

最終的にどのアプローチを採用すべきか、理由とともに推薦してください。
""")

print("\n=== 比較評価結果 ===")
print(evaluation.output)

# 採用したアプローチの Fork を継続して詳細実装を進める
best_approach = "in-memory"  # 評価結果から決定
chosen_fork = fork_results[best_approach]["fork"]

final_implementation = chosen_fork.run(
    "採用決定しました。テストコードも含めた完全な実装を作成してください。"
)

print(f"\n=== {best_approach} アプローチの最終実装 ===")
print(final_implementation.output)
```

### Step 3: 実行する

```bash
pip install anthropic
python fork_comparison.py
```

### Step 4: 比較基準の事前固定と Decision ログ（Codex パターン）

Codex の設計では「採用/棄却の理由を必ず記録する」ことが重要とされています。Fork で試した内容と意思決定を記録することで、将来の設計議論でも再利用できます。

```python
# decision_log.py
import json
from pathlib import Path
from datetime import datetime

def record_fork_decision(
    task_id: str,
    forks: dict,           # {アプローチ名: {code, pros, cons}}
    chosen: str,           # 採用したアプローチ名
    reason: str,           # 採用理由
    rejected: dict,        # {アプローチ名: 棄却理由}
):
    """Fork の意思決定を記録する"""
    decision = {
        "task_id": task_id,
        "timestamp": datetime.now().isoformat(),
        "comparison_criteria": [      # 比較基準（事前に固定することが重要）
            "実装の複雑さ",
            "パフォーマンス",
            "テストのしやすさ",
            "運用コスト",
        ],
        "forks_evaluated": [
            {
                "name": name,
                "summary": data.get("summary", ""),
                "pros": data.get("pros", []),
                "cons": data.get("cons", []),
            }
            for name, data in forks.items()
        ],
        "decision": {
            "chosen": chosen,
            "reason": reason,
            "rejected": rejected,
        },
    }

    # decisions/ ディレクトリに保存
    Path("decisions").mkdir(exist_ok=True)
    log_path = Path(f"decisions/{task_id}-{datetime.now().strftime('%Y%m%d')}.json")
    with open(log_path, "w") as f:
        json.dump(decision, f, ensure_ascii=False, indent=2)

    print(f"意思決定を記録しました: {log_path}")
    return decision


# 使用例
record_fork_decision(
    task_id="cache-implementation",
    forks={
        "in-memory": {
            "summary": "Node.js Mapを使ったインメモリキャッシュ",
            "pros": ["実装シンプル", "依存なし", "テスト容易"],
            "cons": ["サーバー再起動で消える", "複数インスタンスで共有不可"],
        },
        "redis": {
            "summary": "Redisを使った分散キャッシュ",
            "pros": ["永続化可能", "複数インスタンス対応"],
            "cons": ["インフラ追加", "接続管理が複雑"],
        },
    },
    chosen="in-memory",
    reason="受入基準を満たしつつ変更量が最小。現時点でRedisが不要なスケール。",
    rejected={"redis": "今回のスケールには過剰。将来的に移行可能な設計にする。"},
)
```

これにより `decisions/` ディレクトリに意思決定ログが蓄積され、将来の設計レビューで参照できます。

---

## 5. Skills の `context: fork` との違い

```
Skills の context: fork
  ├─ スキル実行中だけ独立したコンテキスト
  ├─ 実行完了後は結果のみをメインに返す
  └─ フォーク内部の詳細はメインに残らない

SDK の session.fork()
  ├─ 任意のタイミングで手動フォーク
  ├─ フォークは独立して継続できる
  ├─ 複数フォークを並列実行可能
  └─ フォークの状態を保存・復元可能
```

---

## 6. Context Fork のユースケース一覧

| ユースケース | 説明 |
|-------------|------|
| **A/Bテスト的な実装比較** | 2つの実装を並列で試して良い方を採用 |
| **仮説検証** | 「このアプローチが正しいか」を試す |
| **リスクの高い操作の事前確認** | フォーク内でシミュレートして問題なければ本番に適用 |
| **大量データのサンプリング** | フォークで一部データを処理して完全実行の判断 |
| **デバッグの探索** | バグの原因仮説を並列で検証 |
| **設計オプションの評価** | 複数の設計を具体化して比較 |

---

## 参考リンク

- [Claude Code 2.1.0 Release Notes - Context Fork](https://releasebot.io/updates/anthropic)
- [Subagents in the SDK - Claude API Docs](https://platform.claude.com/docs/en/agent-sdk/subagents)
- [Understanding Claude Code's Full Stack](https://alexop.dev/posts/understanding-claude-code-full-stack/)
