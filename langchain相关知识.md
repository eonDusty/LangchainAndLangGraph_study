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

****

#### Memory

在构建基于大语言模型（LLM）的应用时，LLM 本身是**无状态的**。它就像一个“失忆症患者”，每次请求都是独立的，不知道之前说过什么。

**Memory（记忆机制）的作用，就是弥补 LLM 的这一缺陷，让应用具备上下文感知能力。**

##### Memory类型

记忆可以定义为获取、存储、保留和后续提取信息的过程。人脑中存在多种类型的记忆。

1. **感觉记忆**：这是记忆的最初阶段，使人能够在原始刺激结束后保留感觉信息（视觉、听觉等）的印象。感觉记忆通常只能持续几秒钟。其子类别包括图像记忆（视觉）、回声记忆（听觉）和触觉记忆（触觉）。
2. **短期记忆**（STM）或**工作记忆**：它存储我们当前意识到的、用于执行复杂认知任务（例如学习和推理）的信息。短期记忆的容量据信约为7个项目（[Miller，1956](https://lilianweng.github.io/posts/2023-06-23-agent/psychclassics.yorku.ca/Miller/)），持续时间为20-30秒。
3. **长期记忆**（LTM）：长期记忆能够将信息存储很长时间，从几天到几十年不等，存储容量几乎无限。长期记忆有两种亚型：
   - 显性/陈述性记忆：这是对事实和事件的记忆，指的是那些可以被有意识地回忆起来的记忆，包括情景记忆（事件和经历）和语义记忆（事实和概念）。
   - 内隐/程序性记忆：这种类型的记忆是无意识的，涉及自动执行的技能和程序，例如骑自行车或在键盘上打字。

在 AI 架构的演进中，**LangChain** 和 **LangGraph** 代表了两种截然不同的记忆设计哲学：前者侧重于**单链对话的包装**，后者侧重于**复杂图状态的全局管理**。

##### Langchain的 Memory 机制：面向“对话链”的封装

LangChain 早期的 Memory 设计非常直观，它的核心思想是：**在每次调用 LLM 之前，自动把历史对话拼接到 Prompt 中。**

**1. 核心原理：隐式的 Prompt 拼接**

无论你用哪种 Memory，底层逻辑都是：取出历史消息 -> 格式化 -> 插入到 System/Human 消息之前。

**2. 主流的 Memory 类型**

- ConversationBufferMemory（缓冲记忆）：
  - **原理**：原封不动地保存所有的 `Human` 和 `AI` 消息。
  - **缺点**：容易撑爆 Token 窗口。
- ConversationBufferWindowMemory（滑动窗口记忆）：
  - **原理**：只保留最近 *K* 轮对话，老对话直接丢弃。
  - **优点**：控制了 Token 成本，但彻底丢失了早期上下文。
- ConversationSummaryMemory（摘要记忆）：
  - **原理**：在后台悄悄调一次 LLM，把旧对话总结成一段话，只保留摘要和最新几轮对话。
  - **优点**：极大地节省 Token。
- ConversationEntityMemory（实体记忆）：
  - **原理**：通过信息抽取（NER），记住对话中出现的“实体”及其属性（例如：记住“张三 -> 职业:程序员，爱好:打游戏”）。

**3. LangChain Memory 的致命痛点（为什么会被边缘化？）**

- **与 `Runnable` 接口不兼容**：LangChain 后来推出了强大的 LCEL（LangChain Expression Language），一切皆 `Runnable`。但旧版 Memory 是个黑盒，很难优雅地融入 `.pipe()` 链式调用中。
- **只支持单线程**：如果你用同一个 Chain 同时服务两个用户 A 和 B，他们的记忆会串台（因为底层共用一个 List 存消息）。
- **无法持久化**：程序一重启，内存里的 List 就清空了。

**解决上述痛点的现代方案（仍在 LangChain 体系内）：**
不再使用特定的 Memory 类，而是直接使用 **`BaseChatMessageHistory`**（如 `RedisChatMessageHistory` 或 `SQLChatMessageHistory`）配合 `RunnableWithMessageHistory`，手动把历史消息注入到 Prompt 中。这把控制权交还给了开发者。

##### LangGraph的 Memory 机制：面向“状态图”的全局共享

LangGraph 是为了构建**多步骤 Agent（智能体）**而生的。在 Agent 场景下，“记忆”不仅仅是聊天记录，还包括：**当前走到了哪个节点？上一步工具调用的返回值是什么？有没有死循环？**

因此，LangGraph **彻底抛弃了传统的 Memory 类**，转而使用了**状态机**的概念。

**1. 核心原理：State 就是 Memory**

在 LangGraph 中，你定义一个 `State`（通常是一个继承自 `TypedDict` 的 Python 字典）。这个 State 在图的所有节点之间传递，**State 的内容就是图的全部记忆**。

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph

# 定义状态（这就是 Memory 的结构）
class AgentState(TypedDict):
    # 核心记忆：完整的消息列表
    messages: Annotated[list, add_messages] 
    
    # 扩展记忆：你可以随意加任何东西！
    current_plan: str      # 记住当前的计划
    retrieved_docs: list   # 记住 RAG 检索到的文档
    loop_count: int        # 记住循环了多少次（防死锁）

# 注解 Annotated[list, add_messages] 是魔法所在：
# 它告诉 LangGraph，如果新节点返回了新消息，不要覆盖旧消息，而是“追加”到 list 中。
```

**2. LangGraph 实现多轮对话的流程**

+ 用户发起第 2 轮提问。
+ LangGraph 从**外部持久化存储**（如 Postgres、Redis）中加载之前的 `messages` 列表，构建出当前的 `State`。
+ 图开始执行，节点读取 `State["messages"]`，就能看到第 1 轮的记录。
+ 节点产生新回复，返回 `{"messages": [new_message]}`。
+ 因为有 `add_messages` 注解，新回复被追加到 State 中。
+ 图执行结束前，触发**检查点**，将最新的 State 整体保存回数据库。

**3. 为什么 LangGraph 的设计更高级？**

- **多用户隔离天然支持**：因为每个图的执行都绑定了唯一的 `thread_id`（对应数据库里的一行记录），A 用户和B用户的 State 完全物理隔离。
- **“时间旅行”**：这是 LangGraph 的杀手级功能。因为每次执行都保存了 State 的快照，你可以获取任意历史时刻的 State，甚至**回滚到过去的状态继续执行**。
- **不仅仅是聊天**：State 里面可以存检索结果、代码片段、结构化数据。这对于复杂的 RAG Agent 来说至关重要。

| 维度           | LangChain            | LangGraph                |
| -------------- | -------------------- | ------------------------ |
| **架构复杂度** | 线性、简单           | 图结构、复杂             |
| **状态管理**   | 基于对话历史的列表   | 基于图的全局状态         |
| **适用场景**   | 客服机器人、问答系统 | 复杂任务编排、多代理系统 |
| **持久化支持** | 有限                 | 原生支持检查点           |
| **扩展性**     | 适合原型开发         | 适合生产环境             |

### MCP Skills Harness

#### MCP

MCP （Model Context Protocol，模型上下文协议）定义了应用程序和 AI 模型之间交换上下文信息的方式。这使得开发者能够**以一致的方式将各种数据源、工具和功能连接到 AI 模型**（一个中间协议层），就像 USB-C 让不同设备能够通过相同的接口连接一样。MCP 的目标是创建一个通用标准，使 AI 应用程序的开发和集成变得更加简单和统一。

MCP 采用 “客户端-主机-服务器”（Client-Host-Server） 的三层架构：

|  Host  | 控制中心   | **通常是运行AI模型的应用程序（如Claude Desktop、IDE），负责管理多个Client实例、用户授权和整体安全策略。** |
| :----: | ---------- | ------------------------------------------------------------ |
| Client | 通信桥梁   | 运行在Host内，与特定的MCP Server建立一对一的有状态连接。负责协议协商、消息路由和调用服务器提供的功能。 |
| Server | 能力提供者 | 独立的轻量程序，封装对某类数据源或工具的访问（如GitHub、数据库、文件系统），通过标准化接口向Client提供资源(Resources)、工具(Tools) 和提示模板(Prompts) |

**MCP 的stdio模式以及Streamable HTTP模式**

1. **Stdio模式：本地进程通信**

Stdio（标准输入/输出）模式是MCP最早定义的传输机制，其核心是让客户端将MCP服务器作为一个子进程启动，并通过标准输入（stdin）和标准输出（stdout）进行通信。

**关键特征**

- 进程启动：客户端负责启动服务器进程，例如通过命令行运行 your-mcp-server。

- 通信方式：所有请求和响应都通过标准输入输出流传递，通常使用JSON-RPC格式

- 适用场景：非常适合本地开发和命令行工具集成，Claude桌面应用就主要使用此模式

2. **Streamable HTTP模式：网络服务通信**

随着MCP协议的发展，引入了Streamable HTTP作为新的传输方式，用以替代原先的HTTP + SSE。这种模式使MCP服务器可以像传统Web服务一样运行。

**关键特征**

- HTTP接口：服务器暴露一个HTTP端点，客户端通过POST请求发送JSON-RPC消息。

- 流式支持：虽然名为"Streamable HTTP"，但主要通过HTTP进行请求-响应通信，对于实时通知可通过SSE（Server-Sent Events）实现。

- 部署优势：非常适合云原生环境，可以部署在云服务器、容器中，通过API网关访问。

3. **实际应用建议**

- 开发环境：使用Stdio模式，便于调试和快速迭代。

- 生产环境：使用Streamable HTTP模式，便于扩展和远程访问。

```python
import asyncio
from typing import List
from langchain_core.tools import BaseTool
from langchain_mcp_adapters.client import MultiServerMCPClient

from langgraph.prebuilt import create_react_agent
from langchain_core.messages import HumanMessage
from langchain.chat_models import init_chat_model

_tools = None

async def get_tool() -> List[BaseTool]:
    global _tools
    # streamable_http方式
    client = MultiServerMCPClient({
        # 高德地图MCP Server
        "amap-maps-streamableHTTP": {
            "url": f"https://mcp.amap.com/mcp?key=5fxxxxxxxxxxxxxxxx7c",
            "transport": "streamable_http"
        }
    })

    # stdio方式
    # client = MultiServerMCPClient({
    #     # 高德地图MCP Server
    #     "amap-mcp-server": {
    #         "command": "uvx",
    #         "args": [
    #             "amap-mcp-server"
    #         ],
    #         "env": {
    #             "AMAP_MAPS_API_KEY": "5fxxxxxxxxxxxxxxxxxxxxxxxxx7c"
    #         },
    #         "transport": "stdio"
    #     }
    # })

    _tools = await client.get_tools()

    print(f"✅ 成功获取 {len(_tools)} 个工具")
    # print(f"🔍 工具列表:\n" + "\n".join(t.name for t in _tools))
    # print(f"tools:\n\n" + "\n\n".join(str(tool) for tool in _tools) + "\n")

    return _tools

async def run_graph_agent():
    tools = await get_tool()

    model = init_chat_model(
    "Qwen/Qwen3-8B",
    model_provider="openai",
    base_url = "https://api.siliconflow.cn/v1",
    api_key = "sk-frhxxxxxxxxxxxxxxxxxxxxxxxxxsezvtvjtmckx"
    )

    agent = create_react_agent(
        model,
        tools=tools,
        # max_retries = 3,
        prompt="你是一个有用的助手，可以使用高德地图工具来帮用户查询地点、路线。"
    )

    # ==================== 执行 Agent ====================
    result = await agent.ainvoke({
        "messages": [HumanMessage(content="从北京故宫到颐和园怎么走？")]
    })

    # ==================== 打印所有中间结果 ====================
    print("=" * 60)
    print("📋 完整执行过程（中间结果）")
    print("=" * 60)

    for i, msg in enumerate(result["messages"]):
        print(f"\n--- 第 {i+1} 步 ---")
        msg_type = msg.__class__.__name__

        if msg_type == "HumanMessage":
            print(f"👤 用户输入: {msg.content}")

        elif msg_type == "AIMessage":
            # 大模型可能同时在思考文本 + 调用工具
            if msg.content:
                print(f"🤖 大模型思考: {msg.content}")
            if msg.tool_calls:
                for tc in msg.tool_calls:
                    print(f"🔧 调用工具: {tc['name']}")
                    print(f"   📥 传入参数: {tc['args']}")

        elif msg_type == "ToolMessage":
            # 工具返回结果可能很长，截断显示
            content = str(msg.content)
            if len(content) > 500:
                content = content[:500] + f"\n... (省略 {len(content) - 500} 字符)"
            print(f"📦 工具返回 ({msg.name}):")
            print(f"   {content}")

    print("\n" + "=" * 60)
    print("🎯 最终回答:")
    print("=" * 60)
    print(result["messages"][-1].content)

if __name__ == '__main__':
    asyncio.run(run_graph_agent())

    # 测试一下是否真的拿到了
    # for i in range(len(tools)):
    #     print(f"\n🎯 主程序拿到工具了，第{i}个工具是: {tools[i].name}")
   
```

#### Skills

**代理技能（Skills）**是一种用于封装可复用行为的机制，通过 `SKILL.md` 文件定义，使 LLM 能够**按需加载、按需执行**特定能力。

简单来说，**Skill 就是 AI 可以调用的一项能力封装**。

**设计上的亮点是“渐进式披露”**：

- **元数据**常驻上下文，AI 知道有哪些技能可用。
- **正文**按需加载，只有触发时才读取，避免挤占 Token。

复杂点的 Skill，还会有附加的资源目录、脚本和参考文档。

```
skill-name/
├── SKILL.md          # 必需：元数据（何时使用）+ 正文（指令、流程、示例）
├── scripts/          # 可选：可执行脚本（Python/Bash），按需调用
├── references/       # 可选：参考文档，按需读取
└── assets/           # 可选：模板、图片等资源
```

**渐进式披露机制：**

1. **元数据层（Metadata） 始终加载**

   - Name：skill的名称

   - Description：skill的作用描述

2. **指令层（Instruction） 按需加载**
   - Skill.md中去了名称和描述之外的内容

3. **资源层（Resource）按需中的按需加载**

   - Reference：是被读取的，Reference.md

   - Script：是被执行的 （Python 脚本、Shell 命令、API 调用函数）

#### 两者的区别

Skills 和 MCP 代表了智能体技术栈中两个关键的抽象层：

| **组件**   | **一句话定义**             | **形象类比** | **关键理解**                       |
| ---------- | -------------------------- | ------------ | ---------------------------------- |
| **MCP**    | 标准化的工具接入协议       | USB-C 接口   | 解决外部系统"如何接入"（连通性）   |
| **Skills** | 用自然语言定义的 sub-agent | 任务说明书   | 解决复杂任务"如何编排"（执行逻辑） |

| 维度         | MCP (Model Context Protocol)               | Skills                                         |
| :----------- | :----------------------------------------- | :--------------------------------------------- |
| **核心思路** | **标准化连接**：通过 JSON-RPC 统一数据格式 | **逻辑编排**：用自然语言描述复杂执行路径       |
| **定义方式** | 在 Server 端用代码（TS/Python）写死逻辑    | 在 `SKILL.md` 中用自然语言引导模型决策         |
| **环境依赖** | 需要运行一个 MCP Server 进程               | 依赖可执行环境（如本地 Shell 或沙箱）          |
| **哲学**     | **以协议为中心**：一次编写，所有 AI 通用   | **以模型为中心**：利用模型推理能力处理不确定性 |

- **MCP 解决的是连通性** ：它像 USB-C，让 AI 能以统一格式读文件、查数据库。
- **Skills 解决的是编排逻辑** ：它像一份说明书，告诉 AI 如何执行复杂任务流——这些任务完全可以包括调用多个 MCP 工具。
- **两者的关系** ：它们解决的是不同层面的问题。MCP 负责把外部系统接入进来，Skills 负责决定什么时候用、怎么组合这些能力。一个高级 Skill 的底层往往就是调用多个 MCP 工具

**高频问题**：

1. **Skills 是什么？** → 延迟加载的 sub-agent，解决"如何编排"问题
2. **Skills 和 MCP 的区别？** → MCP 负责连通性，Skills 负责执行逻辑，互补关系
3. **如何降低 token 消耗？** → 渐进式披露：元数据常驻，正文按需加载
4. **什么是渐进式披露？** → 三层架构：元数据 → 正文 → 附加资源
5. **如何编写高质量 Skills？** → 精准 description + 单一职责 + 确定性优先

#### Harness 核心概念

 **一、Harness 到底是什么？**

一句话：**Agent = Model + Harness。你不是模型，那你就是 Harness。**

**Harness 就是模型之外的一切**——系统提示词、工具调用、文件系统、沙箱环境、编排逻辑、钩子中间件、反馈回路、约束机制。模型本身只是能力的来源，只有通过 Harness 把状态、工具、反馈和约束串起来，它才真正变成一个 Agent。

| 层级                    | 解决的核心问题                                 | 关注点                                       | 典型工作                                   |
| ----------------------- | ---------------------------------------------- | -------------------------------------------- | ------------------------------------------ |
| **Prompt Engineering**  | 表达——怎么写好指令                             | 塑造局部概率空间，让模型听懂意图             | 系统提示词设计、Few-shot 示例、思维链引导  |
| **Context Engineering** | 信息——给 Agent 看什么                          | 确保模型在合适的时机拿到正确且必要的事实信息 | 上下文管理、RAG、记忆注入、Token 优化      |
| **Harness Engineering** | 执行——整个系统怎么防崩、怎么量化、怎么持续运转 | 长链路任务中的持续正确、偏差纠正、故障恢复   | 文件系统、沙箱、约束执行、熵管理、反馈回路 |

简单任务里，提示词最重要——你把话说清楚就行；依赖外部知识的任务里，上下文很关键——你得把正确的信息喂进去；但在长链路、可执行、低容错的真实商业场景里，Harness 才是决定成败的东西。一线团队的重心都放在了 Harness 上，原因就在这。

**包含的组件**

理解 Harness 的最好方式，不是直接看它包含什么，而是看模型做不到什么。不管大模型看起来多能干，本质就是一个文本（或图像、音频）进、文本出的函数。

**模型做不到的，就是 Harness 要补的：**

| 模型做不到                         | Harness 怎么补                     | 核心组件         |
| ---------------------------------- | ---------------------------------- | ---------------- |
| 记住多轮对话历史                   | 维护对话历史，每次请求时拼进上下文 | **记忆系统**     |
| 执行代码、跑命令                   | 提供 Bash + 代码执行环境           | **通用执行环境** |
| 获取实时信息（新库版本、API 变化） | Web Search、MCP 工具               | **外部知识获取** |
| 操作文件和环境                     | 文件系统抽象 + Git 版本控制        | **文件系统**     |
| 知道自己做对了没有                 | 沙箱环境 + 测试工具 + 浏览器自动化 | **验证闭环**     |
| 在长任务中保持连贯                 | 上下文压缩、记忆文件、进度追踪     | **上下文管理**   |

把这些”模型做不了但你希望 Agent 能做到”的事情一个个补上，就得到了 Harness 的核心组件。LangChain 把这件事拆解为五个子系统：文件系统（持久化）、Bash 执行（通用工具）、沙箱环境（安全隔离）、记忆机制（跨会话积累）、上下文压缩（对抗衰减）。

**二、Harness进阶**

**六层体系**

| 层级   | 名称                   | 解决什么问题                   | 关键设计                                                     |
| ------ | ---------------------- | ------------------------------ | ------------------------------------------------------------ |
| **L1** | **信息边界层**         | Agent 该知道什么、不该知道什么 | 定义角色与目标，裁剪无关信息，结构化组织任务状态             |
| **L2** | **工具系统层**         | Agent 怎么跟外部世界交互       | 工具的选拔、调用时机、结果的提炼与反馈                       |
| **L3** | **执行编排层**         | 多步骤任务怎么串起来           | 让模型像人一样走完“理解目标→判断信息→分析→生成→检查”的完整轨道 |
| **L4** | **记忆与状态层**       | 长任务中间结果怎么管           | 独立管理当前任务状态、中间产物和长期记忆，防止系统混乱       |
| **L5** | **评估与观测层**       | Agent 怎么知道自己做对了没有   | 建立独立于生成过程的验证机制，让 Agent 具备“自知之明”        |
| **L6** | **约束、校验与恢复层** | 出错了怎么办                   | 预设规则拦截错误，失败时（API 超时、格式混乱）提供重试或回滚机制 |

可以类比成给一个新手员工搭建的完整工作环境。L1 是岗位说明书（告诉 ta 该关注什么），L2 是办公工具（给 ta 用什么干活），L3 是标准操作流程（按什么步骤做事），L4 是项目管理系统和笔记本（怎么记住做过的事），L5 是质检流程（怎么检验做对了没有），L6 是红线规则和应急预案（什么事绝对不能做、出了事怎么补救）。

Harness Engineering 相关的高频面试问题整理在下面，方便你快速回顾：

**基础概念**

| 问题                                                         | 核心回答                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| **Harness 是什么？**                                         | 模型之外的一切——系统提示词、工具调用、文件系统、沙箱、编排逻辑、约束机制。Agent = Model + Harness。 |
| **Harness 和 Prompt Engineering、Context Engineering 的关系？** | 嵌套关系：Prompt ⊂ Context ⊂ Harness。三者分别解决表达、信息、执行三个层面的问题。 |
| **为什么瓶颈不在模型而在 Harness？**                         | [Can.ac](http://Can.ac) 实验证明同一模型只换工具调用格式，分数从 6.7% 跳到 68.3%。基础设施质量决定了模型能力的实际发挥。 |

**架构设计**

| 问题                           | 核心回答                                                     |
| ------------------------------ | ------------------------------------------------------------ |
| **Harness 六层架构是什么？**   | L1 信息边界 → L2 工具系统 → L3 执行编排 → L4 记忆与状态 → L5 评估与观测 → L6 约束校验与恢复。从“定义边界”到“兜底恢复”的完整闭环。 |
| **上下文管理有什么经验法则？** | 利用率控制在 40% 以内。超过后 Agent 质量明显下降（幻觉增多、兜圈子）。策略是压缩或交接，不是继续塞信息。 |
| **单 Agent 还是多 Agent？**    | 规模决定。小项目单 Agent 够用（Hashimoto 模式），大项目几乎必然需要专业化分工（Carlini 用 16 个并行 Agent）。 |