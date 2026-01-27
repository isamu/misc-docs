# Clawdbot 長期記憶システム

## 概要

Clawdbot の長期記憶システムは、会話やワークスペースの情報をベクトル埋め込みとして保存し、セマンティック検索で取得する仕組みです。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        長期記憶システム                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐              │
│  │  MEMORY.md  │     │  memory/*.md │     │  Sessions   │              │
│  │  (手動)     │     │  (自動生成)  │     │  (JSONL)    │              │
│  └──────┬──────┘     └──────┬──────┘     └──────┬──────┘              │
│         │                   │                   │                      │
│         └───────────────────┼───────────────────┘                      │
│                             │                                          │
│                             ▼                                          │
│                   ┌──────────────────┐                                 │
│                   │    Chunking      │                                 │
│                   │  (400トークン)   │                                 │
│                   └────────┬─────────┘                                 │
│                            │                                           │
│                            ▼                                           │
│                   ┌──────────────────┐                                 │
│                   │   Embedding      │                                 │
│                   │  Provider        │                                 │
│                   │  (OpenAI/Gemini/ │                                 │
│                   │   Local)         │                                 │
│                   └────────┬─────────┘                                 │
│                            │                                           │
│                            ▼                                           │
│                   ┌──────────────────┐                                 │
│                   │    SQLite +      │                                 │
│                   │   sqlite-vec     │                                 │
│                   │   (ベクトルDB)   │                                 │
│                   └──────────────────┘                                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## 1. ストレージ構造

### データベースの場所

```
~/.clawdbot/
├── state/
│   └── memory/
│       └── {agentId}.sqlite    # メインDB
└── agents/
    └── {agentId}/
        └── sessions/
            └── *.jsonl          # セッション履歴（ソース）
```

### SQLite テーブル構造

```sql
-- メタデータ（インデックス情報）
CREATE TABLE meta (
    key TEXT PRIMARY KEY,
    value TEXT
);
-- 例: model, provider, chunking_config, vector_dims

-- ソースファイル追跡
CREATE TABLE files (
    path TEXT PRIMARY KEY,
    hash TEXT,              -- SHA256
    mtime INTEGER,
    size INTEGER
);

-- チャンク（メインデータ）
CREATE TABLE chunks (
    id TEXT PRIMARY KEY,    -- hash(source:path:lines:model)
    text TEXT,
    embedding TEXT,         -- JSON配列
    source TEXT,            -- "memory" | "sessions"
    file_path TEXT,
    start_line INTEGER,
    end_line INTEGER
);

-- ベクトル検索用（sqlite-vec 拡張）
CREATE VIRTUAL TABLE chunks_vec USING vec0 (
    id TEXT PRIMARY KEY,
    embedding FLOAT[1536]   -- 次元数は動的
);

-- 全文検索用（FTS5）
CREATE VIRTUAL TABLE chunks_fts USING fts5 (
    id,
    text,
    content='chunks'
);

-- 埋め込みキャッシュ
CREATE TABLE embedding_cache (
    provider TEXT,
    model TEXT,
    provider_key TEXT,      -- API設定のハッシュ
    hash TEXT,              -- コンテンツのハッシュ
    embedding TEXT,         -- JSON配列
    dims INTEGER,
    PRIMARY KEY (provider, model, provider_key, hash)
);
```

## 2. データソース

### メモリファイル（手動）

```
{workspace}/
├── MEMORY.md           # メインメモリファイル
└── memory/
    ├── 2024-01-15-project-decisions.md
    ├── 2024-01-20-meeting-notes.md
    └── ...
```

エージェントが `memory_store` ツールで書き込むか、ユーザーが手動で作成。

### セッション（自動）

```
~/.clawdbot/agents/{agentId}/sessions/
├── session-abc123.jsonl
├── session-def456.jsonl
└── ...
```

会話履歴から自動的にインデックス（オプション機能）。

### データソースタイプ

| タイプ | 説明 | 設定 |
|-------|------|------|
| `memory` | MEMORY.md + memory/*.md | デフォルトで有効 |
| `sessions` | セッション履歴 | 実験的機能 |

## 3. 埋め込みプロバイダー

### OpenAI（デフォルト）

```typescript
// 設定
{
  provider: "openai",
  model: "text-embedding-3-small",  // または "text-embedding-3-large"
}

// API呼び出し
POST https://api.openai.com/v1/embeddings
{
  "model": "text-embedding-3-small",
  "input": ["chunk text here..."]
}

// 次元数
// text-embedding-3-small: 1536
// text-embedding-3-large: 3072
```

### Google Gemini

```typescript
// 設定
{
  provider: "gemini",
  model: "gemini-embedding-001",
}

// API呼び出し
POST https://generativelanguage.googleapis.com/v1beta/models/gemini-embedding-001:embedContent

// タスクタイプ
// - RETRIEVAL_DOCUMENT: ドキュメント埋め込み時
// - RETRIEVAL_QUERY: 検索クエリ埋め込み時
```

### Local（オフライン）

```typescript
// 設定
{
  provider: "local",
  model: "ggml-org/embeddinggemma-300M-GGUF/embeddinggemma-300M-Q8_0.gguf",
}

// 実装
// node-llama-cpp を使用（オプション依存）
// 遅延読み込みで起動を軽量に保つ
```

### 自動選択

```typescript
// provider: "auto" の場合
// 1. ローカルモデルファイルが存在 → local
// 2. OpenAI APIキーあり → openai
// 3. Gemini APIキーあり → gemini
```

### フォールバック

```typescript
// 設定
{
  provider: "openai",
  fallback: "gemini",  // openai 失敗時に gemini を使用
}

// フォールバック時は自動的に再インデックスがトリガー
```

## 4. チャンキング

### チャンク分割戦略

```typescript
// デフォルト設定
{
  chunking: {
    tokens: 400,    // 1チャンクあたり約400トークン
    overlap: 80,    // 80トークンのオーバーラップ
  }
}
```

### 分割フロー

```
[元のMarkdownファイル]
    │
    ▼
[セクション分割（見出しで区切り）]
    │
    ▼
[トークン数でチャンク化]
    │  - 400トークンごとに分割
    │  - 80トークンのオーバーラップで文脈を保持
    ▼
[チャンクリスト]
    │
    ▼
[各チャンクを埋め込み]
```

### チャンク例

```
元のテキスト (1200トークン):
┌─────────────────────────────────────────┐
│ セクション1: プロジェクト概要...          │
│ セクション2: 技術的決定事項...            │
│ セクション3: 今後の予定...               │
└─────────────────────────────────────────┘

チャンク化後:
┌──────────────┐
│ チャンク1    │ (0-400トークン)
│ overlap ─────┼──┐
└──────────────┘  │
       ┌──────────┘
       │
┌──────┴───────┐
│ チャンク2    │ (320-720トークン)
│ overlap ─────┼──┐
└──────────────┘  │
       ┌──────────┘
       │
┌──────┴───────┐
│ チャンク3    │ (640-1040トークン)
└──────────────┘
```

## 5. ハイブリッド検索

### 検索フロー

```
[検索クエリ]
    │
    ├────────────────────────────────────┐
    │                                    │
    ▼                                    ▼
┌─────────────────┐            ┌─────────────────┐
│ ベクトル検索    │            │ キーワード検索   │
│ (セマンティック) │            │ (BM25/FTS5)    │
│                 │            │                 │
│ 1. クエリを埋め込み│          │ 1. クエリを分割  │
│ 2. コサイン類似度 │          │ 2. FTS5検索     │
│ 3. 上位N件取得   │          │ 3. BM25ランキング│
└────────┬────────┘            └────────┬────────┘
         │                              │
         └──────────────┬───────────────┘
                        │
                        ▼
              ┌──────────────────┐
              │  ハイブリッド     │
              │  スコアリング    │
              │                  │
              │ 70% ベクトル     │
              │ 30% キーワード   │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │  フィルタリング   │
              │  (minScore: 0.35)│
              └────────┬─────────┘
                       │
                       ▼
              [上位6件の結果]
```

### スコア計算

```typescript
// ベクトル検索スコア
vectorScore = 1 - cosineDistance(queryEmbedding, chunkEmbedding);

// キーワード検索スコア
keywordScore = 1 / (1 + bm25Rank);

// ハイブリッドスコア
finalScore = (0.7 * vectorScore) + (0.3 * keywordScore);

// フィルタリング
if (finalScore >= 0.35) {
  results.push(chunk);
}
```

### 検索設定

```typescript
{
  query: {
    maxResults: 6,           // 最大結果数
    minScore: 0.35,          // 最小スコア閾値
    hybrid: {
      enabled: true,         // ハイブリッド検索有効
      vectorWeight: 0.7,     // ベクトル重み
      textWeight: 0.3,       // テキスト重み
      candidateMultiplier: 4 // 候補数の倍率（24件から6件に絞る）
    }
  }
}
```

## 6. 同期トリガー

### 自動同期タイミング

| トリガー | 設定キー | 説明 |
|---------|---------|------|
| ファイル監視 | `sync.watch` | MEMORY.md/memory/ の変更検出時 |
| セッション開始 | `sync.onSessionStart` | 新しいセッション開始時 |
| 検索時 | `sync.onSearch` | 検索前にdirtyならsync |
| 定期実行 | `sync.intervalMinutes` | N分ごとに実行 |
| セッションデルタ | `sync.sessions.deltaBytes/Messages` | 閾値超過時 |

### デバウンス設定

```typescript
{
  sync: {
    watch: true,
    watchDebounceMs: 1500,    // ファイル変更後1.5秒待機
    // セッション同期は5秒デバウンス
  }
}
```

### セッションデルタ

```typescript
// セッション履歴からのインクリメンタルインデックス
{
  sync: {
    sessions: {
      deltaBytes: 100_000,    // 100KB以上の変更
      deltaMessages: 50,      // 50メッセージ以上の変更
    }
  }
}
```

## 7. メモリプラグイン比較

### memory-core（デフォルト）

```typescript
// 特徴
- SQLite + sqlite-vec ベース
- 複数の埋め込みプロバイダー対応
- ファイル + セッションソース
- ハイブリッド検索
- 手動トリガー

// 提供ツール
- memory_search: セマンティック検索
- memory_get: ファイル読み取り

// 設定不要（APIキーのみ）
```

### memory-lancedb（代替）

```typescript
// 特徴
- LanceDB ベース（列指向ベクトルDB）
- OpenAIのみ対応
- 会話からの自動キャプチャ
- コンテキストへの自動注入
- 重要度スコアリング

// メモリカテゴリ
- preference: ユーザーの好み
- fact: 事実情報
- decision: 決定事項
- entity: 人物・組織
- other: その他

// 設定必須（ルールベースのトリガー設定）
```

### 比較表

| 機能 | memory-core | memory-lancedb |
|------|-------------|----------------|
| ストレージ | SQLite | LanceDB |
| プロバイダー | OpenAI/Gemini/Local | OpenAI のみ |
| 自動キャプチャ | なし（手動） | ルールベース |
| 自動リコール | なし（手動検索） | 自動注入 |
| チャンキング | トークンベース | なし |
| ソース | ファイル + セッション | 会話のみ |
| カテゴリ | なし | 5カテゴリ |
| 重要度 | なし | あり |

## 8. エージェント統合

### メモリ検索の呼び出し

```typescript
// エージェントからのツール呼び出し
await tools.memory_search({
  query: "先週のミーティングで決まったこと",
  maxResults: 6,
  minScore: 0.35,
});

// 結果
[
  {
    id: "chunk-abc123",
    text: "2024年1月15日のミーティングで、新しいAPIの設計方針を決定...",
    score: 0.87,
    source: "memory",
    file: "memory/2024-01-15-meeting.md",
    lines: [10, 25]
  },
  // ...
]
```

### 検索トリガーの例

エージェントは以下のような質問で `memory_search` を呼び出します:

- 「先週話したことを覚えてる？」
- 「以前決めた方針は何だった？」
- 「私の好みは？」
- 「プロジェクトXについて教えて」

### セッションメモリフック

```typescript
// /new コマンド時のセッション保存
// src/hooks/bundled/session-memory/handler.ts

// 1. セッション履歴の最後15行を抽出
// 2. LLMでスラッグを生成（例: "api-design-discussion"）
// 3. memory/YYYY-MM-DD-{slug}.md として保存
// 4. 次回の検索で自動的にインデックス
```

## 9. キャッシュシステム

### 埋め込みキャッシュ

```typescript
// キャッシュキー
key = (provider, model, provider_key_hash, content_hash)

// 例
("openai", "text-embedding-3-small", "abc123", "def456")

// キャッシュヒット時は埋め込みAPI呼び出しをスキップ
// コンテンツが変更されていなければ再利用
```

### キャッシュ設定

```typescript
{
  cache: {
    enabled: true,
    maxEntries: undefined,  // 無制限（デフォルト）
  }
}

// maxEntries 設定時は LRU で古いエントリを削除
```

### リインデックス時のキャッシュ保持

```typescript
// 安全なリインデックスフロー
// 1. 一時DBを作成
// 2. 既存キャッシュをシード
// 3. 新しいチャンクを処理
// 4. 成功時にアトミックスワップ
// 5. 失敗時はバックアップから復元
```

## 10. バッチ処理

### OpenAI Batch API

```typescript
{
  remote: {
    batch: {
      enabled: true,
      concurrency: 4,       // 並行ワーカー数
      timeoutMinutes: 120,  // タイムアウト
      wait: true,           // 完了を待機
    }
  }
}

// バッチ処理のメリット
// - コスト削減（約50%オフ）
// - レート制限回避
// - 24時間以内の完了保証
```

### バッチフロー

```
[大量のチャンク]
    │
    ▼
[8000トークンごとにバッチ化]
    │
    ▼
[POST /v1/batches でジョブ作成]
    │
    ▼
[ポーリングで完了待機]
    │
    ▼
[結果取得・DBに保存]
```

### エラーハンドリング

```typescript
// リトライ戦略
maxRetries: 3
backoff: 500ms → 1000ms → 2000ms → ... → 8000ms

// バッチ失敗時
// - 2回失敗で自動的にバッチを無効化
// - 直接埋め込みにフォールバック
```

## 11. 設定リファレンス

### 完全な設定例

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "enabled": true,
        "provider": "openai",
        "fallback": "gemini",
        "model": "text-embedding-3-small",
        "sources": ["memory", "sessions"],

        "store": {
          "driver": "sqlite",
          "path": "~/.clawdbot/state/memory/{agentId}.sqlite",
          "vector": {
            "enabled": true,
            "extensionPath": null
          }
        },

        "chunking": {
          "tokens": 400,
          "overlap": 80
        },

        "sync": {
          "watch": true,
          "watchDebounceMs": 1500,
          "onSearch": true,
          "onSessionStart": true,
          "intervalMinutes": 0,
          "sessions": {
            "deltaBytes": 100000,
            "deltaMessages": 50
          }
        },

        "query": {
          "maxResults": 6,
          "minScore": 0.35,
          "hybrid": {
            "enabled": true,
            "vectorWeight": 0.7,
            "textWeight": 0.3,
            "candidateMultiplier": 4
          }
        },

        "cache": {
          "enabled": true,
          "maxEntries": null
        },

        "remote": {
          "batch": {
            "enabled": false,
            "concurrency": 4,
            "timeoutMinutes": 120,
            "wait": true
          }
        }
      }
    }
  }
}
```

## 12. 統計情報

| 項目 | デフォルト値 |
|------|-------------|
| チャンクサイズ | 400 トークン |
| オーバーラップ | 80 トークン |
| 検索結果数 | 6 件 |
| 最小スコア | 0.35 (35%) |
| ベクトル重み | 70% |
| キーワード重み | 30% |
| 候補倍率 | 4x |
| 監視デバウンス | 1500ms |
| セッションデバウンス | 5000ms |
| ベクトル読み込みタイムアウト | 30秒 |
| クエリタイムアウト | 60秒 (リモート) / 300秒 (ローカル) |
| バッチタイムアウト | 120秒 (リモート) / 600秒 (ローカル) |
| バッチ最大トークン | 8000/リクエスト |

## まとめ

Clawdbot の長期記憶システムは:

1. **SQLite + sqlite-vec** によるベクトル検索
2. **複数の埋め込みプロバイダー** 対応（OpenAI/Gemini/Local）
3. **ハイブリッド検索**（70% セマンティック + 30% キーワード）
4. **効率的なキャッシュ** による API 呼び出し削減
5. **自動同期** によるリアルタイム更新
6. **バッチ処理** によるコスト最適化

を実現した、本格的なメモリシステムです。
