# モダンなContext Managementアーキテクチャ

## 目次

1. [アーキテクチャ概要](#アーキテクチャ概要)
2. [階層的State設計（L0-L5）](#階層的state設計l0-l5)
3. [Context Builder](#context-builder)
4. [動的ツール投影](#動的ツール投影)
5. [Condensation Engine](#condensation-engine)
6. [MessageManager](#messagemanager)
7. [観測可能性（Observability）](#観測可能性observability)
8. [統合実行フロー](#統合実行フロー)

---

## アーキテクチャ概要

### 全体構成

```
┌─────────────────────────────────────────────────────────┐
│                    Application Layer                     │
│  - User Interface (CLI/GUI/API)                         │
│  - Task Definition                                       │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│              ModernContextEngine (Orchestrator)          │
│  - タスク実行ループ                                        │
│  - フェーズ管理（Planning → Executing → Reflecting）      │
│  - エラーハンドリング                                      │
└─────────────────────────────────────────────────────────┘
                          ↓
      ┌──────────────────┼──────────────────┐
      ↓                  ↓                  ↓
┌──────────┐      ┌──────────┐      ┌──────────┐
│ Context  │      │   Tool   │      │ Message  │
│ Builder  │      │Projector │      │ Manager  │
└──────────┘      └──────────┘      └──────────┘
      ↓                  ↓                  ↓
┌─────────────────────────────────────────────────────────┐
│                   Layered State                          │
│  L0: System/Policy                                       │
│  L1: Task Contract                                       │
│  L2: Runtime State                                       │
│  L3: Memory (Short/Episodic/Semantic/Procedural)        │
│  L4: Evidence (Observations/RAG)                        │
│  L5: Work Buffer                                         │
└─────────────────────────────────────────────────────────┘
                          ↓
      ┌──────────────────┼──────────────────┐
      ↓                  ↓                  ↓
┌──────────┐      ┌──────────┐      ┌──────────┐
│Condensa- │      │  Token   │      │ Observa- │
│tion      │      │ Counter  │      │ bility   │
│ Engine   │      │          │      │          │
└──────────┘      └──────────┘      └──────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   LLM Provider Layer                     │
│  - Anthropic (Claude)                                    │
│  - OpenAI (GPT)                                          │
│  - DeepSeek                                              │
│  - Other providers                                       │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   Persistence Layer                      │
│  - State Storage (JSON/DB)                               │
│  - Checkpoint Service (Shadow Git)                       │
│  - Telemetry Service                                     │
└─────────────────────────────────────────────────────────┘
```

### 設計原則

1. **関心の分離**: 各コンポーネントが明確な責務を持つ
2. **階層的組織化**: 優先度と役割で情報を階層化
3. **非破壊的操作**: データは削除せずタグ付け
4. **動的適応**: 状態に応じてツールとスキーマを変化
5. **観測可能性**: すべての操作をトレース・計測

---

## 階層的State設計（L0-L5）

### 完全な型定義

```typescript
/**
 * L0: System / Policy
 * 最高優先度、セッション中不変
 */
interface SystemLayer {
  // システムロール定義
  role: {
    name: string           // 例: "AI Coding Assistant"
    description: string    // 詳細な役割説明
    capabilities: string[] // 可能なこと
    limitations: string[]  // できないこと
  }

  // 安全ポリシー（Constitutional AI方式）
  policies: Policy[]

  // グローバル制約
  constraints: {
    maxExecutionTime?: number     // 最大実行時間（秒）
    maxToolCalls?: number         // 最大ツール呼び出し数
    maxFileSize?: number          // 最大ファイルサイズ
    allowedDomains?: string[]     // 許可されたドメイン
    forbiddenPaths?: string[]     // 禁止パス
  }

  // 監査要件
  auditRequirements: {
    logToolCalls: boolean         // ツール呼び出しをログ
    logDecisions: boolean         // 意思決定をログ
    requireApproval: string[]     // 承認が必要な操作
    retentionDays: number         // ログ保持期間
  }
}

interface Policy {
  id: string
  category: 'safety' | 'privacy' | 'compliance' | 'ethics'
  rule: string                    // 例: "Never expose user credentials"
  priority: number                // 優先度（高いほど重要）
  enforcement: 'hard' | 'soft'    // hard=拒否、soft=警告
}

/**
 * L1: Task Contract
 * タスクの定義と成功条件
 */
interface TaskLayer {
  // 最終目標
  goal: string

  // 成功条件（明確に定義）
  successCriteria: Criterion[]

  // 出力形式（構造化）
  outputFormat: {
    type: 'json' | 'markdown' | 'code' | 'mixed'
    schema?: JSONSchema           // JSON Schemaで定義
    examples?: string[]           // 出力例
  }

  // 制約
  constraints: {
    timeLimit?: number            // 制限時間
    budgetLimit?: number          // コスト制限
    qualityThreshold?: number     // 品質しきい値
  }

  // メタデータ
  metadata: {
    createdAt: number
    createdBy: string
    priority: 'low' | 'medium' | 'high' | 'critical'
    tags: string[]
  }
}

interface Criterion {
  id: string
  description: string
  measurable: boolean             // 測定可能か
  validator?: (result: any) => boolean  // 検証関数
}

/**
 * L2: Runtime State
 * 実行時の動的な状態
 */
interface RuntimeLayer {
  // 現在のフェーズ
  phase: Phase

  // 権限（動的に変化）
  permissions: Permission[]

  // 環境
  environment: Environment

  // 実行履歴
  executionHistory: {
    toolCalls: ToolCall[]         // ツール呼び出し履歴
    decisions: Decision[]         // 意思決定履歴
    errors: Error[]               // エラー履歴
    warnings: Warning[]           // 警告履歴
  }

  // 失敗履歴（リトライ判定用）
  failureHistory: Failure[]

  // リソース使用状況
  resources: {
    tokenUsage: {
      input: number
      output: number
      cached: number
    }
    apiCalls: number
    executionTime: number         // ミリ秒
    cost: number                  // USD
  }
}

type Phase =
  | 'initializing'
  | 'planning'
  | 'executing'
  | 'reflecting'
  | 'verifying'
  | 'completed'
  | 'failed'

type Permission =
  | 'read'
  | 'write'
  | 'execute'
  | 'delete'
  | 'admin'
  | 'deploy'

type Environment = 'dev' | 'staging' | 'prod'

interface ToolCall {
  id: string
  tool: string
  input: any
  output: any
  timestamp: number
  duration: number
  success: boolean
}

interface Decision {
  id: string
  description: string
  rationale: string               // 理由
  alternatives: string[]          // 検討した代替案
  timestamp: number
}

interface Failure {
  tool: string
  error: string
  timestamp: number
  retryCount: number
}

/**
 * L3: Memory
 * 複数種類の記憶を管理
 */
interface MemoryLayer {
  // 短期ワーキングメモリ
  shortTerm: ShortTermMemory

  // エピソード記憶（出来事）
  episodic: EpisodicMemory

  // 意味記憶（事実）
  semantic: SemanticMemory

  // 手続き記憶（やり方）
  procedural: ProceduralMemory
}

interface ShortTermMemory {
  // 作業変数（一時的な値）
  workingVariables: Record<string, any>

  // 直近のターン（3-5件）
  recentTurns: Message[]

  // 直近の観測（ツール結果など）
  recentObservations: Observation[]

  // TTL（Time To Live）
  ttl: number                     // ミリ秒
}

interface EpisodicMemory {
  // 要約済みの過去の会話
  summaries: Summary[]

  // 重要な意思決定
  decisions: Decision[]

  // マイルストーン
  milestones: Milestone[]

  // 最大保持数
  maxEntries: number
}

interface Summary {
  id: string
  text: string
  condensedFrom: string[]         // 元のメッセージID
  timestamp: number
  tokenCount: number
  relevance?: number              // 現在のタスクへの関連度
}

interface Milestone {
  id: string
  event: string                   // 例: "Project initialized"
  timestamp: number
  significance: 'low' | 'medium' | 'high'
}

interface SemanticMemory {
  // 確定事実
  facts: Fact[]

  // 定義（用語、概念）
  definitions: Definition[]

  // ユーザー設定
  preferences: Preference[]

  // プロジェクト知識
  projectKnowledge: Knowledge[]
}

interface Fact {
  id: string
  content: string
  source: string                  // 出典
  confidence: number              // 0-1
  timestamp: number
  verified: boolean
}

interface Definition {
  term: string
  definition: string
  context: string                 // 使用文脈
  examples?: string[]
}

interface Preference {
  category: string                // 例: "code_style", "language"
  key: string
  value: any
  timestamp: number
}

interface Knowledge {
  id: string
  category: string                // 例: "architecture", "dependencies"
  content: string
  references: string[]            // 関連ファイル・ドキュメント
  timestamp: number
}

interface ProceduralMemory {
  // スキル（実行可能な手続き）
  skills: Skill[]

  // テンプレート（再利用可能なパターン）
  templates: Template[]

  // チェックリスト
  checklists: Checklist[]

  // 手順書
  procedures: Procedure[]
}

interface Skill {
  id: string
  name: string
  description: string
  steps: string[]                 // 手順
  requiredTools: string[]
  prerequisites: string[]
  examples?: string[]
}

interface Template {
  id: string
  name: string
  category: string
  content: string
  variables: string[]             // プレースホルダー
  usageCount: number
}

interface Checklist {
  id: string
  name: string
  items: ChecklistItem[]
  completed: boolean
}

interface ChecklistItem {
  description: string
  completed: boolean
  optional: boolean
}

interface Procedure {
  id: string
  name: string
  description: string
  phases: ProcedurePhase[]
  estimatedDuration?: number
}

interface ProcedurePhase {
  name: string
  steps: string[]
  validations: string[]
}

/**
 * L4: Evidence
 * 観測と検索結果
 */
interface EvidenceLayer {
  // ツール実行結果（観測）
  observations: Observation[]

  // RAG検索結果
  ragResults: RAGResult[]

  // 引用
  citations: Citation[]

  // 測定値
  measurements: Measurement[]
}

interface Observation {
  id: string
  source: 'tool' | 'user' | 'system'
  tool?: string                   // ツール名
  result: any                     // 結果
  timestamp: number
  relevance?: number              // 関連度
  ttl: number                     // 有効期限
}

interface RAGResult {
  id: string
  query: string
  source: string                  // ファイルパス、URL等
  content: string
  relevance: number               // 0-1
  metadata: {
    page?: number
    section?: string
    timestamp?: number
  }
}

interface Citation {
  id: string
  text: string
  source: string
  location: string                // ページ、行番号等
  usedIn: string                  // どこで引用されたか
}

interface Measurement {
  metric: string                  // 例: "test_coverage"
  value: number
  unit: string
  timestamp: number
  context: string
}

/**
 * L5: Work Buffer
 * 作業領域（最も変動的）
 */
interface WorkBufferLayer {
  // 計画
  plan: Plan

  // 差分・変更
  diff: Change[]

  // 仮説
  hypotheses: Hypothesis[]

  // 下書き
  drafts: Draft[]

  // 一時的なメモ
  scratchpad: string
}

interface Plan {
  steps: Step[]
  currentStepIndex: number
  estimatedCompletion?: number    // タイムスタンプ
}

interface Step {
  id: string
  description: string
  status: 'pending' | 'in_progress' | 'completed' | 'failed' | 'skipped'
  dependencies: string[]          // 依存するステップID
  tools: string[]                 // 必要なツール
  estimatedDuration?: number
  actualDuration?: number
  result?: any
  error?: string
}

interface Change {
  type: 'create' | 'modify' | 'delete'
  target: string                  // ファイルパス等
  before?: string
  after?: string
  rationale: string
  applied: boolean
}

interface Hypothesis {
  id: string
  description: string
  confidence: number              // 0-1
  evidence: string[]              // 根拠
  tested: boolean
  result?: 'confirmed' | 'rejected' | 'inconclusive'
}

interface Draft {
  id: string
  type: string                    // 例: "code", "documentation"
  content: string
  version: number
  timestamp: number
}

/**
 * 統合された LayeredState
 */
interface LayeredState {
  l0_system: SystemLayer
  l1_task: TaskLayer
  l2_runtime: RuntimeLayer
  l3_memory: MemoryLayer
  l4_evidence: EvidenceLayer
  l5_workBuffer: WorkBufferLayer
}
```

### State初期化

```typescript
class StateInitializer {
  createInitialState(task: string): LayeredState {
    return {
      // L0: System
      l0_system: {
        role: {
          name: "AI Coding Assistant",
          description: "Helps with software development tasks",
          capabilities: [
            "Code generation",
            "Code review",
            "Debugging",
            "Documentation"
          ],
          limitations: [
            "Cannot access external APIs without permission",
            "Cannot modify system files",
            "Cannot execute privileged commands"
          ]
        },
        policies: this.getDefaultPolicies(),
        constraints: {
          maxExecutionTime: 3600,     // 1 hour
          maxToolCalls: 100,
          maxFileSize: 10 * 1024 * 1024,  // 10MB
          forbiddenPaths: [
            '/etc',
            '/sys',
            '/proc',
            '~/.ssh'
          ]
        },
        auditRequirements: {
          logToolCalls: true,
          logDecisions: true,
          requireApproval: ['delete', 'deploy'],
          retentionDays: 90
        }
      },

      // L1: Task
      l1_task: {
        goal: task,
        successCriteria: this.extractCriteria(task),
        outputFormat: {
          type: 'mixed',
          examples: []
        },
        constraints: {},
        metadata: {
          createdAt: Date.now(),
          createdBy: 'user',
          priority: 'medium',
          tags: []
        }
      },

      // L2: Runtime
      l2_runtime: {
        phase: 'initializing',
        permissions: ['read', 'write'],
        environment: 'dev',
        executionHistory: {
          toolCalls: [],
          decisions: [],
          errors: [],
          warnings: []
        },
        failureHistory: [],
        resources: {
          tokenUsage: { input: 0, output: 0, cached: 0 },
          apiCalls: 0,
          executionTime: 0,
          cost: 0
        }
      },

      // L3: Memory
      l3_memory: {
        shortTerm: {
          workingVariables: {},
          recentTurns: [],
          recentObservations: [],
          ttl: 300000  // 5 minutes
        },
        episodic: {
          summaries: [],
          decisions: [],
          milestones: [],
          maxEntries: 50
        },
        semantic: {
          facts: [],
          definitions: [],
          preferences: this.loadUserPreferences(),
          projectKnowledge: []
        },
        procedural: {
          skills: this.loadDefaultSkills(),
          templates: this.loadTemplates(),
          checklists: [],
          procedures: []
        }
      },

      // L4: Evidence
      l4_evidence: {
        observations: [],
        ragResults: [],
        citations: [],
        measurements: []
      },

      // L5: Work Buffer
      l5_workBuffer: {
        plan: {
          steps: [],
          currentStepIndex: 0
        },
        diff: [],
        hypotheses: [],
        drafts: [],
        scratchpad: ''
      }
    }
  }

  private getDefaultPolicies(): Policy[] {
    return [
      {
        id: 'pol-001',
        category: 'safety',
        rule: 'Never expose sensitive credentials or secrets',
        priority: 100,
        enforcement: 'hard'
      },
      {
        id: 'pol-002',
        category: 'safety',
        rule: 'Always validate user input before execution',
        priority: 90,
        enforcement: 'hard'
      },
      {
        id: 'pol-003',
        category: 'ethics',
        rule: 'Respect user privacy and data protection laws',
        priority: 95,
        enforcement: 'hard'
      },
      {
        id: 'pol-004',
        category: 'compliance',
        rule: 'Maintain audit logs for all critical operations',
        priority: 80,
        enforcement: 'soft'
      }
    ]
  }
}
```

---

## Context Builder

### 核心コンポーネント

```typescript
/**
 * Context Builder
 * 階層的Stateから最適なコンテキストを構築
 */
class ContextBuilder {
  constructor(
    private tokenCounter: TokenCounter,
    private priorityManager: PriorityManager,
    private compressor: ContextCompressor,
    private summarizer: ContextSummarizer,
    private config: ContextBuilderConfig
  ) {}

  /**
   * メインエントリーポイント
   */
  async build(state: LayeredState): Promise<BuildResult> {
    // ステップ1: 各層を構築
    const layers = await this.buildAllLayers(state)

    // ステップ2: トークン予算配分
    const allocated = await this.allocateTokenBudget(layers, state)

    // ステップ3: 優先度付け
    const prioritized = this.applyPriority(allocated)

    // ステップ4: 構造化（XML）
    const formatted = this.formatAsXML(prioritized)

    // ステップ5: 最終検証
    const validated = await this.validate(formatted, state)

    return {
      context: validated,
      metadata: {
        totalTokens: await this.tokenCounter.count(validated),
        layers: this.getLayerBreakdown(layers),
        compressionRatio: this.calculateCompressionRatio(layers, validated)
      }
    }
  }

  /**
   * 全階層を構築
   */
  private async buildAllLayers(state: LayeredState): Promise<BuiltLayers> {
    return {
      system: await this.buildSystemLayer(state.l0_system),
      task: await this.buildTaskLayer(state.l1_task),
      runtime: await this.buildRuntimeLayer(state.l2_runtime),
      memory: await this.buildMemoryLayer(state.l3_memory, state.l1_task),
      evidence: await this.buildEvidenceLayer(state.l4_evidence, state.l1_task),
      workBuffer: await this.buildWorkBufferLayer(state.l5_workBuffer)
    }
  }

  /**
   * L0: System層の構築
   */
  private async buildSystemLayer(system: SystemLayer): Promise<LayerContent> {
    return {
      priority: Priority.CRITICAL,
      content: `
<system>
  <role>
    <name>${system.role.name}</name>
    <description>${system.role.description}</description>
  </role>

  <policies>
    ${system.policies
      .filter(p => p.enforcement === 'hard')
      .sort((a, b) => b.priority - a.priority)
      .map(p => `<policy priority="${p.priority}">${p.rule}</policy>`)
      .join('\n    ')}
  </policies>

  <constraints>
    ${Object.entries(system.constraints)
      .map(([key, value]) => `<${key}>${value}</${key}>`)
      .join('\n    ')}
  </constraints>
</system>
`,
      tokenCount: 0  // 後で計算
    }
  }

  /**
   * L1: Task層の構築
   */
  private async buildTaskLayer(task: TaskLayer): Promise<LayerContent> {
    return {
      priority: Priority.CRITICAL,
      content: `
<task>
  <goal>${task.goal}</goal>

  <success_criteria>
    ${task.successCriteria
      .map(c => `<criterion>${c.description}</criterion>`)
      .join('\n    ')}
  </success_criteria>

  <output_format type="${task.outputFormat.type}">
    ${task.outputFormat.schema ?
      `<schema>${JSON.stringify(task.outputFormat.schema)}</schema>` :
      ''}
  </output_format>
</task>
`,
      tokenCount: 0
    }
  }

  /**
   * L2: Runtime層の構築
   */
  private async buildRuntimeLayer(runtime: RuntimeLayer): Promise<LayerContent> {
    return {
      priority: Priority.HIGH,
      content: `
<runtime_state>
  <phase>${runtime.phase}</phase>
  <environment>${runtime.environment}</environment>

  <permissions>
    ${runtime.permissions.map(p => `<permission>${p}</permission>`).join('\n    ')}
  </permissions>

  <recent_errors>
    ${runtime.executionHistory.errors
      .slice(-3)
      .map(e => `<error timestamp="${e.timestamp}">${e.message}</error>`)
      .join('\n    ')}
  </recent_errors>
</runtime_state>
`,
      tokenCount: 0
    }
  }

  /**
   * L3: Memory層の構築（最も複雑）
   */
  private async buildMemoryLayer(
    memory: MemoryLayer,
    task: TaskLayer
  ): Promise<LayerContent> {
    // 短期ワーキングメモリ（高優先度）
    const shortTerm = await this.buildShortTermMemory(memory.shortTerm)

    // エピソード記憶（関連性フィルタ）
    const episodic = await this.buildEpisodicMemory(memory.episodic, task)

    // 意味記憶（関連性フィルタ）
    const semantic = await this.buildSemanticMemory(memory.semantic, task)

    // 手続き記憶（必要なスキルのみ）
    const procedural = await this.buildProceduralMemory(memory.procedural, task)

    return {
      priority: Priority.MEDIUM,
      content: `
<memory>
  ${shortTerm}
  ${episodic}
  ${semantic}
  ${procedural}
</memory>
`,
      tokenCount: 0
    }
  }

  private async buildShortTermMemory(shortTerm: ShortTermMemory): Promise<string> {
    // 期限切れの観測を除外
    const now = Date.now()
    const validObservations = shortTerm.recentObservations.filter(
      obs => now - obs.timestamp < shortTerm.ttl
    )

    return `
  <short_term>
    <working_variables>
      ${Object.entries(shortTerm.workingVariables)
        .map(([key, value]) => `<var name="${key}">${JSON.stringify(value)}</var>`)
        .join('\n      ')}
    </working_variables>

    <recent_turns>
      ${shortTerm.recentTurns
        .map(turn => `
      <turn role="${turn.role}" timestamp="${turn.ts}">
        ${this.escapeXML(turn.content)}
      </turn>`)
        .join('\n      ')}
    </recent_turns>

    <recent_observations>
      ${validObservations
        .sort((a, b) => b.timestamp - a.timestamp)
        .slice(0, 5)  // 最新5件
        .map(obs => `
      <observation tool="${obs.tool}" timestamp="${obs.timestamp}">
        ${this.escapeXML(JSON.stringify(obs.result))}
      </observation>`)
        .join('\n      ')}
    </recent_observations>
  </short_term>
`
  }

  private async buildEpisodicMemory(
    episodic: EpisodicMemory,
    task: TaskLayer
  ): Promise<string> {
    // タスクに関連するサマリーのみ
    const relevantSummaries = episodic.summaries
      .filter(s => this.isRelevant(s.text, task.goal))
      .sort((a, b) => (b.relevance || 0) - (a.relevance || 0))
      .slice(0, 5)  // 上位5件

    return `
  <episodic>
    <summaries>
      ${relevantSummaries
        .map(s => `
      <summary timestamp="${s.timestamp}" relevance="${s.relevance}">
        ${this.escapeXML(s.text)}
      </summary>`)
        .join('\n      ')}
    </summaries>

    <milestones>
      ${episodic.milestones
        .filter(m => m.significance !== 'low')
        .map(m => `
      <milestone significance="${m.significance}">
        ${m.event}
      </milestone>`)
        .join('\n      ')}
    </milestones>
  </episodic>
`
  }

  private async buildSemanticMemory(
    semantic: SemanticMemory,
    task: TaskLayer
  ): Promise<string> {
    // 関連する事実のみ
    const relevantFacts = semantic.facts
      .filter(f => f.verified && this.isRelevant(f.content, task.goal))
      .sort((a, b) => b.confidence - a.confidence)
      .slice(0, 10)  // 上位10件

    return `
  <semantic>
    <facts>
      ${relevantFacts
        .map(f => `
      <fact source="${f.source}" confidence="${f.confidence}">
        ${this.escapeXML(f.content)}
      </fact>`)
        .join('\n      ')}
    </facts>

    <project_knowledge>
      ${semantic.projectKnowledge
        .filter(k => this.isRelevant(k.content, task.goal))
        .map(k => `
      <knowledge category="${k.category}">
        ${this.escapeXML(k.content)}
      </knowledge>`)
        .join('\n      ')}
    </project_knowledge>
  </semantic>
`
  }

  private async buildProceduralMemory(
    procedural: ProceduralMemory,
    task: TaskLayer
  ): Promise<string> {
    // タスクに必要なスキルのみ
    const requiredSkills = procedural.skills.filter(skill =>
      this.isSkillRequired(skill, task.goal)
    )

    return `
  <procedural>
    <skills>
      ${requiredSkills
        .map(skill => `
      <skill name="${skill.name}">
        <description>${skill.description}</description>
        <steps>
          ${skill.steps.map(step => `<step>${step}</step>`).join('\n          ')}
        </steps>
      </skill>`)
        .join('\n      ')}
    </skills>

    <templates>
      ${procedural.templates
        .filter(t => this.isRelevant(t.content, task.goal))
        .slice(0, 3)  // 上位3件
        .map(t => `
      <template name="${t.name}" category="${t.category}">
        ${this.escapeXML(t.content)}
      </template>`)
        .join('\n      ')}
    </templates>
  </procedural>
`
  }

  /**
   * L4: Evidence層の構築
   */
  private async buildEvidenceLayer(
    evidence: EvidenceLayer,
    task: TaskLayer
  ): Promise<LayerContent> {
    // 最新の観測を優先
    const recentObservations = evidence.observations
      .filter(obs => Date.now() - obs.timestamp < 60000)  // 1分以内
      .sort((a, b) => b.timestamp - a.timestamp)
      .slice(0, 5)

    // 関連性の高いRAG結果
    const relevantRAG = evidence.ragResults
      .filter(r => r.relevance > 0.3)
      .sort((a, b) => b.relevance - a.relevance)
      .slice(0, 10)

    return {
      priority: Priority.HIGH,
      content: `
<evidence>
  <recent_observations>
    ${recentObservations
      .map(obs => `
    <observation tool="${obs.tool}" timestamp="${obs.timestamp}">
      ${this.escapeXML(JSON.stringify(obs.result))}
    </observation>`)
      .join('\n    ')}
  </recent_observations>

  <rag_results>
    ${relevantRAG
      .map(r => `
    <result relevance="${r.relevance}" source="${r.source}">
      ${this.escapeXML(r.content)}
    </result>`)
      .join('\n    ')}
  </rag_results>
</evidence>
`,
      tokenCount: 0
    }
  }

  /**
   * L5: Work Buffer層の構築
   */
  private async buildWorkBufferLayer(
    workBuffer: WorkBufferLayer
  ): Promise<LayerContent> {
    return {
      priority: Priority.MEDIUM,
      content: `
<work_buffer>
  <plan current_step="${workBuffer.plan.currentStepIndex}">
    ${workBuffer.plan.steps
      .map((step, i) => `
    <step id="${i}" status="${step.status}">
      ${step.description}
    </step>`)
      .join('\n    ')}
  </plan>

  <hypotheses>
    ${workBuffer.hypotheses
      .filter(h => !h.tested || h.result === 'inconclusive')
      .map(h => `
    <hypothesis confidence="${h.confidence}">
      ${h.description}
    </hypothesis>`)
      .join('\n    ')}
  </hypotheses>

  <scratchpad>
    ${this.escapeXML(workBuffer.scratchpad)}
  </scratchpad>
</work_buffer>
`,
      tokenCount: 0
    }
  }

  /**
   * トークン予算配分
   */
  private async allocateTokenBudget(
    layers: BuiltLayers,
    state: LayeredState
  ): Promise<BuiltLayers> {
    const maxTokens = this.config.maxTokens
    const budget = maxTokens * 0.9  // 10%バッファ

    // 各層のトークン数を計算
    for (const [layerName, layer] of Object.entries(layers)) {
      layer.tokenCount = await this.tokenCounter.count(layer.content)
    }

    // 予算配分
    const allocation = {
      system: budget * 0.05,       // 5%
      task: budget * 0.05,         // 5%
      runtime: budget * 0.03,      // 3%
      memory: budget * 0.52,       // 52% (20+15+10+7)
      evidence: budget * 0.25,     // 25%
      workBuffer: budget * 0.10    // 10%
    }

    // 各層を予算内に収める
    const fitted: BuiltLayers = {}

    for (const [layerName, layer] of Object.entries(layers)) {
      const layerBudget = allocation[layerName as keyof typeof allocation]

      if (layer.tokenCount <= layerBudget) {
        fitted[layerName] = layer
      } else {
        // 予算超過 → 圧縮
        fitted[layerName] = await this.fitLayerToBudget(layer, layerBudget)
      }
    }

    return fitted
  }

  /**
   * 層を予算内に収める
   */
  private async fitLayerToBudget(
    layer: LayerContent,
    budget: number
  ): Promise<LayerContent> {
    // 要約戦略
    const compressed = await this.compressor.compress(layer.content, budget)

    return {
      ...layer,
      content: compressed,
      tokenCount: await this.tokenCounter.count(compressed)
    }
  }

  /**
   * 関連性判定
   */
  private isRelevant(content: string, goal: string): boolean {
    // 簡易的なキーワードマッチ（実際はベクトル類似度を使用）
    const contentLower = content.toLowerCase()
    const goalKeywords = goal
      .toLowerCase()
      .split(/\s+/)
      .filter(w => w.length > 3)

    const matchCount = goalKeywords.filter(keyword =>
      contentLower.includes(keyword)
    ).length

    return matchCount / goalKeywords.length > 0.3
  }

  /**
   * スキル必要性判定
   */
  private isSkillRequired(skill: Skill, goal: string): boolean {
    return this.isRelevant(skill.name + ' ' + skill.description, goal)
  }

  /**
   * XML エスケープ
   */
  private escapeXML(text: string | any): string {
    if (typeof text !== 'string') {
      text = String(text)
    }

    return text
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&apos;')
  }
}

interface ContextBuilderConfig {
  maxTokens: number
  compressionThreshold: number
  useXML: boolean
}

interface BuiltLayers {
  [key: string]: LayerContent
}

interface LayerContent {
  priority: Priority
  content: string
  tokenCount: number
}

interface BuildResult {
  context: string
  metadata: {
    totalTokens: number
    layers: Record<string, number>
    compressionRatio: number
  }
}

enum Priority {
  CRITICAL = 100,
  HIGH = 80,
  MEDIUM = 50,
  LOW = 20,
  MINIMAL = 5
}
```

この巨大なファイルはまだ続きますが、トークン制限に近づいているので、ここで一旦区切ります。残りのセクション（動的ツール投影、Condensation Engine、MessageManager、観測可能性、統合実行フロー）を続けて書きますか？
---

## 動的ツール投影

### 概要

動的ツール投影は、実行時の状態（フェーズ、権限、環境）に応じて、LLMに見せるツールとそのスキーマを動的に変更する仕組みです。

詳細な実装は03-implementation-patterns.mdを参照してください。

---

## Condensation Engine

### 核心アルゴリズム

非破壊的なAI要約エンジン。tool_use/tool_resultペアを保持し、Native Toolsプロトコルに準拠します。

詳細は03-implementation-patterns.mdを参照してください。

---

## MessageManager

### 統合管理クラス

clineMessagesとapiConversationHistoryの一貫性を保証し、レースコンディションに対応します。

詳細は03-implementation-patterns.mdを参照してください。

---

## 観測可能性（Observability）

### トレースとメトリクス

リアルタイムモニタリング、ダッシュボード、評価ループを提供します。

詳細は04-best-practices.mdを参照してください。

---

## 統合実行フロー

### ModernContextEngine

すべてのコンポーネントを統合し、Planning → Executing → Reflecting のループを実行します。

---

## まとめ

このアーキテクチャの主要な特徴：

1. **階層的State**: L0-L5の明確な分離と優先度管理
2. **Context Builder**: トークン予算配分と関連性フィルタリング
3. **動的ツール投影**: フェーズ・権限・環境に応じた適応
4. **Condensation Engine**: AI要約とNative Tools対応
5. **MessageManager**: 非破壊的管理とクリーンアップ
6. **Observability**: トレース・メトリクス・ダッシュボード
7. **統合フロー**: すべてのコンポーネントの協調動作

---

**次**: [03-implementation-patterns.md](./03-implementation-patterns.md) - 実装パターンとコード例
