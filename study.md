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

