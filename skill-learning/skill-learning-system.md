# Self-Evolving Skill System

## Overview

A distributed learning system where plugins define base skills, servers enhance them through usage data, and improvements are automatically contributed back to plugin repositories via PRs.

```
Plugin (Base) → Server (Learn & Enhance) → PR back to Plugin → Global Evolution
```

## Concept

### What is a Skill?

A **Skill** is an interactive dialogue flow that guides users through complex tasks before executing a tool.

```
User: "I want to create an LP"
     ↓
Skill: Asks questions, presents choices
     ↓
Tool: Executes with collected context
     ↓
Result: High-quality output
```

### The Learning Loop

```
┌─────────────────────────────────────────────────────────────────┐
│                    GLOBAL SKILL EVOLUTION                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐             │
│   │ Plugin   │      │ Plugin   │      │ Plugin   │             │
│   │ GitHub   │      │ GitHub   │      │ GitHub   │             │
│   └────┬─────┘      └────┬─────┘      └────┬─────┘             │
│        │                 │                 │                    │
│        │ base skills     │                 │                    │
│        ▼                 ▼                 ▼                    │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐             │
│   │ Server A │      │ Server B │      │ Server C │             │
│   │ (Japan)  │      │ (US)     │      │ (EU)     │             │
│   └────┬─────┘      └────┬─────┘      └────┬─────┘             │
│        │                 │                 │                    │
│        │ usage data (anonymized, opt-in)   │                    │
│        ▼                 ▼                 ▼                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              Central Learning Hub                       │   │
│   │  - Aggregate usage patterns                             │   │
│   │  - Analyze effective flows                              │   │
│   │  - Generate skill enhancements                          │   │
│   │  - Create PRs to plugin repositories                    │   │
│   └─────────────────────────────────────────────────────────┘   │
│        │                                                        │
│        │ Auto-generated PRs                                     │
│        ▼                                                        │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐             │
│   │ Plugin   │      │ Plugin   │      │ Plugin   │             │
│   │ GitHub   │ ←────│ GitHub   │←─────│ GitHub   │             │
│   └──────────┘      └──────────┘      └──────────┘             │
│                                                                 │
│   The cycle continues... Skills get better globally            │
└─────────────────────────────────────────────────────────────────┘
```

## Architecture

### Layer 1: Plugin (Base Skills)

Plugins define minimal skill definitions alongside their tools.

```
GUIChatPluginHTML/
├── src/
│   ├── core/
│   │   ├── definition.ts    # Tool definition
│   │   ├── plugin.ts        # Execute function
│   │   └── skills.ts        # ★ Skill definitions
│   └── vue/
│       ├── View.vue
│       └── Preview.vue
└── skills/
    └── lp-builder.yaml      # Skill in YAML format
```

#### Skill Definition (Plugin-side)

```typescript
// src/core/skills.ts
import type { SkillDefinition } from "gui-chat-protocol";

export const skills: SkillDefinition[] = [
  {
    name: "lp-builder",
    version: "1.0.0",
    description: "Interactive LP page builder",
    trigger: ["LP", "ランディングページ", "landing page"],

    objective: `
      Create a high-converting landing page.
      Understand target users and generate optimized HTML.
    `,

    tools: ["html", "exa", "generateImage"],

    procedures: [
      {
        phase: "gather",
        steps: [
          {
            type: "question",
            id: "topic",
            question: "What is the LP for?",
            required: true,
          },
          {
            type: "question",
            id: "mode",
            question: "How would you like to proceed?",
            options: [
              { label: "Quick", value: "quick", skipTo: "phase:generate" },
              { label: "Guided", value: "guided" },
            ],
          },
          {
            type: "action",
            tool: "exa",
            purpose: "Research competitors",
            args: { query: "{{topic}} landing page examples" },
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
            purpose: "Generate the LP",
            args: {
              prompt: "Create LP for {{topic}}. Research: {{research}}",
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

#### YAML Format (Alternative)

```yaml
# skills/lp-builder.yaml
name: lp-builder
version: 1.0.0
description: Interactive LP page builder
trigger:
  - LP
  - ランディングページ
  - landing page

objective: |
  Create a high-converting landing page.
  Understand target users and generate optimized HTML.

tools:
  - html
  - exa
  - generateImage

procedures:
  - phase: gather
    steps:
      - type: question
        id: topic
        question: What is the LP for?
        questionJa: 何のLPですか？
        required: true

      - type: question
        id: mode
        question: How would you like to proceed?
        questionJa: 進め方は？
        options:
          - label: Quick
            labelJa: すぐ作る
            value: quick
            skipTo: "phase:generate"
          - label: Guided
            labelJa: 相談しながら
            value: guided

      - type: action
        tool: exa
        purpose: Research competitors
        args:
          query: "{{topic}} landing page examples"
        storeAs: research

  - phase: generate
    steps:
      - type: action
        tool: html
        purpose: Generate the LP
        args:
          prompt: "Create LP for {{topic}}. Research: {{research}}"
        storeAs: result

output:
  primary: result
  artifacts: [research]
```

### Layer 2: Server (Learning & Enhancement)

The server loads plugin skills and enhances them through usage data.

```
server/
├── skills/
│   ├── engine.ts           # Skill execution engine
│   ├── loader.ts           # Load skills from plugins
│   ├── enhancer.ts         # Apply learned enhancements
│   ├── learner.ts          # Analyze usage and learn
│   └── pr-generator.ts     # Generate PRs for plugins
├── data/
│   └── skills/
│       ├── base/           # Loaded from plugins
│       │   └── html.yaml
│       ├── learned/        # Server-generated enhancements
│       │   ├── html-v2.yaml
│       │   └── html-v3.yaml
│       └── usage/          # Usage logs (local)
│           └── events.jsonl
```

#### Enhancement Example

```yaml
# data/skills/learned/html-v2.yaml
name: lp-builder
version: 2.0.0
extends: base/html.yaml
source: learned
confidence: 0.85
basedOn: 1247  # usage events

enhancements:
  # New step discovered from usage patterns
  - phase: gather
    after: topic
    add:
      - type: question
        id: industry
        question: What industry?
        questionJa: 業種は？
        options:
          - label: SaaS
            value: saas
            meta:
              template: tech-modern
              usageCount: 423
          - label: E-commerce
            labelJa: EC
            value: ec
            meta:
              template: product-focus
              usageCount: 312
          - label: Restaurant
            labelJa: 飲食店
            value: restaurant
            meta:
              template: warm-inviting
              usageCount: 187

  # Improved question wording based on user confusion
  - phase: gather
    modify: mode
    question: "Choose creation mode:"
    questionJa: "作成モードを選択："
    reason: "Original wording caused 23% user confusion"

  # New option added based on user requests
  - phase: gather
    extend: mode.options
    add:
      - label: "Template-based"
        labelJa: "テンプレートから"
        value: template
        meta:
          requestedBy: 89  # users

  # New action discovered: competitor analysis improves results
  - phase: gather
    after: research
    add:
      - type: action
        tool: browse
        purpose: Analyze top competitor in detail
        args:
          url: "{{research.results[0].url}}"
        storeAs: competitorDetail
        meta:
          addedReason: "Users who saw competitor analysis had 40% higher satisfaction"
          usageCount: 892
```

### Layer 3: Central Learning Hub

Optional cloud service that aggregates anonymized data from multiple instances.

```typescript
// Central Hub API
interface LearningHubAPI {
  // Submit anonymized usage data
  submitUsage(data: AnonymizedUsageData[]): Promise<void>;

  // Get latest skill enhancements
  getEnhancements(skillName: string): Promise<Enhancement[]>;

  // Get global skill rankings
  getSkillRankings(): Promise<SkillRanking[]>;
}
```

## Data Structures

### Skill Definition

```typescript
interface SkillDefinition {
  name: string;
  version: string;
  description: string;
  trigger: string | string[] | RegExp;

  // ★ Objective: What this skill aims to achieve
  objective: string;

  // ★ Tools/MCPs this skill uses
  tools: string[];

  // ★ Procedures: Questions AND Actions
  procedures: SkillPhase[];

  // ★ Output definition
  output: SkillOutput;
}

interface SkillPhase {
  phase: string;           // e.g., "gather", "strategy", "generate"
  description?: string;
  steps: SkillStep[];
}

interface SkillStep {
  // Step type
  type: "question" | "action" | "decision" | "condition" | "parallel";

  id?: string;

  // For questions/decisions
  question?: string;
  questionJa?: string;
  options?: SkillOption[];
  context?: string;          // Show context to help user decide
  suggestions?: {            // AI-generated suggestions
    from: string;            // Template reference
  };

  // For actions (tool calls)
  tool?: string;
  purpose?: string;          // Why this action is needed
  args?: Record<string, unknown>;
  storeAs?: string;          // Store result for later use

  // For conditions
  condition?: string;        // e.g., "mode === 'guided'"
  then?: SkillStep[];
  else?: SkillStep[];

  // For parallel execution
  parallel?: SkillStep[];    // Run multiple actions in parallel

  // Control flow
  skipTo?: string;           // Jump to phase:step
  required?: boolean;
  validation?: string;
}

interface SkillOption {
  label: string;
  labelJa?: string;
  value: string;
  description?: string;
  skipTo?: string;           // Jump to specific phase/step
  meta?: Record<string, unknown>;
}

interface SkillOutput {
  primary: string;           // Main output reference
  artifacts?: string[];      // Additional outputs
  summary?: string;          // Template for summary message
}
```

### Usage Event

```typescript
interface SkillUsageEvent {
  // Identification (hashed for privacy)
  instanceId: string;        // Hashed server instance ID
  sessionId: string;         // Hashed session ID

  // Skill execution
  skillName: string;
  skillVersion: string;
  stepId: string;

  // User interaction
  answer: unknown;           // May be redacted for privacy
  timestamp: Date;
  durationMs: number;        // Time spent on step

  // Outcome
  completed: boolean;
  abandoned: boolean;
  backtracked: boolean;      // User went back

  // Feedback
  userSatisfaction?: 1 | 2 | 3 | 4 | 5;
  userFeedback?: string;     // Free text (opt-in only)

  // Context
  locale: string;
  toolResult?: "success" | "error";
}
```

### Learned Enhancement

```typescript
interface LearnedEnhancement {
  id: string;
  skillName: string;
  type: "add_step" | "add_option" | "modify_question" | "reorder" | "remove";

  // What to change
  target?: string;           // Step ID or option ID
  position?: "before" | "after" | "replace";
  content: Partial<SkillStep> | Partial<SkillOption>;

  // Confidence metrics
  confidence: number;        // 0-1
  basedOnEvents: number;     // Number of usage events
  basedOnInstances: number;  // Number of server instances

  // Reasoning
  reason: string;            // Human-readable explanation
  metrics: {
    completionRateBefore: number;
    completionRateAfter: number;
    satisfactionBefore: number;
    satisfactionAfter: number;
  };
}
```

## PR Generation

When enhancements reach sufficient confidence, automatically create PRs.

### PR Content Example

```markdown
## 🤖 Auto-learned Skill Enhancement

**Skill**: lp-builder
**Version**: 1.0.0 → 2.0.0
**Based on**: 1,247 usage events from 23 instances

### Changes

#### 1. New Step: Industry Selection
After analyzing 1,247 LP creation sessions, we found that users frequently
mention industry-specific needs. Adding an industry selection step improved
completion rate by 34%.

```yaml
- id: industry
  question: What industry?
  type: choice
  options:
    - { label: SaaS, value: saas }
    - { label: E-commerce, value: ec }
    - { label: Restaurant, value: restaurant }
```

#### 2. Improved Question Wording
The original "進め方は？" caused 23% user confusion. Updated to clearer wording.

### Metrics

| Metric | Before | After | Change |
|--------|--------|-------|--------|
| Completion Rate | 67% | 89% | +22% |
| User Satisfaction | 3.2 | 4.1 | +0.9 |
| Avg. Steps to Complete | 4.2 | 3.8 | -0.4 |

### Privacy Note
This enhancement was generated from anonymized, aggregated usage data.
No personal information or specific user inputs were used.

---
Generated by MulmoChat Skill Learning System v1.0
```

### PR Generation Code

```typescript
// server/skills/pr-generator.ts

export async function generateSkillPR(
  pluginRepo: string,
  enhancement: LearnedEnhancement[]
): Promise<string> {
  const newVersion = bumpVersion(enhancement[0].skillVersion);

  // Generate enhanced skill YAML
  const skillYaml = applyEnhancements(
    loadBaseSkill(enhancement[0].skillName),
    enhancement
  );

  // Generate PR description
  const description = generatePRDescription(enhancement);

  // Create PR via GitHub API
  const prUrl = await github.createPullRequest({
    owner: parseOwner(pluginRepo),
    repo: parseRepo(pluginRepo),
    base: "main",
    head: `skill-enhancement-${newVersion}`,
    title: `feat(skill): enhance ${enhancement[0].skillName} v${newVersion}`,
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

## Privacy & Security

### Data Collection Principles

1. **Opt-in Only**: Data sharing is disabled by default
2. **Anonymization**: All identifiers are hashed before transmission
3. **No PII**: Personal information is never collected
4. **Redaction**: Free-text inputs are redacted or excluded
5. **Aggregation**: Only aggregated patterns are used for learning

### Configuration

```yaml
# server/config/learning.yaml
learning:
  enabled: true

  # Local learning (always safe)
  local:
    enabled: true
    storagePath: ./data/skills/usage
    retentionDays: 90

  # Central hub sharing (opt-in)
  centralHub:
    enabled: false           # Opt-in
    endpoint: https://hub.mulmochat.dev/api

    # What to share
    share:
      stepCompletions: true  # Which steps were completed
      optionChoices: true    # Which options were selected
      timings: true          # How long each step took
      satisfaction: true     # User satisfaction ratings
      freeText: false        # Never share free text by default

    # What NOT to share
    exclude:
      fields:
        - topic              # May contain sensitive info
        - custom_*           # Any custom fields
      patterns:
        - "*email*"
        - "*password*"
        - "*secret*"

  # PR generation
  pullRequests:
    enabled: true
    minConfidence: 0.8       # Minimum confidence to create PR
    minEvents: 100           # Minimum usage events
    githubToken: ${GITHUB_TOKEN}
```

## Implementation Phases

### Phase 1: Local Skills (MVP)
- [ ] Define `SkillDefinition` types in `gui-chat-protocol`
- [ ] Add skill loading to plugin system
- [ ] Implement basic skill execution engine
- [ ] Add skill UI (choice buttons, text input)

### Phase 2: Local Learning
- [ ] Implement usage event logging
- [ ] Create local learning algorithm
- [ ] Generate local enhancements
- [ ] Apply enhancements to skill execution

### Phase 3: PR Generation
- [ ] Implement PR generator
- [ ] Add GitHub integration
- [ ] Create enhancement review UI
- [ ] Add manual approval workflow

### Phase 4: Central Hub (Future)
- [ ] Design hub API
- [ ] Implement data anonymization
- [ ] Create aggregation algorithms
- [ ] Build hub infrastructure
- [ ] Add opt-in flow

## Example Skills

### LP Builder (html plugin) - Full Example

```yaml
name: lp-builder
version: 1.0.0
description: Interactive LP page builder with market research

# ★ Objective: What this skill achieves
objective: |
  Create a high-converting landing page.
  Understand target users, analyze competitors,
  and generate optimized HTML with compelling content.

# ★ Tools/MCPs to orchestrate
tools:
  - html          # HTML generation
  - browse        # Competitor analysis
  - exa           # Market research
  - generateImage # Hero image generation

trigger:
  - LP
  - landing page
  - ランディングページ

# ★ Procedures: Multi-phase workflow
procedures:
  # Phase 1: Information Gathering
  - phase: gather
    description: Collect basic info and research market
    steps:
      - type: question
        id: topic
        question: What is the LP for?
        questionJa: 何のLPですか？
        required: true

      - type: question
        id: mode
        question: How would you like to proceed?
        questionJa: 進め方は？
        options:
          - label: Quick
            labelJa: すぐ作る
            value: quick
            skipTo: "phase:generate"
          - label: Guided
            labelJa: 相談しながら
            value: guided

      # ★ Action: Tool call for research
      - type: action
        tool: exa
        purpose: Research market and competitors
        args:
          query: "{{topic}} landing page examples best practices"
        storeAs: marketResearch

      # ★ Parallel actions
      - type: parallel
        parallel:
          - type: action
            tool: browse
            purpose: Analyze top competitor
            args:
              url: "{{marketResearch.results[0].url}}"
            storeAs: competitor1

          - type: action
            tool: browse
            purpose: Analyze second competitor
            args:
              url: "{{marketResearch.results[1].url}}"
            storeAs: competitor2

  # Phase 2: Strategy Decision
  - phase: strategy
    description: Decide positioning and approach
    steps:
      # ★ Decision with context from research
      - type: decision
        id: targetAudience
        question: Who is the target audience?
        questionJa: ターゲット層は？
        context: |
          Based on competitor analysis:
          {{competitor1.summary}}
          {{competitor2.summary}}
        options:
          - { label: Developers, labelJa: 開発者, value: developers }
          - { label: Executives, labelJa: 経営者, value: executives }
          - { label: Consumers, labelJa: 一般ユーザー, value: consumers }

      - type: question
        id: keyMessage
        question: What's the key message?
        questionJa: キーメッセージは？
        suggestions:
          from: "{{marketResearch.insights}}"

      - type: question
        id: tone
        question: What tone should it have?
        questionJa: トーンは？
        options:
          - { label: Professional, labelJa: プロフェッショナル, value: professional }
          - { label: Friendly, labelJa: フレンドリー, value: friendly }
          - { label: Bold, labelJa: 大胆, value: bold }
          - { label: Minimal, labelJa: ミニマル, value: minimal }

      - type: question
        id: sections
        question: Which sections to include?
        questionJa: どのセクションを含めますか？
        type: multiChoice
        options:
          - { label: Hero, value: hero, default: true }
          - { label: Features, value: features, default: true }
          - { label: Testimonials, value: testimonials }
          - { label: Pricing, value: pricing }
          - { label: FAQ, value: faq }
          - { label: CTA, value: cta, default: true }

  # Phase 3: Generation
  - phase: generate
    description: Create the LP assets and HTML
    steps:
      # ★ Conditional action
      - type: condition
        condition: "sections.includes('hero')"
        then:
          - type: action
            tool: generateImage
            purpose: Create hero image
            args:
              prompt: |
                Hero image for {{topic}} landing page.
                Target: {{targetAudience}}
                Tone: {{tone}}
                Style: modern, professional, 16:9 aspect ratio
            storeAs: heroImage

      # ★ Main generation
      - type: action
        tool: html
        purpose: Generate the landing page
        args:
          prompt: |
            Create a high-converting landing page for {{topic}}.

            ## Context
            - Target audience: {{targetAudience}}
            - Key message: {{keyMessage}}
            - Tone: {{tone}}

            ## Competitor Insights
            {{competitor1.insights}}
            {{competitor2.insights}}

            ## Sections to include
            {{sections}}

            ## Assets
            Hero image: {{heroImage.url}}

            ## Requirements
            - Mobile responsive
            - Fast loading
            - Clear CTAs
            - SEO optimized
        storeAs: finalLP

# ★ Output definition
output:
  primary: finalLP
  artifacts:
    - heroImage
    - marketResearch
    - competitor1
    - competitor2
  summary: |
    Created LP for {{topic}} targeting {{targetAudience}}.
    Includes {{sections.length}} sections with {{tone}} tone.
```

### LP Builder (Simple Version)

```yaml
name: lp-builder
targetTool: html
objective: Create a landing page quickly
tools: [html]

procedures:
  - phase: main
    steps:
      - type: question
        id: topic
        question: What is the LP for?
      - type: question
        id: mode
        question: Creation mode?
        options:
          - { label: Quick, value: quick }
          - { label: Guided, value: guided }
      - type: action
        tool: html
        args:
          prompt: "Create LP for {{topic}}, mode: {{mode}}"
        storeAs: result

output:
  primary: result
```

### Podcast Creator (mulmocast plugin)

```yaml
name: podcast-creator
version: 1.0.0
objective: |
  Create an engaging podcast episode with research,
  script generation, and audio production.

tools:
  - mulmocast    # Podcast generation
  - exa          # Topic research
  - browse       # Source gathering

procedures:
  - phase: research
    steps:
      - type: question
        id: topic
        question: What's the podcast about?

      - type: action
        tool: exa
        purpose: Research the topic
        args:
          query: "{{topic}} latest news insights"
        storeAs: research

  - phase: planning
    steps:
      - type: decision
        id: style
        question: Podcast style?
        context: "Based on topic: {{research.summary}}"
        options:
          - { label: Interview, value: interview }
          - { label: Solo, value: solo }
          - { label: Panel, value: panel }

      - type: question
        id: duration
        question: Target duration?
        options:
          - { label: "5 min", value: 5 }
          - { label: "15 min", value: 15 }
          - { label: "30 min", value: 30 }

  - phase: generate
    steps:
      - type: action
        tool: mulmocast
        purpose: Generate podcast
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

### Quiz Designer (quiz plugin)

```yaml
name: quiz-designer
version: 1.0.0
objective: |
  Create an educational quiz with researched questions
  and accurate answers.

tools:
  - quiz         # Quiz generation
  - exa          # Fact checking

procedures:
  - phase: setup
    steps:
      - type: question
        id: subject
        question: Quiz subject?

      - type: action
        tool: exa
        purpose: Research subject for accurate questions
        args:
          query: "{{subject}} facts trivia"
        storeAs: facts

      - type: question
        id: difficulty
        question: Difficulty level?
        options:
          - { label: Easy, value: easy }
          - { label: Medium, value: medium }
          - { label: Hard, value: hard }

      - type: question
        id: count
        question: Number of questions?
        default: 5

  - phase: generate
    steps:
      - type: action
        tool: quiz
        purpose: Generate quiz
        args:
          subject: "{{subject}}"
          difficulty: "{{difficulty}}"
          count: "{{count}}"
          facts: "{{facts}}"
        storeAs: quiz

output:
  primary: quiz
```

### Travel Planner (map plugin)

```yaml
name: travel-planner
version: 1.0.0
objective: |
  Create a personalized travel itinerary with
  local recommendations and route planning.

tools:
  - map          # Route and locations
  - exa          # Travel research
  - browse       # Local info

procedures:
  - phase: gather
    steps:
      - type: question
        id: destination
        question: Where do you want to go?

      - type: parallel
        parallel:
          - type: action
            tool: exa
            purpose: Research destination
            args:
              query: "{{destination}} travel guide attractions"
            storeAs: travelGuide

          - type: action
            tool: exa
            purpose: Find local tips
            args:
              query: "{{destination}} local tips hidden gems"
            storeAs: localTips

  - phase: preferences
    steps:
      - type: question
        id: duration
        question: How many days?

      - type: question
        id: interests
        question: What interests you?
        type: multiChoice
        context: |
          Popular in {{destination}}:
          {{travelGuide.highlights}}
        options:
          - { label: Food, value: food }
          - { label: History, value: history }
          - { label: Nature, value: nature }
          - { label: Shopping, value: shopping }

  - phase: generate
    steps:
      - type: action
        tool: map
        purpose: Create itinerary with routes
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
  summary: "{{duration}}-day trip to {{destination}} planned!"
```

## Conclusion

The Self-Evolving Skill System creates a virtuous cycle:

1. **Plugins** provide base skills
2. **Servers** learn from usage
3. **Enhancements** are generated automatically
4. **PRs** contribute back to plugins
5. **Global ecosystem** improves continuously

This enables MulmoChat to become smarter over time, learning from every interaction across the world, while respecting user privacy and maintaining plugin maintainer control.
