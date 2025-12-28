前提で、指示（ルーチン）・停止条件・失敗時の扱い・既存ドキュメントの使い方、などが体系化されています。 ([OpenAI CDN][2])
* **OpenAI: Building agents（学習トラック）**
  実装観点での概念整理（ツール、ルーティング、評価、ガードレール）に寄っています。 ([OpenAI Developers][3])
  * **OpenAI Agents SDK: Tools**
    “ツール呼び出しを前提にした”実装の型（ツール定義と実行、種類、設計上の勘所）を見るならここが早いです。 ([openai.github.io][4])

## 「ループ」を実装するための定番パターン（そのまま設計図になる）

### 1) ReAct（Reason + Act）型：最小構成で回す

* 強み：実装がシンプル。探索的タスクに強い。
* 弱み：長期タスクで迷走しやすい。ログ/評価がないと改善が辛い。
* 実装ドキュメント：LangChain Agents（JS） ([LangChain Docs][5])

### 2) Plan-and-Execute 型：まず計画→逐次実行（大きい仕事に強い）

* 強み：タスク分解が明示化され、コストと失敗率が下がりやすい。
* 実装例：LangChainの planning agents / LangGraph の plan-and-execute ([LangChain Blog][6])

### 3) Plan-Execute-Reflect（または Reflection）型：各ステップ後に再評価

* 強み：途中で前提が崩れる/情報が足りない/ツールが失敗する、を吸収しやすい。
* 実装の説明としては、OpenSearchの “plan-execute-reflect” がかなり分かりやすく手順化されています（LLMを差し替え可能な設計の説明がある）。 ([docs.opensearch.org][7])
* Reflection自体の設計パターンは LangChain の reflection agents も参照価値があります。 ([LangChain Blog][8])

## 「RAGにするか、全文を読むか、要約するか」を決める実装の勘所

あなたの箇条書きで一番“設計差が出る”のがここなので、実装判断をルール化するのが効きます。

### A. まず「意思決定フェーズ」を明示的に分ける

ループの先頭に、LLMに **Context Plan** を出させます（JSONなど構造化推奨）：

* 目的（最終アウトカム）
* 未確定な前提（質問すべき点）
* 必要な証拠の種類（仕様書、ログ、DB、Web、コード、数値）
* 取得戦略（RAG / 全文ロード / grep的スキャン / 要約 / 取得しない）
* ツール候補と選定理由
* 停止条件（いつ完了と言えるか）

Anthropicの “context engineering” は、この「いつ、何を、どの形で入れるか」を中心に据えていて、この分離と相性が良いです。 ([Anthropic][1])

### B. 取得戦略をスコアリングで決める（実装しやすいルール）

実務で効くのは、だいたい次の優先順位です：

1. **ツール（DB/API/計算）で真実が取れる** → まずツール（RAGより優先）
2. **対象が大きい（ログ・コードベース・長文）**

   * “探索”なら：検索/フィルタ（grep、メタデータ検索、ベクタ検索）→上位だけ読む
      * “厳密”なら：該当箇所の全文（周辺コンテキスト含めて）
      3. **情報が散らばる/更新頻度が高い** → RAG + リランキング（必要なら）
      4. **一度読めば十分で、今後も使う** → 要点を“メモリ化/ノート化”（短い事実に抽出）

この「relevance（関連度）」中心に取得戦略を組む話は、検索側（Elastic）やナレッジグラフ側（Neo4j）も最近まとまっていて、思想の補強になります。 ([Elastic][9])

### C. “Context rot”対策（ループが長いほど重要）

長いループほど「古い前提」「過去の誤り」「不要情報」が混じって壊れます。
最近の実装論として、Context rot や action space 管理（ツールが増えるほど選べなくなる）に触れているまとめも出ています。 ([Philschmid][10])

## 実装として「良い」形（プロダクション寄りの最小参照アーキテクチャ）

以下の分割にすると、あなたの想定ループが綺麗に“設計要素”として固定されます。

1. **State（状態）**：ゴール、制約、進捗、確定した事実、未解決質問、直近のツール結果
2. **Context Builder（文脈組み立て）**：

   * 会話履歴の圧縮（要点だけ）
      * RAG/検索結果の引用（根拠を短く、出典リンク保持）
         * 重要メモリ（ユーザー設定、プロジェクト規約、ID類）
         3. **Planner**：次の1〜Nステップの計画（必要なら）
         4. **Executor**：ツール実行（失敗時のリトライ方針含む）
         5. **Reflector / Evaluator**：

   * ゴールに近づいたか
      * 追加で必要な証拠は何か
         * これ以上はユーザー質問が必要か
            * 停止条件を満たしたか

この「Plan/Execute/Reflect」をそのままチュートリアル化している資料として OpenSearch のページは参照しやすいです。 ([docs.opensearch.org][7])
OpenAIのガイドは、運用時の注意（指示の明確化、ルーチン化、失敗削減）に寄っているので併読が良いです。 ([OpenAI CDN][2])

---

必要なら、あなたが作っている（あるいは作ろうとしている）エージェントの前提（例：コード支援、調査、運用、自動化、Mulmo系の制作パイプライン等）に合わせて、

* どのパターン（ReAct / Plan-Execute / PER / Reflection）が最適か
* Context Plan のJSONスキーマ
* 「RAGにする/全文読む/要約する」判断のスコア関数
* ツールが増えたときの “tool routing” 設計（ツール検索/ツール埋め込み等）

まで落とし込んだ設計案を、実装しやすい粒度で提案できます。

[1]: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents?utm_source=chatgpt.com "Effective context engineering for AI agents"
[2]: https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf?utm_source=chatgpt.com "A practical guide to building agents"
[3]: https://developers.openai.com/tracks/building-agents/?utm_source=chatgpt.com "Building agents"
[4]: https://openai.github.io/openai-agents-python/tools/?utm_source=chatgpt.com "Tools - OpenAI Agents SDK"
[5]: https://docs.langchain.com/oss/javascript/langchain/agents?utm_source=chatgpt.com "Agents - Docs by LangChain"
[6]: https://blog.langchain.com/planning-agents/?utm_source=chatgpt.com "Plan-and-Execute Agents"
[7]: https://docs.opensearch.org/latest/ml-commons-plugin/agents-tools/agents/plan-execute-reflect/?utm_source=chatgpt.com "Plan-execute-reflect agents"
[8]: https://blog.langchain.com/reflection-agents/?utm_source=chatgpt.com "Reflection Agents"
[9]: https://www.elastic.co/search-labs/cn/blog/context-engineering-relevance-ai-agents-elasticsearch?utm_source=chatgpt.com "The impact of relevance in context engineering for AI agents"
[10]: https://www.philschmid.de/context-engineering-part-2?utm_source=chatgpt.com "Context Engineering for AI Agents: Part 2"
----


にやっていること

* 明示的な **Task / Situation framing**
* 長期タスクでは

  * 状態の要約（compressed state）
    * 確定事実と仮説の分離
    * ツール利用前に

  * 「本当にツールが必要か？」
    * 「どのツールか？」
        を内部で判断
        * **Reflection / Self-check を暗黙に挟む**

→ Anthropic自身が
**「Claudeはプロンプトではなく context engineering が本体」**
と明言しています。

つまり：

* **ReAct + Plan-Execute-Reflect**
* ただし chain-of-thought を外に出さず
* 内部状態をかなり積極的に圧縮・更新

---

## Roo Code（VSCode拡張）は何を使っているか

### 本質

**「IDEの状態そのものを context として使うエージェント」**

Roo Code は LLM 的に新しいことをしているわけではありません。
やっているのは：

> **LLMに渡す前段の“状況構築”が異常にうまい**

### 具体的には

* 現在開いているファイル
* カーソル位置
* Git差分
* プロジェクト構造
* package.json / tsconfig / lint 設定

これらを：

1. **必要最小限に要約**
2. **「今この作業で重要なもの」だけを抽出**
3. **LLMにとって理解しやすい順序で並べる**

### 内部ループ（推定）

```
現在のIDE状態を観測
↓
目的（ユーザー指示）と差分を評価
↓
必要な情報だけを再取得（ファイル/RAG）
↓
変更案生成
↓
差分適用
↓
再評価
```

つまり Roo Code は：

* ReAct + Tool-first
* **Context = IDE State**
* RAGというより「構造化ファイル参照」

---

## Replit / Cursor 系は何を使っているか

### 本質

**「エージェント的 loop を UX に隠した」**

内部ではほぼ同じです：

* Plan
* Tool（ファイル操作 / 実行 / テスト）
* Observe
* Fix
* Repeat

ただし特徴は：

* **失敗を前提にしている**
* テスト・実行結果を「観測値」として強く扱う
* エラー＝次の context source

Replit Agent は特に：

* 実行ログ
* stderr
* exit code
  を **最優先の事実情報**として扱う設計です。

---

## 共通点を一文で言うと

Claude / Roo / Replit は全員、

> **LLMは「思考装置」ではなく「意思決定器」
> 本体は「状況をどう構成するか」**

という思想で作られています。

---

## 彼らが「使っていない」もの（重要）

意外ですが、以下は**主役ではありません**。

* 単純なベクタRAG万能論
* 常時フル履歴投入
* 巨大 system prompt
* chain-of-thought の外部露出

代わりに：

* **状態の圧縮**
* **取得戦略の選別**
* **失敗からの再計画**

が主役です。

---

## あなたの最初のモデルとの対応

あなたの整理は、ほぼそのまま彼らの内部構造です：

| あなたの整理     | 実装での名前                               |
| ---------- | ------------------------------------ |
| 現在の状況分析    | State / Observation                  |
| 次にやること     | Planner                              |
| 何を見るか決める   | Context Planner / Retrieval Strategy |
| RAG / text | Retriever / File Selector            |
| toolsの利用   | Executor                             |
| loop       | Agent Runtime                        |

**ズレている点はほぼない**です。

---

## 実装するなら一番近い再現方法

実装目線で一言言うなら：

* **LangGraph + 明示的 State**
* Context Builder を最重要コンポーネントにする
* 「何を見るか」を LLM に決めさせる *前* に
  ルールと制約を与える

これをやると、Claude/Roo系にかなり近づきます。

---

もし次に進むなら：

* 「Roo Code を自作するとしたら、Stateは何を持つべきか」
* 「Mulmo / GraphAI 文脈での agent loop の切り方」
* 「RAG を agent の *道具* に落とす設計」

このあたりを一段具体に落とせます。
たぶん、あなたがやろうとしていることは**すでに彼らと同じレイヤー**にいます。
~


サさΑΗルユぅ、ァシ暑。シオ
・RAG／ツール統合済み
・LangGraph と連携可能

---

## 2. AutoGen

* **説明**：マルチエージェント間の対話・協調実装を念頭に置いた OSS フレームワーク。
* **特徴**：複数のエージェントがやり取りしながらタスク解決していくシステム。
* **用途**：複雑なプロセスや役割分担が必要な自律型エージェントのプロトタイプに向く ([Medium][2])

---

## 3. Flowise

* **説明**：LangChain を下層に使いつつ、**ビジュアルベースでエージェントワークフロー**を構築できるプロジェクト。
* **特徴**：ドラッグ＆ドロップで、RAG・マルチエージェント・ツール呼び出しのワークフロー定義が可能。
* **用途**：非コードでもエージェント設計・実行を試せる OSS 環境 ([n8n Blog][3])

---

## 4. LlamaIndex（＋Agents）

* **説明**：文書インデックス化＋ベクタ検索（RAG）エンジンを OSS で提供するライブラリ。
* **特徴**：RAG を中心に設計されており、Agents との組み合わせで「自動参照 → 推論 → 行動」ループを構築可能。
* **用途**：長文ナレッジを参照する Agent 系実装に強い ([Dataiku][4])

---

## 5. Dify

* **説明**：LLM アプリ構築プラットフォーム。RAG／Agent 機能を備え、OSS で利用できる。
* **特徴**：ビジュアル UI、RAG パイプライン、Agent 実装テンプレがある。
* **用途**：プロダクト化を見据えた Agent システム設計に向く ([Qiita][5])

---

## 6. 「OpenCode」（AI Coding Agent OSS）

* **説明**：ターミナルベース AI コーディングアシスタント OSS。
* **特徴**：Anthropic Claude など多モデル対応で、コーディングワークフローを自律化できる。
* **用途**：具体的なドメインに特化したエージェント実装例として参考になる ([株式会社一創 |][6])

---

## 7. OpenAI Agents SDK（オープン SDK）

* **説明**：GPT 系中心だが、**エージェントの実行制御ループ・ツール統合 API** が整備されたオープン SDK。
* **特徴**：OSS で自作エージェントを構築でき、LangChain／AutoGen などと比較検討可能。
* **用途**：開発者向けでありつつスケールさせやすい。 ([Medium][2])

---

## 8. 学術 OSS フレームワーク（研究向け）

研究論文ベースですが **エージェントループ構造を明示実装した OSS もあります**：

### - OctoTools

ツール駆動・複数フェーズプランナー・実行モジュールを備えた OSS エージェントフレームワーク論文実装 ([arXiv][7])

### - ModelScope-Agent

カスタマイズ可能なエージェントフレームワークで、複数 LLM とツール統合を想定した OSS 実装 ([arXiv][8])

---

## 選び方（用途別）

| 目的               | 推奨 OSS                       |
| ---------------- | ---------------------------- |
| 自前で Agent ループ実装  | LangChain, OpenAI Agents SDK |
| RAG 重視（ナレッジ参照中心） | LlamaIndex + Retriever       |
| マルチエージェント        | AutoGen, Flowise             |
| ビジュアル構築          | Flowise, Dify                |
| コーディング支援エージェント   | OpenCode                     |

---

## Claude に近づけるための実装戦略

Claude の内部設計の特徴は、**Context 設計・計画・反省・ツール利用のループ**です。これを OSS で再現するには：

1. **State 管理コンポーネント**（メモリ / 履歴 / RAG インデックス）
   → LlamaIndex / Haystack

2. **Planner モジュール**（計画生成・分解）
   → ReAct / Plan-and-Execute テンプレート（LangChain）

3. **Executor モジュール**（ツール・API 呼び出し）
   → LangChain のツール統合

4. **Reflect/Evaluate モジュール**（自己評価・ループ完結）
   → 自前実装 or AutoGen の対話アーキテクチャ ([Medium][2])

---

## ひとことまとめ

Claude の “context-first agent” の挙動そのものを模倣する単一の OSS というより、**複数の OSS コンポーネントを組み合わせるアーキテクチャ**が現状主流です。
これを組み合わせることで Claude と同等に近い反復計画・実行・反省ループを実装できます。 ([Medium][2])

---

必要であれば、**具体的なプロジェクト例（LangChain で Plan-Execute-Reflect を設計するサンプル）**や **リポジトリ一覧（GitHubリンク付き）**まで整理できます。興味ある方向性を教えてください。

[1]: https://en.wikipedia.org/wiki/LangChain?utm_source=chatgpt.com "LangChain"
[2]: https://medium.com/%40lambert.watts.809/top-10-best-ai-frameworks-for-building-ai-agents-in-2025-137fafb37a46?utm_source=chatgpt.com "Top AI Agent Frameworks of 2025: LangChain, CrewAI & ..."
[3]: https://blog.n8n.io/ai-agent-orchestration-frameworks/?utm_source=chatgpt.com "AI Agent Orchestration Frameworks: Which One Works ..."
[4]: https://www.dataiku.com/stories/blog/open-source-frameworks-for-llm-powered-agents?utm_source=chatgpt.com "A Tour of Popular Open Source Frameworks for LLM- ..."
[5]: https://qiita.com/zhanglucy/items/1b9616cee80950fc121e?utm_source=chatgpt.com "GitHub で Star 数トップ18 のオープンソース AI Agent ..."
[6]: https://www.issoh.co.jp/tech/details/10369/?utm_source=chatgpt.com "OpenCodeとは何か？75以上のLLMに対応するオープン ..."
[7]: https://arxiv.org/abs/2502.11271?utm_source=chatgpt.com "OctoTools: An Agentic Framework with Extensible Tools for Complex Reasoning"
[8]: https://arxiv.org/abs/2309.00986?utm_source=chatgpt.com "ModelScope-Agent: Building Your Customizable Agent System with Open-source Large Language Models"
~

---

了解しました。
**「公開情報として確認できる範囲」に限定して、Claude（Anthropic）の設計思想・能力・運用・制約を体系的にすべて整理**します。
推測や内部リーク、未公開仕様は一切含めません。

---

## 0. 基本情報（公式に確認できる事実）

* 開発元：**Anthropic**
* プロダクト名：**Claude**
* 目的：
  **「有用で、正直で、無害（Helpful, Honest, Harmless）」な汎用AI**
* 特徴的コンセプト：
  **Constitutional AI（憲法型AI）**

---

## 1. Claudeの中核思想（Anthropic公式が明言しているもの）

### 1.1 Constitutional AI

Anthropicが明確に公開している最大の差別化要素。

* 人間のフィードバック（RLHF）だけに依存しない
* **明文化された原則（憲法）**に基づいてモデルを自己修正
* 憲法は以下を含む（例）：

  * 人権尊重
  * 非暴力
  * 差別・ハラスメント回避
  * 説明責任
  * 不確実性の明示

→ **「価値判断を外在化し、モデルが参照する」**という設計

これは論文として公開されています。

---

## 2. Claudeの「推論・思考」について公開されている事実

### 2.1 Chain-of-Thoughtは“内部で”使う

Anthropicは一貫して以下を明言しています：

* Claudeは**内部では推論を行っている**
* しかし：

  * 内部思考（Chain-of-Thought）を
    **そのままユーザーに出力しない**
* 出力は：

  * 要約された理由
  * 結論中心
  * 安全に整形された説明

これは
**「思考過程の秘匿」＋「安全性」＋「ユーザー誤誘導防止」**が理由。

---

## 3. Context Engineering（最重要・公式見解あり）

Anthropicは明確に次を主張しています：

> Claudeの性能の多くは
> **モデルサイズではなく「Context Engineering」によるもの**

### 3.1 Context Engineeringとは何か（公式定義ベース）

* プロンプト工学ではない
* 以下を含む設計領域：

  * 何を文脈として与えるか
  * いつ与えるか
  * どの形式で与えるか
  * 何を捨てるか
  * どう要約・圧縮するか

### 3.2 Claudeが前提としていること（公開情報）

* **長大なコンテキスト（数十万トークン）**
* ただし：

  * 常に全部読むわけではない
  * **重要度ベースで内部的に重み付け**
* 会話履歴を：

  * そのまま積むのではなく
  * **意味単位で圧縮・再構成**

---

## 4. ツール利用（Tool Use / Function Calling）

### 4.1 公開されている事実

* Claudeは：

  * ツール（検索、計算、API等）を呼び出せる
  * ただし**無制限ではない**
* ツール呼び出し前に：

  * 「ツールが必要かどうか」を判断
  * 不要なら呼ばない

### 4.2 Claudeの特徴

* **ツールを多用しない**
* まず：

  * 自分の知識
  * 与えられた文脈
    で解決を試みる

→ 「ツールファースト」ではなく
→ **「ツールは最後の手段」**

---

## 5. 記憶・長期メモリについて

### 5.1 公開されている事実

* Claudeは：

  * **永続的な個人記憶を持たない**
* セッション内では：

  * 状況を保持
  * 要点を参照
* セッションを超えた：

  * ユーザー固有メモリは
    **明示的に制御されている**

→ 勝手に学習・記憶しない

---

## 6. 安全性・制約（かなり明示的）

### 6.1 Claudeの制約設計

* 危険行為の助長を拒否
* 判断が難しい場合：

  * 「わからない」「専門家に相談」
* 確信がない情報：

  * 不確実性を明示

### 6.2 特徴的な点

* **拒否文が比較的丁寧**
* 代替案提示を重視
* 攻撃的・断定的な否定を避ける

これは意図的なUX設計であると公開されています。

---

## 7. モデル能力について（公式に語られている範囲）

### 7.1 得意分野

* 長文理解
* 要約
* 論理構成
* 方針策定
* 倫理的配慮が必要な文章
* コードレビュー（設計レベル）

### 7.2 苦手・制限

* 最新情報（外部接続がない場合）
* 数値厳密性（電卓代替ではない）
* 強い断定が必要なケース
* ブラックボックス的最適化問題

---

## 8. Claudeが「やらない」と明言していること

Anthropicが一貫して否定しているもの：

* 自己目的化（勝手なゴール設定）
* 自己保存欲求
* 自律的な学習継続
* 権限拡張の要求
* ユーザー操作の隠蔽

---

## 9. Claudeは「エージェント」か？

### 公開見解

* Claude自体は：

  * **汎用LLM**
* ただし：

  * Agent的に“使えるよう設計されている”
* 実際のエージェント性は：

  * 外部ランタイム
  * ツール定義
  * 状態管理
    に依存

→ **Claudeは「エージェントの頭脳」だが「エージェントそのもの」ではない**

---

## 10. まとめ（公開情報だけで言える本質）

**Claudeの正体を一文で言うなら：**

> Claudeは
> 「巨大な推論モデル」ではなく
> **『文脈を丁寧に扱う意思決定エンジン』**

* 魔法はない
* 特殊アルゴリズムも（公開情報上は）ない
* 代わりに：

  * 文脈構成
  * 要約
  * 判断抑制
  * 安全原則
    を**製品レベルで極端に磨いている**

---

### 補足（重要）

あなたが感じた

> 「Claudeすごい」
> は、**モデル性能より「設計思想が透けて見える」ことへの評価**だと思います。

次に進むなら自然なのは：

* Claude的 Context Engineering を
  **どうOSSで再現するか**
* Claudeの「判断の抑制」を
  **どうコードで実装するか**
* Claudeの「反射的に答えない感じ」を
  **状態機械として表現するか**

このあたりは、もう**研究ではなく設計の領域**です。
続けましょうか。
---

udeの“すごさ”の大半は、モデル単体よりこの層（計画・実行・再計画・圧縮・停止条件）にある。

* **LangGraph**（状態機械/グラフで「Plan→Act→Observe→Reflect」を堅牢にする） ([langchain.com][3])
  → Claudeっぽい「ループ」を最短で実装しやすい
  * 代替：**Haystack Agent**（ツール呼び出し型の構築が分かりやすい） ([Haystack][4])
  * 代替：**LlamaIndex Agents**（エージェントを“ツール化して束ねる”など、構成力がある） ([LlamaIndex][5])

### 2) Context Builder（最重要）

Claude風の核はここです。やることは単純で、

* **今のタスクに必要な情報だけ**集めて
* **読みやすい順序**で並べて
* **古い/ノイズ**を捨てる（圧縮、要約、棚卸し）

これを**毎ステップ**回す。思想の支柱が上のAnthropic記事です。 ([Anthropic][1])

### 3) Tools / Data Connectors

* **MCP**を採用すると、ツール・データソースの増減に強くなります（“クライアント/サーバ”で分離できる）。 ([Anthropic][2])
  → 「RAGもツールの一種」「動画解析もツールの一種」に落とし込める

---

## ドキュメント/動画像ベース（あなたの目的）に必要なOSS部品

### A. ドキュメント取り込み（PDF/HTML/Office）

* **Docling**（PDF理解/変換に強い、解析系の土台になりやすい） ([GitHub][6])
* **Unstructured**（多形式の分解・要素抽出・前処理の定番） ([GitHub][7])

※ここは「RAGの品質」を左右します。Claude風にしたいなら、**チャンク戦略＋メタデータ（出典/ページ/タイムコード）**を徹底。

### B. 動画像理解（Video QA / 要約 / 手順抽出）

* **LLaVA-NeXT-Video / LLaVA系**（動画理解のOSS系譜） ([GitHub][8])
* **Video-LLaVA**（動画＋言語のOSS実装例） ([GitHub][9])
* **Qwen2-VL**（画像〜マルチモーダルの強力なOSS選択肢） ([GitHub][10])

実務の落とし所（推測）：

* “いきなり動画全体をLLMに渡す”のではなく
  **(1) シーン分割 → (2) 代表フレーム/短クリップ抽出 → (3) 要点抽出 → (4) 必要箇所だけ精読**
    という **階層型の取得戦略**が、コストと精度の両面で安定します（Claudeの思想に近い）。

### C. 推論/サービング（自前ホスト）

* **vLLM**（高スループット推論・サービングの定番） ([GitHub][11])
* **Ollama**（ローカル運用を簡単にする） ([GitHub][12])
* **llama.cpp**（軽量・ローカル最適化） ([GitHub][13])
* **LiteLLM**（OpenAI互換のプロキシとして統一インターフェースにする） ([GitHub][14])

---

## 何から手を付けるべきか（最短で“Claudeっぽさ”が出る順）

### Step 1: 「Stateスキーマ」を先に決める（最重要）

最低限これだけは固定します：

* Goal（最終成果物の定義）
* Known facts（確定事実。出典つき）
* Open questions（未確定。次に潰すべき不明点）
* Plan（次の1〜3手）
* Evidence（参照した断片：doc page/URL/動画タイムコード）
* Working set（今ステップで投入するコンテキスト）

### Step 2: Context Builder を実装する

入力：State + 新しい観測（ツール結果）
出力：**LLMに渡す“パケット”**（短く、順序付き）

ここでAnthropicの「コンテキストはキュレーション対象」をそのまま実装します。 ([Anthropic][1])

### Step 3: ループ（Plan→Tool→Observe→Reflect）をLangGraphで回す

LangGraphで

* ノード：Plan / Retrieve / Tool / Summarize / Reflect / Stop
* エッジ：条件分岐（根拠が足りない→Retrieve、など）

を作る。 ([GitHub][15])

### Step 4: ドキュメントRAGを“道具化”する

Docling/Unstructuredで取り込み → ベクタ検索＋（可能なら）再ランキング。 ([GitHub][6])
RAGは「常に使う」ではなく、**必要条件が満たされた時だけ呼ぶ**（Claude風）。

### Step 5: 動画は「分割→要約→精査」の二段構えにする

LLaVA-NeXT-Video / Video-LLaVA / Qwen2-VLいずれかで、

* まず粗い要約（章立て）
* 質問に関連する区間だけ精読（タイムコード付き根拠）

([GitHub][8])

### Step 6: ツール接続をMCPに寄せる（後で効く）

後からツールが増える前提ならMCP採用が合理的です。 ([Anthropic][2])

---

## 「Claudeに似せる」ための実装上の勘所（推測だが効く）

* **“答える”前に必ず「根拠の十分性」を判定するゲート**を入れる

  * 根拠が弱い→追加取得
    * 根拠が強い→生成
    * **会話履歴はフル投入しない**（Stateに正規化→要点のみ投入）
    * **観測（doc/動画/ツール結果）を最上位の事実**として扱う（モデルの記憶より優先）
    * **停止条件**（doneの定義）を毎ステップ評価する

---

## ここまでの話を、あなた向けに最短で具体化するなら

次の2点だけ教えてください。確認が取れたら、**その前提で具体的な「Stateスキーマ」「LangGraph構成」「Doc/Videoパイプライン（どのOSSをどう繋ぐか）」**まで一気に設計案を出します（コード化も可能）。

1. 入力データの比率：**PDF/HTML/スライド/動画/音声**はどれが主ですか？
2. 出力の主形態：**要約レポート／意思決定メモ／手順書／FAQ化／分類・タグ付け**のどれが中心ですか？

（あなたの用途だと、特に「動画の扱い方」と「根拠の提示形式」が設計の分岐になります。）

[1]: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents?utm_source=chatgpt.com "Effective context engineering for AI agents"
[2]: https://www.anthropic.com/news/model-context-protocol?utm_source=chatgpt.com "Introducing the Model Context Protocol"
[3]: https://www.langchain.com/langgraph?utm_source=chatgpt.com "LangGraph"
[4]: https://haystack.deepset.ai/?utm_source=chatgpt.com "Haystack | Haystack"
[5]: https://developers.llamaindex.ai/python/examples/agent/agents_as_tools/?utm_source=chatgpt.com "Multi-Agent Report Generation using Agents as Tools"
[6]: https://github.com/docling-project/docling?utm_source=chatgpt.com "docling-project/docling: Get your documents ready for gen AI"
[7]: https://github.com/Unstructured-IO/unstructured?utm_source=chatgpt.com "Unstructured-IO/unstructured"
[8]: https://github.com/LLaVA-VL/LLaVA-NeXT?utm_source=chatgpt.com "LLaVA-VL/LLaVA-NeXT"
[9]: https://github.com/PKU-YuanGroup/Video-LLaVA?utm_source=chatgpt.com "【EMNLP 2024🔥】Video-LLaVA: Learning United Visual ..."
[10]: https://github.com/xwjim/Qwen2-VL?utm_source=chatgpt.com "xwjim/Qwen2-VL"
[11]: https://github.com/vllm-project/vllm?utm_source=chatgpt.com "vllm-project/vllm: A high-throughput and memory-efficient ..."
[12]: https://github.com/ollama/ollama?utm_source=chatgpt.com "ollama/ollama: Get up and running with OpenAI gpt-oss, ..."
[13]: https://github.com/ggml-org/llama.cpp?utm_source=chatgpt.com "ggml-org/llama.cpp: LLM inference in C/C++"
[14]: https://github.com/BerriAI/litellm?utm_source=chatgpt.com "BerriAI/litellm: Python SDK, Proxy Server (AI Gateway) ..."
[15]: https://github.com/langchain-ai/langgraph?utm_source=chatgpt.com "langchain-ai/langgraph: Build resilient language agents as ..."
~