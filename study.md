# RAG完整流程系统

## 项目概述

本项目基于多阶段检索增强生成（RAG, Retrieval-Augmented Generation）与模型上下文协议（MCP, Model Context Protocol）设计，目标是搭建一个可扩展、高可观测、易迭代的智能问答与知识检索框架。

### 环境配置

```
llm:
  provider: "deepseek"  # Options: openai, azure, ollama, deepseek
  model: "deepseek-chat"
  deployment_name: ""
  azure_endpoint: ""
  api_version: ""
  api_key: "sk-abexxxxxxxxx2b7a29"
  base_url: "https://api.deepseek.com/v1"
  temperature: 0.0
  max_tokens: 4096

# =============================================================================
# Embedding Configuration
# =============================================================================
embedding:
  provider: "ollama"  # Options: openai, azure, ollama
  model: "bge-m3:latest"
  dimensions: 1024
  azure_endpoint: ""
  deployment_name: ""
  api_version: ""
  api_key: ""
  base_url: "http://localhost:11434"

# =============================================================================
# Vision LLM Configuration (for Image Captioning)
# =============================================================================
vision_llm:
  enabled: true  # Set to true to enable vision capabilities
  provider: "siliconflow"
  model: "Qwen/Qwen3-VL-32B-Instruct"
  azure_endpoint: ""
  deployment_name: ""
  api_version: ""
  api_key: "sk-frhnyxxxxxxxxxxzvtvjtmckx"
  base_url: "https://api.siliconflow.com/v1"
  max_image_size: 2048  # Max dimension (width/height) for image compression

# =============================================================================
# Vector Store Configuration
# =============================================================================
vector_store:
  provider: "chroma"  # Options: chroma, qdrant, pinecone
  persist_directory: "./data/db/chroma"
  collection_name: "knowledge_hub"

# =============================================================================
# Retrieval Configuration
# =============================================================================
retrieval:
  dense_top_k: 20
  sparse_top_k: 20
  fusion_top_k: 10
  rrf_k: 60  # RRF constant

# =============================================================================
# Rerank Configuration
# =============================================================================
rerank:
  enabled: true
  provider: "llm"  # Options: none, cross_encoder, llm
  model: ""
  top_k: 5
```



### 启动

```cmd
$ .\.venv\Scripts\python scripts/start_dashboard.py --port 8501
```

```python
Quick Start（在仓库根目录执行）

- 导入文档：.\.venv\Scripts\python scripts/ingest.py <path-to-pdf-or-folder>

- 发起查询：.\.venv\Scripts\python scripts/query.py "你的问题"

- 启动 Dashboard（如需手动重启）：.\.venv\Scripts\python scripts/start_dashboard.py --port 8501

- 启动 MCP Server：.\.venv\Scripts\python main.py
```

### 基础概念

1. **类** 

2. **工厂模式**

   用最通俗的话来说：**工厂模式就是一个“专门负责造对象的代工厂”。**

   假设你要买一辆车：

   - **不用工厂（自己造）**：你需要自己去买钢铁、橡胶、发动机，自己焊接、组装。你（调用者）必须知道造车的所有复杂细节。
   - **使用工厂（4S店/汽车工厂）**：你只需要告诉销售：“我要一辆红色的特斯拉 Model 3”。你不需要懂怎么造车，工厂直接把车钥匙交给你。

   在代码中，**“造对象”**（`obj = MyClass()`）就是“造车”。如果造对象的过程很复杂（比如要根据配置文件决定造哪种对象），我们就把“造对象”的逻辑剥离出来，交给一个专门的“工厂类”去处理。

   **工厂模式的核心优势**
   	解耦（分离关注点）：调用者只关心“给我一个能切分文本的工具”，不关心工具怎么造出来的；工厂只关心“怎么根据配置造工具”，不关心工具拿去干什么。
   	集中管理（单一职责）：所有创建对象的代码都在工厂里。如果某个切分器的初始化参数变了，只需要改工厂类，不用去翻遍整个项目的业务代码。
   	完美配合多态/ABC：工厂返回的通常是抽象基类（如 BaseSplitter）的接口，调用者用统一的方法（如 split()）操作，根本不知道具体是哪个子类。
   
   3. **幂等性**
   
      幂等性是分布式系统和可靠软件设计中的一个核心概念，它确保一个操作无论执行一次还是多次，其产生的效果和系统状态都保持完全一致。简单来说，**“重复请求，结果不变”**。
   
      幂等性（Idempotence）源于数学，指一个函数或操作多次应用与一次应用的效果相同。在计算机领域，特别是**分布式系统**和**网络通信**中，它是一个关键的设计原则。
   
      - **核心定义**：对于同一个请求，无论被调用多少次，对系统的影响（状态变化）都与第一次调用相同，不会产生多余的副作用。
      - **关键目的**：主要为了应对**网络不确定性**（如延迟、超时、中断）和**用户重复操作**（如多次点击提交），确保系统在重试时不会出现重复创建订单、重复扣款等严重错误。
   
      **幂等性在HTTP方法中的体现**
   
      RESTful API 的设计直接利用了HTTP方法的天然幂等性来规范行为：
   
      | HTTP方法   | 幂等性 | 典型用途与说明                                               |
      | :--------- | :----- | :----------------------------------------------------------- |
      | **GET**    | **是** | 获取资源。多次请求同一URL，返回结果应相同，且不会改变服务器状态。 |
      | **PUT**    | **是** | 更新/替换资源。用客户端提供的完整数据替换服务器上的资源。多次用相同数据更新同一资源，最终状态一致。 |
      | **DELETE** | **是** | 删除资源。第一次删除后，资源不存在。后续相同请求通常会返回404，但不会产生其他影响。 |
      | **POST**   | **否** | 创建新资源或提交数据。每次请求可能会在服务器上创建一个新的、不同的资源（如新增一条订单）。 |
   
      






### LLM

核心函数：src/libs/llm/llm_factory.py

用来根据 settings.yaml/Settings 中的配置，创建并返回对应的 LLM 实例，实现“换模型/换后端只改配置不改代码”。

- 文本 LLM 创建：LLMFactory.create(settings, **override_kwargs)

  - 从 settings.llm.provider 读取 provider 名称

  - 在 _PROVIDERS 注册表里找到对应的 BaseLLM 子类并实例化

  - 支持用 override_kwargs 覆盖部分配置参数

  - provider 不存在或配置缺失时抛出明确异常

- 视觉（多模态）LLM 创建：LLMFactory.create_vision_llm(settings, **override_kwargs)

  - 优先读 settings.vision_llm.provider，否则回退到 settings.llm.provider

  - 在 _VISION_PROVIDERS 注册表里找到对应的 BaseVisionLLM 子类并实例化

  - provider 不存在或配置缺失时抛出明确异常

- 注册与枚举

  - register_provider() / list_providers()

  - register_vision_provider() / list_vision_providers()

- 模块加载时自动注册视觉 provider
  - _register_vision_providers() 在导入时运行，尝试注册 Azure/OpenAI/SiliconFlow 等 Vision LLM（未安装/未实现则跳过）。

测试：

BaseLLM + LLMFactory + 16个单元测试

```cmd
(.venv) PS F:\A_SCU\Code\MODULAR-RAG-MCP-SERVER\tests\unit> pytest -q .\test_llm_factory.py   

collected 16 items

test_llm_factory.py ................                                                               [100%]

========================================== 16 passed in 0.29s =========================================== 
```

****

### 文档加载器 Loader

核心函数：src/libs/loader/pdf_loader.py

把 PDF 文件解析成统一的 Document 对象，供后续摄取流程使用。

主要做了这些事：

- 用 MarkItDown 把 PDF 转成文本（尽量是 Markdown 结构）

- 校验输入文件是否存在、是否是 .pdf

- 计算文件哈希并生成文档 ID（如 doc_xxx）

- 组织标准元数据：source_path、doc_type、doc_hash，并尝试提取标题

- 可选启用图片处理（extract_images=True）：

- 用 PyMuPDF 提取图片并保存到 data/images/{doc_hash}/

- 在文本中插入 [IMAGE: image_id] 占位符

- 记录图片元数据到 Document.metadata["images"]

- 支持优雅降级：图片提取失败不会中断整体解析，仍返回可用的纯文本 Document

一句话：它是一个“PDF → 标准化文本+元数据（含可选图片引用）”的加载器。

测试：

BaseLoader + PdfLoader + PyMuPDF图片提取 + 21单元测试

```cmd
(.venv) PS F:\A_SCU\Code\MODULAR-RAG-MCP-SERVER> pytest -q .\tests\unit\test_base_loader_pdf_loader_unit.py
========================================= test session starts ========================================== 
platform win32 -- Python 3.11.7, pytest-9.0.3, pluggy-1.6.0
rootdir: F:\A_SCU\Code\MODULAR-RAG-MCP-SERVER
configfile: pyproject.toml
plugins: anyio-4.13.0, langsmith-0.7.29, asyncio-1.3.0, cov-7.1.0, mock-3.15.1
asyncio: mode=Mode.STRICT, debug=False, asyncio_default_fixture_loop_scope=None, asyncio_default_test_loop_scope=function
collected 21 items                                                                                       

tests\unit\test_base_loader_pdf_loader_unit.py .....................                              [100%] 

========================================== 21 passed in 0.99s ==========================================
```

****

### Splitter

核心函数：src/ingestion/chunking/document_chunker.py 

把一个 Document 切分成多个 Chunk，并在切分结果上做业务层增强：

- 调用底层 splitter（通过 SplitterFactory.create(settings) 创建）把 document.text 切成若干文本片段

- 为每个分块生成稳定且唯一的 ID：{doc_id}_{index:04d}_{sha256前8位}（同一文档同一切分结果可复现）

- 继承并扩展元数据：拷贝 document.metadata，添加

  - chunk_index：分块序号

  - source_ref：父文档 id（溯源）

  - image_refs：从 chunk 文本里解析 [IMAGE: xxx] 得到的图片引用 id 列表

  - images：如果 image_refs 命中 document.metadata["images"]，则带上被引用图片的完整元数据

  - page_num：尝试从第一张被引用图片的 page 推断页码

- 输出：返回 List[Chunk]（每个 Chunk 含 id/text/metadata）

测试：

BaseSplitter + SplitterFactory + 20个单元测试

```cmd
(.venv) PS F:\A_SCU\Code\MODULAR-RAG-MCP-SERVER\tests\unit> pytest -q test_splitter_factory.py

collected 20 items

test_splitter_factory.py ....................                                                     [100%]  

========================================== 20 passed in 0.30s ==========================================  
```

****

### Embedding 文本嵌入（稀疏+稠密）

核心函数：src/ingestion/embedding/batch_processor.py

功能可以概括为：

- 这是一个批处理编排器，负责把 Chunk 列表按 batch_size 分批。

- 对每个批次同时做两件事：

  1. 用 DenseEncoder 生成稠密向量（dense embeddings）

  2. 用 SparseEncoder 生成稀疏词项统计

- 汇总成 BatchResult 返回，包含：

  - dense_vectors

  - sparse_stats

  - batch_count

  - total_time

  - successful_chunks

  - failed_chunks

- 支持可选 trace 记录每批耗时、错误和总体统计，便于观测与排障。

- 具备容错：单个批次失败不会中断整个流程，会记录失败数并继续后续批次。

- 还提供两个辅助能力：

  - _create_batches()：按批切分并保持顺序

  - get_batch_count()：根据总 chunk 数计算需要多少批次。

测试：

```cmd
(.venv) PS F:\A_SCU\Code\MODULAR-RAG-MCP-SERVER> pytest tests/unit/test_batch_processor.py
================================ test session starts ================================
platform win32 -- Python 3.11.7, pytest-9.0.3, pluggy-1.6.0 -- F:\A_SCU\Code\MODULAR-RAG-MCP-SERVER\.venv\Scripts\python.exe
cachedir: .pytest_cache
rootdir: F:\A_SCU\Code\MODULAR-RAG-MCP-SERVER
configfile: pyproject.toml
plugins: anyio-4.13.0, langsmith-0.7.29, asyncio-1.3.0, cov-7.1.0, mock-3.15.1        
asyncio: mode=Mode.STRICT, debug=False, asyncio_default_fixture_loop_scope=None, asyncio_default_test_loop_scope=function
collected 20 items

tests/unit/test_batch_processor.py::test_batch_processor_initialization PASSED [  5%] 
tests/unit/test_batch_processor.py::test_batch_processor_initialization_with_default_batch_size PASSED [ 10%]
tests/unit/test_batch_processor.py::test_batch_processor_rejects_invalid_batch_size PASSED [ 15%]
tests/unit/test_batch_processor.py::test_create_batches_divides_evenly PASSED  [ 20%] 
tests/unit/test_batch_processor.py::test_create_batches_handles_remainder PASSED [ 25%]
tests/unit/test_batch_processor.py::test_create_batches_preserves_order PASSED [ 30%] 
tests/unit/test_batch_processor.py::test_create_batches_single_chunk PASSED    [ 35%]
tests/unit/test_batch_processor.py::test_get_batch_count PASSED                [ 40%] 
tests/unit/test_batch_processor.py::test_process_encodes_all_chunks PASSED     [ 45%] 
tests/unit/test_batch_processor.py::test_process_maintains_chunk_order PASSED  [ 50%] 
tests/unit/test_batch_processor.py::test_process_returns_correct_batch_count PASSED [ 
55%]
tests/unit/test_batch_processor.py::test_process_records_timing FAILED         [ 60%]
tests/unit/test_batch_processor.py::test_process_with_trace_records_batch_info PASSED 
[ 65%]
tests/unit/test_batch_processor.py::test_process_with_trace_records_individual_batches PASSED [ 70%]
tests/unit/test_batch_processor.py::test_process_rejects_empty_chunks PASSED   [ 75%] 
tests/unit/test_batch_processor.py::test_process_continues_on_batch_failure PASSED [ 80%]
tests/unit/test_batch_processor.py::test_process_records_batch_errors_to_trace PASSED 
[ 85%]
tests/unit/test_batch_processor.py::test_batch_result_structure PASSED         [ 90%] 
tests/unit/test_batch_processor.py::test_process_integration_with_encoders FAILED [ 95%]
tests/unit/test_batch_processor.py::test_process_deterministic_output PASSED   [100%]
```

****

### 向量存储

1. **bm25**

核心函数：src/ingestion/storage/bm25_indexer.py 

功能可以概括为：

- 负责构建、保存、加载、查询 BM25 倒排索引。

- 输入通常是 SparseEncoder 产出的 term_stats（含 chunk_id / term_frequencies / doc_length）。

- 在 build() 中会：

  1. 计算语料统计（文档数、平均长度）

  2. 计算每个词的 df 和 idf

  3. 生成倒排列表 postings

  4. 把索引和元数据写到磁盘（data/db/bm25）

- 在 query() 中按 BM25 公式给每个文档（chunk）打分，返回按分数降序的 top_k 结果。

- 支持：

  - load()：从磁盘恢复索引

  - rebuild()：语义化重建

  - add_documents()：增量合并新文档

  - remove_document()：按 doc_id 前缀删除并重算统计/IDF

- 还有若干私有辅助方法：_calculate_idf、_calculate_bm25_score、结构校验、索引路径和原子保存。

2. **图像存储**

核心函数：src/ingestion/storage/image_storage.py 

功能是做**图片文件存储 + SQLite 索引管理**，用于多模态 RAG 的图片资产管理。

简述如下：

- 存储层：把图片保存到文件系统（按 collection 分目录）。

- 索引层：在 SQLite 的 image_index 表里维护图片元数据（image_id、路径、collection、doc_hash、page_num、created_at）。 - **核心能力**： 

  save_image()：保存图片并登记索引（幂等，重复 image_id 会覆盖更新）。

  register_image()：不复制文件，只把已有图片登记进索引。 

  get_image_path() / image_exists()：按 image_id 查路径、判存在。 

  list_images()：按 collection / doc_hash 过滤列出图片。

  delete_image()：删索引，且可选同步删文件。

  get_collection_stats()：统计集合图片数量与总字节大小。 

  **初始化与可靠性**： 自动创建数据库与目录。 

  建索引（如 collection、doc_hash）提升查询效率。 

  使用 WAL 模式增强并发读写能力。

3. **向量增量更新**

核心函数：src/ingestion/storage/vector_upserter.py 

功能是：把分块文本及其 embedding 稳定地写入向量库。

简述如下：

- 接收 chunks + vectors，并做长度一致性校验。

- 为每个 chunk 生成确定性 chunk_id（基于 source_path、chunk_index、chunk.text 的哈希），保证幂等写入。

- 将数据组装成向量库记录（id / vector / metadata），再调用 VectorStoreFactory 创建的 vector_store.upsert() 写入。

- 提供 upsert_batch()，可把多个批次拍平后一次写入，减少往返。

- 失败时会抛出明确异常（如输入不合法、upsert 失败包装为 RuntimeError）。

一句话：它是 ingestion 阶段“向量落库”的统一入口，重点保障可重复、可追溯、可批量。

****

### LangChain LangGraph 相关工具接口

#### 切分器 *splitter*

文档分块：用于切分chunk的策略

##### LangChain

一、基础通用类（最常用）

```python
### 按字符列表递归尝试分割（如 \n\n → \n →   → 字符）
from langchain_text_splitters import RecursiveCharacterTextSplitter
# （递归字符切分器） #
RecursiveCharacterTextSplitter()


# 1. 定义切分器
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,       # 每个分块的最大字符数
    chunk_overlap=50,     # 相邻分块重叠的字符数
    separators=["\n\n", "\n", " ", ""]  # 优先尝试的分隔符列表
)

from langchain_text_splitters import CharacterTextSplitter
# （字符切分器） #
# *最简单的切分器，只按指定的单一分隔符硬切。* #
CharacterTextSplitter()

text = "苹果\n香蕉\n西瓜\n葡萄\n橘子"

# 初始化：严格按照 \n 切分，不设置 chunk_size
splitter = CharacterTextSplitter(separator="\n", chunk_size=100, chunk_overlap=0)
```

二、 结构感知类

```python
# （Markdown 标题切分器） #
# *按标题切分，并将标题层级作为元数据保留。* #
MarkdownHeaderTextSplitter()

from langchain_text_splitters import MarkdownHeaderTextSplitter
markdown_document = """
# 第一章：RAG系统介绍
RAG（检索增强生成）是当前大模型落地的核心技术。

## 1.1 什么是RAG？
RAG结合了信息检索和文本生成。
"""

# 定义要提取的标题级别
headers_to_split_on = [
    ("#", "Header 1"),
    ("##", "Header 2"),
]

splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers_to_split_on)

# 注意：split_text 返回的是 Document 对象列表
docs = splitter.split_text(markdown_document)

for doc in docs:
    print(f"内容: {doc.page_content[:20]}...")
    print(f"元数据: {doc.metadata}\n")
# 输出示例：元数据: {'Header 1': '第一章：RAG系统介绍', 'Header 2': '1.1 什么是RAG？'}

#-------------------------------------------------------------------------------------#

#（HTML 标题切分器）#
# *提取 HTML 正文，并根据 h1, h2 等标签分配元数据。* #
HTMLHeaderTextSplitter()

from langchain_text_splitters import HTMLHeaderTextSplitter

html_string = """
<html>
<body>
    <h1>公司简介</h1>
    <p>我们是一家专注于AI的科技公司。</p>
</body>
</html>
"""

headers_to_split_on = [
    ("h1", "Header 1"),
    ("h2", "Header 2"),
]

splitter = HTMLHeaderTextSplitter(headers_to_split_on=headers_to_split_on)
docs = splitter.split_text(html_string)


#-------------------------------------------------------------------------------------#

#（Python 代码切分器）#
# *按类、函数等代码结构切分，防止逻辑被截断。* # 

PythonCodeTextSplitter()

python_doc = '''
class DataProcessor:
    def __init__(self, data):
        self.data = data

    def clean(self):
        # 清洗数据逻辑
        pass
'''
code_splitter = PythonCodeTextSplitter(chunk_size = 100, chunk_overlap = 10)
chunks = code_splitter(python_doc)

for chunk in chunks:
    print(chunk)
```

三、 模型对齐类

```python
#（Token 切分器）#
# 使用 tiktoken 计算 Token 数，严格确保不超过大模型的 Token 限制。#
TokenTextSplitter()

from langchain_text_splitters import TokenTextSplitter

text = "这是一段用来测试 Token 切分的文本。大模型并不是按字符来计算费用的，而是按照 Token。有时候一个中文字可能对应多个 Token。"

# 初始化 Token 切分器
splitter = TokenTextSplitter(
    chunk_size=20,       # 每块最多 20 个 Token
    chunk_overlap=5      # 重叠 5 个 Token
)

chunks = splitter.split_text(text)

#-------------------------------------------------------------------------------------#


# 针对特定 Embedding 模型（如 all-MiniLM-L6-v2，限制 256 tokens）进行严格切分。 #
SentenceTransformersTokenTextSplitter()

from langchain_text_splitters import SentenceTransformersTokenTextSplitter

text = "LangChain 是一个强大的框架。" * 50 # 制造一段长文本

# 指定目标模型的 Token 限制
splitter = SentenceTransformersTokenTextSplitter(
    tokens_per_chunk=50,             # 目标模型的单块上限
    chunk_overlap=10
)

chunks = splitter.split_text(text)

#-------------------------------------------------------------------------------------#
```

四、 自然语言处理类

```python
# （语义切割） #
# *利用 Spacy 强大的 NLP 模型进行最高质量的语义切分。* #
SpacyTextSplitter()

from langchain_text_splitters import SpacyTextSplitter

text = "北京市是中国的首都。上海是经济中心。深圳被称为硅谷。杭州以西湖闻名。这些城市各有特色。"

# 初始化时需指定 spacy 的语言模型（需提前 pip install 并 python -m spacy download）
splitter = SpacyTextSplitter(
    pipeline="zh_core_web_sm", # 使用中文模型
    chunk_size=20, 
    chunk_overlap=0
)

chunks = splitter.split_text(text)
```

终极技巧：组合使用（Pipeline）
在实际项目中，我们通常会将 结构切分 和 长度切分 组合起来。

下面是如何将 **MarkdownHeaderTextSplitter** 和 **RecursiveCharacterTextSplitter** 组合使用的标准写法：

```python
from langchain_text_splitters import MarkdownHeaderTextSplitter, RecursiveCharacterTextSplitter

md_text = """
# 项目介绍
这是一个长篇的项目介绍文档，里面包含了很多细节内容...
（此处省略几百字）
## 技术栈
后端使用了 Python 和 FastAPI，前端使用了 React...
（此处省略几百字）
"""

# 第一步：按 Markdown 结构切，提取元数据
md_splitter = MarkdownHeaderTextSplitter(headers_to_split_on=[("#", "h1"), ("##", "h2")])
raw_docs = md_splitter.split_text(md_text)

# 第二步：对切出来的长块，再按字符长度进行二次切分
text_splitter = RecursiveCharacterTextSplitter(chunk_size=50, chunk_overlap=10)

# 遍历第一步的结果，进行第二步切分
final_chunks = []
for doc in raw_docs:
    sub_chunks = text_splitter.split_documents([doc])
    final_chunks.extend(sub_chunks)

# 检查结果：你会发现小的块依然保留着第一步切出来的 h1 和 h2 元数据！
for chunk in final_chunks:
    print(f"内容: {chunk.page_content}")
    print(f"继承的元数据: {chunk.metadata}\n")
```

##### LangGraph

在 LangGraph 构建的 RAG（检索增强生成）系统中，**文档切分器是“索引管道”中至关重要的第一个处理节点**，其职责是将大型文档拆分成适合向量化和检索的、更小的语义单元。

LangGraph 本身是一个用于构建**有状态、多步骤应用**的框架，它并不直接提供新的切分器实现。相反，它无缝集成了 **LangChain 生态中成熟的文本切分器**。在 LangGraph 中，切分操作通常被封装为一个独立的 **“节点”**，与其他节点（如检索、生成）一起定义工作流。

****

##### 总结

| 切分器                             |                        核心逻辑                         | 测量单位 | 优点                       | 缺点                             |                   推荐场景                   |
| :--------------------------------- | :-----------------------------------------------------: | :------: | :------------------------- | :------------------------------- | :------------------------------------------: |
| **RecursiveCharacterTextSplitter** | 按字符列表递归尝试分割（如 `\n\n` → `\n` → ` ` → 字符） |  字符数  | 灵活、智能，能较好保持语义 | 对复杂排版（如列表）识别能力有限 |       **绝大多数通用文本**，是默认首选       |
| **CharacterTextSplitter**          |   按指定的**单个字符**（如 `\n\n`）硬性切割csdn.net+1   |  字符数  | 简单直接，性能高           | 容易切断语义，产生超大块         | 简单日志、配置文件等对语义完整性要求低的场景 |

| 切分器                         | 核心逻辑                             | 优点                                     | 缺点                             | 推荐场景                           |
| :----------------------------- | :----------------------------------- | :--------------------------------------- | :------------------------------- | :--------------------------------- |
| **MarkdownHeaderTextSplitter** | 按Markdown标题（`#`, `##`）分割      | 保留文档结构，将标题作为元数据传递给分块 | 不直接控制块大小，可能产生超大块 | **Markdown技术文档、博客、知识库** |
| **PythonCodeTextSplitter**     | 针对Python代码，按函数、类等结构分割 | 保证代码逻辑单元完整性                   | 仅适用于Python                   | **源码分析、Code Review**          |
| **HTMLHeaderTextSplitter**     | 按HTML的标题标签分割                 | 保留网页语义结构                         | 依赖HTML结构质量                 | 网页内容提取                       |

|                  切分器                   |                    核心逻辑                    | 测量单位 |                  优点                  |           缺点           |                   推荐场景                   |
| :---------------------------------------: | :--------------------------------------------: | :------: | :------------------------------------: | :----------------------: | :------------------------------------------: |
|           **TokenTextSplitter**           | 使用 `tiktoken` 等库，按LLM的**Token**计算切分 | Token数  | **绝对保证**不超过模型的上下文窗口限制 | 人类不可读，解析速度较慢 |       接入API时**控制成本**、避免溢出        |
| **SentenceTransformersTokenTextSplitter** |  专为特定Embedding模型设计，按其Token限制切分  | Token数  |     与向量模型完美契合，保证可索引     |         速度稍慢         | 使用有严格Token限制的**特定Embedding模型**时 |

|        切分器         | 核心逻辑                          | 优点                       | 缺点                     | 推荐场景                                       |
| :-------------------: | :-------------------------------- | :------------------------- | :----------------------- | :--------------------------------------------- |
| **NLTKTextSplitter**  | 调用NLTK库，按自然语言“句子”分割  | 准确识别句子边界           | 需要额外下载NLTK数据     | 长篇小说、论文等需要保持句子完整的文本csdn.net |
| **SpacyTextSplitter** | 使用Spacy NLP模型进行更精准的切分 | 语义理解最强，切分质量极高 | 性能开销大，需要加载模型 | 对切分准确度要求极高的复杂文本                 |

****

#### LLM大模型连接与通信

**LLM大模型连接与通信**
在构建 AI 应用时，与底层大模型（如 GPT-4、Claude、Qwen 等）进行稳定、规范的连接与通信是第一步。

LangChain 和 LangGraph 在这方面扮演着不同的角色：

LangChain 提供了标准化的连接器，而 LangGraph 则定义了在复杂流程中如何精准地调度这些连接。

**主要有两种方式**

1. ChatOpenAI( )

   ```python
   from langchain_openai import ChatOpenAI
   from langchain_core.messages import SystemMessage, HumanMessage
   import dotenv
   import dotenv
   import os
   
   dotenv.load_dotenv()
   chat_model = ChatOpenAI(
       model="gpt-3.5-turbo",
       api_key=os.getenv("openai_api_key_chat"),
       base_url=os.getenv("openai_base_url_chat"),
       temperature=0.8,
       max_tokens=1000
   )
   system_message = SystemMessage(
       content="你是一个AI助手，请根据用户的问题给出回答。",
       additional_kwargs={"tool":"inv"})
   human_message = HumanMessage(content="什么是langchain",)
   messages = [system_message, human_message]
   
   response = chat_model.invoke(messages)
   print(type(response)) # AIMessage
   print(response.content)
   ```

2. init_chat_model( )

   ```python
   from langchain.chat_models import init_chat_model
   model = init_chat_model(
       "Qwen/Qwen3-8B",
       model_provider="openai",
       base_url = "https://api.siliconflow.cn/v1",
       api_key = "sk-frhnxxxxxxxxxxxxxxxxxxxxxjtmckx"
   )
   ```

**四种通信方式**

在通过 LangChain 建立连接后，我们如何让模型干活？LangChain 提供了四个核心动词：`invoke`、`stream`、`astream`、`batch`。它们的区别在于**阻塞与否**以及**数据返回的粒度**。

假设我们向模型提问：“写一篇500字的作文”。以下是四种通信方式的直观对比：

| 通信方式      | 中文释义 | 阻塞/异步      | 用户体验                                                     | 适用场景                                             |
| :------------ | :------- | :------------- | :----------------------------------------------------------- | :--------------------------------------------------- |
| **`invoke`**  | 同步调用 | 🔴 阻塞         | **“憋大招”**：等几十秒完全没反应，然后瞬间输出全部500字。    | 后台脚本、数据处理流水线、不需要直接面向用户的场景。 |
| **`stream`**  | 同步流式 | 🔴 阻塞(生成器) | **“打字机”**：一个字一个字往外蹦，但有感知。                 | 简单的命令行工具、Jupyter Notebook 测试。            |
| **`astream`** | 异步流式 | 🟢 非阻塞       | **“极速打字机”**：一边蹦字，程序还能同时处理其他任务（如接收用户中断指令）。 | **Web 后端、高并发 API、最核心的生产环境首选。**     |
| **`batch`**   | 批量调用 | 🔴 阻塞         | 并发处理多个独立任务，全做完后一起返回。                     | 离线数据打标、一次性翻译1000条句子。                 |

```python
from langchain_core.messages import SystemMessage, HumanMessage

# 统一的提问
essay_messages = [
    SystemMessage(content="你是一位中文写作老师，文章结构清晰，语言自然。"),
    HumanMessage(content="写一篇500字的作文，题目《春天的校园》。")
]

# 1) invoke：同步 + 一次性返回完整结果
resp = llm.invoke(essay_messages)
print(resp.content)

# 2) stream：同步 + 分块返回（边生成边输出）
for chunk in llm.stream(essay_messages):
    if chunk.content:
        print(chunk.content, end="", flush=True)
print()  # 换行

# 3) astream：异步 + 分块返回（适合并发/异步服务）
# 说明：有些版本/模型返回的 chunk 里，content 可能阶段性为空，
#      因此不要用 `if chunk.content:` 过早过滤；先尽量打印/累积。

async def run_astream():
    parts = []
    async for chunk in llm.astream(essay_messages):
        content = getattr(chunk, "content", None)

        # content 通常是 str；少数情况下可能是 list/None
        if isinstance(content, str):
            if content:
                print(content, end="", flush=True)
            parts.append(content)
        elif content is not None:
            # 兜底：直接打印非字符串内容，避免“看起来没输出”
            print(str(content), end="", flush=True)
            parts.append(str(content))
        else:
            # 再兜底：把 chunk 的结构打出来，方便定位
            # （只打很短的一行，避免刷屏）
            parts.append("")

    final_text = "".join([p for p in parts if isinstance(p, str)])
    if final_text.strip() == "":
        print("\n[debug] astream 收到的 chunk.content 全为空；请改用下面的 ainvoke/stream 或检查依赖版本。")
    else:
        print()  # 换行

await run_astream()

# 4) batch：同步 + 批量输入，一次性返回多个结果
batch_inputs = [
    essay_messages,
    [
        SystemMessage(content="你是一位中文写作老师，文章结构清晰，语言自然。"),
        HumanMessage(content="写一篇500字的作文，题目《雨后的城市》。")
    ],
]

batch_resps = llm.batch(batch_inputs)
for i, r in enumerate(batch_resps, start=1):
    print(f"\n--- 第{i}篇 ---")
    print(r.content) 
#--- 第1篇 ---
#《春天的校园》
#    ....
#--- 第2篇 ---
#题目：《雨后的城市》
#    ....
```

##### 消息类型

- 标准消息类型
  - `SystemMessage`：设定 AI 的人设和全局规则。
  - `HumanMessage`：用户的输入。
  - `AIMessage`：模型的历史回复（在多轮对话中至关重要）。

|     类型     | LangChain 类名  |        角色定位         | 底层对应的 API 标识 |                           核心作用                           |
| :----------: | :-------------: | :---------------------: | :-----------------: | :----------------------------------------------------------: |
| **系统消息** | `SystemMessage` | **幕后导演 / 人设设定** |     `"system"`      | 最高优先级。设定 AI 的行为准则、性格、全局规则，模型通常不会在回复中直接提及它。 |
| **人类消息** | `HumanMessage`  |    **用户 / 提问者**    |      `"user"`       |  触发模型生成的源头。包含用户的提问、指令或需要处理的数据。  |
| **AI 消息**  |   `AIMessage`   |    **助手 / 回答者**    |    `"assistant"`    | 记录模型历史生成的文本。在**多轮对话**中，必须把 AI 之前的回答作为 `AIMessage` 喂给它，它才知道自己刚才说了什么。 |

**工具消息**：`ToolMessage`（工具消息）是 Agent（智能体）架构中**不可或缺的“结果回执”**。

`ToolMessage` 就是代码环境写给大模型的“工作汇报”，它的存在补齐了“模型发令 -> 代码执行 -> 结果反馈 -> 模型总结”的闭环，没有它，Agent 就是个只会发号施令却得不到反馈的瞎子。

```python
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage
```

##### LangGraph

既有langchain的messages机制，在构建状态图中主要使用自定义的State来进行消息的流转

LangGraph 本身**不负责**创建大模型连接（它直接复用 LangChain 的 `ChatOpenAI` 等对象）。但是，在 LangGraph 构建的多步骤 Agent 工作流中，大模型的通信方式发生了本质变化：

1. **状态驱动通信**：模型不再孤立地被调用，它必须读取当前图的 `State`（状态），处理后将结果写回 `State`。
2. **工具调用通信**：在 Agent 场景中，与大模型通信不仅是为了获取文本，更关键的是获取它想要调用的“工具指令”（如搜索数据库）。
3. **基于条件的循环通信**：如果模型返回“需要调用工具”，Graph 会拦截这个响应，跳转到工具节点执行，然后再把工具结果作为新消息发给模型（形成通信循环），直到模型给出最终答案。

##### 总结

| 维度         | LangChain 层面                                 | LangGraph 层面                                               |
| :----------- | :--------------------------------------------- | :----------------------------------------------------------- |
| **职责**     | 建立 TCP/HTTP 连接，处理鉴权、流式、异常重试。 | 将大模型调用编排为有状态图中的一个节点。                     |
| **输入来源** | 开发者手动传入的 `messages` 列表。             | 从图的统一 `State` 中自动读取上下文。                        |
| **输出去向** | 直接返回给 Python 变量。                       | 必须返回特定格式的字典，用于更新图的 `State`。               |
| **通信模式** | 通常是“一问一答”的单次通信。                   | 支持“模型调用工具 -> 观察结果 -> 再次调用模型”的**循环通信**。 |

****

#### Prompt提示词模板

##### Langchain

在 LangChain 中，直接把一长串字符串写死在代码里是非常糟糕的做法。

**提示词模板**的作用就是**“分离结构与人设（模板）和动态数据（变量）”**。

**一、 最基础的模板（输出纯字符串）**

1. PromptTemplate（字符串提示模板）

- **概念**：最基础的模板，用来生成一段**纯粹的字符串**。它不区分什么系统、用户角色，最终拼接出来的结果就是一个大字符串。
- **适用场景**：简单的单轮问答、文本摘要、翻译、格式化输出（如让模型输出 JSON）。
- **特点**：使用 `{变量名}` 作为占位符。

```python
from langchain_core.prompts import PromptTemplate

template = "请把下面的{lang}翻译成中文：{text}"
prompt = PromptTemplate.from_template(template)
# 调用: prompt.format(lang="英文", text="Hello world")
# 结果: "请把下面的英文翻译成中文：Hello world"

```

**二、 样本增强模板（提升准确率）**

2. FewShotPromptTemplate（少样本提示词模板）

- **概念**：大模型有“举一反三”的能力。这个模板专门用来包装**“几个优秀的问答示例”**，然后再附上用户真正要问的问题。
- **适用场景**：需要严格控制输出格式（如自定义情感分析标签、特定的结构化数据提取），但不想微调模型时。
- **结构**：通常包含一个 `PromptTemplate`（作为最终提问的格式）+ 一个 `examples` 列表（作为示范）。

```python
# 内部逻辑类似：
# 示例1：输入:"苹果很好吃" -> 输出:"正向"
# 示例2："这电影太烂了" -> 输出："负向"
# 现在请判断：{user_input}
```

**三、 角色扮演模板（贴合 Chat API）**

现代大模型（如 GPT）是基于聊天优化的，需要明确区分谁在说话。**3~7 号模板都是为了生成前面讲过的 `Message` 对象列表**。

3. ChatPromptTemplate（聊天提示模板 - 🌟 最常用）

- **概念**：大管家。它不直接写字符串，而是负责把下面 4、5、6 号模板组装成一个完整的“消息列表”。
- **适用场景**：99% 的多轮对话、Agent 开发、RAG 检索增强。

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}，只用{style}回答。"), # 隐式调用了 SystemMessagePromptTemplate
    ("user", "{question}")                          # 隐式调用了 HumanMessagePromptTemplate
])
# 调用后会直接生成: [SystemMessage(...), HumanMessage(...)]

```

4. HumanMessagePromptTemplate（人类消息模板）

- **概念**：专门生成 `HumanMessage` 的模板。
- **注意**：现在**很少单独使用它**了，因为在 `ChatPromptTemplate.from_messages()` 中直接写 `("user", "...")` 就等于隐式调用了它。

5. AIMessagePromptTemplate（AI 消息模板）

- **概念**：专门生成 `AIMessage` 的模板。
- **使用场景**：在**多轮对话历史恢复**时使用。比如你想在提问前，伪造一段“你之前问了什么，AI 答了什么”的历史上下文塞给模型。

```python
# 假设你要构建包含历史的 prompt
history = AIMessagePromptTemplate.from_template("我刚才查到了结果：{result}")
```

6. SystemMessagePromptTemplate（系统消息模板）

- **概念**：专门生成 `SystemMessage` 的模板。
- **注意**：同上，通常被 `("system", "...")` 元组语法取代，很少单独实例化。

7. ChatMessagePromptTemplate（自定义角色消息模板）

- **概念**：用来生成 `ChatMessage` 的模板。它比上面几个多了一个 `role` 参数。
- **使用场景**：极少数非标准的第三方模型，它们不支持 system/user/assistant，而是支持自定义角色（比如 role=“bartender” 酒保，role=“interviewer” 面试官）。

```python
from langchain.prompts import ChatMessagePromptTemplate
# 指定一个非标准的自定义角色
prompt = ChatMessagePromptTemplate.from_template(role="兽医", template="作为一名{animal_type}医生，{question}")
```

**四、 复杂工程化模板**

8. PipelinePrompt（管道提示词模板）

- **概念**：类似于 Linux 的管道符 `|`。当你的系统提示词非常长（比如包含几百行的规则文档、各种 FewShot 示例），写在一个文件里没法维护。`PipelinePrompt` 允许你把大模板**拆分成多个小块（部分）**，最后像拼积木一样拼起来。
- **适用场景**：超大型企业级应用的 Prompt 管理。

```python
# 伪代码逻辑
final_prompt = PipelinePrompt(
    final_prompt=PromptTemplate.from_template("{introduction}\n{examples}\n{user_question}"),
    pipeline_prompts=[
        ("introduction", intro_template),  # 拼接块1：公司背景介绍
        ("examples", few_shot_template),   # 拼接块2：示例库
        ("user_question", user_template)   # 拼接块3：用户真实问题
    ]
)
```

9. 自定义模板

- **概念**：LangChain 默认使用 Python 的字符串 `format()` 方法来填充 `{变量}`。

****

#### 输出解析器

##### 🔥 常用解析器

1. StrOutputParser（字符串解析器）

最基础的解析器，只做一件事：把 `AIMessage` 对象剥壳，提取出里面的纯文本字符串。日常开发中用得最多，几乎每条 LCEL 链都会用到它。

```python
# StrOutputParser：字符串解析器
from langchain_core.output_parsers import StrOutputParser

parser = StrOutputParser()
str_response = parser.invoke(response)
print(str_response)
print(type(str_response)) # <class 'str'>
```

2. JsonOutputParser（JSON 解析器）

把模型输出的 JSON 字符串解析成 Python 字典。**前提是你必须在 Prompt 中明确告诉模型"以 JSON 格式返回"**，否则模型可能夹杂一堆废话导致解析失败。结构化数据提取的首选。

```python
# JsonOutputParser：JSON解析器，确保输出符合特定JSON对象格式，在提示词中必须指定返回的数据格式为json
from langchain_core.output_parsers import JsonOutputParser

parser = JsonOutputParser()
print("parser.get_format_instructions():",parser.get_format_instructions()) # 可将parser.get_format_instructions()复制到提示词中
json_response = parser.invoke(response)
print(json_response)
print(type(json_response)) # <class 'dict'>
# # 链：将多个组件组合在一起，并自动处理输入和输出
chain = chat_model_stream | parser
json_result = chain.invoke("请用json格式返回你的答案，你的答案是：1+1=?")
print(json_result)
```

3. XMLOutputParser（XML 解析器）

让模型以 XML 格式输出，然后**自动解析成 Python 字典**返回给你（不是原始 XML 字符串）。同样需要在 Prompt 中指定返回 XML 格式。适合处理层级嵌套较深的数据结构。

```python
parser = XMLOutputParser()
format_instructions = XMLOutputParser().get_format_instructions()
prompt = PromptTemplate.from_template(
    template = "用户的问题：{question},需要使用的格式：{format_instructions}",
    # input_variables = ["question"], # 会报错，因为input_variables和partial_variables不能同时使用
    partial_variables = {"format_instructions": format_instructions}
)
response = chat.invoke(prompt.format(question="什么是langchain?"))
print(response.content)
print(type(response))
xml_result = parser.invoke(response)
print("xml_result:")
print(xml_result)
print(type(xml_result))
```

4. CommaSeparatedListOutputParser（逗号分隔列表解析器）

让模型以逗号分隔输出内容，解析后返回 Python 列表。比如让模型"列举三种编程语言"，返回的就是 `["Python", "Java", "Go"]`。列举类任务的利器。

5. DatetimeOutputParser（日期时间解析器）

专门处理日期时间。模型经常输出"大概下周三"或"2024年3月15号下午"这种口语化表达，这个解析器能将其统一转换为标准的 `datetime` 对象。

⚡ 特殊场景解析器

6. EnumOutputParser（枚举解析器）

预先定义好一组合法选项（如"正面/负面/中性"），模型输出后强制映射到其中某个枚举值。输出不在选项内就报错。**适合情感分析、分类任务**，杜绝模型自由发挥。

7. StructuredOutputParser（结构化解析器）

老版本的 JSON 解析器，把非结构化文本转换为预定义的字典结构。功能与 `JsonOutputParser` 类似，但现在官方更推荐用 `JsonOutputParser`，属于逐步被替代的角色。

🛠️ 自动修复型解析器（最巧妙的设计）

这两个解析器解决的是同一个痛点：**模型输出的格式不对怎么办？**

8. OutputFixingParser（输出修复解析器） 

模型返回的 JSON 缺了个括号怎么办？这个解析器会尝试**自动修补**常见的格式错误（少引号、少逗号、多出注释等），尽量抢救回来，避免直接报错。（langchain 0.3.14 具有这个特性）

```python
# from langchain.output_parsers.fix import OutputFixingParser  
from langchain_core.exceptions import OutputParserException
```

9. RetryOutputParser（重试解析器）

比 OutputFixingParser 更暴力——**直接把错误的输出发回给 LLM**，告诉它"你刚才的回答格式不对，请修正后重新输出"，然后用修正后的结果再次尝试解析。相当于让模型自己给自己打补丁。（langchain 0.3.14 具有这个特性）

```python
from langchain.output_parsers.retry import RetryOutputParser
```

```bash
模型输出（自由文本）
    │
    ├─ 不需要结构化 → StrOutputParser → 纯字符串
    │
    ├─ 需要列表     → CommaSeparatedListOutputParser → ["a", "b", "c"]
    │
    ├─ 需要日期     → DatetimeOutputParser → datetime对象
    │
    ├─ 需要分类     → EnumOutputParser → 枚举值（正面/负面/中性）
    │
    ├─ 需要JSON     → JsonOutputParser → Python字典
    │
    ├─ JSON格式坏了 → OutputFixingParser → 自动修补 → 再解析
    │                RetryOutputParser   → 扔回给LLM重答 → 再解析
    │
    └─ 需要XML      → XMLOutputParser → Python字典

```

##### 总结

| #    | 解析器                     | 输入示例                                   | 输出                                        | 类型       |
| :--- | :------------------------- | :----------------------------------------- | :------------------------------------------ | :--------- |
| 1    | **StrOutputParser**        | `AIMessage("你好，我是AI助手")`            | `"你好，我是AI助手"`                        | `str`      |
| 2    | **JsonOutputParser**       | `json\n{"name":"张三"}\n`                  | `{'name': '张三'}`                          | `dict`     |
| 3    | **XMLOutputParser**        | `<person><name>张三</name></person>`       | `{'name': '张三'}`                          | `dict`     |
| 4    | **CommaSeparatedList**     | `"Python, Java, Go, Rust"`                 | `['Python', 'Java', 'Go', 'Rust']`          | `list`     |
| 5    | **DatetimeOutputParser**   | `"2024年3月15号下午3点30分"`               | `2024-03-15 15:30:00`                       | `datetime` |
| 6    | **EnumOutputParser**       | `"正面"`                                   | `Sentiment.POSITIVE`                        | `Enum`     |
| 7    | **StructuredOutputParser** | `电影名称：星际穿越\n评分：9.5`            | `{'movie': '星际穿越', 'rating': 9.5}`      | `dict`     |
| 8    | **OutputFixingParser**     | `{"name":"张三", skills: [...]}`（缺引号） | 自动修补 → `{'name':'张三','skills':[...]}` | `dict`     |
| 9    | **RetryOutputParser**      | `"好的，张三的信息如下：..."`（非JSON）    | 发回LLM重答 → `{'name':'张三'}`             | `dict`     |

****

#### 文档加载器 Loader

在 LangChain 生态中，**文档加载器** 的核心使命只有一个：**将各种五花八门的外部数据源（PDF、网页、数据库、代码等），统一转换为 LangChain 能够理解的标准化格式 —— `Document` 对象。**

一个标准的 `Document` 对象包含两个核心属性：

- `page_content` (str)：提取出的纯文本内容。
- `metadata` (dict)：关于这段文本的元数据（如来源 URL、页码、作者、文件名等）。

##### 一、 LangChain 中的文档加载器

1. 传统文件加载器 (本地文件)

用于解析本地磁盘上的文件。

加载最常见的结构化和半结构化文本文件

| 加载器             | 主要用途与特点                                              | 安装依赖              |
| :----------------- | :---------------------------------------------------------- | :-------------------- |
| **`TextLoader()`** | 加载**纯文本文件**（如 `.txt`，也可用于 `.md`）。           | `langchain-community` |
| **`CSVLoader()`**  | 加载 **CSV 表格文件**。支持自定义分隔符、引号字符和列名。   | `langchain-community` |
| **`JSONLoader()`** | 加载 **JSON 文件**。可以通过 `jq_schema` 参数提取特定字段。 | `jq`                  |

用于加载常见的办公文档，是构建知识库的核心

| 加载器                                 | 主要用途与特点                                               | 安装依赖                      |
| :------------------------------------- | :----------------------------------------------------------- | :---------------------------- |
| **`PyPDFLoader()`**                    | 最常用的 **PDF 加载器**。使用 `pypdf` 库逐页提取文本内容。   | `pypdf`                       |
| **`UnstructuredWordDocumentLoader()`** | 加载 **Word 文档** (`.docx`)。底层使用 `unstructured` 库进行解析。 | `unstructured`, `python-docx` |
| **`UnstructuredExcelLoader()`**        | 加载 **Excel 表格** (`.xlsx`)。                              | `unstructured`, `openpyxl`    |
| **`UnstructuredPowerPointLoader()`**   | 加载 **PowerPoint 演示文稿** (`.pptx`)。                     | `unstructured`, `python-pptx` |

当需要处理大量或多种类型的文件时，以下加载器非常高效

| 加载器                         | 主要用途与特点                                               |
| :----------------------------- | :----------------------------------------------------------- |
| **`DirectoryLoader()`**        | **批量加载**指定目录下的文件。支持通配符（如 `glob="**/*.pdf"`）匹配，并可结合多线程加速。 |
| **`UnstructuredFileLoader()`** | **智能加载器**，能自动检测文件格式并调用相应的解析器。       |

部分代码示例：

```python
# 文档加载示例

from langchain_community.document_loaders import (
    PyPDFLoader,
    TextLoader,
    Docx2txtLoader,
    DirectoryLoader, # 批量加载
)
# 加载pdf
pdf_loader = PyPDFLoader("./02-load.pdf")
pdf_docs = pdf_loader.load()

print(f"加载了 {len(pdf_docs)} documents")
print("第一页的内容预览：")
print(pdf_docs[0].page_content[:100])
print("Documents结构：")
print(pdf_docs)

# # 加载word
# docx_loader = Docx2txtLoader("./01-load.docx")
# docx_docs = docx_loader.load()

# # 加载txt
# txt_loader = TextLoader("./test.txt",encoding="utf-8")
# txt_docs = txt_loader.load()
# print(f"加载了 {len(txt_docs)} documents")
# print("第一页的内容预览：")
# print(txt_docs[0].page_content[:100])

# # 加载目录下的所有文件
# dir_loader = DirectoryLoader(
#     "./",
#     glob = "**/*.pdf", # 匹配所有pdf文件
#     loader_cls = PyPDFLoader
# )
# all_docs = dir_loader.load()
# print(f"加载了 {len(all_docs)} documents")
# print("第一页的内容预览：")
# print(all_docs[0].page_content[:100])

# Documents 对象结构
from langchain_core.documents import Document
doc = Document(
    page_content = "这是文档内容",
    metadata = {
        "source": "test.txt",
        "page": 1,
        "author": "LangChain",
        "date": "2023-01-01"
    }
)
```



2. 网络与云服务加载器 (在线数据)

```python
from langchain_community.document_loaders import WebBaseLoader
import asyncio

# 加载网页（底层通常使用 httpx 或 BeautifulSoup）
web_loader = WebBaseLoader("https://python.langchain.com/docs/")
web_docs = web_loader.load()
# web_docs[0].metadata = {'source': 'https://...', 'title': '...'}

```

3. 高级变体：懒加载 与 异步加载

当处理几百个 PDF 或大量网页时，一次性 `.load()` 全部放进内存会撑爆 RAM。

- **`.lazy_load()`**：返回一个生成器，一次只 yield 一个 Document。
- **`.aload()`**：异步版本，配合 `asyncio` 并发抓取几十个网页，速度提升数倍。

##### 二、 LangGraph 中的文档加载器（节点中的工具）

**首先澄清一个概念：LangGraph 本身没有、也不需要发明新的文档加载器。** LangGraph 直接复用 LangChain 的所有加载器。

但在 LangGraph 的**状态图**架构中，文档加载器的定位发生了变化：

- 在 LangChain 中，它通常是**写在脚本最开头**的流水线代码。
- 在 LangGraph 中，它被封装成一个**图节点**，成为自动化工作流中的一个可控制、可分支、可重试的步骤。

##### 三、llamaIndex文档加载器

LlamaIndex 加载器的核心设计哲学是：**一切为了构建高效的 Node（节点）树。** 它不仅仅满足于把文件变成字符串，而是强调整合元数据，为后续的切片和向量索引打下完美基础。

在 LlamaIndex 中，加载器统一被称为 `Reader`。

**LlamaIndex 的两大数据容器概念：**

在看加载器之前，必须先理解 LlamaIndex 独有的数据结构（这与 LangChain 的 `Document` 不同）：

1. **`Document`**：代表整个原始文件（比如一整篇 PDF、一个完整的网页）。它包含 `text` (全文) 和 `metadata` (文件级元数据，如文件名、作者)。
2. **`TextNode`**：代表从 `Document` 中切分出来的**文本块**（Chunk）。它不仅有 `text`，还有 `metadata`（块级元数据，如属于第几页、在原文的起止位置），以及**关系指针**（指向它的父 Document 和前后相邻的 Node）。*（注意：LlamaIndex 的加载器直接产出 `Document`，切片后才产出 `TextNode`）*。

**核心加载器分类与代码示例：**

1. 简单文本与标记语言

对于 `.txt` 和 `.md`，LlamaIndex 提供了最轻量级的原生加载器。

```python
from llama_index.core import SimpleDirectoryReader

# LlamaIndex 的杀手锏：自动根据文件后缀名派发对应的 Reader！
# 它会自动识别 .txt, .md, .pdf, .csv 等，无需你手动选择类
documents = SimpleDirectoryReader("./data_folder").load_data()

print(f"加载了 {len(documents)} 个完整文档")
print(documents[0].metadata) # 通常包含 'file_name', 'file_path'

```

如果你想**手动指定**加载单个 Markdown 并保留其结构元数据：

```python
from llama_index.readers.file import MarkdownReader

reader = MarkdownReader()
docs = reader.load_data(file_path="./example.md")
```

2. 网页与在线文档加载器

LlamaIndex 把第三方数据源的加载器统称为 `LlamaHub`。现在它们都被整合到了 `llama-index-readers-*` 的命名空间下。

```
# 需安装：pip install llama-index-readers-web
from llama_index.readers.web import SimpleWebPageReader

reader = SimpleWebPageReader()
docs = reader.load_data(urls=["https://example.com"])
```

加载高级结构化网页（如 Readthedocs、Gitbook、Notion）：
LlamaIndex 在这方面做得比 LangChain 更专精，它有专门的爬虫来保留网站的目录结构。

```python
# 需安装：pip install llama-index-readers-web
from llama_index.readers.web import ReadabilityWebPageReader

# 使用 Mozilla 的 Readability 算法，自动剔除网页广告、导航栏，只留核心正文
reader = ReadabilityWebPageReader()
docs = reader.load_data(urls=["https://blog.example.com/article-123"])
```

**LlamaIndex 加载器的高级特性**

LlamaIndex 的 `SimpleDirectoryReader` 表面上看只是个遍历文件夹的工具，实际上它内置了许多 RAG 必备的高级功能：

**特性 1：文件级元数据继承**

如果你在加载时手动注入了额外的元数据，切分后的每一个小块都会自动继承这些信息。

**特性 2：内置多线程与异步极速加载**

处理大量文件时，不需要自己写并发代码，加两个参数即可：

```python
# 开启多线程加载 1000 个文件
docs = SimpleDirectoryReader(
    "./huge_folder", 
    num_workers=20  # 开启 20 个线程并发读取
).load_data()

# 异步加载
# docs = await reader.aload_data()
```

**特性 3：按需裁剪**

如果你在加载阶段（还没到切片阶段）就知道某些文件太大或太小，可以直接在加载器层面过滤。

```python
docs = SimpleDirectoryReader(
    "./data",
    required_exts=[".pdf", ".md"],   # 只要这两种后缀
    exclude=["./data/junk/"],        # 排除这个垃圾文件夹
).load_data()
```

##### LlamaIndex vs LangChain 加载器对比总结

| 维度           | LangChain                                  | LlamaIndex                                                   |
| :------------- | :----------------------------------------- | :----------------------------------------------------------- |
| **设计哲学**   | 格式转换器（把任意格式变成纯文本）         | 知识索引基石（为构建 Node 树和图谱服务）                     |
| **核心基类**   | `BaseLoader` -> 输出 `List[Document]`      | `BaseReader` -> 输出 `List[Document]`                        |
| **调用方式**   | 针对 PDF/MD/HTML 要 `import` 不同的类      | **90% 场景只用 `SimpleDirectoryReader`，通过后缀名自动派发** |
| **元数据处理** | 通常在加载后手动拼接或用 Unstructured 生成 | 加载器原生支持 `file_metadata` 注入，并自动向下游 Node 传递  |
| **生态集成**   | 直接写在 `langchain-community` 里          | 统一放在 `LlamaHub` (`llama-index-readers-xxx`)              |

如果你只是想把一个 PDF 的文字抠出来做简单的问答，两者没区别。
但如果你要构建一个包含几千个文件、需要精准按部门/时间/文件名过滤的**生产级 RAG 系统**，LlamaIndex 的加载器配合其自动继承的元数据机制，能帮你省去大量的“胶水代码”。

****

#### Embedding 文本嵌入

在构建现代AI应用（如RAG系统）时，文本嵌入（Embedding）是连接非结构化文本与向量数据库的核心桥梁。而根据生成向量形态的不同，主要分为**稀疏向量**与**稠密向量**，它们各有千秋，也催生了更强大的混合检索方案。

简单来说，**Embedding** 是将文本、图像等非结构化数据转换为计算机能理解和计算的**数值向量**的过程。这个向量试图捕捉数据的语义信息，使得在语义上相似的内容，其对应的向量在空间中也更靠近。

##### 构建稀疏向量

主流方法：

1. TF-IDF (最古老的基线)

- **原理**：如果一个词在当前文档中出现频率高（TF高），但在整个语料库中很少见（IDF高），那么这个词对区分该文档就越重要。
- **缺点**：无法控制词频的饱和度（一个词出现10次和100次对得分的贡献应该不同，但TF-IDF处理不好）。

2. 传统概率模型：BM25 (工业界标准，最常用)

- **原理**：基于概率检索框架，引入了文档长度归一化和词频饱和机制。**目前依然是 Elasticsearch、Lucene 等搜索引擎的默认打分算法。**
- **优点**：无需训练、速度快、对专有名词/ID/错误码极其敏感。

3. 学习型稀疏表示

- **原理**：使用神经网络（通常是 BERT 类模型）将文本映射为一个高维稀疏向量（如 30522 维，对应 BERT 词表大小）。与 TF-IDF 不同的是，**模型会为文本中“未出现但语义相关”的词也赋予非零权重**。
- **代表**：SPLADE, DeepImpact。
- **优点**：填补了传统稀疏向量“无法理解同义词”的致命短板，同时又保留了可解释性。

**BM25的构建与检索完整流程：**

BM25 的核心逻辑是**“倒排索引”**。假设我们要处理一个包含 3 篇文档的小型语料库，最终要回答用户的查询。

**阶段一：构建阶段 (索引构建 / Indexing)**

构建阶段的目标是建立从“词”到“文档”的映射表（倒排表）。

**步骤 1：文本预处理**

- **分词**：将句子拆分为词汇。例如：`"LangChain is great"` -> `["langchain", "is", "great"]`。*(注：中文需要使用 Jieba 或 HanLP 等分词器)*。
- **清洗**：去除标点符号、转小写、去除停用词。

**步骤 2：建立倒排索引**
系统扫描所有文档，建立如下数据结构：

| 词汇        | 包含该词的文档ID (Posting List) | 词频 (TF)          | 文档长度 (DL)           |
| :---------- | :------------------------------ | :----------------- | :---------------------- |
| `langchain` | [Doc1, Doc3]                    | Doc1:2次, Doc3:1次 | Doc1: 50词, Doc3: 120词 |
| `python`    | [Doc1, Doc2]                    | Doc1:1次, Doc2:5次 | Doc1: 50词, Doc2: 80词  |
| `error`     | [Doc2]                          | Doc2:3次           | Doc2: 80词              |

**步骤 3：计算全局统计量 (预先算好)**

- **文档总数 (N)**：总共有多少篇文档。

- 逆文档频率 (IDF)

  ：计算每个词的稀有程度。

  - 公式核心逻辑：包含某词的文档越多，IDF 越低（越不重要）。
  - 例如：`python` 出现在 10000 篇文档中（IDF极低），`error_404` 只出现在 5 篇中（IDF极高）。

- **平均文档长度**：所有文档长度的平均值，用于后续归一化。

**阶段二：检索阶段 (查询打分 / Querying)**

假设用户输入查询：`"langchain python error"`

**步骤 4：查询预处理**
对用户的查询进行与构建阶段相同的分词和清洗，得到词项：`["langchain", "python", "error"]`。

**步骤 5：召回候选文档**
通过倒排索引表，快速找到这三个词分别出现在哪些文档中。

- `langchain` -> Doc1, Doc3
- `python` -> Doc1, Doc2
- `error` -> Doc2
- **并集得出候选集**：Doc1, Doc2, Doc3。

**步骤 6：计算 BM25 得分 (核心函数)**
对候选集中的每个文档，计算查询词与该文档的 BM25 相关性得分。其简化版直觉公式如下：
$$
\text{score}(D,Q)=\sum_{i=1}^n \text{IDF}(q_i) \cdot \frac{f(q_i,D)\cdot(k_1+1)}{f(q_i,D)+k_1\cdot\left(1-b+b\cdot\frac{|D|}{\text{avgdl}}\right)}
$$
**通俗拆解这个公式：**

- $f(q_i,D)$：查询词$q_i$在文档$D$中的词频（TF），这个词在文档里出现了几次。
- $\text{IDF}(q_i)$：逆文档频率，衡量词的稀有度（稀有词权重更高）`error` 的 IDF 分高，`python` 的 IDF 分低。
- $k_1$：词频饱和参数（默认**1.2~2.0**），控制词频增长上限，避免高频词过度主导。意味着 `error` 出现 3 次和 30 次，对得分的提升是有限的，不会无限拉高总分。
- $b$：文档长度归一化参数（默认**0.75**），$b=1$完全按长度归一，$b=0$忽略长度影响。文档长度惩罚，如果文档 $D$ 的长度 ($∣D∣$) 远大于平均长度，分母变大，导致该词的得分被稀释。**这保证了长文档不会仅仅因为词多而占便宜。**
- $|D|$：文档长度，$\text{avgdl}$：语料平均文档长度

**步骤 7：排序与返回**
假设计算结果：Doc1 得 8.5 分，Doc2 得 12.1 分，Doc3 得 2.0 分。系统将 Doc2, Doc1 返回给用户。

**一、BM25 是什么**

BM25（Best Matching 25）是**基于概率模型的稀疏关键词检索算法**，是TF-IDF的升级版，核心是计算查询与文档的关键词匹配相关性得分，擅长**精确词匹配、专业术语、实体检索**，不依赖语义嵌入，计算快、可解释性强。

**核心公式**
$$
\text{score}(D,Q)=\sum_{i=1}^n \text{IDF}(q_i) \cdot \frac{f(q_i,D)\cdot(k_1+1)}{f(q_i,D)+k_1\cdot\left(1-b+b\cdot\frac{|D|}{\text{avgdl}}\right)}
$$
- $f(q_i,D)$：查询词$q_i$在文档$D$中的词频（TF）
- $\text{IDF}(q_i)$：逆文档频率，衡量词的稀有度（稀有词权重更高）
- $k_1$：词频饱和参数（默认**1.2~2.0**），控制词频增长上限，避免高频词过度主导
- $b$：文档长度归一化参数（默认**0.75**），$b=1$完全按长度归一，$b=0$忽略长度影响
- $|D|$：文档长度，$\text{avgdl}$：语料平均文档长度

**核心特点**

1. 词频饱和：词频增加到一定程度后，得分增长放缓，避免短文档高频词碾压长文档
2. 长度归一：平衡长短文档的得分，避免长文档天然得分偏高
3. 稀疏检索：只依赖词袋（Bag-of-Words），不理解语义，适合精确匹配场景

**二、BM25 在 RAG 中的作用与定位**

RAG（检索增强生成）的核心是“先检索、再生成”，单一向量检索（稠密检索）有明显短板：
- 向量检索：擅长**语义相似、同义改写、模糊查询**（如“癌症”→“恶性肿瘤”），但对**精确关键词、专业术语、实体ID、数字、符号**匹配弱，易漏召回
- BM25检索：擅长**精确词匹配、关键词命中、实体/术语检索**，但不理解语义，无法处理同义/近义查询

**BM25在RAG中的定位**：作为**稀疏召回通道**，与向量检索（稠密通道）互补，解决单一检索的召回盲区，提升检索全面性与相关性，是混合检索的核心稀疏组件。

**三、RAG 混合检索（BM25+向量）完整实现**

**核心流程（四步）**

1. 文档预处理与双索引构建（BM25索引 + 向量索引）
2. 并行召回：查询分别走BM25、向量检索，各取Top-K候选
3. 分数归一化+结果融合（线性加权 / RRF 排名融合）
4. 重排（可选）+ 送入LLM生成

**问题：**

**Q1：如果需要增加新的文档到数据库里面，是不是需要重新计算BM25。**

**是的，需要“更新”BM25，但不需要“从零开始全部重新计算”。**

准确地说，BM25 的更新机制分为**全局统计量的更新**和**倒排索引的追加**。现代系统对这块的处理非常成熟。

为了让你彻底弄懂，我们把 BM25 拆开来看，看看增加文档时到底发生了什么：

**1. 哪些东西需要“重新计算”？（全局变量）**

回顾一下 BM25 公式里的几个关键参数，它们是基于**整个语料库**算出来的：

- **文档总数 ($N$)**：原来 1000 篇，加了 1 篇变成 1001 篇。
- **逆文档频率 (IDF)**：某个词的 IDF 依赖它出现在多少篇文档中。如果新加的文档里包含了 `"Python"`，那么 `"Python"` 的 IDF 值就会**变小**（因为它变得不那么稀有了）。
- **平均文档长度 ($|avgdl|$)**：新文档拉高了或拉低了整体平均长度。

👉 **结论：每增加一批文档，这 3 个全局统计量必须重新计算一遍。**

**2. 哪些东西不需要重新计算？（局部变量）**

- **旧文档的倒排列表**：原来 Doc1 包含 `"Python"` 3次，这个事实不会因为新文档的加入而改变。
- **旧文档的长度 ($∣D∣$)**：Doc1 的字数是固定的。

👉 **结论：旧数据原封不动，只需要做“增量追加”。**



##### 构建稠密向量

如果说稀疏向量（如 BM25）是**“字典式查询”**（依靠关键词直接翻到对应页码），那么稠密向量就是**“语义空间导航”**（依靠理解含义，在茫茫星海中寻找距离最近的星座）。

构建稠密向量完全依赖于**深度神经网络**。以下是构建稠密向量的主流方法流派，以及从零到检索的完整生命周期拆解。

**主流方法：**

1. **双塔模型 / 对称编码器 (目前绝对主流)**

- **原理**：使用 Transformer 架构（如 BERT）。将整段文本输入模型，取 `[CLS]` 标签的隐藏层输出（或做 Mean Pooling 平均池化），压缩成固定维度（如 768、1024 维）的浮点数数组。模型通过**对比学习** 训练：让语义相似的文本在空间中靠近，不相似的拉远。
- 代表模型：
  - **BGE 系列 (BAAI)**：目前开源界的性能王者，支持多语言。
  - **E5 系列**：微软出品，首创了在输入文本前加指令前缀（如 `query: xxx`）的设计。
  - **OpenAI Embeddings**：闭源商用标杆（`text-embedding-3-small/large`），支持高达 3072 维度，并通过 Matryoshka Representation Learning (MRL) 技术实现了**无损降维**（存储时可以截断成 256 维，精度损失极小）。

2. **交叉编码器 (Cross-Encoder / Late Interaction)**

- **注意**：交叉编码器**不产生稠密向量**，不能存入向量数据库。它把 Query 和 Document 拼接在一起输入模型，直接输出一个相关性分数。精度极高，但速度极慢。通常作为**重排器**，在稠密向量召回后使用。

**深度拆解：稠密向量的构建与检索完整流程**

与 BM25 的“倒排索引”截然不同，稠密向量依赖的是**近似最近邻搜索 (ANN)** 算法。

**阶段一：构建阶段**

**步骤 1：文本预处理与截断**
深度模型有最大输入长度限制（如 BGE 通常是 512 个 Token）。如果文档超过 512 Token，必须先进行**文本切片**。

- *注意：切片策略（按句号切、按固定字数切、重叠切）对最终 RAG 效果影响极大。*

**步骤 2：模型推理 (生成 Embedding)**
将切分好的文本块 送入 Embedding 模型。

- **输入**：`"LangChain 是一个用于开发 LLM 应用的框架"`
- **底层发生**：Tokenization -> 多层 Transformer 注意力力计算 -> 提取最后一层隐藏状态 -> Mean Pooling（将变长序列压缩为固定长度） -> L2 归一化（将向量缩放到单位球面上，使得内积等价于余弦相似度）。
- **输出**：一个包含 1024 个浮点数的数组，例如 `[0.021, -0.145, 0.887, ..., 0.002]`。

**步骤 3：向量索引构建 (核心黑科技)**
如果你有 1000 万个文本块，每个 1024 维，用暴力遍历计算相似度需要几秒甚至几分钟，无法满足实时查询。因此必须构建 **ANN 索引**。主流算法包括：

- **HNSW (分层可导航小世界图)**：目前最主流的算法。它在高维空间中构建一张多层级的“高速公路图”，搜索时从顶层快速定位到目标所在区域，再在底层精确定位。**特点：检索速度极快，精度极高，但内存占用巨大。**
- **IVF_PQ (倒排文件 + 乘积量化)**：先对向量做聚类（IVF，缩小搜索范围），再把向量压缩成极短的字节码（PQ，节省内存）。**特点：极度节省内存，适合十亿级数据，但有精度损失。**

**步骤 4：存储入向量数据库**
将原始文本、生成的稠密向量、以及文件名等元数据，一并写入数据库（如 Milvus, Qdrant, FAISS）。

**阶段二：检索阶段**

**步骤 5：查询向量化**
用户发起查询：`"什么是大模型开发框架？"`。系统使用**同一个 Embedding 模型**，将这句问句转换为 1024 维的稠密向量。*(如果用的是 E5 或 BGE，还会在前面加上 `query:` 前缀)。*

**步骤 6：ANN 近似最近邻搜索**
在向量数据库中，用查询向量去“撞击” HNSW 或 IVF_PQ 索引。

- 数据库不需要计算查询向量与所有 1000 万个文档向量的距离，而是通过图的跳跃或聚类的筛选，在**毫秒级**内找出空间距离最近（余弦相似度最高）的 Top-K（如前 10 个）向量。

**步骤 7：返回结果**
根据找出的 Top-K 个向量 ID，从数据库中取出对应的原始文本块和元数据，返回给下游的 LLM。

**重点：**在 RAG 系统中，图片和表格是传统的“盲区”。如果你直接用之前的文本切片 + 稠密向量方法，遇到表格会变成一团乱码（所有行列黏在一起），遇到图片则会直接被丢弃（无法 encode）。

要解决这两个痛点，现代 RAG 架构进化出了两条清晰的路线：**“结构化还原”**（针对表格）和**“多模态对齐”**（针对图片）。

**遇见表格：从“拍扁的文本”到“保留逻辑的结构”**

传统解析器（如基础的 PyPDF）会把表格抽成纯文本：`姓名 张三 年龄 25 城市 北京 ...`。这会导致向量完全无法理解行列关系。

1. **构建阶段：结构化提取**

现在的核心目标是**把表格还原成机器能懂的格式**（Markdown 或 HTML）。

- **方案 A：视觉大模型提取（当前最强，强烈推荐）**
  不用管底层是什么 PDF 格式，直接把表格那一页截图，发给多模态大模型（如 GPT-4o, Claude 3.5 Sonnet, Qwen-VL），用 Prompt 强制它输出 Markdown 表格：

  > *“请将图片中的表格转换为严格的 Markdown 格式，保留所有表头和合并单元格信息。”*
  > 得到的结果：`| 姓名 | 年龄 | 城市 | \n |---|---|---| \n | 张三 | 25 | 北京 |`

- **方案 B：专用解析库（适合低成本、大批量）**
  使用 `Unstructured` 或 `docTR`、`Camelot` 等库，它们内置了线条检测和单元格聚合算法，能直接输出 Markdown 或 HTML 格式的表格字符串。

**存入数据库**：将提取出的 Markdown 字符串作为一个独立的 `Document`/`TextNode` 存入向量库，并在 Metadata 中打上标签 `{"type": "table", "page": 5}`。

2. **检索阶段：如何精准命中表格？**

单纯的稠密向量检索对表格依然不够好（用户可能搜索“25岁的员工”，但表格里没有这句话，只有“25”）。

**最佳实践：表格的混合检索**

1. **稠密向量**：对整个 Markdown 表格做 Embedding，用于匹配用户的**语义意图**（如“查询员工信息表”）。
2. **稀疏向量 (BM25)**：对 Markdown 表格做分词索引，用于精准匹配用户查询中的**具体单元格内容**（如“张三”、“北京”）。
3. **元数据过滤**：如果用户明确说“找一下文档里的表格”，直接在向量库查询时加上过滤条件 `where={"type": "table"}`。

**遇见图片：从“被忽略的视觉”到“多模态嵌入”**

图片处理取决于图片的**类型**：是插图/照片，还是图表？

1. **构建阶段：多模态对齐**

传统的文本 Embedding 模型无法处理图片，必须引入**多模态 Embedding 模型**（如 OpenAI CLIP, 优秀的开源模型如 `Jina-CLIP`, `BGE-Visualized`）。这类模型的魔法在于：**它能把图片和文本映射到同一个向量空间中。**

- **针对图表/流程图（推荐先转文本）**：
  和表格一样，最稳妥的方法是用多模态 LLM 看图说话，生成详尽的**图片描述文本**，然后将这段文本当做普通文档去生成文本 Embedding。

  > *“这是一张微服务架构图，包含用户网关、订单服务、支付服务，通过 Kafka 进行异步通信…”*

- **针对真实照片/插图（推荐生成图像向量）**：
  如果是产品图、人物照片，文字描述无法穷尽。此时需要调用 CLIP 类模型的**图像编码器**，直接从像素中提取出一个稠密向量（比如 768 维）。

**存入数据库**：

- 方法一（文本派）：存生成的“图片描述文本” + 对应的“文本向量”。
- 方法二（原生派）：在向量数据库中建立支持多模态的 Collection，同时存储“图片的二进制路径/URL”、“图片向量”和“人工标注的简短 Caption”。

2. **检索阶段：跨模态检索**

多模态检索最酷的地方在于**“以文搜图”或“以图搜图”**。

- **以文搜图**：用户输入“看看红色的运动鞋”。系统用同一个多模态模型将这句话变成文本向量，去数据库中与“图片向量”计算余弦相似度，直接召回对应图片。
- **以图搜图**：用户上传一张鞋子的照片，系统提取图片向量，去数据库找最近的图片向量。



##### 稀疏向量与稠密向量的融合 —— RRF (Reciprocal Rank Fusion，倒数秩融合)

在构建高级 RAG 系统时，我们已经知道：**稠密向量懂语义，稀疏向量（BM25）懂精准匹配。** 如果我们同时用这两种方法去检索，会得到两份排序不同的文档列表。

那么问题来了：**如何将这两份列表合并成一份最终结果，交给大模型（LLM）？**

最简单粗暴的方法是“分数相加”，但这种方法存在致命缺陷：稠密向量的分数通常是余弦相似度（0~1之间），而 BM25 的分数是一个没有上界的概率值（可能是 15.3，也可能是 200）。把它们直接加在一起，数值大的那一方会完全吞噬掉数值小的一方。

为了解决这个问题，工业界普遍采用了一种极其优雅、无需调参的算法——**RRF (Reciprocal Rank Fusion，倒数秩融合)**。

**一、 什么是 RRF？核心直觉是什么？**

RRF 的核心思想只有一句话：**我不看你具体考了多少分，我只看你在这份榜单里排第几名。**

**生活中的类比：**
假设有两个人去相亲，你想评估谁更优秀。

- **传统“分数融合”**：A 在富婆圈财富排名第 80 分（满分 100），B 在学霸圈智商排名第 150 分（满分 1000）。你直接把 80 和 150 相加？这显然不合理，量纲不同。
- **RRF 的思路**：不管分数多少，A 在富婆圈排 **第 1 名**，B 在学霸圈排 **第 3 名**。RRF 认为，在任何一个维度拿到第 1 名，都应该给极高的基础分。

**数学公式：**
对于某个文档 *d*，它在所有检索系统中的 RRF 得分计算如下：
$$
RRF\_Score(d) = \sum_{i=1}^{N} \frac{1}{k + rank_i(d)}
$$

**参数说明：**
- $d$：候选文档
- $N$：参与融合的检索通道总数（例如稀疏+稠密，则 $N=2$）
- $i$：第 $i$ 个检索通道
- $rank_i(d)$：文档 $d$ 在第 $i$ 个检索通道结果列表中的排名（从 $1$ 开始）
- $k$：平滑常数，通常默认设置为 $60$，用于防止排名靠前的文档得分趋于无穷大

**二、 RRF 为什么神？三大优势**

1. **绝对无视分数量纲**：彻底解决了 BM25 分数和余弦相似度无法直接比较的问题。
2. **零调参**：公式里唯一的参数 *k*，学术界和工业界经过大量验证，固定填 `60` 就是最优解，不需要根据业务瞎调。
3. **“错位互补”惩罚机制**：
   - 如果一个文档在 BM25 排第 1，在稠密排第 50，它的得分是 `1/61 + 1/110 ≈ 0.024`。
   - 如果一个文档在 BM25 排第 20，在稠密排第 1，它的得分是 `1/80 + 1/61 ≈ 0.029`。
   - *结论*：RRF 天然偏爱**在多个榜单中都排名靠前**的文档，而不是只在某一个榜单里称霸的偏科生。

#### 向量数据库

向量数据库是现代AI应用（尤其是RAG系统）的核心基础设施，它能够高效存储和检索高维向量数据，实现基于语义的相似性搜索。主流向量数据库各有其设计理念与适用场景，下表为您提供了一个全景式的对比概览。

| 数据库       | 核心定位                         | 主要语言           | 最大优势                                              | 主要局限                                 | 最佳适用场景                                         |
| :----------- | :------------------------------- | :----------------- | :---------------------------------------------------- | :--------------------------------------- | :--------------------------------------------------- |
| **Milvus**   | **企业级分布式**                 | Go, C++            | **云原生、高扩展、高性能**；支持数十亿向量            | 架构复杂，运维门槛高                     | **超大规模、高并发生产环境**（如推荐系统、图像检索） |
| **Qdrant**   | 高性能向量搜索                   | Rust               | **查询延迟低、过滤性能优秀**、内存效率高              | 超大规模分布式能力略逊于Milvus           | 对**查询延迟和过滤**有高要求的中大规模应用           |
| **Weaviate** | 知识图谱融合                     | Go                 | **原生支持混合检索**、内置向量化模块、GraphQL API灵活 | GraphQL有学习曲线，大规模性能不如Milvus  | 需要**复杂语义查询和关系推理**的场景（如智能问答）   |
| **Chroma**   | 开发友好型                       | Python, TypeScript | **极度易用、轻量级**、与LangChain/LlamaIndex集成紧密  | **扩展性弱**，不适合生产环境的大规模数据 | **快速原型开发、实验、小规模项目**                   |
| **Faiss**    | **高性能搜索库**（非完整数据库） | C++                | **极致搜索速度**，支持GPU加速，算法丰富               | 无持久化、无服务层，需自行封装           | 作为**底层检索引擎**嵌入到自定义系统中               |

**深度解析：Milvus**

Milvus 是一款开源的、云原生向量数据库，专为在海量向量数据集上进行高性能相似性搜索而设计。其核心优势在于其**分布式、存算分离的架构**，使其能够胜任最苛刻的企业级应用场景。

1. **架构设计：计算与存储分离**

Milvus 2.x 采用了**共享存储架构**，彻底分离了计算层和存储层，这是其实现高扩展性和高可用性的基础。其架构自上而下分为四层。

- **接入层**：由一组无状态的**代理**组成，是系统的入口，负责请求验证、路由和结果聚合。
- **协调服务**：系统的“大脑”，负责集群拓扑管理、任务调度、时间戳生成（TSO）和负载均衡。
- **工作节点**：系统的“手脚”，是无状态的计算单元。包括**查询节点**（处理搜索）、**数据节点**（处理写入）和**索引节点**（构建索引）。
- **存储层**：系统的“骨骼”，负责数据持久化。
  - **元存储**：存储Collection、Partition等元信息，通常使用`etcd`
  - **日志代理**：记录数据变更，实现系统崩溃后的恢复和分布式同步，通常使用`Pulsar`或`Kafka`
  - **对象存储**：存储实际的向量数据、索引文件等大规模数据，通常使用`MinIO`、`S3`等

2. **核心优势与特性**

- **卓越的扩展性**：得益于存算分离架构，可以独立扩展计算节点或存储节点，轻松应对数据量从百万到数百亿的增长。
- **高性能**：集成并优化了多种近似最近邻（ANN）索引，如 **HNSW、IVF_FLAT、IVF_PQ** 等，通过`Knowhere`引擎支持异构计算（CPU/GPU）。在亿级向量场景下，单机可达数百QPS，集群模式可处理更高并发。
- **功能丰富：**
  - **混合检索**：支持向量搜索与标量过滤的结合t。
  - **多向量与稀疏向量**：支持在同一个Collection中存储和检索多种向量（例如，同时用稠密向量做语义检索，稀疏向量做关键词匹配）。
  - **多租户**：支持在同一个集群中为不同租户隔离数据。
- **云原生**：支持Kubernetes部署，提供Operator，便于在云环境进行弹性伸缩和运维。

3. **潜在考量**

- **运维复杂度高**：分布式部署需要配置多个组件（etcd, Pulsar, MinIO等），对运维团队要求较高。
- **资源消耗相对较高**：为了达到高性能，需要较好的硬件支撑。

4. **适用场景**

Milvus 是以下场景的理想选择：

- **超大规模数据**：预期向量数据量超过亿级，需要分布式架构。
- **高并发在线服务**：需要毫秒级响应和高吞吐量（QPS）。
- **企业级关键应用**：对系统稳定性、可用性和扩展性有严苛要求的推荐系统、搜索引擎、图像视频检索等
