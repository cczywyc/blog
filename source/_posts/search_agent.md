---
title: "动手写一个搜索 Agent：从能跑通到受控运行"
description: "Tool Calling、结构化输出与 Agent Loop 的工程实践记录"
date: 2026-04-05
tags:
  - AI
categories:
  - AI文章合集
cover: https://img.cczywyc.com/post-cover/search_agent_cover.jpg
---

[上一篇文章](https://cczywyc.com/2026/03/16/再聊Chatbot-Workflow-RAG-Agent/)把 Chatbot/Workflow/RAG/Agent 四个概念的边界理清楚了。按照我的 roadmap，接下来两周正式上手写代码：先学 tool calling，做一个"网页搜索 + 总结"的最小 Agent；再做 prompt 约束、结构化输出和控制循环，把它从"能跑通"升级到"受控运行"。这篇文章就是这两周的完整记录——一个搜索 Agent 从 v1 到 v2 的演进过程，完整代码在文末的仓库里。

## Tool Calling：Agent 的行动能力从哪来

上篇文章说过，Agent 的"行动能力"来自工具。LLM 本身只能生成文本，是 tool calling 机制让它能搜索、调 API、读写文件。这个机制的核心流程在任何模型上都一样：

```
定义工具（JSON Schema）→ 模型决策 → 开发者执行 → 结果回传 → 模型综合
```

写第一个 tool calling 示例时，有个分界线让我印象很深：**模型不执行工具，它只决定调用什么。** 模型返回 `tool_calls` 之后，所有的执行权都在我的代码手中——参数校验、权限检查、危险操作确认，都可以在执行前拦截。这个设计是后面所有安全控制的基石。

把这个流程放进一个循环，就得到了所有 Agent 系统的骨架：

```python
messages = [system_prompt, user_message]

for turn in range(max_turns):
    response = llm.call(messages, tools)

    if finish_reason == "stop":          # 出口 1：模型给出最终回答
        return response.content

    if finish_reason == "tool_calls":    # 模型要调用工具
        messages.append(response)        # 保留模型的决策记录
        for tool_call in response.tool_calls:
            result = execute(tool_call)  # 执行权在开发者手中
            messages.append(result)      # 结果回传，进入下一轮

return "达到最大轮次"                     # 出口 2：安全终止
```

1. 循环只有两个正常出口：模型主动结束（finish_reason 为 stop），或者达到最大轮次被强制终止——后者是必须有的安全机制，防止模型陷入无限的工具调用。
2. 模型要调工具时，要先把它的决策记录（带 tool_calls 的 message）追加进历史，再把工具结果以 tool 角色追加进去，模型下一轮才能"看到"自己上一轮做了什么。
3. 千问通过 OpenAI 兼容接口调用，`arguments` 是 JSON 字符串，执行前需要 `json.loads()` 解析。

这个 while 循环看起来平平无奇，但后面会看到，"能跑通"的循环和"生产可用"的循环之间差距巨大。

## v1：最小可用的搜索 Agent

模型用的是阿里千问（qwen-plus），通过 OpenAI SDK 兼容接口调用（原计划用 OpenAI，中转服务折腾半天没通，千问有免费额度且国内直连）。两个工具都是只读的 Data 类工具：

- **web_search**：接入 DuckDuckGo（ddgs 库），返回精简的 title + url + snippet
- **fetch_webpage**：httpx 抓取网页 + readability 提取正文，过滤噪音

过程中踩的几个坑：

| # | 现象 | 原因 | 解法 |
|---|------|------|------|
| 1 | "北京"查不到天气数据 | 模型用中文填参数，mock 数据的 key 是英文 "Beijing" | description 里约束参数用英文 + 工具内部做中英文映射兜底 |
| 2 | duckduckgo_search 运行时 Warning | 这个包已更名为 ddgs | `pip install ddgs` 并更新 import |
| 3 | 搜索直接 ConnectError | DuckDuckGo 底层走 Bing，网络不通 | DDGS 构造函数里配置 proxy |
| 4 | fetch_webpage 在 Medium 等站点 403 | 反爬机制 | 当时没解，留到 v2（URL 黑名单） |

这两周里对我影响最大的一个认知是：**工具描述（description）是影响 Agent 质量的最大杠杆。** 读 Anthropic 的文档时看到这个说法还觉得浮于表面，亲手写代码才体会到差距——给 `web_search` 的 description 加上一句 "Does NOT fetch full content — use fetch_webpage for that" 之后，模型对两个工具的选择准确率明显提升，需要深入内容时会主动换用 fetch_webpage。

v1 跑了 7 个测试用例，结果不好：**通过 4/7**。失败的三个 case 很有代表性：

- **诺贝尔奖**：模型直接从训练数据回答，完全跳过搜索。回答"看起来正确"，但无法验证时效性——如果训练数据里有错误信息，这就是一个自信的错误，用户完全无法分辨。**最危险的失败，是看起来像成功的失败。**
- **LangGraph vs CrewAI 对比**：搜索成功了，但 fetch_webpage 连续遇到 403（Medium、DataCamp 都有反爬），5 轮全部耗在重试抓取上，最终无输出。
- **解释 RAG**：模型认为这是已知概念，直接回答，没有引用来源。

归纳下来，v1 暴露了三个问题：

1. 模型倾向于"能回答就回答"，即使 system prompt 里要求先搜索——这是 LLM 的固有行为模式
2. 输出格式不可控，没法程序化地校验和评测
3. 工具连续失败时没有降级机制，一条路走到黑

第三周的升级就是冲着这三个问题去的。

## 先做实验，再改代码

升级前我设计的方案是：在同一个请求里同时传 `tools` 和 `response_format`，让模型既能调工具、又输出固定 JSON。动手改代码之前，先写了三组最小脚本验证千问 API 的实际行为——结果直接推翻了这个方案：

| 组合 | 结果 |
|------|------|
| 只传 tools | ✅ 正常调用工具 |
| tools + json_object | ❌ 400 报错（千问强制要求 messages 里包含 "json" 关键词，文档没明说） |
| tools + json_schema | **不报错，但 json_schema 把 tools 吃掉了——模型不调用工具，直接返回 JSON** |

第三个结果最致命：它不是 API 层面的互斥（报错反而好排查），而是行为层面的互斥。如果跳过实验直接写代码，大概率要花一下午 debug 一个"工具怎么都不被调用"的灵异问题。

修正后的方案：主流程只传 `tools`；需要结构化输出时，等模型给出最终回答后，**另发一个独立请求**用 json_schema 做格式化。json_schema 的 `strict: True` 实测非常可靠——required 字段 100% 存在，enum 约束即使被 prompt 故意诱导也不会违反。

这件事给我的原则是：**文档告诉你"支持什么"，但不会告诉你"组合使用时谁优先"。API 的实际行为必须用实验验证，不能只看文档。**

## v2：给 Agent Loop 装上三张安全网

v2 的升级围绕一个核心认知：Agent Loop 不是"信任模型的循环"，而是**带安全网的决策循环**——模型的每次决策都可能出错，需要在它决策之后、执行前后插入检查和保护。

### 第一层防御：翻转 System Prompt 的默认方向

v1 的 prompt 说"你可以使用搜索工具"（许可语气），模型经常无视。v2 翻转默认方向——**必须先搜索，以下情况例外**：

```
# Core Rule: Search First
对于以下类型的问题，你 **必须** 先使用 web_search，禁止直接从记忆回答：
- 涉及具体人物、事件、时间、数据的事实性问题
- 涉及技术产品、框架、协议的特性、版本、对比
- 你不认识或不了解的任何名词、术语、缩写

仅以下情况允许不搜索直接回答：
- 纯创意写作（写诗、写故事、起名字）
- 数学计算或逻辑推理
- 日常闲聊和打招呼

当你不确定该不该搜索时，选择搜索。
```

再配上 3 个 few-shot examples（事实查询、技术概念、创意写作各一个），代替继续往上堆规则。这一层解决了约 80% 的问题：10 个测试 case 里 8 个在第一轮就做出了正确决策。

### 第二层防御：三个代码检查机制

剩下 20% 的 case，prompt 再怎么写都拦不住——模型"非常确信自己知道答案"的时候就是会跳过工具。这时候需要代码兜底。v2 在循环里加了三个检查机制，每个机制都明确定义三要素：触发条件、恢复行动、防护限制：

| 检查机制 | 触发条件 | 恢复行动 | 防护限制 |
|---------|---------|---------|---------|
| 纠正 | 模型 stop 时，规则判断"该搜没搜" | 注入纠正指令，强制重新决策 | 最多 1 次 |
| 降级 | 工具连续失败 ≥ 2 次 | 注入降级指令，让模型用已有 snippet 回答 | 最多 1 次 |
| 资源控制 | 上下文超过阈值 / 轮次到上限 | 警告 / 强制终止 | 每轮检查 |

"防护限制"这一列很重要——没有它，检查机制本身就可能把 Agent 拖进"检查 → 纠正 → 再检查"的死循环。纠正为什么最多 1 次？实测里跳过工具的 case 在纠正 1 次后都乖乖去搜索了；如果 1 次纠正还不够，说明 prompt 有更深层的问题，该去改 prompt 而不是加纠正次数。

其中"该搜没搜"的判断用的是一个纯规则引擎 `should_have_searched()`：6 条搜索规则（时间指示词、概念查询、对比类、专有名词特征——驼峰命名、全大写缩写、版本号等）加 4 条排除规则（创意写作、数学、闲聊、写代码），28 条单元测试全部通过后才集成进循环。规则引擎无法理解语义，但成本为零、可单独测试，作为第一道过滤足够了。

### 把每一步决策记下来：结构化 Trace

v1 调试全靠 print，看完日志还要人脑重建"模型在第几轮做了什么决策"。v2 给每次运行记录三层结构化 trace：

```
AgentTrace（整体：是否搜索过、是否触发纠正/降级、总耗时）
  └── TurnTrace（每轮：finish_reason、是否注入指令）
        └── ToolCallTrace（每次工具调用：参数、成败、耗时）
```

后面会看到，本周最有价值的一个问题就是 trace 暴露出来的。

## 100% 通过率下面藏着的问题

v2 跑完测试，10/10 全部通过，v1 失败的三个 case 全部修复。如果只看通过率，到这里就可以收工了。但翻 trace 数据时发现了一个问题：**10 个 case 里 7 个触发了纠正，其中 4 个是假阳性**——模型在第 1 轮已经搜索过了，第 2 轮准备回答时被误判为"跳过工具"，又被强制多搜了一次。

根因不复杂：`should_have_searched()` 是个无状态函数，只看用户的原始问题，不知道对话已经进行到哪了。修复方式是在纠正前加一个有状态的前置检查：

```python
already_searched = any(
    tc.tool_name == "web_search" and tc.result_success
    for t in trace.turns
    for tc in t.tool_calls
)

if (not correction_injected
        and not already_searched   # 新增：已经搜索过就不再纠正
        and should_have_searched(user_message)):
    inject_correction()
```

1. `already_searched` 遍历 trace 里的所有轮次，检查是否已经有过一次成功的 web_search 调用——这是一个有状态的判断，弥补了规则引擎只看静态输入的盲区。
2. 纠正的触发条件从两个变成三个：没纠正过、没搜索过、规则判断该搜索。三者同时满足才注入纠正指令。

修复后纠正触发率从 70% 降到 20%，平均轮次从 3.5 降到 2.5 左右。这个问题给我两个提醒：

1. **假阳性不导致功能错误，只导致效率下降，所以更容易被忽视。** 评测不能只看通过率，还要看代价——轮次、延迟、多余的 API 调用都是成本。
2. 这个问题是 trace 暴露的。如果没有 `correction_triggered` 这个字段，我根本不会知道 70% 的 case 被纠正过。**Agent 系统的可改进性，取决于它的可观测性。**

另一个有意思的数据是降级机制 0 次触发——v1 里连续 403 失败的那个 case，在 v2 里模型直接用搜索摘要回答了，根本没走到降级那一步。预防层（prompt）解决了问题，治疗层（代码）没被需要。这是好事，最好的安全网是永远不需要接住人的安全网——但它必须存在，就像数据库的备份恢复机制可能一年用不上一次，你也不会因此取消备份。

## v1 → v2 量化对比

| 指标 | v1 | v2 |
|------|:---:|:---:|
| 测试用例数 | 7 | 10 |
| 通过率 | 4/7 (57%) | 10/10 (100%) |
| 工具使用准确率 | 57% | 100% |
| 平均轮次 | 3.0 | ~2.5 |
| 核心代码行数 | ~250 | ~600 |
| 单元测试 | 0 | 28 条 |

## 几个核心认知

1. **Prompt 优化和代码检查是互补关系，不是替代关系。** Prompt 是预防层，解决 80% 的问题；代码检查是检测层，捕获漏网之鱼。最佳策略是让 prompt 层尽可能强、让代码层尽可能不被触发，但两层都必须存在——和安全领域的纵深防御是一个道理。
2. **Trace 是 Agent 工程的基础设施，不是可有可无的 debug 工具。** 不能观测的系统就不能被可靠地改进。
3. **API 的组合行为必须用实验验证。** 文档不会告诉你 tools 和 response_format 同时传的时候谁说了算。
4. **手写控制循环的复杂度是指数增长的。** v1 的循环只有两个分支，很清晰；v2 加了三个检查机制后，主循环 150+ 行、嵌套 4-5 层，每加一个机制分支数接近翻倍。这也让我提前理解了 LangGraph 这类框架存在的意义——把纠正、降级、回答拆成状态图上的独立节点，是管理这种复杂度的更好工具（这是 roadmap 第六周的内容，到时候再写）。

按照 roadmap，下一步是 RAG。搜索 Agent 的"搜索 → 抓取 → 基于内容回答"和 RAG 的"检索 → 取文档 → 基于文档回答"在逻辑上几乎同构，Agent Loop、trace、规则引擎应该都能直接复用，到时候验证一下这个判断。

本文的完整代码（含测试用例和 trace 报告）在 [ai-agent-lab](https://github.com/cczywyc/ai-agent-lab) 仓库的 week_2 和 week_3 目录。

（全文完）

## 附：引用

- [Anthropic - Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents)：Agent 工程实践与"先简单后复杂"原则
- [Anthropic - Writing Tools for Agents](https://www.anthropic.com/engineering/writing-tools-for-agents)：工具设计方法论
- [OpenAI - A Practical Guide to Building AI Agents](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)：产品与工程视角的 Agent 设计
- [Anthropic - Effective Context Engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)：上下文工程与"正确高度"原则
