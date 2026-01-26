# Clawdbot 実装詳細

## エントリーポイント

### メインエントリー (`src/entry.ts`)

```typescript
// Node.js 環境の正規化
// 1. 警告ハンドリング
// 2. 必要に応じて再スポーン
// 3. Windows argv 正規化
// 4. プロファイル引数のパース

import { runMain } from "./cli/run-main.js";
runMain();
```

### CLI プログラム (`src/cli/program.ts`)

Commander.js を使用した CLI 構造:

```typescript
import { Command } from "commander";

const program = new Command()
  .name("clawdbot")
  .description("Personal AI assistant platform")
  .version(version);

// サブコマンド登録
program
  .command("gateway")      // Gateway 制御
  .command("agent")        // エージェント実行
  .command("send")         // メッセージ送信
  .command("channels")     // チャネル管理
  .command("config")       // 設定管理
  .command("models")       // モデル設定
  .command("plugins")      // プラグイン管理
  .command("skills")       // スキル管理
  .command("onboard")      // セットアップウィザード
  .command("doctor")       // 診断ツール
  // ... その他
```

## Gateway サーバー実装

### サーバーコア (`src/gateway/server.impl.ts`)

```typescript
export class GatewayServer {
  private wss: WebSocketServer;
  private channels: ChannelRegistry;
  private models: ModelCatalog;
  private plugins: PluginRegistry;
  private sessions: SessionManager;

  async start(options: GatewayOptions): Promise<void> {
    // 1. 設定読み込み
    const config = await loadConfig(options.configPath);

    // 2. チャネル初期化
    await this.initializeChannels(config.channels);

    // 3. モデルカタログ構築
    await this.initializeModels(config.models);

    // 4. プラグイン読み込み
    await this.loadPlugins(config.plugins);

    // 5. WebSocket サーバー起動
    this.wss = new WebSocketServer({
      port: config.gateway.port,
      host: config.gateway.bind,
    });

    // 6. RPC ハンドラ登録
    this.registerMethods();

    // 7. イベントリスナー設定
    this.setupEventListeners();
  }

  private registerMethods(): void {
    // 各種 RPC メソッドの登録
    this.method("session.create", this.sessionCreate);
    this.method("session.get", this.sessionGet);
    this.method("message.send", this.messageSend);
    this.method("channel.status", this.channelStatus);
    // ...
  }
}
```

### RPC メソッドハンドラ (`src/gateway/server-methods/`)

```typescript
// server-methods/session.ts
export async function sessionCreate(
  ctx: MethodContext,
  params: SessionCreateParams
): Promise<SessionCreateResult> {
  const { agentId, chatType, chatId } = params;

  // セッションキー導出
  const sessionKey = deriveSessionKey(agentId, chatType, chatId);

  // 既存チェック
  const existing = await ctx.sessions.get(sessionKey);
  if (existing) {
    return { session: existing, created: false };
  }

  // 新規作成
  const session = await ctx.sessions.create({
    key: sessionKey,
    agentId,
    chatType,
    chatId,
    createdAt: Date.now(),
  });

  return { session, created: true };
}
```

## チャネルシステム

### チャネルレジストリ (`src/channels/registry.ts`)

```typescript
export class ChannelRegistry {
  private channels: Map<ChannelId, ChannelAdapter> = new Map();

  register(id: ChannelId, adapter: ChannelAdapter): void {
    this.channels.set(id, adapter);
  }

  async connect(id: ChannelId): Promise<void> {
    const channel = this.channels.get(id);
    if (!channel) throw new Error(`Unknown channel: ${id}`);
    await channel.connect();
  }

  async send(id: ChannelId, message: OutboundMessage): Promise<void> {
    const channel = this.channels.get(id);
    await channel.send(message);
  }
}
```

### チャネル Dock インターフェース (`src/channels/dock.ts`)

```typescript
export interface ChannelDock {
  // 基本情報
  id: ChannelId;
  name: string;
  icon: string;

  // 機能
  capabilities: {
    chatTypes: ChatType[];        // dm, group, thread
    commands: boolean;            // スラッシュコマンド対応
    streaming: boolean;           // ストリーミング対応
    reactions: boolean;           // リアクション対応
    threads: boolean;             // スレッド対応
    media: MediaCapabilities;
  };

  // 制限
  limits: {
    textChunk: number;            // 最大文字数
    mediaSize: number;            // 最大メディアサイズ
    rateLimit: RateLimit;
  };

  // ルーティング
  routing: {
    allowlistFormat: RegExp;
    groupPolicy: GroupPolicy;
    dmPairing: DmPairingPolicy;
  };

  // アダプターメソッド
  formatMessage(msg: OutboundMessage): ChannelMessage;
  parseInbound(raw: unknown): InboundMessage;
}
```

### Telegram チャネル実装 (`src/telegram/`)

```typescript
// telegram/bot.ts
import { Bot, Context } from "grammy";

export class TelegramChannel implements ChannelAdapter {
  private bot: Bot<Context>;

  constructor(private config: TelegramConfig) {
    this.bot = new Bot(config.token);
  }

  async connect(): Promise<void> {
    // ハンドラ登録
    this.bot.on("message", this.handleMessage.bind(this));
    this.bot.on("callback_query", this.handleCallback.bind(this));

    // Bot 起動
    await this.bot.start();
  }

  private async handleMessage(ctx: Context): Promise<void> {
    const inbound: InboundMessage = {
      id: ctx.message.message_id.toString(),
      chatId: ctx.chat.id.toString(),
      chatType: ctx.chat.type === "private" ? "dm" : "group",
      sender: {
        id: ctx.from.id.toString(),
        name: ctx.from.first_name,
      },
      content: ctx.message.text || "",
      attachments: await this.extractAttachments(ctx),
      timestamp: Date.now(),
    };

    // Gateway へルーティング
    await this.gateway.route(inbound);
  }

  async send(message: OutboundMessage): Promise<void> {
    const formatted = this.formatMessage(message);
    await this.bot.api.sendMessage(
      message.chatId,
      formatted.text,
      {
        parse_mode: "Markdown",
        reply_to_message_id: message.replyTo,
      }
    );
  }
}
```

## エージェントランタイム

### Pi エージェント統合 (`src/agents/pi-embedded*.ts`)

```typescript
// pi-embedded-runner.ts
import { Agent, AgentConfig, Tool } from "p-mono";

export class PiEmbeddedRunner {
  private agent: Agent;

  async run(params: RunParams): Promise<AgentResult> {
    const { sessionKey, message, config } = params;

    // 1. コンテキスト構築
    const context = await this.buildContext(sessionKey, config);

    // 2. ツール準備
    const tools = await this.prepareTools(config.tools);

    // 3. エージェント設定
    const agentConfig: AgentConfig = {
      model: config.model,
      systemPrompt: context.systemPrompt,
      tools,
      maxTokens: config.maxTokens,
      thinking: config.thinking,
    };

    // 4. 実行
    const result = await this.agent.run({
      messages: [...context.history, { role: "user", content: message }],
      config: agentConfig,
      onStream: this.handleStream.bind(this),
    });

    // 5. セッション更新
    await this.updateSession(sessionKey, message, result);

    return result;
  }

  private async buildContext(
    sessionKey: string,
    config: AgentConfig
  ): Promise<Context> {
    // セッション履歴読み込み
    const history = await this.sessions.getHistory(sessionKey);

    // ブートストラップファイル読み込み
    const bootstrap = await this.loadBootstrapFiles(config.workspace);

    // システムプロンプト構築
    const systemPrompt = this.buildSystemPrompt({
      bootstrap,
      skills: config.skills,
      thinking: config.thinking,
      toolPolicies: config.toolPolicies,
    });

    return { history, systemPrompt };
  }
}
```

### ツール実行 (`src/agents/tools/`)

```typescript
// tools/bash-exec.ts
export const bashExecTool: Tool = {
  name: "bash_exec",
  description: "Execute a bash command",
  parameters: {
    type: "object",
    properties: {
      command: {
        type: "string",
        description: "The command to execute",
      },
      cwd: {
        type: "string",
        description: "Working directory",
      },
      timeout: {
        type: "number",
        description: "Timeout in milliseconds",
      },
    },
    required: ["command"],
  },

  async execute(params: BashExecParams, ctx: ToolContext): Promise<ToolResult> {
    const { command, cwd, timeout } = params;

    // ポリシーチェック
    if (!ctx.policies.exec.isAllowed(command)) {
      return {
        success: false,
        error: "Command not allowed by policy",
      };
    }

    // 実行
    const result = await execCommand(command, {
      cwd: cwd || ctx.workspace,
      timeout: timeout || 30000,
      env: ctx.env,
    });

    return {
      success: result.exitCode === 0,
      stdout: result.stdout,
      stderr: result.stderr,
      exitCode: result.exitCode,
    };
  },
};
```

### ブラウザ制御 (`src/browser/`)

```typescript
// browser/controller.ts
import { Browser, Page } from "playwright";

export class BrowserController {
  private browser: Browser | null = null;
  private pages: Map<string, Page> = new Map();

  async launch(): Promise<void> {
    this.browser = await chromium.launch({
      headless: true,
      args: ["--no-sandbox"],
    });
  }

  async navigate(pageId: string, url: string): Promise<NavigateResult> {
    const page = await this.getOrCreatePage(pageId);
    await page.goto(url, { waitUntil: "networkidle" });

    return {
      url: page.url(),
      title: await page.title(),
      screenshot: await page.screenshot({ type: "png" }),
    };
  }

  async click(pageId: string, selector: string): Promise<ClickResult> {
    const page = this.pages.get(pageId);
    await page.click(selector);
    return { success: true };
  }

  async type(pageId: string, selector: string, text: string): Promise<void> {
    const page = this.pages.get(pageId);
    await page.fill(selector, text);
  }

  async screenshot(pageId: string): Promise<Buffer> {
    const page = this.pages.get(pageId);
    return page.screenshot({ type: "png", fullPage: true });
  }
}
```

## プラグインシステム

### プラグインローダー (`src/plugins/loader.ts`)

```typescript
import { createJiti } from "jiti";

export class PluginLoader {
  private jiti = createJiti(import.meta.url);
  private loaded: Map<string, LoadedPlugin> = new Map();

  async load(entry: PluginEntry): Promise<LoadedPlugin> {
    const { path, options } = entry;

    // jiti による動的インポート
    const module = await this.jiti.import(path);

    // プラグイン初期化
    const plugin = await module.default(options);

    // 検証
    this.validatePlugin(plugin);

    // 登録
    this.loaded.set(plugin.id, plugin);

    return plugin;
  }

  private validatePlugin(plugin: Plugin): void {
    if (!plugin.id) throw new Error("Plugin must have an id");
    if (!plugin.name) throw new Error("Plugin must have a name");
    // 追加の検証...
  }
}
```

### プラグインインターフェース

```typescript
export interface Plugin {
  // 識別
  id: string;
  name: string;
  version: string;

  // ライフサイクル
  onLoad?(ctx: PluginContext): Promise<void>;
  onUnload?(): Promise<void>;

  // 拡張
  gatewayMethods?: GatewayMethod[];
  httpRoutes?: HttpRoute[];
  agentTools?: Tool[];
  cliCommands?: CliCommand[];
  skills?: SkillDirectory[];
  backgroundServices?: BackgroundService[];

  // フック
  hooks?: {
    preMessage?(msg: InboundMessage): Promise<InboundMessage | null>;
    postMessage?(msg: OutboundMessage): Promise<OutboundMessage>;
    preToolCall?(call: ToolCall): Promise<ToolCall | null>;
    postToolCall?(result: ToolResult): Promise<ToolResult>;
  };
}
```

## メディアパイプライン

### 画像処理 (`src/media/image-ops.ts`)

```typescript
import sharp from "sharp";

export class ImageProcessor {
  async process(input: Buffer, options: ImageOptions): Promise<Buffer> {
    let pipeline = sharp(input);

    // リサイズ
    if (options.maxWidth || options.maxHeight) {
      pipeline = pipeline.resize(options.maxWidth, options.maxHeight, {
        fit: "inside",
        withoutEnlargement: true,
      });
    }

    // フォーマット変換
    if (options.format) {
      pipeline = pipeline.toFormat(options.format, {
        quality: options.quality || 80,
      });
    }

    // EXIF 削除
    if (options.stripExif) {
      pipeline = pipeline.rotate(); // 自動回転後に EXIF 削除
    }

    return pipeline.toBuffer();
  }

  async extractMetadata(input: Buffer): Promise<ImageMetadata> {
    const metadata = await sharp(input).metadata();
    return {
      width: metadata.width,
      height: metadata.height,
      format: metadata.format,
      size: input.length,
    };
  }
}
```

### 音声トランスクリプション

```typescript
// media/transcription.ts
export class Transcriber {
  async transcribe(audio: Buffer, options: TranscribeOptions): Promise<string> {
    const { provider, language } = options;

    switch (provider) {
      case "openai":
        return this.transcribeWithOpenAI(audio, language);
      case "anthropic":
        return this.transcribeWithAnthropic(audio, language);
      default:
        throw new Error(`Unknown provider: ${provider}`);
    }
  }

  private async transcribeWithOpenAI(
    audio: Buffer,
    language?: string
  ): Promise<string> {
    const response = await this.openai.audio.transcriptions.create({
      file: new File([audio], "audio.mp3"),
      model: "whisper-1",
      language,
    });
    return response.text;
  }
}
```

## 設定管理

### 設定ローダー (`src/config/config.ts`)

```typescript
import { z } from "zod";
import { configSchema } from "./zod-schema.js";

export async function loadConfig(path?: string): Promise<ClawdbotConfig> {
  // 1. デフォルト値
  let config = getDefaults();

  // 2. グローバル設定読み込み
  const globalPath = path || getGlobalConfigPath();
  if (await fileExists(globalPath)) {
    const raw = await readFile(globalPath, "utf-8");
    const parsed = JSON.parse(raw);
    config = deepMerge(config, parsed);
  }

  // 3. ワークスペース設定読み込み
  const workspacePath = await findWorkspaceConfig();
  if (workspacePath) {
    const raw = await readFile(workspacePath, "utf-8");
    const parsed = JSON.parse(raw);
    config = deepMerge(config, parsed);
  }

  // 4. 環境変数オーバーライド
  config = applyEnvOverrides(config);

  // 5. バリデーション
  const result = configSchema.safeParse(config);
  if (!result.success) {
    throw new ConfigValidationError(result.error);
  }

  return result.data;
}
```

### Zod スキーマ (`src/config/zod-schema.*.ts`)

```typescript
// zod-schema.gateway.ts
import { z } from "zod";

export const gatewaySchema = z.object({
  port: z.number().default(18789),
  bind: z.string().default("127.0.0.1"),
  auth: z.object({
    type: z.enum(["none", "token", "oauth"]).default("none"),
    token: z.string().optional(),
    oauth: z.object({
      clientId: z.string(),
      clientSecret: z.string(),
    }).optional(),
  }).default({}),
  tailscale: z.object({
    enabled: z.boolean().default(false),
    hostname: z.string().optional(),
  }).optional(),
});

// zod-schema.channels.ts
export const channelsSchema = z.object({
  telegram: z.object({
    token: z.string(),
    allowlist: z.array(z.string()).default([]),
    dmPolicy: z.enum(["allow", "deny", "pair"]).default("pair"),
  }).optional(),

  discord: z.object({
    token: z.string(),
    guildIds: z.array(z.string()).optional(),
    allowlist: z.array(z.string()).default([]),
  }).optional(),

  // 他のチャネル...
});
```

## セッション管理

### セッションストア (`src/config/sessions.ts`)

```typescript
import { appendFile, readFile, writeFile } from "fs/promises";

export class SessionStore {
  private basePath: string;

  constructor(agentId: string) {
    this.basePath = `~/.clawdbot/agents/${agentId}/sessions`;
  }

  async getHistory(sessionId: string): Promise<Message[]> {
    const path = `${this.basePath}/${sessionId}.jsonl`;

    if (!await fileExists(path)) {
      return [];
    }

    const content = await readFile(path, "utf-8");
    const lines = content.trim().split("\n").filter(Boolean);

    return lines.map(line => JSON.parse(line));
  }

  async appendMessage(sessionId: string, message: Message): Promise<void> {
    const path = `${this.basePath}/${sessionId}.jsonl`;
    const line = JSON.stringify(message) + "\n";
    await appendFile(path, line);
  }

  async compact(sessionId: string, maxMessages: number): Promise<void> {
    const history = await this.getHistory(sessionId);

    if (history.length <= maxMessages) {
      return;
    }

    // 古いメッセージを要約
    const toSummarize = history.slice(0, -maxMessages);
    const toKeep = history.slice(-maxMessages);

    const summary = await this.summarize(toSummarize);

    // 書き換え
    const path = `${this.basePath}/${sessionId}.jsonl`;
    const content = [summary, ...toKeep]
      .map(m => JSON.stringify(m))
      .join("\n") + "\n";

    await writeFile(path, content);
  }
}
```

## ルーティングエンジン

### ルート解決 (`src/routing/resolve-route.ts`)

```typescript
export async function resolveRoute(
  message: InboundMessage,
  config: RoutingConfig
): Promise<Route | null> {
  const { chatType, chatId, sender, channelId } = message;

  // 1. チャネル設定取得
  const channelConfig = config.channels[channelId];
  if (!channelConfig) {
    return null;
  }

  // 2. 許可リストチェック
  if (!matchAllowlist(sender, channelConfig.allowlist)) {
    return null;
  }

  // 3. チャットタイプによるルーティング
  if (chatType === "dm") {
    return resolveDmRoute(message, channelConfig);
  }

  if (chatType === "group") {
    return resolveGroupRoute(message, channelConfig);
  }

  if (chatType === "thread") {
    return resolveThreadRoute(message, channelConfig);
  }

  return null;
}

function resolveDmRoute(
  message: InboundMessage,
  config: ChannelConfig
): Route | null {
  const { dmPolicy, agentId } = config;

  switch (dmPolicy) {
    case "allow":
      return { agentId, chatType: "dm", chatId: message.chatId };

    case "deny":
      return null;

    case "pair":
      // ペアリング済みかチェック
      if (isPaired(message.sender.id)) {
        return { agentId, chatType: "dm", chatId: message.chatId };
      }
      return null;
  }
}
```

## 次のセクション

- [機能一覧](./04-features.md) - 全機能の詳細説明
- [拡張ガイド](./05-extension-guide.md) - プラグイン開発方法
