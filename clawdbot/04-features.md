# Clawdbot 機能一覧

## 1. メッセージングチャネル

### コアチャネル（組み込み）

| チャネル | ライブラリ | 機能 |
|---------|-----------|------|
| **WhatsApp** | Baileys (Web) | DM、グループ、メディア送受信、リアクション |
| **Telegram** | grammY | DM、グループ、スレッド、インラインキーボード、ファイル共有 |
| **Discord** | discord.js | DM、サーバー、スレッド、Embeds、スラッシュコマンド |
| **Slack** | Bolt | DM、チャンネル、スレッド、Blocks、アプリホーム |
| **Signal** | signal-cli | DM、グループ、消えるメッセージ |
| **iMessage** | imsg | DM、グループ（macOS のみ） |
| **LINE** | LINE Bot SDK | DM、グループ、Flex Message |
| **Google Chat** | API | DM、スペース |
| **WebChat** | 組み込み Web UI | ブラウザベースのチャット |

### 拡張チャネル（プラグイン）

| チャネル | パッケージ | 説明 |
|---------|-----------|------|
| **Microsoft Teams** | `@clawdbot/msteams` | チーム、チャンネル、DM |
| **Matrix** | `@clawdbot/matrix` | オープンソースの分散型チャット |
| **Zalo** | `@clawdbot/zalo` | ベトナムの主要メッセンジャー |
| **Zalo Personal** | `@clawdbot/zalouser` | 個人アカウント経由 |
| **BlueBubbles** | 組み込み | Apple Messages リレー |
| **Nostr** | `@clawdbot/nostr` | 分散型ソーシャルプロトコル |
| **Twitch** | `@clawdbot/twitch` | ライブストリーミングチャット |
| **Mattermost** | `@clawdbot/mattermost` | オープンソース Slack 代替 |
| **Nextcloud Talk** | `@clawdbot/nextcloud-talk` | セルフホスト型チャット |

### チャネル機能比較

```
                  DM  Group Thread Media Stream Commands
WhatsApp          ✓    ✓     -      ✓     -       -
Telegram          ✓    ✓     ✓      ✓     ✓       ✓
Discord           ✓    ✓     ✓      ✓     ✓       ✓
Slack             ✓    ✓     ✓      ✓     ✓       ✓
Signal            ✓    ✓     -      ✓     -       -
iMessage          ✓    ✓     -      ✓     -       -
LINE              ✓    ✓     -      ✓     -       -
MS Teams          ✓    ✓     ✓      ✓     -       ✓
Matrix            ✓    ✓     ✓      ✓     -       -
```

## 2. AI/LLM プロバイダー

### サポートプロバイダー

| プロバイダー | 認証方式 | モデル例 |
|-------------|---------|---------|
| **Anthropic** | OAuth / API Key | Claude Opus 4.5, Claude Sonnet 4 |
| **OpenAI** | OAuth / API Key | GPT-4o, GPT-4 Turbo |
| **Google Gemini** | OAuth / CLI Auth | Gemini 1.5 Pro, Gemini 1.5 Flash |
| **GitHub Copilot** | Device Login | Copilot |
| **Amazon Bedrock** | AWS SDK | Claude, Titan |
| **Qwen** | Portal OAuth | Qwen-Max, Qwen-Plus |
| **Ollama** | ローカル | Llama, Mistral, etc. |
| **Minimax** | API Key | abab6.5 |

### モデル選択機能

```typescript
// 設定例
{
  "models": {
    "primary": "claude-sonnet-4-20250514",
    "fallback": [
      "gpt-4o",
      "gemini-1.5-pro"
    ],
    "providers": {
      "anthropic": {
        "apiKey": "sk-ant-..."
      },
      "openai": {
        "apiKey": "sk-..."
      }
    }
  }
}
```

### 機能

- **フォールバックチェーン**: プライマリモデル失敗時に自動切り替え
- **認証プロファイルローテーション**: 複数のAPIキーを循環
- **拡張思考（Extended Thinking）**: 深い推論モードのサポート
- **コンテキストウィンドウガード**: トークン制限の自動管理

## 3. エージェントツール

### 組み込みツール

| カテゴリ | ツール | 説明 |
|---------|-------|------|
| **実行** | `bash_exec` | シェルコマンド実行 |
| | `python_exec` | Python コード実行 |
| **ファイル** | `file_read` | ファイル読み取り |
| | `file_write` | ファイル書き込み |
| | `file_search` | ファイル検索 |
| **ブラウザ** | `browser_navigate` | URL へナビゲート |
| | `browser_click` | 要素クリック |
| | `browser_type` | テキスト入力 |
| | `browser_screenshot` | スクリーンショット |
| **Canvas** | `canvas_push` | A2UI HTML をプッシュ |
| | `canvas_clear` | Canvas クリア |
| **メッセージ** | `message_send` | 他チャネルへ送信 |
| | `message_reply` | 現在のチャットに返信 |
| **メモリ** | `memory_search` | ベクトル検索 |
| | `memory_store` | 情報を保存 |
| **セッション** | `session_switch` | 別セッションに切り替え |
| | `session_list` | セッション一覧 |
| **HTTP** | `http_fetch` | HTTP リクエスト |
| **カメラ** | `camera_snap` | 写真撮影 |
| | `camera_clip` | ビデオ録画 |

### ツールポリシー

```typescript
// 設定例
{
  "tools": {
    "exec": {
      "mode": "approval",    // "allow", "deny", "approval"
      "allowlist": [
        "ls",
        "cat",
        "grep",
        "git *"
      ],
      "denylist": [
        "rm -rf",
        "sudo *"
      ]
    },
    "browser": {
      "enabled": true,
      "headless": true,
      "timeout": 30000
    }
  }
}
```

## 4. スキルシステム

### スキルとは

スキルは、特定のタスクに対するエージェントの能力を拡張するプロンプトベースのモジュールです。

### バンドルスキル（54種類）

```
skills/
├── coding/
│   ├── code-review/
│   ├── debug/
│   ├── refactor/
│   └── test-write/
├── writing/
│   ├── summarize/
│   ├── translate/
│   └── proofread/
├── research/
│   ├── web-search/
│   └── fact-check/
├── productivity/
│   ├── calendar/
│   ├── todo/
│   └── email/
└── ...
```

### スキル構造

```
skills/my-skill/
├── SKILL.md              # スキル定義（必須）
├── README.md             # ドキュメント
└── examples/             # 使用例
```

### SKILL.md フォーマット

```markdown
---
name: my-skill
description: A brief description
triggers:
  - "keyword1"
  - "keyword2"
gated: false
---

# Instructions

Detailed instructions for the agent...

## Examples

Example inputs and outputs...
```

### スキル管理コマンド

```bash
# スキル一覧
clawdbot skills list

# スキル詳細
clawdbot skills show <skill-name>

# スキル有効化/無効化
clawdbot skills enable <skill-name>
clawdbot skills disable <skill-name>

# カスタムスキル追加
clawdbot skills add <path>
```

## 5. メディア処理

### サポートフォーマット

| カテゴリ | フォーマット | 処理 |
|---------|-------------|------|
| **画像** | PNG, JPG, WebP, GIF | リサイズ、圧縮、EXIF除去 |
| **音声** | MP3, WAV, OGG, M4A | トランスクリプション |
| **動画** | MP4, MOV | フレーム抽出、音声トランスクリプション |
| **文書** | PDF, DOCX | テキスト/画像抽出 |

### メディアパイプライン

```
[入力メディア]
    │
    ▼
[MIME タイプ検出]
    │
    ├── 画像 → [リサイズ/圧縮] → [Base64 エンコード]
    │
    ├── 音声 → [トランスクリプション] → [テキスト]
    │
    ├── 動画 → [フレーム抽出] + [音声トランスクリプション]
    │
    └── 文書 → [テキスト抽出] + [画像抽出]
    │
    ▼
[エージェントコンテキストに注入]
```

### トランスクリプションプロバイダー

- **OpenAI Whisper**: 高精度の音声認識
- **Anthropic**: Claude ネイティブの音声処理
- **Edge TTS**: Microsoft Edge のテキスト読み上げ

## 6. 音声機能

### Voice Wake（音声起動）

常時リスニングでウェイクワードを検出し、エージェントを起動します。

```bash
# 有効化
clawdbot config set voice.wake.enabled true
clawdbot config set voice.wake.keyword "Hey Claude"
```

### Talk Mode（会話モード）

連続的な音声会話を可能にするモードです。

```bash
# 開始
clawdbot talk

# オプション
clawdbot talk --language ja --voice alloy
```

### TTS（テキスト読み上げ）

エージェントのレスポンスを音声で読み上げます。

**プロバイダー:**
- **ElevenLabs**: 高品質な音声合成
- **OpenAI TTS**: GPT ベースの音声
- **Edge TTS**: 無料の Microsoft 音声

## 7. Canvas/A2UI

### 概要

Canvas は、エージェントが生成した HTML コンテンツをインタラクティブに表示するシステムです。A2UI (Agent-to-UI) プロトコルを使用します。

### 機能

- **リアルタイムプッシュ**: エージェントから UI への即座の更新
- **インタラクティブ**: ユーザー入力をエージェントにフィードバック
- **マルチプラットフォーム**: macOS、iOS、Web で動作

### 使用例

```typescript
// エージェントからの Canvas プッシュ
await tools.canvas_push({
  content: `
    <div class="chart">
      <h2>売上データ</h2>
      <canvas id="sales-chart"></canvas>
    </div>
    <script>
      // Chart.js でグラフ描画
      new Chart(document.getElementById('sales-chart'), {...});
    </script>
  `,
});
```

## 8. 自動化

### Cron ジョブ

定期的なタスクをスケジュール実行します。

```typescript
// 設定例
{
  "automation": {
    "cron": [
      {
        "name": "daily-summary",
        "schedule": "0 9 * * *",      // 毎日 9:00
        "agentId": "default",
        "message": "今日の予定を教えて",
        "channel": "telegram",
        "chatId": "123456789"
      },
      {
        "name": "weekly-report",
        "schedule": "0 18 * * 5",     // 毎週金曜 18:00
        "agentId": "default",
        "message": "今週のサマリーを作成して",
        "channel": "slack",
        "chatId": "#reports"
      }
    ]
  }
}
```

### Webhook

外部サービスからのイベントを受信してエージェントをトリガーします。

```typescript
{
  "automation": {
    "webhooks": [
      {
        "id": "github-push",
        "path": "/webhook/github",
        "secret": "your-secret",
        "agentId": "default",
        "template": "GitHub にプッシュがありました: {{payload.commits}}"
      }
    ]
  }
}
```

### Gmail Pub/Sub

Gmail の新着メールをリアルタイムで受信します。

```bash
# 設定
clawdbot config set automation.gmail.enabled true
clawdbot config set automation.gmail.pubsubTopic "projects/.../topics/..."
```

## 9. メモリシステム

### 短期メモリ（セッション）

- セッションごとの会話履歴
- 自動コンパクション（要約）
- JSONL 形式での永続化

### 長期メモリ（ベクトルDB）

```typescript
// memory-core プラグイン（デフォルト）
{
  "plugins": {
    "entries": [
      { "id": "memory-core", "enabled": true }
    ]
  }
}

// memory-lancedb プラグイン（高性能）
{
  "plugins": {
    "entries": [
      { "id": "memory-lancedb", "enabled": true }
    ]
  }
}
```

### メモリ検索

```typescript
// ツールとしての使用
await tools.memory_search({
  query: "先週の会議で決まったこと",
  limit: 5,
});

// 結果
[
  { content: "...", score: 0.95, timestamp: "2024-01-15" },
  { content: "...", score: 0.87, timestamp: "2024-01-14" },
  // ...
]
```

## 10. ブートストラップファイル

### ワークスペースファイル

| ファイル | 目的 |
|---------|------|
| `AGENTS.md` | 操作指示、ルール、制約 |
| `SOUL.md` | ペルソナ、トーン、境界 |
| `TOOLS.md` | ツールの使用注意事項 |
| `IDENTITY.md` | エージェント名、特性、絵文字 |
| `USER.md` | ユーザープロファイル、好み |
| `BOOTSTRAP.md` | 初回実行時の儀式 |

### 例: AGENTS.md

```markdown
# Operating Instructions

## Core Behaviors
- Always respond in Japanese unless asked otherwise
- Be concise and direct
- Ask clarifying questions when needed

## Constraints
- Never execute destructive commands without confirmation
- Respect user privacy
- Stay within the scope of the conversation

## Preferences
- Use bullet points for lists
- Include code examples when explaining technical concepts
```

## 11. セキュリティ機能

### 認証

- **Gateway 認証**: トークン、OAuth
- **チャネル許可リスト**: 承認済みユーザーのみ
- **DM ペアリング**: 初回接続時の承認フロー

### サンドボックス

- **ワークスペース分離**: エージェントごとの作業ディレクトリ
- **実行ポリシー**: コマンドの許可/拒否リスト
- **ネットワーク制限**: オプションの外部アクセス制限

### クレデンシャル管理

```
~/.clawdbot/
├── credentials/              # 暗号化されたクレデンシャル
│   ├── anthropic.json
│   ├── openai.json
│   └── ...
├── sessions/                 # セッション認証トークン
└── clawdbot.json            # メイン設定（シークレットなし）
```

## 12. CLI コマンド一覧

### 主要コマンド

```bash
# Gateway 管理
clawdbot gateway run          # Gateway 起動
clawdbot gateway stop         # Gateway 停止
clawdbot gateway status       # ステータス確認

# エージェント
clawdbot agent --message "..."  # メッセージ送信
clawdbot agent --interactive    # 対話モード

# チャネル
clawdbot channels status      # チャネルステータス
clawdbot channels configure   # チャネル設定

# 設定
clawdbot config get <key>     # 設定取得
clawdbot config set <key> <value>  # 設定変更
clawdbot config list          # 全設定一覧

# モデル
clawdbot models list          # 利用可能モデル
clawdbot models select        # モデル選択

# プラグイン
clawdbot plugins list         # プラグイン一覧
clawdbot plugins install <id> # インストール
clawdbot plugins enable <id>  # 有効化

# スキル
clawdbot skills list          # スキル一覧
clawdbot skills enable <name> # 有効化

# ユーティリティ
clawdbot doctor               # 診断ツール
clawdbot onboard              # セットアップウィザード
clawdbot wizard               # フルセットアップ
clawdbot login                # Web プロバイダーログイン
```

## 次のセクション

- [拡張ガイド](./05-extension-guide.md) - プラグイン開発方法
