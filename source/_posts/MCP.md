---
title: "聊聊MCP"
description: "关于MCP入门和使用的文章"
date: 2025-04-26
tags:
  - AI
categories:
  - AI文章合集
cover: https://img.cczywyc.com/post-cover/mcp_cover.jpg
---

模型上下文协议（MCP）是一个开放协议，用于标准化应用程序向大语言模型提供上下文的方式，由 Anthropic 于 2024 年 11 月推出。在 MCP 出现之前，将 AI 模型连接到各种数据源通常需要为每个源开发定制化的集成方案，导致系统碎片化、难以扩展且开发成本高昂，MCP 则通过提供一个通用的、标准化的协议来解决这个问题。

## MCP 架构

MCP 的核心架构遵循客户端-服务器模型，主要包含三个组件：

1. **MCP 主机（MCP Host）**：指的是希望通过 MCP 利用外部数据或工具的应用程序。例如 AI 助手（如 Claude Desktop）、集成开发环境（IDE）、或其他 AI 工具。MCP Host 负责管理 MCP 客户端实例，并控制权限和安全策略。
2. **MCP 客户端（MCP Client）**：嵌入在主机应用程序中，是实现 MCP 协议客户端逻辑的部分。它负责与一个或多个 MCP 建立并维护安全、一对一的连接，管理通信、状态和能力协商。
3. **MCP 服务器（MCP Server）**：是实现协议服务器端逻辑的程序。它连接到具体的数据源（本地文件、数据库等）或远程服务（API），并通过 MCP 协议暴露其能力。MCP Server 可以是本地进程，也可以是远程服务。

![](https://img.cczywyc.com/MCP_core_components.webp)

### MCP Server 暴露的核心能力

MCP Server 主要向客户端暴露三种类型的能力，是 AI 模型能够更深入地理解上下文并执行任务：

1. Resources：提供 AI 模型可以读取的类文件数据或上下文信息。这可以包括文件内容、数据库记录、API 响应、系统信息或应用程序状态等 。例如，文件系统服务器可以暴露本地文档作为资源，数据库服务器可以暴露查询结果作为资源。AI 模型可以将这些资源加载到其上下文中，以获取执行任务所需的信息。
2. Tools：定义 AI 模型可以（在用户批准后）调用的函数或操作 。这些工具使 AI 能够执行计算、访问外部 API、查询数据库、操作文件或与外部系统交互 。例如，GitHub 服务器可能提供 `create_issue` 或 `list_repositories` 等工具 。工具是实现“代理行为”（Agentic Behavior）的关键，让 AI 从信息处理者转变为任务执行者。
3. Prompts：提供预定义的、可重用的提示模板或工作流，通常可以通过斜杠命令或菜单选项触发，以帮助用户或 AI 模型完成特定任务 。这有助于引导交互、结构化请求，并确保一致性。

通过以上三种能力的组合，MCP Server 极大地扩展了 AI 模型的功能边界，使其能够更有效地与现实世界的数据和系统进行交互。

## MCP 解决了哪些问题

MCP 通过标准化协议解决了 AI 工具在实际应用中面临的两大核心问题：数据接入的局限性和任务执行的缺失。

在 MCP 出现之前，通用大语言模型（LLM）的训练数据是有限的，它只能来自公开的数据源。在实际使用场景下，我们往往需要使用多个不同的数据源作为 LLM 的上下文语料输入，例如：在 AI 工具里，我需要同时使用本地文件的一些资料和高德地图以及 github 的资源，在这种场景下，一般都需要单独定制开发；再比如，我需要 AI 工具帮我在本地 terminal 执行一些命令时，往往需要借助系统提供的 api 来完成。

那么，再有了 MCP 以后，上述场景变成了什么样子的呢？现在可以使用不同的 MCP Server，用一个标准的、通用的协议来完成上述事情。比如，我可以配置本地 file 和 terminal 的 MCP，github 和 高德也都有官方提供的 MCP Server，我们只需要进行一些简单的配置，就可以通过自然语言来描述我们的需求，AI 工具（也叫 MCP Host）会自动选择合适的 MCP Client 和 对应的 MCP Server 通信，来完成各种复杂的任务。

![](https://img.cczywyc.com/MCP_before_and_after.png)

总得来说，MCP 通过以下方式解决了上述问题：

* 提供标准化的交互方式：MCP 为 AI 工具提供了一种标准化的方法来发现和调用外部系统中的操作。它充当了AI“理解”与“执行”之间的桥梁，使AI不仅能回答问题，更能主动完成任务。
* 简化集成开发：通过标准化的协议，MCP大大降低了开发者为AI构建与关键工具安全集成的难度和时间成本。
* 提高 AI 工具间的可替换性：由于采用了统一标准，相似类型的应用（如不同的云存储服务）在MCP服务器实现上会很相似，使得在它们之间切换变得非常简单，可能只需要修改几行代码，而无需重写整个集成。

## 工作原理

### 通信协议：JSON-RPC 2.0

MCP 的核心通信机制建立在 JSON-RPC 2.0 协议之上。这是一种轻量级的远程过程调用（RPC）协议，使用 JSON 作为数据格式。选择 JSON-RPC 2.0 提供了几个优势：

* 标准化：它是一个成熟且广泛理解的标准，简化了不同语言和平台之间的互操作性。
* 轻量级：JSON 格式简洁，易于解析，开销较低。
* 支持多种消息类型：JSON-RPC 2.0 定义了请求（Request）、响应（Response，包括成功结果 Result 和错误 Error）以及通知（Notification）等消息类型，满足了 MCP 所需的双向、有状态通信需求。
* 状态管理：MCP 利用 JSON-RPC 建立有状态的连接，支持能力协商、进度跟踪、取消操作和错误报告等高级功能。

所有通过 MCP 传输层发送的消息（无论是请求、响应还是通知）都遵循 JSON-RPC 2.0 的规范进行编码和解码。

### 传输层：STDIO 与 SSE

MCP 协议本身是与传输层无关的，但它定义了两种标准的传输机制，用于在客户端和服务器之间实际传输 JSON-RPC 消息：

1. STDIO (Standard Input/Output):
    * 机制：在这种模式下，MCP 客户端（通常在主机应用内）将 MCP 服务器作为一个子进程启动。服务器通过其标准输入（stdin）读取来自客户端的 JSON-RPC 消息，并将响应或通知写入其标准输出（stdout）。
    * 适用场景：这是本地集成的理想选择，即客户端和服务器在同一台机器上运行时。它设置简单，通信效率高，适用于进程间通信。官方建议客户端尽可能支持 stdio。许多 SDK 提供内置的 STDIO 传输实现。
    * 由于通信限制在本地进程间，通常被认为相对安全，但仍需注意服务器进程的权限管理。
2. Streamable HTTP (with SSE - Server-Sent Events):
    * 机制：这种模式下，服务器作为一个独立的进程运行（可能在远程机器上），能够处理多个客户端连接。客户端通过标准的 HTTP POST 请求向服务器发送 JSON-RPC 消息。服务器可以使用 Server-Sent Events (SSE) 技术通过一个持久的 HTTP 连接向客户端流式发送响应、通知或服务器发起的请求。客户端也可以通过 HTTP GET 请求启动 SSE 流，以便服务器可以主动向客户端发送消息。
    * 适用场景：适用于需要网络通信的场景，例如连接到远程服务器或基于 Web 的应用。它利用了广泛使用的 HTTP 协议，易于穿透防火墙。微软 Copilot Studio 目前仅支持 SSE 传输。
    * 安全：由于涉及网络通信，安全性变得尤为重要。必须使用 TLS 加密传输。服务器需要实施严格的认证和授权机制，并验证请求来源（例如，检查 Origin 头）以防止 DNS 重绑定等攻击。实现者需要仔细处理连接管理、错误恢复（SSE 支持通过 Last-Event-ID 头进行断线重连）和潜在的拒绝服务攻击。

### 工作流程

当用户与支持 MCP 功能的 AI 工具交互时，会进行一系列的通信过程。

#### 协议握手

1. 连接初始化（Initial connection）：交互开始，MCP Host 通过 MCP Client 连接到已经配置好的每个 MCP Server；
2. 能力发现（Capability discovery）：MCP Host 会询问每个 MCP Server：“你可以提供什么能力？”，每个 MCP Server 会响应自身可以提供的功能、可以调用的工具等；
3. 登记注册（Registration）：MCP Host 登记注册每个 MCP Server 的能力，以便在后续交互过程中决定请求哪个 MCP Server。

![](https://img.cczywyc.com/MCP_protocol_handshake.png)

下面以询问 “杭州今天天气怎么样？”为例来介绍整个流程。

1. 需求识别：MCP Host 分析识别问题，识别到此任务需要获取外部数据源的实时信息；
2. 工具或资源选择：MCP Host 确定需要使用 MCP 来完成此任务；
3. 权限请求：MCP Client 确认是否有访问外部数据源的权限；
4. 请求过程：权限一旦得到确认，MCP Host 便调用相应的 MCP Client 使用标准的协议格式向对应的 MCP Server 发送请求；
5. 外部处理：MCP Server 收到来自 MCP Client 的请求，开始调用外部资源进行处理，如：查询天气服务、房访问文件系统或访问数据库；
6. 返回结果：MCP Server 将处理后的结果以标准化格式返回给 MCP Client；
7. 上下文整合：MCP Host 将返回的结果结合上下文环境进行结果整合；
8. 生成响应：MCP Host 根据整合后的上下文生成最终的结果返回给用户。

## 配置与开发

### 配置

MCP 的使用配置也相对简单。

首先需要选用支持 MCP 功能的 MCP Host，如 Claude Desktop、cursor、cline、trae 等，将 MCP Server 集成到 MCP Host 中，配置的核心就是告诉 MCP Host 如何启动和与 MCP Server 通信。

* 通用配置元素：配置通常涉及以下信息：

  * 命令（Command）：启动服务器进程的可执行文件或命令（例如 `npx`, `python`, `docker`）
  * 参数（Args）：传递给启动命令的参数列表（例如脚本路径、服务器选项、要暴露的目录等）
  * 环境变量（Env）：为服务器进程设置的环境变量（例如 API 密钥、访问令牌、数据库连接字符串等）

* Claude Desktop 配置示例（clause_desktop_config.json）：

  Clause Desktop 使用一个 JSON 文件（通常位于 `~/Clause/clause_desktop_config.json`）来配置 MCP Server，其基本结构如下：

  ```json
  {
    "mcpServers": {
      "server-identifier-1": {
        "command": "command_to_run_server_1",
        "args": ["arg1", "arg2"],
        "env": {
          "API_KEY": "your_api_key_here"
        }
      },
      "server-identifier-2": {
        "command": "docker",
        "args": [],
        "env": {
          "GITHUB_TOKEN": "<YOUR_GITHUB_TOKEN>"
        }
      }
    }
  }
  ```
  
  * `mcpServers` 是包含所有 MCP Server 配置的顶层对象
  * `server-identifier-1`，`server-identifier-2` 是为每个 MCP Server 指定的唯一标识符（例如：filesystem，github，minecraft）

备注：上述只是介绍了使用 Claude Desktop 配置 MCP Server 的示例，其他的工具比如 cursor、trae 等现在都提供了比较简单的配置方式和交互路径，没有什么难度。

### 开发

我们可以根据实际场景开发自己的 MCP Client 和 MCP Server，为了简化整个开发流程，[官方](https://modelcontextprotocol.io/introduction)和社区提供了多种语言的 SDK 和框架，这些 SDK 封装了协议的复杂性，提供了更高级别的抽象。

* 官方主要 SDK：
  * TypeScript/JavaScript
  * Python
  * Java/Kotlin（包括对 Spring 的支持）
  * Go
  * Swift
* SDK 特性：
  * 核心 MCP Client 和 MCP Server 实现
  * 内置的传输层实现（如 STDIO 和 SSE），通常无需外部 Web 框架即可使用
  * 对同步和异步编程范式的支持
  * 用于定义和处理能力的辅助类和函数
  * 协议版本协商、列表变更通知等协议功能的实现

## 比较与总结

MCP 提供了一种独特的 AI 集成方法，与其他现有技术既有重叠也有区别，最后再来简单总结一下 MCP 与其他集成工具的区别：

* MCP vs 传统 API 调用：
  * 范围：传统 API 是通用的软件接口；MCP 专为 AI 与外部世界交互而设计，侧重于以标准化方式提供上下文（资源）、动作（工具）和引导（提示）
  * 抽象层级：直接调用 API 需要 AI 或其宿主应用处理认证、请求格式化、错误处理等细节。MCP Server 则封装了这些细节，为 AI 提供了一个更高层次、更一致的接口
  * 集成复杂度：MCP 旨在解决 N×M 的集成难题，通过标准化接口减少所需的连接数量
* MCP vs RAG（Retrieval-Augmented Generation）：
  * 交互模式：RAG 主要是一种被动的信息检索机制，通过从知识库（通常是向量数据库）中查找相关文本片段并注入提示词来增强 LLM 的上下文 。MCP 则支持主动交互，AI 可以通过“工具”执行实时查询、调用 API 或执行操作，而不仅仅是检索静态文本
  * 数据时效性：RAG 依赖于预先索引的数据，可能存在时效性问题。MCP 可以直接访问实时数据源
  * 计算开销：RAG 通常需要计算密集型的嵌入生成和向量搜索。MCP 的直接访问模式可以避免这部分开销
  * 关系：MCP 和 RAG 并非完全互斥，可以互补。例如，一个 MCP Server 可以实现一个“工具”，该工具的功能是执行向量数据库查询，从而将 RAG 的能力整合到 MCP 框架下
  
  **备注：我本人对 RAG 也十分感兴趣，后面肯定会单独研究一下 RAG。**
* MCP vs 框架工具（如 LangChain Tools）：
  * 标准化层面：LangChain 等框架提供了一个开发者面向的标准（例如，其 Tool 类接口），用于在代码层面集成工具。MCP 则提供了一个模型（或运行时 Agent）面向的标准，允许 AI Agent 在运行时发现和使用任何符合 MCP 规范的服务器暴露的工具，即使这些工具在编写 Agent 代码时是未知的
  * 互补性： 这两者是高度互补的。LangChain 已经提供了适配器，可以将 MCP Server 暴露的工具视为 LangChain 工具来使用 。开发者可以使用 LangChain 来构建 Agent 的逻辑流程，同时利用 MCP 作为与外部工具交互的标准化接口

结来说，MCP 的核心优势在于其开放性、标准化和对主动交互（工具执行）的强调，旨在创建一个更统一、更灵活的 AI 集成生态系统。然而，它的成功依赖于是否被广泛的采纳，并且它将集成的复杂性从直接 API 调用转移到了实现和维护符合协议标准的服务器上。

（全文完）

## Refrence

1. https://modelcontextprotocol.io/introduction -- MCP 官方文档