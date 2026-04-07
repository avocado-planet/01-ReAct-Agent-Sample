# LangChain ReAct Agent 簡易ガイド（新式 vs 旧式）

## はじめに

このドキュメントは、LangChain / LangGraph を使った ReAct Agent の仕組みを、初心者向けにゼロから解説するものです。添付の2つのサンプルコード（新式・旧式）を読み解きながら、違いを理解できるようになっています。

---

## 1. 登場人物の整理

### LangChain とは？

LLM（大規模言語モデル）を使ったアプリケーションを構築するためのフレームワークです。OpenAI、Anthropic、Google などの LLM を統一的なインターフェースで扱えるようにし、ツール呼び出しやプロンプト管理の仕組みを提供します。

主なコンポーネント:

| コンポーネント | 役割 | 例 |
|---|---|---|
| **LLM / ChatModel** | 言語モデルへの接続 | `ChatOpenAI`, `ChatAnthropic` |
| **Tool** | エージェントが使える外部機能 | Web検索、計算機、API呼び出し |
| **Prompt** | LLM への指示テンプレート | システムプロンプト、Few-shot例 |

### LangGraph とは？

LangChain の上に構築された、**ステートフルなワークフロー**を「グラフ」として定義するフレームワークです。

- **ノード（Node）**: 処理の単位（LLM呼び出し、ツール実行など）
- **エッジ（Edge）**: ノード間の接続（次にどのノードへ進むか）
- **ステート（State）**: ノード間で共有されるデータ（会話履歴など）

従来の「チェーン」（直線的な処理の連鎖）では表現しにくかった「条件分岐」「ループ」「並列処理」を、グラフ構造で自然に記述できます。

### LangSmith とは？

LangChain が提供する開発者向けプラットフォームで、以下の機能があります:

- **トレース**: エージェントの各ステップを可視化・デバッグ
- **Prompt Hub**: プロンプトテンプレートの共有・管理（hwchase17/react など）
- **評価**: エージェントの品質を定量的にテスト

---

## 2. Agent（エージェント）とは？

### 普通の LLM アプリとの違い

```
【普通の LLM アプリ】
  ユーザー → プロンプト → LLM → 回答
  （1回のやりとりで完結）

【エージェント】
  ユーザー → LLM が「考える」→ 必要なら「ツールを使う」
           → 結果を見て「また考える」→ ... → 最終回答
  （LLM 自身が判断して、複数回の行動を繰り返す）
```

エージェントの本質は、**LLM が自律的に「次に何をするか」を判断する**ことにあります。

### ReAct パターン

ReAct（Reasoning + Acting）は、最も広く使われているエージェントのアーキテクチャです。以下のループを繰り返します:

```
1. 思考（Thought）  : 現状を分析し、次に何をすべきか考える
2. 行動（Action）   : ツールを選んで実行する
3. 観察（Observation）: ツールの実行結果を確認する
4. → 1に戻る（十分な情報が集まれば最終回答を生成）
```

具体例:

```
Thought: 最新のスポーツ情報が必要。Web検索ツールを使おう。
Action: Search("2026/4/1 Dodgers starting pitcher")
Observation: "4月1日のドジャーズ対パドレス戦の先発は..."
Thought: 検索結果から回答を構成できる。
Final Answer: "2026年4月1日のドジャーズの先発投手は○○です。"
```

---

## 3. 世代の変遷と非推奨の状況

### 3つの世代

```
【第1世代】AgentExecutor + create_react_agent（langchain.agents / langchain_classic）
    テキストパース方式。{tools} 等のプレースホルダーを展開。
    LangChain 0.2 で非推奨。1.0 では langchain-classic パッケージに移動。
    → 旧式サンプル: react_agent_classic_style.py

【第2世代】langgraph.prebuilt.create_react_agent
    tool calling 方式。元のサンプルコードがこれに該当。
    LangGraph 1.0 で非推奨。
    → 元のサンプルコードはこの世代

【第3世代】langchain.agents.create_agent
    tool calling 方式 + ミドルウェア。LangChain 1.0 で導入。現在の推奨。
    内部で LangGraph を使用。
    → 新式サンプル: react_agent_new_style.py
```

### 注意事項（2026年4月時点）

第3世代の `create_agent` は v1.1.0 で一時的に消えた報告や、非推奨メッセージに記載されたインポートパスに `create_agent` が実際に存在しないというドキュメントバグの指摘もあり、移行先がまだ完全には安定していない状況です。バージョンピンニングを推奨します。

---

## 4. 新式と旧式の詳細比較

### 根本的な違い: ツール情報の渡し方

```
【旧式（テキストパース方式）】
  プロンプトに {tools} を「テキストとして埋め込む」
  → LLM が「テキストとして」"Action: Search" と出力
  → 正規表現でパースして実行

【新式（tool calling 方式）】
  ツール情報は OpenAI の function calling API で渡す
  → LLM が「構造化データとして」tool_call を返す
  → パース不要、直接実行
```

### 比較表

| 項目 | 旧式（第1世代） | 新式（第3世代） |
|---|---|---|
| **インポート** | `from langchain_classic.agents import AgentExecutor, create_react_agent` | `from langchain.agents import create_agent` |
| **エージェント作成** | `create_react_agent()` + `AgentExecutor()` の2段階 | `create_agent()` 1つだけ |
| **ツール情報の伝達** | プロンプトのテキストに埋め込み | OpenAI function calling API |
| **LLMの出力形式** | プレーンテキスト (`"Action: Search"`) | 構造化データ (`tool_call`) |
| **出力のパース** | 正規表現でパース | パース不要 |
| **プロンプト引数名** | `prompt=` | `system_prompt=`（改名） |
| **入力の渡し方** | `{"input": "質問文"}` | `{"messages": [("user", "質問文")]}` |
| **結果の取得** | `result["output"]` | `result["messages"]` |
| **プロンプトの `{}`** | `{tools}` `{tool_names}` `{input}` `{agent_scratchpad}` すべて展開 | 展開されない（不要） |
| **verbose モード** | `verbose=True` で全過程表示 | `debug=True` |
| **モデル指定** | `ChatOpenAI(model="gpt-4o-mini")` インスタンス必須 | 文字列 `"openai:gpt-4o-mini"` でもOK |
| **ミドルウェア** | なし | `middleware=[]` で拡張可能 |
| **非推奨ステータス** | 完全に非推奨 | **現在の推奨** |

---

## 5. hwchase17/react プロンプトの動作解説

### プロンプトテンプレートの内容

```
Answer the following questions as best you can. You have access to the following tools:

{tools}

Use the following format:

Question: the input question you must answer
Thought: you should always think about what to do
Action: the action to take, should be one of [{tool_names}]
Action Input: the input to the action
Observation: the result of the action
... (this Thought/Action/Action Input/Observation can repeat N times)
Thought: I now know the final answer
Final Answer: the final answer to the original input question

Begin!

Question: {input}
Thought:{agent_scratchpad}
```

### 旧式での `{}` 展開の詳細

**{tools} に埋まる内容（create_react_agent 呼び出し時に展開）:**
```
Search: Useful for answering questions about current events,
recent news, or real-time information that the LLM may not know.
```

**{tool_names} に埋まる内容（create_react_agent 呼び出し時に展開）:**
```
Search
```

**{input} に埋まる内容（invoke 実行時に展開）:**
```
2026/4/1ドジャーズの先発投手は？
```

**{agent_scratchpad} に埋まる内容（ループの各ステップで自動更新）:**
```
初回: ""（空文字列）

2回目以降:
  I need to search for current sports information.
  Action: Search
  Action Input: "2026年4月1日 ドジャーズ 先発投手"
  Observation: 4月1日のドジャーズ対パドレス戦の先発は...
  Thought:
```

### 旧式の実行フロー（図解）

```
【ステップ1: 初回 LLM 呼び出し】
  ┌──────────────────────────────────────────────────────────────┐
  │ Answer the following questions as best you can.              │
  │ You have access to the following tools:                      │
  │                                                              │
  │ Search: Useful for answering questions about current         │ ← {tools}
  │ events, recent news, or real-time information.               │
  │                                                              │
  │ ...                                                          │
  │ Action: should be one of [Search]                            │ ← {tool_names}
  │ ...                                                          │
  │ Question: 2026/4/1ドジャーズの先発投手は？                     │ ← {input}
  │ Thought:                                                     │ ← {agent_scratchpad}（空）
  └──────────────────────────────────────────────────────────────┘
      ↓
  LLM 出力: "Action: Search / Action Input: ..."
      ↓
  正規表現パース → SerpAPI 実行 → 検索結果取得
      ↓

【ステップ2: 2回目 LLM 呼び出し】
  ┌──────────────────────────────────────────────────────────────┐
  │ ...                                                          │
  │ Question: 2026/4/1ドジャーズの先発投手は？                     │
  │ Thought: I need to search for current sports information.    │
  │ Action: Search                                               │ ← {agent_scratchpad}
  │ Action Input: "2026年4月1日 ドジャーズ 先発投手"               │    に蓄積された
  │ Observation: 4月1日のドジャーズ対パドレス戦の先発は○○...       │    中間結果
  │ Thought:                                                     │
  └──────────────────────────────────────────────────────────────┘
      ↓
  LLM 出力: "Final Answer: ○○です。"
      ↓
  "Final Answer:" 検出 → ループ終了 → 回答を返す
```

### 新式ではどうなるか？

新式（create_agent）では Hub プロンプトは使わない。ツール情報は function calling API で伝達される。

```
  LLM に送られるもの:
  ┌──────────────────────────────────────────────────────────────┐
  │ 1. system メッセージ:                                        │
  │    "You are a helpful assistant. Think step by step..."      │
  │                                                              │
  │ 2. tools 定義（OpenAI function calling 形式）:               │
  │    [{"name":"Search", "description":"...", "parameters":{}}] │
  │                                                              │
  │ 3. messages:                                                 │
  │    user: "2026/4/1ドジャーズの先発投手は？"                    │
  └──────────────────────────────────────────────────────────────┘
```

---

## 6. 元のサンプルコードの問題点

元のサンプルコードは第2世代（`langgraph.prebuilt.create_react_agent`）を使い、Hub から hwchase17/react プロンプトを取得していましたが、以下の問題がありました:

1. **第2世代は非推奨**: `langgraph.prebuilt.create_react_agent` は LangGraph 1.0 で非推奨。現在の推奨は `langchain.agents.create_agent`（第3世代）。

2. **Hub プロンプトが無意味**: 第2世代の `create_react_agent` は `prompt=` をシステムメッセージとして扱うだけなので、`{tools}` `{tool_names}` 等は展開されない。Hub からのプロンプト取得は不要で、シンプルなシステムプロンプト文字列で十分。

3. **モデルが非推奨**: `gpt-3.5-turbo` は 2025 年以降非推奨。`gpt-4o-mini` がコスパ良好。

4. **エラーハンドリング不足**: 環境変数チェック、プロンプト取得失敗時のフォールバック、実行時エラーのハンドリングがなかった。

5. **pip パッケージの不足**: `google-search-results`（SerpAPIWrapper の内部依存）が未記載。

---

## 7. 必要なパッケージ一覧

### 新式（create_agent / 第3世代）

```bash
pip install \
    "langchain>=1.0" \         # LangChain 本体（create_agent を含む）
    langchain-openai \         # OpenAI 連携
    langchain-community \      # コミュニティユーティリティ（SerpAPIWrapper等）
    google-search-results      # SerpAPI の Python クライアント
```

### 旧式（AgentExecutor / 第1世代）

```bash
# LangChain 0.2〜0.3 の場合
pip install "langchain>=0.2,<1.0" langchain-community langchain-openai \
            langsmith google-search-results

# LangChain 1.0 以降の場合
pip install "langchain>=1.0" langchain-classic langchain-community \
            langchain-openai langsmith google-search-results
```

---

## 8. 必要な環境変数

| 環境変数 | 必須 | 用途 |
|---|---|---|
| `OPENAI_API_KEY` | ○ | OpenAI API の認証 |
| `SERPAPI_API_KEY` | ○ | SerpAPI（Google検索）の認証 |
| `LANGSMITH_API_KEY` | △ | LangSmith のプロンプトHub / トレース |
| `LANGCHAIN_TRACING_V2` | × | `"true"` でLangSmithトレースを有効化 |

---

## 9. 今後のステップアップ

### ツールを増やす

```python
from langchain_core.tools import tool

@tool
def calculator(expression: str) -> str:
    """Evaluate a mathematical expression and return the result."""
    return str(eval(expression))

tools = [search_tool, calculator]
```

### 別の LLM を使う

```python
# Anthropic Claude（新式では文字列でも指定可能）
agent = create_agent(model="anthropic:claude-sonnet-4-20250514", tools=[...])

# ChatModel インスタンスで指定
from langchain_anthropic import ChatAnthropic
llm = ChatAnthropic(model="claude-sonnet-4-20250514", temperature=0)
agent = create_agent(model=llm, tools=[...])
```

### ミドルウェアを活用する（第3世代の新機能）

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware, PIIMiddleware

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[...],
    system_prompt="You are a helpful assistant.",
    middleware=[
        # 会話が長くなったら自動要約
        SummarizationMiddleware(model="openai:gpt-4o-mini"),
        # 個人情報を自動マスク
        PIIMiddleware("email", strategy="redact", apply_to_input=True),
    ]
)
```

### メモリ（会話履歴の永続化）

```python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[...],
    checkpointer=memory,
)

config = {"configurable": {"thread_id": "session-001"}}
result = agent.invoke({"messages": [("user", "こんにちは")]}, config)
```

---

## 10. よくあるエラーと対処法

| エラー | 原因 | 対処法 |
|---|---|---|
| `AuthenticationError` | APIキーが無効 | 環境変数に正しいキーを設定 |
| `ModuleNotFoundError: serpapi` | pip 不足 | `pip install google-search-results` |
| `ImportError: AgentExecutor` | LangChain 1.0+で移動 | `pip install langchain-classic` |
| `ImportError: create_agent` | LangChain バージョン問題 | `pip install "langchain>=1.0"` を確認 |
| `RateLimitError` | API呼び出し上限超過 | 時間をおいて再実行 |
| `InvalidRequestError` | モデル名の誤り | `gpt-4o-mini` 等の有効なモデル名を指定 |
| `OutputParserException` (旧式) | LLM が Thought/Action 形式で出力しなかった | `handle_parsing_errors=True` を設定 |
| プロンプト取得失敗 | LangSmith接続不可 | `LANGSMITH_API_KEY` 確認 / フォールバック利用 |

---

## 11. 参考リンク

- [LangChain 公式ドキュメント](https://python.langchain.com/)
- [LangChain Agents ドキュメント](https://docs.langchain.com/oss/python/langchain/agents)
- [LangGraph 公式ドキュメント](https://langchain-ai.github.io/langgraph/)
- [LangChain v1 移行ガイド](https://docs.langchain.com/oss/python/migrate/langchain-v1)
- [Agent Middleware ブログ](https://blog.langchain.com/agent-middleware/)
- [LangChain / LangGraph 1.0 リリースブログ](https://blog.langchain.com/langchain-langgraph-1dot0/)
- [LangSmith](https://smith.langchain.com/)
- [ReAct 論文（2023）](https://arxiv.org/abs/2210.03629)
- [SerpAPI](https://serpapi.com/)
- [create_agent API リファレンス](https://reference.langchain.com/python/langchain/agents/factory/create_agent)
