# マルチモーダル対応の実装

## 概要

このドキュメントでは、ドキュメント・動画・画像など、マルチモーダルな入力を処理するClaude風システムの実装を解説します。

Claudeは画像・PDFを直接処理できますが、OSSでこれを実装する場合は別のアプローチが必要です。

---

## 📄 ドキュメント処理

### 推奨ツール

| ツール | 用途 | 特徴 |
|-------|------|------|
| **Docling** | PDF解析 | Anthropic推奨、高精度 |
| **Unstructured** | 多形式対応 | PDF, DOCX, HTML, Markdown |
| **PyMuPDF** | 軽量PDF処理 | 高速、シンプル |

### Doclingによる実装

Docling（[GitHub](https://github.com/docling-project/docling)）はAnthropicが推奨するPDF処理ツールです。

#### インストール

```bash
pip install docling
```

#### 基本的な使い方

```python
from docling.document_converter import DocumentConverter

def extract_document(pdf_path: str) -> dict:
    """PDFから構造化データを抽出"""
    converter = DocumentConverter()

    # PDFを変換
    result = converter.convert(pdf_path)

    # 構造化されたデータを取得
    structured_data = {
        "title": result.document.title,
        "text": result.document.export_to_markdown(),
        "tables": extract_tables(result),
        "images": extract_images(result),
        "metadata": {
            "pages": result.document.page_count,
            "created_at": result.document.metadata.get("created_at")
        }
    }

    return structured_data

def extract_tables(result) -> list:
    """テーブルを抽出"""
    tables = []

    for page in result.document.pages:
        for table in page.tables:
            tables.append({
                "page": page.page_no,
                "data": table.export_to_dataframe().to_dict(),
                "caption": table.caption
            })

    return tables

def extract_images(result) -> list:
    """画像を抽出"""
    images = []

    for page in result.document.pages:
        for image in page.images:
            images.append({
                "page": page.page_no,
                "path": image.save(f"./images/page_{page.page_no}_{image.id}.png"),
                "caption": image.caption,
                "bbox": image.bbox
            })

    return images
```

### Unstructuredによる多形式対応

```python
from unstructured.partition.auto import partition

def process_any_document(file_path: str) -> dict:
    """任意の形式のドキュメントを処理"""
    # 自動的に形式を判定して処理
    elements = partition(filename=file_path)

    # 要素を種類別に分類
    categorized = {
        "title": [],
        "text": [],
        "tables": [],
        "lists": []
    }

    for element in elements:
        element_type = element.category

        if element_type == "Title":
            categorized["title"].append(element.text)
        elif element_type == "NarrativeText":
            categorized["text"].append(element.text)
        elif element_type == "Table":
            categorized["tables"].append(element.metadata.text_as_html)
        elif element_type == "ListItem":
            categorized["lists"].append(element.text)

    return categorized
```

### 階層型ドキュメント取得

大きなドキュメントは階層的に処理します。

```python
class HierarchicalDocumentRetriever:
    def __init__(self):
        self.doc_summaries = {}  # ドキュメント全体の要約
        self.section_summaries = {}  # セクションごとの要約
        self.chunks = {}  # 詳細チャンク

    async def index_document(self, doc_path: str):
        """ドキュメントを階層的にインデックス"""
        # 1. ドキュメント全体を抽出
        full_text = extract_document(doc_path)

        # 2. セクション分割
        sections = self.split_into_sections(full_text)

        # 3. 各レベルで要約生成
        doc_id = hash(doc_path)

        # ドキュメント全体の要約
        self.doc_summaries[doc_id] = await self.summarize(
            full_text["text"],
            max_length=500
        )

        # セクションごとの要約
        for i, section in enumerate(sections):
            section_id = f"{doc_id}_section_{i}"

            self.section_summaries[section_id] = await self.summarize(
                section["content"],
                max_length=200
            )

            # チャンクに分割
            chunks = self.split_into_chunks(section["content"], chunk_size=1000)

            for j, chunk in enumerate(chunks):
                chunk_id = f"{section_id}_chunk_{j}"
                self.chunks[chunk_id] = {
                    "content": chunk,
                    "doc_id": doc_id,
                    "section_id": section_id,
                    "metadata": {
                        "section_title": section["title"],
                        "page": section["page"]
                    }
                }

    async def retrieve(self, query: str, detail_level: str = "auto") -> str:
        """クエリに応じて適切な詳細度で取得"""

        if detail_level == "overview":
            # ドキュメントレベルの要約のみ
            relevant_docs = self.find_relevant_docs(query)
            return "\n\n".join([
                self.doc_summaries[doc_id]
                for doc_id in relevant_docs
            ])

        elif detail_level == "section":
            # セクションレベル
            relevant_sections = self.find_relevant_sections(query)
            return "\n\n".join([
                self.section_summaries[sec_id]
                for sec_id in relevant_sections
            ])

        elif detail_level == "detailed":
            # チャンクレベル
            relevant_chunks = self.find_relevant_chunks(query)
            return "\n\n".join([
                self.chunks[chunk_id]["content"]
                for chunk_id in relevant_chunks
            ])

        else:  # auto
            # クエリの複雑さで判断
            if self.is_simple_query(query):
                return await self.retrieve(query, "overview")
            elif self.needs_details(query):
                return await self.retrieve(query, "detailed")
            else:
                return await self.retrieve(query, "section")

    def split_into_sections(self, doc: dict) -> list:
        """セクションに分割"""
        # Markdown見出しベースで分割
        text = doc["text"]
        sections = []

        current_section = {"title": "", "content": "", "page": 0}

        for line in text.split("\n"):
            if line.startswith("# "):
                # 新しいセクション
                if current_section["content"]:
                    sections.append(current_section)

                current_section = {
                    "title": line[2:].strip(),
                    "content": "",
                    "page": 0  # ページ番号は別途取得
                }
            else:
                current_section["content"] += line + "\n"

        if current_section["content"]:
            sections.append(current_section)

        return sections
```

---

## 🎥 動画処理

### 推奨ツール

| ツール | 用途 | 特徴 |
|-------|------|------|
| **LLaVA-NeXT-Video** | 動画理解 | 高精度、タイムスタンプ対応 |
| **Qwen2-VL** | ビジョン言語モデル | 多言語対応 |
| **VideoLLaMA** | 動画Q&A | 軽量 |

### LLaVA-NeXT-Videoによる実装

#### インストール

```bash
pip install llava-next
pip install av  # 動画処理
```

#### 基本的な使い方

```python
from llava.model.builder import load_pretrained_model
from llava.mm_utils import process_video
import av

class VideoProcessor:
    def __init__(self, model_path: str = "lmms-lab/LLaVA-NeXT-Video-7B"):
        # モデルロード
        self.tokenizer, self.model, self.image_processor, self.context_len = \
            load_pretrained_model(model_path)

    def extract_frames(
        self,
        video_path: str,
        num_frames: int = 8
    ) -> list:
        """均等にフレームを抽出"""
        container = av.open(video_path)
        video_stream = container.streams.video[0]

        total_frames = video_stream.frames
        indices = np.linspace(0, total_frames - 1, num_frames, dtype=int)

        frames = []
        for i, frame in enumerate(container.decode(video=0)):
            if i in indices:
                frames.append(frame.to_ndarray(format='rgb24'))

        return frames

    async def understand_video(
        self,
        video_path: str,
        query: str
    ) -> dict:
        """動画の内容を理解"""
        # フレーム抽出
        frames = self.extract_frames(video_path, num_frames=8)

        # 処理
        video_tensor = process_video(frames, self.image_processor)

        # 質問
        prompt = f"USER: <video>\n{query}\nASSISTANT:"

        # 推論
        input_ids = self.tokenizer(prompt, return_tensors="pt").input_ids

        output_ids = self.model.generate(
            input_ids,
            images=video_tensor,
            max_new_tokens=512
        )

        response = self.tokenizer.decode(
            output_ids[0],
            skip_special_tokens=True
        )

        return {
            "query": query,
            "response": response,
            "num_frames": len(frames)
        }

    async def extract_timeline(
        self,
        video_path: str
    ) -> list:
        """タイムラインを抽出（シーン変化検出）"""
        frames = self.extract_frames(video_path, num_frames=32)

        timeline = []

        # 各フレームで内容を要約
        for i, frame in enumerate(frames):
            timestamp = (i / len(frames)) * self.get_duration(video_path)

            description = await self.describe_frame(frame)

            timeline.append({
                "timestamp": timestamp,
                "description": description,
                "frame_index": i
            })

        return timeline

    async def describe_frame(self, frame) -> str:
        """1フレームの内容を説明"""
        # フレーム処理
        frame_tensor = self.image_processor(frame)

        prompt = "USER: <image>\nDescribe what you see in this image briefly.\nASSISTANT:"

        input_ids = self.tokenizer(prompt, return_tensors="pt").input_ids

        output_ids = self.model.generate(
            input_ids,
            images=frame_tensor,
            max_new_tokens=100
        )

        return self.tokenizer.decode(output_ids[0], skip_special_tokens=True)
```

### タイムコード付き回答

```python
class VideoQA:
    def __init__(self, video_processor: VideoProcessor):
        self.processor = video_processor
        self.timeline_cache = {}

    async def answer_with_timestamp(
        self,
        video_path: str,
        question: str
    ) -> dict:
        """タイムスタンプ付きで回答"""

        # タイムライン取得（キャッシュ）
        if video_path not in self.timeline_cache:
            self.timeline_cache[video_path] = \
                await self.processor.extract_timeline(video_path)

        timeline = self.timeline_cache[video_path]

        # 質問に関連するタイムスタンプを特定
        relevant_moments = await self.find_relevant_moments(
            question,
            timeline
        )

        # 詳細な回答を生成
        detailed_answer = await self.processor.understand_video(
            video_path,
            question
        )

        return {
            "answer": detailed_answer["response"],
            "timestamps": [
                {
                    "time": moment["timestamp"],
                    "description": moment["description"]
                }
                for moment in relevant_moments
            ],
            "sources": [
                f"Video at {self.format_timestamp(m['timestamp'])}: {m['description']}"
                for m in relevant_moments
            ]
        }

    async def find_relevant_moments(
        self,
        query: str,
        timeline: list
    ) -> list:
        """質問に関連するタイムラインを特定"""
        # ベクトル類似度で検索
        query_embedding = self.embed(query)

        scored_moments = []
        for moment in timeline:
            moment_embedding = self.embed(moment["description"])
            similarity = cosine_similarity(query_embedding, moment_embedding)

            scored_moments.append({
                **moment,
                "relevance": similarity
            })

        # 上位3件を返す
        scored_moments.sort(key=lambda x: x["relevance"], reverse=True)
        return scored_moments[:3]

    def format_timestamp(self, seconds: float) -> str:
        """タイムスタンプをフォーマット"""
        minutes = int(seconds // 60)
        secs = int(seconds % 60)
        return f"{minutes:02d}:{secs:02d}"
```

---

## 🖼️ 画像処理

Claudeは画像を直接処理できますが、OSSの場合はVision LLMを使います。

### Qwen2-VLによる実装

```python
from transformers import Qwen2VLForConditionalGeneration, AutoProcessor
from PIL import Image

class ImageUnderstanding:
    def __init__(self, model_name: str = "Qwen/Qwen2-VL-7B-Instruct"):
        self.model = Qwen2VLForConditionalGeneration.from_pretrained(
            model_name,
            torch_dtype=torch.float16,
            device_map="auto"
        )
        self.processor = AutoProcessor.from_pretrained(model_name)

    async def analyze_image(
        self,
        image_path: str,
        query: str = "Describe this image in detail."
    ) -> str:
        """画像を分析"""
        image = Image.open(image_path)

        messages = [
            {
                "role": "user",
                "content": [
                    {"type": "image", "image": image},
                    {"type": "text", "text": query}
                ]
            }
        ]

        # 推論
        text = self.processor.apply_chat_template(
            messages,
            tokenize=False,
            add_generation_prompt=True
        )

        inputs = self.processor(
            text=[text],
            images=[image],
            return_tensors="pt"
        ).to(self.model.device)

        output_ids = self.model.generate(**inputs, max_new_tokens=512)

        response = self.processor.batch_decode(
            output_ids,
            skip_special_tokens=True
        )[0]

        return response

    async def extract_text_from_image(self, image_path: str) -> str:
        """画像からテキストを抽出（OCR）"""
        return await self.analyze_image(
            image_path,
            "Extract all text visible in this image."
        )

    async def answer_visual_question(
        self,
        image_path: str,
        question: str
    ) -> str:
        """画像に関する質問に回答"""
        return await self.analyze_image(image_path, question)
```

---

## 🔗 マルチモーダルRAG

異なるモダリティを統合します。

```python
class MultimodalRAG:
    def __init__(self):
        self.doc_processor = DocumentConverter()
        self.video_processor = VideoProcessor()
        self.image_processor = ImageUnderstanding()
        self.vector_store = ChromaDB()

    async def index_multimodal_content(self, content_dir: str):
        """マルチモーダルコンテンツをインデックス"""
        for file_path in Path(content_dir).rglob("*"):
            if file_path.suffix.lower() in [".pdf", ".docx", ".md"]:
                await self.index_document(file_path)

            elif file_path.suffix.lower() in [".mp4", ".avi", ".mov"]:
                await self.index_video(file_path)

            elif file_path.suffix.lower() in [".png", ".jpg", ".jpeg"]:
                await self.index_image(file_path)

    async def index_document(self, doc_path: str):
        """ドキュメントをインデックス"""
        data = extract_document(doc_path)

        # テキストをチャンク化
        chunks = self.chunk_text(data["text"])

        # ベクトル化して保存
        for i, chunk in enumerate(chunks):
            self.vector_store.add(
                id=f"{doc_path}_chunk_{i}",
                text=chunk,
                metadata={
                    "source": doc_path,
                    "type": "document",
                    "chunk_index": i
                }
            )

    async def index_video(self, video_path: str):
        """動画をインデックス"""
        # タイムライン抽出
        timeline = await self.video_processor.extract_timeline(video_path)

        # 各タイムスタンプをインデックス
        for moment in timeline:
            self.vector_store.add(
                id=f"{video_path}_t_{moment['timestamp']}",
                text=moment["description"],
                metadata={
                    "source": video_path,
                    "type": "video",
                    "timestamp": moment["timestamp"]
                }
            )

    async def index_image(self, image_path: str):
        """画像をインデックス"""
        # 画像の説明を生成
        description = await self.image_processor.analyze_image(image_path)

        self.vector_store.add(
            id=image_path,
            text=description,
            metadata={
                "source": image_path,
                "type": "image"
            }
        )

    async def retrieve(self, query: str, k: int = 5) -> list:
        """クエリに関連するコンテンツを取得"""
        results = self.vector_store.search(query, k=k)

        enriched_results = []

        for result in results:
            metadata = result["metadata"]

            if metadata["type"] == "document":
                # ドキュメントチャンク
                enriched_results.append({
                    "content": result["text"],
                    "source": metadata["source"],
                    "type": "document"
                })

            elif metadata["type"] == "video":
                # 動画の該当箇所
                enriched_results.append({
                    "content": result["text"],
                    "source": metadata["source"],
                    "type": "video",
                    "timestamp": metadata["timestamp"],
                    "reference": f"{metadata['source']} at {self.format_time(metadata['timestamp'])}"
                })

            elif metadata["type"] == "image":
                # 画像
                enriched_results.append({
                    "content": result["text"],
                    "source": metadata["source"],
                    "type": "image",
                    "reference": f"Image: {metadata['source']}"
                })

        return enriched_results

    async def answer_with_sources(self, query: str) -> dict:
        """ソース付きで回答"""
        # 関連コンテンツを取得
        sources = await self.retrieve(query, k=5)

        # コンテキストを構築
        context = self.build_context(sources)

        # LLMで回答生成
        answer = await self.generate_answer(query, context)

        return {
            "answer": answer,
            "sources": [
                {
                    "type": s["type"],
                    "reference": s.get("reference", s["source"]),
                    "excerpt": s["content"][:200]
                }
                for s in sources
            ]
        }

    def build_context(self, sources: list) -> str:
        """ソースからコンテキストを構築"""
        context_parts = []

        for source in sources:
            if source["type"] == "document":
                context_parts.append(f"""
<document_excerpt source="{source['source']}">
{source['content']}
</document_excerpt>
""")

            elif source["type"] == "video":
                context_parts.append(f"""
<video_moment source="{source['source']}" timestamp="{source['timestamp']}">
{source['content']}
</video_moment>
""")

            elif source["type"] == "image":
                context_parts.append(f"""
<image_description source="{source['source']}">
{source['content']}
</image_description>
""")

        return "\n".join(context_parts)
```

---

## 💰 コスト最適化

マルチモーダル処理はコストがかかるため、最適化が重要です。

### キャッシング戦略

```python
class MultimodalCache:
    def __init__(self, cache_dir: str = "./cache"):
        self.cache_dir = Path(cache_dir)
        self.cache_dir.mkdir(exist_ok=True)

    def get_cache_key(self, file_path: str, operation: str) -> str:
        """キャッシュキーを生成"""
        file_hash = hashlib.md5(Path(file_path).read_bytes()).hexdigest()
        return f"{operation}_{file_hash}"

    async def get_or_compute(
        self,
        file_path: str,
        operation: str,
        compute_fn
    ):
        """キャッシュがあれば返す、なければ計算"""
        cache_key = self.get_cache_key(file_path, operation)
        cache_path = self.cache_dir / f"{cache_key}.json"

        if cache_path.exists():
            # キャッシュヒット
            with open(cache_path, 'r') as f:
                return json.load(f)

        # キャッシュミス、計算実行
        result = await compute_fn(file_path)

        # キャッシュに保存
        with open(cache_path, 'w') as f:
            json.dump(result, f)

        return result

# 使用例

cache = MultimodalCache()

# 動画処理（キャッシュ利用）
timeline = await cache.get_or_compute(
    video_path,
    "extract_timeline",
    video_processor.extract_timeline
)
```

### 段階的処理

```python
async def process_video_efficiently(video_path: str, query: str):
    """段階的に処理してコストを削減"""

    # ステップ1: タイムラインのみ（低コスト）
    timeline = await video_processor.extract_timeline(video_path)

    # ステップ2: 関連部分を特定
    relevant_moments = find_relevant_moments(query, timeline)

    if not relevant_moments:
        return "動画に関連する情報が見つかりませんでした"

    # ステップ3: 関連部分のみ詳細処理（高コスト）
    detailed_analyses = []

    for moment in relevant_moments[:3]:  # 上位3件のみ
        analysis = await video_processor.analyze_frame(
            video_path,
            moment["frame_index"],
            query
        )
        detailed_analyses.append(analysis)

    return synthesize_answer(query, detailed_analyses)
```

---

## 📚 参考資料

### 公式リソース

- [Docling GitHub](https://github.com/docling-project/docling)
- [LLaVA-NeXT](https://llava-vl.github.io/blog/2024-01-30-llava-next/)
- [Qwen2-VL](https://github.com/QwenLM/Qwen2-VL)

### 関連ドキュメント

- [04-implementation-guide.md](./04-implementation-guide.md) - 基本実装
- [06-practical-examples.md](./06-practical-examples.md) - 実践例

---

**次**: [06-practical-examples.md](./06-practical-examples.md) - 動作するコード例
