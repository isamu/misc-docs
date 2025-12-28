# 実践例とコードテンプレート

## 概要

このドキュメントでは、これまでの内容を統合した**実際に動作するコード例**を提供します。

各例はそのままコピーして使えるように設計されています。

---

## 🎯 例1: 最小限のClaude風エージェント

最もシンプルな実装です。

### minimal_agent.py

```python
"""
最小限のClaude風エージェント

依存関係:
pip install langgraph langchain-anthropic
"""

from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END, add_messages
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import HumanMessage, SystemMessage
import os

# === State定義 ===

class MinimalState(TypedDict):
    messages: Annotated[list, add_messages]
    goal: str
    status: str  # 'working' | 'done'
    iteration: int

# === LLM初期化 ===

llm = ChatAnthropic(
    model="claude-sonnet-4-5-20251101",
    api_key=os.getenv("ANTHROPIC_API_KEY"),
    temperature=0
)

SYSTEM_PROMPT = """
あなたは有益で誠実なアシスタントです。

原則:
1. ユーザーの目標達成を支援する
2. 不確実な情報は明示する
3. 有害な出力を避ける

タスクを1ステップずつ実行してください。
各ステップの後、目標が達成されたか判断してください。
"""

# === ノード定義 ===

def think_node(state: MinimalState) -> MinimalState:
    """思考ノード"""
    print(f"\n[Iteration {state['iteration']}] Thinking...")

    messages = [
        SystemMessage(content=SYSTEM_PROMPT),
        *state["messages"],
        HumanMessage(content=f"""
<goal>{state['goal']}</goal>

次に何をすべきか考えてください。
目標が達成されている場合は「DONE」と明示してください。

回答形式:
<next_action>
行うべきこと、または「DONE」
</next_action>
<reasoning>
理由
</reasoning>
""")
    ]

    response = llm.invoke(messages)

    # DONEチェック
    if "DONE" in response.content:
        return {
            **state,
            "status": "done",
            "messages": state["messages"] + [response]
        }

    return {
        **state,
        "iteration": state["iteration"] + 1,
        "messages": state["messages"] + [response]
    }

def should_continue(state: MinimalState) -> str:
    """継続判定"""
    if state["status"] == "done":
        return "end"

    if state["iteration"] >= 10:
        print("Max iterations reached")
        return "end"

    return "continue"

# === グラフ構築 ===

workflow = StateGraph(MinimalState)

workflow.add_node("think", think_node)

workflow.set_entry_point("think")

workflow.add_conditional_edges(
    "think",
    should_continue,
    {
        "continue": "think",
        "end": END
    }
)

app = workflow.compile()

# === 実行 ===

def run_agent(goal: str):
    """エージェントを実行"""
    initial_state = {
        "messages": [],
        "goal": goal,
        "status": "working",
        "iteration": 0
    }

    print(f"🎯 Goal: {goal}\n")
    print("=" * 60)

    result = app.invoke(initial_state)

    print("\n" + "=" * 60)
    print(f"✅ Completed in {result['iteration']} iterations")
    print("\nFinal messages:")

    for msg in result["messages"]:
        role = msg.__class__.__name__
        content = msg.content[:200] + "..." if len(msg.content) > 200 else msg.content
        print(f"\n[{role}]\n{content}")

if __name__ == "__main__":
    run_agent("Calculate 15 * 23 and explain the process")
```

### 実行

```bash
export ANTHROPIC_API_KEY="your-key-here"
python minimal_agent.py
```

---

## 🔧 例2: ツール統合エージェント

ツールを使えるエージェントです。

### tool_agent.py

```python
"""
ツール統合エージェント

追加の依存関係:
pip install requests beautifulsoup4
"""

from typing import TypedDict, Annotated, Literal
from langgraph.graph import StateGraph, END, add_messages
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import HumanMessage, SystemMessage, ToolMessage
from langchain_core.tools import tool
import requests
from bs4 import BeautifulSoup
import json

# === ツール定義 ===

@tool
def web_search(query: str) -> str:
    """
    Web検索を実行します。

    Args:
        query: 検索クエリ
    """
    # DuckDuckGo Instant Answer API（無料）
    url = "https://api.duckduckgo.com/"
    params = {"q": query, "format": "json"}

    try:
        response = requests.get(url, params=params, timeout=10)
        data = response.json()

        # 結果を整形
        results = []

        if data.get("AbstractText"):
            results.append(f"Summary: {data['AbstractText']}")

        if data.get("RelatedTopics"):
            results.append("\nRelated:")
            for topic in data["RelatedTopics"][:3]:
                if "Text" in topic:
                    results.append(f"- {topic['Text']}")

        return "\n".join(results) if results else "No results found"

    except Exception as e:
        return f"Search failed: {str(e)}"

@tool
def fetch_url(url: str) -> str:
    """
    URLのコンテンツを取得します。

    Args:
        url: 取得するURL
    """
    try:
        response = requests.get(url, timeout=10)
        soup = BeautifulSoup(response.text, 'html.parser')

        # テキストを抽出
        text = soup.get_text(separator='\n', strip=True)

        # 最初の2000文字のみ
        return text[:2000]

    except Exception as e:
        return f"Failed to fetch URL: {str(e)}"

@tool
def calculate(expression: str) -> str:
    """
    数式を計算します。

    Args:
        expression: 計算式（例: "2 + 2"）
    """
    try:
        # 安全な評価
        result = eval(expression, {"__builtins__": {}}, {})
        return str(result)
    except Exception as e:
        return f"Calculation failed: {str(e)}"

# ツールリスト
tools = [web_search, fetch_url, calculate]

# === State定義 ===

class ToolAgentState(TypedDict):
    messages: Annotated[list, add_messages]
    goal: str
    status: Literal['planning', 'acting', 'done']

# === LLM初期化（ツールバインド） ===

llm = ChatAnthropic(
    model="claude-sonnet-4-5-20251101",
    temperature=0
).bind_tools(tools)

SYSTEM_PROMPT = """
あなたは有益なアシスタントです。

利用可能なツール:
- web_search: Web検索
- fetch_url: URLの内容取得
- calculate: 計算

原則:
1. まず内部知識で回答できるか検討
2. 必要な場合のみツールを使用
3. ツール結果を検証してから回答
"""

# === ノード定義 ===

def agent_node(state: ToolAgentState) -> ToolAgentState:
    """エージェントノード"""
    messages = [
        SystemMessage(content=SYSTEM_PROMPT),
        *state["messages"]
    ]

    response = llm.invoke(messages)

    return {
        **state,
        "messages": state["messages"] + [response]
    }

def tool_node(state: ToolAgentState) -> ToolAgentState:
    """ツール実行ノード"""
    last_message = state["messages"][-1]

    tool_calls = last_message.tool_calls

    tool_messages = []

    for tool_call in tool_calls:
        tool_name = tool_call["name"]
        tool_args = tool_call["args"]

        print(f"\n🔧 Calling tool: {tool_name}")
        print(f"   Args: {tool_args}")

        # ツール実行
        selected_tool = {t.name: t for t in tools}[tool_name]
        result = selected_tool.invoke(tool_args)

        print(f"   Result: {result[:100]}...")

        tool_messages.append(
            ToolMessage(
                content=str(result),
                tool_call_id=tool_call["id"]
            )
        )

    return {
        **state,
        "messages": state["messages"] + tool_messages
    }

def should_continue(state: ToolAgentState) -> str:
    """継続判定"""
    last_message = state["messages"][-1]

    # ツール呼び出しがあれば実行
    if hasattr(last_message, 'tool_calls') and last_message.tool_calls:
        return "tools"

    # なければ終了
    return "end"

# === グラフ構築 ===

workflow = StateGraph(ToolAgentState)

workflow.add_node("agent", agent_node)
workflow.add_node("tools", tool_node)

workflow.set_entry_point("agent")

workflow.add_conditional_edges(
    "agent",
    should_continue,
    {
        "tools": "tools",
        "end": END
    }
)

workflow.add_edge("tools", "agent")

app = workflow.compile()

# === 実行 ===

def run_tool_agent(goal: str):
    """ツールエージェントを実行"""
    initial_state = {
        "messages": [HumanMessage(content=goal)],
        "goal": goal,
        "status": "planning"
    }

    print(f"🎯 Goal: {goal}\n")
    print("=" * 60)

    result = app.invoke(initial_state)

    print("\n" + "=" * 60)
    print("✅ Completed\n")

    # 最終回答を抽出
    for msg in reversed(result["messages"]):
        if hasattr(msg, 'content') and msg.content and not hasattr(msg, 'tool_calls'):
            print(f"Answer:\n{msg.content}")
            break

if __name__ == "__main__":
    # 例1: 計算
    run_tool_agent("What is 456 * 789?")

    print("\n\n")

    # 例2: Web検索
    run_tool_agent("What is LangGraph?")
```

---

## 📚 例3: RAG統合エージェント

ドキュメント検索を使うエージェントです。

### rag_agent.py

```python
"""
RAG統合エージェント

追加の依存関係:
pip install chromadb sentence-transformers
"""

from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END, add_messages
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import HumanMessage, SystemMessage
import chromadb
from chromadb.utils import embedding_functions

# === RAGシステム ===

class SimpleRAG:
    def __init__(self, collection_name: str = "knowledge"):
        # ChromaDBクライアント
        self.client = chromadb.Client()

        # 埋め込み関数
        self.embedding_fn = embedding_functions.SentenceTransformerEmbeddingFunction(
            model_name="all-MiniLM-L6-v2"
        )

        # コレクション作成
        self.collection = self.client.create_collection(
            name=collection_name,
            embedding_function=self.embedding_fn,
            get_or_create=True
        )

    def add_documents(self, documents: list[dict]):
        """ドキュメントを追加"""
        self.collection.add(
            ids=[doc["id"] for doc in documents],
            documents=[doc["text"] for doc in documents],
            metadatas=[doc.get("metadata", {}) for doc in documents]
        )

        print(f"✅ Added {len(documents)} documents")

    def search(self, query: str, n_results: int = 3) -> list[dict]:
        """検索"""
        results = self.collection.query(
            query_texts=[query],
            n_results=n_results
        )

        return [
            {
                "text": doc,
                "metadata": meta
            }
            for doc, meta in zip(
                results["documents"][0],
                results["metadatas"][0]
            )
        ]

# === State定義 ===

class RAGAgentState(TypedDict):
    messages: Annotated[list, add_messages]
    query: str
    retrieved_docs: list[dict]
    status: str

# === ノード定義 ===

def retrieve_node(state: RAGAgentState, rag: SimpleRAG) -> RAGAgentState:
    """検索ノード"""
    print("\n🔍 Retrieving documents...")

    docs = rag.search(state["query"], n_results=3)

    print(f"   Found {len(docs)} relevant documents")

    return {
        **state,
        "retrieved_docs": docs
    }

def generate_node(state: RAGAgentState, llm) -> RAGAgentState:
    """生成ノード"""
    print("\n🤖 Generating answer...")

    # コンテキスト構築
    context = "\n\n".join([
        f"<document>\n{doc['text']}\n</document>"
        for doc in state["retrieved_docs"]
    ])

    prompt = f"""
<context>
{context}
</context>

<query>
{state['query']}
</query>

上記のコンテキストに基づいて、質問に回答してください。
コンテキストに情報がない場合は、その旨を伝えてください。
"""

    messages = [
        SystemMessage(content="あなたは提供されたコンテキストに基づいて回答するアシスタントです。"),
        HumanMessage(content=prompt)
    ]

    response = llm.invoke(messages)

    return {
        **state,
        "messages": state["messages"] + [response],
        "status": "done"
    }

# === グラフ構築 ===

def create_rag_agent(rag: SimpleRAG):
    """RAGエージェントを作成"""
    llm = ChatAnthropic(model="claude-sonnet-4-5-20251101", temperature=0)

    workflow = StateGraph(RAGAgentState)

    workflow.add_node("retrieve", lambda s: retrieve_node(s, rag))
    workflow.add_node("generate", lambda s: generate_node(s, llm))

    workflow.set_entry_point("retrieve")
    workflow.add_edge("retrieve", "generate")
    workflow.add_edge("generate", END)

    return workflow.compile()

# === 実行 ===

def main():
    # RAGシステム初期化
    rag = SimpleRAG()

    # サンプルドキュメントを追加
    sample_docs = [
        {
            "id": "doc1",
            "text": "LangGraphは、LangChainチームが開発したステートフルなマルチアクターアプリケーションを構築するためのライブラリです。グラフベースの実行モデルを使用します。",
            "metadata": {"source": "docs", "topic": "langgraph"}
        },
        {
            "id": "doc2",
            "text": "Model Context Protocol (MCP)は、Anthropicが開発したLLMアプリケーションとデータソースを接続するための標準プロトコルです。",
            "metadata": {"source": "docs", "topic": "mcp"}
        },
        {
            "id": "doc3",
            "text": "Claude 3.5 Sonnetは、Anthropicの最新AIモデルで、コーディング、推論、視覚処理に優れています。200Kトークンのコンテキストウィンドウを持ちます。",
            "metadata": {"source": "docs", "topic": "claude"}
        }
    ]

    rag.add_documents(sample_docs)

    # エージェント作成
    app = create_rag_agent(rag)

    # 質問
    queries = [
        "LangGraphとは何ですか？",
        "MCPの目的は？",
        "Claudeのコンテキストウィンドウは？"
    ]

    for query in queries:
        print(f"\n{'='*60}")
        print(f"❓ Query: {query}")
        print('='*60)

        result = app.invoke({
            "messages": [],
            "query": query,
            "retrieved_docs": [],
            "status": "searching"
        })

        # 回答を表示
        answer = result["messages"][-1].content
        print(f"\n💡 Answer:\n{answer}")

if __name__ == "__main__":
    main()
```

---

## 🎬 例4: マルチモーダルエージェント

画像・ドキュメントを扱うエージェントです。

### multimodal_agent.py

```python
"""
マルチモーダルエージェント

追加の依存関係:
pip install anthropic pillow
"""

import anthropic
import base64
from pathlib import Path
from typing import TypedDict
import json

class MultimodalAgent:
    def __init__(self, api_key: str):
        self.client = anthropic.Anthropic(api_key=api_key)

    def encode_image(self, image_path: str) -> dict:
        """画像をBase64エンコード"""
        with open(image_path, "rb") as f:
            image_data = base64.standard_b64encode(f.read()).decode("utf-8")

        # MIMEタイプ判定
        suffix = Path(image_path).suffix.lower()
        mime_types = {
            ".jpg": "image/jpeg",
            ".jpeg": "image/jpeg",
            ".png": "image/png",
            ".gif": "image/gif",
            ".webp": "image/webp"
        }

        return {
            "type": "image",
            "source": {
                "type": "base64",
                "media_type": mime_types.get(suffix, "image/jpeg"),
                "data": image_data
            }
        }

    def analyze_image(self, image_path: str, query: str) -> str:
        """画像を分析"""
        print(f"\n🖼️  Analyzing image: {image_path}")
        print(f"   Query: {query}")

        image_content = self.encode_image(image_path)

        message = self.client.messages.create(
            model="claude-sonnet-4-5-20251101",
            max_tokens=1024,
            messages=[
                {
                    "role": "user",
                    "content": [
                        image_content,
                        {
                            "type": "text",
                            "text": query
                        }
                    ]
                }
            ]
        )

        return message.content[0].text

    def analyze_document_with_images(
        self,
        images: list[str],
        query: str
    ) -> str:
        """複数画像を含むドキュメントを分析"""
        print(f"\n📄 Analyzing document with {len(images)} images")

        content = []

        # すべての画像を追加
        for img_path in images:
            content.append(self.encode_image(img_path))

        # クエリを追加
        content.append({
            "type": "text",
            "text": f"""
以下の画像は1つのドキュメントから抽出されたものです。

質問: {query}

すべての画像を参照して、包括的に回答してください。
"""
        })

        message = self.client.messages.create(
            model="claude-sonnet-4-5-20251101",
            max_tokens=2048,
            messages=[
                {
                    "role": "user",
                    "content": content
                }
            ]
        )

        return message.content[0].text

    def compare_images(self, image1: str, image2: str) -> str:
        """2つの画像を比較"""
        print(f"\n🔍 Comparing images:")
        print(f"   Image 1: {image1}")
        print(f"   Image 2: {image2}")

        content = [
            self.encode_image(image1),
            self.encode_image(image2),
            {
                "type": "text",
                "text": "これら2つの画像を比較して、違いと共通点を説明してください。"
            }
        ]

        message = self.client.messages.create(
            model="claude-sonnet-4-5-20251101",
            max_tokens=1024,
            messages=[
                {
                    "role": "user",
                    "content": content
                }
            ]
        )

        return message.content[0].text

# === 使用例 ===

def main():
    import os

    agent = MultimodalAgent(api_key=os.getenv("ANTHROPIC_API_KEY"))

    # 例1: 単一画像分析
    # result = agent.analyze_image(
    #     "path/to/image.jpg",
    #     "この画像に何が写っていますか？"
    # )
    # print(f"\n回答:\n{result}")

    # 例2: 複数画像分析
    # result = agent.analyze_document_with_images(
    #     ["page1.jpg", "page2.jpg", "page3.jpg"],
    #     "このドキュメントの主なポイントは何ですか？"
    # )
    # print(f"\n回答:\n{result}")

    # 例3: 画像比較
    # result = agent.compare_images(
    #     "before.jpg",
    #     "after.jpg"
    # )
    # print(f"\n比較結果:\n{result}")

    print("✅ Multimodal agent ready")
    print("   Uncomment examples in main() to test")

if __name__ == "__main__":
    main()
```

---

## 🧪 例5: デバッグとモニタリング

エージェントの動作を可視化します。

### debug_tools.py

```python
"""
デバッグとモニタリングツール
"""

import time
import json
from typing import Any
from functools import wraps

class AgentMonitor:
    def __init__(self):
        self.logs = []
        self.start_time = None

    def start(self):
        """モニタリング開始"""
        self.start_time = time.time()
        self.logs = []

    def log(self, event_type: str, data: Any):
        """イベントをログ"""
        self.logs.append({
            "timestamp": time.time() - self.start_time,
            "type": event_type,
            "data": data
        })

    def print_summary(self):
        """サマリーを表示"""
        print("\n" + "="*60)
        print("📊 Agent Execution Summary")
        print("="*60)

        # 実行時間
        total_time = self.logs[-1]["timestamp"] if self.logs else 0
        print(f"\n⏱️  Total time: {total_time:.2f}s")

        # イベント数
        event_counts = {}
        for log in self.logs:
            event_type = log["type"]
            event_counts[event_type] = event_counts.get(event_type, 0) + 1

        print(f"\n📈 Events:")
        for event_type, count in event_counts.items():
            print(f"   {event_type}: {count}")

        # トークン使用量（もしあれば）
        total_tokens = sum(
            log["data"].get("tokens", 0)
            for log in self.logs
            if isinstance(log["data"], dict)
        )

        if total_tokens > 0:
            print(f"\n🎫 Total tokens: {total_tokens:,}")

    def save_logs(self, filepath: str):
        """ログをファイルに保存"""
        with open(filepath, 'w') as f:
            json.dump(self.logs, f, indent=2)

        print(f"💾 Logs saved to {filepath}")

# デコレータ

def monitor_node(monitor: AgentMonitor):
    """ノード実行をモニター"""
    def decorator(func):
        @wraps(func)
        def wrapper(state):
            node_name = func.__name__

            # 開始ログ
            monitor.log("node_start", {
                "node": node_name,
                "state_keys": list(state.keys())
            })

            start = time.time()

            # 実行
            result = func(state)

            # 終了ログ
            duration = time.time() - start
            monitor.log("node_end", {
                "node": node_name,
                "duration": duration
            })

            print(f"   [{node_name}] {duration:.2f}s")

            return result

        return wrapper
    return decorator

# === 使用例 ===

monitor = AgentMonitor()

@monitor_node(monitor)
def example_node(state):
    """例ノード"""
    time.sleep(0.5)  # 処理のシミュレーション
    return state

# 実行
monitor.start()

state = {"test": "data"}

for i in range(3):
    state = example_node(state)

monitor.print_summary()
monitor.save_logs("agent_logs.json")
```

---

## 📝 テンプレート: カスタムエージェント

自分専用のエージェントを作る際のテンプレートです。

### custom_agent_template.py

```python
"""
カスタムエージェントテンプレート

このテンプレートをコピーして、独自のエージェントを作成してください。
"""

from typing import TypedDict, Annotated, Literal
from langgraph.graph import StateGraph, END, add_messages
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import HumanMessage, SystemMessage

# ===========================
# 1. State定義
# ===========================

class CustomState(TypedDict):
    """
    エージェントの状態を定義

    ここに必要なフィールドを追加してください
    """
    messages: Annotated[list, add_messages]

    # カスタムフィールド（例）
    goal: str
    current_step: int
    max_steps: int
    status: Literal['working', 'done', 'failed']

    # 以下に追加...

# ===========================
# 2. システムプロンプト
# ===========================

SYSTEM_PROMPT = """
あなたの役割とルールをここに記述してください。

例:
あなたは〇〇を支援するアシスタントです。

原則:
1. ...
2. ...
3. ...
"""

# ===========================
# 3. LLM初期化
# ===========================

llm = ChatAnthropic(
    model="claude-sonnet-4-5-20251101",  # または他のモデル
    temperature=0,  # 必要に応じて調整
    max_tokens=4096
)

# ===========================
# 4. ノード定義
# ===========================

def my_node_1(state: CustomState) -> CustomState:
    """
    最初のノード

    ここで何をするか説明してください
    """
    print(f"\n[Node 1] Step {state['current_step']}")

    # 処理をここに実装
    # ...

    return {
        **state,
        "current_step": state["current_step"] + 1
    }

def my_node_2(state: CustomState) -> CustomState:
    """
    2番目のノード
    """
    print(f"\n[Node 2] Processing...")

    # LLM呼び出しの例
    messages = [
        SystemMessage(content=SYSTEM_PROMPT),
        *state["messages"],
        HumanMessage(content="...")  # プロンプトを構築
    ]

    response = llm.invoke(messages)

    return {
        **state,
        "messages": state["messages"] + [response]
    }

# ===========================
# 5. ルーティング関数
# ===========================

def should_continue(state: CustomState) -> str:
    """
    次にどのノードに進むか決定
    """
    if state["status"] == "done":
        return "end"

    if state["current_step"] >= state["max_steps"]:
        return "end"

    # 条件に応じてルーティング
    # ...

    return "continue"

# ===========================
# 6. グラフ構築
# ===========================

workflow = StateGraph(CustomState)

# ノードを追加
workflow.add_node("node1", my_node_1)
workflow.add_node("node2", my_node_2)
# ... 他のノードを追加

# エントリーポイント
workflow.set_entry_point("node1")

# エッジを定義
workflow.add_edge("node1", "node2")

# 条件付きエッジ
workflow.add_conditional_edges(
    "node2",
    should_continue,
    {
        "continue": "node1",  # ループ
        "end": END
    }
)

# コンパイル
app = workflow.compile()

# ===========================
# 7. 実行関数
# ===========================

def run(goal: str, max_steps: int = 10):
    """
    エージェントを実行

    Args:
        goal: 達成したい目標
        max_steps: 最大ステップ数
    """
    initial_state: CustomState = {
        "messages": [],
        "goal": goal,
        "current_step": 0,
        "max_steps": max_steps,
        "status": "working"
    }

    print(f"🚀 Starting agent")
    print(f"   Goal: {goal}")
    print(f"   Max steps: {max_steps}")
    print("="*60)

    result = app.invoke(initial_state)

    print("\n" + "="*60)
    print(f"✅ Completed")
    print(f"   Steps: {result['current_step']}")
    print(f"   Status: {result['status']}")

    return result

# ===========================
# 8. メイン
# ===========================

if __name__ == "__main__":
    # テスト実行
    result = run(
        goal="Your goal here",
        max_steps=5
    )
```

---

## 📚 参考資料

### すべての例で使用したライブラリ

```bash
pip install langgraph langchain-anthropic anthropic
pip install chromadb sentence-transformers
pip install requests beautifulsoup4
pip install pillow
```

### 関連ドキュメント

- [01-claude-design-philosophy.md](./01-claude-design-philosophy.md) - 設計思想
- [02-context-engineering.md](./02-context-engineering.md) - Context Engineering
- [03-agent-architecture.md](./03-agent-architecture.md) - アーキテクチャ
- [04-implementation-guide.md](./04-implementation-guide.md) - 実装ガイド
- [05-multimodal-implementation.md](./05-multimodal-implementation.md) - マルチモーダル

---

## 🎯 次のステップ

1. **最小限の例から始める**: `minimal_agent.py` を実行
2. **ツールを追加**: `tool_agent.py` でツール統合を学ぶ
3. **RAGを統合**: `rag_agent.py` で知識ベース統合
4. **カスタマイズ**: テンプレートを使って独自エージェントを作成

---

これで、Claude風システムを実装するための完全なガイドが完成しました！
