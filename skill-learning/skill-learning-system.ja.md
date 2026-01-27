# 自己進化型スキルシステム

## 概要

プラグインが基本スキルを定義し、サーバーが使用データから学習・拡張し、改善をPRとしてプラグインリポジトリに自動還元する分散学習システム。

```
プラグイン(基本) → サーバー(学習・拡張) → PRでプラグインに還元 → グローバル進化
```

## コンセプト

### スキルとは？

**スキル**とは、ツール実行前にユーザーを対話的にガイドするフローです。

```
ユーザー: 「LPを作りたい」
     ↓
スキル: 質問して、選択肢を提示
     ↓
ツール: 収集した情報で実行
     ↓
結果: 高品質なアウトプット
```

### 学習ループ

```
┌─────────────────────────────────────────────────────────────────┐
│                    グローバルスキル進化                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐             │
│   │ Plugin   │      │ Plugin   │      │ Plugin   │             │
│   │ GitHub   │      │ GitHub   │      │ GitHub   │             │
│   └────┬─────┘      └────┬─────┘      └────┬─────┘             │
│        │                 │                 │                    │
│        │ 基本スキル       │                 │                    │
│        ▼                 ▼                 ▼                    │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐             │
│   │ Server A │      │ Server B │      │ Server C │             │
│   │ (日本)   │      │ (US)     │      │ (EU)     │             │
│   └────┬─────┘      └────┬─────┘      └────┬─────┘             │
│        │                 │                 │                    │
│        │ 使用データ (匿名化、オプトイン)    │                    │
│        ▼                 ▼                 ▼                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              Central Learning Hub                       │   │
│   │  - 使用パターンを集約                                    │   │
│   │  - 効果的なフローを分析                                  │   │
│   │  - スキル拡張を生成                                      │   │
│   │  - プラグインリポジトリにPR作成                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│        │                                                        │
│        │ 自動生成PR                                             │
│        ▼                                                        │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐             │
│   │ Plugin   │      │ Plugin   │      │ Plugin   │             │
│   │ GitHub   │ ←────│ GitHub   │←─────│ GitHub   │             │
│   └──────────┘      └──────────┘      └──────────┘             │
│                                                                 │
│   サイクルは続く... スキルはグローバルに改善される               │
└─────────────────────────────────────────────────────────────────┘
```

## アーキテクチャ

### レイヤー1: プラグイン（基本スキル）

プラグインはツールと一緒に最小限のスキル定義を提供します。

```
GUIChatPluginHTML/
├── src/
│   ├── core/
│   │   ├── definition.ts    # ツール定義
│   │   ├── plugin.ts        # 実行関数
│   │   └── skills.ts        # ★ スキル定義
│   └── vue/
│       ├── View.vue
│       └── Preview.vue
└── skills/
    └── lp-builder.yaml      # YAML形式のスキル
```

#### スキル定義（プラグイン側）

```typescript
// src/core/skills.ts
import type { SkillDefinition } from "gui-chat-protocol";

export const skills: SkillDefinition[] = [
  {
    name: "lp-builder",
    version: "1.0.0",
    description: "対話型LPページビルダー",
    trigger: ["LP", "ランディングページ", "landing page"],

    objective: `
      高コンバージョンのランディングページを作成する。
      ターゲットユーザーを理解し、最適化されたHTMLを生成する。
    `,

    tools: ["html", "exa", "generateImage"],

    procedures: [
      {
        phase: "gather",
        steps: [
          {
            type: "question",
            id: "topic",
            question: "何のLPですか？",
            required: true,
          },
          {
            type: "question",
            id: "mode",
            question: "進め方は？",
            options: [
              { label: "すぐ作る", value: "quick", skipTo: "phase:generate" },
              { label: "相談しながら", value: "guided" },
            ],
          },
          {
            type: "action",
            tool: "exa",
            purpose: "競合を調査",
            args: { query: "{{topic}} ランディングページ 事例" },
            storeAs: "research",
          },
        ],
      },
      {
        phase: "generate",
        steps: [
          {
            type: "action",
            tool: "html",
            purpose: "LPを生成",
            args: {
              prompt: "{{topic}}のLPを作成。調査結果: {{research}}",
            },
            storeAs: "result",
          },
        ],
      },
    ],

    output: {
      primary: "result",
      artifacts: ["research"],
    },
  },
];
```

#### YAML形式（代替）

```yaml
# skills/lp-builder.yaml
name: lp-builder
version: 1.0.0
description: 対話型LPページビルダー
trigger:
  - LP
  - ランディングページ
  - landing page

objective: |
  高コンバージョンのランディングページを作成する。
  ターゲットユーザーを理解し、最適化されたHTMLを生成する。

tools:
  - html
  - exa
  - generateImage

procedures:
  - phase: gather
    steps:
      - type: question
        id: topic
        question: 何のLPですか？
        required: true

      - type: question
        id: mode
        question: 進め方は？
        options:
          - label: すぐ作る
            value: quick
            skipTo: "phase:generate"
          - label: 相談しながら
            value: guided

      - type: action
        tool: exa
        purpose: 競合を調査
        args:
          query: "{{topic}} ランディングページ 事例"
        storeAs: research

  - phase: generate
    steps:
      - type: action
        tool: html
        purpose: LPを生成
        args:
          prompt: "{{topic}}のLPを作成。調査結果: {{research}}"
        storeAs: result

output:
  primary: result
  artifacts: [research]
```

### レイヤー2: サーバー（学習・拡張）

サーバーはプラグインのスキルを読み込み、使用データから拡張します。

```
server/
├── skills/
│   ├── engine.ts           # スキル実行エンジン
│   ├── loader.ts           # プラグインからスキル読み込み
│   ├── enhancer.ts         # 学習した拡張を適用
│   ├── learner.ts          # 使用分析と学習
│   └── pr-generator.ts     # プラグインへのPR生成
├── data/
│   └── skills/
│       ├── base/           # プラグインから読み込み
│       │   └── html.yaml
│       ├── learned/        # サーバーが生成した拡張
│       │   ├── html-v2.yaml
│       │   └── html-v3.yaml
│       └── usage/          # 使用ログ（ローカル）
│           └── events.jsonl
```

#### 拡張の例

```yaml
# data/skills/learned/html-v2.yaml
name: lp-builder
version: 2.0.0
extends: base/html.yaml
source: learned
confidence: 0.85
basedOn: 1247  # 使用イベント数

enhancements:
  # 使用パターンから発見された新ステップ
  - phase: gather
    after: topic
    add:
      - type: question
        id: industry
        question: 業種は？
        options:
          - label: SaaS
            value: saas
            meta:
              template: tech-modern
              usageCount: 423
          - label: EC
            value: ec
            meta:
              template: product-focus
              usageCount: 312
          - label: 飲食店
            value: restaurant
            meta:
              template: warm-inviting
              usageCount: 187

  # ユーザーの混乱に基づく質問文改善
  - phase: gather
    modify: mode
    question: "作成モードを選択："
    reason: "元の文言で23%のユーザーが混乱"

  # ユーザーリクエストに基づく新オプション
  - phase: gather
    extend: mode.options
    add:
      - label: テンプレートから
        value: template
        meta:
          requestedBy: 89  # ユーザー数

  # 新アクション発見: 競合分析が結果を改善
  - phase: gather
    after: research
    add:
      - type: action
        tool: browse
        purpose: トップ競合を詳細分析
        args:
          url: "{{research.results[0].url}}"
        storeAs: competitorDetail
        meta:
          addedReason: "競合分析を見たユーザーは満足度が40%向上"
          usageCount: 892
```

### レイヤー3: Central Learning Hub

複数インスタンスから匿名化データを集約するオプションのクラウドサービス。

```typescript
// Central Hub API
interface LearningHubAPI {
  // 匿名化した使用データを送信
  submitUsage(data: AnonymizedUsageData[]): Promise<void>;

  // 最新のスキル拡張を取得
  getEnhancements(skillName: string): Promise<Enhancement[]>;

  // グローバルスキルランキングを取得
  getSkillRankings(): Promise<SkillRanking[]>;
}
```

## データ構造

### スキル定義

```typescript
interface SkillDefinition {
  name: string;
  version: string;
  description: string;
  trigger: string | string[] | RegExp;

  // ★ 目的: このスキルが達成すること
  objective: string;

  // ★ 使用するツール/MCP
  tools: string[];

  // ★ 手順: 質問とアクション
  procedures: SkillPhase[];

  // ★ 出力定義
  output: SkillOutput;
}

interface SkillPhase {
  phase: string;           // 例: "gather", "strategy", "generate"
  description?: string;
  steps: SkillStep[];
}

interface SkillStep {
  // ステップタイプ
  type: "question" | "action" | "decision" | "condition" | "parallel";

  id?: string;

  // question/decision用
  question?: string;
  questionJa?: string;
  options?: SkillOption[];
  context?: string;          // ユーザーの判断を助けるコンテキスト
  suggestions?: {            // AI生成の提案
    from: string;            // テンプレート参照
  };

  // action用（ツール呼び出し）
  tool?: string;
  purpose?: string;          // なぜこのアクションが必要か
  args?: Record<string, unknown>;
  storeAs?: string;          // 結果を後で使うために保存

  // condition用
  condition?: string;        // 例: "mode === 'guided'"
  then?: SkillStep[];
  else?: SkillStep[];

  // parallel用（並列実行）
  parallel?: SkillStep[];    // 複数アクションを並列実行

  // 制御フロー
  skipTo?: string;           // phase:stepにジャンプ
  required?: boolean;
  validation?: string;
}

interface SkillOption {
  label: string;
  labelJa?: string;
  value: string;
  description?: string;
  skipTo?: string;           // 特定phase/stepにジャンプ
  meta?: Record<string, unknown>;
}

interface SkillOutput {
  primary: string;           // メイン出力の参照
  artifacts?: string[];      // 追加出力
  summary?: string;          // サマリーメッセージのテンプレート
}
```

### 使用イベント

```typescript
interface SkillUsageEvent {
  // 識別子（プライバシーのためハッシュ化）
  instanceId: string;        // ハッシュ化されたサーバーインスタンスID
  sessionId: string;         // ハッシュ化されたセッションID

  // スキル実行
  skillName: string;
  skillVersion: string;
  stepId: string;

  // ユーザーインタラクション
  answer: unknown;           // プライバシーのため編集される場合あり
  timestamp: Date;
  durationMs: number;        // ステップにかかった時間

  // 結果
  completed: boolean;
  abandoned: boolean;
  backtracked: boolean;      // ユーザーが戻った

  // フィードバック
  userSatisfaction?: 1 | 2 | 3 | 4 | 5;
  userFeedback?: string;     // フリーテキスト（オプトインのみ）

  // コンテキスト
  locale: string;
  toolResult?: "success" | "error";
}
```

### 学習された拡張

```typescript
interface LearnedEnhancement {
  id: string;
  skillName: string;
  type: "add_step" | "add_option" | "modify_question" | "reorder" | "remove";

  // 変更内容
  target?: string;           // ステップIDまたはオプションID
  position?: "before" | "after" | "replace";
  content: Partial<SkillStep> | Partial<SkillOption>;

  // 信頼度メトリクス
  confidence: number;        // 0-1
  basedOnEvents: number;     // 使用イベント数
  basedOnInstances: number;  // サーバーインスタンス数

  // 理由
  reason: string;            // 人間が読める説明
  metrics: {
    completionRateBefore: number;
    completionRateAfter: number;
    satisfactionBefore: number;
    satisfactionAfter: number;
  };
}
```

## PR生成

拡張が十分な信頼度に達したら、自動的にPRを作成します。

### PRコンテンツ例

```markdown
## 🤖 自動学習スキル拡張

**スキル**: lp-builder
**バージョン**: 1.0.0 → 2.0.0
**基づくデータ**: 23インスタンスからの1,247使用イベント

### 変更内容

#### 1. 新ステップ: 業種選択
1,247件のLP作成セッションを分析した結果、ユーザーが頻繁に
業種固有のニーズに言及していることがわかりました。業種選択ステップを
追加することで、完了率が34%向上しました。

```yaml
- id: industry
  question: 業種は？
  type: choice
  options:
    - { label: SaaS, value: saas }
    - { label: EC, value: ec }
    - { label: 飲食店, value: restaurant }
```

#### 2. 質問文の改善
元の「進め方は？」では23%のユーザーが混乱しました。より明確な文言に更新。

### メトリクス

| 指標 | 変更前 | 変更後 | 変化 |
|------|--------|--------|------|
| 完了率 | 67% | 89% | +22% |
| ユーザー満足度 | 3.2 | 4.1 | +0.9 |
| 完了までの平均ステップ数 | 4.2 | 3.8 | -0.4 |

### プライバシーに関する注記
この拡張は匿名化・集約された使用データから生成されました。
個人情報や特定のユーザー入力は使用されていません。

---
Generated by MulmoChat Skill Learning System v1.0
```

### PR生成コード

```typescript
// server/skills/pr-generator.ts

export async function generateSkillPR(
  pluginRepo: string,
  enhancement: LearnedEnhancement[]
): Promise<string> {
  const newVersion = bumpVersion(enhancement[0].skillVersion);

  // 拡張されたスキルYAMLを生成
  const skillYaml = applyEnhancements(
    loadBaseSkill(enhancement[0].skillName),
    enhancement
  );

  // PR説明文を生成
  const description = generatePRDescription(enhancement);

  // GitHub APIでPR作成
  const prUrl = await github.createPullRequest({
    owner: parseOwner(pluginRepo),
    repo: parseRepo(pluginRepo),
    base: "main",
    head: `skill-enhancement-${newVersion}`,
    title: `feat(skill): ${enhancement[0].skillName} v${newVersion}を拡張`,
    body: description,
    files: [
      {
        path: `skills/${enhancement[0].skillName}.yaml`,
        content: skillYaml,
      },
    ],
  });

  return prUrl;
}
```

## プライバシーとセキュリティ

### データ収集の原則

1. **オプトインのみ**: データ共有はデフォルトで無効
2. **匿名化**: すべての識別子は送信前にハッシュ化
3. **個人情報なし**: 個人情報は収集しない
4. **編集**: フリーテキスト入力は編集または除外
5. **集約**: 学習には集約されたパターンのみ使用

### 設定

```yaml
# server/config/learning.yaml
learning:
  enabled: true

  # ローカル学習（常に安全）
  local:
    enabled: true
    storagePath: ./data/skills/usage
    retentionDays: 90

  # Central Hub共有（オプトイン）
  centralHub:
    enabled: false           # オプトイン
    endpoint: https://hub.mulmochat.dev/api

    # 共有する内容
    share:
      stepCompletions: true  # どのステップが完了したか
      optionChoices: true    # どのオプションが選択されたか
      timings: true          # 各ステップにかかった時間
      satisfaction: true     # ユーザー満足度評価
      freeText: false        # フリーテキストはデフォルトで共有しない

    # 共有しない内容
    exclude:
      fields:
        - topic              # 機密情報を含む可能性あり
        - custom_*           # カスタムフィールド
      patterns:
        - "*email*"
        - "*password*"
        - "*secret*"

  # PR生成
  pullRequests:
    enabled: true
    minConfidence: 0.8       # PR作成の最小信頼度
    minEvents: 100           # 最小使用イベント数
    githubToken: ${GITHUB_TOKEN}
```

## 実装フェーズ

### フェーズ1: ローカルスキル（MVP）
- [ ] `gui-chat-protocol`で`SkillDefinition`型を定義
- [ ] プラグインシステムにスキル読み込みを追加
- [ ] 基本的なスキル実行エンジンを実装
- [ ] スキルUI（選択ボタン、テキスト入力）を追加

### フェーズ2: ローカル学習
- [ ] 使用イベントログを実装
- [ ] ローカル学習アルゴリズムを作成
- [ ] ローカル拡張を生成
- [ ] スキル実行に拡張を適用

### フェーズ3: PR生成
- [ ] PR生成器を実装
- [ ] GitHub連携を追加
- [ ] 拡張レビューUIを作成
- [ ] 手動承認ワークフローを追加

### フェーズ4: Central Hub（将来）
- [ ] Hub APIを設計
- [ ] データ匿名化を実装
- [ ] 集約アルゴリズムを作成
- [ ] Hubインフラを構築
- [ ] オプトインフローを追加

## スキル例

### LPビルダー（htmlプラグイン）- 完全版

```yaml
name: lp-builder
version: 1.0.0
description: 市場調査付き対話型LPページビルダー

# ★ 目的: このスキルが達成すること
objective: |
  高コンバージョンのランディングページを作成する。
  ターゲットユーザーを理解し、競合を分析し、
  魅力的なコンテンツで最適化されたHTMLを生成する。

# ★ オーケストレーションするツール/MCP
tools:
  - html          # HTML生成
  - browse        # 競合分析
  - exa           # 市場調査
  - generateImage # ヒーロー画像生成

trigger:
  - LP
  - landing page
  - ランディングページ

# ★ 手順: マルチフェーズワークフロー
procedures:
  # フェーズ1: 情報収集
  - phase: gather
    description: 基本情報を収集し市場を調査
    steps:
      - type: question
        id: topic
        question: 何のLPですか？
        required: true

      - type: question
        id: mode
        question: 進め方は？
        options:
          - label: すぐ作る
            value: quick
            skipTo: "phase:generate"
          - label: 相談しながら
            value: guided

      # ★ アクション: 調査のためのツール呼び出し
      - type: action
        tool: exa
        purpose: 市場と競合を調査
        args:
          query: "{{topic}} ランディングページ 事例 ベストプラクティス"
        storeAs: marketResearch

      # ★ 並列アクション
      - type: parallel
        parallel:
          - type: action
            tool: browse
            purpose: トップ競合を分析
            args:
              url: "{{marketResearch.results[0].url}}"
            storeAs: competitor1

          - type: action
            tool: browse
            purpose: 2番目の競合を分析
            args:
              url: "{{marketResearch.results[1].url}}"
            storeAs: competitor2

  # フェーズ2: 戦略決定
  - phase: strategy
    description: ポジショニングとアプローチを決定
    steps:
      # ★ 調査結果のコンテキスト付き意思決定
      - type: decision
        id: targetAudience
        question: ターゲット層は？
        context: |
          競合分析の結果:
          {{competitor1.summary}}
          {{competitor2.summary}}
        options:
          - { label: 開発者, value: developers }
          - { label: 経営者, value: executives }
          - { label: 一般ユーザー, value: consumers }

      - type: question
        id: keyMessage
        question: キーメッセージは？
        suggestions:
          from: "{{marketResearch.insights}}"

      - type: question
        id: tone
        question: トーンは？
        options:
          - { label: プロフェッショナル, value: professional }
          - { label: フレンドリー, value: friendly }
          - { label: 大胆, value: bold }
          - { label: ミニマル, value: minimal }

      - type: question
        id: sections
        question: どのセクションを含めますか？
        type: multiChoice
        options:
          - { label: ヒーロー, value: hero, default: true }
          - { label: 機能紹介, value: features, default: true }
          - { label: お客様の声, value: testimonials }
          - { label: 料金, value: pricing }
          - { label: FAQ, value: faq }
          - { label: CTA, value: cta, default: true }

  # フェーズ3: 生成
  - phase: generate
    description: LPアセットとHTMLを作成
    steps:
      # ★ 条件付きアクション
      - type: condition
        condition: "sections.includes('hero')"
        then:
          - type: action
            tool: generateImage
            purpose: ヒーロー画像を作成
            args:
              prompt: |
                {{topic}}ランディングページ用ヒーロー画像。
                ターゲット: {{targetAudience}}
                トーン: {{tone}}
                スタイル: モダン、プロフェッショナル、16:9アスペクト比
            storeAs: heroImage

      # ★ メイン生成
      - type: action
        tool: html
        purpose: ランディングページを生成
        args:
          prompt: |
            {{topic}}の高コンバージョンランディングページを作成。

            ## コンテキスト
            - ターゲット層: {{targetAudience}}
            - キーメッセージ: {{keyMessage}}
            - トーン: {{tone}}

            ## 競合インサイト
            {{competitor1.insights}}
            {{competitor2.insights}}

            ## 含めるセクション
            {{sections}}

            ## アセット
            ヒーロー画像: {{heroImage.url}}

            ## 要件
            - モバイルレスポンシブ
            - 高速ロード
            - 明確なCTA
            - SEO最適化
        storeAs: finalLP

# ★ 出力定義
output:
  primary: finalLP
  artifacts:
    - heroImage
    - marketResearch
    - competitor1
    - competitor2
  summary: |
    {{topic}}のLPを作成しました。ターゲット: {{targetAudience}}。
    {{sections.length}}セクション、{{tone}}トーン。
```

### LPビルダー（シンプル版）

```yaml
name: lp-builder
targetTool: html
objective: ランディングページを素早く作成
tools: [html]

procedures:
  - phase: main
    steps:
      - type: question
        id: topic
        question: 何のLPですか？
      - type: question
        id: mode
        question: 作成モードは？
        options:
          - { label: すぐ作る, value: quick }
          - { label: 相談しながら, value: guided }
      - type: action
        tool: html
        args:
          prompt: "{{topic}}のLPを作成、モード: {{mode}}"
        storeAs: result

output:
  primary: result
```

### ポッドキャスト作成（mulmocastプラグイン）

```yaml
name: podcast-creator
version: 1.0.0
objective: |
  リサーチ、スクリプト生成、音声制作を含む
  魅力的なポッドキャストエピソードを作成する。

tools:
  - mulmocast    # ポッドキャスト生成
  - exa          # トピック調査
  - browse       # ソース収集

procedures:
  - phase: research
    steps:
      - type: question
        id: topic
        question: ポッドキャストのテーマは？

      - type: action
        tool: exa
        purpose: トピックを調査
        args:
          query: "{{topic}} 最新ニュース インサイト"
        storeAs: research

  - phase: planning
    steps:
      - type: decision
        id: style
        question: スタイルは？
        context: "トピックに基づく: {{research.summary}}"
        options:
          - { label: インタビュー, value: interview }
          - { label: ソロ, value: solo }
          - { label: パネル, value: panel }

      - type: question
        id: duration
        question: 目標の長さは？
        options:
          - { label: "5分", value: 5 }
          - { label: "15分", value: 15 }
          - { label: "30分", value: 30 }

  - phase: generate
    steps:
      - type: action
        tool: mulmocast
        purpose: ポッドキャストを生成
        args:
          topic: "{{topic}}"
          style: "{{style}}"
          duration: "{{duration}}"
          research: "{{research}}"
        storeAs: podcast

output:
  primary: podcast
  artifacts: [research]
```

### クイズデザイナー（quizプラグイン）

```yaml
name: quiz-designer
version: 1.0.0
objective: |
  調査に基づいた問題と正確な回答を含む
  教育的なクイズを作成する。

tools:
  - quiz         # クイズ生成
  - exa          # ファクトチェック

procedures:
  - phase: setup
    steps:
      - type: question
        id: subject
        question: クイズのテーマは？

      - type: action
        tool: exa
        purpose: 正確な問題のためにテーマを調査
        args:
          query: "{{subject}} 事実 トリビア"
        storeAs: facts

      - type: question
        id: difficulty
        question: 難易度は？
        options:
          - { label: 簡単, value: easy }
          - { label: 普通, value: medium }
          - { label: 難しい, value: hard }

      - type: question
        id: count
        question: 問題数は？
        default: 5

  - phase: generate
    steps:
      - type: action
        tool: quiz
        purpose: クイズを生成
        args:
          subject: "{{subject}}"
          difficulty: "{{difficulty}}"
          count: "{{count}}"
          facts: "{{facts}}"
        storeAs: quiz

output:
  primary: quiz
```

### 旅行プランナー（mapプラグイン）

```yaml
name: travel-planner
version: 1.0.0
objective: |
  ローカルおすすめとルート計画を含む
  パーソナライズされた旅行プランを作成する。

tools:
  - map          # ルートと場所
  - exa          # 旅行リサーチ
  - browse       # 現地情報

procedures:
  - phase: gather
    steps:
      - type: question
        id: destination
        question: どこに行きたいですか？

      - type: parallel
        parallel:
          - type: action
            tool: exa
            purpose: 目的地を調査
            args:
              query: "{{destination}} 旅行ガイド 観光スポット"
            storeAs: travelGuide

          - type: action
            tool: exa
            purpose: 現地のヒントを探す
            args:
              query: "{{destination}} 穴場 地元のおすすめ"
            storeAs: localTips

  - phase: preferences
    steps:
      - type: question
        id: duration
        question: 何日間？

      - type: question
        id: interests
        question: 興味があるのは？
        type: multiChoice
        context: |
          {{destination}}で人気:
          {{travelGuide.highlights}}
        options:
          - { label: グルメ, value: food }
          - { label: 歴史, value: history }
          - { label: 自然, value: nature }
          - { label: ショッピング, value: shopping }

  - phase: generate
    steps:
      - type: action
        tool: map
        purpose: ルート付き旅程を作成
        args:
          destination: "{{destination}}"
          days: "{{duration}}"
          interests: "{{interests}}"
          recommendations: "{{travelGuide}}"
          localTips: "{{localTips}}"
        storeAs: itinerary

output:
  primary: itinerary
  artifacts: [travelGuide, localTips]
  summary: "{{destination}}への{{duration}}日間の旅程を作成しました！"
```

## まとめ

自己進化型スキルシステムは好循環を生み出します：

1. **プラグイン**が基本スキルを提供
2. **サーバー**が使用から学習
3. **拡張**が自動生成
4. **PR**がプラグインに還元
5. **グローバルエコシステム**が継続的に改善

これにより、MulmoChatはユーザーのプライバシーを尊重し、プラグインメンテナーの管理を維持しながら、世界中のすべてのインタラクションから学習し、時間とともによりスマートになります。
