# Python 后端知识点

### FastAPI

#### SSE与Web Socket的区别

1. **SSE**

SSE（Server-Sent Events） 是一种基于HTTP长连接的服务器单向推送技术，其核心特点是为数据穿上了“制服”（固定格式），并提供了内置的“断线重连”机制。而您采用的NDJSON方案更像是“裸奔”的JSON数据流，它更轻量、更灵活，特别适合后端微服务间的高效通信。

2. **WebSocket**

WebSocket 是基于独立的 TCP 连接实现的，使用自定义的协议。客户端和服务器之间可以建立持久的全双工通信的连接，可以双向发送和接收数据。

WebSocket 协议是基于帧的，可以通过发送不同类型的帧进行通信。

**差异：**

+ SSE 适用于服务器向客户端单向发送实时更新的数据，适合实时事件推送场景。SSE 使用的是标准的 HTTP 协议，对于浏览器的兼容性较好，但只支持客户端接收数据。
+ WebSocket 适用于客户端和服务器之间的双向实时通信，适合聊天应用、实时游戏等场景。WebSocket 需要独立的 TCP 连接，因此相比 SSE，会增加一定的网络开销，但能够实现双向通信。

**适用场景：**

[SSE](http://apifox.com/apiskills/sse-testing-tools/) 适用于需要服务器向客户端单向实时推送数据的场景，例如实时更新的新闻、股票行情等。

**优点：**简单易用，对服务器压力小，浏览器兼容性好。

**缺点：**只支持单向通信，无法进行双向交互。

[WebSocket](http://apifox.com/apiskills/websocket-test-tools/) 适用于需要客户端和服务器之间实时双向通信的场景，例如聊天室、实时协作应用等。

**优点：**支持双向通信，实时性更高，可以实现更丰富的交互效果。

**缺点：**需要独立的 TCP 连接，对服务器压力更大，浏览器兼容性相对较差。

****

#### 进程、线程、协程的区别

**1. 进程（Process）—— 独立的厨房**

- 进程是**操作系统分配资源的基本单位**，每个进程有**独立的内存**，互不影响。
- 进程间通信（IPC）成本较高，比如需要用**管道、消息队列**等方式。

**2. 线程（Thread）—— 厨房里的多位厨师**

**比喻：一个厨房里有多个厨师，他们可以同时炒菜，但共用同一个冰箱、调料架（共享内存）。**

- 线程是**CPU 调度的基本单位**，一个进程可以有多个线程，**线程共享内存**。
- 线程之间可以**快速通信**，但如果多个线程同时修改共享资源（比如冰箱里的食材），可能会发生**数据竞争（race condition）** ，需要加锁（比如 GIL 在 Python 里限制了真正的并行）。

**3. 协程（Coroutine）—— 一个厨师能同时做多个菜**

**比喻：一个超级厨师，一边煮汤，一边炒菜，汤炖着的时候，顺便去切菜。**

- 协程是一种**用户态的"轻量级线程"** ，本质上是**单线程**，但可以**在等待 I/O 的时候主动切换任务**，从而提高效率（异步 I/O）。
- 协程不需要操作系统的调度，切换速度比线程快很多，也不会有线程锁的问题。

**协程（Coroutine）实际应用：**

- 在 **FastAPI** 里，`async` / `await` 允许 API 在等待数据库查询或 HTTP 请求时，不阻塞其他请求，提高并发能力。
- 适用于**I/O 密集型**任务（如数据库查询、网络请求）。
- **不适用于 CPU 密集型任务**，因为它还是**单线程**的。

**4. Flask与FastAPI**

**Flask** 默认是**同步的**  用默认的 `app.run()`，那就是**单进程 + 单线程**，并发能力很低。（需要 Gunicorn + 线程/进程支持并发）。
 **FastAPI** 天生支持**协程**（`async` / `await`），可以在**单线程**下高效处理并发请求。 FastAPI 之所以高并发，核心就是基于 **Python 的 `async` 和 `await`**，实现了**协程**并发处理请求。

- 当遇到 I/O 操作（比如数据库查询、HTTP 请求）时，FastAPI **不会阻塞线程**，而是**让出 CPU**，去处理其他请求。
- 这和传统的**多线程、进程不同**，是**单线程的异步并发**。

****

#### WSGI、uWSGI和uwsgi的全面介绍

**背景**

在深入了解WSGI之前，先回顾一下Web开发的基本原理。当用户在浏览器中输入一个URL并按下回车时，发生了什么？

1. 浏览器发送HTTP请求到Web服务器。
2. Web服务器接收请求并解析URL，确定要访问的资源。
3. Web服务器将请求传递给相应的应用程序（如Python应用）。
4. 应用程序处理请求并生成HTTP响应。
5. Web服务器将响应返回给浏览器，浏览器渲染页面或执行其他操作。

**一、WSGI**

WSGI（[Web Server Gateway Interface](https://zhida.zhihu.com/search?content_id=236392546&content_type=Article&match_order=1&q=Web+Server+Gateway+Interface&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NzY0MTUxNjIsInEiOiJXZWIgU2VydmVyIEdhdGV3YXkgSW50ZXJmYWNlIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MjM2MzkyNTQ2LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.IN7imfbODrjhxl9eWHP_9xBm249zbL7hpdRPwm7Fwgo&zhida_source=entity)）是**Python Web应用程序与Web服务器之间的接口标准**。它定义了应用程序和服务器之间的通信协议，使得不同的应用程序和不同的Web服务器可以无缝协作。 （**注：它只是一种规范**）

**WSGI的工作原理**

WSGI的核心思想是将Web应用程序与Web服务器解耦。它规定了应用程序需要实现的接口，以便能够与任何兼容WSGI的Web服务器通信。这种标准化的接口使得开发者可以专注于应用程序的逻辑，而无需关心与特定Web服务器的交互。

WSGI定义了两个主要组件：

- **应用程序（Application）**：WSGI应用程序是一个可调用对象，通常是一个函数或一个类的实例。它接受两个参数：`environ`和`start_response`，并返回一个迭代器，用于生成HTTP响应。
- **服务器网关（Server Gateway）**：服务器网关是一个中间件组件，它负责处理HTTP请求并将请求传递给WSGI应用程序。服务器网关还负责调用应用程序生成的响应，并将响应返回给客户端。

**WSGI中间件**

WSGI中间件是一种**用于在WSGI应用程序和Web服务器之间执行预处理或后处理操作的机制**。中间件可以用于添加额外的功能，如请求/响应处理、身份验证、缓存等。它们是构建复杂Web应用程序的重要组成部分。

**WSGI中间件的作用包括：**

- **请求处理**：中间件可以在请求到达应用程序之前执行一些处理逻辑，如身份验证、请求重定向等。
- **响应处理**：中间件可以在应用程序生成响应后对响应进行处理，例如添加HTTP头、压缩响应内容等。
- **异常处理**：中间件可以捕获应用程序抛出的异常，并根据需要执行特定的操作，如记录错误日志、返回自定义错误页面等。

**二、uWSGI**

uWSGI是一个**应用服务器**，它实现了WSGI协议并提供了高性能的Web应用程序托管环境。它支持多种协议，包括HTTP、[FastCGI](https://zhida.zhihu.com/search?content_id=236392546&content_type=Article&match_order=1&q=FastCGI&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NzY0MTUxNjIsInEiOiJGYXN0Q0dJIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MjM2MzkyNTQ2LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.qdrPgVUBSlFUh5DVMA95w7zoRNqqRRWGJBLplKmKfYg&zhida_source=entity)、[SCGI](https://zhida.zhihu.com/search?content_id=236392546&content_type=Article&match_order=1&q=SCGI&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NzY0MTUxNjIsInEiOiJTQ0dJIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MjM2MzkyNTQ2LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.1jwD1_gDbtnL39-UMiTgoAxWzUP2dfXXS2epOcsPABc&zhida_source=entity)等，使得Python应用程序可以与不同类型的Web服务器通信。

**三、uwsgi**

uwsgi是一个**通信协议**，它定义了应用服务器和Web服务器之间的通信方式。uWSGI应用服务器是uwsgi协议的一种实现。

**性能对比**

- **WSGI**：WSGI是一个标准接口，它提供了基本的通信协议，但不处理高性能问题。在生产环境中，通常需要额外的应用服务器来提供更好的性能。
- **uWSGI**：uWSGI应用服务器是一个高性能的解决方案，它可以处理大量并发请求，并提供各种优化选项。它是一个强大的工具，特别适用于高流量的Web应用程序。
- **uwsgi**：uwsgi协议是uWSGI应用服务器与Web服务器之间的通信协议，它是一种高效的协议，有助于提高性能。

**适用场景**

- **WSGI**：适用于开发和调试阶段，也可用于小型应用。在生产环境中，通常需要结合应用服务器来获得更好的性能。
- **uWSGI**：适用于高流量的Web应用程序，特别是需要处理大量并发请求的情况。它提供了各种性能调优选项。
- **uwsgi**：uwsgi协议适用于与uWSGI应用服务器配合使用，以提供高性能的通信。

****

#### ORM 调用

**ORM 调用**，简单来说，就是**在代码里用“面向对象”的方式去操作数据库，而不是写原始的 SQL 语句。**

ORM 的全称是 **Object-Relational Mapping（对象关系映射）**。

**ORM 调用方式（以 Python 最著名的 SQLAlchemy 为例）**

```python
from sqlalchemy.orm import Session
from models import User  # 这是一个你定义好的类

# 1. 拿到数据库会话（相当于连接）
db: Session = get_db()

# 2. 插入数据（就像是创建了一个 Python 对象）
new_user = User(name="李四", age=25)
db.add(new_user)
db.commit()

# 3. 查询数据（就像是调用类的属性去过滤）
zhangsan = db.query(User).filter(User.name == "张三").first()
print(zhangsan.age) # 直接当作对象用：28
```

**ORM 的本质**：当你在代码里写了 `db.add(new_user)` 时，ORM 框架在底层**自动帮你生成**了那条 `INSERT INTO ...` 的 SQL 语句，并帮你发送给数据库执行。查出来的结果集，它也会**自动帮你封装**成 Python 对象。

**优点：**

**① 绝对的安全：防御 SQL 注入**

如果你用原生 SQL 拼接字符串：`f"SELECT * FROM users WHERE name = '{user_input}'"`，黑客输入 `'; DROP TABLE users; --`，你的库就没了。ORM 调用（如 `User.name == user_input`）底层使用的是参数化查询，**从根本上杜绝了 SQL 注入**。

**② 解耦：换数据库成本极低**

今天你用 MySQL，明天老板要求换成 PostgreSQL。如果是原生 SQL，你要把项目里成千上万条 SQL 语句翻出来改语法（比如分页 `LIMIT` 和 `OFFSET` 的细微差别）。
如果用 ORM，你**一行业务代码都不用改**，只需要改一下 ORM 的配置连接字符串即可。

**③ 与现代框架（如 FastAPI）天作之合**

在 FastAPI 中，ORM 对象可以直接被 Pydantic 序列化成 JSON 返回给前端。
更重要的是，ORM 配合 **async（异步）**，可以实现**异步 ORM 调用**（如 `asyncpg` + SQLAlchemy，或者 Tortoise ORM）。在等待数据库查询的几十毫秒里，FastAPI 可以去处理其他用户的请求，这在大并发 RAG 系统中至关重要。

#### 联调链路（实际实现）

**前端请求方式**

- 在多个页面/组件里直接 fetch('/api/xxx')

- 例如：

  - demoView.vue：/api/view/upload、/api/view/files、/api/view/file_content

  - TokenManagerComponent.vue：/api/admin/tokens

  - ChatComponent.vue：/api/chat/（流式）

  - AgentChatComponent.vue：/api/chat/agent/{agent_name}（流式）

**代理转发（关键）**

- 在 web/vite.config.js 配了：

  - 匹配 ^/api

  - 转发到 VITE_API_URL 或默认 http://127.0.0.1:5050

  - 并 rewrite 去掉 /api 前缀

  - 所以前端请求 /api/chat/agent，后端实际收到的是 /chat/agent

**后端路由注册**

- server/main.py 中 app.include_router(router)

- server/routers/__init__.py 把各模块路由统一挂载：chat、admin、view、strate 等

- 所以 /chat/...、/admin/... 等都可直接访问

**为什么这样就能联调**

- 前端开发服务器和后端是不同端口，但前端只访问同源路径 /api/...

- 由 Vite 在开发时做代理，避免了前端手写跨域地址和端口

- 后端也开了 CORS（allow_origins=["*"]），即使直连后端也能过

****

#### **CORS** 跨源资源共享 

CORS（Cross-Origin Resource Sharing，跨域资源共享）是一种安全机制，允许或限制网页从不同域名请求资源。当前后端分离开发时，前端页面通常运行在不同的域名或端口上，需要配置 CORS 才能正常访问后端 API。

现代 Web 开发都是前后端分离（前端 5173，后端 8000），必定跨源。那就必须用到 CORS 机制。

**工作机制：**

CORS 的核心逻辑是：**把决定权交给后端服务器。**

1. 前端发请求（带上 `Origin: http://localhost:5173` 头，标明我是谁）。
2. 请求到达后端（比如 FastAPI，此时后端是能收到请求并正常处理的）。
3. **后端在响应头里加上一句话**：`Access-Control-Allow-Origin: http://localhost:5173`（意思是：我认识这个前端，允许它看我的数据） 。
4. 浏览器收到响应，一看后端允许了，才把数据交给前端的 JavaScript 代码。如果后端没加这个头，浏览器就会把数据拦截，并在控制台报红字。

****

#### is 与 == 的区别

这是一个 Python 中极其经典的易错点！一句话总结：

> **`==` 比较的是**值**，`is` 比较的是**身份**（内存地址）。**

当 `a` 和 `b` 是链表节点（`ListNode`）对象时，情况非常明朗，结论非常清晰：

> 对于 `ListNode`，**`is` 比较的是节点在内存中的物理位置，`==` 比较的也是节点在内存中的物理位置！**

是的，你没看错，对于自定义类的实例对象，在没有特殊处理的情况下，**`==` 和 `is` 的效果是完全一模一样的。**

****

#### 设计模式

设计模式是软件开发中解决常见问题的经典方案，Python 因其语言特性（动态类型、鸭子类型、装饰器等），很多模式的实现比传统 OOP 语言更加简洁。

根据设计模式的参考书 **Design Patterns - Elements of Reusable Object-Oriented Software（中文译名：设计模式 - 可复用的面向对象软件元素）** 中所提到的，总共有 23 种设计模式。

这些模式可以分为三大类：**创建型模式（Creational Patterns）**、**结构型模式（Structural Patterns）**、行为型模式（Behavioral Patterns）。

****







### MySQL

#### 内连接 外连接 左连接 右连接

在日常的数据库增删改查任务中，由于数据的规范设计，数据通常不集中在同一张表里，所以经常会涉及到多表的数据查询，多表数据查询需要表之间的连接，而表间连接方式有很多，下面就针对各种表连接方式进行介绍。在介绍之前，为了方便对文字概念的深入理解，本文利用实例和图例进行概念的补充深化，为准确理解提供支持。

**1）数据库的常用连接方式：**

内连接：inner join，最常见的一种连接方式

左连接：也叫左外连接（left [outer] join）

右连接：也叫右外连接（right [outer] join）

全连接：full [outer] join ，MySQL不能直接支持。

![img](https://img2023.cnblogs.com/blog/735202/202212/735202-20221225103947180-1990964259.png)

**内连接**

内连接，也叫等值连接， inner join得出同时存在t1表和t2表的数据集，通俗一点说就是求两个表的交集。

```sql
-- inner join
select * from course c inner join teacher t on c.t_id = t.t_id 
```

**外连接**

**1） 左连接**

左连接：left [outer] join,左连接从左表取出所有记录,与右表匹配。如果没有匹配，以null值代表右边表的列。outer 可以不写，默认情况下不写outer关键字。

```sql
-- left join
select * from course c left join teacher t  on  c.t_id = t.t_id 

-- left outer join
select * from course c left outer join teacher t  on c.t_id = t.t_id 
```

**2） 右连接**

右连接：right [outer] join，右连接从右表取出所有记录，与左表匹配。如果没有匹配，以null值代表左边表的列。outer 可以不写，默认情况下不写outer关键字。

```sql
-- left join
select * from course c right join teacher t  on  c.t_id = t.t_id 

-- left outer join
select * from course c right outer join teacher t  on c.t_id = t.t_id 
```

**全连接**

两个表的并集，MySQL暂不支持这种语句，不过可以使用union将两个结果集“堆一起”，利用左连接，右连接分两次将数据取出，然后用union将数据合并去重。

```sql
-- oracle的全连接
select * from a full join b on a.id = b.id

-- mysql的全连接
-- mysql中没有full join,mysql可以使用union实现全连接；
select * from a left join b on a.id = b.id
union
select * from a right join b on a.id = b.id
```

****

#### InnoDB vs MyISAM

**InnoDB**和**MyISAM**是MySQL中两种常用的存储引擎，它们在多个方面存在显著差异。

**InnoDB：[聚簇索引](https://zhida.zhihu.com/search?content_id=262185621&content_type=Article&match_order=1&q=聚簇索引&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NzY0MTQ4OTMsInEiOiLogZrnsIfntKLlvJUiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjIxODU2MjEsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.dZJiW6vYeb87N74PiLmJkn-UWRyeTAJIyXnxLfZS4N0&zhida_source=entity)的“紧凑派”**
InnoDB的数据存储就像一本“合订书”，数据和主键索引紧密结合在一起，存储在.ibd文件中。这种结构称为聚簇索引（Clustered Index），意味着数据行按主键顺序物理排列。好处是主键查询效率极高，但插入和更新时可能需要调整页面，略显“笨重”。

**MyISAM：[非聚簇索引](https://zhida.zhihu.com/search?content_id=262185621&content_type=Article&match_order=1&q=非聚簇索引&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NzY0MTQ4OTMsInEiOiLpnZ7ogZrnsIfntKLlvJUiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjIxODU2MjEsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.orau8gUejcNrwwrRfurvM20fhkxYVi5lO8WSLAh5pBw&zhida_source=entity)的“分家派”**
相比之下，MyISAM更像把“书页”和“目录”分开存放。数据文件（.MYD）和索引文件（.MYI）各自独立，索引指向数据的物理位置。这种非聚簇索引（Non-Clustered Index）让MyISAM在顺序写入时更轻快，但查询时可能需要额外的磁盘寻址。

****

### Redis

