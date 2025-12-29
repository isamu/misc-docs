# 05. 統合実装例

このドキュメントでは、モダンなContext Management（2025年版）を**実際のフレームワークやシステムと統合**する具体的な方法を示します。

---

## 目次

1. [LangGraph統合](#1-langgraph統合)
2. [GraphAI統合](#2-graphai統合)
3. [Mulmo統合](#3-mulmo統合)
4. [RAGシステム統合](#4-ragシステム統合)
5. [チェックポイントシステム統合](#5-チェックポイントシステム統合)
6. [既存システムからのマイグレーション](#6-既存システムからのマイグレーション)

---

## 1. LangGraph統合

LangGraphは状態ベースのワークフロー構築に適しています。Context ManagementをLangGraphの状態管理と統合します。

### 1.1 基本的な統合

```typescript
import { StateGraph, END } from '@langchain/langgraph'

// LangGraph用の状態定義
interface GraphState {
  // Context Management統合
  layeredState: LayeredState
  messages: Message[]
  contextBudget: BudgetAllocation

  // LangGraph固有
  currentNode: string
  nextAction: string | null
  toolResults: Record<string, any>
}

class LangGraphContextIntegration {
  private contextEngine: ModernContextEngine
  private graph: StateGraph<GraphState>

  constructor() {
    this.contextEngine = new ModernContextEngine({
      maxTokens: 200000,
      model: 'claude-opus-4'
    })

    this.graph = this.buildGraph()
  }

  private buildGraph(): StateGraph<GraphState> {
    const graph = new StateGraph<GraphState>({
      channels: {
        layeredState: null,
        messages: null,
        contextBudget: null,
        currentNode: null,
        nextAction: null,
        toolResults: null
      }
    })

    // ノード定義
    graph.addNode('initialize', this.initializeNode.bind(this))
    graph.addNode('plan', this.planNode.bind(this))
    graph.addNode('execute', this.executeNode.bind(this))
    graph.addNode('reflect', this.reflectNode.bind(this))
    graph.addNode('compress', this.compressNode.bind(this))

    // エッジ定義
    graph.addEdge('initialize', 'plan')
    graph.addConditionalEdges(
      'plan',
      this.shouldExecute.bind(this),
      {
        execute: 'execute',
        reflect: 'reflect',
        end: END
      }
    )
    graph.addConditionalEdges(
      'execute',
      this.needsCompression.bind(this),
      {
        compress: 'compress',
        reflect: 'reflect'
      }
    )
    graph.addEdge('compress', 'reflect')
    graph.addConditionalEdges(
      'reflect',
      this.shouldContinue.bind(this),
      {
        plan: 'plan',
        end: END
      }
    )

    graph.setEntryPoint('initialize')

    return graph
  }

  /**
   * 初期化ノード: Context Managementのセットアップ
   */
  private async initializeNode(state: GraphState): Promise<Partial<GraphState>> {
    // LayeredStateを初期化
    const layeredState = this.contextEngine.getState()

    // 予算配分
    const contextBudget = this.contextEngine.allocateBudget(
      200000,
      'planning',
      'claude-opus-4'
    )

    return {
      layeredState,
      contextBudget,
      currentNode: 'initialize',
      messages: []
    }
  }

  /**
   * プランニングノード: コンテキストを使って計画
   */
  private async planNode(state: GraphState): Promise<Partial<GraphState>> {
    // Phase更新
    const updatedState = {
      ...state.layeredState,
      l2_runtime: {
        ...state.layeredState.l2_runtime,
        phase: 'planning' as Phase
      }
    }

    // コンテキスト構築
    const context = await this.contextEngine.buildContext(
      updatedState,
      state.messages,
      state.contextBudget
    )

    // LLM呼び出し（プランニング用ツール投影）
    const tools = this.contextEngine.projectTools(updatedState, 'planning')

    const response = await this.callLLM({
      system: context.system,
      messages: context.messages,
      tools
    })

    // メッセージ追加
    const newMessages = [...state.messages, {
      id: crypto.randomUUID(),
      role: 'assistant' as const,
      content: response.content,
      timestamp: Date.now()
    }]

    return {
      layeredState: updatedState,
      messages: newMessages,
      currentNode: 'plan',
      nextAction: this.extractNextAction(response)
    }
  }

  /**
   * 実行ノード: ツールを実行
   */
  private async executeNode(state: GraphState): Promise<Partial<GraphState>> {
    // Phase更新
    const updatedState = {
      ...state.layeredState,
      l2_runtime: {
        ...state.layeredState.l2_runtime,
        phase: 'execution' as Phase
      }
    }

    // 実行用ツール投影（planningより多い）
    const tools = this.contextEngine.projectTools(updatedState, 'execution')

    // ツール実行
    const toolResults: Record<string, any> = {}
    const lastMessage = state.messages[state.messages.length - 1]

    for (const content of lastMessage.content) {
      if (content.type === 'tool_use') {
        const result = await this.executeTool(content.name, content.input)
        toolResults[content.id] = result

        // tool_result メッセージを追加
        state.messages.push({
          id: crypto.randomUUID(),
          role: 'user',
          content: [{
            type: 'tool_result',
            tool_use_id: content.id,
            content: JSON.stringify(result)
          }],
          timestamp: Date.now()
        })
      }
    }

    return {
      layeredState: updatedState,
      messages: state.messages,
      currentNode: 'execute',
      toolResults
    }
  }

  /**
   * 圧縮ノード: トークン数が多い場合に実行
   */
  private async compressNode(state: GraphState): Promise<Partial<GraphState>> {
    const compressed = await this.contextEngine.compress(
      state.messages,
      state.contextBudget.memory * 0.5  // 50%まで圧縮
    )

    return {
      messages: compressed.messages,
      currentNode: 'compress'
    }
  }

  /**
   * 振り返りノード: 結果を評価
   */
  private async reflectNode(state: GraphState): Promise<Partial<GraphState>> {
    const updatedState = {
      ...state.layeredState,
      l2_runtime: {
        ...state.layeredState.l2_runtime,
        phase: 'reflection' as Phase
      }
    }

    // Work Bufferを更新（実行結果を記録）
    const workBuffer = {
      ...state.layeredState.l5_workBuffer,
      executionResults: state.toolResults
    }

    const finalState = {
      ...updatedState,
      l5_workBuffer: workBuffer
    }

    return {
      layeredState: finalState,
      currentNode: 'reflect'
    }
  }

  // 条件分岐関数
  private shouldExecute(state: GraphState): string {
    if (state.nextAction === 'execute') return 'execute'
    if (state.nextAction === 'reflect') return 'reflect'
    return 'end'
  }

  private needsCompression(state: GraphState): string {
    const tokens = this.contextEngine.countTokens(state.messages)
    const threshold = state.contextBudget.memory * 0.75

    return tokens > threshold ? 'compress' : 'reflect'
  }

  private shouldContinue(state: GraphState): string {
    // タスク完了判定
    const isComplete = this.isTaskComplete(state)
    return isComplete ? 'end' : 'plan'
  }

  // ヘルパー
  private async callLLM(input: any): Promise<any> {
    // LLM呼び出しの実装
    return { content: [] }
  }

  private async executeTool(name: string, input: any): Promise<any> {
    // ツール実行の実装
    return {}
  }

  private extractNextAction(response: any): string {
    // レスポンスから次のアクションを抽出
    return 'execute'
  }

  private isTaskComplete(state: GraphState): boolean {
    // タスク完了判定
    return false
  }
}
```

### 1.2 LangGraphのCheckpointと統合

```typescript
import { MemorySaver, Checkpoint } from '@langchain/langgraph'

class LangGraphCheckpointIntegration {
  private checkpointSaver: MemorySaver
  private contextCheckpoint: CheckpointManager

  constructor() {
    this.checkpointSaver = new MemorySaver()
    this.contextCheckpoint = new CheckpointManager(storage)
  }

  /**
   * LangGraphのチェックポイント保存時に、Context Managementのチェックポイントも保存
   */
  async saveCheckpoint(
    threadId: string,
    checkpoint: Checkpoint,
    contextState: LayeredState,
    messages: Message[]
  ): Promise<void> {
    // 1. LangGraphのチェックポイント保存
    await this.checkpointSaver.put(threadId, checkpoint)

    // 2. Context Managementのチェックポイント保存
    await this.contextCheckpoint.saveCheckpoint(
      threadId,
      contextState,
      messages
    )
  }

  /**
   * 復元時は両方から復元
   */
  async restoreCheckpoint(threadId: string): Promise<{
    graphCheckpoint: Checkpoint
    contextState: LayeredState
    messages: Message[]
  }> {
    // 1. LangGraphのチェックポイント復元
    const graphCheckpoint = await this.checkpointSaver.get(threadId)

    // 2. Context Managementのチェックポイント復元
    const contextData = await this.contextCheckpoint.restore(threadId)

    return {
      graphCheckpoint,
      contextState: contextData.state,
      messages: contextData.messages
    }
  }
}
```

---

## 2. GraphAI統合

GraphAIは軽量なグラフベースのAIエージェントフレームワークです。

### 2.1 GraphAIノードとしてのContext Management

```typescript
import { GraphAI } from 'graphai'

// Context ManagementをGraphAIノードとして定義
const contextManagementNode = {
  name: 'contextManager',
  processor: async (context: any, inputs: any[]) => {
    const { userInput, state } = inputs[0]

    // Context Engine初期化
    const engine = new ModernContextEngine({
      conversationId: context.threadId,
      maxTokens: 200000,
      model: 'claude-opus-4'
    })

    // メッセージ追加
    await engine.addUserMessage(userInput)

    // 圧縮判定
    if (engine.needsCompression()) {
      await engine.compress()
    }

    // コンテキスト構築
    const contextOutput = await engine.buildContext()

    // ツール投影
    const tools = engine.projectTools()

    return {
      context: contextOutput,
      tools,
      messages: engine.getMessages(),
      state: engine.getState()
    }
  }
}

// GraphAIグラフ定義
const graph = {
  version: 0.5,
  nodes: {
    // ユーザー入力
    userInput: {
      value: 'What is the weather today?'
    },

    // Context Management
    context: {
      processor: contextManagementNode.processor,
      inputs: [
        { userInput: ':userInput', state: ':previousState' }
      ]
    },

    // LLM呼び出し
    llm: {
      processor: async (context: any, inputs: any[]) => {
        const { context: ctx, tools } = inputs[0]

        const response = await callClaude({
          system: ctx.system,
          messages: ctx.messages,
          tools
        })

        return response
      },
      inputs: [':context']
    },

    // レスポンス処理
    processResponse: {
      processor: async (context: any, inputs: any[]) => {
        const [ctxData, llmResponse] = inputs

        // メッセージ追加
        const updatedMessages = [
          ...ctxData.messages,
          {
            id: crypto.randomUUID(),
            role: 'assistant',
            content: llmResponse.content,
            timestamp: Date.now()
          }
        ]

        // Stateを更新
        const updatedState = updateStateFromResponse(
          ctxData.state,
          llmResponse
        )

        return {
          messages: updatedMessages,
          state: updatedState,
          response: llmResponse
        }
      },
      inputs: [':context', ':llm']
    }
  }
}

// GraphAI実行
const graphAI = new GraphAI(graph)
const result = await graphAI.run()
```

### 2.2 GraphAIのストリーミングと統合

```typescript
class GraphAIStreamingIntegration {
  /**
   * ストリーミングレスポンスを処理しながら、リアルタイムでContext更新
   */
  async runWithStreaming(userInput: string): Promise<void> {
    const engine = new ModernContextEngine({
      conversationId: 'stream-123',
      maxTokens: 200000,
      model: 'claude-opus-4'
    })

    // ユーザーメッセージ追加
    await engine.addUserMessage(userInput)

    // コンテキスト構築
    const context = await engine.buildContext()
    const tools = engine.projectTools()

    // ストリーミングLLM呼び出し
    const stream = await callClaudeStreaming({
      system: context.system,
      messages: context.messages,
      tools
    })

    // ストリーミング受信
    let accumulatedContent: ContentBlock[] = []

    for await (const chunk of stream) {
      if (chunk.type === 'content_block_delta') {
        // チャンク蓄積
        accumulatedContent.push(chunk.delta)

        // ユーザーに表示（リアルタイム）
        this.displayToUser(chunk.delta)
      }

      if (chunk.type === 'message_stop') {
        // 完了時にメッセージ追加
        await engine.addAssistantMessage(accumulatedContent)

        // Stateを更新
        engine.updateStateFromResponse(accumulatedContent)
      }
    }
  }

  private displayToUser(delta: any): void {
    // ユーザーに表示する実装
    process.stdout.write(delta.text || '')
  }
}
```

---

## 3. Mulmo統合

Mulmoはマルチモーダル対応のエージェントフレームワークです。

### 3.3 Mulmoのマルチモーダルコンテンツとの統合

```typescript
interface MultimodalMessage extends Message {
  content: Array<
    | TextBlock
    | ImageBlock
    | AudioBlock
    | VideoBlock
    | ToolUseBlock
    | ToolResultBlock
  >
}

interface ImageBlock {
  type: 'image'
  source: {
    type: 'base64' | 'url'
    media_type: string
    data?: string
    url?: string
  }
}

class MulmoContextIntegration {
  private engine: ModernContextEngine

  /**
   * マルチモーダルメッセージの追加
   */
  async addMultimodalMessage(
    text: string,
    images?: ImageBlock[],
    audio?: AudioBlock[]
  ): Promise<void> {
    const content: ContentBlock[] = [
      { type: 'text', text }
    ]

    if (images) {
      content.push(...images)
    }

    if (audio) {
      content.push(...audio)
    }

    const message: MultimodalMessage = {
      id: crypto.randomUUID(),
      role: 'user',
      content,
      timestamp: Date.now()
    }

    await this.engine.addMessage(message)
  }

  /**
   * 画像を含むメッセージの圧縮
   * 画像は要約テキストに置き換え
   */
  async compressMultimodal(
    messages: MultimodalMessage[]
  ): Promise<MultimodalMessage[]> {
    const toCondense = messages.slice(0, -10)
    const toKeep = messages.slice(-10)

    // 画像を含むメッセージを検出
    const imageMessages = toCondense.filter(m =>
      m.content.some(c => c.type === 'image')
    )

    // 画像を要約
    const imageSummaries = await Promise.all(
      imageMessages.map(async (msg) => {
        const imageBlocks = msg.content.filter(
          c => c.type === 'image'
        ) as ImageBlock[]

        const summaries = await Promise.all(
          imageBlocks.map(img => this.summarizeImage(img))
        )

        return {
          messageId: msg.id,
          summaries
        }
      })
    )

    // テキスト要約を作成
    const textSummary = await this.summarizeText(toCondense)

    // 画像要約を追加
    const imageSummaryText = imageSummaries
      .map(s => `Images: ${s.summaries.join(', ')}`)
      .join('\n')

    // 圧縮メッセージを作成
    const condensedMessage: MultimodalMessage = {
      id: crypto.randomUUID(),
      role: 'assistant',
      content: [
        {
          type: 'text',
          text: `## Context Summary\n\n${textSummary}\n\n${imageSummaryText}`
        }
      ],
      condenseId: crypto.randomUUID(),
      timestamp: Date.now()
    }

    return [condensedMessage, ...toKeep]
  }

  private async summarizeImage(image: ImageBlock): Promise<string> {
    // 画像をLLMで要約（Claude Vision等）
    const response = await callClaudeVision({
      image: image.source,
      prompt: 'Describe this image briefly in one sentence.'
    })

    return response.text
  }

  private async summarizeText(messages: Message[]): Promise<string> {
    // テキスト要約（通常のCondensation）
    const result = await this.engine.condense(messages)
    return result.condensedMessage?.content[0].text || ''
  }
}
```

### 3.2 Mulmoのツールチェーン統合

```typescript
class MulmoToolChainIntegration {
  /**
   * Mulmoのツールチェーンと動的ツール投影を統合
   */
  async executeToolChain(
    state: LayeredState,
    toolChain: string[]
  ): Promise<any> {
    const results: any[] = []

    for (const toolName of toolChain) {
      // 現在のphaseでツールが許可されているかチェック
      const allowedTools = this.projectTools(state)
      const tool = allowedTools.find(t => t.name === toolName)

      if (!tool) {
        throw new Error(
          `Tool ${toolName} not allowed in phase ${state.l2_runtime.phase}`
        )
      }

      // ツール実行
      const result = await this.executeTool(tool, state)
      results.push(result)

      // Stateを更新（次のツール実行に影響）
      state = this.updateStateFromToolResult(state, result)
    }

    return results
  }

  private projectTools(state: LayeredState): Tool[] {
    const projector = new DynamicToolProjector()
    return projector.project(state)
  }

  private async executeTool(tool: Tool, state: LayeredState): Promise<any> {
    // ツール実行の実装
    return {}
  }

  private updateStateFromToolResult(
    state: LayeredState,
    result: any
  ): LayeredState {
    // ツール結果からStateを更新
    return {
      ...state,
      l4_evidence: {
        ...state.l4_evidence,
        observations: [
          ...state.l4_evidence.observations,
          {
            timestamp: Date.now(),
            source: 'tool_execution',
            content: JSON.stringify(result)
          }
        ]
      }
    }
  }
}
```

---

## 4. RAGシステム統合

Retrieval-Augmented Generation（RAG）とContext Managementを統合します。

### 4.1 Evidence層へのRAG結果の統合

```typescript
interface RAGResult {
  query: string
  results: Array<{
    id: string
    content: string
    metadata: {
      source: string
      relevanceScore: number
      timestamp: number
    }
  }>
  totalResults: number
}

class RAGContextIntegration {
  private vectorStore: VectorStore
  private contextEngine: ModernContextEngine

  /**
   * ユーザークエリからRAGを実行し、Evidence層に統合
   */
  async queryWithRAG(
    userQuery: string,
    state: LayeredState
  ): Promise<LayeredState> {
    // 1. クエリの再構成（Stateを考慮）
    const rewrittenQuery = this.rewriteQuery(userQuery, state)

    // 2. ベクトル検索
    const ragResults = await this.vectorStore.search(
      rewrittenQuery,
      {
        topK: 5,
        filter: this.buildFilter(state)
      }
    )

    // 3. Evidence層に追加
    const updatedState = {
      ...state,
      l4_evidence: {
        ...state.l4_evidence,
        ragResults: ragResults.results.map(r => ({
          id: r.id,
          content: r.content,
          source: r.metadata.source,
          relevanceScore: r.metadata.relevanceScore,
          timestamp: r.metadata.timestamp
        }))
      }
    }

    return updatedState
  }

  /**
   * クエリ再構成: Stateから関連情報を抽出してクエリを改善
   */
  private rewriteQuery(query: string, state: LayeredState): string {
    // タスクコンテキストを追加
    const taskContext = state.l1_task.goal

    // 最近の会話内容から関連キーワードを抽出
    const recentContext = state.l3_memory.shortTerm
      .slice(-3)
      .join(' ')

    // LLMでクエリを再構成
    return `${query} (Context: ${taskContext}, Recent: ${recentContext})`
  }

  /**
   * フィルタ構築: 権限・環境に応じてフィルタリング
   */
  private buildFilter(state: LayeredState): Record<string, any> {
    const filter: Record<string, any> = {}

    // 権限に応じたフィルタ
    const permissions = state.l2_runtime.permissions
    if (!permissions.includes('admin')) {
      filter.accessLevel = { $in: ['public', 'user'] }
    }

    // 環境に応じたフィルタ
    if (state.l2_runtime.environment === 'production') {
      filter.environment = { $in: ['production', 'all'] }
    }

    return filter
  }

  /**
   * Evidence予算内での最適化
   */
  async optimizeEvidenceForBudget(
    ragResults: RAGResult[],
    budget: number
  ): Promise<RAGResult[]> {
    // 1. 関連度でソート
    const sorted = [...ragResults].sort(
      (a, b) => b.metadata.relevanceScore - a.metadata.relevanceScore
    )

    // 2. 予算内で詰め込む
    const selected: RAGResult[] = []
    let usedTokens = 0

    for (const result of sorted) {
      const tokens = this.countTokens(result.content)

      if (usedTokens + tokens <= budget) {
        selected.push(result)
        usedTokens += tokens
      } else {
        break
      }
    }

    return selected
  }

  private countTokens(text: string): number {
    return Math.ceil(text.length / 4)
  }
}
```

### 4.2 Hybrid Search（ベクトル + キーワード）

```typescript
class HybridSearchIntegration {
  /**
   * ベクトル検索とキーワード検索を組み合わせ
   */
  async hybridSearch(
    query: string,
    state: LayeredState,
    options: {
      vectorWeight: number  // 0.0 - 1.0
      keywordWeight: number // 0.0 - 1.0
    }
  ): Promise<RAGResult[]> {
    // 1. 並列に両方実行
    const [vectorResults, keywordResults] = await Promise.all([
      this.vectorSearch(query),
      this.keywordSearch(query)
    ])

    // 2. スコアを正規化して統合
    const combined = this.combineResults(
      vectorResults,
      keywordResults,
      options
    )

    // 3. 再ランキング（Stateを考慮）
    const reranked = this.rerankWithContext(combined, state)

    return reranked
  }

  private async vectorSearch(query: string): Promise<RAGResult[]> {
    // ベクトル検索の実装
    return []
  }

  private async keywordSearch(query: string): Promise<RAGResult[]> {
    // キーワード検索の実装（BM25等）
    return []
  }

  private combineResults(
    vectorResults: RAGResult[],
    keywordResults: RAGResult[],
    weights: { vectorWeight: number; keywordWeight: number }
  ): RAGResult[] {
    // Reciprocal Rank Fusion (RRF)
    const k = 60
    const scores = new Map<string, number>()

    // Vector results
    vectorResults.forEach((result, rank) => {
      const score = weights.vectorWeight / (k + rank + 1)
      scores.set(result.id, (scores.get(result.id) || 0) + score)
    })

    // Keyword results
    keywordResults.forEach((result, rank) => {
      const score = weights.keywordWeight / (k + rank + 1)
      scores.set(result.id, (scores.get(result.id) || 0) + score)
    })

    // Merge and sort
    const allResults = [...vectorResults, ...keywordResults]
    const unique = Array.from(
      new Map(allResults.map(r => [r.id, r])).values()
    )

    return unique.sort((a, b) => {
      const scoreA = scores.get(a.id) || 0
      const scoreB = scores.get(b.id) || 0
      return scoreB - scoreA
    })
  }

  private rerankWithContext(
    results: RAGResult[],
    state: LayeredState
  ): RAGResult[] {
    // Stateのタスク・権限・環境を考慮して再ランキング
    return results.map(result => {
      let bonus = 0

      // タスク関連性
      if (this.isTaskRelevant(result, state.l1_task)) {
        bonus += 0.2
      }

      // 新鮮さ
      const age = Date.now() - result.metadata.timestamp
      const daysSinceUpdate = age / (1000 * 60 * 60 * 24)
      if (daysSinceUpdate < 7) {
        bonus += 0.1
      }

      return {
        ...result,
        metadata: {
          ...result.metadata,
          relevanceScore: result.metadata.relevanceScore + bonus
        }
      }
    }).sort((a, b) =>
      b.metadata.relevanceScore - a.metadata.relevanceScore
    )
  }

  private isTaskRelevant(result: RAGResult, task: TaskLayer): boolean {
    // タスクのgoalと結果の関連性をチェック
    const taskKeywords = task.goal.toLowerCase().split(/\s+/)
    const resultText = result.content.toLowerCase()

    return taskKeywords.some(kw => resultText.includes(kw))
  }
}
```

---

## 5. チェックポイントシステム統合

### 5.1 Shadow Git統合

Roo Codeで使われているShadow Git方式との統合：

```typescript
import { simpleGit, SimpleGit } from 'simple-git'

class ShadowGitIntegration {
  private git: SimpleGit
  private shadowDir: string

  constructor(shadowDir: string) {
    this.shadowDir = shadowDir
    this.git = simpleGit(shadowDir)
  }

  /**
   * メッセージとStateをGitコミットとして保存
   */
  async saveCheckpoint(
    conversationId: string,
    state: LayeredState,
    messages: Message[],
    metadata: CheckpointMetadata
  ): Promise<string> {
    // 1. JSONファイルを作成
    const stateFile = `${this.shadowDir}/state.json`
    const messagesFile = `${this.shadowDir}/messages.json`
    const metadataFile = `${this.shadowDir}/metadata.json`

    await Promise.all([
      fs.writeFile(stateFile, JSON.stringify(state, null, 2)),
      fs.writeFile(messagesFile, JSON.stringify(messages, null, 2)),
      fs.writeFile(metadataFile, JSON.stringify(metadata, null, 2))
    ])

    // 2. Git add
    await this.git.add([stateFile, messagesFile, metadataFile])

    // 3. Git commit
    const commitMessage = `Checkpoint: ${metadata.phase} (${messages.length} messages, ${metadata.totalTokens} tokens)`

    await this.git.commit(commitMessage)

    // 4. コミットハッシュを取得
    const log = await this.git.log({ maxCount: 1 })
    return log.latest!.hash
  }

  /**
   * 特定のコミットから復元
   */
  async restoreCheckpoint(commitHash: string): Promise<{
    state: LayeredState
    messages: Message[]
    metadata: CheckpointMetadata
  }> {
    // 1. コミットをチェックアウト
    await this.git.checkout(commitHash)

    // 2. ファイルを読み込み
    const [state, messages, metadata] = await Promise.all([
      fs.readFile(`${this.shadowDir}/state.json`, 'utf-8').then(JSON.parse),
      fs.readFile(`${this.shadowDir}/messages.json`, 'utf-8').then(JSON.parse),
      fs.readFile(`${this.shadowDir}/metadata.json`, 'utf-8').then(JSON.parse)
    ])

    return { state, messages, metadata }
  }

  /**
   * チェックポイント一覧を取得
   */
  async listCheckpoints(limit: number = 10): Promise<CheckpointInfo[]> {
    const log = await this.git.log({ maxCount: limit })

    return log.all.map(commit => ({
      hash: commit.hash,
      message: commit.message,
      date: new Date(commit.date),
      author: commit.author_name
    }))
  }

  /**
   * ブランチを作成（実験的なパスを試す）
   */
  async createExperimentalBranch(name: string): Promise<void> {
    await this.git.checkoutLocalBranch(name)
  }

  /**
   * ブランチをマージ（実験が成功したら）
   */
  async mergeExperimentalBranch(branchName: string): Promise<void> {
    await this.git.checkout('main')
    await this.git.merge([branchName])
  }

  /**
   * 差分を確認
   */
  async diffCheckpoints(
    fromHash: string,
    toHash: string
  ): Promise<string> {
    return await this.git.diff([fromHash, toHash])
  }
}

interface CheckpointInfo {
  hash: string
  message: string
  date: Date
  author: string
}
```

### 5.2 Time-travel debugging

```typescript
class TimeTravelDebugger {
  private shadowGit: ShadowGitIntegration
  private contextEngine: ModernContextEngine

  /**
   * 特定の時点に戻って再実行
   */
  async replayFromCheckpoint(
    checkpointHash: string,
    newUserInput: string
  ): Promise<any> {
    // 1. チェックポイントから復元
    const { state, messages } = await this.shadowGit.restoreCheckpoint(
      checkpointHash
    )

    // 2. エンジンを復元状態で初期化
    this.contextEngine.setState(state)
    this.contextEngine.setMessages(messages)

    // 3. 新しい入力で再実行
    const result = await this.contextEngine.executeTask(newUserInput)

    return result
  }

  /**
   * 複数のパスを並列に試す（A/Bテスト）
   */
  async compareAlternativePaths(
    checkpointHash: string,
    alternatives: string[]
  ): Promise<ComparisonResult[]> {
    const results = await Promise.all(
      alternatives.map(async (input, idx) => {
        // 各代替パスで実験的ブランチを作成
        await this.shadowGit.createExperimentalBranch(`experiment-${idx}`)

        const result = await this.replayFromCheckpoint(checkpointHash, input)

        return {
          input,
          result,
          branch: `experiment-${idx}`
        }
      })
    )

    return results
  }

  /**
   * 最良のパスをメインブランチにマージ
   */
  async chooseBestPath(branchName: string): Promise<void> {
    await this.shadowGit.mergeExperimentalBranch(branchName)
  }
}

interface ComparisonResult {
  input: string
  result: any
  branch: string
}
```

---

## 6. 既存システムからのマイグレーション

### 6.1 シンプルな履歴管理からの移行

```typescript
class MigrationHelper {
  /**
   * 古い形式（単純な配列）から新しい形式へ
   */
  async migrateFromSimpleHistory(
    oldMessages: Array<{ role: string; content: string }>
  ): Promise<{
    state: LayeredState
    messages: Message[]
  }> {
    // 1. メッセージを新形式に変換
    const messages: Message[] = oldMessages.map((old, idx) => ({
      id: crypto.randomUUID(),
      role: old.role as 'user' | 'assistant',
      content: [
        {
          type: 'text',
          text: old.content
        }
      ],
      timestamp: Date.now() - (oldMessages.length - idx) * 60000 // 推定
    }))

    // 2. Stateを推定
    const state = this.inferStateFromMessages(messages)

    return { state, messages }
  }

  /**
   * メッセージからStateを推定
   */
  private inferStateFromMessages(messages: Message[]): LayeredState {
    // 簡易的な推定ロジック
    const recentMessages = messages.slice(-10)
    const allText = recentMessages
      .map(m => m.content.filter(c => c.type === 'text').map(c => c.text))
      .flat()
      .join(' ')

    // タスクを推定
    const goal = this.extractGoal(allText)

    // フェーズを推定
    const phase = this.inferPhase(allText)

    return {
      l0_system: {
        policies: [],
        constraints: [],
        safetyGuidelines: []
      },
      l1_task: {
        goal,
        successCriteria: [],
        constraints: [],
        outputFormat: 'text'
      },
      l2_runtime: {
        phase,
        permissions: ['read', 'write'],
        environment: 'development',
        resourceLimits: {}
      },
      l3_memory: {
        shortTerm: recentMessages.map(m =>
          m.content.filter(c => c.type === 'text').map(c => c.text).join(' ')
        ),
        episodic: [],
        semantic: {},
        procedural: []
      },
      l4_evidence: {
        observations: [],
        ragResults: [],
        citations: []
      },
      l5_workBuffer: {
        plan: [],
        currentStep: null,
        diff: [],
        hypotheses: []
      }
    }
  }

  private extractGoal(text: string): string {
    // 簡易的なゴール抽出（最初のユーザーメッセージから）
    const sentences = text.split(/[.!?]/)
    return sentences[0] || 'Assist user with their request'
  }

  private inferPhase(text: string): Phase {
    // キーワードからフェーズを推定
    if (text.includes('plan') || text.includes('how should')) {
      return 'planning'
    }
    if (text.includes('execute') || text.includes('run')) {
      return 'execution'
    }
    if (text.includes('review') || text.includes('evaluate')) {
      return 'reflection'
    }
    return 'planning'
  }

  /**
   * 段階的移行（既存システムと並行稼働）
   */
  async gradualMigration(
    conversationId: string,
    useNewSystem: boolean
  ): Promise<void> {
    if (useNewSystem) {
      // 新システムを使用
      const engine = new ModernContextEngine({
        conversationId,
        maxTokens: 200000,
        model: 'claude-opus-4'
      })

      // ... 新システムのロジック
    } else {
      // 旧システムを使用（後方互換性）
      const oldEngine = new LegacyContextManager()

      // ... 旧システムのロジック
    }
  }

  /**
   * データ整合性チェック
   */
  async validateMigration(
    oldData: any,
    newData: { state: LayeredState; messages: Message[] }
  ): Promise<ValidationResult> {
    const errors: string[] = []

    // メッセージ数の一致
    if (oldData.messages.length !== newData.messages.length) {
      errors.push('Message count mismatch')
    }

    // 内容の一致（テキストのみ）
    for (let i = 0; i < oldData.messages.length; i++) {
      const oldText = oldData.messages[i].content
      const newText = newData.messages[i].content
        .filter(c => c.type === 'text')
        .map(c => c.text)
        .join('')

      if (oldText !== newText) {
        errors.push(`Message ${i} content mismatch`)
      }
    }

    return {
      valid: errors.length === 0,
      errors
    }
  }
}

interface ValidationResult {
  valid: boolean
  errors: string[]
}
```

### 6.2 ロールバック計画

```typescript
class RollbackManager {
  /**
   * 新システムで問題が発生した場合のロールバック
   */
  async rollbackToOldSystem(
    conversationId: string,
    state: LayeredState,
    messages: Message[]
  ): Promise<any> {
    // 1. 新形式を旧形式に変換
    const oldMessages = messages.map(msg => ({
      role: msg.role,
      content: msg.content
        .filter(c => c.type === 'text')
        .map(c => c.text)
        .join('\n')
    }))

    // 2. 旧システムで保存
    await this.legacyStorage.save(conversationId, oldMessages)

    // 3. 新システムのデータをバックアップ
    await this.backupNewData(conversationId, state, messages)

    return oldMessages
  }

  private async backupNewData(
    conversationId: string,
    state: LayeredState,
    messages: Message[]
  ): Promise<void> {
    const backupPath = `./backups/${conversationId}-${Date.now()}.json`

    await fs.writeFile(
      backupPath,
      JSON.stringify({ state, messages }, null, 2)
    )
  }
}
```

---

## まとめ

この統合実装例では、以下を提供しました：

1. **LangGraph統合**: 状態ベースワークフローとContext Managementの統合、チェックポイント連携
2. **GraphAI統合**: ノードベース実装、ストリーミング対応
3. **Mulmo統合**: マルチモーダルコンテンツの圧縮、ツールチェーン統合
4. **RAGシステム統合**: Evidence層への統合、Hybrid Search、予算最適化
5. **チェックポイント統合**: Shadow Git方式、Time-travel debugging
6. **マイグレーション**: 既存システムからの移行、段階的移行、ロールバック

これらの統合パターンを参考に、あなたのシステムに最適な形でContext Managementを組み込むことができます。

---

**ドキュメントシリーズ完了！**

このContext Management 2025完全ガイドは以下の5つのドキュメントで構成されています：

1. **[01-evolution.md](./01-evolution.md)**: 1年間の進化の分析
2. **[02-modern-architecture.md](./02-modern-architecture.md)**: モダンアーキテクチャ設計
3. **[03-implementation-patterns.md](./03-implementation-patterns.md)**: 実装パターン集
4. **[04-best-practices.md](./04-best-practices.md)**: ベストプラクティス
5. **[05-integration-examples.md](./05-integration-examples.md)**: 統合実装例（本ドキュメント）

実装の成功を祈ります！🚀
