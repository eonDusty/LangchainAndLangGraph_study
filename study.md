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

主要分为：

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

