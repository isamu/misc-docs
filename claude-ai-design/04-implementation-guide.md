# Claude風システム実装ガイド

## 概要

このドキュメントでは、OSSツールを使ってClaude風のエージェントシステムを実装する具体的な手順を解説します。

---

## 🛠️ 必要なツールスタック

### 推奨構成

| カテゴリ | ツール | 用途 | 理由 |
|---------|-------|------|------|
| **エージェント** | LangGraph | 状態機械・実行ループ | Anthropic推奨 |
| **LLM接続** | LangChain | LLM抽象化 | エコシステムが豊富 |
| **ツール接続** | Model Context Protocol (MCP) | 統一インターフェース | Anthropic公式 |
| **ベクトルDB** | Chroma / Qdrant | セマンティック検索 | 軽量・高速 |
| **推論サーバー** | vLLM / Ollama | ローカルLLM | 低コスト |

### インストール

```bash
# Python環境（推奨: 3.11+）
pip install langgraph langchain langchain-anthropic
pip install chromadb qdrant-client
pip install tiktoken  # トークンカウント

# Model Context Protocol SDK
npm install @modelcontextprotocol/sdk

# ドキュメント処理
pip install docling unstructured

# 動画処理（オプション）
pip install llava-next qwen-vl
```

---

## 🏗️ ステップ1: Stateスキーマの定義

LangGraphのStateGraphを使います。

### state.py

```python
from typing import TypedDict, List, Annotated
from langgraph.graph import add_messages

class Fact(TypedDict):
    content: str
    source: str  # 'user' | 'tool' | 'inference'
    confidence: str  # 'high' | 'medium' | 'low'
    timestamp: float

class Question(TypedDict):
    question: str
    priority: str  # 'high' | 'medium' | 'low'
    blocks_progress: bool

class PlanStep(TypedDict):
    id: str
    description: str
    status: str  # 'pending' | 'in_progress' | 'completed' | 'failed'
    dependencies: List[str]
    result: str | None

class AgentState(TypedDict):
    # 目標
    goal: str
    original_goal: str

    # 知識
    known_facts: List[Fact]
    open_questions: List[Question]

    # 計画
    plan_steps: List[PlanStep]
    current_step_index: int

    # 会話履歴（LangGraphの組み込み）
    messages: Annotated[list, add_messages]

    # 観測
    observations: List[dict]

    # メタ情報
    iteration_count: int
    status: str  # 'planning' | 'executing' | 'blocked' | 'completed'
```

---

## 🔄 ステップ2: Plan-Act-Observe-Reflectの実装

LangGraphでノードとエッジを定義します。

### agent.py

```python
from langgraph.graph import StateGraph, END
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage

# LLMの初期化
llm = ChatAnthropic(
    model="claude-opus-4-5-20251101",
    temperature=0
)

# ノード定義

def plan_node(state: AgentState) -> AgentState:
    """計画フェーズ"""
    print(f"[PLAN] Planning for goal: {state['goal']}")

    # Context Builderでコンテキスト構築
    context = build_planning_context(state)

    # LLMに計画を依頼
    messages = [
        SystemMessage(content=SYSTEM_PROMPT),
        HumanMessage(content=context)
    ]

    response = llm.invoke(messages)
    plan_steps = parse_plan(response.content)

    return {
        **state,
        "plan_steps": plan_steps,
        "current_step_index": 0,
        "status": "executing",
        "messages": state["messages"] + [response]
    }

def act_node(state: AgentState) -> AgentState:
    """実行フェーズ"""
    current_step = state["plan_steps"][state["current_step_index"]]
    print(f"[ACT] Executing step: {current_step['description']}")

    # ツール必要性の判断
    context = build_action_context(state, current_step)

    messages = [
        SystemMessage(content=SYSTEM_PROMPT),
        HumanMessage(content=context)
    ]

    response = llm.invoke(messages)
    action = parse_action(response.content)

    # アクション実行
    new_state = state.copy()

    if action["type"] == "use_tool":
        # ツール実行
        tool_result = execute_tool(action["tool"], action["params"])

        new_state["observations"].append({
            "source": action["tool"],
            "content": tool_result,
            "timestamp": time.time()
        })

    new_state["messages"] = state["messages"] + [response]

    return new_state

def observe_node(state: AgentState) -> AgentState:
    """観察フェーズ"""
    print("[OBSERVE] Analyzing observations")

    if not state["observations"]:
        return state

    latest_obs = state["observations"][-1]

    # 観測結果の解釈
    context = build_observation_context(state, latest_obs)

    messages = [
        SystemMessage(content=SYSTEM_PROMPT),
        HumanMessage(content=context)
    ]

    response = llm.invoke(messages)
    interpretation = parse_interpretation(response.content)

    # 新しい事実を追加
    new_facts = state["known_facts"] + interpretation["new_facts"]

    # ステップのステータス更新
    plan_steps = state["plan_steps"].copy()
    if interpretation["step_impact"] == "completed":
        plan_steps[state["current_step_index"]]["status"] = "completed"

    return {
        **state,
        "known_facts": new_facts,
        "plan_steps": plan_steps,
        "messages": state["messages"] + [response]
    }

def reflect_node(state: AgentState) -> AgentState:
    """反省フェーズ"""
    print("[REFLECT] Evaluating progress")

    context = build_reflection_context(state)

    messages = [
        SystemMessage(content=SYSTEM_PROMPT),
        HumanMessage(content=context)
    ]

    response = llm.invoke(messages)
    reflection = parse_reflection(response.content)

    new_state = state.copy()
    new_state["iteration_count"] += 1
    new_state["messages"] = state["messages"] + [response]

    # 次の状態を決定
    if reflection["goal_achieved"]:
        new_state["status"] = "completed"
    elif reflection["needs_replanning"]:
        new_state["status"] = "planning"
        new_state["plan_steps"] = []
    elif reflection["needs_user_input"]:
        new_state["status"] = "blocked"
    else:
        # 次のステップへ
        new_state["current_step_index"] += 1
        if new_state["current_step_index"] >= len(state["plan_steps"]):
            new_state["status"] = "completed"

    return new_state

# ルーティング関数

def should_continue(state: AgentState) -> str:
    """次のノードを決定"""
    status = state["status"]

    if status == "completed":
        return "end"
    elif status == "planning":
        return "plan"
    elif status == "blocked":
        return "wait_user"
    elif status == "executing":
        return "act"
    else:
        return "end"

def after_reflect(state: AgentState) -> str:
    """Reflect後の遷移"""
    if state["status"] == "completed":
        return "end"
    elif state["status"] == "planning":
        return "plan"
    elif state["status"] == "blocked":
        return "wait_user"
    else:
        return "act"

# グラフ構築

workflow = StateGraph(AgentState)

# ノードを追加
workflow.add_node("plan", plan_node)
workflow.add_node("act", act_node)
workflow.add_node("observe", observe_node)
workflow.add_node("reflect", reflect_node)

# エントリーポイント
workflow.set_entry_point("plan")

# エッジ
workflow.add_edge("plan", "act")
workflow.add_edge("act", "observe")
workflow.add_edge("observe", "reflect")

# 条件付きエッジ
workflow.add_conditional_edges(
    "reflect",
    after_reflect,
    {
        "plan": "plan",
        "act": "act",
        "wait_user": END,  # ユーザー入力待ち
        "end": END
    }
)

# コンパイル
app = workflow.compile()
```

---

## 🧠 ステップ3: Context Builderの実装

### context_builder.py

```python
from typing import List, Dict
import tiktoken

class ContextBuilder:
    def __init__(self, max_tokens: int = 180000):
        self.max_tokens = max_tokens
        self.encoder = tiktoken.get_encoding("o200k_base")  # Claude用

    def build_planning_context(self, state: AgentState) -> str:
        """計画フェーズ用コンテキスト"""
        context = f"""
<goal>
{state['goal']}
</goal>

<known_facts>
{self._format_facts(state['known_facts'])}
</known_facts>

<open_questions>
{self._format_questions(state['open_questions'])}
</open_questions>

タスク: 上記の目標を達成するための詳細な計画を立ててください。

計画形式:
<plan>
  <strategy>全体戦略</strategy>
  <steps>
    <step id="1">ステップ1</step>
    <step id="2" depends_on="1">ステップ2</step>
  </steps>
</plan>
"""
        return self._compress_if_needed(context)

    def build_action_context(
        self,
        state: AgentState,
        current_step: PlanStep
    ) -> str:
        """実行フェーズ用コンテキスト"""
        context = f"""
<current_step>
{current_step['description']}
</current_step>

<recent_observations>
{self._format_recent_observations(state['observations'], max_count=5)}
</recent_observations>

<available_tools>
{self._format_tools(AVAILABLE_TOOLS)}
</available_tools>

タスク: このステップを実行するアクションを選択してください。

以下の形式で回答:
<action type="use_tool|no_tool|ask_user">
  <tool>ツール名（use_toolの場合）</tool>
  <params>パラメータJSON</params>
  <reason>理由</reason>
</action>
"""
        return self._compress_if_needed(context)

    def _format_facts(self, facts: List[Fact]) -> str:
        """事実をXML形式で整形"""
        if not facts:
            return "<none />"

        return "\n".join([
            f'<fact source="{f["source"]}" confidence="{f["confidence"]}">'
            f'{f["content"]}'
            f'</fact>'
            for f in facts
        ])

    def _format_observations(
        self,
        observations: List[dict],
        max_count: int = 10
    ) -> str:
        """観測をXML形式で整形"""
        recent = observations[-max_count:]

        if not recent:
            return "<none />"

        return "\n".join([
            f'<observation source="{o["source"]}">'
            f'{o["content"]}'
            f'</observation>'
            for o in recent
        ])

    def _compress_if_needed(self, context: str) -> str:
        """必要に応じて圧縮"""
        token_count = len(self.encoder.encode(context))

        if token_count > self.max_tokens * 0.8:
            # 圧縮戦略を適用
            # 例: 古い観測を削除、要約など
            pass

        return context

    def count_tokens(self, text: str) -> int:
        """トークン数をカウント"""
        return len(self.encoder.encode(text))
```

---

## 🔌 ステップ4: Model Context Protocol (MCP)統合

MCPを使ってツールを標準化します。

### mcp_tools.py

```python
from mcp import Server, Tool
from mcp.server.stdio import stdio_server

# MCPサーバーの定義

app = Server("claude-style-agent")

@app.list_tools()
async def list_tools() -> list[Tool]:
    """利用可能なツールのリスト"""
    return [
        Tool(
            name="web_search",
            description="Search the web for information",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "Search query"
                    },
                    "max_results": {
                        "type": "number",
                        "description": "Max results",
                        "default": 5
                    }
                },
                "required": ["query"]
            }
        ),
        Tool(
            name="read_file",
            description="Read contents of a file",
            inputSchema={
                "type": "object",
                "properties": {
                    "path": {
                        "type": "string",
                        "description": "File path"
                    }
                },
                "required": ["path"]
            }
        )
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> str:
    """ツールの実行"""
    if name == "web_search":
        return await web_search(
            arguments["query"],
            arguments.get("max_results", 5)
        )
    elif name == "read_file":
        return await read_file(arguments["path"])
    else:
        raise ValueError(f"Unknown tool: {name}")

# ツール実装

async def web_search(query: str, max_results: int) -> str:
    """Web検索の実装"""
    # 実際の検索API呼び出し
    # 例: SerpAPI, Brave Search API, etc.
    results = perform_search(query, max_results)

    return json.dumps(results, ensure_ascii=False, indent=2)

async def read_file(path: str) -> str:
    """ファイル読み込みの実装"""
    try:
        with open(path, 'r', encoding='utf-8') as f:
            return f.read()
    except Exception as e:
        return f"Error reading file: {str(e)}"

# サーバー起動

if __name__ == "__main__":
    stdio_server(app)
```

### エージェントからの利用

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def execute_tool_via_mcp(tool_name: str, params: dict) -> str:
    """MCPを通じてツールを実行"""
    server_params = StdioServerParameters(
        command="python",
        args=["mcp_tools.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # ツール呼び出し
            result = await session.call_tool(tool_name, params)

            return result.content[0].text
```

---

## 📚 ステップ5: RAG統合（オプション）

外部知識を取り込みます。

### rag.py

```python
from langchain_community.document_loaders import DirectoryLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.embeddings import HuggingFaceEmbeddings
from langchain_community.vectorstores import Chroma

class RAGSystem:
    def __init__(self, persist_directory: str = "./chroma_db"):
        self.embeddings = HuggingFaceEmbeddings(
            model_name="sentence-transformers/all-MiniLM-L6-v2"
        )
        self.vectorstore = None
        self.persist_directory = persist_directory

    def index_documents(self, docs_path: str):
        """ドキュメントをインデックス"""
        # ドキュメント読み込み
        loader = DirectoryLoader(docs_path, glob="**/*.md")
        documents = loader.load()

        # チャンク分割
        text_splitter = RecursiveCharacterTextSplitter(
            chunk_size=1000,
            chunk_overlap=200
        )
        chunks = text_splitter.split_documents(documents)

        # ベクトル化
        self.vectorstore = Chroma.from_documents(
            documents=chunks,
            embedding=self.embeddings,
            persist_directory=self.persist_directory
        )

    def retrieve(self, query: str, k: int = 5) -> List[str]:
        """関連文書を取得"""
        if not self.vectorstore:
            # 永続化されたDBから読み込み
            self.vectorstore = Chroma(
                persist_directory=self.persist_directory,
                embedding_function=self.embeddings
            )

        docs = self.vectorstore.similarity_search(query, k=k)

        return [doc.page_content for doc in docs]

# エージェントへの統合

def build_context_with_rag(state: AgentState, rag: RAGSystem) -> str:
    """RAGを使ったコンテキスト構築"""
    # 関連文書を取得
    relevant_docs = rag.retrieve(state["goal"], k=3)

    context = f"""
<goal>
{state['goal']}
</goal>

<relevant_knowledge>
{chr(10).join([f'<doc>{doc}</doc>' for doc in relevant_docs])}
</relevant_knowledge>

<known_facts>
{format_facts(state['known_facts'])}
</known_facts>

...
"""
    return context
```

---

## 🎨 ステップ6: UIとモニタリング

### Streamlitダッシュボード

```python
import streamlit as st
from agent import app, AgentState

st.title("Claude-Style Agent")

# 初期状態
if "state" not in st.session_state:
    st.session_state.state = {
        "goal": "",
        "original_goal": "",
        "known_facts": [],
        "open_questions": [],
        "plan_steps": [],
        "current_step_index": 0,
        "messages": [],
        "observations": [],
        "iteration_count": 0,
        "status": "planning"
    }

# ユーザー入力
goal = st.text_area("目標を入力してください:", height=100)

if st.button("実行"):
    if goal:
        # 初期化
        st.session_state.state["goal"] = goal
        st.session_state.state["original_goal"] = goal

        # エージェント実行
        with st.spinner("実行中..."):
            result = app.invoke(st.session_state.state)

        st.session_state.state = result

# 進捗表示
if st.session_state.state["plan_steps"]:
    st.subheader("計画")
    for i, step in enumerate(st.session_state.state["plan_steps"]):
        status_icon = {
            "completed": "✅",
            "in_progress": "🔄",
            "pending": "⏳",
            "failed": "❌"
        }.get(step["status"], "❓")

        st.write(f"{status_icon} **Step {i+1}**: {step['description']}")

# 事実表示
if st.session_state.state["known_facts"]:
    st.subheader("判明した事実")
    for fact in st.session_state.state["known_facts"]:
        st.write(f"- {fact['content']} ({fact['confidence']})")

# 観測表示
if st.session_state.state["observations"]:
    st.subheader("観測結果")
    for obs in st.session_state.state["observations"][-5:]:
        with st.expander(f"📊 {obs['source']}"):
            st.code(obs['content'])
```

---

## 🧪 ステップ7: テスト

```python
import pytest
from agent import plan_node, act_node, AgentState

def test_plan_node():
    """計画ノードのテスト"""
    initial_state: AgentState = {
        "goal": "Analyze sales data for Q1 2024",
        "original_goal": "Analyze sales data for Q1 2024",
        "known_facts": [],
        "open_questions": [],
        "plan_steps": [],
        "current_step_index": 0,
        "messages": [],
        "observations": [],
        "iteration_count": 0,
        "status": "planning"
    }

    result = plan_node(initial_state)

    assert result["status"] == "executing"
    assert len(result["plan_steps"]) > 0
    assert result["plan_steps"][0]["id"] == "1"

def test_context_builder():
    """Context Builderのテスト"""
    builder = ContextBuilder(max_tokens=100000)

    state: AgentState = {
        "goal": "Test goal",
        "known_facts": [
            {
                "content": "Fact 1",
                "source": "user",
                "confidence": "high",
                "timestamp": time.time()
            }
        ],
        "open_questions": [],
        # ...
    }

    context = builder.build_planning_context(state)

    assert "<goal>Test goal</goal>" in context
    assert "<fact" in context
    assert builder.count_tokens(context) < 100000
```

---

## 📦 完全な実装例

すべてを統合したサンプル：

### main.py

```python
import asyncio
from agent import app, AgentState
from rag import RAGSystem
from context_builder import ContextBuilder

async def main():
    # RAGシステム初期化
    rag = RAGSystem()
    rag.index_documents("./knowledge_base")

    # Context Builder初期化
    context_builder = ContextBuilder(max_tokens=180000)

    # 初期状態
    initial_state: AgentState = {
        "goal": "Pythonでファイル読み込みの最適な方法を調べて、コード例を作成",
        "original_goal": "Pythonでファイル読み込みの最適な方法を調べて、コード例を作成",
        "known_facts": [],
        "open_questions": [],
        "plan_steps": [],
        "current_step_index": 0,
        "messages": [],
        "observations": [],
        "iteration_count": 0,
        "status": "planning"
    }

    # エージェント実行
    print("🚀 Starting agent...")
    final_state = app.invoke(initial_state)

    # 結果表示
    print("\n✅ Task completed!")
    print(f"Status: {final_state['status']}")
    print(f"Iterations: {final_state['iteration_count']}")

    print("\n📋 Completed Steps:")
    for step in final_state['plan_steps']:
        if step['status'] == 'completed':
            print(f"  ✓ {step['description']}")
            if step['result']:
                print(f"    → {step['result']}")

    print("\n📚 Known Facts:")
    for fact in final_state['known_facts']:
        print(f"  - {fact['content']} ({fact['confidence']})")

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 🚀 実行

```bash
# 依存関係インストール
pip install -r requirements.txt

# エージェント実行
python main.py

# ダッシュボード起動
streamlit run dashboard.py
```

---

## 📚 参考資料

### 公式ドキュメント

- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [Anthropic API Docs](https://docs.anthropic.com/)

### 関連ドキュメント

- [03-agent-architecture.md](./03-agent-architecture.md) - アーキテクチャ設計
- [05-multimodal-implementation.md](./05-multimodal-implementation.md) - マルチモーダル対応
- [06-practical-examples.md](./06-practical-examples.md) - 実践例

---

**次**: [05-multimodal-implementation.md](./05-multimodal-implementation.md) - ドキュメント・動画処理の実装
