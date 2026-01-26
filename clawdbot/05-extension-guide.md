# Clawdbot 拡張ガイド

## 概要

Clawdbot は TypeScript ベースのプラグインシステムを提供し、以下の領域を拡張できます:

- **Gateway RPC メソッド**: 新しい WebSocket API エンドポイント
- **Gateway HTTP ルート**: カスタム HTTP エンドポイント
- **Agent ツール**: AI エージェントの新機能
- **CLI コマンド**: 新しい `clawdbot` サブコマンド
- **バックグラウンドサービス**: 常駐プロセス
- **スキル**: プロンプトベースの能力拡張
- **チャネル**: 新しいメッセージングプラットフォーム

## プラグインの基本構造

### ディレクトリ構造

```
my-plugin/
├── package.json          # パッケージ定義
├── tsconfig.json         # TypeScript 設定
├── src/
│   ├── index.ts          # エントリーポイント
│   ├── tools/            # Agent ツール
│   │   └── my-tool.ts
│   ├── commands/         # CLI コマンド
│   │   └── my-command.ts
│   ├── methods/          # Gateway RPC メソッド
│   │   └── my-method.ts
│   └── skills/           # スキル
│       └── my-skill/
│           └── SKILL.md
└── dist/                 # ビルド出力
```

### package.json

```json
{
  "name": "@clawdbot/my-plugin",
  "version": "1.0.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "types": "./dist/index.d.ts"
    }
  },
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch"
  },
  "peerDependencies": {
    "clawdbot": "^2024.0.0"
  },
  "devDependencies": {
    "clawdbot": "^2024.0.0",
    "typescript": "^5.0.0"
  }
}
```

### エントリーポイント (src/index.ts)

```typescript
import type { Plugin, PluginContext } from "clawdbot/plugin-sdk";
import { myTool } from "./tools/my-tool.js";
import { myCommand } from "./commands/my-command.js";
import { myMethod } from "./methods/my-method.js";

export default function createPlugin(options?: PluginOptions): Plugin {
  return {
    // 識別情報
    id: "my-plugin",
    name: "My Plugin",
    version: "1.0.0",
    description: "A custom Clawdbot plugin",

    // ライフサイクルフック
    async onLoad(ctx: PluginContext): Promise<void> {
      console.log("Plugin loaded:", ctx.config);
    },

    async onUnload(): Promise<void> {
      console.log("Plugin unloaded");
    },

    // 拡張
    agentTools: [myTool],
    cliCommands: [myCommand],
    gatewayMethods: [myMethod],

    // スキルディレクトリ
    skills: [
      { path: "./skills" }
    ],

    // フック
    hooks: {
      async preMessage(msg) {
        console.log("Incoming message:", msg.content);
        return msg; // null を返すとメッセージをドロップ
      },
      async postMessage(msg) {
        console.log("Outgoing message:", msg.content);
        return msg;
      },
    },
  };
}
```

## Agent ツールの作成

### ツール定義

```typescript
// src/tools/my-tool.ts
import type { Tool, ToolContext, ToolResult } from "clawdbot/plugin-sdk";

export interface MyToolParams {
  query: string;
  limit?: number;
}

export const myTool: Tool = {
  // ツール識別
  name: "my_tool",
  description: "A custom tool that does something useful",

  // パラメータスキーマ (JSON Schema)
  parameters: {
    type: "object",
    properties: {
      query: {
        type: "string",
        description: "The search query",
      },
      limit: {
        type: "number",
        description: "Maximum number of results",
        default: 10,
      },
    },
    required: ["query"],
  },

  // 実行関数
  async execute(
    params: MyToolParams,
    ctx: ToolContext
  ): Promise<ToolResult> {
    const { query, limit = 10 } = params;

    try {
      // 何らかの処理
      const results = await searchSomething(query, limit);

      return {
        success: true,
        data: results,
      };
    } catch (error) {
      return {
        success: false,
        error: error instanceof Error ? error.message : "Unknown error",
      };
    }
  },
};
```

### ToolContext の活用

```typescript
async execute(params: MyToolParams, ctx: ToolContext): Promise<ToolResult> {
  // 設定へのアクセス
  const apiKey = ctx.config.get("myPlugin.apiKey");

  // ワークスペースパス
  const workspacePath = ctx.workspace;

  // 他のツールの呼び出し
  const fileContent = await ctx.callTool("file_read", {
    path: `${workspacePath}/data.json`,
  });

  // ログ出力
  ctx.log.info("Processing query:", params.query);

  // メッセージ送信（現在のチャットへ）
  await ctx.reply("Processing your request...");

  // ...
}
```

## CLI コマンドの作成

### コマンド定義

```typescript
// src/commands/my-command.ts
import type { CliCommand } from "clawdbot/plugin-sdk";
import { Command } from "commander";

export const myCommand: CliCommand = {
  name: "my-command",
  description: "A custom CLI command",

  // Commander.js でコマンドを構築
  build(program: Command): void {
    program
      .command("my-command")
      .description("Do something custom")
      .option("-v, --verbose", "Enable verbose output")
      .option("-o, --output <path>", "Output file path")
      .argument("[input]", "Input file or value")
      .action(async (input, options) => {
        await runMyCommand(input, options);
      });
  },
};

async function runMyCommand(
  input: string | undefined,
  options: { verbose?: boolean; output?: string }
): Promise<void> {
  const { verbose, output } = options;

  if (verbose) {
    console.log("Running in verbose mode");
  }

  // コマンドのロジック
  const result = await processInput(input);

  if (output) {
    await writeFile(output, JSON.stringify(result, null, 2));
    console.log(`Output written to: ${output}`);
  } else {
    console.log(JSON.stringify(result, null, 2));
  }
}
```

### Gateway クライアントの使用

```typescript
import { createGatewayClient } from "clawdbot/plugin-sdk";

async function runMyCommand(): Promise<void> {
  // Gateway に接続
  const client = await createGatewayClient();

  // RPC メソッド呼び出し
  const sessions = await client.call("session.list", {});
  console.log("Active sessions:", sessions);

  // イベント購読
  client.on("message.received", (event) => {
    console.log("New message:", event.content);
  });

  // 接続を閉じる
  await client.close();
}
```

## Gateway RPC メソッドの作成

### メソッド定義

```typescript
// src/methods/my-method.ts
import type { GatewayMethod, MethodContext } from "clawdbot/plugin-sdk";

export interface MyMethodParams {
  action: string;
  data?: Record<string, unknown>;
}

export interface MyMethodResult {
  status: string;
  result: unknown;
}

export const myMethod: GatewayMethod<MyMethodParams, MyMethodResult> = {
  name: "myPlugin.doSomething",
  description: "Perform a custom action",

  // パラメータスキーマ
  params: {
    type: "object",
    properties: {
      action: { type: "string" },
      data: { type: "object" },
    },
    required: ["action"],
  },

  // ハンドラ
  async handler(
    ctx: MethodContext,
    params: MyMethodParams
  ): Promise<MyMethodResult> {
    const { action, data } = params;

    // 認証チェック（必要に応じて）
    if (!ctx.isAuthenticated) {
      throw new Error("Authentication required");
    }

    // アクションの実行
    switch (action) {
      case "process":
        const result = await processData(data);
        return { status: "success", result };

      case "status":
        return { status: "ok", result: null };

      default:
        throw new Error(`Unknown action: ${action}`);
    }
  },
};
```

## HTTP ルートの作成

```typescript
// src/routes/my-route.ts
import type { HttpRoute } from "clawdbot/plugin-sdk";
import type { Request, Response } from "express";

export const myRoute: HttpRoute = {
  method: "POST",
  path: "/my-plugin/webhook",

  async handler(req: Request, res: Response): Promise<void> {
    const { body, headers } = req;

    // 署名検証（必要に応じて）
    const signature = headers["x-signature"];
    if (!verifySignature(body, signature)) {
      res.status(401).json({ error: "Invalid signature" });
      return;
    }

    // ペイロード処理
    try {
      await processWebhook(body);
      res.status(200).json({ status: "ok" });
    } catch (error) {
      res.status(500).json({ error: "Processing failed" });
    }
  },
};
```

## スキルの作成

### スキルディレクトリ構造

```
skills/
└── my-skill/
    ├── SKILL.md          # スキル定義（必須）
    ├── README.md         # ドキュメント
    └── examples/
        ├── example1.md
        └── example2.md
```

### SKILL.md フォーマット

```markdown
---
name: my-skill
description: A skill for doing something specific
triggers:
  - "analyze"
  - "process data"
  - "run analysis"
gated: false
priority: 10
---

# My Skill

## Overview

This skill helps the agent perform specific analysis tasks.

## Instructions

When the user asks you to analyze data or run analysis:

1. First, ask for the data source if not provided
2. Process the data using the appropriate tools
3. Present results in a clear, structured format

## Guidelines

- Always validate input data before processing
- Provide intermediate progress updates for long operations
- Include confidence scores when making predictions

## Output Format

```json
{
  "analysis": {
    "summary": "...",
    "findings": [...],
    "recommendations": [...]
  }
}
```

## Examples

### Example 1: Basic Analysis

User: "Analyze the sales data from last month"

Expected behavior:
1. Request the data file location
2. Read and parse the data
3. Generate summary statistics
4. Present key findings
```

## チャネルプラグインの作成

### チャネルアダプター

```typescript
// src/channel/my-channel.ts
import type {
  ChannelAdapter,
  ChannelDock,
  InboundMessage,
  OutboundMessage,
} from "clawdbot/plugin-sdk";

export class MyChannelAdapter implements ChannelAdapter {
  private client: MyServiceClient;

  constructor(private config: MyChannelConfig) {
    this.client = new MyServiceClient(config);
  }

  // チャネルメタデータ
  get dock(): ChannelDock {
    return {
      id: "my-channel",
      name: "My Channel",
      icon: "📱",
      capabilities: {
        chatTypes: ["dm", "group"],
        commands: false,
        streaming: true,
        reactions: true,
        threads: false,
        media: {
          images: true,
          audio: false,
          video: false,
          files: true,
        },
      },
      limits: {
        textChunk: 4096,
        mediaSize: 10 * 1024 * 1024, // 10MB
        rateLimit: { messages: 30, window: 60000 },
      },
      routing: {
        allowlistFormat: /^[a-z0-9_]+$/,
        groupPolicy: "mention",
        dmPairing: "pair",
      },
    };
  }

  // 接続
  async connect(): Promise<void> {
    await this.client.connect();

    // メッセージハンドラ登録
    this.client.on("message", this.handleInbound.bind(this));
  }

  // 切断
  async disconnect(): Promise<void> {
    await this.client.disconnect();
  }

  // インバウンドメッセージ処理
  private async handleInbound(raw: RawMessage): Promise<void> {
    const message: InboundMessage = {
      id: raw.id,
      channelId: "my-channel",
      chatId: raw.chatId,
      chatType: raw.isGroup ? "group" : "dm",
      sender: {
        id: raw.senderId,
        name: raw.senderName,
      },
      content: raw.text,
      attachments: await this.processAttachments(raw.attachments),
      timestamp: Date.now(),
    };

    // Gateway へルーティング
    await this.gateway.route(message);
  }

  // アウトバウンドメッセージ送信
  async send(message: OutboundMessage): Promise<void> {
    const formatted = this.formatMessage(message);

    await this.client.sendMessage(message.chatId, {
      text: formatted.text,
      attachments: formatted.attachments,
      replyTo: message.replyTo,
    });
  }

  // メッセージフォーマット
  private formatMessage(message: OutboundMessage): FormattedMessage {
    // チャネル固有のフォーマット変換
    return {
      text: this.convertMarkdown(message.content),
      attachments: message.attachments?.map(this.convertAttachment),
    };
  }
}
```

## フックの実装

### 利用可能なフック

```typescript
interface PluginHooks {
  // メッセージフック
  preMessage?(msg: InboundMessage): Promise<InboundMessage | null>;
  postMessage?(msg: OutboundMessage): Promise<OutboundMessage>;

  // ツールフック
  preToolCall?(call: ToolCall): Promise<ToolCall | null>;
  postToolCall?(result: ToolResult): Promise<ToolResult>;

  // セッションフック
  onSessionCreate?(session: Session): Promise<void>;
  onSessionEnd?(session: Session): Promise<void>;

  // エージェントフック
  onAgentStart?(ctx: AgentContext): Promise<void>;
  onAgentComplete?(ctx: AgentContext, result: AgentResult): Promise<void>;
}
```

### フック実装例

```typescript
// src/hooks/content-filter.ts
export const contentFilterHooks: PluginHooks = {
  // 入力メッセージのフィルタリング
  async preMessage(msg: InboundMessage): Promise<InboundMessage | null> {
    // スパム検出
    if (isSpam(msg.content)) {
      console.log("Spam detected, dropping message");
      return null; // メッセージをドロップ
    }

    // コンテンツのサニタイズ
    return {
      ...msg,
      content: sanitize(msg.content),
    };
  },

  // 出力メッセージの加工
  async postMessage(msg: OutboundMessage): Promise<OutboundMessage> {
    // 署名の追加
    return {
      ...msg,
      content: `${msg.content}\n\n---\n_Powered by Clawdbot_`,
    };
  },

  // ツール呼び出しの監査
  async preToolCall(call: ToolCall): Promise<ToolCall | null> {
    // 危険なコマンドの検出
    if (call.name === "bash_exec" && isDangerous(call.params.command)) {
      console.log("Dangerous command blocked:", call.params.command);
      return null;
    }
    return call;
  },
};
```

## プラグインの配布

### npm への公開

```bash
# ビルド
pnpm build

# 公開
npm publish --access public
```

### ローカルインストール

```bash
# パスから直接
clawdbot plugins install ./path/to/my-plugin

# npm から
clawdbot plugins install @clawdbot/my-plugin
```

### 設定での有効化

```json
{
  "plugins": {
    "entries": [
      {
        "id": "@clawdbot/my-plugin",
        "enabled": true,
        "options": {
          "apiKey": "xxx",
          "setting1": "value1"
        }
      }
    ]
  }
}
```

## ベストプラクティス

### 1. エラーハンドリング

```typescript
async execute(params: MyToolParams, ctx: ToolContext): Promise<ToolResult> {
  try {
    const result = await riskyOperation();
    return { success: true, data: result };
  } catch (error) {
    ctx.log.error("Operation failed:", error);
    return {
      success: false,
      error: error instanceof Error ? error.message : "Unknown error",
      details: process.env.NODE_ENV === "development" ? error : undefined,
    };
  }
}
```

### 2. 設定のバリデーション

```typescript
import { z } from "zod";

const configSchema = z.object({
  apiKey: z.string().min(1),
  endpoint: z.string().url().optional(),
  timeout: z.number().positive().default(30000),
});

export default function createPlugin(options?: unknown): Plugin {
  const config = configSchema.parse(options);
  // ...
}
```

### 3. リソースのクリーンアップ

```typescript
export default function createPlugin(): Plugin {
  let interval: NodeJS.Timer | null = null;

  return {
    id: "my-plugin",

    async onLoad(ctx) {
      interval = setInterval(() => {
        // 定期処理
      }, 60000);
    },

    async onUnload() {
      if (interval) {
        clearInterval(interval);
        interval = null;
      }
    },
  };
}
```

### 4. 型安全性

```typescript
// 厳密な型定義
interface MyToolParams {
  query: string;
  options?: {
    limit: number;
    offset: number;
  };
}

// Zod による実行時バリデーション
const paramsSchema = z.object({
  query: z.string(),
  options: z.object({
    limit: z.number().int().positive().max(100),
    offset: z.number().int().nonnegative(),
  }).optional(),
});
```

## デバッグ

### ログ出力

```typescript
// PluginContext のログを使用
ctx.log.debug("Debug message");
ctx.log.info("Info message");
ctx.log.warn("Warning message");
ctx.log.error("Error message", error);
```

### 開発モード

```bash
# 詳細ログ有効化
CLAWDBOT_DEBUG=1 clawdbot gateway run

# プラグイン開発モード
clawdbot plugins dev ./my-plugin
```

## テスト

### ユニットテスト

```typescript
// src/tools/my-tool.test.ts
import { describe, it, expect, vi } from "vitest";
import { myTool } from "./my-tool.js";

describe("myTool", () => {
  it("should return results for valid query", async () => {
    const mockCtx = {
      config: { get: vi.fn().mockReturnValue("test-key") },
      log: { info: vi.fn() },
    };

    const result = await myTool.execute(
      { query: "test" },
      mockCtx as any
    );

    expect(result.success).toBe(true);
    expect(result.data).toBeDefined();
  });

  it("should handle errors gracefully", async () => {
    const mockCtx = {
      config: { get: vi.fn().mockReturnValue(null) },
      log: { error: vi.fn() },
    };

    const result = await myTool.execute(
      { query: "" },
      mockCtx as any
    );

    expect(result.success).toBe(false);
    expect(result.error).toBeDefined();
  });
});
```

### 統合テスト

```typescript
// test/integration.test.ts
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import { createTestGateway } from "clawdbot/test-utils";

describe("MyPlugin Integration", () => {
  let gateway: TestGateway;

  beforeAll(async () => {
    gateway = await createTestGateway({
      plugins: ["./dist/index.js"],
    });
  });

  afterAll(async () => {
    await gateway.close();
  });

  it("should register custom RPC method", async () => {
    const result = await gateway.call("myPlugin.doSomething", {
      action: "status",
    });

    expect(result.status).toBe("ok");
  });
});
```

## 参考リンク

- [Clawdbot ドキュメント](https://docs.clawd.bot)
- [プラグイン API リファレンス](https://docs.clawd.bot/plugins/api)
- [公式プラグイン例](https://github.com/clawdbot/clawdbot/tree/main/extensions)
