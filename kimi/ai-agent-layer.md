# 2026年現在のAIエージェントにおける接続関係（ハーネス・モデルサービスプロバイダ・モデル）

2026年3月1日時点での整理です。  
（用語）`ハーネス` = エージェント実行基盤/オーケストレーション層、`モデルサービスプロバイダ` = API提供者、`モデル` = 実際に推論するLLM本体。

> **補足**: 以下のダイアグラムでは主要な接続例を示しています。`Claude Code` および `Claude Cowork` もハーネスとして機能しますが、ここでは代表的な接続パターンとして図示していません。

```mermaid
flowchart LR
    A[Agent App / Business Logic]

    subgraph H[Harness（代表例）]
      H1[OpenAI Agents SDK]
      H2[LangGraph]
      H3[Dify]
      H4[Codex]
      H5[GitHub Copilot]
      H6[Google Antigravity]
    end

    subgraph P[Model Service Provider（代表例）]
      P1[OpenAI API]
      P2[Anthropic API]
      P3[Google Gemini API]
      P4[OpenRouter]
      P5[Azure AI]
      P6[Vertex AI]
    end

    subgraph M[Models（代表例）]
      M1[GPT最新系（通常: GPT-5.1 / Codex: GPT-5.3-Codex）]
      M2[Gemini最新系（Pro: Gemini 3.1 Pro / 軽量: Gemini 2.5 Flash-Lite）]
      M3[Claude Sonnet 4.5]
      M4[Qwen3.5]
      M5[DeepSeek3.2]
      M6[GLM-5]
    end

    A --> H1
    A --> H2
    A --> H3
    A --> H4
    A --> H5
    A --> H6

    H1 --> P1
    H2 --> P2
    H2 --> P6
    H3 --> P4
    H4 --> P1
    H5 --> P5
    H6 --> P3

    P1 --> M1
    P2 --> M3
    P3 --> M2
    P4 --> M4
    P4 --> M5
    P4 --> M6
    P5 --> M1
    P6 --> M2
```

## 著名な例と特徴

### 1. ハーネス（実行基盤）
1. **OpenAI Agents SDK**: エージェントを「モデル+instructions+tools(+MCP)」として構成し、handoff/manager型のマルチエージェント設計がしやすい。  
2. **Codex**: コーディング用途に最適化されたエージェント実行環境。コード編集・実行・検証のループを短く回しやすい。  
3. **Claude Code**: ターミナル内での実装作業に強いコードエージェント。リポジトリ読解から修正までを一貫して進めやすい。  
4. **Claude Cowork**: 人間との共同作業を前提にしたエージェント体験。文脈共有と段階的な作業分担に向く。  
5. **Google Antigravity**: エージェント駆動の開発・自動化に焦点を当てた実行基盤。複数モデルを用途別に使い分けやすい。  
6. **AutoGen AgentChat**: 高レベルAPIでマルチエージェントを組みやすく、下層`autogen-core`でイベント駆動に落とせる。  
7. **GitHub Copilot**: IDE/CLI統合が強く、補完・チャット・編集提案を開発フローに自然に組み込める。  
8. **CrewAI**: crews/flows中心で、guardrails・memory・observabilityを最初から組み込みやすい。  
9. **Dify**: GUI中心でLLMアプリ/エージェントを素早く構築しやすい。ワークフロー・ナレッジ・運用機能がまとまっている。  
10. **LangGraph**: 永続化チェックポイントを前提にした durable execution が強み。停止・再開・HITLに向く。  
11. **OpenClaw**: OSS寄りのエージェント実行/実験基盤。ツール実行やワークフローを組み合わせやすい。

> **「完全自立対応」の定義**: サーバ型実行基盤で、人間の介入なしに自律的にタスクを完結できるかどうか。「一部対応」はローカル実行主体や、人間の承認/確認を前提とする設計のもの。

| ハーネス | 主な操作形態 | 実行基盤 | 完全自立対応 | 利用者目線の補足 |
|---|---|---|---|---|
| OpenAI Agents SDK | SDK型 | サーバ型 | はい | API中心でアプリへ組み込みやすい |
| Codex | CLI型 | ローカル型/クラウド型 | 一部対応 | 実装・修正・検証の開発ループ向け |
| Claude Code | CLI型 | ローカル型/クラウド型 | 一部対応 | リポジトリ作業を対話で進めやすい |
| Claude Cowork | GUI/対話型 | ローカル型/クラウド型 | 一部対応 | 人間との共同作業を前提に運用しやすい |
| Google Antigravity | IDE型 | ローカル型 | 一部対応 | マルチエージェントの管理・フィードバックを前提に、開発ワークフローへ統合しやすい |
| AutoGen AgentChat | SDK型 | サーバ型 | はい | マルチエージェント実験から本番まで展開しやすい |
| GitHub Copilot | IDE型 | ローカル型/クラウド型 | 一部対応 | 開発者作業への常時支援に向く |
| CrewAI | SDK型 | サーバ型 | はい | 役割分担型エージェント設計がしやすい |
| Dify | GUI型 | サーバ型 | はい | ノーコード/ローコードで立ち上げやすい |
| LangGraph | SDK型 | サーバ型 | はい | 状態管理・停止再開・承認フローに強い |
| OpenClaw | CLI型 | ローカル型 | 一部対応 | OSSベースで柔軟に拡張しやすく、端末操作フローに馴染みやすい |

### 2. モデルサービスプロバイダ（API提供者）
1. **Amazon Bedrock**: 単一のAWS窓口で複数社モデルを扱える「集約プロバイダ」型。  
2. **Azure AI**: Azure上でOpenAI系を含むモデル提供・運用統合がしやすい。  
3. **Google Gemini API**: 2.5 Pro/Flash-Liteなど、推論重視と低コスト高速を分けて選びやすい。  
4. **Vertex AI**: Google Cloud上でGeminiを中心にモデル利用とMLOpsを統合しやすい。  
5. **OpenAI API**: GPT-5系を中心に、関数呼び出し/構造化出力などエージェント向け機能が厚い。  
6. **Anthropic API**: Claude 4.x系（Opus/Sonnet/Haiku）の性能帯が明確で、長文コンテキスト運用がしやすい。  
7. **OpenRouter**: 複数ベンダーのモデルを単一APIでルーティングできる集約レイヤー。

### 3. モデル（代表）
1. **GPT-5.1 / GPT-5.3-Codex**: 用途別に系統を分けた最新構成。Codex版はコーディング特化。  
2. **Claude Opus 4.6 / Sonnet 4.5 / Haiku 4.5**: 知能-速度-コストの階層が明確。  
3. **Gemini 3.1 Pro / Gemini 2.5 Flash-Lite**: 複雑タスクと高速・低コスト用途を分けて選びやすい。  
4. **Qwen3.5**: 多言語・実用タスクで使われることが多いモデル系列。  
5. **DeepSeek-V3.2**: コーディングや推論系ワークロードで採用されることがあるモデル系列。  
6. **GLM-5**: 中国語・英語を含む多言語用途で選択肢になりやすいモデル系列。

## 参照（公式）
- OpenAI Agents SDK: https://openai.github.io/openai-agents-python/agents/  
- LangGraph Durable Execution: https://docs.langchain.com/oss/javascript/langgraph/durable-execution  
- AutoGen AgentChat: https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/index.html  
- CrewAI Docs: https://docs.crewai.com/  
- OpenAI GPT-5.1 model: https://platform.openai.com/docs/models/gpt-5.1  
- Anthropic Claude models overview: https://platform.claude.com/docs/about-claude/models/overview  
- Gemini models: https://ai.google.dev/gemini-api/docs/models  
- Bedrock supported models: https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html
