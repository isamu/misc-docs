# 03. 実装パターン集

このドキュメントでは、モダンなContext Management（2025年版）を実装する際の**具体的なパターンとアルゴリズム**を提供します。実際のコードを交えながら、実務で使える形で解説します。

---

## 目次

1. [優先度管理の実装](#1-優先度管理の実装)
2. [トークン予算配分アルゴリズム](#2-トークン予算配分アルゴリズム)
3. [圧縮戦略の実装](#3-圧縮戦略の実装)
4. [tool_use/tool_resultペア保持](#4-tool_usetool_resultペア保持)
5. [レースコンディション対策](#5-レースコンディション対策)
6. [チェックポイント統合](#6-チェックポイント統合)
7. [実践的なコード例](#7-実践的なコード例)

---

## 1. 優先度管理の実装

### 1.1 メッセージ優先度スコアリング

各メッセージに優先度スコアを付与し、圧縮時の判断材料とします。

```typescript
interface MessagePriority {
  messageId: string
  score: number
  reason: string[]
  category: 'critical' | 'important' | 'normal' | 'low'
}

class PriorityScorer {
  /**
   * メッセージの優先度を計算
   * 複数の要素を加算して最終スコアを算出
   */
  scoreMessage(message: Message, context: ScoringContext): MessagePriority {
    let score = 0
    const reasons: string[] = []

    // 1. Role-based scoring
    if (message.role === 'user') {
      score += 10
      reasons.push('User message (+10)')
    } else if (message.role === 'assistant') {
      score += 5
      reasons.push('Assistant message (+5)')
    }

    // 2. Recency scoring (最新ほど高い)
    const age = context.currentIndex - context.messageIndex
    const recencyScore = Math.max(0, 20 - age)
    score += recencyScore
    reasons.push(`Recency (+${recencyScore})`)

    // 3. Content type scoring
    const hasToolUse = message.content.some(c => c.type === 'tool_use')
    const hasToolResult = message.content.some(c => c.type === 'tool_result')

    if (hasToolUse || hasToolResult) {
      score += 15
      reasons.push('Contains tools (+15)')
    }

    // 4. Task relevance (タスク関連キーワードを含むか)
    if (this.isTaskRelevant(message, context.taskKeywords)) {
      score += 25
      reasons.push('Task relevant (+25)')
    }

    // 5. Has important flags (明示的にマーク済み)
    if (message.important) {
      score += 30
      reasons.push('Marked as important (+30)')
    }

    // 6. Contains decisions or commitments
    if (this.containsDecision(message)) {
      score += 20
      reasons.push('Contains decision (+20)')
    }

    // 7. Penalty for already condensed
    if (message.condenseParent) {
      score -= 10
      reasons.push('Already condensed (-10)')
    }

    // Category classification
    let category: 'critical' | 'important' | 'normal' | 'low'
    if (score >= 70) category = 'critical'
    else if (score >= 50) category = 'important'
    else if (score >= 30) category = 'normal'
    else category = 'low'

    return {
      messageId: message.id,
      score,
      reason: reasons,
      category
    }
  }

  private isTaskRelevant(message: Message, keywords: string[]): boolean {
    const text = this.extractText(message)
    return keywords.some(kw =>
      text.toLowerCase().includes(kw.toLowerCase())
    )
  }

  private containsDecision(message: Message): boolean {
    const text = this.extractText(message)
    const decisionPatterns = [
      /we (will|should|must|decided to|agreed to)/i,
      /let's/i,
      /the plan is/i,
      /we'll/i
    ]
    return decisionPatterns.some(pattern => pattern.test(text))
  }

  private extractText(message: Message): string {
    return message.content
      .filter(c => c.type === 'text')
      .map(c => c.text)
      .join(' ')
  }
}

interface ScoringContext {
  currentIndex: number
  messageIndex: number
  taskKeywords: string[]
}
```

### 1.2 優先度に基づく選択アルゴリズム

```typescript
class PrioritySelector {
  /**
   * 優先度に基づいてメッセージを選択
   * トークン予算内で最も重要なメッセージを選ぶ
   */
  selectMessages(
    messages: Message[],
    priorities: MessagePriority[],
    tokenBudget: number
  ): Message[] {
    // 1. 優先度でソート（高→低）
    const sorted = messages
      .map((msg, idx) => ({
        message: msg,
        priority: priorities.find(p => p.messageId === msg.id)!,
        index: idx
      }))
      .sort((a, b) => b.priority.score - a.priority.score)

    // 2. Greedy selection (予算内で詰め込む)
    const selected: Message[] = []
    let usedTokens = 0

    for (const item of sorted) {
      const tokens = this.countTokens(item.message)

      if (usedTokens + tokens <= tokenBudget) {
        selected.push(item.message)
        usedTokens += tokens
      } else if (item.priority.category === 'critical') {
        // Criticalは強制的に含める（予算超過しても）
        selected.push(item.message)
        usedTokens += tokens
      }
    }

    // 3. 元の順序を復元（時系列を維持）
    return selected.sort((a, b) => {
      const idxA = messages.indexOf(a)
      const idxB = messages.indexOf(b)
      return idxA - idxB
    })
  }

  private countTokens(message: Message): number {
    // Implementation using tiktoken or similar
    const text = JSON.stringify(message)
    return Math.ceil(text.length / 4) // Rough estimate
  }
}
```

---

## 2. トークン予算配分アルゴリズム

### 2.1 動的予算配分

状態とモデルに応じて、動的に予算を配分します。

```typescript
interface BudgetAllocation {
  system: number      // L0
  task: number        // L1
  runtime: number     // L2
  memory: number      // L3
  evidence: number    // L4
  workBuffer: number  // L5
  reserve: number     // 予備
}

class BudgetAllocator {
  /**
   * Phase と Model に応じた動的予算配分
   */
  allocate(
    totalBudget: number,
    phase: Phase,
    model: ModelType
  ): BudgetAllocation {
    // Base allocation (default)
    const baseRatios = {
      system: 0.05,     // 5%
      task: 0.05,       // 5%
      runtime: 0.03,    // 3%
      memory: 0.52,     // 52%
      evidence: 0.25,   // 25%
      workBuffer: 0.10  // 10%
    }

    // Phase-specific adjustments
    const phaseAdjustments = this.getPhaseAdjustments(phase)

    // Model-specific adjustments
    const modelAdjustments = this.getModelAdjustments(model)

    // Apply adjustments
    const adjusted = this.applyAdjustments(
      baseRatios,
      phaseAdjustments,
      modelAdjustments
    )

    // Calculate actual token amounts
    const allocation: BudgetAllocation = {
      system: Math.floor(totalBudget * adjusted.system),
      task: Math.floor(totalBudget * adjusted.task),
      runtime: Math.floor(totalBudget * adjusted.runtime),
      memory: Math.floor(totalBudget * adjusted.memory),
      evidence: Math.floor(totalBudget * adjusted.evidence),
      workBuffer: Math.floor(totalBudget * adjusted.workBuffer),
      reserve: 0
    }

    // Calculate reserve (remaining)
    const used = Object.values(allocation).reduce((a, b) => a + b, 0)
    allocation.reserve = totalBudget - used

    return allocation
  }

  private getPhaseAdjustments(phase: Phase): Partial<Record<keyof BudgetAllocation, number>> {
    switch (phase) {
      case 'planning':
        return {
          memory: 0.10,    // Increase memory for context
          evidence: 0.10,  // Increase evidence for informed planning
          workBuffer: -0.20 // Decrease work buffer
        }

      case 'execution':
        return {
          memory: -0.10,   // Decrease memory
          evidence: -0.05, // Decrease evidence
          workBuffer: 0.15 // Increase work buffer for intermediate results
        }

      case 'reflection':
        return {
          memory: 0.15,    // Increase memory for reviewing history
          evidence: 0.05,  // Slight increase for evidence review
          workBuffer: -0.20 // Decrease work buffer
        }

      default:
        return {}
    }
  }

  private getModelAdjustments(model: ModelType): Partial<Record<keyof BudgetAllocation, number>> {
    switch (model) {
      case 'claude-opus-4':
        // Large context window, can afford more evidence
        return {
          evidence: 0.05,
          memory: 0.05
        }

      case 'claude-haiku-3-5':
        // Smaller context, prioritize efficiency
        return {
          evidence: -0.10,
          memory: -0.05,
          workBuffer: 0.15
        }

      default:
        return {}
    }
  }

  private applyAdjustments(
    base: Record<string, number>,
    ...adjustments: Partial<Record<string, number>>[]
  ): Record<string, number> {
    const result = { ...base }

    for (const adjustment of adjustments) {
      for (const [key, delta] of Object.entries(adjustment)) {
        if (key in result) {
          result[key] = Math.max(0, result[key] + (delta || 0))
        }
      }
    }

    // Normalize to ensure sum = 1.0
    const sum = Object.values(result).reduce((a, b) => a + b, 0)
    for (const key in result) {
      result[key] = result[key] / sum
    }

    return result
  }
}
```

### 2.2 層内での二次配分

L3 Memory層内での細かい予算配分例：

```typescript
class MemoryBudgetAllocator {
  /**
   * Memory層内の予算を4種類のメモリに配分
   */
  allocateMemoryBudget(
    memoryBudget: number,
    state: MemoryLayer
  ): {
    shortTerm: number
    episodic: number
    semantic: number
    procedural: number
  } {
    // Check what data is available
    const hasShortTerm = state.shortTerm.length > 0
    const hasEpisodic = state.episodic.length > 0
    const hasSemantic = Object.keys(state.semantic).length > 0
    const hasProcedural = state.procedural.length > 0

    // Count active memory types
    const activeTypes = [
      hasShortTerm,
      hasEpisodic,
      hasSemantic,
      hasProcedural
    ].filter(Boolean).length

    if (activeTypes === 0) {
      return { shortTerm: 0, episodic: 0, semantic: 0, procedural: 0 }
    }

    // Base allocation (equal split among active types)
    const basePerType = memoryBudget / activeTypes

    // Priority adjustments
    const priorities = {
      shortTerm: 1.5,    // Higher priority (recent context)
      episodic: 1.0,     // Normal
      semantic: 0.8,     // Lower (can be looked up)
      procedural: 1.2    // Higher (action-oriented)
    }

    // Calculate weighted allocation
    let totalWeight = 0
    if (hasShortTerm) totalWeight += priorities.shortTerm
    if (hasEpisodic) totalWeight += priorities.episodic
    if (hasSemantic) totalWeight += priorities.semantic
    if (hasProcedural) totalWeight += priorities.procedural

    return {
      shortTerm: hasShortTerm
        ? Math.floor(memoryBudget * priorities.shortTerm / totalWeight)
        : 0,
      episodic: hasEpisodic
        ? Math.floor(memoryBudget * priorities.episodic / totalWeight)
        : 0,
      semantic: hasSemantic
        ? Math.floor(memoryBudget * priorities.semantic / totalWeight)
        : 0,
      procedural: hasProcedural
        ? Math.floor(memoryBudget * priorities.procedural / totalWeight)
        : 0
    }
  }
}
```

---

## 3. 圧縮戦略の実装

### 3.1 Condensation（AI要約）の実装

```typescript
class CondensationEngine {
  constructor(
    private llm: LLMClient,
    private tokenCounter: TokenCounter
  ) {}

  /**
   * メッセージ群をAIで要約して圧縮
   * 70-90%のトークン削減を目指す
   */
  async condense(
    messages: Message[],
    keepRecent: number = 10
  ): Promise<CondensationResult> {
    // 1. Split messages into "to condense" and "to keep"
    const { toCondense, toKeep, toolBlocks } = this.splitMessages(messages, keepRecent)

    if (toCondense.length === 0) {
      return { success: false, reason: 'Nothing to condense' }
    }

    // 2. Create condensation prompt
    const prompt = this.createCondensationPrompt(toCondense)

    // 3. Call LLM for summarization
    const summaryResponse = await this.llm.generate({
      messages: [
        {
          role: 'user',
          content: prompt
        }
      ],
      temperature: 0.3,  // Lower temperature for consistent summaries
      maxTokens: 4000
    })

    const summary = summaryResponse.content[0].text

    // 4. Extract reasoning blocks if present
    const reasoningBlocks = summaryResponse.content
      .filter(c => c.type === 'reasoning')

    // 5. Create condensed message
    const condensedMessage: Message = {
      id: crypto.randomUUID(),
      role: 'assistant',
      content: [
        {
          type: 'reasoning',
          text: `[Context Condensation: ${toCondense.length} messages → summary]`
        },
        ...reasoningBlocks,
        {
          type: 'text',
          text: `## Context Summary\n\n${summary}`
        },
        ...toolBlocks  // Preserve tool_use blocks
      ],
      condenseId: crypto.randomUUID(),
      timestamp: Date.now()
    }

    // 6. Tag original messages
    const tagged = toCondense.map(msg => ({
      ...msg,
      condenseParent: condensedMessage.condenseId
    }))

    // 7. Calculate compression ratio
    const originalTokens = this.tokenCounter.countMessages(toCondense)
    const condensedTokens = this.tokenCounter.countMessages([condensedMessage])
    const ratio = 1 - (condensedTokens / originalTokens)

    return {
      success: true,
      condensedMessage,
      taggedMessages: tagged,
      compressionRatio: ratio,
      originalTokens,
      condensedTokens
    }
  }

  private splitMessages(messages: Message[], keepRecent: number) {
    const toKeep = messages.slice(-keepRecent)
    const toCondense = messages.slice(0, -keepRecent)

    // Extract tool_use blocks to preserve (Native Tools requirement)
    const toolBlocks: ContentBlock[] = []
    for (const msg of toCondense) {
      const tools = msg.content.filter(c => c.type === 'tool_use')
      toolBlocks.push(...tools)
    }

    return { toCondense, toKeep, toolBlocks }
  }

  private createCondensationPrompt(messages: Message[]): string {
    return `You are tasked with creating a concise summary of the following conversation history.

## Requirements:
1. Capture key decisions, commitments, and outcomes
2. Preserve important facts and context
3. Maintain chronological flow where relevant
4. Use bullet points for clarity
5. Aim for 70-90% token reduction
6. Focus on information that will be useful for continuing the conversation

## Conversation to summarize:

${this.formatMessages(messages)}

## Your summary:
`
  }

  private formatMessages(messages: Message[]): string {
    return messages.map(msg => {
      const text = msg.content
        .filter(c => c.type === 'text')
        .map(c => c.text)
        .join('\n')
      return `[${msg.role}]: ${text}`
    }).join('\n\n---\n\n')
  }
}

interface CondensationResult {
  success: boolean
  reason?: string
  condensedMessage?: Message
  taggedMessages?: Message[]
  compressionRatio?: number
  originalTokens?: number
  condensedTokens?: number
}
```

### 3.2 Truncation（スライディングウィンドウ）の実装

```typescript
class TruncationEngine {
  /**
   * スライディングウィンドウで古いメッセージをタグ付け
   * Condensation失敗時のフォールバック
   */
  truncate(
    messages: Message[],
    ratio: number = 0.5  // 50% of messages to truncate
  ): TruncationResult {
    // 1. Calculate how many to truncate
    const truncateCount = Math.floor(messages.length * ratio)

    if (truncateCount === 0) {
      return { success: false, reason: 'Nothing to truncate' }
    }

    // 2. Identify candidates (oldest messages first)
    const candidates = messages.slice(0, truncateCount)
    const keeping = messages.slice(truncateCount)

    // 3. Preserve critical messages
    const { toTruncate, preserved } = this.preserveCritical(candidates)

    // 4. Tag messages for truncation
    const truncateId = crypto.randomUUID()
    const tagged = toTruncate.map(msg => ({
      ...msg,
      truncationParent: truncateId
    }))

    // 5. Calculate reduction
    const originalTokens = this.countTokens(messages)
    const newTokens = this.countTokens([...preserved, ...keeping])
    const reduction = originalTokens - newTokens

    return {
      success: true,
      taggedMessages: tagged,
      preservedMessages: preserved,
      truncationId: truncateId,
      tokensReduced: reduction,
      reductionRatio: reduction / originalTokens
    }
  }

  /**
   * Preserve messages that should not be truncated
   */
  private preserveCritical(candidates: Message[]): {
    toTruncate: Message[]
    preserved: Message[]
  } {
    const preserved: Message[] = []
    const toTruncate: Message[] = []

    for (const msg of candidates) {
      // Preserve if:
      // 1. Marked as important
      // 2. Contains tool_use/tool_result (Native Tools requirement)
      // 3. Contains decisions
      // 4. User message (higher priority)

      const hasTools = msg.content.some(c =>
        c.type === 'tool_use' || c.type === 'tool_result'
      )

      const isImportant = msg.important === true
      const isUser = msg.role === 'user'
      const hasDecision = this.containsDecision(msg)

      if (hasTools || isImportant || (isUser && hasDecision)) {
        preserved.push(msg)
      } else {
        toTruncate.push(msg)
      }
    }

    return { toTruncate, preserved }
  }

  private containsDecision(msg: Message): boolean {
    const text = msg.content
      .filter(c => c.type === 'text')
      .map(c => c.text)
      .join(' ')

    const patterns = [
      /decided/i,
      /agreed/i,
      /will do/i,
      /let's/i,
      /plan is/i
    ]

    return patterns.some(p => p.test(text))
  }

  private countTokens(messages: Message[]): number {
    // Use tiktoken or similar
    const text = JSON.stringify(messages)
    return Math.ceil(text.length / 4)
  }
}

interface TruncationResult {
  success: boolean
  reason?: string
  taggedMessages?: Message[]
  preservedMessages?: Message[]
  truncationId?: string
  tokensReduced?: number
  reductionRatio?: number
}
```

### 3.3 二段階圧縮戦略

Condensation → Truncation のフォールバック戦略：

```typescript
class CompressionOrchestrator {
  constructor(
    private condensation: CondensationEngine,
    private truncation: TruncationEngine
  ) {}

  /**
   * 二段階圧縮: まずCondensation、失敗したらTruncation
   */
  async compress(
    messages: Message[],
    targetTokens: number
  ): Promise<CompressionResult> {
    const currentTokens = this.countTokens(messages)

    if (currentTokens <= targetTokens) {
      return {
        method: 'none',
        success: true,
        messages: messages
      }
    }

    // Stage 1: Try Condensation (AI-powered)
    try {
      const condensed = await this.condensation.condense(messages, 10)

      if (condensed.success && condensed.condensedTokens! < targetTokens) {
        return {
          method: 'condensation',
          success: true,
          messages: this.applyCondensation(messages, condensed),
          metadata: {
            compressionRatio: condensed.compressionRatio,
            originalTokens: condensed.originalTokens,
            finalTokens: condensed.condensedTokens
          }
        }
      }
    } catch (error) {
      console.warn('Condensation failed, falling back to truncation', error)
    }

    // Stage 2: Fallback to Truncation
    const ratio = 1 - (targetTokens / currentTokens)
    const truncated = this.truncation.truncate(messages, ratio)

    if (truncated.success) {
      return {
        method: 'truncation',
        success: true,
        messages: this.applyTruncation(messages, truncated),
        metadata: {
          tokensReduced: truncated.tokensReduced,
          reductionRatio: truncated.reductionRatio
        }
      }
    }

    // Both failed
    return {
      method: 'failed',
      success: false,
      messages: messages
    }
  }

  private applyCondensation(
    messages: Message[],
    result: CondensationResult
  ): Message[] {
    const condensedIds = new Set(result.taggedMessages!.map(m => m.id))

    return [
      ...messages.filter(m => !condensedIds.has(m.id)),
      result.condensedMessage!
    ]
  }

  private applyTruncation(
    messages: Message[],
    result: TruncationResult
  ): Message[] {
    const truncatedIds = new Set(result.taggedMessages!.map(m => m.id))

    return messages.filter(m => !truncatedIds.has(m.id))
  }

  private countTokens(messages: Message[]): number {
    return Math.ceil(JSON.stringify(messages).length / 4)
  }
}

interface CompressionResult {
  method: 'none' | 'condensation' | 'truncation' | 'failed'
  success: boolean
  messages: Message[]
  metadata?: {
    compressionRatio?: number
    originalTokens?: number
    finalTokens?: number
    tokensReduced?: number
    reductionRatio?: number
  }
}
```

---

## 4. tool_use/tool_resultペア保持

Native Toolsプロトコルに準拠したペア保持の実装。

### 4.1 ペア検出

```typescript
class ToolPairDetector {
  /**
   * tool_use と tool_result のペアを検出
   */
  detectPairs(messages: Message[]): ToolPair[] {
    const pairs: ToolPair[] = []
    const toolUseMap = new Map<string, ToolUseInfo>()

    // First pass: collect all tool_use blocks
    for (let i = 0; i < messages.length; i++) {
      const msg = messages[i]

      for (const content of msg.content) {
        if (content.type === 'tool_use') {
          toolUseMap.set(content.id, {
            toolUseId: content.id,
            toolName: content.name,
            messageIndex: i,
            messageId: msg.id,
            content: content
          })
        }
      }
    }

    // Second pass: find matching tool_result blocks
    for (let i = 0; i < messages.length; i++) {
      const msg = messages[i]

      for (const content of msg.content) {
        if (content.type === 'tool_result') {
          const toolUse = toolUseMap.get(content.tool_use_id)

          if (toolUse) {
            pairs.push({
              toolUseId: content.tool_use_id,
              toolName: toolUse.toolName,
              toolUseMessageIndex: toolUse.messageIndex,
              toolResultMessageIndex: i,
              toolUseMessageId: toolUse.messageId,
              toolResultMessageId: msg.id,
              toolUseContent: toolUse.content,
              toolResultContent: content
            })
          }
        }
      }
    }

    return pairs
  }

  /**
   * メッセージが未完了のtool_useを含むかチェック
   */
  hasOpenToolUse(message: Message, pairs: ToolPair[]): boolean {
    const completedToolUseIds = new Set(pairs.map(p => p.toolUseId))

    return message.content.some(c =>
      c.type === 'tool_use' && !completedToolUseIds.has(c.id)
    )
  }

  /**
   * ペアの相方メッセージを取得
   */
  getPairPartner(
    messageId: string,
    messages: Message[],
    pairs: ToolPair[]
  ): Message | null {
    const relevantPairs = pairs.filter(p =>
      p.toolUseMessageId === messageId || p.toolResultMessageId === messageId
    )

    if (relevantPairs.length === 0) return null

    // Get partner message ID
    const partnerId = relevantPairs[0].toolUseMessageId === messageId
      ? relevantPairs[0].toolResultMessageId
      : relevantPairs[0].toolUseMessageId

    return messages.find(m => m.id === partnerId) || null
  }
}

interface ToolPair {
  toolUseId: string
  toolName: string
  toolUseMessageIndex: number
  toolResultMessageIndex: number
  toolUseMessageId: string
  toolResultMessageId: string
  toolUseContent: ContentBlock
  toolResultContent: ContentBlock
}

interface ToolUseInfo {
  toolUseId: string
  toolName: string
  messageIndex: number
  messageId: string
  content: ContentBlock
}
```

### 4.2 圧縮時のペア保護

```typescript
class PairPreservingCompressor {
  constructor(
    private pairDetector: ToolPairDetector,
    private compressor: CompressionOrchestrator
  ) {}

  /**
   * ペアを壊さずに圧縮
   */
  async compressPreservingPairs(
    messages: Message[],
    targetTokens: number
  ): Promise<Message[]> {
    // 1. Detect all pairs
    const pairs = this.pairDetector.detectPairs(messages)

    // 2. Mark messages as part of pairs
    const pairMessageIds = new Set<string>()
    for (const pair of pairs) {
      pairMessageIds.add(pair.toolUseMessageId)
      pairMessageIds.add(pair.toolResultMessageId)
    }

    // 3. Separate pair messages from others
    const pairMessages: Message[] = []
    const otherMessages: Message[] = []

    for (const msg of messages) {
      if (pairMessageIds.has(msg.id)) {
        pairMessages.push(msg)
      } else {
        otherMessages.push(msg)
      }
    }

    // 4. Calculate token budget
    const pairTokens = this.countTokens(pairMessages)
    const remainingBudget = targetTokens - pairTokens

    if (remainingBudget <= 0) {
      // Pairs alone exceed budget - keep all pairs anyway (critical)
      console.warn('Tool pairs exceed token budget, keeping anyway')
      return pairMessages
    }

    // 5. Compress other messages
    const compressed = await this.compressor.compress(
      otherMessages,
      remainingBudget
    )

    // 6. Merge and sort by original order
    const allMessages = [...pairMessages, ...compressed.messages]
    const originalOrder = new Map(messages.map((m, i) => [m.id, i]))

    allMessages.sort((a, b) => {
      const orderA = originalOrder.get(a.id) ?? Infinity
      const orderB = originalOrder.get(b.id) ?? Infinity
      return orderA - orderB
    })

    return allMessages
  }

  private countTokens(messages: Message[]): number {
    return Math.ceil(JSON.stringify(messages).length / 4)
  }
}
```

---

## 5. レースコンディション対策

### 5.1 メッセージロックメカニズム

```typescript
class MessageLock {
  private locks = new Map<string, Promise<void>>()
  private lockHolders = new Map<string, string>()

  /**
   * メッセージIDに対してロックを取得
   */
  async acquire(messageId: string, holderId: string): Promise<() => void> {
    // Wait for existing lock if any
    while (this.locks.has(messageId)) {
      await this.locks.get(messageId)
    }

    // Create new lock
    let releaseFn: () => void
    const lockPromise = new Promise<void>(resolve => {
      releaseFn = resolve
    })

    this.locks.set(messageId, lockPromise)
    this.lockHolders.set(messageId, holderId)

    // Return release function
    return () => {
      this.locks.delete(messageId)
      this.lockHolders.delete(messageId)
      releaseFn!()
    }
  }

  /**
   * ロックの強制解放（タイムアウト時など）
   */
  forceRelease(messageId: string): void {
    this.locks.delete(messageId)
    this.lockHolders.delete(messageId)
  }

  /**
   * ロック状態を確認
   */
  isLocked(messageId: string): boolean {
    return this.locks.has(messageId)
  }

  /**
   * ロックホルダーを取得
   */
  getHolder(messageId: string): string | undefined {
    return this.lockHolders.get(messageId)
  }
}
```

### 5.2 トランザクショナルなメッセージ操作

```typescript
class TransactionalMessageManager {
  constructor(
    private lock: MessageLock,
    private storage: MessageStorage
  ) {}

  /**
   * アトミックなメッセージ更新
   */
  async updateMessage<T>(
    messageId: string,
    updateFn: (msg: Message) => Message,
    operationId: string
  ): Promise<Message> {
    const release = await this.lock.acquire(messageId, operationId)

    try {
      // 1. Read current state
      const current = await this.storage.getMessage(messageId)

      if (!current) {
        throw new Error(`Message ${messageId} not found`)
      }

      // 2. Apply update
      const updated = updateFn(current)

      // 3. Validate
      this.validate(updated)

      // 4. Write back
      await this.storage.updateMessage(messageId, updated)

      return updated
    } finally {
      release()
    }
  }

  /**
   * バッチ更新（複数メッセージを一貫性を保って更新）
   */
  async batchUpdate(
    updates: Array<{ messageId: string; updateFn: (msg: Message) => Message }>,
    operationId: string
  ): Promise<Message[]> {
    // 1. Sort message IDs to prevent deadlock
    const sorted = updates.slice().sort((a, b) =>
      a.messageId.localeCompare(b.messageId)
    )

    // 2. Acquire all locks
    const releases: Array<() => void> = []
    for (const { messageId } of sorted) {
      const release = await this.lock.acquire(messageId, operationId)
      releases.push(release)
    }

    try {
      // 3. Apply all updates
      const results: Message[] = []

      for (const { messageId, updateFn } of sorted) {
        const current = await this.storage.getMessage(messageId)
        if (!current) {
          throw new Error(`Message ${messageId} not found`)
        }

        const updated = updateFn(current)
        this.validate(updated)

        await this.storage.updateMessage(messageId, updated)
        results.push(updated)
      }

      return results
    } finally {
      // 4. Release all locks (in reverse order)
      for (const release of releases.reverse()) {
        release()
      }
    }
  }

  private validate(message: Message): void {
    if (!message.id) throw new Error('Message must have ID')
    if (!message.role) throw new Error('Message must have role')
    if (!message.content) throw new Error('Message must have content')
  }
}
```

### 5.3 楽観的ロック（バージョニング）

```typescript
interface VersionedMessage extends Message {
  version: number
  lastModified: number
}

class OptimisticLockManager {
  /**
   * 楽観的ロックでメッセージを更新
   */
  async updateWithOptimisticLock(
    messageId: string,
    updateFn: (msg: VersionedMessage) => VersionedMessage,
    maxRetries: number = 3
  ): Promise<VersionedMessage> {
    let retries = 0

    while (retries < maxRetries) {
      // 1. Read current version
      const current = await this.storage.getMessage(messageId) as VersionedMessage

      if (!current) {
        throw new Error(`Message ${messageId} not found`)
      }

      // 2. Apply update
      const updated = updateFn({ ...current })
      updated.version = current.version + 1
      updated.lastModified = Date.now()

      // 3. Try to write with version check
      const success = await this.storage.updateIfVersionMatches(
        messageId,
        updated,
        current.version
      )

      if (success) {
        return updated
      }

      // Version mismatch - retry
      retries++
      await this.sleep(Math.pow(2, retries) * 100) // Exponential backoff
    }

    throw new Error(`Failed to update message ${messageId} after ${maxRetries} retries`)
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms))
  }
}
```

---

## 6. チェックポイント統合

### 6.1 チェックポイント保存

```typescript
class CheckpointManager {
  constructor(
    private storage: CheckpointStorage,
    private compressor: CompressionOrchestrator
  ) {}

  /**
   * 現在の状態をチェックポイントとして保存
   */
  async saveCheckpoint(
    conversationId: string,
    state: LayeredState,
    messages: Message[]
  ): Promise<Checkpoint> {
    const checkpoint: Checkpoint = {
      id: crypto.randomUUID(),
      conversationId,
      timestamp: Date.now(),
      state: this.cloneState(state),
      messages: this.cloneMessages(messages),
      metadata: {
        messageCount: messages.length,
        totalTokens: this.countTokens(messages),
        phase: state.l2_runtime.phase
      }
    }

    await this.storage.save(checkpoint)

    return checkpoint
  }

  /**
   * チェックポイントから復元
   */
  async restore(checkpointId: string): Promise<{
    state: LayeredState
    messages: Message[]
  }> {
    const checkpoint = await this.storage.load(checkpointId)

    if (!checkpoint) {
      throw new Error(`Checkpoint ${checkpointId} not found`)
    }

    return {
      state: this.cloneState(checkpoint.state),
      messages: this.cloneMessages(checkpoint.messages)
    }
  }

  /**
   * チェックポイント一覧を取得
   */
  async listCheckpoints(
    conversationId: string,
    limit: number = 10
  ): Promise<CheckpointMetadata[]> {
    const checkpoints = await this.storage.list(conversationId, limit)

    return checkpoints.map(cp => ({
      id: cp.id,
      timestamp: cp.timestamp,
      messageCount: cp.metadata.messageCount,
      totalTokens: cp.metadata.totalTokens,
      phase: cp.metadata.phase
    }))
  }

  /**
   * 古いチェックポイントを削除
   */
  async cleanup(
    conversationId: string,
    keepCount: number = 5
  ): Promise<number> {
    const checkpoints = await this.storage.list(conversationId)

    if (checkpoints.length <= keepCount) {
      return 0
    }

    // Sort by timestamp (newest first)
    checkpoints.sort((a, b) => b.timestamp - a.timestamp)

    // Delete old ones
    const toDelete = checkpoints.slice(keepCount)

    for (const cp of toDelete) {
      await this.storage.delete(cp.id)
    }

    return toDelete.length
  }

  private cloneState(state: LayeredState): LayeredState {
    return JSON.parse(JSON.stringify(state))
  }

  private cloneMessages(messages: Message[]): Message[] {
    return JSON.parse(JSON.stringify(messages))
  }

  private countTokens(messages: Message[]): number {
    return Math.ceil(JSON.stringify(messages).length / 4)
  }
}

interface Checkpoint {
  id: string
  conversationId: string
  timestamp: number
  state: LayeredState
  messages: Message[]
  metadata: {
    messageCount: number
    totalTokens: number
    phase: Phase
  }
}

interface CheckpointMetadata {
  id: string
  timestamp: number
  messageCount: number
  totalTokens: number
  phase: Phase
}
```

### 6.2 自動チェックポイント

```typescript
class AutoCheckpointManager {
  private lastCheckpoint: number = 0
  private checkpointInterval: number = 10 * 60 * 1000  // 10 minutes

  constructor(
    private checkpointManager: CheckpointManager
  ) {}

  /**
   * 条件に応じて自動的にチェックポイントを作成
   */
  async maybeCreateCheckpoint(
    conversationId: string,
    state: LayeredState,
    messages: Message[],
    force: boolean = false
  ): Promise<Checkpoint | null> {
    const now = Date.now()

    // Check if checkpoint is needed
    const shouldCheckpoint = force ||
      this.checkTimeBased(now) ||
      this.checkPhaseBased(state) ||
      this.checkMessageBased(messages) ||
      this.checkTokenBased(messages)

    if (!shouldCheckpoint) {
      return null
    }

    // Create checkpoint
    const checkpoint = await this.checkpointManager.saveCheckpoint(
      conversationId,
      state,
      messages
    )

    this.lastCheckpoint = now

    return checkpoint
  }

  private checkTimeBased(now: number): boolean {
    return now - this.lastCheckpoint >= this.checkpointInterval
  }

  private checkPhaseBased(state: LayeredState): boolean {
    // Checkpoint on phase transitions
    const previousPhase = this.previousPhase
    const currentPhase = state.l2_runtime.phase

    if (previousPhase && previousPhase !== currentPhase) {
      this.previousPhase = currentPhase
      return true
    }

    this.previousPhase = currentPhase
    return false
  }

  private checkMessageBased(messages: Message[]): boolean {
    // Checkpoint every N messages
    const messagesSinceCheckpoint = messages.length - this.lastMessageCount
    this.lastMessageCount = messages.length

    return messagesSinceCheckpoint >= 50
  }

  private checkTokenBased(messages: Message[]): boolean {
    // Checkpoint when tokens exceed threshold
    const tokens = this.countTokens(messages)
    return tokens >= 100000
  }

  private previousPhase: Phase | null = null
  private lastMessageCount: number = 0

  private countTokens(messages: Message[]): number {
    return Math.ceil(JSON.stringify(messages).length / 4)
  }
}
```

---

## 7. 実践的なコード例

### 7.1 完全な実行フロー

すべてのパターンを統合した実行例：

```typescript
class ModernContextEngine {
  private state: LayeredState
  private messages: Message[] = []
  private checkpointManager: CheckpointManager
  private compressor: CompressionOrchestrator
  private messageManager: TransactionalMessageManager
  private toolProjector: DynamicToolProjector
  private budgetAllocator: BudgetAllocator

  constructor(private config: EngineConfig) {
    this.state = this.initializeState()
    this.checkpointManager = new CheckpointManager(config.storage, compressor)
    this.compressor = new CompressionOrchestrator(condensation, truncation)
    this.messageManager = new TransactionalMessageManager(lock, storage)
    this.toolProjector = new DynamicToolProjector()
    this.budgetAllocator = new BudgetAllocator()
  }

  /**
   * メインの実行ループ
   */
  async executeTask(userInput: string): Promise<TaskResult> {
    // 1. Add user message
    await this.addUserMessage(userInput)

    // 2. Check if compression is needed
    if (this.needsCompression()) {
      await this.compressContext()
    }

    // 3. Build context for LLM
    const context = await this.buildContext()

    // 4. Project tools based on current state
    const tools = this.toolProjector.project(this.state)

    // 5. Call LLM
    const response = await this.llm.generate({
      system: context.system,
      messages: context.messages,
      tools: tools,
      maxTokens: this.config.maxTokens
    })

    // 6. Process response
    await this.processResponse(response)

    // 7. Update state
    this.updateState(response)

    // 8. Maybe create checkpoint
    await this.checkpointManager.maybeCreateCheckpoint(
      this.config.conversationId,
      this.state,
      this.messages
    )

    return {
      success: true,
      response: response.content
    }
  }

  private async addUserMessage(input: string): Promise<void> {
    const message: Message = {
      id: crypto.randomUUID(),
      role: 'user',
      content: [{ type: 'text', text: input }],
      timestamp: Date.now()
    }

    this.messages.push(message)
  }

  private needsCompression(): boolean {
    const currentTokens = this.countTokens(this.messages)
    const threshold = this.config.maxTokens * 0.75  // 75%

    return currentTokens >= threshold
  }

  private async compressContext(): Promise<void> {
    const targetTokens = this.config.maxTokens * 0.5  // Compress to 50%

    const result = await this.compressor.compress(
      this.messages,
      targetTokens
    )

    if (result.success) {
      this.messages = result.messages
    }
  }

  private async buildContext(): Promise<ContextOutput> {
    const budget = this.budgetAllocator.allocate(
      this.config.maxTokens,
      this.state.l2_runtime.phase,
      this.config.model
    )

    const builder = new ContextBuilder(this.state, this.messages, budget)
    return builder.buildAllLayers()
  }

  private async processResponse(response: LLMResponse): Promise<void> {
    const message: Message = {
      id: crypto.randomUUID(),
      role: 'assistant',
      content: response.content,
      timestamp: Date.now()
    }

    this.messages.push(message)

    // Handle tool calls if any
    for (const content of response.content) {
      if (content.type === 'tool_use') {
        await this.executeTool(content)
      }
    }
  }

  private async executeTool(toolUse: ToolUseBlock): Promise<void> {
    // Execute the tool
    const result = await this.toolExecutor.execute(
      toolUse.name,
      toolUse.input
    )

    // Add tool result message
    const message: Message = {
      id: crypto.randomUUID(),
      role: 'user',
      content: [{
        type: 'tool_result',
        tool_use_id: toolUse.id,
        content: result
      }],
      timestamp: Date.now()
    }

    this.messages.push(message)
  }

  private updateState(response: LLMResponse): void {
    // Update phase if needed
    // Update permissions if needed
    // Update memory if needed
    // etc.
  }

  private initializeState(): LayeredState {
    return {
      l0_system: { /* ... */ },
      l1_task: { /* ... */ },
      l2_runtime: { phase: 'planning', /* ... */ },
      l3_memory: { /* ... */ },
      l4_evidence: { /* ... */ },
      l5_workBuffer: { /* ... */ }
    }
  }

  private countTokens(messages: Message[]): number {
    return Math.ceil(JSON.stringify(messages).length / 4)
  }
}

interface EngineConfig {
  conversationId: string
  maxTokens: number
  model: ModelType
  storage: CheckpointStorage
}

interface TaskResult {
  success: boolean
  response: ContentBlock[]
  error?: string
}
```

### 7.2 エラーハンドリングとリカバリ

```typescript
class RobustContextEngine extends ModernContextEngine {
  /**
   * エラーハンドリング付き実行
   */
  async executeTaskSafely(userInput: string): Promise<TaskResult> {
    try {
      return await this.executeTask(userInput)
    } catch (error) {
      return await this.handleError(error as Error, userInput)
    }
  }

  private async handleError(
    error: Error,
    userInput: string
  ): Promise<TaskResult> {
    console.error('Task execution failed:', error)

    // Try to recover
    if (error.message.includes('token limit')) {
      // Force compression
      await this.forceCompression()
      return await this.executeTask(userInput)
    }

    if (error.message.includes('checkpoint')) {
      // Restore from last checkpoint
      await this.restoreFromCheckpoint()
      return await this.executeTask(userInput)
    }

    // Cannot recover
    return {
      success: false,
      response: [],
      error: error.message
    }
  }

  private async forceCompression(): Promise<void> {
    const targetTokens = this.config.maxTokens * 0.3  // Aggressive: 30%

    const result = await this.compressor.compress(
      this.messages,
      targetTokens
    )

    if (result.success) {
      this.messages = result.messages
    } else {
      // Truncate as last resort
      this.messages = this.messages.slice(-10)
    }
  }

  private async restoreFromCheckpoint(): Promise<void> {
    const checkpoints = await this.checkpointManager.listCheckpoints(
      this.config.conversationId,
      1
    )

    if (checkpoints.length > 0) {
      const restored = await this.checkpointManager.restore(checkpoints[0].id)
      this.state = restored.state
      this.messages = restored.messages
    }
  }
}
```

---

## まとめ

この実装パターン集では、以下を提供しました：

1. **優先度管理**: スコアリングと選択アルゴリズム
2. **トークン予算配分**: 動的配分と層内二次配分
3. **圧縮戦略**: Condensation、Truncation、二段階戦略
4. **ペア保持**: Native Toolsプロトコル準拠
5. **レースコンディション対策**: ロック、トランザクション、楽観的ロック
6. **チェックポイント**: 保存、復元、自動化
7. **実践例**: 完全な実行フローとエラーハンドリング

これらのパターンを組み合わせることで、堅牢で効率的なContext Management システムを構築できます。

次のドキュメント [04-best-practices.md](./04-best-practices.md) では、これらの実装を使う際のベストプラクティスと落とし穴の回避方法を解説します。
