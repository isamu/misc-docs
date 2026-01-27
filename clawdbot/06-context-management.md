# Clawdbot コンテキスト管理

## 概要

Clawdbot のコンテキスト管理システムは、AI との会話を効果的に維持するための多層アーキテクチャを採用しています。

```
┌─────────────────────────────────────────────────────────────────┐
│                    コンテキスト構築パイプライン                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │   Session    │    │  Bootstrap   │    │   Memory     │      │
│  │   History    │    │    Files     │    │   Search     │      │
│  │  (JSONL)     │    │ (Markdown)   │    │  (Vector)    │      │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘      │
│         │                   │                   │               │
│         └───────────────────┼───────────────────┘               │
│                             │                                   │
│                             ▼                                   │
│                   ┌──────────────────┐                         │
│                   │  System Prompt   │                         │
│                   │    Builder       │                         │
│                   └────────┬─────────┘                         │
│                            │                                    │
│                            ▼                                    │
│                   ┌──────────────────┐                         │
│                   │ Context Window   │                         │
│                   │   Validator      │                         │
│                   └────────┬─────────┘                         │
│                            │                                    │
│                            ▼                                    │
│                   ┌──────────────────┐                         │
│                   │ Context Pruning  │                         │
│                   │   (if needed)    │                         │
│                   └────────┬─────────┘                         │
│                            │                                    │
│                            ▼                                    │
│                   ┌──────────────────┐                         │
│                   │   LLM Request    │                         │
│                   │ (systemPrompt +  │                         │
│                   │  messages + tools)│                         │
│                   └──────────────────┘                         │
└─────────────────────────────────────────────────────────────────┘
```

## 1. セッション管理

### セッションキーの導出

セッションは一意のキーで識別され、以下の構造を持ちます:

```typescript
// セッションキーの形式
sessionKey = `agent:${agentId}:${mainKey}`

// mainKey の例
// - DM: sender ID またはユーザー識別子
// - Group: グループ ID
// - Thread: 親スレッド ID

// 実際の例
"agent:default:telegram:123456789"      // Telegram DM
"agent:default:discord:guild:channel"   // Discord チャンネル
"agent:work:slack:C12345678"            // Slack チャンネル
```

### セッションスコープ

```typescript
enum SessionScope {
  GLOBAL = "global",       // 全チャット共有
  PER_GROUP = "per-group", // グループごと
  PER_SENDER = "per-sender" // 送信者ごと（デフォルト）
}
```

### セッションストレージ構造

```
~/.clawdbot/
└── agents/
    └── {agentId}/
        ├── sessions.json           # セッションメタデータ
        └── sessions/
            ├── {sessionId}.jsonl   # 会話履歴
            └── ...
```

### sessions.json の構造

```json
{
  "entries": {
    "session-abc123": {
      "id": "session-abc123",
      "agentId": "default",
      "sessionKey": "agent:default:telegram:123456",
      "createdAt": 1700000000000,
      "updatedAt": 1700001000000,
      "model": "claude-sonnet-4-20250514",
      "messageCount": 42,
      "tokenUsage": {
        "input": 15000,
        "output": 8000
      },
      "overrides": {
        "thinking": "medium",
        "maxTokens": 4096
      },
      "state": "active"
    }
  }
}
```

### JSONL トランスクリプト形式

```jsonl
{"type":"header","version":1,"sessionId":"abc123","createdAt":1700000000000}
{"role":"user","content":"Hello","createdAt":1700000001000,"meta":{"sender":"user123"}}
{"role":"assistant","content":"Hi there!","createdAt":1700000002000,"meta":{"model":"claude-sonnet-4"}}
{"role":"user","content":"How are you?","createdAt":1700000003000,"meta":{"sender":"user123"}}
{"role":"assistant","content":"I'm doing well!","createdAt":1700000004000,"meta":{"model":"claude-sonnet-4"}}
```

## 2. コンテキスト構築

### 構築フロー

```typescript
async function buildContext(sessionKey: string, config: AgentConfig): Promise<Context> {
  // 1. セッションエントリとヒストリーを読み込み
  const session = await loadSession(sessionKey);
  const history = await loadHistory(session.id);

  // 2. ターン数制限を適用
  const limitedHistory = limitTurns(history, config.maxTurns);

  // 3. ブートストラップファイルを読み込み
  const bootstrap = await loadBootstrapFiles(config.workspace);

  // 4. システムプロンプトを構築
  const systemPrompt = buildSystemPrompt({
    bootstrap,
    skills: config.skills,
    tools: config.tools,
    memory: config.memory,
    runtime: config.runtime,
  });

  // 5. コンテキストウィンドウを検証
  validateContextWindow(systemPrompt, limitedHistory, config.contextWindow);

  // 6. 必要に応じてプルーニング
  const prunedHistory = await pruneIfNeeded(limitedHistory, config);

  // 7. オプションでメモリ検索
  const memoryContext = config.memory.enabled
    ? await searchMemory(history, config.memory)
    : null;

  return {
    systemPrompt,
    messages: prunedHistory,
    memoryContext,
  };
}
```

### ブートストラップファイル

| ファイル | 目的 | 最大長 |
|---------|------|--------|
| `AGENTS.md` | 操作指示、ルール、制約 | 20,000文字 |
| `SOUL.md` | ペルソナ、トーン、境界 | 20,000文字 |
| `TOOLS.md` | ツール使用の注意事項 | 20,000文字 |
| `IDENTITY.md` | エージェント名、特性 | 5,000文字 |
| `USER.md` | ユーザープロファイル | 10,000文字 |
| `README.md` | プロジェクト説明 | 10,000文字 |

### ファイル切り詰め戦略

長いブートストラップファイルは、重要な部分を保持しながら切り詰められます:

```typescript
function truncateFile(content: string, maxChars: number): string {
  if (content.length <= maxChars) {
    return content;
  }

  // 70% を先頭から、20% を末尾から保持
  const headRatio = 0.7;
  const tailRatio = 0.2;

  const headChars = Math.floor(maxChars * headRatio);
  const tailChars = Math.floor(maxChars * tailRatio);

  const head = content.slice(0, headChars);
  const tail = content.slice(-tailChars);

  return `${head}\n\n[...truncated...]\n\n${tail}`;
}
```

## 3. システムプロンプト構成

システムプロンプトは21のセクションで構成されます:

```typescript
const systemPromptSections = [
  // 基本情報
  "identity",           // エージェント名、特性
  "persona",            // SOUL.md からのペルソナ

  // 指示
  "instructions",       // AGENTS.md からの操作指示
  "guidelines",         // 一般的なガイドライン

  // ツール
  "tooling",            // 利用可能なツール一覧
  "toolPolicies",       // ツール使用ポリシー

  // スキル
  "skills",             // 有効化されたスキル
  "skillTriggers",      // スキルトリガー条件

  // メモリ
  "memory",             // メモリシステム設定
  "memoryContext",      // 検索されたメモリ

  // ワークスペース
  "workspace",          // 作業ディレクトリ情報
  "projectContext",     // README.md などのプロジェクト情報

  // サンドボックス
  "sandbox",            // サンドボックス設定
  "permissions",        // 実行許可設定

  // ランタイム
  "runtime",            // ランタイム情報
  "model",              // モデル固有の調整
  "thinking",           // 思考レベル設定

  // チャネル
  "channel",            // 現在のチャネル情報
  "chatContext",        // チャットタイプ、制限

  // ユーザー
  "user",               // USER.md からのユーザー情報

  // その他
  "timestamp",          // 現在時刻
];
```

### プロンプトモード

| モード | 用途 | 含まれるセクション |
|-------|------|------------------|
| `full` | メインエージェント | 全21セクション |
| `minimal` | サブエージェント | identity, instructions, tooling のみ |
| `none` | 基本実行 | identity のみ |

## 4. コンテキストウィンドウ管理

### ウィンドウサイズの決定

```typescript
function getContextWindow(config: AgentConfig): number {
  // 優先順位:
  // 1. モデル API からの値
  // 2. 設定ファイルの値
  // 3. エージェントデフォルト
  // 4. フォールバック: 200,000 トークン

  const modelWindow = getModelContextWindow(config.model);
  const configWindow = config.contextWindow;
  const agentDefault = config.agent?.contextWindow;
  const fallback = 200_000;

  return modelWindow ?? configWindow ?? agentDefault ?? fallback;
}
```

### 制限値

```typescript
const CONTEXT_LIMITS = {
  HARD_MINIMUM: 16_000,      // これ未満は実行拒否
  WARNING_THRESHOLD: 32_000, // 警告を出す閾値
  DEFAULT_WINDOW: 200_000,   // デフォルト値
};
```

### トークンカウント

```typescript
async function countTokens(content: string, model: string): Promise<number> {
  // モデル固有のトークナイザーを使用
  // Anthropic: claude-tokenizer
  // OpenAI: tiktoken
  // その他: 概算（文字数 / 4）

  const tokenizer = getTokenizer(model);
  return tokenizer.count(content);
}
```

## 5. コンテキストプルーニング

### プルーニング戦略

```
コンテキスト使用率
│
├── 0-30%: プルーニングなし
│
├── 30-50%: ソフトトリム
│   └── 古いメッセージを要約して保持
│
└── 50%+: ハードクリア
    └── 古いメッセージを完全削除
```

### ソフトトリム

```typescript
async function softTrim(
  history: Message[],
  targetTokens: number
): Promise<Message[]> {
  const protected = history.slice(-3); // 最後の3メッセージを保護
  const toTrim = history.slice(0, -3);

  // 要約を生成
  const summary = await summarize(toTrim);

  // 要約メッセージとして挿入
  return [
    {
      role: "system",
      content: `[Previous conversation summary]\n${summary}`,
    },
    ...protected,
  ];
}
```

### ハードクリア

```typescript
function hardClear(
  history: Message[],
  keepLast: number = 3
): Message[] {
  // 最後の N メッセージのみ保持
  return history.slice(-keepLast);
}
```

### プルーニングフロー

```typescript
async function pruneIfNeeded(
  history: Message[],
  config: PruningConfig
): Promise<Message[]> {
  const totalTokens = await countTokens(history);
  const windowSize = config.contextWindow;
  const usage = totalTokens / windowSize;

  if (usage < 0.3) {
    // プルーニング不要
    return history;
  }

  if (usage < 0.5) {
    // ソフトトリム
    const targetTokens = windowSize * 0.2; // 20% まで削減
    return softTrim(history, targetTokens);
  }

  // ハードクリア
  return hardClear(history, config.keepLastMessages);
}
```

## 6. メモリシステム

### 短期メモリ（セッションヒストリー）

```typescript
interface ShortTermMemory {
  // セッション内の会話履歴
  history: Message[];

  // ターン数制限
  maxTurns: number;

  // プロバイダー別制限
  perProviderLimits: {
    anthropic: 50,
    openai: 100,
    // ...
  };
}
```

### 長期メモリ（ベクトル埋め込み）

```typescript
interface LongTermMemory {
  // ストレージ
  storage: "sqlite-vec" | "lancedb";

  // 埋め込みプロバイダー
  embeddingProvider: "openai" | "gemini" | "local";

  // 検索設定
  search: {
    // ハイブリッド検索の重み
    semanticWeight: 0.7,  // ベクトル類似度
    keywordWeight: 0.3,   // BM25

    // 結果数
    topK: 10,
  };
}
```

### メモリストレージ構造

```
~/.clawdbot/
└── memory/
    ├── embeddings.db      # SQLite + sqlite-vec
    ├── index/             # ベクトルインデックス
    └── cache/             # 埋め込みキャッシュ
```

### メモリ検索フロー

```typescript
async function searchMemory(
  query: string,
  config: MemoryConfig
): Promise<MemoryResult[]> {
  // 1. クエリを埋め込みに変換
  const embedding = await embed(query, config.embeddingProvider);

  // 2. ベクトル検索（セマンティック）
  const semanticResults = await vectorSearch(embedding, {
    topK: config.search.topK,
    threshold: 0.7,
  });

  // 3. キーワード検索（BM25）
  const keywordResults = await bm25Search(query, {
    topK: config.search.topK,
  });

  // 4. ハイブリッドスコアリング
  const combined = hybridScore(
    semanticResults,
    keywordResults,
    config.search.semanticWeight,
    config.search.keywordWeight
  );

  // 5. 重複除去と上位K件を返却
  return dedupe(combined).slice(0, config.search.topK);
}
```

### メモリの保存

```typescript
async function storeMemory(
  content: string,
  metadata: MemoryMetadata
): Promise<void> {
  // 1. コンテンツを埋め込みに変換
  const embedding = await embed(content);

  // 2. メタデータと共に保存
  await db.insert({
    content,
    embedding,
    metadata: {
      sessionId: metadata.sessionId,
      timestamp: Date.now(),
      source: metadata.source,
      tags: metadata.tags,
    },
  });
}
```

## 7. ターン数制限

### 設定

```typescript
interface TurnLimits {
  // グローバル制限
  global: 100;

  // プロバイダー別制限
  perProvider: {
    anthropic: 50,
    openai: 100,
    gemini: 80,
  };

  // ユーザー別制限
  perUser: 30;
}
```

### 適用フロー

```typescript
function limitTurns(history: Message[], config: TurnConfig): Message[] {
  let limit = config.global;

  // プロバイダー別制限を適用
  if (config.perProvider[config.currentProvider]) {
    limit = Math.min(limit, config.perProvider[config.currentProvider]);
  }

  // ユーザー別制限を適用
  if (config.perUser) {
    limit = Math.min(limit, config.perUser);
  }

  // 制限を超える場合は古いものから削除
  if (history.length > limit * 2) { // user + assistant で 2 メッセージ
    return history.slice(-(limit * 2));
  }

  return history;
}
```

## 8. 完全なコンテキストフロー図

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        新規メッセージ受信                                │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  1. セッション解決                                                       │
│     - セッションキー導出                                                 │
│     - 既存セッション読み込み or 新規作成                                  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  2. ヒストリー読み込み                                                   │
│     - JSONL からメッセージ読み込み                                       │
│     - ターン数制限適用                                                   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  3. ブートストラップファイル読み込み                                      │
│     - AGENTS.md, SOUL.md, etc.                                         │
│     - 長いファイルは切り詰め（70% head + 20% tail）                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  4. システムプロンプト構築                                               │
│     - 21セクションを組み立て                                             │
│     - モード適用（full / minimal / none）                               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  5. コンテキストウィンドウ検証                                           │
│     - トークン数カウント                                                 │
│     - 最小閾値チェック（16K）                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  6. プルーニング（必要に応じて）                                         │
│     - 30-50%: ソフトトリム（要約）                                       │
│     - 50%+: ハードクリア（削除）                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  7. メモリ検索（オプション）                                             │
│     - ベクトル検索（70%）                                                │
│     - キーワード検索（30%）                                              │
│     - ハイブリッドスコアリング                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  8. LLM リクエスト送信                                                   │
│     {                                                                   │
│       systemPrompt: "...",                                             │
│       messages: [...history, newMessage],                              │
│       tools: [...availableTools],                                      │
│       memoryContext: "..."  // オプション                               │
│     }                                                                   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  9. レスポンス処理                                                       │
│     - ストリーミング処理                                                 │
│     - ツール実行ループ                                                   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  10. セッション更新                                                      │
│      - ユーザーメッセージを JSONL に追加                                 │
│      - アシスタントレスポンスを JSONL に追加                             │
│      - メタデータ更新（トークン使用量など）                               │
│      - オプション: 長期メモリに保存                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

## 9. 設定例

### コンテキスト管理設定

```json
{
  "agents": {
    "default": {
      "contextWindow": 200000,
      "maxTurns": 50,
      "pruning": {
        "enabled": true,
        "softTrimThreshold": 0.3,
        "hardClearThreshold": 0.5,
        "keepLastMessages": 3
      }
    }
  },
  "memory": {
    "enabled": true,
    "storage": "sqlite-vec",
    "embedding": {
      "provider": "openai",
      "model": "text-embedding-3-small"
    },
    "search": {
      "topK": 10,
      "semanticWeight": 0.7,
      "keywordWeight": 0.3,
      "threshold": 0.7
    }
  }
}
```

## 10. 主要ファイル一覧

| カテゴリ | パス | 説明 |
|---------|------|------|
| セッション | `src/config/sessions/` | セッション型定義、キー、ストレージ |
| セッション | `src/auto-reply/reply/session.ts` | セッション解決ロジック |
| コンテキスト構築 | `src/agents/system-prompt.ts` | システムプロンプト構築 |
| コンテキスト構築 | `src/agents/bootstrap-files.ts` | ブートストラップファイル読み込み |
| コンテキスト構築 | `src/agents/pi-embedded-helpers/bootstrap.ts` | Pi 統合用ブートストラップ |
| プルーニング | `src/agents/pi-extensions/context-pruning/` | コンテキストプルーニング |
| ヒストリー | `src/agents/pi-embedded-runner/history.ts` | ヒストリー管理 |
| メモリ | `src/memory/manager.ts` | メモリマネージャー |
| メモリ | `src/agents/memory-search.ts` | メモリ検索 |

## まとめ

Clawdbot のコンテキスト管理は以下の特徴を持ちます:

1. **多層ストレージ**: JSONL（短期）+ ベクトルDB（長期）
2. **スマートプルーニング**: 使用率に応じたソフト/ハード切り詰め
3. **ハイブリッド検索**: セマンティック + キーワードの組み合わせ
4. **柔軟な設定**: プロバイダー別、ユーザー別の制限
5. **効率的な構築**: 21セクションの動的システムプロンプト
