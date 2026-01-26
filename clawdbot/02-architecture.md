# Clawdbot アーキテクチャ詳細

## システム全体像

Clawdbot は、以下の主要コンポーネントで構成されるマイクロサービス的アーキテクチャを採用しています。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           外部サービス                                  │
├─────────────────────────────────────────────────────────────────────────┤
│  WhatsApp API │ Telegram API │ Discord API │ Slack API │ Signal CLI    │
│  iMessage     │ Google Chat  │ LINE API    │ Matrix    │ MS Teams      │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      Channel Adapters Layer                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │Telegram  │ │ Discord  │ │  Slack   │ │  Signal  │ │ iMessage │      │
│  │Adapter   │ │ Adapter  │ │ Adapter  │ │ Adapter  │ │ Adapter  │      │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘      │
│       │            │            │            │            │             │
│       └────────────┴────────────┴────────────┴────────────┘             │
│                                 │                                        │
│                    ┌────────────▼────────────┐                          │
│                    │   Channel Registry      │                          │
│                    │   (Unified Interface)   │                          │
│                    └────────────┬────────────┘                          │
└─────────────────────────────────┼───────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Gateway Server                                   │
│  ┌──────────────────────────────────────────────────────────────┐       │
│  │                   WebSocket API Layer                        │       │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐            │       │
│  │  │ RPC     │ │ HTTP    │ │ Events  │ │ Auth    │            │       │
│  │  │ Methods │ │ Routes  │ │ Stream  │ │ Mgmt    │            │       │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘            │       │
│  └──────────────────────────────────────────────────────────────┘       │
│                                │                                         │
│  ┌─────────────────────────────┼─────────────────────────────────┐      │
│  │              Core Services  │                                 │      │
│  │  ┌─────────┐ ┌─────────┐ ┌─┴───────┐ ┌─────────┐             │      │
│  │  │ Session │ │ Routing │ │ Config  │ │ Plugin  │             │      │
│  │  │ Manager │ │ Engine  │ │ Loader  │ │ Loader  │             │      │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘             │      │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐             │      │
│  │  │ Model   │ │ Queue   │ │ Media   │ │ Cron    │             │      │
│  │  │ Catalog │ │ Manager │ │ Store   │ │ Runner  │             │      │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘             │      │
│  └───────────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Agent Runtime                                    │
│  ┌──────────────────────────────────────────────────────────────┐       │
│  │                    Pi Agent Core                             │       │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐            │       │
│  │  │ Context │ │ Tool    │ │ Stream  │ │ Memory  │            │       │
│  │  │ Builder │ │ Router  │ │ Handler │ │ Search  │            │       │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘            │       │
│  └──────────────────────────────────────────────────────────────┘       │
│                                │                                         │
│  ┌─────────────────────────────┼─────────────────────────────────┐      │
│  │                   Tool Execution Layer                        │      │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐             │      │
│  │  │  Bash   │ │ Browser │ │ Canvas  │ │ Message │             │      │
│  │  │  Exec   │ │ Control │ │  Push   │ │  Send   │             │      │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘             │      │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐             │      │
│  │  │ File    │ │ HTTP    │ │ Session │ │ Custom  │             │      │
│  │  │  Ops    │ │ Fetch   │ │ Cross   │ │ Tools   │             │      │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘             │      │
│  └───────────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         LLM Providers                                    │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │Anthropic │ │ OpenAI   │ │ Gemini   │ │ GitHub   │ │ Ollama   │      │
│  │ Claude   │ │ GPT-4    │ │          │ │ Copilot  │ │ (Local)  │      │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘      │
└─────────────────────────────────────────────────────────────────────────┘
```

## ディレクトリ構造

```
clawdbot/
├── src/                           # TypeScript ソースコード
│   ├── entry.ts                   # エントリーポイント
│   ├── cli/                       # CLI 実装 (100+ ファイル)
│   │   ├── program.ts             # メインコマンド構造
│   │   ├── run-main.ts            # CLI メインランナー
│   │   ├── gateway-cli/           # Gateway 制御
│   │   ├── daemon-cli/            # デーモン管理
│   │   ├── agent-cli/             # エージェント呼び出し
│   │   ├── plugins-cli/           # プラグイン管理
│   │   ├── models-cli/            # モデル設定
│   │   ├── skills-cli/            # スキル管理
│   │   └── progress.ts            # プログレス表示
│   │
│   ├── gateway/                   # Gateway サーバー (127+ ファイル)
│   │   ├── server.ts              # 型定義
│   │   ├── server.impl.ts         # メイン実装
│   │   ├── server-methods/        # RPC メソッドハンドラ
│   │   ├── server-models/         # モデルカタログ
│   │   ├── server-node-*/         # ノード管理
│   │   ├── client.ts              # クライアント処理
│   │   └── protocol/              # プロトコル定義
│   │
│   ├── agents/                    # エージェントランタイム (292 ファイル)
│   │   ├── pi-embedded*.ts        # Pi エージェント統合
│   │   ├── bash-tools.ts          # Bash 実行
│   │   ├── model-*.ts             # モデル選択/フォールバック
│   │   ├── skills/                # スキルローディング
│   │   ├── auth-profiles/         # 認証管理
│   │   ├── tools/                 # 組み込みツール (59 ディレクトリ)
│   │   ├── workspace.ts           # ワークスペース初期化
│   │   └── sandbox/               # サンドボックス環境
│   │
│   ├── channels/                  # チャネルハンドラ (33 ファイル)
│   │   ├── dock.ts                # チャネルメタデータ
│   │   ├── registry.ts            # チャネルレジストリ
│   │   ├── plugins/               # チャネルプラグイン
│   │   └── allowlists/            # 許可リスト管理
│   │
│   ├── telegram/                  # Telegram チャネル (73 ファイル)
│   ├── discord/                   # Discord チャネル (42 ファイル)
│   ├── slack/                     # Slack チャネル (36 ファイル)
│   ├── signal/                    # Signal チャネル (24 ファイル)
│   ├── imessage/                  # iMessage チャネル (15 ファイル)
│   ├── line/                      # LINE チャネル (34 ファイル)
│   │
│   ├── config/                    # 設定管理 (121 ファイル)
│   │   ├── config.ts              # メイン設定ローダー
│   │   ├── zod-schema.*.ts        # Zod バリデーションスキーマ
│   │   ├── types.channels.ts      # チャネル型定義
│   │   └── sessions.ts            # セッションストア
│   │
│   ├── plugins/                   # プラグインシステム (37 ファイル)
│   │   ├── loader.ts              # プラグインローダー
│   │   ├── discovery.ts           # プラグイン探索
│   │   ├── hooks.ts               # フックシステム
│   │   └── registry.ts            # プラグインレジストリ
│   │
│   ├── media/                     # メディアパイプライン (21 ファイル)
│   │   ├── image-ops.ts           # 画像処理
│   │   ├── input-files.ts         # ファイル処理
│   │   └── store.ts               # メディアストア
│   │
│   ├── routing/                   # メッセージルーティング (6 ファイル)
│   │   ├── resolve-route.ts       # ルート解決
│   │   └── session-key.ts         # セッションキー導出
│   │
│   ├── infra/                     # インフラストラクチャ (147 ファイル)
│   │   ├── outbound/              # アウトバウンド送信
│   │   └── env/                   # 環境変数
│   │
│   ├── browser/                   # ブラウザ制御 (69 ファイル)
│   ├── canvas-host/               # Canvas/A2UI ホスト (6 ファイル)
│   ├── memory/                    # メモリ/埋め込み (35 ファイル)
│   ├── sessions/                  # セッション管理 (9 ファイル)
│   ├── cron/                      # Cron 自動化 (23 ファイル)
│   ├── tui/                       # ターミナル UI (28 ファイル)
│   ├── logging/                   # ロギング (16 ファイル)
│   ├── security/                  # セキュリティ (11 ファイル)
│   └── terminal/                  # ターミナルユーティリティ
│
├── extensions/                    # プラグインパッケージ (31 ディレクトリ)
│   ├── msteams/                   # Microsoft Teams
│   ├── matrix/                    # Matrix
│   ├── zalo/                      # Zalo
│   ├── zalouser/                  # Zalo Personal
│   ├── voice-call/                # 音声通話
│   ├── memory-lancedb/            # 長期記憶
│   └── google-antigravity-auth/   # OAuth プロバイダー
│
├── apps/                          # ネイティブアプリ
│   ├── macos/                     # macOS アプリ (Swift/SwiftUI)
│   ├── ios/                       # iOS ノード (Swift)
│   └── android/                   # Android ノード (Kotlin)
│
├── skills/                        # バンドルスキル (54 ディレクトリ)
├── docs/                          # ドキュメント (51+ ファイル)
├── ui/                            # Web UI
├── test/                          # テストユーティリティ
├── scripts/                       # ビルド/リリーススクリプト
└── patches/                       # 依存関係パッチ
```

## メッセージフロー詳細

### 1. インバウンドメッセージ処理

```
[チャネルからのメッセージ受信]
    │
    ▼
[InboundMessage への正規化]
    - sender: 送信者情報抽出
    - content: メッセージ本文
    - attachments: 添付ファイル
    - chatType: DM / グループ / スレッド
    - accountId: チャネルアカウント識別子
    - chatId: チャット識別子
    │
    ▼
[Channel Dock & 許可リストチェック]
    - チャネル固有のルール適用
    - DM ペアリングポリシー確認
    - 許可リストとのマッチング
    │
    ▼
[セッション解決]
    - セッションキー導出: agentId + chatType + chatId
    - 既存セッション読み込み or 新規作成
    - アクティベーションルール確認
      - グループでのメンション検出
      - リプライタグの確認
    │
    ▼
[メッセージフォーマット]
    - メディア抽出（画像、音声、動画）
    - 音声トランスクリプション（有効時）
    - チャネル制限に応じたチャンキング
    │
    ▼
[キュー管理]
    - キューモード:
      - steer: メッセージ挿入
      - followup: 完了待機
      - collect: バッチ処理
    - デバウンス/レート制限チェック
    │
    ▼
[Agent 呼び出し]
```

### 2. エージェント処理

```
[コンテキスト構築]
    - セッション履歴読み込み
    - ブートストラップファイル注入
      - AGENTS.md: 操作指示
      - SOUL.md: ペルソナ定義
      - TOOLS.md: ツール注意事項
      - IDENTITY.md: エージェント名/特性
      - USER.md: ユーザープロファイル
    - スキル読み込み
    │
    ▼
[システムプロンプト構築]
    - 思考レベル設定
    - ツールポリシー適用
    - モデル固有の調整
    │
    ▼
[Pi Agent Runtime 実行]
    - LLM プロバイダーへのリクエスト
    - ストリーミングレスポンス処理
    │
    ▼
[ツール実行ループ]
    ┌─────────────────────┐
    │  ツール呼び出し検出  │◄──────┐
    └──────────┬──────────┘       │
               │                   │
               ▼                   │
    ┌─────────────────────┐       │
    │   ツール実行        │       │
    │   - Bash exec       │       │
    │   - Browser control │       │
    │   - Canvas push     │       │
    │   - Memory search   │       │
    │   - Message send    │       │
    └──────────┬──────────┘       │
               │                   │
               ▼                   │
    ┌─────────────────────┐       │
    │   結果をコンテキスト │       │
    │   に追加            │───────┘
    └──────────┬──────────┘
               │
               ▼
    [最終レスポンス生成]
```

### 3. アウトバウンドメッセージ処理

```
[ブロックストリーミング]
    - テキストブロック完了時に送信（有効時）
    - チャネル UI に応じたチャンク化
    │
    ▼
[アウトバウンドフォーマット]
    - チャネル固有のフォーマット適用
    - テキストチャンク制限
      - Telegram: 4000 文字
      - Discord: 2000 文字
      - Slack: 3000 文字
    - インラインツール/ボタンのフォーマット
      - Discord: Embeds
      - Slack: Blocks
    - スレッド処理
    │
    ▼
[チャネルへの送信]
    - 指数バックオフによるリトライ
    - 使用量/コスト追跡
    │
    ▼
[セッション更新]
    - アシスタントレスポンスを保存
    - 永続化
```

## Gateway サーバー詳細

### WebSocket API

Gateway は型付き WebSocket API を提供し、以下のメソッドカテゴリをサポートします:

| カテゴリ | メソッド例 |
|---------|-----------|
| **セッション** | `session.create`, `session.get`, `session.list` |
| **メッセージ** | `message.send`, `message.stream` |
| **チャネル** | `channel.status`, `channel.configure` |
| **モデル** | `model.list`, `model.select` |
| **プラグイン** | `plugin.load`, `plugin.unload` |
| **ノード** | `node.register`, `node.heartbeat` |

### HTTP ルート

```
GET  /health          # ヘルスチェック
GET  /version         # バージョン情報
POST /webhook/:id     # Webhook 受信
GET  /canvas/:id      # Canvas コンテンツ
```

## セッション管理

### セッションキー導出

```typescript
// セッションキーの構造
sessionKey = `${agentId}:${chatType}:${chatId}`

// 例
sessionKey = "default:dm:+1234567890"
sessionKey = "default:group:telegram-123456"
```

### セッションストレージ

```
~/.clawdbot/
├── agents/
│   └── <agentId>/
│       └── sessions/
│           ├── <sessionId>.jsonl     # 会話履歴
│           └── metadata.json         # セッションメタデータ
└── clawdbot.json                     # メイン設定
```

### JSONL フォーマット

```json
{"role": "user", "content": "Hello", "createdAt": 1700000000000, "meta": {...}}
{"role": "assistant", "content": "Hi there!", "createdAt": 1700000001000, "meta": {...}}
```

## 設定システム

### 設定ファイル階層

```
1. デフォルト値 (src/config/defaults.ts)
    ↓
2. グローバル設定 (~/.clawdbot/clawdbot.json)
    ↓
3. ワークスペース設定 (<workspace>/.clawdbot/config.json)
    ↓
4. 環境変数 (CLAWDBOT_*)
    ↓
5. CLI フラグ (--config, --model, etc.)
```

### メイン設定スキーマ

```typescript
interface ClawdbotConfig {
  gateway: {
    port: number;           // デフォルト: 18789
    bind: string;           // デフォルト: "127.0.0.1"
    auth: AuthConfig;
    tailscale?: TailscaleConfig;
  };

  channels: {
    telegram?: TelegramConfig;
    discord?: DiscordConfig;
    slack?: SlackConfig;
    signal?: SignalConfig;
    // ...
  };

  agents: {
    default: AgentConfig;
    workspace?: string;
    sandbox?: SandboxConfig;
  };

  models: {
    primary: string;        // 例: "claude-sonnet-4-20250514"
    fallback?: string[];
    providers: ProviderConfigs;
  };

  tools: {
    exec: ExecPolicy;
    browser?: BrowserConfig;
    allowlists: AllowlistConfig;
  };

  plugins: {
    entries: PluginEntry[];
    slots: PluginSlots;
    load: { paths: string[] };
  };

  skills: {
    gated: string[];
    bundled: boolean;
  };

  automation: {
    cron: CronJob[];
    webhooks: WebhookConfig[];
  };

  memory: {
    search: MemorySearchConfig;
  };
}
```

## 次のセクション

- [実装詳細](./03-implementation.md) - コードレベルの解説
- [機能一覧](./04-features.md) - 全機能の詳細
- [拡張ガイド](./05-extension-guide.md) - プラグイン開発
