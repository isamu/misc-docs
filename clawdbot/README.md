# Clawdbot 技術ドキュメント

このディレクトリには、Clawdbot リポジトリの包括的な技術ドキュメントが含まれています。

## ドキュメント一覧

| ファイル | 内容 | 対象読者 |
|---------|------|---------|
| [01-overview.md](./01-overview.md) | プロジェクト概要、ハイレベルアーキテクチャ、技術スタック | 全般 |
| [02-architecture.md](./02-architecture.md) | システム設計、ディレクトリ構造、メッセージフロー、設定システム | 開発者 |
| [03-implementation.md](./03-implementation.md) | コードレベルの実装詳細、主要クラス、内部API | 開発者 |
| [04-features.md](./04-features.md) | 全機能の詳細説明、CLI コマンド、設定オプション | ユーザー/開発者 |
| [05-extension-guide.md](./05-extension-guide.md) | プラグイン開発ガイド、API リファレンス、ベストプラクティス | プラグイン開発者 |

## クイックナビゲーション

### Clawdbot とは？
→ [01-overview.md](./01-overview.md)

### システム構成を理解したい
→ [02-architecture.md](./02-architecture.md)

### コードを読み解きたい
→ [03-implementation.md](./03-implementation.md)

### 機能を一覧したい
→ [04-features.md](./04-features.md)

### プラグインを作りたい
→ [05-extension-guide.md](./05-extension-guide.md)

## プロジェクト概要

**Clawdbot** は、複数のメッセージングプラットフォームを統合するパーソナル AI アシスタントプラットフォームです。

### 主要な特徴

- **10+ チャネル対応**: WhatsApp, Telegram, Discord, Slack, Signal, iMessage, LINE, MS Teams, Matrix 等
- **マルチ LLM プロバイダー**: Anthropic Claude, OpenAI GPT, Google Gemini, GitHub Copilot 等
- **50+ 組み込みツール**: Bash 実行、ブラウザ制御、Canvas、ファイル操作等
- **54 バンドルスキル**: コーディング、ライティング、リサーチ等
- **拡張可能**: TypeScript プラグインシステム
- **マルチプラットフォーム**: macOS, iOS, Android, Linux, Docker

### 技術スタック

| カテゴリ | 技術 |
|---------|------|
| 言語 | TypeScript (ESM) |
| ランタイム | Node.js 22+, Bun (開発) |
| ビルド | tsc |
| パッケージ管理 | pnpm |
| テスト | Vitest |
| リンティング | Oxlint |

### ディレクトリ構造（簡略）

```
clawdbot/
├── src/              # TypeScript ソース
│   ├── cli/          # CLI 実装
│   ├── gateway/      # Gateway サーバー
│   ├── agents/       # Agent ランタイム
│   ├── channels/     # チャネルアダプター
│   ├── plugins/      # プラグインシステム
│   └── media/        # メディアパイプライン
├── extensions/       # 公式プラグイン
├── apps/             # ネイティブアプリ (macOS/iOS/Android)
├── skills/           # バンドルスキル
└── docs/             # 公式ドキュメント (Mintlify)
```

## 関連リンク

- **公式ドキュメント**: https://docs.clawd.bot
- **GitHub**: https://github.com/clawdbot/clawdbot
- **npm**: https://www.npmjs.com/package/clawdbot

## ドキュメント更新履歴

| 日付 | 変更内容 |
|------|---------|
| 2025-01-27 | 初版作成 |
