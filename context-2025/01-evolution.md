# Context Management の進化 (2023-2024 → 2025)

## 目次

1. [はじめに](#はじめに)
2. [1年前のシンプルなアプローチ](#1年前のシンプルなアプローチ)
3. [7つの主要な進化](#7つの主要な進化)
4. [なぜこの進化が必要だったか](#なぜこの進化が必要だったか)
5. [実際の影響](#実際の影響)

---

## はじめに

2023-2024年のLLMアプリケーションは、比較的単純なContext Management（コンテキスト管理）で動作していました。しかし、以下の課題が顕在化しました：

- **長期セッションでの情報損失**: 古いメッセージを削除すると重要な文脈が失われる
- **トークン制限への機械的対応**: しきい値を超えたら先頭から削除するだけ
- **復元不可能**: 一度削除したデータは戻せない
- **ツールの無秩序**: すべてのツールが常に見える状態
- **プロトコル非対応**: Native Toolsなどの新しい仕様に対応できない

2024年後半から2025年にかけて、**Context Engineering**という概念が確立され、より高度で柔軟なアプローチが標準になりました。

このドキュメントでは、この1年間の進化を7つのポイントに分けて詳しく解説します。

---

## 1年前のシンプルなアプローチ

### アーキテクチャ

```typescript
// 2023-2024年の典型的な実装
interface OldContextManagement {
  // チャットログをそのまま配列で保持
  messages: Message[]

  // 利用可能なツール一覧（静的）
  tools: Tool[]

  // 設定
  config: {
    maxMessages: number      // 最大メッセージ数（例: 100）
    contextWindow: number    // トークン制限（例: 4096）
  }
}

class SimpleContextManager {
  private messages: Message[] = []

  // メッセージ追加
  addMessage(message: Message) {
    this.messages.push(message)

    // 上限チェック（単純）
    if (this.messages.length > this.config.maxMessages) {
      // 古いものから削除
      this.messages.shift()  // ❌ 永久に失われる
    }
  }

  // API送信用
  getMessages(): Message[] {
    return this.messages  // そのまま返す
  }

  // ツール取得
  getTools(): Tool[] {
    return this.tools  // 常に同じ
  }
}
```

### 問題点

#### 1. 情報の永久的な損失

```typescript
// メッセージが100件を超えると...
messages.length = 101

// 最初のメッセージを削除
messages.shift()  // ❌ このメッセージは永久に失われる

// 結果: 重要な文脈情報が消える
// - プロジェクトの初期要件
// - ユーザーの重要な指示
// - 過去の決定事項
```

#### 2. トークンカウントの不正確さ

```typescript
// 単純なカウント
const tokenCount = messages.reduce(
  (sum, msg) => sum + msg.content.length / 4,  // ❌ 粗い推定
  0
)

// 問題:
// - 実際のトークン数と大きくずれる
// - tool_use/tool_resultのトークン数を考慮していない
// - 画像のトークン数が不正確
```

#### 3. 復元不可能

```typescript
// 編集や削除
messages = messages.filter(msg => msg.id !== deleteId)  // ❌

// 問題:
// - 誤って削除したら戻せない
// - チェックポイント機能を実装できない
// - デバッグが困難
```

#### 4. ツールの無秩序

```typescript
const tools = [
  readFileTool,
  writeFileTool,
  deleteFileTool,     // ❌ 常に見える（危険）
  executeTool,        // ❌ 常に見える（危険）
  searchTool,
  analyzeTool
]

// 問題:
// - フェーズに関係なく全ツールが見える
// - 権限チェックなし
// - 誤操作のリスク
// - トークンの無駄
```

---

## 7つの主要な進化

### 進化1: 非破壊的管理 → データ損失ゼロ

#### Before: 物理削除

```typescript
// 古い方法: 削除したら戻せない
class OldManager {
  deleteOldMessages(cutoff: number) {
    this.messages = this.messages.filter(
      msg => msg.timestamp > cutoff
    )
    // ❌ 削除されたメッセージは永久に失われる
  }
}
```

#### After: タグ付け管理

```typescript
// 新しい方法（Roo Code方式）
interface ApiMessage {
  role: "user" | "assistant"
  content: string | ContentBlock[]
  ts: number

  // 非破壊的なタグ
  condenseParent?: string      // このサマリーに置き換えられた
  truncationParent?: string    // このマーカーで隠された
  isSummary?: boolean          // サマリーメッセージか
  isTruncationMarker?: boolean // トランケーションマーカーか
  condenseId?: string          // サマリーの一意ID
  truncationId?: string        // マーカーの一意ID
}

class ModernManager {
  // メッセージは削除せずタグ付け
  hideMessages(messagesToHide: Message[], markerId: string) {
    return this.messages.map(msg => {
      if (messagesToHide.includes(msg)) {
        // タグ付けするだけ（削除しない）
        return {
          ...msg,
          truncationParent: markerId  // ✅ タグ付け
        }
      }
      return msg
    })
  }

  // API送信時にフィルタリング
  getEffectiveHistory(messages: ApiMessage[]): ApiMessage[] {
    // 存在するマーカーIDを収集
    const existingMarkers = new Set(
      messages
        .filter(m => m.isTruncationMarker && m.truncationId)
        .map(m => m.truncationId)
    )

    // truncationParentを持ち、対応するマーカーが存在する場合のみ除外
    return messages.filter(msg => {
      if (msg.truncationParent) {
        return !existingMarkers.has(msg.truncationParent)
      }
      return true  // それ以外は含める
    })
  }

  // 復元（マーカー削除でタグをクリーンアップ）
  restoreMessages(markerId: string) {
    // マーカーを削除
    const withoutMarker = this.messages.filter(
      msg => !(msg.isTruncationMarker && msg.truncationId === markerId)
    )

    // 孤立したタグをクリーンアップ
    return withoutMarker.map(msg => {
      if (msg.truncationParent === markerId) {
        const { truncationParent, ...rest } = msg
        return rest  // ✅ タグを削除して復元
      }
      return msg
    })
  }
}
```

#### 具体例: マーカーによる隠蔽

```typescript
// 初期状態
const messages = [
  { id: 1, role: "user", content: "Hello" },
  { id: 2, role: "assistant", content: "Hi!" },
  { id: 3, role: "user", content: "How are you?" },
  { id: 4, role: "assistant", content: "I'm good!" },
  { id: 5, role: "user", content: "Great!" }
]

// トランケーション実行（50%削除）
const truncationId = "uuid-abc-123"
const tagged = [
  { id: 1, role: "user", content: "Hello" },  // 最初は保持
  { id: 2, role: "assistant", content: "Hi!", truncationParent: "uuid-abc-123" },  // タグ
  { id: 3, role: "user", content: "How are you?", truncationParent: "uuid-abc-123" },  // タグ
  {
    role: "user",
    content: "[Truncation: 2 messages hidden]",
    isTruncationMarker: true,
    truncationId: "uuid-abc-123",
    ts: messages[3].ts - 1  // 保持メッセージの直前
  },  // マーカー挿入
  { id: 4, role: "assistant", content: "I'm good!" },
  { id: 5, role: "user", content: "Great!" }
]

// API送信時
const effective = getEffectiveHistory(tagged)
// => [
//   { id: 1, ... },
//   { role: "user", content: "[Truncation: 2 messages hidden]", ... },
//   { id: 4, ... },
//   { id: 5, ... }
// ]

// 復元（マーカー削除）
const restored = restoreMessages("uuid-abc-123")
// => 元の5件すべてが復元される ✅
```

#### 利点

✅ **データ損失なし**: すべてのメッセージが保持される
✅ **復元可能**: いつでも過去の状態に戻せる
✅ **チェックポイント統合**: Shadow Gitと完全に統合可能
✅ **デバッグ容易**: 全履歴を確認できる
✅ **監査証跡**: すべての操作を追跡可能

---

### 進化2: 単純削除 → 知的圧縮（Condensation）

#### Before: 機械的削除

```typescript
// 古い方法
class OldCompressor {
  compress(messages: Message[], limit: number) {
    const tokenCount = this.countTokens(messages)

    if (tokenCount > limit) {
      // 先頭から削除
      const toRemove = Math.floor(messages.length * 0.3)  // 30%削除
      return messages.slice(toRemove)  // ❌ 機械的に削除
    }

    return messages
  }
}

// 問題:
// - 重要な情報も容赦なく削除
// - 文脈が分断される
// - 情報が完全に失われる
```

#### After: AI要約 + フォールバック

```typescript
// 新しい方法（二段階アプローチ）
class ModernCompressor {
  async manageContext(
    messages: ApiMessage[],
    tokenCount: number,
    contextWindow: number
  ): Promise<ContextResult> {
    const threshold = contextWindow * 0.75  // 75%

    if (tokenCount < threshold) {
      return { messages }  // 何もしない
    }

    // 第1段階: Condensation（AI要約）
    if (this.config.autoCondenseContext) {
      try {
        const result = await this.condense(messages)

        // 成功チェック
        if (result.newTokens < tokenCount) {
          return {
            messages: result.messages,
            summary: result.summary,
            condenseId: result.condenseId,
            reduction: tokenCount - result.newTokens
          }
        }
      } catch (error) {
        console.warn("Condensation failed, falling back to truncation")
        // フォールスルー
      }
    }

    // 第2段階: Truncation（スライディングウィンドウ）
    return this.truncate(messages, 0.5)  // 50%削除
  }

  private async condense(
    messages: ApiMessage[]
  ): Promise<CondenseResult> {
    // 保持するメッセージ（最新3件）
    const keepMessages = messages.slice(-3)

    // 要約対象
    const toSummarize = messages.slice(0, -3)

    if (toSummarize.length === 0) {
      throw new Error("Not enough messages to summarize")
    }

    // LLMで要約生成
    const summary = await this.llm.generate(`
以下の会話を要約してください。

重要な情報:
1. Previous Conversation: 会話の流れ
2. Current Work: 現在作業していたこと
3. Key Technical Concepts: 技術的な概念
4. Relevant Files and Code: 関連ファイル
5. Problem Solving: 解決した問題
6. Pending Tasks: 未完了のタスク

会話履歴:
${toSummarize.map(m => \`\${m.role}: \${m.content}\`).join('\\n')}

500語以内で要約してください。
`)

    // サマリーメッセージ作成
    const condenseId = crypto.randomUUID()
    const summaryMessage: ApiMessage = {
      role: "assistant",
      content: [
        {
          type: "reasoning",
          text: "Condensing conversation context. The summary captures key information."
        },
        {
          type: "text",
          text: summary
        }
      ],
      ts: keepMessages[0].ts - 1,  // 保持メッセージの直前
      isSummary: true,
      condenseId
    }

    // 中間メッセージにタグ付け
    const tagged = messages.map((msg, i) => {
      if (i === 0) return msg  // 最初は保持
      if (i >= messages.length - 3) return msg  // 最後3件は保持

      // 中間はタグ付け
      return {
        ...msg,
        condenseParent: condenseId
      }
    })

    // サマリー挿入
    const result = [
      tagged[0],  // 最初のメッセージ
      ...tagged.slice(1, -3),  // タグ付きの中間メッセージ
      summaryMessage,  // サマリー
      ...keepMessages  // 最新3件
    ]

    // トークン数計算
    const newTokens = await this.countTokens(result)

    return {
      messages: result,
      summary,
      condenseId,
      newTokens
    }
  }
}
```

#### 具体例: Condensation の動作

```typescript
// 初期状態（12メッセージ）
const messages = [
  { id: 0, role: "user", content: "プロジェクト初期化して" },
  { id: 1, role: "assistant", content: "npm init実行しました" },
  { id: 2, role: "user", content: "Reactをインストール" },
  { id: 3, role: "assistant", content: "インストール完了" },
  { id: 4, role: "user", content: "App.tsxを作成" },
  { id: 5, role: "assistant", content: "作成しました" },
  { id: 6, role: "user", content: "ボタンコンポーネント追加" },
  { id: 7, role: "assistant", content: "Button.tsxを作成" },
  { id: 8, role: "user", content: "スタイル調整" },
  { id: 9, role: "assistant", content: "CSSを更新" },
  { id: 10, role: "user", content: "ビルド実行" },  // 最新3件
  { id: 11, role: "assistant", content: "成功しました" }  // 最新3件
]

// Condensation実行後
const condensed = [
  { id: 0, role: "user", content: "プロジェクト初期化して" },  // 最初は保持

  // 中間メッセージ（タグ付き）
  { id: 1, role: "assistant", content: "...", condenseParent: "uuid-123" },
  { id: 2, role: "user", content: "...", condenseParent: "uuid-123" },
  // ... id: 3-8 すべて condenseParent: "uuid-123"
  { id: 9, role: "assistant", content: "...", condenseParent: "uuid-123" },

  // サマリーメッセージ
  {
    role: "assistant",
    content: [
      { type: "reasoning", text: "Condensing conversation context..." },
      {
        type: "text",
        text: `
Previous Conversation: ユーザーがReactプロジェクトのセットアップを依頼
Current Work: UIコンポーネント（Button.tsx）の作成とスタイリングを完了
Key Technical Concepts: React, TypeScript, CSS
Relevant Files:
  - package.json: 依存関係
  - App.tsx: メインコンポーネント
  - Button.tsx: ボタンコンポーネント
Problem Solving: プロジェクト初期化からコンポーネント作成まで順調に完了
Pending Tasks: なし（ビルド実行予定）
`
      }
    ],
    isSummary: true,
    condenseId: "uuid-123"
  },

  // 最新3件（保持）
  { id: 10, role: "user", content: "ビルド実行" },
  { id: 11, role: "assistant", content: "成功しました" },
  { id: 12, role: "user", content: "テスト実行して" }  // 新しいメッセージ
]

// API送信時（フィルタ後）
const effective = getEffectiveHistory(condensed)
// => [
//   { id: 0, ... },
//   { role: "assistant", content: [...], isSummary: true },  // サマリー
//   { id: 10, ... },
//   { id: 11, ... },
//   { id: 12, ... }
// ]

// トークン削減
// Before: 12メッセージ ≈ 3000トークン
// After:  5メッセージ ≈ 1000トークン
// 削減率: 67% ✅
```

#### 利点

✅ **大幅なトークン削減**: 70-90%の削減が可能
✅ **情報保持**: 重要な情報はサマリーに含まれる
✅ **文脈の連続性**: 会話の流れが維持される
✅ **失敗時のフォールバック**: Truncationで確実に削減
✅ **復元可能**: サマリーを削除すれば元に戻る

---

### 進化3: フラットな履歴 → 階層的State管理

#### Before: フラットな配列

```typescript
// 古い方法
interface OldState {
  messages: Message[]  // すべて同じレベル
  currentTool: Tool | null
  userInput: string
}

// 問題:
// - すべてが同じ優先度
// - 古い情報と新しい情報の区別なし
// - 圧縮戦略が限定的
```

#### After: 階層的構造（Context Engineering）

```typescript
// 新しい方法（Anthropic方式）
interface LayeredState {
  // L0: System / Policy（最高優先度、不変）
  system: {
    role: string                    // システムロール
    policies: Policy[]              // 安全ポリシー
    constraints: Constraint[]       // 制約
    auditRequirements: AuditConfig  // 監査要件
  }

  // L1: Task contract（タスク定義）
  task: {
    goal: string                    // 最終目標
    successCriteria: Criterion[]    // 成功条件
    outputFormat: Schema            // 出力形式
    deadline?: Date                 // 期限
  }

  // L2: Runtime State（実行時状態）
  runtime: {
    phase: "planning" | "executing" | "reflecting" | "verifying"
    permissions: Permission[]       // 現在の権限
    environment: "dev" | "staging" | "prod"
    failureHistory: Failure[]       // 失敗履歴
  }

  // L3: Memory（記憶、複数種類）
  memory: {
    // 短期ワーキングメモリ
    shortTerm: {
      workingVariables: Record<string, any>  // 作業変数
      recentTurns: Message[]                 // 直近3-5ターン
    }

    // エピソード記憶（出来事）
    episodic: {
      summaries: Summary[]      // 要約
      decisions: Decision[]     // 意思決定
      milestones: Milestone[]   // マイルストーン
    }

    // 意味記憶（事実）
    semantic: {
      facts: Fact[]             // 確定事実
      definitions: Definition[] // 定義
      preferences: Preference[] // ユーザー設定
      projectKnowledge: Knowledge[]  // プロジェクト知識
    }

    // 手続き記憶（やり方）
    procedural: {
      skills: Skill[]           // スキル
      templates: Template[]     // テンプレート
      checklists: Checklist[]   // チェックリスト
      procedures: Procedure[]   // 手順
    }
  }

  // L4: Evidence（観測・検索結果）
  evidence: {
    observations: Observation[]  // ツール実行結果
    ragResults: RAGResult[]      // RAG検索結果
    citations: Citation[]        // 引用
    measurements: Measurement[]  // 測定値
  }

  // L5: Work Buffer（作業領域、最も変動的）
  workBuffer: {
    plan: Step[]                // 計画
    diff: Change[]              // 差分
    hypotheses: Hypothesis[]    // 仮説
    draft: string              // 下書き
  }
}
```

#### Context Builder: 階層を組み立てる

```typescript
class ContextBuilder {
  async build(state: LayeredState): Promise<Context> {
    // 各層を構造化されたXMLで構築
    return `
<context>
  <!-- L0: System（常に最優先） -->
  <system>
    <role>${state.system.role}</role>
    <policies>
      ${state.system.policies.map(p => `<policy>${p}</policy>`).join('\n')}
    </policies>
  </system>

  <!-- L1: Task -->
  <task>
    <goal>${state.task.goal}</goal>
    <success_criteria>
      ${state.task.successCriteria.map(c => `<criterion>${c}</criterion>`).join('\n')}
    </success_criteria>
  </task>

  <!-- L2: Runtime State -->
  <runtime_state>
    <phase>${state.runtime.phase}</phase>
    <environment>${state.runtime.environment}</environment>
    <permissions>
      ${state.runtime.permissions.map(p => `<permission>${p}</permission>`).join('\n')}
    </permissions>
  </runtime_state>

  <!-- L3: Memory（圧縮・フィルタ済み） -->
  <memory>
    <!-- 短期ワーキングメモリ（最新） -->
    <short_term>
      <recent_turns>
        ${state.memory.shortTerm.recentTurns.map(t =>
          `<turn role="${t.role}">${t.content}</turn>`
        ).join('\n')}
      </recent_turns>
    </short_term>

    <!-- エピソード記憶（要約済み） -->
    <episodic>
      ${state.memory.episodic.summaries
        .filter(s => this.isRelevant(s, state.task))
        .map(s => `<summary>${s.text}</summary>`)
        .join('\n')}
    </episodic>

    <!-- 意味記憶（関連する事実のみ） -->
    <semantic>
      ${state.memory.semantic.facts
        .filter(f => this.isRelevant(f, state.task))
        .map(f => `<fact source="${f.source}">${f.content}</fact>`)
        .join('\n')}
    </semantic>
  </memory>

  <!-- L4: Evidence（新しい観測を優先） -->
  <evidence>
    <recent_observations>
      ${state.evidence.observations
        .filter(o => Date.now() - o.timestamp < 60000)  // 1分以内
        .map(o => `
          <observation tool="${o.tool}" timestamp="${o.timestamp}">
            ${o.result}
          </observation>
        `).join('\n')}
    </recent_observations>

    <rag_results>
      ${state.evidence.ragResults.map(r => `
        <result relevance="${r.relevance}">
          <source>${r.source}</source>
          <content>${r.content}</content>
        </result>
      `).join('\n')}
    </rag_results>
  </evidence>

  <!-- L5: Work Buffer（現在の作業） -->
  <work_buffer>
    <plan>
      ${state.workBuffer.plan.map((step, i) => `
        <step id="${i}" status="${step.status}">
          ${step.description}
        </step>
      `).join('\n')}
    </plan>
  </work_buffer>
</context>
`
  }

  private isRelevant(item: any, task: Task): boolean {
    // ベクトル類似度やキーワードマッチで関連性判定
    const similarity = this.cosineSimilarity(
      this.embed(item.content),
      this.embed(task.goal)
    )

    return similarity > 0.3  // 閾値
  }
}
```

#### トークン予算配分

```typescript
class TokenBudgetAllocator {
  allocate(state: LayeredState, maxTokens: number): BudgetAllocation {
    const budget = maxTokens * 0.9  // 10%バッファ

    // 階層別の予算配分
    return {
      system: {
        budget: budget * 0.05,      // 5% - 重要だが短い
        priority: Priority.CRITICAL
      },

      task: {
        budget: budget * 0.05,      // 5%
        priority: Priority.CRITICAL
      },

      runtime: {
        budget: budget * 0.03,      // 3%
        priority: Priority.HIGH
      },

      memory: {
        shortTerm: {
          budget: budget * 0.20,    // 20% - 直近の文脈
          priority: Priority.HIGH
        },
        episodic: {
          budget: budget * 0.15,    // 15% - 要約済み
          priority: Priority.MEDIUM
        },
        semantic: {
          budget: budget * 0.10,    // 10% - 関連する事実
          priority: Priority.MEDIUM
        },
        procedural: {
          budget: budget * 0.07,    // 7% - 必要なスキル
          priority: Priority.MEDIUM
        }
      },

      evidence: {
        observations: {
          budget: budget * 0.25,    // 25% - 最新の観測
          priority: Priority.HIGH
        },
        ragResults: {
          budget: budget * 0.10,    // 10%
          priority: Priority.MEDIUM
        }
      },

      workBuffer: {
        budget: budget * 0.10,      // 10%
        priority: Priority.MEDIUM
      }
    }
  }

  async fitToBudget(
    content: any,
    budgetTokens: number
  ): Promise<any> {
    const currentTokens = await this.countTokens(content)

    if (currentTokens <= budgetTokens) {
      return content  // 予算内
    }

    // 予算超過 → 圧縮
    if (Array.isArray(content)) {
      // 配列の場合: 優先度でソート→上位を保持
      const withPriority = content.map(item => ({
        item,
        priority: this.calculatePriority(item)
      }))

      withPriority.sort((a, b) => b.priority - a.priority)

      // 予算内に収まるまで保持
      let accumulated = 0
      const result = []

      for (const { item } of withPriority) {
        const tokens = await this.countTokens(item)

        if (accumulated + tokens <= budgetTokens) {
          result.push(item)
          accumulated += tokens
        } else {
          break
        }
      }

      return result
    } else if (typeof content === 'string') {
      // 文字列の場合: 要約
      return await this.summarize(content, budgetTokens)
    }

    return content
  }
}
```

#### 利点

✅ **明確な優先度**: どの情報が重要かが明確
✅ **柔軟な圧縮**: 層ごとに最適な戦略を適用
✅ **効率的なトークン使用**: 予算配分により最適化
✅ **拡張性**: 新しい層を追加しやすい
✅ **関連性フィルタ**: タスクに関連する情報のみ保持

---

### 進化4: 静的ツール → 動的ツール投影

#### Before: すべてのツールを常に提供

```typescript
// 古い方法
class OldToolManager {
  getTools(): Tool[] {
    // 常に全ツールを返す
    return [
      readFileTool,
      writeFileTool,
      deleteFileTool,      // ❌ 常に見える
      executeTool,         // ❌ 常に見える
      searchTool,
      analyzeTool,
      deployTool,          // ❌ 常に見える（危険）
      rollbackTool         // ❌ 常に見える（危険）
    ]
  }
}

// 問題:
// 1. セキュリティリスク
// 2. トークンの無駄
// 3. LLMの混乱（選択肢が多すぎる）
// 4. 誤操作のリスク
```

#### After: 状態に応じた動的投影

```typescript
// 新しい方法
class DynamicToolProjector {
  project(
    state: LayeredState,
    allTools: Tool[]
  ): Tool[] {
    let tools = [...allTools]

    // ステップ1: フェーズフィルタリング
    tools = this.filterByPhase(tools, state.runtime.phase)

    // ステップ2: 権限フィルタリング
    tools = this.filterByPermission(tools, state.runtime.permissions)

    // ステップ3: 環境に応じたスキーマ制限
    tools = this.restrictSchema(tools, state.runtime.environment)

    // ステップ4: Tool budget（トークン効率化）
    tools = this.applyToolBudget(tools, state)

    return tools
  }

  // フェーズ別フィルタリング
  private filterByPhase(
    tools: Tool[],
    phase: RuntimeState['phase']
  ): Tool[] {
    const phaseToolMap: Record<string, string[]> = {
      planning: [
        'search',      // 情報収集
        'analyze',     // 分析
        'estimate',    // 見積もり
        'readFile'     // ファイル読み込み
      ],

      executing: [
        'writeFile',   // ファイル書き込み
        'execute',     // 実行
        'modify',      // 修正
        'test'         // テスト
      ],

      reflecting: [
        'evaluate',    // 評価
        'test',        // テスト
        'compare',     // 比較
        'measure'      // 測定
      ],

      verifying: [
        'verify',      // 検証
        'audit',       // 監査
        'approve'      // 承認
      ]
    }

    const allowedCategories = phaseToolMap[phase] || []

    return tools.filter(tool =>
      allowedCategories.includes(tool.category)
    )
  }

  // 権限別フィルタリング
  private filterByPermission(
    tools: Tool[],
    permissions: Permission[]
  ): Tool[] {
    return tools.filter(tool => {
      // ツールに必要な権限をチェック
      const requiredPermissions = tool.requiredPermissions || []

      return requiredPermissions.every(required =>
        permissions.some(p => p.includes(required))
      )
    })
  }

  // 環境に応じたスキーマ制限
  private restrictSchema(
    tools: Tool[],
    environment: RuntimeState['environment']
  ): Tool[] {
    if (environment === 'production') {
      return tools.map(tool => this.addProductionSafety(tool))
    }

    return tools
  }

  private addProductionSafety(tool: Tool): Tool {
    // 例: deleteFileツール
    if (tool.name === 'deleteFile') {
      return {
        ...tool,
        parameters: {
          ...tool.parameters,
          properties: {
            ...tool.parameters.properties,
            path: {
              type: 'string',
              // 本番環境では許可されたパスのみ
              enum: [
                '/tmp/allowed-file-1.txt',
                '/tmp/allowed-file-2.txt'
              ],
              description: 'File path (production: whitelist only)'
            }
          }
        }
      }
    }

    // 例: executeツール
    if (tool.name === 'execute') {
      return {
        ...tool,
        parameters: {
          ...tool.parameters,
          properties: {
            ...tool.parameters.properties,
            command: {
              type: 'string',
              // 本番環境では安全なコマンドのみ
              enum: [
                'npm test',
                'npm run build',
                'npm run lint'
              ],
              description: 'Command (production: safe commands only)'
            }
          }
        }
      }
    }

    return tool
  }

  // Tool budget適用
  private applyToolBudget(
    tools: Tool[],
    state: LayeredState
  ): Tool[] {
    // タスクの緊急度・重要度に応じてツール数を制限
    const urgency = this.calculateUrgency(state.task)

    if (urgency === 'high') {
      // 緊急時は重要なツールのみ
      return tools
        .sort((a, b) => b.priority - a.priority)
        .slice(0, 5)  // 上位5件
    }

    // 通常時はトークン予算内に収める
    const budgetTokens = 2000  // ツール定義用のトークン予算
    let accumulated = 0
    const result: Tool[] = []

    for (const tool of tools) {
      const toolTokens = this.estimateToolTokens(tool)

      if (accumulated + toolTokens <= budgetTokens) {
        result.push(tool)
        accumulated += toolTokens
      } else {
        break
      }
    }

    return result
  }
}
```

#### 具体例: フェーズごとのツール変化

```typescript
const allTools = [
  { name: 'search', category: 'search', priority: 80 },
  { name: 'readFile', category: 'readFile', priority: 90 },
  { name: 'writeFile', category: 'writeFile', priority: 85 },
  { name: 'deleteFile', category: 'deleteFile', priority: 50, requiredPermissions: ['admin'] },
  { name: 'execute', category: 'execute', priority: 70 },
  { name: 'test', category: 'test', priority: 75 },
  { name: 'deploy', category: 'deploy', priority: 60, requiredPermissions: ['deploy'] }
]

// Planning フェーズ
const state1 = {
  runtime: {
    phase: 'planning',
    permissions: ['read', 'search'],
    environment: 'dev'
  }
}

const tools1 = projector.project(state1, allTools)
// => [search, readFile]
// ✅ 情報収集系のみ
// ✅ write/delete/execute は見えない

// Executing フェーズ
const state2 = {
  runtime: {
    phase: 'executing',
    permissions: ['read', 'write', 'execute'],
    environment: 'dev'
  }
}

const tools2 = projector.project(state2, allTools)
// => [readFile, writeFile, execute, test]
// ✅ 実行系が見える
// ✅ deploy は権限不足で見えない

// Executing フェーズ（本番環境）
const state3 = {
  runtime: {
    phase: 'executing',
    permissions: ['read', 'write', 'execute', 'admin'],
    environment: 'production'
  }
}

const tools3 = projector.project(state3, allTools)
// => [
//   readFile,
//   writeFile,
//   deleteFile (with enum restrictions),  // ✅ パス制限
//   execute (with enum restrictions),     // ✅ コマンド制限
//   test
// ]
```

#### 利点

✅ **セキュリティ強化**: フェーズ・権限・環境で制限
✅ **トークン効率**: 必要なツールのみ提供
✅ **誤操作防止**: 危険なツールを隠す
✅ **LLMの精度向上**: 選択肢が絞られて判断しやすい
✅ **監査証跡**: どのツールがいつ使えたか記録

---

### 進化5: tool_use単体 → tool_use/tool_resultペア保持

#### Before: ペアが崩れる

```typescript
// 古い方法: 要約時にtool_useが失われる
async summarize(messages: Message[]): Promise<Message[]> {
  const toSummarize = messages.slice(0, -3)
  const keep = messages.slice(-3)  // 最新3件保持

  const summary = await this.llm.generate(...)

  return [
    { role: 'assistant', content: summary },  // サマリー
    ...keep  // 最新3件
  ]
}

// 問題: 最初のkeepメッセージがtool_resultを含む場合
// Before:
// [
//   { role: 'assistant', content: [..., tool_use] },  // 要約対象
//   { role: 'user', content: [..., tool_result] }     // 保持
// ]

// After:
// [
//   { role: 'assistant', content: 'Summary...' },     // tool_use がない
//   { role: 'user', content: [..., tool_result] }     // ❌ ペアが崩れる
// ]

// エラー: "tool_result requires matching tool_use"
```

#### After: ペア保持（Native Tools対応）

```typescript
// 新しい方法（Roo Code方式）
async summarize(
  messages: Message[],
  useNativeTools: boolean
): Promise<Message[]> {
  const keep = messages.slice(-3)
  const toSummarize = messages.slice(0, -3)

  // Native Tools使用時: tool_useブロック抽出
  let toolUseBlocks: ToolUseBlock[] = []
  let reasoningBlocks: ReasoningBlock[] = []

  if (useNativeTools) {
    const result = this.extractToolBlocks(keep, messages)
    toolUseBlocks = result.toolUseBlocks
    reasoningBlocks = result.reasoningBlocks
  }

  // サマリー生成
  const summaryText = await this.llm.generate(...)

  // サマリーメッセージ構築
  const summaryContent: ContentBlock[] = [
    // 合成reasoning（DeepSeek対応）
    {
      type: 'reasoning',
      text: 'Condensing conversation context. Summary captures key information.'
    },

    // 元のreasoningブロック保持
    ...reasoningBlocks,

    // サマリーテキスト
    {
      type: 'text',
      text: summaryText
    },

    // tool_useブロック保持（重要！）
    ...toolUseBlocks
  ]

  return [
    {
      role: 'assistant',
      content: summaryContent,
      isSummary: true
    },
    ...keep
  ]
}

// tool_useブロック抽出
private extractToolBlocks(
  keepMessages: Message[],
  allMessages: Message[]
): ExtractResult {
  // 最初のkeepメッセージをチェック
  const firstKeep = keepMessages[0]

  // tool_resultを含むか？
  const hasToolResult = this.hasToolResultBlocks(firstKeep)

  if (!hasToolResult) {
    return { toolUseBlocks: [], reasoningBlocks: [] }
  }

  // 直前のassistantメッセージを取得
  const keepStartIndex = allMessages.indexOf(firstKeep)
  const precedingIndex = keepStartIndex - 1

  if (precedingIndex < 0) {
    return { toolUseBlocks: [], reasoningBlocks: [] }
  }

  const precedingMessage = allMessages[precedingIndex]

  if (precedingMessage.role !== 'assistant') {
    return { toolUseBlocks: [], reasoningBlocks: [] }
  }

  // tool_useブロックを抽出
  const toolUseBlocks = this.getToolUseBlocks(precedingMessage)

  // reasoningブロックを抽出（DeepSeek対応）
  const reasoningBlocks = this.getReasoningBlocks(precedingMessage)

  return { toolUseBlocks, reasoningBlocks }
}

private getToolUseBlocks(message: Message): ToolUseBlock[] {
  if (!Array.isArray(message.content)) {
    return []
  }

  return message.content.filter(
    block => block.type === 'tool_use'
  ) as ToolUseBlock[]
}

private getReasoningBlocks(message: Message): ReasoningBlock[] {
  if (!Array.isArray(message.content)) {
    return []
  }

  return message.content.filter(
    block => block.type === 'reasoning'
  ) as ReasoningBlock[]
}
```

#### 具体例: ペア保持の動作

```typescript
// 初期状態
const messages = [
  { id: 1, role: 'user', content: 'ファイルを読んで' },
  { id: 2, role: 'assistant', content: 'readFileツールを使います' },
  // ... 多数のメッセージ ...

  // 直前のassistantメッセージ（tool_use含む）
  {
    id: 98,
    role: 'assistant',
    content: [
      { type: 'reasoning', text: 'Using readFile to check content' },
      {
        type: 'tool_use',
        id: 'tool_abc',
        name: 'readFile',
        input: { path: 'test.js' }
      }
    ]
  },

  // 最初のkeepメッセージ（tool_result含む）
  {
    id: 99,
    role: 'user',
    content: [
      {
        type: 'tool_result',
        tool_use_id: 'tool_abc',
        content: 'console.log("hello")'
      }
    ]
  },  // ← keep開始位置

  { id: 100, role: 'assistant', content: 'ファイル内容を確認しました' },
  { id: 101, role: 'user', content: '次は何をする？' }
]

// 要約実行（useNativeTools: true）
const result = await summarize(messages, true)

// 結果
// [
//   {
//     role: 'assistant',
//     content: [
//       // 合成reasoning
//       { type: 'reasoning', text: 'Condensing conversation context...' },
//
//       // 元のreasoningブロック（保持）
//       { type: 'reasoning', text: 'Using readFile to check content' },
//
//       // サマリーテキスト
//       { type: 'text', text: 'Previous conversation: ユーザーがファイル読み込みを依頼...' },
//
//       // tool_useブロック（保持！）
//       {
//         type: 'tool_use',
//         id: 'tool_abc',
//         name: 'readFile',
//         input: { path: 'test.js' }
//       }
//     ],
//     isSummary: true
//   },
//
//   // keep（最新3件）
//   {
//     id: 99,
//     role: 'user',
//     content: [
//       {
//         type: 'tool_result',
//         tool_use_id: 'tool_abc',  // ✅ 対応するtool_useがサマリーに含まれている
//         content: 'console.log("hello")'
//       }
//     ]
//   },
//   { id: 100, role: 'assistant', content: '...' },
//   { id: 101, role: 'user', content: '...' }
// ]

// ✅ tool_use/tool_resultペアが維持される
// ✅ Native Toolsプロトコルに準拠
// ✅ DeepSeek対応（reasoningブロック含む）
```

#### 利点

✅ **Native Toolsプロトコル準拠**: tool_use/resultペア維持
✅ **DeepSeek/Z.ai対応**: reasoning_contentを保持
✅ **API互換性**: エラーなく動作
✅ **文脈保持**: ツール呼び出しの意図が残る

---

### 進化6: 直接削除 → MessageManager統合

#### Before: 直接操作（危険）

```typescript
// 古い方法: 直接削除
class OldMessageHandler {
  deleteMessage(messageId: string) {
    // clineMessagesから削除
    this.clineMessages = this.clineMessages.filter(
      m => m.id !== messageId
    )  // ❌ 直接削除

    // apiConversationHistoryから削除
    this.apiConversationHistory = this.apiConversationHistory.filter(
      m => m.id !== messageId
    )  // ❌ 直接削除

    // 問題:
    // 1. サマリーが孤立する
    // 2. マーカーが孤立する
    // 3. タグが孤立する
    // 4. チェックポイントと不整合
  }
}
```

#### After: MessageManager統合

```typescript
// 新しい方法（Roo Code方式）
class MessageManager {
  constructor(
    private state: LayeredState,
    private checkpointService: CheckpointService
  ) {}

  async rewindToTimestamp(
    ts: number,
    options: RewindOptions = {}
  ): Promise<void> {
    const { includeTargetMessage = false } = options

    // ステップ1: 削除されるコンテキストイベントIDを収集
    const removedIds = this.collectRemovedContextEventIds(ts)

    // ステップ2: clineMessagesを削除
    await this.truncateClineMessages(ts, includeTargetMessage)

    // ステップ3: apiConversationHistoryを削除 + クリーンアップ
    await this.truncateApiHistory(ts, removedIds)

    // ステップ4: チェックポイント同期
    await this.syncCheckpoint(ts)
  }

  // コンテキストイベントID収集
  private collectRemovedContextEventIds(
    cutoffTs: number
  ): ContextEventIds {
    const condenseIds = new Set<string>()
    const truncationIds = new Set<string>()

    // cutoff以降のclineMessagesを走査
    for (const msg of this.state.clineMessages) {
      if (msg.ts >= cutoffTs) {
        // condense_contextイベント
        if (msg.say === 'condense_context' && msg.contextCondense?.condenseId) {
          condenseIds.add(msg.contextCondense.condenseId)
        }

        // sliding_window_truncationイベント
        if (msg.say === 'sliding_window_truncation' && msg.contextTruncation?.truncationId) {
          truncationIds.add(msg.contextTruncation.truncationId)
        }
      }
    }

    return { condenseIds, truncationIds }
  }

  // apiConversationHistory削除 + クリーンアップ
  private async truncateApiHistory(
    cutoffTs: number,
    removedIds: ContextEventIds
  ): Promise<void> {
    let history = [...this.state.apiConversationHistory]

    // ステップ1: レースコンディション対策（タイムスタンプ調整）
    const actualCutoff = this.adjustCutoffTimestamp(history, cutoffTs)

    // ステップ2: タイムスタンプでフィルタ
    history = history.filter(m => !m.ts || m.ts < actualCutoff)

    // ステップ3: 孤立したサマリー削除
    if (removedIds.condenseIds.size > 0) {
      history = history.filter(msg => {
        if (msg.isSummary && msg.condenseId && removedIds.condenseIds.has(msg.condenseId)) {
          console.log(`[MessageManager] Removing orphaned summary: ${msg.condenseId}`)
          return false  // 削除
        }
        return true
      })
    }

    // ステップ4: 孤立したマーカー削除
    if (removedIds.truncationIds.size > 0) {
      history = history.filter(msg => {
        if (msg.isTruncationMarker && msg.truncationId && removedIds.truncationIds.has(msg.truncationId)) {
          console.log(`[MessageManager] Removing orphaned marker: ${msg.truncationId}`)
          return false  // 削除
        }
        return true
      })
    }

    // ステップ5: タグクリーンアップ
    history = this.cleanupOrphanedTags(history)

    // 保存
    this.state.apiConversationHistory = history
    await this.saveApiHistory(history)
  }

  // 孤立したタグのクリーンアップ
  private cleanupOrphanedTags(messages: Message[]): Message[] {
    // 存在するサマリー/マーカーIDを収集
    const existingCondenseIds = new Set(
      messages
        .filter(m => m.isSummary && m.condenseId)
        .map(m => m.condenseId)
    )

    const existingTruncationIds = new Set(
      messages
        .filter(m => m.isTruncationMarker && m.truncationId)
        .map(m => m.truncationId)
    )

    // 孤立したタグを削除
    return messages.map(msg => {
      let needsUpdate = false

      // condenseParentチェック
      if (msg.condenseParent && !existingCondenseIds.has(msg.condenseParent)) {
        needsUpdate = true
      }

      // truncationParentチェック
      if (msg.truncationParent && !existingTruncationIds.has(msg.truncationParent)) {
        needsUpdate = true
      }

      if (!needsUpdate) {
        return msg
      }

      // タグを削除して復元
      const { condenseParent, truncationParent, ...rest } = msg
      const result: Message = rest

      // 存在する参照のみ保持
      if (condenseParent && existingCondenseIds.has(condenseParent)) {
        result.condenseParent = condenseParent
      }

      if (truncationParent && existingTruncationIds.has(truncationParent)) {
        result.truncationParent = truncationParent
      }

      return result
    })
  }

  // レースコンディション対策
  private adjustCutoffTimestamp(
    history: Message[],
    cutoffTs: number
  ): number {
    // cutoffに完全一致するメッセージがあるか
    const hasExactMatch = history.some(m => m.ts === cutoffTs)

    // cutoff以前にメッセージがあるか
    const hasMessageBefore = history.some(m => m.ts !== undefined && m.ts < cutoffTs)

    // レースコンディション検出
    if (!hasExactMatch && hasMessageBefore) {
      // 最初のuserメッセージ（cutoff以降）を境界とする
      const firstUserAfter = history.find(
        m => m.ts !== undefined && m.ts >= cutoffTs && m.role === 'user'
      )

      if (firstUserAfter && firstUserAfter.ts) {
        console.log(`[MessageManager] Adjusting cutoff: ${cutoffTs} → ${firstUserAfter.ts}`)
        return firstUserAfter.ts
      }
    }

    return cutoffTs
  }

  // チェックポイント同期
  private async syncCheckpoint(ts: number): Promise<void> {
    if (!this.checkpointService) {
      return
    }

    // チェックポイントも同じタイムスタンプまで巻き戻し
    await this.checkpointService.rewindTo(ts)
  }
}
```

#### レースコンディション問題

```typescript
// ストリーミング実行中の問題

// Timeline:
// T1: Assistant message starts streaming
// T2: Tool execution completes → user_feedback clineMessage created (ts=T2)
// T3: Assistant message stream completes → API message saved (ts=T3)

// 結果:
// clineMessage(user_feedback, ts=T2) < apiMessage(assistant, ts=T3)

// 問題:
// T2で巻き戻すと、clineMessageのT2は見つかるが、
// apiConversationHistoryにT2のメッセージがない（T3だから）

// 解決策: 最初のuserメッセージを境界とする
```

#### 利点

✅ **データ一貫性**: サマリー/マーカー/タグがすべて正しく管理される
✅ **レースコンディション対策**: タイムスタンプのずれに対応
✅ **チェックポイント統合**: 完全に同期
✅ **デバッグ容易**: ログで何が削除されたか追跡可能
✅ **安全性**: 直接操作を禁止

---

### 進化7: 単一しきい値 → プロファイル別最適化

#### Before: 全モデル共通

```typescript
// 古い方法
const config = {
  contextWindow: 200000,
  threshold: 0.75  // すべてのモデルで75%
}

// 問題:
// - Claude Opus: 高性能だがしきい値が低すぎる（もっと使える）
// - Claude Haiku: 低コストだがしきい値が高すぎる（早めに凝縮すべき）
// - GPT-4 Turbo: 128kだが200kの75%で計算（不正確）
```

#### After: モデルごとの最適化

```typescript
// 新しい方法（Roo Code方式）
interface ProfileConfig {
  // グローバル設定
  autoCondenseContext: boolean
  autoCondenseContextPercent: number  // デフォルトしきい値

  // プロファイル別オーバーライド
  profileThresholds: Record<string, number>

  // 現在のプロファイル
  currentProfileId: string
}

class ProfileBasedContextManager {
  constructor(private config: ProfileConfig) {}

  // 有効なしきい値を決定
  getEffectiveThreshold(): number {
    const profileThreshold = this.config.profileThresholds[this.config.currentProfileId]

    // プロファイル設定がない場合
    if (profileThreshold === undefined) {
      return this.config.autoCondenseContextPercent
    }

    // -1 は親設定を継承
    if (profileThreshold === -1) {
      return this.config.autoCondenseContextPercent
    }

    // 有効範囲チェック（5-100%）
    if (profileThreshold < MIN_CONDENSE_THRESHOLD || profileThreshold > MAX_CONDENSE_THRESHOLD) {
      console.warn(
        `Invalid threshold ${profileThreshold} for profile "${this.config.currentProfileId}". ` +
        `Using global default of ${this.config.autoCondenseContextPercent}%`
      )
      return this.config.autoCondenseContextPercent
    }

    // カスタムしきい値
    return profileThreshold
  }

  // コンテキスト管理判定
  async manageContext(
    messages: Message[],
    totalTokens: number
  ): Promise<ContextResult> {
    const contextWindow = this.getContextWindow()
    const threshold = this.getEffectiveThreshold()

    const contextPercent = (100 * totalTokens) / contextWindow

    // しきい値チェック
    if (contextPercent < threshold) {
      return { messages }  // 何もしない
    }

    // 凝縮実行
    return await this.condense(messages)
  }

  // モデル別のコンテキストウィンドウ
  private getContextWindow(): number {
    const windows: Record<string, number> = {
      'claude-opus-4-5-20251101': 200000,
      'claude-3-7-sonnet-20250219': 200000,
      'claude-3-5-haiku-20241022': 200000,
      'gpt-4-turbo': 128000,
      'gpt-3.5-turbo': 16000
    }

    return windows[this.config.currentProfileId] || 200000
  }
}
```

#### 推奨設定

```typescript
// 推奨プロファイル設定
const recommendedThresholds: Record<string, number> = {
  // Claude Opus 4: 高性能、高コスト → 最大限活用
  'claude-opus-4-5-20251101': 80,  // 80%まで使う

  // Claude Sonnet: バランス型 → デフォルト
  'claude-3-7-sonnet-20250219': 75,  // 75%（推奨デフォルト）

  // Claude Haiku: 低コスト → 早めに凝縮
  'claude-3-5-haiku-20241022': 60,  // 60%で凝縮開始

  // GPT-4 Turbo: 128k → やや早め
  'gpt-4-turbo': 70,

  // GPT-3.5: 16k → かなり早め
  'gpt-3.5-turbo': 50,

  // カスタムプロファイル: 親設定を継承
  'custom-profile': -1
}

// VSCode設定での運用
// settings.json:
{
  "rooCode.autoCondenseContext": true,
  "rooCode.autoCondenseContextPercent": 75,  // グローバル設定
  "rooCode.profileThresholds": {
    "claude-opus-4-5-20251101": 80,
    "claude-3-5-haiku-20241022": 60,
    "gpt-4-turbo": 70
  }
}
```

#### モデル別の戦略

```typescript
// モデル特性に応じた最適化
class ModelAwareOptimizer {
  optimizeForModel(modelId: string): OptimizationStrategy {
    // Claude Opus: 高性能、高コスト
    if (modelId.includes('opus')) {
      return {
        threshold: 80,  // 高め
        condenseModel: modelId,  // 同じモデルで要約（品質重視）
        compressionRatio: 0.7,   // 70%削減目標
        keepMessages: 5          // 多めに保持
      }
    }

    // Claude Haiku: 低コスト
    if (modelId.includes('haiku')) {
      return {
        threshold: 60,  // 低め（早めに凝縮）
        condenseModel: modelId,  // 同じモデル（コスト重視）
        compressionRatio: 0.8,   // 80%削減目標（積極的）
        keepMessages: 3          // 最小限
      }
    }

    // Claude Sonnet: バランス型
    if (modelId.includes('sonnet')) {
      return {
        threshold: 75,  // 標準
        condenseModel: 'claude-3-5-haiku-20241022',  // 安価なモデルで要約
        compressionRatio: 0.75,  // 75%削減
        keepMessages: 3
      }
    }

    // GPT系: 小さめのコンテキストウィンドウ
    if (modelId.includes('gpt')) {
      return {
        threshold: 70,  // やや低め
        condenseModel: 'gpt-4-turbo',  // GPTで要約
        compressionRatio: 0.75,
        keepMessages: 3
      }
    }

    // デフォルト
    return {
      threshold: 75,
      condenseModel: modelId,
      compressionRatio: 0.75,
      keepMessages: 3
    }
  }
}
```

#### 利点

✅ **コスト最適化**: モデルの特性に合わせた設定
✅ **品質維持**: 高性能モデルを最大限活用
✅ **柔軟性**: プロジェクトごとに調整可能
✅ **継承**: -1で親設定を継承（階層的設定）
✅ **バリデーション**: 無効な値のフォールバック

---

## なぜこの進化が必要だったか

### 1. LLMアプリケーションの成熟

**2023-2024**: 実験的なデモが中心
- 短いセッション
- 限定的なタスク
- プロトタイプ品質

**2025**: プロダクションレベルの要求
- 長時間セッション（数時間〜数日）
- 複雑なタスク（マルチステップ、複数ファイル）
- エンタープライズ品質（監査、復元、チェックポイント）

### 2. コンテキストウィンドウの拡大

**2023**: 4k-16kトークン
→ 単純削除でも何とかなった

**2024-2025**: 128k-200kトークン
→ 賢く使わないと無駄、かつ制限に達するのが早い

### 3. Native Toolsプロトコルの登場

**Before**: 基本的なfunction calling

**After**: 複雑なtool_use/tool_resultペア、reasoning_content
→ プロトコル準拠が必須

### 4. エージェント的な使用の増加

**Before**: 1往復の質問応答

**After**: 長期的なタスク実行（planning → executing → reflecting）
→ フェーズ管理、権限管理が必要

### 5. コスト意識の高まり

**Before**: 実験だからコストは気にしない

**After**: プロダクション運用でコストが重要
→ モデル別最適化、効率的な圧縮が必要

### 6. セキュリティとコンプライアンス

**Before**: 個人利用、セキュリティは後回し

**After**: 企業利用、規制対応が必須
→ 監査証跡、権限管理、安全なツール投影

---

## 実際の影響

### トークン削減効果

```typescript
// Before（単純削除）
初期: 12メッセージ ≈ 3000トークン
削除後: 6メッセージ ≈ 1500トークン
削減率: 50%
情報損失: 大（重要な文脈が失われる）

// After（Condensation）
初期: 12メッセージ ≈ 3000トークン
凝縮後: 5メッセージ（サマリー1 + 保持4） ≈ 900トークン
削減率: 70%
情報損失: 小（サマリーに重要情報が含まれる）
```

### コスト削減

```typescript
// Before: 常にフルモデル
Claude Opus使用: すべての要約にOpus使用
月額コスト: $500

// After: 専用モデル
メイン: Claude Opus
要約: Claude Haiku（1/10のコスト）
月額コスト: $350

削減: 30% ✅
```

### セキュリティ向上

```typescript
// Before: すべてのツールが常に見える
危険操作の誤実行: 月10件

// After: 動的ツール投影
危険操作の誤実行: 月0件

改善: 100% ✅
```

### 開発効率

```typescript
// Before: データ損失でデバッグ困難
バグ修正時間: 平均4時間

// After: 非破壊的管理でデバッグ容易
バグ修正時間: 平均1時間

改善: 75% ✅
```

---

## まとめ: 1年間の進化

| 観点 | 1年前（2023-2024） | 現在（2025） | 改善度 |
|------|------------------|-------------|--------|
| **データ管理** | 削除 | タグ付け | ★★★★★ |
| **圧縮** | 機械的 | AI要約 | ★★★★★ |
| **State** | フラット | 階層的 | ★★★★☆ |
| **ツール** | 静的 | 動的投影 | ★★★★★ |
| **プロトコル** | 基本 | Native Tools対応 | ★★★★☆ |
| **管理** | 直接操作 | MessageManager | ★★★★★ |
| **最適化** | 単一 | プロファイル別 | ★★★★☆ |
| **総合** | プロトタイプ品質 | プロダクション品質 | ★★★★★ |

---

**次**: [02-modern-architecture.md](./02-modern-architecture.md) - 現代的なアーキテクチャの詳細設計
