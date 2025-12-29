# 04. ベストプラクティス

このドキュメントでは、モダンなContext Management（2025年版）を実装・運用する際の**ベストプラクティス、よくある落とし穴、設計原則**を解説します。

---

## 目次

1. [設計原則](#1-設計原則)
2. [よくある落とし穴と回避方法](#2-よくある落とし穴と回避方法)
3. [パフォーマンス最適化](#3-パフォーマンス最適化)
4. [セキュリティとコンプライアンス](#4-セキュリティとコンプライアンス)
5. [テスト戦略](#5-テスト戦略)
6. [モニタリングとデバッグ](#6-モニタリングとデバッグ)
7. [プロダクション運用チェックリスト](#7-プロダクション運用チェックリスト)

---

## 1. 設計原則

### 1.1 SOLID原則の適用

#### Single Responsibility Principle（単一責任の原則）

各コンポーネントは1つの責任のみを持つべきです。

```typescript
// ❌ BAD: 複数の責任が混在
class ContextManager {
  compressMessages() { /* ... */ }
  allocateBudget() { /* ... */ }
  executeTool() { /* ... */ }
  saveCheckpoint() { /* ... */ }
  projectTools() { /* ... */ }
}

// ✅ GOOD: 責任を分離
class CompressionEngine {
  compressMessages() { /* ... */ }
}

class BudgetAllocator {
  allocateBudget() { /* ... */ }
}

class ToolExecutor {
  executeTool() { /* ... */ }
}

class CheckpointManager {
  saveCheckpoint() { /* ... */ }
}

class ToolProjector {
  projectTools() { /* ... */ }
}
```

#### Interface Segregation（インターフェース分離の原則）

クライアントは使わないメソッドに依存すべきではありません。

```typescript
// ❌ BAD: 巨大なインターフェース
interface ContextEngine {
  compress(): Promise<void>
  expand(): Promise<void>
  allocate(): void
  project(): Tool[]
  save(): Promise<void>
  restore(): Promise<void>
  execute(): Promise<void>
}

// ✅ GOOD: 小さく分離されたインターフェース
interface Compressible {
  compress(): Promise<void>
}

interface Expandable {
  expand(): Promise<void>
}

interface BudgetManaged {
  allocate(): void
}

interface ToolProjecting {
  project(): Tool[]
}

interface Checkpointable {
  save(): Promise<void>
  restore(): Promise<void>
}
```

### 1.2 依存性注入（DI）

ハードコーディングではなく、依存性を注入します。

```typescript
// ❌ BAD: ハードコーディング
class ContextEngine {
  private compressor = new CondensationEngine()
  private storage = new FileStorage()

  async compress() {
    await this.compressor.condense(/* ... */)
  }
}

// ✅ GOOD: 依存性注入
class ContextEngine {
  constructor(
    private compressor: ICompressor,
    private storage: IStorage
  ) {}

  async compress() {
    await this.compressor.condense(/* ... */)
  }
}

// 使用例
const engine = new ContextEngine(
  new CondensationEngine(llm, tokenCounter),
  new FileStorage('/path/to/data')
)
```

### 1.3 不変性（Immutability）

可能な限り不変なデータ構造を使います。

```typescript
// ❌ BAD: 破壊的変更
function addMessage(messages: Message[], newMessage: Message): void {
  messages.push(newMessage)  // 元の配列を変更
}

// ✅ GOOD: 非破壊的
function addMessage(messages: Message[], newMessage: Message): Message[] {
  return [...messages, newMessage]  // 新しい配列を返す
}

// ❌ BAD: オブジェクトの直接変更
function updateState(state: LayeredState, phase: Phase): void {
  state.l2_runtime.phase = phase
}

// ✅ GOOD: 新しいオブジェクトを返す
function updateState(state: LayeredState, phase: Phase): LayeredState {
  return {
    ...state,
    l2_runtime: {
      ...state.l2_runtime,
      phase
    }
  }
}
```

### 1.4 非同期処理のベストプラクティス

```typescript
// ❌ BAD: Promise地獄
function processMessages() {
  return loadMessages().then(messages => {
    return compressMessages(messages).then(compressed => {
      return saveMessages(compressed).then(result => {
        return result
      })
    })
  })
}

// ✅ GOOD: async/await
async function processMessages() {
  const messages = await loadMessages()
  const compressed = await compressMessages(messages)
  const result = await saveMessages(compressed)
  return result
}

// ❌ BAD: 並列実行可能なのに直列
async function loadData() {
  const messages = await loadMessages()
  const state = await loadState()
  const checkpoints = await loadCheckpoints()
  return { messages, state, checkpoints }
}

// ✅ GOOD: 並列実行
async function loadData() {
  const [messages, state, checkpoints] = await Promise.all([
    loadMessages(),
    loadState(),
    loadCheckpoints()
  ])
  return { messages, state, checkpoints }
}
```

---

## 2. よくある落とし穴と回避方法

### 2.1 トークンカウントの不正確さ

**問題**: 簡易的な文字数カウントではトークン数が不正確

```typescript
// ❌ BAD: 文字数ベース（不正確）
function countTokens(text: string): number {
  return Math.ceil(text.length / 4)
}

// ✅ GOOD: tiktoken使用（正確）
import { encoding_for_model } from 'tiktoken'

function countTokens(text: string, model: string = 'gpt-4'): number {
  const encoding = encoding_for_model(model)
  const tokens = encoding.encode(text)
  encoding.free()
  return tokens.length
}
```

**ベストプラクティス**:
- 正確なトークンカウントには `tiktoken` を使用
- モデルごとに適切なエンコーディングを選択（`o200k_base` など）
- カウント結果はキャッシュする（同じテキストの再計算を避ける）

### 2.2 tool_use/tool_resultペアの破壊

**問題**: 圧縮時にペアが壊れてプロトコル違反

```typescript
// ❌ BAD: ペアを考慮せずに削除
function compress(messages: Message[]): Message[] {
  return messages.slice(-100)  // 単純に古いものを削除
}

// ✅ GOOD: ペアを保持
function compress(messages: Message[]): Message[] {
  const pairs = detectToolPairs(messages)
  const pairMessageIds = new Set(
    pairs.flatMap(p => [p.toolUseMessageId, p.toolResultMessageId])
  )

  // ペアメッセージは必ず保持
  const mustKeep = messages.filter(m => pairMessageIds.has(m.id))
  const canCompress = messages.filter(m => !pairMessageIds.has(m.id))

  const compressed = compressMessages(canCompress)

  return [...mustKeep, ...compressed].sort(byTimestamp)
}
```

**ベストプラクティス**:
- tool_use と tool_result は常にペアで保持
- 圧縮前に必ずペア検出を実行
- ペアメッセージは `critical` 扱いにする

### 2.3 レースコンディション

**問題**: 並行アクセスでメッセージが破損

```typescript
// ❌ BAD: 保護なし
async function addMessage(message: Message) {
  const messages = await loadMessages()
  messages.push(message)
  await saveMessages(messages)
}

// ✅ GOOD: ロック機構
async function addMessage(message: Message) {
  const release = await lock.acquire('messages')
  try {
    const messages = await loadMessages()
    messages.push(message)
    await saveMessages(messages)
  } finally {
    release()
  }
}
```

**ベストプラクティス**:
- 並行アクセスが予想される場合は必ずロック
- トランザクション境界を明確に
- デッドロックを避けるためロック順序を統一

### 2.4 メモリリーク

**問題**: 古いメッセージやチェックポイントが溜まり続ける

```typescript
// ❌ BAD: 無限に蓄積
class ContextEngine {
  private messages: Message[] = []
  private checkpoints: Checkpoint[] = []

  async addMessage(msg: Message) {
    this.messages.push(msg)  // 永遠に増え続ける
  }

  async saveCheckpoint() {
    this.checkpoints.push(createCheckpoint())  // これも増え続ける
  }
}

// ✅ GOOD: 定期的にクリーンアップ
class ContextEngine {
  private messages: Message[] = []
  private checkpoints: Checkpoint[] = []
  private maxCheckpoints = 10

  async addMessage(msg: Message) {
    this.messages.push(msg)

    // 定期的に圧縮
    if (this.needsCompression()) {
      await this.compress()
    }
  }

  async saveCheckpoint() {
    this.checkpoints.push(createCheckpoint())

    // 古いチェックポイントを削除
    if (this.checkpoints.length > this.maxCheckpoints) {
      this.checkpoints = this.checkpoints.slice(-this.maxCheckpoints)
    }
  }
}
```

**ベストプラクティス**:
- 最大サイズ/数を設定
- 定期的なクリーンアップ処理
- ガベージコレクション可能な設計

### 2.5 圧縮の過度な実行

**問題**: 頻繁に圧縮してコスト増大

```typescript
// ❌ BAD: 毎回圧縮
async function addMessage(msg: Message) {
  messages.push(msg)
  await compress()  // 毎回LLM呼び出し！
}

// ✅ GOOD: しきい値ベース
async function addMessage(msg: Message) {
  messages.push(msg)

  const tokens = countTokens(messages)
  const threshold = maxTokens * 0.75  // 75%で圧縮

  if (tokens >= threshold) {
    await compress()
  }
}
```

**ベストプラクティス**:
- しきい値ベースのトリガー（75-80%）
- 圧縮間隔の最小値を設定（例: 最低10分間隔）
- コスト追跡とアラート

### 2.6 State同期の失敗

**問題**: メッセージとStateが不整合

```typescript
// ❌ BAD: 個別に更新
async function processResponse(response: LLMResponse) {
  await saveMessage(response)
  await updateState(extractState(response))  // 失敗すると不整合
}

// ✅ GOOD: トランザクショナル
async function processResponse(response: LLMResponse) {
  await transaction(async (tx) => {
    await tx.saveMessage(response)
    await tx.updateState(extractState(response))
  })
}
```

**ベストプラクティス**:
- メッセージとStateは一緒に更新
- チェックポイントに両方含める
- 復元時も両方を復元

---

## 3. パフォーマンス最適化

### 3.1 トークンカウントのキャッシュ

```typescript
class CachedTokenCounter {
  private cache = new Map<string, number>()

  count(text: string): number {
    const hash = this.hash(text)

    if (this.cache.has(hash)) {
      return this.cache.get(hash)!
    }

    const tokens = this.actualCount(text)
    this.cache.set(hash, tokens)

    // キャッシュサイズ制限
    if (this.cache.size > 1000) {
      const firstKey = this.cache.keys().next().value
      this.cache.delete(firstKey)
    }

    return tokens
  }

  private hash(text: string): string {
    // Fast hash (e.g., FNV-1a)
    let hash = 2166136261
    for (let i = 0; i < text.length; i++) {
      hash ^= text.charCodeAt(i)
      hash = Math.imul(hash, 16777619)
    }
    return hash.toString(36)
  }

  private actualCount(text: string): number {
    const encoding = encoding_for_model('gpt-4')
    const tokens = encoding.encode(text)
    encoding.free()
    return tokens.length
  }
}
```

### 3.2 遅延読み込み（Lazy Loading）

```typescript
class LazyContextBuilder {
  private _messages: Message[] | null = null
  private _state: LayeredState | null = null

  // メッセージは必要になるまで読み込まない
  private async getMessages(): Promise<Message[]> {
    if (!this._messages) {
      this._messages = await this.storage.loadMessages()
    }
    return this._messages
  }

  // Evidence層だけ必要な場合、他の層は読み込まない
  async buildEvidenceLayer(): Promise<string> {
    const state = await this.getState()
    return this.formatEvidence(state.l4_evidence)
  }
}
```

### 3.3 並列処理

```typescript
class ParallelContextBuilder {
  async buildAllLayers(): Promise<ContextOutput> {
    // 独立した層は並列にビルド
    const [system, task, runtime, memory, evidence, workBuffer] =
      await Promise.all([
        this.buildSystemLayer(),
        this.buildTaskLayer(),
        this.buildRuntimeLayer(),
        this.buildMemoryLayer(),
        this.buildEvidenceLayer(),
        this.buildWorkBufferLayer()
      ])

    return {
      system: this.combineSystem(system, task),
      messages: this.combineMessages(
        runtime,
        memory,
        evidence,
        workBuffer
      )
    }
  }
}
```

### 3.4 インクリメンタルな更新

```typescript
class IncrementalCompressor {
  private lastCompressedIndex = 0

  async compress(messages: Message[]): Promise<Message[]> {
    // 前回圧縮済みの部分はスキップ
    const newMessages = messages.slice(this.lastCompressedIndex)

    if (newMessages.length < 10) {
      return messages  // 少量なら圧縮しない
    }

    const compressed = await this.condense(newMessages)
    this.lastCompressedIndex = messages.length

    return [...messages.slice(0, this.lastCompressedIndex), compressed]
  }
}
```

---

## 4. セキュリティとコンプライアンス

### 4.1 機密情報の保護

```typescript
class SecureContextManager {
  private sensitivePatterns = [
    /\b\d{3}-\d{2}-\d{4}\b/,  // SSN
    /\b\d{16}\b/,              // Credit card
    /sk-[a-zA-Z0-9]{48}/,      // API key
    /password[:\s]+\S+/i       // Password
  ]

  /**
   * メッセージから機密情報を検出・マスク
   */
  sanitize(message: Message): Message {
    return {
      ...message,
      content: message.content.map(c => {
        if (c.type === 'text') {
          return {
            ...c,
            text: this.maskSensitive(c.text)
          }
        }
        return c
      })
    }
  }

  private maskSensitive(text: string): string {
    let masked = text
    for (const pattern of this.sensitivePatterns) {
      masked = masked.replace(pattern, '[REDACTED]')
    }
    return masked
  }

  /**
   * チェックポイント保存前に必ずサニタイズ
   */
  async saveCheckpoint(checkpoint: Checkpoint): Promise<void> {
    const sanitized = {
      ...checkpoint,
      messages: checkpoint.messages.map(m => this.sanitize(m))
    }

    await this.storage.save(sanitized)
  }
}
```

### 4.2 権限チェック

```typescript
class PermissionAwareToolProjector {
  project(state: LayeredState, user: User): Tool[] {
    const allTools = this.getAllTools()

    return allTools.filter(tool => {
      // 1. Phase check
      if (!this.isAllowedInPhase(tool, state.l2_runtime.phase)) {
        return false
      }

      // 2. Permission check
      if (!this.hasPermission(user, tool)) {
        return false
      }

      // 3. Environment check
      if (!this.isAllowedInEnv(tool, state.l2_runtime.environment)) {
        return false
      }

      return true
    })
  }

  private hasPermission(user: User, tool: Tool): boolean {
    const requiredPermissions = tool.requiredPermissions || []
    return requiredPermissions.every(p => user.permissions.includes(p))
  }

  private isAllowedInEnv(tool: Tool, env: Environment): boolean {
    if (tool.destructive && env === 'production') {
      return false  // 本番環境で破壊的操作は禁止
    }
    return true
  }
}
```

### 4.3 監査ログ

```typescript
class AuditLogger {
  /**
   * すべてのツール呼び出しをログ
   */
  async logToolExecution(
    user: User,
    tool: Tool,
    input: any,
    output: any,
    timestamp: number
  ): Promise<void> {
    const entry: AuditEntry = {
      id: crypto.randomUUID(),
      timestamp,
      userId: user.id,
      toolName: tool.name,
      input: this.sanitize(input),
      output: this.sanitize(output),
      success: !output.error,
      ipAddress: user.ipAddress,
      sessionId: user.sessionId
    }

    await this.storage.append(entry)
  }

  /**
   * 機密情報をマスク
   */
  private sanitize(data: any): any {
    // Deep clone and mask sensitive fields
    const cloned = JSON.parse(JSON.stringify(data))
    return this.maskFields(cloned, ['password', 'apiKey', 'token'])
  }

  private maskFields(obj: any, fields: string[]): any {
    if (typeof obj !== 'object' || obj === null) {
      return obj
    }

    for (const key in obj) {
      if (fields.includes(key)) {
        obj[key] = '[REDACTED]'
      } else if (typeof obj[key] === 'object') {
        obj[key] = this.maskFields(obj[key], fields)
      }
    }

    return obj
  }
}
```

### 4.4 GDPR対応（削除の権利）

```typescript
class GDPRCompliantStorage {
  /**
   * ユーザーデータの完全削除
   */
  async deleteUserData(userId: string): Promise<void> {
    // 1. メッセージ削除
    await this.messageStorage.deleteByUser(userId)

    // 2. チェックポイント削除
    await this.checkpointStorage.deleteByUser(userId)

    // 3. 監査ログから個人情報をマスク（ログは残す）
    await this.auditLog.anonymizeUser(userId)

    // 4. キャッシュクリア
    await this.cache.clearUser(userId)

    // 5. 削除ログを記録
    await this.deletionLog.record({
      userId,
      timestamp: Date.now(),
      requestedBy: 'user'
    })
  }

  /**
   * データエクスポート（データポータビリティ）
   */
  async exportUserData(userId: string): Promise<UserDataExport> {
    const [messages, checkpoints, auditLogs] = await Promise.all([
      this.messageStorage.getByUser(userId),
      this.checkpointStorage.getByUser(userId),
      this.auditLog.getByUser(userId)
    ])

    return {
      userId,
      exportedAt: Date.now(),
      messages,
      checkpoints,
      auditLogs
    }
  }
}
```

---

## 5. テスト戦略

### 5.1 ユニットテスト

```typescript
describe('PriorityScorer', () => {
  it('should score user messages higher than assistant', () => {
    const scorer = new PriorityScorer()

    const userMsg: Message = {
      id: '1',
      role: 'user',
      content: [{ type: 'text', text: 'Hello' }]
    }

    const assistantMsg: Message = {
      id: '2',
      role: 'assistant',
      content: [{ type: 'text', text: 'Hi' }]
    }

    const userScore = scorer.scoreMessage(userMsg, createContext())
    const assistantScore = scorer.scoreMessage(assistantMsg, createContext())

    expect(userScore.score).toBeGreaterThan(assistantScore.score)
  })

  it('should preserve tool_use/tool_result pairs', () => {
    const compressor = new PairPreservingCompressor(detector, orchestrator)

    const messages = [
      createUserMessage('Use calculator'),
      createAssistantMessageWithToolUse('calculator', { expr: '1+1' }),
      createToolResultMessage('calculator', '2'),
      createAssistantMessage('The answer is 2')
    ]

    const compressed = await compressor.compressPreservingPairs(messages, 100)

    // ペアは必ず両方残っている
    const hasToolUse = compressed.some(m =>
      m.content.some(c => c.type === 'tool_use')
    )
    const hasToolResult = compressed.some(m =>
      m.content.some(c => c.type === 'tool_result')
    )

    expect(hasToolUse && hasToolResult).toBe(true)
  })
})
```

### 5.2 統合テスト

```typescript
describe('ContextEngine Integration', () => {
  it('should handle full execution flow', async () => {
    const engine = new ModernContextEngine({
      conversationId: 'test-123',
      maxTokens: 200000,
      model: 'claude-opus-4'
    })

    // 1. User input
    const result1 = await engine.executeTask('What is 2+2?')
    expect(result1.success).toBe(true)

    // 2. Follow-up
    const result2 = await engine.executeTask('What about 3+3?')
    expect(result2.success).toBe(true)

    // 3. Verify state
    const state = engine.getState()
    expect(state.l3_memory.shortTerm.length).toBeGreaterThan(0)

    // 4. Verify messages
    const messages = engine.getMessages()
    expect(messages.length).toBeGreaterThan(0)
  })

  it('should compress when threshold exceeded', async () => {
    const engine = new ModernContextEngine({
      conversationId: 'test-456',
      maxTokens: 1000,  // Very small
      model: 'claude-haiku-3-5'
    })

    // Fill with many messages
    for (let i = 0; i < 50; i++) {
      await engine.executeTask(`Message ${i}`)
    }

    // Verify compression occurred
    const messages = engine.getMessages()
    const hasSummary = messages.some(m =>
      m.content.some(c =>
        c.type === 'text' && c.text.includes('Context Summary')
      )
    )

    expect(hasSummary).toBe(true)
  })
})
```

### 5.3 パフォーマンステスト

```typescript
describe('Performance', () => {
  it('should handle 1000 messages efficiently', async () => {
    const messages = Array.from({ length: 1000 }, (_, i) =>
      createMessage(`Message ${i}`)
    )

    const start = Date.now()
    const scorer = new PriorityScorer()

    const scores = messages.map((msg, idx) =>
      scorer.scoreMessage(msg, { currentIndex: 1000, messageIndex: idx })
    )

    const elapsed = Date.now() - start

    expect(elapsed).toBeLessThan(1000)  // < 1 second
    expect(scores.length).toBe(1000)
  })

  it('should cache token counts', async () => {
    const counter = new CachedTokenCounter()
    const text = 'Hello world '.repeat(100)

    // First count (uncached)
    const start1 = Date.now()
    const count1 = counter.count(text)
    const time1 = Date.now() - start1

    // Second count (cached)
    const start2 = Date.now()
    const count2 = counter.count(text)
    const time2 = Date.now() - start2

    expect(count1).toBe(count2)
    expect(time2).toBeLessThan(time1 / 10)  // 10x faster
  })
})
```

### 5.4 Property-based Testing

```typescript
import fc from 'fast-check'

describe('Property-based tests', () => {
  it('compression should always reduce token count', () => {
    fc.assert(
      fc.asyncProperty(
        fc.array(fc.record({
          id: fc.uuid(),
          role: fc.constantFrom('user', 'assistant'),
          content: fc.array(fc.record({
            type: fc.constant('text'),
            text: fc.string({ minLength: 10, maxLength: 100 })
          }))
        }), { minLength: 20, maxLength: 100 }),
        async (messages) => {
          const originalTokens = countTokens(messages)
          const compressed = await compressor.compress(messages, originalTokens * 0.5)

          if (compressed.success) {
            const compressedTokens = countTokens(compressed.messages)
            expect(compressedTokens).toBeLessThanOrEqual(originalTokens)
          }
        }
      )
    )
  })
})
```

---

## 6. モニタリングとデバッグ

### 6.1 メトリクス収集

```typescript
class MetricsCollector {
  private metrics: Metrics = {
    tokenUsage: {
      total: 0,
      byLayer: {},
      utilization: 0
    },
    compression: {
      attempts: 0,
      successes: 0,
      failures: 0,
      avgRatio: 0
    },
    performance: {
      avgLatency: 0,
      p95Latency: 0,
      p99Latency: 0
    },
    errors: {
      total: 0,
      byType: {}
    }
  }

  recordTokenUsage(layer: string, tokens: number): void {
    this.metrics.tokenUsage.total += tokens
    this.metrics.tokenUsage.byLayer[layer] =
      (this.metrics.tokenUsage.byLayer[layer] || 0) + tokens
  }

  recordCompression(success: boolean, ratio: number): void {
    this.metrics.compression.attempts++

    if (success) {
      this.metrics.compression.successes++
      this.updateAvgRatio(ratio)
    } else {
      this.metrics.compression.failures++
    }
  }

  recordLatency(ms: number): void {
    this.latencies.push(ms)
    this.updateLatencyMetrics()
  }

  recordError(error: Error): void {
    this.metrics.errors.total++
    const type = error.constructor.name
    this.metrics.errors.byType[type] =
      (this.metrics.errors.byType[type] || 0) + 1
  }

  getMetrics(): Metrics {
    return { ...this.metrics }
  }

  private latencies: number[] = []

  private updateLatencyMetrics(): void {
    const sorted = this.latencies.slice().sort((a, b) => a - b)
    this.metrics.performance.avgLatency =
      sorted.reduce((a, b) => a + b, 0) / sorted.length
    this.metrics.performance.p95Latency =
      sorted[Math.floor(sorted.length * 0.95)]
    this.metrics.performance.p99Latency =
      sorted[Math.floor(sorted.length * 0.99)]
  }

  private updateAvgRatio(newRatio: number): void {
    const n = this.metrics.compression.successes
    const oldAvg = this.metrics.compression.avgRatio
    this.metrics.compression.avgRatio =
      (oldAvg * (n - 1) + newRatio) / n
  }
}
```

### 6.2 構造化ログ

```typescript
class StructuredLogger {
  log(level: LogLevel, message: string, context: Record<string, any>): void {
    const entry = {
      timestamp: new Date().toISOString(),
      level,
      message,
      ...context,
      // 自動的に追加される情報
      conversationId: this.conversationId,
      userId: this.userId,
      sessionId: this.sessionId
    }

    console.log(JSON.stringify(entry))
  }

  info(message: string, context?: Record<string, any>): void {
    this.log('INFO', message, context || {})
  }

  warn(message: string, context?: Record<string, any>): void {
    this.log('WARN', message, context || {})
  }

  error(message: string, error: Error, context?: Record<string, any>): void {
    this.log('ERROR', message, {
      ...context,
      error: {
        name: error.name,
        message: error.message,
        stack: error.stack
      }
    })
  }

  // 使用例
  logCompression(result: CompressionResult): void {
    this.info('Compression completed', {
      method: result.method,
      success: result.success,
      originalTokens: result.metadata?.originalTokens,
      finalTokens: result.metadata?.finalTokens,
      ratio: result.metadata?.compressionRatio
    })
  }
}
```

### 6.3 トレーシング

```typescript
class DistributedTracer {
  startSpan(operation: string, parent?: Span): Span {
    const span: Span = {
      id: crypto.randomUUID(),
      parentId: parent?.id,
      operation,
      startTime: Date.now(),
      attributes: {}
    }

    this.activeSpans.set(span.id, span)
    return span
  }

  endSpan(span: Span): void {
    span.endTime = Date.now()
    span.duration = span.endTime - span.startTime

    this.activeSpans.delete(span.id)
    this.completedSpans.push(span)

    // Send to tracing backend (e.g., Jaeger, Zipkin)
    this.backend.send(span)
  }

  addAttribute(span: Span, key: string, value: any): void {
    span.attributes[key] = value
  }

  // 使用例
  async executeWithTracing<T>(
    operation: string,
    fn: (span: Span) => Promise<T>
  ): Promise<T> {
    const span = this.startSpan(operation)

    try {
      const result = await fn(span)
      this.addAttribute(span, 'success', true)
      return result
    } catch (error) {
      this.addAttribute(span, 'success', false)
      this.addAttribute(span, 'error', (error as Error).message)
      throw error
    } finally {
      this.endSpan(span)
    }
  }

  private activeSpans = new Map<string, Span>()
  private completedSpans: Span[] = []
}

// 使用例
async function compressWithTracing(messages: Message[]): Promise<Message[]> {
  return tracer.executeWithTracing('compress', async (span) => {
    tracer.addAttribute(span, 'messageCount', messages.length)
    tracer.addAttribute(span, 'tokens', countTokens(messages))

    const result = await compressor.compress(messages, 100000)

    tracer.addAttribute(span, 'compressionRatio', result.metadata?.compressionRatio)

    return result.messages
  })
}
```

---

## 7. プロダクション運用チェックリスト

### 7.1 デプロイ前チェックリスト

#### セキュリティ

- [ ] 機密情報のマスキングが実装されている
- [ ] 権限チェックがすべてのツール呼び出しで実行される
- [ ] 監査ログが記録されている
- [ ] GDPR対応（削除、エクスポート）が実装されている
- [ ] 本番環境で破壊的操作が制限されている

#### パフォーマンス

- [ ] トークンカウントがキャッシュされている
- [ ] 圧縮しきい値が適切に設定されている（75-80%）
- [ ] 並列処理が可能な箇所で使われている
- [ ] メモリリーク対策がされている
- [ ] パフォーマンステストが通っている

#### 信頼性

- [ ] レースコンディション対策がされている
- [ ] tool_use/tool_resultペアが保持される
- [ ] エラーハンドリングとリカバリが実装されている
- [ ] チェックポイントからの復元が可能
- [ ] すべてのユニットテスト・統合テストが通っている

#### 観測可能性

- [ ] メトリクス収集が実装されている
- [ ] 構造化ログが出力されている
- [ ] トレーシングが有効になっている
- [ ] アラートが設定されている
- [ ] ダッシュボードが準備されている

### 7.2 本番監視項目

```typescript
const productionAlerts = {
  // トークン使用率
  tokenUtilization: {
    warning: 0.80,   // 80%で警告
    critical: 0.90   // 90%でクリティカル
  },

  // 圧縮成功率
  compressionSuccessRate: {
    warning: 0.90,   // 90%未満で警告
    critical: 0.80   // 80%未満でクリティカル
  },

  // レイテンシ
  latency: {
    p95Warning: 5000,    // 5秒
    p95Critical: 10000,  // 10秒
    p99Warning: 10000,   // 10秒
    p99Critical: 20000   // 20秒
  },

  // エラー率
  errorRate: {
    warning: 0.01,   // 1%
    critical: 0.05   // 5%
  },

  // メモリ使用量
  memoryUsage: {
    warning: 0.80,   // 80%
    critical: 0.90   // 90%
  }
}
```

### 7.3 インシデント対応手順

```typescript
class IncidentHandler {
  /**
   * トークン超過エラー
   */
  async handleTokenOverflow(): Promise<void> {
    // 1. アグレッシブに圧縮
    await this.forceCompress(0.3)  // 30%まで削減

    // 2. 古いチェックポイントを削除
    await this.cleanupCheckpoints()

    // 3. キャッシュクリア
    await this.clearCache()

    // 4. アラート通知
    await this.notify('Token overflow handled')
  }

  /**
   * 圧縮失敗
   */
  async handleCompressionFailure(): Promise<void> {
    // 1. Truncationにフォールバック
    await this.fallbackToTruncation()

    // 2. エラーログ記録
    this.logger.error('Compression failed, used truncation fallback')

    // 3. メトリクス記録
    this.metrics.recordCompressionFailure()
  }

  /**
   * データ破損検出
   */
  async handleDataCorruption(): Promise<void> {
    // 1. 最新のチェックポイントから復元
    const checkpoint = await this.getLatestCheckpoint()
    await this.restore(checkpoint)

    // 2. 破損データを隔離
    await this.quarantineCorruptedData()

    // 3. クリティカルアラート
    await this.alertCritical('Data corruption detected and recovered')
  }
}
```

### 7.4 段階的ロールアウト

```typescript
class FeatureFlags {
  private flags = {
    useCondensation: {
      enabled: false,
      rolloutPercentage: 0
    },
    useDynamicToolProjection: {
      enabled: false,
      rolloutPercentage: 0
    },
    useLayeredState: {
      enabled: false,
      rolloutPercentage: 0
    }
  }

  isEnabled(feature: string, userId: string): boolean {
    const flag = this.flags[feature]

    if (!flag || !flag.enabled) {
      return false
    }

    // Hash-based rollout
    const hash = this.hashUserId(userId)
    const bucket = hash % 100

    return bucket < flag.rolloutPercentage
  }

  // ロールアウト計画
  async rollout(feature: string): Promise<void> {
    // Phase 1: 5% (1 week)
    await this.setRollout(feature, 5)
    await this.monitor(7 * 24 * 60 * 60 * 1000)

    // Phase 2: 25% (1 week)
    await this.setRollout(feature, 25)
    await this.monitor(7 * 24 * 60 * 60 * 1000)

    // Phase 3: 50% (1 week)
    await this.setRollout(feature, 50)
    await this.monitor(7 * 24 * 60 * 60 * 1000)

    // Phase 4: 100%
    await this.setRollout(feature, 100)
  }

  private async monitor(duration: number): Promise<void> {
    // Monitor metrics and rollback if needed
    const metrics = await this.getMetrics()

    if (metrics.errorRate > 0.05) {
      throw new Error('Error rate too high, aborting rollout')
    }
  }

  private hashUserId(userId: string): number {
    let hash = 0
    for (let i = 0; i < userId.length; i++) {
      hash = ((hash << 5) - hash) + userId.charCodeAt(i)
      hash = hash & hash
    }
    return Math.abs(hash)
  }
}
```

---

## まとめ

このベストプラクティスガイドでは、以下を提供しました：

1. **設計原則**: SOLID、DI、不変性、非同期処理
2. **よくある落とし穴**: トークンカウント、ペア破壊、レースコンディション、メモリリーク、過度な圧縮、State同期
3. **パフォーマンス最適化**: キャッシュ、遅延読み込み、並列処理、インクリメンタル更新
4. **セキュリティ**: 機密情報保護、権限チェック、監査ログ、GDPR対応
5. **テスト戦略**: ユニット、統合、パフォーマンス、Property-based
6. **モニタリング**: メトリクス、ログ、トレーシング
7. **プロダクション運用**: チェックリスト、監視項目、インシデント対応、段階的ロールアウト

これらのプラクティスを適用することで、堅牢で信頼性の高いContext Management システムを本番環境で運用できます。

次のドキュメント [05-integration-examples.md](./05-integration-examples.md) では、実際のフレームワーク（LangGraph、GraphAI、Mulmo）との統合例を示します。
