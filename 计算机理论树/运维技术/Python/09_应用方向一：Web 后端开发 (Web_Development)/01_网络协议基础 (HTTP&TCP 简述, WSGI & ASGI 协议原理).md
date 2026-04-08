## 本章概述

本章聚焦 Python 最核心的工业级应用方向 ——Web 后端开发，从 Web 通信的底层协议基础出发，系统覆盖 Python 三大主流 Web 框架的架构设计、核心原理、工程化实践与生态体系。内容严格遵循 Python 官方 PEP 规范与 Web 开发行业标准，深度拆解 WSGI/ASGI 通信协议的底层实现、Django 的企业级全栈能力、Flask 的微内核灵活架构、FastAPI 的现代异步高性能特性，同时覆盖 ORM 映射、权限认证、数据验证、数据库迁移等后端开发核心能力。本章兼顾底层原理深度与工程化落地实践，同时覆盖 Python 后端开发面试的超高频核心考点。

---

Web 后端开发的本质是基于网络协议实现客户端与服务端的通信与数据交互，底层协议是理解 Web 框架运行原理的核心前提。

### 9.1.1 TCP 与 HTTP 协议简述

#### 1. TCP 协议：传输控制协议

TCP（Transmission Control Protocol）是位于传输层的**面向连接、可靠、基于字节流**的传输协议，是 HTTP 协议的底层传输载体，Web 服务的 HTTP 请求 / 响应均基于 TCP 连接传输。

核心特性与 Web 开发相关的核心机制：

- **三次握手建立连接**：客户端与服务端通过三次交互确认双方的发送与接收能力正常，建立可靠的 TCP 连接，是 HTTP 请求发起的前置步骤；
- **四次挥手断开连接**：连接终止时，通过四次交互确认双方数据传输完成，安全释放连接资源；
- **可靠传输保证**：通过序列号、确认应答、超时重传、流量控制、拥塞控制机制，保证数据无丢失、无重复、按序到达，HTTP 协议无需额外处理数据可靠性问题；
- **面向字节流**：将应用层数据视为无结构的字节流，不限制数据大小，HTTP 的请求体、响应体可传输任意长度的内容。

#### 2. HTTP 协议：超文本传输协议

HTTP（Hypertext Transfer Protocol）是位于应用层的 Web 通信标准协议，定义了客户端（浏览器、APP 等）与服务端之间的通信格式、交互规则与语义规范，是 Web 开发的核心协议。

##### （1）核心版本演进

|版本|核心特性|Web 开发影响|
|---|---|---|
|HTTP/1.1|长连接默认开启、管道化请求、Host 头强制、断点续传支持|目前兼容性最广的版本，Web 框架默认兼容|
|HTTP/2|二进制帧传输、多路复用、头部压缩、服务器推送|大幅减少 TCP 连接数，提升高并发场景性能，主流 Web 服务器与框架均支持|
|HTTP/3|基于 QUIC 协议（UDP 底层）、0-RTT 握手、连接迁移、无队头阻塞|弱网环境性能提升显著，现代 CDN 与大型 Web 服务已大规模落地|

##### （2）HTTP 请求结构

一个完整的 HTTP 请求由 4 部分组成：

1. **请求行**：包含请求方法、请求路径（URI）、HTTP 版本，示例：`GET /api/users HTTP/1.1`；
2. **请求头（Request Headers）**：键值对格式，传递请求的元信息，核心通用头包括：
    
    - `Host`：请求的目标域名与端口，HTTP/1.1 强制必填；
    - `User-Agent`：客户端标识（浏览器、设备、版本）；
    - `Content-Type`：请求体的媒体类型，如`application/json`、`application/x-www-form-urlencoded`、`multipart/form-data`；
    - `Authorization`：认证信息，如 Token、JWT；
    - `Cookie`：客户端存储的会话信息；
    - `Accept`：客户端可接收的响应内容类型；
    
3. **空行**：分隔请求头与请求体；
4. **请求体（Request Body）**：可选，POST/PUT 等请求携带的业务数据，格式由`Content-Type`指定。

##### （3）HTTP 响应结构

一个完整的 HTTP 响应由 4 部分组成：

1. **状态行**：包含 HTTP 版本、状态码、状态描述，示例：`HTTP/1.1 200 OK`；
2. **响应头（Response Headers）**：键值对格式，传递响应的元信息，核心通用头包括：
    
    - `Content-Type`：响应体的媒体类型与编码，如`application/json; charset=utf-8`；
    - `Content-Length`：响应体的字节长度；
    - `Set-Cookie`：向客户端设置 Cookie；
    - `Cache-Control`：缓存控制策略；
    - `Access-Control-Allow-*`：CORS 跨域资源共享控制；
    
3. **空行**：分隔响应头与响应体；
4. **响应体（Response Body）**：服务端返回的业务数据，如 HTML 页面、JSON 数据、文件流等。

##### （4）核心请求方法与语义

HTTP/1.1 定义了 9 种标准请求方法，RESTful API 开发中核心使用 5 种：

|方法|核心语义|幂等性|安全性|典型场景|
|---|---|---|---|---|
|`GET`|从服务端获取资源，不修改服务端数据|是|是|查询数据、页面渲染|
|`POST`|向服务端提交数据，创建新资源|否|否|新增数据、表单提交、登录认证|
|`PUT`|全量更新服务端指定资源，客户端提供完整的资源数据|是|否|全量修改数据|
|`PATCH`|增量更新服务端指定资源，客户端仅提供需修改的字段|否|否|部分字段修改|
|`DELETE`|删除服务端指定资源|是|否|删除数据|

> 幂等性：多次执行相同的请求，对服务端资源的影响与单次执行完全一致，是分布式系统、接口设计的核心规范；安全性：请求不会修改服务端的任何资源。

##### （5）HTTP 状态码

状态码是服务端对请求处理结果的标准化标识，分为 5 大类，是 Web 开发中必须掌握的核心规范：

|状态码分类|核心含义|典型常用码|
|---|---|---|
|1xx 信息性|请求已接收，继续处理|101 协议切换（WebSocket）|
|2xx 成功|请求已成功处理|200 OK、201 Created、204 No Content|
|3xx 重定向|需要客户端进一步操作完成请求|301 永久重定向、302 临时重定向、304 资源未修改（缓存）|
|4xx 客户端错误|客户端请求有误，服务端无法处理|400 Bad Request、401 Unauthorized、403 Forbidden、404 Not Found、405 Method Not Allowed、409 Conflict|
|5xx 服务端错误|服务端处理请求时发生异常|500 Internal Server Error、502 Bad Gateway、503 Service Unavailable、504 Gateway Timeout|

##### （6）HTTP 无状态特性

HTTP 协议本身是**无状态**的：服务端不会保留客户端的任何状态信息，两次独立的 HTTP 请求之间没有任何关联，服务端无法识别两个请求是否来自同一个客户端。

Web 开发中，通过以下方案解决无状态带来的会话保持问题：

- **Cookie+Session**：服务端为客户端生成唯一的会话标识 Session ID，通过`Set-Cookie`响应头写入客户端 Cookie，客户端后续请求自动携带 Cookie 中的 Session ID，服务端通过 ID 匹配对应的会话数据；
- **Token/JWT**：服务端登录成功后生成加密的 Token 令牌，返回给客户端，客户端后续请求通过`Authorization`请求头携带 Token，服务端验证 Token 有效性并解析用户信息，无状态、跨域友好，是前后端分离 API 开发的主流方案。

### 9.1.2 WSGI 与 ASGI 协议原理

Python Web 框架与 Web 服务器之间的通信必须遵循标准化的网关协议，解决框架与服务器的兼容性问题，Python 生态先后形成了 WSGI 同步协议与 ASGI 异步协议两大标准。

#### 1. WSGI 协议：Web 服务器网关接口

WSGI（Web Server Gateway Interface）是 Python 同步 Web 应用的官方标准协议，定义在**PEP 3333**规范中，是 Django、Flask 等传统同步框架的底层通信基础，解决了早期 Python Web 框架与 Web 服务器互不兼容的问题。

##### （1）核心设计思想

WSGI 协议将 Web 应用分为两个核心角色：

- **WSGI 服务器**：负责接收客户端的 HTTP 请求，解析后传递给 WSGI 应用，接收应用返回的响应，封装为 HTTP 响应返回给客户端，典型实现：Gunicorn、uWSGI、Apache mod_wsgi；
- **WSGI 应用**：Python 编写的 Web 应用 / 框架，接收服务器传递的请求数据，执行业务逻辑，返回响应数据给服务器，典型实现：Django、Flask、Bottle。

WSGI 协议定义了二者之间的标准化通信接口，任何遵循该接口的服务器与应用都可以无缝配合。

##### （2）WSGI 应用接口规范

一个合法的 WSGI 应用，必须是一个**可调用对象**（函数、实现了`__call__`方法的类实例），且必须满足以下参数与返回值规范：

```python
def simple_wsgi_app(environ, start_response):
    """最简WSGI应用示例"""
    # 1. 业务逻辑处理
    status = "200 OK"
    response_headers = [("Content-Type", "text/plain; charset=utf-8")]
    response_body = b"Hello WSGI"

    # 2. 调用start_response回调，设置响应状态与响应头
    start_response(status, response_headers)

    # 3. 返回可迭代的字节流响应体
    return [response_body]
```

参数详解：

1. `environ`：字典类型，包含所有 HTTP 请求的元信息、服务器环境变量、WSGI 规范变量，如请求方法、请求路径、请求头、查询参数、服务器地址等，是 WSGI 应用获取请求数据的唯一来源；
2. `start_response`：服务器提供的回调函数，WSGI 应用必须在返回响应体之前调用该函数，传入两个必选参数：`status`（HTTP 状态码与描述）、`response_headers`（响应头的元组列表），可选参数`exc_info`用于异常处理。

返回值规则：必须返回一个**可迭代的字节对象**，服务器会迭代该对象，将字节流作为 HTTP 响应体发送给客户端。

##### （3）WSGI 的核心局限性

WSGI 是为同步阻塞应用设计的协议，存在无法突破的核心缺陷：

- 同步阻塞：一个请求处理完成之前，无法处理其他请求，高并发场景下性能瓶颈显著；
- 仅支持 HTTP 协议：无法支持 WebSocket、HTTP2 Server Push、长连接等现代 Web 协议；
- 不支持异步：无法兼容`async/await`异步语法，无法利用 Python 异步生态的高性能优势。

#### 2. ASGI 协议：异步服务器网关接口

ASGI（Asynchronous Server Gateway Interface）是 WSGI 协议的现代异步替代方案，是 Python 异步 Web 开发的官方标准，为同步、异步应用提供了统一的通信接口，原生支持 HTTP、WebSocket、HTTP3 等现代 Web 协议，是 FastAPI、Starlette、Django 3.0 + 异步视图的底层基础。

##### （1）核心设计优势

- 全异步原生支持：基于协程模型，单线程即可支持高并发请求，性能远超 WSGI 同步模型；
- 协议无关性：同时支持 HTTP、WebSocket、Server-Sent Events（SSE）、HTTP2 等多种协议，突破了 WSGI 仅支持 HTTP 的限制；
- 向后兼容：完全兼容 WSGI 应用，可通过适配器将传统 WSGI 应用运行在 ASGI 服务器上；
- 事件驱动：基于事件循环模型，IO 等待时自动切换到其他请求处理，无内核态线程切换开销，高并发场景下资源占用极低。

##### （2）ASGI 应用接口规范

ASGI 应用是一个**异步可调用对象**，接收三个固定参数，完整规范如下：

```python
async def simple_asgi_app(scope, receive, send):
    """最简ASGI应用示例"""
    # 1. scope：字典类型，包含请求/连接的所有元信息
    # 包括协议类型(http/websocket)、请求方法、路径、请求头、客户端地址等
    if scope["type"] != "http":
        return

    # 2. receive：异步可调用对象，用于接收服务端的事件消息（如请求体、客户端数据）
    message = await receive()
    if message["type"] == "http.request":
        body = message.get("body", b"")

    # 3. send：异步可调用对象，用于向客户端发送响应事件
    # 先发送响应起始事件（状态码、响应头）
    await send({
        "type": "http.response.start",
        "status": 200,
        "headers": [(b"content-type", b"text/plain; charset=utf-8")],
    })
    # 再发送响应体事件
    await send({
        "type": "http.response.body",
        "body": b"Hello ASGI",
    })
```

参数详解：

1. `scope`：字典类型，包含当前连接的所有上下文信息，核心字段`type`标识协议类型（`http`/`websocket`），是 ASGI 支持多协议的核心；
2. `receive`：异步可调用对象，无参数，await 后返回事件消息字典，用于接收客户端发送的数据、请求体、连接事件等；
3. `send`：异步可调用对象，接收事件消息字典，await 后向客户端发送响应、数据、控制事件等。

##### （3）ASGI 服务器与生态

- 主流 ASGI 服务器：Uvicorn（基于 uvloop 与 httptools，FastAPI 默认推荐）、Hypercorn、Daphne（Django Channels 官方）；
- 兼容 ASGI 的框架：FastAPI、Starlette、Django 3.0+、Sanic、Tornado 6.0+；
- 异步生态：ASGI 协议推动了 Python 异步 Web 生态的爆发，形成了完整的异步 ORM、异步 HTTP 客户端、异步缓存、异步消息队列等全链路生态。

#### 3. WSGI 与 ASGI 核心对比

|特性|WSGI|ASGI|
|---|---|---|
|规范定义|PEP 3333|社区标准，已成为 Python 异步 Web 事实标准|
|核心模型|同步阻塞|异步事件驱动|
|协议支持|仅支持 HTTP|支持 HTTP、WebSocket、SSE、HTTP2 等多协议|
|异步支持|不支持原生异步|原生支持 async/await 异步语法|
|高并发性能|低，依赖多进程 / 多线程提升并发|极高，单线程协程模型支持上万级并发|
|兼容框架|Django、Flask、Bottle 等传统同步框架|FastAPI、Starlette、Django 3.0+、Sanic 等，兼容 WSGI 应用|
|适用场景|低并发、传统同步业务、企业级管理系统|高并发 API、实时通信、长连接、微服务、IoT 场景|

---

